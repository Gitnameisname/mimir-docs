# Phase 2 - Task 2-2. 리소스 범위와 보호 대상 식별

---

## 1. 목표

권한 체계의 주체(Task 2-1)를 정의했다면, 이번 단계는 그 권한이 **적용될 대상 리소스와 범위(scope)**를 구조적으로 식별한다.

- 플랫폼에서 권한 통제가 필요한 모든 객체를 식별한다
- 리소스의 scope 레벨을 계층적으로 정의한다
- 상속형 권한과 독립 통제가 필요한 리소스를 구분한다
- MVP에서 우선 보호해야 할 핵심 리소스를 도출한다
- Task 2-3 ACL 설계의 직접 입력 자료를 준비한다

---

## 2. 설계 전제

| # | 전제 |
|---|---|
| 1 | Phase 1에서 확정된 Document / Version / Node / DocumentType / metadata / DocumentStatus 구조를 기반으로 한다 |
| 2 | 범용 문서 플랫폼. 단순 파일 저장소가 아니라 구조화된 문서 + 부가 객체 + 운영 객체 모두 포함 |
| 3 | API-first 구조. UI 메뉴 단위가 아닌 API에서 접근 가능한 리소스 단위로 식별한다 |
| 4 | Task 2-1 결과: 주체는 User, Organization, Membership, Role. 전역 역할과 조직 역할은 분리됨 |
| 5 | 콘텐츠 리소스(문서 본문 등)와 운영 리소스(감사 로그, API 자격 증명 등)는 권한 체계를 분리한다 |
| 6 | 상속은 기본값. 예외 통제가 필요한 리소스는 명시적으로 분리한다 |
| 7 | 향후 Attachment, Comment, Workflow, Approval, Share, AI/RAG artifact 등 부가 객체가 확장됨 |

---

## 3. 보호 대상 리소스 후보 분석

### 3-1. 콘텐츠 계열 리소스

#### Document

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 플랫폼의 핵심 데이터 단위. 문서 접근 자체를 통제해야 함 |
| **접근 가능 주체** | 조직 멤버(역할에 따라), 외부 공유 수신자, API 클라이언트 |
| **읽기/수정/관리 구분 필요** | 필수. 읽기(Viewer), 수정(Editor), 관리(OrganizationAdmin 이상) 분리 필요 |
| **MVP 필요 여부** | **MVP 핵심** |

#### DocumentVersion

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 특정 버전은 초안(draft) 또는 내부 검토 중일 수 있어 공개 전 보호 필요 |
| **접근 가능 주체** | Document에 접근 가능한 주체 중 버전 열람/관리 권한 보유자 |
| **읽기/수정/관리 구분 필요** | 필요. 게시 버전은 Viewer 접근 가능, 초안 버전은 Editor 이상만 접근 |
| **MVP 필요 여부** | **MVP 핵심** — 초안/게시 구분이 핵심 운영 규칙 |

#### DocumentNode

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 문서 본문의 최소 구조 단위. 노드 단위 편집 권한이 필요할 수 있음 |
| **접근 가능 주체** | 상위 Document/Version에 접근 가능한 주체 |
| **읽기/수정/관리 구분 필요** | 기본은 상위 Document 권한을 상속. 노드 수준 독립 ACL은 확장 단계에서 검토 |
| **MVP 필요 여부** | **MVP 핵심 (상속 방식)** — 독립 ACL은 후속 |

#### Attachment

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 첨부 파일은 민감한 이진 데이터를 포함할 수 있음. 직접 URL 접근 제어 필요 |
| **접근 가능 주체** | 연결된 Document에 접근 가능한 주체. 단, 예외 공유 가능성 있음 |
| **읽기/수정/관리 구분 필요** | 읽기/업로드/삭제 구분 필요 |
| **MVP 필요 여부** | **MVP 포함** — Document와 함께 등장하는 자연스러운 부가 객체 |

#### Comment

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 내부 검토 의견이 포함될 수 있음. 작성/수정/삭제 주체 통제 필요 |
| **접근 가능 주체** | 상위 Document 열람 가능한 주체 중 Commenter 이상 |
| **읽기/수정/관리 구분 필요** | 필요. 자신이 작성한 코멘트만 수정/삭제 가능. 관리자는 모두 삭제 가능 |
| **MVP 필요 여부** | **MVP 포함** |

