# Task 0-11. [Low] 회귀 게이트 스크립트 FG 단위 복제 준비

## 1. 작업 목적

S1 `test_integration_security_phase14_16.py`를 **참조 기반**으로 하여, **FG(Feature Group) 단위 회귀 테스트** 디렉토리 구조와 pytest 템플릿을 설계한다.

목표:
- 각 FG 완료 시 해당 FG의 회귀 테스트 자동 실행
- Phase 진행 시 **이전 Phase의 모든 FG 회귀 테스트 누적 실행**
- CI/CD 파이프라인에 통합 (`.github/workflows/ci.yml`)
- **S2 절대 규칙** "개발 후에는 검수 수행 및 검수 결과 보고서 작성"의 자동화 기반 제공

현재 상태:
- S1의 회귀 테스트는 Phase 14/16 수준의 통합 스크립트만 존재
- FG 단위 테스트 구조 부재
- 각 FG 완료 후 회귀 검증 프로세스 불명확

---

## 2. 작업 범위

### 포함 범위

1. FG 단위 회귀 테스트 디렉토리 구조 설계
   - `/backend/tests/regression/phase{N}/fg{N.M}/` 구조 정의
   - conftest.py 통합 fixture 관리
   - 공유 테스트 유틸리티 (helpers, fixtures)

2. pytest 기반 템플릿 제공
   - 기본 테스트 함수 구조
   - 데이터베이스 초기화 (fresh-boot)
   - API 호출 및 응답 검증
   - 예시 테스트 케이스

3. 누적 실행 구조 설계
   - Phase 3 시작 시: Phase 0, 1, 2의 모든 FG 회귀 테스트 실행
   - Phase N 시작 시: Phase 0 ~ N-1의 모든 FG 회귀 테스트 실행
   - pytest 마커(marker)를 활용한 선택적 실행

4. CI 통합 지침
   - `.github/workflows/ci.yml` 또는 별도 `regression.yml` workflow 설계
   - FG별 회귀 테스트 실행 지점 정의
   - 실패 시 알림 전략

5. conftest.py 공유 fixture
   - 데이터베이스 세션
   - API 클라이언트
   - 테스트 사용자 생성
   - 감시 로그 검증 헬퍼

### 제외 범위

- 실제 FG별 테스트 케이스 작성 (향후 각 FG 작업에서)
- 성능 테스트 (별도 benchmark.py)
- 부하 테스트 (별도 load-test.py)
- E2E 테스트 (별도 selenium/playwright 스크립트)

---

## 3. 선행 조건

- S1 `test_integration_security_phase14_16.py` 분석 완료
- `/backend/tests/unit/` 및 `/backend/tests/integration/` 기존 구조 파악
- pytest, pytest-cov, httpx(또는 requests) 설치 확인
- PostgreSQL fresh-boot 환경 테스트 완료 (Task 0-6)
- GitHub Actions 워크플로우 기본 이해

---

## 4. 주요 작업 항목

### 4-1. FG 단위 회귀 테스트 디렉토리 구조 설계

**구조:**

```
/backend/tests/regression/
├── __init__.py
├── conftest.py                    # 전체 공유 fixture
├── phase0/
│   ├── __init__.py
│   ├── conftest.py                # Phase 0 공유 fixture
│   ├── fg0_1/
│   │   ├── __init__.py
│   │   ├── test_principles.py     # FG0.1: 원칙 및 규약 확정
│   │   └── conftest.py            # FG0.1 로컬 fixture
│   ├── fg0_2/
│   │   ├── __init__.py
│   │   ├── test_navigation.py     # FG0.2: 네비게이션
│   │   ├── test_seed_users.py     # FG0.2: seed_users 가드
│   │   ├── test_capabilities.py   # FG0.2: capabilities endpoint
│   │   ├── test_unwrap_envelope.py # FG0.2: API 응답 정규화
│   │   ├── test_fresh_boot.py     # FG0.2: fresh-boot 검증
│   │   └── conftest.py            # FG0.2 로컬 fixture
│   └── pytest.ini                 # Phase 0 pytest 설정
├── phase1/
│   ├── fg1_1/
│   ├── fg1_2/
│   └── pytest.ini
├── phase2/
│   ...
└── common/
    ├── __init__.py
    ├── fixtures.py                # 공유 fixture 정의
    ├── helpers.py                 # 테스트 유틸리티
    └── constants.py               # 상수 (테스트 사용자, API_BASE_URL 등)
```

