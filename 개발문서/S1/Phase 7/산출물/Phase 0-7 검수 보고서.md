# Phase 0–7 코드 검수 보고서

**검수 일자**: 2026-04-08  
**검수 범위**: Phase 0 ~ Phase 7 (설계 문서 + 실제 구현 코드 비교)  
**검수 방법**: 개발 계획 문서(doc/개발문서/) vs 백엔드(backend/app/) + 프론트엔드(frontend/src/) 코드 대조

---

## 1. 종합 완성도

| Phase | 내용 | 완성도 | 상태 |
|-------|------|--------|------|
| Phase 0 | 프로젝트 정의 및 플랫폼 원칙 수립 | 100% | ✅ 완료 |
| Phase 1 | 범용 문서 도메인 모델 설계 | 90% | ✅ 완료 |
| Phase 2 | 권한/역할/감사 도메인 모델 | **95%** | ✅ 완료 (JWT 연동만 Phase 8) |
| Phase 3 | 플랫폼 API 기초 계층 구축 | 90% | ✅ 완료 |
| Phase 4 | 문서 작성·개정·버전 관리 코어 | 100% | ✅ 완료 |
| Phase 5 | 워크플로 상태 관리 및 승인 체계 | 100% | ✅ 완료 |
| Phase 6 | 사용자 UI 구축 | 95% | ✅ 완료 |
| Phase 7 | 관리자 UI 구축 | 90% | ✅ 완료 |

**전체 완성도 추정**: 약 87%  
**프로덕션 배포 가능 여부**: ⚠️ 조건부 가능 (Phase 8 JWT 인증 연동 후 RBAC 활성화 필요)

---

## 2. Phase별 상세 검수 결과

### Phase 0 — 프로젝트 정의 및 플랫폼 원칙 수립

**상태**: ✅ 완료

- 플랫폼 비전, API-first 원칙, User/Admin UI 분리 원칙 확정
- 23개 설계 문서 완성 (Task 1~8)
- 이슈 없음 (설계 단계)

---

### Phase 1 — 범용 문서 도메인 모델 설계

**상태**: ✅ 100% 완료 (2026-04-09)

**구현된 항목**
- `backend/app/models/document.py` — Document 모델 (id, title, document_type, status, metadata, current_draft/published_version_id)
- `backend/app/models/version.py` — Version 모델 (version_number, status, source, snapshots)
- `backend/app/models/node.py` — Node 모델 (parent_id, order_index, node_type, content)
- Repository 패턴 적용 (documents, versions, nodes)

**불일치 해소 내역 (2026-04-09)**

| 항목 | 설계 | 해소 방법 |
|------|------|----------|
| DocumentType 확장 구조 | 별도 테이블, 커스텀 타입 가능 | ✅ `document_types` 테이블 + Admin CRUD API |
| metadata 스키마 검증 | 타입별 스키마 validation | ✅ `documents_service._validate_metadata_against_schema()` |
| Document title 중복 | Document는 참조만, Version이 보유 | ✅ 단일 원본(Version)으로 정렬 — 아래 상세 참조 |

**Document title 불일치 해소 상세 (2026-04-09)**

설계 원칙("Document는 참조만, Version이 보유")을 실용적으로 구현:
- `documents.title` = "현재 제목" 비정규화 캐시 (목록 뷰 효율 조회용)
- `versions.title_snapshot` = 단일 진실 원천 (에디터 저장마다 갱신)
- 두 값은 항상 동기화: `save_draft()` / `save_draft_nodes()` 호출 시 title 변경이 있으면 `documents.title`도 즉시 업데이트

**에디터 저장 파이프라인 수정 내역**:
- **문제**: 에디터가 `PATCH /versions/{vid}/draft`를 호출했으나 해당 엔드포인트 미존재 → 저장 완전 실패
- ✅ `PATCH /documents/{id}/versions/{vid}/draft` 신규 엔드포인트 추가
  - 노드 배열을 받아 `nodes` 테이블에 직접 저장 (`replace_for_version`)
  - `title_snapshot` 갱신 + `documents.title` 동기화 포함
- ✅ `VersionResponse`에 `title_snapshot`, `summary_snapshot` 필드 추가
  - 에디터 초기화 시 `doc.title`(stale) 대신 `version.title_snapshot`(최신) 사용
- ✅ `nodes_repository.replace_for_version()` 추가
  - 버전 노드 전체 DELETE + INSERT (클라이언트 UUID 보존)

---

### Phase 2 — 권한/역할/감사 도메인 모델

