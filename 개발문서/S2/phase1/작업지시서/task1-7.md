# Task 1-7. 로컬 Embedding Provider 구현 (BGE-M3 / E5)

## 1. 작업 목적

**폐쇄망 환경 완벽 지원**을 위해 로컬에서 실행 가능한 BGE-M3와 E5 임베딩 모델 구현을 완성한다. 외부 API 의존성을 제거하면서 높은 임베딩 품질을 제공하며, S2 원칙 ⑦(폐쇄망 환경에서 전체 기능 동작)을 충족한다. 두 모델은 오프라인 환경에서 선택 가능하며, 모델 다운로드 및 캐싱을 자동으로 관리한다.

## 2. 작업 범위

### 포함 범위

1. **BGE-M3 Provider 구현**
   - 모델: `BAAI/bge-m3` (sentence-transformers)
   - 차원: 1024
   - 특성: 다국어 지원, 우수한 검색 성능
   - 로컬 실행, 비용 0
   - 환경변수: `BGE_MODEL_PATH` (선택, 기본값: HuggingFace hub 자동 다운로드)

2. **E5 Provider 구현**
   - 모델: `intfloat/e5-base-v2` (768 차원) 또는 `intfloat/e5-large` (1024 차원)
   - 로컬 실행, 비용 0
   - 특성: 높은 품질, 다국어 지원
   - 환경변수: `E5_MODEL_PATH`, `E5_MODEL_SIZE` (base|large)

3. **모델 관리 및 캐싱**
   - 자동 모델 다운로드 (첫 사용 시)
   - HuggingFace hub 캐싱 활용
   - 사전 로드 옵션 (startup latency 최소화)
   - 메모리 효율성 (batch processing)

4. **배치 처리 최적화**
   - 대량 텍스트 효율적 처리
   - GPU 가속 지원 (CUDA/CPU 자동 선택)
   - 배치 크기 조정 가능

5. **폐쇄망 환경 지원**
   - 환경변수로 HuggingFace hub 다운로드 disable 가능
   - 로컬 모델 경로 지정 가능
   - 인터넷 없이 동작 (미리 다운로드한 경우)

### 제외 범위

- OpenAI Provider 구현 (Task 1-6에서)
- Factory 패턴 (Task 1-8에서)
- 재임베딩 마이그레이션 (Task 1-8에서)
- 모델 학습 또는 파인튜닝
- 다른 오픈소스 모델 (DPR, SBERT 등)

## 3. 선행 조건

- Phase 0 완료
- Task 1-6 완료 (EmbeddingProvider 인터페이스)
- `sentence-transformers>=2.2.0` 설치
- `torch>=2.0.0` 설치
- Python 3.10+
- 메모리 최소 4GB (권장 8GB)
- 인터넷 연결 (첫 모델 다운로드 시, 약 500MB~2GB)

## 4. 주요 작업 항목

### 4-1. BGE-M3 Embedding Provider 구현

**파일:** `/backend/app/services/embedding/bge_provider.py`

