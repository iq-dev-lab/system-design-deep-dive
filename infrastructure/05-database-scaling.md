# 데이터베이스 확장

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 읽기 복제본(Read Replica)은 어떤 원리로 동작하고 어떤 한계가 있는가?
- 파티셔닝(Partitioning)과 샤딩(Sharding)은 어떻게 다른가?
- 샤딩 키를 잘못 선택하면 무슨 일이 생기는가 (Hot Shard 문제)?
- CQRS 패턴은 언제 도입하는가?
- DB 확장의 단계별 전환 시점은 언제인가?

---

## 🔍 왜 실무에서 중요한가

대부분의 서비스에서 읽기가 쓰기보다 10~100배 많습니다. 단일 DB 서버는 읽기 트래픽으로 먼저 병목이 됩니다. 읽기 복제본 → 캐싱 → 파티셔닝 → 샤딩 순서로 단계적으로 확장하는데, 각 단계의 트리거와 트레이드오프를 모르면 너무 일찍 샤딩을 도입하거나(복잡도 증가), 너무 늦게 도입해 서비스가 다운됩니다.

---

## 📖 읽기 복제본 (Read Replica)

### 동작 원리

```
Master (Write): 쓰기 전용 + WAL(Write-Ahead Log) 생성
                    │
                    │ 비동기 복제 (수 ms 지연)
                    ▼
Replica 1 (Read): WAL 적용 → 마스터와 동일한 데이터
Replica 2 (Read): WAL 적용 → 마스터와 동일한 데이터

App Server:
  쓰기 (INSERT/UPDATE/DELETE) → Master
  읽기 (SELECT) → Replica 1 또는 2 (라운드 로빈)
```

### Replication Lag (복제 지연)

```
마스터에 쓴 데이터가 레플리카에 즉시 반영되지 않습니다.

t=0ms:  Master에 user 생성 (name="김철수")
t=5ms:  Replica에 복제 완료

t=0~5ms 사이에 Replica 읽기:
  → "김철수" 없음 (아직 복제 안 됨)
  → 404 또는 빈 결과 반환 → 사용자 혼란

해결책:
  방법 1: 쓰기 직후 읽기는 Master에서 (Session Consistency)
          App이 "방금 쓰기를 했다"는 플래그를 세션에 저장
          → 짧은 시간 동안 Master 읽기 강제

  방법 2: 중요한 읽기만 Master에서
          결제 후 잔액 조회, 계정 생성 후 로그인 등

  방법 3: 캐시 활용
          쓰기 후 캐시에도 즉시 저장 → 캐시에서 읽기
```

### 언제 읽기 복제본을 추가하는가

```
시그널:
  ├── DB CPU 사용률 지속 70%+ 
  ├── SELECT 쿼리 응답 시간 증가
  ├── Slow Query 로그에 SELECT가 대부분
  └── 쓰기 성능은 괜찮지만 읽기가 느림

효과:
  레플리카 1개 추가: 읽기 처리량 2배
  레플리카 2개 추가: 읽기 처리량 3배
  (쓰기는 마스터 한 대가 담당 → 쓰기 확장 불가)
```

---

## 🗂️ 파티셔닝 (Partitioning)

같은 서버 내에서 하나의 테이블을 여러 파티션으로 분할합니다.

### 수평 파티셔닝 (Range Partitioning)

```
orders 테이블:

Partition 2022: ORDER_DATE BETWEEN '2022-01-01' AND '2022-12-31'
Partition 2023: ORDER_DATE BETWEEN '2023-01-01' AND '2023-12-31'
Partition 2024: ORDER_DATE BETWEEN '2024-01-01' AND '2024-12-31'

쿼리:
  SELECT * FROM orders WHERE order_date = '2023-06-01'
  → MySQL이 자동으로 Partition 2023만 스캔 (Partition Pruning)

장점:
  ✅ 오래된 파티션 삭제 쉬움 (DROP PARTITION)
  ✅ 날짜 기반 쿼리 매우 빠름

단점:
  ❌ 같은 서버 내 분할 → 서버 한계 여전히 존재
  ❌ 날짜 범위 밖의 쿼리는 모든 파티션 스캔
```

---

## 🔀 샤딩 (Sharding)

데이터를 여러 **다른 서버**에 분산 저장합니다.

