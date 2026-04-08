# Phase 2 - Task 2-5. 감사 로그(Audit Log) 체계 설계

---

## 1. 목표

보안/컴플라이언스/사후 추적/운영 조사에 활용할 수 있도록, **누가, 언제, 어떤 자격으로, 어떤 리소스에 대해, 어떤 행위를 했고, 그 결과가 무엇이었는지**를 추적 가능한 감사 로그 체계를 설계한다.

- 감사 로그의 목적과 범위를 일반 로그/Activity Trace와 명확히 구분한다
- 감사 대상 이벤트 범주와 naming 규칙을 정의한다
- Audit Log 레코드의 핵심 필드 구조를 확정한다
- 기록 대상/비대상 원칙과 before/after 기록 수준을 결정한다
- 무결성/보존/접근 통제 원칙을 정립한다
- Task 2-4 Authorization Decision과의 연계 방식을 정의한다
- Task 2-6 Activity Trace 설계로 넘길 경계와 연계점을 정리한다

---

## 2. 설계 전제

| # | 전제 |
|---|---|
| 1 | API-first 플랫폼. UI, 외부 API, 서비스 계정 모두 감사 대상 |
| 2 | 감사 로그는 일반 운영/디버그 로그와 완전히 분리된 독립 체계 |
| 3 | Task 2-4 결과: Authorization Decision에서 decision_metadata가 생성됨. 이를 감사 로그에 연결 |
| 4 | 감사 로그는 append-only. 수정/삭제 불가 원칙 |
| 5 | 콘텐츠 원문 저장 최소화. 감사 목적에 필요한 요약만 저장 |
| 6 | denied와 failed는 반드시 구분하여 기록 |
| 7 | MVP는 보안/컴플라이언스 핵심 이벤트 중심. 모든 이벤트를 다 남기지 않음 |

---

## 3. 감사 로그의 역할과 범위

### 3-1. 로그 유형 비교

| 유형 | 목적 | 생성 주체 | 보존 | 변조 방지 |
|---|---|---|---|---|
| **감사 로그 (Audit Log)** | 보안/컴플라이언스/사후 추적 | 비즈니스 이벤트 발생 시 명시적 생성 | 장기 (수년) | 필수. append-only |
| **Activity Trace** | 사용자 행동 흐름 분석, 운영 개선 | 탐색/조회 흐름 포함 광범위 수집 | 중기 (수개월) | 권장 |
| **애플리케이션 로그** | 디버깅, 오류 추적 | 코드 실행 흐름에서 자동 생성 | 단기 (수주) | 불필요 |
| **운영/메트릭 로그** | 성능 모니터링, 상태 추적 | 시스템 자동 수집 | 단기~중기 | 불필요 |

### 3-2. 감사 로그가 별도 모델이어야 하는 이유

- **접근 통제가 다르다**: 감사 로그는 SecurityAuditor/PlatformAdmin만 열람 가능. 일반 운영 로그와 동일 시스템에 두면 통제 경계가 무너짐
- **보존 정책이 다르다**: 컴플라이언스 요구사항(수년 보존)이 운영 로그(수주)와 다름
- **무결성 요구가 다르다**: 감사 로그는 append-only. 삭제/수정 불가. 운영 로그는 로테이션 가능
- **목적이 다르다**: "무슨 일이 있었는가" + "누가 어떤 자격으로" — 이 두 질문에 동시에 답해야 함

### 3-3. Activity Trace와의 경계

| 기준 | 감사 로그 | Activity Trace |
|---|---|---|
| **기록 목적** | 보안/법적 증적 | 사용자 행동 패턴 분석 |
| **기록 단위** | 명시적 행위 이벤트 (생성, 삭제, 권한 변경 등) | 탐색 흐름, 조회 시퀀스, 페이지 전환 등 |
| **read 포함 여부** | 민감 리소스 read만 포함 | 모든 조회 포함 |
| **보존** | 법적 기준 (수년) | 분석 목적 (수개월) |
| **접근 권한** | 감사 역할 전용 | 운영/분석 팀 |
| **무결성** | 필수 (append-only) | 권장 |
| **correlation** | request_id 공유 가능 | trace_id로 연결 |

### 3-4. 감사 로그의 핵심 목적

