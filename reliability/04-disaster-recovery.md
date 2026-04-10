# 재해 복구 (RPO / RTO / DR 전략)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 데이터센터/리전 전체 장애 시 서비스 복구
- 데이터 유실 최소화
- 복구 절차 자동화
- 정기적 DR 훈련

### 비기능 요구사항
- RPO (Recovery Point Objective): 1시간 이하 (최대 데이터 유실 허용량)
- RTO (Recovery Time Objective): 4시간 이하 (복구까지 허용 시간)
- 연간 DR 훈련 2회 이상
- 복구 절차 문서화 및 자동화

---

## 📊 RPO / RTO 정의

```
RPO (Recovery Point Objective): 데이터 관점
  "장애 발생 시 최대 얼마만큼의 데이터 유실을 허용하는가?"
  
  RPO = 0:      무손실 (동기 복제 필요, 비용 매우 높음)
  RPO = 1분:    마지막 1분 데이터 유실 허용
  RPO = 1시간:  마지막 1시간 데이터 유실 허용 (비용 낮음)
  
  결정 요소: 데이터의 비즈니스 가치
    금융 거래: RPO = 0 (1건도 유실 불가)
    SNS 게시물: RPO = 1시간 (적당)
    로그 데이터: RPO = 24시간 (허용)

RTO (Recovery Time Objective): 시간 관점
  "장애 발생 후 서비스가 복구되기까지 허용 시간은?"
  
  RTO = 0:      즉시 전환 (Active-Active, 매우 비쌈)
  RTO = 15분:   빠른 자동 Failover
  RTO = 4시간:  수동 복구 허용
  RTO = 24시간: 재해 복구 계획 (비상 상황)
  
  RTO가 짧을수록 → 복잡한 자동화 필요 → 비용 증가

비용 vs RPO/RTO:
  RPO↓ + RTO↓ = 비용↑↑
  
  (낮은) RPO ────────────────────── (높은) RPO
              ↑비용↑                 ↑비용↓
  Zero-RPO  1분RPO  1시간RPO  24시간RPO
```

---

## 🏗️ DR 전략 4단계

### 전략 1: Backup & Restore

```
RPO: 24시간, RTO: 24시간+

가장 단순하고 저렴한 방법:

매일 자정:
  DB 스냅샷 → S3 저장 (다른 리전에 복제)
  
장애 발생:
  1. 새 인프라 프로비저닝 (Terraform)
  2. S3에서 최신 스냅샷 다운로드
  3. DB 복원
  4. 서비스 재시작
  
  소요 시간: 수 시간 ~ 하루

적합: 개발/스테이징 환경, 중요도 낮은 서비스
비용: 스냅샷 저장 비용만 (매우 저렴)
```

### 전략 2: Pilot Light

```
RPO: 1시간, RTO: 1~4시간

최소한의 인프라만 DR 리전에 항상 실행:

평상시:
  DR 리전 (도쿄):
    ├── DB (Read Replica, 복제 중)
    └── 핵심 서비스 (최소 인스턴스로 실행)
  
  서울 → 도쿄 DB로 비동기 복제 (1시간 지연 가능)

장애 발생:
  1. DR 리전 DB를 Primary로 승격
  2. Auto Scaling 그룹 인스턴스 수 증가 (Scale Out)
  3. DNS를 도쿄로 전환
  
  소요 시간: 1~4시간

적합: 일반 서비스 (SaaS, 이커머스)
비용: DR 리전 최소 인프라 비용 (약 20~30%)
```

### 전략 3: Warm Standby

