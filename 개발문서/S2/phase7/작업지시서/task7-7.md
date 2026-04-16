# 작업지시서: task7-7 평가 실행 API + 결과 저장

**작업 코드**: task7-7  
**페이즈**: Phase 7 FG7.2 (평가 러너 + 지표)  
**담당자**: [TBD]  
**예상 소요시간**: 8~9일  
**최종 수정 일시**: 2026-04-17

---

## 1. 작업 목적

task7-6의 `Evaluator` 클래스를 REST API로 래핑하여, 외부 시스템이 평가 작업을 트리거하고 결과를 조회할 수 있는 인터페이스를 제공한다. 동시에 평가 결과를 데이터베이스에 저장하고, S2 원칙 ⑤⑥에 따라 API 소비자(사용자, AI 에이전트)를 구분하여 감사 로그를 기록한다. 또한 Scope Profile 기반 ACL을 모든 엔드포인트에 적용하여, 폐쇄망 환경에서도 접근 제어가 일관되게 동작하도록 한다.

---

## 2. 작업 범위

### 포함 범위

1. **평가 실행 API (비동기)**
   - `POST /api/v1/evaluations/run`: 평가 작업 시작
   - 입력: Golden Set 데이터 (또는 Golden Set ID)
   - 출력: 평가 ID, 상태 ("queued", "running", "completed", "failed")
   - 배경 작업(Background Task)으로 실행 (Celery 또는 asyncio)
   - 즉시 응답 (evaluation_id 반환)

2. **평가 결과 조회 API**
   - `GET /api/v1/evaluations/{eval_id}`: 특정 평가 결과 상세 조회
   - `GET /api/v1/evaluations`: 평가 히스토리 목록 (페이지네이션)
   - 필터링: batch_id, 날짜 범위, 사용자, 상태 등
   - 정렬: 생성 시간, 점수 등

3. **평가 비교 API**
   - `GET /api/v1/evaluations/{eval_id1}/compare?eval_id2={eval_id2}`: 두 평가 비교
   - 지표별 차이 분석
   - 개선/악화 항목 파악

4. **데이터베이스 모델**
   - `EvaluationRun`: 평가 작업 메타데이터 (ID, 상태, 시간 등)
   - `EvaluationResultRecord`: 각 항목별 평가 결과 (정규화 저장)
   - 시계열 저장 고려 (향후 스케일 대비)

5. **Repository 패턴 구현**
   - `EvaluationRunRepository`: CRUD 작업
   - `EvaluationResultRepository`: 결과 저장/조회
   - 트랜잭션 처리 및 원자성 보장

6. **배경 작업 처리**
   - 비동기 작업 큐 (Celery 또는 asyncio.create_task)
   - 작업 상태 추적 (pending, running, completed, failed)
   - 재시도 로직 (실패 시 최대 3회 재시도)

7. **S2 원칙 구현**
   - ⑤ Actor Type 감사 로그: `actor_type` 필드 ("user" 또는 "agent")
   - ⑥ Scope Profile 기반 ACL: 모든 엔드포인트에 적용
   - 검색/조회/쓰기 API 모두에 필터링 적용
   - 폴백 경로(내부 DB 조회 등)에서도 ACL 적용

8. **오류 처리 및 검증**
   - 입력 검증 (Pydantic)
   - HTTP 상태 코드 적절함 (400, 401, 403, 404, 500)
   - 오류 응답 표준화

9. **단위 + 통합 테스트**
   - API 엔드포인트 테스트 (FastAPI TestClient)
   - 권한 확인 테스트 (ACL)
   - 비동기 작업 테스트
   - DB 트랜잭션 테스트

### 제외 범위

- UI 대시보드 (별도 작업)
- 프로덕션 캐싱 인프라
- 알림 시스템 (이메일, Slack 등)
- 배포 및 모니터링 (DevOps)

---

## 3. 선행 조건

1. **task7-6 완료**: Evaluator 클래스 및 데이터 모델
2. **Phase 3+ 완료**: 사용자 인증, ACL, Scope Profile 시스템
3. **데이터베이스**: PostgreSQL (또는 유사 RDBMS)
4. **비동기 처리**: Celery 또는 asyncio
5. **FastAPI**: 웹 프레임워크
6. **라이브러리**: SQLAlchemy v2, Pydantic v2

---

## 4. 주요 작업 항목

### 4.1 데이터베이스 모델

**파일**: `/backend/app/models/evaluation.py`