| 목적 | 설명 |
|---|---|
| **보안 사고 대응** | 침해 사고 발생 시 "언제, 누가, 무엇을 했는가" 타임라인 재구성 |
| **내부 통제** | 권한 남용, 무단 접근, 비정상적 행위 탐지 |
| **컴플라이언스 증적** | 규정 준수 여부 입증. 외부 감사/보안 심사 대응 |
| **권한 남용 추적** | 역할/ACL 변경 이력. "누가 권한을 부여했는가" 추적 |
| **민감 리소스 접근 이력** | 기밀 문서, 운영 리소스, API 자격 증명 접근 기록 |

---

## 4. 감사 대상 이벤트 범주

### 4-1. 인증/세션/접근 관련

| 이벤트 | 감사 이유 | 기록 조건 | MVP |
|---|---|---|---|
| 로그인 성공 | 접근 주체 식별의 시작점 | 항상 | 필수 |
| 로그인 실패 | 무차별 대입 공격 탐지, 계정 탈취 시도 | 항상 | 필수 |
| 토큰 발급 | 인증 세션 생성 추적 | 항상 | 필수 |
| 토큰 폐기/로그아웃 | 세션 종료 추적 | 항상 | 필수 |
| 서비스 계정 사용 | 비인간 주체 행위 추적 | 항상 | 필수 |
| Authorization denied | 권한 거부 패턴 탐지 | 민감 리소스 deny 필수. 일반 콘텐츠는 선택 | 필수(민감) |
| 외부 API 호출 (Service Account) | 외부 클라이언트 행위 추적 | 항상 | 확장 |

### 4-2. 문서/콘텐츠 관련

| 이벤트 | 감사 이유 | 기록 조건 | MVP |
|---|---|---|---|
| 문서 생성 | 콘텐츠 생명주기 시작 | 성공 시 | 필수 |
| 문서 수정 (새 Version 생성) | 변경 이력 추적 | 성공 시 | 필수 |
| 문서 삭제 | 데이터 손실 추적, 복구 근거 | 성공 시 | 필수 |
| 문서 게시 (published) | 공식 배포 행위 | 성공 시 | 필수 |
| 문서 아카이브 | 상태 전이 | 성공 시 | 필수 |
| Version 복원 | 과거 버전 복원. 현재 내용 대체 | 성공 시 | 필수 |
| 문서 export/download | 데이터 유출 경로 | 성공 시 | 필수 |
| 공유 링크 생성/폐기 | 외부 접근 경로 생성 | 성공 시 | 확장 |
| Attachment 업로드/삭제 | 파일 데이터 변경 | 성공 시 | 필수 |
| Comment 생성/삭제 | 검토 의견 추적 | 성공 시 | 확장 |
| 민감 문서 read | 기밀 문서 접근 이력 | 조건부 (민감 태그 시) | 확장 |

### 4-3. 권한/조직 관련

| 이벤트 | 감사 이유 | 기록 조건 | MVP |
|---|---|---|---|
| 조직 생성/변경/삭제 | 테넌트 관리 추적 | 항상 | 필수 |
| 멤버 추가/제거 | 접근 주체 변경 | 항상 | 필수 |
| 역할 부여/회수 | 권한 변경. 남용 탐지의 핵심 | 항상 | 필수 |
| ACL 생성/수정/삭제 | 리소스 단위 권한 변경 | 항상 | 필수 |
| Permission Policy 변경 | 정책 자체 변경. 최고 감사 등급 | 항상 | 필수 |
| 전역 역할 변경 (PlatformAdmin 부여 등) | 플랫폼 전체 권한 변경 | 항상 | 필수 |

### 4-4. 운영/보안 관련

| 이벤트 | 감사 이유 | 기록 조건 | MVP |
|---|---|---|---|
| API Credential 생성/폐기 | 인증 자격 변경. 보안 필수 | 항상 | 필수 |
| Integration 설정 변경 | 외부 데이터 유출 경로 변경 | 항상 | 확장 |
| Webhook/Connector 변경 | 이벤트 전달 경로 변경 | 항상 | 확장 |
| 감사 로그 조회 | "누가 감사 로그를 봤는가" 재감사 | 항상 | 필수 |
| 보안 설정 변경 (MFA, SSO 등) | 인증 정책 변경 | 항상 | 확장 |
| 시스템 관리자 액션 (백업, 마이그레이션) | 운영 행위 추적 | 항상 | 확장 |

---

## 5. Audit Event Taxonomy 초안

### 5-1. Naming Convention 설계

**권장 형식: `{domain}.{object}.{action}`**

- `domain`: 최상위 도메인 구분 (auth, document, version, space, org, permission, admin)
- `object`: 행위 대상 객체 (선택. 도메인과 동일하면 생략 가능)
- `action`: 동사 과거형 또는 동사 원형 (created, updated, deleted, published, granted, revoked)

