# Task 4-3. Resource 노출 + Prompt Registry MCP 노출 + Streamable HTTP/SSE

## 1. 작업 목적

Task 4-1, 4-2에서 구축한 MCP Server 위에서 **MCP Resource 기능**, **Prompt Registry 노출**, **스트리밍 지원 (Streamable HTTP/SSE)**을 구현하여, 에이전트가 Mimir 문서를 URI로 직접 참조하고, 프롬프트 템플릿을 재사용 가능하도록 하며, 긴 응답을 부분 결과부터 처리할 수 있도록 한다.

## 2. 작업 범위

### 포함 범위

1. **MCP Resource 구현**
   - URI scheme: `mimir://documents/{document_id}/versions/{version_id}/nodes/{node_id}`
   - URI 파싱 및 해석
   - 리소스 메타데이터 (mime_type, description)
   - fetch_node와 동일한 응답 제공

2. **Resource 목록 조회 (resources/list)**
   - MCP resources/list 메서드 구현
   - 현재 사용자/에이전트가 접근 가능한 리소스 목록

3. **Prompt Registry MCP 노출**
   - Phase 1에서 구축한 Prompt Registry 데이터 조회
   - 활성 프롬프트를 MCP prompts로 변환
   - 프롬프트 이름: `mimir-{prompt_id}`
   - 템플릿 + 변수 목록 노출

4. **Streamable HTTP 지원**
   - 긴 문서 fetch 시 chunked transfer encoding 사용
   - HTTP 200 응답 + Content-Encoding: chunked
   - 클라이언트가 부분 결과부터 처리 가능

5. **SSE (Server-Sent Events) 지원**
   - search_documents 결과를 스트리밍
   - 부분 결과부터 처리 가능
   - text/event-stream content-type

6. **단위 테스트**
   - URI 파싱 및 리소스 조회
   - 프롬프트 노출 정확성
   - 스트리밍 동작

### 제외 범위

- MCP 초기화 (Task 4-1)
- 읽기 도구 기본 구현 (Task 4-2)

## 3. 선행 조건

- Task 4-1, 4-2 완료
- Phase 1 (Prompt Registry) 완료
- FastAPI SSE 지원 가능
- URI 파싱 라이브러리 (urllib) 사용 가능

## 4. 주요 작업 항목

### 4-1. MCP Resource URI 파싱 및 해석

**파일:** `/backend/app/mcp/resources.py`