#### Annotation

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 노드 수준 주석. 개인 메모 vs 공유 주석 구분 필요 |
| **접근 가능 주체** | 작성자, Document 편집자 이상 |
| **읽기/수정/관리 구분 필요** | 개인/공유 모드 구분 필요 |
| **MVP 필요 여부** | **확장** — 초기에는 Comment로 통합 가능 |

#### Tag / Label

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 분류 체계 자체를 오용할 경우 검색/필터 악용 가능 |
| **접근 가능 주체** | 조직 멤버. 관리 태그는 관리자만 |
| **읽기/수정/관리 구분 필요** | 사용자 태그 vs 시스템 태그 구분 |
| **MVP 필요 여부** | **확장** — Metadata 확장 구조로 초기 흡수 가능 |

#### Link / Reference

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 문서 간 참조 관계. 비공개 문서에 대한 참조 노출 방지 필요 |
| **접근 가능 주체** | 참조 양쪽 Document에 접근 가능한 주체에 한해 관계 노출 |
| **읽기/수정/관리 구분 필요** | 읽기 접근 시 상대 문서 권한 연동 필요 |
| **MVP 필요 여부** | **확장** |

### 3-2. 관리 계열 리소스

#### Organization

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 테넌트 단위. 조직 설정, 멤버 관리, 전체 리소스 포함 |
| **접근 가능 주체** | PlatformAdmin(전체), OrganizationOwner/Admin(해당 조직) |
| **읽기/수정/관리 구분 필요** | 필수. 조회/설정 변경/삭제/멤버 관리 각각 통제 |
| **MVP 필요 여부** | **MVP 핵심** |

#### Workspace / Space / Folder

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 문서 컨테이너 단위. 공간 접근 자체를 통제하면 하위 문서를 일괄 보호 가능 |
| **접근 가능 주체** | 조직 멤버 중 해당 공간 접근 권한 보유자 |
| **읽기/수정/관리 구분 필요** | 공간 탐색/문서 생성/공간 관리 구분 |
| **MVP 필요 여부** | **MVP 포함** — 문서를 조직화하는 핵심 컨테이너. 단, 구조 단순화 검토 필요 |

#### DocumentType Definition

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 플랫폼 전체 문서 구조에 영향. 임의 변경 시 일관성 파괴 |
| **접근 가능 주체** | PlatformAdmin 또는 OrganizationAdmin(조직 커스텀 타입) |
| **읽기/수정/관리 구분 필요** | 읽기는 모든 멤버. 수정/삭제는 관리자 전용 |
| **MVP 필요 여부** | **MVP 포함 (관리자 전용)** |

#### Template

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 표준 양식. 무단 수정 시 일관성 파괴 |
| **접근 가능 주체** | Editor 이상(사용), OrganizationAdmin 이상(관리) |
| **읽기/수정/관리 구분 필요** | 사용 vs 관리 구분 필요 |
| **MVP 필요 여부** | **확장** |

#### Metadata Schema

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 문서 유형별 확장 필드 정의. 무단 수정 시 데이터 일관성 파괴 |
| **접근 가능 주체** | PlatformAdmin 또는 OrganizationAdmin |
| **읽기/수정/관리 구분 필요** | 읽기는 시스템 내부. 수정은 관리자 전용 |
| **MVP 필요 여부** | **확장** — 초기에는 시스템 정의 스키마만 사용 |

#### Permission Policy Object

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 권한 정책 자체를 변경할 수 있는 권한. 보안 최우선 보호 대상 |
| **접근 가능 주체** | PlatformAdmin(전역), OrganizationOwner/Admin(조직 범위) |
| **읽기/수정/관리 구분 필요** | 필수. 읽기(감사), 수정(관리자), 삭제(Owner급) |
| **MVP 필요 여부** | **MVP 핵심** |

#### Share Link / External Grant

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 외부에 문서를 공개하는 자격 부여 객체. 오용 시 정보 유출 |
| **접근 가능 주체** | 해당 문서의 관리 권한 보유자 |
| **읽기/수정/관리 구분 필요** | 생성/조회/취소 구분 |
| **MVP 필요 여부** | **확장** |

#### Workflow / Approval Object

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 문서 상태 전이의 공식 절차. 무결성 보호 필요 |
| **접근 가능 주체** | 워크플로 참여 역할(검토자, 승인자 등) |
| **읽기/수정/관리 구분 필요** | 역할별 전이 가능 액션 제한 |
| **MVP 필요 여부** | **확장 (Phase 5)** |

