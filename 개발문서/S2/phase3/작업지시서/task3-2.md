# Task 3-2. 대화 CRUD API 및 Repository 패턴 구현

## 1. 작업 목적

Task 3-1에서 정의한 Conversation/Turn/Message 도메인 모델을 기반으로 **FastAPI 라우터와 Repository 계층**을 구현하여, 사용자와 에이전트가 대화를 조회, 생성, 삭제, 검색할 수 있는 RESTful API를 제공한다. 이때 **Scope Profile 기반 ACL 필터링**(S2 원칙 ⑥)과 **actor_type 감시 로그**(S2 원칙 ⑥)를 의무적으로 적용한다.

## 2. 작업 범위

### 포함 범위

1. **Pydantic 스키마 정의**
   - ConversationSchema, TurnSchema, MessageSchema (응답용)
   - ConversationCreateRequest, ConversationUpdateRequest (요청용)
   - 페이지네이션 응답: PaginatedResponse[ConversationSchema]

2. **Repository 계층 확장**
   - ConversationRepository: CRUD, 필터링, 전체 텍스트 검색 (FTS)
   - TurnRepository: 턴 조회, 생성, 메타데이터 갱신
   - MessageRepository: 메시지 조회, 생성

3. **FastAPI 라우터**
   - `GET /api/v1/conversations`: 대화 목록 조회 (페이지네이션, 정렬, 필터)
   - `POST /api/v1/conversations`: 새 대화 생성
   - `GET /api/v1/conversations/{conversation_id}`: 대화 상세 조회
   - `PUT /api/v1/conversations/{conversation_id}`: 대화 수정 (제목, 메타데이터)
   - `DELETE /api/v1/conversations/{conversation_id}`: 대화 삭제 (soft delete)
   - `GET /api/v1/conversations/{conversation_id}/turns`: 대화의 턴 목록
   - `GET /api/v1/conversations/{conversation_id}/turns/{turn_id}`: 특정 턴 상세 조회
   - `POST /api/v1/conversations/{conversation_id}/turns/{turn_id}/redact`: 턴의 민감 정보 제거

4. **Scope Profile 기반 ACL 필터링 (S2 원칙 ⑥)**
   - 모든 조회/쓰기 API에 필수 적용
   - 코드에 `if scope == "team"` 같은 하드코딩 금지
   - Scope Profile (조직 설정)으로 접근 범위 관리
   - `unfilteredQuery` → `aclFiltered(query, actor_id, scope_profile)` 패턴

5. **감시 로그 기록 (S2 원칙 ⑥)**
   - 생성/수정/삭제/redact 작업마다 audit_log 기록
   - actor_type 필드 필수: `actor_type=request.headers.get("X-Actor-Type", "user")`
   - 변경 사항(changes JSONB)에 이전/새 값 기록

6. **unwrapEnvelope 패턴 (S2 프론트엔드 규약)**
   - API 응답 형식: `{status: "ok", data: {...}, errors: null}`
   - 프론트엔드가 일관성 있게 응답을 처리할 수 있도록 표준화

7. **단위/통합 테스트**
   - Repository 메서드 단위 테스트
   - API 엔드포인트 통합 테스트 (FastAPI TestClient)
   - ACL 필터링 정확성 확인
   - 감시 로그 기록 확인

### 제외 범위

- PII 자동 만료 배치 (Task 3-3)
- 세션 기반 RAG 통합 (FG3.2)
- 대화 내 전체 텍스트 검색 FTS 고급 기능 (Phase 6에서)
- UI 구현 (FG3.3)

## 3. 선행 조건

- Task 3-1 완료 (Conversation/Turn/Message 모델, DB 스키마)
- Phase 1, Phase 2 완료
- FastAPI 및 SQLAlchemy async 세션 관리 완료
- 기존 Scope Profile 구조 분석 완료 (조직 설정에서 접근 범위 관리 방식)
- 기존 감시 로그 테이블 구조 확인 (`audit_logs` 테이블)

## 4. 주요 작업 항목

### 4-1. Pydantic 스키마 정의

**파일:** `/backend/app/schemas/conversation.py`

