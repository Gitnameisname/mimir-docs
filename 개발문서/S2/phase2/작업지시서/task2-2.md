# Task 2-2. Citation 역참조 API

## 1. 작업 목적

챗봇·에이전트·사용자가 Citation 5-tuple을 받았을 때 "이 인용이 여전히 유효한가?" 를 직접 검증하고 원문을 조회할 수 있는 **2개의 역참조 API 엔드포인트**를 구현한다. Phase 4 Agent Interface가 MCP Tool을 통해 이 API를 호출하므로, 스키마 계약을 명확히 정의하는 것이 핵심이다.

## 2. 작업 범위

### 포함 범위

1. **`GET /api/v1/citations/{document_id}/versions/{version_id}/nodes/{node_id}/verify`**
   - 요청 파라미터: `span_offset` (optional), `content_hash` (required)
   - 응답: `{verified: bool, original_text: str, modified: bool, citation: Citation}`
   - 용도: 챗봇이 "이 citation이 여전히 유효한가?" 검증

2. **`GET /api/v1/citations/{document_id}/versions/{version_id}/nodes/{node_id}/content`**
   - 요청 파라미터: (없음)
   - 응답: `{content: str, metadata: {...}, citation: Citation}`
   - 용도: 사용자/에이전트가 citation 클릭 시 원문 보기

3. **ACL 적용**
   - 요청자가 해당 document에 대한 read 권한을 가진 경우에만 응답 반환
   - 권한 없음: 404 반환 (존재 여부 노출 방지)

4. **DB 조회 로직**
   - `document_chunks` 테이블에서 `document_id + version_id + node_id` 조합으로 청크 조회
   - `content_hash` 재계산 후 요청값과 비교

5. **통합 테스트**
   - 유효한 citation verify 성공 케이스
   - 문서 수정 후 `modified: true` 반환 케이스
   - 존재하지 않는 citation 404 케이스
   - ACL 위반 404 케이스

### 제외 범위

- Citation 모델 정의 (Task 2-1)
- Citation 일괄 검증 배치 API
- Citation 이력 조회

## 3. 선행 조건

- Task 2-1 완료 (`Citation` 모델, `CitationVerifier`)
- S1 `document_chunks` 테이블 스키마 확인
- ACL 미들웨어 (`/backend/app/api/auth/`) 분석 완료
- FastAPI 라우터 구조 이해 (`/backend/app/api/v1/`)

## 4. 주요 작업 항목

### 4-1. 요청/응답 스키마

**파일:** `/backend/app/schemas/citation.py` (Task 2-1 파일 확장)

```python
class CitationVerifyRequest(BaseModel):
    """Citation 검증 요청 파라미터."""
    content_hash: str = Field(..., min_length=64, max_length=64, description="SHA-256 hex digest")
    span_offset: Optional[int] = Field(None, ge=0)


class CitationVerifyResponse(BaseModel):
    """Citation 검증 응답."""
    verified: bool
    original_text: Optional[str] = None   # 원문 (권한 있을 때만 포함)
    modified: bool = False                 # 내용 변경 여부
    citation: Citation                     # 요청받은 Citation 그대로 echo


class CitationContentResponse(BaseModel):
    """Citation 원문 조회 응답."""
    content: str
    metadata: dict
    citation: Citation
```

### 4-2. Citation DB 조회 서비스

**파일:** `/backend/app/services/retrieval/citation_service.py`

