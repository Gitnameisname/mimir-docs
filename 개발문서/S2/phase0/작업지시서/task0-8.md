# Task 0-8. [Critical] 시스템 정보 엔드포인트 3-tier 신설

## 1. 작업 목적

클라이언트(User UI, Admin UI, 외부 AI 에이전트)가 **런타임에 현재 플랫폼의 기능 가용성**을 조회할 수 있도록 하는 **3-tier 엔드포인트**를 구축한다.

> **보안 근거**: provider 상세, pgvector 활성 여부 등 내부 구성 정보는 공격자의 공격 표면 분석에 악용될 수 있으므로, 정보 노출 수준을 인증 계층별로 분리한다.

3-tier 구조:
- **Tier 1 — Public Health** (`/api/v1/system/health`): 인증 불필요. 서비스 상태와 버전만 노출
- **Tier 2 — Capabilities** (`/api/v1/system/capabilities`): 인증 필요 (authenticated user). RAG·chunking 가용 여부 노출
- **Tier 3 — Admin Capabilities** (`/api/v1/admin/system/capabilities`): Admin 권한 필요. provider 상세, pgvector 상태, 내부 스위치 노출

이는:
- 정보 노출 최소화 원칙(Least Information Disclosure) 준수
- 자체 호스팅 환경에서 pgvector 설치 유무 감지 (Admin 전용)
- RAG 검색 가능 여부 판단 (인증 사용자)
- Chunking 활성 여부 확인 (인증 사용자)
- Phase 1에서 LLM provider 목록 동적 확장 지원 (Admin 전용)

S2 원칙 ⑦ "폐쇄망 동등성"의 검증 기반이 된다.

---

## 2. 작업 범위

### 포함 범위

1. **FastAPI 라우터 신설 (3-tier)**
   - 파일: `/backend/app/api/v1/system.py` (신규)
   - Tier 1: `GET /api/v1/system/health` — 인증 불필요 (public)
   - Tier 2: `GET /api/v1/system/capabilities` — 인증 필요 (authenticated user)
   - Tier 3: `GET /api/v1/admin/system/capabilities` — Admin 권한 필요
   - 대안 (Tier 1): `GET /.well-known/mimir-health` (선택)

2. **응답 스키마 정의 (tier별 분리)**

   **Tier 1 — Health 응답:**
   ```json
   {
     "success": true,
     "data": {
       "status": "ok",
       "version": "2.0.0"
     }
   }
   ```

   **Tier 2 — Capabilities 응답:**
   ```json
   {
     "success": true,
     "data": {
       "version": "2.0.0",
       "rag_available": true,
       "chunking_enabled": true,
       "mcp_spec_version": null
     }
   }
   ```

   **Tier 3 — Admin Capabilities 응답:**
   ```json
   {
     "success": true,
     "data": {
       "version": "2.0.0",
       "pgvector_enabled": true,
       "rag_available": true,
       "chunking_enabled": true,
       "supported_providers": ["openai"],
       "mcp_spec_version": null,
       "deployment_type": "standalone",
       "closed_network": false
     }
   }
   ```

3. **환경변수 기반 동적 응답**
   - `PGVECTOR_ENABLED` 환경변수 확인
   - `RAG_SERVICE_URL` 설정 여부 → rag_available
   - `CHUNKING_ENABLED` 환경변수 확인

4. **응답 캐싱**
   - 5분 캐시 (변경 가능성 낮음)
   - Cache-Control 헤더 설정

5. **인증 계층 적용**
   - Tier 1 (Health): 인증 불필요 — CORS 허용
   - Tier 2 (Capabilities): `get_current_user` 의존성 주입 — 로그인 필수
   - Tier 3 (Admin Capabilities): `get_current_admin_user` 의존성 주입 — Admin 권한 필수

6. **단위 테스트**
   - Tier별 인증 요구 검증 (비인증 접근 시 401/403)
   - pgvector on/off 환경별 응답 검증
   - 캐시 동작 검증

### 제외 범위

- 프론트엔드 UI 수정 (사용은 Task 0-5에서)
- 실시간 모니터링 (별도 엔드포인트)
- 기능 설정 변경 (읽기 전용)

---

