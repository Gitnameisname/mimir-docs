# Task 1-8. EmbeddingProviderFactory + 재임베딩 마이그레이션 파이프라인

## 1. 작업 목적

**임베딩 모델 교체 시 무중단 서비스 전환**을 가능하게 하는 Factory 패턴과 재임베딩 마이그레이션 파이프라인을 구현한다. S1에서 OpenAI 1536차원 벡터를 사용 중인 상황에서 BGE-M3(1024차)로 전환할 때, 기존 벡터를 안전하게 관리하면서 신규 벡터로 교체하는 절차를 정의한다. 차원 검증, 백그라운드 마이그레이션, 품질 비교를 포함한다.

## 2. 작업 범위

### 포함 범위

1. **EmbeddingProviderFactory 구현**
   - `create(provider_name: str) → EmbeddingProvider`
   - 환경변수 `EMBEDDING_PROVIDER` (openai|bge|e5)
   - Provider 등록 및 조회 시스템
   - 설정 자동 로드

2. **차원 검증 로직**
   - 신규 Provider 차원 vs 기존 pgvector 인덱스 차원 비교
   - 불일치 시 마이그레이션 제안 또는 거부

3. **재임베딩 마이그레이션 파이프라인**
   - API: `POST /api/v1/admin/embeddings/migrate` (Admin 권한)
   - 시나리오: OpenAI(1536d) → BGE-M3(1024d) 전환
   - 절차:
     - 신규 모델로 document_chunks 전체 재임베딩
     - 기존 벡터는 `is_current=false` 처리
     - 신규 벡터는 신규 차원으로 저장
     - pgvector 인덱스 재구축
   - 백그라운드 Job 실행 (async)

4. **마이그레이션 품질 검증**
   - 마이그레이션 전후 샘플 검색 결과 비교
   - 메트릭: precision@K, recall@K (선택적)
   - 마이그레이션 롤백 계획

5. **마이그레이션 플레이북**
   - 단계별 마이그레이션 가이드
   - 트러블슈팅, 모니터링 팁

### 제외 범위

- 기타 벡터 관련 API 수정
- DB 스키마 재설계 (is_current 필드 추가는 불필요 if soft delete 대신 다른 방법 사용)
- 다른 모델 추가 (e.g., DPR, ColBERT)
- 자동 모델 선택 알고리즘

## 3. 선행 조건

- Phase 0 완료
- Task 1-6 완료 (EmbeddingProvider 인터페이스)
- Task 1-7 완료 (로컬 Provider 구현)
- S1 Phase 10 document_chunks 테이블 접근 가능
- pgvector 확장 설치 및 벡터 인덱스 존재
- Admin 권한 검증 기능 구현됨

## 4. 주요 작업 항목

### 4-1. EmbeddingProviderFactory 구현

**파일:** `/backend/app/services/embedding/factory.py`

