# `/admin/extraction-schemas` — P7-1 보안 취약점 검사 보고서

작성일: 2026-04-22
대상: P7-1 변경 (백엔드 FK violation → 422 변환, `doc_type_code` 대문자 정규화, 프론트 에러 배너 FK 분기)
선행 보고서: P4~P6 보안 보고서 (누적 threat model 은 P6 보고서 C1~C9 참조)

## 0. 방법론

OWASP ASVS 4.0.3 관련 컨트롤을 기준으로, P7 변경 파일을 대상으로 8 개 카테고리 threat model 수행.

| 카테고리 | 검사 영역 |
|---------|----------|
| C1 | 인증/인가 우회 |
| C2 | 입력 검증 / 정규화 |
| C3 | 정보 노출 (서버 에러 메시지) |
| C4 | 로깅/감사 |
| C5 | DoS / 리소스 소모 |
| C6 | XSS / HTML 삽입 |
| C7 | Open Redirect / SSRF |
| C8 | CSRF / 상태 변경 취약점 |

## C1. 인증/인가 우회

### 변경사항
- `create_extraction_schema` 는 기존과 동일하게 `resolve_current_actor` 디펜던시를 거침.
- 새로 추가된 except 분기는 422 만 내리고 본 로직을 우회하지 않음.

### 검사 결과
- ✅ **통과**. 인증/인가 플로우가 변경 전후 동일.
- FK 위반은 repo 내부 INSERT 시점에 발생 → 이미 인증을 통과한 후의 분기.
- 잠재 우려: 클라이언트가 존재/미존재 `type_code` 를 probing 해 document_types 의 존재 여부를 추측할 수 있음. 그러나 `/admin/document-types` GET 은 동일한 권한 (관리자) 으로 직접 조회 가능 → **신규 정보 유출 없음** (기존에 이미 열람 가능).

## C2. 입력 검증 / 정규화

### 변경사항
- 서버 `CreateExtractionSchemaRequest._validate_doc_type_code`:
  - `v.strip().upper()` → regex `^[a-zA-Z][a-zA-Z0-9_-]*$`.
  - regex 는 대문자 전제가 아니어서 `.upper()` 이후에도 동일 패턴으로 동작.
- 서버 `CreateDocumentTypeBody._validate_type_code`:
  - `v.strip().upper()` → regex `^[A-Z][A-Z0-9_-]*$` + 길이 100 제한.
  - 정규화 **이후** 에 regex 검증 — 우회 경로 없음.
- 프론트 `normalizeDocTypeCode` → `isValidDocTypeCode` 순서도 동일.

### 공격 시나리오 테스트
- `"<script>alert(1)</script>"` 입력 → `.strip().upper()` → `"<SCRIPT>ALERT(1)</SCRIPT>"` → regex 위반 → 422.
- `"A\x00B"` 입력 → regex 위반 (NUL 제외) → 422.
- `"AAA' OR 1=1 --"` 입력 → `"AAA' OR 1=1 --"` → regex 위반 (공백·특수문자) → 422.
- `" " * 1000` (공백 폭탄) → `.strip()` 후 빈 문자열 → `min_length=1` Pydantic 에서 이미 차단.
- 유니코드 homoglyph (`Α` (U+0391 Greek Capital Alpha) 등) → regex `[A-Z]` 는 ASCII 만 매치 → 거절.

### 검사 결과
- ✅ **통과**. 정규화가 regex 이전에 수행되어 우회 불가.
- `VARCHAR(100)` PK 로 길이 제한은 DB 도 enforce. 프론트 100자 제한은 UX 목적.

## C3. 정보 노출

### 변경사항
- 422 detail: `DocumentType '{body.doc_type_code}' 이(가) 존재하지 않습니다. 먼저 /admin/document-types 에서 동일한 코드를 생성한 뒤 다시 시도하세요.`
- `logger.info` 에 `doc_type_code=%r` 기록 (level: info).

### 검사 관점
- 사용자가 제공한 `doc_type_code` 를 그대로 에러 메시지/로그에 담음. 이 필드는 이미 Pydantic regex 로 필터링되어 있어 **ASCII 영문/숫자/`-`/`_`** 만 포함. 로그 injection / 터미널 이스케이프 / HTML injection 불가.
- psycopg2 원본 메시지 (`constraint "extraction_schemas_doc_type_code_fkey"`) 는 로그에도 노출하지 않고 클라이언트에게도 돌려주지 않음 — DB 스키마 정보 leak 없음.
- `except Exception:` 의 `logger.exception` 경로는 변경 전후 동일 (스택만 서버 로그로, 클라이언트에게는 generic `"내부 서버 오류가 발생했습니다."` 반환).

### 검사 결과
- ✅ **통과**. 외부로 내려가는 메시지는 정규화된 사용자 입력과 안내 문구만. DB 내부 정보 비노출.