```python
from __future__ import annotations

from typing import Optional, List, Dict, Any
from uuid import UUID
from datetime import datetime

from pydantic import BaseModel, Field, field_validator


class MessageSchema(BaseModel):
    """메시지 응답 스키마."""
    
    id: UUID
    turn_id: UUID
    role: str  # user, assistant, system
    content: str
    created_at: datetime
    metadata: Dict[str, Any] = Field(default_factory=dict)
    
    class Config:
        from_attributes = True


class TurnSchema(BaseModel):
    """턴 응답 스키마."""
    
    id: UUID
    conversation_id: UUID
    turn_number: int
    created_at: datetime
    user_message: str
    assistant_response: str
    retrieval_metadata: Dict[str, Any] = Field(default_factory=dict)
    messages: List[MessageSchema] = Field(default_factory=list)
    
    class Config:
        from_attributes = True


class ConversationSchema(BaseModel):
    """대화 응답 스키마 (요약)."""
    
    id: UUID
    owner_id: UUID
    organization_id: UUID
    created_at: datetime
    updated_at: datetime
    title: str
    status: str
    metadata: Dict[str, Any] = Field(default_factory=dict)
    retention_days: int
    expires_at: Optional[datetime] = None
    access_level: str
    turn_count: Optional[int] = None  # 턴 개수 (선택)
    
    class Config:
        from_attributes = True


class ConversationDetailSchema(ConversationSchema):
    """대화 상세 응답 스키마 (턴 포함)."""
    
    turns: List[TurnSchema] = Field(default_factory=list)


class ConversationCreateRequest(BaseModel):
    """대화 생성 요청 스키마."""
    
    title: str = Field(..., min_length=1, max_length=256)
    metadata: Optional[Dict[str, Any]] = Field(None)
    retention_days: Optional[int] = Field(None, ge=1)
    access_level: Optional[str] = Field("private")
    
    @field_validator("access_level")
    @classmethod
    def validate_access_level(cls, v):
        if v not in ("private", "organization", "public"):
            raise ValueError("Invalid access_level")
        return v


class ConversationUpdateRequest(BaseModel):
    """대화 수정 요청 스키마."""
    
    title: Optional[str] = Field(None, min_length=1, max_length=256)
    metadata: Optional[Dict[str, Any]] = Field(None)
    status: Optional[str] = Field(None)
    access_level: Optional[str] = Field(None)


class TurnCreateRequest(BaseModel):
    """턴 생성 요청 스키마."""
    
    user_message: str = Field(..., min_length=1)
    assistant_response: str = Field(..., min_length=1)
    retrieval_metadata: Optional[Dict[str, Any]] = Field(None)


class RedactRequest(BaseModel):
    """민감 정보 제거(redact) 요청 스키마."""
    
    fields: List[str] = Field(..., min_items=1)  # ['user_message', 'assistant_response']
    reason: str = Field(..., min_length=1)


class PaginatedResponse(BaseModel):
    """페이지네이션된 응답 래퍼 (unwrapEnvelope 패턴)."""
    
    status: str = "ok"
    data: List[Any] = Field(default_factory=list)
    pagination: Dict[str, Any] = Field(default_factory=dict)  # {total, limit, offset, has_more}
    errors: Optional[List[str]] = None


class ApiResponse(BaseModel):
    """단일 응답 래퍼 (unwrapEnvelope 패턴)."""
    
    status: str = "ok"
    data: Optional[Any] = None
    errors: Optional[List[str]] = None
```

### 4-2. Repository 계층 확장

**파일:** `/backend/app/repositories/conversation_repository.py` (Task 3-1 확장)

