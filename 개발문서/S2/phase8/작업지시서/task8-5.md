# Task 8-5. 추출 결과 검토 API + ApprovedExtraction 저장

## 1. 작업 목적

Task 8-4에서 생성된 `ExtractionCandidate`를 사용자가 검토하고, 승인/수정/거절하는 **추출 결과 검토 API**를 구현한다. 사용자의 승인 결정을 `ApprovedExtraction` artifact로 저장하여 Document.metadata와 독립적으로 관리하고(S1 원칙 ②: generic + config), Scope Profile 기반 ACL과 감시 로깅(S2 원칙 ⑤, ⑥)을 준수한다. 추출 결과의 버전 관리와 인간의 수정 이력을 추적 가능하게 한다.

## 2. 작업 범위

### 포함 범위

1. **ApprovedExtraction 도메인 모델 및 ORM**
   - ApprovedExtraction (Pydantic): document_id, document_version, scope_id, extraction_schema_version, extraction_model, human_edits, approved_at, approved_by
   - SQLAlchemy ORM: approved_extractions 테이블
   - 복합 키: (document_id, document_version, extraction_schema_version)
   - Scope Profile 기반 ACL: scope_id 필드

2. **추출 결과 검토 API 엔드포인트**
   - `GET /api/v1/extractions/pending` — pending 상태의 추출 결과 목록 (페이지네이션)
   - `GET /api/v1/extractions/{id}` — 추출 결과 상세 조회 (문서 미리보기 정보 포함)
   - `POST /api/v1/extractions/{id}/approve` — 모든 필드 자동 추출값 승인
   - `POST /api/v1/extractions/{id}/modify` — 일부 필드 수정 후 승인
   - `POST /api/v1/extractions/{id}/reject` — 전체 거절 (feedback 필수)
   - `POST /api/v1/extractions/batch-approve` — 일괄 승인 (여러 추출 결과)
   - `POST /api/v1/extractions/batch-reject` — 일괄 거절

3. **ApprovedExtraction Repository**
   - CRUD 연산 (create, read, update, delete)
   - Scope Profile 필터 적용 (모든 조회에 의무 적용)
   - 폐쇄망 환경: 외부 의존 없이 로컬 DB에서만 쿼리

4. **Alembic 마이그레이션**
   - approved_extractions 테이블 생성
   - 복합 유니크 제약: (document_id, document_version, extraction_schema_version)
   - 인덱스: (scope_id, approved_at), (document_id, version)

5. **감시 로깅 (S2 원칙 ⑤)**
   - 승인 시: action="extraction_approved", actor_type="user", approved_by, field_count
   - 수정 시: action="extraction_modified", human_edits (필드별 수정 기록)
   - 거절 시: action="extraction_rejected", reason

6. **Projection/Materialized View (선택사항)**
   - ApprovedExtraction을 Document.metadata로 반영하는 SQL view
   - Document 조회 시 최신 approved extraction을 자동으로 JOIN

7. **단위 및 통합 테스트**
   - 승인/수정/거절 흐름 검증
   - Scope Profile 필터링 정확성
   - 동시성 제어 (동일 문서 추출 결과 동시 승인 방지)
   - API 응답 스키마 검증

### 제외 범위

- 추출 파이프라인 (Task 8-4)
- ExtractionTargetSchema CRUD API (Task 8-3)
- 검토 UI (Task 8-6)
- 재실행 배치 (Task 8-7)

## 3. 선행 조건

- Task 8-4 완료: ExtractionCandidate 도메인 모델 및 DB 스키마
- PostgreSQL + SQLAlchemy + Alembic 설정
- Pydantic v2 설치
- FastAPI 라우터 설정 완료

## 4. 주요 작업 항목

### 4-1. ApprovedExtraction 도메인 모델

**파일:** `/backend/app/models/approved_extraction.py`

```python
"""
ApprovedExtraction 도메인 모델.

사용자 승인 후 최종 저장되는 추출 결과 artifact.
Document.metadata와 별도로 관리됨 (S1 원칙 ②: generic DB model).
"""

from __future__ import annotations

from datetime import datetime
from typing import Any, Dict, List, Optional
from uuid import UUID

from pydantic import BaseModel, Field, validator


class HumanEdit(BaseModel):
    """인간이 수정한 필드의 기록."""
    field_name: str
    before_value: Any
    after_value: Any
    edited_at: datetime
    edited_by: UUID
    reason: Optional[str] = None  # 수정 이유


class ApprovedExtraction(BaseModel):
    """
    최종 승인된 추출 결과 artifact.
    
    Document과 분리하여 버전 관리 (S1 원칙 ②).
    """
    id: UUID = Field(
        default_factory=lambda: UUID('00000000-0000-0000-0000-000000000000')
    )
    
    # ==================== Document Reference ====================
    document_id: UUID
    document_version: int
    scope_id: UUID  # S2: Scope Profile ACL
    
    # ==================== Extraction Schema ====================
    extraction_schema_id: str  # DocumentType ID
    extraction_schema_version: int
    
    # ==================== Extraction Metadata ====================
    extraction_model: str  # 사용한 모델명
    extraction_latency_ms: int
    extraction_tokens: Optional[Dict[str, int]] = None
    extraction_cost_estimate: Optional[float] = None
    extraction_prompt_version: Optional[str] = None
    
    # ==================== Approved Data ====================
    approved_fields: Dict[str, Any] = Field(
        default_factory=dict,
        description="최종 승인된 필드값 {field_name: approved_value}"
    )
    
    # ==================== Human Edits ====================
    human_edits: List[HumanEdit] = Field(
        default_factory=list,
        description="인간이 수정한 필드 기록"
    )
    
    # ==================== Approval Info ====================
    approved_by: UUID
    approved_at: datetime
    approval_comment: Optional[str] = None
    
    # ==================== Audit ====================
    actor_type: str = "user"  # S2 원칙 ⑤
    created_at: datetime
    updated_at: datetime
    
    class Config:
        use_enum_values = False
        json_encoders = {
            datetime: lambda v: v.isoformat(),
            UUID: lambda v: str(v),
        }
    
    @validator('approved_fields')
    def validate_approved_fields_not_empty(cls, v):
        """최소 하나의 필드는 승인되어야 함."""
        if not v:
            raise ValueError("approved_fields must contain at least one field")
        return v


class ApprovedExtractionResponse(BaseModel):
    """조회 응답."""
    id: UUID
    document_id: UUID
    document_version: int
    extraction_schema_id: str
    extraction_schema_version: int
    approved_fields: Dict[str, Any]
    human_edits: List[HumanEdit]
    approved_by: UUID
    approved_at: datetime
    approval_comment: Optional[str]
    created_at: datetime


# ==================== Request DTOs ====================

class ApproveExtractionRequest(BaseModel):
    """모든 필드 자동 값 승인."""
    approval_comment: Optional[str] = None


class ModifyExtractionRequest(BaseModel):
    """필드 수정 후 승인."""
    modifications: Dict[str, Any] = Field(
        description="수정할 필드: {field_name: new_value}"
    )
    reasons: Optional[Dict[str, str]] = None  # 수정 이유: {field_name: reason}
    approval_comment: Optional[str] = None


class RejectExtractionRequest(BaseModel):
    """추출 결과 거절."""
    reason: str = Field(
        min_length=1,
        max_length=500,
        description="거절 이유 (필수)"
    )


class BatchApproveRequest(BaseModel):
    """일괄 승인."""
    extraction_ids: List[UUID] = Field(
        min_items=1,
        max_items=100,
        description="승인할 추출 결과 ID 목록"
    )
    approval_comment: Optional[str] = None


class BatchRejectRequest(BaseModel):
    """일괄 거절."""
    extraction_ids: List[UUID] = Field(
        min_items=1,
        max_items=100,
        description="거절할 추출 결과 ID 목록"
    )
    reason: str = Field(
        min_length=1,
        max_length=500,
        description="거절 이유"
    )
```

