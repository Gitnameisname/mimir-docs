# Task 3-10. API 오류 모델 및 운영 관측성 규약 설계

**Phase 3 설계 기준 문서**
**산출물**: `Task10_api_error_and_observability.md`
**선행 Tasks**: Task 3-1 ~ 3-9 (API 아키텍처, 리소스 구조, 인증/인가, 버전 전략, 응답 포맷, 목록 조회 규약, 멱등성 전략, 비동기/이벤트/웹훅, AI/RAG 인터페이스)

---

## 1. 문서 목적

이 문서는 Mimir 범용 문서 플랫폼의 API 전반에서 일관되게 적용할 **오류 모델(error model)** 과 **운영 관측성(observability) 규약**을 정의한다.

이 문서가 정의하는 내용:

- 오류 응답의 공통 구조와 필드 기준
- HTTP status code와 business error code의 관계 원칙
- 오류 분류 체계 (입력/인증/리소스/운영/비동기/AI 연계)
- 운영 추적 메타데이터 규약 (request_id, trace_id 등)
- 보안 노출 제한 원칙
- async operation, webhook, idempotency, AI/RAG 연계 오류 처리 관점
- API 오류 응답 / 운영 로그 / 감사 로그의 관계

이 문서는 구현 코드, exception handler 구현, OpenAPI 전체 스키마, 로깅/모니터링 제품 선택, SLO 수치를 다루지 않는다. **실패 계약과 운영 추적의 플랫폼 표준**에 집중한다.

---

## 2. 왜 오류 모델과 관측성 규약이 필요한가

### 2-1. 성공 계약만으로는 불충분하다

성공 경로(happy path)의 응답 포맷을 표준화하는 것으로는 충분하지 않다. 실제 운영에서는 실패 응답의 일관성이 성공 응답만큼 중요하다.

- **UI 관점**: 사용자에게 "무슨 문제인지, 어떻게 해결할 수 있는지"를 적절히 표시해야 한다. 서버가 던지는 raw 오류 메시지를 UI에 직접 노출하면 보안 문제와 UX 문제가 동시에 발생한다.
- **외부 시스템/AI agent 관점**: 기계적으로 오류를 분기 처리해야 한다. `error.code`가 매번 다른 형태로 오면 클라이언트가 모든 엔드포인트마다 별도 파싱 로직을 짜야 한다.
- **운영/장애 대응 관점**: 어느 요청이 실패했는지, 어떤 actor가 무엇을 시도했는지, 어느 컴포넌트에서 실패가 발생했는지를 `request_id` 또는 `trace_id` 기준으로 추적할 수 있어야 한다.
- **보안 감사 관점**: 인증 실패, 권한 위반, 비정상 접근 패턴이 기록되고 감사 로그와 연결되어야 한다.

### 2-2. 엔드포인트마다 제각각인 오류 구조의 문제

엔드포인트별로 오류 형식이 다르면:
- 클라이언트가 모든 엔드포인트마다 다른 오류 처리 코드를 작성해야 한다.
- AI tool이 오류 응답 파싱에 실패하면 pipeline이 중단된다.
- 지원팀이 고객 문의를 받아도 `request_id` 없이는 추적이 불가능하다.
- 레거시 오류 형식이 남아 있으면 API 버전 업그레이드 시 호환성 문제가 발생한다.

**결론**: 오류 모델과 관측성 규약은 부가 요소가 아니라 플랫폼 신뢰성과 운영 가능성을 결정하는 핵심 계약이다.

---

## 3. 상위 설계 원칙

### 원칙 1. Structured Failures over Ad-hoc Messages

**의미**: 오류 응답은 문자열 메시지만 반환하지 않고 `code` / `message` / `details` 구조를 일관되게 유지한다.

**왜 필요한지**: 비구조화 오류 메시지는 기계 처리가 불가능하다. "Invalid input"만으로는 어느 필드가 왜 잘못됐는지 알 수 없다.

**설계 결정 유도**: 모든 오류 응답은 `error.code`, `error.message`, 필요 시 `error.details` 구조를 포함한다. `{"error": "something went wrong"}` 형태는 금지.

---

### 원칙 2. HTTP Semantics Plus Platform-Specific Codes

**의미**: HTTP status code는 transport/semantic 계층의 1차 신호이고, business error code는 더 세밀한 분기를 위한 2차 신호다. 두 계층을 함께 사용한다.

**왜 필요한지**: 같은 400 Bad Request 안에도 validation 실패, invalid cursor, unsupported sort 등 여러 의미가 있다. HTTP status만으로는 클라이언트가 적절히 분기할 수 없다.

**설계 결정 유도**: HTTP status는 클래스 구분(4xx/5xx), error.code는 세부 원인 구분. 두 값이 모순되면 안 된다.

---

### 원칙 3. Machine-Readable and Human-Usable

**의미**: `error.code`는 기계가 분기 처리할 수 있도록 안정적인 식별자로, `error.message`는 사람이 읽을 수 있는 서술형으로 분리한다.

**왜 필요한지**: `error.message`는 언어별로 다를 수 있고, 버전에 따라 표현이 바뀔 수 있다. 기계가 message 문자열을 파싱해서 분기 처리하면 취약하다.

**설계 결정 유도**: 클라이언트는 `error.code`를 기준으로 로직을 분기한다. `error.message`는 로그 표시와 사용자 안내에만 사용한다.

---

### 원칙 4. Consistent Error Shape Across Endpoints

**의미**: 어느 엔드포인트에서 오류가 발생하든 오류 응답의 최상위 구조가 동일하다.

**왜 필요한지**: 클라이언트가 엔드포인트마다 다른 오류 파서를 작성하지 않아도 된다. AI tool의 오류 처리 로직이 단순해진다.

**설계 결정 유도**: 오류 응답은 항상 `{"error": {...}, "meta": {...}}` 봉투를 사용한다. Task 3-5의 공통 응답 포맷과 일관성을 유지한다.

---

### 원칙 5. Traceable by Default

**의미**: 모든 오류 응답에 `meta.request_id`가 기본 포함된다. 이를 통해 클라이언트가 보고한 오류를 서버 로그와 연결할 수 있다.

**왜 필요한지**: `request_id` 없이는 고객 지원이나 장애 추적이 불가능하다.

**설계 결정 유도**: 성공 응답과 동일하게 오류 응답에도 `meta.request_id`를 항상 포함한다. 내부적으로는 `trace_id`도 연결한다.

---

### 원칙 6. Secure Disclosure

**의미**: 내부 스택 트레이스, DB 오류 메시지, 정책 엔진 상세, 민감 식별자는 외부 오류 응답에 노출하지 않는다.

**왜 필요한지**: 내부 구현 세부사항이 외부에 노출되면 공격 벡터가 될 수 있다.