## 3. 선행 조건

- FastAPI 앱 구조 이해 (라우터, 의존성)
- 환경변수 설정 방식 이해
- Task 0-7 Fresh-boot CI 완료 (테스트 환경)

---

## 4. 주요 작업 항목

### 4-1. 응답 스키마 정의 (3-tier 분리)

**파일:** `/backend/app/api/v1/schemas/system.py` (신규)

```python
"""
시스템 정보 및 기능 가용성 스키마 — 3-tier 분리
Tier 1 (Public Health): HealthData — 인증 불필요
Tier 2 (Capabilities): CapabilitiesData — 인증 필요
Tier 3 (Admin Capabilities): AdminCapabilitiesData — Admin 권한 필요
"""

from pydantic import BaseModel, Field
from typing import Optional, List
from enum import Enum


class ServiceStatus(str, Enum):
    """서비스 상태"""
    OK = "ok"
    DEGRADED = "degraded"
    ERROR = "error"


# ========== Tier 1: Public Health ==========

class HealthData(BaseModel):
    """Tier 1 — 공개 서비스 상태 (인증 불필요)
    
    보안 원칙: 비인증 사용자에게는 서비스 가용성 정보만 제공.
    내부 구성(provider, pgvector 등)은 절대 노출하지 않음.
    """
    status: ServiceStatus = Field(
        ...,
        description="서비스 상태 (ok, degraded, error)"
    )
    version: str = Field(
        ...,
        description="Mimir 플랫폼 버전 (예: 2.0.0)"
    )


class HealthResponse(BaseModel):
    """health 엔드포인트 응답"""
    success: bool = Field(True, description="요청 성공 여부")
    data: HealthData = Field(..., description="서비스 상태 정보")


# ========== Tier 2: Capabilities (인증 사용자) ==========

class CapabilitiesData(BaseModel):
    """Tier 2 — 기능 가용성 (인증 사용자)
    
    보안 원칙: 인증된 사용자에게 기능 활성 여부만 제공.
    provider 상세, 인프라 구성은 Admin tier로 분리.
    """
    version: str = Field(
        ...,
        description="Mimir 플랫폼 버전 (예: 2.0.0)"
    )
    rag_available: bool = Field(
        ...,
        description="RAG(검색 보강 생성) 서비스 가용 여부"
    )
    chunking_enabled: bool = Field(
        ...,
        description="문서 청킹 활성화 여부"
    )
    mcp_spec_version: Optional[str] = Field(
        default=None,
        description="지원하는 MCP 스펙 버전 (예: '2025-11-25')"
    )


class CapabilitiesResponse(BaseModel):
    """capabilities 엔드포인트 응답"""
    success: bool = Field(True, description="요청 성공 여부")
    data: CapabilitiesData = Field(..., description="기능 정보")


# ========== Tier 3: Admin Capabilities (Admin 전용) ==========

class AdminCapabilitiesData(BaseModel):
    """Tier 3 — 전체 시스템 정보 (Admin 전용)
    
    보안 근거: supported_providers, pgvector_enabled, deployment_type 등
    내부 구성 정보는 공격 표면 분석에 악용될 수 있으므로 Admin만 접근 가능.
    """
    version: str = Field(
        ...,
        description="Mimir 플랫폼 버전 (예: 2.0.0)"
    )
    pgvector_enabled: bool = Field(
        ...,
        description="PostgreSQL pgvector 확장 설치 여부"
    )
    rag_available: bool = Field(
        ...,
        description="RAG(검색 보강 생성) 서비스 가용 여부"
    )
    chunking_enabled: bool = Field(
        ...,
        description="문서 청킹 활성화 여부"
    )
    supported_providers: List[str] = Field(
        default_factory=list,
        description="활성화된 LLM provider 목록 (예: ['openai', 'anthropic'])"
    )
    mcp_spec_version: Optional[str] = Field(
        default=None,
        description="지원하는 MCP 스펙 버전 (예: '2025-11-25')"
    )
    deployment_type: str = Field(
        default="standalone",
        description="배포 타입 (standalone, cloud, on-premise)"
    )
    closed_network: bool = Field(
        default=False,
        description="폐쇄망 환경 여부 (외부 네트워크 불가)"
    )


class AdminCapabilitiesResponse(BaseModel):
    """admin capabilities 엔드포인트 응답"""
    success: bool = Field(True, description="요청 성공 여부")
    data: AdminCapabilitiesData = Field(..., description="전체 시스템 정보")
```

