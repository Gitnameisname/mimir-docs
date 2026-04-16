# Task 1-4. 폐쇄망 Provider 구현 (vLLM / Ollama / llama.cpp)

## 1. 작업 목적

**폐쇄망(폐쇄 인트라넷) 환경에서 외부 인터넷 없이 LLM을 사용할 수 있도록**, 세 가지 자체 호스팅 모델 서버와 통신하는 provider들을 구현한다. 이는 **S2 원칙 ⑦ (폐쇄망 환경에서 전체 기능 동작)**을 구조적으로 충족하는 핵심 작업이다.

세 provider 모두 **외부망을 사용하지 않으며 비용이 0**이고, 로컬 HTTP 서버와의 통신만 필요하다.

## 2. 작업 범위

### 포함 범위

1. **vLLMProvider 구현**
   - vLLM 서버 (OpenAI 호환 API)
   - `/v1/completions`, `/v1/chat/completions` 엔드포인트
   - 스트리밍 SSE 지원
   - 로컬 tokenizer 활용

2. **OllamaProvider 구현**
   - Ollama 서버 (`http://localhost:11434`)
   - `/api/generate`, `/api/chat` 엔드포인트
   - JSON Lines 스트리밍
   - 근사 토큰 계산

3. **LlamaCppProvider 구현**
   - llama.cpp HTTP 서버
   - `/completions`, `/chat/completions` (OpenAI 호환)
   - SSE 스트리밍
   - 기본 tokenizer

4. **공통 특성**
   - 환경변수로 서버 URL 지정
   - HTTP/HTTPS 클라이언트 (httpx)
   - 타임아웃, 재시도, 연결 오류 처리
   - **외부망 미사용 검증**
   - 비용 추적: 항상 0

5. **단위 테스트**
   - Mock HTTP 응답
   - 정상 응답, 스트리밍, 에러 케이스

### 제외 범위

- 모델 서버 설치 및 관리 (Mimir 담당자가 별도 구성)
- 성능 튜닝 (Task 외)
- GPU 최적화

## 3. 선행 조건

- Task 1-1 완료 (`LLMProvider` ABC)
- httpx 설치 (`pip install httpx`)
- (테스트 시) 세 서버 중 하나 이상 로컬 실행
  - vLLM: `python -m vllm.entrypoints.openai.api_server --model meta-llama/Llama-2-7b-hf`
  - Ollama: `ollama serve`
  - llama.cpp: `./server --model model.gguf`

## 4. 주요 작업 항목

### 4-1. vLLMProvider 구현

**파일:** `/backend/app/services/llm/vllm_provider.py`

