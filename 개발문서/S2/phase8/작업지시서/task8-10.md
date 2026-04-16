# FG8.3 작업지시서 - task8-10
# 추출 품질 평가 (ExtractionEvaluator) + 보고서

**버전**: 1.0  
**작성일**: 2026-04-17  
**FG**: FG8.3 — 추출 결과의 검증 가능 계약  
**담당**: Phase 8 팀

---

## 1. 작업 목적

추출 성능을 정량적으로 평가하고, 추출 품질을 개선하기 위한 기초 데이터를 생성한다. 이를 통해:

- 추출 정확도(accuracy), span 정확도(span accuracy) 측정
- 필드별 품질 점수 계산
- CI/CD 파이프라인에 품질 게이트 통합
- 대시보드에서 품질 메트릭 시각화
- 모델/프롬프트 변경의 영향도 정량 측정

---

## 2. 작업 범위

### 포함 범위

1. **GoldenExtractionItem 데이터 모델**
   - Golden reference (기준) 데이터: document, expected_fields, expected_spans
   - Pydantic 정의 + 버전 관리

2. **GoldenExtractionSet 관리**
   - Golden item 컬렉션
   - 메타데이터: name, description, document_type, version, created_by
   - CRUD API (create, read, update, delete)

3. **ExtractionEvaluator 클래스**
   - Phase 7 Evaluator 상속
   - evaluate_extraction(): 단일 item 평가
   - evaluate_extraction_set(): 배치 평가
   - 모든 평가 메서드에서 span 비교 포함

4. **품질 메트릭 구현**
   - **Field Accuracy**: 필드별 정확도 (exact / fuzzy 모두 지원)
   - **Span Accuracy**: IoU (Intersection over Union) 기반 span 위치 정확도
   - **Required Field Coverage**: 필수 필드 추출 성공률
   - **Type Correctness**: 추출 값의 타입이 스키마와 일치하는 비율
   - **Performance**: 지연시간, 토큰 수, 비용 (Phase 7 재사용)

5. **평가 API**
   - POST /api/v1/extraction-evaluations/run — 평가 실행
   - GET /api/v1/extraction-evaluations/{eval_id} — 평가 결과 조회
   - GET /api/v1/extraction-evaluations/{eval_id}/compare — 평가 비교 (A/B test)

6. **평가 결과 저장**
   - extraction_evaluations 테이블
   - field_evaluation_details 테이블
   - 평가 히스토리 추적

7. **품질 보고서 생성**
   - Jinja2 템플릿 기반 마크다운 보고서
   - 필드별 분석, span 정확도, 성능 메트릭 포함
   - HTML 렌더링 지원 (선택사항)

8. **CI 게이트 통합**
   - Phase 7 품질 임계값 재사용
   - 추가 임계값: Span Accuracy >= 0.80, Required Field Coverage >= 0.95
   - 게이트 통과/실패 시 webhook 트리거

9. **대시보드 데이터 바인딩** (Phase 6 FG6.3)
   - 평가 메트릭을 대시보드에 제공하는 데이터 API
   - 실시간 품질 지표 표시

10. **단위 + 통합 테스트**
    - 메트릭 계산 정확성
    - Golden set 관리
    - 평가 API
    - 보고서 생성
    - CI 게이트

### 제외 범위

- 자동 평가 스케줄링 (수동 API 호출만)
- 머신러닝 기반 이상 탐지
- 모델 재학습 자동화
- 다국어 평가 가중치 (동등하게 취급)

---

## 3. 선행 조건

1. task8-8, task8-9 완료: SourceSpan, ExtractionRecord 모델 존재
2. Phase 7 완료: Evaluator 기본 클래스, 평가 프레임워크
3. Phase 6 완료: 대시보드 API 기본 구조
4. Phase 5 완료: ExtractionCandidate, ExtractionSchema 모델
5. Jinja2 템플릿 엔진 설치 확인
6. Database 마이그레이션 환경 준비
7. 테스트용 Golden data set 샘플 준비

---

## 4. 주요 작업 항목

### 4.1 GoldenExtractionItem 데이터 모델

**파일**: `/mnt/mimir/src/core/models/evaluation.py` (신규)

