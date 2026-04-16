# Task 3-1. Conversation / Turn / Message 도메인 모델 및 DB 스키마 설계

## 1. 작업 목적

멀티턴 대화를 1급 도메인 객체로 정의하기 위해 **Conversation, Turn, Message 도메인 모델**을 설계하고, Document와 동일 수준의 권한 관리(ACL)와 감사 이벤트를 적용하는 구조를 구축한다. 이를 통해 사용자의 대화 이력을 검색·조회·관리할 수 있는 기반을 마련한다.

## 2. 작업 범위

### 포함 범위

1. **Conversation 도메인 모델 (SQLAlchemy ORM)**
   - 필드: `id`, `owner_id`, `organization_id`, `created_at`, `updated_at`, `title`, `status`, `metadata`, `retention_days`, `expires_at`, `deleted_at` (soft delete)
   - 상태: `active`, `archived`, `expired`, `deleted`
   - ACL 적용: Document와 동일 수준 (owner, organization, access_level)
   - 감사: 생성/수정/삭제 이벤트 기록

2. **Turn 도메인 모델 (SQLAlchemy ORM)**
   - 필드: `id`, `conversation_id`, `turn_number`, `created_at`, `user_message`, `assistant_response`, `retrieval_metadata` (JSONB)
   - `retrieval_metadata`: citation 5-tuple 리스트, 검색어 재작성, 컨텍스트 윈도우 상태
   - 외래키: conversations(conversation_id)
   - 유니크 제약: (conversation_id, turn_number)

3. **Message 도메인 모델 (SQLAlchemy ORM)**
   - 필드: `id`, `turn_id`, `role` (user/assistant/system), `content`, `created_at`, `metadata` (JSONB)
   - metadata: 토큰 카운트, 모델명, 생성 비용 등
   - 외래키: turns(turn_id)

4. **DB 스키마 및 마이그레이션 (Alembic)**
   - conversations 테이블: 기본 CRUD 컬럼 + ACL 필드
   - turns 테이블: conversation 참조 + 메타데이터
   - messages 테이블: turn 참조 + 역할별 메시지
   - 인덱스: (conversation_id, turn_number), (owner_id, created_at), (expires_at), (organization_id, status)
   - 제약: NOT NULL 필드, CHECK 제약 (status 값), Foreign Key 캐스케이드

5. **Soft Delete 및 보존 정책 구조**
   - `deleted_at` 필드: NULL이면 활성, 값이 있으면 삭제됨
   - `expires_at`: 자동 만료 시간
   - 쿼리 시 기본적으로 `deleted_at IS NULL` 필터 적용

6. **감사 로그 통합 구조**
   - Conversation 생성/수정/삭제 시 audit_log 테이블에 기록
   - 필드: `event_id`, `entity_type` (conversation), `entity_id`, `action` (create/update/delete), `actor_id`, `actor_type` (user/agent), `timestamp`, `changes` (JSONB)
   - S2 원칙 ⑥: actor_type 필드 필수 (사용자와 에이전트 구분)

### 제외 범위

- PII 자동 만료 배치 작업 (Task 3-3)
- 대화 CRUD API 엔드포인트 (Task 3-2)
- 세션 기반 RAG 통합 (FG3.2)
- UI 구현 (FG3.3)

## 3. 선행 조건

- Phase 0, Phase 1, Phase 2 완료
- PostgreSQL 설치 및 연결 가능
- SQLAlchemy 2.0+ 설치
- Alembic 초기화 완료 (`/backend/alembic` 디렉토리 존재)
- Document 도메인 모델의 ACL 구조 분석 완료 (`/backend/app/models/document.py`)

## 4. 주요 작업 항목

### 4-1. Conversation 도메인 모델

**파일:** `/backend/app/models/conversation.py`