**작업 항목:**
```python
import os
import time
import httpx
from typing import AsyncIterator, List, Optional, Dict, Any

from backend.app.services.llm.base import (
    LLMProvider, LLMResponse, LLMDelta,
    TimeoutError, LLMProviderError
)

class VLLMProvider(LLMProvider):
    """vLLM (OpenAI-compatible) Provider"""
    
    def __init__(self):
        super().__init__()
        self.provider_name = "vllm"
        
        # 환경변수에서 설정 로드
        self.base_url = os.getenv("VLLM_BASE_URL", "http://localhost:8000").rstrip("/")
        self.model_name = os.getenv("VLLM_MODEL", "default")
        
        if not self.base_url or not self.model_name:
            raise LLMProviderError(
                "VLLM_BASE_URL and VLLM_MODEL environment variables required"
            )
        
        self.http_client = httpx.AsyncClient(base_url=self.base_url, timeout=30.0)
    
    async def generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> LLMResponse:
        """vLLM Chat Completions API 호출"""
        start_time = time.time()
        
        messages = []
        if "system_prompt" in kwargs:
            messages.append({"role": "system", "content": kwargs["system_prompt"]})
        messages.append({"role": "user", "content": prompt})
        
        try:
            response = await self.http_client.post(
                "/v1/chat/completions",
                json={
                    "model": self.model_name,
                    "messages": messages,
                    "temperature": temperature,
                    "max_tokens": max_tokens or 1000,
                }
            )
            response.raise_for_status()
            
            latency_ms = (time.time() - start_time) * 1000
            result = response.json()
            
            content = result["choices"][0]["message"]["content"]
            tokens_used = result.get("usage", {}).get("total_tokens", 0)
            
            return LLMResponse(
                content=content,
                tokens_used=tokens_used,
                cost=0.0,  # 폐쇄망, 비용 없음
                latency_ms=latency_ms,
                model=self.model_name,
                provider="vllm",
                finish_reason=result["choices"][0].get("finish_reason"),
            )
        
        except httpx.TimeoutException as e:
            raise TimeoutError(f"vLLM request timeout: {str(e)}")
        except httpx.HTTPError as e:
            raise LLMProviderError(f"vLLM HTTP error: {str(e)}")
        except Exception as e:
            raise LLMProviderError(f"vLLM error: {str(e)}")
    
    async def stream_generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> AsyncIterator[LLMDelta]:
        """vLLM SSE 스트리밍"""
        messages = []
        if "system_prompt" in kwargs:
            messages.append({"role": "system", "content": kwargs["system_prompt"]})
        messages.append({"role": "user", "content": prompt})
        
        try:
            async with self.http_client.stream(
                "POST",
                "/v1/chat/completions",
                json={
                    "model": self.model_name,
                    "messages": messages,
                    "temperature": temperature,
                    "max_tokens": max_tokens or 1000,
                    "stream": True,
                }
            ) as response:
                response.raise_for_status()
                
                async for line in response.aiter_lines():
                    if line.startswith("data: "):
                        data_str = line[6:]  # "data: " 제거
                        if data_str == "[DONE]":
                            yield LLMDelta("", finish_reason="stop", is_final=True)
                            break
                        
                        try:
                            import json
                            data = json.loads(data_str)
                            content = data["choices"][0]["delta"].get("content", "")
                            if content:
                                yield LLMDelta(content, is_final=False)
                        except Exception:
                            pass
        
        except httpx.TimeoutException as e:
            raise TimeoutError(str(e))
        except httpx.HTTPError as e:
            raise LLMProviderError(str(e))
    
    async def batch_generate(
        self,
        prompts: List[str],
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> List[LLMResponse]:
        """배치 생성"""
        responses = []
        for prompt in prompts:
            response = await self.generate(prompt, temperature, max_tokens, **kwargs)
            responses.append(response)
        return responses
    
    async def count_tokens(self, text: str) -> int:
        """토큰 개수 추정 (근사)"""
        # vLLM은 별도 token count API가 없으므로 휴리스틱 사용
        return int(len(text.split()) * 1.3)
    
    async def __aenter__(self):
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.http_client.aclose()
```

### 4-2. OllamaProvider 구현

**파일:** `/backend/app/services/llm/ollama_provider.py`

