# Task 4-7. 에이전트용 구조화 응답 포맷 + 에러 코드 표준화

## 1. 작업 목적

에이전트가 일관되게 처리할 수 있도록 **모든 에이전트 응답을 표준 envelope 형식으로 구조화**하고, **MCP 호환 오류 코드를 정의·적용**한다. 이를 통해 챗봇 등 외부 에이전트 클라이언트가 Mimir의 응답을 안정적으로 파싱하고 오류를 처리할 수 있도록 한다.

## 2. 작업 범위

### 포함 범위

1. **에이전트용 구조화 응답 포맷 정의**
   - AgentResponse envelope 클래스: success, data, error, metadata 필드
   - metadata: request_id(UUID), timestamp(ISO8601), agent_id, execution_time_ms, trusted(Phase 5용 예약)
   - Pydantic 스키마 정의 (JSON 직렬화 가능)

2. **응답 미들웨어/데코레이터 구현**
   - 모든 에이전트 엔드포인트에 자동으로 envelope 적용
   - FastAPI 미들웨어 또는 데코레이터 패턴 선택
   - 실행 시간(execution_time_ms) 자동 계산

3. **MCP 호환 에러 코드 정의 (Python Enum)**
   - UNAUTHORIZED: 권한 부족
   - NOT_FOUND: 리소스 없음
   - INVALID_SCOPE: Scope 설정 오류
   - INVALID_CITATION: Citation 검증 실패
   - RATE_LIMIT: 속도 제한
   - INTERNAL_ERROR: 서버 오류
   
4. **기존 예외 → MCP 에러 코드 매핑**
   - Python 예외 타입(ValueError, PermissionError, etc.) → 에러 코드 변환 로직
   - HTTP 상태 코드 ↔ MCP 에러 코드 매핑 테이블

5. **Pydantic 스키마 정의**
   - AgentResponseSchema: success, data, error, metadata 구조
   - AgentErrorSchema: code, message 구조
   - AgentMetadataSchema: 메타데이터 필드

6. **단위 테스트**
   - 성공 응답 포맷 검증
   - 에러 응답 포맷 검증
   - 미들웨어 자동 적용 확인
   - 실행 시간 계산 정확성

### 제외 범위

- Tool Schema 정의 (Task 4-8)
- Mimir Extension 선언 (Task 4-8)
- 통합 테스트 (Task 4-9)
- MCP 공식 문서 (Task 4-9)

## 3. 선행 조건

- Phase 1, 2, 3 완료
- FG4.1 (MCP Server 초기화) 완료
- FG4.2 (Agent Principal, Scope Profile) 완료
- FastAPI, Pydantic 설치
- PostgreSQL 및 SQLAlchemy 설정 완료

## 4. 주요 작업 항목

### 4-1. MCP 에러 코드 정의 (Enum)

**파일:** `/backend/app/mcp/error_codes.py`

