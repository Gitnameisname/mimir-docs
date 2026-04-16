# Task 1-5. LLMProviderFactory + 레지스트리 + capabilities 연동

## 1. 작업 목적

Task 1-1~1-4에서 구현한 모든 provider들을 **통합 관리**하는 팩토리 패턴과 레지스트리를 구축한다. 런타임에 `LLM_PROVIDER` 환경변수를 통해 provider를 교체 가능하게 하고, `/api/v1/system/capabilities` 엔드포인트의 `supported_providers` 필드를 동적으로 채운다.

또한 **평가용 LLM(EVALUATION_LLM_PROVIDER)**을 별도로 관리하여 Phase 7 (AI 품질 평가)에서 사용할 수 있도록 한다.

## 2. 작업 범위

### 포함 범위

1. **LLMProviderFactory 클래스**
   - `create(provider_name: str) → LLMProvider`: provider 생성 팩토리
   - 환경변수 검증 및 오류 메시지
   - 싱글톤 캐싱 (동일 provider 반복 생성 시)
   - 지원하는 provider 목록 조회

2. **LLMProviderRegistry 클래스**
   - 런타임 등록된 provider 목록
   - 각 provider의 상태 (active/inactive/error)
   - 헬스 체크 결과 저장
   - `get_status()`, `list_all()` 메서드

3. **기본 Provider 등록**
   - OpenAI, Anthropic, vLLM, Ollama, llama.cpp 자동 등록
   - 환경변수로 활성화 여부 결정

4. **API 엔드포인트**
   - `GET /api/v1/llm/providers` (Admin 권한)
     - 모든 등록된 provider, 상태, 활성화 여부
   - `/api/v1/system/capabilities` 업데이트
     - `supported_providers: [{"name": "openai", "status": "active"}, ...]`

5. **평가용 LLM (EVALUATION_LLM_PROVIDER)**
   - `EVALUATION_LLM_PROVIDER`: 별도 환경변수
   - 평가용 provider 싱글톤 인스턴스
   - Phase 7에서 사용

6. **에러 처리 및 로깅**
   - Provider 초기화 실패 시 명확한 메시지
   - 필수 환경변수 체크리스트 제시
   - 감사 로그에 provider 변경 기록

### 제외 범위

- Provider 성능 모니터링 (Task 외)
- 자동 폴백 로직 (선택적)

## 3. 선행 조건

- Task 1-1~1-4 완료 (모든 provider 구현)
- Phase 0 완료 (`/api/v1/system/capabilities` 엔드포인트 기초)
- FastAPI 기본 라우터 구조

## 4. 주요 작업 항목

### 4-1. LLMProviderFactory 클래스

**파일:** `/backend/app/services/llm/factory.py`

