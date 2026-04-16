# Task 3-4: 세션 기반 RAG API (/rag/answer + conversation_id)

## 1. 작업 목적

기존 단발 RAG 엔드포인트 `/rag/answer`를 확장하여 선택사항 `conversation_id` 매개변수를 지원하고, 멀티턴 대화 모드로 전환 가능하도록 설계한다. 이를 통해:
- 단발 RAG 모드의 하위호환성 100% 유지
- 새로운 멀티턴 모드에서 Turn 자동 생성 및 저장
- 요청/응답 스키마의 명확한 정의 및 Pydantic 검증

---

## 2. 작업 범위

### 포함 항목

1. **Request/Response Pydantic 스키마 정의**
   - `ConversationRAGRequest`: query, conversation_id (선택), user_id, top_k, context_window_size, model
   - `ConversationRAGResponse`: answer, citations, turn_id, retrieval_metadata

2. **ConversationRAGService 클래스 설계 및 구현**
   - `/rag/answer` 엔드포인트 로직을 분리하여 핵심 비즈니스 로직 캡슐화
   - conversation_id 유무 분기 처리 (단발 vs 멀티턴)
   - Turn 객체 자동 생성 및 저장
   - 검색 결과 및 메타데이터 저장

3. **FastAPI 엔드포인트 확장**
   - 기존 `/rag/answer` 호출 시 request body 구조 변경
   - 하위호환성 검증 (conversation_id=null이면 기존 동작과 동일)
   - 적절한 HTTP 상태 코드 반환 (400, 404, 500 등)

4. **S2 원칙 ⑤ 적용 (AI 에이전트 동등 소비자)**
   - request body에 `actor_type` 필드 추가 ("user" 또는 "agent")
   - 감시 로그(audit log)에 actor_type 기록

5. **S2 원칙 ⑥ 적용 (Scope Profile 기반 ACL)**
   - 검색 결과 필터링 시 user_id의 Scope Profile 기반 ACL 적용
   - Turn 저장 시 owner_id = user_id로 설정
   - 조회 시 ACL 검증

6. **S2 원칙 ⑦ 적용 (폐쇄망 동작)**
   - 환경변수 `EXTERNAL_DEPENDENCIES_ENABLED` (기본값: true)
   - 외부 LLM 서비스 unavailable 시 로컬 모델 fallback
   - 성능 저하하지만 실패하지 않음

7. **단위 테스트**
   - `test_conversation_rag_single_turn()`: conversation_id=null, 기존 동작 검증
   - `test_conversation_rag_multi_turn()`: conversation_id 제공, Turn 생성 검증
   - `test_backward_compatibility()`: request 필드 누락 시 기본값 적용
   - `test_actor_type_logging()`: actor_type이 감사 로그에 기록되는지 확인
   - `test_acl_filtering()`: 검색 결과가 user의 ACL 범위 내인지 확인
   - `test_fallback_to_local_model()`: 외부 LLM 미사용 시 로컬 모델 동작

### 제외 항목

- 멀티턴 컨텍스트 윈도우 관리 (Task 3-5)
- 쿼리 재작성 통합 (Task 3-5)
- Citation 재활용 및 캐싱 (Task 3-6)
- 성능 벤치마크 작성 (Task 3-6)
- UI 구현 (Task 3-3)

---

## 3. 선행 조건

1. **Phase 2 완료**
   - Citation 5-tuple 스키마 정의 및 검증
   - 기존 `/rag/answer` 엔드포인트 정상 작동
   - 쿼리 재작성 로직 (Phase 2 FG2.3) 구현 완료

2. **FG3.1 완료**
   - Conversation, Turn, Message 도메인 모델 정의
   - DB 마이그레이션 스크립트 (conversations, turns 테이블)
   - ORM 클래스 (SQLAlchemy) 구현

3. **Phase 1 완료**
   - LLM 모델 추상화 인터페이스
   - 토큰 카운팅 함수 (count_tokens())
   - 기본 모델 설정

4. **기존 의존 라이브러리**
   - FastAPI >= 0.100.0
   - Pydantic >= 2.0
   - SQLAlchemy >= 2.0
   - (선택) Redis >= 7.0 (캐싱용, Task 3-6에서 사용)

---

## 4. 주요 작업 항목

### 4-1. Request/Response 스키마 정의

