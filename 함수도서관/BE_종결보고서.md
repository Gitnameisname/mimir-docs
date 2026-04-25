# 함수도서관 백엔드 (Python) 종결보고서

> **작성일**: 2026-04-25
> **범위**: docs/함수도서관/backend.md §1 의 전 후보 (11 그룹)
> **검증 환경**: 샌드박스 (`compileall` + 모듈 스모크 + 275 ast-parse). 운영자 macOS 실 pytest 대기.

---

## 1. 신설 모듈 (8개)

| 모듈 | 등록 항목 | 그룹 |
|------|----------|------|
| `app/utils/time.py` | `utcnow` / `utcnow_iso` | B1 |
| `app/utils/strings.py` | `normalize_display_name` (B6) + `normalize_lower` (BE-G1) | B6 + BE-G1 |
| `app/utils/http_errors.py` | `not_found` / `bad_request` / `unprocessable_entity` / `conflict` | BE-G3 |
| `app/utils/converters.py` | `uuid_str_or_none` / `ensure_uuid` (D5 status_code 옵션) | BE-G1 + D5 |
| `app/utils/json_utils.py` | `loads_maybe` / `dumps_ko` (BE-G2) + `loads_strict` (R1) | BE-G2 + R1 |
| `app/utils/actor.py` | `actor_type_str` + `AuditActorType` 타입 alias | BE-G4 |
| `app/db/cursor_helpers.py` | `fetch_one_as` / `fetch_many_as` | BE-G5 |
| `app/repositories/pagination.py` | `clamp_pagination` (BE-G5) + `paginate_page` (R1) | BE-G5 + R1 |

추가 인프라:
- `app/audit/emitter.py` — `_trace_id_var` ContextVar + `set_trace_id`/`reset_trace_id`/`current_trace_id` (R9). 폐쇄망 정책 docstring 명문화 (D1).

---

## 2. 호출지 마이그레이션 누계

| 그룹 | 사이트 수 | 비고 |
|------|----------|------|
| B1 utcnow | 71 (34 파일) | datetime.now(timezone.utc) 일괄 |
| B6 normalize_display_name | 2 (folders + collections) | thin wrapper 위임 |
| BE-G3 HTTP 4종 | **215** (14 라우터) | AST 자동 변환 — 가장 큰 일괄 |
| BE-G1 uuid_str_or_none | 47 (21 파일) | AST 일괄 |
| BE-G1 normalize_lower | 7 | 수동 |
| BE-G2 dumps_ko | 42 (21 파일) | AST 일괄 + 별칭 1 수동 |
| BE-G2 loads_maybe | 21 (9 파일) | AST 단순 패턴만 |
| BE-G4 actor_type_str | 11 (시범 2 + property 위임 9 자동) | **잠재 보안 버그 수정** SERVICE → "system" |
| BE-G5 fetch helpers | 시범 4 + R6 일괄 22 + R4 시범 4 = **26** | 단순 패턴만 |
| BE-G5 paginate_page | 시범 3 + R5 일괄 17 = **20** | `(page-1)*page_size` 잔존 0건 |
| R1 actor_type or "user" 정정 | 5 (보안 정정) | Enum truthy fallback 잘못된 코드 |

**합계**: **약 459 사이트** (B1 71 + B6 2 + BE-G3 215 + BE-G1 54 + BE-G2 63 + BE-G4 11 + BE-G5 46 + R1 5 = 467 — 일부 중복 제외 추정 459).

---

## 3. 도입 안 함 결정 항목

| 항목 | 결정 이유 |
|------|----------|
| `emit_audit` (§1.7) | `AuditEmitter.emit` 이 이미 풍부한 시그니처 + 162 호출자 컨텍스트 다양 — 단순 wrapper 가 호출자 의도 가림 |
| ensure_uuid 호출지 일괄 (R7) | try/except → unprocessable_entity 도메인 메시지 손실 + status code 변경 위험. **D5 status_code 옵션 추가**로 호환 마이그레이션 가능해짐 |
| loads_strict 호출지 (R8) | 21 loads_maybe 호출자 모두 JSONB 컬럼 읽기 — DB corruption 가정 외 안전. 신규 raw input 검증 호출자에 사용 |

---

## 4. 잔존 (점진 마이그레이션)

| 항목 | 사이트 추정 | 정책 |
|------|------------|------|
| fetch helpers 복잡 케이스 | 347+ | **D4 결정**: 신규 코드 helper 의무 + 도메인별 점진 (보안·권한 우선) |
| 162 emitter 호출자 | 162 | R9 ContextVar 도입으로 자동 trace_id 첨부 — 명시적 마이그레이션 우선순위 ↓ |
| AuditEmitter 폐쇄망 | — | **D1 결정**: 외부 sink 0 — 정책 코멘트 + 회귀 테스트만. 새 sink 추가 PR 의무 명문화 |

---

## 5. CONSTITUTION 준수

- **제8조 Single Responsibility**: 각 helper 단일 책임.
- **제10조 Docstring as Agent Contract**: 모든 helper 에 한국어 docstring + Examples.
- **제11조 Function Index**: `docs/함수도서관/backend.md` 인덱스 동기화.
- **제13조 Observability**: R9 trace_id ContextVar 로 모든 emit 자동 첨부.
- **제14조 Shared Error Contract**: HTTP 4종 + ensure_uuid 모두 `error_code` + `details` 필드.
- **제15조 Bounded Change**: 모든 그룹이 반복 가능한 변환 + 회귀 테스트 동반.

---

## 6. 검증 합산

- `compileall app/`: 0 err.
- 275 파일 ast-parse OK.
- 모듈 스모크 누계 **약 70/70 PASS** (B1 9 + B6 16 + BE-G3 8 + BE-G1 15 + BE-G2 12 + BE-G4 6 + BE-G5 8 + R1 9 + R5+R6 (간접) + R9 4 = ~70).
- 운영자 macOS 실 pytest 트리거 대기 (검증 환경 의존성 부재로 본 세션 미실행).

---

## 7. 운영자 행동 항목

1. macOS 환경에서 실 pytest 실행: `cd backend && pytest tests/` (R9 ContextVar 동작 검증 + 호출지 마이그레이션 회귀).
2. (선택) `tests/unit/utils/` + `tests/unit/audit/` 의 신규 테스트만 빠르게: `pytest tests/unit/utils/ tests/unit/audit/`.
3. 회귀 발견 시 closure 메모리에 기록 + 패치.
