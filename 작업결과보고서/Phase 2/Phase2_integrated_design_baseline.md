# Phase 2. 권한, 역할, 감사 추적 체계 설계 — 통합 기준선 문서

---

## 1. 문서 목적

이 문서는 Task 2-1부터 2-7까지의 설계 결과를 하나의 일관된 기준선으로 통합한 Phase 2 최종 설계 문서다.

이 문서는 다음 목적으로 사용된다.

- 구현 착수 전 모든 팀이 공유하는 권한/감사/추적 체계의 **단일 기준점**
- 구현 중 "어떻게 해야 하는가"를 판단할 때 참조하는 **의사결정 근거**
- Phase 3 이후 확장 결정 시 "Phase 2에서 무엇을 열어뒀는가"를 확인하는 **연결 포인트 지도**

이 문서는 새로운 설계를 추가하지 않는다. Task 2-1~2-7의 결론을 정리하고, 용어를 통일하고, 상호 충돌을 제거하고, MVP 기준선을 명확히 하는 데 집중한다.

---

## 1-A. Phase 1 → Phase 2 연결점

Phase 2의 권한 체계는 Phase 1에서 확정된 문서 도메인 위에 얹히는 구조다. 아래 연결점을 이해해야 Phase 2 설계가 실제 도메인 모델과 정합성을 유지할 수 있다.

### Phase 1 핵심 엔티티와 Phase 2 권한 연결

| Phase 1 엔티티 | Phase 1 정의 | Phase 2 권한 적용 방식 |
|---|---|---|
| **Document** | Identity shell. current_version_id 포인터를 보유하는 문서 껍데기 | 권한 통제의 핵심 단위. `document:read`, `document:update` 등 Permission의 주 대상. Space 하위에 귀속됨 |
| **DocumentVersion** | 불변 스냅샷(Immutable Snapshot). Document의 특정 시점 내용 | Document에서 권한 상속. Draft 상태인 Version은 `document:read_draft` 추가 권한 필요 |
| **DocumentNode** | 트리 구조 단위. Version의 본문을 구성하는 재귀 구조 | Version에서 권한 상속. MVP에서 독립 ACL 없음. 향후 노드 단위 편집 제어 시 확장 |
| **DocumentType** | 문서 유형 정의 (generic + config 기반). 하드코딩 금지 | 플랫폼/조직 관리자 전용 관리 리소스. 타입 변경은 강화 감사 대상 |
| **DocumentStatus** | 문서 상태 enum (draft, published, archived 등) | Phase 2의 Resource State와 동일 개념. 상태 기반 제한(Step 7)의 직접 입력값 |

### 핵심 설계 원칙 연결

```
[Phase 1]                         [Phase 2]
Document (identity shell)    →    resource_type: "document", resource_id
  └── current_version_id     →    post-load 시 Version 상태까지 확인
Version (immutable)          →    update = 새 Version 생성. 기존 Version 수정 불가
  └── Node (tree)            →    Node 접근 = Version 권한 상속
DocumentStatus               →    resource_state → Step 7 상태 제한 입력
DocumentType                 →    type-aware 로직은 서비스 레이어에서만 분기
```

> **중요**: Document를 update할 때 기존 Version을 수정하지 않고 **새 Version을 생성**한다. 이 불변 원칙이 감사 로그에서 "변경 전/후 version_id"를 기록하는 근거다.

---

## 2. Phase 2 목표 요약

### 2-1. 왜 이 플랫폼에 권한/감사/행위추적 체계가 필요한가

이 플랫폼은 범용 문서 플랫폼이다. 단일 사용자의 개인 저장소가 아니라 **여러 조직, 여러 역할의 사람들이 공동으로 문서를 생성·관리·검토하는 환경**이다. 이 환경에서는 다음이 필수다.

- **누가 어떤 문서에 접근할 수 있는가** (권한 통제)
- **누가 무엇을 했는가** (감사 추적)
- **어떻게 작업이 이루어졌는가** (행위 흐름 추적)

이 세 가지 없이는 문서의 기밀성, 무결성, 책임 추적이 불가능하다.

### 2-2. 단순 RBAC만으로 충분하지 않은 이유

RBAC만으로는 다음 시나리오를 처리할 수 없다.

- Editor 역할이지만 **특정 기밀 문서만** 열람 불가
- 외부 사용자에게 **특정 문서 하나**만 공유
- AI 에이전트(Service Account)에게 **특정 리소스만** 허용
- 문서가 **locked/archived 상태**일 때는 편집 역할이 있어도 차단

이를 처리하려면 RBAC 위에 **리소스 단위 ACL 예외 계층**이 필요하다.

### 2-3. 감사와 Activity Trace를 왜 동시에 설계했는가

두 체계는 **목적이 완전히 다르다**.

- **감사 로그**: "무슨 일이 있었는가" — 보안/컴플라이언스 증거. 법적 기준 장기 보존.
- **Activity Trace**: "어떻게 작업했는가" — 흐름/맥락 분석. UX 개선, AI 행위 이해.

같은 구현 시점에 설계하지 않으면, 나중에 두 시스템의 수집 범위, 보존 정책, 접근 통제가 뒤섞인다. Phase 2에서 경계를 명확히 설계해야 이후 확장에서 혼용을 방지할 수 있다.

### 2-4. 구현 전에 왜 통합 설계 문서가 필요한가

- 7개 Task의 결론이 분산되어 있으면 구현자가 전체 그림을 파악하기 어렵다
- 용어와 구조가 문서마다 조금씩 달라지면 코드에서 불일치가 발생한다
- MVP와 확장의 경계가 불분명하면 초기 구현이 과도하게 복잡해지거나, 반대로 나중에 부수어야 하는 구조가 생긴다

### 2-5. Phase 2 핵심 목표 (7개)

| # | 목표 |
|---|---|
| G1 | Membership 중심 RBAC + 제한적 resource-scoped ACL로 예측 가능한 권한 체계 구축 |
| G2 | 조직 경계(Organization Boundary)를 최상위 보호막으로 엄격히 유지 |
| G3 | Authorization Service 중앙화로 모든 API 요청에 일관된 권한 판단 보장 |
| G4 | 감사 로그(Audit Log)로 보안/컴플라이언스 증적 체계 구축 |
| G5 | Activity Trace로 사용자/AI 행위 흐름 분석 기반 마련 |
| G6 | 감사 로그와 Activity Trace의 목적과 구조를 분리하여 혼용 방지 |
| G7 | MVP는 단순하고 예측 가능하게. 후속 확장 경로를 막지 않는 구조 유지 |

---

## 3. 핵심 개념 사전 (Glossary)

이 사전에 정의된 용어는 코드, API, DB 스키마, 문서 전반에서 동일하게 사용한다. 동의어나 유사 표현으로 흔들리지 않도록 한다.

---

### Principal (주체)

| 항목 | 내용 |
|---|---|
| **정의** | 권한 평가의 대상이 되는 행위 주체의 추상 개념. 인간 사용자, 비인간 시스템, 팀을 모두 포함 |
| **유사 용어와 차이** | User는 Principal의 한 종류. Principal이 상위 개념 |
| **MVP 여부** | 필수 개념 (User + Membership이 MVP의 실질 Principal) |

---

### User (사용자)

| 항목 | 내용 |
|---|---|
| **정의** | 플랫폼에 인증된 실제 인간 사용자. 모든 행위 감사의 귀착점 |
| **유사 용어와 차이** | Principal(상위 추상), Member(Membership을 통해 조직에 소속된 User를 비공식적으로 지칭) |
| **MVP 여부** | 필수 |

---

### Organization (조직)

| 항목 | 내용 |
|---|---|
| **정의** | 리소스(문서, 공간 등)의 소유 단위이자 멀티테넌트 경계 단위. 조직 간 데이터 분리의 기준 |
| **유사 용어와 차이** | Tenant(동의어로 사용 가능하나 이 문서에서는 Organization으로 통일) |
| **MVP 여부** | 필수 |

---

### Membership (멤버십)

| 항목 | 내용 |
|---|---|
| **정의** | User와 Organization 사이의 소속 관계를 표현하는 연결 엔티티. 조직 범위 역할 부여의 실질 단위 |
| **유사 용어와 차이** | User가 "사람 자체"라면 Membership은 "A 조직에 소속된 사람 자격". 감사 추적 시 "어떤 자격으로 행동했는지"는 Membership이 표현함 |
| **MVP 여부** | 필수 (역할 부여의 핵심 단위) |