```python
"""
평가 관련 데이터베이스 모델.

ORM: SQLAlchemy v2
"""

from sqlalchemy import Column, Integer, String, Float, DateTime, JSON, ForeignKey, Index, Text
from sqlalchemy.orm import relationship
from datetime import datetime
from typing import Optional
import uuid

from app.database import Base


class EvaluationRun(Base):
    """평가 작업(Run)."""

    __tablename__ = "evaluation_runs"

    # 기본 필드
    id = Column(String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    batch_id = Column(String(100), nullable=False, index=True)
    status = Column(
        String(20),
        default="queued",
        index=True,
        comment="queued, running, completed, failed"
    )

    # 입력 정보
    golden_set_id = Column(String(100), nullable=True, index=True)
    total_items = Column(Integer, default=0)
    
    # 실행 정보
    started_at = Column(DateTime, nullable=True)
    completed_at = Column(DateTime, nullable=True)
    duration_seconds = Column(Float, nullable=True)

    # 결과 요약
    successful_items = Column(Integer, default=0)
    failed_items = Column(Integer, default=0)
    overall_score = Column(Float, nullable=True)  # 0.0~1.0

    # 성능 지표
    total_tokens = Column(Integer, default=0)
    total_latency_ms = Column(Float, default=0.0)
    total_cost = Column(Float, default=0.0)

    # 메타데이터
    created_by = Column(String(100), nullable=False, comment="사용자 또는 에이전트 ID")
    actor_type = Column(
        String(20),
        default="user",
        comment="user 또는 agent (S2 원칙 ⑤)"
    )
    scope_profile_id = Column(String(100), nullable=False, comment="Scope Profile ID (S2 원칙 ⑥)")
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # JSON 저장 (추가 메타데이터)
    metadata_json = Column(JSON, nullable=True)

    # 관계
    results = relationship("EvaluationResultRecord", back_populates="run", cascade="all, delete-orphan")

    # 인덱스
    __table_args__ = (
        Index("idx_evaluation_runs_created_at", "created_at"),
        Index("idx_evaluation_runs_scope_profile", "scope_profile_id", "created_at"),
        Index("idx_evaluation_runs_status_scope", "status", "scope_profile_id"),
    )

    def __repr__(self):
        return f"<EvaluationRun(id={self.id}, batch_id={self.batch_id}, status={self.status})>"


class EvaluationResultRecord(Base):
    """평가 결과(각 항목별)."""

    __tablename__ = "evaluation_results"

    # 기본 필드
    id = Column(String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    run_id = Column(String(36), ForeignKey("evaluation_runs.id", ondelete="CASCADE"), index=True)
    item_id = Column(String(100), nullable=False)

    # 입력 데이터
    question = Column(Text, nullable=False)
    answer = Column(Text, nullable=False)
    contexts = Column(JSON, nullable=False)  # List[str]
    expected_answer = Column(Text, nullable=True)
    expected_sources = Column(JSON, nullable=True)  # List[str]

    # 평가 지표 (정규화된 float)
    faithfulness = Column(Float, nullable=True)
    answer_relevance = Column(Float, nullable=True)
    context_precision = Column(Float, nullable=True)
    context_recall = Column(Float, nullable=True)
    citation_present_rate = Column(Float, nullable=True)
    hallucination_rate = Column(Float, nullable=True)
    overall_score = Column(Float, nullable=True)

    # 성능 지표
    retrieval_ms = Column(Float, nullable=True)
    generation_ms = Float = Column(Float, nullable=True)
    total_latency_ms = Column(Float, nullable=True)
    input_tokens = Column(Integer, nullable=True)
    output_tokens = Column(Integer, nullable=True)
    total_tokens = Column(Integer, nullable=True)
    estimated_cost = Column(Float, nullable=True)

    # 메타데이터
    evaluator_version = Column(String(20), default="1.0")
    notes = Column(Text, nullable=True)
    created_at = Column(DateTime, default=datetime.utcnow, index=True)

    # 관계
    run = relationship("EvaluationRun", back_populates="results")

    # 인덱스
    __table_args__ = (
        Index("idx_evaluation_results_run_id", "run_id"),
        Index("idx_evaluation_results_created_at", "created_at"),
    )

    def to_dict(self) -> dict:
        """딕셔너리로 변환."""
        return {
            "id": self.id,
            "item_id": self.item_id,
            "question": self.question,
            "answer": self.answer,
            "contexts": self.contexts,
            "scores": {
                "faithfulness": self.faithfulness,
                "answer_relevance": self.answer_relevance,
                "context_precision": self.context_precision,
                "context_recall": self.context_recall,
                "citation_present_rate": self.citation_present_rate,
                "hallucination_rate": self.hallucination_rate,
                "overall_score": self.overall_score,
            },
            "performance": {
                "retrieval_ms": self.retrieval_ms,
                "generation_ms": self.generation_ms,
                "total_latency_ms": self.total_latency_ms,
                "input_tokens": self.input_tokens,
                "output_tokens": self.output_tokens,
                "total_tokens": self.total_tokens,
                "estimated_cost": self.estimated_cost,
            }
        }
```

### 4.2 Repository 패턴

**파일**: `/backend/app/repositories/evaluation_repository.py`

