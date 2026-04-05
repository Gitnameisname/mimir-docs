# Phase 2 - Task 2-4. API 레벨 Authorization Enforcement 설계

---

## 1. 목표

Task 2-3에서 설계한 RBAC + ACL 권한 모델을 실제 API 요청 흐름에 연결하여, **API 요청이 들어왔을 때 어디서, 어떤 순서로, 어떤 책임 분리 하에 권한 검사를 수행할 것인지**를 확정한다.

- Authorization enforcement 위치와 계층별 책임을 정의한다
- API 요청 유형별 권한 검사 흐름을 정리한다
- Pre-check / Post-load check 원칙을 결정한다
- List/Search 권한 처리 및 오류 응답 정책을 정의한다
- Authorization Context 공통 모델을 확정한다
- Task 2-5 감사 로그 설계의 직접 입력 자료를 준비한다

---

## 2. 설계 전제

| # | 전제 |
|---|---|
| 1 | Task 2-3 결과: RBAC 1차 → Resource ACL 2차 → 상속 ACL 3차 → default deny |
| 2 | Task 2-2 결과: Scope = Platform → Organization → Space → Document → Version → Node/Sub-resource |
| 3 | API-first 플랫폼. UI, 외부 앱, AI 도구, 서비스 계정 모두 동일한 enforcement 원칙 적용 |
| 4 | 인증(Authentication)과 권한(Authorization)은 분리되어 있으나 연속된 흐름 안에서 동작 |
| 5 | 권한 판단 로직은 중앙화. 중복이나 파편화를 방지 |
| 6 | 감사 가능성 필수. allow/deny 결정은 사후 설명 가능한 metadata를 남긴다 |

---

## 3. Authorization Enforcement 위치와 책임

### 3-1. 계층별 배치 방식 비교

**Router/Controller에서 전부 처리하면?**
- 리소스를 아직 로드하지 않은 상태이므로 리소스 상태(draft, archived)나 소유자 기반 예외 적용 불가
- 엔드포인트마다 권한 로직이 복사되어 파편화 심각
- 비즈니스 로직 변경 시 권한 체크 누락 발생 위험

**Service Layer에서만 처리하면?**
- 비즈니스 유스케이스와 권한 판단이 혼재 → 단위 테스트 어려움
- 권한 로직이 여러 서비스에 흩어져 일관성 유지 어려움

**Repository까지 권한 판단이 내려가면?**
- 쿼리 레벨에서 데이터를 숨기는 효과는 있으나, "왜 거부됐는가"의 이유를 상위 계층에 전달하기 어려움
- 감사 로그에 decision reason을 남기기 어려움
- 테스트 복잡도 급증

**중앙 Authorization Service를 두는 방식:**
- 장점: 권한 판단 로직 단일화. 모든 계층에서 동일 서비스 호출 → 일관성 보장. 감사 로그와 자연스럽게 연결
- 단점: 모든 요청에 Authorization Service 호출 비용 발생. 캐싱 전략 필요

### 3-2. 권장 계층 구조

```
HTTP Request
    │
    ▼
[Authentication Middleware]
  └─ 토큰 검증 → Principal 식별 → request context에 주입
    │
    ▼
[Router / Controller]
  └─ 입력 파싱, resource identity 수집, Service 호출
    │
    ▼
[Service / Application Layer]           ◄── Authorization Service 호출 지점
  └─ 유스케이스 오케스트레이션
  └─ 리소스 로드 (필요 시)
  └─ Authorization Service 호출
  └─ 비즈니스 로직 실행
  └─ 감사 이벤트 생성 트리거
    │
    ▼
[Authorization Service]                 ◄── 권한 판단 전담
  └─ RBAC 평가 (Role → Permission)
  └─ ACL 평가 (direct → inherited)
  └─ allow/deny + reason 반환
    │
    ▼
[Repository / Data Access]
  └─ 순수 데이터 조회/저장
  └─ List filtering 지원 (authorization-aware query)
```

### 3-3. 계층별 책임 요약

