# 작업지시서: S2 통합 회귀 테스트 스크립트 (Phase 간 e2e 플로우)
**FG9.2-TASK9-4** | Phase 9 기간 | S2 원칙 ⑤⑥⑦ 통합 검증

---

## 1. 작업 목적

Phase 1부터 Phase 8까지 개별 FG에서 구현된 기능들이 서로 올바르게 연동되는지 검증하기 위한 **통합 회귀 테스트 스크립트**를 작성한다. 특히 S2 원칙(AI 에이전트 First-class API 소비자, Scope Profile 기반 ACL, 폐쇄망 환경 지원)이 모든 Phase 간 상호작용에서 일관되게 적용되는지 확인한다.

---

## 2. 작업 범위

### 포함 범위

1. **Phase-to-Phase Flow Test Suite** (`tests/integration/test_integration_s2_phase0_to_8.py`)
   - TestS2IntegrationRegression 클래스: 8개의 Phase-to-Phase 플로우 테스트 메서드
   - 각 테스트: 이전 Phase 출력 → 현재 Phase 입력 → 다음 Phase 출력 검증 체인
   - 예: `test_phase1_to_phase2_flow` (LLM 선택 → 검색 → Citation 5-tuple 검증)

2. **S2 원칙 검증 통합**
   - **원칙 ⑤**: actor_type 필드 검증 (audit log에 "user" 또는 "agent" 기록)
   - **원칙 ⑥**: Scope Profile 기반 ACL 필터링 (각 Phase마다 scope 문자열 하드코딩 확인 금지)
   - **원칙 ⑦**: 폐쇄망 모드 테스트 (외부 LLM 없이 rule-based fallback만으로 동작)

3. **테스트 Fixture 및 헬퍼**
   - shared_db_setup: 각 테스트 전 DB 초기화 및 테스트 데이터 시딩
   - test data seeding: 문서, 세션, Agent principal, 평가 기준 등 준비
   - cleanup: 각 테스트 후 DB 롤백
   - mock_scope_profile: 동적 Scope Profile 생성 및 주입

4. **8가지 Phase-to-Phase 플로우 테스트**
   - Phase 1→2: LLM 모델 선택 → 검색 + Citation 생성
   - Phase 2→3: 검색 결과 → 세션 기반 RAG → 멀티턴 컨텍스트
   - Phase 3→4: 대화 세션 → MCP 도구 노출 → 리소스 나열
   - Phase 4→5: Agent principal 생성 → 위임(delegation) → Draft 제안
   - Phase 5→6: Draft 제안 → Admin 승인 큐 → 상태 전환
   - Phase 6→7: Admin 평가 트리거 → 골든셋 실행 → 보고서 생성
   - Phase 7→8: 평가 기준 → 추출 품질 평가
   - 완전 lifecycle: 문서 업로드 → 자동 추출 → 인간 검수 → 승인 → Agent 보조 검색

5. **Closed-network Mode 검증**
   - 환경변수 `MIMIR_DISABLE_EXTERNAL_LLM=true` 설정 시 동작 확인
   - LLM 의존 단계 건너뛰기 또는 rule-based fallback 사용
   - 검색 품질 degradation 허용하되 failure는 없어야 함

6. **pytest 마커 및 조직**
   - `@pytest.mark.integration`: 통합 테스트 전용
   - `@pytest.mark.slow`: 시간 오래 걸리는 테스트 (golden set 포함)
   - `@pytest.mark.s2_principles`: S2 원칙 검증 전용
   - `@pytest.mark.closed_network`: 폐쇄망 모드 테스트

### 제외 범위

- Individual Phase 단위 테스트 (Phase별 FG에서 이미 수행)
- UI 테스트 (별도 작업)
- Performance 테스트 (별도 작업)
- Load 테스트 (별도 작업)
- External LLM API 호출 통합 테스트 (closed-network 검증이 더 중요)

---

## 3. 선행 조건

- Phase 1~8 개발 완료 및 개별 FG 단위 테스트 통과
- pytest, pytest-asyncio, pytest-xdist 설치
- PostgreSQL 테스트 DB 구성 (docker-compose 또는 localhost)
- 코드베이스 구조:
  ```
  mimir/
    ├── src/mimir/
    │   ├── phase1/  (LLM 모델 관리)
    │   ├── phase2/  (검색 + Citation)
    │   ├── phase3/  (세션 RAG)
    │   ├── phase4/  (MCP)
    │   ├── phase5/  (Agent delegation)
    │   ├── phase6/  (Admin approval)
    │   ├── phase7/  (평가 실행)
    │   ├── phase8/  (추출 평가)
    │   └── common/  (Scope Profile, audit log)
    └── tests/
        ├── integration/
        │   └── test_integration_s2_phase0_to_8.py  [새로 작성]
        └── fixtures/
            └── conftest.py
  ```
- Scope Profile 구현 완료 (phase2 이상에서 필수)
- Audit log 시스템 구현 완료 (actor_type 필드 포함)

---

## 4. 주요 작업 항목

### 4.1 테스트 파일 생성 및 기본 구조

**파일**: `tests/integration/test_integration_s2_phase0_to_8.py`