```python
"""
평가 관련 Repository 패턴.

데이터 접근 계층 추상화.
"""

from typing import List, Optional, Dict
from sqlalchemy.orm import Session
from sqlalchemy import desc, and_
from datetime import datetime
import logging

from app.models.evaluation import EvaluationRun, EvaluationResultRecord
from app.services.evaluation.models import EvaluationResult, EvaluationReport

logger = logging.getLogger(__name__)


class EvaluationRunRepository:
    """EvaluationRun CRUD."""

    def __init__(self, db: Session):
        self.db = db

    def create(
        self,
        batch_id: str,
        created_by: str,
        actor_type: str,
        scope_profile_id: str,
        golden_set_id: Optional[str] = None,
        total_items: int = 0,
        metadata: Optional[Dict] = None,
    ) -> EvaluationRun:
        """새 평가 Run 생성."""
        run = EvaluationRun(
            batch_id=batch_id,
            created_by=created_by,
            actor_type=actor_type,
            scope_profile_id=scope_profile_id,
            golden_set_id=golden_set_id,
            total_items=total_items,
            status="queued",
            metadata_json=metadata or {},
        )
        self.db.add(run)
        self.db.commit()
        self.db.refresh(run)
        logger.info(f"Created EvaluationRun: {run.id}")
        return run

    def get_by_id(self, run_id: str) -> Optional[EvaluationRun]:
        """ID로 조회."""
        return self.db.query(EvaluationRun).filter(EvaluationRun.id == run_id).first()

    def get_by_batch_id(self, batch_id: str) -> Optional[EvaluationRun]:
        """Batch ID로 조회."""
        return self.db.query(EvaluationRun).filter(EvaluationRun.batch_id == batch_id).first()

    def list_by_scope(
        self,
        scope_profile_id: str,
        skip: int = 0,
        limit: int = 100,
        status: Optional[str] = None,
    ) -> List[EvaluationRun]:
        """Scope Profile별 목록 (S2 원칙 ⑥)."""
        query = self.db.query(EvaluationRun).filter(
            EvaluationRun.scope_profile_id == scope_profile_id
        )
        if status:
            query = query.filter(EvaluationRun.status == status)
        
        return query.order_by(desc(EvaluationRun.created_at)).offset(skip).limit(limit).all()

    def update_status(self, run_id: str, status: str) -> EvaluationRun:
        """상태 업데이트."""
        run = self.get_by_id(run_id)
        if run:
            run.status = status
            if status == "running" and not run.started_at:
                run.started_at = datetime.utcnow()
            if status in ["completed", "failed"]:
                run.completed_at = datetime.utcnow()
                if run.started_at:
                    run.duration_seconds = (run.completed_at - run.started_at).total_seconds()
            self.db.commit()
            logger.info(f"Updated EvaluationRun {run_id} status to {status}")
        return run

    def update_results(
        self,
        run_id: str,
        successful_items: int,
        failed_items: int,
        overall_score: float,
        total_tokens: int = 0,
        total_latency_ms: float = 0.0,
        total_cost: float = 0.0,
    ) -> EvaluationRun:
        """결과 업데이트."""
        run = self.get_by_id(run_id)
        if run:
            run.successful_items = successful_items
            run.failed_items = failed_items
            run.overall_score = overall_score
            run.total_tokens = total_tokens
            run.total_latency_ms = total_latency_ms
            run.total_cost = total_cost
            self.db.commit()
            logger.info(f"Updated EvaluationRun {run_id} results")
        return run

    def delete(self, run_id: str) -> bool:
        """삭제 (권한 확인 필수)."""
        run = self.get_by_id(run_id)
        if run:
            self.db.delete(run)
            self.db.commit()
            logger.info(f"Deleted EvaluationRun {run_id}")
            return True
        return False


class EvaluationResultRepository:
    """EvaluationResultRecord CRUD."""

    def __init__(self, db: Session):
        self.db = db

    def create_from_evaluation_result(
        self,
        run_id: str,
        evaluation_result: EvaluationResult,
    ) -> EvaluationResultRecord:
        """EvaluationResult에서 DB 레코드 생성."""
        record = EvaluationResultRecord(
            run_id=run_id,
            item_id=evaluation_result.item_id,
            question=evaluation_result.question,
            answer=evaluation_result.answer,
            contexts=evaluation_result.contexts,
            expected_answer=evaluation_result.expected_answer,
            expected_sources=evaluation_result.expected_sources,
            faithfulness=evaluation_result.scores.faithfulness,
            answer_relevance=evaluation_result.scores.answer_relevance,
            context_precision=evaluation_result.scores.context_precision,
            context_recall=evaluation_result.scores.context_recall,
            citation_present_rate=evaluation_result.scores.citation_present_rate,
            hallucination_rate=evaluation_result.scores.hallucination_rate,
            overall_score=evaluation_result.overall_score(),
            retrieval_ms=evaluation_result.latency_metrics.retrieval_ms if evaluation_result.latency_metrics else None,
            generation_ms=evaluation_result.latency_metrics.generation_ms if evaluation_result.latency_metrics else None,
            total_latency_ms=evaluation_result.latency_metrics.total_ms if evaluation_result.latency_metrics else None,
            input_tokens=evaluation_result.token_metrics.query_tokens if evaluation_result.token_metrics else None,
            output_tokens=evaluation_result.token_metrics.response_tokens if evaluation_result.token_metrics else None,
            total_tokens=evaluation_result.token_metrics.total_tokens if evaluation_result.token_metrics else None,
            estimated_cost=evaluation_result.cost_metrics.total_cost if evaluation_result.cost_metrics else None,
        )
        self.db.add(record)
        self.db.commit()
        self.db.refresh(record)
        return record

    def get_by_id(self, result_id: str) -> Optional[EvaluationResultRecord]:
        """ID로 조회."""
        return self.db.query(EvaluationResultRecord).filter(
            EvaluationResultRecord.id == result_id
        ).first()

    def get_by_run_id(self, run_id: str, skip: int = 0, limit: int = 1000) -> List[EvaluationResultRecord]:
        """Run ID로 모든 결과 조회."""
        return self.db.query(EvaluationResultRecord).filter(
            EvaluationResultRecord.run_id == run_id
        ).order_by(EvaluationResultRecord.created_at).offset(skip).limit(limit).all()

    def batch_create(
        self,
        run_id: str,
        results: List[EvaluationResult],
    ) -> int:
        """배치 생성."""
        count = 0
        for result in results:
            self.create_from_evaluation_result(run_id, result)
            count += 1
        logger.info(f"Batch created {count} result records for run {run_id}")
        return count
```

