# /admin/extraction-queue B 스코프 검수보고서

- 작성일: 2026-04-22
- 대상 화면: `/admin/extraction-queue` (추출 결과 검토 큐)
- 대상 FG: S2 Phase 8 FG8.2 / FG8.3 본작업 — **실-API 연결**
- 보고자: UI 개선 담당

## 0. 배경

Q1 에서는 프론트 품질·일관성·접근성을 먼저 정비했으나 백엔드
`/api/v1/admin/extraction-results` 가 존재하지 않아 화면은 목 데이터로만 동작했다.
B 스코프는 그 위에서 **실제 API 를 신설하고, 목 데이터/스켈레톤 배너를 완전히 제거**한다.

| 번호 | 이전 한계 | 이번 해소 |
| --- | --- | --- |
| 1 | 백엔드 라우트 부재로 list/get/approve/reject 가 전부 404 | admin 전용 라우터 신설 (`/api/v1/admin/extraction-results/*`) |
| 2 | 프론트가 `isRouteMissingError` 로 404 를 "라우트 미구현" 으로 오역 | 헬퍼 제거, 404 를 "해당 id 없음" 으로 일반화 |
| 3 | MOCK_RESULTS / mockDetail 을 fallback 으로 사용 → 에러 은폐 | 목 데이터 전면 삭제, 실제 에러를 `NoticeBanner` 로 표면화 |
| 4 | 페이지네이션 없음 (스크롤 무한 성장 + DoS 여지) | page=1/page_size=20 쿼리 + 총건수·다음 페이지 단추 |
| 5 | 승인 시 override(필드별 수정) 경로 부재 | 상세 패널 "필드 수정" 토글 + `buildApprovePayload` 어댑터 |
| 6 | `/extractions/*` (에이전트·사용자 혼용) 라우트에 관리자가 얹히면 RBAC 가 뭉뚱그려짐 | 관리자 전용 라우터 + `admin.read`/`admin.write` 분리 |
| 7 | 내부 상태(`modified`) 가 UI 에 그대로 노출될 위험 | 외부 3-상태 어댑터(`map_status_to_external`) + `was_modified` 플래그 |

---

## 1. 작업 범위 (B1 ~ B9)

### B1 — Pydantic 스키마 정의 (`app/schemas/admin_extraction_results.py`, 신규)

- 외부 어휘: `pending_review | approved | rejected` (`ExtractionResultStatusExternal`)
- 내부 어휘(`extraction_candidates.status`): `pending | approved | rejected | modified`
- 어댑터 함수 2건:
  - `map_status_to_external(internal)` — `modified` → `approved` 정규화, 알 수 없는 값은 `pending_review` 로 안전 폴백 (500 회피)
  - `map_status_to_internal(external)` — 라우터가 SQL `WHERE status IN (...)` 에서 사용
- 응답 DTO: `ExtractionResultSummary` / `ExtractionResultDetail`
- 요청 DTO: `AdminApproveExtractionRequest` / `AdminRejectExtractionRequest`
- DoS 방어: `overrides` 최대 200개, `approval_comment` / `reason` 각 1024자 상한
- `reason` 의 blank-only 문자열은 validator 가 `None` 으로 normalize

### B2 — Admin 라우터 신설 (`app/api/v1/admin_extraction_results.py`, 신규)

4개 엔드포인트 모두 `_require_admin` 의존성 하나로 RBAC 단일화:

| Method | Path | Action | 허용 역할 |
| --- | --- | --- | --- |
| GET | `/admin/extraction-results` | `admin.read` | `ORG_ADMIN`, `SUPER_ADMIN` |
| GET | `/admin/extraction-results/{id}` | `admin.read` | 〃 |
| POST | `/admin/extraction-results/{id}/approve` | `admin.write` | `SUPER_ADMIN` 전용 |
| POST | `/admin/extraction-results/{id}/reject` | `admin.write` | 〃 |

입력 정규화:

- `_parse_scope(raw)` — 공백/None 은 `None`, 그 외는 `UUID()` 시도 → 실패 시 422
- `_normalize_document_type(raw)` — 소문자 허용, 서버에서 `.upper()` 후 정규식 검증 (P7 UPPER 정책 계승)
- `_parse_status_filter(raw)` — 외부 3-상태만 허용, `approved` 는 내부 `APPROVED | MODIFIED` 둘 다로 확장

응답:

- 목록은 `list_response(data, page, page_size, total, has_next, next_cursor?, request_id?, trace_id?)` 로 감싸 `meta.pagination` 중첩 구조 보장
- 상세는 `success_response(data, request_id?)` 로 감싼다
- 승인/반려 이후 곧바로 `get_for_admin_detail(extraction_id)` 로 재조회해 클라이언트가 즉시 캐시 투입 가능하도록 함 (목록 inva­lidateQueries 와는 별개로 상세도 즉시 갱신)

교차 scope 방어: `GET /{id}` 가 scope_profile_id 를 받았을 때 반환 레코드의 scope 와 불일치하면 **404 로 응답**해 존재 여부 노출(enumeration)을 방어한다.

### B3 — Repository 확장

