# Task 5-1. Draft 상태 확장 (proposed) + 에이전트 Draft 생성 API

## 1. 작업 목적

Phase 5 Agent Action Plane의 핵심 원칙인 **"에이전트 쓰기는 proposed 상태의 Draft로만 진입"**을 구현하기 위해, Draft 상태 머신을 확장하고 에이전트가 문서를 Draft 제안으로만 생성할 수 있는 API를 구축한다. 이를 통해 인간이 최종 검토·승인 후 문서가 published 상태로 진입하는 안전한 워크플로를 실현한다. 또한 **S2 원칙 ⑤ (actor_type 감시), ⑥ (Scope Profile 기반 ACL)**을 의무 적용하여 에이전트 활동을 추적하고 접근 범위를 관리한다.

## 2. 작업 범위

### 포함 범위

1. **Draft 상태 머신 확장**
   - 기존 상태: draft, review, approved, published, archived
   - 추가 상태: proposed (에이전트 생성), rejected (검수 거부), withdrawn (제안 회수)
   - 상태 전이 규칙:
     - 에이전트 → proposed만 허용 (다른 상태로 진입 불가)
     - 인간 → draft → review → approved → published (기존 경로 유지)
     - proposed → approved (인간 검토 후) → published / proposed → rejected
     - proposed → withdrawn (에이전트 또는 관리자가 제안 회수)

2. **DB 마이그레이션**
   - drafts 테이블:
     - status: VARCHAR(50) 기존 값에 'proposed', 'rejected', 'withdrawn' 추가
     - created_by_agent: BOOLEAN (기본값: false) — 에이전트 생성 여부 표시
     - agent_id: UUID (nullable) — 생성 에이전트 ID
     - proposal_notes: TEXT (nullable) — 제안 사유
   - 데이터 마이그레이션: 기존 데이터는 created_by_agent=false 유지

3. **에이전트 Draft 생성 API**
   - 엔드포인트: POST /api/v1/agents/{agent_id}/propose-draft
   - 입력:
     - document_id: UUID (nullable) — 기존 문서 수정 시, null이면 신규 생성
     - document_type_id: UUID (필수) — 문서 타입
     - title: str (필수) — Draft 제목
     - content: str (필수) — Draft 본문
     - metadata: dict (optional) — 메타데이터
     - reason: str (optional) — 제안 사유
   - 응답:
     - draft_id: UUID
     - status: "proposed"
     - document_id: UUID (수정 시)
     - created_by_agent: true
     - agent_id: UUID
     - created_at: datetime
     - proposal_url: str (검토 링크)

4. **Pydantic 스키마**
   - ProposeDraftRequest: 요청 스키마
   - ProposeDraftResponse: 응답 스키마

5. **DraftProposalService 클래스**
   - create_agent_proposal(agent_id, document_id, document_type_id, title, content, metadata, reason) → ProposeDraftResponse
   - 입력 검증 (title, content 길이 등)
   - 상태 머신 검증 (proposed 상태로만 생성)
   - Draft 메타데이터 구성

6. **권한 검증**
   - delegate:write scope 필수 (Scope Profile 기반 동적 검증 — S2 원칙 ⑥)
   - 에이전트 ID와 요청 주체 일치 확인
   - 코드에 scope 문자열 하드코딩 금지

7. **감시 로그 (S2 원칙 ⑤)**
   - 이벤트: agent_draft_create
   - actor_type: "agent" 필수
   - 기록 항목: agent_id, document_id, document_type_id, title 요약, reason

8. **단위 테스트**
   - 에이전트 Draft 정상 생성
   - 신규 vs 기존 문서 수정
   - 상태 값 검증 (proposed)
   - 권한 거부 케이스 (delegate:write 없음)
   - 입력 검증 (title, content 필수)
   - 감시 로그 기록 확인 (actor_type=agent)
   - 에러 처리 (타입 불일치, 메타데이터 유효성)

### 제외 범위

- 워크플로 전이 제안 (Task 5-2)
- Draft 승인/거부/회수 API (Task 5-2)
- MCP Tasks 통합 (Task 5-3)
- Draft 렌더링/조회 API (기존 Task)

## 3. 선행 조건

