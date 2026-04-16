# Task 5-4: 에이전트 제안 큐 도메인 모델 + 관리 API

## 1. 작업 목적

AI 에이전트가 문서 수정을 제안할 때, 사용자 검토를 거친 후 승인/거부하는 **에이전트 제안 큐(Agent Proposals Queue)** 시스템을 구축합니다. 

- **에이전트 제안의 투명성**: 모든 에이전트 제안을 추적 가능한 상태 머신으로 관리
- **감사 추적성**: actor_type 필드로 사용자/에이전트 행동 구분 기록
- **관리자 오버뷰**: 승인 대기 제안을 한눈에 모니터링
- **S2 원칙 ⑤ 구현**: AI 에이전트를 동등한 API 소비자로 취급 (감사 로그 필수)

---

## 2. 작업 범위

### 포함 사항
- agent_proposals 테이블 설계 및 Alembic 마이그레이션 스크립트
- AgentProposal SQLAlchemy ORM 모델
- AgentProposalRepository 클래스 (CRUD, 필터링, 페이지네이션, 통계)
- 관리자 API 엔드포인트 (FastAPI):
  - GET /api/v1/admin/proposals (상태/에이전트/제안타입 필터)
  - GET /api/v1/admin/proposals/{proposal_id} (상세 조회)
  - GET /api/v1/admin/proposals/stats (통계: 전체 수, 승인 대기, 승인율)
- Scope Profile 기반 ACL (admin 역할 필수)
- 감사 로그 통합 (actor_type 필드)
- 단위 테스트, 통합 테스트

### 제외 사항
- 프롬프트 인젝션 탐지 (Task 5-5, 5-6)
- 사용자 UI "제안" 탭 (Task 5-7)
- 에이전트 응답 생성 로직 (Phase 4에서 완료)

---

## 3. 선행 조건

- Phase 4 완료: 에이전트 응답 생성, Document, DocumentDraft, Scope Profile 모델 완성
- PostgreSQL 데이터베이스 접근 가능
- Alembic 마이그레이션 환경 설정 완료
- FastAPI 관리자 라우터 기본 구조 존재
- 감시 로그(Audit Log) 시스템 구축 (actor_type 필드 지원)

---

## 4. 주요 작업 항목

### 4-1. agent_proposals 테이블 설계 및 마이그레이션

**목표**: PostgreSQL에 agent_proposals 테이블 생성

**상세 작업**:

1. 테이블 스키마 정의
   ```
   agent_proposals:
     - id (UUID, PRIMARY KEY)
     - agent_id (STRING, NOT NULL)
     - document_id (UUID, NOT NULL, FK)
     - proposal_type (STRING, NOT NULL) - 'draft' | 'transition'
     - reference_id (UUID, NULLABLE) - 관련 DocumentDraft, TransitionProposal ID
     - status (STRING, NOT NULL) - 'pending' | 'approved' | 'rejected' | 'withdrawn'
     - created_at (TIMESTAMP, NOT NULL)
     - updated_at (TIMESTAMP, NOT NULL)
     - reviewed_by (UUID, NULLABLE) - 검토한 사용자 ID
     - review_notes (TEXT, NULLABLE) - 검토 의견
     - review_timestamp (TIMESTAMP, NULLABLE) - 검토 시간
     - metadata (JSONB, DEFAULT '{}') - 향후 확장용
   ```

2. Alembic 마이그레이션 스크립트 작성
   - 파일명: `mimir/db/migrations/versions/<timestamp>_create_agent_proposals_table.py`
   - 마이그레이션 up/down 구현
   - 인덱스 설정:
     - (agent_id, status) 복합 인덱스 (에이전트별 상태 조회 성능)
     - (document_id, status) 복합 인덱스 (문서별 제안 조회)
     - created_at DESC 인덱스 (최신 제안 조회)

3. 마이그레이션 테스트
   - up 마이그레이션 실행 후 테이블 생성 확인
   - down 마이그레이션 실행 후 테이블 삭제 확인

**산출물**:
- `alembic/versions/<timestamp>_create_agent_proposals_table.py`

**완료 기준**:
- 테이블이 정상 생성/삭제됨
- 인덱스가 올바르게 생성됨

---

### 4-2. AgentProposal SQLAlchemy ORM 모델

**목표**: Pythonic한 ORM 모델로 agent_proposals 테이블 매핑

**상세 작업**:

1. 모델 클래스 정의
   - 파일: `mimir/domain/models/agent_proposal.py`
   - 필드 타입 정의 (UUID, String, DateTime 등)
   - 관계 정의:
     - Document 모델과 역참조 관계 (한 문서는 여러 제안 보유)
     - Scope Profile 캐시 (제안 생성 시점의 scope 정보 저장)

   ```python
   from sqlalchemy import Column, String, DateTime, UUID, ForeignKey, Text, JSON
   from sqlalchemy.orm import relationship
   from datetime import datetime
   
   class AgentProposal(Base):
       __tablename__ = "agent_proposals"
       
       id = Column(UUID(as_uuid=True), primary_key=True, default=uuid4)
       agent_id = Column(String(255), nullable=False, index=True)
       document_id = Column(UUID(as_uuid=True), ForeignKey("documents.id"), nullable=False)
       proposal_type = Column(String(50), nullable=False)  # 'draft' | 'transition'
       reference_id = Column(UUID(as_uuid=True), nullable=True)
       status = Column(String(50), nullable=False, index=True)  # 'pending' | 'approved' | 'rejected' | 'withdrawn'
       created_at = Column(DateTime, nullable=False, default=datetime.utcnow)
       updated_at = Column(DateTime, nullable=False, default=datetime.utcnow, onupdate=datetime.utcnow)
       reviewed_by = Column(UUID(as_uuid=True), nullable=True)
       review_notes = Column(Text, nullable=True)
       review_timestamp = Column(DateTime, nullable=True)
       metadata = Column(JSON, nullable=False, default=dict)
       
       # 관계
       document = relationship("Document", back_populates="agent_proposals")
       
       # 메서드
       def is_pending(self) -> bool:
           return self.status == "pending"
       
       def approve(self, reviewer_id: UUID, notes: str = None):
           self.status = "approved"
           self.reviewed_by = reviewer_id
           self.review_notes = notes
           self.review_timestamp = datetime.utcnow()
       
       def reject(self, reviewer_id: UUID, notes: str = None):
           self.status = "rejected"
           self.reviewed_by = reviewer_id
           self.review_notes = notes
           self.review_timestamp = datetime.utcnow()
       
       def withdraw(self):
           self.status = "withdrawn"
           self.updated_at = datetime.utcnow()
   ```

2. Pydantic DTO 정의
   - 파일: `mimir/domain/schemas/agent_proposal_schema.py`
   - AgentProposalCreate, AgentProposalUpdate, AgentProposalResponse 스키마

**산출물**:
- `mimir/domain/models/agent_proposal.py`
- `mimir/domain/schemas/agent_proposal_schema.py`

**완료 기준**:
- ORM 모델이 마이그레이션된 테이블 구조와 일치
- 모든 필드에 타입 힌팅 적용
- 비즈니스 메서드(approve, reject, withdraw) 구현

---

### 4-3. AgentProposalRepository 구현

**목표**: 데이터 접근 계층에서 CRUD, 필터링, 페이지네이션, 통계 지원

**상세 작업**:

1. Repository 클래스 정의
   - 파일: `mimir/repository/agent_proposal_repository.py`
   
   ```python
   from typing import List, Optional, Tuple
   from uuid import UUID
   from sqlalchemy.orm import Session
   from sqlalchemy import and_, or_
   
   class AgentProposalRepository:
       def __init__(self, db_session: Session):
           self.db = db_session
       
       # 기본 CRUD
       def create(self, proposal: AgentProposal) -> AgentProposal:
           self.db.add(proposal)
           self.db.commit()
           self.db.refresh(proposal)
           return proposal
       
       def get_by_id(self, proposal_id: UUID) -> Optional[AgentProposal]:
           return self.db.query(AgentProposal).filter(
               AgentProposal.id == proposal_id
           ).first()
       
       def update(self, proposal_id: UUID, updates: dict) -> Optional[AgentProposal]:
           proposal = self.get_by_id(proposal_id)
           if proposal:
               for key, value in updates.items():
                   setattr(proposal, key, value)
               self.db.commit()
               self.db.refresh(proposal)
           return proposal
       
       # 필터링 및 페이지네이션
       def list_with_filters(
           self,
           status: Optional[str] = None,
           agent_id: Optional[str] = None,
           document_id: Optional[UUID] = None,
           proposal_type: Optional[str] = None,
           skip: int = 0,
           limit: int = 20,
           order_by: str = "created_at_desc"
       ) -> Tuple[List[AgentProposal], int]:
           query = self.db.query(AgentProposal)
           
           if status:
               query = query.filter(AgentProposal.status == status)
           if agent_id:
               query = query.filter(AgentProposal.agent_id == agent_id)
           if document_id:
               query = query.filter(AgentProposal.document_id == document_id)
           if proposal_type:
               query = query.filter(AgentProposal.proposal_type == proposal_type)
           
           total = query.count()
           
           # 정렬
           if order_by == "created_at_desc":
               query = query.order_by(AgentProposal.created_at.desc())
           elif order_by == "created_at_asc":
               query = query.order_by(AgentProposal.created_at.asc())
           
           results = query.offset(skip).limit(limit).all()
           return results, total
       
       # 통계
       def get_stats(self) -> dict:
           total = self.db.query(AgentProposal).count()
           pending = self.db.query(AgentProposal).filter(
               AgentProposal.status == "pending"
           ).count()
           approved = self.db.query(AgentProposal).filter(
               AgentProposal.status == "approved"
           ).count()
           rejected = self.db.query(AgentProposal).filter(
               AgentProposal.status == "rejected"
           ).count()
           
           approval_rate = (approved / total * 100) if total > 0 else 0
           
           return {
               "total": total,
               "pending": pending,
               "approved": approved,
               "rejected": rejected,
               "approval_rate": approval_rate
           }
   ```

2. 테스트 작성
   - 파일: `tests/repository/test_agent_proposal_repository.py`
   - CRUD 테스트, 필터링 테스트, 페이지네이션 테스트, 통계 테스트

**산출물**:
- `mimir/repository/agent_proposal_repository.py`
- `tests/repository/test_agent_proposal_repository.py`

**완료 기준**:
- 모든 CRUD 메서드 정상 작동
- 필터링 조합이 정확하게 적용됨
- 페이지네이션이 skip/limit 올바르게 처리
- 통계 계산 정확함

---

### 4-4. 관리자 API 엔드포인트 구현

**목표**: FastAPI로 관리자용 제안 큐 조회 API 구현

**상세 작업**:

1. 라우터 구현
   - 파일: `mimir/api/v1/admin/proposals.py`

   ```python
   from fastapi import APIRouter, Depends, Query, HTTPException
   from uuid import UUID
   from typing import Optional
   from mimir.api.v1.dependencies import get_current_user, require_scope
   from mimir.repository.agent_proposal_repository import AgentProposalRepository
   from mimir.domain.schemas.agent_proposal_schema import (
       AgentProposalResponse, AgentProposalListResponse, ProposalStatsResponse
   )
   from mimir.audit_log import AuditLogger
   
   router = APIRouter(prefix="/api/v1/admin/proposals", tags=["admin-proposals"])
   
   @router.get("", response_model=AgentProposalListResponse)
   async def list_proposals(
       status: Optional[str] = Query(None, description="pending|approved|rejected|withdrawn"),
       agent_id: Optional[str] = Query(None),
       proposal_type: Optional[str] = Query(None, description="draft|transition"),
       skip: int = Query(0, ge=0),
       limit: int = Query(20, ge=1, le=100),
       current_user = Depends(get_current_user),
       scope_profile = Depends(require_scope("admin"))
   ):
       """
       관리자: 에이전트 제안 목록 조회
       - 상태, 에이전트 ID, 제안 타입으로 필터링
       - Scope Profile 기반 ACL: admin 역할 필수
       """
       
       # 감사 로그
       audit_logger = AuditLogger()
       audit_logger.log(
           action="list_agent_proposals",
           actor_id=str(current_user.id),
           actor_type="user",  # S2 원칙 ⑤
           resource_type="agent_proposals",
           metadata={
               "status": status,
               "agent_id": agent_id,
               "proposal_type": proposal_type,
               "skip": skip,
               "limit": limit
           }
       )
       
       repo = AgentProposalRepository(db_session)
       proposals, total = repo.list_with_filters(
           status=status,
           agent_id=agent_id,
           proposal_type=proposal_type,
           skip=skip,
           limit=limit
       )
       
       return AgentProposalListResponse(
           items=[AgentProposalResponse.from_orm(p) for p in proposals],
           total=total,
           skip=skip,
           limit=limit
       )
   
   @router.get("/{proposal_id}", response_model=AgentProposalResponse)
   async def get_proposal(
       proposal_id: UUID,
       current_user = Depends(get_current_user),
       scope_profile = Depends(require_scope("admin"))
   ):
       """
       관리자: 제안 상세 조회
       """
       
       audit_logger = AuditLogger()
       audit_logger.log(
           action="get_agent_proposal",
           actor_id=str(current_user.id),
           actor_type="user",
           resource_type="agent_proposals",
           resource_id=str(proposal_id)
       )
       
       repo = AgentProposalRepository(db_session)
       proposal = repo.get_by_id(proposal_id)
       
       if not proposal:
           raise HTTPException(status_code=404, detail="Proposal not found")
       
       return AgentProposalResponse.from_orm(proposal)
   
   @router.get("/stats/overview", response_model=ProposalStatsResponse)
   async def get_proposal_stats(
       current_user = Depends(get_current_user),
       scope_profile = Depends(require_scope("admin"))
   ):
       """
       관리자: 제안 통계 (전체 수, 승인 대기, 승인율)
       """
       
       audit_logger = AuditLogger()
       audit_logger.log(
           action="get_proposal_stats",
           actor_id=str(current_user.id),
           actor_type="user",
           resource_type="agent_proposals"
       )
       
       repo = AgentProposalRepository(db_session)
       stats = repo.get_stats()
       
       return ProposalStatsResponse(**stats)
   ```

