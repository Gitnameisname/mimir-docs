# Task 4-6. Scope Profile CRUD API + API Key 바인딩 + Kill Switch

## 1. 작업 목적

Scope Profile을 관리자가 API를 통해 관리할 수 있도록 FastAPI 엔드포인트를 구현한다. 또한 에이전트용 API Key 발급 시 scope_profile_id를 바인딩하고, kill switch API를 통해 문제 있는 에이전트를 즉시 비활성화할 수 있는 기능을 제공한다. 이를 통해 Scope Profile 기반의 ACL과 kill switch를 완전히 운영 가능하게 만든다.

## 2. 작업 범위

### 포함 범위

1. **Scope Profile CRUD API (FastAPI)**
   - 엔드포인트: POST/GET/PUT/DELETE /api/v1/admin/scope-profiles
   - admin 역할 필수 (RequestScope 검증)
   - 요청/응답 Pydantic 모델 정의

2. **POST /api/v1/admin/scope-profiles (생성)**
   - 요청 본문:
     ```json
     {
       "name": "string (필수)",
       "description": "string (선택)",
       "scope_definitions": [
         {
           "scope_name": "string",
           "acl_filter": {
             "and": [...]
           },
           "description": "string (선택)"
         }
       ]
     }
     ```
   - 응답: 생성된 ScopeProfile 객체 + scope_definitions

3. **GET /api/v1/admin/scope-profiles (목록 조회)**
   - 쿼리 파라미터: page (기본 1), page_size (기본 20)
   - 응답: 페이지네이션된 Scope Profile 목록 + 메타데이터

4. **GET /api/v1/admin/scope-profiles/{id} (상세 조회)**
   - 응답: 해당 Scope Profile + 모든 scope_definitions

5. **PUT /api/v1/admin/scope-profiles/{id} (수정)**
   - 요청 본문: name, description, scope_definitions (부분 수정 가능)
   - 응답: 수정된 Scope Profile 객체

6. **DELETE /api/v1/admin/scope-profiles/{id} (삭제)**
   - 논리 삭제 (soft delete) 또는 물리 삭제 정책 결정
   - 응답: 204 No Content 또는 {success: true}

7. **API Key + Scope Profile 바인딩**
   - API Key 발급 시 (Task 4-4의 AuthService) scope_profile_id 지정 필수 (agent 타입)
   - APIKey 모델에 이미 scope_profile_id 필드 포함 (Task 4-4 완료 가정)
   - Scope Profile 미지정 시, 에이전트 생성 시 default scope 사용

8. **Kill Switch API**
   - 엔드포인트:
     - POST /api/v1/admin/agents/{agent_id}/kill-switch (활성화)
     - DELETE /api/v1/admin/agents/{agent_id}/kill-switch (해제)
   - admin 역할 필수

9. **Kill Switch 동작**
   - agents.is_disabled = true로 설정
   - 모든 읽기/쓰기 요청 시 is_disabled 확인 → true이면 ForbiddenError 발생
   - 캐시 TTL: 1분 이하 (즉시 반영에 가까움)

10. **Kill Switch 감시 로그**
    - 활성화/해제 이벤트 감시 로그 기록
    - actor_id (admin), action ("kill_switch_enable" / "kill_switch_disable")

11. **에러 처리 및 검증**
    - 범위: 조직별 리소스 검증 (다른 조직의 Scope Profile 접근 불가)
    - 권한: admin 역할 검증
    - 404: 리소스 없음
    - 400: 잘못된 FilterExpression
    - 403: 권한 부족

12. **단위/통합 테스트**
    - CRUD 엔드포인트 테스트
    - API Key 바인딩 테스트
    - Kill Switch 동작 테스트
    - 권한 검증 테스트
    - scope_profile_id 캐시 TTL 테스트

### 제외 범위

- Agent Principal Type 및 Delegation 모델 (Task 4-4)
- FilterExpression 파서 (Task 4-5)
- Scope Profile CRUD API 이외의 리소스 (예: scope_definitions 단독 CRUD)
- MCP Server 구현 (FG4.1)

## 3. 선행 조건

- Phase 1, Phase 2, Phase 3 완료
- Task 4-4 (Agent Principal Type) 완료
- Task 4-5 (Scope Profile 모델) 완료
- FastAPI 애플리케이션 구조 이해
- Pydantic v2 설치
- 기존 RequestScope 검증 로직 분석 완료

## 4. 주요 작업 항목

### 4-1. Pydantic 요청/응답 모델

**파일:** `/backend/app/schemas/scope_profile_schema.py`

