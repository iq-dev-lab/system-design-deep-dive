# 스토리지 계층

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Block Storage, Object Storage, File Storage는 어떻게 다른가?
- S3 같은 Object Storage가 이미지/영상 저장에 적합한 이유는?
- 이미지를 DB에 저장하면 왜 안 되는가?
- CDN + Object Storage 조합은 어떻게 동작하는가?
- 대용량 파일 업로드를 어떻게 처리하는가?

---

## 🔍 왜 실무에서 중요한가

사용자가 업로드하는 이미지, 동영상, 문서를 어디에 저장하느냐는 서비스 확장성과 비용에 직접 영향을 미칩니다. 이미지를 DB에 저장하면 DB 성능이 급락하고, 서버 로컬 디스크에 저장하면 수평 확장이 불가능합니다. 스토리지 선택은 처음부터 올바르게 해야 나중에 마이그레이션 비용을 아낄 수 있습니다.

---

## 😱 흔한 실수

```
❌ 실수 1: 이미지를 DB BLOB으로 저장
  INSERT INTO images (data) VALUES (LOAD_FILE('image.jpg'))
  
  문제:
  ├── DB 용량 급증 (수 GB → 수십 TB)
  ├── DB 백업 시간 폭발
  ├── 이미지 하나에도 DB Connection 사용
  └── DB가 이미지 서빙까지 담당 → 핵심 비즈니스 쿼리 느려짐

❌ 실수 2: 서버 로컬 디스크에 저장
  /var/www/uploads/image.jpg
  
  문제:
  ├── 서버 1에 저장한 파일을 서버 2에서 접근 불가
  ├── 서버 장애 시 파일 손실
  └── 수평 확장 불가 (Sticky Session 강제)

❌ 실수 3: 파일명을 원본 그대로 사용
  user_photo.jpg → 다른 사용자가 같은 파일명 업로드 시 덮어쓰기
```

---

## 📖 스토리지 유형 비교

### Block Storage

```
특징:
  ├── 운영체제가 블록(섹터) 단위로 관리 (파일시스템 위에 올라감)
  ├── 서버에 직접 마운트 → 로컬 디스크처럼 사용
  └── 지연시간 매우 낮음 (μs ~ ms 수준)

사용:
  ├── 운영체제 볼륨
  ├── DB 스토리지 (MySQL Data 디렉토리)
  └── 고성능 I/O가 필요한 앱

AWS: EBS (Elastic Block Store)
예: EC2에 EBS 볼륨 마운트 → MySQL 데이터 저장

한계:
  ├── 특정 서버 한 대에만 마운트 가능 (기본)
  └── 확장 시 마이그레이션 필요
```

### File Storage (Network File System)

```
특징:
  ├── 파일 시스템 인터페이스 (디렉토리 구조, 파일 이름)
  ├── 여러 서버에서 동시에 마운트 가능
  └── NFS, SMB 프로토콜

사용:
  ├── 여러 서버가 공유해야 하는 파일 (설정 파일, 공유 자원)
  └── 레거시 애플리케이션이 파일 시스템을 기대하는 경우

AWS: EFS (Elastic File System)
예: 여러 EC2가 같은 EFS를 마운트 → 파일 공유

한계:
  ├── Object Storage보다 비쌈
  └── 대규모 파일 서빙에는 부적합
```

### Object Storage

```
특징:
  ├── 파일을 "객체"로 저장 (Key-Value 방식)
  ├── 계층적 디렉토리 없음 (Bucket + Key로 식별)
  ├── HTTP API로 접근 (PUT, GET, DELETE)
  ├── 이론적으로 무한 확장
  └── 지연시간: 수십~수백 ms (Block보다 느림)

사용:
  ├── 이미지, 동영상, 오디오
  ├── 백업 파일, 로그
  ├── 정적 웹사이트 호스팅
  └── 데이터 레이크

AWS: S3 (Simple Storage Service)
GCP: Cloud Storage
Azure: Blob Storage

접근 예:
  s3.amazonaws.com/mybucket/users/1001/profile.jpg
  ├── mybucket: 버킷 이름
  └── users/1001/profile.jpg: 객체 키 (디렉토리처럼 보이지만 단순 문자열)
```

---

## 🖼️ 이미지/영상 저장 설계 — S3 + CDN

### 왜 S3 + CDN인가

```
S3만 사용:
  서울 사용자 → S3 (미국) → 이미지 (180ms RTT) ← 느림

S3 + CloudFront:
  서울 사용자 → CloudFront 서울 엣지 → 이미지 (10ms)
  엣지에 없으면 → S3에서 가져와 캐시

  ┌───────────────────────────────────────────────────┐
  │  서울 CloudFront 엣지                              │
  │  (이미지 캐시)                                     │
  └──────────────────────────┬────────────────────────┘
                             │ Cache Miss
  ┌──────────────────────────▼────────────────────────┐
  │  S3 (오리진)                                       │
  │  s3://my-bucket/images/                           │
  └───────────────────────────────────────────────────┘
```

### 이미지 업로드 플로우