2. 스키마 확장
   - 파일: `mimir/domain/schemas/agent_proposal_schema.py` (추가)
   
   ```python
   from pydantic import BaseModel
   from typing import List, Optional
   from uuid import UUID
   from datetime import datetime
   
   class AgentProposalResponse(BaseModel):
       id: UUID
       agent_id: str
       document_id: UUID
       proposal_type: str
       reference_id: Optional[UUID]
       status: str
       created_at: datetime
       updated_at: datetime
       reviewed_by: Optional[UUID]
       review_notes: Optional[str]
       review_timestamp: Optional[datetime]
       
       class Config:
           from_attributes = True
   
   class AgentProposalListResponse(BaseModel):
       items: List[AgentProposalResponse]
       total: int
       skip: int
       limit: int
   
   class ProposalStatsResponse(BaseModel):
       total: int
       pending: int
       approved: int
       rejected: int
       approval_rate: float
   ```

3. 라우터 메인 API에 포함
   - 파일: `mimir/api/v1/main.py` (또는 app.py)
   ```python
   from mimir.api.v1.admin import proposals
   
   app.include_router(proposals.router)
   ```

**산출물**:
- `mimir/api/v1/admin/proposals.py`
- `mimir/domain/schemas/agent_proposal_schema.py` (확장)

**완료 기준**:
- 3개 엔드포인트 모두 정상 작동
- admin 역할 검증 동작
- 감시 로그에 actor_type="user" 기록
- 필터링/페이지네이션 정확

---

### 4-5. Scope Profile 기반 ACL 검증

**목표**: 모든 admin API가 admin 역할 필수 확인

**상세 작업**:

1. require_scope 의존성 함수 재확인
   - 파일: `mimir/api/v1/dependencies.py`
   ```python
   async def require_scope(required_scope: str):
       """
       필요한 scope이 Scope Profile에서 활성화되어 있는지 확인
       S2 원칙 ⑥: scope 문자열 하드코딩 금지, Scope Profile 읽기
       """
       async def dependency(current_user = Depends(get_current_user)):
           scope_profile = get_scope_profile(current_user.scope_profile_id)
           if not scope_profile.has_capability(required_scope):
               raise HTTPException(status_code=403, detail="Insufficient permissions")
           return scope_profile
       return dependency
   ```

2. 모든 admin 엔드포인트에 적용 (위 4-4에서 이미 적용됨)

**산출물**:
- 기존 dependencies.py 검증만 수행

**완료 기준**:
- admin이 아닌 사용자 접근 시 403 반환
- admin 사용자 접근 시 정상 처리

---

### 4-6. 감시 로그 통합 (actor_type 필드)

**목표**: 모든 제안 큐 API 호출을 감시 로그에 기록, actor_type 필드 필수

**상세 작업**:

1. 감시 로그 스키마 재확인
   - 파일: `mimir/domain/models/audit_log.py`
   ```python
   class AuditLog(Base):
       __tablename__ = "audit_logs"
       
       id = Column(UUID(as_uuid=True), primary_key=True, default=uuid4)
       action = Column(String(255), nullable=False)
       actor_id = Column(UUID(as_uuid=True), nullable=False)
       actor_type = Column(String(50), nullable=False)  # "user" | "agent"  S2 원칙 ⑤
       resource_type = Column(String(255), nullable=False)
       resource_id = Column(UUID(as_uuid=True), nullable=True)
       metadata = Column(JSON, nullable=False, default=dict)
       timestamp = Column(DateTime, nullable=False, default=datetime.utcnow)
   ```

2. AuditLogger 클래스 확인
   - 파일: `mimir/audit_log.py`
   ```python
   class AuditLogger:
       def log(self, action: str, actor_id: str, actor_type: str, 
               resource_type: str, resource_id: Optional[str] = None,
               metadata: Optional[dict] = None):
           """
           S2 원칙 ⑤: actor_type은 "user" 또는 "agent"만 허용
           """
           if actor_type not in ["user", "agent"]:
               raise ValueError("actor_type must be 'user' or 'agent'")
           
           log_entry = AuditLog(
               action=action,
               actor_id=actor_id,
               actor_type=actor_type,
               resource_type=resource_type,
               resource_id=resource_id,
               metadata=metadata or {}
           )
           db_session.add(log_entry)
           db_session.commit()
   ```

3. 4-4의 모든 엔드포인트에 actor_type="user" 적용 (이미 포함됨)

**산출물**:
- 기존 audit_log 모델 검증

**완료 기준**:
- 모든 API 호출이 감시 로그에 기록됨
- actor_type이 정확하게 기록됨

---

### 4-7. 단위 및 통합 테스트

**목표**: 모든 컴포넌트에 대한 테스트 작성

**상세 작업**:

1. 데이터베이스 마이그레이션 테스트
   - 파일: `tests/db/test_migrations.py`
   ```python
   def test_agent_proposals_table_creation(migration_context):
       """마이그레이션이 테이블 생성"""
       # up 마이그레이션 실행
       # 테이블 존재 확인
       # 인덱스 확인
   
   def test_agent_proposals_table_deletion(migration_context):
       """다운 마이그레이션이 테이블 삭제"""
       # up → down 마이그레이션
       # 테이블 삭제 확인
   ```

2. ORM 모델 테스트
   - 파일: `tests/models/test_agent_proposal.py`
   ```python
   def test_agent_proposal_creation(db_session):
       """AgentProposal 객체 생성 및 저장"""
       
   def test_agent_proposal_approve(db_session):
       """제안 승인 메서드"""
       proposal = AgentProposal(...)
       proposal.approve(reviewer_id=..., notes="...")
       assert proposal.status == "approved"
       assert proposal.review_timestamp is not None
   
   def test_agent_proposal_reject(db_session):
       """제안 거부 메서드"""
   
   def test_agent_proposal_withdraw(db_session):
       """제안 철회 메서드"""
   ```

3. Repository 테스트
   - 파일: `tests/repository/test_agent_proposal_repository.py` (4-3에서 작성)

4. API 통합 테스트
   - 파일: `tests/api/v1/admin/test_proposals.py`
   ```python
   def test_list_proposals_admin(admin_client, agent_proposals_fixture):
       """관리자: 제안 목록 조회"""
       response = admin_client.get("/api/v1/admin/proposals")
       assert response.status_code == 200
       assert response.json()["total"] >= 0
   
   def test_list_proposals_unauthorized(user_client, agent_proposals_fixture):
       """비관리자: 접근 거부"""
       response = user_client.get("/api/v1/admin/proposals")
       assert response.status_code == 403
   
   def test_list_proposals_filter_by_status(admin_client, agent_proposals_fixture):
       """상태 필터링 정확성"""
       response = admin_client.get("/api/v1/admin/proposals?status=pending")
       assert all(p["status"] == "pending" for p in response.json()["items"])
   
   def test_list_proposals_pagination(admin_client, agent_proposals_fixture):
       """페이지네이션 정확성"""
       response = admin_client.get("/api/v1/admin/proposals?skip=0&limit=10")
       assert len(response.json()["items"]) <= 10
   
   def test_get_proposal_detail(admin_client, agent_proposals_fixture):
       """제안 상세 조회"""
       
   def test_get_proposal_stats(admin_client, agent_proposals_fixture):
       """통계 조회"""
       response = admin_client.get("/api/v1/admin/proposals/stats/overview")
       assert response.status_code == 200
       assert "total" in response.json()
       assert "approval_rate" in response.json()
   
   def test_audit_log_recorded(admin_client, audit_log_repository):
       """감시 로그 기록 확인"""
       response = admin_client.get("/api/v1/admin/proposals")
       logs = audit_log_repository.find_by_action("list_agent_proposals")
       assert len(logs) > 0
       assert logs[0].actor_type == "user"
   ```

