# 작업지시서: task7-6 Answer Relevance + Evaluator 클래스 통합

**작업 코드**: task7-6  
**페이즈**: Phase 7 FG7.2 (평가 러너 + 지표)  
**담당자**: [TBD]  
**예상 소요시간**: 7~8일  
**최종 수정 일시**: 2026-04-17

---

## 1. 작업 목적

RAG 시스템의 응답 품질 평가를 위해 Answer Relevance(임베딩 기반) 지표를 구현하고, task7-4, task7-5의 모든 지표들(Faithfulness, Answer Relevance, Context Precision, Context Recall, Citation-present, Hallucination)을 통합하는 `Evaluator` 클래스를 개발한다. 또한 성능 지표(Latency, Token Count, Cost)를 포함하여 골든셋 데이터에 대한 종합적인 평가 보고서를 생성할 수 있도록 한다.

---

## 2. 작업 범위

### 포함 범위

1. **Answer Relevance 지표 (임베딩 기반)**
   - 질문(question)과 답변(answer)의 의미적 관련도 계산
   - Embedding Model (Phase 1)를 사용한 코사인 유사도
   - 임베딩 불가능 시 폴백: 키워드 오버랩
   - 0.0 ~ 1.0 범위 점수 반환

2. **Evaluator 클래스**
   - 모든 6가지 지표를 통합하여 관리
   - `evaluate_golden_item()`: 단일 goldenset 항목 평가
   - `evaluate_golden_set()`: 여러 항목 일괄 평가 (비동기)
   - 동시성(Concurrency) 처리: asyncio.gather() 활용
   - 지표 선택 가능 (특정 지표만 계산 옵션)

3. **Pydantic 데이터 모델**
   - `TokenMetrics`: 토큰 수 (입력, 출력, 총합)
   - `LatencyMetrics`: 응답 시간 (밀리초)
   - `CostMetrics`: 예상 비용 (달러)
   - `EvaluationResult`: 단일 항목 평가 결과 (6개 지표 + 성능)
   - `EvaluationReport`: 배치 평가 결과 요약 (집계, 분포)

4. **성능 지표 계산**
   - Latency: RAG 응답 생성 시간 (ms)
   - Token Count: 입력/출력 토큰 수 (LLM 기반)
   - Cost: 추정 비용 (토큰 수 × 단가, USD)
   - 비용 구성: query + response 분리

5. **집계 및 통계**
   - 평균, 중앙값, 표준편차, 최소/최대값
   - 지표별 분포 (버킷 기반)
   - 상관관계 분석 (지표 간의 영향도, 선택)
   - Failover Rate: 평가 실패 항목의 비율

6. **컴포넌트 구조**
   - `/backend/app/services/evaluation/metrics/embedding_based.py`
   - `/backend/app/services/evaluation/evaluator.py`
   - `/backend/app/services/evaluation/models.py` (Pydantic)

7. **단위 + 통합 테스트**
   - AnswerRelevanceMetric 단위 테스트 (6개 이상)
   - Evaluator 통합 테스트 (8개 이상)
   - Mock golden items 사용
   - 비동기 처리 테스트

### 제외 범위

- 데이터베이스 저장 (task7-7에서)
- REST API 엔드포인트 (task7-7에서)
- UI 대시보드 (별도 작업)
- 캐싱 인프라 (선택사항)

---

## 3. 선행 조건

1. **task7-4 완료**: 규칙 기반 지표 구현
2. **task7-5 완료**: LLM 기반 지표 + 폴백
3. **Phase 1 완료**: Embedding Model 추상화, LLM Provider
4. **프로젝트 구조**: `/backend/app/services/evaluation/` 디렉토리 존재
5. **라이브러리**: Pydantic v2, numpy, asyncio

---

## 4. 주요 작업 항목

### 4.1 Answer Relevance 지표

**파일**: `/backend/app/services/evaluation/metrics/embedding_based.py`