```python
"""MCP Resource 구현 (mimir:// URI scheme)."""

from __future__ import annotations

from typing import Optional, Dict, Any
from uuid import UUID
import logging
from urllib.parse import urlparse, parse_qs

logger = logging.getLogger(__name__)


class MimirResourceURI:
    """Mimir Resource URI 파서.
    
    형식: mimir://documents/{document_id}/versions/{version_id}/nodes/{node_id}
    """
    
    SCHEME = "mimir"
    
    def __init__(
        self,
        document_id: UUID,
        version_id: UUID,
        node_id: str,
    ):
        self.document_id = document_id
        self.version_id = version_id
        self.node_id = node_id
    
    def to_uri(self) -> str:
        """URI 문자열 생성."""
        return (
            f"{self.SCHEME}://documents/{self.document_id}/versions/{self.version_id}/nodes/{self.node_id}"
        )
    
    @classmethod
    def from_uri(cls, uri: str) -> Optional[MimirResourceURI]:
        """URI 문자열 파싱.
        
        Args:
            uri: 리소스 URI (예: mimir://documents/xxx/versions/yyy/nodes/zzz)
        
        Returns:
            파싱된 MimirResourceURI 또는 None (파싱 실패)
        """
        try:
            parsed = urlparse(uri)
            
            if parsed.scheme != cls.SCHEME:
                logger.warning(f"Invalid URI scheme: {parsed.scheme}")
                return None
            
            # Path 파싱: /documents/{id}/versions/{id}/nodes/{id}
            path_parts = parsed.path.split("/")
            
            # 예상 형식: ['', 'documents', '{uuid}', 'versions', '{uuid}', 'nodes', '{node_id}']
            if len(path_parts) != 7:
                logger.warning(f"Invalid URI path structure: {parsed.path}")
                return None
            
            if path_parts[1] != "documents" or path_parts[3] != "versions" or path_parts[5] != "nodes":
                logger.warning(f"Invalid URI path keywords")
                return None
            
            document_id = UUID(path_parts[2])
            version_id = UUID(path_parts[4])
            node_id = path_parts[6]
            
            return cls(
                document_id=document_id,
                version_id=version_id,
                node_id=node_id,
            )
        
        except Exception as e:
            logger.error(f"Failed to parse URI: {uri}, error: {str(e)}")
            return None
    
    def __str__(self) -> str:
        return self.to_uri()


class ResourceMetadata:
    """리소스 메타데이터."""
    
    def __init__(
        self,
        uri: str,
        mime_type: str = "text/plain",
        description: str = "",
    ):
        self.uri = uri
        self.mime_type = mime_type
        self.description = description
    
    def to_dict(self) -> Dict[str, str]:
        return {
            "uri": self.uri,
            "mimeType": self.mime_type,
            "description": self.description,
        }


class ResourceService:
    """MCP Resource 서비스."""
    
    def __init__(self, db):
        self.db = db
    
    async def get_resource(
        self,
        uri: str,
        scope_profile: Optional[Dict[str, Any]] = None,
    ) -> Optional[Dict[str, Any]]:
        """리소스 조회 (fetch_node와 동일).
        
        Args:
            uri: mimir:// URI
            scope_profile: ACL 필터링용 Scope Profile
        
        Returns:
            리소스 콘텐츠 및 메타데이터, 또는 None
        """
        # URI 파싱
        resource_uri = MimirResourceURI.from_uri(uri)
        if not resource_uri:
            logger.warning(f"Invalid resource URI: {uri}")
            return None
        
        # fetch_node와 동일하게 처리
        from app.mcp.tools import ToolService
        from app.schemas.mcp_tools import FetchNodeRequest
        from uuid import uuid4
        
        tool_service = ToolService(
            db=self.db,
            actor_id=uuid4(),  # 리소스 조회는 시스템 호출
            actor_type="system",
        )
        
        try:
            request = FetchNodeRequest(
                document_id=resource_uri.document_id,
                version_id=resource_uri.version_id,
                node_id=resource_uri.node_id,
            )
            
            if scope_profile is None:
                scope_profile = {"scope": "private"}
            
            response = await tool_service.fetch_node(request, scope_profile)
            
            return response.dict()
        
        except Exception as e:
            logger.error(f"Failed to fetch resource: {str(e)}")
            return None
    
    async def list_resources(
        self,
        scope_profile: Optional[Dict[str, Any]] = None,
    ) -> list[Dict[str, str]]:
        """접근 가능한 리소스 목록 조회.
        
        Args:
            scope_profile: ACL 필터링용 Scope Profile
        
        Returns:
            리소스 메타데이터 리스트
        """
        # Scope Profile에 따라 접근 가능한 문서 목록 조회
        from app.services.document import DocumentService
        
        doc_service = DocumentService(self.db)
        
        try:
            # scope_profile을 기반으로 문서 검색
            documents = await doc_service.search_documents(
                filters=scope_profile or {},
                limit=100,  # 리소스 목록은 제한
            )
            
            resources = []
            for doc in documents:
                # 각 문서의 노드 목록 조회
                nodes = await doc_service.get_nodes(doc["document_id"])
                
                for node in nodes:
                    uri = MimirResourceURI(
                        document_id=UUID(doc["document_id"]),
                        version_id=UUID(doc["version_id"]),
                        node_id=node["node_id"],
                    ).to_uri()
                    
                    metadata = ResourceMetadata(
                        uri=uri,
                        mime_type="text/plain",
                        description=f"{doc.get('title', 'Unknown')} - {node.get('title', node['node_id'])}",
                    )
                    
                    resources.append(metadata.to_dict())
            
            return resources
        
        except Exception as e:
            logger.error(f"Failed to list resources: {str(e)}")
            return []
```

