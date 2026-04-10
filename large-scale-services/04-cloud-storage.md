# 클라우드 스토리지 (Google Drive / Dropbox)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 파일 업로드, 다운로드, 삭제
- 파일 동기화 (여러 기기에서 최신 상태 유지)
- 폴더 구조 관리
- 파일 버전 관리 (이전 버전 복원)
- 파일 공유 (링크 공유, 특정 사용자 공유)

### 비기능 요구사항
- DAU 5,000만 명
- 최대 파일 크기: 5GB
- 파일 저장 총 용량: 1억 사용자 × 평균 10GB = 1EB(엑사바이트)
- 강한 일관성 (동기화된 파일 항상 최신 버전)

---

## 📊 규모 산정

```
업로드/다운로드:
  DAU 5,000만 × 2회 업로드/일 = 1억 건/일 ≈ 1,200 TPS
  평균 파일 크기: 500KB
  일일 업로드: 1억 × 500KB = 50TB/일

저장소:
  1억 사용자 × 15GB(평균 사용) = 1.5EB
  버전 관리(5버전): 1.5EB × 5 = 7.5EB
  중복 제거(Dedup): 실제 저장 약 50% 절감 → ~3.75EB

메타데이터 DB:
  파일 1억 × 사용자 1억 = 너무 많음
  파일 수: 사용자당 평균 1,000개 → 1,000억 개 파일 레코드
```

---

## 🏗️ 개략적 설계

```
[업로드 흐름]

클라이언트
  │ (1) 업로드 시작 요청
  ▼
┌──────────────┐
│  API 서버    │  메타데이터 관리
│  (Upload)    │  파일 청크 조율
└──────┬───────┘
       │ (2) 청크 업로드 URL 발급
       ▼
클라이언트 → Chunk Upload 서버 → S3 (파일 실제 저장)
                                       │
                                       │ (3) 업로드 완료
                                       ▼
                               메타데이터 DB 업데이트
                                       │
                                       │ (4) 동기화 이벤트
                                       ▼
                               Kafka (file-sync-topic)
                                       │
                                       ▼
                               Notification Service
                               → 다른 기기/공유자에게 알림

[동기화 흐름]

기기 A에서 파일 수정
      │
      ▼
  변경 감지 (파일 시스템 Watch)
      │
      ▼
  변경된 청크만 업로드 (델타 동기화)
      │
      ▼
  API 서버 → 메타데이터 업데이트 → Kafka 이벤트
                                        │
                                        ▼
                               기기 B, C에 변경 알림
                                        │
                                        ▼
                               기기 B, C가 변경된 청크만 다운로드
```

---

## 🔬 핵심 컴포넌트

### 청크 분할 업로드 (Block-level Sync)

```
문제: 5GB 파일 수정 후 전체 재업로드 → 너무 느림

해결: 파일을 4MB 청크로 분할, 변경된 청크만 동기화

파일: document.docx (100MB)
청크 분할:
  chunk_001 (4MB) SHA256: abc123
  chunk_002 (4MB) SHA256: def456
  chunk_003 (4MB) SHA256: ghi789
  ...
  chunk_025 (4MB) SHA256: xyz000

수정 후:
  chunk_002 내용 변경 → SHA256: NEW_456 (해시 변경)
  
  클라이언트: chunk_002만 업로드 (4MB)
  서버: chunk_002만 교체, 나머지 재활용

장점:
  100MB 파일 수정 → 4MB만 전송 (96% 절감)
  중복 청크 재활용 (여러 파일이 같은 청크 공유)
```

### 중복 제거 (Deduplication)

```
같은 내용의 청크를 한 번만 저장:

사용자 A: photo.jpg → 청크 SHA256: "aaa111" → S3에 저장
사용자 B: 같은 photo.jpg 업로드 → 청크 SHA256: "aaa111"
  → 이미 존재! → S3 재업로드 없이 동일 청크 참조

메타데이터 DB:
  chunks 테이블: (chunk_hash, s3_path, ref_count)
  file_chunks 테이블: (file_id, chunk_index, chunk_hash)

효과:
  같은 파일이 여러 사용자에 의해 공유되는 경우
  (예: 회사 공통 문서) → 한 번만 저장
  전체 저장 공간 20~50% 절감
```

### 메타데이터 DB 설계

```sql
-- 파일 테이블
CREATE TABLE files (
    file_id     BIGINT PRIMARY KEY,
    owner_id    BIGINT NOT NULL,
    parent_id   BIGINT,           -- 폴더 ID (NULL이면 루트)
    name        VARCHAR(255),
    is_folder   BOOLEAN DEFAULT FALSE,
    size        BIGINT,
    checksum    VARCHAR(64),      -- 전체 파일 SHA256
    version     INT DEFAULT 1,
    created_at  TIMESTAMP,
    updated_at  TIMESTAMP,
    INDEX (owner_id, parent_id)   -- 폴더 내 파일 목록 조회
);

-- 청크 테이블
CREATE TABLE chunks (
    chunk_hash  VARCHAR(64) PRIMARY KEY,  -- SHA256
    s3_path     VARCHAR(500),
    size        INT,
    ref_count   INT DEFAULT 1             -- 참조 카운트
);

-- 파일-청크 매핑
CREATE TABLE file_chunks (
    file_id     BIGINT,
    version     INT,
    chunk_index INT,              -- 순서
    chunk_hash  VARCHAR(64),
    PRIMARY KEY (file_id, version, chunk_index)
);

-- 버전 이력
CREATE TABLE file_versions (
    file_id     BIGINT,
    version     INT,
    created_at  TIMESTAMP,
    created_by  BIGINT,
    comment     VARCHAR(200),    -- 버전 설명
    PRIMARY KEY (file_id, version)
);
```

