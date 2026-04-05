# Phase 2 - Task 2-3. ACL 기반 권한 모델 설계

---

## 1. 목표

Task 2-1(권한 주체)과 Task 2-2(보호 대상 리소스)의 결과를 연결하여, **누가(principal) 어떤 리소스(resource)에 대해 어떤 행위(permission)를 어떤 효과(effect)로 수행할 수 있는지**를 표현하는 ACL 기반 권한 모델의 기준선을 확정한다.

- ACL 엔트리 개념 구조를 정의한다
- Permission taxonomy 초안을 작성한다
- RBAC와 ACL의 결합 방식을 결정한다
- 권한 평가 흐름의 논리 구조를 정리한다
- Task 2-4 API enforcement 설계의 직접 입력 자료를 준비한다

---

## 2. 설계 전제

| # | 전제 |
|---|---|
| 1 | Task 2-1 결과: 주체는 User, Organization, Membership, Role. 조직 역할은 Membership 중심 |
| 2 | Task 2-2 결과: Scope 계층 = Platform → Organization → Space → Document → Version → Node/Sub-resource |
| 3 | 기본 권한 모델은 RBAC. ACL은 예외/세분화 보조 계층으로만 사용 |
| 4 | API-first. UI 메뉴가 아닌 resource + action 기준으로 권한 정의 |
| 5 | 감사 추적 필수. "왜 허용/거부되었는가"를 사후에 설명할 수 있는 구조 |
| 6 | MVP는 단순하게. deny, condition, attribute-based rule은 신중히 도입 |

---

## 3. ACL 도입 목적과 역할

### 3-1. RBAC만으로 부족한 이유

RBAC(Role-Based Access Control)는 역할에 권한 집합을 연결하고 사용자에게 역할을 부여하는 방식이다. 대부분의 일상적 접근 제어는 RBAC로 충분하다. 그러나 이 플랫폼에서는 다음과 같은 시나리오가 반드시 발생한다.

| 시나리오 | RBAC로 처리 가능 여부 | 이유 |
|---|---|---|
| 조직 내 모든 Editor가 특정 문서를 수정 가능 | 가능 | 역할 기반으로 처리됨 |
| **특정 Editor만 특정 기밀 문서 열람 허용** | 불가 | 같은 역할 내 예외 부여 불가 |
| **특정 문서를 조직 외부인에게 읽기 공유** | 불가 | 역할 체계 밖의 주체에게 부여 불가 |
| **특정 공간을 특정 팀에게만 접근 허용** | 부분 가능 | Space 단위 역할 부여가 없으면 처리 어려움 |
| AI 에이전트(Service Account)에 특정 문서만 허용 | 불가 | Service Account는 역할 외 주체 |
| 특정 문서의 Viewer에게만 특정 버전 다운로드 차단 | 불가 | 역할보다 세밀한 리소스 단위 제어 불가 |

결론: **RBAC는 "조직 내 역할 기반 일반 접근"을 담당하고, ACL은 "리소스 단위 예외 및 세분화 접근"을 담당한다.**

### 3-2. ACL의 역할 정의

- ACL은 RBAC를 **대체하지 않는다**
- ACL은 RBAC 평가 이후, **리소스 단위로 예외를 부여하거나 제한하는 보조 계층**이다
- ACL 없이는 RBAC만으로 처리하고, 특수 상황에서만 ACL 엔트리를 생성한다

### 3-3. 기본 원칙

1. **기본 권한은 Role 기반으로 부여한다.** 사용자 역할(OrganizationAdmin, Editor, Viewer 등)이 대부분의 접근을 결정한다.
2. **예외 권한은 ACL로 부여한다.** 특정 리소스에 대한 예외적 허용/제한만 ACL 엔트리로 표현한다.
3. **ACL 사용 범위를 제한한다.** ACL을 남발하면 권한 체계가 불투명해지고 감사가 어려워진다. MVP에서는 Space와 Document 수준에서만 ACL 부여를 허용한다.