### 3-3. 운영 계열 리소스

#### AuditLog

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 보안/컴플라이언스 증거 자료. 무결성 훼손 방지 필수 |
| **접근 가능 주체** | SecurityAuditor, PlatformAdmin (읽기 전용). 수정/삭제 불가 원칙 |
| **읽기/수정/관리 구분 필요** | 읽기만 허용. 수정/삭제는 시스템 내부도 원칙적으로 불가 |
| **MVP 필요 여부** | **MVP 핵심** |

#### ActivityTrace

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 사용자 행동 분석 데이터. 개인정보 포함 가능 |
| **접근 가능 주체** | PlatformAdmin, OrganizationAdmin(조직 범위), 본인(자기 활동) |
| **읽기/수정/관리 구분 필요** | 읽기 범위 통제 핵심 |
| **MVP 필요 여부** | **확장 (Task 2-6 설계 후 결정)** |

#### API Credential / Service Account

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 인증 자격 증명. 유출 시 전체 시스템 접근 가능 |
| **접근 가능 주체** | 발급 주체(소유자), PlatformAdmin |
| **읽기/수정/관리 구분 필요** | 생성/조회/취소 구분. 실제 secret 값은 생성 시 1회만 노출 |
| **MVP 필요 여부** | **MVP 포함 (최소 사용자 토큰 관리)** |

#### Integration Configuration / Webhook

| 항목 | 내용 |
|---|---|
| **왜 보호 대상인가** | 외부 시스템 연동 설정. 오용 시 데이터 외부 유출 |
| **접근 가능 주체** | PlatformAdmin, OrganizationAdmin |
| **읽기/수정/관리 구분 필요** | 생성/조회/수정/삭제 구분 |
| **MVP 필요 여부** | **확장** |

---

## 4. 리소스 범위(Scope) 정의

### 4-1. Scope 레벨 구조

| Scope 레벨 | 설명 | 속하는 리소스 | 접근 주체 | 상위 Scope 영향 |
|---|---|---|---|---|
| **Platform** | 플랫폼 전체 | Organization, AuditLog(전체), DocumentType, API Credential, Webhook | PlatformOwner, PlatformAdmin, SecurityAuditor | 없음 (최상위) |
| **Organization** | 단일 조직 경계 | Organization 설정, Membership, Permission Policy, Space/Folder 목록, AuditLog(조직) | 조직 Membership 보유자 | Platform 역할이 override 가능 (Task 2-7 결정) |
| **Space / Container** | 조직 내 문서 묶음 | Space 설정, Space 내 Document 목록 | SpaceManager 이상, 또는 명시적 할당 | Organization 권한 상속. 독립 ACL 부여 가능 |
| **Document** | 단일 문서 | Document 메타, Version 목록, current_version 참조 | Document 접근 권한 보유자 | Space 권한 상속. 독립 ACL 부여 가능 |
| **Version** | 문서의 특정 버전 | Version 본문 스냅샷, Node 트리 | Document 접근 권한 상속. Draft 버전은 Editor 이상 한정 | Document 권한 상속. Draft/Published 구분 |
| **Node** | 버전 내 구조 단위 | Node 본문, 자식 Node | Version 접근 권한 상속 | Version 권한 상속. 독립 ACL은 확장 단계 |
| **Sub-resource** | Comment, Attachment, Annotation | 각 객체 | 상위 Document 접근 권한 상속. 작성자 추가 권한 | Document 권한 상속. 작성자별 수정/삭제 예외 |

### 4-2. 각 Scope의 독립 ACL 필요성

| Scope | 기본 상속 | 독립 ACL 필요 경우 |
|---|---|---|
| Platform | 해당 없음 | 항상 전역 역할로 통제 |
| Organization | Platform으로부터 독립 | 특정 조직에만 접근 허용하는 경우 |
| Space | Organization 상속 | 특정 팀에만 공간 접근 허용하는 경우 |
| Document | Space 상속 | 특정 문서를 공간 외부 사용자에게 공유하는 경우, 비공개 문서 |
| Version | Document 상속 | Draft 버전 접근을 Editor로 한정하는 경우 (기본 규칙으로 처리 가능) |
| Node | Version 상속 | 노드 단위 수정 권한 세분화 (확장 단계) |
| Sub-resource | Document 상속 | 작성자 본인 수정/삭제 (기본 규칙으로 처리 가능) |