```python
"""
임베딩 기반 평가 지표.

Answer Relevance: 질문과 답변의 의미적 관련도
"""

import logging
from typing import Optional, List, Tuple
from abc import ABC, abstractmethod
import numpy as np

from .rule_based import MetricCalculator, MetricScore
from .fallback import KeywordExtractor

logger = logging.getLogger(__name__)


class EmbeddingModel(ABC):
    """임베딩 모델 추상 기반 클래스."""

    @abstractmethod
    def embed(self, text: str) -> List[float]:
        """
        텍스트를 임베딩으로 변환.

        Args:
            text: 입력 텍스트

        Returns:
            임베딩 벡터 (리스트)
        """
        pass

    @abstractmethod
    def embed_batch(self, texts: List[str]) -> List[List[float]]:
        """배치 임베딩."""
        pass


class AnswerRelevanceMetric(MetricCalculator):
    """
    Answer Relevance 지표 (임베딩 기반 + 폴백).

    = cosine_similarity(embed(question), embed(answer))
    범위: 0.0 ~ 1.0 (1.0이 최고, 완벽히 관련 있음)
    """

    def __init__(
        self,
        embedding_model: Optional[EmbeddingModel] = None,
        embedding_enabled: Optional[bool] = None,
        fallback_method: str = "keyword"
    ):
        """
        Args:
            embedding_model: Phase 1의 EmbeddingModel 인스턴스
            embedding_enabled: 임베딩 사용 여부 (None이면 환경변수 참조)
            fallback_method: 폴백 방법 ("keyword")
        """
        self.embedding_model = embedding_model
        self.embedding_enabled = embedding_enabled
        if self.embedding_enabled is None:
            import os
            self.embedding_enabled = os.getenv("EVAL_EMBEDDING_ENABLED", "true").lower() == "true"
        self.fallback_method = fallback_method

    @property
    def metric_name(self) -> str:
        return "answer_relevance"

    def compute(
        self,
        question: str,
        answer_text: str,
        **kwargs
    ) -> MetricScore:
        """
        Args:
            question: 사용자 질문
            answer_text: RAG 답변
            **kwargs: 추가 인자

        Returns:
            MetricScore with answer_relevance
        """
        if not question or not answer_text:
            return MetricScore(
                metric_name=self.metric_name,
                score=0.0,
                details={"reason": "missing question or answer"}
            )

        # 임베딩 기반 계산 시도
        if self.embedding_enabled and self.embedding_model:
            try:
                question_emb = self.embedding_model.embed(question)
                answer_emb = self.embedding_model.embed(answer_text)

                score = self._cosine_similarity(question_emb, answer_emb)

                return MetricScore(
                    metric_name=self.metric_name,
                    score=score,
                    details={"method": "embedding"}
                )
            except Exception as e:
                logger.warning(f"Embedding computation failed: {e}, using fallback")

        # 폴백
        logger.info("Using fallback method for answer relevance")
        score = self._fallback_relevance(question, answer_text)
        return MetricScore(
            metric_name=self.metric_name,
            score=score,
            details={"method": f"fallback_{self.fallback_method}"}
        )

    @staticmethod
    def _cosine_similarity(vec1: List[float], vec2: List[float]) -> float:
        """코사인 유사도 계산."""
        if not vec1 or not vec2:
            return 0.0

        vec1 = np.array(vec1, dtype=np.float32)
        vec2 = np.array(vec2, dtype=np.float32)

        dot_product = np.dot(vec1, vec2)
        norm1 = np.linalg.norm(vec1)
        norm2 = np.linalg.norm(vec2)

        if norm1 == 0 or norm2 == 0:
            return 0.0

        similarity = dot_product / (norm1 * norm2)
        # 범위를 [0, 1]로 정규화 (코사인 유사도는 [-1, 1] 범위)
        return max(0.0, min(similarity, 1.0))

    @staticmethod
    def _fallback_relevance(question: str, answer: str) -> float:
        """키워드 기반 폴백."""
        # 질문과 답변의 키워드 오버랩
        from .fallback import KeywordExtractor
        
        question_keywords = set(KeywordExtractor.extract_keywords(question, top_k=10))
        answer_keywords = set(KeywordExtractor.extract_keywords(answer, top_k=10))

        if not question_keywords:
            return 0.5  # 중립 점수

        overlap = len(question_keywords & answer_keywords)
        score = overlap / len(question_keywords)
        return min(score, 1.0)
```

### 4.2 Pydantic 데이터 모델

**파일**: `/backend/app/services/evaluation/models.py`

