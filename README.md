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
[![Ch4](https://img.shields.io/badge/🔹_Large_Scale-유튜브_설계-2B7BB9?style=for-the-badge)](./large-scale-services/01-youtube-netflix.md)
[![Ch5](https://img.shields.io/badge/🔹_Data_Intensive-데이터_파이프라인-2B7BB9?style=for-the-badge)](./data-intensive-patterns/01-data-pipeline.md)
[![Ch6](https://img.shields.io/badge/🔹_Reliability-장애_허용_설계-2B7BB9?style=for-the-badge)](./reliability/01-fault-tolerance.md)
[![Ch7](https://img.shields.io/badge/🔹_Decision-ADR_아키텍처_결정-2B7BB9?style=for-the-badge)](./decision-framework/01-adr.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Fundamentals — 시스템 설계의 기초 원칙

> **핵심 질문:** 수직 확장의 한계는 언제 오는가? 99.9%와 99.99% 가용성의 실제 차이는? CAP Theorem은 어떤 설계 결정에 어떻게 적용되는가?

<details>
<summary><b>확장성부터 설계 면접 접근법까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| 01. 확장성(Scalability) 완전 분해 | 수직 확장(Scale Up)의 물리적 한계, 수평 확장이 가능하려면 무엇이 달라야 하는가(Stateless), 로드밸런서 분산 알고리즘(Round Robin/Least Connection/IP Hash), Sticky Session의 문제 |
| 02. 가용성(Availability) 계산 | 99.9%(3 Nine) vs 99.99%(4 Nine)의 실제 다운타임 차이(연간 8.7시간 vs 52분), SLA와 SLO, 단일 장애점(SPOF) 제거 전략, 이중화(Redundancy)의 비용 |
| 03. 일관성 vs 가용성 | CAP Theorem이 실제 설계 결정에 어떻게 적용되는가, CP 시스템(MySQL Master)과 AP 시스템(Redis Cluster) 선택 기준, Eventual Consistency를 허용하는 설계 |
| 04. 지연시간 vs 처리량 | P50/P95/P99 지연시간의 의미, 롱테일 지연시간 문제, 처리량을 높이면 지연시간이 올라가는 이유, 배치 처리 트레이드오프 |
| 05. 용량 산정(Capacity Estimation) | DAU/MAU로 초당 요청 수 계산, 저장 용량 산정(5년치), 네트워크 대역폭 계산, Back-of-the-Envelope 계산법과 면접 활용법 |
| 06. 데이터 모델 선택 | RDB(강한 일관성, 복잡한 관계) vs NoSQL(확장성, 유연한 스키마) 선택 기준, 각 NoSQL 유형(Document/Key-Value/Column-Family/Graph) 적합한 사용 사례 |
| 07. 설계 면접 접근법 | 요구사항 명확화(기능/비기능), 개략적 설계 먼저, 핵심 컴포넌트 상세화, 병목 식별과 해결, 면접관이 원하는 것은 완벽한 답이 아닌 트레이드오프 논의 |

</details>

<br/>

### 🔹 Infrastructure — 핵심 인프라 컴포넌트

> **핵심 질문:** 로드밸런서 자체가 SPOF가 되지 않으려면? CDN이 없으면 이미지 서비스가 왜 느린가? 캐시를 어디에 놓아야 하는가?

<details>
<summary><b>DNS부터 스토리지 계층까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| 01. DNS와 로드밸런서 | DNS가 지역별 트래픽을 분산하는 방법(GeoDNS), L4 vs L7 로드밸런서 차이, HAProxy vs Nginx vs AWS ALB, 로드밸런서 자체의 SPOF 제거 |
| 02. CDN(Content Delivery Network) | 정적 자원을 엣지 서버에 캐싱하는 원리, Push CDN vs Pull CDN, Cache-Control 헤더와 CDN 갱신 전략 |
| 03. 캐싱 계층 설계 | 캐시를 어디에 놓는가(클라이언트/CDN/API Gateway/애플리케이션/DB), 캐시 히트율과 DB 부하의 관계, 캐시 갱신 전략(TTL/Event-Driven/Write-Through), 분산 캐시 구성 |
| 04. 메시지 큐와 비동기 처리 | 어떤 작업을 비동기로 분리해야 하는가, 메시지 큐가 없으면 트래픽 급증 시 무슨 일이 생기는가(백프레셔), Kafka vs RabbitMQ 선택 기준 |
| 05. 데이터베이스 확장 | 읽기 복제본으로 읽기 트래픽 분산, 파티셔닝 vs 샤딩, 샤딩 키 선택의 중요성(Hot Shard 문제), CQRS와 DB 분리 |
| 06. 검색 인프라 | 풀텍스트 검색에 DB LIKE 쿼리가 안 되는 이유, Elasticsearch 도입 시점, DB와 Elasticsearch 데이터 동기화 전략 |
| 07. 스토리지 계층 | Block Storage vs Object Storage vs File Storage 선택 기준, S3 같은 Object Storage가 이미지/영상 저장에 적합한 이유, CDN + Object Storage 조합 |

</details>

<br/>

### 🔹 Basic Services — 실전 설계 기본 서비스

> **핵심 질문:** URL 단축 키 충돌을 어떻게 처리하는가? 분산 Rate Limiter는 어떻게 구현하는가? 검색 자동완성을 어떻게 수백만 QPS로 처리하는가?

<details>
<summary><b>URL 단축기부터 검색 자동완성까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| 01. URL 단축 서비스(bit.ly) | 10억 개 URL 저장, 초당 10만 리디렉션, 단축 키 생성 전략(랜덤/Base62/Counter), 충돌 처리, 캐싱(핫 URL), 커스텀 URL, 분석 기능 |
| 02. 키-값 저장소 설계 | 분산 해시 테이블, 일관성 해싱으로 노드 추가/삭제 시 리밸런싱 최소화, 복제 전략, 가용성 vs 일관성, Gossip Protocol로 노드 상태 전파 |
| 03. 웹 크롤러 설계 | 수십억 페이지를 크롤링하는 분산 크롤러, 중복 URL 감지(BloomFilter), 크롤링 우선순위(Page Rank), Robots.txt 준수, 스케일아웃 전략 |
| 04. 분산 ID 생성기 | UUID vs DB Auto Increment vs Snowflake ID(Twitter 방식 — 타임스탬프+서버ID+시퀀스), 순서 보장 필요 여부별 선택 기준 |
| 05. Rate Limiter 설계 | Token Bucket vs Leaky Bucket vs Fixed Window vs Sliding Window 비교, 분산 환경에서 Rate Limiter(Redis + Lua Script), API Gateway 구현 |
| 06. 알림 시스템 | Push(iOS APNs/Android FCM)/SMS/이메일 채널, 메시지 큐로 알림 발송 비동기화, 재시도 전략, 알림 중복 방지(멱등성) |
| 07. 검색 자동완성(Typeahead) | 트라이(Trie) 자료구조로 접두사 검색, 실시간 vs 배치 업데이트, 분산 트라이(Trie Sharding), 캐싱 전략 |

</details>

<br/>

### 🔹 Large Scale Services — 실전 설계 대규모 서비스

> **핵심 질문:** 트위터 셀러브리티의 Fan-out 문제를 어떻게 해결하는가? WhatsApp 메시지 순서를 어떻게 보장하는가? Twitch 라이브 스트리밍은 어떻게 동작하는가?

<details>
<summary><b>유튜브부터 라이브 스트리밍까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| 01. 유튜브/넷플릭스 설계 | 영상 업로드(멀티파트 + Object Storage), 인코딩 파이프라인(여러 해상도/코덱으로 비동기 변환), CDN 스트리밍, 어댑티브 비트레이트(ABR) |
| 02. 트위터 타임라인 설계 | Fan-out on Write vs Fan-out on Read 트레이드오프, 셀러브리티(팔로워 1000만)의 Fan-out 문제, 하이브리드 전략, Redis Sorted Set 타임라인 |
| 03. Facebook 뉴스피드 설계 | Edge Rank 알고리즘, 친구/그룹/페이지 게시물 랭킹, 실시간 업데이트 vs 배치 처리, 읽기 최적화 vs 쓰기 최적화 트레이드오프 |
| 04. 채팅 시스템(Slack/WhatsApp) | 실시간 메시지 전달(WebSocket vs Long Polling vs SSE), 온라인/오프라인 상태 관리, 1:1 채팅 vs 그룹 채팅 차이, 메시지 순서 보장 |
| 05. Google Drive / Dropbox 설계 | 파일 청크 업로드(대용량 파일), 중복 청크 감지(해시 기반), 델타 동기화(변경된 청크만), 파일 버저닝, 동시 편집 충돌 해결 |
| 06. 대규모 검색 엔진 | 웹 크롤러 + 인덱서 + 검색 서버 파이프라인, 역색인 분산 저장(Elasticsearch 클러스터), 페이지랭크 계산, 검색 결과 개인화 |
| 07. 라이브 스트리밍 플랫폼(트위치) | RTMP 스트림 수신, 트랜스코딩 파이프라인, HLS 세그먼트 생성, CDN 분산, Low Latency vs 안정성 트레이드오프, 채팅 동시성(초당 수만 메시지) |

</details>

<br/>

### 🔹 Data Intensive Patterns — 데이터 집약적 시스템 패턴

> **핵심 질문:** 배치 처리와 스트림 처리는 언제 선택하는가? Cache Stampede를 어떻게 막는가? 글로벌 다중 데이터센터에서 일관성을 어떻게 유지하는가?

<details>
<summary><b>데이터 파이프라인부터 글로벌 분산까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| 01. 데이터 파이프라인 설계 | 배치 처리(Hadoop MapReduce) vs 스트림 처리(Kafka Streams/Flink), Lambda Architecture(배치+스트림), Kappa Architecture(스트림만), 선택 기준 |
| 02. 대규모 데이터 집계 | 광고 클릭 집계(실시간 vs 배치), 시계열 DB(InfluxDB/TimescaleDB), 사전 집계(Pre-aggregation) vs 실시간 집계, 정확성 vs 속도 트레이드오프 |
| 03. 분산 캐시 심화 | 캐시 일관성 문제(Cache Invalidation이 어려운 이유), 분산 환경에서 캐시 갱신 전략, Cache Stampede 방지(Mutex/PER), Hot Key 분산(Local Cache + Redis) |
| 04. 샤딩 전략 심화 | 범위 기반 샤딩의 핫스팟 문제, 해시 기반 샤딩의 리밸런싱 문제, 일관성 해싱으로 리밸런싱 최소화, Vitess(MySQL 샤딩 미들웨어) 아키텍처 |
| 05. 글로벌 분산 시스템 | 다중 데이터센터 배포, 지역 간 데이터 복제(비동기 복제의 일관성 문제), 사용자를 가까운 DC로 라우팅(GeoDNS), 재해 복구(DR) 전략 |

</details>

<br/>

### 🔹 Reliability — 신뢰성과 장애 대응

> **핵심 질문:** Circuit Breaker가 없으면 장애가 어떻게 전파되는가? 분산 환경에서 2PC 없이 일관성을 달성하는 방법은? RPO와 RTO를 어떻게 설정하는가?

<details>
<summary><b>장애 허용 설계부터 포스트모텀까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| 01. 장애 허용 설계 원칙 | Chaos Engineering(의도적 장애 주입), Circuit Breaker 없으면 장애가 전파되는 시나리오, Bulkhead로 장애 격리, Timeout 없이 서비스를 호출하면 생기는 자원 고갈 |
| 02. 데이터 일관성 보장 | 분산 환경에서 2PC 없이 일관성 달성(Saga), 멱등성(Idempotency) 설계, Exactly-Once 처리 구현 패턴 |
| 03. 모니터링과 경보 | SLI 측정 항목 선택(에러율/지연시간/처리량), SLO 설정 기준, 경보 피로(Alert Fatigue) 방지, Runbook 작성 |
| 04. 재해 복구(DR) 전략 | RPO(데이터 손실 허용 시간)와 RTO(복구 목표 시간) 설정, 백업 전략(풀 백업/증분 백업/연속 복제), 복구 절차 자동화 |
| 05. 포스트모텀(Post-Mortem) | 장애 발생 후 원인 분석 방법, 비난 없는 문화(Blameless), 재발 방지 액션 아이템 도출, 5-Why 분석법 |

</details>

<br/>

### 🔹 Decision Framework — 설계 결정 프레임워크

> **핵심 질문:** 기술 선택의 이유를 어떻게 문서화하는가? 스타트업에서 대기업까지 각 성장 단계를 전환하는 실제 트리거는? 클라우드 비용이 폭발하는 이유는?

<details>
<summary><b>ADR부터 비용 최적화까지 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| 01. 아키텍처 결정 기록(ADR) | 기술 선택의 이유를 문서화하는 방법, 미래의 나와 팀을 위한 컨텍스트 보존, ADR 템플릿과 예시(왜 Kafka를 선택했는가, 왜 MySQL을 PostgreSQL 대신 썼는가) |
| 02. 기술 부채 관리 | 의도적 부채(빠른 출시를 위한 타협)와 비의도적 부채, 부채 측정과 상환 우선순위, 점진적 리팩터링 전략 |
| 03. 스타트업부터 대기업까지 성장 경로 | 단일 서버 → 읽기 복제본 분리 → 캐시 도입 → 샤딩 → 마이크로서비스 각 단계의 트리거(사용자 수/트래픽/팀 규모) |
| 04. 비용 최적화 | 클라우드 비용이 폭발하는 이유(오버프로비저닝, 비효율적 쿼리), Auto Scaling 설계, Reserved vs Spot Instance, 스토리지 계층화(Hot/Warm/Cold) |

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