**작업 항목:**
```python
import os
import time
import json
import httpx
from typing import AsyncIterator, List, Optional, Dict, Any

from backend.app.services.llm.base import (
    LLMProvider, LLMResponse, LLMDelta, TimeoutError, LLMProviderError
)

class OllamaProvider(LLMProvider):
    """Ollama Local Server Provider"""
    
    def __init__(self):
        super().__init__()
        self.provider_name = "ollama"
        
        self.base_url = os.getenv("OLLAMA_BASE_URL", "http://localhost:11434").rstrip("/")
        self.model_name = os.getenv("OLLAMA_MODEL", "llama2")
        
        if not self.base_url or not self.model_name:
            raise LLMProviderError(
                "OLLAMA_BASE_URL and OLLAMA_MODEL environment variables required"
            )
        
        self.http_client = httpx.AsyncClient(base_url=self.base_url, timeout=60.0)
    
    async def generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> LLMResponse:
        """Ollama /api/generate 엔드포인트"""
        start_time = time.time()
        
        # system prompt 처리
        system = kwargs.get("system_prompt", "")
        
        try:
            response = await self.http_client.post(
                "/api/generate",
                json={
                    "model": self.model_name,
                    "prompt": prompt,
                    "system": system,
                    "temperature": temperature,
                    "stream": False,  # 비스트리밍
                }
            )
            response.raise_for_status()
            
            latency_ms = (time.time() - start_time) * 1000
            result = response.json()
            
            # Ollama는 전체 응답을 한 번에 반환
            content = result.get("response", "")
            # Ollama는 정확한 토큰 수를 제공하지 않으므로 추정
            tokens_used = result.get("eval_count", 0)
            
            return LLMResponse(
                content=content,
                tokens_used=tokens_used,
                cost=0.0,
                latency_ms=latency_ms,
                model=self.model_name,
                provider="ollama",
                finish_reason="stop",
            )
        
        except httpx.TimeoutException as e:
            raise TimeoutError(f"Ollama request timeout: {str(e)}")
        except httpx.HTTPError as e:
            raise LLMProviderError(f"Ollama HTTP error: {str(e)}")
        except Exception as e:
            raise LLMProviderError(f"Ollama error: {str(e)}")
    
    async def stream_generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> AsyncIterator[LLMDelta]:
        """Ollama 스트리밍 (JSON Lines)"""
        system = kwargs.get("system_prompt", "")
        
        try:
            async with self.http_client.stream(
                "POST",
                "/api/generate",
                json={
                    "model": self.model_name,
                    "prompt": prompt,
                    "system": system,
                    "temperature": temperature,
                    "stream": True,
                }
            ) as response:
                response.raise_for_status()
                
                async for line in response.aiter_lines():
                    if line.strip():
                        try:
                            chunk = json.loads(line)
                            content = chunk.get("response", "")
                            done = chunk.get("done", False)
                            
                            if content:
                                yield LLMDelta(content, is_final=False)
                            
                            if done:
                                yield LLMDelta("", finish_reason="stop", is_final=True)
                                break
                        except json.JSONDecodeError:
                            pass
        
        except httpx.TimeoutException as e:
            raise TimeoutError(str(e))
        except httpx.HTTPError as e:
            raise LLMProviderError(str(e))
    
    async def batch_generate(
        self,
        prompts: List[str],
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> List[LLMResponse]:
        """배치 생성"""
        responses = []
        for prompt in prompts:
            response = await self.generate(prompt, temperature, max_tokens, **kwargs)
            responses.append(response)
        return responses
    
    async def count_tokens(self, text: str) -> int:
        """토큰 개수 추정"""
        return int(len(text.split()) * 1.3)
```

### 4-3. LlamaCppProvider 구현

**파일:** `/backend/app/services/llm/llama_cpp_provider.py`

