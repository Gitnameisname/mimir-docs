# Task 5-3. MCP Tasks 통합 (비동기 승인 플로우)

## 1. 작업 목적

Task 5-1, 5-2에서 구축한 Draft 제안(proposed)과 승인(approve) 플로우에 **MCP Tasks** (Anthropic MCP의 experimental 기능)를 통합하여 **비동기 인간 승인을 자동화**한다. 에이전트가 Draft를 proposed 상태로 생성하면, 자동으로 MCP Task가 생성되어 인간 검토자에게 "Draft 검수 필요" 알림을 보낸다. 인간이 Task를 완료(MCP Task completed) 또는 실패(MCP Task failed)로 처리하면, 자동으로 Draft 상태가 approved 또는 rejected로 동기화된다. 또한 **S2 원칙 ⑦ (폐쇄망 환경 지원)**을 준수하여, MCP Tasks가 없어도 (또는 외부 의존을 끌 때도) 제안 큐만으로 서비스가 degrade하지만 실패하지 않도록 한다.

## 2. 작업 범위

### 포함 범위

1. **TaskProposalBridge 클래스**
   - Draft proposed 생성 시 MCP Task 자동 생성
   - Draft 상태 변경 시 MCP Task 상태 동기화
   - MCP Task 상태 변화 감시 (polling 또는 webhook)

2. **MCP Task 생성**
   - Task 생성: POST /mcp/tasks/create
   - 입력: title (Draft 제목), description (Draft 요약), state (input_required)
   - 응답: task_id, state (input_required)
   - 메타데이터: draft_id, document_id, agent_id

3. **MCP Task 상태 동기화**
   - Task completed → Draft approved
   - Task failed → Draft rejected
   - Task state 변화 감시 (polling 또는 webhook)

4. **Task 상태 감시 (Polling)**
   - 주기적으로 MCP Task 상태 조회: GET /mcp/tasks/{task_id}
   - 응답: task_id, state, progressPercentage
   - Draft 상태에 반영

5. **Fallback 메커니즘 (S2 원칙 ⑦)**
   - MCP Tasks 미지원 시에도 Draft 제안 큐만으로 동작
   - 환경변수: MCP_TASKS_ENABLED (on/off)
   - off 시: Task 생성 스킵, 감시 로그에 fallback 기록

6. **감시 로그 (S2 원칙 ⑤)**
   - 이벤트: task_create, task_completed, task_failed, task_state_change
   - actor_type: "system" (MCP Task 동기화는 시스템 작업)
   - 기록 항목: task_id, draft_id, state, progressPercentage

7. **TaskProposalBridge 통합**
   - DraftProposalService와 통합
   - Draft 생성 → MCP Task 생성
   - MCP Task 상태 변화 감시 시작

8. **단위/통합 테스트**
   - MCP Task 생성 및 동기화
   - Fallback 동작 (MCP Tasks disabled)
   - 상태 동기화 검증
   - 감시 로그 기록

### 제외 범위

- Draft 생성 (Task 5-1)
- Draft 승인/거부 API (Task 5-2)
- MCP Server 초기화 (Task 4-1)
- MCP 도구 구현 (Task 4-2)

## 3. 선행 조건

- Task 5-1, 5-2 완료 (Draft proposed, approve API)
- MCP Tasks (experimental) 클라이언트 라이브러리 설치
- 기존 MCP Server 통신 인프라 (Task 4-1)
- 비동기 작업 스케줄러 (APScheduler 또는 Celery)

## 4. 주요 작업 항목

### 4-1. MCP Tasks 클라이언트 래퍼

**파일:** `/backend/app/mcp/tasks_client.py` (신규)