---

### Role (역할)

| 항목 | 내용 |
|---|---|
| **정의** | 권한(Permission) 집합에 의미 있는 이름을 부여한 분류 단위. 전역(global)과 조직(organization) 범위로 구분 |
| **유사 용어와 차이** | Permission은 개별 행위 권한. Role은 Permission 집합의 묶음 이름 |
| **MVP 여부** | 필수 |

---

### Team / Group (팀/그룹)

| 항목 | 내용 |
|---|---|
| **정의** | 조직 내 사용자 집합에 역할을 일괄 부여하기 위한 중간 단위 |
| **유사 용어와 차이** | Department(비공식 조직 단위)와 다르게 Team은 역할/ACL 부여의 주체로 작동 |
| **MVP 여부** | 확장 개념. MVP에서 도입하지 않음 |

---

### Service Account (서비스 계정)

| 항목 | 내용 |
|---|---|
| **정의** | 외부 애플리케이션, AI 에이전트, 자동화 파이프라인이 API를 호출할 때 사용하는 비인간 주체 |
| **유사 용어와 차이** | Bot, API Client와 유사. 이 플랫폼에서는 Service Account로 통일 |
| **MVP 여부** | 확장 개념. MVP에서는 User 토큰으로 대체. Phase 3 API 계층 설계 시 도입 |

---

### Resource (리소스)

| 항목 | 내용 |
|---|---|
| **정의** | 권한 통제의 대상이 되는 모든 객체. Document, Space, Organization, AuditLog 등 포함 |
| **유사 용어와 차이** | Object, Entity와 유사하나 이 문서에서는 권한 대상을 지칭할 때 Resource로 통일 |
| **MVP 여부** | 필수 개념 |

---

### Scope (범위)

| 항목 | 내용 |
|---|---|
| **정의** | 권한이 유효한 리소스 계층 수준. Platform → Organization → Space → Document → Version → Node/Sub-resource |
| **유사 용어와 차이** | Level, Granularity와 유사. 이 문서에서는 Scope로 통일 |
| **MVP 여부** | 필수 개념 (구현에서는 Organization과 Document 수준이 핵심) |

---

### Permission (권한)

| 항목 | 내용 |
|---|---|
| **정의** | 특정 리소스에 대한 특정 행위를 허용하거나 거부하는 최소 단위. `{resource_type}:{action}` 형식 |
| **유사 용어와 차이** | Privilege, Right와 유사. 이 문서에서는 Permission으로 통일. 예: `document:read`, `org:manage_members` |
| **MVP 여부** | 필수 |

---

### ACL (Access Control List)

| 항목 | 내용 |
|---|---|
| **정의** | 특정 리소스에 대해 특정 Principal에게 특정 Permission을 직접 부여하는 레코드 집합. RBAC 외의 예외 허용 수단 |
| **유사 용어와 차이** | RBAC는 역할 단위 권한 관리. ACL은 리소스 단위 개별 부여. 이 플랫폼에서 ACL은 RBAC를 대체하지 않고 보완만 함 |
| **MVP 여부** | 필수 (Space, Document 수준에서만 허용) |

---

### Policy (정책)

| 항목 | 내용 |
|---|---|
| **정의** | 권한 평가에 적용되는 규칙의 총칭. Role 기반 정책, 상태 기반 정책, 고위험 가드, 조직 경계 규칙 등을 포함 |
| **유사 용어와 차이** | Rule, Constraint와 유사. 이 문서에서 Policy는 "평가 흐름에서 적용되는 모든 규칙"을 지칭 |
| **MVP 여부** | 필수 개념 (정책 엔진 자체는 확장) |

---

### Authorization (인가)

| 항목 | 내용 |
|---|---|
| **정의** | 인증된 Principal이 특정 Resource에 특정 Action을 수행할 권한이 있는지 판단하는 프로세스 |
| **유사 용어와 차이** | Authentication(인증, 신원 확인)과 다름. Authorization은 인증 이후 권한 판단 단계 |
| **MVP 여부** | 필수 |

---

### Audit Log (감사 로그)

| 항목 | 내용 |
|---|---|
| **정의** | 보안/컴플라이언스 목적으로 "누가, 언제, 어떤 자격으로, 무엇에, 어떤 행위를, 어떤 결과로 했는가"를 기록하는 append-only 이벤트 저장소 |
| **유사 용어와 차이** | Activity Trace(사용 흐름 분석 목적)와 명확히 분리. 애플리케이션 로그(디버깅 목적)와도 다름 |
| **MVP 여부** | 필수 (핵심 이벤트 중심) |

---

### Activity Trace (행위 추적)

| 항목 | 내용 |
|---|---|
| **정의** | 사용자/AI 에이전트가 리소스를 탐색·조회·편집하는 흐름과 맥락을 추적하는 분석 목적의 이벤트 모델 |
| **유사 용어와 차이** | Audit Log와 목적이 다름. Activity Trace는 UX 개선, 협업 패턴 분석, 이상 탐지 입력 목적. 법적 증거 목적이 아님 |
| **MVP 여부** | 필수 (핵심 이벤트 중심) |

---

### Trace ID (트레이스 ID)

| 항목 | 내용 |
|---|---|
| **정의** | 하나의 의미 있는 작업 흐름을 묶는 식별자. 검색 → 열람 → 편집 흐름처럼 연속된 행동 묶음의 공유 키 |
| **유사 용어와 차이** | Session ID(인증 세션 전체)보다 좁은 단위. 하나의 세션에 여러 Trace가 존재할 수 있음 |
| **MVP 여부** | 필수 (Activity Trace의 핵심 연결 단위) |

---

### Correlation ID (상관 ID)

| 항목 | 내용 |
|---|---|
| **정의** | 하나의 API 요청에서 생성되어 Audit Log, Activity Trace, API Request Log를 가로질러 공유되는 연결용 식별자. Task 2-4에서의 `request_id`와 동일 |
| **유사 용어와 차이** | Trace ID(작업 흐름 묶음)보다 작은 단위. 단일 요청 수준. Trace ID가 여러 Correlation ID를 포함할 수 있음 |
| **MVP 여부** | 필수 |

---

### Resource State (리소스 상태)

| 항목 | 내용 |
|---|---|
| **정의** | 리소스의 lifecycle 상태. draft, published, locked, archived, soft_deleted, pending_approval, read_only, external_shared 등 |
| **유사 용어와 차이** | Document Status(Phase 1 용어)와 동일한 개념. 이 문서에서 권한 평가 맥락에서는 Resource State로 부름 |
| **MVP 여부** | 필수 (locked/archived/soft_deleted 상태 제한은 MVP 필수) |

---

### Sensitive Action (고위험 액션)

| 항목 | 내용 |
|---|---|
| **정의** | 기본 권한이 있어도 추가 가드(사유 기록, 강화 감사 등)가 필요한 행위. 권한 정책 변경, 대량 export, API Credential 관리 등 |
| **유사 용어와 차이** | High-risk action, Privileged action과 유사. 이 문서에서는 Sensitive Action으로 통일 |
| **MVP 여부** | 필수 개념 (MVP에서는 사유 기록 + 강화 감사 수준) |

---

### Break-glass (긴급 접근)

| 항목 | 내용 |
|---|---|
| **정의** | 정상 권한 경로를 통해 접근할 수 없는 상황에서 PlatformOwner가 예외적으로 테넌트 경계를 넘어 접근하는 통제된 비상 절차 |
| **유사 용어와 차이** | Emergency access, Privileged access override와 유사. MVP에서는 정책 정의만 하고 자동화된 break-glass 실행은 확장 |
| **MVP 여부** | 정책 정의만 필수. 구현은 확장 |

---

## 4. 전체 아키텍처 관점 요약

Phase 2의 설계는 7개의 축으로 구성된다.

---

### 4-1. 주체(Principal) 계층

**핵심 역할**: 권한 판단의 주체를 식별하고, 그 주체가 어떤 조직적 맥락에서 행동하는지를 표현한다.

```
[전역 수준]
  User ──── GlobalRoleAssignment ──── Role (scope_type: global)

[조직 수준]
  User ──── Membership ──── MembershipRole ──── Role (scope_type: organization)
                │
                └── organization_id, status (active | invited | suspended | left)

[확장]
  Team ──── TeamMembership ──── User
  ServiceAccount (비인간 주체)
```

