# Task5_common_response_format.md
# 공통 요청/응답 포맷 표준화

---

## 1. 문서 목적

이 문서는 Mimir 플랫폼 API 전반에서 일관되게 사용할 **공통 요청/응답 포맷 표준**을 정의한다.

단순한 JSON 보기 규칙이 아니다. 이 문서는 다음을 확정하는 **플랫폼 API 계약의 표면 형식(surface contract) 기준**이다:
- 성공/목록/오류/비동기 응답의 일관된 구조
- metadata 위치와 내용 원칙
- 오류 응답의 구조화된 계약
- 운영 추적 메타데이터 자리 확보
- AI/외부 시스템 소비자 친화성

이 문서는 Task 3-6(pagination/filter/sort), Task 3-7(idempotency), Task 3-8(async/event/webhook), Task 3-10(오류 모델)의 기반 계약으로 사용된다.

---

## 2. 공통 요청/응답 포맷이 필요한 이유

### 2-1. 소비자 비용의 문제

Mimir API는 User UI, Admin UI, 외부 시스템, AI/RAG, 내부 서비스가 공유한다. 각 소비자는 응답을 파싱하는 코드를 갖는다.

응답 구조가 엔드포인트마다 다르면:
- 파서 코드가 엔드포인트마다 달라진다
- SDK/클라이언트 라이브러리 유지보수 비용이 급증한다
- AI tool이 안정적으로 파싱하기 어려워진다
- 오류 응답 처리 로직이 흩어진다
- 테스트 코드가 비대해진다

### 2-2. 오류 응답도 계약이다

오류 응답이 `"Internal Server Error"` 같은 문자열로만 전달되면 소비자가 기계적으로 대응할 수 없다. 성공 응답만큼 오류 응답도 일관된 구조가 필요하다.

### 2-3. 공통 포맷은 계약 비용을 낮춘다

공통 envelope 구조가 있으면:
- 새 엔드포인트를 추가해도 기존 파서가 상위 구조를 재사용한다
- pagination, tracing, deprecation warning을 한 곳에서 다룬다
- AI tool이 예측 가능한 키로 데이터를 추출한다

**공통 포맷은 보기 좋은 JSON이 아니라 계약 비용을 낮추는 핵심 표준이다.**

---

## 3. 상위 설계 원칙

### 원칙 1. Consistent Shape Across Endpoints

**의미**: 모든 성공 응답과 오류 응답은 동일한 상위 키 구조를 가진다.

**설계 영향**: 단건이든 목록이든 `data` 키에 리소스가 위치한다. `meta` 키에 운영 정보가 위치한다.

---

### 원칙 2. Predictable Parsing

**의미**: 소비자는 응답 타입에 상관없이 동일한 방식으로 파싱을 시작할 수 있다.

**설계 영향**: `data`가 객체이면 단건, 배열이면 목록이라는 일관된 규칙을 따른다. 오류는 항상 `error` 키에 위치한다.

---

### 원칙 3. Machine-readable First, Human-readable Also

**의미**: 기계가 분기 처리하기 쉬운 구조를 우선하되, 사람도 읽을 수 있어야 한다.

**설계 영향**: 오류 코드는 문자열 enum(기계용), 메시지는 자연어(사람용). 두 가지를 모두 포함.

---

### 원칙 4. Minimal but Extensible Envelope

**의미**: 초기부터 과도한 중첩을 피하되, pagination/tracing/warnings를 넣을 공간은 확보한다.

**설계 영향**: `data`와 `meta` 두 최상위 키로 시작. `meta` 하위에 필요에 따라 섹션을 추가.

---

### 원칙 5. Separation of Resource Data and Metadata

**의미**: 리소스 데이터(비즈니스 속성)와 운영 메타데이터(tracing, pagination 등)는 섞이지 않는다.

**설계 영향**: `data`에는 순수 리소스 속성만. `meta`에는 운영 정보. 리소스 필드에 `_pagination` 같은 것을 섞지 않는다.

---

### 원칙 6. Explicit Error Contracts

**의미**: 오류 응답은 단순 문자열이 아니라 구조화된 객체다.

**설계 영향**: `error.code`, `error.message`, `error.details` 구조를 모든 오류에 적용.

---

### 원칙 7. Traceable Responses

**의미**: 모든 응답(성공/오류)에 `request_id`가 포함되어 추적이 가능하다.

**설계 영향**: `meta.request_id`를 전역 필수 필드로 지정.

