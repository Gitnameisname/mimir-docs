# Task 1-11. PromptRegistry 런타임 로더 및 Jinja2 템플릿 처리

## 1. 작업 목적

Prompt 데이터베이스에서 활성 버전의 프롬프트 템플릿을 로드하고, Jinja2 템플릿 엔진으로 변수 치환 및 렌더링을 수행하는 런타임 서비스를 구현한다. 메모리 캐싱과 템플릿 인젝션 방어를 포함한다.

## 2. 작업 범위

### 포함 범위

1. **PromptRegistry 클래스 (Singleton)**
   - `load_prompt(name: str) → str`: 활성 버전 템플릿 로드
   - `render(name: str, **variables) → str`: 프롬프트 렌더링 (변수 치환)
   - `load_version(name: str, version: int) → str`: 특정 버전 로드
   - `list_variables(name: str) → List[str]`: 템플릿의 변수 목록 조회
   - `invalidate_cache(name: str)`: 캐시 무효화 (DB 변경 시)

2. **Jinja2 템플릿 엔진**
   - SandboxedEnvironment 사용 (보안)
   - 기본 필터/함수 지원 (length, join, upper, lower 등)
   - 변수 유효성 검사 (필수 변수 확인)
   - 렌더링 오류 처리 및 구분

3. **메모리 캐싱**
   - TTL 기반 캐시 (기본 1시간)
   - 사용 통계 (usage_count 추적)
   - DB 변경 시 자동 무효화

4. **보안 검증**
   - 템플릿 인젝션 방어 (SandboxedEnvironment)
   - 접근 불가능한 속성 차단 (__class__, __globals__ 등)

5. **오류 처리**
   - PromptNotFoundError: 프롬프트 존재하지 않음
   - VariableNotProvidedError: 필수 변수 누락
   - RenderError: 템플릿 렌더링 실패
   - VersionNotFoundError: 버전 존재하지 않음

### 제외 범위

- 프롬프트 저장소 (Task 1-9)
- API 엔드포인트 (Task 1-10)
- 시드 데이터 (Task 1-12)

## 3. 선행 조건

- Task 1-9 완료 (PromptRepository)
- Task 1-10 완료 (Prompt API)
- Jinja2 라이브러리 설치 (`pip install jinja2`)
- SQLAlchemy Session 접근 가능

## 4. 주요 작업 항목

### 4-1. 템플릿 엔진 구현

**파일:** `/backend/app/services/prompt/template_engine.py`

**작업 항목:**

```python
from jinja2 import Environment, StrictUndefined, TemplateSyntaxError, UndefinedError
from jinja2.sandbox import SandboxedEnvironment
from typing import Dict, Any, List
import re

class TemplateError(Exception):
    """템플릿 처리 기본 예외"""
    pass

class VariableNotProvidedError(TemplateError):
    """필수 변수가 제공되지 않음"""
    pass

class RenderError(TemplateError):
    """렌더링 실패"""
    pass

class JinjaTemplateEngine:
    """Jinja2 기반 샌드박스 템플릿 엔진"""
    
    def __init__(self, strict_undefined: bool = True):
        """
        Args:
            strict_undefined: True면 undefined 변수 사용 시 에러
        """
        undefined_behavior = StrictUndefined if strict_undefined else None
        
        self.env = SandboxedEnvironment(
            undefined=undefined_behavior,
            trim_blocks=True,
            lstrip_blocks=True
        )
        
        # 기본 필터 추가 (Jinja2 내장 필터는 이미 포함)
        self.env.filters['upper'] = str.upper
        self.env.filters['lower'] = str.lower
        self.env.filters['length'] = len
    
    def render(self, template_str: str, variables: Dict[str, Any]) -> str:
        """
        템플릿 렌더링
        
        Args:
            template_str: Jinja2 템플릿 문자열
            variables: 변수 딕셔너리
        
        Returns:
            렌더링된 문자열
        
        Raises:
            TemplateError: 렌더링 실패
        """
        try:
            template = self.env.from_string(template_str)
            result = template.render(variables)
            return result
        
        except UndefinedError as e:
            raise VariableNotProvidedError(f"Missing variable: {str(e)}")
        except TemplateSyntaxError as e:
            raise RenderError(f"Template syntax error: {str(e)}")
        except Exception as e:
            raise RenderError(f"Rendering failed: {str(e)}")
    
    def validate_template(self, template_str: str) -> bool:
        """
        템플릿 구문 검증 (컴파일 가능성)
        
        Args:
            template_str: 검증할 템플릿
        
        Returns:
            True if valid, raises exception if invalid
        """
        try:
            self.env.from_string(template_str)
            return True
        except TemplateSyntaxError as e:
            raise RenderError(f"Invalid template: {str(e)}")
    
    def extract_variables_from_template(self, template_str: str) -> List[str]:
        """
        템플릿에서 변수 이름 추출
        
        Args:
            template_str: 템플릿 문자열
        
        Returns:
            변수 이름 리스트 (중복 제거, 정렬)
        """
        # Jinja2 {{ var }}, {{ var.attr }}, {{ var[0] }} 등
        pattern = r'\{\{\s*(\w+)(?:[.\[]|[\s}])'
        matches = re.findall(pattern, template_str)
        return sorted(list(set(matches)))
```