**설계 결정 유도**: 외부 응답에는 최소 충분 정보만 포함. 상세 진단 정보는 내부 운영 로그에만 기록. 보안 오류는 의도적으로 메시지를 일반화한다.

---

### 원칙 7. Retry-Aware Signaling

**의미**: 재시도가 가능한 실패와 클라이언트 수정이 필요한 실패를 기계적으로 구분할 수 있어야 한다.

**왜 필요한지**: AI agent나 자동화 시스템이 실패 시 어떻게 대응할지 판단해야 한다. `retryable: true`이면 backoff 후 재시도, `retryable: false`이면 수정 후 재요청.

**설계 결정 유도**: `error.retryable` 필드를 제공한다. 5xx 일부와 `rate_limited`는 retryable=true, validation/auth/permission 오류는 retryable=false.

---

### 원칙 8. Async-Aware Failure Modeling

**의미**: async operation의 실패는 즉각 오류 응답이 아니라 operation 상태 조회를 통해 확인한다. 이 두 경로의 오류 표현 방식이 플랫폼 표준 안에 함께 포함된다.

**왜 필요한지**: 비동기 작업 실패를 polling 결과에서 발견하는 경우와 즉각 오류 응답을 받는 경우가 다른 처리 흐름을 갖는다.

**설계 결정 유도**: operation 결과 객체에도 오류 구조를 포함한다. `operation.error`는 일반 API `error` 구조와 동일한 형식을 따른다.

---

### 원칙 9. Domain-Neutral but Extensible Taxonomy

**의미**: 오류 분류 체계는 특정 도메인(문서, 버전, 노드)에 종속되지 않고 플랫폼 공통으로 적용 가능하며, 도메인 특화 코드로 확장 가능하다.

**왜 필요한지**: 플랫폼이 성장할수록 새로운 도메인과 오류 유형이 추가된다. 기반 체계가 확장 가능해야 한다.

**설계 결정 유도**: 오류 코드를 `category.specific_code` 계층 구조로 설계한다. 공통 코드와 도메인 특화 코드를 구분 가능하게 한다.

---

### 원칙 10. Observability without Overexposure

**의미**: 운영 추적에 필요한 정보(request_id, operation_id 등)는 응답에 포함하되, 과도한 내부 정보 노출은 피한다.

**왜 필요한지**: 너무 적은 정보는 추적을 불가능하게 하고, 너무 많은 정보는 보안 취약점이 된다.

**설계 결정 유도**: 응답 포함 / 조건부 포함 / 내부 로그 전용 세 가지 tier로 정보를 구분한다.

---

## 4. 오류 모델의 기본 구조

### 4-1. 오류 응답 봉투 (Task 3-5 연결)

Task 3-5의 공통 응답 포맷과 일관되게, 오류 응답은 다음 최상위 구조를 가진다.

```json
{
  "error": {
    "code": "...",
    "message": "...",
    "category": "...",
    "details": [...],
    "target": "...",
    "retryable": false,
    "help_ref": "..."
  },
  "meta": {
    "request_id": "...",
    "timestamp": "...",
    "trace_id": "...",
    "operation_id": "...",
    "idempotency_key": "..."
  }
}
```

성공 응답에 `data` + `meta`가 있다면, 오류 응답에는 `data` 대신 `error`가 온다. `meta`는 항상 포함된다.

### 4-2. 오류 객체 필드 정의

| 필드 | 필수 여부 | 설명 |
|------|----------|------|
| `code` | 필수 | 기계 판별용 안정적 식별자. `category.specific_code` 형식 권장 |
| `message` | 필수 | 사람이 읽는 오류 설명. 버전/언어에 따라 변할 수 있음 |
| `category` | 필수 | 오류 분류 (validation / auth / resource / operation / async / ai) |
| `details` | 조건부 | validation 오류 등에서 필드별 상세 오류 목록 |
| `target` | 조건부 | 오류가 특정 필드나 리소스를 가리키는 경우 명시 |
| `retryable` | 필수 | 재시도 가능 여부 (boolean). 클라이언트 자동화 판단용 |
| `help_ref` | 선택 | 관련 문서/가이드 참조 URL (공개 문서에 한해 포함 가능) |

### 4-3. 단일 오류 vs 다중 Validation 오류

**단일 오류** (일반적):
```json
{
  "error": {
    "code": "resource.not_found",
    "message": "The requested document was not found.",
    "category": "resource",
    "retryable": false
  },
  "meta": { "request_id": "req-abc123" }
}
```

**다중 Validation 오류** (요청 본문 여러 필드 실패):
```json
{
  "error": {
    "code": "validation.multiple_errors",
    "message": "The request contains multiple validation errors.",
    "category": "validation",
    "retryable": false,
    "details": [
      {
        "code": "validation.required_field",
        "message": "Field 'title' is required.",
        "target": "title"
      },
      {
        "code": "validation.max_length",
        "message": "Field 'description' exceeds maximum length of 500 characters.",
        "target": "description"
      }
    ]
  },
  "meta": { "request_id": "req-abc123" }
}
```

`details` 배열의 각 항목도 `code` / `message` / `target` 구조를 따른다.

---

## 5. HTTP Status Code와 Business Error Code 관계

### 5-1. HTTP Status만으로 불충분한 이유

HTTP status는 요청의 의미적 결과를 표현하지만, 같은 status 안에도 여러 원인이 있다.

| HTTP Status | 가능한 원인 (구분 필요) |
|-------------|----------------------|
| 400 | validation 실패, 잘못된 filter 문법, 잘못된 cursor, unsupported sort, malformed JSON |
| 401 | 토큰 없음, 토큰 만료, 잘못된 credentials |
| 403 | 리소스 접근 권한 없음, 테넌트 경계 위반, 금지된 action |
| 409 | 버전 충돌, 중복 생성 의도, 잘못된 상태 전환, idempotency key 충돌 |
| 422 | 요청 구조는 맞지만 의미적으로 처리 불가 |
| 429 | rate limit 초과 |
| 503 | 서비스 일시 불가, 의존 서비스 실패 |

클라이언트가 HTTP status만으로 분기하면 같은 400에 대해 "필드를 고쳐야 하는지", "커서를 다시 받아야 하는지", "필터 문법을 수정해야 하는지"를 구분할 수 없다.

### 5-2. Business Error Code 역할

`error.code`는 클라이언트가 자동화된 분기 처리를 할 수 있게 한다.

```
HTTP 409  +  error.code = "state.invalid_transition"
           → 상태 전환 로직 오류, 클라이언트가 workflow를 재검토해야 함

HTTP 409  +  error.code = "idempotency.key_mismatch"
           → 같은 key로 다른 payload를 전송한 경우, 요청 재검토 필요

HTTP 409  +  error.code = "resource.conflict"
           → 낙관적 동시성 충돌, 최신 상태 조회 후 재시도 가능
```