---

### 원칙 8. Version-Compatible Structure

**의미**: 공통 envelope 구조 자체는 major version이 바뀌지 않는 한 유지된다.

**설계 영향**: envelope 최상위 키(`data`, `meta`, `error`)를 breaking change 없이는 변경하지 않는다.

---

### 원칙 9. AI/Tool-Friendly Representation

**의미**: AI tool이 citation, lookup, follow-up action을 수행하기 쉬운 구조를 지향한다.

**설계 영향**: 리소스 `id`는 항상 `data.id` 또는 `data[n].id`에 위치. 식별자가 예측 가능한 위치에 있어야 한다.

---

## 4. 성공 응답 Envelope 방향 결정

### 4-1. 후보 비교

**후보 A: 리소스 직접 반환**
```json
{ "id": "doc_123", "title": "규정 문서" }
```

| 항목 | 평가 |
|---|---|
| 장점 | 단순. 초기 구현 빠름. |
| 단점 | pagination, tracing, warnings를 어디에 넣을지 불명확. 목록과 단건의 상위 구조가 달라짐. |
| 파서 일관성 | 낮음. 단건은 객체, 목록은 배열이 되면 파서가 타입 분기를 해야 함. |
| 확장성 | 낮음. 나중에 메타 추가 시 기존 응답 구조 파괴. |
| 외부/AI 친화성 | 낮음. 식별자 위치가 응답마다 다를 수 있음. |
| 이 플랫폼 적합성 | **부적합** |

---

**후보 B: `data` + `meta` envelope**
```json
{
  "data": { "id": "doc_123", "title": "규정 문서" },
  "meta": { "request_id": "req_abc", "timestamp": "2026-04-02T12:00:00Z" }
}
```

| 항목 | 평가 |
|---|---|
| 장점 | 상위 키 구조 일관. pagination/tracing/warnings를 meta에 자연스럽게 추가. 확장성 높음. |
| 단점 | 한 단계 중첩 추가. 초기엔 단순 케이스에 약간 verbose. |
| 파서 일관성 | 높음. `data`는 항상 리소스 데이터. `meta`는 항상 운영 정보. |
| 확장성 | 높음. meta 하위에 pagination/tracing 등 자유롭게 추가 가능. |
| 외부/AI 친화성 | 높음. `data.id`가 항상 동일 위치. `meta`와 `data` 분리 명확. |
| 이 플랫폼 적합성 | **적합** |

---

**후보 C: 상황별 혼용**

| 항목 | 평가 |
|---|---|
| 단점 | 파서가 응답 타입마다 다르게 동작해야 함. 일관성 파괴. |
| 이 플랫폼 적합성 | **부적합** |

### 4-2. 채택: 후보 B — `data` + `meta` envelope

**이유**: 이 플랫폼은 다중 소비자(UI/외부/AI), pagination, tracing, deprecation warning, idempotency replay 표시 등 다양한 운영 정보가 필요하다. 초기 단순성보다 장기 확장성과 파서 일관성이 더 중요하다.

**최상위 응답 구조 (성공)**:
```
{
  "data": <리소스 객체 또는 배열>,
  "meta": { ...운영 메타데이터... }
}
```

**최상위 응답 구조 (오류)**:
```
{
  "error": { ...오류 객체... },
  "meta": { request_id, timestamp, ... }
}
```

---

## 5. 단건 응답 규약

### 5-1. 단건 조회 (GET /resources/{id})

- `data`: 리소스 단일 객체
- `meta`: request_id, timestamp 최소 포함
- HTTP Status: `200 OK`

```json
{
  "data": {
    "id": "doc_123",
    "type": "regulation",
    "title": "개인정보 처리 방침",
    "status": "published",
    "created_at": "2026-01-15T09:00:00Z",
    "updated_at": "2026-03-20T14:30:00Z"
  },
  "meta": {
    "request_id": "req_a1b2c3",
    "timestamp": "2026-04-02T12:00:00Z"
  }
}
```

### 5-2. 생성 성공 (POST /resources)

- `data`: 생성된 리소스 전체 반환 (id 포함)
- HTTP Status: `201 Created`
- 응답 헤더에 `Location: /api/v1/resources/{id}` 포함

```json
{
  "data": {
    "id": "doc_456",
    "type": "policy",
    "title": "보안 정책 v2",
    "status": "draft",
    "created_at": "2026-04-02T12:00:00Z",
    "updated_at": "2026-04-02T12:00:00Z"
  },
  "meta": {
    "request_id": "req_d4e5f6",
    "timestamp": "2026-04-02T12:00:00Z"
  }
}
```