```python
from __future__ import annotations

from datetime import datetime
from typing import Optional
from uuid import UUID, uuid4

from sqlalchemy import (
    Column, String, DateTime, Text, Integer, ForeignKey, 
    Index, CheckConstraint, Text as SQLText
)
from sqlalchemy.dialects.postgresql import UUID as PGUUID, JSONB
from sqlalchemy.orm import relationship, declarative_base, mapped_column
from sqlalchemy.sql import func

Base = declarative_base()


class Conversation(Base):
    """사용자의 멀티턴 대화 세션 도메인 모델.
    
    Attributes:
        id: 고유 식별자 (UUID)
        owner_id: 소유자 사용자 ID (UUID)
        organization_id: 소속 조직 ID (UUID)
        created_at: 생성 시간
        updated_at: 마지막 수정 시간
        title: 대화 제목 (사용자 입력 또는 첫 질문 자동 생성)
        status: 상태 (active, archived, expired, deleted)
        metadata: 확장 가능한 메타데이터 (JSONB)
            - category: 대화 카테고리
            - tags: 태그 리스트
            - language: 사용 언어 (기본: ko)
            - domain: 도메인 정보
        retention_days: 보존 기간 (일), 조직 기본값에서 상속
        expires_at: 자동 만료 타임스탬프 (created_at + retention_days)
        deleted_at: soft delete 타임스탬프 (NULL이면 활성)
        access_level: ACL 레벨 (private, organization, public)
    """
    
    __tablename__ = "conversations"
    
    id = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    owner_id = mapped_column(PGUUID(as_uuid=True), nullable=False, index=True)
    organization_id = mapped_column(PGUUID(as_uuid=True), nullable=False, index=True)
    
    created_at = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)
    
    title = mapped_column(String(256), nullable=False)
    status = mapped_column(String(32), default="active", nullable=False)
    metadata = mapped_column(JSONB, default=dict, nullable=False)
    
    retention_days = mapped_column(Integer, default=90, nullable=False)
    expires_at = mapped_column(DateTime(timezone=True), nullable=True, index=True)
    deleted_at = mapped_column(DateTime(timezone=True), nullable=True)
    
    access_level = mapped_column(String(32), default="private", nullable=False)  # private, organization, public
    
    # 관계
    turns = relationship("Turn", back_populates="conversation", cascade="all, delete-orphan", lazy="selectin")
    
    # 인덱스
    __table_args__ = (
        Index("ix_conversations_owner_created", "owner_id", "created_at"),
        Index("ix_conversations_organization_status", "organization_id", "status"),
        Index("ix_conversations_expires_at", "expires_at"),
        CheckConstraint(
            "status IN ('active', 'archived', 'expired', 'deleted')",
            name="check_conversation_status"
        ),
        CheckConstraint(
            "access_level IN ('private', 'organization', 'public')",
            name="check_conversation_access_level"
        ),
    )
    
    def __repr__(self) -> str:
        return f"<Conversation(id={self.id}, title={self.title}, status={self.status})>"


class Turn(Base):
    """대화 내 하나의 턴 (사용자 질문 + AI 응답 쌍).
    
    Attributes:
        id: 고유 식별자 (UUID)
        conversation_id: 소속 대화 ID (UUID)
        turn_number: 대화 내 순서 (1부터 시작)
        created_at: 턴 생성 시간
        user_message: 사용자의 질문
        assistant_response: AI의 응답
        retrieval_metadata: 검색 관련 메타데이터 (JSONB)
            - citations: [{document_id, version_id, node_id, span_offset, content_hash}, ...]
            - query_original: 원문 쿼리
            - query_rewritten: 재작성된 쿼리
            - context_window_turns: [turn_id1, turn_id2, ...] (이 턴의 프롬프트에 포함된 이전 턴들)
            - retrieval_time_ms: 검색 소요 시간
    """
    
    __tablename__ = "turns"
    
    id = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    conversation_id = mapped_column(PGUUID(as_uuid=True), ForeignKey("conversations.id", ondelete="CASCADE"), nullable=False)
    
    turn_number = mapped_column(Integer, nullable=False)
    created_at = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    
    user_message = mapped_column(Text, nullable=False)
    assistant_response = mapped_column(Text, nullable=False)
    retrieval_metadata = mapped_column(JSONB, default=dict, nullable=False)
    
    # 관계
    conversation = relationship("Conversation", back_populates="turns")
    messages = relationship("Message", back_populates="turn", cascade="all, delete-orphan", lazy="selectin")
    
    # 유니크 제약: 같은 대화 내에서 turn_number는 고유
    __table_args__ = (
        Index("ix_turns_conversation_turn_number", "conversation_id", "turn_number", unique=True),
        Index("ix_turns_created_at", "created_at"),
    )
    
    def __repr__(self) -> str:
        return f"<Turn(id={self.id}, conversation_id={self.conversation_id}, turn_number={self.turn_number})>"


class Message(Base):
    """턴 내 세분화된 메시지 (시스템 프롬프트, 사용자 질문, AI 응답).
    
    Attributes:
        id: 고유 식별자 (UUID)
        turn_id: 소속 턴 ID (UUID)
        role: 메시지 역할 (user, assistant, system)
        content: 메시지 내용
        created_at: 메시지 생성 시간
        metadata: 추가 메타데이터 (JSONB)
            - token_count: 이 메시지의 토큰 수
            - model: 사용된 모델명 (assistant 메시지만 해당)
            - cost: 생성 비용 (폐쇄망 환경에서는 0)
    """
    
    __tablename__ = "messages"
    
    id = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    turn_id = mapped_column(PGUUID(as_uuid=True), ForeignKey("turns.id", ondelete="CASCADE"), nullable=False)
    
    role = mapped_column(String(32), nullable=False)  # user, assistant, system
    content = mapped_column(Text, nullable=False)
    created_at = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    metadata = mapped_column(JSONB, default=dict, nullable=False)
    
    # 관계
    turn = relationship("Turn", back_populates="messages")
    
    # 인덱스
    __table_args__ = (
        Index("ix_messages_turn_id", "turn_id"),
        CheckConstraint(
            "role IN ('user', 'assistant', 'system')",
            name="check_message_role"
        ),
    )
    
    def __repr__(self) -> str:
        return f"<Message(id={self.id}, turn_id={self.turn_id}, role={self.role})>"
```

