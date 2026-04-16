# Task 8-7. 추출 재실행 배치 + 비동기 처리

## 1. 작업 목적

Task 8-4, 8-5의 자동 추출 및 사용자 승인 시스템이 완성된 후, **스키마 변경 시 기존 문서들의 추출을 재실행하고, 새 모델 배포 후 샘플 재추출로 성능을 검증**하는 배치 처리 시스템을 구축한다. 장시간 실행되는 배치 작업을 비동기로 처리하고, 진행률을 실시간으로 추적하며, 부분 실패 시 재시도 메커니즘을 제공한다. Scope Profile 기반 ACL과 감시 로깅(S2 원칙)을 준수한다.

## 2. 작업 범위

### 포함 범위

1. **배치 재실행 API**
   - `POST /api/v1/extractions/batch-retry` — 특정 DocumentType의 모든 승인된 문서 재추출
     * 쿼리 파라미터: extraction_schema_id, date_from, date_to, sample_count (optional)
     * 응답: batch_job_id (UUID)
   - `GET /api/v1/extractions/batch-jobs/{job_id}` — 배치 작업 상태 조회 (실시간 진행률)
   - `POST /api/v1/extractions/batch-jobs/{job_id}/cancel` — 실행 중인 배치 취소

2. **BatchExtractionJob 도메인 모델 및 ORM**
   - BatchExtractionJob: job_id, extraction_schema_id, total_count, completed_count, failed_count, status (pending/running/completed/failed/cancelled)
   - 진행률 추적: progress_percentage, current_processing
   - 메타데이터: started_at, completed_at, error_summary
   - Scope Profile 기반 ACL: scope_id

3. **백그라운드 워커 구현**
   - Celery task 또는 asyncio worker로 배치 처리
   - 청크 단위 처리: 한 번에 N개의 문서씩 추출 (N=10)
   - 지수 백오프 재시도: LLM 호출 실패 시 3회 재시도
   - Rate limiting: LLM API 호출 속도 제한 (초당 최대 5개)
   - 진행 상황 업데이트: 매 청크 완료마다 DB 업데이트

4. **비교 모드 (Comparison Mode)**
   - 재추출 결과를 이전 ApprovedExtraction과 비교
   - 변경사항 감지: new/updated/deleted 필드
   - 신뢰도 변화: 이전 vs 새 extraction의 confidence 비교
   - CSV 리포트 생성: comparison_report.csv

5. **샘플 재추출 (Sample Re-extraction)**
   - 새 모델 배포 후 무작위 N개 샘플로 재추출
   - API: `POST /api/v1/extractions/sample-retry` — 무작위 샘플 선택 및 재추출
   - 성능 평가: 신뢰도, 타입 정확도, 필수 필드 coverage

6. **Alembic 마이그레이션**
   - batch_extraction_jobs 테이블 생성
   - extraction_retry_logs 테이블 (재시도 이력)
   - 인덱스: (scope_id, status), (created_at, status)

7. **배치 진행률 SSE (Server-Sent Events)**
   - `/api/v1/extractions/batch-jobs/{job_id}/progress` — SSE endpoint
   - 실시간 업데이트: 진행률, 완료 수, 실패 수, 예상 소요 시간

8. **에러 처리 및 부분 실패**
   - 개별 문서 추출 실패 시 다음 문서로 계속 진행
   - 재시도 로직: 지수 백오프 (1s, 2s, 4s)
   - 반복되는 실패: 재시도 횟수 초과 후 skip and log

9. **단위 및 통합 테스트**
   - BatchExtractionJob 생성 및 상태 전이 테스트
   - 배치 처리 흐름 테스트 (mock LLM)
   - 부분 실패 시나리오
   - SSE 진행률 업데이트 검증

### 제외 범위

- 추출 파이프라인 (Task 8-4)
- 추출 결과 검토 API (Task 8-5)
- 검토 UI (Task 8-6)
- 평가 로직 (FG8.3)

## 3. 선행 조건

- Task 8-4, 8-5 완료: ExtractionCandidate, ApprovedExtraction 모델 및 API
- Phase 1 LLM Provider 추상화 완료
- Celery 또는 Python asyncio 설정
- PostgreSQL + SQLAlchemy + Alembic 설정

## 4. 주요 작업 항목

### 4-1. BatchExtractionJob 도메인 모델

**파일:** `/backend/app/models/batch_extraction.py`

