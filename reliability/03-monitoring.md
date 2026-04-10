# 모니터링과 알림 (SLI / SLO / Alert Fatigue)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 서비스 가용성, 응답시간, 에러율 실시간 추적
- 임계값 초과 시 알림 (PagerDuty, Slack)
- 대시보드 (Grafana)
- 로그 수집 및 검색 (ELK Stack)
- 분산 추적 (Jaeger, Zipkin)

### 비기능 요구사항
- 알림 지연: 장애 발생 후 1분 이내
- Alert Fatigue 최소화 (불필요한 알림 억제)
- 장애 원인 파악 MTTR (평균 복구 시간) < 30분
- 로그 보존: 30일 (Hot), 1년 (Cold)

---

## 📊 규모 산정

```
메트릭:
  서버 1,000대 × 메트릭 50개 × 1분마다
  → 초당 833 메트릭 포인트

로그:
  서버 1,000대 × 로그 1,000줄/분
  → 초당 17,000줄 → 약 170KB/s

알림:
  하루 평균 알림: 100건
  유효 알림(Action Required): 10%

분산 추적:
  초당 요청 10,000건 × 샘플링 1% = 100 trace/s
```

---

## 🏗️ 모니터링 스택 구성

```
[데이터 수집]

서비스/서버
  ├── 메트릭 → Prometheus (Pull) / Telegraf (Push)
  ├── 로그   → Fluentd / Logstash → Elasticsearch
  └── 추적   → OpenTelemetry SDK → Jaeger

[저장 및 처리]

Prometheus → Grafana (메트릭 대시보드)
Elasticsearch → Kibana (로그 검색)
Jaeger → Jaeger UI (트레이스 시각화)

[알림]

Prometheus AlertManager → PagerDuty (온콜 담당자)
                        → Slack (팀 채널)
                        → 이메일

[전체 아키텍처]

서비스 A → OpenTelemetry Agent → Collector (중앙)
서비스 B →                    │
서비스 C →                    │
                              ▼
                    ┌─────────────────────────┐
                    │  Observability Platform  │
                    │  ├── Prometheus (메트릭) │
                    │  ├── Elasticsearch (로그) │
                    │  └── Jaeger (추적)        │
                    └─────────────────────────┘
                              │
                    ┌─────────────────────────┐
                    │  시각화 & 알림           │
                    │  ├── Grafana            │
                    │  ├── Kibana             │
                    │  └── AlertManager       │
                    └─────────────────────────┘
```

---

## 🔬 핵심 컴포넌트 — SLI / SLO / SLA

### SLI (Service Level Indicator)

```
SLI: 서비스 품질을 나타내는 측정 지표

주요 SLI:
  가용성 SLI = 성공 요청 수 / 전체 요청 수
  지연 SLI   = N ms 이하로 응답한 요청 비율
  오류율 SLI  = 오류 요청 수 / 전체 요청 수
  처리량 SLI  = 초당 처리 요청 수

예시:
  지난 30일 동안:
  전체 요청: 1,000,000건
  성공 요청: 999,500건
  → 가용성 SLI = 99.95%
  
  응답시간 500ms 이하: 997,000건
  → 지연 SLI = 99.7%
```

### SLO (Service Level Objective)

```
SLO: 서비스가 달성해야 할 목표 수준

예시 SLO:
  가용성 SLO: 월간 99.9% 이상
  지연 SLO:   P95 응답시간 500ms 이하 (월간 99.5%의 요청)
  오류율 SLO: 오류율 0.1% 미만

에러 버짓 (Error Budget):
  가용성 SLO 99.9% → 허용 다운타임 = 0.1% = 월 44분
  이 44분이 "에러 버짓"
  
  잔여 에러 버짓 > 50% → 새 기능 배포 OK
  잔여 에러 버짓 < 20% → 새 기능 배포 중단, 안정화 집중
  에러 버짓 소진 → 배포 동결, SRE 우선 대응

에러 버짓 소진 속도 추적:
  현재 가용성: 99.85% → SLO 미달
  에러 버짓 소진률: (99.9% - 99.85%) / (100% - 99.9%) = 50%/일
  → 이 속도면 2일 후 에러 버짓 소진 → 즉시 대응
```

### Alert 설계 — Alert Fatigue 방지

```
Alert Fatigue: 너무 많은 알림 → 담당자가 알림을 무시하게 됨

나쁜 알림의 특징:
  ❌ 너무 낮은 임계값 (CPU > 70% → 빈번한 알림)
  ❌ 일시적 스파이크에 즉시 반응 (1초 임계값)
  ❌ 조치 불가능한 알림 (아무것도 할 수 없는 상황)
  ❌ 중복 알림 (동일 이슈에 20개 알림)

좋은 알림의 특징:
  ✅ 지속적인 문제에만 반응 (5분 평균 CPU > 90%)
  ✅ 사용자에게 영향이 있을 때만 (에러율 > 1%)
  ✅ 조치 가능 (담당자가 무엇을 해야 하는지 명확)
  ✅ 그루핑 (같은 원인의 10개 알림 → 1개로 통합)

알림 계층화:
  Critical (즉시 대응, 온콜 호출):
    "결제 서비스 에러율 > 5%", "서비스 완전 다운"
  Warning (업무 시간 내 처리):
    "응답시간 P95 > 1000ms", "디스크 사용률 > 80%"
  Info (로깅만, 알림 없음):
    "캐시 히트율 감소", "배포 시작"

AlertManager 설정 예시:
  groups:
  - name: payment_alerts
    rules:
    - alert: PaymentServiceDown
      expr: up{job="payment-service"} == 0
      for: 1m           # 1분 지속 시에만 알림
      labels:
        severity: critical
      annotations:
        summary: "결제 서비스 다운"
        runbook: "https://wiki/runbooks/payment-down"
```