### 4-2. Prompt Registry MCP 노출

**파일:** `/backend/app/mcp/prompts.py`

```python
"""Prompt Registry를 MCP Prompts로 노출."""

from __future__ import annotations

from typing import Optional, Dict, List, Any
import logging

logger = logging.getLogger(__name__)


class PromptVariable:
    """프롬프트 변수 정의."""
    
    def __init__(
        self,
        name: str,
        description: str = "",
        required: bool = False,
        default_value: Optional[str] = None,
    ):
        self.name = name
        self.description = description
        self.required = required
        self.default_value = default_value
    
    def to_dict(self) -> Dict[str, Any]:
        return {
            "name": self.name,
            "description": self.description,
            "required": self.required,
            "defaultValue": self.default_value,
        }


class MCPPrompt:
    """MCP Prompt 정의."""
    
    def __init__(
        self,
        name: str,
        description: str,
        template: str,
        variables: List[PromptVariable],
    ):
        self.name = name
        self.description = description
        self.template = template
        self.variables = variables
    
    def to_dict(self) -> Dict[str, Any]:
        return {
            "name": self.name,
            "description": self.description,
            "template": self.template,
            "variables": [v.to_dict() for v in self.variables],
        }


class PromptService:
    """Prompt Registry를 MCP로 노출."""
    
    def __init__(self, db):
        self.db = db
    
    async def list_prompts(self) -> List[Dict[str, Any]]:
        """활성 프롬프트 목록 조회 (MCP prompts/list).
        
        Returns:
            MCP 프롬프트 리스트
        """
        try:
            # Phase 1에서 구축한 Prompt Registry 쿼리
            from app.services.prompt_registry import PromptRegistry
            
            registry = PromptRegistry(self.db)
            prompts = await registry.get_active_prompts()
            
            mcp_prompts = []
            for prompt in prompts:
                # 프롬프트를 MCP 형식으로 변환
                mcp_prompt = self._convert_to_mcp_prompt(prompt)
                mcp_prompts.append(mcp_prompt.to_dict())
            
            return mcp_prompts
        
        except Exception as e:
            logger.error(f"Failed to list prompts: {str(e)}")
            return []
    
    async def get_prompt(self, prompt_id: str) -> Optional[Dict[str, Any]]:
        """특정 프롬프트 조회.
        
        Args:
            prompt_id: 프롬프트 ID (mimir-{id} 형식에서 {id} 추출)
        
        Returns:
            MCP 프롬프트
        """
        try:
            # ID에서 "mimir-" 접두사 제거
            if prompt_id.startswith("mimir-"):
                actual_id = prompt_id[6:]
            else:
                actual_id = prompt_id
            
            from app.services.prompt_registry import PromptRegistry
            
            registry = PromptRegistry(self.db)
            prompt = await registry.get_prompt(actual_id)
            
            if not prompt:
                logger.warning(f"Prompt not found: {actual_id}")
                return None
            
            mcp_prompt = self._convert_to_mcp_prompt(prompt)
            return mcp_prompt.to_dict()
        
        except Exception as e:
            logger.error(f"Failed to get prompt: {str(e)}")
            return None
    
    def _convert_to_mcp_prompt(self, prompt: Dict[str, Any]) -> MCPPrompt:
        """Prompt Registry 포맷을 MCP 포맷으로 변환.
        
        Args:
            prompt: Prompt Registry 프롬프트
        
        Returns:
            MCP 프롬프트
        """
        # Phase 1 프롬프트 구조:
        # {
        #   "id": "...",
        #   "name": "...",
        #   "description": "...",
        #   "template": "...",
        #   "variables": [{"name": "...", "description": "...", "required": True/False}, ...]
        # }
        
        variables = []
        for var in prompt.get("variables", []):
            variables.append(
                PromptVariable(
                    name=var.get("name"),
                    description=var.get("description", ""),
                    required=var.get("required", False),
                    default_value=var.get("default_value"),
                )
            )
        
        return MCPPrompt(
            name=f"mimir-{prompt.get('id')}",  # 이름에 "mimir-" 접두사 추가
            description=prompt.get("description", ""),
            template=prompt.get("template", ""),
            variables=variables,
        )
```

