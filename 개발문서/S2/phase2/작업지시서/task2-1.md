# Task 2-1. Citation 데이터 모델 및 생성 로직

## 1. 작업 목적

검색 결과의 근거를 "검증 가능한 좌표"로 표준화하는 **Citation 5-tuple 데이터 모델**을 정의하고, Retriever가 청크를 반환할 때마다 자동으로 Citation을 생성하는 로직을 구현한다. 이후 Phase 3(Conversation), Phase 4(Agent Interface)가 이 Citation 계약을 기준으로 동작하므로, 정확한 모델 정의가 이 Task의 핵심이다.

## 2. 작업 범위

### 포함 범위

1. **Citation Pydantic 모델**
   - 5-tuple: `document_id`, `version_id`, `node_id`, `span_offset`, `content_hash`
   - SHA-256 해시 계산 유틸리티
   - 직렬화/역직렬화 (JSON)

2. **Citation 생성 로직**
   - `CitationBuilder` 클래스: 청크 메타데이터 → Citation 자동 생성
   - S1 `RetrievedChunk`에서 필드 추출 매핑 정의

3. **Citation 검증 유틸리티**
   - `CitationVerifier`: `content_hash` 재계산 후 원본과 비교
   - 변조 감지 인터페이스

4. **S1 호환 래퍼**
   - 기존 `{document_id, node_id, score, snippet}` 구조에 `citation` 필드를 **선택적으로** 추가
   - 클라이언트가 citation 필드를 무시해도 정상 동작

5. **단위 테스트**
   - Citation 생성 정확성 (document_id, version_id, node_id, content_hash)
   - SHA-256 해시 안정성 (동일 입력 → 동일 출력)
   - 문서 수정 후 content_hash 불일치 감지

### 제외 범위

- Citation 역참조 API (Task 2-2)
- Retriever 플러그인 구조 (Task 2-4)
- 멀티턴 Citation 캐시 (Task 2-8)

## 3. 선행 조건

- Phase 0, Phase 1 완료
- S1 Phase 11 `RetrievedChunk` 구조 분석 완료 (`/backend/app/services/rag_service.py`)
- S1 `document_chunks` 테이블의 컬럼 구조 확인 (`document_id`, `version_id`, `node_id`, `source_text`)

## 4. 주요 작업 항목

### 4-1. Citation Pydantic 모델

**파일:** `/backend/app/schemas/citation.py`

```python
from __future__ import annotations

import hashlib
from typing import Optional
from uuid import UUID

from pydantic import BaseModel, Field, model_validator


class Citation(BaseModel):
    """검색 결과의 검증 가능한 위치 좌표 (5-tuple).

    Attributes:
        document_id: 근거 문서 UUID
        version_id: 인용 시점의 문서 버전 UUID (개정 후에도 원본 확인 가능)
        node_id: 문서 내 섹션 UUID (위치 추적)
        span_offset: 청크 내 문자 오프셋 — 단락 단위 인용 시 None
        content_hash: SHA-256(청크 원문) — 내용 변조 감지용
    """

    document_id: UUID
    version_id: UUID
    node_id: UUID
    span_offset: Optional[int] = Field(None, ge=0)
    content_hash: str = Field(..., min_length=64, max_length=64)

    model_config = {"frozen": True}

    @classmethod
    def from_chunk(
        cls,
        document_id: str | UUID,
        version_id: str | UUID,
        node_id: str | UUID,
        source_text: str,
        span_offset: Optional[int] = None,
    ) -> "Citation":
        """청크 메타데이터로부터 Citation을 생성한다."""
        content_hash = hashlib.sha256(source_text.encode("utf-8")).hexdigest()
        return cls(
            document_id=UUID(str(document_id)),
            version_id=UUID(str(version_id)),
            node_id=UUID(str(node_id)),
            span_offset=span_offset,
            content_hash=content_hash,
        )

    def verify(self, source_text: str) -> bool:
        """source_text의 SHA-256이 content_hash와 일치하는지 검증한다."""
        return hashlib.sha256(source_text.encode("utf-8")).hexdigest() == self.content_hash


class CitationVerifyResult(BaseModel):
    """Citation 검증 결과."""

    verified: bool
    original_text: Optional[str] = None
    modified: bool = False
    citation: Citation
```

