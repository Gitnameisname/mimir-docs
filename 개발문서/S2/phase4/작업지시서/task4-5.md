# Task 4-5. Scope Profile 도메인 모델 + FilterExpression 파서 + $ctx 치환

## 1. 작업 목적

접근 범위를 코드에 하드코딩하지 않고 동적으로 관리하기 위해 **Scope Profile 도메인 모델**을 설계한다. 관리자가 Scope Profile을 통해 에이전트와 사용자의 접근 범위를 동적으로 설정할 수 있으며, FilterExpression 파서를 구현하여 조건부 ACL 필터링을 지원한다. 또한 $ctx 동적 변수를 통해 요청 컨텍스트 (organization_id, user_id, team_id 등)에 따라 유동적인 필터를 적용한다 (S2 원칙 ⑥).

## 2. 작업 범위

### 포함 범위

1. **Scope Profile 도메인 모델 (SQLAlchemy ORM)**
   - 테이블: scope_profiles (UUID id, name, description, organization_id, created_at, updated_at)
   - ScopeProfile 클래스: scope_definitions 관계 (일대다)
   - 조직별 Scope Profile 관리

2. **Scope Definition 도메인 모델**
   - 테이블: scope_definitions (UUID id, scope_profile_id FK, scope_name, acl_filter JSONB, description)
   - scope_name: 논리명 (예: "team_search", "public_documents")
   - acl_filter: FilterExpression JSON 저장
   - 하나의 scope_profile은 여러 scope_definition을 포함 가능

3. **FilterExpression 파서**
   - 파일: `/backend/app/models/filter_expression.py`
   - 목적: JSONB 형식의 FilterExpression을 파싱하여 SQLAlchemy WHERE 절로 변환
   - 지원 연산자: eq, neq, in, not_in, contains
   - 논리 연산: and, or (재귀적 파싱)
   - 형식:
     ```json
     {
       "and": [
         {"field": "organization_id", "op": "eq", "value": "$ctx.organization_id"},
         {"or": [
           {"field": "owner_id", "op": "eq", "value": "$ctx.user_id"},
           {"field": "access_level", "op": "eq", "value": "public"}
         ]}
       ]
     }
     ```

4. **$ctx 동적 변수 치환**
   - 지원 변수: $ctx.organization_id, $ctx.user_id, $ctx.team_id, $ctx.scope_profile_id
   - 화이트리스트 검증: 정의된 변수만 치환 허용 (인젝션 방어, S2 원칙 ⑥)
   - 치환 시점: FilterExpression 파싱 전, access_context에서 변수 추출

5. **FilterExpression → SQLAlchemy WHERE 절 변환**
   - FilterParser 클래스: parse(filter_expr, access_context) → SQLAlchemy BinaryExpression
   - 타겟 모델 (Conversation, Document, etc.)에 따라 동적으로 필터 생성
   - 다중 필터 합성 (AND/OR 로직 지원)

6. **보안: $ctx 변수 화이트리스트**
   - 허용 변수: organization_id, user_id, team_id, scope_profile_id
   - 미허용 변수 시도 시 BadRequestError 발생
   - SQL 인젝션 방어: 리터럴 값은 문자열로 직접 사용하지 않고, SQLAlchemy의 바인드 파라미터 사용

7. **Scope Definition CRUD 로직**
   - Repository: ScopeProfileRepository, ScopeDefinitionRepository
   - 조회, 생성, 업데이트, 삭제 메서드
   - scope_profile_id로 정의 조회

8. **Alembic 마이그레이션 스크립트**
   - scope_profiles 테이블 생성
   - scope_definitions 테이블 생성
   - 외래키 및 인덱스 정의

9. **단위 테스트**
   - FilterExpression 파서 정확성 테스트
   - $ctx 변수 치환 테스트
   - 복잡한 and/or 조건 파싱 테스트
   - 인젝션 방어 테스트 (미허용 변수 거부)
   - SQLAlchemy WHERE 절 변환 테스트

### 제외 범위

- Agent Principal Type 및 Delegation 모델 (Task 4-4)
- Scope Profile CRUD API 엔드포인트 (Task 4-6)
- Kill Switch API (Task 4-6)
- MCP Server 구현 (FG4.1)

## 3. 선행 조건

- Phase 1, Phase 2, Phase 3 완료
- Task 4-4 (Agent Principal Type) 완료
- PostgreSQL 설치 및 연결 가능
- SQLAlchemy 2.0+ 설치
- Alembic 초기화 완료