```python
"""
배치 재추출 작업 도메인 모델.
"""

from __future__ import annotations

from datetime import datetime
from enum import Enum
from typing import Any, Dict, List, Optional
from uuid import UUID

from pydantic import BaseModel, Field, validator


class BatchJobStatus(str, Enum):
    """배치 작업 상태."""
    PENDING = "pending"        # 대기 중
    RUNNING = "running"        # 실행 중
    COMPLETED = "completed"    # 완료
    FAILED = "failed"          # 실패
    CANCELLED = "cancelled"    # 취소됨


class BatchExtractionJob(BaseModel):
    """배치 재추출 작업."""
    
    id: UUID = Field(default_factory=lambda: UUID('00000000-0000-0000-0000-000000000000'))
    
    # ==================== Job Info ====================
    extraction_schema_id: str  # DocumentType ID
    extraction_schema_version: int
    scope_id: UUID
    
    # ==================== Progress ====================
    status: BatchJobStatus = BatchJobStatus.PENDING
    total_count: int  # 처리할 총 문서 수
    completed_count: int = 0
    failed_count: int = 0
    skipped_count: int = 0
    progress_percentage: float = 0.0
    
    # ==================== Filtering ====================
    date_from: Optional[datetime] = None  # 문서 생성일 범위
    date_to: Optional[datetime] = None
    sample_count: Optional[int] = None  # None = 전체, N = 무작위 N개
    sample_mode: bool = False  # True이면 무작위 샘플
    
    # ==================== Comparison ====================
    comparison_mode: bool = False  # 이전 결과와 비교
    comparison_report_url: Optional[str] = None  # S3 or local path
    
    # ==================== Timing ====================
    created_at: datetime
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    estimated_completion_time: Optional[datetime] = None
    
    # ==================== Current Processing ====================
    current_processing: Optional[int] = None  # 현재 처리 중인 문서 수
    
    # ==================== Error Info ====================
    error_summary: Optional[str] = None  # 최종 에러 메시지
    failed_documents: List[str] = Field(default_factory=list)  # 실패한 document_id 목록
    
    # ==================== Metadata ====================
    created_by: UUID
    is_cancellation_requested: bool = False
    actor_type: str = "user"
    
    class Config:
        use_enum_values = False
        json_encoders = {
            datetime: lambda v: v.isoformat(),
            UUID: lambda v: str(v),
        }
    
    @validator('completed_count', 'failed_count', 'skipped_count')
    def validate_non_negative(cls, v):
        if v < 0:
            raise ValueError("Count must be non-negative")
        return v
    
    @validator('progress_percentage')
    def validate_progress(cls, v):
        if not (0.0 <= v <= 100.0):
            raise ValueError("Progress must be between 0.0 and 100.0")
        return v


class BatchExtractionJobResponse(BaseModel):
    """조회 응답."""
    id: UUID
    extraction_schema_id: str
    extraction_schema_version: int
    status: BatchJobStatus
    total_count: int
    completed_count: int
    failed_count: int
    skipped_count: int
    progress_percentage: float
    started_at: Optional[datetime]
    completed_at: Optional[datetime]
    estimated_completion_time: Optional[datetime]
    created_at: datetime


# ==================== Request DTOs ====================

class BatchRetryRequest(BaseModel):
    """배치 재추출 요청."""
    extraction_schema_id: str
    extraction_schema_version: int
    date_from: Optional[datetime] = None
    date_to: Optional[datetime] = None
    sample_count: Optional[int] = None  # None = 전체, N = 무작위 샘플
    comparison_mode: bool = False  # 이전 결과와 비교
    created_by: UUID


class SampleRetryRequest(BaseModel):
    """샘플 재추출 요청."""
    extraction_schema_id: str
    extraction_schema_version: int
    sample_count: int = Field(default=10, ge=1, le=100)
    created_by: UUID


class CancelBatchRequest(BaseModel):
    """배치 취소 요청."""
    reason: Optional[str] = None


class ExtractionRetryLog(BaseModel):
    """재시도 이력 기록."""
    job_id: UUID
    document_id: UUID
    attempt_number: int
    status: str  # success, failed, skipped
    error_reason: Optional[str] = None
    latency_ms: int
    created_at: datetime
```

### 4-2. BatchExtractionJob ORM 및 마이그레이션

**파일:** `/backend/app/db/models/batch_extraction.py`

```python
"""
BatchExtractionJob SQLAlchemy ORM.
"""

from datetime import datetime
from typing import Optional

from sqlalchemy import (
    Boolean, Column, DateTime, Enum, Float, ForeignKey,
    Integer, JSON, String, Text, UniqueConstraint, Index
)
from sqlalchemy.dialects.postgresql import UUID as PG_UUID

from app.db.base import Base
from app.models.batch_extraction import BatchJobStatus


class BatchExtractionJobORM(Base):
    """배치 추출 작업 ORM."""
    
    __tablename__ = "batch_extraction_jobs"
    
    id = Column(
        PG_UUID(as_uuid=True),
        primary_key=True,
        nullable=False,
        index=True
    )
    
    extraction_schema_id = Column(
        String(255),
        nullable=False
    )
    extraction_schema_version = Column(
        Integer,
        nullable=False
    )
    
    scope_id = Column(
        PG_UUID(as_uuid=True),
        nullable=False,
        index=True
    )
    
    status = Column(
        Enum(BatchJobStatus),
        nullable=False,
        default=BatchJobStatus.PENDING,
        index=True
    )
    
    total_count = Column(Integer, nullable=False)
    completed_count = Column(Integer, nullable=False, default=0)
    failed_count = Column(Integer, nullable=False, default=0)
    skipped_count = Column(Integer, nullable=False, default=0)
    progress_percentage = Column(Float, nullable=False, default=0.0)
    
    date_from = Column(DateTime(timezone=True), nullable=True)
    date_to = Column(DateTime(timezone=True), nullable=True)
    sample_count = Column(Integer, nullable=True)
    sample_mode = Column(Boolean, nullable=False, default=False)
    
    comparison_mode = Column(Boolean, nullable=False, default=False)
    comparison_report_url = Column(String(1024), nullable=True)
    
    created_at = Column(
        DateTime(timezone=True),
        nullable=False,
        default=datetime.utcnow,
        index=True
    )
    started_at = Column(DateTime(timezone=True), nullable=True)
    completed_at = Column(DateTime(timezone=True), nullable=True)
    estimated_completion_time = Column(DateTime(timezone=True), nullable=True)
    
    current_processing = Column(Integer, nullable=True)
    
    error_summary = Column(Text, nullable=True)
    failed_documents = Column(JSON, nullable=False, default=[])
    
    created_by = Column(PG_UUID(as_uuid=True), nullable=False, index=True)
    is_cancellation_requested = Column(Boolean, nullable=False, default=False)
    actor_type = Column(String(32), nullable=False, default="user")
    
    __table_args__ = (
        Index('idx_batch_job_scope_status', 'scope_id', 'status'),
        Index('idx_batch_job_created_at', 'created_at', 'status'),
    )


class ExtractionRetryLogORM(Base):
    """재시도 이력 로그."""
    
    __tablename__ = "extraction_retry_logs"
    
    id = Column(
        PG_UUID(as_uuid=True),
        primary_key=True,
        nullable=False,
        index=True
    )
    
    job_id = Column(
        PG_UUID(as_uuid=True),
        ForeignKey('batch_extraction_jobs.id'),
        nullable=False,
        index=True
    )
    
    document_id = Column(
        PG_UUID(as_uuid=True),
        nullable=False,
        index=True
    )
    
    attempt_number = Column(Integer, nullable=False)
    status = Column(String(32), nullable=False)  # success, failed, skipped
    error_reason = Column(Text, nullable=True)
    latency_ms = Column(Integer, nullable=False)
    
    created_at = Column(
        DateTime(timezone=True),
        nullable=False,
        default=datetime.utcnow,
        index=True
    )
    
    __table_args__ = (
        Index('idx_retry_log_job_id', 'job_id', 'document_id'),
    )
```

