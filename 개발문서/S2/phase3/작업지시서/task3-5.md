# Task 3-5: 멀티턴 컨텍스트 윈도우 + 쿼리 재작성 통합

## 1. 작업 목적

멀티턴 대화에서 이전 턴들의 컨텍스트를 효율적으로 관리하고, Phase 2에서 구축한 쿼리 재작성 로직을 통합하여:
- 프롬프트 토큰 수를 계산하고 오버플로우 상황 처리
- 최근 N개 Turn만 유지하여 응답 시간 최적화
- Follow-up 쿼리를 자립적 쿼리로 재작성하여 검색 품질 향상
- 원문과 재작성된 쿼리 모두를 메타데이터에 저장

---

## 2. 작업 범위

### 포함 항목

1. **ContextWindowManager 클래스 설계 및 구현**
   - Conversation에서 최근 N개 Turn 조회
   - Phase 1 count_tokens() 사용하여 토큰 계산
   - 컨텍스트 윈도우 오버플로우 감지 및 처리
   - 오래된 Turn 요약 또는 제거 전략

2. **토큰 카운팅 정확도 향상**
   - system prompt + 최근 N턴의 user_message + assistant_response 토큰 계산
   - 모델별로 다른 토큰 계산 함수 호출 (Phase 1)
   - 버퍼 크기 설정 (예: 최대 토큰의 90% 사용 권장)

3. **쿼리 재작성 통합 (Phase 2 FG2.3)**
   - Phase 2에서 구축한 QueryRewriterService 호출
   - Follow-up query → standalone query 변환
   - 컨텍스트 윈도우의 이전 턴 정보 전달

4. **Prompt 구성 최적화**
   - 시스템 프롬프트 (고정)
   - 컨텍스트 윈도우의 이전 턴 대화 이력 (동적)
   - 현재 쿼리 (필수)
   - 검색 결과 (필수)
   - 각 섹션 토큰 계산 분리

5. **Turn 메타데이터 저장**
   - query_rewritten: 재작성된 쿼리 문자열
   - retrieval_time_ms: 검색 시간
   - context_window_turns: 컨텍스트 윈도우에 포함된 Turn ID 리스트
   - token_count: 이번 턴의 총 토큰 수 (선택)

6. **S2 원칙 반영**
   - S2-⑥: ACL 필터링이 컨텍스트 윈도우 Turn 조회에도 적용
   - 사용자는 자신의 conversation의 Turn만 조회 가능

7. **단위 테스트**
   - `test_context_window_fetch_recent_n_turns()`: 최근 N개 Turn 조회
   - `test_token_counting_single_turn()`: 단일 턴 토큰 계산
   - `test_token_counting_multi_turn()`: 멀티턴 토큰 계산 누적
   - `test_context_window_overflow_handling()`: 오버플로우 시 오래된 Turn 제거
   - `test_query_rewriting_integration()`: Phase 2 쿼리 재작성 통합
   - `test_prompt_construction_with_context()`: 컨텍스트 포함 prompt 구성
   - `test_turn_metadata_save()`: retrieval_metadata 정확하게 저장
   - `test_acl_filtering_in_context_window()`: 컨텍스트 윈도우 ACL 필터링
   - **총 8개 이상의 test case**

### 제외 항목

- RAG API 엔드포인트 (Task 3-4 참고)
- Citation 재활용 및 캐싱 (Task 3-6)
- 성능 벤치마크 (Task 3-6)
- UI 구현 (Task 3-3)

---

## 3. 선행 조건

1. **Task 3-4 완료**
   - ConversationRAGRequest/Response 스키마
   - ConversationRAGService 기본 구조

2. **Phase 2 FG2.3 완료**
   - QueryRewriterService 구현
   - 쿼리 재작성 로직

3. **Phase 1 완료**
   - LLM 모델 추상화 (count_tokens 함수)
   - 기본 모델 및 모델 선택 로직

4. **FG3.1 완료**
   - Conversation, Turn 도메인 모델
   - 데이터베이스 스키마

---

## 4. 주요 작업 항목

### 4-1. ContextWindowManager 클래스 설계

**파일**: `backend/src/services/context_window_manager.py`