---

## 4. ACL 엔트리 개념 모델

### 4-1. 핵심 구성 요소

```
ACLEntry {
  id                    // 고유 식별자
  principal_type        // 주체 유형 (user | membership | team | service_account)
  principal_id          // 주체 식별자
  resource_type         // 리소스 유형 (space | document | ...)
  resource_id           // 리소스 식별자
  permission            // 권한 행위 (read | update | delete | share | ...)
  effect                // allow (MVP. deny는 확장)
  inherited             // 이 엔트리가 상위에서 상속된 것인지, 직접 부여된 것인지
  condition             // (확장) 조건 표현식. MVP에서는 null
  granted_by            // 부여한 주체 (User ID)
  granted_at            // 부여 일시
  expires_at            // (확장) 만료 시점. null이면 영구
  reason                // (선택) 부여 사유. 감사 목적
  source                // 출처 (manual | share_link | workflow | system)
}
```

### 4-2. 필드별 설명

| 필드 | 필수 여부 | MVP 포함 | 설명 |
|---|---|---|---|
| `id` | 필수 | 예 | ACL 엔트리 고유 식별. 감사 추적 시 특정 엔트리를 참조 가능 |
| `principal_type` | 필수 | 예 | 주체 유형 구분. User 직접 부여인지, Membership 기반인지 판별 |
| `principal_id` | 필수 | 예 | 주체 식별자 |
| `resource_type` | 필수 | 예 | 리소스 유형 (space, document 등). 평가 경로 결정에 사용 |
| `resource_id` | 필수 | 예 | 특정 리소스 인스턴스 식별자 |
| `permission` | 필수 | 예 | 허용/거부할 행위 (read, update 등) |
| `effect` | 필수 | 예 (allow만) | MVP는 allow만. deny는 확장 단계 |
| `inherited` | 필수 | 예 | 상속된 ACL인지 직접 부여된 ACL인지 구분. 감사 추적용 |
| `condition` | 선택 | 아니오 | 조건부 권한 (예: 특정 시간대만 허용). ABAC 확장 시 도입 |
| `granted_by` | 필수 | 예 | 감사 필수 필드. 누가 이 ACL을 만들었는가 |
| `granted_at` | 필수 | 예 | 감사 필수 필드 |
| `expires_at` | 선택 | 아니오 | 임시 권한 부여 시 필요. 외부 공유 기능 도입 시 활용 |
| `reason` | 선택 | 예 (권장) | "왜 이 ACL이 만들어졌는가" 설명. 감사 품질 향상 |
| `source` | 필수 | 예 | manual(관리자 직접 부여), share_link(공유 링크 파생), system(시스템 자동 생성) 구분 |

### 4-3. ACL 엔트리 예시

```
// 예시 1: OrganizationAdmin이 특정 기밀 문서를 특정 Membership에 read 허용
{
  principal_type: "membership",
  principal_id:   "mbr_abc123",
  resource_type:  "document",
  resource_id:    "doc_xyz789",
  permission:     "read",
  effect:         "allow",
  inherited:      false,
  granted_by:     "user_admin001",
  reason:         "기밀 문서 한시적 열람 허용 - 감사팀 요청",
  source:         "manual"
}

// 예시 2: 특정 공간을 특정 팀에 접근 허용
{
  principal_type: "team",
  principal_id:   "team_dev01",
  resource_type:  "space",
  resource_id:    "space_proj42",
  permission:     "read",
  effect:         "allow",
  inherited:      false,
  granted_by:     "user_spacemanager",
  source:         "manual"
}
```

---

## 5. Principal 유형 정의

### 5-1. Principal 후보 검토

