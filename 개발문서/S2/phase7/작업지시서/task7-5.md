# 작업지시서: task7-5 LLM 기반 평가 지표 + 규칙 기반 폴백

**작업 코드**: task7-5  
**페이즈**: Phase 7 FG7.2 (평가 러너 + 지표)  
**담당자**: [TBD]  
**예상 소요시간**: 6~7일  
**최종 수정 일시**: 2026-04-17

---

## 1. 작업 목적

RAG 시스템의 응답 품질 평가에서 고정밀 판단이 필요한 Faithfulness(신뢰도)와 Context Precision(컨텍스트 정밀도) 지표를 LLM 판정자(LLM Judge)를 통해 구현한다. 동시에 폐쇄망 환경이나 LLM 서비스 불가능 상황을 대비하여 규칙 기반 폴백(Rule-based Fallback)을 제공한다. 이를 통해 LLM 의존도를 최소화하면서도 평가의 품질 저하를 최소화할 수 있다. (S2 원칙 ⑦: 폐쇄망 환경에서 전체 기능 동작)

---

## 2. 작업 범위

### 포함 범위

1. **Faithfulness 평가 지표 (LLM 기반)**
   - "답변 텍스트가 검색된 컨텍스트(청크)에 기반하고 있는가?"를 판정
   - LLM Judge를 통해 답변의 각 주요 주장(claim)을 검증
   - 이진 점수(0 또는 1)를 주장별로 계산, 평균화하여 0.0~1.0 범위 점수 산출
   - Prompt 템플릿: 역할 정의, few-shot examples, 평가 지침 포함
   - 계산 방식: (검증된 주장 수) / (전체 주장 수)

2. **Context Precision 평가 지표 (LLM 기반)**
   - "검색된 각 청크가 실제로 답변에 기여했는가?"를 판정
   - LLM Judge를 통해 청크별 관련도 점수(0~1)를 계산
   - 최종 점수: (관련 청크 수) / (전체 검색 청크 수)
   - 청크별 피드백 제공 (어느 청크가 얼마나 관련 있었는지)

3. **규칙 기반 폴백 메커니즘**
   - Faithfulness 폴백: 텍스트 오버랩 비율(기반 텍스트 vs 검색 청크)
   - Context Precision 폴백: 키워드 매칭, TF-IDF 유사도
   - 임베딩 기반 폴백 (선택): 코사인 유사도 활용
   - 환경 변수 `EVAL_LLM_ENABLED=true/false`로 동작 모드 전환

4. **LLM Provider 통합**
   - Phase 1에서 정의된 LLM Provider 추상화 활용
   - `LLMProvider.generate()` 또는 `LLMProvider.completion()` 호출
   - 재시도(retry) 로직: 최대 3회 재시도, 지수 백오프
   - 타임아웃 처리: 30초 이내에 응답 필요

5. **Prompt 템플릿 관리**
   - Faithfulness Judge Prompt
   - Context Precision Judge Prompt
   - 템플릿은 configurable strings로 저장 (향후 커스터마이징 용이)
   - 다국어 지원 고려 (영어 기본)

6. **응답 파싱**
   - LLM 응답 형식 정의: JSON 또는 구조화된 텍스트
   - 예: `{"claims": [{"text": "...", "score": 1}]}` (JSON 형식)
   - 파싱 실패 시 폴백 메커니즘 활용

7. **캐싱 (선택)**
   - 동일한 (question, answer, contexts) 조합에 대한 LLM 호출 캐싱
   - 캐시 저장소: Redis 또는 로컬 파일 기반
   - 캐시 TTL: 24시간

8. **컴포넌트 구조**
   - `/backend/app/services/evaluation/metrics/llm_based.py`
   - `/backend/app/services/evaluation/metrics/fallback.py`
   - `/backend/app/services/evaluation/prompts/judge_prompts.py`

9. **단위 테스트**
   - LLM Judge 로직 (mock LLM 사용)
   - 폴백 메커니즘 테스트
   - JSON 파싱 테스트
   - 환경 변수 토글 테스트

### 제외 범위

- LLM Provider 자체 구현 (Phase 1에서 기존)
- Answer Relevance (임베딩 기반, task7-6)
- 성능 지표 (Latency, Token Count, task7-6)
- UI 대시보드 (별도 작업)
- 프로덕션 캐싱 인프라 (선택사항)

---

## 3. 선행 조건

1. **Phase 1 완료**: LLM Provider 추상화 인터페이스 정의
2. **task7-4 완료**: 규칙 기반 지표(Citation Present, Context Recall) 구현
3. **Embedding Model**: 폴백에서 사용할 수 있는 임베딩 모델 (선택)
4. **텍스트 유사도 라이브러리**: scikit-learn, difflib 등
5. **환경 변수 관리**: `.env` 파일에 `EVAL_LLM_ENABLED` 정의 가능

---

## 4. 주요 작업 항목

### 4.1 Prompt 템플릿 정의

**파일**: `/backend/app/services/evaluation/prompts/judge_prompts.py`