### 5-3. 일관성 원칙

- **HTTP status와 error.code가 모순되어서는 안 된다.** `401 Unauthorized`를 반환하면서 `error.code = "resource.not_found"`를 넣는 것은 금지.
- **내부 exception 클래스명을 외부 code로 직접 노출하지 않는다.** `NullPointerException`, `DocumentNotFoundException` 같은 내부 식별자가 외부 응답에 노출되면 구현 세부사항이 드러난다.
- **HTTP status는 표준 의미를 따른다.** 플랫폼이 임의로 의미를 재정의하지 않는다.

### 5-4. 주요 HTTP Status와 용도 매핑

| HTTP Status | 플랫폼 용도 | retryable 방향 |
|-------------|-----------|--------------|
| 200 OK | 성공 | — |
| 201 Created | 리소스 생성 성공 | — |
| 202 Accepted | 비동기 작업 접수 | — |
| 204 No Content | 삭제/변경 성공, 본문 없음 | — |
| 400 Bad Request | 입력 오류, 문법 오류 | false |
| 401 Unauthorized | 인증 필요/실패 | false (갱신 후 재시도) |
| 403 Forbidden | 권한 없음 | false |
| 404 Not Found | 리소스 없음 | false |
| 409 Conflict | 충돌, 중복, 상태 전환 오류 | 상황에 따라 다름 |
| 410 Gone | 삭제된 API 버전 | false |
| 422 Unprocessable Entity | 의미적으로 처리 불가 | false |
| 429 Too Many Requests | Rate limit 초과 | true (backoff) |
| 500 Internal Server Error | 서버 내부 오류 | true (일부) |
| 502 Bad Gateway | 업스트림 오류 | true |
| 503 Service Unavailable | 서비스 일시 불가 | true |

---

## 6. 오류 분류 체계

### 6-1. 입력 / 계약 오류 (category: validation)

클라이언트가 잘못된 요청을 보낸 경우. 대부분 retryable=false.

| error.code | 상황 | HTTP Status |
|-----------|------|-------------|
| `validation.required_field` | 필수 필드 누락 | 400 |
| `validation.invalid_field` | 필드 값 형식/범위 오류 | 400 |
| `validation.multiple_errors` | 복수 필드 validation 실패 | 400 |
| `validation.malformed_request` | JSON 문법 오류 등 | 400 |
| `query.invalid_filter` | 지원하지 않는 filter 문법 | 400 |
| `query.unsupported_filter` | 지원하지 않는 filter 필드 | 400 |
| `query.unsupported_sort` | 지원하지 않는 sort 필드 | 400 |
| `query.invalid_cursor` | 잘못된 cursor 값 | 400 |

**노출 수준**: 필드명, 오류 이유를 details에 포함해도 무방. 내부 DB 스키마는 노출 금지.

---

### 6-2. 인증 / 보안 오류 (category: auth)

인증 실패 또는 권한 위반. 보안상 상세 정보 노출을 제한한다.

| error.code | 상황 | HTTP Status |
|-----------|------|-------------|
| `auth.authentication_required` | 인증 헤더 없음 | 401 |
| `auth.token_expired` | 토큰 만료 | 401 |
| `auth.token_invalid` | 유효하지 않은 토큰 | 401 |
| `auth.permission_denied` | 리소스 접근 권한 없음 | 403 |
| `auth.tenant_scope_violation` | 테넌트 경계 위반 시도 | 403 |
| `auth.forbidden_action` | 역할상 금지된 action | 403 |

**노출 수준**: "무엇이 없어서 실패"했는지는 최소한으로. "왜 권한이 없는지" (ACL 세부 정책, 역할 계층)는 노출 금지.

**보안 주의**: 권한 없는 리소스에 대해 403 대신 404를 반환하는 것이 리소스 존재 여부 노출을 막는 데 유리한 경우가 있다. 단, 이 선택은 도메인별로 정책을 정하고 일관되게 적용한다.

---

### 6-3. 리소스 / 상태 오류 (category: resource / state)

리소스 존재나 상태와 관련된 오류.

| error.code | 상황 | HTTP Status |
|-----------|------|-------------|
| `resource.not_found` | 요청한 리소스 없음 | 404 |
| `resource.already_exists` | 이미 존재하는 리소스 생성 시도 | 409 |
| `resource.conflict` | 동시성 충돌 (낙관적 잠금) | 409 |
| `state.invalid_transition` | 허용되지 않은 상태 전환 | 409 |
| `state.stale_version` | 오래된 버전 기반 수정 시도 | 409 |

**retryable 기준**:
- `resource.conflict`, `state.stale_version`: 최신 상태 조회 후 재시도 가능 → retryable=true (하지만 즉각 재시도가 아닌 수정 후 재시도)
- `state.invalid_transition`: 상태 모델을 이해한 후 올바른 action으로 재요청 → retryable=false

---

### 6-4. 운영 / 제한 오류 (category: operation)

서버측 또는 인프라 수준 문제. 일반적으로 retryable.

| error.code | 상황 | HTTP Status |
|-----------|------|-------------|
| `operation.rate_limited` | Rate limit 초과 | 429 |
| `operation.service_unavailable` | 서비스 일시 불가 | 503 |
| `operation.timeout` | 처리 시간 초과 | 503 또는 504 |
| `operation.dependency_failure` | 의존 서비스 실패 | 502 또는 503 |
| `operation.internal_error` | 서버 내부 오류 | 500 |

**노출 수준**: 구체적인 의존 서비스 이름, 내부 오류 상세는 외부 응답에 포함하지 않는다. `request_id`로 추적 가능하게 하는 것으로 충분하다.

---

### 6-5. 비동기 / 통합 오류 (category: async / webhook / idempotency / ai)

Task 3-7, 3-8, 3-9와 연결되는 확장 오류 범주.

| error.code | 상황 | HTTP Status |
|-----------|------|-------------|
| `async.operation_failed` | 비동기 작업 최종 실패 (operation 결과) | — (operation 상태) |
| `async.operation_not_ready` | 아직 처리 중인 결과 조회 시도 | 202 또는 404 |
| `webhook.delivery_failed` | 웹훅 delivery 실패 (delivery 기록) | — (delivery 상태) |
| `webhook.signature_invalid` | 웹훅 수신 측의 HMAC 검증 실패 | 400 |
| `webhook.subscription_invalid` | 잘못된 웹훅 구독 구성 | 422 |
| `idempotency.key_missing` | Idempotency-Key 헤더 누락 (필수인 경우) | 400 |
| `idempotency.key_mismatch` | 동일 key로 다른 payload 전송 | 409 |
| `idempotency.replay_unavailable` | replay 기간 만료 | 410 |
| `ai.indexing_not_ready` | 인덱싱 미완료 상태에서 retrieval 시도 | 503 |
| `ai.retrieval_unavailable` | retrieval 서비스 일시 불가 | 503 |
| `ai.citation_not_found` | citation 대상 리소스 없음 | 404 |
| `ai.ingestion_in_progress` | ingestion 진행 중 (완료 전 조회) | 202 |

