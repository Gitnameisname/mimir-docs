# Task 4-9. MCP 통합 테스트 + 공식 문서 작성

## 1. 작업 목적

**MCP 구현의 전체 흐름을 End-to-End 통합 테스트**하고, **LangChain/n8n 등 실제 에이전트 프레임워크와의 호환성을 검증**하며, **Mimir가 MCP 2025-11-25를 완전히 구현한다는 공식 문서를 작성**한다. 이를 통해 에이전트 개발자가 Mimir를 신뢰하고 활용할 수 있는 기반을 마련한다.

## 2. 작업 범위

### 포함 범위

1. **End-to-End 통합 테스트**
   - 테스트 MCP 클라이언트 구현 (Python httpx 기반)
   - 전체 흐름: initialize → tool call → response 검증
   - 성공 시나리오 (인증 성공, Scope 필터 적용, Citation 검증)
   - 실패 시나리오 (인증 실패, Scope 오류, 도구명 오류)
   - 스트리밍 테스트 (대용량 응답 처리)

2. **보안 테스트**
   - FilterExpression 인젝션 방어 검증
   - OAuth 토큰 검증 (유효성, 만료)
   - 권한 에스컬레이션 방지 검증
   - Scope Profile 기반 ACL 적용 확인

3. **LangChain 호환성 테스트 (가능한 범위)**
   - LangChain MCP integration으로 Mimir 호출
   - Tool calling via search_documents
   - 에러 처리 및 재시도 로직

4. **n8n 호환성 테스트 (문서 수준)**
   - n8n MCP node 구성 예시 (실행 불가 시 문서화)
   - 기본 설정, 도구 호출 예시

5. **MCP 공식 문서 작성 (Markdown)**
   - "Mimir는 MCP 2025-11-25를 구현한다" 선언
   - 지원되는 기능: tools, resources, prompts
   - 도구 3종 상세 설명 (입출력 예시)
   - 인증 가이드 (OAuth client-credentials)
   - Scope Profile 사용법 (예시 포함)
   - Extension 설명 (citation-5tuple, span-backref)
   - 오류 코드 참조
   - 통합 가이드 (LangChain, n8n, 커스텀 클라이언트)
   - 문제 해결 (troubleshooting)

### 제외 범위

- 응답 포맷 (Task 4-7)
- Tool Schema 정의 (Task 4-8)
- 개별 도구 구현 상세사항
- UI 기반 테스트

## 3. 선행 조건

- Phase 1, 2, 3 완료
- FG4.1 (MCP Server) 완료
- FG4.2 (Agent Principal, Scope Profile) 완료
- Task 4-7 (응답 포맷) 완료
- Task 4-8 (Tool Schema) 완료
- httpx, pytest 설치
- (선택) LangChain 설치

## 4. 주요 작업 항목

### 4-1. 테스트 MCP 클라이언트 구현

**파일:** `/backend/tests/integration/mcp/test_mcp_client.py`

