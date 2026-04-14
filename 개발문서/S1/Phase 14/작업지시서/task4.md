# Task 14-4. GitLab OAuth2/OIDC 로그인 구현

## 1. 작업 목적

GitLab을 OAuth2/OIDC 제공자로 연동하여 GitLab 계정으로 Mimir에 로그인할 수 있도록 한다.

이 작업의 목표는 다음과 같다.

- GitLab OAuth2 Authorization Code Flow + PKCE 구현
- GitLab OIDC Discovery 자동 설정
- `oauth_accounts` 테이블 생성
- 이메일 기반 계정 자동 연결/생성
- GitLab 토큰 암호화 저장 (AES-256-GCM)
- Self-managed 및 GitLab.com 모두 지원

---

## 2. 작업 범위

### 포함 범위

- `oauth_accounts` 테이블 생성
- `GET /api/v1/auth/oauth/gitlab` — OAuth 인증 시작 (GitLab 리다이렉트)
- `GET /api/v1/auth/oauth/gitlab/callback` — 콜백 처리
- GitLab OIDC Discovery (`/.well-known/openid-configuration`) 활용
- PKCE (code_verifier/code_challenge) 구현
- 이메일 기반 계정 연결 로직
- GitLab 토큰 암호화 저장
- `config.py` GitLab 설정 추가

### 제외 범위

- 다른 OAuth 제공자 (Google, GitHub 등 — 향후 확장)
- 프론트엔드 GitLab 로그인 버튼 UI (Task 14-6에서 구현)
- 계정 관리에서의 GitLab 연결/해제 UI (Task 14-7에서 구현)

---

## 3. 선행 조건

- Task 14-1 인증 아키텍처 설계 완료
- Task 14-3 JWT 이중 토큰 체계 구현 완료 (토큰 발급 함수 사용)
- GitLab Application 등록 (Client ID, Client Secret 확보)
- `authlib` 라이브러리 이해

---

## 4. 주요 구현 대상

### 4-1. oauth_accounts 테이블 생성

```sql
CREATE TABLE oauth_accounts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  provider        VARCHAR(50)  NOT NULL,
  provider_uid    VARCHAR(255) NOT NULL,
  provider_email  VARCHAR(255),
  provider_name   VARCHAR(255),
  avatar_url      VARCHAR(500),
  access_token    TEXT,
  refresh_token   TEXT,
  token_expires_at TIMESTAMPTZ,
  raw_profile     JSONB DEFAULT '{}',
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now(),
  UNIQUE (provider, provider_uid)
);

CREATE INDEX idx_oauth_accounts_user ON oauth_accounts(user_id);
CREATE INDEX idx_oauth_accounts_provider ON oauth_accounts(provider, provider_uid);
```

---

### 4-2. GitLab OAuth 클라이언트 설정

```python
# app/api/auth/oauth_client.py

from authlib.integrations.starlette_client import OAuth

oauth = OAuth()

def configure_gitlab(settings):
    """GitLab OIDC 클라이언트 등록 (Discovery 자동 설정)"""
    gitlab_base = settings.gitlab_base_url.rstrip("/")
    oauth.register(
        name="gitlab",
        client_id=settings.gitlab_client_id,
        client_secret=settings.gitlab_client_secret,
        server_metadata_url=f"{gitlab_base}/.well-known/openid-configuration",
        client_kwargs={
            "scope": "openid profile email read_user",
            "code_challenge_method": "S256",  # PKCE
        },
    )
```

---

### 4-3. OAuth 인증 시작 엔드포인트

```python
# GET /api/v1/auth/oauth/gitlab

처리 흐름:
1. PKCE code_verifier 생성 → Valkey에 임시 저장 (state와 매핑, TTL 10분)
2. authlib의 authorize_redirect() 호출
3. GitLab 인증 페이지로 302 리다이렉트

GitLab 리다이렉트 URL 예시:
  https://gitlab.example.com/oauth/authorize
    ?client_id=...
    &redirect_uri=...
    &response_type=code
    &scope=openid+profile+email+read_user
    &state=<random>
    &code_challenge=<S256 hash>
    &code_challenge_method=S256
```

---

### 4-4. OAuth 콜백 처리

