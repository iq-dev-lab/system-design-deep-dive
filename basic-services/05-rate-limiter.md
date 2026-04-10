# Rate Limiter 설계

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 초과 요청을 차단하거나 지연
- 사용자별, IP별, API 키별 다양한 제한 기준
- 분산 환경에서 여러 서버가 공유하는 카운터

### 비기능 요구사항
- 매우 낮은 지연시간 (Rate Limit 체크 자체가 응답을 느리게 하면 안 됨)
- 정확성 (허용치 초과 요청이 통과되거나 허용치 내 요청이 차단되지 않아야 함)
- 고가용성 (Rate Limiter 다운 시 서비스도 다운되면 안 됨)

---

## 📊 규모 산정

```
초당 API 요청: 100만 req/s
Rate Limit 체크 지연시간 목표: 1ms 이하

→ Redis를 통한 카운터 관리가 유일한 현실적 방법
→ 각 요청마다 Redis 조회 1회 (in-memory이므로 1ms 이하 가능)
```

---

## 🏗️ 알고리즘 비교

### 1. Token Bucket (토큰 버킷)

```
개념: 버킷에 일정 속도로 토큰이 채워지고, 요청마다 토큰 1개 소비

버킷: 용량 10 (최대 버스트)
채움 속도: 2개/초

t=0s: [●●●●●●●●●●] 10개
t=1s: 요청 3개 소비 → [●●●●●●●●] (빈칸 2개) → 채움 +2 → 10개까지만 = 10
       실제: 10 - 3 = 7개 + 채움 2개 = 9개

요청 처리:
  토큰 있음 → 처리 + 토큰 -1
  토큰 없음 → 429 Too Many Requests

장점:
  ✅ 순간적 버스트 허용 (버킷에 남은 토큰만큼)
  ✅ 평균 속도 제어 가능
  ✅ 메모리 효율적 (버킷 1개만 관리)

단점:
  ❌ 분산 환경에서 버킷 상태 동기화 필요
  ❌ 파라미터 2개 (버킷 크기, 채움 속도) 설정 어려울 수 있음

적합: API 속도 제한, 네트워크 트래픽 쉐이핑
```

### 2. Leaky Bucket (누출 버킷)

```
개념: 큐에 요청을 담고 일정한 속도로 처리 (큐가 꽉 차면 거부)

큐: 최대 크기 10
처리 속도: 2 req/s (고정)

   입력 (가변)
   │ ││││││
   ▼ ▼▼▼▼▼▼
  [●●●●●●●●●●] 큐 (최대 10)
        │
        │ 2 req/s (고정 출력)
        ▼
    처리

장점:
  ✅ 출력이 항상 일정한 속도 → 서버에 균등한 부하
  ✅ 구현 단순

단점:
  ❌ 순간 버스트를 전혀 허용 안 함 (항상 2req/s만 처리)
  ❌ 큐가 오래된 요청으로 가득 차면 최신 요청이 항상 거부됨

적합: 결제 처리 등 일정한 속도가 중요한 경우
```

### 3. Fixed Window Counter (고정 윈도우 카운터)

```
개념: 시간 창(예: 1분)마다 카운터를 리셋

1분당 10회 제한:

  [11:00 ~ 11:01]  요청 10개 → 허용
                   요청 11번째 → 거부

  [11:01 ~ 11:02]  카운터 리셋 → 요청 10개 → 허용

구현:
  key = f"ratelimit:{user_id}:{minute}"
  count = Redis.INCR(key)
  Redis.EXPIRE(key, 60)
  if count > 10:
      return 429

단점:
  ❌ 윈도우 경계에서 순간 2배 요청 허용

  예:
  11:00:59에 10개 → 허용
  11:01:00에 10개 → 허용 (새 윈도우)
  → 2초 사이에 20개 요청이 통과됨!
```

### 4. Sliding Window Log (슬라이딩 윈도우 로그)

```
개념: 현재 시각에서 정확히 N초 전까지의 요청만 카운트

1분당 10회 제한:

11:01:30에 요청 도착:
  Redis ZSET에서 타임스탬프 11:00:30 ~ 11:01:30 범위 카운트
  → 이 범위 내 요청이 10개 미만이면 허용

Redis 구현:
  ZADD ratelimit:{user_id} {timestamp} {request_id}
  ZREMRANGEBYSCORE ratelimit:{user_id} 0 {now - 60}
  count = ZCARD ratelimit:{user_id}
  if count >= 10:
      return 429

장점:
  ✅ Fixed Window의 경계 문제 없음
  ✅ 정확한 1분 슬라이딩 윈도우

단점:
  ❌ 요청마다 타임스탬프 저장 → 메모리 사용량 높음
  ❌ 트래픽 많으면 Redis 메모리 급증
```

### 5. Sliding Window Counter (슬라이딩 윈도우 카운터) — 권장

```
Fixed Window + Sliding Window의 절충안:

현재 윈도우 카운터 + 이전 윈도우 카운터의 가중치 합산

예: 1분당 10회 제한
  이전 윈도우: 8회
  현재 윈도우: 3회
  현재 윈도우 내 경과 시간: 30% (현재 분의 30초 지점)

  추정 요청 수 = 현재 윈도우 3회 + 이전 윈도우 8회 × (1 - 0.3)
              = 3 + 5.6 = 8.6회 → 10회 미만 → 허용

장점:
  ✅ 메모리 효율적 (윈도우 2개만 저장)
  ✅ Fixed Window 경계 문제 해결
  ✅ 대부분의 실제 서비스에서 오차 0.003% 수준

적합: 대부분의 API Rate Limiting
```

