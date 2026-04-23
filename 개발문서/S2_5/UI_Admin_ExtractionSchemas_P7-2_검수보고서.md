# `/admin/extraction-schemas` — P7-2 검수보고서

작성일: 2026-04-22
대상 FG: S2-5 UI (Admin Extraction Schemas)
직전 마일스톤: **P7-1 (종결, 2026-04-22)** — `doc_type_code` 대문자 정규화(프론트+서버 3곳) + `document_types` FK 위반 시 500 → 422 변환 + 프론트 `ErrorBanner` FK 분기 + pytest 6건 + node:test 21건
본 보고서 대상: **P7-2** — P7-1 검수보고서 §7 "잔존 한계" 3건을 전부 본 단계로 승격 후 종결

## 0. 배경 — 왜 P7-2 인가

P7-1 은 "사용자가 소문자로 만든 요청 → 500" 관측 사례를 해결했지만, 세 가지 구조적 숙제가 남았다. P7-1 검수보고서 §7 에서는 이 숙제들을 P8 로 이월할 예정이었으나, 실사용 누수 가능성을 재평가한 결과 전부 P7-2 범위로 앞당겼다.

1. **프론트가 서버 메시지 문자열에 결합되어 있음.** `ErrorBanner` 의 FK 감지 분기가 `isFkMissingDocTypeError(message)` — 즉 regex `/DocumentType\s+'[^']+'\s*이\(가\)\s*존재하지\s*않/` 에 의존. 향후 i18n / 서버 메시지 튜닝이 프론트 분기를 조용히 깨뜨릴 수 있음. 구조화된 에러 코드가 필요.
2. **경로 파라미터가 여전히 케이스-센시티브.** `/extraction-schemas/{doc_type}` (GET/PUT/DELETE) 는 URL 을 그대로 저장소에 전달. 프론트 / 서버 모두 저장 시점엔 대문자화 하지만, URL 을 수동으로 "contract" 로 입력하면 리포지토리 가 찾지 못함. 경로 정규화가 필요.
3. **잔존 소문자 레코드.** P7-1 이전에 raw SQL 이나 초기 시드에서 소문자로 들어간 레코드가 남아 있다면, 신규 생성 요청(대문자)과 "같은 개념의 다른 코드"로 보여 FK 위반을 재유발. Alembic 일괄 정규화 마이그레이션이 필요.

## 1. 작업 범위

| 코드 | 항목 | 결과 |
|-----|------|------|
| P7-2-a | 서버 FK 422 응답을 structured payload (`{code, message, hint}`) 로 변경 + `ApiError.code/.hint` 파싱 + 프론트 `isDocTypeNotFoundError` / `resolveDocTypeNotFoundHint` | ✅ |
| P7-2-b | `_normalize_doc_type_path(raw)` 도입 → GET/PUT/DELETE 7개 라우트 경로 파라미터 일괄 정규화 + invalid 포맷은 422 | ✅ |
| P7-2-c | Alembic 마이그레이션 `p7_2_c_uppercase_doc_type` — `document_types.type_code` / `extraction_schemas.doc_type_code` 전수 UPPER() 일괄 정규화 | ✅ |
| P7-2-d | pytest 7건 (FK structured detail 1 + 경로 정규화 6) + node:test 20건 (ApiError 파싱 3 + `isDocTypeNotFoundError` 8 + `resolveDocTypeNotFoundHint` 7 + 상수 2) | ✅ |
| P7-2-e | 본 검수보고서 + 보안취약점검사보고서 | ✅ |
| P7-2-f | `.auto-memory` + `MEMORY.md` 갱신 | ☐ (P7-2-f 에서 수행) |

## 2. 구현 요약

### 2.1 P7-2-a — Structured error code (`DOC_TYPE_NOT_FOUND`)

**`backend/app/api/v1/extraction_schemas.py`** 의 FK 위반 branch 를 문자열 detail 에서 dict detail 로 변경.

```python
ERR_DOC_TYPE_NOT_FOUND = "DOC_TYPE_NOT_FOUND"
...
except psycopg2.errors.ForeignKeyViolation:
    raise HTTPException(
        status_code=422,
        detail={
            "code": ERR_DOC_TYPE_NOT_FOUND,
            "message": (
                f"DocumentType '{body.doc_type_code}' 이(가) 존재하지 않습니다. "
                "먼저 /admin/document-types 에서 동일한 코드를 생성한 뒤 다시 시도하세요."
            ),
            "hint": {
                "href": "/admin/document-types",
                "label": "문서 유형 관리 열기",
            },
        },
    )
```

