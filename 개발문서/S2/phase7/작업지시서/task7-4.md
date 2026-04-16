# 작업지시서: task7-4 규칙 기반 평가 지표 구현

**작업 코드**: task7-4  
**페이즈**: Phase 7 FG7.2 (평가 러너 + 지표)  
**담당자**: [TBD]  
**예상 소요시간**: 5~6일  
**최종 수정 일시**: 2026-04-17

---

## 1. 작업 목적

RAG 시스템의 응답 품질을 정량적으로 평가하기 위한 규칙 기반 평가 지표를 구현한다. 특히 Citation-present Rate(인용 포함도), Hallucination Rate(환각 비율), Context Recall(컨텍스트 재현율)의 3가지 지표를 구현하여, 골든셋 데이터에 대한 자동 평가를 가능하게 한다. 이 지표들은 외부 LLM 없이도 동작하며, 폐쇄망 환경에서도 신뢰성 있는 평가를 제공한다.

---

## 2. 작업 범위

### 포함 범위

1. **Citation 5-tuple 형식 정의 및 감지**
   - Citation 5-tuple 형식: `(entity_type, entity_value, source_type, source_id, chunk_id)`
   - 정규식 기반 파서: `[CITE: entity_type=X, entity_value=Y, source_type=Z, source_id=W, chunk_id=V]` 형식 감지
   - Markdown 링크 형식 지원: `[text](source://source_id#chunk_id)`
   - 여러 Citation 형식 호환성 유지

2. **문장 분할 유틸리티 (Sentence Splitter)**
   - 마침표(.), 느낌표(!), 물음표(?) 기반 문장 분할
   - 약자(e.g., U.S.A, Dr., etc.)에 대한 예외 처리
   - 줄바꿈 기반 문장 분할 옵션
   - 최소 토큰 수 기준으로 짧은 문장 병합

3. **CitationPresentMetric 구현**
   - 답변 텍스트에서 각 문장별로 Citation 포함 여부 검사
   - Citation Present Rate = (인용 있는 문장 수) / (전체 문장 수)
   - 공백 답변, 단일 문장 답변 등 엣지 케이스 처리
   - 0.0 ~ 1.0 범위의 float 점수 반환

4. **HallucinationRateMetric 구현**
   - 답변 텍스트에서 Citation이 없는 문장의 비율 계산
   - Hallucination Rate = (인용 없는 문장 수) / (전체 문장 수)
   - Citation Present Rate와의 관계: Hallucination Rate = 1.0 - Citation Present Rate (논리 일관성)
   - 0.0 ~ 1.0 범위의 float 점수 반환

5. **ContextRecallMetric 구현**
   - 기대되는 소스 문서(expected_source_docs)와 실제 검색된 청크(retrieved_chunks)의 매칭
   - 소스 문서 기반 커버리지 계산: (매칭된 소스 수) / (기대 소스 수)
   - 청크 기반 커버리지 계산: (매칭된 청크 수) / (기대 청크 수)
   - 0.0 ~ 1.0 범위의 float 점수 반환

6. **MetricCalculator 추상 기반 클래스**
   - ABC 패턴으로 compute() 메서드 정의
   - Metric 인터페이스 표준화
   - 모든 구체적 지표는 MetricCalculator 상속
   - Dependency Injection을 통한 확장 가능성

7. **컴포넌트 구조**
   - `/backend/app/services/evaluation/metrics/` 디렉토리 생성
   - `rule_based.py`: 위 3가지 Metric 클래스 + 유틸리티 함수
   - `__init__.py`: 공개 인터페이스 정의

8. **단위 테스트**
   - 각 Metric별 10~15개 테스트 케이스
   - 엣지 케이스: 공백 답변, 마크다운 형식, 특수문자 포함 등
   - Citation 감지 정확도 테스트
   - 문장 분할 정확도 테스트
   - 경계값 테스트 (0.0, 1.0 등)

### 제외 범위

- LLM 기반 평가 지표 (Faithfulness, Context Precision) → task7-5에서 담당
- 임베딩 기반 평가 지표 (Answer Relevance) → task7-6에서 담당
- 성능 지표 (Latency, Token Count, Cost) → task7-6에서 담당
- 평가 실행 API 및 저장소 → task7-7에서 담당
- UI 대시보드 및 시각화 → 별도 작업

---

## 3. 선행 조건

1. **Phase 1 완료**: LLM Provider, Embedding Model 추상화 구현 완료
2. **Phase 6 완료**: Golden Set DB 모델 정의, Data Format 확정
3. **프로젝트 구조**: `/backend/app/services/` 디렉토리 존재
4. **라이브러리**: Python 3.11+, Pydantic v2, pytest
5. **Scope Profile**: 사용자별 접근 제어 정책 정의 (task7-7에서 활용)

---

## 4. 주요 작업 항목

### 4.1 Citation 형식 정의 및 레지스트리

**파일**: `/backend/app/services/evaluation/metrics/citation.py`

