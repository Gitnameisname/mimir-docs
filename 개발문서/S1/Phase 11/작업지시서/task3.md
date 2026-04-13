# Task 11-3. Retriever 구현

## 1. 작업 목적

Phase 10의 HybridSearchProvider를 기반으로 RAG용 Retriever를 구현한다.

이 작업의 목표는 다음과 같다.

- RAGContextProvider 구현 (Phase 10 Task 10-11 인터페이스 구현체)
- 권한 필터가 적용된 Top-K 청크 검색
- 검색 결과를 LLM 컨텍스트용 RetrievedChunk로 변환
- 단위/통합 테스트 작성

---

## 2. 작업 범위

### 포함 범위

- RAGContextProvider 구현
- HybridSearchProvider 연계
- RetrievedChunk 변환 로직
- 검색 파라미터 설정 (Top-K, mode)

### 제외 범위

- Reranker (Task 11-4에서 다룸)
- LLM 연동 (Task 11-5에서 다룸)

---

## 3. 선행 조건

- Task 11-1 (아키텍처 설계) 완료
- Task 11-2 (QueryProcessor) 완료 — ProcessedQuery 사용
- Phase 10 Task 10-9 (HybridSearchProvider) 완료

---

## 4. 주요 구현 대상

### 4-1. RAGContextProvider 구현

```
RAGContextProvider
  .retrieve(query: RAGQuery) → RAGContext

RAGQuery
  processed_query: ProcessedQuery
  top_k: int = 20
  mode: str = "hybrid"
  document_type: str | None = None

RAGContext
  chunks: RetrievedChunk[]
  total_retrieved: int
  search_mode: str
```

---

### 4-2. HybridSearchProvider 연계

ProcessedQuery를 SearchQuery로 변환하여 HybridSearchProvider 호출:

```python
def retrieve(self, query: RAGQuery) -> RAGContext:
    search_query = SearchQuery(
        text=query.processed_query.standalone_text,
        embedding=query.processed_query.embedding,
        user_roles=query.processed_query.user_roles,
        user_org_ids=query.processed_query.user_org_ids,
        mode=query.mode,
        document_type=query.document_type,
        limit=query.top_k,
    )
    results = HybridSearchProvider().search(search_query)

    chunks = [RetrievedChunk.from_search_result(r) for r in results]
    return RAGContext(chunks=chunks, total_retrieved=len(chunks), search_mode=query.mode)
```

---

### 4-3. RetrievedChunk 구조

```
RetrievedChunk
  chunk_id: str
  document_id: str
  version_id: str
  node_id: str | None
  document_title: str
  node_path: list[str]
  source_text: str
  context_text: str       (부모 컨텍스트 포함 텍스트 — LLM 입력용)
  similarity_score: float
  fts_score: float | None
  hybrid_score: float
  document_type: str
```

---

### 4-4. 문서 제목 조합

document_chunks에 document_title이 없으므로 document_id로 Document 테이블에서 조회:

```python
def _enrich_with_title(chunks: list[RetrievedChunk]) -> list[RetrievedChunk]:
    doc_ids = {c.document_id for c in chunks}
    titles = DocumentRepository.get_titles(list(doc_ids))
    for chunk in chunks:
        chunk.document_title = titles.get(chunk.document_id, "")
    return chunks
```

---

### 4-5. 통합 테스트

실제 DB(pgvector)와 Mock EmbeddingProvider 사용:

| 시나리오 | 기대 결과 |
|---------|----------|
| 기본 검색 | RetrievedChunk[] 반환 |
| 권한 없는 문서 | 결과에서 제외 |
| top_k=5 | 최대 5개 반환 |
| mode=semantic | 벡터 검색만 사용 |
| 검색 결과 없음 | 빈 RAGContext 반환 |

---

## 5. 산출물

1. RAGContextProvider 구현 코드
2. RAGQuery / RAGContext / RetrievedChunk 데이터 클래스
3. 통합 테스트

---

## 6. 완료 기준

- RAGContextProvider가 ProcessedQuery를 받아 RetrievedChunk[] 를 반환한다
- 권한 필터가 Retriever 단계에서 적용된다
- document_title이 청크에 포함된다
- 통합 테스트가 통과한다

---

## 7. Codex 작업 지침

- 권한 필터는 SQL 레벨에서 처리하며 Python 코드에서 후처리(post-filter)하지 않는다
- top_k 기본값(20)은 설정으로 조정 가능하게 한다
- document_title 조회는 N+1이 발생하지 않도록 배치 조회로 처리한다
