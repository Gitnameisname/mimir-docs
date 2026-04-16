# Task 8-1: ExtractionTargetSchema 도메인 모델 + DB 스키마

**작업 식별자**: FG8.1-task8-1  
**작업 단계**: Phase 8 - DocumentType metadata schema → 추출 타겟 스키마  
**예상 소요 시간**: 32-40시간  
**담당자**: [Backend/Database Engineer]  

---

## 1. 작업 목적

Phase 6에서 DocumentType의 metadata schema를 정의했으며, Phase 8 FG8.1에서는 이 metadata schema를 기반으로 **실제 추출 시 어떤 필드를 어떻게 추출할 것인지 정의하는 매핑 계층**을 제공한다. 

Task 8-1은 이 계층의 기초가 되는 도메인 모델 및 데이터베이스 스키마를 설계하고 구현하는 것을 목표로 한다:

1. **ExtractionTargetSchema** — DocumentType별 추출 대상 필드 및 그 제약조건을 정의하는 Pydantic 모델
2. **ExtractionFieldDef** — 각 필드의 타입, 필수 여부, 검증 패턴, 추출 지침, 예제 등을 정의하는 FieldDefinition 모델
3. **SQLAlchemy ORM 모델** — extraction_schemas, extraction_schema_versions 테이블 및 버전 관리 지원
4. **Alembic 마이그레이션** — 프로덕션 배포 대비 DB 마이그레이션 스크립트
5. **Repository 패턴** — ExtractionSchemaRepository로 데이터 접근 계층 추상화
6. **Scope Profile ACL** — 관리자가 정의한 Scope Profile 기반 접근 제어 통합
7. **Unit 테스트** — 도메인 로직 및 검증 규칙에 대한 광범위한 테스트

---

## 2. 작업 범위

### 포함 범위

#### 2.1 Pydantic 도메인 모델 정의
- **ExtractionFieldDef** 모델
  - field_name: str (필드명, 예: "invoice_number", "total_amount")
  - field_type: Literal["string", "number", "date", "boolean", "array", "object", "enum"] (필드 타입)
  - required: bool (필수 여부)
  - description: str (필드 설명)
  - pattern: Optional[str] (정규식 검증 패턴, e.g., r"^INV-\d{6}$")
  - instruction: Optional[str] (AI 추출 에이전트를 위한 지침)
  - examples: List[str] (예제 데이터, 최소 2개)
  - max_length: Optional[int] (문자열 최대 길이)
  - min_value: Optional[float] (숫자 최소값)
  - max_value: Optional[float] (숫자 최대값)
  - date_format: Optional[str] (날짜 포맷, e.g., "YYYY-MM-DD")
  - enum_values: Optional[List[str]] (enum 타입일 때 허용 값 리스트)
  - default_value: Optional[Any] (기본값)
  - nested_schema: Optional[Dict[str, "ExtractionFieldDef"]] (object 타입일 때 중첩 필드 스키마)

- **ExtractionTargetSchema** 모델
  - id: UUID (스키마 ID)
  - doc_type_id: UUID (DocumentType 참조, FK)
  - version: int (버전 번호, 1부터 시작)
  - fields: Dict[str, ExtractionFieldDef] (필드 정의 맵)
  - metadata: Dict[str, Any] (추가 메타데이터: created_at, updated_at, created_by, updated_by 등)
  - is_deprecated: bool (폐기 여부)
  - deprecation_reason: Optional[str]
  - created_at: datetime
  - updated_at: datetime
  - created_by: str (actor ID, user 또는 agent)
  - updated_by: str (actor ID)

- **FieldValidationRule** 모델
  - field_name: str
  - validators: List[str] (Literal 타입: "required", "pattern", "length", "range", "enum", "custom")
  - error_message: str
  - severity: Literal["warning", "error"] (위반 시 경고 또는 오류)

#### 2.2 SQLAlchemy ORM 모델
- **ExtractionSchemaORM**
  ```
  Table: extraction_schemas
  Columns:
    - id (UUID, PK)
    - doc_type_id (UUID, FK → document_types.id, NOT NULL, index)
    - version (INT, NOT NULL, default 1)
    - fields_json (JSONB, NOT NULL) — fields 직렬화
    - metadata_json (JSONB) — 메타데이터 직렬화
    - is_deprecated (BOOLEAN, default False)
    - deprecation_reason (TEXT)
    - created_at (TIMESTAMP, NOT NULL)
    - updated_at (TIMESTAMP, NOT NULL)
    - created_by (VARCHAR, NOT NULL) — actor ID
    - updated_by (VARCHAR, NOT NULL) — actor ID
    - scope_profile_id (UUID, FK → scope_profiles.id, nullable) — Scope Profile ACL
    - is_soft_deleted (BOOLEAN, default False)
    - deleted_at (TIMESTAMP, nullable)
    - deleted_by (VARCHAR, nullable)
  
  Indexes:
    - idx_doc_type_id
    - idx_doc_type_id_version (composite, UNIQUE where is_soft_deleted=False)
    - idx_scope_profile_id
    - idx_is_deprecated
  ```

- **ExtractionSchemaVersionORM**
  ```
  Table: extraction_schema_versions
  Columns:
    - id (UUID, PK)
    - schema_id (UUID, FK → extraction_schemas.id, NOT NULL, index)
    - version (INT, NOT NULL)
    - fields_json (JSONB, NOT NULL)
    - metadata_json (JSONB)
    - is_deprecated (BOOLEAN)
    - deprecation_reason (TEXT)
    - change_summary (TEXT) — 버전 간 변경사항 요약
    - changed_fields (JSONB) — 변경된 필드명 리스트
    - created_at (TIMESTAMP, NOT NULL)
    - created_by (VARCHAR, NOT NULL)
  
  Indexes:
    - idx_schema_id_version (UNIQUE)
    - idx_created_at
  ```

- **Relationship with DocumentType**
  - DocumentType에 `extraction_schemas` relationship 추가 (1:N, cascade delete 금지)
  - ExtractionSchemaORM에 `document_type` relationship 추가

#### 2.3 Alembic 마이그레이션
- **Migration 파일**: alembic/versions/{timestamp}_add_extraction_schema_tables.py
- 내용:
  1. extraction_schemas 테이블 생성
  2. extraction_schema_versions 테이블 생성
  3. DocumentType ← ExtractionSchema FK 관계 설정
  4. 인덱스 생성 (doc_type_id, version, scope_profile_id, deprecated 등)
  5. Rollback 로직 포함

#### 2.4 Repository 패턴 구현
- **ExtractionSchemaRepository** 클래스
  - create(doc_type_id, fields, scope_profile_id, actor_info) → ExtractionTargetSchema
  - get_by_doc_type(doc_type_id, scope_profile_id=None) → ExtractionTargetSchema (최신 버전)
  - get_by_doc_type_and_version(doc_type_id, version) → ExtractionTargetSchema
  - get_versions(doc_type_id, limit=10, offset=0) → List[ExtractionTargetSchema]
  - update(doc_type_id, fields, scope_profile_id, actor_info) → ExtractionTargetSchema (새 버전 생성)
  - delete(doc_type_id, actor_info) → bool (soft delete)
  - restore(doc_type_id, actor_info) → bool (soft delete 복구)
  - deprecate(doc_type_id, reason, actor_info) → ExtractionTargetSchema
  - list_all(is_deprecated=None, scope_profile_id=None, limit=100) → List[ExtractionTargetSchema]
  - search_by_field_name(field_name, scope_profile_id=None) → List[ExtractionTargetSchema]

  각 메서드는 **Scope Profile 기반 ACL 필터링** 적용:
  - scope_profile_id가 주어지면, 해당 프로필에 접근 권한이 있는 DocumentType만 반환
  - 권한 검증 로직: `_check_access_scope(doc_type_id, scope_profile_id)`

#### 2.5 Scope Profile ACL 통합
- **_check_access_scope(doc_type_id, scope_profile_id)** 메서드
  - ScopeProfileService를 호출하여 해당 scope_profile_id가 doc_type_id에 대한 접근 권한이 있는지 확인
  - 권한 없으면 ForbiddenError 발생
  - 권한 캐싱 고려 (레디스 등)