```python
"""Conversation/Turn/Message 저장소 (Query, Filter, FTS)."""

from __future__ import annotations

from typing import Optional, List, Dict, Any, Tuple
from uuid import UUID
from datetime import datetime, timedelta
import logging

from sqlalchemy import and_, or_, select, desc, func, text
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from app.models.conversation import Conversation, Turn, Message
from app.models.audit_log import AuditLog

logger = logging.getLogger(__name__)


class ConversationRepository:
    """Conversation 조회, 필터링, 생성, 수정, 삭제."""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def get_by_id(self, conversation_id: UUID) -> Optional[Conversation]:
        """ID로 대화 조회 (활성 대화만)."""
        stmt = (
            select(Conversation)
            .where(
                and_(
                    Conversation.id == conversation_id,
                    Conversation.deleted_at.is_(None)
                )
            )
            .options(selectinload(Conversation.turns))
        )
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()
    
    async def list_by_owner(
        self,
        owner_id: UUID,
        organization_id: UUID,
        scope_profile: Dict[str, Any],  # S2 원칙 ⑥: Scope Profile 기반 ACL
        status: Optional[str] = "active",
        limit: int = 50,
        offset: int = 0,
    ) -> Tuple[List[Conversation], int]:
        """소유자별 대화 목록 조회 (ACL 필터링 포함).
        
        S2 원칙 ⑥: scope_profile에서 접근 범위를 결정.
        코드에 'if scope == "team"' 같은 하드코딩 금지.
        """
        filters = [Conversation.deleted_at.is_(None)]
        
        # ACL 필터링: scope_profile 기반
        # 예: scope_profile = {"scope": "organization", "include_public": False}
        scope = scope_profile.get("scope", "private")
        
        if scope == "private":
            filters.append(Conversation.owner_id == owner_id)
        elif scope == "organization":
            filters.append(
                or_(
                    Conversation.owner_id == owner_id,
                    and_(
                        Conversation.organization_id == organization_id,
                        Conversation.access_level.in_(["organization", "public"])
                    )
                )
            )
        elif scope == "public":
            filters.append(
                or_(
                    Conversation.owner_id == owner_id,
                    Conversation.access_level == "public"
                )
            )
        
        if status:
            filters.append(Conversation.status == status)
        
        # 총 개수
        count_stmt = select(func.count(Conversation.id)).where(and_(*filters))
        count_result = await self.db.execute(count_stmt)
        total = count_result.scalar() or 0
        
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
    
    async def search_by_title(
        self,
        owner_id: UUID,
        organization_id: UUID,
        scope_profile: Dict[str, Any],
        query: str,
        limit: int = 50,
        offset: int = 0,
    ) -> Tuple[List[Conversation], int]:
        """대화 제목/메타데이터 전체 텍스트 검색 (ACL 필터링).
        
        PostgreSQL FTS 또는 ILIKE 사용.
        """
        filters = [Conversation.deleted_at.is_(None)]
        
        # ACL 필터링 (위와 동일)
        scope = scope_profile.get("scope", "private")
        if scope == "private":
            filters.append(Conversation.owner_id == owner_id)
        elif scope == "organization":
            filters.append(
                or_(
                    Conversation.owner_id == owner_id,
                    and_(
                        Conversation.organization_id == organization_id,
                        Conversation.access_level.in_(["organization", "public"])
                    )
                )
            )
        
        # 제목 검색 (ILIKE 또는 FTS)
        search_filter = Conversation.title.ilike(f"%{query}%")
        filters.append(search_filter)
        
        # 총 개수
        count_stmt = select(func.count(Conversation.id)).where(and_(*filters))
        count_result = await self.db.execute(count_stmt)
        total = count_result.scalar() or 0
        
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
        retention_days: Optional[int] = None,
        metadata: Optional[Dict[str, Any]] = None,
        access_level: str = "private",
        actor_id: UUID = None,
        actor_type: str = "user",  # S2 원칙 ⑥: actor_type
    ) -> Conversation:
        """새 대화 생성 및 감시 로그 기록."""
        if retention_days is None:
            retention_days = 90  # 조직 기본값 (이후 설정 테이블에서 조회)
        
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
        
        # 감시 로그 기록
        if actor_id:
            audit_log = AuditLog(
                entity_type="conversation",
                entity_id=conversation.id,
                action="create",
                actor_id=actor_id,
                actor_type=actor_type,
                changes={"title": title, "access_level": access_level}
            )
            self.db.add(audit_log)
            await self.db.flush()
        
        return conversation
    
    async def update(
        self,
        conversation_id: UUID,
        title: Optional[str] = None,
        metadata: Optional[Dict[str, Any]] = None,
        status: Optional[str] = None,
        access_level: Optional[str] = None,
        actor_id: UUID = None,
        actor_type: str = "user",
    ) -> Optional[Conversation]:
        """대화 수정 및 감시 로그 기록."""
        stmt = select(Conversation).where(Conversation.id == conversation_id)
        result = await self.db.execute(stmt)
        conversation = result.scalar_one_or_none()
        
        if not conversation:
            return None
        
        changes = {}
        
        if title is not None:
            changes["title"] = (conversation.title, title)
            conversation.title = title
        
        if metadata is not None:
            changes["metadata"] = (conversation.metadata, metadata)
            conversation.metadata = metadata
        
        if status is not None:
            changes["status"] = (conversation.status, status)
            conversation.status = status
        
        if access_level is not None:
            changes["access_level"] = (conversation.access_level, access_level)
            conversation.access_level = access_level
        
        conversation.updated_at = datetime.utcnow()
        await self.db.flush()
        
        # 감시 로그
        if changes and actor_id:
            audit_log = AuditLog(
                entity_type="conversation",
                entity_id=conversation.id,
                action="update",
                actor_id=actor_id,
                actor_type=actor_type,
                changes=changes
            )
            self.db.add(audit_log)
            await self.db.flush()
        
        return conversation
    
    async def soft_delete(
        self,
        conversation_id: UUID,
        actor_id: UUID = None,
        actor_type: str = "user",
        reason: Optional[str] = None,
    ) -> bool:
        """대화 soft delete 및 감시 로그 기록."""
        stmt = select(Conversation).where(Conversation.id == conversation_id)
        result = await self.db.execute(stmt)
        conversation = result.scalar_one_or_none()
        
        if not conversation:
            return False
        
        conversation.deleted_at = datetime.utcnow()
        conversation.status = "deleted"
        await self.db.flush()
        
        # 감시 로그
        if actor_id:
            audit_log = AuditLog(
                entity_type="conversation",
                entity_id=conversation.id,
                action="delete",
                actor_id=actor_id,
                actor_type=actor_type,
                changes={"reason": reason or "user_requested"}
            )
            self.db.add(audit_log)
            await self.db.flush()
        
        return True


class TurnRepository:
    """Turn 조회, 생성, 메타데이터 갱신."""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def get_by_id(self, turn_id: UUID) -> Optional[Turn]:
        """ID로 턴 조회."""
        stmt = (
            select(Turn)
            .where(Turn.id == turn_id)
            .options(selectinload(Turn.messages))
        )
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()
    
    async def list_by_conversation(self, conversation_id: UUID) -> List[Turn]:
        """대화의 모든 턴 조회 (순서대로)."""
        stmt = (
            select(Turn)
            .where(Turn.conversation_id == conversation_id)
            .order_by(Turn.turn_number)
            .options(selectinload(Turn.messages))
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
    
    async def update_metadata(
        self,
        turn_id: UUID,
        retrieval_metadata: Dict[str, Any],
    ) -> Optional[Turn]:
        """턴의 메타데이터 갱신."""
        stmt = select(Turn).where(Turn.id == turn_id)
        result = await self.db.execute(stmt)
        turn = result.scalar_one_or_none()
        
        if turn:
            turn.retrieval_metadata = retrieval_metadata
            await self.db.flush()
        
        return turn


class MessageRepository:
    """Message 조회, 생성."""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def create(
        self,
        turn_id: UUID,
        role: str,
        content: str,
        metadata: Optional[Dict[str, Any]] = None,
    ) -> Message:
        """새 메시지 생성."""
        message = Message(
            turn_id=turn_id,
            role=role,
            content=content,
            metadata=metadata or {},
        )
        self.db.add(message)
        await self.db.flush()
        return message
    
    async def list_by_turn(self, turn_id: UUID) -> List[Message]:
        """턴의 모든 메시지 조회."""
        stmt = (
            select(Message)
            .where(Message.turn_id == turn_id)
            .order_by(Message.created_at)
        )
        result = await self.db.execute(stmt)
        return result.scalars().all()
```

