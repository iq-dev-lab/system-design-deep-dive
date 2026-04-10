# 검색 자동완성 (Typeahead / Autocomplete)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 사용자가 타이핑할 때마다 실시간 검색어 후보 5개 반환
- 최근 검색어가 아닌 인기 검색어 기준 (전체 사용자 집계)
- 한국어 포함 다국어 지원

### 비기능 요구사항
- DAU 1,000만, 검색창 평균 5회 타이핑 → 초당 약 600건 쿼리
- 응답 지연시간: 100ms 이하 (타이핑감 유지)
- 검색어 순위는 실시간이 아닌 주기적 업데이트 (수십 분 단위) 허용

---

## 📊 규모 산정

```
DAU: 1,000만
사용자당 하루 평균 검색: 10회
타이핑 중 평균 자동완성 호출: 5회/검색
→ 일일 자동완성 쿼리: 10만 × 10 × 5 = 5억 건
→ 초당 쿼리: 5억 / 86,400 ≈ 5,800 QPS (피크 ×3 = 17,000 QPS)

저장 용량:
  상위 검색어 수: 1,000만 개
  단어 평균 길이: 20자
  빈도 데이터: 8 bytes
  총 메모리(Trie): 약 2~5GB (공유 접두사로 압축됨)
```

---

## 🏗️ 개략적 설계

```
사용자 타이핑: "syst"
      │
      ▼
┌─────────────────┐
│  API Gateway    │
│  /autocomplete  │
│  ?q=syst        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐    Cache Hit  ┌──────────────┐
│  Autocomplete   │──────────────▶│  Redis       │
│  Service        │               │  (Trie Cache)│
└────────┬────────┘               └──────────────┘
         │ Cache Miss
         ▼
┌─────────────────┐
│  Trie 서버      │  인메모리 Trie
│  (Read 전용)    │  접두사 → Top-5 검색어
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Trie DB        │  주기적 스냅샷 저장
│  (영속성)        │
└─────────────────┘

[비동기] 검색 로그 수집 파이프라인:
사용자 검색 완료
      │
      ▼
  Kafka (search-log-topic)
      │
      ▼
  배치 집계 (Spark/Flink, 1시간마다)
      │
      ▼
  Trie 업데이트 → Trie 서버에 배포
```

---

## 🔬 핵심 컴포넌트 — Trie 자료구조

### Trie 기본 개념

```
Trie (트라이): 문자열 집합을 트리로 표현, 공통 접두사 공유

삽입 검색어: "system", "systematic", "sys", "sql", "scale"

         root
        /    \
       s      ...
      / \
     y   q
     |   |
     s   l
     |
     t───────┐
     |       |
     e    (sys 자체가 검색어)
     |
     m───────┐
     |       |
     (system) a
              |
              t
              |
              i
              |
              c
              |
             (systematic)

노드 구조:
  - children: Map<char, TrieNode>
  - isEnd: Boolean (검색어 완성 여부)
  - frequency: Long (검색 빈도)
  - topK: List<String> (이 노드를 접두사로 하는 상위 K개 검색어)
```

### Top-K 저장 최적화

```
문제: "s" 입력 시 모든 s로 시작하는 단어를 탐색 → 느림

해결: 각 노드에 Top-K 결과를 미리 캐싱

     root
      │
      s ─── topK: ["system", "search", "sql", "scale", "sort"]
      │
      y ─── topK: ["system", "systematic", "synergy", ...]
      │
      s ─── topK: ["system", "systematic"]

탐색:
  "sys" 입력 → sys 노드로 이동 (O(접두사 길이))
  → topK 반환 (O(1))

빈도 업데이트 시:
  단어의 모든 조상 노드 topK 갱신 필요
  → 업데이트 비용: O(단어 길이 × K log K)
```

### Trie 구현 예시

```python
from collections import defaultdict
import heapq

class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False
        self.frequency = 0
        self.top_k = []  # (frequency, word) 최대 힙

class Trie:
    def __init__(self, k=5):
        self.root = TrieNode()
        self.k = k
    
    def insert(self, word: str, frequency: int):
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
            # 각 노드에 top-k 업데이트
            self._update_top_k(node, word, frequency)
        node.is_end = True
        node.frequency = frequency
    
    def _update_top_k(self, node, word, frequency):
        # top_k를 (frequency, word) 튜플로 관리
        node.top_k.append((frequency, word))
        node.top_k.sort(reverse=True)
        node.top_k = node.top_k[:self.k]
    
    def search(self, prefix: str) -> list:
        node = self.root
        for char in prefix:
            if char not in node.children:
                return []  # 접두사 없음
            node = node.children[char]
        # top-k 바로 반환 (O(1))
        return [word for freq, word in node.top_k]

# 사용 예시
trie = Trie(k=5)
trie.insert("system", 1000)
trie.insert("systematic", 800)
trie.insert("search", 1500)

print(trie.search("sys"))
# → ["system", "systematic"]
```

---

## 🔄 데이터 흐름 — 검색어 빈도 업데이트

