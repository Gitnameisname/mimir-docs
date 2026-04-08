# Phase 2 - Task 2-1. 권한/역할/조직 도메인 모델 정의

---

## 1. 목표

Phase 2 권한 체계의 출발점으로, 문서 플랫폼에서 권한 판단의 기준이 되는 핵심 주체들을 정의하고 그 관계 구조를 확정한다.

- 권한 체계를 구성하는 핵심 엔티티(주체)를 식별하고 각각의 책임을 정의한다.
- 사용자/조직/멤버십/역할 간 관계 구조를 확정한다.
- 전역 역할과 조직 역할의 분리 원칙을 수립한다.
- 역할 부여의 기본 단위를 결정한다.
- 팀/그룹, 서비스 계정의 도입 시점을 판단한다.
- Task 2-2(리소스 범위 정의), 2-3(ACL 설계), 2-4(API enforcement)의 설계 기준선을 제공한다.

---

## 2. 설계 전제

| # | 전제 |
|---|---|
| 1 | 범용 문서 플랫폼. 규정집 전용이 아니라 모든 유형의 문서를 다룬다 |
| 2 | API-first 구조. 외부 애플리케이션 및 AI 도구 연계를 기본으로 한다 |
| 3 | User UI와 Admin UI는 분리되므로 관리자 역할과 일반 사용자 역할의 경계가 명확해야 한다 |
| 4 | Document / Version / Node / Attachment / Comment / Workflow 등 다양한 리소스가 존재하므로 역할 모델은 확장 가능해야 한다 |
| 5 | 단순 RBAC으로 끝내지 않는다. RBAC → resource-scoped assignment → ACL 확장 경로를 보장한다 |
| 6 | 감사 추적 친화적 구조. 누가 어떤 자격으로 행동했는지 항상 추적 가능해야 한다 |

---

## 3. 권한 주체 후보 분석

### 3-1. 후보 엔티티 검토

#### User (사용자)

| 항목 | 내용 |
|---|---|
| **왜 필요한가** | 모든 권한 판단의 최종 주체. 인증(Authentication) 후 식별되는 실제 인간 사용자 |
| **책임** | 인증 자격 증명 보유, 플랫폼 접근의 실질적 주체, 행위 감사 추적의 귀착점 |
| **다른 엔티티와의 관계** | Membership을 통해 Organization에 속함. 전역 역할을 직접 보유할 수 있음 |
| **MVP 필요 여부** | **필수** — 권한 체계의 기반 주체 |

#### Organization (조직)

| 항목 | 내용 |
|---|---|
| **왜 필요한가** | 문서 및 리소스를 소유하는 테넌트 단위. 여러 사용자가 협업하는 최상위 단위 |
| **책임** | 리소스(문서, 공간 등)의 소유 단위. 조직 내 역할 부여의 범위 경계 |
| **다른 엔티티와의 관계** | Membership을 통해 User를 포함. Organization 단위 Role 할당의 범위 |
| **MVP 필요 여부** | **필수** — 멀티테넌트 구조를 위한 핵심 경계 단위 |

#### Membership (멤버십)

| 항목 | 내용 |
|---|---|
| **왜 필요한가** | User와 Organization 사이의 소속 관계를 표현하는 연결 엔티티. 역할 부여의 실질적 단위 |
| **책임** | 특정 조직 내 사용자의 소속 상태 관리, 조직 범위 역할 보유, 초대/탈퇴/상태 추적 |
| **다른 엔티티와의 관계** | User ↔ Organization을 연결. Role을 직접 보유. 감사 추적 시 "어떤 자격으로 행동했는지"의 기준 |
| **MVP 필요 여부** | **필수** — 역할 부여의 핵심 단위 |

#### Role (역할)

