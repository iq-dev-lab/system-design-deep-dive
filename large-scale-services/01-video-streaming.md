# 동영상 스트리밍 서비스 (YouTube / Netflix)

---

## 🎯 설계 목표와 요구사항

### 기능 요구사항
- 동영상 업로드 (최대 50GB)
- 동영상 스트리밍 (다양한 해상도: 360p, 720p, 1080p, 4K)
- 검색 및 추천
- 좋아요, 댓글, 구독

### 비기능 요구사항
- DAU 5,000만, 동시 스트리밍 1,000만 명
- 동영상 업로드 후 수 분 내 시청 가능
- 스트리밍 버퍼링 최소화 (네트워크 상황에 따른 적응형 화질)
- 99.99% 가용성

---

## 📊 규모 산정

```
동영상 업로드:
  하루 업로드: 50만 건
  평균 크기: 300MB (원본)
  일일 저장: 150TB/일
  1년 저장: 54PB

스트리밍:
  DAU 5,000만 × 평균 60분 시청
  1080p 스트리밍 대역폭: 5Mbps
  동시 시청 1,000만 명 × 5Mbps = 50Tbps 대역폭
  → 글로벌 CDN 필수

트랜스코딩:
  원본 1개 → 해상도별 6개 파일 생성
  50만 건 × 6 = 300만 파일/일
```

---

## 🏗️ 개략적 설계

```
[업로드 흐름]

사용자
  │ (1) 업로드 URL 요청
  ▼
API 서버
  │ (2) Presigned URL 반환 (S3 직접 업로드)
  ▼
사용자 → S3 Raw Storage (원본 저장)
                │
                │ (3) 업로드 완료 이벤트
                ▼
           ┌─────────────┐
           │  Kafka      │  upload-complete-topic
           └──────┬──────┘
                  │
                  ▼
         ┌──────────────────┐
         │  Transcoding     │  비동기 처리
         │  Worker Pool     │
         │  (FFmpeg 기반)   │
         └────────┬─────────┘
                  │ 각 해상도별 파일 생성
                  ▼
           S3 Processed Storage
                  │
                  │ CDN 엣지 배포
                  ▼
              CloudFront / Akamai


[스트리밍 흐름]

사용자
  │ (1) /watch?v=abc123
  ▼
API 서버
  │ (2) 동영상 메타데이터 + CDN URL 반환
  ▼
사용자 플레이어 → CDN Edge Server
                      │ (Cache Miss)
                      ▼
                  S3 Processed Storage
```

---

## 🔬 핵심 컴포넌트

### 트랜스코딩 파이프라인

```
원본 동영상 (50GB, 4K H.264)
      │
      ▼
  ┌─────────────────────────────────┐
  │  Transcoding Workers           │
  │                                 │
  │  병렬 처리 (각 해상도 독립):    │
  │  ├── Worker 1: 4K (2160p)      │
  │  ├── Worker 2: 1080p           │
  │  ├── Worker 3: 720p            │
  │  ├── Worker 4: 480p            │
  │  ├── Worker 5: 360p            │
  │  └── Worker 6: 144p (저용량)   │
  └─────────────────────────────────┘
      │
      ▼
  ┌──────────────────────────────────┐
  │  HLS (HTTP Live Streaming) 세그먼트화 │
  │                                      │
  │  video_1080p_001.ts  (10초 조각)    │
  │  video_1080p_002.ts                 │
  │  ...                                 │
  │  video_1080p.m3u8  (재생목록)       │
  └──────────────────────────────────────┘
      │
      ▼
  S3 → CDN 배포

HLS 장점:
  - 10초 세그먼트 단위로 스트리밍 → 버퍼링 최소화
  - 화질 적응 (ABR): 네트워크 속도에 따라 자동 화질 전환
  - HTTP 기반 → CDN 캐싱 가능
```

### 적응형 비트레이트 스트리밍 (ABR)

```
재생목록 파일 (master.m3u8):
  #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
  360p/index.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720
  720p/index.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=8000000,RESOLUTION=1920x1080
  1080p/index.m3u8

플레이어 동작:
  1. master.m3u8 다운로드
  2. 현재 네트워크 속도 측정
  3. 적합한 화질 선택 후 해당 세그먼트 다운로드
  4. 네트워크 변화 → 자동 화질 전환 (끊김 없이)
```

### 동영상 메타데이터 DB 설계

