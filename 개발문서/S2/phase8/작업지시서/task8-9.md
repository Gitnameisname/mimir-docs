# FG8.3 작업지시서 - task8-9
# ExtractionRecord + 재현성 검증 API

**버전**: 1.0  
**작성일**: 2026-04-17  
**FG**: FG8.3 — 추출 결과의 검증 가능 계약  
**담당**: Phase 8 팀

---

## 1. 작업 목적

추출 작업의 모든 조건(모델, 프롬프트, 온도, 문서 버전 등)을 기록하고, 동일 조건으로 재추출하여 결과를 비교 검증할 수 있도록 설계한다. 이를 통해:

- 추출 재현성(reproducibility) 보장
- 모델/프롬프트 변경 영향도 측정
- 추출 감사(audit) 추적
- 테스트 및 QA 자동화 기반 제공

---

## 2. 작업 범위

### 포함 범위

1. **ExtractionRecord 데이터 모델**
   - 추출 조건 전체 기록: document_id, document_version, document_content_hash
   - 스키마 정보: extraction_schema_id, extraction_schema_version
   - 모델 정보: extraction_model, model_version, extraction_prompt_version
   - 실행 파라미터: extraction_mode (deterministic/probabilistic), temperature, seed
   - 결과 저장: extracted_result (JSON), extracted_timestamp
   - Pydantic 정의 + SQLAlchemy ORM

2. **ExtractionRecord 저장 및 조회**
   - DB 마이그레이션 (Alembic)
   - extraction_records 테이블 생성
   - 인덱스: (document_id, extraction_id), (schema_id, extraction_id)

3. **재현성 검증 API**
   - POST /api/v1/extractions/{extraction_id}/verify
   - 입력: 재검증 파라미터 (기존 파라미터 override 가능)
   - 처리: 동일 조건으로 재추출, 결과 비교
   - 응답: match_status (identical/partial/mismatch), field_level diff, diff_details
   - 결과를 verification_record 로 저장

4. **감사 API**
   - GET /api/v1/extractions/{extraction_id}/audit
   - 응답: 전체 추출 provenance, 조건, 결과 히스토리
   - 선택적 필터: date range, field_name, status

5. **결정론적 모드(Deterministic Mode) 강제화**
   - temperature = 0.0 고정
   - seed 파라미터 지원 (모델이 지원할 경우)
   - 동일 seed 입력 시 동일 결과 보장

6. **차이점 계산(Diff Logic)**
   - Field-by-field 비교
   - 문자열: 정확 일치 / Levenshtein distance (fuzzy)
   - 숫자: floating point tolerance (±epsilon)
   - 배열/객체: deep diff
   - Span 기반 비교: IoU(Intersection over Union)

7. **Scope Profile ACL**
   - POST /verify, GET /audit 모두에 ACL 적용
   - actor_type (user / agent) 감사 로깅
   - 사용자는 자신의 extraction 만 검증 가능

8. **검증 API 결과 저장**
   - verification_results 테이블 생성
   - 검증 기록 조회 가능
   - 히스토리 추적

### 제외 범위

- 외부 LLM 서비스 통합 (Phase 5 기존 호출 방식 재사용)
- 실시간 스트리밍 응답 처리 (단일 재추출만)
- 자동 재추출 스케줄링 (수동 API 호출만)
- 추출 결과 자동 수정 (검증만, 수정은 수동)

---

## 3. 선행 조건

1. task8-8 완료: SourceSpan, ExtractedFieldWithAttribution 모델 존재
2. Phase 5 완료: ExtractionCandidate, ExtractionSchema ORM 모델
3. Phase 2 완료: Document, DocumentVersion ORM 모델
4. 기존 LLM 호출 서비스 가능 (Phase 5 의 extraction service)
5. 데이터베이스 마이그레이션 환경 구성
6. 테스트 DB 준비

---

## 4. 주요 작업 항목

### 4.1 ExtractionRecord Pydantic 모델

**파일**: `/mnt/mimir/src/core/models/extraction.py` (기존 파일에 추가)

```python
from pydantic import BaseModel, Field, field_validator
from typing import Optional, Dict, Any
from datetime import datetime
from enum import Enum

class ExtractionMode(str, Enum):
    """추출 실행 모드"""
    DETERMINISTIC = "deterministic"  # 재현 가능 (temperature=0.0)
    PROBABILISTIC = "probabilistic"  # 확률적 (temperature > 0)


class ExtractionRecord(BaseModel):
    """
    추출 작업의 완전한 조건 및 결과 기록.
    
    동일 조건으로 재추출할 때 모든 파라미터를 복원하기 위한 스냅샷.
    """
    extraction_id: str = Field(..., description="추출 작업 ID (UUID)")
    document_id: str = Field(..., description="원본 문서 ID")
    document_version: str = Field(..., description="문서 버전")
    document_content_hash: str = Field(..., description="문서 내용의 SHA256 해시")
    
    # 추출 스키마 정보
    extraction_schema_id: str = Field(..., description="추출 스키마 ID")
    extraction_schema_version: str = Field(..., description="스키마 버전 (예: 1.0.0)")
    
    # LLM 모델 정보
    extraction_model: str = Field(..., description="사용 모델 (예: gpt-4, claude-3-opus)")
    model_version: str = Field(..., description="모델 버전 (예: gpt-4-0125-preview)")
    
    # 프롬프트 정보
    extraction_prompt_version: str = Field(..., description="프롬프트 템플릿 버전")
    extraction_prompt_hash: Optional[str] = Field(
        None, description="실제 사용된 프롬프트의 SHA256 해시"
    )
    
    # 실행 파라미터
    extraction_mode: ExtractionMode = Field(
        ..., description="추출 모드: deterministic 또는 probabilistic"
    )
    temperature: float = Field(
        ..., description="LLM temperature (0.0 ~ 2.0)"
    )
    seed: Optional[int] = Field(
        None, description="재현성을 위한 seed (모델이 지원할 경우)"
    )
    max_tokens: Optional[int] = Field(None, description="최대 토큰 수")
    
    # 추출 결과
    extracted_result: Dict[str, Any] = Field(
        ..., description="추출된 JSON 결과"
    )
    extracted_timestamp: datetime = Field(
        default_factory=datetime.utcnow, description="추출 완료 시간"
    )
    
    # 추가 메타데이터
    execution_time_ms: Optional[int] = Field(None, description="실행 시간 (밀리초)")
    tokens_used: Optional[int] = Field(None, description="사용한 토큰 수")
    cost_usd: Optional[float] = Field(None, description="추정 비용 (USD)")
    
    # 추출 성공 여부
    is_successful: bool = Field(default=True, description="추출 성공 여부")
    error_message: Optional[str] = Field(None, description="실패 시 에러 메시지")
    
    @field_validator('temperature')
    @classmethod
    def validate_temperature(cls, v: float) -> float:
        """Temperature 범위 검증"""
        if not (0.0 <= v <= 2.0):
            raise ValueError("temperature must be between 0.0 and 2.0")
        return v
    
    @field_validator('extraction_mode')
    @classmethod
    def validate_mode_and_temperature(cls, v: ExtractionMode) -> ExtractionMode:
        """추출 모드 검증 (나중에 temperature 와 함께 검증)"""
        return v
    
    def ensure_deterministic_reproducibility(self):
        """재현성 조건 검증"""
        if self.extraction_mode == ExtractionMode.DETERMINISTIC:
            if self.temperature != 0.0:
                raise ValueError(
                    f"Deterministic mode requires temperature=0.0, got {self.temperature}"
                )
        return True
    
    def get_reproducibility_key(self) -> str:
        """재현성을 위한 unique key 생성"""
        import hashlib
        import json
        
        key_data = {
            "document_id": self.document_id,
            "document_version": self.document_version,
            "schema_id": self.extraction_schema_id,
            "model": self.extraction_model,
            "model_version": self.model_version,
            "prompt_version": self.extraction_prompt_version,
            "temperature": self.temperature,
            "seed": self.seed
        }
        
        key_json = json.dumps(key_data, sort_keys=True)
        return hashlib.sha256(key_json.encode()).hexdigest()
    
    class Config:
        json_schema_extra = {
            "example": {
                "extraction_id": "550e8400-e29b-41d4-a716-446655440000",
                "document_id": "550e8400-e29b-41d4-a716-446655440001",
                "document_version": "1.2.3",
                "document_content_hash": "abc123...",
                "extraction_schema_id": "schema_001",
                "extraction_schema_version": "2.0.0",
                "extraction_model": "gpt-4",
                "model_version": "gpt-4-0125-preview",
                "extraction_prompt_version": "1.0.0",
                "extraction_mode": "deterministic",
                "temperature": 0.0,
                "seed": 12345,
                "extracted_result": {
                    "company_name": "Acme Corp",
                    "founded_year": 2010
                },
                "extracted_timestamp": "2026-04-17T10:30:00Z",
                "execution_time_ms": 2500,
                "tokens_used": 450,
                "cost_usd": 0.012
            }
        }


class DiffDetail(BaseModel):
    """필드별 차이 상세 정보"""
    field_name: str
    original_value: Any
    reproduced_value: Any
    diff_type: str  # "exact_match", "fuzzy_match", "mismatch", "span_mismatch"
    similarity: Optional[float] = Field(
        None, description="유사도 (0.0 ~ 1.0), 없으면 None"
    )
    diff_reason: Optional[str] = Field(None, description="차이 원인 설명")


class VerificationResult(BaseModel):
    """재현성 검증 결과"""
    verification_id: str = Field(..., description="검증 작업 ID (UUID)")
    original_extraction_id: str = Field(..., description="원본 추출 ID")
    reproduced_extraction_id: str = Field(..., description="재현 추출 ID")
    
    match_status: str = Field(
        ..., description="identical | partial | mismatch"
    )
    
    # 필드 레벨 비교 결과
    field_diffs: list[DiffDetail] = Field(default_factory=list)
    matched_fields: list[str] = Field(default_factory=list)
    mismatched_fields: list[str] = Field(default_factory=list)
    
    # 통계
    total_fields: int = Field(default=0)
    matched_count: int = Field(default=0)
    mismatch_count: int = Field(default=0)
    match_ratio: float = Field(default=0.0, description="일치율 (0.0 ~ 1.0)")
    
    verification_timestamp: datetime = Field(
        default_factory=datetime.utcnow, description="검증 완료 시간"
    )
    verification_notes: Optional[str] = Field(None)
```

