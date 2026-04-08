# Task2_rest_resource_map.md
# 핵심 리소스 기반 REST API 구조 설계 — 리소스 맵

---

## 1. 문서 목적

이 문서는 Mimir 플랫폼 API에서 다뤄야 할 핵심 리소스를 식별하고, REST 기반 URL 구조의 기준선을 정의한다.

엔드포인트 전체 목록을 나열하는 것이 아니라, 다음을 확정하는 것이 목적이다:
- 어떤 객체가 API 리소스인가
- 각 리소스는 top-level인가 subresource인가
- 리소스 간 관계는 어떻게 표현되는가
- 공개 범위와 주요 소비자는 누구인가

이 문서는 Task 3-3(인증/인가), Task 3-5(응답 포맷), Task 3-6(목록 조회), Task 3-8(async/event), Task 3-9(AI/RAG)의 기준선이다.

---

## 2. 리소스 분류 체계

리소스를 6개 그룹으로 분류한다.

| 분류 | 설명 | 주요 소비자 |
|---|---|---|
| **Core Domain** | 문서 플랫폼의 핵심 도메인 객체 | User UI, AI/RAG, External |
| **Identity / Access** | 조직, 사용자, 역할, 멤버십 | Admin UI, External |
| **Governance / Audit** | 감사 로그, 활동 추적 | Admin UI, 컴플라이언스 |
| **Integration / Event** | 이벤트, 웹훅, 비동기 작업 | External, Internal Service |
| **Search / AI Extension** | 검색, AI/RAG 인터페이스 | User UI, AI/RAG |
| **Operational** | 인덱싱 상태, 시스템 작업 | Admin UI, Internal |

---

## 3. 핵심 리소스 식별 및 판단

### 3-1. Core Domain Resources

#### `documents`
- **분류**: Top-level
- **이유**: 플랫폼의 핵심 도메인 객체. 독립적 식별/조회 가치 최상.
- **canonical path**: `/api/v1/documents`
- **노출 여부**: 공개 (권한 필터링 적용)

#### `versions`
- **분류**: Subresource of `documents` + 독립 조회 canonical endpoint
- **이유**: 버전은 문서 없이 독립 존재하지 않으나, `versionId`로 직접 참조하는 케이스(AI citation 등)가 많다.
- **canonical path**: `/api/v1/documents/{documentId}/versions` (primary), `/api/v1/versions/{versionId}` (read-only lookup)
- **노출 여부**: 공개 (권한 필터링 적용)

#### `nodes`
- **분류**: Subresource of `versions`
- **이유**: 노드는 버전 아래에 존재. 단, AI/RAG가 노드를 직접 참조해야 하므로 canonical endpoint 고려.
- **canonical path**: `/api/v1/documents/{documentId}/versions/{versionId}/nodes` (primary), `/api/v1/nodes/{nodeId}` (read-only lookup)
- **노출 여부**: 공개 (권한 필터링 적용)
- **nesting 주의**: 3단계 중첩. 깊이 문제 있으나 도메인 구조상 불가피. 직접 lookup endpoint를 병행 제공하여 완화.

#### `document-types`
- **분류**: Top-level (설정성 리소스)
- **이유**: 문서 타입은 여러 문서에 공유되는 독립 설정 객체. 문서 경로 하위에 두면 관리 불가.
- **canonical path**: `/api/v1/document-types`
- **노출 여부**: User는 조회만, Admin은 CRUD

#### `attachments`
- **분류**: Subresource of `documents` 또는 `versions`
- **이유**: 첨부파일은 문서/버전에 종속.
- **canonical path**: `/api/v1/documents/{documentId}/attachments`
- **노출 여부**: 공개 (권한 필터링 적용)

---

### 3-2. Identity / Access Resources

#### `organizations`
- **분류**: Top-level
- **이유**: 테넌트 단위 최상위 관리 객체. 플랫폼의 조직 스코프 기준.
- **canonical path**: `/api/v1/organizations`
- **노출 여부**: Admin UI 주. 일반 사용자는 자신이 속한 org 조회만.

#### `users`
- **분류**: Top-level (또는 org 하위 subresource로 병행)
- **이유**: 사용자는 독립적 식별 가치가 있으나, org 내 사용자 관리는 subresource로 표현 가능.
- **canonical path**: `/api/v1/users` (전역 관리용), `/api/v1/organizations/{orgId}/members` (조직 내 멤버)
- **노출 여부**: Admin UI. 일반 사용자는 본인 정보만.

