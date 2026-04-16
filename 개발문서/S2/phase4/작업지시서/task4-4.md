# Task 4-4. Agent Principal Type + Delegation 모델 + DB 마이그레이션 + PII 배치 잡

## 1. 작업 목적

AI 에이전트를 Mimir의 독립 Principal로 모델링하고, 에이전트가 사용자를 대행하여 API를 호출할 수 있는 위임 구조를 구축한다. 기존의 user/admin/system 역할에 agent 타입을 추가하고, OAuth 2.0 client-credentials 인증 흐름에 통합한다. 모든 API 호출에 대해 actor_type (user/agent) 및 acting_on_behalf_of 필드를 감시 로그에 기록하여 감사 추적을 강화한다 (S2 원칙 ⑤, ⑥).

또한 Phase 3에서 이월된 **PII 배치 잡**을 구현한다. `conversations.expires_at` 컬럼에 기록된 만료 시각을 기준으로 만료된 대화 데이터를 주기적으로 삭제하여 GDPR 및 개인정보 처리방침을 준수한다.

## 2. 작업 범위

### 포함 범위

1. **Agent Principal Type 역할 모델 확장**
   - 기존 역할 시스템 (user/admin/system)에 agent 타입 추가
   - 에이전트는 API Key 기반 OAuth client-credentials 인증 사용
   - 에이전트별 scope_profile_id 바인딩으로 접근 범위 제어 (Scope Profile은 Task 4-5)

2. **agents 테이블 설계 및 DB 스키마**
   - 테이블: agents (UUID id, api_key_id FK, organization_id, scope_profile_id, is_disabled, created_at, metadata)
   - agents와 api_keys 테이블의 관계 설정
   - Agent 생성/조회/업데이트 로직
   - 인덱스: (organization_id, created_at), (api_key_id), (is_disabled)

3. **access_context 스키마 정의**
   - 인증 후 생성되는 컨텍스트 객체
   - 필드: user_id (UUID), organization_id (UUID), principal_type (user/agent), permissions (list), scope_profile_id (UUID, agent만 해당)
   - 요청 처리 파이프라인에서 access_context를 생성하여 모든 API 레이어에 전파

4. **Delegation 모델 구현**
   - delegate:search, delegate:write 권한 정의
   - 에이전트가 특정 사용자를 대행하는 경우, access_context에 acting_on_behalf_of 필드 설정
   - 위임 권한 검증 로직 (delegate:write는 쓰기 작업에만 허용, 감시 로그 필드 체크)

5. **OAuth client-credentials 인증 흐름 통합**
   - 에이전트 API Key를 클라이언트 크레덴셜로 사용
   - 토큰 엔드포인트: POST /api/v1/auth/token (client_id, client_secret)
   - 응답: access_token (JWT), scope_profile_id 포함
   - 폐쇄망 환경: JWT 검증 fallback (외부 OAuth 서버 없을 시 로컬 토큰 생성, S2 원칙 ⑦)

6. **감사 로그 확장**
   - actor_type 필드: "user", "agent" (S2 원칙 ⑥)
   - acting_on_behalf_of 필드: agent가 사용자를 대행할 때 해당 user_id 기록
   - 모든 CRUD 작업에서 access_context.principal_type과 access_context.acting_on_behalf_of를 감시 로그에 기록

7. **Alembic 마이그레이션 스크립트**
   - agents 테이블 생성
   - api_keys 테이블 확장: scope_profile_id, agent_principal_type 컬럼 추가 (NULL 가능, agent용만)
   - 인덱스 및 제약 정의
   - upgrade/downgrade 경로 작성

8. **단위 테스트**
   - Agent 생성, 조회, 업데이트 테스트
   - Delegation 권한 검증 테스트 (delegate:search, delegate:write)
   - access_context 생성 및 전파 테스트
   - 감시 로그 actor_type, acting_on_behalf_of 필드 검증 테스트

9. **PII 배치 잡 — conversations 자동 만료 삭제** _(Phase 3 이월: PH3-CARRY-001)_
   - 대상: `conversations` 테이블의 `expires_at IS NOT NULL AND expires_at < NOW()` 레코드
   - 연쇄 삭제: `turns` → `messages` (FK CASCADE 또는 명시적 순서 삭제)
   - 스케줄러: APScheduler (in-process, 폐쇄망 동작 보장 — S2 원칙 ⑦)
   - 실행 주기: 환경변수 `PII_CLEANUP_INTERVAL_HOURS` (기본: 24시간)
   - 삭제 전 감사 로그 기록 (`actor_type="system"`, `action="pii_cleanup"`)
   - 환경변수 `PII_CLEANUP_ENABLED=false`로 전체 비활성화 가능

### 제외 범위

- Scope Profile 모델 및 FilterExpression 파서 (Task 4-5)
- Scope Profile CRUD API 엔드포인트 및 Kill Switch (Task 4-6)
- MCP Server 구현 (FG4.1)
- 에이전트 UI 관리 페이지 (FG4.3)

## 3. 선행 조건

- Phase 1, Phase 2, Phase 3 완료
- PostgreSQL 설치 및 연결 가능
- SQLAlchemy 2.0+ 설치
- Alembic 초기화 완료
- 기존 api_keys 테이블 구조 분석 완료
- 기존 audit_logs 테이블 구조 분석 완료
- OAuth 2.0 client-credentials 개념 이해

## 4. 주요 작업 항목

### 4-1. Agent 도메인 모델

**파일:** `/backend/app/models/agent.py`

