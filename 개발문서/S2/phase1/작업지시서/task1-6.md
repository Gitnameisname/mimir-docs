# Task 1-6. EmbeddingProvider 인터페이스 설계 및 OpenAI Embedding Provider 구현

## 1. 작업 목적

서로 다른 임베딩 모델(OpenAI, BGE-M3, E5 등)을 단일 인터페이스로 통합하기 위한 **추상 베이스 클래스(ABC)와 OpenAI Embedding Provider 구현**을 완성한다. 이는 Phase 1의 임베딩 추상화 기초이며, 모든 Embedding Provider 구현체가 상속받을 계약(contract)을 정의한다. S1 Phase 10에서 사용 중인 OpenAI text-embedding-3-small을 먼저 마이그레이션하여 신규 인터페이스 검증을 수행한다.

## 2. 작업 범위

### 포함 범위

1. **EmbeddingProvider 추상 베이스 클래스**
   - 핵심 메서드: `embed(text: str) → np.ndarray` (1D 벡터)
   - 배치 처리: `embed_batch(texts: List[str]) → List[np.ndarray]`
   - 차원 조회: `get_dimension() → int`
   - 모델명 조회: `get_model_name() → str`
   - 응답 구조: `EmbeddingResponse` 데이터 클래스

2. **OpenAI Embedding Provider 구현**
   - 모델: `text-embedding-3-small` (기본, 1536 차원), `text-embedding-3-large` (선택, 3072 차원)
   - 배치 최대: 2048 건
   - 비용: $0.02 / 1M tokens
   - 환경변수: `OPENAI_API_KEY`, `OPENAI_EMBEDDING_MODEL`
   - 토큰 사용량 및 비용 추적

3. **에러 계층 구조**
   - `EmbeddingProviderError`: 기본 예외
   - `DimensionMismatchError`: 벡터 차원 불일치
   - `RateLimitError`: API 비율 제한
   - `AuthError`: 인증 실패
   - `InvalidInputError`: 입력 유효성 실패

4. **설정 및 응답 모델**
   - `EmbeddingResponse`: vector, dimension, tokens_used, cost, model
   - `EmbeddingConfig`: 공통 설정 (timeout, retry_count 등)

### 제외 범위

- BGE-M3, E5 등 로컬 모델 구현 (Task 1-7에서)
- Factory 패턴 구현 (Task 1-8에서)
- 재임베딩 마이그레이션 파이프라인 (Task 1-8에서)
- DB 모델 수정

## 3. 선행 조건

- Phase 0 완료 (Python 환경, FastAPI 기본 구조)
- `/backend/app/services/` 디렉토리 접근 가능
- OpenAI Python 라이브러리 (`openai>=1.0.0`) 설치
- Pydantic v2 설치 확인
- Python 3.10+
- S1 Phase 10 document_chunks 벡터화 관련 코드 검토

## 4. 주요 작업 항목

### 4-1. EmbeddingProvider 추상 베이스 클래스 설계

**파일:** `/backend/app/services/embedding/base.py`