| Principal 유형 | ACL 직접 대상 가능 | MVP 지원 | 운영 복잡성 | 감사 추적성 | 비고 |
|---|:---:|:---:|---|---|---|
| **User** | 가능 | **예 (제한적)** | 낮음 | 높음 — User ID로 명확히 추적 | 전역 역할 부여 시. 조직 내에서는 Membership 우선 |
| **Membership** | 가능 | **예 (주요)** | 낮음 | 높음 — 어느 조직에서 어떤 자격인지 포함 | MVP 핵심 principal |
| **Team / Group** | 가능 | **예 (부분)** | 중간 | 중간 — 팀 멤버 변경 시 ACL 영향 범위 복잡 | Team 엔티티 도입 이후 활성화 |
| **Role** | 주의 필요 | **아니오** | 높음 | 낮음 — Role에 직접 ACL 부여 시 "역할을 가진 모든 사람"에게 적용되어 통제 범위 불명확 | Role은 RBAC 경로로만. ACL principal로는 사용하지 않음 |
| **Service Account** | 가능 | 아니오 (확장) | 중간 | 높음 — API 자동화 행위 추적 용이 | Phase 3 API 계층 도입 시 활성화 |
| **ExternalSharePrincipal** | 가능 | 아니오 (확장) | 중간 | 중간 — 외부 주체 식별 체계 별도 필요 | Share Link 기능 도입 시 활성화 |

### 5-2. 최종 권장안

**MVP ACL Principal: Membership (주요) + User (전역 역할 한정)**

- **Membership**: 조직 내 문서/공간 예외 접근의 주요 principal. "A 조직의 User X에게 특정 문서 read 허용"은 Membership ID로 표현
- **User (직접)**: 전역 리소스(AuditLog, API Credential 등) 접근 제어에 한해 직접 부여. 조직 범위 리소스에 User 직접 ACL은 지양
- **Team**: Team 엔티티 도입 이후 허용. 팀 단위 공간 접근에 유용
- **Role를 ACL principal로 사용하지 않는 이유**: Role 자체에 ACL을 부여하면 해당 역할을 가진 모든 사용자에게 적용되어 예외 제어의 의미가 사라진다. Role은 RBAC 경로를 통해서만 권한에 영향을 준다

---

## 6. Permission Taxonomy 초안

### 6-1. Permission 명명 규칙

- `{resource_type}:{action}` 형식을 기본으로 한다
- action은 **동사 중심** (read, create, update, delete, share, publish 등)
- CRUD 기반을 기초로 하되, 도메인 특화 액션을 추가한다
- 예: `document:read`, `document:publish`, `space:manage_members`

### 6-2. MVP Permission Set

#### 콘텐츠 리소스 권한

| Permission | 설명 | 적용 대상 |
|---|---|---|
| `document:read` | 문서(게시 버전) 열람 | Document |
| `document:read_draft` | 초안 버전 열람 | Document (Draft Version) |
| `document:create` | 새 문서 생성 | Space 내 |
| `document:update` | 문서 수정 (새 Version 생성) | Document |
| `document:delete` | 문서 삭제 (소프트) | Document |
| `document:publish` | 초안 → 게시 상태 전이 | Document |
| `document:archive` | 문서 아카이브 | Document |
| `document:share` | 문서 공유 링크 생성 또는 직접 공유 | Document |
| `version:view` | 특정 Version 열람 | DocumentVersion |
| `version:restore` | 이전 Version으로 복원 | DocumentVersion |
| `attachment:read` | 첨부파일 다운로드/열람 | Attachment |
| `attachment:create` | 첨부파일 업로드 | Attachment |
| `attachment:delete` | 첨부파일 삭제 | Attachment |
| `comment:read` | 코멘트 열람 | Comment |
| `comment:create` | 코멘트 작성 | Comment |
| `comment:delete` | 코멘트 삭제 (본인+관리자) | Comment |

#### 공간/조직 관리 권한

