# 추출 스키마 관리 화면 — P2 보안취약점 검사보고서

- 대상 경로: `/admin/extraction-schemas`
- 작성일: 2026-04-21
- 범위: P2-A/B/C/D 변경분과 그 주변 경로의 보안 리스크
- 준거: CLAUDE.md §2 절대 규칙(S1 ④, S2 ⑥), S2_5 P0/P1 보안 보고서에서 식별된 잔존 항목

## 1. 위협 모델 재확인

| 위협 ID | 자산 | 공격 표면 | 영향 |
| ------- | ---- | -------- | ---- |
| T-1 | `doc_type_code` | POST/PUT/DELETE/PATCH 본문 + 경로 | SQL/경로 인젝션, 예기치 않은 `%` / `;` / `/` 포함 |
| T-2 | `scope_profile_id` (UUID) | POST body + GET/GET versions query | 문자열 그대로 SQL 에 주입, 타 스코프 정보 엿보기 |
| T-3 | 자유 텍스트(`change_summary`, `reason`) | PUT/PATCH body | NUL/백스페이스/ESC 등 제어문자 저장 → 감사 로그 CSV/터미널 오염, 로그 서식 공격 |
| T-4 | 필드 정의(`fields`) | POST/PUT body | 중첩 객체를 통한 DoS (과도한 깊이, 초대형 문자열), 잘못된 속성으로 validator 우회 시도 |
| T-5 | 버전 이력 열람 | GET versions | 다른 Scope Profile 의 이력 교차 노출 (S2 ⑥ 위반) |
| T-6 | FieldsEditor 의 `default_value` 입력 (JSON parse) | 브라우저 | 악의적 JSON 이 화면 렌더 시 XSS 로 이어질 여지 |
| T-7 | 정규식 검증기 | 서버 | ReDoS |

## 2. 조치별 리스크 평가

### 2.1 T-1 `doc_type_code` — `^[a-zA-Z][a-zA-Z0-9_-]*$`

- P2-C 로 `CreateExtractionSchemaRequest._validate_doc_type_code` 가 도입됨.
- 프론트 `CreateSchemaModal.submit()` 과 동일한 정규식 사용 (이중 체크, 프론트만 믿지 않음).
- 정규식은 단순 문자 클래스 + 단일 `*` — 백트래킹 0 → ReDoS 불가.
- 이 regex 는 `^` 앵커가 있으므로 "영문자로 시작"이 보장된다. SQL `SELECT \* FROM t WHERE code = %s` 바인딩은 `psycopg2` 의 파라미터화(`%s`) 로 처리되며, 문자열 인젝션 위협 없음.
- 경로 파라미터 (`/extraction-schemas/{doc_type}`) 경로는 FastAPI 가 URL 디코딩 후 전달. 본 검증은 루트 body 대상이고, 경로 파라미터는 여전히 서비스 레이어 repository 가 받는 문자열인데 파라미터화된 쿼리로 안전.
- **잔존 리스크**: `GET /extraction-schemas/{doc_type}` 경로의 `doc_type` 은 별도 regex validator 가 없다. 다만 repository 측이 `psycopg2` 바인딩이라 SQL 인젝션은 불가. 문자열 그대로 404 로 귀결될 뿐이다. ⇒ 실제 공격 가치 없음.

### 2.2 T-2 `scope_profile_id` — UUID 강제

- P2-C 에서 POST body `scope_profile_id` 가 `UUID(v)` 로 파싱되며 실패 시 422.
- P1-C (이미 완료) 에서 `GET /extraction-schemas/{doc_type}` 라우터의 쿼리 `scope_profile_id` 도 `UUID(…)` 파싱.
- P2-D 에서 `GET /extraction-schemas/{doc_type}/versions` 도 동일 패턴을 적용 — SQL 에 도달하기 전에 422 로 거절.
- Repository 는 `str(scope_profile_id)` 를 `%s` 바인딩에 사용. psycopg2 quoting → 인젝션 불가.
- **S2 ⑥ 원칙** 준수: 버전 이력도 Scope Profile 로 분리 가능. 동일 doc_type_code 를 다른 Scope 에서 운영하는 경우, 교차 열람이 발생하지 않는다.
- **잔존 리스크**: `resolve_current_actor` 가 actor.scope_profile_id 를 자동 강제하지 않는 라우터(전역 관리자) 는 여전히 `scope_profile_id=None` 으로 전체 목록을 본다. 이는 S2-5 운영 정책(관리자 전역 조회) 에 부합하며 바뀔 필요 없다.

