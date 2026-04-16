# Task 5-8: 에이전트 감사 로그 확장 + 감사 조회/통계 API

## 1. 작업 목적

Mimir S2 Phase 5 Agent Action Plane에서 AI 에이전트의 모든 활동을 완전히 감시하고, 관리자가 에이전트별 행동 패턴, 승인율, 거부 사유를 추적할 수 있는 감사 로그 시스템을 구축합니다. S2 원칙 ⑤ "AI 에이전트는 사람과 동등한 API 소비자"를 구현하여, 에이전트 활동을 사용자 활동과 동등하게 감사 기록합니다.

## 2. 작업 범위

### 포함 사항
- 기존 S1 Phase 12 감사 로그 스키마 확장 (audit_logs 테이블)
- 에이전트 전용 action enum 추가 및 actor_type=agent 지원
- actor_id, acting_on_behalf_of, details JSONB 필드 확장
- DB 마이그레이션 스크립트
- AgentAuditService: 에이전트 활동 기록 로직
- 에이전트 감사 조회 API: GET /api/v1/admin/agents/{agent_id}/audit
- 에이전트 통계 API: GET /api/v1/admin/agents/{agent_id}/statistics
- AgentStatisticsService: 통계 집계 로직
- Scope Profile 기반 ACL (admin 권한 필수)
- 단위 테스트 및 통합 테스트

### 제외 사항
- 회귀 테스트 시나리오 (Task 5-9에서 담당)
- Admin UI 대시보드 (Phase 6 FG6.2에서 담당)
- 성능 벤치마크 및 부하 테스트 (Task 5-10에서 담당)
- 감사 로그 익스포트/아카이빙 기능

## 3. 선행 조건

- Phase 4 FG4.1 ~ FG4.3: 에이전트 인프라, 검색 API, 킬스위치 완료
- Phase 5 FG5.1 ~ FG5.2: 에이전트 Draft 제안, 인젝션 감지 완료
- S1 Phase 12 감사 로그 시스템 존재 (audit_logs 테이블)
- DB 마이그레이션 도구 구성 완료
- Scope Profile ACL 시스템 완료

## 4. 주요 작업 항목

### 4-1. 감사 로그 스키마 설계 및 DB 마이그레이션

#### 4-1-1. audit_logs 테이블 확장 설계
기존 audit_logs 테이블 구조에 다음 필드를 추가합니다:

**기존 필드 (변경 없음):**
- id (UUID, PK)
- timestamp (timestamptz)
- user_id (UUID, FK to users)
- resource_type (enum: 'document', 'workspace', 'agent')
- resource_id (UUID)
- action (enum)

**신규 필드:**
```sql
ALTER TABLE audit_logs
ADD COLUMN actor_type VARCHAR(20) DEFAULT 'user' NOT NULL,
    -- 'user' | 'agent' : 행동 주체의 타입
ADD COLUMN actor_id UUID,
    -- actor_type='user'일 때는 user_id와 동일, 
    -- actor_type='agent'일 때는 agent principal id
ADD COLUMN acting_on_behalf_of UUID,
    -- agent가 user를 대신할 때만 설정 (FK to users)
ADD COLUMN details JSONB DEFAULT '{}',
    -- agent_name, proposal_id, draft_id, reason 등 상세 정보
ADD INDEX idx_actor_type_id (actor_type, actor_id),
ADD INDEX idx_acting_on_behalf_of (acting_on_behalf_of),
ADD INDEX idx_resource_agent (resource_type, actor_id, timestamp DESC)
    -- 에이전트별 활동 조회 최적화
```

#### 4-1-2. action enum 확장
기존 action enum에 에이전트 전용 action 추가:

```sql
-- 기존: 'create', 'update', 'delete', 'publish', 'approve', 'reject', ...
-- 신규 추가:
-- 'agent_search': 에이전트가 문서 검색 실행
-- 'agent_draft_create': 에이전트가 Draft 생성 제안
-- 'agent_transition_propose': 에이전트가 상태 전이 제안
-- 'agent_proposal_withdraw': 에이전트가 제안 철회
```