### 4-2. 라우터 구현 (3-tier)

**파일:** `/backend/app/api/v1/system.py` (신규)

```python
"""
시스템 정보 및 기능 가용성 엔드포인트 — 3-tier 구조

Tier 1: GET /api/v1/system/health          — 인증 불필요
Tier 2: GET /api/v1/system/capabilities    — 인증 필요 (authenticated user)
Tier 3: GET /api/v1/admin/system/capabilities — Admin 권한 필요
"""

import os
import logging
from datetime import datetime, timedelta
from fastapi import APIRouter, Depends, Response
from typing import Optional

from app.api.v1.schemas.system import (
    ServiceStatus,
    HealthData,
    HealthResponse,
    CapabilitiesData,
    CapabilitiesResponse,
    AdminCapabilitiesData,
    AdminCapabilitiesResponse,
)
from app.core.auth import get_current_user, get_current_admin_user
from app.models.user import User

logger = logging.getLogger(__name__)

# ========== 라우터 정의 ==========

# Tier 1, 2 — 일반 system 라우터
router = APIRouter(prefix="/system", tags=["System"])

# Tier 3 — Admin system 라우터
admin_router = APIRouter(prefix="/admin/system", tags=["Admin System"])


# ========== 캐시 설정 ==========

CACHE_TTL = timedelta(minutes=5)

_health_cache: dict = {"data": None, "timestamp": None}
_capabilities_cache: dict = {"data": None, "timestamp": None}
_admin_capabilities_cache: dict = {"data": None, "timestamp": None}


def _is_cache_valid(cache: dict) -> bool:
    """캐시 유효성 확인 (5분)"""
    if cache["timestamp"] is None:
        return False
    return datetime.utcnow() - cache["timestamp"] < CACHE_TTL


# ========== 내부 상태 조회 함수 ==========

def _get_pgvector_status() -> bool:
    """pgvector 활성화 여부 확인"""
    pgvector_env = os.getenv("PGVECTOR_ENABLED", "false").lower()
    if pgvector_env in ["true", "1", "yes"]:
        return True
    return False


def _get_rag_status() -> bool:
    """RAG 서비스 가용 여부"""
    if not _get_pgvector_status():
        return False
    rag_enabled = os.getenv("RAG_ENABLED", "false").lower() in ["true", "1"]
    return rag_enabled


def _get_chunking_status() -> bool:
    """문서 청킹 활성화 여부"""
    return os.getenv("CHUNKING_ENABLED", "true").lower() in ["true", "1"]


def _get_supported_providers() -> list:
    """활성화된 LLM provider 목록 (Admin 전용 노출)"""
    providers = []
    if os.getenv("OPENAI_API_KEY"):
        providers.append("openai")
    if os.getenv("ANTHROPIC_API_KEY"):
        providers.append("anthropic")
    if os.getenv("VLLM_ENDPOINT"):
        providers.append("vllm")
    if os.getenv("OLLAMA_ENDPOINT"):
        providers.append("ollama")
    return providers


def _get_mcp_spec_version() -> Optional[str]:
    """지원하는 MCP 스펙 버전 (Phase 4에서 활성화)"""
    mcp_supported = os.getenv("MCP_ENABLED", "false").lower() in ["true", "1"]
    if mcp_supported:
        return "2025-11-25"
    return None


def _get_service_status() -> ServiceStatus:
    """전체 서비스 상태 결정"""
    try:
        # 기본 DB 연결 확인 등 (확장 가능)
        # 현재는 단순 OK 반환
        return ServiceStatus.OK
    except Exception as e:
        logger.warning(f"Service health check failed: {e}")
        return ServiceStatus.DEGRADED


# ========== Tier 1: Public Health (인증 불필요) ==========

@router.get(
    "/health",
    response_model=HealthResponse,
    summary="서비스 상태 조회 (Public)",
    description="서비스 가용성 및 버전 정보를 조회합니다. 인증 불필요.",
)
async def get_health(response: Response) -> HealthResponse:
    """
    Tier 1 — Public Health 엔드포인트
    
    비인증 사용자에게 노출하는 최소 정보:
    - status: 서비스 상태 (ok / degraded / error)
    - version: 애플리케이션 버전
    
    보안: 내부 구성 정보(provider, pgvector 등) 절대 노출하지 않음.
    """
    if _is_cache_valid(_health_cache) and _health_cache["data"] is not None:
        health_data = _health_cache["data"]
    else:
        health_data = HealthData(
            status=_get_service_status(),
            version=os.getenv("APP_VERSION", "2.0.0"),
        )
        _health_cache["data"] = health_data
        _health_cache["timestamp"] = datetime.utcnow()

    response.headers["Cache-Control"] = "public, max-age=300"
    return HealthResponse(success=True, data=health_data)


# ========== Tier 2: Capabilities (인증 필요) ==========

@router.get(
    "/capabilities",
    response_model=CapabilitiesResponse,
    summary="기능 가용성 조회 (인증 필요)",
    description="현재 플랫폼의 기능 가용성을 조회합니다. 로그인 필요.",
)
async def get_capabilities(
    response: Response,
    current_user: User = Depends(get_current_user),
) -> CapabilitiesResponse:
    """
    Tier 2 — Capabilities 엔드포인트 (인증 사용자)
    
    노출 정보:
    - rag_available: RAG 서비스 가용 여부
    - chunking_enabled: 문서 청킹 활성화 여부
    - mcp_spec_version: 지원 MCP 스펙 버전
    
    비노출 (Admin tier로 분리):
    - pgvector_enabled, supported_providers, deployment_type, closed_network
    """
    if _is_cache_valid(_capabilities_cache) and \
       _capabilities_cache["data"] is not None:
        capabilities = _capabilities_cache["data"]
    else:
        capabilities = CapabilitiesData(
            version=os.getenv("APP_VERSION", "2.0.0"),
            rag_available=_get_rag_status(),
            chunking_enabled=_get_chunking_status(),
            mcp_spec_version=_get_mcp_spec_version(),
        )
        _capabilities_cache["data"] = capabilities
        _capabilities_cache["timestamp"] = datetime.utcnow()

    response.headers["Cache-Control"] = "private, max-age=300"
    return CapabilitiesResponse(success=True, data=capabilities)


# ========== Tier 3: Admin Capabilities (Admin 전용) ==========

@admin_router.get(
    "/capabilities",
    response_model=AdminCapabilitiesResponse,
    summary="전체 시스템 정보 조회 (Admin 전용)",
    description="내부 구성 정보 포함. Admin 권한 필요.",
)
async def get_admin_capabilities(
    response: Response,
    current_user: User = Depends(get_current_admin_user),
) -> AdminCapabilitiesResponse:
    """
    Tier 3 — Admin Capabilities 엔드포인트
    
    Admin만 접근 가능한 전체 시스템 정보:
    - pgvector_enabled, supported_providers, deployment_type, closed_network 등
    
    보안 근거: 이 정보는 공격자에게 내부 구성과 공격 표면을 
    정리해 주는 효과가 있으므로 Admin으로 제한.
    """
    if _is_cache_valid(_admin_capabilities_cache) and \
       _admin_capabilities_cache["data"] is not None:
        admin_capabilities = _admin_capabilities_cache["data"]
    else:
        admin_capabilities = AdminCapabilitiesData(
            version=os.getenv("APP_VERSION", "2.0.0"),
            pgvector_enabled=_get_pgvector_status(),
            rag_available=_get_rag_status(),
            chunking_enabled=_get_chunking_status(),
            supported_providers=_get_supported_providers(),
            mcp_spec_version=_get_mcp_spec_version(),
            deployment_type=os.getenv("DEPLOYMENT_TYPE", "standalone"),
            closed_network=os.getenv("CLOSED_NETWORK", "false").lower()
                          in ["true", "1"],
        )
        _admin_capabilities_cache["data"] = admin_capabilities
        _admin_capabilities_cache["timestamp"] = datetime.utcnow()

    response.headers["Cache-Control"] = "private, max-age=300"
    return AdminCapabilitiesResponse(success=True, data=admin_capabilities)


# ========== .well-known 별칭 (Tier 1 전용, 선택) ==========

@router.get(
    "/.well-known/mimir-health",
    response_model=HealthResponse,
    summary="Well-known health endpoint",
    description="RFC 5785 .well-known 규약 준수. 인증 불필요.",
    include_in_schema=False,
)
async def get_health_well_known(response: Response) -> HealthResponse:
    """Tier 1 .well-known 별칭"""
    return await get_health(response)
```