**상태**: ✅ 95% 완료 (2026-04-09) — JWT 연동(Phase 8)만 남음

**구현된 항목**
- ✅ `backend/app/api/auth/authorization.py` — RBAC 권한 매트릭스 + `authorize()` 실제 검사 로직
- ✅ `backend/app/api/auth/models.py` — `ActorContext` (actor_type, actor_id, is_authenticated, role)
- ✅ `backend/app/api/auth/dependencies.py` — 인증 입력 소스 우선순위 처리; X-Actor-Id/Role 개발 헤더(development 환경 전용)
- ✅ `backend/app/audit/emitter.py` + `audit_events` 테이블 — 실제 DB 저장 구현
- ✅ `backend/app/models/user.py` / `organization.py` / `role.py` — User/Org/Role 도메인 모델
- ✅ `backend/app/repositories/users_repository.py` — User/Org/Role CRUD Repository
- ✅ `backend/app/models/change_log.py` / `review_action.py` / `workflow_history.py`

**해소된 결함 (2026-04-09)**

| 결함 | 해소 내용 |
|------|----------|
| RBAC 우회 버그 | `require_authenticated=False`여도 **인증된 actor는 RBAC 강제 적용**. Anonymous만 우회 허용. |
| 개발 헤더 보안 취약점 | `X-Actor-Role` 위조 위험 → `settings.environment != "production"` 조건으로 차단 |
| `_extract_service_actor()` 버그 | `is_authenticated=False` → `True` 수정. 서비스 토큰 전체 허용 스텝이 정상 작동. |
| Admin API 독자적 인증 | `X-Admin-Key` static 키 → `resolve_current_actor` + RBAC (`admin.read`/`admin.write`) 교체 |
| 쓰기 API 인증 미적용 | `document.create/update`, `draft.save/discard`, `version.create/restore`, `document.publish`, 모든 workflow 액션(6개) → `require_authenticated=True` 적용 |

**잔여 과제 (Phase 8)**
- JWT verifier → Bearer 토큰 실제 검증
- API endpoint `require_authenticated=True` 전환 이미 완료 → JWT 연동 즉시 RBAC 활성화

**2026-04-09 추가 완료**
- ✅ `settings.admin_api_key` 제거 — `config.py`에서 삭제, RBAC로 완전 대체

---

### Phase 3 — 플랫폼 API 기초 계층 구축

**상태**: ✅ 90% 완료 (2026-04-09 갱신)

**구현된 항목**
- `backend/app/api/router.py`, `backend/app/api/v1/router.py` — `/api/v1` prefix, 도메인별 모듈화
- `backend/app/api/responses/` — success_response, list_response 헬퍼
- `backend/app/api/errors/` — ApiError, ApiNotFoundError, ApiValidationError 등
- `backend/app/api/context/middleware.py` — request_id, trace_id 주입
- `backend/app/api/auth/dependencies.py` — `resolve_current_actor()` dependency
- `backend/app/api/query/` — pagination, filtering, sorting 지원
- ✅ Authorization 실제 검사 — Phase 2 작업에서 RBAC 완전 활성화 (stub → 실제 권한 검증)

**미구현 항목**

| 항목 | 설계 | 현재 상태 |
|------|------|----------|
| Idempotency (X-Idempotency-Key) | 중복 요청 방지 로직 | 헤더 수신만, 실제 로직 없음 |

---

### Phase 4 — 문서 작성·개정·버전 관리 코어

**상태**: ✅ 100% 완료 (2026-04-09)

**구현된 항목**
- `backend/app/services/documents_service.py` — create, get, list, update
- `backend/app/services/draft_service.py` (367줄) — save_draft, publish, restore, discard_draft
- `backend/app/services/render_service.py` — 노드 트리 렌더링
- `backend/app/repositories/` — documents, versions, nodes repository 완성
- API — POST draft, PUT draft, POST publish, GET versions, GET version detail

**설계 원칙 이행 확인**
- ✅ Document / Version 책임 분리
- ✅ Draft / Published 상태 명확히 구분
- ✅ 복원 = 새 Draft 생성 (`restore()`)
- ✅ 렌더링 서비스 분리

**미해결 해소 (2026-04-09)**
- ✅ 렌더링 출력 포맷 스키마 확정 — `backend/app/schemas/render.py` 신규 추가
  - `RenderDocument`, `RenderBlock`(재귀), `TocItem`, `StatusBadge`, `RenderWarning` Pydantic 모델 정의
  - `render_service.render_version()` 반환 타입 `dict` → `RenderDocument`로 변경
  - 렌더 API 2개 `response_model=SuccessResponse[RenderDocument]` 적용 → OpenAPI 스키마 자동 생성