#### 4-1-3. DB 마이그레이션 스크립트
```python
# mimir/db/migrations/002_agent_audit_extension.py

def migrate_up():
    """
    audit_logs 테이블 확장:
    - actor_type, actor_id, acting_on_behalf_of, details 추가
    - 인덱스 추가
    """
    with get_db_connection() as conn:
        conn.execute("""
            ALTER TABLE audit_logs
            ADD COLUMN actor_type VARCHAR(20) DEFAULT 'user' NOT NULL;
        """)
        conn.execute("""
            ALTER TABLE audit_logs
            ADD COLUMN actor_id UUID;
        """)
        conn.execute("""
            ALTER TABLE audit_logs
            ADD COLUMN acting_on_behalf_of UUID REFERENCES users(id);
        """)
        conn.execute("""
            ALTER TABLE audit_logs
            ADD COLUMN details JSONB DEFAULT '{}';
        """)
        conn.execute("""
            CREATE INDEX idx_actor_type_id ON audit_logs(actor_type, actor_id);
        """)
        conn.execute("""
            CREATE INDEX idx_acting_on_behalf_of ON audit_logs(acting_on_behalf_of);
        """)
        conn.execute("""
            CREATE INDEX idx_resource_agent ON audit_logs(resource_type, actor_id, timestamp DESC);
        """)
        conn.commit()

def migrate_down():
    """롤백"""
    with get_db_connection() as conn:
        conn.execute("""
            DROP INDEX IF EXISTS idx_resource_agent;
        """)
        conn.execute("""
            DROP INDEX IF EXISTS idx_acting_on_behalf_of;
        """)
        conn.execute("""
            DROP INDEX IF EXISTS idx_actor_type_id;
        """)
        conn.execute("""
            ALTER TABLE audit_logs
            DROP COLUMN IF EXISTS details;
        """)
        conn.execute("""
            ALTER TABLE audit_logs
            DROP COLUMN IF EXISTS acting_on_behalf_of;
        """)
        conn.execute("""
            ALTER TABLE audit_logs
            DROP COLUMN IF EXISTS actor_id;
        """)
        conn.execute("""
            ALTER TABLE audit_logs
            DROP COLUMN IF EXISTS actor_type;
        """)
        conn.commit()
```

### 4-2. AgentAuditService 구현