---

## 5. 리소스 계층 구조 초안

### 5-1. 전체 계층 구조

```
Platform (전역)
├── Organization (조직)
│     ├── [관리] Membership, Permission Policy, DocumentType(조직 커스텀)
│     ├── [운영] AuditLog(조직 범위), ActivityTrace(조직 범위)
│     └── Space / Container (공간)
│           ├── [관리] Space 설정
│           └── Document (문서)
│                 ├── [부가] Attachment
│                 ├── [부가] Comment
│                 ├── [부가] Annotation
│                 ├── [참조] Tag/Label, Link/Reference
│                 └── Version (버전 스냅샷)
│                       └── Node (구조 단위)
│                             └── Node (자식, 재귀)
│
├── [전역 운영] AuditLog(플랫폼 전체)
├── [전역 관리] DocumentType(시스템 정의), Template(플랫폼)
└── [전역 보안] API Credential, Service Account, Webhook, Integration Config
```

### 5-2. 리소스 관계 분류

**계층형으로 보는 것이 맞는 리소스**
- Platform → Organization → Space → Document → Version → Node
- 상위 계층의 존재 없이 하위가 존재할 수 없는 구조

**참조형으로 보는 것이 맞는 리소스**
- Comment → Document (문서가 없어도 독립 존재 가능성 낮으나, 코멘트는 Document ID를 참조하는 형태)
- Attachment → Document (파일 자체는 스토리지에 독립 존재. Document와 참조 관계)
- Tag/Label → Document (다수 문서에 공유될 수 있음)
- Link/Reference → Document (양방향 참조 관계)

**상속형 권한이 어울리는 리소스**
- Version ← Document: 동일 접근 제어. Draft 여부만 추가 규칙
- Node ← Version: 본문 구조이므로 Version과 동일 접근 제어
- Attachment ← Document: 기본적으로 상위 문서 권한 상속
- Comment ← Document: 열람은 Document 접근 시 같이 허용

**독립 통제가 필요한 리소스**
- Space: 조직 내에서도 특정 팀에만 공개되는 공간 운영 가능
- Document: 공간 권한과 별개로 특정 문서만 외부 공유하는 경우
- AuditLog: 콘텐츠 역할(Editor, Viewer)과 무관하게 SecurityAuditor만 접근
- API Credential: 소유자 + 관리자만 접근. 콘텐츠 역할과 완전 분리
- Permission Policy Object: 권한 관리자(OrganizationOwner, PlatformAdmin)만 접근

### 5-3. 주요 경계 질문 답변

**Q. DocumentVersion은 Document 하위지만, 별도 접근 통제가 필요한가?**

> Draft 버전과 Published 버전의 접근 통제가 다르다. Published 버전은 Viewer도 접근 가능하나, Draft 버전은 Editor 이상만 접근 가능하다. 이는 별도 ACL 엔트리가 아닌 **Version 상태 기반 규칙**으로 처리한다. 별도 ACL 엔트리 없이 시스템 내장 규칙으로 충분하다.

**Q. Node는 Document의 일부로만 볼지, 독립 수정 권한을 둘지?**

> MVP에서는 Node는 상위 Version/Document 권한을 완전 상속한다. 노드 단위 독립 ACL(예: 특정 섹션만 특정 편집자에게 허용)은 고급 편집 시나리오로, 확장 단계에서 도입한다.

**Q. Attachment는 상위 문서 권한을 따르는지, 예외 공유가 가능한지?**

> 기본은 상위 Document 권한 상속. 단, 첨부파일을 직접 URL로 공유하는 시나리오를 위해 Share Link 기능 도입 시 예외 ACL을 적용할 수 있도록 설계한다. MVP에서는 상속만 적용.

**Q. Comment는 문서 조회 권한과 별개로 작성/삭제 통제가 필요한가?**

> 네. 조회(읽기)는 Document 열람 권한 상속이 적절하지만, 작성은 Commenter 이상 역할이 필요하고, 삭제는 작성자 본인 또는 관리자만 가능하다. 이는 **역할 기반 기본 규칙 + 소유자 예외**로 처리한다.

**Q. 운영 리소스는 콘텐츠 리소스와 분리된 권한 체계가 필요한가?**