### 4-2. 감사 로그 모델 (audit_log 통합)

**파일:** `/backend/app/models/audit_log.py` (확장 또는 기존 audit_log 활용)

```python
from __future__ import annotations

from datetime import datetime
from uuid import UUID, uuid4

from sqlalchemy import Column, String, DateTime, Text, ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID as PGUUID, JSONB
from sqlalchemy.orm import mapped_column
from sqlalchemy.sql import func

from app.models.base import Base  # 기존 Base 클래스 사용


class AuditLog(Base):
    """감시 이벤트 로그 (모든 도메인에 공통).
    
    Attributes:
        event_id: 이벤트 고유 ID (UUID)
        entity_type: 엔티티 타입 (conversation, document, user, etc.)
        entity_id: 엔티티 ID (UUID)
        action: 작업 종류 (create, update, delete, redact)
        actor_id: 작업 수행자 ID (UUID)
        actor_type: 작업 수행자 타입 (user, agent) — S2 원칙 ⑥
        timestamp: 이벤트 발생 시간
        changes: 변경 사항 (JSONB)
            - for update: {field_name: {old_value, new_value}, ...}
            - for delete: {reason: str, metadata: {...}}
    """
    
    __tablename__ = "audit_logs"
    
    event_id = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    entity_type = mapped_column(String(64), nullable=False)
    entity_id = mapped_column(PGUUID(as_uuid=True), nullable=False)
    action = mapped_column(String(32), nullable=False)  # create, update, delete, redact
    actor_id = mapped_column(PGUUID(as_uuid=True), nullable=False)
    actor_type = mapped_column(String(32), nullable=False)  # user, agent
    timestamp = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    changes = mapped_column(JSONB, default=dict, nullable=False)
    
    __table_args__ = (
        Index("ix_audit_logs_entity", "entity_type", "entity_id"),
        Index("ix_audit_logs_timestamp", "timestamp"),
        Index("ix_audit_logs_actor", "actor_id", "actor_type"),
    )
    
    def __repr__(self) -> str:
        return f"<AuditLog(event_id={self.event_id}, entity_type={self.entity_type}, action={self.action})>"
```