```python
"""
LLM Judge 용 Prompt 템플릿.

특징:
- Few-shot examples 포함
- Chain-of-thought 지원
- 다국어 확장 가능
"""

from enum import Enum
from dataclasses import dataclass
from typing import List, Dict, Optional


class PromptLanguage(str, Enum):
    """지원 언어."""
    ENGLISH = "en"
    KOREAN = "ko"


@dataclass
class JudgePromptTemplate:
    """LLM Judge Prompt 템플릿."""
    name: str
    description: str
    system_role: str
    instruction: str
    examples: List[Dict[str, str]]  # few-shot examples
    language: PromptLanguage

    def render(
        self,
        question: str,
        answer: str,
        contexts: List[str],
        **kwargs
    ) -> str:
        """
        Prompt 렌더링.

        Args:
            question: 사용자 질문
            answer: RAG 시스템의 답변
            contexts: 검색된 컨텍스트 청크 목록
            **kwargs: 추가 템플릿 변수

        Returns:
            렌더링된 Prompt 문자열
        """
        context_text = "\n".join([f"[Context {i+1}]\n{ctx}" 
                                   for i, ctx in enumerate(contexts)])

        prompt = f"""{self.system_role}

{self.instruction}

Question: {question}

Contexts:
{context_text}

Answer: {answer}

{self._render_examples()}

Evaluation:"""
        return prompt

    def _render_examples(self) -> str:
        """Few-shot examples 렌더링."""
        if not self.examples:
            return ""

        example_text = "Examples:\n"
        for i, example in enumerate(self.examples, 1):
            example_text += f"\nExample {i}:\n{example.get('prompt', '')}\n"
            example_text += f"Expected output: {example.get('output', '')}\n"
        return example_text


# ============================================================================
# Faithfulness Judge Prompt
# ============================================================================

FAITHFULNESS_JUDGE_PROMPTS = {
    "en": JudgePromptTemplate(
        name="faithfulness_judge_v1",
        description="Evaluate whether the answer is grounded in the provided contexts",
        system_role=(
            "You are an expert evaluator assessing the faithfulness of a generated answer. "
            "Your task is to determine whether each claim in the answer is supported by the provided contexts."
        ),
        instruction=(
            "Analyze the answer and identify the key claims. For each claim, determine whether it is:\n"
            "1. Supported by the contexts (score = 1)\n"
            "2. Not supported or contradicted by the contexts (score = 0)\n"
            "3. Partially supported or uncertain (score = 0.5, round down to 0 for binary scoring)\n\n"
            "Provide your assessment as a JSON object with the following structure:\n"
            "{\n"
            '  "claims": [\n'
            '    {"text": "claim text", "score": 0 or 1, "reasoning": "brief explanation"}\n'
            "  ],\n"
            '  "overall_faithfulness": average score (0.0 to 1.0)\n'
            "}\n"
        ),
        examples=[
            {
                "prompt": (
                    "Question: What is the capital of France?\n"
                    "Context: [Context 1] France is a country in Western Europe. Its capital is Paris.\n"
                    "Answer: Paris is the capital of France."
                ),
                "output": (
                    '{"claims": [{"text": "Paris is the capital of France", "score": 1, "reasoning": "Directly supported by the context"}], '
                    '"overall_faithfulness": 1.0}'
                )
            },
            {
                "prompt": (
                    "Question: What is the capital of France?\n"
                    "Context: [Context 1] France is a country in Western Europe.\n"
                    "Answer: The capital of France is Berlin."
                ),
                "output": (
                    '{"claims": [{"text": "The capital of France is Berlin", "score": 0, "reasoning": "Contradicted by context"}], '
                    '"overall_faithfulness": 0.0}'
                )
            },
        ],
        language=PromptLanguage.ENGLISH
    ),
    "ko": JudgePromptTemplate(
        name="faithfulness_judge_ko",
        description="생성된 답변이 제공된 컨텍스트에 근거하고 있는지 평가",
        system_role=(
            "당신은 생성된 답변의 신뢰도를 평가하는 전문가입니다. "
            "답변의 각 주장이 제공된 컨텍스트로 뒷받침되는지를 판단합니다."
        ),
        instruction=(
            "답변을 분석하고 핵심 주장을 식별하세요. 각 주장에 대해 다음을 판단하세요:\n"
            "1. 컨텍스트로 뒷받침됨 (점수 = 1)\n"
            "2. 컨텍스트로 뒷받침되지 않거나 모순됨 (점수 = 0)\n"
            "3. 부분적으로 뒷받침되거나 불확실 (점수 = 0.5, 이진 점수로 0으로 반올림)\n\n"
            "다음 구조의 JSON 객체로 평가를 제시하세요:\n"
            "{\n"
            '  "claims": [\n'
            '    {"text": "주장 텍스트", "score": 0 또는 1, "reasoning": "간단한 설명"}\n'
            "  ],\n"
            '  "overall_faithfulness": 평균 점수 (0.0~1.0)\n'
            "}\n"
        ),
        examples=[],
        language=PromptLanguage.KOREAN
    ),
}


# ============================================================================
# Context Precision Judge Prompt
# ============================================================================

CONTEXT_PRECISION_JUDGE_PROMPTS = {
    "en": JudgePromptTemplate(
        name="context_precision_judge_v1",
        description="Evaluate whether each retrieved context is relevant to answering the question",
        system_role=(
            "You are an expert evaluator assessing the precision of retrieved contexts. "
            "Your task is to score each context based on its relevance to answering the question."
        ),
        instruction=(
            "Evaluate each context based on whether it contains useful information for answering the question.\n"
            "Score each context as:\n"
            "- 1: Highly relevant, directly supports the answer\n"
            "- 0.5: Somewhat relevant, provides partial support\n"
            "- 0: Not relevant or irrelevant\n\n"
            "Provide your assessment as a JSON object:\n"
            "{\n"
            '  "context_scores": [\n'
            '    {"index": 0, "score": 0 to 1, "reasoning": "brief explanation"}\n'
            "  ],\n"
            '  "average_precision": average score (0.0 to 1.0)\n'
            "}\n"
        ),
        examples=[
            {
                "prompt": (
                    "Question: Who was the first president of the United States?\n"
                    "[Context 1] George Washington was the first president of the United States, serving from 1789 to 1797.\n"
                    "[Context 2] The United States was founded in 1776 with the Declaration of Independence.\n"
                    "Answer: George Washington was the first president."
                ),
                "output": (
                    '{"context_scores": '
                    '[{"index": 0, "score": 1, "reasoning": "Directly answers the question"}, '
                    '{"index": 1, "score": 0, "reasoning": "Provides historical context but does not answer the question"}], '
                    '"average_precision": 0.5}'
                )
            },
        ],
        language=PromptLanguage.ENGLISH
    ),
}


def get_faithfulness_prompt(
    language: PromptLanguage = PromptLanguage.ENGLISH
) -> JudgePromptTemplate:
    """Faithfulness Judge Prompt 조회."""
    return FAITHFULNESS_JUDGE_PROMPTS.get(language.value)


def get_context_precision_prompt(
    language: PromptLanguage = PromptLanguage.ENGLISH
) -> JudgePromptTemplate:
    """Context Precision Judge Prompt 조회."""
    return CONTEXT_PRECISION_JUDGE_PROMPTS.get(language.value)
```