| 항목 | 내용 |
|---|---|
| **왜 필요한가** | 권한 집합에 의미 있는 이름을 부여하는 분류 단위. 권한 관리의 기본 추상화 |
| **책임** | 권한 집합을 의미 있는 단위로 그룹화. 역할 범위(전역/조직) 구분. 시스템 정의 vs 사용자 정의 구분 |
| **다른 엔티티와의 관계** | Membership에 할당되거나, User에 전역 역할로 직접 부여됨 |
| **MVP 필요 여부** | **필수** — 권한 체계의 기본 추상화 단위 |

#### Team / Group (팀/그룹)

| 항목 | 내용 |
|---|---|
| **왜 필요한가** | 조직 내 사용자 집합에 한 번에 역할을 부여하기 위한 간접 주체 |
| **책임** | 여러 사용자를 논리 단위로 묶어 역할 관리 효율화. 팀 단위 리소스 접근 제어 |
| **다른 엔티티와의 관계** | Organization에 속함. User를 멤버로 포함. Role 할당의 주체가 될 수 있음 |
| **MVP 필요 여부** | **확장용** — 대규모 조직 운영에 필요하지만 MVP 핵심 흐름에 필수는 아님. Phase 2 이후 도입 권장 |

#### Service Account / API Client

| 항목 | 내용 |
|---|---|
| **왜 필요한가** | 외부 애플리케이션, AI 에이전트, 자동화 파이프라인이 플랫폼 API를 호출할 때 필요한 비인간 주체 |
| **책임** | API 인증 자격 증명 보유. 비인간 행위자의 감사 추적 귀착점. 제한된 권한 범위로 운영 |
| **다른 엔티티와의 관계** | Organization에 귀속되거나 전역 수준. 역할 할당의 주체 |
| **MVP 필요 여부** | **확장용** — API-first 설계상 중요하지만, 초기에는 User 토큰 기반으로 대체 가능. Phase 3 API 계층 설계 시 도입 권장 |

### 3-2. 주체 후보 요약

| 엔티티 | MVP 필요 | 확장 단계 |
|---|---|---|
| User | 필수 | — |
| Organization | 필수 | — |
| Membership | 필수 | — |
| Role | 필수 | — |
| Team / Group | 확장용 | Phase 2 후속 또는 Phase 7 관리자 UI |
| Service Account | 확장용 | Phase 3 API 계층 설계 시 |

---

## 4. 관계 모델 초안

### 4-1. 핵심 관계 구조

```
User (1) ──────── (N) Membership (N) ──────── (1) Organization
                        │
                        │ (N)
                        ▼
                      Role  ←── scope_type: organization | global

User (N) ──── GlobalRoleAssignment (N) ──── Role (scope_type: global)

Organization (1) ──── (N) Team
Team (N) ──────── TeamMembership (N) ──────── User
Team (N) ──────── TeamRoleAssignment (N) ──────── Role
```

### 4-2. 관계 설계 핵심 질문 답변

**Q. 사용자는 여러 조직에 속할 수 있는가?**

> 예. User는 복수의 Membership을 통해 여러 Organization에 동시에 속할 수 있다.
> 각 Membership은 독립적이며, 조직별로 다른 역할을 가질 수 있다.

**Q. 조직 안에서 사용자에게 복수 역할 부여가 가능한가?**

> 가능하다. 단, Membership에 다중 Role을 직접 부여하는 방식(MembershipRole 조인 테이블)으로 구현한다.
> 역할 충돌 시 평가 우선순위는 Task 2-7에서 정의한다.

**Q. 역할은 사용자에 직접 붙는가, Membership에 붙는가?**

> **원칙: 조직 범위 역할은 Membership에 붙는다.**
> User에 직접 붙는 역할은 전역 역할(PlatformOwner, PlatformAdmin, SecurityAuditor 등)에 한정한다.
> 이렇게 분리하면 "어느 조직에서의 어떤 역할인가"를 명확히 추적할 수 있다.

**Q. 팀/그룹은 초기 MVP에 필요한가, 후순위인가?**