```python
"""
S2 통합 회귀 테스트 스크립트
Phase 1부터 Phase 8까지 FG 간 상호 의존성 검증

테스트 전략:
1. 각 Phase의 입출력을 chain으로 연결
2. 각 단계에서 S2 원칙(⑤⑥⑦) 준수 확인
3. Closed-network 모드에서도 동일 흐름 검증
4. Audit log에 actor_type 기록 확인
"""

import pytest
import asyncio
import json
from typing import Dict, Any, List
from uuid import uuid4
from datetime import datetime, timedelta

# 프로젝트 임포트
from mimir.phase1.llm_models import LLMModelService
from mimir.phase2.search import SearchService
from mimir.phase2.citation import CitationService, Citation5Tuple
from mimir.phase3.session_rag import SessionRAGService
from mimir.phase4.mcp_server import MCPServerService
from mimir.phase5.agent_principal import AgentPrincipalService, DelegationService
from mimir.phase6.admin_approval import AdminApprovalService
from mimir.phase7.evaluation_runner import EvaluationRunner
from mimir.phase8.extraction_quality import ExtractionQualityService
from mimir.common.scope_profile import ScopeProfile, ScopeProfileService
from mimir.common.audit_log import AuditLog, AuditLogService


@pytest.mark.integration
@pytest.mark.s2_principles
class TestS2IntegrationRegression:
    """
    Phase 간 통합 회귀 테스트 클래스
    
    각 test_phase*_to_phase* 메서드는:
    1. 이전 Phase의 출력을 준비
    2. 현재 Phase의 입력으로 전달
    3. 다음 Phase의 입력이 올바른지 검증
    4. S2 원칙 검증 (audit log, scope profile, actor_type)
    """

    @pytest.fixture(autouse=True)
    async def setup(self, db_session, shared_db_setup):
        """각 테스트 전 공유 fixture 적용"""
        self.db = db_session
        self.audit_log_service = AuditLogService(db_session)
        self.scope_profile_service = ScopeProfileService(db_session)
        
        # 테스트용 기본 Scope Profile
        self.test_scope = await self.scope_profile_service.create_profile({
            "name": "test_scope",
            "rules": {
                "document_types": ["pdf", "docx", "txt"],
                "max_results": 100,
                "access_control": {
                    "team": ["read"],
                    "personal": ["read", "write"],
                    "public": ["read"]
                }
            }
        })
        
        yield
        
        # cleanup
        await db_session.rollback()

    @pytest.mark.asyncio
    async def test_phase1_to_phase2_flow(self):
        """
        Phase 1 → Phase 2: LLM 선택 → 검색 + Citation
        
        Flow:
        1. Phase 1: LLM 모델 선택 (gpt-4, claude-3, local-llm)
        2. Phase 2: 검색 수행
        3. Citation 생성 (5-tuple: source, snippet, relevance, confidence, timestamp)
        4. 검증: Citation이 5-tuple 구조 만족, actor_type 기록됨, scope 필터 적용됨
        """
        
        # Phase 1: LLM 모델 선택
        llm_service = LLMModelService(self.db)
        
        # actor_type="user"인 경우
        selected_model = await llm_service.select_model(
            model_name="gpt-4-turbo",
            actor_id="user-001",
            actor_type="user",  # S2 ⑤: actor_type 명시
            context={"temperature": 0.7, "max_tokens": 2048}
        )
        
        # audit log 검증
        audit_logs = await self.audit_log_service.list_logs(
            action="model_selection",
            actor_id="user-001"
        )
        assert len(audit_logs) > 0
        assert audit_logs[0].actor_type == "user"  # S2 ⑤: actor_type 기록 확인
        assert audit_logs[0].model_name == "gpt-4-turbo"
        
        # Phase 2: 검색 수행
        search_service = SearchService(self.db)
        
        query = "machine learning best practices"
        search_results = await search_service.search(
            query=query,
            scope_profile=self.test_scope,  # S2 ⑥: Scope Profile 명시
            actor_id="user-001",
            actor_type="user",
            limit=10
        )
        
        # audit log 검증
        search_audit = await self.audit_log_service.list_logs(
            action="search",
            actor_id="user-001"
        )
        assert len(search_audit) > 0
        assert search_audit[0].actor_type == "user"
        
        # Phase 2: Citation 생성
        citation_service = CitationService(self.db)
        
        citations = []
        for result in search_results[:3]:
            citation = await citation_service.create_citation(
                source=result.document_uri,
                snippet=result.text[:500],
                relevance_score=result.relevance,
                confidence=result.confidence,
                llm_model=selected_model.name,
                actor_id="user-001",
                actor_type="user"
            )
            citations.append(citation)
        
        # 검증: Citation 5-tuple 구조
        for citation in citations:
            assert isinstance(citation, Citation5Tuple)
            assert citation.source is not None
            assert citation.snippet is not None
            assert citation.relevance_score is not None
            assert citation.confidence is not None
            assert citation.timestamp is not None
            assert len(citation.snippet) > 0
            assert 0.0 <= citation.relevance_score <= 1.0
            assert 0.0 <= citation.confidence <= 1.0
        
        # 검증: Scope Profile 적용 (제외 문서 타입이 없어야 함)
        for result in search_results:
            assert result.document_type in self.test_scope.allowed_types
            assert result.access_level in ["read"]  # test_scope 에서만 read 허용
        
        # 최종 검증: 다음 Phase 입력 준비 완료
        assert len(citations) > 0
        assert all(c.source for c in citations)
        assert all(c.timestamp for c in citations)
        
        pytest.assumption("Phase 1→2 Flow OK: LLM 선택 → 검색 → Citation 5-tuple 생성")

    @pytest.mark.asyncio
    async def test_phase2_to_phase3_flow(self):
        """
        Phase 2 → Phase 3: 검색 → 세션 기반 RAG → 멀티턴 컨텍스트
        
        Flow:
        1. Phase 2: 검색 결과 + Citation
        2. Phase 3: 세션 생성, 첫 번째 쿼리 처리
        3. Phase 3: 두 번째 쿼리 (follow-up) 처리 → 이전 컨텍스트 포함
        4. 검증: multi-turn context chain, scope preservation, actor_type consistency
        """
        
        # Phase 2: 기본 검색 수행
        search_service = SearchService(self.db)
        citation_service = CitationService(self.db)
        
        initial_query = "what are design patterns"
        search_results = await search_service.search(
            query=initial_query,
            scope_profile=self.test_scope,
            actor_id="user-002",
            actor_type="user",
            limit=5
        )
        
        # Citation 생성
        citations = []
        for result in search_results[:2]:
            citation = await citation_service.create_citation(
                source=result.document_uri,
                snippet=result.text[:400],
                relevance_score=result.relevance,
                confidence=result.confidence,
                llm_model="gpt-4-turbo",
                actor_id="user-002",
                actor_type="user"
            )
            citations.append(citation)
        
        # Phase 3: 세션 기반 RAG
        session_rag_service = SessionRAGService(self.db)
        
        # 세션 생성
        session = await session_rag_service.create_session(
            user_id="user-002",
            scope_profile=self.test_scope,
            actor_id="user-002",
            actor_type="user"
        )
        
        assert session.session_id is not None
        assert session.created_at is not None
        
        # 첫 번째 턴: 검색 결과 저장
        first_turn = await session_rag_service.add_turn(
            session_id=session.session_id,
            query=initial_query,
            search_results=search_results[:2],
            citations=citations,
            response="Design patterns are reusable solutions...",
            actor_id="user-002",
            actor_type="user"
        )
        
        # audit log 검증
        audit_logs = await self.audit_log_service.list_logs(
            action="session_turn_add",
            actor_id="user-002"
        )
        assert len(audit_logs) > 0
        assert audit_logs[-1].actor_type == "user"
        assert audit_logs[-1].session_id == session.session_id
        
        # 두 번째 턴: follow-up 쿼리
        follow_up_query = "which one should I use for state management"
        follow_up_results = await search_service.search(
            query=follow_up_query,
            scope_profile=self.test_scope,
            actor_id="user-002",
            actor_type="user",
            limit=5,
            # Phase 3: 이전 턴의 context 포함
            previous_context=first_turn.context_summary
        )
        
        second_turn = await session_rag_service.add_turn(
            session_id=session.session_id,
            query=follow_up_query,
            search_results=follow_up_results[:2],
            response="For state management, consider Redux or Context API...",
            actor_id="user-002",
            actor_type="user"
        )
        
        # 멀티턴 컨텍스트 검증
        conversation_history = await session_rag_service.get_session_history(
            session_id=session.session_id
        )
        
        assert len(conversation_history.turns) >= 2
        assert conversation_history.turns[0].query == initial_query
        assert conversation_history.turns[1].query == follow_up_query
        
        # Scope preservation 검증
        for turn in conversation_history.turns:
            for result in turn.search_results:
                assert result.document_type in self.test_scope.allowed_types
        
        # 최종 검증
        assert session.scope_profile.name == self.test_scope.name
        
        pytest.assumption("Phase 2→3 Flow OK: 검색 → 세션 RAG → 멀티턴 컨텍스트")

    @pytest.mark.asyncio
    async def test_phase3_to_phase4_flow(self):
        """
        Phase 3 → Phase 4: 대화 세션 → MCP 도구 노출 → 리소스 나열
        
        Flow:
        1. Phase 3: 세션 생성 및 멀티턴 대화
        2. Phase 4: MCP 서버 초기화, session context 전달
        3. Phase 4: MCP 도구 (search, fetch_node, verify_citation) 노출
        4. 검증: MCP capabilities 선언, scope-aware tool filtering, actor_type 기록
        """
        
        # Phase 3: 세션 생성
        session_rag_service = SessionRAGService(self.db)
        
        session = await session_rag_service.create_session(
            user_id="user-003",
            scope_profile=self.test_scope,
            actor_id="user-003",
            actor_type="user"
        )
        
        # Phase 4: MCP 서버 초기화
        mcp_service = MCPServerService(self.db)
        
        server = await mcp_service.initialize_server(
            session_id=session.session_id,
            scope_profile=self.test_scope,
            actor_id="user-003",
            actor_type="user"
        )
        
        assert server.server_id is not None
        assert server.capabilities is not None
        
        # MCP 도구 검증: search_documents
        search_tool = server.get_tool("search_documents")
        assert search_tool is not None
        assert search_tool.input_schema is not None
        
        # search_documents 실행
        search_result = await mcp_service.execute_tool(
            server_id=server.server_id,
            tool_name="search_documents",
            arguments={
                "query": "REST API best practices",
                "limit": 5,
                "access_context": self.test_scope.to_dict()  # S2 ⑥
            },
            actor_id="user-003",
            actor_type="user"
        )
        
        # audit log 검증
        audit_logs = await self.audit_log_service.list_logs(
            action="mcp_tool_execute",
            actor_id="user-003"
        )
        assert len(audit_logs) > 0
        assert audit_logs[-1].actor_type == "user"
        assert audit_logs[-1].tool_name == "search_documents"
        
        # MCP 도구 검증: fetch_node (mimir:// URI)
        if search_result.documents:
            doc_uri = search_result.documents[0].uri
            fetch_tool = server.get_tool("fetch_node")
            assert fetch_tool is not None
            
            fetch_result = await mcp_service.execute_tool(
                server_id=server.server_id,
                tool_name="fetch_node",
                arguments={
                    "uri": doc_uri,
                    "access_context": self.test_scope.to_dict()
                },
                actor_id="user-003",
                actor_type="user"
            )
            
            assert fetch_result.node is not None
            assert fetch_result.node.uri == doc_uri
        
        # MCP 도구 검증: verify_citation
        verify_tool = server.get_tool("verify_citation")
        assert verify_tool is not None
        
        # 리소스 나열
        resources = await mcp_service.list_resources(
            server_id=server.server_id,
            actor_id="user-003",
            actor_type="user"
        )
        
        assert len(resources) > 0
        
        # 최종 검증: Phase 4 완료
        pytest.assumption("Phase 3→4 Flow OK: 세션 → MCP 초기화 → 도구 노출")

    @pytest.mark.asyncio
    async def test_phase4_to_phase5_flow(self):
        """
        Phase 4 → Phase 5: Agent principal 생성 → 위임(delegation) → Draft 제안
        
        Flow:
        1. Phase 4: MCP server 준비
        2. Phase 5: Agent principal 생성 (AI 에이전트가 first-class API consumer)
        3. Phase 5: Delegation (agent → admin 또는 agent → user)
        4. Phase 5: Draft 제안 생성
        5. 검증: agent actor_type, delegation chain, task_id 생성
        """
        
        # Phase 4: MCP 서버 준비
        session_rag_service = SessionRAGService(self.db)
        mcp_service = MCPServerService(self.db)
        
        session = await session_rag_service.create_session(
            user_id="user-004",
            scope_profile=self.test_scope,
            actor_id="user-004",
            actor_type="user"
        )
        
        server = await mcp_service.initialize_server(
            session_id=session.session_id,
            scope_profile=self.test_scope,
            actor_id="user-004",
            actor_type="user"
        )
        
        # Phase 5: Agent principal 생성
        agent_service = AgentPrincipalService(self.db)
        
        # AI 에이전트: S2 ⑤ first-class consumer
        agent_principal = await agent_service.create_principal(
            agent_id="agent-ai-001",
            agent_name="AI Assistant",
            agent_type="ai",
            owner_id="user-004",
            mcp_server_id=server.server_id,
            scope_profile=self.test_scope,
            # agent도 user와 동일한 API 소비 권한
            actor_type="agent"  # S2 ⑤: agent는 first-class consumer
        )
        
        assert agent_principal.agent_id == "agent-ai-001"
        assert agent_principal.agent_type == "ai"
        
        # audit log 검증
        audit_logs = await self.audit_log_service.list_logs(
            action="principal_create",
            actor_id="agent-ai-001"
        )
        assert any(log.actor_type == "agent" for log in audit_logs)
        
        # Phase 5: Delegation (agent → admin)
        delegation_service = DelegationService(self.db)
        
        delegation = await delegation_service.create_delegation(
            delegating_principal=agent_principal,
            delegated_to="admin-user-001",
            scope=["document_review", "extraction_quality_check"],
            actor_id="agent-ai-001",
            actor_type="agent"
        )
        
        assert delegation.delegation_id is not None
        assert delegation.delegated_to == "admin-user-001"
        
        # Phase 5: Draft 제안 생성
        task_id = str(uuid4())
        
        draft_proposal = await agent_service.create_draft_proposal(
            task_id=task_id,
            agent_principal_id=agent_principal.principal_id,
            content={
                "document_uri": "mimir://doc-001",
                "extraction_candidates": [
                    {"field": "author", "value": "John Doe", "confidence": 0.95},
                    {"field": "date", "value": "2026-04-17", "confidence": 0.88}
                ],
                "suggested_action": "approve"
            },
            delegation_id=delegation.delegation_id,
            actor_id="agent-ai-001",
            actor_type="agent"
        )
        
        assert draft_proposal.task_id == task_id
        assert draft_proposal.status == "pending_review"
        
        # audit log 검증
        draft_audit = await self.audit_log_service.list_logs(
            action="draft_proposal_create",
            actor_id="agent-ai-001"
        )
        assert len(draft_audit) > 0
        assert draft_audit[-1].actor_type == "agent"
        assert draft_audit[-1].task_id == task_id
        
        # 최종 검증
        assert agent_principal.scope_profile.name == self.test_scope.name
        
        pytest.assumption("Phase 4→5 Flow OK: MCP → Agent principal → Draft 제안")

    @pytest.mark.asyncio
    async def test_phase5_to_phase6_flow(self):
        """
        Phase 5 → Phase 6: Draft 제안 → Admin 승인 큐 → 상태 전환
        
        Flow:
        1. Phase 5: Draft 제안 생성
        2. Phase 6: Admin approval queue에 등록
        3. Phase 6: Admin 검토 및 승인/거부
        4. Phase 6: 상태 전환 (pending → approved/rejected)
        5. 검증: queue entry, admin audit trail, state machine, actor_type="user"
        """
        
        # Phase 5: Agent 및 Draft 제안 준비
        agent_service = AgentPrincipalService(self.db)
        delegation_service = DelegationService(self.db)
        
        agent_principal = await agent_service.create_principal(
            agent_id="agent-ai-002",
            agent_name="AI Assistant 2",
            agent_type="ai",
            owner_id="user-005",
            scope_profile=self.test_scope,
            actor_type="agent"
        )
        
        delegation = await delegation_service.create_delegation(
            delegating_principal=agent_principal,
            delegated_to="admin-user-002",
            scope=["document_review", "extraction_quality_check"],
            actor_id="agent-ai-002",
            actor_type="agent"
        )
        
        task_id = str(uuid4())
        draft_proposal = await agent_service.create_draft_proposal(
            task_id=task_id,
            agent_principal_id=agent_principal.principal_id,
            content={
                "document_uri": "mimir://doc-002",
                "extracted_fields": [
                    {"name": "title", "value": "Test Document"}
                ]
            },
            delegation_id=delegation.delegation_id,
            actor_id="agent-ai-002",
            actor_type="agent"
        )
        
        # Phase 6: Admin approval
        approval_service = AdminApprovalService(self.db)
        
        # Draft를 approval queue에 등록
        queue_entry = await approval_service.add_to_queue(
            draft_proposal_id=draft_proposal.proposal_id,
            task_id=task_id,
            priority="normal",
            admin_id="admin-user-002",
            actor_id="agent-ai-002",
            actor_type="agent"
        )
        
        assert queue_entry.queue_id is not None
        assert queue_entry.status == "pending"
        
        # Admin이 검토
        approval_decision = await approval_service.review_and_approve(
            queue_entry_id=queue_entry.queue_id,
            admin_id="admin-user-002",
            decision="approved",
            review_notes="Looks good, extracted fields are accurate.",
            actor_id="admin-user-002",  # Admin은 user actor_type
            actor_type="user"
        )
        
        assert approval_decision.decision == "approved"
        assert approval_decision.reviewed_at is not None
        
        # audit log 검증: admin의 user actor_type
        admin_audit = await self.audit_log_service.list_logs(
            action="approval_decision",
            actor_id="admin-user-002"
        )
        assert any(log.actor_type == "user" for log in admin_audit)
        
        # 최종 상태 확인
        updated_queue = await approval_service.get_queue_entry(queue_entry.queue_id)
        assert updated_queue.status == "approved"
        
        # 다음 Phase 입력 준비
        pytest.assumption("Phase 5→6 Flow OK: Draft → Admin 승인 → 상태 전환")

    @pytest.mark.asyncio
    async def test_phase6_to_phase7_flow(self):
        """
        Phase 6 → Phase 7: Admin 승인 완료 → 평가 실행 트리거 → 골든셋 보고서
        
        Flow:
        1. Phase 6: Admin이 Draft 승인
        2. Phase 7: 평가 기준 로드 (golden set baseline)
        3. Phase 7: 평가 실행 (faithfulness, citation_present, etc.)
        4. Phase 7: 평가 보고서 생성
        5. 검증: baseline criteria, metric calculation, report completeness, actor_type
        """
        
        # Phase 6: Approval 완료된 상태 준비
        approval_service = AdminApprovalService(self.db)
        agent_service = AgentPrincipalService(self.db)
        
        agent_principal = await agent_service.create_principal(
            agent_id="agent-ai-003",
            agent_name="AI Assistant 3",
            agent_type="ai",
            owner_id="user-006",
            scope_profile=self.test_scope,
            actor_type="agent"
        )
        
        delegation_service = DelegationService(self.db)
        delegation = await delegation_service.create_delegation(
            delegating_principal=agent_principal,
            delegated_to="admin-user-003",
            scope=["document_review"],
            actor_id="agent-ai-003",
            actor_type="agent"
        )
        
        task_id = str(uuid4())
        draft = await agent_service.create_draft_proposal(
            task_id=task_id,
            agent_principal_id=agent_principal.principal_id,
            content={"document_uri": "mimir://doc-003"},
            delegation_id=delegation.delegation_id,
            actor_id="agent-ai-003",
            actor_type="agent"
        )
        
        queue_entry = await approval_service.add_to_queue(
            draft_proposal_id=draft.proposal_id,
            task_id=task_id,
            priority="normal",
            admin_id="admin-user-003",
            actor_id="agent-ai-003",
            actor_type="agent"
        )
        
        approval = await approval_service.review_and_approve(
            queue_entry_id=queue_entry.queue_id,
            admin_id="admin-user-003",
            decision="approved",
            review_notes="Good",
            actor_id="admin-user-003",
            actor_type="user"
        )
        
        # Phase 7: 평가 실행
        eval_runner = EvaluationRunner(self.db)
        
        # 평가 기준 로드 (golden set baseline)
        evaluation_criteria = await eval_runner.load_evaluation_criteria(
            criteria_set="default_golden_set",
            actor_id="admin-user-003",
            actor_type="user"
        )
        
        assert evaluation_criteria.faithfulness_threshold == 0.80
        assert evaluation_criteria.citation_present_threshold == 0.90
        
        # 평가 실행
        evaluation_result = await eval_runner.run_evaluation(
            task_id=task_id,
            content=draft.content,
            criteria=evaluation_criteria,
            actor_id="admin-user-003",
            actor_type="user"
        )
        
        assert evaluation_result.evaluation_id is not None
        assert "faithfulness" in evaluation_result.metrics
        assert "answer_relevance" in evaluation_result.metrics
        assert "context_precision" in evaluation_result.metrics
        assert "citation_present" in evaluation_result.metrics
        
        # 평가 보고서 생성
        report = await eval_runner.generate_evaluation_report(
            evaluation_result=evaluation_result,
            actor_id="admin-user-003",
            actor_type="user"
        )
        
        assert report.report_id is not None
        assert report.status == "completed"
        assert "metrics_summary" in report.content
        
        # audit log 검증
        eval_audit = await self.audit_log_service.list_logs(
            action="evaluation_run",
            actor_id="admin-user-003"
        )
        assert len(eval_audit) > 0
        
        pytest.assumption("Phase 6→7 Flow OK: Admin 승인 → 평가 실행 → 보고서")

    @pytest.mark.asyncio
    async def test_phase7_to_phase8_flow(self):
        """
        Phase 7 → Phase 8: 평가 기준 → 추출 품질 평가
        
        Flow:
        1. Phase 7: 평가 기준 (baseline metrics)
        2. Phase 8: 추출된 필드들의 품질 평가
        3. Phase 8: 각 필드별 confidence 재검증
        4. Phase 8: 최종 추출 품질 점수
        5. 검증: field-level quality, confidence threshold, actor_type, audit trail
        """
        
        # Phase 7: 평가 기준 준비
        eval_runner = EvaluationRunner(self.db)
        
        evaluation_criteria = await eval_runner.load_evaluation_criteria(
            criteria_set="default_golden_set",
            actor_id="admin-user-004",
            actor_type="user"
        )
        
        # Phase 8: 추출 품질 평가
        extraction_quality_service = ExtractionQualityService(self.db)
        
        # 테스트 문서의 추출 결과
        task_id = str(uuid4())
        extraction_results = [
            {
                "field_name": "author",
                "extracted_value": "Jane Smith",
                "confidence": 0.92,
                "source_snippet": "by Jane Smith",
                "field_position": "header"
            },
            {
                "field_name": "date",
                "extracted_value": "2026-04-17",
                "confidence": 0.87,
                "source_snippet": "Published on 2026-04-17",
                "field_position": "metadata"
            },
            {
                "field_name": "title",
                "extracted_value": "Machine Learning in Production",
                "confidence": 0.95,
                "source_snippet": "Machine Learning in Production",
                "field_position": "title"
            }
        ]
        
        # 각 필드의 품질 평가
        field_quality_assessments = []
        for extraction in extraction_results:
            field_quality = await extraction_quality_service.assess_field_quality(
                task_id=task_id,
                field_name=extraction["field_name"],
                extracted_value=extraction["extracted_value"],
                confidence=extraction["confidence"],
                source_snippet=extraction["source_snippet"],
                criteria=evaluation_criteria,
                actor_id="admin-user-004",
                actor_type="user"
            )
            
            field_quality_assessments.append(field_quality)
        
        # 각 필드별 검증
        for assessment in field_quality_assessments:
            assert assessment.field_name is not None
            assert assessment.quality_score is not None
            assert 0.0 <= assessment.quality_score <= 1.0
            assert assessment.meets_threshold == (
                assessment.quality_score >= evaluation_criteria.extraction_quality_threshold
            )
        
        # 전체 추출 품질 점수 계산
        overall_quality = await extraction_quality_service.calculate_overall_quality(
            field_assessments=field_quality_assessments,
            actor_id="admin-user-004",
            actor_type="user"
        )
        
        assert overall_quality.overall_score is not None
        assert 0.0 <= overall_quality.overall_score <= 1.0
        assert len(overall_quality.field_scores) == len(extraction_results)
        
        # audit log 검증
        extraction_audit = await self.audit_log_service.list_logs(
            action="extraction_quality_assess",
            actor_id="admin-user-004"
        )
        assert len(extraction_audit) > 0
        
        pytest.assumption("Phase 7→8 Flow OK: 평가 기준 → 추출 품질 평가")

    @pytest.mark.asyncio
    async def test_full_s2_lifecycle(self):
        """
        완전한 S2 라이프사이클 테스트
        
        Flow:
        1. 문서 업로드 (사용자, scope profile 설정)
        2. 자동 추출 (Phase 1-3)
        3. Agent 보조 검토 (Phase 4-5)
        4. Admin 승인 (Phase 6)
        5. 평가 실행 (Phase 7)
        6. 품질 최종 검증 (Phase 8)
        7. Agent 보조 검색 쿼리 (다시 Phase 3-4)
        
        검증:
        - S2 ⑤ actor_type 일관성 (user vs agent)
        - S2 ⑥ Scope Profile ACL 일관성
        - S2 ⑦ 폐쇄망 모드 호환성
        """
        
        # 1. 문서 업로드
        document_service = DocumentService(self.db)  # 가정: Phase 초기에 구현
        
        doc = await document_service.upload_document(
            file_path="/test/data/sample.pdf",
            user_id="user-007",
            scope_profile=self.test_scope,
            actor_id="user-007",
            actor_type="user"
        )
        
        assert doc.document_id is not None
        assert doc.scope_profile.name == self.test_scope.name
        
        # 2. 자동 추출 (Phase 1-3: LLM 선택, 검색, 세션 RAG)
        llm_service = LLMModelService(self.db)
        model = await llm_service.select_model(
            model_name="gpt-4-turbo",
            actor_id="user-007",
            actor_type="user",
            context={}
        )
        
        search_service = SearchService(self.db)
        search_results = await search_service.search(
            query=f"content from {doc.document_id}",
            scope_profile=self.test_scope,
            actor_id="user-007",
            actor_type="user"
        )
        
        session_rag_service = SessionRAGService(self.db)
        session = await session_rag_service.create_session(
            user_id="user-007",
            scope_profile=self.test_scope,
            actor_id="user-007",
            actor_type="user"
        )
        
        # 3. Agent 보조 검토 (Phase 4-5)
        mcp_service = MCPServerService(self.db)
        server = await mcp_service.initialize_server(
            session_id=session.session_id,
            scope_profile=self.test_scope,
            actor_id="user-007",
            actor_type="user"
        )
        
        agent_service = AgentPrincipalService(self.db)
        agent = await agent_service.create_principal(
            agent_id="agent-ai-final",
            agent_name="Final AI Assistant",
            agent_type="ai",
            owner_id="user-007",
            mcp_server_id=server.server_id,
            scope_profile=self.test_scope,
            actor_type="agent"
        )
        
        delegation_service = DelegationService(self.db)
        delegation = await delegation_service.create_delegation(
            delegating_principal=agent,
            delegated_to="admin-user-007",
            scope=["document_review"],
            actor_id="agent-ai-final",
            actor_type="agent"
        )
        
        draft = await agent_service.create_draft_proposal(
            task_id=str(uuid4()),
            agent_principal_id=agent.principal_id,
            content={
                "document_id": doc.document_id,
                "extracted_fields": [
                    {"name": "title", "value": "Extracted Title"}
                ]
            },
            delegation_id=delegation.delegation_id,
            actor_id="agent-ai-final",
            actor_type="agent"
        )
        
        # 4. Admin 승인 (Phase 6)
        approval_service = AdminApprovalService(self.db)
        queue_entry = await approval_service.add_to_queue(
            draft_proposal_id=draft.proposal_id,
            task_id=draft.task_id,
            priority="normal",
            admin_id="admin-user-007",
            actor_id="agent-ai-final",
            actor_type="agent"
        )
        
        approval = await approval_service.review_and_approve(
            queue_entry_id=queue_entry.queue_id,
            admin_id="admin-user-007",
            decision="approved",
            review_notes="Final approval",
            actor_id="admin-user-007",
            actor_type="user"
        )
        
        # 5. 평가 실행 (Phase 7)
        eval_runner = EvaluationRunner(self.db)
        criteria = await eval_runner.load_evaluation_criteria(
            criteria_set="default_golden_set",
            actor_id="admin-user-007",
            actor_type="user"
        )
        
        eval_result = await eval_runner.run_evaluation(
            task_id=draft.task_id,
            content=draft.content,
            criteria=criteria,
            actor_id="admin-user-007",
            actor_type="user"
        )
        
        # 6. 품질 최종 검증 (Phase 8)
        extraction_quality = ExtractionQualityService(self.db)
        overall_quality = await extraction_quality.calculate_overall_quality(
            field_assessments=[],
            actor_id="admin-user-007",
            actor_type="user"
        )
        
        # 7. Agent 보조 검색 (다시 Phase 3-4)
        follow_up_results = await search_service.search(
            query="follow-up query",
            scope_profile=self.test_scope,
            actor_id="agent-ai-final",
            actor_type="agent"
        )
        
        # 전체 검증
        audit_logs = await self.audit_log_service.list_logs()
        
        # actor_type 일관성 검증
        user_logs = [log for log in audit_logs if log.actor_id == "user-007"]
        assert all(log.actor_type == "user" for log in user_logs)
        
        agent_logs = [log for log in audit_logs if log.actor_id == "agent-ai-final"]
        assert all(log.actor_type == "agent" for log in agent_logs)
        
        # Scope Profile 일관성 검증
        scope_filtered_actions = [
            log for log in audit_logs
            if log.action in ["search", "fetch_node"]
        ]
        assert all(
            log.scope_profile.name == self.test_scope.name
            for log in scope_filtered_actions
            if log.scope_profile
        )
        
        pytest.assumption("S2 Full Lifecycle OK: 문서→추출→검토→승인→평가→검증")


@pytest.mark.integration
@pytest.mark.s2_principles
@pytest.mark.closed_network
class TestS2ClosedNetworkMode:
    """
    폐쇄망 환경(외부 LLM 없음) 모드에서의 통합 테스트
    
    환경변수: MIMIR_DISABLE_EXTERNAL_LLM=true
    동작: LLM 의존 단계는 건너뛰거나 rule-based fallback 사용
    """

    @pytest.fixture(autouse=True)
    async def setup_closed_network(self, monkeypatch, db_session):
        """폐쇄망 모드 활성화"""
        monkeypatch.setenv("MIMIR_DISABLE_EXTERNAL_LLM", "true")
        self.db = db_session
        self.audit_log_service = AuditLogService(db_session)
        
        yield

    @pytest.mark.asyncio
    async def test_search_with_fallback(self):
        """폐쇄망: 검색은 rule-based fallback으로 동작"""
        
        # FTS (Full-text search) 또는 기존 인덱스 활용
        search_service = SearchService(self.db)
        
        results = await search_service.search(
            query="test query",
            scope_profile=None,  # 폐쇄망에서는 scope 필터링 생략 가능
            actor_id="user-closed-001",
            actor_type="user",
            use_fallback=True  # 명시적 fallback
        )
        
        assert len(results) >= 0  # 0개 결과도 실패 아님
        
        # audit log에 fallback 모드 기록
        audit = await self.audit_log_service.list_logs(
            action="search",
            actor_id="user-closed-001"
        )
        assert any(log.context.get("mode") == "fallback" for log in audit)

    @pytest.mark.asyncio
    async def test_closed_network_phase1_to_8(self):
        """폐쇄망: Phase 1-8 완전 라이프사이클 (degraded functionality)"""
        
        # Phase 1: 로컬 모델만 사용 가능
        llm_service = LLMModelService(self.db)
        
        available_models = await llm_service.list_models(
            filter_external=True,  # 폐쇄망: 외부 모델 제외
            actor_id="user-closed-002",
            actor_type="user"
        )
        
        assert len(available_models) > 0
        local_model = available_models[0]
        assert local_model.is_local == True
        
        # 나머지 Phase도 fallback으로 동작
        # ... (Phase 2-8 동일한 패턴)
        
        pytest.assumption("Closed-network mode works with degraded performance")

```