### Hash Sharding

```
shard_id = hash(user_id) % 4

user_id=1001 → hash % 4 = 1 → Shard 1
user_id=1002 → hash % 4 = 2 → Shard 2
user_id=1003 → hash % 4 = 3 → Shard 3
user_id=1004 → hash % 4 = 0 → Shard 0

┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  Shard 0 │  │  Shard 1 │  │  Shard 2 │  │  Shard 3 │
│ user 0,4 │  │ user 1,5 │  │ user 2,6 │  │ user 3,7 │
└──────────┘  └──────────┘  └──────────┘  └──────────┘

장점: 데이터가 균등하게 분산됨
단점: 서버 추가 시 대부분의 데이터 이동 필요 (리밸런싱 비용)
```

### Range Sharding

```
user_id 0 ~ 249,999 → Shard 0
user_id 250,000 ~ 499,999 → Shard 1
user_id 500,000 ~ 749,999 → Shard 2
user_id 750,000 ~ 999,999 → Shard 3

장점: 범위 쿼리 가능, 새 샤드 추가 쉬움
단점: 최신 user_id가 Shard 3에 집중 → Hot Shard 문제 발생
```

---

## 🔥 Hot Shard 문제

```
문제 시나리오:
  SNS 서비스, 샤딩 키 = user_id

  셀러브리티 user_id=7890 → Shard 2 담당
  팔로워 1,000만 명 → Shard 2로 조회 집중 💥

                    초당 100만 요청
                          │
                          ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  Shard 0 │  │  Shard 1 │  │  Shard 2 │  │  Shard 3 │
  │   한가함  │  │   한가함  │  │ 과부하!! │  │   한가함  │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘

해결책:
  방법 1: 셀러브리티 전용 샤딩
           인기 유저 데이터를 여러 샤드에 복제 (read-only 복제)
  방법 2: 캐싱으로 샤드 읽기 부하 분산
           인기 데이터를 Redis에 캐싱 → 샤드 직접 조회 최소화
  방법 3: 샤딩 키 변경 (근본적 해결)
           user_id 대신 post_id 기반으로 데이터 분산
```

### 샤딩 키 선택 기준

```
좋은 샤딩 키의 조건:

1. 높은 카디널리티 (Cardinality)
   → 값의 종류가 많아야 균등 분산 가능
   → user_id(수백만): 좋음 / status(0~2): 나쁨

2. 접근 패턴과 일치
   → "user의 모든 주문 조회"라면 user_id로 샤딩
   → 한 user의 모든 데이터가 같은 샤드에 → JOIN 가능

3. Hot Spot이 없는 키
   → 시간 기반 Range Sharding → 최신 데이터가 마지막 샤드에 집중
   → Hash Sharding으로 균등 분산

4. 변경 불가능한 값
   → 샤딩 키가 변경되면 다른 샤드로 데이터 이동 필요
   → user_id(변경 없음): 좋음 / email(변경 가능): 나쁨
```

---

## 🏗️ CQRS (Command Query Responsibility Segregation)

쓰기(Command)와 읽기(Query)를 완전히 다른 모델로 분리합니다.

```
일반적 구조:
  App → DB (읽기 + 쓰기 모두)
  문제: 복잡한 집계 쿼리가 쓰기 성능에 영향

CQRS:
  Command (쓰기) → MySQL Master → 이벤트 발행 → Kafka
  Query  (읽기) ← Elasticsearch (검색 최적화 읽기 모델)
               ← Redis (캐시 기반 빠른 읽기)
               ← MySQL Replica (단순 쿼리)

       ┌──────────────┐
Write  │  Command     │──▶ MySQL Master
Path   │  Handler     │──▶ Kafka (이벤트)
       └──────────────┘         │
                                 │ 동기화 Worker
                          ┌──────▼───────┐
Read   ← Elasticsearch    │ Read Model   │
Path   ← Redis Cache  ◀── │ Projector    │
       ← MySQL Replica    └──────────────┘

장점:
  ✅ 읽기 모델을 목적에 맞게 최적화 (검색 → ES, 속도 → Redis)
  ✅ 쓰기와 읽기 서버를 독립적으로 확장
  ✅ 복잡한 집계가 쓰기 성능에 영향 없음

단점:
  ❌ 읽기 모델이 최신 데이터보다 약간 늦을 수 있음 (Eventual Consistency)
  ❌ 아키텍처 복잡도 증가
  ❌ 여러 저장소 관리 필요

도입 시점:
  읽기 패턴이 쓰기 패턴과 매우 다를 때
  복잡한 집계 쿼리가 성능 문제를 일으킬 때
  여러 저장소를 활용해 각 읽기 용도를 최적화하고 싶을 때
```

