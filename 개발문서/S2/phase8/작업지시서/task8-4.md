# Task 8-4. 자동 추출 파이프라인 + ExtractionCandidate 모델

## 1. 작업 목적

Phase 8 FG8.2의 핵심 요소인 **자동 추출 파이프라인**을 구현한다. 문서 생성/개정 이벤트를 감지하여 `ExtractionCandidate` 도메인 모델 및 데이터베이스 스키마를 정의하고, LLM 기반 자동 추출을 비동기로 처리하는 `ExtractionPipelineService`를 구축한다. 추출 결과는 pending 상태로 사용자 승인 큐에 진입하며, Scope Profile 기반 ACL과 폐쇄망 환경 지원(S2 원칙)을 준수한다.

## 2. 작업 범위

### 포함 범위

1. **ExtractionCandidate 도메인 모델 정의 (Pydantic)**
   - 추출 캔디데이트 기본 필드: id, document_id, document_version, extraction_schema_version
   - 추출된 데이터: extracted_fields (Dict[str, Any]), confidence_scores (Dict[str, float])
   - 상태 관리: status (pending/approved/rejected/modified)
   - 추출 메타데이터: extraction_model, extraction_latency_ms, extraction_tokens, extraction_cost_estimate
   - 인간 검토 필드: reviewed_by, reviewed_at, human_feedback, approved_fields
   - 감시 정보 (S2 추가): actor_type ("agent" for LLM)

2. **SQLAlchemy ORM 모델 정의 및 마이그레이션**
   - extraction_candidates 테이블 정의
   - 복합 인덱스: (document_id, document_version), (scope_id, status), (created_at)
   - Soft delete 지원: deleted_at, is_deleted
   - Scope Profile 기반 ACL: scope_id 필드로 테넌트/부서 단위 격리 (S2 원칙 ⑥)
   - Alembic 마이그레이션 스크립트

3. **ExtractionCandidateRepository 구현**
   - CRUD 연산 (create, read, update, delete)
   - Scope Profile 필터 적용 (모든 조회/쓰기에 의무 적용)
   - 상태별 조회: find_by_status(scope_id, status)
   - 페이지네이션 지원: list_pending(scope_id, offset, limit)
   - 폐쇄망 환경: 외부 의존 없이 로컬 DB에서만 쿼리 (S2 원칙 ⑦)

4. **LLM 추출 프롬프트 템플릿**
   - Jinja2 기반 동적 프롬프트 생성
   - 템플릿 경로: `/backend/app/services/extraction/templates/extraction_prompt.jinja2`
   - 변수: schema_json, field_instructions, document_text, extraction_mode
   - 프롬프트 버전 관리 (파일의 hash 또는 VCS 커밋으로 추적)

5. **ExtractionPipelineService 구현**
   - 이벤트 기반 트리거: Document 생성/개정 시 (fastapi event 또는 webhook)
   - 처리 흐름:
     1. Document 업로드 이벤트 수신
     2. DocumentType 및 ExtractionTargetSchema 로드 (task8-3 선행)
     3. 문서 텍스트 추출 (노드 통합)
     4. Jinja2 프롬프트 렌더링
     5. Phase 1 LLM Provider 호출 (actor_type="agent" in audit log)
     6. LLM 응답 파싱 (JSON validation)
     7. ExtractionCandidate 생성 (status="pending")
     8. 감시 정보 기록 (model, latency, tokens, cost)
   - 비동기 실행: asyncio.create_task 또는 Celery worker

6. **에러 처리 및 복원력**
   - LLM 호출 타임아웃 (30초): exponential backoff 재시도 (3회)
   - 유효하지 않은 JSON 응답: 폴백으로 로컬 모델 호출 (s2-7: closed-network)
   - 스키마 미스매치: 에러 로그 기록 후 manual review 큐에 진입
   - 네트워크 오류: DLQ (Dead Letter Queue) 저장

7. **폐쇄망 환경 지원 (S2 원칙 ⑦)**
   - 외부 LLM 환경변수: LLM_PROVIDER (openai | anthropic | local)
   - LLM_PROVIDER=local 일 때: 로컬 모델(ollama, llama.cpp 등) 호출
   - API 명령어: `/api/v1/extractions/retry-with-local-model/{extraction_id}`

8. **단위 테스트 및 통합 테스트**
   - Mock LLM response으로 ExtractionCandidate 생성 검증
   - JSON parsing 에러 케이스
   - Scope Profile 필터링 정확성
   - 비동기 태스크 완료 확인
   - 폐쇄망 fallback 동작

9. **감사 로깅 (S2 원칙 ⑤)**
   - 추출 시작: action="extraction_started", actor_type="agent", document_id, schema_version
   - 추출 완료: action="extraction_completed", actor_type="agent", latency_ms, model_name
   - 추출 실패: action="extraction_failed", actor_type="agent", error_reason

### 제외 범위

- ExtractionTargetSchema CRUD API (Task 8-3)
- 추출 결과 검토 API 및 승인 로직 (Task 8-5)
- UI 컴포넌트 (Task 8-6)
- 추출 재실행 배치 (Task 8-7)
- 평가 로직 (FG8.3)

## 3. 선행 조건

- Phase 0~7 완료
- Phase 1 FG1.1: LLM Provider 추상화 완료 (llm_provider_service.py 또는 유사)
- DocumentType 및 metadata schema 정의 완료 (S1 기반)
- Task 8-3 (ExtractionTargetSchema): 완료 예정 (Task 8-4와 병렬 개발 가능)
- PostgreSQL + SQLAlchemy + Alembic 설정 완료
- Pydantic v2 설치
- Python asyncio 또는 Celery 설정
- Jinja2 템플릿 엔진 설치

## 4. 주요 작업 항목

### 4-1. ExtractionCandidate Pydantic 모델

**파일:** `/backend/app/models/extraction.py`

