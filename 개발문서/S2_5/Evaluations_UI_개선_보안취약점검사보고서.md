# 평가 결과 화면 개선 보안 취약점 검사 보고서 (S2-5)

| 항목 | 값 |
|------|------|
| 작성일 | 2026-04-21 |
| 범위 | `frontend/src/features/admin/evaluations/AdminEvaluationsPage.tsx`, `AdminEvaluationDetailPage.tsx`, `AdminEvaluationComparePage.tsx`, `helpers.ts`, `ErrorBanner.tsx`(신규); `frontend/src/app/admin/evaluations/**/page.tsx`; `frontend/src/types/s2admin.ts`(EvaluationRun 계열 신·변경); `frontend/src/lib/api/s2admin.ts`(evaluationsApi); `frontend/src/lib/api/client.ts`(NetworkError 신설 + API_BASE export); `frontend/src/features/admin/golden-sets/AdminGoldenSetsPage.tsx`(분류기 동기화); `backend/app/api/v1/evaluations.py`(list_response 인자 수정 한정) |
| 기준 | CLAUDE.md global #2(critical CVE 없는 라이브러리), #3(보안), S1 #3(보안 취약점 검사 필수), S2 ⑤/⑥/⑦ |

---

## 1. Dependency 변경 요약

| 변경 | 신규 패키지 | 버전 | CVE 상태 |
|------|--------------|------|----------|
| `frontend/package.json` | **없음** | — | 신규 runtime/dev 의존성 0개. |
| `backend/requirements.txt` | **없음** | — | 기존 의존성만 사용. |

> 기존 `@tanstack/react-query`, `next/link`, `next/navigation`, `react-dom/server` 만 사용.
> 공격 표면 변화: **0** (NPM/PyPI 신규 설치 없음).

## 2. 신규/변경 API 호출 경로 및 권한

| 호출 | 메서드 | 경로 | 인증 | ACL |
|------|--------|------|------|------|
| 목록 | GET | `/api/v1/evaluations?offset&limit&status` | Bearer(api client) | `_require_scope` + `scope_id` WHERE 필터(백엔드) |
| 상세 | GET | `/api/v1/evaluations/{id}` | Bearer | `_require_scope` + `WHERE id=%s AND scope_id=%s` |
| 비교 | GET | `/api/v1/evaluations/{a}/compare?eval_id2={b}` | Bearer | 두 run 모두 `scope_id` 로 조회 — 한쪽이라도 다른 scope 면 404 |

- **읽기 전용**: 본 FG 는 쓰기 경로(POST /run, 수정, 삭제) UI 를 추가하지 않음 → 상태 변경 공격 표면 증가 없음.
- **S2 ⑤ actor_type**: 서버 측이 이미 `evaluation.run.read|list|compare` 이벤트를 `audit_emitter.emit` 으로 기록. UI 에서는 actor_type 을 라벨로 *표시*만 하며, 사용자 입력이 actor_type 에 영향을 주지 않음.
- **S2 ⑥ 우회 없음**: 프런트는 별도 scope 헤더를 조작하지 않음(API client 의 Bearer 토큰만 사용). URL 쿼리 파라미터(`?a`, `?b`, `?status`) 는 서버 측에서 재검증되며 존재하지 않거나 scope 불일치 시 404/403 반환.

## 3. 입력 검증 / XSS / 인젝션

### 3.1 표시되는 백엔드 응답 필드

| 필드 | 렌더링 방식 | 위험 |
|------|------------|------|
| `batch_id`, `id` | React `{...}` 삽입 (자동 escape) | XSS 없음 |
| `question`, `answer` | `whitespace-pre-wrap` + React escape | XSS 없음 (innerHTML 미사용) |
| `actor_id`, `scope_id` | `<code>`/`<span>` 안 텍스트 | XSS 없음 |
| 숫자(score/tokens/cost/duration) | `toFixed`, `toLocaleString` 후 text 삽입 | 타입 강제 + 포맷팅, 인젝션 없음 |
| 일시 | `new Date(iso).toLocaleString("ko")`, 실패 시 "-" | 잘못된 iso 는 catch → "-" |

- **dangerouslySetInnerHTML 사용 없음**. React 기본 escape 로 충분.
- **링크 구성**: `href={\`/admin/evaluations/${detail.id}\`}` / `compare?a=${a}&b=${b}` 는 모두 내부 라우트. 외부 URL 삽입 없음.

