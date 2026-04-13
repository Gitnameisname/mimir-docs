# Task2_endpoint_naming_rules.md
# 엔드포인트 명명 규칙

---

## 1. 문서 목적

이 문서는 Mimir 플랫폼 API의 모든 엔드포인트에 적용되는 URI 명명 원칙과 규칙을 확정한다.

이 문서에서 정의한 규칙은 이후 구현되는 모든 API에 예외 없이 적용된다. 예외가 필요한 경우 명시적 이유와 함께 기록해야 한다.

연관 문서: `Task2_rest_resource_map.md` (리소스 구조), `Task1_api_architecture_principles.md` (아키텍처 원칙)

---

## 2. 기본 원칙

### 2-1. Path는 명사 중심

| 구분 | 예시 | 판정 |
|---|---|---|
| 권장 | `/api/v1/documents` | 명사 중심 |
| 금지 | `/api/v1/getDocuments` | 동사 포함 |
| 금지 | `/api/v1/fetchDocument` | 동사 포함 |
| 금지 | `/api/v1/createVersion` | 동사 포함 |

행위는 HTTP 메서드(GET/POST/PUT/PATCH/DELETE)로 표현한다.

### 2-2. Collection은 복수형

모든 컬렉션 path는 복수형 명사를 사용한다.

| 구분 | 예시 | 판정 |
|---|---|---|
| 권장 | `/api/v1/documents` | 복수형 |
| 권장 | `/api/v1/document-types` | 복수형 |
| 금지 | `/api/v1/document` | 단수형 |
| 금지 | `/api/v1/documentType` | 단수형 + camelCase |

### 2-3. Path는 kebab-case

Path segment는 **kebab-case**를 사용한다. camelCase, snake_case 금지.

| 구분 | 예시 | 판정 |
|---|---|---|
| 권장 | `/api/v1/document-types` | kebab-case |
| 권장 | `/api/v1/audit-logs` | kebab-case |
| 금지 | `/api/v1/documentTypes` | camelCase |
| 금지 | `/api/v1/document_types` | snake_case |
| 금지 | `/api/v1/DocumentTypes` | PascalCase |

**적용 범위**: path segment에만 해당. 쿼리 파라미터 명명은 별도 규칙 적용 (Task 3-6에서 정의).

---

## 3. Path Parameter 명명 규칙

### 3-1. 기본 규칙

- Path parameter는 **camelCase** 사용
- 리소스 이름을 접두사로 붙인 `{resourceId}` 형태를 기본으로 한다
- 축약어 사용 금지 (예: `docId` → `documentId`)

| 구분 | 예시 | 판정 |
|---|---|---|
| 권장 | `{documentId}` | 리소스명 + Id |
| 권장 | `{versionId}` | 리소스명 + Id |
| 권장 | `{organizationId}` | 리소스명 + Id |
| 비권장 | `{id}` | 어떤 리소스인지 불명확 |
| 금지 | `{doc_id}` | snake_case |
| 금지 | `{docId}` | 축약어 |

### 3-2. 예시

```
GET /api/v1/documents/{documentId}
GET /api/v1/documents/{documentId}/versions/{versionId}
GET /api/v1/documents/{documentId}/versions/{versionId}/nodes/{nodeId}
GET /api/v1/webhooks/{webhookId}/deliveries/{deliveryId}
```

---

## 4. Subresource Naming 규칙

- Subresource는 부모 리소스 path 하위에 복수형 명사로 표현
- Subresource도 kebab-case 적용
- 관계를 나타내는 동사형 사용 금지

| 구분 | 예시 | 판정 |
|---|---|---|
| 권장 | `/documents/{documentId}/versions` | 복수형 명사 |
| 권장 | `/documents/{documentId}/permissions` | 복수형 명사 |
| 권장 | `/documents/{documentId}/audit-logs` | kebab-case 복수형 |
| 금지 | `/documents/{documentId}/getVersions` | 동사형 |
| 금지 | `/documents/{documentId}/version` | 단수형 |
| 금지 | `/documents/{documentId}/auditLogs` | camelCase |

---

## 5. Action Endpoint 규칙

### 5-1. `:action` suffix 패턴 채택

Action endpoint는 **리소스 path에 `:action` suffix**를 붙이는 방식을 사용한다.