### 4-3. FastAPI 라우터

**파일:** `/backend/app/routes/conversations.py`

```python
"""Conversation CRUD API 라우터."""

from __future__ import annotations

from typing import Optional
from uuid import UUID
import logging

from fastapi import APIRouter, Depends, HTTPException, Query, Header
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.auth import get_current_user  # 기존 인증 미들웨어
from app.schemas.conversation import (
    ConversationSchema, ConversationDetailSchema, ConversationCreateRequest,
    ConversationUpdateRequest, TurnSchema, PaginatedResponse, ApiResponse,
    RedactRequest
)
from app.repositories.conversation_repository import (
    ConversationRepository, TurnRepository, MessageRepository
)

logger = logging.getLogger(__name__)
router = APIRouter(prefix="/api/v1/conversations", tags=["conversations"])


async def get_scope_profile(user_id: UUID, db: AsyncSession) -> dict:
    """사용자의 Scope Profile 조회 (조직 설정에서)."""
    # TODO: 실제 구현은 사용자 설정 또는 조직 설정 테이블에서 조회
    # 기본값: private (자신의 대화만 조회 가능)
    return {"scope": "private", "include_public": False}


@router.get("", response_model=PaginatedResponse)
async def list_conversations(
    limit: int = Query(50, ge=1, le=500),
    offset: int = Query(0, ge=0),
    status: Optional[str] = Query(None),
    current_user: dict = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
    x_actor_type: str = Header("user"),  # S2 원칙 ⑥: actor_type
):
    """사용자의 대화 목록 조회 (ACL 필터링).
    
    Query Parameters:
        limit: 페이지 크기 (기본: 50)
        offset: 페이지 오프셋 (기본: 0)
        status: 필터 (active, archived, expired)
    
    Headers:
        X-Actor-Type: user | agent (감시 로그용)
    """
    repo = ConversationRepository(db)
    scope_profile = await get_scope_profile(current_user["id"], db)
    
    conversations, total = await repo.list_by_owner(
        owner_id=current_user["id"],
        organization_id=current_user["organization_id"],
        scope_profile=scope_profile,
        status=status,
        limit=limit,
        offset=offset,
    )
    
    return PaginatedResponse(
        status="ok",
        data=[ConversationSchema.from_orm(c) for c in conversations],
        pagination={
            "total": total,
            "limit": limit,
            "offset": offset,
            "has_more": offset + limit < total,
        }
    )


@router.post("", response_model=ApiResponse)
async def create_conversation(
    request: ConversationCreateRequest,
    current_user: dict = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
    x_actor_type: str = Header("user"),
):
    """새 대화 생성.
    
    Request Body:
        title: 대화 제목
        metadata: 선택 메타데이터
        access_level: private | organization | public (기본: private)
    """
    repo = ConversationRepository(db)
    
    conversation = await repo.create(
        owner_id=current_user["id"],
        organization_id=current_user["organization_id"],
        title=request.title,
        retention_days=request.retention_days,
        metadata=request.metadata,
        access_level=request.access_level,
        actor_id=current_user["id"],
        actor_type=x_actor_type,
    )
    
    await db.commit()
    
    logger.info(
        f"Conversation created: {conversation.id} by {current_user['id']} ({x_actor_type})"
    )
    
    return ApiResponse(
        status="ok",
        data=ConversationSchema.from_orm(conversation),
    )


@router.get("/{conversation_id}", response_model=ApiResponse)
async def get_conversation(
    conversation_id: UUID,
    current_user: dict = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """대화 상세 조회."""
    repo = ConversationRepository(db)
    conversation = await repo.get_by_id(conversation_id)
    
    if not conversation:
        raise HTTPException(status_code=404, detail="Conversation not found")
    
    # ACL 검사
    if conversation.owner_id != current_user["id"] and conversation.access_level == "private":
        raise HTTPException(status_code=403, detail="Forbidden")
    
    return ApiResponse(
        status="ok",
        data=ConversationDetailSchema.from_orm(conversation),
    )


@router.put("/{conversation_id}", response_model=ApiResponse)
async def update_conversation(
    conversation_id: UUID,
    request: ConversationUpdateRequest,
    current_user: dict = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
    x_actor_type: str = Header("user"),
):
    """대화 수정 (제목, 메타데이터, 상태, 접근 레벨)."""
    repo = ConversationRepository(db)
    conversation = await repo.get_by_id(conversation_id)
    
    if not conversation:
        raise HTTPException(status_code=404, detail="Conversation not found")
    
    # 권한 검사: 소유자만 수정 가능
    if conversation.owner_id != current_user["id"]:
        raise HTTPException(status_code=403, detail="Forbidden")
    
    conversation = await repo.update(
        conversation_id=conversation_id,
        title=request.title,
        metadata=request.metadata,
        status=request.status,
        access_level=request.access_level,
        actor_id=current_user["id"],
        actor_type=x_actor_type,
    )
    
    await db.commit()
    
    return ApiResponse(
        status="ok",
        data=ConversationSchema.from_orm(conversation),
    )


@router.delete("/{conversation_id}", response_model=ApiResponse)
async def delete_conversation(
    conversation_id: UUID,
    current_user: dict = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
    x_actor_type: str = Header("user"),
):
    """대화 삭제 (soft delete)."""
    repo = ConversationRepository(db)
    conversation = await repo.get_by_id(conversation_id)
    
    if not conversation:
        raise HTTPException(status_code=404, detail="Conversation not found")
    
    # 권한 검사
    if conversation.owner_id != current_user["id"]:
        raise HTTPException(status_code=403, detail="Forbidden")
    
    success = await repo.soft_delete(
        conversation_id=conversation_id,
        actor_id=current_user["id"],
        actor_type=x_actor_type,
        reason="user_requested",
    )
    
    await db.commit()
    
    if not success:
        raise HTTPException(status_code=500, detail="Failed to delete conversation")
    
    return ApiResponse(status="ok", data={"deleted": True})


@router.get("/{conversation_id}/turns", response_model=ApiResponse)
async def list_turns(
    conversation_id: UUID,
    current_user: dict = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """대화의 모든 턴 조회."""
    repo_conversation = ConversationRepository(db)
    repo_turn = TurnRepository(db)
    
    conversation = await repo_conversation.get_by_id(conversation_id)
    if not conversation:
        raise HTTPException(status_code=404, detail="Conversation not found")
    
    # ACL 검사
    if conversation.owner_id != current_user["id"] and conversation.access_level == "private":
        raise HTTPException(status_code=403, detail="Forbidden")
    
    turns = await repo_turn.list_by_conversation(conversation_id)
    
    return ApiResponse(
        status="ok",
        data=[TurnSchema.from_orm(t) for t in turns],
    )


@router.get("/{conversation_id}/turns/{turn_id}", response_model=ApiResponse)
async def get_turn(
    conversation_id: UUID,
    turn_id: UUID,
    current_user: dict = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """특정 턴 상세 조회."""
    repo_conversation = ConversationRepository(db)
    repo_turn = TurnRepository(db)
    
    conversation = await repo_conversation.get_by_id(conversation_id)
    if not conversation:
        raise HTTPException(status_code=404, detail="Conversation not found")
    
    # ACL 검사
    if conversation.owner_id != current_user["id"] and conversation.access_level == "private":
        raise HTTPException(status_code=403, detail="Forbidden")
    
    turn = await repo_turn.get_by_id(turn_id)
    if not turn or turn.conversation_id != conversation_id:
        raise HTTPException(status_code=404, detail="Turn not found")
    
    return ApiResponse(
        status="ok",
        data=TurnSchema.from_orm(turn),
    )


@router.post("/{conversation_id}/turns/{turn_id}/redact", response_model=ApiResponse)
async def redact_turn(
    conversation_id: UUID,
    turn_id: UUID,
    request: RedactRequest,
    current_user: dict = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
    x_actor_type: str = Header("user"),
):
    """턴의 민감 정보 제거 (redact).
    
    Request Body:
        fields: 제거할 필드 리스트 (user_message, assistant_response)
        reason: 제거 사유
    """
    repo_conversation = ConversationRepository(db)
    repo_turn = TurnRepository(db)
    
    conversation = await repo_conversation.get_by_id(conversation_id)
    if not conversation:
        raise HTTPException(status_code=404, detail="Conversation not found")
    
    # 권한 검사: 소유자만 redact 가능
    if conversation.owner_id != current_user["id"]:
        raise HTTPException(status_code=403, detail="Forbidden")
    
    turn = await repo_turn.get_by_id(turn_id)
    if not turn or turn.conversation_id != conversation_id:
        raise HTTPException(status_code=404, detail="Turn not found")
    
    # 민감 정보 제거
    original_data = {}
    for field in request.fields:
        if field == "user_message":
            original_data["user_message"] = turn.user_message
            turn.user_message = "[REDACTED]"
        elif field == "assistant_response":
            original_data["assistant_response"] = turn.assistant_response
            turn.assistant_response = "[REDACTED]"
    
    await db.flush()
    
    # 감시 로그
    audit_log = AuditLog(
        entity_type="turn",
        entity_id=turn.id,
        action="redact",
        actor_id=current_user["id"],
        actor_type=x_actor_type,
        changes={"fields": request.fields, "reason": request.reason}
    )
    db.add(audit_log)
    
    await db.commit()
    
    logger.info(
        f"Turn {turn_id} redacted by {current_user['id']} ({x_actor_type}): {request.reason}"
    )
    
    return ApiResponse(
        status="ok",
        data={"redacted": True, "fields": request.fields},
    )
```