### 4-3. Alembic 마이그레이션 스크립트

**파일:** `/backend/alembic/versions/0003_create_conversation_domain.py`

```python
"""Create conversation, turn, message domain models.

Revision ID: 0003
Revises: 0002
Create Date: 2026-04-17 00:00:00.000000

"""

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# Revision identifiers
revision = "0003"
down_revision = "0002"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # Create conversations table
    op.create_table(
        "conversations",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("owner_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("organization_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.Column("title", sa.String(256), nullable=False),
        sa.Column("status", sa.String(32), server_default="active", nullable=False),
        sa.Column("metadata", postgresql.JSONB(astext_type=sa.Text()), server_default=sa.text("'{}'::jsonb"), nullable=False),
        sa.Column("retention_days", sa.Integer(), server_default="90", nullable=False),
        sa.Column("expires_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("deleted_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("access_level", sa.String(32), server_default="private", nullable=False),
        sa.PrimaryKeyConstraint("id"),
        sa.CheckConstraint("status IN ('active', 'archived', 'expired', 'deleted')", name="check_conversation_status"),
        sa.CheckConstraint("access_level IN ('private', 'organization', 'public')", name="check_conversation_access_level"),
    )
    op.create_index("ix_conversations_owner_created", "conversations", ["owner_id", "created_at"])
    op.create_index("ix_conversations_organization_status", "conversations", ["organization_id", "status"])
    op.create_index("ix_conversations_expires_at", "conversations", ["expires_at"])
    
    # Create turns table
    op.create_table(
        "turns",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("conversation_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("turn_number", sa.Integer(), nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.Column("user_message", sa.Text(), nullable=False),
        sa.Column("assistant_response", sa.Text(), nullable=False),
        sa.Column("retrieval_metadata", postgresql.JSONB(astext_type=sa.Text()), server_default=sa.text("'{}'::jsonb"), nullable=False),
        sa.ForeignKeyConstraint(["conversation_id"], ["conversations.id"], ondelete="CASCADE"),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index("ix_turns_conversation_turn_number", "turns", ["conversation_id", "turn_number"], unique=True)
    op.create_index("ix_turns_created_at", "turns", ["created_at"])
    
    # Create messages table
    op.create_table(
        "messages",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("turn_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("role", sa.String(32), nullable=False),
        sa.Column("content", sa.Text(), nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.Column("metadata", postgresql.JSONB(astext_type=sa.Text()), server_default=sa.text("'{}'::jsonb"), nullable=False),
        sa.ForeignKeyConstraint(["turn_id"], ["turns.id"], ondelete="CASCADE"),
        sa.PrimaryKeyConstraint("id"),
        sa.CheckConstraint("role IN ('user', 'assistant', 'system')", name="check_message_role"),
    )
    op.create_index("ix_messages_turn_id", "messages", ["turn_id"])


def downgrade() -> None:
    op.drop_index("ix_messages_turn_id", table_name="messages")
    op.drop_table("messages")
    
    op.drop_index("ix_turns_created_at", table_name="turns")
    op.drop_index("ix_turns_conversation_turn_number", table_name="turns")
    op.drop_table("turns")
    
    op.drop_index("ix_conversations_expires_at", table_name="conversations")
    op.drop_index("ix_conversations_organization_status", table_name="conversations")
    op.drop_index("ix_conversations_owner_created", table_name="conversations")
    op.drop_table("conversations")
```

### 4-4. Repository 패턴 (조회 계층)

**파일:** `/backend/app/repositories/conversation_repository.py`