```python
"""MCP Tasks 클라이언트 래퍼 (S2-FG5.3)."""

from __future__ import annotations

import logging
from typing import Optional, Dict, Any
from uuid import UUID
import os
import httpx

logger = logging.getLogger(__name__)


class MCPTasksClient:
    """MCP Tasks 클라이언트.
    
    MCP Tasks (experimental)와 통신하는 래퍼.
    S2 원칙 ⑦: MCP_TASKS_ENABLED=off 시에도 graceful degrade.
    """
    
    def __init__(self):
        """
        MCP Tasks 클라이언트 초기화.
        
        환경변수:
        - MCP_TASKS_ENABLED: true/false (기본: false)
        - MCP_TASKS_BASE_URL: MCP Tasks 서버 URL
        """
        self.enabled = os.getenv("MCP_TASKS_ENABLED", "false").lower() == "true"
        self.base_url = os.getenv("MCP_TASKS_BASE_URL", "http://localhost:8000")
        self.timeout = 30  # 초
        
        logger.info(f"MCPTasksClient initialized: enabled={self.enabled}")
    
    async def create_task(
        self,
        title: str,
        description: str,
        metadata: Optional[Dict[str, Any]] = None,
    ) -> Optional[str]:
        """MCP Task 생성.
        
        Args:
            title: Task 제목
            description: Task 설명
            metadata: 메타데이터 (draft_id, document_id, agent_id 등)
        
        Returns:
            task_id (성공), None (실패 또는 disabled)
        """
        if not self.enabled:
            logger.debug("MCP Tasks disabled, skipping task creation")
            return None
        
        try:
            async with httpx.AsyncClient(timeout=self.timeout) as client:
                response = await client.post(
                    f"{self.base_url}/mcp/tasks/create",
                    json={
                        "title": title,
                        "description": description,
                        "state": "input_required",
                        "metadata": metadata or {},
                    },
                )
                
                if response.status_code == 200:
                    data = response.json()
                    task_id = data.get("task_id")
                    logger.info(f"MCP Task created: task_id={task_id}")
                    return task_id
                else:
                    logger.error(
                        f"Failed to create MCP Task: "
                        f"status={response.status_code}, "
                        f"body={response.text}"
                    )
                    return None
        
        except Exception as e:
            logger.error(f"Exception creating MCP Task: {str(e)}")
            return None
    
    async def get_task_state(self, task_id: str) -> Optional[Dict[str, Any]]:
        """MCP Task 상태 조회.
        
        Args:
            task_id: Task ID
        
        Returns:
            {task_id, state, progressPercentage, ...} (성공),
            None (실패 또는 disabled)
        """
        if not self.enabled:
            return None
        
        try:
            async with httpx.AsyncClient(timeout=self.timeout) as client:
                response = await client.get(
                    f"{self.base_url}/mcp/tasks/{task_id}",
                )
                
                if response.status_code == 200:
                    data = response.json()
                    logger.debug(f"MCP Task state: task_id={task_id}, state={data.get('state')}")
                    return data
                else:
                    logger.error(
                        f"Failed to get MCP Task state: "
                        f"status={response.status_code}"
                    )
                    return None
        
        except Exception as e:
            logger.error(f"Exception getting MCP Task state: {str(e)}")
            return None
    
    async def complete_task(
        self,
        task_id: str,
        result: Optional[Dict[str, Any]] = None,
    ) -> bool:
        """MCP Task 완료 처리.
        
        Args:
            task_id: Task ID
            result: 완료 결과 (선택)
        
        Returns:
            성공 여부
        """
        if not self.enabled:
            return False
        
        try:
            async with httpx.AsyncClient(timeout=self.timeout) as client:
                response = await client.post(
                    f"{self.base_url}/mcp/tasks/{task_id}/complete",
                    json={"result": result or {}},
                )
                
                success = response.status_code == 200
                logger.info(f"MCP Task completed: task_id={task_id}, success={success}")
                return success
        
        except Exception as e:
            logger.error(f"Exception completing MCP Task: {str(e)}")
            return False
    
    async def fail_task(
        self,
        task_id: str,
        reason: Optional[str] = None,
    ) -> bool:
        """MCP Task 실패 처리.
        
        Args:
            task_id: Task ID
            reason: 실패 사유 (선택)
        
        Returns:
            성공 여부
        """
        if not self.enabled:
            return False
        
        try:
            async with httpx.AsyncClient(timeout=self.timeout) as client:
                response = await client.post(
                    f"{self.base_url}/mcp/tasks/{task_id}/fail",
                    json={"reason": reason or "Unknown reason"},
                )
                
                success = response.status_code == 200
                logger.info(f"MCP Task failed: task_id={task_id}, success={success}")
                return success
        
        except Exception as e:
            logger.error(f"Exception failing MCP Task: {str(e)}")
            return False
```

