<div align="center">

# 🏗️ System Design Deep Dive

**"기술을 아는 것과, 수백만 사용자를 위한 시스템을 설계할 수 있는 것은 다르다"**

<br/>

> *"`Redis를 쓴다` — 와 — `이 시점에 왜 캐시를 넣어야 하는지, 어떤 갱신 전략을 선택할지 트레이드오프로 설명할 수 있다`의 차이를 만드는 레포"*

Redis, Kafka, MySQL, Elasticsearch를 각각 아는 것과,  
**실제 서비스를 설계할 때 그것들을 조합하는 결정**을 내리는 것은 완전히 다른 능력입니다.

이 레포는 개별 기술을 다시 설명하지 않습니다.  
**"어떤 트레이드오프 때문에 어떤 기술을 선택하고, 어떻게 조합해서 대규모 시스템을 만드는가"** 를 다룹니다.

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-IQ_Dev_Lab-181717?style=flat-square&logo=github)](https://github.com/iq-dev-lab)
[![Docs](https://img.shields.io/badge/Docs-42개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

시스템 설계에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"이 기술이 무엇인가"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "캐시를 쓰면 빠릅니다" | 이 시점에 왜 캐시를 넣어야 하는가 — DB 부하 임계점, 캐시 갱신 전략(TTL/Write-Through/Event), Cache Stampede 방지까지 |
| "Kafka로 비동기 처리하세요" | 어떤 작업을 메시지 큐로 분리해야 하는가 — 동기 처리를 유지하면 트래픽 급증 시 무슨 일이 생기는지 |
| "DB가 느리면 샤딩하세요" | 어떤 키로 샤딩할 것인가 — Hot Shard 문제, 일관성 해싱, Vitess 아키텍처까지 |
| "MSA를 도입하세요" | 단일 서버 → 읽기 복제본 → 캐시 → 샤딩 → MSA 각 단계를 전환하는 실제 트리거 |
| 아키텍처 다이어그램 나열 | DAU → 초당 요청 수 → 저장 용량 계산부터 시작하는 설계, 모든 기술 선택에 반드시 트레이드오프 포함 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Ch1](https://img.shields.io/badge/🔹_Fundamentals-확장성_완전_분해-2B7BB9?style=for-the-badge)](./fundamentals/01-scalability.md)
[![Ch2](https://img.shields.io/badge/🔹_Infrastructure-DNS와_로드밸런서-2B7BB9?style=for-the-badge)](./infrastructure/01-dns-load-balancer.md)
[![Ch3](https://img.shields.io/badge/🔹_Basic_Services-URL_단축_서비스-2B7BB9?style=for-the-badge)](./basic-services/01-url-shortener.md)
[![Ch4](https://img.shields.io/badge/🔹_Large_Scale-동영상_스트리밍_설계-2B7BB9?style=for-the-badge)](./large-scale-services/01-video-streaming.md)
[![Ch5](https://img.shields.io/badge/🔹_Data_Intensive-데이터_파이프라인-2B7BB9?style=for-the-badge)](./data-intensive-patterns/01-data-pipeline.md)
[![Ch6](https://img.shields.io/badge/🔹_Reliability-장애_허용_설계-2B7BB9?style=for-the-badge)](./reliability/01-fault-tolerance.md)
[![Ch7](https://img.shields.io/badge/🔹_Decision-ADR_아키텍처_결정-2B7BB9?style=for-the-badge)](./decision-framework/01-adr.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Fundamentals — 시스템 설계의 기초 원칙

> **핵심 질문:** 수직 확장의 한계는 언제 오는가? 99.9%와 99.99% 가용성의 실제 차이는? CAP Theorem은 어떤 설계 결정에 어떻게 적용되는가?

<details>
<summary><b>확장성부터 설계 면접 접근법까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 확장성(Scalability) 완전 분해](./fundamentals/01-scalability.md) | 수직 확장(Scale Up)의 물리적 한계, 수평 확장이 가능하려면 무엇이 달라야 하는가(Stateless), 로드밸런서 분산 알고리즘(Round Robin/Least Connection/IP Hash), Sticky Session의 문제 |
| [02. 가용성(Availability) 계산](./fundamentals/02-availability.md) | 99.9%(3 Nine) vs 99.99%(4 Nine)의 실제 다운타임 차이(연간 8.7시간 vs 52분), SLA와 SLO, 단일 장애점(SPOF) 제거 전략, 이중화(Redundancy)의 비용 |
| [03. 일관성 vs 가용성](./fundamentals/03-consistency-vs-availability.md) | CAP Theorem이 실제 설계 결정에 어떻게 적용되는가, CP 시스템(MySQL Master)과 AP 시스템(Redis Cluster) 선택 기준, Eventual Consistency를 허용하는 설계 |
| [04. 지연시간 vs 처리량](./fundamentals/04-latency-vs-throughput.md) | P50/P95/P99 지연시간의 의미, 롱테일 지연시간 문제, 처리량을 높이면 지연시간이 올라가는 이유, 배치 처리 트레이드오프 |
| [05. 용량 산정(Capacity Estimation)](./fundamentals/05-capacity-estimation.md) | DAU/MAU로 초당 요청 수 계산, 저장 용량 산정(5년치), 네트워크 대역폭 계산, Back-of-the-Envelope 계산법과 면접 활용법 |
| [06. 데이터 모델 선택](./fundamentals/06-data-model-selection.md) | RDB(강한 일관성, 복잡한 관계) vs NoSQL(확장성, 유연한 스키마) 선택 기준, 각 NoSQL 유형(Document/Key-Value/Column-Family/Graph) 적합한 사용 사례 |
| [07. 설계 면접 접근법](./fundamentals/07-interview-approach.md) | 요구사항 명확화(기능/비기능), 개략적 설계 먼저, 핵심 컴포넌트 상세화, 병목 식별과 해결, 면접관이 원하는 것은 완벽한 답이 아닌 트레이드오프 논의 |

</details>

<br/>

### 🔹 Chapter 2: Infrastructure — 핵심 인프라 컴포넌트

> **핵심 질문:** 로드밸런서 자체가 SPOF가 되지 않으려면? CDN이 없으면 이미지 서비스가 왜 느린가? 캐시를 어디에 놓아야 하는가?

<details>
<summary><b>DNS부터 스토리지 계층까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. DNS와 로드밸런서](./infrastructure/01-dns-load-balancer.md) | DNS가 지역별 트래픽을 분산하는 방법(GeoDNS), L4 vs L7 로드밸런서 차이, HAProxy vs Nginx vs AWS ALB, 로드밸런서 자체의 SPOF 제거 |
| [02. CDN(Content Delivery Network)](./infrastructure/02-cdn.md) | 정적 자원을 엣지 서버에 캐싱하는 원리, Push CDN vs Pull CDN, Cache-Control 헤더와 CDN 갱신 전략 |
| [03. 캐싱 계층 설계](./infrastructure/03-caching-layer.md) | 캐시를 어디에 놓는가(클라이언트/CDN/API Gateway/애플리케이션/DB), 캐시 히트율과 DB 부하의 관계, 캐시 갱신 전략(TTL/Event-Driven/Write-Through), 분산 캐시 구성 |
| [04. 메시지 큐와 비동기 처리](./infrastructure/04-message-queue.md) | 어떤 작업을 비동기로 분리해야 하는가, 메시지 큐가 없으면 트래픽 급증 시 무슨 일이 생기는가(백프레셔), Kafka vs RabbitMQ 선택 기준 |
| [05. 데이터베이스 확장](./infrastructure/05-database-scaling.md) | 읽기 복제본으로 읽기 트래픽 분산, 파티셔닝 vs 샤딩, 샤딩 키 선택의 중요성(Hot Shard 문제), CQRS와 DB 분리 |
| [06. 검색 인프라](./infrastructure/06-search-infrastructure.md) | 풀텍스트 검색에 DB LIKE 쿼리가 안 되는 이유, Elasticsearch 도입 시점, DB와 Elasticsearch 데이터 동기화 전략 |
| [07. 스토리지 계층](./infrastructure/07-storage-layer.md) | Block Storage vs Object Storage vs File Storage 선택 기준, S3 같은 Object Storage가 이미지/영상 저장에 적합한 이유, CDN + Object Storage 조합 |

</details>

<br/>

### 🔹 Chapter 3: Basic Services — 실전 설계 기본 서비스

> **핵심 질문:** URL 단축 키 충돌을 어떻게 처리하는가? 분산 Rate Limiter는 어떻게 구현하는가? 검색 자동완성을 어떻게 수백만 QPS로 처리하는가?

<details>
<summary><b>URL 단축기부터 검색 자동완성까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. URL 단축 서비스(bit.ly)](./basic-services/01-url-shortener.md) | 10억 개 URL 저장, 초당 10만 리디렉션, 단축 키 생성 전략(랜덤/Base62/Counter), 충돌 처리, 캐싱(핫 URL), 커스텀 URL, 분석 기능 |
| [02. 키-값 저장소 설계](./basic-services/02-key-value-store.md) | 분산 해시 테이블, 일관성 해싱으로 노드 추가/삭제 시 리밸런싱 최소화, 복제 전략, 가용성 vs 일관성, Gossip Protocol로 노드 상태 전파 |
| [03. 웹 크롤러 설계](./basic-services/03-web-crawler.md) | 수십억 페이지를 크롤링하는 분산 크롤러, 중복 URL 감지(BloomFilter), 크롤링 우선순위(Page Rank), Robots.txt 준수, 스케일아웃 전략 |
| [04. 분산 ID 생성기](./basic-services/04-distributed-id.md) | UUID vs DB Auto Increment vs Snowflake ID(Twitter 방식 — 타임스탬프+서버ID+시퀀스), 순서 보장 필요 여부별 선택 기준 |
| [05. Rate Limiter 설계](./basic-services/05-rate-limiter.md) | Token Bucket vs Leaky Bucket vs Fixed Window vs Sliding Window 비교, 분산 환경에서 Rate Limiter(Redis + Lua Script), API Gateway 구현 |
| [06. 알림 시스템](./basic-services/06-notification-system.md) | Push(iOS APNs/Android FCM)/SMS/이메일 채널, 메시지 큐로 알림 발송 비동기화, 재시도 전략, 알림 중복 방지(멱등성) |
| [07. 검색 자동완성(Typeahead)](./basic-services/07-typeahead.md) | 트라이(Trie) 자료구조로 접두사 검색, Top-K 사전 캐싱, 배치 업데이트(감쇠 가중치), 분산 트라이(Trie Sharding), 한국어 자모 처리 |

</details>

<br/>

### 🔹 Chapter 4: Large Scale Services — 실전 설계 대규모 서비스

> **핵심 질문:** 트위터 셀러브리티의 Fan-out 문제를 어떻게 해결하는가? WhatsApp 메시지 순서를 어떻게 보장하는가? 동시 수만 명 예약 시도 시 이중 예약을 어떻게 막는가?

<details>
<summary><b>동영상 스트리밍부터 위치 기반 서비스까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 동영상 스트리밍(YouTube/Netflix)](./large-scale-services/01-video-streaming.md) | 영상 업로드(S3 Presigned URL), 트랜스코딩 파이프라인(해상도별 비동기 변환), CDN 스트리밍, HLS 어댑티브 비트레이트(ABR), 조회수 비동기 집계 |
| [02. 소셜 피드/타임라인(Twitter/Instagram)](./large-scale-services/02-social-feed.md) | Fan-out on Write vs Fan-out on Read 트레이드오프, 셀러브리티(팔로워 1억) Hybrid 전략, Redis List 피드 캐시, 좋아요 Redis Counter |
| [03. 채팅 시스템(WhatsApp/Messenger)](./large-scale-services/03-chat-system.md) | 실시간 메시지 전달(WebSocket), Redis Presence 온라인 상태 관리, Cassandra 메시지 저장, 그룹 채팅 Kafka Pub/Sub |
| [04. 클라우드 스토리지(Google Drive/Dropbox)](./large-scale-services/04-cloud-storage.md) | 파일 청크 분할(4MB), SHA256 중복 제거, 델타 동기화(변경 청크만 전송), 버전 관리, 파일 충돌 해결 |
| [05. 검색 엔진(Google/Naver)](./large-scale-services/05-search-engine.md) | 역색인 구축, TF-IDF + PageRank 랭킹, Scatter-Gather 분산 검색, 인기 쿼리 Redis 캐싱, 실시간 색인 업데이트 |
| [06. 예약 시스템(Ticketmaster)](./large-scale-services/06-reservation-system.md) | Redis DECR 원자적 재고 감소, 임시 점유 TTL 10분, 피크 트래픽 대기열 시스템, 이중 예약 방지(DB 유니크 제약) |
| [07. 위치 기반 서비스(Uber/배달의민족)](./large-scale-services/07-location-service.md) | Redis GEO(GEOADD/GEORADIUS), 5초 위치 업데이트, GeoHash 근접 검색, Pub/Sub 실시간 위치 추적, 드라이버 배정 ETA |

</details>

<br/>

### 🔹 Chapter 5: Data Intensive Patterns — 데이터 집약적 시스템 패턴

> **핵심 질문:** 배치 처리와 스트림 처리는 언제 선택하는가? Cache Stampede를 어떻게 막는가? 글로벌 다중 데이터센터에서 일관성을 어떻게 유지하는가?

<details>
<summary><b>데이터 파이프라인부터 글로벌 분산까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 데이터 파이프라인 설계](./data-intensive-patterns/01-data-pipeline.md) | 배치 처리(Spark) vs 스트림 처리(Flink), Lambda Architecture(배치+스트림), Kappa Architecture(스트림만), ML Feature Store(Online/Offline) |
| [02. 시계열 데이터 처리](./data-intensive-patterns/02-time-series.md) | 시계열 DB(InfluxDB/ClickHouse)의 Append-Only 특성, Rollup 다운샘플링(7일→1달→1년), 고카디널리티 문제, Flink 임계값 알림 |
| [03. 분산 캐시 심화](./data-intensive-patterns/03-distributed-cache.md) | Redis Cluster 16384 슬롯, Hash Tag로 관련 키 동일 노드 배치, Redlock 분산 잠금(5노드), PER 알고리즘(Cache Stampede 방지) |
| [04. 샤딩 전략 심화](./data-intensive-patterns/04-sharding-deep-dive.md) | Hash/Range/Directory/Consistent 샤딩 비교, 크로스-샤드 쿼리(Scatter-Gather), Hot Shard 해결, 온라인 리샤딩 절차 |
| [05. 글로벌 분산 시스템](./data-intensive-patterns/05-global-distribution.md) | Active-Active vs Active-Passive, CRDT(G-Counter/LWW), 비동기 복제 RPO 트레이드오프, GeoDNS Failover, GDPR 리전 격리 |

</details>

<br/>

### 🔹 Chapter 6: Reliability — 신뢰성과 장애 대응

> **핵심 질문:** Circuit Breaker가 없으면 장애가 어떻게 전파되는가? 분산 환경에서 2PC 없이 일관성을 달성하는 방법은? RPO와 RTO를 어떻게 설정하는가?

<details>
<summary><b>장애 허용 설계부터 포스트모텀까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 장애 허용 설계 원칙](./reliability/01-fault-tolerance.md) | Circuit Breaker(CLOSED/OPEN/HALF-OPEN), Bulkhead 쓰레드 풀 격리, Fallback 계층(캐시→정적→빈 응답), Chaos Engineering 안전한 실험 절차 |
| [02. 데이터 일관성 보장](./reliability/02-data-consistency.md) | Saga 패턴(Choreography vs Orchestration), 보상 트랜잭션, 멱등성 키(Redis SETNX), Outbox Pattern(DB+이벤트 원자성) |
| [03. 모니터링과 경보](./reliability/03-monitoring.md) | SLI/SLO/에러 버짓, Alert Fatigue 방지(지속적 문제만+그루핑), 분산 추적(OpenTelemetry+Jaeger), 구조화 로그(JSON+trace_id) |
| [04. 재해 복구(DR) 전략](./reliability/04-disaster-recovery.md) | RPO/RTO 정의, Backup-Restore/Pilot Light/Warm Standby/Active-Active 4전략 비용 비교, Terraform 자동화, 정기 DR 훈련 |
| [05. 포스트모텀(Post-Mortem)](./reliability/05-postmortem.md) | Blameless 문화(개인 탓 → 시스템 개선), 5-Why 근본 원인 분석, 포스트모텀 템플릿, 48시간 내 작성, 액션 아이템 추적 |

</details>

<br/>

### 🔹 Chapter 7: Decision Framework — 설계 결정 프레임워크

> **핵심 질문:** 기술 선택의 이유를 어떻게 문서화하는가? 스타트업에서 대기업까지 각 성장 단계를 전환하는 실제 트리거는? 클라우드 비용이 폭발하는 이유는?

<details>
<summary><b>ADR부터 비용 최적화까지 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 아키텍처 결정 기록(ADR)](./decision-framework/01-adr.md) | 기술 선택의 이유를 문서화하는 방법, ADR 생명주기(Proposed→Accepted→Superseded), MADR 템플릿과 실제 예시(SQS 선택, Cache-Aside 채택) |
| [02. 기술 부채 관리](./decision-framework/02-tech-debt.md) | 의도적/비의도적 부채 분류, SonarQube 정량 측정, Sprint 20% 예약, 보이 스카우트 규칙, Strangler Fig 점진적 교체 |
| [03. 성장 단계별 아키텍처](./decision-framework/03-growth-path.md) | DAU 1천→1만→100만→1천만→그 이상, 단일 서버→Read Replica→캐시→마이크로서비스→글로벌 분산 각 전환 트리거 |
| [04. 비용 최적화(FinOps)](./decision-framework/04-cost-optimization.md) | On-Demand/Reserved/Spot 구매 전략, 개발 환경 야간 종료(70% 절감), S3 Lifecycle 롤업(83% 절감), Right-sizing, 팀별 태깅 |

</details>

---

## 🧭 학습 순서 가이드

```
┌─────────────────────────────────────────────────────────────┐
│                    권장 학습 경로                             │
│                                                             │
│  선행: IQ Dev Lab 각 기술 레포                               │
│  (redis-deep-dive / kafka-deep-dive / database-internals)   │
│                        │                                    │
│                        ▼                                    │
│            fundamentals — 기초 원칙 (필수)                   │
│                        │                                    │
│                        ▼                                    │
│            infrastructure — 인프라 컴포넌트 (필수)            │
│                        │                                    │
│            ┌───────────┤                                    │
│            │           │                                    │
│            ▼           ▼                                    │
│      basic-services  large-scale-services                   │
│       (URL단축,        (유튜브,                              │
│        Rate Limit)      트위터)                              │
│            │           │                                    │
│            └───────────┘                                    │
│                        │                                    │
│                        ▼                                    │
│     data-intensive-patterns / reliability                   │
│                        │                                    │
│                        ▼                                    │
│              decision-framework                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📌 모든 문서의 공통 구조

이 레포의 모든 42개 문서는 동일한 구조를 따릅니다.

```markdown
## 🎯 설계 목표와 요구사항
## 📊 규모 산정 (DAU, 초당 요청 수, 저장 용량)
## 🏗️ 개략적 설계 (High-Level Architecture)
## 🔬 핵심 컴포넌트 상세 설계
## ⚡ 병목 식별과 해결
## 🔄 데이터 흐름 (주요 시나리오별)
## ⚖️ 트레이드오프 (왜 이 기술을 선택했는가)
## 🚀 확장 전략 (10배 트래픽이 오면 어떻게 대응하는가)
## 📌 핵심 결정 요약
## 🤔 심화 질문 (면접에서 나올 수 있는 후속 질문 + 답)
```

---

## 🔗 IQ Dev Lab 연결 지점

이 레포는 독립적으로 완결되지 않습니다. 설계 결정의 근거는 각 기술 레포의 내부 원리에 있습니다.

| 설계 결정 | 참조 레포 |
|-----------|----------|
| 캐싱 계층에 Redis를 선택하는 이유 | `redis-deep-dive` — 단일 스레드 이벤트 루프, maxmemory 정책 |
| 메시지 큐로 Kafka를 선택하는 이유 | `kafka-deep-dive` — 파티션 복제, Consumer Group, Offset 관리 |
| 읽기 복제본 분리 타이밍 | `database-internals` — WAL 기반 복제, MVCC |
| 샤딩 키 선택 기준 | `database-internals` — B-Tree 인덱스, 쿼리 플랜 |
| Elasticsearch 도입 시점 | `elasticsearch-deep-dive` — 역색인, 분산 샤드 |
| 분산 환경 네트워크 설계 | `network-deep-dive` — TCP/IP, TLS, HTTP/2 |

---

## 📚 참고 자료

- **System Design Interview – An Insider's Guide** (Alex Xu) Vol.1 & Vol.2
- **Designing Data-Intensive Applications** (Martin Kleppmann) — 분산 시스템의 바이블
- **Building Microservices**, 2nd Edition (Sam Newman)
- **The Art of Scalability** (Martin Abbott, Michael Fisher)
- [High Scalability 블로그](http://highscalability.com/) — 실제 기업 아키텍처 사례
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [Netflix Tech Blog](https://netflixtechblog.com/)
- [Uber Engineering Blog](https://www.uber.com/en-KR/blog/engineering/)

---

<div align="center">

**총 42개 문서 · 7개 디렉토리**

*"정답은 없고 트레이드오프만 있다"*

</div>
