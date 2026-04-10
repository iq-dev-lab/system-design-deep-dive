# 분산 캐시 심화 (Redis Cluster / Memcached)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- Get/Set/Delete 연산
- TTL (자동 만료)
- 원자적 연산 (INCR, SETNX, GETSET)
- Pub/Sub (캐시 무효화 브로드캐스트)
- 분산 잠금 (Distributed Lock)

### 비기능 요구사항
- 초당 100만 ops/초
- 지연시간: P99 < 1ms
- 고가용성: 노드 1개 장애 시 자동 복구
- 용량: 수 TB 캐시 (분산)

---

## 📊 규모 산정

```
캐시 요구사항:
  초당 읽기: 900,000 ops/s
  초당 쓰기: 100,000 ops/s
  평균 값 크기: 1KB
  → 네트워크 대역폭: 1MB × 1,000,000 = 1TB/s (불가 → 압축 필수)

실제 대역폭:
  평균 값 크기를 200 bytes로 줄이면:
  200 × 1,000,000 = 200MB/s → 1Gbps 이더넷으로 처리 가능

노드 용량:
  Redis 노드 1대: 32GB 메모리 → 초당 수십만 ops
  100만 ops/s → 최소 5~10대 클러스터 필요
```

---

## 🏗️ Redis Cluster 아키텍처

```
Redis Cluster: 16,384개 해시 슬롯으로 데이터 분산

노드 3대 구성:
  Node 1 (Primary): 슬롯 0 ~ 5460
  Node 2 (Primary): 슬롯 5461 ~ 10922
  Node 3 (Primary): 슬롯 10923 ~ 16383

각 Primary에 Replica 1대:
  Node 1-replica: Node 1 데이터 복제
  Node 2-replica: Node 2 데이터 복제
  Node 3-replica: Node 3 데이터 복제

                 ┌──────────────────────┐
                 │    Redis Cluster     │
                 │                      │
  ┌──────────┐   │ ┌────────┐ ┌──────┐ │
  │ 클라이언트│──▶│ │Node 1P │ │Node1R│ │
  │ (Smart   │   │ │slot0~  │ │(복제)│ │
  │  Client) │   │ │5460    │ └──────┘ │
  └──────────┘   │ └────────┘          │
                 │ ┌────────┐ ┌──────┐ │
                 │ │Node 2P │ │Node2R│ │
                 │ │slot5461│ │(복제)│ │
                 │ │~10922  │ └──────┘ │
                 │ └────────┘          │
                 │ ┌────────┐ ┌──────┐ │
                 │ │Node 3P │ │Node3R│ │
                 │ │slot    │ │(복제)│ │
                 │ │~16383  │ └──────┘ │
                 │ └────────┘          │
                 └──────────────────────┘

키 → 슬롯 매핑:
  slot = CRC16(key) % 16384
  "user:1001" → CRC16("user:1001") % 16384 = 7234 → Node 2
```

---

## 🔬 핵심 컴포넌트

### 해시 슬롯과 키 분산

```
Hash Tag로 관련 키를 같은 노드에 배치:

문제: 여러 키를 Multi-get할 때 다른 노드에 분산되면 비효율

해결: Hash Tag {}
  key: "user:{1001}:profile"  → CRC16("1001")으로 슬롯 결정
  key: "user:{1001}:session"  → 같은 슬롯 → 같은 노드
  key: "user:{1001}:cart"     → 같은 슬롯 → 같은 노드
  
  → 사용자 1001 관련 키 모두 같은 노드 → 단일 노드 Multi-get 가능

주의: Hash Tag 과사용 → 특정 노드 Hot Shard 위험
  "{user}:1001", "{user}:1002" → 모두 "user" 슬롯 → 1개 노드에 집중
  → Tag는 구분자 역할에만 사용
```

### 분산 잠금 (Redlock)

