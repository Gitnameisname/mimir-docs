# Task 1-1. LLMProvider 인터페이스 설계 및 LLMResponse 모델

## 1. 작업 목적

서로 다른 LLM 서비스(OpenAI, Anthropic, vLLM, Ollama 등)를 단일 인터페이스로 통합하기 위한 **추상 베이스 클래스(ABC)와 핵심 응답 모델**을 설계한다. 이는 Phase 1의 기초이며, 모든 LLM Provider 구현체가 상속받을 계약(contract)을 정의한다.

## 2. 작업 범위

### 포함 범위

1. **LLMProvider 추상 베이스 클래스**
   - 핵심 메서드: `generate()`, `stream_generate()`, `batch_generate()`, `count_tokens()`
   - 메서드 시그니처 및 반환 타입 정의
   - 각 메서드별 docstring (동작 설명, 파라미터, 반환값)
   - 기본 구현은 NotImplementedError 발생

2. **응답 데이터 모델**
   - `LLMResponse`: 단일 응답 구조
   - `LLMDelta`: 스트리밍 청크 단위 응답
   - Pydantic BaseModel 사용 (검증 포함)

3. **에러 계층 구조**
   - `LLMProviderError`: 기본 예외
   - `RateLimitError`: API 비율 제한
   - `AuthError`: 인증 실패
   - `TimeoutError`: 요청 시간 초과
   - `ContextOverflowError`: 컨텍스트 윈도우 초과

4. **설정 클래스**
   - `LLMConfig`: 공통 설정 (timeout, retry, temperature 등)
   - Provider별 필수/선택 필드 정의

### 제외 범위

- 특정 provider 구현 (Task 1-2~1-4에서)
- 실제 API 호출 로직
- 외부 라이브러리 초기화 (openai, anthropic 등)

## 3. 선행 조건

- Phase 0 완료 (Python 환경, FastAPI 기본 구조)
- `/backend/app/` 디렉토리 접근 가능
- Pydantic v2 설치 확인
- Python 3.10+

## 4. 주요 작업 항목

### 4-1. LLMProvider 추상 베이스 클래스 설계

**파일:** `/backend/app/services/llm/base.py`

**작업 항목:**
```python
from abc import ABC, abstractmethod
from typing import AsyncIterator, List, Optional, Dict, Any
from dataclasses import dataclass
from datetime import datetime
import time

@dataclass
class LLMResponse:
    """LLM 응답 데이터 모델"""
    content: str                    # 생성된 텍스트
    tokens_used: int               # 총 토큰 사용량
    cost: float                    # 추정 비용 (외부 API용, 폐쇄망은 0)
    latency_ms: float              # 응답 시간 (밀리초)
    model: str                     # 사용한 모델명
    provider: str                  # 제공자 (openai, anthropic 등)
    finish_reason: Optional[str] = None  # 종료 이유 (stop, max_tokens 등)
    raw_response: Optional[Dict[str, Any]] = None  # 원본 API 응답

@dataclass
class LLMDelta:
    """스트리밍 응답 청크"""
    partial_content: str           # 부분 텍스트
    finish_reason: Optional[str] = None
    is_final: bool = False         # 마지막 청크 여부

class LLMProviderError(Exception):
    """LLM Provider 기본 예외"""
    pass

class RateLimitError(LLMProviderError):
    """API 비율 제한 초과"""
    pass

class AuthError(LLMProviderError):
    """인증 실패 (API 키 없음, 만료됨 등)"""
    pass

class TimeoutError(LLMProviderError):
    """요청 시간 초과"""
    pass

class ContextOverflowError(LLMProviderError):
    """컨텍스트 윈도우 초과"""
    pass

class LLMProvider(ABC):
    """LLM 서비스 추상 인터페이스"""
    
    def __init__(self, config: Optional[Dict[str, Any]] = None):
        """
        Provider 초기화
        
        Args:
            config: 설정 딕셔너리 (timeout, retry_count 등)
        """
        self.config = config or {}
        self.model_name: Optional[str] = None
        self.provider_name: str = "unknown"
    
    @abstractmethod
    async def generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> LLMResponse:
        """
        동기 텍스트 생성
        
        Args:
            prompt: 입력 프롬프트
            temperature: 생성 다양성 (0~2)
            max_tokens: 최대 생성 토큰 수
            **kwargs: Provider별 추가 파라미터
        
        Returns:
            LLMResponse 객체
        
        Raises:
            RateLimitError: API 비율 제한
            AuthError: 인증 실패
            TimeoutError: 시간 초과
            ContextOverflowError: 컨텍스트 초과
        """
        pass
    
    @abstractmethod
    async def stream_generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> AsyncIterator[LLMDelta]:
        """
        스트리밍 텍스트 생성
        
        Args:
            prompt: 입력 프롬프트
            temperature: 생성 다양성
            max_tokens: 최대 생성 토큰 수
            **kwargs: Provider별 추가 파라미터
        
        Yields:
            LLMDelta 청크들
        
        Raises:
            RateLimitError, AuthError, TimeoutError, ContextOverflowError
        """
        pass
    
    @abstractmethod
    async def batch_generate(
        self,
        prompts: List[str],
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> List[LLMResponse]:
        """
        배치 텍스트 생성 (여러 프롬프트 동시 처리)
        
        Args:
            prompts: 프롬프트 리스트
            temperature: 생성 다양성
            max_tokens: 최대 생성 토큰 수
            **kwargs: Provider별 추가 파라미터
        
        Returns:
            LLMResponse 리스트 (순서 보장)
        """
        pass
    
    @abstractmethod
    async def count_tokens(self, text: str) -> int:
        """
        텍스트의 토큰 개수 계산
        
        Args:
            text: 입력 텍스트
        
        Returns:
            토큰 개수
        """
        pass
    
    async def get_model_info(self) -> Dict[str, Any]:
        """
        모델 정보 조회 (선택)
        
        Returns:
            {'model': str, 'max_tokens': int, 'context_window': int, ...}
        """
        return {
            'model': self.model_name or 'unknown',
            'provider': self.provider_name
        }
```