### 4-3. 라우터 등록

**파일:** `/backend/app/api/v1/__init__.py` (수정)

```python
"""
FastAPI v1 라우터 집계
"""

from fastapi import APIRouter
from app.api.v1 import (
    auth,
    documents,
    nodes,
    search,
    users,
    # ... 기존 라우터들 ...
    system,  # 신규 추가
)

api_router = APIRouter(prefix="/api/v1")

# 기존 라우터 등록
api_router.include_router(auth.router)
api_router.include_router(documents.router)
api_router.include_router(nodes.router)
api_router.include_router(search.router)
api_router.include_router(users.router)
# ...

# 신규: system 라우터 (Tier 1 health + Tier 2 capabilities)
api_router.include_router(system.router)

# 신규: admin system 라우터 (Tier 3 admin capabilities)
api_router.include_router(system.admin_router)
```

main app.py에서:

```python
# /backend/app/main.py

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.api.v1 import api_router

app = FastAPI(
    title="Mimir",
    version="2.0.0",
    description="Knowledge Management Platform for AI",
)

# CORS 설정
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # 또는 명시적 화이트리스트
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# API 라우터 (Tier 1/2/3 모두 포함)
app.include_router(api_router)
```

### 4-4. 환경변수 설정

**파일:** `.env.example` (수정)