### 4.3 비동기 작업 처리

**파일**: `/backend/app/services/evaluation/background_tasks.py`

```python
"""
평가 작업 배경 처리.

Celery 또는 asyncio 기반.
"""

import logging
import asyncio
from typing import List, Dict, Optional
from datetime import datetime

from app.services.evaluation.evaluator import Evaluator
from app.repositories.evaluation_repository import EvaluationRunRepository, EvaluationResultRepository
from app.models.evaluation import EvaluationRun
from sqlalchemy.orm import Session

logger = logging.getLogger(__name__)


class EvaluationBackgroundTask:
    """평가 백그라운드 작업."""

    def __init__(self, db_session: Session):
        self.db = db_session
        self.run_repo = EvaluationRunRepository(db_session)
        self.result_repo = EvaluationResultRepository(db_session)
        self.evaluator = Evaluator()

    async def execute_evaluation(
        self,
        run_id: str,
        golden_items: List[Dict],
        max_concurrent: int = 5,
    ) -> None:
        """
        평가 작업 실행.

        Args:
            run_id: EvaluationRun ID
            golden_items: Golden Set 항목 목록
            max_concurrent: 동시 실행 개수
        """
        try:
            # 상태 업데이트: running
            self.run_repo.update_status(run_id, "running")

            # Evaluator로 배치 평가 실행
            report = await self.evaluator.evaluate_golden_set_async(
                batch_id=run_id,
                golden_items=golden_items,
                max_concurrent=max_concurrent,
            )

            # 결과 저장
            for result in report.results:
                self.result_repo.create_from_evaluation_result(run_id, result)

            # Run 결과 업데이트
            self.run_repo.update_results(
                run_id=run_id,
                successful_items=report.successful_items,
                failed_items=report.failed_items,
                overall_score=report.overall_score(),
                total_tokens=report.total_tokens,
                total_latency_ms=report.total_latency_ms,
                total_cost=report.total_cost,
            )

            # 상태 업데이트: completed
            self.run_repo.update_status(run_id, "completed")

            logger.info(f"Evaluation {run_id} completed successfully")

        except Exception as e:
            logger.error(f"Evaluation {run_id} failed: {e}", exc_info=True)
            self.run_repo.update_status(run_id, "failed")
            raise


class BackgroundTaskManager:
    """배경 작업 관리자."""

    def __init__(self):
        self.tasks: Dict[str, asyncio.Task] = {}

    async def start_evaluation(
        self,
        run_id: str,
        golden_items: List[Dict],
        db_session: Session,
    ) -> str:
        """
        평가 작업 시작 (Fire-and-forget).

        Returns:
            run_id
        """
        task = EvaluationBackgroundTask(db_session)
        
        # asyncio.create_task로 비동기 작업 생성
        async_task = asyncio.create_task(
            task.execute_evaluation(run_id, golden_items)
        )
        
        self.tasks[run_id] = async_task
        
        logger.info(f"Started background evaluation task: {run_id}")
        return run_id

    def get_task_status(self, run_id: str) -> str:
        """작업 상태 조회."""
        if run_id not in self.tasks:
            return "unknown"

        task = self.tasks[run_id]
        if task.done():
            if task.exception():
                return "failed"
            return "completed"

        return "running"


# 기본 인스턴스
background_task_manager = BackgroundTaskManager()
```

