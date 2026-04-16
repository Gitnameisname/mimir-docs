# Task 4-1. MCP Server 초기화 + 핸드셰이크 + .well-known 메타데이터

## 1. 작업 목적

MCP (Model Context Protocol) 2025-11-25 스펙을 기반으로 **Mimir MCP Server 초기화 및 핸드셰이크 로직**을 구현하고, **`.well-known/mimir-mcp` 엔드포인트**를 통해 에이전트 클라이언트가 Mimir의 MCP 기능을 자동 발견할 수 있도록 한다. 이는 AI 에이전트가 표준 계약으로 Mimir를 도구로 호출하기 위한 기초를 제공한다.

## 2. 작업 범위

### 포함 범위

1. **MCP Server 클래스 구현 (Python)**
   - MCPServer: MCP 프로토콜을 처리하는 메인 서버 클래스
   - MCPRequestHandler: HTTP POST 요청 → MCP 요청 해석
   - protocol_version 검증 로직

2. **MCP initialize 요청/응답 처리**
   - 클라이언트 메타데이터 (client_id, protocol_version) 수신
   - 서버 메타데이터 (server_id="mimir-s2", version, capabilities) 응답
   - 지원 기능 선언: tools, resources, prompts, tasks

3. **OAuth 2.0 client-credentials 인증 통합**
   - Authorization 헤더에서 OAuth token 추출
   - 기존 S1 인증 시스템과 통합
   - 토큰 검증 로직

4. **`.well-known` 메타데이터 엔드포인트 (FastAPI)**
   - GET /.well-known/mimir-mcp
   - 응답: mcp_version, server_id, capabilities, authentication, scope_profile_required, documentation_url
   - 에이전트 클라이언트 자동 발견 지원

5. **MCP 프로토콜 버전 검증**
   - 클라이언트 protocol_version과 서버 호환성 검사
   - 버전 불일치 시 적절한 에러 응답

6. **MCP 스펙 준수 에러 응답**
   - error code (INVALID_REQUEST, AUTHENTICATION_FAILED, etc.)
   - error message

7. **단위 테스트**
   - MCP 핸드셰이크 성공 케이스
   - 버전 불일치 에러 처리
   - 인증 실패 에러 처리
   - `.well-known` 메타데이터 정확성

### 제외 범위