### 4-3. BatchExtractionJobRepository

**파일:** `/backend/app/repositories/batch_extraction_repository.py`

```python
"""
BatchExtractionJobRepository.
"""

from datetime import datetime, timedelta
from typing import Optional, List
from uuid import UUID

from sqlalchemy.orm import Session
from sqlalchemy import and_, desc

from app.db.models.batch_extraction import BatchExtractionJobORM, ExtractionRetryLogORM
from app.models.batch_extraction import BatchExtractionJob, BatchJobStatus
from app.core.scope import ScopeProfile, enforce_scope_acl


class BatchExtractionJobRepository:
    """배치 작업 저장소."""
    
    def __init__(self, db: Session, scope_profile: ScopeProfile):
        self.db = db
        self.scope_profile = scope_profile
    
    async def create(
        self,
        job: BatchExtractionJob
    ) -> BatchExtractionJob:
        """새 배치 작업 생성."""
        enforce_scope_acl(
            self.scope_profile,
            job.scope_id,
            action="create_batch_job"
        )
        
        orm = BatchExtractionJobORM(
            id=job.id,
            extraction_schema_id=job.extraction_schema_id,
            extraction_schema_version=job.extraction_schema_version,
            scope_id=job.scope_id,
            status=job.status,
            total_count=job.total_count,
            completed_count=job.completed_count,
            failed_count=job.failed_count,
            skipped_count=job.skipped_count,
            progress_percentage=job.progress_percentage,
            date_from=job.date_from,
            date_to=job.date_to,
            sample_count=job.sample_count,
            sample_mode=job.sample_mode,
            comparison_mode=job.comparison_mode,
            created_at=datetime.utcnow(),
            created_by=job.created_by,
            actor_type=job.actor_type
        )
        
        self.db.add(orm)
        self.db.flush()
        
        return self._to_model(orm)
    
    async def get_by_id(
        self,
        job_id: UUID,
        scope_id: UUID
    ) -> Optional[BatchExtractionJob]:
        """ID로 조회."""
        enforce_scope_acl(self.scope_profile, scope_id, action="read_batch_job")
        
        orm = self.db.query(BatchExtractionJobORM).filter(
            and_(
                BatchExtractionJobORM.id == job_id,
                BatchExtractionJobORM.scope_id == scope_id
            )
        ).first()
        
        return self._to_model(orm) if orm else None
    
    async def update_progress(
        self,
        job_id: UUID,
        scope_id: UUID,
        completed: int,
        failed: int,
        skipped: int
    ) -> Optional[BatchExtractionJob]:
        """진행률 업데이트."""
        enforce_scope_acl(self.scope_profile, scope_id, action="update_batch_job")
        
        orm = self.db.query(BatchExtractionJobORM).filter(
            and_(
                BatchExtractionJobORM.id == job_id,
                BatchExtractionJobORM.scope_id == scope_id
            )
        ).first()
        
        if not orm:
            return None
        
        orm.completed_count = completed
        orm.failed_count = failed
        orm.skipped_count = skipped
        
        # 진행률 계산
        processed = completed + failed + skipped
        if orm.total_count > 0:
            orm.progress_percentage = (processed / orm.total_count) * 100.0
        
        # 예상 완료 시간 계산
        if orm.started_at and completed > 0:
            elapsed = (datetime.utcnow() - orm.started_at).total_seconds()
            avg_time_per_item = elapsed / completed
            remaining_items = orm.total_count - processed
            estimated_remaining = avg_time_per_item * remaining_items
            orm.estimated_completion_time = (
                datetime.utcnow() + timedelta(seconds=estimated_remaining)
            )
        
        self.db.flush()
        return self._to_model(orm)
    
    async def update_status(
        self,
        job_id: UUID,
        scope_id: UUID,
        new_status: BatchJobStatus,
        error_summary: Optional[str] = None
    ) -> Optional[BatchExtractionJob]:
        """상태 업데이트."""
        enforce_scope_acl(self.scope_profile, scope_id, action="update_batch_job")
        
        orm = self.db.query(BatchExtractionJobORM).filter(
            and_(
                BatchExtractionJobORM.id == job_id,
                BatchExtractionJobORM.scope_id == scope_id
            )
        ).first()
        
        if not orm:
            return None
        
        orm.status = new_status
        
        if new_status == BatchJobStatus.RUNNING and not orm.started_at:
            orm.started_at = datetime.utcnow()
        
        if new_status in (BatchJobStatus.COMPLETED, BatchJobStatus.FAILED, BatchJobStatus.CANCELLED):
            orm.completed_at = datetime.utcnow()
        
        if error_summary:
            orm.error_summary = error_summary
        
        self.db.flush()
        return self._to_model(orm)
    
    async def add_failed_document(
        self,
        job_id: UUID,
        scope_id: UUID,
        document_id: UUID
    ) -> bool:
        """실패한 문서 목록에 추가."""
        enforce_scope_acl(self.scope_profile, scope_id, action="update_batch_job")
        
        orm = self.db.query(BatchExtractionJobORM).filter(
            and_(
                BatchExtractionJobORM.id == job_id,
                BatchExtractionJobORM.scope_id == scope_id
            )
        ).first()
        
        if not orm:
            return False
        
        doc_id_str = str(document_id)
        if doc_id_str not in orm.failed_documents:
            orm.failed_documents.append(doc_id_str)
        
        self.db.flush()
        return True
    
    async def request_cancellation(
        self,
        job_id: UUID,
        scope_id: UUID
    ) -> bool:
        """취소 요청."""
        enforce_scope_acl(self.scope_profile, scope_id, action="update_batch_job")
        
        orm = self.db.query(BatchExtractionJobORM).filter(
            and_(
                BatchExtractionJobORM.id == job_id,
                BatchExtractionJobORM.scope_id == scope_id
            )
        ).first()
        
        if not orm or orm.status not in (BatchJobStatus.PENDING, BatchJobStatus.RUNNING):
            return False
        
        orm.is_cancellation_requested = True
        self.db.flush()
        return True
    
    async def log_retry(
        self,
        job_id: UUID,
        document_id: UUID,
        attempt: int,
        status: str,
        error_reason: Optional[str],
        latency_ms: int
    ) -> None:
        """재시도 이력 기록."""
        log = ExtractionRetryLogORM(
            id=UUID('00000000-0000-0000-0000-000000000000'),
            job_id=job_id,
            document_id=document_id,
            attempt_number=attempt,
            status=status,
            error_reason=error_reason,
            latency_ms=latency_ms,
            created_at=datetime.utcnow()
        )
        
        self.db.add(log)
        self.db.flush()
    
    def _to_model(self, orm: BatchExtractionJobORM) -> BatchExtractionJob:
        """ORM → Pydantic 변환."""
        return BatchExtractionJob(
            id=orm.id,
            extraction_schema_id=orm.extraction_schema_id,
            extraction_schema_version=orm.extraction_schema_version,
            scope_id=orm.scope_id,
            status=orm.status,
            total_count=orm.total_count,
            completed_count=orm.completed_count,
            failed_count=orm.failed_count,
            skipped_count=orm.skipped_count,
            progress_percentage=orm.progress_percentage,
            date_from=orm.date_from,
            date_to=orm.date_to,
            sample_count=orm.sample_count,
            sample_mode=orm.sample_mode,
            comparison_mode=orm.comparison_mode,
            comparison_report_url=orm.comparison_report_url,
            created_at=orm.created_at,
            started_at=orm.started_at,
            completed_at=orm.completed_at,
            estimated_completion_time=orm.estimated_completion_time,
            current_processing=orm.current_processing,
            error_summary=orm.error_summary,
            failed_documents=orm.failed_documents,
            created_by=orm.created_by,
            is_cancellation_requested=orm.is_cancellation_requested,
            actor_type=orm.actor_type
        )
```