### 4-2. ApprovedExtraction ORM 및 마이그레이션

**파일:** `/backend/app/db/models/approved_extraction.py`

```python
"""
ApprovedExtraction SQLAlchemy ORM.
"""

from datetime import datetime
from typing import Optional
from uuid import UUID

from sqlalchemy import (
    Boolean, Column, DateTime, Float, ForeignKey, 
    Integer, JSON, String, Text, UniqueConstraint, Index
)
from sqlalchemy.dialects.postgresql import UUID as PG_UUID

from app.db.base import Base


class ApprovedExtractionORM(Base):
    """
    최종 승인된 추출 결과 ORM.
    
    테이블: approved_extractions
    복합 키: (document_id, document_version, extraction_schema_version)
    """
    
    __tablename__ = "approved_extractions"
    
    # ==================== Primary Key ====================
    id = Column(
        PG_UUID(as_uuid=True),
        primary_key=True,
        nullable=False,
        index=True
    )
    
    # ==================== Document Reference ====================
    document_id = Column(
        PG_UUID(as_uuid=True),
        nullable=False,
        index=True,
        comment="Document ID"
    )
    document_version = Column(
        Integer,
        nullable=False
    )
    
    # ==================== Scope & ACL ====================
    scope_id = Column(
        PG_UUID(as_uuid=True),
        nullable=False,
        index=True,
        comment="Scope Profile ID"
    )
    
    # ==================== Extraction Schema ====================
    extraction_schema_id = Column(
        String(255),
        nullable=False
    )
    extraction_schema_version = Column(
        Integer,
        nullable=False
    )
    
    # ==================== Extraction Metadata ====================
    extraction_model = Column(
        String(255),
        nullable=False
    )
    extraction_latency_ms = Column(
        Integer,
        nullable=False
    )
    extraction_tokens = Column(
        JSON,
        nullable=True,
        comment="{input: int, output: int}"
    )
    extraction_cost_estimate = Column(
        Float,
        nullable=True
    )
    extraction_prompt_version = Column(
        String(64),
        nullable=True
    )
    
    # ==================== Approved Data ====================
    approved_fields = Column(
        JSON,
        nullable=False,
        comment="최종 승인된 필드값"
    )
    
    # ==================== Human Edits ====================
    human_edits = Column(
        JSON,
        nullable=False,
        default=[],
        comment="[{field_name, before_value, after_value, edited_at, edited_by, reason}]"
    )
    
    # ==================== Approval Info ====================
    approved_by = Column(
        PG_UUID(as_uuid=True),
        nullable=False,
        index=True
    )
    approved_at = Column(
        DateTime(timezone=True),
        nullable=False,
        index=True,
        default=datetime.utcnow
    )
    approval_comment = Column(
        Text,
        nullable=True
    )
    
    # ==================== Audit ====================
    actor_type = Column(
        String(32),
        nullable=False,
        default="user",
        comment="'user' or 'agent'"
    )
    
    # ==================== Timestamps ====================
    created_at = Column(
        DateTime(timezone=True),
        nullable=False,
        default=datetime.utcnow
    )
    updated_at = Column(
        DateTime(timezone=True),
        nullable=False,
        default=datetime.utcnow,
        onupdate=datetime.utcnow
    )
    
    # ==================== Soft Delete ====================
    deleted_at = Column(
        DateTime(timezone=True),
        nullable=True
    )
    is_deleted = Column(
        Boolean,
        nullable=False,
        default=False,
        index=True
    )
    
    # ==================== Constraints ====================
    __table_args__ = (
        # 복합 유니크 제약
        UniqueConstraint(
            'document_id', 'document_version', 'extraction_schema_version',
            name='uq_approved_extraction_version',
            postgresql_where='is_deleted = false'
        ),
        
        # 복합 인덱스
        Index('idx_approved_extraction_scope_time',
              'scope_id', 'approved_at'),
        Index('idx_approved_extraction_document',
              'document_id', 'document_version'),
    )
```

### 4-3. ApprovedExtractionRepository

**파일:** `/backend/app/repositories/approved_extraction_repository.py`