```python
"""Scope Profile 및 Kill Switch API 스키마."""

from __future__ import annotations

from typing import Optional, List, Dict, Any
from uuid import UUID
from datetime import datetime

from pydantic import BaseModel, Field, validator


class ScopeDefinitionRequest(BaseModel):
    """ScopeDefinition 생성/수정 요청."""

    scope_name: str = Field(..., min_length=1, max_length=256)
    acl_filter: Dict[str, Any] = Field(...)  # FilterExpression (JSON)
    description: Optional[str] = Field(None, max_length=512)

    @validator("acl_filter")
    def validate_acl_filter(cls, v):
        """FilterExpression 기본 검증 (심화 검증은 FilterParser에서)."""
        if not isinstance(v, dict):
            raise ValueError("acl_filter must be a dictionary")
        if "and" not in v and "or" not in v:
            raise ValueError("acl_filter must contain 'and' or 'or'")
        return v


class ScopeDefinitionResponse(BaseModel):
    """ScopeDefinition 응답."""

    id: UUID
    scope_profile_id: UUID
    scope_name: str
    acl_filter: Dict[str, Any]
    description: Optional[str]
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True


class ScopeProfileCreateRequest(BaseModel):
    """Scope Profile 생성 요청."""

    name: str = Field(..., min_length=1, max_length=256)
    description: Optional[str] = Field(None, max_length=512)
    scope_definitions: Optional[List[ScopeDefinitionRequest]] = None


class ScopeProfileUpdateRequest(BaseModel):
    """Scope Profile 수정 요청 (부분 수정 가능)."""

    name: Optional[str] = Field(None, min_length=1, max_length=256)
    description: Optional[str] = Field(None, max_length=512)
    scope_definitions: Optional[List[ScopeDefinitionRequest]] = None


class ScopeProfileResponse(BaseModel):
    """Scope Profile 응답."""

    id: UUID
    organization_id: UUID
    name: str
    description: Optional[str]
    scope_definitions: List[ScopeDefinitionResponse]
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True


class ScopeProfileListResponse(BaseModel):
    """Scope Profile 목록 조회 응답 (페이지네이션)."""

    items: List[ScopeProfileResponse]
    total: int
    page: int
    page_size: int

    @property
    def total_pages(self) -> int:
        """총 페이지 수."""
        return (self.total + self.page_size - 1) // self.page_size


class KillSwitchRequest(BaseModel):
    """Kill Switch 활성화 요청 (선택: 설명 메모)."""

    reason: Optional[str] = Field(None, max_length=512)


class KillSwitchResponse(BaseModel):
    """Kill Switch 상태 응답."""

    agent_id: UUID
    is_disabled: bool
    reason: Optional[str]
    activated_at: Optional[datetime]


class ErrorResponse(BaseModel):
    """API 에러 응답."""

    error: str
    message: str
    details: Optional[Dict[str, Any]] = None
```

### 4-2. Scope Profile CRUD API 구현

**파일:** `/backend/app/routers/admin_scope_profiles.py`