**파일**: `backend/src/api/schemas/conversation_rag.py`

```python
from typing import Optional, List, Dict, Any
from uuid import UUID
from pydantic import BaseModel, Field
from datetime import datetime

# ===== Request Schemas =====

class CitationSchema(BaseModel):
    """Citation 5-tuple 응답 스키마"""
    document_id: UUID
    version_id: UUID
    node_id: str
    span_offset: Optional[int] = None
    content_hash: str
    content: str

class RetrievalMetadataSchema(BaseModel):
    """검색 메타데이터 응답 스키마"""
    query_rewritten: Optional[str] = None  # Phase 2 쿼리 재작성 (Task 3-5)
    retrieval_time_ms: int
    context_window_turns: Optional[List[UUID]] = []  # Task 3-5
    reranker_scores: Optional[Dict[str, float]] = {}  # 선택사항

class ConversationRAGRequest(BaseModel):
    """세션 기반 RAG 요청 스키마"""
    query: str = Field(..., min_length=1, max_length=5000)
    conversation_id: Optional[UUID] = None
    user_id: UUID
    actor_type: str = Field(default="user", pattern="^(user|agent)$")
    top_k: int = Field(default=5, ge=1, le=100)
    context_window_size: int = Field(default=3, ge=1, le=20)
    model: Optional[str] = None  # None이면 기본 모델 사용

    class Config:
        json_schema_extra = {
            "example": {
                "query": "Mimir의 주요 기능은 무엇인가?",
                "conversation_id": "550e8400-e29b-41d4-a716-446655440000",
                "user_id": "650e8400-e29b-41d4-a716-446655440000",
                "actor_type": "user",
                "top_k": 5,
                "context_window_size": 3,
                "model": "gpt-4"
            }
        }

class ConversationRAGResponse(BaseModel):
    """세션 기반 RAG 응답 스키마"""
    answer: str
    citations: List[CitationSchema]
    turn_id: Optional[UUID] = None  # 멀티턴 모드에서만 생성
    retrieval_metadata: RetrievalMetadataSchema

    class Config:
        json_schema_extra = {
            "example": {
                "answer": "Mimir는 멀티턴 대화 기반의 RAG 시스템입니다...",
                "citations": [
                    {
                        "document_id": "550e8400-e29b-41d4-a716-446655440000",
                        "version_id": "650e8400-e29b-41d4-a716-446655440001",
                        "node_id": "doc#1:node#3",
                        "span_offset": 42,
                        "content_hash": "sha256:abc123...",
                        "content": "Mimir는..."
                    }
                ],
                "turn_id": "750e8400-e29b-41d4-a716-446655440002",
                "retrieval_metadata": {
                    "query_rewritten": None,
                    "retrieval_time_ms": 145,
                    "context_window_turns": []
                }
            }
        }
```

**요구사항**:
- Pydantic `field_validator` 사용하여 query, conversation_id, user_id 검증
- actor_type은 "user" 또는 "agent" 중 하나만 허용
- top_k와 context_window_size는 합리적 범위 내에서만 허용
- response schema는 모든 필드가 직렬화 가능해야 함

---

### 4-2. ConversationRAGService 클래스 설계

**파일**: `backend/src/services/conversation_rag_service.py`