| Permission | 설명 | 적용 대상 |
|---|---|---|
| `space:read` | 공간 탐색 및 문서 목록 열람 | Space |
| `space:create_document` | 공간 내 문서 생성 | Space |
| `space:manage` | 공간 설정 변경, 구조 관리 | Space |
| `org:read` | 조직 정보 열람 | Organization |
| `org:manage` | 조직 설정 변경 | Organization |
| `org:manage_members` | 멤버 초대/제거/역할 변경 | Organization |

#### 운영 리소스 권한

| Permission | 설명 | 적용 대상 |
|---|---|---|
| `audit_log:read` | 감사 로그 열람 | AuditLog |
| `permission:manage` | 역할/ACL 정책 관리 | Permission Policy |
| `api_credential:manage` | API 자격 증명 발급/취소 | API Credential |

### 6-3. 확장 Permission Set (Phase 3 이후)

| Permission | 설명 | 도입 시점 |
|---|---|---|
| `document:export` | 문서 내보내기 (PDF 등) | Phase 4+ |
| `document:lock` | 문서 편집 잠금 | Phase 5+ |
| `document:approve` | 승인 워크플로 승인 | Phase 5 |
| `document:annotate` | 주석 작성 | Phase 4+ |
| `template:use` | 템플릿 사용 | Phase 3~4 |
| `template:manage` | 템플릿 생성/수정/삭제 | Phase 3~4 |
| `document_type:manage` | DocumentType 관리 | Phase 3+ |
| `integration:manage` | 외부 연동 설정 | Phase 3+ |
| `webhook:manage` | 웹훅 관리 | Phase 3+ |
| `activity_trace:read` | 행위 추적 데이터 열람 | Phase 2-6 이후 |

### 6-4. Permission 설계 원칙 답변

**Q. Permission은 action verb 중심이 좋은가?**

> 예. `{resource}:{action}` 형식이 API-first 환경에서 가장 명확하다. "이 API 엔드포인트를 호출하려면 `document:update` 권한이 필요하다"는 매핑이 자연스럽다.

**Q. CRUD 기반과 domain-specific action을 어떻게 혼합할 것인가?**

> CRUD(read, create, update, delete)를 기반으로 하되, 문서 플랫폼에 의미 있는 도메인 액션(publish, archive, restore, share, approve)을 추가한다. 단, 모든 도메인 액션을 처음부터 정의하지 않고 구현 단계에서 점진적으로 추가한다.

**Q. Permission naming rule?**

> - 형식: `{resource_type}:{action}` (소문자, snake_case)
> - 리소스 타입: document, version, node, space, org, attachment, comment, audit_log, permission, api_credential
> - 액션: read, create, update, delete, share, publish, archive, manage, manage_members 등
> - 복합 관리 권한은 `manage` 또는 `manage_{sub_resource}` 형태로 통합

---

## 7. Effect 및 충돌 규칙

### 7-1. Allow-only vs Allow+Deny 비교

| 항목 | Allow-only | Allow + Deny |
|---|---|---|
| **장점** | 단순. "허용 목록"만 관리. 구현 복잡도 낮음 | 특정 주체에 대한 명시적 차단 표현 가능 |
| **단점** | 예외 차단 불가. "역할은 Editor지만 특정 문서는 읽기 금지" 표현 불가 | 평가 로직 복잡. allow/deny 충돌 시 우선순위 결정 필요. 남용 시 권한 체계 불투명 |
| **감사 가능성** | 높음 — 단순해서 추적 쉬움 | 중간 — 충돌 평가 경로가 복잡해짐 |
| **MVP 적합성** | 높음 | 낮음 |

### 7-2. 최종 권장안

**MVP: allow 전용. deny는 확장 단계에서 도입.**

이유:
- 이 플랫폼의 MVP 권한 체계는 "허용할 사람을 명시"하는 방향으로 충분하다
- deny가 필요한 시나리오(예: 특정 Editor를 특정 문서에서 차단)는 역할 재조정 또는 Space 구조 변경으로 대부분 해결 가능하다
- deny 도입 시 "deny가 allow보다 우선한다"는 원칙을 세워야 하고, 이는 디버깅과 감사를 어렵게 만든다

