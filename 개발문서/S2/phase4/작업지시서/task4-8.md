# Task 4-8. MCP Tool Schema 정의 + Mimir Extension 선언 + Curated Tool Set

## 1. 작업 목적

**3가지 읽기 도구(search_documents, fetch_node, verify_citation)의 MCP Tool Schema를 명시적으로 정의**하고, **Mimir 고유 기능을 MCP Extension으로 선언**하며, **에이전트에 노출할 도구 목록을 엄격하게 관리**한다. 이를 통해 에이전트 클라이언트가 Mimir의 능력을 정확하게 인식하고 안전하게 호출할 수 있도록 한다.

## 2. 작업 범위

### 포함 범위

1. **3가지 도구의 MCP Tool Schema 정의 (JSON Schema)**
   - search_documents: FTS/Vector 전문 검색
   - fetch_node: 특정 노드의 전문 조회
   - verify_citation: Citation 5-tuple 검증
   - 각 도구: name, description, inputSchema, outputSchema

2. **OpenAPI → MCP Tool Schema 변환**
   - 기존 OpenAPI 스펙을 수동 큐레이션하여 MCP Tool schema로 변환
   - 불필요한 필드 제거, 도구명/설명 간결화

3. **Mimir Extension 선언 (MCP Extensions)**
   - mimir.citation-5tuple v1.0: Citation 검증 기능 선언
   - mimir.span-backref v1.0: span 역참조 기능 선언
   - capabilities 선언 (각 extension의 기능)

4. **Curated Tool Set 관리**
   - CuratedToolSet 클래스: 도구 허용/차단 목록 관리
   - 도구별 인증 요구사항 명시:
     - oauth2_client_credentials (필수)
     - scope_profile 필수 여부
     - delegation 권한 필요 여부
   - Tool Discovery 엔드포인트: tools/list에서 허용된 도구만 반환

5. **도구별 인증 레벨 검증 미들웨어**
   - 도구 호출 시 인증 요구사항 확인
   - 부족한 권한이면 UNAUTHORIZED 에러 반환

6. **도구 메타데이터 저장**
   - tools 테이블 (또는 설정 파일): 도구 이름, 설명, 인증 요구사항

7. **단위 테스트**
   - Tool Schema 유효성 검사 (JSON Schema 준수)
   - Extension 호환성 검사
   - CuratedToolSet 필터링 로직
   - 도구별 인증 검증

### 제외 범위

- 응답 포맷 (Task 4-7)
- 도구 구현 세부사항 (Task 4-1~4-3)
- 통합 테스트 (Task 4-9)
- MCP 공식 문서 (Task 4-9)

## 3. 선행 조건

- Phase 1, 2, 3 완료
- FG4.1 (MCP Server) 완료
- FG4.2 (Agent Principal, Scope Profile) 완료
- Task 4-7 (응답 포맷) 완료
- 기존 OpenAPI 스펙 분석 완료

## 4. 주요 작업 항목

### 4-1. MCP Tool Schema 정의

**파일:** `/backend/app/mcp/tool_schemas.py`

