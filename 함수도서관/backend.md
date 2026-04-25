# 함수도서관 — 백엔드 (Python)

> 배치 규약: 본 카탈로그의 모든 신규 유틸은 `backend/app/utils/` 하위에 도메인별 모듈로 둔다.
> 기존 `backend/app/utils/html_sanitizer.py` 는 표준 XSS 방어 유틸로 유지한다.

---

## 0. 기존 유틸 (이미 사용 중)

### `app.utils.html_sanitizer`

- `sanitize_html(text: str) -> str` ✅
  - status: ✅ Existing
  - purpose: HTML 문자열에서 위험 태그·이벤트 핸들러·javascript:/data: URI 를 제거한다
  - effects: none
  - errors: none (입력이 `None`·빈 문자열이면 입력 그대로 반환)
  - replaces: N/A
  - tests: `backend/tests/unit/utils/test_html_sanitizer.py` (존재 여부는 구현 시 확인 필요)
  - notes: 화이트리스트는 `_ALLOWED_TAGS`, `_ALLOWED_ATTRS` 에 집중. 확장 시 XSS 회귀 테스트 필수 (CLAUDE.md §4.5).

---

## 1. 제안 유틸 (🟡 Proposed, 2026-04-25 감사 결과 기반)

아래 후보는 모두 **실제 grep 스팟 체크로 ≥ 2회 등장**이 확인되었다. 구현 턴에서 하나씩 분리 PR로 진행한다.

### 1.1 `app.utils.time` ✅ (B1, 2026-04-25)

- `utcnow() -> datetime` ✅
  - purpose: `datetime.now(timezone.utc)` 를 감싸는 얇은 래퍼. 호출지 가독성과 테스트에서의 mock 주입성 모두를 위해 도입.
  - effects: none (순수 함수)
  - errors: none
  - module: `backend/app/utils/time.py`
  - tests: `backend/tests/unit/utils/test_time.py`
  - replaces: `datetime.now(timezone.utc)` 65회 + `datetime.utcnow()` 1회 = 66회 치환 (2026-04-25)
  - notes:
    - `datetime.utcnow()` 는 **deprecated** (Python 3.12+). 일괄 대체로 deprecation 경고 해소됨 (CLAUDE.md 글로벌 §1 "deprecated 금지").
    - 테스트에서 시간 고정 시 `unittest.mock.patch("app.utils.time.utcnow")` 를 쓴다 (기존에 `freezegun` 을 쓰는 경우에도 추가 호환 OK).

- `utcnow_iso() -> str` ✅
  - purpose: ISO 8601 UTC 타임스탬프 문자열 (`.isoformat()`) 생성.
  - effects: none
  - replaces: `datetime.now(timezone.utc).isoformat()` 5회 치환 (2026-04-25). `(utcnow() - timedelta(...)).isoformat()` 같은 중간 연산이 낀 표현은 그대로 유지.
  - notes: 감사 로그·응답 직렬화에서 쓰이므로 timezone-aware 보장이 중요.

### 1.2 `app.utils.converters` ✅ (BE-G1, 2026-04-25)

- `uuid_str_or_none(value: UUID | str | None) -> str | None` ✅
  - status: ✅ Existing (2026-04-25 BE-G1 승격)
  - purpose: `str(uuid) if uuid else None` 보일러플레이트 통합. `UUID`/`str`/`None` 세 입력을 falsy 단일 분기로 처리.
  - effects: none
  - errors: none (형식 미검증 — 호출자가 보장하거나 `ensure_uuid` 사용)
  - source: `backend/app/utils/converters.py`
  - tests: `backend/tests/unit/utils/test_converters.py` (UUID instance / str / None / 빈문자열 falsy / 형식 미검증)
  - replaced in: 21 파일 47 호출자 (admin 7 / vectorization 8 / extractions 3 / extraction_evaluation_repository 6 / batch_extractions 2 / admin_extraction_results 2 / rag 2 / approved_extraction model+repo 4 / batch_extraction model+repo 2 / extraction_candidate_repo 1 / extraction_record_repo 1 / extraction_schema_repo 1 / extraction_span_repo 2 / extraction schema 1 / scope_profiles 1 / migration 1 / snapshot_sync_service 1 / vectorization_status_service 1 / extraction_verification_service 1).
  - notes:
    - 호출지 의미와 100% 동치 (`str(x) if x else None` ↔ `uuid_str_or_none(x)`) — 형식 미검증, 빈문자열 falsy 동일.
    - AST 변환 스크립트(`outputs/migrate_uuid_str_or_none.py`)가 정확한 패턴 (`str(NAME) if NAME else None` 또는 `is not None else None`) 만 매칭, NAME 일치 검사 + module-level import 추가.