#### `memberships`
- **분류**: Subresource (org 또는 user 하위)
- **이유**: 조직-사용자 관계 표현. 독립 API 리소스보다 subresource가 적합.
- **canonical path**: `/api/v1/organizations/{orgId}/members`
- **노출 여부**: Admin UI

#### `roles`
- **분류**: Top-level (또는 org 하위 설정 리소스)
- **이유**: 역할은 조직 범위 설정 객체.
- **canonical path**: `/api/v1/roles`
- **노출 여부**: Admin UI

#### `permissions` (ACL entries)
- **분류**: Subresource of documents/versions (read), top-level 관리 가능
- **이유**: ACL은 리소스별 접근 제어. 문서에 귀속되는 것이 자연스럽다.
- **canonical path**: `/api/v1/documents/{documentId}/permissions`
- **노출 여부**: 문서 소유자 및 Admin

---

### 3-3. Governance / Audit Resources

#### `audit-logs`
- **분류**: Top-level (전역 관리 리소스)
- **이유**: 감사 로그는 특정 도메인 리소스에 종속되지 않는 전역 추적 기록. Admin이 전체 조회해야 함.
- **canonical path**: `/api/v1/audit-logs`
- **노출 여부**: Admin 전용. 특정 문서의 감사는 `/api/v1/documents/{documentId}/audit-logs` subresource로도 노출 가능.

#### `activities`
- **분류**: Subresource (사용자 또는 문서 하위) 또는 별도 top-level
- **이유**: 활동 추적은 감사 로그보다 UX에 가까움 (최근 활동 피드 등). 용도에 따라 분리.
- **canonical path**: `/api/v1/activities` (사용자 피드용) 또는 `/api/v1/documents/{documentId}/activities`
- **노출 여부**: User UI (본인 활동), Admin UI (전체)
- **미결정**: activities와 audit-logs의 경계를 Task 3-6에서 세부 정의 필요

---

### 3-4. Integration / Event Resources

#### `webhooks`
- **분류**: Top-level
- **이유**: 웹훅 구독은 특정 리소스에 종속되지 않는 독립 설정.
- **canonical path**: `/api/v1/webhooks`
- **노출 여부**: External 통합, Admin

#### `webhook-deliveries`
- **분류**: Subresource of `webhooks`
- **이유**: 전달 기록은 특정 웹훅에 종속.
- **canonical path**: `/api/v1/webhooks/{webhookId}/deliveries`
- **노출 여부**: External 통합, Admin

#### `events`
- **분류**: Top-level (read-only 이벤트 로그)
- **이유**: 플랫폼 이벤트를 외부가 폴링 또는 조회하는 경우를 위한 인터페이스. 실시간 스트리밍은 별도.
- **canonical path**: `/api/v1/events`
- **노출 여부**: External 통합, Admin
- **참고**: 이벤트 발행 자체는 내부 구현. API는 조회 인터페이스만 노출.

#### `operations` (Jobs)
- **분류**: Top-level
- **이유**: 비동기 장기 작업 결과 추적. 특정 도메인 리소스에 종속되지 않는 작업 레지스트리.
- **canonical path**: `/api/v1/operations`
- **노출 여부**: 작업을 트리거한 주체 및 Admin
- **명명 결정**: `operations` 채택 (`jobs`보다 REST 관행에 가깝고 의미 범위가 넓음)

---

### 3-5. Search / AI Extension Resources

#### `searches`
- **분류**: Top-level (리소스 개념)
- **이유**: 검색은 단순 RPC가 아니라 검색 세션/결과를 리소스로 다룰 수 있음. 검색 히스토리, 저장 검색 등 확장 가능.
- **canonical path**: `/api/v1/searches`
- **노출 여부**: User UI, AI/RAG
- **참고**: 단순 조회성 검색은 `POST /searches` 또는 `GET /documents?q=...` 형태 병행 허용. 세부는 Task 3-9에서 결정.

#### AI/RAG 접근 리소스
- **분류**: 별도 top-level 리소스로 분리하지 않음
- **이유**: AI는 기존 `documents`, `versions`, `nodes`, `searches` 리소스를 공식 API 소비자로 사용. 별도 AI 전용 리소스 생성 금지 (Task 3-1 원칙).
- **확장 방향**: 쿼리 파라미터 확장, 필드 선택, projection으로 AI 요구 수용.
- **인덱싱 상태**: `/api/v1/documents/{documentId}/index-status` 형태로 subresource 고려. 세부는 Task 3-9.

---

### 3-6. 내부 구현으로 남길 것 (API 리소스 아님)

