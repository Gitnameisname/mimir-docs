# FG8.3 작업지시서 - task8-8
# SourceSpan 역참조 + ExtractedFieldWithAttribution 모델

**버전**: 1.0  
**작성일**: 2026-04-17  
**FG**: FG8.3 — 추출 결과의 검증 가능 계약  
**담당**: Phase 8 팀

---

## 1. 작업 목적

추출된 데이터가 원문의 정확한 위치를 추적할 수 있도록 SourceSpan 모델과 ExtractedFieldWithAttribution 데이터 구조를 설계 및 구현한다. 이를 통해:

- 추출 결과의 원문 근거(provenance) 기록
- 동일 조건 재현 검증 시 span 일치도 측정
- UI 에서 원문 하이라이트 표시 가능
- Phase 2 Citation 5-tuple 과 호환되는 span 개념 도입

---

## 2. 작업 범위

### 포함 범위

1. **SourceSpan 데이터 모델**
   - Pydantic 정의: `document_id`, `version_id`, `node_id`, `span_offset` (tuple[int, int]), `source_text`, `content_hash`
   - Field validation: span offset 경계 검사, hash 검증
   - JSON 직렬화 지원

2. **ExtractedFieldWithAttribution 모델**
   - 필드명, 추출값, source_spans 리스트
   - 다중 span 지원 (단일 필드가 여러 곳에서 추출된 경우)
   - Pydantic 정의 + 타입 검증

3. **SQLAlchemy ORM 계층**
   - `extraction_spans` 테이블 스키마 정의
   - `ExtractionSpanORM` 모델 + 외래키 관계
   - extraction_candidates 테이블과의 조인 설정

4. **Alembic 마이그레이션**
   - extraction_spans 테이블 생성 스크립트
   - 인덱스: (document_id, extraction_id), (document_id, node_id)

5. **Span 계산 서비스**
   - LLM 응답에서 위치 힌트(예: "단락 3의 4-10번째 문자") 파싱
   - 문서의 실제 텍스트 기반 정확한 character offset 계산
   - 다중 span 자동 병합 로직 (overlapping span 처리)

6. **Content Hash 검증**
   - Document 버전별 content hash 계산 및 저장
   - Span 추출 텍스트와 실제 문서 내용 비교 검증
   - Hash 불일치 시 에러 처리

7. **LLM 프롬프트 개선**
   - 추출 지시문에 "반드시 추출한 텍스트의 위치(단락 번호/키)를 명시" 추가
   - 모델이 location hint 를 JSON 응답에 포함하도록 구조화

8. **Span 시각화 데이터**
   - UI 용 highlight range 표현: `{ start: int, end: int, field_name: str }`
   - 배치 변환 함수 제공

9. **단위 테스트**
   - Span 계산 정확성: 정상 경우, edge case (빈 span, 다중 span)
   - Hash 검증: 일치, 불일치, 손상된 문서
   - 다국어 문자열 offset (UTF-8 바이트 vs 문자 인덱스)

### 제외 범위

- Phase 2 Citation 5-tuple 자체의 리팩토링 (호환성만 확인)
- UI 하이라이트 렌더링 구현 (데이터 제공만)
- 외부 벡터 DB 의존성 추가
- Span 별 confidence score (Phase 9 에서 고려)

---

## 3. 선행 조건

1. Phase 2 완료: Document, DocumentVersion ORM 모델 존재
2. Phase 5 완료: ExtractionCandidate, ExtractionSchema ORM 모델 존재
3. 프로젝트 구조 확인:
   - `/mnt/mimir/src/core/models/` — Pydantic 모델
   - `/mnt/mimir/src/core/db/models/` — SQLAlchemy ORM
   - `/mnt/mimir/src/core/db/migrations/` — Alembic 스크립트
   - `/mnt/mimir/src/services/` — 비즈니스 로직
   - `/mnt/mimir/tests/` — 테스트 코드
4. 데이터베이스 백업 완료 (Alembic 마이그레이션 적용 전)
5. Pydantic v2 이상 설치 확인

---

## 4. 주요 작업 항목

### 4.1 SourceSpan Pydantic 모델 정의

**파일**: `/mnt/mimir/src/core/models/extraction.py`