- `ensure_uuid(value: UUID | str, *, label: str = "id") -> UUID` ✅ (유틸만 신설, 호출지 마이그레이션은 별 라운드)
  - status: ✅ Existing (2026-04-25 BE-G1 신설). 호출지 마이그레이션은 ⚠️ 미진행.
  - purpose: `UUID(value)` 호출을 중앙화. `str/UUID` 모두 받고 `UUID` 반환. 검증 실패 시 동일 에러 계약 (`ApiValidationError`) 으로 매핑.
  - effects: none
  - errors: `ApiValidationError(f"{label}이(가) 올바른 UUID 형식이 아닙니다", details=[{"field": label, "reason": "must be a valid UUID", "code": "INVALID_UUID"}])`. `__cause__` 에 원본 `ValueError`/`TypeError` 보존.
  - source: `backend/app/utils/converters.py`
  - tests: `backend/tests/unit/utils/test_converters.py` (UUID instance passthrough / str → UUID / 잘못된 입력 → ApiValidationError + details + code / label 메시지·details 반영 / keyword-only label / __cause__ 보존)
  - excluded by design (호출지 미마이그레이션):
    - `UUID(...)` 직호출 72회 (extraction_schemas 등) — try/except ValueError 흐름이 라우터마다 다양 (FastAPI Path 자동 검증 / 명시적 try/except / 무가드 raise) → 의미 변화 검토 필요. 본 라운드는 유틸만 신설하고 마이그레이션은 라우터별 검증 후 별 라운드.
    - `_NIL_NODE_ID = UUID(int=0)` 같은 keyword arg 호출은 본 헬퍼 시그니처 외 (positional only).
  - notes:
    - CONSTITUTION 제14조 Shared Error Contract 준수 — `details[0].code = "INVALID_UUID"` 로 wire 호환.
    - `label` 은 keyword-only — 잘못된 위치 호출 방어.
    - 호출지 마이그레이션 시 try/except ValueError → ApiValidationError 도 같이 정리 필요.

### 1.3 `app.utils.json_utils` ✅ (BE-G2, 2026-04-25)

- `loads_maybe(value: Any) -> Any` ✅
  - status: ✅ Existing (2026-04-25 BE-G2 승격)
  - purpose: ``str`` 이면 ``json.loads``, 아니면 그대로 반환. JSONB 컬럼이 드라이버에 따라 ``str`` 또는 ``dict``/``list`` 로 반환되는 경우를 단일 호출로 정규화.
  - effects: none
  - errors: ``json.JSONDecodeError`` 가 그대로 전파됨 (호출자 ``try/except`` 흐름 보존).
  - source: `backend/app/utils/json_utils.py`
  - tests: `backend/tests/unit/utils/test_json_utils.py` `TestLoadsMaybe` (str→dict / str→list / 한글 / dict/list/None passthrough / int passthrough / 잘못된 JSON → JSONDecodeError / 빈문자열)
  - replaced in: 9 파일 21 사이트 (agent_repository 1 / approved_extraction_repository 3 / extraction_candidate_repository 3 / extraction_evaluation_repository 4 / extraction_record_repository 2 / extraction_schema_repository 3 / golden_set_repository 2 / rag_repository 2 / vectorization_service 1).
  - excluded by design:
    - try/except 안에서 ``json.JSONDecodeError`` 또는 ``ValueError`` 를 도메인 오류로 매핑하는 호출 — 그대로 보존 (의미 유지).
    - ``str`` 외 타입 검사가 다른 호출 (예: ``isinstance(raw, (str, bytes))``) — 본 스크립트 매칭 외.
  - notes:
    - **`ApiValidationError` 매핑 변형 (`loads_strict`)** 은 별 라운드 — try/except 흐름 검토 후 (BE-G1 의 `ensure_uuid` 와 동일 패턴).

