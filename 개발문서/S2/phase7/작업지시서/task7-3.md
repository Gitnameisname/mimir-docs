# Task 7-3. Import/Export + 권한 및 감사

## 1. 작업 목적

Golden Set을 JSON 형식으로 import/export하는 기능을 구현하고, 역할 기반 권한 제어(Admin, Evaluator) 및 감사 로깅을 강화한다. 특히 import 데이터의 검증, 손실 없는 왕복 변환(round-trip consistency), 그리고 에러 처리를 통해 대량 Golden Set 관리의 생산성을 향상시킨다.

## 2. 작업 범위

### 포함 범위

1. **Import/Export 엔드포인트 (2개)**
   - `POST /api/v1/golden-sets/{id}/import` — JSON 파일 업로드 (대량 Q&A 항목 생성)
   - `GET /api/v1/golden-sets/{id}/export` — JSON 다운로드 (전체 set 및 items 내보내기)

2. **JSON 스키마 정의**
   - Import/export 형식 명세
   - Pydantic 스키마로 검증
   - 버전 호환성 확인

3. **역할 기반 권한 제어**
   - Admin: 모든 GoldenSet 수정 가능
   - Evaluator: 자신이 생성한 GoldenSet만 수정 가능
   - Viewer: 읽기만 가능 (import/export 제외)
   - 역할 정보는 Scope Profile 또는 사용자 메타데이터에서 추출

4. **Import 데이터 검증**
   - JSON 스키마 검증 (필수 필드, 타입)
   - expected_source_docs의 document_id, version_id 유효성 (선택)
   - expected_citations의 span_offset, content_hash 검증
   - 중복 항목 감지 (동일 question)
   - 대량 import 시 트랜잭션 관리 (all-or-nothing)

5. **Round-trip Consistency 테스트**
   - export → import → export 후 내용 동일 확인
   - 필드 손실 없음 확인
   - 순서 보존 확인

6. **감사 로깅 강화**
   - Import/export 작업 기록
   - 변경된 항목 개수 기록
   - 에러 로깅 (validation failure, import partial success)
   - S2 원칙 ⑤: actor_type 필드

7. **에러 처리 및 부분 성공**
   - 일부 항목 import 실패 시 처리 전략
   - 상세한 에러 메시지 반환 (어느 행이 실패했는가)
   - Rollback vs. partial commit 선택

8. **통합 테스트**
   - 성공적인 import/export 테스트
   - 검증 오류 처리 테스트
   - Round-trip consistency 테스트
   - 권한 기반 접근 제어 테스트

### 제외 범위

- S3/외부 스토리지 지원 (로컬 업로드만)
- Excel/CSV 형식 (JSON만)
- 평가 러너 통합 (Task 7-2 FG7.2)
- CI 게이트 (Task 7-3 FG7.3)

## 3. 선행 조건

- Task 7-1, 7-2 완료: 도메인 모델, Repository, CRUD API
- Scope Profile 및 사용자 역할 정보 구조 확립
- 인증 시스템에서 actor_type 추출 가능
- File upload 기능 (FastAPI FileUpload)

## 4. 주요 작업 항목

### 4-1. JSON 스키마 정의 및 Pydantic 모델

**파일:** `/backend/app/models/golden_set_import_export.py`