**작업 항목:**
```python
import os
from typing import Optional, Dict, Type, List
from backend.app.services.llm.base import LLMProvider, LLMProviderError, AuthError
from backend.app.services.llm.openai_provider import OpenAILLMProvider
from backend.app.services.llm.anthropic_provider import AnthropicLLMProvider
from backend.app.services.llm.vllm_provider import VLLMProvider
from backend.app.services.llm.ollama_provider import OllamaProvider
from backend.app.services.llm.llama_cpp_provider import LlamaCppProvider

class LLMProviderFactory:
    """LLM Provider 팩토리 (싱글톤 캐싱)"""
    
    # 지원하는 provider 목록
    PROVIDER_MAP: Dict[str, Type[LLMProvider]] = {
        'openai': OpenAILLMProvider,
        'anthropic': AnthropicLLMProvider,
        'vllm': VLLMProvider,
        'ollama': OllamaProvider,
        'llama_cpp': LlamaCppProvider,
    }
    
    # 싱글톤 캐시
    _instances: Dict[str, LLMProvider] = {}
    
    @classmethod
    def create(cls, provider_name: Optional[str] = None) -> LLMProvider:
        """
        Provider 인스턴스 생성 (또는 캐시에서 조회)
        
        Args:
            provider_name: provider 이름 (기본: LLM_PROVIDER 환경변수)
        
        Returns:
            LLMProvider 인스턴스
        
        Raises:
            LLMProviderError: provider 미지원, 초기화 실패 등
        """
        # provider 이름 결정
        if provider_name is None:
            provider_name = os.getenv('LLM_PROVIDER', 'openai')
        
        provider_name = provider_name.lower()
        
        # 캐시에서 조회
        if provider_name in cls._instances:
            return cls._instances[provider_name]
        
        # Provider 클래스 조회
        if provider_name not in cls.PROVIDER_MAP:
            available = ', '.join(cls.PROVIDER_MAP.keys())
            raise LLMProviderError(
                f"Unsupported LLM provider: {provider_name}\n"
                f"Available providers: {available}\n"
                f"Set LLM_PROVIDER environment variable to one of the above."
            )
        
        provider_class = cls.PROVIDER_MAP[provider_name]
        
        # 필수 환경변수 검증
        validation = cls._validate_provider_env(provider_name, provider_class)
        if not validation['is_valid']:
            raise LLMProviderError(
                f"Provider '{provider_name}' initialization failed:\n"
                f"{validation['message']}\n"
                f"Required environment variables:\n"
                f"{validation['required_vars']}"
            )
        
        # Provider 인스턴스 생성
        try:
            instance = provider_class()
            cls._instances[provider_name] = instance
            return instance
        except Exception as e:
            raise LLMProviderError(
                f"Failed to initialize {provider_name} provider: {str(e)}"
            )
    
    @classmethod
    def _validate_provider_env(
        cls,
        provider_name: str,
        provider_class: Type[LLMProvider]
    ) -> Dict[str, any]:
        """
        Provider별 필수 환경변수 검증
        
        Returns:
            {'is_valid': bool, 'message': str, 'required_vars': str}
        """
        required_vars = {
            'openai': ['OPENAI_API_KEY'],
            'anthropic': ['ANTHROPIC_API_KEY'],
            'vllm': ['VLLM_BASE_URL', 'VLLM_MODEL'],
            'ollama': ['OLLAMA_BASE_URL', 'OLLAMA_MODEL'],
            'llama_cpp': ['LLAMA_CPP_BASE_URL', 'LLAMA_CPP_MODEL'],
        }
        
        needed = required_vars.get(provider_name, [])
        missing = [v for v in needed if not os.getenv(v)]
        
        if missing:
            return {
                'is_valid': False,
                'message': f"Missing environment variables: {', '.join(missing)}",
                'required_vars': '\n'.join([f"  - {v}" for v in needed])
            }
        
        # hasattr 체크로 validate_environment 메서드 호출 (있으면)
        if hasattr(provider_class, 'validate_environment'):
            validation = provider_class.validate_environment()
            if not all(validation.values()):
                return {
                    'is_valid': False,
                    'message': f"Provider validation failed",
                    'required_vars': '\n'.join([f"  - {k}" for k, v in validation.items() if not v])
                }
        
        return {
            'is_valid': True,
            'message': '',
            'required_vars': ''
        }
    
    @classmethod
    def list_available_providers(cls) -> List[str]:
        """지원하는 provider 목록"""
        return list(cls.PROVIDER_MAP.keys())
    
    @classmethod
    def get_evaluation_provider(cls) -> LLMProvider:
        """
        평가용 LLM Provider 조회 (싱글톤)
        
        환경변수 EVALUATION_LLM_PROVIDER 사용
        기본값: LLM_PROVIDER와 동일
        """
        eval_provider_name = os.getenv(
            'EVALUATION_LLM_PROVIDER',
            os.getenv('LLM_PROVIDER', 'openai')
        )
        
        cache_key = f'_evaluation_{eval_provider_name}'
        if cache_key in cls._instances:
            return cls._instances[cache_key]
        
        instance = cls.create(eval_provider_name)
        cls._instances[cache_key] = instance
        return instance
    
    @classmethod
    def clear_cache(cls) -> None:
        """캐시 초기화 (테스트용)"""
        cls._instances.clear()
```

### 4-2. LLMProviderRegistry 클래스

**파일:** `/backend/app/services/llm/registry.py`

