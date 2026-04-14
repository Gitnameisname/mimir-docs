# Phase 14 보안 취약점 검사 보고서

**작성일**: 2026-04-14
**검사 범위**: Phase 14 전체 (Task 14-1 ~ 14-16) — 인증, 인가, 세션, 계정 관리, Admin UI, 감사/알림/배치/API 키
**검사 방법**: 정적 분석 (AST + 패턴 grep), 통합 검증 스크립트 (41 항목), 수동 코드 리뷰, OWASP Top 10 / OWASP Authentication 체크리스트 대조

---

## 1. 검사 결과 요약

| 분류 | 건수 |
|------|------|
| 발견 취약점 (신규) | 1건 |
| Medium | 1건 |
| High | 0건 |
| Critical | 0건 |
| 수정 완료 | 1건 |
| 잔존 취약점 | 0건 |

Phase 14-16 통합 검증 스크립트: **41/41 PASS (0 FAIL, 3 SKIP)**.

---

## 2. OWASP Top 10 항목별 검사 결과

| OWASP 항목 | 결과 | 비고 |
|-----------|------|------|
| A01 — 접근 제어 실패 | 통과 | 모든 Admin API `Depends(require_admin_access)`, `_PERMISSION_MATRIX` 화이트리스트, 프런트 `AuthGuard(requiredRole=ORG_ADMIN)` |
| A02 — 암호화 실패 | 통과 | bcrypt (cost=12), SHA-256 (RT/API Key), AES 암호화 (GitLab access/refresh), TLS 기본 검증 활성 |
| A03 — 인젝션 | 통과 | 5개 Phase 14 모듈 전수 `%s` 바인딩 확인, React 자동 이스케이프, `dangerouslySetInnerHTML` 유일 사용처 화이트리스트 |
| A04 — 불안전한 설계 | **수정** | VULN-P14-01 (OAuth 이메일 자동 연결로 인한 계정 탈취) → 수정 완료 |
| A05 — 보안 설정 오류 | 통과 | RT 쿠키 `HttpOnly + SameSite=strict + Secure(prod)`, JWT `algorithms=[HS256]` 고정 |
| A06 — 취약 컴포넌트 | 통과 | Phase 14 신규 의존성 0 — `bcrypt`, `PyJWT`, `psycopg2`, `httpx`, `redis` 모두 기존 |
| A07 — 인증 실패 | 통과 | dummy_verify(타이밍 공격 방어), 통합 오류 메시지, rate-limit 429, RT rotation 탈취 감지 |
| A08 — 소프트웨어 무결성 | 통과 | 감사 로그 hash-chain (Phase 14-13), JWT 서명 |
| A09 — 로깅 실패 | 통과 | `audit_emitter.emit` 필수 경로 + `logger.exception` 이중 기록, password 미로깅 |
| A10 — SSRF | 통과 | 외부 요청은 OIDC Discovery + GitLab 토큰 엔드포인트만 (`settings.gitlab_base_url`), 사용자 입력 URL 사용 안 함 |

---

## 3. 발견 취약점 상세

### VULN-P14-01 ⚠️ Medium — OAuth 이메일 자동 연결로 인한 계정 탈취

| 항목 | 내용 |
|------|------|
| 파일 | `backend/app/api/auth/oauth_service.py` `_link_or_create_account` |
| OWASP | A04 (불안전한 설계) / A07 (인증 실패) |
| 심각도 | **Medium** — 피해자가 GitLab 계정이 있고 Mimir 에는 아직 가입하지 않은 경우 한정 |

**취약점 내용**

기존 로직은 OAuth 콜백에서 `oauth_accounts` 매핑이 없을 때 **이메일만으로** 기존 `users` 행을 찾아 자동 연결했다.

```python
# 취약 코드 (수정 전)
existing_user = users_repository.get_by_email(conn, email)
if existing_user:
    if existing_user.status != "ACTIVE":
        return None
    # 기존 사용자에 OAuth 계정 연결 — 이메일 인증 여부 확인 없음
    self._create_oauth_account(conn, user_id=existing_user.id, ...)
```

**공격 시나리오**

1. 공격자가 피해자의 이메일(예: `victim@company.com`)로 Mimir 자체 가입(`/auth/register`)을 수행한다. 이 단계에서 `users` 행이 `email_verified=False`, `auth_provider="local"` 로 생성된다. 공격자는 비밀번호를 본인이 아는 값으로 설정한다.
2. 공격자는 메일을 받지 못하므로 이메일 인증은 완료하지 못한다. 그러나 `users` 행은 `status=ACTIVE` 로 이미 존재한다.
3. 나중에 피해자가 GitLab 을 통해 Mimir 에 로그인한다. GitLab 은 `victim@company.com` 을 userinfo 로 반환한다.
4. `_link_or_create_account` 가 동일 이메일의 기존 사용자를 찾아 공격자가 만든 계정에 GitLab 연결을 수립한다.
5. 이후 공격자는 본인이 알고 있는 비밀번호로 `/auth/login` 성공 → 피해자 GitLab 신원과 같은 계정에 접근하여 권한 상승/데이터 조회가 가능해진다.

