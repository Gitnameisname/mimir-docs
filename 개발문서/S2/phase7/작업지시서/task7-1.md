# Task 7-1. Golden Q&A 도메인 모델 + DB 스키마

## 1. 작업 목적

Phase 7 RAG 품질 평가 인프라의 기초가 되는 **Golden Q&A 도메인 모델**을 정의하고, 데이터베이스 스키마를 설계 및 구현한다. Golden Set을 관리 가능한 도메인 객체로 정의하여, 기대하는 질문-답변-근거-인용 4-tuple의 메타데이터를 체계적으로 관리하고, 버전 관리, 권한 제어(Scope Profile 기반), 감사 로깅(actor_type 필드 포함)을 지원한다.

## 2. 작업 범위

### 포함 범위

1. **Golden Q&A 도메인 모델 정의**
   - GoldenSet: Q&A 항목들의 컬렉션 (이름, 설명, 도메인, 버전 메타데이터)
   - GoldenItem: 개별 Q&A 항목 (question, expected_answer, expected_source_docs, expected_citations, notes)
   - SourceRef: 기대 근거 문서 참조 (document_id, version_id, node_id)
   - Citation5Tuple: Phase 2 Citation 구조와 통합 (document_id, version_id, node_id, span_offset, content_hash)
   - Pydantic 스키마 정의 (요청/응답 DTO)

2. **SQLAlchemy ORM 모델 정의**
   - GoldenSetORM: PostgreSQL 테이블 매핑 (golden_sets 테이블)
   - GoldenItemORM: golden_items 테이블 (다대일 관계 GoldenSet)
   - 버전 관리 필드: version, created_at, updated_at, created_by, updated_by
   - Soft delete 지원: deleted_at, is_deleted 플래그
   - Scope Profile 기반 ACL: scope_id 필드로 접근 범위 제어 (S2 원칙 ⑤)

3. **Alembic 마이그레이션 스크립트**
   - golden_sets 테이블 생성 마이그레이션
   - golden_items 테이블 생성 및 foreign key 마이그레이션
   - 인덱스: (scope_id, domain), (golden_set_id, is_deleted), (created_by)
   - 감사 로그와 연계할 수 있는 audit_log_id 외래키 (optional)

4. **Repository 패턴 구현**
   - GoldenSetRepository: CRUD + 버전 조회 + 필터링
   - GoldenItemRepository: CRUD + 일괄 조회
   - Scope Profile 기반 ACL 필터 적용 (S2 원칙 ⑥: 모든 조회/쓰기 API에 의무 적용)
   - 폐쇄망 환경 지원 (S2 원칙 ⑦: 외부 의존 없이 로컬 저장소에서만 쿼리)

5. **단위 테스트**
   - GoldenSet, GoldenItem 생성 및 관계 검증
   - Scope Profile 기반 필터링 정확성
   - 버전 자동 증가 로직
   - Soft delete 후 복구 불가능 확인
   - 감사 로그 연계 데이터 정합성

6. **관계 정합성**
   - Phase 2 Citation 5-tuple 구조와 일치하는지 검증
   - expected_citations가 Citation 도메인 모델과 호환되는지 확인
   - expected_source_docs의 document_id, version_id, node_id가 Document/Node ORM과 참조 가능한지 확인

### 제외 범위

- CRUD API 라우터 (Task 7-2)
- import/export 엔드포인트 (Task 7-3)
- 평가 러너 및 지표 계산 로직 (Task 7-2)
- CI 게이트 구현 (Task 7-3)
- UI 모듈 (별도 FG)

## 3. 선행 조건

- Phase 2 완료: Citation 5-tuple 정의 및 Document/Node 도메인 모델 확립
- Phase 4 완료: 버전 관리 기본 구조 (Document version 테이블)
- PostgreSQL + SQLAlchemy + Alembic 설정 완료
- Pydantic v2 설치
- S2 Scope Profile 구조 최종 확정

## 4. 주요 작업 항목

### 4-1. Pydantic 도메인 모델 정의

**파일:** `/backend/app/models/golden_set.py`