```python
"""Conversation/Turn/Message 저장소 (조회/필터링 로직)."""

from __future__ import annotations

from typing import Optional, List, Dict, Any
from uuid import UUID
from datetime import datetime

from sqlalchemy import and_, or_, select, desc
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.conversation import Conversation, Turn, Message


class ConversationRepository:
    """Conversation 조회 및 필터링."""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def get_by_id(self, conversation_id: UUID) -> Optional[Conversation]:
        """ID로 대화 조회 (활성 대화만)."""
        stmt = select(Conversation).where(
            and_(
                Conversation.id == conversation_id,
                Conversation.deleted_at.is_(None)
            )
        )
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()
    
    async def list_by_owner(
        self,
        owner_id: UUID,
        organization_id: UUID,
        status: Optional[str] = "active",
        limit: int = 50,
        offset: int = 0,
    ) -> tuple[List[Conversation], int]:
        """소유자별 대화 목록 조회 (페이지네이션)."""
        filters = [
            Conversation.owner_id == owner_id,
            Conversation.organization_id == organization_id,
            Conversation.deleted_at.is_(None),
        ]
        if status:
            filters.append(Conversation.status == status)
        
        # 총 개수
        count_stmt = select(func.count(Conversation.id)).where(and_(*filters))
        count_result = await self.db.execute(count_stmt)
        total = count_result.scalar()
        
        # 목록 조회
        stmt = (
            select(Conversation)
            .where(and_(*filters))
            .order_by(desc(Conversation.created_at))
            .limit(limit)
            .offset(offset)
        )
        result = await self.db.execute(stmt)
        conversations = result.scalars().all()
        
        return conversations, total
    
    async def create(
        self,
        owner_id: UUID,
        organization_id: UUID,
        title: str,
        retention_days: int = 90,
        metadata: Optional[Dict[str, Any]] = None,
        access_level: str = "private",
    ) -> Conversation:
        """새 대화 생성."""
        expires_at = datetime.utcnow() + timedelta(days=retention_days)
        
        conversation = Conversation(
            owner_id=owner_id,
            organization_id=organization_id,
            title=title,
            retention_days=retention_days,
            expires_at=expires_at,
            metadata=metadata or {},
            access_level=access_level,
        )
        self.db.add(conversation)
        await self.db.flush()
        return conversation
    
    async def update_title(self, conversation_id: UUID, title: str) -> None:
        """대화 제목 수정."""
        stmt = select(Conversation).where(Conversation.id == conversation_id)
        result = await self.db.execute(stmt)
        conversation = result.scalar_one_or_none()
        
        if conversation:
            conversation.title = title
            conversation.updated_at = datetime.utcnow()
            await self.db.flush()


class TurnRepository:
    """Turn 조회 및 조작."""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def get_by_id(self, turn_id: UUID) -> Optional[Turn]:
        """ID로 턴 조회."""
        stmt = select(Turn).where(Turn.id == turn_id)
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()
    
    async def list_by_conversation(self, conversation_id: UUID) -> List[Turn]:
        """대화의 모든 턴 조회 (순서대로)."""
        stmt = (
            select(Turn)
            .where(Turn.conversation_id == conversation_id)
            .order_by(Turn.turn_number)
        )
        result = await self.db.execute(stmt)
        return result.scalars().all()
    
    async def create(
        self,
        conversation_id: UUID,
        turn_number: int,
        user_message: str,
        assistant_response: str,
        retrieval_metadata: Optional[Dict[str, Any]] = None,
    ) -> Turn:
        """새 턴 생성."""
        turn = Turn(
            conversation_id=conversation_id,
            turn_number=turn_number,
            user_message=user_message,
            assistant_response=assistant_response,
            retrieval_metadata=retrieval_metadata or {},
        )
        self.db.add(turn)
        await self.db.flush()
        return turn
```

### 4-5. 단위 테스트

**파일:** `/backend/tests/unit/models/test_conversation_models.py`