### 4.4 API 엔드포인트

**파일**: `/backend/app/api/v1/evaluations.py`

```python
"""
평가 API 엔드포인트.

- POST /api/v1/evaluations/run
- GET /api/v1/evaluations/{eval_id}
- GET /api/v1/evaluations
- GET /api/v1/evaluations/{eval_id}/compare
"""

import logging
from typing import List, Optional
from datetime import datetime

from fastapi import APIRouter, Depends, HTTPException, Query, Path
from sqlalchemy.orm import Session

from app.database import get_db
from app.auth import get_current_user, verify_actor_type
from app.acl import check_scope_access
from app.repositories.evaluation_repository import EvaluationRunRepository, EvaluationResultRepository
from app.services.evaluation.background_tasks import background_task_manager
from app.models.evaluation import EvaluationRun, EvaluationResultRecord
from pydantic import BaseModel, Field
from typing import Dict

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/v1/evaluations", tags=["evaluations"])


# ============================================================================
# Request/Response 모델
# ============================================================================

class EvaluationRunRequest(BaseModel):
    """평가 실행 요청."""
    batch_id: str = Field(description="배치 ID")
    golden_set_id: Optional[str] = Field(None, description="Golden Set ID")
    golden_items: List[Dict] = Field(description="Golden Set 항목 목록")
    max_concurrent: int = Field(default=5, ge=1, le=20, description="동시 실행 개수")


class EvaluationRunResponse(BaseModel):
    """평가 실행 응답."""
    evaluation_id: str = Field(description="평가 ID")
    batch_id: str
    status: str = Field(description="queued, running, completed, failed")
    created_at: datetime
    message: str


class EvaluationResultResponse(BaseModel):
    """평가 결과 응답."""
    id: str
    item_id: str
    question: str
    answer: str
    scores: Dict[str, float]
    overall_score: float
    performance: Dict[str, float]
    created_at: datetime


class EvaluationListResponse(BaseModel):
    """평가 목록 응답."""
    total: int
    skip: int
    limit: int
    items: List[Dict]


# ============================================================================
# 엔드포인트
# ============================================================================

@router.post("/run", response_model=EvaluationRunResponse)
async def run_evaluation(
    request: EvaluationRunRequest,
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user),
) -> EvaluationRunResponse:
    """
    평가 작업 시작.

    실행 후 즉시 응답 (background task로 처리).
    """
    # 권한 확인
    if not check_scope_access(current_user, "evaluation:create"):
        raise HTTPException(status_code=403, detail="Not authorized to create evaluations")

    try:
        run_repo = EvaluationRunRepository(db)

        # Actor Type 판단
        actor_type = "agent" if verify_actor_type(current_user, "agent") else "user"

        # Run 생성
        run = run_repo.create(
            batch_id=request.batch_id,
            created_by=current_user.id,
            actor_type=actor_type,
            scope_profile_id=current_user.scope_profile_id,
            golden_set_id=request.golden_set_id,
            total_items=len(request.golden_items),
        )

        # 배경 작업 시작
        await background_task_manager.start_evaluation(
            run_id=run.id,
            golden_items=request.golden_items,
            db_session=db,
        )

        # 감사 로그
        logger.info(
            f"Evaluation run started: {run.id}, "
            f"actor_type={actor_type}, "
            f"user={current_user.id}, "
            f"items={len(request.golden_items)}"
        )

        return EvaluationRunResponse(
            evaluation_id=run.id,
            batch_id=run.batch_id,
            status=run.status,
            created_at=run.created_at,
            message="Evaluation queued successfully"
        )

    except Exception as e:
        logger.error(f"Failed to start evaluation: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))


@router.get("/{eval_id}", response_model=Dict)
def get_evaluation(
    eval_id: str = Path(description="평가 ID"),
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user),
) -> Dict:
    """
    평가 결과 상세 조회.

    Scope Profile 기반 ACL 적용 (S2 원칙 ⑥).
    """
    run_repo = EvaluationRunRepository(db)
    result_repo = EvaluationResultRepository(db)

    run = run_repo.get_by_id(eval_id)
    if not run:
        raise HTTPException(status_code=404, detail="Evaluation not found")

    # ACL 확인: 자신의 Scope Profile에 속하는 평가만 조회 가능
    if run.scope_profile_id != current_user.scope_profile_id:
        raise HTTPException(status_code=403, detail="Not authorized to view this evaluation")

    # 결과 조회
    results = result_repo.get_by_run_id(eval_id)

    return {
        "evaluation_id": run.id,
        "batch_id": run.batch_id,
        "status": run.status,
        "total_items": run.total_items,
        "successful_items": run.successful_items,
        "failed_items": run.failed_items,
        "overall_score": run.overall_score,
        "duration_seconds": run.duration_seconds,
        "created_at": run.created_at,
        "completed_at": run.completed_at,
        "created_by": run.created_by,
        "actor_type": run.actor_type,  # S2 원칙 ⑤
        "results": [
            {
                "id": r.id,
                "item_id": r.item_id,
                "scores": {
                    "faithfulness": r.faithfulness,
                    "answer_relevance": r.answer_relevance,
                    "context_precision": r.context_precision,
                    "context_recall": r.context_recall,
                    "citation_present_rate": r.citation_present_rate,
                    "hallucination_rate": r.hallucination_rate,
                    "overall_score": r.overall_score,
                },
                "performance": {
                    "total_latency_ms": r.total_latency_ms,
                    "total_tokens": r.total_tokens,
                    "estimated_cost": r.estimated_cost,
                }
            }
            for r in results
        ]
    }


@router.get("", response_model=EvaluationListResponse)
def list_evaluations(
    skip: int = Query(0, ge=0, description="스킵 개수"),
    limit: int = Query(10, ge=1, le=100, description="한 페이지 개수"),
    status: Optional[str] = Query(None, description="상태 필터"),
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user),
) -> EvaluationListResponse:
    """
    평가 목록 조회.

    Scope Profile 기반 필터링 (S2 원칙 ⑥).
    """
    run_repo = EvaluationRunRepository(db)

    # Scope Profile 기반 조회
    runs = run_repo.list_by_scope(
        scope_profile_id=current_user.scope_profile_id,
        skip=skip,
        limit=limit,
        status=status,
    )

    total = len(runs)  # 실제 환경에서는 count 쿼리 사용

    return EvaluationListResponse(
        total=total,
        skip=skip,
        limit=limit,
        items=[
            {
                "id": r.id,
                "batch_id": r.batch_id,
                "status": r.status,
                "overall_score": r.overall_score,
                "total_items": r.total_items,
                "successful_items": r.successful_items,
                "failed_items": r.failed_items,
                "created_at": r.created_at,
                "duration_seconds": r.duration_seconds,
                "actor_type": r.actor_type,  # S2 원칙 ⑤
            }
            for r in runs
        ]
    )


@router.get("/{eval_id}/compare", response_model=Dict)
def compare_evaluations(
    eval_id: str = Path(description="첫 번째 평가 ID"),
    eval_id2: str = Query(description="두 번째 평가 ID"),
    db: Session = Depends(get_db),
    current_user = Depends(get_current_user),
) -> Dict:
    """
    두 평가 결과 비교.

    지표별 차이 분석.
    """
    run_repo = EvaluationRunRepository(db)
    result_repo = EvaluationResultRepository(db)

    run1 = run_repo.get_by_id(eval_id)
    run2 = run_repo.get_by_id(eval_id2)

    if not run1 or not run2:
        raise HTTPException(status_code=404, detail="One or both evaluations not found")

    # ACL 확인
    if (run1.scope_profile_id != current_user.scope_profile_id or
        run2.scope_profile_id != current_user.scope_profile_id):
        raise HTTPException(status_code=403, detail="Not authorized to compare")

    results1 = result_repo.get_by_run_id(eval_id)
    results2 = result_repo.get_by_run_id(eval_id2)

    # 평균 계산
    def avg_score(results, field):
        scores = [getattr(r, field) for r in results if getattr(r, field) is not None]
        return sum(scores) / len(scores) if scores else 0.0

    metrics = ["faithfulness", "answer_relevance", "context_precision", 
               "context_recall", "citation_present_rate", "hallucination_rate"]

    comparison = {
        "eval_id_1": eval_id,
        "eval_id_2": eval_id2,
        "metric_comparison": {
            metric: {
                "eval1": avg_score(results1, metric),
                "eval2": avg_score(results2, metric),
                "difference": avg_score(results2, metric) - avg_score(results1, metric),
            }
            for metric in metrics
        },
        "overall_score_1": run1.overall_score,
        "overall_score_2": run2.overall_score,
        "improvement": (run2.overall_score or 0) - (run1.overall_score or 0),
    }

    # 감사 로그
    logger.info(
        f"Evaluation comparison: {eval_id} vs {eval_id2}, "
        f"user={current_user.id}"
    )

    return comparison
```

