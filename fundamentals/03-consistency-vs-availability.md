# 일관성(Consistency) vs 가용성(Availability)

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CAP Theorem이 실제 설계 결정에 어떻게 적용되는가?
- CP 시스템과 AP 시스템은 각각 언제 선택하는가?
- Eventual Consistency는 어떤 서비스에서 허용 가능한가?
- 네트워크 장애 시 MySQL과 Redis Cluster는 어떻게 다르게 동작하는가?
- PACELC는 CAP Theorem의 어떤 한계를 보완하는가?

---

## 🔍 왜 실무에서 중요한가

분산 시스템에서는 네트워크 장애가 반드시 발생합니다. 이때 일관성과 가용성 중 하나를 포기해야 합니다. 이 선택을 모르고 기술을 선택하면, 장애 상황에서 시스템이 어떻게 동작할지 예측할 수 없습니다. 결제 시스템에서 Eventual Consistency를 허용하면 이중 결제가 발생하고, SNS 타임라인에서 Strong Consistency를 요구하면 불필요한 지연이 생깁니다.

---

## 📖 CAP Theorem

분산 시스템은 세 가지 속성을 동시에 완벽히 보장할 수 없습니다.

```
               Consistency
              (모든 노드가
             같은 데이터 반환)
                    △
                   / \
                  /   \
                 /     \
                /  분산  \
               /  시스템  \
              /           \
             /─────────────\
Availability                Partition
(항상 응답 반환)             Tolerance
                           (네트워크 분리
                           에도 동작)

현실: 분산 시스템에서 P(네트워크 파티션)는 피할 수 없음
→ C와 A 중 하나를 선택해야 함
```

### 네트워크 파티션 상황

```
정상 상태:
  Node A ──────────── Node B
  [data=100]          [data=100]

파티션 발생:
  Node A ──✂──────── Node B
  [data=100]          [data=100]

User가 Node A에 쓰기 (data=200):
  Node A [data=200]   Node B [data=100]  ← 다름!

이 상태에서 Node B에 읽기 요청이 오면?
  CP 선택: Node B가 응답 거부 (일관성 보장, 가용성 희생)
  AP 선택: Node B가 오래된 값(100) 반환 (가용성 보장, 일관성 희생)
```

---

## 🔒 CP 시스템 — 일관성 우선

```
C + P 시스템: 파티션 발생 시 일관성을 위해 가용성을 포기

          ┌─────────────────────────────┐
          │     User Request            │
          └──────────┬──────────────────┘
                     │
          ┌──────────▼──────────────────┐
          │     MySQL Master            │
          │     (Leader)                │
          └──────────┬──────────────────┘
                     │ 파티션 발생!
                 ✂───┘
          ┌──────────────────────────────┐
          │     MySQL Replica            │
          │ 쓰기 요청 → 에러 반환 (거부) │
          └──────────────────────────────┘

CP의 동작:
  ✅ 읽기는 항상 최신 데이터
  ❌ 파티션 중 일부 노드는 쓰기 거부
  ❌ 가용성이 낮아짐 (일부 요청 실패)
```

**CP 시스템 예시**:
- MySQL (Master-Replica, Galera Cluster)
- PostgreSQL
- ZooKeeper (분산 코디네이터)
- HBase

**언제 CP를 선택하는가**:
- 금융 거래, 결제 (잔액 불일치 불허용)
- 재고 관리 (초과 판매 불허용)
- 분산 락, 리더 선출 (정확성이 핵심)
- 사용자 계정 생성 (중복 계정 불허용)

---

## 🌐 AP 시스템 — 가용성 우선

```
A + P 시스템: 파티션 발생 시 가용성을 위해 일관성을 포기

          ┌─────────────────────────────┐
          │     User Request            │
          └──────────┬──────────────────┘
                     │
          ┌──────────▼──────────────────┐
          │     Redis Node 1            │
          │     (data=200, 최신)        │
          └──────────────────────────────┘
                   파티션 발생! ✂
          ┌──────────────────────────────┐
          │     Redis Node 2            │
          │ 읽기 → 100 반환 (오래된 값) │ ← 응답은 함
          └──────────────────────────────┘

AP의 동작:
  ✅ 항상 응답 (파티션 중에도)
  ❌ 응답 값이 최신이 아닐 수 있음 (Eventual Consistency)
  ✅ 나중에 파티션 해소되면 데이터 동기화 (Convergence)
```

**AP 시스템 예시**:
- Redis Cluster (기본 설정)
- Cassandra
- DynamoDB
- CouchDB

**언제 AP를 선택하는가**:
- SNS 타임라인 (1초 늦게 보여도 무관)
- 제품 카탈로그, 가격 (잠시 오래된 가격 표시 허용 가능)
- 검색 색인 (약간의 시간 차 허용)
- 장바구니 (추후 정합성 맞춰도 됨)
- 조회수, 좋아요 수 (정확한 실시간값 불필요)

---

## 🔄 Eventual Consistency (최종 일관성)

Eventual Consistency는 "지금 당장은 다를 수 있지만, 결국에는 같아진다"는 보장입니다.

```
시간 흐름:

t=0: Node A [data=100], Node B [data=100]  ✅ 일치

t=1: Node A에 쓰기 (data=200)
     Node A [data=200], Node B [data=100]  ❌ 불일치 (Inconsistent Window)

t=2: 복제 진행 중...
     Node A [data=200], Node B [data=150]  ❌ 불일치

t=3: 복제 완료
     Node A [data=200], Node B [data=200]  ✅ 일치 (Converged)

→ Inconsistent Window = t=1 ~ t=3 (보통 수십 ms ~ 수 초)
```