**작업 항목:**
```python
import os
import logging
from typing import List, Optional, Dict, Any
import numpy as np
from sentence_transformers import SentenceTransformer
import torch

from .base import (
    EmbeddingProvider, EmbeddingResponse,
    InvalidInputError, EmbeddingProviderError
)

logger = logging.getLogger(__name__)

class BGEEmbeddingProvider(EmbeddingProvider):
    """BAAI BGE-M3 로컬 임베딩 제공자"""
    
    MODEL_NAME = "BAAI/bge-m3"
    DIMENSION = 1024
    
    def __init__(self, config: Optional[Dict[str, Any]] = None):
        """
        BGE-M3 Provider 초기화
        
        Args:
            config: 설정 딕셔너리
                - model_path: 로컬 모델 경로 (기본: HuggingFace hub)
                - device: torch device (기본: cuda if available else cpu)
                - preload: 초기화 시 모델 로드 (기본: False)
                - batch_size: 배치 크기 (기본: 32)
                - normalize_embeddings: 임베딩 정규화 (기본: True)
        
        Raises:
            InvalidInputError: 모델 경로가 유효하지 않음
            EmbeddingProviderError: 모델 로드 실패
        """
        super().__init__(config)
        
        self.model_name = self.MODEL_NAME
        self.provider_name = "bge"
        self._dimension = self.DIMENSION
        
        # 설정 파싱
        model_path = config.get("model_path") if config else None
        if not model_path:
            model_path = os.getenv("BGE_MODEL_PATH")
        
        if not model_path:
            model_path = self.MODEL_NAME
        
        # 디바이스 설정
        self.device = config.get("device") if config else None
        if not self.device:
            self.device = "cuda" if torch.cuda.is_available() else "cpu"
        
        self.batch_size = config.get("batch_size", 32) if config else 32
        self.normalize = config.get("normalize_embeddings", True) if config else True
        
        # 모델 로드
        try:
            logger.info(f"BGE-M3 모델 로드 중... (path={model_path}, device={self.device})")
            self.model = SentenceTransformer(model_path, device=self.device)
            logger.info("BGE-M3 모델 로드 완료")
        except Exception as e:
            raise EmbeddingProviderError(f"BGE-M3 모델 로드 실패: {str(e)}")
    
    def get_dimension(self) -> int:
        """BGE-M3 임베딩 차원"""
        return self._dimension
    
    def get_model_name(self) -> str:
        """모델명"""
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
        except Exception as e:
            raise EmbeddingProviderError(f"임베딩 실패: {str(e)}")
    
    async def embed_batch(self, texts: List[str]) -> List[EmbeddingResponse]:
        """
        배치 텍스트 임베딩
        
        Args:
            texts: 임베딩할 텍스트 리스트
        
        Returns:
            EmbeddingResponse 리스트 (입력 순서 보장)
        """
        await self.validate_input(texts)
        
        try:
            # 배치 처리 (메모리 효율성)
            embeddings = self.model.encode(
                texts,
                batch_size=self.batch_size,
                normalize_embeddings=self.normalize,
                show_progress_bar=False,
                convert_to_numpy=True
            )
            
            results = []
            for i, embedding in enumerate(embeddings):
                vector = np.array(embedding, dtype=np.float32)
                
                if vector.shape[0] != self._dimension:
                    raise EmbeddingProviderError(
                        f"차원 불일치: 예상 {self._dimension}, 실제 {vector.shape[0]}"
                    )
                
                result = EmbeddingResponse(
                    vector=vector,
                    dimension=self._dimension,
                    tokens_used=0,  # 로컬이므로 0
                    cost=0.0,
                    model=self.model_name,
                    provider=self.provider_name
                )
                results.append(result)
            
            return results
        
        except EmbeddingProviderError:
            raise
        except Exception as e:
            raise EmbeddingProviderError(f"배치 임베딩 실패: {str(e)}")
```

### 4-2. E5 Embedding Provider 구현

**파일:** `/backend/app/services/embedding/e5_provider.py`