## 4. 주요 작업 항목

### 4-1. Scope Profile 도메인 모델

**파일:** `/backend/app/models/scope_profile.py`

```python
from __future__ import annotations

from datetime import datetime
from typing import Optional
from uuid import UUID, uuid4

from sqlalchemy import (
    Column, String, DateTime, ForeignKey,
    Index, Text as SQLText
)
from sqlalchemy.dialects.postgresql import UUID as PGUUID, JSONB
from sqlalchemy.orm import relationship, mapped_column
from sqlalchemy.sql import func

from app.models.base import Base


class ScopeProfile(Base):
    """접근 범위를 정의하는 프로필 (에이전트/사용자별).
    
    ScopeProfile은 여러 ScopeDefinition을 포함하여,
    각 scope_name에 대해 다른 ACL 필터를 적용할 수 있도록 함.
    
    Attributes:
        id: 고유 식별자 (UUID)
        organization_id: 소속 조직 ID (UUID)
        name: 프로필 이름 (예: "team-a-reader", "public-agent")
        description: 프로필 설명
        created_at: 생성 시간
        updated_at: 마지막 수정 시간
        scope_definitions: ScopeDefinition 관계 (일대다)
    """

    __tablename__ = "scope_profiles"

    id = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    organization_id = mapped_column(PGUUID(as_uuid=True), nullable=False, index=True)

    name = mapped_column(String(256), nullable=False)
    description = mapped_column(String(512), nullable=True)

    created_at = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)

    # 관계
    scope_definitions = relationship(
        "ScopeDefinition",
        back_populates="scope_profile",
        cascade="all, delete-orphan",
        lazy="selectin"
    )

    # 인덱스
    __table_args__ = (
        Index("ix_scope_profiles_organization_created", "organization_id", "created_at"),
    )

    def __repr__(self) -> str:
        return f"<ScopeProfile(id={self.id}, name={self.name})>"


class ScopeDefinition(Base):
    """특정 scope_name에 대한 ACL 필터 정의.
    
    하나의 ScopeDefinition은 하나의 scope_name을 정의하고,
    해당 scope에 적용할 FilterExpression을 JSONB로 저장.
    
    Attributes:
        id: 고유 식별자 (UUID)
        scope_profile_id: 소속 ScopeProfile ID (UUID, FK)
        scope_name: 논리 이름 (예: "team_search", "public_documents")
        acl_filter: FilterExpression (JSONB)
            {
              "and": [
                {"field": "organization_id", "op": "eq", "value": "$ctx.organization_id"},
                ...
              ]
            }
        description: 필터 설명
        created_at: 생성 시간
        updated_at: 마지막 수정 시간
    """

    __tablename__ = "scope_definitions"

    id = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    scope_profile_id = mapped_column(
        PGUUID(as_uuid=True),
        ForeignKey("scope_profiles.id", ondelete="CASCADE"),
        nullable=False,
        index=True
    )

    scope_name = mapped_column(String(256), nullable=False)
    acl_filter = mapped_column(JSONB, default=dict, nullable=False)
    description = mapped_column(String(512), nullable=True)

    created_at = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)

    # 관계
    scope_profile = relationship("ScopeProfile", back_populates="scope_definitions")

    # 인덱스
    __table_args__ = (
        Index("ix_scope_definitions_scope_profile_name", "scope_profile_id", "scope_name", unique=True),
    )

    def __repr__(self) -> str:
        return f"<ScopeDefinition(id={self.id}, scope_name={self.scope_name})>"
```

### 4-2. FilterExpression 파서 및 $ctx 치환

**파일:** `/backend/app/models/filter_expression.py`

