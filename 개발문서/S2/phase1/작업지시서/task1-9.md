# Task 1-9. Prompt 도메인 모델 및 DB 마이그레이션

## 1. 작업 목적

프롬프트 템플릿을 데이터베이스에서 버전 관리하기 위한 도메인 모델(Prompt, PromptVersion)을 설계하고, SQLAlchemy ORM 기반 모델 정의 및 Alembic 마이그레이션 스크립트를 작성한다. 이는 런타임 로더(Task 1-11)와 API(Task 1-10)의 기반이 된다.

## 2. 작업 범위

### 포함 범위

1. **SQLAlchemy ORM 모델 설계**
   - `Prompt` 엔티티: UUID id, name(unique), description, category enum, template text, variables (JSON), created_at, updated_at, created_by, is_deleted
   - `PromptVersion` 엔티티: UUID id, prompt_id FK, version int(auto-increment), template, is_active bool, created_at, usage_count, average_quality_score
   - 관계 설정 (Prompt → PromptVersion: 1:N)
   - 인덱스: (prompt_id, is_active), (name), (category)

2. **Repository 패턴 구현**
   - `PromptRepository` 클래스: CRUD + 버전 관리 메서드
   - `create()`: Prompt 생성 + v1 PromptVersion 자동 생성
   - `update()`: Prompt 수정 + 새 PromptVersion 자동 생성
   - `get_active_version()`: 활성 버전 조회
   - `get_version()`: 특정 버전 조회
   - `list_versions()`: 버전 이력 조회
   - `activate_version()`: 특정 버전을 활성화
   - `soft_delete()`: Soft delete 처리

3. **Alembic 마이그레이션 스크립트**
   - 초기 스키마 생성 migration
   - `alembic/versions/` 디렉토리에 스크립트 위치

4. **변수 자동 추출 유틸리티**
   - `extract_variables()`: Jinja2 템플릿에서 변수 추출 (regex: `{{ var_name }}`)
   - 반환값: List[str]

### 제외 범위

- API endpoint 구현 (Task 1-10)
- 캐싱 로직 (Task 1-11)
- 프롬프트 렌더링 (Task 1-11)
- 시드 데이터 (Task 1-12)

## 3. 선행 조건

- Phase 0 완료 (FastAPI, SQLAlchemy, Alembic 설정됨)
- PostgreSQL 또는 SQLite 개발용 DB 접근 가능
- `/backend/app/models/`, `/backend/app/repositories/`, `/backend/app/db/` 디렉토리 접근
- 기존 audit_logs 테이블 존재 (Phase 0에서 생성됨)

## 4. 주요 작업 항목

### 4-1. SQLAlchemy 모델 정의

**파일:** `/backend/app/models/prompt.py`

**작업 항목:**

```python
import uuid
from datetime import datetime
from typing import List, Optional
from enum import Enum
from sqlalchemy import (
    Column, String, Text, DateTime, Boolean, Integer, Float, ForeignKey,
    Index, UniqueConstraint, event
)
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import declarative_base, relationship

Base = declarative_base()

class PromptCategory(str, Enum):
    """프롬프트 카테고리"""
    RETRIEVAL = "retrieval"                          # 검색 질의 재작성
    EVALUATION = "evaluation"                        # 품질 평가
    EXTRACTION = "extraction"                        # 구조화 추출
    CONVERSATION = "conversation"                    # 대화형 프롬프트
    INJECTION_DETECTION = "injection_detection"      # 보안 (인젝션 탐지)

class Prompt(Base):
    """프롬프트 템플릿 엔티티"""
    __tablename__ = "prompts"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(255), unique=True, nullable=False, index=True)
    description = Column(Text, nullable=True)
    category = Column(String(50), nullable=False, index=True)  # PromptCategory enum 값
    template = Column(Text, nullable=False)          # 기본 템플릿 (최신 버전)
    variables = Column(JSONB, nullable=False, default=list)  # ["var1", "var2", ...]
    created_at = Column(DateTime, default=datetime.utcnow, nullable=False)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    created_by = Column(String(255), nullable=False)  # User ID or "system"
    is_deleted = Column(Boolean, default=False, nullable=False, index=True)
    
    # 관계
    versions = relationship("PromptVersion", back_populates="prompt", cascade="all, delete-orphan")
    
    __table_args__ = (
        Index("ix_prompts_category_deleted", "category", "is_deleted"),
        Index("ix_prompts_name_deleted", "name", "is_deleted"),
    )

class PromptVersion(Base):
    """프롬프트 버전 이력"""
    __tablename__ = "prompt_versions"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    prompt_id = Column(UUID(as_uuid=True), ForeignKey("prompts.id"), nullable=False)
    version = Column(Integer, nullable=False)         # 1, 2, 3, ...
    template = Column(Text, nullable=False)           # 이 버전의 템플릿
    is_active = Column(Boolean, default=False, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow, nullable=False)
    usage_count = Column(Integer, default=0, nullable=False)  # 통계용
    average_quality_score = Column(Float, nullable=True)  # Phase 7 연계
    
    # 관계
    prompt = relationship("Prompt", back_populates="versions")
    
    __table_args__ = (
        UniqueConstraint("prompt_id", "version", name="uq_prompt_version"),
        Index("ix_prompt_versions_active", "prompt_id", "is_active"),
        Index("ix_prompt_versions_created", "created_at"),
    )

    def __repr__(self):
        return f"<PromptVersion prompt={self.prompt_id} v{self.version}>"
```