**작업 항목:**
```python
import os
import logging
from typing import List, Optional, Dict, Any, Literal
import numpy as np
from sentence_transformers import SentenceTransformer
import torch

from .base import (
    EmbeddingProvider, EmbeddingResponse,
    InvalidInputError, EmbeddingProviderError
)

logger = logging.getLogger(__name__)

class E5EmbeddingProvider(EmbeddingProvider):
    """Intfloat E5 로컬 임베딩 제공자"""
    
    MODELS = {
        "base": {
            "name": "intfloat/e5-base-v2",
            "dimension": 768
        },
        "large": {
            "name": "intfloat/e5-large",
            "dimension": 1024
        }
    }
    
    def __init__(self, config: Optional[Dict[str, Any]] = None):
        """
        E5 Provider 초기화
        
        Args:
            config: 설정 딕셔너리
                - model_size: base (768d) 또는 large (1024d), 기본 base
                - model_path: 로컬 모델 경로 (기본: HuggingFace hub)
                - device: torch device (기본: cuda if available else cpu)
                - batch_size: 배치 크기 (기본: 32)
                - normalize_embeddings: 임베딩 정규화 (기본: True)
        
        Raises:
            InvalidInputError: 모델 크기가 유효하지 않음
            EmbeddingProviderError: 모델 로드 실패
        """
        super().__init__(config)
        
        # 모델 크기 설정
        model_size = config.get("model_size", "base") if config else "base"
        if model_size not in self.MODELS:
            raise InvalidInputError(f"유효하지 않은 모델 크기: {model_size}. base 또는 large만 가능")
        
        model_info = self.MODELS[model_size]
        self.model_name = model_info["name"]
        self._dimension = model_info["dimension"]
        self.provider_name = "e5"
        
        # 로컬 모델 경로 설정
        model_path = config.get("model_path") if config else None
        if not model_path:
            model_path = os.getenv("E5_MODEL_PATH")
        
        if not model_path:
            model_path = self.model_name
        
        # 디바이스 설정
        self.device = config.get("device") if config else None
        if not self.device:
            self.device = "cuda" if torch.cuda.is_available() else "cpu"
        
        self.batch_size = config.get("batch_size", 32) if config else 32
        self.normalize = config.get("normalize_embeddings", True) if config else True
        
        # 모델 로드
        try:
            logger.info(f"E5 모델 로드 중... (model={self.model_name}, device={self.device})")
            self.model = SentenceTransformer(model_path, device=self.device)
            logger.info(f"E5 모델 로드 완료 (dimension={self._dimension})")
        except Exception as e:
            raise EmbeddingProviderError(f"E5 모델 로드 실패: {str(e)}")
    
    def get_dimension(self) -> int:
        """E5 임베딩 차원"""
        return self._dimension
    
    def get_model_name(self) -> str:
        """모델명"""
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
        except Exception as e:
            raise EmbeddingProviderError(f"임베딩 실패: {str(e)}")
    
    async def embed_batch(self, texts: List[str]) -> List[EmbeddingResponse]:
        """
        배치 텍스트 임베딩
        
        Args:
            texts: 임베딩할 텍스트 리스트
        
        Returns:
            EmbeddingResponse 리스트 (입력 순서 보장)
        """
        await self.validate_input(texts)
        
        try:
            # 배치 처리
            embeddings = self.model.encode(
                texts,
                batch_size=self.batch_size,
                normalize_embeddings=self.normalize,
                show_progress_bar=False,
                convert_to_numpy=True
            )
            
            results = []
            for i, embedding in enumerate(embeddings):
                vector = np.array(embedding, dtype=np.float32)
                
                if vector.shape[0] != self._dimension:
                    raise EmbeddingProviderError(
                        f"차원 불일치: 예상 {self._dimension}, 실제 {vector.shape[0]}"
                    )
                
                result = EmbeddingResponse(
                    vector=vector,
                    dimension=self._dimension,
                    tokens_used=0,
                    cost=0.0,
                    model=self.model_name,
                    provider=self.provider_name
                )
                results.append(result)
            
            return results
        
        except EmbeddingProviderError:
            raise
        except Exception as e:
            raise EmbeddingProviderError(f"배치 임베딩 실패: {str(e)}")
```

### 4-3. 모델 관리 유틸리티

**파일:** `/backend/app/services/embedding/model_manager.py`

