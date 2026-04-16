# Task 2-4. Retriever 인터페이스 및 구현체 (FTS / Vector / Hybrid)

## 1. 작업 목적

검색 전략(FTS, Vector, Hybrid)을 **플러그인 구조**로 추상화하여, DocumentType별로 런타임에 Retriever를 교체할 수 있게 한다. S1의 `search_service.py`와 `rag_service.py`에 분산된 검색 로직을 `Retriever` 인터페이스 아래로 통합한다. FG2.1의 Citation 생성 로직을 각 Retriever의 반환값에 내재화한다.

## 2. 작업 범위

### 포함 범위

1. **`Retriever` 추상 베이스 클래스**
   - `retrieve(query, document_type, top_k, filters) → List[RetrievalResult]`
   - `RetrievalResult` 데이터 모델 (citation 포함)

2. **구현체 3개**
   - `FTSRetriever`: PostgreSQL FTS (S1 기존 로직 이관)
   - `VectorRetriever`: pgvector cosine similarity (S1 Phase 10 벡터 활용)
   - `HybridRetriever`: FTS + Vector → RRF(k=60) 통합

3. **`RetrieverFactory`**
   - `create(name: str, params: Dict) → Retriever`
   - 환경변수 또는 DocumentType 설정 기반 동적 생성

4. **ACL 필터 통합**
   - 모든 Retriever의 `filters` 파라미터에 ACL 범위 적용 의무화
   - S2 원칙: 폴백 경로(FTS, 로컬 모델)에서도 동일 ACL 적용

5. **Citation 생성 내재화**
   - 각 Retriever의 반환값 `RetrievalResult`에 `Citation` 포함
   - `CitationBuilder.build()`를 청크 조회 직후 호출

6. **플러그인 교체 테스트**

### 제외 범위

- Reranker (Task 2-5)
- DocumentType 설정 통합 (Task 2-6)
- 쿼리 재작성 (Task 2-7)

## 3. 선행 조건

- Task 2-1 완료 (Citation 모델, CitationBuilder)
- S1 `search_service.py` 기존 FTS 코드 분석 완료
- S1 `rag_service.py` 기존 Vector 검색 코드 분석 완료
- `document_chunks` 테이블에 pgvector 인덱스 존재 확인

## 4. 주요 작업 항목

### 4-1. RetrievalResult 및 Retriever ABC

**파일:** `/backend/app/services/retrieval/base.py`

```python
"""Retriever 추상 인터페이스.

모든 검색 전략은 Retriever를 상속하고 retrieve()를 구현한다.
반환값 RetrievalResult에는 Citation이 필수 포함된다 (FG2.1 계약).
"""
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional
from uuid import UUID

from app.schemas.citation import Citation


@dataclass
class RetrievalResult:
    """단일 검색 결과.

    Attributes:
        document_id: 문서 UUID
        version_id: 버전 UUID
        node_id: 노드 UUID
        content: 청크 원문
        score: 검색 스코어 (정규화 범위 0~1 또는 BM25 스코어)
        metadata: 문서/청크 메타데이터
        citation: Citation 5-tuple (필수, FG2.1)
        document_type: DocumentType 이름
        document_title: 문서 제목 (UI 표시용)
    """

    document_id: UUID
    version_id: UUID
    node_id: UUID
    content: str
    score: float
    citation: Citation          # 필수 — FG2.1 Citation 계약
    metadata: Dict[str, Any] = field(default_factory=dict)
    document_type: str = ""
    document_title: Optional[str] = None


class Retriever(ABC):
    """검색 전략 추상 인터페이스."""

    @abstractmethod
    async def retrieve(
        self,
        query: str,
        document_type: str,
        top_k: int = 10,
        filters: Optional[Dict[str, Any]] = None,
    ) -> List[RetrievalResult]:
        """쿼리를 검색하여 상위 top_k 결과를 반환한다.

        Args:
            query: 검색 쿼리 문자열
            document_type: DocumentType 이름 (전략 선택, 하드코딩 금지)
            top_k: 반환할 최대 결과 수
            filters: ACL, 날짜 범위 등 추가 필터 (ACL 필수 포함)

        Returns:
            RetrievalResult 리스트 (score 내림차순)

        Raises:
            ValueError: filters에 ACL 정보 없을 때 (개발 모드에서 경고)
        """

    def _warn_if_no_acl(self, filters: Optional[Dict]) -> None:
        """filters에 actor 정보가 없으면 경고를 로깅한다."""
        import logging
        if not filters or "actor_id" not in filters:
            logging.getLogger(__name__).warning(
                "retrieve() called without ACL filters — all documents may be exposed"
            )
```

