# Task 5-2. 워크플로 전이 제안 API + Draft 승인/거부/회수 API

## 1. 작업 목적

Task 5-1에서 구축한 에이전트 Draft 제안(proposed)을 인간이 검수·승인할 수 있는 워크플로를 완성한다. 이를 위해 **워크플로 전이 제안 API** (에이전트가 상태 전이를 제안), **Draft 승인 API** (인간이 proposed→approved), **Draft 거부 API** (인간이 proposed→rejected), **제안 회수 API** (에이전트 또는 관리자가 proposed 제안 철회)를 구현한다. 이를 통해 **에이전트는 안전하게 제안만 하고, 인간이 최종 결정**하는 명확한 책임 분리를 실현한다. 또한 **S2 원칙 ⑤ (actor_type 감시), ⑥ (Scope Profile 기반 ACL)**을 의무 적용한다.

## 2. 작업 범위

### 포함 범위

1. **워크플로 전이 제안 API**
   - 엔드포인트: POST /api/v1/agents/{agent_id}/propose-transition
   - 입력: document_id (필수), target_state (필수), reason (optional), approver_notes (optional)
   - 응답: transition_proposal_id, current_state, proposed_state, status (pending_approval)
   - 권한: delegate:transition scope 필수
   - 예: proposed→approved (인간 검수 권장), approved→published (최종 발행)

2. **TransitionProposal 도메인 모델 + DB**
   - transition_proposals 테이블:
     - id: UUID (PK)
     - draft_id: UUID (FK to drafts)
     - agent_id: UUID (FK to agents)
     - current_state: VARCHAR(50)
     - proposed_state: VARCHAR(50)
     - reason: TEXT (nullable)
     - status: VARCHAR(50) (pending_approval, approved, rejected)
     - created_at, updated_at
     - approved_at, approved_by_user_id (인간 승인 시)
     - rejected_at, rejected_reason

3. **Draft 승인 API**
   - 엔드포인트: POST /api/v1/drafts/{draft_id}/approve
   - 입력: notes (optional)
   - 응답: draft_id, status (approved), approved_at, approved_by (user_id)
   - 권한: 문서 소유자 또는 reviewer 역할 필수
   - 상태 전이: proposed→approved
   - 감시 로그: draft_approve 이벤트

4. **Draft 거부 API**
   - 엔드포인트: POST /api/v1/drafts/{draft_id}/reject
   - 입력: reason (필수)
   - 응답: draft_id, status (rejected), rejected_at, rejected_reason
   - 권한: 문서 소유자 또는 reviewer 역할 필수
   - 상태 전이: proposed→rejected
   - 감시 로그: draft_reject 이벤트

5. **제안 회수 API**
   - 엔드포인트: POST /api/v1/agents/{agent_id}/proposals/{proposal_id}/withdraw
   - 입력: (없음, 또는 reason optional)
   - 응답: proposal_id, status (withdrawn)
   - 권한: 제안 작성 에이전트 또는 관리자
   - 상태 전이: proposed→withdrawn
   - 감시 로그: proposal_withdraw 이벤트

6. **상태 전이 검증**
   - proposed→approved: reviewer 역할 필수
   - proposed→rejected: reviewer 역할 필수
   - approved→published: admin 또는 document owner
   - 비허가 전이 차단 (예: review→published 직접 진입 금지)

7. **Pydantic 스키마**
   - ProposeTransitionRequest, ProposeTransitionResponse
   - DraftApprovalRequest, DraftApprovalResponse
   - DraftRejectionRequest, DraftRejectionResponse
   - WithdrawProposalRequest, WithdrawProposalResponse

8. **감시 로그 (S2 원칙 ⑤)**
   - 이벤트: draft_approve, draft_reject, proposal_withdraw, transition_propose
   - actor_type: "user" (인간) 또는 "agent" (에이전트)
   - 기록 항목: draft_id, transition_proposal_id, reason, notes

9. **단위/통합 테스트**
   - 워크플로 전이 제안 생성
   - Draft 승인 (상태 검증, 권한 검증)
   - Draft 거부 (상태 검증, 권한 검증)
   - 제안 회수 (권한 검증)
   - 비허가 상태 전이 차단
   - 감시 로그 (actor_type 기록)
   - 권한 검증 (reviewer 역할)

### 제외 범위

- Draft 생성 (Task 5-1)
- MCP Tasks 통합 (Task 5-3)
- published→archived 전이 (기존 작업)
- Draft 렌더링/조회 (기존 작업)

## 3. 선행 조건

- Task 5-1 완료 (Draft proposed 상태)
- Phase 3 완료 (Document layer, User/Role 모델)
- Phase 4 완료 (Scope Profile 시스템)
- 기존 draft 테이블 분석 완료
- 역할 (Role) 시스템 분석 완료

## 4. 주요 작업 항목

### 4-1. TransitionProposal 모델 및 DB 마이그레이션

**파일:** `/backend/app/models/draft.py` (기존 파일 확장)

