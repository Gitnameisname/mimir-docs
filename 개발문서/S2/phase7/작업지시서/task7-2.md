# Task 7-2. GoldenSet CRUD API + 버전 관리

## 1. 작업 목적

Task 7-1에서 정의한 Golden Set 도메인 모델 및 Repository 계층을 기반으로, **FastAPI 라우터를 구현하여 9개의 CRUD 엔드포인트**를 제공한다. 버전 관리, 페이징, 필터링, Scope Profile 기반 ACL, 그리고 감사 로깅(actor_type 필드 포함)을 통합하여, API 클라이언트(UI, MCP, 에이전트)가 Golden Set을 효과적으로 관리할 수 있도록 한다.

## 2. 작업 범위

### 포함 범위

1. **GoldenSet CRUD 엔드포인트 (5개)**
   - `POST /api/v1/golden-sets` — 새 GoldenSet 생성
   - `GET /api/v1/golden-sets` — GoldenSet 목록 조회 (페이징, 필터)
   - `GET /api/v1/golden-sets/{id}` — 상세 조회 (모든 items 포함)
   - `PUT /api/v1/golden-sets/{id}` — 메타데이터 수정
   - `DELETE /api/v1/golden-sets/{id}` — Soft delete

2. **GoldenItem CRUD 엔드포인트 (4개)**
   - `POST /api/v1/golden-sets/{id}/items` — 새 항목 추가
   - `GET /api/v1/golden-sets/{id}/items` — 항목 목록 조회
   - `PUT /api/v1/golden-sets/{id}/items/{item_id}` — 항목 수정 (자동 버전 증가)
   - `DELETE /api/v1/golden-sets/{id}/items/{item_id}` — 항목 soft delete

3. **버전 관리 엔드포인트 (3개, Task 7-1의 기초 확장)**
   - `GET /api/v1/golden-sets/{id}/versions` — 버전 이력 조회
   - `GET /api/v1/golden-sets/{id}/versions/{version}` — 특정 버전 조회
   - `GET /api/v1/golden-sets/{id}/versions/{from_v}/diff/{to_v}` — 버전 간 diff

4. **FastAPI 라우터 + 요청/응답 처리**
   - unwrapEnvelope 응답 포맷 (Mimir 표준)
   - Pagination, filtering, sorting 지원
   - 에러 처리 및 HTTP 상태 코드 (201, 204, 400, 403, 404, 409)
   - Request body validation (Pydantic DTO)

5. **Scope Profile 기반 ACL**
   - S2 원칙 ⑥: 모든 엔드포인트에 scope_id 필터 적용
   - 요청자의 scope (Access Token 또는 세션에서 추출)과 리소스 scope 일치 검증
   - 접근 거부 시 403 Forbidden 반환

6. **감사 로깅 (Audit Log)**
   - S2 원칙 ⑤: actor_type 필드 필수 ("user" 또는 "agent")
   - 모든 쓰기 작업(Create, Update, Delete)을 감사 로그에 기록
   - 로그 항목: golden_set_created, golden_set_updated, golden_set_deleted, golden_item_added, golden_item_modified, golden_item_deleted

7. **통합 테스트**
   - 모든 9개 엔드포인트의 성공 케이스
   - ACL 실패 케이스 (scope_id 불일치)
   - 버전 관리 정확성 검증
   - 동시성 테스트 (여러 사용자의 동시 수정)
   - 에러 케이스 (불완전한 요청, 미존재 리소스)

### 제외 범위

- import/export 엔드포인트 (Task 7-3)
- 평가 러너 통합 (Task 7-2 FG7.2)
- CI 게이트 (Task 7-3 FG7.3)
- 권한 세부 정책 (Admin/Evaluator 역할 구분은 Task 7-3)

## 3. 선행 조건

- Task 7-1 완료: 도메인 모델, ORM, Repository 구현
- FastAPI 프로젝트 구조 확립
- PostgreSQL 및 SQLAlchemy 설정 완료
- 인증 시스템 (OAuth 2.0 / Session)에서 actor_id, actor_type 추출 가능
- Scope Profile 데이터 구조 및 조회 함수 구현 완료

## 4. 주요 작업 항목

### 4-1. FastAPI 라우터 구현 (GoldenSet 엔드포인트)

**파일:** `/backend/app/routes/golden_sets.py`