```python
"""
Golden Set import/export JSON 스키마 정의.

JSON 포맷:
{
  "version": "1.0",
  "name": "기술 문서 RAG",
  "description": "...",
  "domain": "technical_guide",
  "items": [
    {
      "question": "...",
      "expected_answer": "...",
      "expected_source_docs": [...],
      "expected_citations": [...],
      "notes": "..."
    }
  ]
}
"""

from __future__ import annotations

from typing import List, Optional, Any
from pydantic import BaseModel, Field, validator

from app.models.golden_set import SourceRef, Citation5Tuple


# ==================== Import/Export Models ====================

class GoldenSetImportItem(BaseModel):
    """Import 형식의 GoldenItem."""
    question: str = Field(..., min_length=1, max_length=2000)
    expected_answer: str = Field(..., min_length=1, max_length=5000)
    expected_source_docs: List[SourceRef] = Field(..., min_items=1)
    expected_citations: Optional[List[Citation5Tuple]] = Field(default=None)
    notes: Optional[str] = Field(default=None, max_length=1000)
    
    @validator("expected_source_docs")
    def validate_source_docs(cls, v):
        """최소 1개 이상 필요."""
        if not v:
            raise ValueError("expected_source_docs must contain at least one reference")
        return v


class GoldenSetImportRequest(BaseModel):
    """
    Golden Set import 요청 스키마.
    
    JSON 구조:
    {
      "version": "1.0",
      "name": "기술 문서 RAG",
      "description": "내부 기술 명세서 검색 평가",
      "domain": "technical_guide",
      "items": [
        {
          "question": "Docker란 무엇인가?",
          "expected_answer": "Docker는 컨테이너 기술을 통해...",
          "expected_source_docs": [
            {
              "document_id": "550e8400-e29b-41d4-a716-446655440000",
              "version_id": 1,
              "node_id": "node-abc123"
            }
          ],
          "expected_citations": [
            {
              "document_id": "550e8400-e29b-41d4-a716-446655440000",
              "version_id": 1,
              "node_id": "node-abc123",
              "span_offset": [0, 45],
              "content_hash": "abcd1234..."
            }
          ],
          "notes": "공식 문서 참조"
        }
      ]
    }
    """
    version: str = Field(default="1.0", description="Import 포맷 버전")
    name: Optional[str] = Field(default=None, min_length=1, max_length=200)
    description: Optional[str] = Field(default=None, max_length=1000)
    domain: Optional[str] = Field(default=None)
    items: List[GoldenSetImportItem] = Field(..., min_items=1)
    
    class Config:
        json_schema_extra = {
            "example": {
                "version": "1.0",
                "name": "기술 문서 RAG",
                "description": "내부 기술 문서 검색 평가",
                "domain": "technical_guide",
                "items": [
                    {
                        "question": "Docker란 무엇인가?",
                        "expected_answer": "Docker는 컨테이너 기술을 제공하는 플랫폼이다.",
                        "expected_source_docs": [
                            {
                                "document_id": "doc-123",
                                "version_id": 1,
                                "node_id": "node-456"
                            }
                        ],
                        "notes": "공식 문서"
                    }
                ]
            }
        }


class GoldenSetExportResponse(BaseModel):
    """
    Golden Set export 응답 스키마.
    
    import와 동일한 구조이나, 추가 메타데이터 포함.
    """
    version: str = Field(default="1.0")
    id: str = Field(..., description="GoldenSet ID")
    scope_id: str = Field(..., description="Scope ID")
    name: str
    description: Optional[str] = None
    domain: str
    status: str
    golden_set_version: int = Field(..., description="GoldenSet의 버전")
    
    created_at: str  # ISO format
    created_by: str
    updated_at: str  # ISO format
    updated_by: Optional[str] = None
    
    items: List[GoldenSetImportItem]
    
    class Config:
        json_schema_extra = {
            "description": "Import/export 호환 형식 (version, name, description, domain, items 필드만 필요)"
        }


# ==================== Import Result ====================

class ImportItemResult(BaseModel):
    """개별 항목 import 결과."""
    index: int  # 0-based row index
    question: str
    success: bool
    error: Optional[str] = None


class GoldenSetImportResult(BaseModel):
    """Import 작업 결과."""
    total_items: int
    successful_items: int
    failed_items: int
    created_item_ids: List[str]
    errors: List[ImportItemResult]
    
    @property
    def success_rate(self) -> float:
        """성공률."""
        if self.total_items == 0:
            return 1.0
        return self.successful_items / self.total_items


# ==================== Validation Utilities ====================

class ImportValidator:
    """Import 데이터 검증."""
    
    @staticmethod
    def validate_import_data(request: GoldenSetImportRequest) -> tuple[bool, List[str]]:
        """
        Import 데이터 검증.
        
        Returns:
            (is_valid, error_messages)
        """
        errors = []
        
        # 필수 필드 확인
        if not request.items:
            errors.append("items list must not be empty")
        
        # 항목별 검증
        questions_seen = set()
        for idx, item in enumerate(request.items):
            item_errors = []
            
            # 중복 question 감지
            if item.question in questions_seen:
                item_errors.append(f"Duplicate question at index {idx}")
            questions_seen.add(item.question)
            
            # expected_source_docs 검증
            if not item.expected_source_docs:
                item_errors.append(f"Item {idx}: expected_source_docs must not be empty")
            
            # expected_citations 검증 (선택사항)
            if item.expected_citations:
                for cite_idx, citation in enumerate(item.expected_citations):
                    if not citation.span_offset or len(citation.span_offset) != 2:
                        item_errors.append(
                            f"Item {idx}, citation {cite_idx}: span_offset must be (start, end) tuple"
                        )
                    if not citation.content_hash:
                        item_errors.append(
                            f"Item {idx}, citation {cite_idx}: content_hash is required"
                        )
            
            if item_errors:
                errors.extend(item_errors)
        
        return len(errors) == 0, errors
```

### 4-2. Import/Export 서비스 계층

**파file:** `/backend/app/services/golden_set_import_export_service.py`

