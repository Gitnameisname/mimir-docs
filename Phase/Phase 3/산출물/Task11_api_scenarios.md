# Task 3-11. 대표 API 시나리오 설계

**Phase 3 설계 검증 문서**
**산출물**: `Task11_api_scenarios.md`
**선행 Tasks**: Task 3-1 ~ 3-10 (전체 Phase 3 API 설계 원칙 시리즈)

---

## 1. 문서 목적

이 문서는 Phase 3에서 정의한 API 설계 원칙들이 실제 사용 흐름에서 일관되게 작동하는지 검증하기 위한 **대표 API 시나리오 세트**를 정리한다.

새로운 규칙을 정의하는 것이 아니라, Task 3-1 ~ 3-10에서 확립한 기준들이 실전 흐름에서 충돌하지 않는지 점검하고, 구현 Phase로 넘기기 위한 기준선을 마련한다.

이 문서가 검증하는 내용:
- API 아키텍처 원칙, REST 리소스 구조, 인증/인가, 버전 관리, 공통 응답 포맷, pagination/filter/sort, idempotency, async/event/webhook, AI/RAG 인터페이스, 오류 모델이 실제 사용 흐름에서 서로 충돌하지 않는지
- User UI, Admin UI, 외부 시스템, AI agent, 운영 관점의 흐름이 하나의 API 계약 안에서 설명 가능한지
- 어떤 엔드포인트군이 구현 우선순위 상 먼저 필요한지

---

## 2. 대표 시나리오 설계가 필요한 이유

### 2-1. 개별 규칙 문서의 한계

Task 3-1 ~ 3-10은 각각 하나의 관점(응답 포맷, idempotency, AI 인터페이스 등)을 깊이 있게 다룬다. 그러나 실제 운영에서 API는 여러 규약이 동시에 작동한다.

- 문서를 생성하는 사용자는 인증 → 요청 포맷 → 리소스 구조 → 응답 포맷 → 오류 처리가 한 번에 맞물려야 한다.
- 한 규약이 다른 규약과 충돌하는 지점은 개별 문서로는 발견하기 어렵다.

### 2-2. 흐름 단위 설계 검증

API 설계는 엔드포인트 단위가 아니라 **사용자/시스템 작업 흐름 단위**로 검토해야 한다.

- "문서를 생성한다"는 단일 요청이 아니라, 생성 → 조회 → 권한 확인 → 오류 처리 → 감사 기록으로 이어지는 흐름이다.
- 흐름 단위 검증을 통해 누락된 리소스, 애매한 상태 전이, 과도한 복잡성, 미비한 오류 모델을 조기에 발견할 수 있다.

### 2-3. 구현 우선순위 기준선

이후 구현 Phase에서 무엇부터 만들지 우선순위를 정하는 데 시나리오 기반 분석이 필요하다.

**결론**: 대표 시나리오는 단순 예시가 아니라 설계 검증과 구현 인계 사이를 잇는 실전형 문서다.

---

## 3. 시나리오 작성 원칙

### 원칙 1. Scenario as Contract Validation, Not Feature Fiction

**의미**: 시나리오는 희망 사항이나 UI 스토리가 아니라, API 계약이 실제로 충족될 수 있는지를 검증하는 도구다.

**왜 필요한지**: "있으면 좋겠는 기능"을 나열하면 구현 우선순위가 흐려진다. 계약 검증에 집중하면 설계상 빈 틈이 드러난다.

**시나리오 결정 유도**: 각 시나리오는 특정 API 계약 조항(응답 포맷, 오류 코드, 권한 모델 등)을 반드시 언급하고 검증한다.

---

### 원칙 2. Happy Path + Failure Path 모두 포함

**의미**: 성공 흐름만이 아니라 실패 흐름도 시나리오에 포함한다.

**왜 필요한지**: 성공 흐름만 보면 오류 모델, 보안 처리, retry 정책이 설계에서 빠진다. 실제 운영은 실패 경로가 더 복잡한 경우가 많다.

**시나리오 결정 유도**: 모든 핵심 시나리오에 정상 흐름(happy path)과 적어도 하나의 실패 흐름(failure path)을 함께 작성한다.

---

### 원칙 3. Consumer-Aware Scenario Design

**의미**: User UI, Admin UI, 외부 시스템, AI agent, 운영자 관점을 균형 있게 포함한다.

**왜 필요한지**: 플랫폼의 소비자가 다양하며, 소비자마다 다른 API 패턴(동기/비동기, 구조 조회/전문 조회, 권한 범위)이 필요하다.

**시나리오 결정 유도**: 시나리오마다 주요 소비자를 명시하고, 해당 소비자의 관점에서 API 흐름을 작성한다.

---

### 원칙 4. Resource, Security, Response, Error, Observability를 함께 검증

**의미**: 한 시나리오에서 리소스 구조, 보안 컨텍스트, 응답 포맷, 오류 처리, 추적 정보를 함께 점검한다.

**왜 필요한지**: 각각을 따로 보면 일관성이 있어 보여도 함께 동작할 때 충돌이 발생할 수 있다.

**시나리오 결정 유도**: 각 시나리오의 검증 포인트 항목에 5가지 관점을 모두 포함한다.

---

### 원칙 5. Minimal but Representative Coverage

**의미**: 모든 엔드포인트의 모든 조합을 다루는 것이 아니라, 각 분류를 대표하는 최소 시나리오를 선택한다.

**왜 필요한지**: 과도한 시나리오는 문서 유지보수 비용이 높고 핵심을 흐린다.

**시나리오 결정 유도**: 각 범주에서 3~5개의 시나리오로 해당 범주의 핵심 설계 결정을 검증한다.

---

### 원칙 6. Cross-Task Consistency Validation

**의미**: 각 시나리오는 Task 3-1 ~ 3-10 중 어떤 설계 결정을 검증하는지 명시한다.

**왜 필요한지**: 설계 문서 간 연결이 명시되어야 구현 시 어떤 규약을 따라야 하는지 명확해진다.

**시나리오 결정 유도**: 각 시나리오의 검증 포인트에 검증 대상 Task를 명시한다.

---

### 원칙 7. Implementation-Oriented Abstraction

**의미**: 구현 코드나 SQL은 작성하지 않지만, 구현 팀이 어떤 API를 먼저 만들어야 하는지 판단할 수 있는 수준의 구체성을 유지한다.

**왜 필요한지**: 너무 추상적이면 구현 인계가 안 된다. 너무 구체적이면 설계 유연성이 제한된다.

**시나리오 결정 유도**: HTTP method, endpoint URI, 주요 필드를 포함하되, 스키마 전체나 코드는 작성하지 않는다.

---

## 4. 시나리오 분류 체계

| 분류 | 포함 시나리오 | 주요 소비자 | 검증 대상 Task |
|-----|------------|-----------|------------|
| A. Core document lifecycle | A, B, C | User UI, Admin UI | T3-2, T3-5, T3-6 |
| B. Access control / security | D, E, F | User UI, External | T3-3, T3-10 |
| C. Retry / conflict / failure | G, H, I, J | User UI, External, AI | T3-7, T3-10 |
| D. Async / event / webhook | K, L, M, N | External, Internal | T3-8, T3-10 |
| E. AI / RAG consumption | O, P, Q, R | AI Agent, RAG Pipeline | T3-9, T3-3 |
| F. Operations / observability / audit | S, T, U | Ops, Admin | T3-10, T3-8 |