**다른 축과의 관계**: Principal은 Authorization Context에 담겨 Authorization Service에 전달된다. Audit Log의 Actor 섹션이 Principal을 기록한다.

**MVP 범위**: User + Membership + Role. Team과 ServiceAccount는 확장.

**확장 포인트**: ServiceAccount 도입 시 동일한 Principal 추상화 적용. Team 도입 시 ACL principal 확장.

---

### 4-2. 보호 대상(Resource) 계층

**핵심 역할**: 권한이 적용될 모든 객체를 Scope 계층으로 구조화한다.

```
Platform
  └── Organization
        └── Space
              └── Document
                    ├── Version (Draft / Published)
                    │     └── Node (상속)
                    ├── Attachment (상속)
                    └── Comment (상속)

[운영 리소스 — 별도 통제]
  AuditLog, ActivityTrace, Permission Policy, API Credential
```

**다른 축과의 관계**: Resource Scope는 ACL 부여 범위를 결정하고, Resource State는 정책 평가 L7 단계에서 추가 제한을 부여한다.

**MVP 범위**: Organization, Space, Document, Version이 핵심. Node/Comment/Attachment는 상속.

**확장 포인트**: Share Link/External Grant, Workflow/Approval Object, Metadata Schema.

---

### 4-3. 권한 모델(RBAC + ACL)

**핵심 역할**: Principal이 Resource에 대해 어떤 Permission을 가지는지를 두 경로로 표현한다.

```
경로 1 (RBAC): Membership → MembershipRole → Role → Permission 집합
경로 2 (ACL):  ACLEntry { principal, resource, permission, effect: allow }
```

두 경로는 "RBAC 1차 평가, ACL 2차 평가" 방식으로 결합된다. 기본은 RBAC로 처리하고, 예외만 ACL로 표현한다.

**다른 축과의 관계**: Permission 판단 결과가 Authorization Service의 최종 산출물이다.

**MVP 범위**: RBAC(Membership → Role → Permission), ACL(Space/Document 수준 allow만), deny는 확장.

**확장 포인트**: deny 효과, 조건부 ACL(condition), 만료 ACL(expires_at), Team ACL.

---

### 4-4. 정책 계층(Policy Layers)

**핵심 역할**: 권한 모델의 allow/deny 판단에 추가 제약을 겹쳐서 적용하는 다단계 필터.

```
L1. Authentication Validity
L2. Principal Active Status
L3. Global Platform Policy
L4. Organization Boundary
L5. RBAC (Role-based Permission)
L6. Resource-specific ACL
L7. Resource State Restriction
L8. Sensitive Operation Guard
L9. Final Decision
```

**다른 축과의 관계**: 정책 계층의 각 단계는 Authorization Service 내부 평가 흐름이다.

**MVP 범위**: L1~L7, L9 필수. L8은 부분(사유 기록 + 강화 감사). L3 글로벌 정책 엔진은 확장.

**확장 포인트**: L3.5 sensitivity label 정책, L7.5 time/rate 정책, L8.5 재인증/2단계 승인.

---

### 4-5. Enforcement 흐름

**핵심 역할**: API 요청이 들어왔을 때 권한 판단을 어느 계층에서, 어떤 순서로 수행하는지 정의한다.

```
HTTP Request
  → Authentication Middleware (토큰 검증, Principal 주입)
  → Router/Controller (리소스 식별, 입력 파싱)
  → Service Layer (Authorization Service 호출, 비즈니스 로직, 감사 트리거)
  → Authorization Service (RBAC + ACL + 정책 평가, allow/deny + metadata 반환)
  → Repository (authorization-aware query, 순수 데이터 조회)
```

**다른 축과의 관계**: Service Layer가 Authorization Service를 호출하고, 감사 이벤트 트리거도 Service Layer에서 발생한다.

**MVP 범위**: 전체 흐름 필수. 캐싱 전략은 확장.

**확장 포인트**: Authorization Service 캐싱, 정책 엔진 플러그인, Admin API 전용 gateway.

---

### 4-6. 감사 로그 계층 (Audit Log)

**핵심 역할**: 보안/컴플라이언스 목적으로 "누가, 언제, 어떤 자격으로, 무엇을, 어떤 결과로" 했는지를 append-only로 기록한다.

**다른 축과의 관계**: Authorization Service의 decision_metadata를 AuditLog의 Decision 섹션으로 연결한다. Correlation ID로 Activity Trace와 연결.

**MVP 범위**: 핵심 이벤트(인증, 문서 CRUD, 권한 변경, 감사 로그 접근) 필수. 민감 문서 read 감사는 확장.

**확장 포인트**: 무결성 검증(hash chain), 장기 보존 스토리지 분리, 감사 로그 내보내기.

---

### 4-7. Activity Trace 계층

**핵심 역할**: 사용자/AI 에이전트의 작업 흐름을 탐색 → 조회 → 편집 → 저장 연속으로 추적한다.

**다른 축과의 관계**: Correlation ID(=request_id)로 Audit Log와 연결. Trace ID로 작업 흐름 묶음 구성.

**MVP 범위**: ActivityEvent 기본 구조, Trace ID/Session ID, 핵심 이벤트(document.opened, edit.committed, search.executed 등).

**확장 포인트**: Span 모델(복잡 흐름), 집계 및 분석 파이프라인, 프론트엔드 이벤트 수집.

---

## 5. 권한 구조 통합 정리

### 5-1. Principal 목록

| Principal | 설명 | MVP | ACL 주체 가능 |
|---|---|:---:|:---:|
| User | 인증된 인간 사용자 | 필수 | 전역 리소스 한정 |
| Membership | 특정 조직 내 User의 소속 자격 | 필수 | **주요 ACL 주체** |
| Team | 조직 내 사용자 그룹 | 확장 | 확장 시 |
| ServiceAccount | 비인간 API 클라이언트 | 확장 | 확장 시 |

### 5-2. 보호 대상 리소스 목록 (MVP)

| 리소스 | Scope | 통제 방식 | 독립 ACL 허용 |
|---|---|---|:---:|
| Document | Document | RBAC + ACL | **예** |
| DocumentVersion | Document 하위 | 상속 + Draft 추가 규칙 | 아니오 |
| DocumentNode | Document 하위 | 상속 | 아니오 (확장) |
| Attachment | Document 하위 | 상속 | 아니오 |
| Comment | Document 하위 | 상속 + 소유자 예외 | 아니오 |
| Space | Space | RBAC + ACL | **예** |
| Organization | Organization | RBAC | 아니오 |
| AuditLog | Platform | 전역 역할 전용 | 아니오 |
| Permission Policy | Platform | 전역/조직 관리자 전용 | 아니오 |
| API Credential | Platform | 전역 관리자 전용 | 아니오 |

### 5-3. Permission 부여 방식 요약

**경로 1: RBAC (기본 경로)**
```
Membership → MembershipRole → Role → Permission 집합
```

**경로 2: ACL 직접 부여 (예외 경로)**
```
ACLEntry { principal_type, principal_id, resource_type, resource_id, permission, effect: allow }
```

**경로 3: 전역 역할 (플랫폼 관리 목적)**
```
User → GlobalRoleAssignment → Role (scope_type: global)
```

### 5-4. Role 목록 (MVP)

**전역 역할 (User에 직접 부여)**

| 역할 | 권한 성격 |
|---|---|
| PlatformOwner | 플랫폼 최고 권한. break-glass 보유 |
| PlatformAdmin | 플랫폼 운영 전반. 테넌트 콘텐츠 자동 접근 없음 |
| SecurityAuditor | 감사 로그/메타데이터 읽기 전용. 콘텐츠 접근 없음 |
| SupportOperator | 제한적 지원 조회. 수정 불가 |

**조직 역할 (Membership에 부여)**

| 역할 | 권한 성격 |
|---|---|
| OrganizationOwner | 조직 내 최고 권한. 멤버/역할/설정 전체 관리 |
| OrganizationAdmin | 조직 운영 관리. 멤버 초대/제거, 역할 변경 |
| SpaceManager | 특정 공간 내 문서 구조/멤버/설정 관리 |
| Editor | 문서 작성/수정/버전 생성 |
| Commenter | 코멘트 작성. 문서 본문 수정 불가 |
| Viewer | 게시된 문서 읽기 전용 |

### 5-5. Permission Naming Convention