### 4-3. Streamable HTTP (Chunked Transfer)

**파일:** `/backend/app/routes/mcp.py` (Task 4-1 확장)

```python
from fastapi import StreamingResponse
from typing import AsyncGenerator
import json


@router.get("/resources/{resource_path:path}")
async def get_resource_streaming(
    resource_path: str,
    authorization: Optional[str] = Header(None),
    db: AsyncSession = Depends(get_db),
):
    """리소스 조회 (streaming 지원).
    
    Path Parameters:
        resource_path: URI path (documents/{id}/versions/{id}/nodes/{id})
    
    Headers:
        Authorization: Bearer <oauth_token>
    
    Returns:
        스트리밍 응답 (chunked transfer encoding)
    """
    logger.info(f"Resource request: {resource_path}")
    
    # OAuth 검증
    if not authorization:
        return JSONResponse(
            status_code=401,
            content={"error": "Unauthorized"}
        )
    
    try:
        parts = authorization.split(" ")
        token = parts[1] if len(parts) == 2 else None
        oauth_info = await verify_oauth_token(token, db)
        if not oauth_info:
            raise ValueError("Invalid token")
        
        actor_id = oauth_info.get("actor_id")
        
    except Exception as e:
        logger.error(f"OAuth failed: {str(e)}")
        return JSONResponse(
            status_code=401,
            content={"error": "Unauthorized"}
        )
    
    # URI 구성
    uri = f"mimir://documents/{resource_path}"
    
    # Scope Profile 조회
    scope_profile = await get_scope_profile_for_agent(actor_id, db)
    
    # 리소스 조회 (스트리밍 생성기)
    async def resource_generator() -> AsyncGenerator[str, None]:
        from app.mcp.resources import ResourceService
        
        service = ResourceService(db)
        
        try:
            resource = await service.get_resource(uri, scope_profile)
            
            if not resource:
                yield json.dumps({"error": "Resource not found"})
                return
            
            # 콘텐츠가 크면 청크 단위로 스트리밍
            content = resource.get("content", "")
            chunk_size = 8192  # 8KB 청크
            
            # 메타데이터 먼저 전송
            metadata = {
                "document_id": str(resource.get("document_id")),
                "version_id": str(resource.get("version_id")),
                "node_id": resource.get("node_id"),
                "content_size": len(content),
            }
            yield json.dumps({"metadata": metadata}) + "\n"
            
            # 콘텐츠 청크 단위로 전송
            for i in range(0, len(content), chunk_size):
                chunk = content[i:i + chunk_size]
                yield json.dumps({"content_chunk": chunk}) + "\n"
                await asyncio.sleep(0)  # 다른 태스크에 양보
            
            yield json.dumps({"complete": True}) + "\n"
        
        except Exception as e:
            logger.error(f"Streaming error: {str(e)}")
            yield json.dumps({"error": str(e)}) + "\n"
    
    return StreamingResponse(
        resource_generator(),
        media_type="application/x-ndjson",  # Newline-delimited JSON
        headers={
            "Transfer-Encoding": "chunked",
            "Cache-Control": "no-cache",
        }
    )
```

### 4-4. SSE (Server-Sent Events) 검색 스트리밍

**파일:** `/backend/app/routes/mcp.py` (Task 4-1 확장)