```python
"""
ExtractionCandidate 도메인 모델 (Pydantic 기반).

문서 업로드 후 LLM이 자동으로 추출한 결과를 캡슐화한다.
상태: pending (인간 검토 대기) → approved/rejected/modified
"""

from __future__ import annotations

from datetime import datetime
from enum import Enum
from typing import Any, Dict, Optional, List
from uuid import UUID

from pydantic import BaseModel, Field, validator


# ==================== Enums ====================

class ExtractionStatus(str, Enum):
    """추출 캔디데이트 상태."""
    PENDING = "pending"          # 인간 검토 대기
    APPROVED = "approved"        # 모든 필드 자동 추출값 승인
    REJECTED = "rejected"        # 전체 거절 (수동 입력 필요)
    MODIFIED = "modified"        # 일부 필드 수정 후 승인


class ExtractionMode(str, Enum):
    """추출 모드."""
    DETERMINISTIC = "deterministic"  # temperature=0 (재현성 중시)
    PROBABILISTIC = "probabilistic"  # temperature=0.7 (다양성 중시)


# ==================== Domain Models ====================

class ExtractionConfidenceScore(BaseModel):
    """필드별 신뢰도 점수."""
    field_name: str
    confidence: float = Field(
        ge=0.0, le=1.0,
        description="0.0~1.0 범위의 신뢰도"
    )
    reason: Optional[str] = None  # 신뢰도 산정 이유


class ExtractedFieldValue(BaseModel):
    """추출된 필드 값."""
    field_name: str
    extracted_value: Any
    original_value: Optional[Any] = None  # 수정 전 원래값 (modified 상태에서만 사용)
    type_mismatch: bool = False  # 타입 불일치 플래그


class HumanEditRecord(BaseModel):
    """인간이 수정한 필드의 기록."""
    field_name: str
    before_value: Any
    after_value: Any
    edited_at: datetime
    edited_by: UUID


class ExtractionCandidate(BaseModel):
    """
    추출 캔디데이트 (Pydantic 도메인 모델).
    
    문서 업로드 후 LLM이 생성한 자동 추출 결과.
    인간의 승인/거절/수정을 거친 후 최종 저장됨 (Task 8-5).
    """
    id: UUID = Field(default_factory=lambda: UUID('00000000-0000-0000-0000-000000000000'))
    
    # ==================== 문서 정보 ====================
    document_id: UUID
    document_version: int
    scope_id: UUID  # S2: Scope Profile 기반 ACL
    
    # ==================== 스키마 정보 ====================
    extraction_schema_id: str  # ExtractionTargetSchema의 ID (doc_type)
    extraction_schema_version: int  # 스키마 버전
    
    # ==================== 추출 데이터 ====================
    extracted_fields: Dict[str, Any] = Field(
        default_factory=dict,
        description="스키마에 따라 추출된 필드 {field_name: value}"
    )
    confidence_scores: List[ExtractionConfidenceScore] = Field(
        default_factory=list,
        description="각 필드별 신뢰도 점수"
    )
    
    # ==================== 추출 메타데이터 ====================
    extraction_model: str  # 사용한 모델명 (gpt-4o, claude-3.5-sonnet, etc)
    extraction_mode: ExtractionMode = ExtractionMode.DETERMINISTIC
    extraction_latency_ms: int  # LLM 호출 지연 시간
    extraction_tokens: Optional[Dict[str, int]] = None  # {"input": 150, "output": 200}
    extraction_cost_estimate: Optional[float] = None  # USD 추정 비용
    
    # ==================== 상태 관리 ====================
    status: ExtractionStatus = ExtractionStatus.PENDING
    
    # ==================== 인간 검토 정보 ====================
    reviewed_by: Optional[UUID] = None  # 검토자의 user_id
    reviewed_at: Optional[datetime] = None
    human_feedback: Optional[str] = None  # 거절 이유 등
    
    # approved_fields는 modified 상태에서만 사용:
    # { "field1": {before_value, after_value}, "field2": ... }
    human_edits: List[HumanEditRecord] = Field(
        default_factory=list,
        description="인간이 수정한 필드 기록"
    )
    
    # ==================== 타임스탐프 ====================
    created_at: datetime
    updated_at: datetime
    
    # ==================== 감시 정보 (S2 원칙 ⑤) ====================
    actor_type: str = "agent"  # "user" 또는 "agent"
    extraction_prompt_version: Optional[str] = None  # 프롬프트 템플릿 버전
    document_content_hash: Optional[str] = None  # 원문 무결성 검증
    
    class Config:
        use_enum_values = False
        json_encoders = {
            datetime: lambda v: v.isoformat(),
            UUID: lambda v: str(v),
        }
    
    @validator('confidence_scores')
    def validate_confidence_scores(cls, v, values):
        """신뢰도 점수가 extracted_fields의 키와 매칭되는지 검증."""
        if 'extracted_fields' in values:
            field_names = set(values['extracted_fields'].keys())
            score_names = set(cs.field_name for cs in v)
            if not score_names.issubset(field_names):
                raise ValueError(
                    f"Confidence scores must be subset of extracted_fields. "
                    f"Extra: {score_names - field_names}"
                )
        return v
    
    @validator('status')
    def validate_status_consistency(cls, v, values):
        """상태와 필드 일관성 검증."""
        if v == ExtractionStatus.MODIFIED and not values.get('human_edits'):
            raise ValueError("modified status requires human_edits")
        if v == ExtractionStatus.REJECTED and not values.get('human_feedback'):
            raise ValueError("rejected status should have human_feedback")
        return v


# ==================== Request/Response DTOs ====================

class ExtractionCandidateCreateRequest(BaseModel):
    """내부용: 자동 추출 파이프라인에서 ExtractionCandidate 생성 요청."""
    document_id: UUID
    document_version: int
    scope_id: UUID
    extraction_schema_id: str
    extraction_schema_version: int
    extracted_fields: Dict[str, Any]
    confidence_scores: List[ExtractionConfidenceScore] = []
    extraction_model: str
    extraction_mode: ExtractionMode = ExtractionMode.DETERMINISTIC
    extraction_latency_ms: int
    extraction_tokens: Optional[Dict[str, int]] = None
    extraction_cost_estimate: Optional[float] = None
    extraction_prompt_version: Optional[str] = None
    document_content_hash: Optional[str] = None


class ExtractionCandidateResponse(BaseModel):
    """조회 응답."""
    id: UUID
    document_id: UUID
    document_version: int
    extraction_schema_id: str
    extraction_schema_version: int
    extracted_fields: Dict[str, Any]
    confidence_scores: List[ExtractionConfidenceScore]
    status: ExtractionStatus
    extraction_model: str
    extraction_latency_ms: int
    reviewed_by: Optional[UUID]
    reviewed_at: Optional[datetime]
    human_feedback: Optional[str]
    human_edits: List[HumanEditRecord]
    created_at: datetime
    updated_at: datetime


class ExtractionCandidateListResponse(BaseModel):
    """페이지네이션 응답."""
    total_count: int
    offset: int
    limit: int
    items: List[ExtractionCandidateResponse]
```

### 4-2. SQLAlchemy ORM 모델 및 마이그레이션

**파일:** `/backend/app/db/models/extraction.py`

