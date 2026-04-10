# 비용 최적화 (FinOps)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 클라우드 비용 가시화 (서비스별, 팀별)
- 낭비 리소스 자동 감지 및 제거
- 비용-성능 최적화 (Right-sizing)
- 예산 초과 알림

### 비기능 요구사항
- 비용 절감 목표: 전체 클라우드 지출의 20~30%
- 비용 가시화 지연: 익일 (실시간 어려움)
- 비용 최적화로 인한 성능 저하 없음

---

## 📊 클라우드 비용 구조

```
일반적인 클라우드 비용 분포:

EC2 (컴퓨팅): 40~50%
  └── 개발/스테이징 환경 과다 사용
  └── 사용률 낮은 인스턴스 (CPU < 10%)

RDS/DB: 20~30%
  └── 과도하게 큰 인스턴스 (프로덕션 스펙 그대로 스테이징)

S3/스토리지: 10~15%
  └── 삭제 안 된 스냅샷, 로그 무기한 보관

데이터 전송: 5~10%
  └── 크로스-리전 데이터 전송

ELB/API GW: 5%

첫 번째 절감 타깃: 개발 환경, 사용률 낮은 인스턴스
```

---

## 🏗️ 비용 최적화 전략

### 1. 인스턴스 구매 방식 최적화

```
On-Demand vs Reserved vs Spot:

On-Demand (기본):
  언제든 시작/중단 가능
  가장 비쌈 (기준가)
  적합: 예측 불가 워크로드, 단기 실험

Reserved Instance (예약):
  1년 또는 3년 약정
  1년 약정: On-Demand 대비 30~40% 절감
  3년 약정: 55~60% 절감
  
  Convertible Reserved: 인스턴스 타입 변경 가능 (유연성 ↑)
  Standard Reserved: 변경 불가 (절감 ↑)
  
  적합: 24/7 실행 서비스 (DB, 상시 API 서버)

Spot Instance:
  AWS 여유 용량 사용 → 70~90% 절감
  단점: 2분 경고 후 종료 가능 (중단 허용 필요)
  
  적합:
  ├── 배치 처리 (데이터 파이프라인, ML 학습)
  ├── 트랜스코딩 Worker
  ├── CI/CD 빌드 서버
  └── 오토스케일링 그룹의 일부 (On-Demand + Spot 혼합)

혼합 전략:
  베이스라인: Reserved (항상 필요한 만큼)
  피크 처리: Spot Instance (중단 허용)
  버스트: On-Demand (최후 수단)
```

### 2. 개발 환경 비용 절감

```
개발/스테이징 환경이 프로덕션 만큼 비싼 경우 많음!

자동 스케줄링:
  개발 서버: 업무 시간만 실행 (09:00~19:00 Mon~Fri)
  → 나머지 72시간 중 50시간 종료
  → 절감: 50/168 = 약 70%
  
  AWS Lambda:
  CRON: "0 9 * * 1-5"  → 개발 서버 시작
  CRON: "0 19 * * 1-5" → 개발 서버 중단
  
  비용 예시:
  EC2 c5.2xlarge ($0.34/h) × 168h = $57/월
  스케줄링 후: $0.34 × 50h = $17/월 (70% 절감!)

인스턴스 다운그레이드:
  프로덕션: c5.4xlarge (16 vCPU, 32GB) → $0.68/h
  스테이징: c5.xlarge (4 vCPU, 8GB) → $0.17/h (75% 절감)
  개발: t3.medium (2 vCPU, 4GB) → $0.04/h (94% 절감)

RDS 절감:
  개발 DB: db.t3.micro (2 vCPU, 1GB) — 기능 테스트에 충분
  개발 DB 스냅샷: 야간 종료 + 아침 복원 (Multi-AZ 불필요)
```

### 3. 스토리지 최적화