- **actor_type 필드** 통합
  - audit_logs 테이블 쿼리 또는 메타데이터에 actor_type ("user" | "agent") 기록
  - Repository 메서드 호출 시 actor_info 매개변수에 actor_type 포함:
    ```python
    actor_info = {"actor_id": user_id, "actor_type": "user"}  # or "agent"
    ```

#### 2.6 Soft Delete 지원
- is_soft_deleted, deleted_at, deleted_by 필드 관리
- Repository의 모든 조회 메서드는 기본적으로 is_soft_deleted=False 필터 적용
- 필요 시 include_deleted=True 매개변수로 삭제된 항목도 조회 가능

#### 2.7 단위 테스트 (Unit Tests)
- **test_extraction_field_def.py**
  - ExtractionFieldDef 생성 및 검증 (field_type별)
  - pattern 정규식 검증
  - enum_values 검증
  - nested_schema (object 타입)에서 재귀적 검증
  - 필드 제약조건 위반 시 예외 발생 확인

- **test_extraction_target_schema.py**
  - ExtractionTargetSchema 생성 및 조회
  - 버전 관리 로직 (version 자동 증가)
  - 필드 추가/수정/삭제 시 버전 증가
  - deprecated 상태 관리
  - JSON 직렬화/역직렬화

- **test_extraction_schema_repository.py**
  - create, get_by_doc_type, get_by_doc_type_and_version, get_versions 테스트
  - update 시 새 버전 생성 확인
  - delete/restore 테스트 (soft delete)
  - deprecate 테스트
  - search_by_field_name 테스트
  - Scope Profile ACL 필터링 테스트
    - scope_profile_id가 없을 때: 모든 스키마 조회
    - scope_profile_id가 있을 때: 해당 스코프에 속한 DocumentType만 조회
    - 권한 없을 때: ForbiddenError 발생
  - soft delete 후 list_all(include_deleted=False)에서 제외 확인
  - actor_info 전달 및 created_by, updated_by 기록 확인

---

### 제외 범위

- CRUD API 엔드포인트 구현 (task8-2에서 수행)
- 마이그레이션 로직 구현 (task8-3에서 수행)
- AI 에이전트 통합 (별도 작업)
- UI 구현 (별도 작업)
- 외부 스토리지(S3 등) 연동 (스코프 외)

---

## 3. 선행 조건

1. **Phase 6 완료**: DocumentType 도메인 모델 및 DB 스키마 이미 구현되어 있음
   - document_types 테이블 존재
   - document_type_metadata_schemas 테이블 존재

2. **ScopeProfileService** 구현 완료
   - `check_access(actor_id, scope_profile_id, resource_type, resource_id)` 메서드 제공
   - Scope Profile 정보 조회 기능

3. **Alembic 환경** 설정 완료
   - alembic/env.py, alembic/script.py.mako 기존 설정 검증

4. **데이터베이스 연결**
   - PostgreSQL 또는 호환 DB에 대한 SQLAlchemy Session 제공
   - JSONB 지원 (PostgreSQL 권장)

5. **Pydantic v2** 설치 (type hints, validation)

6. **테스트 환경**
   - pytest, pytest-asyncio 설치
   - 테스트용 임시 DB 스키마 격리

---

## 4. 주요 작업 항목

### 4.1 ExtractionFieldDef 모델 설계 및 구현

**파일**: `src/domain/extraction/models.py`

