# 채팅 시스템 (Messenger / WhatsApp)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 1:1 실시간 채팅
- 그룹 채팅 (최대 500명)
- 온라인 상태 표시 (Online Presence)
- 읽음 확인 (Read Receipt)
- 메시지 영속 저장 (과거 메시지 조회)

### 비기능 요구사항
- DAU 5,000만 명
- 메시지 전송 지연: 100ms 이하
- 동시 접속 연결: 5,000만 개 (1인 1연결)
- 99.99% 가용성, 메시지 유실 없음

---

## 📊 규모 산정

```
메시지 전송:
  DAU 5,000만 × 40메시지/일 = 20억 건/일 ≈ 23,000 TPS
  평균 메시지 크기: 100 bytes
  일일 저장: 200GB

동시 연결:
  5,000만 WebSocket 연결
  서버 1대당 최대 5만 연결 유지 가능
  → WebSocket 서버 최소 1,000대 필요

메시지 저장:
  30일 보관: 200GB × 30 = 6TB
  → Cassandra (쓰기 많음, 시간 기반 조회)
```

---

## 🏗️ 개략적 설계

```
                    ┌───────────────────┐
      WebSocket     │  Chat Server 1    │──────┐
User A ─────────────▶  (user A 연결)   │      │
                    └───────────────────┘      │
                                               ▼
                    ┌───────────────────┐  ┌──────────┐
      WebSocket     │  Chat Server 2    │  │  Kafka   │
User B ─────────────▶  (user B 연결)   │  │(메시지 큐)│
                    └───────────────────┘  └────┬─────┘
                                               │
                    ┌───────────────────────────▼──────────────┐
                    │           Message Service                │
                    │  - 메시지 영속 저장 (Cassandra)          │
                    │  - Push 알림 (오프라인 사용자)            │
                    │  - 읽음 확인 업데이트                    │
                    └──────────────────────────────────────────┘

핵심: 두 사용자가 다른 Chat Server에 연결된 경우
  → Chat Server 1이 A의 메시지를 Kafka에 발행
  → Chat Server 2가 Kafka에서 소비 → B에게 전달
```

---

## 🔬 핵심 컴포넌트 — WebSocket 연결 관리

### WebSocket vs HTTP Polling

```
HTTP Long Polling:
  클라이언트 → 서버에 요청 전송
  서버가 새 메시지 올 때까지 연결 유지 (최대 30초)
  메시지 있으면 응답 → 클라이언트 즉시 재요청
  
  문제: 서버 연결 자원 낭비, 30초마다 재연결 오버헤드

WebSocket (권장):
  초기 HTTP Upgrade 핸드셰이크 → 양방향 TCP 연결 유지
  서버 → 클라이언트 언제든지 Push 가능
  HTTP 헤더 오버헤드 없음 (binary frame)
  
  장점: 낮은 지연 (TCP 유지), 서버 Push 지원
  단점: Stateful (연결 유지 서버 메모리 사용)

서버 1대당 WebSocket 연결 수:
  파일 디스크립터 기본 제한 1024 → ulimit 증가
  실제 한계: 메모리 (연결당 ~50KB) + CPU
  c5.4xlarge (16 vCPU, 32GB): 최대 5만 연결
  → 5,000만 연결 / 5만 = 1,000대 서버
```

### 메시지 전달 흐름

```
[User A → User B 메시지 전송]

1. User A가 Chat Server 1에 WebSocket 메시지 전송:
   {
     "type": "message",
     "to": "user_B",
     "content": "안녕하세요",
     "client_msg_id": "cli_12345"  // 클라이언트 중복 방지 ID
   }

2. Chat Server 1:
   a. 메시지 ID 생성 (Snowflake ID)
   b. Kafka에 발행 (chat-message-topic)
   c. User A에게 ACK 반환 {"msg_id": "server_999", "status": "sent"}

3. Kafka Consumer (Message Service):
   a. Cassandra에 메시지 영속 저장
   b. User B가 연결된 서버 조회 (Redis: "presence:user_B" → "server_2")
   c. Chat Server 2에 전달 요청

4. Chat Server 2:
   a. User B의 WebSocket 연결로 메시지 Push
   b. B가 오프라인: Push 알림 서비스로 전달

5. User B가 메시지 수신:
   → Chat Server 2에 읽음 확인 전송
   → Chat Server 2가 A의 서버(1)에 읽음 전달
   → A의 채팅창에 "읽음" 표시
```

### Presence 서비스 (온라인 상태)

```
문제: User A가 User B의 온라인 상태를 어떻게 아는가?

Redis를 활용한 Presence:
  로그인 시: SET "presence:{user_id}" "{server_id}" EX=30
  WebSocket 연결 중: 매 10초마다 갱신 (Heartbeat)
  로그아웃/연결 끊김: TTL 30초 후 자동 만료 → 오프라인

조회:
  GET "presence:{user_id}" → 값 있으면 온라인, 없으면 오프라인

그룹 채팅 참여자 온라인 상태:
  다수를 한번에 조회:
  MGET "presence:101" "presence:102" "presence:103" ...
  → 온라인인 경우만 WebSocket 서버에 전달
```