```python
"""
ExtractionCandidate SQLAlchemy ORM 모델.

extraction_candidates 테이블 매핑.
"""

from datetime import datetime
from typing import Any, Dict, Optional
from uuid import UUID

from sqlalchemy import (
    Boolean, Column, DateTime, Enum, Float, ForeignKey, 
    Integer, JSON, String, Text, UniqueConstraint, Index, TIMESTAMP
)
from sqlalchemy.dialects.postgresql import UUID as PG_UUID
from sqlalchemy.orm import relationship

from app.db.base import Base
from app.models.extraction import ExtractionStatus, ExtractionMode


class ExtractionCandidateORM(Base):
    """
    추출 캔디데이트 ORM 모델.
    
    테이블: extraction_candidates
    복합 키: (document_id, document_version, extraction_schema_version)
    """
    
    __tablename__ = "extraction_candidates"
    
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
        index=True
    )
    document_version = Column(
        Integer,
        nullable=False,
        default=1
    )
    
    # ==================== Scope & ACL (S2 원칙 ⑥) ====================
    scope_id = Column(
        PG_UUID(as_uuid=True),
        nullable=False,
        index=True,
        comment="Scope Profile ID (테넌트/부서 단위 격리)"
    )
    
    # ==================== Schema Reference ====================
    extraction_schema_id = Column(
        String(255),
        nullable=False,
        comment="DocumentType ID (e.g., 'regulation')"
    )
    extraction_schema_version = Column(
        Integer,
        nullable=False,
        default=1
    )
    
    # ==================== Extracted Data ====================
    extracted_fields = Column(
        JSON,
        nullable=False,
        default={},
        comment="추출된 필드 {field_name: value}"
    )
    confidence_scores = Column(
        JSON,
        nullable=False,
        default=[],
        comment="[{field_name, confidence, reason}]"
    )
    
    # ==================== Extraction Metadata ====================
    extraction_model = Column(
        String(255),
        nullable=False,
        comment="Model name (gpt-4o, claude-3.5-sonnet, etc)"
    )
    extraction_mode = Column(
        Enum(ExtractionMode),
        nullable=False,
        default=ExtractionMode.DETERMINISTIC,
        comment="deterministic or probabilistic"
    )
    extraction_latency_ms = Column(
        Integer,
        nullable=False,
        comment="LLM response latency in milliseconds"
    )
    extraction_tokens = Column(
        JSON,
        nullable=True,
        comment="{input: int, output: int}"
    )
    extraction_cost_estimate = Column(
        Float,
        nullable=True,
        comment="Estimated cost in USD"
    )
    extraction_prompt_version = Column(
        String(64),
        nullable=True,
        comment="Prompt template version (SHA256 hash or VCS)"
    )
    document_content_hash = Column(
        String(64),
        nullable=True,
        comment="Document content hash for integrity check"
    )
    
    # ==================== Status ====================
    status = Column(
        Enum(ExtractionStatus),
        nullable=False,
        default=ExtractionStatus.PENDING,
        index=True
    )
    
    # ==================== Human Review ====================
    reviewed_by = Column(
        PG_UUID(as_uuid=True),
        nullable=True,
        index=True
    )
    reviewed_at = Column(
        DateTime(timezone=True),
        nullable=True
    )
    human_feedback = Column(
        Text,
        nullable=True,
        comment="Rejection reason or review notes"
    )
    human_edits = Column(
        JSON,
        nullable=False,
        default=[],
        comment="[{field_name, before_value, after_value, edited_at, edited_by}]"
    )
    
    # ==================== Timestamps ====================
    created_at = Column(
        DateTime(timezone=True),
        nullable=False,
        default=datetime.utcnow,
        index=True
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
    
    # ==================== Audit (S2 원칙 ⑤) ====================
    actor_type = Column(
        String(32),
        nullable=False,
        default="agent",
        comment="'user' or 'agent' (LLM)"
    )
    
    # ==================== Constraints ====================
    __table_args__ = (
        # 복합 인덱스
        Index('idx_extraction_pending', 'scope_id', 'status', 'created_at'),
        Index('idx_extraction_document', 'document_id', 'document_version'),
        Index('idx_extraction_schema', 'extraction_schema_id', 'extraction_schema_version'),
        
        # 복합 유니크 제약 (soft delete 고려)
        UniqueConstraint(
            'document_id', 'document_version', 'extraction_schema_version',
            name='uq_extraction_candidate_version',
            postgresql_where='is_deleted = false'
        ),
    )


# ==================== Alembic Migration ====================

"""
마이그레이션 파일: /backend/alembic/versions/<timestamp>_create_extraction_candidates_table.py

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

def upgrade():
    op.create_table(
        'extraction_candidates',
        sa.Column('id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('document_id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('document_version', sa.Integer(), nullable=False),
        sa.Column('scope_id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('extraction_schema_id', sa.String(255), nullable=False),
        sa.Column('extraction_schema_version', sa.Integer(), nullable=False),
        sa.Column('extracted_fields', postgresql.JSON(), nullable=False),
        sa.Column('confidence_scores', postgresql.JSON(), nullable=False),
        sa.Column('extraction_model', sa.String(255), nullable=False),
        sa.Column('extraction_mode', sa.Enum('DETERMINISTIC', 'PROBABILISTIC'), nullable=False),
        sa.Column('extraction_latency_ms', sa.Integer(), nullable=False),
        sa.Column('extraction_tokens', postgresql.JSON(), nullable=True),
        sa.Column('extraction_cost_estimate', sa.Float(), nullable=True),
        sa.Column('extraction_prompt_version', sa.String(64), nullable=True),
        sa.Column('document_content_hash', sa.String(64), nullable=True),
        sa.Column('status', sa.Enum('PENDING', 'APPROVED', 'REJECTED', 'MODIFIED'), nullable=False),
        sa.Column('reviewed_by', postgresql.UUID(as_uuid=True), nullable=True),
        sa.Column('reviewed_at', sa.DateTime(timezone=True), nullable=True),
        sa.Column('human_feedback', sa.Text(), nullable=True),
        sa.Column('human_edits', postgresql.JSON(), nullable=False),
        sa.Column('created_at', sa.DateTime(timezone=True), nullable=False),
        sa.Column('updated_at', sa.DateTime(timezone=True), nullable=False),
        sa.Column('deleted_at', sa.DateTime(timezone=True), nullable=True),
        sa.Column('is_deleted', sa.Boolean(), nullable=False),
        sa.Column('actor_type', sa.String(32), nullable=False),
        sa.PrimaryKeyConstraint('id')
    )
    op.create_index('idx_extraction_pending', 'extraction_candidates', 
                    ['scope_id', 'status', 'created_at'])
    op.create_index('idx_extraction_document', 'extraction_candidates',
                    ['document_id', 'document_version'])
    op.create_index('idx_extraction_schema', 'extraction_candidates',
                    ['extraction_schema_id', 'extraction_schema_version'])
    op.create_unique_constraint('uq_extraction_candidate_version',
                               'extraction_candidates',
                               ['document_id', 'document_version', 'extraction_schema_version'],
                               postgresql_where='is_deleted = false')

def downgrade():
    op.drop_table('extraction_candidates')
"""
```

### 4-3. ExtractionCandidateRepository

**파일:** `/backend/app/repositories/extraction_candidate_repository.py`