### 4.2 ExtractionRecord SQLAlchemy ORM

**파일**: `/mnt/mimir/src/core/db/models/extraction.py` (추가)

```python
from sqlalchemy import Column, String, Integer, Float, DateTime, JSON, Boolean, Index
from sqlalchemy.orm import relationship
from datetime import datetime
import uuid
from enum import Enum

class ExtractionMode(str, Enum):
    """추출 모드 enum"""
    DETERMINISTIC = "deterministic"
    PROBABILISTIC = "probabilistic"


class ExtractionRecordORM(Base):
    """
    추출 작업의 완전한 조건 및 결과 저장.
    
    하나의 extraction_candidate 에 여러 ExtractionRecord 가 있을 수 있음:
    - 원본 추출
    - 재현성 검증 추출 (같은 조건 반복)
    - 다른 모델/프롬프트로 추출 (비교 분석용)
    """
    __tablename__ = 'extraction_records'
    
    id = Column(String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    
    # FK: extraction_candidate 와의 관계
    extraction_id = Column(String(36), ForeignKey('extraction_candidates.id'), nullable=False)
    
    # 문서 정보
    document_id = Column(String(36), nullable=False, index=True)
    document_version = Column(String(50), nullable=False)
    document_content_hash = Column(String(64), nullable=False)
    
    # 스키마 정보
    extraction_schema_id = Column(String(36), nullable=False, index=True)
    extraction_schema_version = Column(String(50), nullable=False)
    
    # 모델 정보
    extraction_model = Column(String(255), nullable=False)
    model_version = Column(String(100), nullable=False)
    
    # 프롬프트 정보
    extraction_prompt_version = Column(String(100), nullable=False)
    extraction_prompt_hash = Column(String(64), nullable=True)
    
    # 실행 파라미터
    extraction_mode = Column(
        String(20),
        default=ExtractionMode.PROBABILISTIC.value,
        nullable=False
    )
    temperature = Column(Float, nullable=False)
    seed = Column(Integer, nullable=True)
    max_tokens = Column(Integer, nullable=True)
    
    # 추출 결과
    extracted_result = Column(JSON, nullable=False)
    extracted_timestamp = Column(DateTime, default=datetime.utcnow, nullable=False)
    
    # 성능 메트릭
    execution_time_ms = Column(Integer, nullable=True)
    tokens_used = Column(Integer, nullable=True)
    cost_usd = Column(Float, nullable=True)
    
    # 상태
    is_successful = Column(Boolean, default=True, nullable=False)
    error_message = Column(String(1000), nullable=True)
    
    # 타임스탐프
    created_at = Column(DateTime, default=datetime.utcnow, nullable=False)
    
    # Indexes
    __table_args__ = (
        Index('ix_extraction_records_document_schema', 'document_id', 'extraction_schema_id'),
        Index('ix_extraction_records_model_version', 'extraction_model', 'model_version'),
        Index('ix_extraction_records_timestamp', 'extracted_timestamp'),
    )
    
    # Relationship
    extraction_candidate = relationship(
        "ExtractionCandidateORM",
        back_populates="records"
    )


class VerificationResultORM(Base):
    """
    재현성 검증 결과 저장.
    """
    __tablename__ = 'verification_results'
    
    id = Column(String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    
    # FK: 검증 대상 추출
    original_extraction_id = Column(
        String(36), ForeignKey('extraction_candidates.id'), nullable=False
    )
    
    # 재현 추출 기록
    reproduced_record_id = Column(
        String(36), ForeignKey('extraction_records.id'), nullable=False
    )
    
    # 검증 결과
    match_status = Column(String(20), nullable=False)  # identical, partial, mismatch
    
    # 상세 비교 결과 (JSON)
    field_diffs = Column(JSON, default=list, nullable=False)  # DiffDetail 리스트
    matched_fields = Column(JSON, default=list, nullable=False)
    mismatched_fields = Column(JSON, default=list, nullable=False)
    
    # 통계
    total_fields = Column(Integer, default=0)
    matched_count = Column(Integer, default=0)
    mismatch_count = Column(Integer, default=0)
    match_ratio = Column(Float, default=0.0)
    
    # 메모
    verification_notes = Column(String(2000), nullable=True)
    
    # 타임스탐프
    verification_timestamp = Column(DateTime, default=datetime.utcnow, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow, nullable=False)
    
    # Indexes
    __table_args__ = (
        Index('ix_verification_results_original', 'original_extraction_id'),
        Index('ix_verification_results_reproduced', 'reproduced_record_id'),
        Index('ix_verification_results_timestamp', 'verification_timestamp'),
    )
```