#### 4-2-1. AgentAuditService 클래스
```python
# mimir/services/agent_audit.py

from uuid import UUID
from datetime import datetime
from enum import Enum
from typing import Optional, Dict, Any
from sqlalchemy import insert
from mimir.db.models import AuditLog

class AgentAction(str, Enum):
    """에이전트 전용 감사 action"""
    SEARCH = "agent_search"
    DRAFT_CREATE = "agent_draft_create"
    TRANSITION_PROPOSE = "agent_transition_propose"
    PROPOSAL_WITHDRAW = "agent_proposal_withdraw"

class AgentAuditService:
    """에이전트 활동을 감사 로그에 기록하는 서비스"""
    
    def __init__(self, db_session):
        self.db = db_session
    
    def log_agent_search(
        self,
        agent_id: UUID,
        agent_name: str,
        acting_on_behalf_of: Optional[UUID],
        query: str,
        result_count: int,
        scope_id: UUID,
        reason: Optional[str] = None
    ) -> UUID:
        """
        에이전트 검색 활동 기록
        
        Args:
            agent_id: 에이전트 principal id
            agent_name: 에이전트 이름
            acting_on_behalf_of: 대신하는 사용자 ID (있으면)
            query: 검색 쿼리
            result_count: 검색 결과 수
            scope_id: 검색 범위 ID (workspace_id 등)
            reason: 검색 이유 (선택)
        
        Returns:
            audit_log_id
        """
        audit_log = AuditLog(
            timestamp=datetime.utcnow(),
            actor_type="agent",
            actor_id=agent_id,
            acting_on_behalf_of=acting_on_behalf_of,
            resource_type="document",
            resource_id=scope_id,
            action=AgentAction.SEARCH,
            details={
                "agent_name": agent_name,
                "query": query,
                "result_count": result_count,
                "reason": reason
            }
        )
        self.db.add(audit_log)
        self.db.flush()
        return audit_log.id
    
    def log_agent_draft_create(
        self,
        agent_id: UUID,
        agent_name: str,
        draft_id: UUID,
        document_id: UUID,
        acting_on_behalf_of: Optional[UUID],
        reason: Optional[str] = None
    ) -> UUID:
        """
        에이전트 Draft 생성 제안 기록
        
        Args:
            agent_id: 에이전트 principal id
            agent_name: 에이전트 이름
            draft_id: 생성된 Draft ID
            document_id: 문서 ID
            acting_on_behalf_of: 대신하는 사용자 ID (있으면)
            reason: 생성 사유
        
        Returns:
            audit_log_id
        """
        audit_log = AuditLog(
            timestamp=datetime.utcnow(),
            actor_type="agent",
            actor_id=agent_id,
            acting_on_behalf_of=acting_on_behalf_of,
            resource_type="document",
            resource_id=document_id,
            action=AgentAction.DRAFT_CREATE,
            details={
                "agent_name": agent_name,
                "draft_id": str(draft_id),
                "reason": reason
            }
        )
        self.db.add(audit_log)
        self.db.flush()
        return audit_log.id
    
    def log_agent_transition_propose(
        self,
        agent_id: UUID,
        agent_name: str,
        proposal_id: UUID,
        document_id: UUID,
        from_status: str,
        to_status: str,
        acting_on_behalf_of: Optional[UUID],
        reason: Optional[str] = None
    ) -> UUID:
        """
        에이전트 상태 전이 제안 기록
        
        Args:
            agent_id: 에이전트 principal id
            agent_name: 에이전트 이름
            proposal_id: 제안 ID
            document_id: 문서 ID
            from_status: 현재 상태
            to_status: 제안 상태
            acting_on_behalf_of: 대신하는 사용자 ID (있으면)
            reason: 제안 사유
        
        Returns:
            audit_log_id
        """
        audit_log = AuditLog(
            timestamp=datetime.utcnow(),
            actor_type="agent",
            actor_id=agent_id,
            acting_on_behalf_of=acting_on_behalf_of,
            resource_type="document",
            resource_id=document_id,
            action=AgentAction.TRANSITION_PROPOSE,
            details={
                "agent_name": agent_name,
                "proposal_id": str(proposal_id),
                "from_status": from_status,
                "to_status": to_status,
                "reason": reason
            }
        )
        self.db.add(audit_log)
        self.db.flush()
        return audit_log.id
    
    def log_agent_proposal_withdraw(
        self,
        agent_id: UUID,
        agent_name: str,
        proposal_id: UUID,
        document_id: UUID,
        acting_on_behalf_of: Optional[UUID],
        reason: Optional[str] = None
    ) -> UUID:
        """
        에이전트 제안 철회 기록
        
        Args:
            agent_id: 에이전트 principal id
            agent_name: 에이전트 이름
            proposal_id: 제안 ID
            document_id: 문서 ID
            acting_on_behalf_of: 대신하는 사용자 ID (있으면)
            reason: 철회 사유
        
        Returns:
            audit_log_id
        """
        audit_log = AuditLog(
            timestamp=datetime.utcnow(),
            actor_type="agent",
            actor_id=agent_id,
            acting_on_behalf_of=acting_on_behalf_of,
            resource_type="document",
            resource_id=document_id,
            action=AgentAction.PROPOSAL_WITHDRAW,
            details={
                "agent_name": agent_name,
                "proposal_id": str(proposal_id),
                "reason": reason
            }
        )
        self.db.add(audit_log)
        self.db.flush()
        return audit_log.id
```

#### 4-2-2. 에이전트 API에 감사 로깅 통합
기존 에이전트 API 각 엔드포인트에서 작업 수행 후 AgentAuditService 호출:

```python
# mimir/api/v1/agents.py

from mimir.services.agent_audit import AgentAuditService

@router.post("/agents/{agent_id}/search")
async def agent_search(
    agent_id: UUID,
    query: str,
    db: Session = Depends(get_db),
    scope_profile: ScopeProfile = Depends(get_scope_profile)
):
    """에이전트 검색 실행"""
    # 권한 확인
    agent = db.query(Agent).filter(Agent.id == agent_id).first()
    if not agent:
        raise HTTPException(status_code=404)
    
    # 에이전트 활성화 확인
    if agent.is_disabled:
        raise HTTPException(status_code=403, detail="Agent is disabled")
    
    # 검색 실행
    results = agent_search_service.search(query, scope_profile)
    
    # 감사 로깅
    audit_service = AgentAuditService(db)
    audit_service.log_agent_search(
        agent_id=agent.principal_id,
        agent_name=agent.name,
        acting_on_behalf_of=None,  # 비사용자 대리인 경우
        query=query,
        result_count=len(results),
        scope_id=scope_profile.workspace_id,
        reason="Scheduled search execution"
    )
    db.commit()
    
    return {"results": results, "count": len(results)}

@router.post("/agents/{agent_id}/drafts")
async def agent_create_draft(
    agent_id: UUID,
    draft_request: DraftCreateRequest,
    db: Session = Depends(get_db),
    scope_profile: ScopeProfile = Depends(get_scope_profile)
):
    """에이전트가 Draft 생성 제안"""
    # ... 검증 및 Draft 생성 로직 ...
    
    # 감사 로깅
    audit_service = AgentAuditService(db)
    audit_service.log_agent_draft_create(
        agent_id=agent.principal_id,
        agent_name=agent.name,
        draft_id=new_draft.id,
        document_id=draft_request.document_id,
        acting_on_behalf_of=None,
        reason=draft_request.reason
    )
    db.commit()
    
    return {"draft_id": new_draft.id, "status": "proposed"}
```

### 4-3. 에이전트 감사 조회 API

#### 4-3-1. GET /api/v1/admin/agents/{agent_id}/audit
```python
# mimir/api/v1/admin/agents.py

from fastapi import APIRouter, Query, Depends
from sqlalchemy import select, and_
from datetime import datetime, timedelta
from mimir.services.scope_profile import ScopeProfile

router = APIRouter(prefix="/api/v1/admin", tags=["admin"])

@router.get("/agents/{agent_id}/audit")
async def get_agent_audit(
    agent_id: UUID,
    start_date: Optional[datetime] = Query(None),
    end_date: Optional[datetime] = Query(None),
    action_type: Optional[str] = Query(None),
    status: Optional[str] = Query(None),  # proposed, approved, rejected, published
    limit: int = Query(50, ge=1, le=500),
    offset: int = Query(0, ge=0),
    db: Session = Depends(get_db),
    scope_profile: ScopeProfile = Depends(get_scope_profile)
) -> Dict[str, Any]:
    """
    에이전트별 감사 로그 조회 (Admin 전용)
    
    Args:
        agent_id: 에이전트 ID
        start_date: 시작 날짜 (ISO 8601)
        end_date: 종료 날짜 (ISO 8601)
        action_type: 필터링 action (agent_search, agent_draft_create, ...)
        status: 제안 상태 필터링
        limit, offset: 페이지네이션
    
    Returns:
        {
            "total_count": int,
            "logs": [
                {
                    "id": UUID,
                    "timestamp": datetime,
                    "action": str,
                    "actor_type": "agent",
                    "resource_id": UUID,
                    "details": {...},
                    "acting_on_behalf_of": UUID | null
                },
                ...
            ],
            "page_info": {
                "limit": int,
                "offset": int,
                "total_count": int
            }
        }
    """
    # Scope Profile ACL: admin 권한 필수
    if "admin" not in scope_profile.permissions:
        raise HTTPException(status_code=403, detail="Admin permission required")
    
    # 에이전트 존재 확인
    agent = db.query(Agent).filter(Agent.id == agent_id).first()
    if not agent:
        raise HTTPException(status_code=404, detail="Agent not found")
    
    # 쿼리 빌드
    query = db.query(AuditLog).filter(
        AuditLog.actor_type == "agent",
        AuditLog.actor_id == agent.principal_id
    )
    
    # 날짜 범위 필터
    if start_date:
        query = query.filter(AuditLog.timestamp >= start_date)
    if end_date:
        query = query.filter(AuditLog.timestamp <= end_date)
    
    # action 필터
    if action_type:
        query = query.filter(AuditLog.action == action_type)
    
    # 상태 필터 (제안과 연결)
    if status:
        # details.status 또는 관련 proposal의 상태로 필터
        proposal_subquery = db.query(Proposal.id).filter(
            Proposal.status == status
        ).scalar_subquery()
        query = query.filter(
            AuditLog.details["proposal_id"].astext.in_(proposal_subquery)
        )
    
    # 페이지네이션
    total_count = query.count()
    logs = query.order_by(AuditLog.timestamp.desc()).limit(limit).offset(offset).all()
    
    return {
        "total_count": total_count,
        "logs": [
            {
                "id": str(log.id),
                "timestamp": log.timestamp.isoformat(),
                "action": log.action,
                "actor_type": log.actor_type,
                "resource_id": str(log.resource_id),
                "details": log.details,
                "acting_on_behalf_of": str(log.acting_on_behalf_of) if log.acting_on_behalf_of else None
            }
            for log in logs
        ],
        "page_info": {
            "limit": limit,
            "offset": offset,
            "total_count": total_count
        }
    }
```