```python
"""
평가 결과 데이터 모델.

Pydantic v2 기반 타입 정의.
"""

from pydantic import BaseModel, Field, field_validator
from typing import List, Dict, Optional
from datetime import datetime
from enum import Enum


class MetricType(str, Enum):
    """지표 유형."""
    FAITHFULNESS = "faithfulness"
    ANSWER_RELEVANCE = "answer_relevance"
    CONTEXT_PRECISION = "context_precision"
    CONTEXT_RECALL = "context_recall"
    CITATION_PRESENT = "citation_present_rate"
    HALLUCINATION_RATE = "hallucination_rate"


class TokenMetrics(BaseModel):
    """토큰 사용량 지표."""
    query_tokens: int = Field(ge=0, description="쿼리 토큰 수")
    response_tokens: int = Field(ge=0, description="응답 토큰 수")
    total_tokens: int = Field(ge=0, description="총 토큰 수")

    @field_validator("total_tokens")
    @classmethod
    def validate_total(cls, v, info):
        """total_tokens = query_tokens + response_tokens 검증."""
        if "query_tokens" in info.data and "response_tokens" in info.data:
            expected = info.data["query_tokens"] + info.data["response_tokens"]
            if v != expected:
                raise ValueError(f"total_tokens should equal {expected}")
        return v


class LatencyMetrics(BaseModel):
    """지연 시간 지표."""
    retrieval_ms: float = Field(ge=0, description="검색 시간 (밀리초)")
    generation_ms: float = Field(ge=0, description="생성 시간 (밀리초)")
    total_ms: float = Field(ge=0, description="총 시간 (밀리초)")

    @field_validator("total_ms")
    @classmethod
    def validate_total(cls, v, info):
        """total_ms 검증."""
        if "retrieval_ms" in info.data and "generation_ms" in info.data:
            expected = info.data["retrieval_ms"] + info.data["generation_ms"]
            if abs(v - expected) > 1.0:  # 1ms 오차 허용
                raise ValueError(f"total_ms should be approximately {expected}")
        return v


class CostMetrics(BaseModel):
    """비용 지표 (USD)."""
    query_cost: float = Field(ge=0, description="쿼리 비용 (USD)")
    response_cost: float = Field(ge=0, description="응답 비용 (USD)")
    total_cost: float = Field(ge=0, description="총 비용 (USD)")

    @field_validator("total_cost")
    @classmethod
    def validate_total(cls, v, info):
        """total_cost 검증."""
        if "query_cost" in info.data and "response_cost" in info.data:
            expected = info.data["query_cost"] + info.data["response_cost"]
            if abs(v - expected) > 0.0001:  # $0.0001 오차 허용
                raise ValueError(f"total_cost should equal {expected}")
        return v


class ScoreMetrics(BaseModel):
    """모든 평가 지표 점수."""
    faithfulness: Optional[float] = Field(None, ge=0.0, le=1.0, description="신뢰도")
    answer_relevance: Optional[float] = Field(None, ge=0.0, le=1.0, description="답변 관련도")
    context_precision: Optional[float] = Field(None, ge=0.0, le=1.0, description="컨텍스트 정밀도")
    context_recall: Optional[float] = Field(None, ge=0.0, le=1.0, description="컨텍스트 재현율")
    citation_present_rate: Optional[float] = Field(None, ge=0.0, le=1.0, description="인용 포함도")
    hallucination_rate: Optional[float] = Field(None, ge=0.0, le=1.0, description="환각 비율")

    def average(self, exclude_none: bool = True) -> float:
        """평균 점수 계산."""
        scores = [
            self.faithfulness,
            self.answer_relevance,
            self.context_precision,
            self.context_recall,
            self.citation_present_rate,
            # hallucination_rate는 낮을수록 좋으므로 반전
            1.0 - self.hallucination_rate if self.hallucination_rate is not None else None
        ]

        if exclude_none:
            scores = [s for s in scores if s is not None]

        return sum(scores) / len(scores) if scores else 0.0

    def to_dict(self) -> Dict[str, float]:
        """딕셔너리로 변환."""
        return {
            "faithfulness": self.faithfulness,
            "answer_relevance": self.answer_relevance,
            "context_precision": self.context_precision,
            "context_recall": self.context_recall,
            "citation_present_rate": self.citation_present_rate,
            "hallucination_rate": self.hallucination_rate,
        }


class EvaluationResult(BaseModel):
    """단일 평가 항목 결과."""
    item_id: str = Field(description="Golden Set 항목 ID")
    question: str = Field(description="사용자 질문")
    answer: str = Field(description="RAG 시스템 답변")
    contexts: List[str] = Field(description="검색된 컨텍스트")
    expected_answer: Optional[str] = Field(None, description="기대 답변")
    expected_sources: Optional[List[str]] = Field(None, description="기대 소스 문서 ID")

    # 평가 결과
    scores: ScoreMetrics = Field(description="모든 지표 점수")
    token_metrics: Optional[TokenMetrics] = Field(None, description="토큰 지표")
    latency_metrics: Optional[LatencyMetrics] = Field(None, description="지연 시간 지표")
    cost_metrics: Optional[CostMetrics] = Field(None, description="비용 지표")

    # 메타데이터
    evaluation_time: datetime = Field(default_factory=datetime.utcnow, description="평가 시간")
    evaluator_version: str = Field(default="1.0", description="평가기 버전")
    notes: Optional[str] = Field(None, description="평가 노트")

    def overall_score(self) -> float:
        """전체 점수 계산."""
        return self.scores.average()


class EvaluationReport(BaseModel):
    """배치 평가 결과 보고서."""
    batch_id: str = Field(description="배치 ID")
    total_items: int = Field(ge=0, description="전체 항목 수")
    successful_items: int = Field(ge=0, description="성공한 항목 수")
    failed_items: int = Field(ge=0, description="실패한 항목 수")

    # 지표별 통계
    scores_summary: Dict[str, Dict[str, float]] = Field(
        description="지표별 통계 (min, max, mean, median, std)"
    )

    # 비용/성능 통계
    total_tokens: int = Field(ge=0, description="총 토큰 수")
    total_latency_ms: float = Field(ge=0, description="총 지연 시간 (ms)")
    total_cost: float = Field(ge=0, description="총 비용 (USD)")

    # 개별 결과
    results: List[EvaluationResult] = Field(description="모든 평가 결과")

    # 메타데이터
    created_at: datetime = Field(default_factory=datetime.utcnow)
    completed_at: Optional[datetime] = Field(None)
    duration_seconds: Optional[float] = Field(None)

    def overall_score(self) -> float:
        """배치 전체 평균 점수."""
        if not self.results:
            return 0.0
        scores = [r.overall_score() for r in self.results]
        return sum(scores) / len(scores)

    def pass_rate(self, threshold: float = 0.7) -> float:
        """통과율 (점수 >= threshold인 항목의 비율)."""
        if not self.results:
            return 0.0
        passed = sum(1 for r in self.results if r.overall_score() >= threshold)
        return passed / len(self.results)
```

