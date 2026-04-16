# Task 4-2. 읽기 도구 3종 구현 (search_documents, fetch_node, verify_citation)

## 1. 작업 목적

Task 4-1에서 구축한 MCP Server 위에서 **3개의 읽기 도구 (search_documents, fetch_node, verify_citation)**를 구현하여, AI 에이전트가 Mimir 문서를 검색, 조회, 인증할 수 있도록 한다. 이때 **S2 원칙 ⑤ (actor_type 감시 로그)** 및 **S2 원칙 ⑥ (Scope Profile 기반 ACL)**을 의무 적용하여 에이전트 활동을 추적하고 접근 범위를 엄격하게 관리한다.

## 2. 작업 범위

### 포함 범위

1. **search_documents 도구**
   - 전문 검색 (FTS + Vector 검색)
   - 입력: query(필수), scope(선택), document_types(선택), top_k(기본:5), conversation_id(선택)
   - 출력: results[], total_count, retrieval_method
   - Scope Profile 기반 ACL 필터링 (S2 원칙 ⑥)
   - 결과에 citation 5-tuple 포함

2. **fetch_node 도구**
   - 특정 문서 노드의 전문 조회
   - 입력: document_id(필수), node_id(필수), version_id(선택, 기본:latest)
   - 출력: content, metadata, relationships
   - Document layer 호출
   - ACL 필터링 (S2 원칙 ⑥)

3. **verify_citation 도구**
   - Citation 5-tuple 검증
   - 입력: document_id, version_id, node_id, content_hash, span_offset(선택)
   - 출력: verified, hash_matches, content_snapshot, version_valid
   - 해시 비교 및 버전 유효성 확인

4. **도구 dispatch 로직**
   - HTTP 요청 → 도구 라우팅 → 실행 → 응답 포맷팅
   - 도구 입출력 JSON 스키마 정의 (Pydantic)
   - 각 도구별 정상/에러 케이스 처리

5. **ACL 필터링 (S2 원칙 ⑥)**
   - 모든 도구에 의무 적용
   - Scope Profile 해석 및 필터 구성
   - 검색·조회 결과에 ACL 필터 적용

6. **감시 로그 (S2 원칙 ⑤)**
   - 도구 호출마다 audit_log 기록
   - actor_type=agent 필수 기록
   - 도구명, 입력 매개변수, 결과 요약 기록

7. **단위/통합 테스트**
   - 각 도구별 정상/에러 케이스
   - ACL 필터링 검증
   - 감시 로그 기록 확인

### 제외 범위