### 파일 공유

```
링크 공유:
  POST /files/{file_id}/share
  → 공유 토큰 생성 (UUID)
  → share_links 테이블에 저장:
     (token, file_id, permission, expires_at, password_hash)
  
  누구나 접근 가능한 URL: /share/{token}
  → 토큰 검증 → 파일 다운로드 허용

특정 사용자 공유:
  file_permissions 테이블:
  (file_id, user_id, permission)
  permission: 'view', 'edit', 'admin'

폴더 권한 상속:
  폴더에 권한 부여 → 하위 모든 파일에 자동 적용
  → 조회 시 부모 폴더 권한까지 재귀 확인
  (성능 최적화: Redis에 캐시)
```

---

## 🔄 데이터 흐름 — 충돌 해결

```
문제: 기기 A와 B에서 동시에 같은 파일 수정 시?

Dropbox 방식:
  기기 A가 먼저 업로드 → 서버 버전 = A버전
  기기 B가 업로드 시도 → 서버 버전 > B의 기준 버전
  → 충돌 감지!
  → B 파일을 "document (B의 충돌 버전 2024-01-01).docx"로 저장
  → 사용자에게 두 버전 모두 제공 → 수동 병합 유도

Google Docs 방식 (Operational Transformation):
  실시간 공동 편집 (Character 단위 연산 변환)
  "Insert A at pos 5" + "Delete B at pos 3" → 순서 변환 처리
  → 충돌 없이 자동 병합 (복잡한 알고리즘 필요)

파일 동기화(Dropbox): 충돌 시 두 버전 저장
문서 협업(Google Docs): OT 알고리즘으로 실시간 병합
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| 파일 저장 | S3 + 청크 분할 | 대용량, 중복 제거, 변경 청크만 전송 |
| 메타데이터 | MySQL (강한 일관성) | 파일 구조는 정확해야 함, ACID 보장 |
| 동기화 | 이벤트 기반 (Kafka) | 실시간 변경 전파, 비동기 |
| 충돌 | 두 버전 저장 (Dropbox식) | 구현 단순, 사용자가 직접 선택 |

---

## 🚀 확장 전략

```
저장 최적화:
  자주 접근 파일: S3 Standard
  30일 미접근: S3 Infrequent Access (비용 40% 절감)
  1년 미접근: S3 Glacier (비용 80% 절감)
  → S3 Lifecycle 정책 자동 설정

다운로드 가속:
  CloudFront CDN (자주 다운로드되는 파일)
  사용자 위치와 가까운 엣지에서 서빙

메타데이터 DB 확장:
  owner_id 기반 샤딩
  파일 수 많은 사용자 → 별도 샤드
  읽기 Read Replica 추가 (파일 목록 조회)
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 청크 분할: 4MB 단위 분할, SHA256 해시 기반 중복 제거
  2. 델타 동기화: 변경된 청크만 업로드 (대역폭 절감)
  3. 메타데이터: MySQL (ACID, 폴더 구조, 버전 관리)
  4. 파일 저장: S3 + Lifecycle (Hot/Warm/Cold)
  5. 충돌: 두 버전 저장 + 사용자 선택 (단순성 우선)
```

---

## 🤔 심화 질문

**Q1. 파일 동기화 시 네트워크 단절 후 재연결되면 어떻게 처리하는가?**
> 클라이언트는 로컬 파일 상태와 서버 상태를 비교하는 "sync queue"를 유지합니다. 재연결 시 마지막 동기화 타임스탬프 이후 변경된 파일 목록을 서버에서 조회합니다. 각 파일의 checksum을 비교해 실제로 변경된 파일만 동기화합니다. Dropbox는 이를 "LongPoll + local journal" 방식으로 구현합니다.

**Q2. 파일을 삭제했다가 복구하는 기능(휴지통)을 어떻게 구현하는가?**
> 삭제 시 실제로 S3에서 지우지 않고 files 테이블의 is_deleted=TRUE, deleted_at=NOW()로만 표시합니다(소프트 삭제). 30일 후 실제 삭제를 실행하는 배치 잡을 실행합니다. 복구 시 is_deleted=FALSE로 되돌립니다. 청크 데이터는 ref_count > 0인 경우에만 유지합니다.

**Q3. 대용량 파일(5GB) 업로드 중 네트워크가 끊기면 어떻게 되는가?**
> S3 Multipart Upload를 활용합니다. 파일을 100MB 단위로 나눠 각 Part를 독립적으로 업로드합니다. 서버는 업로드된 Part 목록을 추적합니다. 재연결 시 미완료 Part만 이어서 업로드합니다. 클라이언트는 어느 청크까지 성공했는지 로컬에 기록해 중단 지점부터 재개합니다.

---

<div align="center">

[⬅️ 이전: 채팅 시스템](./03-chat-system.md) | [README로 돌아가기](../README.md) | [다음: 검색 엔진 ➡️](./05-search-engine.md)

</div>