### 4.3 Evaluator 클래스

**파일**: `/backend/app/services/evaluation/evaluator.py`

```python
"""
통합 평가기 (Evaluator).

모든 지표를 통합하여 Golden Set 평가 실행.
"""

import logging
import asyncio
from typing import List, Optional, Dict
from datetime import datetime
import time
from statistics import mean, median, stdev

from .metrics.rule_based import (
    CitationPresentMetric,
    HallucinationRateMetric,
    ContextRecallMetric,
)
from .metrics.llm_based import FaithfulnessMetric, ContextPrecisionMetric
from .metrics.embedding_based import AnswerRelevanceMetric
from .models import (
    ScoreMetrics,
    TokenMetrics,
    LatencyMetrics,
    CostMetrics,
    EvaluationResult,
    EvaluationReport,
    MetricType,
)

logger = logging.getLogger(__name__)


class Evaluator:
    """
    통합 평가기.

    6가지 지표를 모두 계산하고, 성능 지표를 추가하여 종합 평가 결과 제공.
    """

    # 지표별 토큰/비용 기본값 (테스트용)
    DEFAULT_COST_PER_1K_INPUT_TOKENS = 0.0005
    DEFAULT_COST_PER_1K_OUTPUT_TOKENS = 0.0015

    def __init__(
        self,
        faithfulness_metric: Optional[FaithfulnessMetric] = None,
        answer_relevance_metric: Optional[AnswerRelevanceMetric] = None,
        context_precision_metric: Optional[ContextPrecisionMetric] = None,
        context_recall_metric: Optional[ContextRecallMetric] = None,
        citation_present_metric: Optional[CitationPresentMetric] = None,
        hallucination_metric: Optional[HallucinationRateMetric] = None,
        cost_calculator=None,
    ):
        """
        Args:
            각 지표 계산기 (None이면 기본값 생성)
            cost_calculator: 비용 계산 함수 (선택)
        """
        self.faithfulness = faithfulness_metric or FaithfulnessMetric()
        self.answer_relevance = answer_relevance_metric or AnswerRelevanceMetric()
        self.context_precision = context_precision_metric or ContextPrecisionMetric()
        self.context_recall = context_recall_metric or ContextRecallMetric()
        self.citation_present = citation_present_metric or CitationPresentMetric()
        self.hallucination = hallucination_metric or HallucinationRateMetric()
        self.cost_calculator = cost_calculator

    async def evaluate_golden_item_async(
        self,
        item_id: str,
        question: str,
        answer: str,
        contexts: List[str],
        expected_answer: Optional[str] = None,
        expected_sources: Optional[List[str]] = None,
        retrieval_time_ms: float = 0.0,
        generation_time_ms: float = 0.0,
        input_tokens: Optional[int] = None,
        output_tokens: Optional[int] = None,
    ) -> EvaluationResult:
        """
        단일 항목 평가 (비동기).

        Args:
            item_id: 항목 ID
            question: 질문
            answer: RAG 답변
            contexts: 검색된 컨텍스트
            expected_answer: 기대 답변
            expected_sources: 기대 소스 문서 ID
            retrieval_time_ms: 검색 시간 (ms)
            generation_time_ms: 생성 시간 (ms)
            input_tokens: 입력 토큰 수 (자동 계산 안 함 시)
            output_tokens: 출력 토큰 수

        Returns:
            EvaluationResult
        """
        start_time = time.time()

        # 지표 계산 병렬화
        tasks = [
            self._compute_metric_async(
                self.faithfulness,
                question=question,
                answer_text=answer,
                contexts=contexts
            ),
            self._compute_metric_async(
                self.answer_relevance,
                question=question,
                answer_text=answer
            ),
            self._compute_metric_async(
                self.context_precision,
                question=question,
                answer_text=answer,
                contexts=contexts
            ),
            self._compute_metric_async(
                self.context_recall,
                expected_source_docs=expected_sources or [],
                retrieved_chunks=[{"source_id": ctx_id} for ctx_id in (expected_sources or [])]
            ),
            self._compute_metric_async(
                self.citation_present,
                answer_text=answer
            ),
            self._compute_metric_async(
                self.hallucination,
                answer_text=answer
            ),
        ]

        results = await asyncio.gather(*tasks, return_exceptions=True)

        # 결과 추출 (예외 처리)
        score_dict = {}
        for i, result in enumerate(results):
            metric_names = [
                "faithfulness",
                "answer_relevance",
                "context_precision",
                "context_recall",
                "citation_present_rate",
                "hallucination_rate"
            ]
            if isinstance(result, Exception):
                logger.error(f"Metric {metric_names[i]} failed: {result}")
                score_dict[metric_names[i]] = None
            else:
                score_dict[metric_names[i]] = result.score

        # ScoreMetrics 생성
        scores = ScoreMetrics(**score_dict)

        # 성능 지표
        token_metrics = None
        if input_tokens is not None and output_tokens is not None:
            token_metrics = TokenMetrics(
                query_tokens=input_tokens,
                response_tokens=output_tokens,
                total_tokens=input_tokens + output_tokens
            )

        latency_metrics = LatencyMetrics(
            retrieval_ms=retrieval_time_ms,
            generation_ms=generation_time_ms,
            total_ms=retrieval_time_ms + generation_time_ms
        )

        # 비용 계산
        cost_metrics = None
        if token_metrics:
            query_cost = (token_metrics.query_tokens / 1000) * self.DEFAULT_COST_PER_1K_INPUT_TOKENS
            response_cost = (token_metrics.response_tokens / 1000) * self.DEFAULT_COST_PER_1K_OUTPUT_TOKENS
            cost_metrics = CostMetrics(
                query_cost=query_cost,
                response_cost=response_cost,
                total_cost=query_cost + response_cost
            )

        elapsed = time.time() - start_time

        result = EvaluationResult(
            item_id=item_id,
            question=question,
            answer=answer,
            contexts=contexts,
            expected_answer=expected_answer,
            expected_sources=expected_sources,
            scores=scores,
            token_metrics=token_metrics,
            latency_metrics=latency_metrics,
            cost_metrics=cost_metrics,
        )

        logger.info(f"Evaluated item {item_id} in {elapsed:.2f}s, score: {result.overall_score():.3f}")
        return result

    def evaluate_golden_item(
        self,
        item_id: str,
        question: str,
        answer: str,
        contexts: List[str],
        expected_answer: Optional[str] = None,
        expected_sources: Optional[List[str]] = None,
        **kwargs
    ) -> EvaluationResult:
        """동기 버전 evaluate_golden_item_async."""
        loop = asyncio.get_event_loop()
        if loop.is_running():
            # 이미 실행 중인 루프가 있으면 새 루프 생성
            loop = asyncio.new_event_loop()
            asyncio.set_event_loop(loop)
        return loop.run_until_complete(
            self.evaluate_golden_item_async(
                item_id=item_id,
                question=question,
                answer=answer,
                contexts=contexts,
                expected_answer=expected_answer,
                expected_sources=expected_sources,
                **kwargs
            )
        )

    async def evaluate_golden_set_async(
        self,
        batch_id: str,
        golden_items: List[Dict],
        max_concurrent: int = 5,
    ) -> EvaluationReport:
        """
        배치 평가 (비동기).

        Args:
            batch_id: 배치 ID
            golden_items: Golden Set 항목 리스트
                각 항목은 다음을 포함:
                {
                    "id": "...",
                    "question": "...",
                    "answer": "...",
                    "contexts": [...],
                    "expected_answer": "...",
                    "expected_sources": [...]
                }
            max_concurrent: 동시 실행 최대 개수

        Returns:
            EvaluationReport
        """
        start_time = datetime.utcnow()
        results = []
        failed_count = 0

        # 세마포어로 동시성 제한
        semaphore = asyncio.Semaphore(max_concurrent)

        async def evaluate_with_semaphore(item):
            async with semaphore:
                try:
                    result = await self.evaluate_golden_item_async(
                        item_id=item.get("id", f"item_{len(results)}"),
                        question=item["question"],
                        answer=item["answer"],
                        contexts=item.get("contexts", []),
                        expected_answer=item.get("expected_answer"),
                        expected_sources=item.get("expected_sources"),
                        **{k: v for k, v in item.items() 
                           if k in ["retrieval_time_ms", "generation_time_ms", 
                                   "input_tokens", "output_tokens"]}
                    )
                    return result
                except Exception as e:
                    logger.error(f"Failed to evaluate item {item.get('id')}: {e}")
                    nonlocal failed_count
                    failed_count += 1
                    return None

        # 병렬 평가
        tasks = [evaluate_with_semaphore(item) for item in golden_items]
        results = await asyncio.gather(*tasks)
        results = [r for r in results if r is not None]

        # 통계 계산
        scores_summary = self._compute_statistics(results)

        # 총계 계산
        total_tokens = sum(
            r.token_metrics.total_tokens for r in results 
            if r.token_metrics
        )
        total_latency_ms = sum(
            r.latency_metrics.total_ms for r in results 
            if r.latency_metrics
        )
        total_cost = sum(
            r.cost_metrics.total_cost for r in results 
            if r.cost_metrics
        )

        completed_at = datetime.utcnow()
        duration = (completed_at - start_time).total_seconds()

        report = EvaluationReport(
            batch_id=batch_id,
            total_items=len(golden_items),
            successful_items=len(results),
            failed_items=failed_count,
            scores_summary=scores_summary,
            total_tokens=total_tokens,
            total_latency_ms=total_latency_ms,
            total_cost=total_cost,
            results=results,
            completed_at=completed_at,
            duration_seconds=duration,
        )

        logger.info(
            f"Batch {batch_id} completed: {len(results)}/{len(golden_items)} items, "
            f"duration: {duration:.1f}s, overall score: {report.overall_score():.3f}"
        )

        return report

    def evaluate_golden_set(
        self,
        batch_id: str,
        golden_items: List[Dict],
        **kwargs
    ) -> EvaluationReport:
        """동기 버전."""
        loop = asyncio.get_event_loop()
        return loop.run_until_complete(
            self.evaluate_golden_set_async(batch_id=batch_id, golden_items=golden_items, **kwargs)
        )

    @staticmethod
    async def _compute_metric_async(metric, **kwargs):
        """지표 계산 (비동기 래퍼)."""
        # 현재는 지표가 동기이므로 executor에서 실행
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(None, metric.compute, **kwargs)

    @staticmethod
    def _compute_statistics(results: List[EvaluationResult]) -> Dict[str, Dict[str, float]]:
        """지표별 통계 계산."""
        stats = {}

        metric_fields = [
            "faithfulness",
            "answer_relevance",
            "context_precision",
            "context_recall",
            "citation_present_rate",
            "hallucination_rate"
        ]

        for field in metric_fields:
            scores = [
                getattr(r.scores, field) for r in results
                if getattr(r.scores, field) is not None
            ]

            if scores:
                stats[field] = {
                    "min": min(scores),
                    "max": max(scores),
                    "mean": mean(scores),
                    "median": median(scores),
                    "std": stdev(scores) if len(scores) > 1 else 0.0,
                    "count": len(scores)
                }
            else:
                stats[field] = {
                    "min": None,
                    "max": None,
                    "mean": None,
                    "median": None,
                    "std": None,
                    "count": 0
                }

        return stats
```

