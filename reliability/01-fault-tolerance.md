# 장애 허용 설계 (Circuit Breaker / Bulkhead / Chaos Engineering)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 외부 서비스 장애 시 자동 차단 (Circuit Breaker)
- 장애 격리 (Bulkhead — 한 서비스 장애가 전체에 미치는 영향 최소화)
- 자동 복구 (Health Check + 점진적 트래픽 복원)
- 의도적 장애 주입으로 내성 검증 (Chaos Engineering)

### 비기능 요구사항
- 외부 서비스 장애 전파 차단: 5초 이내 감지
- 부분 장애 시 서비스 계속 운영 (Degraded Mode)
- Chaos 실험은 프로덕션에서 안전하게 실행

---

## 📊 장애의 종류와 영향

```
장애 유형:
  1. 서비스 크래시 (즉각 실패, 감지 쉬움)
  2. 느린 응답 (Slow Response) - 타임아웃, 감지 어려움
  3. 간헐적 오류 (Intermittent) - 재시도로 해결될 수 있음
  4. 네트워크 파티션 (연결 단절)

장애 전파 시나리오:
  서비스 A → 서비스 B (느린 응답)
  A가 B 응답 대기 중 → A의 쓰레드 모두 점유
  새 요청 → A 쓰레드 없음 → A도 느려짐
  서비스 C → A 호출 → C도 느려짐
  → 전체 시스템 카스케이드 장애!

없었다면:
  A → B 타임아웃 → 빠른 실패 반환 → A의 쓰레드 반환
  C → A 정상 응답 받음 (B 기능 일부 비활성화)
```

---

## 🏗️ Circuit Breaker 패턴

```
전기 회로 차단기에서 유래: 과전류 시 회로 차단 → 장치 보호

세 가지 상태:

┌─────────────────────────────────────────────────────┐
│  CLOSED (정상)                                       │
│  요청 정상 통과, 실패율 모니터링                        │
│  실패율 > 임계값 → OPEN 전환                          │
└─────────────────────┬───────────────────────────────┘
                      │ 실패율 > 50% (5초 내)
                      ▼
┌─────────────────────────────────────────────────────┐
│  OPEN (차단)                                         │
│  모든 요청 즉시 실패 반환 (외부 서비스 호출 없음)       │
│  설정 시간(30초) 후 → HALF-OPEN 전환                  │
└─────────────────────┬───────────────────────────────┘
                      │ 30초 후
                      ▼
┌─────────────────────────────────────────────────────┐
│  HALF-OPEN (탐색)                                    │
│  소수 요청만 통과 (10%)                               │
│  성공 → CLOSED 전환                                  │
│  실패 → OPEN으로 복귀                                 │
└─────────────────────────────────────────────────────┘
```

### Circuit Breaker 구현

```python
import time
from enum import Enum
from threading import Lock

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self,
                 failure_threshold=5,    # 연속 실패 N회
                 recovery_timeout=30,    # OPEN → HALF_OPEN 대기 시간
                 half_open_max_calls=3): # HALF_OPEN에서 허용 요청 수
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = None
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_calls = half_open_max_calls
        self.half_open_call_count = 0
        self._lock = Lock()
    
    def call(self, func, *args, **kwargs):
        with self._lock:
            if self.state == CircuitState.OPEN:
                if time.time() - self.last_failure_time > self.recovery_timeout:
                    self.state = CircuitState.HALF_OPEN
                    self.half_open_call_count = 0
                else:
                    raise Exception("Circuit OPEN: 빠른 실패")
            
            if self.state == CircuitState.HALF_OPEN:
                if self.half_open_call_count >= self.half_open_max_calls:
                    raise Exception("Circuit HALF_OPEN: 제한 초과")
                self.half_open_call_count += 1
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        with self._lock:
            self.failure_count = 0
            self.state = CircuitState.CLOSED
    
    def _on_failure(self):
        with self._lock:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN

# 사용
payment_cb = CircuitBreaker(failure_threshold=5, recovery_timeout=30)

def charge_user(user_id, amount):
    return payment_cb.call(payment_service.charge, user_id, amount)
```

---

## 🔬 Bulkhead 패턴