### 4-2. Pydantic 스키마 정의

**파일:** `/backend/app/schemas/llm.py`

**작업 항목:**
```python
from pydantic import BaseModel, Field
from typing import Optional, Dict, Any, List
from datetime import datetime

class LLMResponseSchema(BaseModel):
    """API 응답 스키마"""
    content: str
    tokens_used: int = Field(..., ge=0)
    cost: float = Field(..., ge=0)
    latency_ms: float = Field(..., ge=0)
    model: str
    provider: str
    finish_reason: Optional[str] = None
    
    class Config:
        json_schema_extra = {
            "example": {
                "content": "안녕하세요",
                "tokens_used": 45,
                "cost": 0.001,
                "latency_ms": 523.4,
                "model": "gpt-4o",
                "provider": "openai",
                "finish_reason": "stop"
            }
        }

class LLMDeltaSchema(BaseModel):
    """스트리밍 청크 스키마"""
    partial_content: str
    finish_reason: Optional[str] = None
    is_final: bool = False

class GenerateRequestSchema(BaseModel):
    """생성 요청 스키마"""
    prompt: str = Field(..., min_length=1)
    temperature: float = Field(0.7, ge=0, le=2)
    max_tokens: Optional[int] = Field(None, ge=1, le=4096)
    
    class Config:
        json_schema_extra = {
            "example": {
                "prompt": "다음 문장을 영어로 번역하세요: 안녕하세요",
                "temperature": 0.7,
                "max_tokens": 100
            }
        }
```

### 4-3. 설정 및 유틸리티 클래스

**파일:** `/backend/app/services/llm/config.py`

**작업 항목:**
```python
from dataclasses import dataclass, field
from typing import Optional, Dict, Any

@dataclass
class LLMConfig:
    """LLM 공통 설정"""
    timeout: int = 30                    # API 타임아웃 (초)
    retry_count: int = 3                 # 재시도 횟수
    retry_delay: float = 1.0             # 재시도 간격 (초)
    max_tokens_default: int = 1000       # 기본 최대 토큰
    temperature_default: float = 0.7     # 기본 temperature
    enable_streaming: bool = True        # 스트리밍 지원 여부
    cost_tracking: bool = True           # 비용 추적 여부
    extra_params: Dict[str, Any] = field(default_factory=dict)

class LLMProviderRegistry:
    """Provider 등록 및 조회"""
    _providers: Dict[str, type] = {}
    
    @classmethod
    def register(cls, name: str, provider_class: type) -> None:
        """Provider 등록"""
        cls._providers[name] = provider_class
    
    @classmethod
    def get(cls, name: str) -> Optional[type]:
        """Provider 클래스 조회"""
        return cls._providers.get(name)
    
    @classmethod
    def list_providers(cls) -> List[str]:
        """등록된 모든 provider 목록"""
        return list(cls._providers.keys())
```

### 4-4. 단위 테스트 (기본 구조)

**파일:** `/backend/tests/unit/services/test_llm_base.py`