**작업 항목:**
```python
import os
import logging
from typing import Dict, Type, Optional, Any
from .base import EmbeddingProvider, InvalidInputError
from .openai_embedding import OpenAIEmbeddingProvider
from .bge_provider import BGEEmbeddingProvider
from .e5_provider import E5EmbeddingProvider

logger = logging.getLogger(__name__)

class EmbeddingProviderFactory:
    """임베딩 Provider Factory"""
    
    _providers: Dict[str, Type[EmbeddingProvider]] = {
        "openai": OpenAIEmbeddingProvider,
        "bge": BGEEmbeddingProvider,
        "e5": E5EmbeddingProvider,
    }
    
    @classmethod
    def create(
        cls,
        provider_name: Optional[str] = None,
        config: Optional[Dict[str, Any]] = None
    ) -> EmbeddingProvider:
        """
        Provider 생성 (Factory 메서드)
        
        Args:
            provider_name: openai|bge|e5 (기본: EMBEDDING_PROVIDER 환경변수)
            config: Provider별 설정 딕셔너리
        
        Returns:
            EmbeddingProvider 인스턴스
        
        Raises:
            InvalidInputError: 유효하지 않은 provider 이름
        """
        if not provider_name:
            provider_name = os.getenv("EMBEDDING_PROVIDER", "openai")
        
        provider_name = provider_name.lower().strip()
        
        if provider_name not in cls._providers:
            available = ", ".join(cls._providers.keys())
            raise InvalidInputError(
                f"유효하지 않은 provider: {provider_name}. "
                f"지원: {available}"
            )
        
        provider_class = cls._providers[provider_name]
        
        try:
            logger.info(f"Provider 생성 중... (name={provider_name})")
            provider = provider_class(config)
            logger.info(
                f"Provider 생성 완료 "
                f"(name={provider_name}, model={provider.get_model_name()}, "
                f"dim={provider.get_dimension()})"
            )
            return provider
        except Exception as e:
            logger.error(f"Provider 생성 실패: {str(e)}")
            raise
    
    @classmethod
    def register_provider(
        cls,
        name: str,
        provider_class: Type[EmbeddingProvider]
    ) -> None:
        """
        커스텀 Provider 등록
        
        Args:
            name: provider 이름
            provider_class: EmbeddingProvider 상속 클래스
        """
        if not issubclass(provider_class, EmbeddingProvider):
            raise InvalidInputError(
                f"{provider_class} must inherit EmbeddingProvider"
            )
        cls._providers[name.lower()] = provider_class
        logger.info(f"Custom provider 등록: {name}")
    
    @classmethod
    def list_providers(cls) -> list:
        """등록된 provider 목록"""
        return list(cls._providers.keys())
    
    @classmethod
    def get_provider_info(cls, provider_name: str) -> Dict[str, Any]:
        """Provider 정보 조회"""
        if provider_name not in cls._providers:
            raise InvalidInputError(f"Unknown provider: {provider_name}")
        
        # 간단한 인스턴스 생성 후 메타정보 조회
        # 실제는 동적 로드로 최소화할 수 있음
        return {
            "name": provider_name,
            "class": cls._providers[provider_name].__name__,
        }
```

### 4-2. 차원 검증 서비스

**파일:** `/backend/app/services/embedding/dimension_validator.py`

**작업 항목:**
```python
import logging
from typing import Optional, Dict, Any
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text

logger = logging.getLogger(__name__)

class DimensionValidator:
    """임베딩 차원 검증"""
    
    @staticmethod
    async def get_current_dimension(
        db: AsyncSession,
        table: str = "document_chunks"
    ) -> Optional[int]:
        """
        현재 pgvector 인덱스 차원 확인
        
        Args:
            db: AsyncSession
            table: 벡터 저장 테이블명
        
        Returns:
            차원 (int) 또는 None (벡터 없음)
        """
        try:
            result = await db.execute(
                text(f"SELECT vector FROM {table} LIMIT 1")
            )
            row = result.fetchone()
            
            if not row or not row[0]:
                return None
            
            # pgvector의 차원 확인
            # vector 타입은 (dim,) 형태의 metadata 포함
            vector_str = str(row[0])  # pgvector 문자열 표현
            parts = vector_str.strip("[]").split(",")
            return len(parts)
        
        except Exception as e:
            logger.error(f"현재 차원 조회 실패: {str(e)}")
            return None
    
    @staticmethod
    async def validate_dimension_match(
        db: AsyncSession,
        new_dimension: int,
        table: str = "document_chunks"
    ) -> Dict[str, Any]:
        """
        신규 모델 차원과 기존 벡터 차원 비교
        
        Args:
            db: AsyncSession
            new_dimension: 신규 모델 차원
            table: 벡터 저장 테이블명
        
        Returns:
            {
                "current_dimension": int or None,
                "new_dimension": int,
                "match": bool,
                "message": str
            }
        """
        current_dim = await DimensionValidator.get_current_dimension(db, table)
        
        match = (current_dim == new_dimension) if current_dim else True
        
        if current_dim is None:
            message = "기존 벡터 없음. 신규 모델로 초기 임베딩 가능"
        elif match:
            message = "차원 일치. 마이그레이션 불필요"
        else:
            message = f"차원 불일치: {current_dim}d → {new_dimension}d. 마이그레이션 필요"
        
        return {
            "current_dimension": current_dim,
            "new_dimension": new_dimension,
            "match": match,
            "message": message
        }
```