**확장 시 deny 도입 원칙:**
- deny는 반드시 감사 로그 기록 필수
- deny가 상속되지 않도록 직접 부여된 리소스에만 적용
- deny는 시스템 관리자 이상만 부여 가능

### 7-3. 충돌 규칙 (확장 대비 사전 정의)

향후 deny 도입 시 적용할 충돌 평가 우선순위:

```
1. 명시적 deny (직접 부여) > 모든 allow
2. 명시적 allow (직접 부여) > 상속된 allow
3. 상속된 allow > 역할 기반 allow
4. 아무 것도 없으면 → 기본 거부 (default deny)
```

---

## 8. 상속 및 Override 원칙

### 8-1. 핵심 질문 답변

**Q. Organization level ACL이 Document까지 내려갈 수 있는가?**

> 예. Organization에 부여된 ACL은 하위 Space, Document까지 상속된다. 단, 실제 상속 평가는 명시적으로 하위 리소스를 조회할 때 수행하며, 중간에 더 구체적인 ACL이 있으면 override된다.

**Q. Document ACL이 Node / Comment / Attachment에 상속되는가?**

> 예. Document에 read가 허용된 principal은 해당 Document의 Version, Node, Comment, Attachment에 동일하게 접근 가능하다. 단, Comment write와 Attachment delete는 역할 규칙이 추가 적용된다(소유자 예외).

**Q. Version은 상속 대상인가 독립 리소스인가?**

> 상속 대상이 기본이지만, Draft Version은 시스템 내장 규칙으로 추가 제한된다. Document read 권한이 있어도, Draft Version은 `document:read_draft` 권한 또는 Editor 이상 역할이 없으면 접근이 차단된다.

**Q. 상위 allow와 하위 예외 ACL이 충돌하면 어떻게 되는가?**

> 하위 직접 ACL이 상위 상속 ACL보다 우선한다. 예: Organization 수준에서 특정 Membership에 read가 허용되어 있어도, 특정 Document에 별도 제한 ACL이 있으면 Document 수준 규칙이 우선 적용된다.

**Q. direct assignment와 inherited assignment 중 무엇이 우선하는가?**

> 직접 부여(direct)가 상속(inherited)보다 항상 우선한다.

### 8-2. 상속 및 Override 원칙 정리

#### 기본 상속 규칙

```
Organization ACL
  └─► Space ACL (상속)
        └─► Document ACL (상속)
              ├─► Version ACL (상속 + Draft 추가 규칙)
              ├─► Node ACL (상속)
              ├─► Attachment ACL (상속)
              └─► Comment ACL (상속 + 소유자 예외)
```

상속은 "명시적 ACL이 없을 때의 기본값"이다. 상위에서 허용되지 않았다면 하위에서도 기본 거부다.

#### 하위 Override 허용 여부

- Space, Document 수준에서 override 허용 (직접 ACL 부여 가능)
- Node, Comment, Attachment 수준에서는 MVP에서 직접 ACL 부여 불가 (상속만)

#### 예외를 허용할 리소스

- **Space**: 조직 내 비공개 공간 운영
- **Document**: 공간 외부 주체에게 특정 문서만 공유

#### Override 남용 방지 원칙

1. ACL 엔트리 생성은 OrganizationAdmin 이상만 가능 (Space 수준), SpaceManager 이상(Document 수준)
2. ACL 생성 시 `reason` 필드 기재 권장 (감사 품질)
3. ACL 생성/수정/삭제는 전량 AuditLog에 기록 (필수)
4. 동일 리소스에 대한 ACL 엔트리 수를 모니터링하여 권한 파편화 감지

---

## 9. RBAC와 ACL 결합 권장안