### 4-2. CitationBuilder 클래스

**파일:** `/backend/app/services/retrieval/citation_builder.py`

```python
"""Citation 생성 로직.

Retriever가 document_chunks에서 청크를 조회할 때마다 CitationBuilder를 호출하여
Citation 5-tuple을 자동 생성한다. S1 RetrievedChunk와의 매핑을 담당한다.
"""

from __future__ import annotations

import logging
from typing import Optional
from uuid import UUID

from app.schemas.citation import Citation

logger = logging.getLogger(__name__)


class CitationBuilder:
    """RetrievedChunk → Citation 변환기."""

    @staticmethod
    def build(
        document_id: str | UUID,
        version_id: str | UUID,
        node_id: str | UUID,
        source_text: str,
        span_offset: Optional[int] = None,
    ) -> Citation:
        """청크 메타데이터로부터 Citation을 생성한다.

        Args:
            document_id: 문서 UUID
            version_id: 버전 UUID
            node_id: 노드 UUID
            source_text: 청크 원문 (SHA-256 계산 대상)
            span_offset: 청크 내 문자 오프셋 (단락 단위 인용 시 None)

        Returns:
            Citation 5-tuple 객체

        Raises:
            ValueError: UUID 변환 실패 또는 source_text 빈 문자열
        """
        if not source_text:
            raise ValueError("source_text must not be empty")

        citation = Citation.from_chunk(
            document_id=document_id,
            version_id=version_id,
            node_id=node_id,
            source_text=source_text,
            span_offset=span_offset,
        )
        logger.debug(
            "Citation built: doc=%s ver=%s node=%s hash=%s...",
            document_id,
            version_id,
            node_id,
            citation.content_hash[:8],
        )
        return citation

    @staticmethod
    def from_retrieved_chunk(chunk: "RetrievedChunk") -> Citation:  # type: ignore[name-defined]
        """S1 RetrievedChunk 객체로부터 Citation을 생성한다 (S1 호환 진입점).

        S1 RetrievedChunk 필드 매핑:
          chunk.document_id  → Citation.document_id
          chunk.version_id   → Citation.version_id
          chunk.node_id      → Citation.node_id
          chunk.source_text  → SHA-256 계산

        Args:
            chunk: S1 Phase 11 RetrievedChunk 데이터클래스 인스턴스

        Returns:
            Citation 5-tuple 객체
        """
        return CitationBuilder.build(
            document_id=chunk.document_id,
            version_id=chunk.version_id,
            node_id=chunk.node_id or "",
            source_text=chunk.source_text,
        )
```

### 4-3. S1 호환 검색 결과 스키마 확장

**파일:** `/backend/app/schemas/search.py` (기존 파일 수정)

- 기존 `DocumentSearchResult`, `NodeSearchResult`에 `citation` 필드를 **Optional**로 추가
- 기본값 `None` — 기존 클라이언트(S1)는 이 필드를 무시

```python
# 기존 DocumentSearchResult에 추가
from app.schemas.citation import Citation

class DocumentSearchResult(BaseModel):
    # ... 기존 필드 유지 ...
    citation: Optional[Citation] = None  # S2 신규 필드 (S1 클라이언트 무시 가능)
```

### 4-4. 단위 테스트

**파일:** `/backend/tests/unit/services/test_citation.py`