- `loads_strict(value: Any, *, label: str = "value") -> Any` ✅
  - status: ✅ Existing (2026-04-25 R1 승격)
  - purpose: ``loads_maybe`` 와 같은 시맨틱이지만 JSON 파싱 실패 시 :class:`ApiValidationError` 매핑 (CONSTITUTION 제14조 Shared Error Contract). BE-G1 ``ensure_uuid`` 와 동일 패턴.
  - effects: none
  - errors: ``ApiValidationError`` (``details=[{"field": label, "reason": "must be valid JSON", "code": "INVALID_JSON"}]``). ``__cause__`` 에 원본 ``json.JSONDecodeError`` 보존.
  - source: `backend/app/utils/json_utils.py`
  - tests: `backend/tests/unit/utils/test_json_utils.py` `TestLoadsStrict` (6 case — 유효 JSON / dict passthrough / invalid → ValidationError / label / keyword-only / __cause__)
  - excluded by design (호출지 마이그레이션 별 라운드):
    - 21 ``loads_maybe`` 호출지 중 try/except 흐름 검토 후 ``loads_strict`` 로 교체할 사이트 — 호출지 의미 검토 필요.

- `dumps_ko(value: Any, **kwargs: Any) -> str` ✅
  - status: ✅ Existing (2026-04-25 BE-G2 승격)
  - purpose: ``json.dumps(value, ensure_ascii=False, **kwargs)`` 의 단일 엔트리. 한글 보존 직렬화 표준.
  - effects: none
  - errors: ``ensure_ascii`` 키워드를 호출자가 다시 명시하면 ``TypeError`` (의도된 보호).
  - source: `backend/app/utils/json_utils.py`
  - tests: `backend/tests/unit/utils/test_json_utils.py` `TestDumpsKo` (한글 escape 안 됨 / dict / list / None / bool / separators / sort_keys / indent / default 콜백 / ensure_ascii 중복 → TypeError / loads_maybe 와 round-trip)
  - replaced in: 21 파일 42 사이트 (mcp_router 1 / rag 1 / migration 2 / observability/log_config 1 / approved_extraction_repository 1 / documents_repository 2 / extraction_candidate_repository 1 / extraction_schema_repository 4 / golden_set_repository 8 / idempotency_repository 1 / nodes_repository 2 / rag_repository 2 / versions_repository 5 / schemas/documents 2 / schemas/versions 2 / diff_service 1 / extraction_pipeline_service 2 / idempotency_service 1 / rag_service 1 / users_repository 1 (수동, _json 별칭 케이스))
  - notes:
    - 호출자가 ``separators``, ``sort_keys``, ``indent``, ``default`` 등을 그대로 넘길 수 있다 (kwargs 통과).
    - `_json = json` 같은 별칭 호출은 자동 매칭에서 빠지므로 수동 처리 (스크립트 한계, 1건 발견·수정 — users_repository).

### 1.4 `app.utils.strings`