---

## 5. 핵심 사용자 시나리오

---

### 시나리오 A — 문서 생성 → 초기 버전 생성 → 구조 조회

**주요 소비자**: User UI (편집자)

**목적**: 문서 생성의 기본 흐름에서 리소스 구조, 응답 포맷, 보안 컨텍스트, canonical reference가 일관되게 작동하는지 검증

**전제 조건**:
- 사용자가 인증된 상태 (Bearer token 보유)
- 해당 조직 내 문서 생성 권한 보유

**주요 리소스**: `documents`, `versions`, `nodes`

**단계별 흐름**:

```
1. 문서 생성
   POST /documents
   Authorization: Bearer {token}
   Idempotency-Key: {client-uuid}
   Body: { "title": "...", "document_type": "policy", "org_id": "org-abc", ... }

   → 201 Created
   { "data": { "document_id": "doc-123", "status": "draft", ... }, "meta": { "request_id": "req-001" } }

2. 초기 버전 생성
   POST /documents/doc-123/versions
   Body: { "label": "v1.0", "content": { ... } }

   → 201 Created
   { "data": { "version_id": "ver-1", "document_id": "doc-123", "status": "draft" }, "meta": { ... } }

3. 버전 구조 요약 조회
   GET /documents/doc-123/versions/ver-1/structure

   → 200 OK
   { "data": { "nodes": [ { "node_id": "n-1", "type": "section", "title": "1. 개요", "depth": 1, "child_count": 3 }, ... ] }, "meta": { ... } }

4. 특정 노드 조회
   GET /documents/doc-123/versions/ver-1/nodes/n-1

   → 200 OK
   { "data": { "node_id": "n-1", "type": "section", "title": "1. 개요", "content": "...", ... }, "meta": { ... } }
```

**핵심 API 규약 검증 포인트**:
- [T3-2] 리소스 계층: `documents` → `versions` → `nodes` 계층 구조가 자연스럽게 연결되는가?
- [T3-5] 응답 봉투: 생성 응답에 `data` + `meta` 포함되는가?
- [T3-7] Idempotency-Key: 생성 요청에 key가 적용되는가? 재시도 시 동일 응답인가?
- [T3-3] Security Context: `org_id`가 Security Context에서 자동 주입되는가, 요청 본문에 포함해야 하는가?
- canonical resource URI: 생성된 리소스의 `self` URI가 응답에 포함되는가?

**예상 오류 응답 포인트**:
- 권한 없는 org_id 지정 → 403 `auth.permission_denied`
- document_type 미지원 값 → 400 `validation.invalid_field`
- 버전 중복 생성 → 409 `resource.already_exists` 또는 `state.invalid_transition`

**관측성/감사 추적 포인트**:
- `meta.request_id`가 모든 응답에 포함되는가?
- 문서 생성 이벤트가 `document.created` 이벤트로 발행되는가?
- 감사 로그: actor_id, org_id, document_id, action=create 기록

**남아 있는 미결정 사항**:
- 초기 버전을 문서 생성과 동시에 생성할지, 분리할지 (API 설계 결정)
- `org_id`를 요청 본문에 포함할지 Security Context에서만 추출할지

---

### 시나리오 B — 문서 목록 조회 with Filter / Sort / Pagination

**주요 소비자**: User UI (목록 화면), RAG 파이프라인 (증분 동기화)

**목적**: 목록 조회에서 pagination, filter, sort 규약과 권한 기반 가시성이 일관되게 작동하는지 검증

**전제 조건**:
- 사용자가 인증된 상태
- 조직 org-abc 내 문서 읽기 권한 보유

**주요 리소스**: `documents`

**단계별 흐름**:

```
1. 첫 페이지 조회 (기본 조건)
   GET /documents?status=published&sort=-updated_at&page=1&page_size=20
   Authorization: Bearer {token}

   → 200 OK
   {
     "data": [ { "document_id": "...", "title": "...", "status": "published", ... }, ... ],
     "meta": {
       "request_id": "req-002",
       "pagination": { "page": 1, "page_size": 20, "total_count": 87, "has_next": true }
     }
   }

2. 다음 페이지 조회
   GET /documents?status=published&sort=-updated_at&page=2&page_size=20

3. RAG 파이프라인용 증분 조회 (cursor 기반)
   GET /documents?updated_after=2025-03-01T00:00:00Z&page_size=50
   → cursor pagination 적용

4. 빈 결과
   GET /documents?document_type=nonexistent_type
   → 200 OK, { "data": [], "meta": { "pagination": { "total_count": 0, ... } } }
```

**핵심 API 규약 검증 포인트**:
- [T3-6] pagination: `page` + `page_size` 기본, `updated_after` 필터와의 호환성
- [T3-6] filter: `status`, `document_type`, `updated_after` 필터 파라미터 규약
- [T3-6] sort: `-updated_at` (내림차순) 규약 준수
- [T3-3] 권한 기반 가시성: 응답 목록이 요청자 권한 범위 내 문서만 포함하는가?
- [T3-5] 빈 목록: `data: []` + pagination meta 정상 반환

**예상 오류 응답 포인트**:
- 지원하지 않는 sort 필드 → 400 `query.unsupported_sort`
- 잘못된 filter 문법 → 400 `query.invalid_filter`
- 만료된 cursor → 400 `query.invalid_cursor`

**관측성/감사 추적 포인트**:
- 목록 조회에 `request_id` 포함
- 대량 조회 요청에 rate limit 적용되는지

**남아 있는 미결정 사항**:
- `updated_after` 필터와 offset pagination의 결합 가능 여부
- 목록 응답에 포함할 필드 subset (전체 vs 요약 필드)

---

### 시나리오 C — 문서 수정 → 새 버전 생성 또는 상태 갱신

**주요 소비자**: User UI (편집자), Admin UI

**목적**: 문서 수정 흐름에서 update vs versioned change 구분, 상태 전이 action endpoint, observability 연결이 일관되게 작동하는지 검증

**전제 조건**:
- 사용자가 문서 편집 권한 보유
- 대상 문서가 draft 상태

**주요 리소스**: `documents`, `versions`, `operations`

**단계별 흐름**:

```
1. 문서 메타데이터 수정 (versioning 불필요한 변경)
   PATCH /documents/doc-123
   Body: { "title": "수정된 제목", "tags": ["정책", "개정"] }

   → 200 OK, { "data": { "document_id": "doc-123", "title": "수정된 제목", ... }, "meta": { "request_id": "req-003" } }

2. 새 버전 생성 (내용 변경)
   POST /documents/doc-123/versions
   Idempotency-Key: {uuid}
   Body: { "label": "v2.0", "based_on_version_id": "ver-1", "content": { ... } }

   → 201 Created, { "data": { "version_id": "ver-2", ... }, "meta": { ... } }

3. 버전 발행 (상태 전이)
   POST /documents/doc-123/versions/ver-2:publish

   → 202 Accepted (비동기) 또는 200 OK (동기)
   { "data": { "operation_id": "op-pub-001", ... }, "meta": { ... } }

4. 상태 확인
   GET /documents/doc-123/versions/ver-2
   → { "data": { "status": "published", ... } }
```