**각 레벨의 역할:**

| 레벨 | 파일 | 역할 |
|------|------|------|
| 전체 | `/regression/conftest.py` | 모든 Phase 공유 fixture (DB 연결, API 클라이언트) |
| Phase | `/regression/phase{N}/conftest.py` | Phase별 공유 fixture (Phase 시작 전 초기화) |
| FG | `/regression/phase{N}/fg{N.M}/conftest.py` | FG별 로컬 fixture (테스트 데이터 생성) |
| 공통 | `/regression/common/` | 재사용 가능한 헬퍼, 상수 |

### 4-2. pytest 기본 템플릿 제공

**파일:** `/backend/tests/regression/common/fixtures.py` (신규)

```python
import pytest
import os
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from httpx import AsyncClient, Client

# 전역 상수
TEST_DATABASE_URL = os.getenv(
    'TEST_DATABASE_URL',
    'postgresql://postgres:postgres@localhost/mimir_test'
)
API_BASE_URL = 'http://localhost:8000'
ADMIN_USER = {'username': 'admin', 'password': 'admin123', 'email': 'admin@test.com'}
TEST_USER = {'username': 'testuser', 'password': 'testpass123', 'email': 'test@test.com'}

@pytest.fixture(scope='session')
def db_engine():
    """테스트 DB 엔진 (session scope)"""
    engine = create_engine(TEST_DATABASE_URL)
    yield engine
    # 테스트 완료 후 정리 (선택)
    # engine.dispose()

@pytest.fixture(scope='function')
def db_session(db_engine) -> Session:
    """테스트용 DB 세션 (function scope - 각 테스트마다 새로 생성)"""
    SessionLocal = sessionmaker(bind=db_engine, autoflush=False)
    session = SessionLocal()
    
    # 테스트 시작 전 트랜잭션 시작
    yield session
    
    # 테스트 완료 후 롤백 (데이터 정리)
    session.rollback()
    session.close()

@pytest.fixture(scope='function')
def api_client():
    """동기 API 클라이언트"""
    with Client(base_url=API_BASE_URL) as client:
        yield client

@pytest.fixture(scope='function')
async def async_api_client():
    """비동기 API 클라이언트"""
    async with AsyncClient(base_url=API_BASE_URL) as client:
        yield client

@pytest.fixture(scope='function')
def admin_auth_headers(api_client: Client):
    """Admin 사용자 인증 토큰"""
    response = api_client.post(
        '/api/v1/auth/login',
        json={'username': ADMIN_USER['username'], 'password': ADMIN_USER['password']}
    )
    token = response.json()['data']['access_token']
    return {'Authorization': f'Bearer {token}'}

@pytest.fixture(scope='function')
def user_auth_headers(api_client: Client):
    """일반 사용자 인증 토큰"""
    response = api_client.post(
        '/api/v1/auth/login',
        json={'username': TEST_USER['username'], 'password': TEST_USER['password']}
    )
    token = response.json()['data']['access_token']
    return {'Authorization': f'Bearer {token}'}
```

**파일:** `/backend/tests/regression/common/helpers.py` (신규)

```python
from typing import Any, Dict, Optional

def unwrap_envelope(response: Dict[str, Any]) -> Any:
    """API 응답 envelope 언래핑"""
    if response.get('success'):
        return response.get('data')
    else:
        error = response.get('error', {})
        raise AssertionError(f"API error: {error.get('code')} - {error.get('message')}")

def assert_api_success(response: Dict[str, Any], expected_status: int = 200):
    """API 응답 성공 확인"""
    assert response.get('success') is True, f"API failed: {response.get('error')}"

def assert_api_error(response: Dict[str, Any], error_code: str, expected_status: int = 400):
    """API 응답 실패 확인"""
    assert response.get('success') is False
    assert response.get('error', {}).get('code') == error_code

def assert_audit_log(db_session, action: str, **filter_kwargs):
    """감시 로그 기록 확인"""
    from app.models.audit_log import AuditLog
    log = db_session.query(AuditLog).filter(
        AuditLog.action == action,
        **filter_kwargs
    ).first()
    assert log is not None, f"Audit log not found: action={action}"
    return log
```