```python
import hashlib
import pytest
from uuid import uuid4
from app.schemas.citation import Citation
from app.services.retrieval.citation_builder import CitationBuilder


DOC_ID = uuid4()
VER_ID = uuid4()
NODE_ID = uuid4()
SOURCE = "Kubernetes는 컨테이너 오케스트레이션 플랫폼입니다."


def test_citation_from_chunk_produces_correct_hash():
    citation = Citation.from_chunk(DOC_ID, VER_ID, NODE_ID, SOURCE)
    expected = hashlib.sha256(SOURCE.encode()).hexdigest()
    assert citation.content_hash == expected


def test_citation_verify_returns_true_for_original():
    citation = Citation.from_chunk(DOC_ID, VER_ID, NODE_ID, SOURCE)
    assert citation.verify(SOURCE) is True


def test_citation_verify_returns_false_after_modification():
    citation = Citation.from_chunk(DOC_ID, VER_ID, NODE_ID, SOURCE)
    modified = SOURCE + " (수정됨)"
    assert citation.verify(modified) is False


def test_citation_is_frozen():
    citation = Citation.from_chunk(DOC_ID, VER_ID, NODE_ID, SOURCE)
    with pytest.raises(Exception):
        citation.content_hash = "tampered"


def test_citation_builder_build():
    citation = CitationBuilder.build(DOC_ID, VER_ID, NODE_ID, SOURCE)
    assert str(citation.document_id) == str(DOC_ID)
    assert str(citation.version_id) == str(VER_ID)
    assert citation.span_offset is None


def test_citation_builder_empty_source_raises():
    with pytest.raises(ValueError, match="source_text must not be empty"):
        CitationBuilder.build(DOC_ID, VER_ID, NODE_ID, "")


def test_citation_hash_stability():
    """동일 입력은 항상 동일 해시를 생성해야 한다."""
    hashes = {
        Citation.from_chunk(DOC_ID, VER_ID, NODE_ID, SOURCE).content_hash
        for _ in range(10)
    }
    assert len(hashes) == 1


def test_citation_json_roundtrip():
    citation = Citation.from_chunk(DOC_ID, VER_ID, NODE_ID, SOURCE)
    json_str = citation.model_dump_json()
    restored = Citation.model_validate_json(json_str)
    assert restored == citation
```

## 5. 산출물

1. **Citation Pydantic 모델** (`/backend/app/schemas/citation.py`)
   - `Citation` (5-tuple, frozen), `CitationVerifyResult`
2. **CitationBuilder** (`/backend/app/services/retrieval/citation_builder.py`)
   - `build()`, `from_retrieved_chunk()`
3. **S1 호환 스키마 수정** (`/backend/app/schemas/search.py`)
   - `citation: Optional[Citation] = None` 필드 추가
4. **단위 테스트** (`/backend/tests/unit/services/test_citation.py`)
   - 해시 정확성, 변조 감지, 직렬화, Builder 동작

## 6. 완료 기준

1. `Citation.from_chunk()` SHA-256 계산이 표준 라이브러리 결과와 일치
2. `Citation.verify()` 가 문서 수정 후 `False` 반환
3. `Citation` 모델이 frozen (불변) 상태 강제
4. `CitationBuilder.from_retrieved_chunk()`가 S1 `RetrievedChunk`를 오류 없이 변환
5. S1 호환 스키마에서 기존 필드 누락 없음 (`citation` 필드는 Optional)
6. 단위 테스트 전체 통과 (`pytest -v`)
7. mypy 타입 검사 통과

## 7. 작업 지침

### 지침 7-1. SHA-256 인코딩 일관성

`source_text`의 SHA-256은 반드시 `utf-8` 인코딩으로 계산한다. DB에 저장된 텍스트와 메모리 텍스트의 인코딩이 다를 수 있으므로, 저장·조회 양방향에서 동일한 인코딩 규약을 적용한다.

### 지침 7-2. UUID 필드 타입 유연성

`Citation.from_chunk()` 는 `str | UUID` 모두 허용한다. S1 `RetrievedChunk`가 `str` 타입으로 ID를 보관하기 때문에 내부에서 `UUID(str(...))` 변환을 수행한다.

### 지침 7-3. S1 하위호환성 원칙

기존 `DocumentSearchResult`에 `citation` 필드를 추가할 때 **기본값 `None`** 으로 설정한다. S1 클라이언트는 이 필드를 무시하며, S2 클라이언트만 활용한다. 필드 추가 외에 기존 필드를 제거하거나 이름을 변경하지 않는다.

### 지침 7-4. node_id 누락 처리

S1 일부 청크에서 `node_id` 가 `None` 일 수 있다. 이 경우 `CitationBuilder.from_retrieved_chunk()`는 빈 문자열(`""`)을 UUID 변환 시도하지 말고, 별도 규약(예: `UUID(int=0)` 또는 `raise ValueError`)을 정의하고 주석으로 문서화한다.