```python
# extraction.py 에 추가
from pydantic import BaseModel, Field, field_validator
from typing import Tuple, Optional
from datetime import datetime

class SourceSpan(BaseModel):
    """
    원문에서 추출된 텍스트의 정확한 위치 정보.
    
    Attributes:
        document_id: 원본 문서 ID (UUID)
        version_id: 문서 버전 ID (UUID)
        node_id: 문서 내 노드/섹션 ID (문서 구조가 계층적일 경우)
        span_offset: (start, end) 문자 인덱스 쌍 (0-based, 끝은 exclusive)
        source_text: 실제 추출된 텍스트 (UTF-8)
        content_hash: 추출된 텍스트의 SHA256 해시 (검증용)
    """
    document_id: str = Field(..., description="원본 문서 ID")
    version_id: str = Field(..., description="문서 버전 ID")
    node_id: Optional[str] = Field(None, description="문서 노드/섹션 ID")
    span_offset: Tuple[int, int] = Field(..., description="(start, end) 문자 인덱스")
    source_text: str = Field(..., description="실제 추출된 텍스트")
    content_hash: str = Field(..., description="source_text 의 SHA256 해시")
    
    @field_validator('span_offset')
    @classmethod
    def validate_span_offset(cls, v: Tuple[int, int]) -> Tuple[int, int]:
        """Span offset 유효성 검사: 0 <= start < end"""
        if not isinstance(v, (tuple, list)) or len(v) != 2:
            raise ValueError("span_offset must be a tuple of (start, end)")
        start, end = v
        if not (isinstance(start, int) and isinstance(end, int)):
            raise ValueError("span_offset must contain integers")
        if start < 0 or end < 0:
            raise ValueError("span_offset must be non-negative")
        if start >= end:
            raise ValueError("span_offset start must be less than end")
        return (int(start), int(end))
    
    @field_validator('content_hash')
    @classmethod
    def validate_hash_format(cls, v: str) -> str:
        """Content hash 는 64 자리 16진수 (SHA256)"""
        if not isinstance(v, str) or len(v) != 64:
            raise ValueError("content_hash must be a 64-character hex string (SHA256)")
        try:
            int(v, 16)
        except ValueError:
            raise ValueError("content_hash must be a valid hex string")
        return v
    
    class Config:
        json_schema_extra = {
            "example": {
                "document_id": "550e8400-e29b-41d4-a716-446655440000",
                "version_id": "550e8400-e29b-41d4-a716-446655440001",
                "node_id": "section_2_paragraph_3",
                "span_offset": [142, 267],
                "source_text": "The quick brown fox jumps over the lazy dog",
                "content_hash": "d7a8fbb307d7d6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c"
            }
        }


class ExtractedFieldWithAttribution(BaseModel):
    """
    추출된 필드와 그 원문 근거 정보.
    
    Attributes:
        field_name: 추출 스키마에 정의된 필드명
        extracted_value: 추출된 값 (Any, 스키마 정의에 따라)
        source_spans: 해당 필드의 원문 위치 리스트 (다중 span 지원)
        extraction_confidence: 추출 신뢰도 (0.0 ~ 1.0, 옵션)
    """
    field_name: str = Field(..., description="추출 스키마의 필드명")
    extracted_value: Optional[str | int | float | bool | dict] = Field(
        None, description="추출된 값"
    )
    source_spans: list[SourceSpan] = Field(
        default_factory=list, description="원문 위치 정보 리스트"
    )
    extraction_confidence: Optional[float] = Field(
        None, description="추출 신뢰도 (0.0 ~ 1.0)"
    )
    
    @field_validator('extraction_confidence')
    @classmethod
    def validate_confidence(cls, v: Optional[float]) -> Optional[float]:
        """추출 신뢰도는 0.0 ~ 1.0 범위"""
        if v is not None:
            if not isinstance(v, (int, float)) or not (0.0 <= v <= 1.0):
                raise ValueError("extraction_confidence must be between 0.0 and 1.0")
        return v
    
    def has_source_attribution(self) -> bool:
        """이 필드가 source span 기반 근거를 가지고 있는가"""
        return len(self.source_spans) > 0
    
    class Config:
        json_schema_extra = {
            "example": {
                "field_name": "company_name",
                "extracted_value": "Acme Corporation",
                "source_spans": [
                    {
                        "document_id": "550e8400-e29b-41d4-a716-446655440000",
                        "version_id": "550e8400-e29b-41d4-a716-446655440001",
                        "node_id": "paragraph_1",
                        "span_offset": [10, 26],
                        "source_text": "Acme Corporation",
                        "content_hash": "abc123..."
                    }
                ],
                "extraction_confidence": 0.95
            }
        }
```

### 4.2 ExtractedFieldWithAttribution 배치 모델

**파일**: `/mnt/mimir/src/core/models/extraction.py` (동일 파일에 추가)

```python
class ExtractionResultWithAttribution(BaseModel):
    """
    전체 추출 결과 + 모든 필드의 원문 근거.
    """
    extraction_id: str = Field(..., description="추출 작업 ID")
    document_id: str = Field(..., description="원본 문서 ID")
    schema_id: str = Field(..., description="추출 스키마 ID")
    fields: list[ExtractedFieldWithAttribution] = Field(
        ..., description="모든 추출 필드"
    )
    extraction_timestamp: datetime = Field(
        default_factory=datetime.utcnow, description="추출 수행 시간"
    )
    model_used: str = Field(..., description="사용된 LLM 모델명")
    
    def get_field_by_name(self, field_name: str) -> Optional[ExtractedFieldWithAttribution]:
        """필드명으로 추출 필드 조회"""
        for field in self.fields:
            if field.field_name == field_name:
                return field
        return None
    
    def get_attributed_fields(self) -> list[ExtractedFieldWithAttribution]:
        """근거가 있는 필드만 필터링"""
        return [f for f in self.fields if f.has_source_attribution()]
    
    def get_unattributed_fields(self) -> list[ExtractedFieldWithAttribution]:
        """근거가 없는 필드 조회"""
        return [f for f in self.fields if not f.has_source_attribution()]
```

### 4.3 SQLAlchemy ORM 모델

**파일**: `/mnt/mimir/src/core/db/models/extraction.py` (새 파일 또는 기존 파일에 추가)