**핵심 API 규약 검증 포인트**:
- [T3-2] action endpoint: `:publish` 형식, 상태 전이가 action endpoint로 표현되는가?
- [T3-8] async/sync 경계: publish가 동기인지 비동기인지 결정되었는가?
- [T3-7] Idempotency: 새 버전 생성에 idempotency-key 적용
- [T3-5] PATCH 응답: 수정된 리소스 전체 반환 vs 변경 필드만 반환

**예상 오류 응답 포인트**:
- archived 문서에 버전 생성 시도 → 409 `state.invalid_transition`
- 이미 published 버전을 publish 재시도 → 409 `state.invalid_transition` 또는 멱등 성공
- `based_on_version_id`가 존재하지 않는 버전 → 422 또는 400

**관측성/감사 추적 포인트**:
- `document.version.published` 이벤트 발행
- 감사 로그: actor_id, document_id, version_id, action=publish, result=success

**남아 있는 미결정 사항**:
- publish가 동기 vs 비동기 (operation 반환 여부)
- PATCH 응답에 전체 리소스 반환 vs 변경 사항만 반환

---

## 6. 권한 / 보안 시나리오

---

### 시나리오 D — 권한 없는 사용자의 문서 조회 시도

**주요 소비자**: User UI (일반 사용자)

**목적**: 인증 vs 인가 오류 구분, 오류 shape, 정보 노출 제한이 API 레벨에서 올바르게 작동하는지 검증

**전제 조건**:
- 케이스 1: 토큰 없이 요청
- 케이스 2: 유효한 토큰이지만 해당 문서 접근 권한 없음

**단계별 흐름**:

```
케이스 1 — 인증 없음
   GET /documents/doc-123
   (Authorization 헤더 없음)

   → 401 Unauthorized
   { "error": { "code": "auth.authentication_required", "message": "인증이 필요합니다.", "retryable": false }, "meta": { "request_id": "req-D1" } }

케이스 2 — 권한 없음 (문서 존재하지만 접근 불가)
   GET /documents/doc-123
   Authorization: Bearer {valid_token_no_access}

   → 403 Forbidden (또는 정책에 따라 404)
   { "error": { "code": "auth.permission_denied", "message": "이 작업을 수행할 권한이 없습니다.", "retryable": false }, "meta": { "request_id": "req-D2" } }
```

**핵심 API 규약 검증 포인트**:
- [T3-3] authentication vs authorization: 401과 403이 올바르게 구분되는가?
- [T3-10] 오류 shape: 두 오류 모두 동일한 `{"error": {...}, "meta": {...}}` 봉투
- [T3-10] 보안 노출: 문서 존재 여부, ACL 세부 정보가 응답에 노출되지 않는가?
- [T3-10] 404 정책: 권한 없는 문서를 404로 반환하는 정책이 적용되는 경우, 일관성이 있는가?

**관측성/감사 추적 포인트**:
- 인증 실패: 감사 로그에 시도한 actor (가능한 범위), endpoint, result=denied 기록
- 권한 실패: 감사 로그에 actor_id, resource_id, action=read, result=denied 기록
- 보안 이상 탐지를 위해 반복 실패 패턴이 운영 로그에 추적 가능해야 함

**남아 있는 미결정 사항**:
- 이 도메인(문서 조회)에서 403 vs 404를 선택하는 정책 확정

---

### 시나리오 E — 다른 조직 문서 접근 시도

**주요 소비자**: External system, AI Agent

**목적**: 테넌트 경계 위반 시도가 API 레벨에서 차단되고 감사 추적되는지 검증

**전제 조건**:
- 사용자가 org-abc에만 접근 권한 보유
- org-xyz의 문서를 직접 id로 접근 시도

**단계별 흐름**:

```
1. 다른 조직 문서 직접 접근
   GET /documents/doc-xyz-999   ← org-xyz 소속 문서
   Authorization: Bearer {org-abc-token}

   → 403 또는 404
   { "error": { "code": "auth.permission_denied" 또는 "resource.not_found", "retryable": false }, "meta": { "request_id": "req-E1" } }

2. 검색에서 다른 조직 문서 포함 시도
   POST /searches
   Body: { "query": "...", "scope": { "org_ids": ["org-abc", "org-xyz"] } }

   → 403 또는 scope 자동 제한 (org-xyz 부분 무시)
   정책: 권한 없는 org_id가 scope에 포함되면 거부 또는 자동 필터
```

**핵심 API 규약 검증 포인트**:
- [T3-3] tenant scope enforcement: Security Context의 tenant 범위가 API 레벨에서 강제되는가?
- [T3-9] permission-preserving retrieval: 검색 scope에 권한 없는 org 포함 시 처리 방식
- [T3-10] 오류 code: `auth.tenant_scope_violation` 또는 `auth.permission_denied` 구분

**관측성/감사 추적 포인트**:
- 테넌트 경계 위반 시도는 반드시 감사 로그에 기록
- 반복 시도 패턴은 보안 알림 대상 (구현 Phase에서 확정)

**남아 있는 미결정 사항**:
- 검색 scope에 권한 없는 org 포함 시: 거부(403) vs 자동 필터(결과 제한) 정책 확정
- 403과 404 중 어느 것이 리소스 존재를 더 안전하게 숨기는지 도메인별 정책 확정

---

### 시나리오 F — 관리자 전용 리소스 접근

**주요 소비자**: Admin UI, Ops

**목적**: 관리자 권한이 필요한 리소스에서 일반 사용자와 관리자의 응답 차이가 API 레벨에서 올바르게 작동하는지 검증

**전제 조건**:
- 케이스 1: 일반 사용자가 관리자 전용 리소스 접근
- 케이스 2: 관리자가 동일 리소스 접근

**단계별 흐름**:

```
케이스 1 — 일반 사용자
   GET /admin/organizations/org-abc/members
   Authorization: Bearer {user_token}

   → 403 Forbidden
   { "error": { "code": "auth.forbidden_action", "message": "이 작업을 수행할 권한이 없습니다.", "retryable": false }, "meta": { ... } }

케이스 2 — 관리자
   GET /admin/organizations/org-abc/members
   Authorization: Bearer {admin_token}

   → 200 OK
   { "data": [ { "user_id": "...", "role": "editor", ... } ], "meta": { ... } }

노출 경계 검토:
   일반 API: GET /documents/doc-123 → 공개 가능 메타데이터만 반환
   관리자 API: GET /admin/documents/doc-123 → 내부 상태, 감사 메타데이터 포함 가능
```

**핵심 API 규약 검증 포인트**:
- [T3-2] 리소스 구조: `/admin/` 네임스페이스가 일반 API와 명확히 분리되는가?
- [T3-3] 역할 기반 접근: 역할(admin) 판단이 Security Context에서 이루어지는가?
- [T3-10] 노출 경계: 관리자 응답에 포함되는 추가 필드가 일반 응답과 명확히 구분되는가?

