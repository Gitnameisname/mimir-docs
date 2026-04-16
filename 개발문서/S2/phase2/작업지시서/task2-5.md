# Task 2-5. Reranker 인터페이스 및 구현체 (CrossEncoder / RuleBased / Null)

## 1. 작업 목적

Retriever가 반환한 상위 후보군을 더 정확하게 재정렬하는 **Reranker 플러그인 구조**를 구현한다. DocumentType별로 다른 재정렬 전략(CrossEncoder 모델, 규칙 기반, 없음)을 런타임에 교체할 수 있게 한다. 폐쇄망에서도 동작하도록 로컬 모델을 기본값으로 설정한다.

## 2. 작업 범위

### 포함 범위

1. **`Reranker` 추상 베이스 클래스**
   - `rerank(query, candidates, top_k) → List[RetrievalResult]`

2. **구현체 3개**
   - `CrossEncoderReranker`: `sentence-transformers` cross-encoder 모델
   - `RuleBasedReranker`: 문서 신선도·중요도 등 메타데이터 기반 스코어 조정
   - `NullReranker`: 재정렬 없이 그대로 통과

3. **`RerankerFactory`**
   - `create(name: str, params: Dict) → Reranker`

4. **폐쇄망 지원**
   - `CrossEncoderReranker`는 환경변수 `RERANKER_MODEL_PATH`가 설정된 경우 로컬 모델 사용
   - 설정 없음 또는 모델 로드 실패 시 `NullReranker`로 폴백 (서비스 중단 없음)

5. **단위 테스트**

### 제외 범위

- LLM 기반 Reranker (Phase 7에서 상세 구현)
- DocumentType 설정 통합 (Task 2-6)
- Retriever 구현 (Task 2-4)

## 3. 선행 조건

- Task 2-4 완료 (`RetrievalResult` 모델)
- `sentence-transformers` 패키지 설치 여부 확인
- 환경변수 `RERANKER_MODEL_PATH`, `RERANKER_ENABLED` 정의

## 4. 주요 작업 항목

### 4-1. Reranker 추상 베이스 클래스

**파일:** `/backend/app/services/retrieval/reranker_base.py`

```python
"""Reranker 추상 인터페이스.

Retriever 결과(후보군)를 입력받아 더 정확한 순서로 재정렬하고
상위 top_k개만 반환한다.
"""
from __future__ import annotations

from abc import ABC, abstractmethod
from typing import List

from app.services.retrieval.base import RetrievalResult


class Reranker(ABC):
    """재정렬 전략 추상 인터페이스."""

    @abstractmethod
    async def rerank(
        self,
        query: str,
        candidates: List[RetrievalResult],
        top_k: int = 10,
    ) -> List[RetrievalResult]:
        """후보 결과를 재정렬하여 상위 top_k개를 반환한다.

        Args:
            query: 원본 검색 쿼리
            candidates: Retriever가 반환한 후보 리스트
            top_k: 반환할 최대 결과 수

        Returns:
            재정렬된 RetrievalResult 리스트 (최대 top_k개)
        """
```

### 4-2. NullReranker

**파일:** `/backend/app/services/retrieval/null_reranker.py`

```python
"""NullReranker — 재정렬 없이 통과.

빠른 응답이 필요하거나 Reranker가 비활성화된 경우 사용.
"""
from __future__ import annotations

from typing import List

from app.services.retrieval.base import RetrievalResult
from app.services.retrieval.reranker_base import Reranker


class NullReranker(Reranker):
    """재정렬 없이 candidates를 그대로 반환한다."""

    async def rerank(
        self,
        query: str,
        candidates: List[RetrievalResult],
        top_k: int = 10,
    ) -> List[RetrievalResult]:
        return candidates[:top_k]
```

### 4-3. CrossEncoderReranker

**파일:** `/backend/app/services/retrieval/cross_encoder_reranker.py`

```python
"""CrossEncoderReranker — sentence-transformers cross-encoder 모델 기반 재정렬.

폐쇄망 지원:
  - RERANKER_MODEL_PATH 환경변수 설정 시 로컬 모델 사용
  - 미설정 또는 로드 실패 시 NullReranker로 폴백 (서비스 중단 없음)
"""
from __future__ import annotations

import logging
import os
from typing import List, Optional

from app.services.retrieval.base import RetrievalResult
from app.services.retrieval.null_reranker import NullReranker
from app.services.retrieval.reranker_base import Reranker

logger = logging.getLogger(__name__)

_DEFAULT_MODEL = "cross-encoder/ms-marco-MiniLM-L-6-v2"


class CrossEncoderReranker(Reranker):
    """Cross-encoder 모델 기반 재정렬.

    Attributes:
        model_name_or_path: 모델명 또는 로컬 경로
        _model: sentence_transformers CrossEncoder 인스턴스 (lazy load)
        _fallback: 모델 로드 실패 시 사용할 NullReranker
    """

    def __init__(self, model_name_or_path: Optional[str] = None) -> None:
        self._model_path = (
            model_name_or_path
            or os.getenv("RERANKER_MODEL_PATH")
            or _DEFAULT_MODEL
        )
        self._model = None
        self._fallback = NullReranker()
        self._load_model()

    def _load_model(self) -> None:
        """Cross-encoder 모델을 로드한다. 실패 시 경고 로그 후 fallback 모드."""
        try:
            from sentence_transformers import CrossEncoder
            self._model = CrossEncoder(self._model_path)
            logger.info("CrossEncoderReranker loaded: %s", self._model_path)
        except Exception as exc:
            logger.warning(
                "CrossEncoderReranker failed to load model %r: %s — falling back to NullReranker",
                self._model_path,
                exc,
            )
            self._model = None

    async def rerank(
        self,
        query: str,
        candidates: List[RetrievalResult],
        top_k: int = 10,
    ) -> List[RetrievalResult]:
        if self._model is None:
            return await self._fallback.rerank(query, candidates, top_k)

        # CrossEncoder는 동기 실행이므로 asyncio.to_thread로 감싸기
        import asyncio
        pairs = [(query, r.content) for r in candidates]
        scores = await asyncio.to_thread(self._model.predict, pairs)

        # score 갱신 후 내림차순 정렬
        scored = sorted(
            zip(scores, candidates),
            key=lambda x: x[0],
            reverse=True,
        )
        return [r for _, r in scored[:top_k]]
```

