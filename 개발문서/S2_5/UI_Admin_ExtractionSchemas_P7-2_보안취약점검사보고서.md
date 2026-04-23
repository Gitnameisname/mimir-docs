# `/admin/extraction-schemas` — P7-2 보안취약점검사보고서

작성일: 2026-04-22
대상 FG: S2-5 UI (Admin Extraction Schemas)
대상 마일스톤: **P7-2 (a/b/c/d/e/f)**
연계 검수보고서: `UI_Admin_ExtractionSchemas_P7-2_검수보고서.md`

## 0. 범위

P7-2 에서 추가·변경된 코드/데이터 경로에 대해 OWASP Top-10 관점 + 내부 S1/S2 정책 관점에서 취약점 검사. 변경 대상:

1. `backend/app/api/v1/extraction_schemas.py` — FK 위반 branch 의 structured detail, `_normalize_doc_type_path` 헬퍼, 7개 라우트에 헬퍼 호출 삽입.
2. `frontend/src/lib/api/client.ts` — `ApiError.hint` 확장, `detail` 객체 파싱.
3. `frontend/src/features/admin/extraction-schemas/docTypeNormalize.ts` — `DOC_TYPE_ERROR_CODES`, `isDocTypeNotFoundError`, `resolveDocTypeNotFoundHint` 신설.
4. `frontend/src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx` — 배너 분기 교체 (렌더 동일).
5. `backend/app/db/migrations/versions/20260422_1200_uppercase_doc_type_codes.py` — Alembic 일괄 대문자화.
6. 테스트: `frontend/tests/ExtractionSchemasP7_2.test.tsx`, `backend/tests/integration/test_extraction_schemas_routes.py` 확장.

## 1. 위협 모델

P7-2 가 도입한 새 신뢰 경계는 두 가지다.

- **A) 서버→클라이언트 structured error payload**: 서버가 HTTPException detail 을 dict 로 내려주는 경로가 새로 생겼다. 프론트는 이 중 특정 필드(`hint.href`) 를 DOM 의 `href` 로 렌더할 가능성이 있어, 서버가 감염된 경우 Open Redirect / `javascript:` XSS / 피싱 URL 의 전달 수단이 될 수 있다.
- **B) URL 경로 파라미터의 사용자 입력 경로**: `doc_type` 문자열이 이전에는 라우트 인자 그대로 SQL 파라미터 바인딩(psycopg2) 까지 전달됐는데, 이제 전처리(`_normalize_doc_type_path`) 를 거친다. 전처리 자체가 새 attack surface 인지 검사.

데이터 레벨 신뢰 경계로는:

- **C) Alembic 마이그레이션의 SQL**: 하드 코딩된 SQL fragments (변수 치환 없음) 이지만 `ALTER TABLE` / `DROP CONSTRAINT` / `UPDATE` 는 고권한 연산. 운영자 실수로 잘못된 시점에 실행되면 FK 무결성이 일시 깨질 수 있다.

## 2. 취약점 항목별 검사

### 2.1 Open Redirect / Reflected XSS via `hint.href` — A 범주

**공격 시나리오**: 서버가 (버그 / 악의적 배포 / 중간자 / 로그 인젝션) 으로 `detail.hint.href = "javascript:alert(document.cookie)"` 또는 `= "https://evil.example/phish?u=<email>"` 를 응답. 프론트가 이 값을 `<a href>` 로 렌더하면 클릭 시 XSS 또는 외부 피싱.

**대응**:

- `resolveDocTypeNotFoundHint(error)` 가 `const allowed = new Set<string>(["/admin/document-types"])` 화이트리스트 대조. 일치 시에만 서버 값 사용, 그 외는 모두 fallback `{href:"/admin/document-types", label:"문서 유형 관리 열기"}` 를 반환.
- 화이트리스트는 문자열 완전 일치 (trailing slash, query string 등 변형 모두 차단). `javascript:`, `data:`, `file:`, 프로토콜-릴레이티브 `//evil`, 외부 절대 URL, 다른 내부 경로 모두 fallback 으로 귀결.
- `AdminExtractionSchemasPage.tsx` 는 `resolveDocTypeNotFoundHint` 결과만 href 로 사용; 이 함수를 우회하는 hint 접근 없음.

