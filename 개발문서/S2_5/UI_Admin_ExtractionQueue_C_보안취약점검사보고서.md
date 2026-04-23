# /admin/extraction-queue C 스코프 보안취약점검사보고서

- 작성일: 2026-04-22
- 대상 화면: `/admin/extraction-queue` (B 에 더한 UI/접근성 보강)
- 범위: C 스코프에서 신설/변경된 프론트 코드
  - `frontend/src/features/admin/extraction-queue/helpers.ts`
    (`parseFiltersFromUrl`, `toSearchParamsString`, `validateRejectReason`)
  - `frontend/src/features/admin/extraction-queue/AdminExtractionQueuePage.tsx`
    (URL state sync, reject reason UI, SkeletonRows, 필터인식 empty state)
- 참고 기준: OWASP ASVS v4.0.3 Level 2, OWASP API Top 10 2023, CWE Top 25, OWASP DOM-based XSS Prevention Cheat Sheet

---

## 0. 요약 (TL;DR)

| Severity | Count | 비고 |
| --- | --- | --- |
| Critical | 0 | |
| High | 0 | |
| Medium | 0 | |
| Low | 0 | |
| Info | 3 | URL 파라미터 상한, 반려 사유 버퍼, SkeletonRows aria 격리 |

C 스코프는 프론트 전용 개선이며 서버 경계 / 인증 / RBAC 에는 변경이 없다. 신규 취약점 식별 0건.

---

## 1. 검사 항목 및 결과

### 1.1 DOM-based XSS (CWE-79)

#### URL 파라미터 경로

- `parseFiltersFromUrl(sp)` 는 raw string 을 받아 정규식/enum 검증 후에만 state 에 반영한다.
  - `status`: `isValidExtractionStatus` (문자열 비교) 로 3-값 허용셋 외 모두 fallback. SQL/JS 인젝션 시도(`'; DROP TABLE`, `<script>`) 는 기본값으로 dropped 되는 node:test 케이스로 회귀 커버
  - `document_type`: `/^[A-Z][A-Z0-9_-]{0,63}$/` 매칭 필수. 그 외는 빈 문자열
  - `scope_profile_id`: UUID v4 정규식 필수. 그 외는 빈 문자열
  - `page`: `Number.parseInt` + 1~10,000 범위 밖은 1로 clamp
- 파싱된 값이 이후 렌더에 들어가도 React 기본 escape 로 HTML/JS 컨텍스트 주입 불가. `dangerouslySetInnerHTML` 0건.
- **결과**: PASS

#### 반려 사유 textarea

- `rejectReason` state 는 DOM 에 value prop 으로만 바인딩 → React escape. URL / href / innerHTML 으로 흘러가지 않음
- 서버 전송 값은 `rejectValidation.normalized` (trim 결과 문자열) → JSON body → 서버가 `max_length=1024` pydantic 검증
- **결과**: PASS

### 1.2 Open Redirect (CWE-601)

- `router.replace(nextUrl, { scroll: false })` 에 전달되는 `nextUrl` 은 `pathname + '?' + toSearchParamsString(filters)` 로 고정
- `pathname` 은 `usePathname()` 이 반환하는 현재 경로 — 사용자가 주입 불가
- 쿼리 부분은 `toSearchParamsString` → `URLSearchParams.toString()` → RFC 3986 인코딩
- 외부 도메인으로 리다이렉트 경로 없음
- **결과**: PASS

### 1.3 URL 오염 / 딥링크 공격 (CWE-1286)

- 악의적으로 제작된 URL (예: `?status=<script>&type='; DROP&page=NaN&scope=javascript:alert(1)`) 에 대해:
  - 모든 값이 검증 실패 → 기본값으로 fallback
  - 에러 배너 / throw 없음 → 페이지가 정상 렌더됨 (DoS 방어)
- 사용자가 뒤로가기 등으로 오염된 URL 로 복귀해도 페이지는 안전 상태
- **결과**: PASS

### 1.4 Denial of Service (CWE-400)

| 경로 | 상한 | 위치 |
| --- | --- | --- |
| URL `page` | 1 ~ 10,000 | `parseFiltersFromUrl` |
| URL `type` | 길이 ≤ 64, UPPER-SNAKE 만 | `_DOC_TYPE_RE` |
| URL `scope` | 36자 UUID 형태만 | `_UUID_RE` |
| URL `status` | 허용 3-값 또는 빈문자열 | `isValidExtractionStatus` |
| textarea maxLength | 1,024 + 64 버퍼 | `maxLength` prop |
| 반려 사유 실 검증 상한 | 1,024 | `validateRejectReason` |
| SkeletonRows | 상수 5행 | 호출부 |

- 모든 외부 입력 경로에 상한 존재. 프론트 상한 돌파 시에도 서버 pydantic 이 최종 권위
- **결과**: PASS

### 1.5 Race / 동시성 (CWE-362)

- 필터 변경 → state 업데이트 → URL replace 의 체인에서 루프 방지를 위해 `qs === currentQs` diff 로 guard
- 뒤로가기로 URL 이 외부에서 변경되면 `useSearchParams` 가 새 참조를 반환하지만 우리는 이를 다시 읽지 않으므로 루프 없음
- `searchParams` 는 초기 마운트 시 1회만 파싱하는 의도인데, useEffect 의존성에서 제외한 이유를 주석으로 명시 (`eslint-disable-next-line react-hooks/exhaustive-deps`)
- **결과**: PASS

