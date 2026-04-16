# Task 8-2: ExtractionTargetSchema CRUD API + 버전 관리

**작업 식별자**: FG8.1-task8-2  
**작업 단계**: Phase 8 - DocumentType metadata schema → 추출 타겟 스키마  
**예상 소요 시간**: 28-36시간  
**담당자**: [Backend/API Engineer]  

---

## 1. 작업 목적

Task 8-1에서 구현한 도메인 모델 및 Repository를 기반으로, 외부 클라이언트(UI, 에이전트, 써드파티)가 추출 스키마를 관리할 수 있는 **RESTful API 엔드포인트**를 제공한다.

주요 목표:
1. **5개 CRUD 엔드포인트** — POST(생성), GET(단일), PUT(수정), DELETE(삭제), GET(리스트)
2. **버전 관리 API** — GET /versions 엔드포인트로 버전 이력 및 변경사항 조회
3. **Scope Profile ACL** — 모든 엔드포인트에서 Scope Profile 기반 접근 제어
4. **actor_type 추적** — 감사 로그에 "user" 또는 "agent" 구분 기록
5. **unwrapEnvelope 응답 포맷** — 일관된 응답 형식 제공
6. **입력 검증** — 필드 타입, 제약조건 일관성 검증
7. **통합 테스트** — API 레벨 E2E 테스트

---

## 2. 작업 범위

### 포함 범위

#### 2.1 FastAPI Router 설계

**5개 핵심 엔드포인트**

1. **POST /api/v1/extraction-schemas**
   - 요청: CreateExtractionSchemaRequest (doc_type_id, fields, scope_profile_id)
   - 응답: ExtractionSchemaResponse (201 Created)
   - 역할: 새로운 추출 스키마 생성
   - 권한: 해당 DocumentType에 대한 write 권한 필수

2. **GET /api/v1/extraction-schemas/{doc_type}**
   - 요청: Query params (scope_profile_id optional)
   - 응답: ExtractionSchemaResponse (200 OK)
   - 역할: 특정 DocumentType의 최신 추출 스키마 조회
   - 권한: 해당 DocumentType에 대한 read 권한 필수

3. **PUT /api/v1/extraction-schemas/{doc_type}**
   - 요청: UpdateExtractionSchemaRequest (fields, change_summary, scope_profile_id)
   - 응답: ExtractionSchemaResponse (200 OK, version 증가)
   - 역할: 추출 스키마 업데이트 (새 버전 생성)
   - 권한: 해당 DocumentType에 대한 write 권한 필수

4. **DELETE /api/v1/extraction-schemas/{doc_type}**
   - 요청: Path param doc_type, optional reason in body
   - 응답: DeleteExtractionSchemaResponse (204 No Content)
   - 역할: 추출 스키마 소프트 삭제
   - 권한: 해당 DocumentType에 대한 admin 권한 필수

5. **GET /api/v1/extraction-schemas/{doc_type}/versions**
   - 요청: Query params (limit=10, offset=0, scope_profile_id optional)
   - 응답: List[ExtractionSchemaVersionResponse] (200 OK)
   - 역할: 버전 이력 및 변경사항 조회
   - 권한: 해당 DocumentType에 대한 read 권한 필수

#### 2.2 요청/응답 Pydantic 모델

**CreateExtractionSchemaRequest**
```
{
  "doc_type_id": "uuid",
  "fields": {
    "field_name": {
      "field_type": "string",
      "required": true,
      "description": "...",
      "pattern": "regex",
      "instruction": "...",
      "examples": ["ex1", "ex2"],
      "max_length": 255,
      ...
    },
    ...
  },
  "scope_profile_id": "uuid (optional)",
  "metadata": { ... } (optional)
}
```

**UpdateExtractionSchemaRequest**
```
{
  "fields": { ... },
  "change_summary": "필드 추가",
  "scope_profile_id": "uuid (optional)"
}
```

**ExtractionSchemaResponse** (unwrapEnvelope)
```
{
  "data": {
    "id": "uuid",
    "doc_type_id": "uuid",
    "version": 2,
    "fields": { ... },
    "is_deprecated": false,
    "deprecation_reason": null,
    "created_at": "2026-04-17T00:00:00Z",
    "updated_at": "2026-04-17T00:00:00Z",
    "created_by": "user_001",
    "updated_by": "user_001"
  },
  "meta": {
    "timestamp": "2026-04-17T00:00:00Z",
    "request_id": "req_xyz"
  }
}
```

**ExtractionSchemaVersionResponse**
```
{
  "version": 1,
  "created_at": "2026-04-17T00:00:00Z",
  "created_by": "user_001",
  "change_summary": "초기 생성",
  "changed_fields": ["field_name1", "field_name2"],
  "fields_snapshot": { ... }
}
```