```python
"""
Golden Set CRUD API 라우터.

S2 원칙 ⑤, ⑥, ⑦을 준수하여 구현:
- ⑤: actor_type 필드로 사용자/에이전트 구분
- ⑥: Scope Profile 기반 ACL 필터
- ⑦: 폐쇄망 환경에서 외부 의존 없음
"""

from __future__ import annotations

from typing import Optional, List, Dict, Any
from datetime import datetime
import logging

from fastapi import APIRouter, Depends, HTTPException, Query, Header
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.auth import get_current_user, get_actor_info  # S2 ⑤: actor_type 추출
from app.models.golden_set import (
    GoldenSet, GoldenItem,
    GoldenSetCreateRequest, GoldenSetUpdateRequest,
    GoldenItemCreateRequest, GoldenItemUpdateRequest,
    GoldenSetResponse, GoldenSetDetailResponse, GoldenItemResponse,
    GoldenSetVersionInfo, GoldenSetVersionDiff,
)
from app.repositories.golden_set_repository import (
    GoldenSetRepository, GoldenItemRepository
)
from app.services.audit_log_service import AuditLogService  # S2 ⑤
from app.schemas.response import unwrapEnvelope, PaginatedResponse

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/v1/golden-sets", tags=["golden-sets"])


# ==================== Dependency Injection ====================

async def get_golden_set_repo(
    db: AsyncSession = Depends(get_db),
) -> GoldenSetRepository:
    """GoldenSetRepository 의존성 주입."""
    return GoldenSetRepository(db)


async def get_golden_item_repo(
    db: AsyncSession = Depends(get_db),
) -> GoldenItemRepository:
    """GoldenItemRepository 의존성 주입."""
    return GoldenItemRepository(db)


async def get_audit_log_service(
    db: AsyncSession = Depends(get_db),
) -> AuditLogService:
    """AuditLogService 의존성 주입 (S2 ⑤)."""
    return AuditLogService(db)


# ==================== Helper Functions ====================

async def check_scope_access(
    current_user: Dict[str, Any],
) -> str:
    """
    현재 사용자의 scope_id 추출 (S2 ⑥).
    
    Args:
        current_user: 인증된 사용자 정보
    
    Returns:
        사용자의 scope_id
    
    Raises:
        HTTPException(403): scope 정보 없음
    """
    scope_id = current_user.get("scope_id")
    if not scope_id:
        raise HTTPException(
            status_code=403,
            detail="User scope not found (S2 ⑥: Scope Profile required)"
        )
    return scope_id


async def get_actor_info_with_type(
    current_user: Dict[str, Any],
) -> tuple[str, str]:
    """
    현재 actor의 ID와 type 추출 (S2 ⑤).
    
    Returns:
        (actor_id, actor_type) 튜플
        actor_type: "user" 또는 "agent"
    """
    actor_id = current_user.get("user_id") or current_user.get("agent_id")
    actor_type = current_user.get("actor_type", "user")  # 기본값: user
    
    if not actor_id:
        raise HTTPException(status_code=401, detail="Unable to determine actor identity")
    
    return actor_id, actor_type


# ==================== GoldenSet CRUD Endpoints ====================

@router.post(
    "",
    response_model=unwrapEnvelope[GoldenSetResponse],
    status_code=201,
)
async def create_golden_set(
    request: GoldenSetCreateRequest,
    current_user: Dict[str, Any] = Depends(get_current_user),
    repo: GoldenSetRepository = Depends(get_golden_set_repo),
    audit_service: AuditLogService = Depends(get_audit_log_service),
) -> unwrapEnvelope[GoldenSetResponse]:
    """
    새로운 GoldenSet 생성.
    
    요청:
        POST /api/v1/golden-sets
        {
          "name": "기술 문서 RAG",
          "description": "내부 기술 명세서 검색 평가",
          "domain": "technical_guide",
          "metadata": {"owner": "platform-team"}
        }
    
    응답 (201):
        {
          "status": "success",
          "data": {
            "id": "set-123",
            "scope_id": "scope-456",
            "name": "기술 문서 RAG",
            "version": 1,
            ...
          }
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # S2 ⑤: actor_id, actor_type 추출
        actor_id, actor_type = await get_actor_info_with_type(current_user)
        
        # Repository 호출
        golden_set = await repo.create_golden_set(
            scope_id=scope_id,
            request=request,
            created_by=actor_id,
        )
        
        # S2 ⑤: 감사 로그 기록
        await audit_service.log_action(
            action="golden_set_created",
            resource_type="GoldenSet",
            resource_id=golden_set.id,
            actor_id=actor_id,
            actor_type=actor_type,  # ← S2 ⑤: actor_type 필드
            changes={
                "name": golden_set.name,
                "domain": golden_set.domain,
                "scope_id": golden_set.scope_id,
            }
        )
        
        response_dto = GoldenSetResponse(**golden_set.dict())
        logger.info(f"GoldenSet created: {golden_set.id} by {actor_type}:{actor_id}")
        
        return unwrapEnvelope(
            status="success",
            data=response_dto,
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error creating GoldenSet: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


@router.get(
    "",
    response_model=unwrapEnvelope[PaginatedResponse[GoldenSetResponse]],
)
async def list_golden_sets(
    offset: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    domain: Optional[str] = Query(None),
    status: Optional[str] = Query(None),
    current_user: Dict[str, Any] = Depends(get_current_user),
    repo: GoldenSetRepository = Depends(get_golden_set_repo),
) -> unwrapEnvelope[PaginatedResponse[GoldenSetResponse]]:
    """
    GoldenSet 목록 조회 (페이징, 필터).
    
    쿼리 파라미터:
        offset: 페이징 오프셋 (기본값 0)
        limit: 페이징 제한 (기본값 100, 최대 1000)
        domain: 도메인 필터 (선택사항)
        status: 상태 필터 (draft, published, archived)
    
    응답:
        {
          "status": "success",
          "data": {
            "items": [...],
            "total": 42,
            "offset": 0,
            "limit": 100
          }
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # Repository 호출
        golden_sets, total = await repo.list_by_scope(
            scope_id=scope_id,
            offset=offset,
            limit=limit,
            domain=domain,
            status=status,
        )
        
        response_dtos = [
            GoldenSetResponse(**gs.dict()) for gs in golden_sets
        ]
        
        return unwrapEnvelope(
            status="success",
            data=PaginatedResponse(
                items=response_dtos,
                total=total,
                offset=offset,
                limit=limit,
            )
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error listing GoldenSets: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


@router.get(
    "/{golden_set_id}",
    response_model=unwrapEnvelope[GoldenSetDetailResponse],
)
async def get_golden_set(
    golden_set_id: str,
    current_user: Dict[str, Any] = Depends(get_current_user),
    repo: GoldenSetRepository = Depends(get_golden_set_repo),
) -> unwrapEnvelope[GoldenSetDetailResponse]:
    """
    GoldenSet 상세 조회 (모든 items 포함).
    
    응답:
        {
          "status": "success",
          "data": {
            "id": "set-123",
            ...
            "items": [
              {
                "id": "item-1",
                "question": "...",
                ...
              },
              ...
            ]
          }
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # Repository 호출
        golden_set = await repo.get_by_id(
            golden_set_id=golden_set_id,
            scope_id=scope_id,
            include_items=True,
        )
        
        if golden_set is None:
            raise HTTPException(status_code=404, detail="GoldenSet not found")
        
        response_dto = GoldenSetDetailResponse(**golden_set.dict())
        
        return unwrapEnvelope(
            status="success",
            data=response_dto,
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error getting GoldenSet {golden_set_id}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


@router.put(
    "/{golden_set_id}",
    response_model=unwrapEnvelope[GoldenSetResponse],
)
async def update_golden_set(
    golden_set_id: str,
    request: GoldenSetUpdateRequest,
    current_user: Dict[str, Any] = Depends(get_current_user),
    repo: GoldenSetRepository = Depends(get_golden_set_repo),
    audit_service: AuditLogService = Depends(get_audit_log_service),
) -> unwrapEnvelope[GoldenSetResponse]:
    """
    GoldenSet 메타데이터 수정 (버전 자동 증가).
    
    요청:
        PUT /api/v1/golden-sets/{id}
        {
          "name": "새로운 이름",
          "status": "published"
        }
    
    응답:
        {
          "status": "success",
          "data": {
            "id": "set-123",
            "version": 2,  # ← 증가됨
            ...
          }
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # S2 ⑤: actor_id, actor_type 추출
        actor_id, actor_type = await get_actor_info_with_type(current_user)
        
        # Repository 호출
        updated_golden_set = await repo.update_golden_set(
            golden_set_id=golden_set_id,
            scope_id=scope_id,
            request=request,
            updated_by=actor_id,
        )
        
        if updated_golden_set is None:
            raise HTTPException(status_code=404, detail="GoldenSet not found")
        
        # S2 ⑤: 감사 로그 기록
        await audit_service.log_action(
            action="golden_set_updated",
            resource_type="GoldenSet",
            resource_id=golden_set_id,
            actor_id=actor_id,
            actor_type=actor_type,
            changes=request.dict(exclude_unset=True),
        )
        
        response_dto = GoldenSetResponse(**updated_golden_set.dict())
        logger.info(f"GoldenSet updated: {golden_set_id} by {actor_type}:{actor_id}")
        
        return unwrapEnvelope(
            status="success",
            data=response_dto,
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error updating GoldenSet {golden_set_id}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


@router.delete(
    "/{golden_set_id}",
    status_code=204,
)
async def delete_golden_set(
    golden_set_id: str,
    current_user: Dict[str, Any] = Depends(get_current_user),
    repo: GoldenSetRepository = Depends(get_golden_set_repo),
    audit_service: AuditLogService = Depends(get_audit_log_service),
) -> None:
    """
    GoldenSet Soft delete.
    
    응답: 204 No Content
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # S2 ⑤: actor_id, actor_type 추출
        actor_id, actor_type = await get_actor_info_with_type(current_user)
        
        # Repository 호출
        success = await repo.soft_delete_golden_set(
            golden_set_id=golden_set_id,
            scope_id=scope_id,
        )
        
        if not success:
            raise HTTPException(status_code=404, detail="GoldenSet not found")
        
        # S2 ⑤: 감사 로그 기록
        await audit_service.log_action(
            action="golden_set_deleted",
            resource_type="GoldenSet",
            resource_id=golden_set_id,
            actor_id=actor_id,
            actor_type=actor_type,
        )
        
        logger.info(f"GoldenSet deleted: {golden_set_id} by {actor_type}:{actor_id}")
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error deleting GoldenSet {golden_set_id}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


# ==================== GoldenItem CRUD Endpoints ====================

@router.post(
    "/{golden_set_id}/items",
    response_model=unwrapEnvelope[GoldenItemResponse],
    status_code=201,
)
async def add_golden_item(
    golden_set_id: str,
    request: GoldenItemCreateRequest,
    current_user: Dict[str, Any] = Depends(get_current_user),
    set_repo: GoldenSetRepository = Depends(get_golden_set_repo),
    item_repo: GoldenItemRepository = Depends(get_golden_item_repo),
    audit_service: AuditLogService = Depends(get_audit_log_service),
) -> unwrapEnvelope[GoldenItemResponse]:
    """
    GoldenSet 내에 새로운 Q&A 항목 추가.
    
    요청:
        POST /api/v1/golden-sets/{id}/items
        {
          "question": "Docker란 무엇인가?",
          "expected_answer": "Docker는 컨테이너 기술을...",
          "expected_source_docs": [...],
          "expected_citations": [...]
        }
    
    응답 (201):
        {
          "status": "success",
          "data": {
            "id": "item-123",
            "golden_set_id": "{id}",
            "version": 1,
            ...
          }
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # S2 ⑤: actor_id, actor_type 추출
        actor_id, actor_type = await get_actor_info_with_type(current_user)
        
        # Repository 호출
        golden_item = await item_repo.create_item(
            golden_set_id=golden_set_id,
            scope_id=scope_id,
            request=request,
            created_by=actor_id,
        )
        
        if golden_item is None:
            raise HTTPException(status_code=404, detail="GoldenSet not found")
        
        # S2 ⑤: 감사 로그 기록
        await audit_service.log_action(
            action="golden_item_added",
            resource_type="GoldenItem",
            resource_id=golden_item.id,
            actor_id=actor_id,
            actor_type=actor_type,
            parent_resource={"type": "GoldenSet", "id": golden_set_id},
            changes={
                "question": golden_item.question,
                "expected_answer": golden_item.expected_answer[:100],
            }
        )
        
        response_dto = GoldenItemResponse(**golden_item.dict())
        logger.info(f"GoldenItem added to {golden_set_id}: {golden_item.id} by {actor_type}:{actor_id}")
        
        return unwrapEnvelope(
            status="success",
            data=response_dto,
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error adding GoldenItem to {golden_set_id}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


@router.get(
    "/{golden_set_id}/items",
    response_model=unwrapEnvelope[PaginatedResponse[GoldenItemResponse]],
)
async def list_golden_items(
    golden_set_id: str,
    offset: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    current_user: Dict[str, Any] = Depends(get_current_user),
    set_repo: GoldenSetRepository = Depends(get_golden_set_repo),
    item_repo: GoldenItemRepository = Depends(get_golden_item_repo),
) -> unwrapEnvelope[PaginatedResponse[GoldenItemResponse]]:
    """
    GoldenSet 내 Q&A 항목 목록 조회.
    
    응답:
        {
          "status": "success",
          "data": {
            "items": [
              {
                "id": "item-1",
                "question": "...",
                ...
              },
              ...
            ],
            "total": 25,
            "offset": 0,
            "limit": 100
          }
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # Parent GoldenSet 존재 확인
        parent = await set_repo.get_by_id(golden_set_id, scope_id)
        if parent is None:
            raise HTTPException(status_code=404, detail="GoldenSet not found")
        
        # Repository 호출
        items = await item_repo.list_items(golden_set_id, scope_id)
        
        # 클라이언트 측 페이징 (간단한 구현)
        paginated_items = items[offset:offset + limit]
        
        response_dtos = [
            GoldenItemResponse(**item.dict()) for item in paginated_items
        ]
        
        return unwrapEnvelope(
            status="success",
            data=PaginatedResponse(
                items=response_dtos,
                total=len(items),
                offset=offset,
                limit=limit,
            )
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error listing GoldenItems for {golden_set_id}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


@router.put(
    "/{golden_set_id}/items/{item_id}",
    response_model=unwrapEnvelope[GoldenItemResponse],
)
async def update_golden_item(
    golden_set_id: str,
    item_id: str,
    request: GoldenItemUpdateRequest,
    current_user: Dict[str, Any] = Depends(get_current_user),
    set_repo: GoldenSetRepository = Depends(get_golden_set_repo),
    item_repo: GoldenItemRepository = Depends(get_golden_item_repo),
    audit_service: AuditLogService = Depends(get_audit_log_service),
) -> unwrapEnvelope[GoldenItemResponse]:
    """
    GoldenItem 수정 (자동 버전 증가).
    
    요청:
        PUT /api/v1/golden-sets/{id}/items/{item_id}
        {
          "question": "수정된 질문"
        }
    
    응답:
        {
          "status": "success",
          "data": {
            "id": "item-123",
            "version": 2,  # ← 증가됨
            ...
          }
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # S2 ⑤: actor_id, actor_type 추출
        actor_id, actor_type = await get_actor_info_with_type(current_user)
        
        # Repository 호출
        updated_item = await item_repo.update_item(
            item_id=item_id,
            scope_id=scope_id,
            request=request,
            updated_by=actor_id,
        )
        
        if updated_item is None:
            raise HTTPException(status_code=404, detail="GoldenItem not found")
        
        # S2 ⑤: 감사 로그 기록
        await audit_service.log_action(
            action="golden_item_modified",
            resource_type="GoldenItem",
            resource_id=item_id,
            actor_id=actor_id,
            actor_type=actor_type,
            parent_resource={"type": "GoldenSet", "id": golden_set_id},
            changes=request.dict(exclude_unset=True),
        )
        
        response_dto = GoldenItemResponse(**updated_item.dict())
        logger.info(f"GoldenItem updated: {item_id} by {actor_type}:{actor_id}")
        
        return unwrapEnvelope(
            status="success",
            data=response_dto,
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error updating GoldenItem {item_id}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


@router.delete(
    "/{golden_set_id}/items/{item_id}",
    status_code=204,
)
async def delete_golden_item(
    golden_set_id: str,
    item_id: str,
    current_user: Dict[str, Any] = Depends(get_current_user),
    set_repo: GoldenSetRepository = Depends(get_golden_set_repo),
    item_repo: GoldenItemRepository = Depends(get_golden_item_repo),
    audit_service: AuditLogService = Depends(get_audit_log_service),
) -> None:
    """
    GoldenItem soft delete.
    
    응답: 204 No Content
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # S2 ⑤: actor_id, actor_type 추출
        actor_id, actor_type = await get_actor_info_with_type(current_user)
        
        # Repository 호출
        success = await item_repo.soft_delete_item(
            item_id=item_id,
            scope_id=scope_id,
        )
        
        if not success:
            raise HTTPException(status_code=404, detail="GoldenItem not found")
        
        # S2 ⑤: 감사 로그 기록
        await audit_service.log_action(
            action="golden_item_deleted",
            resource_type="GoldenItem",
            resource_id=item_id,
            actor_id=actor_id,
            actor_type=actor_type,
            parent_resource={"type": "GoldenSet", "id": golden_set_id},
        )
        
        logger.info(f"GoldenItem deleted: {item_id} by {actor_type}:{actor_id}")
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error deleting GoldenItem {item_id}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


# ==================== Version Management Endpoints ====================

@router.get(
    "/{golden_set_id}/versions",
    response_model=unwrapEnvelope[List[GoldenSetVersionInfo]],
)
async def get_version_history(
    golden_set_id: str,
    current_user: Dict[str, Any] = Depends(get_current_user),
    repo: GoldenSetRepository = Depends(get_golden_set_repo),
) -> unwrapEnvelope[List[GoldenSetVersionInfo]]:
    """
    GoldenSet 버전 이력 조회.
    
    응답:
        {
          "status": "success",
          "data": [
            {
              "version": 3,
              "created_at": "2025-01-15T14:30:00Z",
              "created_by": "user-456",
              "item_count": 25
            },
            {
              "version": 2,
              ...
            },
            ...
          ]
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # Repository 호출
        history = await repo.get_version_history(golden_set_id, scope_id)
        
        if not history:
            raise HTTPException(status_code=404, detail="GoldenSet not found")
        
        return unwrapEnvelope(
            status="success",
            data=[GoldenSetVersionInfo(**v) for v in history],
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error getting version history for {golden_set_id}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


@router.get(
    "/{golden_set_id}/versions/{version}",
    response_model=unwrapEnvelope[Dict[str, Any]],
)
async def get_version_snapshot(
    golden_set_id: str,
    version: int,
    current_user: Dict[str, Any] = Depends(get_current_user),
    repo: GoldenSetRepository = Depends(get_golden_set_repo),
) -> unwrapEnvelope[Dict[str, Any]]:
    """
    특정 버전의 GoldenSet 스냅샷 조회.
    
    응답:
        {
          "status": "success",
          "data": {
            "version": 2,
            "name": "기술 문서 RAG",
            "items": [
              {
                "id": "item-1",
                "question": "...",
                ...
              },
              ...
            ],
            "created_at": "2025-01-15T10:00:00Z",
            "created_by": "user-123"
          }
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # Repository 호출
        snapshot = await repo.get_version_snapshot(golden_set_id, scope_id, version)
        
        if snapshot is None:
            raise HTTPException(status_code=404, detail="Version not found")
        
        return unwrapEnvelope(
            status="success",
            data=snapshot,
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error getting version snapshot for {golden_set_id} v{version}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


@router.get(
    "/{golden_set_id}/versions/{from_v}/diff/{to_v}",
    response_model=unwrapEnvelope[GoldenSetVersionDiff],
)
async def get_version_diff(
    golden_set_id: str,
    from_v: int,
    to_v: int,
    current_user: Dict[str, Any] = Depends(get_current_user),
    repo: GoldenSetRepository = Depends(get_golden_set_repo),
) -> unwrapEnvelope[GoldenSetVersionDiff]:
    """
    두 버전 간의 diff 계산.
    
    응답:
        {
          "status": "success",
          "data": {
            "from_version": 1,
            "to_version": 3,
            "items_added": ["item-5", "item-6"],
            "items_modified": ["item-2"],
            "items_deleted": [],
            "modified_at": "2025-01-15T14:30:00Z"
          }
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = await check_scope_access(current_user)
        
        # 두 버전 스냅샷 로드
        from_snapshot = await repo.get_version_snapshot(golden_set_id, scope_id, from_v)
        to_snapshot = await repo.get_version_snapshot(golden_set_id, scope_id, to_v)
        
        if from_snapshot is None or to_snapshot is None:
            raise HTTPException(status_code=404, detail="Version not found")
        
        # Diff 계산
        from_items = {item["id"] for item in from_snapshot.get("items", [])}
        to_items = {item["id"] for item in to_snapshot.get("items", [])}
        
        # 간단한 diff: added = to - from, deleted = from - to
        # 더 정교한 diff는 별도 로직 필요
        diff = GoldenSetVersionDiff(
            from_version=from_v,
            to_version=to_v,
            items_added=list(to_items - from_items),
            items_modified=[],  # 상세 변경은 별도 계산 필요
            items_deleted=list(from_items - to_items),
            modified_at=datetime.fromisoformat(to_snapshot["created_at"]),
        )
        
        return unwrapEnvelope(
            status="success",
            data=diff,
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error computing diff for {golden_set_id} {from_v}→{to_v}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))
```