### 4.4 통합 테스트

**파일**: `/backend/tests/services/evaluation/test_evaluator.py`

```python
"""
Evaluator 통합 테스트.
"""

import pytest
import asyncio
from datetime import datetime

from app.services.evaluation.evaluator import Evaluator
from app.services.evaluation.models import (
    EvaluationResult,
    EvaluationReport,
    ScoreMetrics,
    TokenMetrics,
    LatencyMetrics,
)
from app.services.evaluation.metrics.rule_based import (
    CitationPresentMetric,
    HallucinationRateMetric,
    ContextRecallMetric,
)
from app.services.evaluation.metrics.embedding_based import AnswerRelevanceMetric


class TestAnswerRelevanceMetric:
    """AnswerRelevanceMetric 테스트."""

    def setup_method(self):
        self.metric = AnswerRelevanceMetric(embedding_enabled=False)

    def test_answer_relevance_fallback_high(self):
        """높은 관련도 (폴백)."""
        question = "What is the capital of France?"
        answer = "Paris is the capital of France."

        result = self.metric.compute(question=question, answer_text=answer)
        assert result.score > 0.5  # 키워드 오버랩이 높음
        assert "fallback" in result.details["method"]

    def test_answer_relevance_fallback_low(self):
        """낮은 관련도 (폴백)."""
        question = "What is the capital of France?"
        answer = "Dogs are animals."

        result = self.metric.compute(question=question, answer_text=answer)
        assert result.score < 0.5  # 키워드 오버랩이 낮음

    def test_answer_relevance_empty_question(self):
        """빈 질문."""
        result = self.metric.compute(question="", answer_text="Some answer")
        assert result.score == 0.0

    def test_answer_relevance_empty_answer(self):
        """빈 답변."""
        result = self.metric.compute(question="What?", answer_text="")
        assert result.score == 0.0


class TestEvaluator:
    """Evaluator 통합 테스트."""

    def setup_method(self):
        self.evaluator = Evaluator()
        self.sample_item = {
            "id": "golden_001",
            "question": "What is Paris?",
            "answer": "Paris is the capital of France. [CITE: entity_type=chunk, entity_value=Paris, source_type=file, source_id=doc_001, chunk_id=ch_001]",
            "contexts": [
                "Paris is the capital and largest city of France.",
                "France is a country in Western Europe."
            ],
            "expected_answer": "The capital of France",
            "expected_sources": ["doc_001"],
            "retrieval_time_ms": 50.0,
            "generation_time_ms": 100.0,
            "input_tokens": 50,
            "output_tokens": 25,
        }

    def test_evaluate_golden_item_basic(self):
        """기본 평가 (동기)."""
        result = self.evaluator.evaluate_golden_item(**self.sample_item)

        assert isinstance(result, EvaluationResult)
        assert result.item_id == "golden_001"
        assert result.scores.faithfulness is not None
        assert result.scores.answer_relevance is not None
        assert result.scores.context_recall is not None
        assert result.scores.citation_present_rate == 1.0  # Citation이 있음
        assert 0.0 <= result.overall_score() <= 1.0

    def test_evaluate_golden_item_with_tokens(self):
        """토큰 및 비용 정보 포함."""
        result = self.evaluator.evaluate_golden_item(**self.sample_item)

        assert result.token_metrics is not None
        assert result.token_metrics.query_tokens == 50
        assert result.token_metrics.response_tokens == 25
        assert result.token_metrics.total_tokens == 75

        assert result.cost_metrics is not None
        assert result.cost_metrics.total_cost > 0.0

    def test_evaluate_golden_item_latency(self):
        """지연 시간 지표."""
        result = self.evaluator.evaluate_golden_item(**self.sample_item)

        assert result.latency_metrics is not None
        assert result.latency_metrics.retrieval_ms == 50.0
        assert result.latency_metrics.generation_ms == 100.0
        assert result.latency_metrics.total_ms == 150.0

    @pytest.mark.asyncio
    async def test_evaluate_golden_item_async(self):
        """비동기 평가."""
        result = await self.evaluator.evaluate_golden_item_async(**self.sample_item)

        assert isinstance(result, EvaluationResult)
        assert result.item_id == "golden_001"

    def test_evaluate_golden_set(self):
        """배치 평가 (동기)."""
        items = [self.sample_item] * 3

        report = self.evaluator.evaluate_golden_set(
            batch_id="batch_001",
            golden_items=items
        )

        assert isinstance(report, EvaluationReport)
        assert report.batch_id == "batch_001"
        assert report.total_items == 3
        assert report.successful_items == 3
        assert len(report.results) == 3

        # 통계 검증
        assert "faithfulness" in report.scores_summary
        assert "mean" in report.scores_summary["faithfulness"]

    @pytest.mark.asyncio
    async def test_evaluate_golden_set_async(self):
        """배치 평가 (비동기)."""
        items = [self.sample_item] * 5

        report = await self.evaluator.evaluate_golden_set_async(
            batch_id="batch_002",
            golden_items=items,
            max_concurrent=2
        )

        assert report.total_items == 5
        assert report.successful_items == 5

    def test_score_metrics_average(self):
        """ScoreMetrics 평균 계산."""
        scores = ScoreMetrics(
            faithfulness=0.8,
            answer_relevance=0.9,
            context_precision=0.7,
            context_recall=0.9,
            citation_present_rate=1.0,
            hallucination_rate=0.1,
        )

        avg = scores.average()
        assert 0.0 <= avg <= 1.0

    def test_evaluation_report_overall_score(self):
        """EvaluationReport 전체 점수."""
        result1 = self.evaluator.evaluate_golden_item(**self.sample_item)
        result2 = self.evaluator.evaluate_golden_item(**self.sample_item)

        report = EvaluationReport(
            batch_id="batch_test",
            total_items=2,
            successful_items=2,
            failed_items=0,
            scores_summary={},
            total_tokens=100,
            total_latency_ms=200.0,
            total_cost=0.01,
            results=[result1, result2],
        )

        overall = report.overall_score()
        assert 0.0 <= overall <= 1.0

    def test_evaluation_report_pass_rate(self):
        """통과율 계산."""
        result1 = self.evaluator.evaluate_golden_item(**self.sample_item)

        report = EvaluationReport(
            batch_id="batch_test",
            total_items=1,
            successful_items=1,
            failed_items=0,
            scores_summary={},
            total_tokens=0,
            total_latency_ms=0.0,
            total_cost=0.0,
            results=[result1],
        )

        pass_rate = report.pass_rate(threshold=0.5)
        assert 0.0 <= pass_rate <= 1.0

    def test_missing_sources_context_recall(self):
        """누락된 소스 처리."""
        item = {
            **self.sample_item,
            "expected_sources": ["doc_001", "doc_002", "doc_003"],
        }
        result = self.evaluator.evaluate_golden_item(**item)

        # Context Recall이 1.0이 아님 (일부 소스만 매칭)
        assert result.scores.context_recall < 1.0
```