### 4-2. TaskProposalBridge 클래스

**파일:** `/backend/app/services/task_proposal_bridge.py` (신규)

```python
"""Task-Draft 제안 브릿지 (S2-FG5.3)."""

from __future__ import annotations

import logging
from typing import Optional, Dict, Any
from uuid import UUID
from datetime import datetime
import asyncio

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.models.draft import Draft, DraftStatus
from app.models.audit_log import AuditLog
from app.mcp.tasks_client import MCPTasksClient
from app.exceptions import InvalidRequestError

logger = logging.getLogger(__name__)


class TaskProposalMetadata:
    """Task 메타데이터."""
    
    @staticmethod
    def build(
        draft_id: UUID,
        document_id: Optional[UUID],
        agent_id: UUID,
    ) -> Dict[str, Any]:
        """Task 메타데이터 구성."""
        return {
            "draft_id": str(draft_id),
            "document_id": str(document_id) if document_id else None,
            "agent_id": str(agent_id),
            "created_at": datetime.utcnow().isoformat(),
        }


class TaskProposalBridge:
    """Task-Draft 제안 브릿지.
    
    Draft proposed 생성 ↔ MCP Task 생성 ↔ Draft 상태 동기화
    """
    
    def __init__(self, db: AsyncSession):
        """Args: db — 데이터베이스 세션"""
        self.db = db
        self.tasks_client = MCPTasksClient()
    
    async def create_task_for_draft_proposal(
        self,
        draft: Draft,
        agent_id: UUID,
    ) -> Optional[str]:
        """Draft proposed 생성 시 MCP Task 생성.
        
        Args:
            draft: Draft 모델
            agent_id: 에이전트 ID
        
        Returns:
            task_id (성공), None (실패 또는 MCP Tasks disabled)
        """
        try:
            # ① Task 제목/설명 구성
            title = f"Review Draft: {draft.title[:100]}"
            description = (
                f"Document: {draft.document_id or 'New'}\n"
                f"Type: {draft.document_type_id}\n"
                f"Agent: {agent_id}\n"
                f"Status: {draft.status}\n"
                f"Content preview: {draft.content[:200]}..."
            )
            
            # ② 메타데이터 구성
            metadata = TaskProposalMetadata.build(
                draft_id=draft.id,
                document_id=draft.document_id,
                agent_id=agent_id,
            )
            
            # ③ MCP Task 생성 (fallback 지원)
            task_id = await self.tasks_client.create_task(
                title=title,
                description=description,
                metadata=metadata,
            )
            
            if task_id:
                # ④ Draft에 task_id 저장 (메타데이터)
                if draft.metadata is None:
                    draft.metadata = {}
                draft.metadata["mcp_task_id"] = task_id
                
                # ⑤ 감시 로그 (S2 원칙 ⑤)
                await self._log_task_create(draft.id, task_id)
                
                logger.info(f"Task created for draft: draft_id={draft.id}, task_id={task_id}")
                
                # ⑥ 비동기 상태 감시 시작
                asyncio.create_task(
                    self._monitor_task_state(draft.id, task_id)
                )
            else:
                logger.warning(
                    f"MCP Task creation failed or disabled for draft {draft.id} "
                    f"(fallback: using proposal queue only)"
                )
            
            return task_id
        
        except Exception as e:
            logger.error(f"Exception creating task for draft: {str(e)}")
            return None
    
    async def _monitor_task_state(self, draft_id: UUID, task_id: str) -> None:
        """MCP Task 상태 감시 (polling).
        
        Task state 변화를 감시하고 Draft 상태를 동기화.
        """
        logger.info(f"Starting task state monitoring: draft_id={draft_id}, task_id={task_id}")
        
        last_state = "input_required"
        max_retries = 60  # 최대 1시간 (1분 간격)
        retry_count = 0
        
        while retry_count < max_retries:
            try:
                await asyncio.sleep(60)  # 1분 간격
                
                # Task 상태 조회
                task_info = await self.tasks_client.get_task_state(task_id)
                
                if not task_info:
                    retry_count += 1
                    continue
                
                current_state = task_info.get("state")
                progress = task_info.get("progressPercentage", 0)
                
                logger.debug(
                    f"Task state: draft_id={draft_id}, "
                    f"task_id={task_id}, state={current_state}, "
                    f"progress={progress}"
                )
                
                # 상태 변화 감시
                if current_state != last_state:
                    await self._sync_draft_state(draft_id, task_id, current_state)
                    last_state = current_state
                
                # Task 완료/실패 시 감시 종료
                if current_state in ["completed", "failed"]:
                    logger.info(
                        f"Task monitoring ended: draft_id={draft_id}, "
                        f"task_id={task_id}, state={current_state}"
                    )
                    break
                
                retry_count += 1
            
            except Exception as e:
                logger.error(
                    f"Exception monitoring task state: "
                    f"draft_id={draft_id}, task_id={task_id}, error={str(e)}"
                )
                retry_count += 1
    
    async def _sync_draft_state(
        self,
        draft_id: UUID,
        task_id: str,
        task_state: str,
    ) -> None:
        """MCP Task 상태 → Draft 상태 동기화.
        
        - Task completed → Draft approved
        - Task failed → Draft rejected
        """
        try:
            # Draft 조회
            stmt = select(Draft).where(Draft.id == draft_id)
            result = await self.db.execute(stmt)
            draft = result.scalar_one_or_none()
            
            if not draft:
                logger.warning(f"Draft not found for sync: draft_id={draft_id}")
                return
            
            # Task 상태 → Draft 상태 매핑
            if task_state == "completed":
                new_status = DraftStatus.APPROVED.value
                action = "task_completed"
            elif task_state == "failed":
                new_status = DraftStatus.REJECTED.value
                action = "task_failed"
            else:
                # 진행 중 (input_required, in_progress 등)
                logger.debug(f"Task in progress: task_id={task_id}")
                return
            
            # Draft 상태 변경
            if draft.status != new_status:
                draft.status = new_status
                draft.updated_at = datetime.utcnow()
                
                # 감시 로그 (S2 원칙 ⑤)
                await self._log_task_state_change(
                    draft_id=draft_id,
                    task_id=task_id,
                    task_state=task_state,
                    new_draft_status=new_status,
                    action=action,
                )
                
                logger.info(
                    f"Draft state synced: draft_id={draft_id}, "
                    f"task_state={task_state} → draft_status={new_status}"
                )
            
            await self.db.commit()
        
        except Exception as e:
            logger.error(
                f"Exception syncing draft state: "
                f"draft_id={draft_id}, task_id={task_id}, error={str(e)}"
            )
            await self.db.rollback()
    
    # ==================== Helper Methods ====================
    
    async def _log_task_create(
        self,
        draft_id: UUID,
        task_id: str,
    ) -> None:
        """Task 생성 감시 로그 (S2 원칙 ⑤)."""
        try:
            audit_log = AuditLog(
                entity_type="draft",
                entity_id=draft_id,
                action="task_create",
                actor_id=UUID(int=0),  # 시스템 작업
                actor_type="system",
                changes={
                    "task_id": task_id,
                    "reason": "MCP Task created for draft proposal",
                },
            )
            self.db.add(audit_log)
            await self.db.flush()
        except Exception as e:
            logger.error(f"Failed to log task creation: {str(e)}")
    
    async def _log_task_state_change(
        self,
        draft_id: UUID,
        task_id: str,
        task_state: str,
        new_draft_status: str,
        action: str,
    ) -> None:
        """Task 상태 변화 감시 로그."""
        try:
            audit_log = AuditLog(
                entity_type="draft",
                entity_id=draft_id,
                action=action,
                actor_id=UUID(int=0),  # 시스템 작업
                actor_type="system",
                changes={
                    "task_id": task_id,
                    "task_state": task_state,
                    "new_draft_status": new_draft_status,
                },
            )
            self.db.add(audit_log)
            await self.db.flush()
        except Exception as e:
            logger.error(f"Failed to log task state change: {str(e)}")
```