```python
"""
Golden Q&A 도메인 모델 (Pydantic 기반).

GoldenSet과 GoldenItem을 도메인 엔티티로 정의하며, 
Phase 2 Citation 5-tuple과의 호환성을 유지한다.
"""

from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import Any, Optional, List
from uuid import UUID

from pydantic import BaseModel, Field, validator


# ==================== Enums ====================

class GoldenSetDomain(str, Enum):
    """Golden Set의 도메인 분류."""
    POLICY = "policy"
    REGULATION = "regulation"
    TECHNICAL_GUIDE = "technical_guide"
    MANUAL = "manual"
    FAQ = "faq"
    CUSTOM = "custom"


class GoldenSetStatus(str, Enum):
    """Golden Set의 상태."""
    DRAFT = "draft"
    PUBLISHED = "published"
    ARCHIVED = "archived"


# ==================== Reference Models ====================

class SourceRef(BaseModel):
    """기대 근거 문서 참조."""
    document_id: str = Field(..., description="Document UUID (문자열)")
    version_id: int = Field(..., ge=1, description="Document version ID")
    node_id: str = Field(..., description="Node UUID (문자열)")
    
    class Config:
        json_schema_extra = {
            "example": {
                "document_id": "550e8400-e29b-41d4-a716-446655440000",
                "version_id": 3,
                "node_id": "node-abc123"
            }
        }


class Citation5Tuple(BaseModel):
    """Phase 2와 동일한 Citation 5-tuple 구조."""
    document_id: str = Field(..., description="Document UUID")
    version_id: int = Field(..., ge=1, description="Document version ID")
    node_id: str = Field(..., description="Node UUID")
    span_offset: tuple[int, int] = Field(
        ..., 
        description="(start_char, end_char) offset in node content"
    )
    content_hash: str = Field(
        ..., 
        description="SHA256 hash of cited content (immutable verification)"
    )
    
    class Config:
        json_schema_extra = {
            "example": {
                "document_id": "550e8400-e29b-41d4-a716-446655440000",
                "version_id": 3,
                "node_id": "node-abc123",
                "span_offset": [45, 120],
                "content_hash": "abcd1234efgh5678..."
            }
        }


# ==================== Domain Models ====================

class GoldenItem(BaseModel):
    """
    개별 Golden Q&A 항목 (도메인 모델).
    
    Attributes:
        id: 항목 UUID
        golden_set_id: 소속 GoldenSet UUID
        version: 이 항목의 버전 (GoldenSet 내에서 독립적 버전 관리)
        question: 사용자가 제시할 질문 (일반화된 형식)
        expected_answer: 기대하는 답변 (요약 또는 전문)
        expected_source_docs: 기대 근거 문서 목록 (SourceRef 리스트)
        expected_citations: 기대 인용 5-tuple 리스트
        notes: 평가자 주석 (왜 이것이 정답인가, 평가 시 유의점)
        created_at, created_by: 생성 메타데이터
        updated_at, updated_by: 수정 메타데이터
    """
    id: str = Field(..., description="GoldenItem UUID")
    golden_set_id: str = Field(..., description="Parent GoldenSet UUID")
    version: int = Field(default=1, ge=1, description="Version number (auto-incremented)")
    
    question: str = Field(
        ...,
        min_length=1,
        max_length=2000,
        description="사용자 질문 (일반화된 형식)"
    )
    expected_answer: str = Field(
        ...,
        min_length=1,
        max_length=5000,
        description="기대하는 답변"
    )
    expected_source_docs: List[SourceRef] = Field(
        default_factory=list,
        description="기대 근거 문서 목록"
    )
    expected_citations: List[Citation5Tuple] = Field(
        default_factory=list,
        description="기대 인용 5-tuple 리스트"
    )
    notes: Optional[str] = Field(
        default=None,
        max_length=1000,
        description="평가자 주석"
    )
    
    created_at: datetime
    created_by: str = Field(..., description="Creator user_id or actor_id (S2 ⑤)")
    updated_at: datetime
    updated_by: Optional[str] = Field(
        default=None,
        description="Last updater actor_id"
    )
    
    @validator("expected_source_docs")
    def validate_source_docs(cls, v: List[SourceRef]) -> List[SourceRef]:
        """expected_source_docs는 최소 1개 이상이어야 한다."""
        if len(v) == 0:
            raise ValueError("expected_source_docs must contain at least one reference")
        return v
    
    class Config:
        json_schema_extra = {
            "example": {
                "id": "item-123",
                "golden_set_id": "set-456",
                "version": 1,
                "question": "Docker란 무엇인가?",
                "expected_answer": "Docker는 컨테이너 기술을 통해 애플리케이션을 격리된 환경에서 실행하는 플랫폼이다.",
                "expected_source_docs": [
                    {
                        "document_id": "doc-789",
                        "version_id": 2,
                        "node_id": "node-001"
                    }
                ],
                "expected_citations": [
                    {
                        "document_id": "doc-789",
                        "version_id": 2,
                        "node_id": "node-001",
                        "span_offset": [0, 80],
                        "content_hash": "abc123..."
                    }
                ],
                "notes": "공식 문서의 Introduction 섹션",
                "created_at": "2025-01-01T10:00:00Z",
                "created_by": "user-123",
                "updated_at": "2025-01-01T10:00:00Z",
                "updated_by": None
            }
        }


class GoldenSet(BaseModel):
    """
    Golden Q&A 컬렉션 (도메인 모델).
    
    Attributes:
        id: GoldenSet UUID
        scope_id: S2 원칙 ⑥: Scope Profile ID (접근 범위)
        name: 컬렉션 이름 (예: "기술 문서 RAG")
        description: 컬렉션 설명
        domain: 도메인 분류 (policy, regulation, technical_guide 등)
        status: 상태 (draft, published, archived)
        version: 이 GoldenSet의 버전 (items 추가/수정 시 자동 증가)
        items: 포함된 GoldenItem 리스트 (조회 시에만 포함)
        metadata: 확장 메타데이터 (JSON)
        created_at, created_by: 생성 메타데이터
        updated_at, updated_by: 수정 메타데이터
        deleted_at, is_deleted: Soft delete 지원
    """
    id: str = Field(..., description="GoldenSet UUID")
    scope_id: str = Field(
        ...,
        description="S2 ⑥: Scope Profile ID (ACL 필터링 기준)"
    )
    
    name: str = Field(
        ...,
        min_length=1,
        max_length=200,
        description="GoldenSet 이름"
    )
    description: Optional[str] = Field(
        default=None,
        max_length=1000,
        description="상세 설명"
    )
    domain: GoldenSetDomain = Field(
        default=GoldenSetDomain.CUSTOM,
        description="도메인 분류"
    )
    status: GoldenSetStatus = Field(
        default=GoldenSetStatus.DRAFT,
        description="상태"
    )
    version: int = Field(default=1, ge=1, description="Version number")
    
    items: Optional[List[GoldenItem]] = Field(
        default=None,
        description="포함 Q&A 항목 (조회 시에만 포함)"
    )
    metadata: dict[str, Any] = Field(
        default_factory=dict,
        description="확장 메타데이터"
    )
    
    created_at: datetime
    created_by: str = Field(..., description="Creator actor_id")
    updated_at: datetime
    updated_by: Optional[str] = Field(default=None, description="Last updater actor_id")
    
    deleted_at: Optional[datetime] = Field(default=None, description="Soft delete timestamp")
    is_deleted: bool = Field(default=False, description="Soft delete flag")
    
    class Config:
        json_schema_extra = {
            "example": {
                "id": "set-456",
                "scope_id": "scope-123",
                "name": "기술 문서 RAG",
                "description": "내부 기술 명세서 검색 평가",
                "domain": "technical_guide",
                "status": "published",
                "version": 3,
                "items": [],  # 상세 조회 시에만 포함
                "metadata": {"tags": ["docker", "kubernetes"], "owner": "platform-team"},
                "created_at": "2025-01-01T09:00:00Z",
                "created_by": "user-123",
                "updated_at": "2025-01-15T14:30:00Z",
                "updated_by": "user-456",
                "deleted_at": None,
                "is_deleted": False
            }
        }


# ==================== Pydantic Schemas (DTO) ====================

class GoldenItemCreateRequest(BaseModel):
    """GoldenItem 생성 요청 DTO."""
    question: str = Field(..., min_length=1, max_length=2000)
    expected_answer: str = Field(..., min_length=1, max_length=5000)
    expected_source_docs: List[SourceRef] = Field(..., min_items=1)
    expected_citations: List[Citation5Tuple] = Field(default_factory=list)
    notes: Optional[str] = Field(default=None, max_length=1000)


class GoldenItemUpdateRequest(BaseModel):
    """GoldenItem 수정 요청 DTO (부분 수정 지원)."""
    question: Optional[str] = Field(default=None, min_length=1, max_length=2000)
    expected_answer: Optional[str] = Field(default=None, min_length=1, max_length=5000)
    expected_source_docs: Optional[List[SourceRef]] = Field(default=None, min_items=1)
    expected_citations: Optional[List[Citation5Tuple]] = Field(default=None)
    notes: Optional[str] = Field(default=None, max_length=1000)


class GoldenItemResponse(GoldenItem):
    """GoldenItem 응답 DTO (읽기 전용)."""
    pass


class GoldenSetCreateRequest(BaseModel):
    """GoldenSet 생성 요청 DTO."""
    name: str = Field(..., min_length=1, max_length=200)
    description: Optional[str] = Field(default=None, max_length=1000)
    domain: GoldenSetDomain = Field(default=GoldenSetDomain.CUSTOM)
    metadata: dict[str, Any] = Field(default_factory=dict)


class GoldenSetUpdateRequest(BaseModel):
    """GoldenSet 수정 요청 DTO (메타데이터만)."""
    name: Optional[str] = Field(default=None, min_length=1, max_length=200)
    description: Optional[str] = Field(default=None, max_length=1000)
    domain: Optional[GoldenSetDomain] = Field(default=None)
    status: Optional[GoldenSetStatus] = Field(default=None)
    metadata: Optional[dict[str, Any]] = Field(default=None)


class GoldenSetResponse(GoldenSet):
    """GoldenSet 응답 DTO (읽기 전용)."""
    item_count: Optional[int] = Field(
        default=None,
        description="포함된 GoldenItem 개수 (목록 조회 시만 포함)"
    )


class GoldenSetDetailResponse(GoldenSetResponse):
    """GoldenSet 상세 조회 응답 DTO (모든 items 포함)."""
    items: List[GoldenItemResponse] = Field(
        default_factory=list,
        description="포함된 모든 GoldenItem"
    )


class GoldenSetVersionInfo(BaseModel):
    """GoldenSet 버전 정보."""
    version: int
    created_at: datetime
    created_by: str
    updated_by: Optional[str] = None
    item_count: int = Field(..., ge=0)


class GoldenSetVersionDiff(BaseModel):
    """두 GoldenSet 버전 간의 차이."""
    from_version: int
    to_version: int
    items_added: List[str] = Field(default_factory=list, description="추가된 item ID 리스트")
    items_modified: List[str] = Field(default_factory=list, description="수정된 item ID 리스트")
    items_deleted: List[str] = Field(default_factory=list, description="삭제된 item ID 리스트")
    modified_at: datetime


# ==================== Dataclass for DB Layer ====================

@dataclass
class GoldenSetRecord:
    """GoldenSet ORM 레코드 -> 도메인 모델 변환용."""
    id: str
    scope_id: str
    name: str
    description: Optional[str]
    domain: str
    status: str
    version: int
    metadata: dict[str, Any]
    created_at: datetime
    created_by: str
    updated_at: datetime
    updated_by: Optional[str]
    deleted_at: Optional[datetime]
    is_deleted: bool
    
    def to_domain(self) -> GoldenSet:
        """ORM 레코드를 도메인 모델로 변환."""
        return GoldenSet(
            id=self.id,
            scope_id=self.scope_id,
            name=self.name,
            description=self.description,
            domain=GoldenSetDomain(self.domain),
            status=GoldenSetStatus(self.status),
            version=self.version,
            metadata=self.metadata or {},
            created_at=self.created_at,
            created_by=self.created_by,
            updated_at=self.updated_at,
            updated_by=self.updated_by,
            deleted_at=self.deleted_at,
            is_deleted=self.is_deleted,
        )


@dataclass
class GoldenItemRecord:
    """GoldenItem ORM 레코드 -> 도메인 모델 변환용."""
    id: str
    golden_set_id: str
    version: int
    question: str
    expected_answer: str
    expected_source_docs: list[dict[str, Any]]  # JSON
    expected_citations: list[dict[str, Any]]    # JSON
    notes: Optional[str]
    created_at: datetime
    created_by: str
    updated_at: datetime
    updated_by: Optional[str]
    
    def to_domain(self) -> GoldenItem:
        """ORM 레코드를 도메인 모델로 변환."""
        source_docs = [
            SourceRef(**doc) for doc in (self.expected_source_docs or [])
        ]
        citations = [
            Citation5Tuple(**cite) for cite in (self.expected_citations or [])
        ]
        
        return GoldenItem(
            id=self.id,
            golden_set_id=self.golden_set_id,
            version=self.version,
            question=self.question,
            expected_answer=self.expected_answer,
            expected_source_docs=source_docs,
            expected_citations=citations,
            notes=self.notes,
            created_at=self.created_at,
            created_by=self.created_by,
            updated_at=self.updated_at,
            updated_by=self.updated_by,
        )
```