| 계층 | 책임 | 하지 않는 것 |
|---|---|---|
| **Authentication Middleware** | 토큰 검증, Principal 식별, request context 주입 | 권한 판단 |
| **Router / Controller** | 경로 파싱, resource identity 수집, 입력 유효성 검사, Service 호출 | 직접 권한 판단 |
| **Service Layer** | 유스케이스 오케스트레이션, Authorization Service 호출, 리소스 로드, 감사 트리거 | 권한 평가 로직 직접 구현 |
| **Authorization Service** | RBAC + ACL 평가, allow/deny + reason 반환, decision context 생성 | 비즈니스 로직, DB 직접 접근 |
| **Repository** | 데이터 조회/저장, list authorization-aware query 지원 | 권한 판단, 비즈니스 규칙 |

---

## 4. API 요청 처리 흐름 초안

### 4-1. Create 요청 (POST /spaces/{space_id}/documents)

```
1. Authentication Middleware
   → user_id, token scope 확인

2. Controller
   → path에서 space_id 추출
   → request body 파싱

3. Service Layer
   → Space 존재 여부 확인 (Repository)
   → Authorization Service 호출:
      { principal: membership, resource: space/{space_id},
        action: "space:create_document" }
   → deny → 403 반환 + AuditDecision 기록
   → allow → Document 생성 로직 수행
   → 생성 완료 후 AuditEvent("document.created") 기록

action 매핑: POST /spaces/{id}/documents → space:create_document
검사 시점: pre-check (Space 권한 기반. Document 미존재)
감사 포인트: 생성 성공/실패, 생성자, space_id, document_id
```

### 4-2. Single Resource Read 요청 (GET /documents/{document_id})

```
1. Authentication Middleware → Principal 확인

2. Controller → document_id 추출

3. Service Layer
   → Document 로드 (Repository)
   → 리소스 상태 확인 (status, organization_id, space_id)
   → Authorization Service 호출:
      { principal: membership, resource: document/{id},
        action: "document:read", resource_state: status }
   → deny → 403 또는 404 (존재 은닉 정책에 따라)
   → allow (draft) → document:read_draft 추가 확인
   → 결과 반환

action 매핑: GET /documents/{id} → document:read (published) / document:read_draft (draft)
검사 시점: post-load check (리소스 상태, ACL 반영 필요)
감사 포인트: deny 시 reason 기록 필수
```

### 4-3. Update 요청 (PUT /documents/{document_id})

```
1. Authentication Middleware → Principal 확인

2. Controller → document_id, request body 추출

3. Service Layer
   → Document 로드
   → 상태 확인 (locked, archived 상태면 편집 불가 사전 체크)
   → Authorization Service 호출:
      { principal: membership, resource: document/{id},
        action: "document:update", resource_state: status }
   → deny → 403
   → allow → 새 Version 생성 로직 수행
   → AuditEvent("document.updated") 기록

action 매핑: PUT /documents/{id} → document:update
검사 시점: post-load check (상태 반영 필수)
주의: update는 새 Version 생성임. Version 불변 원칙 확인
감사 포인트: 변경 전 version_id, 변경 후 version_id
```

### 4-4. Delete 요청 (DELETE /documents/{document_id})

```
1~2. 동일 패턴

3. Service Layer
   → Document 로드
   → Authorization Service 호출:
      { action: "document:delete" }
   → deny → 403
   → allow → 소프트 삭제 (deleted_at 설정)
   → AuditEvent("document.deleted") 기록 필수

action 매핑: DELETE /documents/{id} → document:delete
검사 시점: post-load check
감사 포인트: 삭제자, document_id, 삭제 전 status (복구 가능성 판단용)
```

### 4-5. List/Search 요청 (GET /spaces/{space_id}/documents)

```
1~2. 동일 패턴

3. Service Layer
   → Space 접근 권한 pre-check:
      { action: "space:read" } → deny → 403
   → allow → Repository에 authorization-aware query 전달:
      { organization_id, membership_id, roles }
   → Repository에서 접근 가능한 Document만 반환
   → 결과 반환 (접근 불가 항목은 목록에서 제외)

검사 시점: pre-check(Space) + authorization-aware query(Document 목록)
감사 포인트: 목록 조회 자체는 감사 기록 선택. deny 시 필수
```

### 4-6. Admin Action 요청 (POST /organizations/{org_id}/members/{user_id}/roles)