**DeleteExtractionSchemaResponse**
```
{
  "data": {
    "doc_type_id": "uuid",
    "deleted_at": "2026-04-17T00:00:00Z",
    "deleted_by": "user_001"
  },
  "meta": { ... }
}
```

#### 2.3 에러 처리

- **400 Bad Request**: 필드 타입/제약조건 오류, 정규식 오류 등
- **401 Unauthorized**: 인증 실패
- **403 Forbidden**: Scope Profile ACL 권한 부족
- **404 Not Found**: DocumentType 또는 스키마 없음
- **409 Conflict**: 스키마 버전 충돌
- **422 Unprocessable Entity**: 입력 데이터 검증 실패
- **500 Internal Server Error**: 예상 불가능한 오류

각 오류 응답:
```json
{
  "error": {
    "code": "SCHEMA_VALIDATION_ERROR",
    "message": "필드 'invoice_date'의 date_format이 YYYY-MM-DD가 아님",
    "details": {
      "field_name": "invoice_date",
      "reason": "format_mismatch"
    }
  },
  "meta": {
    "request_id": "req_xyz",
    "timestamp": "2026-04-17T00:00:00Z"
  }
}
```

#### 2.4 Scope Profile ACL 및 actor_type 통합

**요청 헤더에서 actor_info 추출**
- Authorization 헤더에서 user_id 또는 agent_id 추출
- actor_type: "user" (일반 사용자) 또는 "agent" (AI 에이전트)
- actor_type 판별: Authorization 헤더 형식 또는 특수 헤더 사용

**권한 검증**
- Scope Profile Service와 연동하여 doc_type_id에 대한 접근 권한 확인
- 권한 없으면 403 Forbidden 반환

**감시 로그(Audit Log) 기록**
- 모든 쓰기 작업(POST, PUT, DELETE)에 대해 audit_logs 테이블에 기록
- actor_type 필드 필수 포함
- 샘플:
  ```sql
  INSERT INTO audit_logs (
    id, actor_id, actor_type, action, resource_type, resource_id,
    old_value, new_value, created_at
  ) VALUES (
    uuid_generate_v4(), 'user_001', 'user', 'CREATE', 'extraction_schema',
    'schema_id', NULL, '{"doc_type_id": "...", "version": 1}',
    NOW()
  );
  ```

#### 2.5 API 라우터 구현

**파일**: `src/api/v1/routers/extraction_schemas.py`