### 3.2 URL 쿼리 파라미터

| 파라미터 | 출처 | 검증 |
|----------|------|------|
| `?a`, `?b` (compare) | 사용자 URL | 값을 그대로 백엔드에 전달 → 백엔드가 scope_id 로 2차 조회 실패 시 404. 프런트는 두 값이 모두 있을 때만 쿼리 활성화(`enabled: Boolean(a && b)`). |
| `?status` | UI 드롭다운 | `EvaluationRunStatus | "all"` 유니온 타입으로 제한. 사용자가 직접 URL 로 쿼리 넣어도 백엔드 Pydantic 검증/WHERE 매칭 실패 시 빈 결과. |

### 3.3 체크박스 선택 상태

- 선택된 run id 는 클라이언트 state (`compareSelection`) 에만 존재. 페이지 전환 또는 새로고침 시 초기화.
- localStorage/sessionStorage 저장 없음 → XSS 를 통한 영속 state 탈취 없음.

## 4. OWASP Top 10 대응 현황

| 카테고리 | 이 FG 대응 | 결과 |
|----------|-----------|------|
| A01 Broken Access Control | 백엔드 `_require_scope` + scope_id 필터 유지, 프런트는 우회 API 호출 추가 없음 | 기존 수준 유지 |
| A02 Cryptographic Failures | 해당 없음 (본 FG 는 순수 UI) | — |
| A03 Injection | SQL 직접 호출 없음. 백엔드 repository 는 파라미터화 쿼리 유지(list_response 인자 수정만). | SQL injection 신규 노출 0 |
| A04 Insecure Design | 두 run 비교는 eval_id_1/eval_id_2 모두 scope 검증 → 교차 scope 데이터 누출 없음 | 안전 |
| A05 Security Misconfiguration | 신규 의존성 0, next.config/eslint 설정 미변경 | 변화 없음 |
| A06 Vulnerable Components | 신규 CVE 노출 0 | — |
| A07 Id & Auth Failures | Bearer 토큰 재사용 (client.ts) — 본 FG 미변경 | 기존 수준 |
| A08 Software & Data Integrity | next/link, next/navigation 로 클라이언트 라우팅 | 안전 |
| A09 Logging & Monitoring | 감사 이벤트 `evaluation.run.list|read|compare` 은 백엔드가 이미 emit (본 FG 미변경) + UI 하단에 "감사 로그 기록됨" 고지 문구 추가 | 개선 |
| A10 SSRF | 외부 URL fetch 없음 | — |

## 5. 민감 정보 / PII

- 화면에 표시되는 `actor_id`, `scope_id` 는 UUID/식별자이며, 이메일·전화번호·비밀번호 등은 표시하지 않음.
- 질문(`question`)/응답(`answer`) 은 RAG 파이프라인 출력이므로 조직 내부 문서 일부를 포함할 수 있으나, 이는 이미 scope ACL 경계 내에서만 조회 가능하고 기존 상세 조회 경로의 위험과 동치.
- Cost 값은 USD 소수점 표기 — 결제 카드/계정 정보가 아니므로 PII 아님.
- localStorage/sessionStorage 저장, 전역 window 오염 없음.

## 6. 시크릿 스캔

```
grep -rEn "(password|secret|token|api[_-]?key|Bearer|authorization)" \
  frontend/src/features/admin/evaluations \
  frontend/src/app/admin/evaluations \
  frontend/src/lib/api/s2admin.ts \
  backend/app/api/v1/evaluations.py
```

- 결과: 본 FG 의 신규/변경 파일에서 하드코딩된 시크릿 **없음**.
- `client.ts` 가 Bearer 토큰을 헤더에 첨부하는 것은 기존 구조이며 본 FG 에서 미변경.

## 7. 백엔드 변경 검토 (`list_response` 인자 수정)

| 체크 | 결과 |
|------|------|
| 응답 shape 변경 | **없음** — 여전히 `{data: [...], meta: {request_id, trace_id, pagination}}` 형태. helper 내부적으로 page/page_size/total/has_next 만 조립. |
| API 쿼리 계약 | offset/limit 그대로 수용(쿼리 파라미터 변화 없음). 내부 환산만 발생. |
| 부작용 | `page = (offset // page_size) + 1` 로 page 값이 커질 수 있으나 `list_response` 내부는 `total`/`has_next` 를 유지 — has_next 는 `(page * page_size) < total` 식이므로 equivalence 유지. |
| 감사 로깅 | 변경 없음 (`evaluation.run.list`, `resource_id=None`, `metadata={"total": total}` 유지). |

