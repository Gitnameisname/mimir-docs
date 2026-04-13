# Task 10-9. 하이브리드 검색 통합 (VectorSearchProvider + HybridSearchProvider)

## 1. 작업 목적

Phase 8에서 예약한 VectorSearchProvider와 HybridSearchProvider를 완성하여 SearchService에 통합한다.

이 작업의 목표는 다음과 같다.

- VectorSearchProvider 구현 (pgvector 기반 시맨틱 검색)
- HybridSearchProvider 구현 (FTS + 벡터 RRF 병합)
- SearchService의 mode 파라미터로 검색 방식 분기
- 권한 필터가 모든 검색 경로에 적용되는 구조 보장

---

## 2. 작업 범위

### 포함 범위

- VectorSearchProvider 구현
- HybridSearchProvider 구현 (RRF 알고리즘)
- SearchService 모드 분기 확장
- 권한 필터 통합
- 통합 테스트

### 제외 범위

- FTSSearchProvider 수정 (Phase 8에서 완성됨, 변경 없음)
- 검색 UI (Phase 8에서 완성됨)
- 성능 튜닝 (Task 10-11에서 다룸)

---

## 3. 선행 조건

- Task 10-6 (벡터화 파이프라인) 완료 — document_chunks에 임베딩 저장됨
- Task 10-4 (EmbeddingProvider) 완료 — 쿼리 텍스트 임베딩 가능
- Phase 8 SearchService 인터페이스 및 FTSSearchProvider 코드 이해

---

## 4. 주요 구현 대상

### 4-1. VectorSearchProvider

pgvector 코사인 유사도 검색:

```
VectorSearchProvider (implements SearchProvider)
  .search(query: SearchQuery) → SearchResult[]
```

#### 쿼리 구조

```python
def search(self, query: SearchQuery) -> list[SearchResult]:
    # 1. 쿼리 텍스트를 임베딩으로 변환
    query_embedding = EmbeddingProvider.embed_single(query.text).embedding

    # 2. pgvector 유사도 검색 (권한 필터 포함)
    rows = db.execute("""
        SELECT
          id, document_id, version_id, node_id,
          source_text, context_text, node_path,
          1 - (embedding <=> :query_vector) AS similarity
        FROM document_chunks
        WHERE is_current = true
          AND (
            accessible_roles && :user_roles
            OR is_public = true
          )
          AND document_status = 'PUBLISHED'
        ORDER BY embedding <=> :query_vector
        LIMIT :k
    """, {
        "query_vector": query_embedding,
        "user_roles": query.user_roles,
        "k": query.limit or 20,
    })

    return [SearchResult.from_chunk_row(row) for row in rows]
```

---

### 4-2. HybridSearchProvider (RRF)

FTS 결과와 벡터 검색 결과를 Reciprocal Rank Fusion으로 병합:

```
HybridSearchProvider (implements SearchProvider)
  .search(query: SearchQuery) → SearchResult[]
  ._rrf_merge(fts_results, vector_results, k=60) → SearchResult[]
```

#### RRF 알고리즘

```python
def _rrf_merge(self, fts_results, vector_results, k=60):
    scores = {}

    for rank, result in enumerate(fts_results):
        doc_id = result.chunk_id
        scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank + 1)

    for rank, result in enumerate(vector_results):
        doc_id = result.chunk_id
        scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank + 1)

    # 점수 기준 내림차순 정렬
    merged = sorted(scores.keys(), key=lambda x: scores[x], reverse=True)

    # 원본 결과에서 메타데이터 조합
    all_results = {r.chunk_id: r for r in fts_results + vector_results}
    return [all_results[cid] for cid in merged if cid in all_results]
```

---

### 4-3. SearchService 모드 분기

Phase 8의 SearchService 인터페이스는 변경 없이 mode 파라미터로 분기:

```python
class SearchService:
    def search(self, query: SearchQuery) -> list[SearchResult]:
        mode = query.mode or settings.DEFAULT_SEARCH_MODE  # "hybrid" | "semantic" | "keyword"

        if mode == "keyword":
            return FTSSearchProvider().search(query)
        elif mode == "semantic":
            return VectorSearchProvider().search(query)
        elif mode == "hybrid":
            return HybridSearchProvider().search(query)
        else:
            raise ValueError(f"Unknown search mode: {mode}")
```

#### 기본 모드 설정

```
DEFAULT_SEARCH_MODE = "hybrid"  (Phase 10 이후 기본값)
```

Phase 10 이전에는 "keyword" 가 기본값이었으며, Phase 10 완료 후 "hybrid"로 전환.

---

### 4-4. SearchQuery 확장

```python
@dataclass
class SearchQuery:
    text: str
    user_roles: list[str]
    user_org_ids: list[str]
    mode: str = "hybrid"       # 신규 추가
    document_type: str = None
    limit: int = 20
    offset: int = 0
```

---

### 4-5. 권한 필터 통합 검증

모든 검색 경로에서 권한 필터 적용 여부 테스트:

| 검색 모드 | 권한 필터 적용 |
|----------|-------------|
| keyword (FTS) | WHERE accessible_roles && :user_roles |
| semantic (벡터) | WHERE accessible_roles && :user_roles |
| hybrid (RRF) | FTS + 벡터 각각 적용 후 병합 |

---

### 4-6. 통합 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| semantic 모드 검색 | 유사도 순 결과 반환 |
| hybrid 모드 검색 | FTS + 벡터 병합 결과 반환 |
| 권한 없는 문서 | 검색 결과에서 제외 |
| 공개 문서 | 권한 없어도 검색 결과에 포함 |
| mode=keyword | FTSSearchProvider 사용 (Phase 8 동작 유지) |

---

## 5. 산출물

1. VectorSearchProvider 구현 코드
2. HybridSearchProvider 구현 코드 (RRF 포함)
3. SearchService 모드 분기 확장
4. SearchQuery 모드 필드 추가
5. 통합 테스트

---

## 6. 완료 기준

- VectorSearchProvider가 pgvector 코사인 유사도 검색을 수행한다
- HybridSearchProvider가 FTS + 벡터 결과를 RRF로 병합한다
- SearchService의 mode 파라미터로 검색 방식을 분기할 수 있다
- 모든 검색 경로에서 권한 필터가 적용된다
- Phase 8 SearchService 인터페이스가 변경 없이 유지된다

---

## 7. Codex 작업 지침

- Phase 8 SearchService API 계약을 변경하지 않는다 — 기존 클라이언트 코드가 수정 없이 동작해야 한다
- RRF의 k 파라미터(기본 60)는 설정으로 조정 가능하게 한다
- 권한 필터는 VectorSearchProvider 내부 SQL에서 직접 처리하여 검색 후 필터링(post-filter)을 피한다
- 하이브리드 검색에서 FTS 또는 벡터 검색 중 하나가 실패하면 나머지 결과만으로 응답한다 (부분 실패 허용)