```python
"""
Citation 형식 정의 및 감지 모듈.

지원 Citation 형식:
1. Structured format: [CITE: entity_type=X, entity_value=Y, source_type=Z, source_id=W, chunk_id=V]
2. Markdown format: [text](source://source_id#chunk_id)
3. 커스텀 형식 확장 가능
"""

from dataclasses import dataclass
from enum import Enum
from typing import Optional, List, Tuple
import re
from abc import ABC, abstractmethod


class EntityType(str, Enum):
    """인용되는 엔티티의 종류."""
    DOCUMENT = "document"
    CHUNK = "chunk"
    PASSAGE = "passage"
    CLAIM = "claim"
    FACT = "fact"


class SourceType(str, Enum):
    """소스의 종류."""
    FILE = "file"
    WEBPAGE = "webpage"
    DATABASE = "database"
    KNOWLEDGE_BASE = "knowledge_base"
    CUSTOM = "custom"


@dataclass
class Citation:
    """표준 Citation 5-tuple."""
    entity_type: EntityType
    entity_value: str  # 인용된 텍스트 또는 개체명
    source_type: SourceType
    source_id: str  # 문서 ID, URL 등
    chunk_id: str  # 특정 청크/섹션 ID

    def to_dict(self) -> dict:
        return {
            "entity_type": self.entity_type.value,
            "entity_value": self.entity_value,
            "source_type": self.source_type.value,
            "source_id": self.source_id,
            "chunk_id": self.chunk_id,
        }

    def __str__(self) -> str:
        return (
            f"[CITE: entity_type={self.entity_type.value}, "
            f"entity_value={self.entity_value}, source_type={self.source_type.value}, "
            f"source_id={self.source_id}, chunk_id={self.chunk_id}]"
        )


class CitationParser(ABC):
    """Citation 형식 파서의 추상 기반 클래스."""

    @abstractmethod
    def parse(self, text: str) -> List[Citation]:
        """텍스트에서 Citation 목록 추출."""
        pass

    @abstractmethod
    def can_parse(self, text: str) -> bool:
        """이 파서가 처리할 수 있는 형식인지 여부."""
        pass


class StructuredCitationParser(CitationParser):
    """
    구조화된 Citation 형식 파서.
    
    예: [CITE: entity_type=chunk, entity_value=..., source_type=file, 
            source_id=doc123, chunk_id=ch001]
    """

    # 정규식: [CITE: key1=val1, key2=val2, ...]
    PATTERN = re.compile(
        r"\[CITE:\s*"
        r"entity_type=([^,]+),\s*"
        r"entity_value=([^,]+),\s*"
        r"source_type=([^,]+),\s*"
        r"source_id=([^,]+),\s*"
        r"chunk_id=([^\]]+)"
        r"\]"
    )

    def can_parse(self, text: str) -> bool:
        return "[CITE:" in text

    def parse(self, text: str) -> List[Citation]:
        citations = []
        for match in self.PATTERN.finditer(text):
            try:
                entity_type_str, entity_value, source_type_str, source_id, chunk_id = match.groups()
                citation = Citation(
                    entity_type=EntityType(entity_type_str.strip()),
                    entity_value=entity_value.strip(),
                    source_type=SourceType(source_type_str.strip()),
                    source_id=source_id.strip(),
                    chunk_id=chunk_id.strip(),
                )
                citations.append(citation)
            except (ValueError, AttributeError):
                # 파싱 실패한 Citation은 무시
                continue
        return citations


class MarkdownCitationParser(CitationParser):
    """
    Markdown 링크 형식 Citation 파서.
    
    예: [text](source://doc123#ch001)
    """

    PATTERN = re.compile(r"\[([^\]]+)\]\(([^)]+)://([^#]+)#([^\)]+)\)")

    def can_parse(self, text: str) -> bool:
        return "://(.*)#" in text or "](" in text

    def parse(self, text: str) -> List[Citation]:
        citations = []
        for match in self.PATTERN.finditer(text):
            try:
                entity_value, source_type_str, source_id, chunk_id = match.groups()
                citation = Citation(
                    entity_type=EntityType.CHUNK,
                    entity_value=entity_value.strip(),
                    source_type=SourceType(source_type_str.strip()),
                    source_id=source_id.strip(),
                    chunk_id=chunk_id.strip(),
                )
                citations.append(citation)
            except (ValueError, IndexError):
                continue
        return citations


class CitationDetector:
    """다중 형식 Citation 감지기."""

    def __init__(self):
        self.parsers: List[CitationParser] = [
            StructuredCitationParser(),
            MarkdownCitationParser(),
        ]

    def detect_citations(self, text: str) -> List[Citation]:
        """텍스트에서 모든 Citation 감지."""
        citations = []
        for parser in self.parsers:
            if parser.can_parse(text):
                citations.extend(parser.parse(text))
        # 중복 제거 (source_id, chunk_id 기반)
        unique_citations = {}
        for cite in citations:
            key = (cite.source_id, cite.chunk_id)
            if key not in unique_citations:
                unique_citations[key] = cite
        return list(unique_citations.values())

    def has_citation(self, text: str) -> bool:
        """텍스트에 Citation이 있는지 여부."""
        return len(self.detect_citations(text)) > 0
```

### 4.2 문장 분할 유틸리티

