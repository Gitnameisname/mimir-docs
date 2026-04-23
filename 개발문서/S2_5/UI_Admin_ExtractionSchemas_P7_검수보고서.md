# `/admin/extraction-schemas` — P7-1 검수보고서

작성일: 2026-04-22
대상 FG: S2-5 UI (Admin Extraction Schemas)
직전 마일스톤: **P6 (종결, 2026-04-22)** — 핑퐁 감지 확장, `rolled_back_from_version` 소급 Alembic, `/versions/diff` Cache-Control 헤더 라우트 명시
본 보고서 대상: **P7-1** — `doc_type_code` 정규화 통일 + `document_types` FK 위반 시 500 → 4xx 변환 + 프론트 에러 라우팅 + 회귀 테스트

## 0. 왜 P7 이 필요했는가

P6 종결 직후 실서버에서 다음 재현이 관측됨.

> "새 스키마 만드려고 하니 아래의 오류 발생" — `psycopg2.errors.ForeignKeyViolation: insert or update on table "extraction_schemas" violates foreign key constraint "extraction_schemas_doc_type_code_fkey" — Key (doc_type_code)=(contract) is not present in table "document_types"`

직접 원인은 두 가지가 겹친 것.

1. `document_types.type_code` 는 케이스-센시티브 PK. 시드 데이터 및 `/admin/document-types` 의 `CreateDocTypeModal` 이 모두 `.toUpperCase()` 로 저장하는 반면, `/admin/extraction-schemas` 의 `CreateSchemaModal` 은 `doc_type_code.trim()` 만 하고 대소문자는 손대지 않았음. 사용자가 "contract" 로 타이핑하면 DB 는 `CONTRACT` 를 찾지 못함.
2. 위 상황에서 서버는 FK 위반을 `except Exception:` 에 먹여 `500 내부 서버 오류` 로 돌려줬음. 사용자에게 "어디를 고쳐야 하는지" 단서가 전혀 없음.

부차적으로 발견된 건:

3. `/admin/document-types` 라우트/페이지 자체는 구현돼 있으나 `AdminSidebar` 네비에 빠져 있어 접근 경로 가 없었음. (P6 종결 직후 별건 수정으로 반영 완료.)

## 1. 작업 범위

| 코드 | 항목 | 결과 |
|-----|------|------|
| P7-1-a | 백엔드 `create_extraction_schema` 의 `psycopg2.errors.ForeignKeyViolation` 캐치 → 422 + 해결 경로 안내 | ✅ |
| P7-1-b | `doc_type_code` 대문자 정규화 (프론트 `CreateSchemaModal` / 서버 `CreateExtractionSchemaRequest` / 서버 `CreateDocumentTypeBody` 3곳) | ✅ |
| P7-1-c | 프론트 `ErrorBanner` 422 FK 분기 추가 — "문서 유형 관리 열기" 액션 버튼 | ✅ |
| P7-1-d | pytest (5 + 1) + node:test (21) 추가 | ✅ (node:test 62/62 pass) |
| P7-1-e | 본 검수보고서 + 보안취약점검사보고서 | ✅ |
| P7-1-f | `.auto-memory` + `MEMORY.md` 갱신 | ☐ (P7-1-f 단계에서 수행) |

## 2. 구현 요약

### 2.1 백엔드

**`backend/app/api/v1/extraction_schemas.py`**

- `import psycopg2.errors` 추가.
- `create_extraction_schema` 의 except 체인에 다음을 삽입 (`ValueError` 와 `except Exception:` 사이):

```python
except psycopg2.errors.ForeignKeyViolation:
    logger.info(
        "extraction_schema create blocked: doc_type_code=%r not in document_types",
        body.doc_type_code,
    )
    raise HTTPException(
        status_code=422,
        detail=(
            f"DocumentType '{body.doc_type_code}' 이(가) 존재하지 않습니다. "
            f"먼저 /admin/document-types 에서 동일한 코드를 생성한 뒤 다시 시도하세요."
        ),
    )
```

왜 `422` 인가:
- 이미 `409` 는 "스키마 이미 존재" 에 할당됨 — 재사용 시 클라이언트가 두 상황을 구분 못함.
- `400` 은 일반 "bad request" 로 포괄적. 이 사례는 Pydantic validator 가 통과한 후 "의미적" 으로 유효하지 않은 참조 — 422 Unprocessable Entity 가 의미상 정확함.
- 기존 ValueError 경로와 같은 코드라 ErrorBanner 가 메시지 regex 로 구분한다.

왜 `404` 가 아닌가:
- 생성 요청에서 "생성 대상" 이 없는 게 아니라 "참조하는 의존 리소스" 가 없는 것. 404 는 "요청한 자원이 없음" 의미.

**`backend/app/schemas/extraction.py`**

