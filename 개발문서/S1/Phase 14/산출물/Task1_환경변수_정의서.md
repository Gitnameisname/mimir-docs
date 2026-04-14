# 환경 변수 정의서

> Phase 14 — Task 14-1 산출물 4/5
> 작성일: 2026-04-13

---

## 1. 개요

Phase 14에서 추가되는 모든 환경 변수를 정의한다. 기존 환경 변수와의 네이밍 규칙 일관성을 유지하며, 개발/운영 환경별 기본값과 필수 여부를 명시한다.

기존 `config.py` 네이밍 규칙: `snake_case` (Pydantic BaseSettings가 환경 변수를 자동 매핑, 대소문자 무관)

---

## 2. 기존 환경 변수 (변경 사항)

| 변수명 | 현재 기본값 | 변경 기본값 | 변경 사유 |
|--------|-----------|-----------|----------|
| `JWT_EXPIRE_MINUTES` | 120 | 15 | RT 도입으로 AT 수명 단축. 기존 120분은 RT 없이 운영하기 위한 임시 값이었음 |

---

## 3. 신규 환경 변수

### 3-1. JWT 확장

| 변수명 | 타입 | 기본값 | 필수(운영) | 설명 |
|--------|------|--------|-----------|------|
| `JWT_REFRESH_EXPIRE_DAYS` | int | 7 | N | Refresh Token 만료 기간 (일) |

JWT_SECRET은 기존 변수를 AT와 RT 모두에 공유하여 사용한다. AT와 RT를 별도 키로 서명하는 방식도 가능하나, `type` claim으로 토큰 유형을 구분하므로 단일 키로 충분하다.

---

### 3-2. GitLab OAuth

| 변수명 | 타입 | 기본값 | 필수(운영) | 설명 |
|--------|------|--------|-----------|------|
| `GITLAB_BASE_URL` | str | `https://gitlab.com` | N | GitLab 인스턴스 URL. Self-managed 시 변경 |
| `GITLAB_CLIENT_ID` | str | `""` | Y* | GitLab OAuth Application ID |
| `GITLAB_CLIENT_SECRET` | str | `""` | Y* | GitLab OAuth Application Secret |
| `GITLAB_REDIRECT_URI` | str | `""` | Y* | OAuth 콜백 URL (`https://{domain}/api/v1/auth/oauth/gitlab/callback`) |

\* OAuth 기능 활성화 시 필수. `GITLAB_CLIENT_ID`가 비어있으면 OAuth 기능이 비활성화된다.

**GitLab Application 설정 가이드:**

```
1. GitLab → Settings → Applications
2. Name: Mimir
3. Redirect URI: https://mimir.example.com/api/v1/auth/oauth/gitlab/callback
4. Confidential: Yes
5. Scopes: openid, profile, email, read_user
6. 생성 후 Application ID → GITLAB_CLIENT_ID
7. Secret → GITLAB_CLIENT_SECRET
```

**OIDC Discovery URL 파생:**

```
{GITLAB_BASE_URL}/.well-known/openid-configuration
```

이 URL에서 `authorization_endpoint`, `token_endpoint`, `userinfo_endpoint`, `jwks_uri`를 자동으로 획득한다.

---

### 3-3. 비밀번호 정책

| 변수명 | 타입 | 기본값 | 필수(운영) | 설명 |
|--------|------|--------|-----------|------|
| `BCRYPT_COST_FACTOR` | int | 12 | N | bcrypt 해싱 비용. 높을수록 안전하지만 느림 |
| `LOGIN_MAX_ATTEMPTS` | int | 5 | N | 계정 잠금까지 허용 로그인 실패 횟수 |
| `LOGIN_LOCKOUT_MINUTES` | int | 15 | N | 계정 잠금 시간 (분) |

**bcrypt cost factor 벤치마크 참고:**

| Cost | 해싱 시간 (평균) | 권장 환경 |
|------|---------------|----------|
| 10 | ~100ms | 개발/테스트 |
| 12 | ~300ms | 운영 (권장) |
| 14 | ~1.2s | 높은 보안 요구 |