```python
from typing import Any, Dict, List, Optional, Literal
from datetime import datetime
from uuid import UUID
from pydantic import BaseModel, Field, field_validator, model_validator


class ExtractionFieldDef(BaseModel):
    """추출 대상 필드 정의"""
    
    field_name: str = Field(..., min_length=1, max_length=255, description="필드명 (snake_case)")
    field_type: Literal[
        "string", "number", "date", "boolean", "array", "object", "enum"
    ] = Field(..., description="필드 타입")
    required: bool = Field(default=False, description="필수 여부")
    description: str = Field(..., min_length=1, max_length=1024, description="필드 설명")
    
    # 검증 규칙
    pattern: Optional[str] = Field(
        default=None,
        description="정규식 패턴 (string 타입일 때)"
    )
    instruction: Optional[str] = Field(
        default=None,
        max_length=2048,
        description="AI 추출 에이전트를 위한 지침"
    )
    examples: List[str] = Field(
        default_factory=list,
        description="예제 값 (최소 2개 권장)"
    )
    
    # 크기/범위 제약
    max_length: Optional[int] = Field(
        default=None,
        ge=1,
        le=65536,
        description="문자열 최대 길이"
    )
    min_value: Optional[float] = Field(
        default=None,
        description="숫자 최소값"
    )
    max_value: Optional[float] = Field(
        default=None,
        description="숫자 최대값"
    )
    
    # 타입별 추가 정보
    date_format: Optional[str] = Field(
        default=None,
        description="날짜 포맷 (ISO 8601 기본, e.g., 'YYYY-MM-DD')"
    )
    enum_values: Optional[List[str]] = Field(
        default=None,
        description="enum 타입일 때 허용 값"
    )
    default_value: Optional[Any] = Field(
        default=None,
        description="기본값"
    )
    
    # 중첩 스키마 (object 타입)
    nested_schema: Optional[Dict[str, "ExtractionFieldDef"]] = Field(
        default=None,
        description="object 타입일 때 중첩 필드 정의"
    )
    
    model_config = {"json_schema_extra": {}}
    
    @field_validator("field_name")
    @classmethod
    def validate_field_name(cls, v: str) -> str:
        """필드명이 snake_case 형식인지 확인"""
        import re
        if not re.match(r"^[a-z][a-z0-9_]*$", v):
            raise ValueError("필드명은 snake_case 형식이어야 함")
        return v
    
    @field_validator("pattern", mode="before")
    @classmethod
    def validate_pattern(cls, v: Optional[str]) -> Optional[str]:
        """정규식 패턴 문법 검증"""
        if v is not None:
            import re
            try:
                re.compile(v)
            except re.error as e:
                raise ValueError(f"정규식 오류: {e}")
        return v
    
    @field_validator("min_value", "max_value", mode="before")
    @classmethod
    def validate_range(cls, v: Optional[float], info) -> Optional[float]:
        """min_value와 max_value의 순서 검증"""
        # 각각의 필드에서만 문법 검증, 관계 검증은 model_validator에서
        return v
    
    @model_validator(mode="after")
    def validate_field_type_consistency(self) -> "ExtractionFieldDef":
        """필드 타입과 속성의 일관성 검증"""
        
        # string 타입
        if self.field_type == "string":
            if self.enum_values is not None:
                raise ValueError("string 타입에서는 enum_values를 사용할 수 없음")
            if self.min_value is not None or self.max_value is not None:
                raise ValueError("string 타입에서는 min_value, max_value를 사용할 수 없음")
        
        # number 타입
        elif self.field_type == "number":
            if self.pattern is not None:
                raise ValueError("number 타입에서는 pattern을 사용할 수 없음")
            if self.max_length is not None:
                raise ValueError("number 타입에서는 max_length를 사용할 수 없음")
            if self.enum_values is not None:
                raise ValueError("number 타입에서는 enum_values를 사용할 수 없음")
            if self.min_value is not None and self.max_value is not None:
                if self.min_value > self.max_value:
                    raise ValueError("min_value는 max_value보다 작아야 함")
        
        # date 타입
        elif self.field_type == "date":
            if self.pattern is not None:
                raise ValueError("date 타입에서는 pattern을 사용할 수 없음")
            if self.min_value is not None or self.max_value is not None:
                raise ValueError("date 타입에서는 min_value, max_value를 사용할 수 없음")
            if self.enum_values is not None:
                raise ValueError("date 타입에서는 enum_values를 사용할 수 없음")
        
        # boolean 타입
        elif self.field_type == "boolean":
            if self.pattern is not None:
                raise ValueError("boolean 타입에서는 pattern을 사용할 수 없음")
            if self.max_length is not None:
                raise ValueError("boolean 타입에서는 max_length를 사용할 수 없음")
            if self.min_value is not None or self.max_value is not None:
                raise ValueError("boolean 타입에서는 min_value, max_value를 사용할 수 없음")
        
        # enum 타입
        elif self.field_type == "enum":
            if not self.enum_values or len(self.enum_values) == 0:
                raise ValueError("enum 타입에서는 enum_values가 필수임")
            if self.pattern is not None:
                raise ValueError("enum 타입에서는 pattern을 사용할 수 없음")
            if self.min_value is not None or self.max_value is not None:
                raise ValueError("enum 타입에서는 min_value, max_value를 사용할 수 없음")
        
        # array 타입
        elif self.field_type == "array":
            if self.pattern is not None:
                raise ValueError("array 타입에서는 pattern을 사용할 수 없음")
            if self.min_value is not None or self.max_value is not None:
                raise ValueError("array 타입에서는 min_value, max_value를 사용할 수 없음")
            if self.enum_values is not None:
                raise ValueError("array 타입에서는 enum_values를 사용할 수 없음")
        
        # object 타입
        elif self.field_type == "object":
            if not self.nested_schema or len(self.nested_schema) == 0:
                raise ValueError("object 타입에서는 nested_schema가 필수임")
            if self.pattern is not None:
                raise ValueError("object 타입에서는 pattern을 사용할 수 없음")
            if self.max_length is not None:
                raise ValueError("object 타입에서는 max_length를 사용할 수 없음")
            if self.min_value is not None or self.max_value is not None:
                raise ValueError("object 타입에서는 min_value, max_value를 사용할 수 없음")
        
        # 필수 필드 검증
        if self.required and self.default_value is not None:
            import warnings
            warnings.warn(
                f"필드 '{self.field_name}'가 required이면서 default_value를 가짐",
                UserWarning
            )
        
        return self
    
    @model_validator(mode="after")
    def validate_examples(self) -> "ExtractionFieldDef":
        """예제 데이터 검증"""
        if len(self.examples) > 0:
            # 최소 2개 권장 경고
            if len(self.examples) < 2:
                import warnings
                warnings.warn(
                    f"필드 '{self.field_name}'의 예제가 2개 미만임 (권장: 2개 이상)",
                    UserWarning
                )
            
            # 타입별 예제 검증
            if self.field_type == "number":
                for ex in self.examples:
                    try:
                        float(ex)
                    except ValueError:
                        raise ValueError(
                            f"number 타입의 예제 '{ex}'가 숫자로 변환 불가"
                        )
            
            elif self.field_type == "enum":
                for ex in self.examples:
                    if ex not in (self.enum_values or []):
                        raise ValueError(
                            f"enum 타입의 예제 '{ex}'가 enum_values에 없음"
                        )
            
            elif self.field_type == "date":
                for ex in self.examples:
                    # 간단한 ISO 날짜 형식 검증
                    if not self._is_valid_date(ex, self.date_format or "YYYY-MM-DD"):
                        raise ValueError(
                            f"date 타입의 예제 '{ex}'가 형식 '{self.date_format}'과 맞지 않음"
                        )
        
        return self
    
    @staticmethod
    def _is_valid_date(value: str, date_format: str) -> bool:
        """날짜 형식 검증"""
        from datetime import datetime
        try:
            # YYYY-MM-DD → %Y-%m-%d 변환
            py_format = date_format.replace("YYYY", "%Y").replace("MM", "%m").replace("DD", "%d")
            datetime.strptime(value, py_format)
            return True
        except ValueError:
            return False


class ExtractionTargetSchema(BaseModel):
    """DocumentType별 추출 대상 스키마"""
    
    id: UUID = Field(..., description="스키마 ID")
    doc_type_id: UUID = Field(..., description="DocumentType 참조")
    version: int = Field(default=1, ge=1, description="버전 번호")
    
    fields: Dict[str, ExtractionFieldDef] = Field(
        default_factory=dict,
        description="필드 정의 맵"
    )
    
    # 메타데이터
    is_deprecated: bool = Field(default=False, description="폐기 여부")
    deprecation_reason: Optional[str] = Field(
        default=None,
        max_length=1024,
        description="폐기 사유"
    )
    
    # 감사 정보 (actor_type 포함)
    created_at: datetime = Field(..., description="생성 시각")
    updated_at: datetime = Field(..., description="수정 시각")
    created_by: str = Field(..., description="생성자 ID (user 또는 agent)")
    updated_by: str = Field(..., description="수정자 ID (user 또는 agent)")
    
    metadata: Dict[str, Any] = Field(
        default_factory=dict,
        description="추가 메타데이터"
    )
    
    model_config = {"json_schema_extra": {}}
    
    @model_validator(mode="after")
    def validate_deprecation(self) -> "ExtractionTargetSchema":
        """폐기 상태 검증"""
        if self.is_deprecated and not self.deprecation_reason:
            raise ValueError("is_deprecated=True일 때 deprecation_reason이 필수")
        if not self.is_deprecated and self.deprecation_reason:
            raise ValueError("is_deprecated=False일 때 deprecation_reason을 설정할 수 없음")
        return self
    
    @model_validator(mode="after")
    def validate_fields_not_empty(self) -> "ExtractionTargetSchema":
        """최소 하나의 필드 정의 필수"""
        if len(self.fields) == 0:
            raise ValueError("추출 스키마는 최소 1개의 필드를 포함해야 함")
        return self


class FieldValidationRule(BaseModel):
    """필드 검증 규칙"""
    
    field_name: str = Field(..., description="필드명")
    validators: List[Literal[
        "required", "pattern", "length", "range", "enum", "custom"
    ]] = Field(..., description="적용할 검증자")
    error_message: str = Field(..., description="위반 시 오류 메시지")
    severity: Literal["warning", "error"] = Field(
        default="error",
        description="위반 시 경고/오류"
    )
```

### 4.2 SQLAlchemy ORM 모델 구현

**파일**: `src/infrastructure/database/models/extraction.py`