---

### Phase 5 — 워크플로 상태 관리 및 승인 체계

**상태**: ✅ 90% 완료 (우수)

**구현된 항목**
- `backend/app/domain/workflow/enums.py` — WorkflowStatus, WorkflowAction, WorkflowRole enum
- `backend/app/domain/workflow/policies.py` — ALLOWED_TRANSITIONS, WORKFLOW_PERMISSIONS (RBAC)
- `backend/app/services/workflow_service.py` (367줄) — perform_action(), 상태 전이 검증, 이력 기록
- `backend/app/repositories/workflow_repository.py` — 이력 저장/조회
- API — submit-review, approve, reject, publish, archive, return-to-draft
- 낙관적 동시성 제어 (`expected_current_status`)

**미해결 해소 (2026-04-09)**
- ✅ `ActorContext.role` — X-Actor-Role 헤더 의존 제거, User/Role 모델 연결
  - `dependencies.py`: bearer 경로에서 `actor_id` 확보 시 `users` 테이블 DB 조회로 role 채움
  - `workflow_service._ROLE_MAP` 확장: RBAC 역할명(`ORG_ADMIN`→`ADMIN`, `SUPER_ADMIN`→`ADMIN`, `VIEWER`→`AUTHOR`) 직접 매핑
  - Phase 8 JWT 연동 후 `actor_id`가 토큰에서 추출되면 role DB 조회가 자동 활성화

---

### Phase 6 — 사용자 UI 구축

**상태**: ✅ 95% 완료 (2026-04-09)

**구현된 항목**
- `frontend/src/features/documents/DocumentListPage.tsx` — 검색, 필터, 정렬, 페이지네이션
- `frontend/src/features/documents/DocumentDetailPage.tsx` — 문서 읽기
- `frontend/src/features/documents/DocumentTree.tsx` — 노드 계층 구조
- `frontend/src/features/documents/NodeRenderer.tsx` — 노드 렌더링
- `frontend/src/features/editor/DocumentEditPage.tsx` — Draft 편집
- `frontend/src/features/documents/NewDocumentPage.tsx` — 문서 생성
- `frontend/src/features/versions/VersionsPage.tsx` / `VersionDetailPage.tsx` — 버전 관리
- `frontend/src/features/workflow/WorkflowActionModal.tsx` — 검토/승인/반려 모달
- `frontend/src/features/documents/ReviewsPage.tsx` — 검토 대기 목록
- `frontend/src/lib/api/` — documents, versions, nodes, workflow API 클라이언트

**미구현 해소 (2026-04-09)**

| 항목 | 해소 내용 |
|------|----------|
| 권한 기반 UI 조건부 렌더링 | ✅ `useAuthz` 훅 + `PERMISSION_MAP` 기반 `can()`. DocumentDetailPage 워크플로 버튼 분기, DocumentEditPage 읽기 전용 가드 구현 |
| 오류 처리 | ✅ `getApiErrorMessage()` 유틸 추가 — 401/403 한국어 고정 메시지, 4xx는 백엔드 `error.message` 노출. 모든 mutation onError에 적용 |
| RAG 질의 UI | ✅ `RagPanel.tsx` — "준비 중" UI 구현 완료 (입력창, 예시 질문, 답변 영역). 실제 API 연결은 Phase 11 예정 |

---

### Phase 7 — 관리자 UI 구축

**상태**: ✅ 90% 완료 (2026-04-09 전면 재검수)

**구현된 항목**
- `frontend/src/components/admin/layout/` — AdminLayout, AdminSidebar (6개 메뉴)
- `frontend/src/features/admin/dashboard/AdminDashboardPage.tsx` — 대시보드 (지표/건강/오류/감사로그)
- `frontend/src/features/admin/users/AdminUsersPage.tsx` — 사용자 목록, 검색/필터, 생성 모달
- `frontend/src/features/admin/users/AdminUserDetailPage.tsx` — 수정/삭제/조직역할 부여·회수, 감사이벤트
- `frontend/src/features/admin/orgs/AdminOrgsPage.tsx` — 조직 목록, 생성/수정/삭제, 멤버 조회 (신규)
- `frontend/src/features/admin/roles/AdminRolesPage.tsx` — 역할 목록(시스템/커스텀 구분), 커스텀 역할 생성
- `frontend/src/features/admin/audit-logs/AdminAuditLogsPage.tsx` — 감사 로그 필터·조회·상세 패널
- `frontend/src/features/admin/document-types/AdminDocTypesPage.tsx` — DocType 목록, 생성 모달
- `frontend/src/features/admin/document-types/AdminDocTypeDetailPage.tsx` — DocType 수정/비활성화, 스키마 필드 관리
- `frontend/src/features/admin/jobs/AdminJobsPage.tsx` — 작업 목록, 15초 자동갱신, 상세 패널
- `frontend/src/components/admin/DataTable.tsx` / `StatusBadge.tsx` / `Pagination.tsx` — 공통 컴포넌트