**파일**: `/backend/app/services/evaluation/metrics/sentence_splitter.py`

```python
"""
자연어 처리를 위한 문장 분할 유틸리티.

특징:
- 약자 처리 (e.g., U.S.A, Dr., Mr.)
- 줄바꿈 기반 분할
- 최소 길이 필터링
- 마크다운 형식 보존
"""

import re
from typing import List, Optional
from dataclasses import dataclass


@dataclass
class Sentence:
    """분할된 문장."""
    text: str
    start_idx: int  # 원본 텍스트에서의 시작 위치
    end_idx: int    # 원본 텍스트에서의 종료 위치
    
    def __len__(self) -> int:
        return len(self.text)
    
    def strip(self) -> str:
        return self.text.strip()


class SentenceSplitter:
    """문장 분할기."""

    # 일반적인 약자 목록
    ABBREVIATIONS = {
        "U.S.A", "U.S", "e.g", "i.e", "etc", "vs", "Dr", "Mr", "Mrs",
        "Ms", "Prof", "Sr", "Jr", "Inc", "Ltd", "Co", "Corp"
    }

    def __init__(
        self,
        min_length: int = 5,
        split_by_newline: bool = True,
        language: str = "en"
    ):
        """
        Args:
            min_length: 최소 문장 길이 (미만은 이전 문장에 병합)
            split_by_newline: 줄바꿈 기반 분할 활성화
            language: "en" 또는 "ko" (향후 확장)
        """
        self.min_length = min_length
        self.split_by_newline = split_by_newline
        self.language = language

    def _is_sentence_end(self, text: str, pos: int) -> bool:
        """위치가 문장의 끝인지 판단."""
        if pos >= len(text):
            return False

        char = text[pos]
        if char not in '.!?':
            return False

        # 약자 확인: 이전 단어가 약자인지
        if char == '.':
            # 뒤로 이동하면서 단어 추출
            word_start = pos - 1
            while word_start >= 0 and text[word_start].isalpha():
                word_start -= 1
            word = text[word_start + 1:pos]
            if word in self.ABBREVIATIONS:
                return False

        # 다음 문자가 공백 또는 개행이면 문장 끝
        if pos + 1 < len(text):
            next_char = text[pos + 1]
            if next_char in ' \n\r':
                return True
            # 따옴표나 괄호 다음은 문장 끝일 수 있음
            if next_char in '"\')':
                return True
            return False

        return True

    def split(self, text: str) -> List[Sentence]:
        """텍스트를 문장으로 분할."""
        if not text or not text.strip():
            return []

        sentences = []
        current_start = 0

        # 줄바꿈 기반 분할 (선택)
        if self.split_by_newline:
            lines = text.split('\n')
            for line in lines:
                if line.strip():
                    sent = Sentence(
                        text=line.strip(),
                        start_idx=current_start,
                        end_idx=current_start + len(line)
                    )
                    if len(sent.strip()) >= self.min_length:
                        sentences.append(sent)
                current_start += len(line) + 1  # +1 for newline
            return sentences

        # 마침표/느낌표/물음표 기반 분할
        i = 0
        current_start = 0

        while i < len(text):
            if self._is_sentence_end(text, i):
                # 문장 끝 발견
                sentence_text = text[current_start:i + 1].strip()
                if len(sentence_text) >= self.min_length:
                    sent = Sentence(
                        text=sentence_text,
                        start_idx=current_start,
                        end_idx=i + 1
                    )
                    sentences.append(sent)
                # 다음 문장의 시작 (공백 건너뛰기)
                current_start = i + 1
                while current_start < len(text) and text[current_start] in ' \n\r':
                    current_start += 1
            i += 1

        # 마지막 문장
        if current_start < len(text):
            remaining = text[current_start:].strip()
            if len(remaining) >= self.min_length:
                sent = Sentence(
                    text=remaining,
                    start_idx=current_start,
                    end_idx=len(text)
                )
                sentences.append(sent)

        return sentences


# 기본 인스턴스
DEFAULT_SPLITTER = SentenceSplitter()


def split_sentences(text: str) -> List[str]:
    """편의 함수: 텍스트를 문장으로 분할하고 문자열 목록 반환."""
    return [s.strip() for s in DEFAULT_SPLITTER.split(text)]
```

### 4.3 규칙 기반 Metric 클래스들

**파일**: `/backend/app/services/evaluation/metrics/rule_based.py`