### 4-4. RuleBasedReranker

**파일:** `/backend/app/services/retrieval/rule_based_reranker.py`

```python
"""RuleBasedReranker — 메타데이터 기반 규칙 재정렬.

규칙:
  1. 문서 신선도: updated_at이 최근일수록 +bonus
  2. pinned 문서: metadata["pinned"] == True 이면 스코어 상승
  3. DocumentType별 규칙은 향후 설정에서 주입 가능

이 Reranker는 S2 원칙 준수: DocumentType 값을 코드에 하드코딩하지 않음.
"""
from __future__ import annotations

import logging
from datetime import datetime, timezone
from typing import Any, Callable, Dict, List, Optional

from app.services.retrieval.base import RetrievalResult
from app.services.retrieval.reranker_base import Reranker

logger = logging.getLogger(__name__)


class RuleBasedReranker(Reranker):
    """메타데이터 기반 휴리스틱 재정렬."""

    def __init__(
        self,
        freshness_bonus: float = 0.05,
        pinned_bonus: float = 0.10,
    ) -> None:
        self._freshness_bonus = freshness_bonus
        self._pinned_bonus = pinned_bonus

    async def rerank(
        self,
        query: str,
        candidates: List[RetrievalResult],
        top_k: int = 10,
    ) -> List[RetrievalResult]:
        scored = []
        now = datetime.now(tz=timezone.utc)

        for result in candidates:
            bonus = 0.0

            # 신선도 보너스 (최근 30일 이내 수정)
            updated_at_str = result.metadata.get("updated_at")
            if updated_at_str:
                try:
                    updated_at = datetime.fromisoformat(updated_at_str)
                    days_old = (now - updated_at).days
                    if days_old <= 30:
                        bonus += self._freshness_bonus * max(0, 1 - days_old / 30)
                except ValueError:
                    pass

            # Pinned 문서 보너스
            if result.metadata.get("pinned") is True:
                bonus += self._pinned_bonus

            scored.append((result.score + bonus, result))

        scored.sort(key=lambda x: x[0], reverse=True)
        return [r for _, r in scored[:top_k]]
```

### 4-5. RerankerFactory

**파일:** `/backend/app/services/retrieval/reranker_factory.py`

```python
"""RerankerFactory — 설정 기반 Reranker 동적 생성."""
from __future__ import annotations

import os
from typing import Any, Dict, Optional

from app.services.retrieval.reranker_base import Reranker


class RerankerFactory:
    """Reranker 인스턴스를 설정 기반으로 생성한다."""

    @staticmethod
    def create(
        name: Optional[str],
        params: Optional[Dict[str, Any]] = None,
    ) -> Reranker:
        """Reranker 이름과 파라미터로 인스턴스를 생성한다.

        Args:
            name: "cross_encoder" | "rule_based" | "null" | None
            params: Reranker별 설정 파라미터

        Returns:
            Reranker 구현체 (None 또는 "null" → NullReranker)

        Raises:
            ValueError: 알 수 없는 Reranker 이름
        """
        from app.services.retrieval.null_reranker import NullReranker
        from app.services.retrieval.cross_encoder_reranker import CrossEncoderReranker
        from app.services.retrieval.rule_based_reranker import RuleBasedReranker

        params = params or {}

        # 환경변수로 전체 비활성화 가능 (폐쇄망 고려)
        if os.getenv("RERANKER_ENABLED", "true").lower() == "false":
            return NullReranker()

        if name is None or name == "null":
            return NullReranker()
        if name == "cross_encoder":
            return CrossEncoderReranker(
                model_name_or_path=params.get("model")
            )
        if name == "rule_based":
            return RuleBasedReranker(
                freshness_bonus=params.get("freshness_bonus", 0.05),
                pinned_bonus=params.get("pinned_bonus", 0.10),
            )
        raise ValueError(
            f"Unknown reranker: {name!r}. Choose from: cross_encoder, rule_based, null"
        )
```

