# URL 단축 서비스 설계 (bit.ly)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 긴 URL을 짧은 URL로 변환 (`https://example.com/very/long/path` → `https://short.ly/aBc3K`)
- 짧은 URL 접속 시 원본 URL로 리디렉션
- 커스텀 단축 키 설정 지원 (`short.ly/mylink`)
- URL 만료 기간 설정 가능
- 클릭 통계 제공 (선택)

### 비기능 요구사항
- 고가용성 (리디렉션은 최대 성능 필요)
- 낮은 지연시간 (리디렉션 < 10ms 목표)
- 단축 URL은 예측 불가능해야 함 (보안)

---

## 📊 규모 산정

```
DAU: 100만 명
읽기(리디렉션) : 쓰기(URL 생성) = 10 : 1

쓰기:
  일일 URL 생성 = 100만 × 0.2회 = 20만 건/일
  초당 쓰기 (QPS) = 200,000 / 86,400 ≈ 2.3 → 3 req/s
  피크 쓰기 (×5) = 15 req/s

읽기:
  일일 리디렉션 = 20만 × 10 = 200만 건/일
  초당 읽기 (QPS) = 2,000,000 / 86,400 ≈ 23 → 25 req/s
  피크 읽기 (×5) = 125 req/s

저장 용량 (5년):
  URL 1건당:
    단축 키: 7자 = 7 bytes
    원본 URL: 평균 200 bytes
    메타데이터 (만료일, 생성일, 사용자ID): 100 bytes
    합계: ≈ 310 bytes ≈ 400 bytes (여유)

  5년 URL 수:
  3 req/s × 86,400 × 365 × 5 = 4.7억 건

  저장 용량:
  4.7억 × 400 bytes ≈ 188GB → 약 200GB

캐시 (인기 URL 20%가 80% 트래픽):
  일일 읽기 200만 × 20% × 400 bytes ≈ 160MB

결론:
  QPS 낮음 → 단일 서버로 시작 가능
  저장: MySQL 200GB (단일 서버 충분)
  캐시: Redis 1GB면 충분
```

---

## 🏗️ 개략적 설계

```
         Client
           │
     ┌─────▼─────┐
     │ LB (ALB)  │
     └─────┬─────┘
           │
     ┌─────▼─────────────────────────────┐
     │  URL Shortener API                │
     │  POST /shorten   → 단축 URL 생성  │
     │  GET /{shortKey} → 리디렉션       │
     └─────┬─────────────────────────────┘
           │
     ┌─────▼─────┐     ┌─────────────┐
     │  Redis    │     │   MySQL     │
     │  (캐시)   │     │  (영구 저장) │
     └───────────┘     └─────────────┘
           │
     ┌─────▼─────┐
     │ Analytics │  (선택, 클릭 통계)
     │  Service  │
     └───────────┘
```

---

## 🔬 핵심 컴포넌트 상세 설계

### 단축 키 생성 전략

#### 방법 1: 랜덤 Base62 (권장)

```
문자셋: a-z, A-Z, 0-9 = 62자
7자 조합: 62^7 = 3.5조 개 (충분)

생성:
  import random, string
  def generate_key():
    chars = string.ascii_letters + string.digits  # 62자
    return ''.join(random.choices(chars, k=7))
    # 예: "aBc3K9m"

충돌 처리:
  key = generate_key()
  while DB.exists(key):     # 충돌 시 재시도
      key = generate_key()  # 충돌 확률 매우 낮음 (3.5조 중 4.7억 = 0.013%)
  DB.insert(key, original_url)

장점: 예측 불가능 (보안), 구현 단순
단점: 충돌 가능 (재시도 필요), 분산 환경에서 중복 체크 필요
```

#### 방법 2: MD5/SHA256 해시 + 앞 7자

```
hash = md5(original_url + timestamp)[:7]

문제: 다른 URL이 같은 해시 앞 7자 → 충돌
해결: 충돌 시 salt 추가 후 재해싱

장점: 같은 URL → 항상 같은 키 (중복 단축 방지)
단점: 긴 URL + 타임스탬프 → 같은 URL도 다른 키
```

#### 방법 3: Counter 기반 Base62 인코딩

```
DB Auto Increment ID → Base62 인코딩

ID=1      → "0000001"
ID=1000   → "000g8"
ID=1억    → "FXsk" (6자)

장점: 충돌 없음, 빠른 생성
단점: 순차적 → 예측 가능 (보안 취약), 단일 DB 의존
```

### DB 스키마

```sql
CREATE TABLE urls (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    short_key   VARCHAR(10) NOT NULL UNIQUE,  -- 인덱스 필수
    original_url TEXT NOT NULL,
    user_id     BIGINT,                       -- 생성자 (NULL = 비회원)
    expires_at  DATETIME,                     -- NULL = 만료 없음
    created_at  DATETIME DEFAULT NOW(),
    click_count BIGINT DEFAULT 0,             -- 통계 (선택)
    INDEX idx_short_key (short_key),          -- 리디렉션 조회 최적화
    INDEX idx_user_id (user_id)               -- 사용자별 URL 목록
);
```

---

## ⚡ 병목 식별과 해결

### 리디렉션 최적화 (핵심 경로)