```
RPO: 분 단위, RTO: 15~60분

DR 리전에 실제 트래픽의 10~20%를 처리하는 축소 버전 실행:

평상시:
  서울 (Primary, 80% 트래픽):   API 100개 인스턴스
  도쿄 (Standby, 20% 트래픽):  API 20개 인스턴스 (실제 서비스 중)
  
  → DB 동기 복제 (또는 짧은 비동기)
  → 도쿄가 이미 실제 트래픽 처리 중 → 검증됨

장애 발생:
  1. DNS 가중치 변경 (서울 0%, 도쿄 100%)
  2. 도쿄 Auto Scaling으로 인스턴스 증가
  
  소요 시간: 15~60분

적합: 중요 서비스 (금융, 의료)
비용: DR 리전 20~30% 수준 인프라 유지
```

### 전략 4: Active-Active (Multi-Site)

```
RPO: 0 (또는 수 초), RTO: 즉시

모든 리전이 100% 트래픽 처리:

서울 (50% 트래픽) ←→ 도쿄 (50% 트래픽)
  양방향 DB 복제 (동기 또는 준동기)
  
  GeoDNS → 사용자 위치 기반 라우팅
  
장애 발생:
  서울 장애 → GeoDNS가 도쿄로 자동 전환 (DNS TTL 30초)
  도쿄는 이미 100% 트래픽 처리 준비됨
  
  소요 시간: 수십 초 (DNS TTL 만료)

적합: 글로벌 서비스, 금융 (RPO=0 필수)
비용: 2배 (모든 리전에 Full 인프라)
단점: 데이터 충돌 해결 필요 (양방향 복제)
```

---

## 🔬 핵심 컴포넌트 — 자동화된 Failover

### DB Failover

```
MySQL/PostgreSQL 자동 Failover:

AWS RDS Multi-AZ:
  Primary (AZ-A) ─동기 복제─▶ Standby (AZ-B)
  
  Primary 장애:
  1. RDS가 자동 감지 (약 60~120초)
  2. Standby를 Primary로 자동 승격
  3. DNS 엔드포인트 자동 변경
  
  RTO: 1~2분 (자동)
  RPO: ~0 (동기 복제)

Aurora Global Database:
  Primary 리전 (서울) ─비동기 복제─▶ Secondary 리전 (도쿄)
  복제 지연: 보통 < 1초
  
  장애 시 Secondary를 Primary로 수동 또는 자동 승격
  
  RTO: < 1분
  RPO: < 1초

Redis Sentinel:
  Master ─복제─▶ Replica 1
         ─복제─▶ Replica 2
  
  Sentinel 3개 (quorum 2):
    Master 장애 → Sentinel 과반수 합의 → Replica 1을 Master로 승격
    자동으로 클라이언트에 새 Master 주소 알림
```

### Terraform으로 인프라 복구 자동화

```hcl
# DR 리전 인프라 (Terraform)
provider "aws" {
  alias  = "tokyo"
  region = "ap-northeast-1"
}

# DR 리전 VPC
resource "aws_vpc" "dr_vpc" {
  provider   = aws.tokyo
  cidr_block = "10.2.0.0/16"
}

# DR 리전 DB (Read Replica)
resource "aws_db_instance" "dr_replica" {
  provider             = aws.tokyo
  replicate_source_db  = aws_db_instance.primary.arn
  instance_class       = "db.r6g.xlarge"
  skip_final_snapshot  = true
  
  # 장애 시 독립 Primary로 승격
  # aws rds promote-read-replica --db-instance-identifier dr-replica
}

# DR 트래픽 전환 (Route 53)
resource "aws_route53_record" "api" {
  zone_id = var.zone_id
  name    = "api.service.com"
  type    = "A"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  alias {
    name    = aws_lb.seoul_alb.dns_name
    zone_id = aws_lb.seoul_alb.zone_id
  }
  
  health_check_id = aws_route53_health_check.seoul.id
  set_identifier  = "primary"
}
```

---

## 🔄 DR 훈련 절차

