# Task 1-10. Prompt 저장소 API (CRUD + 버전 관리)

## 1. 작업 목적

Prompt 도메인 모델을 기반으로 FastAPI RESTful API 엔드포인트를 구현한다. 관리자만 접근 가능한 Admin API로 프롬프트 CRUD, 버전 관리, 활성화를 제공한다. 모든 변경사항은 감사 로깅하고, 표준 envelope 응답 형식을 사용한다.

## 2. 작업 범위

### 포함 범위

1. **FastAPI 라우터 구현**
   - `/backend/app/api/v1/admin/prompts.py` 라우터 파일
   - Admin 권한 확인 (role check) 데코레이터

2. **API 엔드포인트**
   - POST /api/v1/admin/prompts - 프롬프트 생성
   - GET /api/v1/admin/prompts - 프롬프트 목록 (페이지네이션, 카테고리 필터)
   - GET /api/v1/admin/prompts/{id} - 프롬프트 상세 조회 (활성 버전 포함)
   - PUT /api/v1/admin/prompts/{id} - 프롬프트 수정 (새 버전 자동 생성)
   - DELETE /api/v1/admin/prompts/{id} - Soft delete
   - GET /api/v1/admin/prompts/{id}/versions - 버전 이력 조회
   - POST /api/v1/admin/prompts/{id}/versions/{v}/activate - 특정 버전 활성화

3. **Pydantic 스키마**
   - PromptCreateRequest, PromptUpdateRequest
   - PromptResponse, PromptVersionResponse
   - StandardEnvelopeResponse 래퍼

4. **감사 로깅**
   - 모든 POST/PUT/DELETE 작업에 감사 로그 기록
   - actor_type: "user" (S2 원칙 ⑥)

5. **단위 테스트**
   - 각 엔드포인트별 성공/실패 케이스

### 제외 범위

- 사용자 권한 시스템 (기존 S1 Phase 13에서 가정)
- 웹 UI (Phase 6 FG6.1에서)
- 프롬프트 렌더링 로직 (Task 1-11)

## 3. 선행 조건

- Task 1-9 완료 (Prompt, PromptVersion 모델, PromptRepository)
- FastAPI 애플리케이션 기본 구조 완료 (Phase 0)
- 감사 로깅 시스템 존재 (Phase 0 또는 S1 Phase 13/14)
- 사용자 인증/권한 메커니즘 작동

## 4. 주요 작업 항목

### 4-1. Pydantic 스키마

**파일:** `/backend/app/schemas/prompt.py`

**작업 항목:**

```python
from pydantic import BaseModel, Field, field_validator
from typing import Optional, List, Dict, Any
from datetime import datetime
from enum import Enum

class PromptCategoryEnum(str, Enum):
    """프롬프트 카테고리"""
    RETRIEVAL = "retrieval"
    EVALUATION = "evaluation"
    EXTRACTION = "extraction"
    CONVERSATION = "conversation"
    INJECTION_DETECTION = "injection_detection"

class PromptCreateRequest(BaseModel):
    """프롬프트 생성 요청"""
    name: str = Field(..., min_length=1, max_length=255)
    description: Optional[str] = Field(None, max_length=1000)
    category: PromptCategoryEnum = Field(...)
    template: str = Field(..., min_length=1)
    
    @field_validator('name')
    @classmethod
    def validate_name(cls, v):
        """이름은 알파벳, 숫자, 언더스코어만 허용"""
        if not all(c.isalnum() or c == '_' for c in v):
            raise ValueError('Name must contain only alphanumeric characters and underscores')
        return v
    
    class Config:
        json_schema_extra = {
            "example": {
                "name": "retrieval_rewrite_prompt",
                "description": "재작성된 검색 질의 프롬프트",
                "category": "retrieval",
                "template": "당신은 검색 질의 재작성 전문가입니다.\n원본 질의: {{ original_query }}"
            }
        }

class PromptUpdateRequest(BaseModel):
    """프롬프트 수정 요청"""
    template: str = Field(..., min_length=1)
    description: Optional[str] = Field(None, max_length=1000)
    
    class Config:
        json_schema_extra = {
            "example": {
                "template": "새로운 템플릿 내용",
                "description": "업데이트된 설명"
            }
        }

class PromptVersionResponse(BaseModel):
    """프롬프트 버전 응답"""
    id: str
    version: int
    is_active: bool
    created_at: datetime
    usage_count: int
    average_quality_score: Optional[float] = None
    
    class Config:
        from_attributes = True

class PromptResponse(BaseModel):
    """프롬프트 응답"""
    id: str
    name: str
    description: Optional[str]
    category: str
    template: str
    variables: List[str]
    created_at: datetime
    updated_at: datetime
    created_by: str
    is_deleted: bool
    active_version: Optional[PromptVersionResponse] = None
    
    class Config:
        from_attributes = True

class PromptListResponse(BaseModel):
    """프롬프트 목록 응답"""
    id: str
    name: str
    category: str
    created_at: datetime
    updated_at: datetime
    active_version: Optional[int] = None
    
    class Config:
        from_attributes = True

class StandardEnvelopeResponse(BaseModel):
    """표준 응답 envelope"""
    success: bool
    data: Optional[Dict[str, Any]] = None
    error: Optional[Dict[str, str]] = None
    
    class Config:
        json_schema_extra = {
            "example": {
                "success": True,
                "data": {"id": "uuid", "name": "prompt_name"},
                "error": None
            }
        }
```