```python
"""
ApprovedExtractionRepository.

Scope Profile 기반 ACL 적용 (S2 원칙 ⑥).
"""

from datetime import datetime
from typing import Optional, List
from uuid import UUID

from sqlalchemy.orm import Session
from sqlalchemy import and_

from app.db.models.approved_extraction import ApprovedExtractionORM
from app.models.approved_extraction import ApprovedExtraction
from app.core.scope import ScopeProfile, enforce_scope_acl


class ApprovedExtractionRepository:
    """ApprovedExtraction 저장소."""
    
    def __init__(self, db: Session, scope_profile: ScopeProfile):
        self.db = db
        self.scope_profile = scope_profile
    
    async def create(
        self,
        approved_extraction: ApprovedExtraction
    ) -> ApprovedExtraction:
        """새 ApprovedExtraction 생성."""
        enforce_scope_acl(
            self.scope_profile,
            approved_extraction.scope_id,
            action="create_approved_extraction"
        )
        
        orm = ApprovedExtractionORM(
            id=approved_extraction.id,
            document_id=approved_extraction.document_id,
            document_version=approved_extraction.document_version,
            scope_id=approved_extraction.scope_id,
            extraction_schema_id=approved_extraction.extraction_schema_id,
            extraction_schema_version=approved_extraction.extraction_schema_version,
            extraction_model=approved_extraction.extraction_model,
            extraction_latency_ms=approved_extraction.extraction_latency_ms,
            extraction_tokens=approved_extraction.extraction_tokens,
            extraction_cost_estimate=approved_extraction.extraction_cost_estimate,
            extraction_prompt_version=approved_extraction.extraction_prompt_version,
            approved_fields=approved_extraction.approved_fields,
            human_edits=[e.dict() for e in approved_extraction.human_edits],
            approved_by=approved_extraction.approved_by,
            approved_at=approved_extraction.approved_at,
            approval_comment=approved_extraction.approval_comment,
            actor_type=approved_extraction.actor_type,
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow()
        )
        
        self.db.add(orm)
        self.db.flush()
        
        return self._to_model(orm)
    
    async def get_by_id(
        self,
        extraction_id: UUID,
        scope_id: UUID
    ) -> Optional[ApprovedExtraction]:
        """ID로 조회 (Scope 필터 적용)."""
        enforce_scope_acl(self.scope_profile, scope_id, action="read_approved_extraction")
        
        orm = self.db.query(ApprovedExtractionORM).filter(
            and_(
                ApprovedExtractionORM.id == extraction_id,
                ApprovedExtractionORM.scope_id == scope_id,
                ApprovedExtractionORM.is_deleted == False
            )
        ).first()
        
        return self._to_model(orm) if orm else None
    
    async def get_by_document_and_schema(
        self,
        document_id: UUID,
        document_version: int,
        extraction_schema_version: int,
        scope_id: UUID
    ) -> Optional[ApprovedExtraction]:
        """문서 버전 + 스키마 버전으로 조회 (복합 키)."""
        enforce_scope_acl(self.scope_profile, scope_id, action="read_approved_extraction")
        
        orm = self.db.query(ApprovedExtractionORM).filter(
            and_(
                ApprovedExtractionORM.document_id == document_id,
                ApprovedExtractionORM.document_version == document_version,
                ApprovedExtractionORM.extraction_schema_version == extraction_schema_version,
                ApprovedExtractionORM.scope_id == scope_id,
                ApprovedExtractionORM.is_deleted == False
            )
        ).first()
        
        return self._to_model(orm) if orm else None
    
    async def list_by_document(
        self,
        document_id: UUID,
        scope_id: UUID,
        offset: int = 0,
        limit: int = 50
    ) -> tuple[List[ApprovedExtraction], int]:
        """문서별 승인된 추출 결과 목록 조회."""
        enforce_scope_acl(self.scope_profile, scope_id, action="read_approved_extraction")
        
        query = self.db.query(ApprovedExtractionORM).filter(
            and_(
                ApprovedExtractionORM.document_id == document_id,
                ApprovedExtractionORM.scope_id == scope_id,
                ApprovedExtractionORM.is_deleted == False
            )
        ).order_by(ApprovedExtractionORM.approved_at.desc())
        
        total = query.count()
        items = query.offset(offset).limit(limit).all()
        
        return [self._to_model(orm) for orm in items], total
    
    async def list_recent(
        self,
        scope_id: UUID,
        offset: int = 0,
        limit: int = 50
    ) -> tuple[List[ApprovedExtraction], int]:
        """최근 승인된 추출 결과 목록 (scope 내)."""
        enforce_scope_acl(self.scope_profile, scope_id, action="read_approved_extraction")
        
        query = self.db.query(ApprovedExtractionORM).filter(
            and_(
                ApprovedExtractionORM.scope_id == scope_id,
                ApprovedExtractionORM.is_deleted == False
            )
        ).order_by(ApprovedExtractionORM.approved_at.desc())
        
        total = query.count()
        items = query.offset(offset).limit(limit).all()
        
        return [self._to_model(orm) for orm in items], total
    
    async def update(
        self,
        extraction_id: UUID,
        scope_id: UUID,
        updates: dict
    ) -> Optional[ApprovedExtraction]:
        """업데이트 (버전 관리)."""
        enforce_scope_acl(self.scope_profile, scope_id, action="update_approved_extraction")
        
        orm = self.db.query(ApprovedExtractionORM).filter(
            and_(
                ApprovedExtractionORM.id == extraction_id,
                ApprovedExtractionORM.scope_id == scope_id,
                ApprovedExtractionORM.is_deleted == False
            )
        ).first()
        
        if not orm:
            return None
        
        for key, value in updates.items():
            if hasattr(orm, key):
                setattr(orm, key, value)
        
        orm.updated_at = datetime.utcnow()
        self.db.flush()
        
        return self._to_model(orm)
    
    async def soft_delete(
        self,
        extraction_id: UUID,
        scope_id: UUID
    ) -> bool:
        """Soft delete."""
        enforce_scope_acl(self.scope_profile, scope_id, action="delete_approved_extraction")
        
        orm = self.db.query(ApprovedExtractionORM).filter(
            and_(
                ApprovedExtractionORM.id == extraction_id,
                ApprovedExtractionORM.scope_id == scope_id,
                ApprovedExtractionORM.is_deleted == False
            )
        ).first()
        
        if not orm:
            return False
        
        orm.is_deleted = True
        orm.deleted_at = datetime.utcnow()
        orm.updated_at = datetime.utcnow()
        self.db.flush()
        
        return True
    
    def _to_model(self, orm: ApprovedExtractionORM) -> ApprovedExtraction:
        """ORM → Pydantic 변환."""
        from app.models.approved_extraction import HumanEdit
        
        human_edits = [
            HumanEdit(**edit) for edit in (orm.human_edits or [])
        ]
        
        return ApprovedExtraction(
            id=orm.id,
            document_id=orm.document_id,
            document_version=orm.document_version,
            scope_id=orm.scope_id,
            extraction_schema_id=orm.extraction_schema_id,
            extraction_schema_version=orm.extraction_schema_version,
            extraction_model=orm.extraction_model,
            extraction_latency_ms=orm.extraction_latency_ms,
            extraction_tokens=orm.extraction_tokens,
            extraction_cost_estimate=orm.extraction_cost_estimate,
            extraction_prompt_version=orm.extraction_prompt_version,
            approved_fields=orm.approved_fields,
            human_edits=human_edits,
            approved_by=orm.approved_by,
            approved_at=orm.approved_at,
            approval_comment=orm.approval_comment,
            actor_type=orm.actor_type,
            created_at=orm.created_at,
            updated_at=orm.updated_at
        )
```