### 4-2. Repository 패턴 구현

**파일:** `/backend/app/repositories/prompt_repository.py`

**작업 항목:**

```python
import uuid
from typing import List, Optional, Dict, Any
from datetime import datetime
from sqlalchemy.orm import Session
from sqlalchemy import and_, desc

from backend.app.models.prompt import Prompt, PromptVersion, PromptCategory
from backend.app.services.prompt.utils import extract_variables

class PromptRepository:
    """프롬프트 저장소 (CRUD + 버전 관리)"""
    
    def __init__(self, session: Session):
        self.session = session
    
    def create(self, name: str, description: str, category: str, 
               template: str, created_by: str) -> Prompt:
        """
        프롬프트 생성 (v1 버전 자동 생성)
        
        Args:
            name: 프롬프트 이름 (unique)
            description: 설명
            category: 카테고리 (PromptCategory enum 값)
            template: Jinja2 템플릿 텍스트
            created_by: 생성자 ID
        
        Returns:
            생성된 Prompt 객체
        """
        variables = extract_variables(template)
        
        prompt = Prompt(
            id=uuid.uuid4(),
            name=name,
            description=description,
            category=category,
            template=template,
            variables=variables,
            created_by=created_by,
            is_deleted=False
        )
        
        # v1 버전 자동 생성
        version = PromptVersion(
            id=uuid.uuid4(),
            prompt_id=prompt.id,
            version=1,
            template=template,
            is_active=True,
            usage_count=0
        )
        
        prompt.versions.append(version)
        self.session.add(prompt)
        self.session.flush()
        return prompt
    
    def get_by_id(self, prompt_id: str) -> Optional[Prompt]:
        """ID로 프롬프트 조회 (삭제되지 않은 것만)"""
        return self.session.query(Prompt).filter(
            and_(Prompt.id == uuid.UUID(prompt_id), Prompt.is_deleted == False)
        ).first()
    
    def get_by_name(self, name: str) -> Optional[Prompt]:
        """이름으로 프롬프트 조회"""
        return self.session.query(Prompt).filter(
            and_(Prompt.name == name, Prompt.is_deleted == False)
        ).first()
    
    def list_all(self, category: Optional[str] = None, 
                 skip: int = 0, limit: int = 100) -> List[Prompt]:
        """프롬프트 목록 조회 (페이지네이션, 카테고리 필터)"""
        query = self.session.query(Prompt).filter(Prompt.is_deleted == False)
        
        if category:
            query = query.filter(Prompt.category == category)
        
        return query.offset(skip).limit(limit).all()
    
    def get_active_version(self, prompt_id: str) -> Optional[PromptVersion]:
        """활성 버전 조회"""
        return self.session.query(PromptVersion).filter(
            and_(
                PromptVersion.prompt_id == uuid.UUID(prompt_id),
                PromptVersion.is_active == True
            )
        ).first()
    
    def get_version(self, prompt_id: str, version: int) -> Optional[PromptVersion]:
        """특정 버전 조회"""
        return self.session.query(PromptVersion).filter(
            and_(
                PromptVersion.prompt_id == uuid.UUID(prompt_id),
                PromptVersion.version == version
            )
        ).first()
    
    def list_versions(self, prompt_id: str) -> List[PromptVersion]:
        """버전 이력 조회 (생성 시간 역순)"""
        return self.session.query(PromptVersion).filter(
            PromptVersion.prompt_id == uuid.UUID(prompt_id)
        ).order_by(desc(PromptVersion.version)).all()
    
    def update(self, prompt_id: str, template: str, 
               description: Optional[str] = None) -> Prompt:
        """
        프롬프트 수정 (새 버전 자동 생성)
        
        Args:
            prompt_id: 프롬프트 ID
            template: 새 템플릿
            description: 설명 (선택)
        
        Returns:
            수정된 Prompt 객체
        """
        prompt = self.get_by_id(prompt_id)
        if not prompt:
            raise ValueError(f"Prompt not found: {prompt_id}")
        
        # 이전 활성 버전 비활성화
        old_version = self.get_active_version(prompt_id)
        if old_version:
            old_version.is_active = False
        
        # 새 버전 생성
        new_version_num = len(prompt.versions) + 1
        variables = extract_variables(template)
        
        new_version = PromptVersion(
            id=uuid.uuid4(),
            prompt_id=prompt.id,
            version=new_version_num,
            template=template,
            is_active=True,
            usage_count=0
        )
        
        prompt.template = template
        prompt.variables = variables
        prompt.updated_at = datetime.utcnow()
        prompt.versions.append(new_version)
        
        self.session.flush()
        return prompt
    
    def activate_version(self, prompt_id: str, version: int) -> PromptVersion:
        """특정 버전을 활성화"""
        prompt = self.get_by_id(prompt_id)
        if not prompt:
            raise ValueError(f"Prompt not found: {prompt_id}")
        
        # 현재 활성 버전 비활성화
        current_active = self.get_active_version(prompt_id)
        if current_active:
            current_active.is_active = False
        
        # 새 활성 버전 설정
        target_version = self.get_version(prompt_id, version)
        if not target_version:
            raise ValueError(f"Version not found: {prompt_id} v{version}")
        
        target_version.is_active = True
        prompt.template = target_version.template
        prompt.updated_at = datetime.utcnow()
        
        self.session.flush()
        return target_version
    
    def soft_delete(self, prompt_id: str) -> Prompt:
        """Soft delete"""
        prompt = self.get_by_id(prompt_id)
        if not prompt:
            raise ValueError(f"Prompt not found: {prompt_id}")
        
        prompt.is_deleted = True
        prompt.updated_at = datetime.utcnow()
        self.session.flush()
        return prompt
    
    def increment_usage_count(self, version_id: str) -> None:
        """버전의 사용 횟수 증가 (렌더링 시 호출)"""
        version = self.session.query(PromptVersion).filter(
            PromptVersion.id == uuid.UUID(version_id)
        ).first()
        if version:
            version.usage_count += 1
            self.session.flush()
```