---

## 7. 주요 오류 유형별 응답 원칙

### 7-1. Validation Failure

- **HTTP Status**: 400 Bad Request
- **대표 error.code**: `validation.invalid_field`, `validation.multiple_errors`
- **message 원칙**: "어느 필드가 왜 잘못됐는지" 명확히. 단, 보안상 민감한 필드 이름 노출은 주의.
- **details**: 필드별 오류 목록 포함. `target` 필드에 필드명 명시.
- **retryable**: false (클라이언트가 입력을 수정해야 함)
- **보안**: 내부 DB 스키마, 정규식 패턴, 비즈니스 규칙 상세 노출 주의.
- **운영 메타데이터**: request_id 포함. 내부 로그에는 요청 본문 요약 기록.

---

### 7-2. Authentication Failure

- **HTTP Status**: 401 Unauthorized
- **대표 error.code**: `auth.authentication_required`, `auth.token_expired`, `auth.token_invalid`
- **message 원칙**: "인증이 필요하다" 또는 "토큰이 만료됐다" 수준으로 간략하게. 토큰 내용, 서명 알고리즘, 검증 실패 상세 노출 금지.
- **details**: 통상 포함하지 않음.
- **retryable**: false (갱신된 토큰으로 재요청 필요)
- **보안**: `WWW-Authenticate` 응답 헤더 포함 고려. 어떤 검증에서 실패했는지 상세 노출 금지.
- **운영 메타데이터**: request_id. 내부 로그에는 token source, tenant, actor 정보 기록.

---

### 7-3. Authorization Failure

- **HTTP Status**: 403 Forbidden (또는 정책에 따라 404)
- **대표 error.code**: `auth.permission_denied`, `auth.tenant_scope_violation`, `auth.forbidden_action`
- **message 원칙**: "이 작업을 수행할 권한이 없습니다" 수준. ACL 세부, 역할 계층, 정책 규칙 노출 금지.
- **details**: 포함하지 않음.
- **retryable**: false
- **보안**: 리소스 존재 노출 방지를 위해 일부 경우 404 반환 정책을 수립해야 함. 이 선택은 도메인별로 문서화하고 일관 적용.
- **운영 메타데이터**: request_id. 내부 로그에는 actor, resource, denied_reason 기록. 감사 로그 대상.

---

### 7-4. Not Found

- **HTTP Status**: 404 Not Found
- **대표 error.code**: `resource.not_found`
- **message 원칙**: "요청한 리소스를 찾을 수 없습니다." 리소스 타입은 명시해도 무방 ("Document not found.").
- **details**: 리소스 식별자 명시 가능 (`"target": "document_id: doc-abc"`). 단, 보안 정책상 리소스 존재를 노출하지 않아야 하는 경우 생략.
- **retryable**: false
- **보안**: 권한 없는 리소스에 대해 404를 반환하는 경우, 리소스 자체의 존재 유무를 노출하지 않는 것이 의도임을 내부 정책으로 명시.
- **운영 메타데이터**: request_id. 내부 로그에는 실제 조회 시도한 식별자 기록.

---

### 7-5. Conflict

- **HTTP Status**: 409 Conflict
- **대표 error.code**: `resource.conflict`, `state.invalid_transition`, `state.stale_version`, `resource.already_exists`
- **message 원칙**: 충돌의 성격에 따라 다름. "이미 존재하는 리소스입니다." / "현재 상태에서는 이 전환이 허용되지 않습니다." / "다른 변경과 충돌이 발생했습니다."
- **details**: 충돌 대상 리소스 식별자, 현재 상태 포함 가능. 단, 민감 데이터 포함 금지.
- **retryable**: code에 따라 다름. `resource.conflict` → 최신 상태 조회 후 재시도. `state.invalid_transition` → false.
- **운영 메타데이터**: request_id, 충돌 대상 resource_id.

---

### 7-6. Idempotency Mismatch

- **HTTP Status**: 409 Conflict
- **대표 error.code**: `idempotency.key_mismatch`
- **message 원칙**: "같은 Idempotency-Key로 다른 내용의 요청이 전송되었습니다."
- **details**: 기존 요청의 timestamp 또는 요약 정보 포함 가능. 기존 요청 본문 반환은 금지.
- **retryable**: false (새로운 key 사용 또는 요청 내용 확인 필요)
- **운영 메타데이터**: request_id, idempotency_key, 기존 요청의 request_id (있는 경우).

---

### 7-7. Invalid State Transition

- **HTTP Status**: 409 Conflict 또는 422 Unprocessable Entity
- **대표 error.code**: `state.invalid_transition`
- **message 원칙**: "현재 상태(published)에서 이 전환(draft)은 허용되지 않습니다." 현재 상태 명시 가능.
- **details**: `current_state`, `requested_transition` 포함 가능.
- **retryable**: false
- **운영 메타데이터**: request_id, resource_id, current_state.

---

### 7-8. Rate Limited

- **HTTP Status**: 429 Too Many Requests
- **대표 error.code**: `operation.rate_limited`
- **message 원칙**: "요청 한도를 초과했습니다. 잠시 후 다시 시도하세요."
- **details**: 포함하지 않음 (rate limit 정책 상세 노출 방지).
- **retryable**: true
- **응답 헤더**: `Retry-After` 헤더로 대기 시간 안내. `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` 고려.
- **보안**: 구체적인 rate limit 수치, quota 설정 상세 노출 주의.
- **운영 메타데이터**: request_id. 내부 로그에는 actor, endpoint, current_count 기록.

---

### 7-9. Async Operation Failed

- **HTTP Status**: 즉각 오류 없음 — operation 상태 조회 결과에 포함
- **대표 error.code**: `async.operation_failed`
- **표현 위치**: `GET /operations/{operationId}` 응답의 `result.error` 또는 `error` 필드
- **message 원칙**: "처리 중 오류가 발생했습니다. operation_id를 기준으로 지원팀에 문의하세요."
- **details**: 작업 유형에 따라 실패 원인 요약 포함 가능. 내부 스택 트레이스 노출 금지.
- **retryable**: 작업 유형에 따라 다름. 일시적 의존 서비스 실패 → true. 입력 오류 → false.
- **운영 메타데이터**: request_id (원본 요청), operation_id, event_id (있는 경우).

---

### 7-10. Upstream / Dependency Failure