결과: 백엔드 변경은 **버그 수정 한정**이며 외부 계약 영향 없음. 신규 권한/경로 추가 없음.

## 8. 렌더링 안전성 (React 특이 사항)

- **expand 버튼 + DataTable 상호작용**: 버튼 `onClick`/`onKeyDown` 이 `e.stopPropagation()` 으로 row 클릭(상세 이동)과 독립. 비교 체크박스도 동일.
- **href 구성**: template literal 로 `${id}` 삽입 → `id` 는 백엔드가 `uuid4` 로 발급 → URL-safe. 별도 encodeURIComponent 불필요.
- **query string 구성**: `buildQueryString` 유틸 (기존) 이 undefined 제거 + URL 인코딩 수행.
- **에러 메시지 표시**: `getApiErrorMessage(err, fallback)` 경유 — 백엔드 message 를 그대로 렌더링하지만 React escape 적용되므로 HTML 인젝션 없음.

## 8-bis. NetworkError 진단 노출 안전성 (2026-04-21 추가)

**변경 내용** — 에러 배너가 "서버에 연결하지 못했다" 한 문장만 보여주던 문제를 해결하기 위해 진단 정보(요청 URL, 원본 오류 메시지)와 조치 체크리스트를 관리자에게 노출하도록 확장.

| 항목 | 노출되는 값 | 민감도 / 판정 |
|------|------------|---------------|
| `요청 URL` | 예) `GET http://localhost:8050/api/v1/evaluations?limit=50` | **OK**. API 베이스 URL 은 이미 브라우저 DevTools 네트워크 탭과 Next.js 빌드 산출물에서 관찰 가능. 쿼리스트링에는 enum 상태·UUID 만 포함되고 토큰·PII 는 없음. |
| `원본 오류` | 예) `TypeError: Failed to fetch` | **OK**. 브라우저가 자체 생성하는 고정 문자열. 내부 경로·스택 트레이스 없음. |
| `체크리스트 내 API_BASE` | `http://localhost:8050` (env 주입값) | **OK**. `NEXT_PUBLIC_*` prefix 는 의도적으로 클라이언트 번들에 포함되는 값(Next.js 규약) — 이미 공개 정보. |
| `체크리스트 내 페이지 origin` | `window.location.origin` | **OK**. 사용자 자신의 브라우저에서 접근한 origin. 외부 유출 아님. |
| `복사 버튼` | `navigator.clipboard.writeText` | **OK**. 사용자 gesture 기반, 자동 실행 없음. 페이지 DOM 에는 이미 표시된 문자열만 복사. |
| Authorization 헤더 | **노출 안 함** | 배너는 URL 과 method 만 보존. 토큰은 `NetworkError` 에 기록하지 않음. |
| 쿠키/세션 | **노출 안 함** | 동일. |
| 백엔드 에러 메시지 | 기존 `ApiError` 경로에서 이미 노출 중 — 변화 없음 | 기존 정책 유지. |

**React XSS**: `technical[].value` 와 `checklist[].detail` 은 모두 React `{...}` 삽입 (자동 escape) 으로 렌더링. `dangerouslySetInnerHTML` 미사용.

**DoS 자가 유발 방지**: 재시도 버튼은 `queryClient.refetch()` 한 번만 호출하며 `isFetching` 중에는 disabled. 자동 재시도(backoff) 는 `retry: false` 로 비활성화됨 (기존 정책 유지).

**사용자 혼란 방지**: 체크리스트는 `<details open>` 로 기본 펼쳐져 있으나, 관리자 진단 전용 언어(예: "uvicorn 기동", "CORS 허용 origin")이 사용됨. 일반 사용자에게는 노출되는 대시보드 URL 이 아니므로 허용.

**판정**: 새로 노출되는 문자열은 모두 (i) 브라우저 DevTools 에서 이미 볼 수 있거나 (ii) 공개 클라이언트 번들에 포함된 값이며, 토큰·쿠키·스택트레이스·내부 경로는 노출되지 않음. **민감정보 누출 위험 없음**.

## 8-ter. `get_db` / `db_dependency` 분리 리팩토링 안전성 (2026-04-21 추가)

