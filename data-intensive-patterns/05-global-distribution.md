# 글로벌 분산 시스템

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 여러 지역(Region)에 데이터 복제
- 사용자가 가장 가까운 데이터센터에서 서비스 받음
- 지역 장애 시 자동 Failover
- 데이터 규정 준수 (GDPR: EU 데이터는 EU에만 저장)

### 비기능 요구사항
- 지역 간 지연: 서울↔도쿄 약 30ms, 서울↔미국 150ms
- 읽기: 로컬 리전에서 처리 (지연 < 50ms)
- 쓰기: 가용성 우선 (일관성 다소 포기 가능)
- RPO: 1분 이하 (장애 시 1분치 데이터 유실 허용)

---

## 📊 규모 산정

```
글로벌 서비스:
  리전: 5개 (서울, 도쿄, 싱가포르, 미국 동부, 유럽)
  DAU 5억 명 (리전별 1억 명)
  
리전 간 복제 트래픽:
  초당 쓰기: 50,000 TPS (리전당)
  복제본 수: 5 (모든 리전에 복제)
  → 크로스-리전 복제: 50,000 × 5 × 1KB = 250MB/s
  
RTT (Round-Trip Time):
  서울 ↔ 도쿄: 30ms (지역 데이터센터 위치에 따라 다름)
  서울 ↔ 싱가포르: 70ms
  서울 ↔ 미국 동부: 200ms
  서울 ↔ 유럽: 260ms
```

---

## 🏗️ 글로벌 아키텍처 패턴

### Active-Active vs Active-Passive

```
Active-Passive:
  Primary 리전 (서울): 모든 쓰기 처리
  Secondary 리전 (도쿄): 읽기만 처리 (복제본)
  
  서울 장애 → 도쿄로 Failover (수 분 소요)
  
  장점: 구현 단순, 데이터 충돌 없음
  단점: 미국 사용자가 서울까지 쓰기 → 높은 지연 (200ms)

Active-Active:
  모든 리전에서 읽기 + 쓰기 처리
  리전 간 양방향 복제
  
  장점: 지역별 낮은 지연 (미국 쓰기 → 미국 리전 즉시)
  단점: 충돌 해결 필요 (두 리전에서 동시 수정)
```

### 데이터 분류별 전략

```
전략 1: 지역 데이터 (User-Owned)
  해당 사용자 홈 리전에만 쓰기, 다른 리전에 읽기 복제
  
  사용자 A (서울 거주):
    → 모든 데이터가 서울 Primary에 쓰기
    → 도쿄/싱가포르는 읽기 복제본
    → 다른 나라 친구가 A의 프로필 조회 → 로컬 리전 복제본에서 읽기

전략 2: 글로벌 데이터 (공유)
  모든 리전에서 쓰기 허용 → 충돌 해결 필요
  예: 좋아요 수 (집계 값) → 정확성보다 가용성 우선 (CRDT 사용)

전략 3: 리전별 격리 (GDPR 준수)
  EU 사용자 데이터 → EU 리전에만 저장 (법적 요구)
  다른 리전으로 복제 금지
```

---

## 🔬 핵심 컴포넌트

### 다중 리전 DB (CockroachDB / Spanner)

```
Google Spanner:
  전 세계에 분산된 관계형 DB
  TrueTime API (GPS + 원자 시계)로 글로벌 트랜잭션 순서 보장
  강한 일관성 + 전 세계 분산 (CA-like, 실제로 CP에 가까움)
  
  쓰기 지연: 리전 간 복제로 50~100ms (일반 DB보다 높음)
  읽기 지연: 로컬 리전 5ms, 다른 리전은 오래된 버전 읽기 허용 시 5ms

CockroachDB:
  오픈소스 분산 SQL DB (Spanner 인스파이어)
  리전별 Primary 설정 가능 (지역 쓰기 지연 최소화)
  자동 Failover + 자동 Rebalancing

CockroachDB 리전 핀닝:
  CREATE TABLE users (
    id          BIGINT PRIMARY KEY,
    home_region STRING NOT NULL,
    ...
  ) LOCALITY REGIONAL BY ROW;
  
  -- home_region 기반으로 자동 가까운 리전에 데이터 배치
  INSERT INTO users (id, home_region, name)
  VALUES (1001, 'us-east', 'John');
  → 데이터가 us-east 리전의 Primary에 저장
```

