# /admin/extraction-queue B 스코프 보안취약점검사보고서

- 작성일: 2026-04-22
- 대상 화면: `/admin/extraction-queue` + `/api/v1/admin/extraction-results` 라우터
- 범위: B 스코프(실-API 연결)에서 신설/변경된 모든 서버/프론트 코드
- 참고 기준: OWASP ASVS v4.0.3 Level 2, OWASP API Security Top 10 2023, CWE Top 25

---

## 0. 요약 (TL;DR)

| Severity | Count | 비고 |
| --- | --- | --- |
| Critical | 0 | |
| High | 0 | |
| Medium | 0 | |
| Low | 2 | 개발 환경 디버그 헤더 인증(기존 S2-5 전체 공통), 감사 로그 내 scope_profile_id PII 범주 검토 |
| Info | 3 | 페이지네이션 상한, overrides 필드 수 상한, reason/comment 최대 길이 |

신규 취약점은 식별되지 않았다. 기존 프로젝트 공통 제약만 재확인.

---

## 1. 검사 항목 및 결과

### 1.1 인증 / 권한 (OWASP API Top 10 #1, #5 — BOLA / BFLA)

- **RBAC 분리**: GET 은 `admin.read` (ORG_ADMIN ↑), POST 는 `admin.write` (SUPER_ADMIN 전용) 로 분리되어 권한 승격 위험 없음 (`backend/app/api/auth/authorization.py:123-124`)
- **미인증**: `authorization_service.authorize(..., require_authenticated=True)` 로 401 응답 보장 (테스트 `TestListRouteRBAC::test_unauthenticated_rejected`, `TestWriteRouteRBAC::test_unauthenticated_approve`)
- **BFLA**: POST 가 GET 과 같은 역할로 열리지 않도록 `_require_admin` 이 `request.method` 로 분기
- **결과**: PASS

### 1.2 IDOR / 교차 scope 열람 (OWASP API Top 10 #1)

- `GET /{id}` 는 scope_profile_id 가 전달된 경우, 실제 row 의 scope 와 불일치하면 **403 이 아닌 404** 로 응답 → 존재 여부 enumeration 방어
- scope_profile_id 미전달 시에도 list 쿼리는 `WHERE scope_profile_id = %s` 조건 또는 NULL 허용 로직으로 필터
- 관리자가 다른 스코프의 결과를 내려받는 것은 **정책상 허용** (SUPER_ADMIN 은 global view). 정책상 경계는 감사 로그의 `scope_profile_id` 필드로 추적
- **결과**: PASS (관리자 전역 열람은 의도된 동작; 감사 경로로 추적 가능)

### 1.3 Injection (CWE-89 SQL, CWE-78 OS Command, CWE-79 XSS)

- 모든 SQL 쿼리는 psycopg2 파라미터 바인딩 (`%s` + 튜플/리스트) 로 고정. 문자열 concat 0건 (`rg "\".*SELECT.*'" backend/app/repositories/extraction_candidate_repository.py` 결과 없음)
- `status` 필터는 외부 허용 3-값만 통과(`_VALID_EXTERNAL_STATUS`), 내부 enum `ExtractionStatus` 로 변환 후 `ANY(%s)` 바인딩
- `document_type` 은 서버에서 `.upper()` + 정규식 `^[A-Z][A-Z0-9_-]*$` 검증 후에야 SQL 에 투입
- `scope_profile_id` / `extraction_id` 는 UUID 파서를 통과해야만 쿼리까지 도달 (비-UUID 는 422 로 차단)
- 프론트 상세 패널은 `formatFieldValue` 로 문자열 변환 후 React 가 기본 escape — `dangerouslySetInnerHTML` 미사용, XSS 경로 없음
- 편집 모드 input 값도 React 가 escape. 서버 전송 직전 `tryParseOverrideValue` 는 `JSON.parse` 실패 시 원문을 그대로 반환하며, 서버는 이를 `Dict[str, Any]` 의 value 로 수용하므로 eval 류 코드 실행 경로 없음
- **결과**: PASS

