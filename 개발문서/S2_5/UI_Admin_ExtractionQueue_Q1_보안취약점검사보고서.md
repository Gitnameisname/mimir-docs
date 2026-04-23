# /admin/extraction-queue Q1 보안취약점 검사보고서

- 작성일: 2026-04-22
- 대상 변경분:
  - `frontend/src/features/admin/extraction-queue/AdminExtractionQueuePage.tsx`
  - `frontend/src/features/admin/extraction-queue/constants.ts`
  - `frontend/src/features/admin/extraction-queue/helpers.ts`
  - `frontend/src/lib/api/s2admin.ts` (extractionQueueApi.list)
- 관련 검수보고서: `UI_Admin_ExtractionQueue_Q1_검수보고서.md`
- 체크리스트: OWASP ASVS 5.0 C1~C8 / 주요 CWE

---

## 1. 위협 모델

### 1.1 자산

- 추출 결과(문서 → 필드 JSON)의 검토 상태와 승인/반려 기록
- 조회/승인/반려에 사용되는 `scope_profile_id` 와 `document_type_code`
- 관리자 세션 (fetch 호출자)

### 1.2 신뢰 경계

- **신뢰**: 관리자 세션 상태, 서버 정책, 페이지 빌드 산출물
- **비신뢰**:
  - URL 파라미터 / 입력 필드 (scope_profile_id, 상태 필터, 타입 필터)
  - 서버 응답 본문 (document_type_code, document_title, original_content_preview, extracted_fields, error.message, error.hint)
  - react-query 캐시(이전 응답 잔재)

### 1.3 주요 위협

| ID | 위협 | 대응 |
| --- | --- | --- |
| T1 | `document_title` / `original_content_preview` 에 HTML/`<script>` 주입 | React JSX 기본 이스케이프 (dangerouslySetInnerHTML 미사용) |
| T2 | `extracted_fields` 값이 순환 참조/거대 객체 | `formatFieldValue` 는 JSON.stringify 단발 호출 — 순환은 throw 로 격리 가능 |
| T3 | `scope_profile_id` 에 악성 입력 (CRLF / `?&` / URL 이스케이프 bypass) | `buildQueryString` 의 `URLSearchParams` 이 자동 인코딩 |
| T4 | 404 Mock 폴백이 실제 장애를 감춤 | 404 만 폴백, 5xx/403/NetworkError 는 명시 에러 |
| T5 | scope_profile_id 검증 우회로 무권한 조회 | 프론트는 UX 가드만; 최종 권한 검증은 서버 몫 (S2 ⑥) |
| T6 | 상세 패널 focus 가 DOM 밖으로 새어 배경 위젯과 상호작용 | useFocusTrap + Escape |

---

## 2. CWE 대응 체크리스트

### 2.1 CWE-79 (XSS)

- **위험**: `document_title`, `original_content_preview`, `extracted_fields` 값이 서버에서 오염되어도 화면에 주입되면 안 됨.
- **검사**:
  - 모든 표시는 JSX 중괄호 `{...}` 로 삽입 — React 가 HTML 이스케이프.
  - `dangerouslySetInnerHTML` 사용 없음 (`grep` 확인).
  - `<a href>` 링크 렌더는 이 페이지에 없음(상세 패널은 plain text 만 렌더).
- **결과**: 이전 대비 신규 노출 없음. 안전.

### 2.2 CWE-918 / CWE-601 (SSRF / Open Redirect)

- **위험**: 서버 응답에 포함된 URL 에 프론트가 자동 navigate 하면 오픈 리다이렉트.
- **검사**: 이 페이지는 어떤 서버 제공 href 도 사용하지 않는다. `mutationErrorMessage` 가 반환하는 문자열은 모두 정적.
- **결과**: 해당 없음.

### 2.3 CWE-89 (SQL Injection)

- **위험**: 필터 문자열이 SQL 로 연결되면 위험.
- **검사**:
  - 프론트는 `buildQueryString` → `URLSearchParams` 로 안전 인코딩.
  - `isValidExtractionStatus` 타입 가드로 3가지 허용 값만 통과 (호출부에서 직접 사용하진 않지만 회귀 테스트로 존재 보증).
  - `status`, `document_type`, `scope_profile_id` 는 서버에서 prepared statement / ORM 로 바인딩 (S2-5 ActorContext 기준).
- **결과**: 프론트 경계에서 인코딩 보장.

### 2.4 CWE-639 / CWE-284 (Broken ACL)

- **위험**: `scope_profile_id` 를 임의로 지정해 타 스코프 추출 결과 조회.
- **검사**:
  - Q1-A4/A5 로 scope_profile_id 가 쿼리에 전달되지만, 프론트 입력은 **제안** 에 불과.
  - 최종 인가는 서버의 ActorContext + Scope Profile 검증에 의존 (S2 ⑥).
  - 프론트가 비워 보내면 서버는 사용자 기본 스코프로 해석.
  - UUID 길이 32자 이하인 값은 `enabled: false` 로 쿼리 자체를 차단 → 혼동된 요청 방지.
- **결과**: 프론트는 올바른 ACL 파라미터를 전달. 서버 구현(향후) 에서 인가 필수.