### 4-2. FTSRetriever

**파일:** `/backend/app/services/retrieval/fts_retriever.py`

```python
"""FTS Retriever — PostgreSQL Full-Text Search.

S1 search_service.py의 FTS 로직을 Retriever 인터페이스로 이관.
"""
from __future__ import annotations

import logging
from typing import Any, Dict, List, Optional
from uuid import UUID

import psycopg2.extensions
import psycopg2.extras

from app.schemas.citation import Citation
from app.services.retrieval.base import Retriever, RetrievalResult
from app.services.retrieval.citation_builder import CitationBuilder

logger = logging.getLogger(__name__)


class FTSRetriever(Retriever):
    """PostgreSQL FTS 기반 검색 (BM25 스코어)."""

    def __init__(self, conn: psycopg2.extensions.connection) -> None:
        self._conn = conn

    async def retrieve(
        self,
        query: str,
        document_type: str,
        top_k: int = 10,
        filters: Optional[Dict[str, Any]] = None,
    ) -> List[RetrievalResult]:
        self._warn_if_no_acl(filters)
        filters = filters or {}
        actor_id = filters.get("actor_id")
        actor_role = filters.get("actor_role")

        sql = """
            SELECT
                dc.id               AS chunk_id,
                dv.document_id,
                dc.version_id,
                dc.node_id,
                dc.source_text,
                dc.metadata,
                d.document_type,
                d.title             AS document_title,
                ts_rank_cd(
                    to_tsvector('simple', dc.source_text),
                    plainto_tsquery('simple', %(query)s)
                ) AS score
            FROM document_chunks dc
            JOIN document_versions dv ON dc.version_id = dv.id
            JOIN documents d ON dv.document_id = d.id
            WHERE to_tsvector('simple', dc.source_text)
                  @@ plainto_tsquery('simple', %(query)s)
              AND d.document_type = %(doc_type)s
              -- ACL 서브쿼리 (기존 search_service 패턴 재사용)
              AND d.id IN (
                  SELECT document_id FROM document_access_view
                  WHERE actor_id = %(actor_id)s
              )
            ORDER BY score DESC
            LIMIT %(top_k)s
        """
        with self._conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
            cur.execute(sql, {
                "query": query,
                "doc_type": document_type,
                "actor_id": str(actor_id) if actor_id else None,
                "top_k": top_k,
            })
            rows = cur.fetchall()

        results = []
        for row in rows:
            citation = CitationBuilder.build(
                document_id=row["document_id"],
                version_id=row["version_id"],
                node_id=row["node_id"] or "",
                source_text=row["source_text"],
            )
            results.append(RetrievalResult(
                document_id=UUID(str(row["document_id"])),
                version_id=UUID(str(row["version_id"])),
                node_id=UUID(str(row["node_id"])) if row["node_id"] else UUID(int=0),
                content=row["source_text"],
                score=float(row["score"]),
                citation=citation,
                metadata=dict(row["metadata"] or {}),
                document_type=row["document_type"],
                document_title=row["document_title"],
            ))
        return results
```

### 4-3. VectorRetriever

**파일:** `/backend/app/services/retrieval/vector_retriever.py`

