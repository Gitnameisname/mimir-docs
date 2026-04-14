# Phase 14. 로그인 및 관리 시스템 구축

## 1. Phase 목적

Mimir 플랫폼에 실제 사용자 인증 플로우(GitLab OAuth2/OIDC 연동 + 자체 JWT 인증)를 구축하고, 플랫폼 전체를 운영·관리할 수 있는 통합 관리 시스템을 완성한다.

Phase 0~13에서 구축한 RBAC 권한 체계, 세션/JWT 토큰 인프라, Admin API를 기반으로 실제 로그인·회원가입·계정 관리 UI 및 플로우를 구현하고, 관리자가 사용자·시스템·모니터링·알림·배치 작업을 하나의 인터페이스에서 통합 관리할 수 있도록 한다.

---

## 2. Phase 14 범위

### 포함 범위

- OAuth2/OIDC 로그인 연동 (GitLab — Self-managed 및 GitLab.com 지원)
- 이메일/비밀번호 기반 자체 인증 (회원가입, 로그인)
- JWT Access/Refresh Token 이중 토큰 체계 구축
- 비밀번호 재설정 및 이메일 인증 플로우
- 로그인/회원가입/계정 관리 UI (User UI)
- 통합 관리 대시보드 UI (Admin UI)
- 사용자 관리 UI (CRUD, 역할 할당, 상태 관리)
- 시스템 설정 관리 UI (DocumentType 설정, 시스템 파라미터)
- 모니터링 대시보드 UI (시스템 상태, 메트릭, 에러)
- 알림 관리 시스템 (알림 규칙, 알림 채널, 알림 이력)
- 배치 작업 관리 UI (작업 목록, 상태, 수동 실행)
- 감사 로그 조회 UI (필터링, 검색, 상세 조회)

### 제외 범위

- SAML/LDAP 엔터프라이즈 SSO (향후 확장)
- Google/GitHub 등 추가 OAuth 제공자 (향후 확장)
- ABAC(속성 기반 접근 제어) — 현재 RBAC 유지
- 외부 알림 채널 연동 구현 (Slack/PagerDuty 웹훅 설정만 포함, 실제 연동은 별도)
- 인프라 프로비저닝 자동화 (Terraform 등)

---

## 3. 선행 조건

- Phase 0~13 완료 — RBAC, 세션, JWT, API Key 인증 인프라, Admin API 완성
- `users`, `roles`, `organizations`, `user_org_roles`, `api_keys`, `audit_events` 테이블 존재
- AuthorizationService (RBAC 매트릭스) 동작 확인
- Phase 7 Admin UI 기반 컴포넌트 존재
- Phase 13 모니터링·로깅 체계 구축 완료

---

## 4. 핵심 기술 결정

### 4-1. GitLab OAuth2/OIDC 로그인 아키텍처

| 항목 | 결정 |
|------|------|
| 프로토콜 | OAuth 2.0 Authorization Code Flow + PKCE |
| 제공자 | GitLab (OIDC — Self-managed 및 GitLab.com 모두 지원) |
| Discovery | GitLab OIDC Discovery (`/.well-known/openid-configuration`) 자동 설정 |
| 토큰 교환 | 백엔드에서 Authorization Code → GitLab Token → 자체 JWT 발급 |
| Scopes | `openid`, `profile`, `email`, `read_user` |
| 계정 연결 | 이메일 기준 자동 연결 (동일 이메일 = 동일 계정) |
| 신규 가입 | GitLab 최초 로그인 시 자동 계정 생성 (VIEWER 역할) |
| GitLab URL | 환경변수 설정 (`GITLAB_BASE_URL`) — Self-managed 인스턴스 대응 |
| 라이브러리 | `authlib` (Python, FastAPI 호환, OIDC Discovery 지원) |

> OAuth 콜백은 반드시 백엔드에서 처리하며, 프론트엔드에 GitLab Token을 노출하지 않는다.
> Self-managed GitLab 인스턴스의 경우 `GITLAB_BASE_URL` 환경변수로 base URL을 지정한다.

