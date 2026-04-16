# 챗봇 연동 인계 메모 v2
## Citation 계약 반영 (Phase 2 완료)

---

## 1. Citation 5-tuple 계약

모든 검색 응답에 Citation 5-tuple이 포함됩니다.

```json
{
  "citation": {
    "document_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "version_id":  "3fa85f64-5717-4562-b3fc-2c963f66afa7",
    "node_id":     "3fa85f64-5717-4562-b3fc-2c963f66afa8",
    "span_offset": null,
    "content_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
  }
}
```

| 필드 | 설명 |
|------|------|
| `document_id` | 문서 UUID |
| `version_id` | 인용 시점 버전 UUID |
| `node_id` | 문서 내 섹션 UUID (없으면 nil UUID `00000000-...`) |
| `span_offset` | 단락 내 시작 오프셋 (단락 단위 인용 시 `null`) |
| `content_hash` | SHA-256 해시 (내용 변조 감지용) |

---

## 2. Citation 유효성 검증 API

```http
GET /api/v1/citations/{document_id}/versions/{version_id}/nodes/{node_id}/verify?content_hash={hash}
```

응답:
```json
{
  "data": {
    "document_id": "...",
    "version_id": "...",
    "node_id": "...",
    "content_hash": "...",
    "verified": true,
    "modified": false,
    "original_text": "인용된 원문..."
  }
}
```

| 응답 필드 | 의미 |
|-----------|------|
| `verified` | true = 현재 내용과 일치 |
| `modified` | true = 원문이 수정됨 (content_hash 불일치) |

**HTTP 404**: 문서 없음 또는 ACL 위반 (존재 여부 미공개)

---

## 3. 멀티턴 RAG API

### 단발 쿼리 (S1 하위호환)

```http
POST /api/v1/rag/answer
Content-Type: application/json
Authorization: Bearer {token}

{
  "query": "Kubernetes 배포 방법은?",
  "top_k": 10
}
```

응답:
```json
{
  "data": {
    "answer": "Kubernetes 배포는...",
    "citations": [
      {
        "index": 1,
        "citation": { "document_id": "...", "content_hash": "..." },
        "snippet": "인용된 원문 일부..."
      }
    ],
    "rewritten_query": null,
    "context_compressed": false,
    "turn_number": 1
  }
}
```

### 멀티턴 대화

```http
POST /api/v1/rag/answer
Content-Type: application/json
Authorization: Bearer {token}

{
  "query": "이게 뭐야?",
  "top_k": 10,
  "conversation_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
}
```

응답:
```json
{
  "data": {
    "answer": "Kubernetes는...",
    "citations": [...],
    "rewritten_query": "Kubernetes 배포 전략은?",
    "context_compressed": false,
    "turn_number": 2
  }
}
```

**멀티턴 동작 규칙**:
- `conversation_id` 없으면 단발 쿼리 (기존 S1 동작, 하위호환)
- `conversation_id` 있으면 멀티턴 모드 활성화
- `rewritten_query`: 재작성된 쿼리 (원본과 다를 때만 제공, 투명성)
- `turn_number`: 이번 턴 번호 (1부터 시작, 최대 10턴 슬라이딩 윈도우)
- `context_compressed`: 대화 이력 압축 사용 여부 (이력 > 6턴 시 활성화)

**주의**: CitationCache는 인메모리 싱글톤 — 서버 재시작 시 대화 이력 초기화.
Phase 3에서 DB 기반 영속 저장으로 교체 예정.

---

## 4. 검색 전략 선택 API

```http
GET /api/v1/search/documents?q=쿼리&retriever=hybrid&reranker=rule_based
```

| 파라미터 | 허용값 | 기본값 |
|----------|--------|--------|
| `retriever` | `fts`, `vector`, `hybrid` | DocumentType 설정 |
| `reranker` | `cross_encoder`, `rule_based`, `null` | DocumentType 설정 |

`retriever`/`reranker` 없으면 DocumentType의 `retrieval_config` 설정 사용.

---

## 5. DocumentType retrieval_config 설정

관리자가 `PATCH /api/v1/admin/document-types/{type_code}`로 설정:

```json
{
  "retrieval_config": {
    "default_retriever": "hybrid",
    "retriever_params": {
      "fts_weight": 0.4,
      "vector_weight": 0.6,
      "similarity_threshold": 0.3
    },
    "default_reranker": "rule_based",
    "reranker_params": {
      "freshness_bonus": 0.05,
      "pinned_bonus": 0.10
    }
  }
}
```

---

## 6. S1 → S2 마이그레이션 가이드

| 항목 | S1 동작 | S2 추가 |
|------|---------|---------|
| 검색 응답 | 결과 목록 | `citation` 5-tuple 포함 (무시 가능) |
| RAG 응답 | answer + citations | `rewritten_query`, `turn_number` 추가 (무시 가능) |
| 검색 파라미터 | q, type, sort... | `retriever`, `reranker` 추가 (선택) |
| DocumentType 수정 | display_name 등 | `retrieval_config` 추가 (선택) |

**하위호환 보장**: 모든 S2 신규 필드는 Optional — S1 클라이언트는 변경 없이 동작.

---

## 7. 주요 변경 파일 목록

| 파일 | 변경 내용 |
|------|----------|
| `app/schemas/citation.py` | Citation 5-tuple Pydantic 모델 |
| `app/api/v1/citations.py` | Citation 역참조 + 유효성 검증 API |
| `app/services/retrieval/base.py` | Retriever ABC + RetrievalResult |
| `app/services/retrieval/fts_retriever.py` | FTS Retriever 플러그인 |
| `app/services/retrieval/vector_retriever.py` | pgvector Retriever 플러그인 |
| `app/services/retrieval/hybrid_retriever.py` | RRF 하이브리드 Retriever |
| `app/services/retrieval/reranker_*.py` | Reranker 플러그인 (null/rule_based/cross_encoder) |
| `app/schemas/retrieval_config.py` | DocumentType별 Retriever/Reranker 설정 스키마 |
| `app/schemas/conversation.py` | ConversationMessage 스키마 |
| `app/services/prompt/registry.py` | Prompt Registry (시드 파일 로딩) |
| `app/services/retrieval/query_rewriter.py` | 쿼리 재작성 (LLM 기반) |
| `app/services/retrieval/conversation_compressor.py` | 대화 압축 (슬라이딩 윈도우/LLM 요약) |
| `app/services/retrieval/citation_cache.py` | 대화별 Citation 인메모리 캐시 |
| `app/services/multiturn_rag_service.py` | 멀티턴 RAG 서비스 |
| `app/schemas/rag.py` | RAGRequest/RAGResponse S2 스키마 추가 |
| `app/api/v1/rag.py` | POST /rag/answer 엔드포인트 추가 |
| `app/api/v1/search.py` | retriever/reranker 파라미터 추가 |
| `app/api/v1/admin.py` | retrieval_config 수정 지원 |

---

*최종 업데이트: 2026-04-16 (Phase 2 FG2.1~FG2.3 구현 완료)*