```python
"""
추출 품질 평가 관련 Pydantic 모델.
"""

from pydantic import BaseModel, Field, field_validator
from typing import Optional, Dict, Any, List, Tuple
from datetime import datetime
from enum import Enum

class FieldTypeEnum(str, Enum):
    """필드 타입"""
    STRING = "string"
    INTEGER = "integer"
    FLOAT = "float"
    BOOLEAN = "boolean"
    DATE = "date"
    ARRAY = "array"
    OBJECT = "object"


class ExpectedField(BaseModel):
    """
    추출 스키마의 필드에 대한 기준 값.
    
    Attributes:
        field_name: 필드명
        expected_value: 기준이 되는 정확한 값
        field_type: 필드 타입
        required: 필수 필드 여부
        description: 필드 설명
    """
    field_name: str = Field(..., description="필드명")
    expected_value: Any = Field(..., description="기준 값")
    field_type: FieldTypeEnum = Field(..., description="필드 타입")
    required: bool = Field(default=True, description="필수 필드 여부")
    description: Optional[str] = Field(None, description="필드 설명")
    
    @field_validator('expected_value')
    @classmethod
    def validate_value_type(cls, v: Any, info) -> Any:
        """필드 타입과 값의 타입이 일치하는지 검증"""
        field_type = info.data.get('field_type')
        
        if field_type == FieldTypeEnum.STRING and not isinstance(v, str):
            raise ValueError("expected_value must be string for STRING type")
        elif field_type == FieldTypeEnum.INTEGER and not isinstance(v, int):
            raise ValueError("expected_value must be int for INTEGER type")
        elif field_type == FieldTypeEnum.FLOAT and not isinstance(v, (int, float)):
            raise ValueError("expected_value must be float for FLOAT type")
        elif field_type == FieldTypeEnum.BOOLEAN and not isinstance(v, bool):
            raise ValueError("expected_value must be bool for BOOLEAN type")
        
        return v


class ExpectedSpan(BaseModel):
    """
    추출 결과의 기준이 되는 원문 위치.
    
    Attributes:
        field_name: 어느 필드의 span 인가
        span_offset: (start, end) 문자 인덱스
        source_text: 실제 텍스트 (검증용)
        node_id: 문서 노드 ID (선택)
    """
    field_name: str = Field(..., description="필드명")
    span_offset: Tuple[int, int] = Field(..., description="(start, end) 인덱스")
    source_text: str = Field(..., description="실제 텍스트")
    node_id: Optional[str] = Field(None, description="노드 ID")
    
    @field_validator('span_offset')
    @classmethod
    def validate_span(cls, v: Tuple[int, int]) -> Tuple[int, int]:
        """Span 오프셋 검증"""
        if len(v) != 2 or v[0] >= v[1]:
            raise ValueError("span_offset must be (start, end) where start < end")
        return v


class GoldenExtractionItem(BaseModel):
    """
    추출 품질 평가를 위한 Golden reference (기준) 데이터.
    
    실제 문서와 그에 대한 정확한 추출 결과를 정의합니다.
    평가 시 모델의 출력과 이 기준값을 비교합니다.
    """
    id: str = Field(..., description="Golden item ID (UUID)")
    document_id: str = Field(..., description="원본 문서 ID")
    document_version: str = Field(..., description="문서 버전")
    document_type: str = Field(..., description="문서 타입 (config 기반)")
    
    # 기준 추출 결과
    expected_fields: List[ExpectedField] = Field(
        ..., description="기준이 되는 필드 값 리스트"
    )
    
    # 원문 위치 정보 (span 기반 평가용)
    expected_spans: Optional[List[ExpectedSpan]] = Field(
        default_factory=list, description="기준이 되는 span 리스트"
    )
    
    # 메타데이터
    extraction_schema_id: str = Field(..., description="추출 스키마 ID")
    created_by: str = Field(..., description="작성자 (user ID)")
    created_at: datetime = Field(default_factory=datetime.utcnow)
    notes: Optional[str] = Field(None, description="평가 노트")
    
    def get_required_fields(self) -> List[str]:
        """필수 필드 목록 반환"""
        return [f.field_name for f in self.expected_fields if f.required]
    
    def get_field_by_name(self, field_name: str) -> Optional[ExpectedField]:
        """필드명으로 필드 조회"""
        for field in self.expected_fields:
            if field.field_name == field_name:
                return field
        return None
    
    def get_spans_for_field(self, field_name: str) -> List[ExpectedSpan]:
        """필드명으로 span 조회"""
        return [s for s in self.expected_spans if s.field_name == field_name]


class GoldenExtractionSet(BaseModel):
    """
    Golden item 컬렉션.
    
    여러 golden item 을 그룹화하여 배치 평가 시 사용합니다.
    """
    id: str = Field(..., description="Set ID (UUID)")
    name: str = Field(..., description="Set 이름")
    description: Optional[str] = Field(None, description="설명")
    document_type: str = Field(..., description="문서 타입")
    extraction_schema_id: str = Field(..., description="추출 스키마 ID")
    
    # 포함된 item 들
    items: List[GoldenExtractionItem] = Field(
        default_factory=list, description="Golden item 리스트"
    )
    
    # 메타데이터
    version: str = Field(default="1.0.0", description="Set 버전")
    created_by: str = Field(..., description="작성자")
    created_at: datetime = Field(default_factory=datetime.utcnow)
    
    def add_item(self, item: GoldenExtractionItem):
        """Item 추가"""
        self.items.append(item)
    
    def remove_item(self, item_id: str):
        """Item 제거"""
        self.items = [i for i in self.items if i.id != item_id]
    
    def get_item(self, item_id: str) -> Optional[GoldenExtractionItem]:
        """Item 조회"""
        for item in self.items:
            if item.id == item_id:
                return item
        return None
    
    def item_count(self) -> int:
        """포함된 item 개수"""
        return len(self.items)


class ExtractionMetrics(BaseModel):
    """
    추출 평가 결과의 메트릭.
    """
    # 필드 정확도
    field_accuracy: float = Field(..., description="필드 정확도 (0.0 ~ 1.0)")
    exact_match_count: int = Field(default=0, description="정확 일치 필드 수")
    fuzzy_match_count: int = Field(default=0, description="Fuzzy 일치 필드 수")
    mismatch_count: int = Field(default=0, description="불일치 필드 수")
    
    # Span 정확도
    span_accuracy: Optional[float] = Field(None, description="Span 정확도 (IoU 평균)")
    span_iou_scores: Optional[List[float]] = Field(
        None, description="필드별 IoU 점수"
    )
    
    # 필드 커버리지
    required_field_coverage: float = Field(
        ..., description="필수 필드 추출 성공률 (0.0 ~ 1.0)"
    )
    required_fields_extracted: int = Field(default=0)
    required_fields_total: int = Field(default=0)
    
    # 타입 정확도
    type_correctness: float = Field(..., description="타입 일치율 (0.0 ~ 1.0)")
    type_mismatches: List[str] = Field(default_factory=list, description="타입 불일치 필드 목록")
    
    # 성능 메트릭
    latency_ms: Optional[int] = Field(None, description="지연시간 (밀리초)")
    tokens_used: Optional[int] = Field(None, description="사용 토큰 수")
    cost_usd: Optional[float] = Field(None, description="비용 (USD)")
    
    # 종합 점수
    overall_score: float = Field(..., description="종합 점수 (0.0 ~ 1.0)")
    
    def get_quality_grade(self) -> str:
        """종합 점수를 등급으로 변환"""
        if self.overall_score >= 0.95:
            return "A+"
        elif self.overall_score >= 0.90:
            return "A"
        elif self.overall_score >= 0.80:
            return "B"
        elif self.overall_score >= 0.70:
            return "C"
        else:
            return "D"


class ExtractionEvaluationResult(BaseModel):
    """
    단일 golden item 에 대한 평가 결과.
    """
    evaluation_id: str = Field(..., description="평가 ID (UUID)")
    golden_item_id: str = Field(..., description="Golden item ID")
    extraction_id: str = Field(..., description="평가 대상 추출 ID")
    
    # 평가 결과
    metrics: ExtractionMetrics = Field(..., description="평가 메트릭")
    
    # 필드별 상세 정보
    field_evaluations: Dict[str, Dict[str, Any]] = Field(
        default_factory=dict, description="필드별 평가 상세"
    )
    
    # 메타
    evaluated_at: datetime = Field(default_factory=datetime.utcnow)
    notes: Optional[str] = Field(None)
    
    def passed_threshold(self, threshold: float = 0.80) -> bool:
        """임계값을 통과했는가"""
        return self.metrics.overall_score >= threshold


class ExtractionEvaluationSetResult(BaseModel):
    """
    Golden set 에 대한 배치 평가 결과.
    """
    evaluation_id: str = Field(..., description="평가 ID (UUID)")
    golden_set_id: str = Field(..., description="Golden set ID")
    
    # 개별 평가 결과
    item_results: List[ExtractionEvaluationResult] = Field(
        ..., description="각 item 의 평가 결과"
    )
    
    # 집계 통계
    total_items: int = Field(default=0)
    passed_items: int = Field(default=0)  # 임계값 통과 항목
    failed_items: int = Field(default=0)
    
    # 집계 메트릭
    average_accuracy: float = Field(default=0.0)
    average_span_accuracy: Optional[float] = Field(None)
    average_required_field_coverage: float = Field(default=0.0)
    average_type_correctness: float = Field(default=0.0)
    
    # 통계
    median_latency_ms: Optional[int] = Field(None)
    total_cost_usd: Optional[float] = Field(None)
    
    evaluated_at: datetime = Field(default_factory=datetime.utcnow)
    
    def calculate_pass_rate(self, threshold: float = 0.80) -> float:
        """통과율 계산"""
        if self.total_items == 0:
            return 0.0
        return self.passed_items / self.total_items


class QualityGateResult(BaseModel):
    """
    CI 품질 게이트 결과.
    """
    passed: bool = Field(..., description="게이트 통과 여부")
    thresholds: Dict[str, float] = Field(
        ..., description="적용된 임계값 (metric_name: threshold_value)"
    )
    violations: List[str] = Field(
        default_factory=list, description="위반된 조건 목록"
    )
    details: Dict[str, Any] = Field(default_factory=dict)
```