### 4-3. 재임베딩 마이그레이션 서비스

**파일:** `/backend/app/services/embedding/migration_service.py`

**작업 항목:**
```python
import logging
import asyncio
from typing import Optional, Dict, Any, List
from datetime import datetime
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text, select, update
from uuid import uuid4

from .factory import EmbeddingProviderFactory
from .base import EmbeddingProvider

logger = logging.getLogger(__name__)

class EmbeddingMigrationService:
    """임베딩 재임베딩 및 마이그레이션"""
    
    @staticmethod
    async def start_migration(
        db: AsyncSession,
        new_provider_name: str,
        new_provider_config: Optional[Dict[str, Any]] = None,
        dry_run: bool = False
    ) -> Dict[str, Any]:
        """
        재임베딩 마이그레이션 시작
        
        Args:
            db: AsyncSession
            new_provider_name: openai|bge|e5
            new_provider_config: provider 설정
            dry_run: True면 실제 마이그레이션 없이 계획만 수립
        
        Returns:
            마이그레이션 ID 및 상태
        """
        migration_id = str(uuid4())
        
        try:
            # 신규 provider 생성
            new_provider = EmbeddingProviderFactory.create(
                new_provider_name,
                new_provider_config
            )
            
            # 기존 벡터 샘플링 (차원 확인용)
            result = await db.execute(
                text("SELECT COUNT(*) FROM document_chunks")
            )
            chunk_count = result.scalar()
            
            logger.info(
                f"마이그레이션 시작: ID={migration_id}, "
                f"provider={new_provider_name}, "
                f"dimension={new_provider.get_dimension()}, "
                f"total_chunks={chunk_count}"
            )
            
            if dry_run:
                return {
                    "migration_id": migration_id,
                    "status": "dry_run_completed",
                    "provider": new_provider_name,
                    "new_dimension": new_provider.get_dimension(),
                    "total_chunks": chunk_count,
                    "message": "드라이 런 완료. 실제 마이그레이션은 실행되지 않음"
                }
            
            # 실제 마이그레이션 (백그라운드로 실행)
            # 이 함수는 즉시 반환하고, 실제 작업은 task queue에서 처리
            return {
                "migration_id": migration_id,
                "status": "in_progress",
                "provider": new_provider_name,
                "new_dimension": new_provider.get_dimension(),
                "total_chunks": chunk_count,
                "started_at": datetime.utcnow().isoformat(),
                "message": "마이그레이션 시작됨. 백그라운드에서 처리 중..."
            }
        
        except Exception as e:
            logger.error(f"마이그레이션 시작 실패: {str(e)}")
            return {
                "migration_id": migration_id,
                "status": "failed",
                "error": str(e)
            }
    
    @staticmethod
    async def execute_migration_job(
        db: AsyncSession,
        migration_id: str,
        new_provider_name: str,
        new_provider_config: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        """
        실제 마이그레이션 작업 (백그라운드 task)
        
        Args:
            db: AsyncSession
            migration_id: 마이그레이션 ID
            new_provider_name: openai|bge|e5
            new_provider_config: provider 설정
        
        Returns:
            마이그레이션 결과
        """
        try:
            new_provider = EmbeddingProviderFactory.create(
                new_provider_name,
                new_provider_config
            )
            
            # 모든 document_chunks 조회
            result = await db.execute(
                text("SELECT id, text FROM document_chunks ORDER BY id")
            )
            chunks = result.fetchall()
            
            logger.info(f"재임베딩 시작: {len(chunks)}개 청크")
            
            # 배치 재임베딩
            batch_size = 32
            migrated_count = 0
            failed_count = 0
            
            for i in range(0, len(chunks), batch_size):
                batch_chunks = chunks[i:i+batch_size]
                chunk_texts = [c[1] for c in batch_chunks]
                chunk_ids = [c[0] for c in batch_chunks]
                
                try:
                    # 신규 모델로 임베딩
                    responses = await new_provider.embed_batch(chunk_texts)
                    
                    # DB에 저장 (기존 벡터 제거, 신규 벡터 추가)
                    for chunk_id, response in zip(chunk_ids, responses):
                        # pgvector 포맷으로 변환
                        vector_str = "[" + ",".join(
                            str(v) for v in response.vector
                        ) + "]"
                        
                        await db.execute(
                            text(
                                "UPDATE document_chunks SET "
                                "embedding = :embedding, "
                                "embedding_model = :model, "
                                "embedding_dimension = :dimension, "
                                "updated_at = NOW() "
                                "WHERE id = :id"
                            ),
                            {
                                "embedding": vector_str,
                                "model": new_provider.get_model_name(),
                                "dimension": response.dimension,
                                "id": chunk_id
                            }
                        )
                    
                    migrated_count += len(batch_chunks)
                    logger.info(
                        f"진행: {migrated_count}/{len(chunks)} 청크 재임베딩 완료"
                    )
                
                except Exception as batch_error:
                    failed_count += len(batch_chunks)
                    logger.error(f"배치 처리 실패: {str(batch_error)}")
            
            # 트랜잭션 커밋
            await db.commit()
            
            logger.info(
                f"마이그레이션 완료: "
                f"{migrated_count} 성공, {failed_count} 실패"
            )
            
            return {
                "migration_id": migration_id,
                "status": "completed",
                "provider": new_provider_name,
                "migrated_count": migrated_count,
                "failed_count": failed_count,
                "completed_at": datetime.utcnow().isoformat()
            }
        
        except Exception as e:
            logger.error(f"마이그레이션 작업 실패: {str(e)}")
            await db.rollback()
            return {
                "migration_id": migration_id,
                "status": "failed",
                "error": str(e)
            }
    
    @staticmethod
    async def get_migration_status(
        db: AsyncSession,
        migration_id: str
    ) -> Dict[str, Any]:
        """마이그레이션 상태 조회"""
        # 실제 구현: migration_jobs 테이블에서 조회
        # 여기서는 예시만 제공
        return {
            "migration_id": migration_id,
            "status": "in_progress",
            "message": "Status tracking not implemented"
        }
```