**파일:** `/backend/tests/regression/conftest.py` (신규)

```python
import pytest
import os
from sqlalchemy import create_engine, event
from sqlalchemy.orm import sessionmaker

# 전체 레벨 fixture 재사용
from tests.regression.common.fixtures import db_engine, db_session, api_client, admin_auth_headers, user_auth_headers

# pytest 설정
def pytest_configure(config):
    """pytest 시작 전 초기화"""
    # 테스트 환경 설정
    os.environ['ENVIRONMENT'] = 'test'
    os.environ['ALLOW_SEED_USERS'] = 'true'
    
    print("\n" + "="*60)
    print("Regression Tests Starting")
    print("="*60)

def pytest_unconfigure(config):
    """pytest 종료 후 정리"""
    print("\n" + "="*60)
    print("Regression Tests Completed")
    print("="*60)

@pytest.fixture(scope='session', autouse=True)
def fresh_boot_setup():
    """전체 테스트 스위트 시작 전 fresh-boot 환경 초기화"""
    print("\n[Fresh-boot] Initializing test database...")
    
    # DB 초기화 스크립트 실행 (또는 alembic migration)
    import subprocess
    result = subprocess.run(
        ['python', '-m', 'alembic', 'upgrade', 'head'],
        cwd='/backend',
        capture_output=True,
        text=True
    )
    
    if result.returncode != 0:
        print(f"Migration failed: {result.stderr}")
        raise RuntimeError("Database initialization failed")
    
    print("[Fresh-boot] Database initialized successfully")
    yield
```

### 4-3. FG 단위 테스트 템플릿

**파일:** `/backend/tests/regression/phase0/fg0_2/test_navigation.py` (예시)

```python
"""
FG0.2 회귀 테스트: 사용자 메뉴 및 네비게이션 구축

시나리오:
1. 사용자 로그인 → 메뉴 표시 → 로그아웃
2. Admin 사용자 → 관리자 설정 접근 → User UI 복귀
"""

import pytest
from httpx import Client
from sqlalchemy.orm import Session


class TestUserNavigation:
    """일반 사용자 네비게이션 회귀 테스트"""
    
    def test_user_logout_flow(self, api_client: Client, user_auth_headers: dict):
        """사용자 로그아웃 흐름 정상 작동"""
        # 1. 인증된 상태로 User API 호출
        response = api_client.get(
            '/api/v1/auth/me',
            headers=user_auth_headers
        )
        assert response.status_code == 200
        assert response.json()['success'] is True
        user_data = response.json()['data']
        assert user_data['username'] == 'testuser'
        
        # 2. 로그아웃 엔드포인트 호출
        response = api_client.post(
            '/api/v1/auth/logout',
            headers=user_auth_headers
        )
        assert response.status_code == 200
        
        # 3. 로그아웃 후 다시 API 호출 시 401 반환 확인
        response = api_client.get(
            '/api/v1/auth/me',
            headers=user_auth_headers
        )
        assert response.status_code == 401
    
    def test_user_menu_shows_profile_settings_logout(self, api_client: Client, user_auth_headers: dict):
        """사용자 메뉴에 프로필, 설정, 로그아웃 항목 표시"""
        # UI 메뉴 아이템은 프론트엔드에서 확인하지만,
        # 백엔드는 각 엔드포인트 존재 확인
        
        # 프로필 조회 가능
        response = api_client.get('/api/v1/auth/me', headers=user_auth_headers)
        assert response.status_code == 200
        
        # 설정 조회 가능
        response = api_client.get('/api/v1/user/settings', headers=user_auth_headers)
        assert response.status_code in [200, 404]  # 페이지 미구현 시 404 허용


class TestAdminNavigation:
    """관리자 네비게이션 회귀 테스트"""
    
    def test_admin_can_access_admin_settings(self, api_client: Client, admin_auth_headers: dict):
        """Admin 사용자만 관리자 설정 접근 가능"""
        # Admin 역할 확인
        response = api_client.get(
            '/api/v1/auth/me',
            headers=admin_auth_headers
        )
        assert response.status_code == 200
        user_data = response.json()['data']
        assert user_data['role'] == 'admin'
        
        # Admin API 접근 가능
        response = api_client.get(
            '/api/v1/admin/users',
            headers=admin_auth_headers
        )
        assert response.status_code in [200, 404]  # 엔드포인트 존재 시 확인
    
    def test_regular_user_cannot_access_admin_api(self, api_client: Client, user_auth_headers: dict):
        """일반 사용자는 관리자 API 접근 불가"""
        response = api_client.get(
            '/api/v1/admin/users',
            headers=user_auth_headers
        )
        assert response.status_code == 403  # Forbidden
```

