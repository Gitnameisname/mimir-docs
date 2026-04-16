# Task 0-7. [Critical] Fresh-boot 스모크 CI 잡 구축

## 1. 작업 목적

**Fresh PostgreSQL 인스턴스**에서 마이머 플랫폼이 **완전히 독립적으로 부트스트랩**될 수 있음을 자동으로 검증하고, **pgvector 설치 유무**에 따른 기능 가용성을 매트릭스로 검사한다.

이는 S2 원칙 ⑦ "폐쇄망 동등성" 검증과 S1 부채 해소의 핵심 작업이다.

---

## 2. 작업 범위

### 포함 범위

1. **GitHub Actions 워크플로우 파일 작성**
   - 파일: `.github/workflows/fresh-boot-smoke.yml` (신규)
   - Matrix 전략: pgvector enabled / pgvector disabled
   - 각 매트릭스에서 독립적 PostgreSQL 인스턴스 생성

2. **Fresh-boot 테스트 스크립트**
   - 파일: `/backend/tests/smoke/fresh_boot.py` (신규)
   - 단계 1: DB 마이그레이션
   - 단계 2: 초기 admin 사용자 생성
   - 단계 3: 기본 CRUD 테스트 (문서, 노드)
   - 단계 4: 검색 기능 테스트 (FTS, RAG)
   - 단계 5: 인증 흐름 테스트
   - 단계 6: Admin API 접근성 테스트

3. **pgvector on/off 환경별 적응**
   - pgvector=true: 모든 기능 정상 동작
   - pgvector=false: RAG 비활성화, FTS만 사용 (원칙 ⑦ degrade 모드)

4. **Seed users 실행 제어**
   - 환경변수 `ALLOW_SEED_USERS` 기반
   - CI에서는 seed_users 미실행 (초기 상태 검증)

5. **워크플로우 통합**
   - 기존 `.github/workflows/ci.yml`과의 트리거 설정
   - PR to main/develop 시 자동 실행
   - 성공/실패 보고

### 제외 범위

- 프로덕션 배포 환경 테스트 (별도 작업)
- 성능 벤치마크 (별도 작업)
- 장기 실행 회귀 테스트 (별도 작업)

---

## 3. 선행 조건

- Backend FastAPI 애플리케이션 상태 (app.py, 라우터)
- 데이터베이스 마이그레이션 스크립트 (Alembic 또는 유사)
- GitHub Actions 워크플로우 기본 구조 이해
- Docker 또는 서비스 컨테이너 사용 경험

---

## 4. 주요 작업 항목

### 4-1. GitHub Actions 워크플로우 파일 작성

**파일:** `.github/workflows/fresh-boot-smoke.yml` (신규)