- 형식: `{resource_type}:{action}` (소문자, snake_case)
- 예: `document:read`, `document:update`, `space:manage`, `org:manage_members`, `audit_log:read`
- 복합 관리 권한: `manage` 또는 `manage_{sub}` 형태

### 5-6. 상태 기반 권한 변화 규칙

| Resource State | read | update | delete | export | publish | archive | 복원 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| draft | Editor+ | Editor+ | Editor+ | Editor+ | Editor+ | Editor+ | — |
| published / active | Viewer+ | Editor+ | OrgAdmin+ | Editor+ | — | OrgAdmin+ | Editor+ |
| locked | Viewer+ | **차단** | **차단** | 역할 따름 | **차단** | OrgAdmin+ | OrgAdmin+ |
| archived | Viewer+ | **차단** | OrgAdmin+ | Viewer+ | OrgAdmin+ (재게시) | — | OrgAdmin+ |
| soft_deleted | OrgAdmin+ | **차단** | **차단** | **차단** | **차단** | **차단** | OrgAdmin+ |
| pending_approval | 참여자+ | **차단** | **차단** | Editor+ | — | **차단** | — |
| read_only | Viewer+ | **차단** | **차단** | Viewer+ | **차단** | OrgAdmin+ | **차단** |
| external_shared | 공유 수신자 | **차단** | **차단** | 조건부 | **차단** | **차단** | **차단** |

> **원칙**: 상태 제한은 역할/ACL allow 위에서 작동한다. 역할이 있어도 상태가 금지하면 차단된다.
> `Editor+` = Editor 이상. `OrgAdmin+` = OrganizationAdmin 이상. `역할 따름` = lock 외 별도 export 제한 없음. `조건부` = 공유 설정에 따라 결정 (Phase 4에서 확정).

### 5-7. 관리자 권한 경계 원칙

1. **PlatformAdmin은 테넌트 콘텐츠에 자동 접근하지 않는다.** 조직 내 문서에 접근하려면 별도 Membership 또는 break-glass 절차가 필요하다.
2. **SecurityAuditor는 감사 로그와 권한 변경 이력만 읽기 전용으로 접근한다.** 문서 본문에 접근할 수 없다.
3. **OrganizationOwner는 자기 조직 범위에만 관리 권한을 가진다.** 다른 조직에 자동으로 영향을 미치지 않는다.
4. **break-glass는 PlatformOwner만 보유한다.** 반드시 감사 기록이 남는다. MVP에서는 정책 정의만 하고, 자동화된 실행 절차는 확장 단계에서 구현한다.

### 5-8. ACL 예외 허용 한계

ACL은 다음을 할 수 없다.

- 조직 경계를 넘어 다른 조직 리소스에 접근 권한 부여
- soft_deleted/locked 리소스에 대한 수정 허용
- SecurityAuditor·PlatformAdmin 전용 권한을 일반 Membership에 부여
- 상태 기반 정책(L7)을 우회

---

## 6. API Authorization Enforcement 구조

### 6-1. 인증 이후 Authorization 흐름 (9단계)

```
[API 요청 수신]
  │
  ▼ Step 1. Authentication Validity
     → 토큰 없음/만료/유효하지 않음 → 401 즉시 반환
  │
  ▼ Step 2. Principal Active Status
     → User 계정 suspended/deleted → 403 반환
     → Membership status: suspended/left → 403 반환
  │
  ▼ Step 3. Global Platform Policy
     → 시스템 점검 중, API 비활성화 → 503 또는 403
  │
  ▼ Step 4. Organization Boundary
     → 요청 리소스의 조직 ≠ 요청자 Membership 조직 → 403/404
     → (예외) PlatformOwner break-glass: 감사 기록 후 조건부 통과
  │
  ▼ Step 5. RBAC 평가
     → Membership → Role → Permission 집합 확인
     → 포함되면 → allow 후보, Step 7로
     → 미포함 → Step 6으로
  │
  ▼ Step 6. Resource-specific ACL 평가
     → 직접 ACL 확인 → 있으면 → allow 후보, Step 7로
     → 상속 ACL 확인 → 있으면 → allow 후보, Step 7로
     → 아무것도 없으면 → default deny → 403/404 + 감사
  │
  ▼ Step 7. Resource State Restriction
     → locked / archived / soft_deleted / pending_approval 상태 확인
     → 상태 제한에 걸리면 → 403 + 감사 (resource_state_restricted)
  │
  ▼ Step 8. Sensitive Operation Guard
     → 고위험 액션 해당 시 → 사유 기록 의무화 + 강화 감사 트리거
  │
  ▼ Step 9. Final Decision
     → allow: 실행 허용 + AuditEvent + AuthorizationDecisionLog
     → deny: 403/404 + AuthorizationDecisionLog (Step 2~7에서 이미 기록됨)
```

### 6-2. 계층별 책임 분리

| 계층 | 책임 | 하지 않는 것 |
|---|---|---|
| Authentication Middleware | 토큰 검증, Principal 식별, request context 주입 | 권한 판단 |
| Router / Controller | 경로 파싱, 리소스 ID 수집, 입력 유효성 검사 | 직접 권한 판단 |
| Service Layer | Authorization Service 호출, 비즈니스 로직, 감사 트리거 | 권한 평가 로직 직접 구현 |
| Authorization Service | RBAC + ACL + 정책 평가, allow/deny + decision_metadata 반환 | 비즈니스 로직, DB 직접 접근 |
| Repository | 데이터 조회/저장, authorization-aware query 지원 | 권한 판단, 비즈니스 규칙 |

### 6-3. Pre-check / Post-load 원칙

| 요청 유형 | 검사 방식 | 이유 |
|---|---|---|
| Create | Pre-check (상위 컨테이너 권한) | 생성 대상이 아직 없으므로. 상위 Space/Org 권한 확인 |
| Read single | Post-load | 리소스 상태, 직접 ACL 반영 필요 |
| Update / Delete | Post-load | 상태(locked 등) 반영 필수 |
| List / Search | Pre-check + authorization-aware query | 컨테이너 접근 확인 후 DB 필터링 |
| Nested Resource | 상위 post-load → 하위 추가 규칙 | 상위 거부 시 하위 로드 없이 즉시 거부 |

### 6-4. List/Search 처리 원칙

- 권한 없는 항목은 **목록에서 완전 제외**한다 (존재 사실도 노출하지 않음)
- `total count`는 접근 가능한 항목 수만 반영한다
- 필터링은 **DB 레벨(authorization-aware query)**에서 수행한다. 서비스 레이어 후처리는 대용량에서 성능 문제를 유발함

### 6-5. 오류 응답 원칙

| 상황 | 응답 | 비고 |
|---|---|---|
| 미인증 | 401 | 토큰 없음/만료 |
| 일반 콘텐츠 — 권한 없음 | 403 | 존재는 알아도 됨 |
| 보안 민감 리소스 — 권한 없음 | 404 | 존재 은닉. AuditLog, Permission Policy, API Credential 등 |
| 리소스 미존재 | 404 | 실제 없음 |
| Admin API — 권한 없음 | 404 | 경로 자체를 숨김 |

**내부 감사에는 실제 거부 이유 전량 기록한다.** 외부 응답에는 최소 정보만 포함한다.

주요 denial_reason 코드: `not_authenticated`, `principal_inactive`, `scope_mismatch`, `rbac_no_role`, `rbac_insufficient_role`, `acl_no_entry`, `resource_state_restricted`, `org_policy_restriction`, `default_deny`

### 6-6. Authorization Context 모델 (Authorization Service 입력)

```
AuthorizationContext {
  principal: {
    user_id:         string
    membership_id:   string | null
    organization_id: string | null
    global_roles:    string[]
    org_roles:       string[]
  }
  resource: {
    resource_type:   string
    resource_id:     string | null
    parent_type:     string | null
    parent_id:       string | null
    organization_id: string | null
    resource_state:  string | null   // locked, archived, draft 등
  }
  action:            string           // "document:update"
  request_source:    string           // "user_ui" | "admin_ui" | "external_api"
  request_id:        string           // correlation_id
}
```

Authorization Service 출력:

```
AuthorizationResult {
  allowed:              boolean
  reason:               string | null      // denial_reason code
  matched_role:         string | null
  matched_acl_entry_id: string | null
  authorization_path:   string | null      // "rbac" | "acl_direct" | "acl_inherited" | "default_deny"
  decision_metadata:    object             // 감사 로그 연결용 상세 정보
}
```