### 4-3. DraftProposalService 통합 (Task 5-1 확장)

**파일:** `/backend/app/services/draft_proposals.py` (기존 파일 수정)

```python
# 기존 DraftProposalService 클래스에 추가

async def create_agent_proposal(
    self,
    agent_id: UUID,
    request: ProposeDraftRequest,
    scope_profile: Dict[str, Any],
) -> ProposeDraftResponse:
    """에이전트 Draft 제안 생성 (Task 생성 통합)."""
    
    logger.info(...)
    
    try:
        # ① ~ ⑦ 기존 로직 (Task 5-1)
        ...
        
        # ⑧ MCP Task 생성 (S2 원칙 ⑦: fallback 지원)
        from app.services.task_proposal_bridge import TaskProposalBridge
        
        bridge = TaskProposalBridge(self.db)
        task_id = await bridge.create_task_for_draft_proposal(
            draft=draft,
            agent_id=agent_id,
        )
        
        # task_id가 None이어도 정상 (fallback)
        if task_id:
            response.proposal_url += f"&task_id={task_id}"
        
        logger.info(f"Agent proposal created with optional MCP Task: draft_id={draft_id}")
        return response
    
    except ...:
        ...
```

### 4-4. 상태 감시 스케줄러 (백그라운드 작업)