### 4.3 Alembic 마이그레이션

**파일**: `/mnt/mimir/src/core/db/migrations/versions/009_create_extraction_records_and_verification.py`

```python
"""Create extraction_records and verification_results tables.

Revision ID: 009
Revises: 008
Create Date: 2026-04-17 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa

revision = '009'
down_revision = '008'
branch_labels = None
depends_on = None

def upgrade():
    """Create extraction_records and verification_results tables."""
    
    # extraction_records 테이블
    op.create_table(
        'extraction_records',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('extraction_id', sa.String(36), nullable=False),
        sa.Column('document_id', sa.String(36), nullable=False),
        sa.Column('document_version', sa.String(50), nullable=False),
        sa.Column('document_content_hash', sa.String(64), nullable=False),
        sa.Column('extraction_schema_id', sa.String(36), nullable=False),
        sa.Column('extraction_schema_version', sa.String(50), nullable=False),
        sa.Column('extraction_model', sa.String(255), nullable=False),
        sa.Column('model_version', sa.String(100), nullable=False),
        sa.Column('extraction_prompt_version', sa.String(100), nullable=False),
        sa.Column('extraction_prompt_hash', sa.String(64), nullable=True),
        sa.Column('extraction_mode', sa.String(20), nullable=False, server_default='probabilistic'),
        sa.Column('temperature', sa.Float, nullable=False),
        sa.Column('seed', sa.Integer, nullable=True),
        sa.Column('max_tokens', sa.Integer, nullable=True),
        sa.Column('extracted_result', sa.JSON, nullable=False),
        sa.Column('extracted_timestamp', sa.DateTime, nullable=False, server_default=sa.func.now()),
        sa.Column('execution_time_ms', sa.Integer, nullable=True),
        sa.Column('tokens_used', sa.Integer, nullable=True),
        sa.Column('cost_usd', sa.Float, nullable=True),
        sa.Column('is_successful', sa.Boolean, nullable=False, server_default=sa.true()),
        sa.Column('error_message', sa.String(1000), nullable=True),
        sa.Column('created_at', sa.DateTime, nullable=False, server_default=sa.func.now()),
        sa.PrimaryKeyConstraint('id'),
        sa.ForeignKeyConstraint(['extraction_id'], ['extraction_candidates.id']),
    )
    
    op.create_index(
        'ix_extraction_records_document_schema',
        'extraction_records',
        ['document_id', 'extraction_schema_id']
    )
    op.create_index(
        'ix_extraction_records_model_version',
        'extraction_records',
        ['extraction_model', 'model_version']
    )
    op.create_index(
        'ix_extraction_records_timestamp',
        'extraction_records',
        ['extracted_timestamp']
    )
    
    # verification_results 테이블
    op.create_table(
        'verification_results',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('original_extraction_id', sa.String(36), nullable=False),
        sa.Column('reproduced_record_id', sa.String(36), nullable=False),
        sa.Column('match_status', sa.String(20), nullable=False),
        sa.Column('field_diffs', sa.JSON, nullable=False, server_default='[]'),
        sa.Column('matched_fields', sa.JSON, nullable=False, server_default='[]'),
        sa.Column('mismatched_fields', sa.JSON, nullable=False, server_default='[]'),
        sa.Column('total_fields', sa.Integer, nullable=False, server_default='0'),
        sa.Column('matched_count', sa.Integer, nullable=False, server_default='0'),
        sa.Column('mismatch_count', sa.Integer, nullable=False, server_default='0'),
        sa.Column('match_ratio', sa.Float, nullable=False, server_default='0.0'),
        sa.Column('verification_notes', sa.String(2000), nullable=True),
        sa.Column('verification_timestamp', sa.DateTime, nullable=False, server_default=sa.func.now()),
        sa.Column('created_at', sa.DateTime, nullable=False, server_default=sa.func.now()),
        sa.PrimaryKeyConstraint('id'),
        sa.ForeignKeyConstraint(['original_extraction_id'], ['extraction_candidates.id']),
        sa.ForeignKeyConstraint(['reproduced_record_id'], ['extraction_records.id']),
    )
    
    op.create_index(
        'ix_verification_results_original',
        'verification_results',
        ['original_extraction_id']
    )
    op.create_index(
        'ix_verification_results_reproduced',
        'verification_results',
        ['reproduced_record_id']
    )
    op.create_index(
        'ix_verification_results_timestamp',
        'verification_results',
        ['verification_timestamp']
    )


def downgrade():
    """Drop extraction_records and verification_results tables."""
    op.drop_table('verification_results')
    op.drop_table('extraction_records')
```

### 4.4 Diff 계산 서비스

**파일**: `/mnt/mimir/src/services/extraction_diff_calculator.py` (신규)