### 4-4. 배치 워커 서비스

**파일:** `/backend/app/services/extraction/batch_extraction_service.py`

```python
"""
배치 재추출 워커 서비스.

Celery task 또는 asyncio로 구현:
- 청크 단위 처리 (N=10)
- Rate limiting (초당 최대 5개)
- 지수 백오프 재시도 (3회)
- 부분 실패 시에도 계속 처리
"""

import asyncio
import logging
import math
from datetime import datetime
from typing import Optional, List
from uuid import UUID

from sqlalchemy.orm import Session

from app.db.database import SessionLocal
from app.db.models.extraction import ExtractionCandidateORM, ApprovedExtractionORM
from app.models.batch_extraction import BatchJobStatus
from app.repositories.batch_extraction_repository import BatchExtractionJobRepository
from app.repositories.extraction_candidate_repository import ExtractionCandidateRepository
from app.repositories.approved_extraction_repository import ApprovedExtractionRepository
from app.services.extraction.extraction_pipeline_service import ExtractionPipelineService
from app.core.scope import ScopeProfile
from app.core.audit import AuditLogger


logger = logging.getLogger(__name__)


class BatchExtractionService:
    """배치 추출 처리 서비스."""
    
    # 설정값
    CHUNK_SIZE = 10  # 한 번에 처리할 문서 수
    RATE_LIMIT_PER_SEC = 5  # 초당 최대 LLM 호출 수
    MAX_RETRIES = 3  # 최대 재시도 횟수
    RETRY_DELAYS = [1, 2, 4]  # exponential backoff: 1s, 2s, 4s
    
    def __init__(
        self,
        extraction_pipeline: ExtractionPipelineService,
        audit_logger: AuditLogger
    ):
        self.extraction_pipeline = extraction_pipeline
        self.audit_logger = audit_logger
    
    async def execute_batch_retry(
        self,
        job_id: UUID,
        extraction_schema_id: str,
        extraction_schema_version: int,
        scope_id: UUID,
        date_from: Optional[datetime] = None,
        date_to: Optional[datetime] = None,
        sample_count: Optional[int] = None,
        comparison_mode: bool = False
    ) -> None:
        """
        배치 재추출 실행 (비동기 백그라운드 작업).
        
        처리 흐름:
        1. 대상 문서 목록 조회
        2. 샘플링 (필요시)
        3. 청크 단위 처리
        4. 각 문서 재추출
        5. 진행률 업데이트
        6. 부분 실패 처리
        """
        db = SessionLocal()
        
        try:
            # 1. Repository 초기화
            batch_repo = BatchExtractionJobRepository(db, None)
            
            # 2. 배치 작업 상태 → RUNNING으로 변경
            job = await batch_repo.update_status(
                job_id, scope_id, BatchJobStatus.RUNNING
            )
            if not job:
                logger.error(f"Job {job_id} not found")
                return
            
            logger.info(f"Starting batch job {job_id} for schema {extraction_schema_id}")
            
            # 3. 감시 로그
            await self.audit_logger.log(
                action="batch_extraction_started",
                actor_type="agent",
                job_id=str(job_id),
                extraction_schema_id=extraction_schema_id,
                total_count=job.total_count
            )
            
            # 4. 대상 문서 ID 조회
            target_docs = await self._fetch_target_documents(
                db, extraction_schema_id, scope_id, date_from, date_to, sample_count
            )
            
            logger.info(f"Found {len(target_docs)} documents to re-extract")
            
            # 5. 청크 단위 처리
            completed = 0
            failed = 0
            skipped = 0
            
            for chunk_idx in range(0, len(target_docs), self.CHUNK_SIZE):
                # 취소 요청 확인
                job = await batch_repo.get_by_id(job_id, scope_id)
                if job and job.is_cancellation_requested:
                    logger.info(f"Cancellation requested for job {job_id}")
                    await batch_repo.update_status(
                        job_id, scope_id, BatchJobStatus.CANCELLED
                    )
                    break
                
                chunk = target_docs[chunk_idx : chunk_idx + self.CHUNK_SIZE]
                chunk_results = await self._process_chunk(
                    job_id, chunk, extraction_schema_id, scope_id, comparison_mode
                )
                
                completed += chunk_results["completed"]
                failed += chunk_results["failed"]
                skipped += chunk_results["skipped"]
                
                # 진행률 업데이트
                await batch_repo.update_progress(
                    job_id, scope_id, completed, failed, skipped
                )
                
                logger.info(
                    f"Chunk {chunk_idx // self.CHUNK_SIZE + 1}: "
                    f"completed={completed}, failed={failed}, skipped={skipped}"
                )
                
                # Rate limiting: 청크 처리 후 대기
                await asyncio.sleep(self.CHUNK_SIZE / self.RATE_LIMIT_PER_SEC)
            
            # 6. 최종 상태 업데이트
            error_summary = None
            if failed > 0:
                error_summary = f"{failed} documents failed out of {len(target_docs)}"
            
            await batch_repo.update_status(
                job_id,
                scope_id,
                BatchJobStatus.COMPLETED if failed == 0 else BatchJobStatus.FAILED,
                error_summary=error_summary
            )
            
            # 7. 최종 감시 로그
            await self.audit_logger.log(
                action="batch_extraction_completed",
                actor_type="agent",
                job_id=str(job_id),
                total=len(target_docs),
                completed=completed,
                failed=failed,
                skipped=skipped
            )
            
            logger.info(f"Batch job {job_id} completed: {completed}/{len(target_docs)}")
        
        except Exception as e:
            logger.error(f"Batch job {job_id} failed: {e}", exc_info=True)
            
            db_retry = SessionLocal()
            try:
                batch_repo = BatchExtractionJobRepository(db_retry, None)
                await batch_repo.update_status(
                    job_id,
                    scope_id,
                    BatchJobStatus.FAILED,
                    error_summary=str(e)
                )
            finally:
                db_retry.close()
        
        finally:
            db.close()
    
    async def _fetch_target_documents(
        self,
        db: Session,
        extraction_schema_id: str,
        scope_id: UUID,
        date_from: Optional[datetime],
        date_to: Optional[datetime],
        sample_count: Optional[int]
    ) -> List[UUID]:
        """
        재추출할 문서 ID 목록 조회.
        
        조건:
        - extraction_schema_id와 일치
        - ApprovedExtraction이 존재 (이미 승인된 것)
        - scope_id 일치
        - 날짜 범위 (optional)
        """
        query = db.query(ApprovedExtractionORM.document_id).filter(
            and_(
                ApprovedExtractionORM.extraction_schema_id == extraction_schema_id,
                ApprovedExtractionORM.scope_id == scope_id,
                ApprovedExtractionORM.is_deleted == False
            )
        )
        
        # 날짜 필터
        if date_from:
            query = query.filter(ApprovedExtractionORM.approved_at >= date_from)
        if date_to:
            query = query.filter(ApprovedExtractionORM.approved_at <= date_to)
        
        doc_ids = [row[0] for row in query.all()]
        
        # 샘플링
        if sample_count and sample_count < len(doc_ids):
            import random
            doc_ids = random.sample(doc_ids, sample_count)
        
        return doc_ids
    
    async def _process_chunk(
        self,
        job_id: UUID,
        document_ids: List[UUID],
        extraction_schema_id: str,
        scope_id: UUID,
        comparison_mode: bool
    ) -> dict:
        """청크 처리."""
        results = {"completed": 0, "failed": 0, "skipped": 0}
        
        batch_repo = BatchExtractionJobRepository(SessionLocal(), None)
        
        for doc_id in document_ids:
            attempt = 1
            success = False
            
            while attempt <= self.MAX_RETRIES and not success:
                try:
                    # 재추출 실행
                    await self.extraction_pipeline._execute_extraction_pipeline(
                        document_id=doc_id,
                        document_version=1,  # 현재 버전 사용
                        scope_id=scope_id,
                        doc_type=extraction_schema_id
                    )
                    
                    results["completed"] += 1
                    success = True
                    
                    # 재시도 로그
                    db = SessionLocal()
                    try:
                        await batch_repo.log_retry(
                            job_id=job_id,
                            document_id=doc_id,
                            attempt=attempt,
                            status="success",
                            error_reason=None,
                            latency_ms=0
                        )
                    finally:
                        db.close()
                
                except Exception as e:
                    logger.warning(
                        f"Extraction failed for doc {doc_id} attempt {attempt}: {e}"
                    )
                    
                    if attempt < self.MAX_RETRIES:
                        # exponential backoff
                        delay = self.RETRY_DELAYS[attempt - 1]
                        await asyncio.sleep(delay)
                    else:
                        # 최대 재시도 초과
                        results["failed"] += 1
                        db = SessionLocal()
                        try:
                            await batch_repo.add_failed_document(job_id, scope_id, doc_id)
                            await batch_repo.log_retry(
                                job_id=job_id,
                                document_id=doc_id,
                                attempt=attempt,
                                status="failed",
                                error_reason=str(e),
                                latency_ms=0
                            )
                        finally:
                            db.close()
                    
                    attempt += 1
        
        return results
```

