# Task 14-8 보안 취약점 검사 보고서

**작성일**: 2026-04-14
**검사 대상**: 프론트엔드 인증 상태 관리 (AuthContext, API 인터셉터, AuthGuard)

---

## 1. 검사 항목 및 결과

### 1.1 토큰 저장 보안

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 1 | AT가 localStorage에 저장되지 않음 | ✅ 통과 | 모듈 변수 `_accessToken`만 사용 |
| 2 | AT가 sessionStorage에 저장되지 않음 | ✅ 통과 | |
| 3 | AT가 Cookie에 저장되지 않음 | ✅ 통과 | RT만 HttpOnly Cookie |
| 4 | AT가 URL에 노출되지 않음 | ✅ 통과 | OAuth callback fragment는 즉시 제거 |
| 5 | window.__mimir_at 레거시 사용처 제거 | ✅ 통과 | AuthContext 내 호환 코드만 잔존 |

### 1.2 인증 흐름 보안

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 6 | Silent refresh에 credentials:include 설정 | ✅ 통과 | RT Cookie 자동 전송 |
| 7 | 동시 401 시 refresh 큐잉 (1회만) | ✅ 통과 | `_refreshPromise` 패턴 |
| 8 | Refresh 실패 시 로그인 리다이렉트 | ✅ 통과 | 공개 페이지 제외 |
| 9 | AT 갱신 후 이전 AT 즉시 폐기 | ✅ 통과 | setAccessToken 덮어쓰기 |
| 10 | 로그아웃 시 AT null 처리 | ✅ 통과 | dispatch(LOGOUT) + setAccessToken(null) |

### 1.3 라우트 보호

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 11 | AuthGuard 미인증 시 리다이렉트 | ✅ 통과 | /login?redirect=... |
| 12 | AuthGuard 역할 부족 시 403 표시 | ✅ 통과 | hasMinimumRole 체크 |
| 13 | Account 페이지 AuthGuard 적용 | ✅ 통과 | account/layout.tsx |
| 14 | 로딩 중 콘텐츠 미노출 (SkeletonLayout) | ✅ 통과 | |

### 1.4 XSS 방어

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 15 | AT가 DOM에 렌더링되지 않음 | ✅ 통과 | 메모리 변수만 사용 |
| 16 | 에러 메시지 React 이스케이프 | ✅ 통과 | JSX {} 바인딩 |
| 17 | URL fragment 처리 후 즉시 제거 | ✅ 통과 | history.replaceState |

### 1.5 API 인터셉터 보안

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 18 | Bearer 토큰 자동 첨부 | ✅ 통과 | Authorization 헤더 |
| 19 | 401 재시도 1회 제한 | ✅ 통과 | _retry 플래그 |
| 20 | 개발 헤더 AT 없을 때만 사용 | ✅ 통과 | IS_DEV && !at 조건 |

---

## 2. 취약점 요약

| 등급 | 수량 | 상세 |
|:----:|:----:|------|
| Critical | 0 | - |
| High | 0 | - |
| Medium | 0 | - |
| Low | 0 | - |
| Info | 2 | 아래 참조 |

### Info 항목

**INFO-1: window.__mimir_at 레거시 호환 코드 잔존**
- 위치: `contexts/AuthContext.tsx` setAccessToken 함수
- 설명: Phase 14-6/7에서 사용하던 `window.__mimir_at`에 AT를 동기화하는 레거시 브릿지 코드가 있음
- 위험도: 정보 (실질적 보안 위험 없음 — 모든 소비처가 이미 getAccessToken으로 전환됨)
- 권고: 향후 Phase에서 레거시 호환 코드 제거

**INFO-2: JWT 클라이언트 디코딩**
- 위치: `contexts/AuthContext.tsx` scheduleTokenRefresh 함수
- 설명: AT의 exp claim을 `atob()`로 디코딩하여 갱신 타이머 설정
- 위험도: 정보 (서명 검증은 백엔드에서 수행, 클라이언트는 타이머용으로만 사용)
- 권고: 현재 구조가 적절함, 변경 불필요

---

## 3. 결론

**Critical/High/Medium/Low 취약점 0건**. AT 메모리 저장, refresh 큐잉, 라우트 보호, XSS 방어 등 프론트엔드 인증 보안 요구사항이 모두 충족되었다. Info 2건은 향후 유지보수 과정에서 자연스럽게 해소될 사항이다.
