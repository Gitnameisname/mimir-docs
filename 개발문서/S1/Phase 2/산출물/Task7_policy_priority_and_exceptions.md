# Phase 2 - Task 2-7. 정책 적용 우선순위와 예외 규칙 정리

---

## 1. 목표

Task 2-1~2-6에서 설계한 권한 주체, 리소스 범위, ACL 모델, API enforcement, 감사 로그, 행위 추적 위에, **실제 시스템에서 정책/상태/예외가 충돌할 때 무엇이 우선하는지**의 기준선을 확정한다.

- 정책 계층 후보를 분석하고 권장 평가 순서를 확정한다
- 정책 충돌 시 해소 원칙을 명확한 규칙으로 정리한다
- 리소스 상태(draft/locked/archived 등) 기반 권한 변화 규칙을 정의한다
- 관리자 권한의 경계와 break-glass 도입 여부를 결정한다
- ACL 예외가 허용되는 범위와 허용되지 않는 범위를 확정한다
- 고위험 액션 추가 가드 원칙을 정의한다
- Task 2-8 통합 설계서에 넘길 정책 결정사항을 정리한다

---

## 2. 설계 전제

| # | 전제 |
|---|---|
| 1 | 기본 권한 모델: RBAC + resource-scoped ACL. Task 2-3에서 확정 |
| 2 | 정책 평가 순서: RBAC 1차 → ACL 2차 → 상속 ACL 3차 → default deny. Task 2-3에서 확정 |
| 3 | 권한이 있더라도 리소스 상태(locked, archived 등)나 시스템 정책으로 실행이 제한될 수 있다 |
| 4 | 시스템 관리자도 무제한이 아니다. 콘텐츠 접근과 운영 권한은 분리되어야 한다 |
| 5 | MVP는 예측 가능하고 명시적인 규칙 우선. 동적/복잡한 정책 엔진은 확장 단계 |
| 6 | 모든 정책 거부/예외/override는 감사 로그에 사후 설명 가능한 형태로 기록 |

---

## 3. 정책 계층 후보 분석

### 3-1. 계층별 정의 및 평가 필요성

#### L1. Authentication Validity (인증 유효성)

| 항목 | 내용 |
|---|---|
| **담당** | 요청자가 유효한 인증 자격 보유 여부. 토큰 유효성/만료 확인 |
| **왜 먼저** | 인증도 안 된 요청에 권한 평가 자체가 무의미 |
| **MVP** | 필수 |

#### L2. Principal Active Status (주체 활성 상태)

| 항목 | 내용 |
|---|---|
| **담당** | User 계정 활성/비활성/정지 여부. Membership suspended/left 여부 |
| **왜 먼저** | 퇴사자, 정지 계정이 보유한 역할/ACL을 평가할 필요 없음 |
| **MVP** | 필수 |

#### L3. Global Platform Policy (전역 플랫폼 정책)

| 항목 | 내용 |
|---|---|
| **담당** | 플랫폼 전체에 적용되는 시스템 수준 제한 (예: 특정 API 비활성화, 점검 중 접근 차단) |
| **왜 먼저** | 조직/역할과 무관하게 플랫폼 운영 정책이 모든 요청에 우선 적용 |
| **MVP** | 부분 포함 (시스템 점검 상태 정도만. 전체 정책 엔진은 확장) |

#### L4. Organization Boundary (조직 경계)

| 항목 | 내용 |
|---|---|
| **담당** | 요청자가 대상 리소스의 조직에 유효한 Membership을 가지는가. 테넌트 경계 |
| **왜 먼저** | 다른 조직 리소스에 대한 접근 시도를 ACL 평가 전에 차단 |
| **MVP** | 필수 |

#### L5. Role-based Permission (역할 기반 권한)

| 항목 | 내용 |
|---|---|
| **담당** | Membership → Role → Permission 집합 평가. RBAC 핵심 |
| **평가 위치** | L1~L4 통과 후 1차 권한 평가 |
| **MVP** | 필수 |

#### L6. Resource-specific ACL (리소스 직접 ACL)

| 항목 | 내용 |
|---|---|
| **담당** | 특정 리소스에 직접 부여된 ACL 엔트리. RBAC 결과를 보완하는 예외 계층 |
| **평가 위치** | RBAC 통과 못했을 때 ACL 직접 부여 확인 |
| **MVP** | 필수 (Space, Document 수준만) |

#### L7. Resource State Restriction (리소스 상태 제한)

| 항목 | 내용 |
|---|---|
| **담당** | 리소스의 lifecycle state(locked, archived, draft 등)에 따른 행위 제한 |
| **왜 별도 계층** | 역할/ACL이 allow를 부여해도 상태 제한이 그 위에 적용되어야 함. "권한은 있지만 상태가 금지" |
| **MVP** | 필수 |

#### L8. Sensitive Operation Guard (고위험 액션 가드)