### 4-5. 배치 API 엔드포인트

**파일:** `/backend/app/api/v1/endpoints/batch_extractions.py`

```python
"""
배치 재추출 API 엔드포인트.

- POST /api/v1/extractions/batch-retry
- GET /api/v1/extractions/batch-jobs/{job_id}
- POST /api/v1/extractions/batch-jobs/{job_id}/cancel
- GET /api/v1/extractions/batch-jobs/{job_id}/progress (SSE)
"""

import logging
from typing import AsyncGenerator
from uuid import UUID, uuid4

from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.responses import StreamingResponse
from sqlalchemy.orm import Session

from app.api.dependencies import get_db, get_current_user, get_scope_profile
from app.models.batch_extraction import (
    BatchRetryRequest, SampleRetryRequest, CancelBatchRequest,
    BatchExtractionJobResponse
)
from app.repositories.batch_extraction_repository import BatchExtractionJobRepository
from app.services.extraction.batch_extraction_service import BatchExtractionService
from app.core.scope import ScopeProfile
from app.core.audit import AuditLogger
from app.models.batch_extraction import BatchExtractionJob, BatchJobStatus
from datetime import datetime
import asyncio
import json


logger = logging.getLogger(__name__)
router = APIRouter(prefix="/extractions", tags=["batch-extractions"])


# ==================== Batch Retry ====================

@router.post(
    "/batch-retry",
    response_model=BatchExtractionJobResponse,
    summary="배치 재추출 시작",
    status_code=status.HTTP_202_ACCEPTED
)
async def start_batch_retry(
    req: BatchRetryRequest,
    db: Session = Depends(get_db),
    current_user: dict = Depends(get_current_user),
    scope_profile: ScopeProfile = Depends(get_scope_profile),
    audit_logger: AuditLogger = Depends(get_audit_logger),
    batch_service: BatchExtractionService = Depends(get_batch_service)
):
    """
    배치 재추출 작업 시작.
    
    요청 후 job_id를 받고, /batch-jobs/{job_id}로 진행률을 조회할 수 있다.
    """
    
    # 1. BatchExtractionJob 생성
    job = BatchExtractionJob(
        id=uuid4(),
        extraction_schema_id=req.extraction_schema_id,
        extraction_schema_version=req.extraction_schema_version,
        scope_id=scope_profile.scope_id,
        status=BatchJobStatus.PENDING,
        total_count=0,  # 실제 개수는 배치 시작 시 계산
        date_from=req.date_from,
        date_to=req.date_to,
        sample_count=req.sample_count,
        sample_mode=req.sample_count is not None,
        comparison_mode=req.comparison_mode,
        created_at=datetime.utcnow(),
        created_by=UUID(current_user["id"]),
        actor_type="user"
    )
    
    # 2. DB 저장
    repo = BatchExtractionJobRepository(db, scope_profile)
    saved_job = await repo.create(job)
    
    # 3. 비동기 백그라운드 작업 시작
    asyncio.create_task(
        batch_service.execute_batch_retry(
            job_id=saved_job.id,
            extraction_schema_id=req.extraction_schema_id,
            extraction_schema_version=req.extraction_schema_version,
            scope_id=scope_profile.scope_id,
            date_from=req.date_from,
            date_to=req.date_to,
            sample_count=req.sample_count,
            comparison_mode=req.comparison_mode
        )
    )
    
    # 4. 감시 로그
    await audit_logger.log(
        action="batch_extraction_requested",
        actor_type="user",
        user_id=current_user["id"],
        job_id=str(saved_job.id),
        extraction_schema_id=req.extraction_schema_id,
        sample_mode=req.sample_count is not None,
        sample_count=req.sample_count
    )
    
    db.commit()
    
    return BatchExtractionJobResponse(**saved_job.dict())


# ==================== Get Job Status ====================

@router.get(
    "/batch-jobs/{job_id}",
    response_model=BatchExtractionJobResponse,
    summary="배치 작업 상태 조회"
)
async def get_batch_job_status(
    job_id: UUID,
    db: Session = Depends(get_db),
    current_user: dict = Depends(get_current_user),
    scope_profile: ScopeProfile = Depends(get_scope_profile)
):
    """배치 작업의 현재 상태와 진행률을 조회."""
    
    repo = BatchExtractionJobRepository(db, scope_profile)
    job = await repo.get_by_id(job_id, scope_profile.scope_id)
    
    if not job:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Batch job not found"
        )
    
    return BatchExtractionJobResponse(**job.dict())


# ==================== Cancel Job ====================

@router.post(
    "/batch-jobs/{job_id}/cancel",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="배치 작업 취소"
)
async def cancel_batch_job(
    job_id: UUID,
    req: CancelBatchRequest,
    db: Session = Depends(get_db),
    current_user: dict = Depends(get_current_user),
    scope_profile: ScopeProfile = Depends(get_scope_profile),
    audit_logger: AuditLogger = Depends(get_audit_logger)
):
    """실행 중인 배치 작업을 취소 요청."""
    
    repo = BatchExtractionJobRepository(db, scope_profile)
    success = await repo.request_cancellation(job_id, scope_profile.scope_id)
    
    if not success:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Cannot cancel job (not in cancellable state)"
        )
    
    # 감시 로그
    await audit_logger.log(
        action="batch_extraction_cancel_requested",
        actor_type="user",
        user_id=current_user["id"],
        job_id=str(job_id),
        reason=req.reason
    )
    
    db.commit()


# ==================== Progress Stream (SSE) ====================

async def _progress_stream(
    job_id: UUID,
    scope_profile: ScopeProfile
) -> AsyncGenerator[str, None]:
    """SSE 진행률 스트림."""
    
    db = SessionLocal()
    repo = BatchExtractionJobRepository(db, scope_profile)
    
    last_progress = -1
    
    try:
        while True:
            job = await repo.get_by_id(job_id, scope_profile.scope_id)
            
            if not job:
                break
            
            # 진행률 변경 시만 업데이트
            if job.progress_percentage != last_progress:
                event_data = {
                    "progress": job.progress_percentage,
                    "completed": job.completed_count,
                    "total": job.total_count,
                    "failed": job.failed_count,
                    "status": job.status,
                    "estimated_completion_time": (
                        job.estimated_completion_time.isoformat()
                        if job.estimated_completion_time
                        else None
                    )
                }
                
                yield f"data: {json.dumps(event_data)}\n\n"
                last_progress = job.progress_percentage
            
            # 작업 완료 또는 취소되면 종료
            if job.status in (
                BatchJobStatus.COMPLETED,
                BatchJobStatus.FAILED,
                BatchJobStatus.CANCELLED
            ):
                break
            
            # 1초마다 체크
            await asyncio.sleep(1)
    
    finally:
        db.close()


@router.get(
    "/batch-jobs/{job_id}/progress",
    summary="배치 작업 진행률 (SSE)"
)
async def stream_batch_progress(
    job_id: UUID,
    scope_profile: ScopeProfile = Depends(get_scope_profile)
):
    """
    배치 작업의 실시간 진행률을 SSE로 전송.
    
    클라이언트:
    ```javascript
    const es = new EventSource(`/api/v1/extractions/batch-jobs/${jobId}/progress`);
    es.onmessage = (event) => {
      const data = JSON.parse(event.data);
      console.log(`Progress: ${data.progress}%`);
    };
    ```
    """
    
    return StreamingResponse(
        _progress_stream(job_id, scope_profile),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        }
    )


# ==================== Dependencies ====================

def get_audit_logger() -> AuditLogger:
    from app.core.audit import AuditLogger
    return AuditLogger()


def get_batch_service() -> BatchExtractionService:
    from app.services.extraction.extraction_pipeline_service import ExtractionPipelineService
    from app.services.extraction.batch_extraction_service import BatchExtractionService
    
    pipeline = ExtractionPipelineService(SessionLocal(), None, None, None, None)
    logger = get_audit_logger()
    
    return BatchExtractionService(pipeline, logger)


from app.db.database import SessionLocal
from sqlalchemy import and_
```