```python
"""Scope Profile CRUD API 엔드포인트."""

from __future__ import annotations

from typing import Optional
from uuid import UUID

from fastapi import APIRouter, Depends, Query, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.access_context import AccessContext
from app.models.scope_profile import ScopeProfile, ScopeDefinition
from app.repositories.scope_profile_repository import (
    ScopeProfileRepository,
    ScopeDefinitionRepository,
)
from app.schemas.scope_profile_schema import (
    ScopeProfileCreateRequest,
    ScopeProfileUpdateRequest,
    ScopeProfileResponse,
    ScopeProfileListResponse,
)
from app.dependencies import (
    get_db,
    get_access_context,
    require_admin,
)
from app.exceptions import ForbiddenError, NotFoundError, BadRequestError


router = APIRouter(prefix="/api/v1/admin", tags=["admin-scope-profiles"])


@router.post(
    "/scope-profiles",
    response_model=ScopeProfileResponse,
    status_code=status.HTTP_201_CREATED,
)
async def create_scope_profile(
    request: ScopeProfileCreateRequest,
    access_context: AccessContext = Depends(get_access_context),
    db: AsyncSession = Depends(get_db),
) -> ScopeProfileResponse:
    """Scope Profile 생성 (admin 필수).
    
    Args:
        request: 생성 요청
        access_context: 요청 사용자 컨텍스트
        db: DB 세션
    
    Returns:
        생성된 Scope Profile
    
    Raises:
        ForbiddenError: admin 권한 부족
    """
    # Admin 권한 검증
    require_admin(access_context)

    profile_repo = ScopeProfileRepository(db)
    definition_repo = ScopeDefinitionRepository(db)

    # Scope Profile 생성
    scope_profile = await profile_repo.create(
        organization_id=access_context.organization_id,
        name=request.name,
        description=request.description,
    )

    # Scope Definitions 추가
    if request.scope_definitions:
        for def_req in request.scope_definitions:
            await definition_repo.create(
                scope_profile_id=scope_profile.id,
                scope_name=def_req.scope_name,
                acl_filter=def_req.acl_filter,
                description=def_req.description,
            )

    # 관계 로드 (SQLAlchemy selectin이 설정되어 있음)
    await db.refresh(scope_profile)

    return ScopeProfileResponse.from_orm(scope_profile)


@router.get(
    "/scope-profiles",
    response_model=ScopeProfileListResponse,
)
async def list_scope_profiles(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    access_context: AccessContext = Depends(get_access_context),
    db: AsyncSession = Depends(get_db),
) -> ScopeProfileListResponse:
    """Scope Profile 목록 조회 (admin 필수).
    
    Args:
        page: 페이지 번호 (기본 1)
        page_size: 페이지 크기 (기본 20)
        access_context: 요청 사용자 컨텍스트
        db: DB 세션
    
    Returns:
        페이지네이션된 Scope Profile 목록
    
    Raises:
        ForbiddenError: admin 권한 부족
    """
    require_admin(access_context)

    profile_repo = ScopeProfileRepository(db)

    offset = (page - 1) * page_size
    profiles, total = await profile_repo.list_by_organization(
        organization_id=access_context.organization_id,
        limit=page_size,
        offset=offset,
    )

    items = [ScopeProfileResponse.from_orm(p) for p in profiles]

    return ScopeProfileListResponse(
        items=items,
        total=total,
        page=page,
        page_size=page_size,
    )


@router.get(
    "/scope-profiles/{scope_profile_id}",
    response_model=ScopeProfileResponse,
)
async def get_scope_profile(
    scope_profile_id: UUID,
    access_context: AccessContext = Depends(get_access_context),
    db: AsyncSession = Depends(get_db),
) -> ScopeProfileResponse:
    """Scope Profile 상세 조회 (admin 필수).
    
    Args:
        scope_profile_id: Scope Profile ID
        access_context: 요청 사용자 컨텍스트
        db: DB 세션
    
    Returns:
        Scope Profile + 모든 scope_definitions
    
    Raises:
        ForbiddenError: admin 권한 부족 또는 다른 조직 리소스 접근
        NotFoundError: Scope Profile 없음
    """
    require_admin(access_context)

    profile_repo = ScopeProfileRepository(db)
    scope_profile = await profile_repo.get_by_id(scope_profile_id)

    if not scope_profile:
        raise NotFoundError(f"Scope Profile {scope_profile_id} not found")

    # 조직 검증 (다른 조직의 Scope Profile 접근 불가)
    if scope_profile.organization_id != access_context.organization_id:
        raise ForbiddenError(
            f"Cannot access Scope Profile from different organization"
        )

    return ScopeProfileResponse.from_orm(scope_profile)


@router.put(
    "/scope-profiles/{scope_profile_id}",
    response_model=ScopeProfileResponse,
)
async def update_scope_profile(
    scope_profile_id: UUID,
    request: ScopeProfileUpdateRequest,
    access_context: AccessContext = Depends(get_access_context),
    db: AsyncSession = Depends(get_db),
) -> ScopeProfileResponse:
    """Scope Profile 수정 (admin 필수).
    
    Args:
        scope_profile_id: Scope Profile ID
        request: 수정 요청
        access_context: 요청 사용자 컨텍스트
        db: DB 세션
    
    Returns:
        수정된 Scope Profile
    
    Raises:
        ForbiddenError: admin 권한 부족 또는 다른 조직 리소스 접근
        NotFoundError: Scope Profile 없음
    """
    require_admin(access_context)

    profile_repo = ScopeProfileRepository(db)
    definition_repo = ScopeDefinitionRepository(db)

    scope_profile = await profile_repo.get_by_id(scope_profile_id)

    if not scope_profile:
        raise NotFoundError(f"Scope Profile {scope_profile_id} not found")

    if scope_profile.organization_id != access_context.organization_id:
        raise ForbiddenError(
            f"Cannot access Scope Profile from different organization"
        )

    # name, description 수정
    if request.name is not None:
        scope_profile.name = request.name
    if request.description is not None:
        scope_profile.description = request.description

    # scope_definitions 수정 (전체 교체)
    if request.scope_definitions is not None:
        # 기존 정의 삭제
        existing_defs = await definition_repo.list_by_scope_profile(scope_profile_id)
        for def_obj in existing_defs:
            await db.delete(def_obj)

        # 새 정의 추가
        for def_req in request.scope_definitions:
            await definition_repo.create(
                scope_profile_id=scope_profile_id,
                scope_name=def_req.scope_name,
                acl_filter=def_req.acl_filter,
                description=def_req.description,
            )

    await db.flush()
    await db.refresh(scope_profile)

    return ScopeProfileResponse.from_orm(scope_profile)


@router.delete(
    "/scope-profiles/{scope_profile_id}",
    status_code=status.HTTP_204_NO_CONTENT,
)
async def delete_scope_profile(
    scope_profile_id: UUID,
    access_context: AccessContext = Depends(get_access_context),
    db: AsyncSession = Depends(get_db),
) -> None:
    """Scope Profile 삭제 (admin 필수, 물리 삭제).
    
    Args:
        scope_profile_id: Scope Profile ID
        access_context: 요청 사용자 컨텍스트
        db: DB 세션
    
    Raises:
        ForbiddenError: admin 권한 부족 또는 다른 조직 리소스 접근
        NotFoundError: Scope Profile 없음
    """
    require_admin(access_context)

    profile_repo = ScopeProfileRepository(db)
    scope_profile = await profile_repo.get_by_id(scope_profile_id)

    if not scope_profile:
        raise NotFoundError(f"Scope Profile {scope_profile_id} not found")

    if scope_profile.organization_id != access_context.organization_id:
        raise ForbiddenError(
            f"Cannot access Scope Profile from different organization"
        )

    await db.delete(scope_profile)
    await db.flush()
```