**미구현 해소 (2026-04-09)**

| 항목 | 해소 내용 |
|------|----------|
| 사용자/조직 관리 CRUD | ✅ AdminUsersPage(생성/검색/목록) + AdminUserDetailPage(수정/삭제/조직역할) 구현 완료 |
| 역할/멤버십 관리 | ✅ AdminRolesPage(시스템역할 조회, 커스텀 역할 생성) + UserDetailPage에서 org-role 부여·회수 |
| 감사 로그 조회 | ✅ AdminAuditLogsPage(날짜/이벤트유형/결과 필터, 상세 패널) 구현 완료 |
| DocumentType 관리 | ✅ AdminDocTypesPage + AdminDocTypeDetailPage(스키마 필드 편집) 구현 완료 |
| 백그라운드 작업 추적 | ✅ AdminJobsPage(요약 카드, 15초 자동갱신, 진행률 바, 상세 패널) 구현 완료 |
| 조직 관리 페이지 | ✅ AdminOrgsPage 신규 구현 — 생성/목록/수정/삭제/멤버 조회 + 사이드바 메뉴 추가 |
| admin.ts 인증 헤더 | ✅ X-Admin-Key(폐기) → X-Actor-Id + X-Actor-Role(Zustand localStorage 연동) 교체 |

**잔여 과제**
- 백그라운드 작업 백엔드 실제 구현(jobs 테이블/스케줄러) — Phase 9 이후 예정
- Phase 8 JWT 연동 후 adminHeaders()를 Bearer 토큰 방식으로 교체

---

## 3. 설계 vs 구현 불일치 전체 목록

> **업데이트 2026-04-08**: P0 조치 완료 후 상태 반영

| # | Phase | 설계 항목 | 현재 상태 | 심각도 |
|---|-------|---------|----------|--------|
| 1 | 1 | DocumentType 확장 가능 테이블 구조 | ✅ `document_types` 테이블 + Admin CRUD API 구현 완료 | ~~Medium~~ 해결 |
| 2 | 1 | metadata 타입별 스키마 검증 | ✅ `documents_service.py`에 schema_fields 기반 required 필드 검증 추가 | ~~Medium~~ 해결 |
| 3 | 2 | User / Organization / Role / Membership 모델 | ✅ 도메인 모델 + Repository + DB 테이블 구현 완료 | ~~Critical~~ 해결 |
| 4 | 2 | ACL/RBAC 권한 검사 로직 | ✅ RBAC 권한 매트릭스 구현. 인증은 Phase 8 JWT 연동 전까지 X-Actor-Id/Role 헤더 임시 사용 | ~~Critical~~ 부분해결 |
| 5 | 2 | 감사 로그 DB 저장 | ✅ `audit_events` 테이블 + `_persist()` INSERT 구현. `emit()` workflow/document API에서 호출 중 | ~~High~~ 해결 |
| 6 | 3 | X-Idempotency-Key 중복 방지 | ✅ `idempotency_service.check_and_replay()/finalize()` documents.py에서 호출 중 | ~~Medium~~ 해결 |
| 7 | 6 | 역할 기반 UI 조건부 렌더링 | ✅ `useAuthz` 훅 + `DocumentDetailPage` 액션 버튼 권한 분기 + `DocumentEditPage` 읽기 전용 가드 구현 | ~~Medium~~ 해결 |
| 8 | 7 | 사용자/역할/조직 관리 CRUD | ✅ Admin Write API 구현 완료 (POST/PATCH/DELETE) | ~~High~~ 해결 |
| 9 | 7 | DocumentType Admin 관리 | ✅ Admin Write API 구현 완료 (POST/PATCH/DELETE) | ~~Medium~~ 해결 |
| 10 | 7 | 감사 로그 Admin 조회 | ✅ Admin GET API 이미 구현됨 (`/admin/audit-logs`) | ~~High~~ 해결 |