**작업 항목:**
```python
from abc import ABC, abstractmethod
from typing import List, Optional, Dict, Any
from dataclasses import dataclass
import numpy as np

@dataclass
class EmbeddingResponse:
    """임베딩 응답 데이터 모델"""
    vector: np.ndarray              # 1D 벡터 (shape: (dimension,))
    dimension: int                  # 벡터 차원 (1536, 3072, 1024, 768 등)
    tokens_used: int                # 사용된 토큰 수 (OpenAI용)
    cost: float                     # 추정 비용 (달러, 로컬은 0)
    model: str                      # 사용한 모델명
    provider: str                   # 제공자 (openai, bge, e5 등)

class EmbeddingProviderError(Exception):
    """Embedding Provider 기본 예외"""
    pass

class DimensionMismatchError(EmbeddingProviderError):
    """벡터 차원 불일치"""
    pass

class RateLimitError(EmbeddingProviderError):
    """API 비율 제한 초과"""
    pass

class AuthError(EmbeddingProviderError):
    """인증 실패"""
    pass

class InvalidInputError(EmbeddingProviderError):
    """입력 유효성 실패"""
    pass

class EmbeddingProvider(ABC):
    """임베딩 모델 추상 인터페이스"""
    
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
    async def embed(self, text: str) -> EmbeddingResponse:
        """
        단일 텍스트 임베딩
        
        Args:
            text: 임베딩할 텍스트 (빈 문자열 제외)
        
        Returns:
            EmbeddingResponse 객체 (vector는 1D np.ndarray)
        
        Raises:
            InvalidInputError: 텍스트가 비어있거나 유효하지 않음
            RateLimitError: API 비율 제한
            AuthError: 인증 실패
            EmbeddingProviderError: 기타 오류
        """
        pass
    
    @abstractmethod
    async def embed_batch(self, texts: List[str]) -> List[EmbeddingResponse]:
        """
        배치 텍스트 임베딩
        
        Args:
            texts: 임베딩할 텍스트 리스트 (최대 2048개, 빈 문자열 제외)
        
        Returns:
            EmbeddingResponse 리스트 (입력 순서 보장)
        
        Raises:
            InvalidInputError: 입력 리스트가 비어있거나 텍스트가 유효하지 않음
            RateLimitError: API 비율 제한
            AuthError: 인증 실패
            EmbeddingProviderError: 기타 오류
        """
        pass
    
    @abstractmethod
    def get_dimension(self) -> int:
        """
        임베딩 벡터의 차원 반환
        
        Returns:
            정수 차원 (1536, 3072, 1024, 768 등)
        """
        pass
    
    @abstractmethod
    def get_model_name(self) -> str:
        """
        모델명 반환
        
        Returns:
            모델명 문자열 (text-embedding-3-small 등)
        """
        pass
    
    async def validate_input(self, texts: List[str]) -> None:
        """
        입력 유효성 검사 (선택적 기본 구현)
        
        Args:
            texts: 텍스트 리스트
        
        Raises:
            InvalidInputError: 입력이 유효하지 않음
        """
        if not texts:
            raise InvalidInputError("텍스트 리스트가 비어있습니다")
        
        for i, text in enumerate(texts):
            if not isinstance(text, str):
                raise InvalidInputError(f"인덱스 {i}: 문자열이 아닙니다")
            if text.strip() == "":
                raise InvalidInputError(f"인덱스 {i}: 빈 문자열입니다")
```

### 4-2. OpenAI Embedding Provider 구현

**파일:** `/backend/app/services/embedding/openai_embedding.py`