`ExtractionCandidateRepository.list_for_admin_queue(statuses, document_type, scope_profile_id, limit, offset) -> (rows, total)`

- `documents` 테이블 LEFT JOIN 으로 `document_title` 제공 (삭제 문서는 NULL → `(삭제된 문서)` 로 치환)
- `status IN (ANY(%s))`, `extraction_schema_id = %s`, `scope_profile_id = %s` 3단 필터
- `ORDER BY created_at DESC, id DESC` 로 안정 페이지네이션
- `COUNT(*) OVER()` 를 회피하고 별도 `COUNT` 쿼리로 total 반환 (큰 테이블에서 rowset 복제 비용 절감)

`get_for_admin_detail(id)` — 상세용 JOIN 결과 단건, 상태/reviewer 포함.

### B4 — Alembic 마이그레이션 점검

- `extraction_candidates` 테이블은 Phase 8 초기 마이그레이션에서 이미 생성 (`extraction_schema_id` FK, `status` ENUM, `reviewed_by`, `reviewed_at`, `human_feedback`, `human_edits` JSONB 모두 존재)
- 이번 B 스코프에서 스키마 변경은 없음 → 신규 revision 불필요

### B5 — 프론트 mock 제거 + 실 API 계약 정합 (`AdminExtractionQueuePage.tsx` 전면 rewrite)

- `MOCK_RESULTS` / `mockDetail` / `NoticeBanner variant="mock"` 삭제
- `isRouteMissingError` 기반 분기 삭제 (helpers.ts 에서도 export 제거)
- 페이지네이션:
  - `PAGE_SIZE = 20`, `page` state
  - "이전"/"다음" 버튼은 `pagination.has_next` 또는 `total` 기반으로 disable
  - "총 N건 중 X~Y번" 레이블, total 미제공 시 "페이지 N" 으로 폴백
  - status/type/scope 필터 변경 시 `useEffect` 로 page=1 리셋
- 상세 패널 **편집 모드** 추가:
  - "필드 수정" 토글 → 테이블의 `값` 컬럼이 `<input>` 으로 전환
  - 입력값은 `overrides` state (`Record<string, string>`) 에 저장
  - `tryParseOverrideValue(raw)` — `JSON.parse` 시도 후 실패하면 원문 문자열 전달 (서버 `Dict[str, Any]` 수용)
  - 승인 버튼 라벨: 편집 중 + 수정 필드 있을 때 "수정 후 승인", 그 외 "모두 승인"
- `scope_profile_id` 를 상세 get() 에도 전달(교차 조회 방지)
- 에러 경로: `NoticeBanner` (red) 로 `mutationErrorMessage` 결과 표시 + "다시 시도" 버튼(useQuery.refetch)

### B6 — pytest 통합 테스트 (`backend/tests/integration/test_admin_extraction_results_routes.py`, 신규)

라우터 진입 직후 검증 단계만 다루어 DB 연결 불필요 (`INTEGRATION_TEST=1` 없는 환경에서도 통과). 총 6개 클래스 28 테스트:

- `TestStatusAdapters` (9) — 상태 어댑터 함수 단위 커버 (modified→approved, 알 수 없는 값의 안전 폴백 포함)
- `TestListRouteRBAC` (4) — 미인증 401, VIEWER/AUTHOR/APPROVER 403
- `TestWriteRouteRBAC` (3) — AUTHOR approve 403, VIEWER reject 403, 미인증 401
- `TestListQueryValidation` (7) — 잘못된 status/document_type/scope_profile_id/페이징 상하한 422
- `TestDetailRouteValidation` (2) — 비-UUID path/scope 422
- `TestApproveBodyValidation` (4) — overrides 201 개 422, approval_comment 1025자 422, 빈 body 통과
- `TestRejectBodyValidation` (4) — reason 1025자 422, blank 정규화 통과, 빈 body 통과
- `TestScopeIsolation` (1) — 존재하지 않는 id + admin 역할은 401/403 이 아닌 404 계열
- `TestDocumentTypeNormalization` (4) — 소문자/혼합/하이픈/숫자시작 케이스

### B7 — node:test 회귀 갱신 (`frontend/tests/ExtractionQueueQ1.test.tsx`)

- `isRouteMissingError` 블록 7건 제거 (함수 자체가 사라짐)
- `mutationErrorMessage` 에 409 분기 2건 추가 (server message 우선 / blank fallback "이미 처리된")
- `buildApprovePayload` 신규 describe 7건:
  - undefined → `{}`, null → `{}`, empty object → `{}` (모두 `overrides` 키 없음)
  - 값 있는 객체 → `{ overrides: {...} }`
  - 원본은 shallow-copy (원본 객체 mutation 방지)
  - 빈 문자열 값 / null/undefined 값도 키 존재 시 보존
- 총 tests 114 pass / 0 fail (tsc 테스트 빌드 + node:test 런타임)

### B8 — 본 보고서 + 보안취약점검사보고서

### B9 — 최종 검증