| 항목 | 내용 |
|---|---|
| **담당** | 기본 권한 외 추가 보호가 필요한 고위험 액션에 대한 추가 검사 (사유 입력, 강화 감사 등) |
| **MVP** | 부분 포함 (사유 기록 의무화, 강화 감사 정도) |

#### L9. Compliance / Audit-required Rule (컴플라이언스 필수 규칙)

| 항목 | 내용 |
|---|---|
| **담당** | 감사 로그 기록이 보장되지 않으면 실행 자체를 차단하는 규칙 |
| **MVP** | 확장 (Phase 13 보안/운영 체계 수립 시) |

#### L10. External Access / Share Rule (외부 공유 규칙)

| 항목 | 내용 |
|---|---|
| **담당** | 조직 외부 주체의 접근. Share Link, External API Client 등 |
| **MVP** | 부분 포함 (Share Link 기능 도입 시 활성화) |

---

## 4. 권장 정책 평가 순서

### 4-1. 핵심 질문 답변

**Q. 인증되지 않은 요청은 가장 먼저 배제해야 하는가?**
> 예. 인증 없이 어떤 평가도 의미 없다. 즉시 401 반환.

**Q. 비활성 사용자/중지된 멤버십은 role/ACL 평가 이전에 차단해야 하는가?**
> 예. 퇴직 처리된 계정이 보유한 Admin 역할이라도 계정 정지 후에는 무효화되어야 한다. 상태 체크가 역할/ACL 평가보다 앞서야 한다.

**Q. 조직 경계 위반은 ACL 예외보다 우선 차단되어야 하는가?**
> 예. ACL로 다른 조직의 리소스에 접근 권한을 줄 수 없다. 조직 경계는 ACL보다 상위 보호막이다.

**Q. role/ACL allow가 있어도 resource state restriction이 더 우선해야 하는가?**
> 예. "Editor 역할이지만 locked 문서는 수정 불가"가 원칙이다. 상태 제한은 역할 위에서 작동하는 추가 규칙이다.

**Q. 고위험 액션은 기본 권한이 있어도 추가 가드가 필요한가?**
> 예. 권한이 있어도 고위험 액션(대량 export, 권한 정책 변경 등)은 사유 기록, 강화 감사 등의 추가 단계가 필요하다.

### 4-2. 권장 정책 평가 순서

```
Step 1. Authentication Validity
  → 인증 토큰 유효성/만료 확인
  → 실패 시: 401 즉시 반환. 평가 중단.

Step 2. Principal Active Status
  → User 계정 활성 여부 (status: active?)
  → Membership 유효 여부 (status: active?)
  → 실패 시: 403 반환 (계정 비활성). 평가 중단.
  → 감사: auth.access.denied (reason: principal_inactive)

Step 3. Global Platform Policy
  → 시스템 점검/비활성 API 등 전역 제한 확인
  → 실패 시: 503 또는 403. 평가 중단.
  → 감사: auth.access.denied (reason: platform_policy_restriction)

Step 4. Organization Boundary
  → 요청 대상 리소스가 요청자의 Membership 조직 내에 있는가
  → 전역 역할(PlatformAdmin 등) 보유 시 조건부 통과 (8항 참조)
  → 실패 시: 403 또는 404. 평가 중단.
  → 감사: auth.access.denied (reason: scope_mismatch)

Step 5. Role-based Permission (RBAC)
  → Membership → Role → Permission 집합 평가
  → action이 permission 집합에 포함되면 → allow 후보
  → 미포함 → Step 6으로 이동

Step 6. Resource-specific ACL
  → 직접 부여된 ACL 엔트리 확인
  → 상속 ACL 확인
  → allow 엔트리 있으면 → allow 후보
  → 없으면 → default deny → 403/404. 감사 기록.

Step 7. Resource State Restriction
  → allow 후보 상태에서 리소스 lifecycle state 확인
  → locked / archived / soft_deleted / pending_approval 등
  → 상태 제한에 걸리면 → 실행 차단. 역할/ACL allow 무효화.
  → 감사: auth.access.denied (reason: resource_state_restricted)

Step 8. Sensitive Operation Guard
  → 고위험 액션 여부 확인
  → 해당 시: 사유 기록 의무화, 강화 감사 트리거
  → 추가 조건(재인증, 2단계 승인 등) → 확장 단계

Step 9. Final Decision + Audit Metadata 생성
  → allow: 실행 허용. AuditEvent + AuthorizationDecisionLog 기록
  → deny: 403/404 응답. AuthorizationDecisionLog 기록 (이전 단계에서 이미 기록됨)
```

### 4-3. 평가 순서 요약도

