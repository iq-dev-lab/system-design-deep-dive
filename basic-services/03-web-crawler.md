# 웹 크롤러 설계

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 수십억 개의 웹 페이지 크롤링
- 새로 추가되거나 변경된 페이지 주기적 재크롤링
- 이미 방문한 URL 중복 방지
- Robots.txt 규칙 준수
- HTML, PDF, 이미지 등 다양한 콘텐츠 타입 지원

### 비기능 요구사항
- 확장성: 수십억 페이지를 수주 내 크롤링
- 예의 (Politeness): 동일 도메인에 과부하 주지 않음
- 안정성: 잘못된 URL, 무한 루프 처리
- 확장 가능한 아키텍처

---

## 📊 규모 산정

```
목표: 10억 페이지 크롤링 (1개월 내)

초당 크롤링 수:
  10억 / (30일 × 86,400초) ≈ 385 페이지/초 ≈ 400 페이지/초

저장 용량:
  HTML 평균 크기: 100KB
  10억 페이지 × 100KB = 100TB (원본)
  메타데이터만 저장: 10억 × 500 bytes = 500GB

대역폭:
  400 페이지/초 × 100KB = 40MB/s = 320Mbps
  → 고대역폭 네트워크 필요
```

---

## 🏗️ 개략적 설계

```
                        ┌─────────────────┐
                        │  Seed URLs      │  (크롤링 시작점)
                        └────────┬────────┘
                                 │
┌────────────────────────────────▼───────────────────────────────────┐
│                        URL Frontier                                │
│                   (크롤링 대기 큐)                                  │
│  Priority Queue: 우선순위 높은 URL 먼저                             │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │    Fetcher (다운로더)    │
                    │    - Robots.txt 확인     │
                    │    - HTTP GET 요청       │
                    │    - 속도 제한 (예의)    │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │    Content Parser       │
                    │    - HTML 파싱           │
                    │    - URL 추출           │
                    │    - 중복 검사           │
                    └────┬───────────┬────────┘
                         │           │
              ┌───────────▼─┐   ┌────▼──────────┐
              │  Content    │   │  URL Frontier  │
              │  Storage    │   │  (새 URL 추가)  │
              │  (S3)       │   └────────────────┘
              └─────────────┘
```

---

## 🔬 핵심 컴포넌트 상세 설계

### URL Frontier — 크롤링 대기 큐

```
단순 FIFO가 아닌 두 가지 기준으로 관리:

1. 예의 (Politeness):
   같은 도메인에 너무 빨리 요청하지 않음
   
   도메인별 큐 분리:
   Queue-google.com:  [url1, url2, url3]
   Queue-naver.com:   [url4, url5]
   Queue-github.com:  [url6]
   
   각 큐는 최소 N초 간격으로 소비 (보통 300ms~1초)
   → 같은 도메인에 초당 최대 1~3 요청만 전송

2. 우선순위 (Priority):
   중요한 페이지 먼저 크롤링
   
   우선순위 기준:
   ├── PageRank (많이 링크된 페이지)
   ├── 업데이트 빈도 (뉴스 사이트 > 개인 블로그)
   ├── 사이트 권위도 (wikipedia.org > 개인블로그)
   └── 신선도 (마지막 크롤링 후 경과 시간)
   
   ┌─────────────────────────────────────────────────────┐
   │  Prioritizer                                        │
   │  URL 입력 → 우선순위 계산 → 우선순위별 큐에 저장    │
   └──────────────────────────────────────────────────── ┘
         │
   High Priority Queue ─────┐
   Medium Priority Queue ──▶│── Front Queue Router ──▶ Fetcher
   Low Priority Queue ──────┘
         │
   Back Queue (도메인별)
```

### URL 중복 방지 — BloomFilter

```
문제: 10억 URL 방문 여부를 어떻게 빠르게 확인하는가?

방법 1: HashSet
  10억 URL × 64 bytes = 64GB 메모리 필요 → 불가능

방법 2: Bloom Filter (확률적 자료구조)
  m = 비트 배열 크기 (예: 1억 비트 = 12.5MB)
  k = 해시 함수 개수

  삽입:
  URL → hash1(URL) = 위치 23 → 비트 1로 설정
       hash2(URL) = 위치 456 → 비트 1로 설정
       hash3(URL) = 위치 789 → 비트 1로 설정

  조회:
  URL → hash1, hash2, hash3 위치 확인
  모두 1이면 → "방문했을 가능성 있음" (False Positive 가능)
  하나라도 0이면 → "방문하지 않음" (확실)

특성:
  ✅ False Negative 없음 (방문한 URL을 미방문으로 판단 안 함)
  ❌ False Positive 존재 (미방문 URL을 방문했다고 오판, 1% 미만)
  ✅ 10억 URL을 약 1.2GB로 표현 가능 (HashSet의 1/50)

오판 허용: 일부 URL을 중복으로 보고 크롤링 안 해도 치명적이지 않음
          (완벽하지 않아도 되는 시스템에 적합)
```

### Robots.txt 처리