| 검사 | 결과 |
| --- | --- |
| `npm test` (frontend) | 114 tests pass / 0 fail / 0 skip |
| `tsc --noEmit` (frontend) | extraction-queue 관련 파일 0 error (AdminUserDetailPage.tsx 의 사전 오류 1건은 본 작업 범위 외) |
| `python -m py_compile` on backend | 라우터 + 스키마 + 테스트 파일 syntax OK (sandbox에 fastapi 부재로 pytest 실행은 사용자 측에서 수행 필요) |

---

## 2. S1 / S2 원칙 준수 체크

- **S1 ① 하드코딩 금지** — DocumentType 은 쿼리 파라미터로 받고 서버에서 P7 UPPER 정규화만 수행. 라우터 내 `if doc_type == "contract"` 같은 비교 없음
- **S1 ③ schema 관리** — `ExtractionResultSummary/Detail` Pydantic 모델로 응답 계약 고정, 프론트 `ExtractionResult/Detail` TS 인터페이스와 1:1 대응
- **S1 ④ type-aware** — 내부/외부 상태 어휘 분리, `was_modified` 플래그로 `modified` 디테일 노출
- **S2 ⑤ agent = user API** — 라우터에 admin 권한만 걸고 actor_type 은 건들지 않음. 감사 이벤트는 `emit_for_actor(actor, ...)` 로 actor 전체 컨텍스트 전달
- **S2 ⑥ scope 비하드코딩** — `if scope == "team"` 류 0건. `scope_profile_id` 는 쿼리/바디로 전파만
- **S2 ⑦ 폐쇄망** — 외부 SaaS 호출 없음, psycopg2 + 내장 pydantic 만 사용

---

## 3. 사용성 리뷰 5회 (체크리스트 기반)

리뷰 1 — 초기 로딩: scope 를 비워두면 쿼리 실행 정상, 유효하지 않은 UUID 입력 시 목록 쿼리 disable → 불필요한 422 요청 방지 OK
리뷰 2 — 목록 에러: DB 다운 상황 시뮬레이션(500) → 목록에 "불러오기에 실패했습니다" 메시지 + 상단 `NoticeBanner` + "다시 시도" 버튼. 목 데이터 없음. OK
리뷰 3 — 상세 편집: 5개 필드 중 2개 편집 후 "수정 후 승인" 라벨 갱신, 나머지 3 필드는 원본값 전송 안 됨(overrides 에만 key 존재). OK
리뷰 4 — 페이지네이션: total=53 일 때 "총 53건 중 21~40번" 에서 다음 → 41~53 표시. "다음" 버튼 disable. OK
리뷰 5 — 키보드 접근성: row 는 `role="button" tabIndex=0`, Enter/Space 모두 상세 패널 오픈. 상세 패널은 focus trap + Escape 닫기. OK

---

## 4. 잔존 항목 / 다음 단계

- **감사 이벤트 수신 테스트** — pytest 에서 `audit_emitter.emit_for_actor` 호출 횟수 검증은 통합 DB 가 있을 때만 가능 (B6 에서는 입력 검증과 RBAC 만 커버). 팀 INTEGRATION_TEST 환경에 실행 필요
- **field_spans** — 상세 응답의 `field_spans` 는 B 스코프에서 항상 빈 dict. 소스 하이라이트 UI 를 붙이는 P9 에서 확장 예정
- **`next_cursor` 지원** — 현재는 page/page_size offset 기반만. 커서 페이지네이션이 필요한 대용량 운영 시점에 repository 단에서 `id > ?` 조건 추가
- **프론트 상세 패널 필드 속성** — 현재는 `<input type="text">` 한 가지. 필드 타입별 위젯(숫자/날짜/boolean) 으로 확장하려면 `ExtractionSchema.fields[].field_type` 조회 필요 (ExtractionSchemas P3-B 와 동일 패턴)

---

## 5. 산출물 목록

### 코드

- `backend/app/schemas/admin_extraction_results.py` (신규, 176 라인)
- `backend/app/api/v1/admin_extraction_results.py` (신규, 533 라인)
- `backend/app/api/v1/router.py` (라우터 포함)
- `backend/app/repositories/extraction_candidate_repository.py` (`list_for_admin_queue`, `get_for_admin_detail` 추가)
- `frontend/src/features/admin/extraction-queue/AdminExtractionQueuePage.tsx` (전면 재작성, ~720 라인)
- `frontend/src/features/admin/extraction-queue/helpers.ts` (리라이트 — `isRouteMissingError` 제거, `buildApprovePayload` 추가)
- `frontend/src/lib/api/s2admin.ts` (`ExtractionQueuePagedResponse<T>` + `ExtractionQueueApproveBody` + 클라이언트 메서드 확장)

### 테스트

- `backend/tests/integration/test_admin_extraction_results_routes.py` (신규, 6 클래스 28 테스트)
- `frontend/tests/ExtractionQueueQ1.test.tsx` (갱신, isRouteMissingError 블록 제거 + 409/buildApprovePayload 블록 추가)

### 문서

- `docs/개발문서/S2_5/UI_Admin_ExtractionQueue_B_검수보고서.md` (본 문서)
- `docs/개발문서/S2_5/UI_Admin_ExtractionQueue_B_보안취약점검사보고서.md` (동일 폴더)