```bash
# 시스템 설정
APP_VERSION=2.0.0
DEPLOYMENT_TYPE=standalone  # standalone, cloud, on-premise
CLOSED_NETWORK=false  # 폐쇄망 환경 여부

# pgvector 및 RAG 설정
PGVECTOR_ENABLED=false  # PostgreSQL pgvector 확장
RAG_ENABLED=false  # RAG 서비스 활성화 (pgvector_enabled=true 필요)
CHUNKING_ENABLED=true  # 문서 청킹 활성화

# LLM Provider (Phase 1 이후)
OPENAI_API_KEY=  # 지정 시 openai provider 활성화
# ANTHROPIC_API_KEY=  # 향후 추가
```

### 4-5. 단위 테스트 (3-tier 분리)

**파일:** `/backend/tests/unit/test_system_endpoints.py` (신규)

```python
"""
시스템 엔드포인트 3-tier 테스트
- Tier 1: /api/v1/system/health (인증 불필요)
- Tier 2: /api/v1/system/capabilities (인증 필요)
- Tier 3: /api/v1/admin/system/capabilities (Admin 필요)
"""

import pytest
from fastapi.testclient import TestClient
from unittest.mock import patch
import os

from app.main import app
from tests.conftest import (
    get_auth_headers,
    get_admin_auth_headers,
)


client = TestClient(app)


# ========== Tier 1: Public Health ==========

class TestHealthEndpoint:
    """Tier 1 — /api/v1/system/health 테스트"""

    def test_accessible_without_auth(self):
        """인증 없이 접근 가능"""
        response = client.get("/api/v1/system/health")
        assert response.status_code == 200

    def test_response_schema(self):
        """응답에 status, version만 포함 (내부 정보 미노출)"""
        response = client.get("/api/v1/system/health")
        data = response.json()
        
        assert data["success"] is True
        assert "status" in data["data"]
        assert "version" in data["data"]
        
        # 내부 구성 정보가 노출되지 않는지 확인
        assert "pgvector_enabled" not in data["data"]
        assert "supported_providers" not in data["data"]
        assert "rag_available" not in data["data"]
        assert "deployment_type" not in data["data"]

    def test_cache_headers_public(self):
        """Cache-Control: public 설정"""
        response = client.get("/api/v1/system/health")
        assert "Cache-Control" in response.headers
        assert "public" in response.headers["Cache-Control"]
        assert "max-age=300" in response.headers["Cache-Control"]

    def test_well_known_health_endpoint(self):
        """/.well-known/mimir-health 별칭"""
        response = client.get("/.well-known/mimir-health")
        assert response.status_code == 200


# ========== Tier 2: Capabilities (인증 필요) ==========

class TestCapabilitiesEndpoint:
    """Tier 2 — /api/v1/system/capabilities 테스트"""

    def test_requires_authentication(self):
        """인증 없이 접근 시 401"""
        response = client.get("/api/v1/system/capabilities")
        assert response.status_code == 401

    def test_authenticated_access(self):
        """인증된 사용자 접근 가능"""
        headers = get_auth_headers()
        response = client.get(
            "/api/v1/system/capabilities",
            headers=headers,
        )
        assert response.status_code == 200

    def test_response_schema(self):
        """Tier 2 응답 스키마 검증 (Admin 정보 미포함)"""
        headers = get_auth_headers()
        response = client.get(
            "/api/v1/system/capabilities",
            headers=headers,
        )
        data = response.json()

        assert data["success"] is True
        capabilities = data["data"]
        assert "version" in capabilities
        assert "rag_available" in capabilities
        assert "chunking_enabled" in capabilities
        assert "mcp_spec_version" in capabilities
        
        # Admin 전용 정보가 노출되지 않는지 확인
        assert "pgvector_enabled" not in capabilities
        assert "supported_providers" not in capabilities
        assert "deployment_type" not in capabilities
        assert "closed_network" not in capabilities

    @patch.dict(os.environ, {"CHUNKING_ENABLED": "false"})
    def test_chunking_disabled(self):
        """청킹 비활성화 반영"""
        headers = get_auth_headers()
        response = client.get(
            "/api/v1/system/capabilities",
            headers=headers,
        )
        data = response.json()
        assert data["data"]["chunking_enabled"] is False

    def test_cache_headers_private(self):
        """Cache-Control: private 설정 (인증 응답)"""
        headers = get_auth_headers()
        response = client.get(
            "/api/v1/system/capabilities",
            headers=headers,
        )
        assert "Cache-Control" in response.headers
        assert "private" in response.headers["Cache-Control"]


# ========== Tier 3: Admin Capabilities (Admin 전용) ==========

class TestAdminCapabilitiesEndpoint:
    """Tier 3 — /api/v1/admin/system/capabilities 테스트"""

    def test_requires_admin(self):
        """일반 사용자 접근 시 403"""
        headers = get_auth_headers()
        response = client.get(
            "/api/v1/admin/system/capabilities",
            headers=headers,
        )
        assert response.status_code == 403

    def test_admin_access(self):
        """Admin 사용자 접근 가능"""
        headers = get_admin_auth_headers()
        response = client.get(
            "/api/v1/admin/system/capabilities",
            headers=headers,
        )
        assert response.status_code == 200

    def test_full_response_schema(self):
        """Tier 3 — 전체 시스템 정보 포함"""
        headers = get_admin_auth_headers()
        response = client.get(
            "/api/v1/admin/system/capabilities",
            headers=headers,
        )
        data = response.json()

        assert data["success"] is True
        admin_cap = data["data"]
        assert "version" in admin_cap
        assert "pgvector_enabled" in admin_cap
        assert "rag_available" in admin_cap
        assert "chunking_enabled" in admin_cap
        assert "supported_providers" in admin_cap
        assert "mcp_spec_version" in admin_cap
        assert "deployment_type" in admin_cap
        assert "closed_network" in admin_cap

    @patch.dict(os.environ, {
        "PGVECTOR_ENABLED": "true",
        "RAG_ENABLED": "true",
    })
    def test_pgvector_enabled(self):
        """pgvector 활성화 상태"""
        headers = get_admin_auth_headers()
        response = client.get(
            "/api/v1/admin/system/capabilities",
            headers=headers,
        )
        data = response.json()
        assert data["data"]["pgvector_enabled"] is True
        assert data["data"]["rag_available"] is True

    @patch.dict(os.environ, {
        "PGVECTOR_ENABLED": "false",
        "RAG_ENABLED": "true",
    })
    def test_pgvector_disabled_rag_unavailable(self):
        """pgvector 비활성화 시 RAG 불가"""
        headers = get_admin_auth_headers()
        response = client.get(
            "/api/v1/admin/system/capabilities",
            headers=headers,
        )
        data = response.json()
        assert data["data"]["pgvector_enabled"] is False
        assert data["data"]["rag_available"] is False

    @patch.dict(os.environ, {"OPENAI_API_KEY": "sk-test"})
    def test_supported_providers(self):
        """LLM provider 목록 (Admin만 확인 가능)"""
        headers = get_admin_auth_headers()
        response = client.get(
            "/api/v1/admin/system/capabilities",
            headers=headers,
        )
        data = response.json()
        assert "openai" in data["data"]["supported_providers"]

    def test_unauthenticated_access_denied(self):
        """비인증 접근 시 401"""
        response = client.get("/api/v1/admin/system/capabilities")
        assert response.status_code == 401
```