### 4-4. 단위/통합 테스트

**파일:** `/backend/tests/integration/test_conversation_api.py`

```python
"""Conversation API 통합 테스트."""

import pytest
from uuid import uuid4
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

from app.main import app
from app.models.conversation import Conversation, Turn
from app.repositories.conversation_repository import ConversationRepository
from app.database import get_db


@pytest.fixture
def mock_current_user():
    """테스트용 현재 사용자."""
    return {
        "id": uuid4(),
        "organization_id": uuid4(),
        "username": "testuser"
    }


@pytest.fixture
async def client():
    """테스트 클라이언트."""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac


@pytest.mark.asyncio
async def test_create_conversation(mock_current_user, client, db_session: AsyncSession):
    """대화 생성 API 테스트."""
    response = await client.post(
        "/api/v1/conversations",
        json={
            "title": "Test Conversation",
            "access_level": "private",
        },
        headers={"X-Actor-Type": "user"},
    )
    
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "ok"
    assert data["data"]["title"] == "Test Conversation"
    assert data["data"]["status"] == "active"


@pytest.mark.asyncio
async def test_list_conversations_with_acl(
    mock_current_user, client, db_session: AsyncSession
):
    """ACL 필터링을 포함한 대화 목록 조회 테스트."""
    repo = ConversationRepository(db_session)
    
    # 현재 사용자의 private 대화
    conv1 = await repo.create(
        owner_id=mock_current_user["id"],
        organization_id=mock_current_user["organization_id"],
        title="Private Conversation",
        access_level="private",
        actor_id=mock_current_user["id"],
    )
    
    # 다른 사용자의 public 대화
    other_user_id = uuid4()
    conv2 = await repo.create(
        owner_id=other_user_id,
        organization_id=mock_current_user["organization_id"],
        title="Public Conversation",
        access_level="public",
        actor_id=other_user_id,
    )
    
    await db_session.commit()
    
    # 조회
    response = await client.get("/api/v1/conversations")
    assert response.status_code == 200
    
    data = response.json()
    assert data["pagination"]["total"] == 2  # private + public


@pytest.mark.asyncio
async def test_redact_turn_with_audit_log(
    mock_current_user, client, db_session: AsyncSession
):
    """Turn redact API 및 감시 로그 기록 테스트."""
    repo_conversation = ConversationRepository(db_session)
    repo_turn = TurnRepository(db_session)
    
    # 대화 및 턴 생성
    conversation = await repo_conversation.create(
        owner_id=mock_current_user["id"],
        organization_id=mock_current_user["organization_id"],
        title="Test",
        actor_id=mock_current_user["id"],
    )
    
    turn = await repo_turn.create(
        conversation_id=conversation.id,
        turn_number=1,
        user_message="What is my SSN?",
        assistant_response="I cannot provide that information.",
    )
    
    await db_session.commit()
    
    # Redact API 호출
    response = await client.post(
        f"/api/v1/conversations/{conversation.id}/turns/{turn.id}/redact",
        json={
            "fields": ["user_message"],
            "reason": "PII removal",
        },
        headers={"X-Actor-Type": "user"},
    )
    
    assert response.status_code == 200
    
    # 감시 로그 확인
    from app.models.audit_log import AuditLog
    from sqlalchemy import select
    
    stmt = select(AuditLog).where(AuditLog.entity_id == turn.id)
    result = await db_session.execute(stmt)
    audit_log = result.scalar_one_or_none()
    
    assert audit_log is not None
    assert audit_log.action == "redact"
    assert audit_log.actor_type == "user"
    assert "fields" in audit_log.changes
```