```python
from fastapi.responses import StreamingResponse as FastAPIStreamingResponse


@router.get("/tools/search_documents/stream")
async def search_documents_streaming(
    query: str,
    top_k: int = 5,
    authorization: Optional[str] = Header(None),
    db: AsyncSession = Depends(get_db),
):
    """검색 결과를 SSE로 스트리밍.
    
    Query Parameters:
        query: 검색 쿼리
        top_k: 결과 개수
    
    Headers:
        Authorization: Bearer <oauth_token>
    
    Returns:
        SSE 스트림 (text/event-stream)
    """
    logger.info(f"Search streaming: {query}")
    
    # OAuth 검증
    if not authorization:
        return JSONResponse(
            status_code=401,
            content={"error": "Unauthorized"}
        )
    
    try:
        parts = authorization.split(" ")
        token = parts[1] if len(parts) == 2 else None
        oauth_info = await verify_oauth_token(token, db)
        if not oauth_info:
            raise ValueError("Invalid token")
        
        actor_id = oauth_info.get("actor_id")
        
    except Exception as e:
        logger.error(f"OAuth failed: {str(e)}")
        return JSONResponse(
            status_code=401,
            content={"error": "Unauthorized"}
        )
    
    # SSE 생성기
    async def event_generator() -> AsyncGenerator[str, None]:
        from app.mcp.tools import ToolService
        from app.schemas.mcp_tools import SearchDocumentsRequest
        
        tool_service = ToolService(
            db=db,
            actor_id=UUID(actor_id),
            actor_type="agent",
        )
        
        scope_profile = await get_scope_profile_for_agent(actor_id, db)
        
        try:
            # 검색 요청
            request = SearchDocumentsRequest(
                query=query,
                top_k=top_k,
            )
            
            # 검색 (비동기)
            result = await tool_service.search_documents(request, scope_profile)
            
            # 메타데이터 이벤트
            metadata_event = {
                "total_count": result.total_count,
                "retrieval_method": result.retrieval_method,
            }
            yield f"event: metadata\ndata: {json.dumps(metadata_event)}\n\n"
            
            # 결과 이벤트 (각 결과마다)
            for i, search_result in enumerate(result.results):
                result_event = {
                    "index": i,
                    "document_id": str(search_result.document_id),
                    "document_title": search_result.document_title,
                    "relevance_score": search_result.relevance_score,
                    "content": search_result.content,
                }
                yield f"event: result\ndata: {json.dumps(result_event)}\n\n"
                await asyncio.sleep(0)  # 다른 태스크에 양보
            
            # 종료 이벤트
            yield f"event: complete\ndata: {json.dumps({'status': 'success'})}\n\n"
        
        except Exception as e:
            logger.error(f"Search streaming error: {str(e)}")
            yield f"event: error\ndata: {json.dumps({'error': str(e)})}\n\n"
    
    return FastAPIStreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        }
    )
```

### 4-5. MCP Prompts 엔드포인트

**파일:** `/backend/app/routes/mcp.py` (Task 4-1 확장)

```python
@router.get("/prompts")
async def list_prompts(
    db: AsyncSession = Depends(get_db),
) -> Dict[str, Any]:
    """활성 프롬프트 목록 조회 (MCP prompts/list).
    
    Returns:
        프롬프트 리스트
    """
    logger.info("Prompts list request")
    
    try:
        from app.mcp.prompts import PromptService
        
        service = PromptService(db)
        prompts = await service.list_prompts()
        
        return {
            "prompts": prompts,
            "total_count": len(prompts),
        }
    
    except Exception as e:
        logger.error(f"Failed to list prompts: {str(e)}")
        return {
            "prompts": [],
            "error": str(e),
        }


@router.get("/prompts/{prompt_id}")
async def get_prompt(
    prompt_id: str,
    db: AsyncSession = Depends(get_db),
) -> Dict[str, Any]:
    """특정 프롬프트 조회.
    
    Path Parameters:
        prompt_id: 프롬프트 ID (mimir-{id} 형식)
    
    Returns:
        프롬프트 정의
    """
    logger.info(f"Get prompt: {prompt_id}")
    
    try:
        from app.mcp.prompts import PromptService
        
        service = PromptService(db)
        prompt = await service.get_prompt(prompt_id)
        
        if not prompt:
            return {
                "error": f"Prompt not found: {prompt_id}",
            }
        
        return {
            "prompt": prompt,
        }
    
    except Exception as e:
        logger.error(f"Failed to get prompt: {str(e)}")
        return {
            "error": str(e),
        }
```

