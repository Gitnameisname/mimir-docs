# 기존 인프라 확장 전략 문서

> Phase 14 — Task 14-1 산출물 3/5
> 작성일: 2026-04-13

---

## 1. 개요

Phase 14는 기존 Phase 0~13에서 구축한 인증/인가 인프라 위에 구축된다. 기존 코드의 안정성을 유지하면서 새로운 인증 기능을 추가하기 위한 모듈별 확장 전략을 정의한다.

---

## 2. 모듈별 현황 및 확장 계획

### 2-1. `tokens.py` — JWT 토큰 관리

**현재 상태:**

```python
# 현재 구현
def create_access_token(actor_id: str, role: str, expires_minutes: int | None = None) -> str:
    payload = {
        "sub": actor_id,
        "role": role,
        "iat": datetime.utcnow(),
        "exp": datetime.utcnow() + timedelta(minutes=expires_minutes or settings.jwt_expire_minutes),
    }
    return jwt.encode(payload, settings.jwt_secret, algorithm="HS256")

def decode_access_token(token: str) -> dict:
    return jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])
```

**확장 방향:**

| 변경 사항 | 상세 |
|-----------|------|
| `create_access_token()` 수정 | `jti`(UUID), `type="access"` claim 추가 |
| `create_refresh_token()` 신규 | `jti`, `family_id`, `type="refresh"`, 7일 만료 |
| `decode_access_token()` 수정 | `type="access"` 검증 로직 추가 |
| `decode_refresh_token()` 신규 | `type="refresh"` 검증, DB 조회 연계 |
| `create_password_reset_token()` 신규 | `type="password_reset"`, 30분 만료, 1회용 |

**하위 호환성:** 기존 `create_access_token()` 호출부는 동일하게 동작한다. 새 claim(`jti`, `type`)이 추가되지만 기존 `decode_access_token()`은 추가 claim을 무시하므로 점진적 마이그레이션이 가능하다. 다만, Phase 14 완료 후에는 모든 토큰에 `type` claim이 포함되어야 하므로, `decode_access_token()`에서 `type` 검증을 필수화한다.

**구현 Task:** Task 14-2 (백엔드 인증 API)

---

### 2-2. `session.py` — Valkey 세션 관리

**현재 상태:**

```python
# 현재 구현
_SESSION_PREFIX = "session:"
_SESSION_TTL_SECONDS = settings.jwt_expire_minutes * 60

async def create_session(user_id: str, role: str, ...) -> str:
    token = secrets.token_urlsafe(32)
    key = f"{_SESSION_PREFIX}{token}"
    await valkey.set(key, json.dumps({...}), ex=_SESSION_TTL_SECONDS)
    return token

async def resolve_session(token: str) -> dict | None:
    key = f"{_SESSION_PREFIX}{token}"
    data = await valkey.get(key)
    return json.loads(data) if data else None
```

**확장 방향:**

| 변경 사항 | 키 패턴 | TTL | 용도 |
|-----------|---------|-----|------|
| 로그인 시도 카운터 | `login_attempts:{email}` | 900초 (15분) | Brute Force 방어 |
| OAuth state 저장 | `oauth_state:{state}` | 600초 (10분) | CSRF + PKCE 검증 |
| RT 블랙리스트 (선택) | `rt_blacklist:{family_id}` | RT 만료 시간 | 빠른 폐기 조회 |

**기존 세션과의 관계:** 기존 `session:` 키 구조는 변경하지 않는다. Cookie 기반 세션은 Phase 14에서 JWT+RT 체계로 대체되지만, 내부 서비스 간 세션(`X-Service-Token` 등)은 기존 방식을 유지한다. 향후 Phase에서 기존 세션 방식의 deprecation 여부를 결정한다.

**새로운 함수:**

```python
# 추가될 함수들

async def increment_login_attempts(email: str) -> int:
    """로그인 실패 횟수 증가. 첫 시도 시 15분 TTL 설정."""

async def get_login_attempts(email: str) -> int:
    """현재 실패 횟수 조회."""

async def clear_login_attempts(email: str) -> None:
    """로그인 성공 시 카운터 초기화."""

async def store_oauth_state(state: str, data: dict) -> None:
    """OAuth state + PKCE code_verifier 저장. 10분 TTL."""

async def get_and_delete_oauth_state(state: str) -> dict | None:
    """OAuth state 조회 및 즉시 삭제 (1회용)."""
```

**구현 Task:** Task 14-2 (로그인 시도 제한), Task 14-4 (OAuth state)