```
3. Service Layer
   → Organization 로드
   → Authorization Service 호출:
      { action: "org:manage_members", resource: organization/{id} }
   → deny → 403
   → allow → 역할 변경 로직
   → AuditEvent("membership.role_changed") 기록 필수
      (변경 전/후 역할, 변경자, 대상자 포함)

검사 시점: post-load check
감사 포인트: 모든 권한 변경 전량 감사 필수
```

### 4-7. Nested Resource 요청 (GET /documents/{doc_id}/versions/{version_id})

```
3. Service Layer
   → Document 로드 → document:read 권한 확인
   → Version 로드 → Version status 확인
   → draft version이면 document:read_draft 추가 확인
   → allow → Version + Node 트리 반환

검사 시점: 상위 리소스(Document) post-load + 하위 리소스(Version) 상태 추가 규칙
원칙: 상위 리소스 권한을 먼저 확인하고, 하위 리소스의 추가 규칙을 적용
```

---

## 5. Pre-check / Post-load Check 원칙

### 5-1. 비교

| 항목 | Pre-check | Post-load Check |
|---|---|---|
| **수행 시점** | 리소스 로드 전 | 리소스 로드 후 |
| **장점** | 불필요한 DB 조회 방지. 빠름 | 리소스 상태, 소유자, 직접 ACL 반영 가능 |
| **단점** | 리소스 상태, 직접 ACL 반영 불가 | 리소스 조회 비용 발생 |
| **적합한 경우** | Create, 상위 컨테이너 접근 확인 | Read/Update/Delete, 상태 기반 제어 필요 시 |

### 5-2. 핵심 질문 답변

**Q. Create는 어떤 기준으로 체크해야 하는가?**
> 생성 대상 리소스는 아직 존재하지 않으므로, **상위 컨테이너 pre-check**만 수행한다.
> 예: Document 생성 → Space의 `space:create_document` 권한 확인.

**Q. Document read/update/delete는 언제 post-load가 필요한가?**
> 항상 post-load. Document의 `status`(draft/published/archived/locked)에 따라 접근 가능한 역할이 다르며, 리소스별 직접 ACL이 존재할 수 있다.

**Q. Nested resource는 상위/하위 리소스를 모두 봐야 하는가?**
> 예. 반드시 상위 리소스 권한을 먼저 확인한 후, 하위 리소스의 추가 규칙을 적용한다. 상위 거부 시 하위 로드 없이 즉시 거부.

**Q. Resource state가 권한 판단에 영향을 주면 어느 계층에서 반영해야 하는가?**
> Service Layer가 리소스를 로드한 후 `resource_state`를 Authorization Service에 전달. Authorization Service가 상태 기반 추가 규칙을 평가한다. Repository는 상태 판단 책임 없음.

### 5-3. 원칙 정리

| 검사 방식 | 적용 요청 유형 |
|---|---|
| **Pre-check만** | Create (상위 컨테이너 권한 확인) |
| **Post-load만** | Read single, Update, Delete |
| **Pre-check + Post-load** | Nested resource 접근, List (Space pre-check 후 항목별 필터) |
| **남용 시 문제** | Post-load를 모든 곳에 쓰면 불필요한 DB 조회 급증. Pre-check만 쓰면 상태 기반 제어 누락 |

---

## 6. List/Search 권한 처리 원칙

### 6-1. 핵심 질문 답변

**Q. 권한 없는 리소스를 어떻게 처리할 것인가?**
> **완전 제외(filtering)**가 기본 원칙. 권한 없는 항목은 목록 결과에 포함하지 않는다. "접근 불가" 항목이 존재한다는 사실 자체도 노출하지 않는다.

**Q. 카운트(total count)에 권한 없는 항목을 포함할 것인가?**
> 포함하지 않는다. total count는 접근 가능한 항목 수만 반영한다. 전체 count를 노출하면 기밀 문서의 존재를 간접 추론할 수 있다.

**Q. 부분 마스킹과 완전 제외 중 어떤 원칙이 적절한가?**
> **기본은 완전 제외**. 단, 검색 결과에서 제목/요약만 노출하고 본문을 숨기는 "접근 제한 안내" 방식은 예외적으로 허용 가능하나, 이는 Phase 8 검색 기능 설계 시 별도 결정한다.