### 4-6. 프론트엔드 연동 (참고)

프론트엔드에서 tier별 기능 가용성 확인:

```typescript
// /frontend/src/lib/api/system.ts

import { ApiResponse, unwrapEnvelope } from './types';

// Tier 2 — 인증 사용자용 (provider 상세 미포함)
export interface CapabilitiesData {
  version: string;
  rag_available: boolean;
  chunking_enabled: boolean;
  mcp_spec_version: string | null;
}

// Tier 3 — Admin 전용 (전체 정보)
export interface AdminCapabilitiesData extends CapabilitiesData {
  pgvector_enabled: boolean;
  supported_providers: string[];
  deployment_type: string;
  closed_network: boolean;
}

// Tier 1 — Public Health (인증 불필요)
export interface HealthData {
  status: 'ok' | 'degraded' | 'error';
  version: string;
}

export async function getHealth(): Promise<HealthData> {
  const response = await fetch('/api/v1/system/health');
  return unwrapEnvelope<HealthData>(await response.json());
}

export async function getCapabilities(
  authToken: string
): Promise<CapabilitiesData> {
  const response = await fetch('/api/v1/system/capabilities', {
    headers: { Authorization: `Bearer ${authToken}` },
  });
  return unwrapEnvelope<CapabilitiesData>(await response.json());
}

export async function getAdminCapabilities(
  authToken: string
): Promise<AdminCapabilitiesData> {
  const response = await fetch('/api/v1/admin/system/capabilities', {
    headers: { Authorization: `Bearer ${authToken}` },
  });
  return unwrapEnvelope<AdminCapabilitiesData>(await response.json());
}
```