### 4-2. SQLAlchemy ORM 모델 정의

**파일:** `/backend/app/db/models/golden_set_orm.py`

```python
"""
Golden Q&A ORM 모델 (SQLAlchemy).

GoldenSet 및 GoldenItem에 해당하는 DB 테이블을 정의한다.
"""

from __future__ import annotations

from datetime import datetime
from typing import Optional, List

from sqlalchemy import (
    String, Integer, Text, DateTime, Boolean, JSON, ForeignKey,
    UniqueConstraint, Index, func, Table, Column, TIMESTAMP
)
from sqlalchemy.orm import declarative_base, relationship
from sqlalchemy.ext.hybrid import hybrid_property

Base = declarative_base()


class GoldenSetORM(Base):
    """Golden Set ORM 모델."""
    
    __tablename__ = "golden_sets"
    
    # ========== Primary & Foreign Keys ==========
    id = Column(String(36), primary_key=True, nullable=False)  # UUID
    scope_id = Column(String(36), nullable=False, index=True)  # S2 ⑥: Scope Profile
    
    # ========== Core Fields ==========
    name = Column(String(200), nullable=False)
    description = Column(Text, nullable=True)
    domain = Column(String(50), nullable=False, default="custom")  # Enum: policy, regulation, etc.
    status = Column(String(20), nullable=False, default="draft")   # Enum: draft, published, archived
    version = Column(Integer, nullable=False, default=1)
    metadata = Column(JSON, nullable=False, default={})  # JSONB for extensions
    
    # ========== Audit Fields ==========
    created_at = Column(TIMESTAMP(timezone=True), nullable=False, default=datetime.utcnow)
    created_by = Column(String(255), nullable=False)  # actor_id (S2 ⑤)
    updated_at = Column(TIMESTAMP(timezone=True), nullable=False, default=datetime.utcnow, onupdate=datetime.utcnow)
    updated_by = Column(String(255), nullable=True)   # Last modifier actor_id
    
    # ========== Soft Delete ==========
    deleted_at = Column(TIMESTAMP(timezone=True), nullable=True)
    is_deleted = Column(Boolean, nullable=False, default=False, index=True)
    
    # ========== Relationships ==========
    items: List[GoldenItemORM] = relationship(
        "GoldenItemORM",
        back_populates="golden_set",
        cascade="all, delete-orphan",
        lazy="selectin",
        foreign_keys="GoldenItemORM.golden_set_id"
    )
    
    # ========== Indices ==========
    __table_args__ = (
        Index("idx_golden_sets_scope_domain", "scope_id", "domain"),
        Index("idx_golden_sets_scope_status", "scope_id", "status"),
        Index("idx_golden_sets_created_by", "created_by"),
        Index("idx_golden_sets_not_deleted", "is_deleted"),
        UniqueConstraint("scope_id", "name", "is_deleted", 
                         name="uq_golden_sets_scope_name"),
    )
    
    @hybrid_property
    def is_active(self) -> bool:
        """활성 상태 여부 (삭제되지 않음)."""
        return not self.is_deleted
    
    def __repr__(self) -> str:
        return (f"<GoldenSetORM(id='{self.id}', name='{self.name}', "
                f"version={self.version}, scope_id='{self.scope_id}')>")


class GoldenItemORM(Base):
    """Golden Item ORM 모델."""
    
    __tablename__ = "golden_items"
    
    # ========== Primary & Foreign Keys ==========
    id = Column(String(36), primary_key=True, nullable=False)  # UUID
    golden_set_id = Column(String(36), ForeignKey("golden_sets.id"), 
                           nullable=False, index=True)
    
    # ========== Core Fields ==========
    version = Column(Integer, nullable=False, default=1)
    question = Column(String(2000), nullable=False)
    expected_answer = Column(Text, nullable=False)
    expected_source_docs = Column(JSON, nullable=False, default=[])  # List[SourceRef]
    expected_citations = Column(JSON, nullable=False, default=[])     # List[Citation5Tuple]
    notes = Column(Text, nullable=True)
    
    # ========== Audit Fields ==========
    created_at = Column(TIMESTAMP(timezone=True), nullable=False, default=datetime.utcnow)
    created_by = Column(String(255), nullable=False)
    updated_at = Column(TIMESTAMP(timezone=True), nullable=False, default=datetime.utcnow, onupdate=datetime.utcnow)
    updated_by = Column(String(255), nullable=True)
    
    # ========== Soft Delete ==========
    deleted_at = Column(TIMESTAMP(timezone=True), nullable=True)
    is_deleted = Column(Boolean, nullable=False, default=False, index=True)
    
    # ========== Relationships ==========
    golden_set: GoldenSetORM = relationship(
        "GoldenSetORM",
        back_populates="items",
        foreign_keys=[golden_set_id]
    )
    
    # ========== Indices ==========
    __table_args__ = (
        Index("idx_golden_items_golden_set", "golden_set_id"),
        Index("idx_golden_items_not_deleted", "is_deleted"),
        Index("idx_golden_items_created_by", "created_by"),
        UniqueConstraint("golden_set_id", "id", "is_deleted",
                         name="uq_golden_items_set_id"),
    )
    
    @hybrid_property
    def is_active(self) -> bool:
        """활성 상태 여부 (삭제되지 않음)."""
        return not self.is_deleted
    
    def __repr__(self) -> str:
        return (f"<GoldenItemORM(id='{self.id}', golden_set_id='{self.golden_set_id}', "
                f"version={self.version})>")


# ==================== Golden Set Version History Table ====================

class GoldenSetVersionORM(Base):
    """GoldenSet 버전 이력 추적 (Document.version과 유사한 구조)."""
    
    __tablename__ = "golden_set_versions"
    
    id = Column(String(36), primary_key=True, nullable=False)
    golden_set_id = Column(String(36), ForeignKey("golden_sets.id"), 
                           nullable=False, index=True)
    version = Column(Integer, nullable=False)
    
    # ========== Snapshot of GoldenSet state ==========
    name = Column(String(200), nullable=False)
    description = Column(Text, nullable=True)
    domain = Column(String(50), nullable=False)
    status = Column(String(20), nullable=False)
    metadata = Column(JSON, nullable=False, default={})
    
    # ========== Items at this version ==========
    # JSON 스냅샷: GoldenItem 전체 상태 저장
    items_snapshot = Column(JSON, nullable=False, default=[])
    
    # ========== Audit ==========
    created_at = Column(TIMESTAMP(timezone=True), nullable=False)
    created_by = Column(String(255), nullable=False)
    
    __table_args__ = (
        Index("idx_golden_set_versions_golden_set", "golden_set_id"),
        Index("idx_golden_set_versions_version", "golden_set_id", "version"),
        UniqueConstraint("golden_set_id", "version",
                         name="uq_golden_set_versions_unique"),
    )
    
    def __repr__(self) -> str:
        return (f"<GoldenSetVersionORM(golden_set_id='{self.golden_set_id}', "
                f"version={self.version})>")
```

### 4-3. Alembic 마이그레이션 스크립트

**파일:** `/backend/alembic/versions/2025_01_15_001_create_golden_sets.py`

