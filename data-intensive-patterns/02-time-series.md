# 시계열 데이터 처리 (모니터링 / 분석)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 시스템 메트릭 수집 (CPU, 메모리, 네트워크, 응답시간)
- 실시간 대시보드 (최근 1시간 메트릭)
- 이상 감지 및 알림 (임계값 초과 시)
- 히스토리 데이터 조회 (최대 1년)
- 사용자 정의 쿼리 (합계, 평균, 최대값 등)

### 비기능 요구사항
- 서버 1만 대 × 메트릭 100개 × 10초마다 = 초당 100만 메트릭
- 쓰기 지연: 1초 이하
- 읽기 (대시보드): 500ms 이하
- 저장: 1년치 데이터, 오래된 데이터 자동 압축

---

## 📊 규모 산정

```
메트릭 수집:
  서버 수: 10,000대
  메트릭/서버: 100개 (CPU, 메모리, 디스크, 네트워크 등)
  수집 주기: 10초
  → 초당 쓰기: 10,000 × 100 / 10 = 100,000 쓰기/초
  메트릭 크기: {host, metric_name, value, timestamp} ≈ 50 bytes
  → 초당 5MB, 일일 432GB

저장:
  원본 (10초 해상도): 1주일 = 432GB × 7 = 3TB
  압축 (1분 평균, 1달): 상당히 줄어듦
  집계 (1시간 평균, 1년): 매우 작음
```

---

## 🏗️ 개략적 설계

```
[메트릭 수집 파이프라인]

서버/앱 → Metrics Agent (Prometheus exporter, Telegraf)
              │ (10초마다 Push 또는 Pull)
              ▼
         Kafka (metrics-topic)
              │ 버퍼링, At-Least-Once
              ▼
         ┌─────────────┐    ┌─────────────────────┐
         │ Stream      │    │ Time Series DB      │
         │ Processor   │───▶│ (InfluxDB / Prometheus│
         │ (Flink)     │    │  / TimescaleDB)      │
         └─────────────┘    └──────────┬──────────┘
              │                        │
              ▼ 임계값 초과             │ 쿼리
         Alert Engine               Dashboard
         (PagerDuty 연동)           (Grafana)
```

---

## 🔬 핵심 컴포넌트 — 시계열 DB 설계

### 시계열 데이터 특성

```
일반 RDBMS와 다른 특성:

1. 쓰기 패턴: 항상 현재 시각에만 쓰기 (과거 수정 없음)
   → Append-Only → LSM Tree 구조에 최적화

2. 읽기 패턴: 시간 범위 쿼리 (최근 1시간, 어제 등)
   → 시간 기준 인덱스 필수

3. 고카디널리티: host × metric_name 조합 = 수백만 시계열
   → 태그 기반 인덱싱

4. 압축: 시간순 값은 연속적 변화 → 높은 압축률
   CPU: [45.2, 45.3, 45.1, 44.9, ...] → Delta encoding

5. 보존 정책: 오래된 데이터 자동 삭제/다운샘플링
```

### InfluxDB 데이터 모델

```
Measurement (테이블 역할):  "cpu_usage"

Tags (인덱스, 필터링):      host="web-01", region="us-east"
Fields (실제 값):           value=45.2, user=20.1, system=25.1
Timestamp:                  1704067200000000000 (Unix 나노초)

쿼리 예시 (InfluxQL):
  SELECT mean("value")
  FROM "cpu_usage"
  WHERE "host" = 'web-01'
    AND time >= now() - 1h
  GROUP BY time(5m)
  → 최근 1시간, 5분 평균으로 집계

vs SQL:
  SELECT AVG(value), DATE_TRUNC('5 minutes', timestamp)
  FROM cpu_usage
  WHERE host='web-01' AND timestamp >= NOW() - INTERVAL '1 hour'
  GROUP BY 2
```

### 데이터 다운샘플링 (Rollup)

```
문제: 1년치 10초 해상도 데이터를 그대로 저장하면 용량 폭발

해결: 시간이 지날수록 해상도 낮춤 (Rollup)

보존 정책:
  0 ~ 7일:    원본 (10초 해상도)
  7일 ~ 1달:  1분 평균으로 롤업 → 1/6 데이터
  1달 ~ 1년:  1시간 평균으로 롤업 → 1/360 데이터
  1년 이상:   삭제 또는 1일 평균 아카이브

InfluxDB Continuous Query (자동 롤업):
  CREATE CONTINUOUS QUERY "cpu_1m_avg"
  ON mydb
  BEGIN
    SELECT mean("value")
    INTO "cpu_1m"."cpu_usage"
    FROM "cpu_usage"
    GROUP BY time(1m), host, region
  END

효과:
  원본: 1년 = 3TB × 52 = 156TB
  롤업 후: ~5TB (약 30배 절감)
```

### 고카디널리티 문제