```python
"""MCP 호환 에러 코드 정의."""

from enum import Enum
from typing import Optional, Dict


class MCPErrorCode(str, Enum):
    """MCP 표준 에러 코드."""
    
    # 인증 및 권한 관련
    UNAUTHORIZED = "UNAUTHORIZED"
    FORBIDDEN = "FORBIDDEN"
    
    # 리소스 관련
    NOT_FOUND = "NOT_FOUND"
    
    # 요청 검증 관련
    INVALID_REQUEST = "INVALID_REQUEST"
    INVALID_SCOPE = "INVALID_SCOPE"
    INVALID_CITATION = "INVALID_CITATION"
    
    # 비즈니스 로직 관련
    RATE_LIMIT = "RATE_LIMIT"
    
    # 서버 관련
    INTERNAL_ERROR = "INTERNAL_ERROR"
    SERVICE_UNAVAILABLE = "SERVICE_UNAVAILABLE"


class ErrorCodeMapper:
    """Python 예외 → MCP 에러 코드 매핑.
    
    기존 애플리케이션 예외를 표준 MCP 에러 코드로 변환한다.
    """
    
    _MAPPING: Dict[type, MCPErrorCode] = {
        PermissionError: MCPErrorCode.UNAUTHORIZED,
        ValueError: MCPErrorCode.INVALID_REQUEST,
        KeyError: MCPErrorCode.NOT_FOUND,
        TimeoutError: MCPErrorCode.SERVICE_UNAVAILABLE,
        RuntimeError: MCPErrorCode.INTERNAL_ERROR,
    }
    
    # 우선순위 높은 커스텀 예외 (앱 specific)
    # 예: DocumentNotFoundError → NOT_FOUND
    
    @classmethod
    def map_exception(cls, exc: Exception) -> MCPErrorCode:
        """예외를 MCP 에러 코드로 매핑.
        
        Args:
            exc: Python 예외 인스턴스
        
        Returns:
            MCPErrorCode
        """
        exc_type = type(exc)
        
        # 정확한 타입 매핑
        if exc_type in cls._MAPPING:
            return cls._MAPPING[exc_type]
        
        # 상속 관계 확인 (부모 클래스 매핑)
        for exc_class, error_code in cls._MAPPING.items():
            if isinstance(exc, exc_class):
                return error_code
        
        # 기본값: INTERNAL_ERROR
        return MCPErrorCode.INTERNAL_ERROR
    
    @classmethod
    def register_mapping(cls, exc_type: type, error_code: MCPErrorCode) -> None:
        """커스텀 예외 매핑 등록.
        
        Args:
            exc_type: 예외 타입
            error_code: 매핑할 MCP 에러 코드
        """
        cls._MAPPING[exc_type] = error_code


# HTTP 상태 코드 ↔ MCP 에러 코드 매핑
HTTP_TO_MCP_ERROR_MAPPING = {
    400: MCPErrorCode.INVALID_REQUEST,
    401: MCPErrorCode.UNAUTHORIZED,
    403: MCPErrorCode.FORBIDDEN,
    404: MCPErrorCode.NOT_FOUND,
    429: MCPErrorCode.RATE_LIMIT,
    500: MCPErrorCode.INTERNAL_ERROR,
    503: MCPErrorCode.SERVICE_UNAVAILABLE,
}

MCP_TO_HTTP_STATUS_MAPPING = {
    MCPErrorCode.UNAUTHORIZED: 401,
    MCPErrorCode.FORBIDDEN: 403,
    MCPErrorCode.NOT_FOUND: 404,
    MCPErrorCode.INVALID_REQUEST: 400,
    MCPErrorCode.INVALID_SCOPE: 400,
    MCPErrorCode.INVALID_CITATION: 400,
    MCPErrorCode.RATE_LIMIT: 429,
    MCPErrorCode.INTERNAL_ERROR: 500,
    MCPErrorCode.SERVICE_UNAVAILABLE: 503,
}
```

### 4-2. Pydantic 응답 스키마

**파일:** `/backend/app/mcp/schemas.py`

