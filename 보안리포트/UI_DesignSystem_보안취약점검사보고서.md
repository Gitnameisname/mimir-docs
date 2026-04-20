# UI 디자인 시스템 개편 — 보안 취약점 검사 보고서

- 작성일: 2026-04-20
- 범위: `frontend/src/app/globals.css`, `components/layout/**` (신규 `SidebarUserPanel.tsx` 포함, `Header.tsx` 사용자 메뉴 제거), `components/button/Button.tsx`, `components/form/SearchInput.tsx`, `components/page/**`, `features/documents/DocumentListPage.tsx`, `features/documents/ReviewsPage.tsx`, `components/feedback/SkeletonBlock.tsx`
- 기준: OWASP Top 10 (2021) 중 FE 관련 항목 + CLAUDE.md 개발 후 보안 취약점 검사 규약 (S1)
- 검사 방법: ① 변경 파일 정적 리뷰 ② 의존성 변경 여부 확인 ③ 인라인 SVG·URL·이벤트 리스너 점검 ④ `npm audit` 시도

## 1. 의존성 변경 여부

**변경 없음.** `package.json`, `package-lock.json` 미수정. 새로운 라이브러리 도입/제거 없음.

## 2. OWASP Top 10 (FE) 체크리스트

| 항목 | 상태 | 상세 |
| --- | --- | --- |
| A03 Injection — XSS (Stored/Reflected/DOM) | ✅ Pass | 모든 사용자 입력은 React JSX 바인딩을 통해 자동 이스케이프. `dangerouslySetInnerHTML` 사용 없음. `doc.title`, `doc.created_by_name` 등 외부 데이터는 텍스트 노드로만 렌더링. 검색 쿼리는 `router.push` 로 이동 전 `encodeURIComponent()` 적용됨 (Header.tsx:30) |
| A01 Broken Access Control | N/A | 권한 관련 코드 변경 없음. `AuthGuard`, `hasMinimumRole` 사용 유지 |
| A02 Cryptographic Failures | N/A | 자격증명/토큰 처리 경로 미변경 |
| A04 Insecure Design | ✅ Pass | 에러 상태(`ErrorState`)·로딩 상태(`SkeletonRow`) 분리 유지 |
| A05 Security Misconfiguration | ✅ Pass | CSP/helmet 관련 설정 미변경. 인라인 스타일/스크립트 없음 (Tailwind 클래스 + CSS 변수) |
| A06 Vulnerable & Outdated Components | ⚠️ Deferred | `npm audit` 이 사내 allowlist 로 403 차단 → 오프라인 검증 불가. 단, **본 변경으로 도입된 의존성이 없으므로 공격 표면 증가 없음** |
| A07 Identification & Auth Failures | N/A | 인증 로직 변경 없음 |
| A08 Software & Data Integrity | ✅ Pass | 외부 CDN/스크립트 추가 없음. 시스템 폰트 스택만 사용 (폐쇄망 S2-⑦ 준수) |
| A09 Security Logging & Monitoring | N/A | 로그/오딧 경로 변경 없음 |
| A10 Server-Side Request Forgery | N/A | FE 단독 변경 |

## 3. 세부 정적 분석

### 3.1 XSS 표면 점검

| 위치 | 렌더링 대상 | 위험도 | 판정 |
| --- | --- | --- | --- |
| `DocumentListPage.tsx:L(doc.title)` | `{doc.title}` | Low | React 자동 이스케이프 — 안전 |
| `DocumentListPage.tsx:L(doc.created_by_name)` | `{doc.created_by_name \|\| "—"}` | Low | 안전 |
| `DocumentListPage.tsx:L(filters.q)` | `{`검색: ${filters.q}`}` + href 에는 noUrlInjection | Low | 텍스트 노드로만 노출. 검색 진입 시 `encodeURIComponent` (Header) |
| `Header.tsx:L(displayName/email/role_name)` | JSX 바인딩 | Low | 안전 |
| `SearchInput.tsx:placeholder/aria` | 정적 문자열 | Low | 안전 |
| `SELECT_CLS` 내부 SVG data-URL | 정적 인라인 SVG (화살표) | Low | 하드코딩 상수 — 동적 보간 없음 → 안전 |
| `SidebarUserPanel.tsx:displayName/email/role_name` | JSX 바인딩 | Low | React 자동 이스케이프 — 안전. `user?.display_name \|\| user?.email \|\| "User"` 폴백 |
| `SidebarUserPanel.tsx:initial` | `displayName.charAt(0).toUpperCase()` | Low | 단일 문자 텍스트 노드 — XSS 경로 없음 |
| `SidebarUserPanel.tsx:router.push(path)` | 하드코딩 경로 (`/account/profile`, `/account`, `/admin`) | Low | 사용자 입력 미포함, 리터럴 문자열 — Open redirect 가능성 없음 |

`dangerouslySetInnerHTML`, `eval`, `new Function`, `Function()` 호출 **없음** (이번 변경분 기준).

### 3.2 이벤트 리스너 메모리/로직 점검