```python
from typing import Optional, List, Dict, Any
from uuid import UUID
from datetime import datetime
import logging
from sqlalchemy.orm import Session

from src.models import Conversation, Turn, User
from src.schemas.conversation_rag import ConversationRAGRequest, ConversationRAGResponse, RetrievalMetadataSchema
from src.services.retrieval_service import RetrievalService
from src.services.llm_service import LLMService
from src.core.security import enforce_acl_filter
from src.core.audit import log_audit_event
from src.core.config import settings

logger = logging.getLogger(__name__)

class ConversationRAGService:
    """멀티턴 대화 기반 RAG 서비스"""
    
    def __init__(self, db: Session, retrieval_service: RetrievalService, llm_service: LLMService):
        self.db = db
        self.retrieval = retrieval_service
        self.llm = llm_service
        self.external_deps_enabled = settings.EXTERNAL_DEPENDENCIES_ENABLED
    
    def answer(self, request: ConversationRAGRequest) -> ConversationRAGResponse:
        """
        세션 기반 RAG 답변 생성
        
        Args:
            request: ConversationRAGRequest 객체
        
        Returns:
            ConversationRAGResponse 객체
        
        Raises:
            ValueError: conversation_id가 존재하지 않을 경우
            PermissionError: user_id가 conversation에 접근 권한 없을 경우
        """
        
        # 1. user_id 검증 (사용자 존재 확인)
        user = self.db.query(User).filter(User.id == request.user_id).first()
        if not user:
            raise ValueError(f"User not found: {request.user_id}")
        
        # 2. 멀티턴 vs 단발 분기
        turn = None
        if request.conversation_id:
            # 멀티턴 모드
            turn = self._handle_multi_turn(request, user)
        else:
            # 단발 모드
            turn = None
        
        # 3. 검색 수행 (ACL 필터링 포함)
        start_time = datetime.utcnow()
        search_query = request.query  # Task 3-5에서 재작성된 쿼리로 변경 예정
        retrieval_results = self.retrieval.search(
            query=search_query,
            top_k=request.top_k,
            user_scope=user.scope_profile  # S2 원칙 ⑥
        )
        retrieval_time_ms = int((datetime.utcnow() - start_time).total_seconds() * 1000)
        
        # 4. ACL 필터링 (S2 원칙 ⑥)
        filtered_results = enforce_acl_filter(retrieval_results, user.scope_profile)
        
        # 5. LLM 응답 생성 (fallback 포함)
        try:
            if self.external_deps_enabled:
                answer, citations = self.llm.generate_answer(
                    query=search_query,
                    context=filtered_results,
                    model=request.model,
                    actor_type=request.actor_type  # S2 원칙 ⑤
                )
            else:
                # 폐쇄망: 로컬 모델 fallback (S2 원칙 ⑦)
                logger.warning("External LLM disabled. Using fallback local model.")
                answer, citations = self.llm.generate_answer_local(
                    query=search_query,
                    context=filtered_results
                )
        except Exception as e:
            logger.error(f"LLM generation failed: {str(e)}")
            if not self.external_deps_enabled:
                # 폐쇄망: 로컬 모델로 재시도
                answer, citations = self.llm.generate_answer_local(
                    query=search_query,
                    context=filtered_results
                )
            else:
                raise
        
        # 6. Turn 저장 (멀티턴 모드)
        if turn:
            turn.assistant_response = answer
            turn.retrieval_metadata = {
                "query_rewritten": None,  # Task 3-5에서 설정
                "retrieval_time_ms": retrieval_time_ms,
                "citations": [c.dict() for c in citations],
                "context_window_turns": []  # Task 3-5에서 설정
            }
            self.db.commit()
            logger.info(f"Turn saved: conversation_id={request.conversation_id}, turn_id={turn.id}")
        
        # 7. 감사 로그 (S2 원칙 ⑤)
        log_audit_event(
            event_type="rag_query",
            actor_type=request.actor_type,
            actor_id=str(request.user_id),
            resource_type="conversation",
            resource_id=str(request.conversation_id) if request.conversation_id else None,
            details={
                "query": request.query,
                "top_k": request.top_k,
                "citations_count": len(citations),
                "retrieval_time_ms": retrieval_time_ms
            }
        )
        
        # 8. 응답 구성
        return ConversationRAGResponse(
            answer=answer,
            citations=citations,
            turn_id=turn.id if turn else None,
            retrieval_metadata=RetrievalMetadataSchema(
                query_rewritten=None,
                retrieval_time_ms=retrieval_time_ms,
                context_window_turns=[]
            )
        )
    
    def _handle_multi_turn(self, request: ConversationRAGRequest, user: User) -> Turn:
        """
        멀티턴 모드: Conversation 및 Turn 생성/조회
        
        Args:
            request: ConversationRAGRequest 객체
            user: User 객체
        
        Returns:
            Turn 객체
        
        Raises:
            ValueError: conversation이 존재하지 않을 경우
            PermissionError: user가 conversation에 접근 권한 없을 경우
        """
        
        # conversation 조회 및 권한 검증
        conversation = self.db.query(Conversation).filter(
            Conversation.id == request.conversation_id
        ).first()
        
        if not conversation:
            raise ValueError(f"Conversation not found: {request.conversation_id}")
        
        # ACL 검증: user가 conversation 소유자이거나 공유받았는지 확인
        if not self._has_conversation_access(conversation, user):
            raise PermissionError(
                f"User {request.user_id} does not have access to conversation {request.conversation_id}"
            )
        
        # Turn 생성
        turn_number = self.db.query(Turn).filter(
            Turn.conversation_id == request.conversation_id
        ).count() + 1
        
        turn = Turn(
            conversation_id=request.conversation_id,
            turn_number=turn_number,
            user_message=request.query,
            assistant_response="",  # 나중에 업데이트
            retrieval_metadata={},
            created_at=datetime.utcnow()
        )
        
        self.db.add(turn)
        self.db.flush()  # ID 생성, 아직 커밋 안 함
        
        logger.info(f"Turn created: turn_id={turn.id}, conversation_id={request.conversation_id}, turn_number={turn_number}")
        
        return turn
    
    def _has_conversation_access(self, conversation: Conversation, user: User) -> bool:
        """
        사용자가 Conversation에 접근 권한이 있는지 확인
        
        Args:
            conversation: Conversation 객체
            user: User 객체
        
        Returns:
            bool: 접근 권한 있으면 True
        """
        # 간단한 구현: 소유자만 접근 가능 (Task 3-1에서 공유 기능 추가 예정)
        return conversation.owner_id == user.id
```