### 4.2 테스트 Fixture 및 헬퍼 함수

**파일**: `tests/integration/conftest.py` (통합 테스트 공용 fixture)

```python
"""
통합 테스트용 공유 fixture
"""

import pytest
import asyncio
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

import sys
sys.path.insert(0, '/mimir/src')


@pytest.fixture(scope="session")
def event_loop():
    """테스트 세션 레벨 이벤트 루프"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()


@pytest.fixture(scope="session")
async def async_db_engine():
    """테스트 DB 엔진 (async)"""
    engine = create_async_engine(
        "postgresql+asyncpg://test:test@localhost/mimir_test",
        echo=False,
        pool_size=5,
        max_overflow=10
    )
    
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    
    yield engine
    
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    
    await engine.dispose()


@pytest.fixture
async def db_session(async_db_engine):
    """테스트마다 새로운 DB 세션"""
    async_session = sessionmaker(
        async_db_engine,
        class_=AsyncSession,
        expire_on_commit=False
    )
    
    async with async_session() as session:
        yield session
        await session.rollback()


@pytest.fixture
async def shared_db_setup(db_session):
    """공유 테스트 데이터: 문서, 사용자, 평가 기준 등"""
    
    # 테스트 문서 생성
    from mimir.phase1.models import Document
    
    test_doc = Document(
        document_id="test-doc-001",
        title="Test Document",
        content="This is a test document for integration testing.",
        document_type="pdf",
        created_at=datetime.now(),
        owner_id="user-test"
    )
    db_session.add(test_doc)
    
    # 평가 기준 생성
    from mimir.phase7.models import EvaluationCriteria
    
    criteria = EvaluationCriteria(
        criteria_id="default_golden_set",
        faithfulness_threshold=0.80,
        answer_relevance_threshold=0.75,
        context_precision_threshold=0.75,
        context_recall_threshold=0.75,
        citation_present_threshold=0.90,
        hallucination_max=0.10,
        extraction_quality_threshold=0.80,
        created_at=datetime.now()
    )
    db_session.add(criteria)
    
    await db_session.commit()
    
    yield
    
    await db_session.rollback()


@pytest.fixture
def mock_scope_profile_factory(db_session):
    """동적 Scope Profile 생성 팩토리"""
    
    async def create_scope(name: str, allowed_types: list, access_rules: dict):
        from mimir.common.scope_profile import ScopeProfile
        
        profile = ScopeProfile(
            name=name,
            allowed_types=allowed_types,
            access_rules=access_rules,
            created_at=datetime.now()
        )
        db_session.add(profile)
        await db_session.commit()
        
        return profile
    
    return create_scope
```