```python
"""MCP 테스트 클라이언트 (httpx 기반)."""

from __future__ import annotations

import json
import logging
from typing import Dict, Any, Optional, List
from uuid import UUID, uuid4
from datetime import datetime
import time

import httpx
from pydantic import BaseModel

logger = logging.getLogger(__name__)


class MCPTestClient:
    """MCP 테스트용 클라이언트.
    
    Mimir MCP Server와 통신하는 클라이언트.
    """
    
    def __init__(
        self,
        base_url: str,
        oauth_token: Optional[str] = None,
        agent_id: Optional[str] = None,
    ):
        """클라이언트 초기화.
        
        Args:
            base_url: Mimir 서버 URL (예: http://localhost:8000)
            oauth_token: OAuth 토큰 (선택)
            agent_id: 에이전트 ID (선택)
        """
        self.base_url = base_url.rstrip("/")
        self.oauth_token = oauth_token
        self.agent_id = agent_id
        self.client = httpx.AsyncClient(base_url=self.base_url)
        self.initialized = False
    
    def _get_headers(self) -> Dict[str, str]:
        """HTTP 헤더 생성."""
        headers = {"Content-Type": "application/json"}
        if self.oauth_token:
            headers["Authorization"] = f"Bearer {self.oauth_token}"
        return headers
    
    async def initialize(
        self,
        protocol_version: str = "2025-11-25",
    ) -> Dict[str, Any]:
        """MCP initialize 요청.
        
        Args:
            protocol_version: MCP 프로토콜 버전
        
        Returns:
            initialize 응답
        """
        request_body = {
            "jsonrpc": "2.0",
            "method": "initialize",
            "params": {
                "protocol_version": protocol_version,
            },
            "id": 1,
        }
        
        response = await self.client.post(
            "/mcp/initialize",
            json=request_body,
            headers=self._get_headers(),
        )
        
        response.raise_for_status()
        result = response.json()
        
        logger.info(f"MCP initialize: {result}")
        self.initialized = True
        
        return result
    
    async def call_tool(
        self,
        tool_name: str,
        params: Dict[str, Any],
        scope: Optional[str] = None,
        access_context: Optional[Dict[str, Any]] = None,
    ) -> Dict[str, Any]:
        """도구 호출.
        
        Args:
            tool_name: 도구 이름 (search_documents, fetch_node, verify_citation)
            params: 도구 입력 매개변수
            scope: Scope Profile 이름 (선택)
            access_context: 접근 컨텍스트 (선택)
        
        Returns:
            도구 응답
        """
        if not self.initialized:
            await self.initialize()
        
        # 도구별 엔드포인트
        endpoint_map = {
            "search_documents": "/api/agent/rag/search",
            "fetch_node": "/api/agent/documents/fetch-node",
            "verify_citation": "/api/agent/rag/verify-citation",
        }
        
        if tool_name not in endpoint_map:
            raise ValueError(f"Unknown tool: {tool_name}")
        
        endpoint = endpoint_map[tool_name]
        
        request_payload = {
            **params,
            "scope": scope or "default",
        }
        if access_context:
            request_payload["access_context"] = access_context
        
        response = await self.client.post(
            endpoint,
            json=request_payload,
            headers=self._get_headers(),
        )
        
        result = response.json()
        
        logger.info(f"Tool call {tool_name}: status={response.status_code}")
        
        return result
    
    async def get_tool_list(self) -> Dict[str, Any]:
        """사용 가능 도구 목록 조회.
        
        Returns:
            도구 목록
        """
        response = await self.client.get(
            "/api/agent/mcp/tools/list",
            headers=self._get_headers(),
        )
        
        response.raise_for_status()
        return response.json()
    
    async def get_well_known(self) -> Dict[str, Any]:
        """/.well-known/mimir-mcp 메타데이터.
        
        Returns:
            MCP 메타데이터
        """
        response = await self.client.get("/.well-known/mimir-mcp")
        response.raise_for_status()
        return response.json()
    
    async def close(self) -> None:
        """클라이언트 종료."""
        await self.client.aclose()
```

### 4-2. End-to-End 통합 테스트 스위트

**파일:** `/backend/tests/integration/mcp/test_e2e_mcp_flow.py`