- `normalize_display_name(raw: str | None, min_len: int, max_len: int, *, forbid_slash: bool = False, label: str = "이름") -> str` ✅
  - status: ✅ Existing (2026-04-25 도입 — B6)
  - purpose: `folders_service._normalize_name` 과 `collections_service._normalize_name` 의 공통 골격. 앞뒤 trim + 연속 공백 1개로 압축 + 길이 검사 + 선택적 `/` 금지.
  - effects: none (순수 함수)
  - errors: `ApiValidationError(f"{label}은 필수입니다" | f"{label}은 {min}~{max}자 사이여야 합니다" | f"{label}에 '/' 는 포함할 수 없습니다")`
  - replaces: `backend/app/services/folders_service.py` 와 `backend/app/services/collections_service.py` 의 `_normalize_name` — 두 모듈은 **외부 import 호환을 위해 thin wrapper 로 유지**되고 내부는 본 유틸에 위임한다 (라인별 사이드-바이-사이드 비교, 2026-04-25).
  - tests: `backend/tests/unit/utils/test_strings.py` (20+ 케이스)
  - notes:
    - 대소문자 보존(기존 동작). UNIQUE 제약은 DB 에서 COLLATE 로 처리하므로 유틸은 대소문자에 관여하지 않는다.
    - `forbid_slash` / `label` 은 keyword-only. 위치 인자로 전달하면 `TypeError`.
    - 한국어 조사 `"은"` 고정. 호출자는 받침(ㅁ·ㅇ·ㄴ 등)으로 끝나는 라벨을 넘긴다는 전제다 (예: "폴더 이름", "컬렉션 이름"). 새 호출지에서 받침 없는 라벨을 쓰려면 조사 분기가 필요.

- `normalize_lower(raw: str | None) -> str | None` ✅
  - status: ✅ Existing (2026-04-25 BE-G1 승격)
  - purpose: `raw.strip().lower()` 또는 `raw.lower().strip()` 혼재 정리. `None` 패스스루. 호출 순서 **strip → lower** 통일 (대부분 의미 동등, 합자 같은 극단 유니코드는 차이 가능하나 본 도메인에선 영향 없음).
  - effects: none
  - errors: none
  - source: `backend/app/utils/strings.py` (B6 모듈에 함께 둠)
  - tests: `backend/tests/unit/utils/test_strings.py` `TestNormalizeLower` (None 패스스루 / 빈문자열 / 공백만 → "" / strip→lower 순서 / 내부 공백 보존 / 이메일 / 한글 / 유니코드 / 멱등)
  - replaced in: 7 사이트 (users_repository / system / golden_sets / auth_router×2 / oauth_service / rate_limit) — 모두 입력이 `str` 인 케이스라 `or ""` 또는 `assert is not None` 으로 narrowing 보강.
  - notes:
    - 유니코드 NFKC 정규화는 적용 안 함 (별 라운드).
    - 이메일 정규화의 IDNA / plus-addressing 처리는 호출자 책임 (별 도메인 유틸).

### 1.5 `app.utils.http_errors`

- `not_found(detail: str, *, error_code: str | None = None, headers: dict[str, str] | None = None) -> HTTPException` ✅
- `bad_request(detail: str, *, error_code: str | None = None, headers: dict[str, str] | None = None) -> HTTPException` ✅
- `unprocessable_entity(detail: str, *, error_code: str | None = None, headers: dict[str, str] | None = None) -> HTTPException` ✅
- `conflict(detail: str, *, error_code: str | None = None, headers: dict[str, str] | None = None) -> HTTPException` ✅
  - status: ✅ Existing (2026-04-25 BE-G3 승격, 4종 세트)
  - purpose: `raise HTTPException(status_code=XXX, detail="...")` 보일러플레이트 통합. CONSTITUTION 제14조 Shared Error Contract 의 `error_code` 필드를 wire 호환 보존하면서 점진 도입.
  - effects: none (예외 인스턴스를 만들 뿐 raise 는 호출자 책임)
  - source: `backend/app/utils/http_errors.py`
  - tests: `backend/tests/unit/utils/test_http_errors.py` (4종 status_code · 한국어 detail 보존 · X-Error-Code 헤더 자동 부착 · 호출자 headers 머지 · 충돌 시 호출자 우선 · keyword-only 강제 · returns-not-raises)
  - replaced in: 14 라우터 파일 215 호출자 (account_router 18 / admin 83 / admin_extraction_results 11 / auth_router 15 / citations 2 / conversations 11 / extractions 7 / extraction_schemas 20 / golden_sets 18 / rag 4 / scope_profiles 11 / search 4 / vectorization 6 / webhooks 5)
  - excluded by design:
    - 401 Unauthorized · 403 Forbidden · 500 Internal Server Error · 501 Not Implemented · 502 Bad Gateway 등 본 4종 외 status code — 별 그룹 후속.
    - `golden_sets.py:86` 의 `status.HTTP_401_UNAUTHORIZED` 같은 `fastapi.status` 상수형 호출도 비대상 (직접 정수 호출만 매칭).
    - 14 라우터의 `from fastapi import HTTPException` 는 위 잔존 코드들이 여전히 직접 호출하므로 import 유지 (스크립트 자동 판정).
  - notes:
    - **wire 호환 정책**: `detail` 은 문자열로 그대로 전달 (프런트 `getApiErrorMessage()` 가 string detail 을 가정). dict-detail 본격 도입은 별 라운드.
    - **error_code 노출 채널**: 응답 헤더 `X-Error-Code: <value>` 로 자동 부착. 호출자가 같은 키를 명시하면 호출자 값이 우선. 둘 다 없으면 `headers=None` 으로 FastAPI 기본 동작.
    - `error_code` 는 `"NOT_FOUND_DOCUMENT"`, `"INVALID_SCOPE"`, `"CONFLICT_VERSION"` 처럼 도메인 접두어로 관리 (호출자 enum 권장).
    - 마이그레이션 1차에서는 `error_code` 인자를 비워두고 detail 만 옮겼다 — 점진적으로 라우터별로 채워 넣을 예정.
    - 보안: `detail` / `error_code` 는 외부 응답에 노출되므로 SQL · 스택트레이스 · 비밀값 금지 (CLAUDE.md §4.3). `X-Error-Code` 값에 개행/CR 금지 (헤더 인젝션).