```python
"""
규칙 기반 평가 지표 구현.

포함:
- CitationPresentMetric: 문장별 Citation 포함도
- HallucinationRateMetric: Citation이 없는 문장의 비율
- ContextRecallMetric: 기대 소스 문서 vs 실제 검색된 청크
"""

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Dict, Optional, Set, Tuple
from datetime import datetime
import logging

from .citation import CitationDetector, Citation
from .sentence_splitter import SentenceSplitter, Sentence


logger = logging.getLogger(__name__)


@dataclass
class MetricScore:
    """지표 점수."""
    metric_name: str
    score: float  # 0.0 ~ 1.0
    details: Dict = None  # 상세 정보 (선택)
    computed_at: datetime = None

    def __post_init__(self):
        if self.computed_at is None:
            self.computed_at = datetime.utcnow()
        if self.details is None:
            self.details = {}

        # 점수 검증
        if not (0.0 <= self.score <= 1.0):
            raise ValueError(f"Metric score must be between 0.0 and 1.0, got {self.score}")


class MetricCalculator(ABC):
    """지표 계산기 추상 기반 클래스."""

    @abstractmethod
    def compute(self, **kwargs) -> MetricScore:
        """지표 계산 (구체 구현은 서브클래스)."""
        pass

    @property
    @abstractmethod
    def metric_name(self) -> str:
        """지표 이름."""
        pass


class CitationPresentMetric(MetricCalculator):
    """
    Citation Present Rate 지표.
    
    = (Citation이 있는 문장 수) / (전체 문장 수)
    
    범위: 0.0 ~ 1.0 (1.0이 최고)
    """

    def __init__(self, sentence_splitter: Optional[SentenceSplitter] = None):
        self.sentence_splitter = sentence_splitter or SentenceSplitter()
        self.citation_detector = CitationDetector()

    @property
    def metric_name(self) -> str:
        return "citation_present_rate"

    def compute(
        self,
        answer_text: str,
        **kwargs
    ) -> MetricScore:
        """
        Args:
            answer_text: 평가할 답변 텍스트
            **kwargs: 추가 인자 (무시)
        
        Returns:
            MetricScore with citation_present_rate
        """
        if not answer_text or not answer_text.strip():
            # 공백 답변: Citation 불가능 → 0.0
            return MetricScore(
                metric_name=self.metric_name,
                score=0.0,
                details={"num_sentences": 0, "cited_sentences": 0}
            )

        # 문장 분할
        sentences = self.sentence_splitter.split(answer_text)
        if not sentences:
            return MetricScore(
                metric_name=self.metric_name,
                score=0.0,
                details={"num_sentences": 0, "cited_sentences": 0}
            )

        # 각 문장의 Citation 여부 확인
        cited_count = 0
        for sentence in sentences:
            if self.citation_detector.has_citation(sentence.text):
                cited_count += 1

        total = len(sentences)
        rate = cited_count / total if total > 0 else 0.0

        return MetricScore(
            metric_name=self.metric_name,
            score=rate,
            details={
                "num_sentences": total,
                "cited_sentences": cited_count,
                "uncited_sentences": total - cited_count
            }
        )


class HallucinationRateMetric(MetricCalculator):
    """
    Hallucination Rate 지표.
    
    = (Citation이 없는 문장 수) / (전체 문장 수)
    = 1.0 - Citation Present Rate
    
    범위: 0.0 ~ 1.0 (0.0이 최고, 환각 없음)
    """

    def __init__(self, sentence_splitter: Optional[SentenceSplitter] = None):
        self.citation_present_metric = CitationPresentMetric(sentence_splitter)

    @property
    def metric_name(self) -> str:
        return "hallucination_rate"

    def compute(
        self,
        answer_text: str,
        **kwargs
    ) -> MetricScore:
        """
        Args:
            answer_text: 평가할 답변 텍스트
            **kwargs: 추가 인자 (무시)
        
        Returns:
            MetricScore with hallucination_rate
        """
        citation_score = self.citation_present_metric.compute(answer_text=answer_text)
        
        hallucination_rate = 1.0 - citation_score.score
        
        return MetricScore(
            metric_name=self.metric_name,
            score=hallucination_rate,
            details={
                "citation_present_rate": citation_score.score,
                **citation_score.details
            }
        )


class ContextRecallMetric(MetricCalculator):
    """
    Context Recall 지표.
    
    = (실제 검색된 청크 중 기대 소스에 포함된 것) / (기대 소스 수)
    
    범위: 0.0 ~ 1.0 (1.0이 최고, 모든 기대 소스 검색됨)
    """

    @property
    def metric_name(self) -> str:
        return "context_recall"

    def compute(
        self,
        expected_source_docs: List[str],
        retrieved_chunks: List[Dict],
        **kwargs
    ) -> MetricScore:
        """
        Args:
            expected_source_docs: 기대되는 소스 문서 ID 목록
                예: ["doc_001", "doc_002"]
            retrieved_chunks: 실제 검색된 청크 목록
                예: [
                    {"chunk_id": "ch_001", "source_id": "doc_001"},
                    {"chunk_id": "ch_002", "source_id": "doc_002"},
                    ...
                ]
            **kwargs: 추가 인자 (무시)
        
        Returns:
            MetricScore with context_recall
        """
        if not expected_source_docs:
            # 기대 소스가 없으면 모두 검색된 것으로 간주
            return MetricScore(
                metric_name=self.metric_name,
                score=1.0,
                details={
                    "expected_sources": 0,
                    "found_sources": 0
                }
            )

        # 검색된 청크에서 source_id 추출
        retrieved_sources = set()
        for chunk in retrieved_chunks:
            if isinstance(chunk, dict) and "source_id" in chunk:
                retrieved_sources.add(chunk["source_id"])
            elif isinstance(chunk, str):
                # 청크가 문자열인 경우 source_id로 처리
                retrieved_sources.add(chunk)

        expected_set = set(expected_source_docs)
        found_sources = expected_set & retrieved_sources
        
        recall = len(found_sources) / len(expected_set) if expected_set else 1.0

        return MetricScore(
            metric_name=self.metric_name,
            score=recall,
            details={
                "expected_sources": list(expected_set),
                "retrieved_sources": list(retrieved_sources),
                "found_sources": list(found_sources),
                "missing_sources": list(expected_set - retrieved_sources),
                "recall_percentage": recall * 100
            }
        )


class MetricRegistry:
    """지표 계산기 레지스트리."""

    def __init__(self):
        self.calculators: Dict[str, MetricCalculator] = {}

    def register(self, calculator: MetricCalculator) -> None:
        """지표 계산기 등록."""
        self.calculators[calculator.metric_name] = calculator

    def get(self, metric_name: str) -> Optional[MetricCalculator]:
        """지표 계산기 조회."""
        return self.calculators.get(metric_name)

    def list_metrics(self) -> List[str]:
        """등록된 지표 목록."""
        return list(self.calculators.keys())

    def compute_all(
        self,
        metrics: List[str],
        **kwargs
    ) -> Dict[str, MetricScore]:
        """
        여러 지표를 일괄 계산.
        
        Args:
            metrics: 계산할 지표 이름 목록
            **kwargs: 지표 계산에 필요한 모든 인자
        
        Returns:
            {metric_name: MetricScore} 딕셔너리
        """
        results = {}
        for metric_name in metrics:
            calculator = self.get(metric_name)
            if calculator is None:
                logger.warning(f"Unknown metric: {metric_name}")
                continue
            try:
                results[metric_name] = calculator.compute(**kwargs)
            except Exception as e:
                logger.error(f"Error computing {metric_name}: {e}")
                # 계산 실패 시 0.0 점수 기록
                results[metric_name] = MetricScore(
                    metric_name=metric_name,
                    score=0.0,
                    details={"error": str(e)}
                )
        return results


# 기본 레지스트리
DEFAULT_REGISTRY = MetricRegistry()
DEFAULT_REGISTRY.register(CitationPresentMetric())
DEFAULT_REGISTRY.register(HallucinationRateMetric())
DEFAULT_REGISTRY.register(ContextRecallMetric())
```