```python
from datetime import datetime
from typing import Any, Dict, Optional
from uuid import UUID, uuid4

from sqlalchemy import (
    Boolean,
    Column,
    DateTime,
    ForeignKey,
    Index,
    Integer,
    String,
    Text,
    UniqueConstraint,
    func,
)
from sqlalchemy.dialects.postgresql import JSONB, UUID as PG_UUID
from sqlalchemy.orm import relationship

from src.infrastructure.database.base import Base


class ExtractionSchemaORM(Base):
    """추출 대상 스키마 ORM 모델"""
    
    __tablename__ = "extraction_schemas"
    
    id = Column(PG_UUID(as_uuid=True), primary_key=True, default=uuid4)
    doc_type_id = Column(
        PG_UUID(as_uuid=True),
        ForeignKey("document_types.id", ondelete="RESTRICT"),
        nullable=False,
        index=True
    )
    version = Column(Integer, nullable=False, default=1)
    
    # JSON 필드들
    fields_json = Column(JSONB, nullable=False)  # Dict[str, ExtractionFieldDef]
    metadata_json = Column(JSONB, nullable=True)
    
    # 폐기 정보
    is_deprecated = Column(Boolean, nullable=False, default=False, index=True)
    deprecation_reason = Column(Text, nullable=True)
    
    # 감사 정보 (actor_type 포함)
    created_at = Column(DateTime(timezone=True), nullable=False, default=func.now())
    updated_at = Column(DateTime(timezone=True), nullable=False, default=func.now(), onupdate=func.now())
    created_by = Column(String(255), nullable=False)  # actor ID
    updated_by = Column(String(255), nullable=False)  # actor ID
    
    # Scope Profile ACL
    scope_profile_id = Column(
        PG_UUID(as_uuid=True),
        ForeignKey("scope_profiles.id", ondelete="SET NULL"),
        nullable=True,
        index=True
    )
    
    # Soft delete
    is_soft_deleted = Column(Boolean, nullable=False, default=False)
    deleted_at = Column(DateTime(timezone=True), nullable=True)
    deleted_by = Column(String(255), nullable=True)
    
    # 관계
    document_type = relationship(
        "DocumentTypeORM",
        back_populates="extraction_schemas",
        foreign_keys=[doc_type_id]
    )
    
    versions = relationship(
        "ExtractionSchemaVersionORM",
        back_populates="schema",
        cascade="all, delete-orphan",
        foreign_keys="ExtractionSchemaVersionORM.schema_id"
    )
    
    # 인덱스
    __table_args__ = (
        Index("idx_doc_type_version", "doc_type_id", "version"),
        UniqueConstraint(
            "doc_type_id", "version",
            name="uq_doc_type_version",
            # PostgreSQL 부분 인덱스: soft delete 제외
            postgresql_where=~is_soft_deleted
        ),
        Index("idx_is_deprecated", "is_deprecated"),
    )
    
    def to_pydantic(self) -> "ExtractionTargetSchema":
        """ORM 모델을 Pydantic 모델로 변환"""
        from src.domain.extraction.models import ExtractionTargetSchema, ExtractionFieldDef
        
        # fields_json에서 ExtractionFieldDef 복원
        fields_dict = {}
        for field_name, field_def in (self.fields_json or {}).items():
            fields_dict[field_name] = ExtractionFieldDef(**field_def)
        
        return ExtractionTargetSchema(
            id=self.id,
            doc_type_id=self.doc_type_id,
            version=self.version,
            fields=fields_dict,
            is_deprecated=self.is_deprecated,
            deprecation_reason=self.deprecation_reason,
            created_at=self.created_at,
            updated_at=self.updated_at,
            created_by=self.created_by,
            updated_by=self.updated_by,
            metadata=self.metadata_json or {},
        )
    
    @staticmethod
    def from_pydantic(
        schema: "ExtractionTargetSchema",
        scope_profile_id: Optional[UUID] = None
    ) -> "ExtractionSchemaORM":
        """Pydantic 모델에서 ORM 모델 생성"""
        # fields를 JSON 직렬화
        fields_json = {
            field_name: field_def.model_dump()
            for field_name, field_def in schema.fields.items()
        }
        
        return ExtractionSchemaORM(
            id=schema.id,
            doc_type_id=schema.doc_type_id,
            version=schema.version,
            fields_json=fields_json,
            metadata_json=schema.metadata,
            is_deprecated=schema.is_deprecated,
            deprecation_reason=schema.deprecation_reason,
            created_at=schema.created_at,
            updated_at=schema.updated_at,
            created_by=schema.created_by,
            updated_by=schema.updated_by,
            scope_profile_id=scope_profile_id,
        )


class ExtractionSchemaVersionORM(Base):
    """추출 스키마 버전 이력"""
    
    __tablename__ = "extraction_schema_versions"
    
    id = Column(PG_UUID(as_uuid=True), primary_key=True, default=uuid4)
    schema_id = Column(
        PG_UUID(as_uuid=True),
        ForeignKey("extraction_schemas.id", ondelete="CASCADE"),
        nullable=False,
        index=True
    )
    version = Column(Integer, nullable=False)
    
    # JSON 데이터
    fields_json = Column(JSONB, nullable=False)
    metadata_json = Column(JSONB, nullable=True)
    
    # 폐기 정보
    is_deprecated = Column(Boolean, nullable=False, default=False)
    deprecation_reason = Column(Text, nullable=True)
    
    # 변경 이력
    change_summary = Column(Text, nullable=True)
    changed_fields = Column(JSONB, nullable=True)  # List[str]
    
    # 감사 정보
    created_at = Column(DateTime(timezone=True), nullable=False, default=func.now())
    created_by = Column(String(255), nullable=False)
    
    # 관계
    schema = relationship(
        "ExtractionSchemaORM",
        back_populates="versions",
        foreign_keys=[schema_id]
    )
    
    # 인덱스
    __table_args__ = (
        UniqueConstraint("schema_id", "version", name="uq_schema_version"),
        Index("idx_created_at", "created_at"),
    )
    
    def to_pydantic(self) -> "ExtractionTargetSchema":
        """ORM 모델을 Pydantic 모델로 변환"""
        from src.domain.extraction.models import ExtractionTargetSchema, ExtractionFieldDef
        
        fields_dict = {}
        for field_name, field_def in (self.fields_json or {}).items():
            fields_dict[field_name] = ExtractionFieldDef(**field_def)
        
        return ExtractionTargetSchema(
            id=self.schema_id,
            doc_type_id=self.schema_id,  # schema_id에서 추출 (버전 이력용)
            version=self.version,
            fields=fields_dict,
            is_deprecated=self.is_deprecated,
            deprecation_reason=self.deprecation_reason,
            created_at=self.created_at,
            updated_at=self.created_at,
            created_by=self.created_by,
            updated_by=self.created_by,
            metadata=self.metadata_json or {},
        )
```

### 4.3 Repository 패턴 구현

**파일**: `src/domain/extraction/repository.py`

