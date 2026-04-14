# Task 14-16. 통합 테스트 및 보안 검증

## 1. 작업 목적

Phase 14에서 구현한 인증 시스템과 관리 시스템 전체를 통합 테스트하고, 보안 취약점을 검증한다.

이 작업의 목표는 다음과 같다.

- 인증 플로우 E2E 테스트 (로컬 로그인, GitLab OAuth, 토큰 갱신)
- 권한 우회 테스트 (RBAC 전수 검증)
- OWASP Authentication 취약점 점검
- Admin UI 접근 제어 테스트
- 보안 취약점 보고서 작성

---

## 2. 작업 범위

### 포함 범위

- 인증 플로우 E2E 테스트 (Playwright)
- 백엔드 통합 테스트 (pytest, 실제 DB)
- RBAC 전수 검증 (모든 엔드포인트 × 모든 역할)
- OWASP Authentication 체크리스트 점검
- Refresh Token 보안 테스트 (Rotation, 탈취 시나리오)
- 입력값 검증 테스트 (SQL Injection, XSS)
- 보안 취약점 보고서 작성

### 제외 범위

- 성능/부하 테스트 (Phase 13에서 다룸)
- 침투 테스트 (별도 외부 감사)

---

## 3. 선행 조건

- Task 14-1~14-15 모든 구현 완료
- 테스트 환경 구축 (테스트 DB, Valkey)
- Phase 13 테스트 전략 이해

---

## 4. 주요 검증 대상

### 4-1. 인증 플로우 E2E 테스트

```
테스트 시나리오:

1. 회원가입 → 이메일 인증 → 로그인 → 로그아웃
   - 회원가입 폼 제출
   - 이메일 인증 토큰으로 인증 완료
   - 로그인 후 메인 페이지 접근
   - 로그아웃 후 보호 페이지 접근 불가

2. GitLab OAuth 로그인 (mock)
   - GitLab 로그인 버튼 클릭
   - Mock GitLab에서 인증 완료
   - 콜백 처리 → 메인 페이지 리다이렉트
   - 자동 계정 생성 확인

3. 토큰 갱신
   - Access Token 만료 시뮬레이션
   - 자동 silent refresh 확인
   - 갱신 후 API 호출 정상 동작

4. 비밀번호 재설정
   - "비밀번호 찾기" 플로우 전체
   - 재설정 토큰 1회 사용 확인
   - 변경 후 기존 세션 폐기 확인
```

---

### 4-2. RBAC 전수 검증

모든 API 엔드포인트에 대해 각 역할별 접근 테스트:

```python
# tests/test_authorization.py

ENDPOINTS = [
    ("GET",  "/api/v1/documents",           "document.read"),
    ("POST", "/api/v1/documents",           "document.create"),
    ("GET",  "/api/v1/admin/users",         "admin.read"),
    ("POST", "/api/v1/admin/users",         "admin.write"),
    ("POST", "/api/v1/auth/register",       None),  # 공개
    ("POST", "/api/v1/auth/login",          None),  # 공개
    ("GET",  "/api/v1/admin/settings",      "admin.read"),
    ("PATCH","/api/v1/admin/settings/auth/max_login_attempts", "admin.write"),
    ("GET",  "/api/v1/admin/alerts/rules",  "admin.read"),
    # ... Phase 14 신규 엔드포인트 포함
]

ROLES = ["VIEWER", "AUTHOR", "REVIEWER", "APPROVER", "ORG_ADMIN", "SUPER_ADMIN"]

@pytest.mark.parametrize("method,path,action", ENDPOINTS)
@pytest.mark.parametrize("role", ROLES)
async def test_endpoint_authorization(method, path, action, role):
    """각 역할이 허용/거부되는지 검증"""
    response = await call_api(method, path, as_role=role)
    if action is None:
        assert response.status_code != 401  # 공개 엔드포인트
    elif is_allowed(role, action):
        assert response.status_code != 403
    else:
        assert response.status_code == 403
```

---

### 4-3. OWASP Authentication 점검 체크리스트

