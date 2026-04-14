# Task 14-8. 프론트엔드 인증 상태 관리

## 1. 작업 목적

프론트엔드 전체의 인증 상태를 관리하고, 라우트 보호 및 토큰 자동 갱신을 구현한다.

이 작업의 목표는 다음과 같다.

- AuthContext (React Context + useReducer) 구현
- Access Token 메모리 저장 및 관리
- 자동 토큰 갱신 (silent refresh)
- AuthGuard (라우트 보호 HOC)
- API 클라이언트 인터셉터 (토큰 자동 첨부 및 갱신)
- 미인증/권한 부족 페이지

---

## 2. 작업 범위

### 포함 범위

- AuthContext Provider 구현
- useAuth 훅
- AuthGuard HOC (역할 기반 라우트 보호)
- API 클라이언트(axios/fetch) 인터셉터
- Silent refresh (만료 5분 전 자동 갱신)
- 401 응답 시 자동 갱신 시도
- 미인증 리다이렉트 (`/login`)
- 403 Forbidden 페이지
- 인증 확인 중 스켈레톤 UI

### 제외 범위

- 로그인/회원가입 UI (Task 14-6에서 완료)
- 계정 관리 UI (Task 14-7에서 완료)

---

## 3. 선행 조건

- Task 14-3 JWT 이중 토큰 체계 완료 (POST /auth/refresh API)
- Task 14-6 로그인 UI 완료
- Phase 6/7 프론트엔드 구조 이해 (Next.js App Router)

---

## 4. 주요 구현 대상

### 4-1. AuthContext 설계

```typescript
// contexts/AuthContext.tsx

interface AuthState {
  user: User | null;
  accessToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;  // 초기 인증 확인 중
}

type AuthAction =
  | { type: "LOGIN_SUCCESS"; payload: { user: User; accessToken: string } }
  | { type: "LOGOUT" }
  | { type: "TOKEN_REFRESHED"; payload: { accessToken: string } }
  | { type: "SET_LOADING"; payload: boolean };

// Context Provider
function AuthProvider({ children }) {
  const [state, dispatch] = useReducer(authReducer, initialState);

  // 앱 마운트 시 silent refresh 시도 (Refresh Token Cookie 존재 확인)
  useEffect(() => {
    attemptSilentRefresh();
  }, []);

  // 토큰 만료 5분 전 자동 갱신 타이머
  useEffect(() => {
    if (state.accessToken) {
      const timer = scheduleTokenRefresh(state.accessToken);
      return () => clearTimeout(timer);
    }
  }, [state.accessToken]);
}
```

---

### 4-2. useAuth 훅

```typescript
// hooks/useAuth.ts

function useAuth() {
  const context = useContext(AuthContext);
  return {
    user: context.user,
    isAuthenticated: context.isAuthenticated,
    isLoading: context.isLoading,
    login: (email, password) => { ... },
    loginWithGitLab: () => { ... },
    logout: () => { ... },
    hasRole: (role: string) => { ... },
  };
}
```

---

### 4-3. Silent Refresh 구현

```typescript
async function attemptSilentRefresh(): Promise<boolean> {
  try {
    const res = await fetch("/api/v1/auth/refresh", {
      method: "POST",
      credentials: "include",  // Cookie 전송
    });
    if (res.ok) {
      const data = await res.json();
      dispatch({ type: "TOKEN_REFRESHED", payload: { accessToken: data.access_token } });
      // 프로필 조회
      const profile = await fetchProfile(data.access_token);
      dispatch({ type: "LOGIN_SUCCESS", payload: { user: profile, accessToken: data.access_token } });
      return true;
    }
    return false;
  } catch {
    return false;
  }
}

function scheduleTokenRefresh(accessToken: string) {
  const payload = JSON.parse(atob(accessToken.split(".")[1]));
  const expiresAt = payload.exp * 1000;
  const refreshAt = expiresAt - 5 * 60 * 1000;  // 만료 5분 전
  const delay = Math.max(refreshAt - Date.now(), 0);
  return setTimeout(() => attemptSilentRefresh(), delay);
}
```