## C4. 로깅/감사

### 변경사항
- `audit_emitter.emit_for_actor(event_type="extraction_schema.created", ...)` 는 성공 시에만 호출 — FK 로 실패한 시도는 audit 이벤트가 발생하지 않음.
- `logger.info("extraction_schema create blocked: doc_type_code=%r not in document_types", body.doc_type_code)` 만 남김.

### 관점
- S2 원칙 ⑤ 는 "actor_type 필드를 감사 로그에 기록" 을 요구. 성공 사례의 audit 은 그대로 유지됨.
- FK 실패는 비즈니스 이벤트가 아니라 입력 검증 실패에 가까움 → audit 대상 아님 (기존 ValueError→422 경로와 동일 대우).
- 반복적인 FK 실패가 공격 probing 의 징후일 수 있으므로 INFO 로그로 남김. 운영팀의 SIEM 이 패턴 탐지 가능.

### 검사 결과
- ✅ **통과**. S2 ⑤ 준수 + probing 흔적 로깅.

## C5. DoS / 리소스 소모

### 변경사항
- 새 경로는 FK violation 을 Python exception 으로 받고 HTTPException 으로 변환 — O(1) 연산.
- 정규화 `.strip().upper()` — O(n), n ≤ 100 (VARCHAR 제한).
- 새 regex `^[A-Z][A-Z0-9_-]*$` 는 백트래킹 없음 (ReDoS 불가능, linear).
- `isFkMissingDocTypeError` 의 regex `/DocumentType\s+'[^']+'\s*이\(가\)\s*존재하지\s*않/` 도 백트래킹 없는 단순 순차 매칭.

### 검사 결과
- ✅ **통과**. ReDoS / pathological regex 없음.

## C6. XSS / HTML 삽입

### 변경사항
- 프론트 `ErrorBanner` 가 서버 `error.message` 를 `{body}` 로 렌더 — React JSX 의 자동 escape 에 의존.
- 새 액션 `<a href={hint.href}>` 의 `hint.href` 는 **하드코딩된 `"/admin/document-types"`** — 사용자 입력 영향 없음.
- `hint.label` 도 하드코딩 `"문서 유형 관리 열기"`.

### 검사 관점
- 만약 `hint.href` 가 서버 응답 기반이었다면 open redirect / javascript: URI 주입 위험. 그러나 현재 구현은 순수 상수.
- React 의 `{body}` 는 string interpolation 이며 `dangerouslySetInnerHTML` 아님 → HTML injection 불가.

### 검사 결과
- ✅ **통과**. XSS 벡터 없음.

## C7. Open Redirect / SSRF

### 변경사항
- `<a href="/admin/document-types">` — 같은 origin 내 상대경로, 외부 URL 로 리다이렉트 불가.
- 서버는 URL 을 만들지 않음. 프론트만 생성.

### 검사 결과
- ✅ **통과**.

## C8. CSRF / 상태 변경

### 변경사항
- 변경은 기존 `POST /extraction-schemas` 의 에러 처리만 수정. 새 엔드포인트 없음.
- 프론트 액션 `<a>` 는 GET 이동 (Next 페이지 전환) — 상태 변경 액션 아님.

### 검사 결과
- ✅ **통과**.

## 종합 리스크 매트릭스

| # | 항목 | 심각도 | 상태 |
|---|------|--------|------|
| R1 | 422 메시지에 입력 그대로 echo → XSS/로그 injection | Low | 정규화 regex 로 위험 입력 사전 차단 → 실질적 위험 0 |
| R2 | FK violation probing 으로 document_types 목록 추측 | Info | 동일 권한으로 직접 조회 가능 → 신규 위험 없음 |
| R3 | `isFkMissingDocTypeError` 메시지 결합으로 서버 메시지 변경 시 분류 실패 | Low | 주석으로 규약 명시 + node:test 로 회귀 방지. 실패해도 UX 만 손상, 보안 영향 없음 |
| R4 | 대소문자 정규화 이전 raw SQL 로 삽입된 소문자 레코드 | Medium | 운영 데이터 점검 쿼리 포함 (검수보고서 §9) |

## 종합 판정

**배포 가능 ✅**

## 운영 체크리스트

배포 전:
- [ ] DB 확인: `SELECT type_code FROM document_types WHERE type_code <> UPPER(type_code);`
  - 결과 > 0 이면 운영팀 개입 필요 (정규화 이전 데이터 정리).
- [ ] 프론트 캐시 무효화 (브라우저 하드 리프레시 안내 공지).

배포 후 모니터링:
- [ ] `extraction_schema create blocked: doc_type_code=...` INFO 로그의 발생 빈도 — 정상 사용자의 단순 오타인지, probing 인지 구분.
- [ ] 신규 422 비율 변화 (기존 500 → 422 전환 효과 확인).