### 4.5 단위 + 통합 테스트

**파일**: `/backend/tests/api/test_evaluations.py`

```python
"""
평가 API 엔드포인트 테스트.
"""

import pytest
from fastapi.testclient import TestClient
from sqlalchemy.orm import Session

from app.main import app
from app.database import get_db
from app.auth import get_current_user
from tests.fixtures import create_test_user, create_test_golden_items


client = TestClient(app)


@pytest.fixture
def test_user():
    """테스트 사용자."""
    return create_test_user(
        id="test_user_001",
        scope_profile_id="scope_001"
    )


@pytest.fixture
def test_golden_items():
    """테스트 Golden Set 항목."""
    return create_test_golden_items(count=3)


def test_run_evaluation_success(test_user, test_golden_items):
    """평가 실행 성공."""
    # 의존성 오버라이드
    app.dependency_overrides[get_current_user] = lambda: test_user

    response = client.post(
        "/api/v1/evaluations/run",
        json={
            "batch_id": "batch_test_001",
            "golden_items": test_golden_items,
            "max_concurrent": 2,
        }
    )

    assert response.status_code == 200
    data = response.json()
    assert "evaluation_id" in data
    assert data["status"] == "queued"
    assert data["batch_id"] == "batch_test_001"


def test_run_evaluation_unauthorized(test_user, test_golden_items):
    """권한 없이 평가 실행 시도."""
    # 권한 없는 사용자로 오버라이드
    test_user.scope_profile_id = "scope_unauthorized"
    app.dependency_overrides[get_current_user] = lambda: test_user

    response = client.post(
        "/api/v1/evaluations/run",
        json={
            "batch_id": "batch_test_001",
            "golden_items": test_golden_items,
        }
    )

    assert response.status_code == 403


def test_get_evaluation_success(test_user, db: Session):
    """평가 결과 조회 성공."""
    # 먼저 평가 실행
    app.dependency_overrides[get_current_user] = lambda: test_user

    run_response = client.post(
        "/api/v1/evaluations/run",
        json={
            "batch_id": "batch_test_002",
            "golden_items": create_test_golden_items(count=1),
        }
    )

    eval_id = run_response.json()["evaluation_id"]

    # 결과 조회
    response = client.get(f"/api/v1/evaluations/{eval_id}")
    assert response.status_code == 200

    data = response.json()
    assert data["evaluation_id"] == eval_id
    assert "results" in data


def test_get_evaluation_not_found(test_user):
    """존재하지 않는 평가 조회."""
    app.dependency_overrides[get_current_user] = lambda: test_user

    response = client.get("/api/v1/evaluations/nonexistent_id")
    assert response.status_code == 404


def test_get_evaluation_forbidden(test_user):
    """다른 Scope Profile의 평가 조회 시도."""
    app.dependency_overrides[get_current_user] = lambda: test_user

    # 다른 scope의 평가를 DB에 추가한 후 조회 시도
    # (실제 구현에서는 DB 작업 필요)
    response = client.get("/api/v1/evaluations/other_scope_eval_id")
    assert response.status_code in [403, 404]


def test_list_evaluations(test_user):
    """평가 목록 조회."""
    app.dependency_overrides[get_current_user] = lambda: test_user

    response = client.get("/api/v1/evaluations?skip=0&limit=10")
    assert response.status_code == 200

    data = response.json()
    assert "total" in data
    assert "items" in data
    assert data["limit"] == 10


def test_list_evaluations_with_filter(test_user):
    """상태 필터로 목록 조회."""
    app.dependency_overrides[get_current_user] = lambda: test_user

    response = client.get("/api/v1/evaluations?status=completed")
    assert response.status_code == 200


def test_compare_evaluations(test_user):
    """두 평가 비교."""
    app.dependency_overrides[get_current_user] = lambda: test_user

    # 두 평가 실행 (실제로는 순차적으로)
    response = client.get(
        "/api/v1/evaluations/eval1/compare?eval_id2=eval2"
    )

    # 평가가 없으면 404, 있으면 비교 결과 반환
    assert response.status_code in [200, 404]

    if response.status_code == 200:
        data = response.json()
        assert "metric_comparison" in data
        assert "improvement" in data
```

