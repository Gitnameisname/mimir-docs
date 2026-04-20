# AdminGoldenSetsPage 와이어링 보안 취약점 검사 보고서

- 작성일: 2026-04-20
- 대상: `/admin/golden-sets` 화면 및 서버측 연계 경로 변경분
  - Frontend: `frontend/src/features/admin/golden-sets/AdminGoldenSetsPage.tsx`, `frontend/src/lib/api/s2admin.ts`
  - Backend: `backend/app/models/golden_set.py`(필드 추가), `backend/app/api/v1/golden_sets.py`(미변경; 계약 검증 대상), Alembic(`users.scope_profile_id`)
- 목적: 이번 와이어링 PR 이 인증/인가, ACL(S2 ⑥), XSS, CSRF, PII, 의존성 취약점, 파일 업로드, 감사 로그, 폐쇄망 제약에 미친 영향을 평가
- 연계: `docs/개발문서/S2_5/UI_Admin_GoldenSets_검수보고서.md`, `docs/개발문서/S2_5/UI_Admin_GoldenSets_런타임회귀리뷰.md`

---

## 1. 변경 요약 (공격 표면 관점)

| 변경 | 공격 표면 변화 |
| --- | --- |
| `AdminGoldenSetsPage` 가 실제 `/api/v1/golden-sets*` 호출 | **신규** — 다만 기존 `goldenSetsApi` 가 이미 사용되던 계약과 동일. 서버 측은 `_require_auth / _require_scope / _require_write` 로 보호 |
| API 경로 `/admin/golden-sets` → `/golden-sets` 치환(#28) | **접근 경로 변경만**; 권한 판정은 `scope_profile_id` 기반이므로 등가 |
| `users.scope_profile_id` Alembic 도입(#29) | **ACL 활성화** — 이전에는 미바인딩 계정이 403 을 받았고, 이제는 바인딩되어 자기 스코프 안에서만 목록·CRUD 가능 |
| `GoldenSet.item_count` 필드 추가(#30) | **없음** — 쓰기 경로/권한/PII 영향 없음. 내부 집계값의 응답 직렬화만 수정 |
| 파일 업로드 UI(JSON import) | **기존 계약 유지** — 서버가 `.json` + `application/json \| text/json \| text/plain` MIME 만 수락, 프런트는 파일 선택 FormData 로만 전송 |
| JSON export 다운로드 (`downloadJsonFile`) | Blob URL 로 브라우저 메모리에서만 처리, 외부 송신 없음 |

---

## 2. OWASP Top 10 (2021) 매핑

### A01 Broken Access Control — 핵심 검토

**S2 ⑥ Scope Profile ACL** 이 이번 기능의 1차 방어선이다.

- `backend/app/api/v1/golden_sets.py` 의 모든 엔드포인트 선두에서 `_require_auth(actor)` → `_require_scope(actor)` 호출. 후자는 `actor.scope_profile_id` 부재 시 **403** 으로 차단.
- Repository `list_by_scope`, `get_by_id`, `update`, `soft_delete` 등 모두 `scope_id=%s` WHERE 절을 강제 — 타 scope 의 row 노출 원천 차단.
- 쓰기 엔드포인트는 `_require_write(actor)` 로 `ORG_ADMIN / SUPER_ADMIN / AUTHOR / REVIEWER / APPROVER` role 만 허용.
- UI 레벨에서 row 를 클릭하면 `setSelectedId(id)` 후 `GET /api/v1/golden-sets/{id}` 가 호출되며, **id 만 안다고** 타 scope row 를 가져올 수 없음 (서버 WHERE scope_id=%s).
- 경로 치환(#28)은 URL prefix 만 바뀌었을 뿐 인가 결정은 actor.scope_profile_id 기반이므로 권한 우회 경로 생성 없음.

**수평 권한 상승(IDOR) 테스트 고려**: 사용자가 다른 사용자의 `golden_set_id` 를 URL 직조립으로 들어가도 서버가 `WHERE id=%s AND scope_id=%s` 로 조회해 `None` → 404 로 응답. UI 에서는 `아직 생성된 골든셋이 없습니다` 메시지를 보여주는 경로와 동일하게 노출되어 정보 누출 없음.

**결론: A01 영향 없음(오히려 #29 로 강화).**

### A02 Cryptographic Failures

- 본 PR 에는 키·토큰·비밀번호 다루는 경로 없음.
- 인증 쿠키(`cowork_session`)는 HttpOnly + SameSite=Lax 가 기존 설정(세션 미들웨어)에서 유지. 본 PR 에서 쿠키 속성 변경 없음.
- **영향 없음.**

### A03 Injection (XSS / SQLi 포함)

- **XSS**: 신규 렌더는 모두 JSX 표현식 바인딩 (`{s.name}`, `aria-label={`${s.name} 골든셋 상세 열기`}`). React 의 기본 이스케이프로 보호.
  - `dangerouslySetInnerHTML` 도입 없음.
  - 도메인/상태 라벨은 서버 값이 아니라 프런트 상수 `DOMAIN_LABELS/STATUS_LABELS` 에서 룩업 — 사용자 입력이 UI 제어 문자열로 쓰이지 않음.
- **SQLi**: 백엔드 repository 는 전부 `%s` 파라미터 바인딩 (`cur.execute(sql, params)`) — 문자열 포맷팅 SQL 주입 없음. `f"{where}"` 로 조립되는 부분은 필터 플래그 추가(`domain=%s`, `status=%s`) 뿐이고 값 자체는 다음 자리 params 로 전달.
- **JSON import**: 업로드 파일은 `GoldenSetImportExportService` 가 파싱. UI 는 파일 객체만 FormData 로 전달하고 내용에 손대지 않음.
- **결론: A03 영향 없음.**

### A04 Insecure Design

- CRUD 작업 전에 프런트 + 서버 이중 검증(길이 제약, 필수 필드) 적용.
- Soft delete 로 영구 삭제 경로 차단(`is_deleted=TRUE`). 복구 가능성 유지.
- 모달 `role="dialog" + aria-modal="true"`, Esc 닫기, 포커스 자동화 — 키보드 사용자에게도 동일한 확인 단계 강제.
- **결론: 영향 없음(오히려 설계 견고화).**

### A05 Security Misconfiguration

- CSP/CORS/서버 헤더 변경 없음.
- `s2admin.ts` 의 API 헬퍼(`api.*`)는 기존 fetch 래퍼 재사용 — 새 엔드포인트에서 임의 헤더 노출 없음.
- 런타임 회귀 R7 에서 응답 헤더에 민감 토큰 미노출 확인.
- **결론: 영향 없음.**

### A06 Vulnerable & Outdated Components

- 신규 의존성 추가 없음. `package.json`, `requirements.txt` 변경 없음.
- 이미 사용 중인 패키지 계보:
  - Frontend: `next@^16.2.4`, `react@19.x`, `@tanstack/react-query@^5`, `zod@^4.3.6`, `dompurify@^3.4.0`
  - Backend: `fastapi`, `pydantic@>=2`, `psycopg2`, `alembic` (이번 S2-5 도입)
- 위 모두 최신 안정 라인 기준 critical CVE 미해당(2026-04-20 시점 본 리뷰의 관찰 범위). `package.json` 변경이 없으므로 본 PR 은 공격 표면 추가 없음.
- **결론: 영향 없음.**

### A07 Identification & Authentication Failures

- 인증 결정은 여전히 `resolve_current_actor(request) → ActorContext` 에서 수행. 본 PR 은 `scope_profile_id` 를 ActorContext 에 추가(#29) 한 것 외에 인증 경로 변경 없음.
- Login/비번/OAuth/세션 라이프사이클 변경 없음.
- **결론: 영향 없음.**

### A08 Software & Data Integrity Failures

- JSON import/export 경로는 서버가 스키마 검증(Pydantic `GoldenSetImportRequest`) 을 수행하며, 파일 무결성은 요청 기준 업로드 시 처리. SRI/패키지 서명 관련 변경 없음.
- **결론: 영향 없음.**

### A09 Security Logging & Monitoring Failures

- 쓰기 엔드포인트(`create`, `update`, `delete`, `add_golden_item`, ...)는 전부 `audit_emitter.emit(... actor_type=_actor_type_str(actor) ...)` 로 **S2 ⑤** 감사 로그를 남긴다.
- `_actor_type_str(actor)` 는 `actor.actor_type.value` 또는 기본값 `"user"` → `user/agent` 구분을 정확히 기록.
- 본 PR 은 기존 감사 경로 그대로 사용하고, 에미터 래핑을 변경하지 않았다.
- **결론: 영향 없음(S2 ⑤ 유지).**

### A10 Server-Side Request Forgery

- 서버 측에서 사용자 입력으로 외부 URL fetch 하는 경로 없음.
- Import 는 업로드된 JSON 바이트만 파싱 — 외부 URL 팔로우 없음.
- **결론: 영향 없음.**

---

## 3. 보조 체크리스트

### 3.1 PII / 민감정보

| 항목 | 검토 |
| --- | --- |
| 사용자 식별자(UUID) URL 노출 | `golden_set_id` / `item_id` 는 내부 UUID 로만 구성, PII 아님 |
| 이름/이메일 URL 파라미터 삽입 | 없음 — 필터는 `domain/status` 만 |
| 에러 배너 내용에 스택트레이스 노출 | 없음 — `getApiErrorMessage(err, fallback)` 로 사용자 메시지로만 표시 |
| 파일 export 에 다른 사용자 데이터 포함 | 서버가 `scope_id` 일치 row 만 직렬화 — 타 scope 유출 없음 |

### 3.2 파일 업로드(JSON import)

| 항목 | 상태 |
| --- | --- |
| 확장자 제한 | `.json` 만 (서버) |
| MIME 타입 제한 | `application/json \| text/json \| text/plain` (서버) |
| 크기 제한 | 기존 FastAPI `UploadFile` + 업스트림 멀티파트 제한(별도 운영 설정) 에 위임. 본 PR 에서 우회 경로 도입 없음 |
| 파일 이름 sanitization | 프론트에서 파일명은 렌더하지 않음 — 서버 로그에만 기록 |
| 감사 로그 | `golden_set.import` 이벤트 발행(기존 서비스 코드) |

### 3.3 다운로드(Export)

- `downloadJsonFile()` 은 `new Blob([JSON.stringify(data, null, 2)])` → `URL.createObjectURL` → 동적 `<a>` 클릭 → `URL.revokeObjectURL`.
- 파일명은 `detailQuery.data?.data?.name` 과 타임스탬프 조합. `name` 값은 백엔드 길이 검증(1~200자)을 이미 통과한 값이며, `a[download]` 속성은 **렌더 DOM 이 아니라** 다운로드 메타데이터로만 사용되므로 XSS 경로 없음.
- `URL.createObjectURL` 직후 `revokeObjectURL` 로 메모리 누수 방지.

### 3.4 CSRF

- API 는 쿠키 기반 세션 + `SameSite=Lax` 로 기본 CSRF 차단.
- 변경 작업(`POST/PUT/DELETE`)은 `fetch` 의 `credentials: "same-origin"` 설정과 동일 오리진에서만 호출 — 본 PR 에서 cross-site 경로 노출 없음.
- **결론: 영향 없음.**

### 3.5 Deprecated API 사용 (CLAUDE.md #1)

| API | 상태 |
| --- | --- |
| `document.title` 수동 조작 | 사용 안 함(`/admin/golden-sets/page.tsx` metadata export 사용) |
| React `findDOMNode` 등 | 사용 안 함 |
| `componentWillMount` 등 legacy lifecycle | 사용 안 함(함수형 컴포넌트) |
| `useEffect` 의 `async` 함수 직접 할당 | 사용 안 함 |

### 3.6 Critical CVE 의존성 (CLAUDE.md #2)

- 의존성 추가/업데이트 없음 → 공격 표면 변화 없음.

### 3.7 보안 일반(CLAUDE.md #3)

| 항목 | 검토 |
| --- | --- |
| localStorage/sessionStorage 신규 사용 | 없음 |
| 외부 iframe/embed | 없음 |
| 외부 리소스 로드 | 없음 — 아이콘 SVG 인라인만 사용 |
| 폐쇄망 동작(S2 ⑦) | 외부 네트워크 의존 없음. Blob export 는 브라우저 내부에서만 동작 |

---

## 4. S2 추가 원칙 부합성

| 원칙 | 검토 |
| --- | --- |
| S2 ⑤ scope 어휘 하드코딩 금지 | 프런트/백엔드 코드에 `"team"/"org"/"enterprise"` 등 scope 문자열 하드코딩 0건. `scope_profile_id` 는 UUID 로만 전파 |
| S2 ⑥ ACL 모든 검색·조회·쓰기 API 의무 적용 | `_require_scope(actor)` 가 모든 엔드포인트 선두에서 강제. Repository 의 WHERE 절도 일치 |
| S2 ⑥ 폴백 경로(FTS, 로컬 모델) 동일 ACL | 본 PR 범위에는 FTS 경로 없음. 골든셋 자체가 FTS 대상 아님 |
| S2 ⑦ 폐쇄망 동작 | 외부 API 호출 경로 신규 없음. Import/Export 모두 로컬-only |
| S2 ⑤ 감사 로그 `actor_type` | `audit_emitter.emit(... actor_type=_actor_type_str(actor))` 가 모든 쓰기 엔드포인트에서 호출됨을 확인 |

---

## 5. 잔존 고위험 항목 (본 PR 범위 밖, 참고)

S2 종결 선언 후에도 잔존하는 P0 중 본 작업과 간접적으로 연관되는 항목(`project_s2_closure_gaps.md` 메모리 참조):

- **테스트 커버리지 35%** — 본 PR 의 `item_count` 패치와 `goldenSetsApi` 확장에 대한 유닛/통합 테스트 부재. 권고:
  1. `list_by_scope` 의 `item_count` 집계 계약 테스트
  2. `GoldenSet` Pydantic 모델 필드 존재 테스트
  3. `/api/v1/golden-sets` 의 Scope Profile 바인딩 없는 계정에 대한 403 회귀 테스트
- **AI 품질 실측 공백** — 본 PR 는 *골든셋을 채우는* 기능이라 오히려 이 갭 해소의 전제 UI. 본 PR 자체는 품질 실측에 영향 없음.
- **compose 기본 시크릿** — UI 무관.

추가로 런타임 리뷰에서 발견된 부차 이슈:

- **#31 에러 배너 S2 ⑥ 단정 표기** — 서버 5xx 에도 "Scope Profile 바인딩 또는 권한" 안내가 함께 뜸. 보안 관점에서는 ACL 실패/장애를 사용자에게 혼동시킬 수 있어 UX 개선 필요(기밀 누출은 아님).
- **ValueError → 503 매핑** — `unhandled_exception_handler` 체인 조사 필요. 운영 관점에서 5xx 원인 추적 지연을 일으킬 수 있음.

---

## 6. 결론

- 본 PR 의 UI/백엔드 변경은 **신규 취약점 도입 0건**.
- OWASP Top 10 전 카테고리에서 **영향 없음** 또는 간접적 개선.
- S2 ⑤ ⑥ ⑦ 모두 유지(바인딩 강화로 ⑥ 는 오히려 정상 가동).
- **권고 후속**:
  1. `item_count` 회귀 유닛 테스트 추가 (S2 종결 P0 커버리지 해소의 일부)
  2. 에러 배너 상태코드별 분기(#31, UX — 정보 혼동 감소)
  3. `ValueError → 500/503` 매핑 명확화 (운영 관측성)
  4. `/api/v1/golden-sets` 에 대한 IDOR/ACL 회귀 테스트(다른 scope 계정으로 조회 시 404) 자동화