```
요청
  │
  ▼ [Step 1] 인증 유효?           → NO → 401
  │
  ▼ [Step 2] 주체 활성?           → NO → 403
  │
  ▼ [Step 3] 플랫폼 정책 통과?    → NO → 403/503
  │
  ▼ [Step 4] 조직 경계 내?        → NO → 403/404
  │
  ▼ [Step 5] RBAC allow?          → YES ─────────────┐
  │                                                    │
  ▼ [Step 6] ACL allow?           → YES ─────────────┤
  │                  NO → 403 (default deny)           │
  │                                                    ▼
  │                                           [Step 7] 상태 제한?
  │                                                    │
  │                                           YES → 403 (state restricted)
  │                                                    │
  │                                                    NO
  │                                                    │
  │                                                    ▼
  │                                           [Step 8] 고위험 가드?
  │                                                    │
  │                                           해당 → 추가 조건 적용
  │                                                    │
  └────────────────────────────────────────────────────▼
                                              [Step 9] ALLOW + 감사
```

---

## 5. 정책 충돌 해소 원칙

### 5-1. 충돌 케이스별 판단

#### 케이스 1: Role은 허용하지만 resource state가 locked인 경우

> **판단: 상태 제한 우선. 실행 차단.**
> 이유: locked 상태는 "권한 여부와 무관하게 변경을 동결"하는 운영 제어다. 역할 기반 권한보다 상위 계층의 제한이다.
> 감사: `resource_state_restricted`, `locked_by`, `locked_at` 포함

#### 케이스 2: Organization policy는 금지하지만 resource ACL은 허용하는 경우

> **판단: 조직 경계 정책 우선. ACL 예외 차단.**
> 이유: ACL은 조직 내 예외를 위한 것이지, 조직 정책 자체를 뒤집기 위한 것이 아니다. 조직 경계는 ACL보다 상위 보호막이다.
> 예: 조직이 "외부 공유 금지" 정책을 설정했다면, 특정 문서에 외부 공유 ACL이 있어도 조직 정책이 우선.
> 감사: `org_policy_restriction`

#### 케이스 3: 상위 scope inherited allow가 있지만 하위 resource가 별도 제한 상태인 경우

> **판단: 하위 리소스 상태 제한 우선. 상속 allow 무효화.**
> 이유: 상속은 "명시적 제한이 없을 때의 기본값"이다. 하위 리소스에 명시적 상태 제한이 있으면 상속이 아닌 직접 규칙이 적용된다.
> 감사: `resource_state_restricted` + `scope_inheritance_overridden`

#### 케이스 4: Platform admin이지만 tenant boundary를 넘는 접근인 경우

> **판단: 기본적으로 차단. break-glass 또는 명시적 cross-org 권한 필요.**
> 이유: PlatformAdmin이 모든 조직 데이터에 자동 접근하면 테넌트 간 데이터 분리 원칙이 무너진다. 운영 목적 접근은 break-glass 절차로 통제한다.
> 예외: SecurityAuditor는 감사 목적으로 조직 AuditLog에 read 접근 가능 (콘텐츠 원문 제외).
> 감사: `cross_tenant_access_attempt` → deny 또는 break-glass 기록

#### 케이스 5: Document read는 허용되지만 export는 금지된 경우

> **판단: export 금지 우선. read allow는 export에 전이되지 않는다.**
> 이유: read와 export는 별도 permission이다. `document:read` allow가 `document:export` allow를 의미하지 않는다.
> 감사: `rbac_insufficient_role` (read_only 역할이 export permission 없을 때)

#### 케이스 6: Edit 권한은 있지만 pending_approval 상태라 수정이 제한되는 경우

> **판단: pending_approval 상태 제한 우선. edit 권한 있어도 실행 차단.**
> 이유: 승인 대기 중인 문서는 워크플로 무결성을 위해 추가 수정이 제한된다. 검토자가 승인/반려한 내용이 수정되면 절차가 무의미해진다.
> 예외: 워크플로를 관리하는 역할(SpaceManager 이상)은 취소 후 재수정 가능.
> 감사: `resource_state_restricted` (reason: pending_approval)

### 5-2. 충돌 해소 기본 원칙 (9개 규칙)

```
Rule 1. 인증/주체 유효성은 모든 것에 우선한다.
        비활성 계정의 Admin 역할도 무효다.

Rule 2. 조직 경계는 ACL 예외보다 우선한다.
        ACL로 테넌트 경계를 넘는 권한을 줄 수 없다.

Rule 3. 리소스 상태 제한은 역할/ACL allow 위에서 작동한다.
        "권한은 있지만 상태가 금지"가 "권한이 있으면 상태를 무시"보다 우선한다.

Rule 4. 더 구체적인 범위의 규칙이 상위 상속 규칙보다 우선한다.
        Document ACL이 Space 상속 ACL보다 우선. 단, 상태 제한은 예외 없이 적용.

Rule 5. 별도 permission은 별도로 평가한다.
        read allow가 export allow를 의미하지 않는다. permission은 전이되지 않는다.

Rule 6. deny는 allow보다 우선한다. (확장 단계에서 deny 도입 시)
        MVP에서는 allow-only이므로 해당 없음. 도입 시 명시적 deny가 최우선.

Rule 7. 관리자 역할이 리소스 상태를 자동 우회하지 않는다.
        OrganizationAdmin도 locked 문서를 unlock 없이 수정할 수 없다.
        단, lock 해제 권한은 관리자에게 부여 가능.

Rule 8. 정책 충돌 시 더 제한적인 쪽이 기본값이다.
        "허용해야 할 이유"가 없으면 거부한다 (default deny).

Rule 9. 모든 정책 거부/예외/override는 감사 로그에 기록한다.
        나중에 "왜 허용/거부됐는가"를 설명할 수 없으면 잘못된 설계다.
```