```python
"""MCP Tool Schema 정의 (search_documents, fetch_node, verify_citation)."""

from typing import Dict, Any, List
from enum import Enum


class ToolSchemaVersion(str, Enum):
    """Tool Schema 버전."""
    V1_0 = "1.0"


# ==================== Tool 1: search_documents ====================

SEARCH_DOCUMENTS_SCHEMA: Dict[str, Any] = {
    "name": "search_documents",
    "version": ToolSchemaVersion.V1_0.value,
    "description": "Search Mimir documents using full-text search (FTS) or vector search",
    "category": "retrieval",
    "inputSchema": {
        "type": "object",
        "title": "SearchDocumentsInput",
        "properties": {
            "query": {
                "type": "string",
                "description": "Search query (required)",
                "minLength": 1,
                "maxLength": 1000,
            },
            "scope": {
                "type": "string",
                "description": "Access scope (e.g., 'team', 'org', 'public'). Managed by Scope Profile. (optional, default: agent's default scope)",
                "pattern": "^[a-z_]+$",
            },
            "document_types": {
                "type": "array",
                "description": "Filter by document types (e.g., ['contract', 'policy']). (optional)",
                "items": {
                    "type": "string",
                    "minLength": 1,
                },
                "maxItems": 10,
            },
            "top_k": {
                "type": "integer",
                "description": "Number of results to return (default: 5, max: 50)",
                "minimum": 1,
                "maximum": 50,
                "default": 5,
            },
            "conversation_id": {
                "type": "string",
                "format": "uuid",
                "description": "Conversation ID for context window. (optional)",
            },
        },
        "required": ["query"],
        "additionalProperties": False,
    },
    "outputSchema": {
        "type": "object",
        "title": "SearchDocumentsOutput",
        "properties": {
            "results": {
                "type": "array",
                "description": "List of search results",
                "items": {
                    "type": "object",
                    "title": "SearchResult",
                    "properties": {
                        "document_id": {
                            "type": "string",
                            "format": "uuid",
                        },
                        "document_title": {
                            "type": "string",
                        },
                        "version_id": {
                            "type": "string",
                            "format": "uuid",
                        },
                        "node_id": {
                            "type": "string",
                        },
                        "content": {
                            "type": "string",
                            "description": "Excerpt or full text",
                        },
                        "citation": {
                            "type": "object",
                            "title": "Citation5Tuple",
                            "properties": {
                                "document_id": {"type": "string", "format": "uuid"},
                                "version_id": {"type": "string", "format": "uuid"},
                                "node_id": {"type": "string"},
                                "span_offset": {"type": "integer"},
                                "content_hash": {"type": "string"},
                            },
                            "required": ["document_id", "version_id", "node_id", "content_hash"],
                        },
                        "relevance_score": {
                            "type": "number",
                            "minimum": 0.0,
                            "maximum": 1.0,
                        },
                        "retrieval_time_ms": {
                            "type": "integer",
                            "minimum": 0,
                        },
                    },
                    "required": ["document_id", "document_title", "content", "citation"],
                },
            },
            "total_count": {
                "type": "integer",
                "description": "Total number of results (may exceed returned results)",
            },
            "retrieval_method": {
                "type": "string",
                "enum": ["fts", "vector", "hybrid"],
                "description": "Method used for retrieval",
            },
        },
        "required": ["results", "total_count", "retrieval_method"],
    },
}


# ==================== Tool 2: fetch_node ====================

FETCH_NODE_SCHEMA: Dict[str, Any] = {
    "name": "fetch_node",
    "version": ToolSchemaVersion.V1_0.value,
    "description": "Fetch full content of a specific document node",
    "category": "retrieval",
    "inputSchema": {
        "type": "object",
        "title": "FetchNodeInput",
        "properties": {
            "document_id": {
                "type": "string",
                "format": "uuid",
                "description": "Document ID (required)",
            },
            "version_id": {
                "type": "string",
                "format": "uuid",
                "description": "Version ID. If not provided, fetches latest version. (optional)",
            },
            "node_id": {
                "type": "string",
                "description": "Node ID within the document (required)",
                "minLength": 1,
            },
        },
        "required": ["document_id", "node_id"],
        "additionalProperties": False,
    },
    "outputSchema": {
        "type": "object",
        "title": "FetchNodeOutput",
        "properties": {
            "document_id": {
                "type": "string",
                "format": "uuid",
            },
            "document_title": {
                "type": "string",
            },
            "version_id": {
                "type": "string",
                "format": "uuid",
            },
            "node_id": {
                "type": "string",
            },
            "content": {
                "type": "string",
                "description": "Full text content of the node",
            },
            "metadata": {
                "type": "object",
                "description": "Node metadata (document type, language, etc.)",
                "additionalProperties": True,
            },
            "relationships": {
                "type": "object",
                "description": "Node relationships (parent_node_id, children)",
                "properties": {
                    "parent_node_id": {
                        "type": "string",
                    },
                    "children": {
                        "type": "array",
                        "items": {"type": "string"},
                    },
                },
            },
        },
        "required": ["document_id", "document_title", "version_id", "node_id", "content"],
    },
}


# ==================== Tool 3: verify_citation ====================

VERIFY_CITATION_SCHEMA: Dict[str, Any] = {
    "name": "verify_citation",
    "version": ToolSchemaVersion.V1_0.value,
    "description": "Verify a citation using the 5-tuple (document_id, version_id, node_id, span_offset, content_hash)",
    "category": "validation",
    "inputSchema": {
        "type": "object",
        "title": "VerifyCitationInput",
        "properties": {
            "document_id": {
                "type": "string",
                "format": "uuid",
                "description": "Document ID (required)",
            },
            "version_id": {
                "type": "string",
                "format": "uuid",
                "description": "Version ID (required)",
            },
            "node_id": {
                "type": "string",
                "description": "Node ID (required)",
                "minLength": 1,
            },
            "content_hash": {
                "type": "string",
                "description": "SHA256 hash of the cited content (required)",
                "pattern": "^[a-f0-9]{64}$",
            },
            "span_offset": {
                "type": "integer",
                "description": "Character offset of the span in the node (optional)",
                "minimum": 0,
            },
        },
        "required": ["document_id", "version_id", "node_id", "content_hash"],
        "additionalProperties": False,
    },
    "outputSchema": {
        "type": "object",
        "title": "VerifyCitationOutput",
        "properties": {
            "verified": {
                "type": "boolean",
                "description": "Whether the citation is valid",
            },
            "current_hash": {
                "type": "string",
                "description": "Current SHA256 hash of the content",
            },
            "hash_matches": {
                "type": "boolean",
                "description": "Whether provided hash matches current content",
            },
            "content_snapshot": {
                "type": "string",
                "description": "Snapshot of the cited content (first 200 chars)",
            },
            "version_valid": {
                "type": "boolean",
                "description": "Whether the version still exists",
            },
            "message": {
                "type": "string",
                "description": "Verification result message",
            },
        },
        "required": ["verified", "hash_matches", "version_valid", "message"],
    },
}


# ==================== Tool Registry ====================

TOOL_SCHEMAS_V1: Dict[str, Dict[str, Any]] = {
    "search_documents": SEARCH_DOCUMENTS_SCHEMA,
    "fetch_node": FETCH_NODE_SCHEMA,
    "verify_citation": VERIFY_CITATION_SCHEMA,
}


def get_tool_schema(tool_name: str) -> Dict[str, Any]:
    """특정 도구의 schema 조회.
    
    Args:
        tool_name: 도구 이름
    
    Returns:
        Tool schema 딕셔너리
    """
    if tool_name not in TOOL_SCHEMAS_V1:
        raise ValueError(f"Unknown tool: {tool_name}")
    return TOOL_SCHEMAS_V1[tool_name]


def list_all_tool_schemas() -> List[Dict[str, Any]]:
    """모든 도구 schema 반환.
    
    Returns:
        Tool schema 리스트
    """
    return list(TOOL_SCHEMAS_V1.values())
```

