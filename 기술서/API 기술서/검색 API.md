# 검색 API

Base URL: `/api/v1/search`

---

## 1. 전체 텍스트 검색

PostgreSQL `tsvector` 기반 전문 검색(FTS). 문서 제목, 버전 요약, 노드 내용을 대상으로 검색한다.

```
GET /api/v1/search?q=정보보안&type=document
Authorization: Bearer <token>
```

### 쿼리 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `q` | string | **필수** | 검색 키워드 |
| `type` | string | `all` | `document`, `version`, `node`, `all` |
| `filter[document_type]` | string | - | 문서 유형 필터 |
| `filter[status]` | string | `published` | 문서 상태 필터 |
| `page` | integer | 1 | 페이지 번호 |
| `page_size` | integer | 20 | 페이지 크기 |
| `highlight` | boolean | `true` | 검색어 하이라이팅 포함 여부 |

### 응답

```json
{
  "status": "success",
  "data": {
    "documents": [
      {
        "id": "550e8400-...",
        "title": "정보보안 정책",
        "document_type": "POLICY",
        "status": "published",
        "headline": "...전사 <b>정보보안</b> 기준을 정의합니다...",
        "rank": 0.876,
        "matched_in": ["title", "content"]
      }
    ],
    "versions": [],
    "nodes": [
      {
        "id": "770e8400-...",
        "document_id": "550e8400-...",
        "document_title": "정보보안 정책",
        "node_type": "paragraph",
        "content": "...",
        "headline": "...<b>정보보안</b> 담당자는...",
        "rank": 0.654
      }
    ]
  },
  "meta": {
    "query": "정보보안",
    "total_documents": 5,
    "total_nodes": 23,
    "search_time_ms": 45
  }
}
```

### 검색 가중치 (tsvector weight)

| 필드 | 가중치 | 설명 |
|------|--------|------|
| `documents.title` | A (높음) | 문서 제목 |
| `versions.title_snapshot` | A | 버전 제목 스냅샷 |
| `nodes.title` | A | 노드 제목 |
| `documents.summary` | C (보통) | 문서 요약 |
| `versions.summary_snapshot` | B | 버전 요약 |
| `nodes.content` | B | 노드 내용 |
| `versions.change_summary` | C | 변경 요약 |

---

## 2. 벡터 검색 (Retrieval API)

pgvector HNSW 인덱스를 활용한 의미 기반 유사도 검색. RAG 파이프라인의 첫 단계이기도 하다.

```
POST /api/v1/retrieval/search
Authorization: Bearer <token>
Content-Type: application/json

{
  "query": "비밀번호 정책 기준이 무엇인가요?",
  "document_type": "POLICY",
  "top_k": 10,
  "min_score": 0.7
}
```

### 요청 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `query` | string | **필수** | 검색 질의 |
| `document_type` | string | - | 문서 유형 필터 |
| `top_k` | integer | 20 | 반환할 청크 수 |
| `min_score` | float | 0.0 | 최소 유사도 점수 (0~1) |
| `document_ids` | UUID[] | - | 특정 문서 내 검색 |

### 응답

```json
{
  "status": "success",
  "data": {
    "chunks": [
      {
        "chunk_id": "880e8400-...",
        "document_id": "550e8400-...",
        "document_title": "정보보안 정책",
        "node_path": ["1장 목적", "1.2 비밀번호 정책"],
        "source_text": "비밀번호는 최소 8자 이상이어야 하며...",
        "score": 0.923,
        "document_type": "POLICY"
      }
    ]
  },
  "meta": {
    "query": "비밀번호 정책 기준이 무엇인가요?",
    "total": 10,
    "search_time_ms": 120
  }
}
```

---

## 3. 검색 인덱스 현황 (Admin)

```
GET /api/v1/admin/search/stats
Authorization: Bearer <admin-token>
```

**응답:**
```json
{
  "data": {
    "documents": {
      "total_rows": 142,
      "indexed_rows": 140,
      "unindexed_rows": 2
    },
    "versions": {
      "total_rows": 438,
      "indexed_rows": 438,
      "unindexed_rows": 0
    },
    "nodes": {
      "total_rows": 12540,
      "indexed_rows": 12538,
      "unindexed_rows": 2
    }
  }
}
```

---

## 4. 검색 권한 필터

검색 결과는 요청자의 역할에 따라 자동으로 필터링된다.

| 역할 | 접근 가능한 문서 |
|------|----------------|
| VIEWER | Published 문서 |
| AUTHOR | Published + 본인 Draft |
| REVIEWER ~ SUPER_ADMIN | Published + 본인 관련 Draft |

벡터 검색은 `document_chunks.accessible_roles` / `is_public` 컬럼 기반 필터 적용.

---

## 5. 오류 응답

| 상황 | 코드 | 설명 |
|------|------|------|
| 검색어 없음 | 400 | `q` 파라미터 필수 |
| 임베딩 생성 실패 | 500 | OpenAI API 오류 |