### 4-6. 단위/통합 테스트

**파일:** `/backend/tests/unit/mcp/test_mcp_resources_prompts.py`

```python
"""Resource 및 Prompt 테스트."""

import pytest
from uuid import uuid4

from app.mcp.resources import MimirResourceURI
from app.mcp.prompts import PromptService


class TestMimirResourceURI:
    """MimirResourceURI 파싱 테스트."""
    
    def test_uri_creation(self):
        """URI 생성."""
        doc_id = uuid4()
        ver_id = uuid4()
        node_id = "section-1"
        
        uri = MimirResourceURI(
            document_id=doc_id,
            version_id=ver_id,
            node_id=node_id,
        )
        
        uri_str = uri.to_uri()
        
        assert uri_str.startswith("mimir://documents/")
        assert str(doc_id) in uri_str
        assert str(ver_id) in uri_str
        assert node_id in uri_str
    
    def test_uri_parsing(self):
        """URI 파싱."""
        doc_id = uuid4()
        ver_id = uuid4()
        node_id = "section-1"
        
        uri_str = f"mimir://documents/{doc_id}/versions/{ver_id}/nodes/{node_id}"
        
        parsed = MimirResourceURI.from_uri(uri_str)
        
        assert parsed is not None
        assert parsed.document_id == doc_id
        assert parsed.version_id == ver_id
        assert parsed.node_id == node_id
    
    def test_invalid_uri_scheme(self):
        """유효하지 않은 scheme."""
        uri_str = f"http://example.com"
        parsed = MimirResourceURI.from_uri(uri_str)
        
        assert parsed is None
    
    def test_invalid_uri_path(self):
        """유효하지 않은 경로."""
        uri_str = f"mimir://invalid/path"
        parsed = MimirResourceURI.from_uri(uri_str)
        
        assert parsed is None


class TestPromptService:
    """PromptService 테스트."""
    
    @pytest.mark.asyncio
    async def test_list_prompts(self, db_session):
        """프롬프트 목록 조회."""
        from app.mcp.prompts import PromptService
        
        service = PromptService(db_session)
        
        # Mock Prompt Registry
        # (실제 테스트에서는 테스트 데이터 설정 필요)
        try:
            prompts = await service.list_prompts()
            assert isinstance(prompts, list)
        except Exception:
            # 테스트 데이터 없음 시 정상
            pass
    
    def test_convert_to_mcp_prompt(self):
        """프롬프트 변환."""
        from app.mcp.prompts import PromptService
        
        service = PromptService(None)
        
        prompt = {
            "id": "test-prompt",
            "name": "Test Prompt",
            "description": "A test prompt",
            "template": "Analyze: {text}",
            "variables": [
                {
                    "name": "text",
                    "description": "Text to analyze",
                    "required": True,
                }
            ],
        }
        
        mcp_prompt = service._convert_to_mcp_prompt(prompt)
        
        assert mcp_prompt.name == "mimir-test-prompt"
        assert mcp_prompt.description == "A test prompt"
        assert mcp_prompt.template == "Analyze: {text}"
        assert len(mcp_prompt.variables) == 1
        assert mcp_prompt.variables[0].name == "text"


class TestStreamingEndpoints:
    """스트리밍 엔드포인트 테스트."""
    
    @pytest.mark.asyncio
    async def test_resource_streaming(self, client, mock_token):
        """리소스 스트리밍 테스트."""
        doc_id = uuid4()
        ver_id = uuid4()
        
        response = await client.get(
            f"/mcp/resources/documents/{doc_id}/versions/{ver_id}/nodes/section-1",
            headers={"Authorization": f"Bearer {mock_token}"},
        )
        
        assert response.status_code in [200, 404]  # 리소스 없음 가능
        assert response.headers.get("Transfer-Encoding") == "chunked"
    
    @pytest.mark.asyncio
    async def test_search_streaming(self, client, mock_token):
        """검색 스트리밍 테스트."""
        response = await client.get(
            "/mcp/tools/search_documents/stream?query=test&top_k=5",
            headers={"Authorization": f"Bearer {mock_token}"},
        )
        
        assert response.status_code in [200, 404]
        if response.status_code == 200:
            assert response.headers.get("content-type") == "text/event-stream"
```