### 5-3. 수정 성공 (PATCH /resources/{id})

- `data`: 수정 후 리소스 전체 반환
- HTTP Status: `200 OK`

```json
{
  "data": {
    "id": "doc_456",
    "title": "보안 정책 v2 (수정)",
    "status": "draft",
    "updated_at": "2026-04-02T12:30:00Z"
  },
  "meta": {
    "request_id": "req_g7h8i9",
    "timestamp": "2026-04-02T12:30:00Z"
  }
}
```

### 5-4. 삭제 성공 (DELETE /resources/{id})

- `data`: `null` 또는 최소 확인 정보 (id만)
- HTTP Status: `204 No Content` (body 없음) 또는 `200 OK` (body 있음)
- **권장**: `204 No Content` — body 없이 status code로만 표현. 클라이언트가 삭제 여부를 추적해야 한다면 `200 OK` + 최소 body 사용.

```json
// 200 OK를 사용하는 경우
{
  "data": { "id": "doc_456", "deleted": true },
  "meta": {
    "request_id": "req_j1k2l3",
    "timestamp": "2026-04-02T12:45:00Z"
  }
}
```

### 5-5. Action 성공 (POST /resources/{id}:action)

- `data`: 행위 결과 리소스 (변경된 리소스 전체 또는 operation 객체)
- HTTP Status: `200 OK` (동기 완료) 또는 `202 Accepted` (비동기 처리 시작)

```json
// 동기 action 성공 (예: :publish)
{
  "data": {
    "id": "doc_123",
    "status": "published",
    "published_at": "2026-04-02T13:00:00Z"
  },
  "meta": {
    "request_id": "req_m4n5o6",
    "timestamp": "2026-04-02T13:00:00Z"
  }
}
```

### 5-6. 비동기 작업 시작 (202 Accepted)

- `data`: operation 객체 (id, status, 진행 추적 링크)
- HTTP Status: `202 Accepted`

```json
{
  "data": {
    "operation_id": "op_p7q8r9",
    "type": "reindex",
    "status": "pending",
    "resource_type": "document",
    "resource_id": "doc_123"
  },
  "meta": {
    "request_id": "req_s1t2u3",
    "timestamp": "2026-04-02T13:05:00Z",
    "links": {
      "operation": "/api/v1/operations/op_p7q8r9"
    }
  }
}
```

---

## 6. 목록 응답 규약

### 6-1. 목록 응답 구조

- `data`: 리소스 **배열** (단건과 동일한 리소스 shape)
- `meta.pagination`: 페이지네이션 정보 (세부는 Task 3-6에서 확정)
- HTTP Status: `200 OK`

```json
{
  "data": [
    { "id": "doc_001", "title": "문서 A", "status": "published" },
    { "id": "doc_002", "title": "문서 B", "status": "draft" }
  ],
  "meta": {
    "request_id": "req_v4w5x6",
    "timestamp": "2026-04-02T13:10:00Z",
    "pagination": {
      "total": 42,
      "page": 1,
      "page_size": 20,
      "has_next": true,
      "has_prev": false
    }
  }
}
```

### 6-2. 빈 목록 응답

- `data`: 빈 배열 `[]` (null 금지)
- `meta.pagination.total`: `0`
- HTTP Status: `200 OK` (404 반환 금지)

```json
{
  "data": [],
  "meta": {
    "request_id": "req_y7z8a9",
    "timestamp": "2026-04-02T13:15:00Z",
    "pagination": {
      "total": 0,
      "page": 1,
      "page_size": 20,
      "has_next": false,
      "has_prev": false
    }
  }
}
```

### 6-3. 목록 응답 원칙

| 항목 | 규칙 |
|---|---|
| items 위치 | `data` 배열. `data.items` 같은 추가 중첩 금지. |
| total count | `meta.pagination.total`에 포함. 세부 제공 범위는 Task 3-6에서 결정. |
| pagination 위치 | `meta.pagination` |
| 빈 목록 | `data: []` + `200 OK`. 절대 `null` 반환 금지. |
| authorization filtering | 필터링 후 결과만 반환. 총 count는 필터링된 항목 기준. |
| data 배열 shape | 목록의 각 항목은 단건 조회와 동일한 필드 구조 (단, 목록에서는 일부 필드 생략 가능) |