```python
"""
추출 결과 차이점 계산 및 비교.
"""

from typing import Any, Dict, List, Optional, Tuple
from difflib import SequenceMatcher
from src.core.models.extraction import DiffDetail

class DiffCalculator:
    """
    두 추출 결과를 필드별로 비교하여 차이점 계산.
    """
    
    # Fuzzy matching 임계값
    FUZZY_MATCH_THRESHOLD = 0.85
    
    # Floating point 비교 epsilon
    FLOAT_EPSILON = 1e-6
    
    @staticmethod
    def calculate_string_similarity(text1: str, text2: str) -> float:
        """
        두 문자열의 유사도 계산 (0.0 ~ 1.0).
        
        SequenceMatcher 를 사용한 간단한 구현.
        """
        matcher = SequenceMatcher(None, text1, text2)
        return matcher.ratio()
    
    @staticmethod
    def normalize_string_for_comparison(text: str) -> str:
        """
        비교를 위해 문자열 정규화: 공백, 구두점 등 처리.
        """
        import unicodedata
        
        # Whitespace 정규화
        normalized = ' '.join(text.split())
        
        # Unicode 정규화 (NFKC)
        normalized = unicodedata.normalize('NFKC', normalized)
        
        return normalized
    
    @staticmethod
    def compare_strings(
        original: str,
        reproduced: str,
        fuzzy: bool = True,
        normalize: bool = True
    ) -> Tuple[bool, Optional[float], str]:
        """
        두 문자열 비교.
        
        Returns:
            (is_match, similarity, match_type)
            - is_match: True if considered equal
            - similarity: 유사도 (0.0 ~ 1.0)
            - match_type: "exact_match" | "fuzzy_match" | "mismatch"
        """
        # 정규화 적용 여부
        if normalize:
            orig = DiffCalculator.normalize_string_for_comparison(original)
            repro = DiffCalculator.normalize_string_for_comparison(reproduced)
        else:
            orig = original
            repro = reproduced
        
        # 정확 일치
        if orig == repro:
            return True, 1.0, "exact_match"
        
        # Fuzzy matching
        if fuzzy:
            similarity = DiffCalculator.calculate_string_similarity(orig, repro)
            if similarity >= DiffCalculator.FUZZY_MATCH_THRESHOLD:
                return True, similarity, "fuzzy_match"
            else:
                return False, similarity, "mismatch"
        
        return False, 0.0, "mismatch"
    
    @staticmethod
    def compare_numbers(
        original: float | int,
        reproduced: float | int
    ) -> Tuple[bool, Optional[float], str]:
        """
        두 숫자 비교 (floating point tolerance 고려).
        
        Returns:
            (is_match, absolute_diff, match_type)
        """
        diff = abs(original - reproduced)
        
        if diff < DiffCalculator.FLOAT_EPSILON:
            return True, 0.0, "exact_match"
        
        # Relative difference (큰 숫자의 경우)
        if abs(original) > 0:
            relative_diff = diff / abs(original)
            if relative_diff < 0.01:  # 1% 허용
                return True, relative_diff, "fuzzy_match"
        
        return False, diff, "mismatch"
    
    @staticmethod
    def compare_arrays(
        original: list,
        reproduced: list
    ) -> Tuple[bool, Optional[float], str]:
        """
        두 배열 비교.
        
        순서와 내용을 모두 고려.
        """
        if len(original) != len(reproduced):
            overlap = set(original) & set(reproduced)
            similarity = len(overlap) / max(len(original), len(reproduced))
            return False, similarity, "mismatch"
        
        matches = sum(1 for o, r in zip(original, reproduced) if o == r)
        similarity = matches / len(original) if original else 1.0
        
        if similarity == 1.0:
            return True, 1.0, "exact_match"
        elif similarity > 0.8:
            return True, similarity, "fuzzy_match"
        else:
            return False, similarity, "mismatch"
    
    @staticmethod
    def compare_dicts(
        original: dict,
        reproduced: dict
    ) -> Tuple[bool, Optional[float], str]:
        """
        두 딕셔너리 비교 (deep comparison).
        """
        if set(original.keys()) != set(reproduced.keys()):
            common_keys = set(original.keys()) & set(reproduced.keys())
            key_similarity = len(common_keys) / max(len(original), len(reproduced))
            return False, key_similarity, "mismatch"
        
        # 각 키별로 비교
        matches = 0
        for key in original.keys():
            if original[key] == reproduced[key]:
                matches += 1
        
        similarity = matches / len(original) if original else 1.0
        
        if similarity == 1.0:
            return True, 1.0, "exact_match"
        elif similarity > 0.8:
            return True, similarity, "fuzzy_match"
        else:
            return False, similarity, "mismatch"
    
    @staticmethod
    def compare_values(
        field_name: str,
        original: Any,
        reproduced: Any,
        fuzzy: bool = True
    ) -> DiffDetail:
        """
        두 값을 타입에 맞게 비교하고 DiffDetail 생성.
        """
        is_match = False
        similarity = None
        diff_type = "mismatch"
        reason = None
        
        # 타입 확인
        if type(original) != type(reproduced):
            is_match = False
            diff_type = "mismatch"
            reason = f"Type mismatch: {type(original).__name__} vs {type(reproduced).__name__}"
        
        elif isinstance(original, str):
            is_match, similarity, diff_type = DiffCalculator.compare_strings(
                original, reproduced, fuzzy=fuzzy
            )
        
        elif isinstance(original, (int, float)):
            is_match, similarity, diff_type = DiffCalculator.compare_numbers(
                original, reproduced
            )
        
        elif isinstance(original, list):
            is_match, similarity, diff_type = DiffCalculator.compare_arrays(
                original, reproduced
            )
        
        elif isinstance(original, dict):
            is_match, similarity, diff_type = DiffCalculator.compare_dicts(
                original, reproduced
            )
        
        elif original == reproduced:
            is_match = True
            diff_type = "exact_match"
            similarity = 1.0
        
        return DiffDetail(
            field_name=field_name,
            original_value=original,
            reproduced_value=reproduced,
            diff_type=diff_type,
            similarity=similarity,
            diff_reason=reason
        )
    
    @staticmethod
    def compare_extraction_results(
        original_result: Dict[str, Any],
        reproduced_result: Dict[str, Any],
        fuzzy: bool = True
    ) -> Tuple[str, List[DiffDetail], List[str], List[str], float]:
        """
        두 추출 결과를 필드별로 비교.
        
        Returns:
            (match_status, field_diffs, matched_fields, mismatched_fields, match_ratio)
        """
        field_diffs: List[DiffDetail] = []
        matched_fields: List[str] = []
        mismatched_fields: List[str] = []
        
        # 모든 필드 검사
        all_fields = set(original_result.keys()) | set(reproduced_result.keys())
        
        for field_name in sorted(all_fields):
            if field_name not in original_result:
                # New field in reproduced
                field_diffs.append(DiffDetail(
                    field_name=field_name,
                    original_value=None,
                    reproduced_value=reproduced_result[field_name],
                    diff_type="mismatch",
                    diff_reason="Field missing in original result"
                ))
                mismatched_fields.append(field_name)
            
            elif field_name not in reproduced_result:
                # Removed field in reproduced
                field_diffs.append(DiffDetail(
                    field_name=field_name,
                    original_value=original_result[field_name],
                    reproduced_value=None,
                    diff_type="mismatch",
                    diff_reason="Field missing in reproduced result"
                ))
                mismatched_fields.append(field_name)
            
            else:
                # Both present: compare
                diff = DiffCalculator.compare_values(
                    field_name,
                    original_result[field_name],
                    reproduced_result[field_name],
                    fuzzy=fuzzy
                )
                field_diffs.append(diff)
                
                if diff.diff_type == "exact_match":
                    matched_fields.append(field_name)
                elif diff.diff_type == "fuzzy_match":
                    matched_fields.append(field_name)
                else:
                    mismatched_fields.append(field_name)
        
        # Match status 결정
        if mismatched_fields:
            if len(matched_fields) == 0:
                match_status = "mismatch"
            else:
                match_status = "partial"
        else:
            match_status = "identical"
        
        match_ratio = len(matched_fields) / len(all_fields) if all_fields else 0.0
        
        return match_status, field_diffs, matched_fields, mismatched_fields, match_ratio


class SpanBasedDiffCalculator:
    """
    Span 기반 차이점 계산 (FG8.3 추가).
    """
    
    @staticmethod
    def calculate_iou(
        original_span: Tuple[int, int],
        reproduced_span: Tuple[int, int]
    ) -> float:
        """
        Intersection over Union (IoU) 계산.
        
        Returns:
            IoU 값 (0.0 ~ 1.0)
        """
        start1, end1 = original_span
        start2, end2 = reproduced_span
        
        # Intersection
        inter_start = max(start1, start2)
        inter_end = min(end1, end2)
        intersection = max(0, inter_end - inter_start)
        
        # Union
        union = (end1 - start1) + (end2 - start2) - intersection
        
        if union == 0:
            return 0.0
        
        return intersection / union
    
    @staticmethod
    def compare_spans(
        original_spans: List[Tuple[int, int]],
        reproduced_spans: List[Tuple[int, int]]
    ) -> Tuple[bool, float, str]:
        """
        Span 리스트 비교.
        
        Returns:
            (is_match, average_iou, match_type)
        """
        if not original_spans and not reproduced_spans:
            return True, 1.0, "exact_match"
        
        if not original_spans or not reproduced_spans:
            return False, 0.0, "mismatch"
        
        # 모든 span 조합의 IoU 계산
        ious = []
        for orig_span in original_spans:
            for repro_span in reproduced_spans:
                iou = SpanBasedDiffCalculator.calculate_iou(orig_span, repro_span)
                ious.append(iou)
        
        avg_iou = sum(ious) / len(ious) if ious else 0.0
        
        if avg_iou >= 0.95:
            return True, avg_iou, "exact_match"
        elif avg_iou >= 0.80:
            return True, avg_iou, "fuzzy_match"
        else:
            return False, avg_iou, "mismatch"
```

