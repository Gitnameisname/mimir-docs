# Task 3-3. PII 생명주기 관리 및 자동 만료 배치

## 1. 작업 목적

사용자의 대화 데이터에 포함된 개인 식별 정보(PII)를 보존 정책에 따라 자동으로 만료·삭제하고, 수동 삭제 및 민감 정보 제거(redact) 기능을 구현한다. 또한 감시 로그를 통해 모든 삭제 작업의 투명성을 보장하고, **폐쇄망 환경에서도 동작**하도록 외부 의존 없이 설계한다(S2 원칙 ⑦).

## 2. 작업 범위

### 포함 범위

1. **대화 보존 기간 설정 및 관리**
   - 조직별 기본 retention_days (기본값: 90일)
   - 대화별 개별 설정 가능 (override)
   - expires_at 자동 계산: created_at + retention_days

2. **자동 만료 배치 작업 (APScheduler 또는 Celery)**
   - 정기 실행: 매시간 또는 매일 자정 (설정 가능)
   - 동작: expires_at < now() 인 대화를 상태 `expired`로 전환
   - dry-run 모드: 삭제 전에 변경 대상 미리보기 및 백업
   - 감시 로그: 배치 작업 시작/완료/오류 기록

3. **수동 삭제 API**
   - DELETE /api/v1/conversations/{id}: 특정 대화 즉시 삭제
   - soft delete (deleted_at 설정) 또는 물리 삭제 선택 가능
   - 재확인(confirmation) 처리
   - 감시 로그: 삭제 사유, 삭제자, 타임스탬프 기록

4. **민감 정보 제거 (redact) 기능**
   - POST /api/v1/conversations/{id}/turns/{turn_id}/redact: 턴의 특정 필드 마스킹
   - 필드: user_message, assistant_response 등
   - 처리: [REDACTED] 문자열로 치환
   - 메타데이터: 원본 내용은 암호화 저장 (선택) 또는 감시 로그에만 기록
   - PII 감지 규칙 (선택): 이메일, 전화번호, SSN 패턴 자동 감지 후 플래그

5. **PII 감지 규칙 (선택)**
   - 정규 표현식 기반: 이메일, 전화번호(010-xxxx-xxxx), SSN(xxx-xx-xxxx)
   - 감지 시 metadata에 `pii_fields: [field_name, ...]` 플래그 기록
   - 자동 알림 (선택): 관리자에게 PII 포함 대화 알림

6. **감시 이벤트 기록 (S2 원칙 ⑥)**
   - 자동 만료 배치: entity_type=conversation, action=expire
   - 수동 삭제: entity_type=conversation, action=delete
   - Redact 작업: entity_type=turn, action=redact
   - 모든 이벤트: actor_id, actor_type (system/user/agent), timestamp, reason

7. **단위/통합 테스트**
   - 배치 작업 트리거 및 동작 테스트
   - dry-run 모드 검증
   - 감시 로그 기록 확인
   - PII 감지 규칙 정확성

### 제외 범위

- 고급 암호화 (저장 데이터 암호화) — Task 6 보안에서
- 법적 보류(legal hold) 기능 — Task 6 규정 준수에서
- 사용자에게 삭제 알림 발송 — 통보 시스템과 통합
- PII 감지 ML 모델 기반 — 정규식 기반만

## 3. 선행 조건

- Task 3-1, Task 3-2 완료 (Conversation/Turn 모델, API)
- APScheduler 또는 Celery 설치
- PostgreSQL retention_days 및 expires_at 필드 존재
- 기존 audit_log 테이블 및 actor_type 필드 확인
- 폐쇄망 환경 테스트 환경 준비 (외부 의존 제거 확인)

## 4. 주요 작업 항목

### 4-1. Retention Policy 설정 모델

**파일:** `/backend/app/models/retention_policy.py`