- 도구 구현 (Task 4-2: search_documents, fetch_node, verify_citation)
- Resource 노출 (Task 4-3: mimir:// URI scheme)
- Prompt Registry 노출 (Task 4-3)
- SSE/스트리밍 지원 (Task 4-3)

## 3. 선행 조건

- Phase 1, 2, 3 완료
- MCP 2025-11-25 스펙 분석 완료
- OAuth 2.0 client-credentials 인증 시스템 (S1에서 기존)
- FastAPI 프로젝트 구조 확립
- PostgreSQL 및 SQLAlchemy 설정 완료

## 4. 주요 작업 항목

### 4-1. MCP Server 클래스 구현

**파일:** `/backend/app/mcp/server.py`

```python
"""MCP Server 구현 (MCP 2025-11-25 스펙)."""

from __future__ import annotations

from typing import Optional, Dict, Any, List
from uuid import UUID, uuid4
from datetime import datetime
import json
import logging

from pydantic import BaseModel, Field

logger = logging.getLogger(__name__)


# ==================== MCP Protocol Models ====================

class ProtocolVersion(BaseModel):
    """MCP 프로토콜 버전 정보."""
    major: int
    minor: int
    patch: int

    def __str__(self) -> str:
        return f"{self.major}.{self.minor}.{self.patch}"
    
    @classmethod
    def from_string(cls, version_str: str) -> ProtocolVersion:
        """문자열로부터 버전 파싱 (예: "2025-11-25")."""
        parts = version_str.split("-")
        if len(parts) == 3:
            try:
                return cls(major=int(parts[0]), minor=int(parts[1]), patch=int(parts[2]))
            except (ValueError, IndexError):
                pass
        raise ValueError(f"Invalid protocol version: {version_str}")
    
    def is_compatible_with(self, other: ProtocolVersion) -> bool:
        """다른 버전과의 호환성 확인 (major.minor 일치)."""
        return self.major == other.major and self.minor == other.minor


class ClientMetadata(BaseModel):
    """클라이언트 메타데이터 (initialize 요청에서)."""
    client_id: str
    protocol_version: str  # "2025-11-25"


class ServerCapabilities(BaseModel):
    """서버 기능 선언."""
    tools: bool = True
    resources: bool = True
    prompts: bool = True
    tasks: bool = False  # Phase 5에서 true로 변경
    
    def to_dict(self) -> Dict[str, bool]:
        return self.dict()


class ServerMetadata(BaseModel):
    """서버 메타데이터 (initialize 응답에서)."""
    server_id: str = "mimir-s2"
    protocol_version: str  # "2025-11-25"
    name: str = "Mimir S2"
    version: str = "1.0.0"
    capabilities: ServerCapabilities
    
    def to_dict(self) -> Dict[str, Any]:
        return {
            "server_id": self.server_id,
            "protocol_version": self.protocol_version,
            "name": self.name,
            "version": self.version,
            "capabilities": self.capabilities.to_dict(),
        }


class MCPInitializeRequest(BaseModel):
    """MCP initialize 요청."""
    jsonrpc: str = "2.0"
    method: str = "initialize"
    params: Dict[str, Any]
    id: int | str


class MCPInitializeResponse(BaseModel):
    """MCP initialize 응답."""
    jsonrpc: str = "2.0"
    result: Dict[str, Any]
    id: int | str


class MCPErrorResponse(BaseModel):
    """MCP 에러 응답."""
    jsonrpc: str = "2.0"
    error: Dict[str, Any]
    id: Optional[int | str] = None


# ==================== MCP Server ====================

class MCPServer:
    """MCP 2025-11-25 스펙을 구현하는 Mimir MCP Server."""
    
    SUPPORTED_VERSION = "2025-11-25"
    
    def __init__(self, server_id: str = "mimir-s2", version: str = "1.0.0"):
        """MCP Server 초기화."""
        self.server_id = server_id
        self.version = version
        self.protocol_version = self.SUPPORTED_VERSION
        self.capabilities = ServerCapabilities(
            tools=True,
            resources=True,
            prompts=True,
            tasks=False,
        )
        self.initialized_clients: Dict[str, Dict[str, Any]] = {}
    
    async def handle_initialize(
        self,
        client_metadata: ClientMetadata,
        request_id: int | str,
    ) -> MCPInitializeResponse | MCPErrorResponse:
        """MCP initialize 요청 처리.
        
        Args:
            client_metadata: 클라이언트 메타데이터 (client_id, protocol_version)
            request_id: 요청 ID
        
        Returns:
            MCPInitializeResponse 또는 MCPErrorResponse
        """
        logger.info(f"MCP initialize request from {client_metadata.client_id}")
        
        # 클라이언트 프로토콜 버전 검증
        try:
            client_version = ProtocolVersion.from_string(client_metadata.protocol_version)
        except ValueError as e:
            logger.error(f"Invalid client protocol version: {client_metadata.protocol_version}")
            return MCPErrorResponse(
                id=request_id,
                error={
                    "code": "INVALID_REQUEST",
                    "message": f"Invalid protocol version: {str(e)}",
                }
            )
        
        server_version = ProtocolVersion.from_string(self.protocol_version)
        
        # 호환성 검사
        if not client_version.is_compatible_with(server_version):
            logger.warning(
                f"Protocol version mismatch: client {client_version} vs server {server_version}"
            )
            return MCPErrorResponse(
                id=request_id,
                error={
                    "code": "VERSION_MISMATCH",
                    "message": f"Server supports version {server_version}, client uses {client_version}",
                }
            )
        
        # 클라이언트 초기화 상태 기록
        self.initialized_clients[client_metadata.client_id] = {
            "client_id": client_metadata.client_id,
            "protocol_version": client_metadata.protocol_version,
            "initialized_at": datetime.utcnow().isoformat(),
        }
        
        logger.info(f"Client {client_metadata.client_id} initialized successfully")
        
        # 응답
        server_metadata = ServerMetadata(
            server_id=self.server_id,
            protocol_version=self.protocol_version,
            name="Mimir S2",
            version=self.version,
            capabilities=self.capabilities,
        )
        
        return MCPInitializeResponse(
            id=request_id,
            result={
                "server": server_metadata.to_dict(),
                "protocolVersion": self.protocol_version,
                "capabilities": self.capabilities.to_dict(),
            }
        )
    
    def get_server_metadata(self) -> ServerMetadata:
        """서버 메타데이터 반환."""
        return ServerMetadata(
            server_id=self.server_id,
            protocol_version=self.protocol_version,
            version=self.version,
            capabilities=self.capabilities,
        )
    
    def is_client_initialized(self, client_id: str) -> bool:
        """클라이언트 초기화 여부 확인."""
        return client_id in self.initialized_clients


# ==================== Request Handler ====================

class MCPRequestHandler:
    """HTTP POST 요청을 MCP 요청으로 해석하고 처리."""
    
    def __init__(self, server: MCPServer):
        self.server = server
    
    async def handle_request(
        self,
        request_body: Dict[str, Any],
        client_id: str,
    ) -> Dict[str, Any]:
        """MCP 요청 처리.
        
        Args:
            request_body: HTTP POST 바디 (JSON-RPC 형식)
            client_id: 클라이언트 ID (헤더에서 추출)
        
        Returns:
            JSON-RPC 응답 객체
        """
        method = request_body.get("method")
        params = request_body.get("params", {})
        request_id = request_body.get("id")
        
        logger.debug(f"MCP request: method={method}, client_id={client_id}")
        
        if method == "initialize":
            try:
                client_metadata = ClientMetadata(
                    client_id=client_id,
                    protocol_version=params.get("protocol_version", "2025-11-25")
                )
                response = await self.server.handle_initialize(client_metadata, request_id)
                return response.dict()
            except Exception as e:
                logger.error(f"Initialize error: {str(e)}")
                return MCPErrorResponse(
                    id=request_id,
                    error={
                        "code": "INTERNAL_ERROR",
                        "message": str(e),
                    }
                ).dict()
        else:
            return MCPErrorResponse(
                id=request_id,
                error={
                    "code": "METHOD_NOT_FOUND",
                    "message": f"Unknown method: {method}",
                }
            ).dict()
```

### 4-2. FastAPI 라우터 및 인증 통합

**파일:** `/backend/app/routes/mcp.py`

```python
"""MCP Server HTTP 라우터 및 .well-known 엔드포인트."""

from __future__ import annotations

from typing import Optional, Dict, Any
from uuid import UUID
import logging

from fastapi import APIRouter, Depends, HTTPException, Header, Request
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.auth import verify_oauth_token  # 기존 OAuth 검증 함수
from app.mcp.server import MCPServer, MCPRequestHandler

logger = logging.getLogger(__name__)

# 전역 MCP Server 인스턴스
_mcp_server: Optional[MCPServer] = None


def get_mcp_server() -> MCPServer:
    """전역 MCP Server 인스턴스 반환 (초기화)."""
    global _mcp_server
    if _mcp_server is None:
        _mcp_server = MCPServer(server_id="mimir-s2", version="1.0.0")
    return _mcp_server


router = APIRouter(prefix="/mcp", tags=["mcp"])


# ==================== .well-known 엔드포인트 ====================

@router.get("/.well-known/mimir-mcp")
async def get_mcp_metadata(db: AsyncSession = Depends(get_db)) -> Dict[str, Any]:
    """MCP 메타데이터 노출 (.well-known 표준).
    
    에이전트 클라이언트가 Mimir의 MCP 기능을 자동 발견할 수 있도록 한다.
    
    Returns:
        MCP 메타데이터 (JSON):
        - mcp_version: "2025-11-25"
        - server_id: "mimir-s2"
        - capabilities: {tools, resources, prompts, tasks}
        - authentication: "oauth2_client_credentials"
        - scope_profile_required: true
        - documentation_url: "https://mimir.example.com/docs/mcp"
    """
    server = get_mcp_server()
    metadata = server.get_server_metadata()
    
    return {
        "mcp_version": server.protocol_version,
        "server_id": metadata.server_id,
        "name": metadata.name,
        "version": metadata.version,
        "capabilities": metadata.capabilities.dict(),
        "authentication": {
            "type": "oauth2",
            "scheme": "client_credentials",
            "token_endpoint": "https://mimir.example.com/oauth/token",  # 조직별로 변경 가능
            "required": True,
        },
        "scope_profile_required": True,
        "documentation_url": "https://mimir.example.com/docs/mcp",
    }


# ==================== MCP 요청 처리 ====================

@router.post("/initialize")
async def handle_mcp_initialize(
    request: Request,
    authorization: Optional[str] = Header(None),
    db: AsyncSession = Depends(get_db),
) -> Dict[str, Any]:
    """MCP initialize 요청 처리.
    
    요청 헤더:
        Authorization: Bearer <oauth_token> (필수)
        Content-Type: application/json
    
    요청 바디 (JSON-RPC 2.0):
        {
          "jsonrpc": "2.0",
          "method": "initialize",
          "params": {
            "client_id": "chatbot-123",
            "protocol_version": "2025-11-25"
          },
          "id": 1
        }
    
    응답:
        JSON-RPC 형식의 initialize 응답
    """
    logger.info("MCP initialize endpoint called")
    
    # OAuth 토큰 검증
    if not authorization:
        logger.error("Missing Authorization header")
        return {
            "jsonrpc": "2.0",
            "error": {
                "code": "AUTHENTICATION_FAILED",
                "message": "Missing Authorization header",
            },
            "id": None,
        }
    
    try:
        # "Bearer <token>" 형식 파싱
        parts = authorization.split(" ")
        if len(parts) != 2 or parts[0].lower() != "bearer":
            raise ValueError("Invalid Authorization header format")
        
        token = parts[1]
        
        # OAuth 토큰 검증 (기존 S1 인증 시스템 활용)
        oauth_info = await verify_oauth_token(token, db)
        if not oauth_info:
            raise ValueError("Invalid or expired token")
        
        client_id = oauth_info.get("client_id")
        
    except Exception as e:
        logger.error(f"OAuth authentication failed: {str(e)}")
        return {
            "jsonrpc": "2.0",
            "error": {
                "code": "AUTHENTICATION_FAILED",
                "message": f"OAuth token validation failed: {str(e)}",
            },
            "id": None,
        }
    
    # 요청 바디 파싱
    try:
        body = await request.json()
    except Exception as e:
        logger.error(f"Invalid request body: {str(e)}")
        return {
            "jsonrpc": "2.0",
            "error": {
                "code": "INVALID_REQUEST",
                "message": f"Invalid JSON: {str(e)}",
            },
            "id": None,
        }
    
    # MCP 요청 처리
    server = get_mcp_server()
    handler = MCPRequestHandler(server)
    
    try:
        response = await handler.handle_request(body, client_id=client_id)
        logger.info(f"MCP initialize successful for client {client_id}")
        return response
    except Exception as e:
        logger.error(f"MCP request handling error: {str(e)}")
        return {
            "jsonrpc": "2.0",
            "error": {
                "code": "INTERNAL_ERROR",
                "message": str(e),
            },
            "id": body.get("id"),
        }


# ==================== Health Check ====================

@router.get("/health")
async def mcp_health_check() -> Dict[str, Any]:
    """MCP Server 상태 확인."""
    server = get_mcp_server()
    return {
        "status": "healthy",
        "server_id": server.server_id,
        "version": server.version,
        "protocol_version": server.protocol_version,
    }
```

### 4-3. 설정 및 의존성 주입

**파일:** `/backend/app/config/mcp.py`

```python
"""MCP Server 설정."""

from pydantic import BaseSettings


class MCPConfig(BaseSettings):
    """MCP Server 설정."""
    
    server_id: str = "mimir-s2"
    version: str = "1.0.0"
    protocol_version: str = "2025-11-25"
    
    # OAuth 설정
    oauth_token_endpoint: str = "https://mimir.example.com/oauth/token"
    oauth_issuer: str = "https://mimir.example.com"
    
    # 문서 URL
    documentation_url: str = "https://mimir.example.com/docs/mcp"
    
    class Config:
        env_file = ".env"
        env_prefix = "MCP_"
```

### 4-4. 단위 테스트

**파일:** `/backend/tests/unit/mcp/test_mcp_server.py`

```python
"""MCP Server 단위 테스트."""

import pytest
from unittest.mock import AsyncMock, patch

from app.mcp.server import (
    MCPServer, MCPRequestHandler, ClientMetadata, ProtocolVersion,
    MCPInitializeRequest, MCPInitializeResponse, MCPErrorResponse
)


class TestProtocolVersion:
    """ProtocolVersion 테스트."""
    
    def test_parse_valid_version(self):
        """유효한 버전 파싱."""
        version = ProtocolVersion.from_string("2025-11-25")
        assert version.major == 2025
        assert version.minor == 11
        assert version.patch == 25
    
    def test_parse_invalid_version(self):
        """유효하지 않은 버전 파싱."""
        with pytest.raises(ValueError):
            ProtocolVersion.from_string("invalid-version")
    
    def test_version_compatibility(self):
        """버전 호환성 검사."""
        v1 = ProtocolVersion(major=2025, minor=11, patch=25)
        v2 = ProtocolVersion(major=2025, minor=11, patch=26)
        v3 = ProtocolVersion(major=2025, minor=10, patch=25)
        
        assert v1.is_compatible_with(v2)  # 같은 major.minor
        assert not v1.is_compatible_with(v3)  # 다른 minor


class TestMCPServer:
    """MCPServer 테스트."""
    
    @pytest.mark.asyncio
    async def test_initialize_success(self):
        """성공적인 initialize 처리."""
        server = MCPServer()
        client_metadata = ClientMetadata(
            client_id="test-client",
            protocol_version="2025-11-25"
        )
        
        response = await server.handle_initialize(client_metadata, request_id=1)
        
        assert isinstance(response, MCPInitializeResponse)
        assert response.result["server"]["server_id"] == "mimir-s2"
        assert response.result["protocolVersion"] == "2025-11-25"
        assert response.result["capabilities"]["tools"] is True
        assert response.result["capabilities"]["resources"] is True
        assert response.result["capabilities"]["prompts"] is True
        assert response.result["capabilities"]["tasks"] is False
    
    @pytest.mark.asyncio
    async def test_initialize_version_mismatch(self):
        """버전 불일치 에러."""
        server = MCPServer()
        client_metadata = ClientMetadata(
            client_id="test-client",
            protocol_version="2024-01-01"  # 다른 버전
        )
        
        response = await server.handle_initialize(client_metadata, request_id=1)
        
        assert isinstance(response, MCPErrorResponse)
        assert response.error["code"] == "VERSION_MISMATCH"
    
    @pytest.mark.asyncio
    async def test_initialize_invalid_version_format(self):
        """유효하지 않은 버전 형식."""
        server = MCPServer()
        client_metadata = ClientMetadata(
            client_id="test-client",
            protocol_version="invalid"
        )
        
        response = await server.handle_initialize(client_metadata, request_id=1)
        
        assert isinstance(response, MCPErrorResponse)
        assert response.error["code"] == "INVALID_REQUEST"
    
    @pytest.mark.asyncio
    async def test_client_initialized_tracking(self):
        """클라이언트 초기화 추적."""
        server = MCPServer()
        client_metadata = ClientMetadata(
            client_id="test-client",
            protocol_version="2025-11-25"
        )
        
        # 초기화 전
        assert not server.is_client_initialized("test-client")
        
        # 초기화 후
        await server.handle_initialize(client_metadata, request_id=1)
        assert server.is_client_initialized("test-client")


class TestMCPRequestHandler:
    """MCPRequestHandler 테스트."""
    
    @pytest.mark.asyncio
    async def test_handle_initialize_request(self):
        """initialize 요청 처리."""
        server = MCPServer()
        handler = MCPRequestHandler(server)
        
        request_body = {
            "jsonrpc": "2.0",
            "method": "initialize",
            "params": {
                "protocol_version": "2025-11-25"
            },
            "id": 1
        }
        
        response = await handler.handle_request(request_body, client_id="test-client")
        
        assert response["jsonrpc"] == "2.0"
        assert "result" in response
        assert response["result"]["server"]["server_id"] == "mimir-s2"
    
    @pytest.mark.asyncio
    async def test_handle_unknown_method(self):
        """알 수 없는 메서드 처리."""
        server = MCPServer()
        handler = MCPRequestHandler(server)
        
        request_body = {
            "jsonrpc": "2.0",
            "method": "unknown_method",
            "params": {},
            "id": 1
        }
        
        response = await handler.handle_request(request_body, client_id="test-client")
        
        assert response["jsonrpc"] == "2.0"
        assert "error" in response
        assert response["error"]["code"] == "METHOD_NOT_FOUND"
```

### 4-5. FastAPI 라우터 통합 테스트

**파일:** `/backend/tests/integration/test_mcp_endpoints.py`

```python
"""MCP API 엔드포인트 통합 테스트."""

import pytest
from httpx import AsyncClient
from unittest.mock import AsyncMock, patch

from app.main import app


@pytest.fixture
async def client():
    """테스트 클라이언트."""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac


@pytest.mark.asyncio
async def test_well_known_endpoint(client):
    """/.well-known/mimir-mcp 엔드포인트 테스트."""
    response = await client.get("/.well-known/mimir-mcp")
    
    assert response.status_code == 200
    data = response.json()
    
    assert data["mcp_version"] == "2025-11-25"
    assert data["server_id"] == "mimir-s2"
    assert "capabilities" in data
    assert data["capabilities"]["tools"] is True
    assert data["capabilities"]["resources"] is True
    assert data["capabilities"]["prompts"] is True
    assert data["capabilities"]["tasks"] is False
    assert data["authentication"]["type"] == "oauth2"
    assert data["authentication"]["scheme"] == "client_credentials"
    assert data["scope_profile_required"] is True


@pytest.mark.asyncio
async def test_initialize_without_auth(client):
    """인증 없이 initialize 요청."""
    request_body = {
        "jsonrpc": "2.0",
        "method": "initialize",
        "params": {
            "protocol_version": "2025-11-25"
        },
        "id": 1
    }
    
    response = await client.post(
        "/mcp/initialize",
        json=request_body,
    )
    
    assert response.status_code == 200
    data = response.json()
    assert "error" in data
    assert data["error"]["code"] == "AUTHENTICATION_FAILED"


@pytest.mark.asyncio
async def test_initialize_with_valid_token(client):
    """유효한 토큰으로 initialize 요청."""
    request_body = {
        "jsonrpc": "2.0",
        "method": "initialize",
        "params": {
            "protocol_version": "2025-11-25"
        },
        "id": 1
    }
    
    with patch("app.auth.verify_oauth_token") as mock_verify:
        mock_verify.return_value = {"client_id": "test-agent"}
        
        response = await client.post(
            "/mcp/initialize",
            json=request_body,
            headers={"Authorization": "Bearer test-token"},
        )
    
    assert response.status_code == 200
    data = response.json()
    
    assert "result" in data
    assert data["result"]["server"]["server_id"] == "mimir-s2"
    assert data["result"]["protocolVersion"] == "2025-11-25"


@pytest.mark.asyncio
async def test_initialize_with_invalid_token(client):
    """유효하지 않은 토큰으로 initialize 요청."""
    request_body = {
        "jsonrpc": "2.0",
        "method": "initialize",
        "params": {
            "protocol_version": "2025-11-25"
        },
        "id": 1
    }
    
    with patch("app.auth.verify_oauth_token") as mock_verify:
        mock_verify.return_value = None  # 검증 실패
        
        response = await client.post(
            "/mcp/initialize",
            json=request_body,
            headers={"Authorization": "Bearer invalid-token"},
        )
    
    assert response.status_code == 200
    data = response.json()
    assert "error" in data
    assert data["error"]["code"] == "AUTHENTICATION_FAILED"


@pytest.mark.asyncio
async def test_mcp_health_check(client):
    """MCP health check 엔드포인트."""
    response = await client.get("/mcp/health")
    
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "healthy"
    assert data["server_id"] == "mimir-s2"
    assert data["protocol_version"] == "2025-11-25"
```

## 5. 산출물

1. **MCP Server 클래스** (`/backend/app/mcp/server.py`)
   - MCPServer: MCP 프로토콜 처리
   - MCPRequestHandler: HTTP 요청 해석
   - 프로토콜 버전 검증 로직

2. **FastAPI 라우터** (`/backend/app/routes/mcp.py`)
   - POST /mcp/initialize: MCP 초기화 요청
   - GET /.well-known/mimir-mcp: 메타데이터 노출
   - GET /mcp/health: 상태 확인

3. **MCP 설정** (`/backend/app/config/mcp.py`)
   - 서버 ID, 버전, protocol_version
   - OAuth 설정
   - 문서 URL

4. **단위 테스트** (`/backend/tests/unit/mcp/test_mcp_server.py`)
   - 버전 파싱, 호환성 검사
   - initialize 성공/실패 케이스

5. **통합 테스트** (`/backend/tests/integration/test_mcp_endpoints.py`)
   - .well-known 메타데이터 정확성
   - 인증 기반 initialize 요청

## 6. 완료 기준

1. MCPServer 클래스가 MCP 2025-11-25 스펙을 구현했는가?
2. initialize 요청/응답이 정확한가?
3. 프로토콜 버전 검증이 정상 작동하는가?
4. OAuth 토큰 검증이 통합되었는가?
5. `.well-known/mimir-mcp` 엔드포인트가 정확한 메타데이터를 반환하는가?
6. 클라이언트 초기화 상태 추적이 작동하는가?
7. MCP 스펙 준수 에러 응답이 제공되는가?
8. 모든 단위/통합 테스트가 통과하는가?
9. mypy 타입 검사를 통과하는가?

## 7. 작업 지침

### 지침 7-1. MCP 스펙 호환성

MCP 2025-11-25 스펙에 정확하게 따라 초기화 요청/응답 형식을 구현한다. JSON-RPC 2.0 형식을 준수하고, 모든 필드명이 스펙과 일치하는지 확인한다.

### 지침 7-2. 프로토콜 버전 호환성

major.minor 버전이 일치해야 호환이라고 판단한다 (patch 버전은 무시). 버전 불일치 시 VERSION_MISMATCH 에러를 반환한다.

### 지침 7-3. OAuth 토큰 검증

Authorization 헤더에서 "Bearer <token>" 형식을 파싱하고, 기존 S1 인증 시스템의 verify_oauth_token 함수를 이용한다. 토큰 검증 실패 시 AUTHENTICATION_FAILED 에러를 반환한다.

### 지침 7-4. 에러 응답 표준화

모든 에러는 MCP 스펙 준수 형식으로 응답한다:
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": "ERROR_CODE",
    "message": "error description"
  },
  "id": request_id
}
```

### 지침 7-5. 클라이언트 초기화 추적

initialize 성공 시 클라이언트 ID와 메타데이터를 서버의 initialized_clients 딕셔너리에 기록한다. 이후 도구 호출 시 클라이언트 초기화 여부를 확인할 수 있다.

### 지침 7-6. .well-known 메타데이터

.well-known 엔드포인트는 인증을 요구하지 않으며, 공개적으로 접근 가능해야 한다. 에이전트 클라이언트가 Mimir의 MCP 기능을 자동 발견할 수 있도록 한다.

### 지침 7-7. S1 OAuth 시스템 활용

기존 S1의 OAuth 2.0 client-credentials 인증 시스템을 활용하여, 토큰 검증 및 클라이언트 ID 추출을 수행한다. 별도의 새로운 인증 시스템을 구축하지 않는다.