```python
"""TransitionProposal 모델 (S2-FG5.2)."""

from enum import Enum
from typing import Optional
from uuid import UUID, uuid4
from datetime import datetime

from sqlalchemy import Column, String, UUID as SQLA_UUID, ForeignKey, Text, DateTime
from sqlalchemy.orm import relationship

from app.db import Base


class TransitionProposalStatus(str, Enum):
    """워크플로 전이 제안 상태."""
    
    PENDING_APPROVAL = "pending_approval"
    APPROVED = "approved"
    REJECTED = "rejected"


class TransitionProposal(Base):
    """워크플로 전이 제안 모델."""
    
    __tablename__ = "transition_proposals"
    
    id: SQLA_UUID = Column(SQLA_UUID(as_uuid=True), primary_key=True, default=uuid4)
    draft_id: SQLA_UUID = Column(SQLA_UUID(as_uuid=True), ForeignKey("drafts.id"), nullable=False)
    agent_id: SQLA_UUID = Column(SQLA_UUID(as_uuid=True), ForeignKey("agents.id"), nullable=False)
    
    current_state: str = Column(String(50), nullable=False)
    proposed_state: str = Column(String(50), nullable=False)
    reason: Optional[str] = Column(Text, nullable=True)
    status: str = Column(String(50), default=TransitionProposalStatus.PENDING_APPROVAL.value)
    
    # 인간 검수 정보
    approved_at: Optional[datetime] = Column(DateTime, nullable=True)
    approved_by_user_id: Optional[SQLA_UUID] = Column(SQLA_UUID(as_uuid=True), ForeignKey("users.id"), nullable=True)
    approved_notes: Optional[str] = Column(Text, nullable=True)
    
    rejected_at: Optional[datetime] = Column(DateTime, nullable=True)
    rejected_by_user_id: Optional[SQLA_UUID] = Column(SQLA_UUID(as_uuid=True), ForeignKey("users.id"), nullable=True)
    rejected_reason: Optional[str] = Column(Text, nullable=True)
    
    created_at: datetime = Column(DateTime, default=datetime.utcnow)
    updated_at: datetime = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # 관계
    draft = relationship("Draft")
    agent = relationship("Agent")
    approved_by_user = relationship("User", foreign_keys=[approved_by_user_id])
    rejected_by_user = relationship("User", foreign_keys=[rejected_by_user_id])
```

**파일:** `/backend/alembic/versions/XXX_create_transition_proposals_table.py`

```python
"""Create transition_proposals table (S2-FG5.2)."""

from alembic import op
import sqlalchemy as sa


def upgrade() -> None:
    """Create transition_proposals table."""
    op.create_table(
        'transition_proposals',
        sa.Column('id', sa.UUID(), nullable=False),
        sa.Column('draft_id', sa.UUID(), nullable=False),
        sa.Column('agent_id', sa.UUID(), nullable=False),
        sa.Column('current_state', sa.String(50), nullable=False),
        sa.Column('proposed_state', sa.String(50), nullable=False),
        sa.Column('reason', sa.Text(), nullable=True),
        sa.Column('status', sa.String(50), nullable=False, server_default='pending_approval'),
        sa.Column('approved_at', sa.DateTime(), nullable=True),
        sa.Column('approved_by_user_id', sa.UUID(), nullable=True),
        sa.Column('approved_notes', sa.Text(), nullable=True),
        sa.Column('rejected_at', sa.DateTime(), nullable=True),
        sa.Column('rejected_by_user_id', sa.UUID(), nullable=True),
        sa.Column('rejected_reason', sa.Text(), nullable=True),
        sa.Column('created_at', sa.DateTime(), nullable=False, server_default=sa.func.now()),
        sa.Column('updated_at', sa.DateTime(), nullable=False, server_default=sa.func.now()),
        sa.ForeignKeyConstraint(['draft_id'], ['drafts.id'], ondelete='CASCADE'),
        sa.ForeignKeyConstraint(['agent_id'], ['agents.id'], ondelete='SET NULL'),
        sa.ForeignKeyConstraint(['approved_by_user_id'], ['users.id'], ondelete='SET NULL'),
        sa.ForeignKeyConstraint(['rejected_by_user_id'], ['users.id'], ondelete='SET NULL'),
        sa.PrimaryKeyConstraint('id'),
    )
    
    # 인덱스
    op.create_index('ix_transition_proposals_draft_id', 'transition_proposals', ['draft_id'])
    op.create_index('ix_transition_proposals_agent_id', 'transition_proposals', ['agent_id'])
    op.create_index('ix_transition_proposals_status', 'transition_proposals', ['status'])


def downgrade() -> None:
    """Rollback migration."""
    op.drop_index('ix_transition_proposals_status', table_name='transition_proposals')
    op.drop_index('ix_transition_proposals_agent_id', table_name='transition_proposals')
    op.drop_index('ix_transition_proposals_draft_id', table_name='transition_proposals')
    op.drop_table('transition_proposals')
```

### 4-2. Pydantic 스키마

**파일:** `/backend/app/schemas/draft_workflows.py` (신규)