### 2.5 CWE-400 / CWE-730 (DoS)

- **위험**: `extracted_fields` 가 수십 MB 일 때 JSON.stringify 로 메인 스레드 블로킹.
- **완화**:
  - 상세 필드 테이블은 `Object.entries` 로 전체 순회 — 서버가 payload 를 제한하는 것이 정책.
  - `max-h-32 overflow-y-auto` 로 원본 미리보기 높이 제한.
  - 페이지네이션 `page_size: 20` 으로 리스트 한도 고정.
- **잔여 위험**: 서버가 필드 수/크기 상한을 강제해야 안전 (추출 스키마 P3-C 와 동일 정책 적용 예정).

### 2.6 CWE-352 (CSRF)

- **위험**: approve / reject 는 상태 변경 요청.
- **검사**: 기존 `api` 클라이언트가 credentials 및 CSRF 토큰 처리 담당 (Q1 에서 변경 없음).
- **결과**: 회귀 없음.

### 2.7 CWE-1021 (UI Redressing / Focus 탈출)

- **위험**: 모달 바깥 요소가 클릭되거나 배경 키보드가 포커스 받음 → 인증 정보 유출.
- **검사**:
  - `useFocusTrap(panelRef, true)` 로 Tab / Shift+Tab 경계에서 wrap.
  - `role="dialog" aria-modal="true"` 로 SR 가 다이얼로그 컨텍스트 인식.
  - Escape 로 닫힘 (`stopPropagation` 으로 배경 Esc 핸들러 간섭 차단).
  - 배경 dim (`bg-black/40`) 으로 시각적 분리.
- **결과**: WCAG 2.4.3 / 2.1.2 만족.

### 2.8 CWE-20 (Input Validation)

- **위험**: 필터 값 / scope 값이 임의 문자열.
- **검사**:
  - `isValidExtractionStatus` 로 "정의된 3가지" 만 허용 (호출부 사용은 향후 URL 파라미터 복원에 대비).
  - `statusFilter` 빈 문자열은 "전체" 를 의미하며 서버로 전송하지 않음 (buildQueryString 이 drop).
  - `typeFilter` 는 서버에서 받은 document_types 에서 선택된 값 중 하나 → 서버가 원래 내려준 값만 가능.
  - `scopeProfileId` 는 `.trim().length ≥ 32` 가드 + `aria-invalid` 표시.
- **결과**: 클라이언트 가드 적절. 서버 검증은 Phase 8 에서 필수.

### 2.9 CWE-200 (정보 노출)

- **위험**: 에러 메시지에 스택/내부 경로 노출.
- **검사**:
  - `mutationErrorMessage` 는 상태 코드별 템플릿 메시지 우선, fallback 은 `error.message` (서버 detail).
  - 서버 detail 은 P7-1/P7-2 에서 구조화 포맷(code/message/hint) 을 따르므로 스택 미포함.
  - `NoticeBanner variant="error"` 는 `role="alert"` — SR 에 즉시 공지.
- **결과**: 추가 노출 없음.

### 2.10 CWE-1333 (ReDoS)

- **위험**: regex 기반 입력 검증이 catastrophic backtracking 에 취약.
- **검사**:
  - `isValidDocTypeCode` (공유 유틸) 의 `^[A-Z][A-Z0-9_-]*$` — backtracking 없음.
  - Q1 신규 정규식 추가 없음.
- **결과**: 해당 없음.

---

## 3. 의존성 변화

- 신규 외부 라이브러리 추가 **없음**.
- `@tanstack/react-query`, `react`, `@/hooks/useFocusTrap`, `@/lib/api/admin`, `@/lib/api/s2admin`, `@/lib/api/client`, `@/features/admin/extraction-schemas/docTypeNormalize` — 모두 기존 스택.
- S2 ⑦ (폐쇄망) 영향 없음.

---

## 4. 로깅/감사 흐름 점검

- Q1 은 **조회 경로만** 변경. 쓰기(approve/reject) 는 기존 API 를 그대로 호출.
- 기존 서버 측 `audit_emitter.emit(actor_type=..., action=...)` 체인 변경 없음 (S2 ⑥).
- scope_profile_id 가 전달되어 서버 audit 에 기록됨 (서버 측 보강 필요 — Phase 8 범위).

---

## 5. 결론

`/admin/extraction-queue` Q1 변경분에서 신규 또는 악화된 보안 취약점은 **발견되지 않았다**.

개선된 보안 속성:

- ACL 파라미터(`scope_profile_id`) 가 조회 경로에 전달됨 → S2 ⑥ 강화.
- 에러/목 데이터 구분이 명확해져 **장애 은폐** 리스크 감소.
- 모달 focus trap 으로 UI Redressing / 포커스 탈출 차단.

잔여 위험은 모두 "서버 라우트가 아직 구현되지 않음" 을 전제로 하며, Phase 8 FG8.2/8.3
백엔드 착수 시 `/api/v1/admin/extraction-results` 경로에 대한 ACL / 입력 검증 / rate
limit / payload 크기 제한을 **필수** 로 설계해야 한다 (§2.5 잔여 위험 참조).