```python
"""
ExtractionCandidateRepository.

Scope Profile 기반 ACL을 모든 연산에 적용 (S2 원칙 ⑥).
"""

from datetime import datetime
from typing import Optional, List
from uuid import UUID

from sqlalchemy.orm import Session
from sqlalchemy import and_, or_

from app.db.models.extraction import ExtractionCandidateORM
from app.models.extraction import (
    ExtractionCandidate, ExtractionStatus, ExtractionCandidateCreateRequest
)
from app.core.scope import ScopeProfile, enforce_scope_acl


class ExtractionCandidateRepository:
    """ExtractionCandidate 저장소."""
    
    def __init__(self, db: Session, scope_profile: ScopeProfile):
        self.db = db
        self.scope_profile = scope_profile
    
    async def create(
        self,
        req: ExtractionCandidateCreateRequest,
        candidate_id: UUID
    ) -> ExtractionCandidate:
        """
        새 ExtractionCandidate 생성.
        
        Scope Profile 확인 필수.
        """
        # S2 원칙 ⑥: Scope 유효성 검증
        enforce_scope_acl(self.scope_profile, req.scope_id, action="create_extraction")
        
        orm = ExtractionCandidateORM(
            id=candidate_id,
            document_id=req.document_id,
            document_version=req.document_version,
            scope_id=req.scope_id,
            extraction_schema_id=req.extraction_schema_id,
            extraction_schema_version=req.extraction_schema_version,
            extracted_fields=req.extracted_fields,
            confidence_scores=[cs.dict() for cs in req.confidence_scores],
            extraction_model=req.extraction_model,
            extraction_mode=req.extraction_mode,
            extraction_latency_ms=req.extraction_latency_ms,
            extraction_tokens=req.extraction_tokens,
            extraction_cost_estimate=req.extraction_cost_estimate,
            extraction_prompt_version=req.extraction_prompt_version,
            document_content_hash=req.document_content_hash,
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow(),
            actor_type="agent"  # S2 원칙 ⑤: 자동 추출은 agent
        )
        
        self.db.add(orm)
        self.db.flush()
        
        return self._to_model(orm)
    
    async def get_by_id(self, candidate_id: UUID, scope_id: UUID) -> Optional[ExtractionCandidate]:
        """ID로 조회 (Scope 필터 적용)."""
        enforce_scope_acl(self.scope_profile, scope_id, action="read_extraction")
        
        orm = self.db.query(ExtractionCandidateORM).filter(
            and_(
                ExtractionCandidateORM.id == candidate_id,
                ExtractionCandidateORM.scope_id == scope_id,
                ExtractionCandidateORM.is_deleted == False
            )
        ).first()
        
        return self._to_model(orm) if orm else None
    
    async def list_pending(
        self,
        scope_id: UUID,
        offset: int = 0,
        limit: int = 50
    ) -> tuple[List[ExtractionCandidate], int]:
        """
        Pending 상태의 추출 결과 목록 조회 (페이지네이션).
        
        반환: (항목 리스트, 총 개수)
        """
        enforce_scope_acl(self.scope_profile, scope_id, action="read_extraction")
        
        query = self.db.query(ExtractionCandidateORM).filter(
            and_(
                ExtractionCandidateORM.scope_id == scope_id,
                ExtractionCandidateORM.status == ExtractionStatus.PENDING,
                ExtractionCandidateORM.is_deleted == False
            )
        ).order_by(ExtractionCandidateORM.created_at.desc())
        
        total = query.count()
        items = query.offset(offset).limit(limit).all()
        
        return [self._to_model(orm) for orm in items], total
    
    async def list_by_status(
        self,
        scope_id: UUID,
        status: ExtractionStatus,
        offset: int = 0,
        limit: int = 50
    ) -> tuple[List[ExtractionCandidate], int]:
        """상태별 조회."""
        enforce_scope_acl(self.scope_profile, scope_id, action="read_extraction")
        
        query = self.db.query(ExtractionCandidateORM).filter(
            and_(
                ExtractionCandidateORM.scope_id == scope_id,
                ExtractionCandidateORM.status == status,
                ExtractionCandidateORM.is_deleted == False
            )
        ).order_by(ExtractionCandidateORM.created_at.desc())
        
        total = query.count()
        items = query.offset(offset).limit(limit).all()
        
        return [self._to_model(orm) for orm in items], total
    
    async def list_by_document(
        self,
        document_id: UUID,
        scope_id: UUID,
        offset: int = 0,
        limit: int = 50
    ) -> tuple[List[ExtractionCandidate], int]:
        """문서별 추출 결과 조회."""
        enforce_scope_acl(self.scope_profile, scope_id, action="read_extraction")
        
        query = self.db.query(ExtractionCandidateORM).filter(
            and_(
                ExtractionCandidateORM.document_id == document_id,
                ExtractionCandidateORM.scope_id == scope_id,
                ExtractionCandidateORM.is_deleted == False
            )
        ).order_by(ExtractionCandidateORM.created_at.desc())
        
        total = query.count()
        items = query.offset(offset).limit(limit).all()
        
        return [self._to_model(orm) for orm in items], total
    
    async def update_status(
        self,
        candidate_id: UUID,
        scope_id: UUID,
        new_status: ExtractionStatus,
        reviewed_by: Optional[UUID] = None,
        human_feedback: Optional[str] = None
    ) -> Optional[ExtractionCandidate]:
        """상태 업데이트."""
        enforce_scope_acl(self.scope_profile, scope_id, action="update_extraction")
        
        orm = self.db.query(ExtractionCandidateORM).filter(
            and_(
                ExtractionCandidateORM.id == candidate_id,
                ExtractionCandidateORM.scope_id == scope_id,
                ExtractionCandidateORM.is_deleted == False
            )
        ).first()
        
        if not orm:
            return None
        
        orm.status = new_status
        orm.reviewed_at = datetime.utcnow()
        orm.reviewed_by = reviewed_by
        orm.human_feedback = human_feedback
        orm.updated_at = datetime.utcnow()
        
        self.db.flush()
        return self._to_model(orm)
    
    async def soft_delete(
        self,
        candidate_id: UUID,
        scope_id: UUID
    ) -> bool:
        """Soft delete."""
        enforce_scope_acl(self.scope_profile, scope_id, action="delete_extraction")
        
        orm = self.db.query(ExtractionCandidateORM).filter(
            and_(
                ExtractionCandidateORM.id == candidate_id,
                ExtractionCandidateORM.scope_id == scope_id,
                ExtractionCandidateORM.is_deleted == False
            )
        ).first()
        
        if not orm:
            return False
        
        orm.is_deleted = True
        orm.deleted_at = datetime.utcnow()
        orm.updated_at = datetime.utcnow()
        self.db.flush()
        
        return True
    
    def _to_model(self, orm: ExtractionCandidateORM) -> ExtractionCandidate:
        """ORM → Pydantic 변환."""
        return ExtractionCandidate(
            id=orm.id,
            document_id=orm.document_id,
            document_version=orm.document_version,
            scope_id=orm.scope_id,
            extraction_schema_id=orm.extraction_schema_id,
            extraction_schema_version=orm.extraction_schema_version,
            extracted_fields=orm.extracted_fields,
            confidence_scores=orm.confidence_scores,
            extraction_model=orm.extraction_model,
            extraction_mode=orm.extraction_mode,
            extraction_latency_ms=orm.extraction_latency_ms,
            extraction_tokens=orm.extraction_tokens,
            extraction_cost_estimate=orm.extraction_cost_estimate,
            extraction_prompt_version=orm.extraction_prompt_version,
            document_content_hash=orm.document_content_hash,
            status=orm.status,
            reviewed_by=orm.reviewed_by,
            reviewed_at=orm.reviewed_at,
            human_feedback=orm.human_feedback,
            human_edits=orm.human_edits,
            created_at=orm.created_at,
            updated_at=orm.updated_at,
            actor_type=orm.actor_type
        )
```

### 4-4. LLM 추출 프롬프트 템플릿

**파일:** `/backend/app/services/extraction/templates/extraction_prompt.jinja2`