### 9-1. 방식 비교

| 비교 항목 | 방식 A (Role = permission bundle, ACL은 예외) | 방식 B (모든 권한을 ACL 엔트리로 통일) | 방식 C (Role-permission mapping + resource 예외 ACL) |
|---|---|---|---|
| **설계 단순성** | 높음 | 낮음 — 모든 일상 권한도 ACL로 관리해야 함 | 높음 |
| **운영 난이도** | 낮음 | 매우 높음 — 엔트리 수 폭발 | 낮음 |
| **감사 가능성** | 높음 | 중간 — 너무 많은 엔트리로 파악 어려움 | 높음 — 일반 권한은 RBAC로, 예외만 ACL로 명확 구분 |
| **API enforcement 용이성** | 높음 — 역할 로드 후 permission 확인 | 낮음 — 모든 요청마다 ACL DB 조회 | 높음 — 역할 기반 1차 평가 후 ACL 2차 확인 |
| **정책 엔진 연결성** | 중간 | 높음 (모든 것이 엔트리화되어 있으므로) | 높음 (RBAC 위에 ACL을 얹는 구조가 정책 엔진과 자연스럽게 연결) |

### 9-2. 최종 권장안: 방식 C (변형)

**RBAC를 1차 평가, Resource-specific ACL을 2차 평가**

```
[기본 흐름]
요청 → 1차: Role 기반 평가 (Membership → Role → Permission)
      → 2차: Resource-specific ACL 확인 (해당 리소스에 직접 ACL이 있는가)
      → 3차: 상속 ACL 확인 (상위 scope의 ACL이 적용되는가)
      → 결론: allow / deny
```

구체적 구조:

```
권한 부여 경로 1 (RBAC 경로):
  Membership → MembershipRole → Role → [Permission 집합]

권한 부여 경로 2 (ACL 경로):
  ACLEntry { principal=Membership, resource=Document#123, permission=read, effect=allow }

평가 시:
  - 경로 1에서 해당 permission이 있으면 → allow
  - 경로 1에 없고 경로 2의 직접 ACL이 있으면 → ACL 결과 적용
  - 모두 없으면 → deny (default deny 원칙)
```

이 방식의 장점:
- 일상적 권한 관리는 역할 부여만으로 처리 → 운영 단순
- 예외 케이스만 ACL 엔트리 생성 → ACL 엔트리 수를 낮게 유지
- "이 사람이 왜 접근 가능한가?" → RBAC 경로 또는 ACL 경로 중 하나로 명확히 설명 가능
- API enforcement에서 역할 체크 후 ACL 체크 순서로 구현 자연스러움

---

## 10. 권한 평가 흐름 초안

### 10-1. 평가 입력 (Evaluation Input)

```
{
  principal: {
    user_id:         "user_abc",
    memberships: [
      { membership_id: "mbr_xyz", organization_id: "org_001", roles: ["editor"] }
    ],
    global_roles: []
  },
  resource: {
    resource_type: "document",
    resource_id:   "doc_123",
    organization_id: "org_001",
    space_id:      "space_456",
    status:        "draft"       // Version 상태 등 추가 컨텍스트
  },
  action: "document:read_draft"
}
```

### 10-2. 평가 흐름 (단계별)