---

## 6. 상태 기반 예외 규칙

### 6-1. 리소스 상태별 행위 허용/제한 표

| 상태 | read | create (하위) | update | delete | export | publish | restore | 관리자 예외 | ACL 우회 | 감사 강도 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|---|
| **draft** | Editor+ | — | Editor+ | Editor+ | Editor+ | Editor+ | — | 가능 | 가능 | 중간 |
| **active/published** | Viewer+ | — | Editor+ | Editor+ | Editor+ | — | — | 가능 | 가능 | 낮음 |
| **locked** | Viewer+ | — | **차단** | **차단** | 역할 따름 | **차단** | — | lock 해제 후 | **불가** | 높음 |
| **archived** | Viewer+ | — | **차단** | **차단** | Viewer+ | **차단** | Admin+ | Admin+ | **불가** | 중간 |
| **soft_deleted** | **숨김** | — | **차단** | **차단** | **차단** | **차단** | Admin+ | **불가** | **불가** | 높음 |
| **pending_approval** | Editor+ | — | **차단** | **차단** | Editor+ | — | — | 워크플로 관리자 | **불가** | 높음 |
| **read_only** | Viewer+ | — | **차단** | **차단** | Viewer+ | — | — | Admin+ | **불가** | 낮음 |
| **external_shared** | 공유 수신자 | — | **차단** | **차단** | 조건부 | — | — | 공유 소유자 | — | 높음 |

> `Editor+` = Editor 이상 역할. `Admin+` = OrganizationAdmin 이상.

---

## 7. Soft Delete / Archived / Locked 상세 원칙

### 7-1. Soft Delete

**조회 가능 여부:**
- 일반 사용자(Viewer, Editor): **숨김**. 목록/검색에서 제외. 직접 URL 접근 시 404 반환
- OrganizationAdmin 이상: **조회 가능** (삭제 이력 확인, 복원 목적)
- SecurityAuditor: **조회 가능** (감사 추적 목적)

**복원 권한:**
- OrganizationAdmin 이상만 복원 가능
- 복원 시 이전 상태(published/draft)로 돌아감
- 복원은 강화 감사 필수

**영구 삭제:**
- MVP에서는 영구 삭제 기능 없음. soft delete만 허용
- 향후 도입 시: PlatformAdmin 또는 OrganizationOwner만 가능. 강화 감사 + 사유 필수

**MVP 권장안:**
- soft_deleted 문서는 일반 사용자에게 404. Admin에게만 별도 "삭제됨" 목록 노출
- deleted_at 필드로 구현. 별도 상태 enum 불필요

### 7-2. Archived

**수정 차단 여부:**
- 수정(update), 게시(publish), 삭제 차단. 아카이브 상태는 완료된 문서를 의미
- "아카이브 문서를 수정하려면 재활성화 먼저" 원칙

**조회 허용 여부:**
- Viewer 이상 모두 조회 가능. 공개 접근 제한은 별도 Space ACL로 처리

**Export/Download 허용 여부:**
- 허용. 아카이브 문서는 읽기 전용이므로 export는 가능

**재활성화 권한:**
- OrganizationAdmin 이상만 archived → published/draft 전환 가능
- 재활성화 시 AuditEvent 기록 필수

**MVP 권장안:**
- archived 상태는 Document.status enum의 하나. 별도 테이블 불필요
- 수정 시도 시 `resource_state_restricted` 오류 반환

### 7-3. Locked

**누가 잠글 수 있는가:**
- MVP: SpaceManager 이상 또는 Document 소유 Editor (자신의 문서에 한해)
- 확장: 워크플로 자동 lock (승인 진행 중 자동 잠금 등)

**잠금 상태에서 누가 수정 가능한가:**
- **아무도 직접 수정 불가.** lock 해제 후에만 수정 가능
- lock을 건 주체 또는 SpaceManager 이상이 해제 가능

**Lock Override:**
- MVP: **허용하지 않음.** 반드시 lock 해제 → 수정 → 재잠금 절차
- 확장: 긴급 override는 OrganizationOwner만 가능. 강화 감사 + 사유 필수 (break-glass 준용)

**Emergency Override 허용 여부:**
- MVP에서는 lock override 없음. 예측 가능성 우선
- 확장 단계에서 break-glass 준용 절차로 도입 가능