---

## 7. Audit Log / Activity Trace 통합 정리

### 7-1. 두 시스템 비교

| 비교 축 | Audit Log | Activity Trace |
|---|---|---|
| **목적** | 보안/컴플라이언스 증적. 법적 책임 추적 | 사용 흐름/패턴 분석. UX 개선, AI 이해 |
| **주요 사용자** | SecurityAuditor, PlatformAdmin, 외부 감사인 | 운영팀, 분석팀, OrganizationAdmin |
| **기록 대상** | 명시적 비즈니스 행위(생성/삭제/권한 변경 등) | 탐색 흐름, 조회 시퀀스, 작업 맥락 |
| **저장 단위** | 단건 이벤트 레코드 | Event, Trace(흐름 묶음), Session |
| **보존 성격** | Append-only. 수정/삭제 불가. 수년 보존 | 분석 목적. 3~6개월 보존 |
| **민감도** | 높음 (접근 자체가 감사 대상) | 중간 (익명화/집계 처리 가능) |
| **read 포함** | 민감 문서 read만 조건부 포함 | 모든 의미 있는 조회/탐색 포함 |
| **correlation 방식** | request_id (= correlation_id) 공유 | trace_id로 흐름 묶음, correlation_id로 Audit Log 연결 |
| **무결성** | 필수 (append-only) | 권장 |

### 7-2. 이벤트 유형 분류

**Audit Log 전용 이벤트** (Activity Trace에 기록하지 않음)

- `auth.login.attempted` / `auth.token.issued` / `auth.token.revoked`
- `auth.access.denied` (권한 거부)
- `permission.acl.created/updated/deleted`
- `permission.policy.updated`
- `permission.global_role.granted/revoked`
- `org.membership.role.granted/revoked`
- `admin.api_credential.created/revoked`
- `audit.log.accessed`

**Activity Trace 전용 이벤트** (Audit Log에 기록하지 않음)

- `document.opened` / `space.browsed`
- `search.executed` / `search.result.opened`
- `edit.started` / `edit.abandoned` / `edit.autosaved`
- `document.node.expanded`
- `ai.action.started` / `ai.action.completed`

**이중 기록 이벤트** (Audit Log + Activity Trace 동시 기록, correlation_id 공유)

| Activity Trace 이벤트 | Audit Log 이벤트 |
|---|---|
| `edit.committed` | `document.updated` |
| `comment.submitted` | `document.comment.created` |
| `attachment.uploaded` | `document.attachment.uploaded` |
| `version.restored` | `document.version.restored` |
| `export.initiated` / `export.completed` | `document.exported` |

### 7-3. 두 시스템 간 공유 필드

| 필드 | 공유 방식 |
|---|---|
| `correlation_id` (= `request_id`) | 동일 요청에서 발생한 두 레코드를 연결 |
| `actor_id`, `actor_type` | Actor 식별 동일 |
| `acting_org_id` | 조직 컨텍스트 공유 |
| `resource_type`, `resource_id` | 대상 리소스 동일 |
| `occurred_at` | 발생 시각 동일 (UTC) |

### 7-4. Audit Log 레코드 핵심 구조 요약

```
AuditLog {
  id, event_type, action_category, result (success|denied|failed), occurred_at

  // Actor
  actor_type (user|service_account|system), actor_id,
  on_behalf_of,                              // 위임 접근 시 원래 사용자 ID (null이면 직접 접근)
  acting_membership_id, acting_org_id, acting_roles,
  auth_method, request_source

  // Target
  target_resource_type, target_resource_id,
  target_parent_type, target_parent_id, target_org_id

  // Decision
  allowed, denial_reason, authorization_path,
  matched_permission, matched_role, matched_acl_entry_id

  // Change Summary (콘텐츠 원문 금지. 요약만)
  before_summary, after_summary, changed_fields, policy_reason

  // Context
  request_id, ip_address, api_endpoint, api_method
}
```

**기록 금지 항목**: 문서 본문(Node content), 코멘트 내용, 첨부파일 내용, 메타데이터 확장 필드 값 전체. 요약(제목, ID, 필드명 목록)만 기록.

### 7-5. Activity Trace 레코드 핵심 구조 요약

```
ActivityEvent {
  id, event_type, event_category, occurred_at, duration_ms

  // Actor / Session
  actor_type (user|service_account|ai_agent|system), actor_id,
  acting_org_id, session_id, client_type

  // Trace 연결
  trace_id, parent_trace_id, span_id (확장), correlation_id

  // Target
  resource_type, resource_id,
  context_metadata (검색 쿼리 텍스트 금지. 결과 수/카테고리 등 요약만)
}
```

---

## 8. 정책 우선순위와 예외 규칙

### 8-1. 권장 정책 평가 순서 (9단계)

| Step | 계층 | MVP | 실패 시 응답 |
|---|---|:---:|---|
| 1 | Authentication Validity | 필수 | 401 |
| 2 | Principal Active Status | 필수 | 403 |
| 3 | Global Platform Policy | 부분 포함 | 503 / 403 |
| 4 | Organization Boundary | 필수 | 403 / 404 |
| 5 | RBAC (Role-based Permission) | 필수 | → Step 6 |
| 6 | Resource-specific ACL | 필수 | 403 (default deny) |
| 7 | Resource State Restriction | 필수 | 403 |
| 8 | Sensitive Operation Guard | 부분 포함 | 추가 조건 요구 |
| 9 | Final Decision + Audit | 필수 | — |

> **L9 / L10 처리 방식**: Task 2-7에서 분석한 10개 정책 계층(L1~L10) 중 L9(Compliance/Audit-required Rule)과 L10(External Access/Share Rule)은 MVP 9단계에 독립 Step으로 포함하지 않는다.
> - **L9**: 감사 로그 기록 보장 여부를 기반으로 실행을 차단하는 규칙. MVP에서는 Step 9 "Final Decision" 내부에 통합하여 처리한다. 감사 기록 실패 시 실행 롤백은 Phase 13에서 별도 설계한다.
> - **L10**: Share Link / 외부 공유 접근 규칙. MVP에서는 외부 공유 기능 자체가 미구현이므로 해당 없음. Phase 4 Share Link 설계 시 Step 4(조직 경계) 처리 흐름에 External Principal 분기를 추가하여 활성화한다.

### 8-2. 충돌 해소 기본 규칙 (9개)

| 규칙 | 원칙 |
|---|---|
| 규칙 1 | 인증 실패는 모든 판단을 중단시킨다 (401 즉시 반환) |
| 규칙 2 | 비활성 계정/멤버십의 역할과 ACL은 유효하지 않다 |
| 규칙 3 | 리소스 상태 제한(L7)은 역할/ACL의 allow를 무효화한다 |
| 규칙 4 | 조직 경계(L4)는 ACL 예외(L6)보다 상위 보호막이다. ACL로 테넌트 경계를 넘을 수 없다 |
| 규칙 5 | 직접 부여된 ACL은 상속된 ACL보다 우선한다 |
| 규칙 6 | 하위 리소스의 명시적 상태 제한은 상위 상속 allow를 무효화한다 |
| 규칙 7 | PlatformAdmin은 테넌트 콘텐츠에 자동 접근하지 않는다. 명시적 절차 필요 |
| 규칙 8 | 아무 경로로도 allow를 얻지 못하면 default deny (403)이다 |
| 규칙 9 | 고위험 액션은 기본 권한이 있어도 추가 가드(사유 기록, 강화 감사)가 필요하다 |

### 8-3. 상태 기반 제한 핵심 원칙

- **locked**: 역할/ACL에 관계없이 수정/삭제/게시 전면 차단. read는 허용. OrgAdmin 이상만 잠금 해제 가능
- **archived**: 수정 차단. read는 설정에 따라 허용. OrgAdmin 이상만 복원 가능
- **soft_deleted**: OrgAdmin 이상만 조회/복원 가능. 일반 사용자에게 404
- **draft**: Editor 이상만 접근. Viewer는 게시(published) 버전만 접근
- **pending_approval**: 워크플로 참여자만 특정 액션 가능. 본문 수정 차단

### 8-4. 고위험 액션 추가 가드 목록 (MVP)

다음 액션은 기본 권한 확인 외에 **사유 기록 의무화 + 강화 감사**가 적용된다.