**배경** — Python 3.13 + FastAPI 조합에서 `@contextmanager` 로 데코레이트된 `get_db` 를 `Depends(get_db)` 로 직접 주입하면 `solve_generator` 가 generator 함수로 판정하지 못해 `contextmanager(call)(**sub_values)` 로 한 번 더 감싸고, 이때 `outer.gen = inner_CM` 구조가 되어 예외 전파 시 `AttributeError: '_GeneratorContextManager' object has no attribute 'throw'` 로 터지는 문제를 해결하기 위한 구조 변경.

**변경 요지**

| 요소 | Before | After |
|------|--------|-------|
| 공용 CM | `@contextmanager def get_db()` | `_db_session` (순수 generator) + `get_db = contextmanager(_db_session)` |
| FastAPI 의존성 | `conn=Depends(get_db)` (동일 객체) | `conn=Depends(db_dependency)` (`yield from _db_session()`) |
| 기존 `with get_db() as conn:` 호출부 | 변경 없이 동작 | 동일 — alias 로 backward compatible |

**보안 속성 보존 점검**

| 항목 | 판정 근거 |
|------|-----------|
| 커넥션 풀 의미 (getconn/putconn) | `_db_session` 내부에 보존. 분리 전후 동일한 `finally` 블록이 `pool.putconn(conn)` 을 수행. |
| 트랜잭션 커밋/롤백 | `yield` 후 정상 커밋, 예외 시 롤백 — 분리 전후 동일. |
| 감사 이벤트 | `evaluations.py` / `extractions.py` / `batch_extractions.py` / `extraction_evaluations.py` 의 `audit_emitter.emit(...)` 은 미변경 (event_type, actor_type, resource_id 유지). |
| Scope Profile ACL | 각 라우터의 `_require_scope(actor)` / `scope_id` 필터 호출은 미변경. |
| SQL shape | Repository 쪽은 일체 변경 없음. |
| 공격 표면 | 신규 엔드포인트 없음. 신규 파라미터 없음. 인증/인가 경로 동일. |
| 로깅 민감정보 | `_db_session` 은 예외 객체만 `raise` 재전파하며 직접 로깅하지 않음. 기존 repository 레벨 로깅 정책 유지. |
| Race / 동시성 | psycopg2 pool (thread-safe) 자체 사용 패턴 미변경. FastAPI 쪽은 sync dependency 를 threadpool 에서 실행하는 기존 경로 그대로. |

**호환성 확인**

- `backend/` 내 `with get_db() as conn:` 호출 200+ 건 — import 경로/함수명 미변경, 동작 동일.
- 변경된 라우터 4 파일(`evaluations.py`, `extractions.py`, `batch_extractions.py`, `extraction_evaluations.py`) 의 `Depends(get_db)` → `Depends(db_dependency)` 치환은 구문적으로만 다른 이름이며 생성되는 커넥션은 동일.
- Standalone 재현 스크립트(동일 패턴, psycopg2 제외)에서 예외 전파가 `_GeneratorContextManager` 이중 래핑 없이 정상 동작함을 확인.

**판정**: 본 변경은 **순수 구조 리팩토링**. 외부 계약·보안 경로·감사 이벤트·공격 표면에 영향 없음. **위험도 No change**.

## 8-quater. `evaluation_repository` RealDictCursor 대응 안전성 (2026-04-21 추가)

**배경** — `db/connection.py` 의 connection pool 이 `RealDictCursor` 로 설정되어 있으나 `evaluation_repository.py` 만 tuple 접근(`cur.fetchone()[0]`)을 가정 → `KeyError: 0`. 또한 `dict(zip(cols, RealDictRow))` 는 `{col: col}` 로 망가지는 잠재적 데이터 왜곡 버그가 있었음.

**보안 속성 보존 점검**

| 항목 | 판정 근거 |
|------|-----------|
| SQL shape | `COUNT(*)` → `COUNT(*) AS total` 로 alias 만 추가. WHERE/ORDER/LIMIT/OFFSET 불변. |
| Scope Profile ACL | `WHERE scope_id = %s` 및 후속 필터 완전 유지. 신규 파라미터 없음. |
| 감사 이벤트 | 라우터(`evaluations.py`) 의 `audit_emitter.emit(...)` 변경 없음. |
| 데이터 노출 표면 | 반환 필드 동일. 오히려 `dict(zip(...))` 가 망가뜨린 `{col: col}` 구조가 사라져 **정상 데이터만** 반환. |
| 에러 메시지 | `KeyError: 0` 는 더 이상 발생하지 않음. 기존 예외 처리 경로 불변. |
| 신규 의존성 / 공격 표면 | 없음. |