```jinja2
당신은 정보 추출 AI입니다.

다음 스키마에 따라 문서에서 정보를 추출하세요:

## 추출 스키마

```json
{{ schema_json }}
```

## 추출 지침

{% for instruction in field_instructions %}
- **{{ instruction.field_name }}** ({{ instruction.field_type }})
  - 필수: {{ instruction.required }}
  - 지침: {{ instruction.instruction }}
  {% if instruction.examples %}
  - 예시: {{ instruction.examples | join(', ') }}
  {% endif %}
{% endfor %}

## 일반 지침

1. **원문 기반만**: 각 필드는 원문에서만 추출하세요. 추론하거나 외부 정보를 추가하지 마세요.
2. **필수 필드**: 필수로 표시된 필드가 원문에 없으면 JSON null 값으로 반환하세요.
3. **타입 변환**: 날짜는 지정된 형식으로, 숫자는 숫자로, 배열은 JSON 배열로 반환하세요.
4. **타입 불일치**: 추출한 값이 스키마의 타입과 맞지 않으면, 유효한 타입으로 변환 또는 null을 반환하세요.
5. **중복 제거**: 배열 필드에서 중복값이 있으면 제거하세요.

## 추출 모드

- **Mode**: {{ extraction_mode }}
- Temperature: {{ temperature }}

## 원문

```
{{ document_text }}
```

## 결과

다음 JSON 포맷으로 추출 결과를 반환하세요. 추가 설명은 포함하지 마세요.

```json
{
  {%- for field_name in field_names %}
  "{{ field_name }}": <extracted_value>,
  {%- endfor %}
}
```
```

### 4-5. ExtractionPipelineService

**파일:** `/backend/app/services/extraction/extraction_pipeline_service.py`