```python
"""Conversation 도메인 모델 단위 테스트."""

import pytest
from uuid import uuid4
from datetime import datetime, timedelta

from sqlalchemy.ext.asyncio import AsyncSession

from app.models.conversation import Conversation, Turn, Message


@pytest.mark.asyncio
async def test_conversation_creation(db_session: AsyncSession):
    """Conversation 생성 및 기본값 확인."""
    owner_id = uuid4()
    org_id = uuid4()
    
    conversation = Conversation(
        owner_id=owner_id,
        organization_id=org_id,
        title="Test Conversation",
        retention_days=90,
    )
    
    db_session.add(conversation)
    await db_session.flush()
    
    assert conversation.id is not None
    assert conversation.owner_id == owner_id
    assert conversation.status == "active"
    assert conversation.access_level == "private"
    assert conversation.deleted_at is None
    assert conversation.expires_at is not None


@pytest.mark.asyncio
async def test_conversation_soft_delete(db_session: AsyncSession):
    """Soft delete 동작 확인."""
    conversation = Conversation(
        owner_id=uuid4(),
        organization_id=uuid4(),
        title="Test",
    )
    db_session.add(conversation)
    await db_session.flush()
    
    # Soft delete
    conversation.deleted_at = datetime.utcnow()
    await db_session.flush()
    
    assert conversation.deleted_at is not None


@pytest.mark.asyncio
async def test_turn_creation(db_session: AsyncSession):
    """Turn 생성 및 conversation 관계 확인."""
    conversation = Conversation(
        owner_id=uuid4(),
        organization_id=uuid4(),
        title="Test",
    )
    db_session.add(conversation)
    await db_session.flush()
    
    turn = Turn(
        conversation_id=conversation.id,
        turn_number=1,
        user_message="What is AI?",
        assistant_response="AI is...",
        retrieval_metadata={"citations": []},
    )
    db_session.add(turn)
    await db_session.flush()
    
    assert turn.conversation_id == conversation.id
    assert turn.turn_number == 1


@pytest.mark.asyncio
async def test_message_creation(db_session: AsyncSession):
    """Message 생성 및 turn 관계 확인."""
    conversation = Conversation(
        owner_id=uuid4(),
        organization_id=uuid4(),
        title="Test",
    )
    db_session.add(conversation)
    
    turn = Turn(
        conversation_id=conversation.id,
        turn_number=1,
        user_message="Q",
        assistant_response="A",
    )
    db_session.add(turn)
    await db_session.flush()
    
    message = Message(
        turn_id=turn.id,
        role="user",
        content="What is AI?",
        metadata={"token_count": 5},
    )
    db_session.add(message)
    await db_session.flush()
    
    assert message.turn_id == turn.id
    assert message.role == "user"
```

## 5. 산출물

1. **Conversation/Turn/Message 도메인 모델** (`/backend/app/models/conversation.py`)
   - SQLAlchemy ORM 클래스 정의
   - ACL, soft delete, 메타데이터 JSONB 필드 포함

2. **감사 로그 통합** (기존 audit_log 활용 또는 확장)
   - actor_type 필드 필수 포함 (S2 원칙 ⑥)

3. **Alembic 마이그레이션 스크립트** (`/backend/alembic/versions/0003_create_conversation_domain.py`)
   - conversations, turns, messages 테이블 생성
   - 인덱스 및 제약 정의

4. **Repository 계층** (`/backend/app/repositories/conversation_repository.py`)
   - ConversationRepository, TurnRepository, MessageRepository
   - 조회, 필터링, 생성 메서드

5. **단위 테스트** (`/backend/tests/unit/models/test_conversation_models.py`)
   - 모델 생성, 관계, soft delete, 제약 검증

## 6. 완료 기준