### 4-3. 변수 추출 유틸리티

**파일:** `/backend/app/services/prompt/utils.py`

**작업 항목:**

```python
import re
from typing import List

def extract_variables(template: str) -> List[str]:
    """
    Jinja2 템플릿에서 변수 추출
    
    예: "{{ var1 }} and {{ var2 }}" → ["var1", "var2"]
    
    Args:
        template: Jinja2 템플릿 문자열
    
    Returns:
        변수 이름 리스트 (중복 제거)
    """
    # Jinja2 변수 패턴: {{ var_name }}
    pattern = r'\{\{\s*(\w+)\s*\}\}'
    matches = re.findall(pattern, template)
    
    # 중복 제거 및 정렬
    return sorted(list(set(matches)))
```

### 4-4. Alembic 마이그레이션 스크립트

**파일:** `/backend/app/db/alembic/versions/20260416_001_create_prompt_tables.py`

**작업 항목:**

```python
"""Create prompt and prompt_version tables

Revision ID: 20260416_001
Revises:
Create Date: 2026-04-16 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

# revision identifiers, used by Alembic.
revision = '20260416_001'
down_revision = None
branch_labels = None
depends_on = None

def upgrade() -> None:
    # prompts 테이블 생성
    op.create_table(
        'prompts',
        sa.Column('id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('name', sa.String(255), nullable=False),
        sa.Column('description', sa.Text(), nullable=True),
        sa.Column('category', sa.String(50), nullable=False),
        sa.Column('template', sa.Text(), nullable=False),
        sa.Column('variables', postgresql.JSONB(), nullable=False, server_default='[]'),
        sa.Column('created_at', sa.DateTime(), nullable=False),
        sa.Column('updated_at', sa.DateTime(), nullable=False),
        sa.Column('created_by', sa.String(255), nullable=False),
        sa.Column('is_deleted', sa.Boolean(), nullable=False, server_default='false'),
        sa.PrimaryKeyConstraint('id')
    )
    
    # prompts 인덱스
    op.create_index('ix_prompts_name', 'prompts', ['name'])
    op.create_index('ix_prompts_category', 'prompts', ['category'])
    op.create_index('ix_prompts_category_deleted', 'prompts', ['category', 'is_deleted'])
    op.create_index('ix_prompts_name_deleted', 'prompts', ['name', 'is_deleted'])
    op.create_unique_constraint('uq_prompts_name', 'prompts', ['name'])
    
    # prompt_versions 테이블 생성
    op.create_table(
        'prompt_versions',
        sa.Column('id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('prompt_id', postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column('version', sa.Integer(), nullable=False),
        sa.Column('template', sa.Text(), nullable=False),
        sa.Column('is_active', sa.Boolean(), nullable=False, server_default='false'),
        sa.Column('created_at', sa.DateTime(), nullable=False),
        sa.Column('usage_count', sa.Integer(), nullable=False, server_default='0'),
        sa.Column('average_quality_score', sa.Float(), nullable=True),
        sa.ForeignKeyConstraint(['prompt_id'], ['prompts.id'], ),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('prompt_id', 'version', name='uq_prompt_version')
    )
    
    # prompt_versions 인덱스
    op.create_index('ix_prompt_versions_active', 'prompt_versions', ['prompt_id', 'is_active'])
    op.create_index('ix_prompt_versions_created', 'prompt_versions', ['created_at'])

def downgrade() -> None:
    op.drop_index('ix_prompt_versions_created', table_name='prompt_versions')
    op.drop_index('ix_prompt_versions_active', table_name='prompt_versions')
    op.drop_table('prompt_versions')
    
    op.drop_constraint('uq_prompts_name', 'prompts', type_='unique')
    op.drop_index('ix_prompts_name_deleted', table_name='prompts')
    op.drop_index('ix_prompts_category_deleted', table_name='prompts')
    op.drop_index('ix_prompts_category', table_name='prompts')
    op.drop_index('ix_prompts_name', table_name='prompts')
    op.drop_table('prompts')
```