---

### 2-3. `dependencies.py` — Actor 추출 체인

**현재 상태:**

```python
# 5단계 추출 체인 (우선순위 순)
async def get_current_actor(request: Request) -> ActorContext:
    actor = _try_service_token(request)       # 1. X-Service-Token (HMAC)
    if not actor:
        actor = _try_bearer_token(request)    # 2. Authorization: Bearer (JWT)
    if not actor:
        actor = await _try_api_key(request)   # 3. X-API-Key (SHA-256)
    if not actor:
        actor = await _try_session(request)   # 4. Session Cookie
    if not actor:
        actor = _anonymous_actor()            # 5. Anonymous
    return actor
```

**확장 방향:** 변경 없음. Phase 14에서 발급하는 JWT는 기존 `_try_bearer_token()` 경로로 자연스럽게 처리된다.

다만, `_try_bearer_token()` 내부에서 `type` claim을 검증해야 한다:

```python
def _try_bearer_token(request: Request) -> ActorContext | None:
    token = _extract_bearer(request)
    if not token:
        return None
    try:
        payload = decode_access_token(token)
        # Phase 14 추가: type claim 검증
        if payload.get("type") != "access":
            return None
        ...
    except jwt.PyJWTError:
        return None
```

이 수정은 하위 호환성에 영향을 주지 않는다. 기존에 `type` claim 없이 발급된 토큰은 Phase 14 마이그레이션 기간 중 자연 만료된다 (최대 2시간). 마이그레이션 기간에는 `type` claim이 없는 토큰도 허용하도록 점진적 적용한다.

**구현 Task:** Task 14-2 (토큰 검증 강화)

---

### 2-4. `authorization.py` — RBAC 매트릭스

**현재 상태:**

```python
# 기존 액션 목록 (25개)
RBAC_MATRIX = {
    "VIEWER":      {"document.read", "search.basic", ...},
    "AUTHOR":      {"document.read", "document.create", ...},
    "REVIEWER":    {...},
    "APPROVER":    {...},
    "ORG_ADMIN":   {...},
    "SUPER_ADMIN": {"*"},  # 모든 권한
}
```

**확장 방향:** 새로운 액션을 추가한다.

| 새 액션 | 허용 역할 | 용도 |
|---------|----------|------|
| `account.read` | 모든 인증 사용자 | 자신의 계정 정보 조회 |
| `account.update` | 모든 인증 사용자 | 자신의 프로필 수정 |
| `account.delete` | 모든 인증 사용자 | 자신의 계정 삭제 |
| `admin.read` | ORG_ADMIN, SUPER_ADMIN | 관리 대시보드 조회 |
| `admin.write` | SUPER_ADMIN | 시스템 설정 변경 |
| `admin.user_manage` | ORG_ADMIN, SUPER_ADMIN | 사용자 관리 |
| `alert.read` | ORG_ADMIN, SUPER_ADMIN | 알림 규칙/이력 조회 |
| `alert.write` | SUPER_ADMIN | 알림 규칙 생성/수정 |
| `settings.read` | ORG_ADMIN, SUPER_ADMIN | 시스템 설정 조회 |
| `settings.write` | SUPER_ADMIN | 시스템 설정 변경 |

**구현 Task:** Task 14-3 (회원가입/비밀번호 재설정에서 account 액션), Task 14-10~14-15 (admin 액션)

---

### 2-5. `models.py` — Actor 모델

**현재 상태:**

```python
class AuthMethod(str, Enum):
    SESSION = "session"
    BEARER = "bearer"
    API_KEY = "api_key"
    INTERNAL_SERVICE = "internal_service"
```

**확장 방향:**

| 변경 사항 | 상세 |
|-----------|------|
| `AuthMethod.OAUTH` 추가 | OAuth 인증 경로 식별 (선택적) |

OAuth로 로그인하더라도 최종적으로 JWT가 발급되므로, `AuthMethod.BEARER`로 처리해도 무방하다. 다만, 감사 로그에서 인증 경로를 구분하려면 `AuthMethod.OAUTH`를 추가하는 것이 유용하다. 이는 JWT의 별도 claim (`auth_method: "oauth"`)으로도 구현 가능하므로, 구현 시점에서 결정한다.

---

### 2-6. `config.py` — 설정 관리

**현재 상태:**

```python
class Settings(BaseSettings):
    jwt_secret: str = "dev-secret"
    jwt_expire_minutes: int = 120
    internal_service_secret: str = "dev-internal-secret"
    debug: bool = True
    # ... DB, Valkey 등
```