```
각 도메인 첫 크롤링 전 robots.txt 확인:
  GET https://www.example.com/robots.txt

  User-agent: *
  Disallow: /admin/         ← 크롤링 금지
  Disallow: /private/
  Crawl-delay: 1            ← 초당 1회 제한
  Allow: /public/

처리:
  1. robots.txt를 캐시에 저장 (TTL=24시간)
  2. URL 크롤링 전 허용 여부 확인
  3. Crawl-delay 준수
  4. 로봇이 아니라 합법적 크롤러임을 User-Agent 헤더로 명시
```

---

## 🔄 데이터 흐름

```
1. Seed URLs → URL Frontier에 추가
2. URL Frontier → 도메인별 큐에서 URL 선택 (예의 + 우선순위)
3. Fetcher → Robots.txt 확인 → HTTP GET
4. 응답 수신 → Content Parser로 전달
5. Parser:
   a. 콘텐츠 해시 계산 → 중복 콘텐츠 감지 (동일 내용 다른 URL)
   b. HTML에서 링크 추출
   c. 링크를 URL Bloom Filter로 중복 검사
   d. 새 URL → 우선순위 계산 → URL Frontier 추가
6. 콘텐츠 → S3 저장 (원본 HTML)
7. 메타데이터 → DB 저장 (URL, 크롤링 시각, 상태)
```

---

## ⚡ 병목 식별과 해결

```
병목 1: 단일 Fetcher의 처리량 한계
  해결: 여러 Fetcher 서버 수평 확장
        각 Fetcher는 다른 도메인 담당 (도메인별 큐로 분리)

병목 2: 무한 크롤링 (루프 감지)
  a.com → b.com → c.com → a.com → (무한)
  
  해결:
  ├── URL 정규화 (https://a.com, http://a.com → 동일하게 처리)
  ├── 방문 깊이 제한 (Seed에서 최대 N홉)
  └── BloomFilter로 방문 URL 추적

병목 3: 크롤링 트랩 (Spider Trap)
  동적 URL이 무한 생성:
  example.com/page/1, /page/2, /page/3 ... /page/999999
  
  해결:
  ├── 같은 도메인의 URL 수 제한 (최대 10만 URL/도메인)
  └── URL 파라미터 정규화
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| 중복 감지 | BloomFilter | 메모리 효율, False Positive 허용 |
| 우선순위 | PageRank + 신선도 | 중요한 페이지 먼저 처리 |
| 예의 | 도메인별 큐 + Crawl-delay | 대상 서버 과부하 방지 (법적 리스크 감소) |
| 저장 | S3 + DB 메타데이터 | 원본은 S3, 조회는 DB |

---

## 🚀 확장 전략

```
수평 확장:
  Fetcher 서버 추가 (각 서버가 다른 도메인 담당)
  URL Frontier: Kafka로 분산 큐 구성
  BloomFilter: Redis Cluster로 분산

지리적 분산:
  크롤링 대상 서버와 같은 리전에 Fetcher 배치
  → 지연시간 감소, 네트워크 비용 절감
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. URL 중복 방지: BloomFilter (메모리 효율, False Positive 허용)
  2. 예의: 도메인별 큐 + Crawl-delay 준수
  3. 우선순위: PageRank + 업데이트 빈도 기반 점수
  4. 무한 루프: URL 정규화 + 방문 깊이 제한
  5. 저장: 원본 S3, 메타데이터 DB
```

---

## 🤔 심화 질문

**Q1. JavaScript 렌더링이 필요한 SPA는 어떻게 크롤링하는가?**
> Headless Chrome(Puppeteer)이나 Selenium으로 JavaScript를 실행한 후 DOM을 파싱합니다. 일반 HTTP 크롤링보다 리소스가 10~100배 더 소모되므로, 먼저 HTML만 크롤링하고 JS 렌더링이 필요한 사이트만 별도 큐에서 Headless Chrome으로 처리합니다.

**Q2. 저작권 있는 콘텐츠 크롤링 시 법적 문제는?**
> Robots.txt를 준수하고 저작권 있는 콘텐츠는 원본 URL만 저장합니다. 콘텐츠 자체를 저장·서빙하지 않고, 검색 색인용으로만 파싱합니다. 또한 User-Agent에 크롤러 정보를 명시하고 연락처를 제공합니다.

**Q3. 면접에서 "URL 중복 방지를 어떻게 할 것인가"라고 하면?**
> "10억 URL을 HashSet으로 메모리에 저장하면 64GB가 필요합니다. Bloom Filter를 사용하면 오판율 1% 이하로 1.2GB에 처리합니다. 일부 URL이 중복으로 인식돼 크롤링 안 되어도 치명적이지 않으므로 False Positive를 허용합니다. 추가로 URL을 정규화(소문자, 파라미터 정렬)해 다른 형태의 같은 URL도 중복 처리합니다."

---

<div align="center">

[⬅️ 이전: 키-값 저장소 설계](./02-key-value-store.md) | [README로 돌아가기](../README.md) | [다음: 분산 ID 생성기 ➡️](./04-distributed-id.md)

</div>