```python
"""Citation 역참조 서비스.

document_chunks 테이블에서 Citation 5-tuple에 해당하는 청크를 조회하고,
content_hash 일치 여부를 검증한다.
"""

from __future__ import annotations

import hashlib
import logging
from typing import Optional
from uuid import UUID

import psycopg2.extensions

from app.schemas.citation import (
    Citation,
    CitationContentResponse,
    CitationVerifyResponse,
)

logger = logging.getLogger(__name__)


class CitationService:
    """Citation 역참조 및 검증 로직."""

    def __init__(self, conn: psycopg2.extensions.connection) -> None:
        self._conn = conn

    def verify(
        self,
        document_id: UUID,
        version_id: UUID,
        node_id: UUID,
        content_hash: str,
        span_offset: Optional[int] = None,
    ) -> CitationVerifyResponse:
        """Citation의 content_hash가 현재 DB 내용과 일치하는지 검증한다.

        Args:
            document_id: 문서 UUID
            version_id: 버전 UUID
            node_id: 노드 UUID
            content_hash: 클라이언트가 저장해둔 SHA-256
            span_offset: 문자 오프셋 (미사용, 향후 확장)

        Returns:
            CitationVerifyResponse (verified, modified, original_text)
        """
        chunk = self._fetch_chunk(document_id, version_id, node_id)
        if chunk is None:
            return CitationVerifyResponse(
                verified=False,
                modified=False,
                citation=Citation(
                    document_id=document_id,
                    version_id=version_id,
                    node_id=node_id,
                    span_offset=span_offset,
                    content_hash=content_hash,
                ),
            )

        current_hash = hashlib.sha256(chunk["source_text"].encode("utf-8")).hexdigest()
        verified = current_hash == content_hash
        citation = Citation(
            document_id=document_id,
            version_id=version_id,
            node_id=node_id,
            span_offset=span_offset,
            content_hash=content_hash,
        )
        return CitationVerifyResponse(
            verified=verified,
            original_text=chunk["source_text"],
            modified=not verified,
            citation=citation,
        )

    def get_content(
        self,
        document_id: UUID,
        version_id: UUID,
        node_id: UUID,
    ) -> Optional[CitationContentResponse]:
        """Citation에 해당하는 청크 원문과 메타데이터를 반환한다."""
        chunk = self._fetch_chunk(document_id, version_id, node_id)
        if chunk is None:
            return None

        current_hash = hashlib.sha256(chunk["source_text"].encode("utf-8")).hexdigest()
        citation = Citation(
            document_id=document_id,
            version_id=version_id,
            node_id=node_id,
            content_hash=current_hash,
        )
        return CitationContentResponse(
            content=chunk["source_text"],
            metadata=chunk.get("metadata") or {},
            citation=citation,
        )

    def _fetch_chunk(
        self,
        document_id: UUID,
        version_id: UUID,
        node_id: UUID,
    ) -> Optional[dict]:
        """document_chunks 테이블에서 청크를 조회한다."""
        sql = """
            SELECT dc.source_text, dc.metadata
            FROM document_chunks dc
            JOIN document_versions dv ON dc.version_id = dv.id
            WHERE dv.document_id = %s
              AND dc.version_id  = %s
              AND dc.node_id     = %s
            LIMIT 1
        """
        with self._conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
            cur.execute(sql, (str(document_id), str(version_id), str(node_id)))
            row = cur.fetchone()
        return dict(row) if row else None
```

### 4-3. FastAPI 라우터

**파일:** `/backend/app/api/v1/citations.py`

```python
"""Citation 역참조 API 라우터.

엔드포인트:
  GET /citations/{document_id}/versions/{version_id}/nodes/{node_id}/verify
  GET /citations/{document_id}/versions/{version_id}/nodes/{node_id}/content
"""

from __future__ import annotations

from typing import Optional
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, Query
from fastapi.responses import JSONResponse

from app.api.auth import get_current_actor   # S1 기존 인증 미들웨어
from app.api.responses import ok_response
from app.db import get_db_conn
from app.schemas.citation import CitationContentResponse, CitationVerifyResponse
from app.services.retrieval.citation_service import CitationService

router = APIRouter(prefix="/citations", tags=["citations"])


def _get_citation_service(conn=Depends(get_db_conn)) -> CitationService:
    return CitationService(conn)


def _check_document_access(document_id: UUID, actor, conn) -> None:
    """문서에 대한 read 권한 확인. 권한 없으면 404 반환 (존재 여부 노출 방지)."""
    # ACL 필터링은 기존 documents 서비스의 check_access() 활용
    # 권한 없음 또는 문서 없음 → 동일하게 404
    from app.services.documents_service import check_document_access
    if not check_document_access(conn, document_id, actor):
        raise HTTPException(status_code=404, detail="Citation not found")


@router.get(
    "/{document_id}/versions/{version_id}/nodes/{node_id}/verify",
    response_model=CitationVerifyResponse,
    summary="Citation 유효성 검증",
    description="content_hash를 재계산하여 현재 DB 내용과 일치 여부를 반환한다.",
)
async def verify_citation(
    document_id: UUID,
    version_id: UUID,
    node_id: UUID,
    content_hash: str = Query(..., min_length=64, max_length=64),
    span_offset: Optional[int] = Query(None, ge=0),
    actor=Depends(get_current_actor),
    svc: CitationService = Depends(_get_citation_service),
    conn=Depends(get_db_conn),
):
    _check_document_access(document_id, actor, conn)
    return svc.verify(document_id, version_id, node_id, content_hash, span_offset)


@router.get(
    "/{document_id}/versions/{version_id}/nodes/{node_id}/content",
    response_model=CitationContentResponse,
    summary="Citation 원문 조회",
    description="Citation 좌표에 해당하는 청크 원문과 메타데이터를 반환한다.",
)
async def get_citation_content(
    document_id: UUID,
    version_id: UUID,
    node_id: UUID,
    actor=Depends(get_current_actor),
    svc: CitationService = Depends(_get_citation_service),
    conn=Depends(get_db_conn),
):
    _check_document_access(document_id, actor, conn)
    result = svc.get_content(document_id, version_id, node_id)
    if result is None:
        raise HTTPException(status_code=404, detail="Citation not found")
    return result
```

**`/backend/app/api/v1/router.py`에 라우터 등록:**

```python
from app.api.v1.citations import router as citations_router
api_router.include_router(citations_router)
```

### 4-4. 통합 테스트

**파일:** `/backend/tests/integration/api/test_citations_api.py`