**MVP 권장안:**
- locked 상태는 Document에 `locked_at`, `locked_by` 필드로 표현
- lock/unlock은 별도 API 엔드포인트 (PATCH /documents/{id}/lock)
- 잠금/해제 시 AuditEvent 필수

---

## 8. 관리자 권한 경계 정의

### 8-1. 핵심 질문 답변

**Q. 플랫폼 관리자는 모든 조직 데이터를 무조건 읽을 수 있어야 하는가?**
> 아니다. PlatformAdmin은 플랫폼 운영(조직 관리, 시스템 설정, 사용자 관리)을 담당하지, 각 조직의 문서 콘텐츠에 자동 접근해서는 안 된다. 테넌트 간 데이터 분리 원칙을 위반한다.

**Q. 보안 감사자는 콘텐츠 원문까지 봐야 하는가?**
> 아니다. SecurityAuditor는 감사 로그(누가 무엇을 했는가)와 메타데이터(문서 존재 여부, 권한 변경 이력)에 접근 가능하면 충분하다. 문서 본문은 업무상 필요 원칙(need-to-know)에 따라 별도 Membership이 있어야 한다.

**Q. 조직 관리자는 soft-deleted 문서나 감사 로그까지 볼 수 있는가?**
> - soft-deleted 문서: **예.** 조직 데이터 관리 목적으로 OrganizationAdmin은 soft-deleted 문서 조회 및 복원 가능
> - 감사 로그(조직 범위): **예.** OrganizationAdmin은 자기 조직 범위의 AuditLog 조회 가능
> - 플랫폼 전체 AuditLog: **아니오.** SecurityAuditor/PlatformAdmin만 접근 가능

**Q. 운영 지원 역할은 실제 문서 본문 접근 없이 문제를 재현할 수 있어야 하는가?**
> 예. SupportOperator는 문서 본문 없이 메타데이터, 상태, 접근 이력만으로 대부분의 지원 업무를 처리할 수 있어야 한다. 필요 시 break-glass 절차로 일시적 접근.

**Q. break-glass 접근이 필요한가?**
> 예. 긴급 운영 상황에서 일시적으로 테넌트 경계를 넘는 접근이 필요할 수 있다. 단, 엄격히 통제되어야 한다.

### 8-2. 관리자 권한 기본 원칙

```
원칙 1. 운영 권한과 콘텐츠 접근 권한은 분리된다.
         PlatformAdmin = 플랫폼 운영. 문서 콘텐츠 자동 접근 없음.
         콘텐츠 접근을 위해서는 별도 Membership 또는 break-glass 필요.

원칙 2. SecurityAuditor는 감사 로그/메타데이터만. 콘텐츠 원문 접근 없음.

원칙 3. OrganizationAdmin은 자기 조직 범위에서만 최고 권한.
         타 조직 데이터에 접근 불가.

원칙 4. PlatformAdmin의 조직 데이터 접근은 break-glass 절차로만 허용.
```

### 8-3. 역할별 접근 범위 정리

| 역할 | 플랫폼 운영 | 조직 관리 | 콘텐츠(문서 본문) | AuditLog | 타 조직 접근 |
|---|:---:|:---:|:---:|:---:|:---:|
| PlatformOwner | 전체 | 전체 | break-glass만 | 전체 | break-glass만 |
| PlatformAdmin | 전체 | 전체 | break-glass만 | 전체 | break-glass만 |
| SecurityAuditor | 읽기 | 메타만 | **없음** | 전체 | **없음** |
| SupportOperator | 읽기 | 메타만 | **없음** | 조직 범위 | **없음** |
| OrganizationOwner | — | 자기 조직 전체 | 자기 조직 | 자기 조직 | **없음** |
| OrganizationAdmin | — | 자기 조직 전체 | 자기 조직 | 자기 조직 | **없음** |

### 8-4. Break-glass 정의

| 항목 | 내용 |
|---|---|
| **정의** | 긴급 운영 상황에서 일반 권한 경계를 임시로 넘는 통제된 접근 |
| **허용 대상** | PlatformOwner, PlatformAdmin (MVP에서는 PlatformOwner만) |
| **요건** | 사유 기록 필수. 별도 승인 절차 (MVP: 사유 입력 + 강화 감사) |
| **범위** | 특정 조직의 특정 리소스에 한해 일시적 접근. 전체 접근 아님 |
| **자동 만료** | 확장 단계. MVP에서는 수동 해제 |
| **감사 강도** | 최고 등급. 모든 접근 행위 전량 기록 |
| **MVP 포함 여부** | 구조만 설계. 실제 기능은 Phase 13 이후 |

### 8-5. 감사 강도가 높아야 하는 관리자 행위

- 전역 역할 부여/회수 (PlatformAdmin ↔ SecurityAuditor)
- Organization 삭제/아카이브
- Break-glass 접근 시작/종료
- Permission Policy 변경
- API Credential 발급/폐기
- AuditLog 내보내기