```python
from __future__ import annotations

from datetime import datetime
from typing import Optional
from uuid import UUID, uuid4

from sqlalchemy import (
    Column, String, DateTime, Boolean, ForeignKey,
    Index, Text as SQLText
)
from sqlalchemy.dialects.postgresql import UUID as PGUUID, JSONB
from sqlalchemy.orm import relationship, mapped_column
from sqlalchemy.sql import func

from app.models.base import Base


class Agent(Base):
    """AI 에이전트를 나타내는 독립 Principal.
    
    Attributes:
        id: 에이전트 고유 식별자 (UUID)
        api_key_id: 인증에 사용할 API Key ID (UUID, FK to api_keys)
        organization_id: 소속 조직 ID (UUID)
        scope_profile_id: 접근 범위를 정의하는 Scope Profile ID (UUID, FK to scope_profiles)
            - Scope Profile이 지정되지 않으면 기본 scope 사용 (Task 4-5)
        name: 에이전트 이름 (예: "document-retriever-bot")
        description: 에이전트 설명
        is_disabled: 킬스위치 플래그 (true면 즉시 쓰기 차단)
        created_at: 생성 시간
        updated_at: 마지막 수정 시간
        metadata: 확장 메타데이터 (JSONB)
            - max_queries_per_minute: 속도 제한
            - allowed_document_types: 허용된 문서 타입 리스트
            - tags: 에이전트 태그
    """

    __tablename__ = "agents"

    id = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    api_key_id = mapped_column(PGUUID(as_uuid=True), ForeignKey("api_keys.id"), nullable=False, unique=True, index=True)
    organization_id = mapped_column(PGUUID(as_uuid=True), nullable=False, index=True)
    scope_profile_id = mapped_column(PGUUID(as_uuid=True), nullable=True)  # FK to scope_profiles (Task 4-5)

    name = mapped_column(String(256), nullable=False)
    description = mapped_column(String(512), nullable=True)
    is_disabled = mapped_column(Boolean, default=False, nullable=False, index=True)

    created_at = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)

    metadata = mapped_column(JSONB, default=dict, nullable=False)

    # 관계
    api_key = relationship("APIKey", back_populates="agent", foreign_keys=[api_key_id])

    # 인덱스
    __table_args__ = (
        Index("ix_agents_organization_created", "organization_id", "created_at"),
        Index("ix_agents_is_disabled", "is_disabled"),
    )

    def __repr__(self) -> str:
        return f"<Agent(id={self.id}, name={self.name}, is_disabled={self.is_disabled})>"


class APIKey(Base):
    """API Key 모델 (에이전트 및 사용자 인증용).
    
    기존 api_keys 테이블에 agent_principal_type, scope_profile_id 필드를 추가.
    
    Attributes:
        id: API Key 고유 ID (UUID)
        key_hash: 해시된 API Key (검색용)
        organization_id: 소속 조직 ID (UUID)
        created_by_user_id: 생성자 (사용자 또는 admin)
        principal_type: "user" 또는 "agent" (S2 원칙 ⑤)
        scope_profile_id: 에이전트용 scope profile (agent만 해당, user는 NULL)
        agent: 관계 설정 (에이전트의 역이동 관계)
        is_revoked: 폐기 플래그
        created_at: 생성 시간
        expires_at: 만료 시간 (NULL이면 만료 없음)
    """

    __tablename__ = "api_keys"

    id = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    key_hash = mapped_column(String(256), nullable=False, unique=True, index=True)
    organization_id = mapped_column(PGUUID(as_uuid=True), nullable=False, index=True)
    created_by_user_id = mapped_column(PGUUID(as_uuid=True), nullable=False)

    principal_type = mapped_column(String(32), default="user", nullable=False)  # "user" or "agent"
    scope_profile_id = mapped_column(PGUUID(as_uuid=True), nullable=True)  # agent only

    is_revoked = mapped_column(Boolean, default=False, nullable=False)
    created_at = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    expires_at = mapped_column(DateTime(timezone=True), nullable=True)

    # 역이동 관계
    agent = relationship("Agent", back_populates="api_key", foreign_keys=[id])

    # 인덱스
    __table_args__ = (
        Index("ix_api_keys_organization", "organization_id"),
        Index("ix_api_keys_is_revoked", "is_revoked"),
    )

    def __repr__(self) -> str:
        return f"<APIKey(id={self.id}, principal_type={self.principal_type})>"
```

### 4-2. AccessContext 스키마 및 모델

**파일:** `/backend/app/models/access_context.py`

```python
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Optional, List
from uuid import UUID


@dataclass
class AccessContext:
    """요청 처리 중 전파되는 접근 컨텍스트.
    
    사용자 또는 에이전트 인증 후 생성되며, 모든 API 레이어에서 ACL 검증에 사용됨.
    S2 원칙 ⑤: 에이전트와 사용자는 동등한 principal로 모델링.
    
    Attributes:
        user_id: 현재 사용자 ID (UUID)
        organization_id: 소속 조직 ID (UUID)
        principal_type: "user" 또는 "agent"
        permissions: 권한 리스트 (예: ["read", "write", "admin", "delegate:search", "delegate:write"])
        scope_profile_id: Scope Profile ID (agent만 해당, Task 4-5에서 활용)
        acting_on_behalf_of: 에이전트가 사용자를 대행하는 경우 해당 user_id (선택)
        is_disabled: principal이 비활성화되었는지 여부 (agent kill switch)
    """

    user_id: UUID
    organization_id: UUID
    principal_type: str  # "user" or "agent"
    permissions: List[str] = field(default_factory=list)
    scope_profile_id: Optional[UUID] = None
    acting_on_behalf_of: Optional[UUID] = None
    is_disabled: bool = False

    @property
    def is_agent(self) -> bool:
        """에이전트 여부."""
        return self.principal_type == "agent"

    @property
    def is_user(self) -> bool:
        """사용자 여부."""
        return self.principal_type == "user"

    def has_permission(self, permission: str) -> bool:
        """권한 확인."""
        return permission in self.permissions

    def can_delegate(self) -> bool:
        """위임 권한 확인 (delegate:search or delegate:write)."""
        return any(p.startswith("delegate:") for p in self.permissions)

    def can_delegate_write(self) -> bool:
        """쓰기 위임 권한 확인."""
        return "delegate:write" in self.permissions

    def __repr__(self) -> str:
        return (
            f"<AccessContext(user_id={self.user_id}, principal_type={self.principal_type}, "
            f"acting_on_behalf_of={self.acting_on_behalf_of})>"
        )
```