### 4.4 통합 초기화 모듈

**파일**: `/backend/app/services/evaluation/metrics/__init__.py`

```python
"""
평가 지표 모듈.

공개 인터페이스:
- CitationDetector: Citation 감지
- SentenceSplitter: 문장 분할
- MetricCalculator: 지표 계산 기반 클래스
- CitationPresentMetric, HallucinationRateMetric, ContextRecallMetric: 구체 구현
- MetricRegistry, DEFAULT_REGISTRY: 지표 레지스트리
"""

from .citation import (
    Citation,
    EntityType,
    SourceType,
    CitationDetector,
    CitationParser,
    StructuredCitationParser,
    MarkdownCitationParser,
)
from .sentence_splitter import SentenceSplitter, Sentence, split_sentences
from .rule_based import (
    MetricScore,
    MetricCalculator,
    CitationPresentMetric,
    HallucinationRateMetric,
    ContextRecallMetric,
    MetricRegistry,
    DEFAULT_REGISTRY,
)

__all__ = [
    # Citation
    "Citation",
    "EntityType",
    "SourceType",
    "CitationDetector",
    "CitationParser",
    "StructuredCitationParser",
    "MarkdownCitationParser",
    # Sentence splitting
    "SentenceSplitter",
    "Sentence",
    "split_sentences",
    # Metrics
    "MetricScore",
    "MetricCalculator",
    "CitationPresentMetric",
    "HallucinationRateMetric",
    "ContextRecallMetric",
    "MetricRegistry",
    "DEFAULT_REGISTRY",
]
```

### 4.5 단위 테스트

**파일**: `/backend/tests/services/evaluation/metrics/test_rule_based.py`