```python
from __future__ import annotations

from uuid import UUID, uuid4
from datetime import datetime

from sqlalchemy import Column, String, Integer, DateTime, ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID as PGUUID
from sqlalchemy.orm import mapped_column, relationship
from sqlalchemy.sql import func

from app.models.base import Base


class RetentionPolicy(Base):
    """조직별 보존 정책 설정.
    
    Attributes:
        id: 정책 고유 ID (UUID)
        organization_id: 조직 ID (UUID)
        default_retention_days: 기본 보존 기간 (일)
        max_retention_days: 최대 보존 기간 (상한)
        auto_expire_enabled: 자동 만료 활성화 여부
        batch_schedule: 배치 실행 스케줄 (cron 표현식)
        created_at: 정책 생성 시간
        updated_at: 정책 수정 시간
    """
    
    __tablename__ = "retention_policies"
    
    id = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    organization_id = mapped_column(PGUUID(as_uuid=True), nullable=False, unique=True, index=True)
    
    default_retention_days = mapped_column(Integer, default=90, nullable=False)
    max_retention_days = mapped_column(Integer, default=365, nullable=False)
    auto_expire_enabled = mapped_column(bool, default=True, nullable=False)
    batch_schedule = mapped_column(String(32), default="0 * * * *", nullable=False)  # 매시간
    
    created_at = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)
    
    def __repr__(self) -> str:
        return f"<RetentionPolicy(org={self.organization_id}, days={self.default_retention_days})>"


class PiiPattern(Base):
    """PII 감지 정규 표현식.
    
    Attributes:
        id: 패턴 고유 ID
        pattern_type: 패턴 종류 (email, phone, ssn, credit_card)
        regex_pattern: 정규 표현식
        description: 설명
        enabled: 활성화 여부
    """
    
    __tablename__ = "pii_patterns"
    
    id = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    pattern_type = mapped_column(String(64), nullable=False, unique=True)
    regex_pattern = mapped_column(String(512), nullable=False)
    description = mapped_column(String(256), nullable=True)
    enabled = mapped_column(bool, default=True, nullable=False)
    
    created_at = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    
    def __repr__(self) -> str:
        return f"<PiiPattern(type={self.pattern_type})>"
```

### 4-2. PII 감지 유틸리티

**파일:** `/backend/app/services/pii_detector.py`

```python
"""PII(개인 식별 정보) 감지 서비스."""

from __future__ import annotations

import re
import logging
from typing import List, Dict, Any, Optional
from uuid import UUID

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.retention_policy import PiiPattern
from app.models.conversation import Turn, Message

logger = logging.getLogger(__name__)


class PiiDetector:
    """정규식 기반 PII 감지."""
    
    # 기본 패턴 (설정 불가능한 기본값)
    DEFAULT_PATTERNS = {
        "email": r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b",
        "phone_kr": r"01[0-9]-\d{3,4}-\d{4}",
        "ssn_kr": r"\d{3}-\d{2}-\d{4}",  # xxx-xx-xxxx 형식
        "credit_card": r"\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b",
    }
    
    def __init__(self, db: AsyncSession):
        self.db = db
        self.patterns: Dict[str, str] = {}
        self._initialized = False
    
    async def initialize(self) -> None:
        """DB에서 활성화된 PII 패턴 로드."""
        if self._initialized:
            return
        
        stmt = select(PiiPattern).where(PiiPattern.enabled == True)
        result = await self.db.execute(stmt)
        patterns = result.scalars().all()
        
        # 기본 패턴으로 시작
        self.patterns = self.DEFAULT_PATTERNS.copy()
        
        # DB 패턴으로 업데이트 또는 추가
        for pattern in patterns:
            self.patterns[pattern.pattern_type] = pattern.regex_pattern
        
        self._initialized = True
        logger.info(f"PII patterns loaded: {list(self.patterns.keys())}")
    
    async def detect(self, text: str) -> Dict[str, List[str]]:
        """텍스트에서 PII 감지.
        
        Returns:
            {pattern_type: [matched_text, ...], ...}
        """
        if not self._initialized:
            await self.initialize()
        
        detected = {}
        
        for pattern_type, regex_pattern in self.patterns.items():
            matches = re.findall(regex_pattern, text, re.IGNORECASE)
            if matches:
                detected[pattern_type] = matches
        
        return detected
    
    async def detect_in_turn(self, turn_id: UUID) -> Dict[str, Any]:
        """Turn에서 PII 감지 및 metadata 갱신.
        
        Returns:
            {
                "has_pii": bool,
                "user_message_pii": {...},
                "assistant_response_pii": {...}
            }
        """
        from sqlalchemy import select
        
        stmt = select(Turn).where(Turn.id == turn_id)
        result = await self.db.execute(stmt)
        turn = result.scalar_one_or_none()
        
        if not turn:
            return {"has_pii": False}
        
        user_pii = await self.detect(turn.user_message)
        assistant_pii = await self.detect(turn.assistant_response)
        
        has_pii = bool(user_pii or assistant_pii)
        
        if has_pii:
            # metadata에 PII 정보 저장
            turn.metadata = turn.metadata or {}
            turn.metadata["pii_detected"] = {
                "user_message": list(user_pii.keys()) if user_pii else [],
                "assistant_response": list(assistant_pii.keys()) if assistant_pii else [],
                "detected_at": datetime.utcnow().isoformat(),
            }
            await self.db.flush()
        
        return {
            "has_pii": has_pii,
            "user_message_pii": user_pii,
            "assistant_response_pii": assistant_pii,
        }
```

