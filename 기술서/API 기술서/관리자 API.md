# 관리자 API

Base URL: `/api/v1/admin`  
필요 권한: 최소 `ORG_ADMIN` (일부 `SUPER_ADMIN`)

---

## 1. 사용자 관리

### 사용자 목록 조회

```
GET /api/v1/admin/users?page=1&page_size=20&search=홍길동
Authorization: Bearer <admin-token>
```

**쿼리 파라미터:**

| 파라미터 | 설명 |
|----------|------|
| `search` | 이름/이메일 검색 |
| `filter[status]` | `ACTIVE`, `INACTIVE`, `SUSPENDED` |
| `filter[role_name]` | 역할 필터 |

---

### 사용자 상세 조회

```
GET /api/v1/admin/users/{user_id}
Authorization: Bearer <admin-token>
```

---

### 사용자 생성

```
POST /api/v1/admin/users
Authorization: Bearer <admin-token>

{
  "email": "newuser@example.com",
  "display_name": "신규사용자",
  "role_name": "AUTHOR",
  "org_id": "org-uuid"
}
```

---

### 사용자 수정

```
PUT /api/v1/admin/users/{user_id}
Authorization: Bearer <admin-token>

{
  "display_name": "수정된 이름",
  "role_name": "REVIEWER",
  "status": "ACTIVE"
}
```

---

### 사용자 비활성화

```
DELETE /api/v1/admin/users/{user_id}
Authorization: Bearer <admin-token>
```

물리 삭제가 아닌 `status = INACTIVE` 처리.

---

## 2. 조직 관리

### 조직 목록 / 생성 / 수정 / 삭제

```
GET    /api/v1/admin/organizations
POST   /api/v1/admin/organizations
PUT    /api/v1/admin/organizations/{org_id}
DELETE /api/v1/admin/organizations/{org_id}
```

```
POST /api/v1/admin/organizations

{
  "name": "법무팀",
  "description": "법무 및 컴플라이언스 담당 조직"
}
```

---

## 3. 역할 관리

### 역할 목록 조회

```
GET /api/v1/admin/roles
Authorization: Bearer <admin-token>
```

**응답:**
```json
{
  "data": [
    {"name": "SUPER_ADMIN", "description": "플랫폼 전체 관리 권한", "is_system": true},
    {"name": "ORG_ADMIN", "description": "조직 범위 관리 권한", "is_system": true},
    {"name": "AUTHOR", "description": "문서 작성 권한", "is_system": true},
    ...
  ]
}
```

시스템 역할(`is_system: true`)은 수정/삭제 불가.

---

## 4. 문서 유형 관리

### 문서 유형 목록

```
GET /api/v1/admin/document-types
Authorization: Bearer <admin-token>
```

내장 타입(POLICY/MANUAL/REPORT/FAQ) + DB 등록 커스텀 타입 반환.

---

### 문서 유형 상세 조회

```
GET /api/v1/admin/document-types/{type_code}
Authorization: Bearer <admin-token>
```

---

### 문서 유형 생성

```
POST /api/v1/admin/document-types
Authorization: Bearer <admin-token>

{
  "type_code": "GUIDE",
  "display_name": "가이드라인",
  "description": "절차 안내 문서",
  "schema_fields": [
    {"name": "target_audience", "type": "string", "required": true},
    {"name": "version", "type": "string", "required": false, "default": "1.0"}
  ]
}
```

---

### 플러그인 설정 조회/수정

```
GET /api/v1/admin/document-types/{type_code}/plugin
PUT /api/v1/admin/document-types/{type_code}/plugin
```

```
PUT /api/v1/admin/document-types/POLICY/plugin

{
  "chunking": {
    "max_tokens": 600,
    "overlap": 80
  },
  "rag": {
    "top_n": 7,
    "max_context_tokens": 8000
  }
}
```

설정은 코드 내 내장 플러그인 기본값을 DB 레벨에서 오버라이드한다.

---

## 5. 감사 로그 조회

```
GET /api/v1/admin/audit-logs
Authorization: Bearer <admin-token>
```

**쿼리 파라미터:**

| 파라미터 | 설명 |
|----------|------|
| `filter[event_type]` | 이벤트 유형 (예: `document.created`, `version.published`) |
| `filter[actor_user_id]` | 특정 사용자의 이벤트만 |
| `filter[document_id]` | 특정 문서의 이벤트만 |
| `date_from` | 시작 날짜 (ISO 8601) |
| `date_to` | 종료 날짜 (ISO 8601) |