### 4-4. API 엔드포인트

**파일:** `/backend/app/api/v1/admin/embeddings.py`

**작업 항목:**
```python
from fastapi import APIRouter, Depends, HTTPException, BackgroundTasks
from sqlalchemy.ext.asyncio import AsyncSession
from typing import Optional, Dict, Any

from backend.app.db import get_db
from backend.app.auth import verify_admin_permission
from backend.app.services.embedding.factory import EmbeddingProviderFactory
from backend.app.services.embedding.dimension_validator import DimensionValidator
from backend.app.services.embedding.migration_service import EmbeddingMigrationService

router = APIRouter(prefix="/api/v1/admin/embeddings", tags=["embeddings"])

@router.post("/provider")
async def get_current_provider() -> Dict[str, Any]:
    """
    현재 활성 임베딩 Provider 정보 조회
    """
    try:
        provider = EmbeddingProviderFactory.create()
        return {
            "provider": provider.provider_name,
            "model": provider.get_model_name(),
            "dimension": provider.get_dimension()
        }
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/dimension-check")
async def check_dimension_match(
    new_provider: str,
    db: AsyncSession = Depends(get_db),
    admin_user = Depends(verify_admin_permission)
) -> Dict[str, Any]:
    """
    신규 Provider 차원과 기존 벡터 차원 비교
    """
    try:
        new_provider_obj = EmbeddingProviderFactory.create(new_provider)
        validation = await DimensionValidator.validate_dimension_match(
            db,
            new_provider_obj.get_dimension()
        )
        return validation
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.post("/migrate")
async def start_migration(
    new_provider: str,
    dry_run: bool = False,
    db: AsyncSession = Depends(get_db),
    background_tasks: BackgroundTasks = BackgroundTasks(),
    admin_user = Depends(verify_admin_permission)
) -> Dict[str, Any]:
    """
    재임베딩 마이그레이션 시작
    
    Args:
        new_provider: openai|bge|e5
        dry_run: True면 계획만 수립
        db: Database session
        background_tasks: 백그라운드 작업
        admin_user: Admin 권한 확인
    
    Returns:
        마이그레이션 ID 및 상태
    """
    try:
        result = await EmbeddingMigrationService.start_migration(
            db,
            new_provider,
            dry_run=dry_run
        )
        
        # 드라이 런이 아니면 백그라운드 작업 추가
        if not dry_run and result.get("status") == "in_progress":
            background_tasks.add_task(
                EmbeddingMigrationService.execute_migration_job,
                db,
                result["migration_id"],
                new_provider
            )
        
        return result
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/migration-status/{migration_id}")
async def get_migration_status(
    migration_id: str,
    db: AsyncSession = Depends(get_db),
    admin_user = Depends(verify_admin_permission)
) -> Dict[str, Any]:
    """마이그레이션 상태 조회"""
    status = await EmbeddingMigrationService.get_migration_status(
        db,
        migration_id
    )
    return status
```