**파일:** `/backend/tests/regression/phase0/fg0_2/test_seed_users.py` (예시)

```python
"""FG0.2 회귀 테스트: seed_users.py 프로덕션 가드"""

import os
import subprocess
import pytest


class TestSeedUsersGuard:
    """seed_users.py 실행 가드 회귀 테스트"""
    
    def test_seed_users_requires_env_var(self):
        """ALLOW_SEED_USERS 환경변수 없으면 실패"""
        env = os.environ.copy()
        env.pop('ALLOW_SEED_USERS', None)  # 제거
        
        result = subprocess.run(
            ['python', '-m', 'app.scripts.seed_users'],
            cwd='/backend',
            env=env,
            capture_output=True,
            text=True
        )
        
        assert result.returncode != 0
        assert 'ERROR' in result.stderr or 'ERROR' in result.stdout
    
    def test_seed_users_succeeds_with_env_var(self):
        """ALLOW_SEED_USERS=true면 성공"""
        env = os.environ.copy()
        env['ALLOW_SEED_USERS'] = 'true'
        env['ENVIRONMENT'] = 'test'
        
        result = subprocess.run(
            ['python', '-m', 'app.scripts.seed_users'],
            cwd='/backend',
            env=env,
            capture_output=True,
            text=True
        )
        
        # 성공 여부 확인 (exit code 0)
        # 또는 stdout에 성공 메시지 포함 확인
        assert result.returncode == 0 or 'success' in result.stdout.lower()
```

### 4-4. pytest.ini 및 마커 설정

**파일:** `/backend/tests/regression/pytest.ini` (신규)

```ini
[pytest]
# 테스트 경로
testpaths = regression

# 마커 정의
markers =
    phase0: Phase 0 회귀 테스트
    phase1: Phase 1 회귀 테스트
    phase2: Phase 2 회귀 테스트
    phase3: Phase 3 회귀 테스트
    fg0_1: FG0.1 (원칙 및 규약)
    fg0_2: FG0.2 (기술 부채 청산)
    fg1_1: FG1.1 (LLM 통합)
    navigation: 네비게이션 관련
    security: 보안 관련
    critical: Critical priority
    high: High priority
    mid: Mid priority

# 추가 옵션
addopts = 
    -v
    --tb=short
    --cov=app
    --cov-report=html
    --cov-report=term-missing
```

**파일:** `/backend/tests/regression/phase0/conftest.py` (신규)

```python
import pytest

# Phase 0 마커 자동 적용
def pytest_configure(config):
    # Phase 0 테스트에만 phase0 마커 자동 적용 (선택)
    pass

# Phase 0 시작 전 초기화
@pytest.fixture(scope='session', autouse=True)
def phase0_setup():
    """Phase 0 회귀 테스트 시작 전 초기화"""
    print("\n[Phase 0] Running regression tests...")
    yield
    print("[Phase 0] Regression tests completed")
```

### 4-5. CI/CD 통합 지침

**파일:** `.github/workflows/regression.yml` (신규, 또는 ci.yml에 추가)