Cost 12는 OWASP 권장 최소값이며, 로그인 요청당 ~300ms의 CPU 시간을 소모한다. 동시 로그인 요청이 많은 환경에서는 별도의 인증 워커를 분리하는 것을 고려한다.

---

### 3-4. 이메일 (SMTP)

| 변수명 | 타입 | 기본값 | 필수(운영) | 설명 |
|--------|------|--------|-----------|------|
| `SMTP_HOST` | str | `""` | Y* | SMTP 서버 호스트 |
| `SMTP_PORT` | int | 587 | N | SMTP 서버 포트 (587=STARTTLS, 465=SSL) |
| `SMTP_USERNAME` | str | `""` | Y* | SMTP 인증 사용자명 |
| `SMTP_PASSWORD` | str | `""` | Y* | SMTP 인증 비밀번호 |
| `SMTP_FROM_ADDRESS` | str | `noreply@mimir.example.com` | N | 발신 이메일 주소 |
| `SMTP_USE_TLS` | bool | True | N | TLS 사용 여부 |

\* 이메일 기능(비밀번호 재설정, 알림) 활성화 시 필수. `SMTP_HOST`가 비어있으면 이메일 기능이 비활성화된다.

**개발 환경 팁:** 개발 시에는 MailHog 등의 로컬 SMTP 서버를 사용한다:

```
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_USERNAME=
SMTP_PASSWORD=
SMTP_USE_TLS=false
```

---

### 3-5. OAuth 토큰 암호화

| 변수명 | 타입 | 기본값 | 필수(운영) | 설명 |
|--------|------|--------|-----------|------|
| `OAUTH_TOKEN_ENCRYPTION_KEY` | str | `""` | Y* | AES-256-GCM 암호화 키 (32바이트, base64 인코딩) |

\* OAuth 기능 활성화 시 필수.

**키 생성 방법:**

```bash
python -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

이 키는 GitLab에서 받은 OAuth access_token/refresh_token을 DB에 저장할 때 AES-256-GCM으로 암호화하는 데 사용한다. 키 로테이션은 향후 지원 예정이다.

---

## 4. 환경별 설정 예시

### 4-1. 개발 환경 (`.env.development`)

```bash
# JWT
JWT_SECRET=dev-secret-key-not-for-production
JWT_EXPIRE_MINUTES=15
JWT_REFRESH_EXPIRE_DAYS=7

# GitLab OAuth (로컬 GitLab 또는 gitlab.com 개발 앱)
GITLAB_BASE_URL=https://gitlab.com
GITLAB_CLIENT_ID=dev-client-id
GITLAB_CLIENT_SECRET=dev-client-secret
GITLAB_REDIRECT_URI=http://localhost:3000/api/v1/auth/oauth/gitlab/callback

# 비밀번호
BCRYPT_COST_FACTOR=10
LOGIN_MAX_ATTEMPTS=5
LOGIN_LOCKOUT_MINUTES=15

# 이메일 (MailHog)
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_USERNAME=
SMTP_PASSWORD=
SMTP_FROM_ADDRESS=noreply@mimir.local
SMTP_USE_TLS=false

# OAuth 토큰 암호화
OAUTH_TOKEN_ENCRYPTION_KEY=ZGV2LWVuY3J5cHRpb24ta2V5LW5vdC1mb3ItcHJvZA==
```

### 4-2. 운영 환경 (`.env.production`)

```bash
# JWT
JWT_SECRET=<강력한-랜덤-문자열-64자-이상>
JWT_EXPIRE_MINUTES=15
JWT_REFRESH_EXPIRE_DAYS=7

# GitLab OAuth
GITLAB_BASE_URL=https://gitlab.company.com
GITLAB_CLIENT_ID=<실제-application-id>
GITLAB_CLIENT_SECRET=<실제-application-secret>
GITLAB_REDIRECT_URI=https://mimir.company.com/api/v1/auth/oauth/gitlab/callback