```
방법 1: 서버를 통한 업로드 (일반적)

  Client ──이미지──▶ App Server ──PUT──▶ S3
  Client ◀── URL ── App Server
  
  단점: 이미지가 App Server를 경유 → 서버 대역폭 소비

방법 2: Presigned URL (권장, 서버 우회)

  1. Client → App Server: "이미지 업로드할게요"
  2. App Server → S3: Presigned URL 생성 (유효 시간 5분)
  3. App Server → Client: Presigned URL 반환
  4. Client → S3: Presigned URL로 직접 PUT 업로드
  5. Client → App Server: "업로드 완료, URL은 이거예요"
  6. App Server → DB: 이미지 URL 저장

  ┌────────┐ 1.요청  ┌──────────┐
  │ Client │ ──────▶ │  App     │ 2.Presigned URL 생성
  │        │ ◀────── │  Server  │ ─────────────────▶ ┌──────┐
  │        │ 3.URL   └──────────┘                    │  S3  │
  │        │ 4.직접 PUT ────────────────────────────▶ │      │
  └────────┘ 5.완료 알림 ──▶ App Server               └──────┘
  
  장점:
    ✅ App Server 대역폭 절약
    ✅ S3가 직접 업로드 처리 → 확장성
    ✅ Presigned URL 만료로 보안 제어
```

---

## 🎬 대용량 파일 업로드 — Multipart Upload

동영상 같은 대용량 파일(수 GB)은 한 번에 올리면 실패 시 처음부터 다시 해야 합니다.

```
Multipart Upload:
  파일을 청크(보통 5~100MB)로 분할 → 각 청크를 병렬 업로드 → S3가 합치기

  1. Initiate Multipart Upload → Upload ID 발급
  
  2. Part 업로드 (병렬):
     Part 1 (0~4.9GB) ──▶ S3 → ETag 1
     Part 2 (5~9.9GB) ──▶ S3 → ETag 2  (동시에)
     Part 3 (10GB~)   ──▶ S3 → ETag 3
  
  3. Complete Multipart Upload (ETag 목록 전송 → S3가 합치기)

장점:
  ├── 실패 시 해당 파트만 재업로드
  ├── 병렬 업로드로 속도 향상
  └── 네트워크 불안정 환경에서 안정적

AWS S3 제한:
  └── 최소 파트 크기: 5MB (마지막 파트 제외)
```

---

## 🗂️ 스토리지 계층화 (Hot / Warm / Cold)

데이터 접근 빈도에 따라 스토리지 티어를 나눠 비용을 절감합니다.

```
Hot (자주 접근, 빠른 응답):
  └── S3 Standard: 최근 3개월 이내 데이터
      비용: $0.023/GB/월

Warm (가끔 접근):
  └── S3 Standard-IA (Infrequent Access): 3~12개월 데이터
      비용: $0.0125/GB/월 (Hot의 54%)
      단점: 조회 시 추가 비용 발생

Cold (드물게 접근, 장기 보관):
  └── S3 Glacier: 1년 이상 데이터 (백업, 규제 보관)
      비용: $0.004/GB/월 (Hot의 17%)
      단점: 복원에 수 분~수 시간 소요

자동 계층화:
  S3 Intelligent-Tiering: 접근 패턴 자동 분석 → 최적 티어 이동
  S3 Lifecycle Policy:
    30일 후 → Standard-IA 이동
    90일 후 → Glacier 이동
    365일 후 → Glacier Deep Archive 이동 또는 삭제
```

---

## 📌 핵심 결정 요약

| 저장 대상 | 스토리지 선택 | 이유 |
|-----------|-------------|------|
| DB 데이터 | Block Storage (EBS) | 낮은 지연시간, DB 마운트 |
| 이미지, 영상, 파일 | Object Storage (S3) + CDN | 무한 확장, 저렴, CDN 연동 쉬움 |
| 공유 파일 시스템 | File Storage (EFS) | 여러 서버 동시 접근 |
| 장기 백업 | S3 Glacier | 매우 저렴, 복원 지연 허용 |
| 대용량 업로드 | S3 Multipart Upload | 청크별 재시도, 병렬 처리 |

**이미지 저장 절대 원칙**:
```
DB에 이미지 저장 → 절대 금지
서버 로컬 디스크 → 수평 확장 시 금지
Object Storage (S3) → 표준 방법
```

---

## 🤔 심화 질문

**Q1. S3에 저장된 파일의 접근 권한은 어떻게 관리하는가?**
> 기본적으로 S3 버킷은 Private입니다. 공개 이미지(프로필, 상품 이미지)는 퍼블릭 읽기를 허용하거나 CloudFront를 통해서만 접근하도록 설정합니다. 개인 파일(계약서, 의료 기록)은 Presigned URL로 임시 접근 권한을 발급합니다(유효 시간 5분~1시간). 절대 S3 버킷 전체를 퍼블릭으로 열지 않습니다.

**Q2. 이미지 리사이징은 어디서 하는가?**
> 업로드 시 비동기로 처리하는 방법(Lambda + S3 Event)과 요청 시 온디맨드로 처리하는 방법(CloudFront + Lambda@Edge) 두 가지가 있습니다. 온디맨드 방법이 더 유연합니다: 원본은 S3에 한 번만 저장하고, CloudFront 요청 시 Lambda@Edge가 URL 파라미터(`?w=300&h=300`)를 파싱해 리사이징 후 캐싱합니다.

**Q3. 면접에서 "수백만 명의 사용자가 사진을 업로드하는 서비스를 설계하라"고 하면?**
> "업로드는 Presigned URL 방식으로 App Server를 우회해 S3에 직접 업로드합니다. 업로드 완료 후 S3 이벤트가 Kafka를 통해 썸네일 생성 Worker를 트리거합니다. 원본과 다양한 크기의 썸네일(300px, 600px, 1200px)을 S3에 저장하고, CloudFront CDN으로 전 세계에 서빙합니다. 오래된 사진은 S3 Lifecycle Policy로 Glacier로 이동해 비용을 절감합니다."

---

<div align="center">

[⬅️ 이전: 검색 인프라](./06-search-infrastructure.md) | [README로 돌아가기](../README.md) | [다음 챕터: Basic Services ➡️](../basic-services/01-url-shortener.md)

</div>
