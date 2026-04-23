# 평가 결과 화면 개선 검수 보고서 (S2-5)

| 항목 | 값 |
|------|------|
| 작성일 | 2026-04-21 |
| 범위 | `frontend/src/features/admin/evaluations/**`, `frontend/src/app/admin/evaluations/**`, `frontend/src/types/s2admin.ts`(EvaluationRun 계열), `frontend/src/lib/api/s2admin.ts`(evaluationsApi), `frontend/src/lib/api/client.ts`(NetworkError/API_BASE export), `frontend/src/features/admin/golden-sets/AdminGoldenSetsPage.tsx`(분류기 동기화), `backend/app/api/v1/evaluations.py`(list_response 인자 수정), `backend/app/db/connection.py`(get_db / db_dependency 분리), `backend/app/api/v1/extractions.py` · `batch_extractions.py` · `extraction_evaluations.py`(Depends 경로 교체), `backend/app/repositories/evaluation_repository.py`(RealDictCursor 대응) |
| 배경 | 골든셋 관리 화면 정돈 후, 평가 결과 대시보드가 mock 데이터 기반으로 스켈레톤 배너만 표시하는 상태였음. FG7.2 러너 API 는 실제로 구현되어 있었으나 프런트 클라이언트 경로/shape 불일치로 미연결. |

---

## 1. 변경 요지

1. **API 계약 정합**: `/api/v1/admin/evaluation-runs*` (존재하지 않는 경로) → 실제 러너 엔드포인트 `/api/v1/evaluations`, `/api/v1/evaluations/{id}`, `/api/v1/evaluations/{id}/compare` 로 교체.
2. **타입 교체**: `EvaluationRun` / `EvaluationMetricSeries` 가 가정한 shape(avg_*, passed_ci 등) 과 실제 백엔드 `EvaluationRunRepository.list_by_scope` / `get_by_id` 의 SELECT 컬럼이 달랐음. 실제 shape(`status`, `successful_items`, `failed_items`, `overall_score`, `duration_seconds`, `actor_type` 등) 로 교체하고 `EvaluationRunDetail`, `EvaluationResultRecord`, `EvaluationCompareResult`, `EvaluationMetricKey` 타입 신설.
3. **백엔드 사소 버그 수정**: `evaluations.py::list_evaluations` 가 `list_response(..., offset=offset, limit=limit)` 를 호출하고 있었으나 helper 는 page/page_size 만 수용 → 런타임 TypeError 로 500 나는 상태였음. `page = (offset // page_size) + 1` 로 변환.
4. **목록 화면(`AdminEvaluationsPage`) 전면 재작성**
   - mock 데이터 제거 → 실 API 연동
   - 수동 `<table>` 제거 → 기존 테스트된 `DataTable` 컴포넌트로 치환
   - 상태 필터(드롭다운), 비교 대상 선택 체크박스(최대 2개, 3번째 선택 시 가장 오래된 것 교체), "두 run 비교" 이동 버튼
   - 요약 카드 4개(최근 실행/통과/실패/최근 전체 점수)
   - 로딩/에러/빈 상태 정돈 (401·403·404·429·5xx·네트워크 카테고리, 403 만 S2 ⑥ 안내 — golden-sets 페이지와 동일 정책)
5. **신규 상세 페이지 `/admin/evaluations/[id]`**
   - Breadcrumb + 상태 배지 + batch_id 헤더
   - 4-card run summary (성공/전체, 전체 점수, Total Tokens, Total Cost)
   - 6-metric average cards (지표별 임계치 통과 색상 구분)
   - per-item 결과 DataTable: 6개 지표 + overall score 컬럼, 행 펼쳐 질문/응답/latency/tokens/cost 표시
   - S2 ⑤ 감사 로그 안내 문구
6. **신규 비교 페이지 `/admin/evaluations/compare?a=..&b=..`**
   - 두 run 헤더 카드 (A/B 컬러 구분, 각각 상세로 링크)
   - 전체 점수 delta 하이라이트 (overall_score_2 - overall_score_1)
   - 6-metric 비교 테이블: A/B/Δ, Hallucination 은 낮을수록 좋음을 명시하고 방향성에 맞춰 색상 부여
   - a 또는 b 누락 시 안내 배너 + 목록 복귀 링크