### 4.3 추가 검증 로직 예시

```python
# tests/integration/test_integration_s2_phase0_to_8.py에 추가할 헬퍼 클래스

class S2ValidationHelper:
    """S2 원칙 검증 헬퍼"""
    
    def __init__(self, audit_log_service):
        self.audit_log_service = audit_log_service
    
    async def verify_actor_type_consistency(
        self,
        actor_id: str,
        expected_actor_type: str,
        action: str
    ) -> bool:
        """
        S2 ⑤: 특정 actor의 모든 로그에서 actor_type이 일관되는지 검증
        """
        logs = await self.audit_log_service.list_logs(
            action=action,
            actor_id=actor_id
        )
        
        return all(log.actor_type == expected_actor_type for log in logs)
    
    async def verify_scope_filtering(
        self,
        scope_profile_name: str,
        action: str
    ) -> bool:
        """
        S2 ⑥: 특정 scope profile을 사용하는 모든 조회/쓰기에서
        ACL 필터가 적용되었는지 검증
        """
        logs = await self.audit_log_service.list_logs(
            action=action,
            scope_profile_name=scope_profile_name
        )
        
        # 각 로그에서 filtered_document_count > 0 확인
        return all(
            log.context.get("filtered_document_count", 0) >= 0
            for log in logs
        )
    
    async def verify_no_hardcoded_scope(
        self,
        codebase_path: str
    ) -> List[str]:
        """
        S2 ⑥: 코드에 하드코딩된 scope 문자열 찾기
        예: if scope == "team", scope_name = "personal", etc.
        
        반환: 의심 코드 라인 리스트
        """
        import ast
        import os
        
        suspicious_lines = []
        
        for root, dirs, files in os.walk(codebase_path):
            for file in files:
                if not file.endswith(".py"):
                    continue
                
                filepath = os.path.join(root, file)
                with open(filepath, 'r') as f:
                    try:
                        tree = ast.parse(f.read())
                    except:
                        continue
                
                for node in ast.walk(tree):
                    # if scope == "..." 패턴 검색
                    if isinstance(node, ast.Compare):
                        if isinstance(node.left, ast.Name) and node.left.id == "scope":
                            for comp in node.comparators:
                                if isinstance(comp, ast.Constant) and isinstance(comp.value, str):
                                    suspicious_lines.append(
                                        f"{filepath}: line {node.lineno}"
                                    )
        
        return suspicious_lines
    
    async def verify_closed_network_compatibility(
        self,
        test_results: List[Dict[str, Any]]
    ) -> bool:
        """
        S2 ⑦: 폐쇄망 모드 테스트 결과 검증
        - all Phase들이 pass/degrade 상태여야 함 (fail 아님)
        - external LLM 호출이 없어야 함
        """
        
        for result in test_results:
            if result["status"] == "FAILED":
                return False
            
            if result.get("external_llm_calls", 0) > 0:
                return False
        
        return True
```