### 4.5 재현성 검증 서비스

**파일**: `/mnt/mimir/src/services/extraction_verification_service.py` (신규)

```python
"""
추출 재현성 검증 서비스.
"""

from typing import Optional, Dict, Any
from datetime import datetime
import uuid

from src.core.models.extraction import (
    ExtractionRecord, ExtractionMode, VerificationResult, DiffDetail
)
from src.services.extraction_diff_calculator import DiffCalculator
from src.services.span_calculator import SpanCalculator


class ExtractionVerificationService:
    """
    추출 결과의 재현성 검증 수행.
    """
    
    def __init__(self, extraction_service, db_session):
        """
        Args:
            extraction_service: Phase 5 의 ExtractionService
            db_session: SQLAlchemy session
        """
        self.extraction_service = extraction_service
        self.db = db_session
    
    async def verify_extraction_reproducibility(
        self,
        original_extraction_id: str,
        override_params: Optional[Dict[str, Any]] = None
    ) -> VerificationResult:
        """
        원본 추출을 동일 조건으로 재추출하여 재현성 검증.
        
        Args:
            original_extraction_id: 검증 대상 추출 ID
            override_params: 파라미터 오버라이드 (옵션)
        
        Returns:
            VerificationResult 객체
        """
        
        # 1. 원본 ExtractionRecord 조회
        original_record = self._get_extraction_record(original_extraction_id)
        if not original_record:
            raise ValueError(f"Extraction record not found: {original_extraction_id}")
        
        # 2. 문서 내용 검증 (document content hash 확인)
        document = self._get_document(original_record.document_id)
        if document is None:
            raise ValueError(f"Document not found: {original_record.document_id}")
        
        actual_hash = SpanCalculator.calculate_content_hash(document.content)
        if actual_hash != original_record.document_content_hash:
            raise ValueError(
                f"Document has been modified. "
                f"Expected hash: {original_record.document_content_hash}, "
                f"Actual: {actual_hash}"
            )
        
        # 3. 재추출 파라미터 준비
        extraction_params = {
            'document_id': original_record.document_id,
            'schema_id': original_record.extraction_schema_id,
            'model': original_record.extraction_model,
            'model_version': original_record.model_version,
            'prompt_version': original_record.extraction_prompt_version,
            'temperature': original_record.temperature,
            'seed': original_record.seed,
            'max_tokens': original_record.max_tokens,
        }
        
        # Override 파라미터 적용
        if override_params:
            extraction_params.update(override_params)
        
        # 4. 재추출 실행
        reproduced_extraction = await self.extraction_service.extract(
            document_id=extraction_params['document_id'],
            schema_id=extraction_params['schema_id'],
            model=extraction_params['model'],
            model_version=extraction_params['model_version'],
            temperature=extraction_params['temperature'],
            seed=extraction_params.get('seed'),
            max_tokens=extraction_params.get('max_tokens'),
        )
        
        # 5. ExtractionRecord 저장
        reproduced_record = self._save_extraction_record(
            extraction_id=reproduced_extraction.id,
            params=extraction_params,
            result=reproduced_extraction.result,
            execution_time=reproduced_extraction.execution_time_ms,
            tokens=reproduced_extraction.tokens_used,
        )
        
        # 6. 결과 비교
        match_status, field_diffs, matched, mismatched, match_ratio = (
            DiffCalculator.compare_extraction_results(
                original_record.extracted_result,
                reproduced_extraction.result,
                fuzzy=True
            )
        )
        
        # 7. VerificationResult 생성 및 저장
        verification_id = str(uuid.uuid4())
        verification = VerificationResult(
            verification_id=verification_id,
            original_extraction_id=original_extraction_id,
            reproduced_extraction_id=reproduced_record.id,
            match_status=match_status,
            field_diffs=field_diffs,
            matched_fields=matched,
            mismatched_fields=mismatched,
            total_fields=len(matched) + len(mismatched),
            matched_count=len(matched),
            mismatch_count=len(mismatched),
            match_ratio=match_ratio,
        )
        
        # DB 에 저장
        self._save_verification_result(verification)
        
        return verification
    
    def ensure_deterministic_mode(self, record: ExtractionRecord) -> None:
        """
        재현성 검증이 결정론적 모드에서 수행되도록 강제.
        """
        if record.extraction_mode != ExtractionMode.DETERMINISTIC:
            raise ValueError(
                f"Reproducibility verification requires deterministic mode, "
                f"got {record.extraction_mode}"
            )
        
        if record.temperature != 0.0:
            raise ValueError(
                f"Deterministic mode requires temperature=0.0, got {record.temperature}"
            )
    
    def _get_extraction_record(self, extraction_id: str) -> Optional[ExtractionRecord]:
        """DB 에서 ExtractionRecord 조회 (Pydantic 모델로 변환)"""
        # 실제 구현: ORM 을 통한 DB 조회
        # from src.core.db.models.extraction import ExtractionRecordORM
        # orm_record = self.db.query(ExtractionRecordORM).filter_by(id=extraction_id).first()
        # if orm_record:
        #     return ExtractionRecord(...) # ORM 을 Pydantic 으로 변환
        pass
    
    def _get_document(self, document_id: str):
        """DB 에서 Document 조회"""
        # from src.core.db.models.document import DocumentORM
        # return self.db.query(DocumentORM).filter_by(id=document_id).first()
        pass
    
    def _save_extraction_record(
        self,
        extraction_id: str,
        params: Dict,
        result: Dict,
        execution_time: int,
        tokens: int
    ):
        """ExtractionRecord 저장"""
        # from src.core.db.models.extraction import ExtractionRecordORM
        # orm_record = ExtractionRecordORM(
        #     id=str(uuid.uuid4()),
        #     extraction_id=extraction_id,
        #     document_id=params['document_id'],
        #     ...
        # )
        # self.db.add(orm_record)
        # self.db.commit()
        # return orm_record
        pass
    
    def _save_verification_result(self, verification: VerificationResult):
        """VerificationResult 저장"""
        # from src.core.db.models.extraction import VerificationResultORM
        # orm_result = VerificationResultORM(
        #     id=verification.verification_id,
        #     original_extraction_id=verification.original_extraction_id,
        #     ...
        # )
        # self.db.add(orm_result)
        # self.db.commit()
        pass
```

### 4.6 API 엔드포인트 구현

**파일**: `/mnt/mimir/src/api/routes/extraction_verification.py` (신규)