| # | 점검 항목 | 검증 방법 | 예상 결과 |
|---|---------|---------|---------|
| 1 | 비밀번호 평문 저장 | DB 직접 확인 | bcrypt 해시만 존재 |
| 2 | 비밀번호 로그 노출 | 로그 파일 검색 | 비밀번호 문자열 0건 |
| 3 | 로그인 실패 메시지 | 잘못된 이메일/비밀번호 시도 | 동일 에러 메시지 |
| 4 | 세션 고정 공격 | 로그인 전후 세션 ID 비교 | 변경됨 |
| 5 | Brute Force | 6회 연속 실패 | 429 또는 잠금 |
| 6 | RT Rotation | 동일 RT 2회 사용 | family 전체 폐기 |
| 7 | CSRF | Cookie 기반 요청 위조 | SameSite 차단 |
| 8 | JWT 알고리즘 혼동 | alg:none 토큰 전송 | 거부 |
| 9 | 비밀번호 재설정 토큰 재사용 | 사용된 토큰 재시도 | 거부 |
| 10 | OAuth state 위조 | 잘못된 state로 콜백 | 거부 |
| 11 | 타이밍 공격 | 존재/미존재 이메일 응답 시간 비교 | 유사한 응답 시간 |
| 12 | HttpOnly Cookie | 브라우저 JS로 쿠키 접근 | 접근 불가 |

---

### 4-4. Refresh Token 보안 시나리오 테스트

```python
# tests/test_refresh_token_security.py

async def test_rotation_normal():
    """정상 Rotation: RT 사용 → 새 RT 발급 → 기존 RT 폐기"""

async def test_rotation_reuse_detection():
    """탈취 감지: 이미 사용된 RT 재사용 → family 전체 폐기"""

async def test_expired_refresh_token():
    """만료된 RT → 401 → 재로그인 필요"""

async def test_logout_revokes_family():
    """로그아웃 → family 전체 RT 폐기"""

async def test_password_change_revokes_all():
    """비밀번호 변경 → 모든 family 폐기 (현재 세션 제외)"""
```

---

### 4-5. 입력값 검증 테스트

```python
# SQL Injection
async def test_login_sql_injection():
    response = await login(email="' OR '1'='1", password="anything")
    assert response.status_code == 401  # 인증 실패, SQL 오류 아님

# XSS
async def test_register_xss():
    response = await register(display_name="<script>alert('xss')</script>")
    user = await get_user(...)
    assert "<script>" not in user.display_name  # 이스케이핑 또는 거부

# 대량 입력
async def test_password_max_length():
    response = await register(password="A" * 10000)
    assert response.status_code == 422  # 길이 제한
```

---

### 4-6. Admin 접근 제어 테스트

```python
async def test_viewer_cannot_access_admin():
    """VIEWER 역할로 /admin/* 접근 → 403"""

async def test_author_cannot_access_admin():
    """AUTHOR 역할로 /admin/* 접근 → 403"""

async def test_org_admin_can_access_admin():
    """ORG_ADMIN 역할로 /admin/* 접근 → 200"""

async def test_settings_change_requires_super_admin():
    """ORG_ADMIN으로 설정 변경 시도 → 403"""
    """SUPER_ADMIN으로 설정 변경 → 200"""
```

---

## 5. 산출물

1. 인증 플로우 E2E 테스트 (Playwright)
2. RBAC 전수 검증 테스트 (pytest parametrize)
3. Refresh Token 보안 테스트
4. 입력값 검증 테스트 (SQL Injection, XSS)
5. OWASP Authentication 점검 결과 보고서
6. Phase 14 보안 취약점 검사 보고서

---

## 6. 완료 기준

- 모든 인증 플로우 E2E 테스트가 통과한다
- 모든 엔드포인트 × 역할 RBAC 테스트가 통과한다
- OWASP Authentication 체크리스트 12항목이 모두 통과한다
- Refresh Token Rotation 시나리오 5종이 모두 통과한다
- SQL Injection, XSS 입력 테스트에서 취약점이 발견되지 않는다
- VIEWER/AUTHOR 역할로 Admin 영역 접근이 차단된다
- 보안 취약점 보고서가 작성되어 있다

---

## 7. Codex 작업 지침

- 테스트는 실제 DB(테스트용)를 사용하며 mock을 최소화한다
- GitLab OAuth 테스트는 mock 서버를 사용한다 (실제 GitLab 불필요)
- RBAC 테스트는 parametrize로 모든 조합을 자동 생성한다
- 보안 테스트 실패 시 즉시 수정하고 재검증한다
- 보고서에는 점검 항목, 결과, 발견 취약점, 수정 내역을 포함한다
- 테스트 커버리지를 측정하여 인증 관련 코드 80% 이상을 목표로 한다