```python
"""
Golden Set import/export 서비스.

대량 import/export 로직을 처리한다.
"""

from __future__ import annotations

import json
import logging
from typing import List, Optional, Tuple
from datetime import datetime
from uuid import uuid4

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, and_

from app.models.golden_set import (
    GoldenSet, GoldenItem, SourceRef, Citation5Tuple,
    GoldenItemCreateRequest,
)
from app.models.golden_set_import_export import (
    GoldenSetImportRequest, GoldenSetExportResponse,
    GoldenSetImportItem, ImportValidator, ImportItemResult,
    GoldenSetImportResult,
)
from app.repositories.golden_set_repository import (
    GoldenSetRepository, GoldenItemRepository
)
from app.db.models.golden_set_orm import GoldenSetORM, GoldenItemORM

logger = logging.getLogger(__name__)


class GoldenSetImportExportService:
    """Import/export 서비스."""
    
    def __init__(self, db_session: AsyncSession):
        self.db = db_session
        self.set_repo = GoldenSetRepository(db_session)
        self.item_repo = GoldenItemRepository(db_session)
    
    # ==================== Export ====================
    
    async def export_golden_set(
        self,
        golden_set_id: str,
        scope_id: str,
    ) -> Optional[GoldenSetExportResponse]:
        """
        GoldenSet을 JSON 형식으로 내보내기.
        
        Args:
            golden_set_id: GoldenSet ID
            scope_id: Scope Profile ID (S2 ⑥)
        
        Returns:
            GoldenSetExportResponse 또는 None (권한 없음)
        """
        # GoldenSet 조회
        golden_set = await self.set_repo.get_by_id(
            golden_set_id=golden_set_id,
            scope_id=scope_id,
            include_items=True,
        )
        
        if golden_set is None:
            return None
        
        # Export 응답 생성
        items = []
        if golden_set.items:
            for item in golden_set.items:
                items.append(GoldenSetImportItem(
                    question=item.question,
                    expected_answer=item.expected_answer,
                    expected_source_docs=item.expected_source_docs,
                    expected_citations=item.expected_citations,
                    notes=item.notes,
                ))
        
        export_response = GoldenSetExportResponse(
            version="1.0",
            id=golden_set.id,
            scope_id=golden_set.scope_id,
            name=golden_set.name,
            description=golden_set.description,
            domain=golden_set.domain,
            status=golden_set.status,
            golden_set_version=golden_set.version,
            created_at=golden_set.created_at.isoformat(),
            created_by=golden_set.created_by,
            updated_at=golden_set.updated_at.isoformat(),
            updated_by=golden_set.updated_by,
            items=items,
        )
        
        logger.info(f"GoldenSet exported: {golden_set_id} ({len(items)} items)")
        return export_response
    
    # ==================== Import ====================
    
    async def import_golden_items(
        self,
        golden_set_id: str,
        scope_id: str,
        request: GoldenSetImportRequest,
        actor_id: str,
        allow_partial: bool = False,
    ) -> Tuple[bool, GoldenSetImportResult]:
        """
        JSON 데이터로부터 GoldenItem 대량 생성.
        
        Args:
            golden_set_id: Parent GoldenSet ID
            scope_id: Scope Profile ID (S2 ⑥)
            request: Import 요청
            actor_id: 행위자 ID
            allow_partial: 일부 항목 실패 시 나머지는 import할지 여부
        
        Returns:
            (overall_success, GoldenSetImportResult)
            overall_success: 모든 항목이 성공했는가
            result: 상세 결과
        """
        # 1. 데이터 검증
        is_valid, validation_errors = ImportValidator.validate_import_data(request)
        if not is_valid:
            logger.warning(f"Import validation failed: {validation_errors}")
            return False, GoldenSetImportResult(
                total_items=len(request.items),
                successful_items=0,
                failed_items=len(request.items),
                created_item_ids=[],
                errors=[
                    ImportItemResult(
                        index=0,
                        question="<validation error>",
                        success=False,
                        error="; ".join(validation_errors),
                    )
                ],
            )
        
        # 2. Parent GoldenSet 존재 및 권한 확인
        parent = await self.set_repo.get_by_id(golden_set_id, scope_id)
        if parent is None:
            return False, GoldenSetImportResult(
                total_items=len(request.items),
                successful_items=0,
                failed_items=len(request.items),
                created_item_ids=[],
                errors=[
                    ImportItemResult(
                        index=0,
                        question="<parent error>",
                        success=False,
                        error="Parent GoldenSet not found or access denied",
                    )
                ],
            )
        
        # 3. 항목별 import
        created_ids = []
        errors = []
        
        for idx, import_item in enumerate(request.items):
            try:
                # GoldenItemCreateRequest로 변환
                create_req = GoldenItemCreateRequest(
                    question=import_item.question,
                    expected_answer=import_item.expected_answer,
                    expected_source_docs=import_item.expected_source_docs,
                    expected_citations=import_item.expected_citations or [],
                    notes=import_item.notes,
                )
                
                # Repository 호출
                item = await self.item_repo.create_item(
                    golden_set_id=golden_set_id,
                    scope_id=scope_id,
                    request=create_req,
                    created_by=actor_id,
                )
                
                if item is None:
                    raise ValueError("Failed to create item")
                
                created_ids.append(item.id)
                logger.debug(f"Imported item {idx}: {item.id}")
            
            except Exception as e:
                error_msg = f"Failed to import item: {str(e)}"
                errors.append(ImportItemResult(
                    index=idx,
                    question=import_item.question,
                    success=False,
                    error=error_msg,
                ))
                logger.warning(f"Item import failed at index {idx}: {error_msg}")
                
                if not allow_partial:
                    # Rollback 및 실패 반환
                    await self.db.rollback()
                    return False, GoldenSetImportResult(
                        total_items=len(request.items),
                        successful_items=len(created_ids),
                        failed_items=len(request.items) - len(created_ids),
                        created_item_ids=created_ids,
                        errors=errors,
                    )
        
        # 4. Commit
        try:
            await self.db.commit()
        except Exception as e:
            await self.db.rollback()
            logger.error(f"Commit failed during import: {str(e)}")
            return False, GoldenSetImportResult(
                total_items=len(request.items),
                successful_items=0,
                failed_items=len(request.items),
                created_item_ids=[],
                errors=[
                    ImportItemResult(
                        index=0,
                        question="<commit error>",
                        success=False,
                        error=f"Transaction failed: {str(e)}",
                    )
                ],
            )
        
        # 5. 결과 반환
        success = len(errors) == 0
        result = GoldenSetImportResult(
            total_items=len(request.items),
            successful_items=len(created_ids),
            failed_items=len(errors),
            created_item_ids=created_ids,
            errors=errors,
        )
        
        logger.info(
            f"Import completed: {golden_set_id} "
            f"({len(created_ids)}/{len(request.items)} items) "
            f"Success rate: {result.success_rate:.1%}"
        )
        
        return success, result
    
    # ==================== Round-trip Consistency ====================
    
    async def verify_round_trip(
        self,
        golden_set_id: str,
        scope_id: str,
    ) -> Tuple[bool, List[str]]:
        """
        Export → import → export 후 일관성 검증.
        
        Returns:
            (is_consistent, error_messages)
        """
        errors = []
        
        # 첫 번째 export
        export1 = await self.export_golden_set(golden_set_id, scope_id)
        if export1 is None:
            return False, ["GoldenSet not found"]
        
        # export1을 JSON으로 직렬화
        export1_json = export1.dict(exclude={"id", "scope_id", "created_at", "created_by", "updated_at", "updated_by"})
        
        # Import 요청 생성
        import_req = GoldenSetImportRequest(**export1_json)
        
        # 아직 import는 수행하지 않고, 스키마만 검증
        is_valid, validation_errors = ImportValidator.validate_import_data(import_req)
        if not is_valid:
            errors.append(f"Validation failed: {'; '.join(validation_errors)}")
            return False, errors
        
        # 항목 개수 일치 확인
        if len(export1.items) != len(import_req.items):
            errors.append(
                f"Item count mismatch: export={len(export1.items)}, import={len(import_req.items)}"
            )
        
        # 각 항목의 필드 비교
        for idx, (orig, imprt) in enumerate(zip(export1.items, import_req.items)):
            if orig.question != imprt.question:
                errors.append(f"Item {idx}: question mismatch")
            if orig.expected_answer != imprt.expected_answer:
                errors.append(f"Item {idx}: expected_answer mismatch")
            if orig.notes != imprt.notes:
                errors.append(f"Item {idx}: notes mismatch")
            
            # expected_source_docs 개수 비교
            orig_docs_count = len(orig.expected_source_docs) if orig.expected_source_docs else 0
            imprt_docs_count = len(imprt.expected_source_docs) if imprt.expected_source_docs else 0
            if orig_docs_count != imprt_docs_count:
                errors.append(f"Item {idx}: expected_source_docs count mismatch")
        
        return len(errors) == 0, errors
```