```python
"""에이전트용 응답 스키마 (Pydantic)."""

from __future__ import annotations

from typing import Optional, Any, Dict
from uuid import UUID
from datetime import datetime
from pydantic import BaseModel, Field

from app.mcp.error_codes import MCPErrorCode


class AgentErrorSchema(BaseModel):
    """에러 응답 스키마.
    
    MCP 호환 에러 응답 형식.
    """
    code: MCPErrorCode = Field(..., description="에러 코드")
    message: str = Field(..., description="에러 메시지")
    
    class Config:
        use_enum_values = True


class AgentMetadataSchema(BaseModel):
    """응답 메타데이터 스키마.
    
    모든 에이전트 응답에 포함되는 메타데이터.
    """
    request_id: UUID = Field(..., description="요청 고유 ID (UUID)")
    timestamp: datetime = Field(..., description="응답 생성 시간 (ISO8601)")
    agent_id: Optional[UUID] = Field(default=None, description="에이전트 ID")
    execution_time_ms: int = Field(..., description="실행 소요 시간 (밀리초)")
    trusted: bool = Field(default=False, description="신뢰도 (Phase 5에서 활용)")
    
    class Config:
        json_encoders = {
            datetime: lambda v: v.isoformat(),
            UUID: lambda v: str(v),
        }


class AgentResponseSchema(BaseModel):
    """에이전트 응답 포맷 (표준 envelope).
    
    모든 에이전트 API 응답은 이 형식을 따른다.
    """
    success: bool = Field(..., description="성공 여부")
    data: Optional[Dict[str, Any]] = Field(default=None, description="응답 데이터")
    error: Optional[AgentErrorSchema] = Field(default=None, description="에러 정보")
    metadata: AgentMetadataSchema = Field(..., description="응답 메타데이터")
    
    def to_dict(self) -> Dict[str, Any]:
        """Pydantic 모델을 딕셔너리로 변환 (JSON 직렬화 가능)."""
        return self.dict(exclude_none=True, by_alias=True)
    
    @classmethod
    def success_response(
        cls,
        data: Dict[str, Any],
        request_id: UUID,
        agent_id: Optional[UUID] = None,
        execution_time_ms: int = 0,
        trusted: bool = False,
    ) -> AgentResponseSchema:
        """성공 응답 생성.
        
        Args:
            data: 응답 데이터
            request_id: 요청 ID
            agent_id: 에이전트 ID
            execution_time_ms: 실행 시간
            trusted: 신뢰도 플래그
        
        Returns:
            AgentResponseSchema
        """
        return cls(
            success=True,
            data=data,
            error=None,
            metadata=AgentMetadataSchema(
                request_id=request_id,
                timestamp=datetime.utcnow(),
                agent_id=agent_id,
                execution_time_ms=execution_time_ms,
                trusted=trusted,
            )
        )
    
    @classmethod
    def error_response(
        cls,
        error_code: MCPErrorCode,
        message: str,
        request_id: UUID,
        agent_id: Optional[UUID] = None,
        execution_time_ms: int = 0,
    ) -> AgentResponseSchema:
        """에러 응답 생성.
        
        Args:
            error_code: 에러 코드
            message: 에러 메시지
            request_id: 요청 ID
            agent_id: 에이전트 ID
            execution_time_ms: 실행 시간
        
        Returns:
            AgentResponseSchema
        """
        return cls(
            success=False,
            data=None,
            error=AgentErrorSchema(code=error_code, message=message),
            metadata=AgentMetadataSchema(
                request_id=request_id,
                timestamp=datetime.utcnow(),
                agent_id=agent_id,
                execution_time_ms=execution_time_ms,
                trusted=False,
            )
        )
    
    class Config:
        use_enum_values = True
        json_encoders = {
            datetime: lambda v: v.isoformat(),
            UUID: lambda v: str(v),
        }
```

### 4-3. 응답 미들웨어/데코레이터

**파일:** `/backend/app/mcp/middleware.py`