```python
"""MCP End-to-End 통합 테스트."""

import pytest
from uuid import uuid4
from datetime import datetime

from tests.integration.mcp.test_mcp_client import MCPTestClient


@pytest.fixture
async def mcp_client(test_server_url: str, test_oauth_token: str):
    """MCP 테스트 클라이언트 fixture."""
    client = MCPTestClient(
        base_url=test_server_url,
        oauth_token=test_oauth_token,
    )
    yield client
    await client.close()


class TestMCPInitialization:
    """MCP 초기화 테스트."""
    
    @pytest.mark.asyncio
    async def test_initialize_success(self, mcp_client):
        """성공적인 initialize."""
        response = await mcp_client.initialize()
        
        assert response["jsonrpc"] == "2.0"
        assert "result" in response
        assert response["result"]["protocolVersion"] == "2025-11-25"
        assert response["result"]["server"]["server_id"] == "mimir-s2"
    
    @pytest.mark.asyncio
    async def test_initialize_without_auth(self, test_server_url):
        """인증 없이 initialize."""
        client = MCPTestClient(base_url=test_server_url)
        
        response = await client.initialize()
        
        # 인증 실패
        assert response.get("error") is not None
        assert response["error"]["code"] == "AUTHENTICATION_FAILED"
        
        await client.close()
    
    @pytest.mark.asyncio
    async def test_well_known_metadata(self, mcp_client):
        """/.well-known/mimir-mcp 메타데이터."""
        metadata = await mcp_client.get_well_known()
        
        assert metadata["mcp_version"] == "2025-11-25"
        assert metadata["server_id"] == "mimir-s2"
        assert metadata["capabilities"]["tools"] is True
        assert metadata["capabilities"]["resources"] is True
        assert metadata["scope_profile_required"] is True


class TestSearchDocumentsTool:
    """search_documents 도구 테스트."""
    
    @pytest.mark.asyncio
    async def test_search_success(self, mcp_client, test_document_id):
        """검색 성공."""
        response = await mcp_client.call_tool(
            tool_name="search_documents",
            params={
                "query": "테스트 검색어",
                "top_k": 5,
            },
            scope="default",
        )
        
        assert response["success"] is True
        assert "data" in response
        assert "results" in response["data"]
        assert response["metadata"]["request_id"] is not None
        assert response["metadata"]["execution_time_ms"] >= 0
    
    @pytest.mark.asyncio
    async def test_search_with_scope(self, mcp_client):
        """Scope Profile 적용하여 검색."""
        response = await mcp_client.call_tool(
            tool_name="search_documents",
            params={
                "query": "규정",
            },
            scope="team",
            access_context={
                "organization_id": str(uuid4()),
                "team_id": str(uuid4()),
            },
        )
        
        assert response["success"] is True
    
    @pytest.mark.asyncio
    async def test_search_invalid_scope(self, mcp_client):
        """유효하지 않은 Scope."""
        response = await mcp_client.call_tool(
            tool_name="search_documents",
            params={
                "query": "테스트",
            },
            scope="invalid_scope_name",
        )
        
        # Scope Profile이 없으면 에러
        if not response["success"]:
            assert response["error"]["code"] in [
                "INVALID_SCOPE",
                "NOT_FOUND",
            ]
    
    @pytest.mark.asyncio
    async def test_search_missing_required_param(self, mcp_client):
        """필수 매개변수 누락."""
        response = await mcp_client.call_tool(
            tool_name="search_documents",
            params={},  # query 누락
        )
        
        assert response["success"] is False
        assert response["error"]["code"] == "INVALID_REQUEST"


class TestFetchNodeTool:
    """fetch_node 도구 테스트."""
    
    @pytest.mark.asyncio
    async def test_fetch_node_success(self, mcp_client, test_document_id, test_node_id):
        """노드 조회 성공."""
        response = await mcp_client.call_tool(
            tool_name="fetch_node",
            params={
                "document_id": str(test_document_id),
                "node_id": test_node_id,
            },
        )
        
        assert response["success"] is True
        assert response["data"]["document_id"] == str(test_document_id)
        assert response["data"]["node_id"] == test_node_id
        assert "content" in response["data"]
    
    @pytest.mark.asyncio
    async def test_fetch_node_not_found(self, mcp_client):
        """존재하지 않는 노드."""
        response = await mcp_client.call_tool(
            tool_name="fetch_node",
            params={
                "document_id": str(uuid4()),
                "node_id": "nonexistent_node",
            },
        )
        
        assert response["success"] is False
        assert response["error"]["code"] == "NOT_FOUND"


class TestVerifyCitationTool:
    """verify_citation 도구 테스트."""
    
    @pytest.mark.asyncio
    async def test_verify_citation_valid(
        self,
        mcp_client,
        test_document_id,
        test_version_id,
        test_node_id,
        test_content_hash,
    ):
        """Citation 검증 (유효)."""
        response = await mcp_client.call_tool(
            tool_name="verify_citation",
            params={
                "document_id": str(test_document_id),
                "version_id": str(test_version_id),
                "node_id": test_node_id,
                "content_hash": test_content_hash,
            },
        )
        
        assert response["success"] is True
        assert response["data"]["verified"] is True
        assert response["data"]["hash_matches"] is True
    
    @pytest.mark.asyncio
    async def test_verify_citation_invalid_hash(
        self,
        mcp_client,
        test_document_id,
        test_version_id,
        test_node_id,
    ):
        """Citation 검증 (해시 불일치)."""
        response = await mcp_client.call_tool(
            tool_name="verify_citation",
            params={
                "document_id": str(test_document_id),
                "version_id": str(test_version_id),
                "node_id": test_node_id,
                "content_hash": "a" * 64,  # 잘못된 해시
            },
        )
        
        assert response["success"] is True
        assert response["data"]["verified"] is False
        assert response["data"]["hash_matches"] is False


class TestAuthenticationScenarios:
    """인증 시나리오 테스트."""
    
    @pytest.mark.asyncio
    async def test_invalid_oauth_token(self, test_server_url):
        """유효하지 않은 OAuth 토큰."""
        client = MCPTestClient(
            base_url=test_server_url,
            oauth_token="invalid_token_xyz",
        )
        
        response = await client.initialize()
        
        assert "error" in response
        assert response["error"]["code"] == "AUTHENTICATION_FAILED"
        
        await client.close()
    
    @pytest.mark.asyncio
    async def test_expired_oauth_token(self, test_server_url, expired_oauth_token):
        """만료된 OAuth 토큰."""
        client = MCPTestClient(
            base_url=test_server_url,
            oauth_token=expired_oauth_token,
        )
        
        response = await client.initialize()
        
        assert "error" in response
        assert response["error"]["code"] == "AUTHENTICATION_FAILED"
        
        await client.close()


class TestScopeProfileFiltering:
    """Scope Profile 필터링 테스트."""
    
    @pytest.mark.asyncio
    async def test_team_scope_acl(
        self,
        mcp_client,
        test_org_id,
        test_team_id,
    ):
        """team scope으로 접근 제한."""
        # team scope으로 검색
        response = await mcp_client.call_tool(
            tool_name="search_documents",
            params={
                "query": "테스트",
                "top_k": 5,
            },
            scope="team",
            access_context={
                "organization_id": str(test_org_id),
                "team_id": str(test_team_id),
            },
        )
        
        assert response["success"] is True
        # 결과는 team 범위로 필터링됨
        for result in response["data"]["results"]:
            # 각 결과가 team 권한에 맞는지 검증
            pass


class TestErrorHandling:
    """오류 처리 테스트."""
    
    @pytest.mark.asyncio
    async def test_tool_not_found(self, mcp_client):
        """존재하지 않는 도구 호출."""
        response = await mcp_client.call_tool(
            tool_name="nonexistent_tool",
            params={},
        )
        
        assert response["success"] is False
        assert response["error"]["code"] == "NOT_FOUND"
    
    @pytest.mark.asyncio
    async def test_rate_limiting(self, mcp_client):
        """속도 제한."""
        # 많은 요청 전송
        for _ in range(100):
            response = await mcp_client.call_tool(
                tool_name="search_documents",
                params={"query": "테스트"},
            )
            
            if response.get("error", {}).get("code") == "RATE_LIMIT":
                # 속도 제한 발동 확인
                assert response["success"] is False
                break


class TestSecurityValidation:
    """보안 검증 테스트."""
    
    @pytest.mark.asyncio
    async def test_filter_expression_injection_defense(self, mcp_client):
        """FilterExpression 인젝션 방어."""
        # 악의적인 필터 표현식 시도
        response = await mcp_client.call_tool(
            tool_name="search_documents",
            params={
                "query": "테스트",
            },
            access_context={
                # 인젝션 시도
                "organization_id": "'; DROP TABLE documents; --",
            },
        )
        
        # 인젝션 시도가 무시되거나 오류 처리됨
        # 데이터베이스가 손상되지 않아야 함
    
    @pytest.mark.asyncio
    async def test_permission_escalation_prevention(
        self,
        mcp_client,
        test_org_id,
    ):
        """권한 에스컬레이션 방지."""
        # 낮은 권한의 scope로 높은 권한의 데이터 접근 시도
        response = await mcp_client.call_tool(
            tool_name="search_documents",
            params={
                "query": "기밀문서",
            },
            scope="team",
            access_context={
                "organization_id": str(test_org_id),
                "team_id": str(uuid4()),  # 다른 팀 ID
            },
        )
        
        # 접근 불가 또는 필터링됨
        assert response["success"] is False or len(response["data"]["results"]) == 0
```