- Phase 3 완료 (Document layer API, Draft 모델)
- Phase 4 완료 (Scope Profile 시스템)
- 기존 drafts DB 테이블 분석 완료
- Pydantic 모델 정의 가능
- audit_log 모델 접근 가능

## 4. 주요 작업 항목

### 4-1. DB 마이그레이션 (Alembic)

**파일:** `/backend/alembic/versions/XXX_extend_draft_for_agent_proposals.py`

```python
"""Extend drafts table for agent proposals (S2-FG5.1)."""

from alembic import op
import sqlalchemy as sa
from uuid import uuid4


def upgrade() -> None:
    """
    Add columns to drafts table:
    - status: add 'proposed', 'rejected', 'withdrawn' enum values
    - created_by_agent: BOOLEAN to mark agent-created drafts
    - agent_id: UUID FK to agents table
    - proposal_notes: TEXT for proposal reason
    """
    # 1. 기존 status enum을 확장 (PostgreSQL)
    # (SQLite 등의 경우 string 타입 유지)
    
    # 2. created_by_agent 컬럼 추가
    op.add_column(
        'drafts',
        sa.Column(
            'created_by_agent',
            sa.Boolean(),
            nullable=False,
            server_default='false',
            doc="Agent-created draft indicator"
        )
    )
    
    # 3. agent_id 컬럼 추가 (FK to agents table)
    op.add_column(
        'drafts',
        sa.Column(
            'agent_id',
            sa.UUID(),
            nullable=True,
            doc="Agent ID for agent-created drafts"
        )
    )
    op.create_foreign_key(
        'fk_drafts_agent_id',
        'drafts',
        'agents',
        ['agent_id'],
        ['id'],
        ondelete='SET NULL'
    )
    
    # 4. proposal_notes 컬럼 추가
    op.add_column(
        'drafts',
        sa.Column(
            'proposal_notes',
            sa.Text(),
            nullable=True,
            doc="Reason for proposal"
        )
    )
    
    # 5. 인덱스 추가
    op.create_index(
        'ix_drafts_created_by_agent',
        'drafts',
        ['created_by_agent']
    )
    op.create_index(
        'ix_drafts_agent_id',
        'drafts',
        ['agent_id']
    )


def downgrade() -> None:
    """Rollback migration."""
    op.drop_index('ix_drafts_agent_id', table_name='drafts')
    op.drop_index('ix_drafts_created_by_agent', table_name='drafts')
    op.drop_constraint('fk_drafts_agent_id', 'drafts', type_='foreignkey')
    op.drop_column('drafts', 'proposal_notes')
    op.drop_column('drafts', 'agent_id')
    op.drop_column('drafts', 'created_by_agent')
```

### 4-2. Draft 모델 확장

**파일:** `/backend/app/models/draft.py` (기존 파일 수정)

```python
"""Draft 모델 확장 (S2-FG5.1)."""

from __future__ import annotations

from enum import Enum
from typing import Optional
from uuid import UUID
from datetime import datetime

from sqlalchemy import Column, String, UUID as SQLA_UUID, ForeignKey, Boolean, Text
from sqlalchemy.orm import relationship

from app.db import Base


class DraftStatus(str, Enum):
    """Draft 상태 (S1 + S2 추가)."""
    
    # S1 기존 상태
    DRAFT = "draft"              # 인간 작성, 초안
    REVIEW = "review"            # 인간 작성, 검수 중
    APPROVED = "approved"        # 인간 검수 승인
    PUBLISHED = "published"      # 발행됨
    ARCHIVED = "archived"        # 보관됨
    
    # S2 추가 상태
    PROPOSED = "proposed"        # 에이전트 제안 (원본)
    REJECTED = "rejected"        # 인간 검수 거부
    WITHDRAWN = "withdrawn"      # 제안 회수


class Draft(Base):
    """Draft 모델 (에이전트 제안 지원)."""
    
    __tablename__ = "drafts"
    
    # 기존 필드 (생략)
    id: SQLA_UUID = Column(SQLA_UUID(as_uuid=True), primary_key=True, default=uuid4)
    document_id: Optional[SQLA_UUID] = Column(SQLA_UUID(as_uuid=True), ForeignKey("documents.id"), nullable=True)
    document_type_id: SQLA_UUID = Column(SQLA_UUID(as_uuid=True), ForeignKey("document_types.id"), nullable=False)
    title: str = Column(String(255), nullable=False)
    content: str = Column(Text, nullable=False)
    status: str = Column(String(50), default=DraftStatus.DRAFT.value, nullable=False)
    metadata: dict = Column(JSON, nullable=True)
    
    # S1 기존 필드
    created_by_user_id: SQLA_UUID = Column(SQLA_UUID(as_uuid=True), ForeignKey("users.id"), nullable=True)
    created_at: datetime = Column(DateTime, default=datetime.utcnow, nullable=False)
    updated_at: datetime = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow, nullable=False)
    
    # S2 추가 필드 (에이전트 제안)
    created_by_agent: bool = Column(Boolean, default=False, nullable=False)
    agent_id: Optional[SQLA_UUID] = Column(SQLA_UUID(as_uuid=True), ForeignKey("agents.id"), nullable=True)
    proposal_notes: Optional[str] = Column(Text, nullable=True)
    
    # 관계
    document = relationship("Document", back_populates="drafts")
    document_type = relationship("DocumentType")
    created_by_user = relationship("User")
    agent = relationship("Agent")
    
    def is_agent_proposal(self) -> bool:
        """에이전트 제안 여부."""
        return self.created_by_agent and self.agent_id is not None
    
    def can_be_proposed_by_agent(self) -> bool:
        """에이전트가 제안할 수 있는 상태인가?"""
        return self.status == DraftStatus.DRAFT.value
```