**작업 항목:**
```python
from typing import Dict, List, Optional
from enum import Enum
import asyncio
from backend.app.services.llm.factory import LLMProviderFactory
from backend.app.services.llm.base import LLMProvider, LLMProviderError

class ProviderStatus(str, Enum):
    """Provider 상태"""
    ACTIVE = "active"
    INACTIVE = "inactive"
    ERROR = "error"

class ProviderInfo:
    """Provider 정보"""
    def __init__(self, name: str, status: ProviderStatus, error: Optional[str] = None):
        self.name = name
        self.status = status
        self.error = error
        self.last_check_at: Optional[float] = None
    
    def to_dict(self) -> Dict:
        return {
            'name': self.name,
            'status': self.status.value,
            'error': self.error,
            'last_check_at': self.last_check_at
        }

class LLMProviderRegistry:
    """LLM Provider 레지스트리 (싱글톤)"""
    
    _instance: Optional['LLMProviderRegistry'] = None
    _providers: Dict[str, ProviderInfo] = {}
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def __init__(self):
        """레지스트리 초기화 및 모든 provider 검사"""
        if not self._providers:
            self._initialize_providers()
    
    def _initialize_providers(self) -> None:
        """지원하는 모든 provider를 레지스트리에 등록"""
        import time
        
        for provider_name in LLMProviderFactory.list_available_providers():
            try:
                # Provider 생성 시도
                provider = LLMProviderFactory.create(provider_name)
                self._providers[provider_name] = ProviderInfo(
                    name=provider_name,
                    status=ProviderStatus.ACTIVE
                )
            except LLMProviderError as e:
                self._providers[provider_name] = ProviderInfo(
                    name=provider_name,
                    status=ProviderStatus.INACTIVE,
                    error=str(e)
                )
            except Exception as e:
                self._providers[provider_name] = ProviderInfo(
                    name=provider_name,
                    status=ProviderStatus.ERROR,
                    error=str(e)
                )
            
            self._providers[provider_name].last_check_at = time.time()
    
    def get_status(self, provider_name: str) -> ProviderInfo:
        """특정 provider 상태 조회"""
        return self._providers.get(provider_name)
    
    def list_all(self) -> List[Dict]:
        """모든 provider 상태 조회"""
        return [info.to_dict() for info in self._providers.values()]
    
    def list_active(self) -> List[str]:
        """활성 provider 목록"""
        return [
            name for name, info in self._providers.items()
            if info.status == ProviderStatus.ACTIVE
        ]
    
    def refresh(self) -> None:
        """레지스트리 새로고침"""
        self._providers.clear()
        self._initialize_providers()
```

### 4-3. API 엔드포인트

**파일:** `/backend/app/api/v1/routers/llm.py`

**작업 항목:**
```python
from fastapi import APIRouter, Depends, HTTPException, status
from typing import List
from backend.app.api.v1.dependencies import check_admin_permission
from backend.app.services.llm.factory import LLMProviderFactory
from backend.app.services.llm.registry import LLMProviderRegistry
from backend.app.schemas.response import ApiResponse

router = APIRouter(prefix="/api/v1/llm", tags=["llm"])

class ProviderStatusResponse:
    """Provider 상태 응답"""
    def __init__(self, data: List[dict]):
        self.data = data

@router.get("/providers", response_model=ApiResponse)
async def list_llm_providers(
    admin_id: str = Depends(check_admin_permission)
) -> dict:
    """
    등록된 모든 LLM Provider 목록 조회
    
    Admin 권한 필수.
    
    Returns:
        {
            "success": true,
            "data": [
                {
                    "name": "openai",
                    "status": "active",
                    "error": null,
                    "last_check_at": 1712345678.123
                },
                ...
            ]
        }
    """
    registry = LLMProviderRegistry()
    
    return ApiResponse(
        success=True,
        data=registry.list_all()
    ).dict()

@router.get("/providers/active", response_model=ApiResponse)
async def list_active_providers(
    admin_id: str = Depends(check_admin_permission)
) -> dict:
    """
    활성 LLM Provider 목록 조회
    
    Returns:
        {
            "success": true,
            "data": ["openai", "anthropic", ...]
        }
    """
    registry = LLMProviderRegistry()
    
    return ApiResponse(
        success=True,
        data=registry.list_active()
    ).dict()

@router.post("/providers/refresh", response_model=ApiResponse)
async def refresh_providers(
    admin_id: str = Depends(check_admin_permission)
) -> dict:
    """
    Provider 레지스트리 새로고침 (헬스 체크)
    
    모든 provider를 다시 검사하고 상태를 갱신한다.
    """
    registry = LLMProviderRegistry()
    registry.refresh()
    
    return ApiResponse(
        success=True,
        data={
            'message': 'Providers refreshed',
            'providers': registry.list_all()
        }
    ).dict()
```

### 4-4. capabilities 엔드포인트 업데이트