**수정 내용**

기존 사용자의 `email_verified=False` 인 경우 자동 연결을 거부하고 감사 이벤트 `user.oauth_link_refused` 를 남긴다. 사용자는 대신 기존 계정에 로그인 후 계정 관리 페이지에서 명시적 `link_to_user_id` 플로우로 연결하거나, 관리자에게 문의해야 한다.

```python
# 수정 후
if not existing_user.email_verified:
    logger.warning(
        "oauth_link_refused: existing user %s has unverified email %s — "
        "potential account takeover attempt",
        existing_user.id, email,
    )
    audit_emitter.emit(
        event_type="user.oauth_link_refused",
        action="auth.oauth.link",
        resource_type="user",
        resource_id=existing_user.id,
        result="failure",
        metadata={"provider": provider, "provider_uid": provider_uid,
                  "reason": "existing_email_unverified"},
    )
    return None
```

**영향 범위 제한**

- 이미 `email_verified=True` 인 계정은 정상 OAuth 로그인 플로우에서 자동 연결 유지 (기존 UX 보존).
- OAuth 로 가입된 계정은 `email_verified=True` 로 생성되므로 OAuth↔OAuth 흐름은 영향 없음.
- 로컬 가입 후 이메일 인증을 완료한 정상 사용자도 영향 없음.
- 공격자가 피해자 이메일로 선점 가입을 해둔 **악의적 시나리오** 만 차단된다.

---

## 4. Phase 14 보안 설계 — 다층 방어 요약

### 4-1. 인증 (Authentication)

| 층 | 방어 수단 |
|---|---------|
| 비밀번호 저장 | bcrypt (cost=`settings.bcrypt_cost_factor`, 기본 12) |
| 비밀번호 규칙 | 8–128자, 문자 카테고리 2종 이상 |
| 로그인 레이트 제한 | Valkey `login_attempts:{email}`, 임계 초과 429 |
| 계정 열거 방지 | user_not_found / invalid_password 동일 메시지 + `dummy_verify()` 로 타이밍 일치 |
| JWT 서명 | HS256, `algorithms=[HS256]` 명시, `require=["sub","exp"]` |
| Access Token | 메모리(모듈 변수) 저장만, localStorage 사용 금지 |
| Refresh Token | `secrets.token_urlsafe(64)` → SHA-256 저장, `hmac.compare_digest` 비교 |
| RT 쿠키 | `HttpOnly=True`, `SameSite=strict`, prod `Secure=True` |
| RT Rotation | 매 rotate 시 revoke + 새 hash, family_id 유지 |
| RT 탈취 감지 | 이미 revoked 된 RT 재사용 → `revoke_family` 호출 |
| OAuth CSRF | state Valkey 10분 TTL + 1회 소진 |
| OAuth Code 탈취 | PKCE S256 (`code_verifier` + `code_challenge`) |
| OAuth 계정 연결 | 이메일 인증 여부 확인 (VULN-P14-01 수정) |
| Purpose Token | `purpose` claim + `jti` + Valkey `used_token:{jti}` 로 1회성 |

### 4-2. 인가 (Authorization)

| 층 | 방어 수단 |
|---|---------|
| 매트릭스 | `_PERMISSION_MATRIX` — action → frozenset of roles 화이트리스트 |
| 역할 계층 | VIEWER < AUTHOR < REVIEWER < APPROVER < ORG_ADMIN < SUPER_ADMIN |
| 기본 정책 | 미등록 action 은 기본 거부 (RBAC-09) |
| Admin API | 전부 `Depends(require_admin_access)` |
| SUPER_ADMIN only | `admin.write`, 조직/역할/API 키/배치 쓰기 |
| ORG_ADMIN+ | `admin.read`, 모니터링/감사 조회 |
| Admin UI | `<AuthGuard requiredRole="ORG_ADMIN">` + `ForbiddenContent` |
| 프런트 게이팅 | `hasMinimumRole` 역할 계층 확인 |

### 4-3. 입력 검증

| 대상 | 검증 |
|------|------|
| email | Pydantic `EmailStr` + lowercase + strip |
| password | 8–128자 + 카테고리 2종 + `Field(max_length=128)` |
| display_name | 1–100자 + 공백만 금지 |
| API 키 이름 | `^[\w\- .]{1,100}$` UNICODE (reDoS 무해) |
| API 키 scope | 화이트리스트 4종 |
| API 키 만료일 | `Field(ge=0, le=3650)` |
| UUID 경로 | `uuid.UUID(x)` 형식 검증 → 422 |
| Cron 표현식 | `cron_util.validate` |

### 4-4. 세션/쿠키