---

## 7. 메타데이터 구조 원칙

### 7-1. `meta` 구조

`meta`는 응답 유형에 따라 해당하는 섹션만 포함한다.

```
meta:
  request_id       (전역 필수)
  timestamp        (전역 필수)
  api_version      (권장)
  pagination       (목록 응답 시)
  links            (관련 리소스 링크, 선택)
  warnings         (deprecated 경고, 부분 실패 경고 등)
  idempotency      (idempotency replay 정보, Task 3-7에서 확정)
  trace_id         (분산 추적 필요 시)
```

### 7-2. 전역 공통 meta 필드

| 필드 | 타입 | 필수 여부 | 설명 |
|---|---|---|---|
| `request_id` | string | **필수** | 요청 추적 ID. 클라이언트 제공 또는 서버 생성. |
| `timestamp` | ISO 8601 | **필수** | 응답 생성 시각 |
| `api_version` | string | 권장 | 처리에 사용된 API 버전 (`v1`) |
| `trace_id` | string | 선택 | 분산 추적 ID |

### 7-3. 조건부 meta 필드

| 필드 | 나타나는 경우 |
|---|---|
| `meta.pagination` | 목록 응답 |
| `meta.links` | 관련 리소스 링크가 있는 경우 |
| `meta.warnings` | deprecated API 사용, partial result 등 |
| `meta.idempotency` | idempotency replay 여부 (Task 3-7에서 확정) |

### 7-4. 원칙

- **리소스 데이터(`data`)에는 운영 메타데이터 포함 금지.** `data.request_id` 같은 방식 금지.
- **`meta`는 비대해지지 않도록 제한한다.** 모든 응답에 모든 메타 필드를 채우지 않는다. 해당 응답 유형에 필요한 것만 포함.
- **`meta.warnings`**: 배열 형태. 각 warning은 `{ "code": "...", "message": "..." }` 구조.

```json
// warnings 예시 (deprecated API 사용)
"meta": {
  "request_id": "req_abc",
  "timestamp": "2026-04-02T12:00:00Z",
  "warnings": [
    {
      "code": "DEPRECATED_FIELD",
      "message": "The 'owner' field is deprecated. Use 'created_by' instead.",
      "sunset": "2027-01-01"
    }
  ]
}
```

---

## 8. 오류 응답 구조 원칙

### 8-1. 오류 응답 최상위 구조

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "요청 데이터가 유효하지 않습니다.",
    "type": "validation",
    "details": [...],
    "retryable": false
  },
  "meta": {
    "request_id": "req_err123",
    "timestamp": "2026-04-02T12:00:00Z"
  }
}
```

### 8-2. `error` 객체 필드

| 필드 | 타입 | 필수 여부 | 설명 |
|---|---|---|---|
| `code` | string (UPPER_SNAKE) | **필수** | 기계 판별용 오류 코드. 전체 카탈로그는 Task 3-10에서 확정. |
| `message` | string | **필수** | 사람이 읽는 오류 설명 (UI 표시 가능) |
| `type` | string | **필수** | 오류 카테고리 (`validation`, `auth`, `authorization`, `not_found`, `conflict`, `server`, `rate_limit` 등) |
| `details` | array | 조건부 | 복수 오류 세부사항 (validation 오류 등에서 사용) |
| `target` | string | 선택 | 오류 발생 대상 (필드명, 파라미터명 등) |
| `retryable` | boolean | 권장 | 재시도 가능 여부 |
| `docs_url` | string | 선택 | 관련 문서 링크 |

### 8-3. `error.details` 구조

복수 오류(validation 등)에서 사용:
```json
"details": [
  {
    "field": "title",
    "code": "REQUIRED_FIELD",
    "message": "제목은 필수입니다."
  },
  {
    "field": "document_type_id",
    "code": "INVALID_REFERENCE",
    "message": "존재하지 않는 문서 타입입니다."
  }
]
```

### 8-4. 오류 응답 원칙

- 오류 응답은 문자열 하나가 아니라 구조화된 객체다.
- `error.code`는 기계 처리용 (switch/if 분기), `error.message`는 사람용 (UI 표시 또는 로그).
- 내부 스택 트레이스, SQL 쿼리, 서버 경로는 외부 응답에 포함하지 않는다.
- `meta.request_id`는 오류 응답에도 항상 포함한다 — 지원팀이 로그를 추적할 수 있다.
- 동일 계열 오류는 어느 엔드포인트에서든 동일한 `type` + `code` 구조를 유지한다.

---

## 9. Validation / Auth / Conflict / Not Found / Async 응답 구분

### 9-1. Validation Error

| 항목 | 값 |
|---|---|
| HTTP Status | `400 Bad Request` |
| `error.type` | `validation` |
| `error.code` | `VALIDATION_ERROR` |
| `error.details` | 필드별 오류 목록 필수 |
| `retryable` | `false` |

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "요청 데이터 유효성 검사에 실패했습니다.",
    "type": "validation",
    "details": [
      { "field": "title", "code": "REQUIRED_FIELD", "message": "제목은 필수입니다." },
      { "field": "expires_at", "code": "INVALID_FORMAT", "message": "날짜 형식이 올바르지 않습니다." }
    ],
    "retryable": false
  },
  "meta": { "request_id": "req_val001", "timestamp": "2026-04-02T12:00:00Z" }
}
```