```python
"""FilterExpression 파서 및 SQLAlchemy WHERE 절 변환."""

from __future__ import annotations

from typing import Any, Dict, List, Optional, Union
from abc import ABC, abstractmethod
from uuid import UUID

from sqlalchemy import and_, or_, func
from sqlalchemy.orm import InspectionAttr
from sqlalchemy.sql import ColumnElement

from app.models.access_context import AccessContext
from app.exceptions import BadRequestError


# FilterExpression 데이터 클래스들

class FilterCondition:
    """단일 필터 조건.
    
    Attributes:
        field: 필터링할 필드명 (예: "organization_id", "owner_id")
        op: 연산자 ("eq", "neq", "in", "not_in", "contains")
        value: 리터럴 값 또는 $ctx 변수
    """

    VALID_OPERATORS = {"eq", "neq", "in", "not_in", "contains"}

    def __init__(self, field: str, op: str, value: Any):
        if op not in self.VALID_OPERATORS:
            raise BadRequestError(f"Invalid operator: {op}")

        self.field = field
        self.op = op
        self.value = value

    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> FilterCondition:
        """딕셔너리에서 FilterCondition 생성."""
        return cls(
            field=data.get("field"),
            op=data.get("op"),
            value=data.get("value")
        )

    def __repr__(self) -> str:
        return f"<FilterCondition(field={self.field}, op={self.op}, value={self.value})>"


class FilterExpression(ABC):
    """필터 표현식 (AND/OR 로직 지원)."""

    @abstractmethod
    def to_sqlalchemy(
        self,
        model: type,
        access_context: AccessContext,
        variable_mapping: Optional[Dict[str, Any]] = None,
    ) -> ColumnElement:
        """SQLAlchemy WHERE 절로 변환."""
        pass


class AndExpression(FilterExpression):
    """AND 조건."""

    def __init__(self, conditions: List[Union[FilterCondition, FilterExpression]]):
        self.conditions = conditions

    @classmethod
    def from_dict(cls, data: List[Dict[str, Any]]) -> AndExpression:
        """딕셔너리 리스트에서 AndExpression 생성."""
        parsed_conditions = []
        for item in data:
            if "and" in item or "or" in item:
                parsed_conditions.append(_parse_filter_expression(item))
            else:
                parsed_conditions.append(FilterCondition.from_dict(item))
        return cls(parsed_conditions)

    def to_sqlalchemy(
        self,
        model: type,
        access_context: AccessContext,
        variable_mapping: Optional[Dict[str, Any]] = None,
    ) -> ColumnElement:
        """SQLAlchemy AND 표현식 생성."""
        conditions = []
        for cond in self.conditions:
            if isinstance(cond, FilterExpression):
                conditions.append(cond.to_sqlalchemy(model, access_context, variable_mapping))
            else:  # FilterCondition
                conditions.append(
                    _condition_to_sqlalchemy(model, cond, access_context, variable_mapping)
                )
        return and_(*conditions)

    def __repr__(self) -> str:
        return f"<AndExpression(conditions={self.conditions})>"


class OrExpression(FilterExpression):
    """OR 조건."""

    def __init__(self, conditions: List[Union[FilterCondition, FilterExpression]]):
        self.conditions = conditions

    @classmethod
    def from_dict(cls, data: List[Dict[str, Any]]) -> OrExpression:
        """딕셔너리 리스트에서 OrExpression 생성."""
        parsed_conditions = []
        for item in data:
            if "and" in item or "or" in item:
                parsed_conditions.append(_parse_filter_expression(item))
            else:
                parsed_conditions.append(FilterCondition.from_dict(item))
        return cls(parsed_conditions)

    def to_sqlalchemy(
        self,
        model: type,
        access_context: AccessContext,
        variable_mapping: Optional[Dict[str, Any]] = None,
    ) -> ColumnElement:
        """SQLAlchemy OR 표현식 생성."""
        conditions = []
        for cond in self.conditions:
            if isinstance(cond, FilterExpression):
                conditions.append(cond.to_sqlalchemy(model, access_context, variable_mapping))
            else:  # FilterCondition
                conditions.append(
                    _condition_to_sqlalchemy(model, cond, access_context, variable_mapping)
                )
        return or_(*conditions)

    def __repr__(self) -> str:
        return f"<OrExpression(conditions={self.conditions})>"


def _parse_filter_expression(data: Dict[str, Any]) -> FilterExpression:
    """FilterExpression 재귀 파싱."""
    if "and" in data:
        return AndExpression.from_dict(data["and"])
    elif "or" in data:
        return OrExpression.from_dict(data["or"])
    else:
        raise BadRequestError("Invalid FilterExpression: must contain 'and' or 'or'")


def _substitute_context_variables(
    value: Any,
    access_context: AccessContext,
) -> Any:
    """$ctx 변수 치환 (화이트리스트 검증).
    
    Args:
        value: 원본 값 (리터럴 또는 "$ctx.variable")
        access_context: 접근 컨텍스트 (변수 소스)
    
    Returns:
        치환된 값
    
    Raises:
        BadRequestError: 미허용 변수 또는 존재하지 않는 컨텍스트 필드
    """
    if not isinstance(value, str) or not value.startswith("$ctx."):
        return value  # 리터럴 값

    # $ctx. 접두사 제거
    var_name = value[5:]  # Remove "$ctx."

    # 화이트리스트 검증 (S2 원칙 ⑥)
    WHITELIST_VARIABLES = {
        "organization_id",
        "user_id",
        "team_id",
        "scope_profile_id",
    }

    if var_name not in WHITELIST_VARIABLES:
        raise BadRequestError(
            f"Context variable '${'{'}ctx.{var_name}${'}'} is not allowed. "
            f"Allowed: {', '.join(WHITELIST_VARIABLES)}"
        )

    # access_context에서 변수 추출
    if var_name == "organization_id":
        return access_context.organization_id
    elif var_name == "user_id":
        return access_context.user_id
    elif var_name == "team_id":
        return getattr(access_context, "team_id", None)  # 선택 필드
    elif var_name == "scope_profile_id":
        return access_context.scope_profile_id

    raise BadRequestError(f"Context variable not found: ${'{'}ctx.{var_name}${'}'}")


def _condition_to_sqlalchemy(
    model: type,
    condition: FilterCondition,
    access_context: AccessContext,
    variable_mapping: Optional[Dict[str, Any]] = None,
) -> ColumnElement:
    """FilterCondition을 SQLAlchemy 표현식으로 변환.
    
    Args:
        model: SQLAlchemy ORM 모델
        condition: FilterCondition
        access_context: 접근 컨텍스트 ($ctx 치환용)
        variable_mapping: 추가 변수 매핑 (선택)
    
    Returns:
        SQLAlchemy ColumnElement (WHERE 절)
    """
    # 모델에서 필드 조회
    if not hasattr(model, condition.field):
        raise BadRequestError(f"Model {model.__name__} has no field '{condition.field}'")

    column = getattr(model, condition.field)

    # $ctx 변수 치환
    value = _substitute_context_variables(condition.value, access_context)

    # 연산자 적용
    if condition.op == "eq":
        return column == value
    elif condition.op == "neq":
        return column != value
    elif condition.op == "in":
        if not isinstance(value, list):
            raise BadRequestError(f"'in' operator requires list value, got {type(value)}")
        return column.in_(value)
    elif condition.op == "not_in":
        if not isinstance(value, list):
            raise BadRequestError(f"'not_in' operator requires list value, got {type(value)}")
        return ~column.in_(value)
    elif condition.op == "contains":
        # JSONB contains 또는 문자열 contains
        if hasattr(column.type, "python_type") and column.type.python_type == str:
            return column.contains(value)
        else:
            # JSONB 타입
            return column.astext.contains(value)
    else:
        raise BadRequestError(f"Unsupported operator: {condition.op}")


class FilterParser:
    """FilterExpression 파서 (공개 인터페이스)."""

    @staticmethod
    def parse(
        filter_dict: Dict[str, Any],
        model: type,
        access_context: AccessContext,
    ) -> ColumnElement:
        """FilterExpression 파싱 및 SQLAlchemy WHERE 절 생성.
        
        Args:
            filter_dict: FilterExpression JSON (and/or 최상위 필요)
            model: SQLAlchemy ORM 모델
            access_context: 접근 컨텍스트
        
        Returns:
            SQLAlchemy WHERE 절 ColumnElement
        
        Example:
            >>> filter_dict = {
            ...     "and": [
            ...         {"field": "organization_id", "op": "eq", "value": "$ctx.organization_id"},
            ...         {"field": "owner_id", "op": "eq", "value": "$ctx.user_id"},
            ...     ]
            ... }
            >>> where_clause = FilterParser.parse(filter_dict, Document, access_context)
            >>> stmt = select(Document).where(where_clause)
        """
        if not isinstance(filter_dict, dict):
            raise BadRequestError("FilterExpression must be a dictionary")

        expression = _parse_filter_expression(filter_dict)
        return expression.to_sqlalchemy(model, access_context)
```