```python
from datetime import datetime
from typing import Any, Dict, List, Optional
from uuid import UUID

from sqlalchemy import and_, desc, select
from sqlalchemy.orm import Session

from src.domain.extraction.models import (
    ExtractionFieldDef,
    ExtractionTargetSchema,
)
from src.infrastructure.database.models.extraction import (
    ExtractionSchemaORM,
    ExtractionSchemaVersionORM,
)
from src.infrastructure.exceptions import (
    ForbiddenError,
    NotFoundError,
    ValidationError,
)
from src.infrastructure.scope.service import ScopeProfileService


class ActorInfo:
    """작업 수행자 정보"""
    
    def __init__(self, actor_id: str, actor_type: str = "user"):
        if actor_type not in ("user", "agent"):
            raise ValueError(f"Invalid actor_type: {actor_type}")
        self.actor_id = actor_id
        self.actor_type = actor_type


class ExtractionSchemaRepository:
    """추출 스키마 저장소"""
    
    def __init__(
        self,
        session: Session,
        scope_service: Optional[ScopeProfileService] = None
    ):
        self.session = session
        self.scope_service = scope_service
    
    async def create(
        self,
        doc_type_id: UUID,
        fields: Dict[str, ExtractionFieldDef],
        actor_info: ActorInfo,
        scope_profile_id: Optional[UUID] = None,
        metadata: Optional[Dict[str, Any]] = None,
    ) -> ExtractionTargetSchema:
        """새로운 추출 스키마 생성"""
        
        # Scope Profile 접근 권한 확인
        if scope_profile_id and self.scope_service:
            await self._check_access_scope(
                doc_type_id=doc_type_id,
                scope_profile_id=scope_profile_id,
                actor_id=actor_info.actor_id
            )
        
        # 기존 스키마 확인 (같은 doc_type_id로 이미 존재하는가)
        existing = self.session.execute(
            select(ExtractionSchemaORM).where(
                and_(
                    ExtractionSchemaORM.doc_type_id == doc_type_id,
                    ExtractionSchemaORM.is_soft_deleted == False
                )
            )
        ).first()
        
        if existing:
            raise ValidationError(
                f"DocumentType {doc_type_id}에 대한 추출 스키마가 이미 존재"
            )
        
        # Pydantic 모델 생성
        schema = ExtractionTargetSchema(
            id=UUID(int=0),  # 임시 ID
            doc_type_id=doc_type_id,
            version=1,
            fields=fields,
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow(),
            created_by=actor_info.actor_id,
            updated_by=actor_info.actor_id,
            metadata=metadata or {},
        )
        
        # ORM 모델로 변환 및 저장
        orm = ExtractionSchemaORM.from_pydantic(schema, scope_profile_id)
        self.session.add(orm)
        self.session.flush()  # ID 생성
        
        # 버전 이력 생성
        version_orm = ExtractionSchemaVersionORM(
            schema_id=orm.id,
            version=1,
            fields_json=orm.fields_json,
            metadata_json=orm.metadata_json,
            is_deprecated=orm.is_deprecated,
            deprecation_reason=orm.deprecation_reason,
            change_summary="초기 생성",
            changed_fields=[],
            created_at=datetime.utcnow(),
            created_by=actor_info.actor_id,
        )
        self.session.add(version_orm)
        self.session.commit()
        
        return orm.to_pydantic()
    
    def get_by_doc_type(
        self,
        doc_type_id: UUID,
        scope_profile_id: Optional[UUID] = None,
    ) -> Optional[ExtractionTargetSchema]:
        """DocumentType별 최신 추출 스키마 조회"""
        
        if scope_profile_id and self.scope_service:
            self._check_access_scope(
                doc_type_id=doc_type_id,
                scope_profile_id=scope_profile_id,
                actor_id=None  # 동기 버전은 actor_id 검증 생략
            )
        
        orm = self.session.execute(
            select(ExtractionSchemaORM).where(
                and_(
                    ExtractionSchemaORM.doc_type_id == doc_type_id,
                    ExtractionSchemaORM.is_soft_deleted == False,
                )
            ).order_by(desc(ExtractionSchemaORM.version)).limit(1)
        ).scalar_one_or_none()
        
        return orm.to_pydantic() if orm else None
    
    def get_by_doc_type_and_version(
        self,
        doc_type_id: UUID,
        version: int,
        scope_profile_id: Optional[UUID] = None,
    ) -> Optional[ExtractionTargetSchema]:
        """특정 버전의 추출 스키마 조회"""
        
        if scope_profile_id and self.scope_service:
            self._check_access_scope(
                doc_type_id=doc_type_id,
                scope_profile_id=scope_profile_id,
                actor_id=None
            )
        
        orm = self.session.execute(
            select(ExtractionSchemaORM).where(
                and_(
                    ExtractionSchemaORM.doc_type_id == doc_type_id,
                    ExtractionSchemaORM.version == version,
                    ExtractionSchemaORM.is_soft_deleted == False,
                )
            )
        ).scalar_one_or_none()
        
        return orm.to_pydantic() if orm else None
    
    def get_versions(
        self,
        doc_type_id: UUID,
        scope_profile_id: Optional[UUID] = None,
        limit: int = 10,
        offset: int = 0,
    ) -> List[ExtractionTargetSchema]:
        """추출 스키마 버전 이력 조회"""
        
        if scope_profile_id and self.scope_service:
            self._check_access_scope(
                doc_type_id=doc_type_id,
                scope_profile_id=scope_profile_id,
                actor_id=None
            )
        
        orms = self.session.execute(
            select(ExtractionSchemaVersionORM).where(
                ExtractionSchemaVersionORM.schema_id.in_(
                    select(ExtractionSchemaORM.id).where(
                        ExtractionSchemaORM.doc_type_id == doc_type_id
                    )
                )
            ).order_by(desc(ExtractionSchemaVersionORM.version)).limit(limit).offset(offset)
        ).scalars().all()
        
        return [orm.to_pydantic() for orm in orms]
    
    async def update(
        self,
        doc_type_id: UUID,
        fields: Dict[str, ExtractionFieldDef],
        actor_info: ActorInfo,
        scope_profile_id: Optional[UUID] = None,
        change_summary: Optional[str] = None,
    ) -> ExtractionTargetSchema:
        """추출 스키마 업데이트 (새 버전 생성)"""
        
        # 기존 스키마 조회
        orm = self.session.execute(
            select(ExtractionSchemaORM).where(
                and_(
                    ExtractionSchemaORM.doc_type_id == doc_type_id,
                    ExtractionSchemaORM.is_soft_deleted == False,
                )
            )
        ).scalar_one_or_none()
        
        if not orm:
            raise NotFoundError(f"DocumentType {doc_type_id}에 대한 추출 스키마 없음")
        
        # 접근 권한 확인
        if orm.scope_profile_id and self.scope_service:
            await self._check_access_scope(
                doc_type_id=doc_type_id,
                scope_profile_id=orm.scope_profile_id,
                actor_id=actor_info.actor_id
            )
        
        # 변경된 필드 감지
        old_fields = set(orm.fields_json.keys())
        new_fields = set(fields.keys())
        changed_fields = list(old_fields.symmetric_difference(new_fields))
        
        # 새 버전 번호
        new_version = orm.version + 1
        
        # ORM 업데이트
        orm.version = new_version
        orm.fields_json = {k: v.model_dump() for k, v in fields.items()}
        orm.updated_at = datetime.utcnow()
        orm.updated_by = actor_info.actor_id
        
        # 버전 이력 생성
        version_orm = ExtractionSchemaVersionORM(
            schema_id=orm.id,
            version=new_version,
            fields_json=orm.fields_json,
            metadata_json=orm.metadata_json,
            is_deprecated=orm.is_deprecated,
            deprecation_reason=orm.deprecation_reason,
            change_summary=change_summary or "필드 업데이트",
            changed_fields=changed_fields,
            created_at=datetime.utcnow(),
            created_by=actor_info.actor_id,
        )
        
        self.session.add(version_orm)
        self.session.commit()
        
        return orm.to_pydantic()
    
    def delete(
        self,
        doc_type_id: UUID,
        actor_info: ActorInfo,
    ) -> bool:
        """추출 스키마 소프트 삭제"""
        
        orm = self.session.execute(
            select(ExtractionSchemaORM).where(
                and_(
                    ExtractionSchemaORM.doc_type_id == doc_type_id,
                    ExtractionSchemaORM.is_soft_deleted == False,
                )
            )
        ).scalar_one_or_none()
        
        if not orm:
            return False
        
        orm.is_soft_deleted = True
        orm.deleted_at = datetime.utcnow()
        orm.deleted_by = actor_info.actor_id
        self.session.commit()
        
        return True
    
    def restore(
        self,
        doc_type_id: UUID,
        actor_info: ActorInfo,
    ) -> bool:
        """삭제된 스키마 복구"""
        
        orm = self.session.execute(
            select(ExtractionSchemaORM).where(
                and_(
                    ExtractionSchemaORM.doc_type_id == doc_type_id,
                    ExtractionSchemaORM.is_soft_deleted == True,
                )
            )
        ).scalar_one_or_none()
        
        if not orm:
            return False
        
        orm.is_soft_deleted = False
        orm.deleted_at = None
        orm.deleted_by = None
        self.session.commit()
        
        return True
    
    def deprecate(
        self,
        doc_type_id: UUID,
        reason: str,
        actor_info: ActorInfo,
    ) -> ExtractionTargetSchema:
        """추출 스키마 폐기 (deprecated 표시)"""
        
        orm = self.session.execute(
            select(ExtractionSchemaORM).where(
                and_(
                    ExtractionSchemaORM.doc_type_id == doc_type_id,
                    ExtractionSchemaORM.is_soft_deleted == False,
                )
            )
        ).scalar_one_or_none()
        
        if not orm:
            raise NotFoundError(f"DocumentType {doc_type_id}에 대한 추출 스키마 없음")
        
        orm.is_deprecated = True
        orm.deprecation_reason = reason
        orm.updated_by = actor_info.actor_id
        orm.updated_at = datetime.utcnow()
        self.session.commit()
        
        return orm.to_pydantic()
    
    def list_all(
        self,
        is_deprecated: Optional[bool] = None,
        scope_profile_id: Optional[UUID] = None,
        include_deleted: bool = False,
        limit: int = 100,
        offset: int = 0,
    ) -> List[ExtractionTargetSchema]:
        """모든 추출 스키마 목록 조회"""
        
        query = select(ExtractionSchemaORM)
        
        # Soft delete 필터
        if not include_deleted:
            query = query.where(ExtractionSchemaORM.is_soft_deleted == False)
        
        # deprecated 필터
        if is_deprecated is not None:
            query = query.where(ExtractionSchemaORM.is_deprecated == is_deprecated)
        
        # Scope Profile 필터
        if scope_profile_id:
            query = query.where(ExtractionSchemaORM.scope_profile_id == scope_profile_id)
        
        orms = self.session.execute(
            query.order_by(desc(ExtractionSchemaORM.updated_at)).limit(limit).offset(offset)
        ).scalars().all()
        
        return [orm.to_pydantic() for orm in orms]
    
    def search_by_field_name(
        self,
        field_name: str,
        scope_profile_id: Optional[UUID] = None,
        include_deleted: bool = False,
    ) -> List[ExtractionTargetSchema]:
        """필드명으로 추출 스키마 검색 (JSONB 쿼리)"""
        
        # PostgreSQL의 JSONB 연산자 사용: fields_json -> field_name
        query = select(ExtractionSchemaORM).where(
            ExtractionSchemaORM.fields_json.has_key(field_name)
        )
        
        if not include_deleted:
            query = query.where(ExtractionSchemaORM.is_soft_deleted == False)
        
        if scope_profile_id:
            query = query.where(ExtractionSchemaORM.scope_profile_id == scope_profile_id)
        
        orms = self.session.execute(query).scalars().all()
        return [orm.to_pydantic() for orm in orms]
    
    async def _check_access_scope(
        self,
        doc_type_id: UUID,
        scope_profile_id: UUID,
        actor_id: Optional[str] = None,
    ) -> None:
        """Scope Profile 기반 ACL 검증"""
        
        if not self.scope_service:
            return
        
        # scope_service.check_access() 호출
        # has_access = await self.scope_service.check_access(
        #     actor_id=actor_id,
        #     scope_profile_id=scope_profile_id,
        #     resource_type="document_type",
        #     resource_id=str(doc_type_id)
        # )
        # 
        # if not has_access:
        #     raise ForbiddenError(
        #         f"DocumentType {doc_type_id}에 대한 접근 권한 없음"
        #     )
        pass
```