5. Fixture 작성
   - 파일: `tests/conftest.py` (또는 `tests/fixtures/agent_proposal_fixtures.py`)
   ```python
   @pytest.fixture
   def agent_proposals_fixture(db_session):
       """테스트용 제안 데이터 생성"""
       proposals = [
           AgentProposal(agent_id="agent1", status="pending", ...),
           AgentProposal(agent_id="agent2", status="approved", ...),
           ...
       ]
       for p in proposals:
           db_session.add(p)
       db_session.commit()
       yield proposals
   ```

**산출물**:
- `tests/db/test_migrations.py`
- `tests/models/test_agent_proposal.py`
- `tests/repository/test_agent_proposal_repository.py`
- `tests/api/v1/admin/test_proposals.py`

**완료 기준**:
- 모든 테스트 통과율 100%
- 커버리지 > 85%
- ACL 검증 테스트 포함
- 감시 로그 기록 확인 테스트 포함

---

## 5. 산출물

### 필수 산출물
1. **Alembic 마이그레이션**
   - `alembic/versions/<timestamp>_create_agent_proposals_table.py`

2. **ORM 모델**
   - `mimir/domain/models/agent_proposal.py`
   - `mimir/domain/schemas/agent_proposal_schema.py`

3. **Repository**
   - `mimir/repository/agent_proposal_repository.py`

4. **API**
   - `mimir/api/v1/admin/proposals.py`

5. **테스트**
   - `tests/db/test_migrations.py`
   - `tests/models/test_agent_proposal.py`
   - `tests/repository/test_agent_proposal_repository.py`
   - `tests/api/v1/admin/test_proposals.py`

### 검수 산출물
- **검수 보고서**: Task 5-4 검수보고서.md
  - 마이그레이션 검증 결과
  - API 엔드포인트 기능 테스트 결과
  - ACL 검증 결과
  - 감시 로그 기록 검증

- **보안 취약점 검사 보고서**: Task 5-4 보안검사보고서.md
  - SQL Injection 위험 검토 (Parameterized Query 사용 확인)
  - ACL Bypass 위험 검토 (require_scope 적용 확인)
  - 민감한 정보 노출 검토 (metadata JSONB 필드 검증)
  - 감시 로그 조작 방지 검토

---

## 6. 완료 기준

### 기능적 완료 기준
- [x] agent_proposals 테이블이 정상 생성/삭제됨
- [x] AgentProposal ORM 모델이 모든 필드 포함
- [x] AgentProposalRepository가 CRUD, 필터링, 페이지네이션, 통계 구현
- [x] 3개 API 엔드포인트 (list, get, stats) 정상 작동
- [x] admin 역할 ACL 검증 동작
- [x] 모든 API 호출이 감시 로그에 actor_type="user"로 기록

### 테스트 완료 기준
- [x] 단위 테스트 커버리지 > 85%
- [x] 통합 테스트 모두 통과
- [x] ACL 테스트 (admin 접근 O, 비admin 403)
- [x] 감시 로그 기록 테스트

### 코드 품질 기준
- [x] S2 원칙 ⑤ 준수 (actor_type 필드 필수)
- [x] S2 원칙 ⑥ 준수 (Scope Profile 기반 ACL)
- [x] 모든 필드에 타입 힌팅 적용
- [x] 에러 처리 (404, 403 등) 명확함

---

## 7. 작업 지침

### 7-1. 마이그레이션 작성 시 주의점

- **Alembic 환경 확인**: `alembic init` 또는 기존 alembic 디렉토리 확인
- **마이그레이션 파일명**: `<timestamp>_<description>.py` 형식 (alembic이 자동 생성)
- **Foreign Key**: `ForeignKey("documents.id", ondelete="CASCADE")` 등으로 참조 무결성 보장
- **인덱스**: 복합 인덱스 (agent_id, status) 등 쿼리 성능 고려
- **테스트**: up/down 마이그레이션을 별도 테스트 DB에서 검증

### 7-2. ORM 모델 작성 시 주의점