**요구사항**:
- 단발 모드 (conversation_id=null)와 멀티턴 모드 명확하게 분기
- 모든 데이터베이스 쿼리 시 transaction 처리
- 에러 발생 시 적절한 예외 타입 발생 (ValueError, PermissionError 등)
- logging을 통해 주요 이벤트 기록
- S2 원칙 ⑤, ⑥, ⑦ 모두 구현

---

### 4-3. FastAPI 엔드포인트 구현

**파일**: `backend/src/api/routes/rag.py` (기존 파일 확장)

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from src.api.schemas.conversation_rag import ConversationRAGRequest, ConversationRAGResponse
from src.services.conversation_rag_service import ConversationRAGService
from src.core.dependencies import get_db, get_retrieval_service, get_llm_service
from src.core.errors import handle_service_error

router = APIRouter(prefix="/api/v1/rag", tags=["rag"])

@router.post("/answer", response_model=ConversationRAGResponse)
async def answer(
    request: ConversationRAGRequest,
    db: Session = Depends(get_db),
    retrieval_service = Depends(get_retrieval_service),
    llm_service = Depends(get_llm_service)
) -> ConversationRAGResponse:
    """
    세션 기반 RAG 엔드포인트
    
    - conversation_id 없으면: 단발 RAG 모드 (기존 동작)
    - conversation_id 있으면: 멀티턴 RAG 모드 (새 Turn 생성)
    
    Args:
        request: ConversationRAGRequest 객체
        db: 데이터베이스 세션
        retrieval_service: 검색 서비스
        llm_service: LLM 서비스
    
    Returns:
        ConversationRAGResponse 객체
    
    Raises:
        HTTPException 400: 입력 검증 실패
        HTTPException 404: conversation/user 존재하지 않음
        HTTPException 403: 권한 없음
        HTTPException 500: 서버 에러
    """
    
    try:
        service = ConversationRAGService(db, retrieval_service, llm_service)
        response = service.answer(request)
        return response
    
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=str(e)
        )
    except PermissionError as e:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=str(e)
        )
    except Exception as e:
        handle_service_error(e)
```

**요구사항**:
- HTTP 상태 코드를 적절하게 반환 (200, 400, 403, 404, 500)
- request/response validation은 Pydantic이 자동 처리
- 모든 예외를 처리하여 graceful error response 반환
- OpenAPI 문서 자동 생성

---

### 4-4. 단위 테스트 작성

**파일**: `backend/tests/test_conversation_rag.py`

```python
import pytest
from uuid import uuid4
from datetime import datetime
from sqlalchemy.orm import Session
from unittest.mock import Mock, patch, MagicMock

from src.api.schemas.conversation_rag import ConversationRAGRequest, ConversationRAGResponse
from src.services.conversation_rag_service import ConversationRAGService
from src.models import Conversation, Turn, User
from src.core.exceptions import PermissionError as PermissionDenied

# ===== Fixtures =====

@pytest.fixture
def test_user():
    """테스트용 User 객체"""
    return User(
        id=uuid4(),
        username="testuser",
        email="test@example.com",
        organization_id=uuid4(),
        scope_profile={"team": ["research"], "level": "read"}
    )