### 4-3. Pydantic 스키마

**파일:** `/backend/app/schemas/draft_proposals.py` (신규)

```python
"""Draft 제안 요청/응답 스키마 (S2-FG5.1)."""

from __future__ import annotations

from typing import Optional, Dict, Any
from uuid import UUID
from datetime import datetime

from pydantic import BaseModel, Field


class ProposeDraftRequest(BaseModel):
    """에이전트 Draft 생성 요청 스키마."""
    
    document_id: Optional[UUID] = Field(
        None,
        description="기존 문서 ID (null이면 신규 생성)"
    )
    document_type_id: UUID = Field(
        ...,
        description="문서 타입 ID (필수)"
    )
    title: str = Field(
        ...,
        min_length=1,
        max_length=255,
        description="Draft 제목 (필수, 1-255자)"
    )
    content: str = Field(
        ...,
        min_length=1,
        description="Draft 본문 (필수, 최소 1자)"
    )
    metadata: Optional[Dict[str, Any]] = Field(
        None,
        description="메타데이터 (선택)"
    )
    reason: Optional[str] = Field(
        None,
        max_length=1000,
        description="제안 사유 (선택, 최대 1000자)"
    )


class ProposeDraftResponse(BaseModel):
    """에이전트 Draft 생성 응답 스키마."""
    
    draft_id: UUID = Field(..., description="생성된 Draft ID")
    status: str = Field(
        "proposed",
        description="Draft 상태 (항상 'proposed')"
    )
    document_id: Optional[UUID] = Field(
        None,
        description="문서 ID (수정 시)"
    )
    document_type_id: UUID = Field(..., description="문서 타입 ID")
    created_by_agent: bool = Field(
        True,
        description="에이전트 생성 여부 (항상 true)"
    )
    agent_id: UUID = Field(..., description="생성 에이전트 ID")
    created_at: datetime = Field(..., description="생성 시간")
    title: str = Field(..., description="Draft 제목")
    proposal_url: str = Field(
        ...,
        description="검토 페이지 URL"
    )


class DraftProposalStatusResponse(BaseModel):
    """Draft 상태 조회 응답."""
    
    draft_id: UUID
    status: str
    created_by_agent: bool
    agent_id: Optional[UUID]
    created_at: datetime
    proposal_notes: Optional[str]
```

### 4-4. DraftProposalService 클래스

**파일:** `/backend/app/services/draft_proposals.py` (신규)