**Eventual Consistency를 허용하는 설계 패턴**:

```
패턴 1: Read-Your-Writes
  자신이 쓴 것은 자신이 바로 읽을 수 있어야 함
  
  구현: 쓰기 직후 Master에서 읽기 (다른 사용자는 Replica)
  
  적용: SNS 게시물 작성 후 즉시 자신의 피드에 표시

패턴 2: Monotonic Reads
  같은 사용자는 시간이 지날수록 최신 데이터를 읽어야 함
  (과거로 되돌아가지 않음)
  
  구현: 사용자 ID를 Key로 특정 Replica에 고정 (IP Hash 방식)
  
  적용: 메시지 읽음 표시가 사라지면 안 됨

패턴 3: Conflict Resolution
  두 노드에서 동시에 같은 데이터를 수정 시
  
  전략 1: Last Write Wins (LWW) — 타임스탬프 기준 최신 값 승리
           → 단순하지만 데이터 유실 가능
  전략 2: CRDT — 합산 가능한 자료구조 (카운터, 세트)
           → 조회수, 좋아요에 적합
  전략 3: 애플리케이션 수준 해결 — 충돌 감지 후 사용자에게 선택 위임
           → Google Drive 충돌 해결
```

---

## 📊 PACELC — CAP의 한계를 보완

CAP는 "파티션 상황"만 다루지만, 실제로는 **정상 상태에서도 Latency와 Consistency 사이의 트레이드오프**가 존재합니다.

```
PACELC:
  P (Partition) 시:  A (Availability) vs C (Consistency)
  E (Else, 정상) 시: L (Latency) vs C (Consistency)

           파티션 발생?
          Yes         No
        ┌─────┐    ┌─────────────┐
        │P→AC │    │E→Latency vs │
        │     │    │  Consistency│
        └─────┘    └─────────────┘

예시:
  MySQL: PC/EC (파티션→C보장, 정상→C보장, 지연 허용)
  DynamoDB: PA/EL (파티션→A보장, 정상→Latency 우선)
  Cassandra: PA/EL (파티션→A보장, 정상→빠른 응답 우선)
  
  Cassandra는 CAP로 AP처럼 보이지만,
  Consistency Level을 QUORUM으로 올리면 지연이 늘어남
  → PACELC가 더 정확히 표현
```

---

## 📌 핵심 결정 요약

| 상황 | 선택 | 근거 |
|------|------|------|
| 결제, 잔액, 재고 | CP (MySQL Master) | 데이터 불일치가 실제 손해로 이어짐 |
| 세션, 캐시, 타임라인 | AP (Redis Cluster) | 잠시 오래된 값 허용 가능 |
| 분산 락, 리더 선출 | CP (ZooKeeper, etcd) | 정확성이 핵심 |
| 검색 색인, 추천 | AP (Elasticsearch) | 수 초 지연 허용 |
| 사용자 프로필 | 중간 (Cassandra, 조정 가능) | 기능별로 Consistency Level 설정 |

**설계 결정 프레임워크**:
```
"이 데이터가 일시적으로 틀렸을 때 실제 비즈니스 피해가 있는가?"
  YES → CP 선택 (강한 일관성)
  NO  → AP 선택 (Eventual Consistency로 성능 이득)
```

---

## 🤔 심화 질문

**Q1. CAP Theorem에서 P(Partition Tolerance)는 왜 항상 선택해야 하는가?**
> 단일 서버라면 P를 포기하고 CA 시스템을 만들 수 있습니다. 하지만 분산 시스템에서 네트워크 파티션은 피할 수 없습니다(케이블 절단, 라우터 장애, 패킷 드롭). P를 포기한다는 것은 "네트워크 장애가 절대 없다"고 가정하는 것이므로, 실제 분산 시스템에서는 현실적이지 않습니다.

**Q2. "강한 일관성(Strong Consistency)이 필요하지만 가용성도 높이려면?"**
> Consensus 알고리즘(Raft, Paxos)을 사용합니다. 과반수(Quorum) 노드가 살아있으면 쓰기가 가능합니다. 3개 노드 클러스터에서 1개 장애 시에도 동작(2/3 쿼럼). 단, 과반수 이상 장애 시 CP의 C를 선택해 쓰기를 거부합니다. MySQL InnoDB Cluster, etcd, ZooKeeper가 이 방식입니다.

**Q3. 면접에서 "트위터 타임라인에 강한 일관성이 필요한가?"라고 물으면?**
> "필요하지 않습니다. 팔로워의 타임라인에 새 트윗이 1~2초 늦게 나타나도 사용자 경험에 문제가 없습니다. AP 시스템(Redis Cluster)을 선택해 타임라인 조회 성능과 가용성을 우선시하겠습니다. 반면, 계정 생성이나 팔로우/언팔로우는 중복이나 불일치가 사용자에게 혼란을 주므로 MySQL을 사용해 강한 일관성을 보장하겠습니다. 같은 서비스 안에서도 기능별로 다른 일관성 수준을 선택하는 것이 실용적입니다."

---

<div align="center">

[⬅️ 이전: 가용성 계산](./02-availability.md) | [README로 돌아가기](../README.md) | [다음: 지연시간 vs 처리량 ➡️](./04-latency-vs-throughput.md)

</div>