### 4.4 Alembic 마이그레이션

**파일**: `alembic/versions/{timestamp}_add_extraction_schema_tables.py`

```python
"""Add extraction_schemas and extraction_schema_versions tables

Revision ID: 0001_extraction_schema
Revises: <previous_revision>
Create Date: 2026-04-17 00:00:00.000000

"""
from typing import Sequence, Union
from uuid import uuid4

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# revision identifiers, used by Alembic.
revision: str = '0001_extraction_schema'
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # extraction_schemas 테이블 생성
    op.create_table(
        'extraction_schemas',
        sa.Column('id', postgresql.UUID(as_uuid=True), nullable=False, default=uuid4),
        sa.Column('doc_type_id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('version', sa.Integer(), nullable=False, server_default='1'),
        sa.Column('fields_json', postgresql.JSONB(), nullable=False),
        sa.Column('metadata_json', postgresql.JSONB(), nullable=True),
        sa.Column('is_deprecated', sa.Boolean(), nullable=False, server_default='false'),
        sa.Column('deprecation_reason', sa.Text(), nullable=True),
        sa.Column('created_at', sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.Column('updated_at', sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.Column('created_by', sa.String(255), nullable=False),
        sa.Column('updated_by', sa.String(255), nullable=False),
        sa.Column('scope_profile_id', postgresql.UUID(as_uuid=True), nullable=True),
        sa.Column('is_soft_deleted', sa.Boolean(), nullable=False, server_default='false'),
        sa.Column('deleted_at', sa.DateTime(timezone=True), nullable=True),
        sa.Column('deleted_by', sa.String(255), nullable=True),
        sa.ForeignKeyConstraint(['doc_type_id'], ['document_types.id'], ondelete='RESTRICT'),
        sa.ForeignKeyConstraint(['scope_profile_id'], ['scope_profiles.id'], ondelete='SET NULL'),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint(
            'doc_type_id', 'version',
            name='uq_doc_type_version',
            postgresql_where=sa.text('is_soft_deleted = false')
        ),
    )
    
    # 인덱스
    op.create_index('idx_doc_type_id', 'extraction_schemas', ['doc_type_id'])
    op.create_index('idx_scope_profile_id', 'extraction_schemas', ['scope_profile_id'])
    op.create_index('idx_is_deprecated', 'extraction_schemas', ['is_deprecated'])
    op.create_index('idx_doc_type_version', 'extraction_schemas', ['doc_type_id', 'version'])
    
    # extraction_schema_versions 테이블 생성
    op.create_table(
        'extraction_schema_versions',
        sa.Column('id', postgresql.UUID(as_uuid=True), nullable=False, default=uuid4),
        sa.Column('schema_id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('version', sa.Integer(), nullable=False),
        sa.Column('fields_json', postgresql.JSONB(), nullable=False),
        sa.Column('metadata_json', postgresql.JSONB(), nullable=True),
        sa.Column('is_deprecated', sa.Boolean(), nullable=False, server_default='false'),
        sa.Column('deprecation_reason', sa.Text(), nullable=True),
        sa.Column('change_summary', sa.Text(), nullable=True),
        sa.Column('changed_fields', postgresql.JSONB(), nullable=True),
        sa.Column('created_at', sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.Column('created_by', sa.String(255), nullable=False),
        sa.ForeignKeyConstraint(['schema_id'], ['extraction_schemas.id'], ondelete='CASCADE'),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('schema_id', 'version', name='uq_schema_version'),
    )
    
    # 인덱스
    op.create_index('idx_schema_id', 'extraction_schema_versions', ['schema_id'])
    op.create_index('idx_created_at', 'extraction_schema_versions', ['created_at'])


def downgrade() -> None:
    op.drop_table('extraction_schema_versions')
    op.drop_table('extraction_schemas')
```

### 4.5 단위 테스트

**파일**: `tests/domain/extraction/test_extraction_field_def.py`