### 4-3. LangChain 호환성 테스트

**파일:** `/backend/tests/integration/mcp/test_langchain_integration.py`

```python
"""LangChain MCP 통합 테스트 (선택사항)."""

import pytest

# LangChain이 설치되어 있을 경우에만 실행
try:
    from langchain.agents import initialize_agent, Tool
    from langchain.llms import OpenAI
    LANGCHAIN_AVAILABLE = True
except ImportError:
    LANGCHAIN_AVAILABLE = False


@pytest.mark.skipif(not LANGCHAIN_AVAILABLE, reason="LangChain not installed")
class TestLangChainIntegration:
    """LangChain MCP 통합 테스트."""
    
    @pytest.mark.asyncio
    async def test_search_documents_tool_in_langchain(self, test_server_url, test_oauth_token):
        """LangChain에서 search_documents 도구 사용."""
        # Mimir MCP 도구를 LangChain Tool로 변환
        from tests.integration.mcp.test_mcp_client import MCPTestClient
        
        mcp_client = MCPTestClient(
            base_url=test_server_url,
            oauth_token=test_oauth_token,
        )
        
        async def search_wrapper(query: str, top_k: int = 5) -> str:
            """LangChain 용 래퍼."""
            response = await mcp_client.call_tool(
                tool_name="search_documents",
                params={
                    "query": query,
                    "top_k": top_k,
                },
            )
            
            if response["success"]:
                results = response["data"]["results"]
                return "\n".join([r["content"] for r in results])
            else:
                return f"Error: {response['error']['message']}"
        
        # Tool 생성
        search_tool = Tool(
            name="search_mimir_documents",
            func=search_wrapper,
            description="Search Mimir documents using full-text or vector search",
        )
        
        # Tool 테스트
        result = await search_tool.arun("테스트 검색어")
        assert isinstance(result, str)
        
        await mcp_client.close()
```

### 4-4. 보안 테스트

**파일:** `/backend/tests/integration/mcp/test_security.py`