```python
"""
추출 재현성 검증 API.

S2 원칙 ⑤ ⑥ 적용:
- AI agents are first-class API consumers: actor_type 감사 로깅
- Scope Profile ACL: 모든 API 에 접근 제어 적용
"""

from fastapi import APIRouter, Depends, HTTPException, Query
from typing import Optional, Dict, Any
from datetime import datetime

from src.core.models.extraction import VerificationResult
from src.services.extraction_verification_service import ExtractionVerificationService
from src.api.dependencies import (
    get_current_user,
    get_db,
    check_scope_access,
    audit_log_action,
)

router = APIRouter(prefix="/api/v1/extractions", tags=["extraction-verification"])


@router.post("/{extraction_id}/verify")
async def verify_extraction_reproducibility(
    extraction_id: str,
    override_params: Optional[Dict[str, Any]] = None,
    current_user: Dict = Depends(get_current_user),
    db = Depends(get_db),
) -> VerificationResult:
    """
    추출 재현성 검증 API.
    
    POST /api/v1/extractions/{extraction_id}/verify
    
    재현성을 확인하기 위해 동일 조건으로 재추출을 수행합니다.
    원본 추출과 재현 추출의 결과를 필드별로 비교합니다.
    
    Args:
        extraction_id: 검증 대상 추출 ID
        override_params: 파라미터 오버라이드 (선택사항)
            {
                "temperature": 0.0,
                "seed": 42,
                "model_version": "gpt-4-0125-preview"
            }
    
    Returns:
        VerificationResult:
            {
                "match_status": "identical" | "partial" | "mismatch",
                "field_diffs": [...],
                "matched_fields": [...],
                "mismatched_fields": [...],
                "match_ratio": 0.95,
                ...
            }
    
    Raises:
        403 Unauthorized: 사용자가 이 추출에 접근할 권한 없음
        404 Not Found: 추출을 찾을 수 없음
        400 Bad Request: 추출이 검증 불가능한 상태 (문서 변경됨 등)
    """
    
    # S2 원칙 ⑥: Scope Profile ACL 확인
    # 사용자가 이 추출에 접근할 수 있는지 확인
    scope_check = await check_scope_access(
        user_id=current_user['id'],
        resource_type='extraction',
        resource_id=extraction_id,
        action='verify'
    )
    
    if not scope_check['allowed']:
        # S2 원칙 ⑤: actor_type 감사 로깅
        await audit_log_action(
            action='extraction_verify_denied',
            actor_type='user',  # or 'agent'
            actor_id=current_user['id'],
            resource_id=extraction_id,
            reason=scope_check['reason'],
            db=db
        )
        raise HTTPException(
            status_code=403,
            detail=f"Access denied: {scope_check['reason']}"
        )
    
    # 재현성 검증 서비스 초기화
    verification_service = ExtractionVerificationService(
        extraction_service=...,  # Phase 5 service injection
        db_session=db
    )
    
    try:
        verification_result = await verification_service.verify_extraction_reproducibility(
            original_extraction_id=extraction_id,
            override_params=override_params
        )
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
    
    # S2 원칙 ⑤: 감사 로깅 (성공)
    await audit_log_action(
        action='extraction_verify',
        actor_type='user',  # or 'agent'
        actor_id=current_user['id'],
        resource_id=extraction_id,
        details={
            'verification_id': verification_result.verification_id,
            'match_status': verification_result.match_status,
            'match_ratio': verification_result.match_ratio,
        },
        db=db
    )
    
    return verification_result


@router.get("/{extraction_id}/audit")
async def get_extraction_audit_trail(
    extraction_id: str,
    start_date: Optional[datetime] = Query(None),
    end_date: Optional[datetime] = Query(None),
    field_name: Optional[str] = Query(None),
    current_user: Dict = Depends(get_current_user),
    db = Depends(get_db),
) -> Dict:
    """
    추출 작업의 완전한 감사 추적 조회.
    
    GET /api/v1/extractions/{extraction_id}/audit
    
    특정 추출의:
    - 원본 ExtractionRecord (조건, 결과)
    - 모든 재현 시도 및 검증 결과
    - 필드별 변경 히스토리
    
    Args:
        extraction_id: 추출 ID
        start_date: 시작 날짜 (선택)
        end_date: 종료 날짜 (선택)
        field_name: 특정 필드만 조회 (선택)
    
    Returns:
        {
            "original_extraction": ExtractionRecord,
            "verification_history": [VerificationResult, ...],
            "field_change_timeline": [...],
            "extraction_record_variants": [...]
        }
    """
    
    # S2 원칙 ⑥: ACL 확인
    scope_check = await check_scope_access(
        user_id=current_user['id'],
        resource_type='extraction',
        resource_id=extraction_id,
        action='audit'
    )
    
    if not scope_check['allowed']:
        await audit_log_action(
            action='extraction_audit_denied',
            actor_type='user',
            actor_id=current_user['id'],
            resource_id=extraction_id,
            reason=scope_check['reason'],
            db=db
        )
        raise HTTPException(status_code=403, detail=scope_check['reason'])
    
    # DB 에서 감사 기록 조회
    # 1. 원본 ExtractionRecord
    # 2. 모든 VerificationResult
    # 3. 필드별 변경 타임라인 재구성
    
    # (실제 구현은 생략 - DB 쿼리 및 데이터 조합)
    
    await audit_log_action(
        action='extraction_audit_viewed',
        actor_type='user',
        actor_id=current_user['id'],
        resource_id=extraction_id,
        db=db
    )
    
    return {
        "extraction_id": extraction_id,
        "original_extraction": {...},
        "verification_history": [...],
        "field_change_timeline": [...],
    }


@router.get("/{extraction_id}/verification-results")
async def list_verification_results(
    extraction_id: str,
    limit: int = Query(20, ge=1, le=100),
    offset: int = Query(0, ge=0),
    current_user: Dict = Depends(get_current_user),
    db = Depends(get_db),
) -> Dict:
    """
    특정 추출의 모든 검증 결과 나열.
    
    GET /api/v1/extractions/{extraction_id}/verification-results
    """
    
    scope_check = await check_scope_access(
        user_id=current_user['id'],
        resource_type='extraction',
        resource_id=extraction_id,
        action='read'
    )
    
    if not scope_check['allowed']:
        raise HTTPException(status_code=403, detail=scope_check['reason'])
    
    # DB 에서 verification_results 조회 (페이징 적용)
    # results = db.query(VerificationResultORM) \
    #     .filter_by(original_extraction_id=extraction_id) \
    #     .order_by(VerificationResultORM.verification_timestamp.desc()) \
    #     .limit(limit) \
    #     .offset(offset) \
    #     .all()
    
    await audit_log_action(
        action='verification_results_listed',
        actor_type='user',
        actor_id=current_user['id'],
        resource_id=extraction_id,
        db=db
    )
    
    return {
        "extraction_id": extraction_id,
        "total_count": 0,  # actual count from DB
        "results": [],  # actual results
        "limit": limit,
        "offset": offset,
    }
```

### 4.7 단위 테스트

**파일**: `/mnt/mimir/tests/test_extraction_diff_calculator.py` (신규)