### 1.6 `app.utils.actor` ✅ (BE-G4, 2026-04-25)

- `actor_type_str(actor: ActorContext | None) -> Literal["user", "agent", "system"]` ✅
  - status: ✅ Existing (2026-04-25 BE-G4 승격)
  - purpose: `ActorContext` 의 4분류 (`anonymous`/`user`/`agent`/`service`) 를 `AuditEmitter` 의 Literal 3분류 (`"user"`/`"agent"`/`"system"`) 로 매핑하는 단일 진입점. SERVICE → `"system"`, ANONYMOUS → `"user"`.
  - effects: none
  - errors: none (None / 누락 actor_type 모두 방어적 fallback)
  - source: `backend/app/utils/actor.py`
  - tests: `backend/tests/unit/utils/test_actor.py` (4 ActorType 매핑 / None 방어 / SERVICE → service 회귀 가드 / ActorContext.audit_actor_type property 위임 / AuditEmitter Literal 일치)
  - replaced in:
    - `app/api/auth/models.py` `ActorContext.audit_actor_type` property — 본 helper 위에서 위임 (지연 import 로 순환 회피). **시맨틱 통일 — SERVICE → "system"** (이전엔 `ActorType.value` 그대로 반환해 `"service"` → `AuditEmitter` Literal 위반 가능. **잠재 보안 버그 수정**).
    - `app/api/v1/golden_sets.py` `_actor_type_str` — `actor.actor_type.value if actor.actor_type else "user"` 인라인 → `actor_type_str(actor)` 위임.
    - `app/api/v1/extraction_schemas.py` `_actor_info` — 동일. SERVICE → "system" 매핑 후 ActorInfo 가 "user|agent" 만 허용하므로 명시적으로 "user" 로 좁힘 (도메인 의도 보존).
  - 자동 영향: `audit_actor_type` property 사용지 9 사이트 (mcp/tools.py + scope_profiles.py 8) 가 자동으로 신규 매핑 적용.
  - excluded by design (별 라운드):
    - 162 emitter 호출자 일괄 마이그레이션 — 호출지마다 actor 컨텍스트가 다양하고 `actor_type=` 인자가 hard-coded literal (`"user"` / `"agent"`) 또는 인라인 분기 등 패턴이 다름. 의미 동등성 자동 검증 어려움.
    - `actor_type=actor.actor_type or "user"` 패턴 (잘못된 코드 — Enum 인스턴스는 항상 truthy) — 별도 정정 라운드.
  - notes:
    - 매핑 표: USER → "user", AGENT → "agent", SERVICE → **"system"**, ANONYMOUS → "user", None → "user".
    - `AuditActorType = Literal["user", "agent", "system"]` 타입 alias 도 함께 export — 호출자가 타입 힌트 사용 가능.
    - **보안 시맨틱**: SERVICE → "system" 회귀 가드 테스트 필수 (`test_service_never_returns_service_literal`).