### CRDT (Conflict-free Replicated Data Type)

```
문제: 두 리전에서 동시에 같은 데이터 수정 → 충돌

CRDT: 수학적으로 충돌 없이 병합 가능한 데이터 구조

G-Counter (증가만 하는 카운터):
  서울에서 좋아요 +5: {seoul: 5, tokyo: 0, us: 0}
  도쿄에서 좋아요 +3: {seoul: 0, tokyo: 3, us: 0}
  
  병합: {seoul: 5, tokyo: 3, us: 0}
  총 좋아요: max(5,0) + max(3,0) + max(0,0) = 8
  
  → 순서 상관없이 항상 같은 결과!

PN-Counter (증가/감소 카운터):
  P (Positive) Counter + N (Negative) Counter
  증가: P 카운터에 추가
  감소: N 카운터에 추가
  최종 값: P - N
  
  좋아요/좋아요 취소에 사용 가능

LWW-Register (Last Write Wins):
  각 업데이트에 타임스탬프 부여
  병합 시 가장 최신 타임스탬프 값 채택
  
  주의: NTP 시계 오차로 "최신"이 부정확할 수 있음
  Hybrid Logical Clock (HLC)으로 개선 가능
```

### 지역 라우팅 (GeoDNS)

```
글로벌 DNS 라우팅:

사용자 IP → DNS 조회 → 가장 가까운 리전의 IP 반환

서울 사용자:
  api.service.com → DNS → 10.1.1.1 (서울 IP)

미국 사용자:
  api.service.com → DNS → 10.2.2.2 (미국 동부 IP)

AWS Route 53 Latency-Based Routing:
  사용자 ← Route 53 → [서울 ALB, 도쿄 ALB, 미국 ALB, 유럽 ALB]
  RTT 기반으로 가장 빠른 리전 선택

Health Check 연동:
  서울 리전 장애 감지 → Route 53이 서울 제외
  도쿄로 자동 Failover (DNS TTL 30초 → 최대 30초 지연)
```

### 리전 간 복제 전략

```
비동기 복제 (권장, 가용성 우선):

서울 Primary → 쓰기 즉시 응답 → 비동기로 도쿄/미국에 복제
  
  장점: 쓰기 지연 낮음 (200ms 왕복 없음)
  단점: 서울 장애 시 복제 안 된 데이터 유실 (RPO > 0)

동기 복제 (일관성 우선):

서울 Primary → 도쿄 Replica 확인 대기 → 쓰기 응답
  
  장점: RPO = 0 (데이터 유실 없음)
  단점: 쓰기 지연 = 서울↔도쿄 RTT (30ms) 추가
  
  30ms 추가 지연이 허용되면 서울-도쿄는 동기, 미국은 비동기
  → 준동기(semi-synchronous) 복제

실무 (Netflix):
  같은 대륙 리전: 동기 복제 (지연 낮음)
  다른 대륙 리전: 비동기 복제 (지연 감수)
```

---

## 🔄 데이터 흐름 — 장애 처리