### 4-4. 추출 결과 검토 API 라우터

**파일:** `/backend/app/api/v1/endpoints/extractions.py`

```python
"""
추출 결과 검토 API 엔드포인트.

- GET /api/v1/extractions/pending — pending 추출 결과 목록
- GET /api/v1/extractions/{id} — 추출 결과 상세
- POST /api/v1/extractions/{id}/approve — 승인
- POST /api/v1/extractions/{id}/modify — 수정 후 승인
- POST /api/v1/extractions/{id}/reject — 거절
- POST /api/v1/extractions/batch-approve — 일괄 승인
- POST /api/v1/extractions/batch-reject — 일괄 거절
"""

import logging
from typing import Optional
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from app.api.dependencies import get_db, get_current_user, get_scope_profile
from app.models.extraction import (
    ExtractionCandidateResponse, ExtractionCandidateListResponse,
    ExtractionStatus
)
from app.models.approved_extraction import (
    ApprovedExtractionResponse, ApproveExtractionRequest,
    ModifyExtractionRequest, RejectExtractionRequest,
    BatchApproveRequest, BatchRejectRequest, HumanEdit
)
from app.repositories.extraction_candidate_repository import ExtractionCandidateRepository
from app.repositories.approved_extraction_repository import ApprovedExtractionRepository
from app.core.scope import ScopeProfile
from app.core.audit import AuditLogger
from app.services.document.document_service import DocumentService
from app.models.approved_extraction import ApprovedExtraction
from datetime import datetime


logger = logging.getLogger(__name__)
router = APIRouter(prefix="/extractions", tags=["extractions"])


# ==================== Pending List ====================

@router.get(
    "/pending",
    response_model=ExtractionCandidateListResponse,
    summary="Pending 추출 결과 목록",
    description="인간 검토 대기 중인 추출 결과를 페이지네이션으로 조회"
)
async def list_pending_extractions(
    offset: int = 0,
    limit: int = 50,
    db: Session = Depends(get_db),
    current_user: dict = Depends(get_current_user),
    scope_profile: ScopeProfile = Depends(get_scope_profile)
):
    """
    Pending 상태의 추출 결과 목록 조회.
    
    Query Parameters:
    - offset: 페이지 오프셋 (기본값: 0)
    - limit: 페이지 크기 (기본값: 50, 최대값: 100)
    """
    if limit > 100:
        limit = 100
    
    repo = ExtractionCandidateRepository(db, scope_profile)
    items, total = await repo.list_pending(
        scope_profile.scope_id,
        offset=offset,
        limit=limit
    )
    
    return ExtractionCandidateListResponse(
        total_count=total,
        offset=offset,
        limit=limit,
        items=[
            ExtractionCandidateResponse(**item.dict())
            for item in items
        ]
    )


# ==================== Extraction Detail ====================

@router.get(
    "/{extraction_id}",
    response_model=ExtractionCandidateResponse,
    summary="추출 결과 상세 조회",
    responses={
        404: {"description": "추출 결과를 찾을 수 없음"}
    }
)
async def get_extraction_detail(
    extraction_id: UUID,
    db: Session = Depends(get_db),
    current_user: dict = Depends(get_current_user),
    scope_profile: ScopeProfile = Depends(get_scope_profile)
):
    """
    추출 결과의 상세 정보 조회.
    
    Path Parameters:
    - extraction_id: 추출 결과 ID (UUID)
    """
    repo = ExtractionCandidateRepository(db, scope_profile)
    candidate = await repo.get_by_id(extraction_id, scope_profile.scope_id)
    
    if not candidate:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Extraction not found"
        )
    
    return ExtractionCandidateResponse(**candidate.dict())


# ==================== Approve All ====================

@router.post(
    "/{extraction_id}/approve",
    response_model=ApprovedExtractionResponse,
    summary="추출 결과 전체 승인",
    status_code=status.HTTP_201_CREATED,
    responses={
        404: {"description": "추출 결과를 찾을 수 없음"},
        409: {"description": "상태 충돌 (이미 처리됨)"}
    }
)
async def approve_extraction(
    extraction_id: UUID,
    req: ApproveExtractionRequest,
    db: Session = Depends(get_db),
    current_user: dict = Depends(get_current_user),
    scope_profile: ScopeProfile = Depends(get_scope_profile),
    audit_logger: AuditLogger = Depends(get_audit_logger)
):
    """
    추출 결과의 모든 필드를 자동 추출값 그대로 승인.
    
    자동 추출값 → ApprovedExtraction 저장 → ExtractionCandidate 상태=approved
    """
    repo_candidate = ExtractionCandidateRepository(db, scope_profile)
    
    # 1. Pending 추출 결과 조회
    candidate = await repo_candidate.get_by_id(extraction_id, scope_profile.scope_id)
    if not candidate:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Extraction not found"
        )
    
    if candidate.status != ExtractionStatus.PENDING:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail=f"Cannot approve extraction with status: {candidate.status}"
        )
    
    # 2. ApprovedExtraction 생성
    approved_extraction = ApprovedExtraction(
        id=UUID('00000000-0000-0000-0000-000000000000'),
        document_id=candidate.document_id,
        document_version=candidate.document_version,
        scope_id=candidate.scope_id,
        extraction_schema_id=candidate.extraction_schema_id,
        extraction_schema_version=candidate.extraction_schema_version,
        extraction_model=candidate.extraction_model,
        extraction_latency_ms=candidate.extraction_latency_ms,
        extraction_tokens=candidate.extraction_tokens,
        extraction_cost_estimate=candidate.extraction_cost_estimate,
        extraction_prompt_version=candidate.extraction_prompt_version,
        approved_fields=candidate.extracted_fields,
        human_edits=[],  # 수정 없음
        approved_by=UUID(current_user["id"]),
        approved_at=datetime.utcnow(),
        approval_comment=req.approval_comment,
        actor_type="user"
    )
    
    # 3. DB 저장
    repo_approved = ApprovedExtractionRepository(db, scope_profile)
    saved = await repo_approved.create(approved_extraction)
    
    # 4. ExtractionCandidate 상태 업데이트
    await repo_candidate.update_status(
        extraction_id,
        scope_profile.scope_id,
        ExtractionStatus.APPROVED,
        reviewed_by=UUID(current_user["id"]),
        human_feedback=None
    )
    
    # 5. 감시 로그
    await audit_logger.log(
        action="extraction_approved",
        actor_type="user",
        user_id=current_user["id"],
        extraction_id=str(extraction_id),
        field_count=len(candidate.extracted_fields),
        comment=req.approval_comment
    )
    
    db.commit()
    
    return ApprovedExtractionResponse(**saved.dict())


# ==================== Approve with Modifications ====================

@router.post(
    "/{extraction_id}/modify",
    response_model=ApprovedExtractionResponse,
    summary="추출 결과 수정 후 승인",
    status_code=status.HTTP_201_CREATED,
    responses={
        404: {"description": "추출 결과를 찾을 수 없음"},
        409: {"description": "상태 충돌"}
    }
)
async def modify_and_approve_extraction(
    extraction_id: UUID,
    req: ModifyExtractionRequest,
    db: Session = Depends(get_db),
    current_user: dict = Depends(get_current_user),
    scope_profile: ScopeProfile = Depends(get_scope_profile),
    audit_logger: AuditLogger = Depends(get_audit_logger)
):
    """
    추출 결과의 일부 필드를 수정한 후 승인.
    
    자동 추출값 → 인간 수정 → ApprovedExtraction 저장
    """
    repo_candidate = ExtractionCandidateRepository(db, scope_profile)
    
    # 1. Pending 추출 결과 조회
    candidate = await repo_candidate.get_by_id(extraction_id, scope_profile.scope_id)
    if not candidate:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Extraction not found"
        )
    
    if candidate.status != ExtractionStatus.PENDING:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail=f"Cannot modify extraction with status: {candidate.status}"
        )
    
    # 2. 인간의 수정 적용
    approved_fields = candidate.extracted_fields.copy()
    human_edits = []
    
    for field_name, new_value in req.modifications.items():
        if field_name in approved_fields:
            before_value = approved_fields[field_name]
            approved_fields[field_name] = new_value
            
            human_edits.append(
                HumanEdit(
                    field_name=field_name,
                    before_value=before_value,
                    after_value=new_value,
                    edited_at=datetime.utcnow(),
                    edited_by=UUID(current_user["id"]),
                    reason=req.reasons.get(field_name) if req.reasons else None
                )
            )
    
    # 3. ApprovedExtraction 생성
    approved_extraction = ApprovedExtraction(
        id=UUID('00000000-0000-0000-0000-000000000000'),
        document_id=candidate.document_id,
        document_version=candidate.document_version,
        scope_id=candidate.scope_id,
        extraction_schema_id=candidate.extraction_schema_id,
        extraction_schema_version=candidate.extraction_schema_version,
        extraction_model=candidate.extraction_model,
        extraction_latency_ms=candidate.extraction_latency_ms,
        extraction_tokens=candidate.extraction_tokens,
        extraction_cost_estimate=candidate.extraction_cost_estimate,
        extraction_prompt_version=candidate.extraction_prompt_version,
        approved_fields=approved_fields,
        human_edits=human_edits,
        approved_by=UUID(current_user["id"]),
        approved_at=datetime.utcnow(),
        approval_comment=req.approval_comment,
        actor_type="user"
    )
    
    # 4. DB 저장
    repo_approved = ApprovedExtractionRepository(db, scope_profile)
    saved = await repo_approved.create(approved_extraction)
    
    # 5. ExtractionCandidate 상태 업데이트
    await repo_candidate.update_status(
        extraction_id,
        scope_profile.scope_id,
        ExtractionStatus.MODIFIED,
        reviewed_by=UUID(current_user["id"]),
        human_feedback=None
    )
    
    # 6. 감시 로그
    await audit_logger.log(
        action="extraction_modified",
        actor_type="user",
        user_id=current_user["id"],
        extraction_id=str(extraction_id),
        modified_fields=list(req.modifications.keys()),
        edit_count=len(human_edits),
        comment=req.approval_comment
    )
    
    db.commit()
    
    return ApprovedExtractionResponse(**saved.dict())


# ==================== Reject ====================

@router.post(
    "/{extraction_id}/reject",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="추출 결과 거절",
    responses={
        404: {"description": "추출 결과를 찾을 수 없음"},
        409: {"description": "상태 충돌"}
    }
)
async def reject_extraction(
    extraction_id: UUID,
    req: RejectExtractionRequest,
    db: Session = Depends(get_db),
    current_user: dict = Depends(get_current_user),
    scope_profile: ScopeProfile = Depends(get_scope_profile),
    audit_logger: AuditLogger = Depends(get_audit_logger)
):
    """
    추출 결과를 거절하고, 사용자에게 수동 입력 요청.
    
    ExtractionCandidate 상태 → rejected
    ApprovedExtraction 생성 안 함
    """
    repo_candidate = ExtractionCandidateRepository(db, scope_profile)
    
    # 1. Pending 추출 결과 조회
    candidate = await repo_candidate.get_by_id(extraction_id, scope_profile.scope_id)
    if not candidate:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Extraction not found"
        )
    
    if candidate.status != ExtractionStatus.PENDING:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail=f"Cannot reject extraction with status: {candidate.status}"
        )
    
    # 2. ExtractionCandidate 상태 업데이트
    await repo_candidate.update_status(
        extraction_id,
        scope_profile.scope_id,
        ExtractionStatus.REJECTED,
        reviewed_by=UUID(current_user["id"]),
        human_feedback=req.reason
    )
    
    # 3. 감시 로그
    await audit_logger.log(
        action="extraction_rejected",
        actor_type="user",
        user_id=current_user["id"],
        extraction_id=str(extraction_id),
        reason=req.reason
    )
    
    db.commit()


# ==================== Batch Approve ====================

@router.post(
    "/batch-approve",
    summary="다중 추출 결과 일괄 승인",
    status_code=status.HTTP_200_OK,
    responses={
        400: {"description": "유효하지 않은 요청"}
    }
)
async def batch_approve_extractions(
    req: BatchApproveRequest,
    db: Session = Depends(get_db),
    current_user: dict = Depends(get_current_user),
    scope_profile: ScopeProfile = Depends(get_scope_profile),
    audit_logger: AuditLogger = Depends(get_audit_logger)
):
    """
    여러 추출 결과를 일괄 승인.
    
    트랜잭션으로 처리: 모두 성공 또는 모두 실패
    """
    repo_candidate = ExtractionCandidateRepository(db, scope_profile)
    repo_approved = ApprovedExtractionRepository(db, scope_profile)
    
    approved_count = 0
    failed_count = 0
    errors = []
    
    try:
        for extraction_id in req.extraction_ids:
            try:
                candidate = await repo_candidate.get_by_id(
                    extraction_id, scope_profile.scope_id
                )
                
                if not candidate or candidate.status != ExtractionStatus.PENDING:
                    failed_count += 1
                    errors.append(
                        {"extraction_id": str(extraction_id), "error": "Invalid status"}
                    )
                    continue
                
                # ApprovedExtraction 생성
                approved_extraction = ApprovedExtraction(
                    id=UUID('00000000-0000-0000-0000-000000000000'),
                    document_id=candidate.document_id,
                    document_version=candidate.document_version,
                    scope_id=candidate.scope_id,
                    extraction_schema_id=candidate.extraction_schema_id,
                    extraction_schema_version=candidate.extraction_schema_version,
                    extraction_model=candidate.extraction_model,
                    extraction_latency_ms=candidate.extraction_latency_ms,
                    extraction_tokens=candidate.extraction_tokens,
                    extraction_cost_estimate=candidate.extraction_cost_estimate,
                    extraction_prompt_version=candidate.extraction_prompt_version,
                    approved_fields=candidate.extracted_fields,
                    human_edits=[],
                    approved_by=UUID(current_user["id"]),
                    approved_at=datetime.utcnow(),
                    approval_comment=req.approval_comment,
                    actor_type="user"
                )
                
                await repo_approved.create(approved_extraction)
                
                # 상태 업데이트
                await repo_candidate.update_status(
                    extraction_id,
                    scope_profile.scope_id,
                    ExtractionStatus.APPROVED,
                    reviewed_by=UUID(current_user["id"])
                )
                
                approved_count += 1
            
            except Exception as e:
                failed_count += 1
                errors.append(
                    {"extraction_id": str(extraction_id), "error": str(e)}
                )
        
        db.commit()
        
        # 감시 로그
        await audit_logger.log(
            action="extraction_batch_approved",
            actor_type="user",
            user_id=current_user["id"],
            total_count=len(req.extraction_ids),
            approved_count=approved_count,
            failed_count=failed_count
        )
    
    except Exception as e:
        db.rollback()
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=f"Batch approval failed: {str(e)}"
        )
    
    return {
        "approved_count": approved_count,
        "failed_count": failed_count,
        "errors": errors if errors else None
    }


# ==================== Batch Reject ====================

@router.post(
    "/batch-reject",
    summary="다중 추출 결과 일괄 거절",
    status_code=status.HTTP_200_OK
)
async def batch_reject_extractions(
    req: BatchRejectRequest,
    db: Session = Depends(get_db),
    current_user: dict = Depends(get_current_user),
    scope_profile: ScopeProfile = Depends(get_scope_profile),
    audit_logger: AuditLogger = Depends(get_audit_logger)
):
    """
    여러 추출 결과를 일괄 거절.
    
    모든 추출 결과를 rejected 상태로 변경.
    """
    repo_candidate = ExtractionCandidateRepository(db, scope_profile)
    
    rejected_count = 0
    failed_count = 0
    errors = []
    
    try:
        for extraction_id in req.extraction_ids:
            try:
                candidate = await repo_candidate.get_by_id(
                    extraction_id, scope_profile.scope_id
                )
                
                if not candidate or candidate.status != ExtractionStatus.PENDING:
                    failed_count += 1
                    errors.append(
                        {"extraction_id": str(extraction_id), "error": "Invalid status"}
                    )
                    continue
                
                await repo_candidate.update_status(
                    extraction_id,
                    scope_profile.scope_id,
                    ExtractionStatus.REJECTED,
                    reviewed_by=UUID(current_user["id"]),
                    human_feedback=req.reason
                )
                
                rejected_count += 1
            
            except Exception as e:
                failed_count += 1
                errors.append(
                    {"extraction_id": str(extraction_id), "error": str(e)}
                )
        
        db.commit()
        
        # 감시 로그
        await audit_logger.log(
            action="extraction_batch_rejected",
            actor_type="user",
            user_id=current_user["id"],
            total_count=len(req.extraction_ids),
            rejected_count=rejected_count,
            failed_count=failed_count,
            reason=req.reason
        )
    
    except Exception as e:
        db.rollback()
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=f"Batch rejection failed: {str(e)}"
        )
    
    return {
        "rejected_count": rejected_count,
        "failed_count": failed_count,
        "errors": errors if errors else None
    }


# ==================== Dependency ====================

def get_audit_logger() -> AuditLogger:
    """감시 로거 주입."""
    # 구체적 구현은 app.core.audit 모듈
    from app.core.audit import AuditLogger
    return AuditLogger()
```