result(success/denied/failed)는 이벤트 이름에 포함하지 않고 **레코드의 별도 필드**로 분리한다.
이유: 이벤트 이름이 동일해야 쿼리/집계가 단순해짐. `document.created`가 성공/실패를 포함하면 쿼리가 두 배로 복잡해짐.

### 5-2. MVP 이벤트 목록

#### 인증/접근

| 이벤트명 | 설명 |
|---|---|
| `auth.login.attempted` | 로그인 시도 (성공/실패 result 필드로 구분) |
| `auth.token.issued` | 토큰 발급 |
| `auth.token.revoked` | 토큰 폐기/로그아웃 |
| `auth.access.denied` | Authorization deny 발생 |

#### 문서/콘텐츠

| 이벤트명 | 설명 |
|---|---|
| `document.created` | 문서 생성 |
| `document.updated` | 문서 수정 (새 Version 생성) |
| `document.deleted` | 문서 삭제 (소프트) |
| `document.published` | 문서 게시 상태 전이 |
| `document.archived` | 문서 아카이브 |
| `document.exported` | 문서 내보내기/다운로드 |
| `document.version.restored` | 이전 버전 복원 |
| `document.attachment.uploaded` | 첨부파일 업로드 |
| `document.attachment.deleted` | 첨부파일 삭제 |

#### 권한/조직

| 이벤트명 | 설명 |
|---|---|
| `org.created` | 조직 생성 |
| `org.updated` | 조직 설정 변경 |
| `org.deleted` | 조직 삭제/아카이브 |
| `org.membership.added` | 멤버 추가 |
| `org.membership.removed` | 멤버 제거 |
| `org.membership.role.granted` | 역할 부여 |
| `org.membership.role.revoked` | 역할 회수 |
| `permission.acl.created` | ACL 엔트리 생성 |
| `permission.acl.updated` | ACL 엔트리 수정 |
| `permission.acl.deleted` | ACL 엔트리 삭제 |
| `permission.policy.updated` | 권한 정책 변경 |
| `permission.global_role.granted` | 전역 역할 부여 |
| `permission.global_role.revoked` | 전역 역할 회수 |

#### 운영/보안

| 이벤트명 | 설명 |
|---|---|
| `admin.api_credential.created` | API Credential 발급 |
| `admin.api_credential.revoked` | API Credential 폐기 |
| `audit.log.accessed` | 감사 로그 조회 |

### 5-3. 확장 이벤트 목록 (Phase 3 이후)

| 이벤트명 | 도입 시점 |
|---|---|
| `document.share_link.created` | Share Link 기능 |
| `document.share_link.revoked` | Share Link 기능 |
| `document.comment.created` | Comment 기능 |
| `document.comment.deleted` | Comment 기능 |
| `document.read` | 민감 문서 read 감사 |
| `admin.integration.updated` | 외부 연동 |
| `admin.webhook.updated` | 웹훅 관리 |
| `workflow.approval.submitted` | 승인 워크플로 |
| `workflow.approval.approved` | 승인 완료 |
| `workflow.approval.rejected` | 승인 반려 |

---

## 6. Audit Log 레코드 구조

### 6-1. 핵심 필드 정의