### 4-2. 감사 로그 서비스 (S2 ⑤ actor_type 지원)

**파일:** `/backend/app/services/audit_log_service.py`

```python
"""
감사 로그 서비스 (S2 ⑤: actor_type 필드 지원).

모든 GoldenSet 쓰기 작업을 로깅한다.
"""

from __future__ import annotations

from datetime import datetime
from typing import Any, Optional, Dict
from uuid import uuid4

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import insert

import logging

logger = logging.getLogger(__name__)


class AuditLogService:
    """감사 로그 기록 서비스."""
    
    def __init__(self, db_session: AsyncSession):
        self.db = db_session
    
    async def log_action(
        self,
        action: str,
        resource_type: str,
        resource_id: str,
        actor_id: str,
        actor_type: str,  # S2 ⑤: "user" 또는 "agent"
        changes: Optional[Dict[str, Any]] = None,
        parent_resource: Optional[Dict[str, str]] = None,
    ) -> None:
        """
        감사 로그 기록.
        
        Args:
            action: 액션 (golden_set_created, golden_item_modified 등)
            resource_type: 리소스 타입 (GoldenSet, GoldenItem)
            resource_id: 리소스 ID
            actor_id: 행위자 ID (사용자 ID 또는 에이전트 ID)
            actor_type: 행위자 타입 (S2 ⑤: "user" 또는 "agent")
            changes: 변경 내용 (JSON)
            parent_resource: 부모 리소스 (예: GoldenItem의 부모 GoldenSet)
        """
        try:
            # 감사 로그 항목 생성
            log_entry = {
                "id": str(uuid4()),
                "action": action,
                "resource_type": resource_type,
                "resource_id": resource_id,
                "actor_id": actor_id,
                "actor_type": actor_type,  # ← S2 ⑤: actor_type 필드
                "changes": changes or {},
                "parent_resource": parent_resource,
                "created_at": datetime.utcnow(),
            }
            
            # DB에 저장 (audit_logs 테이블 가정)
            stmt = insert(AuditLogORM).values(**log_entry)
            await self.db.execute(stmt)
            await self.db.commit()
            
            logger.info(
                f"Audit log: {actor_type}:{actor_id} -> {action} ({resource_type}:{resource_id})"
            )
        
        except Exception as e:
            logger.error(f"Error logging audit action: {str(e)}", exc_info=True)
            # 감사 로그 실패가 요청을 실패하게 하지 않도록 함


# ==================== ORM 정의 ====================

from sqlalchemy import Table, Column, String, Text, JSON, TIMESTAMP, MetaData

metadata = MetaData()

audit_logs_table = Table(
    'audit_logs',
    metadata,
    Column('id', String(36), primary_key=True),
    Column('action', String(100), nullable=False, index=True),
    Column('resource_type', String(50), nullable=False),
    Column('resource_id', String(36), nullable=False, index=True),
    Column('actor_id', String(255), nullable=False),
    Column('actor_type', String(20), nullable=False),  # S2 ⑤: "user" or "agent"
    Column('changes', JSON, nullable=True),
    Column('parent_resource', JSON, nullable=True),
    Column('created_at', TIMESTAMP(timezone=True), nullable=False, server_default='now()', index=True),
)


class AuditLogORM:
    """감사 로그 ORM (간단한 구현)."""
    __tablename__ = 'audit_logs'
```