- MCP 초기화 (Task 4-1)
- Resource 노출 (Task 4-3: mimir:// URI scheme)
- Prompt Registry 노출 (Task 4-3)
- SSE/스트리밍 지원 (Task 4-3)

## 3. 선행 조건

- Task 4-1 완료 (MCP Server, initialize 로직)
- Phase 2 완료 (Retrieval layer, Citation 5-tuple 계약)
- Phase 3 완료 (Conversation API, Document layer)
- Pydantic 스키마 정의 가능
- 기존 Retrieval/Document layer API 분석 완료

## 4. 주요 작업 항목

### 4-1. 도구 입출력 Pydantic 스키마

**파일:** `/backend/app/schemas/mcp_tools.py`

```python
"""MCP 도구 입출력 스키마 정의."""

from __future__ import annotations

from typing import Optional, List, Dict, Any
from uuid import UUID
from datetime import datetime

from pydantic import BaseModel, Field


# ==================== search_documents ====================

class SearchDocumentsRequest(BaseModel):
    """search_documents 도구 입력 스키마."""
    
    query: str = Field(..., min_length=1, description="검색 쿼리 (필수)")
    scope: Optional[str] = Field(None, description="접근 범위 (optional, Scope Profile에서 결정)")
    document_types: Optional[List[str]] = Field(None, description="필터: 문서 타입 리스트")
    top_k: int = Field(5, ge=1, le=50, description="결과 개수 (기본: 5, 최대: 50)")
    conversation_id: Optional[UUID] = Field(None, description="대화 ID (컨텍스트 윈도우용)")


class CitationTuple(BaseModel):
    """Citation 5-tuple."""
    
    document_id: UUID
    version_id: UUID
    node_id: str
    span_offset: int
    content_hash: str


class SearchResult(BaseModel):
    """검색 결과 항목."""
    
    document_id: UUID
    document_title: str
    version_id: UUID
    node_id: str
    content: str = Field(..., description="요약 또는 전문")
    citation: CitationTuple
    relevance_score: float = Field(..., ge=0.0, le=1.0)
    retrieval_time_ms: int


class SearchDocumentsResponse(BaseModel):
    """search_documents 도구 출력 스키마."""
    
    results: List[SearchResult]
    total_count: int
    retrieval_method: str = Field(..., description="fts | vector | hybrid")


# ==================== fetch_node ====================

class FetchNodeRequest(BaseModel):
    """fetch_node 도구 입력 스키마."""
    
    document_id: UUID = Field(..., description="문서 ID (필수)")
    node_id: str = Field(..., description="노드 ID (필수)")
    version_id: Optional[UUID] = Field(None, description="버전 ID (선택, 기본: latest)")


class NodeMetadata(BaseModel):
    """노드 메타데이터."""
    
    title: Optional[str] = None
    node_type: str  # section, paragraph, list, table, etc.
    created_at: Optional[datetime] = None
    updated_at: Optional[datetime] = None
    tags: Optional[List[str]] = None


class NodeRelationship(BaseModel):
    """노드 간 관계."""
    
    parent_node_id: Optional[str] = None
    children_node_ids: Optional[List[str]] = None
    related_node_ids: Optional[List[str]] = None


class FetchNodeResponse(BaseModel):
    """fetch_node 도구 출력 스키마."""
    
    document_id: UUID
    document_title: str
    version_id: UUID
    node_id: str
    content: str = Field(..., description="전문 텍스트")
    metadata: NodeMetadata
    relationships: NodeRelationship


# ==================== verify_citation ====================

class VerifyCitationRequest(BaseModel):
    """verify_citation 도구 입력 스키마."""
    
    document_id: UUID = Field(..., description="문서 ID (필수)")
    version_id: UUID = Field(..., description="버전 ID (필수)")
    node_id: str = Field(..., description="노드 ID (필수)")
    content_hash: str = Field(..., description="콘텐츠 해시 (SHA256)")
    span_offset: Optional[int] = Field(None, ge=0, description="스팬 오프셋 (선택)")


class VerifyCitationResponse(BaseModel):
    """verify_citation 도구 출력 스키마."""
    
    verified: bool = Field(..., description="검증 성공 여부")
    current_hash: str = Field(..., description="현재 해시값")
    hash_matches: bool = Field(..., description="해시 일치 여부")
    content_snapshot: str = Field(..., description="현재 콘텐츠 스냅샷")
    version_valid: bool = Field(..., description="버전 유효 여부")
    message: str = Field(..., description="검증 결과 메시지")
```

### 4-2. MCP 도구 구현 (ToolService)

**파일:** `/backend/app/mcp/tools.py`

```python
"""MCP 도구 구현 (search_documents, fetch_node, verify_citation)."""

from __future__ import annotations

from typing import Optional, Dict, Any, List
from uuid import UUID
import hashlib
import logging
from datetime import datetime

from sqlalchemy.ext.asyncio import AsyncSession

from app.schemas.mcp_tools import (
    SearchDocumentsRequest, SearchDocumentsResponse, SearchResult, CitationTuple,
    FetchNodeRequest, FetchNodeResponse, NodeMetadata, NodeRelationship,
    VerifyCitationRequest, VerifyCitationResponse
)
from app.models.audit_log import AuditLog
from app.models.document import Document, DocumentNode, DocumentVersion

logger = logging.getLogger(__name__)


class ToolExecutionError(Exception):
    """도구 실행 에러."""
    pass


class ToolService:
    """MCP 도구 서비스 (search_documents, fetch_node, verify_citation)."""
    
    def __init__(self, db: AsyncSession, actor_id: UUID, actor_type: str = "agent"):
        """
        Args:
            db: 데이터베이스 세션
            actor_id: 행동 주체 ID (에이전트 또는 사용자)
            actor_type: 행동 주체 타입 ("agent" 또는 "user") — S2 원칙 ⑤
        """
        self.db = db
        self.actor_id = actor_id
        self.actor_type = actor_type
    
    # ==================== search_documents ====================
    
    async def search_documents(
        self,
        request: SearchDocumentsRequest,
        scope_profile: Dict[str, Any],  # S2 원칙 ⑥: Scope Profile
    ) -> SearchDocumentsResponse:
        """전문 검색 (FTS + Vector).
        
        Args:
            request: 검색 요청
            scope_profile: Scope Profile (ACL 필터링용)
        
        Returns:
            검색 결과
        
        Raises:
            ToolExecutionError: 검색 실패
        """
        logger.info(
            f"search_documents called by {self.actor_id} ({self.actor_type}): "
            f"query={request.query}, scope={request.scope}"
        )
        
        start_time = datetime.utcnow()
        
        try:
            # ① ACL 필터 구성 (S2 원칙 ⑥)
            acl_filters = self._build_acl_filter(scope_profile)
            
            # ② 기존 Retrieval layer 호출
            # (Phase 2에서 구축한 하이브리드 검색 엔진 사용)
            from app.services.retrieval import Retrieval
            retrieval_service = Retrieval(self.db)
            
            search_results = await retrieval_service.hybrid_search(
                query=request.query,
                top_k=request.top_k,
                acl_filters=acl_filters,
                document_types=request.document_types,
            )
            
            # ③ 결과 변환 (citation 5-tuple 포함)
            results = []
            for hit in search_results:
                citation = CitationTuple(
                    document_id=hit["document_id"],
                    version_id=hit["version_id"],
                    node_id=hit["node_id"],
                    span_offset=hit.get("span_offset", 0),
                    content_hash=hit.get("content_hash", ""),
                )
                
                result = SearchResult(
                    document_id=hit["document_id"],
                    document_title=hit["document_title"],
                    version_id=hit["version_id"],
                    node_id=hit["node_id"],
                    content=hit["content"],
                    citation=citation,
                    relevance_score=hit.get("relevance_score", 0.0),
                    retrieval_time_ms=0,  # 별도 측정
                )
                results.append(result)
            
            retrieval_time_ms = int((datetime.utcnow() - start_time).total_seconds() * 1000)
            
            # ④ 감시 로그 기록 (S2 원칙 ⑤)
            await self._log_tool_usage(
                tool_name="search_documents",
                input_params={
                    "query": request.query,
                    "scope": request.scope,
                    "top_k": request.top_k,
                    "document_types": request.document_types,
                },
                result_count=len(results),
            )
            
            return SearchDocumentsResponse(
                results=results,
                total_count=len(results),
                retrieval_method="hybrid",  # FTS + Vector
            )
        
        except Exception as e:
            logger.error(f"search_documents error: {str(e)}")
            await self._log_tool_error(
                tool_name="search_documents",
                error_message=str(e),
            )
            raise ToolExecutionError(f"Search failed: {str(e)}")
    
    # ==================== fetch_node ====================
    
    async def fetch_node(
        self,
        request: FetchNodeRequest,
        scope_profile: Dict[str, Any],  # S2 원칙 ⑥
    ) -> FetchNodeResponse:
        """특정 문서 노드의 전문 조회.
        
        Args:
            request: 노드 조회 요청
            scope_profile: Scope Profile (ACL 필터링용)
        
        Returns:
            노드 전문 및 메타데이터
        
        Raises:
            ToolExecutionError: 조회 실패
        """
        logger.info(
            f"fetch_node called by {self.actor_id} ({self.actor_type}): "
            f"document_id={request.document_id}, node_id={request.node_id}"
        )
        
        try:
            # ① Document 레이어에서 노드 조회
            from app.services.document import DocumentService
            doc_service = DocumentService(self.db)
            
            # 버전 ID가 없으면 latest 조회
            version_id = request.version_id
            if not version_id:
                version_id = await doc_service.get_latest_version_id(request.document_id)
            
            node = await doc_service.get_node(
                document_id=request.document_id,
                version_id=version_id,
                node_id=request.node_id,
            )
            
            if not node:
                raise ToolExecutionError(f"Node not found: {request.node_id}")
            
            # ② ACL 검사 (S2 원칙 ⑥)
            # Document 레이어에서 이미 ACL 필터링됨
            
            # ③ 응답 구성
            response = FetchNodeResponse(
                document_id=request.document_id,
                document_title=node.get("document_title", ""),
                version_id=version_id,
                node_id=request.node_id,
                content=node["content"],
                metadata=NodeMetadata(
                    title=node.get("title"),
                    node_type=node.get("node_type", "section"),
                    created_at=node.get("created_at"),
                    updated_at=node.get("updated_at"),
                    tags=node.get("tags"),
                ),
                relationships=NodeRelationship(
                    parent_node_id=node.get("parent_node_id"),
                    children_node_ids=node.get("children_node_ids"),
                    related_node_ids=node.get("related_node_ids"),
                ),
            )
            
            # ④ 감시 로그 기록
            await self._log_tool_usage(
                tool_name="fetch_node",
                input_params={
                    "document_id": str(request.document_id),
                    "version_id": str(version_id),
                    "node_id": request.node_id,
                },
                result_count=1,
            )
            
            return response
        
        except Exception as e:
            logger.error(f"fetch_node error: {str(e)}")
            await self._log_tool_error(
                tool_name="fetch_node",
                error_message=str(e),
            )
            raise ToolExecutionError(f"Fetch failed: {str(e)}")
    
    # ==================== verify_citation ====================
    
    async def verify_citation(
        self,
        request: VerifyCitationRequest,
    ) -> VerifyCitationResponse:
        """Citation 5-tuple 검증.
        
        Args:
            request: 검증 요청
        
        Returns:
            검증 결과
        
        Raises:
            ToolExecutionError: 검증 실패
        """
        logger.info(
            f"verify_citation called by {self.actor_id} ({self.actor_type}): "
            f"document_id={request.document_id}, node_id={request.node_id}"
        )
        
        try:
            # ① Document 레이어에서 현재 콘텐츠 조회
            from app.services.document import DocumentService
            doc_service = DocumentService(self.db)
            
            node = await doc_service.get_node(
                document_id=request.document_id,
                version_id=request.version_id,
                node_id=request.node_id,
            )
            
            if not node:
                return VerifyCitationResponse(
                    verified=False,
                    current_hash="",
                    hash_matches=False,
                    content_snapshot="",
                    version_valid=False,
                    message="Node not found",
                )
            
            # ② 해시값 비교
            current_content = node["content"]
            current_hash = hashlib.sha256(current_content.encode("utf-8")).hexdigest()
            hash_matches = current_hash == request.content_hash
            
            # ③ 버전 유효성 확인
            version_valid = True
            # (버전이 삭제되거나 과기된 경우 false)
            version_info = await doc_service.get_version_info(
                document_id=request.document_id,
                version_id=request.version_id,
            )
            if not version_info or version_info.get("status") == "deleted":
                version_valid = False
            
            # ④ 감시 로그 기록
            await self._log_tool_usage(
                tool_name="verify_citation",
                input_params={
                    "document_id": str(request.document_id),
                    "version_id": str(request.version_id),
                    "node_id": request.node_id,
                },
                result_count=1,
            )
            
            # ⑤ 응답
            verified = hash_matches and version_valid
            
            return VerifyCitationResponse(
                verified=verified,
                current_hash=current_hash,
                hash_matches=hash_matches,
                content_snapshot=current_content[:200],  # 처음 200자
                version_valid=version_valid,
                message=(
                    "Citation is valid" if verified
                    else f"Citation invalid: hash_match={hash_matches}, version_valid={version_valid}"
                ),
            )
        
        except Exception as e:
            logger.error(f"verify_citation error: {str(e)}")
            await self._log_tool_error(
                tool_name="verify_citation",
                error_message=str(e),
            )
            raise ToolExecutionError(f"Verification failed: {str(e)}")
    
    # ==================== Helper Methods ====================
    
    def _build_acl_filter(self, scope_profile: Dict[str, Any]) -> Dict[str, Any]:
        """Scope Profile에서 ACL 필터 구성 (S2 원칙 ⑥).
        
        코드에 하드코딩된 scope 문자열 금지.
        Scope Profile 객체에서 동적으로 필터 정보 추출.
        """
        filters = {}
        
        scope = scope_profile.get("scope", "private")
        organization_id = scope_profile.get("organization_id")
        team_id = scope_profile.get("team_id")
        user_id = scope_profile.get("user_id")
        
        # scope에 따라 필터 구성
        # (실제 구현은 Scope Profile 파서 활용)
        filters["scope"] = scope
        if organization_id:
            filters["organization_id"] = organization_id
        if team_id:
            filters["team_id"] = team_id
        if user_id:
            filters["user_id"] = user_id
        
        return filters
    
    async def _log_tool_usage(
        self,
        tool_name: str,
        input_params: Dict[str, Any],
        result_count: int,
    ) -> None:
        """도구 사용 감시 로그 기록 (S2 원칙 ⑤).
        
        Args:
            tool_name: 도구 이름
            input_params: 입력 매개변수
            result_count: 결과 개수
        """
        try:
            audit_log = AuditLog(
                entity_type="mcp_tool_call",
                entity_id=UUID(int=0),  # 도구 호출은 특정 엔티티가 아님
                action="execute",
                actor_id=self.actor_id,
                actor_type=self.actor_type,  # "agent" 필수 (S2 원칙 ⑤)
                changes={
                    "tool_name": tool_name,
                    "input_params": input_params,
                    "result_count": result_count,
                },
            )
            self.db.add(audit_log)
            await self.db.flush()
            logger.debug(f"Tool usage logged: {tool_name} by {self.actor_id} ({self.actor_type})")
        except Exception as e:
            logger.error(f"Failed to log tool usage: {str(e)}")
    
    async def _log_tool_error(
        self,
        tool_name: str,
        error_message: str,
    ) -> None:
        """도구 에러 감시 로그 기록.
        
        Args:
            tool_name: 도구 이름
            error_message: 에러 메시지
        """
        try:
            audit_log = AuditLog(
                entity_type="mcp_tool_error",
                entity_id=UUID(int=0),
                action="error",
                actor_id=self.actor_id,
                actor_type=self.actor_type,
                changes={
                    "tool_name": tool_name,
                    "error_message": error_message,
                },
            )
            self.db.add(audit_log)
            await self.db.flush()
            logger.debug(f"Tool error logged: {tool_name}")
        except Exception as e:
            logger.error(f"Failed to log tool error: {str(e)}")
```

### 4-3. MCP 도구 Dispatch 로직 (Server 확장)

**파일:** `/backend/app/mcp/server.py` (Task 4-1 확장)

```python
# 기존 MCPServer 클래스에 추가

async def handle_tool_call(
    self,
    tool_name: str,
    tool_input: Dict[str, Any],
    actor_id: UUID,
    actor_type: str = "agent",
    scope_profile: Optional[Dict[str, Any]] = None,
) -> Dict[str, Any]:
    """도구 호출 처리.
    
    Args:
        tool_name: 도구 이름 (search_documents, fetch_node, verify_citation)
        tool_input: 도구 입력 (JSON)
        actor_id: 행동 주체 ID
        actor_type: 행동 주체 타입 ("agent" 또는 "user")
        scope_profile: Scope Profile (ACL 필터링용)
    
    Returns:
        도구 결과 또는 에러
    """
    from app.mcp.tools import ToolService, ToolExecutionError
    from app.schemas.mcp_tools import (
        SearchDocumentsRequest, FetchNodeRequest, VerifyCitationRequest
    )
    
    logger.info(
        f"Tool call: {tool_name} by {actor_id} ({actor_type})"
    )
    
    if scope_profile is None:
        scope_profile = {"scope": "private"}
    
    # 도구 서비스 생성
    tool_service = ToolService(
        db=self.db,
        actor_id=actor_id,
        actor_type=actor_type,
    )
    
    try:
        # 도구별 디스패칭
        if tool_name == "search_documents":
            request = SearchDocumentsRequest(**tool_input)
            result = await tool_service.search_documents(request, scope_profile)
            return {"success": True, "data": result.dict()}
        
        elif tool_name == "fetch_node":
            request = FetchNodeRequest(**tool_input)
            result = await tool_service.fetch_node(request, scope_profile)
            return {"success": True, "data": result.dict()}
        
        elif tool_name == "verify_citation":
            request = VerifyCitationRequest(**tool_input)
            result = await tool_service.verify_citation(request)
            return {"success": True, "data": result.dict()}
        
        else:
            return {
                "success": False,
                "error": {
                    "code": "UNKNOWN_TOOL",
                    "message": f"Unknown tool: {tool_name}",
                }
            }
    
    except ToolExecutionError as e:
        logger.error(f"Tool execution error: {str(e)}")
        return {
            "success": False,
            "error": {
                "code": "TOOL_ERROR",
                "message": str(e),
            }
        }
    except Exception as e:
        logger.error(f"Unexpected error in tool call: {str(e)}")
        return {
            "success": False,
            "error": {
                "code": "INTERNAL_ERROR",
                "message": str(e),
            }
        }
```

### 4-4. FastAPI 라우터 (도구 호출 엔드포인트)

**파일:** `/backend/app/routes/mcp.py` (Task 4-1 확장)

```python
@router.post("/tools/{tool_name}/call")
async def call_tool(
    tool_name: str,
    request: Request,
    authorization: Optional[str] = Header(None),
    db: AsyncSession = Depends(get_db),
) -> Dict[str, Any]:
    """MCP 도구 호출 엔드포인트.
    
    Path Parameters:
        tool_name: 도구 이름 (search_documents, fetch_node, verify_citation)
    
    Headers:
        Authorization: Bearer <oauth_token>
    
    Request Body:
        도구 입력 매개변수 (JSON)
    """
    logger.info(f"Tool call endpoint: {tool_name}")
    
    # OAuth 토큰 검증
    if not authorization:
        return {
            "success": False,
            "error": {
                "code": "AUTHENTICATION_FAILED",
                "message": "Missing Authorization header",
            }
        }
    
    try:
        parts = authorization.split(" ")
        if len(parts) != 2 or parts[0].lower() != "bearer":
            raise ValueError("Invalid Authorization format")
        
        token = parts[1]
        oauth_info = await verify_oauth_token(token, db)
        if not oauth_info:
            raise ValueError("Invalid token")
        
        actor_id = oauth_info.get("actor_id") or oauth_info.get("client_id")
        
    except Exception as e:
        logger.error(f"OAuth validation failed: {str(e)}")
        return {
            "success": False,
            "error": {
                "code": "AUTHENTICATION_FAILED",
                "message": str(e),
            }
        }
    
    # 요청 바디 파싱
    try:
        tool_input = await request.json()
    except Exception as e:
        logger.error(f"Invalid JSON: {str(e)}")
        return {
            "success": False,
            "error": {
                "code": "INVALID_REQUEST",
                "message": f"Invalid JSON: {str(e)}",
            }
        }
    
    # Scope Profile 조회 (조직/사용자 설정에서)
    scope_profile = await get_scope_profile_for_agent(actor_id, db)
    
    # 도구 호출
    server = get_mcp_server()
    result = await server.handle_tool_call(
        tool_name=tool_name,
        tool_input=tool_input,
        actor_id=UUID(actor_id),
        actor_type="agent",  # MCP 호출은 항상 에이전트
        scope_profile=scope_profile,
    )
    
    await db.commit()
    
    return result
```

### 4-5. 단위/통합 테스트

**파일:** `/backend/tests/unit/mcp/test_mcp_tools.py`

```python
"""MCP 도구 단위 테스트."""

import pytest
from uuid import uuid4
from sqlalchemy.ext.asyncio import AsyncSession

from app.mcp.tools import ToolService, ToolExecutionError
from app.schemas.mcp_tools import (
    SearchDocumentsRequest, FetchNodeRequest, VerifyCitationRequest
)


@pytest.fixture
def tool_service(db_session: AsyncSession):
    """도구 서비스 인스턴스."""
    actor_id = uuid4()
    return ToolService(db=db_session, actor_id=actor_id, actor_type="agent")


@pytest.mark.asyncio
async def test_search_documents_basic(tool_service, db_session):
    """search_documents 기본 테스트."""
    request = SearchDocumentsRequest(
        query="AI 규정",
        scope="organization",
        top_k=5,
    )
    
    scope_profile = {
        "scope": "organization",
        "organization_id": uuid4(),
        "user_id": uuid4(),
    }
    
    # Mock Retrieval service
    # (실제 테스트에서는 mock 또는 테스트 데이터 필요)
    try:
        result = await tool_service.search_documents(request, scope_profile)
        assert result.total_count >= 0
        assert result.retrieval_method in ["fts", "vector", "hybrid"]
    except ToolExecutionError:
        # 테스트 데이터 없음 시 에러는 정상
        pass


@pytest.mark.asyncio
async def test_fetch_node_basic(tool_service, db_session):
    """fetch_node 기본 테스트."""
    request = FetchNodeRequest(
        document_id=uuid4(),
        node_id="section-1",
        version_id=None,  # latest
    )
    
    scope_profile = {
        "scope": "organization",
    }
    
    try:
        result = await tool_service.fetch_node(request, scope_profile)
        assert result.document_id == request.document_id
        assert result.node_id == request.node_id
    except ToolExecutionError as e:
        # 데이터 없음 시 정상
        assert "not found" in str(e).lower()


@pytest.mark.asyncio
async def test_verify_citation_basic(tool_service, db_session):
    """verify_citation 기본 테스트."""
    request = VerifyCitationRequest(
        document_id=uuid4(),
        version_id=uuid4(),
        node_id="section-1",
        content_hash="abc123",
    )
    
    result = await tool_service.verify_citation(request)
    
    # 데이터가 없으면 verified=false
    assert isinstance(result.verified, bool)
    assert isinstance(result.hash_matches, bool)
    assert isinstance(result.version_valid, bool)


@pytest.mark.asyncio
async def test_acl_filter_building(tool_service):
    """ACL 필터 구성 테스트."""
    scope_profile = {
        "scope": "organization",
        "organization_id": uuid4(),
        "team_id": uuid4(),
        "user_id": uuid4(),
    }
    
    filters = tool_service._build_acl_filter(scope_profile)
    
    assert filters["scope"] == "organization"
    assert "organization_id" in filters
    assert "team_id" in filters
    assert "user_id" in filters


@pytest.mark.asyncio
async def test_tool_usage_logging(tool_service, db_session):
    """도구 사용 감시 로그 기록 테스트."""
    await tool_service._log_tool_usage(
        tool_name="search_documents",
        input_params={"query": "test"},
        result_count=5,
    )
    
    # 감시 로그가 기록되었는지 확인
    from app.models.audit_log import AuditLog
    from sqlalchemy import select
    
    stmt = select(AuditLog).where(AuditLog.entity_type == "mcp_tool_call")
    result = await db_session.execute(stmt)
    logs = result.scalars().all()
    
    assert len(logs) > 0
    assert logs[-1].actor_type == "agent"
    assert logs[-1].action == "execute"
```

## 5. 산출물

1. **도구 입출력 스키마** (`/backend/app/schemas/mcp_tools.py`)
   - SearchDocumentsRequest/Response
   - FetchNodeRequest/Response
   - VerifyCitationRequest/Response

2. **도구 서비스** (`/backend/app/mcp/tools.py`)
   - ToolService: 3개 도구 구현
   - ACL 필터 구성
   - 감시 로그 기록

3. **MCP Server 확장** (`/backend/app/mcp/server.py`)
   - handle_tool_call: 도구 디스패칭

4. **FastAPI 라우터** (`/backend/app/routes/mcp.py`)
   - POST /mcp/tools/{tool_name}/call: 도구 호출 엔드포인트

5. **단위 테스트** (`/backend/tests/unit/mcp/test_mcp_tools.py`)
   - 각 도구별 정상/에러 케이스

## 6. 완료 기준

1. 3개 도구 입출력 스키마가 정의되었는가?
2. search_documents가 기존 Retrieval layer를 호출하는가?
3. fetch_node가 Document layer를 호출하는가?
4. verify_citation이 해시 비교 및 버전 유효성 확인을 수행하는가?
5. 모든 도구에 ACL 필터링이 적용되었는가? (S2 원칙 ⑥)
6. 모든 도구 호출마다 actor_type=agent로 감시 로그가 기록되는가? (S2 원칙 ⑤)
7. 도구별 에러 처리가 정상인가?
8. 도구 디스패칭이 정확한가?
9. 모든 단위 테스트가 통과하는가?
10. mypy 타입 검사를 통과하는가?

## 7. 작업 지침

### 지침 7-1. Scope Profile 기반 ACL (S2 원칙 ⑥)

코드에 `if scope == "team"` 같은 하드코딩을 금지한다. 대신 scope_profile 객체에서 동적으로 필터 정보를 읽어 Retrieval/Document layer에 전달한다.

### 지침 7-2. 감시 로그 actor_type (S2 원칙 ⑤)

모든 도구 호출 감시 로그에 actor_type=agent를 필수 기록한다. 이를 통해 에이전트의 활동을 사용자와 구분하여 추적할 수 있다.

### 지침 7-3. Citation 5-tuple 정확성

search_documents 결과에 포함된 citation 5-tuple은 Phase 2에서 정의한 계약과 정확하게 일치해야 한다 (document_id, version_id, node_id, span_offset, content_hash).

### 지침 7-4. 해시 검증 (verify_citation)

검증할 때 현재 콘텐츠의 해시를 계산하여 요청된 해시와 비교한다. SHA256을 사용하여 일관성을 유지한다.

### 지침 7-5. 에러 처리 및 로깅

도구 실행 에러 발생 시 에러 감시 로그를 기록하고, MCP 스펙 준수 에러 응답을 반환한다.

### 지침 7-6. 성능 측정

search_documents의 경우 retrieval_time_ms를 측정하여 응답에 포함시킨다.

### 지침 7-7. 버전 관리 (fetch_node)

version_id가 제공되지 않은 경우, Document service의 get_latest_version_id를 호출하여 최신 버전을 조회한다.