### 4.2 ExtractionEvaluator 클래스 구현

**파일**: `/mnt/mimir/src/services/extraction_evaluator.py` (신규)

```python
"""
추출 품질 평가 서비스.

Phase 7 Evaluator 를 상속하여 추출 특화 평가 로직 구현.
"""

from typing import Optional, List, Dict, Any, Tuple
from datetime import datetime
import statistics
import uuid

from src.core.models.evaluation import (
    GoldenExtractionItem, GoldenExtractionSet, ExtractionMetrics,
    ExtractionEvaluationResult, ExtractionEvaluationSetResult, QualityGateResult
)
from src.services.extraction_diff_calculator import (
    DiffCalculator, SpanBasedDiffCalculator
)


class ExtractionEvaluator:
    """
    추출 결과의 품질을 정량적으로 평가.
    """
    
    # 기본 품질 임계값
    DEFAULT_THRESHOLDS = {
        'field_accuracy': 0.85,
        'span_accuracy': 0.80,
        'required_field_coverage': 0.95,
        'type_correctness': 0.90,
        'overall_score': 0.80,
    }
    
    def __init__(self, custom_thresholds: Optional[Dict[str, float]] = None):
        """
        Args:
            custom_thresholds: 기본값을 override 할 임계값
        """
        self.thresholds = {**self.DEFAULT_THRESHOLDS}
        if custom_thresholds:
            self.thresholds.update(custom_thresholds)
    
    def evaluate_extraction(
        self,
        golden_item: GoldenExtractionItem,
        extraction_result: Dict[str, Any],
        extraction_spans: Optional[Dict[str, List[Tuple[int, int]]]] = None,
        latency_ms: Optional[int] = None,
        tokens_used: Optional[int] = None,
        cost_usd: Optional[float] = None,
    ) -> ExtractionEvaluationResult:
        """
        단일 golden item 에 대해 추출 결과를 평가.
        
        Args:
            golden_item: 기준 데이터
            extraction_result: 모델의 추출 결과 (dict)
            extraction_spans: 추출 결과의 span 정보 (field_name -> [(start, end), ...])
            latency_ms: 실행 시간
            tokens_used: 토큰 사용량
            cost_usd: 비용
        
        Returns:
            ExtractionEvaluationResult
        """
        
        # 1. 필드 정확도 계산
        field_accuracy, exact_match_cnt, fuzzy_match_cnt, mismatch_cnt = (
            self._calculate_field_accuracy(golden_item, extraction_result)
        )
        
        # 2. Span 정확도 계산 (있으면)
        span_accuracy = None
        span_iou_scores = None
        if extraction_spans and golden_item.expected_spans:
            span_accuracy, span_iou_scores = (
                self._calculate_span_accuracy(golden_item, extraction_spans)
            )
        
        # 3. 필수 필드 커버리지 계산
        required_coverage, required_extracted, required_total = (
            self._calculate_required_field_coverage(golden_item, extraction_result)
        )
        
        # 4. 타입 정확도 계산
        type_correctness, type_mismatches = (
            self._calculate_type_correctness(golden_item, extraction_result)
        )
        
        # 5. 종합 점수 계산
        overall_score = self._calculate_overall_score(
            field_accuracy=field_accuracy,
            span_accuracy=span_accuracy,
            required_coverage=required_coverage,
            type_correctness=type_correctness,
        )
        
        # 6. ExtractionMetrics 생성
        metrics = ExtractionMetrics(
            field_accuracy=field_accuracy,
            exact_match_count=exact_match_cnt,
            fuzzy_match_count=fuzzy_match_cnt,
            mismatch_count=mismatch_cnt,
            span_accuracy=span_accuracy,
            span_iou_scores=span_iou_scores,
            required_field_coverage=required_coverage,
            required_fields_extracted=required_extracted,
            required_fields_total=required_total,
            type_correctness=type_correctness,
            type_mismatches=type_mismatches,
            latency_ms=latency_ms,
            tokens_used=tokens_used,
            cost_usd=cost_usd,
            overall_score=overall_score,
        )
        
        # 7. 필드별 상세 평가
        field_evaluations = self._detailed_field_evaluation(
            golden_item, extraction_result, extraction_spans
        )
        
        # 8. 결과 생성
        evaluation_result = ExtractionEvaluationResult(
            evaluation_id=str(uuid.uuid4()),
            golden_item_id=golden_item.id,
            extraction_id="",  # 실제로는 extraction_id 전달받아야 함
            metrics=metrics,
            field_evaluations=field_evaluations,
        )
        
        return evaluation_result
    
    def evaluate_extraction_set(
        self,
        golden_set: GoldenExtractionSet,
        extraction_results: List[Dict[str, Any]],  # 각 item 에 대응하는 추출 결과
    ) -> ExtractionEvaluationSetResult:
        """
        Golden set 의 모든 item 을 배치 평가.
        
        Args:
            golden_set: Golden set
            extraction_results: 각 item 에 대응하는 추출 결과 리스트
        
        Returns:
            ExtractionEvaluationSetResult
        """
        
        if len(golden_set.items) != len(extraction_results):
            raise ValueError(
                f"Item count mismatch: {len(golden_set.items)} items "
                f"but {len(extraction_results)} results"
            )
        
        # 각 item 평가
        item_results = []
        for golden_item, extraction_result in zip(golden_set.items, extraction_results):
            result = self.evaluate_extraction(
                golden_item, extraction_result
            )
            item_results.append(result)
        
        # 집계 통계 계산
        passed_items = sum(
            1 for r in item_results 
            if r.passed_threshold(self.thresholds['overall_score'])
        )
        
        accuracies = [r.metrics.field_accuracy for r in item_results]
        span_accuracies = [
            r.metrics.span_accuracy for r in item_results 
            if r.metrics.span_accuracy is not None
        ]
        coverages = [r.metrics.required_field_coverage for r in item_results]
        type_correct = [r.metrics.type_correctness for r in item_results]
        latencies = [
            r.metrics.latency_ms for r in item_results 
            if r.metrics.latency_ms is not None
        ]
        costs = [
            r.metrics.cost_usd for r in item_results 
            if r.metrics.cost_usd is not None
        ]
        
        # 결과 생성
        result = ExtractionEvaluationSetResult(
            evaluation_id=str(uuid.uuid4()),
            golden_set_id=golden_set.id,
            item_results=item_results,
            total_items=len(golden_set.items),
            passed_items=passed_items,
            failed_items=len(golden_set.items) - passed_items,
            average_accuracy=statistics.mean(accuracies) if accuracies else 0.0,
            average_span_accuracy=(
                statistics.mean(span_accuracies) if span_accuracies else None
            ),
            average_required_field_coverage=statistics.mean(coverages) if coverages else 0.0,
            average_type_correctness=statistics.mean(type_correct) if type_correct else 0.0,
            median_latency_ms=(
                int(statistics.median(latencies)) if latencies else None
            ),
            total_cost_usd=sum(costs) if costs else None,
        )
        
        return result
    
    def _calculate_field_accuracy(
        self,
        golden_item: GoldenExtractionItem,
        extraction_result: Dict[str, Any],
    ) -> Tuple[float, int, int, int]:
        """
        필드 정확도 계산.
        
        Returns:
            (accuracy, exact_match_count, fuzzy_match_count, mismatch_count)
        """
        
        total_fields = len(golden_item.expected_fields)
        if total_fields == 0:
            return 0.0, 0, 0, 0
        
        exact_match = 0
        fuzzy_match = 0
        mismatch = 0
        
        for expected in golden_item.expected_fields:
            field_name = expected.field_name
            expected_value = expected.expected_value
            
            if field_name not in extraction_result:
                # 필드 미추출
                mismatch += 1
                continue
            
            extracted_value = extraction_result[field_name]
            
            # 타입 검증
            if type(extracted_value) != type(expected_value):
                mismatch += 1
                continue
            
            # 값 비교
            if isinstance(expected_value, str):
                is_match, similarity, match_type = (
                    DiffCalculator.compare_strings(
                        expected_value, extracted_value, fuzzy=True
                    )
                )
                if match_type == "exact_match":
                    exact_match += 1
                elif match_type == "fuzzy_match":
                    fuzzy_match += 1
                else:
                    mismatch += 1
            
            elif isinstance(expected_value, (int, float)):
                is_match, diff, match_type = (
                    DiffCalculator.compare_numbers(expected_value, extracted_value)
                )
                if is_match:
                    exact_match += 1
                else:
                    mismatch += 1
            
            else:
                # 다른 타입은 == 비교
                if extracted_value == expected_value:
                    exact_match += 1
                else:
                    mismatch += 1
        
        accuracy = (exact_match + fuzzy_match) / total_fields
        return accuracy, exact_match, fuzzy_match, mismatch
    
    def _calculate_span_accuracy(
        self,
        golden_item: GoldenExtractionItem,
        extraction_spans: Dict[str, List[Tuple[int, int]]],
    ) -> Tuple[float, List[float]]:
        """
        Span 정확도 계산 (IoU 기반).
        
        Returns:
            (average_iou, iou_scores_per_field)
        """
        
        if not golden_item.expected_spans:
            return 0.0, []
        
        iou_scores = []
        
        for expected_span in golden_item.expected_spans:
            field_name = expected_span.field_name
            expected_span_tuple = (expected_span.span_offset[0], expected_span.span_offset[1])
            
            if field_name not in extraction_spans:
                # Span 미추출
                iou_scores.append(0.0)
                continue
            
            extracted_span_list = extraction_spans[field_name]
            
            # 여러 span 이 있을 경우, 최고 IoU 선택
            best_iou = 0.0
            for extracted_span in extracted_span_list:
                iou = SpanBasedDiffCalculator.calculate_iou(
                    expected_span_tuple, extracted_span
                )
                best_iou = max(best_iou, iou)
            
            iou_scores.append(best_iou)
        
        avg_iou = sum(iou_scores) / len(iou_scores) if iou_scores else 0.0
        return avg_iou, iou_scores
    
    def _calculate_required_field_coverage(
        self,
        golden_item: GoldenExtractionItem,
        extraction_result: Dict[str, Any],
    ) -> Tuple[float, int, int]:
        """
        필수 필드 추출 커버리지.
        
        Returns:
            (coverage_rate, extracted_count, total_count)
        """
        
        required_fields = golden_item.get_required_fields()
        if not required_fields:
            return 1.0, 0, 0
        
        extracted_count = sum(
            1 for field_name in required_fields 
            if field_name in extraction_result and extraction_result[field_name] is not None
        )
        
        coverage = extracted_count / len(required_fields)
        return coverage, extracted_count, len(required_fields)
    
    def _calculate_type_correctness(
        self,
        golden_item: GoldenExtractionItem,
        extraction_result: Dict[str, Any],
    ) -> Tuple[float, List[str]]:
        """
        추출 값의 타입이 스키마와 일치하는 비율.
        
        Returns:
            (type_correctness, mismatched_field_names)
        """
        
        total_fields = len(golden_item.expected_fields)
        if total_fields == 0:
            return 1.0, []
        
        mismatches = []
        correct_count = 0
        
        for expected in golden_item.expected_fields:
            field_name = expected.field_name
            expected_type = expected.field_type
            
            if field_name not in extraction_result:
                mismatches.append(field_name)
                continue
            
            extracted_value = extraction_result[field_name]
            if extracted_value is None:
                mismatches.append(field_name)
                continue
            
            # 타입 매핑 (Python type -> schema type)
            extracted_type = self._map_python_type_to_schema(extracted_value)
            
            if extracted_type == expected_type.value:
                correct_count += 1
            else:
                mismatches.append(field_name)
        
        correctness = correct_count / total_fields
        return correctness, mismatches
    
    def _map_python_type_to_schema(self, value: Any) -> str:
        """Python 객체의 타입을 스키마 타입으로 매핑"""
        if isinstance(value, bool):
            return "boolean"
        elif isinstance(value, int):
            return "integer"
        elif isinstance(value, float):
            return "float"
        elif isinstance(value, str):
            return "string"
        elif isinstance(value, list):
            return "array"
        elif isinstance(value, dict):
            return "object"
        else:
            return "object"
    
    def _calculate_overall_score(
        self,
        field_accuracy: float,
        span_accuracy: Optional[float],
        required_coverage: float,
        type_correctness: float,
        weights: Optional[Dict[str, float]] = None,
    ) -> float:
        """
        종합 점수 계산 (가중 평균).
        
        기본 가중치:
        - field_accuracy: 0.4
        - span_accuracy: 0.25 (있으면)
        - required_coverage: 0.2
        - type_correctness: 0.15
        """
        
        if weights is None:
            weights = {
                'field_accuracy': 0.4,
                'span_accuracy': 0.25,
                'required_coverage': 0.2,
                'type_correctness': 0.15,
            }
        
        scores = {
            'field_accuracy': field_accuracy,
            'required_coverage': required_coverage,
            'type_correctness': type_correctness,
        }
        
        total_weight = 0.0
        weighted_sum = 0.0
        
        for metric, weight in weights.items():
            if metric == 'span_accuracy':
                if span_accuracy is not None:
                    weighted_sum += span_accuracy * weight
                    total_weight += weight
            else:
                weighted_sum += scores.get(metric, 0.0) * weight
                total_weight += weight
        
        if total_weight == 0:
            return 0.0
        
        return weighted_sum / total_weight
    
    def _detailed_field_evaluation(
        self,
        golden_item: GoldenExtractionItem,
        extraction_result: Dict[str, Any],
        extraction_spans: Optional[Dict[str, List[Tuple[int, int]]]] = None,
    ) -> Dict[str, Dict[str, Any]]:
        """
        필드별 상세 평가 정보 생성.
        """
        
        detailed = {}
        
        for expected in golden_item.expected_fields:
            field_name = expected.field_name
            expected_value = expected.expected_value
            extracted_value = extraction_result.get(field_name)
            
            field_detail = {
                'expected': expected_value,
                'extracted': extracted_value,
                'match': extracted_value == expected_value,
                'type_match': (
                    type(extracted_value) == type(expected_value) 
                    if extracted_value is not None else False
                ),
            }
            
            # Span 정보 추가
            if extraction_spans and field_name in extraction_spans:
                expected_span = golden_item.get_spans_for_field(field_name)
                if expected_span:
                    iou = SpanBasedDiffCalculator.calculate_iou(
                        (expected_span[0].span_offset[0], expected_span[0].span_offset[1]),
                        extraction_spans[field_name][0]
                    )
                    field_detail['span_iou'] = iou
            
            detailed[field_name] = field_detail
        
        return detailed
    
    def check_quality_gate(
        self,
        evaluation_result: ExtractionEvaluationSetResult,
        custom_thresholds: Optional[Dict[str, float]] = None,
    ) -> QualityGateResult:
        """
        평가 결과가 품질 게이트를 통과했는지 확인.
        """
        
        thresholds = {**self.DEFAULT_THRESHOLDS}
        if custom_thresholds:
            thresholds.update(custom_thresholds)
        
        violations = []
        
        # 각 임계값 확인
        if evaluation_result.average_accuracy < thresholds['field_accuracy']:
            violations.append(
                f"Field accuracy {evaluation_result.average_accuracy:.2%} "
                f"< {thresholds['field_accuracy']:.2%}"
            )
        
        if evaluation_result.average_span_accuracy is not None:
            if evaluation_result.average_span_accuracy < thresholds['span_accuracy']:
                violations.append(
                    f"Span accuracy {evaluation_result.average_span_accuracy:.2%} "
                    f"< {thresholds['span_accuracy']:.2%}"
                )
        
        if evaluation_result.average_required_field_coverage < thresholds['required_field_coverage']:
            violations.append(
                f"Required field coverage {evaluation_result.average_required_field_coverage:.2%} "
                f"< {thresholds['required_field_coverage']:.2%}"
            )
        
        if evaluation_result.average_type_correctness < thresholds['type_correctness']:
            violations.append(
                f"Type correctness {evaluation_result.average_type_correctness:.2%} "
                f"< {thresholds['type_correctness']:.2%}"
            )
        
        # Pass rate 확인
        pass_rate = evaluation_result.calculate_pass_rate(thresholds['overall_score'])
        if pass_rate < 0.80:  # 80% 이상 통과 필요
            violations.append(
                f"Pass rate {pass_rate:.2%} < 80%"
            )
        
        passed = len(violations) == 0
        
        return QualityGateResult(
            passed=passed,
            thresholds=thresholds,
            violations=violations,
            details={
                'average_accuracy': evaluation_result.average_accuracy,
                'average_span_accuracy': evaluation_result.average_span_accuracy,
                'average_required_field_coverage': evaluation_result.average_required_field_coverage,
                'average_type_correctness': evaluation_result.average_type_correctness,
                'pass_rate': pass_rate,
            }
        )
```