```python
"""
추출 결과 diff 계산 테스트.
"""

import pytest
from src.services.extraction_diff_calculator import (
    DiffCalculator, SpanBasedDiffCalculator
)
from src.core.models.extraction import DiffDetail


class TestDiffCalculator:
    """DiffCalculator 단위 테스트"""
    
    def test_string_similarity_exact_match(self):
        """정확한 문자열 일치"""
        similarity = DiffCalculator.calculate_string_similarity("hello", "hello")
        assert similarity == 1.0
    
    def test_string_similarity_partial(self):
        """부분 일치"""
        similarity = DiffCalculator.calculate_string_similarity("hello world", "hello")
        assert 0.4 < similarity < 0.6
    
    def test_normalize_string(self):
        """문자열 정규화"""
        result = DiffCalculator.normalize_string_for_comparison("  hello   world  ")
        assert result == "hello world"
    
    def test_compare_strings_exact(self):
        """문자열 비교 - 정확 일치"""
        is_match, sim, match_type = DiffCalculator.compare_strings(
            "hello", "hello"
        )
        assert is_match
        assert sim == 1.0
        assert match_type == "exact_match"
    
    def test_compare_strings_fuzzy(self):
        """문자열 비교 - Fuzzy matching"""
        is_match, sim, match_type = DiffCalculator.compare_strings(
            "hello world", "hello  world", fuzzy=True
        )
        assert is_match
        assert match_type == "fuzzy_match"
    
    def test_compare_numbers_exact(self):
        """숫자 비교 - 정확 일치"""
        is_match, diff, match_type = DiffCalculator.compare_numbers(42, 42)
        assert is_match
        assert diff == 0.0
        assert match_type == "exact_match"
    
    def test_compare_numbers_float_tolerance(self):
        """숫자 비교 - floating point tolerance"""
        is_match, diff, match_type = DiffCalculator.compare_numbers(42.0, 42.0000001)
        assert is_match  # epsilon 내에서 일치
    
    def test_compare_arrays_exact(self):
        """배열 비교 - 정확 일치"""
        is_match, sim, match_type = DiffCalculator.compare_arrays(
            [1, 2, 3], [1, 2, 3]
        )
        assert is_match
        assert sim == 1.0
    
    def test_compare_arrays_different_length(self):
        """배열 비교 - 길이 차이"""
        is_match, sim, match_type = DiffCalculator.compare_arrays(
            [1, 2, 3], [1, 2]
        )
        assert not is_match
    
    def test_compare_extraction_results_identical(self):
        """추출 결과 비교 - 동일"""
        original = {"name": "Alice", "age": 30}
        reproduced = {"name": "Alice", "age": 30}
        
        status, diffs, matched, mismatched, ratio = (
            DiffCalculator.compare_extraction_results(original, reproduced)
        )
        
        assert status == "identical"
        assert ratio == 1.0
        assert len(mismatched) == 0
    
    def test_compare_extraction_results_partial(self):
        """추출 결과 비교 - 부분 일치"""
        original = {"name": "Alice", "age": 30, "email": "alice@example.com"}
        reproduced = {"name": "Alice", "age": 31, "email": "alice@example.com"}
        
        status, diffs, matched, mismatched, ratio = (
            DiffCalculator.compare_extraction_results(original, reproduced)
        )
        
        assert status == "partial"
        assert "age" in mismatched
        assert "name" in matched
        assert ratio > 0.5
    
    def test_compare_extraction_results_mismatch(self):
        """추출 결과 비교 - 완전 불일치"""
        original = {"name": "Alice", "age": 30}
        reproduced = {"name": "Bob", "age": 25}
        
        status, diffs, matched, mismatched, ratio = (
            DiffCalculator.compare_extraction_results(original, reproduced)
        )
        
        assert status == "mismatch"
        assert len(mismatched) == 2


class TestSpanBasedDiffCalculator:
    """SpanBasedDiffCalculator 테스트"""
    
    def test_calculate_iou_exact_match(self):
        """IoU 계산 - 정확 일치"""
        iou = SpanBasedDiffCalculator.calculate_iou(
            (10, 20), (10, 20)
        )
        assert iou == 1.0
    
    def test_calculate_iou_partial_overlap(self):
        """IoU 계산 - 부분 겹침"""
        iou = SpanBasedDiffCalculator.calculate_iou(
            (10, 20), (15, 25)
        )
        assert 0.3 < iou < 0.4  # 5 / 15
    
    def test_calculate_iou_no_overlap(self):
        """IoU 계산 - 겹치지 않음"""
        iou = SpanBasedDiffCalculator.calculate_iou(
            (10, 20), (30, 40)
        )
        assert iou == 0.0
    
    def test_compare_spans_exact(self):
        """Span 비교 - 정확 일치"""
        is_match, iou, match_type = SpanBasedDiffCalculator.compare_spans(
            [(10, 20)], [(10, 20)]
        )
        assert is_match
        assert match_type == "exact_match"
    
    def test_compare_spans_fuzzy(self):
        """Span 비교 - Fuzzy matching"""
        is_match, iou, match_type = SpanBasedDiffCalculator.compare_spans(
            [(10, 20)], [(10, 21)]
        )
        assert is_match
        assert match_type == "fuzzy_match"
```

### 4.8 통합 테스트

**파일**: `/mnt/mimir/tests/test_extraction_verification_integration.py` (신규)

```python
"""
추출 재현성 검증 통합 테스트.
"""

import pytest
from datetime import datetime

from src.core.models.extraction import (
    ExtractionRecord, ExtractionMode, VerificationResult
)
from src.services.extraction_verification_service import ExtractionVerificationService


class TestExtractionVerificationIntegration:
    """통합 테스트: 전체 검증 워크플로우"""
    
    @pytest.mark.asyncio
    async def test_full_verification_workflow(self, mock_extraction_service, db_session):
        """전체 재현성 검증 워크플로우"""
        
        # 1. 원본 추출 기록
        original_record = ExtractionRecord(
            extraction_id="extraction_001",
            document_id="doc_001",
            document_version="1.0",
            document_content_hash="hash_abc123",
            extraction_schema_id="schema_001",
            extraction_schema_version="1.0",
            extraction_model="gpt-4",
            model_version="gpt-4-0125-preview",
            extraction_prompt_version="1.0.0",
            extraction_mode=ExtractionMode.DETERMINISTIC,
            temperature=0.0,
            seed=42,
            extracted_result={"name": "Alice", "age": 30},
        )
        
        # 2. 검증 서비스 초기화
        verification_service = ExtractionVerificationService(
            extraction_service=mock_extraction_service,
            db_session=db_session
        )
        
        # 3. 검증 실행
        # (실제로는 재추출이 수행되고, mock 에서 동일한 결과 반환)
        verification_result = await verification_service.verify_extraction_reproducibility(
            original_extraction_id=original_record.extraction_id
        )
        
        # 4. 검증 결과 확인
        assert verification_result.match_status == "identical"
        assert verification_result.match_ratio == 1.0
        assert len(verification_result.mismatched_fields) == 0
    
    @pytest.mark.asyncio
    async def test_verification_with_model_change(self):
        """모델 변경 후 재현성 검증"""
        # 원본: gpt-4, 재현: gpt-4-turbo
        # 결과가 다를 가능성이 높음 → partial 또는 mismatch 예상
        pass
    
    @pytest.mark.asyncio
    async def test_verification_document_changed_error(self):
        """문서가 변경된 경우 검증 실패"""
        # Document content hash 불일치 → ValueError 발생
        pass
    
    def test_deterministic_mode_enforcement(self):
        """결정론적 모드 강제"""
        record = ExtractionRecord(
            extraction_id="ext_001",
            document_id="doc_001",
            document_version="1.0",
            document_content_hash="hash_123",
            extraction_schema_id="schema_001",
            extraction_schema_version="1.0",
            extraction_model="gpt-4",
            model_version="gpt-4-0125-preview",
            extraction_prompt_version="1.0.0",
            extraction_mode=ExtractionMode.PROBABILISTIC,
            temperature=0.7,  # 결정론적이 아님
            extracted_result={},
        )
        
        verification_service = ExtractionVerificationService(..., ...)
        
        with pytest.raises(ValueError, match="Deterministic mode required"):
            verification_service.ensure_deterministic_mode(record)
```

---

## 5. 산출물

### 5.1 코드 파일