### 6-2. 시나리오별 처리

| 시나리오 | 처리 원칙 |
|---|---|
| 사용자가 접근 가능한 문서 목록 조회 | authorization-aware query로 DB에서 필터링. 서비스 레이어 후처리 지양 |
| 조직 관리자 전용 목록 조회 | Space:read 또는 org:manage 권한 pre-check 후 진입. 권한 없으면 403 |
| 감사 담당자의 감사 로그 조회 | audit_log:read 권한 pre-check 후 진입. 콘텐츠 역할과 완전 분리 |
| 검색에서 제목만 노출, 본문 숨김 | Phase 8에서 별도 결정. MVP에서는 검색 결과도 완전 접근 가능 항목만 반환 |

### 6-3. 원칙 정리

**List Endpoint 기본 원칙**
- 컨테이너(Space, Organization) 접근 권한 pre-check 먼저 수행
- 통과 시 authorization-aware query로 접근 가능 항목만 조회
- total count는 접근 가능 항목 수만 반영

**Search Endpoint 기본 원칙**
- 검색 결과도 동일한 권한 필터링 적용
- 권한 없는 문서의 제목/내용이 검색 결과에 포함되지 않음
- (Phase 8) 검색 인덱스 구축 시 권한 정보를 인덱스에 포함 검토

**Count / Aggregate 응답 원칙**
- 접근 가능한 리소스 기준으로만 계산
- 전체 수를 추론할 수 있는 정보 노출 금지

**부분 공개 허용 여부**
- MVP에서는 허용하지 않음. 완전 접근 가능한 항목만 응답
- 향후 "접근 제한 문서 존재 안내" 기능 도입 시 별도 설계

### 6-4. Authorization-aware Query vs 서비스 레이어 후처리

**authorization-aware query 권장 이유:**
- 서비스 레이어에서 전체 목록 로드 후 필터링하면, 대용량 데이터에서 심각한 성능 문제 발생
- DB 레벨 필터링이 성능과 보안 모두 유리
- Repository에 `membership_id`, `accessible_space_ids`, `accessible_document_ids` 등을 전달하여 쿼리 조건으로 사용

---

## 7. 오류 응답 정책

### 7-1. 상태 코드 정의

| 코드 | 의미 | 사용 조건 |
|---|---|---|
| **401 Unauthorized** | 인증 실패 | 토큰 없음, 만료, 유효하지 않은 토큰 |
| **403 Forbidden** | 인증은 됐으나 권한 없음 | 리소스 존재하고 접근 권한 없음 (일반 콘텐츠) |
| **404 Not Found** | 리소스 없음 또는 존재 은닉 | 리소스가 실제로 없거나, 보안 민감 리소스에서 존재를 숨길 때 |

### 7-2. 핵심 질문 답변

**Q. 존재하지만 접근 권한 없는 리소스에 403을 줄지 404를 줄지?**

| 리소스 유형 | 응답 | 이유 |
|---|---|---|
| 일반 콘텐츠 (Document, Space) | **403** | 존재는 알아도 됨. 접근 불가 이유를 명확히 전달 |
| 기밀/보안 민감 리소스 | **404** | 존재 자체를 노출하지 않음 (존재 은닉) |
| 운영 리소스 (AuditLog, Permission Policy) | **404** | 일반 사용자에게 해당 경로의 존재 자체를 숨김 |

**Q. 오류 응답에서 어느 정도까지 이유를 설명할 것인가?**
> 외부 응답에는 최소한의 이유만 포함. 내부 감사 로그에는 상세 reason을 기록.
> 외부: `{"error": "forbidden", "message": "접근 권한이 없습니다"}`
> 내부 감사: reason = `rbac_insufficient_role`, matched_role = `viewer`, required_permission = `document:update`

### 7-3. 응답 정책 정리

**일반 콘텐츠 리소스 (Document, Space, Comment 등)**
- 미인증 → 401
- 리소스 없음 → 404
- 권한 없음 → 403
- 접근 가능하나 Draft 상태 제한 → 403 (read_draft 권한 필요)

**보안 민감 리소스 (API Credential, Permission Policy)**
- 미인증 → 401
- 권한 없음 또는 리소스 없음 → **404** (존재 은닉)
- 내부 감사: 실제 거부 이유 전량 기록

