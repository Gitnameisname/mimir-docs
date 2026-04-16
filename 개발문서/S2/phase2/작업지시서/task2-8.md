# Task 2-8. 멀티턴 RAG 엔드포인트 + Citation 캐시 + 품질 평가

## 1. 작업 목적

FG2.3의 최종 통합 Task로, `POST /api/v1/rag/answer` 엔드포인트를 멀티턴 대화를 지원하도록 확장한다. `conversation_id` 파라미터 포함 시 `QueryRewriter`와 `ConversationCompressor`를 활용하고, 이전 턴의 Citation을 캐시하여 불필요한 재검색을 방지한다. 쿼리 재작성 품질 평가(테스트 세트 포함)와 챗봇 인계 메모 v2를 작성한다.

## 2. 작업 범위

### 포함 범위

1. **멀티턴 RAG 엔드포인트 확장**
   - `POST /api/v1/rag/answer` — `conversation_id` 포함 시 멀티턴 모드 활성화
   - 응답에 `rewritten_query`, `context_compressed`, `turn_number` 필드 추가
   - 단발 쿼리 모드 (기존 S1 동작) 완전 하위호환

2. **Citation 캐시 로직**
   - 각 턴의 검색 결과(Citation 리스트)를 대화 이력에 저장
   - 다음 턴에서 `QueryRewriter`가 이전 Citation 정보를 프롬프트에 포함
   - 동일 문서에 대한 불필요한 재검색 감소

3. **멀티턴 대화 이력 저장소**
   - 인메모리 캐시 (슬라이딩 윈도우 최대 10턴)
   - `conversation_id` 키로 관리

4. **쿼리 재작성 품질 평가**
   - 3~5턴 멀티턴 대화 샘플 10개 (수동 검수 데이터)
   - 평가 항목: 정보 보존, 명확성 증가, 맥락 통합
   - AI품질평가보고서 작성 기초 데이터

5. **챗봇 인계 메모 v2** (특별 산출물)
   - Citation 계약 반영
   - 멀티턴 API 계약 명세

### 제외 범위

- Phase 3 Conversation 도메인 (DB 기반 영속적 대화 저장)
- LLM 응답 품질 평가 (Phase 7)

## 3. 선행 조건

- Task 2-1~2-7 완료 (전체 FG2.1, FG2.2, FG2.3 구현)
- S1 `POST /api/v1/rag/answer` 기존 구현 분석 완료
- Phase 1 LLMProvider 사용 가능

## 4. 주요 작업 항목

### 4-1. 멀티턴 RAG 요청/응답 스키마

**파일:** `/backend/app/schemas/rag.py` (기존 파일 확장)

```python
"""RAG 스키마 — S2 멀티턴 확장.

기존 S1 RAGRequest/RAGResponse를 유지하면서
conversation_id 기반 멀티턴 필드를 추가한다.
"""
from __future__ import annotations

from typing import List, Optional
from uuid import UUID

from pydantic import BaseModel, Field

from app.schemas.citation import Citation


# ── 요청 ──────────────────────────────────────────────────────

class RAGRequest(BaseModel):
    """RAG 질의 요청.

    S1 하위호환: conversation_id 없으면 단발 쿼리 모드 (기존 동작).
    S2 확장: conversation_id 있으면 멀티턴 모드 (QueryRewriter, Citation 캐시 활성화).
    """
    query: str = Field(..., min_length=1, description="사용자 질의")
    top_k: int = Field(10, ge=1, le=50, description="검색 결과 수")
    document_type: Optional[str] = Field(None, description="검색 대상 DocumentType")
    # S2 신규 필드
    conversation_id: Optional[UUID] = Field(
        None,
        description="멀티턴 대화 ID — 제공 시 멀티턴 모드 활성화",
    )


# ── 응답 ──────────────────────────────────────────────────────

class RAGCitationInfo(BaseModel):
    """응답 내 단일 Citation 정보."""
    index: int               # 응답 텍스트에서의 인용 번호 [1], [2], ...
    citation: Citation
    snippet: str             # 인용된 청크 원문 일부


class RAGResponse(BaseModel):
    """RAG 질의 응답.

    S1 하위호환: citations에 Citation 5-tuple 포함 (기존 document_id, node_id 유지).
    S2 확장: rewritten_query, context_compressed, turn_number 필드 추가.
    """
    answer: str
    citations: List[RAGCitationInfo] = Field(default_factory=list)
    # S2 신규 필드 (S1 클라이언트 무시 가능)
    rewritten_query: Optional[str] = Field(
        None,
        description="재작성된 쿼리 (멀티턴 모드에서만 제공, 투명성)",
    )
    context_compressed: bool = Field(
        False,
        description="대화 요약 사용 여부",
    )
    turn_number: int = Field(1, description="현재 대화 턴 번호")
```