```python
"""에이전트 응답 자동 envelope 적용 미들웨어 및 데코레이터."""

from __future__ import annotations

import time
import logging
from functools import wraps
from typing import Callable, Any, Optional, Dict
from uuid import UUID, uuid4
from starlette.requests import Request
from starlette.responses import JSONResponse, Response

from app.mcp.schemas import AgentResponseSchema
from app.mcp.error_codes import MCPErrorCode, ErrorCodeMapper, MCP_TO_HTTP_STATUS_MAPPING

logger = logging.getLogger(__name__)


class AgentResponseMiddleware:
    """FastAPI 미들웨어: 에이전트 응답을 자동으로 envelope로 감싼다.
    
    모든 /api/agent/* 엔드포인트에 적용되어 표준 응답 형식을 보장한다.
    """
    
    def __init__(self, app):
        self.app = app
    
    async def __call__(self, scope, receive, send) -> None:
        """ASGI 미들웨어 호출."""
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return
        
        path = scope.get("path", "")
        
        # /api/agent/* 경로만 처리
        if not path.startswith("/api/agent/"):
            await self.app(scope, receive, send)
            return
        
        # 요청 시간 기록
        request_start_time = time.time()
        request_id = uuid4()
        
        # 응답 감시를 위한 send 래퍼
        async def send_with_envelope(message: Dict[str, Any]) -> None:
            if message["type"] == "http.response.start":
                # 응답 헤더 처리 (필요시)
                pass
            elif message["type"] == "http.response.body":
                # 응답 바디 처리 (envelope 적용)
                pass
            await send(message)
        
        await self.app(scope, receive, send_with_envelope)


def agent_response(
    func: Callable,
) -> Callable:
    """데코레이터: 함수의 반환값을 AgentResponseSchema으로 감싼다.
    
    @agent_response
    async def tool_search(...) -> Dict[str, Any]:
        return {"results": [...]}
    
    응답:
    {
      "success": true,
      "data": {"results": [...]},
      "error": null,
      "metadata": {...}
    }
    """
    
    @wraps(func)
    async def async_wrapper(*args, **kwargs) -> Response:
        request_id = uuid4()
        agent_id = kwargs.get("agent_id")  # 요청 컨텍스트에서 추출 (구현 필요)
        
        start_time = time.time()
        
        try:
            result = await func(*args, **kwargs)
            
            execution_time_ms = int((time.time() - start_time) * 1000)
            
            # 성공 응답
            response = AgentResponseSchema.success_response(
                data=result if isinstance(result, dict) else {"result": result},
                request_id=request_id,
                agent_id=agent_id,
                execution_time_ms=execution_time_ms,
            )
            
            return JSONResponse(
                content=response.to_dict(),
                status_code=200,
            )
        
        except Exception as exc:
            execution_time_ms = int((time.time() - start_time) * 1000)
            
            # 예외를 MCP 에러 코드로 변환
            error_code = ErrorCodeMapper.map_exception(exc)
            error_message = str(exc) or error_code.value
            
            logger.error(
                f"Agent endpoint error: {func.__name__}",
                exc_info=True,
                extra={
                    "error_code": error_code.value,
                    "request_id": str(request_id),
                }
            )
            
            # 에러 응답
            response = AgentResponseSchema.error_response(
                error_code=error_code,
                message=error_message,
                request_id=request_id,
                agent_id=agent_id,
                execution_time_ms=execution_time_ms,
            )
            
            # HTTP 상태 코드 결정
            http_status = MCP_TO_HTTP_STATUS_MAPPING.get(error_code, 500)
            
            return JSONResponse(
                content=response.to_dict(),
                status_code=http_status,
            )
    
    @wraps(func)
    def sync_wrapper(*args, **kwargs) -> Response:
        # 동기 함수는 이벤트 루프에서 실행해야 하므로 경고
        logger.warning(f"Sync agent endpoint {func.__name__} is not recommended")
        raise RuntimeError(f"Agent endpoint must be async: {func.__name__}")
    
    # async/sync 구분
    import inspect
    if inspect.iscoroutinefunction(func):
        return async_wrapper
    else:
        return sync_wrapper
```

### 4-4. 에러 처리 유틸리티

**파일:** `/backend/app/mcp/error_handling.py`

```python
"""에러 처리 유틸리티."""

from __future__ import annotations

from typing import Optional, Dict, Any
from uuid import UUID
from datetime import datetime
import logging

from app.mcp.schemas import AgentResponseSchema, AgentErrorSchema
from app.mcp.error_codes import MCPErrorCode

logger = logging.getLogger(__name__)


def create_error_response(
    error_code: MCPErrorCode,
    message: str,
    request_id: UUID,
    agent_id: Optional[UUID] = None,
    execution_time_ms: int = 0,
) -> Dict[str, Any]:
    """에러 응답 딕셔너리 생성 (JSON 직렬화 가능).
    
    Args:
        error_code: MCP 에러 코드
        message: 에러 메시지
        request_id: 요청 ID
        agent_id: 에이전트 ID (선택)
        execution_time_ms: 실행 시간
    
    Returns:
        JSON 직렬화 가능한 딕셔너리
    """
    response = AgentResponseSchema.error_response(
        error_code=error_code,
        message=message,
        request_id=request_id,
        agent_id=agent_id,
        execution_time_ms=execution_time_ms,
    )
    return response.to_dict()


def log_error(
    error_code: MCPErrorCode,
    message: str,
    request_id: UUID,
    context: Optional[Dict[str, Any]] = None,
) -> None:
    """표준 형식으로 에러 로깅.
    
    Args:
        error_code: MCP 에러 코드
        message: 에러 메시지
        request_id: 요청 ID
        context: 추가 컨텍스트 (딕셔너리)
    """
    extra = {
        "error_code": error_code.value,
        "request_id": str(request_id),
    }
    if context:
        extra.update(context)
    
    logger.error(message, extra=extra)
```