```python
# GET /api/v1/auth/oauth/gitlab/callback?code=...&state=...

처리 흐름:
1. state 검증 (CSRF 방어)
2. Authorization Code → GitLab Token 교환 (authlib)
3. GitLab userinfo 엔드포인트 호출 또는 id_token 디코딩
4. 프로필 정보 추출: sub(uid), email, name, avatar_url
5. 계정 연결/생성 로직 (4-5 참조)
6. Access Token + Refresh Token 발급 (Task 14-3 함수 사용)
7. 프론트엔드 리다이렉트 (AT는 URL fragment, RT는 Cookie)

프론트엔드 리다이렉트:
  302 → https://mimir.example.com/auth/callback#access_token=...&expires_in=900
  + Set-Cookie: refresh_token=...; HttpOnly; Secure; SameSite=Strict
```

---

### 4-5. 계정 연결/생성 로직

```
GitLab 프로필 수신 후:

1. oauth_accounts에서 (provider=gitlab, provider_uid) 조회
   → 존재: 기존 계정으로 로그인

2. 없으면: users에서 email로 조회
   → 존재: 기존 사용자에 GitLab 계정 연결 (oauth_accounts INSERT)
   → 미존재: 새 사용자 생성 + GitLab 계정 연결
     - 사용자 생성: status=ACTIVE, role_name=VIEWER, auth_provider=gitlab
     - email_verified=TRUE (GitLab이 이메일 인증 보장)

3. oauth_accounts에 GitLab 토큰 저장 (암호화)
4. users.last_login_at 갱신
5. 감사 이벤트 기록
```

---

### 4-6. GitLab 토큰 암호화 저장

```python
# app/api/auth/encryption.py

from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os, base64

def encrypt_token(plaintext: str, key: bytes) -> str:
    """AES-256-GCM 암호화, nonce 포함하여 base64 반환"""
    aesgcm = AESGCM(key)
    nonce = os.urandom(12)
    ciphertext = aesgcm.encrypt(nonce, plaintext.encode(), None)
    return base64.b64encode(nonce + ciphertext).decode()

def decrypt_token(encrypted: str, key: bytes) -> str:
    """AES-256-GCM 복호화"""
    data = base64.b64decode(encrypted)
    nonce, ciphertext = data[:12], data[12:]
    aesgcm = AESGCM(key)
    return aesgcm.decrypt(nonce, ciphertext, None).decode()
```

`OAUTH_TOKEN_ENCRYPTION_KEY`는 `config.py`에서 환경 변수로 로드.

---

### 4-7. config.py 확장

```python
# GitLab OAuth 설정 추가
gitlab_base_url: str = "https://gitlab.com"
gitlab_client_id: str = ""
gitlab_client_secret: str = ""
gitlab_redirect_uri: str = ""
oauth_token_encryption_key: str = ""  # 32바이트 base64

# Production 검증에 추가
def _validate_production_secrets(self):
    # 기존 검증 + GitLab 설정 검증
    if not self.gitlab_client_id or not self.gitlab_client_secret:
        raise ValueError("GitLab OAuth 설정이 필요합니다")
```

---

## 5. 산출물

1. `oauth_accounts` 테이블 생성 DDL
2. `app/api/auth/oauth_client.py` — GitLab OIDC 클라이언트 설정
3. `app/api/auth/encryption.py` — AES-256-GCM 토큰 암호화
4. `GET /api/v1/auth/oauth/gitlab` 엔드포인트
5. `GET /api/v1/auth/oauth/gitlab/callback` 엔드포인트
6. 계정 연결/생성 서비스 로직
7. `config.py` GitLab 설정 추가
8. 통합 테스트 (OAuth 플로우 mock 테스트)

---

## 6. 완료 기준

- GitLab 로그인 버튼 클릭 → GitLab 인증 → Mimir 로그인이 완료된다
- Self-managed GitLab (`GITLAB_BASE_URL` 변경)에서도 동작한다
- 동일 이메일의 로컬 계정이 있으면 자동 연결된다
- 최초 GitLab 로그인 시 VIEWER 역할로 계정이 자동 생성된다
- GitLab 토큰이 AES-256-GCM으로 암호화 저장된다
- PKCE가 적용되어 Authorization Code Interception이 방지된다
- OAuth state 검증이 동작하여 CSRF가 방어된다

---

## 7. Codex 작업 지침

- `authlib`를 `requirements.txt`에 추가한다
- `cryptography`를 `requirements.txt`에 추가한다 (AES-256-GCM용)
- GitLab Application 등록 시 Redirect URI를 정확히 맞춰야 한다
- OAuth 콜백에서 에러 응답(access_denied 등)도 처리한다
- PKCE code_verifier는 Valkey에 저장하고 10분 TTL을 설정한다
- GitLab이 이메일을 반환하지 않는 경우(비공개 설정)를 에러 처리한다
- 프론트엔드 리다이렉트 URL은 환경 변수로 설정 가능하게 한다
