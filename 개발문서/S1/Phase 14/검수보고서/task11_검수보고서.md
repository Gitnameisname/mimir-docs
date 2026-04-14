# Task 14-11 검수보고서: 시스템 설정 관리 API 및 UI

**작성일**: 2026-04-14
**검수 대상**: `system_settings` 테이블, Settings API 3종, AdminSettingsPage UI

---

## 1. 구현 범위

### 1.1 백엔드 — DDL + 시드
- `app/db/connection.py`:
  - `_SYSTEM_SETTINGS_DDL` — `id/category/key/value(JSONB)/description/updated_by/updated_at`, `UNIQUE(category, key)`, `idx_system_settings_category` 인덱스
  - `_SYSTEM_SETTINGS_SEED_DDL` — 12개 초기 설정 (`auth` 5, `system` 3, `notification` 2, `security` 2). `ON CONFLICT (category, key) DO NOTHING`로 멱등
  - `init_db()`에서 두 DDL 순차 실행

### 1.2 백엔드 — Repository
- `app/repositories/settings_repository.py` 신규
- `SettingsRepository`: `list_all`, `list_by_category`, `get_one`, `list_categories`, `update_value`
- 모든 SQL은 `%s` 파라미터 바인딩 사용. JSONB는 `json.dumps(...)::jsonb`로 직렬화 (타입 보존)
- `update_value`는 `RETURNING` 절로 갱신된 row 반환 — 별도 SELECT 없음

### 1.3 백엔드 — API 엔드포인트 (admin.py 末尾)
- `GET /api/v1/admin/settings` — 카테고리 그룹 응답 `{ categories: [{name, label, items: [{key, value, description, updated_at}]}] }`. Valkey 캐시 5분 TTL.
- `GET /api/v1/admin/settings/{category}` — 단일 카테고리 조회. 카테고리 형식 regex 검증 (`^[a-z][a-z0-9_]{0,99}$`).
- `PATCH /api/v1/admin/settings/{category}/{key}` — 값 변경. 키/카테고리 정규식 검증, 존재 확인 (404), 타입 일치 검증 (422), 변경 없음 시 no-op, 캐시 무효화, 감사 이벤트 (`SETTING_CHANGED`).
- `_type_signature` 함수: `bool`을 `int`보다 먼저 분기하여 boolean이 int로 오인되지 않게 처리.

### 1.4 보안 / 캐시
- 모든 settings 엔드포인트에 `Depends(require_admin_access)` — `GET=admin.read(ORG_ADMIN+)`, `PATCH=admin.write(SUPER_ADMIN)`. 별도 SUPER_ADMIN 가드 추가 불필요.
- 감사 이벤트 metadata에 `old_value` / `new_value` 포함. `previous_state` / `new_state`는 500자 truncate (DoS 방어).
- 캐시 키 `system_settings:all:v1`, TTL 300초. PATCH 시 `_invalidate_settings_cache()`로 무효화. Valkey 장애 시 DB 폴백.

### 1.5 프론트엔드 — 타입 / API 클라이언트
- `types/admin.ts`: `SettingValue`, `SettingItem`, `SettingCategory`, `AllSettingsResponse`
- `lib/api/admin.ts`: `getAllSettings`, `getSettingsByCategory`, `updateSetting`. URL path는 `encodeURIComponent`로 escape.

### 1.6 AdminSettingsPage 컴포넌트
- `useQuery` 로 설정 fetch, `useAuth().hasRole("SUPER_ADMIN")`로 편집 권한 판정 (`canEdit`)
- 카테고리 탭 (`role="tablist"/tab/tabpanel"`, `aria-selected`, `aria-controls`)
- 타입별 입력: boolean → 토글(`role="switch"`, `aria-checked`), number → numeric input, string → text input
- `drafts` state에 변경 항목 누적 (`{[category.key]: value}`), `dirtyChanges` 계산
- `ConfirmModal` — 변경 항목 목록 (이전→새 값 시각화)
- `useMutationWithToast` — 직렬 PATCH 호출, 부분 실패 보고
- 저장 후 `queryClient.invalidateQueries`로 즉시 갱신
- UI 측 타입 검증 (mismatch 시 `aria-invalid` + 에러 메시지)

### 1.7 라우팅
- `app/admin/settings/page.tsx` — 기존 placeholder를 `<AdminSettingsPage />` 호출로 교체
- 사이드바에는 Phase 14-9에서 이미 `/admin/settings → 시스템 설정` 항목 등록되어 있음 (변경 없음)

---

## 2. 검수 결과