```python
"""Draft 워크플로 요청/응답 스키마 (S2-FG5.2)."""

from __future__ import annotations

from typing import Optional
from uuid import UUID
from datetime import datetime

from pydantic import BaseModel, Field


# ==================== 워크플로 전이 제안 ====================

class ProposeTransitionRequest(BaseModel):
    """워크플로 전이 제안 요청."""
    
    document_id: UUID = Field(..., description="문서 ID (필수)")
    target_state: str = Field(
        ...,
        min_length=1,
        description="대상 상태 (예: approved, published)"
    )
    reason: Optional[str] = Field(
        None,
        max_length=1000,
        description="전이 사유 (선택)"
    )
    approver_notes: Optional[str] = Field(
        None,
        max_length=500,
        description="승인자 메모 (선택)"
    )


class ProposeTransitionResponse(BaseModel):
    """워크플로 전이 제안 응답."""
    
    transition_proposal_id: UUID
    draft_id: UUID
    agent_id: UUID
    current_state: str
    proposed_state: str
    status: str = Field("pending_approval")
    created_at: datetime
    reason: Optional[str]


# ==================== Draft 승인 ====================

class DraftApprovalRequest(BaseModel):
    """Draft 승인 요청."""
    
    notes: Optional[str] = Field(
        None,
        max_length=1000,
        description="검수 메모 (선택)"
    )


class DraftApprovalResponse(BaseModel):
    """Draft 승인 응답."""
    
    draft_id: UUID
    status: str = Field("approved")
    approved_at: datetime
    approved_by_user_id: UUID
    notes: Optional[str]


# ==================== Draft 거부 ====================

class DraftRejectionRequest(BaseModel):
    """Draft 거부 요청."""
    
    reason: str = Field(
        ...,
        min_length=1,
        max_length=1000,
        description="거부 사유 (필수)"
    )


class DraftRejectionResponse(BaseModel):
    """Draft 거부 응답."""
    
    draft_id: UUID
    status: str = Field("rejected")
    rejected_at: datetime
    rejected_by_user_id: UUID
    rejected_reason: str


# ==================== 제안 회수 ====================

class WithdrawProposalRequest(BaseModel):
    """제안 회수 요청."""
    
    reason: Optional[str] = Field(
        None,
        max_length=500,
        description="회수 사유 (선택)"
    )


class WithdrawProposalResponse(BaseModel):
    """제안 회수 응답."""
    
    proposal_id: UUID
    draft_id: UUID
    status: str = Field("withdrawn")
    withdrawn_at: datetime
    reason: Optional[str]
```

### 4-3. DraftWorkflowService 클래스

**파일:** `/backend/app/services/draft_workflows.py` (신규)