**확장 방향:**

```python
class Settings(BaseSettings):
    # 기존 유지
    jwt_secret: str = "dev-secret"
    jwt_expire_minutes: int = 15           # 기본값 변경: 120 → 15
    internal_service_secret: str = "dev-internal-secret"

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

    # OAuth 토큰 암호화
    oauth_token_encryption_key: str = ""
```

**`jwt_expire_minutes` 기본값 변경 (120 → 15):**

기존에는 AT 만료가 120분이었으나, RT 도입으로 15분으로 단축한다. 기존 코드에서 `settings.jwt_expire_minutes`를 참조하는 곳은 `session.py`의 `_SESSION_TTL_SECONDS` 뿐이며, 이는 별도로 조정한다.

**Production 검증 확장:** 기존 `_validate_production_settings()`에 GitLab OAuth 설정 필수 검증을 추가한다 (단, OAuth가 활성화된 경우에만).

**구현 Task:** Task 14-2 (JWT/비밀번호 설정), Task 14-4 (GitLab OAuth 설정), Task 14-5 (이메일 설정)

---

## 3. 데이터베이스 확장

### 3-1. `users` 테이블 ALTER

```sql
ALTER TABLE users
  ADD COLUMN IF NOT EXISTS password_hash    VARCHAR(255),
  ADD COLUMN IF NOT EXISTS is_email_verified BOOLEAN DEFAULT FALSE,
  ADD COLUMN IF NOT EXISTS auth_provider     VARCHAR(50) DEFAULT 'local',
  ADD COLUMN IF NOT EXISTS last_login_at     TIMESTAMPTZ,
  ADD COLUMN IF NOT EXISTS failed_login_count INTEGER DEFAULT 0,
  ADD COLUMN IF NOT EXISTS locked_until       TIMESTAMPTZ;
```

기존 `users` 테이블의 기존 컬럼에 영향을 주지 않는 ADD COLUMN만 수행한다.

### 3-2. 신규 테이블

| 테이블 | 용도 | 구현 Task |
|--------|------|-----------|
| `oauth_accounts` | GitLab OAuth 계정 연결 | Task 14-4 |
| `refresh_tokens` | RT Rotation 관리 | Task 14-2 |
| `password_reset_tokens` | 비밀번호 재설정 토큰 | Task 14-3 |
| `system_settings` | 동적 시스템 설정 | Task 14-11 |
| `alert_rules` | 알림 규칙 | Task 14-13 |
| `alert_history` | 알림 이력 | Task 14-13 |

---

## 4. 마이그레이션 전략

Phase 14 배포 시 기존 인증 세션과의 호환성을 보장하기 위한 마이그레이션 순서:

```
1단계: DB 마이그레이션 (테이블 추가, users ALTER)
   → 기존 기능에 영향 없음 (ADD COLUMN만)

2단계: config.py 확장 배포
   → 새 환경 변수 추가 (기본값으로 동작)

3단계: tokens.py 확장 배포
   → 새 토큰은 type claim 포함, 기존 토큰도 허용

4단계: 인증 API 배포 (/auth/login, /auth/register 등)
   → 새로운 엔드포인트 추가 (기존 경로 충돌 없음)

5단계: OAuth 배포 (/auth/oauth/gitlab/*)
   → 새로운 엔드포인트 추가

6단계: dependencies.py type 검증 강화
   → 기존 type 없는 토큰 자연 만료 후 (최대 15분)

7단계: 프론트엔드 배포
   → 로그인 UI, AuthContext 적용
```

각 단계는 독립적으로 롤백 가능하다.

---

## 5. 외부 라이브러리 의존성

| 라이브러리 | 용도 | 현재 설치 여부 | 구현 Task |
|-----------|------|--------------|-----------|
| `passlib[bcrypt]` | 비밀번호 해싱 | 미설치 | Task 14-2 |
| `authlib` | OAuth2/OIDC 클라이언트 | 미설치 | Task 14-4 |
| `cryptography` | AES-256-GCM (OAuth 토큰 암호화) | 미설치 | Task 14-4 |
| `PyJWT` | JWT 생성/검증 | 설치됨 | - |
| `httpx` | GitLab API 호출 | 설치됨 | - |
| `cronstrue` (JS) | Cron 표현식 → 사람 읽기 변환 | 미설치 | Task 14-14 |
| `recharts` (JS) | 모니터링 차트 | 미설치 | Task 14-12 |

설치 전 취약점 점검을 수행한다 (pip-audit, npm audit).
