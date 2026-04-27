# 함수도서관 (Function Library Index)

> **목적**: Mimir 프로젝트의 공통 유틸 함수 카탈로그. 새로운 기능을 만들기 전에 이 문서를 먼저 확인해 기존 함수를 재사용하도록 안내한다.
>
> **근거 규약**:
> - `docs/규칙/CONSTITUTION.md` 제8조(AI-Modifiable Function Unit), 제10조(Docstring as Agent Contract), 제11조(Function Index as Agent Navigation Map), 제15조(Bounded Change)
> - 프로젝트 `CLAUDE.md` §3.1 "두 번 이상 사용하는 기능은 함수화를 할 것"
> - `CLAUDE.md` §3.1 "공통 유틸을 추가/변경하면 `docs/함수도서관`의 참조 문서도 함께 업데이트할 것"

---

## 0. 최초 생성 메모 (2026-04-25)

- 본 도서관은 2026-04-25 `철균` 지시로 신설되었다. 초판은 **감사(audit) + 제안(proposed) + 초안(draft)** 단계다.
- 이번 턴의 스코프는 아래로 제한되었다 (사용자 선택, CONSTITUTION 제15·32조 준수).
  1. 중복 후보 감사 (`감사보고서_2026-04-25.md`)
  2. 후보에 대한 제안 시그니처·배치·주의점 정리 (`backend.md`, `frontend.md`)
  3. 본 README + 3개 문서의 **텍스트만** 작성 — 실제 `.py`/`.ts` 파일 생성은 이월.
- 후속 턴에서 후보를 선정해 실제 유틸 모듈을 만들 때, 본 도서관을 같은 커밋에서 업데이트한다 (CONSTITUTION 제12조 Index Updates).

---

## 1. 구조

```
docs/함수도서관/
├── README.md                       # 본 문서 — 최상위 인덱스·사용법·상태 범례
├── backend.md                      # 백엔드(Python) 공통 유틸 카탈로그
├── frontend.md                     # 프론트엔드(TypeScript) 공통 유틸 카탈로그
└── 감사보고서_2026-04-25.md        # 최초 중복 감사 결과 (스팟 체크 포함)
```

**배치 규약 (2026-04-25 확정)**:

- 백엔드 공통 유틸은 기존 파이썬 패키지 안에 통합한다: `backend/app/utils/`.
- 프론트엔드 공통 유틸은 기존 TypeScript alias 안에 통합한다: `frontend/src/lib/utils/`.
  - `frontend/src/lib/utils.ts` 단일 파일은 유지(기존 import 호환). 새 유틸은 `frontend/src/lib/utils/<도메인>.ts` 에 도메인별로 분할한다.
- 신규 path `./backend/utils` / `./frontend/utils` 는 만들지 않는다 (CLAUDE.md §3.1 표기는 차기 턴에 실제 경로로 정정 예정).

---

## 2. 상태 범례

| 기호 | 의미 | 해석 |
|------|------|------|
| ✅  | Existing | 이미 구현·사용 중인 유틸 |
| 🟡  | Proposed | 감사에서 후보로 식별됨. 아직 구현되지 않음 |
| 🔵  | Drafting | 초안 작성 중 (PR 진행) |
| ⚠️  | Deprecated | 대체 유틸이 있으며 제거 예정 |

각 항목은 다음 필드를 갖는다 (CONSTITUTION 제11조 권장 포맷 준수).

```
- `module.function_name(args) -> ReturnType`
  - status: 🟡 Proposed
  - purpose: 한 줄 목적
  - effects: none | db_write | external_io | audit_log
  - errors: ErrorClass, ...
  - replaces: 대체 대상 파일/함수 목록
  - tests: 테스트 파일 (구현 시 채움)
  - notes: 보안·ACL·스코프·회귀 주의점
```

---

## 3. 사용 규칙 (개발 전 반드시 확인)