**작업 항목:**
```python
import os
import logging
from typing import Optional, Dict, Any
from pathlib import Path
from sentence_transformers import SentenceTransformer
import torch

logger = logging.getLogger(__name__)

class EmbeddingModelManager:
    """임베딩 모델 생명 주기 관리"""
    
    @staticmethod
    def get_cache_dir() -> Path:
        """HuggingFace 캐시 디렉토리"""
        cache_dir = os.getenv("SENTENCE_TRANSFORMERS_HOME")
        if not cache_dir:
            cache_dir = Path.home() / ".cache" / "sentence-transformers"
        return Path(cache_dir)
    
    @staticmethod
    def model_exists(model_name: str, local_path: Optional[str] = None) -> bool:
        """모델 파일 존재 여부 확인"""
        if local_path:
            return os.path.isdir(local_path)
        
        cache_dir = EmbeddingModelManager.get_cache_dir()
        model_dir = cache_dir / model_name.replace("/", "--")
        return model_dir.exists()
    
    @staticmethod
    def preload_model(model_name: str, device: str = "cpu") -> SentenceTransformer:
        """
        모델 사전 로드 (startup latency 최소화)
        
        Args:
            model_name: HuggingFace 모델명 또는 로컬 경로
            device: torch device
        
        Returns:
            로드된 SentenceTransformer 객체
        """
        try:
            logger.info(f"모델 사전 로드 시작... (model={model_name}, device={device})")
            model = SentenceTransformer(model_name, device=device)
            logger.info("모델 사전 로드 완료")
            return model
        except Exception as e:
            logger.error(f"모델 사전 로드 실패: {str(e)}")
            raise
    
    @staticmethod
    def get_device() -> str:
        """사용 가능한 torch device 반환"""
        if torch.cuda.is_available():
            return "cuda"
        elif hasattr(torch.backends, "mps") and torch.backends.mps.is_available():
            return "mps"  # Apple Silicon
        else:
            return "cpu"
    
    @staticmethod
    def get_model_info(model_name: str) -> Dict[str, Any]:
        """모델 정보 조회"""
        info = {
            "model_name": model_name,
            "cache_dir": str(EmbeddingModelManager.get_cache_dir()),
            "available_device": EmbeddingModelManager.get_device(),
            "cuda_available": torch.cuda.is_available()
        }
        
        if torch.cuda.is_available():
            info["cuda_device_count"] = torch.cuda.device_count()
            info["cuda_device_name"] = torch.cuda.get_device_name(0)
        
        return info
```

### 4-4. 단위 테스트

**파일:** `/backend/tests/unit/services/test_embedding_local.py`

**작업 항목:**
```python
import pytest
import numpy as np
import os
from unittest.mock import patch, MagicMock

# 주의: 실제 모델 로드는 오래 걸리므로, Mock 또는 작은 테스트 모델 사용

@pytest.mark.asyncio
async def test_bge_provider_invalid_input():
    """BGE Provider 입력 유효성 테스트"""
    from backend.app.services.embedding.bge_provider import BGEEmbeddingProvider
    from backend.app.services.embedding.base import InvalidInputError
    
    with patch('backend.app.services.embedding.bge_provider.SentenceTransformer'):
        provider = BGEEmbeddingProvider()
        
        with pytest.raises(InvalidInputError):
            await provider.embed("")
        
        with pytest.raises(InvalidInputError):
            await provider.embed_batch([])

@pytest.mark.asyncio
async def test_e5_provider_model_size_validation():
    """E5 Provider 모델 크기 검증"""
    from backend.app.services.embedding.e5_provider import E5EmbeddingProvider
    from backend.app.services.embedding.base import InvalidInputError
    
    # 유효한 모델 크기
    config_base = {"model_size": "base"}
    config_large = {"model_size": "large"}
    
    with patch('backend.app.services.embedding.e5_provider.SentenceTransformer'):
        # base 크기는 768 차원
        provider_base = E5EmbeddingProvider(config_base)
        assert provider_base.get_dimension() == 768
        
        # large 크기는 1024 차원
        provider_large = E5EmbeddingProvider(config_large)
        assert provider_large.get_dimension() == 1024
    
    # 유효하지 않은 모델 크기
    with pytest.raises(InvalidInputError):
        E5EmbeddingProvider({"model_size": "xlarge"})

@pytest.mark.asyncio
async def test_embedding_response_shape():
    """임베딩 응답 벡터 차원 테스트"""
    from backend.app.services.embedding.base import EmbeddingResponse
    
    # BGE-M3 차원 (1024)
    vector_bge = np.random.randn(1024).astype(np.float32)
    response_bge = EmbeddingResponse(
        vector=vector_bge,
        dimension=1024,
        tokens_used=0,
        cost=0.0,
        model="BAAI/bge-m3",
        provider="bge"
    )
    assert response_bge.vector.shape == (1024,)
    
    # E5-base 차원 (768)
    vector_e5_base = np.random.randn(768).astype(np.float32)
    response_e5_base = EmbeddingResponse(
        vector=vector_e5_base,
        dimension=768,
        tokens_used=0,
        cost=0.0,
        model="intfloat/e5-base-v2",
        provider="e5"
    )
    assert response_e5_base.vector.shape == (768,)

def test_model_manager_cache_dir():
    """모델 캐시 디렉토리 테스트"""
    from backend.app.services.embedding.model_manager import EmbeddingModelManager
    
    cache_dir = EmbeddingModelManager.get_cache_dir()
    assert isinstance(cache_dir, type(os.path.exists("")).__class__)  # Path 타입

def test_model_manager_device():
    """Device 선택 테스트"""
    from backend.app.services.embedding.model_manager import EmbeddingModelManager
    
    device = EmbeddingModelManager.get_device()
    assert device in ["cuda", "cpu", "mps"]
```