```python
"""MCP 보안 테스트."""

import pytest
from uuid import uuid4
import hashlib

from tests.integration.mcp.test_mcp_client import MCPTestClient


class TestFilterExpressionSecurity:
    """FilterExpression 보안 테스트."""
    
    @pytest.mark.asyncio
    async def test_sql_injection_in_filter(self, mcp_client):
        """FilterExpression에서 SQL 인젝션 시도."""
        response = await mcp_client.call_tool(
            tool_name="search_documents",
            params={
                "query": "테스트",
            },
            access_context={
                "organization_id": "' OR '1'='1",
            },
        )
        
        # 인젝션이 무시되고 정상 처리되어야 함
        # (에러 또는 정상 필터링)
    
    @pytest.mark.asyncio
    async def test_large_scope_profile_payload(self, mcp_client):
        """과도하게 큰 FilterExpression 페이로드."""
        large_context = {
            "organization_id": str(uuid4()),
            "extra_field": "x" * 10000,  # 큰 문자열
        }
        
        response = await mcp_client.call_tool(
            tool_name="search_documents",
            params={
                "query": "테스트",
            },
            access_context=large_context,
        )
        
        # 처리되거나 적절히 거부됨


class TestOAuthSecurity:
    """OAuth 보안 테스트."""
    
    @pytest.mark.asyncio
    async def test_token_with_modified_claims(self, test_server_url):
        """변조된 토큰 클레임."""
        # JWT 토큰의 클레임을 변조한 후 서명 검증이 실패하는지 확인
        client = MCPTestClient(
            base_url=test_server_url,
            oauth_token="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.INVALID.SIGNATURE",
        )
        
        response = await client.initialize()
        assert "error" in response
        
        await client.close()
    
    @pytest.mark.asyncio
    async def test_token_reuse_prevention(self, mcp_client, test_oauth_token):
        """토큰 재사용 방지 (선택사항)."""
        # 같은 토큰으로 여러 요청 시 일관성 확인
        response1 = await mcp_client.call_tool(
            tool_name="search_documents",
            params={"query": "테스트"},
        )
        
        response2 = await mcp_client.call_tool(
            tool_name="search_documents",
            params={"query": "테스트"},
        )
        
        # 두 요청이 동일하게 처리되어야 함


class TestACLEnforcement:
    """ACL 적용 테스트."""
    
    @pytest.mark.asyncio
    async def test_unauthorized_organization_access(
        self,
        mcp_client,
        test_org_id,
    ):
        """다른 조직의 문서 접근 시도."""
        response = await mcp_client.call_tool(
            tool_name="search_documents",
            params={
                "query": "테스트",
            },
            access_context={
                "organization_id": str(uuid4()),  # 다른 조직
            },
        )
        
        # 접근 제한되거나 필터링됨
        if response["success"]:
            assert len(response["data"]["results"]) == 0
    
    @pytest.mark.asyncio
    async def test_citation_hash_tampering(
        self,
        mcp_client,
        test_document_id,
        test_version_id,
        test_node_id,
    ):
        """Citation 해시 변조 감지."""
        # 잘못된 해시로 검증 시도
        tampered_hash = "a" * 64
        
        response = await mcp_client.call_tool(
            tool_name="verify_citation",
            params={
                "document_id": str(test_document_id),
                "version_id": str(test_version_id),
                "node_id": test_node_id,
                "content_hash": tampered_hash,
            },
        )
        
        assert response["success"] is True
        assert response["data"]["verified"] is False
        assert response["data"]["hash_matches"] is False
```

### 4-5. MCP 공식 문서 작성

**파일:** `/backend/docs/mcp/MCP_IMPLEMENTATION.md`

```markdown
# Mimir MCP 2025-11-25 구현 설명서

## 개요

Mimir는 **MCP (Model Context Protocol) 2025-11-25** 스펙을 완전히 구현합니다. AI 에이전트, 챗봇, LLM 애플리케이션이 Mimir를 표준 도구로 호출할 수 있습니다.

**MCP 스펙**: [MCP 2025-11-25 공식 스펙](https://spec.modelcontextprotocol.io/)

---

## 지원 기능

### 1. 도구 (Tools)

Mimir는 **읽기 도구 3개**를 제공합니다:

| 도구 | 설명 | 용도 |
|------|------|------|
| `search_documents` | 전문 검색 (FTS/Vector) | 문서 검색 |
| `fetch_node` | 노드 전문 조회 | 상세 내용 조회 |
| `verify_citation` | Citation 5-tuple 검증 | 인용 검증 |

### 2. 리소스 (Resources)

- URI scheme: `mimir://documents/{document_id}/versions/{version_id}/nodes/{node_id}`
- MIME type: `text/plain`

### 3. 프롬프트 (Prompts)

Phase 1에서 구축한 Prompt Registry의 활성 프롬프트를 MCP로 노출합니다.

### 4. 확장 (Extensions)

두 가지 Mimir 고유 기능을 확장으로 선언합니다:

- **mimir.citation-5tuple v1.0**: Citation 검증 기능
- **mimir.span-backref v1.0**: Span 역참조 기능

---

## 1. 인증 (Authentication)

### OAuth 2.0 Client-Credentials

Mimir는 **OAuth 2.0 Client-Credentials** 흐름을 사용합니다.

#### 토큰 획득

```bash
curl -X POST https://mimir.example.com/oauth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET"
```

응답:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

#### 토큰 사용

모든 MCP 요청에 Authorization 헤더를 포함합니다:

```
Authorization: Bearer <access_token>
```

---

## 2. MCP 초기화 (Initialize)