```
단일 Redis 잠금의 문제:
  SETNX "lock:resource" "token" EX=30
  → Primary 장애 → Replica가 Promote됨
  → 하지만 Replica는 잠금 정보를 받기 전일 수 있음
  → 새 Primary에 SETNX 성공 → 두 클라이언트 동시에 잠금 획득!

Redlock (Martin Kleppmann이 비판하지만, 실무에서 많이 사용):
  N개의 독립된 Redis 인스턴스 (보통 5개)
  과반수 (N/2 + 1)에서 잠금 획득 시 유효

Redlock 알고리즘:
  1. 현재 시각 T1 기록
  2. 각 Redis에 SETNX "lock:resource" random_token EX=lock_time
  3. N개 중 3개 이상 성공 + (현재 시각 - T1) < lock_time → 잠금 성공
  4. 실패 시 모든 Redis에서 잠금 해제

구현:
  # Python redlock-py
  from redlock import RedLock
  
  redlock = RedLock([{"host": "redis1"}, {"host": "redis2"}, {"host": "redis3"}])
  
  with redlock.lock("resource_name", ttl=30000):
      # 임계 구역
      process()
  # 자동 해제
```

### Cache Stampede 방지 (PER 알고리즘)

```
문제: TTL 만료 순간 다수 요청이 동시에 DB 조회

PER (Probabilistic Early Recomputation):
  만료 전 미리 갱신 (확률적)

  def get_with_per(key, ttl, delta=1.0):
      value, expiry = cache.get_with_ttl(key)
      if value is None:
          # 완전 캐시 미스 → 즉시 재계산
          return recompute_and_cache(key, ttl)
      
      # 남은 TTL과 재계산 비용 기반 조기 갱신 결정
      remaining_ttl = expiry - time.time()
      if remaining_ttl - delta * math.log(random.random()) < 0:
          # 확률적으로 미리 갱신 (만료 임박)
          refresh_in_background(key, ttl)
      
      return value

실제 적용:
  TTL 300초, 남은 TTL 30초 → 갱신 확률 10%
  남은 TTL 5초 → 갱신 확률 94%
  → 한 클라이언트만 갱신, 나머지는 기존 값 사용 (Stale 허용)
```

### Redis vs Memcached 비교

```
┌───────────────────┬───────────────────┬───────────────────┐
│ 특성              │ Redis             │ Memcached         │
├───────────────────┼───────────────────┼───────────────────┤
│ 데이터 구조       │ String, Hash,     │ String만          │
│                   │ List, Set, ZSet   │                   │
├───────────────────┼───────────────────┼───────────────────┤
│ 영속성            │ RDB + AOF 지원    │ 없음 (메모리만)   │
├───────────────────┼───────────────────┼───────────────────┤
│ 클러스터          │ Redis Cluster     │ Client-side 샤딩  │
│                   │ (내장)            │                   │
├───────────────────┼───────────────────┼───────────────────┤
│ 성능              │ 단일 쓰레드       │ 멀티 쓰레드       │
│                   │ (6.0+ 멀티쓰레드) │ (CPU 다수 활용)   │
├───────────────────┼───────────────────┼───────────────────┤
│ Pub/Sub           │ 지원              │ 없음              │
├───────────────────┼───────────────────┼───────────────────┤
│ 원자적 연산       │ INCR, SETNX,      │ 제한적            │
│                   │ Lua Script        │                   │
├───────────────────┼───────────────────┼───────────────────┤
│ 선택 기준         │ 대부분의 경우     │ 단순 캐시,        │
│                   │ (풍부한 기능)     │ 멀티코어 최대 활용│
└───────────────────┴───────────────────┴───────────────────┘
```

---

## 🔄 데이터 흐름 — 캐시 계층 무효화