```python
"""Vector Retriever — pgvector cosine similarity.

Phase 10 임베딩 파이프라인의 벡터를 활용.
"""
from __future__ import annotations

import logging
from typing import Any, Dict, List, Optional
from uuid import UUID

import psycopg2.extensions
import psycopg2.extras

from app.schemas.citation import Citation
from app.services.embedding_service import get_embedding_provider
from app.services.retrieval.base import Retriever, RetrievalResult
from app.services.retrieval.citation_builder import CitationBuilder

logger = logging.getLogger(__name__)


class VectorRetriever(Retriever):
    """pgvector cosine similarity 기반 의미 검색."""

    def __init__(
        self,
        conn: psycopg2.extensions.connection,
        similarity_threshold: float = 0.3,
    ) -> None:
        self._conn = conn
        self._threshold = similarity_threshold

    async def retrieve(
        self,
        query: str,
        document_type: str,
        top_k: int = 10,
        filters: Optional[Dict[str, Any]] = None,
    ) -> List[RetrievalResult]:
        self._warn_if_no_acl(filters)
        filters = filters or {}
        actor_id = filters.get("actor_id")

        # 쿼리 임베딩 생성 (Phase 1 EmbeddingProvider 활용)
        embedding_provider = get_embedding_provider()
        query_vector = await embedding_provider.embed(query)

        sql = """
            SELECT
                dv.document_id,
                dc.version_id,
                dc.node_id,
                dc.source_text,
                dc.metadata,
                d.document_type,
                d.title AS document_title,
                1 - (dc.embedding <=> %(vector)s::vector) AS score
            FROM document_chunks dc
            JOIN document_versions dv ON dc.version_id = dv.id
            JOIN documents d ON dv.document_id = d.id
            WHERE d.document_type = %(doc_type)s
              AND 1 - (dc.embedding <=> %(vector)s::vector) >= %(threshold)s
              AND d.id IN (
                  SELECT document_id FROM document_access_view
                  WHERE actor_id = %(actor_id)s
              )
            ORDER BY score DESC
            LIMIT %(top_k)s
        """
        with self._conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
            cur.execute(sql, {
                "vector": query_vector,
                "doc_type": document_type,
                "threshold": self._threshold,
                "actor_id": str(actor_id) if actor_id else None,
                "top_k": top_k,
            })
            rows = cur.fetchall()

        results = []
        for row in rows:
            citation = CitationBuilder.build(
                document_id=row["document_id"],
                version_id=row["version_id"],
                node_id=row["node_id"] or "",
                source_text=row["source_text"],
            )
            results.append(RetrievalResult(
                document_id=UUID(str(row["document_id"])),
                version_id=UUID(str(row["version_id"])),
                node_id=UUID(str(row["node_id"])) if row["node_id"] else UUID(int=0),
                content=row["source_text"],
                score=float(row["score"]),
                citation=citation,
                metadata=dict(row["metadata"] or {}),
                document_type=row["document_type"],
                document_title=row["document_title"],
            ))
        return results
```

### 4-4. HybridRetriever (RRF)

**파일:** `/backend/app/services/retrieval/hybrid_retriever.py`