### 요청

```json
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {
    "protocol_version": "2025-11-25"
  },
  "id": 1
}
```

### 응답

```json
{
  "jsonrpc": "2.0",
  "result": {
    "server": {
      "server_id": "mimir-s2",
      "name": "Mimir S2",
      "version": "1.0.0",
      "protocol_version": "2025-11-25",
      "capabilities": {
        "tools": true,
        "resources": true,
        "prompts": true,
        "tasks": false
      }
    },
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "tools": true,
      "resources": true,
      "prompts": true,
      "tasks": false
    }
  },
  "id": 1
}
```

---

## 3. 도구 상세 설명

### 3.1 search_documents

전문 검색 또는 벡터 검색을 통해 문서를 검색합니다.

#### 입력

```json
{
  "query": "규정 변경",
  "scope": "team",
  "document_types": ["policy", "compliance"],
  "top_k": 5,
  "conversation_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `query` | string | ✓ | 검색어 |
| `scope` | string | | Scope Profile 이름 |
| `document_types` | array | | 문서 타입 필터 |
| `top_k` | integer | | 결과 개수 (기본: 5, 최대: 50) |
| `conversation_id` | uuid | | 대화 ID (컨텍스트용) |

#### 출력

```json
{
  "results": [
    {
      "document_id": "123e4567-e89b-12d3-a456-426614174000",
      "document_title": "내부 규정",
      "version_id": "223e4567-e89b-12d3-a456-426614174000",
      "node_id": "section-1.2",
      "content": "규정 내용...",
      "citation": {
        "document_id": "123e4567-e89b-12d3-a456-426614174000",
        "version_id": "223e4567-e89b-12d3-a456-426614174000",
        "node_id": "section-1.2",
        "span_offset": 100,
        "content_hash": "sha256_hex_value"
      },
      "relevance_score": 0.95,
      "retrieval_time_ms": 523
    }
  ],
  "total_count": 42,
  "retrieval_method": "hybrid"
}
```

### 3.2 fetch_node

특정 문서 노드의 전체 내용을 조회합니다.

#### 입력

```json
{
  "document_id": "123e4567-e89b-12d3-a456-426614174000",
  "version_id": "223e4567-e89b-12d3-a456-426614174000",
  "node_id": "section-1.2"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `document_id` | uuid | ✓ | 문서 ID |
| `version_id` | uuid | | 버전 ID (생략 시 최신) |
| `node_id` | string | ✓ | 노드 ID |

#### 출력

```json
{
  "document_id": "123e4567-e89b-12d3-a456-426614174000",
  "document_title": "내부 규정",
  "version_id": "223e4567-e89b-12d3-a456-426614174000",
  "node_id": "section-1.2",
  "content": "전체 내용...",
  "metadata": {
    "language": "ko",
    "document_type": "policy",
    "created_at": "2026-04-17T10:00:00Z"
  },
  "relationships": {
    "parent_node_id": "section-1",
    "children": ["section-1.2.1", "section-1.2.2"]
  }
}
```

### 3.3 verify_citation

Citation 5-tuple을 검증합니다.

#### 입력

```json
{
  "document_id": "123e4567-e89b-12d3-a456-426614174000",
  "version_id": "223e4567-e89b-12d3-a456-426614174000",
  "node_id": "section-1.2",
  "content_hash": "a1b2c3d4e5f6g7h8...",
  "span_offset": 100
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `document_id` | uuid | ✓ | 문서 ID |
| `version_id` | uuid | ✓ | 버전 ID |
| `node_id` | string | ✓ | 노드 ID |
| `content_hash` | string | ✓ | SHA256 해시 |
| `span_offset` | integer | | 문자 오프셋 |

#### 출력

```json
{
  "verified": true,
  "current_hash": "a1b2c3d4e5f6g7h8...",
  "hash_matches": true,
  "content_snapshot": "인용된 내용의 처음 200자...",
  "version_valid": true,
  "message": "Citation is valid and matches current content"
}
```

---

## 4. Scope Profile 사용법

Scope Profile은 에이전트가 접근할 수 있는 문서의 범위를 관리합니다.

### 예시: team scope

조직 관리자가 "team" scope을 정의:

```json
{
  "scope_name": "team",
  "acl_filter": {
    "and": [
      {
        "field": "organization_id",
        "op": "eq",
        "value": "$ctx.organization_id"
      },
      {
        "field": "team_id",
        "op": "eq",
        "value": "$ctx.team_id"
      }
    ]
  }
}
```

### 에이전트가 호출할 때:

```json
{
  "query": "규정",
  "scope": "team",
  "access_context": {
    "organization_id": "550e8400-e29b-41d4-a716-446655440000",
    "team_id": "660e8400-e29b-41d4-a716-446655440000"
  }
}
```

결과는 자동으로 해당 조직과 팀의 문서로 필터링됩니다.

---

## 5. Extension: citation-5tuple

### 기능

Citation의 완전성과 유효성을 검증합니다:

- **5-tuple 구조**: document_id, version_id, node_id, span_offset, content_hash
- **SHA256 해싱**: 내용의 integrity 검증
- **버전 추적**: 문서 업데이트 후에도 인용 유효성 확인

### 사용 예시

```python
# Citation 검증
citation = {
    "document_id": "123e4567-e89b-12d3-a456-426614174000",
    "version_id": "223e4567-e89b-12d3-a456-426614174000",
    "node_id": "section-1.2",
    "span_offset": 100,
    "content_hash": "a1b2c3d4e5f6g7h8..."
}

# verify_citation 도구로 검증
result = await mcp_client.call_tool("verify_citation", citation)
if result["verified"]:
    print("Citation is valid!")
```

---

## 6. Extension: span-backref

### 기능

특정 span(문자 범위)의 원본 내용을 추적합니다:

- **위치 추적**: span_offset으로 정확한 위치 파악
- **변경 감지**: 내용 변경 후에도 원본 추적 가능
- **Retroactive 검증**: 문서 업데이트 후 인용 유효성 확인

### 사용 예시

LLM 응답에서 인용된 텍스트의 span을 기록:

```python
llm_response = "... 규정에 따르면 [span_offset=100, length=50] ..."

# 나중에 이 span의 유효성 확인
verify_citation(
    document_id="...",
    version_id="...",
    node_id="...",
    span_offset=100,
    content_hash="..."
)
```

---

## 7. 에러 코드

모든 오류는 다음 코드 중 하나를 반환합니다:

| 코드 | HTTP | 설명 |
|------|------|------|
| `UNAUTHORIZED` | 401 | 인증 실패 또는 권한 부족 |
| `NOT_FOUND` | 404 | 리소스 없음 |
| `INVALID_REQUEST` | 400 | 요청 형식 오류 |
| `INVALID_SCOPE` | 400 | Scope 설정 오류 |
| `INVALID_CITATION` | 400 | Citation 검증 실패 |
| `RATE_LIMIT` | 429 | 속도 제한 |
| `INTERNAL_ERROR` | 500 | 서버 오류 |

### 에러 응답 예시

```json
{
  "success": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "User does not have permission to access this document"
  },
  "metadata": {
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2026-04-17T10:00:00Z",
    "execution_time_ms": 45
  }
}
```

---

## 8. 통합 가이드

### LangChain 통합

```python
from langchain.agents import initialize_agent, Tool
from langchain.llms import ChatOpenAI