> 예. AuditLog, API Credential, Permission Policy, Integration Config 등 운영 리소스는 콘텐츠 역할(Editor, Viewer 등)과 완전히 독립된 전용 역할(SecurityAuditor, PlatformAdmin 등)로만 접근 가능해야 한다. 두 체계를 혼합하지 않는다.

---

## 6. 보호 수준 분류

### 6-1. 보호 수준 정의

| 수준 | 정의 |
|---|---|
| **Read-sensitive** | 열람 자체를 통제해야 하는 리소스 |
| **Write-sensitive** | 수정/생성/삭제를 통제해야 하는 리소스 |
| **Admin-sensitive** | 역할/정책/멤버 등 운영 설정을 통제해야 하는 리소스 |
| **Security-sensitive** | 유출 시 전체 시스템 보안에 영향을 미치는 리소스 |
| **Compliance-sensitive** | 법적/감사 목적으로 무결성이 보장되어야 하는 리소스 |

### 6-2. 리소스별 보호 수준 분류

| 리소스 | Read | Write | Admin | Security | Compliance | 분류 근거 |
|---|:---:|:---:|:---:|:---:|:---:|---|
| Document (published) | ● | ● | | | | 열람 통제 + 수정 권한 분리 필요 |
| Document (draft) | ● | ● | | | | 미공개 상태. 내부 관계자만 열람 가능 |
| DocumentVersion (published) | ● | | | | ● | 불변 스냅샷. 수정 불가. 컴플라이언스 증거 |
| DocumentVersion (draft) | ● | ● | | | | 초안 단계. Editor 이상만 접근 |
| DocumentNode | ● | ● | | | | Version 상속. 본문 수정 통제 |
| Attachment | ● | ● | | | | 파일 직접 접근 통제 |
| Comment | ● | ● | | | | 작성/삭제 주체 통제 |
| Organization | ● | | ● | | | 조직 설정 변경은 관리자만 |
| Space | ● | | ● | | | 공간 접근 + 설정 분리 |
| DocumentType | ● | | ● | | | 플랫폼 구조 일관성 영향 |
| Permission Policy | | | ● | ● | ● | 권한 정책 자체. 변경 이력 추적 필수 |
| AuditLog | ● | | | | ● | 읽기 전용. 무결성 보장 필수 |
| ActivityTrace | ● | | ● | | | 개인정보 포함 가능. 접근 범위 제한 |
| API Credential | | | | ● | | 유출 시 시스템 전체 영향 |
| Integration Config | | | ● | ● | | 외부 연동. 데이터 유출 경로 |
| Share Link | ● | ● | | ● | | 외부 공개 권한. 오용 방지 |
| Membership | ● | | ● | | ● | 소속 관계 변경은 감사 대상 |

---

## 7. 권한 상속 및 독립 통제 원칙

### 7-1. 상속 관련 핵심 질문 답변

**Q. 상위 리소스 권한이 하위 리소스에 자동 상속되어야 하는가?**

> 예. 기본 원칙은 상속이다. 상위 Space 접근 권한이 있으면 하위 Document 접근이 가능하고, Document 접근 권한이 있으면 Version, Node, Comment, Attachment 접근이 가능하다. 이 원칙이 없으면 모든 리소스에 개별 ACL을 설정해야 하는 운영 부담이 발생한다.

**Q. 모든 하위 리소스가 상속을 따르게 하면 어떤 문제가 생기는가?**

> - 특정 문서만 공간 외부 사용자에게 공유 불가 → Document 수준 독립 ACL 필요
> - 특정 공간만 비공개로 운영 불가 → Space 수준 독립 ACL 필요
> - 민감 첨부파일을 문서 Viewer에게 숨기기 불가 → 확장 단계에서 Attachment 독립 ACL 검토
> 따라서 **상속을 기본으로 하되, Space와 Document 수준에서 독립 ACL 부여를 허용**한다.

**Q. 예외 ACL이 자주 필요한 리소스는 무엇인가?**

> Space, Document. 이 두 리소스가 독립 ACL 부여의 주요 대상이다.

**Q. 문서 버전은 문서와 동일 권한으로 볼지, 별도 보호가 필요한가?**

> 기본은 Document 권한 상속. 단, Draft 버전에 대한 추가 제한(Editor 이상만 접근)은 **시스템 내장 규칙**으로 처리한다. 별도 ACL 엔트리가 아닌 Version의 `status` 필드 기반 평가로 충분하다.