```yaml
name: Regression Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  regression:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        python-version: ['3.10', '3.11']
        pgvector: [true, false]
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: mimir_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Install pgvector (if enabled)
      if: ${{ matrix.pgvector == true }}
      run: |
        # PostgreSQL 클라이언트를 통해 pgvector 설치
        sudo apt-get update
        sudo apt-get install -y postgresql-client
        PGPASSWORD=postgres psql -h localhost -U postgres -d mimir_test -c "CREATE EXTENSION IF NOT EXISTS vector;"
    
    - name: Install backend dependencies
      run: |
        cd backend
        pip install -r requirements.txt
        pip install pytest pytest-cov pytest-asyncio
    
    - name: Run migrations
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost/mimir_test
        ENVIRONMENT: test
      run: |
        cd backend
        python -m alembic upgrade head
    
    - name: Run Phase 0 regression tests
      env:
        TEST_DATABASE_URL: postgresql://postgres:postgres@localhost/mimir_test
        ENVIRONMENT: test
        ALLOW_SEED_USERS: 'true'
      run: |
        cd backend
        pytest tests/regression/phase0/ -v --tb=short -m "phase0"
    
    - name: Upload coverage reports
      if: always()
      uses: codecov/codecov-action@v3
      with:
        files: ./backend/coverage.xml
        fail_ci_if_error: false
```

**CI 실행 전략:**

```
Phase 0 완료 시:
  - Phase 0 FG0.1 회귀 테스트 실행
  - Phase 0 FG0.2 회귀 테스트 실행

Phase 1 시작 시:
  - Phase 0 모든 FG 회귀 테스트 실행 (기본선 유지)
  - Phase 1 FG1.1 회귀 테스트 실행
  - Phase 1 FG1.2 회귀 테스트 실행

Phase 3 시작 시:
  - Phase 0 모든 FG 회귀 테스트 실행
  - Phase 1 모든 FG 회귀 테스트 실행
  - Phase 2 모든 FG 회귀 테스트 실행
  - Phase 3 FG3.1 회귀 테스트 실행
  - ... (계속 누적)
```

### 4-6. 실행 명령어 문서

**파일:** `/backend/tests/regression/README.md` (신규)

```markdown
# 회귀 테스트 실행 가이드

## 전체 실행

```bash
cd backend
pytest tests/regression/ -v
```

## Phase 0만 실행

```bash
pytest tests/regression/phase0/ -v
```

## FG0.2만 실행

```bash
pytest tests/regression/phase0/fg0_2/ -v
```

## 특정 테스트만 실행

```bash
pytest tests/regression/phase0/fg0_2/test_navigation.py::TestUserNavigation::test_user_logout_flow -v
```

## 마커별 실행

```bash
# navigation 관련 테스트만
pytest tests/regression/ -m "navigation" -v

# Phase 0 테스트만
pytest tests/regression/ -m "phase0" -v

# Critical priority만
pytest tests/regression/ -m "critical" -v
```

## 커버리지 리포트

```bash
pytest tests/regression/ --cov=app --cov-report=html
# 생성된 htmlcov/index.html 열기
```

## 병렬 실행 (pytest-xdist)

```bash
pip install pytest-xdist
pytest tests/regression/ -n auto -v
```
```

---

## 5. 산출물

1. **디렉토리 구조**
   - `/backend/tests/regression/` 전체 구조

2. **conftest.py**
   - `/backend/tests/regression/conftest.py` (전역)
   - `/backend/tests/regression/phase0/conftest.py` (Phase 0)
   - `/backend/tests/regression/phase0/fg0_2/conftest.py` (FG별, 예시)

3. **공유 유틸리티**
   - `/backend/tests/regression/common/fixtures.py` (DB, API 클라이언트)
   - `/backend/tests/regression/common/helpers.py` (assertion, audit log)
   - `/backend/tests/regression/common/constants.py` (상수)

4. **테스트 템플릿**
   - `/backend/tests/regression/phase0/fg0_2/test_navigation.py` (예시)
   - `/backend/tests/regression/phase0/fg0_2/test_seed_users.py` (예시)