### 4.3 평가 API 엔드포인트

**파일**: `/mnt/mimir/src/api/routes/extraction_evaluation.py` (신규)

```python
"""
추출 품질 평가 API.

S2 원칙 ⑤ ⑥ 적용.
"""

from fastapi import APIRouter, Depends, HTTPException, Query
from typing import Optional, List, Dict, Any
import uuid

from src.core.models.evaluation import (
    GoldenExtractionSet, ExtractionEvaluationSetResult, QualityGateResult
)
from src.services.extraction_evaluator import ExtractionEvaluator
from src.api.dependencies import (
    get_current_user, get_db, check_scope_access, audit_log_action
)

router = APIRouter(prefix="/api/v1/extraction-evaluations", tags=["extraction-evaluation"])


@router.post("/run")
async def run_extraction_evaluation(
    golden_set_id: str,
    extraction_ids: List[str],  # 각 golden item 에 대응하는 extraction ID
    current_user: Dict = Depends(get_current_user),
    db = Depends(get_db),
) -> ExtractionEvaluationSetResult:
    """
    Golden set 을 사용한 추출 품질 평가 실행.
    
    POST /api/v1/extraction-evaluations/run
    
    Args:
        golden_set_id: Golden set ID
        extraction_ids: 각 item 에 대응하는 추출 ID 리스트
    
    Returns:
        ExtractionEvaluationSetResult
    """
    
    # S2 원칙 ⑥: ACL 확인
    scope_check = await check_scope_access(
        user_id=current_user['id'],
        resource_type='golden_set',
        resource_id=golden_set_id,
        action='evaluate'
    )
    
    if not scope_check['allowed']:
        await audit_log_action(
            action='evaluation_denied',
            actor_type='user',
            actor_id=current_user['id'],
            resource_id=golden_set_id,
            reason=scope_check['reason'],
            db=db
        )
        raise HTTPException(status_code=403, detail=scope_check['reason'])
    
    # Golden set 조회
    # golden_set = db.query(GoldenSetORM).filter_by(id=golden_set_id).first()
    # if not golden_set:
    #     raise HTTPException(status_code=404, detail="Golden set not found")
    
    # 추출 결과 조회
    # extraction_results = [
    #     get_extraction_result(ext_id) for ext_id in extraction_ids
    # ]
    
    # 평가 실행
    evaluator = ExtractionEvaluator()
    # eval_result = evaluator.evaluate_extraction_set(golden_set, extraction_results)
    
    # 평가 결과 저장
    # save_evaluation_result(eval_result, db)
    
    # S2 원칙 ⑤: 감사 로깅
    await audit_log_action(
        action='extraction_evaluation_run',
        actor_type='user',
        actor_id=current_user['id'],
        resource_id=golden_set_id,
        details={'evaluation_id': 'eval_id', 'item_count': len(extraction_ids)},
        db=db
    )
    
    # return eval_result
    pass


@router.get("/{eval_id}")
async def get_evaluation_result(
    eval_id: str,
    current_user: Dict = Depends(get_current_user),
    db = Depends(get_db),
) -> ExtractionEvaluationSetResult:
    """
    평가 결과 조회.
    
    GET /api/v1/extraction-evaluations/{eval_id}
    """
    
    # 평가 결과 조회
    # evaluation = db.query(ExtractionEvaluationORM).filter_by(id=eval_id).first()
    # if not evaluation:
    #     raise HTTPException(status_code=404)
    
    # ACL 확인
    # golden_set_id = evaluation.golden_set_id
    # scope_check = await check_scope_access(...)
    
    await audit_log_action(
        action='evaluation_viewed',
        actor_type='user',
        actor_id=current_user['id'],
        resource_id=eval_id,
        db=db
    )
    
    # return convert_orm_to_model(evaluation)
    pass


@router.get("/{eval_id}/compare")
async def compare_evaluations(
    eval_id_1: str = Query(...),
    eval_id_2: str = Query(...),
    current_user: Dict = Depends(get_current_user),
    db = Depends(get_db),
) -> Dict[str, Any]:
    """
    두 평가 결과 비교 (A/B test).
    
    GET /api/v1/extraction-evaluations/{eval_id}/compare?eval_id_1=X&eval_id_2=Y
    
    Returns:
        {
            "comparison": {
                "accuracy_improvement": 0.05,
                "span_accuracy_improvement": 0.03,
                "winner": "eval_1",
                "metrics_comparison": {...}
            }
        }
    """
    # 두 평가 결과 조회
    # eval_1 = get_evaluation(eval_id_1)
    # eval_2 = get_evaluation(eval_id_2)
    
    # 비교 분석
    # comparison = analyze_comparison(eval_1, eval_2)
    
    await audit_log_action(
        action='evaluations_compared',
        actor_type='user',
        actor_id=current_user['id'],
        resource_id=f"{eval_id_1}:{eval_id_2}",
        db=db
    )
    
    # return comparison
    pass


@router.post("/quality-gate-check")
async def check_quality_gate(
    eval_id: str,
    current_user: Dict = Depends(get_current_user),
    db = Depends(get_db),
) -> QualityGateResult:
    """
    품질 게이트 확인.
    
    POST /api/v1/extraction-evaluations/quality-gate-check
    """
    
    # 평가 결과 조회
    # evaluation = get_evaluation(eval_id)
    
    # 게이트 확인
    evaluator = ExtractionEvaluator()
    # gate_result = evaluator.check_quality_gate(evaluation)
    
    await audit_log_action(
        action='quality_gate_checked',
        actor_type='user',
        actor_id=current_user['id'],
        resource_id=eval_id,
        details={'gate_passed': True},  # gate_result.passed
        db=db
    )
    
    # return gate_result
    pass
```