**작업 항목:**
```python
import os
import time
from typing import List, Optional, Dict, Any
import numpy as np
from openai import AsyncOpenAI, RateLimitError as OpenAIRateLimitError
from openai import AuthenticationError as OpenAIAuthError

from .base import (
    EmbeddingProvider, EmbeddingResponse,
    RateLimitError, AuthError, InvalidInputError, EmbeddingProviderError
)

class OpenAIEmbeddingProvider(EmbeddingProvider):
    """OpenAI text-embedding-3 시리즈 제공자"""
    
    # 모델별 차원 맵
    MODEL_DIMENSIONS = {
        "text-embedding-3-small": 1536,
        "text-embedding-3-large": 3072,
    }
    
    # 비용 정보 ($0.02 / 1M tokens)
    COST_PER_MILLION_TOKENS = 0.02
    
    def __init__(self, config: Optional[Dict[str, Any]] = None):
        """
        OpenAI Embedding Provider 초기화
        
        Args:
            config: 설정 딕셔너리
                - model: text-embedding-3-small (기본) 또는 text-embedding-3-large
                - api_key: OPENAI_API_KEY 환경변수 (기본)
                - timeout: API 타임아웃 (초, 기본 30)
                - retry_count: 재시도 횟수 (기본 3)
        
        Raises:
            AuthError: OPENAI_API_KEY 미설정
        """
        super().__init__(config)
        
        api_key = config.get("api_key") if config else None
        if not api_key:
            api_key = os.getenv("OPENAI_API_KEY")
        
        if not api_key:
            raise AuthError("OPENAI_API_KEY 환경변수가 설정되지 않았습니다")
        
        self.client = AsyncOpenAI(api_key=api_key)
        
        # 모델명 설정
        model_name = config.get("model", "text-embedding-3-small") if config else "text-embedding-3-small"
        if model_name not in self.MODEL_DIMENSIONS:
            raise InvalidInputError(f"지원하지 않는 모델: {model_name}")
        
        self.model_name = model_name
        self.provider_name = "openai"
        self._dimension = self.MODEL_DIMENSIONS[model_name]
        
        self.timeout = config.get("timeout", 30) if config else 30
        self.retry_count = config.get("retry_count", 3) if config else 3
    
    def get_dimension(self) -> int:
        """OpenAI 임베딩 차원 반환"""
        return self._dimension
    
    def get_model_name(self) -> str:
        """모델명 반환"""
        return self.model_name
    
    async def embed(self, text: str) -> EmbeddingResponse:
        """단일 텍스트 임베딩"""
        if not isinstance(text, str) or text.strip() == "":
            raise InvalidInputError("유효한 문자열을 제공해주세요")
        
        try:
            responses = await self.embed_batch([text])
            return responses[0]
        except EmbeddingProviderError:
            raise
        except OpenAIRateLimitError as e:
            raise RateLimitError(f"OpenAI API 비율 제한: {str(e)}")
        except OpenAIAuthError as e:
            raise AuthError(f"OpenAI 인증 실패: {str(e)}")
        except Exception as e:
            raise EmbeddingProviderError(f"임베딩 실패: {str(e)}")
    
    async def embed_batch(self, texts: List[str]) -> List[EmbeddingResponse]:
        """
        배치 텍스트 임베딩
        
        Args:
            texts: 임베딩할 텍스트 리스트 (최대 2048개)
        
        Returns:
            EmbeddingResponse 리스트
        """
        await self.validate_input(texts)
        
        if len(texts) > 2048:
            raise InvalidInputError(f"배치 크기는 2048 이하여야 합니다 (입력: {len(texts)})")
        
        try:
            start_time = time.time()
            response = await self.client.embeddings.create(
                model=self.model_name,
                input=texts,
                timeout=self.timeout
            )
            latency_ms = (time.time() - start_time) * 1000
            
            # 토큰 사용량 및 비용 계산
            tokens_used = response.usage.total_tokens
            cost = (tokens_used / 1_000_000) * self.COST_PER_MILLION_TOKENS
            
            # 벡터 추출 및 검증
            results = []
            for i, embedding_data in enumerate(response.data):
                vector = np.array(embedding_data.embedding, dtype=np.float32)
                
                # 벡터 차원 검증
                if vector.shape[0] != self._dimension:
                    raise EmbeddingProviderError(
                        f"차원 불일치: 예상 {self._dimension}, 실제 {vector.shape[0]}"
                    )
                
                result = EmbeddingResponse(
                    vector=vector,
                    dimension=self._dimension,
                    tokens_used=tokens_used,
                    cost=cost,
                    model=self.model_name,
                    provider=self.provider_name
                )
                results.append(result)
            
            return results
        
        except EmbeddingProviderError:
            raise
        except OpenAIRateLimitError as e:
            raise RateLimitError(f"OpenAI API 비율 제한: {str(e)}")
        except OpenAIAuthError as e:
            raise AuthError(f"OpenAI 인증 실패: {str(e)}")
        except Exception as e:
            raise EmbeddingProviderError(f"배치 임베딩 실패: {str(e)}")
```

### 4-3. Pydantic 스키마 정의

**파일:** `/backend/app/schemas/embedding.py`

**작업 항목:**
```python
from pydantic import BaseModel, Field
from typing import Optional, List
from enum import Enum

class EmbeddingProviderEnum(str, Enum):
    """임베딩 제공자"""
    OPENAI = "openai"
    BGE = "bge"
    E5 = "e5"

class EmbeddingModel(str, Enum):
    """지원 임베딩 모델"""
    OPENAI_SMALL = "text-embedding-3-small"
    OPENAI_LARGE = "text-embedding-3-large"
    BGE_M3 = "BAAI/bge-m3"
    E5_BASE = "intfloat/e5-base-v2"
    E5_LARGE = "intfloat/e5-large"

class EmbedRequestSchema(BaseModel):
    """임베딩 요청 스키마"""
    text: str = Field(..., min_length=1, max_length=8192)
    
    class Config:
        json_schema_extra = {
            "example": {
                "text": "다음 문장을 임베딩합니다."
            }
        }

class EmbedBatchRequestSchema(BaseModel):
    """배치 임베딩 요청 스키마"""
    texts: List[str] = Field(..., min_items=1, max_items=2048)
    
    class Config:
        json_schema_extra = {
            "example": {
                "texts": ["첫 번째 문장", "두 번째 문장", "세 번째 문장"]
            }
        }

class EmbedResponseSchema(BaseModel):
    """임베딩 응답 스키마"""
    dimension: int = Field(..., ge=256, le=4096)
    tokens_used: int = Field(default=0, ge=0)
    cost: float = Field(default=0.0, ge=0)
    model: str
    provider: str
    
    class Config:
        json_schema_extra = {
            "example": {
                "dimension": 1536,
                "tokens_used": 45,
                "cost": 0.0009,
                "model": "text-embedding-3-small",
                "provider": "openai"
            }
        }
```