### 4-4. 에이전트 통계 API

#### 4-4-1. GET /api/v1/admin/agents/{agent_id}/statistics
```python
# mimir/api/v1/admin/agents.py (계속)

@router.get("/agents/{agent_id}/statistics")
async def get_agent_statistics(
    agent_id: UUID,
    period_days: int = Query(30, ge=1, le=365),
    db: Session = Depends(get_db),
    scope_profile: ScopeProfile = Depends(get_scope_profile)
) -> Dict[str, Any]:
    """
    에이전트별 통계 조회 (Admin 전용)
    
    Args:
        agent_id: 에이전트 ID
        period_days: 조회 기간 (기본값 30일)
    
    Returns:
        {
            "agent_id": UUID,
            "agent_name": str,
            "period_days": int,
            "total_proposals": int,
            "approved_count": int,
            "rejected_count": int,
            "withdrawn_count": int,
            "approval_rate": float (0.0 ~ 1.0),
            "average_review_time_minutes": float | null,
            "last_activity": datetime | null,
            "rejection_reasons": [
                {"reason": str, "count": int},
                ...
            ]
        }
    """
    # Scope Profile ACL
    if "admin" not in scope_profile.permissions:
        raise HTTPException(status_code=403, detail="Admin permission required")
    
    # 에이전트 존재 확인
    agent = db.query(Agent).filter(Agent.id == agent_id).first()
    if not agent:
        raise HTTPException(status_code=404, detail="Agent not found")
    
    stats_service = AgentStatisticsService(db)
    stats = stats_service.calculate_statistics(
        agent_id=agent.principal_id,
        period_days=period_days
    )
    
    return stats
```