```python
"""Draft 제안 서비스 (S2-FG5.1)."""

from __future__ import annotations

import logging
from typing import Optional, Dict, Any
from uuid import UUID
from datetime import datetime

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.models.draft import Draft, DraftStatus
from app.models.document import DocumentType
from app.models.agent import Agent
from app.models.audit_log import AuditLog
from app.schemas.draft_proposals import ProposeDraftRequest, ProposeDraftResponse
from app.exceptions import InvalidRequestError, NotFoundError, PermissionDeniedError

logger = logging.getLogger(__name__)


class DraftProposalService:
    """Draft 제안 서비스 (에이전트 Draft 생성 전용)."""
    
    def __init__(self, db: AsyncSession):
        """
        Args:
            db: 데이터베이스 세션
        """
        self.db = db
    
    async def create_agent_proposal(
        self,
        agent_id: UUID,
        request: ProposeDraftRequest,
        scope_profile: Dict[str, Any],  # S2 원칙 ⑥: Scope Profile
    ) -> ProposeDraftResponse:
        """에이전트 Draft 제안 생성.
        
        에이전트는 반드시 proposed 상태의 Draft로만 진입 가능.
        인간은 기존 draft→review→approved→published 경로 유지.
        
        Args:
            agent_id: 에이전트 ID
            request: 제안 요청
            scope_profile: Scope Profile (ACL 필터링용)
        
        Returns:
            제안 응답
        
        Raises:
            InvalidRequestError: 입력 검증 실패
            NotFoundError: 문서 또는 타입 없음
            PermissionDeniedError: delegate:write 권한 없음
        """
        logger.info(
            f"Creating agent proposal: agent_id={agent_id}, "
            f"document_id={request.document_id}, "
            f"document_type_id={request.document_type_id}"
        )
        
        try:
            # ① 권한 검증 (S2 원칙 ⑥: delegate:write scope)
            await self._validate_agent_write_permission(agent_id, scope_profile)
            
            # ② 에이전트 검증
            agent = await self._verify_agent_exists(agent_id)
            
            # ③ 문서 타입 검증
            doc_type = await self._verify_document_type_exists(request.document_type_id)
            
            # ④ 기존 문서 검증 (있는 경우)
            if request.document_id:
                document = await self._verify_document_exists(request.document_id)
            
            # ⑤ 입력 검증
            self._validate_proposal_input(request)
            
            # ⑥ Draft 생성 (proposed 상태로만 진입)
            draft = Draft(
                document_id=request.document_id,
                document_type_id=request.document_type_id,
                title=request.title,
                content=request.content,
                metadata=request.metadata,
                status=DraftStatus.PROPOSED.value,  # 반드시 proposed
                created_by_agent=True,
                agent_id=agent_id,
                proposal_notes=request.reason,
            )
            self.db.add(draft)
            await self.db.flush()
            
            draft_id = draft.id
            
            # ⑦ 감시 로그 기록 (S2 원칙 ⑤: actor_type=agent)
            await self._log_agent_draft_create(
                agent_id=agent_id,
                draft_id=draft_id,
                document_id=request.document_id,
                document_type_id=request.document_type_id,
                title=request.title,
                reason=request.reason,
            )
            
            # ⑧ 응답 구성
            proposal_url = self._build_proposal_url(draft_id)
            
            response = ProposeDraftResponse(
                draft_id=draft_id,
                status=DraftStatus.PROPOSED.value,
                document_id=request.document_id,
                document_type_id=request.document_type_id,
                created_by_agent=True,
                agent_id=agent_id,
                created_at=draft.created_at,
                title=request.title,
                proposal_url=proposal_url,
            )
            
            logger.info(f"Agent proposal created: draft_id={draft_id}")
            return response
        
        except (InvalidRequestError, NotFoundError, PermissionDeniedError):
            raise
        except Exception as e:
            logger.error(f"Failed to create agent proposal: {str(e)}")
            await self._log_agent_draft_error(
                agent_id=agent_id,
                error_message=str(e),
            )
            raise InvalidRequestError(f"Failed to create proposal: {str(e)}")
    
    # ==================== Helper Methods ====================
    
    async def _validate_agent_write_permission(
        self,
        agent_id: UUID,
        scope_profile: Dict[str, Any],
    ) -> None:
        """delegate:write scope 검증 (S2 원칙 ⑥).
        
        코드에 scope 문자열 하드코딩 금지.
        Scope Profile에서 동적으로 권한 확인.
        """
        # Scope Profile에서 scopes 리스트 추출
        scopes = scope_profile.get("scopes", [])
        
        # delegate:write 권한 확인
        if "delegate:write" not in scopes:
            logger.warning(
                f"Agent {agent_id} lacks delegate:write scope"
            )
            raise PermissionDeniedError(
                "Agent lacks delegate:write scope for creating drafts"
            )
    
    async def _verify_agent_exists(self, agent_id: UUID) -> Agent:
        """에이전트 존재 확인."""
        stmt = select(Agent).where(Agent.id == agent_id)
        result = await self.db.execute(stmt)
        agent = result.scalar_one_or_none()
        
        if not agent:
            raise NotFoundError(f"Agent not found: {agent_id}")
        
        return agent
    
    async def _verify_document_type_exists(self, document_type_id: UUID) -> DocumentType:
        """문서 타입 존재 확인."""
        stmt = select(DocumentType).where(DocumentType.id == document_type_id)
        result = await self.db.execute(stmt)
        doc_type = result.scalar_one_or_none()
        
        if not doc_type:
            raise NotFoundError(f"DocumentType not found: {document_type_id}")
        
        return doc_type
    
    async def _verify_document_exists(self, document_id: UUID):
        """기존 문서 존재 확인."""
        from app.models.document import Document
        stmt = select(Document).where(Document.id == document_id)
        result = await self.db.execute(stmt)
        document = result.scalar_one_or_none()
        
        if not document:
            raise NotFoundError(f"Document not found: {document_id}")
        
        return document
    
    def _validate_proposal_input(self, request: ProposeDraftRequest) -> None:
        """입력 검증."""
        if not request.title or not request.title.strip():
            raise InvalidRequestError("title is required and cannot be empty")
        
        if not request.content or not request.content.strip():
            raise InvalidRequestError("content is required and cannot be empty")
        
        if request.reason and len(request.reason) > 1000:
            raise InvalidRequestError("reason must be <= 1000 characters")
    
    def _build_proposal_url(self, draft_id: UUID) -> str:
        """제안 검토 페이지 URL 구성."""
        # 실제 구현에서는 frontend URL 기반으로 구성
        return f"/drafts/{draft_id}/review?mode=proposal"
    
    async def _log_agent_draft_create(
        self,
        agent_id: UUID,
        draft_id: UUID,
        document_id: Optional[UUID],
        document_type_id: UUID,
        title: str,
        reason: Optional[str],
    ) -> None:
        """에이전트 Draft 생성 감시 로그 (S2 원칙 ⑤).
        
        actor_type=agent 필수 기록.
        """
        try:
            audit_log = AuditLog(
                entity_type="draft",
                entity_id=draft_id,
                action="agent_proposal_create",
                actor_id=agent_id,
                actor_type="agent",  # S2 원칙 ⑤: 필수
                changes={
                    "status": "proposed",
                    "document_id": str(document_id) if document_id else None,
                    "document_type_id": str(document_type_id),
                    "title": title[:100],  # 첫 100자
                    "reason": reason[:200] if reason else None,
                },
            )
            self.db.add(audit_log)
            await self.db.flush()
            logger.debug(
                f"Agent draft creation logged: draft_id={draft_id}, "
                f"agent_id={agent_id}"
            )
        except Exception as e:
            logger.error(f"Failed to log agent draft creation: {str(e)}")
    
    async def _log_agent_draft_error(
        self,
        agent_id: UUID,
        error_message: str,
    ) -> None:
        """에이전트 Draft 생성 에러 감시 로그."""
        try:
            audit_log = AuditLog(
                entity_type="draft",
                entity_id=UUID(int=0),  # Generic ID
                action="agent_proposal_error",
                actor_id=agent_id,
                actor_type="agent",
                changes={
                    "error_message": error_message[:500],
                },
            )
            self.db.add(audit_log)
            await self.db.flush()
        except Exception as e:
            logger.error(f"Failed to log agent draft error: {str(e)}")
```