```python
"""Create golden_sets and golden_items tables.

Revision ID: 2025_01_15_001
Revises: <previous_revision>
Create Date: 2025-01-15 10:00:00.000000

Golden Set 및 Golden Item 테이블을 생성하고,
Scope Profile 기반 ACL 및 soft delete 지원을 추가한다.
"""

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# revision identifiers, used by Alembic.
revision = '2025_01_15_001'
down_revision = '<previous_revision>'
branch_labels = None
depends_on = None


def upgrade() -> None:
    """Upgrade to create golden_sets and golden_items tables."""
    
    # ========== Create golden_sets table ==========
    op.create_table(
        'golden_sets',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('scope_id', sa.String(36), nullable=False),
        sa.Column('name', sa.String(200), nullable=False),
        sa.Column('description', sa.Text(), nullable=True),
        sa.Column('domain', sa.String(50), nullable=False, server_default='custom'),
        sa.Column('status', sa.String(20), nullable=False, server_default='draft'),
        sa.Column('version', sa.Integer(), nullable=False, server_default='1'),
        sa.Column('metadata', postgresql.JSON(astext_type=sa.Text()), nullable=False, server_default='{}'),
        sa.Column('created_at', sa.TIMESTAMP(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.Column('created_by', sa.String(255), nullable=False),
        sa.Column('updated_at', sa.TIMESTAMP(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.Column('updated_by', sa.String(255), nullable=True),
        sa.Column('deleted_at', sa.TIMESTAMP(timezone=True), nullable=True),
        sa.Column('is_deleted', sa.Boolean(), nullable=False, server_default='false'),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('scope_id', 'name', 'is_deleted', name='uq_golden_sets_scope_name'),
    )
    
    # Create indices for golden_sets
    op.create_index('idx_golden_sets_scope_domain', 'golden_sets', ['scope_id', 'domain'])
    op.create_index('idx_golden_sets_scope_status', 'golden_sets', ['scope_id', 'status'])
    op.create_index('idx_golden_sets_created_by', 'golden_sets', ['created_by'])
    op.create_index('idx_golden_sets_not_deleted', 'golden_sets', ['is_deleted'])
    
    # ========== Create golden_items table ==========
    op.create_table(
        'golden_items',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('golden_set_id', sa.String(36), nullable=False),
        sa.Column('version', sa.Integer(), nullable=False, server_default='1'),
        sa.Column('question', sa.String(2000), nullable=False),
        sa.Column('expected_answer', sa.Text(), nullable=False),
        sa.Column('expected_source_docs', postgresql.JSON(astext_type=sa.Text()), nullable=False, server_default='[]'),
        sa.Column('expected_citations', postgresql.JSON(astext_type=sa.Text()), nullable=False, server_default='[]'),
        sa.Column('notes', sa.Text(), nullable=True),
        sa.Column('created_at', sa.TIMESTAMP(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.Column('created_by', sa.String(255), nullable=False),
        sa.Column('updated_at', sa.TIMESTAMP(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.Column('updated_by', sa.String(255), nullable=True),
        sa.Column('deleted_at', sa.TIMESTAMP(timezone=True), nullable=True),
        sa.Column('is_deleted', sa.Boolean(), nullable=False, server_default='false'),
        sa.ForeignKeyConstraint(['golden_set_id'], ['golden_sets.id'], name='fk_golden_items_golden_set'),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('golden_set_id', 'id', 'is_deleted', name='uq_golden_items_set_id'),
    )
    
    # Create indices for golden_items
    op.create_index('idx_golden_items_golden_set', 'golden_items', ['golden_set_id'])
    op.create_index('idx_golden_items_not_deleted', 'golden_items', ['is_deleted'])
    op.create_index('idx_golden_items_created_by', 'golden_items', ['created_by'])
    
    # ========== Create golden_set_versions table (버전 이력) ==========
    op.create_table(
        'golden_set_versions',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('golden_set_id', sa.String(36), nullable=False),
        sa.Column('version', sa.Integer(), nullable=False),
        sa.Column('name', sa.String(200), nullable=False),
        sa.Column('description', sa.Text(), nullable=True),
        sa.Column('domain', sa.String(50), nullable=False),
        sa.Column('status', sa.String(20), nullable=False),
        sa.Column('metadata', postgresql.JSON(astext_type=sa.Text()), nullable=False, server_default='{}'),
        sa.Column('items_snapshot', postgresql.JSON(astext_type=sa.Text()), nullable=False, server_default='[]'),
        sa.Column('created_at', sa.TIMESTAMP(timezone=True), nullable=False),
        sa.Column('created_by', sa.String(255), nullable=False),
        sa.ForeignKeyConstraint(['golden_set_id'], ['golden_sets.id'], name='fk_golden_set_versions_golden_set'),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('golden_set_id', 'version', name='uq_golden_set_versions_unique'),
    )
    
    # Create indices for golden_set_versions
    op.create_index('idx_golden_set_versions_golden_set', 'golden_set_versions', ['golden_set_id'])
    op.create_index('idx_golden_set_versions_version', 'golden_set_versions', ['golden_set_id', 'version'])


def downgrade() -> None:
    """Downgrade to remove golden_sets and golden_items tables."""
    
    # Drop indices
    op.drop_index('idx_golden_set_versions_version')
    op.drop_index('idx_golden_set_versions_golden_set')
    op.drop_index('idx_golden_items_created_by')
    op.drop_index('idx_golden_items_not_deleted')
    op.drop_index('idx_golden_items_golden_set')
    op.drop_index('idx_golden_sets_not_deleted')
    op.drop_index('idx_golden_sets_created_by')
    op.drop_index('idx_golden_sets_scope_status')
    op.drop_index('idx_golden_sets_scope_domain')
    
    # Drop tables
    op.drop_table('golden_set_versions')
    op.drop_table('golden_items')
    op.drop_table('golden_sets')
```

### 4-4. Repository 패턴 구현

**파일:** `/backend/app/repositories/golden_set_repository.py`