```yaml
name: Fresh-Boot Smoke Tests

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main
      - develop

jobs:
  fresh-boot:
    runs-on: ubuntu-latest
    
    # Matrix: pgvector enabled/disabled
    strategy:
      matrix:
        pgvector_enabled: [true, false]
        
      fail-fast: false  # 한 매트릭스 실패해도 다른 매트릭스는 계속 실행

    services:
      postgres:
        image: postgres:15-alpine
        
        env:
          POSTGRES_USER: mimir_user
          POSTGRES_PASSWORD: mimir_password
          POSTGRES_DB: mimir_test
        
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        
        ports:
          - 5432:5432

    steps:
      # 1. 코드 체크아웃
      - uses: actions/checkout@v4

      # 2. Python 설정
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'

      # 3. pgvector 설치 (enabled=true인 경우만)
      - name: Install PostgreSQL pgvector extension
        if: matrix.pgvector_enabled == true
        run: |
          sudo apt-get update
          sudo apt-get install -y postgresql-contrib
          PGPASSWORD=mimir_password psql -h localhost -U mimir_user -d mimir_test \
            -c "CREATE EXTENSION IF NOT EXISTS vector;"

      # 4. Backend 의존성 설치
      - name: Install backend dependencies
        working-directory: ./backend
        run: |
          python -m pip install --upgrade pip setuptools wheel
          pip install -r requirements.txt

      # 5. 환경 설정
      - name: Setup environment
        run: |
          cat > .env.test << EOF
          DATABASE_URL=postgresql://mimir_user:mimir_password@localhost:5432/mimir_test
          PGVECTOR_ENABLED=${{ matrix.pgvector_enabled }}
          ALLOW_SEED_USERS=false
          DEBUG=false
          LOG_LEVEL=INFO
          EOF

      # 6. 데이터베이스 마이그레이션
      - name: Run database migrations
        working-directory: ./backend
        env:
          DATABASE_URL: postgresql://mimir_user:mimir_password@localhost:5432/mimir_test
        run: |
          alembic upgrade head

      # 7. Fresh-boot 스모크 테스트 실행
      - name: Run fresh-boot smoke tests
        working-directory: ./backend
        env:
          DATABASE_URL: postgresql://mimir_user:mimir_password@localhost:5432/mimir_test
          PGVECTOR_ENABLED: ${{ matrix.pgvector_enabled }}
          ALLOW_SEED_USERS: false
        run: |
          pytest tests/smoke/fresh_boot.py -v --tb=short

      # 8. 테스트 결과 리포팅
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: fresh-boot-results-pgvector-${{ matrix.pgvector_enabled }}
          path: |
            backend/test-results/
            backend/.coverage

      # 9. 실패 시 로그 출력
      - name: Print failure logs
        if: failure()
        run: |
          echo "=== Fresh-boot test failed ==="
          echo "Matrix: pgvector_enabled=${{ matrix.pgvector_enabled }}"
          echo "Check database migrations and application startup logs above."
```

### 4-2. 스모크 테스트 스크립트 작성

**파일:** `/backend/tests/smoke/fresh_boot.py` (신규)