---

## 🔄 분산 추적 (Distributed Tracing)

```
문제: 마이크로서비스에서 "왜 이 요청이 느린가?" 파악 어려움

분산 추적:
  사용자 요청 하나가 여러 서비스를 거침
  각 구간의 시간을 측정해 병목 파악

Trace 예시 (Jaeger):
  Request ID: trace-abc123

  [API Gateway]  ─── 50ms ───▶
      [Auth Service]  ── 30ms ──▶
      [User Service]  ── 5ms ──▶
      [Post Service]  ─────────── 200ms ───▶
          [DB Query]  ── 180ms ──▶  ← 병목!
          [Cache]     ── 10ms ──▶

→ Post Service의 DB Query가 180ms → 인덱스 확인 필요

구현 (OpenTelemetry):
  # Python
  from opentelemetry import trace
  
  tracer = trace.get_tracer(__name__)
  
  def get_user_posts(user_id):
      with tracer.start_as_current_span("get_user_posts") as span:
          span.set_attribute("user.id", user_id)
          
          with tracer.start_as_current_span("db_query"):
              posts = db.query("SELECT * FROM posts WHERE user_id=?", user_id)
          
          with tracer.start_as_current_span("format_response"):
              return format(posts)
```

### 로그 구조화 (Structured Logging)

```
나쁜 로그:
  "2024-01-15 14:30:15 ERROR Failed to process order"
  → 검색/분석 어려움

좋은 로그 (JSON):
  {
    "timestamp": "2024-01-15T14:30:15Z",
    "level": "ERROR",
    "service": "order-service",
    "version": "1.2.3",
    "trace_id": "abc123",
    "span_id": "def456",
    "user_id": 1001,
    "order_id": "ord-789",
    "error": "PaymentDeclined",
    "message": "결제 거절: 잔액 부족",
    "duration_ms": 250
  }

Elasticsearch 쿼리:
  {service: "order-service", level: "ERROR", user_id: 1001}
  → 1001 사용자의 주문 서비스 에러 내역 즉시 조회

Trace ID 연결:
  로그의 trace_id로 Jaeger에서 전체 요청 추적 가능
  → 에러 로그 → Jaeger에서 전체 Trace 확인 → 병목 파악
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| 메트릭 | Prometheus + Grafana | 오픈소스, K8s 표준, 강력한 쿼리 |
| 로그 | ELK Stack | 풀텍스트 검색, 집계 분석 |
| 추적 | Jaeger/Zipkin | OpenTelemetry 표준 지원 |
| 알림 | PagerDuty + 에러 버짓 기반 | Alert Fatigue 방지 |

---

## 🚀 확장 전략

```
Prometheus 확장:
  Thanos / Cortex: Prometheus Federation
  글로벌 집계 가능, 장기 저장 (S3)

로그 확장:
  Elasticsearch → OpenSearch (AWS)
  Loki (Grafana) 경량 대안 (로그를 구조적으로 저장하지 않음, 저렴)

AIOps:
  ML로 이상 감지 (AWS Lookout for Metrics)
  자동 RCA (Root Cause Analysis)
  노이즈 감소 (상관관계 있는 알림 자동 그루핑)
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. SLI/SLO: 측정 가능한 목표 수립 (99.9% 가용성)
  2. 에러 버짓: 잔여 버짓으로 배포/안정화 균형 결정
  3. Alert 설계: 지속적 문제만 + 조치 가능 + 그루핑
  4. 분산 추적: 요청 하나가 여러 서비스 거칠 때 병목 파악
  5. 구조화 로그: JSON + trace_id 연결 (로그↔추적 연계)
```

---

## 🤔 심화 질문

**Q1. SLO를 99.9%로 설정하면 왜 팀이 더 행복해지는가?**
> 99.9%는 월 44분의 에러 버짓을 줍니다. 에러 버짓이 있으면 개발팀은 "이 버짓 내에서 배포해도 된다"고 판단할 수 있습니다. SLO 없이 "무조건 완벽"을 추구하면 아무도 배포를 못하게 됩니다. 에러 버짓이 소진되면 배포를 멈추고 안정화에 집중합니다. 이는 개발 속도와 안정성의 균형을 수학적으로 관리하는 방법입니다.

**Q2. P99 지연을 측정할 때 어떤 함정이 있는가?**
> 서비스가 여러 인스턴스로 구성될 때 각 인스턴스의 P99가 500ms이어도, 한 사용자 요청이 5개 서비스를 거치면 최악의 경우 P99 × 5 = 2,500ms가 됩니다(꼬리 지연 증폭). 또한 작은 샘플에서 P99는 불안정합니다. 초당 100 요청 × 60초 = 6,000 샘플에서 P99는 60개 샘플 중 가장 느린 것으로, 이 60개가 어떤 요청이냐에 따라 크게 달라집니다.

**Q3. 온콜 담당자의 번아웃을 방지하는 방법은?**
> 알림 품질 향상이 핵심입니다. 주간 알림 검토 미팅에서 불필요한 알림을 제거합니다. 중복 알림 그루핑(같은 원인의 10개 알림 → 1개). 자동 해결되는 알림(5분 후 자동 정상화)은 알림 발송 자체를 없앱니다. 온콜 순환 주기를 1주일로 유지하고, 야간 알림 수를 주간에 검토합니다. Netflix SRE는 주당 온콜 알림 5건 이하를 목표로 합니다.

---

<div align="center">

[⬅️ 이전: 데이터 일관성](./02-data-consistency.md) | [README로 돌아가기](../README.md) | [다음: 재해 복구 ➡️](./04-disaster-recovery.md)

</div>