### 4-2. Citation 캐시 (인메모리)

**파일:** `/backend/app/services/retrieval/citation_cache.py`

```python
"""Citation 캐시 — 대화별 인메모리 Citation 저장.

각 턴의 검색 결과(Citation 리스트)를 저장하여
다음 턴에서 불필요한 재검색을 방지한다.

최대 10턴 슬라이딩 윈도우로 메모리 사용량을 제한한다.
"""
from __future__ import annotations

import logging
from collections import defaultdict, deque
from typing import Dict, List, Optional
from uuid import UUID

from app.schemas.citation import Citation
from app.schemas.conversation import ConversationMessage, MessageRole

logger = logging.getLogger(__name__)

_MAX_TURNS = 10  # 슬라이딩 윈도우 최대 턴 수


class ConversationTurn:
    """단일 대화 턴 기록."""

    def __init__(
        self,
        turn_number: int,
        query: str,
        rewritten_query: Optional[str],
        citations: List[Citation],
        answer: str,
    ) -> None:
        self.turn_number = turn_number
        self.query = query
        self.rewritten_query = rewritten_query
        self.citations = citations
        self.answer = answer

    def to_message_pair(self) -> List[ConversationMessage]:
        """(사용자 질의, 어시스턴트 답변) 메시지 쌍으로 변환."""
        return [
            ConversationMessage(
                role=MessageRole.USER,
                content=self.rewritten_query or self.query,
                turn_number=self.turn_number,
            ),
            ConversationMessage(
                role=MessageRole.ASSISTANT,
                content=self.answer,
                turn_number=self.turn_number,
            ),
        ]


class CitationCache:
    """대화별 Citation + 이력 인메모리 캐시."""

    def __init__(self) -> None:
        # conversation_id → deque of ConversationTurn
        self._store: Dict[str, deque] = defaultdict(lambda: deque(maxlen=_MAX_TURNS))

    def add_turn(
        self,
        conversation_id: UUID,
        turn: ConversationTurn,
    ) -> None:
        """새 턴을 저장한다."""
        self._store[str(conversation_id)].append(turn)
        logger.debug(
            "CitationCache: conversation=%s turn=%d stored",
            conversation_id,
            turn.turn_number,
        )

    def get_history(
        self,
        conversation_id: UUID,
    ) -> List[ConversationMessage]:
        """대화 이력을 ConversationMessage 리스트로 반환한다."""
        turns = self._store.get(str(conversation_id), deque())
        messages = []
        for turn in turns:
            messages.extend(turn.to_message_pair())
        return messages

    def get_citations(
        self,
        conversation_id: UUID,
    ) -> List[Citation]:
        """이전 턴에서 사용된 모든 Citation 리스트를 반환한다."""
        turns = self._store.get(str(conversation_id), deque())
        citations = []
        for turn in turns:
            citations.extend(turn.citations)
        return citations

    def get_turn_number(self, conversation_id: UUID) -> int:
        """현재 대화의 다음 턴 번호를 반환한다."""
        return len(self._store.get(str(conversation_id), [])) + 1

    def clear(self, conversation_id: UUID) -> None:
        """대화 이력을 초기화한다."""
        self._store.pop(str(conversation_id), None)


# 모듈 수준 싱글톤 (FastAPI 앱 수명 동안 유지)
_citation_cache = CitationCache()


def get_citation_cache() -> CitationCache:
    """전역 CitationCache 인스턴스를 반환한다."""
    return _citation_cache
```

### 4-3. 멀티턴 RAG 서비스

**파일:** `/backend/app/services/multiturn_rag_service.py`