---

## 5. 산출물

1. **Answer Relevance 지표**
   - `/backend/app/services/evaluation/metrics/embedding_based.py` (250줄)

2. **데이터 모델**
   - `/backend/app/services/evaluation/models.py` (400줄)

3. **Evaluator 클래스**
   - `/backend/app/services/evaluation/evaluator.py` (500줄)

4. **통합 테스트**
   - `/backend/tests/services/evaluation/test_evaluator.py` (300줄)

5. **문서**
   - 이 작업지시서 (task7-6.md)
   - 코드 내 docstring
   - 검수보고서 (task7-6-검수보고서.md) - 완료 후

6. **선택 산출물**
   - 보안 취약점 검사 보고서 (task7-6-보안검사보고서.md) - 완료 후

---

## 6. 완료 기준

1. **Answer Relevance 지표**
   - [ ] 임베딩 기반 코사인 유사도 계산
   - [ ] 폴백: 키워드 오버랩 (임베딩 불가 시)
   - [ ] 0.0~1.0 정규화 점수
   - [ ] 단위 테스트 4개 이상

2. **Pydantic 모델**
   - [ ] TokenMetrics: query, response, total 토큰
   - [ ] LatencyMetrics: retrieval, generation, total 시간 (ms)
   - [ ] CostMetrics: query, response, total 비용 (USD)
   - [ ] ScoreMetrics: 6개 지표 + average() 메서드
   - [ ] EvaluationResult: 단일 항목 결과 (모든 정보 포함)
   - [ ] EvaluationReport: 배치 결과 (통계, 총계 포함)
   - [ ] 모든 필드 타입 검증 및 범위 체크