**운영 리소스 (AuditLog, Integration Config)**
- 미인증 → 401
- 권한 없음 → **404** (일반 사용자에게 경로 자체를 숨김)
- 권한 있는 역할(SecurityAuditor 등)에게는 403 대신 정상 응답

**Admin API vs 일반 API**
- Admin API(`/admin/*`)는 진입 자체를 전역 역할로 pre-check
- 일반 API는 리소스 단위 권한 검사
- Admin API 접근 실패는 외부에 404 응답 (admin 경로 존재 은닉)

**감사 기록 시 남겨야 할 실제 거부 사유**

| reason code | 의미 |
|---|---|
| `not_authenticated` | 인증 실패 |
| `rbac_no_role` | Membership에 역할 없음 |
| `rbac_insufficient_role` | 역할은 있으나 해당 permission 부족 |
| `acl_no_entry` | RBAC 통과 못하고 ACL 엔트리도 없음 |
| `resource_state_restricted` | 리소스 상태(draft/locked/archived)로 인한 제한 |
| `scope_mismatch` | 요청 organization과 리소스 organization 불일치 |
| `default_deny` | 어떤 경로로도 allow를 얻지 못함 |

---

## 8. Authorization Context 모델

### 8-1. Authorization Service 입력 구조

```
AuthorizationContext {
  // 필수 입력
  principal: {
    user_id:         string         // 인증된 사용자 ID
    membership_id:   string | null  // 현재 요청의 조직 컨텍스트 membership
    organization_id: string | null  // 요청 조직 범위
    global_roles:    string[]       // 전역 역할 목록
    org_roles:       string[]       // 조직 내 역할 목록 (membership 기반)
  }

  resource: {
    resource_type:   string         // "document" | "space" | "org" | ...
    resource_id:     string         // 특정 리소스 ID
    parent_type:     string | null  // 상위 리소스 유형 (nested resource 시)
    parent_id:       string | null  // 상위 리소스 ID
    organization_id: string         // 리소스 소속 조직
    resource_state:  string | null  // "draft" | "published" | "archived" | "locked"
  }

  action: string                    // "document:read" | "space:manage" | ...

  // 선택 입력
  request_source:    string         // "user_ui" | "admin_ui" | "external_api" | "service_account"
  target_subject_id: string | null  // 권한 변경 대상 사용자 (manage 액션 시)
  request_id:        string         // 요청 추적용 correlation ID
}
```

### 8-2. Authorization Service 출력 구조

```
AuthorizationResult {
  allowed:              boolean
  reason:               string          // reason code (감사 로그용)
  matched_role:         string | null   // RBAC 경로 시
  matched_acl_entry_id: string | null   // ACL 경로 시
  evaluated_at:         datetime
  decision_metadata:    {               // 감사 로그에 그대로 전달
    user_id, membership_id, organization_id,
    resource_type, resource_id,
    action, allowed, reason,
    matched_role, matched_acl_entry_id,
    request_id, evaluated_at
  }
}
```

### 8-3. 필드 책임 분류

| 필드 | 자동 추론 가능 | Service Layer가 채워야 함 |
|---|---|---|
| `user_id` | Authentication Middleware에서 주입 | |
| `membership_id`, `org_roles` | Organization ID → Membership 조회로 자동 | |
| `global_roles` | user_id → GlobalRoleAssignment 조회로 자동 | |
| `resource_type`, `resource_id` | Router가 경로에서 파싱 | |
| `organization_id` (resource) | | 리소스 로드 후 Service가 전달 |
| `resource_state` | | 리소스 로드 후 Service가 전달 |
| `parent_type`, `parent_id` | | Nested resource 시 Service가 전달 |
| `request_source` | | 요청 출처에 따라 Service 또는 Middleware가 설정 |
| `request_id` | Middleware에서 생성/주입 | |

---

## 9. 계층별 책임 분리안

### 9-1. 각 계층 상세 책임

#### Authentication Middleware

- JWT/세션 토큰 검증
- `user_id` 추출, request context에 주입
- 토큰 만료/유효성 실패 → 즉시 401 반환
- `request_id` 생성 및 주입 (correlation ID)
- **권한 판단 없음**

#### Router / Controller