### 4-3. Delegation 권한 검증 로직

**파일:** `/backend/app/services/delegation_service.py`

```python
from typing import Optional
from uuid import UUID

from app.models.access_context import AccessContext
from app.exceptions import ForbiddenError, BadRequestError


class DelegationService:
    """위임 권한 검증 및 관리 서비스."""

    @staticmethod
    def validate_search_delegation(
        access_context: AccessContext,
        target_user_id: Optional[UUID] = None,
    ) -> None:
        """검색 위임 권한 검증.
        
        에이전트가 사용자를 대행하여 검색할 때 호출.
        
        Args:
            access_context: 요청의 접근 컨텍스트
            target_user_id: 대행 대상 사용자 ID (선택)
        
        Raises:
            ForbiddenError: delegate:search 권한 부족
            BadRequestError: 에이전트가 아닌 경우
        """
        if not access_context.is_agent:
            return  # 일반 사용자는 검증 스킵

        if not access_context.has_permission("delegate:search"):
            raise ForbiddenError(
                f"Agent {access_context.user_id} lacks delegate:search permission"
            )

        if access_context.is_disabled:
            raise ForbiddenError(
                f"Agent {access_context.user_id} is disabled (kill switch active)"
            )

        if target_user_id and access_context.acting_on_behalf_of != target_user_id:
            # Optional: 대행 권한 검증 (추가 로직)
            pass

    @staticmethod
    def validate_write_delegation(
        access_context: AccessContext,
        target_user_id: Optional[UUID] = None,
    ) -> None:
        """쓰기 위임 권한 검증.
        
        에이전트가 사용자를 대행하여 쓰기 작업할 때 호출.
        
        Args:
            access_context: 요청의 접근 컨텍스트
            target_user_id: 대행 대상 사용자 ID (선택)
        
        Raises:
            ForbiddenError: delegate:write 권한 부족 또는 agent 비활성화
        """
        if not access_context.is_agent:
            return  # 일반 사용자는 검증 스킵

        if not access_context.has_permission("delegate:write"):
            raise ForbiddenError(
                f"Agent {access_context.user_id} lacks delegate:write permission"
            )

        if access_context.is_disabled:
            raise ForbiddenError(
                f"Agent {access_context.user_id} is disabled (kill switch active)"
            )

    @staticmethod
    def create_delegation_context(
        agent_access_context: AccessContext,
        on_behalf_of_user_id: UUID,
    ) -> AccessContext:
        """에이전트의 위임 컨텍스트 생성.
        
        에이전트가 특정 사용자를 대행할 때 새로운 access context 반환.
        감시 로그에는 agent가 acting_on_behalf_of 필드로 기록됨.
        
        Args:
            agent_access_context: 에이전트의 접근 컨텍스트
            on_behalf_of_user_id: 대행 대상 사용자 ID
        
        Returns:
            acting_on_behalf_of 필드가 설정된 새 AccessContext
        """
        delegation_context = AccessContext(
            user_id=agent_access_context.user_id,
            organization_id=agent_access_context.organization_id,
            principal_type=agent_access_context.principal_type,
            permissions=agent_access_context.permissions,
            scope_profile_id=agent_access_context.scope_profile_id,
            acting_on_behalf_of=on_behalf_of_user_id,
            is_disabled=agent_access_context.is_disabled,
        )
        return delegation_context
```

### 4-4. Alembic 마이그레이션 스크립트

**파일:** `/backend/alembic/versions/0010_create_agent_and_extend_api_keys.py`

```python
"""Create agents table and extend api_keys with scope_profile_id.

Revision ID: 0010
Revises: 0009
Create Date: 2026-04-17 00:00:00.000000

"""

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# Revision identifiers
revision = "0010"
down_revision = "0009"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # Create agents table
    op.create_table(
        "agents",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("api_key_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("organization_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("scope_profile_id", postgresql.UUID(as_uuid=True), nullable=True),
        sa.Column("name", sa.String(256), nullable=False),
        sa.Column("description", sa.String(512), nullable=True),
        sa.Column("is_disabled", sa.Boolean(), server_default="false", nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.Column("metadata", postgresql.JSONB(astext_type=sa.Text()), server_default=sa.text("'{}'::jsonb"), nullable=False),
        sa.ForeignKeyConstraint(["api_key_id"], ["api_keys.id"], ),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("api_key_id", name="uq_agents_api_key_id"),
    )
    op.create_index("ix_agents_organization_created", "agents", ["organization_id", "created_at"])
    op.create_index("ix_agents_is_disabled", "agents", ["is_disabled"])
    op.create_index("ix_agents_api_key_id", "agents", ["api_key_id"])

    # Extend api_keys table with scope_profile_id and principal_type
    op.add_column("api_keys", sa.Column("principal_type", sa.String(32), server_default="user", nullable=False))
    op.add_column("api_keys", sa.Column("scope_profile_id", postgresql.UUID(as_uuid=True), nullable=True))

    # Create index for principal_type
    op.create_index("ix_api_keys_principal_type", "api_keys", ["principal_type"])

    # Alter audit_logs table to add actor_type and acting_on_behalf_of fields (if not present)
    # Check if columns exist before adding
    try:
        op.add_column(
            "audit_logs",
            sa.Column("actor_type", sa.String(32), server_default="user", nullable=False)
        )
    except Exception:
        pass  # Column may already exist

    try:
        op.add_column(
            "audit_logs",
            sa.Column("acting_on_behalf_of", postgresql.UUID(as_uuid=True), nullable=True)
        )
    except Exception:
        pass  # Column may already exist


def downgrade() -> None:
    op.drop_index("ix_agents_is_disabled", table_name="agents")
    op.drop_index("ix_agents_organization_created", table_name="agents")
    op.drop_index("ix_agents_api_key_id", table_name="agents")
    op.drop_table("agents")

    op.drop_index("ix_api_keys_principal_type", table_name="api_keys")
    op.drop_column("api_keys", "scope_profile_id")
    op.drop_column("api_keys", "principal_type")

    # Conditionally drop audit_logs columns if needed
    # Note: In downgrade, we typically keep audit_logs columns as they don't break
```