```
AuditLog {

  // ─── 레코드 식별 ───────────────────────────────
  id:                   string      // 고유 식별자 (UUID). 불변
  event_type:           string      // "document.created", "permission.acl.updated" 등
  action_category:      string      // "content" | "permission" | "auth" | "admin"
  result:               string      // "success" | "denied" | "failed"
  occurred_at:          datetime    // 이벤트 발생 시각 (UTC, immutable)

  // ─── Actor (누가) ───────────────────────────────
  actor_type:           string      // "user" | "service_account" | "system"
  actor_id:             string      // user_id 또는 service_account_id
  on_behalf_of:         string|null // 위임 접근 시 원래 사용자 ID
  acting_membership_id: string|null // 어느 조직 Membership으로 행동했는가
  acting_org_id:        string|null // 행동 컨텍스트 조직
  acting_roles:         string[]    // 평가 시 적용된 역할 목록 (스냅샷)
  auth_method:          string|null // "session" | "api_key" | "oauth" | "sso"
  request_source:       string      // "user_ui" | "admin_ui" | "external_api" | "service_account"

  // ─── Target (무엇에 대해) ────────────────────────
  target_resource_type: string      // "document" | "space" | "org" | "acl" | ...
  target_resource_id:   string|null // 특정 리소스 ID
  target_parent_type:   string|null // 상위 리소스 유형 (nested resource 시)
  target_parent_id:     string|null // 상위 리소스 ID
  target_org_id:        string|null // 대상 리소스 소속 조직
  target_subject_id:    string|null // 권한 변경 시 대상 사용자/서비스

  // ─── Decision (왜 허용/거부) ─────────────────────
  allowed:              boolean     // true | false
  denial_reason:        string|null // Task 2-4 reason code (rbac_insufficient_role 등)
  matched_permission:   string|null // 평가된 permission (document:update 등)
  matched_role:         string|null // RBAC 경로 시 매칭된 역할
  matched_acl_entry_id: string|null // ACL 경로 시 매칭된 ACL 엔트리 ID
  authorization_path:   string|null // "rbac" | "acl_direct" | "acl_inherited" | "default_deny"

  // ─── Change Summary (무엇이 바뀌었는가) ──────────
  before_summary:       object|null // 변경 전 요약 (full snapshot 금지)
  after_summary:        object|null // 변경 후 요약
  changed_fields:       string[]    // 변경된 필드 목록
  policy_reason:        string|null // 변경 사유 (ACL 생성 시 reason 필드 등)

  // ─── Context (요청 추적) ────────────────────────
  request_id:           string      // 요청 단위 correlation ID
  trace_id:             string|null // 분산 추적 trace ID (확장)
  ip_address:           string|null // 요청 IP (보안 분석용)
  user_agent:           string|null // 클라이언트 정보 (선택)
  api_endpoint:         string|null // 호출된 API 경로 (예: PUT /documents/{id})
  api_method:           string|null // HTTP method
}
```

### 6-2. 필드별 상세 설명

| 필드 그룹 | 핵심 필드 | 필수 | MVP | 민감도 | 설명 |
|---|---|:---:|:---:|---|---|
| 레코드 식별 | `id`, `event_type`, `result`, `occurred_at` | 필수 | 포함 | 낮음 | 모든 레코드의 기본 |
| Actor | `actor_type`, `actor_id` | 필수 | 포함 | 중간 | "누가"의 기본 |
| Actor | `acting_membership_id`, `acting_org_id`, `acting_roles` | 필수 | 포함 | 낮음 | "어떤 자격으로" — 감사의 핵심 |
| Actor | `auth_method`, `request_source` | 권장 | 포함 | 낮음 | 접근 경로 식별 |
| Actor | `on_behalf_of` | 선택 | 포함 | 중간 | 위임 접근 시만 |
| Target | `target_resource_type`, `target_resource_id` | 필수 | 포함 | 낮음 | "무엇에" |
| Target | `target_org_id`, `target_parent_type/id` | 권장 | 포함 | 낮음 | 범위 추적 |
| Target | `target_subject_id` | 선택 | 포함 | 중간 | 권한 변경 대상 시만 |
| Decision | `allowed`, `denial_reason`, `authorization_path` | 필수 | 포함 | 낮음 | "왜" — Task 2-4 연계 |
| Decision | `matched_permission`, `matched_role`, `matched_acl_entry_id` | 권장 | 포함 | 낮음 | 판단 근거 |
| Change | `changed_fields`, `before_summary`, `after_summary` | 선택 | 부분 포함 | **높음** | 콘텐츠 원문 제외 |
| Change | `policy_reason` | 선택 | 포함 | 낮음 | 권한/정책 변경 시 |
| Context | `request_id` | 필수 | 포함 | 낮음 | 요청 추적 |
| Context | `ip_address` | 권장 | 포함 | **높음** | 개인정보. 마스킹 검토 |
| Context | `api_endpoint`, `api_method` | 선택 | 포함 | 낮음 | 경로 추적 |
| Context | `trace_id`, `user_agent` | 선택 | 확장 | 낮음 | 분산 추적 확장 시 |

---

## 7. 기록 대상 / 비대상 원칙

### 7-1. 핵심 질문 답변

**Q. document read도 모두 감사 로그로 남겨야 하는가?**
> 아니다. 일반 문서 read는 감사 로그가 아닌 Activity Trace로 수집한다. 단, 민감 태그가 붙은 기밀 문서나 Draft 상태 문서의 read는 조건부로 감사 로그에 기록한다.

**Q. list/search 조회는 어떤 수준까지 남겨야 하는가?**
> MVP에서는 list/search 자체는 감사 로그에 기록하지 않는다. 검색 쿼리나 목록 조회는 Activity Trace로 처리한다. 단, 감사 로그/운영 리소스 목록 조회(예: GET /audit-logs)는 기록한다.