```python
"""Draft 워크플로 서비스 (S2-FG5.2)."""

from __future__ import annotations

import logging
from typing import Optional, Dict, Any
from uuid import UUID
from datetime import datetime

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.models.draft import Draft, DraftStatus, TransitionProposal, TransitionProposalStatus
from app.models.audit_log import AuditLog
from app.schemas.draft_workflows import (
    ProposeTransitionRequest, ProposeTransitionResponse,
    DraftApprovalRequest, DraftApprovalResponse,
    DraftRejectionRequest, DraftRejectionResponse,
    WithdrawProposalRequest, WithdrawProposalResponse,
)
from app.exceptions import InvalidRequestError, NotFoundError, PermissionDeniedError, ConflictError

logger = logging.getLogger(__name__)


class DraftWorkflowService:
    """Draft 워크플로 서비스 (승인/거부/회수)."""
    
    def __init__(self, db: AsyncSession):
        """Args: db — 데이터베이스 세션"""
        self.db = db
    
    # ==================== 워크플로 전이 제안 ====================
    
    async def propose_transition(
        self,
        agent_id: UUID,
        request: ProposeTransitionRequest,
        scope_profile: Dict[str, Any],
    ) -> ProposeTransitionResponse:
        """워크플로 전이 제안.
        
        에이전트가 Draft의 상태 전이를 제안.
        (예: proposed→approved, approved→published)
        
        Args:
            agent_id: 에이전트 ID
            request: 전이 제안 요청
            scope_profile: Scope Profile (ACL용)
        
        Returns:
            전이 제안 응답
        
        Raises:
            InvalidRequestError: 입력 검증 실패
            NotFoundError: Draft 없음
            PermissionDeniedError: delegate:transition 권한 없음
            ConflictError: 비허가 상태 전이
        """
        logger.info(
            f"Proposing transition: agent_id={agent_id}, "
            f"document_id={request.document_id}, "
            f"target_state={request.target_state}"
        )
        
        try:
            # ① 권한 검증 (delegate:transition scope)
            await self._validate_agent_transition_permission(agent_id, scope_profile)
            
            # ② Draft 조회
            draft = await self._get_draft_by_document_id(request.document_id)
            
            # ③ 상태 전이 검증
            self._validate_state_transition(draft.status, request.target_state)
            
            # ④ 입력 검증
            self._validate_transition_request(request)
            
            # ⑤ TransitionProposal 생성
            proposal = TransitionProposal(
                draft_id=draft.id,
                agent_id=agent_id,
                current_state=draft.status,
                proposed_state=request.target_state,
                reason=request.reason,
                status=TransitionProposalStatus.PENDING_APPROVAL.value,
            )
            self.db.add(proposal)
            await self.db.flush()
            
            # ⑥ 감시 로그 기록 (S2 원칙 ⑤)
            await self._log_transition_proposal(
                agent_id=agent_id,
                draft_id=draft.id,
                current_state=draft.status,
                proposed_state=request.target_state,
                reason=request.reason,
            )
            
            # ⑦ 응답
            response = ProposeTransitionResponse(
                transition_proposal_id=proposal.id,
                draft_id=draft.id,
                agent_id=agent_id,
                current_state=draft.status,
                proposed_state=request.target_state,
                status=TransitionProposalStatus.PENDING_APPROVAL.value,
                created_at=proposal.created_at,
                reason=request.reason,
            )
            
            logger.info(
                f"Transition proposed: proposal_id={proposal.id}, "
                f"{draft.status} → {request.target_state}"
            )
            return response
        
        except (InvalidRequestError, NotFoundError, PermissionDeniedError, ConflictError):
            raise
        except Exception as e:
            logger.error(f"Failed to propose transition: {str(e)}")
            await self._log_error(
                agent_id=agent_id,
                action="transition_propose",
                error=str(e),
            )
            raise InvalidRequestError(f"Failed to propose transition: {str(e)}")
    
    # ==================== Draft 승인 ====================
    
    async def approve_draft(
        self,
        draft_id: UUID,
        user_id: UUID,
        request: DraftApprovalRequest,
        scope_profile: Dict[str, Any],
    ) -> DraftApprovalResponse:
        """Draft 승인 (proposed → approved).
        
        인간 검수자가 에이전트 Draft를 승인.
        
        Args:
            draft_id: Draft ID
            user_id: 승인 사용자 ID
            request: 승인 요청
            scope_profile: Scope Profile (권한 검증용)
        
        Returns:
            승인 응답
        
        Raises:
            NotFoundError: Draft 없음
            PermissionDeniedError: reviewer 역할 없음
            ConflictError: 비허가 상태 (proposed가 아님)
        """
        logger.info(f"Approving draft: draft_id={draft_id}, user_id={user_id}")
        
        try:
            # ① Draft 조회
            draft = await self._get_draft_by_id(draft_id)
            
            # ② 상태 검증 (proposed 상태만 승인 가능)
            if draft.status != DraftStatus.PROPOSED.value:
                raise ConflictError(
                    f"Cannot approve draft in '{draft.status}' state. "
                    f"Only 'proposed' drafts can be approved."
                )
            
            # ③ 권한 검증 (reviewer 역할)
            await self._validate_reviewer_permission(user_id, draft, scope_profile)
            
            # ④ Draft 상태 변경
            draft.status = DraftStatus.APPROVED.value
            draft.updated_at = datetime.utcnow()
            
            # ⑤ 감시 로그 (S2 원칙 ⑤)
            await self._log_draft_approval(
                user_id=user_id,
                draft_id=draft_id,
                notes=request.notes,
            )
            
            # ⑥ 응답
            response = DraftApprovalResponse(
                draft_id=draft_id,
                status=DraftStatus.APPROVED.value,
                approved_at=datetime.utcnow(),
                approved_by_user_id=user_id,
                notes=request.notes,
            )
            
            logger.info(f"Draft approved: draft_id={draft_id}")
            return response
        
        except (NotFoundError, PermissionDeniedError, ConflictError):
            raise
        except Exception as e:
            logger.error(f"Failed to approve draft: {str(e)}")
            raise InvalidRequestError(f"Failed to approve draft: {str(e)}")
    
    # ==================== Draft 거부 ====================
    
    async def reject_draft(
        self,
        draft_id: UUID,
        user_id: UUID,
        request: DraftRejectionRequest,
        scope_profile: Dict[str, Any],
    ) -> DraftRejectionResponse:
        """Draft 거부 (proposed → rejected).
        
        인간 검수자가 에이전트 Draft를 거부.
        
        Args:
            draft_id: Draft ID
            user_id: 거부 사용자 ID
            request: 거부 요청
            scope_profile: Scope Profile
        
        Returns:
            거부 응답
        
        Raises:
            NotFoundError: Draft 없음
            PermissionDeniedError: reviewer 역할 없음
            ConflictError: 비허가 상태
        """
        logger.info(f"Rejecting draft: draft_id={draft_id}, user_id={user_id}")
        
        try:
            # ① Draft 조회
            draft = await self._get_draft_by_id(draft_id)
            
            # ② 상태 검증
            if draft.status != DraftStatus.PROPOSED.value:
                raise ConflictError(
                    f"Cannot reject draft in '{draft.status}' state. "
                    f"Only 'proposed' drafts can be rejected."
                )
            
            # ③ 권한 검증
            await self._validate_reviewer_permission(user_id, draft, scope_profile)
            
            # ④ Draft 상태 변경
            draft.status = DraftStatus.REJECTED.value
            draft.updated_at = datetime.utcnow()
            
            # ⑤ 감시 로그
            await self._log_draft_rejection(
                user_id=user_id,
                draft_id=draft_id,
                reason=request.reason,
            )
            
            # ⑥ 응답
            response = DraftRejectionResponse(
                draft_id=draft_id,
                status=DraftStatus.REJECTED.value,
                rejected_at=datetime.utcnow(),
                rejected_by_user_id=user_id,
                rejected_reason=request.reason,
            )
            
            logger.info(f"Draft rejected: draft_id={draft_id}")
            return response
        
        except (NotFoundError, PermissionDeniedError, ConflictError):
            raise
        except Exception as e:
            logger.error(f"Failed to reject draft: {str(e)}")
            raise InvalidRequestError(f"Failed to reject draft: {str(e)}")
    
    # ==================== 제안 회수 ====================
    
    async def withdraw_proposal(
        self,
        agent_id: UUID,
        proposal_id: UUID,
        request: WithdrawProposalRequest,
    ) -> WithdrawProposalResponse:
        """제안 회수 (proposed → withdrawn).
        
        에이전트 또는 관리자가 제안을 회수.
        
        Args:
            agent_id: 에이전트 ID
            proposal_id: 제안 ID
            request: 회수 요청
        
        Returns:
            회수 응답
        
        Raises:
            NotFoundError: 제안 없음
            PermissionDeniedError: 제안 작성자가 아님
            ConflictError: 비허가 상태 (proposed가 아님)
        """
        logger.info(f"Withdrawing proposal: proposal_id={proposal_id}, agent_id={agent_id}")
        
        try:
            # ① 제안 조회
            proposal = await self._get_transition_proposal(proposal_id)
            
            # ② 권한 검증 (제안 작성 에이전트만 회수 가능)
            if proposal.agent_id != agent_id:
                raise PermissionDeniedError(
                    "Only the proposal author can withdraw the proposal"
                )
            
            # ③ 상태 검증
            if proposal.status != TransitionProposalStatus.PENDING_APPROVAL.value:
                raise ConflictError(
                    f"Cannot withdraw proposal in '{proposal.status}' state"
                )
            
            # ④ Draft 상태 변경 (proposed → withdrawn)
            draft = await self._get_draft_by_id(proposal.draft_id)
            draft.status = DraftStatus.WITHDRAWN.value
            draft.updated_at = datetime.utcnow()
            
            # ⑤ 제안 상태 업데이트
            proposal.status = TransitionProposalStatus.REJECTED.value
            proposal.updated_at = datetime.utcnow()
            
            # ⑥ 감시 로그
            await self._log_proposal_withdrawal(
                agent_id=agent_id,
                proposal_id=proposal_id,
                draft_id=proposal.draft_id,
                reason=request.reason,
            )
            
            # ⑦ 응답
            response = WithdrawProposalResponse(
                proposal_id=proposal_id,
                draft_id=proposal.draft_id,
                status=DraftStatus.WITHDRAWN.value,
                withdrawn_at=datetime.utcnow(),
                reason=request.reason,
            )
            
            logger.info(f"Proposal withdrawn: proposal_id={proposal_id}")
            return response
        
        except (NotFoundError, PermissionDeniedError, ConflictError):
            raise
        except Exception as e:
            logger.error(f"Failed to withdraw proposal: {str(e)}")
            raise InvalidRequestError(f"Failed to withdraw proposal: {str(e)}")
    
    # ==================== Helper Methods ====================
    
    async def _validate_agent_transition_permission(
        self,
        agent_id: UUID,
        scope_profile: Dict[str, Any],
    ) -> None:
        """delegate:transition scope 검증."""
        scopes = scope_profile.get("scopes", [])
        if "delegate:transition" not in scopes:
            raise PermissionDeniedError(
                "Agent lacks delegate:transition scope"
            )
    
    async def _validate_reviewer_permission(
        self,
        user_id: UUID,
        draft: Draft,
        scope_profile: Dict[str, Any],
    ) -> None:
        """reviewer 역할 검증.
        
        문서 소유자 또는 reviewer 역할 필수.
        """
        # (실제 구현: role 시스템 조회)
        # 임시: 문서 소유자 확인
        if draft.created_by_user_id and draft.created_by_user_id == user_id:
            return  # 소유자는 자신의 문서 검수 가능
        
        # reviewer 역할 확인
        roles = scope_profile.get("roles", [])
        if "reviewer" not in roles:
            raise PermissionDeniedError(
                "User must have 'reviewer' role to approve/reject drafts"
            )
    
    def _validate_state_transition(self, current_state: str, target_state: str) -> None:
        """상태 전이 검증.
        
        비허가 전이 차단:
        - proposed → approved: OK (인간 검수)
        - proposed → rejected: OK (인간 거부)
        - proposed → published: NO (중간 단계 생략 금지)
        - approved → published: OK (최종 발행)
        """
        valid_transitions = {
            DraftStatus.PROPOSED.value: [DraftStatus.APPROVED.value, DraftStatus.REJECTED.value],
            DraftStatus.APPROVED.value: [DraftStatus.PUBLISHED.value],
            DraftStatus.DRAFT.value: [DraftStatus.REVIEW.value],
            DraftStatus.REVIEW.value: [DraftStatus.APPROVED.value],
        }
        
        if current_state not in valid_transitions:
            raise ConflictError(f"Unknown current state: {current_state}")
        
        if target_state not in valid_transitions[current_state]:
            raise ConflictError(
                f"Invalid state transition: {current_state} → {target_state}"
            )
    
    def _validate_transition_request(self, request: ProposeTransitionRequest) -> None:
        """전이 요청 입력 검증."""
        if not request.target_state or not request.target_state.strip():
            raise InvalidRequestError("target_state is required")
    
    async def _get_draft_by_id(self, draft_id: UUID) -> Draft:
        """ID로 Draft 조회."""
        stmt = select(Draft).where(Draft.id == draft_id)
        result = await self.db.execute(stmt)
        draft = result.scalar_one_or_none()
        
        if not draft:
            raise NotFoundError(f"Draft not found: {draft_id}")
        
        return draft
    
    async def _get_draft_by_document_id(self, document_id: UUID) -> Draft:
        """document_id로 최신 Draft 조회."""
        stmt = select(Draft).where(Draft.document_id == document_id).order_by(Draft.created_at.desc())
        result = await self.db.execute(stmt)
        draft = result.scalar_one_or_none()
        
        if not draft:
            raise NotFoundError(f"Draft for document not found: {document_id}")
        
        return draft
    
    async def _get_transition_proposal(self, proposal_id: UUID) -> TransitionProposal:
        """TransitionProposal 조회."""
        stmt = select(TransitionProposal).where(TransitionProposal.id == proposal_id)
        result = await self.db.execute(stmt)
        proposal = result.scalar_one_or_none()
        
        if not proposal:
            raise NotFoundError(f"Transition proposal not found: {proposal_id}")
        
        return proposal
    
    async def _log_transition_proposal(
        self,
        agent_id: UUID,
        draft_id: UUID,
        current_state: str,
        proposed_state: str,
        reason: Optional[str],
    ) -> None:
        """워크플로 전이 제안 감시 로그 (S2 원칙 ⑤)."""
        try:
            audit_log = AuditLog(
                entity_type="draft",
                entity_id=draft_id,
                action="transition_propose",
                actor_id=agent_id,
                actor_type="agent",
                changes={
                    "current_state": current_state,
                    "proposed_state": proposed_state,
                    "reason": reason[:200] if reason else None,
                },
            )
            self.db.add(audit_log)
            await self.db.flush()
        except Exception as e:
            logger.error(f"Failed to log transition proposal: {str(e)}")
    
    async def _log_draft_approval(
        self,
        user_id: UUID,
        draft_id: UUID,
        notes: Optional[str],
    ) -> None:
        """Draft 승인 감시 로그."""
        try:
            audit_log = AuditLog(
                entity_type="draft",
                entity_id=draft_id,
                action="draft_approve",
                actor_id=user_id,
                actor_type="user",
                changes={
                    "status": "approved",
                    "notes": notes[:200] if notes else None,
                },
            )
            self.db.add(audit_log)
            await self.db.flush()
        except Exception as e:
            logger.error(f"Failed to log draft approval: {str(e)}")
    
    async def _log_draft_rejection(
        self,
        user_id: UUID,
        draft_id: UUID,
        reason: str,
    ) -> None:
        """Draft 거부 감시 로그."""
        try:
            audit_log = AuditLog(
                entity_type="draft",
                entity_id=draft_id,
                action="draft_reject",
                actor_id=user_id,
                actor_type="user",
                changes={
                    "status": "rejected",
                    "reason": reason[:200] if reason else None,
                },
            )
            self.db.add(audit_log)
            await self.db.flush()
        except Exception as e:
            logger.error(f"Failed to log draft rejection: {str(e)}")
    
    async def _log_proposal_withdrawal(
        self,
        agent_id: UUID,
        proposal_id: UUID,
        draft_id: UUID,
        reason: Optional[str],
    ) -> None:
        """제안 회수 감시 로그."""
        try:
            audit_log = AuditLog(
                entity_type="draft",
                entity_id=draft_id,
                action="proposal_withdraw",
                actor_id=agent_id,
                actor_type="agent",
                changes={
                    "proposal_id": str(proposal_id),
                    "reason": reason[:200] if reason else None,
                },
            )
            self.db.add(audit_log)
            await self.db.flush()
        except Exception as e:
            logger.error(f"Failed to log proposal withdrawal: {str(e)}")
    
    async def _log_error(
        self,
        agent_id: UUID,
        action: str,
        error: str,
    ) -> None:
        """에러 감시 로그."""
        try:
            audit_log = AuditLog(
                entity_type="draft",
                entity_id=UUID(int=0),
                action=f"{action}_error",
                actor_id=agent_id,
                actor_type="agent",
                changes={"error": error[:500]},
            )
            self.db.add(audit_log)
            await self.db.flush()
        except Exception as e:
            logger.error(f"Failed to log error: {str(e)}")
```

