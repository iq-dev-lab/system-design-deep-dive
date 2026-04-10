# 소셜 피드 / 타임라인 (Twitter / Instagram)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 게시물 작성 (텍스트, 이미지, 동영상)
- 팔로우/언팔로우
- 홈 피드 조회 (팔로우한 사람들의 최신 게시물)
- 좋아요, 댓글, 공유

### 비기능 요구사항
- DAU 3억 명
- 게시물 작성 후 팔로워의 피드에 수 초 내 반영
- 피드 조회 지연시간: 200ms 이하
- 셀럽(Celebrity) 계정: 수천만 팔로워 보유

---

## 📊 규모 산정

```
게시물 작성:
  DAU 3억 × 0.1회 작성/일 = 3,000만 건/일 ≈ 350 TPS

피드 조회:
  DAU 3억 × 10회 피드 조회/일 = 30억 건/일 ≈ 35,000 QPS
  쓰기:읽기 = 1:100 → 읽기 최적화가 핵심

피드 데이터:
  팔로워 평균 200명 → 게시물 1개당 200개 피드 업데이트
  350 TPS × 200 = 70,000 피드 업데이트/초
```

---

## 🏗️ Fan-out 패턴 비교

```
핵심 질문: 내 피드를 어떻게 구성하는가?

방법 1: Pull (Fan-out on Read)
  피드 조회 시점에 팔로우 목록 조회 → 각각의 최신 게시물 합산
  
  흐름:
  User A 피드 요청 → 팔로우 목록 (B, C, D) 조회
                   → B, C, D의 최신 게시물 각각 조회
                   → 병합 + 정렬 → 반환
  
  장점: 쓰기 간단 (게시물 DB에 1건만 저장)
  단점: 읽기 느림 (팔로워 수 × DB 쿼리), 핫 유저 병목

방법 2: Push (Fan-out on Write)
  게시물 작성 시점에 모든 팔로워의 피드 캐시에 미리 추가
  
  흐름:
  User B 게시물 작성 → B의 팔로워 목록 (A, C, D...) 조회
                    → 각 팔로워의 피드 캐시에 게시물 ID 추가
  
  User A 피드 요청 → 자신의 피드 캐시 바로 반환 (매우 빠름)
  
  장점: 읽기 O(1), 피드 조회 매우 빠름
  단점: 셀럽(1억 팔로워)이 게시물 1개 → 1억 건 캐시 업데이트!
```

---

## 🔬 핵심 컴포넌트 — Hybrid Fan-out

```
현실: Twitter/Instagram 모두 Hybrid 방식 사용

규칙:
  일반 사용자 (팔로워 < 1만): Fan-out on Write (Push)
  셀럽 (팔로워 > 1만):        Fan-out on Read  (Pull)

[일반 사용자 게시물 작성]

user_id=1001 (팔로워 500명) 게시물 작성
      │
      ▼
  Post Service → DB에 게시물 저장
      │
      │ (비동기)
      ▼
  Kafka (fanout-topic)
      │
      ▼
  Fan-out Worker:
    팔로워 500명 목록 조회
    각 팔로워의 피드 캐시에 post_id 추가
    Redis LIST: LPUSH "feed:{follower_id}" post_id (최대 1000개 유지)


[셀럽 게시물 포함 피드 조회]

user_id=9999 피드 요청
      │
      ▼
  Feed Service:
    1. 캐시에서 일반 피드 로드 (Push로 미리 채워진 게시물들)
    2. 팔로우 중인 셀럽 목록 조회 (user_id=9999가 팔로우하는 셀럽)
    3. 각 셀럽의 최신 게시물 조회 (Pull)
    4. 1번 + 3번 결과 병합 + 시간 정렬
    5. 반환
```

### 피드 캐시 구조

```
Redis (피드 캐시):
  Key: "feed:{user_id}"
  Type: List (최신 게시물 ID, 최대 1000개)
  
  user_id=9999의 피드:
  "feed:9999" → [post_500, post_499, post_498, ..., post_1]
  
  조회:
  LRANGE "feed:9999" 0 19  → 최신 20개 반환

피드 페이지네이션:
  첫 페이지: LRANGE 0 19
  다음 페이지: LRANGE 20 39
  캐시 미스 (오래된 게시물): DB에서 직접 조회
  
  TTL 없음 (자주 쓰이는 데이터) + LRU 정책으로 비활성 사용자 자동 제거
```

### 게시물 DB 설계

```sql
-- 게시물 테이블 (Cassandra 또는 MySQL)
CREATE TABLE posts (
    post_id     BIGINT PRIMARY KEY,  -- Snowflake ID
    user_id     BIGINT NOT NULL,
    content     TEXT,
    media_urls  JSON,                -- 이미지/동영상 URL 배열
    like_count  BIGINT DEFAULT 0,
    reply_count BIGINT DEFAULT 0,
    created_at  TIMESTAMP,
    INDEX (user_id, created_at)      -- 특정 유저의 게시물 최신순 조회
);

-- 팔로우 관계 테이블
CREATE TABLE follows (
    follower_id  BIGINT,
    followee_id  BIGINT,
    created_at   TIMESTAMP,
    PRIMARY KEY  (follower_id, followee_id)
);

-- 반대 방향 인덱스 (내 팔로워 목록 빠른 조회)
CREATE TABLE followers (
    followee_id  BIGINT,
    follower_id  BIGINT,
    PRIMARY KEY  (followee_id, follower_id)
);
```