### 4-6. 단위 테스트

**파일:** `/backend/tests/unit/services/test_rerankers.py`

```python
"""Reranker 단위 테스트."""
import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from uuid import uuid4
from datetime import datetime, timezone

from app.services.retrieval.base import RetrievalResult
from app.schemas.citation import Citation
from app.services.retrieval.null_reranker import NullReranker
from app.services.retrieval.rule_based_reranker import RuleBasedReranker
from app.services.retrieval.reranker_factory import RerankerFactory


def _make_result(score: float, metadata: dict = None) -> RetrievalResult:
    return RetrievalResult(
        document_id=uuid4(), version_id=uuid4(), node_id=uuid4(),
        content="test content", score=score,
        citation=Citation.from_chunk(uuid4(), uuid4(), uuid4(), "test content"),
        metadata=metadata or {},
    )


@pytest.mark.asyncio
async def test_null_reranker_returns_top_k():
    reranker = NullReranker()
    candidates = [_make_result(float(i)) for i in range(5)]
    result = await reranker.rerank("query", candidates, top_k=3)
    assert len(result) == 3
    assert result[0].score == 4.0  # 원래 순서 유지


@pytest.mark.asyncio
async def test_rule_based_pinned_bonus():
    reranker = RuleBasedReranker(pinned_bonus=0.5)
    low_score = _make_result(0.3)
    high_pinned = _make_result(0.1, metadata={"pinned": True})
    result = await reranker.rerank("q", [low_score, high_pinned], top_k=2)
    # pinned 문서가 더 높은 순위여야 함
    assert result[0].metadata.get("pinned") is True


@pytest.mark.asyncio
async def test_rule_based_freshness_bonus():
    reranker = RuleBasedReranker(freshness_bonus=0.5)
    old = _make_result(0.8, metadata={"updated_at": "2020-01-01T00:00:00"})
    fresh = _make_result(0.5, metadata={"updated_at": datetime.now(timezone.utc).isoformat()})
    result = await reranker.rerank("q", [old, fresh], top_k=2)
    # freshness 보너스로 fresh가 앞에 오거나 비슷해야 함 (최소 오류 없이 실행)
    assert len(result) == 2


def test_reranker_factory_null():
    r = RerankerFactory.create(None)
    assert isinstance(r, NullReranker)


def test_reranker_factory_unknown_raises():
    with pytest.raises(ValueError, match="Unknown reranker"):
        RerankerFactory.create("unknown_algo")


def test_reranker_factory_disabled_by_env(monkeypatch):
    monkeypatch.setenv("RERANKER_ENABLED", "false")
    r = RerankerFactory.create("cross_encoder")
    assert isinstance(r, NullReranker)
```

## 5. 산출물

1. **Reranker ABC** (`/backend/app/services/retrieval/reranker_base.py`)
2. **NullReranker** (`/backend/app/services/retrieval/null_reranker.py`)
3. **CrossEncoderReranker** (`/backend/app/services/retrieval/cross_encoder_reranker.py`)
4. **RuleBasedReranker** (`/backend/app/services/retrieval/rule_based_reranker.py`)
5. **RerankerFactory** (`/backend/app/services/retrieval/reranker_factory.py`)
6. **단위 테스트** (`/backend/tests/unit/services/test_rerankers.py`)

## 6. 완료 기준

1. `NullReranker`는 candidates 순서 유지 + top_k 슬라이스
2. `RuleBasedReranker`는 pinned/freshness 보너스 적용 후 재정렬
3. `CrossEncoderReranker`는 모델 로드 실패 시 `NullReranker`로 폴백 (예외 미발생)
4. `RERANKER_ENABLED=false` 환경변수 시 모든 Reranker → NullReranker
5. `RerankerFactory.create("unknown")` → ValueError
6. 단위 테스트 전체 통과

## 7. 작업 지침

### 지침 7-1. 폐쇄망 폴백 필수

`CrossEncoderReranker`는 반드시 `try/except`로 모델 로드를 감싸야 한다. 모델이 없어도 서비스가 degrade(NullReranker 폴백)하되 실패하지 않아야 한다. S2 원칙: 외부 의존 off 시 서비스는 degrade하지만 실패하지 않음.

### 지침 7-2. CrossEncoder 동기 실행

`sentence-transformers`의 `CrossEncoder.predict()`는 동기 함수이다. `async def rerank()`에서 호출할 때 반드시 `asyncio.to_thread()`로 감싸서 이벤트 루프 블로킹을 방지한다.

### 지침 7-3. RuleBasedReranker 하드코딩 금지

DocumentType별 규칙을 코드에 `if document_type == "report":` 형태로 하드코딩하지 않는다. 규칙은 `params`로 주입받거나 향후 DocumentType 설정에서 로드하도록 인터페이스를 설계한다.

### 지침 7-4. top_k 상한 제한

Reranker는 후보군 100개 이내에서만 동작하도록 설계한다. Retriever가 `top_k * 10`개를 반환해도 Reranker에서는 최대 100개만 처리하여 CPU 비용을 절약한다. 초과 입력에 대해 경고 로그를 출력한다.