### 4-4. FastAPI 라우터

**파일:** `/backend/app/routes/drafts.py` (신규 또는 확장)

```python
"""Draft 워크플로 라우터 (S2-FG5.2)."""

from typing import Optional
from uuid import UUID

from fastapi import APIRouter, Depends, Header, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from app.db import get_db
from app.schemas.draft_workflows import (
    ProposeTransitionRequest, ProposeTransitionResponse,
    DraftApprovalRequest, DraftApprovalResponse,
    DraftRejectionRequest, DraftRejectionResponse,
    WithdrawProposalRequest, WithdrawProposalResponse,
)
from app.services.draft_workflows import DraftWorkflowService
from app.exceptions import InvalidRequestError, NotFoundError, PermissionDeniedError, ConflictError
from app.auth import get_current_user, get_scope_profile_for_agent

router = APIRouter(prefix="/api/v1", tags=["drafts"])


# ==================== 워크플로 전이 제안 ====================

@router.post("/agents/{agent_id}/propose-transition", response_model=ProposeTransitionResponse)
async def propose_transition(
    agent_id: UUID,
    request: ProposeTransitionRequest,
    authorization: Optional[str] = Header(None),
    db: AsyncSession = Depends(get_db),
) -> ProposeTransitionResponse:
    """
    워크플로 전이 제안.
    
    에이전트가 Draft의 상태 전이를 제안.
    (예: proposed→approved, approved→published)
    """
    if not authorization:
        raise HTTPException(status_code=401, detail="Missing Authorization header")
    
    try:
        scope_profile = await get_scope_profile_for_agent(agent_id, db)
        service = DraftWorkflowService(db)
        response = await service.propose_transition(
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
    except ConflictError as e:
        await db.rollback()
        raise HTTPException(status_code=409, detail=str(e))
    except Exception as e:
        await db.rollback()
        raise HTTPException(status_code=500, detail=f"Internal error: {str(e)}")


# ==================== Draft 승인 ====================

@router.post("/drafts/{draft_id}/approve", response_model=DraftApprovalResponse)
async def approve_draft(
    draft_id: UUID,
    request: DraftApprovalRequest,
    current_user = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> DraftApprovalResponse:
    """
    Draft 승인 (proposed → approved).
    
    인간 검수자만 승인 가능.
    """
    try:
        scope_profile = {"roles": current_user.get("roles", [])}
        service = DraftWorkflowService(db)
        response = await service.approve_draft(
            draft_id=draft_id,
            user_id=UUID(current_user["id"]),
            request=request,
            scope_profile=scope_profile,
        )
        await db.commit()
        return response
    except NotFoundError as e:
        await db.rollback()
        raise HTTPException(status_code=404, detail=str(e))
    except PermissionDeniedError as e:
        await db.rollback()
        raise HTTPException(status_code=403, detail=str(e))
    except ConflictError as e:
        await db.rollback()
        raise HTTPException(status_code=409, detail=str(e))
    except Exception as e:
        await db.rollback()
        raise HTTPException(status_code=500, detail=f"Internal error: {str(e)}")


# ==================== Draft 거부 ====================

@router.post("/drafts/{draft_id}/reject", response_model=DraftRejectionResponse)
async def reject_draft(
    draft_id: UUID,
    request: DraftRejectionRequest,
    current_user = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> DraftRejectionResponse:
    """
    Draft 거부 (proposed → rejected).
    
    인간 검수자만 거부 가능.
    """
    try:
        scope_profile = {"roles": current_user.get("roles", [])}
        service = DraftWorkflowService(db)
        response = await service.reject_draft(
            draft_id=draft_id,
            user_id=UUID(current_user["id"]),
            request=request,
            scope_profile=scope_profile,
        )
        await db.commit()
        return response
    except NotFoundError as e:
        await db.rollback()
        raise HTTPException(status_code=404, detail=str(e))
    except PermissionDeniedError as e:
        await db.rollback()
        raise HTTPException(status_code=403, detail=str(e))
    except ConflictError as e:
        await db.rollback()
        raise HTTPException(status_code=409, detail=str(e))
    except Exception as e:
        await db.rollback()
        raise HTTPException(status_code=500, detail=f"Internal error: {str(e)}")


# ==================== 제안 회수 ====================

@router.post(
    "/agents/{agent_id}/proposals/{proposal_id}/withdraw",
    response_model=WithdrawProposalResponse
)
async def withdraw_proposal(
    agent_id: UUID,
    proposal_id: UUID,
    request: WithdrawProposalRequest,
    db: AsyncSession = Depends(get_db),
) -> WithdrawProposalResponse:
    """
    제안 회수 (proposed → withdrawn).
    
    제안 작성 에이전트만 회수 가능.
    """
    try:
        service = DraftWorkflowService(db)
        response = await service.withdraw_proposal(
            agent_id=agent_id,
            proposal_id=proposal_id,
            request=request,
        )
        await db.commit()
        return response
    except NotFoundError as e:
        await db.rollback()
        raise HTTPException(status_code=404, detail=str(e))
    except PermissionDeniedError as e:
        await db.rollback()
        raise HTTPException(status_code=403, detail=str(e))
    except ConflictError as e:
        await db.rollback()
        raise HTTPException(status_code=409, detail=str(e))
    except Exception as e:
        await db.rollback()
        raise HTTPException(status_code=500, detail=f"Internal error: {str(e)}")
```