| 항목 | 판단 이유 |
|---|---|
| `comments` / `annotations` | 현재 도메인 모델에 없음. 노드 메타데이터로 수용 가능. 후속 Phase에서 검토. |
| `tags` / `labels` | 문서/버전의 메타데이터 필드로 수용. 독립 리소스 불필요. |
| `search-results` | searches의 응답 데이터. 별도 리소스 불필요. |
| `document-schemas` | `document-types` 내 속성으로 관리. 별도 리소스 분리 여부는 Task 3-5에서 결정. |
| 임베딩/벡터 인덱스 내부 데이터 | 완전히 내부 구현. API 노출 불필요. |

---

## 4. 상위 URL 구조

### 4-1. 버전 prefix 원칙

- 모든 API는 `/api/v1/` prefix를 사용한다.
- Admin 전용 API는 `/api/v1/admin/` 또는 동일 path에서 권한으로만 구분하는 방향 중 하나를 선택한다.
- **채택 방향**: `/api/v1/` 하에 통일. Admin 전용 기능은 path 분리 대신 인가 레벨로 구분 우선. 단, 명확한 관리 전용 리소스(예: 시스템 설정)는 `/api/v1/admin/` prefix를 선택적으로 허용.
- 세부 버전 관리 전략은 Task 3-4에서 확정.

### 4-2. 최상위 리소스 구조 (Top-level Collections)

```
/api/v1/
  documents
  document-types
  organizations
  users
  roles
  webhooks
  events
  operations
  searches
  audit-logs
  activities
```

### 4-3. 주요 Subresource 관계

```
/api/v1/documents/{documentId}/
  versions/
    {versionId}/
      nodes/
        {nodeId}
  attachments/
  permissions/
  audit-logs/          ← 문서 단위 감사 조회
  activities/          ← 문서 단위 활동 조회

/api/v1/organizations/{orgId}/
  members/

/api/v1/webhooks/{webhookId}/
  deliveries/

/api/v1/operations/{operationId}/
  (결과 조회)
```

### 4-4. 조직 스코프 처리 방향

**결정**: 조직 스코프는 path에 직접 포함하지 않고, 인증 컨텍스트(request security context)로 처리한다.

이유:
- path에 `orgId`를 포함하면 (`/api/v1/organizations/{orgId}/documents/...`) 모든 URL이 깊어지고 소비자 부담이 증가한다.
- 대부분의 요청은 단일 조직 내에서 발생하므로, 인증 토큰에서 org 스코프를 추출하는 방식이 더 실용적이다.
- 단, 관리자가 여러 조직을 관리하는 경우, `orgId` 쿼리 파라미터 또는 org 하위 endpoint를 선택적으로 허용한다.

---

## 5. Collection / Item / Subresource 규칙

### 5-1. 기본 패턴

| 패턴 | 형태 | 예시 |
|---|---|---|
| Collection | `/resources` | `/api/v1/documents` |
| Item | `/resources/{id}` | `/api/v1/documents/{documentId}` |
| Subresource Collection | `/resources/{id}/subresources` | `/api/v1/documents/{documentId}/versions` |
| Subresource Item | `/resources/{id}/subresources/{subId}` | `/api/v1/documents/{documentId}/versions/{versionId}` |

### 5-2. Canonical Path 원칙

- 모든 리소스는 **하나의 canonical path**를 가진다.
- 동일 리소스를 여러 부모 아래에 중복 노출하는 것은 원칙적으로 금지.
- 단, 직접 조회 편의를 위한 **read-only secondary path** 허용 (예: `/api/v1/versions/{versionId}`).
- Secondary path는 canonical path와 동일한 응답 구조를 반환해야 한다.

### 5-3. Nesting 깊이 원칙

- **1~2단계 nesting을 기본 권장.**
- **3단계 nesting은 도메인 구조상 불가피한 경우에만 허용** (예: documents → versions → nodes).
- **4단계 이상 nesting 금지.** 깊은 탐색이 필요하면 직접 lookup endpoint 제공.
- 예외 케이스는 반드시 명시적 이유와 함께 기록.

### 5-4. 관계성 리소스 표현

- 조직-사용자 관계 → `/organizations/{orgId}/members` (membership을 member로 표현)
- 문서-권한 관계 → `/documents/{documentId}/permissions`
- 웹훅-전달 관계 → `/webhooks/{webhookId}/deliveries`

---

## 6. Action Endpoint 허용 기준

### 6-1. 기준 원칙

Action endpoint는 **허용하되, 엄격한 기준 아래 제한적으로 사용**한다.