**Q. 실패한 로그인과 권한 거부는 반드시 남겨야 하는가?**
> 예. 로그인 실패는 무차별 대입 공격 탐지의 핵심 데이터이며, 권한 거부는 내부 통제와 보안 이상 탐지에 필수다. 특히 민감 리소스(Permission Policy, API Credential, AuditLog 등)에 대한 denied는 전량 필수 기록이다.

**Q. before/after 전체 snapshot을 저장하면 어떤 위험이 있는가?**
> - **저장 비용 폭증**: 문서 본문이 수백 KB라면 버전마다 두 개 복사본이 감사 로그에 쌓임
> - **민감 정보 노출**: 감사 로그 접근자에게 문서 전체 내용이 노출됨. 감사 담당자가 기밀 문서 내용을 볼 필요는 없음
> - **GDPR 등 규정 충돌**: 개인정보가 포함된 문서 본문을 장기 보존하면 삭제권 충돌 가능

**Q. 민감 문서 본문을 감사 로그에 넣어도 되는가?**
> 원칙적으로 금지. 문서 제목, 문서 ID, 변경된 섹션 수 정도의 요약만 허용한다.

### 7-2. 원칙 정리

**반드시 기록할 행위**
- 모든 인증 시도 (성공/실패)
- 문서 생성/수정/삭제/게시/아카이브
- 문서 export/download
- 역할 부여/회수, ACL 변경, Permission Policy 변경
- 조직/멤버십 변경
- API Credential 발급/폐기
- 민감 리소스 접근 거부 (auth.access.denied)
- 감사 로그 조회 행위 자체

**조건부 기록할 행위**
- 민감 태그 문서 read (민감 분류 기능 도입 시)
- 문서 Version 복원
- 관리자 설정 변경

**기록하지 않을 행위**
- 일반 문서 read (Activity Trace로 처리)
- list/search 조회 (Activity Trace로 처리)
- UI 페이지 탐색, 클릭 이벤트
- 내부 시스템 health check, 메트릭 수집

**요약만 기록할 행위**
- 문서 내용 수정: 변경된 섹션 수, 변경 전/후 제목, version_id만 기록. 본문 전체 금지
- 메타데이터 변경: 변경된 필드명 목록만 기록 (changed_fields). 값은 요약

**콘텐츠 원문을 넣지 말아야 할 항목**
- 문서 본문 (Node content)
- 코멘트 내용
- 메타데이터 확장 필드 값 (유형별 상세 데이터)
- 첨부파일 내용 (파일명, 크기만 기록)
- ACL `reason` 필드의 장문 설명 (요약 수준만)

---

## 8. Success / Failed / Denied 기록 원칙

### 8-1. 결과 코드 정의

| result | 의미 |
|---|---|
| `success` | 인증/권한 통과 + 비즈니스 로직 성공 |
| `denied` | 인증은 됐으나 Authorization 거부. 또는 인증 자체 실패 |
| `failed` | 인증/권한 통과 후 비즈니스 로직/시스템 오류 |

### 8-2. 핵심 질문 답변

**Q. 시스템 오류로 실패한 경우와 권한 거부를 어떻게 구분하는가?**
> - `denied`: Authorization Service가 deny를 반환한 경우. 또는 인증 토큰 실패
> - `failed`: Authorization 통과 후, Repository 오류/비즈니스 규칙 위반/외부 서비스 장애 등으로 처리 실패
> 두 코드를 명확히 분리하면 "보안 사고인가, 시스템 장애인가"를 즉시 구분할 수 있다.

**Q. 허용되었지만 후속 비즈니스 규칙으로 실패한 경우?**
> `result: "failed"` + `failure_reason: "business_rule_violation"` (예: archived 문서 수정 시도). 이 경우 authorization은 통과했으나 상태 제약으로 실패한 것이므로 구분하여 기록.

**Q. 존재 은닉을 위해 API는 404를 주더라도, 내부 audit에는 denied로 남겨야 하는가?**
> 예. API 응답과 감사 로그는 독립적이다. 외부에는 404를 주지만, 감사 로그 내부에는 `result: "denied"`, `denial_reason: "access_denied_existence_hidden"` 으로 정확히 기록한다. 이렇게 해야 사후 조사 시 "접근 시도가 있었음"을 파악할 수 있다.

**Q. permission change 실패도 남겨야 하는가?**
> 예. 권한 변경 시도 자체가 중요한 감사 데이터다. 성공뿐 아니라 실패한 권한 변경 시도도 `result: "failed"` 또는 `result: "denied"`로 기록한다.

