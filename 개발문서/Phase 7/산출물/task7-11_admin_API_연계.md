# Task 7-11 산출물: Admin UI API 연계 매핑

---

## 1. 공통 API 규약

### 1-1. 인증/인가

- 모든 Admin API: `Authorization: Bearer {jwt_token}` 헤더 필수
- JWT payload에서 `role` 필드 검증: `SUPER_ADMIN` 또는 `ORG_ADMIN`
- 인가 실패 시: `403 PERMISSION_DENIED`

### 1-2. Base URL

```
/api/v1/admin/...
```

### 1-3. 페이지네이션 공통 규약

**Request:**
```
GET /api/v1/admin/users?page=1&limit=20&sort=created_at&order=desc
```

**Response:**
```json
{
  "items": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 350,
    "total_pages": 18
  }
}
```

### 1-4. 필터 파라미터 공통 규약

```
GET /api/v1/admin/users?status=ACTIVE&search=john&role=EDITOR
```

### 1-5. 오류 응답 형식

```json
{
  "error": {
    "code": "PERMISSION_DENIED",
    "message": "접근 권한이 없습니다.",
    "details": {}
  }
}
```

---

## 2. 화면별 API 매핑 표

### 2-1. 사용자/조직/역할 관리

| 화면 액션 | Method | Endpoint | 권한 | 비고 |
|----------|--------|----------|------|------|
| 사용자 목록 조회 | GET | `/admin/users` | ADMIN | `?status&search&org_id&role&sort&order&page&limit` |
| 사용자 상세 조회 | GET | `/admin/users/{user_id}` | ADMIN | |
| 사용자 상태 변경 | PATCH | `/admin/users/{user_id}/status` | SUPER_ADMIN | body: `{status, reason?}` |
| 사용자 역할 변경 | PATCH | `/admin/users/{user_id}/roles` | SUPER_ADMIN | body: `{role_ids, reason?}` |
| 조직 목록 조회 | GET | `/admin/organizations` | ADMIN | `?status&search&page&limit` |
| 조직 상세 조회 | GET | `/admin/organizations/{org_id}` | ADMIN | |
| 조직 사용자 목록 | GET | `/admin/organizations/{org_id}/users` | ADMIN | |
| 역할 목록 조회 | GET | `/admin/roles` | ADMIN | |
| 역할 상세 조회 | GET | `/admin/roles/{role_id}` | ADMIN | |
| 역할별 권한 조회 | GET | `/admin/roles/{role_id}/permissions` | SUPER_ADMIN | |

### 2-2. 권한 정책 관리

| 화면 액션 | Method | Endpoint | 권한 | 비고 |
|----------|--------|----------|------|------|
| 정책 목록 조회 | GET | `/admin/permissions` | SUPER_ADMIN | `?resource_type&status&role_id&search` |
| 정책 상세 조회 | GET | `/admin/permissions/{policy_id}` | SUPER_ADMIN | |
| 정책 수정 | PUT | `/admin/permissions/{policy_id}` | SUPER_ADMIN | body: `{description?, actions?, conditions?, reason}` |
| 정책 상태 변경 | PATCH | `/admin/permissions/{policy_id}/status` | SUPER_ADMIN | body: `{status, reason}` |
| 역할-정책 매핑 조회 | GET | `/admin/permissions/role-matrix` | SUPER_ADMIN | 역할×정책 매트릭스 |

### 2-3. DocumentType 관리

| 화면 액션 | Method | Endpoint | 권한 | 비고 |
|----------|--------|----------|------|------|
| 타입 목록 조회 | GET | `/admin/document-types` | ADMIN | `?status&search` |
| 타입 상세 조회 | GET | `/admin/document-types/{type_code}` | ADMIN | 스키마 + 플러그인 설정 포함 |
| 타입 상태 변경 | PATCH | `/admin/document-types/{type_code}/status` | SUPER_ADMIN | body: `{status, reason?}` |

### 2-4. 감사 로그

| 화면 액션 | Method | Endpoint | 권한 | 비고 |
|----------|--------|----------|------|------|
| 감사 로그 목록 조회 | GET | `/admin/audit-logs` | ADMIN | 필터 파라미터 별도 정의 |
| 감사 로그 상세 조회 | GET | `/admin/audit-logs/{event_id}` | ADMIN | |

**감사 로그 필터 파라미터:**

| Parameter | 타입 | 설명 |
|-----------|------|------|
| `from` | datetime | 시작 일시 (ISO 8601) |
| `to` | datetime | 종료 일시 |
| `actor_id` | uuid | 행위자 ID |
| `resource_type` | string | 리소스 유형 |
| `event_type` | string (다중) | 이벤트 타입 (콤마 구분) |
| `severity` | string | `CRITICAL`, `HIGH`, `NORMAL` |
| `result` | string | `SUCCESS`, `FAILURE` |

### 2-5. 시스템 대시보드

| 화면 섹션 | Method | Endpoint | 권한 | 비고 |
|----------|--------|----------|------|------|
| 핵심 지표 카드 | GET | `/admin/dashboard/metrics` | ADMIN | |
| 컴포넌트 상태 | GET | `/admin/dashboard/health` | ADMIN | |
| 최근 오류 요약 | GET | `/admin/dashboard/errors` | ADMIN | `?limit=10` |
| 최근 감사 이벤트 | GET | `/admin/dashboard/recent-audit-logs` | ADMIN | `?limit=10` |