| 경로 | 설명 | 상태 |
|------|------|------|
| `/mnt/mimir/src/core/models/extraction.py` | ExtractionRecord, VerificationResult, DiffDetail Pydantic 모델 | 수정 |
| `/mnt/mimir/src/core/db/models/extraction.py` | ExtractionRecordORM, VerificationResultORM SQLAlchemy 모델 | 신규 |
| `/mnt/mimir/src/core/db/migrations/versions/009_create_extraction_records_and_verification.py` | Alembic 마이그레이션 | 신규 |
| `/mnt/mimir/src/services/extraction_diff_calculator.py` | DiffCalculator, SpanBasedDiffCalculator 서비스 | 신규 |
| `/mnt/mimir/src/services/extraction_verification_service.py` | ExtractionVerificationService 구현 | 신규 |
| `/mnt/mimir/src/api/routes/extraction_verification.py` | 검증 API 엔드포인트 (S2 원칙 적용) | 신규 |
| `/mnt/mimir/tests/test_extraction_diff_calculator.py` | Diff 계산 단위 테스트 | 신규 |
| `/mnt/mimir/tests/test_extraction_verification_integration.py` | 검증 통합 테스트 | 신규 |

### 5.2 데이터베이스 스키마

| 테이블 | 설명 |
|--------|------|
| `extraction_records` | 추출 작업의 완전한 조건 및 결과 기록 |
| `verification_results` | 재현성 검증 결과 저장 |

---

## 6. 완료 기준

1. ✓ ExtractionRecord Pydantic 모델 구현 + validation
2. ✓ ExtractionMode enum (deterministic/probabilistic)
3. ✓ VerificationResult 모델 구현
4. ✓ DiffDetail 모델 구현
5. ✓ ExtractionRecordORM SQLAlchemy 모델 + 관계 정의
6. ✓ VerificationResultORM SQLAlchemy 모델
7. ✓ Alembic 마이그레이션 작성 및 테스트
8. ✓ DiffCalculator: 문자열, 숫자, 배열, 딕셔너리 비교
9. ✓ SpanBasedDiffCalculator: IoU 계산 및 span 비교
10. ✓ ExtractionVerificationService: 재현성 검증 로직
11. ✓ POST /api/v1/extractions/{extraction_id}/verify 엔드포인트
12. ✓ GET /api/v1/extractions/{extraction_id}/audit 엔드포인트
13. ✓ GET /api/v1/extractions/{extraction_id}/verification-results 엔드포인트
14. ✓ S2 원칙 ⑤ ⑥ 적용: actor_type 감사 로깅, Scope Profile ACL
15. ✓ 결정론적 모드 강제 (temperature=0.0, seed)
16. ✓ 단위 테스트: 최소 20개 (diff 계산, 비교 로직, edge cases)
17. ✓ 통합 테스트: 전체 검증 워크플로우
18. ✓ 모든 API 에 docstring 및 예제 포함
19. ✓ Type hints 완전 구현

---

## 7. 작업 지침

### 지침 7-1: Diff 계산 견고성

여러 타입의 값 비교를 지원해야 합니다. 각 타입별로 적절한 비교 로직을 적용합니다:

```python
# String: 정확 일치 또는 fuzzy matching
# Number: floating point tolerance (epsilon)
# Array: element-wise 비교
# Dict: recursive deep diff
# Custom types: __eq__ 메서드 활용
```

### 지침 7-2: 감사 로깅 (S2 원칙 ⑤)

모든 검증 API 호출을 감사 로그에 기록합니다:

```python
await audit_log_action(
    action='extraction_verify',
    actor_type='user',  # or 'agent' (S2 원칙 ⑤)
    actor_id=current_user['id'],
    resource_id=extraction_id,
    details={
        'match_status': verification_result.match_status,
        'match_ratio': verification_result.match_ratio,
    },
    db=db
)
```

### 지침 7-3: ACL 적용 (S2 원칙 ⑥)

모든 검증 API 에 Scope Profile 기반 접근 제어를 적용합니다:

```python
scope_check = await check_scope_access(
    user_id=user_id,
    resource_type='extraction',
    resource_id=extraction_id,
    action='verify'  # or 'audit'
)

if not scope_check['allowed']:
    raise HTTPException(status_code=403, detail=scope_check['reason'])
```

### 지침 7-4: 결정론적 재현 보장

재현성 검증은 반드시 결정론적 모드에서 수행되어야 합니다:

```python
def ensure_deterministic_reproducibility(self, record: ExtractionRecord):
    if record.extraction_mode != ExtractionMode.DETERMINISTIC:
        raise ValueError("Deterministic mode required for reproducibility")
    if record.temperature != 0.0:
        raise ValueError("Temperature must be 0.0 for deterministic mode")
```

### 지침 7-5: 문서 무결성 검증

재현 추출 전에 문서의 내용이 변경되지 않았는지 확인합니다:

```python
document = get_document(original_record.document_id)
actual_hash = calculate_content_hash(document.content)

if actual_hash != original_record.document_content_hash:
    raise ValueError(
        "Document has been modified. "
        "Reproducibility verification cannot proceed."
    )
```

### 지침 7-6: 재현성 키(Reproducibility Key) 생성

동일 조건으로의 추출을 식별하기 위해 unique key 를 생성합니다:

```python
def get_reproducibility_key(self) -> str:
    """
    다음 항목을 포함하는 SHA256 hash:
    - document_id
    - document_version
    - schema_id
    - model + model_version
    - prompt_version
    - temperature
    - seed
    """
    key_data = {
        "document_id": self.document_id,
        "document_version": self.document_version,
        ...
    }
    return hashlib.sha256(
        json.dumps(key_data, sort_keys=True).encode()
    ).hexdigest()
```

### 지침 7-7: Fuzzy Matching 임계값 설정

fuzzy matching 의 경우 임계값을 명시적으로 정의합니다:

```python
FUZZY_MATCH_THRESHOLD = 0.85  # 85% 이상 유사하면 일치로 판정
FLOAT_EPSILON = 1e-6           # 숫자 비교 허용오차
RELATIVE_TOLERANCE = 0.01      # 상대 오차율 1%
```

### 지침 7-8: 검증 결과 저장 및 조회

모든 검증 결과는 DB 에 저장되어 나중에 조회할 수 있어야 합니다:

```python
# verification_results 테이블에 저장
verification_result_orm = VerificationResultORM(
    id=uuid.uuid4(),
    original_extraction_id=original_id,
    reproduced_record_id=reproduced_id,
    match_status=status,
    field_diffs=diffs,
    ...
)
db.add(verification_result_orm)
db.commit()
```

### 지침 7-9: 필드별 변경 타임라인 추적

여러 번의 검증을 통해 필드별 변경 과정을 추적할 수 있어야 합니다:

```python
# audit API 응답 예시
{
    "field_change_timeline": [
        {
            "timestamp": "2026-04-17T10:00:00Z",
            "field_name": "company_name",
            "original_value": "Acme Corp",
            "reproduced_value": "Acme Corp",
            "status": "unchanged",
            "verification_id": "ver_001"
        },
        {
            "timestamp": "2026-04-17T10:30:00Z",
            "field_name": "founded_year",
            "original_value": 2010,
            "reproduced_value": 2011,
            "status": "changed",
            "verification_id": "ver_002"
        }
    ]
}
```

### 지침 7-10: 성능 고려사항

대규모 추출 결과 비교 시 성능 최적화:

- Field-level 비교는 병렬화 불가능 (순차 처리)
- JSON 직렬화/역직렬화 최소화
- 검증 결과는 캐싱 가능 (reproducibility key 기반)
- Large document (> 10MB) 처리 테스트 필수

---

**끝.**