---

### 4-2. JWT 이중 토큰 체계

| 항목 | Access Token | Refresh Token |
|------|-------------|---------------|
| 알고리즘 | HS256 (기존 유지) | HS256 |
| 유효기간 | 15분 | 7일 |
| 저장 위치 | 메모리 (프론트엔드) | HttpOnly Secure Cookie |
| 갱신 방식 | Refresh Token으로 재발급 | Rotation (사용 시 새 RT 발급) |
| 폐기 | 만료 대기 | Valkey 블랙리스트 (즉시 폐기) |
| Claims | sub, role, exp, iat, jti | sub, exp, iat, jti, family |

> Refresh Token Rotation을 적용하여 탈취 시 family 단위 전체 폐기로 보안을 강화한다.

---

### 4-3. 비밀번호 정책

| 항목 | 결정 |
|------|------|
| 해싱 알고리즘 | bcrypt (cost factor 12) |
| 최소 길이 | 8자 |
| 복잡도 | 영문 + 숫자 + 특수문자 중 2종 이상 |
| 재설정 | 이메일 인증 링크 (JWT 기반, 30분 유효) |
| 로그인 시도 제한 | 5회 실패 시 15분 잠금 (Valkey 기반) |
| 라이브러리 | `passlib[bcrypt]` |

---

### 4-4. 관리 대시보드 구조

| 영역 | 주요 기능 | 데이터 소스 |
|------|---------|-----------|
| 홈 대시보드 | 핵심 메트릭 카드, 시스템 상태, 최근 활동 | `/admin/dashboard/*` API |
| 사용자 관리 | CRUD, 역할 변경, 상태 관리, 조직 할당 | `/admin/users/*` API |
| 조직 관리 | 조직 CRUD, 구성원 관리 | `/admin/organizations/*` API |
| 역할/권한 관리 | 역할 CRUD, 권한 매트릭스 조회 | `/admin/roles/*` API |
| 시스템 설정 | DocumentType 설정, 시스템 파라미터 | `/admin/settings/*` API (신규) |
| 모니터링 | 시스템 상태, 에러 추이, 응답 시간 | `/admin/dashboard/*` API |
| 알림 관리 | 알림 규칙 CRUD, 채널 설정, 이력 조회 | `/admin/alerts/*` API (신규) |
| 배치 작업 | 작업 목록, 상태, 수동 실행, 스케줄 | `/admin/jobs/*` API (확장) |
| 감사 로그 | 이벤트 검색, 필터링, 상세 조회 | `/admin/audit-logs/*` API |
| API 키 관리 | 키 CRUD, 사용량 조회 | `/admin/api-keys/*` API |

---

### 4-5. 프론트엔드 인증 상태 관리

| 항목 | 결정 |
|------|------|
| 상태 관리 | React Context + useReducer |
| 토큰 저장 | Access Token → 메모리, Refresh Token → HttpOnly Cookie |
| 자동 갱신 | Access Token 만료 5분 전 자동 갱신 (silent refresh) |
| 라우트 보호 | AuthGuard HOC (미인증 → 로그인 페이지, 권한 부족 → 403 페이지) |
| 로딩 상태 | 인증 확인 중 스켈레톤 UI 표시 |

---

## 5. Task 목록

