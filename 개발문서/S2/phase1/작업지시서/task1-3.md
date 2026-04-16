# Task 1-3. Anthropic Provider 구현

## 1. 작업 목적

Task 1-1에서 정의한 `LLMProvider` 추상 인터페이스를 상속받아, **Anthropic API(Claude 모델)와의 통신을 담당하는 provider**를 구현한다. claude-3-opus, claude-3-sonnet, claude-3-haiku 모델을 지원하며, 스트리밍, 토큰 계산, 비용 추적 기능을 포함한다.

## 2. 작업 범위

### 포함 범위

1. **AnthropicLLMProvider 클래스**
   - `generate()`: 동기 텍스트 생성 (Messages API)
   - `stream_generate()`: 스트리밍 응답 (EventStream)
   - `batch_generate()`: 배치 처리
   - `count_tokens()`: Anthropic API의 count_tokens 엔드포인트 사용

2. **환경 변수 기반 설정**
   - `ANTHROPIC_API_KEY`: API 키 (필수)
   - `ANTHROPIC_MODEL`: 모델명 (기본값: claude-3-sonnet)
   - `ANTHROPIC_BASE_URL`: 커스텀 베이스 URL (선택)

3. **에러 처리**
   - RateLimitError (rate_limit_error)
   - AuthError (invalid_api_key)
   - TimeoutError (request timeout)
   - ContextOverflowError (max_tokens_to_sample, prompt too long)
   - 기타 오류 (LLMProviderError로 변환)

4. **비용 추적**
   - API 응답의 usage 필드에서 input_tokens, output_tokens 추출
   - 모델별 가격표 (claude-3-opus, claude-3-sonnet, claude-3-haiku)
   - cost = input_tokens * input_price + output_tokens * output_price

5. **단위 테스트**
   - Mock Anthropic API 응답
   - 정상 응답, 스트리밍, 에러 케이스 검증

### 제외 범위

- Vision 기능 (Task 4+ 고려사항)
- 도구 사용(Tool Use) 기능
- 프롬프트 캐싱 (선택적)

## 3. 선행 조건

- Task 1-1 완료 (`LLMProvider` ABC 정의)
- Anthropic Python SDK 설치 (`pip install anthropic`)
- ANTHROPIC_API_KEY 환경변수 설정 (테스트 시)

## 4. 주요 작업 항목

### 4-1. AnthropicLLMProvider 클래스 구현

**파일:** `/backend/app/services/llm/anthropic_provider.py`