```
Step 1. Principal 식별
  - 요청 토큰에서 user_id 추출
  - 요청 컨텍스트(organization_id)에서 Membership 조회
  - 전역 역할 확인

Step 2. Resource 및 Scope 확인
  - resource_type, resource_id 추출
  - 해당 리소스의 organization_id, space_id 확인 (scope 계층 파악)
  - 리소스 상태 확인 (document status, version status 등)

Step 3. RBAC 1차 평가
  - Membership → MembershipRole → Role → Permission 집합 조회
  - 요청 action이 Permission 집합에 포함되는가?
  - 포함되면 → [allow 후보]. Step 4로 이동 (ACL override 확인)
  - 미포함이면 → Step 5 (ACL 직접 부여 확인)

Step 4. Resource-specific ACL 확인 (override)
  - 해당 resource에 직접 부여된 ACL 엔트리 조회
  - MVP에서는 deny 없으므로, 직접 ACL이 있으면 추가 allow 경로 확인
  - deny 도입 시: 명시적 deny가 있으면 → deny (최우선)

Step 5. ACL 직접 부여 확인 (RBAC 부족 시)
  - 해당 resource에 principal_id 또는 membership_id 기반 ACL 엔트리 검색
  - allow 엔트리가 있으면 → allow
  - 없으면 → Step 6 (상속 ACL 확인)

Step 6. 상속 ACL 확인
  - 상위 scope (space, organization) ACL 엔트리 검색
  - 상위에서 allow가 있으면 → allow
  - 없으면 → deny (default deny)

Step 7. 최종 결론 도출
  - allow: 접근 허용
  - deny: 403 Forbidden 응답
```

### 10-3. 평가 출력 (Evaluation Output)

```
{
  allowed:   true | false,
  reason:    "rbac_role_match" | "acl_direct" | "acl_inherited" | "default_deny",
  matched_role:        "editor",           // RBAC 경로 시
  matched_acl_entry:   "acl_entry_id_789", // ACL 경로 시
  principal_context:   { user_id, membership_id, organization_id },
  resource_context:    { resource_type, resource_id, scope_path },
  evaluated_at:        "2026-04-01T10:00:00Z"
}
```

### 10-4. 감사 로그에 남겨야 할 Decision Context

```
AuditDecision {
  user_id
  membership_id        // 어느 조직 자격으로 행동했는가
  resource_type
  resource_id
  action
  allowed              // true | false
  reason               // 어떤 경로로 결론이 났는가
  matched_role         // RBAC 경로인 경우
  matched_acl_entry_id // ACL 경로인 경우
  evaluated_at
}
```

---

## 11. MVP ACL 제한안

### 11-1. 허용 리소스 범위

| 리소스 | MVP ACL 허용 | 이유 |
|---|---|---|
| Space | **허용** | 비공개 공간, 팀 전용 공간 운영에 필수 |
| Document | **허용** | 특정 문서 개별 공유, 기밀 문서 접근 제한에 필수 |
| Organization | 제한적 허용 | 전역 역할 부여로 대부분 처리 가능. ACL은 예외 케이스만 |
| Version | **불허** | Document ACL + Draft 시스템 규칙으로 충분 |
| Node | **불허** | 확장 단계 |
| Attachment | **불허** | Document ACL 상속으로 충분 |
| Comment | **불허** | 소유자 예외 규칙으로 충분 |
| AuditLog | **전역 역할로만** | ACL 아닌 SecurityAuditor 역할로 통제 |
| Permission Policy | **전역 역할로만** | OrganizationOwner/Admin 역할로 통제 |

### 11-2. 허용 Principal 범위

| Principal | MVP ACL 허용 |
|---|---|
| Membership | **허용** (주요) |
| User (직접) | 전역 리소스 한정 허용 |
| Team | Team 엔티티 도입 후 허용 |
| Role | **불허** — ACL principal로 사용하지 않음 |
| Service Account | Phase 3 이후 허용 |
| External Share Principal | Phase 3~4 이후 허용 |

### 11-3. 허용 Permission 범위

MVP에서 ACL로 예외 부여 가능한 Permission:

| Permission | ACL 예외 부여 허용 |
|---|---|
| `space:read` | 허용 |
| `document:read` | 허용 |
| `document:read_draft` | 허용 (Editor 미만 주체에게 특별 열람 부여) |
| `document:update` | 허용 |
| `document:share` | 허용 |
| `document:publish` | 허용 |
| `permission:manage` | **불허** — 역할로만 제어 |
| `audit_log:read` | **불허** — 역할로만 제어 |
| `api_credential:manage` | **불허** — 역할로만 제어 |

