# Task 14-1. 인증 아키텍처 설계

## 1. 작업 목적

Phase 14 전체 인증 시스템의 기술 방향과 구조를 확정한다.

이 작업의 목표는 다음과 같다.

- GitLab OAuth2/OIDC + 자체 JWT 인증 플로우 설계
- JWT 이중 토큰(Access/Refresh) 체계 설계
- 비밀번호 인증 플로우 설계
- 기존 Phase 0~13 인증 인프라와의 통합 전략 수립
- 보안 위협 모델링 및 대응 전략 확정

---

## 2. 작업 범위

### 포함 범위

- 인증 플로우 시퀀스 다이어그램 (로컬 로그인, GitLab OAuth, 토큰 갱신)
- JWT Access/Refresh Token 생명주기 설계
- 기존 `tokens.py`, `session.py`, `dependencies.py` 확장 전략
- 보안 위협 모델링 (OWASP Authentication 기준)
- 환경 변수 및 설정 값 정의

### 제외 범위

- 실제 코드 구현 (Task 14-2~14-5에서 다룸)
- UI 설계 (Task 14-6~14-8에서 다룸)

---

## 3. 선행 조건

- Phase 3 인증/인가 구조 이해 (`dependencies.py` Actor 추출 체인)
- `tokens.py` — 기존 Access Token (HS256, 15분) 구현 이해
- `session.py` — Valkey 기반 세션 관리 이해
- `authorization.py` — RBAC 매트릭스 및 역할 계층 이해

---

## 4. 주요 설계 대상

### 4-1. 인증 플로우 시퀀스 다이어그램

다음 3가지 플로우를 시퀀스 다이어그램으로 작성:

```
1. 이메일/비밀번호 로그인 플로우
   User → POST /auth/login (email, password)
        → 비밀번호 검증 (bcrypt)
        → failed_login_count 확인/갱신
        → Access Token + Refresh Token 발급
        → Set-Cookie: refresh_token (HttpOnly, Secure, SameSite=Strict)
        → Response: { access_token, expires_in }

2. GitLab OAuth2/OIDC 플로우
   User → GET /auth/oauth/gitlab
        → 302 Redirect to GitLab (Authorization Code + PKCE)
        → GitLab 인증 완료 → Callback
        → GET /auth/oauth/gitlab/callback?code=...&state=...
        → Backend: Code → GitLab Token 교환
        → Backend: GitLab userinfo → 이메일로 계정 조회/생성
        → Access Token + Refresh Token 발급
        → 302 Redirect to Frontend (with tokens)

3. 토큰 갱신 플로우
   Client → POST /auth/refresh (Cookie: refresh_token)
          → Refresh Token 유효성 검증
          → Rotation: 새 RT 발급, 기존 RT 폐기
          → 새 Access Token + 새 Refresh Token 반환
```

---

### 4-2. JWT 이중 토큰 생명주기

| 단계 | Access Token | Refresh Token |
|------|-------------|---------------|
| 발급 | 로그인 성공 시 | 로그인 성공 시 |
| 전송 | Response Body | Set-Cookie (HttpOnly) |
| 저장 | 프론트엔드 메모리 | 브라우저 Cookie |
| 사용 | Authorization: Bearer 헤더 | POST /auth/refresh 요청 |
| 갱신 | RT로 재발급 | Rotation (사용 시 새 RT) |
| 폐기 | 만료 대기 (15분) | 로그아웃 시 블랙리스트, family 폐기 |

Refresh Token Rotation 상세:

```
Family 개념:
  - 최초 로그인 시 family_id(UUID) 생성
  - 같은 family의 RT는 한 번만 사용 가능
  - RT 사용 시 → 새 RT 발급 (같은 family_id)
  - 이미 사용된 RT 재사용 감지 시 → 해당 family 전체 폐기 (탈취 대응)
```

---

### 4-3. 기존 인프라 확장 전략