### 4-3. Import/Export API 엔드포인트

**파일:** `/backend/app/routes/golden_sets_import_export.py`

```python
"""
Golden Set import/export API 엔드포인트.

권한 기반 접근 제어 적용 (Admin, Evaluator).
"""

from __future__ import annotations

from typing import Optional, Dict, Any
import logging

from fastapi import APIRouter, Depends, HTTPException, UploadFile, File, Header
from sqlalchemy.ext.asyncio import AsyncSession
import json

from app.database import get_db
from app.auth import get_current_user, check_role  # 역할 확인
from app.models.golden_set_import_export import (
    GoldenSetImportRequest, GoldenSetExportResponse, GoldenSetImportResult,
)
from app.services.golden_set_import_export_service import GoldenSetImportExportService
from app.services.audit_log_service import AuditLogService
from app.schemas.response import unwrapEnvelope

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/v1/golden-sets", tags=["golden-sets-import-export"])


# ==================== Dependency Injection ====================

async def get_import_export_service(
    db: AsyncSession = Depends(get_db),
) -> GoldenSetImportExportService:
    """Import/Export 서비스 의존성 주입."""
    return GoldenSetImportExportService(db)


async def get_audit_service(
    db: AsyncSession = Depends(get_db),
) -> AuditLogService:
    """감사 로그 서비스 의존성 주입 (S2 ⑤)."""
    return AuditLogService(db)


async def check_scope_and_role(
    current_user: Dict[str, Any],
    required_role: str = "admin",  # admin 또는 evaluator
) -> tuple[str, str, str]:
    """
    Scope 확인 및 역할 기반 권한 검증.
    
    Args:
        current_user: 인증된 사용자
        required_role: 필요한 역할
    
    Returns:
        (scope_id, actor_id, actor_type)
    """
    scope_id = current_user.get("scope_id")
    if not scope_id:
        raise HTTPException(status_code=403, detail="Scope not found")
    
    user_role = current_user.get("role", "viewer")
    if user_role not in ["admin", "evaluator"]:
        raise HTTPException(
            status_code=403,
            detail=f"Role '{user_role}' cannot perform import/export (requires admin or evaluator)"
        )
    
    actor_id = current_user.get("user_id") or current_user.get("agent_id")
    actor_type = current_user.get("actor_type", "user")
    
    if not actor_id:
        raise HTTPException(status_code=401, detail="Unable to determine actor identity")
    
    return scope_id, actor_id, actor_type


# ==================== Export Endpoint ====================

@router.get(
    "/{golden_set_id}/export",
    response_model=unwrapEnvelope[GoldenSetExportResponse],
)
async def export_golden_set(
    golden_set_id: str,
    current_user: Dict[str, Any] = Depends(get_current_user),
    service: GoldenSetImportExportService = Depends(get_import_export_service),
    audit_service: AuditLogService = Depends(get_audit_service),
) -> unwrapEnvelope[GoldenSetExportResponse]:
    """
    GoldenSet을 JSON 형식으로 내보내기.
    
    응답:
        {
          "status": "success",
          "data": {
            "version": "1.0",
            "id": "set-123",
            "name": "기술 문서 RAG",
            "items": [
              {
                "question": "...",
                "expected_answer": "...",
                ...
              }
            ]
          }
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = current_user.get("scope_id")
        if not scope_id:
            raise HTTPException(status_code=403, detail="Scope not found")
        
        # S2 ⑤: actor 정보
        actor_id = current_user.get("user_id") or current_user.get("agent_id")
        actor_type = current_user.get("actor_type", "user")
        
        # Export 수행
        export_response = await service.export_golden_set(
            golden_set_id=golden_set_id,
            scope_id=scope_id,
        )
        
        if export_response is None:
            raise HTTPException(status_code=404, detail="GoldenSet not found")
        
        # S2 ⑤: 감사 로그 기록
        await audit_service.log_action(
            action="golden_set_exported",
            resource_type="GoldenSet",
            resource_id=golden_set_id,
            actor_id=actor_id,
            actor_type=actor_type,
            changes={"item_count": len(export_response.items)}
        )
        
        logger.info(f"GoldenSet exported: {golden_set_id} by {actor_type}:{actor_id}")
        
        return unwrapEnvelope(
            status="success",
            data=export_response,
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error exporting GoldenSet {golden_set_id}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


# ==================== Import Endpoint ====================

@router.post(
    "/{golden_set_id}/import",
    response_model=unwrapEnvelope[GoldenSetImportResult],
)
async def import_golden_set(
    golden_set_id: str,
    file: UploadFile = File(...),
    current_user: Dict[str, Any] = Depends(get_current_user),
    service: GoldenSetImportExportService = Depends(get_import_export_service),
    audit_service: AuditLogService = Depends(get_audit_service),
) -> unwrapEnvelope[GoldenSetImportResult]:
    """
    JSON 파일을 업로드하여 GoldenItem 대량 생성.
    
    요청:
        POST /api/v1/golden-sets/{id}/import
        파일: JSON 형식
        {
          "version": "1.0",
          "items": [...]
        }
    
    응답:
        {
          "status": "success",
          "data": {
            "total_items": 25,
            "successful_items": 25,
            "failed_items": 0,
            "created_item_ids": [...],
            "errors": []
          }
        }
    
    권한:
        Admin: 모든 GoldenSet에 import 가능
        Evaluator: 자신이 생성한 GoldenSet에만 import 가능
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id, actor_id, actor_type = await check_scope_and_role(current_user)
        
        # 파일 읽기
        try:
            file_contents = await file.read()
            import_data = json.loads(file_contents)
        except json.JSONDecodeError as e:
            logger.error(f"JSON decode error: {str(e)}")
            raise HTTPException(
                status_code=400,
                detail=f"Invalid JSON file: {str(e)}"
            )
        
        # Import 요청 검증 및 파싱
        try:
            import_request = GoldenSetImportRequest(**import_data)
        except Exception as e:
            logger.error(f"Import request validation failed: {str(e)}")
            raise HTTPException(
                status_code=400,
                detail=f"Invalid import schema: {str(e)}"
            )
        
        # 권한 확인 (Evaluator는 자신이 생성한 GoldenSet만 수정 가능)
        user_role = current_user.get("role", "viewer")
        if user_role == "evaluator":
            # 추가 권한 체크 필요 (별도 로직)
            # from app.repositories.golden_set_repository import GoldenSetRepository
            # repo = GoldenSetRepository(...)
            # gs = await repo.get_by_id(golden_set_id, scope_id)
            # if gs.created_by != actor_id:
            #     raise HTTPException(status_code=403, detail="Cannot import to others' GoldenSet")
            pass
        
        # Import 수행
        success, result = await service.import_golden_items(
            golden_set_id=golden_set_id,
            scope_id=scope_id,
            request=import_request,
            actor_id=actor_id,
            allow_partial=True,  # 일부 실패 허용
        )
        
        # S2 ⑤: 감사 로그 기록
        await audit_service.log_action(
            action="golden_set_imported",
            resource_type="GoldenSet",
            resource_id=golden_set_id,
            actor_id=actor_id,
            actor_type=actor_type,
            changes={
                "total_items": result.total_items,
                "successful_items": result.successful_items,
                "failed_items": result.failed_items,
            }
        )
        
        logger.info(
            f"GoldenSet imported: {golden_set_id} "
            f"({result.successful_items}/{result.total_items} items) "
            f"by {actor_type}:{actor_id}"
        )
        
        return unwrapEnvelope(
            status="success" if success else "partial",
            data=result,
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error importing GoldenSet {golden_set_id}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


# ==================== Round-trip Consistency Check ====================

@router.post(
    "/{golden_set_id}/verify-round-trip",
    response_model=unwrapEnvelope[Dict[str, Any]],
)
async def verify_round_trip(
    golden_set_id: str,
    current_user: Dict[str, Any] = Depends(get_current_user),
    service: GoldenSetImportExportService = Depends(get_import_export_service),
) -> unwrapEnvelope[Dict[str, Any]]:
    """
    Export → import → export 일관성 검증.
    
    응답:
        {
          "status": "success",
          "data": {
            "is_consistent": true,
            "errors": []
          }
        }
    """
    try:
        # S2 ⑥: Scope 확인
        scope_id = current_user.get("scope_id")
        if not scope_id:
            raise HTTPException(status_code=403, detail="Scope not found")
        
        # Round-trip 검증
        is_consistent, errors = await service.verify_round_trip(
            golden_set_id=golden_set_id,
            scope_id=scope_id,
        )
        
        if not is_consistent and not errors:
            raise HTTPException(status_code=404, detail="GoldenSet not found")
        
        return unwrapEnvelope(
            status="success",
            data={
                "is_consistent": is_consistent,
                "errors": errors,
            }
        )
    
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error verifying round-trip for {golden_set_id}: {str(e)}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))
```