1. Conversation, Turn, Message 모델이 정규화되었는가?
2. ACL 필드가 포함되고 Document와 동등한가?
3. soft delete (deleted_at) 구조가 올바른가?
4. retention_days, expires_at 필드가 정확하게 정의되었는가?
5. 감사 로그 통합 시 actor_type 필드가 필수인가? (S2 원칙 ⑥)
6. Alembic 마이그레이션이 정상 작동하는가 (upgrade/downgrade)?
7. 모든 제약(CHECK, FOREIGN KEY, UNIQUE)이 적용되었는가?
8. 인덱스가 쿼리 성능을 고려하여 설정되었는가?
9. 단위 테스트가 모두 통과하는가 (`pytest -v`)?
10. mypy 타입 검사를 통과하는가?

## 7. 작업 지침

### 지침 7-1. ACL 일관성

Conversation의 ACL 구조는 Document와 동일하게 설계한다. Document의 ACL 로직을 분석하고, 동일한 필드(`owner_id`, `organization_id`, `access_level`)와 검사 로직을 적용한다.

```python
# 예시: Scope Profile 기반 필터링 (Task 3-2에서 사용)
# ACL 필터는 모든 조회 쿼리에 적용됨
def apply_acl_filter(query, actor_id: UUID, actor_type: str, scope_profile: dict):
    # S2 원칙 ⑥: 접근 범위는 Scope Profile로 관리
    # 하드코딩된 scope 문자열 금지
    pass
```

### 지침 7-2. retention_days와 expires_at 계산

대화 생성 시 `expires_at = created_at + retention_days` 로 자동 계산한다:

```python
conversation = Conversation(
    owner_id=owner_id,
    organization_id=org_id,
    title=title,
    retention_days=90,  # 조직 기본값
)
# DB 트리거 또는 애플리케이션에서 expires_at 계산
conversation.expires_at = conversation.created_at + timedelta(days=90)
```

### 지침 7-3. retrieval_metadata 구조

Turn의 `retrieval_metadata` JSONB 필드는 다음 구조를 따른다:

```json
{
  "citations": [
    {
      "document_id": "uuid",
      "version_id": "uuid",
      "node_id": "uuid",
      "span_offset": 100,
      "content_hash": "sha256_hex"
    }
  ],
  "query_original": "사용자가 입력한 원문",
  "query_rewritten": "재작성된 쿼리 (Phase 2 통합)",
  "context_window_turns": ["turn_id_1", "turn_id_2"],
  "retrieval_time_ms": 523
}
```

### 지침 7-4. 마이그레이션 버전 관리

Alembic 마이그레이션 파일의 down_revision을 정확하게 설정하여 이전 Phase의 마이그레이션과 연결한다. 현재 task는 기존 마이그레이션(Phase 0, Phase 1, Phase 2)의 최신 버전을 기반으로 진행한다.

### 지침 7-5. 외래키 캐스케이드

Turn은 Conversation을 참조하고, Message는 Turn을 참조한다. 대화 삭제 시 하위 턴과 메시지가 자동으로 삭제되도록 `ondelete="CASCADE"`를 설정한다.

```python
conversation_id = mapped_column(
    PGUUID(as_uuid=True),
    ForeignKey("conversations.id", ondelete="CASCADE"),
    nullable=False
)
```

### 지침 7-6. JSONB 필드 기본값

PostgreSQL의 JSONB 필드는 `server_default=sa.text("'{}'::jsonb")`로 빈 객체를 기본값으로 설정한다. Python 객체로는 `default=dict`를 사용하되, mutable default를 피하기 위해 factory 함수 사용을 권장한다.

### 지침 7-7. 감사 로그 actor_type 필수

S2 원칙 ⑥에 따라, 모든 감사 이벤트에 `actor_type` (user/agent) 필드를 기록한다. Task 3-2에서 API 엔드포인트를 구현할 때, 요청 헤더나 인증 컨텍스트에서 actor_type을 추출하여 감사 로그에 기록한다.

```python
# 감사 로그 작성 예시
audit_log = AuditLog(
    entity_type="conversation",
    entity_id=conversation_id,
    action="delete",
    actor_id=user_id,
    actor_type="user",  # 또는 "agent"
    changes={"reason": "user_requested"}
)
```
