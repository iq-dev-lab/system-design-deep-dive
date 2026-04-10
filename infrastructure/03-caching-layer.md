# 캐싱 계층 설계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 캐시를 어디에 놓는 것이 가장 효과적인가?
- 캐시 히트율이 DB 부하에 어떤 영향을 미치는가?
- Cache-Aside, Write-Through, Write-Behind의 차이와 적합한 상황은?
- Cache Stampede(캐시 스탬피드)란 무엇이고 어떻게 막는가?
- 분산 캐시 구성에서 일관성은 어떻게 유지하는가?

---

## 🔍 왜 실무에서 중요한가

DB 조회는 네트워크 + 디스크 I/O로 보통 1~100ms가 걸립니다. 메모리 캐시는 0.1ms 이하입니다. 캐시 히트율이 90%이면 DB 요청이 10분의 1로 줄어듭니다. 반대로 캐시를 잘못 설계하면 Cache Stampede로 DB가 한꺼번에 폭발하거나, 오래된 데이터가 사용자에게 서빙됩니다. 캐시는 단순히 "빠르게"가 아니라 전략적으로 설계해야 합니다.

---

## 📍 캐시를 어디에 놓는가 — 캐시 계층 구조

```
Client
  │
  ├─▶ 1. 브라우저 캐시 (로컬, 수 MB)
  │      Cache-Control 헤더 기반
  │      효과: 완전히 동일한 요청 재발생 방지
  │
  ├─▶ 2. CDN 엣지 캐시 (지역 분산)
  │      정적 자원, 공개 API 응답 캐싱
  │      효과: 오리진 서버 요청 차단
  │
  ├─▶ 3. API Gateway / Reverse Proxy 캐시
  │      Nginx proxy_cache, API Gateway 캐싱
  │      효과: 동일 요청이 백엔드까지 도달하지 않음
  │
  ├─▶ 4. 애플리케이션 로컬 캐시 (서버 메모리)
  │      Caffeine, Guava Cache (Java)
  │      효과: Redis 요청도 줄일 수 있음
  │      단점: 서버마다 다른 캐시 → 일관성 문제
  │
  ├─▶ 5. 분산 캐시 (Redis / Memcached)
  │      모든 서버가 공유하는 캐시
  │      효과: DB 요청 대폭 감소
  │      → 대부분의 서비스에서 "캐시 도입"은 이것을 의미
  │
  └─▶ 6. DB 내부 캐시 (Buffer Pool)
         MySQL InnoDB Buffer Pool: 최근 조회 데이터 메모리 보관
         효과: 동일 쿼리의 디스크 I/O 방지
```

### 캐시 히트율과 DB 부하의 관계

```
초당 읽기 요청: 10,000 req/s

캐시 히트율  DB 요청/s  의미
    0%       10,000     캐시 없음 (DB가 모두 처리)
   80%        2,000     DB 부하 80% 감소
   90%        1,000     DB 부하 90% 감소
   99%          100     DB 부하 99% 감소

→ 히트율 80% → 90%로 올리는 것이 0% → 80%만큼 중요한 이유:
  80% 히트: DB 2,000 req/s 처리
  90% 히트: DB 1,000 req/s 처리 (DB 부하 절반으로 줄어듦)
```

---

## 🔄 캐시 갱신 전략 (Cache Update Patterns)

### 1. Cache-Aside (Lazy Loading) — 가장 일반적

```
읽기:
  App → Redis GET "user:1001" → 캐시 히트? → 반환
  캐시 미스 → DB 조회 → Redis SET "user:1001" → 반환

  ┌─────┐   GET user:1001  ┌───────┐
  │ App │ ────────────────▶ │ Redis │
  │     │ ◀─── 미스 ──────  └───────┘
  │     │                       
  │     │ SELECT * WHERE id=1001
  │     │ ────────────────▶ ┌──────┐
  │     │ ◀─── 결과 ──────  │  DB  │
  │     │                   └──────┘
  │     │ SET user:1001 ...
  │     │ ────────────────▶ ┌───────┐
  └─────┘                   │ Redis │
                             └───────┘

쓰기:
  App → DB UPDATE → Redis DEL "user:1001" (캐시 무효화)
  → 다음 읽기 시 DB에서 최신 데이터 조회 후 캐시 저장

장점:
  ✅ 실제로 사용되는 데이터만 캐싱 (공간 효율)
  ✅ Redis 장애 시 DB로 폴백 가능
  ✅ 구현이 직관적

단점:
  ❌ 캐시 미스 시 3번의 왕복 (Redis 조회 → DB 조회 → Redis 저장)
  ❌ 쓰기 후 Cache Stampede 가능 (많은 요청이 동시에 캐시 미스)
  ❌ 캐시와 DB 사이 짧은 불일치 시간 존재

적합: 대부분의 읽기 위주 서비스
```

