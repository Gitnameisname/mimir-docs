# Task 2-6. DocumentType retrieval_config 통합 및 검색 API 확장

## 1. 작업 목적

DocumentType 설정에 `retrieval_config` 필드를 추가하여, 각 문서 유형별로 다른 Retriever/Reranker를 선택·적용할 수 있게 한다. 기존 `GET /api/v1/search/documents` API에 `retriever`, `reranker` 파라미터를 추가하여 클라이언트가 런타임에 전략을 오버라이드할 수 있게 한다.

## 2. 작업 범위

### 포함 범위

1. **DocumentType `retrieval_config` 스키마 추가**
   - JSON Schema 정의 및 DB 컬럼 확장
   - 기본값 설정 (생략 시 FTS + NullReranker)

2. **`SearchService` 리팩터링**
   - DocumentType 설정에서 Retriever/Reranker를 동적으로 로드
   - `RetrieverFactory`, `RerankerFactory` 통합

3. **검색 API 파라미터 추가**
   - `retriever=fts|vector|hybrid` (기본: DocumentType 설정)
   - `reranker=cross_encoder|rule_based|null` (기본: DocumentType 설정)
   - 파라미터 있을 때: DocumentType 설정 오버라이드
   - 파라미터 없을 때: DocumentType 설정 사용

4. **Admin API: DocumentType retrieval_config 수정**
   - `PATCH /api/v1/admin/document-types/{type_name}` 확장

5. **통합 테스트**
   - DocumentType별 Retriever 자동 선택 테스트
   - API 파라미터 오버라이드 테스트

### 제외 범위

- 새로운 DocumentType 생성 (Admin UI는 기존 기능)
- Reranker 품질 벤치마크 (Phase 7)

## 3. 선행 조건

- Task 2-4 완료 (Retriever 구현체 + Factory)
- Task 2-5 완료 (Reranker 구현체 + Factory)
- S1 Admin `document_types` 테이블 스키마 확인
- 기존 `GET /api/v1/search/documents` API 파라미터 확인

## 4. 주요 작업 항목

### 4-1. retrieval_config JSON Schema

**파일:** `/docs/개발문서/S2/phase2/산출물/retrieval_config_schema.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "RetrievalConfig",
  "description": "DocumentType별 Retriever/Reranker 설정",
  "type": "object",
  "properties": {
    "default_retriever": {
      "type": "string",
      "enum": ["fts", "vector", "hybrid"],
      "default": "fts",
      "description": "기본 검색 전략"
    },
    "retriever_params": {
      "type": "object",
      "properties": {
        "fts_weight": { "type": "number", "minimum": 0, "maximum": 1 },
        "vector_weight": { "type": "number", "minimum": 0, "maximum": 1 },
        "similarity_threshold": { "type": "number", "minimum": 0, "maximum": 1 }
      },
      "additionalProperties": false
    },
    "default_reranker": {
      "type": ["string", "null"],
      "enum": ["cross_encoder", "rule_based", "null", null],
      "default": null
    },
    "reranker_params": {
      "type": "object",
      "properties": {
        "model": { "type": "string" },
        "freshness_bonus": { "type": "number" },
        "pinned_bonus": { "type": "number" }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

### 4-2. DB 마이그레이션

**파일:** `/backend/migrations/versions/xxxx_add_retrieval_config_to_document_types.py`

```python
"""Add retrieval_config to document_types table.

Revision ID: xxxx
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import JSONB


def upgrade():
    op.add_column(
        "document_types",
        sa.Column(
            "retrieval_config",
            JSONB,
            nullable=False,
            server_default='{"default_retriever": "fts", "default_reranker": null}',
        ),
    )


def downgrade():
    op.drop_column("document_types", "retrieval_config")
```

### 4-3. Pydantic 스키마

**파일:** `/backend/app/schemas/document_type.py` (기존 파일 확장)

```python
from typing import Optional, Literal
from pydantic import BaseModel, Field


class RetrieverParams(BaseModel):
    fts_weight: float = Field(0.4, ge=0, le=1)
    vector_weight: float = Field(0.6, ge=0, le=1)
    similarity_threshold: float = Field(0.3, ge=0, le=1)


class RerankerParams(BaseModel):
    model: Optional[str] = None
    freshness_bonus: float = 0.05
    pinned_bonus: float = 0.10


class RetrievalConfig(BaseModel):
    """DocumentType별 Retriever/Reranker 설정."""
    default_retriever: Literal["fts", "vector", "hybrid"] = "fts"
    retriever_params: RetrieverParams = Field(default_factory=RetrieverParams)
    default_reranker: Optional[Literal["cross_encoder", "rule_based", "null"]] = None
    reranker_params: RerankerParams = Field(default_factory=RerankerParams)


# DocumentType 응답 스키마에 추가
class DocumentTypeResponse(BaseModel):
    # ... 기존 필드 유지 ...
    retrieval_config: RetrievalConfig = Field(default_factory=RetrievalConfig)
```

### 4-4. SearchService 리팩터링

**파일:** `/backend/app/services/search_service.py` (기존 파일 확장)

```python
"""SearchService — Retriever/Reranker 플러그인 통합.