### 4-2. PromptRegistry 싱글톤

**파일:** `/backend/app/services/prompt/registry.py`

**작업 항목:**

```python
from typing import Optional, List, Dict, Any
from datetime import datetime, timedelta
from threading import Lock
from sqlalchemy.orm import Session
import logging

from backend.app.repositories.prompt_repository import PromptRepository
from backend.app.services.prompt.template_engine import JinjaTemplateEngine

logger = logging.getLogger(__name__)

class PromptNotFoundError(Exception):
    """프롬프트를 찾을 수 없음"""
    pass

class VersionNotFoundError(Exception):
    """버전을 찾을 수 없음"""
    pass

class PromptRegistry:
    """프롬프트 런타임 로더 (Singleton)"""
    
    _instance = None
    _lock = Lock()
    _cache = {}  # {prompt_name: (template, cached_at)}
    _cache_ttl = timedelta(hours=1)
    _template_engine = JinjaTemplateEngine()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
    
    def load_prompt(self, db_session: Session, name: str) -> str:
        """
        활성 버전의 프롬프트 템플릿 로드
        
        캐시 사용 (TTL: 1시간)
        
        Args:
            db_session: SQLAlchemy 세션
            name: 프롬프트 이름
        
        Returns:
            프롬프트 템플릿 문자열
        
        Raises:
            PromptNotFoundError: 프롬프트 미존재
        """
        # 캐시 확인
        if name in self._cache:
            template, cached_at = self._cache[name]
            if datetime.utcnow() - cached_at < self._cache_ttl:
                logger.info(f"Prompt '{name}' loaded from cache")
                return template
        
        # DB에서 로드
        repo = PromptRepository(db_session)
        prompt = repo.get_by_name(name)
        
        if not prompt:
            raise PromptNotFoundError(f"Prompt not found: {name}")
        
        active_version = repo.get_active_version(str(prompt.id))
        if not active_version:
            raise PromptNotFoundError(f"No active version for prompt: {name}")
        
        template = active_version.template
        
        # 캐시에 저장
        self._cache[name] = (template, datetime.utcnow())
        
        # 사용 횟수 증가
        repo.increment_usage_count(str(active_version.id))
        
        logger.info(f"Prompt '{name}' loaded from database (v{active_version.version})")
        return template
    
    def render(self, db_session: Session, name: str, **variables) -> str:
        """
        프롬프트 템플릿을 로드하고 변수를 치환하여 렌더링
        
        Args:
            db_session: SQLAlchemy 세션
            name: 프롬프트 이름
            **variables: 템플릿 변수 (예: query="...", context=["..."])
        
        Returns:
            렌더링된 프롬프트 문자열
        
        Raises:
            PromptNotFoundError: 프롬프트 미존재
            VariableNotProvidedError: 필수 변수 누락
            RenderError: 렌더링 실패
        """
        try:
            template = self.load_prompt(db_session, name)
            result = self._template_engine.render(template, variables)
            return result
        
        except (PromptNotFoundError, Exception) as e:
            logger.error(f"Failed to render prompt '{name}': {str(e)}")
            raise
    
    def load_version(self, db_session: Session, name: str, version: int) -> str:
        """
        특정 버전의 프롬프트 템플릿 로드
        
        Args:
            db_session: SQLAlchemy 세션
            name: 프롬프트 이름
            version: 버전 번호
        
        Returns:
            프롬프트 템플릿 문자열
        
        Raises:
            PromptNotFoundError: 프롬프트 미존재
            VersionNotFoundError: 버전 미존재
        """
        repo = PromptRepository(db_session)
        prompt = repo.get_by_name(name)
        
        if not prompt:
            raise PromptNotFoundError(f"Prompt not found: {name}")
        
        prompt_version = repo.get_version(str(prompt.id), version)
        if not prompt_version:
            raise VersionNotFoundError(f"Version not found: {name} v{version}")
        
        return prompt_version.template
    
    def list_variables(self, db_session: Session, name: str) -> List[str]:
        """
        프롬프트의 변수 목록 조회
        
        Args:
            db_session: SQLAlchemy 세션
            name: 프롬프트 이름
        
        Returns:
            변수 이름 리스트
        
        Raises:
            PromptNotFoundError: 프롬프트 미존재
        """
        repo = PromptRepository(db_session)
        prompt = repo.get_by_name(name)
        
        if not prompt:
            raise PromptNotFoundError(f"Prompt not found: {name}")
        
        return prompt.variables or []
    
    def invalidate_cache(self, name: Optional[str] = None) -> None:
        """
        캐시 무효화
        
        Args:
            name: 특정 프롬프트만 무효화. None이면 전체 캐시 제거
        """
        if name:
            if name in self._cache:
                del self._cache[name]
                logger.info(f"Cache invalidated for '{name}'")
        else:
            self._cache.clear()
            logger.info("All prompt cache invalidated")
    
    def get_cache_stats(self) -> Dict[str, Any]:
        """캐시 통계 조회"""
        return {
            "cached_prompts": len(self._cache),
            "prompts": list(self._cache.keys()),
            "ttl_hours": self._cache_ttl.total_seconds() / 3600
        }
```