---

## 🔄 데이터 흐름 — 좋아요 집계

```
좋아요 폭주 방지 (셀럽 게시물에 동시 수만 건 좋아요):

방법 1: DB 직접 업데이트
  UPDATE posts SET like_count = like_count + 1 WHERE post_id = ?
  → 수만 TPS → DB 락 경합 → 병목

방법 2: Redis Counter + 배치 DB 반영 (권장)
  좋아요 → Redis INCR "likes:{post_id}"
  
  Redis Counter: 즉시 반응 (사용자가 봄)
  배치 (1분마다): SELECT count FROM Redis → UPDATE posts DB
  
  사용자에게 보이는 숫자:
    Redis에서 읽음 (빠름, 최신)
    DB는 최종 집계용

좋아요 중복 방지:
  Redis SET: SADD "liked:{post_id}" user_id
  → SET NX 방식으로 동일 사용자 중복 좋아요 차단
```

---

## ⚡ 셀럽 문제 (Hot Shard)

```
문제: 셀럽이 게시물 → 수억 팔로워에게 1초 내 배포?
  → Fan-out Worker 1억 건 처리 → 지연

해결: 우선순위 기반 Fan-out
  1. 셀럽 게시물 → 특별 처리 (Fan-out on Read)
  2. Fanout Worker 여러 개 병렬 처리
  3. 팔로워를 활성/비활성으로 분리
     - 비활성 팔로워 (30일 이상 미접속): Fan-out 스킵
     - 접속 시 최신 게시물 Pull 방식으로 로드

Thundering Herd (동시 대량 팬 피드 조회):
  셀럽 게시물 직후 수백만 팬이 동시에 피드 새로고침
  → Cache Stampede 발생
  → 해결: 게시물을 캐시에 Pre-populate (Push) 후 공개
           또는 짧은 TTL 캐시로 DB 부하 분산
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| Fan-out | Hybrid (Push + Pull) | 일반 사용자 빠른 피드 + 셀럽 쓰기 폭발 방지 |
| 피드 캐시 | Redis List | O(1) 읽기, 쓰기 비용 낮음 |
| 좋아요 | Redis Counter + 배치 | DB 직접 쓰기 경합 방지 |
| 게시물 DB | Cassandra (NoSQL) | 쓰기 많고, 시간 기반 조회, 수평 확장 |

---

## 🚀 확장 전략

```
지역별 피드 캐시:
  서울 사용자 → 서울 Redis 클러스터
  New York 사용자 → AWS us-east-1 Redis

Fan-out 우선순위 큐:
  High: 팔로워 < 1,000명 (즉시 처리)
  Medium: 팔로워 1,000 ~ 1만
  Low: 팔로워 > 1만 (비동기, 지연 허용)

피드 용량 절감:
  Redis에는 post_id만 저장 (정수 8 bytes)
  피드 조회 시 post_id 목록으로 게시물 내용 Batch 조회
  → post 캐시(별도 Redis)에서 멀티-get
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. Fan-out: Hybrid (일반 Push / 셀럽 Pull)
  2. 피드 캐시: Redis List (post_id만 저장, 최대 1000개)
  3. 셀럽 기준: 팔로워 1만 명 이상 → Pull 방식 전환
  4. 좋아요: Redis Counter → 배치 DB 반영
  5. 비활성 팔로워: Fan-out 스킵으로 불필요한 캐시 낭비 방지
```

---

## 🤔 심화 질문

**Q1. 팔로우한 사람이 매우 많은 경우 피드는 어떻게 구성하는가?**
> 팔로우가 10,000명이라면 Push 방식으로는 10,000명의 게시물이 실시간으로 내 피드 캐시에 쌓입니다. Twitter는 이 경우 피드 캐시를 단순 최신순이 아닌 랭킹 알고리즘(참여도, 관계 강도)으로 정렬해 가장 관련성 높은 게시물을 상위에 배치합니다.

**Q2. 피드 랭킹(알고리즘 피드)을 어떻게 구현하는가?**
> 단순 시간 역순 대신 ML 랭킹을 적용합니다. 피처로는 게시물 좋아요/댓글 수, 작성자와의 상호작용 빈도, 게시물 나이, 미디어 포함 여부 등을 사용합니다. 실시간 랭킹은 너무 비싸므로, 캐시에 미리 랭킹된 피드를 30~60초 주기로 업데이트합니다.

**Q3. 피드 무한 스크롤을 어떻게 구현하는가?**
> 커서 기반 페이지네이션을 사용합니다. 첫 페이지 반환 시 마지막 게시물의 post_id를 cursor로 반환합니다. 다음 페이지 요청 시 `WHERE post_id < cursor` 조건으로 조회합니다. Redis List에서는 LRANGE의 offset을 cursor로 사용할 수 있으나, 피드가 실시간으로 추가되면 offset이 밀릴 수 있어 post_id 기반 커서가 더 안정적입니다.

---

<div align="center">

[⬅️ 이전: 동영상 스트리밍](./01-video-streaming.md) | [README로 돌아가기](../README.md) | [다음: 채팅 시스템 ➡️](./03-chat-system.md)

</div>