**파일:** `/backend/app/api/v1/routers/system.py` (기존 수정)

**작업 항목:**
```python
from fastapi import APIRouter
from backend.app.services.llm.registry import LLMProviderRegistry
from backend.app.schemas.response import ApiResponse

router = APIRouter(prefix="/api/v1/system", tags=["system"])

@router.get("/capabilities")
async def get_capabilities() -> dict:
    """
    시스템 capabilities 조회
    
    Returns:
        {
            "success": true,
            "data": {
                "version": "2.0.0",
                "supported_providers": [
                    {
                        "name": "openai",
                        "status": "active",
                        "models": ["gpt-4o", "gpt-4-turbo", "gpt-3.5-turbo"]
                    },
                    {
                        "name": "anthropic",
                        "status": "active",
                        "models": ["claude-3-opus", "claude-3-sonnet", "claude-3-haiku"]
                    },
                    {
                        "name": "vllm",
                        "status": "inactive",
                        "error": "VLLM_BASE_URL not configured"
                    },
                    ...
                ],
                "features": ["retrieval", "conversation", "agent", ...],
                "api_version": "v1"
            }
        }
    """
    registry = LLMProviderRegistry()
    
    # Provider 정보를 더 상세하게 구성
    supported_providers = []
    for info in registry.list_all():
        provider_data = {
            'name': info['name'],
            'status': info['status']
        }
        
        # 각 provider별 지원 모델 정보
        models_map = {
            'openai': ['gpt-4o', 'gpt-4-turbo', 'gpt-3.5-turbo'],
            'anthropic': ['claude-3-opus', 'claude-3-sonnet', 'claude-3-haiku'],
            'vllm': ['user_defined'],
            'ollama': ['user_defined'],
            'llama_cpp': ['user_defined'],
        }
        
        if info['status'] == 'active':
            provider_data['models'] = models_map.get(info['name'], [])
        else:
            provider_data['error'] = info.get('error')
        
        supported_providers.append(provider_data)
    
    return ApiResponse(
        success=True,
        data={
            'version': '2.0.0',
            'supported_providers': supported_providers,
            'features': [
                'retrieval_rewrite',
                'conversation',
                'agent_interface',
                'prompt_injection_detection',
                'ai_evaluation',
                'structured_extraction'
            ],
            'api_version': 'v1'
        }
    ).dict()
```

### 4-5. 통합 테스트

**파일:** `/backend/tests/integration/test_llm_factory.py`

**작업 항목:**
```python
import pytest
import os
from backend.app.services.llm.factory import LLMProviderFactory
from backend.app.services.llm.registry import LLMProviderRegistry, ProviderStatus
from backend.app.services.llm.base import LLMProviderError

@pytest.fixture
def clear_cache():
    """테스트 후 캐시 초기화"""
    yield
    LLMProviderFactory.clear_cache()

@pytest.mark.asyncio
async def test_factory_create_with_env(clear_cache, monkeypatch):
    """환경변수로 provider 생성"""
    monkeypatch.setenv('LLM_PROVIDER', 'openai')
    monkeypatch.setenv('OPENAI_API_KEY', 'sk-test-key')
    
    provider = LLMProviderFactory.create()
    assert provider is not None
    assert provider.provider_name == 'openai'

@pytest.mark.asyncio
async def test_factory_singleton_caching(clear_cache, monkeypatch):
    """싱글톤 캐싱 검증"""
    monkeypatch.setenv('OPENAI_API_KEY', 'sk-test-key')
    
    provider1 = LLMProviderFactory.create('openai')
    provider2 = LLMProviderFactory.create('openai')
    
    assert provider1 is provider2  # 동일 인스턴스

@pytest.mark.asyncio
async def test_factory_unsupported_provider(clear_cache):
    """지원하지 않는 provider 에러"""
    with pytest.raises(LLMProviderError) as exc_info:
        LLMProviderFactory.create('unsupported_provider')
    
    assert 'Unsupported' in str(exc_info.value)

@pytest.mark.asyncio
async def test_factory_missing_env_vars(clear_cache, monkeypatch):
    """필수 환경변수 누락 시 에러"""
    monkeypatch.delenv('OPENAI_API_KEY', raising=False)
    
    with pytest.raises(LLMProviderError) as exc_info:
        LLMProviderFactory.create('openai')
    
    assert 'OPENAI_API_KEY' in str(exc_info.value)

@pytest.mark.asyncio
async def test_registry_initialize(clear_cache, monkeypatch):
    """레지스트리 초기화"""
    monkeypatch.setenv('OPENAI_API_KEY', 'sk-test')
    
    registry = LLMProviderRegistry()
    providers = registry.list_all()
    
    assert len(providers) > 0
    assert any(p['name'] == 'openai' for p in providers)

@pytest.mark.asyncio
async def test_registry_list_active(clear_cache, monkeypatch):
    """활성 provider 목록"""
    monkeypatch.setenv('OPENAI_API_KEY', 'sk-test')
    monkeypatch.delenv('ANTHROPIC_API_KEY', raising=False)
    
    registry = LLMProviderRegistry()
    active = registry.list_active()
    
    assert 'openai' in active
    assert 'anthropic' not in active

@pytest.mark.asyncio
async def test_evaluation_provider(clear_cache, monkeypatch):
    """평가용 provider 별도 관리"""
    monkeypatch.setenv('OPENAI_API_KEY', 'sk-test-openai')
    monkeypatch.setenv('EVALUATION_LLM_PROVIDER', 'openai')
    
    eval_provider = LLMProviderFactory.get_evaluation_provider()
    assert eval_provider.provider_name == 'openai'
```