```python
"""
규칙 기반 평가 지표 단위 테스트.
"""

import pytest
from app.services.evaluation.metrics import (
    CitationPresentMetric,
    HallucinationRateMetric,
    ContextRecallMetric,
    CitationDetector,
    SentenceSplitter,
    MetricScore,
)


class TestCitationPresentMetric:
    """CitationPresentMetric 테스트."""

    def setup_method(self):
        self.metric = CitationPresentMetric()

    def test_citation_present_full(self):
        """모든 문장에 Citation이 있는 경우."""
        answer = (
            "The capital is Paris. [CITE: entity_type=fact, entity_value=Paris, "
            "source_type=file, source_id=doc123, chunk_id=ch001] "
            "It is located in France. [CITE: entity_type=fact, entity_value=France, "
            "source_type=file, source_id=doc123, chunk_id=ch002]"
        )
        result = self.metric.compute(answer_text=answer)
        assert result.score == 1.0
        assert result.details["num_sentences"] == 2
        assert result.details["cited_sentences"] == 2

    def test_citation_present_partial(self):
        """일부 문장에만 Citation이 있는 경우."""
        answer = (
            "The capital is Paris. [CITE: entity_type=fact, entity_value=Paris, "
            "source_type=file, source_id=doc123, chunk_id=ch001] "
            "It is beautiful."
        )
        result = self.metric.compute(answer_text=answer)
        assert result.score == 0.5
        assert result.details["num_sentences"] == 2
        assert result.details["cited_sentences"] == 1

    def test_citation_present_none(self):
        """Citation이 없는 경우."""
        answer = "The capital is Paris. It is beautiful."
        result = self.metric.compute(answer_text=answer)
        assert result.score == 0.0
        assert result.details["num_sentences"] == 2
        assert result.details["cited_sentences"] == 0

    def test_empty_answer(self):
        """공백 답변."""
        result = self.metric.compute(answer_text="")
        assert result.score == 0.0
        assert result.details["num_sentences"] == 0

    def test_single_sentence_with_citation(self):
        """단일 문장 with Citation."""
        answer = "Paris is the capital. [CITE: entity_type=fact, entity_value=Paris, source_type=file, source_id=doc123, chunk_id=ch001]"
        result = self.metric.compute(answer_text=answer)
        assert result.score == 1.0

    def test_markdown_citation_format(self):
        """Markdown Citation 형식."""
        answer = "The capital is [Paris](file://doc123#ch001). It's beautiful."
        result = self.metric.compute(answer_text=answer)
        assert result.score == 0.5  # 첫 문장만 Citation


class TestHallucinationRateMetric:
    """HallucinationRateMetric 테스트."""

    def setup_method(self):
        self.metric = HallucinationRateMetric()

    def test_hallucination_none(self):
        """환각 없음 (모든 문장에 Citation)."""
        answer = (
            "The capital is Paris. [CITE: entity_type=fact, entity_value=Paris, "
            "source_type=file, source_id=doc123, chunk_id=ch001]"
        )
        result = self.metric.compute(answer_text=answer)
        assert result.score == 0.0

    def test_hallucination_partial(self):
        """부분 환각 (일부 문장에만 Citation)."""
        answer = (
            "The capital is Paris. [CITE: entity_type=fact, entity_value=Paris, "
            "source_type=file, source_id=doc123, chunk_id=ch001] "
            "It is beautiful."
        )
        result = self.metric.compute(answer_text=answer)
        assert result.score == 0.5

    def test_hallucination_full(self):
        """완전 환각 (Citation 없음)."""
        answer = "The capital is Paris. It is beautiful."
        result = self.metric.compute(answer_text=answer)
        assert result.score == 1.0


class TestContextRecallMetric:
    """ContextRecallMetric 테스트."""

    def setup_method(self):
        self.metric = ContextRecallMetric()

    def test_context_recall_full(self):
        """모든 기대 소스 검색됨."""
        expected_sources = ["doc_001", "doc_002"]
        retrieved_chunks = [
            {"chunk_id": "ch_001", "source_id": "doc_001"},
            {"chunk_id": "ch_002", "source_id": "doc_002"},
        ]
        result = self.metric.compute(
            expected_source_docs=expected_sources,
            retrieved_chunks=retrieved_chunks
        )
        assert result.score == 1.0

    def test_context_recall_partial(self):
        """일부 기대 소스만 검색됨."""
        expected_sources = ["doc_001", "doc_002"]
        retrieved_chunks = [
            {"chunk_id": "ch_001", "source_id": "doc_001"},
        ]
        result = self.metric.compute(
            expected_source_docs=expected_sources,
            retrieved_chunks=retrieved_chunks
        )
        assert result.score == 0.5
        assert len(result.details["missing_sources"]) == 1

    def test_context_recall_none(self):
        """기대 소스가 검색되지 않음."""
        expected_sources = ["doc_001", "doc_002"]
        retrieved_chunks = [
            {"chunk_id": "ch_001", "source_id": "doc_003"},
        ]
        result = self.metric.compute(
            expected_source_docs=expected_sources,
            retrieved_chunks=retrieved_chunks
        )
        assert result.score == 0.0

    def test_context_recall_empty_expected(self):
        """기대 소스가 없는 경우."""
        result = self.metric.compute(
            expected_source_docs=[],
            retrieved_chunks=[{"chunk_id": "ch_001", "source_id": "doc_001"}]
        )
        assert result.score == 1.0

    def test_context_recall_duplicate_sources(self):
        """검색된 청크에 중복 source_id."""
        expected_sources = ["doc_001"]
        retrieved_chunks = [
            {"chunk_id": "ch_001", "source_id": "doc_001"},
            {"chunk_id": "ch_002", "source_id": "doc_001"},
        ]
        result = self.metric.compute(
            expected_source_docs=expected_sources,
            retrieved_chunks=retrieved_chunks
        )
        assert result.score == 1.0


class TestSentenceSplitter:
    """SentenceSplitter 테스트."""

    def setup_method(self):
        self.splitter = SentenceSplitter()

    def test_basic_split(self):
        """기본 문장 분할."""
        text = "The capital is Paris. It is beautiful."
        sentences = self.splitter.split(text)
        assert len(sentences) == 2
        assert sentences[0].text == "The capital is Paris."
        assert sentences[1].text == "It is beautiful."

    def test_split_with_exclamation(self):
        """느낌표 포함 분할."""
        text = "What a beautiful city! Paris is the capital."
        sentences = self.splitter.split(text)
        assert len(sentences) == 2

    def test_split_with_question(self):
        """물음표 포함 분할."""
        text = "Is Paris the capital? Yes, it is."
        sentences = self.splitter.split(text)
        assert len(sentences) == 2

    def test_split_with_abbreviation(self):
        """약자 포함 분할 (약자는 분할하지 않음)."""
        text = "The U.S.A is large. Paris is small."
        sentences = self.splitter.split(text)
        # U.S.A 다음은 분할하지 않아야 함
        assert len(sentences) <= 2

    def test_empty_text(self):
        """공백 텍스트."""
        sentences = self.splitter.split("")
        assert len(sentences) == 0

    def test_split_by_newline(self):
        """줄바꿈 기반 분할."""
        splitter = SentenceSplitter(split_by_newline=True)
        text = "Line 1\nLine 2\nLine 3"
        sentences = splitter.split(text)
        assert len(sentences) == 3

    def test_min_length_filtering(self):
        """최소 길이 필터링."""
        splitter = SentenceSplitter(min_length=10)
        text = "Hi. This is a longer sentence."
        sentences = splitter.split(text)
        # "Hi."는 필터링됨
        assert all(len(s) >= 10 for s in sentences)


class TestCitationDetector:
    """CitationDetector 통합 테스트."""

    def setup_method(self):
        self.detector = CitationDetector()

    def test_detect_structured_format(self):
        """구조화된 Citation 형식 감지."""
        text = "Some text [CITE: entity_type=chunk, entity_value=info, source_type=file, source_id=doc123, chunk_id=ch001]"
        citations = self.detector.detect_citations(text)
        assert len(citations) == 1
        assert citations[0].source_id == "doc123"
        assert citations[0].chunk_id == "ch001"

    def test_detect_markdown_format(self):
        """Markdown Citation 형식 감지."""
        text = "Some text [info](file://doc123#ch001)"
        citations = self.detector.detect_citations(text)
        assert len(citations) == 1
        assert citations[0].source_id == "doc123"
        assert citations[0].chunk_id == "ch001"

    def test_has_citation(self):
        """Citation 포함 여부 확인."""
        text_with_citation = "Text [CITE: entity_type=chunk, entity_value=info, source_type=file, source_id=doc123, chunk_id=ch001]"
        text_without_citation = "Plain text without citation"
        
        assert self.detector.has_citation(text_with_citation)
        assert not self.detector.has_citation(text_without_citation)

    def test_multiple_citations(self):
        """여러 Citation 감지."""
        text = (
            "First [CITE: entity_type=chunk, entity_value=a, source_type=file, source_id=doc1, chunk_id=ch1] "
            "Second [CITE: entity_type=chunk, entity_value=b, source_type=file, source_id=doc2, chunk_id=ch2]"
        )
        citations = self.detector.detect_citations(text)
        assert len(citations) == 2

    def test_duplicate_citation_deduplication(self):
        """중복 Citation 제거."""
        text = (
            "First [CITE: entity_type=chunk, entity_value=a, source_type=file, source_id=doc1, chunk_id=ch1] "
            "Second [CITE: entity_type=chunk, entity_value=b, source_type=file, source_id=doc1, chunk_id=ch1]"
        )
        citations = self.detector.detect_citations(text)
        assert len(citations) == 1  # 중복 제거됨
```

