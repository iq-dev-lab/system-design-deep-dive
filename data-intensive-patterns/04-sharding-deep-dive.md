# 샤딩 심화 (DB 수평 분할)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 수 TB~PB 규모 데이터 저장
- 단일 DB 서버의 한계 극복
- 읽기/쓰기 부하 분산
- 특정 샤드 장애 시 부분 가용성 유지

### 비기능 요구사항
- 쓰기: 초당 100,000 TPS (단일 DB 한계 초과)
- 읽기: 초당 1,000,000 QPS
- 데이터 용량: 100TB+ (단일 서버 불가)
- 샤드 추가/삭제 시 데이터 이동 최소화

---

## 📊 규모 산정

```
단일 MySQL 서버 한계:
  스토리지: ~20TB (HDD) / ~10TB (SSD)
  쓰기: ~10,000 TPS
  읽기: ~100,000 QPS (Read Replica 포함 시)

100TB, 초당 100,000 쓰기 요구:
  필요 샤드 수: 스토리지 기준 5~10개, 쓰기 기준 10~15개
  → 약 16개 샤드 (2의 거듭제곱, 리샤딩 용이)

샤드당 Read Replica 2개:
  총 DB 인스턴스: 16 × 3 = 48대
```

---

## 🏗️ 샤딩 전략 비교

### 방법 1: Hash Sharding (가장 일반적)

```
shard_id = hash(shard_key) % num_shards

예: user_id를 샤드 키로 사용
  user_id=1001: hash(1001) % 4 = 1 → Shard 1
  user_id=1002: hash(1002) % 4 = 2 → Shard 2
  user_id=2001: hash(2001) % 4 = 3 → Shard 3

장점:
  ✅ 균등한 데이터 분산 (Hot Shard 방지)
  ✅ 구현 단순

단점:
  ❌ 샤드 수(N) 변경 시 대부분의 키가 다른 샤드로 이동
  ❌ 범위 쿼리 불가 (user_id 100~200 → 모든 샤드 조회)

적합: 랜덤 접근, 특정 키 조회 위주 (소셜 미디어 사용자 데이터)
```

### 방법 2: Range Sharding

```
날짜 또는 ID 범위 기준 분할:

Shard 1: user_id 1 ~ 10,000,000
Shard 2: user_id 10,000,001 ~ 20,000,000
Shard 3: user_id 20,000,001 ~ 30,000,000
Shard 4: user_id 30,000,001 ~ ...

장점:
  ✅ 범위 쿼리 효율적 (한 샤드 또는 연속 샤드에만 쿼리)
  ✅ 시간 기반 분할 → 최신 데이터 샤드만 활성

단점:
  ❌ Hot Shard (최신 데이터 샤드에 쓰기 집중)
  ❌ 불균등 분산 (초기 샤드는 데이터 적음)

적합: 주문 이력, 로그 데이터 (시간 기반 조회 많음)
```

### 방법 3: Directory-based Sharding

```
별도 매핑 테이블(Lookup Table)로 키 → 샤드 매핑 관리

Shard Directory (Redis 또는 DB):
  user_id=1001 → Shard 3
  user_id=1002 → Shard 1
  user_id=5000 → Shard 7

장점:
  ✅ 임의로 샤드 재배치 가능 (핫 샤드 마이그레이션)
  ✅ 이기종 샤드 크기 허용

단점:
  ❌ 모든 쿼리 전 Lookup 테이블 조회 (성능 오버헤드)
  ❌ Lookup 테이블 자체가 단일 장애점

적합: 동적 샤드 조정이 필요한 경우 (대규모 멀티테넌트)
```

### 방법 4: Consistent Hashing (분산 캐시, 키-값 저장소)

```
해시 링 기반 (키-값 저장소 섹션 참조):
  노드 추가/삭제 시 K/N개 키만 이동 (최소 재분배)
  
적합: Redis Cluster, Cassandra, 동적 노드 추가 빈번한 경우
```

---

## 🔬 핵심 컴포넌트 — 샤딩 라우팅

### 애플리케이션 레벨 샤딩