### 4-5. Repository 패턴 (Agent 조회/관리)

**파일:** `/backend/app/repositories/agent_repository.py`

```python
"""Agent 및 API Key 저장소."""

from __future__ import annotations

from typing import Optional, List, Dict, Any
from uuid import UUID
from datetime import datetime

from sqlalchemy import and_, or_, select, desc
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.agent import Agent, APIKey
from app.models.access_context import AccessContext


class AgentRepository:
    """Agent 조회 및 관리."""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_id(self, agent_id: UUID) -> Optional[Agent]:
        """ID로 에이전트 조회."""
        stmt = select(Agent).where(Agent.id == agent_id)
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()

    async def get_by_api_key_id(self, api_key_id: UUID) -> Optional[Agent]:
        """API Key ID로 에이전트 조회."""
        stmt = select(Agent).where(Agent.api_key_id == api_key_id)
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()

    async def list_by_organization(
        self,
        organization_id: UUID,
        limit: int = 50,
        offset: int = 0,
    ) -> tuple[List[Agent], int]:
        """조직별 에이전트 목록 조회."""
        filters = [Agent.organization_id == organization_id]

        count_stmt = select(func.count(Agent.id)).where(and_(*filters))
        count_result = await self.db.execute(count_stmt)
        total = count_result.scalar()

        stmt = (
            select(Agent)
            .where(and_(*filters))
            .order_by(desc(Agent.created_at))
            .limit(limit)
            .offset(offset)
        )
        result = await self.db.execute(stmt)
        agents = result.scalars().all()

        return agents, total

    async def create(
        self,
        api_key_id: UUID,
        organization_id: UUID,
        name: str,
        scope_profile_id: Optional[UUID] = None,
        description: Optional[str] = None,
        metadata: Optional[Dict[str, Any]] = None,
    ) -> Agent:
        """새 에이전트 생성."""
        agent = Agent(
            api_key_id=api_key_id,
            organization_id=organization_id,
            scope_profile_id=scope_profile_id,
            name=name,
            description=description,
            metadata=metadata or {},
        )
        self.db.add(agent)
        await self.db.flush()
        return agent

    async def update_is_disabled(self, agent_id: UUID, is_disabled: bool) -> None:
        """에이전트 비활성화 상태 변경 (kill switch)."""
        stmt = select(Agent).where(Agent.id == agent_id)
        result = await self.db.execute(stmt)
        agent = result.scalar_one_or_none()

        if agent:
            agent.is_disabled = is_disabled
            agent.updated_at = datetime.utcnow()
            await self.db.flush()


class APIKeyRepository:
    """API Key 조회 및 관리."""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_hash(self, key_hash: str) -> Optional[APIKey]:
        """해시된 key로 API Key 조회."""
        stmt = select(APIKey).where(
            and_(
                APIKey.key_hash == key_hash,
                APIKey.is_revoked == False,
            )
        )
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()

    async def get_by_id(self, api_key_id: UUID) -> Optional[APIKey]:
        """ID로 API Key 조회."""
        stmt = select(APIKey).where(APIKey.id == api_key_id)
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()

    async def create(
        self,
        key_hash: str,
        organization_id: UUID,
        created_by_user_id: UUID,
        principal_type: str = "user",
        scope_profile_id: Optional[UUID] = None,
        expires_at: Optional[datetime] = None,
    ) -> APIKey:
        """새 API Key 생성."""
        api_key = APIKey(
            key_hash=key_hash,
            organization_id=organization_id,
            created_by_user_id=created_by_user_id,
            principal_type=principal_type,
            scope_profile_id=scope_profile_id,
            expires_at=expires_at,
        )
        self.db.add(api_key)
        await self.db.flush()
        return api_key

    async def revoke(self, api_key_id: UUID) -> None:
        """API Key 폐기."""
        stmt = select(APIKey).where(APIKey.id == api_key_id)
        result = await self.db.execute(stmt)
        api_key = result.scalar_one_or_none()

        if api_key:
            api_key.is_revoked = True
            await self.db.flush()
```

### 4-6. OAuth 토큰 생성 및 검증 (인증 서비스)

**파일:** `/backend/app/services/auth_service.py` (확장)