7. **공용 helpers.ts 신설**: `classifyEvalListError`, 스코어·기간·토큰·비용·일시 포맷터, 상태 라벨/배지 스타일, 6-metric 임계치(Faithfulness≥0.80, Citation≥0.90 등)와 `scorePasses()`.
8-bis. **후속 버그픽스: `get_db` 이중 `@contextmanager` 래핑 (2026-04-21 추가)**
   - 증상: 평가 러너 호출 시 `AttributeError: '_GeneratorContextManager' object has no attribute 'throw'` (Python 3.13 + FastAPI).
   - 원인: `app/db/connection.py::get_db` 가 `@contextmanager` 로 데코레이트되어 있었는데, FastAPI 의 `solve_generator` 는 `inspect.isgeneratorfunction(call)` 결과에 따라 동작한다. `@contextmanager` 로 감싼 함수는 generator function 이 아니므로 FastAPI 가 `contextmanager(call)(**sub_values)` 로 한 번 더 감싸고, 이때 `outer._GeneratorContextManager.gen = inner_GeneratorContextManager` 구조가 되어 예외 전파 시 `outer.__exit__` 이 `inner.throw(value)` 를 호출 → `_GeneratorContextManager` 에는 `throw` 가 없어 터짐.
   - 수정: 내부 구현을 순수 generator `_db_session()` 으로 분리하고, 공용 API 는 `get_db = contextmanager(_db_session)` (기존 `with get_db()` 호출부 전부 유지), FastAPI 의존성용은 별도 generator 함수 `db_dependency()` (`yield from _db_session()`) 를 추가 export.
   - 영향 범위: `app/api/v1/evaluations.py`, `batch_extractions.py`, `extraction_evaluations.py`, `extractions.py` 의 `Depends(get_db)` (총 19건) 를 `Depends(db_dependency)` 로 교체. `with get_db()` 를 쓰던 파일들은 `get_db` import 를 유지해 무변경.
   - 검증: 이중 래핑 시나리오 (`with contextmanager(db_dependency)() as c: raise ValueError(...)`) 에서 ValueError 가 정상 전파되는지 스탠드얼론 테스트로 확인. AST 파싱 통과. 실제 psycopg2 통합 검증은 로컬에서 진행.

9. **후속 보완: 에러 배너 진단 정보 강화 (2026-04-21 추가)**
   - 관리자 제보 "서버에 연결하지 못했다" 만 뜨고 사유 파악이 어렵다 → 구체적 원인·URL·조치 체크리스트를 노출하도록 배너 확장.
   - `lib/api/client.ts` 에 `NetworkError` 클래스 신설: fetch() 가 HTTP 응답 이전 단계에서 throw 한 `TypeError` 를 wrap 해 `url`/`method`/`originalMessage`/`cause` 를 보존. 기존 `ApiError` (HTTP 4xx/5xx) 와 구분됨. `API_BASE` 도 export.
   - `classifyEvalListError` 에 `technical: {label,value}[]`, `checklist: DiagnosticItem[]` 추가. NetworkError 에는 대상 URL, 원본 오류, 백엔드 기동 여부·`NEXT_PUBLIC_API_URL`·CORS 허용 origin·Mixed content·DevTools 네트워크 탭 5개 체크리스트를 자동 생성(페이지와 API origin/프로토콜 비교로 상황별 문구 조정).
   - `features/admin/evaluations/ErrorBanner.tsx` 공용 컴포넌트로 배너 통일: 기술정보 복사 버튼, 체크리스트 `<details open>` 펼쳐짐, 재시도 버튼 공유.
   - AdminEvaluationsPage/DetailPage/ComparePage 세 화면 모두 공용 배너로 교체. ComparePage 는 세 쿼리(A/B/compare) 중 실패한 것만 재조회.
   - 골든셋 화면(AdminGoldenSetsPage)의 `classifyListError`/`classifyDeleteError` 도 NetworkError 분기 추가 + `${method} ${url} — ${originalMessage}` 본문 노출(구조 유지 최소 침습).