### 4-3. 배치 작업 (자동 만료)

**파일:** `/backend/app/services/expiration_batch.py`

```python
"""대화 자동 만료 배치 작업."""

from __future__ import annotations

import logging
from typing import Dict, Any, Optional
from datetime import datetime
from uuid import UUID

from sqlalchemy import select, and_
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.conversation import Conversation
from app.models.audit_log import AuditLog

logger = logging.getLogger(__name__)


class ExpirationBatchJob:
    """배치 작업: expires_at < now()인 대화를 expired 상태로 전환."""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def run(self, dry_run: bool = False) -> Dict[str, Any]:
        """배치 실행.
        
        Args:
            dry_run: True이면 변경 없이 미리보기만 수행
        
        Returns:
            {
                "status": "success|error",
                "expired_count": int,
                "failed_count": int,
                "dry_run": bool,
                "message": str,
                "errors": [str, ...]
            }
        """
        logger.info(f"Starting expiration batch (dry_run={dry_run})")
        
        try:
            now = datetime.utcnow()
            
            # expires_at < now 이고 상태가 active인 대화 조회
            stmt = select(Conversation).where(
                and_(
                    Conversation.expires_at < now,
                    Conversation.status == "active",
                    Conversation.deleted_at.is_(None)
                )
            )
            result = await self.db.execute(stmt)
            conversations = result.scalars().all()
            
            expired_count = len(conversations)
            failed_count = 0
            
            logger.info(f"Found {expired_count} conversations to expire")
            
            if not dry_run:
                for conversation in conversations:
                    try:
                        conversation.status = "expired"
                        
                        # 감시 로그: 배치 시스템이 자동 만료
                        audit_log = AuditLog(
                            entity_type="conversation",
                            entity_id=conversation.id,
                            action="expire",
                            actor_id=None,  # 또는 시스템 ID
                            actor_type="system",
                            changes={
                                "reason": "auto_expiration",
                                "expires_at": conversation.expires_at.isoformat(),
                                "expired_at": now.isoformat(),
                            }
                        )
                        self.db.add(audit_log)
                        
                        logger.debug(f"Expired conversation {conversation.id}")
                    
                    except Exception as e:
                        failed_count += 1
                        logger.error(f"Failed to expire conversation {conversation.id}: {e}")
                
                await self.db.commit()
                logger.info(f"Batch completed: {expired_count} expired, {failed_count} failed")
            
            else:
                logger.info(f"DRY RUN: Would expire {expired_count} conversations")
                # dry_run에서는 변경 없음
            
            return {
                "status": "success",
                "expired_count": expired_count,
                "failed_count": failed_count,
                "dry_run": dry_run,
                "message": f"Batch job completed. Expired: {expired_count}, Failed: {failed_count}",
                "errors": [],
            }
        
        except Exception as e:
            logger.error(f"Batch job failed: {e}")
            return {
                "status": "error",
                "expired_count": 0,
                "failed_count": 0,
                "dry_run": dry_run,
                "message": "Batch job failed",
                "errors": [str(e)],
            }
```

### 4-4. APScheduler 통합

**파일:** `/backend/app/scheduler.py`