```python
"""OAuth 2.0 client-credentials 인증 서비스.

폐쇄망 환경에서도 동작하는 로컬 JWT 토큰 생성 (S2 원칙 ⑦).
"""

from __future__ import annotations

import jwt
import hashlib
from datetime import datetime, timedelta
from typing import Optional
from uuid import UUID

from app.config import settings
from app.models.agent import Agent, APIKey
from app.models.access_context import AccessContext
from app.exceptions import UnauthorizedError, BadRequestError
from app.repositories.agent_repository import APIKeyRepository, AgentRepository


class AuthService:
    """인증 및 토큰 관리 서비스."""

    def __init__(self, api_key_repo: APIKeyRepository, agent_repo: AgentRepository):
        self.api_key_repo = api_key_repo
        self.agent_repo = agent_repo

    async def authenticate_client_credentials(
        self,
        client_id: str,  # API Key ID (UUID)
        client_secret: str,  # Raw API Key (hashed for comparison)
    ) -> str:
        """OAuth 2.0 client-credentials 인증 흐름.
        
        Args:
            client_id: API Key ID (UUID string)
            client_secret: Raw API Key
        
        Returns:
            JWT access token
        
        Raises:
            UnauthorizedError: 크레덴셜 유효하지 않음
        """
        try:
            client_id_uuid = UUID(client_id)
        except ValueError:
            raise UnauthorizedError("Invalid client_id format")

        # API Key 해시
        key_hash = hashlib.sha256(client_secret.encode()).hexdigest()

        # DB에서 API Key 조회
        api_key = await self.api_key_repo.get_by_hash(key_hash)

        if not api_key or api_key.id != client_id_uuid:
            raise UnauthorizedError("Invalid client_id or client_secret")

        # 만료 확인
        if api_key.expires_at and api_key.expires_at < datetime.utcnow():
            raise UnauthorizedError("API Key expired")

        # 에이전트 정보 로드 (principal_type이 agent인 경우)
        agent = None
        scope_profile_id = None
        is_disabled = False

        if api_key.principal_type == "agent":
            agent = await self.agent_repo.get_by_api_key_id(api_key.id)
            if not agent:
                raise UnauthorizedError("Agent not found for API Key")

            scope_profile_id = api_key.scope_profile_id or agent.scope_profile_id
            is_disabled = agent.is_disabled

        # JWT 토큰 생성 (폐쇄망 환경에서도 동작)
        token = self._create_jwt_token(
            api_key_id=api_key.id,
            organization_id=api_key.organization_id,
            principal_type=api_key.principal_type,
            scope_profile_id=scope_profile_id,
            is_disabled=is_disabled,
        )

        return token

    def _create_jwt_token(
        self,
        api_key_id: UUID,
        organization_id: UUID,
        principal_type: str,
        scope_profile_id: Optional[UUID] = None,
        is_disabled: bool = False,
    ) -> str:
        """JWT access token 생성 (로컬 발급).
        
        폐쇄망 환경에서 외부 OAuth 서버 없이 동작 (S2 원칙 ⑦).
        """
        now = datetime.utcnow()
        payload = {
            "iss": "mimir-s2",
            "sub": str(api_key_id),
            "organization_id": str(organization_id),
            "principal_type": principal_type,
            "scope_profile_id": str(scope_profile_id) if scope_profile_id else None,
            "is_disabled": is_disabled,
            "iat": now,
            "exp": now + timedelta(hours=1),
        }

        token = jwt.encode(
            payload,
            settings.JWT_SECRET_KEY,
            algorithm=settings.JWT_ALGORITHM,
        )
        return token

    def verify_jwt_token(self, token: str) -> dict:
        """JWT 토큰 검증 및 payload 추출.
        
        Args:
            token: JWT access token
        
        Returns:
            토큰 payload 딕셔너리
        
        Raises:
            UnauthorizedError: 토큰 유효하지 않음
        """
        try:
            payload = jwt.decode(
                token,
                settings.JWT_SECRET_KEY,
                algorithms=[settings.JWT_ALGORITHM],
            )
            return payload
        except jwt.ExpiredSignatureError:
            raise UnauthorizedError("Token expired")
        except jwt.InvalidTokenError:
            raise UnauthorizedError("Invalid token")

    async def create_access_context_from_token(
        self,
        token: str,
    ) -> AccessContext:
        """JWT 토큰에서 AccessContext 생성.
        
        Args:
            token: JWT access token
        
        Returns:
            AccessContext 객체
        """
        payload = self.verify_jwt_token(token)

        principal_type = payload.get("principal_type", "user")
        scope_profile_id = payload.get("scope_profile_id")
        is_disabled = payload.get("is_disabled", False)

        # 권한 설정 (기본값)
        permissions = []
        if principal_type == "agent":
            permissions = ["delegate:search", "delegate:write"]  # Task 4-6에서 동적 설정
        else:
            permissions = ["read", "write"]

        access_context = AccessContext(
            user_id=UUID(payload["sub"]),
            organization_id=UUID(payload["organization_id"]),
            principal_type=principal_type,
            permissions=permissions,
            scope_profile_id=UUID(scope_profile_id) if scope_profile_id else None,
            is_disabled=is_disabled,
        )

        return access_context
```

### 4-7. 감시 로그 통합 (AuditLog 저장)

**파일:** `/backend/app/services/audit_service.py` (확장)

```python
"""감시 로그 기록 서비스.

actor_type (user/agent) 및 acting_on_behalf_of 필드를 필수로 기록 (S2 원칙 ⑥).
"""

from __future__ import annotations

from uuid import UUID
from datetime import datetime
from typing import Optional, Dict, Any

from sqlalchemy.ext.asyncio import AsyncSession

from app.models.audit_log import AuditLog
from app.models.access_context import AccessContext


class AuditService:
    """감시 로그 기록 서비스."""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def log_event(
        self,
        entity_type: str,
        entity_id: UUID,
        action: str,
        access_context: AccessContext,
        changes: Optional[Dict[str, Any]] = None,
    ) -> AuditLog:
        """감시 이벤트 기록.
        
        S2 원칙 ⑥: actor_type 및 acting_on_behalf_of 필드 필수 기록.
        
        Args:
            entity_type: 엔티티 타입 (conversation, document, agent, etc.)
            entity_id: 엔티티 ID
            action: 작업 종류 (create, update, delete, etc.)
            access_context: 요청의 접근 컨텍스트
            changes: 변경 사항 메타데이터
        
        Returns:
            생성된 AuditLog 객체
        """
        audit_log = AuditLog(
            entity_type=entity_type,
            entity_id=entity_id,
            action=action,
            actor_id=access_context.user_id,
            actor_type=access_context.principal_type,  # "user" or "agent"
            acting_on_behalf_of=access_context.acting_on_behalf_of,
            timestamp=datetime.utcnow(),
            changes=changes or {},
        )

        self.db.add(audit_log)
        await self.db.flush()

        return audit_log
```

### 4-8. 단위 테스트

**파일:** `/backend/tests/unit/models/test_agent_models.py`