```python
"""멀티턴 RAG 서비스.

단발 쿼리(S1 호환) + 멀티턴 대화(S2 확장) 모드를 모두 지원한다.
"""
from __future__ import annotations

import logging
from typing import List, Optional
from uuid import UUID

import psycopg2.extensions

from app.schemas.citation import Citation
from app.schemas.rag import RAGCitationInfo, RAGResponse
from app.services.retrieval.citation_cache import (
    CitationCache,
    ConversationTurn,
    get_citation_cache,
)
from app.services.retrieval.conversation_compressor import ConversationCompressor
from app.services.retrieval.query_rewriter import QueryRewriter
from app.services.search_service import SearchService
from app.services.rag_service import RAGService  # S1 기존 서비스

logger = logging.getLogger(__name__)


class MultiturnRAGService:
    """멀티턴 대화를 지원하는 RAG 서비스."""

    def __init__(
        self,
        conn: psycopg2.extensions.connection,
        query_rewriter: QueryRewriter,
        compressor: ConversationCompressor,
        citation_cache: CitationCache,
    ) -> None:
        self._conn = conn
        self._rewriter = query_rewriter
        self._compressor = compressor
        self._cache = citation_cache
        self._search_service = SearchService(conn)
        self._rag_service = RAGService(conn)

    async def answer(
        self,
        query: str,
        top_k: int = 10,
        document_type: Optional[str] = None,
        conversation_id: Optional[UUID] = None,
        filters: Optional[dict] = None,
    ) -> RAGResponse:
        """질의에 답변한다.

        Args:
            query: 사용자 질의
            top_k: 검색 결과 수
            document_type: 대상 DocumentType
            conversation_id: 멀티턴 대화 ID (None → 단발 쿼리 모드)
            filters: ACL 포함 필터

        Returns:
            RAGResponse
        """
        # ── 단발 쿼리 모드 (S1 하위호환) ──────────────────────
        if conversation_id is None:
            return await self._single_turn_answer(query, top_k, document_type, filters)

        # ── 멀티턴 모드 ────────────────────────────────────────
        turn_number = self._cache.get_turn_number(conversation_id)
        history = self._cache.get_history(conversation_id)

        # 1. 쿼리 재작성
        rewritten_query = await self._rewriter.rewrite_query(query, history)

        # 2. 대화 압축 (이력이 길 경우)
        context_compressed = False
        if len(history) > 6:
            await self._compressor.compress(history, strategy="summarize")
            context_compressed = True

        # 3. 검색 (재작성된 쿼리 사용)
        search_query = rewritten_query or query
        results = await self._search_service.search_with_plugins(
            query=search_query,
            document_type=document_type or "",
            top_k=top_k,
            filters=filters,
        )

        # 4. LLM 답변 생성 (S1 RAGService 활용)
        answer_text, citation_infos = await self._build_answer(search_query, results)

        # 5. 대화 이력에 이번 턴 저장
        citations = [ci.citation for ci in citation_infos]
        turn = ConversationTurn(
            turn_number=turn_number,
            query=query,
            rewritten_query=rewritten_query if rewritten_query != query else None,
            citations=citations,
            answer=answer_text,
        )
        self._cache.add_turn(conversation_id, turn)

        return RAGResponse(
            answer=answer_text,
            citations=citation_infos,
            rewritten_query=rewritten_query if rewritten_query != query else None,
            context_compressed=context_compressed,
            turn_number=turn_number,
        )

    async def _single_turn_answer(
        self,
        query: str,
        top_k: int,
        document_type: Optional[str],
        filters: Optional[dict],
    ) -> RAGResponse:
        """단발 쿼리 모드 — S1 RAGService를 직접 호출한다."""
        # 기존 S1 rag_service 로직 그대로 활용 (하위호환)
        s1_response = await self._rag_service.answer(
            query=query,
            top_k=top_k,
            document_type=document_type,
            filters=filters,
        )
        # S1 응답을 S2 RAGResponse로 래핑
        return RAGResponse(
            answer=s1_response.answer,
            citations=s1_response.citations,
            rewritten_query=None,
            context_compressed=False,
            turn_number=1,
        )

    async def _build_answer(self, query, results) -> tuple:
        """검색 결과로 LLM 답변을 생성한다. (S1 RAGService 내부 로직 재사용)"""
        # 실제 구현은 S1 rag_service의 answer generation 로직을 호출
        # 이 메서드는 구현 시 S1 코드를 분석하여 채워 넣는다
        raise NotImplementedError("S1 rag_service answer generation 로직 통합 필요")
```