```python
from typing import Optional, List, Dict, Any, Tuple
from uuid import UUID
from datetime import datetime
import logging
from sqlalchemy.orm import Session

from src.models import Conversation, Turn, User
from src.core.llm import count_tokens
from src.core.config import settings

logger = logging.getLogger(__name__)

class ContextWindowManager:
    """멀티턴 대화의 컨텍스트 윈도우 관리"""
    
    # 기본 설정
    DEFAULT_CONTEXT_WINDOW_SIZE = 3  # 최근 N개 턴
    MAX_CONTEXT_TOKENS = 4000  # 컨텍스트용 최대 토큰 (모델별로 다를 수 있음)
    BUFFER_RATIO = 0.9  # 최대 토큰의 90% 사용 권장
    
    def __init__(self, db: Session, model: Optional[str] = None):
        """
        Args:
            db: SQLAlchemy Session
            model: LLM 모델명 (기본값: Phase 1에서 설정된 기본 모델)
        """
        self.db = db
        self.model = model or settings.DEFAULT_LLM_MODEL
    
    def fetch_context_window(
        self,
        conversation_id: UUID,
        user: User,
        window_size: int = DEFAULT_CONTEXT_WINDOW_SIZE
    ) -> Tuple[List[Turn], int]:
        """
        Conversation에서 최근 N개 Turn을 조회 (ACL 필터링 포함)
        
        Args:
            conversation_id: Conversation ID
            user: User 객체 (ACL 검증용)
            window_size: 조회할 최근 턴 수 (기본: 3)
        
        Returns:
            Tuple[최근 N개 Turn 리스트, 누적 토큰 수]
        
        Raises:
            ValueError: conversation이 존재하지 않을 경우
            PermissionError: user가 conversation에 접근 권한 없을 경우
        """
        
        # Conversation 조회 및 ACL 검증 (S2 원칙 ⑥)
        conversation = self.db.query(Conversation).filter(
            Conversation.id == conversation_id
        ).first()
        
        if not conversation:
            raise ValueError(f"Conversation not found: {conversation_id}")
        
        # ACL 검증: user가 conversation 소유자인지 확인
        if conversation.owner_id != user.id:
            raise PermissionError(
                f"User {user.id} does not have access to conversation {conversation_id}"
            )
        
        # 최근 N개 Turn 조회 (역순 = 최신부터 오래된 순)
        all_turns = self.db.query(Turn).filter(
            Turn.conversation_id == conversation_id
        ).order_by(Turn.turn_number.desc()).all()
        
        # 최근 N개만 선택
        recent_turns = sorted(all_turns[:window_size], key=lambda t: t.turn_number)
        
        # 토큰 계산
        total_tokens = self._calculate_context_tokens(recent_turns)
        
        logger.info(
            f"Context window fetched: conversation_id={conversation_id}, "
            f"turns={len(recent_turns)}, tokens={total_tokens}"
        )
        
        return recent_turns, total_tokens
    
    def build_context_string(self, turns: List[Turn]) -> str:
        """
        Turn 리스트를 컨텍스트 문자열로 변환
        
        Args:
            turns: Turn 객체 리스트 (turn_number 순서)
        
        Returns:
            포맷된 컨텍스트 문자열
        """
        
        if not turns:
            return ""
        
        context_lines = []
        for turn in turns:
            context_lines.append(f"Turn {turn.turn_number}:")
            context_lines.append(f"  User: {turn.user_message}")
            context_lines.append(f"  Assistant: {turn.assistant_response}")
            context_lines.append("")
        
        return "\n".join(context_lines)
    
    def manage_overflow(
        self,
        conversation_id: UUID,
        user: User,
        query: str,
        search_results: List[Dict[str, Any]],
        model: Optional[str] = None,
        max_tokens: Optional[int] = None
    ) -> Tuple[List[Turn], List[Dict[str, Any]], Dict[str, Any]]:
        """
        토큰 오버플로우 상황에서 컨텍스트 윈도우 동적 조정
        
        Args:
            conversation_id: Conversation ID
            user: User 객체
            query: 현재 쿼리
            search_results: 검색 결과
            model: 사용할 LLM 모델
            max_tokens: 최대 토큰 제한 (기본: MAX_CONTEXT_TOKENS)
        
        Returns:
            Tuple[조정된 Turn 리스트, 조정된 검색결과, 메타데이터]
        """
        
        if max_tokens is None:
            max_tokens = int(self.MAX_CONTEXT_TOKENS * self.BUFFER_RATIO)
        
        model = model or self.model
        
        # 초기 컨텍스트 윈도우 (최근 3턴)
        context_turns, context_tokens = self.fetch_context_window(
            conversation_id, user, self.DEFAULT_CONTEXT_WINDOW_SIZE
        )
        
        # 모든 컴포넌트의 토큰 계산
        query_tokens = count_tokens(query, model)
        search_context_tokens = self._calculate_search_context_tokens(search_results, model)
        prompt_overhead = 100  # system prompt, formatting 등
        
        # 필요한 토큰 계산
        required_tokens = query_tokens + search_context_tokens + prompt_overhead
        
        # 오버플로우 감지
        while context_tokens + required_tokens > max_tokens and context_turns:
            # 가장 오래된 Turn 제거
            removed_turn = context_turns.pop(0)
            context_tokens -= count_tokens(
                removed_turn.user_message + removed_turn.assistant_response,
                model
            )
            logger.warning(
                f"Context window overflow detected. Removed Turn {removed_turn.turn_number}. "
                f"Remaining tokens: {context_tokens + required_tokens}/{max_tokens}"
            )
        
        # 검색 결과도 토큰 제한에 맞게 조정 (선택사항)
        adjusted_search_results = self._trim_search_results(
            search_results, max_tokens - context_tokens - query_tokens - prompt_overhead, model
        )
        
        metadata = {
            "context_overflow_handled": context_tokens + required_tokens > max_tokens,
            "context_tokens": context_tokens,
            "query_tokens": query_tokens,
            "search_tokens": self._calculate_search_context_tokens(adjusted_search_results, model),
            "total_tokens": context_tokens + query_tokens + self._calculate_search_context_tokens(adjusted_search_results, model)
        }
        
        return context_turns, adjusted_search_results, metadata
    
    def _calculate_context_tokens(self, turns: List[Turn], model: Optional[str] = None) -> int:
        """
        Turn 리스트의 토큰 계산
        
        Args:
            turns: Turn 객체 리스트
            model: 사용할 LLM 모델
        
        Returns:
            누적 토큰 수
        """
        
        model = model or self.model
        total = 0
        
        for turn in turns:
            total += count_tokens(turn.user_message, model)
            total += count_tokens(turn.assistant_response, model)
        
        return total
    
    def _calculate_search_context_tokens(self, search_results: List[Dict[str, Any]], model: str) -> int:
        """
        검색 결과의 토큰 계산
        
        Args:
            search_results: 검색 결과 리스트
            model: 사용할 LLM 모델
        
        Returns:
            누적 토큰 수
        """
        
        total = 0
        for result in search_results:
            content = result.get("content", "")
            total += count_tokens(content, model)
        
        return total
    
    def _trim_search_results(
        self,
        search_results: List[Dict[str, Any]],
        max_tokens: int,
        model: str
    ) -> List[Dict[str, Any]]:
        """
        검색 결과를 토큰 제한에 맞게 조정
        
        Args:
            search_results: 원본 검색 결과
            max_tokens: 최대 토큰
            model: 사용할 LLM 모델
        
        Returns:
            조정된 검색 결과
        """
        
        trimmed = []
        total_tokens = 0
        
        for result in search_results:
            content = result.get("content", "")
            content_tokens = count_tokens(content, model)
            
            if total_tokens + content_tokens <= max_tokens:
                trimmed.append(result)
                total_tokens += content_tokens
            else:
                logger.info(f"Search results trimmed. Remaining tokens: {max_tokens - total_tokens}")
                break
        
        return trimmed
```