> 후순위다. MVP에서는 User → Membership → Role 경로로 충분하다.
> 팀/그룹 도입 시에도 Team → TeamMembership → User 구조가 기존 모델을 깨지 않고 추가된다.

**Q. 서비스 계정은 초기부터 고려해야 하는가?**

> 모델 설계에는 반영하지 않되, 인터페이스 수준에서 "Principal" 개념으로 추상화하여 확장 가능성을 열어둔다.
> User와 ServiceAccount는 동일한 Principal 인터페이스를 만족하는 형태로 설계하면, ACL 평가 로직이 주체 유형에 독립적으로 동작한다.
> 실제 ServiceAccount 엔티티는 Phase 3 API 계층 설계 시 도입한다.

---

## 5. 역할 체계 초안

### 5-1. 전역 역할 (Global Role)

전역 역할은 특정 조직에 귀속되지 않고 플랫폼 전체에 적용된다. 플랫폼 운영 및 보안 관리 목적이다.

| 역할 | 적용 범위 | 권한 성격 | 다른 역할과의 차이 |
|---|---|---|---|
| **PlatformOwner** | 플랫폼 전체 | 플랫폼 최고 권한. 모든 설정 및 관리자 계정 관리 가능 | 단 1명 또는 소수. 양도 가능. 시스템 부트스트랩 수준 |
| **PlatformAdmin** | 플랫폼 전체 | 플랫폼 운영 전반 관리. 조직 생성/삭제, 시스템 설정 | PlatformOwner보다 권한 범위 좁음. 복수 할당 가능 |
| **SecurityAuditor** | 플랫폼 전체 | 감사 로그 조회, 권한 변경 이력 확인. 읽기 전용 | 수정 권한 없음. 보안/컴플라이언스 전담 |
| **SupportOperator** | 플랫폼 전체 | 사용자 지원 목적의 제한적 조회 권한 | 감사 로그보다 좁은 범위. 내용 수정 불가 |

> 전역 역할은 User에 **직접 할당**된다 (GlobalRoleAssignment).
> 전역 역할 보유자도 특정 조직 내 문서에 접근하려면 해당 조직 Membership이 필요하거나, 전역 역할의 override 정책이 적용되어야 한다. (정책 우선순위는 Task 2-7에서 정의)

### 5-2. 조직 역할 (Organization-scoped Role)

조직 역할은 Membership에 할당되며, 해당 조직 내 리소스에만 적용된다.

| 역할 | 적용 범위 | 권한 성격 | 다른 역할과의 차이 |
|---|---|---|---|
| **OrganizationOwner** | 조직 전체 | 조직 내 최고 권한. 멤버 관리, 역할 부여, 조직 설정 | 조직 생성자가 자동 부여. 양도 가능 |
| **OrganizationAdmin** | 조직 전체 | 조직 운영 관리. 멤버 초대/제거, 공간 생성, 역할 관리 | OrganizationOwner보다 권한 좁음. 복수 할당 가능 |
| **SpaceManager** | 공간(Workspace/Space) 단위 | 특정 공간 내 문서 구조, 멤버, 설정 관리 | 조직 전체가 아닌 공간 단위로 제한됨 |
| **Editor** | 문서/공간 단위 | 문서 작성, 수정, 버전 생성, 초안 관리 | 멤버 관리나 역할 부여 권한 없음 |
| **Commenter** | 문서 단위 | 코멘트 작성 및 수정(자기 것) | 문서 본문 수정 불가 |
| **Viewer** | 문서/공간 단위 | 읽기 전용. 게시된 문서 조회만 가능 | 모든 수정, 코멘트 불가 |

> 조직 역할은 Membership에 할당된다. 동일 Membership에 복수 역할 부여 가능.
> SpaceManager, Editor, Commenter, Viewer는 이후 ACL resource-scoped 확장 시 특정 공간/문서에 좁혀서 적용될 수 있다.

### 5-3. 역할 계층 개요