# 비밀번호
BCRYPT_COST_FACTOR=12
LOGIN_MAX_ATTEMPTS=5
LOGIN_LOCKOUT_MINUTES=15

# 이메일 (실제 SMTP)
SMTP_HOST=smtp.company.com
SMTP_PORT=587
SMTP_USERNAME=mimir-noreply
SMTP_PASSWORD=<smtp-비밀번호>
SMTP_FROM_ADDRESS=noreply@mimir.company.com
SMTP_USE_TLS=true

# OAuth 토큰 암호화
OAUTH_TOKEN_ENCRYPTION_KEY=<base64-인코딩된-32바이트-랜덤-키>
```

---

## 5. `config.py` 확장 구현 명세

```python
class Settings(BaseSettings):
    # ─── 기존 유지 ───
    jwt_secret: str = "dev-secret"
    jwt_expire_minutes: int = 15           # 120 → 15 변경
    internal_service_secret: str = "dev-internal-secret"
    debug: bool = True

    # ─── Phase 14 신규 ───

    # JWT 확장
    jwt_refresh_expire_days: int = 7

    # GitLab OAuth
    gitlab_base_url: str = "https://gitlab.com"
    gitlab_client_id: str = ""
    gitlab_client_secret: str = ""
    gitlab_redirect_uri: str = ""

    # 비밀번호 정책
    bcrypt_cost_factor: int = 12
    login_max_attempts: int = 5
    login_lockout_minutes: int = 15

    # 이메일 (SMTP)
    smtp_host: str = ""
    smtp_port: int = 587
    smtp_username: str = ""
    smtp_password: str = ""
    smtp_from_address: str = "noreply@mimir.example.com"
    smtp_use_tls: bool = True

    # OAuth 토큰 암호화
    oauth_token_encryption_key: str = ""

    # ─── 헬퍼 프로퍼티 ───

    @property
    def is_oauth_enabled(self) -> bool:
        """GitLab OAuth가 활성화되어 있는지 확인"""
        return bool(self.gitlab_client_id)

    @property
    def is_email_enabled(self) -> bool:
        """이메일 기능이 활성화되어 있는지 확인"""
        return bool(self.smtp_host)
```

**Production 검증 확장:**

```python
def _validate_production_settings(self):
    # 기존 검증 유지
    assert self.jwt_secret != "dev-secret"
    assert self.postgres_password
    assert self.internal_service_secret != "dev-internal-secret"

    # Phase 14 추가 검증
    if self.is_oauth_enabled:
        assert self.gitlab_client_secret, "GITLAB_CLIENT_SECRET required when OAuth enabled"
        assert self.gitlab_redirect_uri, "GITLAB_REDIRECT_URI required when OAuth enabled"
        assert self.oauth_token_encryption_key, "OAUTH_TOKEN_ENCRYPTION_KEY required when OAuth enabled"

    if self.is_email_enabled:
        assert self.smtp_username, "SMTP_USERNAME required when email enabled"
        assert self.smtp_password, "SMTP_PASSWORD required when email enabled"
```

---

## 6. 보안 고려사항

| 변수 | 민감도 | 보관 방법 |
|------|--------|----------|
| `JWT_SECRET` | 최상 | Secret Manager / 환경 변수 (절대 코드/Git에 포함 금지) |
| `GITLAB_CLIENT_SECRET` | 상 | Secret Manager / 환경 변수 |
| `SMTP_PASSWORD` | 상 | Secret Manager / 환경 변수 |
| `OAUTH_TOKEN_ENCRYPTION_KEY` | 최상 | Secret Manager (키 분실 시 모든 OAuth 토큰 무효화) |
| `INTERNAL_SERVICE_SECRET` | 최상 | Secret Manager / 환경 변수 |

`.env` 파일은 `.gitignore`에 포함되어야 하며, `.env.example` 파일에 빈 값으로 변수 목록만 관리한다.