### 4-3. Kill Switch API 구현

**파일:** `/backend/app/routers/admin_agents.py` (신규 또는 확장)

```python
"""Agent 관리 API 엔드포인트 (Kill Switch)."""

from __future__ import annotations

from uuid import UUID
from datetime import datetime

from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.access_context import AccessContext
from app.repositories.agent_repository import AgentRepository
from app.services.audit_service import AuditService
from app.schemas.scope_profile_schema import (
    KillSwitchRequest,
    KillSwitchResponse,
)
from app.dependencies import (
    get_db,
    get_access_context,
    require_admin,
)
from app.exceptions import ForbiddenError, NotFoundError


router = APIRouter(prefix="/api/v1/admin", tags=["admin-agents"])


@router.post(
    "/agents/{agent_id}/kill-switch",
    response_model=KillSwitchResponse,
)
async def enable_kill_switch(
    agent_id: UUID,
    request: KillSwitchRequest,
    access_context: AccessContext = Depends(get_access_context),
    db: AsyncSession = Depends(get_db),
) -> KillSwitchResponse:
    """에이전트 Kill Switch 활성화 (admin 필수).
    
    is_disabled를 true로 설정하여 에이전트 모든 작업 즉시 차단.
    
    Args:
        agent_id: 에이전트 ID
        request: Kill Switch 활성화 요청 (선택: reason)
        access_context: 요청 사용자 컨텍스트
        db: DB 세션
    
    Returns:
        Kill Switch 상태
    
    Raises:
        ForbiddenError: admin 권한 부족 또는 다른 조직 리소스 접근
        NotFoundError: 에이전트 없음
    """
    require_admin(access_context)

    agent_repo = AgentRepository(db)
    agent = await agent_repo.get_by_id(agent_id)

    if not agent:
        raise NotFoundError(f"Agent {agent_id} not found")

    # 조직 검증
    if agent.organization_id != access_context.organization_id:
        raise ForbiddenError("Cannot access agent from different organization")

    # Kill Switch 활성화
    await agent_repo.update_is_disabled(agent_id, True)

    # 감시 로그 기록
    audit_service = AuditService(db)
    await audit_service.log_event(
        entity_type="agent",
        entity_id=agent_id,
        action="kill_switch_enable",
        access_context=access_context,
        changes={"reason": request.reason} if request.reason else {},
    )

    await db.commit()

    return KillSwitchResponse(
        agent_id=agent_id,
        is_disabled=True,
        reason=request.reason,
        activated_at=datetime.utcnow(),
    )


@router.delete(
    "/agents/{agent_id}/kill-switch",
    response_model=KillSwitchResponse,
)
async def disable_kill_switch(
    agent_id: UUID,
    access_context: AccessContext = Depends(get_access_context),
    db: AsyncSession = Depends(get_db),
) -> KillSwitchResponse:
    """에이전트 Kill Switch 해제 (admin 필수).
    
    is_disabled를 false로 설정하여 에이전트 작업 재개.
    
    Args:
        agent_id: 에이전트 ID
        access_context: 요청 사용자 컨텍스트
        db: DB 세션
    
    Returns:
        Kill Switch 상태
    
    Raises:
        ForbiddenError: admin 권한 부족 또는 다른 조직 리소스 접근
        NotFoundError: 에이전트 없음
    """
    require_admin(access_context)

    agent_repo = AgentRepository(db)
    agent = await agent_repo.get_by_id(agent_id)

    if not agent:
        raise NotFoundError(f"Agent {agent_id} not found")

    # 조직 검증
    if agent.organization_id != access_context.organization_id:
        raise ForbiddenError("Cannot access agent from different organization")

    # Kill Switch 해제
    await agent_repo.update_is_disabled(agent_id, False)

    # 감시 로그 기록
    audit_service = AuditService(db)
    await audit_service.log_event(
        entity_type="agent",
        entity_id=agent_id,
        action="kill_switch_disable",
        access_context=access_context,
    )

    await db.commit()

    return KillSwitchResponse(
        agent_id=agent_id,
        is_disabled=False,
        reason=None,
        activated_at=None,
    )
```