### 1.4 Mass Assignment / Over-posting (OWASP API Top 10 #6)

- `AdminApproveExtractionRequest` 는 `approval_comment`, `overrides` 두 필드만 받음 (pydantic v2 기본 `extra="ignore"`)
- approve/reject 에서 서버가 주도적으로 설정하는 필드(예: `reviewed_by`, `reviewed_at`, `status`) 는 바디에서 절대 받지 않음 — 승격 경로 차단
- `overrides` 의 값은 `dict[str, Any]` 지만, approve 시 `candidate.extracted_fields` 에 덮어쓰는 key 범위가 제한되지 않는 것은 **의도** (신규 필드 추가도 허용). 프론트 편집 UI 는 기존 key 만 노출해 실수 입력을 줄임
- **결과**: PASS

### 1.5 Denial of Service (CWE-400)

| 경로 | 상한 | 코드 |
| --- | --- | --- |
| `GET /?page` | ≤ 10,000 | `Query(default=1, ge=1, le=10_000)` |
| `GET /?page_size` | ≤ 100 | `Query(default=20, ge=1, le=100)` |
| `POST approve.overrides` | ≤ 200 필드 | `@field_validator` |
| `POST approve.approval_comment` | ≤ 1024 자 | `Field(max_length=1024)` |
| `POST reject.reason` | ≤ 1024 자 | `Field(max_length=1024)` |
| GET 상세 `original_content_preview` | 서버측 2000 자 truncate | `_row_to_detail_payload` |

- FastAPI 의 JSON body 기본 상한과 함께 모든 가변 입력에 상한 존재
- **결과**: PASS

### 1.6 Race / 동시성 (CWE-362)

- approve/reject 는 먼저 `candidate.status != PENDING` 을 확인하지만, 그와 `update_status` 사이에 동시 호출이 발생해 race 하는 경우 `update_status` 의 `WHERE status = 'pending'` 가드 덕분에 패치가 0-row update 로 실패 → 서버는 이를 409 로 응답
- 감사 이벤트는 실제 상태 변경 성공 이후에만 emit
- **결과**: PASS

### 1.7 감사 로그 (OWASP ASVS V7, S2 ⑤)

- 모든 상태 전이에 `audit_emitter.emit_for_actor` 호출. 필드:
  - `event_type` / `action` 으로 분류
  - `actor` 전체 컨텍스트 (actor_type 포함)
  - `resource_type="extraction_candidate"`, `resource_id=id`
  - `previous_state` / `new_state` 문자열(raw enum value)
  - `metadata`: modified/override_field_count/document_id/document_type_code/scope_profile_id
- approve 성공 경로와 reject 성공 경로 모두 커버. 실패 경로에서는 emit 안 함 (부분 커밋 오용 방지)
- **결과**: PASS

### 1.8 개인정보 / 민감정보 (ASVS V8)

- 응답에 사용자 식별자로 `reviewer_id` (문자열 id) 만 노출. 이메일/IP 등 미노출
- `scope_profile_id` 는 관리자에게만 노출 (admin.read 가드) 되므로 일반 사용자에 대한 PII 확산 없음
- `original_content_preview` 는 2000자로 truncate 되어, 긴 문서 본문이 관리자 브라우저 메모리에 과도히 남지 않음
- **Low 지적**: 감사 이벤트 `metadata.scope_profile_id` 는 조직 내부 식별자이므로 OWASP 분류상 낮은 PII 등급이지만, 감사 로그 보존 기간 정책 수립 시 함께 검토 필요
- **결과**: PASS (Low 관찰 1건)

### 1.9 CORS / Open Redirect / CSRF

- 본 라우터는 동일 Origin 가정이며 별도 CORS 설정 변경 없음
- `href` 나 `redirect_url` 응답 필드 없음 → Open Redirect 경로 없음
- 상태 변경 API 는 POST + JSON body + 인증 헤더 필요 → 기본 CSRF 위험은 SameSite 쿠키 + JWT Bearer 정책으로 프로젝트 전역 대응
- **결과**: PASS