| Task | 이름 | 주요 내용 |
|------|------|---------|
| 14-1 | 인증 아키텍처 설계 | OAuth2/JWT 플로우 설계, 시퀀스 다이어그램, 보안 검토 |
| 14-2 | 비밀번호 인증 구현 | 회원가입, 이메일/비밀번호 로그인, bcrypt 해싱, 로그인 시도 제한 |
| 14-3 | JWT 이중 토큰 체계 구현 | Access/Refresh Token 발급·갱신·폐기, Rotation, 블랙리스트 |
| 14-4 | GitLab OAuth2/OIDC 로그인 구현 | GitLab 연동(Self-managed/GitLab.com), OIDC Discovery, Authorization Code + PKCE, 계정 자동 연결 |
| 14-5 | 비밀번호 재설정 및 이메일 인증 | 재설정 링크 발송, 토큰 검증, 이메일 인증 플로우 |
| 14-6 | 로그인/회원가입 UI 구현 | 로그인 페이지, 회원가입 폼, GitLab 로그인 버튼, 비밀번호 재설정 UI |
| 14-7 | 계정 관리 UI 구현 | 프로필 편집, 비밀번호 변경, GitLab 계정 연결/해제, 세션 관리 |
| 14-8 | 프론트엔드 인증 상태 관리 | AuthContext, AuthGuard, 자동 토큰 갱신, 라우트 보호 |
| 14-9 | 통합 관리 대시보드 UI 구현 | 홈 대시보드, 메트릭 카드, 시스템 상태, 최근 활동 |
| 14-10 | 사용자/조직/역할 관리 UI 구현 | 사용자 CRUD, 역할 할당, 조직 관리, 권한 매트릭스 뷰 |
| 14-11 | 시스템 설정 관리 API 및 UI | 시스템 파라미터 관리, DocumentType 설정, 설정 변경 감사 로그 |
| 14-12 | 모니터링 대시보드 UI 구현 | 실시간 시스템 상태, 에러 추이 차트, 응답 시간 그래프 |
| 14-13 | 알림 관리 시스템 구축 | 알림 규칙 CRUD, 알림 채널 설정, 알림 이력 관리, 웹훅 설정 |
| 14-14 | 배치 작업 관리 UI 구현 | 작업 목록/상태 조회, 수동 실행, 스케줄 관리 |
| 14-15 | 감사 로그 및 API 키 관리 UI | 감사 로그 검색/필터, API 키 CRUD, 사용량 조회 |
| 14-16 | 통합 테스트 및 보안 검증 | 인증 플로우 E2E 테스트, OWASP 검증, 권한 우회 테스트 |

---

## 6. 데이터 구조

### 6-1. users 테이블 확장

```sql
ALTER TABLE users
  ADD COLUMN password_hash     VARCHAR(255),        -- bcrypt 해시 (소셜 전용 계정은 NULL)
  ADD COLUMN auth_provider     VARCHAR(50) DEFAULT 'local',  -- local, gitlab
  ADD COLUMN email_verified    BOOLEAN DEFAULT FALSE,
  ADD COLUMN email_verified_at TIMESTAMPTZ,
  ADD COLUMN failed_login_count INTEGER DEFAULT 0,
  ADD COLUMN locked_until      TIMESTAMPTZ,
  ADD COLUMN avatar_url        VARCHAR(500);
```

### 6-2. oauth_accounts 테이블 (신규)

```sql
CREATE TABLE oauth_accounts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  provider        VARCHAR(50)  NOT NULL,  -- gitlab (향후 다른 제공자 확장 가능)
  provider_uid    VARCHAR(255) NOT NULL,  -- 제공자 측 고유 ID
  provider_email  VARCHAR(255),
  provider_name   VARCHAR(255),
  avatar_url      VARCHAR(500),
  access_token    TEXT,                    -- 암호화 저장 (AES-256-GCM)
  refresh_token   TEXT,                    -- 암호화 저장
  token_expires_at TIMESTAMPTZ,
  raw_profile     JSONB DEFAULT '{}',
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now(),
  UNIQUE (provider, provider_uid)
);

CREATE INDEX idx_oauth_accounts_user ON oauth_accounts(user_id);
CREATE INDEX idx_oauth_accounts_provider ON oauth_accounts(provider, provider_uid);
```

### 6-3. refresh_tokens 테이블 (신규)

```sql
CREATE TABLE refresh_tokens (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash  VARCHAR(255) NOT NULL UNIQUE,  -- SHA-256 해시
  family_id   UUID NOT NULL,                 -- Rotation 그룹 식별
  expires_at  TIMESTAMPTZ NOT NULL,
  revoked     BOOLEAN DEFAULT FALSE,
  revoked_at  TIMESTAMPTZ,
  created_at  TIMESTAMPTZ DEFAULT now(),
  ip_address  VARCHAR(50),
  user_agent  TEXT
);

CREATE INDEX idx_refresh_tokens_user   ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_family ON refresh_tokens(family_id);
CREATE INDEX idx_refresh_tokens_hash   ON refresh_tokens(token_hash);
```