**남아 있는 미결정 사항**:
- `/admin/` 경로를 별도 API 버전으로 분리할지 (`/api/v1/admin/`) 검토

---

## 7. Idempotency / Conflict / 오류 처리 시나리오

---

### 시나리오 G — 문서 생성 요청 타임아웃 후 동일 Idempotency-Key로 재시도

**주요 소비자**: User UI, External system

**목적**: 네트워크 타임아웃 후 재시도 시 idempotency 보장이 실제로 작동하는지 검증

**전제 조건**:
- 최초 요청이 처리됐는지 여부가 클라이언트에 불명확 (타임아웃)
- 클라이언트가 동일한 Idempotency-Key로 재시도

**단계별 흐름**:

```
1. 최초 요청 (타임아웃)
   POST /documents
   Idempotency-Key: key-abc-001
   Body: { "title": "정책 v1", ... }
   → 네트워크 타임아웃 (서버에서는 처리 완료)

2. 동일 key로 재시도
   POST /documents
   Idempotency-Key: key-abc-001
   Body: { "title": "정책 v1", ... }   ← 동일한 payload

   → 201 Created (또는 200 OK)
   {
     "data": { "document_id": "doc-123", ... },
     "meta": {
       "request_id": "req-G2",
       "idempotency": { "replayed": true, "original_request_id": "req-G1" }
     }
   }

3. 결과 확인
   GET /documents/doc-123
   → 문서가 단 한 번만 생성됨을 확인
```

**핵심 API 규약 검증 포인트**:
- [T3-7] replay 응답: `meta.idempotency.replayed: true` + `original_request_id` 포함
- [T3-7] 단일 처리 보장: 동일 key 재요청이 중복 생성을 유발하지 않음
- [T3-5] 응답 shape: replay 응답도 동일한 봉투 구조

**관측성/감사 추적 포인트**:
- 원본 요청과 replay 요청이 모두 운영 로그에 기록되되, replay임이 표시됨
- 감사 로그: 생성은 최초 요청 기준으로 단 한 번 기록

**남아 있는 미결정 사항**:
- replay 기간(window) 수치 결정
- replay 응답 HTTP status: 201 vs 200 선택

---

### 시나리오 H — 같은 Idempotency-Key로 다른 Payload 전송

**주요 소비자**: User UI, External system (클라이언트 버그 또는 공격 시도)

**목적**: 동일 key + 다른 payload 전송 시 충돌이 안전하게 처리되는지 검증

**단계별 흐름**:

```
1. 최초 요청
   POST /documents
   Idempotency-Key: key-abc-002
   Body: { "title": "정책 A" }
   → 201 Created

2. 동일 key, 다른 payload
   POST /documents
   Idempotency-Key: key-abc-002
   Body: { "title": "완전히 다른 문서" }   ← 다른 payload

   → 409 Conflict
   {
     "error": {
       "code": "idempotency.key_mismatch",
       "message": "동일한 Idempotency-Key로 내용이 다른 요청이 이미 처리되었습니다.",
       "category": "idempotency",
       "retryable": false
     },
     "meta": {
       "request_id": "req-H2",
       "idempotency_key": "key-abc-002"
     }
   }
```

**핵심 API 규약 검증 포인트**:
- [T3-7] key_mismatch: 409 `idempotency.key_mismatch` 발생
- [T3-10] 안전한 실패: 두 번째 문서가 생성되지 않음
- [T3-10] meta.idempotency_key: 응답에 충돌한 key 반영

---

### 시나리오 I — 상태 전이 충돌 (Invalid State Transition)

**주요 소비자**: User UI, Admin UI

**목적**: 잘못된 상태 전이 요청이 올바른 오류로 처리되는지 검증

**케이스 1 — archived 문서에 publish 요청**:

```
POST /documents/doc-999/versions/ver-5:publish
← doc-999는 archived 상태

→ 409 Conflict
{
  "error": {
    "code": "state.invalid_transition",
    "message": "현재 상태(archived)에서 publish 전환은 허용되지 않습니다.",
    "category": "state",
    "retryable": false,
    "details": [
      { "code": "state.invalid_transition", "message": "archived 상태에서 허용된 전환: [restore]", "target": "status" }
    ]
  },
  "meta": { "request_id": "req-I1" }
}
```

**케이스 2 — 이미 published인 버전 publish 재요청**:

```
POST /documents/doc-123/versions/ver-2:publish
← ver-2가 이미 published 상태

→ 정책에 따라:
  Option A: 200 OK (멱등 처리, 이미 완료된 전이)
  Option B: 409 "state.invalid_transition" (이미 완료된 전이를 오류로 처리)
```

**핵심 API 규약 검증 포인트**:
- [T3-2] action endpoint: `:publish` action endpoint 의미
- [T3-10] 충돌 vs 전이 오류: `resource.conflict` vs `state.invalid_transition` 구분
- [T3-10] details: 허용된 전이 목록이 도움이 되는지

**남아 있는 미결정 사항**:
- "이미 같은 상태" 재요청을 멱등 성공으로 처리할지, 오류로 처리할지 정책 확정

---

### 시나리오 J — 잘못된 Filter / Invalid Cursor / Unsupported Sort

**주요 소비자**: User UI, External system, AI pipeline

**목적**: 잘못된 query 파라미터에 대한 structured 오류 처리 검증

```
케이스 1 — 지원하지 않는 sort 필드
   GET /documents?sort=-nonexistent_field
   → 400 { "error": { "code": "query.unsupported_sort", "message": "지원하지 않는 정렬 필드: nonexistent_field", "target": "sort", "retryable": false }, ... }

케이스 2 — 잘못된 cursor
   GET /documents?cursor=corrupted_cursor_value
   → 400 { "error": { "code": "query.invalid_cursor", "retryable": false }, ... }

케이스 3 — 잘못된 filter 문법
   GET /documents?status=published,invalid_status
   → 400 { "error": { "code": "query.invalid_filter", "message": "...", "target": "status", "retryable": false }, ... }
```

**핵심 API 규약 검증 포인트**:
- [T3-6] query 규약: filter/sort/cursor 파라미터 형식
- [T3-10] structured error: `target` 필드에 어느 파라미터가 잘못됐는지 명시
- [T3-10] machine-readable: AI pipeline이 `error.code`로 자동 분기 가능한가?

---

## 8. Async Operation / Event / Webhook 시나리오

---

### 시나리오 K — Export 요청 → Async Operation 생성 → 상태 Polling

**주요 소비자**: User UI, External system

**목적**: async/sync 경계, operation 리소스, 상태 전이, 완료 후 결과 참조가 일관되게 작동하는지 검증

**단계별 흐름**:

```
1. Export 시작
   POST /documents:export
   Idempotency-Key: key-export-001
   Body: { "document_ids": ["doc-123", "doc-456"], "format": "pdf" }

   → 202 Accepted
   {
     "data": { "operation_id": "op-exp-001", "status": "accepted", "type": "document.export" },
     "meta": { "request_id": "req-K1", "operation_id": "op-exp-001" }
   }

2. 상태 Polling
   GET /operations/op-exp-001

   → 200 OK
   { "data": { "operation_id": "op-exp-001", "status": "running", "progress": 0.6 }, "meta": { ... } }

3. 완료 확인
   GET /operations/op-exp-001

   → 200 OK
   {
     "data": {
       "operation_id": "op-exp-001",
       "status": "succeeded",
       "result": { "download_uri": "...", "expires_at": "..." }
     },
     "meta": { ... }
   }

4. 실패 시
   → 200 OK (polling 응답)
   {
     "data": {
       "operation_id": "op-exp-001",
       "status": "failed",
       "error": { "code": "async.operation_failed", "message": "...", "retryable": true }
     }
   }
```

**핵심 API 규약 검증 포인트**:
- [T3-8] 202 Accepted + operation_id 패턴
- [T3-8] operation 상태 전이: accepted → running → succeeded/failed
- [T3-7] Idempotency: 동일 key 재요청 시 같은 operation_id 반환 (중복 export 방지)
- [T3-10] operation 실패 오류: `async.operation_failed` + retryable 신호

**관측성/감사 추적 포인트**:
- operation 시작/완료 모두 운영 로그 기록
- operation_id가 요청 request_id와 연결되는가?

**남아 있는 미결정 사항**:
- export 결과 download_uri의 만료 시간 정책
- operation polling 권장 간격 안내 방식 (헤더? 응답 필드?)

---

### 시나리오 L — document.published 이벤트 발생 → Webhook Delivery

**주요 소비자**: External system (webhook 수신)

**목적**: 도메인 이벤트와 webhook delivery가 올바르게 연결되고, subscription 개념이 명확한지 검증

**단계별 흐름**:

```
1. Webhook 구독 설정 (선행)
   POST /webhook-subscriptions
   Body: {
     "target_url": "https://example.com/hooks",
     "subscribed_events": ["document.version.published"],
     "filters": { "org_ids": ["org-abc"] },
     "secret_reference": "secret-ref-abc"
   }
   → 201 Created, { "data": { "webhook_id": "wh-001", "status": "active" } }

2. 문서 발행 (이벤트 트리거)
   POST /documents/doc-123/versions/ver-2:publish
   → 성공 → document.version.published 이벤트 발행

3. Webhook Delivery (플랫폼 → 외부 서버)
   플랫폼이 target_url로 POST 요청:
   {
     "event_id": "evt-001",
     "event_type": "document.version.published",
     "schema_version": "1.0",
     "occurred_at": "...",
     "tenant_id": "tenant-A",
     "subject_type": "document_version",
     "subject_id": "ver-2",
     "payload": { "document_id": "doc-123", "version_id": "ver-2", ... }
   }
   X-Mimir-Signature: hmac-sha256=...

4. Delivery 상태 조회
   GET /webhook-subscriptions/wh-001/deliveries/del-001
   → { "data": { "delivery_id": "del-001", "status": "delivered", "attempt_count": 1 } }
```

**핵심 API 규약 검증 포인트**:
- [T3-8] 이벤트 → webhook 연결: 이벤트 발행 → 구독 필터 매칭 → delivery 생성
- [T3-8] HMAC signature: `X-Mimir-Signature` 헤더 포함
- [T3-8] at-least-once: delivery 실패 시 재시도 정책

---

### 시나리오 M — Webhook Delivery 실패 → Delivery 조회 → 재전송

**주요 소비자**: Admin UI, Ops

**목적**: delivery 실패 관측성과 재전송 action이 운영 관점에서 작동하는지 검증

**단계별 흐름**:

```
1. 외부 서버 응답 오류로 delivery 실패 (3회 재시도 후 permanently_failed)

2. Delivery 상태 조회
   GET /webhook-subscriptions/wh-001/deliveries/del-002
   → {
       "data": {
         "delivery_id": "del-002",
         "status": "permanently_failed",
         "attempt_count": 3,
         "last_error_summary": "HTTP 500 from target",
         "event_id": "evt-002"
       }
     }

3. 재전송 시도
   POST /webhook-subscriptions/wh-001/deliveries/del-002:resend

   → 202 Accepted 또는 200 OK (정책에 따라)

4. 연결된 이벤트 조회
   GET /events/evt-002
   → 해당 delivery를 유발한 이벤트 상세 확인
```

**핵심 API 규약 검증 포인트**:
- [T3-8] delivery 상태 가시성: `permanently_failed`, `attempt_count`, `last_error_summary`
- [T3-2] `:resend` action endpoint 패턴
- [T3-10] delivery 오류 정보: webhook.delivery_failed → delivery 리소스에 기록

**관측성/감사 추적 포인트**:
- `delivery_id` → `event_id` → `request_id` (원본 trigger) 추적 체인
- 운영자가 어느 이벤트가 배달 실패했는지 파악 가능해야 함

---

### 시나리오 N — Reindex 시작 → 동일 Idempotency-Key 재요청

**주요 소비자**: Internal indexing service, AI pipeline

**목적**: async operation과 idempotency가 결합될 때 중복 작업이 방지되는지 검증

**단계별 흐름**:

```
1. 인덱싱 요청
   POST /documents/doc-123/versions/ver-2:reindex
   Idempotency-Key: key-reindex-ver2

   → 202 Accepted, { "data": { "operation_id": "op-reindex-001" }, ... }

2. 동일 key로 재요청 (중복 트리거)
   POST /documents/doc-123/versions/ver-2:reindex
   Idempotency-Key: key-reindex-ver2

   → 202 Accepted (replay)
   {
     "data": { "operation_id": "op-reindex-001" },
     "meta": { "idempotency": { "replayed": true, "original_request_id": "..." } }
   }
   ← 동일 operation_id 반환, 새 작업 시작 안 함
```

**핵심 API 규약 검증 포인트**:
- [T3-7] async + idempotency 결합: 중복 operation 방지
- [T3-8] operation_id 재사용: replay 시 기존 operation_id 반환
- [T3-9] AI 처리 작업: Task 3-9의 operations 재사용 원칙 검증

---

## 9. AI / RAG 사용 시나리오

---

### 시나리오 O — 대화형 Assistant의 권한 범위 내 검색 → 노드 조회 → Citation 생성

**주요 소비자**: AI Agent (Delegated 모드, 사용자 위임)

**목적**: retrieval → source fetch → citation 체인이 permission-preserving 방식으로 작동하는지 검증

**단계별 흐름**:

```
1. Retrieval Query
   POST /searches
   Authorization: Bearer {delegated_user_token}
   Body: {
     "query": "개인정보 처리방침 보존 기간",
     "retrieval_unit": "node",
     "max_results": 5,
     "scope": { "statuses": ["published"] }
   }

   → 200 OK
   {
     "data": [
       {
         "score": 0.93,
         "snippet": "개인정보는 수집 목적 달성 후 5년간 보존한다...",
         "citation_ref": {
           "document_id": "doc-pp-001",
           "version_id": "ver-3",
           "node_id": "node-§4.2",
           "canonical_uri": "/documents/doc-pp-001/versions/ver-3/nodes/node-§4.2",
           "human_label": "개인정보처리방침 v3.0 §4.2"
         },
         "context": { "document_title": "개인정보처리방침", "version_label": "v3.0", "node_title": "§4.2 보존기간" }
       }
     ],
     "meta": { "request_id": "req-O1", "search_id": "srch-001" }
   }

2. 원문 노드 조회 (follow-up fetch)
   GET /documents/doc-pp-001/versions/ver-3/nodes/node-§4.2
   Authorization: Bearer {delegated_user_token}

   → 200 OK
   { "data": { "node_id": "node-§4.2", "content": "...", "type": "section", ... }, "meta": { ... } }

3. Citation 정보로 Assistant 응답 생성
   출처: citation_ref.human_label + canonical_uri
```

**핵심 API 규약 검증 포인트**:
- [T3-9] retrieval_unit=node: 노드 단위 결과 반환
- [T3-9] citation_ref: document_id + version_id + node_id + canonical_uri 포함
- [T3-3] permission-preserving: 사용자 권한 밖 문서가 검색 결과에 포함되지 않음
- [T3-5] tool-friendly shape: `citation_ref`, `context`, `score`가 명확히 분리됨

**관측성/감사 추적 포인트**:
- 검색 요청: request_id, actor_id (delegated), search_id
- 원문 조회: request_id, document_id, version_id, node_id

---

### 시나리오 P — 외부 RAG 시스템의 문서 구조 조회 및 인덱싱

**주요 소비자**: External RAG pipeline (Service Account 인증)

**목적**: version-aware access, structured content access, 메타데이터 활용이 외부 시스템 관점에서 자연스럽게 작동하는지 검증

**단계별 흐름**:

```
1. 변경된 문서 파악 (증분 동기화)
   GET /documents?org_id=org-abc&updated_after=2025-03-01T00:00:00Z&page_size=50
   Authorization: Bearer {service_account_token}

   → 문서 목록 (변경된 것만)

2. 특정 문서 최신 버전 구조 조회
   GET /documents/doc-123/versions/latest/structure

   → { "data": { "nodes": [ { "node_id": "n-1", "type": "section", "title": "...", "child_count": 3 }, ... ] } }

3. 섹션별 노드 목록 조회
   GET /documents/doc-123/versions/ver-2/nodes?node_type=section&page_size=100

   → 섹션 노드 목록 (본문 없이 구조만)

4. 각 섹션 노드 단건 조회 (본문 포함)
   GET /documents/doc-123/versions/ver-2/nodes/n-1
   → 본문 콘텐츠 + 메타데이터

5. AI 처리 상태 확인
   GET /documents/doc-123/ai-status
   → { "indexing": { "status": "indexed", "last_indexed_version_id": "ver-2" } }
```

**핵심 API 규약 검증 포인트**:
- [T3-9] structured access: 구조 조회 → 노드 목록 → 노드 단건 3단계
- [T3-9] version-aware: `versions/latest` 와 `versions/{versionId}` 구분
- [T3-3] Service Account: 조직 범위로 제한된 service account 인증 작동
- [T3-6] cursor pagination: 대량 노드 목록 조회에 cursor 적용 가능한가?

---

### 시나리오 Q — 문서 변경 → Indexing Operation → Retrieval 가능 상태 반영

**주요 소비자**: Internal indexing service, AI pipeline

**목적**: 문서 변경부터 retrieval 가능 상태까지의 lifecycle이 API로 추적 가능한지 검증

**단계별 흐름**:

```
1. 문서 버전 발행 → 이벤트 발행
   POST /documents/doc-123/versions/ver-3:publish
   → document.version.published 이벤트

2. Indexing service가 이벤트 수신 → reindex 시작
   POST /documents/doc-123/versions/ver-3:reindex
   → 202 Accepted, operation_id: op-reindex-003

3. Indexing 진행 중 retrieval 시도
   POST /searches { "query": "..." }

   → (ver-3가 아직 인덱싱 안 된 경우 ver-2 결과 반환)
   또는
   → 503 ai.indexing_not_ready (정책에 따라)

4. Indexing operation 완료 확인
   GET /operations/op-reindex-003
   → { "status": "succeeded" }

5. AI 처리 상태 확인
   GET /documents/doc-123/ai-status
   → { "indexing": { "status": "indexed", "last_indexed_version_id": "ver-3" } }

6. 이제 검색에서 ver-3 내용 반영됨
```

**핵심 API 규약 검증 포인트**:
- [T3-9] indexing lifecycle: 발행 → 인덱싱 → retrieval 가능 상태의 API 추적성
- [T3-8] event → operation 연결: 이벤트 수신 → operation 시작
- [T3-10] ai.indexing_not_ready: 인덱싱 중 검색 요청 오류 처리

**남아 있는 미결정 사항**:
- 인덱싱 중 검색 요청 시 이전 버전 결과 반환 vs 503 반환 정책 확정

---

### 시나리오 R — AI Tool이 Citation Identifier를 따라 원문 노드 재조회

**주요 소비자**: AI Agent (tool-using)

**목적**: stable reference 기반 follow-up fetch 체인이 tool-friendly하게 작동하는지 검증

**단계별 흐름**:

```
1. 검색 결과에서 citation_ref 추출
   POST /searches { "query": "..." }
   → citation_ref.canonical_uri = "/documents/doc-123/versions/ver-2/nodes/node-§3.1"

2. canonical_uri로 원문 노드 조회
   GET /documents/doc-123/versions/ver-2/nodes/node-§3.1?include=context

   → {
       "data": {
         "node_id": "node-§3.1",
         "type": "section",
         "title": "§3.1 처리 목적",
         "content": "...",
         "context": {
           "parent_node_id": "node-§3",
           "parent_title": "§3 개인정보 처리",
           "prev_sibling_title": null,
           "next_sibling_title": "§3.2 수집 항목"
         }
       },
       "meta": { "request_id": "req-R2" }
     }

3. AI가 인용 포함 응답 생성
   "개인정보처리방침 v2.0 §3.1에 따르면..."
   출처: canonical_uri, human_label
```

**핵심 API 규약 검증 포인트**:
- [T3-9] stable reference: canonical_uri가 버전 고정 URI로 안정적
- [T3-9] ?include=context: 주변 노드 컨텍스트 포함 옵션
- [T3-5] tool-friendly shape: agent가 파싱 없이 citation 정보를 사용할 수 있는가?

---

## 10. 운영 / 관측성 / 감사 추적 시나리오

---

### 시나리오 S — 실패한 요청을 request_id로 추적

**주요 소비자**: Ops, 고객 지원팀

**목적**: 클라이언트가 받은 request_id로 서버 측 로그와 연결하여 실패 원인을 파악할 수 있는지 검증

**단계별 흐름**:

```
1. 클라이언트가 실패 응답 수신
   POST /documents/doc-123/versions
   → 409 Conflict
   {
     "error": { "code": "state.invalid_transition", "message": "...", "retryable": false },
     "meta": { "request_id": "req-S1", "timestamp": "2025-03-15T10:30:00Z" }
   }

2. 클라이언트가 request_id=req-S1을 지원팀에 보고

3. 운영팀이 내부 도구로 조회
   내부 로그 검색: request_id=req-S1
   → endpoint, actor_id, tenant_id, error_detail, elapsed_ms, trace_id 확인

4. 감사 로그 조회
   감사 로그: request_id=req-S1 → actor_id, resource_id, action, result=denied
```