3. **Evaluator 클래스**
   - [ ] evaluate_golden_item(): 동기 단일 항목 평가
   - [ ] evaluate_golden_item_async(): 비동기 단일 항목 평가
   - [ ] evaluate_golden_set(): 동기 배치 평가
   - [ ] evaluate_golden_set_async(): 비동기 배치 평가 (세마포어 동시성 제한)
   - [ ] 모든 6가지 지표 통합
   - [ ] 성능 지표 자동 계산 (latency, tokens, cost)
   - [ ] 통계 계산: min, max, mean, median, std

4. **성능 지표**
   - [ ] Latency: 검색 + 생성 시간 (ms)
   - [ ] Token Count: 입력/출력 토큰 수
   - [ ] Cost: USD 기반 비용 계산 (기본 가격 적용)

5. **통계 및 집계**
   - [ ] 지표별 min, max, mean, median, std 계산
   - [ ] 배치 전체 평균 점수
   - [ ] 통과율(pass_rate) 계산
   - [ ] 실패율(failed_items) 추적

6. **단위 + 통합 테스트**
   - [ ] AnswerRelevanceMetric: 4개 이상 테스트
   - [ ] Evaluator: 8개 이상 테스트
   - [ ] 동기 및 비동기 모두 테스트
   - [ ] 모든 테스트 통과 (pytest 100%)