### 4-5. Projection/Materialized View (SQL)

**파일:** `/backend/alembic/versions/*_create_approved_extraction_materialized_view.py`

```python
"""
ApprovedExtraction materialized view.

Document의 최신 approved extraction을 조회할 때 사용.
"""

def upgrade():
    # View: document_latest_approved_extraction
    op.execute("""
    CREATE MATERIALIZED VIEW document_latest_approved_extraction AS
    SELECT
        ae.document_id,
        ae.document_version,
        ae.scope_id,
        ae.extraction_schema_id,
        ae.extraction_schema_version,
        ae.approved_fields,
        ae.human_edits,
        ae.approved_by,
        ae.approved_at,
        ae.created_at,
        ROW_NUMBER() OVER (
            PARTITION BY ae.document_id
            ORDER BY ae.document_version DESC, ae.approved_at DESC
        ) AS rn
    FROM approved_extractions ae
    WHERE ae.is_deleted = FALSE
    """)
    
    op.create_index(
        'idx_mv_document_latest_extraction',
        'document_latest_approved_extraction',
        ['document_id', 'rn']
    )

def downgrade():
    op.execute("DROP MATERIALIZED VIEW IF EXISTS document_latest_approved_extraction")
```

### 4-6. 단위 및 통합 테스트

**파일:** `/backend/tests/api/v1/endpoints/test_extractions.py`