**핵심 API 규약 검증 포인트**:
- [T3-10] request_id: 모든 응답(성공/오류)에 request_id 포함
- [T3-10] 3계층 연결: API 오류 응답 → 운영 로그 → 감사 로그가 request_id로 연결
- [T3-10] 보안 노출: 운영 로그에는 더 많은 정보가 있지만, API 응답에는 최소 정보만

**관측성 검증**:
- request_id가 실제로 내부 로그 시스템과 연결 가능한가?
- trace_id와 request_id의 관계가 명확한가?

---

### 시나리오 T — 누가 문서를 Publish 했는지 추적

**주요 소비자**: Admin UI, 감사 담당자

**목적**: actor context, 도메인 이벤트, 감사 로그가 연결되어 누가 무엇을 언제 했는지 추적 가능한지 검증

**단계별 흐름**:

```
1. 편집자가 문서 발행
   POST /documents/doc-123/versions/ver-2:publish
   Authorization: Bearer {editor_token}
   → 202 Accepted, operation_id: op-pub-001, request_id: req-T1

2. 이벤트 발행
   document.version.published 이벤트:
   { "event_id": "evt-pub-001", "actor_id": "user-editor-abc", "subject_id": "ver-2", ... }

3. 감사 로그 기록
   actor_id: user-editor-abc
   action: document.version.publish
   resource_type: document_version
   resource_id: ver-2
   result: success
   request_id: req-T1
   timestamp: ...

4. 관리자가 감사 조회
   "ver-2는 누가 언제 발행했는가?"
   → 감사 로그에서 actor_id=user-editor-abc, timestamp 확인
   → 이벤트 evt-pub-001에서 상세 컨텍스트 확인

5. 추적 체인
   request_id(req-T1) → operation_id(op-pub-001) → event_id(evt-pub-001)
```

**핵심 API 규약 검증 포인트**:
- [T3-8] 이벤트에 actor_id 포함: `actor_id`가 이벤트 페이로드에 포함되는가?
- [T3-10] 감사 로그 대상: publish는 감사 로그 필수 대상인가?
- [T3-8] 추적 체인: request_id → operation_id → event_id 연결이 실제로 추적 가능한가?

---

### 시나리오 U — 실패한 Webhook Delivery와 관련 이벤트 조사

**주요 소비자**: Admin UI, Ops

**목적**: webhook delivery 실패를 운영자가 API로 조사하고 원인을 파악할 수 있는지 검증

**단계별 흐름**:

```
1. 운영자가 webhook 실패 알림 수신

2. 실패 delivery 목록 조회
   GET /webhook-subscriptions/wh-001/deliveries?status=failed

   → delivery 목록 (del-002, del-003 등)

3. 특정 delivery 상세 조회
   GET /webhook-subscriptions/wh-001/deliveries/del-002
   → {
       "delivery_id": "del-002",
       "event_id": "evt-002",
       "status": "permanently_failed",
       "attempt_count": 3,
       "last_error_summary": "HTTP 503 from target",
       "last_attempt_at": "2025-03-15T10:00:00Z"
     }

4. 연결된 이벤트 조회
   GET /events/evt-002
   → { "event_type": "document.version.published", "occurred_at": "...", "subject_id": "ver-5", ... }

5. 이벤트를 유발한 원본 요청 추적
   이벤트의 correlation_id → 원본 publish 요청의 request_id 확인

6. 재전송 결정
   POST /webhook-subscriptions/wh-001/deliveries/del-002:resend
```

**핵심 API 규약 검증 포인트**:
- [T3-8] delivery 관측성: 실패 delivery를 API로 조회 가능한가?
- [T3-8] event → delivery 연결: event_id → delivery_id 연결이 명확한가?
- [T3-10] webhook_delivery_id: 오류 meta에 delivery_id 포함 가능한가?
- [T3-10] 추적 체인: delivery_id → event_id → request_id(원본 trigger)

---

## 11. 시나리오별 공통 검증 포인트

모든 시나리오에서 다음 항목을 공통으로 점검한다.

| 검증 항목 | 확인 내용 |
|---------|---------|
| 리소스 구조 | 리소스 계층이 자연스럽게 연결되는가? (T3-2) |
| Security Context | actor/tenant/scope가 Security Context에서 자동 주입되는가? (T3-3) |
| 요청/응답 포맷 | 성공: `data`+`meta`. 오류: `error`+`meta`. 일관성 유지 (T3-5) |
| Pagination/Filter/Sort | 목록 조회에 규약이 자연스럽게 적용되는가? (T3-6) |
| Idempotency | 변경 요청에 Idempotency-Key 적용 가능한가? replay가 안전한가? (T3-7) |
| 오류 모델 | error.code + retryable + meta.request_id 일관 적용 (T3-10) |
| 추적 포인트 | request_id → operation_id → event_id → delivery_id 체인 (T3-8, T3-10) |
| AI 재사용성 | AI 소비자도 같은 계약을 재사용할 수 있는가? (T3-9) |
| 보안 노출 | 오류 응답에 내부 정보가 과도하게 노출되지 않는가? (T3-10) |
| 감사 추적 | 민감 action이 감사 로그에 기록되는가? (T3-3, T3-10) |

---

## 12. 시나리오별 미결정 항목 및 후속 설계 포인트

### Phase 3에서 결정된 사항 (구현 준비 완료)

- 오류 응답 봉투 구조 (`error` + `meta`)
- request_id 포함 원칙 (모든 응답)
- idempotency-key replay 기본 동작
- 202 Accepted + operation_id 패턴
- at-least-once webhook delivery + HMAC signature
- permission-preserving retrieval 원칙
- citation_ref 구조체 개념

### 구현 Phase에서 확정 필요한 사항

| 미결정 항목 | 관련 시나리오 | 우선순위 |
|----------|-----------|--------|
| 초기 버전을 문서 생성과 동시에 생성할지 여부 | A | 높음 |
| publish action의 동기 vs 비동기 결정 | C | 높음 |
| 403 vs 404 도메인별 정책 확정 | D, E | 높음 |
| 인덱싱 중 검색 요청 처리 정책 | Q | 중간 |
| 검색 scope에 권한 없는 org 포함 시 처리 | E | 중간 |
| replay HTTP status: 201 vs 200 | G | 낮음 |
| operation polling 권장 간격 안내 방식 | K | 낮음 |
| citation_ref 포맷 확정 (anchor 표현 등) | O, R | 중간 |
| node_type 목록 확정 | P, R | 중간 |
| export 결과 download_uri 만료 시간 | K | 낮음 |

### MVP에서 축소 가능한 시나리오