사용 예:

```typescript
// /frontend/src/components/SearchBar.tsx

'use client';
import { useEffect, useState } from 'react';
import { getCapabilities } from '@/lib/api/system';
import { useAuth } from '@/hooks/useAuth';

export function SearchBar() {
  const { token } = useAuth();
  const [ragAvailable, setRagAvailable] = useState(false);

  useEffect(() => {
    if (!token) return;
    getCapabilities(token).then(cap => {
      setRagAvailable(cap.rag_available);
    }).catch(err => {
      console.error('Failed to get capabilities:', err);
    });
  }, [token]);

  return (
    <div>
      <input type="text" placeholder="검색..." />
      {ragAvailable ? (
        <button>시맨틱 검색</button>
      ) : (
        <button>기본 검색</button>
      )}
    </div>
  );
}
```

---

## 5. 산출물

1. **`/backend/app/api/v1/schemas/system.py`** (신규)
   - HealthData, HealthResponse (Tier 1)
   - CapabilitiesData, CapabilitiesResponse (Tier 2)
   - AdminCapabilitiesData, AdminCapabilitiesResponse (Tier 3)

2. **`/backend/app/api/v1/system.py`** (신규)
   - Tier 1: GET /api/v1/system/health (인증 불필요)
   - Tier 2: GET /api/v1/system/capabilities (인증 필요)
   - Tier 3: GET /api/v1/admin/system/capabilities (Admin 필요)
   - /.well-known/mimir-health 별칭 (Tier 1)