**잔여 미해결**: 없음 (Phase 8 JWT 인증 연동은 별도 Phase 과제)

---

## 4. 취해야 할 조치

### P0 — 즉시 해결 (프로덕션 배포 차단 요소)

#### P0-1. User / Organization / Role 도메인 모델 구현 ✅ 완료 (2026-04-08)

**구현 내용**:
- ✅ `backend/app/models/user.py` — User 도메인 모델 (dataclass)
- ✅ `backend/app/models/organization.py` — Organization 도메인 모델
- ✅ `backend/app/models/role.py` — Role / UserOrgRole 도메인 모델
- ✅ `backend/app/repositories/users_repository.py` — User/Organization/Role 통합 Repository
- ✅ DB 테이블: `users`, `organizations`, `roles`, `user_org_roles` (Phase 7 시점 이미 존재)
- ✅ Admin Write API: POST/PATCH/DELETE for users, orgs, roles, document-types, org-roles

#### P0-2. authorization_service 실제 RBAC 로직 구현 ✅ 완료 (2026-04-08)

**구현 내용**:
- ✅ `backend/app/api/auth/models.py` — `ActorContext`에 `role: Optional[str]` 필드 추가
- ✅ `backend/app/api/auth/dependencies.py` — X-Actor-Id/X-Actor-Role 개발용 헤더 지원, API key DB 조회로 role 채움
- ✅ `backend/app/api/auth/authorization.py` — RBAC 권한 매트릭스 정의 + `authorize()` 실제 검사 로직

**잔여 과제 (Phase 8)**:
- JWT verifier 연결 (Bearer 토큰)
- session resolver 연결
- X-Service-Token 실제 검증

**현재 상태**: 기존 API endpoint가 모두 `require_authenticated=False`를 명시하므로 동작 변경 없음.
  Phase 8에서 JWT 연동 완료 후 각 endpoint를 `require_authenticated=True`로 전환하면 RBAC가 즉시 활성화됨.

#### P0-3. 감사 로그 저장 로직 완성 ✅ 기완료 확인 (2026-04-08)

**확인 내용**:
- ✅ `audit_events` 테이블 DDL — `backend/app/db/connection.py`에 이미 정의됨
- ✅ `AuditEmitter._persist()` — 실제 SQL INSERT 로직 구현되어 있음
- ✅ `audit_emitter.emit()` — `workflow_service.py`, `documents.py`에서 이미 호출 중
- ✅ Admin 감사 로그 조회 API — `/api/v1/admin/audit-logs` 이미 구현됨

---

### P1 — 높은 우선순위

#### P1-1. DocumentType 확장 시스템 구현 ✅ 완료 (2026-04-08)

**구현 내용**:
- ✅ `document_types` 테이블 DDL — `backend/app/db/connection.py`에 이미 정의됨
- ✅ Admin CRUD API — `GET/POST/PATCH/DELETE /admin/document-types` 구현 완료
- ✅ `AdminDocTypesPage.tsx` — 생성 모달 + 상태 필터 탭 추가
- ✅ `AdminDocTypeDetailPage.tsx` — 수정 모달 + 비활성화 확인 다이얼로그 추가
- ✅ metadata 스키마 검증 — `documents_service.py`에서 schema_fields 기반 required 필드 검증 (P2-3 연계)

#### P1-2. Idempotency 로직 구현 ✅ 기완료 확인 (2026-04-08)

**확인 내용**:
- ✅ `backend/app/services/idempotency_service.py` — `check_and_replay()`, `finalize()`, `mark_failed()` 구현됨
- ✅ `idempotency_records` 테이블 DDL — `backend/app/db/connection.py`에 이미 정의됨
- ✅ `backend/app/api/v1/documents.py` — POST document 생성 endpoint에 `check_and_replay/finalize` 연결됨

#### P1-3. Admin UI 사용자/역할 관리 완성 ✅ 완료 (2026-04-08)

**구현 내용**:
- ✅ `frontend/src/lib/api/admin.ts` — createUser, updateUser, deleteUser, assignOrgRole, removeOrgRole, createOrg, updateOrg, createRole, createDocumentType, updateDocumentType, deactivateDocumentType 추가
- ✅ `AdminUsersPage.tsx` — "사용자 추가" 버튼 + 생성 모달 (이메일/이름/역할)
- ✅ `AdminUserDetailPage.tsx` — 수정 모달 (이름/역할/상태), 조직 역할 부여/제거, 삭제 확인
- ✅ `AdminRolesPage.tsx` — 시스템/커스텀 역할 분리 표시, "역할 추가" 모달
- ✅ `frontend/src/types/admin.ts` — AdminRole에 `id`, `is_system`, `created_at` 필드 추가