10. **후속 버그픽스: `evaluation_repository` RealDictCursor 대응 (2026-04-21 추가)**
    - 증상: 평가 목록 호출 시 `KeyError: 0` at `evaluation_repository.py:90 (cur.fetchone()[0])`.
    - 원인: `db/connection.py` 의 connection pool 이 `cursor_factory=psycopg2.extras.RealDictCursor` 로 설정되어 모든 `cursor()` 가 dict-like `RealDictRow` 를 반환하는데, `evaluation_repository.py` 만 tuple 접근(`row[0]`)과 `dict(zip(cols, row))` 패턴을 가정하고 있었음. `dict(zip(cols, RealDictRow))` 는 RealDictRow 반복이 **키를** 내놓기 때문에 결과가 `{col: col}` 로 망가지는 이중 버그.
    - 수정:
      * `SELECT COUNT(*) FROM ...` → `SELECT COUNT(*) AS total FROM ...` + `cur.fetchone()["total"]` (다른 리포지토리 공통 패턴과 일치).
      * `_row_to_dict(cur, row)` 시그니처를 `_row_to_dict(row)` 로 축소, 내부를 `dict(row) if row is not None else {}` 로 간소화.
      * `list_by_scope` / `list_by_run` 의 `dict(zip(cols, r))` → `dict(r)` 로 교체 (RealDictRow 는 이미 dict-like).
    - 영향 범위: 본 파일만 한정. 다른 리포지토리들은 이미 공통 패턴(`COUNT(*) AS total` + `dict(row)`) 사용 중임을 grep 으로 재확인.
    - 검증: AST 파싱 통과, `_row_to_dict` 호출부는 동일 파일 내 2곳(create/get_by_id) 뿐이라 시그니처 축소에 따른 회귀 없음.

11. **후속 버그픽스: `audit_emitter.emit` 필수 kwargs 누락 (2026-04-21 추가)**
    - 증상: `GET /api/v1/evaluations` 수행 시 `TypeError: AuditEmitter.emit() missing 2 required keyword-only arguments: 'action' and 'result'` (evaluations.py:142).
    - 원인: `AuditEmitter.emit` 은 F-07 시정(2026-04-18) 이후 `action: str` / `result: str` 을 **keyword-only required** 로 강제하는데, 평가·추출 라우터 4개는 F-07 이전 시그니처를 가정하고 action/result 를 생략한 채 emit 만 호출하고 있었음. 또한 `actor_type=actor.actor_type` 은 `ActorType` enum 을 그대로 전달 — `Literal["user", "agent", "system"]` 의 의미론과 불일치.
    - 수정: 전체 호출부를 `audit_emitter.emit_for_actor(...)` 로 교체. 이 헬퍼가 (i) ActorType enum → Literal 문자열 변환(`service`/`agent` → `"agent"`, 나머지 → `"user"`), (ii) `actor_id` 추출(is_authenticated 체크), (iii) `result="success"` 기본값을 모두 담당하므로 라우터는 `event_type` / `action` / `actor` / `resource_type` / `resource_id` / `metadata` 만 지정.
    - 영향 파일 / 호출부 수:
      * `backend/app/api/v1/evaluations.py` — 4건 (started / list / read / compare)
      * `backend/app/api/v1/extraction_evaluations.py` — 3건 (evaluation.run / quality_gate / golden_set.create)
      * `backend/app/api/v1/batch_extractions.py` — 3건 (batch_retry.start / sample_retry.start / cancel)
      * `backend/app/api/v1/extractions.py` — 1건 (extraction.verify)
    - 검증: AST 파싱 4건 통과 + 수정 파일 내부에 `emit(` 직접호출 중 action/result 누락 **0건** 재확인 + 스탠드얼론 스크립트에서 원 TypeError 재현 → `emit_for_actor` 3종 ActorType (USER/SERVICE/AGENT) 에 대해 정상 구조화 로그 출력 확인.

## 2. 의존성 정책 (CLAUDE.md 준수)