```
S3 비용 최적화:

S3 스토리지 티어:
  Standard:            $0.023/GB/월 (자주 접근)
  Infrequent Access:   $0.0125/GB/월 (월 1회 미만 접근)
  Glacier:             $0.004/GB/월 (아카이브, 복원 수 분~시간)
  Glacier Deep Archive:$0.00099/GB/월 (장기 보관, 복원 12시간)

S3 Lifecycle 정책 (자동 계층 이동):
  0~30일: Standard
  30~90일: Infrequent Access (자동 이동, 45% 절감)
  90~365일: Glacier (자동 이동, 83% 절감)
  365일+: 삭제 (설정에 따라)

예시 절감:
  로그 데이터 100TB:
  S3 Standard 유지: $2,300/월
  Lifecycle 적용:   $2,300 → $400/월 (83% 절감)

스냅샷 정리:
  오래된 EBS 스냅샷: 매주 자동 삭제 (최근 4주만 유지)
  미사용 S3 버킷: AWS Cost Explorer로 식별 후 삭제
  AMI 정리: 사용 안 하는 AMI + 연결된 스냅샷 제거
```

### 4. 데이터 전송 비용 절감

```
AWS 데이터 전송 비용:
  EC2 → 인터넷: $0.09/GB
  EC2 → S3 (같은 리전): 무료
  EC2 (us-east-1) → EC2 (ap-northeast-1): $0.02/GB

절감 방법:

1. 같은 리전에 서비스 배치:
   EC2와 RDS를 같은 AZ에 → AZ 간 전송 비용 ($0.01/GB) 없음

2. VPC Endpoint:
   EC2 → S3: 인터넷 게이트웨이 경유 → $0.09/GB
   EC2 → S3 (VPC Endpoint): VPC 내부 → 무료

3. CloudFront 활용:
   S3 → 사용자 직접: 전송 비용 발생
   S3 → CloudFront → 사용자: CloudFront 가격이 더 저렴
   CloudFront: $0.0085/GB vs S3 직접: $0.09/GB (90% 절감)

4. 압축:
   API 응답 gzip 압축 → 데이터 전송량 60~80% 감소
   → 전송 비용 + 지연 모두 감소
```

---

## 🔬 핵심 컴포넌트 — 비용 모니터링

### AWS Cost Explorer + 태깅

```
리소스 태깅 전략:
  모든 AWS 리소스에 태그 필수:
  
  환경: production, staging, development
  서비스: payment-service, user-service, api-gateway
  팀: backend-team, data-team
  프로젝트: project-x, migration-2024

태깅 강제화:
  AWS Organizations SCP:
  {
    "Effect": "Deny",
    "Action": ["ec2:RunInstances"],
    "Condition": {
      "Null": {"aws:RequestedRegion": "true"}
    }
  }
  → 특정 태그 없으면 리소스 생성 차단

팀별 비용 보고:
  AWS Cost Explorer 그룹화: Team 태그 기준
  월간 팀별 클라우드 비용 보고서 → 팀장에게 자동 발송
  → 팀이 자신의 비용에 책임감

AWS Budgets:
  월별 예산 설정 (예: $50,000/월)
  예산 80% 도달 → Slack 알림
  100% 도달 → PagerDuty 알림 + 자동 Spot 인스턴스 종료
```

### Right-sizing (인스턴스 최적화)

```
문제: 인스턴스 사양을 너무 크게 설정 (버퍼를 많이 잡음)
  c5.4xlarge (16 vCPU) 사용 중인데 평균 CPU 10%

AWS Compute Optimizer:
  실제 사용 패턴 분석
  권장 사항: "c5.4xlarge → c5.xlarge로 다운그레이드 가능"
  절감: $0.68/h → $0.17/h = $368/월 절감

Right-sizing 프로세스:
  1. CloudWatch 메트릭 2주 수집 (CPU, 메모리, 네트워크)
  2. P95 기준 사용률 계산
  3. P95 사용률 < 30% → 인스턴스 타입 다운
  4. 다운그레이드 후 모니터링 (1주)
  5. 문제 없으면 확정

주의:
  피크 처리 용량 여유 유지 (P95 기준, P99 고려)
  Auto Scaling과 조합하면 right-sizing 효과 극대화
  (평소 작은 인스턴스 + 피크 시 자동 증설)
```