### 8-3. 결과별 기록 원칙

| 이벤트 유형 | success | denied | failed |
|---|:---:|:---:|:---:|
| 로그인 | 필수 | 필수 | 필수 |
| 문서 생성/수정/삭제 | 필수 | 필수 (민감 리소스) | 선택 |
| 권한/역할/ACL 변경 | 필수 | 필수 | 필수 |
| 문서 export | 필수 | 필수 | 선택 |
| 감사 로그 조회 | 필수 | 필수 | 선택 |
| 일반 문서 read | 기록 안 함 | 민감 리소스만 | 기록 안 함 |

---

## 9. Before/After 및 변경 요약 원칙

### 9-1. 기록 수준 비교

| 방식 | 저장 내용 | 장점 | 단점 |
|---|---|---|---|
| **changed_fields만** | 변경된 필드 이름 목록 | 가볍다 | "무엇이 어떻게 바뀌었는지" 파악 불가 |
| **before/after summary** | 변경 전/후 핵심 요약 | 균형 | 요약 범위 정의 필요 |
| **full snapshot** | 전체 객체 상태 | 완전한 재현 가능 | 저장 비용 폭증, 민감 정보 노출 위험 |

### 9-2. 리소스별 권장 기록 수준

**일반 콘텐츠 객체 (Document, Attachment)**

```
// 권장: before/after summary + changed_fields
{
  changed_fields: ["title", "status"],
  before_summary: { title: "이전 제목", status: "draft", version_id: "ver_001" },
  after_summary:  { title: "새 제목",  status: "published", version_id: "ver_002" }
}
// 금지: 본문(Node content) 포함
```

**권한/정책 객체 (ACL, Role Assignment, Permission Policy)**

```
// 권장: full snapshot 허용 (단, 정책 객체는 크기가 작고 보안상 완전 재현 필요)
{
  changed_fields: ["role_id"],
  before_summary: { membership_id: "mbr_001", role: "viewer" },
  after_summary:  { membership_id: "mbr_001", role: "editor" },
  policy_reason:  "프로젝트 참여로 인한 편집 권한 부여"
}
```

**보안 민감 객체 (API Credential)**

```
// 권장: 발급/폐기 행위만 기록. 실제 secret 값 절대 금지
{
  before_summary: null,   // 신규 발급
  after_summary:  { credential_id: "cred_abc", label: "CI/CD Pipeline", org_id: "org_001" }
}
// 금지: secret_key, token 값
```

### 9-3. Full Snapshot 제한 이유

1. 문서 본문이 수십 KB이면 버전마다 감사 로그 크기가 폭증
2. 감사 담당자에게 문서 전체 내용이 노출되어 권한 분리 원칙 위배
3. 장기 보존 시 개인정보 삭제권(GDPR Right to Erasure)과 충돌 가능
4. 문서 내용은 Document/Version 테이블에 이미 보존됨. 감사 로그에 중복 저장 불필요

---

## 10. 무결성 / 보존 / 접근 통제 원칙

### 10-1. 무결성 원칙

**Append-only 원칙**
- 감사 로그는 생성 후 수정/삭제 불가
- 시스템 관리자(PlatformOwner)도 삭제 불가. 물리적 삭제 경로 없음
- 잘못 기록된 로그는 별도 보정 레코드(correction record)를 추가하는 방식으로 처리
- DB 레벨에서 UPDATE/DELETE 권한 제거 (저장소 설계 시 적용)

**무결성 검증 (확장)**
- 레코드 해시 체인 또는 디지털 서명 → Phase 13 보안/운영 단계에서 도입
- MVP에서는 append-only DB 설계 + 접근 통제로 무결성 보장

### 10-2. 보존 정책 초안

| 이벤트 분류 | 최소 보존 기간 | 근거 |
|---|---|---|
| 인증/접근 이벤트 | 1년 | 보안 사고 소급 조사 |
| 문서 생성/수정/삭제 | 문서 수명 + 5년 | 문서 이력 추적 |
| 권한/역할/ACL 변경 | 5년 | 컴플라이언스 규정 |
| Permission Policy 변경 | 영구 또는 10년 | 정책 책임 추적 |
| API Credential 발급/폐기 | 5년 | 보안 감사 |
| 감사 로그 조회 이력 | 2년 | 재감사 목적 |
| 일반 운영 admin 행위 | 3년 | 운영 책임 추적 |