#### 4-4-2. AgentStatisticsService 구현
```python
# mimir/services/agent_statistics.py

from uuid import UUID
from datetime import datetime, timedelta
from typing import Dict, List, Any
from sqlalchemy import and_, func

class AgentStatisticsService:
    """에이전트 통계를 계산하는 서비스"""
    
    def __init__(self, db_session):
        self.db = db_session
    
    def calculate_statistics(
        self,
        agent_id: UUID,
        period_days: int = 30
    ) -> Dict[str, Any]:
        """
        에이전트 통계 계산
        
        Args:
            agent_id: 에이전트 principal id
            period_days: 조회 기간
        
        Returns:
            통계 딕셔너리
        """
        cutoff_date = datetime.utcnow() - timedelta(days=period_days)
        
        # 에이전트 정보 조회
        agent = self.db.query(Agent).filter(
            Agent.principal_id == agent_id
        ).first()
        
        if not agent:
            raise ValueError(f"Agent {agent_id} not found")
        
        # Draft 생성 로그 조회
        draft_logs = self.db.query(AuditLog).filter(
            and_(
                AuditLog.actor_type == "agent",
                AuditLog.actor_id == agent_id,
                AuditLog.action == "agent_draft_create",
                AuditLog.timestamp >= cutoff_date
            )
        ).all()
        
        total_proposals = len(draft_logs)
        
        # 각 proposal별 상태 조회
        approved_count = 0
        rejected_count = 0
        withdrawn_count = 0
        rejection_reasons = {}
        review_times = []
        
        for log in draft_logs:
            draft_id = log.details.get("draft_id")
            if not draft_id:
                continue
            
            # Draft 관련 Proposal 조회
            proposal = self.db.query(Proposal).filter(
                Proposal.draft_id == draft_id
            ).first()
            
            if not proposal:
                continue
            
            if proposal.status == "approved":
                approved_count += 1
            elif proposal.status == "rejected":
                rejected_count += 1
                reason = proposal.rejection_reason or "Unknown"
                rejection_reasons[reason] = rejection_reasons.get(reason, 0) + 1
            elif proposal.status == "withdrawn":
                withdrawn_count += 1
            
            # 리뷰 시간 계산 (생성 ~ 최종 상태 변경)
            if proposal.status in ("approved", "rejected"):
                time_diff = (proposal.updated_at - log.timestamp).total_seconds() / 60
                review_times.append(time_diff)
        
        # 통계 계산
        approval_rate = (
            approved_count / total_proposals 
            if total_proposals > 0 
            else 0.0
        )
        
        avg_review_time = (
            sum(review_times) / len(review_times)
            if review_times
            else None
        )
        
        # 마지막 활동 조회
        last_log = self.db.query(AuditLog).filter(
            and_(
                AuditLog.actor_type == "agent",
                AuditLog.actor_id == agent_id,
                AuditLog.timestamp >= cutoff_date
            )
        ).order_by(AuditLog.timestamp.desc()).first()
        
        last_activity = last_log.timestamp if last_log else None
        
        return {
            "agent_id": str(agent_id),
            "agent_name": agent.name,
            "period_days": period_days,
            "total_proposals": total_proposals,
            "approved_count": approved_count,
            "rejected_count": rejected_count,
            "withdrawn_count": withdrawn_count,
            "approval_rate": round(approval_rate, 4),
            "average_review_time_minutes": (
                round(avg_review_time, 2) if avg_review_time else None
            ),
            "last_activity": last_activity.isoformat() if last_activity else None,
            "rejection_reasons": [
                {"reason": reason, "count": count}
                for reason, count in sorted(
                    rejection_reasons.items(),
                    key=lambda x: x[1],
                    reverse=True
                )
            ]
        }
```

### 4-5. 단위 및 통합 테스트

#### 4-5-1. AgentAuditService 단위 테스트
```python
# tests/unit/services/test_agent_audit.py

import pytest
from uuid import uuid4
from datetime import datetime
from mimir.services.agent_audit import AgentAuditService, AgentAction
from mimir.db.models import AuditLog

class TestAgentAuditService:
    """AgentAuditService 단위 테스트"""
    
    def test_log_agent_search(self, db_session):
        """에이전트 검색 로깅 테스트"""
        service = AgentAuditService(db_session)
        agent_id = uuid4()
        scope_id = uuid4()
        
        log_id = service.log_agent_search(
            agent_id=agent_id,
            agent_name="TestAgent",
            acting_on_behalf_of=None,
            query="test query",
            result_count=5,
            scope_id=scope_id,
            reason="Test search"
        )
        
        db_session.flush()
        log = db_session.query(AuditLog).filter(AuditLog.id == log_id).first()
        
        assert log is not None
        assert log.actor_type == "agent"
        assert log.actor_id == agent_id
        assert log.action == AgentAction.SEARCH
        assert log.details["agent_name"] == "TestAgent"
        assert log.details["result_count"] == 5
    
    def test_log_agent_draft_create(self, db_session):
        """에이전트 Draft 생성 로깅 테스트"""
        service = AgentAuditService(db_session)
        agent_id = uuid4()
        draft_id = uuid4()
        document_id = uuid4()
        
        log_id = service.log_agent_draft_create(
            agent_id=agent_id,
            agent_name="TestAgent",
            draft_id=draft_id,
            document_id=document_id,
            acting_on_behalf_of=None,
            reason="Auto-correction"
        )
        
        db_session.flush()
        log = db_session.query(AuditLog).filter(AuditLog.id == log_id).first()
        
        assert log is not None
        assert log.action == AgentAction.DRAFT_CREATE
        assert log.details["draft_id"] == str(draft_id)
        assert log.resource_id == document_id
    
    def test_log_with_acting_on_behalf_of(self, db_session):
        """대리인으로 행동 시 로깅 테스트"""
        service = AgentAuditService(db_session)
        agent_id = uuid4()
        user_id = uuid4()
        
        log_id = service.log_agent_search(
            agent_id=agent_id,
            agent_name="TestAgent",
            acting_on_behalf_of=user_id,
            query="test",
            result_count=0,
            scope_id=uuid4()
        )
        
        db_session.flush()
        log = db_session.query(AuditLog).filter(AuditLog.id == log_id).first()
        
        assert log.acting_on_behalf_of == user_id
```