---

## 9. ACL 예외 허용 범위 제한안

### 9-1. ACL 예외 허용 가능/불가 판단

**Q. ACL로 조직 경계를 넘는 권한을 줄 수 있는가?**
> **불가.** 조직 경계는 ACL보다 상위 보호막이다. 다른 조직의 Membership에 ACL을 부여하는 것은 허용하지 않는다.

**Q. ACL로 locked/archived 상태를 우회할 수 있는가?**
> **불가.** 상태 제한은 ACL보다 상위 계층(Step 7)에서 적용된다. ACL이 allow를 부여해도 상태 제한이 그 위에서 차단한다.

**Q. ACL로 admin-only action을 일반 사용자에게 줄 수 있는가?**
> **불가.** `permission:manage`, `audit_log:read`, `api_credential:manage` 같은 admin-only permission은 ACL로 부여할 수 없다. 역할 경로로만 부여 가능.

**Q. ACL로 export/share만 별도 허용하는 것이 가능한가?**
> **가능.** `document:export`, `document:share`는 ACL로 예외 부여 허용 범위 내에 있다.

### 9-2. ACL 허용/불가 원칙 정리

**ACL로 허용 가능한 예외:**
- Space, Document 수준 read/update/share 예외 부여
- 특정 Membership에 특정 문서 read_draft 접근 허용
- 특정 Team에 특정 공간 접근 허용

**ACL로 허용하면 안 되는 예외:**
- 조직 경계를 넘는 접근 (타 조직 Membership에 부여)
- locked/archived/soft_deleted 상태 우회
- admin-only permission (`permission:manage`, `audit_log:read` 등)
- 리소스 상태 변경 권한 (lock/unlock, archive/restore) — 역할로만

**정책적으로 상위에 두어야 하는 제한:**
- 조직 경계 (L4. Organization Boundary)
- 리소스 상태 제한 (L7. Resource State Restriction)
- 플랫폼 전역 정책 (L3. Global Platform Policy)

**MVP ACL 예외 허용 범위 (Task 2-3 재확인):**
- 대상 리소스: Space, Document만
- 대상 principal: Membership (주요), User (전역 리소스 한정)
- 허용 permission: read, read_draft, update, share, publish (admin/운영 permission 제외)
- ACL 생성 권한: Space → OrganizationAdmin 이상. Document → SpaceManager 이상

---

## 10. 고위험 액션 정책 초안

### 10-1. 고위험 액션 목록 및 가드 원칙

| 액션 | 왜 고위험인가 | 추가 가드 | MVP 적용 |
|---|---|---|---|
| **권한/역할/ACL 변경** | 권한 체계 자체를 변경. 남용 시 보안 구조 붕괴 | 사유 기록 필수. 강화 감사(전/후 snapshot). 변경 후 알림 | 필수 |
| **대량 export** | 단시간 대량 데이터 유출 가능 경로 | 사유 기록 필수. 건수/용량 임계치 초과 시 추가 확인 (확장). 강화 감사 | 필수 (사유 기록) |
| **외부 share 생성** | 조직 외부로 데이터 공개. 오용 시 정보 유출 | 사유 기록 권장. 만료 기간 강제 (확장). 강화 감사 | 확장 |
| **API Credential 발급/폐기** | 인증 자격 변경. 유출 시 시스템 전체 접근 가능 | 사유 기록 필수. 강화 감사. 발급 후 즉시 알림 | 필수 |
| **Organization Owner 변경** | 조직 최고 권한 이전 | 사유 기록 필수. 현 Owner의 명시적 확인 필요. 강화 감사 | 필수 |
| **soft-deleted 문서 영구 삭제** | 복구 불가. 증거 인멸 가능성 | MVP에서 영구 삭제 불허. 확장 시: PlatformAdmin만. 사유 + 강화 감사 | 확장 |
| **locked resource override** | 잠금 우회. 승인 절차 파괴 | MVP: 불허. 확장 시 break-glass 준용 | 확장 |
| **break-glass 접근** | 테넌트 경계 일시 우회 | PlatformOwner만. 사유 필수. 접근 범위 제한. 모든 행위 전량 감사 | 구조만 |
| **감사 로그 내보내기** | 보안 데이터 외부 반출 가능 | SecurityAuditor 이상만. 사유 기록 필수. 강화 감사 | 필수 |
| **Permission Policy 변경** | 권한 정책 자체 변경 | 사유 기록 필수. 변경 전/후 full snapshot 감사 | 필수 |

### 10-2. MVP 고위험 가드 구현 수준

MVP에서 적용할 최소 가드:
1. **사유 기록 필수**: 고위험 액션 API 요청에 `reason` 파라미터 필수화
2. **강화 감사**: 고위험 이벤트는 before/after full snapshot 포함 AuditLog 기록
3. **Role 제한 강화**: admin-only permission은 ACL로 부여 불가 (역할 경로만)