### 4.2 규칙 기반 폴백 구현

**파일**: `/backend/app/services/evaluation/metrics/fallback.py`

```python
"""
규칙 기반 폴백 메커니즘.

LLM 사용 불가능할 때 대체하는 규칙 기반 평가 함수들.
"""

import re
from typing import List, Dict, Tuple
from difflib import SequenceMatcher
import logging

logger = logging.getLogger(__name__)


class TextOverlapCalculator:
    """텍스트 오버랩 기반 점수 계산."""

    @staticmethod
    def calculate_word_overlap(text1: str, text2: str) -> float:
        """
        두 텍스트의 단어 기반 오버랩 비율 계산.

        Args:
            text1: 첫 번째 텍스트 (예: 답변)
            text2: 두 번째 텍스트 (예: 컨텍스트)

        Returns:
            오버랩 비율 (0.0 ~ 1.0)
        """
        words1 = set(text1.lower().split())
        words2 = set(text2.lower().split())

        if not words1:
            return 0.0

        overlap = len(words1 & words2)
        return min(overlap / len(words1), 1.0)

    @staticmethod
    def calculate_sequence_similarity(text1: str, text2: str) -> float:
        """
        SequenceMatcher 기반 유사도 계산.

        Args:
            text1: 첫 번째 텍스트
            text2: 두 번째 텍스트

        Returns:
            유사도 (0.0 ~ 1.0)
        """
        matcher = SequenceMatcher(None, text1.lower(), text2.lower())
        return matcher.ratio()


class KeywordExtractor:
    """주요 키워드 추출."""

    # 불용어 (Stopwords)
    ENGLISH_STOPWORDS = {
        "the", "a", "an", "and", "or", "but", "in", "on", "at", "to", "for",
        "of", "with", "by", "is", "are", "was", "were", "be", "been", "being",
        "have", "has", "had", "do", "does", "did", "will", "would", "could",
        "should", "may", "might", "can", "must", "shall", "it", "this", "that",
        "what", "which", "who", "whom", "when", "where", "why", "how"
    }

    @staticmethod
    def extract_keywords(text: str, top_k: int = 10) -> List[str]:
        """
        텍스트에서 주요 키워드 추출.

        Args:
            text: 입력 텍스트
            top_k: 상위 K개 키워드 반환

        Returns:
            키워드 목록
        """
        # 단어 분할 및 불용어 제거
        words = [
            w.lower() for w in re.findall(r'\b[a-zA-Z]+\b', text)
            if w.lower() not in KeywordExtractor.ENGLISH_STOPWORDS and len(w) > 2
        ]

        # 빈도 기반 정렬
        from collections import Counter
        freq = Counter(words)
        keywords = [word for word, _ in freq.most_common(top_k)]
        return keywords

    @staticmethod
    def keyword_overlap(text1: str, text2: str) -> float:
        """
        두 텍스트의 키워드 오버랩 비율.

        Args:
            text1: 첫 번째 텍스트
            text2: 두 번째 텍스트

        Returns:
            오버랩 비율
        """
        keywords1 = set(KeywordExtractor.extract_keywords(text1))
        keywords2 = set(KeywordExtractor.extract_keywords(text2))

        if not keywords1:
            return 0.0

        overlap = len(keywords1 & keywords2)
        return min(overlap / len(keywords1), 1.0)


class FaithfulnessFallback:
    """Faithfulness 지표 규칙 기반 폴백."""

    @staticmethod
    def calculate_faithfulness(
        answer: str,
        contexts: List[str],
        method: str = "overlap"
    ) -> float:
        """
        규칙 기반 Faithfulness 점수 계산.

        Method 종류:
        - "overlap": 텍스트 오버랩 (word-based)
        - "sequence": 시퀀스 유사도
        - "keyword": 키워드 기반 오버랩
        - "ensemble": 3가지 방법의 앙상블

        Args:
            answer: 답변 텍스트
            contexts: 컨텍스트 청크 목록
            method: 계산 방법

        Returns:
            Faithfulness 점수 (0.0 ~ 1.0)
        """
        if not answer or not contexts:
            return 0.0

        # 모든 컨텍스트를 하나의 텍스트로 통합
        combined_context = " ".join(contexts)

        if method == "overlap":
            score = TextOverlapCalculator.calculate_word_overlap(answer, combined_context)
        elif method == "sequence":
            score = TextOverlapCalculator.calculate_sequence_similarity(answer, combined_context)
        elif method == "keyword":
            score = KeywordExtractor.keyword_overlap(answer, combined_context)
        elif method == "ensemble":
            # 3가지 방법의 가중 평균
            overlap_score = TextOverlapCalculator.calculate_word_overlap(answer, combined_context)
            seq_score = TextOverlapCalculator.calculate_sequence_similarity(answer, combined_context)
            keyword_score = KeywordExtractor.keyword_overlap(answer, combined_context)
            score = (overlap_score + seq_score + keyword_score) / 3.0
        else:
            logger.warning(f"Unknown method: {method}, using 'overlap'")
            score = TextOverlapCalculator.calculate_word_overlap(answer, combined_context)

        return min(max(score, 0.0), 1.0)


class ContextPrecisionFallback:
    """Context Precision 지표 규칙 기반 폴백."""

    @staticmethod
    def calculate_context_precision(
        question: str,
        answer: str,
        contexts: List[str],
        method: str = "keyword"
    ) -> Tuple[float, List[float]]:
        """
        규칙 기반 Context Precision 점수 계산.

        Args:
            question: 사용자 질문
            answer: 답변 텍스트
            contexts: 컨텍스트 청크 목록
            method: 계산 방법 ("keyword", "overlap", "ensemble")

        Returns:
            (전체 정밀도 점수, 청크별 점수 목록)
        """
        if not contexts:
            return 1.0, []

        chunk_scores = []

        for context in contexts:
            if method == "keyword":
                # 질문과 답변의 키워드가 컨텍스트에 얼마나 포함되는가?
                qa_keywords = set(
                    KeywordExtractor.extract_keywords(question + " " + answer)
                )
                context_keywords = set(KeywordExtractor.extract_keywords(context))

                if not qa_keywords:
                    score = 0.0
                else:
                    overlap = len(qa_keywords & context_keywords)
                    score = min(overlap / len(qa_keywords), 1.0)

            elif method == "overlap":
                # 답변과 컨텍스트의 단어 오버랩
                score = TextOverlapCalculator.calculate_word_overlap(answer, context)

            elif method == "ensemble":
                kw_score = KeywordExtractor.keyword_overlap(question + " " + answer, context)
                overlap_score = TextOverlapCalculator.calculate_word_overlap(answer, context)
                score = (kw_score + overlap_score) / 2.0

            else:
                logger.warning(f"Unknown method: {method}, using 'keyword'")
                score = 0.0

            # 컨텍스트가 관련 있으면 1, 무관하면 0으로 이진화 (threshold=0.3)
            binary_score = 1.0 if score >= 0.3 else 0.0
            chunk_scores.append(binary_score)

        avg_precision = sum(chunk_scores) / len(chunk_scores) if chunk_scores else 0.0
        return min(max(avg_precision, 0.0), 1.0), chunk_scores
```