> 실제 보존 기간은 적용 규정(ISO 27001, 내부 정책 등)에 따라 조정. 위 값은 합리적 최소값.

### 10-3. 열람 권한 원칙

| 역할 | 열람 범위 |
|---|---|
| **SecurityAuditor** | 플랫폼 전체 감사 로그 읽기 전용 |
| **PlatformAdmin** | 플랫폼 전체 감사 로그 읽기 전용 |
| **OrganizationAdmin** | 해당 조직 범위 감사 로그만 (target_org_id 필터) |
| **OrganizationOwner** | 해당 조직 범위 감사 로그만 |
| **일반 사용자 (Editor, Viewer 등)** | 자기 자신의 행위 로그만 (actor_id = 본인) |
| **서비스 계정** | 접근 불가 |

> 조직 관리자와 플랫폼 감사자가 동일 범위를 보는 것은 적절하지 않다.
> OrganizationAdmin은 자기 조직 내 이벤트만. PlatformAdmin/SecurityAuditor는 전체.

### 10-4. 감사 로그 조회의 재감사

- 감사 로그 조회 행위 자체도 감사 로그로 기록한다 (`audit.log.accessed`)
- 기록 내용: 조회자, 조회 시각, 조회 필터 조건 (날짜 범위, event_type 필터 등)
- 이유: "누가 언제 어떤 로그를 봤는가"를 알아야 내부 정보 유출 가능성을 파악할 수 있음
- 단, 재귀적 기록(감사 로그 조회 이벤트 자체를 또 기록)은 발생하지 않도록 제어

---

## 11. Authorization Decision 연계 방식

### 11-1. Task 2-4 decision_metadata와의 연결

Task 2-4에서 Authorization Service가 반환하는 `decision_metadata`의 필드를 감사 로그에 직접 포함한다.

| Task 2-4 decision_metadata 필드 | Audit Log 필드 | 포함 여부 |
|---|---|---|
| `user_id` | `actor_id` | 포함 |
| `membership_id` | `acting_membership_id` | 포함 |
| `organization_id` | `acting_org_id` | 포함 |
| `acting_roles` | `acting_roles` | 포함 (스냅샷) |
| `resource_type` | `target_resource_type` | 포함 |
| `resource_id` | `target_resource_id` | 포함 |
| `action` | `matched_permission` | 포함 |
| `allowed` | `allowed` | 포함 |
| `reason` | `denial_reason` | 포함 |
| `matched_role` | `matched_role` | 포함 |
| `matched_acl_entry_id` | `matched_acl_entry_id` | 포함 |
| `request_id` | `request_id` | 포함 (연결 키) |
| `evaluated_at` | `occurred_at` | 포함 |

### 11-2. Authorization 경로 기록

```
authorization_path 값:
  "rbac"           — Role 기반으로 allow 판단
  "acl_direct"     — 리소스 직접 ACL로 allow 판단
  "acl_inherited"  — 상위 scope 상속 ACL로 allow 판단
  "default_deny"   — 어떤 경로도 allow 없음
  "system_rule"    — 시스템 내장 규칙 (Draft 접근 제한 등)
```

이 필드를 통해 "왜 허용/거부됐는가"를 사후에 명확히 설명할 수 있다.

### 11-3. Deny 이벤트의 특별 처리

- Authorization deny 발생 시, Service Layer는 비즈니스 로직을 실행하지 않고 즉시 감사 로그를 기록한다
- deny 이벤트는 `event_type: "auth.access.denied"` 또는 각 도메인 이벤트의 `result: "denied"`로 기록
- 존재 은닉(404 응답) 케이스에서도 내부적으로는 `result: "denied"` 기록

---

## 12. MVP 감사 로그 범위 제한안

### 12-1. MVP 필수 이벤트