### 4-2. Mimir Extension 선언

**파일:** `/backend/app/mcp/extensions.py`

```python
"""Mimir MCP Extensions 선언."""

from typing import Dict, Any, List
from enum import Enum


class ExtensionVersion(str, Enum):
    """Extension 버전."""
    V1_0 = "1.0"


# ==================== Extension 1: citation-5tuple ====================

CITATION_5TUPLE_EXTENSION: Dict[str, Any] = {
    "name": "mimir.citation-5tuple",
    "version": ExtensionVersion.V1_0.value,
    "description": "Citation verification using 5-tuple (document_id, version_id, node_id, span_offset, content_hash)",
    "spec_url": "https://mimir.example.com/docs/extensions/citation-5tuple",
    "capabilities": {
        "verify_citation": {
            "enabled": True,
            "description": "Verify citation using 5-tuple",
        },
        "cite_format": {
            "enabled": True,
            "format": "5-tuple",
            "description": "Citation format: (document_id, version_id, node_id, span_offset, content_hash)",
        },
        "hash_algorithm": {
            "enabled": True,
            "algorithm": "sha256",
            "description": "SHA256 hash of cited content",
        },
        "version_tracking": {
            "enabled": True,
            "description": "Track citation validity across document versions",
        },
    },
}


# ==================== Extension 2: span-backref ====================

SPAN_BACKREF_EXTENSION: Dict[str, Any] = {
    "name": "mimir.span-backref",
    "version": ExtensionVersion.V1_0.value,
    "description": "Span-level back-reference to original content with position tracking",
    "spec_url": "https://mimir.example.com/docs/extensions/span-backref",
    "capabilities": {
        "span_verification": {
            "enabled": True,
            "description": "Verify span-level citations",
        },
        "position_tracking": {
            "enabled": True,
            "description": "Track character offset of cited span",
        },
        "content_snapshots": {
            "enabled": True,
            "description": "Store snapshots of cited content for change detection",
        },
        "retroactive_validation": {
            "enabled": True,
            "description": "Validate citations even after document updates",
        },
    },
}


# ==================== Extension Registry ====================

MIMIR_EXTENSIONS: List[Dict[str, Any]] = [
    CITATION_5TUPLE_EXTENSION,
    SPAN_BACKREF_EXTENSION,
]


def list_extensions() -> List[Dict[str, Any]]:
    """모든 Mimir extension 반환.
    
    Returns:
        Extension 리스트
    """
    return MIMIR_EXTENSIONS


def get_extension(extension_name: str) -> Dict[str, Any]:
    """특정 extension 조회.
    
    Args:
        extension_name: extension 이름
    
    Returns:
        Extension 딕셔너리
    """
    for ext in MIMIR_EXTENSIONS:
        if ext["name"] == extension_name:
            return ext
    raise ValueError(f"Unknown extension: {extension_name}")
```

