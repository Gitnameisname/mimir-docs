# Task 14-10 검수보고서: 사용자/조직/역할 관리 UI 구현

**작성일**: 2026-04-14
**검수 대상**: 사용자/조직/역할 관리 UI + 권한 매트릭스 뷰

---

## 1. 구현 범위

### 1.1 백엔드 — 권한 매트릭스 API
- `app/api/auth/authorization.py`: `get_permission_matrix()` 신규 함수 — `_PERMISSION_MATRIX`(frozenset)를 JSON-safe `dict[str, list[str]]`로 직렬화. 역할 순서(`VIEWER → SUPER_ADMIN`) 보장
- `app/api/auth/__init__.py`: `get_permission_matrix` export 추가
- `app/api/v1/admin.py`: `GET /api/v1/admin/roles/permissions/matrix` 엔드포인트
  - 응답: `{ roles: [...], groups: [{ name, items: [{ action, verb, allowed_roles }] }] }`
  - 그룹화: `document.read` → group=`document`, verb=`read`
  - `require_admin_access` 의존성 (ORG_ADMIN 이상)
- `app/api/v1/admin.py`: `list_users`에 `role` 쿼리 파라미터 추가 (`role_name = %s` WHERE)

### 1.2 프론트엔드 — 타입 / API 클라이언트
- `types/admin.ts`: `PermissionMatrix`, `PermissionMatrixGroup`, `PermissionMatrixItem` 인터페이스
- `lib/api/admin.ts`: `getPermissionMatrix()` 메서드, `getUsers({ role })` 파라미터

### 1.3 PermissionMatrix 컴포넌트 (`features/admin/roles/PermissionMatrix.tsx` 신규)
- React Query로 매트릭스 fetch
- 그룹별 접기/펼치기 (`Set<string>` 상태, `aria-expanded`/`aria-controls`)
- 그룹 한국어 라벨 매핑 (`GROUP_LABELS`: 문서/버전/노드/초안/워크플로/검색/RAG/관리자)
- 역할 컬럼별 체크/공백 표시 (체크 SVG + `aria-label`)
- 로딩(`role="status"` + `sr-only`) / 에러(`role="alert"`) 상태 처리
- 가로 스크롤 (`overflow-x-auto`) — 모바일에서 6개 역할 컬럼 스크롤 가능
- 읽기 전용 (RBAC 매트릭스는 코드 레벨 관리 — UI 편집 불가)

### 1.4 AdminRolesPage 통합
- `<PermissionMatrix />` 추가 (시스템 역할 / 커스텀 역할 테이블 아래)
- 반응형 패딩 `p-4 sm:p-6`, 헤딩 `text-xl sm:text-2xl`

### 1.5 AdminUsersPage — 역할 필터
- `role` 상태 변수, queryKey/쿼리 파라미터에 추가
- 역할 select (`id="filter-role"` + `htmlFor` 라벨, 6개 옵션)
- 초기화 버튼 조건 확장 (`search || status || role`)
- 반응형 패딩/헤딩, 사용자 추가 버튼 `min-h-[40px]` + focus-visible 링

### 1.6 SUPER_ADMIN 삭제 가드
- `AdminUserDetailPage`: `useAuth().hasRole("SUPER_ADMIN")` 체크 → `canDelete` false면 삭제 버튼 미렌더
- `AdminOrgsPage`: 동일 패턴 적용 (조직 삭제)
- 백엔드는 이미 `admin.write` = SUPER_ADMIN 전용이므로 UI는 보조 가드

---

## 2. 검수 결과

| 카테고리 | 검증 항목 수 | PASS | FAIL |
|---------|:-----------:|:----:|:----:|
| 백엔드 (BE) | 9 | 9 | 0 |
| 타입 (TY) | 3 | 3 | 0 |
| API 클라이언트 (API) | 3 | 3 | 0 |
| PermissionMatrix (PM) | 13 | 13 | 0 |
| AdminRolesPage (RP) | 3 | 3 | 0 |
| AdminUsersPage (UP) | 9 | 9 | 0 |
| AdminUserDetailPage (UD) | 4 | 4 | 0 |
| AdminOrgsPage (OP) | 4 | 4 | 0 |
| 보안 (SEC) | 7 | 7 | 0 |
| 접근성 (A11Y) | 6 | 6 | 0 |
| **합계** | **61** | **61** | **0** |