```
문제: host × service × region × environment 조합
  host: 10,000개
  service: 100개
  region: 10개
  environment: 3개
  → 총 카디널리티: 3,000만 시계열!

InfluxDB 문제: 카디널리티가 높으면 메모리 사용량 폭발
  각 시계열마다 인덱스 항목 → 수천만 인덱스 → OOM

해결책:
  방법 1: 태그 카디널리티 제한
    host를 태그로 두지 말고, 별도 측정값으로 집계
    
  방법 2: ClickHouse 사용
    컬럼형 DB, 집계 쿼리 최적화
    고카디널리티 허용 (인덱스 방식 다름)
    
  방법 3: Prometheus + VictoriaMetrics
    Prometheus: Pull 방식 수집, 로컬 저장
    VictoriaMetrics: 원격 저장, 고카디널리티 지원
```

---

## 🔄 데이터 흐름 — 알림 시스템

```
임계값 기반 알림:
  규칙: "web-01의 CPU > 90% 가 5분 이상 지속"

Flink 스트리밍 처리:
  metrics 스트림
      │
      ▼
  필터: host="web-01", metric="cpu_usage"
      │
      ▼
  5분 텀블링 윈도우
      │
      ▼
  최대값 계산
      │
      ▼
  임계값(90) 초과? → Alert 이벤트 발행
      │
      ▼
  Alert Manager (중복 제거, 그룹화)
      │
      ▼
  PagerDuty / Slack / 이메일

Alert 중복 방지:
  동일 알림 반복 발생 시 → 1회만 통지
  Silencing: 유지보수 시간 알림 억제
  Grouping: 여러 서버 동시 CPU 급등 → 1개 알림으로 묶음
```

---

## ⚖️ 트레이드오프

| DB 선택 | 장점 | 단점 | 추천 사용처 |
|---------|------|------|------------|
| InfluxDB | 시계열 특화, InfluxQL | 고카디널리티 취약 | 메트릭 수백만 이하 |
| TimescaleDB | PostgreSQL 호환, SQL | 성능이 InfluxDB보다 낮음 | SQL 팀에 익숙할 때 |
| ClickHouse | 고카디널리티, 초고성능 | 학습 곡선 높음 | 수십억 시계열 |
| Prometheus | K8s 표준, 경량 | 장기 저장 부적합 | 소규모 모니터링 |

---

## 🚀 확장 전략

```
쓰기 확장:
  Kafka → 파티션 증가 → 시계열 DB Writer 병렬화

읽기 확장:
  Read Replica (InfluxDB Enterprise)
  Caching: 자주 쿼리되는 대시보드 결과 Redis 캐시

저장 최적화:
  압축 알고리즘: Gorilla (Facebook, 시계열 특화, 12배 압축)
  컬럼형 저장: 같은 메트릭의 연속값 → 높은 압축률

글로벌 집계:
  리전별 시계열 DB → 글로벌 집계 레이어
  "전체 서버 평균 CPU" = 각 리전 평균의 가중 평균
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 수집: Kafka 버퍼링 → 100,000 쓰기/초 처리
  2. 저장: 시계열 DB (InfluxDB/ClickHouse) - Append-Only
  3. 다운샘플링: 7일→1달→1년 해상도 자동 낮춤 (30배 절감)
  4. 알림: Flink 스트리밍 + 임계값 규칙 + Alert Manager
  5. 시각화: Grafana (InfluxDB/Prometheus 직접 연결)
```

---

## 🤔 심화 질문

**Q1. 시계열 DB와 일반 RDBMS 중 선택 기준은?**
> 초당 1만 건 이하 메트릭, SQL 친숙, 복잡한 JOIN 필요 → TimescaleDB(PostgreSQL 확장). 초당 수십만 건, 시계열 특화 집계, 자동 롤업 → InfluxDB. 수백억 시계열, 분석 쿼리 중심 → ClickHouse. 이미 PostgreSQL 인프라가 있다면 TimescaleDB로 시작하고 성능 문제가 생기면 마이그레이션합니다.

**Q2. Prometheus의 Pull 방식과 Push 방식의 차이는?**
> Pull: Prometheus 서버가 각 서비스의 `/metrics` 엔드포인트를 주기적으로 호출합니다. 설정이 중앙화되고 서비스가 Prometheus를 알 필요가 없습니다. 단, 방화벽 뒤 서비스나 단기 실행 Job은 Pull이 어렵습니다. Push: 서비스가 Pushgateway에 메트릭을 전송합니다. 배치 Job, 람다 함수에 적합합니다.

**Q3. 메트릭 이상 감지를 규칙 기반이 아닌 ML로 하면 어떤 점이 다른가?**
> 규칙 기반은 "CPU > 90%"처럼 정적 임계값이라 계절성, 시간대별 패턴을 고려하지 못합니다. ML 기반(Facebook Prophet, AWS Lookout for Metrics)은 정상 패턴을 학습해 "월요일 오전 9시 트래픽 급증은 정상"이라고 구분합니다. 오탐(False Positive)이 줄어드나 모델 학습/유지 비용이 증가합니다. Netflix는 두 방식을 조합해 사용합니다.

---

<div align="center">

[⬅️ 이전: 데이터 파이프라인](./01-data-pipeline.md) | [README로 돌아가기](../README.md) | [다음: 분산 캐시 심화 ➡️](./03-distributed-cache.md)

</div>