**Q. 댓글/첨부/주석은 상위 문서 권한으로 충분한가?**

> 열람은 충분하다. 그러나 **작성/삭제는 역할 + 소유자 규칙**이 필요하다. Commenter 이상이어야 작성 가능하고, 삭제는 작성자 본인 또는 관리자만 가능하다. 이는 ACL 엔트리가 아닌 **소유자 예외 규칙**으로 처리한다.

**Q. 운영 리소스는 콘텐츠 리소스와 분리된 권한 체계가 필요한가?**

> 필수. AuditLog, API Credential, Permission Policy, Integration Config는 콘텐츠 역할과 독립된 별도 체계로 관리한다. 콘텐츠 역할(Editor, Viewer 등)이 운영 리소스에 접근하는 경로를 원천 차단한다.

### 7-2. 원칙 정리

#### 기본 상속 원칙

- 권한은 상위 Scope에서 하위 Scope로 상속된다
- 상속 경로: Organization → Space → Document → Version → Node / Attachment / Comment
- 상속은 명시적 ACL이 없을 때의 기본 평가 결과다

#### 예외 부여 가능 리소스 (독립 ACL 허용)

- **Space**: 조직 내에서도 특정 그룹에만 공개되는 공간
- **Document**: 공간 권한 외 특정 문서의 개별 공유

#### 독립 관리 리소스 (상속과 무관)

- **AuditLog**: 콘텐츠 역할과 독립. SecurityAuditor, PlatformAdmin 전용
- **Permission Policy**: OrganizationOwner/Admin, PlatformAdmin 전용
- **API Credential**: 소유자 + PlatformAdmin 전용
- **Integration Config / Webhook**: OrganizationAdmin, PlatformAdmin 전용

#### 보안상 별도 분리 리소스

- **API Credential / Service Account**: 최고 보안 등급. 접근 이력 전량 감사 대상
- **Permission Policy**: 수정 시 반드시 감사 로그 기록
- **AuditLog**: 읽기 전용 원칙. 시스템 내부도 삭제 불가 (보존 정책 별도)

---

## 8. MVP 우선 보호 대상

### 8-1. MVP 핵심 보호 대상

| 리소스 | 보호 이유 | 최소 통제 액션 |
|---|---|---|
| Organization | 테넌트 경계 | 조회 / 설정 변경 / 멤버 관리 |
| Space | 문서 컨테이너 접근 제어 | 탐색 / 문서 생성 / 설정 변경 |
| Document | 핵심 콘텐츠 단위 | 조회 / 수정 / 삭제 / 공유 |
| DocumentVersion | 초안/게시 구분 | 조회(draft 한정) / 게시 / 삭제 |
| DocumentNode | 본문 수정 통제 | 조회 / 수정 (Document 상속) |
| Attachment | 파일 접근 통제 | 조회 / 업로드 / 삭제 |
| Comment | 코멘트 주체 통제 | 조회 / 작성 / 삭제(본인+관리자) |
| Permission Policy | 권한 체계 보호 | 조회(감사) / 수정(관리자) |
| AuditLog | 컴플라이언스 | 조회(감사 역할 전용) |
| API Credential | 인증 자격 보호 | 발급 / 조회 / 취소 |

### 8-2. 확장 단계 보호 대상

| 리소스 | 도입 시점 | 이유 |
|---|---|---|
| Share Link / External Grant | Phase 3~4 | 외부 공유 기능 도입 시 |
| Workflow / Approval Object | Phase 5 | 승인 워크플로 도입 시 |
| Template | Phase 3~4 | 템플릿 기능 도입 시 |
| Metadata Schema | Phase 3 이후 | 커스텀 스키마 관리 도입 시 |
| Integration Config / Webhook | Phase 3 이후 | 외부 연동 기능 도입 시 |
| ActivityTrace | Phase 2-6 설계 후 | 행위 추적 체계 확정 시 |
| Annotation | Comment 통합 이후 | 별도 주석 기능 도입 시 |
| Tag/Label (독립 관리) | Phase 4 이후 | 독립 태그 체계 도입 시 |
| Node (독립 ACL) | Phase 6 이후 | 고급 편집 권한 세분화 시 |

---

## 9. 다음 Task로 넘길 결정사항

### Task 2-3 (ACL 기반 권한 모델 설계) 에 전달