```
서울 리전 장애 시나리오:

[정상 상태]
클라이언트 (서울) → GeoDNS → 서울 API 서버 → 서울 DB

[서울 장애]
1. Health Check 실패 감지 (30초 이내)
2. Route 53이 서울 엔드포인트 제거
3. 다음 DNS TTL 만료 시 도쿄로 라우팅 전환

[도쿄 Failover 중]
서울 클라이언트 → GeoDNS → 도쿄 API 서버
도쿄 DB (서울 복제본) → 마지막 동기화 시점의 데이터

[고려 사항]
- 미복제 데이터 (RPO): 비동기 복제 시 수 초~분 유실 가능
- RTO (복구 시간): DNS TTL + 헬스체크 = 30~60초
- 서울 복구 후 데이터 동기화 (Reconciliation)
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| 복제 방식 | 비동기 (대부분) | 쓰기 지연 최소화, 가용성 우선 |
| 일관성 | 최종 일관성 | 글로벌 강한 일관성은 지연 비용 너무 높음 |
| 충돌 해결 | CRDT (집계) / LWW (레코드) | 자동 병합 가능 |
| 라우팅 | GeoDNS + Health Check | 사용자 위치 기반 최적 리전 |

---

## 🚀 확장 전략

```
리전 추가:
  새 리전 → 기존 리전에서 스냅샷 복제 → 동기화 완료 후 트래픽 인입
  GeoDNS에 새 리전 추가

데이터 레이크 글로벌 집계:
  각 리전의 이벤트 → 중앙 S3 (크로스-리전 복제)
  중앙 Spark/BigQuery로 글로벌 분석
  
규정 준수 (GDPR):
  EU 사용자 데이터 → EU 리전에만 저장
  EU 리전 외 복제 차단
  데이터 삭제 요청 → EU 리전의 해당 사용자 데이터 완전 삭제
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 리전별 격리: 사용자 데이터를 홈 리전에 Primary 배치
  2. 복제: 비동기 (가용성) + 같은 대륙 내 동기 (일관성)
  3. 충돌: CRDT (집계 데이터) / LWW (사용자 레코드)
  4. 라우팅: GeoDNS + Health Check Failover
  5. 규정: GDPR 대상 데이터 리전 격리 (EU only)
```

---

## 🤔 심화 질문

**Q1. 글로벌 서비스에서 "분산 트랜잭션"이 어려운 이유는?**
> 서울 DB와 미국 DB를 묶는 2PC(Two-Phase Commit) 트랜잭션은 코디네이터가 모든 참여자의 확인을 기다려야 합니다. 서울↔미국 RTT가 200ms이므로 트랜잭션 하나에 최소 400ms 지연이 발생합니다. 또한 코디네이터 장애 시 참여자들이 잠금 상태로 블로킹됩니다. Google Spanner는 TrueTime API로 이 문제를 해결했지만, 전용 하드웨어(GPS + 원자 시계)가 필요합니다.

**Q2. 서울 사용자가 미국 사용자의 게시물에 좋아요를 누를 때 어떻게 처리하는가?**
> 좋아요는 집계 값이므로 CRDT(PN-Counter)로 처리합니다. 서울 사용자의 좋아요 이벤트는 서울 리전에 기록되고, 비동기로 미국 리전에 복제됩니다. 미국 사용자가 자신의 게시물 좋아요 수를 볼 때 미국 리전의 값을 봅니다. 복제 지연(수 초)으로 인해 좋아요 수가 미국에서 약간 늦게 반영됩니다. 이는 최종 일관성(Eventual Consistency)이며 SNS에서 허용됩니다.

**Q3. 멀티-리전 DB 선택 시 CockroachDB와 PostgreSQL + 수동 샤딩 중 무엇을 선택하는가?**
> 팀 규모와 트래픽에 따라 다릅니다. 초기에는 PostgreSQL + 단일 리전 + Read Replica로 시작합니다. 글로벌 확장이 필요해지면 두 가지 선택지가 있습니다. CockroachDB/Spanner: 자동 분산, 강한 일관성, 높은 비용(Spanner), 학습 곡선. PostgreSQL + 수동 샤딩 + 비동기 복제: 직접 구현, 유연성, 유지보수 부담. 대부분의 스타트업은 Managed DB(Aurora Global)로 시작해 성장하면 검토합니다.

---

<div align="center">

[⬅️ 이전: 샤딩 심화](./04-sharding-deep-dive.md) | [README로 돌아가기](../README.md) | [다음: 장애 허용 설계 ➡️](../reliability/01-fault-tolerance.md)

</div>