#### 4-5-2. 감사 조회 API 통합 테스트
```python
# tests/integration/api/test_agent_audit_api.py

import pytest
from datetime import datetime, timedelta
from uuid import uuid4

class TestAgentAuditAPI:
    """에이전트 감사 API 통합 테스트"""
    
    def test_get_agent_audit_success(self, client, db_session, admin_token):
        """감사 로그 조회 성공 테스트"""
        # 테스트 에이전트 및 로그 생성
        agent = create_test_agent(db_session)
        create_test_audit_logs(db_session, agent.principal_id, count=5)
        
        response = client.get(
            f"/api/v1/admin/agents/{agent.id}/audit",
            headers={"Authorization": f"Bearer {admin_token}"}
        )
        
        assert response.status_code == 200
        data = response.json()
        assert data["total_count"] == 5
        assert len(data["logs"]) == 5
        assert data["logs"][0]["actor_type"] == "agent"
    
    def test_get_agent_audit_with_filters(self, client, db_session, admin_token):
        """필터링을 포함한 감사 로그 조회 테스트"""
        agent = create_test_agent(db_session)
        create_test_audit_logs(db_session, agent.principal_id, count=10)
        
        start_date = datetime.utcnow() - timedelta(days=1)
        
        response = client.get(
            f"/api/v1/admin/agents/{agent.id}/audit",
            params={
                "start_date": start_date.isoformat(),
                "action_type": "agent_search",
                "limit": 5,
                "offset": 0
            },
            headers={"Authorization": f"Bearer {admin_token}"}
        )
        
        assert response.status_code == 200
        data = response.json()
        assert len(data["logs"]) <= 5
        assert all(log["action"] == "agent_search" for log in data["logs"])
    
    def test_get_agent_audit_unauthorized(self, client, db_session, user_token):
        """비admin 사용자 접근 거부 테스트"""
        agent = create_test_agent(db_session)
        
        response = client.get(
            f"/api/v1/admin/agents/{agent.id}/audit",
            headers={"Authorization": f"Bearer {user_token}"}
        )
        
        assert response.status_code == 403
```

#### 4-5-3. AgentStatisticsService 테스트
```python
# tests/unit/services/test_agent_statistics.py

class TestAgentStatisticsService:
    """AgentStatisticsService 단위 테스트"""
    
    def test_calculate_statistics_basic(self, db_session):
        """기본 통계 계산 테스트"""
        service = AgentStatisticsService(db_session)
        agent = create_test_agent(db_session)
        
        # 테스트 로그 및 Proposal 생성
        create_test_audit_logs_with_proposals(
            db_session, agent.principal_id,
            approved=3, rejected=2, withdrawn=1
        )
        
        stats = service.calculate_statistics(agent.principal_id)
        
        assert stats["total_proposals"] == 6
        assert stats["approved_count"] == 3
        assert stats["rejected_count"] == 2
        assert stats["withdrawn_count"] == 1
        assert stats["approval_rate"] == pytest.approx(0.5, rel=0.01)
    
    def test_calculate_statistics_approval_rate(self, db_session):
        """승인율 계산 정확성 테스트"""
        service = AgentStatisticsService(db_session)
        agent = create_test_agent(db_session)
        
        create_test_audit_logs_with_proposals(
            db_session, agent.principal_id,
            approved=8, rejected=2, withdrawn=0
        )
        
        stats = service.calculate_statistics(agent.principal_id)
        
        assert stats["approval_rate"] == 0.8
```