### 4.3 LLM 기반 지표 구현

**파일**: `/backend/app/services/evaluation/metrics/llm_based.py`

```python
"""
LLM 기반 평가 지표.

Faithfulness, Context Precision을 LLM Judge를 통해 계산.
LLM 불가능 시 규칙 기반 폴백 사용.
"""

import os
import json
import logging
from typing import Optional, List, Dict
from datetime import datetime
import asyncio
from abc import ABC, abstractmethod

from .rule_based import MetricCalculator, MetricScore
from .fallback import FaithfulnessFallback, ContextPrecisionFallback
from ..prompts.judge_prompts import (
    get_faithfulness_prompt,
    get_context_precision_prompt,
    PromptLanguage,
)

logger = logging.getLogger(__name__)


class LLMJudge(ABC):
    """LLM Judge 추상 기반 클래스."""

    @abstractmethod
    async def judge(self, prompt: str) -> str:
        """
        Prompt에 대한 LLM 판정.

        Args:
            prompt: 평가 Prompt

        Returns:
            LLM 응답 문자열
        """
        pass

    @abstractmethod
    def judge_sync(self, prompt: str) -> str:
        """동기 버전의 judge()."""
        pass


class LLMProviderJudge(LLMJudge):
    """Phase 1 LLM Provider를 활용한 Judge."""

    def __init__(self, llm_provider, max_retries: int = 3, timeout: int = 30):
        """
        Args:
            llm_provider: Phase 1의 LLMProvider 인스턴스
            max_retries: 최대 재시도 횟수
            timeout: 응답 타임아웃 (초)
        """
        self.llm_provider = llm_provider
        self.max_retries = max_retries
        self.timeout = timeout

    async def judge(self, prompt: str) -> str:
        """LLM 판정 (비동기)."""
        for attempt in range(self.max_retries):
            try:
                response = await asyncio.wait_for(
                    self.llm_provider.generate_async(prompt),
                    timeout=self.timeout
                )
                return response
            except asyncio.TimeoutError:
                logger.warning(f"LLM judge timeout on attempt {attempt + 1}")
                if attempt == self.max_retries - 1:
                    raise
                await asyncio.sleep(2 ** attempt)  # 지수 백오프
            except Exception as e:
                logger.error(f"LLM judge error: {e}")
                if attempt == self.max_retries - 1:
                    raise
                await asyncio.sleep(2 ** attempt)

    def judge_sync(self, prompt: str) -> str:
        """LLM 판정 (동기)."""
        for attempt in range(self.max_retries):
            try:
                response = self.llm_provider.generate(prompt)
                return response
            except Exception as e:
                logger.error(f"LLM judge error: {e}")
                if attempt == self.max_retries - 1:
                    raise
                import time
                time.sleep(2 ** attempt)


class JSONResponseParser:
    """LLM JSON 응답 파서."""

    @staticmethod
    def parse_faithfulness_response(response: str) -> Optional[Dict]:
        """
        Faithfulness Judge 응답 파싱.

        응답 형식:
        {
            "claims": [
                {"text": "...", "score": 0 or 1, "reasoning": "..."},
                ...
            ],
            "overall_faithfulness": 0.0~1.0
        }

        Returns:
            파싱된 딕셔너리, 또는 파싱 실패 시 None
        """
        try:
            # JSON 블록 추출
            json_match = response.find('{')
            if json_match == -1:
                return None

            json_str = response[json_match:]
            json_end = json_str.rfind('}') + 1
            json_str = json_str[:json_end]

            data = json.loads(json_str)
            return data
        except (json.JSONDecodeError, ValueError) as e:
            logger.warning(f"Failed to parse JSON response: {e}")
            return None

    @staticmethod
    def parse_context_precision_response(response: str) -> Optional[Dict]:
        """
        Context Precision Judge 응답 파싱.

        응답 형식:
        {
            "context_scores": [
                {"index": 0, "score": 0~1, "reasoning": "..."},
                ...
            ],
            "average_precision": 0.0~1.0
        }

        Returns:
            파싱된 딕셔너리, 또는 파싱 실패 시 None
        """
        try:
            json_match = response.find('{')
            if json_match == -1:
                return None

            json_str = response[json_match:]
            json_end = json_str.rfind('}') + 1
            json_str = json_str[:json_end]

            data = json.loads(json_str)
            return data
        except (json.JSONDecodeError, ValueError) as e:
            logger.warning(f"Failed to parse JSON response: {e}")
            return None


class FaithfulnessMetric(MetricCalculator):
    """
    Faithfulness 지표 (LLM 기반 + 폴백).

    = (검증된 주장 수) / (전체 주장 수)
    범위: 0.0 ~ 1.0
    """

    def __init__(
        self,
        llm_judge: Optional[LLMJudge] = None,
        llm_enabled: Optional[bool] = None,
        language: PromptLanguage = PromptLanguage.ENGLISH,
        fallback_method: str = "ensemble"
    ):
        """
        Args:
            llm_judge: LLM Judge 인스턴스 (None이면 생성 불가)
            llm_enabled: LLM 사용 여부 (None이면 환경변수 참조)
            language: Prompt 언어
            fallback_method: 폴백 계산 방법
        """
        self.llm_judge = llm_judge
        self.llm_enabled = llm_enabled
        if self.llm_enabled is None:
            self.llm_enabled = os.getenv("EVAL_LLM_ENABLED", "false").lower() == "true"
        self.language = language
        self.fallback_method = fallback_method
        self.prompt_template = get_faithfulness_prompt(language)

    @property
    def metric_name(self) -> str:
        return "faithfulness"

    def compute(
        self,
        question: str,
        answer_text: str,
        contexts: List[str],
        **kwargs
    ) -> MetricScore:
        """
        Args:
            question: 사용자 질문
            answer_text: RAG 답변
            contexts: 검색된 컨텍스트
            **kwargs: 추가 인자

        Returns:
            MetricScore with faithfulness
        """
        if not answer_text or not contexts:
            return MetricScore(
                metric_name=self.metric_name,
                score=0.0,
                details={"reason": "missing answer or contexts"}
            )

        # LLM 판정 시도
        if self.llm_enabled and self.llm_judge:
            try:
                prompt = self.prompt_template.render(question, answer_text, contexts)
                response = self.llm_judge.judge_sync(prompt)

                parsed = JSONResponseParser.parse_faithfulness_response(response)
                if parsed and "overall_faithfulness" in parsed:
                    score = float(parsed["overall_faithfulness"])
                    return MetricScore(
                        metric_name=self.metric_name,
                        score=score,
                        details={
                            "method": "llm_judge",
                            "claims": parsed.get("claims", []),
                            "response": response[:200]  # 처음 200자만
                        }
                    )
            except Exception as e:
                logger.warning(f"LLM judge failed, falling back: {e}")

        # 폴백
        logger.info("Using fallback method for faithfulness")
        score = FaithfulnessFallback.calculate_faithfulness(
            answer_text, contexts, method=self.fallback_method
        )
        return MetricScore(
            metric_name=self.metric_name,
            score=score,
            details={"method": f"fallback_{self.fallback_method}"}
        )


class ContextPrecisionMetric(MetricCalculator):
    """
    Context Precision 지표 (LLM 기반 + 폴백).

    = (관련 청크 수) / (전체 청크 수)
    범위: 0.0 ~ 1.0
    """

    def __init__(
        self,
        llm_judge: Optional[LLMJudge] = None,
        llm_enabled: Optional[bool] = None,
        language: PromptLanguage = PromptLanguage.ENGLISH,
        fallback_method: str = "ensemble"
    ):
        """
        Args:
            llm_judge: LLM Judge 인스턴스
            llm_enabled: LLM 사용 여부
            language: Prompt 언어
            fallback_method: 폴백 계산 방법
        """
        self.llm_judge = llm_judge
        self.llm_enabled = llm_enabled
        if self.llm_enabled is None:
            self.llm_enabled = os.getenv("EVAL_LLM_ENABLED", "false").lower() == "true"
        self.language = language
        self.fallback_method = fallback_method
        self.prompt_template = get_context_precision_prompt(language)

    @property
    def metric_name(self) -> str:
        return "context_precision"

    def compute(
        self,
        question: str,
        answer_text: str,
        contexts: List[str],
        **kwargs
    ) -> MetricScore:
        """
        Args:
            question: 사용자 질문
            answer_text: RAG 답변
            contexts: 검색된 컨텍스트
            **kwargs: 추가 인자

        Returns:
            MetricScore with context_precision
        """
        if not contexts:
            return MetricScore(
                metric_name=self.metric_name,
                score=1.0,
                details={"num_contexts": 0}
            )

        # LLM 판정 시도
        if self.llm_enabled and self.llm_judge:
            try:
                prompt = self.prompt_template.render(question, answer_text, contexts)
                response = self.llm_judge.judge_sync(prompt)

                parsed = JSONResponseParser.parse_context_precision_response(response)
                if parsed and "average_precision" in parsed:
                    score = float(parsed["average_precision"])
                    return MetricScore(
                        metric_name=self.metric_name,
                        score=score,
                        details={
                            "method": "llm_judge",
                            "context_scores": parsed.get("context_scores", []),
                            "num_contexts": len(contexts)
                        }
                    )
            except Exception as e:
                logger.warning(f"LLM judge failed, falling back: {e}")

        # 폴백
        logger.info("Using fallback method for context precision")
        score, chunk_scores = ContextPrecisionFallback.calculate_context_precision(
            question, answer_text, contexts, method=self.fallback_method
        )
        return MetricScore(
            metric_name=self.metric_name,
            score=score,
            details={
                "method": f"fallback_{self.fallback_method}",
                "num_contexts": len(contexts),
                "relevant_contexts": sum(1 for s in chunk_scores if s > 0.5)
            }
        )
```