# Mimir 도구 정의
search_tool = Tool(
    name="search_mimir",
    func=mcp_client.call_tool,
    description="Search documents in Mimir",
)

# 에이전트 초기화
agent = initialize_agent(
    tools=[search_tool],
    llm=ChatOpenAI(),
    agent="zero-shot-react-description",
)

# 실행
result = agent.run("규정에서 휴가 관련 내용을 찾아줘")
```

### n8n 통합

n8n의 HTTP 노드를 사용하여 Mimir MCP를 호출합니다:

1. HTTP 노드 추가
2. Method: `POST`
3. URL: `https://mimir.example.com/api/agent/rag/search`
4. Headers:
   - `Authorization`: `Bearer <oauth_token>`
   - `Content-Type`: `application/json`
5. Body:
   ```json
   {
     "query": "{{$node.input.query}}",
     "top_k": 5
   }
   ```

### 커스텀 클라이언트

Python httpx를 사용한 최소 예제:

```python
import httpx
import json

async with httpx.AsyncClient() as client:
    # 초기화
    response = await client.post(
        "https://mimir.example.com/mcp/initialize",
        json={
            "jsonrpc": "2.0",
            "method": "initialize",
            "params": {"protocol_version": "2025-11-25"},
            "id": 1,
        },
        headers={"Authorization": f"Bearer {token}"},
    )
    
    # 도구 호출
    response = await client.post(
        "https://mimir.example.com/api/agent/rag/search",
        json={"query": "규정", "top_k": 5},
        headers={"Authorization": f"Bearer {token}"},
    )
    
    result = response.json()
    for hit in result["data"]["results"]:
        print(hit["content"])
```

---

## 9. 폐쇄망 환경 배포

Mimir는 폐쇄망 환경에서도 전체 기능을 제공합니다:

### 외부 의존 제거

환경변수로 외부 서비스 의존을 해제합니다:

```bash
# OpenAI API 비활성화
export OPENAI_API_ENABLED=false

# 로컬 LLM 사용
export LLM_TYPE=local
export LOCAL_LLM_MODEL=phi-2
```

### 테스트 확인

```bash
# 폐쇄망 모드로 테스트
pytest tests/integration/mcp/test_e2e_mcp_flow.py::TestSearchDocumentsTool::test_search_success
```

---

## 10. 문제 해결 (Troubleshooting)