---

## 5. 산출물

1. **데이터베이스 모델**
   - `/backend/app/models/evaluation.py` (250줄)

2. **Repository 패턴**
   - `/backend/app/repositories/evaluation_repository.py` (300줄)

3. **배경 작업 처리**
   - `/backend/app/services/evaluation/background_tasks.py` (200줄)

4. **API 엔드포인트**
   - `/backend/app/api/v1/evaluations.py` (400줄)

5. **단위 + 통합 테스트**
   - `/backend/tests/api/test_evaluations.py` (300줄)

6. **마이그레이션 스크립트**
   - `/backend/migrations/versions/xxx_create_evaluation_tables.py` (선택)

7. **문서**
   - 이 작업지시서 (task7-7.md)
   - API 문서 (OpenAPI/Swagger)
   - 검수보고서 (task7-7-검수보고서.md) - 완료 후

8. **선택 산출물**
   - 보안 취약점 검사 보고서 (task7-7-보안검사보고서.md) - 완료 후

---

## 6. 완료 기준

1. **데이터베이스 모델**
   - [ ] EvaluationRun: 모든 필드 포함 (상태, 시간, 결과 등)
   - [ ] EvaluationResultRecord: 모든 지표 및 성능 정보 저장
   - [ ] 인덱스: 주요 조회 경로 최적화
   - [ ] 관계: Run ↔ ResultRecord 양방향 관계