@pytest.fixture
def test_conversation(test_user):
    """테스트용 Conversation 객체"""
    return Conversation(
        id=uuid4(),
        owner_id=test_user.id,
        organization_id=test_user.organization_id,
        title="Test Conversation",
        created_at=datetime.utcnow(),
        updated_at=datetime.utcnow(),
        status="active"
    )

@pytest.fixture
def mock_retrieval_service():
    """Mock RetrievalService"""
    service = Mock()
    service.search.return_value = [
        {
            "document_id": uuid4(),
            "version_id": uuid4(),
            "node_id": "doc#1:node#1",
            "content": "Test content",
            "score": 0.95
        }
    ]
    return service

@pytest.fixture
def mock_llm_service():
    """Mock LLMService"""
    service = Mock()
    service.generate_answer.return_value = (
        "This is a test answer",
        [
            {
                "document_id": uuid4(),
                "version_id": uuid4(),
                "node_id": "doc#1:node#1",
                "span_offset": 0,
                "content_hash": "sha256:abc123",
                "content": "Test content"
            }
        ]
    )
    return service

# ===== Test Cases =====

class TestConversationRAGSingleTurn:
    """단발 RAG 모드 테스트"""
    
    def test_single_turn_without_conversation_id(
        self, mock_retrieval_service, mock_llm_service, test_user, db: Session
    ):
        """conversation_id=null인 경우 기존 동작 검증"""
        
        service = ConversationRAGService(db, mock_retrieval_service, mock_llm_service)
        
        request = ConversationRAGRequest(
            query="What is Mimir?",
            conversation_id=None,
            user_id=test_user.id,
            actor_type="user",
            top_k=5,
            context_window_size=3,
            model="gpt-4"
        )
        
        response = service.answer(request)
        
        assert response.answer == "This is a test answer"
        assert len(response.citations) > 0
        assert response.turn_id is None  # 단발 모드는 turn_id 없음
        assert response.retrieval_metadata.retrieval_time_ms >= 0
    
    def test_backward_compatibility_default_values(self, mock_retrieval_service, mock_llm_service, test_user, db: Session):
        """기본값이 올바르게 적용되는지 확인"""
        
        service = ConversationRAGService(db, mock_retrieval_service, mock_llm_service)
        
        request = ConversationRAGRequest(
            query="Test query",
            user_id=test_user.id
        )
        
        assert request.conversation_id is None
        assert request.actor_type == "user"
        assert request.top_k == 5
        assert request.context_window_size == 3
        assert request.model is None

class TestConversationRAGMultiTurn:
    """멀티턴 RAG 모드 테스트"""
    
    def test_multi_turn_creates_turn(
        self, mock_retrieval_service, mock_llm_service, test_user, test_conversation, db: Session
    ):
        """conversation_id 제공 시 새 Turn 생성 검증"""
        
        # 데이터베이스에 conversation 저장
        db.add(test_conversation)
        db.add(test_user)
        db.commit()
        
        service = ConversationRAGService(db, mock_retrieval_service, mock_llm_service)
        
        request = ConversationRAGRequest(
            query="Follow-up question",
            conversation_id=test_conversation.id,
            user_id=test_user.id,
            actor_type="user"
        )
        
        response = service.answer(request)
        
        # Turn이 생성되었는지 확인
        assert response.turn_id is not None
        
        # 데이터베이스에 저장되었는지 확인
        created_turn = db.query(Turn).filter(Turn.id == response.turn_id).first()
        assert created_turn is not None
        assert created_turn.conversation_id == test_conversation.id
        assert created_turn.turn_number == 1
        assert created_turn.user_message == "Follow-up question"
        assert created_turn.assistant_response == "This is a test answer"
    
    def test_multi_turn_nonexistent_conversation(
        self, mock_retrieval_service, mock_llm_service, test_user, db: Session
    ):
        """존재하지 않는 conversation_id 제공 시 ValueError"""
        
        service = ConversationRAGService(db, mock_retrieval_service, mock_llm_service)
        
        request = ConversationRAGRequest(
            query="Test query",
            conversation_id=uuid4(),  # 존재하지 않는 ID
            user_id=test_user.id
        )
        
        with pytest.raises(ValueError, match="Conversation not found"):
            service.answer(request)
    
    def test_multi_turn_permission_denied(
        self, mock_retrieval_service, mock_llm_service, db: Session
    ):
        """권한 없는 사용자가 conversation 접근 시 PermissionError"""
        
        # 사용자 1이 소유한 conversation
        user1 = User(
            id=uuid4(),
            username="user1",
            organization_id=uuid4()
        )
        conversation = Conversation(
            id=uuid4(),
            owner_id=user1.id,
            organization_id=user1.organization_id,
            title="User1's Conversation",
            status="active"
        )
        
        # 사용자 2가 접근 시도
        user2 = User(
            id=uuid4(),
            username="user2",
            organization_id=uuid4()
        )
        
        db.add(user1)
        db.add(conversation)
        db.add(user2)
        db.commit()
        
        service = ConversationRAGService(db, mock_retrieval_service, mock_llm_service)
        
        request = ConversationRAGRequest(
            query="Test query",
            conversation_id=conversation.id,
            user_id=user2.id
        )
        
        with pytest.raises(PermissionError, match="does not have access"):
            service.answer(request)

