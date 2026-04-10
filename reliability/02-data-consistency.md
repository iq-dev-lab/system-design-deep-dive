# 데이터 일관성 (Saga / 분산 트랜잭션)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 마이크로서비스 간 트랜잭션 (주문 → 결제 → 재고 → 배송)
- 일부 단계 실패 시 롤백 (보상 트랜잭션)
- 분산 환경에서 데이터 일관성 유지
- 중복 처리 방지 (멱등성)

### 비기능 요구사항
- 전체 트랜잭션이 실패하면 완전 롤백
- 분산 서비스 간 강한 결합 없음 (독립 배포 가능)
- 부분 완료 상태로 오래 남지 않음 (빠른 보상)

---

## 📊 분산 트랜잭션의 문제

```
전통적 ACID 트랜잭션 (단일 DB):
  BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
  UPDATE orders SET status = 'PAID' WHERE order_id = 1;
  COMMIT;  -- 성공 or 전부 롤백

마이크로서비스에서는?
  주문 서비스 (Order DB)
  결제 서비스 (Payment DB)
  재고 서비스 (Inventory DB)
  → 각각 별도 DB! → 단일 트랜잭션 불가

2PC (Two-Phase Commit) 문제:
  Phase 1: 코디네이터가 모든 서비스에 "준비됐어?" 질문
  Phase 2: 모두 준비 → "커밋해!" / 하나라도 실패 → "롤백해!"
  
  문제:
  ❌ 코디네이터 장애 시 → 참여자들이 잠금 상태로 블로킹
  ❌ 성능 저하 (모든 참여자 동기 대기)
  ❌ 마이크로서비스 독립성 위배 (강한 결합)
```

---

## 🏗️ Saga 패턴

```
아이디어: 긴 트랜잭션을 여러 로컬 트랜잭션으로 분리
          각 단계 실패 시 이전 단계를 되돌리는 "보상 트랜잭션" 실행

이커머스 주문 Saga:

정방향 트랜잭션:
  1. 주문 생성 (Order Service) → status='PENDING'
  2. 결제 처리 (Payment Service) → 카드 청구
  3. 재고 감소 (Inventory Service) → 재고 -1
  4. 배송 시작 (Shipping Service) → 배송 등록
  5. 주문 완료 (Order Service) → status='COMPLETED'

보상 트랜잭션 (단계 3 실패 시):
  3. 재고 감소 실패
  ← 2. 결제 취소 (Payment Service) → 환불
  ← 1. 주문 취소 (Order Service) → status='CANCELLED'
```

---

## 🔬 핵심 컴포넌트 — Saga 구현 방식

### 방법 1: Choreography (이벤트 기반)

```
각 서비스가 이벤트를 통해 자율적으로 다음 단계 실행

이벤트 흐름:

Order Service
  │ ORDER_CREATED 이벤트 발행
  ▼
Payment Service (구독)
  │ 결제 성공 → PAYMENT_PROCESSED 이벤트 발행
  │ 결제 실패 → PAYMENT_FAILED 이벤트 발행 → Order Service가 취소
  ▼
Inventory Service (구독)
  │ 재고 감소 성공 → INVENTORY_RESERVED 이벤트 발행
  │ 재고 없음 → INVENTORY_FAILED → Payment Service가 환불
  ▼
Shipping Service (구독)
  │ 배송 등록 → SHIPMENT_STARTED 이벤트 발행
  ▼
Order Service (구독)
  │ ORDER_COMPLETED 상태 업데이트

장점: 서비스 간 직접 의존 없음, 확장성 좋음
단점: 전체 트랜잭션 흐름 파악 어려움, 이벤트 체인 디버깅 복잡

보상 트랜잭션 예시 (INVENTORY_FAILED):
  INVENTORY_FAILED 이벤트 발행
  Payment Service: PAYMENT_REVERSED (환불) 이벤트 발행
  Order Service: ORDER_CANCELLED 상태 변경
```

### 방법 2: Orchestration (중앙 조율)

```
Saga Orchestrator가 전체 흐름 제어

Saga Orchestrator
  │ (1) "결제해" → Payment Service
  │ (2) 결제 성공 응답 수신
  │ (3) "재고 감소해" → Inventory Service
  │ (4) 재고 없음 응답 수신
  │ (5) "결제 취소해" → Payment Service (보상)
  │ (6) "주문 취소해" → Order Service (보상)

장점: 전체 흐름이 한 곳에서 관리됨 (디버깅 쉬움)
단점: Orchestrator가 단일 장애점, 모든 서비스에 의존

구현 (상태 머신):
  class OrderSaga:
      def execute(self, order_id):
          try:
              payment_id = self.payment_svc.charge(order_id)
              self.save_state(order_id, "PAYMENT_DONE", payment_id)
              
              self.inventory_svc.reserve(order_id)
              self.save_state(order_id, "INVENTORY_RESERVED")
              
              self.shipping_svc.schedule(order_id)
              self.save_state(order_id, "COMPLETED")
          
          except InventoryError:
              # 보상 트랜잭션
              self.payment_svc.refund(payment_id)
              self.order_svc.cancel(order_id)
```

---

## 🔄 멱등성 (Idempotency)