## 5. 산출물

1. **Pydantic 스키마** (`/backend/app/schemas/conversation.py`)
   - ConversationSchema, TurnSchema, MessageSchema
   - 요청/응답 모델
   - PaginatedResponse, ApiResponse (unwrapEnvelope 패턴)

2. **Repository 계층 확장** (`/backend/app/repositories/conversation_repository.py`)
   - ConversationRepository: CRUD, ACL 필터링, FTS
   - TurnRepository: 턴 조회, 생성, 메타데이터 갱신
   - MessageRepository: 메시지 관리
   - 감시 로그 통합

3. **FastAPI 라우터** (`/backend/app/routes/conversations.py`)
   - GET/POST /conversations
   - GET /conversations/{id}
   - PUT /conversations/{id}
   - DELETE /conversations/{id}
   - GET /conversations/{id}/turns
   - GET /conversations/{id}/turns/{turn_id}
   - POST /conversations/{id}/turns/{turn_id}/redact

4. **단위/통합 테스트** (`/backend/tests/integration/test_conversation_api.py`)
   - API 엔드포인트 테스트
   - ACL 필터링 검증
   - 감시 로그 기록 확인

## 6. 완료 기준

1. 모든 API 엔드포인트가 정상 작동하는가?
2. ACL 필터링이 모든 조회 API에 적용되었는가? (S2 원칙 ⑥)
3. 감시 로그에 actor_type이 기록되는가? (S2 원칙 ⑥)
4. unwrapEnvelope 패턴 (ApiResponse, PaginatedResponse)이 일관성 있게 적용되었는가?
5. 페이지네이션이 정확하게 작동하는가?
6. Soft delete 조회 시 deleted_at IS NULL 필터가 자동 적용되는가?
7. 권한 검사가 모든 쓰기 API에서 수행되는가?
8. 통합 테스트가 모두 통과하는가?
9. mypy 타입 검사를 통과하는가?