### 6-4. alert_rules 테이블 (신규)

```sql
CREATE TABLE alert_rules (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name         VARCHAR(255) NOT NULL,
  description  TEXT,
  metric_name  VARCHAR(255) NOT NULL,        -- 모니터링 대상 메트릭
  condition    JSONB NOT NULL,               -- { "operator": "gt", "threshold": 100, "duration_seconds": 300 }
  severity     VARCHAR(50) NOT NULL,         -- critical, warning, info
  channels     JSONB DEFAULT '[]',           -- ["email", "webhook"]
  enabled      BOOLEAN DEFAULT TRUE,
  created_by   UUID REFERENCES users(id),
  created_at   TIMESTAMPTZ DEFAULT now(),
  updated_at   TIMESTAMPTZ DEFAULT now()
);
```

### 6-5. alert_history 테이블 (신규)

```sql
CREATE TABLE alert_history (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  rule_id       UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
  triggered_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at   TIMESTAMPTZ,
  status        VARCHAR(50) NOT NULL,          -- firing, resolved, acknowledged
  metric_value  NUMERIC,
  message       TEXT,
  notified_channels JSONB DEFAULT '[]'
);

CREATE INDEX idx_alert_history_rule   ON alert_history(rule_id);
CREATE INDEX idx_alert_history_status ON alert_history(status, triggered_at DESC);
```

### 6-6. system_settings 테이블 (신규)

```sql
CREATE TABLE system_settings (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  category    VARCHAR(100) NOT NULL,           -- auth, notification, system, security
  key         VARCHAR(255) NOT NULL,
  value       JSONB NOT NULL,
  description TEXT,
  updated_by  UUID REFERENCES users(id),
  updated_at  TIMESTAMPTZ DEFAULT now(),
  UNIQUE (category, key)
);

CREATE INDEX idx_system_settings_category ON system_settings(category);
```

---

## 7. API 구조 개요

### 7-1. 인증 API (신규)

```
POST   /api/v1/auth/register              이메일/비밀번호 회원가입
POST   /api/v1/auth/login                 이메일/비밀번호 로그인
POST   /api/v1/auth/logout                로그아웃 (Refresh Token 폐기)
POST   /api/v1/auth/refresh               Access Token 갱신
POST   /api/v1/auth/forgot-password       비밀번호 재설정 요청
POST   /api/v1/auth/reset-password        비밀번호 재설정 실행
POST   /api/v1/auth/verify-email          이메일 인증 확인

GET    /api/v1/auth/oauth/{provider}      OAuth 인증 시작 (리다이렉트)
GET    /api/v1/auth/oauth/{provider}/callback   OAuth 콜백 처리
```

### 7-2. 계정 관리 API (신규)

```
GET    /api/v1/account/profile            내 프로필 조회
PATCH  /api/v1/account/profile            프로필 수정
POST   /api/v1/account/change-password    비밀번호 변경
GET    /api/v1/account/oauth-accounts     연결된 소셜 계정 목록
POST   /api/v1/account/oauth-accounts/{provider}/link     소셜 계정 연결
DELETE /api/v1/account/oauth-accounts/{provider}/unlink   소셜 계정 해제
GET    /api/v1/account/sessions           활성 세션 목록
DELETE /api/v1/account/sessions/{session_id}   세션 강제 종료
```

### 7-3. 시스템 설정 API (신규)

```
GET    /api/v1/admin/settings                    전체 설정 조회
GET    /api/v1/admin/settings/{category}         카테고리별 설정 조회
PATCH  /api/v1/admin/settings/{category}/{key}   설정 값 변경
```

### 7-4. 알림 관리 API (신규)

