# ValueError → HTTP 503 매핑 의심 경로 조사 보고서

- 작성일: 2026-04-21
- 원 문제 제기: `docs/개발문서/S2_5/UI_Admin_GoldenSets_런타임회귀리뷰.md` §"본 리뷰에서 추가로 관찰된 파생 이슈" 2번
- 범위: `backend/app/main.py`, `backend/app/api/errors/handlers.py`, `backend/app/api/context/middleware.py`, `backend/app/api/rate_limit.py`, `backend/app/observability/metrics.py`, `backend/app/api/security/*`, `backend/app/db/connection.py`, `frontend/next.config.ts`
- 연계: #30 (item_count 필드 누락 ValueError — 기수정), #31 (목록 에러 배너 분기 — 기수정)

---

## 1. 요약 결론

- 런타임 회귀 리뷰(#30 재현 단계)에서 "ValueError → 503" 으로 관측되었다고 기록한 부분은 **코드베이스 상으로는 재현될 수 없는 경로**다. 현 구현의 전역 예외 처리 체인은 **모든** 미분류 `Exception`(ValueError 포함) 을 **500** `internal_server_error` 로 매핑한다. 503 으로 바뀌는 중간 변환 지점은 없다.
- 503 관측 기록은 **가장 가능성 높은 설명은 오관측**(혹은 이전 코드 스냅샷의 잔상) 이며, 코드상 의도대로 동작하게 하기 위한 **계약 테스트를 추가**하여 향후 회귀(임의 middleware 가 500→503 을 덧쓰는 경우 등) 를 즉시 감지하도록 한다.
- 503 은 앞으로도 **`ApiServiceUnavailableError`** 를 명시적으로 raise 하는 경로에서만 나와야 한다. 이 계약이 깨지는지 여부는 신규 단위테스트로 상시 검증된다.

---

## 2. 전역 예외 처리 체인 재구성

### 2.1 등록된 핸들러 (5종)

| # | 대상 클래스 | 핸들러 | 매핑 결과 |
| --- | --- | --- | --- |
| 1 | `ApiError` | `api_error_handler` | `exc.http_status` 그대로 |
| 2 | `RequestValidationError` | `request_validation_error_handler` | 400 |
| 3 | `StarletteHTTPException` | `http_exception_handler` | `exc.status_code` 그대로 |
| 4 | `Exception` (catch-all) | `unhandled_exception_handler` | **500** `internal_server_error` |
| 5 | `RateLimitExceeded` (slowapi) | `_rate_limit_exceeded_handler` | 429 |

Starlette 의 `build_middleware_stack` 은 `Exception`/`500` 핸들러를 `ServerErrorMiddleware` 로 분리하고 나머지는 `ExceptionMiddleware` 로 묶는다. 따라서 `unhandled_exception_handler` 는 **가장 바깥쪽 안전망** 역할을 한다.

### 2.2 실제 미들웨어 체인 (outer → inner)

`main.py:create_app` 에서 `add_middleware` 순서(뒤에 호출할수록 outer):

```
ServerErrorMiddleware(handler=unhandled_exception_handler)   ← 500 catch-all
  RequestContextMiddleware        (BaseHTTPMiddleware)
    RequestSizeLimitMiddleware    (BaseHTTPMiddleware)
      SecurityHeadersMiddleware   (BaseHTTPMiddleware)
        PrometheusMiddleware      (BaseHTTPMiddleware)
          CORSMiddleware          (순수 ASGI, preflight 만 관여)
            ExceptionMiddleware(ApiError / RequestValidationError / StarletteHTTPException / RateLimitExceeded)
              Router → /api/v1/golden-sets ... (sync endpoint)
```

### 2.3 sync endpoint 가 raise 한 `ValueError` 의 경로

1. Router 가 `ValueError` 를 던진다(FastAPI 는 sync endpoint 를 `anyio.to_thread` 로 실행한다 — 예외는 그대로 외부 이벤트루프로 전파된다).
2. `ExceptionMiddleware` 는 `ApiError/RequestValidationError/StarletteHTTPException/RateLimitExceeded` 중 어느 것에도 매칭되지 않으므로 **그대로 re-raise**.
3. `CORSMiddleware` 는 응답 객체에만 헤더를 덧붙이는 구조라 예외에 관여하지 않음.
4. `PrometheusMiddleware.dispatch`:
   ```python
   try:
       response = await call_next(request); status = response.status_code
   except Exception:
       status = 500; raise
   ```
   → **로컬 status 변수만 500 으로 기록하고 예외는 re-raise**.
5. `SecurityHeadersMiddleware`, `RequestSizeLimitMiddleware`: 응답 헤더 덧붙이기 / 요청 크기 검사만 수행하며 예외 경로에서는 통과(별도 catch 없음).
6. `RequestContextMiddleware`:
   ```python
   try:
       response = await call_next(request)
   except Exception:
       log_request_completion(..., status_code=500, result="failure"); raise
   ```
   → **로그만 남기고 re-raise**.
7. `ServerErrorMiddleware` 가 `Exception` 매칭으로 `unhandled_exception_handler` 호출:
   ```python
   return _build_response(status_code=500, error_code="internal_server_error", ...)
   ```

**결과 wire status = 500.** 어디에도 503 을 만들 경로가 없다.

### 2.4 503 을 만들 수 있는 유일한 경로

리포지토리 전역 grep:

```text
HTTPException(...status_code=503...)     : 0건
status_code=503                           : 0건 (handlers.py 의 _HTTP_STATUS_MAP 엔트리뿐)
ApiServiceUnavailableError raise          : 0건  ← 정의만 있고 호출처 없음
HTTPException.status_code == 503 path     : 없음
except Exception: raise HTTPException(503): 없음
```

→ 현재 코드에서 **실제로 503 이 생길 수 있는 경로는 0개**. `ApiServiceUnavailableError` 는 정의만 되어 있을 뿐, 실제 호출처가 없어 지금은 dead code 상태.

### 2.5 reverse proxy / Next.js / rate limiter / slowapi

- `next.config.ts` 에 `rewrites`/`proxy` 설정 없음. 프런트는 `NEXT_PUBLIC_API_URL=http://localhost:8050` 으로 **브라우저가 백엔드를 직접 호출**. Next dev 서버가 중간에서 상태를 번역할 일이 없음.
- slowapi `RateLimitExceeded` → 429. 503 아님.
- `_SafeLimiter` 는 Valkey 오류 시 `swallow_errors=True` 로 폐기 — 429/503 어느 쪽도 올리지 않음.
- `docker-compose.yml` 의 `healthcheck` 는 **정보성**이며 트래픽을 막지 않는다(Swarm/K8s 가 아니므로).
- `get_db()` 는 예외 발생 시 rollback + re-raise. 503 으로 래핑하지 않음.

### 2.6 BaseHTTPMiddleware + anyio 의 소멸 경로는?

Starlette 0.27+ 이후 `BaseHTTPMiddleware` 는 `anyio.TaskGroup` 기반으로 재작성되었고, 내부 app 예외는 원본 그대로 `ServerErrorMiddleware` 까지 전달된다. 과거(<0.27) 에는 `BaseHTTPMiddleware` 가 일부 예외를 `anyio.BrokenResourceError` 등으로 변환하기도 했지만, 그 경우에도 `ServerErrorMiddleware` 의 `Exception` 매칭에 걸려 **500** 으로 수렴한다 (`BrokenResourceError` 는 `Exception` 의 자손).

---

## 3. 왜 리뷰 당시 "503" 으로 기록되었을까

런타임 회귀 리뷰에서 관측이 가능한 가설은 다음 중 하나다 (모두 코드 원인이 아님):

1. **오타/오관측**: 개발자 도구 Network panel 에서 `500` 을 `503` 으로 잘못 옮겨 적음. 근본 원인(`item_count` 필드 누락)을 밝히는 데 초점이 있어 상태 코드 기록이 2차였음.
2. **uvicorn reload 순간의 잔상**: 백엔드를 `--reload` 로 돌리는 경우, 파일 변경 직후 찰나에 ASGI app 이 교체되면서 일부 in-flight 요청이 `ConnectionResetError` 또는 부분 응답으로 끝날 수 있음. 이때 브라우저는 "503" 대신 net::ERR_EMPTY_RESPONSE / ERR_CONNECTION_RESET 을 표시하는 것이 더 일반적이나, 일부 extension/툴체인이 이를 503 으로 대리 표기하는 경우가 있음.
3. **캐시된 과거 응답**: service worker 또는 브라우저 bfcache 가 이전 세션에서 받은 503(예: 서버 다운 중 Docker healthcheck 실패로 컨테이너가 재기동 루프에 들어간 상태에서 프록시 없이도 TCP 연결이 끊어져 클라이언트가 로컬에서 503 처럼 표시한 과거 관측) 을 노출.

세 가설 모두 현재 코드베이스와는 무관하며, 재현 불가. 유일하게 reproducible 한 fact 는 "ValueError 발생 시 현재 체인은 500 을 반환해야 한다" 이며, 이는 아래 §4 의 테스트로 고정된다.

---

## 4. 방어적 보완 — 계약 테스트 추가

`backend/tests/unit/test_unhandled_exception_mapping.py` 신설.

테스트가 검증하는 것:

| 시나리오 | 기대 | 의도 |
| --- | --- | --- |
| `ValueError` 발생 sync 엔드포인트 | HTTP 500, `error.code == internal_server_error`, 외부 노출 메시지 = "예기치 않은 오류가 발생했습니다" | **#32 핵심 계약** — 임의 middleware 가 500→503 을 덮어쓰는 회귀를 즉시 탐지 |
| `RuntimeError`, `TypeError`, `KeyError` | 동일 (HTTP 500) | 모든 미분류 Exception 서브클래스가 동일하게 500 으로 수렴하는지 확인 |
| `ApiServiceUnavailableError` | HTTP 503, `error.code == service_unavailable` | 503 은 **이 경로로만** 나와야 한다는 계약을 잠금 |
| `ApiNotFoundError`, `ApiPermissionDeniedError` | 각각 HTTP 404 / 403 | 타입별 status 가 섞이지 않는지 확인 |

이 테스트는 `register_exception_handlers()` 만 조립한 최소 FastAPI 앱을 사용하여 business-middleware 의 side effect 를 제외한 채로 핸들러 계약만 검증한다(테스트가 DB 초기화/slowapi 부트스트랩에 묶이지 않도록).

---

## 5. 운영 권고

### 5.1 원 리뷰 기록 정정 (문서)

`UI_Admin_GoldenSets_런타임회귀리뷰.md` 의 "503 → 500 매핑 차이" 항목에 본 보고서 링크와 결론(오관측 가능성, 테스트로 계약 고정)을 추가한다. 해당 수정은 본 PR 에서 함께 반영.

### 5.2 503 을 진짜 쓰는 경로 정비 (코드)

`ApiServiceUnavailableError` 는 현재 **정의만** 되어 있다. 폐쇄망(S2 ⑦) 폴백, 의존 서비스(Valkey/Milvus/OpenAI) 일시 장애 등 503 이 실제로 의미 있는 상황에서는 이 예외를 **명시적으로 raise** 해서 다음을 보장한다:

- 클라이언트가 에러 배너(#31 에서 정비) 의 "5xx → 일시 장애" 분기 힌트("잠시 후 다시 시도") 를 정확히 받음.
- 감사 로그에서 500(버그) 와 503(의존성 이슈) 이 분리되어 운영 대시보드의 오류율 해석이 명확해짐.

도입 후보 경로:

| 경로 | 적용 이유 |
| --- | --- |
| `services/alert_notifier.py` — 외부 webhook 호출 실패 | 외부 의존 일시 장애 |
| `services/rag_service.py` — OpenAI/Anthropic timeout/ConnectionError | 외부 LLM 장애 |
| `db/milvus.py` — Milvus connection failure | 벡터 DB 장애 |
| `cache/*` — Valkey circuit break | 세션 캐시 장애 |

본 PR 범위 밖(별건 작업). 본 보고서에서는 **경로 목록만 명시**.

### 5.3 uvicorn 재시작 중 관측에 대한 안내 (운영 가이드)

개발 중 `--reload` 재시작 순간에 in-flight 요청이 있으면 연결이 끊기거나 부분 응답으로 끝날 수 있음. 이때 브라우저가 표시하는 상태는 브라우저/DevTools 조합에 따라 다르며, 본 서버 스택이 503 을 낸 것이 **아님**. 향후 리뷰 문서는 500/503 구분을 **서버 로그의 실제 응답 상태**로 검증 후 기록하도록 한다 (Next.js Network panel 의 표기만으로 결론짓지 않음).

---

## 6. 보안 영향

- 본 조사는 **코드를 읽고 테스트를 추가**한 것이 전부. 프로덕션 동작 경로 변경 0건.
- 추가된 테스트 파일(`tests/unit/test_unhandled_exception_mapping.py`) 은 **test-only scope**. 빌드 아티팩트에 포함되지 않음.
- OWASP Top 10: 영향 없음. 오히려 A04(Insecure Design) 관점에서 "500 vs 503 의미 분리" 가 명시화되어 운영 관찰성(A09) 이 개선되는 방향.

---

## 7. 결론

- **코드 분석**: ValueError 경로는 500 으로 수렴. 503 매핑 경로는 존재하지 않음.
- **리뷰 기록**: 503 관측은 오관측 가능성이 압도적으로 높음. 재현 불가.
- **방어 조치**: `test_unhandled_exception_mapping.py` 신설로 계약 회귀 상시 감시.
- **후속**: `ApiServiceUnavailableError` 를 의존 서비스 경로에 의도적으로 배치 — 별건 태스크로 분리.
- **#32 상태**: 조사 완료(resolved). 실제 버그는 없었음(코드는 의도대로 500 매핑). 산출물은 본 보고서 + 계약 테스트.