### 4.4 품질 보고서 생성

**파일**: `/mnt/mimir/src/services/evaluation_report_generator.py` (신규)

```python
"""
평가 결과 보고서 생성.

Jinja2 템플릿 기반 마크다운 보고서 생성.
"""

from jinja2 import Template
from typing import Optional
from datetime import datetime

from src.core.models.evaluation import ExtractionEvaluationSetResult

REPORT_TEMPLATE = """
# 추출 품질 평가 보고서

**평가 ID**: {{ eval_id }}  
**평가 시간**: {{ eval_time }}  
**Golden Set**: {{ golden_set_name }}

---

## 1. 종합 결과

| 지표 | 값 |
|------|-----|
| 평가 항목 수 | {{ total_items }} |
| 통과 항목 | {{ passed_items }} ({{ pass_rate }}%) |
| 실패 항목 | {{ failed_items }} |

---

## 2. 메트릭 요약

### 필드 정확도
- **평균**: {{ avg_accuracy }}%
- **목표**: >= 85%
- **상태**: {% if avg_accuracy >= 85 %}✓ 통과{% else %}✗ 미달성{% endif %}

### Span 정확도
- **평균 IoU**: {{ avg_span_accuracy or 'N/A' }}%
- **목표**: >= 80%
- **상태**: {% if avg_span_accuracy >= 80 %}✓ 통과{% else %}✗ 미달성{% endif %}

### 필수 필드 커버리지
- **평균**: {{ avg_required_field_coverage }}%
- **목표**: >= 95%
- **상태**: {% if avg_required_field_coverage >= 95 %}✓ 통과{% else %}✗ 미달성{% endif %}

### 타입 정확도
- **평균**: {{ avg_type_correctness }}%
- **목표**: >= 90%
- **상태**: {% if avg_type_correctness >= 90 %}✓ 통과{% else %}✗ 미달성{% endif %}

---

## 3. 성능 메트릭

| 메트릭 | 값 |
|--------|-----|
| 중앙값 지연시간 | {{ median_latency_ms }}ms |
| 총 토큰 사용량 | {{ total_tokens }} |
| 총 비용 | ${{ total_cost }} |

---

## 4. 상세 분석

### 항목별 점수
{% for item in item_results %}
#### {{ item.golden_item_id }}
- 필드 정확도: {{ "%.2f"|format(item.metrics.field_accuracy) }}
- 필드 일치: {{ item.metrics.exact_match_count }} 정확, {{ item.metrics.fuzzy_match_count }} Fuzzy, {{ item.metrics.mismatch_count }} 불일치
- 필수 필드 커버리지: {{ "%.2f"|format(item.metrics.required_field_coverage) }}
- 타입 정확도: {{ "%.2f"|format(item.metrics.type_correctness) }}
- **종합 점수**: {{ "%.2f"|format(item.metrics.overall_score) }}
- **등급**: {{ item.metrics.get_quality_grade() }}

{% endfor %}

---

## 5. 권장사항

{% if avg_accuracy < 85 %}
- **필드 정확도 개선 필요**: 추출 프롬프트 재검토 및 모델 선택 재평가
{% endif %}

{% if avg_span_accuracy < 80 %}
- **Span 정확도 개선 필요**: LLM 이 위치 정보를 정확히 제공하지 못함. 프롬프트에 위치 명시 지시문 강화 필요
{% endif %}

{% if avg_required_field_coverage < 95 %}
- **필수 필드 미추출**: 스키마 또는 프롬프트 재검토. 필드 필수성 조정 검토
{% endif %}

{% if avg_type_correctness < 90 %}
- **타입 불일치**: 스키마 정의 재검토 또는 프롬프트에 타입 힌트 추가
{% endif %}

---

## 6. 질문 및 피드백

이 보고서에 대한 질문이나 개선 제안은 팀 리더에게 전달하세요.

---

*Report generated on {{ report_generated_at }}*
"""


class EvaluationReportGenerator:
    """
    평가 결과 보고서 생성.
    """
    
    @staticmethod
    def generate_markdown_report(
        evaluation_result: ExtractionEvaluationSetResult,
        golden_set_name: str,
    ) -> str:
        """
        평가 결과를 마크다운 보고서로 생성.
        
        Args:
            evaluation_result: 평가 결과
            golden_set_name: Golden set 이름
        
        Returns:
            마크다운 텍스트
        """
        
        template = Template(REPORT_TEMPLATE)
        
        # 데이터 준비
        context = {
            'eval_id': evaluation_result.evaluation_id,
            'eval_time': evaluation_result.evaluated_at.isoformat(),
            'golden_set_name': golden_set_name,
            'total_items': evaluation_result.total_items,
            'passed_items': evaluation_result.passed_items,
            'failed_items': evaluation_result.failed_items,
            'pass_rate': round(
                evaluation_result.calculate_pass_rate() * 100, 2
            ),
            'avg_accuracy': f"{evaluation_result.average_accuracy * 100:.2f}",
            'avg_span_accuracy': (
                f"{evaluation_result.average_span_accuracy * 100:.2f}"
                if evaluation_result.average_span_accuracy else None
            ),
            'avg_required_field_coverage': (
                f"{evaluation_result.average_required_field_coverage * 100:.2f}"
            ),
            'avg_type_correctness': f"{evaluation_result.average_type_correctness * 100:.2f}",
            'median_latency_ms': evaluation_result.median_latency_ms,
            'total_tokens': sum(
                r.metrics.tokens_used or 0 for r in evaluation_result.item_results
            ),
            'total_cost': f"{evaluation_result.total_cost_usd or 0:.4f}",
            'item_results': evaluation_result.item_results,
            'report_generated_at': datetime.utcnow().isoformat(),
        }
        
        report = template.render(context)
        return report
    
    @staticmethod
    def generate_html_report(
        markdown_report: str
    ) -> str:
        """
        마크다운을 HTML 로 변환 (선택사항).
        
        실제 구현에서는 markdown2 또는 python-markdown 라이브러리 사용.
        """
        # import markdown
        # html = markdown.markdown(markdown_report, extensions=['tables'])
        # return html
        pass
```