1. **재사용 우선**: 새 헬퍼를 만들기 전에 `backend.md`, `frontend.md` 에서 유사 목적의 ✅ 항목을 검색한다.
2. **2회 규칙**: 두 곳에서 같은 로직을 쓰게 되는 순간 유틸화 후보(🟡)로 등록한다. 구현하지 않더라도 이 도서관에 남겨 추적한다.
3. **인덱스 동기화**: 유틸을 추가·시그니처 변경·제거할 때 **같은 커밋에서** 본 도서관을 업데이트한다(CONSTITUTION 제12조).
4. **ACL·스코프 주의**: 쓰기 계열·DB 접근·외부 I/O 유틸에는 `S2 ⑤~⑦` 원칙(스코프 하드코딩 금지, actor_type 기록, 폐쇄망 degrade)을 반드시 기록한다.
5. **테스트 결합**: 순수 함수 외 유틸은 단위 테스트를 함께 둔다(CONSTITUTION 제15조 Coupled Verification).
6. **단일 책임**: 한 유틸은 한 가지만 한다 (제8조). 분기 플래그로 기능을 덧붙이지 않는다.

---

## 4. 인덱스 대상 판정 (제11조 준수)

- **필수**: public/exported 함수, 서비스·리포지토리 경계 헬퍼, 권한·스코프·감사 관련 헬퍼, 외부 I/O 헬퍼
- **선택**: 단일 파일 내 private helper(`_xxx`), 로컬 변환·계산 함수, 테스트 전용 유틸
- 테스트 전용 유틸은 `backend/tests/`·`frontend/tests/` 내부에 두며 본 도서관에서 제외한다.

---

## 5. 변경 로그