**작업 항목:**
```python
import os
import time
import json
import httpx
from typing import AsyncIterator, List, Optional, Dict, Any

from backend.app.services.llm.base import (
    LLMProvider, LLMResponse, LLMDelta, TimeoutError, LLMProviderError
)

class LlamaCppProvider(LLMProvider):
    """llama.cpp HTTP Server Provider"""
    
    def __init__(self):
        super().__init__()
        self.provider_name = "llama_cpp"
        
        self.base_url = os.getenv("LLAMA_CPP_BASE_URL", "http://localhost:8000").rstrip("/")
        self.model_name = os.getenv("LLAMA_CPP_MODEL", "default")
        
        if not self.base_url or not self.model_name:
            raise LLMProviderError(
                "LLAMA_CPP_BASE_URL and LLAMA_CPP_MODEL environment variables required"
            )
        
        self.http_client = httpx.AsyncClient(base_url=self.base_url, timeout=30.0)
    
    async def generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> LLMResponse:
        """llama.cpp Chat Completions API"""
        start_time = time.time()
        
        messages = []
        if "system_prompt" in kwargs:
            messages.append({"role": "system", "content": kwargs["system_prompt"]})
        messages.append({"role": "user", "content": prompt})
        
        try:
            response = await self.http_client.post(
                "/chat/completions",
                json={
                    "messages": messages,
                    "temperature": temperature,
                    "max_tokens": max_tokens or 1000,
                }
            )
            response.raise_for_status()
            
            latency_ms = (time.time() - start_time) * 1000
            result = response.json()
            
            content = result["choices"][0]["message"]["content"]
            tokens_used = result.get("usage", {}).get("total_tokens", 0)
            
            return LLMResponse(
                content=content,
                tokens_used=tokens_used,
                cost=0.0,
                latency_ms=latency_ms,
                model=self.model_name,
                provider="llama_cpp",
                finish_reason=result["choices"][0].get("finish_reason"),
            )
        
        except httpx.TimeoutException as e:
            raise TimeoutError(f"llama.cpp request timeout: {str(e)}")
        except httpx.HTTPError as e:
            raise LLMProviderError(f"llama.cpp HTTP error: {str(e)}")
        except Exception as e:
            raise LLMProviderError(f"llama.cpp error: {str(e)}")
    
    async def stream_generate(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> AsyncIterator[LLMDelta]:
        """llama.cpp SSE 스트리밍"""
        messages = []
        if "system_prompt" in kwargs:
            messages.append({"role": "system", "content": kwargs["system_prompt"]})
        messages.append({"role": "user", "content": prompt})
        
        try:
            async with self.http_client.stream(
                "POST",
                "/chat/completions",
                json={
                    "messages": messages,
                    "temperature": temperature,
                    "max_tokens": max_tokens or 1000,
                    "stream": True,
                }
            ) as response:
                response.raise_for_status()
                
                async for line in response.aiter_lines():
                    if line.startswith("data: "):
                        data_str = line[6:]
                        if data_str == "[DONE]":
                            yield LLMDelta("", finish_reason="stop", is_final=True)
                            break
                        
                        try:
                            data = json.loads(data_str)
                            content = data["choices"][0]["delta"].get("content", "")
                            if content:
                                yield LLMDelta(content, is_final=False)
                        except Exception:
                            pass
        
        except httpx.TimeoutException as e:
            raise TimeoutError(str(e))
        except httpx.HTTPError as e:
            raise LLMProviderError(str(e))
    
    async def batch_generate(
        self,
        prompts: List[str],
        temperature: float = 0.7,
        max_tokens: Optional[int] = None,
        **kwargs
    ) -> List[LLMResponse]:
        """배치 생성"""
        responses = []
        for prompt in prompts:
            response = await self.generate(prompt, temperature, max_tokens, **kwargs)
            responses.append(response)
        return responses
    
    async def count_tokens(self, text: str) -> int:
        """토큰 개수 추정"""
        return int(len(text.split()) * 1.3)
```

### 4-4. 외부망 미사용 검증

**파일:** `/backend/tests/integration/test_closed_network.py`

**작업 항목:**
```python
import pytest
import httpx
from unittest.mock import patch, MagicMock
from backend.app.services.llm.vllm_provider import VLLMProvider
from backend.app.services.llm.ollama_provider import OllamaProvider
from backend.app.services.llm.llama_cpp_provider import LlamaCppProvider

@pytest.mark.asyncio
async def test_vllm_no_external_calls(monkeypatch):
    """vLLM Provider가 외부망을 사용하지 않는지 확인"""
    monkeypatch.setenv("VLLM_BASE_URL", "http://localhost:8000")
    monkeypatch.setenv("VLLM_MODEL", "test-model")
    
    provider = VLLMProvider()
    
    # httpx 클라이언트가 localhost만 호출하는지 검증
    assert provider.base_url == "http://localhost:8000"
    assert not provider.base_url.startswith("http://api")
    assert not provider.base_url.startswith("https://api")

@pytest.mark.asyncio
async def test_ollama_no_external_calls(monkeypatch):
    """Ollama Provider가 외부망을 사용하지 않는지 확인"""
    monkeypatch.setenv("OLLAMA_BASE_URL", "http://localhost:11434")
    monkeypatch.setenv("OLLAMA_MODEL", "llama2")
    
    provider = OllamaProvider()
    
    assert provider.base_url == "http://localhost:11434"
    assert "localhost" in provider.base_url

@pytest.mark.asyncio
async def test_llama_cpp_no_external_calls(monkeypatch):
    """llama.cpp Provider가 외부망을 사용하지 않는지 확인"""
    monkeypatch.setenv("LLAMA_CPP_BASE_URL", "http://localhost:8000")
    monkeypatch.setenv("LLAMA_CPP_MODEL", "model.gguf")
    
    provider = LlamaCppProvider()
    
    assert provider.base_url == "http://localhost:8000"
    assert "localhost" in provider.base_url
```

