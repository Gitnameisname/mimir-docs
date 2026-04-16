# Task 0-8. [Critical] /api/v1/system/capabilities 엔드포인트 신설

## 1. 작업 목적

클라이언트(User UI, Admin UI, 외부 AI 에이전트)가 **런타임에 현재 플랫폼의 기능 가용성**을 조회할 수 있도록 하는 **공개 엔드포인트**를 구축한다.

이는:
- 자체 호스팅 환경에서 pgvector 설치 유무 감지
- RAG 검색 가능 여부 판단
- Chunking 활성 여부 확인
- Phase 1에서 LLM provider 목록 동적 확장 지원

S2 원칙 ⑦ "폐쇄망 동등성"의 검증 기반이 된다.

---

## 2. 작업 범위

### 포함 범위

1. **FastAPI 라우터 신설**
   - 파일: `/backend/app/api/v1/system.py` (신규)
   - Endpoint: `GET /api/v1/system/capabilities`
   - 대안: `GET /.well-known/mimir-capabilities` (선택)

2. **응답 스키마 정의**
   ```json
   {
     "success": true,
     "data": {
       "version": "2.0.0",
       "pgvector_enabled": true/false,
       "rag_available": true/false,
       "chunking_enabled": true/false,
       "supported_providers": [],
       "mcp_spec_version": null
     }
   }
   ```

3. **환경변수 기반 동적 응답**
   - `PGVECTOR_ENABLED` 환경변수 확인
   - `RAG_SERVICE_URL` 설정 여부 → rag_available
   - `CHUNKING_ENABLED` 환경변수 확인

4. **응답 캐싱**
   - 5분 캐시 (변경 가능성 낮음)
   - Cache-Control 헤더 설정

5. **인증 불필요**
   - Public endpoint (로그인 없이 접근 가능)
   - CORS 허용

6. **단위 테스트**
   - pgvector on/off 환경별 응답 검증
   - 캐시 동작 검증

### 제외 범위

- 프론트엔드 UI 수정 (사용은 Task 0-5에서)
- 실시간 모니터링 (별도 엔드포인트)
- 기능 설정 변경 (읽기 전용)

---

## 3. 선행 조건

- FastAPI 앱 구조 이해 (라우터, 의존성)
- 환경변수 설정 방식 이해
- Task 0-7 Fresh-boot CI 완료 (테스트 환경)

---

## 4. 주요 작업 항목

### 4-1. 응답 스키마 정의

**파일:** `/backend/app/api/v1/schemas/system.py` (신규)

```python
"""
시스템 정보 및 기능 가용성 스키마
"""

from pydantic import BaseModel, Field
from typing import Optional, List
from enum import Enum


class CapabilityStatus(str, Enum):
    """기능 상태"""
    ENABLED = "enabled"
    DISABLED = "disabled"
    DEGRADED = "degraded"  # 부분 기능만 가능


class CapabilitiesData(BaseModel):
    """시스템 기능 정보"""
    
    # 버전 정보
    version: str = Field(
        ..., 
        description="Mimir 플랫폼 버전 (예: 2.0.0)"
    )
    
    # 주요 기능 가용성
    pgvector_enabled: bool = Field(
        ...,
        description="PostgreSQL pgvector 확장 설치 여부"
    )
    
    rag_available: bool = Field(
        ...,
        description="RAG(검색 보강 생성) 서비스 가용 여부"
    )
    
    chunking_enabled: bool = Field(
        ...,
        description="문서 청킹 활성화 여부"
    )
    
    # Phase 1에서 확장
    supported_providers: List[str] = Field(
        default_factory=list,
        description="활성화된 LLM provider 목록 (예: ['openai', 'anthropic'])"
    )
    
    # MCP 스펙 버전 (Phase 4)
    mcp_spec_version: Optional[str] = Field(
        default=None,
        description="지원하는 MCP 스펙 버전 (예: '2025-11-25')"
    )
    
    # 추가 메타데이터
    deployment_type: str = Field(
        default="standalone",
        description="배포 타입 (standalone, cloud, on-premise)"
    )
    
    closed_network: bool = Field(
        default=False,
        description="폐쇄망 환경 여부 (외부 네트워크 불가)"
    )


class CapabilitiesResponse(BaseModel):
    """capabilities 엔드포인트 응답"""
    success: bool = Field(True, description="요청 성공 여부")
    data: CapabilitiesData = Field(..., description="기능 정보")
```