```python
from typing import Optional, List
from uuid import UUID
from fastapi import APIRouter, Depends, HTTPException, Header, status
from sqlalchemy.orm import Session
from datetime import datetime

from src.api.v1.schemas import (
    CreateExtractionSchemaRequest,
    UpdateExtractionSchemaRequest,
    ExtractionSchemaResponse,
    ExtractionSchemaVersionResponse,
    DeleteExtractionSchemaResponse,
    ErrorResponse,
    unwrap_envelope,
)
from src.domain.extraction.models import ExtractionTargetSchema
from src.domain.extraction.repository import (
    ExtractionSchemaRepository,
    ActorInfo,
)
from src.infrastructure.database.connection import get_session
from src.infrastructure.scope.service import ScopeProfileService
from src.infrastructure.audit import AuditLogger
from src.infrastructure.exceptions import (
    NotFoundError,
    ForbiddenError,
    ValidationError,
)


router = APIRouter(prefix="/extraction-schemas", tags=["extraction-schemas"])


def get_actor_info(
    authorization: Optional[str] = Header(None),
    x_actor_type: Optional[str] = Header(None),
) -> ActorInfo:
    """요청 헤더에서 actor_info 추출"""
    if not authorization:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Authorization 헤더 필수"
        )
    
    # Bearer token 파싱: "Bearer user_001" 또는 "Bearer agent_gpt4"
    try:
        scheme, credentials = authorization.split(" ", 1)
        if scheme.lower() != "bearer":
            raise ValueError("Bearer 스키마만 지원")
        
        # credentials에서 actor_id, actor_type 추출
        # 형식: "user_001" (user 타입) 또는 "agent_gpt4" (agent 타입)
        actor_type = x_actor_type or ("agent" if credentials.startswith("agent_") else "user")
        
        return ActorInfo(actor_id=credentials, actor_type=actor_type)
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=f"Authorization 헤더 파싱 오류: {e}"
        )


@router.post(
    "",
    response_model=ExtractionSchemaResponse,
    status_code=status.HTTP_201_CREATED,
    summary="추출 스키마 생성",
)
async def create_extraction_schema(
    request: CreateExtractionSchemaRequest,
    session: Session = Depends(get_session),
    actor_info: ActorInfo = Depends(get_actor_info),
    scope_service: ScopeProfileService = Depends(),
    audit_logger: AuditLogger = Depends(),
):
    """
    새로운 추출 스키마를 생성합니다.
    
    - DocumentType별로 최대 1개의 활성 스키마만 존재 가능
    - 필드 정의는 필드 타입과 일관성 있어야 함
    - actor_type은 감시 로그에 기록됨
    """
    try:
        # 필드 모델 변환
        fields = {
            field_name: ExtractionFieldDef(**field_data)
            for field_name, field_data in request.fields.items()
        }
        
        # Repository 작업
        repository = ExtractionSchemaRepository(
            session=session,
            scope_service=scope_service,
        )
        
        schema = await repository.create(
            doc_type_id=request.doc_type_id,
            fields=fields,
            actor_info=actor_info,
            scope_profile_id=request.scope_profile_id,
            metadata=request.metadata,
        )
        
        # 감시 로그
        await audit_logger.log(
            actor_id=actor_info.actor_id,
            actor_type=actor_info.actor_type,
            action="CREATE",
            resource_type="extraction_schema",
            resource_id=str(schema.id),
            new_value={
                "doc_type_id": str(request.doc_type_id),
                "version": schema.version,
                "field_count": len(schema.fields),
            },
        )
        
        return unwrap_envelope(schema, request_id="req_xyz")
    
    except ValidationError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e),
        )
    except ForbiddenError as e:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=str(e),
        )
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"서버 오류: {e}",
        )


@router.get(
    "/{doc_type}",
    response_model=ExtractionSchemaResponse,
    summary="추출 스키마 조회",
)
def get_extraction_schema(
    doc_type: UUID,
    scope_profile_id: Optional[UUID] = None,
    session: Session = Depends(get_session),
    actor_info: ActorInfo = Depends(get_actor_info),
    scope_service: ScopeProfileService = Depends(),
):
    """
    특정 DocumentType의 최신 추출 스키마를 조회합니다.
    
    - 가장 최신 버전만 반환
    - scope_profile_id가 있으면 ACL 필터링 적용
    """
    try:
        repository = ExtractionSchemaRepository(
            session=session,
            scope_service=scope_service,
        )
        
        schema = repository.get_by_doc_type(
            doc_type_id=doc_type,
            scope_profile_id=scope_profile_id,
        )
        
        if not schema:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"DocumentType {doc_type}에 대한 추출 스키마 없음",
            )
        
        return unwrap_envelope(schema, request_id="req_xyz")
    
    except ForbiddenError as e:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=str(e),
        )
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"서버 오류: {e}",
        )


@router.put(
    "/{doc_type}",
    response_model=ExtractionSchemaResponse,
    summary="추출 스키마 업데이트",
)
async def update_extraction_schema(
    doc_type: UUID,
    request: UpdateExtractionSchemaRequest,
    session: Session = Depends(get_session),
    actor_info: ActorInfo = Depends(get_actor_info),
    scope_service: ScopeProfileService = Depends(),
    audit_logger: AuditLogger = Depends(),
):
    """
    추출 스키마를 업데이트합니다 (새 버전 생성).
    
    - 버전 번호는 자동 증가
    - 변경된 필드는 change_summary에 기록
    - 이전 버전은 보존됨
    """
    try:
        fields = {
            field_name: ExtractionFieldDef(**field_data)
            for field_name, field_data in request.fields.items()
        }
        
        repository = ExtractionSchemaRepository(
            session=session,
            scope_service=scope_service,
        )
        
        schema = await repository.update(
            doc_type_id=doc_type,
            fields=fields,
            actor_info=actor_info,
            scope_profile_id=request.scope_profile_id,
            change_summary=request.change_summary,
        )
        
        # 감시 로그
        await audit_logger.log(
            actor_id=actor_info.actor_id,
            actor_type=actor_info.actor_type,
            action="UPDATE",
            resource_type="extraction_schema",
            resource_id=str(schema.id),
            new_value={"version": schema.version},
        )
        
        return unwrap_envelope(schema, request_id="req_xyz")
    
    except ValidationError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e),
        )
    except NotFoundError as e:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=str(e),
        )
    except ForbiddenError as e:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=str(e),
        )
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"서버 오류: {e}",
        )


@router.delete(
    "/{doc_type}",
    response_model=DeleteExtractionSchemaResponse,
    status_code=status.HTTP_204_NO_CONTENT,
    summary="추출 스키마 삭제",
)
async def delete_extraction_schema(
    doc_type: UUID,
    session: Session = Depends(get_session),
    actor_info: ActorInfo = Depends(get_actor_info),
    audit_logger: AuditLogger = Depends(),
):
    """
    추출 스키마를 소프트 삭제합니다.
    
    - 실제로는 is_soft_deleted=True로 표시
    - restore() 메서드로 복구 가능 (admin only)
    """
    try:
        repository = ExtractionSchemaRepository(session=session)
        
        deleted = repository.delete(doc_type_id=doc_type, actor_info=actor_info)
        
        if not deleted:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail=f"DocumentType {doc_type}에 대한 추출 스키마 없음",
            )
        
        # 감시 로그
        await audit_logger.log(
            actor_id=actor_info.actor_id,
            actor_type=actor_info.actor_type,
            action="DELETE",
            resource_type="extraction_schema",
            resource_id=str(doc_type),
            new_value={"is_soft_deleted": True},
        )
        
        return DeleteExtractionSchemaResponse(
            data={
                "doc_type_id": str(doc_type),
                "deleted_at": datetime.utcnow(),
                "deleted_by": actor_info.actor_id,
            }
        )
    
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"서버 오류: {e}",
        )


@router.get(
    "/{doc_type}/versions",
    response_model=List[ExtractionSchemaVersionResponse],
    summary="스키마 버전 이력",
)
def get_extraction_schema_versions(
    doc_type: UUID,
    limit: int = 10,
    offset: int = 0,
    scope_profile_id: Optional[UUID] = None,
    session: Session = Depends(get_session),
    actor_info: ActorInfo = Depends(get_actor_info),
    scope_service: ScopeProfileService = Depends(),
):
    """
    추출 스키마의 버전 이력을 조회합니다.
    
    - 최신 버전부터 내림차순 정렬
    - 각 버전의 변경사항(change_summary, changed_fields) 포함
    - 필드 스냅샷(fields_snapshot)은 버전별로 다름
    """
    try:
        if limit > 100:
            limit = 100
        
        repository = ExtractionSchemaRepository(
            session=session,
            scope_service=scope_service,
        )
        
        versions = repository.get_versions(
            doc_type_id=doc_type,
            scope_profile_id=scope_profile_id,
            limit=limit,
            offset=offset,
        )
        
        # ExtractionSchemaVersionResponse로 변환
        return [
            ExtractionSchemaVersionResponse(
                version=v.version,
                created_at=v.created_at,
                created_by=v.created_by,
                change_summary=v.metadata.get("change_summary"),
                changed_fields=v.metadata.get("changed_fields", []),
                fields_snapshot=v.fields,
            )
            for v in versions
        ]
    
    except ForbiddenError as e:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=str(e),
        )
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"서버 오류: {e}",
        )
```