- **HTTP Status**: 502 Bad Gateway 또는 503 Service Unavailable
- **대표 error.code**: `operation.dependency_failure`, `operation.service_unavailable`
- **message 원칙**: "일시적으로 서비스를 처리할 수 없습니다. 잠시 후 다시 시도하세요."
- **details**: 어떤 의존 서비스가 실패했는지 외부에 노출하지 않음.
- **retryable**: true
- **보안**: 내부 서비스 이름, 의존 그래프, 토폴로지 노출 금지.
- **운영 메타데이터**: request_id. 내부 로그에는 failed_dependency, error_detail 기록.

---

### 7-11. AI / Retrieval / Indexing Not Ready or Unavailable

- **HTTP Status**: 503 Service Unavailable (일시적), 202 Accepted (처리 중)
- **대표 error.code**: `ai.indexing_not_ready`, `ai.retrieval_unavailable`, `ai.ingestion_in_progress`
- **message 원칙**: "인덱싱이 아직 완료되지 않았습니다." / "검색 서비스를 일시적으로 사용할 수 없습니다."
- **details**: last_indexed_version_id, estimated_ready_at (있는 경우) 포함 가능.
- **retryable**: true (인덱싱 완료 또는 서비스 복구 후 재시도)
- **운영 메타데이터**: request_id, document_id, version_id, operation_id (있는 경우).

---

### 7-12. Webhook Delivery / Signature Failure

**Delivery 실패** (내부 운영 개념, API 직접 응답 아님):
- webhook delivery 기록의 `status: failed`, `error_summary` 필드로 표현
- 외부 클라이언트가 조회하는 경우 delivery 상태 리소스에 포함

**Signature 검증 실패** (웹훅 수신 측에서 반환):
- **HTTP Status**: 400 Bad Request
- **대표 error.code**: `webhook.signature_invalid`
- **message**: "웹훅 서명 검증에 실패했습니다."
- **retryable**: false (보안 설정 확인 필요)
- **보안**: 구체적인 서명 알고리즘, secret 관련 정보 노출 금지.

---

## 8. 운영 추적 메타데이터 규약

### 8-1. 항상 응답에 포함할 항목

| 메타데이터 필드 | 설명 | 비고 |
|-------------|-----|------|
| `meta.request_id` | 이 API 요청의 고유 식별자 (UUID) | 성공/오류 응답 모두 포함 |
| `meta.timestamp` | 서버 처리 시각 (ISO 8601 UTC) | 성공/오류 응답 모두 포함 |

### 8-2. 조건부 포함 항목

| 메타데이터 필드 | 포함 조건 | 설명 |
|-------------|---------|------|
| `meta.trace_id` | 분산 추적이 활성화된 환경 | 내부 trace 연결용. 외부에 노출 시 의미 있는 값이어야 함 |
| `meta.operation_id` | 비동기 작업과 관련된 요청/응답 | 작업 추적 연결용 |
| `meta.idempotency_key` | Idempotency-Key 헤더가 포함된 요청 | 클라이언트가 전송한 key를 응답에 반영하여 확인 가능 |
| `meta.webhook_delivery_id` | 웹훅 delivery 관련 오류 응답 | delivery 추적용 |
| `meta.event_id` | 이벤트와 연결된 응답 | 이벤트 추적용 |

### 8-3. 외부 응답 비노출 / 내부 로그 전용 항목

| 항목 | 이유 |
|------|------|
| `actor_id`, `tenant_id` | 요청자 정보. 보안상 응답 본문에 항상 포함하지 않음. 내부 로그와 감사 로그에 기록. |
| `internal_error_id` | 내부 오류 추적 ID. `request_id`로 연결 가능하면 충분. |
| 스택 트레이스 | 절대 외부 응답에 포함하지 않음 |
| DB 쿼리, 내부 서비스 이름 | 외부 노출 금지 |

### 8-4. Retry-After와 Rate Limit 헤더

429 응답 시 응답 본문 외에 HTTP 헤더로도 추적 정보 제공.

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1711958400
```

---

## 9. 보안 노출 제한 원칙

### 9-1. 절대 외부 응답에 포함하지 말 것

- 내부 스택 트레이스 (stack trace)
- DB 오류 메시지 (SQLException, PostgreSQL error code 등)
- 내부 서비스 이름, 호스트명, 포트
- 정책 엔진 내부 규칙, ACL 세부 내용
- 다른 사용자/테넌트의 식별자나 존재 정보
- credential, token 값, secret

### 9-2. 보안 오류의 의도적 일반화

인증/권한 오류 메시지는 공격자에게 정보를 주지 않도록 의도적으로 일반화한다.

```
비권장: "사용자 U123의 문서 doc-abc에 대한 읽기 권한이 없습니다."
권장:   "이 작업을 수행할 권한이 없습니다."