- **datetime 필드**: `datetime.utcnow`로 UTC 타임존 통일
- **UUID 필드**: `UUID(as_uuid=True)` 옵션으로 Python uuid 객체 사용
- **관계 정의**: `relationship()` 및 `back_populates`로 양방향 관계 설정
- **메서드**: 상태 전환(approve, reject, withdraw) 메서드는 비즈니스 로직 포함

### 7-3. Repository 작성 시 주의점

- **세션 관리**: `__init__`에 db_session 주입받아 저장
- **필터 조합**: AND 조건 여러 개 필터를 `filter()` 연쇄 호출
- **페이지네이션**: `offset(skip).limit(limit)` 순서 유지
- **통계**: aggregate 함수 (COUNT, SUM 등) 활용

### 7-4. API 엔드포인트 작성 시 주의점

- **Depends 의존성**: `get_current_user`, `require_scope` 의존성 필수
- **감시 로그**: 모든 엔드포인트에서 `AuditLogger().log()` 호출, actor_type="user" 필수
- **응답 스키마**: Pydantic DTO 사용 (from_orm=True)
- **에러 처리**: `HTTPException(status_code=404/403)` 사용
- **S2 원칙**: 코드에 `if scope == "admin"` 같은 하드코딩 금지, require_scope 함수 사용

### 7-5. 테스트 작성 시 주의점

- **Fixture**: conftest.py에서 재사용 가능한 데이터 fixture 정의
- **Mock**: 외부 API 호출은 Mock 처리 (LLM 호출 제외하려면 @mock.patch 사용)
- **트랜잭션**: 각 테스트 후 롤백 (pytest-sqlalchemy 또는 @pytest.fixture(autouse=True))
- **ACL 테스트**: admin_client, user_client 등 여러 역할 클라이언트 준비

### 7-6. 코드 검토 체크리스트

**마이그레이션**:
- [ ] `revision` 변수가 올바르게 설정됨
- [ ] `down_revision`이 이전 마이그레이션을 참조
- [ ] `op.create_table()`, `op.create_index()` 호출 정확
- [ ] `upgrade()`, `downgrade()` 함수 모두 구현

**ORM 모델**:
- [ ] 모든 필드가 Column() 으로 정의됨
- [ ] NULL 허용 필드에 `nullable=True` 명시
- [ ] 기본값은 `default=` 또는 `default=func()` 사용
- [ ] 관계는 `relationship()` 및 `back_populates` 설정

**Repository**:
- [ ] CRUD 메서드 (create, read, update, delete) 모두 구현
- [ ] 필터 메서드가 여러 조건 조합 지원
- [ ] 페이지네이션이 skip/limit 정확하게 적용
- [ ] 통계 메서드가 정확한 결과 반환

**API**:
- [ ] 모든 엔드포인트가 `@require_scope` 적용
- [ ] 감시 로그가 모든 메서드에서 기록됨
- [ ] actor_type="user" 명시
- [ ] 응답 스키마가 Pydantic DTO 사용

**테스트**:
- [ ] 각 테스트 메서드가 한 가지만 테스트 (단일 책임)
- [ ] Fixture가 각 테스트 독립적으로 실행 가능
- [ ] Mock/Patch가 필요한 부분에만 적용
- [ ] 성공 경로 + 실패 경로 둘 다 테스트

### 7-7. 배포 체크리스트

- [ ] 모든 마이그레이션이 순서대로 실행 가능한지 확인
- [ ] 스키마 변경이 기존 데이터 영향 없음을 확인
- [ ] API 문서 (Swagger/OpenAPI) 업데이트
- [ ] 모니터링/로깅 설정 (감시 로그 저장 경로, 보관 기간)
- [ ] 성능 테스트 (대량 데이터 시 인덱스 효율성 검증)

---

## 8. 참고 자료

- OWASP Top 10 for LLM Applications: LLM01 - Prompt Injection
- S2 원칙 ⑤ (AI 에이전트 동등 소비자): 감사 로그에 actor_type 필드 필수
- S2 원칙 ⑥ (Scope Profile 기반 ACL): 코드에 scope 문자열 하드코딩 금지
- Phase 4 완료: Document, DocumentDraft 모델 및 API

---

**작업 시작일**: 2026-04-17  
**예상 소요시간**: 4-5일 (병렬 처리 가능)  
**담당자**: [개발팀]  
**우선순위**: High (Phase 5 Critical Path)