```python
"""Hybrid Retriever — FTS + Vector → RRF(k=60).

두 Retriever를 병렬 실행하고 Reciprocal Rank Fusion으로 통합.
"""
from __future__ import annotations

import asyncio
import logging
from typing import Any, Dict, List, Optional

from app.services.retrieval.base import Retriever, RetrievalResult
from app.services.retrieval.fts_retriever import FTSRetriever
from app.services.retrieval.vector_retriever import VectorRetriever

logger = logging.getLogger(__name__)

_RRF_K = 60  # RRF 상수 (표준값)


class HybridRetriever(Retriever):
    """FTS와 Vector 검색을 RRF로 통합하는 Hybrid Retriever."""

    def __init__(
        self,
        fts: FTSRetriever,
        vector: VectorRetriever,
        fts_weight: float = 0.4,
        vector_weight: float = 0.6,
    ) -> None:
        self._fts = fts
        self._vector = vector
        self._fts_weight = fts_weight
        self._vector_weight = vector_weight

    async def retrieve(
        self,
        query: str,
        document_type: str,
        top_k: int = 10,
        filters: Optional[Dict[str, Any]] = None,
    ) -> List[RetrievalResult]:
        self._warn_if_no_acl(filters)

        # FTS + Vector 병렬 실행
        fts_results, vec_results = await asyncio.gather(
            self._fts.retrieve(query, document_type, top_k * 3, filters),
            self._vector.retrieve(query, document_type, top_k * 3, filters),
        )

        return self._rrf_merge(fts_results, vec_results, top_k)

    def _rrf_merge(
        self,
        fts_results: List[RetrievalResult],
        vec_results: List[RetrievalResult],
        top_k: int,
    ) -> List[RetrievalResult]:
        """Reciprocal Rank Fusion으로 두 리스트를 통합한다."""
        scores: Dict[str, float] = {}
        result_map: Dict[str, RetrievalResult] = {}

        def _key(r: RetrievalResult) -> str:
            return f"{r.document_id}:{r.node_id}"

        for rank, result in enumerate(fts_results, start=1):
            k = _key(result)
            scores[k] = scores.get(k, 0.0) + self._fts_weight / (_RRF_K + rank)
            result_map[k] = result

        for rank, result in enumerate(vec_results, start=1):
            k = _key(result)
            scores[k] = scores.get(k, 0.0) + self._vector_weight / (_RRF_K + rank)
            if k not in result_map:
                result_map[k] = result

        sorted_keys = sorted(scores, key=lambda k: scores[k], reverse=True)[:top_k]
        merged = []
        for k in sorted_keys:
            r = result_map[k]
            # RRF 통합 스코어로 교체
            merged.append(RetrievalResult(
                document_id=r.document_id,
                version_id=r.version_id,
                node_id=r.node_id,
                content=r.content,
                score=scores[k],
                citation=r.citation,
                metadata=r.metadata,
                document_type=r.document_type,
                document_title=r.document_title,
            ))
        return merged
```

### 4-5. RetrieverFactory

**파일:** `/backend/app/services/retrieval/factory.py`

```python
"""RetrieverFactory — 설정 기반 Retriever 동적 생성."""
from __future__ import annotations

from typing import Any, Dict, Optional

import psycopg2.extensions

from app.services.retrieval.base import Retriever
from app.services.retrieval.fts_retriever import FTSRetriever
from app.services.retrieval.hybrid_retriever import HybridRetriever
from app.services.retrieval.vector_retriever import VectorRetriever


class RetrieverFactory:
    """Retriever 인스턴스를 설정 기반으로 생성한다."""

    @staticmethod
    def create(
        name: str,
        conn: psycopg2.extensions.connection,
        params: Optional[Dict[str, Any]] = None,
    ) -> Retriever:
        """Retriever 이름과 파라미터로 인스턴스를 생성한다.

        Args:
            name: "fts" | "vector" | "hybrid"
            conn: DB 연결
            params: Retriever별 설정 파라미터

        Returns:
            Retriever 구현체

        Raises:
            ValueError: 알 수 없는 Retriever 이름
        """
        params = params or {}
        if name == "fts":
            return FTSRetriever(conn)
        if name == "vector":
            return VectorRetriever(conn, **params)
        if name == "hybrid":
            fts = FTSRetriever(conn)
            vector = VectorRetriever(conn, similarity_threshold=params.get("similarity_threshold", 0.3))
            return HybridRetriever(
                fts=fts,
                vector=vector,
                fts_weight=params.get("fts_weight", 0.4),
                vector_weight=params.get("vector_weight", 0.6),
            )
        raise ValueError(f"Unknown retriever: {name!r}. Choose from: fts, vector, hybrid")
```

### 4-6. 단위/통합 테스트

**파일:** `/backend/tests/unit/services/test_retrievers.py`