```python
"""
추출 결과 검토 API 테스트.
"""

import json
from datetime import datetime
from uuid import uuid4

import pytest
from fastapi.testclient import TestClient

from app.models.extraction import ExtractionStatus


@pytest.fixture
def client(app):
    """FastAPI TestClient."""
    return TestClient(app)


@pytest.fixture
def test_user():
    """테스트 사용자."""
    return {
        "id": str(uuid4()),
        "email": "user@example.com",
        "scope_id": str(uuid4())
    }


@pytest.fixture
def test_scope_profile(test_user):
    """테스트 Scope Profile."""
    return {
        "scope_id": test_user["scope_id"],
        "scope_type": "department",
        "readable_scopes": [test_user["scope_id"]],
        "writable_scopes": [test_user["scope_id"]]
    }


@pytest.mark.asyncio
async def test_list_pending_extractions(client, test_user, test_scope_profile, db):
    """Pending 추출 결과 목록 조회."""
    # 테스트 데이터 사전 준비
    extraction_id = uuid4()
    
    # API 호출
    response = client.get(
        "/api/v1/extractions/pending?offset=0&limit=50",
        headers={"Authorization": f"Bearer {test_user['id']}"}
    )
    
    assert response.status_code == 200
    data = response.json()
    assert "total_count" in data
    assert "items" in data
    assert data["offset"] == 0
    assert data["limit"] == 50


@pytest.mark.asyncio
async def test_get_extraction_detail_not_found(client, test_user):
    """추출 결과 조회 - 없음."""
    non_existent_id = uuid4()
    
    response = client.get(
        f"/api/v1/extractions/{non_existent_id}",
        headers={"Authorization": f"Bearer {test_user['id']}"}
    )
    
    assert response.status_code == 404
    assert "not found" in response.json()["detail"].lower()


@pytest.mark.asyncio
async def test_approve_extraction_success(client, test_user, db, mocker):
    """추출 결과 승인 - 성공."""
    extraction_id = uuid4()
    
    # Mock: ExtractionCandidate 조회
    mocker.patch(
        "app.repositories.extraction_candidate_repository.ExtractionCandidateRepository.get_by_id",
        return_value={
            "id": extraction_id,
            "document_id": uuid4(),
            "document_version": 1,
            "extraction_schema_id": "regulation",
            "extraction_schema_version": 1,
            "extracted_fields": {
                "article_number": "1.2",
                "summary": "Test summary"
            },
            "status": ExtractionStatus.PENDING,
            "extraction_model": "gpt-4o",
            "extraction_latency_ms": 100
        }
    )
    
    response = client.post(
        f"/api/v1/extractions/{extraction_id}/approve",
        json={"approval_comment": "Looks good"},
        headers={"Authorization": f"Bearer {test_user['id']}"}
    )
    
    assert response.status_code == 201
    data = response.json()
    assert "approved_fields" in data
    assert data["approved_at"] is not None


@pytest.mark.asyncio
async def test_modify_extraction_success(client, test_user, db, mocker):
    """추출 결과 수정 후 승인 - 성공."""
    extraction_id = uuid4()
    
    # Mock
    mocker.patch(
        "app.repositories.extraction_candidate_repository.ExtractionCandidateRepository.get_by_id",
        return_value={
            "id": extraction_id,
            "document_id": uuid4(),
            "document_version": 1,
            "extraction_schema_id": "regulation",
            "extraction_schema_version": 1,
            "extracted_fields": {
                "article_number": "1.2",
                "summary": "Original summary"
            },
            "status": ExtractionStatus.PENDING,
            "extraction_model": "gpt-4o",
            "extraction_latency_ms": 100
        }
    )
    
    response = client.post(
        f"/api/v1/extractions/{extraction_id}/modify",
        json={
            "modifications": {
                "summary": "Modified summary with improvements"
            },
            "reasons": {
                "summary": "Original was too vague"
            }
        },
        headers={"Authorization": f"Bearer {test_user['id']}"}
    )
    
    assert response.status_code == 201
    data = response.json()
    assert len(data["human_edits"]) == 1
    assert data["human_edits"][0]["field_name"] == "summary"
    assert data["human_edits"][0]["before_value"] == "Original summary"
    assert data["human_edits"][0]["after_value"] == "Modified summary with improvements"


@pytest.mark.asyncio
async def test_reject_extraction_success(client, test_user, db, mocker):
    """추출 결과 거절 - 성공."""
    extraction_id = uuid4()
    
    response = client.post(
        f"/api/v1/extractions/{extraction_id}/reject",
        json={
            "reason": "LLM output does not match source document"
        },
        headers={"Authorization": f"Bearer {test_user['id']}"}
    )
    
    assert response.status_code == 204


@pytest.mark.asyncio
async def test_batch_approve_success(client, test_user, db, mocker):
    """일괄 승인 - 성공."""
    extraction_ids = [uuid4() for _ in range(3)]
    
    response = client.post(
        "/api/v1/extractions/batch-approve",
        json={
            "extraction_ids": [str(eid) for eid in extraction_ids],
            "approval_comment": "Batch approved"
        },
        headers={"Authorization": f"Bearer {test_user['id']}"}
    )
    
    assert response.status_code == 200
    data = response.json()
    assert "approved_count" in data
    assert "failed_count" in data


@pytest.mark.asyncio
async def test_batch_reject_success(client, test_user, db, mocker):
    """일괄 거절 - 성공."""
    extraction_ids = [uuid4() for _ in range(2)]
    
    response = client.post(
        "/api/v1/extractions/batch-reject",
        json={
            "extraction_ids": [str(eid) for eid in extraction_ids],
            "reason": "All outputs are incorrect"
        },
        headers={"Authorization": f"Bearer {test_user['id']}"}
    )
    
    assert response.status_code == 200
    data = response.json()
    assert "rejected_count" in data


@pytest.mark.asyncio
async def test_approve_extraction_already_processed(client, test_user, db, mocker):
    """추출 결과 승인 - 이미 처리됨 (상태 충돌)."""
    extraction_id = uuid4()
    
    mocker.patch(
        "app.repositories.extraction_candidate_repository.ExtractionCandidateRepository.get_by_id",
        return_value={
            "id": extraction_id,
            "status": ExtractionStatus.APPROVED  # 이미 승인됨
        }
    )
    
    response = client.post(
        f"/api/v1/extractions/{extraction_id}/approve",
        json={},
        headers={"Authorization": f"Bearer {test_user['id']}"}
    )
    
    assert response.status_code == 409
    assert "status" in response.json()["detail"].lower()
```