| 리스너 | 위치 | 정리(cleanup) | 비고 |
| --- | --- | --- | --- |
| `keydown` (Cmd/Ctrl+K) | Header.tsx | ✅ | `key.toLowerCase()` 로 수정되어 대소문자 무관 (기존 코드에서 대문자 "K" 케이스는 캐치 못함 → 개선). 사용자 메뉴 관련 리스너는 모두 `SidebarUserPanel`로 이동 |
| `matchMedia change` | Sidebar.tsx:useIsMobile | ✅ | `removeEventListener` 수행 |
| `keydown` (ESC overlay close) | Sidebar.tsx | ✅ | `isMobile && !collapsed` 조건부 |
| `mousedown` (user menu close) | SidebarUserPanel.tsx | ✅ useEffect return | 조건부 등록 (`!open` early return) — 외부 클릭으로 메뉴 닫기 |
| `keydown` (ESC user menu close) | SidebarUserPanel.tsx | ✅ | 동일 — Chrome 에서 실측 확인 (menuClosedByEsc=true) |

메모리 누수 없음. **unmount 시 모든 리스너 해제됨.**

### 3.3 URL/Navigation 취약점

- `router.push` 대상 URL 에 사용자 입력이 포함될 때 `encodeURIComponent` 적용 (Header.tsx:31)
- `<Link href={`/documents/${doc.id}`}>`: `doc.id` 는 서버 UUID 이므로 신뢰 가능. 본 변경에서 FE 가 위조할 수 없는 경로로 유지됨
- Open redirect: 외부 URL 로의 동적 navigation 없음

### 3.4 CSS/SVG 주의 사항

- CSS 변수 사용은 XSS 경로가 아님 (브라우저가 값을 literal 로 해석)
- `background-image: url("data:image/svg+xml,...")` 는 Tailwind 4 엔진이 CSS 로 처리. SVG 내부에 스크립트 없고 정적 상수 → 안전
- `scrollbar-color: rgba(...)` 역시 CSS-only

### 3.5 CSP 호환성

현재 프로젝트 CSP 설정은 이번 변경 범위 밖이나, 변경된 코드는 다음을 따른다:

- Inline script: 없음
- Inline style: Tailwind JIT 의 class 기반 스타일만 (CSP `style-src 'self' 'unsafe-hashes'?` 수준에서 기존 정책 유지 가능)
- `javascript:` URL: 없음
- `eval` 계열: 없음

## 4. Container Query / matchMedia 관련 보안 검토

- `@container` 쿼리는 브라우저 네이티브 CSS — JS 보안 영향 없음
- `matchMedia("(max-width: 767px)")` 는 브라우저 네이티브 — 외부 입력 미반영

## 5. `npm audit` 결과

```
npm warn audit 403 Forbidden - POST https://registry.npmjs.org/-/npm/v1/security/audits/quick
"x-proxy-error": "blocked-by-allowlist"
```

**사내 allowlist 에 audit 엔드포인트 미등록.** 본 변경은 **의존성을 추가하지 않으므로** 취약점 증가 가능성은 없다. 단, 기존 의존성(특히 `@types/dompurify`, `tiptap`, `recharts` 등)에 대한 정기 스캔은 별도 필요.

**후속 조치(권장)**
- 사내 프록시/레지스트리의 audit 엔드포인트 allowlist 추가, 또는
- `npm audit --registry=사내-미러` 적용, 또는
- CI 파이프라인에서 SBOM 스캔(예: Trivy, Grype) 도입

## 6. 결론

| 항목 | 결과 |
| --- | --- |
| 신규 의존성 | 없음 |
| XSS / 인젝션 표면 | 증가 없음 (React 자동 이스케이프 유지) |
| 이벤트 리스너 누수 | 없음 |
| 인라인 스크립트/eval | 없음 |
| CSP 위반 요소 | 없음 |
| 외부 리소스(폰트/CDN) | 추가 없음 (폐쇄망 S2-⑦ 준수) |

**판정: PASS.** 본 변경은 프레젠테이션 계층에 국한되며, 공격 표면을 증가시키지 않는다. `npm audit` 온라인 검증은 환경 제약으로 보류하되, **의존성 변경이 없어 알려진 취약점 노출도는 이전 상태와 동일**하다.

## 7. 사용자 패널 재배치(Header → Sidebar) 보안 재검토 (2026-04-20 추가)

사용자 메뉴를 `Header.tsx`에서 `SidebarUserPanel.tsx`로 이동한 변경에 대한 보안 재검토 결과:

| 항목 | 결과 | 비고 |
| --- | --- | --- |
| 인증 상태 체크 | ✅ | `isAuthenticated && user` 가드 유지. 미인증 시 "로그인되지 않음" 폴백 |
| 권한 체크 | ✅ | `hasMinimumRole("ORG_ADMIN")` 기반 관리자 메뉴 조건부 렌더 유지 — 하드코딩된 role 문자열은 기존 Header 와 동일 수준, 서버 사이드 ACL 은 이 UI 분기와 독립적으로 강제됨 |
| 로그아웃 흐름 | ✅ | `logout()` → `router.push("/login")` 로 세션 정리. 에러 시 `setLoggingOut(false)` 로 상태 복구 |
| 내비게이션 경로 | ✅ | `/account/profile`, `/account`, `/admin` — 전부 하드코딩 리터럴, 사용자 입력 미포함 |
| 이벤트 리스너 누수 | ✅ | `mousedown`/`keydown` 모두 `useEffect` cleanup 수행 — Chrome 실측: menuClosedByOutsideMousedown=true, menuClosedByEsc=true |
| aria 속성 | ✅ | `aria-haspopup`, `aria-expanded`, `role="menu"`, `role="menuitem"` 유지 |
| 민감정보 노출 | ✅ | 이메일·역할 표시는 본인 정보만, localStorage/쿠키 저장 없음 |

**재검토 결론:** 사용자 패널의 물리적 위치만 변경됐으며 인증/권한/네트워크 동작은 동일. 공격 표면 증감 없음.
