# Admin 관리자 패널 재배치 — 보안 취약점 검사 보고서

- 작성일: 2026-04-20
- 범위: `components/admin/layout/AdminHeader.tsx`, `components/admin/layout/AdminSidebar.tsx`, `components/admin/layout/AdminUserPanel.tsx`(신규)
- 기준: OWASP Top 10 (2021) 중 FE 관련 항목 + CLAUDE.md S1 보안 취약점 검사 규약

## 1. 의존성 변경 여부

**변경 없음.** `package.json`, `package-lock.json` 미수정. 신규 라이브러리 도입/제거 없음.

## 2. OWASP Top 10 (FE) 체크리스트

| 항목 | 상태 | 상세 |
| --- | --- | --- |
| A01 Broken Access Control | ✅ Pass | `AdminLayout` 의 `<AuthGuard requiredRole="ORG_ADMIN">` 가 서버와 별개로 UI 게이트 유지. `AdminUserPanel` 은 단순히 `isAuthenticated && user` 만 체크하여 UI 렌더 — ACL 결정은 기존 계층 그대로 |
| A02 Cryptographic Failures | N/A | 자격증명/토큰 처리 경로 미변경 |
| A03 Injection — XSS | ✅ Pass | 모든 외부 값(`user.display_name`, `user.email`, `user.role_name`) 은 React JSX 바인딩으로 자동 이스케이프. `dangerouslySetInnerHTML` / `eval` / `new Function` 사용 없음 |
| A04 Insecure Design | ✅ Pass | 메뉴 닫기 상태 전이 명확 (`open` boolean), 외부 클릭·ESC 양쪽 커버 |
| A05 Security Misconfiguration | ✅ Pass | 인라인 스크립트/스타일 없음. 토큰/환경변수 노출 없음 |
| A06 Vulnerable & Outdated Components | ⚠️ Deferred | `npm audit` 사내 allowlist 로 403 차단(기존 이슈 그대로). 본 변경은 의존성 추가 없음 → 공격 표면 증가 없음 |
| A07 Identification & Auth Failures | ✅ Pass | `logout()` → `router.push("/login")` 기존 흐름 유지. 재로그인 시 서버 세션·쿠키가 재발급 |
| A08 Software & Data Integrity | ✅ Pass | 외부 CDN/스크립트 추가 없음 |
| A09 Security Logging & Monitoring | N/A | 로그/감사 경로 변경 없음 |
| A10 SSRF | N/A | FE 단독 변경 |

## 3. 세부 정적 분석

### 3.1 XSS 표면 점검

| 위치 | 렌더링 대상 | 판정 |
| --- | --- | --- |
| `AdminUserPanel.tsx:displayName` | `user?.display_name \|\| user?.email \|\| "관리자"` | React 자동 이스케이프 — 안전 |
| `AdminUserPanel.tsx:initial` | `displayName.charAt(0).toUpperCase()` — 1문자 텍스트 | 안전 |
| `AdminUserPanel.tsx:user.email` | 드롭다운 상단 이메일 표시 | JSX 바인딩 — 안전 |
| `AdminUserPanel.tsx:user.role_name` | role 라벨 | JSX 바인딩 — 안전 |
| `navigate(path)` 호출 | `"/account/profile"`, `"/account"`, `"/documents"` — 전부 하드코딩 리터럴 | 사용자 입력 미포함 → Open redirect 불가 |
| `AdminSidebar.tsx` / `AdminHeader.tsx` | 변경 분에서 외부 값 삽입 경로 추가 없음 | 안전 |

### 3.2 이벤트 리스너 cleanup 점검

| 리스너 | 위치 | cleanup | 비고 |
| --- | --- | --- | --- |
| `mousedown` (menu close) | AdminUserPanel.tsx | ✅ useEffect return | 조건부 등록(`!open` early return) |
| `keydown` (ESC menu close) | AdminUserPanel.tsx | ✅ | 조건부 등록 |
| `keydown` (ESC mobile overlay) | AdminLayout.tsx | ✅ | 기존 코드 그대로 |
| `body.style.overflow` 복원 | AdminLayout.tsx | ✅ | return 으로 `""` 복원 |

Chrome 실측: `closedByEsc=true`, `closedByOutsideMousedown=true` → cleanup 및 조건부 등록 정상 동작 확인.

### 3.3 권한 / 인증 경로

- 본 패널은 **UI 게이트** 역할만 수행. 실제 권한 결정은:
  1. 네트워크 계층: `AuthGuard requiredRole="ORG_ADMIN"` (AdminLayout 레벨)
  2. 서버: 각 API 엔드포인트의 Scope Profile 검사 (S2 원칙 ⑤)
- `AdminUserPanel` 은 `isAuthenticated && user` 만 체크하여 "인증 없음" 상태에서 빈 placeholder 로 degrade (민감정보 비노출)
- 로그아웃은 기존 `useAuth().logout()` 그대로 경유 — 새 네트워크 엔드포인트 도입 없음

### 3.4 URL/Navigation 취약점

- `router.push(path)` 의 `path` 인자는 전부 하드코딩 리터럴 — 사용자 입력이나 외부 URL 미포함
- Open redirect 위험 없음

### 3.5 Inline Script / eval / dangerous*

- `dangerouslySetInnerHTML`, `eval`, `new Function`, `innerHTML=` 사용 **없음**
- 인라인 `<script>`, `javascript:` URL 사용 **없음**

## 4. 재배치에 따른 신규/제거된 공격 표면

| 항목 | 변화 | 비고 |
| --- | --- | --- |
| AdminHeader 의 로그아웃 버튼 | 제거 | `useAuth` 의존성도 제거됨 |
| AdminHeader 의 "User UI" 링크 | 제거 | 하드코딩 리터럴 → 위험 감소 효과도 중립 |
| AdminSidebar 하단 독립 "/documents" 링크 | 제거 | 동일 |
| `AdminUserPanel` 의 메뉴(프로필/계정 설정/일반 화면/로그아웃) | 신규 | 경로 전부 하드코딩, 사용자 입력 미포함 |

**순증감: 없음.** 단지 UI 엘리먼트의 위치 이동 및 통합. 공격 표면 증가 없음.

## 5. `npm audit`

- 사내 프록시 allowlist 에 audit 엔드포인트 미포함(403) — 기존 이슈
- 본 변경은 의존성 무변경이므로 추가 알려진 취약점 노출도 동일
- 후속: CI 파이프라인 SBOM 스캔(Trivy/Grype) 도입 권장

## 6. 결론

| 항목 | 결과 |
| --- | --- |
| 신규 의존성 | 없음 |
| XSS / 인젝션 표면 | 증가 없음 (React 자동 이스케이프 유지) |
| 이벤트 리스너 누수 | 없음 (Chrome 실측) |
| 인라인 스크립트/eval | 없음 |
| CSP 위반 요소 | 없음 |
| 권한/인증 경로 | 기존과 동일 (AuthGuard + 서버 ACL) |
| Open redirect | 불가 (리터럴 경로만) |

**판정: PASS.** Admin 영역의 사용자 패널 재배치는 UI 계층 내 위치·통합 변경에 한하며, 인증/권한/네트워크/데이터 흐름은 모두 동일하게 유지된다.