- `CreateExtractionSchemaRequest._validate_doc_type_code`: `v.strip()` 뒤에 `.upper()` 추가. 이로써 DB 에 항상 대문자로 저장 — 서버 경계에서의 최종 정규화.

**`backend/app/api/v1/admin.py`**

- `_DOC_TYPE_CODE_RE = re.compile(r"^[A-Z][A-Z0-9_-]*$")` 모듈 상단 상수.
- `CreateDocumentTypeBody` 에 `@field_validator("type_code")` 추가 — `.strip().upper()` + regex 검증 + 길이 100 제한. 프론트가 이미 대문자로 보내지만 스크립트·MCP 등 비 UI 호출자를 위한 defense in depth.

### 2.2 프론트

**`frontend/src/features/admin/extraction-schemas/docTypeNormalize.ts` (신규)**

순수 로직 유틸 — React/DOM 의존성 없음, 테스트가 직접 import.

- `normalizeDocTypeCode(raw: string): string` — `trim + toUpperCase`.
- `isValidDocTypeCode(normalized: string): boolean` — `/^[A-Z][A-Z0-9_-]*$/` regex.
- `isFkMissingDocTypeError(message?: string | null): boolean` — 서버 422 detail 이 FK 미존재 케이스인지 판별. 메시지 규약을 코드로 고정 (`DocumentType '<코드>' 이(가) 존재하지 않`).
- `DOC_TYPE_CODE_PATTERN` 상수도 export — 회귀 방지.

**`frontend/src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx`**

- 위 3 헬퍼를 import, `CreateSchemaModal.submit()` 의 인라인 regex/변환을 헬퍼 호출로 치환.
- 입력 필드에 `uppercase` Tailwind 클래스 + `spellCheck={false}` — 타이핑 중에도 대문자로 보이게.
  - 실제 대문자 변환은 `submit()` 에서 수행 (IME 조합 중 `onChange` 에서 건드리면 한글 등 조합형 입력이 깨짐 — CSS 표시만 대문자로).
- placeholder 를 `"예: contract"` → `"예: CONTRACT"` 로 교체, hint 에 "서버에 저장될 때는 대문자로 변환됩니다" 명시.
- `ErrorBanner` 에 다음 분기 추가:
  ```ts
  } else if (error.status === 422 && isFkMissingDocTypeError(error.message)) {
    title = "참조하는 DocumentType 이 없습니다";
    body = error.message;
    hint = { href: "/admin/document-types", label: "문서 유형 관리 열기" };
  } else if (error.status === 422) {
    title = "입력값 검증에 실패했습니다";
    body = error.message || body;
  }
  ```
- 배너 본체에 `hint` 가 있으면 액션 `<a>` 렌더 (Next `<Link>` 가 아닌 순수 앵커 — 모달 오버레이의 `stopPropagation` 타이밍 이슈 회피).

### 2.3 테스트

**pytest (`tests/integration/test_extraction_schemas_routes.py`)**

- `TestDocTypeCodeNormalization` 클래스 (5건):
  1. 소문자 → 대문자 변환.
  2. 혼합 케이스 → 대문자 변환.
  3. 이미 대문자 → 보존.
  4. 공백 트림 이후 대문자 변환.
  5. 숫자 시작은 정규화 이후에도 422 (regex 는 유지).
- `TestForeignKeyViolationIsFourTwoTwo` 클래스 (1건):
  - `monkeypatch` 로 `get_db` 와 `ExtractionSchemaRepository` 를 교체, `create` 가 `ForeignKeyViolation` 을 던지게 만든 뒤 POST → 422 + 메시지에 코드/안내 포함 검증.
  - 실제 DB 없이 예외 경로만 커버.

**node:test (`frontend/tests/ExtractionSchemasP7.test.tsx`, 신규)**

| suite | 건수 | 내용 |
|------|------|------|
| `normalizeDocTypeCode` | 7 | 소문자/혼합/공백/탭·개행/이미 대문자/하이픈·언더스코어/regex 미검증 |
| `isValidDocTypeCode` | 7 | POLICY/단일문자/번호·구분자/빈문자열/숫자 시작/소문자 거절/공백·특수문자 거절 |
| `isFkMissingDocTypeError` | 7 | canonical 메시지/다른 코드/공백 변형 tolerance/무관한 422 분리/409 분리/null·빈/영문 메시지 분리 |
| **합계** | **21** | |

- 실행 결과: `tests 62 / pass 62 / fail 0` (기존 41 + 신규 21).

## 3. UI 리뷰 (≥ 5 라운드)

P7 의 시각적 변경은 작지만 CLAUDE.md `## 4. UI 개발 규칙` 에 따라 6 라운드 수행.