### 4-5. 단위 테스트

**파일:** `/backend/tests/unit/mcp/test_agent_response.py`

```python
"""에이전트 응답 포맷 및 에러 코드 단위 테스트."""

import pytest
from uuid import uuid4
from datetime import datetime

from app.mcp.schemas import AgentResponseSchema, AgentErrorSchema, AgentMetadataSchema
from app.mcp.error_codes import MCPErrorCode, ErrorCodeMapper, MCP_TO_HTTP_STATUS_MAPPING


class TestAgentErrorSchema:
    """AgentErrorSchema 테스트."""
    
    def test_error_schema_creation(self):
        """에러 스키마 생성."""
        error = AgentErrorSchema(
            code=MCPErrorCode.UNAUTHORIZED,
            message="User is not authorized"
        )
        
        assert error.code == MCPErrorCode.UNAUTHORIZED
        assert error.message == "User is not authorized"
    
    def test_error_schema_to_dict(self):
        """에러 스키마 딕셔너리 변환."""
        error = AgentErrorSchema(
            code=MCPErrorCode.NOT_FOUND,
            message="Document not found"
        )
        
        error_dict = error.dict()
        assert error_dict["code"] == MCPErrorCode.NOT_FOUND.value
        assert error_dict["message"] == "Document not found"


class TestAgentMetadataSchema:
    """AgentMetadataSchema 테스트."""
    
    def test_metadata_creation(self):
        """메타데이터 생성."""
        request_id = uuid4()
        metadata = AgentMetadataSchema(
            request_id=request_id,
            timestamp=datetime.utcnow(),
            agent_id=uuid4(),
            execution_time_ms=123,
            trusted=False,
        )
        
        assert metadata.request_id == request_id
        assert metadata.execution_time_ms == 123
        assert metadata.trusted is False
    
    def test_metadata_to_dict(self):
        """메타데이터 딕셔너리 변환."""
        request_id = uuid4()
        timestamp = datetime.utcnow()
        metadata = AgentMetadataSchema(
            request_id=request_id,
            timestamp=timestamp,
            execution_time_ms=100,
        )
        
        metadata_dict = metadata.dict()
        assert metadata_dict["request_id"] == str(request_id)
        assert metadata_dict["execution_time_ms"] == 100


class TestAgentResponseSchema:
    """AgentResponseSchema 테스트."""
    
    def test_success_response(self):
        """성공 응답 생성."""
        request_id = uuid4()
        response = AgentResponseSchema.success_response(
            data={"results": ["item1", "item2"]},
            request_id=request_id,
            execution_time_ms=200,
        )
        
        assert response.success is True
        assert response.data == {"results": ["item1", "item2"]}
        assert response.error is None
        assert response.metadata.request_id == request_id
        assert response.metadata.execution_time_ms == 200
    
    def test_error_response(self):
        """에러 응답 생성."""
        request_id = uuid4()
        response = AgentResponseSchema.error_response(
            error_code=MCPErrorCode.UNAUTHORIZED,
            message="Authentication failed",
            request_id=request_id,
            execution_time_ms=50,
        )
        
        assert response.success is False
        assert response.data is None
        assert response.error is not None
        assert response.error.code == MCPErrorCode.UNAUTHORIZED
        assert response.error.message == "Authentication failed"
    
    def test_response_to_dict(self):
        """응답 딕셔너리 변환."""
        request_id = uuid4()
        response = AgentResponseSchema.success_response(
            data={"count": 10},
            request_id=request_id,
        )
        
        response_dict = response.to_dict()
        assert response_dict["success"] is True
        assert response_dict["data"] == {"count": 10}
        assert response_dict["metadata"]["request_id"] == str(request_id)


class TestErrorCodeMapper:
    """ErrorCodeMapper 테스트."""
    
    def test_map_permission_error(self):
        """PermissionError 매핑."""
        exc = PermissionError("Access denied")
        error_code = ErrorCodeMapper.map_exception(exc)
        
        assert error_code == MCPErrorCode.UNAUTHORIZED
    
    def test_map_value_error(self):
        """ValueError 매핑."""
        exc = ValueError("Invalid input")
        error_code = ErrorCodeMapper.map_exception(exc)
        
        assert error_code == MCPErrorCode.INVALID_REQUEST
    
    def test_map_key_error(self):
        """KeyError 매핑."""
        exc = KeyError("not_found")
        error_code = ErrorCodeMapper.map_exception(exc)
        
        assert error_code == MCPErrorCode.NOT_FOUND
    
    def test_map_unknown_error(self):
        """알 수 없는 예외는 INTERNAL_ERROR로."""
        exc = Exception("Unknown error")
        error_code = ErrorCodeMapper.map_exception(exc)
        
        assert error_code == MCPErrorCode.INTERNAL_ERROR
    
    def test_custom_mapping_registration(self):
        """커스텀 예외 매핑 등록."""
        class CustomError(Exception):
            pass
        
        ErrorCodeMapper.register_mapping(CustomError, MCPErrorCode.INVALID_SCOPE)
        
        exc = CustomError("Custom error")
        error_code = ErrorCodeMapper.map_exception(exc)
        
        assert error_code == MCPErrorCode.INVALID_SCOPE


class TestHTTPMCPMapping:
    """HTTP 상태 코드 ↔ MCP 에러 코드 매핑 테스트."""
    
    def test_mcp_to_http_status(self):
        """MCP 에러 코드 → HTTP 상태 코드 변환."""
        assert MCP_TO_HTTP_STATUS_MAPPING[MCPErrorCode.UNAUTHORIZED] == 401
        assert MCP_TO_HTTP_STATUS_MAPPING[MCPErrorCode.NOT_FOUND] == 404
        assert MCP_TO_HTTP_STATUS_MAPPING[MCPErrorCode.RATE_LIMIT] == 429
        assert MCP_TO_HTTP_STATUS_MAPPING[MCPErrorCode.INTERNAL_ERROR] == 500


class TestAgentResponseDecorator:
    """@agent_response 데코레이터 테스트."""
    
    @pytest.mark.asyncio
    async def test_decorator_success(self):
        """데코레이터 성공 케이스."""
        from app.mcp.middleware import agent_response
        
        @agent_response
        async def sample_tool():
            return {"data": "test_value"}
        
        response = await sample_tool()
        
        # FastAPI JSONResponse를 확인
        assert response.status_code == 200
        # 응답 바디 확인은 response.body_iterator 또는 json() 사용
    
    @pytest.mark.asyncio
    async def test_decorator_error(self):
        """데코레이터 에러 케이스."""
        from app.mcp.middleware import agent_response
        
        @agent_response
        async def failing_tool():
            raise ValueError("Invalid input")
        
        response = await failing_tool()
        
        assert response.status_code == 400  # INVALID_REQUEST → 400


class TestErrorHandlingUtils:
    """에러 처리 유틸리티 테스트."""
    
    def test_create_error_response(self):
        """에러 응답 생성 유틸리티."""
        from app.mcp.error_handling import create_error_response
        
        request_id = uuid4()
        response_dict = create_error_response(
            error_code=MCPErrorCode.NOT_FOUND,
            message="Resource not found",
            request_id=request_id,
            execution_time_ms=50,
        )
        
        assert response_dict["success"] is False
        assert response_dict["error"]["code"] == MCPErrorCode.NOT_FOUND.value
        assert response_dict["metadata"]["request_id"] == str(request_id)
```