#### 2.6 요청/응답 스키마

**파일**: `src/api/v1/schemas/extraction_schemas.py`

```python
from typing import Any, Dict, List, Optional
from uuid import UUID
from datetime import datetime
from pydantic import BaseModel, Field


class ExtractionFieldDefRequest(BaseModel):
    """필드 정의 요청"""
    field_name: str
    field_type: str
    required: bool = False
    description: str
    pattern: Optional[str] = None
    instruction: Optional[str] = None
    examples: List[str] = []
    max_length: Optional[int] = None
    min_value: Optional[float] = None
    max_value: Optional[float] = None
    date_format: Optional[str] = None
    enum_values: Optional[List[str]] = None
    default_value: Optional[Any] = None
    nested_schema: Optional[Dict[str, "ExtractionFieldDefRequest"]] = None


class CreateExtractionSchemaRequest(BaseModel):
    """추출 스키마 생성 요청"""
    doc_type_id: UUID = Field(..., description="DocumentType ID")
    fields: Dict[str, ExtractionFieldDefRequest] = Field(
        ...,
        description="필드 정의 맵"
    )
    scope_profile_id: Optional[UUID] = Field(
        None,
        description="Scope Profile ID (optional)"
    )
    metadata: Optional[Dict[str, Any]] = Field(
        None,
        description="추가 메타데이터"
    )


class UpdateExtractionSchemaRequest(BaseModel):
    """추출 스키마 업데이트 요청"""
    fields: Dict[str, ExtractionFieldDefRequest] = Field(
        ...,
        description="업데이트된 필드 정의"
    )
    change_summary: Optional[str] = Field(
        None,
        max_length=1024,
        description="변경 요약"
    )
    scope_profile_id: Optional[UUID] = Field(
        None,
        description="Scope Profile ID (optional)"
    )


class ExtractionSchemaResponse(BaseModel):
    """추출 스키마 응답"""
    data: Dict[str, Any] = Field(..., description="스키마 데이터")
    meta: Dict[str, Any] = Field(
        ...,
        description="메타데이터 (timestamp, request_id 등)"
    )


class ExtractionSchemaVersionResponse(BaseModel):
    """버전 이력 응답"""
    version: int = Field(..., description="버전 번호")
    created_at: datetime = Field(..., description="생성 시각")
    created_by: str = Field(..., description="생성자 ID")
    change_summary: Optional[str] = Field(None, description="변경 요약")
    changed_fields: List[str] = Field(default_factory=list, description="변경된 필드")
    fields_snapshot: Dict[str, Any] = Field(..., description="필드 스냅샷")


class DeleteExtractionSchemaResponse(BaseModel):
    """삭제 응답"""
    data: Dict[str, Any] = Field(..., description="삭제 정보")
    meta: Dict[str, Any] = Field(..., description="메타데이터")


def unwrap_envelope(
    data: Any,
    request_id: str,
    timestamp: Optional[datetime] = None,
) -> Dict[str, Any]:
    """응답을 unwrapEnvelope 포맷으로 변환"""
    return {
        "data": data.model_dump() if hasattr(data, "model_dump") else data,
        "meta": {
            "request_id": request_id,
            "timestamp": timestamp or datetime.utcnow().isoformat(),
        },
    }
```

