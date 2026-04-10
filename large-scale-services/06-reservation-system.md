# 예약 시스템 (Ticketmaster / 항공권 예약)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 좌석/티켓 조회 (잔여 수량 실시간 확인)
- 예약 및 결제
- 예약 취소 및 환불
- 인기 공연 오픈 시 동시 수만 명 예약 처리

### 비기능 요구사항
- DAU 100만 명 (평소), 피크 10만 명 동시 접속 (인기 공연 오픈)
- 이중 예약 절대 방지 (같은 좌석 2명에게 판매 불가)
- 결제 완료 후 예약 확정까지 5초 이내
- 예약 임시 점유 (10분 내 결제 미완료 → 자동 반환)

---

## 📊 규모 산정

```
평상시:
  초당 예약 요청: 100 TPS
  초당 조회: 5,000 QPS

피크 (콘서트 오픈):
  10만 명 동시 → 티켓 1,000장
  → 초당 수천 건의 예약 시도, 1,000명만 성공
  → 나머지 9만9천 명 → 재고 없음 처리

핵심 문제:
  1. 재고 정확성: 1,000장을 정확히 1,001번째부터 막아야 함
  2. 동시성: 수천 명이 동시에 같은 좌석 시도
  3. 확장성: 평소 100 TPS → 피크 10,000 TPS
```

---

## 🏗️ 개략적 설계

```
사용자
  │ (1) 좌석 선택
  ▼
┌────────────────┐
│  API 서버      │
│  (예약 서버)   │
└───────┬────────┘
        │ (2) 재고 감소 시도 (원자적)
        ▼
┌────────────────┐        ┌──────────────────┐
│  Redis         │        │  DB (예약 테이블) │
│  (재고 카운터)  │───────▶│  (영속성)        │
│  DECR 원자적   │        └──────────────────┘
└───────┬────────┘
        │ (3) 재고 성공 → 임시 예약 생성
        ▼
┌────────────────┐
│  임시 예약     │  TTL=10분 (Redis)
│  점유 상태     │  결제 완료 → 확정
└───────┬────────┘  결제 미완료 → 자동 반환
        │
        ▼
┌────────────────┐
│  결제 서버     │  외부 PG사 연동
│                │  결제 성공 → DB 예약 확정
└────────────────┘
```

---

## 🔬 핵심 컴포넌트 — 재고 동시성 제어

### 방법 1: DB 낙관적 잠금 (Optimistic Lock)

```sql
-- 버전 기반 낙관적 잠금
CREATE TABLE seats (
    seat_id     BIGINT PRIMARY KEY,
    event_id    BIGINT,
    status      ENUM('AVAILABLE', 'HELD', 'BOOKED'),
    version     INT DEFAULT 0,    -- 낙관적 잠금 버전
    held_by     BIGINT,           -- 임시 점유 사용자
    held_until  TIMESTAMP         -- 점유 만료 시각
);

-- 예약 시도:
UPDATE seats
SET status='HELD', held_by=?, held_until=NOW()+600,
    version = version + 1
WHERE seat_id = ?
  AND status = 'AVAILABLE'
  AND version = ?;   -- 읽어온 버전과 같아야만 성공

-- 영향받은 행 = 0 → 다른 사용자가 먼저 예약 → 재시도 또는 실패
-- 영향받은 행 = 1 → 예약 성공
```

### 방법 2: Redis 원자적 재고 감소 (권장)

```
재고 초기화 (공연 오픈 전):
  SET "inventory:{event_id}" 1000  (1000장)

예약 시도 (Lua Script - 원자적):
  local current = redis.call('GET', KEYS[1])
  if tonumber(current) <= 0 then
    return 0  -- 재고 없음
  end
  redis.call('DECR', KEYS[1])
  return 1  -- 재고 감소 성공

  → DECR은 원자적 → Race Condition 없음
  → 1000명만 정확히 통과

성공 시:
  임시 예약 생성 (DB에 HELD 상태로 저장)
  Redis SET "held:{user_id}:{event_id}" seat_id EX=600

10분 내 결제:
  결제 성공 → DB UPDATE status='BOOKED'
  결제 실패/타임아웃 → Redis에서 자동 만료 → 재고 +1 (INCR)
```

### 대기열 (Queue) 시스템

```
문제: 10만 명이 동시에 예약 버튼 클릭 → 서버 과부하

해결: 대기열 도입

사용자 → 대기열 진입 (Kafka 또는 Redis Queue)
           │
           │ (순서대로 처리)
           ▼
      예약 Worker (초당 N명 처리)
           │
           ▼
      예약 성공/실패 응답 (WebSocket 또는 Polling)

대기열 UI:
  "현재 대기 순번: 5,432번 (예상 대기: 12분)"
  
대기열 구현:
  Redis Sorted Set: ZADD "waitlist:{event_id}" {timestamp} {user_id}
  순번 조회: ZRANK "waitlist:{event_id}" user_id
  처리: Worker가 ZPOPMIN으로 순서대로 꺼내 처리
```

### 이중 예약 방지