다음 조건 중 하나 이상을 충족해야 action endpoint를 허용한다:
1. **상태 전이**: PATCH로 표현 가능한 상태 변경이지만 의미적으로 명확한 행위가 있는 경우 (예: publish, archive)
2. **계산/검증 트리거**: 리소스 생성/수정 없이 계산/검증만 하는 경우 (예: validate, compare)
3. **비동기 작업 트리거**: 장기 작업을 시작하는 경우 (예: reindex, regenerate-preview)
4. **재시도/재전송**: 기존 작업/전달을 다시 실행하는 경우 (예: retry, resend)

다음은 **action endpoint를 사용하지 말아야 하는** 경우:
- 단순 필드 업데이트 → PATCH 사용
- 리소스 생성 → POST 사용
- 리소스 삭제 → DELETE 사용

### 6-2. Action Endpoint 표현 방식

**채택**: `:action` suffix 방식 (Google API Design Guide 방식)

```
POST /api/v1/documents/{documentId}:publish
POST /api/v1/documents/{documentId}:archive
POST /api/v1/documents/{documentId}:restore
POST /api/v1/documents/{documentId}:clone
POST /api/v1/documents/{documentId}:validate
POST /api/v1/versions/{versionId}:compare
POST /api/v1/documents/{documentId}:reindex
POST /api/v1/webhooks/{webhookId}/deliveries/{deliveryId}:resend
POST /api/v1/operations/{operationId}:cancel
```

**비채택 이유**:
- `/documents/{id}/publish` → 별도 subresource처럼 읽혀서 혼동 가능
- `/publish-document` → 리소스 중심 원칙 위반

### 6-3. 상태 전이의 PATCH vs Action 판단 기준

| 케이스 | 권장 방식 | 이유 |
|---|---|---|
| 단일 필드 상태 변경 | PATCH | 단순 업데이트로 충분 |
| 의미 있는 상태 전이 (비가역적 또는 워크플로우) | Action | 명시적 의도 표현 필요 |
| 비동기 작업 시작 | Action → operation 반환 | 장기 작업이므로 operation ID 반환 |
| 검증/계산 (부수효과 없음) | Action (POST) | 멱등 여부와 무관, 명시적 트리거 |

---

## 7. Bulk Operation 설계 방향

### 7-1. Bulk 작업의 필요성

다음 bulk 작업이 플랫폼에서 필요하다:
- bulk document create (일괄 임포트)
- bulk permission change (여러 문서 권한 일괄 변경)
- bulk tag/label update
- bulk archive/delete

### 7-2. 처리 방향

| 방식 | 적용 기준 |
|---|---|
| 동기 bulk (소량) | 건수가 적고 즉각 응답 가능한 경우 |
| 비동기 bulk → operation 반환 | 건수가 많거나 처리 시간이 불확실한 경우 |

**Bulk endpoint 형태**:
```
POST /api/v1/documents:bulk-create
POST /api/v1/documents:bulk-archive
PATCH /api/v1/documents:bulk-update
```

또는 별도 리소스 방식:
```
POST /api/v1/operations
  body: { type: "bulk-archive", targets: [...] }
```

**채택 방향**: 비동기 bulk는 `operations` 리소스를 통해 처리. 응답으로 `operationId`를 반환하고 상태를 폴링. 동기 소량 bulk는 `:bulk-action` suffix 허용.

### 7-3. Partial Success

- Bulk 작업에서 일부 항목만 성공하는 partial success 케이스가 있음.
- 응답 구조에서 성공/실패 항목을 구분하는 방식이 필요하다.
- 세부 응답 스키마는 Task 3-5(공통 응답 포맷)에서 결정.

---

## 8. 검색 / AI 연계 리소스 방향

### 8-1. 검색 리소스

- `/api/v1/searches`를 top-level 리소스로 채택.
- 검색 실행: `POST /api/v1/searches` (검색 실행 → 결과 반환 또는 search session 생성)
- 단순 필터링: `GET /api/v1/documents?q=...&status=published` (쿼리 파라미터 방식 병행 허용)
- 저장 검색/검색 히스토리: `/api/v1/searches/{searchId}` 형태로 확장 가능.

### 8-2. AI/RAG 접근 구조

- AI 소비자는 기존 리소스 구조를 그대로 사용.
- 노드 단위 접근: `/api/v1/documents/{documentId}/versions/{versionId}/nodes/{nodeId}`
- 버전 명시 접근: `/api/v1/versions/{versionId}` canonical endpoint 제공
- 구조 탐색: document, version, node hierarchy를 그대로 API로 표현
- 인덱싱 상태: `/api/v1/documents/{documentId}/index-status` 제공 (세부는 Task 3-9)