```
[전역 수준]
  PlatformOwner
    └─ PlatformAdmin
         ├─ SecurityAuditor (읽기 전용 감사)
         └─ SupportOperator (제한적 지원)

[조직 수준]
  OrganizationOwner
    └─ OrganizationAdmin
         └─ SpaceManager
              ├─ Editor
              │    └─ Commenter
              └─ Viewer
```

계층은 논리적 상하관계를 나타내며, 상위 역할이 하위 역할의 권한을 포함한다는 단순 계승(inheritance)이 아니다. 실제 권한 평가는 Permission 집합 + ACL 정책으로 결정되며, 역할 계층은 참조 구조일 뿐이다.

---

## 6. 역할 부여 방식 비교

### 6-1. 세 가지 방식 비교

| 비교 항목 | A. 사용자 직접 부여 | B. Membership에 부여 (권장) | C. Team/Group 간접 부여 |
|---|---|---|---|
| **단순성** | 높음 — 구조 단순 | 중간 — Membership 중간 계층 필요 | 낮음 — Team, TeamMembership 추가 필요 |
| **관리 편의성** | 낮음 — 조직별 역할 파악이 어려움 | 높음 — 조직별 역할 명확히 분리됨 | 매우 높음 — 팀 단위로 일괄 관리 가능 |
| **확장성** | 낮음 — 다조직 시나리오에서 혼란 | 높음 — 다조직을 명확히 지원 | 매우 높음 — 대규모 조직에 적합 |
| **ACL 연결 용이성** | 낮음 — principal이 User뿐이라 ACL 설계 단순하나, 조직 컨텍스트 파악이 어려움 | 높음 — ACL principal을 Membership으로도 표현 가능. 컨텍스트 명확 | 높음 — Team을 ACL principal로 추가하면 확장 가능 |
| **API enforcement 적합성** | 중간 — 토큰에서 User ID만 추출하면 됨 | 높음 — 요청 컨텍스트에 Organization ID 포함 시 Membership 기반 검사 자연스러움 | 중간 — Team 해석 추가 비용 |
| **감사 추적** | 중간 — "User X가 했다"는 알 수 있으나 "어느 조직 자격인지" 불명확 | 높음 — Membership을 통해 "A 조직의 Editor 자격으로 행동했음" 추적 가능 | 높음 — Team 포함 시 더욱 풍부한 추적 가능 |

### 6-2. 최종 권장안

**권장: B. Membership 중심 역할 부여 (+ 전역 역할은 User 직접 부여)**

구체적 원칙:
1. **조직 범위 역할** → Membership에 부여 (MembershipRole 조인)
2. **전역 역할** → User에 직접 부여 (GlobalRoleAssignment)
3. **팀/그룹 역할** → Phase 2 이후 Team에 부여하고 Membership 경로와 병합 평가

이 방식을 선택하는 이유:
- 다조직 지원이 명확하다 (User가 조직 A에서 Editor, 조직 B에서 Viewer일 수 있다)
- 감사 추적 시 "어느 조직에서 어떤 자격으로"를 Membership이 표현한다
- 전역 역할(PlatformAdmin 등)과 조직 역할의 평가 경로가 분리되어 보안 관리가 용이하다
- ACL 확장 시 principal을 User ID + Membership ID로 표현 가능하며, 이후 Team ID로도 확장된다

---

## 7. 권장 도메인 모델

### 7-1. 개념 필드 수준 모델

#### User

```
User
  - id                      (고유 식별자, UUID)
  - email                   (인증 이메일, 유일)
  - display_name            (표시 이름)
  - status                  (active | suspended | deleted)
  - auth_provider           (local | oauth | sso 등 — 확장 고려)
  - created_at
  - last_login_at
```

#### Organization

```
Organization
  - id                      (고유 식별자, UUID)
  - name                    (조직명)
  - slug                    (URL 식별용 단축명, 유일)
  - status                  (active | suspended | archived)
  - plan_type               (향후 플랜 확장 대비 — 확장 필드)
  - created_at
```