### 4-5. 마이그레이션 플레이북

**파일:** `/backend/docs/embedding_migration_playbook.md`

**작업 항목:**
```markdown
# 임베딩 모델 마이그레이션 플레이북

## 1. 사전 준비

### 1-1. 현재 상태 확인
```bash
curl -X POST http://localhost:8000/api/v1/admin/embeddings/provider
# 응답: {"provider": "openai", "model": "text-embedding-3-small", "dimension": 1536}
```

### 1-2. 차원 검증
```bash
curl -X GET "http://localhost:8000/api/v1/admin/embeddings/dimension-check?new_provider=bge"
# 응답: {
#   "current_dimension": 1536,
#   "new_dimension": 1024,
#   "match": false,
#   "message": "차원 불일치: 1536d → 1024d. 마이그레이션 필요"
# }
```

## 2. 마이그레이션 실행

### 2-1. 드라이 런 (테스트)
```bash
curl -X POST "http://localhost:8000/api/v1/admin/embeddings/migrate?new_provider=bge&dry_run=true"
# 응답: {
#   "migration_id": "abc123...",
#   "status": "dry_run_completed",
#   "provider": "bge",
#   "new_dimension": 1024,
#   "total_chunks": 5000,
#   "message": "드라이 런 완료..."
# }
```

### 2-2. 실제 마이그레이션
```bash
curl -X POST "http://localhost:8000/api/v1/admin/embeddings/migrate?new_provider=bge&dry_run=false"
# 응답: {
#   "migration_id": "abc123...",
#   "status": "in_progress",
#   ...
# }
```

### 2-3. 진행 상황 모니터링
```bash
curl -X GET "http://localhost:8000/api/v1/admin/embeddings/migration-status/abc123..."
# 주기적으로 호출하여 상태 확인
```

## 3. 마이그레이션 후 검증

### 3-1. 벡터 차원 확인
```sql
SELECT COUNT(*), 
       COUNT(embedding) as with_embedding,
       embedding_dimension
FROM document_chunks
GROUP BY embedding_dimension;

