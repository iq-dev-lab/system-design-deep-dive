# 데이터 파이프라인 (Lambda / Kappa 아키텍처)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 실시간 데이터 수집 (클릭, 구매, 위치 등 이벤트)
- 배치 데이터 처리 (일/주/월별 집계)
- 실시간 분석 대시보드
- ML 학습 데이터 준비

### 비기능 요구사항
- 이벤트 처리 지연: 실시간 < 5초, 배치 < 1시간
- 일일 이벤트 10억 건 처리
- 데이터 유실 없음 (At-Least-Once)
- 수평 확장 가능

---

## 📊 규모 산정

```
이벤트 수집:
  DAU 1,000만 × 100 이벤트/일 = 10억 건/일
  초당: 10억 / 86,400 ≈ 11,600 이벤트/초
  이벤트 크기: 1KB
  일일 데이터: 1TB

저장:
  S3: 원본 이벤트 데이터 (90일 보관) = 90TB
  데이터 웨어하우스: 집계 결과 (2년) = 수 TB
  실시간 대시보드: 최근 7일 (실시간 집계)
```

---

## 🏗️ Lambda 아키텍처

```
Lambda Architecture:
  실시간 처리(Speed Layer)와 배치 처리(Batch Layer)를 병행

원본 데이터 (Kafka)
      │
      ├─────────────────────────────────────┐
      │                                     │
      ▼ (실시간)                             ▼ (배치)
┌──────────────┐                    ┌──────────────┐
│ Speed Layer  │                    │ Batch Layer  │
│ (Flink/Spark │                    │ (Spark)      │
│  Streaming)  │                    │              │
│              │                    │ 매일 자정     │
│ 최근 5분~1시간│                    │ 전날 데이터   │
│ 집계 결과     │                    │ 완전 집계     │
└──────┬───────┘                    └──────┬───────┘
       │                                   │
       ▼                                   ▼
┌──────────────┐                    ┌──────────────┐
│ Speed View   │                    │ Batch View   │
│ (Redis)      │                    │ (DW: Redshift│
│ 실시간 결과   │                    │  BigQuery)   │
└──────┬───────┘                    └──────┬───────┘
       │                                   │
       └────────────────┬──────────────────┘
                        │ 병합
                        ▼
                  ┌─────────────┐
                  │ Serving Layer│
                  │ API / 대시보드│
                  └─────────────┘

Lambda 특성:
  Batch Layer: 정확한 집계 (지연 있음)
  Speed Layer: 대략적 실시간 결과 (빠름)
  사용자는 두 결과의 Union(최신 배치 + 최신 실시간)을 봄
```

---

## 🔬 Kappa 아키텍처 (Lambda 개선)

```
문제: Lambda는 코드를 두 번 작성해야 함 (Batch + Streaming)
  → 유지보수 비용 2배, 결과 불일치 위험

해결: Kappa Architecture — 모든 처리를 Streaming으로

원본 데이터 → Kafka (긴 보존 기간, 예: 30일)
                  │
                  ▼
           Flink / Kafka Streams
           (단일 스트리밍 엔진)
                  │
           ┌──────┴──────┐
           │             │
           ▼             ▼
     실시간 집계        배치 재처리
     (최근 N분 윈도우)  (Kafka 처음부터 다시 읽기)
                          └── 집계 로직 수정 시 재처리

장점: 단일 코드베이스, 일관된 결과
단점: Kafka 보존 비용 증가, 대용량 재처리 시 느릴 수 있음

현실:
  Netflix, LinkedIn: Kappa 아키텍처 채택
  전통적인 대기업: Lambda 아키텍처 (배치 파이프라인이 이미 존재)
```

---

## 🔬 핵심 컴포넌트

### Kafka 이벤트 스트림

```
이벤트 스키마 (Avro / JSON):
{
  "event_type": "page_view",
  "user_id": 1234567,
  "session_id": "sess_abc",
  "timestamp": "2024-01-15T14:30:00Z",
  "properties": {
    "page": "/products/123",
    "referrer": "google.com",
    "device": "mobile"
  }
}

Kafka 토픽 설계:
  page-view-events    (파티션 100개)
  purchase-events     (파티션 50개)
  click-events        (파티션 200개)
  
  파티션 키: user_id → 같은 사용자 이벤트는 순서 보장
  보존 기간: 7일 (Kappa) 또는 24시간 (Lambda Speed)
```

### Flink 실시간 집계

```java
// Flink: 5분 윈도우 내 상품별 클릭 수 집계
DataStream<ClickEvent> clicks = env.addSource(kafkaSource);

clicks
  .keyBy(e -> e.getProductId())
  .window(TumblingEventTimeWindows.of(Time.minutes(5)))
  .aggregate(new CountAggregate())
  .addSink(redisSink);  // Redis에 실시간 집계 저장

// 집계 결과:
// Redis HSET "clicks:product:123:window:14:30" count 1500
// → 14:30~14:35 사이 상품 123의 클릭 수 1500
```