```python
import pytest
from datetime import datetime

from src.domain.extraction.models import (
    ExtractionFieldDef,
    ExtractionTargetSchema,
)


class TestExtractionFieldDef:
    """ExtractionFieldDef 테스트"""
    
    def test_string_field_creation(self):
        """문자열 필드 생성"""
        field = ExtractionFieldDef(
            field_name="invoice_number",
            field_type="string",
            required=True,
            description="인보이스 번호",
            pattern=r"^INV-\d{6}$",
            examples=["INV-000001", "INV-999999"],
            max_length=20,
        )
        assert field.field_name == "invoice_number"
        assert field.field_type == "string"
        assert field.required is True
    
    def test_number_field_creation(self):
        """숫자 필드 생성"""
        field = ExtractionFieldDef(
            field_name="total_amount",
            field_type="number",
            required=True,
            description="총액",
            examples=["1000.50", "2500.00"],
            min_value=0.0,
            max_value=999999.99,
        )
        assert field.field_type == "number"
        assert field.min_value == 0.0
    
    def test_date_field_creation(self):
        """날짜 필드 생성"""
        field = ExtractionFieldDef(
            field_name="invoice_date",
            field_type="date",
            required=True,
            description="인보이스 발행일",
            date_format="YYYY-MM-DD",
            examples=["2026-04-17", "2026-01-01"],
        )
        assert field.field_type == "date"
        assert field.date_format == "YYYY-MM-DD"
    
    def test_enum_field_creation(self):
        """enum 필드 생성"""
        field = ExtractionFieldDef(
            field_name="status",
            field_type="enum",
            required=True,
            description="상태",
            enum_values=["draft", "approved", "rejected"],
            examples=["draft", "approved"],
        )
        assert field.field_type == "enum"
        assert len(field.enum_values) == 3
    
    def test_object_field_creation(self):
        """object 필드 생성 (중첩 필드)"""
        field = ExtractionFieldDef(
            field_name="buyer_info",
            field_type="object",
            required=True,
            description="구매자 정보",
            nested_schema={
                "name": ExtractionFieldDef(
                    field_name="name",
                    field_type="string",
                    required=True,
                    description="이름",
                    examples=["홍길동", "김철수"],
                ),
                "email": ExtractionFieldDef(
                    field_name="email",
                    field_type="string",
                    required=False,
                    description="이메일",
                    pattern=r"^[\w\.-]+@[\w\.-]+\.\w+$",
                    examples=["hong@example.com"],
                ),
            }
        )
        assert field.field_type == "object"
        assert "name" in field.nested_schema
        assert "email" in field.nested_schema
    
    def test_invalid_field_name(self):
        """잘못된 필드명 검증"""
        with pytest.raises(ValueError, match="필드명은 snake_case 형식"):
            ExtractionFieldDef(
                field_name="InvoiceNumber",  # PascalCase (invalid)
                field_type="string",
                required=True,
                description="필드",
                examples=["ex1", "ex2"],
            )
    
    def test_invalid_pattern(self):
        """잘못된 정규식 패턴 검증"""
        with pytest.raises(ValueError, match="정규식 오류"):
            ExtractionFieldDef(
                field_name="phone",
                field_type="string",
                required=True,
                description="전화번호",
                pattern=r"[invalid(regex",  # invalid regex
                examples=["01012345678"],
            )
    
    def test_type_consistency_violation_string_with_enum(self):
        """타입 일관성 위반: string with enum_values"""
        with pytest.raises(ValueError, match="string 타입에서는 enum_values"):
            ExtractionFieldDef(
                field_name="type",
                field_type="string",
                required=True,
                description="타입",
                enum_values=["A", "B"],
                examples=["A"],
            )
    
    def test_type_consistency_violation_number_with_pattern(self):
        """타입 일관성 위반: number with pattern"""
        with pytest.raises(ValueError, match="number 타입에서는 pattern"):
            ExtractionFieldDef(
                field_name="code",
                field_type="number",
                required=True,
                description="코드",
                pattern=r"\d+",
                examples=["123"],
            )
    
    def test_enum_without_values(self):
        """enum 타입에 enum_values 없음"""
        with pytest.raises(ValueError, match="enum 타입에서는 enum_values가 필수"):
            ExtractionFieldDef(
                field_name="status",
                field_type="enum",
                required=True,
                description="상태",
                enum_values=None,
                examples=["a"],
            )
    
    def test_object_without_nested_schema(self):
        """object 타입에 nested_schema 없음"""
        with pytest.raises(ValueError, match="object 타입에서는 nested_schema가 필수"):
            ExtractionFieldDef(
                field_name="info",
                field_type="object",
                required=True,
                description="정보",
                nested_schema=None,
                examples=["{}"],
            )
    
    def test_min_max_value_order(self):
        """min_value > max_value 검증"""
        with pytest.raises(ValueError, match="min_value는 max_value보다 작아야"):
            ExtractionFieldDef(
                field_name="amount",
                field_type="number",
                required=True,
                description="금액",
                min_value=1000.0,
                max_value=100.0,  # invalid order
                examples=["500.0"],
            )
    
    def test_example_validation_number_type(self):
        """number 타입의 예제 검증"""
        with pytest.raises(ValueError, match="number 타입의 예제.*숫자로 변환 불가"):
            ExtractionFieldDef(
                field_name="amount",
                field_type="number",
                required=True,
                description="금액",
                examples=["not_a_number"],
            )
    
    def test_example_validation_enum_type(self):
        """enum 타입의 예제가 enum_values에 있는지 확인"""
        with pytest.raises(ValueError, match="enum 타입의 예제.*enum_values에 없음"):
            ExtractionFieldDef(
                field_name="status",
                field_type="enum",
                required=True,
                description="상태",
                enum_values=["approved", "rejected"],
                examples=["approved", "invalid_status"],
            )
    
    def test_required_with_default_value_warning(self, caplog):
        """required=True이면서 default_value가 있을 때 경고"""
        import warnings
        with warnings.catch_warnings(record=True) as w:
            warnings.simplefilter("always")
            ExtractionFieldDef(
                field_name="status",
                field_type="string",
                required=True,
                description="상태",
                default_value="approved",
                examples=["approved"],
            )
            assert len(w) > 0
            assert "required이면서 default_value" in str(w[-1].message)


class TestExtractionTargetSchema:
    """ExtractionTargetSchema 테스트"""
    
    def test_schema_creation(self):
        """스키마 생성"""
        from uuid import uuid4
        
        doc_type_id = uuid4()
        schema = ExtractionTargetSchema(
            id=uuid4(),
            doc_type_id=doc_type_id,
            version=1,
            fields={
                "invoice_number": ExtractionFieldDef(
                    field_name="invoice_number",
                    field_type="string",
                    required=True,
                    description="인보이스 번호",
                    examples=["INV-001", "INV-002"],
                ),
                "total_amount": ExtractionFieldDef(
                    field_name="total_amount",
                    field_type="number",
                    required=True,
                    description="총액",
                    examples=["1000.0", "2000.0"],
                ),
            },
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow(),
            created_by="user_001",
            updated_by="user_001",
        )
        assert schema.version == 1
        assert len(schema.fields) == 2
    
    def test_deprecated_schema_validation(self):
        """폐기 스키마 검증"""
        from uuid import uuid4
        
        with pytest.raises(ValueError, match="is_deprecated=True일 때 deprecation_reason이 필수"):
            ExtractionTargetSchema(
                id=uuid4(),
                doc_type_id=uuid4(),
                version=1,
                fields={
                    "field1": ExtractionFieldDef(
                        field_name="field1",
                        field_type="string",
                        required=True,
                        description="필드1",
                        examples=["a", "b"],
                    ),
                },
                is_deprecated=True,  # true인데
                deprecation_reason=None,  # reason이 없음
                created_at=datetime.utcnow(),
                updated_at=datetime.utcnow(),
                created_by="user_001",
                updated_by="user_001",
            )
    
    def test_empty_fields_validation(self):
        """빈 필드 검증"""
        from uuid import uuid4
        
        with pytest.raises(ValueError, match="추출 스키마는 최소 1개의 필드를 포함"):
            ExtractionTargetSchema(
                id=uuid4(),
                doc_type_id=uuid4(),
                version=1,
                fields={},  # empty
                created_at=datetime.utcnow(),
                updated_at=datetime.utcnow(),
                created_by="user_001",
                updated_by="user_001",
            )
```

**파일**: `tests/domain/extraction/test_extraction_schema_repository.py`

```python
import pytest
from datetime import datetime
from uuid import uuid4
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from src.domain.extraction.models import (
    ExtractionFieldDef,
    ExtractionTargetSchema,
)
from src.domain.extraction.repository import (
    ExtractionSchemaRepository,
    ActorInfo,
)
from src.infrastructure.database.models.extraction import (
    ExtractionSchemaORM,
    ExtractionSchemaVersionORM,
)
from src.infrastructure.database.base import Base


@pytest.fixture
def db_session():
    """테스트용 임시 DB 세션"""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    Session = sessionmaker(bind=engine)
    session = Session()
    yield session
    session.close()


@pytest.fixture
def repository(db_session):
    """테스트용 Repository"""
    return ExtractionSchemaRepository(session=db_session)


@pytest.fixture
def sample_fields():
    """샘플 필드 정의"""
    return {
        "invoice_number": ExtractionFieldDef(
            field_name="invoice_number",
            field_type="string",
            required=True,
            description="인보이스 번호",
            pattern=r"^INV-\d{6}$",
            examples=["INV-000001"],
            max_length=20,
        ),
        "total_amount": ExtractionFieldDef(
            field_name="total_amount",
            field_type="number",
            required=True,
            description="총액",
            examples=["1000.50"],
            min_value=0.0,
            max_value=999999.99,
        ),
    }


class TestExtractionSchemaRepository:
    """ExtractionSchemaRepository 테스트"""
    
    @pytest.mark.asyncio
    async def test_create_schema(self, repository, sample_fields):
        """스키마 생성 테스트"""
        doc_type_id = uuid4()
        actor_info = ActorInfo(actor_id="user_001", actor_type="user")
        
        schema = await repository.create(
            doc_type_id=doc_type_id,
            fields=sample_fields,
            actor_info=actor_info,
        )
        
        assert schema.doc_type_id == doc_type_id
        assert schema.version == 1
        assert len(schema.fields) == 2
        assert schema.created_by == "user_001"
    
    def test_get_by_doc_type(self, repository, sample_fields):
        """DocumentType별 조회 테스트"""
        doc_type_id = uuid4()
        actor_info = ActorInfo(actor_id="user_001", actor_type="user")
        
        # 생성 (동기 버전이 필요하면 동기 메서드로 수정)
        orm = ExtractionSchemaORM.from_pydantic(
            ExtractionTargetSchema(
                id=uuid4(),
                doc_type_id=doc_type_id,
                version=1,
                fields=sample_fields,
                created_at=datetime.utcnow(),
                updated_at=datetime.utcnow(),
                created_by="user_001",
                updated_by="user_001",
            )
        )
        repository.session.add(orm)
        repository.session.commit()
        
        # 조회
        schema = repository.get_by_doc_type(doc_type_id)
        assert schema is not None
        assert schema.doc_type_id == doc_type_id
    
    def test_soft_delete(self, repository, sample_fields):
        """소프트 삭제 테스트"""
        doc_type_id = uuid4()
        actor_info = ActorInfo(actor_id="user_001", actor_type="user")
        
        # 생성
        orm = ExtractionSchemaORM.from_pydantic(
            ExtractionTargetSchema(
                id=uuid4(),
                doc_type_id=doc_type_id,
                version=1,
                fields=sample_fields,
                created_at=datetime.utcnow(),
                updated_at=datetime.utcnow(),
                created_by="user_001",
                updated_by="user_001",
            )
        )
        repository.session.add(orm)
        repository.session.commit()
        
        # 삭제
        result = repository.delete(doc_type_id, actor_info)
        assert result is True
        
        # 삭제 후 조회 안 됨
        schema = repository.get_by_doc_type(doc_type_id)
        assert schema is None
    
    def test_deprecate_schema(self, repository, sample_fields):
        """스키마 폐기 테스트"""
        doc_type_id = uuid4()
        actor_info = ActorInfo(actor_id="user_001", actor_type="user")
        
        # 생성
        orm = ExtractionSchemaORM.from_pydantic(
            ExtractionTargetSchema(
                id=uuid4(),
                doc_type_id=doc_type_id,
                version=1,
                fields=sample_fields,
                created_at=datetime.utcnow(),
                updated_at=datetime.utcnow(),
                created_by="user_001",
                updated_by="user_001",
            )
        )
        repository.session.add(orm)
        repository.session.commit()
        
        # 폐기
        deprecated = repository.deprecate(
            doc_type_id=doc_type_id,
            reason="새로운 스키마로 대체됨",
            actor_info=actor_info,
        )
        
        assert deprecated.is_deprecated is True
        assert "새로운 스키마" in deprecated.deprecation_reason
```