```python
# extraction.py ORM 섹션
from sqlalchemy import Column, String, Integer, Text, DateTime, ForeignKey, Tuple, LargeBinary
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from datetime import datetime
import uuid

Base = declarative_base()

class ExtractionSpanORM(Base):
    """
    추출 결과의 원문 위치 정보 저장.
    
    한 추출 작업(extraction_candidate) 은 여러 span 을 가질 수 있음 (다중 필드).
    각 span 은 (document_id, version_id, node_id, start_offset, end_offset) 로 고유 식별.
    """
    __tablename__ = 'extraction_spans'
    
    id = Column(String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    
    # FK: extraction_candidate 와의 관계
    extraction_id = Column(String(36), ForeignKey('extraction_candidates.id'), nullable=False)
    
    # 원문 위치 정보
    document_id = Column(String(36), nullable=False, index=True)
    document_version_id = Column(String(36), nullable=False)
    node_id = Column(String(255), nullable=True)  # 문서 계층 구조의 노드 ID
    
    # Span offset: 문자 단위 (0-based, end exclusive)
    span_start = Column(Integer, nullable=False)
    span_end = Column(Integer, nullable=False)
    
    # 실제 추출된 텍스트 저장 (검증 및 UI 표시용)
    source_text = Column(Text, nullable=False)
    
    # 내용 검증용 해시
    content_hash = Column(String(64), nullable=False, index=True)
    
    # 필드 메타데이터
    field_name = Column(String(255), nullable=False, index=True)  # 어느 필드의 span 인가?
    
    # 타임스탬프
    created_at = Column(DateTime, default=datetime.utcnow, nullable=False)
    
    # Indexes
    __table_args__ = (
        Index('ix_extraction_spans_document_version', 'document_id', 'document_version_id'),
        Index('ix_extraction_spans_extraction_field', 'extraction_id', 'field_name'),
        Index('ix_extraction_spans_content_hash', 'content_hash'),
    )
    
    # Relationship
    extraction_candidate = relationship(
        "ExtractionCandidateORM",
        back_populates="spans"
    )


# ExtractionCandidateORM 에 relationship 추가
# extraction_candidate.py 에서 수정:
# class ExtractionCandidateORM(Base):
#     ...
#     spans = relationship("ExtractionSpanORM", back_populates="extraction_candidate", cascade="all, delete-orphan")
```

### 4.4 Alembic 마이그레이션

**파일**: `/mnt/mimir/src/core/db/migrations/versions/008_create_extraction_spans_table.py` (신규)

```python
"""Create extraction_spans table for source span attribution.

Revision ID: 008
Revises: 007
Create Date: 2026-04-17 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa

revision = '008'
down_revision = '007'
branch_labels = None
depends_on = None

def upgrade():
    """Create extraction_spans table and indexes."""
    op.create_table(
        'extraction_spans',
        sa.Column('id', sa.String(36), nullable=False),
        sa.Column('extraction_id', sa.String(36), nullable=False),
        sa.Column('document_id', sa.String(36), nullable=False),
        sa.Column('document_version_id', sa.String(36), nullable=False),
        sa.Column('node_id', sa.String(255), nullable=True),
        sa.Column('span_start', sa.Integer, nullable=False),
        sa.Column('span_end', sa.Integer, nullable=False),
        sa.Column('source_text', sa.Text, nullable=False),
        sa.Column('content_hash', sa.String(64), nullable=False),
        sa.Column('field_name', sa.String(255), nullable=False),
        sa.Column('created_at', sa.DateTime, nullable=False, server_default=sa.func.now()),
        sa.PrimaryKeyConstraint('id'),
        sa.ForeignKeyConstraint(['extraction_id'], ['extraction_candidates.id'], ),
    )
    
    # Create indexes
    op.create_index(
        'ix_extraction_spans_document_version',
        'extraction_spans',
        ['document_id', 'document_version_id']
    )
    op.create_index(
        'ix_extraction_spans_extraction_field',
        'extraction_spans',
        ['extraction_id', 'field_name']
    )
    op.create_index(
        'ix_extraction_spans_content_hash',
        'extraction_spans',
        ['content_hash']
    )


def downgrade():
    """Drop extraction_spans table."""
    op.drop_table('extraction_spans')
```

### 4.5 Span 계산 서비스

**파일**: `/mnt/mimir/src/services/span_calculator.py` (신규)