| 액션 | 가드 수준 |
|---|---|
| 역할 부여/회수 (org:manage_members) | 사유 기록 + 강화 감사 |
| ACL 생성/수정/삭제 (permission:manage) | 사유 기록 + 강화 감사 |
| Permission Policy 변경 | 사유 기록 + 강화 감사 + 이중 감사 |
| API Credential 생성/폐기 | 강화 감사 |
| 전역 역할 변경 (PlatformAdmin 부여) | 사유 기록 + 강화 감사 |
| 감사 로그 조회 | 조회 행위 자체도 감사 기록 |
| 대량 export | 사유 기록 + 강화 감사 (확장: 수량 임계값 설정) |
| 문서 버전 복원 | 강화 감사 (복원 전 version_id 기록 필수) |

확장 단계에서 추가될 가드: 재인증(2FA 재검증), 2단계 승인, break-glass 자동 만료.

---

## 9. MVP 기준선

### 9-1. MVP 필수 구현 항목

| 영역 | 항목 | 설명 |
|---|---|---|
| 주체 | Membership 중심 역할 구조 | User → Membership → MembershipRole → Role 경로 |
| 주체 | GlobalRoleAssignment | PlatformOwner, PlatformAdmin, SecurityAuditor 전역 역할 |
| 경계 | Organization Boundary Enforcement | Step 4. 모든 API 요청에서 조직 경계 확인 |
| 권한 | RBAC 기본 권한 | Membership → Role → Permission 집합 평가 |
| 권한 | Resource-scoped ACL (allow만) | Space/Document 수준 예외 허용. deny는 미포함 |
| 권한 | Permission Naming (`{resource}:{action}`) | 일관된 권한 명칭 규칙 |
| Enforcement | 공통 Authorization Service | 권한 판단 로직 단일화 |
| 상태 | locked / archived / soft_deleted 상태 제한 | Resource State 기반 추가 제한 |
| 감사 | 핵심 Audit Log 이벤트 | 인증, 문서 CRUD, 권한 변경, 감사 로그 접근 |
| 감사 | 고위험 액션 강화 감사 | 역할 변경, ACL 변경, 정책 변경 등 |
| 추적 | 핵심 Activity Trace 이벤트 | document.opened, edit.committed, search.executed 등 |
| 추적 | Trace ID / Correlation ID 연결 | Audit Log ↔ Activity Trace 연결 |
| 응답 | 존재 은닉(Existence Hiding) | 보안 민감 리소스 → 404 |

### 9-2. 후속 확장 항목

| 영역 | 항목 | 도입 시점 |
|---|---|---|
| 주체 | Team / Group | Phase 7+ (관리자 UI) |
| 주체 | Service Account | Phase 3 (API 계층 설계) |
| 권한 | deny 기반 ACL | Phase 5+ |
| 권한 | 조건부 ACL (condition) | Phase 5+ |
| 권한 | 만료 ACL (expires_at) | Phase 4 (Share Link) |
| 권한 | sensitivity label 기반 제어 | Phase 13 (보안 거버넌스) |
| 정책 | time/rate-based policy | Phase 13 |
| 정책 | Global Policy Engine | Phase 13 |
| 감사 | break-glass 자동 승인/만료 | Phase 13 |
| 감사 | Audit Log hash chain (무결성 검증) | Phase 13 |
| 감사 | 민감 문서 read 자동 감사 | Phase 13 |
| 추적 | Span 모델 (복잡 흐름) | AI 도구 통합 Phase |
| 추적 | 프론트엔드 이벤트 수집 | Phase 8 (검색/UI) |
| 추적 | 집계/분석 파이프라인 | Phase 8+ |
| 워크플로 | approval/workflow gate | Phase 5 |
| 공유 | Share Link / External Grant | Phase 4 |

### 9-3. MVP 단순화 원칙

1. **Allow-only ACL**: deny 평가 로직 없이 allow 엔트리만 관리. 예외 차단이 필요하면 역할 재설계 또는 Space 구조 변경으로 해결
2. **Space/Document 수준 ACL만**: Node, Comment, Attachment는 상속만. 직접 ACL 부여 불가
3. **역할 계층은 참조용**: 상위 역할이 하위 역할 권한을 자동으로 포함하지 않음. Permission 집합으로 명시적 정의
4. **조건부 권한 없음**: MVP에서 condition, time-bound 등 복잡한 ACL 조건은 도입하지 않음
5. **Break-glass 정책 정의만**: 자동화된 break-glass 실행 절차는 MVP 이후

---

## 10. 구현 전 체크리스트

### 10-1. 구현 착수 전 체크리스트

이 항목들이 확인되지 않으면 구현을 시작하지 않는다.

- [ ] **용어 정의 일치**: Glossary의 용어가 코드 변수명, API 필드명, DB 컬럼명, 문서 전반에서 동일하게 사용되는가
- [ ] **Principal 식별 방식 확정**: API 요청에서 user_id → membership_id → organization_id를 어떻게 추출하고 주입할 것인가
- [ ] **AuthorizationContext 입력 모델 확정**: Authorization Service에 전달할 정확한 필드와 타입이 결정되었는가
- [ ] **Permission 목록 확정**: MVP에서 사용할 `{resource}:{action}` Permission 전체 목록이 코드화되었는가
- [ ] **Role-Permission 매핑 확정**: 각 Role이 어떤 Permission 집합을 포함하는지 명시적으로 정의되었는가
- [ ] **Resource Scope 계층과 API 경로 정합성**: `Organization → Space → Document → Version` 계층이 API 경로 설계에 반영되었는가
- [ ] **ACL 부여 가능 리소스 확정**: Space와 Document만 직접 ACL 부여 허용. 이 범위가 API 설계에 반영되었는가
- [ ] **Audit Event taxonomy 확정**: MVP 감사 이벤트 명칭(`{domain}.{object}.{action}`) 목록이 코드에 상수로 정의되었는가
- [ ] **Activity Trace 수집 범위 결정**: 어떤 이벤트를 Activity Trace로 수집할지, 고빈도 이벤트(autosave 등) 억제 정책이 있는가
- [ ] **관리자 권한 과도함 여부 검토**: PlatformAdmin이 테넌트 콘텐츠에 자동 접근하지 않는 구조가 설계에 반영되었는가
- [ ] **오류 응답 정책 결정**: 어떤 리소스에 403을 반환하고 어떤 리소스에 404를 반환하는지 목록화되었는가
- [ ] **상태 기반 제한 규칙 연결**: Resource State 변화가 Document 도메인 모델(Phase 1)과 어떻게 연결되는가

### 10-2. 구현 리뷰 시 체크리스트

구현 중 흔들리기 쉬운 부분을 코드 리뷰 시 확인한다.

- [ ] **권한 로직 중앙화**: 권한 판단이 Authorization Service 외의 곳(Service Layer 직접 구현, Router, Repository)에서 이루어지고 있지 않은가
- [ ] **pre-check / post-load 원칙 준수**: Create는 pre-check, Read/Update/Delete는 post-load 패턴을 따르는가
- [ ] **List 쿼리에서 후처리 필터링 없음**: 전체 목록 로드 후 서비스 레이어에서 필터링하고 있지 않은가
- [ ] **감사 로그에 콘텐츠 원문 없음**: `before_summary`, `after_summary`에 문서 본문/코멘트 내용이 포함되지 않는가
- [ ] **result 코드 3분류**: Audit Log의 result 필드가 `success` / `denied` / `failed` 세 가지로만 사용되는가
- [ ] **Activity Trace에 검색 쿼리 텍스트 없음**: `context_metadata`에 원문 검색 쿼리가 저장되지 않는가
- [ ] **이중 기록 이벤트 correlation_id 공유**: `edit.committed` → `document.updated` 같은 이중 기록 이벤트가 동일 `correlation_id`를 공유하는가
- [ ] **Membership status 확인**: 역할/ACL 평가 전에 Membership status(active 여부) 확인이 선행되는가
- [ ] **존재 은닉 일관성**: 보안 민감 리소스(AuditLog, Permission Policy 등)에 대한 권한 없는 접근이 일관되게 404를 반환하는가
- [ ] **상태 제한이 allow 이후 적용**: Resource State Restriction이 RBAC/ACL 평가 이후(Step 7)에 적용되는가, 아니면 혼재하는가
- [ ] **감사 이벤트 누락 없음**: 고위험 액션(역할 변경, ACL 변경, 정책 변경)에 대한 감사 기록이 항상 생성되는가

---