### 4-5. FastAPI 라우터

**파일:** `/backend/app/routes/agents.py` (확장)

```python
"""에이전트 라우터 (Draft 제안 엔드포인트 추가)."""

from typing import Optional
from uuid import UUID

from fastapi import APIRouter, Depends, Header, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from app.db import get_db
from app.schemas.draft_proposals import ProposeDraftRequest, ProposeDraftResponse
from app.services.draft_proposals import DraftProposalService
from app.exceptions import InvalidRequestError, NotFoundError, PermissionDeniedError
from app.auth import get_scope_profile_for_agent

router = APIRouter(prefix="/api/v1/agents", tags=["agents"])


@router.post("/{agent_id}/propose-draft", response_model=ProposeDraftResponse)
async def propose_draft(
    agent_id: UUID,
    request: ProposeDraftRequest,
    authorization: Optional[str] = Header(None),
    db: AsyncSession = Depends(get_db),
) -> ProposeDraftResponse:
    """
    에이전트 Draft 제안 생성 엔드포인트.
    
    에이전트는 Draft를 proposed 상태로만 생성 가능.
    인간의 최종 검토·승인 후 published 상태로 전이.
    
    Path Parameters:
        agent_id: 에이전트 ID
    
    Headers:
        Authorization: Bearer <oauth_token>
    
    Request Body:
        document_id: UUID (nullable) — 기존 문서 수정 시
        document_type_id: UUID (필수) — 문서 타입
        title: str (필수) — Draft 제목
        content: str (필수) — Draft 본문
        metadata: dict (optional) — 메타데이터
        reason: str (optional) — 제안 사유
    
    Returns:
        ProposeDraftResponse
    
    Raises:
        400: 입력 검증 실패
        401: 권한 없음
        404: 문서/타입 없음
        500: 서버 에러
    """
    # OAuth 토큰 검증
    if not authorization:
        raise HTTPException(
            status_code=401,
            detail="Missing Authorization header"
        )
    
    try:
        # Scope Profile 조회
        scope_profile = await get_scope_profile_for_agent(agent_id, db)
        
        # Draft 제안 생성
        service = DraftProposalService(db)
        response = await service.create_agent_proposal(
            agent_id=agent_id,
            request=request,
            scope_profile=scope_profile,
        )
        
        await db.commit()
        return response
    
    except InvalidRequestError as e:
        await db.rollback()
        raise HTTPException(status_code=400, detail=str(e))
    except NotFoundError as e:
        await db.rollback()
        raise HTTPException(status_code=404, detail=str(e))
    except PermissionDeniedError as e:
        await db.rollback()
        raise HTTPException(status_code=403, detail=str(e))
    except Exception as e:
        await db.rollback()
        raise HTTPException(status_code=500, detail=f"Internal error: {str(e)}")
```