### 4-4. Kill Switch 캐시 및 미들웨어

**파일:** `/backend/app/middleware/kill_switch_middleware.py`

```python
"""Kill Switch 상태 캐시 및 미들웨어.

TTL 1분 이하로 에이전트 비활성화 상태를 빠르게 반영.
"""

from __future__ import annotations

from typing import Optional
from uuid import UUID
from datetime import datetime, timedelta
from functools import lru_cache
import asyncio

from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

from app.repositories.agent_repository import AgentRepository
from app.exceptions import ForbiddenError


class KillSwitchCache:
    """Kill Switch 상태 캐시 (TTL 기반)."""

    def __init__(self, ttl_seconds: int = 60):
        self.ttl_seconds = ttl_seconds
        self._cache: dict[UUID, tuple[bool, datetime]] = {}

    async def is_disabled(self, agent_id: UUID, agent_repo: AgentRepository) -> bool:
        """에이전트 비활성화 여부 조회 (캐시 활용).
        
        Args:
            agent_id: 에이전트 ID
            agent_repo: Agent 저장소
        
        Returns:
            is_disabled 상태
        """
        now = datetime.utcnow()

        # 캐시 확인
        if agent_id in self._cache:
            is_disabled, cached_at = self._cache[agent_id]
            if (now - cached_at).total_seconds() < self.ttl_seconds:
                return is_disabled

        # 캐시 만료 또는 없음 → DB 조회
        agent = await agent_repo.get_by_id(agent_id)
        if not agent:
            return False  # 에이전트 없으면 정상 처리 (별도 에러 처리)

        is_disabled = agent.is_disabled
        self._cache[agent_id] = (is_disabled, now)

        return is_disabled

    def invalidate(self, agent_id: UUID) -> None:
        """특정 에이전트 캐시 무효화 (kill switch 변경 시 호출)."""
        if agent_id in self._cache:
            del self._cache[agent_id]


# 글로벌 인스턴스
kill_switch_cache = KillSwitchCache(ttl_seconds=60)


class KillSwitchMiddleware(BaseHTTPMiddleware):
    """Kill Switch 미들웨어.
    
    인증된 요청에서 에이전트인 경우, 모든 작업 전에 is_disabled 확인.
    """

    async def dispatch(self, request: Request, call_next):
        """요청 처리 전 kill switch 확인."""
        # access_context는 Depends(get_access_context)로 주입되므로,
        # 미들웨어에서는 직접 접근 어려움.
        # 대신 endpoint 핸들러에서 확인하거나, 별도 의존성으로 처리.

        # 현재는 endpoint handler에서 처리하기로 변경
        # (FastAPI 의존성이 더 유연함)

        response = await call_next(request)
        return response
```

### 4-5. API Key 바인딩 (기존 코드 확장)

**파일:** `/backend/app/services/api_key_service.py` (신규 또는 확장)