3. **`/backend/app/api/v1/__init__.py`** (수정)
   - system.router + system.admin_router 등록

4. **`.env.example`** (수정)
   - 환경변수 예시 추가

5. **`/backend/tests/unit/test_system_endpoints.py`** (신규)
   - Tier별 인증 계층 검증 (401/403)
   - 응답 스키마 검증 (각 tier에서 비노출 필드 확인)
   - pgvector on/off 환경별 응답 검증
   - 캐시 동작 검증

6. **API 문서**
   - Swagger/OpenAPI 자동 생성 (FastAPI)

---

## 6. 완료 기준

1. **Tier 1**: `/api/v1/system/health` — 인증 없이 접근 가능, status + version만 반환
2. **Tier 2**: `/api/v1/system/capabilities` — 비인증 시 401, 인증 사용자에게 rag_available/chunking_enabled 반환
3. **Tier 3**: `/api/v1/admin/system/capabilities` — 일반 사용자 403, Admin에게 전체 정보(pgvector, providers 등) 반환
4. **정보 격리 검증**: Tier 1에서 pgvector/providers 미노출, Tier 2에서 pgvector/providers/deployment_type 미노출
5. pgvector=true/false 환경별 응답 검증 (Tier 3)
6. 5분 캐시 동작 확인 (Tier 1: public, Tier 2/3: private)
7. 단위 테스트 모두 통과
8. OpenAPI 문서에 3개 엔드포인트 모두 등재
9. Task 0-7 Fresh-boot CI에서 health 엔드포인트 호출 가능

---

## 7. 작업 지침

### 지침 7-1. Phase 1 확장 준비

`supported_providers` 필드는 Tier 3 (Admin) 응답에서만 노출된다. Phase 1에서 새 provider 추가 시 `_get_supported_providers()` 함수만 확장하면 됨. Tier 2 응답에는 영향 없음.

### 지침 7-2. MCP 스펙 버전 관리

Phase 4에서 MCP 스펙 2025-11-25을 지원할 때 `_get_mcp_spec_version()` 함수에서 `MCP_ENABLED` 환경변수를 확인하여 반환. Tier 2, Tier 3 양쪽에 모두 노출됨.

### 지침 7-3. 캐시 무효화 방법

환경변수가 런타임에 변경되는 경우 (드물지만):

```python
# 관리자 API로 캐시 무효화 (Tier 3 Admin 라우터에 추가)
@admin_router.post("/cache/clear", tags=["Admin System"])
async def clear_system_cache(
    current_user: User = Depends(get_current_admin_user),
):
    """3-tier 캐시 전체 무효화 (Admin 전용)"""
    for cache in [_health_cache, _capabilities_cache, _admin_capabilities_cache]:
        cache["data"] = None
        cache["timestamp"] = None
    return {"success": True}
```

### 지침 7-4. 에러 처리

Tier 1 health 엔드포인트는 DB 연결 실패 시에도 응답해야 한다 (status를 `degraded`로 반환). `_get_service_status()` 함수에서 예외를 잡아 안전하게 처리:

```python
def _get_service_status() -> ServiceStatus:
    try:
        # DB ping, 필수 서비스 확인
        return ServiceStatus.OK
    except Exception as e:
        logger.warning(f"Service health check failed: {e}")
        return ServiceStatus.DEGRADED
```

### 지침 7-5. 보안 고려사항 (3-tier 분리 근거)

- **Tier 1 (Health)**: 인증 불필요. status와 version만 노출. 공격 표면 정보 없음
- **Tier 2 (Capabilities)**: 인증 필요. 기능 활성 여부만 노출. provider 상세/인프라 정보 미포함
- **Tier 3 (Admin Capabilities)**: Admin 전용. provider 목록, pgvector 상태, deployment 타입 등 내부 구성 정보 포함
- **Cache-Control**: Tier 1은 `public`, Tier 2/3은 `private` (인증 응답 캐시 분리)
- CORS 설정 적절히 (필요시 화이트리스트)
- 레이트 제한 (선택): Tier 1에 `slowapi` 적용 (public이므로 남용 방지)

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@limiter.limit("100/minute")  # 분당 100회
@router.get("/health")
async def get_health(response: Response):
    ...
```