### 4-4. 설정 클래스

**파일:** `/backend/app/services/embedding/config.py`

**작업 항목:**
```python
from dataclasses import dataclass, field
from typing import Optional, Dict, Any

@dataclass
class EmbeddingConfig:
    """임베딩 공통 설정"""
    provider: str = "openai"                # 제공자 (openai, bge, e5)
    model: str = "text-embedding-3-small"   # 모델명
    timeout: int = 30                       # API 타임아웃 (초)
    retry_count: int = 3                    # 재시도 횟수
    retry_delay: float = 1.0                # 재시도 간격 (초)
    batch_size: int = 32                    # 기본 배치 크기
    enable_caching: bool = False            # 임베딩 캐싱 활성화
    cache_dir: Optional[str] = None         # 캐시 디렉토리
    extra_params: Dict[str, Any] = field(default_factory=dict)
```

### 4-5. 단위 테스트 (기본 구조)

**파일:** `/backend/tests/unit/services/test_embedding_base.py`

**작업 항목:**
```python
import pytest
import numpy as np
from unittest.mock import AsyncMock, patch, MagicMock

from backend.app.services.embedding.base import (
    EmbeddingProvider, EmbeddingResponse,
    InvalidInputError, EmbeddingProviderError
)

class MockEmbeddingProvider(EmbeddingProvider):
    """테스트용 Mock Provider"""
    
    def __init__(self):
        super().__init__()
        self.model_name = "mock-embedding"
        self.provider_name = "mock"
        self._dimension = 768
    
    def get_dimension(self) -> int:
        return self._dimension
    
    def get_model_name(self) -> str:
        return self.model_name
    
    async def embed(self, text: str) -> EmbeddingResponse:
        if not text or text.strip() == "":
            raise InvalidInputError("빈 문자열")
        
        return EmbeddingResponse(
            vector=np.random.randn(self._dimension).astype(np.float32),
            dimension=self._dimension,
            tokens_used=10,
            cost=0.0,
            model=self.model_name,
            provider=self.provider_name
        )
    
    async def embed_batch(self, texts: list) -> list:
        await self.validate_input(texts)
        return [await self.embed(text) for text in texts]

@pytest.mark.asyncio
async def test_embedding_response_creation():
    """EmbeddingResponse 객체 생성 테스트"""
    vector = np.array([0.1, 0.2, 0.3], dtype=np.float32)
    response = EmbeddingResponse(
        vector=vector,
        dimension=3,
        tokens_used=5,
        cost=0.001,
        model="test-model",
        provider="test"
    )
    assert response.dimension == 3
    assert response.tokens_used == 5
    np.testing.assert_array_equal(response.vector, vector)

@pytest.mark.asyncio
async def test_mock_provider_embed():
    """Mock Provider의 embed 메서드 테스트"""
    provider = MockEmbeddingProvider()
    response = await provider.embed("테스트 텍스트")
    assert response.dimension == 768
    assert response.vector.shape == (768,)

@pytest.mark.asyncio
async def test_mock_provider_embed_batch():
    """Mock Provider의 embed_batch 테스트"""
    provider = MockEmbeddingProvider()
    responses = await provider.embed_batch(["텍스트1", "텍스트2"])
    assert len(responses) == 2
    assert all(r.dimension == 768 for r in responses)

@pytest.mark.asyncio
async def test_invalid_input_empty_string():
    """빈 문자열 입력 에러 테스트"""
    provider = MockEmbeddingProvider()
    with pytest.raises(InvalidInputError):
        await provider.embed("")

@pytest.mark.asyncio
async def test_invalid_input_empty_list():
    """빈 리스트 입력 에러 테스트"""
    provider = MockEmbeddingProvider()
    with pytest.raises(InvalidInputError):
        await provider.embed_batch([])
```