```
POST /api/v1/documents/{documentId}:publish
POST /api/v1/documents/{documentId}:archive
POST /api/v1/documents/{documentId}:restore
POST /api/v1/documents/{documentId}:clone
POST /api/v1/documents/{documentId}:validate
POST /api/v1/documents/{documentId}:reindex
POST /api/v1/versions/{versionId}:compare
POST /api/v1/webhooks/{webhookId}/deliveries/{deliveryId}:resend
POST /api/v1/operations/{operationId}:cancel
```

### 5-2. Action 명명 규칙

- Action은 **kebab-case 동사 또는 동사+명사** 형태
- 단어 결합 시 hyphen 사용

| 구분 | 예시 | 판정 |
|---|---|---|
| 권장 | `:publish` | 단일 동사 |
| 권장 | `:bulk-archive` | 동사+형용사 |
| 권장 | `:regenerate-preview` | 동사+명사 |
| 금지 | `:doPublish` | 불필요한 do 접두사 |
| 금지 | `:publishDocument` | camelCase |
| 금지 | `:publish_document` | snake_case |

### 5-3. 비채택 패턴 비교

| 패턴 | 예시 | 비채택 이유 |
|---|---|---|
| subresource 방식 | `/documents/{id}/publish` | 별도 리소스처럼 읽혀 혼동 |
| 최상위 동사 | `/publish-document` | 리소스 중심 원칙 위반 |
| suffix 없이 POST | `POST /documents/{id}` + body type | 의도 불명확 |
| `:action` suffix | `POST /documents/{id}:publish` | **채택** |

### 5-4. Action Endpoint 남발 방지

다음 경우는 action endpoint 없이 표준 HTTP 메서드로 처리한다:

| 케이스 | 올바른 방식 |
|---|---|
| 단순 필드 업데이트 | `PATCH /documents/{documentId}` |
| 리소스 삭제 | `DELETE /documents/{documentId}` |
| 새 리소스 생성 | `POST /documents` |
| 상태를 단순 PATCH로 변경 가능한 경우 | `PATCH /documents/{documentId}` + body |

---

## 6. Bulk Endpoint Naming

### 6-1. `:bulk-action` suffix 방식

```
POST /api/v1/documents:bulk-create
POST /api/v1/documents:bulk-archive
PATCH /api/v1/documents:bulk-update
DELETE /api/v1/documents:bulk-delete
```

### 6-2. 비동기 Bulk → operations 위임

건수가 많거나 처리 시간이 불확실한 bulk 작업은 `operations` 리소스를 통해 처리한다.

```
POST /api/v1/operations
  body: { type: "bulk-archive", targets: [...] }
→ 응답: { operationId: "...", status: "pending" }

GET /api/v1/operations/{operationId}
→ 작업 상태 폴링
```

---

## 7. Admin / 시스템 전용 API 구분

### 7-1. 채택 방향

Admin 전용 기능은 **원칙적으로 인가 레벨로 구분**한다. 동일한 리소스 path에서 요청자의 역할에 따라 접근 가능 범위가 달라진다.

```
GET /api/v1/audit-logs       → Admin만 접근 가능 (인가 레벨 제어)
GET /api/v1/documents        → 일반 사용자도 접근 (본인 권한 범위)
```

### 7-2. `/admin/` prefix 허용 케이스

다음 경우에는 `/api/v1/admin/` prefix를 선택적으로 허용한다:
- 일반 사용자에게 절대 노출해서는 안 되는 시스템 제어 기능
- 시스템 설정, 문서 타입 스키마 관리, 사용자 강제 조치 등

```
/api/v1/admin/system-config
/api/v1/admin/users/{userId}/force-deactivate
```

**주의**: `/admin/` prefix는 보안의 대체 수단이 아니다. 인가 검증은 반드시 유지.

### 7-3. 내부/시스템 전용 API

플랫폼 내부 서비스 간 통신에만 사용하는 API는:
- 외부 API 문서에 포함하지 않는다
- `/internal/` prefix를 사용하거나 별도 서비스 포트에 분리한다
- 동일한 인가 원칙을 따른다

---

## 8. 검색 / 조회 / 상태 계열 Naming

| 케이스 | 채택 방식 | 이유 |
|---|---|---|
| 검색 실행 | `POST /api/v1/searches` | 리소스 개념으로 표현 |
| 단순 키워드 필터 | `GET /api/v1/documents?q=...` | 쿼리 파라미터 |
| 인덱싱 상태 조회 | `GET /documents/{id}/index-status` | 상태 서브리소스 |
| 비동기 작업 상태 | `GET /operations/{operationId}` | operations 리소스 |
| 이벤트 조회 | `GET /api/v1/events` | 전역 이벤트 컬렉션 |