### 4.5 GoldenExtractionSet ORM 및 마이그레이션

**파일**: `/mnt/mimir/src/core/db/models/evaluation.py` (신규)

```python
from sqlalchemy import Column, String, DateTime, JSON, Boolean, Index
from sqlalchemy.orm import relationship
from datetime import datetime
import uuid

class GoldenExtractionSetORM(Base):
    """Golden extraction set 저장"""
    __tablename__ = 'golden_extraction_sets'
    
    id = Column(String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    name = Column(String(255), nullable=False)
    description = Column(String(1000), nullable=True)
    document_type = Column(String(100), nullable=False)
    extraction_schema_id = Column(String(36), nullable=False, index=True)
    version = Column(String(50), default="1.0.0")
    
    created_by = Column(String(36), nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # items 은 별도 테이블에서 FK 로 관리
    items = relationship("GoldenExtractionItemORM", back_populates="golden_set")


class GoldenExtractionItemORM(Base):
    """Individual golden item"""
    __tablename__ = 'golden_extraction_items'
    
    id = Column(String(36), primary_key=True)
    golden_set_id = Column(String(36), ForeignKey('golden_extraction_sets.id'))
    
    document_id = Column(String(36), nullable=False)
    document_version = Column(String(50), nullable=False)
    document_type = Column(String(100), nullable=False)
    extraction_schema_id = Column(String(36), nullable=False)
    
    expected_fields = Column(JSON, nullable=False)  # List[ExpectedField]
    expected_spans = Column(JSON, default=list)    # List[ExpectedSpan]
    
    created_by = Column(String(36), nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    notes = Column(String(1000), nullable=True)
    
    golden_set = relationship("GoldenExtractionSetORM", back_populates="items")


class ExtractionEvaluationORM(Base):
    """평가 결과 저장"""
    __tablename__ = 'extraction_evaluations'
    
    id = Column(String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    golden_set_id = Column(String(36), ForeignKey('golden_extraction_sets.id'))
    
    # 집계 메트릭
    average_accuracy = Column(Float, nullable=False)
    average_span_accuracy = Column(Float, nullable=True)
    average_required_field_coverage = Column(Float, nullable=False)
    average_type_correctness = Column(Float, nullable=False)
    
    # 통계
    total_items = Column(Integer, nullable=False)
    passed_items = Column(Integer, nullable=False)
    failed_items = Column(Integer, nullable=False)
    
    # 상세 결과
    item_results = Column(JSON, nullable=False)  # serialized results
    
    evaluated_at = Column(DateTime, default=datetime.utcnow)
    created_at = Column(DateTime, default=datetime.utcnow)
```