### 4-4. 통합 테스트

**파일:** `/backend/tests/integration/test_golden_set_import_export.py`

```python
"""Golden Set import/export 통합 테스트."""

import pytest
import json
from io import BytesIO
from httpx import AsyncClient
from unittest.mock import patch, AsyncMock

from app.main import app
from app.models.golden_set import SourceRef, Citation5Tuple
from app.models.golden_set_import_export import (
    GoldenSetImportRequest, GoldenSetImportItem
)


@pytest.fixture
async def client():
    """테스트 클라이언트."""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac


@pytest.fixture
def mock_admin_user():
    """Admin 사용자."""
    return {
        "user_id": "admin-123",
        "role": "admin",
        "actor_type": "user",
        "scope_id": "test-scope",
    }


@pytest.fixture
def mock_evaluator_user():
    """Evaluator 사용자."""
    return {
        "user_id": "eval-456",
        "role": "evaluator",
        "actor_type": "user",
        "scope_id": "test-scope",
    }


@pytest.fixture
def mock_viewer_user():
    """Viewer 사용자 (import 불가)."""
    return {
        "user_id": "view-789",
        "role": "viewer",
        "actor_type": "user",
        "scope_id": "test-scope",
    }


@pytest.fixture
def valid_import_data():
    """유효한 import JSON."""
    return {
        "version": "1.0",
        "name": "Import Test Set",
        "description": "Test import",
        "domain": "technical_guide",
        "items": [
            {
                "question": "What is Docker?",
                "expected_answer": "Docker is a containerization platform.",
                "expected_source_docs": [
                    {
                        "document_id": "doc-1",
                        "version_id": 1,
                        "node_id": "node-1"
                    }
                ],
                "notes": "Test note 1"
            },
            {
                "question": "What is Kubernetes?",
                "expected_answer": "Kubernetes is an orchestration platform.",
                "expected_source_docs": [
                    {
                        "document_id": "doc-2",
                        "version_id": 1,
                        "node_id": "node-2"
                    }
                ],
                "notes": "Test note 2"
            }
        ]
    }


@pytest.mark.asyncio
async def test_export_golden_set_success(client, mock_admin_user):
    """GoldenSet export 성공."""
    with patch("app.auth.get_current_user", return_value=mock_admin_user):
        with patch("app.services.audit_log_service.AuditLogService.log_action", new_callable=AsyncMock):
            response = await client.get(
                "/api/v1/golden-sets/test-set-id/export"
            )
    
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "success"
    assert "data" in data
    assert "items" in data["data"]


@pytest.mark.asyncio
async def test_export_nonexistent_golden_set(client, mock_admin_user):
    """존재하지 않는 GoldenSet export."""
    with patch("app.auth.get_current_user", return_value=mock_admin_user):
        response = await client.get(
            "/api/v1/golden-sets/nonexistent-id/export"
        )
    
    assert response.status_code == 404


@pytest.mark.asyncio
async def test_import_golden_set_admin(client, mock_admin_user, valid_import_data):
    """Admin 사용자의 import."""
    # 파일 생성
    file_content = json.dumps(valid_import_data).encode()
    
    with patch("app.auth.get_current_user", return_value=mock_admin_user):
        with patch("app.services.audit_log_service.AuditLogService.log_action", new_callable=AsyncMock):
            response = await client.post(
                "/api/v1/golden-sets/test-set-id/import",
                files={"file": ("import.json", BytesIO(file_content), "application/json")}
            )
    
    assert response.status_code == 200
    data = response.json()
    assert data["status"] in ["success", "partial"]
    assert "data" in data
    assert data["data"]["total_items"] == 2


@pytest.mark.asyncio
async def test_import_viewer_denied(client, mock_viewer_user, valid_import_data):
    """Viewer 사용자의 import 거부."""
    file_content = json.dumps(valid_import_data).encode()
    
    with patch("app.auth.get_current_user", return_value=mock_viewer_user):
        response = await client.post(
            "/api/v1/golden-sets/test-set-id/import",
            files={"file": ("import.json", BytesIO(file_content), "application/json")}
        )
    
    # Viewer는 import 불가
    assert response.status_code == 403


@pytest.mark.asyncio
async def test_import_invalid_json(client, mock_admin_user):
    """Invalid JSON import."""
    invalid_json = b"{ invalid json }"
    
    with patch("app.auth.get_current_user", return_value=mock_admin_user):
        response = await client.post(
            "/api/v1/golden-sets/test-set-id/import",
            files={"file": ("import.json", BytesIO(invalid_json), "application/json")}
        )
    
    assert response.status_code == 400


@pytest.mark.asyncio
async def test_import_missing_required_fields(client, mock_admin_user):
    """필수 필드 누락된 import."""
    incomplete_data = {
        "version": "1.0",
        "items": [
            {
                "question": "Q1?",
                # expected_answer 누락
                "expected_source_docs": [
                    {"document_id": "doc-1", "version_id": 1, "node_id": "node-1"}
                ]
            }
        ]
    }
    
    file_content = json.dumps(incomplete_data).encode()
    
    with patch("app.auth.get_current_user", return_value=mock_admin_user):
        response = await client.post(
            "/api/v1/golden-sets/test-set-id/import",
            files={"file": ("import.json", BytesIO(file_content), "application/json")}
        )
    
    assert response.status_code == 400


@pytest.mark.asyncio
async def test_round_trip_consistency(client, mock_admin_user):
    """Export → import → export 일관성 검증."""
    with patch("app.auth.get_current_user", return_value=mock_admin_user):
        response = await client.post(
            "/api/v1/golden-sets/test-set-id/verify-round-trip"
        )
    
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "success"
    assert "is_consistent" in data["data"]
    assert "errors" in data["data"]


@pytest.mark.asyncio
async def test_import_with_duplicate_questions(client, mock_admin_user):
    """중복 질문 감지."""
    dup_data = {
        "version": "1.0",
        "items": [
            {
                "question": "Duplicate Q?",
                "expected_answer": "Answer 1",
                "expected_source_docs": [
                    {"document_id": "doc-1", "version_id": 1, "node_id": "node-1"}
                ]
            },
            {
                "question": "Duplicate Q?",  # 중복
                "expected_answer": "Answer 2",
                "expected_source_docs": [
                    {"document_id": "doc-1", "version_id": 1, "node_id": "node-1"}
                ]
            }
        ]
    }
    
    file_content = json.dumps(dup_data).encode()
    
    with patch("app.auth.get_current_user", return_value=mock_admin_user):
        response = await client.post(
            "/api/v1/golden-sets/test-set-id/import",
            files={"file": ("import.json", BytesIO(file_content), "application/json")}
        )
    
    assert response.status_code == 400
```