### 9-2. Authentication Failure

| 항목 | 값 |
|---|---|
| HTTP Status | `401 Unauthorized` |
| `error.type` | `auth` |
| `error.code` | `AUTHENTICATION_REQUIRED` 또는 `TOKEN_EXPIRED` |
| 주의 | 내부 토큰 구조 정보 노출 금지 |

```json
{
  "error": {
    "code": "AUTHENTICATION_REQUIRED",
    "message": "인증이 필요합니다.",
    "type": "auth",
    "retryable": false
  },
  "meta": { "request_id": "req_auth001", "timestamp": "2026-04-02T12:00:00Z" }
}
```

### 9-3. Authorization Failure (Permission Denied)

| 항목 | 값 |
|---|---|
| HTTP Status | `403 Forbidden` |
| `error.type` | `authorization` |
| `error.code` | `PERMISSION_DENIED` |
| 주의 | 리소스 존재 여부를 숨겨야 하는 경우 `404 Not Found` 반환 가능 |

```json
{
  "error": {
    "code": "PERMISSION_DENIED",
    "message": "이 리소스에 대한 접근 권한이 없습니다.",
    "type": "authorization",
    "retryable": false
  },
  "meta": { "request_id": "req_authz001", "timestamp": "2026-04-02T12:00:00Z" }
}
```

### 9-4. Not Found