- URL path parameter 파싱 (`document_id`, `space_id` 등)
- Query parameter 파싱 (pagination, filter 등)
- Request body 파싱 및 기본 유효성 검사 (타입, 필수 필드)
- 적절한 Service 메서드 호출
- Service에서 받은 결과를 HTTP 응답으로 변환
- **권한 판단 없음. 에러 코드 변환(AuthorizationError → 403/404)만 담당**

#### Service / Application Layer

- 유스케이스 오케스트레이션 (비즈니스 흐름 조율)
- 리소스 존재 확인 및 로드 (Repository 호출)
- `resource_state`, `parent_id` 등 authorization context 구성
- **Authorization Service 호출** (권한 검사 위임)
- 권한 거부 시 적절한 에러 throw (AuthorizationError)
- 비즈니스 로직 실행 (권한 통과 후에만)
- 감사 이벤트 생성 트리거 (AuditEvent 발행)

#### Authorization Service

- AuthorizationContext를 입력받아 allow/deny 판단
- RBAC 평가: `org_roles` + `global_roles` → Permission 집합 → action 포함 여부
- ACL 평가: `resource_id` 기반 직접 ACL 조회 → 상속 ACL 조회
- `resource_state` 기반 추가 규칙 적용 (draft 접근 제한 등)
- `AuthorizationResult` 반환 (decision_metadata 포함)
- **비즈니스 로직, DB 직접 접근 없음**

#### Repository / Data Access Layer

- 순수 데이터 조회/저장 (비즈니스 로직 없음)
- List 조회 시 authorization-aware query 파라미터 수용:
  - `accessible_space_ids`, `accessible_document_ids` 등을 WHERE 조건으로 적용
- **권한 판단 없음. 쿼리 필터링은 Service가 전달한 조건을 반영하는 수동적 역할**

### 9-2. Repository Authorization-aware Query

**서비스 레이어 후처리 필터링의 문제:**
- 전체 데이터를 로드 후 메모리에서 필터링 → 대용량 데이터에서 성능 심각
- N+1 문제: 각 항목마다 ACL 조회 반복

**authorization-aware query 방식:**

```
// Service Layer에서 접근 가능한 공간 목록을 먼저 계산
accessible_space_ids = AuthorizationService.get_accessible_spaces(membership)

// Repository에 조건으로 전달
Repository.list_documents(
  organization_id=...,
  space_ids=accessible_space_ids,  // 접근 가능한 공간만 조회
  include_draft=(has_read_draft_permission)
)
```

단, 초기 MVP에서는 문서 수가 많지 않으므로 Service 레이어 필터링도 허용 가능. 데이터 규모 증가 시 authorization-aware query로 전환.

---

## 10. 공통 Enforcement 패턴 카탈로그

### Create Under Parent
```
대표 API:    POST /spaces/{space_id}/documents
필요 정보:   parent(space) 권한
action:      space:create_document
검사 시점:   pre-check (parent 기반)
주의:        parent 미존재 → 404. parent 권한 없음 → 403
감사 포인트: 생성 성공 시 document_id, space_id, creator 기록
```

### Read Single Resource
```
대표 API:    GET /documents/{document_id}
필요 정보:   resource + state
action:      document:read 또는 document:read_draft
검사 시점:   post-load check
주의:        draft 상태면 추가 read_draft 권한 확인
감사 포인트: deny 시 reason 기록 필수
```

### Update Mutable Resource
```
대표 API:    PUT /documents/{document_id}
필요 정보:   resource + state
action:      document:update
검사 시점:   post-load check
주의:        archived/locked 상태면 update 불가 (resource_state_restricted)
감사 포인트: 이전 version_id, 새 version_id 기록
```

### Delete Resource
```
대표 API:    DELETE /documents/{document_id}
필요 정보:   resource
action:      document:delete
검사 시점:   post-load check
주의:        소프트 삭제. 삭제 전 status 기록
감사 포인트: 삭제자, document_id, 삭제 시점, 삭제 전 status
```

### List Resources in Scope
```
대표 API:    GET /spaces/{space_id}/documents
필요 정보:   container 권한
action:      space:read (pre-check) + authorization-aware query
검사 시점:   pre-check(space) + DB query 필터
주의:        total count도 접근 가능 항목만 반영
감사 포인트: 대규모 목록 조회는 선택적 기록. deny 시 필수
```

