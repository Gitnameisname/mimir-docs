# Task 13-2. 로깅 체계 구축

## 1. 작업 목적

운영 환경에서 문제 진단과 감사에 활용할 수 있는 구조화 로그 체계를 구축한다.

이 작업의 목표는 다음과 같다.

- JSON 구조화 로그 표준 정의
- 로그 레벨 정책 수립
- 민감 정보 마스킹 적용
- 요청 추적 (Request ID / Correlation ID)
- 로그 수집 파이프라인 구성

---

## 2. 작업 범위

### 포함 범위

- 로그 포맷 표준 정의 및 구현
- 로그 레벨 정책 수립
- 민감 정보 마스킹 필터
- Request ID 미들웨어
- 로그 수집 설정 (stdout → 수집 에이전트)

### 제외 범위

- 로그 저장/분석 인프라 구성 (운영 환경별 진행)
- 감사 로그 자체 (Phase 7에서 완성됨 — 보존 정책은 Task 13-6)

---

## 3. 주요 구현 대상

### 3-1. JSON 구조화 로그 표준

모든 로그를 JSON 형식으로 출력:

```json
{
  "timestamp": "2026-04-08T12:34:56.789Z",
  "level": "INFO",
  "service": "mimir-api",
  "version": "1.0.0",
  "request_id": "req-uuid-1234",
  "user_id": "user-uuid-5678",
  "method": "POST",
  "path": "/api/rag/query",
  "status_code": 200,
  "duration_ms": 234,
  "message": "RAG query completed",
  "extra": {
    "conversation_id": "conv-uuid-9012",
    "chunk_count": 5
  }
}
```

---

### 3-2. 로그 레벨 정책

| 레벨 | 사용 기준 |
|------|---------|
| ERROR | 예외, 시스템 오류, 데이터 손상 위험 |
| WARNING | 비정상 상황이지만 서비스 계속 가능 (재시도 성공 등) |
| INFO | 주요 비즈니스 이벤트 (로그인, 문서 생성, 배포 등) |
| DEBUG | 개발/진단 목적 (운영 환경에서 비활성화) |

운영 환경 기본 레벨: INFO
디버그 모드: 환경 변수 LOG_LEVEL=DEBUG로 조정

---

### 3-3. 민감 정보 마스킹 필터

```python
MASKED_FIELDS = ["password", "api_key", "token", "secret", "authorization"]
EMAIL_PATTERN = re.compile(r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}')

class SensitiveDataFilter(logging.Filter):
    def filter(self, record):
        msg = str(record.getMessage())
        # 이메일 마스킹
        msg = EMAIL_PATTERN.sub(lambda m: m.group()[:3] + "***@***", msg)
        # 키워드 마스킹
        for field in MASKED_FIELDS:
            msg = re.sub(
                rf'"{field}"\s*:\s*"[^"]*"',
                f'"{field}": "***"',
                msg, flags=re.IGNORECASE
            )
        record.msg = msg
        return True
```

---

### 3-4. Request ID 미들웨어

모든 요청에 고유 ID를 부여하고 응답 헤더에 포함:

```python
@app.middleware("http")
async def request_id_middleware(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID") or str(uuid4())
    # 컨텍스트에 저장 (로그에서 접근 가능)
    ctx_request_id.set(request_id)
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response
```

---

### 3-5. 핵심 로깅 이벤트 목록

| 이벤트 | 레벨 | 포함 정보 |
|-------|------|---------|
| 로그인 성공/실패 | INFO / WARNING | user_id, IP, 실패 횟수 |
| 문서 생성/수정/삭제 | INFO | document_id, user_id |
| 권한 거부 | WARNING | resource, user_id, required_role |
| RAG 쿼리 | INFO | duration_ms, chunk_count, conversation_id |
| 임베딩 API 호출 | INFO | token_count, model |
| LLM API 호출 | INFO | token_count, model, cost_usd |
| 백그라운드 Job 완료/실패 | INFO / ERROR | job_type, duration_ms |
| API 오류 (5xx) | ERROR | path, error_type, stack_trace |

---

### 3-6. 로그 출력 설정

```python
# logging_config.py
import structlog

structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.stdlib.add_log_level,
        SensitiveDataFilter(),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(LOG_LEVEL),
    logger_factory=structlog.PrintLoggerFactory(),
)
```

stdout으로 출력하여 컨테이너 환경에서 수집 에이전트가 가져가도록 함.

---

## 4. 산출물

1. JSON 구조화 로그 포맷 정의 및 구현
2. 민감 정보 마스킹 필터
3. Request ID 미들웨어
4. 로그 레벨 정책 문서
5. 핵심 로깅 이벤트 구현

---

## 5. 완료 기준

- 모든 API 요청/응답이 JSON 구조화 로그로 기록된다
- 로그에 API 키, 비밀번호가 마스킹 처리된다
- 요청별 Request ID가 로그와 응답 헤더에 포함된다
- 핵심 비즈니스 이벤트가 INFO 레벨로 기록된다

---

## 6. Codex 작업 지침

- structlog 또는 loguru 라이브러리를 사용하여 구조화 로그를 구현한다
- 로그에 stack_trace를 포함할 때 개인정보가 포함되지 않도록 주의한다
- DEBUG 로그가 운영 환경에서 기본으로 비활성화되도록 환경 변수로 제어한다
- Request ID는 외부에서 주입된 경우(Load Balancer 포워딩)도 허용하되 신뢰하지 않는 ID 값에 대한 길이 제한을 둔다