| 설정 | 값 |
|------|---|
| RT Cookie | `HttpOnly=True, SameSite=strict, Secure=prod` |
| AT 저장 | 메모리(React 모듈 변수) — localStorage 금지 |
| URL fragment | OAuth AT 전달은 `#` 뒤 (서버 로그 비포함) |
| CSRF | SameSite=strict + JSON API + Bearer → 쿠키 기반 CSRF 표면 제한 |

### 4-5. 로깅 / 감사

| 항목 | 내용 |
|------|------|
| 비밀번호 | logger 에 전달 0건 (정적 검사 확인) |
| 감사 이벤트 | 로그인·로그아웃·회원가입·OAuth 연결/해제/거부·비밀번호 변경·RT revoke·API 키 발급/폐기 등 25종 이벤트 |
| 이중 기록 | `audit_emitter.emit` + `logger.exception` (실패 경보 수집) |
| hash-chain | Phase 14-13 에서 감사 레코드 변조 감지 |

### 4-6. 감사 로그 필터 / 관리 UI

| 방어 | 내용 |
|------|------|
| 필터 메타데이터 엔드포인트 | 정적 카탈로그 (DB DISTINCT 미사용) → DoS 방지 |
| 사용자 검색 드롭다운 | page_size=10 제한 |
| 감사 이벤트 유형 | 서버측 정적 리스트 + 클라 staleTime 5분 |

---

## 5. OWASP Authentication 체크리스트 (task16 통합 검증)

| # | 항목 | 결과 |
|---|------|------|
| 01 | 비밀번호 평문 저장 금지 | PASS |
| 02 | 비밀번호 로그 노출 금지 | PASS |
| 03 | 로그인 실패 통합 메시지 | PASS |
| 04 | 세션 고정 방지 (로그인 시 새 family) | PASS |
| 05 | 로그인 시도 제한 (429) | PASS |
| 06 | RT 재사용 감지 → family 전체 폐기 | PASS |
| 07 | RT 쿠키 SameSite=strict | PASS |
| 08 | JWT algorithms 명시 (HS256 고정) | PASS |
| 09 | purpose token 1회성 (jti + used_token) | PASS |
| 10 | OAuth state Valkey + 1회 소진 | PASS |
| 11 | 타이밍 공격 방어 (dummy_verify + compare_digest) | PASS |
| 12 | RT HttpOnly Cookie | PASS |

---

## 6. 잠재 위험과 운영 완화

| 위험 | 심각도 | 완화 |
|------|--------|------|
| bcrypt cost 가 낮게 설정되어 브루트포스 허용 | Low | `bcrypt_cost_factor` 12 이상 강제 (배포 체크리스트) |
| Valkey 장애 시 rate-limit fail-open | Low | 의도된 가용성 우선. P1 경보 연동 |
| 서명키 유출 | High | 비밀 관리 외부 범위 (Phase 15 rotation 절차 필요) |
| RT 쿠키를 공격자 네트워크에서 스니핑 | Medium | prod `Secure=True` + HSTS (Phase 13) |
| OAuth provider 유출로 공격자가 GitLab 에 가짜 계정 생성 | Low | `oauth_accounts` 매핑은 provider_uid 기준 unique, email 재활용 차단 (VULN-P14-01 수정) |
| 브라우저 `atob` base64url 미변환 | Medium | 검수 보고서 결함 3 수정 (silent refresh 안정화) |
| 회원가입 TOCTOU 중복 | Low | 검수 보고서 결함 4 수정 (UniqueViolation → 409) |
| 관리자 세션 장기 유지 | Low | AT 짧은 만료 + RT rotation |
| `dangerouslySetInnerHTML` 추가 사용 회귀 | Low | 통합 검증 XSS-01 가 PR 게이트 |
| SQL 바인딩 회귀 | Low | 통합 검증 SQLI-* 가 PR 게이트 |

---

## 7. 검증 결과

```
Phase 14-16 통합 보안 검증
======================================================================
[ADMIN-FE]  4/4
[COVERAGE] 13/13
[FLOW]      6/6
[OWASP]    12/12
[SQLI]      5/5
[XSS]       1/1
[SKIPPED]   3  (fastapi 미설치 샌드박스 — CI 에서 자동 수행)

합계: 41/41 PASS  (0 FAIL)
```

VULN-P14-01 수정 후에도 41/41 유지 (oauth_service 의 거부 경로 추가는 기존 동작을 깨지 않음).

---

## 8. 결론

Phase 14 는 인증·인가·세션·입력검증·Admin UI 접근제어·감사 전 스택에서 OWASP 주요 항목을 충족한다. 신규 취약점 1건(Medium, 계정 탈취) 을 발견·수정했으며, 잔존 취약점 0건. 수정 후 통합 검증 41/41 유지, 회귀 없음.

Phase 14 는 운영 투입 가능 상태이며, 배포 체크리스트는 §6 의 운영 완화 항목을 참조한다.