| 범주 | 이벤트명 | 기록 조건 |
|---|---|---|
| **인증** | `auth.login.attempted` | 성공/실패 모두 |
| **인증** | `auth.token.issued` | 항상 |
| **인증** | `auth.token.revoked` | 항상 |
| **접근 거부** | `auth.access.denied` | 민감 리소스 대상 deny 전량 |
| **문서** | `document.created` | 성공 시 |
| **문서** | `document.updated` | 성공 시 |
| **문서** | `document.deleted` | 성공 시 |
| **문서** | `document.published` | 성공 시 |
| **문서** | `document.exported` | 성공/실패 |
| **문서** | `document.version.restored` | 성공 시 |
| **문서** | `document.attachment.uploaded` | 성공 시 |
| **문서** | `document.attachment.deleted` | 성공 시 |
| **권한** | `org.membership.added` | 항상 |
| **권한** | `org.membership.removed` | 항상 |
| **권한** | `org.membership.role.granted` | 항상 |
| **권한** | `org.membership.role.revoked` | 항상 |
| **권한** | `permission.acl.created` | 항상 |
| **권한** | `permission.acl.updated` | 항상 |
| **권한** | `permission.acl.deleted` | 항상 |
| **권한** | `permission.policy.updated` | 항상 |
| **권한** | `permission.global_role.granted` | 항상 |
| **권한** | `permission.global_role.revoked` | 항상 |
| **조직** | `org.created`, `org.updated`, `org.deleted` | 항상 |
| **운영** | `admin.api_credential.created` | 항상 |
| **운영** | `admin.api_credential.revoked` | 항상 |
| **운영** | `audit.log.accessed` | 항상 |

### 12-2. 확장 단계 이벤트

| 이벤트 | 도입 조건 |
|---|---|
| `document.read` (민감 문서) | 민감 문서 분류 기능 도입 시 |
| `document.share_link.*` | 공유 링크 기능 도입 시 |
| `document.comment.*` | 댓글 기능 도입 시 |
| `admin.integration.*`, `admin.webhook.*` | 외부 연동 기능 도입 시 |
| `workflow.approval.*` | 승인 워크플로 도입 시 |
| 모든 list/search 쿼리 감사 | 고급 컴플라이언스 요구 시 |

---

## 13. 다음 Task로 넘길 결정사항

### Task 2-6 (Activity Trace 모델 설계) 에 전달

**감사 로그가 담당하는 영역 (Task 2-6에서 제외):**
- 명시적 비즈니스 행위 이벤트 (생성/수정/삭제/권한 변경)
- 보안/컴플라이언스 목적 장기 보존 데이터
- append-only, 무결성 보장 필요 데이터

**Activity Trace가 담당할 영역 (Task 2-6에서 설계):**
- 문서 열람 흐름, 탐색 시퀀스, 검색 쿼리
- UI 상호작용 패턴 (어떤 섹션을 얼마나 읽었는가)
- 사용자 행동 분석을 위한 light-weight 이벤트
- 중기 보존 (수개월). 개인정보 고려 필요

**두 시스템이 공유할 수 있는 식별자:**

| 필드 | 감사 로그 | Activity Trace | 용도 |
|---|---|---|---|
| `request_id` | 포함 | 포함 | 단일 요청 내 두 이벤트 연결 |
| `actor_id` | 포함 | 포함 | 동일 사용자의 감사 행위와 활동 흐름 연결 |
| `acting_org_id` | 포함 | 포함 | 조직 범위 분석 |
| `target_resource_type`, `target_resource_id` | 포함 | 포함 | 특정 문서에 대한 감사 + 활동 연결 |
| `occurred_at` | 포함 | 포함 | 타임라인 재구성 |

---

## 14. 오픈 이슈

| # | 이슈 | 설명 | 결정 필요 시점 |
|---|---|---|---|
| OI-1 | IP 주소 마스킹 범위 | IP 주소는 개인정보로 분류될 수 있음. 전체 저장 vs 마스킹(끝자리 제거) vs 해시 저장 중 선택 필요 | Phase 13 보안 정책 수립 시 |
| OI-2 | 감사 로그 저장소 분리 여부 | 감사 로그를 일반 DB와 동일 저장소에 둘지, 별도 append-only 저장소(예: 전용 테이블, 외부 로그 시스템)로 분리할지 | Phase 3 인프라 설계 시 |
| OI-3 | acting_roles 스냅샷 방식 | 평가 시점의 역할 목록을 JSON 배열로 스냅샷 저장할 경우, 역할 이름 변경 시 이력과 현재 불일치 발생. role_id로 저장하면 역할 삭제 시 참조 깨짐 | Phase 3 구현 단계 |
| OI-4 | 개인정보 삭제권(GDPR) 대응 | 감사 로그는 append-only지만, GDPR 삭제 요청 시 actor_id 익명화 가능 여부 검토 필요 | Phase 13 법적 검토 |
| OI-5 | 감사 로그 자체 열람 로그의 무한 루프 방지 | `audit.log.accessed` 이벤트를 기록할 때, 이 이벤트가 다시 감사 기록 대상이 되어 무한 생성되지 않도록 방지 로직 필요 | Phase 3 구현 단계 |

---

*작성일: 2026-04-01*
*Phase: 2 / Task: 2-5*
*상태: 설계 초안 완료*