### 4-3. Curated Tool Set 관리

**파일:** `/backend/app/mcp/curated_tools.py`

```python
"""Curated Tool Set 관리: 에이전트에 노출할 도구의 명시적 선택."""

from __future__ import annotations

from typing import Dict, List, Set, Optional
from enum import Enum
from uuid import UUID

from app.mcp.tool_schemas import TOOL_SCHEMAS_V1


class AuthRequirement(str, Enum):
    """도구 호출 시 필수 인증 요구사항."""
    
    OAUTH2_CLIENT_CREDENTIALS = "oauth2_client_credentials"
    SCOPE_PROFILE = "scope_profile"
    DELEGATION = "delegation"


class ToolPermission:
    """도구 권한 정의."""
    
    def __init__(
        self,
        tool_name: str,
        enabled: bool = True,
        auth_requirements: Optional[List[AuthRequirement]] = None,
        description: str = "",
    ):
        """도구 권한 초기화.
        
        Args:
            tool_name: 도구 이름
            enabled: 도구 활성화 여부
            auth_requirements: 필수 인증 요구사항 리스트
            description: 도구 설명
        """
        self.tool_name = tool_name
        self.enabled = enabled
        self.auth_requirements = auth_requirements or []
        self.description = description
    
    def to_dict(self) -> Dict[str, any]:
        """딕셔너리로 변환."""
        return {
            "tool_name": self.tool_name,
            "enabled": self.enabled,
            "auth_requirements": [r.value for r in self.auth_requirements],
            "description": self.description,
        }


class CuratedToolSet:
    """Curated Tool Set: 에이전트에 노출할 도구 목록 관리.
    
    모든 OpenAPI 엔드포인트를 자동으로 도구화하지 않음.
    명시적으로 선택된 도구만 에이전트에 노출한다.
    """
    
    def __init__(self):
        """Curated Tool Set 초기화 (기본값)."""
        self._tools: Dict[str, ToolPermission] = {}
        self._init_default_tools()
    
    def _init_default_tools(self) -> None:
        """기본 도구 세트 초기화 (Phase 4 수위)."""
        
        # 읽기 도구 3종: 모두 활성화
        self._tools["search_documents"] = ToolPermission(
            tool_name="search_documents",
            enabled=True,
            auth_requirements=[
                AuthRequirement.OAUTH2_CLIENT_CREDENTIALS,
                AuthRequirement.SCOPE_PROFILE,
            ],
            description="Search documents using FTS or vector search",
        )
        
        self._tools["fetch_node"] = ToolPermission(
            tool_name="fetch_node",
            enabled=True,
            auth_requirements=[
                AuthRequirement.OAUTH2_CLIENT_CREDENTIALS,
                AuthRequirement.SCOPE_PROFILE,
            ],
            description="Fetch full content of a specific node",
        )
        
        self._tools["verify_citation"] = ToolPermission(
            tool_name="verify_citation",
            enabled=True,
            auth_requirements=[
                AuthRequirement.OAUTH2_CLIENT_CREDENTIALS,
                AuthRequirement.SCOPE_PROFILE,
            ],
            description="Verify citation using 5-tuple",
        )
        
        # 쓰기 도구: 비활성화 (Phase 5 이후)
        self._tools["create_document"] = ToolPermission(
            tool_name="create_document",
            enabled=False,
            description="[Phase 5] Create a draft document",
        )
        
        self._tools["delete_document"] = ToolPermission(
            tool_name="delete_document",
            enabled=False,
            description="[Admin only] Delete a document",
        )
    
    def is_tool_enabled(self, tool_name: str) -> bool:
        """도구 활성화 여부 확인.
        
        Args:
            tool_name: 도구 이름
        
        Returns:
            활성화 여부
        """
        if tool_name not in self._tools:
            return False
        return self._tools[tool_name].enabled
    
    def get_auth_requirements(self, tool_name: str) -> List[AuthRequirement]:
        """도구의 인증 요구사항 조회.
        
        Args:
            tool_name: 도구 이름
        
        Returns:
            AuthRequirement 리스트
        """
        if tool_name not in self._tools:
            return []
        return self._tools[tool_name].auth_requirements
    
    def list_enabled_tools(self) -> List[str]:
        """활성화된 도구 목록 반환.
        
        Returns:
            도구 이름 리스트
        """
        return [
            name for name, perm in self._tools.items()
            if perm.enabled
        ]
    
    def list_all_tools(self) -> Dict[str, ToolPermission]:
        """모든 도구(활성/비활성) 반환.
        
        Returns:
            도구명 → ToolPermission 맵
        """
        return self._tools.copy()
    
    def enable_tool(self, tool_name: str) -> None:
        """도구 활성화.
        
        Args:
            tool_name: 도구 이름
        """
        if tool_name in self._tools:
            self._tools[tool_name].enabled = True
    
    def disable_tool(self, tool_name: str) -> None:
        """도구 비활성화.
        
        Args:
            tool_name: 도구 이름
        """
        if tool_name in self._tools:
            self._tools[tool_name].enabled = False
    
    def register_tool(
        self,
        tool_name: str,
        enabled: bool = True,
        auth_requirements: Optional[List[AuthRequirement]] = None,
        description: str = "",
    ) -> None:
        """새로운 도구 등록.
        
        Args:
            tool_name: 도구 이름
            enabled: 활성화 여부
            auth_requirements: 인증 요구사항
            description: 설명
        """
        self._tools[tool_name] = ToolPermission(
            tool_name=tool_name,
            enabled=enabled,
            auth_requirements=auth_requirements,
            description=description,
        )


# 전역 Curated Tool Set 인스턴스
_curated_tool_set: Optional[CuratedToolSet] = None


def get_curated_tool_set() -> CuratedToolSet:
    """전역 Curated Tool Set 인스턴스 반환.
    
    Returns:
        CuratedToolSet
    """
    global _curated_tool_set
    if _curated_tool_set is None:
        _curated_tool_set = CuratedToolSet()
    return _curated_tool_set
```