**파일:** `/backend/app/tasks/task_monitor.py` (신규)

```python
"""MCP Task 상태 감시 백그라운드 작업."""

import logging
from uuid import UUID

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.models.draft import Draft
from app.services.task_proposal_bridge import TaskProposalBridge

logger = logging.getLogger(__name__)


async def monitor_pending_tasks(db: AsyncSession) -> None:
    """
    pending 상태의 모든 Draft에 대해 MCP Task 감시.
    
    APScheduler 또는 Celery에서 주기적으로 호출 (예: 매 1분마다).
    """
    try:
        # 메타데이터에 mcp_task_id가 있는 Draft 조회
        stmt = select(Draft).where(
            (Draft.status == "proposed") &
            (Draft.metadata != None)
        )
        result = await db.execute(stmt)
        drafts = result.scalars().all()
        
        bridge = TaskProposalBridge(db)
        
        for draft in drafts:
            metadata = draft.metadata or {}
            task_id = metadata.get("mcp_task_id")
            
            if task_id:
                # Task 상태 감시
                await bridge._sync_draft_state(draft.id, task_id, None)
        
        logger.info(f"Task monitoring completed for {len(drafts)} drafts")
    
    except Exception as e:
        logger.error(f"Exception in task monitoring: {str(e)}")
```

**파일:** `/backend/app/scheduler.py` (신규 또는 기존 파일 확장)