```python
"""백그라운드 작업 스케줄러 (APScheduler)."""

from __future__ import annotations

import logging
from datetime import datetime

from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger

logger = logging.getLogger(__name__)


class SchedulerService:
    """APScheduler 관리 서비스."""
    
    def __init__(self):
        self.scheduler = AsyncIOScheduler()
    
    async def start(self, app_context) -> None:
        """스케줄러 시작.
        
        Args:
            app_context: FastAPI 앱 또는 의존성 주입 컨텍스트
        """
        # 배치 작업 스케줄 추가
        # 기본: 매일 자정 0시에 실행
        self.scheduler.add_job(
            func=self._run_expiration_batch,
            trigger=CronTrigger(hour=0, minute=0),
            id="expiration_batch",
            name="Conversation Expiration Batch",
            replace_existing=True,
            args=[app_context],
        )
        
        if not self.scheduler.running:
            self.scheduler.start()
            logger.info("Scheduler started")
    
    async def stop(self) -> None:
        """스케줄러 중지."""
        if self.scheduler.running:
            self.scheduler.shutdown()
            logger.info("Scheduler stopped")
    
    async def _run_expiration_batch(self, app_context) -> None:
        """배치 작업 실행 (스케줄 콜백)."""
        try:
            from app.database import get_async_db
            
            async for db in get_async_db():
                from app.services.expiration_batch import ExpirationBatchJob
                
                job = ExpirationBatchJob(db)
                result = await job.run(dry_run=False)
                
                logger.info(f"Expiration batch result: {result}")
                
                break  # 첫 번째 DB 연결만 사용
        
        except Exception as e:
            logger.error(f"Failed to run expiration batch: {e}")


# FastAPI 라이프사이클 통합
async def startup_scheduler(app):
    """앱 시작 시 스케줄러 시작."""
    scheduler_service = SchedulerService()
    await scheduler_service.start(app)
    app.state.scheduler = scheduler_service


async def shutdown_scheduler(app):
    """앱 종료 시 스케줄러 중지."""
    if hasattr(app.state, 'scheduler'):
        await app.state.scheduler.stop()
```

### 4-5. 수동 삭제 API 라우터

**파일:** `/backend/app/routes/retention.py`