| 항목 | 값 |
|---|---|
| HTTP Status | `404 Not Found` |
| `error.type` | `not_found` |
| `error.code` | `RESOURCE_NOT_FOUND` |
| 주의 | 다른 테넌트 리소스는 존재를 드러내지 않기 위해 `404` 반환 |

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "요청한 문서를 찾을 수 없습니다.",
    "type": "not_found",
    "target": "document",
    "retryable": false
  },
  "meta": { "request_id": "req_nf001", "timestamp": "2026-04-02T12:00:00Z" }
}
```

### 9-5. Conflict

| 항목 | 값 |
|---|---|
| HTTP Status | `409 Conflict` |
| `error.type` | `conflict` |
| `error.code` | `RESOURCE_CONFLICT` 또는 더 구체적 코드 |
| 사용 케이스 | 이미 존재하는 리소스 생성, 동시 편집 충돌, 상태 전이 불가 등 |

```json
{
  "error": {
    "code": "DOCUMENT_ALREADY_PUBLISHED",
    "message": "이미 게시된 문서는 다시 게시할 수 없습니다.",
    "type": "conflict",
    "retryable": false
  },
  "meta": { "request_id": "req_conf001", "timestamp": "2026-04-02T12:00:00Z" }
}
```

### 9-6. Async Accepted

| 항목 | 값 |
|---|---|
| HTTP Status | `202 Accepted` |
| `data` | operation 객체 (id, status, type) |
| 의미 | 요청은 수락되었으나 처리가 완료되지 않음 |

*(섹션 5-6 예시 참조)*

### 9-7. Rate Limit (검토 가능)

| 항목 | 값 |
|---|---|
| HTTP Status | `429 Too Many Requests` |
| `error.type` | `rate_limit` |
| `error.code` | `RATE_LIMIT_EXCEEDED` |
| `retryable` | `true` |
| 응답 헤더 | `Retry-After: <seconds>` |

구체 rate limit 정책은 구현 Phase에서 확정.

---

## 10. 운영 추적 메타데이터 규칙

### 10-1. request_id 처리

- 모든 응답에 `meta.request_id` **필수 포함**.
- 클라이언트가 요청 헤더 `X-Request-ID`로 제공하면 그 값을 그대로 반영.
- 클라이언트가 제공하지 않으면 서버가 UUID로 생성.
- 오류 발생 시 `meta.request_id`로 서버 로그 추적 가능.

### 10-2. trace_id 처리

- 분산 추적이 필요한 환경에서 `meta.trace_id` 포함.
- 초기 단계에서는 선택 필드.
- 향후 분산 환경 구성 시 필수화 고려.

### 10-3. timestamp

- 모든 응답에 서버 응답 생성 시각 `meta.timestamp` 포함.
- ISO 8601 UTC 형식: `"2026-04-02T12:00:00Z"`

### 10-4. 민감 정보 노출 제한

- 서버 내부 경로, 스택 트레이스, DB 쿼리, 내부 서비스명은 외부 응답에 포함 금지.
- 디버깅 정보는 서버 로그에만 기록하고 `request_id`로 연결.

### 10-5. deprecation info

- deprecated API 사용 시 `meta.warnings` 배열에 경고 포함.
- 응답 헤더에도 `Deprecation: true`, `Sunset: <date>` 포함.

---

## 11. AI / 외부 시스템 친화성 관점

### 11-1. AI tool이 필요로 하는 응답 구조 특성

| 요구 | 이 포맷의 대응 |
|---|---|
| 예측 가능한 상위 키 | `data`, `meta`, `error` 세 가지만 |
| 리소스 id 위치 일관성 | 항상 `data.id` 또는 `data[n].id` |
| 오류 기계 분기 가능 | `error.code` (UPPER_SNAKE enum) |
| metadata와 resource payload 분리 | `data`와 `meta` 명확 분리 |
| 목록과 단건 일관성 | `data`가 객체(단건) vs 배열(목록)로만 구분 |
| pagination 정보 위치 | `meta.pagination` — 항상 동일 위치 |

### 11-2. AI 소비자를 위한 추가 권장 사항

- 리소스에 `type` 필드를 포함하여 응답 타입 식별 가능하게 한다.
- `meta.links`에 관련 리소스 링크를 제공하여 follow-up 접근 가능하게 한다.
- 오류 `code`는 AI가 분기 처리 가능한 일관된 enum 값을 사용한다.

---

## 12. 요청 포맷 최소 기준

*(응답 중심 Task이므로 최소 기준만 정리)*

### 12-1. 요청 body 원칙

- 모든 요청 body는 **JSON** (`Content-Type: application/json`)
- 최상위 wrapper 없이 리소스 속성을 직접 포함하는 flat 구조 기본

```json
// 권장 (flat)
POST /api/v1/documents
{ "title": "문서 제목", "document_type_id": "dt_001" }