| 날짜 | 변경 | PR/세션 | 비고 |
|------|------|---------|------|
| 2026-04-25 | 초판 생성 (감사 + 제안 + 초안) | 2026-04-25 Cowork 세션 | 실제 구현 없음. 사용자 후속 승인 대기 |
| 2026-04-25 | B1 `utcnow`/`utcnow_iso` ✅ 승격 | B1 세션 | `backend/app/utils/time.py` 신설 + 34 파일 71건 치환 |
| 2026-04-25 | B6 `normalize_display_name` ✅ 승격 | B6 세션 | `backend/app/utils/strings.py` + folders/collections wrapper |
| 2026-04-25 | F1 `formatDateTime` ✅ 승격 | F1 세션 | `frontend/src/lib/utils/date.ts` 신설 + 3 사이트 치환. ISO식(`YYYY-MM-DD HH:mm`) 표기로 통일. golden-sets date-only 는 후속 이월 |
| 2026-04-25 | F6 `useDebouncedValue` ✅ 승격 | F6 세션 | `frontend/src/hooks/useDebouncedValue.ts` 신설 + 3 사이트 치환 (TagChipsEditor 150ms / AddDocsModal 300ms / DocumentListPage 300ms) |
| 2026-04-25 | §1.6a `useDebouncedCallback` ✅ 승격 | 1.6a 세션 | `frontend/src/hooks/useDebouncedCallback.ts` 신설 + 3 사이트 치환 (useUserPreferences PATCH 400ms / DocumentEditPage autoSave 30s / SearchBox onChange 500ms). autoSave 시맨틱 진짜 debounce 로 변경. useUserPreferences cleanup 잠재 버그(mid-burst PATCH 발화) 부수 수정 |
| 2026-04-25 | F §1.2 `toQueryString` ✅ 승격 | F1.2 세션 | `frontend/src/lib/utils/url.ts` 신설 + 7 파일 11 사이트 치환 (tags×2 / extractions / conversation / collections / diff×2 / proposals×3 / CitationItem). 배열·boolean·NaN/Infinity 일관 처리. node:test 32건 추가 (총 355). `parseListFilterParams` 는 별도 후속 |
| 2026-04-25 | F §1.2b `parseListFilterParams` + readers ✅ 승격 | F1.2b 세션 | 같은 `lib/utils/url.ts` 에 `parseListFilterParams` + readers (string/optionalString/bool/enum/boundedInt) + `SearchParamsLike` 인터페이스 추가. DocumentListPage 5필드 / SearchPage 4필드×2회 (초기+useEffect, 모듈 상수 schema 공유) 마이그레이션. node:test 40건 추가 (총 395). extraction-queue/parseFiltersFromUrl 은 bespoke sentinel 보존 위해 비대상 |
| 2026-04-25 | F §1.2c G-Carry `readRegexString` + `mutateSearchParams` ✅ 승격 + buildQueryString 일괄 deprecate | G-Carry 세션 | (1) `readRegexString` reader + `mutateSearchParams` URL state cloning helper 신설. (2) extraction-queue/parseFiltersFromUrl 부분 차용 (documentType+scopeProfileId+page → reader; status 만 bespoke). (3) DocumentListPage 3 + SearchPage 1 mutate 사이트 마이그레이션. (4) admin.ts(12) + s2admin.ts(18) + documents.ts(1) buildQueryString → toQueryString 일괄. buildQueryString 정의 자체는 @deprecated JSDoc 추가, 정의 유지(차후 PR 제거). node:test 23건 추가 (총 418). 정책 전환: "1 작업 = 1 세션" → 그룹화 |
| 2026-04-25 | B §1.5 BE-G3 HTTP 예외 4종 ✅ 승격 (`not_found` / `bad_request` / `unprocessable_entity` / `conflict`) | BE-G3 세션 | `backend/app/utils/http_errors.py` 신설 + `tests/unit/utils/test_http_errors.py`. 14 라우터 파일 215 호출자 일괄 마이그레이션 (AST 기반 변환 스크립트 + UTF-8 byte offset 처리 + top-level import 한정 룰 + final compile 검증). detail 문자열 보존(프런트 호환), error_code 는 X-Error-Code 헤더로 노출. 401/403/500/501/502 는 본 그룹 외 (잔존). 모듈 스모크 8/8 / compileall 0 err / 33 라우터 ast-parse OK. 운영자 macOS 실 pytest 대기 |
| 2026-04-25 | B §1.2 BE-G1 `uuid_str_or_none` ✅ 승격 + `ensure_uuid` ✅ 신설 (호출지 미마이그레이션) + §1.4 `normalize_lower` ✅ 승격 | BE-G1 세션 | `backend/app/utils/converters.py` 신설 (uuid_str_or_none + ensure_uuid) + `app/utils/strings.py` 에 normalize_lower 추가. uuid_str_or_none AST 일괄 마이그레이션 21 파일 47 사이트 (BE-G3 스크립트 룰 재사용). normalize_lower 7 사이트 수동. ensure_uuid 호출지 (UUID() 직호출 72회) 는 try/except ValueError 흐름 다양성 때문에 별 라운드. 스모크 15/15 / compileall 0 err / 271 ast-parse OK. 운영자 macOS 실 pytest 대기 |
| 2026-04-25 | B §1.3 BE-G2 `loads_maybe` + `dumps_ko` ✅ 승격 | BE-G2 세션 | `backend/app/utils/json_utils.py` 신설 + `tests/unit/utils/test_json_utils.py`. dumps_ko AST 일괄 20 파일 41 사이트 + 별칭(_json) 1건 수동 = 42 사이트. loads_maybe 단순 패턴 9 파일 21 사이트 (`if isinstance(X,str): X=json.loads(X)` + `X if isinstance(X,dict) else json.loads(X)` ternary). loads_maybe 는 JSONDecodeError 흐름 보존 (loads_strict 변형은 별 라운드). 자기참조 회피 룰 추가 (스크립트가 자기 모듈 변환 → 무한 재귀 방어). 스모크 12/12 / compileall 0 err / 272 ast-parse OK. 운영자 macOS 실 pytest 대기 |
| 2026-04-25 | F §1.3 + §1.4 + §1.5 FE-G2 작은 단발 utils ✅ 승격 (`classifyApiError` + `downloadJsonFile` + type guards 3종) | FE-G2 세션 | `lib/utils/errors.ts` + `lib/utils/download.ts` + `lib/utils/guards.ts` 신설 + `tests/UtilsErrorsDownloadGuardsFG2.test.ts` (22 cases). 시범 마이그레이션: extraction-queue/helpers.ts mutationErrorMessage 본 helper 위에서 재구성 + golden-sets/AdminGoldenSetsPage 의 로컬 downloadJsonFile thin wrapper 화 + ConversationList handleExport 인라인 5줄 제거. type guards 51회 호출지 마이그레이션은 별 라운드. node:test 440 (+22) / tsc 0 touched 오류 |
| 2026-04-25 | F §1.7 FE-G3 admin 라벨/배지 상수 ✅ 승격 (5 도메인 라벨 + 3 도메인 배지 = 8 객체) | FE-G3 세션 | `lib/constants/{labels,badges}/` 디렉토리 신설 + 5 도메인 파일 (extraction/goldenSet/goldenSetDomain/evaluation/evaluationMetric) + index.ts 재노출. 마이그레이션: extraction-queue/constants.ts thin re-export + AdminGoldenSetsPage 인라인 3 객체 제거 + evaluations/helpers.ts 의 STATUS_LABEL/STATUS_BADGE_STYLE/METRIC_LABELS 3 객체 thin re-export (정책 상수 METRIC_THRESHOLDS/scorePasses 등은 helper 잔존). 호출지 16+ 파일 호환 보존. labels/badges 파일 분리로 i18n 도입 대비. node:test 461 (+21) / tsc 0 touched |
| 2026-04-25 | F §1.8 FE-G4 alert/badge className 토큰 ✅ 승격 (BADGE_BASE + ALERT_ERROR/WARNING/INFO/SUCCESS = 5 토큰) | FE-G4 세션 | `lib/styles/tokens.ts` 신설 + `tests/StylesTokensFG4.test.ts` (14 case). 시범 마이그레이션: BADGE_BASE 4 사이트 (extraction-queue + evaluations 3종) + ALERT_ERROR 3 사이트 (login/register/reset-password). 변형 (rounded-full / responsive padding / text-red-600) 23+ 사이트와 다크모드 토큰은 별 라운드. base 에 색상 토큰 미포함 — 호출자가 cn 으로 합성. node:test 475 (+14) / tsc 0 touched |
| 2026-04-25 | B §1.6 BE-G4 `actor_type_str` ✅ 승격 + §1.7 `emit_audit` 도입 안 함 결정 | BE-G4 세션 | `backend/app/utils/actor.py` 신설 (actor_type_str + AuditActorType alias) + `tests/unit/utils/test_actor.py`. **잠재 보안 버그 수정** — `ActorContext.audit_actor_type` 가 SERVICE 일 때 `"service"` 반환 (AuditEmitter Literal 위반) → 본 helper 위임으로 `"system"` 매핑 통일. property 위임으로 9 호출 사이트 자동 영향. 시범 마이그레이션 2 (golden_sets/_actor_type_str + extraction_schemas/_actor_info). emit_audit 은 AuditEmitter 가 이미 충분 + 162 호출자 컨텍스트 다양성 때문에 도입 안 함. 스모크 6/6 / compileall 0 err / 273 ast-parse OK. 운영자 macOS 실 pytest 대기 |
| 2026-04-25 | B §1.8 + §1.9 BE-G5 DB 헬퍼 + 페이지네이션 ✅ 승격 (`fetch_one_as` + `fetch_many_as` + `clamp_pagination`) | BE-G5 세션 | `backend/app/db/cursor_helpers.py` + `backend/app/repositories/pagination.py` 신설 + `tests/unit/utils/test_cursor_helpers.py` (9 case) + `test_pagination.py` (17 case). 시범 마이그레이션: tags_repository autocomplete (fetch_many_as + clamp_pagination) + popular (clamp_pagination). 일괄 (266+107=373 fetchone/fetchall + 10+ page→offset 계산) 은 mapper/컨텍스트 다양성 때문에 별 라운드. 스모크 8/8 / compileall 0 err / 275 ast-parse OK. 운영자 macOS 실 pytest 대기 |
| 2026-04-25 | F §1.1 FE-G1 `formatDateOnly` ✅ 승격 + `formatRelativeShort` 도입 안 함 결정 | FE-G1 세션 | `lib/utils/date.ts` 에 formatDateOnly 추가 (F1 모듈에 함께) + tests/FormatDateTimeF1.test.ts 에 8 case 추가. 시범 마이그레이션: golden-sets 로컬 formatDate → formatDateOnly alias 위임 (가시 UX: ko locale → ISO식 통일, 철균 합의). admin 인라인 toLocaleDateString 8+ 사이트는 가시 변경 영향으로 별 라운드. formatRelativeShort 는 호출지 0건 + 영/한 표기 결정 미정으로 도입 안 함. node:test 483 (+8) / tsc 0 touched |
| 2026-04-25 | R1 잔존 후보 묶음 (`paginate_page` + `loads_strict` ✅ 신설 + `actor_type or "user"` 정정) | R1 세션 | (1) paginate_page (clamp_pagination 위 page→offset 흡수) + 10 테스트. (2) loads_strict (JSONDecodeError → ApiValidationError) + 6 테스트. (3) **잠재 보안 정정**: `actor_type=actor.actor_type or "user"` 5 사이트 (extractions / extraction_evaluations×2 / batch_extractions×2) — Enum 인스턴스는 항상 truthy 라 fallback 작동 안 함. actor_type_str(actor) 로 정정 (BE-G4 보안 시맨틱 적용). 스모크 9/9 / compileall 0 err / 275 ast-parse OK |
| 2026-04-25 | R2 FE 호출지 점진 (admin date 11 + type guards 20 + classifyApiError 결정) | R2 세션 | (R2-A) admin toLocaleDateString 11 사이트 → formatDateOnly: account/profile + extraction-schemas + ai-platform + orgs ×3 + users/Detail ×3 + users/List ×2 + document-types + jobs (가시 UX: ko locale → ISO식 통일). (R2-B) type guards `typeof X === "string"` AST 자동 변환 20 사이트 in 10 파일 (FieldsEditor 7 / api/client 5 / settings 2 / 그 외 6). url.ts 의 union narrowing 케이스 1건은 user-defined type guard 가 좁히지 못해 typeof 유지. import 누락 2건 수동 보강 (FieldsEditor / lib/api/auth). (R2-C) classifyApiError 49 호출지 마이그레이션 **안 함** — getApiErrorMessage 가 401/403/409 도메인 한국어 라벨을 인라인 보존. 두 helper 공존이 도서관 §1.3 의도. node:test 483 / tsc 0 |
| 2026-04-25 | R3 §1.8 디자인 토큰 확장 (변형 alert + 다크모드) | R3 세션 | lib/styles/tokens.ts 에 BADGE_BASE_PILL (rounded-full) + ALERT_ERROR_COMPACT (text-red-600 + responsive padding + animate-in) + 다크모드 5 토큰 (BADGE_BASE_DARK + ALERT_*_DARK 4종, dark: 접두). preferences.theme 토글과 호환 (cn 합성으로 light+dark 동시 적용). 호출지 마이그레이션은 별 라운드 (admin 다크 전면 도입 시점). 테스트 12 case 추가. node:test 495 (+12) / tsc 0 |
| 2026-04-25 | R4 BE 호출지 점진 시범 (paginate_page 3 + fetch helpers 4) | R4 세션 | (R4-A) paginate_page 3 사이트 시범 (proposal_queue listMine + listAdmin + admin/listUsers). `(page-1)*page_size` 인라인 → `page, page_size, offset = paginate_page(page, page_size, max_page_size=...)`. (R4-B) fetch_one_as/fetch_many_as 4 사이트 (settings_repository 의 list_all/list_by_category/get_one/list_categories). 이외 17+ paginate_page 사이트 + 369+ fetch 사이트 + 162 emitter 호출자 + ensure_uuid 호출지 + AuditEmitter trace_id contextvar 는 호출 컨텍스트 다양성 + 보안 영향으로 **별 라운드 (점진)**. compileall 0 err / 275 ast-parse OK. 운영자 macOS 실 pytest 대기 |
| 2026-04-25 | R5+R6 paginate_page 잔존 일괄 + fetch helpers 단순 점진 | R5+R6 세션 | (R5) paginate_page AST 일괄 17 사이트 (admin 7 / vectorization 2 / search_service 3 / scope_profiles / admin_extraction_results / rag / documents_repo / versions_repo). `(page-1)*page_size` 패턴 잔존 0건 (helper 자체 docstring 외). (R6) fetch_one_as/fetch_many_as 단순 패턴 AST 일괄 22 사이트 (users_repository 12 / tags 3 / collections 2 / folders 2 / rag 2 / workflow 1). 다중 execute / RETURNING / 트랜잭션 / 별칭 conn 변수 케이스는 보존. compileall 0 err / 275 ast-parse OK. 운영자 macOS 실 pytest 대기 |
| 2026-04-25 | R7+R8+R9 잔존 카테고리 결정 + AuditEmitter ContextVar | R7-R9 세션 | (R7) ensure_uuid 호출지: 안전 단순 직호출 13 사이트 분석 — try/except 흐름 다양성 + ApiValidationError vs unprocessable_entity 도메인 메시지 손실 위험 → **마이그레이션 안 함 결정**, 별 도메인 PR 검토. (R8) loads_strict 호출지: 21 loads_maybe 호출자 모두 JSONB 컬럼 읽기 (DB corruption 가정 외 안전) → **마이그레이션 안 함 결정**, raw 사용자 입력 검증 신규 호출자에 사용. (R9) AuditEmitter `_trace_id_var` ContextVar 도입 + `set_trace_id`/`reset_trace_id`/`current_trace_id` export + emit() 안에서 trace_id None 이면 contextvar fallback + RequestContextMiddleware 가 try/finally 로 set/reset. 모듈 스모크 4/4 PASS. compileall 0 err / 275 ast-parse OK |
| 2026-04-25 | 감사 2회차 + §1.10+ 신규 후보 5건 등록 | 감사 2회차 세션 | `감사보고서_2026-04-25_2회차.md` 신설. **BE 4 신규 후보**: B-N1 require_actor_id (folders+collections 2 사이트, B6 패턴) / B-N2 require_admin (proposal_queue+scope_profiles 2 사이트) / B-N3 ADMIN_ROLES 상수 (3+ 사이트) / B-N4 not_found_resource (영문 `f"X '{id}' not found"` 24 사이트, 한국어 통일 옵션). **FE 1 신규 후보**: F-N1 FORM_ERROR_INLINE 토큰 (text-xs text-red-600 mt-1 변형 5+ 패턴, 45 사이트). 보류 재검토: success_response 213 / queryKey 180 / EmptyState 26 / useMutationWithToast 38 — 의미 다양성으로 보류 유지. 코드 변경 없음 (감사 + 문서만) |
| 2026-04-25 | B §1.10 BE-G6 권한 가드 ✅ 승격 (`require_actor_id` + `require_admin` + `ADMIN_ROLES`) | BE-G6 세션 | `app/utils/actor.py` 확장 (3 export 추가) + `tests/unit/utils/test_actor.py` 16 case 추가 (TestRequireActorId 6 + TestRequireAdmin 6 + TestAdminRoles 4). 4 사이트 마이그레이션: folders_service + collections_service `_require_actor` thin wrapper 위임 (메시지 도메인 라벨 표준화) + proposal_queue + scope_profiles `_require_admin` + `_ADMIN_ROLES` thin re-export. 모듈 스모크 10/10 PASS / compileall 0 err / 275 ast-parse OK. 잔존 _ADMIN_ROLES 정의 0건 (위임 외) |
| 2026-04-25 | B §1.5-extension B-N4 `not_found_resource` ✅ 승격 (한국어 통일) | B-N4 세션 | `backend/app/utils/http_errors.py` 에 `not_found_resource(label_ko, resource_id, *, error_code=None) -> ApiNotFoundError` 신설 (지연 import 로 ApiError 순환 회피). 표준 메시지 `f"{label_ko}을(를) 찾을 수 없습니다: {resource_id}"`. **BE-G3 `not_found` 와 다른 계층** — 그건 HTTPException, 본 helper 는 ApiError 계층 (handlers 가 404 응답 변환). `tests/unit/utils/test_http_errors.py` §6 (10 case) 추가. AST 일괄 마이그레이션 **26 사이트** in 8 files (documents_service 2 / collections_service 2 / folders_service 7 / versions_service 3 / nodes_service 2 / draft_service 7 / workflow_service 2 / api/v1/documents 1). 라벨 매핑: Document→문서 / Version→버전 / Folder, Parent folder→폴더 / Collection→컬렉션. 비대상 5 사이트 (`f"X '{id}' not found in version '{y}'"` 이중 컨텍스트, 별 라운드). compileall 0 err / 275 ast-parse OK / helper 스모크 5/5 PASS. 운영자 macOS 실 pytest 대기 |
| 2026-04-25 | F §1.9 F-N1 폼 인라인 에러 토큰 4종 ✅ 승격 (FORM_ERROR_INLINE/STRONG/BANNER/BOX + 다크 변형) | F-N1 세션 | `lib/styles/tokens.ts` 에 4 캐논 토큰 (`FORM_ERROR_INLINE` `mt-1 text-xs text-red-600` / `_STRONG` auth `mt-1.5 + font-medium` / `_BANNER` mb-3 / `_BOX` `bg-red-50 + rounded-lg`) + 4 다크 변형 (`dark:` 접두) 추가. R3 패턴 따라 라이트+다크 동시 도입. `tests/StylesTokensFG4.test.ts` §F-N1 23 case 추가 (토큰 클래스 정확성 + cn 합성 + 다크 변형). 시범 마이그레이션 **11 사이트** in 6 files (AuthInput 1 + PasswordInput 1 + AdminPromptsPage 3 + AdminScopeProfilesPage 2 + AdminAgentsPage 3 + FieldEditor 1). 잔존 30+ 변형 사이트 (text-red-500 / text-red-700 도메인 컨텍스트 / 통계 표시 / 인터랙티브 요소) 별 라운드. **R3 `ALERT_ERROR_COMPACT` 와 다른 계층** (그건 박스, 본 토큰은 인라인). 컴포넌트화 (`<FormErrorMessage>`) 는 도입 보류 — 호출자가 `role="alert"` 명시. node:test 518 (+23) / tsc 0 touched 오류 |

---

## 6. 관련 문서

- `docs/규칙/CONSTITUTION.md` — 헌법 본문
- `docs/규칙/CLAUDE.md`, `AGENTS.md` — 역할별 운영 지침
- `docs/규칙/CONTRIBUTING.md` — 일상 기여 루틴 (`functions.index.md` 언급 조항 §0)
- `CLAUDE.md` (프로젝트 루트) — 프로젝트 개발 규칙