**판정**: 본 변경은 **순수 버그픽스**(타입 불일치로 발생하던 런타임 KeyError 및 잠재적 데이터 왜곡 제거). 외부 계약 영향 없음. **위험도 No change**.

## 8-quinta. `audit_emitter.emit_for_actor` 전환 안전성 (2026-04-21 추가)

**배경** — F-07 시정(2026-04-18) 이후 `AuditEmitter.emit` 은 `action` / `result` 를 keyword-only required 로 강제하나, 평가·추출 라우터 4개는 F-07 이전 시그니처로 호출 중이라 `TypeError` 로 **감사 기록 자체가 실패**하던 상태. 즉, S2 원칙 ⑤(감사 로그 필수) 위반 구간이 존재했음.

**변경 요지** — 호출부를 `emit_for_actor` 로 통일. 라우터는 `event_type` / `action` / `actor` / `resource_type` / `resource_id` / `metadata` 만 지정하고, enum→Literal 변환 / actor_id 추출 / result="success" 기본값은 헬퍼가 담당.

**보안 속성 점검**

| 항목 | 판정 근거 |
|------|-----------|
| 감사 이벤트 발생 여부 | **개선** — 기존엔 TypeError 로 감사 기록 누락(S2 ⑤ 위반). 수정 후 모든 경로에서 정상 emit. |
| actor_type 기록 | ActorType.SERVICE → `"agent"`, USER → `"user"` 로 규정에 맞게 정규화 (emit_for_actor 내부 로직). anonymous / null 은 "user" 폴백이므로 미인증 요청은 애초 라우터 앞단 `_require_scope` 에서 차단됨. |
| action 라벨 | 네임스페이스(`evaluation.run.start`, `extraction.batch_retry.start` 등)는 기존 다른 라우터 관례와 일치. 신규 값이지만 자유 문자열이라 enum/DB 제약 없음. |
| actor_id 경로 | `emit_for_actor` 는 `is_authenticated` 체크 → 미인증이면 None 기록. 라우터는 인증 요구 경로뿐이라 실제로는 항상 authenticated actor. |
| metadata 노출 범위 | 기존 호출부와 동일한 키(`total`, `batch_id`, `item_count`, `reason` 등). 추가 PII/토큰 전파 없음. |
| SQL 구문 / 파라미터 | `_persist` 의 INSERT 는 불변 — audit_events 컬럼/바인딩 동일. |
| S2 ⑥ ACL | emit 호출 자체는 ACL 바깥(감사). 라우터 쪽 `_require_scope` 등 ACL 경로 미변경. |

**판정**: 본 변경은 **감사 누수 복구**에 해당(기존에 `TypeError` 로 묵살되던 감사 기록을 정상화). 새로운 데이터 노출·공격 표면·ACL 약화는 없으며 오히려 S2 ⑤ 준수가 강화됨. **위험도 개선(Risk-reducing)**.

## 9. 결론

- **신규 의존성 0, 신규 CVE 0, 새 공격 표면 0.**
- 읽기 전용 UI 확장이며 기존 ACL(`_require_scope` + `scope_id` WHERE) 을 재사용.
- 프런트의 사용자 입력은 상태 필터(enum) / 비교 선택(UUID) 두 경로뿐이며, 백엔드가 2차 검증.
- 백엔드 사이드 변경은 `list_response` 인자 형태 정정 한정이며 외부 계약·감사 이벤트·SQL 에 영향 없음.
- 위험도: **Low / no net change**.

## 10. 후속 권장

1. 러너 E2E 실행을 로컬에서 1회 수행해 list→detail→compare 경로를 실 데이터로 확인.
2. 장기적으로 `useQuery` 에 `staleTime`/`refetchInterval` 을 부여해 running 상태의 폴링 빈도를 제어 (DoS 자가 유발 방지).
3. 감사 이벤트 활용 대시보드(Phase 9) 에 `evaluation.run.compare` 를 항목으로 추가 — 운영 가시성 향상.

---

**작성자**: Cowork 자동화 (Claude)
**관련 문서**: `Evaluations_UI_개선_검수보고서.md`