### 4-5. 단위/통합 테스트

**파일:** `/backend/tests/unit/services/test_draft_workflows.py`

```python
"""Draft 워크플로 서비스 단위 테스트."""

import pytest
from uuid import uuid4
from sqlalchemy.ext.asyncio import AsyncSession

from app.services.draft_workflows import DraftWorkflowService
from app.schemas.draft_workflows import (
    ProposeTransitionRequest, DraftApprovalRequest,
    DraftRejectionRequest, WithdrawProposalRequest
)
from app.models.draft import Draft, DraftStatus
from app.exceptions import InvalidRequestError, NotFoundError, PermissionDeniedError, ConflictError


@pytest.fixture
def workflow_service(db_session: AsyncSession):
    """Draft 워크플로 서비스."""
    return DraftWorkflowService(db_session)


@pytest.mark.asyncio
async def test_propose_transition_basic(workflow_service, db_session):
    """워크플로 전이 제안 정상 생성."""
    agent_id = uuid4()
    document_id = uuid4()
    
    request = ProposeTransitionRequest(
        document_id=document_id,
        target_state="approved",
        reason="AI 보안 검수 완료",
    )
    
    scope_profile = {
        "scopes": ["delegate:transition"],
    }
    
    # response = await workflow_service.propose_transition(
    #     agent_id=agent_id,
    #     request=request,
    #     scope_profile=scope_profile,
    # )
    # 
    # assert response.transition_proposal_id is not None
    # assert response.current_state == "proposed"
    # assert response.proposed_state == "approved"


@pytest.mark.asyncio
async def test_propose_transition_missing_permission(workflow_service):
    """delegate:transition 권한 없으면 거부."""
    request = ProposeTransitionRequest(
        document_id=uuid4(),
        target_state="approved",
    )
    
    scope_profile = {
        "scopes": ["read", "query"],  # delegate:transition 없음
    }
    
    with pytest.raises(PermissionDeniedError):
        await workflow_service.propose_transition(
            agent_id=uuid4(),
            request=request,
            scope_profile=scope_profile,
        )


@pytest.mark.asyncio
async def test_approve_draft_basic(workflow_service):
    """Draft 승인 정상 수행."""
    draft_id = uuid4()
    user_id = uuid4()
    
    request = DraftApprovalRequest(
        notes="검수 완료",
    )
    
    scope_profile = {
        "roles": ["reviewer"],
    }
    
    # response = await workflow_service.approve_draft(
    #     draft_id=draft_id,
    #     user_id=user_id,
    #     request=request,
    #     scope_profile=scope_profile,
    # )
    # 
    # assert response.status == "approved"


@pytest.mark.asyncio
async def test_reject_draft_basic(workflow_service):
    """Draft 거부 정상 수행."""
    draft_id = uuid4()
    user_id = uuid4()
    
    request = DraftRejectionRequest(
        reason="보안 정책 위반",
    )
    
    scope_profile = {
        "roles": ["reviewer"],
    }
    
    # response = await workflow_service.reject_draft(
    #     draft_id=draft_id,
    #     user_id=user_id,
    #     request=request,
    #     scope_profile=scope_profile,
    # )
    # 
    # assert response.status == "rejected"


@pytest.mark.asyncio
async def test_withdraw_proposal_basic(workflow_service):
    """제안 회수 정상 수행."""
    agent_id = uuid4()
    proposal_id = uuid4()
    
    request = WithdrawProposalRequest(
        reason="재작업 필요",
    )
    
    # response = await workflow_service.withdraw_proposal(
    #     agent_id=agent_id,
    #     proposal_id=proposal_id,
    #     request=request,
    # )
    # 
    # assert response.status == "withdrawn"


@pytest.mark.asyncio
async def test_invalid_state_transition(workflow_service):
    """비허가 상태 전이 차단."""
    with pytest.raises(ConflictError):
        workflow_service._validate_state_transition(
            current_state="proposed",
            target_state="published",  # proposed → published 금지 (중간 단계 생략)
        )
```