### 4-4. API 엔드포인트 확장

**파일:** `/backend/app/api/v1/rag.py` (기존 파일 수정)

```python
"""RAG API — 멀티턴 확장.

기존 POST /rag/answer 엔드포인트에 conversation_id 파라미터 추가.
S1 호환: conversation_id 없으면 기존 동작 그대로.
"""
from fastapi import APIRouter, Depends, HTTPException
from app.schemas.rag import RAGRequest, RAGResponse
from app.services.multiturn_rag_service import MultiturnRAGService
from app.services.retrieval.citation_cache import get_citation_cache
from app.api.auth import get_current_actor
from app.db import get_db_conn


router = APIRouter(prefix="/rag", tags=["rag"])


@router.post(
    "/answer",
    response_model=RAGResponse,
    summary="RAG 질의응답",
    description=(
        "문서 기반 질의응답. "
        "conversation_id 제공 시 멀티턴 모드 — 쿼리 재작성 및 Citation 캐시 활성화. "
        "conversation_id 없으면 S1 단발 쿼리 모드 (하위호환)."
    ),
)
async def rag_answer(
    request: RAGRequest,
    actor=Depends(get_current_actor),
    conn=Depends(get_db_conn),
):
    from app.services.retrieval.query_rewriter import QueryRewriter
    from app.services.retrieval.conversation_compressor import ConversationCompressor
    from app.services.llm.factory import LLMProviderFactory  # Phase 1 산출물

    llm = LLMProviderFactory.create()
    rewriter = QueryRewriter(llm)
    compressor = ConversationCompressor(llm=llm)
    cache = get_citation_cache()

    svc = MultiturnRAGService(
        conn=conn,
        query_rewriter=rewriter,
        compressor=compressor,
        citation_cache=cache,
    )
    filters = {"actor_id": actor.id, "actor_role": actor.role}

    return await svc.answer(
        query=request.query,
        top_k=request.top_k,
        document_type=request.document_type,
        conversation_id=request.conversation_id,
        filters=filters,
    )
```

### 4-5. 쿼리 재작성 품질 평가 테스트 세트

**파일:** `/backend/tests/quality/test_query_rewrite_quality.py`