### 4-3. Alembic 마이그레이션 스크립트

**파일:** `/backend/alembic/versions/0011_create_scope_profiles.py`

```python
"""Create scope_profiles and scope_definitions tables.

Revision ID: 0011
Revises: 0010
Create Date: 2026-04-17 00:00:00.000000

"""

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# Revision identifiers
revision = "0011"
down_revision = "0010"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # Create scope_profiles table
    op.create_table(
        "scope_profiles",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("organization_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("name", sa.String(256), nullable=False),
        sa.Column("description", sa.String(512), nullable=True),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index("ix_scope_profiles_organization_created", "scope_profiles", ["organization_id", "created_at"])

    # Create scope_definitions table
    op.create_table(
        "scope_definitions",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("scope_profile_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("scope_name", sa.String(256), nullable=False),
        sa.Column("acl_filter", postgresql.JSONB(astext_type=sa.Text()), server_default=sa.text("'{}'::jsonb"), nullable=False),
        sa.Column("description", sa.String(512), nullable=True),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.ForeignKeyConstraint(["scope_profile_id"], ["scope_profiles.id"], ondelete="CASCADE"),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index("ix_scope_definitions_scope_profile_name", "scope_definitions", ["scope_profile_id", "scope_name"], unique=True)


def downgrade() -> None:
    op.drop_index("ix_scope_definitions_scope_profile_name", table_name="scope_definitions")
    op.drop_table("scope_definitions")

    op.drop_index("ix_scope_profiles_organization_created", table_name="scope_profiles")
    op.drop_table("scope_profiles")
```