## 5. 산출물

1. **vLLMProvider 클래스** (`/backend/app/services/llm/vllm_provider.py`)
2. **OllamaProvider 클래스** (`/backend/app/services/llm/ollama_provider.py`)
3. **LlamaCppProvider 클래스** (`/backend/app/services/llm/llama_cpp_provider.py`)
4. **통합 테스트** (`/backend/tests/integration/test_closed_network.py`)
5. **설정 예시** (`.env.example`)
   ```
   VLLM_BASE_URL=http://localhost:8000
   VLLM_MODEL=meta-llama/Llama-2-7b-hf
   OLLAMA_BASE_URL=http://localhost:11434
   OLLAMA_MODEL=llama2
   LLAMA_CPP_BASE_URL=http://localhost:8000
   LLAMA_CPP_MODEL=model.gguf
   ```

## 6. 완료 기준

1. 세 provider 모두 LLMProvider를 올바르게 상속
2. 각 provider의 generate() 메서드가 정상 동작
3. stream_generate()가 해당 서버의 스트리밍 형식 처리 (SSE vs JSON Lines)
4. batch_generate()가 배치 처리
5. count_tokens()가 토큰 수 반환
6. **모든 provider에서 cost == 0.0**
7. **외부망 미사용 검증 통과** (localhost 또는 지정된 내부 URL만 사용)
8. HTTP 타임아웃, 연결 오류 등 예외 처리
9. 단위 테스트 모두 통과 (`pytest -v`)
10. 코드 스타일 통일

## 7. 작업 지침

### 지침 7-1. 서버 URL 설정

각 provider의 기본 포트:
- vLLM: `http://localhost:8000`
- Ollama: `http://localhost:11434`
- llama.cpp: `http://localhost:8000` (포트는 시작 시 지정)

운영 환경에서는 환경변수로 IP 또는 도메인 지정:

```bash
export VLLM_BASE_URL=http://192.168.1.100:8000
export OLLAMA_BASE_URL=http://internal-server:11434
```

### 지침 7-2. 스트리밍 형식 차이

세 서버는 다른 스트리밍 형식을 사용한다:

- **vLLM**: OpenAI 호환 SSE (`data: {json}`)
- **Ollama**: JSON Lines (`{json}\n{json}`)
- **llama.cpp**: OpenAI 호환 SSE

### 지침 7-3. 타임아웃 설정

각 서버의 응답 속도가 다르므로 타임아웃을 조정할 수 있다:

```python
self.http_client = httpx.AsyncClient(base_url=self.base_url, timeout=60.0)
```

Ollama는 모델 로드 시 오래 걸릴 수 있으므로 60초 권장.

### 지침 7-4. 에러 처리

네트워크 오류 시 "서버에 연결할 수 없음"을 명확히 구분:

```python
except httpx.ConnectError:
    raise LLMProviderError(
        f"Cannot connect to {self.provider_name} server at {self.base_url}. "
        "Is the server running?"
    )
```

### 지침 7-5. 폐쇄망 배포 검증

배포 체크리스트:

- [ ] `.env` 파일에서 모든 BASE_URL이 localhost 또는 내부 IP
- [ ] DNS 조회 없음 (IP 주소 직접 사용 권장)
- [ ] 외부 인터넷 연결 필요 없음
- [ ] 모든 provider의 cost = 0.0
- [ ] 로그에 외부 도메인 참조 없음