| 카테고리 | 검증 항목 수 | PASS | FAIL |
|---------|:-----------:|:----:|:----:|
| DDL/시드 (DDL) | 7 | 7 | 0 |
| Repository (REPO) | 8 | 8 | 0 |
| API (API) | 11 | 11 | 0 |
| 보안 (SEC) | 7 | 7 | 0 |
| 캐시 (CACHE) | 5 | 5 | 0 |
| 타입 (TY) | 4 | 4 | 0 |
| API 클라이언트 (API-CL) | 4 | 4 | 0 |
| UI (UI) | 16 | 16 | 0 |
| 반응형 (RESP) | 5 | 5 | 0 |
| 접근성 (A11Y) | 8 | 8 | 0 |
| 라우팅 (ROUTE) | 3 | 3 | 0 |
| XSS / 시크릿 | 6 | 6 | 0 |
| **합계** | **84** | **84** | **0** |

검수 스크립트: `backend/tests/unit/test_admin_settings_phase14_11.py`

---

## 3. UI 디자인 리뷰 (5회)

| 라운드 | 초점 | 개선 내용 |
|:------:|------|---------|
| 1 | 정보 위계 | 카테고리 탭 → 표 → 행 단위 (label+key 모노스페이스 보조). dirty 항목은 `bg-amber-50/40`로 즉시 식별. |
| 2 | 접근성 | tablist/tab/tabpanel + `aria-selected/controls`, 토글 `role="switch"` + `aria-checked`, 입력 `aria-invalid` + `aria-describedby` 에러 연결, `htmlFor` 라벨 |
| 3 | 입력 UX | 타입별 위젯 (boolean→토글, number→numeric, string→text), 변경 카운트 배지 (탭별/전체), 되돌리기 버튼, 저장 시 확인 다이얼로그(이전→새 값) |
| 4 | 반응형 / 마감 | `p-4 sm:p-6`, `text-xl sm:text-2xl`, 헤더 `flex-wrap`, 표 `overflow-x-auto`, 버튼 `min-h-[40px]` + `active:scale-[0.98]`, sticky thead, dirty 행 강조 |
| 5 | 권한 / 안전 | `canEdit` false 시 모든 입력 disabled + 헤더에 "읽기 전용" 안내, 저장 중 모달 닫기 차단, 부분 실패 시 성공/실패 카운트 토스트 |

---

## 4. 생성/수정 파일 목록

### 백엔드
- `backend/app/db/connection.py` — `_SYSTEM_SETTINGS_DDL`, `_SYSTEM_SETTINGS_SEED_DDL` 추가, `init_db()` 호출 추가
- `backend/app/repositories/settings_repository.py` — 신규
- `backend/app/api/v1/admin.py` — settings API 3종 + 캐시/감사 헬퍼

### 프론트엔드
- `frontend/src/types/admin.ts` — Setting 관련 타입
- `frontend/src/lib/api/admin.ts` — getAllSettings/getSettingsByCategory/updateSetting
- `frontend/src/features/admin/settings/AdminSettingsPage.tsx` — 신규 페이지 (~430 line)
- `frontend/src/app/admin/settings/page.tsx` — placeholder → 실제 컴포넌트 호출

### 검수
- `backend/tests/unit/test_admin_settings_phase14_11.py` — 84개 항목 검수 스크립트

---

## 5. 프로젝트 규칙 준수

- ✅ 문서 타입 하드코딩 금지 → 해당 없음 (시스템 설정은 문서 타입 무관)
- ✅ generic + config 기반 → 카테고리/키/값은 모두 DB row, 코드에는 한국어 라벨 매핑만
- ✅ JSONB schema 관리 → `value`는 JSONB지만 백엔드가 기존 row의 타입과 일치 여부 검증 (`_type_signature`)
- ✅ 타입-aware 로직 → bool/int/float/str/list/dict 구분하여 검증
- ✅ deprecated 기능 미사용 (psycopg2 + redis-py 정상)
- ✅ UI 리뷰 5회 수행
- ✅ 데스크탑/모바일 반응형
- ✅ 검수 + 보안 보고서 작성

---

## 6. 결론

84/84 검증 항목 전부 PASS. JSONB 기반 generic 설정 저장 + 백엔드 단계의 strict 타입 검증 + Valkey 5분 캐시 + 감사 로깅 + SUPER_ADMIN 전용 변경의 다층 통제를 갖춘 시스템 설정 관리 시스템이 완성됨. UI는 타입별 적합한 입력, dirty 추적, 변경 확인 다이얼로그, 부분 실패 보고를 갖춤. 향후 `maintenance_mode` 미들웨어 / 환경 변수 우선 순위는 다음 작업 범위.