#### 2.7 메인 앱 통합

**파일**: `src/api/v1/main.py` (기존 수정)

```python
from fastapi import FastAPI
from src.api.v1.routers import extraction_schemas

app = FastAPI(title="Mimir API", version="1.0.0")

# 라우터 등록
app.include_router(extraction_schemas.router, prefix="/api/v1")
```

#### 2.8 감시 로그 구현

**파일**: `src/infrastructure/audit.py`

```python
from datetime import datetime
from typing import Any, Dict, Optional
from uuid import UUID, uuid4
from sqlalchemy.orm import Session
from sqlalchemy import Column, String, DateTime, Text, func
from src.infrastructure.database.base import Base


class AuditLogORM(Base):
    """감시 로그 ORM"""
    __tablename__ = "audit_logs"
    
    id = Column(UUID, primary_key=True, default=uuid4)
    actor_id = Column(String(255), nullable=False, index=True)
    actor_type = Column(String(50), nullable=False)  # "user" or "agent"
    action = Column(String(50), nullable=False, index=True)  # CREATE, READ, UPDATE, DELETE
    resource_type = Column(String(100), nullable=False, index=True)
    resource_id = Column(String(255), nullable=False, index=True)
    old_value = Column(Text, nullable=True)  # JSON
    new_value = Column(Text, nullable=True)  # JSON
    created_at = Column(DateTime(timezone=True), nullable=False, default=func.now(), index=True)


class AuditLogger:
    """감시 로그 기록자"""
    
    def __init__(self, session: Session):
        self.session = session
    
    async def log(
        self,
        actor_id: str,
        actor_type: str,
        action: str,
        resource_type: str,
        resource_id: str,
        old_value: Optional[Dict[str, Any]] = None,
        new_value: Optional[Dict[str, Any]] = None,
    ) -> None:
        """감시 로그 기록"""
        import json
        
        log_entry = AuditLogORM(
            actor_id=actor_id,
            actor_type=actor_type,
            action=action,
            resource_type=resource_type,
            resource_id=resource_id,
            old_value=json.dumps(old_value) if old_value else None,
            new_value=json.dumps(new_value) if new_value else None,
        )
        
        self.session.add(log_entry)
        self.session.commit()
```

---

### 제외 범위

- 인증(Authentication) 구현 (별도 작업)
- UI 통합 (별도 작업)
- OpenAPI 문서 자동 생성 (FastAPI 기본 제공)
- 성능 최적화(캐싱) — 기본 구현만 (task8-3에서 보강 가능)

---

## 3. 선행 조건

1. **Task 8-1 완료**
   - ExtractionTargetSchema, ExtractionFieldDef 도메인 모델 완성
   - ExtractionSchemaRepository 구현 완성
   - DB 마이그레이션 적용 완료

2. **FastAPI 환경 설정**
   - FastAPI 0.100+, Pydantic v2 설치
   - ASGI 서버(uvicorn) 설정

3. **ScopeProfileService 구현**
   - check_access() 메서드 제공
   - Scope Profile 데이터 조회 기능

4. **AuditLogger 기반 구현**
   - audit_logs 테이블 존재
   - actor_type 필드 지원

5. **테스트 환경**
   - pytest, pytest-asyncio, httpx (async HTTP client)
   - 테스트용 FastAPI TestClient

---

## 4. 주요 작업 항목

### 4.1 요청/응답 Pydantic 모델 정의

**파일**: `src/api/v1/schemas/extraction_schemas.py`

(위 2.6 섹션 참조)

### 4.2 FastAPI Router 구현

**파일**: `src/api/v1/routers/extraction_schemas.py`

(위 2.5 섹션 참조)

### 4.3 감시 로그 통합

**파일**: `src/infrastructure/audit.py`

(위 2.8 섹션 참조)

### 4.4 에러 응답 처리

**파일**: `src/api/v1/middleware/error_handler.py`