### 1.7-extension `app.audit.emitter` ContextVar (R9, 2026-04-25)

- `set_trace_id(trace_id) -> Token` / `reset_trace_id(token)` / `current_trace_id() -> str | None` ✅
  - status: ✅ Existing (2026-04-25 R9 도입)
  - purpose: AuditEmitter 가 호출자가 매번 `trace_id` 인자를 명시하지 않아도 자동 첨부하도록 ContextVar 도입.
  - effects: ContextVar set/reset (asyncio task-local, 안전).
  - integration: `RequestContextMiddleware.dispatch` 가 try/finally 로 `set_trace_id(trace_id)` → `reset_trace_id(token)` 패턴.
  - emit() 동작: 호출자가 명시한 `trace_id` 가 우선, `None` 이면 `_trace_id_var.get()` 으로 fallback. 둘 다 None 이면 emit 시 trace_id 누락 (기존 동작 호환).
  - tests: 모듈 스모크 4/4 — default None / set·reset / 자동 첨부 / 명시 우선.
  - notes:
    - asyncio.Task 안전 — ContextVar 는 task-local 이라 동시 요청 분리.
    - 미들웨어 dispatch 종료 시 reset 필수 — 재진입 (테스트/하위 요청 등) 안전.

### 1.7 `app.utils.audit_helpers` ⚠️ (도입 안 함, BE-G4 결정)

- ~~`emit_audit(event: str, *, actor: ActorContext, extra: dict[str, Any] | None = None) -> None`~~ ⚠️
  - status: ⚠️ Not Adopted (2026-04-25 BE-G4 결정)
  - decision: 도입하지 않음.
  - reasoning:
    - `app.audit.emitter.AuditEmitter.emit` 이 이미 매우 풍부한 시그니처 (event_type / action / actor_id / actor_type / resource_type / resource_id / result / request_id / trace_id / tenant_id / metadata / actor_role / target_version_id / previous_state / new_state) 를 제공.
    - 162 호출자가 각자 다른 인자 조합을 사용 — 단순 wrapper 가 호출자별 의도를 가리고 회귀 위험.
    - 본 §1.7 사양의 핵심 가치 (trace_id 자동 첨부 / 폐쇄망 degrade) 는 `AuditEmitter.emit` 자체 책임에 더 적합 — 별 라운드에서 emitter 내부 정비.
    - 본 라운드는 `actor_type_str` (§1.6) 만 도입. emit_audit 신규 layer 는 추가 가치 < 회귀 위험.
  - alternatives:
    - 호출지가 `actor.audit_actor_type` (자동으로 신규 매핑 적용) + `AuditEmitter.emit` 직접 호출.
    - trace_id 자동 첨부가 필요하면 `AuditEmitter` 내부에서 contextvar 로 처리 (별 라운드).

### 1.7 `app.utils.audit_helpers`

- `emit_audit(event: str, *, actor: ActorContext, extra: dict[str, Any] | None = None) -> None` 🟡
  - purpose: `audit_emitter.log(...)` 호출 시 `actor_type`, `actor_id`, `trace_id`, `scope_profile_id` 를 일관되게 첨부한다.
  - effects: audit_log (외부 I/O — DB 또는 로그 싱크)
  - errors: 내부에서 `logger.warning` 로 삼킴 (감사 실패가 본 트랜잭션을 깨지 않도록)
  - replaces: `from app.audit.emitter import audit_emitter` 하는 파일 ~20개 + emitter.log 호출 보일러플레이트
  - notes:
    - 폐쇄망에서도 동작해야 함(S2 ⑦). 외부 sink 가 off 이면 파일 로그로 degrade.
    - CONSTITUTION 제13조 Observability 의 trace_id 필드 스키마와 호환되어야 함.
    - 이 유틸은 **필수 제11조 Index 대상** (권한·감사 관련).

