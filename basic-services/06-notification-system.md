# 알림 시스템 설계

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- Push 알림 (iOS APNs, Android FCM)
- SMS 문자 발송
- 이메일 발송
- 사용자별 알림 수신 설정 (특정 채널 거부 가능)
- 알림 전송 로그 및 상태 추적

### 비기능 요구사항
- DAU 1000만, 일일 알림 1억 건 (초당 약 1,200건)
- 적어도 한 번 전달 보장 (At-Least-Once)
- 중복 알림 방지 (멱등성)
- 알림 지연 허용 범위: Push < 5초, SMS < 30초, 이메일 < 2분

---

## 📊 규모 산정

```
일일 알림 1억 건:
  Push: 7,000만 건 → 초당 810건
  SMS:  500만 건  → 초당 58건
  이메일: 2,500만 건 → 초당 290건

피크 (×5):
  Push: 초당 4,000건
  SMS:  초당 290건
  이메일: 초당 1,450건

저장 (30일 보관):
  건당 메타데이터: 200 bytes
  1억 × 30 × 200 = 600GB → MySQL 또는 Cassandra
```

---

## 🏗️ 개략적 설계

```
이벤트 소스 (결제완료, 팔로우, 댓글 등)
      │
      ▼
┌─────────────────┐
│  Notification   │  API 서버: 알림 요청 수신 및 라우팅
│  Service        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Kafka       │  채널별 토픽 분리
│  ├ push-topic   │
│  ├ sms-topic    │
│  └ email-topic  │
└────────┬────────┘
         │
    ┌────┴────────────┐
    │                 │                    │
    ▼                 ▼                    ▼
┌─────────┐    ┌─────────────┐    ┌────────────┐
│  Push   │    │  SMS Worker │    │Email Worker│
│  Worker │    │             │    │            │
└────┬────┘    └──────┬──────┘    └─────┬──────┘
     │                │                  │
     ▼                ▼                  ▼
┌────────┐    ┌──────────────┐    ┌──────────┐
│ APNs   │    │ Twilio/AWS   │    │ SendGrid │
│ FCM    │    │ SNS          │    │ SES      │
└────────┘    └──────────────┘    └──────────┘
```

---

## 🔬 핵심 컴포넌트 상세 설계

### Push 알림 — iOS APNs / Android FCM

```
푸시 알림 흐름:

서비스 서버                 APNs/FCM              사용자 기기
    │                         │                        │
    │──알림 페이로드 전송────▶│                        │
    │                         │──푸시 알림 전달──────▶│
    │◀──delivery receipt──────│                        │

Device Token:
  앱 설치 시 기기마다 고유한 토큰 발급
  DB에 {user_id: device_token} 저장 (N:1, 기기 여러 대 가능)

페이로드 예시 (FCM):
  {
    "to": "fcm_device_token_abc123",
    "notification": {
      "title": "김철수님이 댓글을 남겼습니다",
      "body": "정말 좋은 글이에요!",
      "icon": "notification_icon"
    },
    "data": {
      "post_id": "12345",
      "type": "comment"
    }
  }

실패 처리:
  APNs/FCM이 InvalidToken 반환 → DB에서 해당 토큰 삭제
  일시적 실패 → 지수 백오프로 재시도 (1s, 2s, 4s, 8s...)
```

### 알림 수신 설정 (Notification Preference)

```sql
CREATE TABLE notification_preferences (
    user_id     BIGINT NOT NULL,
    channel     ENUM('push', 'sms', 'email') NOT NULL,
    event_type  VARCHAR(50) NOT NULL,  -- 'comment', 'follow', 'payment' 등
    enabled     BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (user_id, channel, event_type)
);

-- 예시: user_id=1001은 이메일 광고 알림 OFF
INSERT INTO notification_preferences
VALUES (1001, 'email', 'marketing', FALSE);
```

알림 발송 전 설정 조회:

```python
def should_send(user_id, channel, event_type) -> bool:
    # DB 조회 (Redis 캐시로 성능 최적화)
    pref = cache.get(f"notif_pref:{user_id}:{channel}:{event_type}")
    if pref is None:
        pref = db.query(user_id, channel, event_type)
        cache.set(f"notif_pref:...", pref, ttl=3600)
    return pref.enabled
```

---

## 🔄 데이터 흐름 — 중복 방지 (멱등성)