```python
"""
Span 계산 및 검증 서비스.

LLM 응답에서 위치 힌트를 파싱하고, 문서의 실제 텍스트 기반으로 
정확한 character offset 을 계산합니다.
"""

import hashlib
import re
from typing import List, Tuple, Optional, Dict
from pydantic import ValidationError
from src.core.models.extraction import SourceSpan

class SpanNotFoundError(Exception):
    """요청한 텍스트를 문서에서 찾을 수 없음"""
    pass

class SpanValidationError(Exception):
    """Span 검증 실패"""
    pass

class SpanCalculator:
    """
    문서 내용과 LLM 응답을 기반으로 정확한 span offset 을 계산.
    """
    
    @staticmethod
    def calculate_content_hash(text: str) -> str:
        """텍스트의 SHA256 해시 계산"""
        return hashlib.sha256(text.encode('utf-8')).hexdigest()
    
    @staticmethod
    def find_text_in_document(
        document_content: str,
        search_text: str,
        start_search_pos: int = 0,
        case_sensitive: bool = False
    ) -> Optional[Tuple[int, int]]:
        """
        문서에서 텍스트 찾기.
        
        Args:
            document_content: 전체 문서 텍스트
            search_text: 찾을 텍스트
            start_search_pos: 검색 시작 위치
            case_sensitive: 대소문자 구분 여부
        
        Returns:
            (start, end) 또는 None
        """
        if case_sensitive:
            search_pos = document_content.find(search_text, start_search_pos)
        else:
            search_pos = document_content.lower().find(
                search_text.lower(),
                start_search_pos
            )
        
        if search_pos == -1:
            return None
        
        end_pos = search_pos + len(search_text)
        return (search_pos, end_pos)
    
    @staticmethod
    def extract_span_from_hint(
        document_content: str,
        location_hint: str,  # 예: "단락 3의 4-10번째 문자" 또는 "page 2, paragraph 4"
        fallback_text: Optional[str] = None
    ) -> Optional[Tuple[int, int]]:
        """
        LLM 응답의 위치 힌트에서 span 추출.
        
        여러 형식 지원:
        - "단락 3의 4-10번째 문자"
        - "Paragraph 2, character 15-30"
        - "Section 1.2, index 50-100"
        
        Args:
            document_content: 전체 문서 텍스트
            location_hint: LLM 에서 제공한 위치 힌트
            fallback_text: span 을 찾을 수 없을 때, 이 텍스트로 폴백
        
        Returns:
            (start, end) 또는 None
        """
        # 한국어 형식: "단락 N의 M-K번째 문자"
        kr_pattern = r'단락\s*(\d+)의\s*(\d+)[~-](\d+)번째\s*문자'
        kr_match = re.search(kr_pattern, location_hint)
        if kr_match:
            # 단락 번호를 기반으로 계산하는 로직 (간단한 예)
            # 실제로는 문서 구조 정보가 필요할 수 있음
            para_num = int(kr_match.group(1))
            start_char = int(kr_match.group(2))
            end_char = int(kr_match.group(3))
            # 단락 오프셋 계산 (모든 단락이 동일 길이라고 가정 - 실제로는 더 복잡)
            # 이 부분은 실제 문서 구조에 맞게 커스터마이징 필요
            return (start_char - 1, end_char)  # 1-based -> 0-based
        
        # 영어 형식: "Paragraph N, character M-K"
        en_pattern = r'(?:paragraph|section|page)\s+(\S+),\s*character\s+(\d+)[~-](\d+)'
        en_match = re.search(en_pattern, location_hint, re.IGNORECASE)
        if en_match:
            start_char = int(en_match.group(2))
            end_char = int(en_match.group(3))
            return (start_char - 1, end_char)  # 1-based -> 0-based
        
        # 숫자만 있는 형식: "123-456"
        num_pattern = r'(\d+)[~-](\d+)'
        num_match = re.search(num_pattern, location_hint)
        if num_match:
            start = int(num_match.group(1))
            end = int(num_match.group(2))
            return (min(start, end), max(start, end))
        
        # 폴백: fallback_text 로 검색
        if fallback_text:
            span = SpanCalculator.find_text_in_document(
                document_content, fallback_text
            )
            return span
        
        return None
    
    @staticmethod
    def verify_span_text(
        document_content: str,
        span: Tuple[int, int],
        expected_text: str
    ) -> bool:
        """
        Span 이 올바른 텍스트를 가리키는지 검증.
        
        Args:
            document_content: 전체 문서
            span: (start, end) offset
            expected_text: 예상되는 텍스트
        
        Returns:
            True if match, False otherwise
        """
        start, end = span
        actual_text = document_content[start:end]
        # Whitespace 정규화를 통한 fuzzy match
        actual_normalized = ' '.join(actual_text.split())
        expected_normalized = ' '.join(expected_text.split())
        return actual_normalized == expected_normalized
    
    @staticmethod
    def merge_overlapping_spans(spans: List[Tuple[int, int]]) -> List[Tuple[int, int]]:
        """
        겹치는 span 병합.
        
        예: [(10, 20), (15, 30)] -> [(10, 30)]
        """
        if not spans:
            return []
        
        sorted_spans = sorted(spans, key=lambda x: x[0])
        merged = [sorted_spans[0]]
        
        for current in sorted_spans[1:]:
            last_start, last_end = merged[-1]
            curr_start, curr_end = current
            
            if curr_start <= last_end:
                # 겹침: 병합
                merged[-1] = (last_start, max(last_end, curr_end))
            else:
                # 겹치지 않음: 추가
                merged.append(current)
        
        return merged
    
    @staticmethod
    def create_source_span(
        document_id: str,
        document_version_id: str,
        document_content: str,
        span_offset: Tuple[int, int],
        node_id: Optional[str] = None
    ) -> SourceSpan:
        """
        SourceSpan 객체 생성 및 검증.
        
        Args:
            document_id: 문서 ID
            document_version_id: 문서 버전 ID
            document_content: 전체 문서 텍스트
            span_offset: (start, end) offset
            node_id: 선택적 노드 ID
        
        Returns:
            검증된 SourceSpan 객체
        
        Raises:
            SpanValidationError: span 이 유효하지 않음
        """
        start, end = span_offset
        
        # Offset 범위 검증
        if start < 0 or end > len(document_content) or start >= end:
            raise SpanValidationError(
                f"Invalid span offset ({start}, {end}) for document of length {len(document_content)}"
            )
        
        # 실제 텍스트 추출
        source_text = document_content[start:end]
        
        # 해시 계산
        content_hash = SpanCalculator.calculate_content_hash(source_text)
        
        # SourceSpan 생성
        try:
            span = SourceSpan(
                document_id=document_id,
                version_id=document_version_id,
                node_id=node_id,
                span_offset=(start, end),
                source_text=source_text,
                content_hash=content_hash
            )
            return span
        except ValidationError as e:
            raise SpanValidationError(f"Failed to create SourceSpan: {e}")


class MultiSpanExtractor:
    """
    여러 개의 span 을 추출하는 고급 기능.
    
    예: 단일 필드가 문서의 여러 위치에서 참조될 경우 처리.
    """
    
    @staticmethod
    def extract_all_matching_spans(
        document_content: str,
        pattern: str,  # regex
        case_sensitive: bool = False
    ) -> List[Tuple[int, int]]:
        """
        정규식 패턴으로 모든 일치 span 추출.
        """
        flags = 0 if case_sensitive else re.IGNORECASE
        spans = []
        for match in re.finditer(pattern, document_content, flags):
            spans.append((match.start(), match.end()))
        return spans
    
    @staticmethod
    def merge_and_filter_spans(
        spans: List[Tuple[int, int]],
        min_span_length: int = 1,
        max_span_length: Optional[int] = None
    ) -> List[Tuple[int, int]]:
        """
        Span 필터링 및 병합.
        
        Args:
            spans: 필터링할 span 리스트
            min_span_length: 최소 span 길이 (문자)
            max_span_length: 최대 span 길이 (문자)
        
        Returns:
            필터링되고 병합된 span 리스트
        """
        filtered = []
        for start, end in spans:
            length = end - start
            if min_span_length <= length <= (max_span_length or float('inf')):
                filtered.append((start, end))
        
        return SpanCalculator.merge_overlapping_spans(filtered)
```