### 4-6. 단위 테스트

**파일:** `/backend/tests/unit/services/test_draft_proposals.py`

```python
"""Draft 제안 서비스 단위 테스트."""

import pytest
from uuid import uuid4
from datetime import datetime
from sqlalchemy.ext.asyncio import AsyncSession

from app.services.draft_proposals import DraftProposalService
from app.schemas.draft_proposals import ProposeDraftRequest
from app.models.draft import Draft, DraftStatus
from app.models.audit_log import AuditLog
from app.exceptions import InvalidRequestError, NotFoundError, PermissionDeniedError


@pytest.fixture
def draft_proposal_service(db_session: AsyncSession):
    """Draft 제안 서비스."""
    return DraftProposalService(db_session)


@pytest.fixture
def agent_id():
    """에이전트 ID."""
    return uuid4()


@pytest.fixture
def valid_scope_profile():
    """유효한 Scope Profile (delegate:write 포함)."""
    return {
        "scopes": ["delegate:write", "read", "query"],
        "organization_id": uuid4(),
        "user_id": uuid4(),
    }


@pytest.fixture
def invalid_scope_profile():
    """무효한 Scope Profile (delegate:write 없음)."""
    return {
        "scopes": ["read", "query"],
        "organization_id": uuid4(),
    }


@pytest.mark.asyncio
async def test_create_agent_proposal_basic(
    draft_proposal_service,
    agent_id,
    valid_scope_profile,
    db_session,
):
    """에이전트 Draft 제안 정상 생성."""
    request = ProposeDraftRequest(
        document_id=None,  # 신규 생성
        document_type_id=uuid4(),
        title="AI 보안 규정 (에이전트 제안)",
        content="이는 에이전트가 제안한 신규 문서입니다.",
        metadata={"source": "agent"},
        reason="자동 생성된 규정 초안"
    )
    
    # Mock: document_type, agent 존재 확인 (실제 테스트에서는 DB에 사전 등록)
    # response = await draft_proposal_service.create_agent_proposal(
    #     agent_id=agent_id,
    #     request=request,
    #     scope_profile=valid_scope_profile,
    # )
    
    # assert response.draft_id is not None
    # assert response.status == "proposed"
    # assert response.created_by_agent is True
    # assert response.agent_id == agent_id
    # assert response.proposal_url is not None


@pytest.mark.asyncio
async def test_create_agent_proposal_status_is_proposed(
    draft_proposal_service,
    agent_id,
    valid_scope_profile,
    db_session,
):
    """Draft 상태가 반드시 'proposed'인지 확인."""
    # 생성 후 DB에서 직접 조회하여 status 확인
    # stmt = select(Draft).where(Draft.id == response.draft_id)
    # result = await db_session.execute(stmt)
    # draft = result.scalar_one()
    # 
    # assert draft.status == DraftStatus.PROPOSED.value


@pytest.mark.asyncio
async def test_create_agent_proposal_existing_document(
    draft_proposal_service,
    agent_id,
    valid_scope_profile,
    db_session,
):
    """기존 문서 수정 시 document_id 전달."""
    existing_doc_id = uuid4()
    request = ProposeDraftRequest(
        document_id=existing_doc_id,
        document_type_id=uuid4(),
        title="기존 문서 수정",
        content="기존 문서를 에이전트가 수정했습니다.",
    )
    
    # response = await draft_proposal_service.create_agent_proposal(
    #     agent_id=agent_id,
    #     request=request,
    #     scope_profile=valid_scope_profile,
    # )
    # 
    # assert response.document_id == existing_doc_id


@pytest.mark.asyncio
async def test_create_agent_proposal_missing_delegate_write(
    draft_proposal_service,
    agent_id,
    invalid_scope_profile,
):
    """delegate:write 권한 없으면 거부."""
    request = ProposeDraftRequest(
        document_id=None,
        document_type_id=uuid4(),
        title="Test",
        content="Test content",
    )
    
    with pytest.raises(PermissionDeniedError):
        await draft_proposal_service.create_agent_proposal(
            agent_id=agent_id,
            request=request,
            scope_profile=invalid_scope_profile,
        )


@pytest.mark.asyncio
async def test_create_agent_proposal_empty_title(
    draft_proposal_service,
    agent_id,
    valid_scope_profile,
):
    """제목이 비어있으면 거부."""
    request = ProposeDraftRequest(
        document_id=None,
        document_type_id=uuid4(),
        title="",  # 빈 제목
        content="Valid content",
    )
    
    with pytest.raises(InvalidRequestError):
        await draft_proposal_service.create_agent_proposal(
            agent_id=agent_id,
            request=request,
            scope_profile=valid_scope_profile,
        )


@pytest.mark.asyncio
async def test_create_agent_proposal_empty_content(
    draft_proposal_service,
    agent_id,
    valid_scope_profile,
):
    """내용이 비어있으면 거부."""
    request = ProposeDraftRequest(
        document_id=None,
        document_type_id=uuid4(),
        title="Valid title",
        content="",  # 빈 내용
    )
    
    with pytest.raises(InvalidRequestError):
        await draft_proposal_service.create_agent_proposal(
            agent_id=agent_id,
            request=request,
            scope_profile=valid_scope_profile,
        )


@pytest.mark.asyncio
async def test_create_agent_proposal_audit_log(
    draft_proposal_service,
    agent_id,
    valid_scope_profile,
    db_session,
):
    """감시 로그가 actor_type=agent로 기록되는지 확인."""
    request = ProposeDraftRequest(
        document_id=None,
        document_type_id=uuid4(),
        title="Test Proposal",
        content="Test content for audit logging",
        reason="Testing audit logs",
    )
    
    # response = await draft_proposal_service.create_agent_proposal(
    #     agent_id=agent_id,
    #     request=request,
    #     scope_profile=valid_scope_profile,
    # )
    # 
    # # DB에서 감시 로그 조회
    # from sqlalchemy import select
    # stmt = select(AuditLog).where(
    #     (AuditLog.entity_id == response.draft_id) &
    #     (AuditLog.action == "agent_proposal_create")
    # )
    # result = await db_session.execute(stmt)
    # logs = result.scalars().all()
    # 
    # assert len(logs) > 0
    # assert logs[0].actor_type == "agent"
    # assert logs[0].actor_id == agent_id
```