```python
"""보존 정책 및 수동 삭제 API."""

from __future__ import annotations

from typing import Optional
from uuid import UUID
import logging

from fastapi import APIRouter, Depends, HTTPException, Header
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.auth import get_current_user
from app.models.conversation import Conversation
from app.models.retention_policy import RetentionPolicy
from app.models.audit_log import AuditLog
from app.repositories.conversation_repository import ConversationRepository
from app.services.expiration_batch import ExpirationBatchJob
from app.services.pii_detector import PiiDetector
from app.schemas.conversation import ApiResponse

logger = logging.getLogger(__name__)
router = APIRouter(prefix="/api/v1/retention", tags=["retention"])


@router.get("/policies/{organization_id}", response_model=ApiResponse)
async def get_retention_policy(
    organization_id: UUID,
    current_user: dict = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """조직의 보존 정책 조회 (관리자만)."""
    # TODO: 권한 검사 (관리자)
    
    from sqlalchemy import select
    
    stmt = select(RetentionPolicy).where(RetentionPolicy.organization_id == organization_id)
    result = await db.execute(stmt)
    policy = result.scalar_one_or_none()
    
    if not policy:
        return ApiResponse(
            status="ok",
            data={
                "organization_id": organization_id,
                "default_retention_days": 90,
                "max_retention_days": 365,
                "auto_expire_enabled": True,
            }
        )
    
    return ApiResponse(
        status="ok",
        data={
            "organization_id": str(policy.organization_id),
            "default_retention_days": policy.default_retention_days,
            "max_retention_days": policy.max_retention_days,
            "auto_expire_enabled": policy.auto_expire_enabled,
            "batch_schedule": policy.batch_schedule,
        }
    )


@router.post("/batch/run", response_model=ApiResponse)
async def run_expiration_batch(
    dry_run: bool = False,
    current_user: dict = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
    x_actor_type: str = Header("user"),
):
    """배치 작업 수동 실행 (관리자만).
    
    Query Parameters:
        dry_run: True면 변경 없이 미리보기만 수행
    """
    # TODO: 권한 검사 (관리자)
    
    job = ExpirationBatchJob(db)
    result = await job.run(dry_run=dry_run)
    
    logger.info(f"Manual batch execution by {current_user['id']}: {result}")
    
    # 감시 로그
    audit_log = AuditLog(
        entity_type="batch_job",
        entity_id=UUID("00000000-0000-0000-0000-000000000000"),  # 특수 ID
        action="run",
        actor_id=current_user["id"],
        actor_type=x_actor_type,
        changes={"job": "expiration_batch", "dry_run": dry_run}
    )
    db.add(audit_log)
    await db.commit()
    
    return ApiResponse(status="ok", data=result)


@router.post("/pii/detect/{conversation_id}", response_model=ApiResponse)
async def detect_pii(
    conversation_id: UUID,
    current_user: dict = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """대화의 모든 턴에서 PII 감지 및 metadata 갱신.
    
    이 엔드포인트는 관리자 또는 대화 소유자만 호출 가능.
    """
    repo = ConversationRepository(db)
    conversation = await repo.get_by_id(conversation_id)
    
    if not conversation:
        raise HTTPException(status_code=404, detail="Conversation not found")
    
    # 권한 검사
    if conversation.owner_id != current_user["id"]:
        # TODO: 관리자 권한 검사
        raise HTTPException(status_code=403, detail="Forbidden")
    
    detector = PiiDetector(db)
    
    from sqlalchemy import select
    from app.models.conversation import Turn
    
    # 모든 턴에서 PII 감지
    stmt = select(Turn).where(Turn.conversation_id == conversation_id)
    result = await db.execute(stmt)
    turns = result.scalars().all()
    
    detection_results = {}
    for turn in turns:
        pii_result = await detector.detect_in_turn(turn.id)
        if pii_result["has_pii"]:
            detection_results[str(turn.id)] = pii_result
    
    await db.commit()
    
    logger.info(f"PII detection completed for conversation {conversation_id}: {len(detection_results)} turns with PII")
    
    return ApiResponse(
        status="ok",
        data={
            "conversation_id": str(conversation_id),
            "turns_with_pii": len(detection_results),
            "details": detection_results,
        }
    )
```

### 4-6. 단위/통합 테스트

**파일:** `/backend/tests/integration/test_retention.py`