## 11. 설계 리스크 및 주의점

### 리스크 1. RBAC와 ACL 경계가 흐려지는 위험

| 항목 | 내용 |
|---|---|
| **왜 생기는가** | 구현 편의를 위해 "특정 사용자에게만 주면 되니까 ACL로 하자"는 판단이 반복되면 RBAC 역할 체계가 의미를 잃음 |
| **어떤 문제** | ACL 엔트리가 폭증. 권한 파악이 불가. 감사 시 "왜 이 사람이 접근 가능한가" 설명 불가 |
| **예방 설계** | ACL은 Space/Document 수준에서만. 조직 범위 접근은 반드시 Role 부여로. ACL reason 필드 기재 의무화 |
| **구현 주의점** | ACL 엔트리 수를 모니터링. 동일 리소스에 10개 이상 ACL 엔트리가 생기면 역할 구조 재검토 신호 |

---

### 리스크 2. 감사 로그와 Activity Trace가 섞이는 위험

| 항목 | 내용 |
|---|---|
| **왜 생기는가** | "어차피 둘 다 이벤트니까 같은 테이블에 넣자" 또는 "탐색도 감사해야 하지 않을까?"라는 판단 |
| **어떤 문제** | 감사 로그가 수십 배 불어남. 보존 정책 충돌. 감사 담당자가 불필요한 데이터에 접근 |
| **예방 설계** | 두 시스템 물리 분리. 이벤트 발생 시 명시적으로 "이건 AuditEvent다 / ActivityEvent다" 결정 |
| **구현 주의점** | 이중 기록 이벤트는 correlation_id로 연결하되 레코드 구조는 각 시스템 스키마를 따름. 하나의 레코드로 통합하지 않음 |

---

### 리스크 3. 관리자 권한이 과도해지는 위험

| 항목 | 내용 |
|---|---|
| **왜 생기는가** | "운영 편의상 PlatformAdmin이 다 보면 안 되나?" 또는 "긴급 상황에 빠른 처리를 위해" |
| **어떤 문제** | 테넌트 간 데이터 분리 원칙 붕괴. 내부자 위협 경로 생성. 컴플라이언스 위반 |
| **예방 설계** | PlatformAdmin은 메타데이터/운영 권한만. 콘텐츠는 break-glass 절차 필수. SecurityAuditor는 읽기 전용 |
| **구현 주의점** | PlatformAdmin 계정으로 일반 문서 API를 호출했을 때 콘텐츠가 반환되면 안 됨. 단위 테스트로 검증 필요 |

---

### 리스크 4. 상태 기반 정책이 나중에 누락되는 위험

| 항목 | 내용 |
|---|---|
| **왜 생기는가** | 새 API 엔드포인트 추가 시 "권한 체크는 했는데 상태 체크를 빼먹음". RBAC 통과만 확인하는 습관 |
| **어떤 문제** | locked 문서가 수정됨. archived 문서가 재편집됨. 의도하지 않은 상태 전이 |
| **예방 설계** | Authorization Service가 resource_state를 입력으로 받아 Step 7에서 상태 제한을 통합 평가. 개별 엔드포인트에서 상태 체크를 중복 구현하지 않음 |
| **구현 주의점** | Post-load check에서 반드시 resource_state를 AuthorizationContext에 포함. Service Layer가 Document 로드 후 state를 빠뜨리지 않도록 |

---

### 리스크 5. List/Search에서 정보 노출 위험

| 항목 | 내용 |
|---|---|
| **왜 생기는가** | "검색에서 제목만 보여주면 되지 않나?" 또는 total count에 전체 수가 노출됨 |
| **어떤 문제** | 기밀 문서 제목 노출. total count로 문서 존재 간접 추론 |
| **예방 설계** | List/Search 응답은 authorization-aware query로 접근 가능한 항목만 반환. total count도 접근 가능 항목 수만 |
| **구현 주의점** | 서비스 레이어에서 전체 로드 후 필터링하는 패턴이 없어야 함. DB 쿼리 수준에서 membership_id, accessible_space_ids로 조건 적용 |

---

### 리스크 6. 후속 Workflow 정책이 기존 모델과 충돌하는 위험

| 항목 | 내용 |
|---|---|
| **왜 생기는가** | Phase 5에서 승인 워크플로를 추가할 때 "pending_approval 상태에서 누가 무엇을 할 수 있는가"가 기존 RBAC/ACL과 충돌 |
| **어떤 문제** | Workflow 참여자 역할이 기존 Editor 역할과 분리되지 않아 승인 없이 게시 가능 |
| **예방 설계** | Resource State에 pending_approval을 이미 포함. 상태 기반 제한 원칙(규칙 3)이 워크플로 단계에도 적용되도록 일관성 유지 |
| **구현 주의점** | Phase 5 워크플로 설계 시 기존 Resource State 제한 테이블을 먼저 확인하고 확장 |

---

## 12. 후속 Phase 연결 포인트

### 12-1. Approval / Workflow Phase (Phase 5)

| 항목 | 내용 |
|---|---|
| **Phase 2 기반 결정** | pending_approval 상태 제한 정의됨. Document 상태 전이가 권한 평가 일부임 |
| **열어 둔 확장 포인트** | Workflow 참여 역할(검토자, 승인자)을 기존 Role에 추가하거나 별도 WorkflowRole로 분리 |
| **다시 결정할 것** | 승인 권한(`document:approve`)의 부여 대상. 워크플로 참여자 역할 모델. 승인 거절 시 상태 전이 규칙 |

### 12-2. External Share / Public Access Phase (Phase 4)

| 항목 | 내용 |
|---|---|
| **Phase 2 기반 결정** | ACLEntry의 `expires_at`, `source: share_link` 필드가 설계에 이미 존재. ExternalSharePrincipal 개념 예비 |
| **열어 둔 확장 포인트** | Share Link를 principal로 표현하는 ACL 엔트리 생성 로직 |
| **다시 결정할 것** | 외부 공유 허용 조직 정책 설정 방식. 외부 접근자의 Activity Trace 기록 여부. Share Link 감사 이벤트 세부 설계 |

### 12-3. Admin UI Phase (Phase 7)

| 항목 | 내용 |
|---|---|
| **Phase 2 기반 결정** | 역할 목록, Permission 명칭, ACL 부여 인터페이스 설계가 완료됨 |
| **열어 둔 확장 포인트** | Admin UI에서 ACL 시각화. 역할별 Permission 매트릭스 표시 |
| **다시 결정할 것** | Admin UI 자체 접근 권한 분리 (Admin UI Path 존재 은닉 정책). Team/Group 관리 UI |

### 12-4. User Collaboration Phase (Phase 6)

| 항목 | 내용 |
|---|---|
| **Phase 2 기반 결정** | Comment, Attachment 권한 상속 원칙 정의됨. Activity Trace의 협업 계열 이벤트 설계됨 |
| **열어 둔 확장 포인트** | 코멘트 권한 세분화(Commenter 역할 독립 부여). Annotation 독립 주석 모델 |
| **다시 결정할 것** | 실시간 협업 편집(Concurrent Edit) 시 Version 생성 정책. 코멘트 삭제 권한 세부 규칙 |

### 12-5. Integration / Service Account Phase (Phase 3)

| 항목 | 내용 |
|---|---|
| **Phase 2 기반 결정** | Principal 추상화가 ServiceAccount를 수용할 구조로 설계됨. actor_type 필드에 service_account 포함 |
| **열어 둔 확장 포인트** | ServiceAccount 엔티티 모델. API Key 발급/폐기 절차 |
| **다시 결정할 것** | ServiceAccount의 조직 귀속 방식. Service Account가 보유할 수 있는 역할 범위. API Key와 JWT 토큰 관계 |

### 12-6. Policy Engine / Governance Phase (Phase 13)

| 항목 | 내용 |
|---|---|
| **Phase 2 기반 결정** | 정책 평가 순서 9단계 정의. L3.5(sensitivity), L7.5(time/rate), L8.5(2FA) 확장 포인트 예비 |
| **열어 둔 확장 포인트** | 외부 정책 엔진(OPA 등) 통합 hook. 동적 정책 규칙 저장소 |
| **다시 결정할 것** | 정책 우선순위 엔진 구현 방식. sensitivity label 체계. break-glass 자동화. 감사 로그 장기 보존 인프라 |

### 12-7. Knowledge / AI Tool Integration Phase (AI Phase)