기존 FTS 로직 위에 Retriever/Reranker 플러그인 레이어를 추가한다.
하위호환성 유지: 기존 search() 메서드는 그대로 작동.
"""
from __future__ import annotations

import logging
from typing import Any, Dict, List, Optional

import psycopg2.extensions

from app.schemas.document_type import RetrievalConfig
from app.services.retrieval.factory import RetrieverFactory
from app.services.retrieval.reranker_factory import RerankerFactory
from app.services.retrieval.base import RetrievalResult

logger = logging.getLogger(__name__)


class SearchService:
    """Retriever/Reranker 플러그인을 활용한 검색 서비스."""

    def __init__(self, conn: psycopg2.extensions.connection) -> None:
        self._conn = conn

    def _get_retrieval_config(self, document_type: str) -> RetrievalConfig:
        """document_types 테이블에서 retrieval_config를 조회한다."""
        sql = "SELECT retrieval_config FROM document_types WHERE name = %s"
        with self._conn.cursor() as cur:
            cur.execute(sql, (document_type,))
            row = cur.fetchone()
        if row and row[0]:
            return RetrievalConfig.model_validate(row[0])
        return RetrievalConfig()  # 기본값

    async def search_with_plugins(
        self,
        query: str,
        document_type: str,
        top_k: int = 10,
        filters: Optional[Dict[str, Any]] = None,
        retriever_override: Optional[str] = None,
        reranker_override: Optional[str] = None,
    ) -> List[RetrievalResult]:
        """Retriever/Reranker 플러그인을 사용한 검색.

        Args:
            query: 검색 쿼리
            document_type: DocumentType 이름
            top_k: 최종 반환 결과 수
            filters: ACL 포함 필터
            retriever_override: API 파라미터 오버라이드 ("fts"|"vector"|"hybrid")
            reranker_override: API 파라미터 오버라이드 ("cross_encoder"|"rule_based"|"null")

        Returns:
            RetrievalResult 리스트 (Reranker 적용 후 top_k개)
        """
        config = self._get_retrieval_config(document_type)

        # Retriever 결정: API 파라미터 > DocumentType 설정
        retriever_name = retriever_override or config.default_retriever
        reranker_name = reranker_override or config.default_reranker

        retriever = RetrieverFactory.create(
            retriever_name,
            self._conn,
            config.retriever_params.model_dump(),
        )
        reranker = RerankerFactory.create(
            reranker_name,
            config.reranker_params.model_dump(),
        )

        # Retriever: 후보군 넉넉히 가져오기 (Reranker 입력용)
        candidates = await retriever.retrieve(
            query=query,
            document_type=document_type,
            top_k=min(top_k * 5, 100),
            filters=filters,
        )

        # Reranker: 최종 top_k로 압축
        return await reranker.rerank(query, candidates, top_k=top_k)
```

### 4-5. 검색 API 파라미터 확장

**파일:** `/backend/app/api/v1/search.py` (기존 파일 수정)

```python
# 기존 GET /search/documents 엔드포인트에 파라미터 추가
from typing import Optional, Literal
from fastapi import Query as QueryParam

@router.get("/search/documents")
async def search_documents(
    q: str = QueryParam(..., description="검색 쿼리"),
    document_type: Optional[str] = QueryParam(None),
    top_k: int = QueryParam(10, ge=1, le=100),
    # S2 신규 파라미터 (선택)
    retriever: Optional[Literal["fts", "vector", "hybrid"]] = QueryParam(
        None, description="검색 전략 오버라이드 (기본: DocumentType 설정 값)"
    ),
    reranker: Optional[Literal["cross_encoder", "rule_based", "null"]] = QueryParam(
        None, description="재정렬 전략 오버라이드 (기본: DocumentType 설정 값)"
    ),
    actor=Depends(get_current_actor),
    conn=Depends(get_db_conn),
):
    """문서 검색 API.

    S1 하위호환: retriever/reranker 파라미터 없이도 정상 동작.
    S2 확장: retriever/reranker 파라미터로 전략 선택 가능.
    """
    filters = {"actor_id": actor.id, "actor_role": actor.role}
    svc = SearchService(conn)

    if document_type:
        results = await svc.search_with_plugins(
            query=q,
            document_type=document_type,
            top_k=top_k,
            filters=filters,
            retriever_override=retriever,
            reranker_override=reranker,
        )
        # RetrievalResult → 기존 응답 스키마 변환
        return _to_search_response(results)
    else:
        # document_type 없을 때는 기존 S1 로직 그대로
        return await _legacy_search(q, top_k, filters, conn)
```

### 4-6. 통합 테스트

**파일:** `/backend/tests/integration/api/test_search_plugins.py`

```python
"""검색 API Retriever/Reranker 플러그인 통합 테스트."""
import pytest


@pytest.mark.integration
def test_search_with_fts_retriever(test_client, seed_documents, auth_headers):
    """retriever=fts 파라미터 정상 작동."""
    resp = test_client.get(
        "/api/v1/search/documents",
        params={"q": "Kubernetes", "document_type": "technical", "retriever": "fts"},
        headers=auth_headers,
    )
    assert resp.status_code == 200
    data = resp.json()
    # 결과에 citation 포함 여부 확인
    if data.get("results"):
        assert "citation" in data["results"][0]


@pytest.mark.integration
def test_search_with_hybrid_retriever(test_client, seed_documents, auth_headers):
    """retriever=hybrid 파라미터 정상 작동."""
    resp = test_client.get(
        "/api/v1/search/documents",
        params={"q": "배포 전략", "document_type": "technical", "retriever": "hybrid"},
        headers=auth_headers,
    )
    assert resp.status_code == 200


@pytest.mark.integration
def test_document_type_retrieval_config_applied(test_client, seed_documents, auth_headers, set_retrieval_config):
    """DocumentType의 retrieval_config가 기본값으로 적용된다."""
    set_retrieval_config("technical", {"default_retriever": "vector", "default_reranker": None})
    resp = test_client.get(
        "/api/v1/search/documents",
        params={"q": "테스트", "document_type": "technical"},
        headers=auth_headers,
    )
    assert resp.status_code == 200


@pytest.mark.integration
def test_search_s1_compat_without_params(test_client, seed_documents, auth_headers):
    """S1 클라이언트 — retriever/reranker 없이도 정상 검색."""
    resp = test_client.get(
        "/api/v1/search/documents",
        params={"q": "문서", "document_type": "technical"},
        headers=auth_headers,
    )
    assert resp.status_code == 200
```

## 5. 산출물

1. **retrieval_config JSON Schema** (`/docs/개발문서/S2/phase2/산출물/retrieval_config_schema.json`)
2. **DB 마이그레이션** (`/backend/migrations/versions/xxxx_add_retrieval_config.py`)
3. **Pydantic 스키마** (`RetrievalConfig`, `/backend/app/schemas/document_type.py` 확장)
4. **SearchService 리팩터링** (`/backend/app/services/search_service.py`)
5. **검색 API 파라미터 추가** (`/backend/app/api/v1/search.py`)
6. **통합 테스트** (`/backend/tests/integration/api/test_search_plugins.py`)

## 6. 완료 기준

1. `DocumentType.retrieval_config` DB 컬럼 추가 및 기본값 적용
2. `search_with_plugins()` 메서드 정상 작동
3. `retriever=fts|vector|hybrid` 파라미터 작동
4. `reranker=cross_encoder|rule_based|null` 파라미터 작동
5. DocumentType별 기본 설정 자동 적용 확인
6. S1 호환: retriever/reranker 파라미터 없이도 기존 결과 반환
7. 통합 테스트 전체 통과

## 7. 작업 지침

### 지침 7-1. retriever 문자열 하드코딩 금지

API 파라미터와 DocumentType 설정에서 `"fts"`, `"vector"`, `"hybrid"` 문자열을 받더라도, 비즈니스 로직에서 `if retriever == "fts":` 형태로 분기하지 않는다. 분기 로직은 `RetrieverFactory.create()` 내부에만 존재한다.

### 지침 7-2. 마이그레이션 안전성

기존 `document_types` 행에 `retrieval_config` 컬럼을 추가할 때 `server_default`로 최소 유효값을 설정한다. `NOT NULL` 제약으로 신규 행의 필드 누락을 방지한다.

### 지침 7-3. S1 API 하위호환성

기존 `GET /search/documents` 엔드포인트의 기존 파라미터(`q`, `page`, `page_size` 등)는 그대로 유지한다. `retriever`, `reranker` 파라미터는 선택적으로 추가되며, 미제공 시 기존 동작과 동일하다.

### 지침 7-4. Admin Config 보안

`retrieval_config`는 Admin 권한(`SUPER_ADMIN`, `ORG_ADMIN`)만 수정 가능하다. `PATCH /admin/document-types/{type_name}` API에 권한 검증 필수. 잘못된 retriever/reranker 값은 `422 Unprocessable Entity`로 반환.