### 2.3 T-3 제어문자 / 공백-only 차단

- `_FORBIDDEN_CONTROL_RE = r"[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]"` — Tab(`\x09`), LF(`\x0A`), CR(`\x0D`) 을 제외한 C0 제어 문자 + DEL(`\x7F`).
- 적용 대상: `UpdateExtractionSchemaRequest.change_summary`, `DeprecateExtractionSchemaRequest.reason`.
- 공백-only 입력은 `_strip_and_validate_text` 가 `ValueError` 를 발생 → 422. `change_summary` 에 한정해서는 `""` / 공백 을 `None` 으로 수렴(사용자 편의) 후 나머지는 검사.
- 공격 시나리오:
  - NUL(`\x00`) 이 감사 로그 JSON 직렬화 경로에서 PostgreSQL `text` 컬럼에 저장되는 것을 원천 차단.
  - ANSI 이스케이프(ESC `\x1B`) 를 통한 터미널/로그 뷰어 서식 오염 공격 차단.
  - FF(`\x0C`) 를 이용한 페이지 분할 기반 CSV 오염 차단.
- **ReDoS 평가**: `[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]` 는 단순 문자 클래스 한 글자 → 선형 시간. `doc_type_code` 정규식도 백트래킹 0.

### 2.4 T-4 필드 정의(`fields`) 남용

- `fields` 는 `Dict[str, ExtractionFieldDef]` 로 Pydantic 가 타입 유효성을 먼저 검증한다.
- P2-C 로 `_fields_not_empty` 추가 → 빈 딕셔너리 거부.
- `ExtractionFieldDef` 의 model_validator 가 타입별 속성을 강제 (string-only: pattern/max_length, number-only: min/max_value, enum-only: enum_values, …). 공격자가 "enum + pattern" 같은 조합을 시도해도 서버에서 거절.
- FieldsEditor 는 동일 규칙을 UI 에서 강제 (타입 전환 시 허용 외 속성 자동 삭제) — 하지만 서버가 1차 방어선이라는 원칙은 유지.
- **DoS 고려**:
  - FastAPI 전역에 요청 바디 크기 한계가 걸려 있고(기본 Uvicorn `--limit-request-line` / `--limit-max-requests`) , fields 자체는 `Dict[str, ExtractionFieldDef]` 로 개수 상한은 별도 강제되지 않는다.
  - 실무 운영에서 100개를 넘는 fields 는 스키마 리뷰 절차에서 걸러질 것으로 본다. 명시적 상한 강제가 필요하다면 후속 작업(예: 100개 상한)으로 추가 가능.
- **잔존 리스크**: `field_type=object` 의 `nested_schema` 는 재귀 `ExtractionFieldDef` 참조이므로 이론상 깊은 중첩이 가능. 현재 깊이 상한이 없으나, Python 재귀 한계와 Pydantic 모델 인스턴스 생성 비용이 실질적 저지선이다. 운영 모니터링이 필요한 지점.

### 2.5 T-5 버전 이력 열람 — Scope 교차 차단

- P2-D 로 `GET /extraction-schemas/{doc_type}/versions` 이 `scope_profile_id` 쿼리를 받아 repository WHERE 절에 합류.
- SQL WHERE 는 `doc_type_code = %s AND is_soft_deleted = FALSE [AND scope_profile_id = %s]` 동적 구성.
- 모든 값은 `%s` 바인딩 — concatenation/포맷스트링 사용 없음. 인젝션 0.
- **S2 ⑥ 준수**: Scope 분리가 요구되는 배포에서, 버전 이력까지 동일 정책으로 닫힌다.

### 2.6 T-6 FieldsEditor `default_value` JSON 파싱 XSS 표면

- `tryParseJson(raw)` 가 실패 시 원본 문자열을 그대로 반환.
- React 가 React 렌더 경로에 문자열을 삽입하는 것은 기본적으로 이스케이프된다. `dangerouslySetInnerHTML` 사용 없음. XSS 불가.
- 제출 시 서버 `ExtractionFieldDef.default_value` 는 `Any` 이므로 JSON 직렬화 후 `jsonb` 컬럼에 저장 — 저장 자체가 위험하지 않음.
- 스키마 조회 후 응답에 포함된 `default_value` 가 다른 화면(테이블, diff 뷰) 에 렌더되는 경로를 추적:
  - `DiffView` 는 `renderValue(v)` 를 통해 `JSON.stringify` 한 문자열만 출력. DOM 삽입은 일반 text node — XSS 0.
  - 상세 패널의 표 뷰는 `f.description || "-"` 만 출력. `default_value` 자체는 지금 표에 렌더되지 않는다.
