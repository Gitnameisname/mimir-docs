# API 공통 규약

---

## 1. 기본 정보

| 항목 | 값 |
|------|-----|
| Base URL | `https://{host}/api/v1` |
| 프로토콜 | HTTPS (운영), HTTP (개발) |
| 인코딩 | UTF-8 |
| Content-Type | `application/json` |
| API 버전 | v1 |

개발 환경: `http://localhost:8000/api/v1`  
OpenAPI 문서: `http://localhost:8000/docs` (개발 환경만)

---

## 2. 응답 포맷 — 공통 Envelope

모든 API 응답은 공통 Envelope 구조로 감싸진다.

### 단일 리소스 응답

```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "title": "정보보안 정책 v3",
    "document_type": "POLICY",
    "status": "published",
    ...
  }
}
```

### 목록 응답 (페이지네이션)

```json
{
  "status": "success",
  "data": [...],
  "meta": {
    "total": 142,
    "page": 1,
    "page_size": 20,
    "total_pages": 8
  }
}
```

### 오류 응답

```json
{
  "status": "error",
  "error": {
    "code": "NOT_FOUND",
    "message": "Document not found",
    "details": null
  }
}
```

---

## 3. HTTP 상태 코드

| 코드 | 의미 | 발생 상황 |
|------|------|-----------|
| `200 OK` | 성공 | GET, PUT, PATCH, DELETE 성공 |
| `201 Created` | 생성됨 | POST 성공 |
| `400 Bad Request` | 요청 오류 | 입력 유효성 실패, 비즈니스 규칙 위반 |
| `401 Unauthorized` | 인증 실패 | 토큰 없음/만료/유효하지 않음 |
| `403 Forbidden` | 권한 없음 | 역할 미충족, 리소스 접근 거부 |
| `404 Not Found` | 없음 | 리소스 존재하지 않음 |
| `409 Conflict` | 충돌 | 중복 요청, 상태 전이 불가 |
| `413 Payload Too Large` | 요청 크기 초과 | 10MB 초과 |
| `422 Unprocessable Entity` | 유효성 오류 | Pydantic 스키마 검증 실패 |
| `429 Too Many Requests` | Rate Limit 초과 | 엔드포인트별 제한 초과 |
| `500 Internal Server Error` | 서버 오류 | 예상치 못한 서버 에러 |

---

## 4. 오류 코드 목록

| code | HTTP | 설명 |
|------|------|------|
| `NOT_FOUND` | 404 | 요청한 리소스를 찾을 수 없음 |
| `VALIDATION_ERROR` | 400 | 입력 유효성 검사 실패 |
| `AUTHENTICATION_REQUIRED` | 401 | 인증이 필요한 엔드포인트 |
| `PERMISSION_DENIED` | 403 | 해당 작업 권한 없음 |
| `CONFLICT` | 409 | 상태 충돌 또는 중복 |
| `RATE_LIMIT_EXCEEDED` | 429 | 요청 횟수 제한 초과 |
| `INTERNAL_SERVER_ERROR` | 500 | 서버 내부 오류 |

---

## 5. 페이지네이션

목록 API는 커서 기반이 아닌 **오프셋 기반** 페이지네이션을 사용한다.

### 요청 쿼리 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `page` | integer | 1 | 페이지 번호 (1부터) |
| `page_size` | integer | 20 | 페이지당 항목 수 (최대 100) |
| `sort_by` | string | `created_at` | 정렬 기준 필드 |
| `sort_order` | `asc` \| `desc` | `desc` | 정렬 방향 |

### 예시

```
GET /api/v1/documents?page=2&page_size=10&sort_by=updated_at&sort_order=desc
```

### 응답 meta

```json
"meta": {
  "total": 142,
  "page": 2,
  "page_size": 10,
  "total_pages": 15
}
```

---

## 6. 필터링

목록 API는 `filter[필드]=값` 형식의 필터 파라미터를 지원한다.

```
GET /api/v1/documents?filter[status]=published&filter[document_type]=POLICY
```

허용 필터 필드는 엔드포인트별로 상이하며, 허용되지 않은 필드로 필터링 시 400 오류를 반환한다.

---

## 7. 멱등성 (Idempotency)

`POST` 쓰기 요청에 `Idempotency-Key` 헤더를 포함하면 중복 요청을 방지할 수 있다.

```
POST /api/v1/documents
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440001
Content-Type: application/json

{...}
```

- 동일 키로 재요청 시: 최초 응답을 그대로 반환 (DB 재처리 없음)
- 유효 기간: 24시간
- 키 형식: UUID 권장

---

## 8. 요청 추적

모든 요청은 추적 ID를 지원한다.

| 헤더 | 방향 | 설명 |
|------|------|------|
| `X-Request-ID` | 요청 → 응답 | 요청 고유 ID. 미포함 시 서버가 자동 생성. 응답에 포함됨. |
| `X-Trace-ID` | 요청 | 분산 추적 ID (외부 추적 시스템 연동용) |

```
요청: X-Request-ID: my-req-001
응답: X-Request-ID: my-req-001  (echo-back)
```

- `X-Request-ID`는 최대 128자로 잘라냄 (보안 정책)

---

## 9. Rate Limiting

엔드포인트별로 Rate Limit이 적용된다.

| 엔드포인트 유형 | 제한 |
|----------------|------|
| 일반 읽기 (`GET`) | 200 req/min |
| 쓰기 (`POST/PUT/DELETE`) | 60 req/min |
| 검색 | 100 req/min |
| RAG 질의 | 20 req/min |
| Admin | 100 req/min |

초과 시 `429 Too Many Requests` 응답.  
`Retry-After` 헤더에 재시도 가능 시각 포함.

---

## 10. CORS

허용 오리진은 `CORS_ALLOW_ORIGINS` 환경 변수로 설정한다.

```
CORS_ALLOW_ORIGINS=https://app.example.com,https://admin.example.com
```

허용 메서드: `GET, POST, PUT, PATCH, DELETE, OPTIONS`  
허용 헤더: `Authorization, Content-Type, X-Request-Id, X-Trace-Id, X-API-Key, ...`