## 5. 산출물

1. **MCP 에러 코드 정의** (`/backend/app/mcp/error_codes.py`)
   - MCPErrorCode Enum (6가지 코드)
   - ErrorCodeMapper 클래스 (예외 → 에러 코드 변환)
   - HTTP ↔ MCP 매핑 테이블

2. **응답 스키마** (`/backend/app/mcp/schemas.py`)
   - AgentResponseSchema: envelope 포맷
   - AgentErrorSchema: 에러 정보
   - AgentMetadataSchema: 메타데이터
   - 팩토리 메서드 (success_response, error_response)

3. **응답 미들웨어/데코레이터** (`/backend/app/mcp/middleware.py`)
   - AgentResponseMiddleware: FastAPI 미들웨어
   - @agent_response 데코레이터: 자동 envelope 적용

4. **에러 처리 유틸리티** (`/backend/app/mcp/error_handling.py`)
   - create_error_response() 헬퍼
   - log_error() 헬퍼

5. **단위 테스트** (`/backend/tests/unit/mcp/test_agent_response.py`)
   - 스키마 생성/변환 테스트
   - 에러 코드 매핑 테스트
   - 데코레이터 기능 테스트
   - 유틸리티 함수 테스트

## 6. 완료 기준

1. AgentResponseSchema가 모든 필드를 포함하는가?
   - success, data, error, metadata (request_id, timestamp, agent_id, execution_time_ms, trusted)