class TestActorTypeLogging:
    """actor_type 감사 로그 기록 테스트 (S2 원칙 ⑤)"""
    
    @patch('src.core.audit.log_audit_event')
    def test_actor_type_in_audit_log(
        self, mock_log_audit, mock_retrieval_service, mock_llm_service, test_user, db: Session
    ):
        """actor_type이 감사 로그에 기록되는지 확인"""
        
        service = ConversationRAGService(db, mock_retrieval_service, mock_llm_service)
        
        request = ConversationRAGRequest(
            query="Test query",
            user_id=test_user.id,
            actor_type="agent"
        )
        
        service.answer(request)
        
        # log_audit_event가 호출되었는지 확인
        mock_log_audit.assert_called_once()
        call_kwargs = mock_log_audit.call_args[1]
        assert call_kwargs["actor_type"] == "agent"

class TestACLFiltering:
    """ACL 필터링 테스트 (S2 원칙 ⑥)"""
    
    @patch('src.core.security.enforce_acl_filter')
    def test_acl_filter_applied(
        self, mock_acl_filter, mock_retrieval_service, mock_llm_service, test_user, db: Session
    ):
        """검색 결과에 ACL 필터링이 적용되는지 확인"""
        
        mock_acl_filter.return_value = [{"filtered": True}]
        
        service = ConversationRAGService(db, mock_retrieval_service, mock_llm_service)
        
        request = ConversationRAGRequest(
            query="Test query",
            user_id=test_user.id
        )
        
        service.answer(request)
        
        # enforce_acl_filter가 호출되었는지 확인
        mock_acl_filter.assert_called_once()

class TestFallbackToLocalModel:
    """폐쇄망 fallback 테스트 (S2 원칙 ⑦)"""
    
    @patch.dict('os.environ', {'EXTERNAL_DEPENDENCIES_ENABLED': 'false'})
    def test_fallback_when_external_disabled(
        self, mock_retrieval_service, mock_llm_service, test_user, db: Session
    ):
        """외부 LLM 비활성화 시 로컬 모델 fallback"""
        
        mock_llm_service.generate_answer_local.return_value = (
            "Local answer",
            []
        )
        
        # ConversationRAGService를 새로 생성하면 EXTERNAL_DEPENDENCIES_ENABLED 값을 읽음
        service = ConversationRAGService(db, mock_retrieval_service, mock_llm_service)
        service.external_deps_enabled = False
        
        request = ConversationRAGRequest(
            query="Test query",
            user_id=test_user.id
        )
        
        response = service.answer(request)
        
        # 로컬 모델이 호출되었는지 확인
        mock_llm_service.generate_answer_local.assert_called_once()