### 메시지 저장 (Cassandra)

```sql
-- 채팅방 메시지 테이블 (Cassandra)
CREATE TABLE messages (
    chat_id     UUID,          -- 채팅방 ID (1:1 또는 그룹)
    msg_id      BIGINT,        -- Snowflake ID (시간 정렬)
    sender_id   BIGINT,
    content     TEXT,
    msg_type    TEXT,          -- 'text', 'image', 'video'
    created_at  TIMESTAMP,
    PRIMARY KEY (chat_id, msg_id)
) WITH CLUSTERING ORDER BY (msg_id DESC);
-- chat_id로 파티션, msg_id로 정렬 (최신 메시지 빠른 조회)

-- 메시지 조회 (최신 50개):
SELECT * FROM messages
WHERE chat_id = ? AND msg_id < ? -- 커서 기반
ORDER BY msg_id DESC LIMIT 50;
```

---

## 🔄 그룹 채팅 메시지 전달

```
그룹 500명에게 메시지 전달:

방법 1: Fan-out (1 → 500)
  메시지 1건 → 500개 WebSocket 전달
  
  단점: 메시지 마다 500건 처리 → 대형 그룹에서 폭발적 증가

방법 2: Group 메시지 저장 + 구독 (권장)
  1. 메시지를 Cassandra (group:chat_id)에 저장
  2. Kafka 토픽: "group:{chat_id}"
  3. 온라인 멤버들의 Chat 서버가 구독 중
  4. Kafka가 각 Chat 서버에 한 번씩만 전달
     → Chat 서버가 해당 멤버에게 WebSocket Push

  장점: 메시지 1건 저장 → 각 서버가 알아서 전달
  N명 멤버, M개 서버라면 → M번 전달 (N번이 아님)
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| 연결 방식 | WebSocket | 양방향, 낮은 지연, 서버 Push |
| 메시지 저장 | Cassandra | 쓰기 성능, 시간 기반 조회, 수평 확장 |
| Presence | Redis TTL (30초) | 단순, 빠름, 자동 만료 |
| 그룹 메시지 | Kafka Pub/Sub | 대형 그룹 Fan-out 효율화 |

---

## 🚀 확장 전략

```
Chat 서버 확장:
  Nginx L7 Load Balancer → WebSocket 연결 분산
  Chat 서버 Auto Scaling (연결 수 기준)
  Kubernetes에서 HPA(수평 파드 오토스케일러) 설정

메시지 저장 확장:
  Cassandra Cluster (chat_id 기반 파티셔닝)
  오래된 메시지 → S3 아카이빙 (6개월 이상)
  최신 7일치만 Cassandra 유지

미디어 파일:
  이미지/동영상은 S3 직접 업로드 (Presigned URL)
  메시지에는 S3 URL만 저장
  CDN으로 빠른 미디어 로딩
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 연결: WebSocket (서버 1대당 5만 연결)
  2. 라우팅: Redis Presence (user → Chat 서버 매핑)
  3. 메시지: Kafka → Cassandra (시간 정렬, 파티셔닝)
  4. 그룹: Kafka Pub/Sub (Fan-out 최소화)
  5. 오프라인: Push 알림 서비스 (APNs/FCM)
```

---

## 🤔 심화 질문

**Q1. 메시지 순서를 어떻게 보장하는가?**
> Snowflake ID를 메시지 ID로 사용합니다. Snowflake는 타임스탬프 기반이므로 시간 순서가 대략 보장됩니다. 같은 ms 내 메시지는 서버 ID + 시퀀스로 구분합니다. 엄격한 순서가 필요하다면 채팅방별 시퀀스 번호를 Redis에서 INCR로 관리합니다.

**Q2. 서버 장애 시 진행 중인 WebSocket 연결은 어떻게 되는가?**
> Chat 서버 장애 시 해당 서버의 WebSocket 연결이 끊깁니다. 클라이언트는 reconnect 로직으로 다른 Chat 서버에 재연결합니다. 재연결 후 마지막 수신 msg_id 이후의 메시지를 DB에서 조회해 미수신 메시지를 복구합니다.

**Q3. E2E 암호화(End-to-End Encryption)를 어떻게 구현하는가?**
> Signal Protocol을 사용합니다(WhatsApp 방식). 각 기기가 공개키/비공개키 쌍 생성, 공개키는 서버에 등록. 메시지 전송 시 수신자의 공개키로 암호화 → 서버는 암호화된 내용을 전달만 하고 복호화 불가. 서버 측에서 메시지 내용 열람 불가능합니다.

---

<div align="center">

[⬅️ 이전: 소셜 피드](./02-social-feed.md) | [README로 돌아가기](../README.md) | [다음: 클라우드 스토리지 ➡️](./04-cloud-storage.md)

</div>