```python
from fastapi import Request, status
from fastapi.responses import JSONResponse
from src.infrastructure.exceptions import (
    ValidationError,
    NotFoundError,
    ForbiddenError,
)


async def global_exception_handler(request: Request, exc: Exception):
    """전역 예외 처리"""
    
    error_code = "INTERNAL_ERROR"
    status_code = status.HTTP_500_INTERNAL_SERVER_ERROR
    message = "서버 오류"
    
    if isinstance(exc, ValidationError):
        error_code = "SCHEMA_VALIDATION_ERROR"
        status_code = status.HTTP_400_BAD_REQUEST
        message = str(exc)
    elif isinstance(exc, NotFoundError):
        error_code = "NOT_FOUND"
        status_code = status.HTTP_404_NOT_FOUND
        message = str(exc)
    elif isinstance(exc, ForbiddenError):
        error_code = "FORBIDDEN"
        status_code = status.HTTP_403_FORBIDDEN
        message = str(exc)
    
    return JSONResponse(
        status_code=status_code,
        content={
            "error": {
                "code": error_code,
                "message": message,
            },
            "meta": {
                "request_id": request.headers.get("x-request-id", "unknown"),
                "timestamp": datetime.utcnow().isoformat(),
            },
        },
    )
```

### 4.5 통합 테스트

**파일**: `tests/api/v1/test_extraction_schemas_api.py`

```python
import pytest
from uuid import uuid4
from httpx import AsyncClient
from datetime import datetime

from src.api.v1.main import app
from src.infrastructure.database.connection import get_session
from tests.fixtures import override_get_session, create_test_session


@pytest.fixture
async def client():
    """FastAPI TestClient"""
    app.dependency_overrides[get_session] = override_get_session
    async with AsyncClient(app=app, base_url="http://test") as c:
        yield c


class TestExtractionSchemasAPI:
    """추출 스키마 API 테스트"""
    
    @pytest.mark.asyncio
    async def test_create_extraction_schema(self, client):
        """추출 스키마 생성 테스트"""
        doc_type_id = uuid4()
        
        payload = {
            "doc_type_id": str(doc_type_id),
            "fields": {
                "invoice_number": {
                    "field_name": "invoice_number",
                    "field_type": "string",
                    "required": True,
                    "description": "인보이스 번호",
                    "pattern": r"^INV-\d{6}$",
                    "examples": ["INV-000001"],
                    "max_length": 20,
                },
                "total_amount": {
                    "field_name": "total_amount",
                    "field_type": "number",
                    "required": True,
                    "description": "총액",
                    "examples": ["1000.50"],
                    "min_value": 0.0,
                    "max_value": 999999.99,
                },
            },
        }
        
        response = await client.post(
            "/api/v1/extraction-schemas",
            json=payload,
            headers={"Authorization": "Bearer user_001"},
        )
        
        assert response.status_code == 201
        data = response.json()
        assert "data" in data
        assert data["data"]["version"] == 1
        assert len(data["data"]["fields"]) == 2
    
    @pytest.mark.asyncio
    async def test_get_extraction_schema(self, client):
        """추출 스키마 조회 테스트"""
        # 먼저 스키마 생성
        doc_type_id = uuid4()
        create_payload = {...}
        create_response = await client.post(
            "/api/v1/extraction-schemas",
            json=create_payload,
            headers={"Authorization": "Bearer user_001"},
        )
        assert create_response.status_code == 201
        
        # 조회
        get_response = await client.get(
            f"/api/v1/extraction-schemas/{doc_type_id}",
            headers={"Authorization": "Bearer user_001"},
        )
        
        assert get_response.status_code == 200
        data = get_response.json()
        assert data["data"]["doc_type_id"] == str(doc_type_id)
    
    @pytest.mark.asyncio
    async def test_update_extraction_schema(self, client):
        """추출 스키마 업데이트 테스트"""
        # 생성 후 업데이트
        doc_type_id = uuid4()
        # ... 생성 로직 ...
        
        update_payload = {
            "fields": {
                "invoice_number": {...},
                "total_amount": {...},
                "invoice_date": {  # 새 필드 추가
                    "field_name": "invoice_date",
                    "field_type": "date",
                    "required": True,
                    "description": "발행일",
                    "date_format": "YYYY-MM-DD",
                    "examples": ["2026-04-17"],
                },
            },
            "change_summary": "invoice_date 필드 추가",
        }
        
        response = await client.put(
            f"/api/v1/extraction-schemas/{doc_type_id}",
            json=update_payload,
            headers={"Authorization": "Bearer user_001"},
        )
        
        assert response.status_code == 200
        data = response.json()
        assert data["data"]["version"] == 2  # 버전 증가
        assert len(data["data"]["fields"]) == 3
    
    @pytest.mark.asyncio
    async def test_delete_extraction_schema(self, client):
        """추출 스키마 삭제 테스트"""
        # ... 생성 로직 ...
        
        response = await client.delete(
            f"/api/v1/extraction-schemas/{doc_type_id}",
            headers={"Authorization": "Bearer user_001"},
        )
        
        assert response.status_code == 204
        
        # 삭제 후 조회 불가
        get_response = await client.get(
            f"/api/v1/extraction-schemas/{doc_type_id}",
            headers={"Authorization": "Bearer user_001"},
        )
        assert get_response.status_code == 404
    
    @pytest.mark.asyncio
    async def test_get_schema_versions(self, client):
        """버전 이력 조회 테스트"""
        # ... 생성 및 업데이트 로직 ...
        
        response = await client.get(
            f"/api/v1/extraction-schemas/{doc_type_id}/versions",
            headers={"Authorization": "Bearer user_001"},
        )
        
        assert response.status_code == 200
        data = response.json()
        assert len(data) >= 2  # 초기 생성 + 최소 1개 업데이트
        assert data[0]["version"] >= data[-1]["version"]  # 역순 정렬
    
    @pytest.mark.asyncio
    async def test_invalid_field_type_error(self, client):
        """필드 타입 오류 테스트"""
        doc_type_id = uuid4()
        
        payload = {
            "doc_type_id": str(doc_type_id),
            "fields": {
                "status": {
                    "field_name": "status",
                    "field_type": "string",
                    "required": True,
                    "description": "상태",
                    "enum_values": ["a", "b"],  # string 타입과 불일치
                    "examples": ["a"],
                },
            },
        }
        
        response = await client.post(
            "/api/v1/extraction-schemas",
            json=payload,
            headers={"Authorization": "Bearer user_001"},
        )
        
        assert response.status_code == 400
        data = response.json()
        assert "error" in data
    
    @pytest.mark.asyncio
    async def test_actor_type_in_audit_log(self, client, db_session):
        """actor_type이 감시 로그에 기록되는지 확인"""
        doc_type_id = uuid4()
        payload = {...}
        
        # agent로 요청
        await client.post(
            "/api/v1/extraction-schemas",
            json=payload,
            headers={
                "Authorization": "Bearer agent_gpt4",
                "X-Actor-Type": "agent",
            },
        )
        
        # 감시 로그 확인
        from src.infrastructure.audit import AuditLogORM
        logs = db_session.query(AuditLogORM).filter(
            AuditLogORM.actor_type == "agent"
        ).all()
        
        assert len(logs) > 0
        assert logs[0].actor_id == "agent_gpt4"
        assert logs[0].actor_type == "agent"
    
    @pytest.mark.asyncio
    async def test_scope_profile_acl(self, client, mock_scope_service):
        """Scope Profile ACL 테스트"""
        doc_type_id = uuid4()
        scope_profile_id = uuid4()
        
        # ACL 거부 상황 모의
        mock_scope_service.check_access.return_value = False
        
        payload = {
            "doc_type_id": str(doc_type_id),
            "fields": {...},
            "scope_profile_id": str(scope_profile_id),
        }
        
        response = await client.post(
            "/api/v1/extraction-schemas",
            json=payload,
            headers={"Authorization": "Bearer user_001"},
        )
        
        assert response.status_code == 403
```