### 4-2. 라우터 구현

**파일:** `/backend/app/api/v1/system.py` (신규)

```python
"""
시스템 정보 및 기능 가용성 엔드포인트
"""

import os
from datetime import datetime, timedelta
from fastapi import APIRouter, Response
from functools import lru_cache

from app.api.v1.schemas.system import (
    CapabilitiesData,
    CapabilitiesResponse,
)


router = APIRouter(prefix="/system", tags=["system"])

# 캐시 설정
_capabilities_cache: dict = {
    "data": None,
    "timestamp": None,
}
CACHE_TTL = timedelta(minutes=5)


def _is_cache_valid() -> bool:
    """캐시 유효성 확인 (5분)"""
    if _capabilities_cache["timestamp"] is None:
        return False
    return datetime.utcnow() - _capabilities_cache["timestamp"] < CACHE_TTL


def _get_pgvector_status() -> bool:
    """pgvector 활성화 여부 확인"""
    # 방법 1: 환경변수
    pgvector_env = os.getenv("PGVECTOR_ENABLED", "false").lower()
    if pgvector_env in ["true", "1", "yes"]:
        return True
    
    # 방법 2: 데이터베이스 확인 (선택)
    # try:
    #     from sqlalchemy import text
    #     from app.core.database import get_db
    #     db = next(get_db())
    #     result = db.execute(text(
    #         "SELECT EXISTS(SELECT 1 FROM pg_extension WHERE extname='vector')"
    #     ))
    #     return result.scalar() is True
    # except:
    #     return False
    
    return False


def _get_rag_status() -> bool:
    """RAG 서비스 가용 여부"""
    # pgvector가 활성화되어야 RAG 사용 가능
    if not _get_pgvector_status():
        return False
    
    # RAG 서비스 설정 확인
    rag_enabled = os.getenv("RAG_ENABLED", "false").lower() in ["true", "1"]
    return rag_enabled


def _get_chunking_status() -> bool:
    """문서 청킹 활성화 여부"""
    chunking_enabled = os.getenv("CHUNKING_ENABLED", "true").lower() \
                      in ["true", "1"]
    return chunking_enabled


def _get_supported_providers() -> list:
    """활성화된 LLM provider 목록"""
    providers = []
    
    # Phase 1에서 확장: OpenAI, Anthropic, vLLM, Ollama 등
    if os.getenv("OPENAI_API_KEY"):
        providers.append("openai")
    
    # 추가 provider는 나중에...
    
    return providers


def _get_mcp_spec_version() -> str | None:
    """지원하는 MCP 스펙 버전"""
    # Phase 4에서 설정
    # 현재: None
    return None


def _build_capabilities() -> CapabilitiesData:
    """기능 정보 객체 구성"""
    return CapabilitiesData(
        version=os.getenv("APP_VERSION", "2.0.0"),
        pgvector_enabled=_get_pgvector_status(),
        rag_available=_get_rag_status(),
        chunking_enabled=_get_chunking_status(),
        supported_providers=_get_supported_providers(),
        mcp_spec_version=_get_mcp_spec_version(),
        deployment_type=os.getenv("DEPLOYMENT_TYPE", "standalone"),
        closed_network=os.getenv("CLOSED_NETWORK", "false").lower() \
                      in ["true", "1"],
    )


@router.get(
    "/capabilities",
    response_model=CapabilitiesResponse,
    summary="시스템 기능 가용성 조회",
    description="현재 플랫폼의 기능 가용성 및 버전 정보를 조회합니다. "
                "인증 불필요 (public endpoint).",
    tags=["System"],
)
async def get_capabilities(response: Response) -> CapabilitiesResponse:
    """
    시스템 기능 가용성 엔드포인트
    
    응답:
    - pgvector_enabled: pgvector 확장 설치 여부
    - rag_available: RAG 서비스 가용 여부
    - chunking_enabled: 문서 청킹 활성화 여부
    - supported_providers: 활성 LLM provider 목록
    - mcp_spec_version: 지원 MCP 스펙 버전
    
    캐시:
    - 5분 캐시 (Cache-Control: max-age=300)
    """
    
    # 캐시 확인
    if _is_cache_valid() and _capabilities_cache["data"] is not None:
        capabilities = _capabilities_cache["data"]
    else:
        # 캐시 미스: 재계산
        capabilities = _build_capabilities()
        _capabilities_cache["data"] = capabilities
        _capabilities_cache["timestamp"] = datetime.utcnow()
    
    # 캐시 헤더 설정
    response.headers["Cache-Control"] = "public, max-age=300"  # 5분
    response.headers["X-Cache"] = \
        "HIT" if _capabilities_cache["timestamp"] else "MISS"
    
    return CapabilitiesResponse(
        success=True,
        data=capabilities
    )


# ========== .well-known 별칭 (선택) ==========

@router.get(
    "/.well-known/mimir-capabilities",
    response_model=CapabilitiesResponse,
    summary="Well-known capabilities endpoint",
    description="RFC 5785 .well-known 규약 준수",
    include_in_schema=False,  # OpenAPI에서 숨김 (중복)
)
async def get_capabilities_well_known(response: Response) -> CapabilitiesResponse:
    """
    .well-known 규약에 따른 capabilities 조회
    (get_capabilities와 동일한 응답)
    """
    return await get_capabilities(response)
```