```python
"""
Golden Set Repository (Data Access Layer).

S2 원칙 ⑥ 준수: 모든 조회/쓰기 API에 Scope Profile 기반 ACL 필터링 적용.
S2 원칙 ⑦ 준수: 폐쇄망 환경에서도 로컬 DB에서만 쿼리 수행.
"""

from __future__ import annotations

import json
from datetime import datetime
from typing import Optional, List, Dict, Any
from uuid import uuid4

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, and_, or_, func, desc, update

from app.db.models.golden_set_orm import GoldenSetORM, GoldenItemORM, GoldenSetVersionORM
from app.models.golden_set import (
    GoldenSet, GoldenItem, GoldenSetRecord, GoldenItemRecord,
    GoldenSetCreateRequest, GoldenSetUpdateRequest,
    GoldenItemCreateRequest, GoldenItemUpdateRequest,
)


class GoldenSetRepository:
    """GoldenSet 데이터 접근 계층."""
    
    def __init__(self, db_session: AsyncSession):
        self.db = db_session
    
    # ==================== Create ====================
    
    async def create_golden_set(
        self,
        scope_id: str,
        request: GoldenSetCreateRequest,
        created_by: str,
    ) -> GoldenSet:
        """
        새로운 GoldenSet 생성.
        
        Args:
            scope_id: S2 ⑥ Scope Profile ID
            request: 생성 요청 DTO
            created_by: 생성자 actor_id (S2 ⑤)
        
        Returns:
            생성된 GoldenSet 도메인 모델
        """
        now = datetime.utcnow()
        golden_set_id = str(uuid4())
        
        orm = GoldenSetORM(
            id=golden_set_id,
            scope_id=scope_id,
            name=request.name,
            description=request.description,
            domain=request.domain.value,
            status="draft",
            version=1,
            metadata=request.metadata or {},
            created_at=now,
            created_by=created_by,
            updated_at=now,
            updated_by=None,
        )
        
        self.db.add(orm)
        await self.db.flush()
        
        # 버전 이력 기록
        await self._create_version_snapshot(golden_set_id, 1, orm, [])
        
        await self.db.commit()
        return self._orm_to_domain(orm)
    
    # ==================== Read ====================
    
    async def get_by_id(
        self,
        golden_set_id: str,
        scope_id: str,
        include_items: bool = False,
    ) -> Optional[GoldenSet]:
        """
        ID로 GoldenSet 조회 (S2 ⑥: scope_id 필터 필수).
        
        Args:
            golden_set_id: GoldenSet ID
            scope_id: 요청자의 Scope Profile ID
            include_items: items를 함께 로드할지 여부
        
        Returns:
            GoldenSet 또는 None (권한 없음 또는 삭제됨)
        """
        query = select(GoldenSetORM).where(
            and_(
                GoldenSetORM.id == golden_set_id,
                GoldenSetORM.scope_id == scope_id,  # S2 ⑥: ACL 필터
                GoldenSetORM.is_deleted == False,
            )
        )
        
        result = await self.db.execute(query)
        orm = result.scalar_one_or_none()
        
        if orm is None:
            return None
        
        # items 로드 (필요한 경우)
        if include_items:
            await self.db.refresh(orm, ["items"])
        
        return self._orm_to_domain(orm, include_items=include_items)
    
    async def list_by_scope(
        self,
        scope_id: str,
        offset: int = 0,
        limit: int = 100,
        domain: Optional[str] = None,
        status: Optional[str] = None,
    ) -> tuple[List[GoldenSet], int]:
        """
        Scope 내 GoldenSet 목록 조회 (S2 ⑥: scope_id 필터 필수).
        
        Args:
            scope_id: Scope Profile ID
            offset: 페이징 오프셋
            limit: 페이징 제한
            domain: 도메인 필터 (optional)
            status: 상태 필터 (optional)
        
        Returns:
            (GoldenSet 목록, 전체 개수)
        """
        filters = [
            GoldenSetORM.scope_id == scope_id,  # S2 ⑥: ACL 필터
            GoldenSetORM.is_deleted == False,
        ]
        
        if domain:
            filters.append(GoldenSetORM.domain == domain)
        if status:
            filters.append(GoldenSetORM.status == status)
        
        # 전체 개수 조회
        count_query = select(func.count()).select_from(GoldenSetORM).where(and_(*filters))
        count_result = await self.db.execute(count_query)
        total = count_result.scalar() or 0
        
        # 페이징 조회
        query = (
            select(GoldenSetORM)
            .where(and_(*filters))
            .order_by(desc(GoldenSetORM.created_at))
            .offset(offset)
            .limit(limit)
        )
        
        result = await self.db.execute(query)
        orms = result.scalars().all()
        
        return [self._orm_to_domain(orm) for orm in orms], total
    
    # ==================== Update ====================
    
    async def update_golden_set(
        self,
        golden_set_id: str,
        scope_id: str,
        request: GoldenSetUpdateRequest,
        updated_by: str,
    ) -> Optional[GoldenSet]:
        """
        GoldenSet 메타데이터 수정 (버전 자동 증가).
        
        Args:
            golden_set_id: GoldenSet ID
            scope_id: Scope Profile ID (S2 ⑥)
            request: 수정 요청 DTO
            updated_by: 수정자 actor_id
        
        Returns:
            수정된 GoldenSet 또는 None (권한 없음)
        """
        orm = await self.get_by_id(golden_set_id, scope_id, include_items=True)
        if orm is None:
            return None
        
        orm_record = await self.db.get(GoldenSetORM, golden_set_id)
        if orm_record is None:
            return None
        
        # 필드 수정
        if request.name is not None:
            orm_record.name = request.name
        if request.description is not None:
            orm_record.description = request.description
        if request.domain is not None:
            orm_record.domain = request.domain.value
        if request.status is not None:
            orm_record.status = request.status.value
        if request.metadata is not None:
            orm_record.metadata = request.metadata
        
        # 버전 증가
        orm_record.version += 1
        orm_record.updated_at = datetime.utcnow()
        orm_record.updated_by = updated_by
        
        self.db.add(orm_record)
        await self.db.flush()
        
        # 버전 이력 기록
        items = await self.get_golden_items(golden_set_id, scope_id)
        await self._create_version_snapshot(
            golden_set_id,
            orm_record.version,
            orm_record,
            items
        )
        
        await self.db.commit()
        await self.db.refresh(orm_record)
        return self._orm_to_domain(orm_record)
    
    # ==================== Delete ====================
    
    async def soft_delete_golden_set(
        self,
        golden_set_id: str,
        scope_id: str,
    ) -> bool:
        """
        Soft delete: GoldenSet 및 모든 items를 비활성화.
        
        Args:
            golden_set_id: GoldenSet ID
            scope_id: Scope Profile ID (S2 ⑥)
        
        Returns:
            성공 여부
        """
        # 존재 확인
        orm = await self.get_by_id(golden_set_id, scope_id)
        if orm is None:
            return False
        
        now = datetime.utcnow()
        
        # GoldenSet soft delete
        stmt = (
            update(GoldenSetORM)
            .where(GoldenSetORM.id == golden_set_id)
            .values(is_deleted=True, deleted_at=now)
        )
        await self.db.execute(stmt)
        
        # 모든 GoldenItem soft delete
        stmt = (
            update(GoldenItemORM)
            .where(GoldenItemORM.golden_set_id == golden_set_id)
            .values(is_deleted=True, deleted_at=now)
        )
        await self.db.execute(stmt)
        
        await self.db.commit()
        return True
    
    # ==================== Version Management ====================
    
    async def get_version_history(
        self,
        golden_set_id: str,
        scope_id: str,
    ) -> List[Dict[str, Any]]:
        """
        GoldenSet 버전 이력 조회 (S2 ⑥: scope_id 확인).
        
        Args:
            golden_set_id: GoldenSet ID
            scope_id: Scope Profile ID
        
        Returns:
            버전 정보 리스트
        """
        # GoldenSet 접근 권한 확인
        orm = await self.get_by_id(golden_set_id, scope_id)
        if orm is None:
            return []
        
        query = (
            select(GoldenSetVersionORM)
            .where(GoldenSetVersionORM.golden_set_id == golden_set_id)
            .order_by(desc(GoldenSetVersionORM.version))
        )
        
        result = await self.db.execute(query)
        versions = result.scalars().all()
        
        return [
            {
                "version": v.version,
                "created_at": v.created_at.isoformat(),
                "created_by": v.created_by,
                "item_count": len(v.items_snapshot),
            }
            for v in versions
        ]
    
    async def get_version_snapshot(
        self,
        golden_set_id: str,
        scope_id: str,
        version: int,
    ) -> Optional[Dict[str, Any]]:
        """특정 버전의 스냅샷 조회."""
        # 접근 권한 확인
        orm = await self.get_by_id(golden_set_id, scope_id)
        if orm is None:
            return None
        
        query = select(GoldenSetVersionORM).where(
            and_(
                GoldenSetVersionORM.golden_set_id == golden_set_id,
                GoldenSetVersionORM.version == version,
            )
        )
        
        result = await self.db.execute(query)
        version_orm = result.scalar_one_or_none()
        
        if version_orm is None:
            return None
        
        return {
            "version": version_orm.version,
            "name": version_orm.name,
            "description": version_orm.description,
            "domain": version_orm.domain,
            "status": version_orm.status,
            "metadata": version_orm.metadata,
            "items": version_orm.items_snapshot,
            "created_at": version_orm.created_at.isoformat(),
            "created_by": version_orm.created_by,
        }
    
    # ==================== Helper Methods ====================
    
    async def _create_version_snapshot(
        self,
        golden_set_id: str,
        version: int,
        orm: GoldenSetORM,
        items: List[GoldenItem],
    ) -> None:
        """버전 스냅샷 생성 및 저장."""
        items_snapshot = [
            {
                "id": item.id,
                "version": item.version,
                "question": item.question,
                "expected_answer": item.expected_answer,
                "notes": item.notes,
            }
            for item in items
        ]
        
        version_orm = GoldenSetVersionORM(
            id=str(uuid4()),
            golden_set_id=golden_set_id,
            version=version,
            name=orm.name,
            description=orm.description,
            domain=orm.domain,
            status=orm.status,
            metadata=orm.metadata,
            items_snapshot=items_snapshot,
            created_at=datetime.utcnow(),
            created_by=orm.created_by if version == 1 else orm.updated_by or orm.created_by,
        )
        
        self.db.add(version_orm)
    
    def _orm_to_domain(
        self,
        orm: GoldenSetORM,
        include_items: bool = False,
    ) -> GoldenSet:
        """ORM -> 도메인 모델 변환."""
        items = None
        if include_items:
            items = [
                self._item_orm_to_domain(item_orm)
                for item_orm in (orm.items or [])
                if not item_orm.is_deleted
            ]
        
        return GoldenSet(
            id=orm.id,
            scope_id=orm.scope_id,
            name=orm.name,
            description=orm.description,
            domain=orm.domain,
            status=orm.status,
            version=orm.version,
            items=items,
            metadata=orm.metadata or {},
            created_at=orm.created_at,
            created_by=orm.created_by,
            updated_at=orm.updated_at,
            updated_by=orm.updated_by,
            deleted_at=orm.deleted_at,
            is_deleted=orm.is_deleted,
        )
    
    def _item_orm_to_domain(self, orm: GoldenItemORM) -> GoldenItem:
        """GoldenItemORM -> 도메인 모델 변환."""
        from app.models.golden_set import SourceRef, Citation5Tuple
        
        source_docs = [
            SourceRef(**doc)
            for doc in (orm.expected_source_docs or [])
        ]
        citations = [
            Citation5Tuple(**cite)
            for cite in (orm.expected_citations or [])
        ]
        
        return GoldenItem(
            id=orm.id,
            golden_set_id=orm.golden_set_id,
            version=orm.version,
            question=orm.question,
            expected_answer=orm.expected_answer,
            expected_source_docs=source_docs,
            expected_citations=citations,
            notes=orm.notes,
            created_at=orm.created_at,
            created_by=orm.created_by,
            updated_at=orm.updated_at,
            updated_by=orm.updated_by,
        )


class GoldenItemRepository:
    """GoldenItem 데이터 접근 계층."""
    
    def __init__(self, db_session: AsyncSession):
        self.db = db_session
        self.golden_set_repo = GoldenSetRepository(db_session)
    
    async def create_item(
        self,
        golden_set_id: str,
        scope_id: str,
        request: GoldenItemCreateRequest,
        created_by: str,
    ) -> Optional[GoldenItem]:
        """
        GoldenSet 내에 새로운 항목 추가.
        
        Args:
            golden_set_id: Parent GoldenSet ID
            scope_id: Scope Profile ID (S2 ⑥)
            request: 생성 요청 DTO
            created_by: 생성자 actor_id
        
        Returns:
            생성된 GoldenItem 또는 None (권한 없음)
        """
        # 부모 GoldenSet 확인
        parent = await self.golden_set_repo.get_by_id(golden_set_id, scope_id)
        if parent is None:
            return None
        
        now = datetime.utcnow()
        item_id = str(uuid4())
        
        orm = GoldenItemORM(
            id=item_id,
            golden_set_id=golden_set_id,
            version=1,
            question=request.question,
            expected_answer=request.expected_answer,
            expected_source_docs=[doc.dict() for doc in request.expected_source_docs],
            expected_citations=[cite.dict() for cite in request.expected_citations],
            notes=request.notes,
            created_at=now,
            created_by=created_by,
            updated_at=now,
            updated_by=None,
        )
        
        self.db.add(orm)
        await self.db.flush()
        
        # GoldenSet 버전 증가
        parent_orm = await self.db.get(GoldenSetORM, golden_set_id)
        parent_orm.version += 1
        parent_orm.updated_at = now
        parent_orm.updated_by = created_by
        self.db.add(parent_orm)
        await self.db.flush()
        
        # 버전 이력 기록
        items = await self.list_items(golden_set_id, scope_id)
        items.append(self._orm_to_domain(orm))
        await self.golden_set_repo._create_version_snapshot(
            golden_set_id,
            parent_orm.version,
            parent_orm,
            items
        )
        
        await self.db.commit()
        return self._orm_to_domain(orm)
    
    async def get_item_by_id(
        self,
        item_id: str,
        scope_id: str,
    ) -> Optional[GoldenItem]:
        """GoldenItem 조회 (scope_id로 접근 권한 확인)."""
        query = select(GoldenItemORM).where(
            and_(
                GoldenItemORM.id == item_id,
                GoldenItemORM.is_deleted == False,
            )
        )
        
        result = await self.db.execute(query)
        orm = result.scalar_one_or_none()
        
        if orm is None:
            return None
        
        # Parent GoldenSet의 scope_id로 접근 권한 확인 (S2 ⑥)
        parent = await self.golden_set_repo.get_by_id(orm.golden_set_id, scope_id)
        if parent is None:
            return None
        
        return self._orm_to_domain(orm)
    
    async def list_items(
        self,
        golden_set_id: str,
        scope_id: str,
    ) -> List[GoldenItem]:
        """GoldenSet 내 모든 활성 항목 조회."""
        # 부모 접근 권한 확인 (S2 ⑥)
        parent = await self.golden_set_repo.get_by_id(golden_set_id, scope_id)
        if parent is None:
            return []
        
        query = select(GoldenItemORM).where(
            and_(
                GoldenItemORM.golden_set_id == golden_set_id,
                GoldenItemORM.is_deleted == False,
            )
        ).order_by(GoldenItemORM.created_at)
        
        result = await self.db.execute(query)
        orms = result.scalars().all()
        
        return [self._orm_to_domain(orm) for orm in orms]
    
    async def update_item(
        self,
        item_id: str,
        scope_id: str,
        request: GoldenItemUpdateRequest,
        updated_by: str,
    ) -> Optional[GoldenItem]:
        """GoldenItem 수정 (버전 자동 증가)."""
        item = await self.get_item_by_id(item_id, scope_id)
        if item is None:
            return None
        
        orm = await self.db.get(GoldenItemORM, item_id)
        if orm is None:
            return None
        
        # 필드 수정
        if request.question is not None:
            orm.question = request.question
        if request.expected_answer is not None:
            orm.expected_answer = request.expected_answer
        if request.expected_source_docs is not None:
            orm.expected_source_docs = [doc.dict() for doc in request.expected_source_docs]
        if request.expected_citations is not None:
            orm.expected_citations = [cite.dict() for cite in request.expected_citations]
        if request.notes is not None:
            orm.notes = request.notes
        
        # 버전 증가
        orm.version += 1
        orm.updated_at = datetime.utcnow()
        orm.updated_by = updated_by
        
        self.db.add(orm)
        await self.db.flush()
        
        # Parent GoldenSet 버전도 증가
        parent_orm = await self.db.get(GoldenSetORM, orm.golden_set_id)
        parent_orm.version += 1
        parent_orm.updated_at = datetime.utcnow()
        parent_orm.updated_by = updated_by
        self.db.add(parent_orm)
        await self.db.flush()
        
        # 버전 이력 기록
        items = await self.list_items(orm.golden_set_id, scope_id)
        await self.golden_set_repo._create_version_snapshot(
            orm.golden_set_id,
            parent_orm.version,
            parent_orm,
            items
        )
        
        await self.db.commit()
        await self.db.refresh(orm)
        return self._orm_to_domain(orm)
    
    async def soft_delete_item(
        self,
        item_id: str,
        scope_id: str,
    ) -> bool:
        """GoldenItem soft delete."""
        item = await self.get_item_by_id(item_id, scope_id)
        if item is None:
            return False
        
        now = datetime.utcnow()
        
        stmt = (
            update(GoldenItemORM)
            .where(GoldenItemORM.id == item_id)
            .values(is_deleted=True, deleted_at=now)
        )
        await self.db.execute(stmt)
        
        # Parent GoldenSet 버전 증가
        orm = await self.db.get(GoldenItemORM, item_id)
        parent_orm = await self.db.get(GoldenSetORM, orm.golden_set_id)
        parent_orm.version += 1
        parent_orm.updated_at = now
        self.db.add(parent_orm)
        await self.db.flush()
        
        # 버전 이력 기록
        items = await self.list_items(orm.golden_set_id, scope_id)
        await self.golden_set_repo._create_version_snapshot(
            orm.golden_set_id,
            parent_orm.version,
            parent_orm,
            items
        )
        
        await self.db.commit()
        return True
    
    def _orm_to_domain(self, orm: GoldenItemORM) -> GoldenItem:
        """ORM -> 도메인 모델 변환."""
        return self.golden_set_repo._item_orm_to_domain(orm)
```