### 4-4. Tool Discovery 엔드포인트 및 인증 검증 미들웨어

**파일:** `/backend/app/routes/mcp_tools.py`

```python
"""MCP Tool Discovery 엔드포인트 및 인증 검증."""

from __future__ import annotations

import logging
from typing import Dict, List, Any, Optional
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, Header
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.auth import verify_oauth_token
from app.mcp.tool_schemas import get_tool_schema, list_all_tool_schemas
from app.mcp.curated_tools import get_curated_tool_set, AuthRequirement
from app.mcp.extensions import list_extensions
from app.mcp.schemas import AgentResponseSchema
from app.mcp.error_codes import MCPErrorCode

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/agent/mcp", tags=["mcp"])


# ==================== Tool Discovery ====================

@router.get("/tools/list")
async def list_available_tools(
    authorization: Optional[str] = Header(None),
    db: AsyncSession = Depends(get_db),
) -> Dict[str, Any]:
    """사용 가능한 도구 목록 조회 (Curated Tool Set).
    
    에이전트가 Mimir에서 호출 가능한 도구 목록을 조회한다.
    비활성화된 도구는 제외된다.
    
    Returns:
        {
          "tools": [
            {
              "name": "search_documents",
              "version": "1.0",
              "description": "...",
              "inputSchema": {...},
              "outputSchema": {...},
              "authRequirements": ["oauth2_client_credentials", "scope_profile"]
            },
            ...
          ],
          "totalCount": 3,
          "extensions": [...]
        }
    """
    try:
        # OAuth 토큰 검증 (선택사항: 공개 도구 목록)
        # 실무에서는 요청자가 agent 로그인했는지 확인 가능
        
        curated_tools = get_curated_tool_set()
        enabled_tool_names = curated_tools.list_enabled_tools()
        
        tools_response = []
        for tool_name in enabled_tool_names:
            try:
                schema = get_tool_schema(tool_name)
                auth_reqs = curated_tools.get_auth_requirements(tool_name)
                
                tool_info = {
                    **schema,
                    "authRequirements": [r.value for r in auth_reqs],
                }
                tools_response.append(tool_info)
            except ValueError as e:
                logger.warning(f"Failed to load schema for {tool_name}: {str(e)}")
        
        extensions = list_extensions()
        
        return {
            "tools": tools_response,
            "totalCount": len(tools_response),
            "extensions": extensions,
        }
    
    except Exception as exc:
        logger.error(f"Tool discovery error: {str(exc)}", exc_info=True)
        raise HTTPException(status_code=500, detail="Tool discovery failed")


@router.get("/tools/{tool_name}")
async def get_tool_details(
    tool_name: str,
    authorization: Optional[str] = Header(None),
    db: AsyncSession = Depends(get_db),
) -> Dict[str, Any]:
    """특정 도구의 상세 정보 조회.
    
    Args:
        tool_name: 도구 이름
    
    Returns:
        Tool schema 및 인증 요구사항
    """
    try:
        curated_tools = get_curated_tool_set()
        
        # 도구 활성화 여부 확인
        if not curated_tools.is_tool_enabled(tool_name):
            raise HTTPException(status_code=404, detail=f"Tool not available: {tool_name}")
        
        schema = get_tool_schema(tool_name)
        auth_reqs = curated_tools.get_auth_requirements(tool_name)
        
        return {
            **schema,
            "authRequirements": [r.value for r in auth_reqs],
        }
    
    except ValueError:
        raise HTTPException(status_code=404, detail=f"Tool not found: {tool_name}")
    except Exception as exc:
        logger.error(f"Tool details error: {str(exc)}", exc_info=True)
        raise HTTPException(status_code=500, detail="Failed to retrieve tool details")


# ==================== Tool 호출 전 인증 검증 ====================

async def verify_tool_auth(
    tool_name: str,
    authorization: Optional[str],
    db: AsyncSession,
) -> Dict[str, Any]:
    """도구 호출 전 인증 검증.
    
    Args:
        tool_name: 도구 이름
        authorization: Authorization 헤더
        db: DB 세션
    
    Returns:
        OAuth 정보 (agent_id, 권한 등)
    
    Raises:
        HTTPException: 인증 실패
    """
    curated_tools = get_curated_tool_set()
    
    # 도구 활성화 여부 확인
    if not curated_tools.is_tool_enabled(tool_name):
        raise HTTPException(
            status_code=404,
            detail=f"Tool not available: {tool_name}"
        )
    
    auth_requirements = curated_tools.get_auth_requirements(tool_name)
    
    # OAUTH2_CLIENT_CREDENTIALS 필수
    if AuthRequirement.OAUTH2_CLIENT_CREDENTIALS not in auth_requirements:
        logger.warning(f"Tool {tool_name} does not require OAuth")
        return {}
    
    if not authorization:
        raise HTTPException(
            status_code=401,
            detail="Missing Authorization header"
        )
    
    try:
        parts = authorization.split(" ")
        if len(parts) != 2 or parts[0].lower() != "bearer":
            raise ValueError("Invalid Authorization header format")
        
        token = parts[1]
        oauth_info = await verify_oauth_token(token, db)
        
        if not oauth_info:
            raise ValueError("Invalid or expired token")
        
        return oauth_info
    
    except ValueError as e:
        raise HTTPException(
            status_code=401,
            detail=f"OAuth authentication failed: {str(e)}"
        )


# ==================== Extensions 목록 ====================

@router.get("/extensions")
async def list_mimir_extensions() -> Dict[str, Any]:
    """Mimir MCP Extensions 목록.
    
    Returns:
        {
          "extensions": [
            {
              "name": "mimir.citation-5tuple",
              "version": "1.0",
              "description": "...",
              "capabilities": {...}
            },
            ...
          ]
        }
    """
    extensions = list_extensions()
    return {
        "extensions": extensions,
        "totalCount": len(extensions),
    }
```