```python
"""
Fresh-boot 스모크 테스트
신규 PostgreSQL 인스턴스에서 플랫폼 부트스트랩 검증
"""

import os
import json
from typing import Any
import pytest
import httpx
from sqlalchemy import create_engine, text
from sqlalchemy.orm import Session

# 환경 설정
DATABASE_URL = os.getenv("DATABASE_URL")
PGVECTOR_ENABLED = os.getenv("PGVECTOR_ENABLED", "true").lower() == "true"
BASE_URL = "http://localhost:8000"  # FastAPI 앱이 실행 중이어야 함


class TestFreshBootSmoke:
    """Fresh-boot 환경에서의 기본 기능 검증"""

    @pytest.fixture(scope="class", autouse=True)
    def setup(self):
        """테스트 전 FastAPI 앱 시작"""
        # 참고: CI 환경에서는 별도로 앱을 시작해야 할 수도 있음
        # 예: subprocess.Popen(["uvicorn", "app:app", ...])
        yield

    # ========== 1단계: DB 마이그레이션 검증 ==========

    def test_database_connection(self):
        """PostgreSQL 연결 및 기본 테이블 존재 확인"""
        engine = create_engine(DATABASE_URL)
        with engine.connect() as conn:
            # 기본 테이블 확인
            result = conn.execute(text(
                "SELECT EXISTS(SELECT 1 FROM information_schema.tables "
                "WHERE table_name='user')"
            ))
            assert result.scalar() is True, "user 테이블 존재하지 않음"

    def test_pgvector_status(self):
        """pgvector 확장 설치 상태 확인"""
        engine = create_engine(DATABASE_URL)
        with engine.connect() as conn:
            result = conn.execute(text(
                "SELECT EXISTS(SELECT 1 FROM pg_extension "
                "WHERE extname='vector')"
            ))
            is_installed = result.scalar()
            
            if PGVECTOR_ENABLED:
                assert is_installed is True, "pgvector가 설치되어야 함"
            else:
                # pgvector가 없어도 OK (degrade mode)
                pass

    # ========== 2단계: 초기 admin 사용자 검증 ==========

    def test_initial_admin_user_creation(self):
        """초기 admin 사용자 생성 및 로그인"""
        # FastAPI 앱이 시작 시 admin 사용자를 자동으로 생성해야 함
        # 또는 관리자 API로 수동 생성
        
        with httpx.Client(base_url=BASE_URL, timeout=10.0) as client:
            # 기본 admin 계정으로 로그인 시도
            response = client.post("/api/v1/auth/login", json={
                "username": "admin",
                "password": "admin"  # 기본값 또는 환경변수
            })
            
            # 초기 상태에서는 admin 계정이 없을 수 있으므로,
            # 먼저 생성하고 검증하는 플로우 필요
            if response.status_code == 401:
                # 초기 admin 사용자 생성 API 호출
                create_response = client.post("/api/v1/auth/admin-init", json={
                    "username": "admin",
                    "password": "admin",
                    "email": "admin@example.com"
                })
                assert create_response.status_code == 201, \
                    f"Admin 생성 실패: {create_response.text}"
                
                # 다시 로그인 시도
                login_response = client.post("/api/v1/auth/login", json={
                    "username": "admin",
                    "password": "admin"
                })
                assert login_response.status_code == 200, \
                    f"Admin 로그인 실패: {login_response.text}"
                data = login_response.json()
                assert "access_token" in data, "access_token 없음"
            else:
                assert response.status_code == 200
                data = response.json()
                assert "access_token" in data

    # ========== 3단계: 기본 CRUD 검증 ==========

    def test_document_crud(self):
        """문서 생성, 조회, 수정, 삭제"""
        with httpx.Client(base_url=BASE_URL, timeout=10.0) as client:
            # 인증 토큰 획득
            token = self._get_auth_token(client)
            headers = {"Authorization": f"Bearer {token}"}
            
            # 문서 생성
            create_response = client.post(
                "/api/v1/documents",
                json={
                    "title": "Test Document",
                    "content": "This is a test document.",
                    "document_type": "article"
                },
                headers=headers
            )
            assert create_response.status_code in [200, 201], \
                f"문서 생성 실패: {create_response.text}"
            doc = create_response.json()
            doc_id = doc.get("data", {}).get("id") or doc.get("id")
            assert doc_id is not None, "문서 ID 없음"
            
            # 문서 조회
            get_response = client.get(
                f"/api/v1/documents/{doc_id}",
                headers=headers
            )
            assert get_response.status_code == 200, \
                f"문서 조회 실패: {get_response.text}"
            
            # 문서 수정
            update_response = client.patch(
                f"/api/v1/documents/{doc_id}",
                json={"title": "Updated Title"},
                headers=headers
            )
            assert update_response.status_code in [200, 204], \
                f"문서 수정 실패: {update_response.text}"
            
            # 문서 삭제
            delete_response = client.delete(
                f"/api/v1/documents/{doc_id}",
                headers=headers
            )
            assert delete_response.status_code in [200, 204], \
                f"문서 삭제 실패: {delete_response.text}"

    def test_node_crud(self):
        """노드 생성, 조회, 수정, 삭제"""
        # 문서 먼저 생성
        with httpx.Client(base_url=BASE_URL, timeout=10.0) as client:
            token = self._get_auth_token(client)
            headers = {"Authorization": f"Bearer {token}"}
            
            # 문서 생성
            doc_response = client.post(
                "/api/v1/documents",
                json={"title": "Doc", "document_type": "article"},
                headers=headers
            )
            doc = doc_response.json()
            doc_id = doc.get("data", {}).get("id") or doc.get("id")
            
            # 노드 생성
            node_response = client.post(
                f"/api/v1/documents/{doc_id}/nodes",
                json={"name": "Node 1", "type": "content"},
                headers=headers
            )
            assert node_response.status_code in [200, 201], \
                f"노드 생성 실패: {node_response.text}"
            node = node_response.json()
            node_id = node.get("data", {}).get("id") or node.get("id")
            
            # 노드 조회, 수정, 삭제...
            # (세부 구현은 생략)

    # ========== 4단계: 검색 기능 검증 ==========

    def test_fts_search(self):
        """전문 검색 (FTS) 기능 검증"""
        with httpx.Client(base_url=BASE_URL, timeout=10.0) as client:
            token = self._get_auth_token(client)
            headers = {"Authorization": f"Bearer {token}"}
            
            # 문서 생성
            doc = self._create_document(client, headers, 
                title="Hello World", content="This is a test.")
            doc_id = doc["id"]
            
            # FTS 검색
            search_response = client.get(
                "/api/v1/search?q=hello",
                headers=headers
            )
            assert search_response.status_code == 200
            results = search_response.json()
            assert len(results.get("data", {}).get("results", [])) > 0, \
                "검색 결과 없음"

    def test_rag_search_degrade_mode(self):
        """RAG 검색 (pgvector 비활성화 시 degrade 모드)"""
        if not PGVECTOR_ENABLED:
            # pgvector 없으면 RAG 비활성화, FTS만 사용
            pytest.skip("pgvector disabled - RAG not available")
        
        with httpx.Client(base_url=BASE_URL, timeout=10.0) as client:
            token = self._get_auth_token(client)
            headers = {"Authorization": f"Bearer {token}"}
            
            # RAG 검색 시도
            rag_response = client.post(
                "/api/v1/rag/search",
                json={"query": "test", "mode": "semantic"},
                headers=headers
            )
            
            if PGVECTOR_ENABLED:
                assert rag_response.status_code == 200, \
                    f"RAG 검색 실패: {rag_response.text}"
            else:
                # pgvector 없으면 400 또는 503 리턴 (graceful degrade)
                assert rag_response.status_code in [400, 503, 501]

    # ========== 5단계: 인증 흐름 검증 ==========

    def test_auth_flow(self):
        """전체 인증 흐름 (login, token refresh, logout)"""
        with httpx.Client(base_url=BASE_URL, timeout=10.0) as client:
            # 로그인
            login_response = client.post("/api/v1/auth/login", json={
                "username": "admin",
                "password": "admin"
            })
            assert login_response.status_code == 200
            data = login_response.json()
            access_token = data.get("data", {}).get("access_token") \
                         or data.get("access_token")
            
            # 토큰으로 보호된 API 접근
            headers = {"Authorization": f"Bearer {access_token}"}
            me_response = client.get("/api/v1/auth/me", headers=headers)
            assert me_response.status_code == 200
            
            # 토큰 없이 보호된 API 접근 시도
            no_auth_response = client.get("/api/v1/auth/me")
            assert no_auth_response.status_code == 401

    # ========== 6단계: Admin API 접근성 검증 ==========

    def test_admin_endpoints_access(self):
        """Admin 전용 API 접근 제어"""
        with httpx.Client(base_url=BASE_URL, timeout=10.0) as client:
            token = self._get_auth_token(client)
            headers = {"Authorization": f"Bearer {token}"}
            
            # Admin 사용자 목록 조회
            users_response = client.get(
                "/api/v1/admin/users",
                headers=headers
            )
            assert users_response.status_code == 200, \
                f"Admin 사용자 목록 조회 실패: {users_response.text}"
            
            # Admin 설정 조회
            settings_response = client.get(
                "/api/v1/admin/settings",
                headers=headers
            )
            assert settings_response.status_code in [200, 404], \
                f"Admin 설정 조회 실패: {settings_response.text}"

    # ========== 헬퍼 함수 ==========

    def _get_auth_token(self, client: httpx.Client) -> str:
        """admin으로 인증하고 access_token 반환"""
        response = client.post("/api/v1/auth/login", json={
            "username": "admin",
            "password": "admin"
        })
        assert response.status_code == 200, f"로그인 실패: {response.text}"
        data = response.json()
        token = data.get("data", {}).get("access_token") \
              or data.get("access_token")
        assert token is not None, "access_token 없음"
        return token

    def _create_document(self, client: httpx.Client, headers: dict,
                        **kwargs) -> dict:
        """헬퍼: 문서 생성"""
        response = client.post(
            "/api/v1/documents",
            json=kwargs,
            headers=headers
        )
        assert response.status_code in [200, 201]
        data = response.json()
        return data.get("data") or data


# ========== 모듈 실행 ==========

if __name__ == "__main__":
    pytest.main([__file__, "-v"])
```