---

## 5. 산출물 (Deliverables)

| 파일 경로 | 설명 | 요구사항 |
|---------|------|--------|
| `src/api/v1/schemas/extraction_schemas.py` | 요청/응답 Pydantic 모델 | 완성도 100% |
| `src/api/v1/routers/extraction_schemas.py` | FastAPI 라우터 (5개 엔드포인트) | 완성도 100% |
| `src/api/v1/middleware/error_handler.py` | 전역 예외 처리 | 완성도 100% |
| `src/infrastructure/audit.py` | 감시 로그 ORM 및 AuditLogger | actor_type 필드 포함 |
| `tests/api/v1/test_extraction_schemas_api.py` | API 통합 테스트 (15개 이상 테스트 케이스) | 커버리지 ≥90% |
| `tests/api/v1/conftest.py` | pytest 픽스처 및 테스트 유틸리티 | 재사용 가능 |

---

## 6. 완료 기준

1. **5개 CRUD 엔드포인트 구현**
   - POST /api/v1/extraction-schemas (201 Created)
   - GET /api/v1/extraction-schemas/{doc_type} (200 OK)
   - PUT /api/v1/extraction-schemas/{doc_type} (200 OK, version 증가)
   - DELETE /api/v1/extraction-schemas/{doc_type} (204 No Content)
   - GET /api/v1/extraction-schemas/{doc_type}/versions (200 OK, 리스트)