MVP에서 제외하는 가드 (확장 단계):
- 재인증 요구 (2FA 재확인)
- 2단계 승인 (다른 관리자 확인)
- 시간대 제한
- 자동 만료 정책

---

## 11. 향후 정책 엔진 연결 포인트

현재 설계한 Step 7(Resource State Restriction)과 Step 8(Sensitive Operation Guard) 사이 또는 그 아래에 정책 엔진 확장 훅을 연결할 수 있다.

### 11-1. Approval Workflow 결합

```
[연결 위치: Step 7.5 — State + Workflow Check]
- pending_approval 상태 문서에 대한 수정 차단은 이미 Step 7에서 처리
- 향후: 특정 액션(publish, delete 등)에 approval gate를 삽입
  → 권한이 있어도 미승인 시 실행 대기 상태로 전환
- 훅: workflow_gate_check(action, resource) → allow | pending
```

### 11-2. Time-based Restriction

```
[연결 위치: Step 3.5 — Platform/Org Policy 확장]
- 예: 외부 공유는 업무시간(09:00~18:00)에만 허용
- 예: 특정 통합 API는 야간 배치 시간대에만 허용
- 훅: time_policy_check(action, current_time, org_id) → allow | deny
```

### 11-3. Environment/Source-based Restriction

```
[연결 위치: Step 3.5 또는 Step 8]
- 예: service_account는 export API 호출 불가
- 예: external_api 소스는 admin 액션 불가
- 훅: source_policy_check(action, request_source, actor_type) → allow | deny
```

### 11-4. Sensitivity-based Restriction

```
[연결 위치: Step 7]
- 예: 민감 태그(confidential) 문서는 export 금지
- 예: 민감 문서는 외부 공유 자동 차단
- 훅: sensitivity_check(resource_id, action) → allow | deny
- 선행 조건: 문서 민감도 분류 시스템 도입 (Phase 8+ 또는 Phase 12)
```

### 11-5. Rate/Volume-based Restriction

```
[연결 위치: Step 8]
- 예: 10분 내 50건 이상 export 시도 → 추가 확인 요구
- 예: 특정 IP에서 단시간 대량 문서 열람 → 경고
- 훅: rate_check(actor_id, action, time_window) → allow | throttle | deny
- 입력: Activity Trace에서 수집된 행위 빈도 데이터 활용
```

### 11-6. 현재 정책 구조에서의 확장 포인트 요약

```
Step 3. Global Platform Policy
  └─► [3.5] Time-based Policy Hook (확장)
  └─► [3.5] Source/Environment Policy Hook (확장)

Step 7. Resource State Restriction
  └─► [7.5] Sensitivity-based Policy Hook (확장)
  └─► [7.5] Approval Workflow Gate Hook (확장)

Step 8. Sensitive Operation Guard
  └─► [8.5] Rate/Volume Policy Hook (확장)
  └─► [8.5] Multi-stage Approval Hook (확장)
```

---

## 12. Audit / Activity 연계 기준

### 12-1. 정책 거부/예외 허용 시 Audit 기록

| 정책 단계 | 거부 시 AuditLog 기록 내용 |
|---|---|
| L2. 주체 비활성 | `denial_reason: principal_inactive`, user_id, membership_id |
| L4. 조직 경계 | `denial_reason: scope_mismatch`, target_org_id, actor_org_id |
| L7. 상태 제한 | `denial_reason: resource_state_restricted`, resource_state, action |
| L8. 고위험 가드 | allow 시에도 `policy_reason` 포함. before/after snapshot |
| Break-glass | `authorization_path: break_glass`, all fields + 사유 |

### 12-2. 상태 기반 제한 denial_reason 코드

| 상태 | denial_reason |
|---|---|
| locked | `resource_locked` |
| archived | `resource_archived` |
| soft_deleted | `resource_deleted` |
| pending_approval | `resource_pending_approval` |
| read_only | `resource_read_only` |
| external_shared 외부 수정 시도 | `external_share_write_denied` |

### 12-3. Break-glass / Override의 Activity Trace 기록

Break-glass 접근은 감사 로그(필수)뿐 아니라 Activity Trace에도 기록한다:
- `event_type: "admin.break_glass.accessed"`
- trace_id로 이후 접근 행위 전체 묶음

### 12-4. 비정상 시도 패턴 해석

Activity Trace + Audit Log 조합으로 감지 가능한 패턴:

| 패턴 | 감지 방법 |
|---|---|
| 반복 권한 거부 후 성공 | AuditLog `denial_reason: rbac_insufficient_role` 반복 → 이후 `allowed: true` |
| 비통상적 대량 접근 | ActivityTrace `document.opened` 이벤트 단기간 급증 |
| 권한 변경 직후 즉각 행위 | AuditLog `permission.acl.created` + 즉시 이전 불가능하던 ActivityTrace 이벤트 |
| 대량 export 시도 | ActivityTrace `export.initiated` 반복 + AuditLog `document.exported` |
| 야간 비정상 접근 | ActivityTrace `occurred_at` 시간대 분석 |