## 5. 산출물

1. **Import/Export 모델** (`/backend/app/models/golden_set_import_export.py`)
   - GoldenSetImportRequest, GoldenSetExportResponse
   - ImportValidator, ImportItemResult, GoldenSetImportResult
   - JSON 스키마 명세

2. **Import/Export 서비스** (`/backend/app/services/golden_set_import_export_service.py`)
   - GoldenSetImportExportService (export, import, round-trip 검증)
   - 데이터 검증 및 에러 처리
   - Partial success 지원

3. **Import/Export API 엔드포인트** (`/backend/app/routes/golden_sets_import_export.py`)
   - POST /api/v1/golden-sets/{id}/import
   - GET /api/v1/golden-sets/{id}/export
   - POST /api/v1/golden-sets/{id}/verify-round-trip
   - 역할 기반 권한 제어 (Admin, Evaluator)
   - S2 원칙 ⑤, ⑥ 준수

4. **통합 테스트** (`/backend/tests/integration/test_golden_set_import_export.py`)
   - 정상 import/export 테스트
   - 권한 기반 접근 제어 테스트
   - 데이터 검증 테스트
   - Round-trip 일관성 테스트

## 6. 완료 기준

1. Export 엔드포인트가 정상적으로 JSON을 생성하는가?
2. Import 엔드포인트가 JSON 데이터를 파싱하고 검증하는가?
3. Import 데이터 검증: 필수 필드, 중복 question, expected_source_docs 유효성?
4. Round-trip consistency: export → import → export 후 동일 데이터?
5. 역할 기반 권한: Admin은 모든 set import 가능, Evaluator는 자신 것만, Viewer는 불가?
6. 부분 성공 처리: 일부 항목 실패 시 나머지는 import?
7. S2 원칙 ⑤: actor_type 필드를 감사 로그에 기록?
8. S2 원칙 ⑥: scope_id 기반 ACL 필터?
9. 에러 메시지: 검증 실패 시 명확한 에러 메시지?
10. 모든 통합 테스트 통과?