```
시나리오: 같은 좌석 A를 User1, User2가 동시에 시도

방법 1: DB 유니크 제약
  CREATE UNIQUE INDEX (event_id, seat_id)
  ON reservations (event_id, seat_id)
  WHERE status != 'CANCELLED';
  → 두 번째 INSERT → 유니크 제약 위반 → 자동 실패

방법 2: Redis 분산 잠금 (Redlock)
  SETNX "lock:seat:{seat_id}" user_id EX=30
  → 첫 번째 사용자만 성공 (원자적)
  → 두 번째 사용자 → 이미 잠금됨 → 실패 또는 대기

방법 3: 데이터베이스 SELECT FOR UPDATE
  BEGIN;
  SELECT * FROM seats WHERE seat_id=? AND status='AVAILABLE' FOR UPDATE;
  -- 이 행에 락 → 다른 트랜잭션 대기
  UPDATE seats SET status='HELD' WHERE seat_id=?;
  COMMIT;
  
  단점: 많은 동시 요청 시 DB 락 경합 → 병목
  적합: 트래픽이 낮은 시스템
```

---

## 🔄 데이터 흐름 — 결제 실패 시 롤백

```
사용자 결제 진행 중 실패 시나리오:

1. 재고 감소: Redis DECR (성공, 재고 999)
2. DB HELD 상태 저장 (성공)
3. 결제 요청 → PG사 타임아웃 (실패)

롤백:
  a. DB: status='CANCELLED' 업데이트
  b. Redis: INCR 재고 (1000으로 복원)
  c. 대기열 사용자에게 예약 기회 제공

10분 TTL 만료 자동 처리:
  Redis Keyspace Notification 활용:
  "held:{user_id}:{event_id}" 키 만료 이벤트 감지
  → Consumer가 재고 INCR 자동 실행
  → 대기열 다음 순번 처리
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| 재고 관리 | Redis DECR (원자적) | 수천 TPS 처리, Race Condition 없음 |
| 이중 예약 방지 | DB 유니크 제약 + Redis 락 | 이중 안전망 |
| 피크 트래픽 | 대기열 (Queue) | 서버 보호, 공정한 처리 |
| 임시 점유 | Redis TTL 10분 | 자동 만료, 재고 자동 반환 |

---

## 🚀 확장 전략

```
이벤트 샤딩:
  인기 공연 → 전용 예약 서버 (격리)
  일반 공연 → 공유 서버 풀

DB 분리:
  읽기 (잔여석 조회): Read Replica
  쓰기 (예약 확정): Primary

결제 처리:
  PG사 타임아웃 → 재시도 (멱등성 키 활용)
  결제 결과 Webhook → 비동기 처리
  Circuit Breaker (PG사 장애 시 대체 PG사로 전환)
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 재고: Redis DECR 원자적 감소 (이중 예약 방지)
  2. 임시 점유: Redis TTL 10분 (결제 미완료 자동 반환)
  3. 피크 처리: 대기열 시스템 (서버 보호 + 공정성)
  4. 이중 예약 방지: DB 유니크 제약 (최후 안전망)
  5. 결제 롤백: Redis INCR + DB 상태 변경 (Saga 패턴)
```

---

## 🤔 심화 질문

**Q1. 항공권 예약에서 "가격이 실시간으로 변하는" 것을 어떻게 구현하는가?**
> 항공권 가격은 잔여 좌석 수, 예약 시점, 경쟁 항공사 가격, 과거 수요 패턴 등을 입력으로 Dynamic Pricing 알고리즘이 결정합니다. 조회 시마다 가격 계산 서비스를 호출하거나, 주기적으로 계산된 가격을 Redis에 캐시합니다. 예약 완료 시점의 가격으로 결제하며, 조회와 결제 사이 가격 변동은 재확인 후 진행합니다.

**Q2. 인기 공연에서 봇(Bot)이 대량 예약하는 것을 어떻게 막는가?**
> CAPTCHA로 봇 감지, 1인당 구매 한도(예: 최대 4장), 동일 IP/신용카드에서의 반복 시도 차단, 예약 전 인증 요구(로그인, 본인인증), 구매 패턴 ML 이상 감지 등을 조합합니다. Ticketmaster는 Verified Fan 프로그램으로 사전 등록한 실제 팬에게 우선권을 부여합니다.

**Q3. 예약 시스템에서 Saga 패턴이 필요한 이유는?**
> 예약 프로세스가 여러 서비스에 걸쳐 있습니다(재고 서비스, 결제 서비스, 알림 서비스). 결제 실패 시 재고를 복원해야 하는데, 분산 트랜잭션(2PC)은 복잡하고 느립니다. Saga 패턴은 각 단계의 보상 트랜잭션(compensating transaction)을 정의합니다. 결제 실패 → "재고 반환" 보상 트랜잭션 실행 → 시스템 일관성 복원.

---

<div align="center">

[⬅️ 이전: 검색 엔진](./05-search-engine.md) | [README로 돌아가기](../README.md) | [다음: 위치 기반 서비스 ➡️](./07-location-service.md)

</div>