### 4-5. 단위 테스트

**파일:** `/backend/tests/unit/mcp/test_tool_schema.py`

```python
"""Tool Schema 및 Curated Tool Set 단위 테스트."""

import pytest
from uuid import uuid4

from app.mcp.tool_schemas import (
    get_tool_schema,
    SEARCH_DOCUMENTS_SCHEMA,
    FETCH_NODE_SCHEMA,
    VERIFY_CITATION_SCHEMA,
)
from app.mcp.curated_tools import (
    CuratedToolSet,
    ToolPermission,
    AuthRequirement,
    get_curated_tool_set,
)
from app.mcp.extensions import list_extensions, get_extension


class TestToolSchemas:
    """Tool Schema 테스트."""
    
    def test_search_documents_schema_structure(self):
        """search_documents schema 구조 검증."""
        schema = get_tool_schema("search_documents")
        
        assert schema["name"] == "search_documents"
        assert "inputSchema" in schema
        assert "outputSchema" in schema
        assert schema["inputSchema"]["required"] == ["query"]
    
    def test_search_documents_input_validation(self):
        """search_documents 입력 검증."""
        schema = get_tool_schema("search_documents")
        input_schema = schema["inputSchema"]
        
        # query 필수
        assert "query" in input_schema["required"]
        
        # scope 선택사항
        assert "scope" in input_schema["properties"]
        
        # top_k 범위 확인
        top_k_prop = input_schema["properties"]["top_k"]
        assert top_k_prop["minimum"] == 1
        assert top_k_prop["maximum"] == 50
    
    def test_fetch_node_schema_structure(self):
        """fetch_node schema 구조 검증."""
        schema = get_tool_schema("fetch_node")
        
        assert schema["name"] == "fetch_node"
        assert set(schema["inputSchema"]["required"]) == {"document_id", "node_id"}
    
    def test_verify_citation_schema_structure(self):
        """verify_citation schema 구조 검증."""
        schema = get_tool_schema("verify_citation")
        
        assert schema["name"] == "verify_citation"
        required = set(schema["inputSchema"]["required"])
        assert required == {"document_id", "version_id", "node_id", "content_hash"}
    
    def test_output_schema_citations(self):
        """search_documents 출력 schema에 citation 포함."""
        schema = get_tool_schema("search_documents")
        output_props = schema["outputSchema"]["properties"]
        
        # results 배열에 citation 포함
        result_item = output_props["results"]["items"]
        assert "citation" in result_item["properties"]
        
        citation_props = result_item["properties"]["citation"]
        required_citation_fields = {
            "document_id", "version_id", "node_id", "content_hash"
        }
        assert set(citation_props["required"]) == required_citation_fields


class TestCuratedToolSet:
    """CuratedToolSet 테스트."""
    
    def test_default_tools_initialization(self):
        """기본 도구 초기화."""
        tool_set = CuratedToolSet()
        enabled_tools = tool_set.list_enabled_tools()
        
        assert "search_documents" in enabled_tools
        assert "fetch_node" in enabled_tools
        assert "verify_citation" in enabled_tools
        assert "create_document" not in enabled_tools  # Phase 5
        assert "delete_document" not in enabled_tools  # Admin only
    
    def test_tool_enable_disable(self):
        """도구 활성화/비활성화."""
        tool_set = CuratedToolSet()
        
        # 기본값: create_document 비활성화
        assert not tool_set.is_tool_enabled("create_document")
        
        # 활성화
        tool_set.enable_tool("create_document")
        assert tool_set.is_tool_enabled("create_document")
        
        # 비활성화
        tool_set.disable_tool("create_document")
        assert not tool_set.is_tool_enabled("create_document")
    
    def test_auth_requirements(self):
        """도구별 인증 요구사항."""
        tool_set = CuratedToolSet()
        
        # search_documents: OAuth + Scope Profile 필수
        search_auth = tool_set.get_auth_requirements("search_documents")
        assert AuthRequirement.OAUTH2_CLIENT_CREDENTIALS in search_auth
        assert AuthRequirement.SCOPE_PROFILE in search_auth
    
    def test_register_custom_tool(self):
        """커스텀 도구 등록."""
        tool_set = CuratedToolSet()
        
        tool_set.register_tool(
            tool_name="custom_tool",
            enabled=True,
            auth_requirements=[AuthRequirement.OAUTH2_CLIENT_CREDENTIALS],
            description="Custom tool for testing",
        )
        
        assert tool_set.is_tool_enabled("custom_tool")
        auth_reqs = tool_set.get_auth_requirements("custom_tool")
        assert AuthRequirement.OAUTH2_CLIENT_CREDENTIALS in auth_reqs


class TestMimirExtensions:
    """Mimir Extensions 테스트."""
    
    def test_list_extensions(self):
        """Extension 목록 조회."""
        extensions = list_extensions()
        
        assert len(extensions) >= 2
        ext_names = [e["name"] for e in extensions]
        assert "mimir.citation-5tuple" in ext_names
        assert "mimir.span-backref" in ext_names
    
    def test_citation_5tuple_extension(self):
        """citation-5tuple extension 검증."""
        ext = get_extension("mimir.citation-5tuple")
        
        assert ext["version"] == "1.0"
        assert "capabilities" in ext
        assert ext["capabilities"]["verify_citation"]["enabled"] is True
    
    def test_span_backref_extension(self):
        """span-backref extension 검증."""
        ext = get_extension("mimir.span-backref")
        
        assert ext["version"] == "1.0"
        assert "capabilities" in ext
        assert ext["capabilities"]["position_tracking"]["enabled"] is True


class TestGlobalCuratedToolSet:
    """전역 CuratedToolSet 인스턴스 테스트."""
    
    def test_singleton_pattern(self):
        """싱글톤 패턴 확인."""
        tool_set_1 = get_curated_tool_set()
        tool_set_2 = get_curated_tool_set()
        
        assert tool_set_1 is tool_set_2
    
    def test_persistent_state(self):
        """상태 지속성."""
        tool_set = get_curated_tool_set()
        
        # 도구 활성화
        tool_set.enable_tool("create_document")
        
        # 다시 조회
        tool_set_2 = get_curated_tool_set()
        assert tool_set_2.is_tool_enabled("create_document")
```