```python
"""Agent 도메인 모델 단위 테스트."""

import pytest
import hashlib
from uuid import uuid4
from datetime import datetime, timedelta

from sqlalchemy.ext.asyncio import AsyncSession

from app.models.agent import Agent, APIKey
from app.models.access_context import AccessContext
from app.services.delegation_service import DelegationService


@pytest.mark.asyncio
async def test_agent_creation(db_session: AsyncSession):
    """Agent 생성 및 기본값 확인."""
    org_id = uuid4()
    api_key_id = uuid4()

    agent = Agent(
        api_key_id=api_key_id,
        organization_id=org_id,
        name="test-agent",
    )

    db_session.add(agent)
    await db_session.flush()

    assert agent.id is not None
    assert agent.is_disabled is False
    assert agent.api_key_id == api_key_id


@pytest.mark.asyncio
async def test_api_key_creation_for_agent(db_session: AsyncSession):
    """에이전트용 API Key 생성."""
    org_id = uuid4()
    user_id = uuid4()
    scope_profile_id = uuid4()

    raw_key = "test-secret-key-12345"
    key_hash = hashlib.sha256(raw_key.encode()).hexdigest()

    api_key = APIKey(
        key_hash=key_hash,
        organization_id=org_id,
        created_by_user_id=user_id,
        principal_type="agent",
        scope_profile_id=scope_profile_id,
    )

    db_session.add(api_key)
    await db_session.flush()

    assert api_key.id is not None
    assert api_key.principal_type == "agent"
    assert api_key.scope_profile_id == scope_profile_id


def test_access_context_delegation_permission():
    """AccessContext 위임 권한 검증."""
    agent_context = AccessContext(
        user_id=uuid4(),
        organization_id=uuid4(),
        principal_type="agent",
        permissions=["delegate:search", "delegate:write"],
    )

    assert agent_context.can_delegate() is True
    assert agent_context.can_delegate_write() is True


def test_delegation_service_validation():
    """DelegationService 위임 권한 검증."""
    agent_id = uuid4()
    user_id = uuid4()

    # 정상 케이스
    valid_context = AccessContext(
        user_id=agent_id,
        organization_id=uuid4(),
        principal_type="agent",
        permissions=["delegate:search", "delegate:write"],
    )

    try:
        DelegationService.validate_search_delegation(valid_context, user_id)
        DelegationService.validate_write_delegation(valid_context, user_id)
    except Exception:
        pytest.fail("Should not raise exception for valid delegation")

    # 권한 부족 케이스
    invalid_context = AccessContext(
        user_id=agent_id,
        organization_id=uuid4(),
        principal_type="agent",
        permissions=[],
    )

    with pytest.raises(ForbiddenError):
        DelegationService.validate_search_delegation(invalid_context, user_id)

    with pytest.raises(ForbiddenError):
        DelegationService.validate_write_delegation(invalid_context, user_id)


def test_delegation_context_creation():
    """위임 컨텍스트 생성."""
    agent_id = uuid4()
    user_id = uuid4()
    on_behalf_of_user_id = uuid4()

    agent_context = AccessContext(
        user_id=agent_id,
        organization_id=uuid4(),
        principal_type="agent",
        permissions=["delegate:search", "delegate:write"],
    )

    delegation_context = DelegationService.create_delegation_context(
        agent_context,
        on_behalf_of_user_id,
    )

    assert delegation_context.acting_on_behalf_of == on_behalf_of_user_id
    assert delegation_context.principal_type == "agent"
    assert delegation_context.permissions == agent_context.permissions


@pytest.mark.asyncio
async def test_audit_log_with_agent(db_session: AsyncSession):
    """에이전트 작업 감시 로그 기록 (actor_type=agent)."""
    agent_id = uuid4()
    org_id = uuid4()

    access_context = AccessContext(
        user_id=agent_id,
        organization_id=org_id,
        principal_type="agent",
        permissions=["delegate:search"],
    )

    # 감시 로그 생성 (AuditService.log_event 호출 가정)
    # audit_log = await audit_service.log_event(
    #     entity_type="document",
    #     entity_id=uuid4(),
    #     action="search",
    #     access_context=access_context,
    # )

    # assert audit_log.actor_type == "agent"
    # assert audit_log.actor_id == agent_id
```

### 4-9. PII 배치 잡 (conversations 자동 만료 삭제)

**파일:** `/backend/app/jobs/conversation_cleanup.py`

```python
"""PII 배치 잡 — 만료된 대화 데이터 자동 삭제.

Phase 3 이월 항목 (PH3-CARRY-001).
conversations.expires_at 기준으로 만료된 레코드를 주기적으로 삭제한다.

환경변수:
    PII_CLEANUP_ENABLED: "true"(기본) / "false" — 잡 전체 비활성화
    PII_CLEANUP_INTERVAL_HOURS: 실행 주기 (기본: 24)
    PII_CLEANUP_BATCH_SIZE: 1회 삭제 최대 건수 (기본: 1000, OOM 방지)
"""

from __future__ import annotations

import logging
from datetime import datetime, timezone
from typing import Optional

from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.interval import IntervalTrigger

from app.config import settings
from app.db import get_db
from app.audit.emitter import audit_emitter

logger = logging.getLogger(__name__)

_scheduler: Optional[AsyncIOScheduler] = None


async def run_conversation_cleanup() -> int:
    """만료된 대화를 삭제하고 삭제 건수를 반환한다.

    삭제 순서: messages → turns → conversations (FK 제약 준수).
    삭제 전 감사 로그를 기록한다.

    Returns:
        삭제된 conversations 건수.
    """
    if not settings.PII_CLEANUP_ENABLED:
        logger.debug("PII cleanup disabled — skipping")
        return 0

    batch_size = settings.PII_CLEANUP_BATCH_SIZE
    now = datetime.now(timezone.utc)

    deleted_count = 0
    with get_db() as conn:
        with conn.cursor() as cur:
            # 만료된 conversation ID 목록 조회 (batch)
            cur.execute(
                """
                SELECT id FROM conversations
                WHERE expires_at IS NOT NULL
                  AND expires_at < %s
                LIMIT %s
                """,
                (now, batch_size),
            )
            rows = cur.fetchall()
            if not rows:
                return 0

            conv_ids = [str(r[0]) for r in rows]
            placeholders = ",".join(["%s"] * len(conv_ids))

            # 감사 로그 먼저 기록
            for conv_id in conv_ids:
                audit_emitter.emit(
                    event_type="conversation.expired",
                    action="pii_cleanup",
                    actor_id=None,
                    actor_type="system",
                    resource_type="conversation",
                    resource_id=conv_id,
                    result="deleted",
                    metadata={"reason": "expires_at_exceeded", "deleted_at": now.isoformat()},
                )

            # messages → turns → conversations 순서로 삭제
            cur.execute(
                f"""
                DELETE FROM messages
                WHERE turn_id IN (
                    SELECT id FROM turns WHERE conversation_id IN ({placeholders})
                )
                """,
                conv_ids,
            )
            cur.execute(
                f"DELETE FROM turns WHERE conversation_id IN ({placeholders})",
                conv_ids,
            )
            cur.execute(
                f"DELETE FROM conversations WHERE id IN ({placeholders})",
                conv_ids,
            )
            deleted_count = len(conv_ids)
            conn.commit()

    logger.info("PII cleanup: deleted %d expired conversations", deleted_count)
    return deleted_count


def start_cleanup_scheduler() -> None:
    """앱 시작 시 APScheduler를 초기화하고 배치 잡을 등록한다."""
    global _scheduler

    if not settings.PII_CLEANUP_ENABLED:
        logger.info("PII cleanup scheduler disabled by config")
        return

    _scheduler = AsyncIOScheduler()
    _scheduler.add_job(
        run_conversation_cleanup,
        trigger=IntervalTrigger(hours=settings.PII_CLEANUP_INTERVAL_HOURS),
        id="pii_conversation_cleanup",
        name="PII Conversation Cleanup",
        replace_existing=True,
        misfire_grace_time=3600,  # 1시간 이내 지연 허용
    )
    _scheduler.start()
    logger.info(
        "PII cleanup scheduler started — interval=%dh batch_size=%d",
        settings.PII_CLEANUP_INTERVAL_HOURS,
        settings.PII_CLEANUP_BATCH_SIZE,
    )


def stop_cleanup_scheduler() -> None:
    """앱 종료 시 스케줄러를 중단한다."""
    global _scheduler
    if _scheduler and _scheduler.running:
        _scheduler.shutdown(wait=False)
        logger.info("PII cleanup scheduler stopped")
```