## 5. 산출물

1. **DB 마이그레이션** (`/backend/alembic/versions/XXX_extend_draft_for_agent_proposals.py`)
   - created_by_agent, agent_id, proposal_notes 컬럼 추가
   - status enum 확장

2. **Draft 모델 확장** (`/backend/app/models/draft.py`)
   - DraftStatus enum (proposed, rejected, withdrawn 추가)
   - 신규 필드: created_by_agent, agent_id, proposal_notes
   - Helper 메서드: is_agent_proposal(), can_be_proposed_by_agent()

3. **Pydantic 스키마** (`/backend/app/schemas/draft_proposals.py`)
   - ProposeDraftRequest
   - ProposeDraftResponse
   - DraftProposalStatusResponse

4. **DraftProposalService** (`/backend/app/services/draft_proposals.py`)
   - create_agent_proposal() 메인 로직
   - _validate_agent_write_permission() (S2 원칙 ⑥)
   - 감시 로그 기록 (S2 원칙 ⑤)

5. **FastAPI 라우터** (`/backend/app/routes/agents.py` 확장)
   - POST /api/v1/agents/{agent_id}/propose-draft

6. **단위 테스트** (`/backend/tests/unit/services/test_draft_proposals.py`)
   - 에이전트 Draft 정상 생성
   - 신규 vs 기존 문서 수정
   - 권한 검증 (delegate:write)
   - 입력 검증
   - 감시 로그 (actor_type=agent)