---

## 🔄 FinOps 실천 사이클

```
월간 FinOps 사이클:

1주차: 비용 리뷰 (Cost Explorer 분석)
  - 전월 대비 증가 항목 파악
  - 팀별 비용 현황 공유

2주차: 낭비 제거 (Quick Win)
  - 미사용 리소스 삭제 (스냅샷, 로드 밸런서)
  - 개발 환경 스케줄링

3주차: 최적화 계획 수립
  - Right-sizing 대상 식별
  - Reserved Instance 구매 계획

4주차: 구현 및 검증
  - Right-sizing 적용
  - 절감 효과 측정

분기별:
  Reserved Instance 구매 검토
  아키텍처 레벨 비용 검토 (서비스 통합 가능성 등)
```

---

## ⚖️ 비용 vs 성능 트레이드오프

```
절감하면 안 되는 것:
  ❌ 프로덕션 DB 다운그레이드 (성능 저하 → 사용자 경험 악화)
  ❌ 모니터링 리소스 (장애 감지 능력 저하)
  ❌ 백업 스토리지 (데이터 유실 위험)
  ❌ 보안 서비스 (WAF, GuardDuty)

절감해야 하는 것:
  ✅ 개발/스테이징 환경 야간/주말 종료
  ✅ 오래된 스냅샷, 로그 정리
  ✅ 사용률 낮은 인스턴스 다운그레이드
  ✅ Spot Instance 활용 (배치, CI/CD)
  ✅ S3 Lifecycle 정책 (자동 계층 이동)

기준:
  "이 비용 절감이 신뢰성/성능에 영향을 주는가?"
  → 아니오: 절감 실행
  → 예: 비용 절감보다 SLO 유지 우선
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 구매 방식: On-Demand + Reserved(기반) + Spot(배치)
  2. 개발 환경: 업무 시간만 실행 (70% 절감)
  3. 스토리지: S3 Lifecycle (Glacier 자동 이동, 83% 절감)
  4. 모니터링: 태깅 + Cost Explorer + 팀별 예산 알림
  5. Right-sizing: 월간 검토, Compute Optimizer 활용
```

---

## 🤔 심화 질문

**Q1. 클라우드 비용 최적화를 시작할 때 가장 먼저 해야 할 것은?**
> 가시화부터 시작합니다. 비용을 모르면 최적화할 수 없습니다. AWS Cost Explorer에서 서비스별, 리전별 비용을 확인합니다. 모든 리소스에 태그를 달아 팀/서비스별 비용을 분리합니다. 그 후 가장 큰 비용 항목부터 집중합니다. 보통 EC2가 최대 비용 항목이므로, 개발 환경 스케줄링과 Right-sizing으로 즉시 20~30% 절감이 가능합니다.

**Q2. Reserved Instance 구매를 어떻게 결정하는가?**
> 최소 6개월 이상 사용이 확실한 리소스에만 구매합니다. 현재 On-Demand 지출에서 1년 이상 지속될 베이스라인을 파악합니다(가장 낮은 사용 구간의 CPU 사용률). 이 베이스라인만큼 Reserved를 구매하고 나머지는 On-Demand 유지합니다. 1년 약정 Convertible Reserved가 유연성과 절감의 균형이 좋습니다.

**Q3. "FinOps 팀이 없는 스타트업에서 누가 비용 최적화를 담당하는가?"**
> 초기에는 플랫폼 엔지니어 또는 DevOps 엔지니어가 담당합니다. 팀 50명 미만에서는 월 1회 Cost Explorer 리뷰 + 기본 자동화(개발 환경 스케줄링, S3 Lifecycle)만으로도 큰 절감이 가능합니다. 팀 100명 이상에서는 전담 FinOps 역할이 효과적이며, 클라우드 지출이 월 수억 원 수준이면 전담 팀이 즉시 비용 회수됩니다.

---

<div align="center">

[⬅️ 이전: 성장 단계별 아키텍처](./03-growth-path.md) | [README로 돌아가기](../README.md)

</div>