## 5. 산출물

1. **EmbeddingProvider 추상 베이스 클래스** (`/backend/app/services/embedding/base.py`)
   - ABC, abstractmethod 정의 (embed, embed_batch, get_dimension, get_model_name)
   - 5개 에러 클래스 (EmbeddingProviderError, DimensionMismatchError, RateLimitError, AuthError, InvalidInputError)
   - EmbeddingResponse 데이터 클래스

2. **OpenAI Embedding Provider** (`/backend/app/services/embedding/openai_embedding.py`)
   - text-embedding-3-small (1536 차원) 및 text-embedding-3-large (3072 차원) 지원
   - 환경변수 기반 API 키 및 모델명 설정
   - 배치 처리 (최대 2048개), 토큰 사용량 추적, 비용 계산

3. **Pydantic 스키마** (`/backend/app/schemas/embedding.py`)
   - EmbedRequestSchema, EmbedBatchRequestSchema, EmbedResponseSchema
   - EmbeddingProviderEnum, EmbeddingModel Enum

4. **설정 클래스** (`/backend/app/services/embedding/config.py`)
   - EmbeddingConfig 데이터 클래스

5. **단위 테스트** (`/backend/tests/unit/services/test_embedding_base.py`)
   - Mock Provider 구현, 기본 동작 테스트

## 6. 완료 기준

1. `EmbeddingProvider` ABC 정의되고, 4개 abstractmethod 명시 (embed, embed_batch, get_dimension, get_model_name)
2. `EmbeddingResponse` 데이터 클래스 정상 작동, vector는 np.ndarray (1D)
3. 5개 에러 클래스 올바른 상속 관계 형성
4. OpenAI Provider: text-embedding-3-small 정상 동작 (1536 차원, 토큰 추적, 비용 계산)
5. OpenAI Provider: text-embedding-3-large 정상 동작 (3072 차원)
6. 배치 처리 (최대 2048개) 정상 작동, 입력 순서 보장
7. 환경변수 `OPENAI_API_KEY`, `OPENAI_EMBEDDING_MODEL` 정상 읽음
8. 단위 테스트 모두 통과 (`pytest -v`)
9. 타입 힌트 정확 (mypy 검사 통과)
10. 모든 메서드 docstring 포함, 예외 처리 명시
11. S1 Phase 10 기존 OpenAI 임베딩 호출 자동 마이그레이션 불필요 (Task 1-8에서)

## 7. 작업 지침

### 지침 7-1. 벡터 데이터 타입

모든 벡터는 `np.float32` 타입으로 반환. pgvector 호환성 및 메모리 효율:
```python
vector = np.array(embedding_data.embedding, dtype=np.float32)
```

### 지침 7-2. 차원 검증

모든 임베딩 응답에서 벡터 차원을 검증하여 Task 1-8 마이그레이션 준비:
```python
if vector.shape[0] != expected_dimension:
    raise DimensionMismatchError(...)
```

### 지침 7-3. 환경변수 설정

- `OPENAI_API_KEY`: 필수, sk-... 형식
- `OPENAI_EMBEDDING_MODEL`: 선택, 기본값 `text-embedding-3-small`

### 지침 7-4. 에러 처리 계층

에러는 구체성 순서로 catch:
```python
try:
    response = await provider.embed(text)
except InvalidInputError:
    # 입력 유효성
except RateLimitError:
    # 비율 제한 + 재시도
except AuthError:
    # API 키 확인
except EmbeddingProviderError:
    # 기타 오류
```

### 지침 7-5. 비용 추적

OpenAI 비용은 토큰 기준으로 계산:
```python
cost = (tokens_used / 1_000_000) * 0.02  # $0.02 / 1M tokens
```

### 지침 7-6. 코드 스타일

- 모든 async 메서드는 `async def` 형식
- 타입 힌트 필수
- docstring은 Google style
- import 정렬: stdlib → third-party → local (isort)