```
문제: 네트워크 실패로 같은 요청을 두 번 보낼 때
  결제 요청 → 서버 처리 중 → 클라이언트 타임아웃 → 재시도
  → 결제가 두 번 처리될 수 있음!

해결: 멱등성 키 (Idempotency Key)

클라이언트:
  POST /payments
  X-Idempotency-Key: unique_request_id_abc123
  { "amount": 50000, "user_id": 1001 }

서버:
  1. Redis: SETNX "idempotency:abc123" "processing" EX=3600
  2. 성공 → 결과 저장
  3. 같은 키로 재요청 → Redis에서 이전 결과 반환 (DB 재처리 없음)

Redis 저장:
  "idempotency:abc123" → {status: "SUCCESS", payment_id: "pay_999"}

동일 키 재요청:
  → Redis에서 조회 → {status: "SUCCESS", payment_id: "pay_999"} 반환
  → 결제 중복 없음!

멱등한 연산 vs 아닌 연산:
  ✅ GET, PUT (동일 값으로): 멱등
  ❌ POST (새 리소스 생성), INCR: 멱등 아님
  → 멱등하지 않은 연산에 반드시 멱등성 키 적용
```

---

## ⚡ 트랜잭션 아웃박스 패턴 (Outbox Pattern)

```
문제: DB 업데이트 + 이벤트 발행이 원자적이지 않음

시나리오:
  1. DB에 주문 저장 (성공)
  2. Kafka에 이벤트 발행 (실패!)
  → 주문은 생겼지만 결제 서비스가 모름 → 데이터 불일치

Outbox Pattern 해결:

1. DB 트랜잭션 내에서:
   a. orders 테이블에 주문 저장
   b. outbox 테이블에 이벤트 저장 (같은 트랜잭션)
   → 원자적! 둘 다 성공하거나 둘 다 실패

outbox 테이블:
  id, event_type, payload, created_at, published

2. Outbox Relay (별도 프로세스):
   outbox 테이블 폴링 → published=false인 이벤트
   → Kafka에 발행
   → published=true 업데이트

또는 CDC (Change Data Capture):
  Debezium이 outbox 테이블 변경 감지 → Kafka 자동 발행
  → 폴링 없이 이벤트 기반으로 처리
```

---

## ⚖️ 트레이드오프

| 방식 | 일관성 | 가용성 | 복잡도 | 선택 시기 |
|------|--------|--------|--------|----------|
| 2PC | 강한 일관성 | 낮음 (블로킹) | 높음 | 레거시 시스템, 필수 시 |
| Saga Choreography | 최종 일관성 | 높음 | 중간 | 소수 서비스, 단순 흐름 |
| Saga Orchestration | 최종 일관성 | 높음 | 낮음 | 복잡한 흐름, 중앙 관리 선호 |

---

## 🚀 확장 전략

```
Saga 라이브러리:
  Axon Framework (Java): Saga + Event Sourcing 통합
  Eventuate Tram: Choreography 기반 Saga 프레임워크
  Conductor (Netflix): 워크플로 오케스트레이션

보상 트랜잭션 한계:
  일부 보상이 불가능한 경우:
  이메일 발송, SMS 전송 → 이미 전달됨 → 취소 불가
  → "취소 이메일"을 추가로 발송하는 방식으로 처리

타임아웃 처리:
  Saga 단계마다 타임아웃 설정
  타임아웃 초과 → 해당 단계 실패 처리 → 보상 시작
  고아 트랜잭션 방지 (영원히 PENDING 상태 남지 않음)
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 2PC 대신 Saga: 마이크로서비스 간 분산 트랜잭션
  2. Choreography vs Orchestration: 복잡도에 따라 선택
  3. 멱등성 키: 중복 처리 방지 (결제, 예약 등 필수)
  4. Outbox Pattern: DB 업데이트 + 이벤트 발행 원자성 보장
  5. 보상 트랜잭션: 각 단계마다 롤백 로직 사전 설계
```

---

## 🤔 심화 질문

**Q1. Saga 패턴에서 "주문은 완료됐지만 환불이 실패"하면 어떻게 하는가?**
> 보상 트랜잭션 자체가 실패하는 시나리오입니다. 이 경우 DLQ(Dead Letter Queue)에 실패한 보상 이벤트를 저장하고, 운영팀에 알림을 보내 수동 처리합니다. 또는 보상 트랜잭션을 재시도 가능하도록 멱등하게 설계합니다(환불 요청에 환불 ID 포함). 이런 경우를 위해 모든 Saga 상태를 DB에 저장해 어느 단계에서 실패했는지 추적합니다.

**Q2. Event Sourcing과 Saga의 차이는?**
> Event Sourcing은 상태를 이벤트의 시퀀스로 저장하는 패턴입니다(현재 잔액 대신 입금/출금 이벤트 목록 저장). Saga는 분산 트랜잭션 패턴입니다. 두 패턴은 함께 사용하면 시너지가 납니다. Saga의 보상 이벤트를 Event Store에 저장하면 전체 트랜잭션 이력을 감사(Audit)할 수 있습니다.

**Q3. 분산 시스템에서 "정확히 한 번(Exactly-Once)" 처리가 얼마나 어려운가?**
> 이론적으로 불가능에 가깝습니다(두 장군 문제). 실제로는 At-Least-Once + 멱등성으로 구현합니다. Kafka는 Producer 멱등성(같은 메시지 중복 발행 방지)과 Consumer의 멱등한 처리(동일 메시지 여러 번 처리해도 결과 동일)를 조합해 Exactly-Once 시맨틱을 제공합니다. 하지만 DB 업데이트와 Kafka 발행의 엄밀한 Exactly-Once는 Transactional Outbox + 멱등 Consumer로 구현합니다.

---

<div align="center">

[⬅️ 이전: 장애 허용 설계](./01-fault-tolerance.md) | [README로 돌아가기](../README.md) | [다음: 모니터링과 알림 ➡️](./03-monitoring.md)

</div>