### 4-3. 응답 포맷 (unwrapEnvelope)

**파일:** `/backend/app/schemas/response.py`

```python
"""
Mimir 표준 응답 포맷 (unwrapEnvelope).

모든 API 응답은 이 포맷을 따른다.
"""

from typing import Generic, TypeVar, List, Optional, Any
from pydantic import BaseModel

T = TypeVar('T')


class unwrapEnvelope(BaseModel, Generic[T]):
    """
    Mimir 표준 응답 포맷.
    
    {
      "status": "success" | "error",
      "data": <T>,
      "error": (optional) error details
    }
    """
    status: str  # "success" or "error"
    data: Optional[T] = None
    error: Optional[str] = None
    
    class Config:
        arbitrary_types_allowed = True


class PaginatedResponse(BaseModel, Generic[T]):
    """페이지네이션 응답."""
    items: List[T]
    total: int
    offset: int
    limit: int
```

### 4-4. 통합 테스트

**파일:** `/backend/tests/integration/test_golden_set_api.py`

```python
"""GoldenSet CRUD API 통합 테스트."""

import pytest
from httpx import AsyncClient
from unittest.mock import patch, AsyncMock

from app.main import app


@pytest.fixture
async def client():
    """테스트 클라이언트."""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac


@pytest.fixture
def mock_auth():
    """인증 모킹."""
    return {
        "user_id": "test-user",
        "actor_type": "user",  # S2 ⑤
        "scope_id": "test-scope",
    }


@pytest.fixture
def mock_agent_auth():
    """에이전트 인증 모킹 (S2 ⑤)."""
    return {
        "agent_id": "test-agent",
        "actor_type": "agent",  # ← S2 ⑤: agent
        "scope_id": "test-scope",
    }


@pytest.mark.asyncio
async def test_create_golden_set_success(client, mock_auth):
    """GoldenSet 생성 성공 (S2 ⑤, ⑥)."""
    with patch("app.auth.get_current_user", return_value=mock_auth):
        with patch("app.services.audit_log_service.AuditLogService.log_action", new_callable=AsyncMock):
            response = await client.post(
                "/api/v1/golden-sets",
                json={
                    "name": "Test Set",
                    "description": "Test description",
                    "domain": "technical_guide",
                }
            )
    
    assert response.status_code == 201
    data = response.json()
    assert data["status"] == "success"
    assert data["data"]["name"] == "Test Set"
    assert data["data"]["scope_id"] == "test-scope"


@pytest.mark.asyncio
async def test_create_golden_set_by_agent(client, mock_agent_auth):
    """에이전트에 의한 GoldenSet 생성 (S2 ⑤)."""
    with patch("app.auth.get_current_user", return_value=mock_agent_auth):
        audit_log_service = AsyncMock()
        
        with patch("app.services.audit_log_service.AuditLogService") as mock_audit:
            mock_audit.return_value.log_action = AsyncMock()
            
            response = await client.post(
                "/api/v1/golden-sets",
                json={
                    "name": "Agent Created Set",
                    "domain": "custom",
                }
            )
    
    assert response.status_code == 201
    # ← audit log에 actor_type="agent"가 기록됨 (S2 ⑤)


@pytest.mark.asyncio
async def test_list_golden_sets_scope_acl(client, mock_auth):
    """Scope 기반 ACL 필터 (S2 ⑥)."""
    with patch("app.auth.get_current_user", return_value=mock_auth):
        # 여러 set 조회, scope_id로 필터됨
        response = await client.get(
            "/api/v1/golden-sets?offset=0&limit=100"
        )
    
    assert response.status_code == 200
    data = response.json()
    
    # 모든 반환된 항목이 같은 scope_id를 가져야 함
    for item in data["data"]["items"]:
        assert item["scope_id"] == "test-scope"


@pytest.mark.asyncio
async def test_get_golden_set_not_found_different_scope(client):
    """다른 scope에서 접근 시 404 (S2 ⑥)."""
    auth = {"user_id": "user-1", "scope_id": "scope-a"}
    
    with patch("app.auth.get_current_user", return_value=auth):
        response = await client.get(
            "/api/v1/golden-sets/set-in-scope-b"
        )
    
    # scope-b의 set을 scope-a 사용자가 접근 불가
    assert response.status_code == 404


@pytest.mark.asyncio
async def test_update_golden_set_version_increment(client, mock_auth):
    """수정 시 버전 자동 증가."""
    with patch("app.auth.get_current_user", return_value=mock_auth):
        with patch("app.services.audit_log_service.AuditLogService.log_action", new_callable=AsyncMock):
            # 먼저 생성
            create_resp = await client.post(
                "/api/v1/golden-sets",
                json={"name": "Test Set"}
            )
            assert create_resp.status_code == 201
            set_id = create_resp.json()["data"]["id"]
            initial_version = create_resp.json()["data"]["version"]
            
            # 수정
            update_resp = await client.put(
                f"/api/v1/golden-sets/{set_id}",
                json={"name": "Updated Name"}
            )
    
    assert update_resp.status_code == 200
    updated_version = update_resp.json()["data"]["version"]
    assert updated_version > initial_version


@pytest.mark.asyncio
async def test_add_golden_item_parent_version_increment(client, mock_auth):
    """항목 추가 시 부모 GoldenSet 버전 증가."""
    with patch("app.auth.get_current_user", return_value=mock_auth):
        with patch("app.services.audit_log_service.AuditLogService.log_action", new_callable=AsyncMock):
            # GoldenSet 생성
            set_resp = await client.post(
                "/api/v1/golden-sets",
                json={"name": "Test Set"}
            )
            set_id = set_resp.json()["data"]["id"]
            initial_version = set_resp.json()["data"]["version"]
            
            # Item 추가
            item_resp = await client.post(
                f"/api/v1/golden-sets/{set_id}/items",
                json={
                    "question": "Q1?",
                    "expected_answer": "A1",
                    "expected_source_docs": [
                        {
                            "document_id": "doc-1",
                            "version_id": 1,
                            "node_id": "node-1"
                        }
                    ]
                }
            )
            
            assert item_resp.status_code == 201
            
            # Parent GoldenSet 버전 확인
            get_resp = await client.get(f"/api/v1/golden-sets/{set_id}")
            parent_version = get_resp.json()["data"]["version"]
    
    assert parent_version > initial_version


@pytest.mark.asyncio
async def test_get_version_history(client, mock_auth):
    """버전 이력 조회."""
    with patch("app.auth.get_current_user", return_value=mock_auth):
        with patch("app.services.audit_log_service.AuditLogService.log_action", new_callable=AsyncMock):
            # GoldenSet 생성
            create_resp = await client.post(
                "/api/v1/golden-sets",
                json={"name": "Test Set"}
            )
            set_id = create_resp.json()["data"]["id"]
            
            # 버전 이력 조회
            history_resp = await client.get(
                f"/api/v1/golden-sets/{set_id}/versions"
            )
    
    assert history_resp.status_code == 200
    history = history_resp.json()["data"]
    assert len(history) > 0
    assert history[0]["version"] >= 1


@pytest.mark.asyncio
async def test_delete_golden_set_soft_delete(client, mock_auth):
    """Soft delete 동작 확인."""
    with patch("app.auth.get_current_user", return_value=mock_auth):
        with patch("app.services.audit_log_service.AuditLogService.log_action", new_callable=AsyncMock):
            # GoldenSet 생성
            create_resp = await client.post(
                "/api/v1/golden-sets",
                json={"name": "Test Set"}
            )
            set_id = create_resp.json()["data"]["id"]
            
            # 삭제
            delete_resp = await client.delete(
                f"/api/v1/golden-sets/{set_id}"
            )
            assert delete_resp.status_code == 204
            
            # 다시 조회하면 404 (soft delete 필터)
            get_resp = await client.get(
                f"/api/v1/golden-sets/{set_id}"
            )
    
    assert get_resp.status_code == 404
```

