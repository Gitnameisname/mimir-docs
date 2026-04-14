# Task 14-8 검수보고서: 프론트엔드 인증 상태 관리

**작성일**: 2026-04-14
**검수 대상**: 프론트엔드 인증 상태 관리 (AuthContext, API 인터셉터, AuthGuard, 페이지 통합)

---

## 1. 구현 범위

### 1.1 AuthContext (`contexts/AuthContext.tsx`)
- `AuthState` (user, accessToken, isAuthenticated, isLoading) + `useReducer` 기반 상태 관리
- 모듈 수준 `_accessToken` 변수로 AT 메모리 저장 (localStorage/sessionStorage 사용 금지)
- `getAccessToken()` / `setAccessToken()` 모듈 함수 (API 인터셉터에서 직접 접근)
- 앱 마운트 시 `attemptSilentRefresh()` → RT Cookie로 AT 갱신 + 프로필 로드
- JWT exp 디코딩 → 만료 5분 전 자동 갱신 타이머 (`scheduleTokenRefresh`)
- `_refreshPromise` 큐잉: 동시 다발 401 시 refresh 1회만 실행
- `login()`, `loginWithGitLab()`, `handleOAuthCallback()`, `logout()`
- `hasRole()`, `hasMinimumRole()` 역할 계층 확인 (VIEWER ~ SUPER_ADMIN)
- `attemptSilentRefreshForInterceptor()` 외부 노출 (API 클라이언트용)
- `window.__mimir_at` 레거시 호환 브릿지 (점진적 제거 예정)

### 1.2 API 클라이언트 인터셉터 (`lib/api/client.ts`)
- `getAccessToken()`으로 Bearer Token 자동 첨부
- 401 응답 시 `attemptSilentRefreshForInterceptor()` → 원래 요청 재시도 (1회 제한)
- 갱신 실패 시 `/login?redirect=...` 리다이렉트 (공개 페이지 제외)
- 개발 환경 fallback: AT 없을 때만 X-Actor-Id/X-Actor-Role 헤더

### 1.3 AuthGuard (`components/auth/AuthGuard.tsx`)
- `isLoading` → `SkeletonLayout` (깜빡임 방지)
- 미인증 → `/login?redirect=...` 리다이렉트
- `requiredRole` 미충족 → `ForbiddenContent` 표시

### 1.4 보조 컴포넌트
- `SkeletonLayout`: 헤더 + 콘텐츠 스켈레톤, `role="status"`, `aria-live`, `sr-only`
- `ForbiddenContent`: 403 UI, 홈/로그인 링크, `aria-hidden` 아이콘
- `/forbidden` 페이지: ForbiddenContent 래핑

### 1.5 기존 페이지 통합
- **로그인 페이지**: `useAuth().login` 사용, `?redirect=` 지원, 이미 인증 시 자동 리다이렉트
- **OAuth 콜백**: `useAuth().handleOAuthCallback` 사용, `window.__mimir_at` 직접 할당 제거
- **Account API 클라이언트**: `getAccessToken()` 사용으로 전환
- **Account 레이아웃**: `<AuthGuard>` 래핑
- **Providers**: `<AuthProvider>` 감싸기 (`QueryClientProvider` 내부)

---

## 2. 검수 결과

| 카테고리 | 검증 항목 수 | PASS | FAIL |
|---------|:-----------:|:----:|:----:|
| AuthContext (CTX) | 30 | 30 | 0 |
| API Client (CLI) | 15 | 15 | 0 |
| AuthGuard (GRD) | 11 | 11 | 0 |
| SkeletonLayout (SKL) | 5 | 5 | 0 |
| ForbiddenContent (FRB) | 5 | 5 | 0 |
| Forbidden Page (FPG) | 2 | 2 | 0 |
| Providers (PRV) | 4 | 4 | 0 |
| Login Page (LGN) | 10 | 10 | 0 |
| OAuth Callback (CBK) | 7 | 7 | 0 |
| Account API (ACC) | 4 | 4 | 0 |
| Account Layout (ALY) | 3 | 3 | 0 |
| 보안 (SEC) | 4 | 4 | 0 |
| 접근성 (A11Y) | 5 | 5 | 0 |
| **합계** | **105** | **105** | **0** |

---

## 3. UI 디자인 리뷰 (5회 수행)

| 라운드 | 개선 내용 |
|:------:|---------|
| 1 | 시맨틱 HTML (`<header>`, `<main>`), `aria-busy`, `aria-hidden` 추가 |
| 2 | 터치 타겟 44px 보장, 모바일 flex-col 전환, 버튼 패딩 증가 |
| 3 | focus-visible 통일, 인풋 포커스 링 일관성, 트랜지션 duration 통일 |
| 4 | 에러 알림 아이콘 추가, 로딩 상태 개선, leading-relaxed 적용 |
| 5 | form spacing 조정, 라벨 색상 강화, 구분선/링크 간격 미세 조정 |

---

## 4. 생성/수정 파일 목록

| 파일 | 상태 |
|-----|------|
| `frontend/src/contexts/AuthContext.tsx` | 신규 |
| `frontend/src/lib/api/client.ts` | 수정 (인터셉터 통합) |
| `frontend/src/components/auth/AuthGuard.tsx` | 신규 |
| `frontend/src/components/auth/SkeletonLayout.tsx` | 신규 |
| `frontend/src/components/auth/ForbiddenContent.tsx` | 신규 |
| `frontend/src/app/forbidden/page.tsx` | 신규 |
| `frontend/src/lib/providers.tsx` | 수정 (AuthProvider 추가) |
| `frontend/src/app/login/page.tsx` | 수정 (AuthContext 통합) |
| `frontend/src/app/auth/callback/page.tsx` | 수정 (AuthContext 통합) |
| `frontend/src/lib/api/account.ts` | 수정 (getAccessToken 전환) |
| `frontend/src/app/account/layout.tsx` | 수정 (AuthGuard 적용) |
| `backend/tests/unit/test_auth_state_phase14_8.py` | 신규 (검수 스크립트) |

---

## 5. 결론

**105/105 검증 항목 통과**. AuthContext 기반 인증 상태 관리가 정상 구현되었으며, 기존 페이지(로그인, OAuth 콜백, 계정 관리)가 모두 새 인증 시스템으로 전환 완료되었다. AT는 메모리에만 저장되며, 401 자동 갱신, 라우트 보호, 역할 기반 접근 제어가 설계대로 동작한다.