```
선박의 격벽(Bulkhead)에서 유래: 한 격실에 물이 차도 전체 침몰 방지

서비스 격리:

[쓰레드 풀 격리]
  결제 서비스 전용 쓰레드 풀: 10개 쓰레드
  검색 서비스 전용 쓰레드 풀: 20개 쓰레드
  알림 서비스 전용 쓰레드 풀: 5개 쓰레드
  
  결제 서비스 느려져 10개 쓰레드 모두 점유
  → 검색/알림 서비스 쓰레드는 영향 없음
  → 전체 서비스 다운 방지

[연결 풀 격리]
  각 외부 서비스마다 별도 HTTP 연결 풀
  결제 PG사 연결 풀: 최대 20개
  SMS 게이트웨이 연결 풀: 최대 10개
  
  PG사 장애로 20개 연결 모두 타임아웃 대기 중
  → SMS 연결 풀 영향 없음

Hystrix (Netflix) 구현:
  @HystrixCommand(
      threadPoolKey = "paymentPool",
      threadPoolProperties = {
          @HystrixProperty(name="coreSize", value="10"),
          @HystrixProperty(name="maxQueueSize", value="100")
      },
      fallbackMethod = "paymentFallback"
  )
  public String processPayment(PaymentRequest req) {
      return paymentService.process(req);
  }
  
  public String paymentFallback(PaymentRequest req) {
      // 결제 서비스 다운 시 대체 동작
      return "결제 서비스 일시 불가. 나중에 다시 시도하세요.";
  }
```

---

## 🔬 Chaos Engineering

```
넷플릭스에서 시작: "프로덕션에서 의도적으로 장애를 일으켜 시스템 내성 검증"

원칙:
  1. 정상 상태 정의: 시스템의 정상 메트릭 기준 수립
  2. 가설: "X가 장애나도 서비스는 정상 동작한다"
  3. 실험: 소규모 트래픽(5%)에 장애 주입
  4. 관찰: 메트릭이 정상 범위 이탈 시 즉시 중단
  5. 개선: 발견된 취약점 수정

Chaos 실험 유형:
  ├── 인스턴스 종료 (Chaos Monkey: 랜덤 EC2 종료)
  ├── 네트워크 지연 추가 (100ms, 500ms 주입)
  ├── 네트워크 패킷 손실 (10% 손실)
  ├── 리전 장애 (Chaos Kong: 전체 AWS 리전 차단)
  ├── CPU/메모리 스트레스
  └── 의존 서비스 오류 주입

도구:
  Chaos Monkey (Netflix OSS)
  Gremlin (상용)
  AWS Fault Injection Simulator
  Chaos Mesh (Kubernetes)
```

### Chaos 실험 절차

```
안전한 Chaos 실험 진행:

1. 사전 준비:
   - 정상 상태 메트릭 스냅샷 (에러율, 지연, 처리량)
   - 자동 중단 조건 정의 (에러율 > 1% → 즉시 종료)
   - 비즈니스 시간 외 실험 (트래픽 적은 시간)

2. 소규모 시작:
   - 전체 인스턴스 5%에만 적용
   - 점진적으로 비율 증가

3. 모니터링:
   - 실시간 대시보드 주시
   - 이상 감지 시 즉시 롤백

4. 문서화:
   - 발견된 취약점 → ADR 또는 장애 보고서 작성
   - 수정 → 재실험으로 검증

실험 예시:
  가설: "결제 서비스 인스턴스 1개가 다운돼도 결제 성공률 99.9% 유지"
  방법: 결제 서비스 인스턴스 중 1개 강제 종료
  관찰: 결제 성공률, 지연 시간 모니터링
  결과: 로드 밸런서가 장애 인스턴스 제거, 나머지 인스턴스로 처리
  → 가설 검증됨 ✅
```

---

## 🔄 데이터 흐름 — 장애 시 Fallback

```
정상 흐름:
  클라이언트 → 추천 서비스 → ML 모델 → 개인화 추천 반환

추천 서비스 장애 시 Fallback 계층:

단계 1: 캐시 Fallback
  ML 모델 응답 없음 → 30분 전 Redis 캐시 반환 (오래된 추천)
  
단계 2: 정적 Fallback
  Redis 캐시도 없음 → 인기 상품 TOP 10 반환 (개인화 없음)
  
단계 3: 빈 응답
  모든 Fallback 실패 → 빈 추천 목록 반환 (서비스는 계속 작동)

코드 패턴:
  def get_recommendations(user_id):
      try:
          # 1차: ML 추천 (200ms 타임아웃)
          return ml_service.recommend(user_id, timeout=0.2)
      except TimeoutError:
          try:
              # 2차: 캐시된 추천
              return cache.get(f"rec:{user_id}")
          except:
              # 3차: 인기 상품
              return popular_items.get_top_10()
```