### 4-5. 단위 테스트

**파일:** `/backend/tests/unit/models/test_golden_set.py`

```python
"""Golden Set 도메인 모델 및 리포지토리 단위 테스트."""

import pytest
from datetime import datetime
from uuid import uuid4

from app.models.golden_set import (
    GoldenSet, GoldenItem, SourceRef, Citation5Tuple,
    GoldenSetCreateRequest, GoldenSetUpdateRequest,
    GoldenItemCreateRequest, GoldenItemUpdateRequest,
    GoldenSetDomain, GoldenSetStatus,
)
from app.repositories.golden_set_repository import (
    GoldenSetRepository, GoldenItemRepository
)


class TestGoldenItemModel:
    """GoldenItem 도메인 모델 테스트."""
    
    def test_create_valid_golden_item(self):
        """유효한 GoldenItem 생성."""
        source_docs = [SourceRef(
            document_id="doc-123",
            version_id=1,
            node_id="node-456"
        )]
        citations = [Citation5Tuple(
            document_id="doc-123",
            version_id=1,
            node_id="node-456",
            span_offset=(0, 50),
            content_hash="abc123..."
        )]
        
        item = GoldenItem(
            id="item-789",
            golden_set_id="set-001",
            version=1,
            question="What is Docker?",
            expected_answer="Docker is a containerization platform.",
            expected_source_docs=source_docs,
            expected_citations=citations,
            notes="From official docs",
            created_at=datetime.utcnow(),
            created_by="user-123",
            updated_at=datetime.utcnow(),
            updated_by=None,
        )
        
        assert item.id == "item-789"
        assert item.version == 1
        assert len(item.expected_source_docs) == 1
        assert len(item.expected_citations) == 1
    
    def test_golden_item_requires_source_docs(self):
        """GoldenItem은 최소 하나의 source_doc가 필요."""
        with pytest.raises(ValueError, match="must contain at least one"):
            GoldenItem(
                id="item-789",
                golden_set_id="set-001",
                version=1,
                question="What is Docker?",
                expected_answer="Docker is a containerization platform.",
                expected_source_docs=[],  # Empty!
                expected_citations=[],
                created_at=datetime.utcnow(),
                created_by="user-123",
                updated_at=datetime.utcnow(),
                updated_by=None,
            )


class TestGoldenSetModel:
    """GoldenSet 도메인 모델 테스트."""
    
    def test_create_valid_golden_set(self):
        """유효한 GoldenSet 생성."""
        gs = GoldenSet(
            id="set-001",
            scope_id="scope-123",  # S2 ⑥
            name="Technical Docs",
            description="Tech documentation RAG",
            domain=GoldenSetDomain.TECHNICAL_GUIDE,
            status=GoldenSetStatus.DRAFT,
            version=1,
            items=[],
            metadata={"owner": "platform-team"},
            created_at=datetime.utcnow(),
            created_by="user-123",
            updated_at=datetime.utcnow(),
            updated_by=None,
            deleted_at=None,
            is_deleted=False,
        )
        
        assert gs.id == "set-001"
        assert gs.scope_id == "scope-123"
        assert gs.status == GoldenSetStatus.DRAFT
        assert gs.version == 1
        assert not gs.is_deleted
    
    def test_golden_set_soft_delete(self):
        """GoldenSet soft delete 필드 관리."""
        gs = GoldenSet(
            id="set-001",
            scope_id="scope-123",
            name="Technical Docs",
            domain=GoldenSetDomain.TECHNICAL_GUIDE,
            status=GoldenSetStatus.PUBLISHED,
            version=1,
            items=[],
            metadata={},
            created_at=datetime.utcnow(),
            created_by="user-123",
            updated_at=datetime.utcnow(),
            updated_by=None,
            deleted_at=None,
            is_deleted=False,
        )
        
        # Soft delete 후 상태 변경
        delete_time = datetime.utcnow()
        gs.deleted_at = delete_time
        gs.is_deleted = True
        
        assert gs.is_deleted is True
        assert gs.deleted_at == delete_time


class TestGoldenSetCreateRequest:
    """GoldenSetCreateRequest DTO 테스트."""
    
    def test_create_request_valid(self):
        """유효한 생성 요청."""
        req = GoldenSetCreateRequest(
            name="Test Set",
            description="A test golden set",
            domain=GoldenSetDomain.POLICY,
            metadata={"tags": ["policy", "compliance"]}
        )
        
        assert req.name == "Test Set"
        assert req.domain == GoldenSetDomain.POLICY
        assert req.metadata["tags"] == ["policy", "compliance"]
    
    def test_create_request_defaults(self):
        """기본값 설정."""
        req = GoldenSetCreateRequest(name="Test Set")
        
        assert req.domain == GoldenSetDomain.CUSTOM
        assert req.metadata == {}


class TestGoldenItemCreateRequest:
    """GoldenItemCreateRequest DTO 테스트."""
    
    def test_create_item_request_valid(self):
        """유효한 항목 생성 요청."""
        req = GoldenItemCreateRequest(
            question="What is Kubernetes?",
            expected_answer="Kubernetes is an orchestration platform.",
            expected_source_docs=[
                SourceRef(document_id="doc-1", version_id=1, node_id="node-1")
            ],
            expected_citations=[
                Citation5Tuple(
                    document_id="doc-1",
                    version_id=1,
                    node_id="node-1",
                    span_offset=(0, 40),
                    content_hash="xyz789..."
                )
            ],
            notes="From Kubernetes docs"
        )
        
        assert req.question == "What is Kubernetes?"
        assert len(req.expected_source_docs) == 1
        assert len(req.expected_citations) == 1


@pytest.mark.asyncio
class TestGoldenSetRepository:
    """GoldenSetRepository 테스트 (DB 통합)."""
    
    async def test_create_golden_set(self, async_db_session):
        """GoldenSet 생성 테스트."""
        repo = GoldenSetRepository(async_db_session)
        
        req = GoldenSetCreateRequest(
            name="Test Set",
            description="Test description",
            domain=GoldenSetDomain.TECHNICAL_GUIDE,
        )
        
        gs = await repo.create_golden_set(
            scope_id="scope-123",
            request=req,
            created_by="user-123",
        )
        
        assert gs is not None
        assert gs.name == "Test Set"
        assert gs.scope_id == "scope-123"
        assert gs.version == 1
        assert not gs.is_deleted
    
    async def test_get_by_id_with_scope_acl(self, async_db_session):
        """scope_id 기반 ACL 필터 테스트 (S2 ⑥)."""
        repo = GoldenSetRepository(async_db_session)
        
        # scope-123으로 생성
        req = GoldenSetCreateRequest(name="Scope-123 Set")
        gs1 = await repo.create_golden_set(
            scope_id="scope-123",
            request=req,
            created_by="user-123",
        )
        
        # scope-123으로 조회 (성공)
        found = await repo.get_by_id(gs1.id, scope_id="scope-123")
        assert found is not None
        assert found.id == gs1.id
        
        # scope-456으로 조회 (실패 - 다른 scope)
        found_other = await repo.get_by_id(gs1.id, scope_id="scope-456")
        assert found_other is None  # S2 ⑥: ACL 필터로 접근 거부
    
    async def test_list_by_scope(self, async_db_session):
        """Scope별 목록 조회 (S2 ⑥)."""
        repo = GoldenSetRepository(async_db_session)
        
        # 3개 생성
        for i in range(3):
            req = GoldenSetCreateRequest(
                name=f"Set {i}",
                domain=GoldenSetDomain.TECHNICAL_GUIDE if i < 2 else GoldenSetDomain.POLICY,
            )
            await repo.create_golden_set(
                scope_id="scope-123",
                request=req,
                created_by="user-123",
            )
        
        # scope-123에서 모두 조회
        sets, total = await repo.list_by_scope(scope_id="scope-123")
        assert total == 3
        assert len(sets) == 3
        
        # scope-456에서 조회 (아무것도 없음)
        sets_other, total_other = await repo.list_by_scope(scope_id="scope-456")
        assert total_other == 0
    
    async def test_soft_delete(self, async_db_session):
        """Soft delete 테스트."""
        repo = GoldenSetRepository(async_db_session)
        
        req = GoldenSetCreateRequest(name="Deletable Set")
        gs = await repo.create_golden_set(
            scope_id="scope-123",
            request=req,
            created_by="user-123",
        )
        
        # 삭제
        success = await repo.soft_delete_golden_set(gs.id, scope_id="scope-123")
        assert success is True
        
        # 다시 조회하면 없음 (is_deleted=True 필터)
        found = await repo.get_by_id(gs.id, scope_id="scope-123")
        assert found is None
    
    async def test_update_version_increment(self, async_db_session):
        """수정 시 버전 자동 증가 테스트."""
        repo = GoldenSetRepository(async_db_session)
        
        req = GoldenSetCreateRequest(name="Original Name")
        gs = await repo.create_golden_set(
            scope_id="scope-123",
            request=req,
            created_by="user-123",
        )
        
        assert gs.version == 1
        
        # 수정
        update_req = GoldenSetUpdateRequest(name="Updated Name")
        updated = await repo.update_golden_set(
            gs.id,
            scope_id="scope-123",
            request=update_req,
            updated_by="user-456",
        )
        
        assert updated is not None
        assert updated.version == 2
        assert updated.name == "Updated Name"
        assert updated.updated_by == "user-456"
    
    async def test_version_history(self, async_db_session):
        """버전 이력 조회 테스트."""
        repo = GoldenSetRepository(async_db_session)
        
        req = GoldenSetCreateRequest(name="Test Set")
        gs = await repo.create_golden_set(
            scope_id="scope-123",
            request=req,
            created_by="user-123",
        )
        
        # 3번 수정해서 version 1 → 2 → 3 → 4
        for i in range(3):
            update_req = GoldenSetUpdateRequest(name=f"Name v{i+2}")
            await repo.update_golden_set(
                gs.id,
                scope_id="scope-123",
                request=update_req,
                updated_by="user-456",
            )
        
        # 이력 조회
        history = await repo.get_version_history(gs.id, scope_id="scope-123")
        assert len(history) == 4  # version 1, 2, 3, 4
        assert history[0]["version"] == 4  # 최신부터 역순


@pytest.mark.asyncio
class TestGoldenItemRepository:
    """GoldenItemRepository 테스트."""
    
    async def test_create_item(self, async_db_session):
        """GoldenItem 생성 테스트."""
        set_repo = GoldenSetRepository(async_db_session)
        item_repo = GoldenItemRepository(async_db_session)
        
        # GoldenSet 먼저 생성
        gs_req = GoldenSetCreateRequest(name="Test Set")
        gs = await set_repo.create_golden_set(
            scope_id="scope-123",
            request=gs_req,
            created_by="user-123",
        )
        
        # GoldenItem 생성
        item_req = GoldenItemCreateRequest(
            question="What is Docker?",
            expected_answer="Docker is a platform.",
            expected_source_docs=[
                SourceRef(document_id="doc-1", version_id=1, node_id="node-1")
            ],
        )
        
        item = await item_repo.create_item(
            golden_set_id=gs.id,
            scope_id="scope-123",
            request=item_req,
            created_by="user-123",
        )
        
        assert item is not None
        assert item.question == "What is Docker?"
        assert item.golden_set_id == gs.id
        
        # Parent GoldenSet 버전 증가 확인
        updated_gs = await set_repo.get_by_id(gs.id, scope_id="scope-123")
        assert updated_gs.version == 2  # 항목 추가로 버전 증가
    
    async def test_update_item_version_increment(self, async_db_session):
        """GoldenItem 수정 시 버전 증가."""
        set_repo = GoldenSetRepository(async_db_session)
        item_repo = GoldenItemRepository(async_db_session)
        
        # Setup
        gs_req = GoldenSetCreateRequest(name="Test Set")
        gs = await set_repo.create_golden_set(
            scope_id="scope-123",
            request=gs_req,
            created_by="user-123",
        )
        
        item_req = GoldenItemCreateRequest(
            question="Q1",
            expected_answer="A1",
            expected_source_docs=[
                SourceRef(document_id="doc-1", version_id=1, node_id="node-1")
            ],
        )
        item = await item_repo.create_item(
            golden_set_id=gs.id,
            scope_id="scope-123",
            request=item_req,
            created_by="user-123",
        )
        
        assert item.version == 1
        
        # Update
        update_req = GoldenItemUpdateRequest(question="Q1 Updated")
        updated_item = await item_repo.update_item(
            item_id=item.id,
            scope_id="scope-123",
            request=update_req,
            updated_by="user-456",
        )
        
        assert updated_item is not None
        assert updated_item.version == 2
        assert updated_item.question == "Q1 Updated"
```