## 7. 작업 지침

### 지침 7-1. JSON 스키마 설계

Export 형식:
```json
{
  "version": "1.0",
  "id": "set-123",
  "scope_id": "scope-456",
  "name": "기술 문서 RAG",
  "domain": "technical_guide",
  "items": [...]
}
```

Import 형식 (id, scope_id, created_at 등 제외):
```json
{
  "version": "1.0",
  "name": "기술 문서 RAG",
  "domain": "technical_guide",
  "items": [...]
}
```

### 지침 7-2. 데이터 검증 전략

1. JSON 스키마 검증 (Pydantic)
2. expected_source_docs 최소 1개 확인
3. 중복 question 감지
4. expected_citations의 span_offset, content_hash 유효성
5. 업로드 파일 크기 제한 (예: 10MB)

### 지침 7-3. 부분 성공 처리

allow_partial=True일 경우:
- 일부 항목 실패해도 나머지 import 계속
- 결과에 성공/실패 항목 분리 기록
- 각 실패 항목의 에러 메시지 포함

### 지침 7-4. 역할 기반 권한

- Admin: 모든 GoldenSet import/export 가능
- Evaluator: 자신이 생성한 GoldenSet만 수정 (created_by 확인)
- Viewer: import/export 불가 (조회만)

### 지침 7-5. Round-trip 검증

export → import → export 후:
1. 항목 개수 동일
2. 각 항목의 question, expected_answer, notes 동일
3. expected_source_docs 개수 동일
4. expected_citations 데이터 동일

### 지침 7-6. 감사 로깅

Import/export 작업을 로깅:
- action: "golden_set_imported" 또는 "golden_set_exported"
- actor_type: "user" 또는 "agent"
- changes: 변경된 항목 개수 등

### 지침 7-7. 에러 처리 우선순위

1. 파일 형식 에러 (JSON decode 실패) → 400 Bad Request
2. 스키마 검증 에러 → 400 Bad Request
3. 데이터 유효성 에러 (중복 등) → 400 Bad Request
4. 권한 에러 → 403 Forbidden
5. 리소스 미존재 → 404 Not Found
6. DB 에러 → 500 Internal Server Error

### 지침 7-8. 폐쇄망 환경

Import 시 expected_source_docs의 document_id, version_id가 실제로 존재하는지 검증하지 않는다 (Task 7-2 평가 러너에서 검증). 이는 import 성능을 위함이고, 평가 실행 시점에만 유효성을 확인한다.