```
리디렉션 흐름:
  GET /aBc3K9m
      │
  Redis GET "aBc3K9m" → 히트: 즉시 301/302 리디렉션 (< 1ms)
      │
  Cache Miss → MySQL SELECT WHERE short_key='aBc3K9m' (5~10ms)
      │
  Redis SET "aBc3K9m" original_url EX 3600 (1시간 캐시)
      │
  301/302 응답

HTTP 상태 코드 선택:
  301 (Permanent): 브라우저가 결과를 캐싱 → 서버 부하 최소
                   단점: 통계 측정 불가 (브라우저가 직접 이동)
  302 (Temporary): 매번 서버 거침 → 통계 측정 가능
                   단점: 서버 부하 더 많음
  
  통계가 필요하면 302, 아니면 301 사용
```

---

## 🔄 데이터 흐름

### URL 생성 흐름

```
POST /shorten
Body: { "url": "https://example.com/very/long/path", "expires_in": 86400 }

1. URL 유효성 검증 (형식, 악성 URL 블랙리스트 체크)
2. 이미 단축된 URL인지 확인 (선택적 캐시 조회)
3. 단축 키 생성 (랜덤 Base62)
4. DB INSERT (short_key, original_url, expires_at)
5. Redis SET (캐시 저장)
6. 응답: { "short_url": "https://short.ly/aBc3K9m" }
```

### 리디렉션 흐름

```
GET /aBc3K9m

1. Redis GET "aBc3K9m"
   ├── 히트: 만료일 확인 → 301/302 리디렉션
   └── 미스: MySQL SELECT short_key='aBc3K9m'
              ├── 없음: 404 반환
              ├── 만료: 410 Gone 반환
              └── 있음: Redis 캐시 저장 → 리디렉션

2. 통계 기록 (비동기, Kafka)
   → Analytics Service가 클릭 수 집계
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| 키 생성 | 랜덤 Base62 | 예측 불가 (보안), 충돌 확률 무시할 수준 |
| 캐시 전략 | Cache-Aside, TTL 1시간 | 인기 URL은 자주 조회됨, TTL로 자동 갱신 |
| 리디렉션 코드 | 302 | 통계 측정 가능 (서버 경유 강제) |
| DB | MySQL | 강한 일관성, 단축 키 UNIQUE 제약 |
| 만료 처리 | 조회 시 확인 + 배치 정리 | 즉시 삭제보다 효율적 |

---

## 🚀 확장 전략

```
현재 스케일: QPS 125 → 단일 서버 충분

10배 트래픽 (QPS 1,250):
  ├── Redis 캐시 히트율 높이기 (캐시 크기 증가)
  ├── API 서버 수평 확장 (Stateless이므로 쉬움)
  └── MySQL 읽기 복제본 1개 추가

100배 트래픽 (QPS 12,500):
  ├── Redis Cluster (메모리 분산)
  ├── MySQL 읽기 복제본 3개
  └── 단축 키 생성 서버 별도 분리 (분산 환경 충돌 방지 → Snowflake ID + Base62)

커스텀 URL 지원:
  ├── short_key에 UNIQUE 제약으로 중복 방지
  └── 예약어 블랙리스트 (api, admin, static 등)
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 단축 키: 랜덤 Base62 7자 (3.5조 가지, 충돌 0.013% 이하)
  2. 리디렉션 경로: Redis 캐시 → MySQL (10ms 이하)
  3. DB: MySQL (UNIQUE 제약으로 중복 방지)
  4. 캐시: Redis (인기 URL 20%로 80% 트래픽 처리)
  5. 통계: Kafka로 비동기 처리 (리디렉션 지연 없음)
```

---

## 🤔 심화 질문

**Q1. 단축 URL 생성을 분산 환경에서 충돌 없이 처리하는 방법은?**
> 방법 1: DB UNIQUE 제약 + 충돌 시 재시도 (간단하지만 분산 락 필요). 방법 2: Snowflake ID(분산 고유 ID)를 Base62로 인코딩 → 서버 ID + 타임스탬프 + 시퀀스로 충돌 없는 키 생성. 방법 3: 키 범위를 서버별로 미리 할당(1~1000만은 서버A, 1000만~2000만은 서버B). → `basic-services/04-distributed-id.md` 참고

**Q2. 악성 URL(피싱, 멀웨어)을 어떻게 차단하는가?**
> Google Safe Browsing API로 URL 검증, 블랙리스트 DB 비교, 사용자 신고 시스템. 생성 시 검증 + 주기적으로 기존 URL도 재검증합니다.

**Q3. 면접에서 "같은 URL을 두 번 단축하면 같은 키를 반환하는가?"라고 하면?**
> "요구사항에 따라 다릅니다. 같은 키를 반환하려면 URL을 해시해서 캐시에 저장하고 조회합니다. 하지만 같은 URL이라도 사용자마다 다른 키를 원할 수 있습니다(통계 추적 분리). 제 설계에서는 같은 URL도 매번 새 키를 생성합니다. 기본 요구사항에서 명확히 않았으므로 면접관에게 확인하겠습니다."

---

<div align="center">

[⬅️ README로 돌아가기](../README.md) | [다음: 키-값 저장소 설계 ➡️](./02-key-value-store.md)

</div>