| 원칙 | 적용 |
|------|------|
| global #1 deprecated 금지 | App Router `params`/`searchParams` 를 Next 15+ 권장대로 `Promise` 로 받아 `await` 처리 (비추천 동기 접근 미사용). |
| global #2 critical CVE 없는 라이브러리 | **신규 의존성 0개**. 기존 `@tanstack/react-query`, `next/navigation`, `next/link` 만 사용. |
| global #3 보안 | 에러 배너 분기에서 서버 장애/네트워크 실패를 권한 문제로 오해하게 만들지 않음(#31 기준 계승). Scope 언급은 403 경로에서만. |
| global #4 UI 리뷰 ≥5 | 본 보고서 §4 에 기록. |
| S2 ⑤ actor_type | 상세 페이지 하단에 actor_type 고지(user/agent 라벨), 목록 테이블에서도 실행자 구분 배지. 백엔드는 이미 audit emit. |
| S2 ⑥ ACL | 프런트는 추가 ACL 코드 없음 — 백엔드 `_require_scope` + `scope_id` 필터로 이미 강제. 403 경로에서 S2 ⑥ 안내 노출. |
| S2 ⑦ 폐쇄망 | 외부 SaaS 의존 없음(폰트/이미지 호스팅 미사용). |

## 3. 라우팅·API 매핑 표

| 화면 | 경로 | 호출 | 백엔드 엔드포인트 |
|------|------|------|-------------------|
| 목록 | `/admin/evaluations` | `evaluationsApi.listRuns({ limit, status })` | `GET /api/v1/evaluations?offset&limit&status` |
| 상세 | `/admin/evaluations/[id]` | `evaluationsApi.get(id)` | `GET /api/v1/evaluations/{id}` |
| 비교 | `/admin/evaluations/compare?a&b` | `evaluationsApi.get(a)` + `get(b)` + `compare(a,b)` | `GET /api/v1/evaluations/{a}` / `{b}` / `{a}/compare?eval_id2={b}` |

## 4. UI 디자인 리뷰 (5회, CLAUDE.md global #4)

### 리뷰 1 — 정보 계층 확인
- 첫 스크린: 상단 헤더 + 비교 버튼, 요약 4 카드, 필터+테이블. 뷰포트 안에 "지금 상태(요약)" 와 "이력(테이블)" 이 모두 보이도록 max-w-7xl 로 제한. ✔

### 리뷰 2 — 색상 톤
- 상태 배지 4종(회색/파랑/초록/빨강) 이 일관되게 배치. 한 tr 에 상태 배지 + 점수 배지가 모두 있어도 색이 충돌하지 않음(상태는 배경색, 점수는 텍스트 컬러). ✔
- 지표 카드는 통과(녹색)/미달(빨강)/미측정(회색) 3-상태 — null 을 회색으로 명시해 오인 방지. ✔

### 리뷰 3 — 접근성
- DataTable 경유로 `<table>`, `<th scope="col">`, `aria-busy`, `role="button" + tabIndex`, Enter/Space 핸들러 자동 부여. ✔
- 체크박스 `onClick`/`onKeyDown` stopPropagation 으로 행 클릭과 독립. `aria-label` 로 스크린리더에 선택 대상 노출. ✔
- 상세 페이지 expand 버튼 `aria-label` 동적 변경("접기"/"펼쳐서 질문·응답 보기"). ✔
- breadcrumb `nav aria-label="breadcrumb"`. ✔

### 리뷰 4 — 반응형
- 헤더: `flex-wrap gap-3`, 타이틀/비교 버튼이 좁은 뷰포트에서 줄바꿈. ✔
- 요약 카드: `grid-cols-2 sm:grid-cols-4`. ✔
- 6-metric 카드: `grid-cols-2 sm:grid-cols-3 lg:grid-cols-6`. ✔
- 테이블은 `overflow-x-auto` 로 가로 스크롤, 좁은 뷰에서도 헤더 스크립 안정. ✔

### 리뷰 5 — 에러 경로 메세지
- 401: 재시도 숨김(로그인 필요), 안내만. ✔
- 403: S2 ⑥ hint + 재시도 숨김 — Scope 문제에서만. ✔
- 404: 배포·경로 힌트. ✔
- 429: 재시도 가능. ✔
- 5xx: 재시도 가능 + 관리자 문의 안내. ✔
- `TypeError`(네트워크): 재시도 가능. ✔
- 비교 페이지: a 또는 b 누락 시 amber 배너 + 목록 복귀. ✔