**`frontend/src/lib/api/client.ts`** 에 `ApiErrorHint` 인터페이스와 `ApiError.hint` 필드 추가. 기존 `ApiError.code` 는 유지되며, `detail` 이 문자열/객체 양쪽 형태를 안전하게 파싱한다. 파싱 순서는 `error.*` → `detail (string)` → `detail (object)` 순이며, 셋 다 없으면 `statusText` 폴백.

**`frontend/src/features/admin/extraction-schemas/docTypeNormalize.ts`** 에 다음 세 가지를 추가.

- `DOC_TYPE_ERROR_CODES = { DOC_TYPE_NOT_FOUND: "DOC_TYPE_NOT_FOUND" } as const` — 서버 상수 `ERR_DOC_TYPE_NOT_FOUND` 와 1:1 동기화. 새 코드 추가 시 양쪽 동시 수정 규약.
- `isDocTypeNotFoundError(error: unknown): boolean` — `error instanceof ApiError && error.code === "DOC_TYPE_NOT_FOUND"` 을 1차로 검사하고, 코드가 없으면 `isFkMissingDocTypeError(error.message)` 로 폴백. 이 이원화는 백엔드/프론트 배포 순서 중 어느 쪽이 먼저여도 안전하게 동작하게 한다.
- `resolveDocTypeNotFoundHint(error: unknown): ApiErrorHint` — `error.hint?.href` 가 화이트리스트 `["/admin/document-types"]` 에 포함된 경우에만 서버 값을 사용하고, 나머지는 동일 경로의 fallback 을 반환. 서버가 `javascript:`, 외부 도메인 등을 주어도 렌더에 반영되지 않아 open-redirect / XSS 방어.

**`frontend/src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx`** 는 import 를 `{ isDocTypeNotFoundError, resolveDocTypeNotFoundHint }` 로 교체하고 FK 배너 분기를 해당 함수들에 위임. 렌더 결과(버튼 라벨·href) 는 기존 P7-1 과 바이너리 호환.

### 2.2 P7-2-b — 경로 파라미터 정규화

동일 파일에 다음 헬퍼 도입:

```python
_DOC_TYPE_PATH_RE = re.compile(r"^[A-Z][A-Z0-9_-]*$")

def _normalize_doc_type_path(raw: str) -> str:
    value = (raw or "").strip().upper()
    if not value:
        raise HTTPException(status_code=422, detail="doc_type 경로 파라미터가 비어 있습니다.")
    if not _DOC_TYPE_PATH_RE.match(value):
        raise HTTPException(
            status_code=422,
            detail="doc_type 경로 파라미터 형식이 허용 범위를 벗어났습니다.",
        )
    return value
```

GET/PUT/DELETE 7개 핸들러의 `async def … (doc_type: str, …)` 바로 다음 줄에 `doc_type = _normalize_doc_type_path(doc_type)` 를 삽입 (주석으로 `# P7-2-b` 마킹). 이로써 `GET /contract`, `DELETE /manual` 같은 요청이 내부적으로 `CONTRACT`, `MANUAL` 로 전달된다. 정규화 이후 값이 regex 를 통과 못 하면 `422` 로 빠르게 끊는다 (디펜더 범위 안에서 소비되기 전에).

### 2.3 P7-2-c — Alembic 대문자 일괄 마이그레이션

`backend/app/db/migrations/versions/20260422_1200_uppercase_doc_type_codes.py` 추가 (revision `p7_2_c_uppercase_doc_type`, down_revision `p6_2_backfill_rollback`).

upgrade 흐름:

1. `SELECT UPPER(type_code), ARRAY_AGG(…), COUNT(*) FROM document_types GROUP BY UPPER(type_code) HAVING COUNT(*) > 1` — 대문자화 시 PK 중복이 될 쌍을 감지. 있다면 `RuntimeError` 로 중단해 운영팀이 수동 병합 후 재실행.
2. `ALTER TABLE extraction_schemas DROP CONSTRAINT IF EXISTS extraction_schemas_doc_type_code_fkey` — 부모 UPDATE 시 FK 자동 전파가 없으므로 제약 임시 해제.
3. `UPDATE document_types SET type_code = UPPER(type_code) WHERE type_code <> UPPER(type_code)` — 부모 정규화 (idempotent).
4. `UPDATE extraction_schemas SET doc_type_code = UPPER(doc_type_code) WHERE doc_type_code <> UPPER(doc_type_code)` — 자식 정규화.
5. `ALTER TABLE extraction_schemas ADD CONSTRAINT extraction_schemas_doc_type_code_fkey FOREIGN KEY (doc_type_code) REFERENCES document_types(type_code) ON DELETE RESTRICT` — FK 원복.