```
연간 2회 DR 훈련 (계획):

사전 준비:
  1. 훈련 목표 설정 (RTO 4시간 검증)
  2. 이해관계자 통보 (일부 서비스 성능 저하 가능)
  3. 롤백 계획 준비
  4. 모니터링 강화

훈련 실행:
  단계 1: 서울 서버 트래픽 차단 시뮬레이션
    (실제 종료 대신 Security Group으로 차단)
  단계 2: DR 절차 실행 타이머 시작
  단계 3: 도쿄 DB Primary 승격
  단계 4: 도쿄 인스턴스 Scale Out
  단계 5: DNS 전환
  단계 6: 검증 (API 테스트, DB 쿼리)
  
  소요 시간 측정 → RTO 달성 여부 확인

훈련 후:
  결과 보고서 작성
  개선 사항 도출 → 다음 훈련 전 수정
  Runbook 업데이트
```

---

## ⚖️ 전략별 비용/복구 비교

```
┌──────────────────┬──────────┬──────────┬──────────────────────┐
│ 전략             │ RPO      │ RTO      │ 비용 (Primary 대비)  │
├──────────────────┼──────────┼──────────┼──────────────────────┤
│ Backup & Restore │ 24시간   │ 24시간   │ 5~10% (저장만)       │
│ Pilot Light      │ 1시간    │ 1~4시간  │ 15~20%              │
│ Warm Standby     │ 분 단위  │ 15~60분  │ 30~50%              │
│ Active-Active    │ 0~초     │ 즉시     │ 100% (2배 인프라)   │
└──────────────────┴──────────┴──────────┴──────────────────────┘
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. RPO/RTO 먼저 정의: 비즈니스 요구에서 출발 (복구 목표 결정)
  2. 전략 선택: RPO/RTO ↔ 비용 트레이드오프
  3. 자동화: 수동 복구 → 인적 오류, Terraform + 자동 Failover
  4. 정기 훈련: 계획만 있고 훈련 안 하면 실제 장애 시 실패
  5. 문서화: Runbook (누가, 무엇을, 어떤 순서로)
```

---

## 🤔 심화 질문

**Q1. RPO와 RTO를 동시에 낮추는 것이 왜 어려운가?**
> RPO를 낮추려면 더 자주 백업하거나 동기 복제를 해야 합니다(비용/지연 증가). RTO를 낮추려면 DR 인프라를 항상 준비 상태로 유지해야 합니다(비용 증가). 둘 다 낮추면(Active-Active) 2배 인프라 비용, 양방향 복제 구현 복잡성, 데이터 충돌 해결이 필요합니다. 금융 거래처럼 데이터 손실이 허용되지 않는 경우에만 이 비용을 감수합니다.

**Q2. DR 훈련에서 실제로 프로덕션 DB를 전환하면 위험하지 않는가?**
> 맞습니다. 그래서 단계적으로 접근합니다. 첫 훈련: 개발 환경에서 전체 절차 연습. 두 번째 훈련: 스테이징 환경에서 실제 DB 전환. 세 번째: 프로덕션에서 소규모 트래픽(1%)만 DR 리전으로 전환해 검증. 이후: 실제 DR 훈련(사전 통보 + 짧은 유지보수 창). Game Day(Netflix) 방식으로 팀 전체가 비상 상황을 시뮬레이션합니다.

**Q3. "RTO 4시간"을 달성하는 Runbook을 어떻게 작성하는가?**
> 단계별 체크리스트 형식으로 작성합니다. 각 단계에 예상 소요 시간, 담당자 역할, 확인 방법을 명시합니다. 예: "1. DR DB Read Replica 승격 (10분, DBA 담당): `aws rds promote-read-replica --db-instance-identifier dr-db` 실행 후 `aws rds describe-db-instances`로 상태 확인". 실제 장애 시 Runbook을 보고 누구든 실행할 수 있어야 합니다. 분기별 Runbook 검토로 최신 상태 유지.

---

<div align="center">

[⬅️ 이전: 모니터링과 알림](./03-monitoring.md) | [README로 돌아가기](../README.md) | [다음: 포스트모텀 ➡️](./05-postmortem.md)

</div>