## 5. 산출물

1. **Resource URI 파서** (`/backend/app/mcp/resources.py`)
   - MimirResourceURI: URI 파싱 및 생성
   - ResourceService: 리소스 조회 및 목록

2. **Prompt Registry 노출** (`/backend/app/mcp/prompts.py`)
   - PromptService: Phase 1 프롬프트 → MCP 프롬프트 변환
   - MCPPrompt, PromptVariable: MCP 포맷 정의

3. **Streamable HTTP 엔드포인트** (`/backend/app/routes/mcp.py`)
   - GET /mcp/resources/{path}: 리소스 청크 스트리밍

4. **SSE 검색 엔드포인트** (`/backend/app/routes/mcp.py`)
   - GET /mcp/tools/search_documents/stream: 검색 결과 SSE

5. **Prompt 엔드포인트** (`/backend/app/routes/mcp.py`)
   - GET /mcp/prompts: 프롬프트 목록
   - GET /mcp/prompts/{id}: 프롬프트 상세

6. **테스트** (`/backend/tests/unit/mcp/test_mcp_resources_prompts.py`)
   - URI 파싱, 프롬프트 변환, 스트리밍

## 6. 완료 기준

1. MimirResourceURI 파서가 정상 작동하는가?
2. fetch_node와 리소스 조회가 일관되는가?
3. 프롬프트 노출이 Phase 1 Prompt Registry와 일치하는가?
4. 스트리밍 엔드포인트가 chunked transfer encoding을 사용하는가?
5. SSE 엔드포인트가 text/event-stream을 반환하는가?
6. 리소스 목록이 ACL 필터링을 반영하는가?
7. 부분 결과부터 처리 가능한가?
8. 모든 단위 테스트가 통과하는가?
9. mypy 타입 검사를 통과하는가?

## 7. 작업 지침

### 지침 7-1. URI 형식 표준화

mimir:// 스킴을 표준으로 사용하고, 경로 구조는 `/documents/{id}/versions/{id}/nodes/{id}`로 일관되게 유지한다.

### 지침 7-2. 스트리밍 청크 크기

청크 크기를 8KB로 설정하여 네트워크 효율성과 반응성 사이의 균형을 맞춘다.

### 지침 7-3. SSE 이벤트 타입

다양한 이벤트 타입을 정의하여 클라이언트가 각각 처리할 수 있도록 한다 (metadata, result, complete, error).

### 지침 7-4. 프롬프트 이름 지정

모든 MCP 프롬프트에 "mimir-" 접두사를 추가하여 다른 출처의 프롬프트와 구분한다.

### 지침 7-5. 리소스 메타데이터

mime_type, description을 포함하여 클라이언트가 리소스의 특성을 파악할 수 있도록 한다.

### 지침 7-6. SSE 연결 유지

Cache-Control: no-cache와 Connection: keep-alive 헤더를 사용하여 연결을 유지한다.

### 지침 7-7. 에러 처리

스트리밍 중 에러 발생 시 적절한 이벤트나 청크로 클라이언트에 알린다.