## 5. 산출물

1. **SQLAlchemy ORM 모델** (`/backend/app/models/prompt.py`)
   - Prompt 엔티티 (id, name, description, category, template, variables, 타임스탬프, soft delete)
   - PromptVersion 엔티티 (버전 관리, is_active, usage_count, quality_score)
   - 관계 설정 및 인덱스

2. **Repository 패턴** (`/backend/app/repositories/prompt_repository.py`)
   - PromptRepository 클래스
   - create(), get_by_id(), get_by_name(), list_all() CRUD
   - get_active_version(), get_version(), list_versions()
   - update() (자동 버전 생성), activate_version()
   - soft_delete(), increment_usage_count()

3. **유틸리티 함수** (`/backend/app/services/prompt/utils.py`)
   - extract_variables(): Jinja2 템플릿에서 변수 자동 추출

4. **Alembic 마이그레이션** (`/backend/app/db/alembic/versions/...`)
   - 초기 스키마 생성 스크립트

## 6. 완료 기준

1. SQLAlchemy 모델이 정의되고, 테이블 관계가 올바름
2. Repository CRUD 메서드가 모두 구현됨
3. 버전 자동 생성 로직이 정상 작동
4. Alembic 마이그레이션 스크립트가 작성되고, upgrade/downgrade 함수가 있음
5. extract_variables() 함수가 Jinja2 `{{ var }}` 패턴을 올바르게 추출
6. 단위 테스트 작성 (create, update, list_versions, activate_version)
7. 마이그레이션 실행 후 테이블이 생성됨 (`alembic upgrade head`)

## 7. 작업 지침

### 지침 7-1. 데이터베이스 설정

PostgreSQL 또는 SQLite 개발 환경에서 마이그레이션을 먼저 실행한다:

```bash
cd /backend
alembic upgrade head
```

### 지침 7-2. 변수 추출 정확성

`extract_variables()` 함수는 다음을 지원해야 한다:

```jinja2
{{ simple }}              # → ["simple"]
{{ with_space }}          # → ["with_space"]
{% for x in list %}{{x}}{% endfor %}  # → ["x", "list"]
```

### 지침 7-3. Soft Delete

Prompt 삭제 시 is_deleted = True만 설정하고 행을 삭제하지 않는다. 모든 조회 쿼리는 `is_deleted == False` 필터를 포함한다.

### 지침 7-4. 버전 관리 규칙

- Prompt 생성 시 자동으로 v1 PromptVersion 생성
- Prompt 수정(update) 시 이전 버전 is_active = False, 새 버전 생성
- 각 PromptVersion은 고유한 (prompt_id, version) 조합