---

## 5. 산출물

1. **지표 구현 모듈**
   - `/backend/app/services/evaluation/metrics/citation.py` (500줄)
   - `/backend/app/services/evaluation/metrics/sentence_splitter.py` (300줄)
   - `/backend/app/services/evaluation/metrics/rule_based.py` (400줄)
   - `/backend/app/services/evaluation/metrics/__init__.py` (50줄)

2. **단위 테스트**
   - `/backend/tests/services/evaluation/metrics/test_rule_based.py` (350줄)

3. **문서**
   - 이 작업지시서 (task7-4.md)
   - 코드 내 주석 및 docstring
   - 검수보고서 (task7-4-검수보고서.md) - 완료 후 작성

4. **선택 산출물**
   - 보안 취약점 검사 보고서 (task7-4-보안검사보고서.md) - 완료 후 작성

---

## 6. 완료 기준

1. **Citation 감지 기능**
   - [ ] StructuredCitationParser가 `[CITE: ...]` 형식을 정확히 파싱
   - [ ] MarkdownCitationParser가 `[text](source://id#chunk)` 형식 파싱
   - [ ] CitationDetector가 여러 형식을 혼합하여 감지
   - [ ] 중복 Citation 자동 제거

2. **문장 분할 기능**
   - [ ] 마침표, 느낌표, 물음표 기반 기본 분할
   - [ ] 약자(e.g., U.S.A) 예외 처리
   - [ ] 줄바꿈 기반 분할 옵션 동작
   - [ ] 최소 길이 필터링 정확
   - [ ] 엣지 케이스: 공백 텍스트, 단일 문장, 특수문자