**요구사항**:
- fetch_context_window: 최근 N개 Turn 조회, ACL 검증 포함
- manage_overflow: 토큰 오버플로우 감지 및 자동 조정
- 모든 토큰 계산은 Phase 1의 count_tokens() 함수 사용
- 로깅으로 주요 이벤트 기록

---

### 4-2. 쿼리 재작성 통합

**파일**: `backend/src/services/conversation_rag_service.py` 확장 (Task 3-4에서 생성한 파일)

```python
# 기존 ConversationRAGService에 다음 메서드 추가

from src.services.query_rewriter_service import QueryRewriterService  # Phase 2

class ConversationRAGService:
    
    def __init__(
        self,
        db: Session,
        retrieval_service,
        llm_service,
        query_rewriter_service: Optional[QueryRewriterService] = None,
        context_manager: Optional[ContextWindowManager] = None
    ):
        # 기존 초기화 코드
        self.query_rewriter = query_rewriter_service or QueryRewriterService(db)
        self.context_manager = context_manager or ContextWindowManager(db)
    
    def answer(self, request: ConversationRAGRequest) -> ConversationRAGResponse:
        """
        기존 answer() 메서드를 확장하여 쿼리 재작성 + 컨텍스트 윈도우 통합
        """
        
        # ... (Task 3-4의 기존 코드) ...
        
        # 2-5. 쿼리 재작성 (멀티턴 모드)
        query_to_search = request.query
        query_rewritten = None
        context_turns = []
        context_metadata = {}
        
        if request.conversation_id:
            # 컨텍스트 윈도우 관리
            context_turns, search_results_adjusted, context_metadata = self.context_manager.manage_overflow(
                conversation_id=request.conversation_id,
                user=user,
                query=request.query,
                search_results=[],  # 검색 전이므로 임시값
                model=request.model
            )
            
            # 컨텍스트 문자열 구성
            context_string = self.context_manager.build_context_string(context_turns)
            
            # Phase 2 쿼리 재작성: follow-up query → standalone query
            query_rewritten = self.query_rewriter.rewrite(
                original_query=request.query,
                conversation_context=context_string,
                conversation_id=request.conversation_id
            )
            
            query_to_search = query_rewritten if query_rewritten else request.query
            logger.info(
                f"Query rewritten: original='{request.query}' → rewritten='{query_to_search}'"
            )
        
        # 3. 검색 수행 (Task 3-4와 동일)
        start_time = datetime.utcnow()
        retrieval_results = self.retrieval.search(
            query=query_to_search,
            top_k=request.top_k,
            user_scope=user.scope_profile
        )
        # ... (ACL 필터링 등) ...
        
        # ... (LLM 응답 생성) ...
        
        # 6. Turn 저장 (메타데이터 포함)
        if turn:
            turn.assistant_response = answer
            turn.retrieval_metadata = {
                "query_rewritten": query_rewritten,  # ← 재작성 쿼리 저장
                "retrieval_time_ms": retrieval_time_ms,
                "citations": [c.dict() for c in citations],
                "context_window_turns": [str(t.id) for t in context_turns],  # ← 컨텍스트 윈도우 Turn ID 저장
                "context_metadata": context_metadata
            }
            self.db.commit()
```