```python
"""APScheduler 설정 (Task 상태 감시)."""

from apscheduler.schedulers.asyncio import AsyncIOScheduler
from app.tasks.task_monitor import monitor_pending_tasks
from app.db import get_db

scheduler = AsyncIOScheduler()


def start_scheduler(app):
    """애플리케이션 시작 시 스케줄러 시작."""
    
    # Task 상태 감시: 매 1분마다
    scheduler.add_job(
        monitor_pending_tasks,
        "interval",
        minutes=1,
        id="monitor_pending_tasks",
        replace_existing=True,
    )
    
    scheduler.start()
    print("Background task scheduler started")


def stop_scheduler():
    """애플리케이션 종료 시 스케줄러 종료."""
    if scheduler.running:
        scheduler.shutdown()
```

**파일:** `/backend/main.py` (FastAPI 진입점 수정)

```python
"""FastAPI 애플리케이션 진입점."""

from fastapi import FastAPI
from app.scheduler import start_scheduler, stop_scheduler

app = FastAPI()

@app.on_event("startup")
async def startup():
    """애플리케이션 시작."""
    start_scheduler(app)

@app.on_event("shutdown")
async def shutdown():
    """애플리케이션 종료."""
    stop_scheduler()
```

### 4-5. 단위/통합 테스트

**파일:** `/backend/tests/unit/services/test_task_proposal_bridge.py`

```python
"""Task-Draft 제안 브릿지 단위 테스트."""

import pytest
from uuid import uuid4
from unittest.mock import AsyncMock, patch
from sqlalchemy.ext.asyncio import AsyncSession

from app.services.task_proposal_bridge import TaskProposalBridge
from app.models.draft import Draft, DraftStatus


@pytest.fixture
def bridge(db_session: AsyncSession):
    """TaskProposalBridge 인스턴스."""
    return TaskProposalBridge(db_session)


@pytest.mark.asyncio
async def test_create_task_for_draft_proposal(bridge):
    """Draft proposed 시 MCP Task 생성."""
    draft = Draft(
        id=uuid4(),
        document_id=uuid4(),
        document_type_id=uuid4(),
        title="Test Draft",
        content="Test content",
        status=DraftStatus.PROPOSED.value,
        created_by_agent=True,
        agent_id=uuid4(),
    )
    
    agent_id = uuid4()
    
    # Mock MCPTasksClient.create_task
    with patch.object(bridge.tasks_client, 'create_task', new_callable=AsyncMock) as mock_create:
        mock_create.return_value = "task-123"
        
        task_id = await bridge.create_task_for_draft_proposal(draft, agent_id)
        
        assert task_id == "task-123"
        assert draft.metadata.get("mcp_task_id") == "task-123"


@pytest.mark.asyncio
async def test_create_task_fallback_when_disabled(bridge):
    """MCP Tasks disabled 시 fallback."""
    bridge.tasks_client.enabled = False
    
    draft = Draft(
        id=uuid4(),
        document_id=uuid4(),
        document_type_id=uuid4(),
        title="Test Draft",
        content="Test content",
        status=DraftStatus.PROPOSED.value,
        created_by_agent=True,
        agent_id=uuid4(),
    )
    
    task_id = await bridge.create_task_for_draft_proposal(draft, uuid4())
    
    # Fallback: task_id는 None
    assert task_id is None
    # Draft는 정상 생성됨
    assert draft.status == DraftStatus.PROPOSED.value


@pytest.mark.asyncio
async def test_sync_draft_state_completed(bridge, db_session):
    """Task completed → Draft approved."""
    draft_id = uuid4()
    
    # (테스트: Draft가 DB에 있다고 가정)
    # await bridge._sync_draft_state(draft_id, "task-123", "completed")
    # 
    # # Draft 상태 확인
    # from sqlalchemy import select
    # stmt = select(Draft).where(Draft.id == draft_id)
    # result = await db_session.execute(stmt)
    # draft = result.scalar_one()
    # assert draft.status == DraftStatus.APPROVED.value


@pytest.mark.asyncio
async def test_sync_draft_state_failed(bridge, db_session):
    """Task failed → Draft rejected."""
    draft_id = uuid4()
    
    # await bridge._sync_draft_state(draft_id, "task-123", "failed")
    # 
    # from sqlalchemy import select
    # stmt = select(Draft).where(Draft.id == draft_id)
    # result = await db_session.execute(stmt)
    # draft = result.scalar_one()
    # assert draft.status == DraftStatus.REJECTED.value
```