**직접 ACL이 부여되는 리소스 (principal이 직접 접근하는 단위)**

| 리소스 | principal 유형 | 비고 |
|---|---|---|
| Organization | User(GlobalRoleAssignment), Membership | 조직 레벨 접근 통제 |
| Space | Membership, (Team: 확장) | 공간 접근 독립 ACL |
| Document | Membership, (Team: 확장), Share Link principal | 문서 독립 공유 ACL |
| Permission Policy | Membership (Owner/Admin) | 권한 정책 수정 통제 |
| AuditLog | User (SecurityAuditor 전용) | 콘텐츠 역할과 분리 |
| API Credential | User (소유자), PlatformAdmin | 보안 리소스 |

**상위 리소스 권한을 상속받는 리소스**

| 리소스 | 상속 경로 |
|---|---|
| DocumentVersion | ← Document |
| DocumentNode | ← Version ← Document |
| Attachment | ← Document |
| Comment (읽기) | ← Document |

**예외 규칙이 추가되는 리소스**

| 리소스 | 예외 규칙 |
|---|---|
| DocumentVersion (draft) | Document 접근 가능자 중 Editor 이상만 열람 |
| Comment (수정/삭제) | 작성자 본인 또는 관리자 |
| Attachment (삭제) | 업로더 본인 또는 관리자 |

**운영자 전용 리소스 (콘텐츠 역할 접근 불가)**

- AuditLog (SecurityAuditor, PlatformAdmin)
- API Credential (소유자, PlatformAdmin)
- Permission Policy (OrganizationOwner/Admin, PlatformAdmin)
- Integration Config / Webhook (OrganizationAdmin, PlatformAdmin)

**감사 대상 중요 리소스 (모든 접근/변경 기록 필수)**

- Permission Policy (수정/삭제 시 감사 필수)
- Membership (초대/제거/역할 변경 시 감사 필수)
- Document (생성/상태 전이/삭제 시 감사 필수)
- AuditLog 자체 접근 (누가 언제 조회했는지 기록)
- API Credential (발급/취소 감사 필수)

### Task 2-4 (API Authorization Enforcement) 에 전달

- API 요청의 resource_type 추출 후 scope 레벨 판별 → 적합한 권한 평가 경로 선택
- Space/Document 수준에서 독립 ACL이 존재할 수 있음 → 상속 + 직접 ACL 병합 평가 필요
- Version 접근 시 status(draft/published) 추가 규칙 적용 필요
- 운영 리소스는 콘텐츠 역할 평가 경로와 분리된 별도 검사 필요

### Task 2-5 (감사 로그 설계) 에 전달

- 감사 대상 중요 리소스 목록 (위 참조)
- AuditLog 자체는 읽기 전용 + 무결성 보장 필수
- Permission Policy 수정, Membership 변경, Document 상태 전이는 반드시 감사 이벤트 생성

---

## 10. 오픈 이슈

| # | 이슈 | 설명 | 결정 필요 시점 |
|---|---|---|---|
| OI-1 | Space(공간) 개념의 계층 단순화 여부 | Space / Folder / Workspace를 단일 개념으로 통합할지, 계층을 둘지. 단순화 시 Folder가 Document를 담는 구조만 남음 | Task 2-3 ACL 설계 전 결정 필요 |
| OI-2 | Document 독립 ACL과 Space 상속의 우선순위 | Space 접근 권한이 없는 사람에게 Document 직접 ACL을 부여할 수 있는가 | Task 2-3 ACL 설계 또는 Task 2-7 |
| OI-3 | Draft Version에 대한 접근 규칙을 ACL로 표현할지, 시스템 규칙으로 처리할지 | ACL 엔트리 없이 Version.status 필드 기반으로 처리하는 방향이 단순하지만, ACL로 표현해야 감사 추적이 명확해짐 | Task 2-3 |
| OI-4 | Node 수준 독립 ACL 도입 시점 | MVP에서는 상속만. 도입 조건(대용량 협업, 섹션별 접근 제어)이 명확해지면 확장 | Phase 6 이후 |
| OI-5 | ActivityTrace의 개인정보 보호 범위 | 어느 수준의 행동 데이터를 수집하고 누구에게 공개할지. GDPR 등 컴플라이언스 요구사항 연동 | Task 2-6 설계 |

---

*작성일: 2026-04-01*
*Phase: 2 / Task: 2-2*
*상태: 설계 초안 완료*