**요구사항**:
- QueryRewriterService (Phase 2)와 통합
- 컨텍스트 문자열을 쿼리 재작성에 전달
- 원문 쿼리와 재작성 쿼리 모두 메타데이터에 저장
- 컨텍스트 윈도우에 포함된 Turn ID 리스트 저장

---

### 4-3. Prompt 구성 최적화

**파일**: `backend/src/services/prompt_builder.py` (새 파일)

```python
from typing import List, Dict, Any, Optional
from src.models import Turn
from src.core.llm import count_tokens

class PromptBuilder:
    """RAG 프롬프트 구성 및 토큰 계산"""
    
    SYSTEM_PROMPT = """당신은 지식 기반 질문-답변 시스템입니다.
주어진 검색 결과를 기반으로 정확하고 간결한 답변을 제공하세요.
인용 출처를 명확히 표시하세요.
모르는 것은 "정보가 없습니다"라고 답하세요."""
    
    def __init__(self, model: str):
        self.model = model
    
    def build_prompt(
        self,
        query: str,
        search_results: List[Dict[str, Any]],
        context_turns: Optional[List[Turn]] = None
    ) -> str:
        """
        전체 프롬프트 구성
        
        Args:
            query: 현재 쿼리
            search_results: 검색 결과
            context_turns: 컨텍스트 윈도우의 Turn 리스트 (선택)
        
        Returns:
            최종 프롬프트 문자열
        """
        
        prompt_parts = [self.SYSTEM_PROMPT, ""]
        
        # 컨텍스트 윈도우 추가
        if context_turns:
            prompt_parts.append("=== 대화 이력 ===")
            for turn in context_turns:
                prompt_parts.append(f"User: {turn.user_message}")
                prompt_parts.append(f"Assistant: {turn.assistant_response}")
            prompt_parts.append("")
        
        # 현재 쿼리
        prompt_parts.append("=== 현재 질문 ===")
        prompt_parts.append(query)
        prompt_parts.append("")
        
        # 검색 결과
        prompt_parts.append("=== 참고 자료 ===")
        for i, result in enumerate(search_results, 1):
            prompt_parts.append(f"[{i}] {result.get('content', '')}")
        prompt_parts.append("")
        
        # 지시사항
        prompt_parts.append("=== 응답 ===")
        prompt_parts.append("위 참고 자료를 기반으로 질문에 답하고, [숫자]로 출처를 표시하세요.")
        
        return "\n".join(prompt_parts)
    
    def count_prompt_tokens(
        self,
        query: str,
        search_results: List[Dict[str, Any]],
        context_turns: Optional[List[Turn]] = None
    ) -> Dict[str, int]:
        """
        프롬프트의 각 섹션 토큰 계산
        
        Args:
            query: 현재 쿼리
            search_results: 검색 결과
            context_turns: 컨텍스트 윈도우 Turn 리스트
        
        Returns:
            섹션별 토큰 수 딕셔너리
        """
        
        system_tokens = count_tokens(self.SYSTEM_PROMPT, self.model)
        query_tokens = count_tokens(query, self.model)
        
        context_tokens = 0
        if context_turns:
            for turn in context_turns:
                context_tokens += count_tokens(turn.user_message, self.model)
                context_tokens += count_tokens(turn.assistant_response, self.model)
        
        search_tokens = 0
        for result in search_results:
            search_tokens += count_tokens(result.get("content", ""), self.model)
        
        return {
            "system": system_tokens,
            "context": context_tokens,
            "query": query_tokens,
            "search": search_tokens,
            "total": system_tokens + context_tokens + query_tokens + search_tokens
        }
```