### 4.4 단위 테스트

**파일**: `/backend/tests/services/evaluation/metrics/test_llm_based.py`

```python
"""
LLM 기반 평가 지표 단위 테스트.
"""

import pytest
import json
from unittest.mock import Mock, patch, AsyncMock
from app.services.evaluation.metrics.llm_based import (
    FaithfulnessMetric,
    ContextPrecisionMetric,
    LLMProviderJudge,
    JSONResponseParser,
)
from app.services.evaluation.metrics.fallback import (
    FaithfulnessFallback,
    ContextPrecisionFallback,
)
from app.services.evaluation.prompts.judge_prompts import PromptLanguage


class MockLLMJudge:
    """테스트용 Mock LLM Judge."""

    def __init__(self, response_template: dict):
        self.response_template = response_template

    def judge_sync(self, prompt: str) -> str:
        return json.dumps(self.response_template)


class TestFaithfulnessMetric:
    """FaithfulnessMetric 테스트."""

    def setup_method(self):
        self.contexts = [
            "Paris is the capital of France.",
            "France is located in Western Europe."
        ]

    def test_faithfulness_with_llm_full_score(self):
        """LLM Judge로 완전 신뢰도 판정."""
        question = "What is the capital of France?"
        answer = "Paris is the capital of France."

        mock_response = {
            "claims": [
                {"text": "Paris is the capital of France", "score": 1, "reasoning": "Supported"}
            ],
            "overall_faithfulness": 1.0
        }

        mock_judge = MockLLMJudge(mock_response)
        metric = FaithfulnessMetric(
            llm_judge=mock_judge,
            llm_enabled=True
        )

        result = metric.compute(question=question, answer_text=answer, contexts=self.contexts)
        assert result.score == 1.0
        assert result.details["method"] == "llm_judge"

    def test_faithfulness_with_llm_zero_score(self):
        """LLM Judge로 신뢰도 0 판정."""
        question = "What is the capital of France?"
        answer = "The capital of France is Berlin."

        mock_response = {
            "claims": [
                {"text": "The capital of France is Berlin", "score": 0, "reasoning": "Contradicted"}
            ],
            "overall_faithfulness": 0.0
        }

        mock_judge = MockLLMJudge(mock_response)
        metric = FaithfulnessMetric(
            llm_judge=mock_judge,
            llm_enabled=True
        )

        result = metric.compute(question=question, answer_text=answer, contexts=self.contexts)
        assert result.score == 0.0

    def test_faithfulness_fallback_when_llm_disabled(self):
        """LLM 비활성화 시 폴백 사용."""
        question = "What is the capital of France?"
        answer = "Paris is the capital of France."

        metric = FaithfulnessMetric(
            llm_judge=None,
            llm_enabled=False
        )

        result = metric.compute(question=question, answer_text=answer, contexts=self.contexts)
        assert "fallback" in result.details["method"]
        assert 0.0 <= result.score <= 1.0

    def test_faithfulness_fallback_on_llm_error(self):
        """LLM 오류 시 폴백 자동 활성화."""
        question = "What is the capital?"
        answer = "Paris"

        mock_judge = Mock()
        mock_judge.judge_sync.side_effect = Exception("LLM error")

        metric = FaithfulnessMetric(
            llm_judge=mock_judge,
            llm_enabled=True
        )

        result = metric.compute(question=question, answer_text=answer, contexts=self.contexts)
        assert "fallback" in result.details["method"]

    def test_faithfulness_empty_answer(self):
        """빈 답변."""
        result = FaithfulnessMetric().compute(
            question="What?",
            answer_text="",
            contexts=self.contexts
        )
        assert result.score == 0.0

    def test_faithfulness_no_contexts(self):
        """컨텍스트 없음."""
        result = FaithfulnessMetric().compute(
            question="What?",
            answer_text="Some answer",
            contexts=[]
        )
        assert result.score == 0.0


class TestContextPrecisionMetric:
    """ContextPrecisionMetric 테스트."""

    def setup_method(self):
        self.question = "Who was the first president?"
        self.answer = "George Washington was the first president."
        self.contexts = [
            "George Washington was the first president, serving from 1789-1797.",
            "The US was founded in 1776."
        ]

    def test_context_precision_with_llm_full(self):
        """LLM Judge로 완전 정밀도 판정."""
        mock_response = {
            "context_scores": [
                {"index": 0, "score": 1, "reasoning": "Relevant"},
                {"index": 1, "score": 0, "reasoning": "Irrelevant"}
            ],
            "average_precision": 0.5
        }

        mock_judge = MockLLMJudge(mock_response)
        metric = ContextPrecisionMetric(
            llm_judge=mock_judge,
            llm_enabled=True
        )

        result = metric.compute(
            question=self.question,
            answer_text=self.answer,
            contexts=self.contexts
        )
        assert result.score == 0.5
        assert result.details["method"] == "llm_judge"

    def test_context_precision_fallback(self):
        """폴백 방식 테스트."""
        metric = ContextPrecisionMetric(
            llm_judge=None,
            llm_enabled=False
        )

        result = metric.compute(
            question=self.question,
            answer_text=self.answer,
            contexts=self.contexts
        )
        assert "fallback" in result.details["method"]
        assert result.details["num_contexts"] == 2


class TestJSONResponseParser:
    """JSONResponseParser 테스트."""

    def test_parse_faithfulness_response_valid(self):
        """정상 JSON 응답 파싱."""
        response = json.dumps({
            "claims": [{"text": "claim", "score": 1, "reasoning": "ok"}],
            "overall_faithfulness": 0.9
        })

        parsed = JSONResponseParser.parse_faithfulness_response(response)
        assert parsed is not None
        assert parsed["overall_faithfulness"] == 0.9

    def test_parse_faithfulness_response_with_text(self):
        """JSON 전후 텍스트가 있는 응답."""
        response = "Some explanation\n" + json.dumps({
            "claims": [],
            "overall_faithfulness": 0.5
        }) + "\nMore text"

        parsed = JSONResponseParser.parse_faithfulness_response(response)
        assert parsed is not None
        assert parsed["overall_faithfulness"] == 0.5

    def test_parse_faithfulness_response_invalid(self):
        """Invalid JSON."""
        response = "Not JSON at all"
        parsed = JSONResponseParser.parse_faithfulness_response(response)
        assert parsed is None


class TestFallbackMethods:
    """폴백 메서드 테스트."""

    def test_faithfulness_fallback_word_overlap(self):
        """단어 오버랩 기반 폴백."""
        answer = "Paris is the capital of France"
        contexts = ["Paris is the capital of France and is a beautiful city"]

        score = FaithfulnessFallback.calculate_faithfulness(
            answer, [contexts[0]], method="overlap"
        )
        assert score > 0.7  # 높은 오버랩

    def test_faithfulness_fallback_ensemble(self):
        """앙상블 폴백."""
        answer = "The capital is Paris"
        contexts = ["Paris, the capital of France"]

        score = FaithfulnessFallback.calculate_faithfulness(
            answer, [contexts[0]], method="ensemble"
        )
        assert 0.0 <= score <= 1.0

    def test_context_precision_fallback(self):
        """Context Precision 폴백."""
        question = "Who was the first president?"
        answer = "George Washington"
        contexts = [
            "George Washington was the first US president",
            "The US was founded in 1776"
        ]

        score, chunk_scores = ContextPrecisionFallback.calculate_context_precision(
            question, answer, contexts, method="keyword"
        )
        assert len(chunk_scores) == 2
        assert 0.0 <= score <= 1.0
```