```python
class ShardRouter:
    def __init__(self, num_shards=16):
        self.num_shards = num_shards
        self.shards = {
            i: DatabaseConnection(f"shard-{i}.db.internal")
            for i in range(num_shards)
        }
    
    def get_shard(self, shard_key: int) -> int:
        return shard_key % self.num_shards
    
    def get_connection(self, user_id: int):
        shard_id = self.get_shard(user_id)
        return self.shards[shard_id]
    
    def execute(self, user_id: int, query: str, params: tuple):
        conn = self.get_connection(user_id)
        return conn.execute(query, params)

# 사용
router = ShardRouter(num_shards=16)
router.execute(user_id=1001,
               query="SELECT * FROM users WHERE id=?",
               params=(1001,))
```

### 크로스-샤드 쿼리 문제

```
문제: "모든 사용자 중 최근 7일 활성 사용자 수는?"

방법 1: Scatter-Gather
  모든 샤드에 동시 쿼리 전송:
    Shard 1: SELECT COUNT(*) FROM users WHERE last_active > ?  → 1,234
    Shard 2: SELECT COUNT(*) FROM users WHERE last_active > ?  → 987
    ...
    Shard 16: SELECT COUNT(*) FROM users WHERE last_active > ?  → 1,105
  
  집계 서버: 1,234 + 987 + ... + 1,105 = 합계 반환
  
  장점: 병렬 처리 → 16배 빠름
  단점: 모든 샤드에 부하, 집계 서버 필요

방법 2: 별도 집계 DB
  OLTP (샤딩): 실시간 사용자 데이터 (쓰기 최적화)
  OLAP (단일): 집계 분석 (읽기 최적화, DW)
  
  CDC로 OLTP → OLAP 동기화
  크로스-샤드 분석 쿼리 → OLAP에서 실행
  → 샤딩 DB에 부하 없음
```

### Hot Shard 문제와 해결

```
문제: 셀럽 유저(팔로워 1억 명)의 데이터가 Shard 5에 집중

시나리오:
  hash(celeb_id) % 16 = 5 → 셀럽 관련 모든 데이터 Shard 5
  팔로우/좋아요 폭발 → Shard 5만 과부하

해결책 1: 서브-파티셔닝
  Shard 5가 너무 크면 → Shard 5a, 5b, 5c로 분할
  hot_user_id → 추가 해시로 서브샤드 결정

해결책 2: 콘텐츠 분리
  셀럽의 게시물/팔로워 데이터를 별도 샤드로 격리
  일반 사용자 데이터와 분리 (Directory-based)

해결책 3: 읽기 레플리카 추가
  Shard 5 Primary + 읽기 레플리카 5대
  읽기 부하 → 레플리카로 분산
  쓰기는 Primary

해결책 4: 캐싱 강화
  Shard 5 부하 감소 → Redis 캐시 계층 강화
  셀럽 데이터는 TTL을 늘려 캐시 히트율 향상
```

---

## 🔄 데이터 흐름 — 리샤딩

```
상황: 샤드 수를 4개 → 8개로 늘려야 함 (데이터 증가)

Hash Sharding 리샤딩 문제:
  user_id=1001: hash(1001) % 4 = 1 (이전 Shard 1)
  user_id=1001: hash(1001) % 8 = 5 (이후 Shard 5)
  → 대부분의 키가 다른 샤드로 이동!

온라인 리샤딩 절차:

1. 더블 쓰기 기간 (점진적 이전):
   새 요청: 이전 샤드 + 새 샤드 모두에 쓰기
   기존 데이터: 백그라운드로 새 샤드로 복사

2. 트래픽 전환 (점진적):
   새 Shard 5~8 준비 완료
   트래픽 1%를 새 샤드로 → 모니터링
   문제 없으면 10% → 50% → 100%

3. 이전 샤드 정리:
   이전 데이터 삭제, 레거시 샤드 종료

일관성 해싱 사용 시:
  새 노드 추가 → 인접 슬롯의 데이터만 이동
  → 최소 재분배 (리샤딩 훨씬 쉬움)
```

---

## ⚖️ 트레이드오프