---

## 5. 산출물 (Deliverables)

| 파일 경로 | 설명 | 요구사항 |
|---------|------|--------|
| `src/domain/extraction/models.py` | Pydantic 도메인 모델 (ExtractionFieldDef, ExtractionTargetSchema, FieldValidationRule) | 완성도 100% |
| `src/infrastructure/database/models/extraction.py` | SQLAlchemy ORM 모델 (ExtractionSchemaORM, ExtractionSchemaVersionORM) | 완성도 100% |
| `alembic/versions/{timestamp}_add_extraction_schema_tables.py` | Alembic 마이그레이션 스크립트 | 완성도 100% |
| `src/domain/extraction/repository.py` | ExtractionSchemaRepository 구현 | 완성도 100%, Scope Profile ACL 포함 |
| `tests/domain/extraction/test_extraction_field_def.py` | ExtractionFieldDef 단위 테스트 (최소 15개 테스트 케이스) | 커버리지 ≥95% |
| `tests/domain/extraction/test_extraction_schema_repository.py` | ExtractionSchemaRepository 단위 테스트 (최소 10개 테스트 케이스) | 커버리지 ≥90% |
| `tests/domain/extraction/conftest.py` | pytest 픽스처 및 공통 테스트 유틸리티 | 재사용 가능 |

---

## 6. 완료 기준

1. **Pydantic 모델 구현**
   - ExtractionFieldDef: 모든 필드 정의, 타입별 검증 로직 포함
   - ExtractionTargetSchema: 버전 관리, 폐기 상태 관리
   - 필드 타입 일관성 검증: string/number/date/boolean/enum/array/object 모두 검증

2. **SQLAlchemy ORM 모델**
   - ExtractionSchemaORM: JSONB 저장, FK 관계, 인덱스 정의
   - ExtractionSchemaVersionORM: 버전 이력 추적, 변경사항 기록
   - to_pydantic(), from_pydantic() 메서드 구현

3. **Alembic 마이그레이션**
   - 마이그레이션 파일 작성 및 테스트
   - upgrade(), downgrade() 로직 완성
   - 인덱스, FK 제약조건 포함

4. **Repository 패턴**
   - 모든 CRUD 메서드 구현
   - ActorInfo를 통한 actor_type 추적
   - Scope Profile ACL 필터링 통합
   - Soft delete 지원

5. **단위 테스트**
   - test_extraction_field_def.py: 타입별 필드 생성, 검증 오류 20개 테스트 케이스
   - test_extraction_schema_repository.py: CRUD, 버전 관리, soft delete 15개 테스트 케이스
   - 테스트 커버리지: 도메인 모델 ≥95%, Repository ≥90%

6. **코드 품질**
   - Linting: flake8, black, isort 통과
   - Type hints: 100% 포함 (mypy --strict 통과)
   - Docstrings: 모든 public 메서드에 포함

7. **문서화**
   - 각 클래스, 메서드에 docstring 포함
   - 필드별 description 포함
   - 예제 코드 포함

---

## 7. 작업 지침

**7-1**: **필드 타입 하드코딩 금지** — DocumentType별로 필드 타입이 다를 수 있으므로, 필드 정의는 Pydantic 모델의 필드 타입 Literal을 통해 동적으로 정의. 서비스 레이어에서 DocumentType별 분기 처리 (S1 원칙 ①).

**7-2**: **Scope Profile ACL 필터링** — 모든 Repository 메서드는 scope_profile_id 매개변수를 받아 Scope Profile 기반 접근 제어 수행. 권한 검증 없을 때는 ForbiddenError 발생 (S2 원칙 ⑥).

**7-3**: **actor_type 필드 통합** — ActorInfo(actor_id, actor_type) 클래스를 통해 "user" 또는 "agent"를 구분. created_by, updated_by, deleted_by에 actor_id만 저장하되, audit_logs 테이블에는 actor_type도 함께 기록 (S2 원칙 ⑤).

**7-4**: **JSON 직렬화/역직렬화** — PostgreSQL의 JSONB 타입 사용. Pydantic 모델과 ORM 모델 간 변환은 to_pydantic(), from_pydantic() 메서드로 추상화. 필드는 ExtractionFieldDef 리스트가 아닌 Dict[str, ExtractionFieldDef] 맵으로 저장 (JSON 접근 용이).

**7-5**: **버전 관리 규칙** — 스키마 생성 시 version=1. 각 업데이트 시 version 자동 증가. 버전 이력은 ExtractionSchemaVersionORM에 모두 기록. version_history API에서는 최신 5개만 기본 조회 (pagination 지원).

**7-6**: **Soft Delete 기본 정책** — is_soft_deleted=True인 항목은 모든 조회에서 기본 제외. 명시적으로 include_deleted=True를 전달해야만 조회 가능. 복구 기능(restore) 지원 (관리자용).

**7-7**: **테스트 커버리지** — 단위 테스트는 pytest를 사용하며, 다음을 반드시 포함:
  - 각 필드 타입별 생성 및 검증 (string, number, date, boolean, enum, array, object)
  - 타입 일관성 검증 (type consistency violations)
  - 필드 제약조건 검증 (pattern, enum_values, min/max 등)
  - 정규식 오류 처리
  - 예제 데이터 검증
  - 버전 증가 로직
  - Soft delete/restore
  - Scope Profile ACL 필터링
  - actor_type 추적

**7-8**: **폐기(Deprecation) 정책** — is_deprecated=True일 때는 반드시 deprecation_reason이 있어야 함. 폐기된 스키마는 조회 가능하지만, "deprecated 스키마 사용 금지" 경고 제시. 신규 추출 작업은 deprecated 스키마를 사용할 수 없음 (task8-2에서 구현).

**7-9**: **nested_schema 검증** — object 타입 필드는 nested_schema를 반드시 포함. nested_schema의 각 필드도 재귀적으로 ExtractionFieldDef 검증 규칙 적용. 중첩 깊이 제한 필요 시 max_depth=3 정책 추가.

**7-10**: **외부 의존성 최소화** (S2 원칙 ⑦) — Scope Profile 서비스와의 연동은 optional(self.scope_service는 None 가능). scope_service가 None이면 ACL 필터링 skip. 이를 통해 폐쇄망(no SaaS) 환경에서도 기본 기능 동작 보장.

---

**작업 예상 소요 시간**: 32-40시간  
**마지막 업데이트**: 2026-04-17