### 4.6 LLM 프롬프트 개선

**파일**: `/mnt/mimir/src/services/extraction_prompt_builder.py` (수정)

```python
"""
추출 프롬프트 생성기. S1 원칙 ① 적용: DocumentType 기반 동적 프롬프트.
"""

class ExtractionPromptBuilder:
    """
    DocumentType 별로 커스터마이즈된 추출 프롬프트 생성.
    """
    
    @staticmethod
    def build_extraction_prompt(
        document_type: str,  # 하드코딩 금지 - config 에서 가져올 것
        schema_config: dict,  # 추출 스키마 정의
        include_span_hints: bool = True
    ) -> str:
        """
        추출 프롬프트 생성.
        
        Args:
            document_type: 문서 타입 (config 에서 정의)
            schema_config: 스키마 설정 (필드, 타입, 필수여부 등)
            include_span_hints: span 위치 힌트 포함 여부
        
        Returns:
            완성된 프롬프트 텍스트
        """
        
        base_prompt = f"""
당신은 {document_type} 문서에서 정보를 추출하는 전문가입니다.

다음 문서에서 요청된 필드들을 추출해주세요.

## 추출해야 할 필드:
"""
        
        # 스키마 정의에서 필드 추가 (하드코딩 대신 config 사용)
        for field in schema_config.get('fields', []):
            field_name = field['name']
            field_type = field.get('type', 'string')
            required = field.get('required', False)
            description = field.get('description', '')
            
            req_marker = "[필수]" if required else "[선택]"
            base_prompt += f"\n- {req_marker} {field_name} ({field_type}): {description}"
        
        # Span 힌트 추가 (FG8.3 핵심)
        if include_span_hints:
            base_prompt += """

## 중요: 원문 위치 명시
반드시 각 추출한 값에 대해 원문에서의 위치를 다음 중 하나의 형식으로 명시해주세요:
- 한국어: "단락 3의 10-25번째 문자"
- English: "Paragraph 2, character 15-30"
- 또는 정확한 인덱스 범위: "123-456"

**예시:**
```json
{
  "company_name": {
    "value": "Acme Corporation",
    "location_hint": "단락 1의 5-21번째 문자"
  },
  "founded_year": {
    "value": 2010,
    "location_hint": "Paragraph 3, character 45-49"
  }
}
```
"""
        
        base_prompt += """

## 응답 형식
JSON 형식으로 응답하세요. 각 필드는 "field_name": "value" 형식이거나,
span 정보가 필요한 경우 { "value": ..., "location_hint": "..." } 형식으로 응답하세요.
"""
        
        return base_prompt
    
    @staticmethod
    def parse_extraction_response_with_spans(
        response_text: str,
        span_calculator
    ) -> dict:
        """
        LLM 응답에서 값과 span 힌트를 추출.
        
        Returns:
            {
                "field_name": {
                    "value": extracted_value,
                    "location_hint": hint_text,
                    "span_offset": (start, end)  # 계산된 offset
                }
            }
        """
        import json
        
        try:
            data = json.loads(response_text)
        except json.JSONDecodeError:
            # JSON 파싱 실패 - 원문 처리
            return {}
        
        result = {}
        for field_name, content in data.items():
            if isinstance(content, dict) and 'value' in content:
                # span 힌트가 포함된 형식
                value = content['value']
                location_hint = content.get('location_hint', '')
                result[field_name] = {
                    'value': value,
                    'location_hint': location_hint,
                    'span_offset': None  # 아직 문서 없음 - 나중에 계산
                }
            else:
                # 단순 값
                result[field_name] = {
                    'value': content,
                    'location_hint': None,
                    'span_offset': None
                }
        
        return result
```

### 4.7 Span 시각화 유틸리티

**파일**: `/mnt/mimir/src/utils/span_visualization.py` (신규)

```python
"""
UI 용 span 시각화 데이터 생성.
"""

from typing import List, Dict
from src.core.models.extraction import SourceSpan, ExtractedFieldWithAttribution

class SpanVisualizationConverter:
    """
    SourceSpan 을 UI 하이라이트 형식으로 변환.
    """
    
    @staticmethod
    def convert_span_to_highlight(
        span: SourceSpan,
        field_name: str,
        color_map: Dict[str, str] = None
    ) -> Dict:
        """
        단일 span 을 하이라이트 데이터로 변환.
        
        Returns:
            {
                "start": int,
                "end": int,
                "field_name": str,
                "text": str,
                "color": str
            }
        """
        if color_map is None:
            color_map = {}
        
        start, end = span.span_offset
        color = color_map.get(field_name, "#FFFF00")  # 기본값: 노란색
        
        return {
            "start": start,
            "end": end,
            "field_name": field_name,
            "text": span.source_text,
            "color": color
        }
    
    @staticmethod
    def convert_field_to_highlights(
        field: ExtractedFieldWithAttribution,
        color_map: Dict[str, str] = None
    ) -> List[Dict]:
        """
        ExtractedFieldWithAttribution 의 모든 span 을 하이라이트로 변환.
        """
        highlights = []
        for span in field.source_spans:
            highlight = SpanVisualizationConverter.convert_span_to_highlight(
                span, field.field_name, color_map
            )
            highlights.append(highlight)
        return highlights
    
    @staticmethod
    def batch_convert_to_highlights(
        fields: List[ExtractedFieldWithAttribution],
        color_map: Dict[str, str] = None
    ) -> List[Dict]:
        """
        여러 필드를 일괄 변환.
        """
        all_highlights = []
        for field in fields:
            highlights = SpanVisualizationConverter.convert_field_to_highlights(
                field, color_map
            )
            all_highlights.extend(highlights)
        
        # 겹치는 하이라이트 처리 (옵션: 우선순위 기반 merge)
        return all_highlights
```