**요구사항**:
- 프롬프트 구성을 명확한 섹션으로 나눔 (시스템, 컨텍스트, 쿼리, 검색 결과)
- 각 섹션의 토큰을 독립적으로 계산
- 프롬프트 포맷은 LLM이 이해하기 쉽도록 구성

---

### 4-4. 단위 테스트 작성

**파일**: `backend/tests/test_context_window_manager.py`

```python
import pytest
from uuid import uuid4
from datetime import datetime
from sqlalchemy.orm import Session
from unittest.mock import patch, Mock

from src.services.context_window_manager import ContextWindowManager
from src.models import Conversation, Turn, User
from src.core.exceptions import PermissionError as PermissionDenied

# ===== Fixtures =====

@pytest.fixture
def test_user():
    return User(
        id=uuid4(),
        username="testuser",
        organization_id=uuid4()
    )

@pytest.fixture
def test_conversation(test_user):
    return Conversation(
        id=uuid4(),
        owner_id=test_user.id,
        organization_id=test_user.organization_id,
        title="Test Conversation",
        created_at=datetime.utcnow(),
        status="active"
    )

@pytest.fixture
def mock_count_tokens():
    """Mock count_tokens 함수"""
    def _count(text: str, model: str) -> int:
        # 간단한 구현: 평균 4글자 = 1토큰
        return len(text) // 4 + 1
    
    with patch('src.core.llm.count_tokens', side_effect=_count) as mock:
        yield mock

# ===== Test Cases =====

class TestContextWindowFetch:
    """컨텍스트 윈도우 조회 테스트"""
    
    def test_fetch_recent_turns(self, test_user, test_conversation, db: Session, mock_count_tokens):
        """최근 N개 Turn 조회"""
        
        db.add(test_user)
        db.add(test_conversation)
        db.commit()
        
        # 5개 Turn 생성
        for i in range(1, 6):
            turn = Turn(
                conversation_id=test_conversation.id,
                turn_number=i,
                user_message=f"Question {i}",
                assistant_response=f"Answer {i}",
                created_at=datetime.utcnow()
            )
            db.add(turn)
        db.commit()
        
        manager = ContextWindowManager(db)
        turns, tokens = manager.fetch_context_window(
            test_conversation.id, test_user, window_size=3
        )
        
        # 최근 3개만 조회되어야 함
        assert len(turns) == 3
        assert turns[0].turn_number == 3
        assert turns[1].turn_number == 4
        assert turns[2].turn_number == 5
        assert tokens > 0
    
    def test_fetch_with_permission_error(self, test_user, db: Session):
        """권한 없는 사용자 접근 시 PermissionError"""
        
        # 다른 사용자가 소유한 conversation
        other_user = User(id=uuid4(), username="other", organization_id=uuid4())
        conversation = Conversation(
            id=uuid4(),
            owner_id=other_user.id,
            organization_id=other_user.organization_id,
            title="Other's Conversation",
            status="active"
        )
        
        db.add(other_user)
        db.add(conversation)
        db.commit()
        
        manager = ContextWindowManager(db)
        
        with pytest.raises(PermissionError, match="does not have access"):
            manager.fetch_context_window(conversation.id, test_user, window_size=3)

class TestTokenCounting:
    """토큰 계산 테스트"""
    
    def test_single_turn_token_count(self, mock_count_tokens):
        """단일 턴 토큰 계산"""
        
        turn = Turn(
            conversation_id=uuid4(),
            turn_number=1,
            user_message="What is AI?" * 10,  # 약 40개 단어
            assistant_response="AI is..." * 20,  # 약 60개 단어
            created_at=datetime.utcnow()
        )
        
        db = Mock()
        manager = ContextWindowManager(db)
        
        tokens = manager._calculate_context_tokens([turn])
        assert tokens > 0
    
    def test_multi_turn_token_accumulation(self, mock_count_tokens):
        """멀티턴 토큰 누적 계산"""
        
        turns = [
            Turn(
                conversation_id=uuid4(),
                turn_number=i,
                user_message=f"Q{i} " * 20,
                assistant_response=f"A{i} " * 30,
                created_at=datetime.utcnow()
            )
            for i in range(1, 4)
        ]
        
        db = Mock()
        manager = ContextWindowManager(db)
        
        tokens = manager._calculate_context_tokens(turns)
        assert tokens > 0

class TestContextWindowOverflow:
    """컨텍스트 윈도우 오버플로우 테스트"""
    
    @patch('src.services.context_window_manager.ContextWindowManager.fetch_context_window')
    def test_overflow_handling_removes_old_turns(self, mock_fetch, mock_count_tokens):
        """오버플로우 시 오래된 Turn 제거"""
        
        # 3개 turn 생성 (각 turn은 많은 토큰)
        turns = [
            Turn(
                conversation_id=uuid4(),
                turn_number=i,
                user_message=f"Q{i} " * 500,  # 높은 토큰
                assistant_response=f"A{i} " * 500,
                created_at=datetime.utcnow()
            )
            for i in range(1, 4)
        ]
        
        mock_fetch.return_value = (turns, 5000)  # 5000 토큰 (오버플로우)
        
        db = Mock()
        db.query = Mock()
        manager = ContextWindowManager(db)
        manager.MAX_CONTEXT_TOKENS = 3000
        
        adjusted_turns, _, metadata = manager.manage_overflow(
            conversation_id=uuid4(),
            user=Mock(),
            query="Test",
            search_results=[],
            max_tokens=2000
        )
        
        # 오버플로우 처리됨
        assert metadata["context_overflow_handled"] or len(adjusted_turns) <= 2

class TestQueryRewritingIntegration:
    """쿼리 재작성 통합 테스트 (Task 3-5)"""
    
    @patch('src.services.conversation_rag_service.QueryRewriterService')
    def test_query_rewriting_called(self, mock_rewriter_service):
        """Phase 2 쿼리 재작성 통합"""
        
        # 이 테스트는 Task 3-4의 ConversationRAGService에서 수행
        # 여기서는 개념 검증만 진행
        
        mock_rewriter = Mock()
        mock_rewriter.rewrite.return_value = "rewritten query"
        mock_rewriter_service.return_value = mock_rewriter
        
        # 실제 구현에서는 answer() 메서드 호출 시
        # query_rewriter.rewrite()가 호출되어야 함

class TestPromptConstruction:
    """프롬프트 구성 테스트"""
    
    def test_prompt_with_context_turns(self):
        """컨텍스트 포함 프롬프트 구성"""
        
        from src.services.prompt_builder import PromptBuilder
        
        builder = PromptBuilder(model="gpt-4")
        
        turns = [
            Turn(
                turn_number=1,
                user_message="What is AI?",
                assistant_response="AI is...",
                created_at=datetime.utcnow()
            )
        ]
        
        prompt = builder.build_prompt(
            query="How to learn AI?",
            search_results=[{"content": "AI fundamentals..."}],
            context_turns=turns
        )
        
        assert "대화 이력" in prompt
        assert "What is AI?" in prompt
        assert "How to learn AI?" in prompt
        assert "AI fundamentals..." in prompt
    
    @patch('src.core.llm.count_tokens')
    def test_prompt_token_counting(self, mock_count_tokens):
        """프롬프트 토큰 계산"""
        
        from src.services.prompt_builder import PromptBuilder
        
        mock_count_tokens.return_value = 100  # 모든 호출에 100 반환
        
        builder = PromptBuilder(model="gpt-4")
        
        token_counts = builder.count_prompt_tokens(
            query="Test query",
            search_results=[{"content": "result 1"}],
            context_turns=None
        )
        
        assert "system" in token_counts
        assert "query" in token_counts
        assert "search" in token_counts
        assert "total" in token_counts
        assert token_counts["total"] >= token_counts["system"]

class TestTurnMetadataSave:
    """Turn 메타데이터 저장 테스트"""
    
    def test_retrieval_metadata_structure(self, test_user, test_conversation, db: Session):
        """Turn에 retrieval_metadata가 올바르게 저장되는지 확인"""
        
        db.add(test_user)
        db.add(test_conversation)
        db.commit()
        
        turn = Turn(
            conversation_id=test_conversation.id,
            turn_number=1,
            user_message="Test query",
            assistant_response="Test answer",
            retrieval_metadata={
                "query_rewritten": "rewritten query",
                "retrieval_time_ms": 145,
                "context_window_turns": ["turn-id-1", "turn-id-2"],
                "context_metadata": {"tokens": 800}
            },
            created_at=datetime.utcnow()
        )
        
        db.add(turn)
        db.commit()
        
        saved_turn = db.query(Turn).filter(Turn.id == turn.id).first()
        
        assert saved_turn.retrieval_metadata["query_rewritten"] == "rewritten query"
        assert saved_turn.retrieval_metadata["retrieval_time_ms"] == 145
        assert len(saved_turn.retrieval_metadata["context_window_turns"]) == 2

class TestACLFilteringInContextWindow:
    """컨텍스트 윈도우 ACL 필터링 테스트 (S2 원칙 ⑥)"""
    
    def test_acl_filtering_in_fetch_context(self, db: Session):
        """컨텍스트 윈도우 조회 시 ACL 검증"""
        
        user1 = User(id=uuid4(), username="user1", organization_id=uuid4())
        user2 = User(id=uuid4(), username="user2", organization_id=uuid4())
        conversation = Conversation(
            id=uuid4(),
            owner_id=user1.id,
            organization_id=user1.organization_id,
            title="User1 Conversation",
            status="active"
        )
        
        db.add(user1)
        db.add(user2)
        db.add(conversation)
        db.commit()
        
        manager = ContextWindowManager(db)
        
        # user1은 접근 가능
        turns, _ = manager.fetch_context_window(conversation.id, user1)
        assert turns is not None
        
        # user2는 접근 불가
        with pytest.raises(PermissionError):
            manager.fetch_context_window(conversation.id, user2)
```