### Manage Permissions on Resource
```
대표 API:    POST /documents/{document_id}/acl
필요 정보:   resource
action:      permission:manage
검사 시점:   post-load check
주의:        권한 정책 변경은 전량 감사 필수
감사 포인트: 변경 전/후 ACL 상태, 변경자, 대상 principal, reason
```

### Perform Admin-only Operation
```
대표 API:    POST /admin/organizations
필요 정보:   전역 역할
action:      org:manage (PlatformAdmin)
검사 시점:   pre-check (global_roles 기반)
주의:        Admin API 접근 실패 시 외부에 404 반환 (경로 은닉)
감사 포인트: 모든 admin 액션 전량 감사 필수
```

### Bulk Update/Delete
```
대표 API:    POST /documents/bulk-delete
필요 정보:   각 리소스별 개별 권한
action:      document:delete (각 항목에 적용)
검사 시점:   각 항목 post-load check
주의:        일부 성공/일부 실패 응답 처리 정책 필요. 권한 없는 항목은 결과에서 제외 또는 오류 항목으로 응답
감사 포인트: 각 항목별 처리 결과 기록
```

### Export / Publish / Approve
```
대표 API:    POST /documents/{id}/publish
필요 정보:   resource + state
action:      document:publish
검사 시점:   post-load check
주의:        상태 전이 규칙(draft → published) 위반 시 비즈니스 오류(422)
감사 포인트: 상태 전이 이벤트, 이전 상태, 새 상태, 수행자
```

### Nested Child Resource Management
```
대표 API:    GET /documents/{doc_id}/versions/{ver_id}
필요 정보:   상위 resource + 하위 resource state
action:      document:read → version:view
검사 시점:   상위 post-load → 하위 추가 규칙
주의:        상위 거부 시 하위 로드 없이 즉시 거부
감사 포인트: 상위/하위 리소스 모두 context에 기록
```

---

## 11. 외부 API / 서비스 계정 대응 원칙

### 11-1. 서비스 계정 요청과 일반 사용자 요청의 차이

| 항목 | 일반 사용자 | 서비스 계정 |
|---|---|---|
| **인증 방식** | OAuth / 세션 토큰 | API Key / Client Credentials |
| **Principal 유형** | User + Membership | ServiceAccount (Task 2-1에서 확장 예약) |
| **권한 부여 방식** | Membership → Role | ServiceAccount → Role (조직 범위) 또는 ACL 직접 부여 |
| **감사 추적** | user_id + membership_id | service_account_id + organization_id |
| **행위 범위** | 사용자가 권한 내에서 자유롭게 | 목적 제한적. 최소 권한 원칙 |

### 11-2. Organization-scoped Token 해석

- API 요청 헤더 또는 토큰 claim에 `organization_id` 포함
- Authorization Service는 해당 organization_id 기반으로 Membership/ServiceAccount 컨텍스트 구성
- organization_id 없는 요청 → 전역 역할만 적용 (PlatformAdmin 등)

### 11-3. Acting Principal 기록 원칙

```
// 서비스 계정이 사용자를 대신하는 경우 (위임 접근)
AuditEvent {
  actor_type:       "service_account"
  actor_id:         "sa_pipeline_001"
  on_behalf_of:     "user_abc"           // 위임 사용자 (있는 경우)
  organization_id:  "org_001"
  ...
}

// 서비스 계정 독립 접근
AuditEvent {
  actor_type:       "service_account"
  actor_id:         "sa_pipeline_001"
  on_behalf_of:     null
  ...
}
```

### 11-4. User Delegated Access vs Service Account Access 구분

| 유형 | 설명 | 감사 표현 |
|---|---|---|
| **User Access** | 사용자가 직접 UI/API 사용 | actor = user |
| **Service Account Access** | 자동화 파이프라인, AI 에이전트 | actor = service_account |
| **User Delegated** | 서비스가 특정 사용자 권한으로 행동 | actor = service_account, on_behalf_of = user |

### 11-5. MVP 기준 대응 수준

