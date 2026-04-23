# DataTable · item_count 단위 테스트 검수 보고서 (S2-5)

| 항목 | 값 |
|------|------|
| 작성일 | 2026-04-21 |
| 범위 | `frontend/src/components/admin/DataTable.tsx`, `backend/app/repositories/golden_set_repository.py` (item_count 경로), `backend/app/models/golden_set.py` (DTO) |
| 관련 작업 | Task #34~#38 (이 conversation) |
| 배경 | auto-memory `project_s2_closure_gaps`의 P0 "커버리지 35%" 항목 대응 — S2-5 에서 가장 많이 사용되는 generic DataTable 과 list_by_scope 경로의 item_count 부착 로직에 실측 단위 테스트 확보 |

---

## 1. 배경

- 이전 S2-5 런타임 회귀 리뷰(#24)에서 `DataTable` 의 키보드 접근성(#15), 목록 에러 분기(#31), 삭제 버튼(#33) 등 여러 분기가 Chrome 상에서만 검증됨.
- auto-memory `S2 종결 선언 vs 실측 갭` 에 의해 "커버리지 35% / AI 품질 실측 부재" 가 P0 으로 남아 있음.
- 이번 FG 는 그 중 재사용성이 가장 높은 2개 모듈(`DataTable` 프런트 + `list_by_scope → item_count` 백엔드) 에 대한 **단위 테스트** 를 우선 확보.

## 2. 의존성 정책 (CLAUDE.md 준수)

| 원칙 | 적용 방식 |
|------|-----------|
| "deprecated 기능 금지" (global #1) | Vitest 대신 Node 22 내장 `node:test` 사용 — 공식 안정 API (v20 이후 stable). |
| "critical 취약점 있는 라이브러리 금지" (global #2) | 신규 devDep **0개**. 기존 React 19, react-dom 19, typescript 5, node_modules 만 사용. |
| "보안" (global #3) | `_count_items` SQL 파라미터화 검증 테스트 추가 (SQL injection 방어). 악성 id 문자열이 SQL 에 interpolation 되지 않는지 정적 검사. |
| "UI 리뷰 5회 이상" (global #4) | 본 테스트는 UI 개발이 아닌 테스트 코드이므로 직접 적용 대상 아님. UI 자체는 기존 #15(P0-2), #19(Chrome 5+회) 로 충족. |
| FG 산출물 (project CLAUDE.md) | 검수보고서 + 보안취약점검사보고서 2종 작성 (본 문서 + `..._보안취약점검사보고서.md`). |

> **사유 기록**: 사용자는 `AskUserQuestion` 으로 "Vitest + RTL 도입 (Recommended)" 을 선택했으나, sandbox npm registry 접근 제한 (E403 Forbidden 확인됨) 으로 설치 불가. 동일 커버리지를 신규 의존성 0개로 달성하기 위해 `node:test` + `react-dom/server` + React element-tree 직접 순회로 피벗. 이 결정은 신규 CVE 리스크를 0 으로 유지하며, 프로젝트 폐쇄망 원칙 (S2 ⑦) 과도 정합함.

## 3. 프런트엔드 (`DataTable`)

### 3.1 인프라

| 파일 | 용도 |
|------|------|
| `frontend/tsconfig.test.json` | tests/ + 테스트 대상 소스 파일만 CJS 로 컴파일. Next.js 런타임 빌드와 완전 분리. |
| `frontend/tests/register.js` | `Module._resolveFilename` 훅으로 `@/` alias 를 `dist-tests/src/` 로 rewrite (CJS 런타임 전용, 번들에 영향 없음). |
| `frontend/tests/DataTable.test.tsx` | 단위 테스트 파일. |
| `frontend/package.json` → scripts.test | `tsc -p tsconfig.test.json && node --require ./tests/register.js --test "dist-tests/tests/*.test.js"` |

### 3.2 테스트 범위 (7 suite / 18 test 전부 pass)

| Suite | 검증 내용 |
|-------|-----------|
| 헤더 | `<th scope="col">` 생성, `col.width` → style, `ariaLabel` → `<table aria-label=...>` 전달 |
| 로딩 | `loading=true` 시 `aria-busy="true"`, 스켈레톤 5행, `animate-pulse`, 실제 rows 무시 |
| 빈 상태 | `rows=[]` → `colspan=컬럼수`, `emptyMessage` 기본값 / 커스텀 |
| 행 렌더링 (non-interactive) | `col.render(row)` 호출, `onRowClick` 미제공 시 `role/tabIndex/onClick/onKeyDown` 전부 미부여 |
| 상호작용 행 (P0-2 회귀 방어) | `onRowClick` 제공 시 `role="button"`, `tabIndex=0`; `onClick()` → 해당 row callback; `onKeyDown Enter/Space` → `preventDefault` + callback; 기타 키 → noop; hover/focus-visible 클래스 |
| rowKey | 각 row 로 호출, HTML 에 콘텐츠 반영 |
| className | prop 이 루트 div 에 병합 |

### 3.3 실행 결과

```
# tests 18
# suites 7
# pass 18
# fail 0
# duration_ms 84.5
```

(전체 TAP 로그는 `npm test` 재실행 시 확인 가능.)

### 3.4 전략 요약

- **마크업 수준** 은 `renderToStaticMarkup` 으로 HTML regex 검증 — jsdom 불필요.
- **핸들러 호출** 은 DataTable 을 함수로 직접 호출해 React element 트리에서 props 를 꺼내 **직접 invoke** — React는 hook 없는 순수 함수 컴포넌트이므로 안전.
- **헤더 tr 제외** 는 `children[0].type === "td"` 로 본문 행만 선별 (`isBodyRow` helper).

## 4. 백엔드 (`item_count`)

### 4.1 파일

- `backend/tests/unit/test_golden_set_repository_item_count.py` (신규, 318 lines, 5 class / 16 test)

### 4.2 테스트 범위

| Class | 테스트 | 검증 |
|-------|--------|------|
| `TestListByScopeAttachesItemCount` | `test_two_rows_each_get_own_count` | 각 row 에 `_count_items` 결과가 attach — side_effect 시퀀스로 fetchone 3회(total, row_a, row_b) |
| | `test_empty_list_returns_total_0` | 빈 결과도 예외 없이 (total=0, sets=[]) |
| | `test_scope_id_is_passed_to_sql` | S2 ⑥: COUNT/LIST 모두 `scope_id=%s` + `is_deleted=FALSE` 포함, params[0]=scope_id |
| `TestCountItemsSql` | `test_count_items_where_clause_has_is_deleted_false` | `FROM golden_items` + `golden_set_id=%s` + `is_deleted=FALSE`, 결과 정수 반환 |
| | `test_count_items_returns_zero_for_empty_set` | 0-count 경로 |
| | `test_count_items_no_hardcoded_id_in_sql` | 악성 문자열(`'; DROP TABLE ...`) 이 SQL 에 포함되지 않고 파라미터로만 전달 (SQL injection 방어 회귀) |
| `TestGetVersionHistoryItemCount` | `test_item_count_from_snapshot_length` | `items_snapshot` 리스트 길이를 `item_count` 에 매핑 (3/1/0) |
| | `test_item_count_null_snapshot_is_zero` | `items_snapshot IS NULL` → `len(None or []) = 0` null-safe |
| | `test_nonexistent_parent_returns_empty_list` | S2 ⑥: 접근 거부/미존재 동일 처리 → `[]` |
| `TestItemCountDTOSerialization` | `test_golden_set_model_item_count_default_none` | 모델 기본값 None |
| | `test_golden_set_model_item_count_attribute_assignment_allowed` | 런타임 `gs.item_count = 42` 가능 (field 로 선언되어 있음) |
| | `test_golden_set_response_item_count_serializes_int` | 정수 직렬화 |
| | `test_golden_set_response_item_count_none_by_default` | None 직렬화 시에도 key 유지 |
| | `test_version_info_item_count_required` | `GoldenSetVersionInfo.item_count` 는 필수 int |
| | `test_version_info_item_count_negative_allowed_but_typed` | 0 케이스 |
| `TestListByScopeRegressionP0` | `test_freshly_created_set_zero_items_listing_does_not_raise` | #30 (생성 직후 GET 503) 회귀 방어 픽스처 — item 0개로도 예외 없음 |

### 4.3 실행

- sandbox 환경은 PyPI 접근 차단 + 기존 `.venv` 가 macOS Framework 경로(`/Library/Frameworks/Python.framework/...`) 로 심볼릭 깨짐 → **sandbox 내 pytest 실행 불가**.
- 대신 수행:
  1. `python3 -m py_compile` 로 구문 검증 → **통과**.
  2. AST 로 top-level 정의 확인 → 5 class / 16 test 모두 인식.
  3. 프로덕션 코드와 mock side_effect 시퀀스의 call-order 를 수동 트레이스 → 정합.
- **로컬 실행 명령** (사용자 머신):
  ```bash
  cd backend
  source .venv/bin/activate
  pytest tests/unit/test_golden_set_repository_item_count.py -v --no-cov
  # 기존 golden_set 모델 테스트와 함께:
  pytest tests/unit/models/test_golden_set.py tests/unit/test_golden_set_repository_item_count.py -v
  ```
- 예상: 16 test 전부 pass. 실패 시 가장 가능성 높은 원인은 (a) psycopg2 RealDictCursor 가 tuple 을 반환하도록 설정된 경우 (dict key 접근 실패), (b) pydantic 2.x strict-mode 로 `item_count` 런타임 할당 제한 — 두 경우 모두 conftest.py 기본과 정합함을 확인.

## 5. 정적 분석 결과

| 도구 | 결과 |
|------|------|
| `tsc -p tsconfig.test.json` | 0 errors. `exactOptionalPropertyTypes` 기반 오류 없음. |
| `python3 -m py_compile` (backend test) | 0 errors. |
| `eslint` (frontend) | 신규 파일 `tests/*` 는 default eslint config 범위 밖(이전 baseline 과 동일 — eslint.config.mjs 검사 대상은 src/). 필요 시 후속으로 tests/ include 추가 가능. |

## 6. 잔존 위험 및 후속 제안

1. **백엔드 테스트 실제 실행** — sandbox 에서 미실행. 사용자 로컬 venv 에서 1회 pytest 실행으로 pass 확인 필요.
2. **DataTable 통합 테스트** — 실제 브라우저 포커스 이동/키보드 순회(Tab, Shift+Tab) 는 node:test 로 검증 불가. #19(Chrome 5+회) 에서 이미 커버된 항목을 회귀 방지용으로 Playwright 도입 시 보강 권장 (별도 FG).
3. **list_by_scope 필터 조합** — domain/status 쿼리 파라미터의 WHERE 조합은 이번 범위에서 제외. 후속 FG 에서 추가 권장.
4. **coverage report 수치 업데이트** — `--cov=app` 기반 coverage.xml 은 이번 작업으로 golden_set_repository.py 의 `list_by_scope`, `_count_items`, `get_version_history` 분기 커버리지가 상승할 것으로 예상. pytest 재실행 후 htmlcov/index.html 에서 확인 가능.

## 7. 결론

- 신규 devDep 0개로 프런트·백엔드 양쪽 코어 경로에 대한 단위 테스트 확보.
- 프런트엔드 18건 sandbox 에서 실측 pass.
- 백엔드 16건은 sandbox 제약(PyPI firewall + 깨진 venv) 으로 구문·AST·트레이스 검증만 수행; 사용자 로컬 환경에서 `pytest` 1회 실행으로 최종 확정.

---

**작성자**: Cowork 자동화 (Claude)
**관련 문서**: `UnitTests_DataTable_ItemCount_보안취약점검사보고서.md`