7. **코드 품질**
   - [ ] Type hints 완전 적용
   - [ ] Docstring 모든 함수/클래스
   - [ ] 로그 라인 (INFO, WARNING, ERROR)
   - [ ] 예외 처리 적절함
   - [ ] PEP 8 준수

---

## 7. 작업 지침

### 지침 7-1: Pydantic v2 마이그레이션
모든 데이터 모델은 Pydantic v2 기반이다. v1의 `Config` 클래스 대신 `ConfigDict`를 사용한다 (필요 시). Field validators도 `@field_validator` 데코레이터를 사용한다.

### 지침 7-2: 비동기 처리의 안전성
`evaluate_golden_set_async()`는 `asyncio.Semaphore`로 동시 실행 개수를 제한한다. 기본값은 5이지만, 환경에 따라 조정 가능하다. 또한 `gather(..., return_exceptions=True)`로 개별 항목 실패가 전체 배치를 중단하지 않도록 한다.

### 지침 7-3: 점수의 일관된 정규화
모든 점수는 0.0~1.0 범위로 정규화된다. 특히 `hallucination_rate`는 낮을수록 좋으므로 (0.0이 최고), `ScoreMetrics.average()`에서는 `1.0 - hallucination_rate`로 반전시켜 평균을 계산한다.

### 지침 7-4: 성능 지표의 기본값
기본 비용 계산에는 OpenAI GPT-3.5 기준 가격을 사용한다:
- 입력: $0.0005 per 1K tokens
- 출력: $0.0015 per 1K tokens

향후 다른 모델 지원 시 `cost_calculator` 함수를 주입하여 커스터마이징 가능하다.

### 지침 7-5: 통계 계산의 견고성
표준편차(std)는 2개 이상의 샘플이 필요하다. 샘플이 1개 이하일 때는 `std=0.0`으로 설정한다.

### 지침 7-6: 로깅의 정보 수준
평가 진행 중 다음 정보를 로깅한다:
- `INFO`: 각 항목 평가 완료, 배치 평가 완료
- `WARNING`: 개별 지표 계산 실패 (폴백 사용 등)
- `ERROR`: 치명적 오류 (예외 발생)

이를 통해 프로덕션 환경에서 평가 시스템의 상태를 모니터링할 수 있다.

### 지침 7-7: 테스트에서의 Mock 데이터
단위 테스트는 실제 LLM/Embedding 호출 없이, Mock 데이터로 진행한다. 특히 `embedding_enabled=False`로 설정하여 폴백 로직을 테스트한다.

---

**작업 시작 예정일**: [TBD]  
**예상 종료일**: [TBD + 7-8일]  
**검수 예정일**: [TBD + 8-9일]