### 4-4. Repository 패턴

**파일:** `/backend/app/repositories/scope_profile_repository.py`

```python
"""Scope Profile 및 Scope Definition 저장소."""

from __future__ import annotations

from typing import Optional, List, Dict, Any
from uuid import UUID

from sqlalchemy import and_, select, desc
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.scope_profile import ScopeProfile, ScopeDefinition


class ScopeProfileRepository:
    """Scope Profile 조회 및 관리."""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_id(self, scope_profile_id: UUID) -> Optional[ScopeProfile]:
        """ID로 Scope Profile 조회."""
        stmt = select(ScopeProfile).where(ScopeProfile.id == scope_profile_id)
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()

    async def list_by_organization(
        self,
        organization_id: UUID,
        limit: int = 50,
        offset: int = 0,
    ) -> tuple[List[ScopeProfile], int]:
        """조직별 Scope Profile 목록 조회."""
        filters = [ScopeProfile.organization_id == organization_id]

        count_stmt = select(func.count(ScopeProfile.id)).where(and_(*filters))
        count_result = await self.db.execute(count_stmt)
        total = count_result.scalar()

        stmt = (
            select(ScopeProfile)
            .where(and_(*filters))
            .order_by(desc(ScopeProfile.created_at))
            .limit(limit)
            .offset(offset)
        )
        result = await self.db.execute(stmt)
        profiles = result.scalars().all()

        return profiles, total

    async def create(
        self,
        organization_id: UUID,
        name: str,
        description: Optional[str] = None,
    ) -> ScopeProfile:
        """새 Scope Profile 생성."""
        profile = ScopeProfile(
            organization_id=organization_id,
            name=name,
            description=description,
        )
        self.db.add(profile)
        await self.db.flush()
        return profile


class ScopeDefinitionRepository:
    """Scope Definition 조회 및 관리."""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_by_id(self, scope_definition_id: UUID) -> Optional[ScopeDefinition]:
        """ID로 Scope Definition 조회."""
        stmt = select(ScopeDefinition).where(ScopeDefinition.id == scope_definition_id)
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()

    async def get_by_scope_name(
        self,
        scope_profile_id: UUID,
        scope_name: str,
    ) -> Optional[ScopeDefinition]:
        """Scope Profile 내에서 scope_name으로 정의 조회."""
        stmt = select(ScopeDefinition).where(
            and_(
                ScopeDefinition.scope_profile_id == scope_profile_id,
                ScopeDefinition.scope_name == scope_name,
            )
        )
        result = await self.db.execute(stmt)
        return result.scalar_one_or_none()

    async def list_by_scope_profile(
        self,
        scope_profile_id: UUID,
    ) -> List[ScopeDefinition]:
        """Scope Profile의 모든 정의 조회."""
        stmt = select(ScopeDefinition).where(
            ScopeDefinition.scope_profile_id == scope_profile_id
        )
        result = await self.db.execute(stmt)
        return result.scalars().all()

    async def create(
        self,
        scope_profile_id: UUID,
        scope_name: str,
        acl_filter: Dict[str, Any],
        description: Optional[str] = None,
    ) -> ScopeDefinition:
        """새 Scope Definition 생성."""
        definition = ScopeDefinition(
            scope_profile_id=scope_profile_id,
            scope_name=scope_name,
            acl_filter=acl_filter,
            description=description,
        )
        self.db.add(definition)
        await self.db.flush()
        return definition
```