```
문제: 네트워크 오류로 동일 알림을 두 번 발송하면?
  → 사용자가 같은 "댓글 알림"을 2번 받음 → 불쾌한 경험

해결: 알림 ID 기반 중복 방지

알림 생성 시:
  notification_id = UUID() 또는 Snowflake ID
  DB INSERT (notification_id, user_id, content, status='PENDING')

Worker 처리 시:
  1. Redis SET NX (notification_id, "processing", EX=300)
  2. 성공 → APNs/FCM 발송
  3. 발송 성공 → DB UPDATE status='SENT'
  4. 중복 처리(SET NX 실패) → 이미 처리됨 → 스킵

  Redis NX (Not eXists): 키가 없을 때만 설정 (원자적 분산 락)
```

---

## ⚡ 병목 식별과 해결

```
병목 1: APNs/FCM 외부 API 속도 제한
  → 외부 API가 초당 N건만 허용
  → 해결: Kafka Consumer 병렬 처리 + 외부 API 별도 연결 풀

병목 2: SMS 비용 최적화
  → SMS는 건당 비용 발생
  → 해결: 중요 알림만 SMS, 나머지는 Push + Email로 폴백

병목 3: 대규모 캠페인 알림 (1000만 명에게 동시 발송)
  문제: 한꺼번에 Kafka에 1000만 건 → Worker 과부하
  해결:
  ├── 시간 분산 발송 (배치 크기 조절)
  └── 사용자 세그먼트별로 큐에 순차 추가 (Rate Control)
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| 발송 방식 | 비동기 (Kafka) | 발송 실패가 API 응답에 영향 없음 |
| 중복 방지 | notification_id + Redis NX | 간단한 분산 락 구현 |
| 재시도 | 지수 백오프 | 즉시 재시도보다 외부 서비스 부하 감소 |
| 실패 메시지 | DLQ | 최대 N회 실패 후 개발팀 알림 |

---

## 🚀 확장 전략

```
채널별 독립 확장:
  Push Worker 10개 (가장 많음)
  Email Worker 5개
  SMS Worker 2개

알림 우선순위 큐:
  즉시 (결제 완료, 보안 알림) → High Priority Queue
  중요 (댓글, 팔로우)         → Normal Queue
  낮음 (마케팅, 추천)         → Low Priority Queue
  
  → 즉시 알림이 마케팅 알림 때문에 지연되지 않음
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 채널 분리: Push/SMS/Email 각각 독립 Kafka 토픽 + Worker
  2. 비동기: API → Kafka 발행 (즉시 응답) → Worker 발송
  3. 중복 방지: notification_id + Redis NX 분산 락
  4. 실패 처리: 지수 백오프 재시도 + DLQ
  5. 수신 설정: DB + Redis 캐시 (채널×이벤트 타입별 ON/OFF)
```

---

## 🤔 심화 질문

**Q1. 앱이 종료된 상태에서도 Push 알림이 오는 이유는?**
> APNs(iOS)와 FCM(Android)이 기기와 상시 연결을 유지합니다. 앱이 꺼져있어도 OS 레벨에서 연결을 유지하므로 서버가 APNs/FCM에 알림을 보내면 OS가 받아서 화면에 표시합니다. 앱 서버는 APNs/FCM에 전달만 하면 되고, 기기와 직접 연결을 유지할 필요가 없습니다.

**Q2. 알림 전송 성공/실패를 어떻게 추적하는가?**
> APNs는 전달 성공/실패 콜백을 제공하지만 기기가 알림을 실제로 봤는지는 알 수 없습니다. FCM은 delivery receipt를 지원합니다. 이메일은 open tracking (1x1 픽셀 이미지)이나 링크 클릭 추적으로 확인합니다. 이 데이터를 DB에 저장해 알림 효과를 분석합니다.

**Q3. 면접에서 "1000만 명에게 동시에 알림 발송을 설계하라"고 하면?**
> "단순히 Kafka에 1000만 건을 한꺼번에 넣으면 Worker가 급증한 요청을 처리하는 동안 실시간 알림(결제 완료 등)이 지연됩니다. 캠페인 알림은 별도의 Low Priority 큐로 분리하고, 사용자 세그먼트별로 시간을 나눠 발송합니다(분당 10만 건씩). APNs/FCM의 외부 API 속도 제한도 고려해 연결 풀과 Worker 수를 조정합니다."

---

<div align="center">

[⬅️ 이전: Rate Limiter 설계](./05-rate-limiter.md) | [README로 돌아가기](../README.md) | [다음: 검색 자동완성(Typeahead) ➡️](./07-typeahead.md)

</div>