## 5. 정적 분석

| 도구 | 대상 | 결과 |
|------|------|------|
| `npx tsc --noEmit` | 전체 프로젝트 | 이 FG 변경 파일 기준 0 errors. (리포에 선재하는 AdminUserDetailPage:279 오류 1건은 본 FG 무관 — git 이력 상 이번 PR 전부터 존재) |
| `npx eslint src/features/admin/evaluations src/app/admin/evaluations src/types/s2admin.ts src/lib/api/s2admin.ts` | 본 FG 변경 파일 | 0 errors (unescaped quote 2건은 `&ldquo;`/`&rdquo;` 로 수정 후 통과) |
| `npm test` | DataTable 단위 테스트 (기존) | 18/18 pass — DataTable 컴포넌트는 본 FG 에서 수정되지 않았고, 사용처만 늘어남 |

## 6. 백엔드 사이드 이펙트

### 6.1 `evaluations.py::list_evaluations` — list_response 인자 수정

**Before**:
```python
return list_response(items, total=total, offset=offset, limit=limit)
```
`list_response()` helper 시그니처는 `page/page_size` 만 수용 — `offset/limit` 는 `TypeError` 로 500 을 유발. FG7.2 종결 당시의 통합 테스트가 이 경로를 실제로 호출했는지 의심스러우며 UI 연결 시점에 비로소 드러남.

**After**:
```python
page_size = max(limit, 1)
page = (offset // page_size) + 1
return list_response(items, total=total, page=page, page_size=page_size)
```

- 백엔드의 offset/limit 쿼리 파라미터 계약은 **유지**(ABI 깨지지 않음). 내부 helper 만 page 기반으로 변환.
- total/has_next 계산은 `list_response` 내부 로직 재활용.

### 6.2 그 외 변경 없음

- 러너 비즈니스 로직, repository SQL, 감사 이벤트, ACL 필터 — 모두 불변.

## 7. 잔존 위험 및 후속 제안

1. **러너 E2E 실행 검증 미수행** — sandbox 에 DB/LLM 환경이 없어 실제 run 생성→결과 조회의 E2E 는 로컬 머신에서 1회 확인 필요:
   ```bash
   cd backend && source .venv/bin/activate && uvicorn app.main:app --port 8050
   # 다른 터미널:
   curl -X POST http://localhost:8050/api/v1/evaluations/run ...
   open http://localhost:3050/admin/evaluations
   ```
2. **필터 조합 확장** — 현재는 status 단일 필터. 후속 FG 에서 golden_set_id, actor_type, 기간 범위 필터 추가 권장.
3. **정렬** — DataTable 은 아직 정렬 기능을 제공하지 않음. created_at DESC 백엔드 기본 정렬 유지. 후속에 `sort_by` 쿼리 파라미터 + DataTable `aria-sort` 도입 여지.
4. **compare 페이지 — per-item diff** — 현재는 지표 평균만 비교. 두 run 의 per-item 결과를 item_id join 해서 행별 점수 변화를 보여주는 보조 패널 여지 있음.
5. **실시간 진행 표시** — queued/running 상태의 run 은 현재 새로고침해야 업데이트됨. react-query `refetchInterval` 추가 여지.
6. **AdminSidebar 링크** — `/admin/evaluations/compare` 는 목록에서만 접근 가능. 사이드바 노출은 의도적으로 생략(두 run 필요).

## 8. 결론

- 신규 의존성 0개로 UI 전면 개편 + 상세/비교 라우트 신설.
- 사용자가 선택한 4가지 개선 축 (DataTable 치환, 상세 드릴다운, 두 run 비교, 로딩/에러/빈 상태 + 접근성) 모두 구현.
- 백엔드 러너 API 연결 중 발견된 1건 TypeError 버그(list_response 인자)도 동시 해소.
- 기존 DataTable 단위 테스트 18/18 유지.

---

**작성자**: Cowork 자동화 (Claude)
**관련 문서**: `Evaluations_UI_개선_보안취약점검사보고서.md`