2. 6가지 MCP 에러 코드가 정의되었는가?
   - UNAUTHORIZED, NOT_FOUND, INVALID_SCOPE, INVALID_CITATION, RATE_LIMIT, INTERNAL_ERROR

3. 예외 → MCP 에러 코드 매핑이 정확한가?
   - 주요 Python 예외(PermissionError, ValueError, etc.)가 올바르게 매핑

4. 응답 미들웨어/데코레이터가 구현되었는가?
   - 모든 에이전트 엔드포인트에 자동으로 envelope 적용

5. 실행 시간(execution_time_ms) 계산이 정확한가?
   - 밀리초 단위로 측정

6. HTTP 상태 코드 ↔ MCP 에러 코드 매핑이 완성되었는가?
   - 400, 401, 403, 404, 429, 500, 503 등 표준 상태 코드

7. Pydantic 스키마가 JSON 직렬화 가능한가?
   - UUID, datetime 타입이 정확하게 직렬화

8. 모든 단위 테스트가 통과하는가?

9. mypy 타입 검사를 통과하는가?

## 7. 작업 지침

### 지침 7-1. 응답 포맷 일관성

모든 에이전트 응답(성공/에러)은 AgentResponseSchema envelope을 사용한다. 에이전트 엔드포인트에서 직접 딕셔너리를 반환하면 안 되고, 데코레이터 또는 미들웨어를 통해 자동으로 envelope이 적용되어야 한다.

```python
# 잘못된 사용 (금지)
@router.post("/api/agent/search")
async def search():
    return {"results": [...]}  # 데코레이터 없음

# 올바른 사용
@router.post("/api/agent/search")
@agent_response
async def search():
    return {"results": [...]}  # 자동으로 envelope 적용됨
```

### 지침 7-2. 에러 코드 표준화

기존 코드에서 임의의 오류 메시지를 반환하지 말고, 6가지 정의된 MCP 에러 코드를 사용한다. 새로운 에러 코드를 추가하기 전에 기존 코드로 표현 가능한지 확인한다.

### 지침 7-3. request_id 자동 생성

각 요청에 고유한 UUID인 request_id를 자동으로 생성하여 응답 메타데이터에 포함한다. 이를 통해 에이전트 클라이언트가 요청-응답 쌍을 추적할 수 있다.

### 지침 7-4. 실행 시간 계산

@agent_response 데코레이터에서 함수 실행 시작부터 끝까지의 시간을 측정하여 execution_time_ms로 기록한다. 예외 발생 시에도 해당 지점까지의 시간을 계산한다.

### 지침 7-5. trusted 플래그 예약

metadata.trusted 필드는 Phase 5에서 Citation 검증 결과를 기반으로 설정할 예정이다. 현재는 기본값 false를 사용하고, 문서화해둔다.

### 지침 7-6. S2 원칙 ⑤ 반영: 에이전트 동등 소비자

응답 포맷이 에이전트와 사용자를 동등한 수준의 소비자로 대우한다. 에이전트도 사용자와 동일한 정보를 받을 수 있도록 설계한다.

### 지침 7-7. 에러 로깅

모든 에러는 logger.error()로 기록하되, request_id와 error_code를 포함한 구조화된 로그를 작성한다. 추후 모니터링 및 감사 추적에 활용된다.

### 지침 7-8. 폐쇄망 환경 대응

응답 포맷에는 외부 의존(OpenAI API 호출 결과, SaaS 정보 등)이 포함되지 않는다. 폐쇄망 환경에서도 응답 생성이 가능해야 한다.