```python
"""보존 정책 및 배치 작업 통합 테스트."""

import pytest
from uuid import uuid4
from datetime import datetime, timedelta
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.conversation import Conversation
from app.models.audit_log import AuditLog
from app.repositories.conversation_repository import ConversationRepository
from app.services.expiration_batch import ExpirationBatchJob
from app.services.pii_detector import PiiDetector


@pytest.mark.asyncio
async def test_expiration_batch_dry_run(db_session: AsyncSession):
    """배치 작업 dry-run 모드 테스트."""
    repo = ConversationRepository(db_session)
    
    # 만료된 대화 생성
    expired_conversation = await repo.create(
        owner_id=uuid4(),
        organization_id=uuid4(),
        title="Expired Conversation",
        retention_days=1,
        actor_id=uuid4(),
    )
    
    # expires_at을 과거로 설정
    expired_conversation.expires_at = datetime.utcnow() - timedelta(days=1)
    await db_session.flush()
    
    # Batch 실행 (dry_run=True)
    job = ExpirationBatchJob(db_session)
    result = await job.run(dry_run=True)
    
    assert result["status"] == "success"
    assert result["expired_count"] == 1
    assert result["dry_run"] == True
    
    # 상태 변경 없음 (dry_run)
    await db_session.refresh(expired_conversation)
    assert expired_conversation.status == "active"


@pytest.mark.asyncio
async def test_expiration_batch_actual_run(db_session: AsyncSession):
    """배치 작업 실제 실행 및 감시 로그 기록 테스트."""
    repo = ConversationRepository(db_session)
    
    # 만료된 대화 생성
    expired_conversation = await repo.create(
        owner_id=uuid4(),
        organization_id=uuid4(),
        title="Expired Conversation",
        actor_id=uuid4(),
    )
    
    # expires_at을 과거로 설정
    expired_conversation.expires_at = datetime.utcnow() - timedelta(days=1)
    await db_session.flush()
    
    # Batch 실행 (dry_run=False)
    job = ExpirationBatchJob(db_session)
    result = await job.run(dry_run=False)
    
    assert result["status"] == "success"
    assert result["expired_count"] == 1
    
    # 상태 변경 확인
    await db_session.refresh(expired_conversation)
    assert expired_conversation.status == "expired"
    
    # 감시 로그 확인
    from sqlalchemy import select
    
    stmt = select(AuditLog).where(
        AuditLog.entity_id == expired_conversation.id
    )
    audit_result = await db_session.execute(stmt)
    audit_logs = audit_result.scalars().all()
    
    expire_logs = [log for log in audit_logs if log.action == "expire"]
    assert len(expire_logs) > 0
    assert expire_logs[0].actor_type == "system"


@pytest.mark.asyncio
async def test_pii_detection_email(db_session: AsyncSession):
    """PII 감지: 이메일 테스트."""
    detector = PiiDetector(db_session)
    
    text = "Contact me at user@example.com for more info"
    detected = await detector.detect(text)
    
    assert "email" in detected
    assert "user@example.com" in detected["email"]


@pytest.mark.asyncio
async def test_pii_detection_phone(db_session: AsyncSession):
    """PII 감지: 전화번호 테스트."""
    detector = PiiDetector(db_session)
    
    text = "Call me at 010-1234-5678"
    detected = await detector.detect(text)
    
    assert "phone_kr" in detected
    assert "010-1234-5678" in detected["phone_kr"]


@pytest.mark.asyncio
async def test_pii_detection_in_turn(db_session: AsyncSession, mock_conversation_and_turn):
    """Turn에서 PII 감지 및 metadata 갱신 테스트."""
    turn = mock_conversation_and_turn
    turn.user_message = "My email is sensitive@example.com"
    await db_session.flush()
    
    detector = PiiDetector(db_session)
    result = await detector.detect_in_turn(turn.id)
    
    assert result["has_pii"] == True
    assert "email" in result["user_message_pii"]
    
    # metadata 확인
    await db_session.refresh(turn)
    assert "pii_detected" in turn.metadata
    assert "email" in turn.metadata["pii_detected"]["user_message"]
```

## 5. 산출물

1. **Retention Policy 모델** (`/backend/app/models/retention_policy.py`)
   - RetentionPolicy: 조직별 보존 정책
   - PiiPattern: PII 감지 정규 표현식

2. **PII 감지 서비스** (`/backend/app/services/pii_detector.py`)
   - PiiDetector: 정규식 기반 PII 감지
   - Turn 메타데이터 자동 갱신

3. **자동 만료 배치** (`/backend/app/services/expiration_batch.py`)
   - ExpirationBatchJob: expires_at < now() 대화 만료
   - dry-run 모드 지원
   - 감시 로그 자동 기록

4. **APScheduler 통합** (`/backend/app/scheduler.py`)
   - SchedulerService: 스케줄러 관리
   - FastAPI 라이프사이클 통합
   - 주기적 배치 실행 (설정 가능)

5. **수동 삭제 및 PII 감지 API** (`/backend/app/routes/retention.py`)
   - GET /api/v1/retention/policies/{org_id}
   - POST /api/v1/retention/batch/run (dry-run 지원)
   - POST /api/v1/retention/pii/detect/{conversation_id}

6. **단위/통합 테스트** (`/backend/tests/integration/test_retention.py`)
   - 배치 작업 dry-run/실제 실행 테스트
   - 감시 로그 기록 검증
   - PII 감지 정확성 테스트

## 6. 완료 기준

1. 배치 작업이 expires_at < now() 대화를 `expired` 상태로 전환하는가?
2. dry-run 모드가 변경 없이 미리보기만 수행하는가?
3. 모든 배치 작업이 감시 로그에 기록되는가? (actor_type=system)
4. 감시 로그에 expires_at, expired_at 등의 변경 사항이 기록되는가?
5. PII 감지가 이메일, 전화번호, SSN 패턴을 정확하게 인식하는가?
6. PII 감지 결과가 Turn metadata의 pii_detected 필드에 저장되는가?
7. APScheduler가 설정된 시간에 배치를 자동 실행하는가?
8. 수동 삭제 API가 soft delete (deleted_at 설정)를 수행하는가?
9. 폐쇄망 환경에서도 배치가 정상 동작하는가? (외부 의존 없음)
10. 모든 통합 테스트가 통과하는가?