**작업 항목:**
```python
import os
import time
from typing import AsyncIterator, List, Optional, Dict, Any
from anthropic import AsyncAnthropic, APIError, APIConnectionError, APITimeoutError
from anthropic import RateLimitError as AnthropicRateLimitError
from anthropic import AuthenticationError

from backend.app.services.llm.base import (
    LLMProvider, LLMResponse, LLMDelta,
    RateLimitError, AuthError, TimeoutError, ContextOverflowError, LLMProviderError
)

# 모델별 가격표 (입력 / 출력, 1M 토큰 기준)
MODEL_PRICING = {
    "claude-3-opus": {"input": 0.015, "output": 0.075},
    "claude-3-sonnet": {"input": 0.003, "output": 0.015},
    "claude-3-haiku": {"input": 0.00025, "output": 0.00125},
}

# 모델별 컨텍스트 윈도우
MODEL_CONTEXT_WINDOWS = {
    "claude-3-opus": 200000,
    "claude-3-sonnet": 200000,
    "claude-3-haiku": 200000,
}

class AnthropicLLMProvider(LLMProvider):
    """Anthropic (Claude) API Provider"""
    
    def __init__(self):
        super().__init__()
        self.provider_name = "anthropic"
        
        # 환경변수에서 설정 로드
        api_key = os.getenv("ANTHROPIC_API_KEY")
        if not api_key:
            raise AuthError("ANTHROPIC_API_KEY environment variable not set")
        
        self.model_name = os.getenv("ANTHROPIC_MODEL", "claude-3-sonnet-20240229")
        base_url = os.getenv("ANTHROPIC_BASE_URL")
        
        self.client = AsyncAnthropic(api_key=api_key, base_url=base_url)
    
    async def generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> LLMResponse:
        """
        Anthropic Messages API를 사용한 동기 생성
        
        Args:
            prompt: 입력 프롬프트
            temperature: 생성 다양성 (0~1)
            max_tokens: 최대 생성 토큰
            **kwargs: system 프롬프트 등 추가 파라미터
        
        Returns:
            LLMResponse 객체
        
        Raises:
            RateLimitError, AuthError, TimeoutError, ContextOverflowError
        """
        start_time = time.time()
        
        # Anthropic API는 system과 messages를 분리
        system_prompt = kwargs.get("system_prompt", None)
        
        try:
            response = await self.client.messages.create(
                model=self.model_name,
                max_tokens=max_tokens or 1024,
                system=system_prompt,
                messages=[
                    {"role": "user", "content": prompt}
                ],
                temperature=temperature
            )
            
            latency_ms = (time.time() - start_time) * 1000
            content = response.content[0].text if response.content else ""
            
            # 토큰 사용량 및 비용 계산
            input_tokens = response.usage.input_tokens
            output_tokens = response.usage.output_tokens
            
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
                provider="anthropic",
                finish_reason=response.stop_reason,
                raw_response=response.model_dump() if hasattr(response, 'model_dump') else None
            )
        
        except AnthropicRateLimitError as e:
            raise RateLimitError(f"Anthropic rate limit exceeded: {str(e)}")
        except APITimeoutError as e:
            raise TimeoutError(f"Anthropic request timeout: {str(e)}")
        except AuthenticationError as e:
            raise AuthError(f"Anthropic authentication failed: {str(e)}")
        except APIError as e:
            error_str = str(e)
            if any(x in error_str for x in ["max_tokens", "context_length", "prompt too long"]):
                raise ContextOverflowError(f"Anthropic context overflow: {error_str}")
            else:
                raise LLMProviderError(f"Anthropic API error: {error_str}")
        except APIConnectionError as e:
            raise LLMProviderError(f"Anthropic connection error: {str(e)}")
    
    async def stream_generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> AsyncIterator[LLMDelta]:
        """
        Anthropic 스트리밍 응답 (Server-Sent Events)
        
        Args:
            prompt: 입력 프롬프트
            temperature: 생성 다양성
            max_tokens: 최대 생성 토큰
            **kwargs: 추가 파라미터
        
        Yields:
            LLMDelta 청크들
        """
        system_prompt = kwargs.get("system_prompt", None)
        
        try:
            with self.client.messages.stream(
                model=self.model_name,
                max_tokens=max_tokens or 1024,
                system=system_prompt,
                messages=[
                    {"role": "user", "content": prompt}
                ],
                temperature=temperature
            ) as stream:
                for event in stream:
                    # Anthropic의 이벤트 처리
                    if event.type == "content_block_delta":
                        if event.delta.type == "text_delta":
                            yield LLMDelta(
                                partial_content=event.delta.text,
                                finish_reason=None,
                                is_final=False
                            )
                    elif event.type == "message_stop":
                        yield LLMDelta(
                            partial_content="",
                            finish_reason="end_turn",
                            is_final=True
                        )
        
        except AnthropicRateLimitError as e:
            raise RateLimitError(str(e))
        except APITimeoutError as e:
            raise TimeoutError(str(e))
        except AuthenticationError as e:
            raise AuthError(str(e))
        except APIError as e:
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
        Anthropic count_tokens API를 사용한 토큰 계산
        
        Args:
            text: 입력 텍스트
        
        Returns:
            토큰 개수
        
        Note:
            Anthropic API는 별도의 token counting 엔드포인트를 제공한다.
            네트워크 오류 시 간단한 휴리스틱 사용 (1 단어 ≈ 1.3 토큰).
        """
        try:
            response = await self.client.messages.count_tokens(
                model=self.model_name,
                messages=[{"role": "user", "content": text}]
            )
            return response.input_tokens
        except Exception:
            # 폴백: 단순 추정
            return int(len(text.split()) * 1.3)
    
    async def get_model_info(self) -> Dict[str, Any]:
        """모델 정보 조회"""
        context_window = MODEL_CONTEXT_WINDOWS.get(self.model_name, 200000)
        pricing = MODEL_PRICING.get(self.model_name, {"input": 0, "output": 0})
        
        return {
            'model': self.model_name,
            'provider': self.provider_name,
            'context_window': context_window,
            'pricing': pricing,
        }
```

### 4-2. 환경 변수 검증

**파일:** `/backend/app/services/llm/anthropic_provider.py` (계속)