```python
"""Citation 역참조 API 통합 테스트.

실제 DB와 연결하여 검증한다.
"""
import pytest
import hashlib
from uuid import uuid4


@pytest.mark.integration
def test_verify_citation_valid(test_client, seed_chunk):
    """유효한 citation의 verify → verified=True."""
    chunk = seed_chunk
    content_hash = hashlib.sha256(chunk["source_text"].encode()).hexdigest()
    resp = test_client.get(
        f"/api/v1/citations/{chunk['document_id']}/versions/{chunk['version_id']}"
        f"/nodes/{chunk['node_id']}/verify",
        params={"content_hash": content_hash},
        headers={"Authorization": f"Bearer {chunk['owner_token']}"},
    )
    assert resp.status_code == 200
    data = resp.json()
    assert data["verified"] is True
    assert data["modified"] is False


@pytest.mark.integration
def test_verify_citation_modified(test_client, seed_chunk, modify_chunk):
    """문서 수정 후 verify → modified=True."""
    chunk = seed_chunk
    old_hash = hashlib.sha256(chunk["source_text"].encode()).hexdigest()
    modify_chunk(chunk["node_id"], "수정된 내용")
    resp = test_client.get(
        f"/api/v1/citations/{chunk['document_id']}/versions/{chunk['version_id']}"
        f"/nodes/{chunk['node_id']}/verify",
        params={"content_hash": old_hash},
        headers={"Authorization": f"Bearer {chunk['owner_token']}"},
    )
    assert resp.status_code == 200
    data = resp.json()
    assert data["verified"] is False
    assert data["modified"] is True


@pytest.mark.integration
def test_verify_citation_acl_forbidden(test_client, seed_chunk, other_user_token):
    """다른 사용자의 문서 verify → 404."""
    chunk = seed_chunk
    content_hash = hashlib.sha256(chunk["source_text"].encode()).hexdigest()
    resp = test_client.get(
        f"/api/v1/citations/{chunk['document_id']}/versions/{chunk['version_id']}"
        f"/nodes/{chunk['node_id']}/verify",
        params={"content_hash": content_hash},
        headers={"Authorization": f"Bearer {other_user_token}"},
    )
    assert resp.status_code == 404


@pytest.mark.integration
def test_get_citation_content(test_client, seed_chunk):
    """유효한 citation content 조회."""
    chunk = seed_chunk
    resp = test_client.get(
        f"/api/v1/citations/{chunk['document_id']}/versions/{chunk['version_id']}"
        f"/nodes/{chunk['node_id']}/content",
        headers={"Authorization": f"Bearer {chunk['owner_token']}"},
    )
    assert resp.status_code == 200
    data = resp.json()
    assert data["content"] == chunk["source_text"]
    assert "citation" in data
```

## 5. 산출물

1. **스키마 확장** (`/backend/app/schemas/citation.py`)
   - `CitationVerifyRequest`, `CitationVerifyResponse`, `CitationContentResponse`
2. **CitationService** (`/backend/app/services/retrieval/citation_service.py`)
   - `verify()`, `get_content()`, `_fetch_chunk()`
3. **FastAPI 라우터** (`/backend/app/api/v1/citations.py`)
   - `/verify`, `/content` 2개 엔드포인트
4. **라우터 등록** (`/backend/app/api/v1/router.py` 수정)
5. **통합 테스트** (`/backend/tests/integration/api/test_citations_api.py`)

## 6. 완료 기준

1. `GET /verify` 엔드포인트 정상 작동 (200, verified=true/false)
2. 문서 수정 후 `modified: true` 반환
3. 존재하지 않는 citation → 404
4. ACL 위반 → 404 (존재 여부 노출 방지)
5. `GET /content` 엔드포인트 정상 작동 (원문, 메타데이터 반환)
6. 통합 테스트 전체 통과
7. OpenAPI 문서(`/docs`)에서 두 엔드포인트 표시

## 7. 작업 지침

### 지침 7-1. ACL 404 처리

권한 없음과 문서 없음 모두 동일하게 **404**를 반환한다. 403을 반환하면 공격자가 문서의 존재 여부를 알 수 있으므로, 모든 접근 불가 케이스는 404로 통일한다.

### 지침 7-2. node_id → chunk 조회 쿼리

`document_chunks` 테이블에서 `(version_id, node_id)` 조합이 유일하지 않을 수 있다 (동일 노드에 여러 청크). 현재는 `LIMIT 1`로 처리하되, 향후 `span_offset` 로 특정 청크를 선택할 수 있도록 쿼리 구조를 유연하게 설계한다.

### 지침 7-3. 감사 로그

Citation 조회 이벤트는 감사 로그에 기록한다. `actor_type` 필드 포함 (`"user"` 또는 `"agent"`). S2 원칙: AI 에이전트가 Citation API를 호출하는 경우 `actor_type="agent"` 로 기록.

### 지침 7-4. content 엔드포인트의 content_hash

`/content` 응답의 `citation.content_hash`는 **현재 DB 내용**의 해시를 반환한다 (클라이언트가 요청 시 제공하지 않았으므로). 즉, 항상 최신 해시를 반환하여 클라이언트가 새로운 Citation을 갱신할 수 있게 한다.