## 5. 산출물

1. **Tool Schema 정의** (`/backend/app/mcp/tool_schemas.py`)
   - 3종 도구의 JSON Schema (search_documents, fetch_node, verify_citation)
   - 입출력 스키마 포함

2. **Mimir Extension 선언** (`/backend/app/mcp/extensions.py`)
   - mimir.citation-5tuple v1.0
   - mimir.span-backref v1.0

3. **Curated Tool Set 관리** (`/backend/app/mcp/curated_tools.py`)
   - CuratedToolSet 클래스
   - ToolPermission, AuthRequirement Enum
   - 전역 인스턴스 관리

4. **Tool Discovery API** (`/backend/app/routes/mcp_tools.py`)
   - GET /api/agent/mcp/tools/list: 사용 가능 도구 목록
   - GET /api/agent/mcp/tools/{tool_name}: 도구 상세 정보
   - GET /api/agent/mcp/extensions: Extension 목록
   - verify_tool_auth() 함수: 인증 검증

5. **단위 테스트** (`/backend/tests/unit/mcp/test_tool_schema.py`)
   - Schema 구조 검증
   - CuratedToolSet 로직 검증
   - Extension 호환성 검증

## 6. 완료 기준

1. 3가지 Tool Schema가 정의되었는가?
   - search_documents, fetch_node, verify_citation