**응답:**
```json
{
  "data": [
    {
      "id": "...",
      "event_type": "version.published",
      "occurred_at": "2026-04-10T14:30:00+00:00",
      "actor_user_id": "user-002",
      "actor_role": "APPROVER",
      "document_id": "550e8400-...",
      "version_id": "660e8400-...",
      "previous_state": "approved",
      "new_state": "published",
      "action_result": "success",
      "request_id": "req-abc123"
    }
  ]
}
```

---

## 6. 백그라운드 작업 관리

### 작업 목록 조회

```
GET /api/v1/admin/jobs?filter[status]=RUNNING
Authorization: Bearer <admin-token>
```

### 작업 상세 조회

```
GET /api/v1/admin/jobs/{job_id}
Authorization: Bearer <admin-token>
```

### 작업 취소

```
DELETE /api/v1/admin/jobs/{job_id}
Authorization: Bearer <admin-token>
```

---

## 7. API Key 관리

### API Key 목록

```
GET /api/v1/admin/api-keys
Authorization: Bearer <admin-token>
```

### API Key 발급

```
POST /api/v1/admin/api-keys
Authorization: Bearer <admin-token>

{
  "name": "CI/CD 연동",
  "scope": "READ_ONLY",
  "expires_at": "2027-01-01T00:00:00Z"
}
```

응답에 `key` 필드 포함 (최초 1회만 노출).

### API Key 폐기

```
DELETE /api/v1/admin/api-keys/{key_id}
Authorization: Bearer <admin-token>

{
  "reason": "보안 이슈로 즉시 폐기"
}
```

---

## 8. Scope Profile 관리 (S3 Phase 4)

외부 AI 에이전트(MCP 클라이언트)의 ACL 을 관리하는 엔드포인트.

### Scope Profile 목록 / 상세 조회

```
GET /api/v1/admin/scope-profiles
GET /api/v1/admin/scope-profiles/{profile_id}
Authorization: Bearer <admin-token>
```

### Scope Profile 생성

```
POST /api/v1/admin/scope-profiles
Authorization: Bearer <admin-token>

{
  "name": "rag-agent-prod",
  "description": "RAG 에이전트 운영 프로파일",
  "rate_limit_per_minute": 60,
  "read_filter": {
    "published_only": true,
    "accessible_roles": ["VIEWER", "AUTHOR"]
  },
  "allowed_tools": [
    "search_documents",
    "fetch_node",
    "verify_citation",
    "read_document_render",
    "resolve_document_reference",
    "search_nodes",
    "read_annotations",
    "mimir.vectorization.status"
  ],
  "use_defaults": false
}
```

`use_defaults=true` 로 설정 시 `allowed_tools` 무시, `manifest.default_enabled=true` 인 도구만 자동 화이트리스트.

### Scope Profile 수정 / 비활성화

```
PUT    /api/v1/admin/scope-profiles/{profile_id}
DELETE /api/v1/admin/scope-profiles/{profile_id}
```

`DELETE` 는 물리 삭제가 아닌 비활성화 (해당 프로파일 사용하는 API Key 즉시 차단).

---

## 9. MCP 매니페스트 (S3 Phase 4)

### 외부 매니페스트 (공개)

```
GET /mcp/manifest
```

외부 노출용 9 도구 manifest — `risk_tier`, `maturity`, `status`, `exposure_policy`, `default_enabled`, `requires`, `preferred_use`, `policy_profile`, `streaming_supported` 9 필드. 노출 정책 `never` 인 도구는 응답에서 제거됨.

### 운영자 매니페스트 (전체 view)

```
GET /api/v1/admin/mcp/manifest
Authorization: Bearer <admin-token>
```

L4 도구 포함 모든 도구의 manifest 를 반환 — drift 게이트 / Admin UI 의 ScopeProfile 도구 토글 UI 가 사용.

자세한 내용은 [매니페스트 정책](../MCP%20기술서/매니페스트%20정책.md) 참조.

---

## 10. 시스템 정보

### 헬스체크 (공개)

```
GET /api/v1/system/health
```

```json
{"status": "success", "data": {"healthy": true}}
```

### 서비스 메타 정보 (공개)

```
GET /api/v1/system/info
```

```json
{
  "data": {
    "service": "mimir",
    "api_version": "v1",
    "environment": "production"
  }
}
```

### Prometheus 메트릭 (내부망 전용)

```
GET /api/v1/system/metrics
```

운영 환경에서는 내부 IP (`127.*`, `10.*`, `192.168.*`)에서만 접근 가능.