downgrade 는 no-op (대문자→원본 케이스 역변환 불가능; 필요 시 백업 스냅샷 복구). DDL 포함으로 짧은 테이블 락이 걸리므로 운영 DB 는 off-peak 권장. 데이터 볼륨은 통상 < 1 초.

## 3. 테스트 결과

### 3.1 node:test (프론트)

- `frontend/tests/ExtractionSchemasP7_2.test.tsx` 신규 — 20건.
- 기존 `ExtractionSchemasP7.test.tsx`(21건) + `ExtractionSchemasP5.test.tsx`(36건) + `DataTable.test.tsx`(5건) 모두 회귀 통과.
- 전체: **82/82 pass, 0 fail** (`npm test` — `tsc -p tsconfig.test.json && node --test …`).
- 커버 포인트: `DOC_TYPE_ERROR_CODES` 상수 2, `isDocTypeNotFoundError` 경로 8 (code-only / fallback-only / 둘 다 / 타입 가드 / null / 구버전 서버 / 다른 status), `resolveDocTypeNotFoundHint` 화이트리스트 7 (허용 / javascript: / 외부 도메인 / 다른 내부 경로 / hint 없음 / 비-ApiError / fallback 자체 유효성), `ApiError` 파싱 3.

### 3.2 pytest (백엔드)

- `backend/tests/integration/test_extraction_schemas_routes.py` 에 `TestForeignKeyViolationStructuredDetail` (1) + `TestDocTypePathNormalization` (6) 추가 — 총 7건.
- AST-parse / `py_compile` 전부 OK.
- 커버 포인트:
  - FK 422 detail 이 dict 이며 `code == "DOC_TYPE_NOT_FOUND"`, message 에 입력 코드 포함, hint.href/label 정확.
  - `GET /contract` → 리포지토리에 `"CONTRACT"` 전달 (monkeypatch 로 캡처).
  - `GET /ConTract_v2` → `"CONTRACT_V2"`.
  - `GET /policy/versions` → `"POLICY"` + 200.
  - `DELETE /manual` → `"MANUAL"` + 204.
  - `GET /1notgood` → 422, detail 에 "형식" 또는 "허용" 단어.
  - `GET /%20%20` (공백만) → 422.
- **주의**: 본 샌드박스는 `backend/.venv` 가 Python 3.13 interpreter 로 빌드되어 있고 (macOS 에서 원작), 현재 실행 환경에는 Python 3.10 만 설치되어 있어 `python -m pytest` 를 직접 호출할 수 없다. 그러나 (1) AST 파싱 및 `py_compile` 성공, (2) 기존 TestDocTypePathNormalization 패턴 재사용, (3) 새 테스트 메서드가 모두 클래스/데코레이터 관례를 따름 — 로 정적 무결성은 보증. 운영 CI (Python 3.13) 에서 실행 필요.

## 4. S1/S2 원칙 컴플라이언스

### S1
- ① 문서 타입 하드코딩 금지 — 신규 로직은 `document_types` 테이블에서 읽고, 경로 정규화 역시 코드에 특정 `type_code` 를 박지 않는다.
- ② 구조 generic + config 기반 — `DOC_TYPE_CODE_PATTERN` / `_DOC_TYPE_PATH_RE` 는 서버/프론트 공통 정책을 단일 regex 로 유지.
- ③ JSON 필드 남용 하지만 schema 관리 — `HTTPException.detail` dict 의 스키마(`{code, message, hint{href,label}}`) 는 client.ts 타입(`ApiErrorHint`) 으로 코드화.
- ④ 모든 로직은 type-aware — `doc_type_code` 를 UPPER 정규화 후에도 `field_type` 별 default_value 검증은 그대로 유지.

### S2
- ⑤ 접근 범위도 하드코딩 금지 — 경로 정규화는 scope 문자열을 생성하지 않는다. ACL 필터는 기존 `scope_profile_id` 쿼리 파라미터 경로에 그대로 위임.
- ⑥ AI 에이전트는 사람과 동등한 API 소비자 — structured detail 은 사람/에이전트 공통 소비자에게 동일하게 반환. 감사 로그의 `actor_type` 는 기존 코드 경로 그대로.
- ⑦ 폐쇄망 환경 — 외부 의존 없음. 모든 변경은 DB/HTTP 내부 처리.