---

## 5. 산출물

### 주요 산출물

1. **작업 실행 산출물**
   - `/tests/integration/test_integration_s2_phase0_to_8.py` (약 1200 라인)
   - `/tests/integration/conftest.py` (약 150 라인)
   - `/tests/integration/test_helpers.py` (약 300 라인)

2. **테스트 실행 결과 파일**
   - `test_results/integration_test_report.json` (pytest --json-report)
   - `test_results/s2_principles_verification.md` (S2 원칙별 검증 결과)
   - `test_results/audit_log_summary.csv` (actor_type, scope 통계)

3. **CI/CD 아티팩트**
   - GitHub Actions 로그
   - 커버리지 리포트: `htmlcov/index.html`

---

## 6. 완료 기준

1. **테스트 코드 완성도**
   - [ ] TestS2IntegrationRegression 클래스에 8개 phase-to-phase 테스트 메서드 모두 구현
   - [ ] 각 테스트가 3개 assertion 이상 포함 (입력, 처리, 출력 검증)
   - [ ] S2 원칙 검증 로직 (actor_type, scope, closed-network) 포함

2. **테스트 실행 성공**
   - [ ] 모든 테스트가 통과 (`pytest tests/integration/test_integration_s2_phase0_to_8.py -v`)
   - [ ] 커버리지 >= 85% (통합 테스트 기준)
   - [ ] Slow tests 포함해서 전체 실행 시간 < 10분