#### Membership

```
Membership
  - id                      (고유 식별자)
  - user_id                 → User
  - organization_id         → Organization
  - status                  (active | invited | suspended | left)
  - invited_at
  - joined_at
  - updated_at
  [복합 유일: user_id + organization_id]
```

#### Role

```
Role
  - id                      (고유 식별자)
  - name                    (역할명, 예: PlatformAdmin, Editor)
  - scope_type              (global | organization | space | document)
  - system_defined          (boolean — 시스템 내장 역할 여부)
  - description
  - created_at
```

> scope_type은 이 역할이 어느 수준에서 유효한지를 나타낸다. MVP에서는 global, organization만 사용.
> space, document 수준은 ACL 확장 시 활용한다.

#### GlobalRoleAssignment (전역 역할 할당)

```
GlobalRoleAssignment
  - id
  - user_id                 → User
  - role_id                 → Role (scope_type: global)
  - assigned_by             → User
  - assigned_at
```

#### MembershipRole (조직 역할 할당)

```
MembershipRole
  - id
  - membership_id           → Membership
  - role_id                 → Role (scope_type: organization 이상)
  - assigned_by             → User
  - assigned_at
```

#### Team / Group (확장 예약)

```
Team
  - id
  - organization_id         → Organization
  - name
  - description
  - status                  (active | archived)
  - created_at

TeamMembership
  - id
  - team_id                 → Team
  - user_id                 → User
  - joined_at

TeamRoleAssignment
  - id
  - team_id                 → Team
  - role_id                 → Role
  - assigned_by             → User
  - assigned_at
```

### 7-2. 개념 관계도

```
User (1) ────────── (N) Membership (N) ────────── (1) Organization
                          │                               │
                     MembershipRole                    (N) Team
                          │                               │
                         Role                      TeamMembership
                          │                               │
                     (scope_type)                      User
                  global | organization

User (1) ──── (N) GlobalRoleAssignment ──── (1) Role (scope_type: global)

[향후 확장]
Principal (User | ServiceAccount)
  └─ ACL entry: principal_id, resource_id, resource_type, permission, effect
```

---

## 8. 권장 도메인 원칙

### 원칙 1. 역할은 Membership 중심으로 부여한다

역할은 사람 자체보다 "어느 조직에 어떤 자격으로 속하는가"에 붙어야 한다. User에 직접 붙는 역할은 플랫폼 전역 관리 역할(PlatformOwner, PlatformAdmin, SecurityAuditor)에만 한정한다. 이는 다조직 지원, 감사 추적, 역할 관리 명확성을 동시에 확보한다.

### 원칙 2. 전역 관리자 권한과 조직 관리자 권한은 반드시 분리한다

플랫폼 운영 권한(조직 생성, 전체 사용자 관리, 시스템 설정)과 조직 내부 운영 권한(문서 관리, 멤버 초대, 공간 설정)은 서로 다른 역할 계층에서 관리한다. 조직 관리자(OrganizationOwner)가 플랫폼 전체를 관리할 수 없고, 플랫폼 관리자(PlatformAdmin)가 자동으로 조직 내부 리소스에 접근할 수 없다.

### 원칙 3. 팀/그룹은 MVP 필수가 아니라 확장 단계에서 도입한다

초기에는 User → Membership → Role 경로로 충분하다. Team/Group은 대규모 조직 운영 시 관리 효율을 위해 도입하며, 기존 모델을 깨지 않고 추가할 수 있는 구조로 설계한다.

### 원칙 4. 서비스 계정은 Phase 3 API 계층 설계 시 도입한다

API-first 구조상 서비스 계정은 반드시 필요하지만, MVP에서는 User 토큰으로 대체 가능하다. 설계 시 "Principal = User | ServiceAccount"로 추상화하여, ACL 평가 로직이 주체 유형에 의존하지 않는 구조를 미리 준비한다.