**요구사항**:
- pytest framework 사용
- Mock 및 patch를 통한 의존성 주입
- 토큰 계산의 경계값 테스트
- 오버플로우 시나리오 테스트
- ACL 검증 테스트

---

## 5. 산출물

1. **ContextWindowManager 클래스** (`backend/src/services/context_window_manager.py`)
   - fetch_context_window(): 최근 N개 Turn 조회, ACL 검증
   - manage_overflow(): 토큰 오버플로우 처리
   - build_context_string(): Turn을 문자열로 변환
   - 헬퍼 메서드: 토큰 계산, 검색 결과 자르기

2. **PromptBuilder 클래스** (`backend/src/services/prompt_builder.py`)
   - build_prompt(): 전체 프롬프트 구성
   - count_prompt_tokens(): 섹션별 토큰 계산

3. **ConversationRAGService 확장** (`backend/src/services/conversation_rag_service.py`)
   - QueryRewriterService 통합
   - ContextWindowManager 통합
   - Turn 메타데이터 저장 (query_rewritten, context_window_turns 등)

4. **단위 테스트** (`backend/tests/test_context_window_manager.py`)
   - 컨텍스트 윈도우 조회 테스트 (2개)
   - 토큰 계산 테스트 (2개)
   - 오버플로우 처리 테스트 (1개)
   - 쿼리 재작성 통합 테스트 (1개)
   - 프롬프트 구성 테스트 (2개)
   - Turn 메타데이터 저장 테스트 (1개)
   - ACL 필터링 테스트 (1개)
   - **총 10개 이상의 test case**