3. **CitationPresentMetric**
   - [ ] 0.0~1.0 범위의 정규화된 점수 반환
   - [ ] 문장 수, 인용된 문장 수 등 상세 정보 제공
   - [ ] 공백 답변 → 0.0 점수
   - [ ] 모든 문장에 Citation → 1.0 점수

4. **HallucinationRateMetric**
   - [ ] Citation Present Rate의 반수 계산 (1.0 - CPR)
   - [ ] 모든 문장에 Citation → 0.0 점수 (환각 없음)
   - [ ] Citation 없는 모든 문장 → 1.0 점수 (완전 환각)

5. **ContextRecallMetric**
   - [ ] 기대 소스 문서 ID와 실제 검색된 청크의 source_id 매칭
   - [ ] 0.0~1.0 범위의 정규화된 점수 반환
   - [ ] missing_sources 목록 제공
   - [ ] 기대 소스 없는 경우 → 1.0 점수

6. **MetricRegistry**
   - [ ] 지표 계산기 등록/조회 기능
   - [ ] compute_all()로 여러 지표 일괄 계산
   - [ ] 계산 실패 시 에러 로깅 및 0.0 점수 기록

7. **단위 테스트**
   - [ ] 각 Metric별 최소 5개 이상의 테스트 케이스
   - [ ] SentenceSplitter 최소 6개 테스트
   - [ ] CitationDetector 최소 5개 테스트
   - [ ] 모든 테스트 통과 (pytest 결과 100%)
   - [ ] 엣지 케이스 커버: 공백, 마크다운, 특수문자, 중복

8. **코드 품질**
   - [ ] Type hints 완전히 적용
   - [ ] Docstring 모든 함수/클래스에 적용
   - [ ] 로그 라인 추가 (INFO, WARNING, ERROR)
   - [ ] 예외 처리 적절함
   - [ ] 코드 스타일 PEP 8 준수

---

## 7. 작업 지침

### 지침 7-1: Citation 형식 확장 가능성
구조화된 Citation 형식과 Markdown 형식을 모두 지원하되, `CitationParser` ABC를 통해 새로운 형식을 쉽게 추가할 수 있도록 설계한다. 향후 프로젝트에서 다른 Citation 형식이 필요할 경우 `CitationParser` 서브클래스를 구현하면 된다.

### 지침 7-2: 문장 분할의 언어 의존성
현재 구현은 영어(English) 중심이다. 향후 한국어, 중국어 등 다국어 지원이 필요한 경우 `SentenceSplitter`의 `language` 매개변수를 확장하고, 각 언어별 약자 사전을 추가한다. 현 단계에서는 `language="en"` 기본값으로 충분하다.

### 지침 7-3: 점수 정규화의 일관성
모든 지표는 0.0 ~ 1.0 범위로 정규화된 점수를 반환한다. `MetricScore` 클래스에서 점수 검증을 수행하므로, 계산 로직에서 범위를 벗어난 값이 발생하면 안 된다. 특히 0으로 나누는 경우(division by zero)를 명시적으로 처리한다.

### 지침 7-4: 상세 정보(Details) 설계
각 지표는 점수뿐 아니라 `MetricScore.details` 딕셔너리에 상세 정보를 담는다. 예를 들어 CitationPresentMetric은 `{"num_sentences": 10, "cited_sentences": 8, "uncited_sentences": 2}`를 제공한다. 이 정보는 UI 대시보드에서 시각화하거나, 검증 보고서 생성에 활용될 수 있다.

### 지침 7-5: 성능 고려사항
현재 구현은 단일 스레드 동기 처리를 가정한다. 그러나 대량의 평가 작업이 필요한 경우(예: 수천 개의 골든 아이템), task7-7의 `Evaluator` 클래스에서 `asyncio.gather()`를 사용하여 병렬 처리할 것으로 예상된다. 따라서 각 Metric 클래스는 상태를 변경하지 않도록(stateless) 설계되어야 한다.

### 지침 7-6: 테스트 커버리지와 엣지 케이스
단위 테스트에서는 다음 엣지 케이스를 반드시 포함한다:
- 공백 또는 None 입력
- 단일 문장
- 매우 긴 텍스트 (1000줄 이상)
- 마크다운, HTML, 특수문자 포함
- Citation이 문장 중간에 있는 경우
- 중복되는 Citation
- 기대 소스가 없는 경우

테스트는 pytest로 실행하며, 최소 80% 라인 커버리지를 목표로 한다.

### 지침 7-7: 로깅과 모니터링
모든 Metric 클래스는 로거를 사용하여 다음을 기록한다:
- 계산 시작/종료
- 파싱 오류 (예: 잘못된 Citation 형식)
- 성능 경고 (예: 매우 긴 텍스트)
- 예외 발생

로그 레벨:
- `INFO`: 정상 계산 완료
- `WARNING`: 데이터 오류 (예: source_id 없음)
- `ERROR`: 계산 실패, 예외 발생

이 로그는 프로덕션 환경에서 평가 시스템 모니터링에 사용된다.

---

**작업 시작 예정일**: [TBD]  
**예상 종료일**: [TBD + 5-6일]  
**검수 예정일**: [TBD + 6-7일]