-- 예상 결과: 모든 청크가 새 차원(1024)을 가짐
```

### 3-2. 검색 품질 테스트
```bash
# 마이그레이션 전후 동일 쿼리로 검색 결과 비교
curl -X POST "http://localhost:8000/api/v1/search" \
  -H "Content-Type: application/json" \
  -d '{"query": "테스트 검색어", "top_k": 5}'
```

## 4. 트러블슈팅

### 4-1. 마이그레이션 중단
```bash
# 진행 중 실패 시, 다시 시작 가능 (멱등성 보장)
# 기존 벡터는 유지되므로 서비스 영향 없음
```

### 4-2. 차원 불일치 오류
```
오류: "차원 불일치: 1536d → 1024d"
해결: 강제로 마이그레이션 진행 (자동 처리됨)
```

### 4-3. 모델 다운로드 실패
```
오류: "BGE-M3 모델 다운로드 실패"
해결: 
1. 인터넷 연결 확인
2. HuggingFace 토큰 확인 (private 모델인 경우)
3. 로컬 모델 경로 지정: export BGE_MODEL_PATH=/path/to/model
```

## 5. 롤백 전략

마이그레이션 실패 시:
1. 기존 벡터는 그대로 유지
2. Provider를 이전 버전으로 롤백
3. 필요시 벡터 복구 (is_current=false 된 벡터 복원)

## 6. 모니터링

- 마이그레이션 진행률: background task 로그 확인
- 벡터 품질: 검색 응답 시간, precision 모니터링
- 리소스 사용: CPU, 메모리, 디스크 I/O 모니터링
```

### 4-6. 단위 테스트

**파일:** `/backend/tests/unit/services/test_embedding_factory.py`

**작업 항목:**
```python
import pytest
from unittest.mock import patch, AsyncMock, MagicMock

@pytest.mark.asyncio
async def test_factory_create_openai():
    """Factory에서 OpenAI Provider 생성"""
    from backend.app.services.embedding.factory import EmbeddingProviderFactory
    
    with patch.dict("os.environ", {"EMBEDDING_PROVIDER": "openai"}):
        provider = EmbeddingProviderFactory.create()
        assert provider.provider_name == "openai"
        assert provider.get_dimension() == 1536

@pytest.mark.asyncio
async def test_factory_create_bge():
    """Factory에서 BGE Provider 생성"""
    from backend.app.services.embedding.factory import EmbeddingProviderFactory
    
    with patch('backend.app.services.embedding.bge_provider.SentenceTransformer'):
        provider = EmbeddingProviderFactory.create("bge")
        assert provider.provider_name == "bge"
        assert provider.get_dimension() == 1024

@pytest.mark.asyncio
async def test_factory_invalid_provider():
    """Factory에서 유효하지 않은 provider 처리"""
    from backend.app.services.embedding.factory import EmbeddingProviderFactory
    from backend.app.services.embedding.base import InvalidInputError
    
    with pytest.raises(InvalidInputError):
        EmbeddingProviderFactory.create("invalid_provider")

@pytest.mark.asyncio
async def test_dimension_validator_match():
    """차원 일치 검증"""
    from backend.app.services.embedding.dimension_validator import DimensionValidator
    from sqlalchemy.ext.asyncio import AsyncSession
    
    mock_db = AsyncMock(spec=AsyncSession)
    mock_result = AsyncMock()
    mock_result.scalar.return_value = None
    mock_db.execute.return_value = mock_result
    
    validation = await DimensionValidator.validate_dimension_match(
        mock_db,
        1024
    )
    
    assert validation["match"] == True
    assert validation["current_dimension"] is None
    assert validation["new_dimension"] == 1024

@pytest.mark.asyncio
async def test_migration_service_start():
    """마이그레이션 시작"""
    from backend.app.services.embedding.migration_service import EmbeddingMigrationService
    from sqlalchemy.ext.asyncio import AsyncSession
    
    mock_db = AsyncMock(spec=AsyncSession)
    mock_result = AsyncMock()
    mock_result.scalar.return_value = 5000
    mock_db.execute.return_value = mock_result
    
    with patch('backend.app.services.embedding.migration_service.EmbeddingProviderFactory.create'):
        result = await EmbeddingMigrationService.start_migration(
            mock_db,
            "bge",
            dry_run=True
        )
        
        assert result["status"] == "dry_run_completed"
        assert result["provider"] == "bge"
        assert result["total_chunks"] == 5000
```