---

## 6. 완료 기준

- [x] ContextWindowManager가 최근 N개 Turn을 조회하는가 (기본 3턴)
- [x] Phase 1 count_tokens()를 사용하여 정확한 토큰 계산하는가
- [x] 토큰 오버플로우 감지 및 자동 처리 (오래된 Turn 제거)
- [x] Phase 2 QueryRewriterService 통합되었는가
- [x] 컨텍스트 문자열이 올바르게 구성되는가
- [x] Turn 메타데이터에 query_rewritten 저장되는가
- [x] Turn 메타데이터에 context_window_turns (Turn ID 리스트) 저장되는가
- [x] 컨텍스트 윈도우 ACL 필터링 적용 (S2 원칙 ⑥)
- [x] PromptBuilder가 섹션별 프롬프트 구성하는가
- [x] 프롬프트 토큰 계산이 정확한가 (system, context, query, search 분리)
- [x] 모든 단위 테스트 통과 (pytest)
- [x] 경계값 테스트 (오버플로우 임계값 테스트)

---

## 7. 작업 지침

### 7-1. 토큰 카운팅 정확도
- Phase 1의 count_tokens()를 반드시 사용
- 모델별로 다른 토큰 계산 결과를 반영
- 버퍼 설정: 최대 토큰의 90% 사용 권장