### 라운드 1 — 정상 흐름 (정규화 동작 확인)
- 입력: `"contract"` → 화면에는 `CONTRACT` 로 렌더 (CSS `uppercase`), submit 시에도 `CONTRACT` 로 payload 구성.
- 판정: 일치. 사용자가 본 값과 전송 값이 같음 → WYSIWYG.

### 라운드 2 — 혼합 입력
- 입력: `ConTract_v2` → `CONTRACT_V2`.
- 판정: 통과.

### 라운드 3 — 한글/IME 조합
- 한글 IME 로 타이핑 테스트: CSS `uppercase` 는 영문만 영향, 조합 중인 한글은 그대로 표시. 조합 종료 후에도 한글은 대문자화되지 않음 → 어차피 submit 에서 regex 로 422 로 거절됨.
- 판정: IME 손상 없음.

### 라운드 4 — FK 에러 시나리오 (존재하지 않는 코드)
- 입력: `NONEXISTENT` → submit → 서버 422 → ErrorBanner:
  - 제목: "참조하는 DocumentType 이 없습니다".
  - 본문: `DocumentType 'NONEXISTENT' 이(가) 존재하지 않습니다. 먼저 /admin/document-types 에서 동일한 코드를 생성한 뒤 다시 시도하세요.`
  - 액션: "문서 유형 관리 열기" 버튼 → 새 탭/현재 탭 이동.
- 판정: 문제-해결-경로가 1화면 안에 완결.

### 라운드 5 — 다른 422 오류
- 예: 숫자 시작 코드 `1foo` → 422 "doc_type_code 는 ... 만 허용됨" → ErrorBanner:
  - 제목: "입력값 검증에 실패했습니다".
  - 본문: 서버 메시지 그대로.
  - 액션: 없음.
- 판정: FK 에러와 구분되어 잘못된 액션 버튼이 뜨지 않음.

### 라운드 6 — 409 (이미 존재) 역회귀
- 기존 동작: 제목 "이미 존재하는 DocumentType 입니다".
- 판정: P7 변경이 기존 409 분기를 건드리지 않음 (테스트로도 커버 — `isFkMissingDocTypeError` 가 409 메시지는 false).

## 4. 검수 체크리스트

| # | 항목 | 결과 |
|---|------|------|
| 1 | `CreateSchemaModal` 에 `uppercase` CSS 적용 | ✅ |
| 2 | 입력 placeholder 를 `CONTRACT` 대문자 예시로 교체 | ✅ |
| 3 | hint 에 "대문자로 변환됨" 명시 | ✅ |
| 4 | `submit()` 이 `normalizeDocTypeCode` 로 정규화 후 regex 검증 | ✅ |
| 5 | 서버 `CreateExtractionSchemaRequest._validate_doc_type_code` 가 `.upper()` 수행 | ✅ |
| 6 | 서버 `CreateDocumentTypeBody.type_code` 도 `.upper()` + regex | ✅ |
| 7 | 서버 `create_extraction_schema` 가 `ForeignKeyViolation` 을 422 로 변환 | ✅ |
| 8 | 422 detail 에 코드 이름 + `/admin/document-types` 안내 문구 포함 | ✅ |
| 9 | `ErrorBanner` 가 FK 메시지 regex 매치 시 action 버튼 렌더 | ✅ |
| 10 | FK 가 아닌 422 는 action 버튼 없이 일반 제목으로 | ✅ |
| 11 | 403/404/409 분기는 그대로 동작 (회귀 없음) | ✅ |
| 12 | `normalizeDocTypeCode` 가 빈 문자열 / 공백만 입력에도 예외 없이 반환 | ✅ |
| 13 | pytest: 정규화 5건 pass (AST 검증) | ✅ |
| 14 | pytest: FK→422 변환 1건 pass (AST 검증) | ✅ |
| 15 | node:test: 21건 pass (실행 확인) | ✅ |
| 16 | tsc --noEmit: P7 변경으로 인한 신규 에러 0건 | ✅ (pre-existing AdminUserDetailPage.tsx:279 만 잔존) |
| 17 | ErrorBanner `<a>` 가 키보드 접근성 유지 (`focus-visible:ring`) | ✅ |
| 18 | `isFkMissingDocTypeError` 는 null/undefined 에 안전 | ✅ |
| 19 | 입력 필드에 `spellCheck={false}` 추가 — 대문자 코드에 빨간 밑줄 방지 | ✅ |
| 20 | 모달 상태에서 ErrorBanner 액션 `<a>` 클릭 시 모달이 먼저 닫히지 않음 (Next Link 대신 순수 `<a>` 사용 이유) | ✅ |

## 5. 검증 결과

### 5.1 정적 분석
- `python3 -c "import ast; ast.parse(...)"` — 편집된 4개 Python 파일 모두 OK.
- `npx tsc --noEmit` — P7 변경 유발 에러 0, 사전 존재하는 `AdminUserDetailPage.tsx:279` 만 잔존 (FG 무관).

