# JWT 이중 토큰 생명주기 설계서

> Phase 14 — Task 14-1 산출물 2/5
> 작성일: 2026-04-13

---

## 1. 개요

Mimir 플랫폼은 Access Token(AT)과 Refresh Token(RT)의 이중 토큰 체계를 사용한다. AT는 짧은 수명으로 API 인가에 사용되고, RT는 긴 수명으로 AT 재발급에 사용된다. 이 설계서는 두 토큰의 전체 생명주기를 정의한다.

---

## 2. 토큰 사양

### 2-1. Access Token

| 항목 | 값 |
|------|-----|
| 알고리즘 | HS256 (HMAC-SHA256) |
| 서명 키 | `JWT_SECRET` 환경 변수 |
| 만료 시간 | 15분 (`JWT_ACCESS_EXPIRE_MINUTES`) |
| 저장 위치 | 프론트엔드 메모리 (JavaScript 변수/Context) |
| 전송 방식 | `Authorization: Bearer {token}` 헤더 |

**Claims 구조:**

```json
{
  "sub": "user-uuid-1234",          // 사용자 ID
  "role": "AUTHOR",                  // 현재 역할
  "type": "access",                  // 토큰 유형 (access | refresh)
  "jti": "at-uuid-5678",            // JWT ID (고유 식별자)
  "iat": 1712991600,                 // 발급 시각
  "exp": 1712992500                  // 만료 시각 (iat + 900초)
}
```

### 2-2. Refresh Token

| 항목 | 값 |
|------|-----|
| 알고리즘 | HS256 (HMAC-SHA256) |
| 서명 키 | `JWT_SECRET` 환경 변수 (AT와 동일) |
| 만료 시간 | 7일 (`JWT_REFRESH_EXPIRE_DAYS`) |
| 저장 위치 | 브라우저 HttpOnly Cookie |
| 전송 방식 | Cookie 자동 전송 (`POST /api/v1/auth/refresh`) |

**Claims 구조:**

```json
{
  "sub": "user-uuid-1234",          // 사용자 ID
  "type": "refresh",                 // 토큰 유형
  "jti": "rt-uuid-9999",            // JWT ID (DB에서 추적)
  "family_id": "fam-uuid-0000",     // Rotation Family ID
  "iat": 1712991600,                 // 발급 시각
  "exp": 1713596400                  // 만료 시각 (iat + 7일)
}
```

**Cookie 속성:**

```
Set-Cookie: refresh_token={JWT}; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth; Max-Age=604800
```

| 속성 | 값 | 목적 |
|------|-----|------|
| `HttpOnly` | true | JavaScript 접근 차단 (XSS 완화) |
| `Secure` | true | HTTPS 전송만 허용 |
| `SameSite` | Strict | 동일 사이트 요청만 쿠키 전송 (CSRF 방지) |
| `Path` | `/api/v1/auth` | 인증 엔드포인트만 쿠키 전송 범위 제한 |
| `Max-Age` | 604800 | 7일 (초) |

---

## 3. 토큰 생명주기 상태 다이어그램

### 3-1. Access Token 생명주기

```
                    ┌──────────┐
                    │  발급됨   │
                    │ (Active) │
                    └────┬─────┘
                         │
              ┌──────────┴──────────┐
              │                     │
         API 요청에 사용          시간 경과
         (Authorization 헤더)     (15분)
              │                     │
              ▼                     ▼
        ┌──────────┐         ┌──────────┐
        │  검증됨   │         │  만료됨   │
        │(Validated)│         │(Expired) │
        └──────────┘         └────┬─────┘
                                  │
                         RT로 새 AT 발급
                                  │
                                  ▼
                           ┌──────────┐
                           │  폐기됨   │
                           │(Discarded)│
                           └──────────┘
```

AT는 stateless하게 검증된다. 별도의 블랙리스트가 없으며, 만료 시간까지만 유효하다. 만료 후에는 RT를 통해 새 AT를 발급받아야 한다.

### 3-2. Refresh Token 생명주기

```
                    ┌──────────┐
                    │  발급됨   │
                    │ (Active) │
                    └────┬─────┘
                         │
           ┌─────────────┼─────────────┐
           │             │             │
      갱신 요청에 사용   시간 경과    로그아웃/
     (POST /auth/refresh) (7일)    비밀번호 변경
           │             │             │
           ▼             ▼             ▼
     ┌──────────┐  ┌──────────┐  ┌──────────┐
     │  사용됨   │  │  만료됨   │  │  폐기됨   │
     │  (Used)  │  │(Expired) │  │(Revoked) │
     └────┬─────┘  └──────────┘  └──────────┘
          │
    새 RT 발급 (같은 family_id)
          │
          ▼
    ┌──────────┐
    │  발급됨   │  ← 새 Rotation 주기 시작
    │ (Active) │
    └──────────┘
```

---

## 4. Refresh Token Rotation 상세

### 4-1. Family 개념

하나의 로그인 세션에서 발급된 모든 RT는 동일한 `family_id`를 공유한다. Family는 다음과 같이 운영된다:

```
로그인 → RT-1 (family_id=F1, jti=J1) [Active]
  │
  ├── 갱신 → RT-1 [Used], RT-2 (family_id=F1, jti=J2) [Active]
  │
  ├── 갱신 → RT-2 [Used], RT-3 (family_id=F1, jti=J3) [Active]
  │
  └── ... (최대 7일까지 Rotation 반복)
```

### 4-2. 탈취 감지 (Reuse Detection)