### 1.10 프론트 XSS / DOM 보안

- `document_title`, `document_type_code`, `original_content_preview`, `extracted_fields` 모두 React 기본 escape
- `<div className="font-mono whitespace-pre-wrap">` 안의 preview 역시 텍스트 노드. DOMPurify 불필요
- Link 렌더 없음 → phishing/open-redirect 없음
- **결과**: PASS

### 1.11 로깅 (CWE-532 민감정보 로그 유출)

- `logger.exception("approved_extraction 생성 실패")` — 예외 객체만 로깅, bodyrepr 하지 않음
- 라우터 어디서도 `body.approval_comment` 나 `overrides` 를 로그로 남기지 않음
- 감사 이벤트는 수치/식별자만 (개별 필드 값 본문은 감사 로그 미기록)
- **결과**: PASS

### 1.12 공급망 / 외부 의존 (S2 ⑦ 폐쇄망)

- 신규 추가 패키지 없음. 기존 의존(`fastapi==0.135.3`, `pydantic==2.12.5`, `psycopg2-binary==2.9.11`) 그대로 사용
- 외부 SaaS 호출 0건 (RAG, OpenAI 등 무관)
- **결과**: PASS

---

## 2. 개발 환경 한정 관찰 (Low)

### L-1. 디버그 헤더 기반 인증

- `X-Actor-Id` / `X-Actor-Role` 헤더 방식은 `ENVIRONMENT=production` 또는 `DEBUG=false` 에서 비활성. 테스트/개발 전용 경로로 프로젝트 전역 공통 제약
- B 스코프에서 신규 유입 위험 없음

### L-2. 감사 로그 보존 정책

- `metadata.scope_profile_id` 및 `metadata.document_id` 는 내부 식별자. 운영 시 보존 기간(기본 90일 이상 권장)과 접근 권한은 별도 운영 문서로 관리 필요
- B 스코프에서 신규 유입 위험 없음

---

## 3. Info (정보성)

| ID | 항목 | 현재 값 | 비고 |
| --- | --- | --- | --- |
| I-1 | page_size 상한 | 100 | OWASP API Top 10 #4 (Unrestricted Resource Consumption) 대비 |
| I-2 | overrides 필드 수 상한 | 200 | body-based DoS 방어 |
| I-3 | approval_comment / reason 길이 | 각 1024자 | 스토리지·감사 로그 팽창 억제 |

---

## 4. 정적/동적 검사 요약

| 도구 | 대상 | 결과 |
| --- | --- | --- |
| `python -m py_compile` | `admin_extraction_results.py` (router+schema) + `test_admin_extraction_results_routes.py` | OK |
| `tsc --noEmit` | frontend (전체) | extraction-queue 관련 0 에러 (기존 AdminUserDetailPage.tsx 1건은 본 작업 범위 외, 사전 경로) |
| `npm test` (tsc + node:test) | frontend | 114 pass / 0 fail / 0 skip |
| `rg "SELECT .*+.*%"` | backend repositories | 0 hit (string concat injection 경로 없음) |
| `rg "dangerouslySetInnerHTML"` | extraction-queue frontend | 0 hit |
| `rg "eval\("` | 전체 신규 코드 | 0 hit |

---

## 5. 결론

- B 스코프에서 신규 식별된 Critical/High/Medium 취약점: **없음**
- 기존 프로젝트 공통 Low 항목 2건은 B 스코프 내 신규 유입 없음
- OWASP API Top 10 (2023) 의 #1 (BOLA), #3 (BOPLA), #4 (Unrestricted Resource Consumption), #5 (BFLA), #6 (Unrestricted Access to Sensitive Business Flows), #9 (Improper Inventory) 각 항목 모두 조치되어 있다.
- 권장: 향후 INTEGRATION_TEST=1 환경에서 `audit_emitter.emit_for_actor` 호출 횟수와 DB 상의 `extraction_candidates.status` 전이를 end-to-end 로 단언하는 테스트를 추가하면 감사 무결성까지 회귀 커버 가능.