| 시나리오 | 축소 방향 |
|---------|---------|
| M (webhook resend) | MVP에서 수동 재전송 UI 없이 자동 재시도만 제공 가능 |
| Q (indexing lifecycle) | MVP에서 ai-status API 없이 operation tracking만 제공 가능 |
| R (context include) | MVP에서 `?include=context` 옵션 없이 기본 노드 조회만 제공 가능 |
| U (delivery 조사) | MVP에서 Admin UI 없이 로그 기반 조사 가능 |

### MVP에서 반드시 포함해야 할 시나리오

시나리오 A, B, C, D, G, K, O는 MVP 범위. 구체적으로:
- 문서 생성/조회/목록 (A, B)
- 기본 인증/권한 오류 처리 (D)
- idempotency (G)
- async operation 기본 패턴 (K)
- 검색 + citation_ref 기본 (O)

---

## 13. 구현 우선순위 제안

이 우선순위는 구현 일정이 아니라 Phase 3 산출물을 다음 Phase 구현에 어떻게 연결할지 보여주는 인계 프레임이다.

### Priority 1 — MVP 핵심

**대상 시나리오**: A, B, C, D

**구현 필요 API 군**:
- `POST /documents`, `GET /documents`, `GET /documents/{id}`
- `POST /documents/{id}/versions`, `GET /documents/{id}/versions/{vid}`
- `POST /documents/{id}/versions/{vid}:publish`
- 공통 인증/권한 미들웨어 (Security Context)
- 공통 오류 응답 봉투 (exception handler)
- request_id 미들웨어

**목표**: 편집자가 문서를 만들고 발행하며, 목록을 조회할 수 있는 최소 기능

---

### Priority 2 — 안정성 / 운영성

**대상 시나리오**: G, H, I, J, S

**구현 필요 API 군**:
- Idempotency-Key 처리 미들웨어
- 공통 오류 code 체계 (`category.specific_code`)
- `query.invalid_filter`, `query.unsupported_sort` 등 query 오류 처리
- 운영 로그 correlation (request_id + trace_id)
- 감사 로그 파이프라인 (publish, delete 등 민감 action)

**목표**: 실패 경로가 안전하게 처리되고, 운영/감사 추적이 가능

---

### Priority 3 — 확장성

**대상 시나리오**: E, F, K, L, M, T, U

**구현 필요 API 군**:
- `POST /documents:export` + `GET /operations/{id}` (async operation)
- `POST /webhook-subscriptions`, `GET /webhook-subscriptions/{id}/deliveries`
- 이벤트 발행 파이프라인 (domain events)
- webhook delivery 엔진 + retry
- `/admin/` 네임스페이스 기본 API

**목표**: 외부 시스템과 연동 가능하고, 비동기 작업을 추적할 수 있음

---

### Priority 4 — AI-Ready 확장

**대상 시나리오**: N, O, P, Q, R

**구현 필요 API 군**:
- `GET /documents/{id}/versions/{vid}/structure`
- `GET /documents/{id}/versions/{vid}/nodes`
- `GET /documents/{id}/versions/{vid}/nodes/{nodeId}`
- `POST /searches` (retrieval 결과 + citation_ref)
- `GET /documents/{id}/ai-status`
- `POST /documents/{id}/versions/{vid}:reindex`

**목표**: AI/RAG 파이프라인이 공식 API만으로 문서를 구조적으로 접근하고 인용할 수 있음

---

## 14. 결론

이 문서에서 검증한 핵심 결과는 다음과 같다.

**설계 일관성 확인**: Task 3-1 ~ 3-10에서 정의한 규약들이 21개의 시나리오에서 충돌 없이 연결된다. 공통 응답 봉투, request_id 추적, idempotency, 오류 코드 체계, async operation, AI citation 체인이 하나의 계약 안에서 일관되게 작동한다.

**발견된 미결정 사항**: publish 동기/비동기, 403 vs 404 정책, 인덱싱 중 검색 처리 방식, citation_ref 포맷 등이 구현 Phase에서 확정이 필요한 사항으로 식별되었다.

**구현 인계 기준**: Priority 1 ~ 4 분류로 어떤 API 군을 먼저 구현해야 하는지 명확한 기준을 제공한다. Priority 1만으로 편집자의 기본 작업 흐름이 가능하고, Priority 4까지 완성되면 AI/RAG 연계가 공식 API만으로 지원된다.

---

## 자체 점검 요약

### 포함한 시나리오 범주 요약
- 6개 범주, 21개 시나리오 (A~U)
- Core lifecycle 3개 / Security 3개 / Retry·conflict·error 4개 / Async·event·webhook 4개 / AI·RAG 4개 / Ops·observability 3개

### MVP 핵심 시나리오 요약
- 시나리오 A (문서 생성·조회), B (목록 조회), C (수정·발행), D (권한 오류)
- Priority 1: `documents`, `versions` CRUD + 인증/권한 + 공통 오류 봉투 + request_id

### 보안/오류/Idempotency 검증 시나리오 요약
- G: 타임아웃 후 동일 key 재시도 → replay 보장
- H: key mismatch → 409 idempotency.key_mismatch
- I: 상태 전이 충돌 → 409 state.invalid_transition + details
- J: 잘못된 query 파라미터 → 400 query.* structured error
- D/E: 401/403/404 구분 + 테넌트 경계 강제

### Async/Event/Webhook 시나리오 요약
- K: export → 202 + operation → polling → 결과
- L: 이벤트 발행 → webhook delivery → HMAC 서명
- M: delivery 실패 → 조회 → resend action
- N: reindex + idempotency 결합 → 중복 operation 방지

### AI/RAG 시나리오 요약
- O: 검색 → citation_ref → 노드 조회 체인 (Delegated 모드)
- P: 외부 RAG 시스템의 구조 조회 + 증분 동기화
- Q: 발행 → 인덱싱 lifecycle → retrieval 가능 상태 추적
- R: citation canonical_uri follow-up fetch + context 포함

### Observability/Audit 시나리오 요약
- S: request_id → 운영 로그 → 감사 로그 3계층 연결
- T: actor_id → 이벤트 → 감사 로그 추적 체인 (publish 감사)
- U: webhook delivery_id → event_id → 원본 request_id 추적

### 구현 우선순위 제안 요약
- Priority 1 (MVP): 문서 CRUD + 인증/권한 + 오류 봉투
- Priority 2 (안정성): Idempotency + query 오류 + 감사 로그
- Priority 3 (확장성): async operation + event + webhook
- Priority 4 (AI-ready): structure/nodes API + retrieval + citation + ai-status

### 후속 Task 연결 포인트
- **Task 3-12**: 이 시나리오 세트를 기반으로 Phase 3 전체 통합 정리 및 구현 Phase 인계 문서 작성
- **구현 Phase**: Priority 1 → 2 → 3 → 4 순으로 API 군 구현

### 의도적으로 후속 단계로 남긴 미결정 사항
- publish action 동기 vs 비동기 정책 확정
- 403 vs 404 도메인별 정책 확정
- 인덱싱 중 검색 처리 정책 (이전 버전 결과 반환 vs 503)
- citation_ref anchor 포맷 확정
- operation polling 권장 간격 안내 방식
- replay HTTP status (201 vs 200) 결정
- node_type 목록 확정
- 검색 scope에 권한 없는 org 포함 시 처리 정책