**테스트 커버**: `ExtractionSchemasP7_2.test.tsx > resolveDocTypeNotFoundHint` 의 7건 중 4건이 악성 href 및 변형을 fallback 으로 해석하는지 회귀. 결과 **PASS**.

**결론**: Open Redirect / `javascript:` XSS **낮음** (화이트리스트 방어 유효).

### 2.2 CWE-20 Improper Input Validation — B 범주

**공격 시나리오**: 경로 파라미터로 SQL injection, path traversal, 제어문자, 극단 길이 문자열 투입.

**대응**:

- `_normalize_doc_type_path` 는 `(raw or "").strip().upper()` 후 `^[A-Z][A-Z0-9_-]*$` regex 로 재검증. SQL meta-character (`'`, `;`, `--`, `\``) / path separator (`/`, `\`) / 공백 / null byte (`\0`) / 다국어 문자는 전부 regex 거부 → 422.
- 정규화 실패는 `HTTPException(status_code=422, …)` 로 일관된 400번대로 응답. 내부 상세(스택트레이스, SQL) 노출 없음.
- 정규화 이후 값은 기존 repository 계층의 prepared statement ($1 바인딩) 를 거쳐 DB 로 전달되므로, 이 경로에서 SQL injection 은 이중 방어 (regex + prepared statement).

**회귀**:

- pytest `TestDocTypePathNormalization.test_invalid_path_returns_422` / `test_whitespace_only_path_rejected` 두 건이 422 응답과 메시지 키워드를 검증.
- `py_compile` / AST-parse OK.

**결론**: Injection / path traversal **낮음** (regex + prepared statement 이중 방어).

### 2.3 CWE-209 Information Exposure Through Error Messages — A 범주

**공격 시나리오**: 서버가 structured detail 로 내부 상태(스택, 쿼리, 파일경로, 토큰) 를 의도치 않게 노출.

**대응**:

- detail dict 는 `{code, message, hint}` 세 필드로 고정. message 는 `body.doc_type_code` 라는 사용자-입력 값만 내포 (서버 내부 식별자 / 경로 / 토큰 / 타임스탬프 등 포함하지 않음). hint 는 정적 링크.
- 감사 로그(`actor_type` 포함) 는 기존 경로 유지, detail 과 혼합되지 않음.
- FastAPI `HTTPException` 의 dict detail 은 JSON 으로 안전 인코딩. 한국어 리터럴은 UTF-8 그대로.

**결론**: Information Exposure **낮음**.

### 2.4 CWE-287 Improper Authentication — 영향 없음

- P7-2 는 기존 인증 미들웨어 / `auth_author`, `auth_viewer`, `auth_reader` Depends 체인을 변경하지 않음.
- `_normalize_doc_type_path` 는 인증 전 단계(라우트 함수 본문) 에 있으나, FastAPI 의 dependency 시스템 상 `Depends(require_author_role)` 등 인증 체인이 함수 실행보다 먼저 끝난 상태이므로 인증 우회 경로 없음. 테스트는 각 `auth_*` fixture 로 정상/권한 부족 케이스를 이전에 검증.

**결론**: 인증 회귀 **없음**.

### 2.5 CWE-285 Improper Authorization — 영향 없음

- S2 ⑥ scope ACL 은 `scope_profile_id` 쿼리 파라미터 → 리포지토리 계층에서 필터링. P7-2 는 이 경로를 건드리지 않음. 경로 정규화는 ACL 이전 단계에서 키 형식만 정리.
- `TestDocTypePathNormalization` 의 monkeypatch 가 리포지토리의 `get_latest_by_doc_type` 등을 직접 더블링해, ACL 필터링이 호출된다는 시그니처를 보존.

**결론**: Authorization 회귀 **없음**.

### 2.6 CWE-352 CSRF — 영향 없음

- 본 변경은 새 HTTP 메서드를 추가하지 않음. 기존 GET/PUT/DELETE 에 전처리만 삽입.
- 인증 세션 쿠키 / CSRF 토큰 정책 불변.

**결론**: CSRF 회귀 **없음**.

### 2.7 CWE-367 TOCTOU — A/C 복합 범주

**공격 시나리오**: Alembic 충돌 감지(SELECT) 와 실제 UPDATE 사이에 다른 세션이 레코드를 추가해, 충돌 없음으로 판정됐던 그룹이 UPDATE 직전에 충돌 상태가 되는 경우.

**대응**:

- 실제 DB 는 `UPDATE … SET type_code = UPPER(…)` 가 PK 중복을 유발하면 `UniqueViolation` 으로 실패 → Alembic 트랜잭션 전체 롤백 → FK 는 `finally` 에서 `ADD CONSTRAINT` 로 복원. 즉 검증과 UPDATE 사이 창이 열려도 "조용히 데이터 손상" 시나리오는 발생하지 않음.
- 운영 가이드에 off-peak 시간 실행 + 동시에 `/admin/document-types` 쓰기 경로가 뜸한 상태에서 실행 권장 명시.

**결론**: TOCTOU **낮음** (DB 제약이 최종 방어선).

### 2.8 CWE-640 Weak Password Recovery / CWE-256 / CWE-312 — 해당 없음

P7-2 는 비밀번호 / 토큰 / 크리덴셜 저장·전송 경로를 수정하지 않음.

### 2.9 DoS / Rate-limit — 낮음

- `_normalize_doc_type_path` 는 O(n) regex. 입력 길이는 URL 경로 세그먼트로 경로 파라미터 상한 (FastAPI 기본 65535B 이내) 에 제한됨.
- Alembic 마이그레이션은 DDL 락이 걸리는 짧은 순간이 있으나 영구 트랜잭션 소비 아님.

### 2.10 공급망 / 의존성 — 변동 없음

- package.json / requirements.txt 변경 없음. 새 라이브러리 설치 없음.
- psycopg2 / fastapi / pydantic / alembic / react / typescript 모두 기존 버전 유지, known CVE 영향권 내에 있는 버전 아님.

## 3. 정적 분석 / 도구

- **Python**: `ast.parse` + `py_compile` 모두 OK (§검수보고서 §3.2).
- **TypeScript**: `tsc -p tsconfig.test.json` — 0 error.
- **Node test runner**: 82/82 pass.
- **bandit / npm audit**: 본 변경은 의존성 / 민감 API 호출을 신설하지 않으므로 추가 실행 불요. 기존 CI 파이프라인의 정기 스캔으로 커버.

## 4. Fuzz / 경계값 케이스

- 빈 문자열 / 공백만 / 제어문자 / 숫자 프리픽스 / 한글 / emoji / null byte → 전부 422 경로 회귀 (pytest 2건 + ExtractionSchemasP7 regex 테스트로 기존 커버리지).
- `hint.href` 에 `javascript:`, `data:`, 외부 도메인, 다른 내부 경로, trailing slash variant → node:test 4건.
- 서버 detail 에 `code` / `message` / `hint` 중 일부만 누락 → node:test 3건 (`ApiError structured payload` describe).

## 5. 로깅 / 감사 추적

- FK 실패 로그는 `logger.info("extraction_schema create blocked: doc_type_code=%r not in document_types", body.doc_type_code)` 한 줄 — 식별자 수준 정보만 기록, 개인정보 없음.
- `actor_type` audit 필드는 기존 audit_emitter 경로에서 일관 유지 (S2 ⑥ 준수).
- Alembic 충돌 RuntimeError 는 메시지에 원본 코드들(대/소문자 조합) 을 포함해 운영 개입을 유도. 민감 정보 아님.

## 6. 결론

| 범주 | 위험도 | 비고 |
|------|--------|------|
| Open Redirect / XSS via hint.href | 낮음 | 화이트리스트 대조 + node:test 회귀 |
| Injection / Path traversal | 낮음 | regex + prepared statement 이중 방어 |
| Information Exposure | 낮음 | detail 필드 고정, 내부 상태 유출 없음 |
| 인증 / 인가 회귀 | 없음 | Depends 체인 불변 |
| CSRF | 없음 | 새 메서드 없음 |
| TOCTOU (Alembic) | 낮음 | DB 제약이 최종 방어선 |
| DoS | 낮음 | 입력 길이 상한 / DDL 짧음 |
| 공급망 | 변동 없음 | 의존성 추가 없음 |

**P7-2 범위에서 신규 혹은 악화된 취약점 없음**. 화이트리스트 기반 hint href 소독과 regex 재검증을 통해 A/B 경계의 위협을 모두 낮은 수준으로 유지한다. 운영 투입 전 단 하나의 선결 조건은 `alembic upgrade head` 실행 결과에 충돌이 없음을 확인하는 것 (충돌 시 안전 중단, 운영 수동 병합 후 재실행).