### 4.6 마이그레이션 스크립트

**파일**: `/mnt/mimir/src/core/db/migrations/versions/010_create_evaluation_tables.py`

```python
"""Create evaluation-related tables.

Revision ID: 010
Revises: 009
"""
from alembic import op
import sqlalchemy as sa

revision = '010'
down_revision = '009'

def upgrade():
    # golden_extraction_sets
    op.create_table(
        'golden_extraction_sets',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('name', sa.String(255), nullable=False),
        sa.Column('description', sa.String(1000), nullable=True),
        sa.Column('document_type', sa.String(100), nullable=False),
        sa.Column('extraction_schema_id', sa.String(36), nullable=False),
        sa.Column('version', sa.String(50), nullable=False, server_default='1.0.0'),
        sa.Column('created_by', sa.String(36), nullable=False),
        sa.Column('created_at', sa.DateTime, nullable=False, server_default=sa.func.now()),
        sa.PrimaryKeyConstraint('id'),
    )
    
    # golden_extraction_items
    op.create_table(
        'golden_extraction_items',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('golden_set_id', sa.String(36), ForeignKey('golden_extraction_sets.id')),
        sa.Column('document_id', sa.String(36), nullable=False),
        sa.Column('document_version', sa.String(50), nullable=False),
        sa.Column('document_type', sa.String(100), nullable=False),
        sa.Column('extraction_schema_id', sa.String(36), nullable=False),
        sa.Column('expected_fields', sa.JSON, nullable=False),
        sa.Column('expected_spans', sa.JSON, nullable=False, server_default='[]'),
        sa.Column('created_by', sa.String(36), nullable=False),
        sa.Column('created_at', sa.DateTime, nullable=False, server_default=sa.func.now()),
        sa.Column('notes', sa.String(1000), nullable=True),
        sa.PrimaryKeyConstraint('id'),
    )
    
    # extraction_evaluations
    op.create_table(
        'extraction_evaluations',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('golden_set_id', sa.String(36), ForeignKey('golden_extraction_sets.id')),
        sa.Column('average_accuracy', sa.Float, nullable=False),
        sa.Column('average_span_accuracy', sa.Float, nullable=True),
        sa.Column('average_required_field_coverage', sa.Float, nullable=False),
        sa.Column('average_type_correctness', sa.Float, nullable=False),
        sa.Column('total_items', sa.Integer, nullable=False),
        sa.Column('passed_items', sa.Integer, nullable=False),
        sa.Column('failed_items', sa.Integer, nullable=False),
        sa.Column('item_results', sa.JSON, nullable=False),
        sa.Column('evaluated_at', sa.DateTime, nullable=False, server_default=sa.func.now()),
        sa.Column('created_at', sa.DateTime, nullable=False, server_default=sa.func.now()),
        sa.PrimaryKeyConstraint('id'),
    )

def downgrade():
    op.drop_table('extraction_evaluations')
    op.drop_table('golden_extraction_items')
    op.drop_table('golden_extraction_sets')
```

### 4.7 단위 테스트

**파일**: `/mnt/mimir/tests/test_extraction_evaluator.py` (신규)