**추가 메서드:**
```python
@classmethod
def validate_environment(cls) -> Dict[str, bool]:
    """
    필수 환경변수 검증
    
    Returns:
        {'ANTHROPIC_API_KEY': True/False, 'ANTHROPIC_MODEL': True/False}
    """
    return {
        'ANTHROPIC_API_KEY': bool(os.getenv('ANTHROPIC_API_KEY')),
        'ANTHROPIC_MODEL': bool(os.getenv('ANTHROPIC_MODEL')),
    }

@classmethod
def is_available(cls) -> bool:
    """Provider 사용 가능 여부"""
    return bool(os.getenv('ANTHROPIC_API_KEY'))
```

### 4-3. 단위 테스트

**파일:** `/backend/tests/unit/services/test_anthropic_provider.py`

**작업 항목:**
```python
import pytest
from unittest.mock import AsyncMock, patch, MagicMock
from backend.app.services.llm.anthropic_provider import (
    AnthropicLLMProvider, LLMResponse, RateLimitError, AuthError
)

@pytest.fixture
def mock_env_vars(monkeypatch):
    """Anthropic 환경변수 설정"""
    monkeypatch.setenv('ANTHROPIC_API_KEY', 'sk-ant-test-key-12345')
    monkeypatch.setenv('ANTHROPIC_MODEL', 'claude-3-sonnet-20240229')

@pytest.fixture
async def provider(mock_env_vars):
    """Provider 인스턴스"""
    with patch('backend.app.services.llm.anthropic_provider.AsyncAnthropic'):
        return AnthropicLLMProvider()

@pytest.mark.asyncio
async def test_generate_success(provider):
    """정상 생성 응답 테스트"""
    mock_response = MagicMock()
    mock_response.content = [MagicMock(text="Claude response")]
    mock_response.stop_reason = "end_turn"
    mock_response.usage.input_tokens = 15
    mock_response.usage.output_tokens = 25
    
    provider.client.messages.create = AsyncMock(return_value=mock_response)
    
    response = await provider.generate("Test prompt")
    
    assert response.content == "Claude response"
    assert response.tokens_used == 40
    assert response.provider == "anthropic"
    assert response.cost > 0

@pytest.mark.asyncio
async def test_count_tokens(provider):
    """토큰 계산 테스트"""
    mock_token_response = MagicMock()
    mock_token_response.input_tokens = 10
    
    provider.client.messages.count_tokens = AsyncMock(return_value=mock_token_response)
    
    tokens = await provider.count_tokens("Hello world")
    assert tokens == 10

@pytest.mark.asyncio
async def test_count_tokens_fallback(provider):
    """토큰 계산 폴백 (API 오류 시)"""
    provider.client.messages.count_tokens = AsyncMock(side_effect=Exception("API Error"))
    
    tokens = await provider.count_tokens("Hello world test")
    
    # 폴백 계산: 3 단어 * 1.3 ≈ 4
    assert tokens > 0

@pytest.mark.asyncio
async def test_stream_generate(provider):
    """스트리밍 생성 테스트"""
    mock_events = [
        MagicMock(type="content_block_delta", delta=MagicMock(type="text_delta", text="Hello ")),
        MagicMock(type="content_block_delta", delta=MagicMock(type="text_delta", text="Claude")),
        MagicMock(type="message_stop"),
    ]
    
    provider.client.messages.stream = MagicMock(
        return_value=MagicMock(__enter__=AsyncMock(return_value=iter(mock_events)), __exit__=AsyncMock())
    )
    
    chunks = []
    async for delta in provider.stream_generate("Test prompt"):
        chunks.append(delta.partial_content)
    
    assert "Hello " in chunks
    assert "Claude" in chunks

@pytest.mark.asyncio
async def test_batch_generate(provider):
    """배치 생성 테스트"""
    mock_response = MagicMock()
    mock_response.content = [MagicMock(text="Response")]
    mock_response.stop_reason = "end_turn"
    mock_response.usage.input_tokens = 10
    mock_response.usage.output_tokens = 15
    
    provider.client.messages.create = AsyncMock(return_value=mock_response)
    
    prompts = ["Prompt 1", "Prompt 2"]
    responses = await provider.batch_generate(prompts)
    
    assert len(responses) == 2
    assert all(isinstance(r, LLMResponse) for r in responses)

@pytest.mark.asyncio
async def test_rate_limit_error(provider):
    """RateLimitError 처리"""
    from anthropic import RateLimitError as AnthropicRateLimitError
    provider.client.messages.create = AsyncMock(
        side_effect=AnthropicRateLimitError("429 Too Many Requests", None, None)
    )
    
    with pytest.raises(RateLimitError):
        await provider.generate("Test prompt")

@pytest.mark.asyncio
async def test_auth_error(provider):
    """AuthError 처리"""
    from anthropic import AuthenticationError
    provider.client.messages.create = AsyncMock(
        side_effect=AuthenticationError("401 Unauthorized", None, None)
    )
    
    with pytest.raises(AuthError):
        await provider.generate("Test prompt")

@pytest.mark.asyncio
async def test_context_overflow_error(provider):
    """ContextOverflowError 처리"""
    from anthropic import APIError
    provider.client.messages.create = AsyncMock(
        side_effect=APIError("max_tokens exceeded")
    )
    
    with pytest.raises(Exception):  # ContextOverflowError
        await provider.generate("Test prompt")

@pytest.mark.asyncio
async def test_validate_environment(monkeypatch):
    """환경변수 검증 테스트"""
    monkeypatch.delenv('ANTHROPIC_API_KEY', raising=False)
    
    validation = AnthropicLLMProvider.validate_environment()
    assert validation['ANTHROPIC_API_KEY'] is False

@pytest.mark.asyncio
async def test_is_available(monkeypatch):
    """Provider 가용성 테스트"""
    monkeypatch.setenv('ANTHROPIC_API_KEY', 'sk-ant-test')
    assert AnthropicLLMProvider.is_available() is True
    
    monkeypatch.delenv('ANTHROPIC_API_KEY', raising=False)
    assert AnthropicLLMProvider.is_available() is False
```