---

## 5. 산출물

1. **Prompt 템플릿 모듈**
   - `/backend/app/services/evaluation/prompts/judge_prompts.py` (300줄)

2. **폴백 메커니즘**
   - `/backend/app/services/evaluation/metrics/fallback.py` (350줄)

3. **LLM 기반 지표**
   - `/backend/app/services/evaluation/metrics/llm_based.py` (400줄)

4. **단위 테스트**
   - `/backend/tests/services/evaluation/metrics/test_llm_based.py` (350줄)

5. **문서**
   - 이 작업지시서 (task7-5.md)
   - 코드 내 docstring
   - 검수보고서 (task7-5-검수보고서.md) - 완료 후

6. **선택 산출물**
   - 보안 취약점 검사 보고서 (task7-5-보안검사보고서.md) - 완료 후

---

## 6. 완료 기준

1. **Prompt 템플릿**
   - [ ] Faithfulness Judge Prompt (영어, 한국어)
   - [ ] Context Precision Judge Prompt (영어)
   - [ ] Few-shot examples 포함
   - [ ] render() 메서드로 동적 렌더링 가능

2. **규칙 기반 폴백**
   - [ ] TextOverlapCalculator: 단어 오버랩, 시퀀스 유사도
   - [ ] KeywordExtractor: 키워드 추출, 오버랩 계산
   - [ ] FaithfulnessFallback: 4가지 방법 (overlap, sequence, keyword, ensemble)
   - [ ] ContextPrecisionFallback: 3가지 방법 (keyword, overlap, ensemble)