```python
"""API Key 관리 서비스 (scope_profile_id 바인딩)."""

from __future__ import annotations

import secrets
import hashlib
from uuid import UUID
from typing import Optional
from datetime import datetime, timedelta

from sqlalchemy.ext.asyncio import AsyncSession

from app.models.agent import APIKey
from app.models.access_context import AccessContext
from app.repositories.agent_repository import APIKeyRepository
from app.exceptions import BadRequestError


class APIKeyService:
    """API Key 생성 및 관리."""

    def __init__(self, db: AsyncSession):
        self.db = db
        self.api_key_repo = APIKeyRepository(db)

    async def create_api_key_for_agent(
        self,
        organization_id: UUID,
        created_by_user_id: UUID,
        scope_profile_id: Optional[UUID] = None,
        expires_at: Optional[datetime] = None,
    ) -> tuple[str, APIKey]:
        """에이전트용 API Key 생성.
        
        Args:
            organization_id: 조직 ID
            created_by_user_id: 생성자 ID
            scope_profile_id: Scope Profile ID (에이전트 권한 범위)
            expires_at: 만료 시간 (선택)
        
        Returns:
            (raw_api_key, api_key_obj) 튜플
            raw_api_key는 클라이언트에게만 한 번 반환됨.
        
        Raises:
            BadRequestError: scope_profile_id 없거나 유효하지 않음
        """
        if not scope_profile_id:
            raise BadRequestError(
                "scope_profile_id is required for agent API Key"
            )

        # TODO: scope_profile_id 유효성 검증 (Task 4-5에서 구현)
        # scope_profile_repo = ScopeProfileRepository(self.db)
        # scope_profile = await scope_profile_repo.get_by_id(scope_profile_id)
        # if not scope_profile or scope_profile.organization_id != organization_id:
        #     raise BadRequestError("Invalid scope_profile_id")

        # Raw API Key 생성
        raw_key = secrets.token_urlsafe(32)  # 44자 정도
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()

        # DB에 저장 (해시된 값)
        api_key = await self.api_key_repo.create(
            key_hash=key_hash,
            organization_id=organization_id,
            created_by_user_id=created_by_user_id,
            principal_type="agent",
            scope_profile_id=scope_profile_id,
            expires_at=expires_at,
        )

        return raw_key, api_key

    async def create_api_key_for_user(
        self,
        organization_id: UUID,
        created_by_user_id: UUID,
        expires_at: Optional[datetime] = None,
    ) -> tuple[str, APIKey]:
        """사용자용 API Key 생성 (legacy/개발용).
        
        Args:
            organization_id: 조직 ID
            created_by_user_id: 생성자 ID
            expires_at: 만료 시간 (선택)
        
        Returns:
            (raw_api_key, api_key_obj) 튜플
        """
        raw_key = secrets.token_urlsafe(32)
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()

        api_key = await self.api_key_repo.create(
            key_hash=key_hash,
            organization_id=organization_id,
            created_by_user_id=created_by_user_id,
            principal_type="user",
            scope_profile_id=None,
            expires_at=expires_at,
        )

        return raw_key, api_key
```

### 4-6. 단위/통합 테스트

**파일:** `/backend/tests/integration/test_scope_profile_api.py`