---

## 13. MVP 정책 제한안

### 13-1. MVP 필수 정책 규칙

| 규칙 | 설명 |
|---|---|
| **인증/주체 선차단** | L1, L2 필수. 비활성 계정/Membership 즉시 차단 |
| **조직 경계 보호** | L4 필수. 타 조직 리소스 ACL 부여 불가 |
| **RBAC 기반 기본 권한** | L5 필수 |
| **ACL 예외 (제한적)** | L6. Space, Document 수준만. admin permission 제외 |
| **상태 기반 제한** | L7. locked, archived, soft_deleted, pending_approval 상태 제한 |
| **고위험 액션 사유 기록** | L8. 권한/역할/ACL 변경, API Credential, Organization Owner 변경 시 사유 필수 |
| **고위험 액션 강화 감사** | Permission Policy 변경, break-glass 등은 full snapshot 감사 |
| **Default Deny 원칙** | 어떤 경로로도 allow를 얻지 못하면 거부 |

### 13-2. 확장 단계 정책 규칙

| 규칙 | 도입 시점 |
|---|---|
| ACL deny 지원 | Phase 3 이후 (필요 시) |
| Time-based policy | Phase 13 보안 체계 |
| Rate/Volume-based policy | Phase 8+ 또는 Phase 13 |
| Sensitivity label 기반 세밀 제어 | Phase 12 문서 유형 확장 시 |
| Multi-stage approval gate | Phase 5 워크플로 도입 시 |
| Break-glass 자동 만료 | Phase 13 |
| 영구 삭제 기능 | Phase 13 (보존 정책 확정 후) |

---

## 14. 다음 Task로 넘길 결정사항

### Task 2-8 (Phase 2 통합 설계서 정리) 에 전달

**정책 평가 순서 최종안:**
> L1(인증) → L2(주체 상태) → L3(플랫폼 정책) → L4(조직 경계) → L5(RBAC) → L6(ACL) → L7(상태 제한) → L8(고위험 가드) → 최종 결정

**상태 기반 제한 핵심 규칙:**
- locked: 모든 수정 차단. ACL/역할 우회 불가
- archived: 수정/게시 차단. 조회/export 허용
- soft_deleted: 일반 사용자 숨김. Admin만 조회/복원
- pending_approval: 수정 차단. 워크플로 관리자만 취소 후 수정 가능

**관리자 권한 경계 규칙:**
- PlatformAdmin = 플랫폼 운영. 콘텐츠 자동 접근 없음
- SecurityAuditor = AuditLog 읽기만. 콘텐츠 없음
- OrganizationAdmin = 자기 조직 범위 최고 권한
- Break-glass = PlatformOwner만. 사유 필수. 강화 감사

**ACL 예외 허용 한계:**
- 조직 경계/상태 제한/admin permission은 ACL로 우회 불가
- Space, Document 수준 read/update/share만 허용

**고위험 액션 가드 원칙:**
- MVP: 사유 기록 + 강화 감사
- 확장: 재인증, 2단계 승인, 자동 만료

**감사/행위추적 연결 정책 metadata:**
- 모든 정책 거부에 `denial_reason` 코드 포함
- 고위험 액션은 `policy_reason` 필드 필수
- break-glass는 별도 `authorization_path: break_glass` 기록

---

## 15. 오픈 이슈

| # | 이슈 | 설명 | 결정 필요 시점 |
|---|---|---|---|
| OI-1 | PlatformAdmin의 조직 AuditLog 접근 범위 | 조직 범위 AuditLog 읽기를 PlatformAdmin에게 자동 부여할지, 조직별 Membership이 필요한지 | Phase 3 API 구현 시 |
| OI-2 | Lock 자동 해제 조건 | 문서 승인 완료 시, 또는 lock 설정자 퇴사 시 lock을 자동 해제하는 조건 정의 필요 | Phase 5 워크플로 설계 시 |
| OI-3 | soft_deleted 보존 기간 | 삭제 후 얼마나 보존할지 (30일? 90일? 영구?). 보존 기간 이후 자동 영구 삭제 여부 | Phase 13 데이터 보존 정책 수립 시 |
| OI-4 | 조직 삭제 시 Membership/ACL/AuditLog 처리 | 조직 삭제 또는 아카이브 시 하위 데이터 처리 정책 (즉시 삭제? 유예기간? 감사 보존?) | Phase 13 |
| OI-5 | pending_approval에서 긴급 수정 시나리오 | 오탈자 수정 등 긴급한 경우 승인 대기 중 수정 허용 여부. 재제출 vs 즉시 수정 | Phase 5 워크플로 설계 시 |

---

*작성일: 2026-04-01*
*Phase: 2 / Task: 2-7*
*상태: 설계 초안 완료*