**작업 항목:**
```python
import pytest
from backend.app.services.llm.base import (
    LLMProvider, LLMResponse, LLMDelta,
    RateLimitError, AuthError, TimeoutError, ContextOverflowError
)

class MockLLMProvider(LLMProvider):
    """테스트용 Mock Provider"""
    async def generate(self, prompt: str, **kwargs) -> LLMResponse:
        return LLMResponse(
            content="Mock response",
            tokens_used=10,
            cost=0.0,
            latency_ms=100.0,
            model="mock",
            provider="mock"
        )
    
    async def stream_generate(self, prompt: str, **kwargs):
        yield LLMDelta(partial_content="Mock ", is_final=False)
        yield LLMDelta(partial_content="streaming", is_final=True)
    
    async def batch_generate(self, prompts: list, **kwargs):
        return [await self.generate(p) for p in prompts]
    
    async def count_tokens(self, text: str) -> int:
        return len(text.split())

@pytest.mark.asyncio
async def test_llm_response_creation():
    """LLMResponse 객체 생성 테스트"""
    response = LLMResponse(
        content="Test",
        tokens_used=5,
        cost=0.001,
        latency_ms=50.0,
        model="test-model",
        provider="test"
    )
    assert response.content == "Test"
    assert response.tokens_used == 5

@pytest.mark.asyncio
async def test_mock_provider_generate():
    """Mock Provider의 generate 메서드 테스트"""
    provider = MockLLMProvider()
    response = await provider.generate("Test prompt")
    assert response.content == "Mock response"
    assert response.provider == "mock"

@pytest.mark.asyncio
async def test_mock_provider_streaming():
    """Mock Provider의 stream_generate 테스트"""
    provider = MockLLMProvider()
    chunks = []
    async for delta in provider.stream_generate("Test"):
        chunks.append(delta.partial_content)
    assert "".join(chunks) == "Mock streaming"
```

## 5. 산출물

1. **LLMProvider 추상 베이스 클래스** (`/backend/app/services/llm/base.py`)
   - ABC, abstractmethod 정의
   - 4개 핵심 메서드 (generate, stream_generate, batch_generate, count_tokens)
   - 5개 에러 클래스 (LLMProviderError, RateLimitError, AuthError, TimeoutError, ContextOverflowError)
   - LLMResponse, LLMDelta 데이터 클래스

2. **Pydantic 스키마** (`/backend/app/schemas/llm.py`)
   - LLMResponseSchema, LLMDeltaSchema
   - GenerateRequestSchema (API 요청 검증용)

3. **설정 및 레지스트리** (`/backend/app/services/llm/config.py`)
   - LLMConfig 데이터 클래스
   - LLMProviderRegistry (런타임 provider 관리)

4. **단위 테스트** (`/backend/tests/unit/services/test_llm_base.py`)
   - Mock Provider 구현
   - 기본 동작 테스트 (response 생성, 스트리밍, 배치)

## 6. 완료 기준

1. `LLMProvider` ABC 정의되고, 모든 abstractmethod가 명시됨
2. `LLMResponse`, `LLMDelta` 데이터 클래스 정상 작동
3. 5개 에러 클래스가 올바른 상속 관계 형성
4. Pydantic 스키마 검증 작동 (유효한 입력, 잘못된 입력 구분)
5. Mock Provider가 모든 추상 메서드 구현
6. 단위 테스트 모두 통과 (`pytest -v`)
7. 타입 힌트 정확 (mypy 검사 통과)
8. docstring이 모든 메서드에 포함됨

## 7. 작업 지침

### 지침 7-1. 메서드 시그니처 확장성

향후 Task 1-2~1-4에서 provider별 추가 파라미터가 필요할 수 있으므로, 모든 메서드에 `**kwargs`를 포함한다:

```python
async def generate(self, prompt: str, temperature=0.7, max_tokens=None, **kwargs):
    # provider별 추가 파라미터 (system_prompt, top_p 등)는 kwargs로 처리
    pass
```

### 지침 7-2. 에러 처리 계층

에러 클래스는 다음과 같이 catch될 수 있어야 한다:

```python
try:
    response = await provider.generate(prompt)
except RateLimitError:
    # 비율 제한 처리 (재시도 등)
except AuthError:
    # 인증 실패 처리 (환경변수 확인)
except LLMProviderError:
    # 기타 LLM 관련 오류
```

### 지침 7-3. 응답 검증

Pydantic 스키마를 사용하여 응답 신뢰성을 높인다. Provider 구현 시 raw_response는 디버깅용으로 선택적 저장한다.

### 지침 7-4. 코드 스타일

- 모든 async 메서드는 `async def` 형식
- 타입 힌트 필수 (파라미터, 반환값)
- docstring은 Google style 또는 NumPy style 통일
- import 정렬: stdlib → third-party → local (isort 활용)

### 지침 7-5. 문서화

각 메서드의 docstring에는:
- 한 줄 요약
- Args: 파라미터 설명
- Returns: 반환값 설명
- Raises: 발생 가능한 예외

이를 Sphinx로 자동 생성 가능하게 작성한다.