---

## 📈 DB 확장 단계별 로드맵

```
단계 1: 단일 서버 (DAU < 10만)
  ├── App + DB 동일 서버
  └── 트리거: DB CPU 70%+ 지속

단계 2: App/DB 분리 (DAU 10~50만)
  ├── App Server (분리) → MySQL (단일)
  └── 트리거: 읽기 느려짐, SELECT Slow Query 증가

단계 3: 읽기 복제본 (DAU 50만~500만)
  ├── App → MySQL Master (쓰기)
  │      → MySQL Replica × 2 (읽기)
  └── 트리거: Master 읽기 부하, Replica도 부족할 때 → 캐시 도입

단계 4: 캐싱 계층 (DAU 50만~500만 병행)
  ├── App → Redis → MySQL Replica
  └── 트리거: 여전히 읽기 부하 → 수평 파티셔닝

단계 5: 파티셔닝 (DAU 500만~)
  ├── 단일 서버 내 테이블 파티셔닝 (날짜 기반)
  └── 트리거: 단일 서버 한계 → 샤딩

단계 6: 샤딩 (DAU 수천만~)
  ├── 여러 DB 서버에 데이터 분산
  └── 마지막 수단 (복잡도 매우 높음)
```

---

## 📌 핵심 결정 요약

| 문제 | 해결책 | 트리거 |
|------|--------|--------|
| 읽기 느림 | 읽기 복제본 | DB CPU 70%+, SELECT Slow Query |
| 읽기 복제본도 부족 | Redis 캐싱 | 반복 조회가 많은 데이터 |
| 데이터 양 증가 | 파티셔닝 | 단일 테이블 수억 건 이상 |
| 쓰기도 병목 | 샤딩 | Master CPU 70%+ 지속 |
| 읽기 패턴 복잡 | CQRS | 집계 쿼리가 쓰기 성능 저하 |

---

## 🤔 심화 질문

**Q1. 샤딩 후 Cross-Shard JOIN은 어떻게 처리하는가?**
> 가능하면 피해야 합니다. 설계 시 "같이 조회되는 데이터는 같은 샤드에"를 원칙으로 샤딩 키를 결정합니다. 불가피한 경우 애플리케이션에서 여러 샤드에 병렬 쿼리 후 결과를 합산합니다. Vitess(MySQL 샤딩 미들웨어)는 Cross-Shard JOIN을 자동으로 처리해줍니다. → `data-intensive-patterns/04-sharding-deep-dive.md` 참고

**Q2. 샤드 수는 처음에 얼마나 설정해야 하는가?**
> 미래 확장을 고려해 넉넉하게 설정합니다. 서버 4대가 필요해도 논리적 샤드는 128개로 만들고, 처음에는 서버 한 대가 샤드 32개를 담당합니다. 나중에 서버가 늘어나면 샤드를 재분배합니다. 처음부터 샤드 수 = 서버 수로 하면 서버 추가 시 전체 데이터 이동이 필요합니다.

**Q3. 면접에서 "DAU 1억 서비스의 DB를 설계하라"고 하면?**
> "읽기:쓰기 비율을 먼저 파악합니다. 100:1이라면 읽기 복제본 + Redis 캐시로 대부분의 트래픽을 처리합니다. 쓰기가 많으면(예: 실시간 채팅) Cassandra 같은 쓰기 최적화 DB를 고려합니다. 핵심 비즈니스 데이터(결제, 계정)는 MySQL Master-Replica로 강한 일관성을 유지하고, 피드, 타임라인은 Redis + Cassandra로 가용성과 성능을 우선시합니다."

---

<div align="center">

[⬅️ 이전: 메시지 큐와 비동기 처리](./04-message-queue.md) | [README로 돌아가기](../README.md) | [다음: 검색 인프라 ➡️](./06-search-infrastructure.md)

</div>