- **MVP**: User 토큰 기반 접근만 지원. Service Account는 Authorization Service의 Principal 추상화로 확장 가능성만 열어둠
- **Phase 3 API 계층 설계 시**: Service Account 엔티티 도입, API Key 인증, organization-scoped token 설계

---

## 12. 다음 Task로 넘길 결정사항

### Task 2-5 (감사 로그 설계) 에 전달

**Authorization decision 시 남겨야 할 핵심 metadata:**

```
AuthorizationDecisionLog {
  request_id:           string     // correlation ID. 요청 전체 추적
  evaluated_at:         datetime
  
  // Principal context
  user_id:              string
  membership_id:        string | null
  organization_id:      string | null
  acting_roles:         string[]   // 평가 시 사용된 역할 목록
  
  // Resource context
  resource_type:        string
  resource_id:          string
  resource_state:       string | null
  parent_resource_type: string | null
  parent_resource_id:   string | null
  
  // Decision
  action:               string
  allowed:              boolean
  reason:               string     // reason code
  matched_role:         string | null
  matched_acl_entry_id: string | null
  
  // Request context
  request_source:       string     // "user_ui" | "external_api" | "service_account"
  ip_address:           string     // (보안 분석용)
}
```

**Denial Reason Category:**

| reason code | 분류 | 설명 |
|---|---|---|
| `not_authenticated` | 인증 실패 | |
| `rbac_no_role` | RBAC 실패 | 역할 없음 |
| `rbac_insufficient_role` | RBAC 실패 | 역할 있으나 permission 부족 |
| `acl_no_entry` | ACL 실패 | RBAC 실패 + ACL 없음 |
| `resource_state_restricted` | 상태 제한 | draft/locked/archived |
| `scope_mismatch` | 범위 불일치 | 조직 컨텍스트 불일치 |
| `default_deny` | 기본 거부 | 어느 경로도 없음 |

**Correlation ID / Request ID:**
- 모든 요청에 `request_id` 생성 (Middleware에서 UUID 생성)
- AuthorizationDecisionLog와 AuditEvent에 동일 `request_id` 포함
- 사후 분석 시 단일 요청의 전체 흐름 추적 가능

**Post-event Audit vs Runtime Authorization Decision Log:**

| 유형 | 목적 | 기록 시점 | 보존 |
|---|---|---|---|
| **AuthorizationDecisionLog** | "왜 허용/거부됐는가" 결정 근거 기록 | 권한 평가 직후 | 보안 감사 목적, 장기 보존 |
| **AuditEvent** | "무엇을 했는가" 사용자 행위 기록 | 비즈니스 로직 완료 후 | 컴플라이언스, 장기 보존 |

- deny 된 요청: AuthorizationDecisionLog 기록 (AuditEvent는 생성되지 않음)
- allow 된 요청: AuthorizationDecisionLog + AuditEvent 모두 기록 (상황에 따라 AuthorizationDecisionLog는 선택적)

---

## 13. 오픈 이슈

| # | 이슈 | 설명 | 결정 필요 시점 |
|---|---|---|---|
| OI-1 | Authorization Service 캐싱 전략 | Role/ACL 조회를 매 요청마다 수행하면 성능 부담. 캐싱 시 권한 변경 즉시 반영 불가 문제 | Phase 3 구현 단계 |
| OI-2 | Bulk action 부분 성공 응답 스펙 | 일부 항목 권한 없을 때 207 Multi-Status 응답 또는 전체 실패 정책 | Phase 3 API 설계 |
| OI-3 | List API에서 authorization-aware query 전환 시점 | MVP에서는 서비스 레이어 필터링 허용 여부. 데이터 규모 기준 미정 | Phase 3~4 구현 |
| OI-4 | Admin API 경로 은닉 수준 | `/admin/*` 경로 전체를 숨길지, 일부만 숨길지. 개발/디버깅 편의성과 보안의 균형 | Phase 3 API 설계 |
| OI-5 | resource_state 기반 규칙의 Authorization Service 내 구현 vs 시스템 규칙 분리 | draft 접근 제한을 Authorization Service가 처리할지, Service Layer에서 사전 차단할지 | Phase 3 구현 단계 |

---

*작성일: 2026-04-01*
*Phase: 2 / Task: 2-4*
*상태: 설계 초안 완료*