## 5. 산출물

1. **ApprovedExtraction Pydantic 모델**
   - 파일: `/backend/app/models/approved_extraction.py`
   - 크기: ~8KB

2. **ApprovedExtractionORM + Alembic 마이그레이션**
   - 파일: `/backend/app/db/models/approved_extraction.py`
   - 마이그레이션: `/backend/alembic/versions/*`
   - 크기: ~6KB

3. **ApprovedExtractionRepository**
   - 파일: `/backend/app/repositories/approved_extraction_repository.py`
   - 크기: ~9KB

4. **추출 결과 검토 API 라우터**
   - 파일: `/backend/app/api/v1/endpoints/extractions.py`
   - 크기: ~30KB

5. **Materialized View**
   - 파일: `/backend/alembic/versions/*`
   - 크기: ~2KB

6. **테스트**
   - 파일: `/backend/tests/api/v1/endpoints/test_extractions.py`
   - 크기: ~12KB

## 6. 완료 기준

1. ApprovedExtraction Pydantic 모델이 정의되고, 모든 필드가 승인 결과를 정확히 캡슐화한다.
2. ApprovedExtractionORM이 정의되고, Alembic 마이그레이션이 작동한다.
3. ApprovedExtractionRepository가 Scope Profile 기반 ACL을 모든 연산에 적용한다.
4. 6가지 API 엔드포인트 (pending 목록, 상세, 승인, 수정, 거절, 일괄 작업)가 모두 구현되고 작동한다.
5. 승인 시 ApprovedExtraction artifact가 생성되고 DB에 저장된다.
6. 인간의 수정 이력이 human_edits에 정확히 기록된다.
7. 감시 로그가 모든 주요 연산 (승인, 수정, 거절)에서 actor_type="user"로 기록된다.
8. Scope Profile ACL이 모든 API 엔드포인트에서 검증된다.
9. Materialized View가 Document 조회 시 최신 approved extraction을 자동으로 JOIN한다.
10. 테스트 커버리지 90% 이상.