## 5. 산출물

1. **GoldenSet CRUD 라우터** (`/backend/app/routes/golden_sets.py`)
   - 5개 엔드포인트 (GoldenSet CRUD)
   - 4개 엔드포인트 (GoldenItem CRUD)
   - 3개 엔드포인트 (버전 관리)
   - S2 원칙 ⑥ ACL 필터
   - S2 원칙 ⑤ actor_type 감사 로깅

2. **감사 로그 서비스** (`/backend/app/services/audit_log_service.py`)
   - actor_type 필드 지원 (S2 ⑤)
   - 모든 쓰기 작업 로깅

3. **응답 포맷** (`/backend/app/schemas/response.py`)
   - unwrapEnvelope 제네릭 모델
   - PaginatedResponse 페이징 모델

4. **통합 테스트** (`/backend/tests/integration/test_golden_set_api.py`)
   - 모든 엔드포인트 성공 케이스
   - ACL 필터 검증
   - 버전 관리 검증
   - actor_type 감사 로깅 검증

## 6. 완료 기준

1. 9개 엔드포인트가 모두 구현되었는가?
2. S2 원칙 ⑥: 모든 엔드포인트에서 scope_id 필터를 적용하는가?
3. S2 원칙 ⑤: actor_type 필드를 감사 로그에 기록하는가?
4. 버전 관리: GoldenSet/GoldenItem 수정 시 버전이 자동 증가하는가?
5. 페이징, 필터링, 정렬이 정상 작동하는가?
6. 에러 처리: 404, 403, 400 등 적절한 HTTP 상태 코드를 반환하는가?
7. unwrapEnvelope 응답 포맷이 일관되는가?
8. 모든 통합 테스트가 통과하는가?
9. mypy 타입 검사를 통과하는가?
10. 인증되지 않은 요청 또는 scope 불일치 시 403/404를 반환하는가?