```

**요구사항**:
- pytest framework 사용
- fixture를 통해 공통 객체 재사용
- Mock 객체로 의존성 주입
- 각 test case는 단일 책임 원칙 준수
- 경계값 테스트 (top_k 최소/최대값 등)

---

## 5. 산출물

1. **Request/Response Pydantic 스키마** (`backend/src/api/schemas/conversation_rag.py`)
   - ConversationRAGRequest
   - ConversationRAGResponse
   - CitationSchema
   - RetrievalMetadataSchema

2. **ConversationRAGService 클래스** (`backend/src/services/conversation_rag_service.py`)
   - answer() 메서드: 멀티턴/단발 분기 처리
   - _handle_multi_turn() 메서드: Turn 생성 및 저장
   - _has_conversation_access() 메서드: ACL 검증

3. **FastAPI 엔드포인트** (`backend/src/api/routes/rag.py`)
   - POST /api/v1/rag/answer

4. **단위 테스트** (`backend/tests/test_conversation_rag.py`)
   - 단발 모드 테스트 (4개)
   - 멀티턴 모드 테스트 (3개)
   - actor_type 로깅 테스트 (1개)
   - ACL 필터링 테스트 (1개)
   - Fallback 테스트 (1개)
   - **총 10개 이상의 test case**

---

## 6. 완료 기준

- [x] ConversationRAGRequest/Response 스키마가 Pydantic으로 검증 가능한가
- [x] ConversationRAGService가 conversation_id 유무 분기를 명확하게 처리하는가
- [x] 단발 모드 (conversation_id=null)에서 기존 /rag/answer 동작과 동일한가
- [x] 멀티턴 모드 (conversation_id 제공)에서 새 Turn 자동 생성되는가
- [x] Turn의 user_message, assistant_response, retrieval_metadata가 올바르게 저장되는가
- [x] 하위호환성 테스트 (필드 누락 시 기본값 적용)
- [x] actor_type이 감사 로그에 기록되는가 (S2 원칙 ⑤)
- [x] ACL 필터링이 검색 결과에 적용되는가 (S2 원칙 ⑥)
- [x] 외부 LLM 비활성화 시 로컬 모델 fallback이 동작하는가 (S2 원칙 ⑦)
- [x] 모든 단위 테스트 통과 (pytest 명령 실행)
- [x] 모든 API 응답이 ConversationRAGResponse 스키마를 준수하는가

---

## 7. 작업 지침

### 7-1. Pydantic 스키마 검증 강화
- `@field_validator` 데코레이터 사용하여 커스텀 검증 로직 추가
- query 길이 제한 (최대 5000자)
- actor_type regex 패턴 검증 ("user" 또는 "agent")
- top_k, context_window_size 범위 검증

### 7-2. 데이터베이스 트랜잭션 안정성
- Turn 생성 시 `db.flush()` 사용하여 ID 생성 후 `db.commit()`
- 예외 발생 시 `db.rollback()` 처리
- 모든 database 연산은 try-except 블록으로 감싸기

### 7-3. 에러 처리 표준화
- 사용자 입력 오류: HTTPException 400
- 리소스 미존재: HTTPException 404
- 권한 오류: HTTPException 403
- 서버 에러: HTTPException 500 + logging

### 7-4. 로깅 및 감사
- logging.debug: 변수값, 함수 진입/탈출
- logging.info: 의미 있는 이벤트 (Turn 생성, ACL 검증 등)
- logging.warning: 폐쇄망 fallback, 권한 실패 등
- logging.error: 예외 발생

### 7-5. S2 원칙 구현 체크리스트
- [ ] S2-⑤: actor_type 필드 추가, 감사 로그 기록
- [ ] S2-⑥: Scope Profile 기반 ACL 검증, enforce_acl_filter 호출
- [ ] S2-⑦: EXTERNAL_DEPENDENCIES_ENABLED 환경변수 확인, fallback 구현

### 7-6. 코드 리뷰 포인트
- ConversationRAGService의 책임이 단일한가 (SRP)
- 매직 넘버는 상수로 정의되었는가
- 모든 public 메서드에 docstring이 있는가
- 순환 참조(circular import)가 없는가

### 7-7. 통합 테스트 (선택사항)
- FastAPI 엔드포인트의 request/response 통합 테스트
- 실제 데이터베이스를 사용한 엔드-투-엔드 테스트
- 성능 테스트: 응답 시간 200ms 이내 (검색 시간 제외)

---

## 참고 자료

- **Phase 2 Citation 5-tuple 정의**: `/docs/개발문서/S2/phase2/citation_contract.md`
- **FG3.1 도메인 모델**: `/docs/개발문서/S2/phase3/domain_model.md`
- **Phase 1 LLM 추상화**: `/docs/개발문서/S2/phase1/llm_abstraction.md`
- **S2 원칙**: `/CLAUDE.md` (Mimir 프로젝트 설정)