2. 각 Tool Schema의 inputSchema, outputSchema가 명확한가?
   - 필수 필드, 타입, 제약 조건 포함

3. Mimir Extension 2개가 선언되었는가?
   - mimir.citation-5tuple, mimir.span-backref

4. CuratedToolSet이 읽기 도구 3개를 활성화했는가?
   - 쓰기 도구는 비활성화

5. 도구별 인증 요구사항이 명시되었는가?
   - oauth2_client_credentials, scope_profile 필수

6. Tool Discovery 엔드포인트가 구현되었는가?
   - 활성화된 도구만 반환

7. 인증 검증 미들웨어가 구현되었는가?
   - 도구 호출 전 권한 확인

8. 모든 단위 테스트가 통과하는가?

9. mypy 타입 검사를 통과하는가?

## 7. 작업 지침

### 지침 7-1. Tool Schema 정확성

OpenAPI 스펙을 참고하되, MCP Tool schema 형식에 맞게 수동으로 큐레이션한다. 불필요한 필드는 제거하고, 에이전트가 이해하기 쉽도록 설명을 명확히 작성한다.

### 지침 7-2. inputSchema 필수 필드

각 도구의 inputSchema에는 필수 필드(required 배열)를 명확히 정의한다:
- search_documents: query 필수
- fetch_node: document_id, node_id 필수
- verify_citation: document_id, version_id, node_id, content_hash 필수

### 지침 7-3. Citation 구조 표준화

모든 search/fetch 응답에 포함되는 citation은 5-tuple 구조를 따른다:
- document_id, version_id, node_id, span_offset, content_hash

### 지침 7-4. Extension 버전 관리

Extension 버전은 semantic versioning을 따른다. 하위호환성이 깨지는 변경이 없는 한 v1.x 를 유지한다.

### 지침 7-5. Curated Tool Set의 명시성

모든 도구를 자동으로 노출하지 말고, 명시적으로 선택된 도구만 CuratedToolSet에 등록한다. 새로운 도구 추가 시 register_tool()을 호출하여 진행한다.

### 지침 7-6. 인증 요구사항 명시

각 도구가 요구하는 인증 레벨을 명시한다:
- oauth2_client_credentials: OAuth 토큰 필수
- scope_profile: Scope Profile 바인딩 필수
- delegation: 사용자 대행 권한 필수

### 지침 7-7. Tool Discovery 공개성

Tool Discovery 엔드포인트는 인증을 요구하지 않거나 낮은 수준의 인증만 요구한다. 에이전트가 Mimir의 능력을 미리 파악할 수 있어야 한다.

### 지침 7-8. S2 원칙 ⑤ 반영

모든 Tool Schema에 에이전트와 사용자의 동등한 접근을 보장한다. 에이전트도 동일한 도구와 정보에 접근 가능해야 한다.

### 지침 7-9. Phase 5 예약

create_document, delete_document 등 쓰기 도구는 현재 비활성화하고, Phase 5에서 활성화할 예정임을 문서화한다.

### 지침 7-10. 보안: FilterExpression 인젝션 방어

Tool Discovery 및 도구 호출 시 FilterExpression(Scope Profile의 조건식)이 유효한 형식인지 검증한다. 악의적인 필터 표현식이 파서를 공격하지 못하도록 한다.