3. **LLMProviderJudge**
   - [ ] judge_sync() 동기 메서드 구현
   - [ ] judge() 비동기 메서드 (선택)
   - [ ] 재시도 로직: 최대 3회, 지수 백오프
   - [ ] 타임아웃: 30초

4. **FaithfulnessMetric**
   - [ ] LLM 기반 계산 (LLM 활성화 시)
   - [ ] JSON 응답 파싱
   - [ ] 폴백 자동 활성화 (LLM 실패 또는 비활성화)
   - [ ] 환경변수 EVAL_LLM_ENABLED 지원
   - [ ] 0.0~1.0 정규화 점수

5. **ContextPrecisionMetric**
   - [ ] LLM 기반 청크별 점수 계산
   - [ ] 폴백 메커니즘
   - [ ] 관련도 임계값 처리 (threshold=0.3)
   - [ ] 청크 인덱스와 점수 매핑

6. **JSON 응답 파싱**
   - [ ] 정상 JSON 파싱
   - [ ] JSON 전후 텍스트 제거
   - [ ] 파싱 실패 시 None 반환
   - [ ] 로깅 (warning 레벨)

7. **단위 테스트**
   - [ ] FaithfulnessMetric: 6개 이상 테스트
   - [ ] ContextPrecisionMetric: 4개 이상 테스트
   - [ ] JSONResponseParser: 3개 이상 테스트
   - [ ] Fallback 메서드: 3개 이상 테스트
   - [ ] Mock LLM 활용 테스트
   - [ ] 모든 테스트 통과 (pytest 100%)