```python
"""
ExtractionPipelineService.

문서 업로드/개정 시 트리거되는 자동 추출 파이프라인.
비동기로 실행되며, Scope Profile ACL과 폐쇄망 환경 지원(S2).
"""

import asyncio
import hashlib
import json
import logging
import os
from datetime import datetime
from typing import Optional, Dict, Any
from uuid import UUID, uuid4

from jinja2 import Environment, FileSystemLoader, Template

from app.core.scope import ScopeProfile
from app.models.extraction import (
    ExtractionCandidate, ExtractionStatus, ExtractionMode,
    ExtractionCandidateCreateRequest, ExtractionConfidenceScore
)
from app.repositories.extraction_candidate_repository import ExtractionCandidateRepository
from app.services.llm.llm_provider_service import LLMProviderService  # Phase 1
from app.services.document.document_service import DocumentService
from app.services.extraction_schema.extraction_target_schema_service import ExtractionTargetSchemaService
from app.db.database import SessionLocal
from app.core.audit import AuditLogger


logger = logging.getLogger(__name__)


class ExtractionPipelineService:
    """자동 추출 파이프라인."""
    
    def __init__(
        self,
        db_session,
        llm_provider: LLMProviderService,
        document_service: DocumentService,
        schema_service: ExtractionTargetSchemaService,
        audit_logger: AuditLogger,
        template_dir: str = "/backend/app/services/extraction/templates"
    ):
        self.db = db_session
        self.llm_provider = llm_provider
        self.document_service = document_service
        self.schema_service = schema_service
        self.audit_logger = audit_logger
        
        # Jinja2 템플릿 로드
        self.jinja_env = Environment(
            loader=FileSystemLoader(template_dir),
            trim_blocks=True,
            lstrip_blocks=True
        )
        
        # 환경 변수
        self.llm_provider_type = os.getenv("LLM_PROVIDER", "openai")  # openai|anthropic|local
        self.extraction_timeout_sec = int(os.getenv("EXTRACTION_TIMEOUT_SEC", "30"))
        self.max_retries = int(os.getenv("EXTRACTION_MAX_RETRIES", "3"))
    
    async def trigger_on_document_create(
        self,
        document_id: UUID,
        document_version: int,
        scope_id: UUID,
        doc_type: str
    ) -> Optional[ExtractionCandidate]:
        """
        문서 생성 이벤트 핸들러.
        
        자동 추출 파이프라인을 비동기로 시작.
        """
        try:
            # S2 원칙 ⑤: 감시 로그 시작
            await self.audit_logger.log(
                action="extraction_started",
                actor_type="agent",
                document_id=str(document_id),
                doc_type=doc_type,
                extraction_mode="automatic"
            )
            
            # 비동기 태스크로 실행
            task = asyncio.create_task(
                self._execute_extraction_pipeline(
                    document_id, document_version, scope_id, doc_type
                )
            )
            
            # 태스크 반환 (caller는 완료 대기 안 함)
            return None
        
        except Exception as e:
            logger.error(f"Failed to trigger extraction: {e}")
            await self.audit_logger.log(
                action="extraction_trigger_failed",
                actor_type="agent",
                document_id=str(document_id),
                error=str(e)
            )
            raise
    
    async def _execute_extraction_pipeline(
        self,
        document_id: UUID,
        document_version: int,
        scope_id: UUID,
        doc_type: str
    ) -> Optional[ExtractionCandidate]:
        """
        실제 추출 파이프라인 실행 (비동기).
        
        처리 흐름:
        1. 문서 로드
        2. 스키마 로드
        3. 프롬프트 렌더링
        4. LLM 호출 (+ 폐쇄망 fallback)
        5. 결과 저장
        6. 감시 로그
        """
        start_time = datetime.utcnow()
        latency_ms = 0
        tokens_used = None
        cost_estimate = None
        
        try:
            # Step 1: 문서 로드
            logger.info(f"Loading document {document_id} v{document_version}")
            document = await self.document_service.get_document(
                document_id, scope_id
            )
            if not document:
                raise ValueError(f"Document {document_id} not found")
            
            # 원문 해시 계산 (무결성 검증용)
            document_text = document.content_text or ""
            document_content_hash = hashlib.sha256(
                document_text.encode('utf-8')
            ).hexdigest()
            
            # Step 2: ExtractionTargetSchema 로드
            logger.info(f"Loading schema for doc_type: {doc_type}")
            schema = await self.schema_service.get_schema_by_doc_type(
                doc_type, scope_id
            )
            if not schema:
                raise ValueError(f"No extraction schema for doc_type: {doc_type}")
            
            # Step 3: 프롬프트 렌더링
            logger.info("Rendering extraction prompt")
            prompt = self._render_extraction_prompt(
                schema=schema,
                document_text=document_text,
                extraction_mode=ExtractionMode.DETERMINISTIC,
                temperature=0.0
            )
            
            # Step 4: LLM 호출 (재시도 + fallback)
            logger.info("Calling LLM for extraction")
            llm_response, usage = await self._call_llm_with_retry(
                prompt=prompt,
                model_hint=self.llm_provider_type
            )
            
            # Step 5: 응답 파싱
            logger.info("Parsing LLM response")
            extracted_fields = self._parse_extraction_response(llm_response)
            
            # 스키마 유효성 검증
            self._validate_extracted_fields(schema, extracted_fields)
            
            # 신뢰도 점수 계산
            confidence_scores = self._compute_confidence_scores(
                schema, extracted_fields
            )
            
            # 지연 시간 계산
            latency_ms = int((datetime.utcnow() - start_time).total_seconds() * 1000)
            tokens_used = usage.get("tokens", {}) if usage else None
            cost_estimate = self._estimate_cost(tokens_used)
            
            # Step 6: ExtractionCandidate 생성
            logger.info("Creating ExtractionCandidate")
            candidate_req = ExtractionCandidateCreateRequest(
                document_id=document_id,
                document_version=document_version,
                scope_id=scope_id,
                extraction_schema_id=doc_type,
                extraction_schema_version=schema.version,
                extracted_fields=extracted_fields,
                confidence_scores=confidence_scores,
                extraction_model=self.llm_provider_type,
                extraction_mode=ExtractionMode.DETERMINISTIC,
                extraction_latency_ms=latency_ms,
                extraction_tokens=tokens_used,
                extraction_cost_estimate=cost_estimate,
                extraction_prompt_version=self._get_prompt_version(),
                document_content_hash=document_content_hash
            )
            
            # DB 저장
            db = SessionLocal()
            try:
                repo = ExtractionCandidateRepository(db, None)  # repo 내에서 ACL 체크
                candidate = await repo.create(
                    candidate_req,
                    candidate_id=uuid4()
                )
                db.commit()
            finally:
                db.close()
            
            # Step 7: 감시 로그
            await self.audit_logger.log(
                action="extraction_completed",
                actor_type="agent",
                document_id=str(document_id),
                extraction_schema_id=doc_type,
                latency_ms=latency_ms,
                tokens=tokens_used,
                cost=cost_estimate,
                status="success"
            )
            
            logger.info(f"Extraction completed: {candidate.id}")
            return candidate
        
        except Exception as e:
            logger.error(f"Extraction pipeline failed: {e}", exc_info=True)
            
            # 실패 로그
            await self.audit_logger.log(
                action="extraction_failed",
                actor_type="agent",
                document_id=str(document_id),
                error_reason=str(e),
                latency_ms=int((datetime.utcnow() - start_time).total_seconds() * 1000)
            )
            
            raise
    
    def _render_extraction_prompt(
        self,
        schema,  # ExtractionTargetSchema
        document_text: str,
        extraction_mode: ExtractionMode,
        temperature: float
    ) -> str:
        """Jinja2로 추출 프롬프트 렌더링."""
        template = self.jinja_env.get_template("extraction_prompt.jinja2")
        
        # 스키마 JSON
        schema_json = json.dumps(
            schema.fields_definition,
            indent=2,
            ensure_ascii=False
        )
        
        # 필드별 지침
        field_instructions = [
            {
                "field_name": name,
                "field_type": field.get("field_type", "string"),
                "required": field.get("required", False),
                "instruction": field.get("instruction", ""),
                "examples": field.get("examples", [])
            }
            for name, field in schema.fields_definition.items()
        ]
        
        # 필드명 목록
        field_names = list(schema.fields_definition.keys())
        
        return template.render(
            schema_json=schema_json,
            field_instructions=field_instructions,
            field_names=field_names,
            document_text=document_text,
            extraction_mode=extraction_mode.value,
            temperature=temperature
        )
    
    async def _call_llm_with_retry(
        self,
        prompt: str,
        model_hint: str
    ) -> tuple[str, Optional[Dict[str, Any]]]:
        """
        LLM 호출 (재시도 + 폐쇄망 fallback).
        
        반환: (응답 텍스트, 사용량 정보)
        """
        last_error = None
        
        for attempt in range(self.max_retries):
            try:
                logger.info(f"LLM call attempt {attempt + 1}/{self.max_retries}")
                
                # Phase 1 LLM Provider 호출
                response = await asyncio.wait_for(
                    self.llm_provider.generate(
                        prompt=prompt,
                        model_hint=model_hint,
                        temperature=0.0,
                        max_tokens=2000
                    ),
                    timeout=self.extraction_timeout_sec
                )
                
                return response.get("text", ""), response.get("usage", {})
            
            except asyncio.TimeoutError as e:
                last_error = e
                logger.warning(f"LLM timeout on attempt {attempt + 1}")
                if attempt < self.max_retries - 1:
                    await asyncio.sleep(2 ** attempt)  # exponential backoff
            
            except Exception as e:
                last_error = e
                logger.warning(f"LLM error on attempt {attempt + 1}: {e}")
                if attempt < self.max_retries - 1:
                    await asyncio.sleep(2 ** attempt)
        
        # S2 원칙 ⑦: 폐쇄망 fallback
        logger.warning("External LLM failed, trying local model fallback")
        try:
            response = await self.llm_provider.generate(
                prompt=prompt,
                model_hint="local",  # 로컬 모델 강제
                temperature=0.0,
                max_tokens=2000
            )
            return response.get("text", ""), response.get("usage", {})
        except Exception as fallback_error:
            logger.error(f"Local model fallback also failed: {fallback_error}")
            raise last_error or fallback_error
    
    def _parse_extraction_response(self, llm_response: str) -> Dict[str, Any]:
        """LLM 응답을 JSON으로 파싱."""
        try:
            # JSON 블록 추출 (```json ... ``` 형식)
            if "```json" in llm_response:
                start = llm_response.index("```json") + len("```json")
                end = llm_response.index("```", start)
                json_str = llm_response[start:end].strip()
            else:
                json_str = llm_response.strip()
            
            return json.loads(json_str)
        except (json.JSONDecodeError, ValueError) as e:
            logger.error(f"Failed to parse LLM response as JSON: {e}")
            raise ValueError(f"Invalid JSON response from LLM: {e}")
    
    def _validate_extracted_fields(
        self,
        schema,  # ExtractionTargetSchema
        extracted_fields: Dict[str, Any]
    ) -> None:
        """추출된 필드가 스키마와 일치하는지 검증."""
        for field_name, field_def in schema.fields_definition.items():
            if field_name not in extracted_fields:
                if field_def.get("required", False):
                    logger.warning(
                        f"Required field '{field_name}' missing from extraction"
                    )
                extracted_fields[field_name] = None
            else:
                # 타입 검증 (선택사항)
                value = extracted_fields[field_name]
                expected_type = field_def.get("field_type", "string")
                if value is not None and not self._check_type(value, expected_type):
                    logger.warning(
                        f"Field '{field_name}' type mismatch: "
                        f"expected {expected_type}, got {type(value).__name__}"
                    )
    
    def _check_type(self, value: Any, expected_type: str) -> bool:
        """값이 기대 타입과 일치하는지 확인."""
        type_map = {
            "string": str,
            "integer": int,
            "float": float,
            "boolean": bool,
            "date": str,  # 문자열로 검증
            "array": list,
            "object": dict
        }
        
        expected_py_type = type_map.get(expected_type, str)
        return isinstance(value, expected_py_type)
    
    def _compute_confidence_scores(
        self,
        schema,
        extracted_fields: Dict[str, Any]
    ) -> list[ExtractionConfidenceScore]:
        """신뢰도 점수 계산 (간단한 휴리스틱)."""
        scores = []
        
        for field_name, value in extracted_fields.items():
            if value is None:
                confidence = 0.0
                reason = "Field missing from source"
            elif isinstance(value, str) and len(value) > 100:
                confidence = 0.85
                reason = "Long text field"
            elif isinstance(value, str) and len(value) < 5:
                confidence = 0.7
                reason = "Short text field"
            elif isinstance(value, (list, dict)):
                confidence = 0.75
                reason = "Complex field (list/dict)"
            else:
                confidence = 0.9
                reason = "Simple field (string/number/date)"
            
            scores.append(
                ExtractionConfidenceScore(
                    field_name=field_name,
                    confidence=confidence,
                    reason=reason
                )
            )
        
        return scores
    
    def _estimate_cost(self, tokens_used: Optional[Dict[str, int]]) -> Optional[float]:
        """토큰 기반 비용 추정."""
        if not tokens_used:
            return None
        
        # 모델별 가격 (USD per 1M tokens)
        pricing = {
            "gpt-4o": {"input": 5.0, "output": 15.0},
            "gpt-4-turbo": {"input": 10.0, "output": 30.0},
            "claude-3.5-sonnet": {"input": 3.0, "output": 15.0},
            "local": {"input": 0.0, "output": 0.0}
        }
        
        input_tokens = tokens_used.get("input", 0)
        output_tokens = tokens_used.get("output", 0)
        model = self.llm_provider_type
        
        rates = pricing.get(model, {"input": 0.0, "output": 0.0})
        cost = (
            (input_tokens / 1_000_000) * rates["input"] +
            (output_tokens / 1_000_000) * rates["output"]
        )
        
        return cost if cost > 0 else None
    
    def _get_prompt_version(self) -> str:
        """프롬프트 템플릿 버전 (SHA256 hash)."""
        try:
            with open(
                "/backend/app/services/extraction/templates/extraction_prompt.jinja2",
                "r"
            ) as f:
                template_content = f.read()
            return hashlib.sha256(
                template_content.encode('utf-8')
            ).hexdigest()[:8]
        except Exception:
            return "unknown"