## 7. 작업 지침

### 지침 7-1. 독립 artifact 설계
ApprovedExtraction은 Document.metadata와 별도로 저장하여 DB 모델의 generic성을 유지하라 (S1 원칙 ②). Projection/view로만 metadata에 반영한다.

### 지침 7-2. Scope Profile ACL
모든 Repository 연산과 API 엔드포인트에서 `enforce_scope_acl()` 검증을 의무 적용하라 (S2 원칙 ⑥).

### 지침 7-3. 감시 로깅
승인/수정/거절 시 audit log를 기록하고, actor_type="user"로 표시하라 (S2 원칙 ⑤).

### 지침 7-4. 동시성 제어
동일 ExtractionCandidate를 여러 사용자가 동시에 승인하려는 경합 상황을 방지하는 메커니즘을 구현하라 (예: Optimistic lock, status 체크).

### 지침 7-5. 인간 수정 추적
human_edits는 필드별 before/after 값과 수정자, 수정 이유를 모두 포함하여 추출 데이터 계보를 보존하라.

### 지침 7-6. API 응답 스키마
모든 API 응답이 명확한 error message와 HTTP 상태 코드를 반환하도록 설계하라.

### 지침 7-7. 일괄 작업 트랜잭션
batch-approve, batch-reject는 모두 성공 또는 모두 실패(rollback)하도록 트랜잭션을 처리하라.

---

End of Task 8-5