### 4-5. 단위 테스트

**파일:** `/backend/tests/unit/models/test_filter_expression.py`

```python
"""FilterExpression 파서 단위 테스트."""

import pytest
from uuid import uuid4

from app.models.filter_expression import (
    FilterCondition,
    FilterParser,
    AndExpression,
    OrExpression,
    _substitute_context_variables,
)
from app.models.access_context import AccessContext
from app.models.document import Document
from app.exceptions import BadRequestError


def test_filter_condition_creation():
    """FilterCondition 생성."""
    cond = FilterCondition(
        field="organization_id",
        op="eq",
        value="some-value"
    )

    assert cond.field == "organization_id"
    assert cond.op == "eq"
    assert cond.value == "some-value"


def test_filter_condition_invalid_operator():
    """유효하지 않은 연산자 검증."""
    with pytest.raises(BadRequestError):
        FilterCondition(
            field="organization_id",
            op="invalid_op",
            value="value"
        )


def test_context_variable_substitution():
    """$ctx 변수 치환."""
    org_id = uuid4()
    user_id = uuid4()

    access_context = AccessContext(
        user_id=user_id,
        organization_id=org_id,
        principal_type="user",
    )

    # $ctx.organization_id 치환
    result = _substitute_context_variables("$ctx.organization_id", access_context)
    assert result == org_id

    # $ctx.user_id 치환
    result = _substitute_context_variables("$ctx.user_id", access_context)
    assert result == user_id

    # 리터럴 값 (치환 없음)
    result = _substitute_context_variables("literal-value", access_context)
    assert result == "literal-value"


def test_context_variable_whitelist_violation():
    """미허용 변수 거부 (인젝션 방어)."""
    access_context = AccessContext(
        user_id=uuid4(),
        organization_id=uuid4(),
        principal_type="user",
    )

    # 미허용 변수
    with pytest.raises(BadRequestError) as exc_info:
        _substitute_context_variables("$ctx.password", access_context)

    assert "not allowed" in str(exc_info.value)


def test_filter_parser_simple_and():
    """간단한 AND 필터 파싱."""
    org_id = uuid4()
    user_id = uuid4()

    access_context = AccessContext(
        user_id=user_id,
        organization_id=org_id,
        principal_type="user",
    )

    filter_dict = {
        "and": [
            {"field": "organization_id", "op": "eq", "value": "$ctx.organization_id"},
            {"field": "owner_id", "op": "eq", "value": "$ctx.user_id"},
        ]
    }

    # 파서는 WHERE 절을 생성할 수 있어야 함
    where_clause = FilterParser.parse(filter_dict, Document, access_context)
    assert where_clause is not None


def test_filter_parser_complex_or():
    """복잡한 OR 필터 파싱."""
    org_id = uuid4()
    user_id = uuid4()

    access_context = AccessContext(
        user_id=user_id,
        organization_id=org_id,
        principal_type="user",
    )

    filter_dict = {
        "or": [
            {"field": "owner_id", "op": "eq", "value": "$ctx.user_id"},
            {"field": "access_level", "op": "eq", "value": "public"},
        ]
    }

    where_clause = FilterParser.parse(filter_dict, Document, access_context)
    assert where_clause is not None


def test_filter_parser_nested_and_or():
    """중첩된 AND/OR 필터 파싱."""
    org_id = uuid4()
    user_id = uuid4()

    access_context = AccessContext(
        user_id=user_id,
        organization_id=org_id,
        principal_type="user",
    )

    filter_dict = {
        "and": [
            {"field": "organization_id", "op": "eq", "value": "$ctx.organization_id"},
            {
                "or": [
                    {"field": "owner_id", "op": "eq", "value": "$ctx.user_id"},
                    {"field": "access_level", "op": "eq", "value": "public"},
                ]
            }
        ]
    }

    where_clause = FilterParser.parse(filter_dict, Document, access_context)
    assert where_clause is not None


def test_filter_parser_in_operator():
    """in 연산자 테스트."""
    access_context = AccessContext(
        user_id=uuid4(),
        organization_id=uuid4(),
        principal_type="user",
    )

    filter_dict = {
        "and": [
            {
                "field": "document_type",
                "op": "in",
                "value": ["memo", "report", "guideline"]
            }
        ]
    }

    where_clause = FilterParser.parse(filter_dict, Document, access_context)
    assert where_clause is not None


def test_filter_parser_in_operator_requires_list():
    """in 연산자는 리스트 값 필요."""
    access_context = AccessContext(
        user_id=uuid4(),
        organization_id=uuid4(),
        principal_type="user",
    )

    filter_dict = {
        "and": [
            {
                "field": "document_type",
                "op": "in",
                "value": "memo"  # 잘못된 타입
            }
        ]
    }

    with pytest.raises(BadRequestError):
        FilterParser.parse(filter_dict, Document, access_context)


def test_filter_parser_invalid_field():
    """존재하지 않는 필드 거부."""
    access_context = AccessContext(
        user_id=uuid4(),
        organization_id=uuid4(),
        principal_type="user",
    )

    filter_dict = {
        "and": [
            {
                "field": "nonexistent_field",
                "op": "eq",
                "value": "value"
            }
        ]
    }

    with pytest.raises(BadRequestError):
        FilterParser.parse(filter_dict, Document, access_context)
```