### 1.8 `app.db.cursor_helpers` ✅ (BE-G5, 2026-04-25)

- `fetch_one_as(conn, sql, params, mapper) -> T | None` ✅
- `fetch_many_as(conn, sql, params, mapper) -> list[T]` ✅
  - status: ✅ Existing (2026-04-25 BE-G5 승격)
  - purpose: ``with conn.cursor() as cur: cur.execute(...); row = cur.fetchone(); return mapper(row) if row else None`` 6~8줄 패턴을 단일 호출로.
  - effects: db_read
  - errors: DB 드라이버 예외 (psycopg2.Error 등) + mapper 예외 그대로 전파 (호출자 책임).
  - source: `backend/app/db/cursor_helpers.py`
  - tests: `backend/tests/unit/utils/test_cursor_helpers.py` (TestFetchOneAs 5 + TestFetchManyAs 4 = 9 case — fake cursor mock 으로 with-context/execute/fetch* + mapper 호출 / None 반환 / 빈 결과 / 예외 전파 / dict params)
  - replaced in (시범):
    - `app/repositories/tags_repository.py` `autocomplete` — `with cur.execute + fetchall + [_row_to_tag(...) for r in rows]` → `fetch_many_as(conn, sql, params, lambda r: _row_to_tag(dict(r)))`.
  - excluded by design (별 라운드):
    - 266 fetchone + 107 fetchall = 373 호출지 일괄 — mapper 패턴 다양 (row → 도메인 모델 / row → 단순 값 / row → tuple), context 다양 (트랜잭션 안 / 다중 execute / RETURNING 절). 자동 변환 어려움. 점진 마이그레이션.
    - 다중 SQL 실행 / RETURNING 절 + commit / cursor_factory 변경 등 비-trivial 케이스.
  - notes:
    - **ACL 적용은 SQL 작성 책임자(호출자)의 몫**. 이 유틸이 임의로 scope_profile_id 필터를 넣지 않는다 (S2 원칙: scope 하드코딩 금지).
    - cursor_factory (RealDictCursor 등) 는 conn 설정에 위임 — helper 가 변경하지 않음.
  - **잔존 마이그레이션 정책 (D4, 2026-04-25 결정)**:
    - **신규 코드는 helper 의무** — code-review 게이트로 강제. `with conn.cursor()` + `execute` + `fetchone/fetchall` + mapper 패턴이 단순하면 helper 사용.
    - **기존 복잡 케이스 (347+ 사이트)** — 도메인별 점진 PR (1~2개씩). 일괄 큰 PR 금지 (회귀 위험).
    - 우선순위: 보안·권한 관련 repository (users/scope_profiles) > 일반 repository.
    - 다중 execute / RETURNING + commit / 트랜잭션 안 / 별칭 conn 변수 케이스는 helper 위에서 표현 어려우면 보존.

### 1.9 `app.repositories.pagination` ✅ (BE-G5, 2026-04-25)

- `paginate_page(page, page_size, *, max_page_size=200, default_page_size=50) -> tuple[int, int, int]` ✅
  - status: ✅ Existing (2026-04-25 R1 승격)
  - purpose: 1-base ``page`` + ``page_size`` clamp + ``offset = (page-1)*page_size`` 계산을 단일 호출로. `clamp_pagination` 위에서 동작.
  - effects: none
  - source: `backend/app/repositories/pagination.py`
  - tests: `backend/tests/unit/utils/test_pagination.py` `TestPaginatePage` (10 case — 정상 / offset 계산 / None 기본값 / 0 page 보호 / 음수 / 상한 cap / first page offset=0 / clamp 일관성)
  - excluded by design (호출지 마이그레이션 별 라운드):
    - 10+ 사이트 (`vectorization.py:509,607` / `proposal_queue.py:72,271` / `scope_profiles.py:595` / `admin_extraction_results.py:226` / `admin.py:299,444` / `documents_repository.py:229` / `versions_repository.py:321` 등) — 호출지마다 max_page_size 정책이 달라 호출자 명시 필요. 점진 마이그레이션.