---

## 9. 운영/거버넌스 리소스 정리

| 리소스 | 접근 유형 | 전역/서브 | 주요 소비자 |
|---|---|---|---|
| `audit-logs` | Admin API | 전역 (+ 문서 하위) | Admin, 감사팀 |
| `activities` | User + Admin API | 전역 (+ 문서/사용자 하위) | User UI, Admin UI |
| `operations` | User + Admin API | 전역 | 작업 트리거한 사용자, Admin |
| `events` | External + Admin API | 전역 (read-only) | External 통합, Admin |
| `webhooks` | External + Admin API | 전역 | External 통합 |
| `webhook-deliveries` | External + Admin API | webhooks 하위 | External 통합, Admin |

---

## 10. 리소스 분류 종합 정리표

| 리소스 | 분류 | Level | canonical path | 공개 범위 | 주요 소비자 |
|---|---|---|---|---|---|
| `documents` | Core Domain | Top-level | `/api/v1/documents` | 공개(권한) | All |
| `versions` | Core Domain | Sub + canonical | `/api/v1/documents/{id}/versions` | 공개(권한) | All |
| `nodes` | Core Domain | Sub + canonical | `/api/v1/documents/{id}/versions/{id}/nodes` | 공개(권한) | All, AI |
| `document-types` | Core Domain | Top-level | `/api/v1/document-types` | User:읽기 Admin:쓰기 | User, Admin |
| `attachments` | Core Domain | Sub | `/api/v1/documents/{id}/attachments` | 공개(권한) | User |
| `organizations` | Identity | Top-level | `/api/v1/organizations` | Admin 주 | Admin |
| `users` | Identity | Top-level | `/api/v1/users` | Admin 주 | Admin, User(본인) |
| `members` | Identity | Sub | `/api/v1/organizations/{id}/members` | Admin | Admin |
| `roles` | Identity | Top-level | `/api/v1/roles` | Admin | Admin |
| `permissions` | Identity | Sub | `/api/v1/documents/{id}/permissions` | 문서소유자, Admin | Admin, User |
| `audit-logs` | Governance | Top-level + Sub | `/api/v1/audit-logs` | Admin | Admin |
| `activities` | Governance | Top-level + Sub | `/api/v1/activities` | User(본인), Admin | User, Admin |
| `operations` | Operational | Top-level | `/api/v1/operations` | 작업 주체, Admin | All |
| `events` | Integration | Top-level | `/api/v1/events` | External, Admin | External |
| `webhooks` | Integration | Top-level | `/api/v1/webhooks` | External, Admin | External |
| `webhook-deliveries` | Integration | Sub | `/api/v1/webhooks/{id}/deliveries` | External, Admin | External |
| `searches` | Search/AI | Top-level | `/api/v1/searches` | 공개 | User, AI |

---

## 자체 점검 요약

### 식별한 핵심 리소스 목록

Core: documents, versions, nodes, document-types, attachments  
Identity: organizations, users, members, roles, permissions  
Governance: audit-logs, activities  
Operational: operations  
Integration: events, webhooks, webhook-deliveries  
Search/AI: searches

### Top-level / Subresource 설계 기준

- 독립 식별/조회 가치가 높으면 top-level
- 부모 없이는 존재 의미가 약하면 subresource
- 편의를 위한 read-only canonical secondary path 병행 허용

### Action Endpoint 허용 기준 요약

상태 전이(비가역/워크플로우), 계산/검증 트리거, 비동기 작업 시작, 재시도/재전송만 허용. `:action` suffix 방식 채택.

### 후속 Task 연결 포인트

- Task 3-3: 권한 스코프를 path 아닌 auth context로 처리하는 방식 구체화
- Task 3-4: `/api/v1/` prefix 버전 관리 전략 확정
- Task 3-5: Bulk partial success 응답 구조 확정
- Task 3-6: pagination/filter/sort를 모든 collection에 일관 적용
- Task 3-8: operations 리소스를 통한 async 작업 흐름 설계
- Task 3-9: AI/RAG 접근 패턴 및 index-status 상세 설계

### 의도적 미결정 사항

- Admin 네임스페이스(`/admin/`) 분리 여부 (Task 3-3에서 결정)
- `activities`와 `audit-logs` 경계 정의 (Task 3-6에서 세부 정의)
- AI composite read 모델 상세 구조 (Task 3-9에서 결정)
- `document-schemas`의 독립 리소스 분리 여부 (Task 3-5에서 결정)
- Bulk bulk 동기/비동기 기준 건수 (구현 Phase에서 결정)