```python
"""Scope Profile CRUD API 통합 테스트."""

import pytest
from uuid import uuid4
from httpx import AsyncClient

from app.main import app  # FastAPI 앱
from app.models.scope_profile import ScopeProfile
from app.repositories.scope_profile_repository import ScopeProfileRepository


@pytest.mark.asyncio
async def test_create_scope_profile(
    async_client: AsyncClient,
    admin_access_token: str,
    organization_id: uuid4,
):
    """Scope Profile 생성 (admin 필수)."""
    payload = {
        "name": "test-profile",
        "description": "Test Profile",
        "scope_definitions": [
            {
                "scope_name": "team_search",
                "acl_filter": {
                    "and": [
                        {"field": "organization_id", "op": "eq", "value": "$ctx.organization_id"},
                    ]
                },
            }
        ],
    }

    response = await async_client.post(
        "/api/v1/admin/scope-profiles",
        json=payload,
        headers={"Authorization": f"Bearer {admin_access_token}"},
    )

    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "test-profile"
    assert len(data["scope_definitions"]) == 1
    assert data["scope_definitions"][0]["scope_name"] == "team_search"


@pytest.mark.asyncio
async def test_list_scope_profiles(
    async_client: AsyncClient,
    admin_access_token: str,
):
    """Scope Profile 목록 조회."""
    response = await async_client.get(
        "/api/v1/admin/scope-profiles?page=1&page_size=20",
        headers={"Authorization": f"Bearer {admin_access_token}"},
    )

    assert response.status_code == 200
    data = response.json()
    assert "items" in data
    assert "total" in data
    assert "page" in data


@pytest.mark.asyncio
async def test_get_scope_profile(
    async_client: AsyncClient,
    admin_access_token: str,
    scope_profile_id: uuid4,
):
    """Scope Profile 상세 조회."""
    response = await async_client.get(
        f"/api/v1/admin/scope-profiles/{scope_profile_id}",
        headers={"Authorization": f"Bearer {admin_access_token}"},
    )

    assert response.status_code == 200
    data = response.json()
    assert data["id"] == str(scope_profile_id)


@pytest.mark.asyncio
async def test_update_scope_profile(
    async_client: AsyncClient,
    admin_access_token: str,
    scope_profile_id: uuid4,
):
    """Scope Profile 수정."""
    payload = {
        "name": "updated-profile",
        "description": "Updated Description",
    }

    response = await async_client.put(
        f"/api/v1/admin/scope-profiles/{scope_profile_id}",
        json=payload,
        headers={"Authorization": f"Bearer {admin_access_token}"},
    )

    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "updated-profile"


@pytest.mark.asyncio
async def test_delete_scope_profile(
    async_client: AsyncClient,
    admin_access_token: str,
    scope_profile_id: uuid4,
):
    """Scope Profile 삭제."""
    response = await async_client.delete(
        f"/api/v1/admin/scope-profiles/{scope_profile_id}",
        headers={"Authorization": f"Bearer {admin_access_token}"},
    )

    assert response.status_code == 204


@pytest.mark.asyncio
async def test_create_scope_profile_non_admin_forbidden(
    async_client: AsyncClient,
    user_access_token: str,
):
    """일반 사용자의 Scope Profile 생성 금지."""
    payload = {
        "name": "test-profile",
        "description": "Test Profile",
    }

    response = await async_client.post(
        "/api/v1/admin/scope-profiles",
        json=payload,
        headers={"Authorization": f"Bearer {user_access_token}"},
    )

    assert response.status_code == 403


@pytest.mark.asyncio
async def test_enable_kill_switch(
    async_client: AsyncClient,
    admin_access_token: str,
    agent_id: uuid4,
):
    """Kill Switch 활성화."""
    payload = {"reason": "Suspicious activity detected"}

    response = await async_client.post(
        f"/api/v1/admin/agents/{agent_id}/kill-switch",
        json=payload,
        headers={"Authorization": f"Bearer {admin_access_token}"},
    )

    assert response.status_code == 200
    data = response.json()
    assert data["agent_id"] == str(agent_id)
    assert data["is_disabled"] is True
    assert data["reason"] == "Suspicious activity detected"


@pytest.mark.asyncio
async def test_disable_kill_switch(
    async_client: AsyncClient,
    admin_access_token: str,
    agent_id: uuid4,
):
    """Kill Switch 해제."""
    # 먼저 활성화
    payload = {"reason": "Test"}
    await async_client.post(
        f"/api/v1/admin/agents/{agent_id}/kill-switch",
        json=payload,
        headers={"Authorization": f"Bearer {admin_access_token}"},
    )

    # 해제
    response = await async_client.delete(
        f"/api/v1/admin/agents/{agent_id}/kill-switch",
        headers={"Authorization": f"Bearer {admin_access_token}"},
    )

    assert response.status_code == 200
    data = response.json()
    assert data["is_disabled"] is False


@pytest.mark.asyncio
async def test_kill_switch_blocks_agent_request(
    async_client: AsyncClient,
    agent_access_token: str,
    agent_id: uuid4,
    admin_access_token: str,
):
    """Kill Switch 활성화 후 에이전트 요청 차단."""
    # Kill Switch 활성화
    await async_client.post(
        f"/api/v1/admin/agents/{agent_id}/kill-switch",
        json={"reason": "Test"},
        headers={"Authorization": f"Bearer {admin_access_token}"},
    )

    # 에이전트 API 요청 시도
    response = await async_client.get(
        "/api/v1/documents/search",  # 예시 엔드포인트
        headers={"Authorization": f"Bearer {agent_access_token}"},
    )

    # 모든 작업이 차단됨
    assert response.status_code == 403  # ForbiddenError
```

## 5. 산출물

1. **Scope Profile CRUD API 스키마** (`/backend/app/schemas/scope_profile_schema.py`)
   - Pydantic 요청/응답 모델
   - 검증 로직 (FilterExpression 기본 검증)

2. **Scope Profile CRUD API 엔드포인트** (`/backend/app/routers/admin_scope_profiles.py`)
   - POST/GET/PUT/DELETE /api/v1/admin/scope-profiles
   - 페이지네이션, 조직 검증, admin 권한 검증

3. **Kill Switch API 엔드포인트** (`/backend/app/routers/admin_agents.py`)
   - POST /api/v1/admin/agents/{agent_id}/kill-switch (활성화)
   - DELETE /api/v1/admin/agents/{agent_id}/kill-switch (해제)
   - 감시 로그 기록

4. **Kill Switch 캐시 및 미들웨어** (`/backend/app/middleware/kill_switch_middleware.py`)
   - TTL 1분 기반 캐시
   - 빠른 반영

5. **API Key 관리 서비스** (`/backend/app/services/api_key_service.py`)
   - 에이전트용 API Key 생성 (scope_profile_id 바인딩)
   - 사용자용 API Key 생성 (legacy)

6. **단위/통합 테스트** (`/backend/tests/integration/test_scope_profile_api.py`)
   - CRUD 엔드포인트 테스트
   - Kill Switch 동작 테스트
   - 권한 검증 테스트