---

## ⚖️ 트레이드오프

| 패턴 | 장점 | 단점 | 적용 시기 |
|------|------|------|----------|
| Circuit Breaker | 장애 전파 차단, 빠른 실패 | 구현 복잡, 튜닝 필요 | 외부 서비스 의존 시 |
| Bulkhead | 장애 격리, 전체 다운 방지 | 리소스 낭비 가능 | 중요도 다른 서비스 혼합 |
| Retry | 일시적 오류 복구 | Thundering Herd 위험 | 멱등한 연산에만 |
| Chaos Engineering | 실제 내성 검증 | 프로덕션 위험 | 성숙한 모니터링 갖춘 후 |

---

## 🚀 확장 전략

```
Retry with Exponential Backoff:
  1회 실패 → 1초 대기 후 재시도
  2회 실패 → 2초
  3회 실패 → 4초
  4회 실패 → 8초 (최대 대기 시간 설정)
  + Jitter (랜덤 ±30%) → Thundering Herd 방지

서비스 메시 (Istio / Linkerd):
  Circuit Breaker, Retry, Timeout을 서비스 메시 레벨에서 처리
  → 애플리케이션 코드 변경 없이 적용 가능
  → 모든 서비스에 일관된 정책 적용
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. Circuit Breaker: 외부 서비스 장애 시 빠른 실패 + 자동 복구
  2. Bulkhead: 서비스별 쓰레드/연결 풀 격리 (장애 격리)
  3. Fallback: 장애 시 대체 동작 (캐시 → 정적 → 빈 응답)
  4. Chaos Engineering: 의도적 장애 주입으로 내성 검증
  5. Retry + Backoff: 일시 오류는 지수 백오프 재시도
```

---

## 🤔 심화 질문

**Q1. Circuit Breaker의 임계값을 어떻게 설정하는가?**
> 서비스 SLA에 따라 결정합니다. 결제 서비스처럼 중요한 경우 실패율 1%에서 OPEN합니다. 추천 서비스처럼 덜 중요한 경우 10~20%로 완화합니다. Recovery Timeout은 외부 서비스의 평균 복구 시간보다 약간 길게 설정합니다(보통 30~60초). Hystrix Dashboard나 Resilience4j Metrics로 실제 실패율 분포를 보고 결정합니다.

**Q2. Chaos Engineering을 아직 모니터링이 미숙한 팀이 도입하면 어떤 위험이 있는가?**
> 장애를 주입했는데 감지하지 못하고 실제 장애로 이어질 수 있습니다. 또한 자동 롤백 조건을 설정하지 않으면 실험이 통제 불능이 됩니다. 따라서 Chaos Engineering 전에 완성된 모니터링(에러율, 지연, 가용성 대시보드), 자동 롤백 메커니즘, 명확한 중단 조건 정의, 팀 전체 동의가 필요합니다. 개발 환경에서 먼저 실험하고 점진적으로 프로덕션에 도입합니다.

**Q3. Netflix의 Chaos Monkey가 실제로 프로덕션에서 인스턴스를 종료해도 괜찮은 이유는?**
> Netflix 아키텍처가 처음부터 인스턴스 장애를 가정하고 설계됐기 때문입니다. 모든 서비스는 여러 인스턴스로 실행되며, Load Balancer가 장애 인스턴스를 즉시 제거합니다. 마이크로서비스는 Circuit Breaker와 Fallback을 갖추고 있어 일부 인스턴스 종료가 사용자 경험에 영향을 주지 않습니다. Chaos Monkey 없이는 이 설계가 실제로 동작하는지 알 수 없습니다.

---

<div align="center">

[⬅️ 이전: 글로벌 분산 시스템](../data-intensive-patterns/05-global-distribution.md) | [README로 돌아가기](../README.md) | [다음: 데이터 일관성 ➡️](./02-data-consistency.md)

</div>