### 4-2. FastAPI 라우터 구현

**파일:** `/backend/app/api/v1/admin/prompts.py`

**작업 항목:**

```python
from fastapi import APIRouter, Depends, HTTPException, Query, status
from sqlalchemy.orm import Session
from typing import Optional, List
from datetime import datetime

from backend.app.db.session import get_db
from backend.app.models.prompt import Prompt, PromptVersion
from backend.app.repositories.prompt_repository import PromptRepository
from backend.app.schemas.prompt import (
    PromptCreateRequest, PromptUpdateRequest, PromptResponse,
    PromptListResponse, PromptVersionResponse, StandardEnvelopeResponse
)
from backend.app.services.auth import get_current_user  # 기존 권한 시스템 가정
from backend.app.services.audit import log_audit  # 기존 감사 로깅 시스템

router = APIRouter(prefix="/api/v1/admin/prompts", tags=["Admin - Prompts"])

async def check_admin(current_user = Depends(get_current_user)):
    """Admin 권한 확인"""
    if current_user.role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required")
    return current_user

@router.post("", response_model=StandardEnvelopeResponse, status_code=201)
async def create_prompt(
    req: PromptCreateRequest,
    db: Session = Depends(get_db),
    admin: dict = Depends(check_admin)
):
    """
    프롬프트 생성
    
    - Admin만 접근 가능
    - v1 PromptVersion 자동 생성
    - 감사 로깅 기록
    """
    try:
        repo = PromptRepository(db)
        
        # 중복 확인
        existing = repo.get_by_name(req.name)
        if existing:
            raise HTTPException(
                status_code=409,
                detail=f"Prompt name already exists: {req.name}"
            )
        
        # 프롬프트 생성
        prompt = repo.create(
            name=req.name,
            description=req.description,
            category=req.category.value,
            template=req.template,
            created_by=admin.id
        )
        
        db.commit()
        
        # 감사 로깅
        await log_audit(
            db,
            action="PROMPT_CREATE",
            resource_type="Prompt",
            resource_id=str(prompt.id),
            actor_id=admin.id,
            actor_type="user",
            details={"name": req.name, "category": req.category.value}
        )
        
        return StandardEnvelopeResponse(
            success=True,
            data={
                "id": str(prompt.id),
                "name": prompt.name,
                "category": prompt.category
            }
        )
    
    except HTTPException:
        raise
    except Exception as e:
        db.rollback()
        raise HTTPException(status_code=500, detail=str(e))

@router.get("", response_model=StandardEnvelopeResponse)
async def list_prompts(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    category: Optional[str] = Query(None),
    db: Session = Depends(get_db),
    admin: dict = Depends(check_admin)
):
    """
    프롬프트 목록 조회
    
    - 페이지네이션 지원 (skip, limit)
    - 카테고리 필터 지원
    """
    try:
        repo = PromptRepository(db)
        prompts = repo.list_all(category=category, skip=skip, limit=limit)
        
        result = []
        for p in prompts:
            active_version = repo.get_active_version(str(p.id))
            result.append({
                "id": str(p.id),
                "name": p.name,
                "category": p.category,
                "created_at": p.created_at,
                "updated_at": p.updated_at,
                "active_version": active_version.version if active_version else None
            })
        
        return StandardEnvelopeResponse(success=True, data={"items": result, "total": len(result)})
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/{prompt_id}", response_model=StandardEnvelopeResponse)
async def get_prompt(
    prompt_id: str,
    db: Session = Depends(get_db),
    admin: dict = Depends(check_admin)
):
    """
    프롬프트 상세 조회
    
    - 활성 버전 포함
    """
    try:
        repo = PromptRepository(db)
        prompt = repo.get_by_id(prompt_id)
        
        if not prompt:
            raise HTTPException(status_code=404, detail="Prompt not found")
        
        active_version = repo.get_active_version(prompt_id)
        active_version_data = None
        if active_version:
            active_version_data = {
                "id": str(active_version.id),
                "version": active_version.version,
                "is_active": active_version.is_active,
                "created_at": active_version.created_at,
                "usage_count": active_version.usage_count,
                "average_quality_score": active_version.average_quality_score
            }
        
        return StandardEnvelopeResponse(
            success=True,
            data={
                "id": str(prompt.id),
                "name": prompt.name,
                "description": prompt.description,
                "category": prompt.category,
                "template": prompt.template,
                "variables": prompt.variables,
                "created_at": prompt.created_at,
                "updated_at": prompt.updated_at,
                "created_by": prompt.created_by,
                "active_version": active_version_data
            }
        )
    
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.put("/{prompt_id}", response_model=StandardEnvelopeResponse)
async def update_prompt(
    prompt_id: str,
    req: PromptUpdateRequest,
    db: Session = Depends(get_db),
    admin: dict = Depends(check_admin)
):
    """
    프롬프트 수정
    
    - 새 PromptVersion 자동 생성
    - 감사 로깅 기록
    """
    try:
        repo = PromptRepository(db)
        prompt = repo.update(
            prompt_id=prompt_id,
            template=req.template,
            description=req.description
        )
        
        if not prompt:
            raise HTTPException(status_code=404, detail="Prompt not found")
        
        db.commit()
        
        # 감사 로깅
        await log_audit(
            db,
            action="PROMPT_UPDATE",
            resource_type="Prompt",
            resource_id=prompt_id,
            actor_id=admin.id,
            actor_type="user",
            details={"new_version": len(prompt.versions)}
        )
        
        return StandardEnvelopeResponse(
            success=True,
            data={"id": prompt_id, "new_version": len(prompt.versions)}
        )
    
    except HTTPException:
        raise
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))
    except Exception as e:
        db.rollback()
        raise HTTPException(status_code=500, detail=str(e))

@router.delete("/{prompt_id}", response_model=StandardEnvelopeResponse)
async def delete_prompt(
    prompt_id: str,
    db: Session = Depends(get_db),
    admin: dict = Depends(check_admin)
):
    """
    프롬프트 삭제 (Soft delete)
    
    - 감사 로깅 기록
    """
    try:
        repo = PromptRepository(db)
        prompt = repo.soft_delete(prompt_id)
        
        if not prompt:
            raise HTTPException(status_code=404, detail="Prompt not found")
        
        db.commit()
        
        # 감사 로깅
        await log_audit(
            db,
            action="PROMPT_DELETE",
            resource_type="Prompt",
            resource_id=prompt_id,
            actor_id=admin.id,
            actor_type="user",
            details={"name": prompt.name}
        )
        
        return StandardEnvelopeResponse(success=True, data={"id": prompt_id})
    
    except HTTPException:
        raise
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))
    except Exception as e:
        db.rollback()
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/{prompt_id}/versions", response_model=StandardEnvelopeResponse)
async def list_versions(
    prompt_id: str,
    db: Session = Depends(get_db),
    admin: dict = Depends(check_admin)
):
    """
    버전 이력 조회
    
    - 생성 시간 역순 정렬
    """
    try:
        repo = PromptRepository(db)
        prompt = repo.get_by_id(prompt_id)
        
        if not prompt:
            raise HTTPException(status_code=404, detail="Prompt not found")
        
        versions = repo.list_versions(prompt_id)
        
        version_list = [
            {
                "id": str(v.id),
                "version": v.version,
                "is_active": v.is_active,
                "created_at": v.created_at,
                "usage_count": v.usage_count,
                "average_quality_score": v.average_quality_score
            }
            for v in versions
        ]
        
        return StandardEnvelopeResponse(
            success=True,
            data={"prompt_id": prompt_id, "versions": version_list}
        )
    
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/{prompt_id}/versions/{version}/activate", response_model=StandardEnvelopeResponse)
async def activate_version(
    prompt_id: str,
    version: int,
    db: Session = Depends(get_db),
    admin: dict = Depends(check_admin)
):
    """
    특정 버전 활성화
    
    - 이전 활성 버전 비활성화
    - 감사 로깅 기록
    """
    try:
        repo = PromptRepository(db)
        
        prompt = repo.get_by_id(prompt_id)
        if not prompt:
            raise HTTPException(status_code=404, detail="Prompt not found")
        
        target_version = repo.activate_version(prompt_id, version)
        db.commit()
        
        # 감사 로깅
        await log_audit(
            db,
            action="PROMPT_ACTIVATE_VERSION",
            resource_type="PromptVersion",
            resource_id=str(target_version.id),
            actor_id=admin.id,
            actor_type="user",
            details={"prompt_id": prompt_id, "version": version}
        )
        
        return StandardEnvelopeResponse(
            success=True,
            data={"prompt_id": prompt_id, "activated_version": version}
        )
    
    except HTTPException:
        raise
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))
    except Exception as e:
        db.rollback()
        raise HTTPException(status_code=500, detail=str(e))
```