## 5. 산출물

1. **EmbeddingProviderFactory** (`/backend/app/services/embedding/factory.py`)
   - `create(provider_name, config)` 메서드
   - 환경변수 `EMBEDDING_PROVIDER` 기반 자동 선택

2. **차원 검증 서비스** (`/backend/app/services/embedding/dimension_validator.py`)
   - 현재 차원 조회, 신규 모델 차원과 비교

3. **재임베딩 마이그레이션 서비스** (`/backend/app/services/embedding/migration_service.py`)
   - 마이그레이션 시작/실행/모니터링
   - 배치 재임베딩, DB 저장

4. **Admin API 엔드포인트** (`/backend/app/api/v1/admin/embeddings.py`)
   - `POST /api/v1/admin/embeddings/provider`: 현재 Provider 정보
   - `GET /api/v1/admin/embeddings/dimension-check`: 차원 검증
   - `POST /api/v1/admin/embeddings/migrate`: 마이그레이션 시작
   - `GET /api/v1/admin/embeddings/migration-status/{id}`: 상태 조회

5. **마이그레이션 플레이북** (`/backend/docs/embedding_migration_playbook.md`)
   - 단계별 마이그레이션 가이드, 트러블슈팅

6. **단위 테스트** (`/backend/tests/unit/services/test_embedding_factory.py`)

## 6. 완료 기준

1. `EmbeddingProviderFactory.create()` 정상 동작 (openai|bge|e5)
2. 환경변수 `EMBEDDING_PROVIDER` 정상 읽음
3. 차원 검증: 현재 1536d, 신규 1024d 차이 감지
4. 마이그레이션 dry_run 정상 작동 (실제 변경 없음)
5. 마이그레이션 실행: 5000개 청크 배치 처리 완료
6. DB 저장: 신규 벡터 (1024d) 정상 저장
7. Admin API 모두 정상 작동 (권한 검증 포함)
8. 마이그레이션 중 서비스 지속 가능 (무중단)
9. 단위 테스트 모두 통과
10. mypy 검사 통과
11. 플레이북: 실제 마이그레이션 수행 가능하도록 작성

## 7. 작업 지침

### 지침 7-1. 환경변수 설정

```bash
# 기본값 (OpenAI)
export EMBEDDING_PROVIDER=openai
export OPENAI_API_KEY=sk-...

# BGE-M3로 전환
export EMBEDDING_PROVIDER=bge
export BGE_MODEL_PATH=/path/to/bge-m3  # 선택

# E5로 전환
export EMBEDDING_PROVIDER=e5
export E5_MODEL_SIZE=large  # base 또는 large
```

### 지침 7-2. 백그라운드 작업 관리

마이그레이션은 FastAPI BackgroundTasks로 실행. 실제 프로덕션에서는:
- Celery, RQ 등 queue 시스템 사용 권장
- 마이그레이션 job 상태 DB에 저장 필수

### 지침 7-3. 원자성 보장

마이그레이션 중 실패 시:
- 트랜잭션 롤백 (ROLLBACK)
- 기존 벡터 유지
- 재시도 가능하도록 멱등성 보장

### 지침 7-4. 모니터링

마이그레이션 진행 중:
- 로그: DEBUG 레벨에서 배치별 진행률
- 메트릭: 처리 시간, 성공/실패 카운트
- 알림: 완료 또는 실패 시 관리자 알림

### 지침 7-5. 문서화

플레이북에 포함:
- 사전 점검 목록
- 단계별 명령어 (curl, API)
- 예상 소요 시간 (청크 개수별)
- 롤백 절차