```sql
-- 동영상 테이블 (MySQL/PostgreSQL)
CREATE TABLE videos (
    video_id    VARCHAR(11) PRIMARY KEY,  -- YouTube 스타일 11자리
    user_id     BIGINT NOT NULL,
    title       VARCHAR(200),
    description TEXT,
    duration    INT,           -- 초 단위
    status      ENUM('UPLOADING', 'PROCESSING', 'READY', 'FAILED'),
    view_count  BIGINT DEFAULT 0,
    like_count  BIGINT DEFAULT 0,
    created_at  TIMESTAMP DEFAULT NOW(),
    thumbnail_url VARCHAR(500)
);

-- 동영상 파일 경로 (각 해상도별)
CREATE TABLE video_files (
    video_id    VARCHAR(11),
    resolution  ENUM('144p','360p','480p','720p','1080p','2160p'),
    s3_path     VARCHAR(500),
    cdn_url     VARCHAR(500),
    file_size   BIGINT,
    PRIMARY KEY (video_id, resolution)
);
```

---

## 🔄 데이터 흐름 — 조회수 집계

```
문제: 1,000만 명이 동시 시청 → 조회수 실시간 업데이트 시 DB 병목

해결: 비동기 집계

시청 이벤트 → Kafka (view-event-topic)
                  │
                  ▼
          Redis 카운터 (5분마다)
          INCR "view:{video_id}"
                  │
                  ▼
          배치 DB 업데이트 (5분마다)
          UPDATE videos SET view_count = view_count + {redis_count}
          WHERE video_id = ?

최종 일관성 허용:
  실시간 조회수가 몇 분 지연되어도 UX 영향 없음
  YouTube도 대략적인 조회수 표시 (정확한 실시간 아님)
```

---

## ⚖️ 트레이드오프

| 결정 | 선택 | 이유 |
|------|------|------|
| 스트리밍 프로토콜 | HLS | HTTP 기반 CDN 캐싱, 범용 지원 |
| 업로드 방식 | S3 Presigned URL | 대용량 파일을 API 서버 우회, 직접 S3 전송 |
| 트랜스코딩 | 비동기 Worker (Kafka) | 업로드 즉시 응답, 처리는 백그라운드 |
| 조회수 | Redis → 배치 DB | DB 직접 쓰기 병목 방지 |

---

## 🚀 확장 전략

```
트랜스코딩 확장:
  AWS MediaConvert / Zencoder 같은 관리형 서비스 활용
  또는 EC2 Spot Instance로 비용 최적화 (중단 허용)

CDN 확장:
  지역별 엣지 서버 (서울, 도쿄, 싱가포르, 미국 동부/서부)
  Popular 콘텐츠는 모든 엣지에 프리워밍

저장 최적화:
  오래된 동영상 → S3 Glacier로 자동 이동 (Lifecycle)
  인기 동영상 → S3 Standard (CDN 직결)
  저화질(144p) 항상 엣지 캐시 유지
```

---

## 📌 핵심 결정 요약

```
핵심 설계 포인트:
  1. 업로드: S3 Presigned URL (대용량 파일 직접 전송)
  2. 트랜스코딩: 비동기 Worker Pool (Kafka 기반, 해상도별 병렬)
  3. 스트리밍: HLS + ABR (10초 세그먼트, 자동 화질 적응)
  4. CDN: 글로벌 엣지 서버 (50Tbps 대역폭 처리)
  5. 조회수: Redis 버퍼링 → 배치 DB 업데이트
```

---

## 🤔 심화 질문

**Q1. 동영상 업로드 후 "처리 중" 상태를 어떻게 사용자에게 알리는가?**
> 두 가지 방법이 있습니다. 폴링 방식: 클라이언트가 5초마다 `/videos/{id}/status` API를 호출. WebSocket/SSE 방식: 트랜스코딩 완료 시 서버가 사용자 브라우저에 Push 이벤트 전송. YouTube는 폴링 방식을 사용하며, 트랜스코딩 완료 시 이메일 알림도 발송합니다.

**Q2. 저작권 침해 동영상을 어떻게 감지하는가?**
> 업로드 시 동영상 핑거프린팅(Content ID)을 생성합니다. 오디오/비디오 웨이브폼에서 특징을 추출해 해시를 만들고, 저작권자 데이터베이스와 비교합니다. YouTube의 Content ID 시스템이 이 방식입니다. 트랜스코딩 파이프라인에 핑거프린팅 단계를 추가해 업로드 시점에 감지합니다.

**Q3. 라이브 스트리밍과 VOD 스트리밍의 차이는?**
> VOD는 미리 트랜스코딩된 파일을 CDN에서 서빙합니다. 라이브 스트리밍은 스트리머의 카메라 → RTMP로 서버 전송 → 실시간 트랜스코딩 → HLS 세그먼트 생성 → CDN 배포로 이어집니다. 라이브는 세그먼트가 실시간으로 생성되므로 지연(Latency)이 10~30초 발생하며, 이를 줄이기 위해 LHLS(Low Latency HLS)나 WebRTC 방식을 사용합니다.

---

<div align="center">

[⬅️ 이전: 검색 자동완성](../basic-services/07-typeahead.md) | [README로 돌아가기](../README.md) | [다음: 소셜 피드 (Fan-out) ➡️](./02-social-feed.md)

</div>