```
문제: DB 업데이트 후 캐시를 어떻게 무효화하는가?

방법 1: Cache-Aside + TTL
  DB 업데이트 → 캐시 DELETE
  다음 읽기 시 DB에서 재로드 → 캐시 SET
  
  문제: 삭제-재로드 사이에 스탬피드 가능

방법 2: Write-Through
  모든 DB 쓰기 시 동시에 캐시 업데이트
  → 항상 최신 캐시 유지
  → 캐시 쓰기 실패 시 DB 쓰기도 롤백 필요

방법 3: Pub/Sub 브로드캐스트 (분산 환경)
  DB 업데이트 발생
       │
       ▼
  Redis PUBLISH "cache-invalidate" "user:1001"
       │
       ▼
  모든 앱 서버가 SUBSCRIBE 중
       │
       ▼
  각 서버의 로컬 캐시 (L1)에서 "user:1001" 삭제
  
  → L1 캐시 (인메모리)와 Redis (L2) 모두 무효화
  → 로컬 캐시 hit rate 높이되, 일관성 문제 해결
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| 클러스터링 | Redis Cluster | 내장 샤딩, 자동 장애 조치 |
| 분산 잠금 | Redlock (5노드) | 단일 노드 잠금의 장애 취약성 방지 |
| Stampede 방지 | PER 알고리즘 | Mutex보다 성능 좋음, 약한 일관성 허용 |
| 캐시 무효화 | Pub/Sub | 실시간 L1 캐시 일관성 유지 |

---

## 🚀 확장 전략

```
수평 확장:
  샤드 추가 (노드 추가)
  → 리샤딩: 슬롯을 새 노드로 이동 (온라인 가능)
  → 클라이언트 설정 업데이트 불필요 (Smart Client 자동 인식)

메모리 최적화:
  OBJECT ENCODING 확인 (ziplist vs hashtable)
  작은 해시/셋 → ziplist (메모리 효율적)
  maxmemory-policy: allkeys-lru (메모리 가득 찰 때 LRU 제거)

읽기 확장:
  Replica에서 읽기 (READONLY 모드)
  → Primary 부하 감소
  → 약간의 Replication Lag 허용 필요
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 클러스터: Redis Cluster (16384 슬롯, 자동 샤딩)
  2. Hash Tag: 관련 키를 같은 노드에 (Multi-get 최적화)
  3. 분산 잠금: Redlock (5개 독립 인스턴스, 과반수 획득)
  4. Stampede 방지: PER 알고리즘 (확률적 조기 갱신)
  5. 무효화: Pub/Sub (L1 로컬 캐시 실시간 무효화)
```

---

## 🤔 심화 질문

**Q1. Redis Cluster에서 노드가 장애날 때 어떤 일이 벌어지는가?**
> Primary 노드 장애 발생 시 나머지 노드들이 Gossip 프로토콜로 장애를 감지합니다(기본 15초). 과반수 노드가 동의하면 해당 Primary의 Replica를 새 Primary로 승격합니다(Failover). 이 과정에서 약 15~30초 동안 해당 슬롯의 읽기/쓰기가 불가합니다. cluster-node-timeout 값을 낮추면 더 빠른 장애 감지가 가능하지만 불필요한 Failover 위험이 있습니다.

**Q2. Redis의 단일 스레드 모델이 왜 빠른가?**
> Context Switching과 Lock 경합이 없습니다. 멀티스레드는 공유 자원 접근 시 뮤텍스가 필요해 오버헤드가 발생하지만, 단일 스레드는 순차 처리로 이 비용이 없습니다. 또한 대부분의 Redis 연산은 메모리 접근이라 CPU보다 메모리 속도가 병목입니다. Redis 6.0부터 네트워크 I/O에 멀티스레드를 도입해 고대역폭 환경에서 성능이 개선됩니다.

**Q3. 캐시 히트율이 낮을 때 어떻게 개선하는가?**
> 먼저 어떤 키가 자주 미스되는지 분석합니다(`redis-cli monitor` 또는 SLOWLOG). TTL이 너무 짧으면 늘리고, 핫 데이터를 미리 워밍합니다(Cache Warming). 캐시 키 설계를 검토해 세분화된 키(user:1001:profile)를 통합된 키(user:1001)로 바꿔 여러 조각을 한 번에 캐시합니다. 이상적인 캐시 히트율은 95% 이상이며, 80% 이하면 DB에 심각한 부하가 됩니다.

---

<div align="center">

[⬅️ 이전: 시계열 데이터](./02-time-series.md) | [README로 돌아가기](../README.md) | [다음: 샤딩 심화 ➡️](./04-sharding-deep-dive.md)

</div>