### 4-6. 단위 테스트 (Scope Profile 모델)

**파일:** `/backend/tests/unit/models/test_scope_profile_models.py`

```python
"""Scope Profile 도메인 모델 단위 테스트."""

import pytest
from uuid import uuid4

from sqlalchemy.ext.asyncio import AsyncSession

from app.models.scope_profile import ScopeProfile, ScopeDefinition


@pytest.mark.asyncio
async def test_scope_profile_creation(db_session: AsyncSession):
    """Scope Profile 생성."""
    org_id = uuid4()

    profile = ScopeProfile(
        organization_id=org_id,
        name="team-a-reader",
        description="Team A 문서 읽기 프로필",
    )

    db_session.add(profile)
    await db_session.flush()

    assert profile.id is not None
    assert profile.organization_id == org_id


@pytest.mark.asyncio
async def test_scope_definition_creation(db_session: AsyncSession):
    """Scope Definition 생성."""
    org_id = uuid4()

    profile = ScopeProfile(
        organization_id=org_id,
        name="test-profile",
    )
    db_session.add(profile)
    await db_session.flush()

    definition = ScopeDefinition(
        scope_profile_id=profile.id,
        scope_name="team_search",
        acl_filter={
            "and": [
                {"field": "organization_id", "op": "eq", "value": "$ctx.organization_id"},
            ]
        },
    )
    db_session.add(definition)
    await db_session.flush()

    assert definition.id is not None
    assert definition.scope_name == "team_search"
    assert definition.acl_filter is not None


@pytest.mark.asyncio
async def test_scope_definition_cascade_delete(db_session: AsyncSession):
    """Scope Profile 삭제 시 ScopeDefinition도 삭제."""
    org_id = uuid4()

    profile = ScopeProfile(
        organization_id=org_id,
        name="test-profile",
    )
    db_session.add(profile)
    await db_session.flush()

    definition = ScopeDefinition(
        scope_profile_id=profile.id,
        scope_name="test_scope",
        acl_filter={},
    )
    db_session.add(definition)
    await db_session.flush()

    definition_id = definition.id

    # Profile 삭제
    await db_session.delete(profile)
    await db_session.flush()

    # Definition도 삭제됨 (cascade)
    deleted_definition = await db_session.get(ScopeDefinition, definition_id)
    assert deleted_definition is None
```

## 5. 산출물

1. **Scope Profile 도메인 모델** (`/backend/app/models/scope_profile.py`)
   - ScopeProfile, ScopeDefinition 클래스 정의
   - 일대다 관계 설정

2. **FilterExpression 파서** (`/backend/app/models/filter_expression.py`)
   - FilterCondition, AndExpression, OrExpression 클래스
   - FilterParser 공개 인터페이스
   - $ctx 변수 치환 로직 (화이트리스트 검증)
   - SQLAlchemy WHERE 절 변환

3. **Alembic 마이그레이션 스크립트** (`/backend/alembic/versions/0011_*.py`)
   - scope_profiles, scope_definitions 테이블 생성
   - 외래키 및 인덱스 정의

4. **Repository 계층** (`/backend/app/repositories/scope_profile_repository.py`)
   - ScopeProfileRepository, ScopeDefinitionRepository
   - 조회, 생성 메서드

5. **단위 테스트** (2개 파일)
   - FilterExpression 파서 정확성 테스트
   - $ctx 변수 치환 및 인젝션 방어 테스트
   - 복잡한 AND/OR 조건 테스트
   - Scope Profile 모델 테스트