### 4-6. 단위 및 통합 테스트

**파일:** `/backend/tests/services/extraction/test_batch_extraction_service.py`

```python
"""
배치 재추출 서비스 테스트.
"""

import pytest
from datetime import datetime
from uuid import uuid4
from unittest.mock import AsyncMock, MagicMock, patch

from app.models.batch_extraction import BatchJobStatus
from app.services.extraction.batch_extraction_service import BatchExtractionService


@pytest.fixture
def mock_extraction_pipeline():
    """Mock ExtractionPipelineService."""
    pipeline = AsyncMock()
    pipeline._execute_extraction_pipeline = AsyncMock()
    return pipeline


@pytest.fixture
def mock_audit_logger():
    """Mock AuditLogger."""
    logger = AsyncMock()
    logger.log = AsyncMock()
    return logger


@pytest.fixture
def batch_service(mock_extraction_pipeline, mock_audit_logger):
    """BatchExtractionService 인스턴스."""
    return BatchExtractionService(
        extraction_pipeline=mock_extraction_pipeline,
        audit_logger=mock_audit_logger
    )


@pytest.mark.asyncio
async def test_execute_batch_retry_success(batch_service, mock_extraction_pipeline):
    """배치 재추출 성공."""
    job_id = uuid4()
    scope_id = uuid4()
    doc_ids = [uuid4() for _ in range(3)]
    
    # Mock: 대상 문서 조회
    with patch.object(
        batch_service,
        "_fetch_target_documents",
        return_value=doc_ids
    ):
        # 실행
        await batch_service.execute_batch_retry(
            job_id=job_id,
            extraction_schema_id="regulation",
            extraction_schema_version=1,
            scope_id=scope_id,
            date_from=None,
            date_to=None,
            sample_count=None,
            comparison_mode=False
        )
    
    # 검증: 모든 문서에 대해 추출 호출됨
    assert mock_extraction_pipeline._execute_extraction_pipeline.call_count >= 3


@pytest.mark.asyncio
async def test_batch_retry_with_partial_failures(batch_service, mock_extraction_pipeline):
    """배치 재추출: 부분 실패."""
    job_id = uuid4()
    scope_id = uuid4()
    doc_ids = [uuid4() for _ in range(3)]
    
    # 첫 번째 호출은 실패, 나머지는 성공
    call_count = [0]
    
    async def side_effect(*args, **kwargs):
        call_count[0] += 1
        if call_count[0] == 1:
            raise Exception("Simulated failure")
        return None
    
    mock_extraction_pipeline._execute_extraction_pipeline = AsyncMock(side_effect=side_effect)
    
    with patch.object(
        batch_service,
        "_fetch_target_documents",
        return_value=doc_ids
    ):
        # 실행: 재시도 포함하여 완료되어야 함
        await batch_service.execute_batch_retry(
            job_id=job_id,
            extraction_schema_id="regulation",
            extraction_schema_version=1,
            scope_id=scope_id
        )


@pytest.mark.asyncio
async def test_sample_extraction(batch_service):
    """샘플 재추출."""
    job_id = uuid4()
    scope_id = uuid4()
    
    # 100개 중 10개 샘플 선택
    with patch.object(
        batch_service,
        "_fetch_target_documents",
        return_value=[uuid4() for _ in range(10)]
    ):
        await batch_service.execute_batch_retry(
            job_id=job_id,
            extraction_schema_id="regulation",
            extraction_schema_version=1,
            scope_id=scope_id,
            sample_count=10
        )


@pytest.mark.asyncio
async def test_cancellation_request(batch_service):
    """배치 취소 요청."""
    # 배치 실행 중 취소 요청 → 정상 종료
    # 이 테스트는 통합 테스트로 진행하는 것이 좋음
    pass
```