## 7. 작업 지침

### 지침 7-1. Scope Profile 기반 ACL

S2 원칙 ⑥에 따라, 코드에 `if scope == "team"` 같은 접근 범위 문자열을 하드코딩하지 않는다. 대신 Scope Profile 객체에서 scope 정보를 동적으로 읽는다:

```python
scope_profile = {
    "scope": "organization",  # 관리자 또는 조직 설정에서 결정
    "include_public": False,
    "include_archived": False
}

# Repository 호출 시 전달
conversations, total = await repo.list_by_owner(
    ...,
    scope_profile=scope_profile,
    ...
)
```

### 지침 7-2. actor_type 헤더 처리

모든 쓰기 API(POST, PUT, DELETE)에서 `X-Actor-Type` 헤더를 읽어 감시 로그에 기록한다:

```python
@router.post("")
async def create_conversation(
    ...,
    x_actor_type: str = Header("user"),  # 기본값: "user"
):
    # actor_type을 Repository 메서드에 전달
    await repo.create(..., actor_type=x_actor_type)
```

### 지침 7-3. unwrapEnvelope 패턴

모든 API 응답은 다음 형식을 따른다:

```json
{
  "status": "ok",
  "data": {...},
  "errors": null
}
```

또는 오류 시:

```json
{
  "status": "error",
  "data": null,
  "errors": ["에러 메시지"]
}
```

### 지침 7-4. soft delete 기본 필터

조회 쿼리에서 항상 `deleted_at IS NULL` 필터를 적용한다. 이를 위해 Repository의 모든 조회 메서드에서 이 필터를 포함시킨다.

### 지침 7-5. 트랜잭션 관리

Repository 메서드는 flush까지만 수행하고, 최종 commit은 라우터에서 수행한다:

```python
# Repository
conversation = await repo.create(...)
await db.flush()  # flush만

# Router
await db.commit()  # 최종 commit
```

### 지침 7-6. 감시 로그 변경 사항 형식

변경 사항(changes)은 JSONB로 저장되며, update의 경우 (old, new) 튜플 형식을 사용한다:

```python
changes = {
    "title": ("Old Title", "New Title"),
    "status": ("active", "archived")
}
```

### 지침 7-7. 에러 응답 표준화

FastAPI HTTPException을 사용하되, 응답은 unwrapEnvelope 패턴을 따른다. 예외 처리 미들웨어에서 변환한다.