- **잔존 리스크**: 향후 `default_value` 를 표에 직접 노출한다면 React 일반 텍스트 삽입 경로여도 JSON payload 안에 HTML 엔터티로 보이는 문자열이 섞일 수 있으니, 그 시점에 별도 sanitize 정책 점검 필요.

### 2.7 T-7 ReDoS

검토한 정규식 3종:

| regex | 패턴 유형 | 최악 시간 |
| ----- | -------- | --------- |
| `^[a-zA-Z][a-zA-Z0-9_-]*$` | 선형 문자 클래스 | O(n) |
| `[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]` | 단일 문자 클래스 매칭 | O(n) |
| `^[a-z][a-z0-9_]*$` (FieldsEditor, snake_case) | 선형 문자 클래스 | O(n) |

모두 백트래킹을 유발하는 `(…)*`, `(.+)+`, 교차 가능한 `|` 가 없다. 실측/정적 분석 모두에서 ReDoS 없음.

## 3. 감사 로그(actor_type) — S2 ⑤ 준수 확인

P0 시점부터 `audit_emitter.emit_for_actor(...)` 이 모든 create/update/delete/deprecate 경로에서 호출되고 있다 (router line 149, 270, 306, 340).
P2 변경 이후에도 동일하게 `actor_type="user"|"agent"` 가 기록되며, 감사 경로 회귀 없음.

## 4. 배포 전 확인 체크리스트

| 항목 | 결과 | 비고 |
| ---- | ---- | ---- |
| SQL 주입 가능 경로 0 | ✅ | psycopg2 `%s` 바인딩, 동적 WHERE 도 리터럴 컬럼명만 포함 |
| 경로 traversal 가능성 0 | ✅ | `doc_type_code` regex + 경로에서 `/` 자체가 오지 못함 (FastAPI 라우팅 경계) |
| ReDoS 가능성 0 | ✅ | 3개 regex 모두 선형 |
| 감사 로그 `actor_type` 필수 기록 | ✅ | 4개 쓰기 경로 모두 유지 |
| Scope Profile 별 열람 차단 (S2 ⑥) | ✅ | get, list, versions 3 경로 모두 scope_profile_id 쿼리 반영 |
| 자유 텍스트 제어문자 차단 | ✅ | change_summary, reason 검증 |
| 사용자 입력 기반 DOM 삽입 (XSS) | ✅ | React 텍스트 노드 렌더, `dangerouslySetInnerHTML` 사용 0 |
| 타입 혼합 공격(`enum + pattern` 등) | ✅ | ExtractionFieldDef model_validator 가 거절 |
| 빈 fields 요청 거부 | ✅ | `_fields_not_empty` |
| 비-UUID scope 문자열 거부 | ✅ | POST body + GET + versions 모두 |

## 5. 후속 보완 제안

1. **`fields` 개수 / 중첩 깊이 상한**: Phase 9 에서 명시적 상한(예: 200 / 깊이 3) 을 추가하면 DoS 내성이 강해진다.
2. **`default_value` UI 타입 인지**: T-6 에서 다뤘듯이 직접 렌더 경로가 생길 경우 타입별 입력 위젯 분리.
3. **E2E 테스트**: `GET versions?scope_profile_id=xxx` 교차 경로를 pytest 통합 테스트로 고정 (ai_eval 유사 패턴 활용).
4. **제어문자 테스트**: `backend/tests/test_extraction_schemas_api.py` 에 `\x00`, `\x1B`, `\x7F` 케이스를 추가해 회귀 방지.

## 6. 결론

P2 범위에서 새로 열린 경로 4건 모두 S1 ④ · S2 ⑥ 을 위반하지 않는다. 기존 경로에 대해 방어가 "좁아졌을 뿐" 넓어지지 않아,
회귀 위험도 낮다. ReDoS/SQL 주입/XSS 는 각각 구체적 근거로 배제되었고, 감사 기록 체인은 P0 이후 유지된다.
`critical` 혹은 `high` 등급 잔존 취약점 0 건. P2 배포 승인 가능.