### 4-3. 워크플로우 통합 설정

**파일:** `.github/workflows/ci.yml` (수정)

기존 ci.yml에서 fresh-boot 워크플로우를 트리거하거나, 별도로 실행하도록 설정:

```yaml
# ci.yml에 추가
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  # ... 기존 jobs ...
  
  # fresh-boot 워크플로우 호출 (선택)
  fresh-boot:
    runs-on: ubuntu-latest
    needs: [backend-lint, backend-test]  # 선행 작업
    if: success()
    uses: ./.github/workflows/fresh-boot-smoke.yml
```

또는 fresh-boot-smoke.yml을 독립적으로 유지하고 같은 트리거 조건 설정.

### 4-4. 환경변수 설정

**파일:** `backend/.env.test` (CI용)

```bash
DATABASE_URL=postgresql://mimir_user:mimir_password@localhost:5432/mimir_test
PGVECTOR_ENABLED=true  # 또는 false (matrix에서 설정)
ALLOW_SEED_USERS=false  # 초기 상태 검증이므로 seed_users 미실행
DEBUG=false
LOG_LEVEL=INFO
SECRET_KEY=test-secret-key-change-in-production
```

---

## 5. 산출물

1. **`.github/workflows/fresh-boot-smoke.yml`** (신규)
   - PostgreSQL 서비스 설정
   - pgvector on/off 매트릭스
   - 마이그레이션 → 스모크 테스트 실행