**`/search` vs `/searches` 결정**: `/searches` 채택. 리소스 개념으로 표현하여 저장 검색, 검색 히스토리 등 확장 가능.

---

## 9. 실험적 / 베타 API Naming

- 실험적 API는 `/api/v1/` 하에 두되, 응답 헤더 또는 문서에 `Experimental: true`를 표시한다.
- 안정화 전까지 계약 변경이 있을 수 있음을 명시한다.
- 별도 `/beta/` prefix 사용은 비권장 (소비자 혼란 유발).

---

## 10. 금지 예시 / 권장 예시 종합

### 금지 예시

```
❌ GET  /api/v1/getDocuments
❌ POST /api/v1/createDocument
❌ GET  /api/v1/documentTypes          (camelCase)
❌ GET  /api/v1/document_types         (snake_case)
❌ GET  /api/v1/document               (단수형 컬렉션)
❌ POST /api/v1/documents/{id}/publish  (action을 subresource처럼 표현)
❌ POST /api/v1/publishDocument         (최상위 동사형)
❌ GET  /api/v1/documents/{doc_id}      (snake_case parameter)
❌ GET  /api/v1/documents/{docId}       (축약 parameter)
❌ GET  /api/v1/search                  (단수형 검색)
❌ GET  /api/v1/documents/1/versions/2/nodes/3/metadata/fields  (4단계 이상 nesting)
```

### 권장 예시

```
✅ GET    /api/v1/documents
✅ POST   /api/v1/documents
✅ GET    /api/v1/documents/{documentId}
✅ PATCH  /api/v1/documents/{documentId}
✅ DELETE /api/v1/documents/{documentId}
✅ POST   /api/v1/documents/{documentId}:publish
✅ POST   /api/v1/documents/{documentId}:archive
✅ POST   /api/v1/documents/{documentId}:clone
✅ GET    /api/v1/documents/{documentId}/versions
✅ GET    /api/v1/documents/{documentId}/versions/{versionId}
✅ GET    /api/v1/documents/{documentId}/versions/{versionId}/nodes
✅ GET    /api/v1/document-types
✅ GET    /api/v1/audit-logs
✅ GET    /api/v1/webhook-deliveries    (또는 /webhooks/{id}/deliveries)
✅ POST   /api/v1/searches
✅ GET    /api/v1/operations/{operationId}
✅ POST   /api/v1/documents:bulk-archive
```

---

## 11. 규칙 요약표

| 항목 | 규칙 |
|---|---|
| Path segment | kebab-case |
| Collection | 복수형 명사 |
| Item | `/{resourceId}` |
| Path parameter | camelCase + `{resourceNameId}` 형태 |
| Subresource | 부모 path 하위 복수형 명사 |
| Action endpoint | `:{kebab-case-action}` suffix, `POST` 메서드 |
| Bulk action | `:{bulk-action}` suffix 또는 operations 리소스 |
| Admin 전용 | 인가 레벨 우선, 필요시 `/admin/` prefix |
| 검색 | `/searches` 리소스 (복수형) |
| 실험적 API | 헤더/문서 표시, `/beta/` prefix 비권장 |

---

## 자체 점검 요약

### 명명 규칙 핵심 요약

- Path: kebab-case, 복수형 명사
- Parameter: camelCase `{resourceNameId}`
- Action: `:kebab-case-action` suffix with POST
- Bulk: `:bulk-action` suffix 또는 operations 리소스
- Admin: 인가 레벨 구분 우선, 예외만 `/admin/` prefix

### 후속 Task 연결 포인트

- Task 3-3: Admin 네임스페이스 분리 수준 최종 확정
- Task 3-4: API 버전 prefix 관리 전략 (`/api/v1/` → `v2` 전환 방식)
- Task 3-6: 쿼리 파라미터 명명 규칙 (pagination, filter, sort) 정의
- Task 3-8: operations/events/webhooks 리소스의 action endpoint 상세 정의

### 의도적 미결정 사항

- 쿼리 파라미터 명명 규칙 (camelCase vs snake_case) — Task 3-6에서 결정
- OpenAPI 스펙의 operationId 명명 규칙 — 구현 Phase에서 결정
- Internal API prefix 분리 수준 — 구현 Phase에서 결정