```python
"""쿼리 재작성 품질 평가 — 수동 검수 데이터 기반.

평가 항목:
  1. 정보 보존: 원본 질의의 핵심 정보가 재작성에 남아있는가
  2. 명확성 증가: 재작성 후 자립적인 쿼리가 되었는가
  3. 맥락 통합: 대화 맥락이 적절히 포함되었는가

각 케이스는 LLM 없이 Mock으로 실행하여 구조 검증만 수행.
실제 LLM 품질 평가는 AI품질평가보고서에서 별도 진행.
"""
import pytest
from unittest.mock import AsyncMock, MagicMock
from app.services.retrieval.query_rewriter import QueryRewriter
from app.schemas.conversation import ConversationMessage, MessageRole


MULTITURN_SAMPLES = [
    {
        "id": 1,
        "history": [
            ("user", "Kubernetes에 대해 알려줘"),
            ("assistant", "Kubernetes는 컨테이너 오케스트레이션 플랫폼입니다."),
        ],
        "original": "이게 뭐야?",
        "expected_keywords": ["Kubernetes"],
        "description": "지시어 '이게' → 이전 주제 연결",
    },
    {
        "id": 2,
        "history": [
            ("user", "Python 비동기 프로그래밍이 뭐야?"),
            ("assistant", "Python asyncio를 사용한 비동기 처리 방식입니다."),
            ("user", "asyncio와 threading의 차이는?"),
            ("assistant", "asyncio는 단일 스레드 이벤트 루프, threading은 멀티스레드입니다."),
        ],
        "original": "더 자세히 설명해줄래?",
        "expected_keywords": ["asyncio", "threading"],
        "description": "4턴 후 '더 자세히' → 직전 주제 연결",
    },
    {
        "id": 3,
        "history": [
            ("user", "FastAPI 라우터 설정 방법"),
            ("assistant", "FastAPI에서 APIRouter를 사용하여 라우터를 구성합니다."),
        ],
        "original": "예시 코드 보여줘",
        "expected_keywords": ["FastAPI", "APIRouter"],
        "description": "코드 예시 요청 → 이전 주제 연결",
    },
    {
        "id": 4,
        "history": [
            ("user", "PostgreSQL 인덱스 종류는?"),
            ("assistant", "B-tree, Hash, GIN, GiST, BRIN 등이 있습니다."),
            ("user", "GIN 인덱스가 뭐야?"),
            ("assistant", "GIN(Generalized Inverted Index)은 배열, JSONB, FTS에 최적화된 인덱스입니다."),
            ("user", "BRIN은?"),
            ("assistant", "BRIN(Block Range INdex)은 큰 테이블의 물리적 순서가 있는 데이터에 효율적입니다."),
        ],
        "original": "언제 쓰는 게 좋아?",
        "expected_keywords": ["BRIN"],
        "description": "5턴 후 '언제 쓰는 게' → 직전 주제 BRIN 연결",
    },
    {
        "id": 5,
        "history": [
            ("user", "Docker 컨테이너와 가상머신의 차이"),
            ("assistant", "Docker는 OS 레벨 가상화, VM은 하드웨어 레벨 가상화입니다."),
        ],
        "original": "어떤 게 더 빠름?",
        "expected_keywords": ["Docker", "가상머신"],
        "description": "비교 질문 → 이전 대상들 연결",
    },
]


def _make_history(pairs):
    messages = []
    for role, content in pairs:
        messages.append(ConversationMessage(
            role=MessageRole(role), content=content
        ))
    return messages


@pytest.mark.parametrize("sample", MULTITURN_SAMPLES, ids=[s["id"] for s in MULTITURN_SAMPLES])
@pytest.mark.asyncio
async def test_rewrite_structure(sample):
    """Mock LLM으로 쿼리 재작성 구조 검증 (키워드 포함 여부)."""
    # Mock LLM이 expected_keywords를 포함한 쿼리를 반환한다고 가정
    expected_rewrite = f"{' '.join(sample['expected_keywords'])} 관련 {sample['original']}"
    llm = MagicMock()
    llm.generate = AsyncMock(return_value=MagicMock(content=expected_rewrite))

    rewriter = QueryRewriter(llm)
    history = _make_history(sample["history"])
    result = await rewriter.rewrite_query(sample["original"], history)

    # LLM이 호출되었는지 확인 (대화 이력이 있으므로)
    llm.generate.assert_called_once()
    # 결과가 빈 문자열이 아닌지 확인
    assert len(result) > 0, f"[Sample {sample['id']}] 재작성 결과가 비어있음"
```

### 4-6. 챗봇 인계 메모 v2

**파일:** `/docs/개발문서/S2/phase2/산출물/chatbot-handoff-memo-v2.md`

```markdown
# 챗봇 연동 인계 메모 v2
## Citation 계약 반영 (Phase 2 완료)

---

## 1. Citation 계약

모든 검색 응답에 Citation 5-tuple이 포함됩니다.

```json
{
  "citation": {
    "document_id": "uuid",
    "version_id": "uuid",       // 인용 시점 버전
    "node_id": "uuid",          // 문서 내 섹션
    "span_offset": null,        // 단락 단위 인용 시 null
    "content_hash": "sha256hex" // 내용 변조 감지
  }
}
```

### Citation 유효성 검증

```
GET /api/v1/citations/{document_id}/versions/{version_id}/nodes/{node_id}/verify
  ?content_hash={hash}
```

응답: `{ "verified": bool, "modified": bool, "original_text": "..." }`

---

## 2. 멀티턴 RAG API

### 단발 쿼리 (S1 하위호환)

```http
POST /api/v1/rag/answer
Content-Type: application/json

{ "query": "Kubernetes 배포 방법", "top_k": 10 }
```

### 멀티턴 대화

```http
POST /api/v1/rag/answer
Content-Type: application/json