## 5. 산출물

1. **MCP Tasks 클라이언트** (`/backend/app/mcp/tasks_client.py`)
   - MCPTasksClient: Task 생성, 상태 조회, 완료/실패 처리

2. **Task-Draft 브릿지** (`/backend/app/services/task_proposal_bridge.py`)
   - TaskProposalBridge: Task 생성, 상태 감시, 동기화
   - TaskProposalMetadata: 메타데이터 관리

3. **DraftProposalService 통합** (`/backend/app/services/draft_proposals.py` 확장)
   - create_agent_proposal() → create_task_for_draft_proposal() 호출

4. **백그라운드 작업** (`/backend/app/tasks/task_monitor.py`)
   - monitor_pending_tasks: Task 상태 감시 (주기적)

5. **스케줄러 설정** (`/backend/app/scheduler.py`)
   - APScheduler 통합

6. **진입점 수정** (`/backend/main.py`)
   - startup/shutdown 이벤트 추가

7. **단위 테스트** (`/backend/tests/unit/services/test_task_proposal_bridge.py`)
   - Task 생성, fallback, 상태 동기화

## 6. 완료 기준

1. MCPTasksClient가 MCP Tasks와 통신하는가?
2. TaskProposalBridge가 Draft proposed 시 Task를 생성하는가?
3. MCP_TASKS_ENABLED=off 시에도 Draft 제안이 정상 동작하는가? (fallback)
4. Task completed → Draft approved로 동기화되는가?
5. Task failed → Draft rejected로 동기화되는가?
6. Task 상태 감시 (polling)가 정상 작동하는가?
7. 모든 이벤트가 actor_type=system과 함께 감시 로그되는가? (S2 원칙 ⑤)
8. 백그라운드 작업이 정기적으로 Task를 감시하는가?
9. 모든 단위 테스트가 통과하는가?
10. mypy 타입 검사를 통과하는가?

## 7. 작업 지침

### 지침 7-1. Fallback 메커니즘 (S2 원칙 ⑦)

MCP Tasks가 disabled되었을 때도 Draft 제안이 정상적으로 동작해야 한다. Task 생성 실패 시 경고 로그를 남기되, Draft 상태 변경을 중단하지 않는다.

### 지침 7-2. 비동기 감시

Task 상태 감시는 비동기 작업(asyncio.create_task)으로 진행하되, 장시간 감시(최대 1시간)되거나 완료/실패 시 자동 종료된다.

### 지침 7-3. Task 메타데이터

Task 생성 시 draft_id, document_id, agent_id를 메타데이터에 포함하여 추적 가능성을 높인다.

### 지침 7-4. 감시 로그 (S2 원칙 ⑤)

- Task 생성: action=task_create, actor_type=system
- Task 상태 변화: action=task_completed/task_failed, actor_type=system
- 이를 통해 자동화된 시스템 작업을 인간과 구분한다.

### 지침 7-5. 중복 감시 방지

Draft에 이미 mcp_task_id가 있으면 새로운 Task를 생성하지 않는다.

### 지침 7-6. 환경변수 관리

MCP_TASKS_ENABLED와 MCP_TASKS_BASE_URL은 환경변수로 관리하여, 폐쇄망 환경에서도 설정 변경만으로 on/off 가능하게 한다.

### 지침 7-7. 에러 처리

Task 통신 에러는 로그에 기록하지만 Draft 상태 변경은 계속 진행한다. 즉, MCP Task 실패가 Draft 제안 시스템을 중단시키지 않는다.

### 지침 7-8. 스케줄러 시작/종료

FastAPI의 startup/shutdown 이벤트에서 스케줄러를 시작/종료하여 애플리케이션 생명주기와 동기화한다.