### 4-3. 라우터 등록

**파일:** `/backend/app/api/v1/__init__.py` (수정)

```python
"""
FastAPI v1 라우터 집계
"""

from fastapi import APIRouter
from app.api.v1 import (
    auth,
    documents,
    nodes,
    search,
    users,
    # ... 기존 라우터들 ...
    system,  # 신규 추가
)

api_router = APIRouter(prefix="/api/v1")

# 기존 라우터 등록
api_router.include_router(auth.router)
api_router.include_router(documents.router)
api_router.include_router(nodes.router)
api_router.include_router(search.router)
api_router.include_router(users.router)
# ...

# 신규: system 라우터
api_router.include_router(system.router)

# .well-known 라우터는 루트에 등록 (선택)
# app_router = APIRouter()
# app_router.include_router(system.router)  # /.well-known/mimir-capabilities
```

또는 main app.py에서:

```python
# /backend/app/main.py

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.api.v1 import api_router
from app.api.v1 import system

app = FastAPI(
    title="Mimir",
    version="2.0.0",
    description="Knowledge Management Platform for AI",
)

# CORS 설정
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # 또는 명시적 화이트리스트
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# API 라우터
app.include_router(api_router)

# .well-known 라우터
app.include_router(system.router)  # /.well-known/mimir-capabilities
```

### 4-4. 환경변수 설정

**파일:** `.env.example` (수정)

```bash
# 시스템 설정
APP_VERSION=2.0.0
DEPLOYMENT_TYPE=standalone  # standalone, cloud, on-premise
CLOSED_NETWORK=false  # 폐쇄망 환경 여부

# pgvector 및 RAG 설정
PGVECTOR_ENABLED=false  # PostgreSQL pgvector 확장
RAG_ENABLED=false  # RAG 서비스 활성화 (pgvector_enabled=true 필요)
CHUNKING_ENABLED=true  # 문서 청킹 활성화

# LLM Provider (Phase 1 이후)
OPENAI_API_KEY=  # 지정 시 openai provider 활성화
# ANTHROPIC_API_KEY=  # 향후 추가
```

### 4-5. 단위 테스트

**파일:** `/backend/tests/unit/test_system_capabilities.py` (신규)