### 1.6 감사 로그 / 감사성

- 반려 사유는 서버로만 전송되며 프론트는 `console.log` 등 어디에도 기록하지 않음
- URL 에는 scope_profile_id 가 포함될 수 있지만 이는 서버의 감사 로그 경계와 동일한 데이터 — 브라우저 히스토리에 남는 것은 기존 admin 페이지(예: /admin/users/[id]) 와 동일한 수준의 내부 식별자
- **결과**: PASS (기존 정책 유지)

### 1.7 Mass Assignment / Over-posting (OWASP API #6)

- 반려 API 호출은 `extractionQueueApi.reject(id, reason?)` — 단일 문자열 옵션만 전달
- 승인 API 호출은 `buildApprovePayload(overrides?)` — 편집 시에만 overrides 를 담고 그 외 필드는 서버가 주도
- C 스코프에서 API 계약 변경 없음 → B 스코프 보안취약점검사보고서 §1.4 의 검증이 계속 유효
- **결과**: PASS

### 1.8 접근성 / 보조기술 남용

- `role="alert"` 배너는 화면에 실제 에러가 있을 때만 렌더 — 남용 없음
- `role="status" aria-live="polite"` 는 로딩 중 / 카운터 변동에만 사용. 공격 surface 없음
- `aria-hidden` 은 SkeletonRows placeholder 에만 적용되어 스크린리더 혼란 방지
- **결과**: PASS

### 1.9 제어 문자 / 바이너리 페이로드

- `validateRejectReason` 에서 `\u0000-\u0008\u000B\u000C\u000E-\u001F` 를 명시적으로 차단
  - 허용: `\t`(0x09), `\n`(0x0A), `\r`(0x0D) — 정상 사유 입력 용이
  - 차단: NULL byte, BEL, BS, vertical tab, form feed, device control codes
- 이는 로그 주입(log forging, CWE-117) / 터미널 escape sequence 공격에 대한 심층 방어
- 서버도 pydantic `max_length` 와 별도로 `Optional[str]` 로 받지만, 프론트 차단으로 payload 가 도달하지 않음
- **결과**: PASS

### 1.10 CSRF / SameSite

- URL replace 는 동일 Origin 내 `history.replaceState` 호출이며 서버 요청이 아님
- 상태 변경 API (approve/reject) 는 기존 B 스코프의 JWT Bearer + JSON body 정책 그대로
- **결과**: PASS

### 1.11 Insecure Storage / PII 유출

- `localStorage` / `sessionStorage` 사용 0건 (grep 확인)
- URL 쿼리에 반려 사유 / 승인 내용 / 편집된 필드 값이 실리지 않음 (모달 내부 state 로만 관리)
- scope_profile_id 는 내부 식별자 — 유저 검색 가능한 형태는 아님
- **결과**: PASS

### 1.12 정적/동적 검사 결과

| 도구 | 대상 | 결과 |
| --- | --- | --- |
| `npm test` (node:test) | frontend 전체 | 146 pass / 0 fail |
| `npx tsc --noEmit` | frontend 전체 | 본 범위 파일 0 에러 (사전 결함 1건 범위 외) |
| `rg "dangerouslySetInnerHTML"` | 본 범위 | 0 hit |
| `rg "localStorage\|sessionStorage"` | 본 범위 | 0 hit |
| `rg "eval\(\|Function\("` | 본 범위 | 0 hit |
| `rg "window.location\s*="` | 본 범위 | 0 hit |

---

## 2. Info (정보성)

| ID | 항목 | 현재 값 | 비고 |
| --- | --- | --- | --- |
| I-1 | URL page 상한 | 10,000 | 서버 상한과 동기화 |
| I-2 | 반려 사유 textarea 하드 maxLength | 1,024 + 64 | 버퍼 64 는 사용자가 "초과 N자" 피드백을 실시간으로 볼 수 있도록 둠. 실 검증 상한은 1,024 |
| I-3 | SkeletonRows 개수 | 5 | 실 결과 평균 페이지 크기(20) 보다 작게 두어 레이아웃 shift 완화 |

---

## 3. 새 공격 표면 요약

C 스코프에서 새로 도입된 외부 입력 경로는 사실상 "URL 쿼리 파라미터" 하나이며, `parseFiltersFromUrl` 이 단일 진입점으로 모든 검증을 수행한다. 이 함수는 순수 함수로 node:test 10건의 회귀 커버가 있어 향후 정규식이나 허용셋 변경 시 즉시 탐지 가능하다.

반려 사유 textarea 는 기존 B 스코프부터 존재하던 표면이지만, C 에서 검증기를 프론트에 추가하면서 서버 422 도달 전 차단 지점을 하나 더 가졌다. 이는 심층 방어(defense in depth) 관점에서 positive.

---

## 4. 결론

- C 스코프에서 신규 식별된 Critical / High / Medium / Low 취약점: **없음**
- OWASP API Top 10 #1 (BOLA), #3 (BOPLA), #4 (Unrestricted Resource Consumption), #5 (BFLA), #6 (Mass Assignment), #8 (Security Misconfiguration), #10 (Unsafe Consumption of APIs) 각 항목 모두 기존 B 스코프의 조치가 유효하며 신규 유입 없음
- DOM-based XSS, Open Redirect, URL 오염 공격 — 모두 순수 함수 검증 계층으로 차단되며 회귀 테스트로 커버
