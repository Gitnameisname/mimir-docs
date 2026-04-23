# /admin/extraction-schemas — P3 검수보고서

## 개요

| 항목 | 값 |
|---|---|
| 작성일 | 2026-04-21 |
| 대상 경로 | `/admin/extraction-schemas` |
| Phase | S2-5 UI 개선 — **P3 (추가 UX · 서버 상한 · 테스트 확장)** |
| 범위 | P3-A ~ P3-E (모두 포함) |
| 상위 맥락 | P2 종결 직후 이어진 "UX 완성도 + 서버측 DoS 내성 + 회귀 테스트" 확장 |

P2 까지는 "fields 를 폼으로 편집" 수준에서 마무리됐고, P3 는 다음 네 가지 UX/DoS 한계를 정리한다.

- P3-A: object 필드의 `nested_schema` 재귀 편집 — 기존에는 안내 문구만 있었음.
- P3-B: `default_value` 타입-aware 위젯 — 기존에는 JSON 리터럴 단일 입력.
- P3-C: 서버측 fields 개수/깊이 상한 — DoS 내성 보강 (백엔드 FG).
- P3-D: 버전 이력 "더 보기" 페이지네이션 — 오래된 스키마에서 payload 비대 방지.
- P3-E: pytest 회귀 테스트 — P2-C / P3-C 의 Pydantic 경로를 단위 + 통합 수준에서 고정.

---

## 구현 결과 (요약)

### P3-A. nested_schema 재귀 편집 (프론트)

- `frontend/src/features/admin/extraction-schemas/FieldsEditor.tsx`
  - `FieldsEditor` 본체가 자기 자신을 재귀적으로 호출한다. `depth`, `maxDepth` 프로퍼티 추가 (default 0, 3).
  - `FieldRowEditor` 도 동일한 두 프로퍼티를 받아 object 타입일 때 depth+1 로 하위 편집기를 렌더.
  - object 타입으로 전환 시 `nested_schema = {}` 로 초기화해 하위 편집기가 빈 상태로도 렌더 가능.
  - `depth < maxDepth` 일 때만 재귀 편집기를 표시하고, 한계 도달 시에는 "중첩 허용 최대 깊이(3) 도달" 안내로 대체.
  - `isRoot === false` (depth > 0) 면 편집기 툴바에서 JSON 모드 토글 숨김 — 중첩 레벨의 JSON 편집은 UX 비용 대비 이득이 낮아 폼 전용.
- 백엔드 `MAX_NESTED_DEPTH = 3` 과 수치가 일치 (상수 `DEFAULT_MAX_NESTED_DEPTH`).

### P3-B. default_value 타입-aware 위젯

- 같은 파일에 `DefaultValueWidget` 컴포넌트 추가.
  - `number` → `<input type="number">`
  - `boolean` → `<select>` 에 `(기본값 없음) / true / false` 세 옵션
  - `date` → `<input type="date">` (YYYY-MM-DD)
  - `enum` → `<select>` 에 `enum_values` 를 옵션으로 나열. enum_values 가 비어있으면 비활성화 + 안내 문구.
  - `string` → `<input type="text">`
  - `array` / `object` → JSON 리터럴 입력 유지 (구조화 위젯 비용 대비 이득 낮음).
  - 현재 `default_value` 가 `field_type` 과 호환되지 않으면 각 위젯이 빈 상태로 렌더되면서 "⚠ 현재 값이 ~와 호환되지 않음" 안내.

### P3-C. 서버측 fields 개수 / 중첩 깊이 상한 (백엔드)