### 4-3. 보안 및 테스트

**파일:** `/backend/tests/unit/services/test_prompt_registry.py`

**작업 항목:**

```python
import pytest
from sqlalchemy.orm import Session
from unittest.mock import MagicMock, patch

from backend.app.services.prompt.registry import (
    PromptRegistry, PromptNotFoundError, VersionNotFoundError
)
from backend.app.services.prompt.template_engine import (
    JinjaTemplateEngine, VariableNotProvidedError, RenderError
)

class TestJinjaTemplateEngine:
    """Jinja2 템플릿 엔진 테스트"""
    
    def test_simple_render(self):
        """단순 변수 치환"""
        engine = JinjaTemplateEngine()
        template = "Hello, {{ name }}!"
        result = engine.render(template, {"name": "World"})
        assert result == "Hello, World!"
    
    def test_missing_variable(self):
        """필수 변수 누락"""
        engine = JinjaTemplateEngine(strict_undefined=True)
        template = "Hello, {{ name }}!"
        
        with pytest.raises(VariableNotProvidedError):
            engine.render(template, {})
    
    def test_loop_render(self):
        """반복문 렌더링"""
        engine = JinjaTemplateEngine()
        template = "{% for item in items %}{{ item }},{% endfor %}"
        result = engine.render(template, {"items": ["a", "b", "c"]})
        assert result == "a,b,c,"
    
    def test_injection_prevention(self):
        """인젝션 방어: __class__ 접근 차단"""
        engine = JinjaTemplateEngine()
        # 사악한 템플릿 시도
        template = "{{ name.__class__.__bases__[0].__subclasses__() }}"
        
        # SandboxedEnvironment는 이를 차단해야 함
        with pytest.raises(Exception):  # AttributeError 또는 UndefinedError
            engine.render(template, {"name": "test"})
    
    def test_extract_variables(self):
        """변수 추출"""
        engine = JinjaTemplateEngine()
        template = "{{ original_query }} and {{ context }} and {{ original_query }}"
        
        vars = engine.extract_variables_from_template(template)
        assert set(vars) == {"original_query", "context"}
    
    def test_template_validation(self):
        """템플릿 구문 검증"""
        engine = JinjaTemplateEngine()
        
        # 유효한 템플릿
        assert engine.validate_template("{{ name }}") == True
        
        # 잘못된 템플릿
        with pytest.raises(RenderError):
            engine.validate_template("{% if x %}unclosed")

class TestPromptRegistry:
    """PromptRegistry 싱글톤 테스트"""
    
    @pytest.fixture
    def registry(self):
        """레지스트리 초기화"""
        registry = PromptRegistry()
        registry.invalidate_cache()  # 캐시 초기화
        return registry
    
    def test_singleton_instance(self, registry):
        """싱글톤 패턴 확인"""
        registry2 = PromptRegistry()
        assert registry is registry2
    
    def test_load_prompt_not_found(self, registry):
        """프롬프트 미존재"""
        mock_db = MagicMock(spec=Session)
        
        with pytest.raises(PromptNotFoundError):
            registry.load_prompt(mock_db, "nonexistent")
    
    def test_render_with_variables(self, registry):
        """변수 포함 렌더링"""
        # Mock 설정 (실제 DB 쿼리 시뮬레이션)
        mock_prompt = MagicMock()
        mock_prompt.id = "uuid-123"
        mock_prompt.name = "test_prompt"
        mock_prompt.variables = ["query", "context"]
        
        mock_version = MagicMock()
        mock_version.id = "version-uuid"
        mock_version.template = "Query: {{ query }}, Context: {{ context }}"
        mock_version.version = 1
        
        mock_repo = MagicMock()
        mock_repo.get_by_name.return_value = mock_prompt
        mock_repo.get_active_version.return_value = mock_version
        
        # registry.render 호출 시 mock repo 사용
        with patch('backend.app.services.prompt.registry.PromptRepository', return_value=mock_repo):
            result = registry.render(
                MagicMock(),
                "test_prompt",
                query="search term",
                context="previous messages"
            )
            assert "search term" in result
            assert "previous messages" in result
    
    def test_cache_invalidation(self, registry):
        """캐시 무효화"""
        registry._cache["test_prompt"] = ("template", MagicMock())
        assert "test_prompt" in registry._cache
        
        registry.invalidate_cache("test_prompt")
        assert "test_prompt" not in registry._cache
        
        # 전체 캐시 무효화
        registry._cache["other"] = ("template", MagicMock())
        registry.invalidate_cache()
        assert len(registry._cache) == 0
```