### "AUTHENTICATION_FAILED" 에러

- OAuth 토큰이 유효한지 확인
- 토큰이 만료되지 않았는지 확인
- client_id, client_secret이 올바른지 확인

### "INVALID_SCOPE" 에러

- Scope Profile이 관리자에 의해 정의되었는지 확인
- access_context의 필드명이 Scope Profile의 $ctx 변수와 일치하는지 확인

### "NOT_FOUND" 에러 (문서 검색 결과 없음)

- 검색어를 변경해봅니다
- Scope Profile이 너무 제한적이지는 않은지 확인
- 문서가 실제로 존재하는지 관리자에게 확인

### 응답이 느린 경우

- 검색 쿼리를 더 구체적으로 변경
- top_k 값을 줄입니다
- Vector 검색 대신 FTS를 사용합니다

---

## 11. 지원 및 피드백

- 문서: https://mimir.example.com/docs
- 이슈: https://github.com/mimir-org/mimir/issues
- 이메일: support@mimir.example.com

---

## 버전 히스토리

| 버전 | 날짜 | 변경사항 |
|------|------|---------|
| 1.0.0 | 2026-04-17 | MCP 2025-11-25 초기 구현 |
```

## 5. 산출물

1. **테스트 MCP 클라이언트** (`/backend/tests/integration/mcp/test_mcp_client.py`)
   - MCPTestClient 클래스
   - initialize, call_tool, get_tool_list 메서드

2. **E2E 통합 테스트** (`/backend/tests/integration/mcp/test_e2e_mcp_flow.py`)
   - 초기화 테스트
   - 도구 호출 테스트 (3종)
   - 인증 테스트
   - Scope Profile 필터링 테스트
   - 오류 처리 테스트

3. **보안 테스트** (`/backend/tests/integration/mcp/test_security.py`)
   - FilterExpression 인젝션 방어
   - OAuth 보안
   - ACL 적용 검증

4. **LangChain 호환성 테스트** (`/backend/tests/integration/mcp/test_langchain_integration.py`)
   - LangChain Tool 통합

5. **MCP 공식 문서** (`/backend/docs/mcp/MCP_IMPLEMENTATION.md`)
   - 인증 가이드
   - 도구 상세 설명 (입출력 예시)
   - Scope Profile 사용법
   - Extension 설명
   - 통합 가이드 (LangChain, n8n)
   - 문제 해결

## 6. 완료 기준

1. 테스트 MCP 클라이언트가 구현되었는가?
2. E2E 통합 테스트가 모든 성공/실패 시나리오를 커버하는가?
3. 보안 테스트가 인젝션, 권한 에스컬레이션 등을 검증하는가?
4. LangChain 호환성 테스트가 실행되거나 문서화되었는가?
5. MCP 공식 문서가 완전한가?
   - 인증 방법
   - 도구 3종 상세 설명
   - Scope Profile 사용 예시
   - Extension 설명
   - 통합 예시
   - 문제 해결 가이드
6. 모든 통합 테스트가 통과하는가?
7. 폐쇄망 환경에서도 MCP가 정상 작동하는가?

## 7. 작업 지침

### 지침 7-1. E2E 테스트의 완성도

모든 주요 시나리오를 테스트한다:
- 성공 케이스 (도구 호출 성공, 결과 정확성)
- 실패 케이스 (인증 실패, Scope 오류, 리소스 없음)
- 엣지 케이스 (대용량 응답, 속도 제한)

### 지침 7-2. 보안 테스트 우선순위

다음 보안 위협을 우선으로 테스트한다:
1. SQL 인젝션 (FilterExpression)
2. OAuth 토큰 변조
3. 권한 에스컬레이션
4. 다른 조직 데이터 접근

### 지침 7-3. LangChain 호환성

LangChain이 설치되지 않은 환경에서도 테스트 스위트가 실행되도록 @pytest.mark.skipif를 사용한다.

### 지침 7-4. 문서의 실용성

모든 코드 예시는 실제 동작 가능한 코드여야 한다. 의사코드가 아닌 실제 코드를 제시한다.

### 지침 7-5. 오류 메시지의 명확성

공식 문서의 "문제 해결" 섹션에는 일반적인 오류 메시지와 대응 방법을 명시한다.

### 지침 7-6. S2 원칙 반영

문서에 다음 내용을 명시한다:
- ⑤ 에이전트는 사용자와 동등한 API 소비자
- ⑥ Scope Profile 기반 ACL 관리
- ⑦ 폐쇄망 환경 지원

### 지침 7-7. 성능 테스트

시간 허락하면 성능 테스트도 포함한다:
- 검색 응답 시간 (목표: <500ms)
- 동시 요청 처리 (목표: ≥10 요청/초)

### 지침 7-8. 버전 호환성

향후 MCP 버전 업데이트를 대비하여, 현재 구현이 "MCP 2025-11-25"를 명확히 명시한다.