| 기존 모듈 | 현재 상태 | 확장 방향 |
|----------|---------|---------|
| `tokens.py` | Access Token만 (HS256, `jwt_expire_minutes`) | Refresh Token 발급/검증 추가, jti/family claim 추가 |
| `session.py` | Valkey 세션 (session:{token}) | RT 블랙리스트 (rt_blacklist:{family_id}), 로그인 시도 카운터 (login_attempts:{email}) |
| `dependencies.py` | 5단계 Actor 추출 체인 | OAuth 인증 후 발급된 JWT도 기존 Bearer 체인으로 처리 (변경 불필요) |
| `authorization.py` | 25개 액션 RBAC | `account.*`, `alert.*`, `settings.*` 액션 추가 |
| `config.py` | jwt_secret, debug 등 | GitLab OAuth 설정, bcrypt 설정, 이메일 설정 추가 |

---

### 4-4. 환경 변수 설계

```
# GitLab OAuth
GITLAB_BASE_URL=https://gitlab.example.com    # Self-managed (기본: https://gitlab.com)
GITLAB_CLIENT_ID=...
GITLAB_CLIENT_SECRET=...
GITLAB_REDIRECT_URI=https://mimir.example.com/api/v1/auth/oauth/gitlab/callback

# JWT
JWT_SECRET=...                    # 기존 유지
JWT_ACCESS_EXPIRE_MINUTES=15      # 기존 jwt_expire_minutes 대체
JWT_REFRESH_EXPIRE_DAYS=7

# 비밀번호
BCRYPT_COST_FACTOR=12
LOGIN_MAX_ATTEMPTS=5
LOGIN_LOCKOUT_MINUTES=15

# 이메일
SMTP_HOST=...
SMTP_PORT=587
SMTP_USERNAME=...
SMTP_PASSWORD=...
SMTP_FROM_ADDRESS=noreply@mimir.example.com

# OAuth 토큰 암호화
OAUTH_TOKEN_ENCRYPTION_KEY=...    # AES-256-GCM 키 (32바이트 base64)
```

---

### 4-5. 보안 위협 모델링

| 위협 | 공격 벡터 | 대응 전략 | 구현 위치 |
|------|---------|---------|---------|
| Credential Stuffing | 유출된 이메일/비밀번호 조합 | 로그인 시도 제한 (5회/15분) | `session.py` (Valkey) |
| Token Theft (XSS) | XSS로 Access Token 탈취 | AT: 메모리 저장, RT: HttpOnly Cookie | 프론트엔드, 쿠키 설정 |
| Token Replay | 탈취된 RT 재사용 | RT Rotation + family 폐기 | `refresh_tokens` 테이블 |
| CSRF | 악성 사이트에서 쿠키 자동 전송 | SameSite=Strict + CSRF 토큰 | 쿠키 설정, 미들웨어 |
| OAuth State Tampering | state 파라미터 위조 | PKCE + state 검증 | OAuth 플로우 |
| Account Takeover | 비밀번호 재설정 악용 | JWT 기반 재설정 토큰 (30분, 1회용) | 이메일 인증 |

---

## 5. 산출물

1. 인증 플로우 시퀀스 다이어그램 3종 (로컬, OAuth, 토큰 갱신)
2. JWT 이중 토큰 생명주기 설계서
3. 기존 인프라 확장 전략 문서
4. 환경 변수 정의서
5. 보안 위협 모델링 문서

---

## 6. 완료 기준

- 3가지 인증 플로우(로컬/OAuth/갱신)의 시퀀스 다이어그램이 작성되어 있다
- JWT 이중 토큰 생명주기가 명확히 정의되어 있다
- 기존 모듈 확장 전략이 수립되어 이후 Task에서 바로 구현 가능하다
- 보안 위협 모델링이 완료되어 각 위협의 대응 전략이 명시되어 있다

---

## 7. Codex 작업 지침

- 실제 코드 구현은 하지 않는다 (설계 중심)
- 기존 `tokens.py`, `session.py`, `dependencies.py`의 현재 구조를 정확히 반영한다
- OWASP Authentication Cheat Sheet를 참조하여 보안 설계를 검토한다
- 환경 변수명은 기존 `config.py` 네이밍 규칙을 따른다