## 5. 산출물

1. **JinjaTemplateEngine** (`/backend/app/services/prompt/template_engine.py`)
   - SandboxedEnvironment 기반 렌더링
   - 변수 추출 및 유효성 검사
   - 보안 필터/함수 설정

2. **PromptRegistry 싱글톤** (`/backend/app/services/prompt/registry.py`)
   - 메모리 캐싱 (TTL 1시간)
   - load_prompt(), render(), load_version(), list_variables()
   - invalidate_cache(), get_cache_stats()

3. **오류 클래스**
   - PromptNotFoundError, VersionNotFoundError
   - VariableNotProvidedError, RenderError

4. **단위 테스트** (`/backend/tests/unit/services/test_prompt_registry.py`)
   - 템플릿 렌더링, 변수 치환 테스트
   - 인젝션 방어 테스트
   - 싱글톤 패턴, 캐시 검증 테스트

## 6. 완료 기준

1. JinjaTemplateEngine이 기본 변수 치환 및 루프/조건문 렌더링
2. SandboxedEnvironment가 __class__, __globals__ 등 위험한 속성 접근 차단
3. PromptRegistry가 싱글톤이고, 메모리 캐싱이 작동 (TTL 확인)
4. load_prompt()가 DB에서 활성 버전을 로드하고 캐시
5. render()가 변수를 치환하여 프롬프트 반환
6. load_version()이 특정 버전 로드
7. list_variables()가 템플릿 변수 목록 반환
8. invalidate_cache()가 캐시 무효화
9. 단위 테스트 모두 통과 (특히 인젝션 방어)
10. usage_count가 로드 시 증가

## 7. 작업 지침

### 지침 7-1. SandboxedEnvironment 설정

Jinja2의 SandboxedEnvironment는 기본적으로 위험한 속성 접근을 차단한다. 추가 커스터마이징이 필요하면:

```python
from jinja2.sandbox import SandboxedEnvironment

env = SandboxedEnvironment()
# 접근 불가: __class__, __bases__, __subclasses__ 등
```

### 지침 7-2. 캐시 TTL

메모리 캐시는 기본 1시간 TTL을 사용한다. 필요시 환경변수로 조정:

```python
_cache_ttl = timedelta(hours=int(os.getenv('PROMPT_CACHE_TTL_HOURS', '1')))
```

### 지침 7-3. 스레드 안전성

PromptRegistry는 Lock을 사용하여 스레드 안전한 싱글톤 구현. 캐시 접근도 필요시 Lock 추가:

```python
with self._lock:
    if name in self._cache:
        # 캐시 로직
```

### 지침 7-4. 오류 구분

- PromptNotFoundError: 프롬프트 DB 레코드 미존재
- VersionNotFoundError: 버전 번호 미존재
- VariableNotProvidedError: 렌더링 시 필수 변수 누락
- RenderError: 템플릿 구문 오류 또는 렌더링 실패