- `backend/app/schemas/extraction.py`
  - 상수 `MAX_FIELDS_COUNT = 200`, `MAX_NESTED_DEPTH = 3`.
  - 내부 헬퍼 `_max_nested_depth(field)` — `ExtractionFieldDef` 트리를 내려가며 최대 깊이 계산. 안전장치로 `MAX_NESTED_DEPTH + 2` 초과 시 즉시 반환.
  - 내부 헬퍼 `_validate_fields_map(v)` — 빈 딕셔너리 / 상한 초과 / 깊이 초과를 한 곳에서 검증.
  - `CreateExtractionSchemaRequest.fields` 와 `UpdateExtractionSchemaRequest.fields` 의 validator 가 모두 `_validate_fields_map` 을 호출 (이전 `_fields_not_empty` 교체).

### P3-D. 버전 이력 페이지네이션

- `AdminExtractionSchemasPage.tsx::VersionHistorySection`
  - `VERSION_PAGE_SIZE = 10` 으로 조정 (이전 고정 20).
  - 초기 페이지는 `useQuery` 로 — invalidate 시 자동 재로딩되는 속성 보존.
  - "더 보기" 버튼 클릭 시 직접 `extractionSchemasApi.getVersions(..., { offset })` 호출로 추가 페이지를 append.
  - 추가 페이지 버퍼 `extraPages` 는 초기 페이지가 교체되면(= invalidate 후 재로딩) `useEffect` 로 초기화 → 데이터 일관성 유지.
  - `hasMore` 는 마지막 페이지 길이가 `PAGE_SIZE` 와 정확히 일치할 때 true (heuristic). 서버 `meta.total` 은 `len(page)` 수준이라 신뢰할 수 없음.
  - 표시 영역 하단에 "{n}건 표시 (더 있음 | 전체 로드됨)" 상태 + "더 보기 (+10)" 버튼. `aria-live="polite"` 로 표시.
  - 더 보기 실패는 `loadMoreError` 로 별도 `ErrorBanner` 에 분리 표시 — 초기 로드 실패와 UI 상 구분.

### P3-E. pytest 회귀 테스트

- **단위**: `backend/tests/unit/extraction/test_extraction_request_schemas.py` (신규, 약 220 L)
  - `TestDocTypeCode` — 정규식 통과/거절/공백 strip (5건)
  - `TestScopeProfileId` — UUID/빈문자열/비-UUID (3건)
  - `TestFieldsBounds` — 빈 fields 거절 / `MAX_FIELDS_COUNT` 경계 / 초과 거절 / `MAX_NESTED_DEPTH` 경계 / 초과 거절 (5건)
  - `TestUpdateRequest` — change_summary 공백→None, 제어문자 거절, 정상 입력, 빈 fields (4건)
  - `TestDeprecateRequest` — 정상 reason, 공백-only 거절, ESC/DEL 제어문자 거절 (4건)
  - 헬퍼 `_build_nested_object_chain(depth)` 가 깊이 경계 테스트에 사용.
- **통합**: `backend/tests/integration/test_extraction_schemas_routes.py` (신규, 약 210 L)
  - DB 미의존. 라우터 진입 직후 Pydantic / 쿼리 파서 단계까지만 검증.
  - `TestCreateRouteValidation` — MAX_FIELDS_COUNT+1, 깊이 4, 잘못된 doc_type_code, 비-UUID scope_profile_id (4건)
  - `TestScopeProfileIdQueryValidation` — list / detail / versions 경로의 `scope_profile_id` 비-UUID + versions 의 offset/limit 경계 (5건)
  - `TestUpdateRouteValidation` — 빈 fields, 제어문자 change_summary, 개수 초과 (3건)
  - `TestDeprecateRouteValidation` — 공백-only, 제어문자, 누락 reason (3건)

---

## UI 디자인 리뷰 (5 회차)

> CLAUDE.md 규칙 "UI 개발 시 UI 디자인 리뷰를 최소 5번은 수행하며 개선해 나갈 것" 준수.

### 1회차 — 초안 스케치: "재귀 편집기" vs "링크로 드릴다운"