### 4-3. 라우터 등록

**파일:** `/backend/app/api/v1/admin/__init__.py` 또는 main FastAPI 앱

**작업 항목:**

```python
# main.py 또는 api router 조합 파일
from fastapi import FastAPI
from backend.app.api.v1.admin import prompts

app = FastAPI()

# Admin API 라우터 등록
app.include_router(prompts.router)
```

## 5. 산출물

1. **Pydantic 스키마** (`/backend/app/schemas/prompt.py`)
   - PromptCreateRequest, PromptUpdateRequest
   - PromptResponse, PromptListResponse, PromptVersionResponse
   - StandardEnvelopeResponse

2. **FastAPI 라우터** (`/backend/app/api/v1/admin/prompts.py`)
   - 7개 엔드포인트 (POST, GET, PUT, DELETE, GET versions, POST activate)
   - Admin 권한 확인 미들웨어
   - 감사 로깅 통합

3. **단위 테스트** (`/backend/tests/unit/api/test_admin_prompts.py`)
   - 각 엔드포인트별 성공/실패 케이스
   - 권한 검증 테스트
   - 감사 로그 기록 확인

## 6. 완료 기준

1. 모든 7개 엔드포인트가 구현되고 OpenAPI 문서 생성됨
2. Admin 권한 확인이 정상 작동 (비관리자 접근 시 403 반환)
3. 프롬프트 생성 시 중복 이름 검사 (409 Conflict 반환)
4. 프롬프트 수정 시 새 버전이 자동 생성되고 is_active 전환
5. Soft delete 정상 작동 (삭제 후 조회 불가)
6. 버전 이력 조회 시 생성 시간 역순 정렬
7. 모든 변경 작업에 감사 로그 기록됨 (actor_type="user")
8. 단위 테스트 모두 통과
9. 표준 envelope 응답 형식 일관성 유지

## 7. 작업 지침

### 지침 7-1. Admin 권한 확인

모든 엔드포인트에서 `@Depends(check_admin)` 사용. 기존 사용자 인증 시스템(get_current_user)과 통합:

```python
async def check_admin(current_user = Depends(get_current_user)):
    if current_user.role != "admin":
        raise HTTPException(status_code=403, detail="Admin role required")
    return current_user
```

### 지침 7-2. 감사 로깅 (S2 원칙 ⑥)

모든 POST/PUT/DELETE에서 log_audit() 호출. actor_type은 항상 "user":

```python
await log_audit(
    db,
    action="PROMPT_CREATE",
    resource_type="Prompt",
    resource_id=str(prompt.id),
    actor_id=admin.id,
    actor_type="user",  # 항상 "user" (이 경우 admin도 user)
    details={...}
)
```

### 지침 7-3. 표준 Envelope 응답

모든 응답은 StandardEnvelopeResponse로 감싸기:

```json
{
  "success": true,
  "data": {...},
  "error": null
}
```

### 지침 7-4. 에러 처리

- 존재하지 않는 리소스: 404
- 중복 이름: 409 Conflict
- 권한 부족: 403 Forbidden
- 서버 오류: 500 + 트랜잭션 롤백