### 4.8 통합 테스트

**파일**: `/mnt/mimir/tests/test_span_calculator.py` (신규)

```python
"""
Span 계산 및 검증 테스트.
"""

import pytest
from src.services.span_calculator import (
    SpanCalculator, MultiSpanExtractor, SpanValidationError, SpanNotFoundError
)
from src.core.models.extraction import SourceSpan


class TestSpanCalculator:
    """SpanCalculator 단위 테스트"""
    
    def test_calculate_content_hash(self):
        """해시 계산 정확성"""
        text = "The quick brown fox"
        hash1 = SpanCalculator.calculate_content_hash(text)
        hash2 = SpanCalculator.calculate_content_hash(text)
        assert hash1 == hash2
        assert len(hash1) == 64  # SHA256 = 64 hex chars
        
        # 다른 텍스트는 다른 해시
        hash3 = SpanCalculator.calculate_content_hash("Different text")
        assert hash1 != hash3
    
    def test_find_text_in_document_exact(self):
        """정확한 텍스트 검색"""
        doc = "The quick brown fox jumps over the lazy dog"
        result = SpanCalculator.find_text_in_document(doc, "brown fox")
        assert result == (10, 19)
    
    def test_find_text_in_document_case_insensitive(self):
        """대소문자 무시 검색"""
        doc = "The Quick Brown Fox"
        result = SpanCalculator.find_text_in_document(
            doc, "quick brown", case_sensitive=False
        )
        assert result == (4, 15)
    
    def test_find_text_not_found(self):
        """텍스트 미발견"""
        doc = "Hello world"
        result = SpanCalculator.find_text_in_document(doc, "xyz")
        assert result is None
    
    def test_extract_span_from_hint_korean_format(self):
        """한국어 위치 힌트 파싱"""
        doc = "단락 1 내용. 단락 2 중요한 정보입니다."
        hint = "단락 1의 4-10번째 문자"
        # 실제 동작은 문서 구조에 따라 달라지므로
        # 여기서는 숫자 파싱만 검증
        span = SpanCalculator.extract_span_from_hint(doc, hint)
        assert span is not None
    
    def test_extract_span_from_hint_fallback(self):
        """폴백 텍스트로 span 계산"""
        doc = "The quick brown fox"
        hint = "invalid_hint_format"
        fallback = "quick brown"
        span = SpanCalculator.extract_span_from_hint(
            doc, hint, fallback_text=fallback
        )
        assert span == (4, 15)
    
    def test_verify_span_text_exact_match(self):
        """Span 텍스트 검증 - 정확 일치"""
        doc = "The quick brown fox"
        span = (4, 9)
        expected = "quick"
        assert SpanCalculator.verify_span_text(doc, span, expected)
    
    def test_verify_span_text_fuzzy_match(self):
        """Span 텍스트 검증 - 공백 무시"""
        doc = "The  quick  brown  fox"  # 여러 공백
        span = (4, 15)
        expected = "quick brown"  # 단일 공백
        assert SpanCalculator.verify_span_text(doc, span, expected)
    
    def test_merge_overlapping_spans(self):
        """겹치는 span 병합"""
        spans = [(10, 20), (15, 30), (5, 8)]
        merged = SpanCalculator.merge_overlapping_spans(spans)
        assert merged == [(5, 8), (10, 30)]
    
    def test_create_source_span_valid(self):
        """SourceSpan 생성 - 유효한 경우"""
        doc = "Hello World"
        span = SpanCalculator.create_source_span(
            document_id="doc_123",
            document_version_id="v1",
            document_content=doc,
            span_offset=(0, 5),
            node_id="section_1"
        )
        
        assert span.document_id == "doc_123"
        assert span.source_text == "Hello"
        assert len(span.content_hash) == 64
    
    def test_create_source_span_invalid_offset(self):
        """SourceSpan 생성 - 잘못된 offset"""
        doc = "Hello World"
        
        with pytest.raises(SpanValidationError):
            SpanCalculator.create_source_span(
                document_id="doc_123",
                document_version_id="v1",
                document_content=doc,
                span_offset=(10, 5),  # start > end
                node_id="section_1"
            )
    
    def test_create_source_span_out_of_bounds(self):
        """SourceSpan 생성 - 범위 초과"""
        doc = "Hello World"
        
        with pytest.raises(SpanValidationError):
            SpanCalculator.create_source_span(
                document_id="doc_123",
                document_version_id="v1",
                document_content=doc,
                span_offset=(0, 100),  # end > document length
                node_id="section_1"
            )


class TestMultiSpanExtractor:
    """MultiSpanExtractor 테스트"""
    
    def test_extract_all_matching_spans(self):
        """정규식으로 모든 span 추출"""
        doc = "apple banana apple cherry apple"
        pattern = r'\bapple\b'
        spans = MultiSpanExtractor.extract_all_matching_spans(doc, pattern)
        
        assert len(spans) == 3
        assert spans[0] == (0, 5)
        assert spans[1] == (13, 18)
        assert spans[2] == (26, 31)
    
    def test_merge_and_filter_spans_by_length(self):
        """길이 기반 span 필터링"""
        spans = [(0, 2), (5, 15), (20, 22), (30, 45)]
        filtered = MultiSpanExtractor.merge_and_filter_spans(
            spans, min_span_length=5, max_span_length=20
        )
        
        assert len(filtered) == 2  # (5, 15) and (30, 45)


class TestIntegration:
    """통합 테스트"""
    
    def test_full_span_extraction_workflow(self):
        """전체 span 추출 워크플로우"""
        document_id = "doc_001"
        version_id = "v1"
        doc_content = """
        Company: Acme Corporation
        Founded: 2010
        Location: New York, USA
        """
        
        # 텍스트 검색
        company_span = SpanCalculator.find_text_in_document(
            doc_content, "Acme Corporation"
        )
        assert company_span is not None
        
        # SourceSpan 생성
        source_span = SpanCalculator.create_source_span(
            document_id=document_id,
            document_version_id=version_id,
            document_content=doc_content,
            span_offset=company_span,
            node_id="paragraph_1"
        )
        
        # 검증
        assert source_span.source_text == "Acme Corporation"
        assert source_span.content_hash == SpanCalculator.calculate_content_hash("Acme Corporation")
        
        # 다중 span 병합
        spans = [(company_span[0], company_span[1]), (company_span[0], company_span[1])]
        merged = SpanCalculator.merge_overlapping_spans(spans)
        assert len(merged) == 1
```