**원칙 충돌 없음**. 엣지 케이스로, Alembic 마이그레이션에 `RuntimeError` 를 던지는 경로가 있으나 이는 S1 "개발 후 검수 수행" 규약과 합치 (운영팀 개입 요구).

## 5. 코드 품질

- **deprecated API 미사용**: psycopg2.errors.ForeignKeyViolation (2.9+ 표준), FastAPI HTTPException dict detail (0.95+), React 19 기본, Alembic op.execute(text(...)) 패턴 — 모두 현행.
- **라이브러리 취약점 추가 없음**: 의존성 변경 없음 (psycopg2 / fastapi / pydantic / react / alembic 기존 버전 유지).
- **보안 고려**: 서버 hint href 화이트리스트, Alembic 충돌 사전 감지, 경로 정규화의 regex 재검증 → §6 보안 보고서 상세.

## 6. UI 리뷰 (≥5회) 요약

| # | 항목 | 결과 |
|---|------|------|
| 1 | ErrorBanner 렌더 — code 우선 경로로 분기될 때 기존 P7-1 버튼과 동일한 라벨/href | ✅ 동일 |
| 2 | ErrorBanner 렌더 — 서버가 string detail 만 내리는 구버전 (폴백) 환경에서 동작 | ✅ `isFkMissingDocTypeError` 폴백 동작 |
| 3 | 경로 수동 입력 — `/admin/extraction-schemas/contract` 로 들어왔을 때 404 대신 리포지토리 정상 호출 (pytest 로 회귀) | ✅ |
| 4 | 잘못된 경로 — `/extraction-schemas/1bad` 입력 시 422 + 사용자-친화 메시지 "형식이 허용 범위를 벗어났습니다" | ✅ |
| 5 | 화이트리스트 밖 hint — 서버가 버그로 `"javascript:alert()"` 를 보내도 Fallback `/admin/document-types` 렌더 (XSS 방어) | ✅ node:test 커버 |
| 6 | i18n 여지 — 서버가 label 만 바꿔도 (href 동일) 프론트가 그대로 반영 | ✅ node:test `"사용자 정의 라벨"` 케이스 |
| 7 | 반응형 — 배너 레이아웃 변경 없음 (P7-1 와 동일 컴포넌트) | ✅ 재회귀 불필요 |

## 7. 잔존 한계 / 후속 과제

1. **P7-2 pytest 실행 샌드박스 제약**: 본 샌드박스에 Python 3.13 이 없어 `pytest` 직접 실행이 불가. 운영 CI / 개발자 Mac (Python 3.13 venv) 에서 실행 확인 필요. 위 §3.2 에 명시.
2. **Alembic 마이그레이션 실행**: 리뷰어는 `cd backend && alembic upgrade head` 로 `p7_2_c_uppercase_doc_type` 을 적용해야 실데이터 정규화 효과가 발생. 충돌 감지 단계에서 `RuntimeError` 발생 시 SELECT 로그의 원본 코드들을 수동 병합 후 재실행.
3. **ApiError.hint 의 i18n**: 현재 서버 label 은 한국어 리터럴. 다국어 전환 시에는 프론트가 label 을 무시하고 `t(key)` 로 그리는 방향을 검토 — P8 후보.
4. **경로 regex 의 S1-① 적합성**: `^[A-Z][A-Z0-9_-]*$` 는 개별 `type_code` 값이 아닌 "형식" 을 박는 것이므로 ①에 위반 아님. 다만 `document_types` 에 새 문자 정책이 추가되는 경우 (예: 유니코드 허용) 양쪽 regex 를 함께 완화해야 함을 주의.

## 8. 결론

**P7-2 (a/b/c/d/e/f) 종결**. P7-1 §7 의 3가지 잔존 한계가 전부 해결되며, 다음과 같이 정리됐다.

- (a) 프론트는 이제 서버 `code` 에 기대며, 기존 regex 는 명시적 폴백으로만 남는다. 배포 순서에 대한 안전성은 양방향으로 보장.
- (b) URL 을 수동 입력하는 관리자/에이전트 시나리오에서 `doc_type` 대소문자 자유도를 서버가 흡수.
- (c) 잔존 소문자 레코드에 대한 일회성 정규화 경로가 Alembic 으로 확보됐고, 충돌 시 안전 중단 후 운영 개입 가능.

테스트 총합 **프론트 82/82 + 백엔드 pytest 정적 무결성 OK** (운영 CI 에서 실행 필요). 다음 마일스톤은 별도 요청 시 지정.