## 5. 산출물

1. **Pydantic 도메인 모델** (`/backend/app/models/golden_set.py`)
   - GoldenSet, GoldenItem, SourceRef, Citation5Tuple
   - Pydantic 스키마 (Create/Update/Response DTO)
   - 버전 정보 및 Diff 모델

2. **SQLAlchemy ORM 모델** (`/backend/app/db/models/golden_set_orm.py`)
   - GoldenSetORM, GoldenItemORM, GoldenSetVersionORM
   - 인덱스 및 제약 조건
   - Soft delete 및 버전 관리 지원

3. **Alembic 마이그레이션** (`/backend/alembic/versions/2025_01_15_001_create_golden_sets.py`)
   - golden_sets, golden_items, golden_set_versions 테이블
   - 외래키, 인덱스, 유니크 제약 조건

4. **Repository 계층** (`/backend/app/repositories/golden_set_repository.py`)
   - GoldenSetRepository (CRUD, 버전 관리, ACL 필터)
   - GoldenItemRepository (CRUD, 부모 버전 관리)
   - S2 원칙 ⑥ Scope Profile 기반 ACL 적용
   - S2 원칙 ⑦ 폐쇄망 환경 지원

5. **단위 테스트** (`/backend/tests/unit/models/test_golden_set.py`)
   - 도메인 모델 검증 테스트
   - Repository CRUD 및 버전 관리 테스트
   - Scope Profile 기반 ACL 필터 정확성 테스트
   - Soft delete 동작 확인 테스트