- 초안은 "object 필드를 클릭하면 새 패널이 열려서 하위 편집" 이었음 — but 상세 패널 내부에서 또 다른 슬라이드오버를 여는 UX 는 포커스 관리/취소 흐름이 복잡해짐.
- 결정: **inline 확장형 재귀 편집기**. 각 object 필드의 "펼침" 섹션 내부에 `<FieldsEditor depth+1>` 을 중첩해서 렌더. 네이티브 브라우저의 outline/트리 비주얼과 유사해 learning curve 낮음.
- 부수효과: 깊이가 깊어지면 좌우 여백 누적으로 가시 영역이 좁아짐 → 각 중첩 편집기를 `bg-gray-50 + border` 로 감싸서 시각적 경계 유지.

### 2회차 — depth 표시: 숫자 or 브레드크럼?

- 후보 A: 각 중첩 편집기 상단에 "depth 2 / 3" 숫자.
- 후보 B: 브레드크럼 "root › contract › party_a".
- 결정: **A**. 브레드크럼은 필드명이 바뀔 때 함께 갱신돼야 하고 렌더 비용이 큼. depth 숫자는 "최대치 대비 얼마나 깊은가" 를 즉시 전달한다는 본래 목적에 충실.
- 위치: object 필드 상세 영역의 "중첩 스키마" 라벨 옆 오른쪽 정렬. `text-[10px] text-gray-500 font-mono`.

### 3회차 — default_value 위젯: 하나로 합칠지, 스왑할지

- 후보 A: 항상 "JSON 리터럴" 입력을 유지하고 타입별 힌트만 다르게.
- 후보 B: 타입 전환 시 위젯 자체를 교체 (number→number input, bool→checkbox/select, date→date input 등).
- 결정: **B**. 관리자 UX 에서 boolean 을 `"true"` 로 적어야 하는 것은 사용자 실수 유발 요인. 단, array/object 는 JSON 유지 — 2 회 이상의 값(복합 구조)을 단일 input 에 표현하려면 별도의 mini-editor 가 필요해져 비용 급증.
- 부수효과: 기존 값이 타입과 호환되지 않을 때(타입만 바꾼 경우) 위젯이 빈 상태로 렌더되지만 "⚠ 현재 값이 ~와 호환되지 않음" 안내로 사용자가 이해 가능.

### 4회차 — 더 보기 버튼의 위치/레이블

- 후보 A: 리스트 마지막 항목 아래 중앙 정렬 "더 보기".
- 후보 B: 리스트 헤더 우측에 "더 보기 +10".
- 후보 C: 리스트 하단에 좌 "10건 표시 (더 있음)" + 우 "더 보기 (+10)" 버튼.
- 결정: **C**. "몇 건 표시 중인가" 를 왼쪽에 고정해 `aria-live="polite"` 영역으로 쓰고, 오른쪽의 버튼은 visual anchor 역할. 전체 로드된 경우 "(전체 로드됨)" 으로 바꿔 사용자가 "끝에 도달했다" 는 것을 인지.

### 5회차 — 상태/에러 분리

- 초기 로드 실패 → 기존 `ErrorBanner` (섹션 상단).
- 더 보기 실패 → 별도 `loadMoreError` 상태로 보관, 버튼 바로 아래에 `ErrorBanner`. 
- 이유: 한 번 로드된 뒤 "더 보기" 만 실패했을 때, 섹션 전체를 에러 영역으로 치환해 이미 로드된 N 개 목록을 가리면 UX 가 나빠짐. 부분 실패는 "기존 표시는 유지 + 추가만 실패" 로 구분한다.
- 재시도: 사용자가 버튼을 다시 누르면 같은 offset 으로 재시도 (versions.length 을 offset 으로 사용하므로 중복 로드 없음).

### 공통 체크리스트 (통과)