### 2. Write-Through — 쓰기 시 항상 캐시 동시 업데이트

```
쓰기:
  App → Redis SET → DB INSERT/UPDATE (순서 보장)
  
  ┌─────┐  SET user:1001  ┌───────┐
  │ App │ ───────────────▶ │ Redis │
  │     │                  └───────┘
  │     │  INSERT user ...
  │     │ ───────────────▶ ┌──────┐
  └─────┘                  │  DB  │
                            └──────┘

읽기:
  항상 캐시에서 바로 조회 (DB 미스 거의 없음)

장점:
  ✅ 캐시와 DB 항상 일치 (강한 일관성)
  ✅ 읽기 항상 캐시 히트 (빠름)

단점:
  ❌ 쓰기 지연 (Redis + DB 순차 저장)
  ❌ 읽히지 않을 데이터도 캐시 저장 (공간 낭비)
  ❌ Redis 장애 시 쓰기 실패 (DB만 저장 옵션 필요)

적합: 읽기가 매우 많고 쓰기가 적은 서비스, 데이터 일관성이 중요한 경우
```

### 3. Write-Behind (Write-Back) — 캐시에 먼저 쓰고 DB는 나중에

```
쓰기:
  App → Redis SET → 즉시 응답
  → 별도 Worker가 일정 시간 후 DB에 배치 저장

  ┌─────┐  SET user:1001  ┌───────┐
  │ App │ ───────────────▶ │ Redis │ ─── (나중에) ──▶ ┌──────┐
  │     │ ◀── 즉시 응답    └───────┘   Worker          │  DB  │
  └─────┘                                             └──────┘

장점:
  ✅ 쓰기 지연시간 최소 (Redis만 기다림)
  ✅ DB 쓰기 요청을 배치로 모아 처리 → DB 부하 감소

단점:
  ❌ Redis 장애 시 데이터 유실 가능 (DB에 아직 저장 안 됨)
  ❌ 구현 복잡 (Worker 관리, 장애 처리)

적합: 쓰기 성능이 매우 중요하고 일시적 데이터 유실 허용되는 경우
      (게임 점수, 임시 세션 데이터, 조회수 카운터)
```

---

## 💥 Cache Stampede (캐시 스탬피드) 방지

인기 캐시 키의 TTL이 만료되는 순간, 수천 개의 요청이 동시에 DB로 몰리는 현상입니다.

```
시나리오:
  인기 상품 캐시 TTL 만료 시 동시 요청 1,000개

  t=0: cache TTL 만료
  t=1: 요청 1,000개 동시 Cache Miss
  t=1: 요청 1,000개 동시 DB 조회 시작
  t=1: DB 과부하 → 응답 지연 → 타임아웃 → 서비스 장애 💥
```

### 방지 전략

```
전략 1: Mutex Lock (분산 락)
  첫 번째 요청만 DB 조회 허용, 나머지는 대기

  if Redis.SET("lock:product:1001", "1", NX, EX=5):
    data = DB.query(1001)      # 첫 요청만 DB 조회
    Redis.SET("product:1001", data, EX=300)
    Redis.DEL("lock:product:1001")
  else:
    time.sleep(0.1)            # 잠시 대기 후 캐시 재조회
    data = Redis.GET("product:1001")

  단점: 락 해제 전까지 다른 요청 대기 (지연 발생)

전략 2: PER (Probabilistic Early Reexpiration)
  TTL 만료 전 확률적으로 미리 갱신
  → 만료 직전(예: 남은 TTL 10% 이하)에 일부 요청이 DB 조회
  → 실제 만료 시 이미 새 캐시 준비됨

  remaining = Redis.TTL("product:1001")
  if remaining < 30 and random() < 0.1:  # 10% 확률로 미리 갱신
    data = DB.query(1001)
    Redis.SET("product:1001", data, EX=300)

전략 3: Stale-While-Revalidate
  TTL 만료 후에도 오래된 데이터를 잠시 더 서빙
  그 사이에 백그라운드로 DB 조회 후 캐시 갱신

  Cache-Control: s-maxage=300, stale-while-revalidate=60
  → 300초 캐시, 만료 후 60초 동안 오래된 데이터 서빙하며 갱신
```

---

## 🔧 분산 캐시 구성

### Redis Cluster (수평 확장)