### 2-6. 인덱싱 상태 관리

| 화면 액션 | Method | Endpoint | 권한 | 비고 |
|----------|--------|----------|------|------|
| 인덱싱 목록 조회 | GET | `/admin/indexing/jobs` | ADMIN | `?status&doc_type&from&to&search&page&limit` |
| 인덱싱 작업 상세 | GET | `/admin/indexing/jobs/{job_id}` | ADMIN | |
| 인덱싱 재시도 | POST | `/admin/indexing/jobs/{job_id}/retry` | ADMIN | |
| 인덱싱 일괄 재시도 | POST | `/admin/indexing/jobs/retry-failed` | SUPER_ADMIN | body: `{filter?: {...}}` |
| 인덱싱 현황 요약 | GET | `/admin/indexing/summary` | ADMIN | 상태별 건수 |

### 2-7. API 키 관리

| 화면 액션 | Method | Endpoint | 권한 | 비고 |
|----------|--------|----------|------|------|
| API 키 목록 조회 | GET | `/admin/api-keys` | ADMIN | `?status&search&page&limit` |
| API 키 상세 조회 | GET | `/admin/api-keys/{key_id}` | ADMIN | |
| API 키 발급 | POST | `/admin/api-keys` | SUPER_ADMIN | body: `{name, description, scope, expires_at?}` |
| API 키 폐기 | DELETE | `/admin/api-keys/{key_id}` | SUPER_ADMIN | body: `{reason}` |
| 외부 연계 목록 조회 | GET | `/admin/integrations` | ADMIN | |
| 외부 연계 상세 조회 | GET | `/admin/integrations/{integration_id}` | ADMIN | |
| 외부 연계 상태 전환 | PATCH | `/admin/integrations/{integration_id}/status` | SUPER_ADMIN | |

### 2-8. 백그라운드 작업 관리

| 화면 액션 | Method | Endpoint | 권한 | 비고 |
|----------|--------|----------|------|------|
| 작업 목록 조회 | GET | `/admin/jobs` | ADMIN | `?status&job_type&from&to&requester&page&limit` |
| 작업 상세 조회 | GET | `/admin/jobs/{job_id}` | ADMIN | |
| 작업 취소 | POST | `/admin/jobs/{job_id}/cancel` | ADMIN | body: `{reason?}` |
| 작업 강제 종료 | POST | `/admin/jobs/{job_id}/force-stop` | SUPER_ADMIN | body: `{reason}` |
| 작업 재시도 | POST | `/admin/jobs/{job_id}/retry` | ADMIN | |
| 작업 현황 요약 | GET | `/admin/jobs/summary` | ADMIN | 상태별 건수 |

---

## 3. 권한 체크 포인트 정리서

### 3-1. 조회 액션 (ADMIN+)

- 사용자/조직/역할 목록 및 상세 조회
- DocumentType 목록 및 상세 조회 (SUPER_ADMIN 전용)
- 감사 로그 조회
- 작업/인덱싱 상태 조회
- API 키 목록 조회 (SUPER_ADMIN은 전체, ORG_ADMIN은 자신 발급분)
- 대시보드 지표 조회

### 3-2. 변경 액션 (SUPER_ADMIN 전용)

- 사용자 상태 변경 및 역할 변경
- 권한 정책 편집 및 상태 변경
- DocumentType 상태 변경
- API 키 발급 및 폐기
- 인덱싱 일괄 재시도
- 작업 강제 종료

### 3-3. ORG_ADMIN 제한 사항

- 권한 정책 화면 접근 불가
- DocumentType 관리 화면 접근 불가
- 역할 매핑 변경 불가
- 자신의 조직 범위 외 데이터 접근 불가 (서버사이드 필터 적용)

---

## 4. 오류 응답 처리 정책

| HTTP 코드 | Error Code | 프론트엔드 처리 |
|-----------|-----------|---------------|
| 400 | `VALIDATION_ERROR` | 폼 필드별 오류 메시지 인라인 표시 |
| 401 | `UNAUTHORIZED` | `/login?redirect={current_path}` 리다이렉트 |
| 403 | `PERMISSION_DENIED` | "접근 권한이 없습니다" Toast + 이전 페이지 복귀 |
| 404 | `NOT_FOUND` | "항목을 찾을 수 없습니다" EmptyState 표시 |
| 409 | `CONFLICT` | 충돌 내용 상세 안내 (예: "이미 존재하는 키 이름") |
| 422 | `UNPROCESSABLE` | 처리 불가 사유 상세 안내 |
| 429 | `RATE_LIMITED` | "요청이 너무 많습니다. 잠시 후 다시 시도하세요" |
| 500 | `INTERNAL_ERROR` | "오류가 발생했습니다. 잠시 후 다시 시도하세요" Toast + 재시도 버튼 |

---

## 5. Admin API Response 공통 구조

### 5-1. 단일 항목 응답

```json
{
  "data": { ... },
  "meta": {
    "requested_at": "2026-04-08T10:00:00Z"
  }
}
```

### 5-2. 목록 응답

```json
{
  "items": [ ... ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 350,
    "total_pages": 18
  }
}
```

### 5-3. 액션 결과 응답

```json
{
  "success": true,
  "message": "사용자 상태가 변경되었습니다.",
  "audit_event_id": "uuid"
}
```