5. **pytest 설정**
   - `/backend/tests/regression/pytest.ini` (마커, 옵션)

6. **CI/CD 워크플로우**
   - `.github/workflows/regression.yml` (또는 ci.yml에 통합)

7. **실행 문서**
   - `/backend/tests/regression/README.md` (명령어, 가이드)

8. **FG별 복제 가이드**
   - `/backend/tests/regression/TEMPLATE.md` (Phase 1~9 적용 방법)

---

## 6. 완료 기준

1. FG 단위 디렉토리 구조 설계 완료
2. 공유 fixture (DB, API 클라이언트, 인증) 구현 완료
3. 테스트 헬퍼 함수 구현 완료 (API 응답 검증, 감시 로그 확인)
4. Phase 0 FG0.2 회귀 테스트 구현 완료 (navigation, seed_users 테스트)
5. pytest.ini 마커 정의 완료
6. CI/CD 워크플로우 설계 완료 (`.github/workflows/regression.yml`)
7. 모든 회귀 테스트 성공 (Phase 0 FG0.1, FG0.2)
8. Phase 1~9 적용 가능한 복제 템플릿 작성
9. 실행 문서 작성 완료 (README.md)

---

## 7. 작업 지침

### 지침 7-1. 테스트 데이터 격리

각 테스트 함수는 독립적이어야 함:

```python
@pytest.fixture
def test_document(db_session):
    """테스트용 문서 생성"""
    doc = Document(title="Test Doc", content="Test content")
    db_session.add(doc)
    db_session.commit()
    yield doc
    # 테스트 후 자동 롤백 (db_session fixture에서 처리)
```

### 지침 7-2. 마커 활용

테스트 함수에 마커 추가:

```python
@pytest.mark.phase0
@pytest.mark.fg0_2
@pytest.mark.critical
def test_user_logout_flow(...):
    ...
```

### 지침 7-3. S1 참조 통합

S1 `test_integration_security_phase14_16.py`의 패턴 유지:

```python
# S1 스타일: fresh-boot 후 기본 CRUD 검증
class TestFreshBootRegression:
    def test_fresh_boot_crud_flow(self):
        # 1. Create
        # 2. Read
        # 3. Update
        # 4. Delete
        pass
```

### 지침 7-4. Phase 진행 시 누적 실행

Phase 3 시작 전:

```bash
# Phase 0, 1, 2 모든 FG 회귀 테스트 실행
pytest tests/regression/phase0/ -v
pytest tests/regression/phase1/ -v
pytest tests/regression/phase2/ -v

# 이후 Phase 3 신규 테스트 실행
pytest tests/regression/phase3/ -v
```

### 지침 7-5. 감시 로그 검증

회귀 테스트에서 감시 로그 확인:

```python
def test_seed_users_audit_log(db_session):
    # seed_users 실행
    # ...
    
    # 감시 로그 확인
    from tests.regression.common.helpers import assert_audit_log
    log = assert_audit_log(db_session, action='admin_seed_users')
    assert log.details['created_users_count'] == 2
```

---

## 8. 참고 (S1 참조)

### S1 test_integration_security_phase14_16.py 패턴

S1에서의 통합 테스트 스타일:

```python
class TestPhase14SecurityIntegration:
    def test_auth_flow_with_fresh_boot(self):
        # 1. Fresh-boot DB 초기화
        # 2. Admin 사용자 생성 (seed_users)
        # 3. 로그인 검증
        # 4. API 접근 제어 검증
        # 5. 감시 로그 기록 검증
```

이 패턴을 **FG별로 세분화**하여 관리:

- Phase 0 FG0.2 수준의 테스트는 이 Template 따름
- Phase 1 FG1.1 (LLM 통합)은 LLM 호출 mock 추가
- Phase 2 FG2.1 (RAG)은 벡터 DB 테스트 추가
- ...

### 향후 확장

- **성능 회귀:** pytest-benchmark 추가
- **E2E 회귀:** Selenium/Playwright 통합
- **부하 회귀:** Locust 통합