```
실시간 업데이트 vs 배치 업데이트 비교:

실시간 업데이트:
  장점: 최신 트렌드 즉시 반영
  단점: 검색마다 Trie 업데이트 → 쓰기 경합, 잠금 복잡
  → 초당 수천 건 쓰기 시 Trie 전체 잠금이 병목

배치 업데이트 (권장):
  1. 검색 로그 → Kafka에 비동기 수집
  2. Spark/Flink가 1시간마다 집계
     (검색어 빈도 카운트 + 감쇠 가중치)
  3. 새 Trie 빌드 (별도 서버에서)
  4. 완성된 Trie를 Trie 서버에 롤링 배포
  
  장점: 읽기 성능 유지, 업데이트 격리
  단점: 최대 1시간 지연 (실시간 트렌드 반영 느림)

감쇠 가중치 (시간에 따른 인기도 감소):
  score = frequency × e^(-λ × elapsed_hours)
  
  최근 1시간 내 검색: score 높음
  3일 전 검색어: score 대폭 감소
  → 오래된 인기어가 영원히 상위 유지되는 문제 방지
```

---

## ⚡ 병목 식별과 해결

```
병목 1: 단일 Trie 서버 처리량
  문제: 초당 17,000 QPS를 단일 서버가 처리 어려움
  
  해결: Trie 샤딩
  방법 1: 알파벳 첫 글자 기준
    a-f → Trie 서버 1
    g-m → Trie 서버 2
    n-s → Trie 서버 3
    t-z → Trie 서버 4
    
  방법 2: 트래픽 분석 기반
    's'로 시작하는 검색어가 많다면 → 'sa-sm', 'sn-sz'로 세분화

병목 2: 다국어 처리 (한국어)
  한국어 자동완성: 자모 단위로 처리
  "시스" 입력 → ㅅ,ㅣ,ㅅ,ㅡ 각 자모를 Trie 키로 사용
  
  또는 초성 검색: "ㅅㅅ" → "시스템", "소설", "사실" 반환
  → 별도 초성 Trie 구축

병목 3: 메모리 부족
  해결: Trie 직렬화 → Redis에 저장 (Redis Trie)
  또는 prefix → top-k 결과만 Redis Hash로 저장:
    Redis HSET "prefix:sys" "results" '["system","systematic"]'
    → Trie 없이 Redis 조회만으로 처리
    → 접두사가 많으면 메모리 사용량 증가 (절충)
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| 업데이트 방식 | 배치 (1시간) | 읽기 성능 우선, 최신성 다소 포기 |
| 자료구조 | Trie + Top-K 캐싱 | 접두사 탐색 O(L), 결과 반환 O(1) |
| 샤딩 | 첫 글자 기준 | 균등 분산이 어려울 수 있으나 구현 단순 |
| Redis 캐시 | 접두사 → 결과 직접 캐시 | Trie 서버 부하 감소, 인기 접두사 즉시 반환 |

---

## 🚀 확장 전략

```
읽기 확장:
  Trie 서버 Read Replica 추가 (읽기 전용 복제)
  → 샤딩된 Trie 서버 각각에 Replica 구성

Redis 캐시 계층:
  인기 접두사 (상위 1만 개) → Redis에 TTL=5분으로 캐시
  롱테일 접두사 → Trie 서버 직접 조회

글로벌 서비스:
  지역별 인기 검색어가 다름
  → 리전별 Trie 구성 (미국 vs 한국 인기어 분리)
  → GeoDNS로 가까운 Trie 서버 라우팅
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 자료구조: Trie + 각 노드에 Top-K 사전 계산 (접두사 탐색 O(L), 결과 O(1))
  2. 업데이트: 배치 (1시간 주기, Kafka→Spark→Trie 재빌드)
  3. 빈도: 감쇠 가중치 적용 (최신 검색어 우대)
  4. 확장: 첫 글자 기준 샤딩 + Redis 캐시 레이어
  5. 한국어: 자모 단위 Trie 또는 초성 검색 전용 Trie 별도 구성
```

---

## 🤔 심화 질문

**Q1. Trie 대신 DB 쿼리로 자동완성을 구현하면 어떤 문제가 있는가?**
> `SELECT * FROM searches WHERE keyword LIKE 'sys%' ORDER BY frequency DESC LIMIT 5` 쿼리는 인덱스를 타지만 초당 수천 건의 실시간 타이핑 이벤트를 처리하면 DB에 큰 부하가 됩니다. Trie는 인메모리에서 O(접두사 길이)로 탐색하므로 응답이 100μs 이하로 가능합니다. DB는 네트워크 왕복만 해도 수 ms가 걸립니다.

**Q2. 검색어 빈도 업데이트를 실시간으로 하려면 어떻게 설계하는가?**
> Trie를 공유 메모리에 두고 읽기-쓰기 잠금(RWLock)을 사용하거나, Segment Locking으로 노드 단위 잠금을 적용할 수 있습니다. 또는 버전 기반으로 "현재 Trie"와 "업데이트 중인 Trie"를 분리해 업데이트 완료 후 원자적 교체(atomic swap)하는 방식을 씁니다. 하지만 복잡도가 높아 대부분 배치 방식을 선택합니다.

**Q3. 면접에서 "Google 검색 자동완성을 설계하라"고 하면 어디에 집중하는가?**
> 두 가지 핵심에 집중합니다. 첫째, Trie에 Top-K를 미리 저장해 빠른 접두사 탐색을 가능하게 하는 자료구조 설계. 둘째, 초당 수만 건의 쿼리를 처리하기 위한 Trie 샤딩과 Redis 캐시 계층. 업데이트는 1시간 배치로 처리하되, 감쇠 가중치로 트렌드를 반영하는 이유를 설명하면 좋습니다.

---

<div align="center">

[⬅️ 이전: 알림 시스템](./06-notification-system.md) | [README로 돌아가기](../README.md) | [다음: 대규모 서비스 ➡️](../large-scale-services/01-video-streaming.md)

</div>