```
정상 시나리오:
  RT-1 사용 → RT-2 발급 → RT-2 사용 → RT-3 발급 (정상)

탈취 시나리오:
  RT-1 사용 → RT-2 발급 (정상 사용자에게)
  RT-1 재사용 시도 (공격자) → Family F1 전체 폐기!

  결과:
    RT-1: Used → 이미 사용됨 → 탈취 감지
    RT-2: Active → Revoked (family 전체 폐기)
    정상 사용자도 재로그인 필요 (안전을 위한 트레이드오프)
```

### 4-3. `refresh_tokens` 테이블 설계

```sql
CREATE TABLE refresh_tokens (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  jti        VARCHAR(255) UNIQUE NOT NULL,
  family_id  UUID NOT NULL,
  user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  used_at    TIMESTAMPTZ,              -- NULL이면 미사용 (Active)
  revoked_at TIMESTAMPTZ,              -- NULL이면 유효
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_rt_jti       ON refresh_tokens(jti);
CREATE INDEX idx_rt_family    ON refresh_tokens(family_id);
CREATE INDEX idx_rt_user      ON refresh_tokens(user_id);
CREATE INDEX idx_rt_expires   ON refresh_tokens(expires_at);
```

**상태 판별 규칙:**

| 조건 | 상태 |
|------|------|
| `revoked_at IS NULL AND used_at IS NULL AND expires_at > now()` | Active (사용 가능) |
| `used_at IS NOT NULL AND revoked_at IS NULL` | Used (정상 사용됨) |
| `revoked_at IS NOT NULL` | Revoked (폐기됨) |
| `expires_at <= now()` | Expired (만료됨) |

---

## 5. 토큰 발급 시점

| 이벤트 | AT 발급 | RT 발급 | Family |
|--------|---------|---------|--------|
| 이메일/비밀번호 로그인 | 새 AT | 새 RT | 새 family_id |
| GitLab OAuth 로그인 | 새 AT | 새 RT | 새 family_id |
| 토큰 갱신 (POST /auth/refresh) | 새 AT | 새 RT | 기존 family_id 유지 |
| 비밀번호 재설정 후 로그인 | 새 AT | 새 RT | 새 family_id |

---

## 6. 토큰 폐기 시점

| 이벤트 | AT 처리 | RT 처리 |
|--------|---------|---------|
| 로그아웃 | 프론트엔드 메모리에서 삭제 | 현재 family 전체 revoke |
| 비밀번호 변경 | (자연 만료 대기) | 해당 사용자의 모든 family revoke |
| RT Reuse 감지 | (자연 만료 대기) | 해당 family 전체 revoke |
| 관리자에 의한 사용자 비활성화 | (자연 만료 대기) | 해당 사용자의 모든 family revoke |
| RT 만료 (7일) | - | 자연 만료 (DB에서 배치 삭제) |

---

## 7. 프론트엔드 토큰 관리

### 7-1. AT 저장 및 사용

```
AuthContext (React Context)
├── accessToken: string | null       ← 메모리 저장
├── isAuthenticated: boolean
├── user: { id, email, role, ... }
└── refreshAccessToken(): Promise     ← silent refresh 호출
```

AT는 React Context 또는 모듈 스코프 변수에 저장한다. `localStorage`나 `sessionStorage`에 저장하지 않는다 (XSS 공격 시 탈취 가능).

### 7-2. Silent Refresh 전략

```
1. AT 발급 시 만료 시각(exp) 기록
2. 타이머 설정: exp - 60초(1분 전)에 자동 갱신
3. 갱신 실패 시: 로그인 페이지로 리다이렉트
4. 페이지 새로고침 시: AT가 없으므로 즉시 POST /auth/refresh 호출
5. 다중 탭: BroadcastChannel로 갱신 결과 공유
```

### 7-3. API 요청 인터셉터

```
Axios/Fetch 인터셉터:
1. 요청 전: Authorization 헤더에 AT 추가
2. 401 수신 시:
   a. 갱신 중이면 대기열에 추가
   b. 갱신 중 아니면 POST /auth/refresh 호출
   c. 갱신 성공 → 대기열의 모든 요청 재시도
   d. 갱신 실패 → 로그인 페이지로 리다이렉트
```

---

## 8. 만료된 RT 정리 배치

```python
# 주기: 매일 03:00 (배치 작업)
# 대상: expires_at < now() - 7 days (만료 후 7일 보관)

DELETE FROM refresh_tokens
WHERE expires_at < now() - INTERVAL '7 days';
```

만료 후 7일간 보관하는 이유는, 탈취 감지 시 이미 만료된 family의 이력을 확인할 수 있도록 하기 위함이다.

---

## 9. 토큰 검증 플로우

### 9-1. AT 검증 (모든 보호된 API 요청)

```
1. Authorization 헤더에서 Bearer 토큰 추출
2. JWT 디코딩 (HS256, JWT_SECRET)
3. type == "access" 확인
4. exp > now() 확인
5. sub에서 user_id 추출
6. role에서 역할 추출
7. ActorContext 생성 → 기존 dependencies.py 체인에 합류
```

기존 `dependencies.py`의 `_try_bearer_token()` 함수에서 처리된다. `type` claim 검증만 추가하면 된다.

### 9-2. RT 검증 (POST /auth/refresh)

```
1. Cookie에서 refresh_token 추출
2. JWT 디코딩 (HS256, JWT_SECRET)
3. type == "refresh" 확인
4. exp > now() 확인
5. jti로 DB 조회 (refresh_tokens 테이블)
6. used_at IS NULL 확인 (미사용)
7. revoked_at IS NULL 확인 (폐기되지 않음)
8. 모든 검증 통과 → 새 AT/RT 발급
```