## 5. 산출물

1. **LLMProviderFactory 클래스** (`/backend/app/services/llm/factory.py`)
   - 팩토리 패턴, 싱글톤 캐싱
   - 환경변수 검증

2. **LLMProviderRegistry 클래스** (`/backend/app/services/llm/registry.py`)
   - 레지스트리 싱글톤
   - Provider 상태 관리

3. **API 라우터** (`/backend/app/api/v1/routers/llm.py`)
   - `GET /api/v1/llm/providers`
   - `GET /api/v1/llm/providers/active`
   - `POST /api/v1/llm/providers/refresh`

4. **capabilities 엔드포인트 업데이트** (`/backend/app/api/v1/routers/system.py`)
   - `supported_providers` 필드 동적 생성

5. **통합 테스트** (`/backend/tests/integration/test_llm_factory.py`)
   - 팩토리, 레지스트리, 평가 provider 테스트

## 6. 완료 기준

1. LLMProviderFactory.create()가 정상 동작
2. 환경변수 기반 provider 선택 가능
3. 싱글톤 캐싱 동작 (동일 provider 반복 생성 시)
4. 미지원 provider 또는 환경변수 누락 시 명확한 오류 메시지
5. LLMProviderRegistry가 모든 provider 상태 추적
6. `GET /api/v1/llm/providers` API 정상 응답 (Admin 권한)
7. `/api/v1/system/capabilities`의 `supported_providers` 필드 정확
8. EVALUATION_LLM_PROVIDER 별도 관리 (싱글톤)
9. 통합 테스트 모두 통과 (`pytest -v`)
10. 코드 스타일 통일

## 7. 작업 지침

### 지침 7-1. 환경변수 우선순위

1. 명시적 파라미터 (`create('openai')`)
2. LLM_PROVIDER 환경변수
3. 기본값 ('openai')

```python
provider_name = param or os.getenv('LLM_PROVIDER', 'openai')
```

### 지침 7-2. 평가 LLM 이중화

평가 LLM은 메인 LLM과 다를 수 있다:

```bash
export LLM_PROVIDER=vllm                    # 메인: 로컬 모델
export EVALUATION_LLM_PROVIDER=openai       # 평가: 고품질 외부 LLM
```

### 지침 7-3. 에러 메시지 가독성

Provider 초기화 실패 시 다음 정보 제시:

```
Failed to initialize openai provider:
Missing environment variables: OPENAI_API_KEY

Required environment variables:
  - OPENAI_API_KEY
```

### 지침 7-4. 레지스트리 캐싱 전략

레지스트리는 앱 시작 시 한 번만 초기화되도록 하되, Admin이 `/api/v1/llm/providers/refresh`를 호출하여 수동 갱신 가능하게 한다.

### 지침 7-5. API 권한 제어

`GET /api/v1/llm/providers`는 Admin 권한 필수. 일반 사용자는 `GET /api/v1/system/capabilities`로 제한된 정보만 조회 가능.

```python
@router.get("/providers")
async def list_llm_providers(
    admin_id: str = Depends(check_admin_permission)
) -> dict:
    # Admin만 조회 가능
    pass
```