**설정 추가 (`/backend/app/config.py`):**

```python
# PII 배치 잡 설정
PII_CLEANUP_ENABLED: bool = env.bool("PII_CLEANUP_ENABLED", default=True)
PII_CLEANUP_INTERVAL_HOURS: int = env.int("PII_CLEANUP_INTERVAL_HOURS", default=24)
PII_CLEANUP_BATCH_SIZE: int = env.int("PII_CLEANUP_BATCH_SIZE", default=1000)
```

**앱 라이프사이클 훅 등록 (`/backend/app/main.py`):**

```python
from app.jobs.conversation_cleanup import start_cleanup_scheduler, stop_cleanup_scheduler

@asynccontextmanager
async def lifespan(app: FastAPI):
    start_cleanup_scheduler()
    yield
    stop_cleanup_scheduler()
```

**단위 테스트 (`/backend/tests/unit/jobs/test_conversation_cleanup.py`):**

```python
"""PII 배치 잡 단위 테스트."""

import pytest
from datetime import datetime, timedelta, timezone
from unittest.mock import patch, MagicMock

from app.jobs.conversation_cleanup import run_conversation_cleanup


@pytest.mark.asyncio
async def test_cleanup_skips_when_disabled(monkeypatch):
    """PII_CLEANUP_ENABLED=false 시 삭제 없이 0 반환."""
    monkeypatch.setattr("app.jobs.conversation_cleanup.settings.PII_CLEANUP_ENABLED", False)
    deleted = await run_conversation_cleanup()
    assert deleted == 0


@pytest.mark.asyncio
async def test_cleanup_deletes_expired_conversations(mock_db_conn):
    """expires_at < NOW() 인 레코드가 삭제된다."""
    past = datetime.now(timezone.utc) - timedelta(days=1)
    # mock_db_conn 에 만료된 conversation 1건 세팅
    mock_db_conn.cursor().__enter__().fetchall.return_value = [("conv-uuid-1",)]

    with patch("app.jobs.conversation_cleanup.get_db", return_value=mock_db_conn):
        deleted = await run_conversation_cleanup()

    assert deleted == 1


@pytest.mark.asyncio
async def test_cleanup_no_op_when_no_expired(mock_db_conn):
    """만료된 레코드 없으면 0 반환."""
    mock_db_conn.cursor().__enter__().fetchall.return_value = []

    with patch("app.jobs.conversation_cleanup.get_db", return_value=mock_db_conn):
        deleted = await run_conversation_cleanup()

    assert deleted == 0
```

---

## 5. 산출물

1. **Agent 도메인 모델** (`/backend/app/models/agent.py`)
   - Agent, APIKey (확장) 클래스 정의
   - 에이전트 활성화/비활성화 상태 관리

2. **AccessContext 스키마** (`/backend/app/models/access_context.py`)
   - 요청 처리 중 전파되는 컨텍스트 객체
   - 위임 권한 확인 메서드 포함

3. **Delegation 서비스** (`/backend/app/services/delegation_service.py`)
   - delegate:search, delegate:write 권한 검증 로직
   - 위임 컨텍스트 생성 헬퍼

4. **Alembic 마이그레이션 스크립트** (`/backend/alembic/versions/0010_*.py`)
   - agents 테이블 생성
   - api_keys 테이블 확장 (principal_type, scope_profile_id)
   - audit_logs 테이블 확장 (actor_type, acting_on_behalf_of)

5. **Repository 계층** (`/backend/app/repositories/agent_repository.py`)
   - AgentRepository, APIKeyRepository 구현
   - 조회, 생성, 업데이트 메서드

6. **OAuth 인증 서비스** (`/backend/app/services/auth_service.py` 확장)
   - client-credentials 흐름 구현
   - JWT 로컬 발급 (폐쇄망 환경 지원)
   - 토큰 검증 및 AccessContext 생성

7. **감시 로그 서비스** (`/backend/app/services/audit_service.py` 확장)
   - actor_type, acting_on_behalf_of 필드 기록

8. **단위 테스트** (`/backend/tests/unit/models/test_agent_models.py`)
   - Agent, APIKey 모델 생성 테스트
   - AccessContext 권한 검증 테스트
   - Delegation 로직 테스트
   - 감시 로그 기록 테스트