### 4.9 API 엔드포인트 (preview)

**파일**: `/mnt/mimir/src/api/routes/extraction.py` (추가)

```python
"""
추출 관련 API 엔드포인트.

Note: 전체 구현은 task8-9 에서 수행. 여기서는 span 관련 부분만 preview.
"""

from fastapi import APIRouter, HTTPException
from src.core.models.extraction import ExtractedFieldWithAttribution, SourceSpan
from src.utils.span_visualization import SpanVisualizationConverter

router = APIRouter(prefix="/api/v1/extractions", tags=["extractions"])

@router.get("/{extraction_id}/spans")
async def get_extraction_spans(extraction_id: str):
    """
    추출 결과의 모든 span 조회.
    """
    # 실제 구현: DB 에서 extraction_id 관련 span 조회
    pass

@router.get("/{extraction_id}/highlights")
async def get_extraction_highlights(extraction_id: str):
    """
    UI 용 하이라이트 데이터.
    """
    # DB 에서 span 조회
    spans = []  # query results
    
    # 시각화 데이터로 변환
    highlights = SpanVisualizationConverter.batch_convert_to_highlights(spans)
    
    return {"highlights": highlights}
```

---

## 5. 산출물

### 5.1 코드 파일

| 경로 | 설명 | 상태 |
|------|------|------|
| `/mnt/mimir/src/core/models/extraction.py` | SourceSpan, ExtractedFieldWithAttribution Pydantic 모델 | 신규 |
| `/mnt/mimir/src/core/db/models/extraction.py` | ExtractionSpanORM SQLAlchemy 모델 | 신규 |
| `/mnt/mimir/src/core/db/migrations/versions/008_create_extraction_spans_table.py` | Alembic 마이그레이션 | 신규 |
| `/mnt/mimir/src/services/span_calculator.py` | SpanCalculator, MultiSpanExtractor 서비스 | 신규 |
| `/mnt/mimir/src/services/extraction_prompt_builder.py` | LLM 프롬프트 생성기 (span 힌트 추가) | 수정 |
| `/mnt/mimir/src/utils/span_visualization.py` | Span 시각화 유틸리티 | 신규 |
| `/mnt/mimir/src/api/routes/extraction.py` | API 엔드포인트 (preview) | 수정 |
| `/mnt/mimir/tests/test_span_calculator.py` | 단위 + 통합 테스트 | 신규 |

### 5.2 설정 파일

| 파일 | 내용 |
|------|------|
| `.env.example` | `SPAN_CALCULATION_MODE` (exact/fuzzy), `SPAN_HASH_ALGORITHM` (SHA256) |

### 5.3 문서

| 파일 | 설명 |
|------|------|
| `/mnt/mimir/docs/technical/span_specification.md` | Span 오프셋 명세, 다국어 지원 |
| `/mnt/mimir/docs/technical/llm_prompt_enhancement.md` | LLM 프롬프트 개선사항 |

---

## 6. 완료 기준

1. ✓ SourceSpan Pydantic 모델 구현 + validation
2. ✓ ExtractedFieldWithAttribution 모델 구현 + 다중 span 지원
3. ✓ ExtractionSpanORM SQLAlchemy 모델 + 외래키 관계 정의
4. ✓ Alembic 마이그레이션 작성 및 테스트 실행
5. ✓ SpanCalculator 서비스 구현 (find, verify, merge, create)
6. ✓ LLM 프롬프트에 "위치 명시" 지시문 추가
7. ✓ SpanVisualizationConverter 구현
8. ✓ 단위 테스트: 최소 15개 (span calculation, validation, edge cases)
9. ✓ 통합 테스트: 전체 span 추출 워크플로우
10. ✓ 모든 모델에 JSON 스키마 예제 포함
11. ✓ 다국어 문자열 처리 검증 (UTF-8, 이모지 등)
12. ✓ Type hints 완전 구현 (모든 함수, 모든 매개변수)

---

## 7. 작업 지침

### 지침 7-1: S1 원칙 ① 준수 — DocumentType 하드코딩 금지

모든 document type 분기 로직은 **config 기반**이어야 합니다.

```python
# ❌ 잘못된 예
if document_type == "contract":
    # ...

# ✅ 올바른 예
span_config = CONFIG['span_calculation_modes'].get(document_type)
if span_config and span_config.get('enabled'):
    # ...
```

### 지침 7-2: S1 원칙 ② 구현 — Generic + Config 기반 Span 계산