{
  "query": "이게 뭐야?",
  "top_k": 10,
  "conversation_id": "uuid"  // 멀티턴 활성화
}
```

응답:
```json
{
  "answer": "...",
  "citations": [...],
  "rewritten_query": "Kubernetes 배포 전략은?",  // 투명성
  "context_compressed": false,
  "turn_number": 2
}
```

---

## 3. 검색 전략 선택

```
GET /api/v1/search/documents?q=쿼리&retriever=hybrid&reranker=rule_based
```

- `retriever`: `fts` | `vector` | `hybrid` (기본: DocumentType 설정)
- `reranker`: `cross_encoder` | `rule_based` | `null` (기본: DocumentType 설정)

---

## 4. S1 → S2 마이그레이션

- 기존 API 호출은 변경 없이 동작
- `citation` 필드: S2 클라이언트만 활용, S1 클라이언트는 무시
- `conversation_id`: 없으면 단발 쿼리 (S1 동작), 있으면 멀티턴

---

## 5. 주요 변경 파일

| 파일 | 변경 내용 |
|------|----------|
| `/backend/app/schemas/citation.py` | Citation 5-tuple 모델 |
| `/backend/app/api/v1/citations.py` | Citation 역참조 API |
| `/backend/app/services/retrieval/` | Retriever/Reranker 플러그인 |
| `/backend/app/schemas/rag.py` | 멀티턴 RAG 스키마 |
| `/backend/app/api/v1/rag.py` | 멀티턴 RAG 엔드포인트 |
```

## 5. 산출물

1. **RAG 스키마 확장** (`/backend/app/schemas/rag.py`)
   - `RAGRequest` (conversation_id 추가), `RAGResponse` (멀티턴 필드)
2. **CitationCache** (`/backend/app/services/retrieval/citation_cache.py`)
3. **MultiturnRAGService** (`/backend/app/services/multiturn_rag_service.py`)
4. **RAG API 엔드포인트 확장** (`/backend/app/api/v1/rag.py`)
5. **품질 평가 테스트 세트** (`/backend/tests/quality/test_query_rewrite_quality.py`)
6. **챗봇 인계 메모 v2** (`/docs/개발문서/S2/phase2/산출물/chatbot-handoff-memo-v2.md`)

## 6. 완료 기준

1. 단발 쿼리 모드 (conversation_id 없음) — S1 동작과 동일
2. 멀티턴 모드 (conversation_id 있음) — QueryRewriter 호출, turn_number 증가
3. `rewritten_query` 필드가 응답에 포함됨 (재작성 시)
4. Citation 캐시 — 10턴 슬라이딩 윈도우 동작 확인
5. 멀티턴 3턴 시나리오 정상 동작 (수동 테스트)
6. 품질 평가 테스트 세트 10개 구조 검증 통과
7. 챗봇 인계 메모 v2 작성 완료
8. 통합 테스트 커버리지 ≥ 85%

## 7. 작업 지침

### 지침 7-1. S1 RAGService 재사용

`MultiturnRAGService._single_turn_answer()`는 S1 `RAGService`를 직접 호출하여 기존 로직을 재사용한다. S1 코드를 복사하지 않고 위임(delegation) 패턴을 사용한다.

### 지침 7-2. CitationCache 싱글톤 주의

`CitationCache`는 인메모리 싱글톤으로 설계한다. FastAPI가 재시작되면 대화 이력이 초기화됨을 문서화한다. Phase 3(Conversation)에서 DB 기반 영속 저장으로 교체 예정임을 주석으로 명시한다.

### 지침 7-3. 감사 로그 — actor_type 필수

멀티턴 RAG API 호출 시 감사 로그에 `actor_type` 필드를 기록한다. AI 에이전트가 `conversation_id`를 포함하여 호출하는 경우 `actor_type="agent"`, 사용자가 호출하는 경우 `actor_type="user"`. S2 원칙: AI 에이전트는 사람과 동등한 API 소비자.

### 지침 7-4. 품질 평가 데이터 확장

테스트 세트 10개는 최소 기준이다. Phase 7 AI품질평가에서 더 많은 샘플로 확장할 수 있도록 `MULTITURN_SAMPLES` 리스트 구조를 유지한다. 새 샘플은 `expected_keywords` 필드를 필수로 포함한다.