## 5. 산출물

1. **BGE-M3 Provider** (`/backend/app/services/embedding/bge_provider.py`)
   - BAAI/bge-m3 모델 (1024 차원)
   - 로컬 실행, 비용 0, 배치 처리

2. **E5 Provider** (`/backend/app/services/embedding/e5_provider.py`)
   - intfloat/e5-base-v2 (768 차원) 및 e5-large (1024 차원)
   - 환경변수 `E5_MODEL_SIZE`로 선택 가능

3. **모델 관리 유틸리티** (`/backend/app/services/embedding/model_manager.py`)
   - HuggingFace 캐시 관리
   - 모델 사전 로드
   - Device 자동 선택 (CUDA/CPU/MPS)

4. **단위 테스트** (`/backend/tests/unit/services/test_embedding_local.py`)
   - 입력 유효성, 모델 크기 검증
   - 벡터 차원 검증

## 6. 완료 기준

1. BGE-M3 Provider 구현 완료, 1024 차원 정상 생성
2. E5 Provider 구현 완료, base(768d) 및 large(1024d) 정상 생성
3. 환경변수 `BGE_MODEL_PATH`, `E5_MODEL_PATH`, `E5_MODEL_SIZE` 정상 읽음
4. 모델 자동 다운로드 및 HuggingFace 캐싱 정상 작동
5. 배치 처리 (100개 이상 텍스트) 정상 작동
6. 메모리 사용량 합리적 (단일 배치 < 2GB)
7. CPU 환경에서 정상 작동 (CUDA 불필요)
8. 단위 테스트 모두 통과
9. mypy 검사 통과
10. 모든 메서드 docstring 포함
11. 폐쇄망 지원: 인터넷 없이 미리 다운로드한 모델 사용 가능

## 7. 작업 지침

### 지침 7-1. 폐쇄망 환경 지원

환경변수로 오프라인 모드 지원:
```bash
# 모델 사전 다운로드 (인터넷 필요)
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('BAAI/bge-m3')"

# 폐쇄망 환경에서 사용
export BGE_MODEL_PATH=/path/to/local/bge-m3
export SENTENCE_TRANSFORMERS_HOME=/path/to/cache
```

### 지침 7-2. 메모리 최적화

배치 크기 조정:
```python
# 메모리 부족 시
config = {"batch_size": 8}  # 기본값 32에서 감소

# 고속 처리 시
config = {"batch_size": 64}
```

### 지침 7-3. Device 선택

자동 device 선택 (CUDA > CPU):
```python
# 수동 지정 가능
config = {"device": "cpu"}  # CPU 강제 사용
```

### 지침 7-4. 정규화

임베딩 벡터 정규화 (기본 True):
```python
# 유사도 계산 시 cosine distance 사용 권장
config = {"normalize_embeddings": True}
```

### 지침 7-5. 로깅

모델 로드 과정 로깅:
```python
import logging
logger = logging.getLogger(__name__)
# DEBUG 로그: 모델 다운로드 진행률
# INFO 로그: 모델 로드 완료
```