| 항목 | 내용 |
|---|---|
| **Phase 2 기반 결정** | actor_type에 ai_agent 포함. Activity Trace의 ai.action.started/completed 이벤트 설계됨. Trace/Span 계층이 AI 작업 흐름 기록 지원 |
| **열어 둔 확장 포인트** | AI 에이전트의 ServiceAccount 인증. AI가 생성한 콘텐츠의 Version 귀착 방식 |
| **다시 결정할 것** | AI 에이전트 권한 범위(어떤 리소스까지 접근 가능한가). AI 행위 감사 기록 세부 설계. RAG 파이프라인 결과물의 리소스 귀속 |

---

## 12-A. 미해결 오픈 이슈 집계

Task 2-1~2-7에서 제기된 오픈 이슈 중 Phase 2 내에서 해결되지 않고 다음 단계로 넘어가는 항목을 정리한다.

| ID | 출처 | 이슈 | 결정 필요 시점 |
|---|---|---|---|
| OI-T1-1 | Task 2-1 | 전역 역할 보유자(PlatformAdmin)의 조직 리소스 접근 범위 — **해결**: Task 2-7에서 "기본 차단, break-glass 절차 필요"로 확정 | ✅ 완료 |
| OI-T1-2 | Task 2-1 | Role이 Permission 집합을 직접 포함하는가 vs 코드에서 해석하는가 — **미결**: MVP 구현 방식 결정 필요 | Phase 3 구현 착수 전 |
| OI-T1-3 | Task 2-1 | Membership 상태(invited, suspended)와 역할 유효성 연동 — **해결**: Task 2-7 Step 2에서 active 상태만 유효로 확정 | ✅ 완료 |
| OI-T1-4 | Task 2-1 | 조직 삭제 시 Membership 및 역할 처리 방식 — **미결**: soft_delete 연계 정책 미정의 | Phase 3 Organization 관리 구현 전 |
| OI-T1-5 | Task 2-1 | SpaceManager 역할의 Space 엔티티 연결 — **해결**: Task 2-2에서 Space Scope 계층 정의 완료 | ✅ 완료 |
| OI-T4-1 | Task 2-4 | Authorization Service 캐싱 전략 — **미결**: 성능 최적화 방안 미정 | Phase 3 구현 중 성능 테스트 후 결정 |
| OI-T5-1 | Task 2-5 | 민감 문서 read 자동 감사 — **미결**: sensitivity label 체계가 없어 조건 적용 불가 | Phase 13 (sensitivity labeling 설계 시) |
| OI-T6-1 | Task 2-6 | Span 모델 도입 시점 — **미결**: AI 에이전트 멀티스텝 작업 추적 필요 시점에 결정 | AI Tool Integration Phase |
| OI-T7-1 | Task 2-7 | Lock override 긴급 절차 — **미결**: MVP에서는 없음. Break-glass 준용 절차 상세화 필요 | Phase 13 (보안 거버넌스) |
| OI-T7-2 | Task 2-7 | 영구 삭제(Hard delete) 정책 — **미결**: MVP에서는 soft delete만. 영구 삭제 절차 미정 | Phase 13 또는 법적 요구사항 발생 시 |
| OI-NEW-1 | Phase 2 전반 | 이중 기록 이벤트 생성 순서 및 실패 처리 — **미결**: edit.committed(Activity) + document.updated(Audit) 동시 발생 시 트랜잭션 경계 미정의 | Phase 3 구현 착수 전 |
| OI-NEW-2 | Phase 2 전반 | Workspace/Space/Folder 용어 — **결정**: 이 문서에서 `Space`로 통일. Workspace와 Folder는 UI 표현에 불과하며 도메인 모델에서는 Space 단일 개념 사용 | ✅ 완료 |

> **구현 착수 전 반드시 해결해야 할 미결 이슈**: OI-T1-2, OI-T1-4, OI-NEW-1

---

## 13. 최종 요약

### Phase 2 핵심 결론 (10개)

1. **권한 주체의 중심은 Membership이다.** 조직 내 역할은 User가 아닌 Membership에 붙는다. 이를 통해 다조직 지원, 감사 추적, 역할 관리 명확성이 동시에 확보된다.

2. **RBAC가 1차, ACL이 2차다.** 일상 권한은 역할 부여로 처리한다. ACL은 Space/Document 수준 예외에만 사용한다. ACL을 남발하면 권한 체계가 불투명해진다.

3. **조직 경계는 ACL보다 상위 보호막이다.** ACL로 다른 조직의 리소스에 접근 권한을 줄 수 없다.

4. **리소스 상태 제한은 역할/ACL allow를 무효화한다.** locked 문서는 Editor도 수정할 수 없다. 상태 제한은 권한 평가 최상위 덮개다.

5. **Authorization Service는 중앙화된 단일 판단 지점이다.** Service Layer는 Authorization Service를 호출할 뿐, 직접 권한 로직을 구현하지 않는다.

6. **감사 로그와 Activity Trace는 목적이 다르다.** 혼용하지 않는다. Audit Log는 보안/컴플라이언스 증거. Activity Trace는 흐름/패턴 분석.

7. **감사 로그에 콘텐츠 원문을 넣지 않는다.** 문서 본문, 코멘트 내용은 금지. 요약(제목, ID, 변경 필드 목록)만 허용.

8. **기본은 deny다.** 어떤 경로로도 allow를 얻지 못하면 403. 명시적 허용이 없으면 접근 불가.

9. **관리자 권한에는 경계가 있다.** PlatformAdmin은 테넌트 콘텐츠에 자동 접근하지 않는다. SecurityAuditor는 감사 데이터만 읽는다.

10. **MVP는 예측 가능성과 단순성이 우선이다.** deny ACL, 조건부 정책, Team, ServiceAccount, break-glass 자동화는 확장으로 미룬다. 지금은 쉽게 이해하고 안전하게 구현하는 것이 목표다.

---

### 구현 시 절대 흔들리면 안 되는 원칙 (8개)

1. **권한 판단은 Authorization Service 한 곳에서만 한다.** Router, Service Layer 내 직접 권한 로직 금지.
2. **List/Search는 authorization-aware query로 DB에서 필터링한다.** 서비스 레이어 후처리 금지.
3. **조직 경계(Organization Boundary)는 모든 API 요청에서 항상 확인한다.** 예외 없음.
4. **리소스 상태(locked/archived/soft_deleted)를 Authorization Context에 항상 포함한다.** 빠뜨리면 상태 제한이 동작하지 않는다.
5. **감사 로그는 append-only다.** 수정/삭제 API를 만들지 않는다.
6. **콘텐츠 원문을 감사 로그에 넣지 않는다.** 요약만.
7. **보안 민감 리소스(AuditLog, Permission Policy)의 권한 없는 접근은 404다.** 403이 아니다.
8. **Audit Log와 Activity Trace는 물리적으로 분리한다.** 같은 테이블에 섞지 않는다.

---

### 다음 Phase에 넘기는 핵심 결정사항 (8개)

1. **Service Account 도입 방식**: 어떤 조직에 귀속되는가. API Key와 인증 토큰 관계는 무엇인가 (Phase 3)
2. **Share Link / External Grant 상세 설계**: 외부 주체를 ACL principal로 어떻게 표현할 것인가 (Phase 4)
3. **Workflow / Approval Role 설계**: 기존 Editor/Viewer 역할과 어떻게 분리하고 연결할 것인가 (Phase 5)
4. **Team / Group 역할 부여 방식**: Team ACL principal 도입 시 기존 Membership 경로와 어떻게 병합 평가할 것인가 (Phase 7)
5. **Deny ACL 도입 규칙**: 어떤 상황에서만 허용할 것인가. 우선순위 충돌 해소 방식은 무엇인가 (Phase 5+)
6. **Break-glass 자동화 절차**: 승인 workflow, 만료 정책, 이중 감사 구조 (Phase 13)
7. **Sensitivity Label 기반 제어**: 민감 문서 분류 체계와 L3.5 정책 연결 방식 (Phase 13)
8. **Audit Log 장기 보존 인프라**: 수년 보존을 위한 스토리지 분리, 내보내기, 무결성 검증 방식 (Phase 13)

---

*작성일: 2026-04-01*
*Phase: 2 / Task: 2-8*
*상태: Phase 2 통합 기준선 문서 — 구현 착수 가능*
*참조: Task 2-1 ~ 2-7 개별 설계 문서*