---

### 4-4. API 클라이언트 인터셉터

```typescript
// lib/apiClient.ts

const apiClient = axios.create({ baseURL: "/api/v1" });

// 요청 인터셉터: Access Token 자동 첨부
apiClient.interceptors.request.use((config) => {
  const token = getAccessToken();  // 메모리에서 읽기
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// 응답 인터셉터: 401 시 자동 갱신 후 재시도
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401 && !error.config._retry) {
      error.config._retry = true;
      const refreshed = await attemptSilentRefresh();
      if (refreshed) {
        error.config.headers.Authorization = `Bearer ${getAccessToken()}`;
        return apiClient(error.config);
      }
      // 갱신 실패 → 로그아웃
      dispatch({ type: "LOGOUT" });
      window.location.href = "/login";
    }
    return Promise.reject(error);
  }
);
```

---

### 4-5. AuthGuard HOC

```typescript
// components/AuthGuard.tsx

function AuthGuard({ children, requiredRole }: { children: ReactNode; requiredRole?: string }) {
  const { isAuthenticated, isLoading, user } = useAuth();
  const router = useRouter();

  if (isLoading) {
    return <SkeletonLayout />;  // 인증 확인 중
  }

  if (!isAuthenticated) {
    router.replace(`/login?redirect=${encodeURIComponent(window.location.pathname)}`);
    return null;
  }

  if (requiredRole && !hasMinimumRole(user.role, requiredRole)) {
    return <ForbiddenPage />;  // 403
  }

  return <>{children}</>;
}

// 역할 계층 확인
const ROLE_HIERARCHY = ["VIEWER", "AUTHOR", "REVIEWER", "APPROVER", "ORG_ADMIN", "SUPER_ADMIN"];
function hasMinimumRole(userRole: string, required: string): boolean {
  return ROLE_HIERARCHY.indexOf(userRole) >= ROLE_HIERARCHY.indexOf(required);
}
```

---

### 4-6. 로그인 후 리다이렉트

```
로그인 페이지 접근 시:
  /login?redirect=/documents/123

로그인 성공 후:
  → redirect 파라미터가 있으면 해당 경로로
  → 없으면 / (메인 페이지)로
```

---

## 5. 산출물

1. `contexts/AuthContext.tsx` — AuthProvider, authReducer
2. `hooks/useAuth.ts` — useAuth 훅
3. `lib/apiClient.ts` — API 클라이언트 인터셉터
4. `components/AuthGuard.tsx` — 라우트 보호 HOC
5. `components/SkeletonLayout.tsx` — 인증 확인 중 스켈레톤
6. `pages/403.tsx` — Forbidden 페이지
7. 기존 페이지에 AuthGuard 적용

---

## 6. 완료 기준

- 페이지 새로고침 시 silent refresh로 로그인 상태가 유지된다
- Access Token 만료 5분 전에 자동 갱신이 실행된다
- 401 응답 시 자동 갱신 후 원래 요청이 재시도된다
- 갱신 실패 시 로그인 페이지로 리다이렉트된다
- 권한 부족 시 403 페이지가 표시된다
- 인증 확인 중 스켈레톤 UI가 표시된다 (깜빡임 방지)
- 로그인 후 원래 접근하려던 페이지로 리다이렉트된다

---

## 7. Codex 작업 지침

- Access Token은 메모리(변수/Context)에만 저장한다 — localStorage/sessionStorage 사용 금지
- Silent refresh 실패 시 로그인 페이지 리다이렉트 전 현재 URL을 redirect 파라미터로 전달한다
- 동시에 여러 401 응답이 발생해도 refresh 요청은 1회만 실행되도록 큐잉한다
- 역할 계층(ROLE_HIERARCHY)은 기존 `authorization.py`와 일치시킨다
- AuthGuard는 서버 사이드 렌더링에서도 동작하도록 클라이언트 전용 처리한다