## 6. 완료 기준

1. POST /api/v1/admin/scope-profiles가 정상 동작하는가?
2. GET /api/v1/admin/scope-profiles (목록)이 페이지네이션을 지원하는가?
3. GET /api/v1/admin/scope-profiles/{id}가 상세 조회하는가?
4. PUT /api/v1/admin/scope-profiles/{id}가 수정하는가?
5. DELETE /api/v1/admin/scope-profiles/{id}가 삭제하는가?
6. 모든 Scope Profile API에서 admin 권한을 검증하는가?
7. 다른 조직의 Scope Profile 접근을 차단하는가?
8. API Key 생성 시 scope_profile_id를 올바르게 바인딩하는가?
9. Kill Switch API (POST/DELETE)가 정상 동작하는가?
10. Kill Switch 활성화 후 에이전트 요청이 차단되는가?
11. Kill Switch 해제 후 에이전트 요청이 다시 가능한가?
12. 캐시 TTL이 1분 이하인가?
13. 감시 로그에 kill switch 이벤트가 기록되는가?
14. 모든 단위/통합 테스트가 통과하는가 (`pytest -v`)?
15. mypy 타입 검사를 통과하는가?

## 7. 작업 지침

### 지침 7-1. Admin 권한 검증

모든 Scope Profile 및 Kill Switch API는 admin 역할 필수이다. require_admin() 헬퍼를 사용하여 일관되게 검증한다.

```python
require_admin(access_context)
```

### 지침 7-2. 조직 경계 검증

모든 리소스 조회/수정 시 organization_id를 검증하여 다른 조직의 리소스 접근을 차단한다.

```python
if scope_profile.organization_id != access_context.organization_id:
    raise ForbiddenError("Cannot access resource from different organization")
```

### 지침 7-3. Kill Switch 캐시 활용

캐시 TTL 1분 이하로 설정하여 kill switch 변경이 빠르게 반영되도록 한다. enable/disable 직후 캐시를 무효화한다.

```python
kill_switch_cache.invalidate(agent_id)
```

### 지침 7-4. API Key + Scope Profile 바인딩

에이전트용 API Key 발급 시 scope_profile_id를 필수로 바인딩한다. 사용자용 API Key는 scope_profile_id = NULL로 유지한다.

```python
# 에이전트
raw_key, api_key = await api_key_service.create_api_key_for_agent(
    organization_id=org_id,
    created_by_user_id=admin_id,
    scope_profile_id=scope_profile_id,  # 필수
)

# 사용자 (legacy)
raw_key, api_key = await api_key_service.create_api_key_for_user(...)
```

### 지침 7-5. FilterExpression 검증 (Pydantic)

FilterExpression을 생성 요청에서 받을 때, Pydantic validator를 통해 기본 구조를 검증한다. 심화 검증은 Task 4-5의 FilterParser에서 수행된다.

```python
@validator("acl_filter")
def validate_acl_filter(cls, v):
    if not isinstance(v, dict):
        raise ValueError("acl_filter must be a dictionary")
    if "and" not in v and "or" not in v:
        raise ValueError("acl_filter must contain 'and' or 'or'")
    return v
```

### 지침 7-6. 감시 로그 기록

Kill Switch 활성화/해제 시 감시 로그를 기록하여 "누가 언제 어떤 에이전트를 비활성화했는지" 추적 가능하게 한다.

```python
await audit_service.log_event(
    entity_type="agent",
    entity_id=agent_id,
    action="kill_switch_enable",
    access_context=access_context,
    changes={"reason": request.reason},
)
```

### 지침 7-7. 페이지네이션 및 에러 처리

목록 조회 엔드포인트는 페이지네이션을 지원한다. page, page_size 쿼리 파라미터로 제어 가능하다.

```python
@router.get("/scope-profiles")
async def list_scope_profiles(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    ...
)
```

### 지침 7-8. FastAPI 의존성 설계

FastAPI의 Depends()를 활용하여 인증, 권한 검증, DB 세션 주입을 일관되게 처리한다.

```python
async def create_scope_profile(
    request: ScopeProfileCreateRequest,
    access_context: AccessContext = Depends(get_access_context),
    db: AsyncSession = Depends(get_db),
):
    ...
```

### 지침 7-9. Soft Delete vs Physical Delete 정책

Scope Profile 삭제는 물리 삭제를 기본으로 한다 (감시 기록 외). 필요 시 soft delete로 변경 가능하다.

### 지침 7-10. 에러 응답 일관성

모든 API 에러 응답은 ErrorResponse 스키마를 따른다.

```json
{
  "error": "forbidden",
  "message": "Cannot access resource from different organization",
  "details": {...}
}
```