| 방식 | 균등 분산 | 범위 쿼리 | 리샤딩 용이 | 핫샤드 방지 |
|------|----------|----------|------------|------------|
| Hash | ✅ 좋음 | ❌ 불가 | ❌ 어려움 | ✅ 좋음 |
| Range | ❌ 불균등 | ✅ 좋음 | ✅ 쉬움 | ❌ 취약 |
| Directory | ✅ 제어 가능 | 보통 | ✅ 유연 | ✅ 수동 제어 |
| Consistent | ✅ 좋음 | ❌ 불가 | ✅ 최소 이동 | 보통 |

---

## 🚀 확장 전략

```
수직 확장 후 수평 확장:
  DB 서버 Scale Up (메모리, SSD 증가) 먼저 시도
  한계 도달 시 샤딩 도입 (복잡성 증가)

샤딩 전 최적화 체크리스트:
  ✅ 인덱스 최적화 (EXPLAIN ANALYZE로 쿼리 분석)
  ✅ Read Replica 추가 (읽기 부하 분산)
  ✅ 캐싱 레이어 도입 (DB 조회 감소)
  ✅ 쿼리 최적화 (N+1 쿼리 제거)
  
  위 방법으로도 해결 안 될 때 샤딩 도입

미들웨어 샤딩:
  Vitess (YouTube), ProxySQL → 애플리케이션 코드 수정 없이
  DB 앞에 프록시를 두고 자동 라우팅
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 샤드 키: 쿼리 패턴에 맞는 키 선택 (대부분 user_id)
  2. 전략: Hash (균등 분산 우선) 또는 Range (범위 쿼리 우선)
  3. 크로스-샤드 쿼리: Scatter-Gather 또는 별도 OLAP DB
  4. 핫샤드: 레플리카 추가 + 캐싱 + 서브-파티셔닝
  5. 리샤딩: 더블 쓰기 + 점진적 트래픽 전환
```

---

## 🤔 심화 질문

**Q1. 샤드 키 선택이 왜 그렇게 중요한가?**
> 샤드 키는 데이터 분산, 쿼리 패턴, 확장성을 결정합니다. 잘못된 샤드 키는 핫샤드(일부 샤드에 부하 집중), 크로스-샤드 쿼리 폭발, 리샤딩 어려움을 초래합니다. 예를 들어 이커머스에서 user_id로 샤딩하면 주문 조회가 한 샤드에서 해결되지만, 상품 ID로 샤딩하면 사용자의 주문 목록이 모든 샤드에 퍼져 크로스-샤드 쿼리가 됩니다.

**Q2. 샤딩 후 JOIN 쿼리를 어떻게 처리하는가?**
> 샤딩 환경에서 크로스-샤드 JOIN은 매우 비쌉니다. 해결책은 세 가지입니다. 첫째, 비정규화(Denormalization): 자주 JOIN하는 데이터를 같은 샤드에 중복 저장합니다. 둘째, 애플리케이션 레벨 JOIN: 각 샤드에서 데이터를 가져와 애플리케이션에서 병합합니다. 셋째, 분석 쿼리는 별도 OLAP DB(BigQuery, Redshift)에서 처리합니다.

**Q3. MongoDB와 Cassandra 중 샤딩 측면에서 어떤 차이가 있는가?**
> MongoDB는 Config Server + mongos 라우터로 수동 샤딩을 설정합니다. 범위 또는 해시 샤딩을 선택할 수 있으며 자동 리밸런싱을 지원합니다. Cassandra는 일관성 해싱 기반으로 처음부터 분산을 염두에 두고 설계됐습니다. 노드 추가 시 자동 데이터 리밸런싱이 되며 중앙 코디네이터가 없어 단일 장애점이 없습니다. 쓰기 많고 선형 확장이 중요하면 Cassandra, 복잡한 쿼리와 유연한 스키마가 필요하면 MongoDB를 선택합니다.

---

<div align="center">

[⬅️ 이전: 분산 캐시 심화](./03-distributed-cache.md) | [README로 돌아가기](../README.md) | [다음: 글로벌 분산 시스템 ➡️](./05-global-distribution.md)

</div>