## 5. 산출물

1. **BatchExtractionJob Pydantic 모델**
   - 파일: `/backend/app/models/batch_extraction.py`
   - 크기: ~9KB

2. **BatchExtractionJobORM + Alembic 마이그레이션**
   - 파일: `/backend/app/db/models/batch_extraction.py`
   - 파일: `/backend/alembic/versions/*`
   - 크기: ~7KB

3. **BatchExtractionJobRepository**
   - 파일: `/backend/app/repositories/batch_extraction_repository.py`
   - 크기: ~12KB

4. **BatchExtractionService (워커)**
   - 파일: `/backend/app/services/extraction/batch_extraction_service.py`
   - 크기: ~18KB

5. **배치 API 엔드포인트**
   - 파일: `/backend/app/api/v1/endpoints/batch_extractions.py`
   - 크기: ~16KB

6. **테스트**
   - 파일: `/backend/tests/services/extraction/test_batch_extraction_service.py`
   - 크기: ~8KB

## 6. 완료 기준

1. BatchExtractionJob 도메인 모델이 정의되고, 모든 상태 전이가 명확하다.
2. BatchExtractionJobORM이 정의되고, Alembic 마이그레이션이 작동한다.
3. 3가지 API 엔드포인트 (start, status, cancel)가 모두 구현되고 작동한다.
4. 배치 워커가 청크 단위 (N=10)로 문서를 처리한다.
5. Rate limiting (초당 최대 5개)이 적용되어 LLM API 호출 속도가 제한된다.
6. 지수 백오프 재시도 (3회)가 구현되어 실패한 문서를 재시도한다.
7. 부분 실패 시에도 계속 처리되고, failed_documents 목록에 기록된다.
8. 진행률이 실시간으로 업데이트되고 SSE로 클라이언트에 전송된다.
9. 배치 취소 요청 (cancellation_requested 플래그)이 작동한다.
10. 감시 로그가 배치 시작, 완료, 실패 시에 기록된다.
11. Scope Profile ACL이 모든 연산에 적용된다.
12. 단위 테스트가 90% 이상 코드 커버리지를 달성한다.