```python
"""Retriever 플러그인 단위 테스트."""
import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from uuid import uuid4
from app.services.retrieval.factory import RetrieverFactory
from app.services.retrieval.hybrid_retriever import HybridRetriever, _RRF_K
from app.services.retrieval.base import RetrievalResult
from app.schemas.citation import Citation


def _make_result(score: float) -> RetrievalResult:
    return RetrievalResult(
        document_id=uuid4(), version_id=uuid4(), node_id=uuid4(),
        content="test", score=score,
        citation=Citation.from_chunk(uuid4(), uuid4(), uuid4(), "test"),
    )


def test_rrf_merge_combines_results():
    """HybridRetriever._rrf_merge가 두 리스트를 통합해야 한다."""
    fts = MagicMock()
    vec = MagicMock()
    hybrid = HybridRetriever(fts, vec)

    r1 = _make_result(0.9)
    r2 = _make_result(0.7)
    merged = hybrid._rrf_merge([r1], [r2], top_k=2)
    assert len(merged) == 2


def test_rrf_boost_overlap():
    """동일 결과가 두 리스트에 있으면 점수가 합산되어야 한다."""
    fts = MagicMock()
    vec = MagicMock()
    hybrid = HybridRetriever(fts, vec)

    r = _make_result(0.9)
    # 동일 document_id + node_id를 가진 결과를 두 리스트에 넣기 위해 동일 객체 사용
    merged = hybrid._rrf_merge([r], [r], top_k=1)
    expected = (0.4 / (_RRF_K + 1)) + (0.6 / (_RRF_K + 1))
    assert abs(merged[0].score - expected) < 1e-9


def test_retriever_factory_unknown_raises():
    conn = MagicMock()
    with pytest.raises(ValueError, match="Unknown retriever"):
        RetrieverFactory.create("unknown", conn)


def test_retriever_factory_creates_fts():
    conn = MagicMock()
    r = RetrieverFactory.create("fts", conn)
    from app.services.retrieval.fts_retriever import FTSRetriever
    assert isinstance(r, FTSRetriever)


def test_retriever_factory_creates_hybrid():
    conn = MagicMock()
    r = RetrieverFactory.create("hybrid", conn)
    assert isinstance(r, HybridRetriever)
```

## 5. 산출물

1. **Retriever ABC + RetrievalResult** (`/backend/app/services/retrieval/base.py`)
2. **FTSRetriever** (`/backend/app/services/retrieval/fts_retriever.py`)
3. **VectorRetriever** (`/backend/app/services/retrieval/vector_retriever.py`)
4. **HybridRetriever** (`/backend/app/services/retrieval/hybrid_retriever.py`)
5. **RetrieverFactory** (`/backend/app/services/retrieval/factory.py`)
6. **단위 테스트** (`/backend/tests/unit/services/test_retrievers.py`)

## 6. 완료 기준

1. `FTSRetriever`, `VectorRetriever`, `HybridRetriever` 각각 단독 실행 가능
2. 모든 Retriever 반환값에 `citation` 필드 포함
3. `filters`에 `actor_id` 없을 때 경고 로그 출력
4. `RetrieverFactory.create("hybrid", ...)` 정상 동작
5. RRF 통합 로직: 동일 결과 점수 합산 확인
6. 단위 테스트 전체 통과

## 7. 작업 지침

### 지침 7-1. ACL 미적용 금지

모든 Retriever는 `filters["actor_id"]`를 SQL 쿼리의 ACL 서브쿼리에 전달해야 한다. `actor_id`가 없는 경우 **폴백으로 전체 문서 반환 금지**. 대신 경고 로깅 후 빈 결과를 반환하거나 S1 기존 ACL 로직을 그대로 유지한다.

### 지침 7-2. HybridRetriever 병렬 실행

`FTSRetriever`와 `VectorRetriever`는 `asyncio.gather()`로 병렬 실행한다. 하나가 실패해도 나머지 결과는 반환하도록 예외 처리를 추가한다.

### 지침 7-3. RRF 상수 노출

`_RRF_K = 60`은 모듈 수준 상수로 정의하여 테스트에서 직접 참조할 수 있게 한다. 향후 설정 파일에서 override 가능하도록 설계한다.

### 지침 7-4. S1 기존 코드 보존

`search_service.py`와 `rag_service.py`의 기존 코드를 제거하지 않는다. 새로운 Retriever 구현체가 검증된 후 기존 코드를 deprecated 처리하는 방식으로 점진적 이관한다.