### 7-2. 컨텍스트 윈도우 전략
- 기본값: 최근 3턴 (설정 가능)
- 오버플로우 시: 가장 오래된 Turn부터 하나씩 제거
- 모든 Turn 제거 후에도 오버플로우면: 검색 결과 자르기

### 7-3. 쿼리 재작성 통합
- Phase 2 FG2.3의 QueryRewriterService 호출
- 컨텍스트 윈도우의 대화 이력 전달
- 원문과 재작성 쿼리 모두 메타데이터에 저장

### 7-4. 메타데이터 구조
```json
{
  "query_rewritten": "재작성된 쿼리 문자열",
  "retrieval_time_ms": 145,
  "context_window_turns": ["turn-uuid-1", "turn-uuid-2"],
  "context_metadata": {
    "total_tokens": 800,
    "context_tokens": 500,
    "overflow_handled": false
  }
}
```

### 7-5. S2 원칙 체크리스트
- [ ] S2-⑥: ACL 필터링이 컨텍스트 윈도우 조회에 적용
- [ ] user가 conversation 소유자인지 검증
- [ ] 비소유자 접근 시 PermissionError 발생

### 7-6. 성능 고려사항
- 토큰 계산은 병목이 될 수 있으므로 캐싱 고려 (Task 3-6)
- 컨텍스트 윈도우 조회는 O(log N) 이하로 유지
- 대규모 대화 (1000+ 턴)에서도 응답 시간 <200ms 목표

---

## 8. 참고 자료

- **Phase 2 QueryRewriterService**: `/docs/개발문서/S2/phase2/query_rewriting.md`
- **Phase 1 LLM 추상화**: `/docs/개발문서/S2/phase1/llm_abstraction.md`
- **Task 3-4 RAG API**: `/docs/개발문서/S2/phase3/작업지시서/task3-4.md`
- **S2 원칙**: `/CLAUDE.md` (Mimir 프로젝트 설정)