## 7. 작업 지침

### 지침 7-1. 비동기 실행
배치 작업은 메인 요청을 블로킹하지 않도록 `asyncio.create_task()` 또는 Celery로 비동기 실행하라.

### 지침 7-2. 청크 처리
한 번에 모든 문서를 처리하지 말고, CHUNK_SIZE (10) 단위로 처리하여 메모리 사용을 제어하라.

### 지침 7-3. Rate Limiting
LLM API 호출 속도를 초당 최대 RATE_LIMIT_PER_SEC (5)로 제한하여 API 할당량(quota)를 초과하지 않도록 하라.

### 지침 7-4. 지수 백오프 재시도
RETRY_DELAYS = [1, 2, 4]로 exponential backoff를 구현하여 일시적 오류에 대한 복원력을 높이라.

### 지침 7-5. 부분 실패 처리
개별 문서 실패 시 다음 문서로 계속 진행하되, failed_documents 목록에 추적하라.

### 지침 7-6. 진행률 추적
매 청크 완료마다 DB 업데이트 (completed_count, progress_percentage)하고, 예상 완료 시간을 계산하라.

### 지침 7-7. 취소 메커니즘
is_cancellation_requested 플래그를 주기적으로 확인하여 사용자의 취소 요청에 빠르게 응답하라.

---

End of Task 8-7