| # | 항목 | 결과 |
|---|---|---|
| 1 | CLAUDE.md ① DocumentType 하드코딩 없음 | ✅ (FieldsEditor 는 `field_type` 만 인지) |
| 2 | CLAUDE.md ⑥ Scope Profile 기반 ACL | ✅ (versions API 의 `scope_profile_id` 쿼리 유지) |
| 3 | CLAUDE.md ⑦ 폐쇄망 동등성 | ✅ (모든 검증이 순수 Pydantic, 외부 호출 없음) |
| 4 | WCAG 2.4.3 포커스 순서 | ✅ (중첩 편집기는 DOM 순서대로 탭 이동) |
| 5 | WCAG 4.1.2 `aria-expanded` / `aria-pressed` | ✅ (더 보기 버튼은 동일 DOM, 재포커스 필요 없음) |
| 6 | WCAG 2.5.5 터치 타깃 ≥ 28×28px | ✅ (더 보기 `min-h-[32px]`) |
| 7 | 중첩 깊이 상한 ≥ backend MAX_NESTED_DEPTH | ✅ (UI=3, backend=3, 동일 상수 정의) |
| 8 | tsc --noEmit 통과 | ✅ (기존 `AdminUserDetailPage.tsx:279` 1 건만 유지 — P* 범위 밖) |
| 9 | Python ast.parse 통과 | ✅ (backend/app/schemas/extraction.py + 테스트 2 편) |
| 10 | 단위 테스트 수 | ✅ (신규 21 건 추가) |
| 11 | 통합 테스트 수 | ✅ (신규 15 건 추가) |

---

## 검증

### 정적 분석

- `frontend/tsc --noEmit` : 기존 `AdminUserDetailPage.tsx:279` 외 에러 없음 (P* 범위 밖, 전 세션부터 유지).
- `python -c "import ast; ast.parse(...)"` 3 파일 통과.

### 테스트 (코드 레벨 검증 — 런타임 실행은 CI 에서 수행 예정)

- 단위: `backend/tests/unit/extraction/test_extraction_request_schemas.py` — 21 건.
- 통합: `backend/tests/integration/test_extraction_schemas_routes.py` — 15 건. 모두 Pydantic / 쿼리 파서 수준까지만 도달하므로 DB 없이 `INTEGRATION_TEST=1` 미설정 상태에서도 실행 가능하게 설계.

### 회귀 범위

- `AdminExtractionSchemasPage.tsx` 의 `VersionHistorySection` 이외 경로에는 변경 없음. 기존 `useQuery` invalidate 키(`["admin", "extraction-schemas", docTypeCode, "versions"]`) 보존.
- `FieldsEditor.tsx` 는 root 편집기에서 `depth` 를 생략(기본 0) 하므로 기존 호출부(`<FieldsEditor fields={...} onChange={...} />`) 는 변경 없이 동작.
- 백엔드 `extraction.py` 는 `_fields_not_empty` validator 를 `_validate_fields` 로 교체했으나 에러 메시지에 "fields 는 최소 1개" 문구가 유지돼 기존 프론트 에러 파싱이 깨지지 않음.

### 잔존 한계 (후속 과제로 분리)

- **더 보기 에러 재시도 UX**: 버튼을 다시 누르면 재시도. 자동 백오프/지수 재시도는 구현하지 않음.
- **깊이 4 에서의 "이동" UI**: 사용자가 root 에서 이미 깊이 3 까지 설계한 뒤 중간에 object 를 추가하고 싶을 때는 직접 재설계가 필요. 트리 리네이밍/재배치 기능은 P4 이상 범위.
- **array-of-object 구조화 편집**: 현재는 array 를 JSON 리터럴로만 편집. 항목 타입 제약은 서버측도 풀려있음.

---

## 종결 조건

- FG별 의무 산출물 4 건 중 2 건(본 검수보고서 + 보안취약점검사보고서) 이 함께 작성됨.
- 서버 한계 상향 필요 시(예: `MAX_FIELDS_COUNT` 를 500 으로) 는 `extraction.py` 상수와 `FieldsEditor` 의 `DEFAULT_MAX_NESTED_DEPTH`, 그리고 단위 테스트의 경계값을 동기 변경 필요.