- `clamp_pagination(limit, offset, *, max_limit=200, default_limit=50) -> tuple[int, int]` ✅
  - status: ✅ Existing (2026-04-25 BE-G5 승격)
  - purpose: limit/offset 정규화 (None 기본값 + 음수 보호 + 상한 cap).
  - effects: none
  - errors: none (음수는 1/0 으로, 초과는 max 로 cap)
  - source: `backend/app/repositories/pagination.py`
  - tests: `backend/tests/unit/utils/test_pagination.py` `TestClampPagination` (13 case + 4 parametrized = 17 case — 정상 / None / 0 / 음수 / 상한 / 경계 / parametrized)
  - replaced in (시범):
    - `app/repositories/tags_repository.py` `autocomplete` — `max(1, min(limit, 100))` → `clamp_pagination(limit, 0, max_limit=100, default_limit=20)`.
    - `app/repositories/tags_repository.py` `popular` — `max(1, min(limit, 200))` → `clamp_pagination(limit, 0, max_limit=200, default_limit=50)`.
  - excluded by design (별 라운드):
    - **page → offset 변환** 패턴 (`offset = (page - 1) * page_size`) 10+ 사이트 — 별 helper (`paginate_page`) 로 후속.
    - 호출지마다 max_limit 정책이 다름 (50/100/200/300) — 호출자 명시 유지.
  - notes:
    - 상한이 도메인별로 다양 — 호출자가 ``max_limit`` 명시.
    - offset 상한은 없음 (호출자 도메인 정책).

### 1.9 `app.repositories.pagination`

- `clamp_pagination(limit: int | None, offset: int | None, *, max_limit: int = 200, default_limit: int = 50) -> tuple[int, int]` 🟡
  - purpose: 라우터에서 받은 limit/offset 을 범위 제한(clamp) + 기본값 적용.
  - effects: none
  - errors: none (음수는 0 으로 cap, 초과는 max 로 cap)
  - replaces: 리포지토리·라우터에 산재한 limit/offset 가드 (추정 43회 근처)
  - notes: pagination 상한은 도메인별로 다를 수 있으므로 `max_limit` 를 호출자가 덮어쓸 수 있게 유지.

---

## 2. 도입 보류 / 주의 후보 (⚠️ 검토 필요)

### 2.1 리포지토리 CRUD 베이스 클래스 ⚠️
- 24개 리포지토리에 유사 CRUD 가 반복되나, **S2 ACL 필터**가 리포지토리마다 다르게 녹아있어 단순 베이스 클래스로 일반화하면 ACL 누락 위험이 있다.
- **결정**: 이번 초판에는 **도입하지 않는다**. 대신 각 리포지토리가 `app.db.cursor_helper`, `app.utils.converters.uuid_str_or_none`, `app.repositories.pagination.clamp_pagination` 를 사용하는 형태로 점진 통합한다.

### 2.2 `re.compile` 상수 모음 ⚠️
- 40회 등장하지만 도메인별 의미가 다름(예: 문서 타입, UUID 검증, 태그명). 통합하면 탐색성이 오히려 떨어진다.
- **결정**: 정규식은 해당 서비스·라우터 파일 상단에 유지. 단, `_DOC_TYPE_RE`, `_UUID_RE` 등 이름이 같고 의미도 같은 케이스가 발견되면 도메인 단위 모듈(`app.utils.patterns.doc_type`)로 뽑는다.

---

## 3. 감사 메타

- 후보 선정 근거: `docs/함수도서관/감사보고서_2026-04-25.md` §1~§3
- 스팟 체크 방법: `Grep` 으로 각 패턴 seed 재조회 (결과는 감사보고서 §5 에 원본 카운트 기재)
- 후속 턴에서 각 항목을 **1개씩 별도 PR** 로 분리해 구현·호출지 치환·테스트·인덱스 업데이트를 묶어 처리한다. (CONSTITUTION 제30조 Commit as Verification Unit, 제32조 Reviewable PRs)