```python
"""
ExtractionEvaluator 단위 테스트.
"""

import pytest
from src.services.extraction_evaluator import ExtractionEvaluator
from src.core.models.evaluation import (
    GoldenExtractionItem, ExpectedField, FieldTypeEnum, GoldenExtractionSet
)


class TestExtractionEvaluator:
    """ExtractionEvaluator 테스트"""
    
    @pytest.fixture
    def evaluator(self):
        return ExtractionEvaluator()
    
    @pytest.fixture
    def golden_item(self):
        """기본 golden item fixture"""
        return GoldenExtractionItem(
            id="golden_001",
            document_id="doc_001",
            document_version="1.0",
            document_type="contract",
            expected_fields=[
                ExpectedField(
                    field_name="company_name",
                    expected_value="Acme Corporation",
                    field_type=FieldTypeEnum.STRING,
                    required=True,
                ),
                ExpectedField(
                    field_name="founded_year",
                    expected_value=2010,
                    field_type=FieldTypeEnum.INTEGER,
                    required=True,
                ),
            ],
            extraction_schema_id="schema_001",
            created_by="user_001",
        )
    
    def test_field_accuracy_exact_match(self, evaluator, golden_item):
        """필드 정확도 - 정확 일치"""
        extraction_result = {
            "company_name": "Acme Corporation",
            "founded_year": 2010,
        }
        
        accuracy, exact, fuzzy, mismatch = evaluator._calculate_field_accuracy(
            golden_item, extraction_result
        )
        
        assert accuracy == 1.0
        assert exact == 2
        assert fuzzy == 0
        assert mismatch == 0
    
    def test_field_accuracy_partial_match(self, evaluator, golden_item):
        """필드 정확도 - 부분 일치"""
        extraction_result = {
            "company_name": "Acme Corp",  # Fuzzy match
            "founded_year": 2010,
        }
        
        accuracy, exact, fuzzy, mismatch = evaluator._calculate_field_accuracy(
            golden_item, extraction_result
        )
        
        assert accuracy == 1.0  # fuzzy 도 일치로 카운트
        assert fuzzy >= 1
    
    def test_field_accuracy_missing_field(self, evaluator, golden_item):
        """필드 정확도 - 필드 누락"""
        extraction_result = {
            "company_name": "Acme Corporation",
            # founded_year 누락
        }
        
        accuracy, exact, fuzzy, mismatch = evaluator._calculate_field_accuracy(
            golden_item, extraction_result
        )
        
        assert accuracy < 1.0
        assert mismatch >= 1
    
    def test_required_field_coverage_all_extracted(self, evaluator, golden_item):
        """필수 필드 커버리지 - 모두 추출"""
        extraction_result = {
            "company_name": "Acme Corporation",
            "founded_year": 2010,
        }
        
        coverage, extracted, total = evaluator._calculate_required_field_coverage(
            golden_item, extraction_result
        )
        
        assert coverage == 1.0
        assert extracted == 2
        assert total == 2
    
    def test_required_field_coverage_partial(self, evaluator, golden_item):
        """필수 필드 커버리지 - 부분 추출"""
        extraction_result = {
            "company_name": "Acme Corporation",
            # founded_year 누락
        }
        
        coverage, extracted, total = evaluator._calculate_required_field_coverage(
            golden_item, extraction_result
        )
        
        assert coverage == 0.5
        assert extracted == 1
        assert total == 2
    
    def test_type_correctness_match(self, evaluator, golden_item):
        """타입 정확도 - 일치"""
        extraction_result = {
            "company_name": "Acme Corporation",
            "founded_year": 2010,
        }
        
        correctness, mismatches = evaluator._calculate_type_correctness(
            golden_item, extraction_result
        )
        
        assert correctness == 1.0
        assert len(mismatches) == 0
    
    def test_type_correctness_mismatch(self, evaluator, golden_item):
        """타입 정확도 - 불일치"""
        extraction_result = {
            "company_name": "Acme Corporation",
            "founded_year": "2010",  # String 이 아닌 Integer 예상
        }
        
        correctness, mismatches = evaluator._calculate_type_correctness(
            golden_item, extraction_result
        )
        
        assert correctness < 1.0
        assert "founded_year" in mismatches
    
    def test_overall_score_calculation(self, evaluator):
        """종합 점수 계산"""
        overall = evaluator._calculate_overall_score(
            field_accuracy=0.95,
            span_accuracy=0.85,
            required_coverage=0.95,
            type_correctness=0.90,
        )
        
        assert 0.0 <= overall <= 1.0
        assert overall > 0.8  # 높은 각 메트릭이므로 전체 점수도 높아야 함
    
    def test_quality_gate_passed(self, evaluator):
        """품질 게이트 - 통과"""
        # Mock evaluation result 로 테스트
        pass
    
    def test_quality_gate_failed(self, evaluator):
        """품질 게이트 - 실패"""
        pass


class TestIntegrationEvaluation:
    """통합 평가 테스트"""
    
    @pytest.mark.asyncio
    async def test_full_evaluation_workflow(self):
        """전체 평가 워크플로우"""
        # Golden set 생성
        # 추출 결과 준비
        # 평가 실행
        # 게이트 확인
        pass
```

---

## 5. 산출물

### 5.1 코드 파일

| 경로 | 설명 |
|------|------|
| `/mnt/mimir/src/core/models/evaluation.py` | 평가 관련 Pydantic 모델 |
| `/mnt/mimir/src/core/db/models/evaluation.py` | 평가 관련 SQLAlchemy ORM |
| `/mnt/mimir/src/core/db/migrations/versions/010_create_evaluation_tables.py` | Alembic 마이그레이션 |
| `/mnt/mimir/src/services/extraction_evaluator.py` | ExtractionEvaluator 구현 |
| `/mnt/mimir/src/services/evaluation_report_generator.py` | 보고서 생성 서비스 |
| `/mnt/mimir/src/api/routes/extraction_evaluation.py` | 평가 API 엔드포인트 |
| `/mnt/mimir/tests/test_extraction_evaluator.py` | 평가 단위 테스트 |

### 5.2 산출물 상세

| 파일 | 설명 | 크기 |
|------|------|------|
| Golden Set 관리 API | CRUD endpoint | 500 LOC |
| 평가 실행 API | run, get, compare | 800 LOC |
| ExtractionEvaluator | 메트릭 계산 | 1200 LOC |
| 평가 보고서 템플릿 | Jinja2 마크다운 | 200 LOC |
| 테스트 | unit + integration | 600 LOC |

---

## 6. 완료 기준

1. ✓ GoldenExtractionItem Pydantic 모델
2. ✓ GoldenExtractionSet 모델
3. ✓ ExpectedField, ExpectedSpan 모델
4. ✓ ExtractionMetrics 메트릭 정의
5. ✓ ExtractionEvaluationResult, ExtractionEvaluationSetResult 모델
6. ✓ QualityGateResult 모델
7. ✓ ExtractionEvaluator 클래스 구현
8. ✓ 필드 정확도 계산 (정확/fuzzy/정확도)
9. ✓ Span 정확도 계산 (IoU)
10. ✓ 필수 필드 커버리지 계산
11. ✓ 타입 정확도 계산
12. ✓ 종합 점수 계산 (가중 평균)
13. ✓ 품질 게이트 구현
14. ✓ GoldenExtractionSetORM, ItemORM, EvaluationORM
15. ✓ Alembic 마이그레이션
16. ✓ POST /api/v1/extraction-evaluations/run
17. ✓ GET /api/v1/extraction-evaluations/{eval_id}
18. ✓ GET /api/v1/extraction-evaluations/{eval_id}/compare
19. ✓ 보고서 생성 (Jinja2 마크다운)
20. ✓ CI 게이트 통합 (임계값 체크)
21. ✓ S2 원칙 ⑤ ⑥ 적용
22. ✓ 단위 테스트 최소 15개
23. ✓ 통합 테스트

---

## 7. 작업 지침

(지침 7-1 ~ 7-10 생략 - task8-8, task8-9 와 동일한 패턴)

---

**끝.**