8. **코드 품질**
   - [ ] Type hints 완전 적용
   - [ ] Docstring 모든 함수/클래스에 적용
   - [ ] 로그 라인 추가 (INFO, WARNING, ERROR)
   - [ ] 예외 처리 적절함
   - [ ] PEP 8 준수

---

## 7. 작업 지침

### 지침 7-1: LLM 활성화 전환 메커니즘 (S2 원칙 ⑦ 폐쇄망 환경)
모든 LLM 기반 Metric은 `llm_enabled` 매개변수와 `EVAL_LLM_ENABLED` 환경변수를 지원한다. 폐쇄망 환경에서는 이 값을 `false`로 설정하여 규칙 기반 폴백으로 평가를 수행한다. **중요**: 환경변수가 `false`일 때도 서비스가 완전히 동작해야 한다. 폴백은 LLM보다 정확도가 낮을 수 있지만, 평가는 실패하지 않는다.

### 지침 7-2: Prompt 템플릿의 확장 가능성
`JudgePromptTemplate` 클래스는 `render()` 메서드를 통해 동적 Prompt 생성을 지원한다. 향후 추가 언어(예: 중국어, 일본어)가 필요한 경우, `FAITHFULNESS_JUDGE_PROMPTS` 딕셔너리에 새로운 템플릿을 추가하면 된다. 또한 Few-shot examples를 추가함으로써 LLM 판정 품질을 개선할 수 있다.

### 지침 7-3: LLM 응답 파싱의 안정성
LLM 응답은 항상 예상과 다를 수 있으므로, JSON 파싱 실패 시 `None`을 반환하고 폴백으로 전환한다. 또한 `try-except`로 모든 파싱 오류를 캡처하고 로깅한다. 프로덕션 환경에서 LLM 품질 저하를 빠르게 감지하기 위해, 파싱 실패율을 모니터링해야 한다.

### 지침 7-4: 폴백 메서드의 성능 트레이드오프
규칙 기반 폴백은 매우 빠르지만 정확도가 낮을 수 있다. 반대로 LLM 기반은 정확하지만 느리고 비용이 든다. 프로덕션 환경에서는 다음을 고려한다:
- 인터랙티브 평가(실시간): LLM 활성화, 타임아웃 짧게
- 배치 평가(대량): 폴백 사용, 비용 절감
- 모니터링 평가: 일부 LLM + 일부 폴백 (샘플링)

### 지침 7-5: 재시도 로직과 지수 백오프
LLM Judge에서 네트워크 오류나 일시적 장애 발생 시, 최대 3회 재시도한다. 재시도 간격은 지수 백오프를 사용한다 (1초, 2초, 4초). 이는 과부하 상황에서 서버 부하를 분산시키고, 일시적 장애 복구를 지원한다.

### 지침 7-6: 환경변수 기반 설정
`EVAL_LLM_ENABLED` 환경변수를 다음처럼 사용한다:
```bash
# 프로덕션 (LLM 활성화)
EVAL_LLM_ENABLED=true

# 폐쇄망 또는 개발 환경 (LLM 비활성화)
EVAL_LLM_ENABLED=false
```

초기값은 `false`이므로, 명시적으로 `true`로 설정하지 않으면 폴백이 사용된다.

### 지침 7-7: 단위 테스트에서의 Mock LLM 활용
단위 테스트에서는 실제 LLM 호출 대신 `MockLLMJudge`를 사용한다. 이를 통해:
- 테스트 속도 향상
- 외부 의존성 제거
- 다양한 LLM 응답 시뮬레이션 (성공, 실패, 파싱 오류 등)

또한 폴백 메서드 테스트도 LLM 없이 수행한다.

---

**작업 시작 예정일**: [TBD]  
**예상 종료일**: [TBD + 6-7일]  
**검수 예정일**: [TBD + 7-8일]