// 비권장 (불필요한 wrapper)
{ "document": { "title": "문서 제목" } }
```

### 12-2. 부분 업데이트 (PATCH)

- PATCH는 변경할 필드만 포함 (전체 교체 금지)
- 명시적으로 `null`을 보내면 해당 필드를 null로 설정하는 것으로 처리

### 12-3. Bulk / Action / Search 요청

- Bulk: 최상위에 `items` 배열 포함
- Action: 필요한 파라미터만 flat 구조로 전달 (wrapper 불필요)
- Search: 검색 조건을 body에 구조적으로 포함 (세부는 Task 3-9에서 확정)

---

## 13. 예시 응답 모음

### 성공 — 단건 조회

```json
// GET /api/v1/documents/doc_123
{
  "data": {
    "id": "doc_123",
    "type": "regulation",
    "title": "개인정보 처리 방침 2026",
    "status": "published",
    "version_count": 3,
    "created_at": "2026-01-10T09:00:00Z",
    "updated_at": "2026-03-15T14:00:00Z"
  },
  "meta": {
    "request_id": "req_a1b2c3d4",
    "timestamp": "2026-04-02T12:00:00Z",
    "api_version": "v1"
  }
}
```

### 성공 — 생성

```json
// POST /api/v1/documents → 201 Created
{
  "data": {
    "id": "doc_789",
    "type": "policy",
    "title": "신규 보안 정책",
    "status": "draft",
    "created_at": "2026-04-02T13:00:00Z",
    "updated_at": "2026-04-02T13:00:00Z"
  },
  "meta": {
    "request_id": "req_e5f6g7h8",
    "timestamp": "2026-04-02T13:00:00Z",
    "api_version": "v1",
    "links": {
      "self": "/api/v1/documents/doc_789"
    }
  }
}
```

### 성공 — 목록 조회

```json
// GET /api/v1/documents?status=published&page=1&page_size=2
{
  "data": [
    {
      "id": "doc_001",
      "type": "regulation",
      "title": "개인정보 방침",
      "status": "published",
      "updated_at": "2026-03-15T14:00:00Z"
    },
    {
      "id": "doc_002",
      "type": "policy",
      "title": "보안 정책",
      "status": "published",
      "updated_at": "2026-02-20T11:00:00Z"
    }
  ],
  "meta": {
    "request_id": "req_i9j0k1l2",
    "timestamp": "2026-04-02T12:00:00Z",
    "api_version": "v1",
    "pagination": {
      "total": 28,
      "page": 1,
      "page_size": 2,
      "has_next": true,
      "has_prev": false
    }
  }
}
```

### 성공 — 빈 목록

```json
// GET /api/v1/documents?status=archived
{
  "data": [],
  "meta": {
    "request_id": "req_m3n4o5p6",
    "timestamp": "2026-04-02T12:00:00Z",
    "pagination": {
      "total": 0,
      "page": 1,
      "page_size": 20,
      "has_next": false,
      "has_prev": false
    }
  }
}
```

### 성공 — Action (동기)

```json
// POST /api/v1/documents/doc_123:publish → 200 OK
{
  "data": {
    "id": "doc_123",
    "status": "published",
    "published_at": "2026-04-02T13:00:00Z",
    "updated_at": "2026-04-02T13:00:00Z"
  },
  "meta": {
    "request_id": "req_q7r8s9t0",
    "timestamp": "2026-04-02T13:00:00Z"
  }
}
```

### 성공 — Async Accepted

```json
// POST /api/v1/documents/doc_123:reindex → 202 Accepted
{
  "data": {
    "operation_id": "op_u1v2w3x4",
    "type": "reindex",
    "status": "pending",
    "resource_type": "document",
    "resource_id": "doc_123",
    "created_at": "2026-04-02T13:05:00Z"
  },
  "meta": {
    "request_id": "req_y5z6a7b8",
    "timestamp": "2026-04-02T13:05:00Z",
    "links": {
      "operation": "/api/v1/operations/op_u1v2w3x4"
    }
  }
}
```

### 오류 — Validation Error

```json
// POST /api/v1/documents → 400 Bad Request
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "요청 데이터가 유효하지 않습니다.",
    "type": "validation",
    "details": [
      {
        "field": "title",
        "code": "REQUIRED_FIELD",
        "message": "제목은 필수입니다."
      },
      {
        "field": "document_type_id",
        "code": "INVALID_REFERENCE",
        "message": "존재하지 않는 문서 타입 ID입니다."
      }
    ],
    "retryable": false
  },
  "meta": {
    "request_id": "req_c9d0e1f2",
    "timestamp": "2026-04-02T12:00:00Z"
  }
}
```

### 오류 — Auth Failure

```json
// 401 Unauthorized
{
  "error": {
    "code": "AUTHENTICATION_REQUIRED",
    "message": "인증이 필요합니다. 유효한 인증 토큰을 제공하세요.",
    "type": "auth",
    "retryable": false
  },
  "meta": {
    "request_id": "req_g3h4i5j6",
    "timestamp": "2026-04-02T12:00:00Z"
  }
}
```

### 오류 — Permission Denied

```json
// 403 Forbidden
{
  "error": {
    "code": "PERMISSION_DENIED",
    "message": "이 문서에 대한 편집 권한이 없습니다.",
    "type": "authorization",
    "retryable": false
  },
  "meta": {
    "request_id": "req_k7l8m9n0",
    "timestamp": "2026-04-02T12:00:00Z"
  }
}
```

### 오류 — Not Found

```json
// 404 Not Found
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "요청한 문서를 찾을 수 없습니다.",
    "type": "not_found",
    "target": "document",
    "retryable": false
  },
  "meta": {
    "request_id": "req_o1p2q3r4",
    "timestamp": "2026-04-02T12:00:00Z"
  }
}
```

### 오류 — Conflict

```json
// 409 Conflict
{
  "error": {
    "code": "DOCUMENT_ALREADY_PUBLISHED",
    "message": "이미 게시된 문서는 다시 게시할 수 없습니다. 새 버전을 만들어 게시하세요.",
    "type": "conflict",
    "retryable": false,
    "docs_url": "/docs/api/documents/publish"
  },
  "meta": {
    "request_id": "req_s5t6u7v8",
    "timestamp": "2026-04-02T12:00:00Z"
  }
}
```

---

## 14. 후속 Task에 전달할 설계 기준

| Task | 전달 기준 |
|---|---|
| Task 3-6 (pagination/filter/sort) | pagination 정보는 `meta.pagination`에 위치. 목록 응답 구조(`data` 배열 + `meta.pagination`)를 기반으로 cursor/offset 세부 문법 결정. |
| Task 3-7 (idempotency) | idempotency replay 여부는 `meta.idempotency` 영역에 표시. 응답 구조는 동일하게 유지. |
| Task 3-8 (async/event/webhook) | async job 응답은 `202 + data(operation 객체)` 패턴. event/webhook 응답도 공통 envelope 활용 원칙. |
| Task 3-9 (AI/RAG) | search/retrieval 결과도 `data` 배열 + `meta.pagination` 구조 활용. AI citation용 식별자는 `data[n].id`에 일관 위치. |
| Task 3-10 (오류 모델) | `error.code` enum 카탈로그 확정. `error.type` 분류 체계 확정. observability metadata 연결. |
| 구현 Phase | serializer(response helper), exception handler, response middleware에 이 구조를 직접 반영. `request_id` 자동 생성/주입 미들웨어 구현. |

---

## 15. 결론

Mimir 플랫폼 API의 공통 응답 포맷은 다음 두 가지 최상위 구조로 요약된다:

**성공 응답**:
```
{ "data": <리소스>, "meta": { request_id, timestamp, [pagination], [warnings], ... } }
```

**오류 응답**:
```
{ "error": { code, message, type, details, retryable }, "meta": { request_id, timestamp } }
```

이 구조는 모든 소비자(UI, 외부, AI, 내부)가 예측 가능하게 파싱할 수 있으며, pagination/tracing/deprecation/idempotency 등 후속 요구를 `meta` 하위에 자연스럽게 수용한다.

---

## 자체 점검 요약

### 권장 응답 Envelope 구조

`data` + `meta` envelope (후보 B). 오류 시 `error` + `meta`. 최상위 키는 세 가지만(`data`, `meta`, `error`).

### 단건 / 목록 / 오류 / 비동기 핵심 Shape 요약

| 유형 | HTTP Status | 최상위 구조 |
|---|---|---|
| 단건 조회 | 200 | `{ data: {}, meta: {} }` |
| 생성 성공 | 201 | `{ data: {}, meta: {} }` |
| 수정 성공 | 200 | `{ data: {}, meta: {} }` |
| 삭제 성공 | 204 (body 없음) 또는 200 | `{ data: {deleted:true}, meta: {} }` |
| 목록 조회 | 200 | `{ data: [], meta: {pagination: {}} }` |
| 빈 목록 | 200 | `{ data: [], meta: {pagination: {total:0}} }` |
| 비동기 시작 | 202 | `{ data: {operation_id:...}, meta: {links:{}} }` |
| 오류 | 4xx/5xx | `{ error: {code, message, type}, meta: {} }` |

### 공통 Metadata 핵심 필드 요약

전역 필수: `request_id`, `timestamp`. 조건부: `api_version`, `trace_id`, `pagination`, `links`, `warnings`, `idempotency`.

### AI / 외부 시스템 친화성 요약

- 상위 키 3개 고정(`data`, `meta`, `error`). 리소스 id는 항상 `data.id` / `data[n].id`. 오류 분기는 `error.code` enum. metadata와 resource 분리.

### 후속 Task 연결 포인트

3-6(pagination → `meta.pagination`), 3-7(idempotency → `meta.idempotency`), 3-8(async → 202 + operation 객체), 3-9(AI retrieval → `data` 배열 + `meta.pagination`), 3-10(error.code 카탈로그)

### 의도적으로 후속 Task로 남긴 미결정 사항

- pagination cursor/offset 세부 문법 → Task 3-6
- total count 항상 제공 여부 → Task 3-6
- idempotency meta 정확한 구조 → Task 3-7
- event/webhook payload envelope 적용 범위 → Task 3-8
- error.code 전체 카탈로그 → Task 3-10
- `meta.links` 구조 상세 → 구현 Phase
- search/retrieval 응답 전용 필드 → Task 3-9