### 원칙 5. ACL 확장 시 현재 모델의 장점

- **Membership이 ACL principal이 될 수 있다**: "조직 A 내 User X의 Membership"을 principal로 삼으면, 조직 컨텍스트를 포함한 세밀한 권한 부여가 가능하다.
- **Role의 scope_type 필드**: 역할이 전역 수준인지, 조직 수준인지, 공간 수준인지를 구분하여 ACL 평가 범위를 명확히 한다.
- **Team 구조의 사전 예약**: ACL principal에 Team ID를 추가하는 것이 기존 구조를 변경하지 않고 확장 가능하다.
- **감사 추적 귀착점 명확**: Membership ID + Role ID 조합으로 "어떤 자격으로 행동했는지"가 항상 기록 가능하다.

---

## 9. 오픈 이슈

| # | 이슈 | 설명 | 결정 필요 시점 |
|---|---|---|---|
| OI-1 | 전역 역할 보유자의 조직 리소스 접근 여부 | PlatformAdmin이 자동으로 모든 조직 문서에 접근 가능한가, 아니면 별도 Membership이 필요한가 | Task 2-3 ACL 설계 또는 Task 2-7 정책 우선순위 |
| OI-2 | Role이 Permission 집합을 직접 포함하는가, 아니면 별도 관리되는가 | RBAC 구현 방식: Role ↔ Permission 명시적 조인 테이블 vs 역할명만 코드에서 해석 | Task 2-3 ACL 설계 |
| OI-3 | Membership 상태(invited, suspended)와 역할 유효성 연동 | invited 상태의 Membership에 부여된 역할이 유효한가 | Task 2-3 또는 Task 2-7 |
| OI-4 | 조직 삭제 시 Membership 및 역할 처리 | 조직 아카이브/삭제 시 연결된 Membership과 MembershipRole을 어떻게 처리하는가 | Task 2-7 정책 우선순위 |
| OI-5 | SpaceManager 역할의 Space 개념 연결 | Space(공간) 엔티티는 아직 확정되지 않음. Task 2-2 리소스 범위 정의 후 역할과 연결 필요 | Task 2-2 |

---

## 10. 다음 Task로 넘길 결정사항

### Task 2-2 (리소스 범위와 보호 대상 식별) 에 전달

- 권한 주체 목록 확정: User, Organization, Membership, Role, (Team: 확장)
- 역할의 scope_type: global, organization → space, document 수준 확장 예정
- Membership이 "조직 내 접근 주체"의 기준 단위임
- OI-1: 전역 역할 보유자의 조직 리소스 접근 범위는 리소스 계층 정의 후 결정

### Task 2-3 (ACL 기반 권한 모델 설계) 에 전달

- principal = User | Membership | Team | ServiceAccount (추상화된 Principal 개념)
- Role의 scope_type으로 ACL 평가 범위 제한
- 전역 역할은 GlobalRoleAssignment, 조직 역할은 MembershipRole 경로로 분리됨
- OI-2: Permission 집합 관리 방식 결정 필요
- OI-3: Membership 상태에 따른 역할 유효성 결정 필요

### Task 2-4 (API Authorization Enforcement) 에 전달

- API 요청 컨텍스트: User ID + Organization ID → Membership 조회 → 역할 평가
- 전역 역할은 Organization 컨텍스트 없이도 평가 가능
- ServiceAccount 도입 시 동일한 평가 흐름 적용 가능하도록 Principal 추상화 권장

### Task 2-7 (정책 우선순위와 예외 규칙) 에 전달

- OI-1: PlatformAdmin의 전체 조직 접근 override 정책
- OI-3: Membership 상태와 역할 유효성 연동 규칙
- OI-4: 조직 삭제/아카이브 시 Membership/역할 처리 정책

---

*작성일: 2026-04-01*
*Phase: 2 / Task: 2-1*
*상태: 설계 초안 완료*