```python
"""
시스템 capabilities 엔드포인트 테스트
"""

import pytest
from fastapi.testclient import TestClient
from unittest.mock import patch
import os

from app.main import app


client = TestClient(app)


class TestCapabilitiesEndpoint:
    """capabilities 엔드포인트 테스트"""

    def test_endpoint_accessible_without_auth(self):
        """인증 없이 접근 가능"""
        response = client.get("/api/v1/system/capabilities")
        assert response.status_code == 200

    def test_response_schema(self):
        """응답 스키마 검증"""
        response = client.get("/api/v1/system/capabilities")
        assert response.status_code == 200
        data = response.json()
        
        # success 필드
        assert data["success"] is True
        
        # data 필드
        capabilities = data["data"]
        assert "version" in capabilities
        assert "pgvector_enabled" in capabilities
        assert "rag_available" in capabilities
        assert "chunking_enabled" in capabilities
        assert "supported_providers" in capabilities
        assert "mcp_spec_version" in capabilities

    @patch.dict(os.environ, {
        "PGVECTOR_ENABLED": "true",
        "RAG_ENABLED": "true",
    })
    def test_pgvector_enabled(self):
        """pgvector 활성화 상태"""
        response = client.get("/api/v1/system/capabilities")
        data = response.json()
        
        assert data["data"]["pgvector_enabled"] is True
        assert data["data"]["rag_available"] is True

    @patch.dict(os.environ, {
        "PGVECTOR_ENABLED": "false",
        "RAG_ENABLED": "true",
    })
    def test_pgvector_disabled_rag_unavailable(self):
        """pgvector 비활성화 시 RAG 불가"""
        response = client.get("/api/v1/system/capabilities")
        data = response.json()
        
        assert data["data"]["pgvector_enabled"] is False
        # pgvector가 없으면 RAG도 불가능
        assert data["data"]["rag_available"] is False

    @patch.dict(os.environ, {
        "CHUNKING_ENABLED": "false",
    })
    def test_chunking_disabled(self):
        """청킹 비활성화"""
        response = client.get("/api/v1/system/capabilities")
        data = response.json()
        
        assert data["data"]["chunking_enabled"] is False

    @patch.dict(os.environ, {
        "OPENAI_API_KEY": "sk-test",
    })
    def test_supported_providers(self):
        """LLM provider 목록"""
        response = client.get("/api/v1/system/capabilities")
        data = response.json()
        
        providers = data["data"]["supported_providers"]
        assert "openai" in providers

    def test_cache_headers(self):
        """캐시 헤더 설정"""
        response = client.get("/api/v1/system/capabilities")
        
        # Cache-Control 헤더 확인
        assert "Cache-Control" in response.headers
        assert "max-age=300" in response.headers["Cache-Control"]

    def test_well_known_endpoint(self):
        """/.well-known/mimir-capabilities 별칭"""
        response = client.get("/.well-known/mimir-capabilities")
        assert response.status_code == 200
        data = response.json()
        assert data["success"] is True

    def test_cors_headers(self):
        """CORS 헤더 존재"""
        response = client.get(
            "/api/v1/system/capabilities",
            headers={"Origin": "https://example.com"}
        )
        # CORS 설정에 따라 Access-Control-Allow-Origin 헤더 확인 (선택)
        # assert "Access-Control-Allow-Origin" in response.headers
```

### 4-6. 프론트엔드 연동 (참고)

프론트엔드에서 기능 가용성 확인:

```typescript
// /frontend/src/lib/api/system.ts

import { ApiResponse } from './types';

export interface CapabilitiesData {
  version: string;
  pgvector_enabled: boolean;
  rag_available: boolean;
  chunking_enabled: boolean;
  supported_providers: string[];
  mcp_spec_version: string | null;
  deployment_type: string;
  closed_network: boolean;
}

export async function getCapabilities(): 
    Promise<CapabilitiesData> {
  const response = await fetch('/api/v1/system/capabilities');
  const envelope = (await response.json()) as ApiResponse<CapabilitiesData>;
  
  if (!envelope.success) {
    throw new Error(envelope.error?.message || 'Failed to get capabilities');
  }
  
  return envelope.data!;
}
```

사용 예:

```typescript
// /frontend/src/components/SearchBar.tsx

'use client';
import { useEffect, useState } from 'react';
import { getCapabilities } from '@/lib/api/system';

export function SearchBar() {
  const [ragAvailable, setRagAvailable] = useState(false);

  useEffect(() => {
    getCapabilities().then(cap => {
      setRagAvailable(cap.rag_available);
    }).catch(err => {
      console.error('Failed to get capabilities:', err);
    });
  }, []);

  return (
    <div>
      <input type="text" placeholder="검색..." />
      {ragAvailable ? (
        <button>시맨틱 검색</button>
      ) : (
        <button>기본 검색</button>
      )}
    </div>
  );
}
```

---

## 5. 산출물

1. **`/backend/app/api/v1/schemas/system.py`** (신규)
   - CapabilitiesData, CapabilitiesResponse 스키마

2. **`/backend/app/api/v1/system.py`** (신규)
   - GET /api/v1/system/capabilities 엔드포인트
   - /.well-known/mimir-capabilities 별칭

3. **`/backend/app/api/v1/__init__.py`** (수정)
   - system 라우터 등록

4. **`.env.example`** (수정)
   - 환경변수 예시 추가

5. **`/backend/tests/unit/test_system_capabilities.py`** (신규)
   - 단위 테스트 (인증, 스키마, pgvector on/off, 캐시)

6. **API 문서**
   - Swagger/OpenAPI 자동 생성 (FastAPI)

---

## 6. 완료 기준

1. `/api/v1/system/capabilities` 엔드포인트 생성 및 정상 응답
2. 응답 JSON에 모든 필수 필드 포함
3. 인증 없이 접근 가능 (public endpoint)
4. pgvector=true/false 환경별 응답 검증
5. 5분 캐시 (Cache-Control 헤더) 동작 확인
6. 단위 테스트 모두 통과
7. OpenAPI 문서에 자동 등재
8. Task 0-7 Fresh-boot CI에서 capabilities 엔드포인트 호출 가능

---

## 7. 작업 지침

### 지침 7-1. Phase 1 확장 준비

`supported_providers` 필드는 현재 빈 배열이지만, Phase 1에서 다음과 같이 확장:

```python
def _get_supported_providers() -> list:
    providers = []
    
    if os.getenv("OPENAI_API_KEY"):
        providers.append("openai")
    if os.getenv("ANTHROPIC_API_KEY"):
        providers.append("anthropic")
    if os.getenv("VLLM_ENDPOINT"):
        providers.append("vllm")
    if os.getenv("OLLAMA_ENDPOINT"):
        providers.append("ollama")
    
    return providers
```

### 지침 7-2. MCP 스펙 버전 관리

Phase 4에서 MCP 스펙 2025-11-25을 지원할 때:

```python
def _get_mcp_spec_version() -> str | None:
    # Phase 4부터 활성화
    mcp_supported = os.getenv("MCP_ENABLED", "false").lower() in ["true", "1"]
    if mcp_supported:
        return "2025-11-25"
    return None
```

### 지침 7-3. 캐시 무효화 방법

환경변수가 런타임에 변경되는 경우 (드물지만):

```python
# 관리자 API로 캐시 무효화 (선택)
@router.post("/admin/cache/clear", tags=["Admin"])
async def clear_capabilities_cache(request: AdminRequest):
    global _capabilities_cache
    _capabilities_cache["data"] = None
    _capabilities_cache["timestamp"] = None
    return {"success": True}
```

### 지침 7-4. 에러 처리

만약 pgvector 상태 조회 중 DB 연결 오류 발생:

```python
def _get_pgvector_status() -> bool:
    try:
        # DB 확인 로직
        ...
        return True
    except Exception as e:
        # 로그 기록, 안전한 기본값 반환
        logger.warning(f"Failed to check pgvector status: {e}")
        return os.getenv("PGVECTOR_ENABLED", "false").lower() in ["true", "1"]
```

### 지침 7-5. 보안 고려사항

- Public endpoint이므로 민감한 정보 노출 금지 (API 키 등)
- CORS 설정 적절히 (필요시 화이트리스트)
- 레이트 제한 (선택): `slowapi` 또는 유사 라이브러리

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@limiter.limit("100/minute")  # 분당 100회
@router.get("/capabilities")
async def get_capabilities(response: Response):
    ...
```