## 6. 완료 기준

1. GoldenSet 및 GoldenItem 도메인 모델이 정의되었는가?
2. Pydantic 스키마 (Request/Response DTO)가 완전한가?
3. SQLAlchemy ORM이 soft delete 및 버전 관리를 지원하는가?
4. Alembic 마이그레이션이 정상적으로 생성되는가?
5. GoldenSetRepository 및 GoldenItemRepository가 모든 CRUD 작업을 수행하는가?
6. S2 원칙 ⑥: Scope Profile 기반 ACL 필터가 모든 조회/쓰기 API에 적용되어 있는가?
7. S2 원칙 ⑦: 폐쇄망 환경에서 외부 의존 없이 로컬 DB만 사용하는가?
8. 버전 관리: GoldenSet 또는 GoldenItem 수정 시 버전이 자동 증가하는가?
9. Citation 5-tuple (Phase 2)과 호환성이 확인되었는가?
10. Document/Node 도메인 모델과의 참조 관계가 정합한가?
11. 모든 단위 테스트가 통과하는가?
12. mypy 타입 검사를 통과하는가?

## 7. 작업 지침

### 지침 7-1. Scope Profile 기반 ACL (S2 ⑥)

모든 조회 및 쓰기 API에서 `scope_id`를 필수 필터로 적용한다. 요청자의 scope_id와 리소스의 scope_id가 일치해야만 접근을 허용한다. 예:

```python
query = select(GoldenSetORM).where(
    and_(
        GoldenSetORM.id == golden_set_id,
        GoldenSetORM.scope_id == scope_id,  # 필수 필터
        GoldenSetORM.is_deleted == False,
    )
)
```

### 지침 7-2. 폐쇄망 환경 지원 (S2 ⑦)

Repository에서 외부 API 호출을 하지 않는다. 모든 데이터는 로컬 PostgreSQL DB에서만 조회/저장한다. 향후 LLM 평가나 embedding이 필요하면, Service 계층에서 이를 호출하고, 결과를 Repository를 통해 저장한다.

### 지침 7-3. Citation 5-tuple 호환성

expected_citations의 구조가 Phase 2 Citation 도메인 모델과 정확히 일치해야 한다:
```python
class Citation5Tuple(BaseModel):
    document_id: str
    version_id: int
    node_id: str
    span_offset: tuple[int, int]  # (start, end)
    content_hash: str
```

### 지침 7-4. Soft Delete 정책

영구 삭제(DELETE 쿼리)는 사용하지 않는다. 대신 is_deleted=True, deleted_at=timestamp로 표시만 한다. 조회 쿼리는 항상 `is_deleted == False` 필터를 적용한다.

### 지침 7-5. 버전 관리 자동화

GoldenSet의 version은 다음 경우에 자동으로 증가한다:
1. GoldenSet 메타데이터 수정 (PUT /api/v1/golden-sets/{id})
2. GoldenItem 추가/수정/삭제

각 버전 변화 시 golden_set_versions 테이블에 스냅샷을 기록한다 (Document 버전 관리와 동일 패턴).

### 지침 7-6. 감사 로그 연계 (S2 ⑤)

created_by, updated_by 필드에 actor_id (사용자 ID 또는 에이전트 ID)를 기록한다. 향후 Task 7-2 평가 실행 시 actor_type="agent"로 기록할 수 있도록 설계한다.

### 지침 7-7. 관계 정합성 검증

expected_source_docs의 document_id, version_id, node_id가 실제 Document/Node 레코드와 존재해야 한다는 제약은 Task 7-2의 평가 러너에서 검증한다 (FK 제약 없음). 이는 평가 시점의 유효성만 확인하면 되기 때문이다.

### 지침 7-8. 페이징 및 정렬

list_by_scope 등 대량 데이터 조회는 항상 페이징을 지원한다 (offset, limit). 기본 정렬은 created_at DESC (최신순).

### 지침 7-9. 에러 처리

Repository 계층에서:
- 접근 권한 없음 (scope_id 불일치): Optional로 None 반환
- 리소스 미존재: Optional로 None 반환
- DB 에러 (FK 위반 등): 상위로 예외 전파 (Task 7-2 Service에서 처리)

### 지침 7-10. 타입 안정성

모든 메서드에 명확한 타입 힌트를 제공한다. mypy strict 모드를 만족해야 한다.