9. **PII 배치 잡** (`/backend/app/jobs/conversation_cleanup.py`) _(Phase 3 이월)_
   - `run_conversation_cleanup()` 함수 및 APScheduler 래퍼
   - 설정 값 (`PII_CLEANUP_ENABLED`, `PII_CLEANUP_INTERVAL_HOURS`, `PII_CLEANUP_BATCH_SIZE`)
   - 앱 라이프사이클 훅 등록 (`main.py` lifespan)

10. **PII 배치 잡 단위 테스트** (`/backend/tests/unit/jobs/test_conversation_cleanup.py`)
    - 비활성화 시 no-op 검증
    - 만료 레코드 삭제 검증
    - 빈 결과 no-op 검증

## 6. 완료 기준

1. Agent 및 APIKey 모델이 정규화되었는가?
2. AccessContext 스키마가 principal_type (user/agent) 구분을 지원하는가?
3. Delegation 권한 검증 로직이 delegate:search, delegate:write를 올바르게 처리하는가?
4. OAuth client-credentials 흐름이 구현되었는가?
5. JWT 토큰 생성 및 검증이 폐쇄망 환경에서 동작하는가? (S2 원칙 ⑦)
6. 감시 로그에 actor_type과 acting_on_behalf_of 필드가 기록되는가? (S2 원칙 ⑥)
7. Alembic 마이그레이션이 정상 작동하는가 (upgrade/downgrade)?
8. 모든 단위 테스트가 통과하는가 (`pytest -v`)?
9. mypy 타입 검사를 통과하는가?
10. kill switch 플래그 (is_disabled)가 모든 쓰기 작업 전에 확인되는가?
11. PII 배치 잡이 `expires_at < NOW()` 인 conversations를 삭제하는가? _(Phase 3 이월)_
12. `PII_CLEANUP_ENABLED=false` 시 잡이 실행되지 않는가?
13. 삭제 전 감사 로그(`actor_type="system"`, `action="pii_cleanup"`)가 기록되는가?
14. 배치 잡 단위 테스트 3건이 모두 통과하는가?

## 7. 작업 지침

### 지침 7-1. S2 원칙 ⑤ (AI 에이전트 동등 소비자)

에이전트는 사용자와 동등한 API 소비자이다. principal_type을 "agent"로 설정하여 모든 API에서 동등하게 처리한다. 감시 로그의 actor_type 필드로 누가 작업을 수행했는지 추적 가능하게 한다.

```python
# 사용자와 에이전트 모두 AccessContext를 통해 권한 검증
if access_context.principal_type == "agent":
    DelegationService.validate_write_delegation(access_context)
elif access_context.principal_type == "user":
    # 사용자 권한 검증
    pass
```

### 지침 7-2. S2 원칙 ⑥ (Scope Profile 기반 ACL)

접근 범위를 하드코딩하지 말고, 에이전트의 scope_profile_id를 통해 동적으로 관리한다. scope_profile_id는 Task 4-5에서 정의된 Scope Profile을 참조한다.

```python
# 하드코딩 금지:
if agent.organization_id == current_org:  # X
    pass

# 올바른 방법:
scope_profile = await scope_profile_repo.get_by_id(agent.scope_profile_id)
# scope_profile 기반 ACL 필터링 적용 (Task 4-5)
```

### 지침 7-3. S2 원칙 ⑦ (폐쇄망 환경 지원)

외부 OAuth 서버가 없을 때도 서비스가 동작하도록, JWT 토큰을 로컬에서 생성하고 검증한다. 환경변수로 on/off 가능하게 구성한다.

```python
# settings.py
OAUTH_EXTERNAL_ENABLED = env.bool("OAUTH_EXTERNAL_ENABLED", default=False)

# auth_service.py
if settings.OAUTH_EXTERNAL_ENABLED:
    # 외부 OAuth 서버로 검증
    pass
else:
    # 로컬 JWT 생성/검증
    token = self._create_jwt_token(...)
```

### 지침 7-4. Delegation 컨텍스트

에이전트가 사용자를 대행할 때, acting_on_behalf_of 필드를 설정하여 감시 로그에 기록한다. 이는 "어느 에이전트가 어느 사용자를 대행했는지" 추적을 가능하게 한다.

```python
# 감시 로그
audit_log = AuditLog(
    actor_id=agent_id,        # 작업 수행자 (에이전트)
    actor_type="agent",       # "agent"
    acting_on_behalf_of=user_id,  # 대행 대상 (사용자)
    ...
)
```

### 지침 7-5. Kill Switch 구현

에이전트의 is_disabled 플래그는 쓰기 작업 전에 반드시 확인된다. Task 4-6에서 kill switch API를 통해 즉시 비활성화할 수 있다.

```python
if access_context.is_disabled:
    raise ForbiddenError("Agent is disabled (kill switch active)")
```

### 지침 7-6. API Key 해싱

API Key의 원본 값은 DB에 저장하지 않는다. 대신 SHA256 해시로 저장하고, 인증 시에만 해시 비교를 수행한다.

```python
import hashlib

raw_key = "test-secret-key-12345"
key_hash = hashlib.sha256(raw_key.encode()).hexdigest()

api_key = APIKey(key_hash=key_hash, ...)
```

### 지침 7-7. 마이그레이션 버전 관리

Alembic 마이그레이션의 down_revision을 정확하게 설정하여 이전 Phase의 최신 마이그레이션과 연결한다. 현재 task는 기존 마이그레이션 이후의 버전을 사용한다.

### 지침 7-8. 타입 안정성

AccessContext, Agent, APIKey, AuditLog 등 모든 모델과 서비스에서 UUID, Literal 타입을 명시적으로 사용하여 mypy 검사를 통과하도록 한다.

```python
# 좋은 예
def log_event(
    self,
    action: Literal["create", "update", "delete"],
    actor_type: Literal["user", "agent"],
) -> AuditLog:
    pass
```