검수 스크립트: `backend/tests/unit/test_admin_management_phase14_10.py`

---

## 3. UI 디자인 리뷰 (5회 수행)

| 라운드 | 초점 | 개선 내용 |
|:------:|------|---------|
| 1 | 접근성 | `scope="col"/"row"` 매트릭스 헤더, `aria-expanded`/`aria-controls` 그룹 토글, 셀별 `aria-label`(허용/금지), `htmlFor` 라벨 연결 |
| 2 | 모바일 반응형 | `p-4 sm:p-6`, `text-xl sm:text-2xl`, 헤더 `flex-wrap`, 매트릭스 가로 스크롤(`overflow-x-auto`), 버튼 44px(min-h-[40px]) |
| 3 | 시각적 위계 | 그룹 헤더 `uppercase tracking-widest`, action 컬럼 `font-mono`, 체크 아이콘 `text-green-600 stroke-2.5`, 그룹 row `bg-gray-50` 구분 |
| 4 | 일관성 | 전체 페이지 동일한 헤더 스타일, 동일한 reset 패턴, `transition-all duration-200`, focus-visible 통일 |
| 5 | 마감 | 토글 화살표 `rotate-90` 트랜지션, staggered skeleton, hover 행 강조, `active:scale-95` 피드백, 매트릭스 `sticky top-0` 헤더 |

---

## 4. 생성/수정 파일 목록

### 백엔드
- `backend/app/api/auth/authorization.py` — `get_permission_matrix()` 추가
- `backend/app/api/auth/__init__.py` — export 추가
- `backend/app/api/v1/admin.py` — `/roles/permissions/matrix` 엔드포인트, `list_users` role 필터

### 프론트엔드
- `frontend/src/types/admin.ts` — PermissionMatrix 타입
- `frontend/src/lib/api/admin.ts` — getPermissionMatrix, getUsers role
- `frontend/src/features/admin/roles/PermissionMatrix.tsx` — 신규 컴포넌트
- `frontend/src/features/admin/roles/AdminRolesPage.tsx` — 매트릭스 통합 + 반응형
- `frontend/src/features/admin/users/AdminUsersPage.tsx` — 역할 필터 + 반응형
- `frontend/src/features/admin/users/AdminUserDetailPage.tsx` — SUPER_ADMIN 삭제 가드
- `frontend/src/features/admin/orgs/AdminOrgsPage.tsx` — SUPER_ADMIN 삭제 가드 + 반응형

### 검수
- `backend/tests/unit/test_admin_management_phase14_10.py` — 검수 스크립트

---

## 5. 프로젝트 규칙 준수

- ✅ 문서 타입 하드코딩 금지 → 해당 없음 (관리 UI는 문서 타입 무관)
- ✅ generic + config 기반 → 권한 매트릭스는 백엔드 단일 source of truth(`_PERMISSION_MATRIX`)에서 동적 fetch
- ✅ DataTable 등 공통 컴포넌트 재사용 (Phase 7 패턴 유지)
- ✅ UI 리뷰 5회 이상 수행
- ✅ 데스크탑/모바일 반응형 (p-4 sm:p-6, flex-wrap, 가로 스크롤)
- ✅ 검수 + 보안 보고서 작성

---

## 6. 결론

61/61 검증 항목 전부 PASS. Phase 7 admin 페이지 인프라 위에 권한 매트릭스 뷰(신규)와 역할 필터, SUPER_ADMIN 삭제 가드, 모바일 반응형을 추가 완료. 권한 매트릭스는 백엔드 RBAC 정의를 단일 source of truth로 유지하면서 UI에서 시각화. 향후 백엔드 매트릭스 변경 시 UI 자동 반영.