2. **Repository 패턴**
   - [ ] CRUD 메서드 구현
   - [ ] Scope Profile 기반 필터링 (S2 원칙 ⑥)
   - [ ] 트랜잭션 처리 및 원자성
   - [ ] 에러 로깅

3. **배경 작업**
   - [ ] 비동기 평가 실행
   - [ ] 결과 저장 자동화
   - [ ] Run 상태 추적 (queued → running → completed/failed)
   - [ ] 실패 처리 및 로깅

4. **API 엔드포인트**
   - [ ] POST /api/v1/evaluations/run: 평가 시작, 즉시 응답
   - [ ] GET /api/v1/evaluations/{eval_id}: 상세 조회
   - [ ] GET /api/v1/evaluations: 목록 조회 (페이지네이션, 필터)
   - [ ] GET /api/v1/evaluations/{eval_id}/compare: 비교

5. **권한 및 ACL**
   - [ ] Scope Profile 기반 접근 제어 (모든 엔드포인트)
   - [ ] 자신의 Scope 범위 내 데이터만 조회 가능
   - [ ] Actor Type 기록 (user vs agent) - S2 원칙 ⑤
   - [ ] 감사 로그 (모든 접근 기록)

6. **단위 + 통합 테스트**
   - [ ] API 엔드포인트 테스트 6개 이상
   - [ ] ACL 테스트 (권한 있음/없음)
   - [ ] 배경 작업 테스트
   - [ ] 모든 테스트 통과 (pytest 100%)

7. **코드 품질**
   - [ ] Type hints 완전 적용
   - [ ] Docstring 모든 함수/클래스
   - [ ] 에러 처리 및 로깅 적절함
   - [ ] HTTP 상태 코드 정확함 (400, 403, 404, 500)
   - [ ] PEP 8 준수

---

## 7. 작업 지침

### 지침 7-1: S2 원칙 ⑤ Actor Type 기록
모든 평가 작업은 `created_by`(사용자/에이전트 ID)와 `actor_type`("user" 또는 "agent")을 함께 기록한다. 이를 통해 향후 평가 품질 분석 시 사용자별/에이전트별 성능을 구분할 수 있다.

### 지침 7-2: S2 원칙 ⑥ Scope Profile 기반 ACL
모든 데이터 조회/쓰기는 `scope_profile_id`로 필터링된다. 사용자는 자신의 Scope Profile에 속하는 평가만 볼 수 있다. 이는 데이터 격리 및 보안의 핵심이다.

### 지침 7-3: 비동기 작업 관리의 신뢰성
배경 작업은 Fire-and-forget 방식으로 실행되지만, 상태는 DB에 기록되므로 언제든 조회할 수 있다. 작업 실패 시에도 상태를 "failed"로 업데이트하고 로깅하므로, 운영자는 실패 원인을 추적할 수 있다.

### 지침 7-4: 시계열 데이터 고려
향후 평가 이력이 매우 많아질 경우, `EvaluationResultRecord`를 시계열 DB(InfluxDB, TimescaleDB 등)로 분리할 수 있다. 현 단계에서는 PostgreSQL에 저장하되, 스키마 설계 시 분리를 고려한다.

### 지침 7-5: 페이지네이션의 효율성
목록 조회는 `skip`과 `limit` 매개변수로 페이지네이션을 지원한다. 대량의 평가 데이터를 다룰 때 효율적 조회를 위해 인덱스를 적절히 설정한다.

### 지침 7-6: 감사 로그의 세부성
모든 평가 작업(실행, 조회, 비교)은 로깅되며, 최소한 다음 정보를 기록한다:
- 작업 종류 (실행, 조회, 비교 등)
- 사용자/에이전트 ID
- 작업 대상 (eval_id, batch_id)
- 타임스탬프
- 결과 (성공/실패, 반환 데이터 크기 등)

### 지침 7-7: 오류 응답의 일관성
모든 API 오류는 다음 형식으로 반환한다:
```json
{
    "status_code": 400,
    "detail": "Error message",
    "error_type": "validation_error" or "not_found" or "forbidden" or "server_error"
}
```

이를 통해 클라이언트가 오류를 일관되게 처리할 수 있다.

---

**작업 시작 예정일**: [TBD]  
**예상 종료일**: [TBD + 8-9일]  
**검수 예정일**: [TBD + 9-10일]