```

### 4-6. 단위 테스트

**파일:** `/backend/tests/services/extraction/test_extraction_pipeline_service.py`

```python
"""
ExtractionPipelineService 단위 테스트.
"""

import json
from datetime import datetime
from unittest.mock import AsyncMock, MagicMock, patch
from uuid import uuid4

import pytest

from app.models.extraction import ExtractionStatus, ExtractionMode
from app.services.extraction.extraction_pipeline_service import ExtractionPipelineService


@pytest.fixture
def mock_llm_provider():
    """Mock LLM Provider."""
    provider = AsyncMock()
    provider.generate = AsyncMock(
        return_value={
            "text": json.dumps({
                "article_number": "1.2",
                "responsible_dept": "기술팀",
                "effective_date": "2024-01-01",
                "summary": "이 조항은...",
                "related_regulations": ["규정A", "규정B"]
            }),
            "usage": {"input": 150, "output": 200}
        }
    )
    return provider


@pytest.fixture
def mock_document_service():
    """Mock DocumentService."""
    service = AsyncMock()
    service.get_document = AsyncMock(
        return_value=MagicMock(
            id=uuid4(),
            content_text="Mock document content...",
            type="regulation"
        )
    )
    return service


@pytest.fixture
def mock_schema_service():
    """Mock ExtractionTargetSchemaService."""
    service = AsyncMock()
    service.get_schema_by_doc_type = AsyncMock(
        return_value=MagicMock(
            id="regulation",
            version=1,
            fields_definition={
                "article_number": {
                    "field_type": "string",
                    "required": True,
                    "instruction": "원문의 조항 번호 추출"
                },
                "responsible_dept": {
                    "field_type": "string",
                    "required": True,
                    "instruction": "담당 부서명 추출"
                },
                "effective_date": {
                    "field_type": "date",
                    "required": False,
                    "instruction": "발효일 추출"
                },
                "summary": {
                    "field_type": "string",
                    "required": True,
                    "instruction": "조항 요약"
                },
                "related_regulations": {
                    "field_type": "array",
                    "required": False,
                    "instruction": "관련 규정 목록"
                }
            }
        )
    )
    return service


@pytest.fixture
def mock_audit_logger():
    """Mock AuditLogger."""
    logger = AsyncMock()
    logger.log = AsyncMock()
    return logger


@pytest.fixture
def extraction_pipeline_service(
    mock_llm_provider,
    mock_document_service,
    mock_schema_service,
    mock_audit_logger
):
    """ExtractionPipelineService 인스턴스."""
    service = ExtractionPipelineService(
        db_session=MagicMock(),
        llm_provider=mock_llm_provider,
        document_service=mock_document_service,
        schema_service=mock_schema_service,
        audit_logger=mock_audit_logger,
        template_dir="/backend/app/services/extraction/templates"
    )
    return service


@pytest.mark.asyncio
async def test_execute_extraction_pipeline_success(extraction_pipeline_service):
    """성공 케이스: 추출 파이프라인 전체 실행."""
    document_id = uuid4()
    scope_id = uuid4()
    
    # 실행
    result = await extraction_pipeline_service._execute_extraction_pipeline(
        document_id=document_id,
        document_version=1,
        scope_id=scope_id,
        doc_type="regulation"
    )
    
    # 검증
    assert result is not None
    assert result.document_id == document_id
    assert result.status == ExtractionStatus.PENDING
    assert result.extraction_model == "openai"  # 기본값
    assert "article_number" in result.extracted_fields
    assert result.extracted_fields["article_number"] == "1.2"


@pytest.mark.asyncio
async def test_parse_extraction_response_json_block(extraction_pipeline_service):
    """JSON 블록 파싱 (```json ... ``` 형식)."""
    llm_response = """
    다음은 추출 결과입니다:
    
    ```json
    {
        "article_number": "1.2",
        "summary": "이 조항은..."
    }
    ```
    """
    
    result = extraction_pipeline_service._parse_extraction_response(llm_response)
    
    assert result["article_number"] == "1.2"
    assert "이 조항은" in result["summary"]


@pytest.mark.asyncio
async def test_parse_extraction_response_invalid_json(extraction_pipeline_service):
    """유효하지 않은 JSON 파싱 에러."""
    llm_response = "{ invalid json }"
    
    with pytest.raises(ValueError, match="Invalid JSON response"):
        extraction_pipeline_service._parse_extraction_response(llm_response)


@pytest.mark.asyncio
async def test_confidence_score_computation(extraction_pipeline_service, mock_schema_service):
    """신뢰도 점수 계산."""
    schema = await mock_schema_service.get_schema_by_doc_type("regulation", uuid4())
    
    extracted_fields = {
        "article_number": "1.2",
        "summary": "This is a very long summary that spans multiple sentences and provides detailed information about the article in question, which should have higher confidence due to the length and detail...",
        "related_regulations": ["규정A", "규정B", "규정C"]
    }
    
    scores = extraction_pipeline_service._compute_confidence_scores(
        schema, extracted_fields
    )
    
    # 각 필드별 신뢰도 확인
    score_dict = {s.field_name: s.confidence for s in scores}
    assert score_dict["article_number"] >= 0.7
    assert score_dict["summary"] >= 0.8  # 긴 텍스트
    assert score_dict["related_regulations"] >= 0.7  # 복합 타입