### 11-4. ACL 생성/수정 권한

| 대상 리소스 | ACL 생성 가능 역할 |
|---|---|
| Space ACL | OrganizationAdmin 이상 |
| Document ACL | SpaceManager 또는 OrganizationAdmin 이상 |
| Organization 수준 ACL | OrganizationOwner 또는 PlatformAdmin |

---

## 12. 다음 Task로 넘길 결정사항

### Task 2-4 (API Authorization Enforcement) 에 전달

**권한 평가에 필요한 입력 요소:**
- `user_id` (인증 토큰에서 추출)
- `organization_id` (요청 경로 또는 헤더에서 추출)
- `resource_type` + `resource_id` (API 경로에서 파싱)
- `action` (API 엔드포인트-action 매핑 테이블에서 결정)

**API 계층에서 반드시 알아야 하는 매핑:**

```
API Endpoint → action 매핑 예시:
GET  /documents/{id}            → document:read
GET  /documents/{id}?draft=true → document:read_draft
PUT  /documents/{id}            → document:update
POST /documents/{id}/publish    → document:publish
DELETE /documents/{id}          → document:delete
POST /documents/{id}/share      → document:share
GET  /audit-logs                → audit_log:read
```

**공통 Authorization Service가 판단해야 할 정보:**
1. Principal 컨텍스트 구성 (user_id → memberships → roles)
2. Resource scope 계층 파악 (document → space → organization)
3. RBAC 1차 평가 (Role → Permission)
4. Resource ACL 2차 평가
5. 상속 ACL 3차 평가
6. 최종 allow/deny + decision context 반환

**감사 로그와 연결할 수 있는 Decision Metadata:**
- `user_id`, `membership_id`, `organization_id`
- `resource_type`, `resource_id`
- `action`
- `allowed`, `reason`, `matched_role`, `matched_acl_entry_id`
- `evaluated_at`

### Task 2-5 (감사 로그 설계) 에 전달

- 모든 ACL 생성/수정/삭제는 AuditLog 기록 필수
- Permission 평가 결과(특히 deny)는 AuditDecision 레코드로 기록
- `reason` 필드가 감사 품질의 핵심

---

## 13. 오픈 이슈

| # | 이슈 | 설명 | 결정 필요 시점 |
|---|---|---|---|
| OI-1 | Role과 Permission의 관계 구조 | Role이 Permission 집합을 명시적 조인 테이블로 관리할지, 코드 내 상수로 관리할지. 조인 테이블이면 동적 변경 가능하지만 복잡도 증가 | Task 2-4 또는 구현 단계 |
| OI-2 | Permission 평가 결과 캐싱 전략 | 매 요청마다 Membership → Role → Permission 조회 비용. 캐싱 적용 시 권한 변경 즉시 반영 문제 | Phase 3 API 설계 |
| OI-3 | PlatformAdmin의 조직 리소스 override 허용 여부 | PlatformAdmin이 Organization ACL 없이도 모든 조직 리소스에 접근 가능한가. 운영 편의 vs 최소 권한 원칙 충돌 | Task 2-7 정책 우선순위 |
| OI-4 | Draft Version 접근 제한의 표현 방식 | `document:read_draft` permission을 별도 정의하는 방식 vs Version.status 기반 시스템 규칙으로 처리하는 방식. 전자가 ACL과 일관성 있으나 permission 수 증가 | Task 2-4 구현 설계 시 확정 |
| OI-5 | ACL 엔트리 만료(expires_at) MVP 포함 여부 | 외부 공유 시 임시 권한 부여에 필요하지만, Share Link 기능 없는 MVP에서는 불필요 | Share Link 기능 도입 시점 |

---

*작성일: 2026-04-01*
*Phase: 2 / Task: 2-3*
*상태: 설계 초안 완료*