## 6. 완료 기준

1. ScopeProfile, ScopeDefinition 모델이 정규화되었는가?
2. FilterExpression 파서가 and/or 조건을 올바르게 파싱하는가?
3. $ctx 변수 치환이 화이트리스트를 검증하고 인젝션을 방어하는가? (S2 원칙 ⑥)
4. FilterParser.parse()가 SQLAlchemy WHERE 절을 정확하게 생성하는가?
5. "in", "not_in", "contains" 연산자가 올바르게 처리되는가?
6. Alembic 마이그레이션이 정상 작동하는가 (upgrade/downgrade)?
7. 모든 단위 테스트가 통과하는가 (`pytest -v`)?
8. mypy 타입 검사를 통과하는가?
9. 복잡한 중첩 조건 (AND + OR)도 올바르게 처리되는가?
10. 접근 범위가 완전히 하드코딩 없이 Scope Profile로 관리되는가? (S2 원칙 ⑥)

## 7. 작업 지침

### 지침 7-1. S2 원칙 ⑥ (Scope Profile 기반 ACL)

접근 범위는 절대 코드에 하드코딩하지 않는다. 모든 ACL 필터링은 Scope Profile의 FilterExpression을 통해 동적으로 적용된다.

```python
# 하드코딩 금지:
if organization_id == current_org and owner_id == user_id:  # X
    return results

# 올바른 방법:
scope_definition = await repo.get_by_scope_name(scope_profile_id, "team_search")
where_clause = FilterParser.parse(scope_definition.acl_filter, Document, access_context)
stmt = select(Document).where(where_clause)
```

### 지침 7-2. $ctx 변수 화이트리스트

$ctx 변수는 사전 정의된 화이트리스트에만 포함될 수 있다. 이는 SQL 인젝션 및 임의의 컨텍스트 정보 노출을 방지한다.

```python
WHITELIST_VARIABLES = {
    "organization_id",
    "user_id",
    "team_id",
    "scope_profile_id",
}

# 허용되지 않는 변수 시도
# "$ctx.password" → BadRequestError
# "$ctx.db_password" → BadRequestError
```

### 지침 7-3. FilterExpression 재귀 파싱

and/or는 중첩될 수 있다. 재귀적 파싱을 통해 복잡한 조건을 지원한다.

```json
{
  "and": [
    {"field": "organization_id", "op": "eq", "value": "$ctx.organization_id"},
    {
      "or": [
        {"field": "owner_id", "op": "eq", "value": "$ctx.user_id"},
        {"field": "access_level", "op": "eq", "value": "public"},
        {
          "and": [
            {"field": "document_type", "op": "eq", "value": "guideline"},
            {"field": "status", "op": "eq", "value": "published"}
          ]
        }
      ]
    }
  ]
}
```

### 지침 7-4. SQLAlchemy 바인드 파라미터

FilterExpression의 리터럴 값은 SQLAlchemy의 바인드 파라미터로 전달되어야 한다. 문자열 연결을 피한다.

```python
# 좋은 예 (SQLAlchemy가 바인드 파라미터 처리)
column == value  # SQLAlchemy가 자동으로 바인드

# 나쁜 예 (SQL 인젝션 위험)
f"WHERE {column} = '{value}'"  # X
```

### 지침 7-5. 연산자별 값 타입 검증

각 연산자는 특정 값 타입을 기대한다. 파서는 타입을 검증하고 오류 시 BadRequestError를 발생시킨다.

```python
# "in" 연산자: 리스트 값 필수
{"field": "type", "op": "in", "value": ["a", "b", "c"]}  # O
{"field": "type", "op": "in", "value": "a"}  # X → BadRequestError

# "eq" 연산자: 모든 값 타입 허용
{"field": "id", "op": "eq", "value": "uuid-string"}  # O
{"field": "count", "op": "eq", "value": 42}  # O
```

### 지침 7-6. 타입 안정성

FilterExpression 파서는 Any 타입을 최소화하고 Literal, Union 등으로 명시적 타입을 지정한다.

### 지침 7-7. Scope Definition 유니크 제약

하나의 ScopeProfile 내에서 같은 scope_name은 중복될 수 없다. DB 스키마에서 (scope_profile_id, scope_name) 복합 유니크 제약을 설정한다.

```python
Index("ix_scope_definitions_scope_profile_name", 
      "scope_profile_id", "scope_name", unique=True)
```