def test_type_check(extraction_pipeline_service):
    """타입 검증."""
    assert extraction_pipeline_service._check_type("string", "string")
    assert extraction_pipeline_service._check_type(42, "integer")
    assert extraction_pipeline_service._check_type(3.14, "float")
    assert extraction_pipeline_service._check_type(True, "boolean")
    assert extraction_pipeline_service._check_type(["a", "b"], "array")
    assert extraction_pipeline_service._check_type({"key": "value"}, "object")
    
    assert not extraction_pipeline_service._check_type("string", "integer")


def test_cost_estimation(extraction_pipeline_service):
    """비용 추정 계산."""
    tokens = {"input": 1_000_000, "output": 1_000_000}
    
    extraction_pipeline_service.llm_provider_type = "gpt-4o"
    cost = extraction_pipeline_service._estimate_cost(tokens)
    
    # gpt-4o: input $5/M, output $15/M
    # total: (1M * 5 + 1M * 15) / 1M = $20
    assert cost == pytest.approx(20.0)


@pytest.mark.asyncio
async def test_validation_missing_required_field(extraction_pipeline_service, mock_schema_service):
    """필수 필드 누락 검증."""
    schema = await mock_schema_service.get_schema_by_doc_type("regulation", uuid4())
    
    extracted_fields = {
        # article_number (필수) 누락
        "summary": "이 조항은..."
    }
    
    # 검증 실행 (필수 필드 누락 경고만 발생, 예외는 아님)
    extraction_pipeline_service._validate_extracted_fields(schema, extracted_fields)
    
    # 누락된 필드는 None으로 채워짐
    assert extracted_fields["article_number"] is None


@pytest.mark.asyncio
async def test_llm_call_with_retry_success(extraction_pipeline_service, mock_llm_provider):
    """LLM 호출 성공 (첫 시도)."""
    response_text, usage = await extraction_pipeline_service._call_llm_with_retry(
        prompt="test prompt",
        model_hint="openai"
    )
    
    assert response_text
    assert usage["input"] == 150
    assert usage["output"] == 200


@pytest.mark.asyncio
async def test_llm_call_fallback_to_local(extraction_pipeline_service, mock_llm_provider):
    """LLM 호출 실패 → 로컬 모델 fallback."""
    # 첫 N번 실패, 마지막에 성공
    call_count = [0]
    
    async def side_effect(**kwargs):
        call_count[0] += 1
        if kwargs.get("model_hint") == "local":
            return {
                "text": json.dumps({"field": "value"}),
                "usage": {}
            }
        raise Exception("External LLM failed")
    
    mock_llm_provider.generate = AsyncMock(side_effect=side_effect)
    
    response_text, usage = await extraction_pipeline_service._call_llm_with_retry(
        prompt="test prompt",
        model_hint="openai"
    )
    
    # 로컬 모델로 성공
    assert response_text


# ==================== Integration Tests ====================

@pytest.mark.asyncio
async def test_trigger_on_document_create(extraction_pipeline_service):
    """문서 생성 이벤트 트리거 테스트."""
    document_id = uuid4()
    scope_id = uuid4()
    
    # 트리거 (비동기, 반환값 없음)
    result = await extraction_pipeline_service.trigger_on_document_create(
        document_id=document_id,
        document_version=1,
        scope_id=scope_id,
        doc_type="regulation"
    )
    
    # 감시 로그 확인
    extraction_pipeline_service.audit_logger.log.assert_called()
    
    # 비동기 실행이므로 None 반환
    assert result is None
```

## 5. 산출물

1. **ExtractionCandidate Pydantic 모델**
   - 파일: `/backend/app/models/extraction.py`
   - 크기: ~15KB

2. **ExtractionCandidateORM + Alembic 마이그레이션**
   - 파일: `/backend/app/db/models/extraction.py`
   - 마이그레이션: `/backend/alembic/versions/*_create_extraction_candidates_table.py`
   - 크기: ~8KB

3. **ExtractionCandidateRepository**
   - 파일: `/backend/app/repositories/extraction_candidate_repository.py`
   - 크기: ~10KB

4. **LLM 추출 프롬프트 템플릿**
   - 파일: `/backend/app/services/extraction/templates/extraction_prompt.jinja2`
   - 크기: ~3KB

5. **ExtractionPipelineService**
   - 파일: `/backend/app/services/extraction/extraction_pipeline_service.py`
   - 크기: ~25KB

6. **단위 테스트**
   - 파일: `/backend/tests/services/extraction/test_extraction_pipeline_service.py`
   - 크기: ~12KB

## 6. 완료 기준

1. ExtractionCandidate Pydantic 모델이 정의되고, 모든 필드가 스키마와 일치한다.
2. ExtractionCandidateORM이 정의되고, Alembic 마이그레이션이 작동한다.
3. ExtractionCandidateRepository가 Scope Profile 기반 ACL을 모든 연산에 적용한다.
4. LLM 추출 프롬프트 템플릿이 Jinja2로 렌더링되며, 동적 스키마 주입을 지원한다.
5. ExtractionPipelineService가 문서 업로드 이벤트를 감지하고 비동기로 추출 파이프라인을 실행한다.
6. LLM 호출이 실패할 때 exponential backoff 재시도를 수행하고, 폐쇄망 환경에서 로컬 모델로 fallback한다.
7. 추출 결과가 ExtractionCandidate로 저장되고, status=pending으로 사용자 승인 큐에 진입한다.
8. 감시 로그(audit log)가 모든 주요 단계에서 actor_type="agent"로 기록된다.
9. 단위 테스트가 90% 이상 코드 커버리지를 달성한다.
10. 통합 테스트에서 Document 업로드 → ExtractionCandidate 생성 → 상태 pending 확인까지 정상 작동한다.

## 7. 작업 지침

### 지침 7-1. Scope Profile ACL 적용
모든 Repository 연산(create, read, update, delete)에서 `enforce_scope_acl()` 함수를 호출하여 Scope 유효성을 검증하라. S2 원칙 ⑥을 준수한다.

### 지침 7-2. 비동기 실행
문서 생성/개정 이벤트는 메인 스레드를 블로킹하지 않도록 `asyncio.create_task()` 또는 Celery worker로 비동기 실행하라.

### 지침 7-3. 폐쇄망 환경 지원
LLM_PROVIDER 환경 변수로 openai/anthropic/local을 선택 가능하게 구현하고, external LLM 호출 실패 시 자동으로 로컬 모델로 fallback하라 (S2 원칙 ⑦).

### 지침 7-4. 에러 처리
LLM 타임아웃, JSON parsing 오류, 스키마 미스매치 등 모든 에러를 명확하게 처리하고, Dead Letter Queue 또는 manual review 큐에 진입시키라.

### 지침 7-5. 감시 로깅
추출 시작, 완료, 실패 시점에 audit log를 기록하고, actor_type="agent"로 표시하라 (S2 원칙 ⑤).

### 지침 7-6. 원문 무결성 검증
문서 업로드 시 SHA256 해시를 계산하여 document_content_hash 필드에 저장하고, 추출 재현성 검증에 사용하라.

### 지침 7-7. 코드 리뷰 & 테스트
구현 완료 후 코드 리뷰를 거쳐 S1 원칙 ①~④과 S2 원칙 ⑤~⑦을 검증하고, 단위/통합 테스트로 정확성을 확인하라.

---

End of Task 8-4