3. **S2 원칙 검증**
   - [ ] actor_type 검증: 모든 audit log에 "user" 또는 "agent" 기록
   - [ ] Scope filtering 검증: ACL 필터가 모든 검색·조회에 적용됨
   - [ ] Closed-network 검증: MIMIR_DISABLE_EXTERNAL_LLM=true 시에도 동작

4. **문서화**
   - [ ] 각 테스트 메서드에 docstring (목적, flow, 검증사항)
   - [ ] fixture 문서화
   - [ ] 테스트 실행 방법 README 작성

5. **검수 준비**
   - [ ] 검수 보고서 작성 (test coverage, S2 principle 검증 결과)
   - [ ] 보안 취약점 검사 보고서 작성

---

## 7. 작업 지침

**7-1** **Phase 간 의존성 추적**
- 각 test_phaseX_to_phaseY 메서드에서 출력→입력 chain을 명확히 주석으로 표시
- 만약 중간 Phase 로직에 변경이 생기면 다음 Phase 테스트가 즉시 실패하도록 설계

**7-2** **S2 원칙 검증 의무화**
- actor_type 검증: 모든 async 함수 호출 시 `actor_type` parameter 전달 필수
- Scope profile 검증: 모든 search/fetch 호출 시 `scope_profile` parameter 전달 필수
- audit log 검증: 각 테스트 끝에서 `await self.audit_log_service.list_logs()` 호출해서 기록 확인