비권장: "토큰의 sub claim이 없습니다."
권장:   "인증에 실패했습니다. 유효한 토큰으로 다시 시도하세요."
```

### 9-3. Not Found vs Forbidden 선택

리소스 존재 자체를 숨겨야 하는 경우(권한 없는 사용자에게 문서 존재 여부를 알리지 않아야 함), 403 대신 404를 반환할 수 있다. 이 경우:
- 해당 리소스 타입에 대해 정책으로 명시하고 일관 적용한다.
- 내부 운영 로그에는 실제 403 이유와 함께 의도적 404 반환임을 기록한다.
- 보안 감사 로그에는 실제 거부 사유를 기록한다.

### 9-4. AI Consumer도 동일 원칙 적용

AI consumer라고 해서 내부 오류 상세를 더 많이 받아서는 안 된다. AI 파이프라인 디버깅을 위한 상세 오류 정보는 내부 운영 로그와 감사 로그를 통해 접근한다.

### 9-5. 외부 응답 정보의 3단계 분류

| 단계 | 포함 정보 | 목적 |
|------|---------|------|
| 외부 응답 | error.code, error.message, error.retryable, meta.request_id | 클라이언트 분기 처리 및 최소 추적 |
| 운영 로그 | 위 + trace_id, actor summary, endpoint, latency, 실패 원인 요약 | 진단 및 장애 분석 |
| 감사 로그 | 위 + actor_id, tenant_id, resource_id, action, result | 보안/컴플라이언스 기록 |

---

## 10. Async Operation / Webhook / Idempotency / AI 연계 오류 관점

### 10-1. Async Operation 오류

비동기 작업에서 오류는 두 가지 시점에 발생할 수 있다.

**즉각 응답 시점** (작업 접수 단계):
- 잘못된 요청으로 작업 자체를 시작할 수 없는 경우 → 일반 4xx 오류 응답
- 서비스 불가로 작업을 받을 수 없는 경우 → 503 응답

**Operation 결과 조회 시점** (작업 완료 후):
- `GET /operations/{operationId}` 응답에서 `status: failed` + `error` 객체로 표현
- `error` 객체 구조는 일반 API 오류 구조와 동일하게 적용

```json
{
  "data": {
    "operation_id": "op-xyz",
    "status": "failed",
    "error": {
      "code": "async.operation_failed",
      "message": "문서 인덱싱 중 오류가 발생했습니다.",
      "category": "async",
      "retryable": true
    }
  },
  "meta": { "request_id": "req-poll-123", "operation_id": "op-xyz" }
}
```

**결과 아직 미준비 표현**:
- `status: running` → 202 또는 200으로 현재 상태 반환. 오류 응답이 아님.
- 클라이언트는 polling 계속.

---

### 10-2. Webhook 오류

**Delivery 실패** (외부 서버가 응답 안 함 또는 비정상 응답):
- webhook delivery 리소스의 `status: failed`, `attempt_count`, `last_error_summary`로 기록
- 클라이언트가 delivery 상태를 조회할 때 확인 가능
- delivery 자체가 API 오류 응답이 아님

**Subscription 설정 오류**:
- 잘못된 target_url, 지원하지 않는 이벤트 타입 → 422 + `webhook.subscription_invalid`

**수신 측 Signature 검증 실패**:
- 플랫폼이 webhook을 수신하는 외부 서비스가 HMAC 검증 실패 시 400 반환
- 플랫폼 자체가 webhook을 보낼 때 이 응답을 수신하면 delivery failed로 처리

---

### 10-3. Idempotency 오류

| 시나리오 | error.code | HTTP Status | 설명 |
|---------|-----------|-------------|------|
| key 필수인데 누락 | `idempotency.key_missing` | 400 | Idempotency-Key 헤더 없음 |
| 동일 key, 다른 payload | `idempotency.key_mismatch` | 409 | 기존 요청과 충돌 |
| replay 기간 만료 | `idempotency.replay_unavailable` | 410 | 보관된 응답 없음 |
| replay 정상 반환 | — (오류 아님) | 기존 응답 status | `meta.idempotency.replayed: true` |

idempotency 오류는 일반 conflict 오류와 같은 409를 사용하지만 code로 구분한다.

---

### 10-4. AI / RAG 연계 오류

AI/RAG 관련 오류는 기존 오류 체계 안으로 편입한다. 별도 예외 처리 체계를 만들지 않는다.

| 시나리오 | error.code | HTTP Status | retryable |
|---------|-----------|-------------|---------|
| 인덱싱 미완료 상태에서 retrieval | `ai.indexing_not_ready` | 503 | true |
| retrieval 서비스 일시 불가 | `ai.retrieval_unavailable` | 503 | true |
| citation 대상 노드 없음 | `ai.citation_not_found` | 404 | false |
| ingestion 진행 중 중복 요청 | `ai.ingestion_in_progress` | 409 | true (완료 후) |
| 권한 범위 밖 retrieval 시도 | `auth.permission_denied` | 403 | false |

`ai.indexing_not_ready` 응답에 `estimated_ready_at`을 선택적으로 포함하면 AI 파이프라인이 재시도 타이밍을 결정하는 데 도움이 된다.

---

## 11. 감사 추적 / 운영 로그 / API 오류 응답의 관계

세 가지는 같은 실패를 다른 목적으로 기록한다.

| 구분 | 목적 | 주요 내용 | 연결 키 |
|------|------|---------|---------|
| API 오류 응답 | 클라이언트 분기 처리 및 최소 정보 전달 | error.code, error.message, retryable, request_id | request_id |
| 운영 로그 | 진단, 장애 분석, 성능 추적 | request_id, trace_id, endpoint, latency, actor summary, error detail, dependency 상태 | request_id + trace_id |
| 감사 로그 | 보안/컴플라이언스 — 누가 무엇을 시도했고 어떤 결과가 났는지 | actor_id, tenant_id, resource_id, action, result (allowed/denied), request_id | request_id + actor_id |

### 11-1. 정보 범위의 차이

같은 `permission_denied` 실패라도:

- **API 오류 응답**: "이 작업을 수행할 권한이 없습니다. (request_id: req-abc)"
- **운영 로그**: request_id=req-abc, actor=service-account-rag, endpoint=POST /searches, denied_reason=tenant_scope_violation, elapsed_ms=12
- **감사 로그**: actor_id=sa-rag-001, tenant_id=tenant-A, resource_type=document, action=search, result=denied, reason=cross_tenant_access, request_id=req-abc, timestamp=...

### 11-2. request_id가 세 가지를 연결한다

클라이언트가 지원팀에 `request_id`를 전달하면:
- 운영 로그에서 해당 요청의 상세 처리 과정 조회 가능
- 감사 로그에서 해당 요청의 보안 컨텍스트 조회 가능
- trace_id로 내부 분산 추적 시스템 연결 가능

**따라서 `request_id`는 API 오류 응답에서 항상 노출해야 한다.** 이것이 최소 충분 운영 정보다.

### 11-3. 감사 로그 대상 이벤트

모든 API 실패가 감사 로그 대상은 아니다. 다음은 반드시 감사 로그에 기록한다.

- 인증 실패 (성공/실패 모두)
- 권한 거부 (403 / 의도적 404)
- 보안 정책 위반 시도 (테넌트 경계 위반)
- 민감 리소스에 대한 접근 (문서 삭제, 권한 변경)
- AI processing 시작 (actor 기록 필요)

---

## 12. 예시 오류 응답 및 운영 메타데이터 예시

### 예시 1 — Validation Error

```json
{
  "error": {
    "code": "validation.multiple_errors",
    "message": "요청에 유효성 오류가 있습니다.",
    "category": "validation",
    "retryable": false,
    "details": [
      {
        "code": "validation.required_field",
        "message": "'title' 필드는 필수입니다.",
        "target": "title"
      },
      {
        "code": "validation.max_length",
        "message": "'description'은 500자를 초과할 수 없습니다.",
        "target": "description"
      }
    ]
  },
  "meta": {
    "request_id": "req-0001",
    "timestamp": "2025-03-15T10:00:00Z"
  }
}
```
*단일 오류 code `validation.multiple_errors` + details 배열로 복수 오류를 일관된 구조로 표현.*

---

### 예시 2 — Permission Denied

```json
{
  "error": {
    "code": "auth.permission_denied",
    "message": "이 작업을 수행할 권한이 없습니다.",
    "category": "auth",
    "retryable": false
  },
  "meta": {
    "request_id": "req-0002",
    "timestamp": "2025-03-15T10:01:00Z"
  }
}
```
*ACL 세부, 역할 계층 없이 최소 정보만. details 없음.*

---

### 예시 3 — Not Found

```json
{
  "error": {
    "code": "resource.not_found",
    "message": "요청한 문서를 찾을 수 없습니다.",
    "category": "resource",
    "retryable": false
  },
  "meta": {
    "request_id": "req-0003",
    "timestamp": "2025-03-15T10:02:00Z"
  }
}
```
*리소스 타입("문서")은 명시. 식별자는 target 필드에 선택적으로 포함.*

---

### 예시 4 — Conflict (상태 전환 오류)

```json
{
  "error": {
    "code": "state.invalid_transition",
    "message": "현재 상태(archived)에서 publish 전환은 허용되지 않습니다.",
    "category": "state",
    "retryable": false,
    "details": [
      {
        "code": "state.invalid_transition",
        "message": "archived 상태에서 허용된 전환: [restore]",
        "target": "status"
      }
    ]
  },
  "meta": {
    "request_id": "req-0004",
    "timestamp": "2025-03-15T10:03:00Z"
  }
}
```
*현재 상태와 허용된 전환을 details에 포함하여 클라이언트가 다음 action을 결정할 수 있음.*

---

### 예시 5 — Idempotency Mismatch

```json
{
  "error": {
    "code": "idempotency.key_mismatch",
    "message": "동일한 Idempotency-Key로 내용이 다른 요청이 이미 처리되었습니다.",
    "category": "idempotency",
    "retryable": false
  },
  "meta": {
    "request_id": "req-0005",
    "timestamp": "2025-03-15T10:04:00Z",
    "idempotency_key": "client-uuid-abc-123"
  }
}
```
*meta에 idempotency_key 반영하여 클라이언트가 어느 key가 충돌했는지 확인 가능.*

---

### 예시 6 — Rate Limited

```json
{
  "error": {
    "code": "operation.rate_limited",
    "message": "요청 한도를 초과했습니다. 잠시 후 다시 시도하세요.",
    "category": "operation",
    "retryable": true
  },
  "meta": {
    "request_id": "req-0006",
    "timestamp": "2025-03-15T10:05:00Z"
  }
}
```

응답 헤더:
```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
```
*retryable=true + Retry-After 헤더로 재시도 타이밍 안내.*

---

### 예시 7 — Async Operation Failed (operation 결과)

```json
{
  "data": {
    "operation_id": "op-reindex-789",
    "type": "document.reindex",
    "status": "failed",
    "error": {
      "code": "async.operation_failed",
      "message": "문서 인덱싱 중 오류가 발생했습니다.",
      "category": "async",
      "retryable": true
    },
    "started_at": "2025-03-15T09:50:00Z",
    "failed_at": "2025-03-15T09:52:30Z"
  },
  "meta": {
    "request_id": "req-poll-007",
    "timestamp": "2025-03-15T10:06:00Z",
    "operation_id": "op-reindex-789"
  }
}
```
*operation 결과에도 동일한 error 구조 적용. 즉각 응답이 아닌 polling 결과.*

---

### 예시 8 — AI Retrieval Not Ready

```json
{
  "error": {
    "code": "ai.indexing_not_ready",
    "message": "요청한 문서 버전의 인덱싱이 아직 완료되지 않았습니다.",
    "category": "ai",
    "retryable": true,
    "details": [
      {
        "code": "ai.indexing_not_ready",
        "message": "현재 인덱싱 중입니다.",
        "target": "version_id: ver-45"
      }
    ]
  },
  "meta": {
    "request_id": "req-0008",
    "timestamp": "2025-03-15T10:07:00Z",
    "operation_id": "op-reindex-789"
  }
}
```
*operation_id로 인덱싱 작업 상태를 추적할 수 있도록 연결.*

---

### 예시 9 — Webhook Signature Invalid

```json
{
  "error": {
    "code": "webhook.signature_invalid",
    "message": "웹훅 서명 검증에 실패했습니다.",
    "category": "webhook",
    "retryable": false
  },
  "meta": {
    "request_id": "req-0009",
    "timestamp": "2025-03-15T10:08:00Z"
  }
}
```
*서명 알고리즘, secret 관련 정보 노출 없이 최소 정보만.*

---

### 예시 10 — Dependency Unavailable

```json
{
  "error": {
    "code": "operation.dependency_failure",
    "message": "일시적으로 서비스를 처리할 수 없습니다. 잠시 후 다시 시도하세요.",
    "category": "operation",
    "retryable": true
  },
  "meta": {
    "request_id": "req-0010",
    "timestamp": "2025-03-15T10:09:00Z"
  }
}
```
*내부 의존 서비스 이름 노출 없음. request_id로 내부 로그에서 세부 확인.*

---

### 같은 HTTP Status에서 code가 다른 비교 예시

```
HTTP 409 + code: "resource.already_exists"
 → 이미 같은 이름의 문서가 존재함. 다른 이름으로 생성하거나 기존 문서를 사용해야 함.