2. **요청/응답 모델**
   - CreateExtractionSchemaRequest: doc_type_id, fields, scope_profile_id, metadata
   - UpdateExtractionSchemaRequest: fields, change_summary, scope_profile_id
   - ExtractionSchemaResponse: unwrapEnvelope 포맷
   - ExtractionSchemaVersionResponse: 버전, 변경사항, 필드 스냅샷

3. **입력 검증**
   - 필드 타입 일관성 검증 (string with enum 금지 등)
   - 정규식 패턴 검증
   - 필수 필드 검증

4. **Scope Profile ACL**
   - 모든 엔드포인트에서 ACL 필터링 적용
   - 권한 없으면 403 Forbidden

5. **actor_type 추적**
   - 요청 헤더에서 actor_type 추출 ("user" 또는 "agent")
   - audit_logs 테이블에 actor_type 기록

6. **에러 처리**
   - 400: 입력 검증 오류
   - 401: 인증 실패
   - 403: 권한 부족
   - 404: 리소스 없음
   - 409: 버전 충돌
   - 422: Unprocessable Entity
   - 500: 서버 오류
   - 모든 오류는 일관된 형식으로 응답

7. **감시 로그**
   - 모든 쓰기 작업(POST, PUT, DELETE)에 대해 audit_logs 기록
   - actor_id, actor_type, action, resource_type, resource_id, new_value 기록

8. **통합 테스트**
   - CRUD 엔드포인트별 테스트 (최소 15개 케이스)
   - 필드 검증 오류 시나리오 (최소 5개)
   - Scope Profile ACL 테스트
   - actor_type 추적 확인
   - 버전 이력 조회 테스트
   - 테스트 커버리지 ≥90%

9. **코드 품질**
   - Linting: flake8, black, isort 통과
   - Type hints: 100% (mypy --strict)
   - Docstrings: 모든 public 메서드

10. **OpenAPI 문서**
    - FastAPI의 자동 생성 문서 정상 작동
    - /docs, /redoc 엔드포인트 정상 제공

---

## 7. 작업 지침

**7-1**: **unwrapEnvelope 응답 포맷** — 모든 성공 응답은 `{ "data": {...}, "meta": {...} }` 형식. meta에는 request_id, timestamp 필수 포함. request_id는 요청의 X-Request-ID 헤더 또는 자동 생성 UUID 사용.

**7-2**: **actor_type 추출** — 요청 헤더:
  - Authorization: "Bearer {credentials}" (필수)
  - X-Actor-Type: "user" | "agent" (선택, 기본: credentials 형식에서 판별)
  - credentials 형식: "user_001" (user 타입) 또는 "agent_gpt4" (agent 타입)

**7-3**: **Scope Profile ACL 필터링** — Repository의 모든 조회/쓰기 메서드는 scope_profile_id 매개변수 받음. ScopeProfileService.check_access()를 호출하여 권한 검증. 권한 없으면 ForbiddenError 발생.

**7-4**: **버전 관리 정책**:
  - POST: version=1 자동 설정
  - PUT: 현재 버전 + 1로 자동 증가
  - 버전 이력은 ExtractionSchemaVersionORM에 모두 보존
  - change_summary와 changed_fields 필수 기록

**7-5**: **감시 로그 기록** — 모든 쓰기 작업(POST, PUT, DELETE)에 대해 즉시 audit_logs 테이블에 기록:
  ```sql
  INSERT INTO audit_logs (..., actor_type, ...) VALUES (..., 'user'|'agent', ...)
  ```

**7-6**: **에러 응답 포맷**:
  ```json
  {
    "error": {
      "code": "ERROR_CODE",
      "message": "설명",
      "details": {...} (optional)
    },
    "meta": {
      "request_id": "...",
      "timestamp": "..."
    }
  }
  ```

**7-7**: **입력 필드 검증** — CreateExtractionSchemaRequest와 UpdateExtractionSchemaRequest의 fields는 ExtractionFieldDefRequest 리스트. 각 필드는 도메인 모델의 ExtractionFieldDef로 변환되며, 이때 모든 검증 규칙 적용 (task8-1 참조).

**7-8**: **캐싱 고려** — Scope Profile 권한 확인은 비용이 높을 수 있으므로, 선택적으로 레디스 캐시 사용 고려 (TTL: 5분). 초기 구현은 캐시 없이 시작, task8-3에서 최적화.

**7-9**: **페이지네이션** — `/versions` 엔드포인트는 limit=10(기본), offset=0(기본) 쿼리 파라미터 지원. limit 최대값: 100.

**7-10**: **인증 실패 처리** — Authorization 헤더가 없거나 형식이 잘못되면 401 Unauthorized. 상세한 오류 메시지는 보안상 최소화 (e.g., "유효하지 않은 토큰").

---

**작업 예상 소요 시간**: 28-36시간  
**마지막 업데이트**: 2026-04-17
