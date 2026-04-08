# Task 13-3. 모니터링 및 알림 체계

## 1. 작업 목적

운영 환경의 상태를 실시간으로 파악하고 이상 징후 발생 시 즉시 알림을 받을 수 있는 모니터링 체계를 구축한다.

이 작업의 목표는 다음과 같다.

- 핵심 메트릭 정의 및 수집 구현
- Grafana 대시보드 구성
- 알림 규칙 설정 (SLO 위반 시 즉시 알림)
- Health Check 엔드포인트 구현

---

## 2. 작업 범위

### 포함 범위

- 애플리케이션 메트릭 수집 (Prometheus 형식)
- 핵심 대시보드 구성
- 알림 규칙 및 채널 설정
- Health Check API 구현
- SLO 기준 알림 임계값 설정

### 제외 범위

- 인프라 메트릭 (서버/DB 모니터링 — 운영 환경 별도)
- APM 상세 구현 (OpenTelemetry — 선택 사항)

---

## 3. 주요 구현 대상

### 3-1. 수집할 핵심 메트릭

#### API 메트릭

```
http_requests_total{method, path, status_code}   (요청 수)
http_request_duration_seconds{method, path}      (응답 시간 히스토그램)
http_errors_total{path, error_type}              (오류 수)
```

#### RAG 파이프라인 메트릭

```
rag_query_total{mode, status}                    (RAG 질의 수)
rag_query_duration_seconds{stage}                (단계별 소요 시간)
  stage: retriever | reranker | llm | total
rag_first_token_seconds                          (TTFT)
```

#### 비용 메트릭

```
llm_tokens_used_total{model, type}               (LLM 토큰 사용량)
  type: prompt | completion
embedding_tokens_used_total{model}               (임베딩 토큰 사용량)
```

#### 백그라운드 Job 메트릭

```
background_job_total{job_type, status}           (Job 실행 수)
background_job_duration_seconds{job_type}        (Job 실행 시간)
background_job_queue_size{job_type}              (대기 중 Job 수)
```

---

### 3-2. Prometheus 메트릭 수집 구현

```python
from prometheus_client import Counter, Histogram, Gauge

http_requests = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "path", "status_code"]
)

http_duration = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration",
    ["method", "path"],
    buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

# /metrics 엔드포인트 노출
from prometheus_client import make_asgi_app
metrics_app = make_asgi_app()
app.mount("/metrics", metrics_app)
```

---

### 3-3. Health Check API

```
GET /health

Response 200 (정상):
{
  "status": "healthy",
  "version": "1.0.0",
  "checks": {
    "database": "healthy",
    "vector_db": "healthy",
    "background_jobs": "healthy"
  },
  "timestamp": "2026-04-08T12:34:56Z"
}

Response 503 (비정상):
{
  "status": "unhealthy",
  "checks": {
    "database": "unhealthy",
    "vector_db": "healthy",
    "background_jobs": "degraded"
  }
}
```

---

### 3-4. 알림 규칙 (Alerting Rules)

| 알림명 | 조건 | 심각도 | 대응 |
|-------|------|-------|------|
| HighErrorRate | 5xx 오류율 > 1% (5분) | Critical | 즉시 대응 |
| HighLatency | API P95 > 1초 (5분) | Warning | 확인 필요 |
| RAGLatency | TTFT P95 > 5초 (5분) | Warning | 확인 필요 |
| JobQueueBacklog | 대기 Job > 100건 (10분) | Warning | 확인 필요 |
| JobFailureRate | Job 실패율 > 10% (10분) | Critical | 즉시 대응 |
| HighLLMCost | 시간당 비용 > $10 | Warning | 비용 검토 |
| DatabaseDown | DB 연결 실패 | Critical | 즉시 대응 |

---

### 3-5. Grafana 대시보드 구성

주요 대시보드:

```
1. 서비스 개요 대시보드
   - 요청 수 / 오류율 / 응답 시간 (실시간)
   - 활성 사용자 수
   - SLO 달성 현황

2. RAG 파이프라인 대시보드
   - 질의 수 / TTFT / 단계별 latency
   - LLM/임베딩 비용 추이

3. 백그라운드 Job 대시보드
   - Job 대기열 / 완료 / 실패
   - 벡터화 현황

4. 비용 대시보드
   - LLM + 임베딩 일별/월별 비용
   - 모델별 비용 분리
```

---

## 4. 산출물

1. Prometheus 메트릭 수집 구현 코드
2. /health 엔드포인트 구현
3. Grafana 대시보드 설정 파일 (JSON 익스포트)
4. 알림 규칙 설정 파일

---

## 5. 완료 기준

- /metrics 엔드포인트가 Prometheus 형식으로 메트릭을 반환한다
- /health 엔드포인트가 DB, 벡터 DB, Job 상태를 반환한다
- 핵심 알림 규칙이 설정되어 있고 테스트 알림을 수신한다
- Grafana 서비스 개요 대시보드가 실시간 데이터를 표시한다

---

## 6. Codex 작업 지침

- /metrics 엔드포인트는 내부 네트워크에서만 접근 가능하도록 설정한다 (외부 노출 금지)
- /health는 인증 없이 접근 가능하게 하되 민감 정보(버전 상세, 환경 변수)를 노출하지 않는다
- 메트릭 레이블(label)은 카디널리티에 주의하여 user_id 등 고유값을 레이블로 쓰지 않는다
- 알림은 PagerDuty 또는 Slack 웹훅으로 전송하며 채널 설정은 환경 변수로 관리한다