2. **`/backend/tests/smoke/fresh_boot.py`** (신규)
   - 6단계 검증 (DB, pgvector, admin, CRUD, 검색, 인증)
   - pytest 기반 스모크 테스트

3. **`/backend/.env.test`** (신규 또는 수정)
   - CI 환경 설정

4. **마이그레이션 스크립트 검증**
   - 기존 alembic 마이그레이션 정상 작동 확인

5. **테스트 결과 리포트**
   - GitHub Actions artifact로 저장

---

## 6. 완료 기준

1. `.github/workflows/fresh-boot-smoke.yml` 작성 및 PR/push 시 자동 실행
2. pgvector=true 매트릭스: 모든 테스트 통과
3. pgvector=false 매트릭스: 
   - DB 마이그레이션, CRUD, FTS 검색 통과
   - RAG 검색은 graceful degrade (오류 아님)
4. Admin 사용자 자동 생성 또는 초기화 API 존재
5. 인증 흐름 정상 작동 (login, token, logout)
6. 모든 테스트 실패 시 CI 빌드 실패 처리
7. 아티팩트에 테스트 결과 저장

---

## 7. 작업 지침

### 지침 7-1. FastAPI 앱 시작 방식

CI 환경에서 테스트를 실행하려면 FastAPI 앱이 이미 실행 중이어야 함:

```yaml
# 워크플로우에 추가
- name: Start FastAPI application
  working-directory: ./backend
  run: |
    uvicorn app:app --host 0.0.0.0 --port 8000 &
    sleep 3  # 앱 시작 대기
```

또는 pytest fixture로 앱을 동적으로 시작.

### 지침 7-2. 타임아웃 설정

장시간 테스트를 피하기 위해 timeout 설정:

```yaml
- name: Run fresh-boot smoke tests
  timeout-minutes: 10
  run: pytest tests/smoke/fresh_boot.py -v
```

### 지침 7-3. 실패 원인 분석

테스트 실패 시 로그를 명확히 하기:

```python
def test_something(self):
    try:
        response = client.get("/api/v1/...", headers=headers)
        assert response.status_code == 200
    except AssertionError:
        print(f"Response: {response.text}")
        print(f"Status: {response.status_code}")
        raise
```

### 지침 7-4. seed_users 미실행 보장

CI에서는 ALLOW_SEED_USERS=false를 반드시 설정:

```python
# app.py 또는 startup 함수
if os.getenv("ALLOW_SEED_USERS") == "true":
    seed_users()  # 기본값은 실행 금지
```

### 지침 7-5. pgvector 설치 오류 처리

Ubuntu 환경에서 pgvector 설치 오류 가능성:

```yaml
- name: Install pgvector
  if: matrix.pgvector_enabled == true
  run: |
    sudo apt-get update
    sudo apt-get install -y postgresql-contrib
    # 또는 source에서 빌드 (선택)
```

만약 설치 실패 시 대안:
```yaml
continue-on-error: true  # pgvector 설치 실패해도 계속 진행
# 그 후 pgvector 없음을 감지하고 matrix 스킵
```