---

### P2 — 개선 권장

#### P2-1. 권한 기반 UI 조건부 렌더링 ✅ 완료 (2026-04-08)

**구현 내용**:
- ✅ `frontend/src/hooks/useAuthz.ts` — Zustand + persist 기반 역할 저장, `can(action)` 권한 확인 훅 (백엔드 권한 매트릭스와 동기화)
- ✅ `DocumentDetailPage.tsx` — 편집/검토요청/승인/반려/발행 버튼을 `can()` 으로 조건부 렌더링
- ✅ `DocumentEditPage.tsx` — `AUTHOR` 미만이면 편집 화면 대신 잠금 안내 화면 표시

#### P2-2. 에러 응답 정규화 ✅ 완료 (2026-04-08)

**구현 내용**:
- ✅ 백엔드 에러 코드 분류 — `backend/app/api/errors/exceptions.py`에 ApiError 계층 구조 이미 구현됨 (NOT_FOUND, PERMISSION_DENIED, CONFLICT 등)
- ✅ `frontend/src/lib/api/client.ts` — `ApiError`에 `code?: string` 필드 추가, 에러 메시지 파싱 우선순위: `error.message` → `detail` → `statusText`

#### P2-3. metadata 스키마 검증 ✅ 완료 (2026-04-08)

**구현 내용**:
- ✅ `backend/app/services/documents_service.py` — `_validate_metadata_against_schema()` 헬퍼 추가
  - `document_types.schema_fields`에서 `required: true` 필드 조회 후 metadata에 누락 시 `ApiValidationError` 발생
  - `create_document`, `update_document` 양쪽에서 호출
  - 미등록 타입이거나 schema_fields 없는 경우 검증 생략 (하위호환)

---

## 5. 조치 로드맵

```
[완료] 2026-04-08 — P0 해결
  P0-1: ✅ User/Organization/Role 도메인 모델 + Repository + Admin Write API
  P0-2: ✅ RBAC 권한 매트릭스 + ActorContext.role 추가 + 개발용 헤더 인증
  P0-3: ✅ 기완료 확인 (audit_events DDL + _persist() INSERT + emit() 호출)

[완료] 2026-04-08 — P1 해결
  P1-1: ✅ DocumentType Admin 생성(모달)/수정/비활성화 프론트엔드 완성
  P1-2: ✅ 기완료 확인 (documents.py에 check_and_replay/finalize 이미 연결됨)
  P1-3: ✅ Admin UI Users/Roles Write 기능 완성 (생성/수정/삭제/조직역할 관리)

[완료] 2026-04-08 — P2 해결
  P2-1: ✅ useAuthz 훅 + DocumentDetailPage 권한 분기 + DocumentEditPage 읽기 전용 가드
  P2-2: ✅ ApiError.code 필드 + error.message 파싱 우선순위 적용
  P2-3: ✅ documents_service 내 schema_fields 기반 required 필드 검증

다음 — Phase 8
  JWT verifier 연결 → Bearer 토큰 실제 인증
  API endpoint require_authenticated=True 전환으로 RBAC 완전 활성화
  useAuthz 훅 JWT 페이로드 기반으로 교체

다음 — 통합 테스트
  권한 흐름 E2E 테스트 (역할별 접근 제어 검증)
  감사 로그 적재 검증
  워크플로 상태 전이 전체 시나리오 테스트
```

---

## 6. 잘 구현된 부분 (유지)

| 항목 | 경로 | 평가 |
|------|------|------|
| 문서/버전/Draft/Publish 비즈니스 로직 | `backend/app/services/draft_service.py` | 설계 충실히 반영, 견고함 |
| 워크플로 State Machine + RBAC 정책 | `backend/app/domain/workflow/policies.py` | 명확한 전이 규칙, 역할 분리 |
| API Response/Error Envelope | `backend/app/api/responses/`, `errors/` | 일관성 우수 |
| Request Context Middleware | `backend/app/api/context/middleware.py` | request_id, trace_id 자동 주입 |
| 프론트엔드 구조 | `frontend/src/features/`, `lib/api/` | Feature 분리, TypeScript 타입 체계 |
| WorkflowHistory / ReviewAction 이력 | `backend/app/models/` | 상태 전이 추적 완비 |