```
GET    /api/v1/admin/alerts/rules                알림 규칙 목록
POST   /api/v1/admin/alerts/rules                알림 규칙 생성
PATCH  /api/v1/admin/alerts/rules/{rule_id}      알림 규칙 수정
DELETE /api/v1/admin/alerts/rules/{rule_id}      알림 규칙 삭제
GET    /api/v1/admin/alerts/history               알림 이력 조회
PATCH  /api/v1/admin/alerts/history/{id}/acknowledge   알림 확인 처리
```

### 7-5. 배치 작업 API (기존 확장)

```
GET    /api/v1/admin/jobs                        작업 목록 (기존)
GET    /api/v1/admin/jobs/{job_id}               작업 상세 (신규)
POST   /api/v1/admin/jobs/{job_id}/run           수동 실행 (신규)
PATCH  /api/v1/admin/jobs/{job_id}/schedule      스케줄 변경 (신규)
POST   /api/v1/admin/jobs/{job_id}/cancel        작업 취소 (신규)
```

---

## 8. 기존 Phase 코드와의 통합

| 기존 구성 요소 | Phase 14 통합 방식 |
|--------------|-------------------|
| `tokens.py` (Access Token only) | Refresh Token 추가, 기존 Access Token 호환 유지 |
| `session.py` (Valkey 세션) | Refresh Token 블랙리스트 저장소로 확장 |
| `dependencies.py` (Actor 추출) | GitLab OAuth 인증 Actor 타입 추가, 기존 체인 유지 |
| `authorization.py` (RBAC) | 새 액션 추가: `account.*`, `alert.*`, `settings.*` |
| `admin.py` (Admin API) | 신규 엔드포인트 추가, 기존 엔드포인트 유지 |
| `users_repository.py` | 비밀번호/GitLab OAuth 필드 처리 로직 확장 |
| Phase 7 Admin UI | 관리 대시보드 레이아웃 및 네비게이션 확장 |
| Phase 13 모니터링 | 알림 관리 시스템이 모니터링 메트릭을 소비 |

---

## 9. 보안 고려사항

| 위협 | 대응 |
|------|------|
| Refresh Token 탈취 | Token Rotation + family 단위 폐기 |
| CSRF | SameSite=Strict Cookie + CSRF 토큰 |
| XSS를 통한 토큰 탈취 | Access Token 메모리 저장, Refresh Token HttpOnly Cookie |
| Brute Force 로그인 | 5회 실패 시 15분 잠금 (Valkey 기반 rate limiting) |
| OAuth State 위조 | PKCE + state 파라미터 검증 |
| 비밀번호 유출 | bcrypt(cost 12), 평문 로깅 금지 |
| 세션 고정 공격 | 로그인 성공 시 세션 ID 재생성 |
| GitLab Token 노출 | GitLab OAuth 토큰은 AES-256-GCM 암호화 저장, 프론트엔드 미노출 |

---

## 10. 완료 기준

- 이메일/비밀번호로 회원가입·로그인·로그아웃이 동작한다
- GitLab OAuth2/OIDC 로그인으로 인증이 완료된다 (Self-managed 및 GitLab.com)
- 동일 이메일의 로컬/GitLab 계정이 자동 연결된다
- Access Token 만료 시 Refresh Token으로 자동 갱신된다
- Refresh Token Rotation이 동작하며 탈취 시 family 전체가 폐기된다
- 비밀번호 재설정 이메일이 발송되고 재설정이 완료된다
- 관리자가 통합 대시보드에서 시스템 상태를 한눈에 파악할 수 있다
- 사용자/조직/역할 CRUD가 Admin UI에서 동작한다
- 시스템 설정을 Admin UI에서 변경하고 감사 로그에 기록된다
- 알림 규칙을 설정하고 조건 충족 시 알림이 발생한다
- 배치 작업 상태를 조회하고 수동 실행할 수 있다
- 감사 로그를 기간·유형·사용자별로 검색할 수 있다
- 모든 인증 플로우에 대한 E2E 테스트가 통과한다
- OWASP 인증 관련 취약점(Broken Authentication)이 없다