### 배치 처리 파이프라인

```python
# Spark: 일별 사용자 행동 분석
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

spark = SparkSession.builder.appName("DailyAnalysis").getOrCreate()

# S3에서 어제 데이터 로드
df = spark.read.parquet(f"s3://data/events/date=2024-01-14/")

# 사용자별 일별 집계
result = df.groupBy("user_id", "event_type") \
           .agg(count("*").alias("event_count"),
                countDistinct("session_id").alias("sessions")) \
           .filter(col("event_count") > 0)

# 데이터 웨어하우스에 저장
result.write.mode("overwrite") \
     .parquet("s3://dw/user-daily-stats/date=2024-01-14/")
```

---

## 🔄 데이터 흐름 — ML 피처 파이프라인

```
ML 모델 학습 데이터 준비:

Raw Events (Kafka)
      │
      ▼
Feature Engineering (Flink/Spark):
  사용자 최근 7일 클릭 패턴 집계
  상품 카테고리별 관심도 점수
  시간대별 활동 패턴
      │
      ▼
Feature Store (Redis + Offline Store):
  Online Feature Store (Redis): 실시간 추론용
    "feature:user:1001" → {click_7d: 150, purchase_7d: 3}
  
  Offline Feature Store (S3/Hive): 학습 데이터용
    date_partitioned Parquet 파일

ML 모델:
  추론 시: Redis에서 피처 즉시 조회 (수 ms)
  재학습 시: Offline Store에서 배치 로드 (수 TB)
```

---

## ⚖️ 트레이드오프

| 아키텍처 | 장점 | 단점 | 선택 시기 |
|---------|------|------|----------|
| Lambda | 정확한 배치 + 빠른 실시간 | 코드 중복, 결과 불일치 가능 | 기존 배치 인프라 있을 때 |
| Kappa | 단일 코드, 일관성 | Kafka 비용 증가, 재처리 복잡 | 신규 구축, 스트리밍 중심 |
| 배치만 | 구현 단순 | 실시간 불가 | 분석 지연 허용 시 |

---

## 🚀 확장 전략

```
Kafka 확장:
  파티션 수 증가 → 소비자 그룹 병렬도 증가
  Kafka 클러스터 추가 (브로커 증설)

Flink 확장:
  병렬도 조정 (parallelism 설정)
  TaskManager 수 증가 (Kubernetes Auto Scaling)

비용 최적화:
  S3 Intelligence-Tiering (접근 패턴 기반 자동 계층 이동)
  Parquet 형식 (CSV 대비 5~10배 압축)
  Presto/Athena로 S3 직접 쿼리 (DW 불필요)
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 수집: Kafka (At-Least-Once, 파티션 = 병렬도)
  2. 실시간: Flink 윈도우 집계 (5분 Tumbling Window)
  3. 배치: Spark (S3 Parquet, 일/주/월 집계)
  4. 아키텍처: Kappa (신규) 또는 Lambda (기존 배치 있을 때)
  5. ML: Feature Store (Online: Redis, Offline: S3)
```

---

## 🤔 심화 질문

**Q1. 이벤트 중복 처리(At-Least-Once)를 어떻게 다루는가?**
> 이벤트에 클라이언트 생성 UUID를 포함합니다. 처리 시 Redis SET에 해당 UUID가 있으면 스킵합니다(Exactly-Once 시맨틱 구현). Flink는 Kafka offset 커밋 + 체크포인트로 재시작 시 중복 방지합니다. 집계 연산(COUNT, SUM)은 멱등성이 없으므로 중복 이벤트 감지가 중요합니다.

**Q2. 실시간 이상 감지(Anomaly Detection)를 스트리밍으로 어떻게 구현하는가?**
> Flink CEP(Complex Event Processing)를 활용합니다. 예를 들어 "5초 내 동일 사용자 10번 결제 시도"를 패턴으로 정의하면, CEP가 실시간으로 이 패턴을 감지하고 Alert을 발행합니다. ML 모델을 Flink에 임베드해 각 이벤트를 모델에 입력하고 이상 점수를 계산하는 방식도 사용합니다.

**Q3. 데이터 파이프라인에서 스키마 변경을 어떻게 관리하는가?**
> Schema Registry(Confluent)를 사용합니다. Avro 스키마를 중앙 레지스트리에 등록하고, Producer/Consumer가 스키마를 조회해 직렬화/역직렬화합니다. 하위 호환성 규칙(새 필드는 기본값 필수)을 강제해 스키마 변경 시 기존 Consumer가 깨지지 않도록 합니다.

---

<div align="center">

[⬅️ 이전: 위치 기반 서비스](../large-scale-services/07-location-service.md) | [README로 돌아가기](../README.md) | [다음: 시계열 데이터 ➡️](./02-time-series.md)

</div>