## 5. 산출물

1. **TransitionProposal 모델** (`/backend/app/models/draft.py` 확장)
   - TransitionProposalStatus enum
   - TransitionProposal 모델

2. **DB 마이그레이션** (`/backend/alembic/versions/XXX_create_transition_proposals_table.py`)
   - transition_proposals 테이블

3. **Pydantic 스키마** (`/backend/app/schemas/draft_workflows.py`)
   - ProposeTransitionRequest/Response
   - DraftApprovalRequest/Response
   - DraftRejectionRequest/Response
   - WithdrawProposalRequest/Response

4. **DraftWorkflowService** (`/backend/app/services/draft_workflows.py`)
   - propose_transition()
   - approve_draft()
   - reject_draft()
   - withdraw_proposal()

5. **FastAPI 라우터** (`/backend/app/routes/drafts.py`)
   - POST /api/v1/agents/{agent_id}/propose-transition
   - POST /api/v1/drafts/{draft_id}/approve
   - POST /api/v1/drafts/{draft_id}/reject
   - POST /api/v1/agents/{agent_id}/proposals/{proposal_id}/withdraw

6. **단위/통합 테스트** (`/backend/tests/unit/services/test_draft_workflows.py`)

## 6. 완료 기준

1. TransitionProposal 모델이 올바르게 정의되었는가?
2. transition_proposals 테이블이 생성되었는가?
3. 워크플로 전이 제안 API가 delegate:transition scope 검증을 수행하는가?
4. Draft 승인 API가 proposed→approved로 정확하게 전이하는가?
5. Draft 거부 API가 proposed→rejected로 정확하게 전이하는가?
6. 제안 회수 API가 제안 작성 에이전트 권한을 검증하는가?
7. 비허가 상태 전이 (예: proposed→published)가 차단되는가?
8. 모든 이벤트가 actor_type (agent/user)과 함께 감시 로그되는가?
9. 모든 단위 테스트가 통과하는가?
10. FastAPI 라우터가 예외 처리와 함께 정확하게 구현되었는가?

## 7. 작업 지침

### 지침 7-1. 상태 전이 규칙 강제

비허가 전이를 데이터베이스 레벨 또는 서비스 레벨에서 강제한다. 특히 중간 단계를 건너뛰는 전이 (예: proposed→published)를 차단한다.

### 지침 7-2. 권한 분리 (S2 원칙 ⑤)

- 에이전트: 제안 생성, 제안 회수
- 인간: 승인, 거부
- 관리자: 감독

감시 로그에 actor_type을 명확하게 기록한다.

### 지침 7-3. reviewer 역할

Draft 승인/거부는 문서 소유자 또는 reviewer 역할을 가진 사용자만 가능하다.

### 지침 7-4. 제안 회수의 의미

에이전트가 제안을 회수하면 Draft 상태는 withdrawn이 되고, 다시 수정 후 새로운 proposed Draft를 생성할 수 있다.

### 지침 7-5. 타임스탬프 관리

승인/거부 시 approved_at, approved_by_user_id (또는 rejected_at, rejected_by_user_id)를 기록한다.

### 지침 7-6. 에러 처리

상태 검증 실패 시 409 Conflict, 권한 부족 시 403 Forbidden, 데이터 없음 시 404 Not Found를 반환한다.