```
16,384개 해시 슬롯을 여러 노드에 분배:

  Key: "user:1001"
  → CRC16("user:1001") % 16384 = 7234번 슬롯
  → 7234번 슬롯 → Node 2 담당

  ┌─────────────────────────────────────────────────┐
  │  Redis Cluster (3 마스터 + 3 레플리카)            │
  │                                                 │
  │  [Node 1 Master] ─복제─▶ [Node 1 Replica]       │
  │  슬롯 0~5460                                    │
  │                                                 │
  │  [Node 2 Master] ─복제─▶ [Node 2 Replica]       │
  │  슬롯 5461~10922                                │
  │                                                 │
  │  [Node 3 Master] ─복제─▶ [Node 3 Replica]       │
  │  슬롯 10923~16383                               │
  └─────────────────────────────────────────────────┘

  Client → 어느 노드에 요청해도 MOVED 응답으로 올바른 노드 안내
```

### 로컬 캐시 + Redis 계층화 (Two-Level Cache)

```
App Server 1                   App Server 2
┌───────────────────┐          ┌───────────────────┐
│  Caffeine L1      │          │  Caffeine L1      │
│  (서버 로컬, 1ms)  │          │  (서버 로컬, 1ms)  │
└────────┬──────────┘          └────────┬──────────┘
         │                              │
         └──────────┬───────────────────┘
                    │
          ┌─────────▼─────────┐
          │  Redis L2 (5ms)   │
          └─────────┬─────────┘
                    │
          ┌─────────▼─────────┐
          │      DB           │
          └───────────────────┘

조회 순서: L1 → (미스) L2 → (미스) DB
          
문제: App Server 1의 L1 캐시가 갱신됐지만
      App Server 2의 L1 캐시는 오래된 값

해결: Redis Pub/Sub로 캐시 무효화 이벤트 전파
  DB 업데이트 → Redis PUBLISH "cache:invalidate" "user:1001"
  → 모든 서버 Subscribe 중 → 자신의 L1에서 해당 키 삭제
```

---

## 📌 핵심 결정 요약

| 상황 | 캐시 전략 | 이유 |
|------|----------|------|
| 읽기 위주, 데이터 변경 드뭄 | Cache-Aside + TTL | 단순하고 공간 효율적 |
| 읽기 매우 많고 쓰기도 있음 | Write-Through | 항상 최신 데이터 캐시 보장 |
| 쓰기 성능 최우선 | Write-Behind | 지연시간 최소화 (데이터 유실 감수) |
| 인기 데이터 집중 | Cache-Aside + PER/Mutex | Cache Stampede 방지 |
| 여러 서버 로컬 캐시 | L1(로컬) + L2(Redis) + Pub/Sub | 속도 + 일관성 |

---

## 🤔 심화 질문

**Q1. Memcached와 Redis의 선택 기준은?**
> 단순 Key-Value 캐시만 필요하고 멀티코어를 활용하려면 Memcached가 적합합니다. 하지만 Redis는 Sorted Set(리더보드), List(큐), Pub/Sub, 지속성(AOF/RDB), Cluster 등 훨씬 풍부한 기능을 제공합니다. 현재 대부분의 서비스는 Redis를 선택합니다. → `redis-deep-dive` 레포 참고

**Q2. 캐시 일관성이 깨졌을 때(Stale Data) 어떻게 감지하는가?**
> 애플리케이션 레벨에서 버전 번호(Version/ETag)를 함께 저장하고, 캐시에서 읽은 버전과 DB의 버전을 비교합니다. 또는 짧은 TTL로 자연 만료를 허용합니다. 일관성이 매우 중요한 경우 Cache-Aside 대신 Write-Through로 항상 최신 데이터를 유지합니다.

**Q3. 면접에서 "캐시 계층을 어떻게 설계하겠는가"라고 하면?**
> "먼저 데이터의 특성을 파악합니다. 읽기 위주이고 업데이트가 드물면 Cache-Aside + 긴 TTL을 사용합니다. 자주 업데이트되면 Write-Through로 일관성을 보장합니다. 인기 데이터 집중이 예상되면 PER이나 Mutex로 Cache Stampede를 방지합니다. 다중 서버 환경에서는 로컬 캐시(Caffeine) + 분산 캐시(Redis)를 계층화하고, Redis Pub/Sub으로 로컬 캐시 무효화를 전파합니다."

---

<div align="center">

[⬅️ 이전: CDN](./02-cdn.md) | [README로 돌아가기](../README.md) | [다음: 메시지 큐와 비동기 처리 ➡️](./04-message-queue.md)

</div>