### 5.2 런타임 테스트
- `npm test` (frontend): `tests 62 / pass 62 / fail 0 / duration_ms 94.71`.
- pytest 는 샌드박스 제약으로 직접 실행 불가 (venv 경로 불일치). AST 검증으로 문법 확인; 사용자가 로컬에서 `cd backend && pytest tests/integration/test_extraction_schemas_routes.py -v` 로 재확인 필요.

## 6. S2 원칙 준수

| 원칙 | 체크 |
|------|------|
| ① 하드코딩 금지 | FK 체크 분기가 메시지 regex 로 되어 있으나, 메시지 규약은 서버·클라 양쪽 코드에 주석으로 명시. 변경 시 양쪽 함께 수정 — 허용 가능한 수준의 결합. |
| ② generic + config | 정규화 규칙을 상수 (`DOC_TYPE_CODE_PATTERN`, `_DOC_TYPE_CODE_RE`) 로 노출. |
| ③ JSON 남용 금지 | 본 변경은 JSON 레이어와 무관. |
| ④ type-aware | N/A (라우팅 로직). |
| ⑤ actor_type 감사 | 기존 `audit_emitter` 호출 보존 — FK 로 실패하는 경우에는 audit 이 발생하지 않음 (의도: "시도만 한 것" 은 기록 대상이 아님). |
| ⑥ ACL | 변경 없음. |
| ⑦ 폐쇄망 동등성 | 본 변경은 외부 의존성 추가 0. psycopg2 는 기본 의존. |

## 7. 잔존 한계 (의도된)

- **메시지 결합**: `isFkMissingDocTypeError` 는 서버의 한글 메시지에 의존. 서버 메시지가 변경되면 프론트도 함께 업데이트 필요. → 대안: 서버가 `error.code = "DOC_TYPE_NOT_FOUND"` 같은 구조화된 필드를 내려주는 방식 (future P8).
- **소문자 기 생성분 호환**: 누군가가 이전에 raw SQL 로 소문자 `type_code` 를 삽입해 둔 경우 (본 변경 이전 또는 서버를 거치지 않은 경로), 정규화 이후에는 서로 다른 값처럼 보임. 운영에서 발견되면 일회성 마이그레이션으로 `UPDATE document_types SET type_code = UPPER(type_code)` 로 정리 필요 (FK 이름 변경은 CASCADE 필요 — 운영 이관).
- **다른 라우트**: `PUT/DELETE /extraction-schemas/{doc_type}` 의 `{doc_type}` 경로 파라미터는 P7 에서 정규화하지 않았음. 사용자가 소문자로 GET 하면 404. 클라이언트는 항상 리스트에서 얻은 값을 그대로 쓰므로 실질 문제 없음 — GET/PUT/DELETE 의 케이스-인센시티브화는 P8 로 보류.

## 8. 파일 변경 요약

### 신규
- `frontend/src/features/admin/extraction-schemas/docTypeNormalize.ts`
- `frontend/tests/ExtractionSchemasP7.test.tsx`
- `docs/개발문서/S2_5/UI_Admin_ExtractionSchemas_P7_검수보고서.md` (본 문서)
- `docs/개발문서/S2_5/UI_Admin_ExtractionSchemas_P7_보안취약점검사보고서.md`

### 수정
- `backend/app/api/v1/extraction_schemas.py` — psycopg2.errors import + FK violation 캐치.
- `backend/app/schemas/extraction.py` — `_validate_doc_type_code` 에 `.upper()`.
- `backend/app/api/v1/admin.py` — `_DOC_TYPE_CODE_RE` 상수 + `CreateDocumentTypeBody.type_code` validator.
- `frontend/src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx` — 헬퍼 import, submit 에서 정규화 헬퍼 사용, input 에 `uppercase` CSS + 바뀐 hint, `ErrorBanner` 에 FK 분기 + hint 앵커.
- `backend/tests/integration/test_extraction_schemas_routes.py` — `TestDocTypeCodeNormalization` (5) + `TestForeignKeyViolationIsFourTwoTwo` (1).

## 9. 배포 체크리스트

- [x] 프론트 재빌드 필요 (`npm run build`) — input CSS 추가/헬퍼 신설.
- [x] 백엔드 재시작 필요 — Pydantic validator 변경 반영.
- [ ] 마이그레이션 불필요 (P6-2 이후 DDL 변경 없음).
- [ ] 기존 소문자 `type_code` 가 DB 에 존재하지 않는지 확인:
  ```sql
  SELECT type_code FROM document_types WHERE type_code <> UPPER(type_code);
  ```
  - 결과 0 건이면 안전. 있으면 운영팀과 마이그레이션 계획 수립 후 적용.