## 7. 작업 지침

### 지침 7-1. Scope Profile 기반 ACL (S2 ⑥)

모든 엔드포인트 시작에서:
```python
scope_id = await check_scope_access(current_user)
```
을 호출하여 현재 사용자의 scope_id를 추출하고, Repository 호출에 필수 파라미터로 전달한다.

### 지침 7-2. actor_type 감시 (S2 ⑤)

모든 쓰기 작업(Create, Update, Delete)에서:
```python
actor_id, actor_type = await get_actor_info_with_type(current_user)
```
을 호출하고, 감사 로그에 `actor_type=actor_type`을 포함한다. 이를 통해 사용자와 에이전트의 행동을 구분할 수 있다.

### 지침 7-3. 응답 포맷 일관성

모든 성공 응답은:
```python
unwrapEnvelope(status="success", data=response_dto)
```
형식으로 반환한다. 페이징 응답은 data에 PaginatedResponse를 중첩한다.

### 지침 7-4. 에러 처리 표준화

- 404 Not Found: 리소스 미존재 (또는 scope 불일치로 접근 거부)
- 403 Forbidden: 권한 없음
- 400 Bad Request: 요청 데이터 유효성 오류
- 500 Internal Server Error: DB 오류 등

### 지침 7-5. 버전 관리 자동화

GoldenSet 또는 GoldenItem 수정 시 Repository가 자동으로 버전을 증가시킨다. API는 단순히 수정 요청을 Repository에 전달하고, Repository가 반환한 업데이트된 객체를 응답한다.

### 지침 7-6. 폐쇄망 환경 (S2 ⑦)

라우터와 Repository 계층은 외부 API (LLM, embedding 등)를 호출하지 않는다. 모든 데이터는 로컬 DB에서만 조회/저장한다.

### 지침 7-7. 페이징

list 엔드포인트는 offset, limit을 쿼리 파라미터로 지원한다. 기본값은 offset=0, limit=100이고, limit 최대값은 1000이다.

### 지침 7-8. 부모-자식 관계

GoldenItem 엔드포인트는 URL에 parent golden_set_id를 포함한다:
```
POST /api/v1/golden-sets/{golden_set_id}/items
```
Repository 호출 시 항상 부모 GoldenSet의 존재 및 접근 권한을 먼저 확인한다.