## 5. 산출물

1. **AnthropicLLMProvider 클래스** (`/backend/app/services/llm/anthropic_provider.py`)
   - 모든 추상 메서드 구현
   - 환경변수 기반 설정
   - 에러 처리 및 비용 추적
   - Anthropic API 이벤트 처리

2. **단위 테스트** (`/backend/tests/unit/services/test_anthropic_provider.py`)
   - 정상 응답, 스트리밍, 배치, 에러 케이스
   - Mock API 활용

3. **설정 예시** (`.env.example`)
   ```
   ANTHROPIC_API_KEY=sk-ant-...
   ANTHROPIC_MODEL=claude-3-sonnet-20240229
   ```

## 6. 완료 기준

1. AnthropicLLMProvider가 LLMProvider를 올바르게 상속
2. generate() 메서드가 Anthropic Messages API 호출
3. stream_generate()가 Server-Sent Events 처리
4. batch_generate()가 여러 프롬프트 순차 처리
5. count_tokens()이 Anthropic count_tokens API 사용 (폴백 포함)
6. 4개 에러 타입 모두 올바르게 처리됨
7. 비용 계산이 정확 (input/output 토큰별)
8. 단위 테스트 모두 통과 (`pytest -v`)
9. 환경변수 검증 함수 구현
10. 코드 스타일 통일 (isort, black 적용)

## 7. 작업 지침

### 지침 7-1. 스트리밍 이벤트 처리

Anthropic의 스트리밍은 `messages.stream()` 컨텍스트 매니저를 사용한다:

```python
with client.messages.stream(...) as stream:
    for event in stream:
        if event.type == "content_block_delta":
            # 텍스트 청크 처리
            text = event.delta.text
        elif event.type == "message_stop":
            # 종료 신호
            break
```

### 지침 7-2. 동기 vs 비동기 스트리밍

Anthropic SDK는 동기 `stream()`과 비동기가 분리되어 있다. 여기서는 동기 버전 사용:

```python
with self.client.messages.stream(...) as stream:  # 동기
    for event in stream:
        yield LLMDelta(...)
```

### 지침 7-3. Token Counting API

Anthropic은 별도의 token counting 엔드포인트를 제공한다:

```python
response = await client.messages.count_tokens(
    model="claude-3-sonnet-20240229",
    messages=[{"role": "user", "content": text}]
)
# response.input_tokens
```

### 지침 7-4. 모델 명시적 지정

Anthropic 모델명은 버전 포함 (예: `claude-3-sonnet-20240229`). 환경변수에는 전체 모델명 지정:

```bash
export ANTHROPIC_MODEL=claude-3-opus-20240229
```

### 지침 7-5. 가격표 업데이트

Anthropic 공식 가격: https://www.anthropic.com/pricing

가격 변경 시 `MODEL_PRICING` 딕셔너리만 수정하면 됨.