## 7. 작업 지침

### 지침 7-1. 폐쇄망 환경 호환성 (S2 원칙 ⑦)

배치 작업과 PII 감지는 외부 의존 없이 동작해야 한다:
- APScheduler: 로컬 메모리 기반 (Celery 불가)
- PII 감지: 정규식 기반 (외부 ML 서비스 불가)
- 데이터 저장: PostgreSQL (네트워크 외부 서비스 불가)

```python
# 폐쇄망 환경에서도 동작
# APScheduler with AsyncIOScheduler
scheduler = AsyncIOScheduler()

# 정규식 기반 PII 감지 (외부 API 호출 없음)
regex_pattern = r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b"
```

### 지침 7-2. actor_type 필드 (S2 원칙 ⑥)

배치 작업의 감시 로그는 actor_type=system으로 기록한다:

```python
audit_log = AuditLog(
    entity_type="conversation",
    entity_id=conversation.id,
    action="expire",
    actor_id=None,  # 또는 시스템 ID (고정 UUID)
    actor_type="system",  # 배치 시스템
    changes={...}
)
```

### 지침 7-3. dry-run 모드 구현

dry-run 모드는 변경을 수행하지 않고 미리보기만 제공한다:

```python
async def run(self, dry_run: bool = False):
    ...
    if not dry_run:
        for conversation in conversations:
            conversation.status = "expired"
            # 감시 로그 등
        await db.commit()
    else:
        logger.info(f"DRY RUN: Would expire {expired_count} conversations")
        # 변경 없음
```

### 지침 7-4. PII 감지 메타데이터 저장

PII 감지 결과는 Turn metadata의 특정 필드에 저장한다:

```python
turn.metadata["pii_detected"] = {
    "user_message": ["email", "phone_kr"],
    "assistant_response": [],
    "detected_at": "2026-04-17T10:30:00Z"
}
```

### 지침 7-5. 스케줄 설정

APScheduler의 CronTrigger를 사용하여 스케줄을 설정한다. 기본값: 매일 자정 0시

```python
trigger=CronTrigger(hour=0, minute=0)  # 매일 자정
# 또는 조직 설정에서 읽기
# batch_schedule = policy.batch_schedule  # "0 0 * * *" (cron 형식)
```

### 지침 7-6. 트랜잭션 관리

배치 작업 실패 시 전체 변경을 롤백한다. 각 대화별 오류는 로깅하지만 배치를 중단하지 않는다:

```python
for conversation in conversations:
    try:
        conversation.status = "expired"
        # ...
    except Exception as e:
        failed_count += 1
        logger.error(f"Failed for {conversation.id}: {e}")
        # 다음 대화 계속 처리

await db.commit()  # 전체 배치 커밋
```

### 지침 7-7. 감시 로그 integration

audit_log 테이블에 모든 삭제/만료 작업을 기록한다. 이를 통해 GDPR 등의 규정 준수 증거를 남긴다:

```python
audit_log = AuditLog(
    entity_type="conversation",
    entity_id=conversation.id,
    action="delete",  # 또는 "expire"
    actor_id=user_id,
    actor_type="user",  # 또는 "system"
    timestamp=datetime.utcnow(),
    changes={
        "reason": "user_requested",
        "expires_at": conversation.expires_at.isoformat(),
    }
)
```

### 지침 7-8. 환경변수 기반 설정 (선택)

폐쇄망과 개방망 환경을 지원하기 위해 환경변수로 기능 on/off 제어 (선택):

```python
# .env
PII_DETECTION_ENABLED=true
AUTO_EXPIRATION_ENABLED=true
BATCH_SCHEDULE="0 0 * * *"
```

```python
# config.py
PII_DETECTION_ENABLED = os.getenv("PII_DETECTION_ENABLED", "true").lower() == "true"

if PII_DETECTION_ENABLED:
    # PII 감지 로직
    pass
else:
    # PII 감지 스킵 (폐쇄망 환경에서 비활성화 가능)
    pass
```