## 5. 산출물

- `/mimir/db/migrations/002_agent_audit_extension.py`: DB 마이그레이션 스크립트
- `/mimir/services/agent_audit.py`: AgentAuditService 클래스
- `/mimir/services/agent_statistics.py`: AgentStatisticsService 클래스
- `/mimir/api/v1/admin/agents.py`: Admin API 엔드포인트 (감사 조회, 통계)
- `/tests/unit/services/test_agent_audit.py`: 단위 테스트
- `/tests/integration/api/test_agent_audit_api.py`: API 통합 테스트
- `/tests/unit/services/test_agent_statistics.py`: 통계 서비스 테스트

## 6. 완료 기준

1. **스키마 확장**: audit_logs 테이블에 actor_type, actor_id, acting_on_behalf_of, details 필드 추가 완료
2. **마이그레이션**: DB 마이그레이션 스크립트 실행 성공
3. **감사 서비스**: AgentAuditService 구현 완료, 모든 에이전트 API에 로깅 통합 완료
4. **감사 조회 API**: GET /api/v1/admin/agents/{agent_id}/audit 동작 확인, 필터링 정확성 검증
5. **통계 API**: GET /api/v1/admin/agents/{agent_id}/statistics 동작 확인, 통계 계산 정확성 검증
6. **ACL 보안**: Scope Profile 기반 admin 권한 검증 동작 확인
7. **테스트 커버리지**: 단위 테스트 ≥ 85%, 통합 테스트 필수 케이스 모두 PASS
8. **인덱싱**: 성능 조회 인덱스 (idx_resource_agent, idx_actor_type_id) 적절히 동작 확인

## 7. 작업 지침

### 7-1. 마이그레이션 실행 절차
- 개발 환경에서 migration up 실행 후 스키마 검증
- 기존 audit_logs 데이터와의 호환성 확인 (actor_type 기본값 'user')
- Rollback 테스트 실행

### 7-2. 감사 로깅 통합 점검
- Phase 4 FG4.1 ~ FG4.3의 모든 에이전트 API에 로깅 호출 추가 필수
- Phase 5 FG5.1 Draft 제안 API에 로깅 추가 필수
- Phase 5 FG5.2 인젝션 감지 로직에는 로깅하지 않음 (Draft 생성 전)

### 7-3. Scope Profile ACL 적용
- 감사 조회 API와 통계 API에 admin 권한 필수 검증
- Scope Profile에 "admin" 권한이 없으면 403 반환
- 권한 없이도 조회되는 데이터 누수 방지

### 7-4. 성능 최적화
- 대량 로그 조회 시 페이지네이션 필수 사용 (limit 최대 500)
- 인덱스 활용 확인: EXPLAIN ANALYZE로 인덱스 탑재 검증
- 통계 계산은 집계 쿼리 최적화 (N+1 방지)

### 7-5. 폐쇄망 환경 고려 (S2 원칙 ⑦)
- 감사 로깅은 DB만 사용, 외부 로깅 서비스 의존성 없음
- 통계 계산도 로컬 DB만 사용
- 환경변수로 로깅 기능 on/off 가능하도록 설계하되, off 시에도 API는 정상 동작 (빈 결과 반환)

### 7-6. 검수 및 보안 검사
- 개발 완료 후 검수보고서 작성
- 보안취약점검사: SQL Injection (매개변수화 쿼리 확인), ACL 우회 (Scope Profile 필수 검증)
- 보안 검사 보고서 작성
