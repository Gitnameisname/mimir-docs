# Task 1-2. OpenAI Provider 구현

## 1. 작업 목적

Task 1-1에서 정의한 `LLMProvider` 추상 인터페이스를 상속받아, **OpenAI API와의 통신을 담당하는 concrete provider**를 구현한다. gpt-4o, gpt-4-turbo, gpt-3.5-turbo 모델을 지원하며, 스트리밍, 토큰 계산, 비용 추적 기능을 포함한다.

## 2. 작업 범위

### 포함 범위

1. **OpenAILLMProvider 클래스**
   - `generate()`: 동기 텍스트 생성 (Chat Completions API)
   - `stream_generate()`: 스트리밍 응답 (SSE)
   - `batch_generate()`: 배치 처리 (순차 또는 병렬)
   - `count_tokens()`: tiktoken 라이브러리 사용

2. **환경 변수 기반 설정**
   - `OPENAI_API_KEY`: API 키 (필수)
   - `OPENAI_MODEL`: 모델명 (기본값: gpt-4o)
   - `OPENAI_BASE_URL`: 커스텀 베이스 URL (선택, 기본: https://api.openai.com/v1)

3. **에러 처리**
   - RateLimitError (429)
   - AuthError (401, 403)
   - TimeoutError (timeout)
   - ContextOverflowError (context_length_exceeded)
   - 기타 오류 (LLMProviderError로 변환)

4. **비용 추적**
   - 토큰 사용량을 API 응답의 usage 필드에서 추출
   - 모델별 가격표 (gpt-4o, gpt-4-turbo, gpt-3.5-turbo)
   - input_tokens * input_price + output_tokens * output_price

5. **단위 테스트**
   - Mock OpenAI API 응답 (httpx 또는 responses 라이브러리)
   - 정상 응답, 스트리밍, 에러 케이스 검증

### 제외 범위

- Vision API, Function Calling (Phase 4+ 고려사항)
- Fine-tuning 관련 기능
- 비동기 배치 처리 (동기 순회로 충분)

## 3. 선행 조건

- Task 1-1 완료 (`LLMProvider` ABC 정의)
- OpenAI Python SDK 설치 (`pip install openai`)
- tiktoken 설치 (`pip install tiktoken`)
- OPENAI_API_KEY 환경변수 설정 (테스트 시)

## 4. 주요 작업 항목

### 4-1. OpenAILLMProvider 클래스 구현

**파일:** `/backend/app/services/llm/openai_provider.py`

**작업 항목:**
```python
import os
import asyncio
import time
from typing import AsyncIterator, List, Optional, Dict, Any
from openai import AsyncOpenAI, RateLimitError as OpenAIRateLimitError
from openai import APIError, APIConnectionError, APITimeoutError
import tiktoken

from backend.app.services.llm.base import (
    LLMProvider, LLMResponse, LLMDelta,
    RateLimitError, AuthError, TimeoutError, ContextOverflowError, LLMProviderError
)

# 모델별 가격표 (입력 / 출력, 1M 토큰 기준)
MODEL_PRICING = {
    "gpt-4o": {"input": 0.005, "output": 0.015},
    "gpt-4-turbo": {"input": 0.01, "output": 0.03},
    "gpt-3.5-turbo": {"input": 0.0005, "output": 0.0015},
}

class OpenAILLMProvider(LLMProvider):
    """OpenAI API Provider"""
    
    def __init__(self):
        super().__init__()
        self.provider_name = "openai"
        
        # 환경변수에서 설정 로드
        api_key = os.getenv("OPENAI_API_KEY")
        if not api_key:
            raise AuthError("OPENAI_API_KEY environment variable not set")
        
        self.model_name = os.getenv("OPENAI_MODEL", "gpt-4o")
        base_url = os.getenv("OPENAI_BASE_URL", "https://api.openai.com/v1")
        
        self.client = AsyncOpenAI(api_key=api_key, base_url=base_url)
        
        # tiktoken 인코더 로드 (모델별)
        try:
            self.tokenizer = tiktoken.encoding_for_model(self.model_name)
        except KeyError:
            # 지원하지 않는 모델인 경우 cl100k_base 사용
            self.tokenizer = tiktoken.get_encoding("cl100k_base")
    
    async def generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> LLMResponse:
        """
        OpenAI Chat Completions API를 사용한 동기 생성
        
        Args:
            prompt: 입력 프롬프트
            temperature: 생성 다양성
            max_tokens: 최대 생성 토큰
            **kwargs: system_prompt, top_p 등 추가 파라미터
        
        Returns:
            LLMResponse 객체
        
        Raises:
            RateLimitError, AuthError, TimeoutError, ContextOverflowError
        """
        start_time = time.time()
        
        # 메시지 구성
        messages = []
        if "system_prompt" in kwargs:
            messages.append({"role": "system", "content": kwargs["system_prompt"]})
        messages.append({"role": "user", "content": prompt})
        
        try:
            response = await self.client.chat.completions.create(
                model=self.model_name,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens or 1000,
                timeout=self.config.get("timeout", 30)
            )
            
            latency_ms = (time.time() - start_time) * 1000
            content = response.choices[0].message.content
            
            # 토큰 사용량 및 비용 계산
            input_tokens = response.usage.prompt_tokens
            output_tokens = response.usage.completion_tokens
            
            pricing = MODEL_PRICING.get(self.model_name, {"input": 0, "output": 0})
            cost = (
                (input_tokens * pricing["input"] / 1_000_000) +
                (output_tokens * pricing["output"] / 1_000_000)
            )
            
            return LLMResponse(
                content=content,
                tokens_used=input_tokens + output_tokens,
                cost=cost,
                latency_ms=latency_ms,
                model=self.model_name,
                provider="openai",
                finish_reason=response.choices[0].finish_reason,
                raw_response=response.model_dump() if hasattr(response, 'model_dump') else None
            )
        
        except OpenAIRateLimitError as e:
            raise RateLimitError(f"OpenAI rate limit exceeded: {str(e)}")
        except APITimeoutError as e:
            raise TimeoutError(f"OpenAI request timeout: {str(e)}")
        except APIError as e:
            if "context_length_exceeded" in str(e):
                raise ContextOverflowError(f"OpenAI context overflow: {str(e)}")
            elif e.status_code in (401, 403):
                raise AuthError(f"OpenAI authentication failed: {str(e)}")
            else:
                raise LLMProviderError(f"OpenAI API error: {str(e)}")
        except APIConnectionError as e:
            raise LLMProviderError(f"OpenAI connection error: {str(e)}")
    
    async def stream_generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> AsyncIterator[LLMDelta]:
        """
        OpenAI 스트리밍 응답
        
        Args:
            prompt: 입력 프롬프트
            temperature: 생성 다양성
            max_tokens: 최대 생성 토큰
            **kwargs: 추가 파라미터
        
        Yields:
            LLMDelta 청크들
        """
        messages = []
        if "system_prompt" in kwargs:
            messages.append({"role": "system", "content": kwargs["system_prompt"]})
        messages.append({"role": "user", "content": prompt})
        
        try:
            async with await self.client.chat.completions.create(
                model=self.model_name,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens or 1000,
                stream=True
            ) as stream:
                async for chunk in stream:
                    if chunk.choices[0].delta.content:
                        yield LLMDelta(
                            partial_content=chunk.choices[0].delta.content,
                            finish_reason=chunk.choices[0].finish_reason,
                            is_final=False
                        )
                    if chunk.choices[0].finish_reason:
                        yield LLMDelta(
                            partial_content="",
                            finish_reason=chunk.choices[0].finish_reason,
                            is_final=True
                        )
        except OpenAIRateLimitError as e:
            raise RateLimitError(str(e))
        except APITimeoutError as e:
            raise TimeoutError(str(e))
        except APIError as e:
            if e.status_code in (401, 403):
                raise AuthError(str(e))
            raise LLMProviderError(str(e))
    
    async def batch_generate(
        self,
        prompts: List[str],
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> List[LLMResponse]:
        """
        배치 생성 (순차 처리)
        
        Args:
            prompts: 프롬프트 리스트
            temperature: 생성 다양성
            max_tokens: 최대 생성 토큰
            **kwargs: 추가 파라미터
        
        Returns:
            LLMResponse 리스트
        """
        responses = []
        for prompt in prompts:
            response = await self.generate(prompt, temperature, max_tokens, **kwargs)
            responses.append(response)
        return responses
    
    async def count_tokens(self, text: str) -> int:
        """
        tiktoken을 사용한 토큰 계산
        
        Args:
            text: 입력 텍스트
        
        Returns:
            토큰 개수
        """
        try:
            tokens = self.tokenizer.encode(text)
            return len(tokens)
        except Exception as e:
            # 폴백: 단순 추정 (대략 1 단어 = 1.3 토큰)
            return int(len(text.split()) * 1.3)
```

### 4-2. 환경 변수 검증

**파일:** `/backend/app/services/llm/openai_provider.py` (계속)

**추가 메서드:**
```python
@classmethod
def validate_environment(cls) -> Dict[str, bool]:
    """
    필수 환경변수 검증
    
    Returns:
        {'OPENAI_API_KEY': True/False, 'OPENAI_MODEL': True/False}
    """
    return {
        'OPENAI_API_KEY': bool(os.getenv('OPENAI_API_KEY')),
        'OPENAI_MODEL': bool(os.getenv('OPENAI_MODEL')),
    }

@classmethod
def is_available(cls) -> bool:
    """Provider 사용 가능 여부"""
    return bool(os.getenv('OPENAI_API_KEY'))
```

### 4-3. 단위 테스트

**파일:** `/backend/tests/unit/services/test_openai_provider.py`

**작업 항목:**
```python
import pytest
import os
from unittest.mock import AsyncMock, patch, MagicMock
from backend.app.services.llm.openai_provider import (
    OpenAILLMProvider, LLMResponse, RateLimitError, AuthError, TimeoutError
)

@pytest.fixture
def mock_env_vars(monkeypatch):
    """OpenAI 환경변수 설정"""
    monkeypatch.setenv('OPENAI_API_KEY', 'sk-test-key-12345')
    monkeypatch.setenv('OPENAI_MODEL', 'gpt-4o')

@pytest.fixture
async def provider(mock_env_vars):
    """Provider 인스턴스"""
    # 실제 API 호출을 피하기 위해 mock 필요
    with patch('backend.app.services.llm.openai_provider.AsyncOpenAI'):
        return OpenAILLMProvider()

@pytest.mark.asyncio
async def test_generate_success(provider):
    """정상 생성 응답 테스트"""
    mock_response = MagicMock()
    mock_response.choices[0].message.content = "Test response"
    mock_response.choices[0].finish_reason = "stop"
    mock_response.usage.prompt_tokens = 10
    mock_response.usage.completion_tokens = 20
    
    provider.client.chat.completions.create = AsyncMock(return_value=mock_response)
    
    response = await provider.generate("Test prompt")
    
    assert response.content == "Test response"
    assert response.tokens_used == 30
    assert response.provider == "openai"
    assert response.cost > 0

@pytest.mark.asyncio
async def test_count_tokens(provider):
    """토큰 계산 테스트"""
    text = "Hello, world! This is a test."
    tokens = await provider.count_tokens(text)
    
    assert isinstance(tokens, int)
    assert tokens > 0

@pytest.mark.asyncio
async def test_rate_limit_error(provider):
    """RateLimitError 처리"""
    from openai import RateLimitError as OpenAIRateLimitError
    provider.client.chat.completions.create = AsyncMock(
        side_effect=OpenAIRateLimitError("429 Too Many Requests", None, None)
    )
    
    with pytest.raises(RateLimitError):
        await provider.generate("Test prompt")

@pytest.mark.asyncio
async def test_auth_error(provider):
    """AuthError 처리"""
    from openai import AuthenticationError
    provider.client.chat.completions.create = AsyncMock(
        side_effect=AuthenticationError("401 Unauthorized", None, None)
    )
    
    with pytest.raises(AuthError):
        await provider.generate("Test prompt")

@pytest.mark.asyncio
async def test_batch_generate(provider):
    """배치 생성 테스트"""
    mock_response = MagicMock()
    mock_response.choices[0].message.content = "Batch response"
    mock_response.choices[0].finish_reason = "stop"
    mock_response.usage.prompt_tokens = 5
    mock_response.usage.completion_tokens = 10
    
    provider.client.chat.completions.create = AsyncMock(return_value=mock_response)
    
    prompts = ["Prompt 1", "Prompt 2", "Prompt 3"]
    responses = await provider.batch_generate(prompts)
    
    assert len(responses) == 3
    assert all(isinstance(r, LLMResponse) for r in responses)

@pytest.mark.asyncio
async def test_validate_environment(monkeypatch):
    """환경변수 검증 테스트"""
    monkeypatch.delenv('OPENAI_API_KEY', raising=False)
    
    validation = OpenAILLMProvider.validate_environment()
    assert validation['OPENAI_API_KEY'] is False

@pytest.mark.asyncio
async def test_is_available(monkeypatch):
    """Provider 가용성 테스트"""
    monkeypatch.setenv('OPENAI_API_KEY', 'sk-test')
    assert OpenAILLMProvider.is_available() is True
    
    monkeypatch.delenv('OPENAI_API_KEY', raising=False)
    assert OpenAILLMProvider.is_available() is False
```

## 5. 산출물

1. **OpenAILLMProvider 클래스** (`/backend/app/services/llm/openai_provider.py`)
   - 모든 추상 메서드 구현
   - 환경변수 기반 설정
   - 에러 처리 및 비용 추적

2. **단위 테스트** (`/backend/tests/unit/services/test_openai_provider.py`)
   - 정상 응답, 스트리밍, 배치, 에러 케이스
   - Mock API 활용

3. **통합 테스트 (선택)** (`.env.example` 파일 예시 포함)
   ```
   OPENAI_API_KEY=sk-...
   OPENAI_MODEL=gpt-4o
   OPENAI_BASE_URL=https://api.openai.com/v1
   ```

## 6. 완료 기준

1. OpenAILLMProvider가 LLMProvider를 올바르게 상속
2. generate() 메서드가 실제 API 호출 시뮬레이션 (mock 포함)
3. stream_generate()가 청크 단위로 응답
4. batch_generate()가 여러 프롬프트 순차 처리
5. count_tokens()이 정확한 토큰 개수 반환
6. 5개 에러 타입 모두 올바르게 처리됨
7. 비용 계산이 정확 (input/output 토큰별)
8. 단위 테스트 모두 통과 (`pytest -v`)
9. 환경변수 검증 함수 구현
10. 코드 스타일 통일 (isort, black 적용)

## 7. 작업 지침

### 지침 7-1. 가격표 관리

모델별 가격은 다음 위치에서 업데이트한다:
- `/backend/app/services/llm/openai_provider.py` (MODEL_PRICING)
- OpenAI 공식 문서: https://openai.com/pricing/

실제 운영 환경에서는 DB에서 관리 고려.

### 지침 7-2. 스트리밍 응답 처리

OpenAI의 스트리밍은 Server-Sent Events(SSE) 형식이다. 클라이언트는 다음과 같이 소비한다:

```python
async for delta in provider.stream_generate(prompt):
    if delta.is_final:
        print(f"Finished: {delta.finish_reason}")
    else:
        print(delta.partial_content, end='')
```

### 지침 7-3. 타임아웃 및 재시도

기본 타임아웃은 30초이다. 필요시 config에서 조정:

```python
provider.config['timeout'] = 60
```

재시도 로직은 Task 1-5 Factory에서 구현.

### 지침 7-4. API 키 보안

- `.env` 파일에는 절대 커밋하지 않음
- GitHub Actions 등에서는 Secrets 사용
- 로그에 API 키 절대 출력 금지

### 지침 7-5. 모델 교체 용이성

`OPENAI_MODEL` 환경변수로 모델 변경 가능하게 설계:

```bash
export OPENAI_MODEL=gpt-3.5-turbo
# 코드 변경 없이 모델 변경됨
```

다른 모델 추가 시 MODEL_PRICING에만 추가하면 됨.