---

## 🔬 핵심 컴포넌트 — 분산 환경 구현

### Redis + Lua Script로 원자적 처리

```lua
-- Sliding Window Counter Lua Script
-- 여러 Redis 명령어를 원자적으로 실행 (Race Condition 방지)

local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])  -- 초 단위
local now = tonumber(ARGV[3])     -- 현재 타임스탬프 (초)

-- 현재 윈도우 시작 시각
local window_start = math.floor(now / window) * window
local current_key = key .. ":" .. window_start
local prev_key = key .. ":" .. (window_start - window)

-- 현재 + 이전 윈도우 카운터 조회
local current_count = tonumber(redis.call('GET', current_key) or 0)
local prev_count = tonumber(redis.call('GET', prev_key) or 0)

-- 윈도우 내 경과 비율
local elapsed_ratio = (now % window) / window

-- 추정 요청 수
local estimated = current_count + prev_count * (1 - elapsed_ratio)

if estimated >= limit then
    return 0  -- 거부
else
    redis.call('INCR', current_key)
    redis.call('EXPIRE', current_key, window * 2)
    return 1  -- 허용
end
```

### API Gateway 통합 구성

```
클라이언트 요청
      │
      ▼
┌─────────────────────────────────┐
│  API Gateway (Rate Limiter)     │
│                                 │
│  1. 요청 식별자 추출             │
│     - API Key (헤더)            │
│     - User ID (JWT에서)         │
│     - IP 주소 (X-Forwarded-For) │
│                                 │
│  2. Redis에서 카운터 조회/증가   │
│     (Lua Script로 원자적 처리)  │
│                                 │
│  3. 한도 초과?                  │
│     YES → 429 + Retry-After 헤더│
│     NO  → 백엔드로 전달         │
└─────────────────────────────────┘
      │
      ▼
  백엔드 API 서버
```

### Rate Limit 응답 헤더

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100          (총 허용 수)
X-RateLimit-Remaining: 73       (남은 허용 수)
X-RateLimit-Reset: 1704067260   (리셋 시각, Unix Timestamp)

HTTP/1.1 429 Too Many Requests
Retry-After: 30                  (30초 후 재시도)
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
```

---

## ⚖️ 트레이드오프

| 상황 | 알고리즘 | 이유 |
|------|----------|------|
| 일반 API | Sliding Window Counter | 정확성 + 메모리 효율 균형 |
| 결제, 이체 | Fixed Window (엄격) | 구현 단순, 정확한 윈도우 |
| 실시간 스트리밍 | Token Bucket | 버스트 허용, 평균 속도 제어 |
| 일정한 출력 필요 | Leaky Bucket | 균등한 서버 부하 |

---

## 🚀 확장 전략

```
Redis 단일 서버 장애 시:
  → Rate Limiter를 통과한 요청이 백엔드에 몰릴 수 있음
  해결: Redis Sentinel(자동 장애 조치) 또는 Redis Cluster
  폴백: Redis 장애 시 Rate Limiting 임시 비활성화 (서비스 유지 우선)

초고처리량 (초당 1억 요청):
  → Redis도 병목 가능
  해결: 로컬 캐시(L1) + Redis(L2) 2단계
        각 서버가 로컬에서 대략적 카운팅
        N초마다 Redis와 동기화
        정확도가 약간 떨어지지만 Redis 부하 대폭 감소
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 알고리즘: Sliding Window Counter (정확성 + 효율 균형)
  2. 저장: Redis (in-memory, 초당 수십만 읽기/쓰기)
  3. 원자성: Lua Script (Race Condition 방지)
  4. 위치: API Gateway (서비스 코드 변경 없이 중앙 관리)
  5. 응답: 429 + Retry-After 헤더 (클라이언트 재시도 가이드)
```

---

## 🤔 심화 질문

**Q1. 분산 Rate Limiter에서 카운터가 정확하지 않을 수 있는 이유는?**
> 요청이 여러 서버에 라운드 로빈으로 분산되고, 각 서버가 Redis의 카운터를 1ms 이내로 증가시키더라도, 네트워크 지연으로 두 서버가 동시에 카운터를 읽고 둘 다 허용할 수 있습니다. Lua Script로 INCR과 GET을 원자적으로 처리하면 이 문제를 해결합니다.

**Q2. Rate Limiter를 API Gateway에 두는 것과 서비스 코드에 두는 것의 차이는?**
> API Gateway에 두면 서비스 코드 변경 없이 모든 API에 일괄 적용 가능하고, 여러 서비스에 공통 정책을 쉽게 적용합니다. 하지만 서비스 레벨의 세부 비즈니스 로직 기반 제한(예: "프리미엄 유저는 더 많이")은 서비스 코드 레벨에서 처리해야 합니다. 두 레벨을 조합하는 것이 실용적입니다.

**Q3. DDoS 공격에도 Rate Limiter가 유효한가?**
> 기본적인 Rate Limiting은 IP 기반으로 동작하므로 수천 개의 IP를 쓰는 분산 DDoS에는 효과가 제한적입니다. DDoS 방어는 Cloudflare, AWS Shield 같은 전문 DDoS 보호 서비스를 앞단에 배치하는 것이 핵심입니다. Rate Limiter는 일반 사용자의 과도한 사용과 단순 공격을 막는 첫 번째 방어선입니다.

---

<div align="center">

[⬅️ 이전: 분산 ID 생성기](./04-distributed-id.md) | [README로 돌아가기](../README.md) | [다음: 알림 시스템 ➡️](./06-notification-system.md)

</div>