HTTP 409 + code: "state.invalid_transition"
 → 현재 상태에서 요청한 전환 불가. 다른 action을 선택해야 함.

HTTP 409 + code: "idempotency.key_mismatch"
 → 같은 Idempotency-Key로 다른 내용 전송. key를 확인하거나 새 key 사용 필요.
```

---

### retryable=true vs retryable=false 비교

```
retryable: true  → AI pipeline이 exponential backoff 후 자동 재시도 가능
  - operation.rate_limited: 429, Retry-After 후 재시도
  - operation.service_unavailable: 503, 잠시 후 재시도
  - ai.indexing_not_ready: 503, 인덱싱 완료 후 재시도

retryable: false → 클라이언트가 요청 자체를 수정해야 함
  - validation.invalid_field: 입력 필드 수정 후 새 요청
  - auth.permission_denied: 권한 취득 또는 요청 포기
  - state.invalid_transition: 다른 action 선택
  - idempotency.key_mismatch: 새 Idempotency-Key 사용
```

---

## 13. Error Code 체계 작성 방향

### 13-1. 네이밍 원칙

- **형식**: `category.specific_code` — 점(.) 구분자 계층 구조
- **소문자 snake_case**: `validation.required_field`, `auth.permission_denied`
- **짧고 안정적**: 한 번 정한 code는 breaking change 없이 변경하지 않는다
- **내부 exception 클래스명 의존 금지**: `NullPointerException`, `NotFoundException` 직접 사용 금지
- **영어 기준**: 언어 중립적 식별자

### 13-2. Category 체계

| Category | 설명 | 예시 |
|---------|-----|------|
| `validation` | 입력/계약 오류 | `validation.required_field` |
| `query` | 쿼리 파라미터 오류 | `query.invalid_cursor` |
| `auth` | 인증/권한 오류 | `auth.permission_denied` |
| `resource` | 리소스 존재/중복 오류 | `resource.not_found` |
| `state` | 상태 전환 오류 | `state.invalid_transition` |
| `idempotency` | 멱등성 오류 | `idempotency.key_mismatch` |
| `operation` | 운영/제한 오류 | `operation.rate_limited` |
| `async` | 비동기 작업 오류 | `async.operation_failed` |
| `webhook` | 웹훅 오류 | `webhook.signature_invalid` |
| `ai` | AI/RAG 연계 오류 | `ai.indexing_not_ready` |

### 13-3. 공통 코드 vs 도메인 특화 코드

- **공통 코드**: 모든 리소스에 적용 가능. `resource.not_found`, `auth.permission_denied` 등
- **도메인 특화 코드**: 특정 도메인에서만 의미 있는 경우. 예: `document.version_limit_exceeded`. 기반 category는 공통 체계를 따르고 specific_code만 도메인 특화.

### 13-4. 전체 Catalog는 구현 Phase에서

이 문서에서는 체계 기준만 확립한다. 전체 error code catalog 완성은 구현 Phase에서 작성한다.

---

## 14. 후속 Task 및 구현 Phase에 전달할 기준

### Task 3-11 (대표 API 시나리오)으로 전달

- 각 대표 시나리오에서 오류 경로(failure path)도 포함하여 오류 응답과 추적 메타데이터가 일관되게 적용되는지 검증
- `request_id` 기준 성공/오류 응답의 일관성 확인
- async operation 시나리오에서 operation 결과의 오류 표현 검증

### Task 3-12 (Phase 3 통합 정리)으로 전달

- 전체 Phase 3 통합 문서에 error model과 observability 규약을 공식 계약으로 반영
- implementation handoff 문서에 error code category 체계, meta 필드 표준, 보안 노출 원칙 포함

### 구현 Phase로 전달

- **Exception Handler**: 플랫폼 공통 exception handler가 이 문서의 오류 구조를 자동으로 생성
- **Request Context Middleware**: 모든 요청에 request_id 생성 및 propagation. 응답에 meta.request_id 자동 포함
- **Logging Correlation**: request_id + trace_id를 기준으로 운영 로그와 감사 로그를 연결
- **Audit Pipeline**: 감사 로그 대상 이벤트 목록 및 기록 항목 구현
- **Operation/Webhook Failure Handling**: async operation 실패 시 error 구조 일관 적용. webhook delivery 실패 기록 방식
- **AI/RAG 오류 처리**: indexing_not_ready, retrieval_unavailable의 retryable 신호와 Retry-After 패턴

### 별도 기술 설계로 전달

- Rate limit 정책 상세 수치 결정
- Concurrency control 상세 설계 (낙관적 잠금 구체 구현)
- Resilience 패턴 (circuit breaker, bulkhead) 선택
- SLO/SLA 수치 정의
- Alert rule 설계
- Observability stack 제품 선택 (tracing, metrics, log aggregation)

---

## 15. 결론

이 문서에서 확립한 핵심 방향은 다음과 같다.

**실패 계약도 공식 계약이다.** 성공 응답만큼 오류 응답의 일관성이 플랫폼 신뢰성을 결정한다. `{"error": {...}, "meta": {...}}` 봉투와 `category.specific_code` 체계로 모든 엔드포인트의 오류를 표준화한다.

**HTTP status + business error code 이중 계층을 사용한다.** HTTP status는 클래스 구분, error.code는 세부 원인 구분. 두 계층이 모순되지 않는다.

**retryable 신호로 클라이언트 자동화를 지원한다.** AI agent와 자동화 시스템이 재시도 가능 여부를 기계적으로 판단할 수 있다.

**request_id가 세 계층(API 응답/운영 로그/감사 로그)을 연결한다.** 이것이 최소 충분 운영 추적 정보다.

**보안 노출은 의도적으로 제한한다.** 스택 트레이스, 내부 서비스 이름, ACL 세부는 외부 응답에서 제외. AI consumer도 예외 없이 적용.

**async, webhook, idempotency, AI 오류는 기존 체계 안으로 편입한다.** 별도 예외 체계를 만들지 않는다.

---

## 자체 점검 요약

### 오류 모델 기본 shape 요약
- 최상위 구조: `{"error": {...}, "meta": {...}}`
- error 객체: `code` (필수, 기계 판별용), `message` (필수, 사람 읽기용), `category` (필수), `details` (조건부), `target` (조건부), `retryable` (필수), `help_ref` (선택)
- meta 객체: `request_id` + `timestamp` 항상 포함. `trace_id`, `operation_id`, `idempotency_key` 조건부

### HTTP Status / Business Error Code 관계 요약
- HTTP status: 1차 semantic 신호 (클래스 구분)
- error.code: 2차 신호 (세부 원인 분기)
- 두 값이 모순되면 안 됨
- 내부 exception 클래스명 외부 노출 금지

### 주요 오류 범주 요약
- 5대 범주: validation / auth / resource+state / operation / async+webhook+idempotency+ai
- 10개 category 코드 체계: validation, query, auth, resource, state, idempotency, operation, async, webhook, ai

### 운영 추적 메타데이터 요약
- 항상 포함: request_id, timestamp
- 조건부 포함: trace_id, operation_id, idempotency_key, webhook_delivery_id, event_id
- 내부 전용: actor_id, tenant_id, stack trace, 내부 서비스 이름

### 보안 노출 제한 원칙 요약
- 3단계 정보 tier: 외부 응답(최소) / 운영 로그(진단) / 감사 로그(보안)
- 스택 트레이스, DB 오류, ACL 세부, credential — 외부 응답 절대 금지
- 보안 오류 메시지 의도적 일반화
- Not Found vs Forbidden 정책 일관 적용

### async/webhook/idempotency/AI 연계 오류 관점 요약
- async: 즉각 응답 오류 + operation 결과 오류 두 경로. 동일 error 구조 적용
- webhook: delivery 실패는 API 오류 아님 (delivery 상태 리소스). signature 검증 실패는 400
- idempotency: key_missing(400), key_mismatch(409), replay_unavailable(410)
- AI/RAG: indexing_not_ready(503, retryable), retrieval_unavailable(503, retryable), citation_not_found(404)

### API 오류 응답 / 운영 로그 / 감사 로그 관계 요약
- 목적이 다름: 클라이언트 분기 / 진단 / 보안 컴플라이언스
- request_id가 세 계층을 연결하는 핵심 식별자
- 감사 로그 대상: 인증 실패, 권한 거부, 보안 위반, 민감 리소스 접근, AI processing 시작

### 후속 Task 연결 포인트
- **Task 3-11**: 대표 API 시나리오에서 오류 경로 포함, 오류 응답 일관성 검증
- **Task 3-12**: Phase 3 통합 문서에 error model/observability 규약 반영
- **구현 Phase**: exception handler, request middleware, logging correlation, audit pipeline, operation 실패 처리

### 의도적으로 후속 Task로 남긴 미결정 사항
- 전체 error code catalog 완성 (공통 + 도메인 특화)
- Rate limit 정책 수치 결정
- Concurrency control 구체 구현 방식
- Not Found vs Forbidden 정책의 도메인별 세부 결정
- Observability stack 제품 선택
- SLO/SLA 수치 정의
- Retry-After 계산 방식 구체화
- 감사 로그 보존 기간 및 접근 정책