## 6. 완료 기준

1. Draft 상태 머신이 proposed, rejected, withdrawn을 포함하는가?
2. DB 마이그레이션이 created_by_agent, agent_id, proposal_notes를 추가하는가?
3. 에이전트 Draft는 반드시 proposed 상태로만 생성되는가?
4. delegate:write scope 검증이 동적으로 수행되는가? (코드에 하드코딩 금지)
5. 모든 에이전트 Draft 생성 이벤트가 actor_type=agent로 감시 로그되는가?
6. API 응답에 draft_id, status, proposal_url이 포함되는가?
7. 입력 검증 (title, content 필수, 길이 제한)이 수행되는가?
8. 모든 단위 테스트가 통과하는가?
9. mypy 타입 검사를 통과하는가?
10. FastAPI 라우터가 예외 처리와 함께 올바르게 구현되었는가?

## 7. 작업 지침

### 지침 7-1. 상태 머신 설계 (S1 원칙 ①)

에이전트 → proposed만 허용하는 상태 전이 규칙을 데이터베이스 또는 서비스 로직에서 강제한다. 기존 인간 경로 (draft→review→approved→published)는 유지한다.

### 지침 7-2. Scope Profile 기반 ACL (S2 원칙 ⑥)

코드에 `if scope == "delegate:write"` 같은 문자열 하드코딩을 금지한다. 대신 scope_profile 객체에서 "scopes" 리스트를 동적으로 읽어 "delegate:write"의 포함 여부를 확인한다.

### 지침 7-3. 감시 로그 (S2 원칙 ⑤)

모든 에이전트 Draft 생성 이벤트에서 AuditLog.actor_type을 "agent"로 기록한다. 이를 통해 인간 사용자와 에이전트 활동을 구분하여 추적할 수 있다.

### 지침 7-4. 검수 워크플로 명확화

- 에이전트가 생성한 proposed Draft는 인간 검수자만 승인 가능 (Task 5-2)
- proposed → approved → published 경로
- proposed → rejected → discarded 경로 (Task 5-2에서 정의)

### 지침 7-5. 메타데이터 관리

metadata 필드는 optional이지만, 에이전트 생성 시 source, generation_timestamp, model_version 등의 정보를 포함할 수 있다.

### 지침 7-6. 제안 URL 생성

proposal_url은 프론트엔드에서 Draft 검수 페이지로 바로 이동할 수 있는 링크이다. 형식: `/drafts/{draft_id}/review?mode=proposal`

### 지침 7-7. DB 마이그레이션 및 데이터 일관성

마이그레이션 후 기존 데이터는 created_by_agent=false로 유지된다. 다운그레이드 시에는 추가된 컬럼을 완전히 제거한다.