SpanCalculator 는 특정 문서 타입에 종속적이지 않아야 합니다. 위치 힌트 파싱은 config 로 확장 가능하게 설계합니다.

```python
# SPAN_HINT_PATTERNS config 예시
SPAN_HINT_PATTERNS = {
    "korean": r"단락\s*(\d+)의\s*(\d+)[~-](\d+)번째\s*문자",
    "english": r"Paragraph\s+(\S+),\s*character\s+(\d+)[~-](\d+)",
    "custom_domain": r"section\s+([A-Z0-9]+)\s+line\s+(\d+)-(\d+)",
}
```

### 지침 7-3: Span Offset 다국어 처리

UTF-8 인코딩 문제 주의. 한글, 중국어, 이모지 등 multi-byte 문자 처리:

```python
# 올바른 방식: 문자 단위 (UTF-8 바이트가 아님)
doc = "안녕하세요. 반갑습니다."
# span_offset = (0, 5) → "안녕하세요"
actual = doc[0:5]  # Python 은 문자 단위 인덱싱
assert actual == "안녕하세요"
```

### 지침 7-4: Content Hash 검증 의무화

모든 SourceSpan 생성 시 content hash 검증을 거쳐야 합니다. DB 저장 전 무결성 확인:

```python
def create_source_span(...):
    source_text = document_content[start:end]
    computed_hash = calculate_content_hash(source_text)
    
    # span 객체의 hash 필드 검증
    if span.content_hash != computed_hash:
        raise SpanValidationError("Content hash mismatch")
```

### 지침 7-5: Phase 2 Citation 호환성 확인

Phase 2 에서 정의한 Citation 5-tuple 구조를 검토하고, SourceSpan 이 호환되도록 설계합니다.

- Citation: (document_id, version, position, text, confidence)
- SourceSpan: (document_id, version_id, node_id, span_offset, source_text, content_hash)

매핑: `position` → `span_offset`, `text` → `source_text`

### 지침 7-6: LLM 응답 파싱의 견고성

LLM 이 항상 위치 힌트를 정확히 제공하지 않을 수 있으므로, fallback 전략을 구현합니다:

1. 위치 힌트 파싱 시도
2. 실패 시 extracted_value 텍스트로 find 시도
3. fuzzy matching (공백, 구두점 무시)
4. 최후 수단: span 없이 값만 저장 (근거 부재 기록)

### 지침 7-7: 테스트 커버리지 최소 80%

모든 public 메서드에 대한 테스트 작성. 특히:

- Normal case (정상 입력)
- Edge case (빈 문자열, 초대형 문서, 다중 span)
- Error case (범위 초과, 잘못된 offset, hash 불일치)
- Integration case (전체 워크플로우)

pytest 로 실행 및 coverage report 생성:

```bash
pytest /mnt/mimir/tests/test_span_calculator.py -v --cov=src.services.span_calculator --cov-report=term-missing
```

### 지침 7-8: 에러 처리 표준화

Custom exception 정의:

```python
class SpanCalculationError(Exception):
    """Span 계산 중 발생한 일반 에러"""
    pass

class SpanValidationError(SpanCalculationError):
    """Span 검증 실패"""
    pass

class SpanNotFoundError(SpanCalculationError):
    """요청한 텍스트를 문서에서 찾을 수 없음"""
    pass
```

모든 함수에서 명시적으로 이 예외를 raise 하고, 호출자에서 적절히 처리합니다.

### 지침 7-9: 성능 최적화

- Document loading 후 span 계산은 O(n) 복잡도 유지
- Large document (> 10MB) 테스트 필수
- Regex 패턴 컴파일 시 캐싱 고려
- Multiple span 병합 시 정렬된 리스트 가정

### 지침 7-10: 문서화 요구사항

모든 클래스, 메서드에 docstring 작성:

```python
def find_text_in_document(
    document_content: str,
    search_text: str,
    start_search_pos: int = 0,
    case_sensitive: bool = False
) -> Optional[Tuple[int, int]]:
    """
    문서에서 텍스트를 검색하여 span offset 반환.
    
    Args:
        document_content: 전체 문서 텍스트 (UTF-8 문자열)
        search_text: 검색할 텍스트
        start_search_pos: 검색 시작 위치 (0-based, 기본값: 0)
        case_sensitive: True 이면 대소문자 구분, False 이면 무시 (기본값: False)
    
    Returns:
        (start, end) 튜플 (0-based, end exclusive) 또는 None (미발견)
    
    Raises:
        TypeError: document_content 또는 search_text 가 str 이 아님
    
    Examples:
        >>> find_text_in_document("Hello World", "World")
        (6, 11)
        
        >>> find_text_in_document("Hello WORLD", "world", case_sensitive=False)
        (6, 11)
        
        >>> find_text_in_document("Hello World", "xyz")
        None
    """
```

---

## 첨부: 데이터 모델 ER 다이어그램

```
ExtractionCandidateORM (Phase 5)
    │
    ├─ id
    ├─ document_id (FK → Document)
    ├─ schema_id (FK → ExtractionSchema)
    └─ spans ──────────┐
                       │
                       ↓
                ExtractionSpanORM (신규)
                    ├─ id
                    ├─ extraction_id (FK)
                    ├─ document_id
                    ├─ document_version_id
                    ├─ node_id
                    ├─ span_start
                    ├─ span_end
                    ├─ source_text
                    ├─ content_hash
                    ├─ field_name
                    └─ created_at

DocumentORM (Phase 2)
    │
    ├─ id
    ├─ content
    ├─ content_hash
    └─ versions ──┐
                  ↓
              DocumentVersionORM
                  ├─ id
                  ├─ document_id (FK)
                  ├─ version_number
                  └─ content_hash
```

---

**끝.**