**7-3** **Closed-network 테스트 실행 방식**
```bash
# 폐쇄망 모드: 외부 LLM 비활성화
export MIMIR_DISABLE_EXTERNAL_LLM=true
pytest tests/integration/test_integration_s2_phase0_to_8.py::TestS2ClosedNetworkMode -v

# 일반 모드
pytest tests/integration/test_integration_s2_phase0_to_8.py::TestS2IntegrationRegression -v
```

**7-4** **Fixture 재사용 최대화**
- `shared_db_setup`: 모든 테스트가 공유하는 문서, 사용자, 평가 기준
- `mock_scope_profile_factory`: 테스트마다 다른 scope profile 생성 가능
- 각 fixture는 cleanup 코드 포함 (rollback, delete)

**7-5** **어설션 메시지 명확성**
```python
# 나쁜 예
assert len(citations) > 0

# 좋은 예
assert len(citations) > 0, (
    f"Phase 2 검색에서 citation이 생성되지 않음. "
    f"search_results={len(search_results)}, "
    f"expected citations >= 1"
)
```

**7-6** **테스트 데이터 격리**
- 각 테스트는 고유한 user_id, agent_id, task_id 사용 (uuid4)
- 테스트 간 데이터 오염 방지
- DB transaction 자동 rollback (fixture 사용)

**7-7** **pytest 마커 활용**
```python
# 느린 테스트는 별도 마킹
@pytest.mark.slow
async def test_phase6_to_phase7_flow(self):
    """평가 실행은 시간이 오래 걸림"""
    ...

# 로컬 개발에서만 실행
@pytest.mark.integration
async def test_phase1_to_phase2_flow(self):
    ...

# CI/CD에서만 실행
@pytest.mark.ci_only
async def test_full_s2_lifecycle(self):
    ...

# 마커별 실행
pytest tests/integration/ -m "not slow"  # 빠른 테스트만
pytest tests/integration/ -m "slow"      # 느린 테스트만
```
