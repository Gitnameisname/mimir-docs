# 벡터화와 RAG API

---

## 1. 벡터화 (Vectorization)

Base URL: `/api/v1/vectorization`

문서를 청크로 분할하고 임베딩을 생성하여 `document_chunks` 테이블에 저장하는 파이프라인.

### 1.1 문서 벡터화 요청

```
POST /api/v1/vectorization/documents/{document_id}
Authorization: Bearer <admin-token>
Content-Type: application/json

{
  "version_id": "660e8400-...",
  "force": false
}
```

| 파라미터 | 설명 |
|----------|------|
| `version_id` | 특정 버전 지정 (생략 시 현재 Published 버전) |
| `force` | 이미 벡터화된 경우 재처리 여부 |

**응답:**
```json
{
  "data": {
    "job_id": "990e8400-...",
    "status": "PENDING",
    "document_id": "550e8400-...",
    "version_id": "660e8400-..."
  }
}
```

벡터화는 비동기 백그라운드 작업으로 실행된다. `job_id`로 진행 상황 조회.

---

### 1.2 벡터화 상태 조회

```
GET /api/v1/vectorization/jobs/{job_id}
Authorization: Bearer <admin-token>
```

**응답:**
```json
{
  "data": {
    "job_id": "990e8400-...",
    "status": "COMPLETED",
    "progress": 100,
    "chunk_count": 45,
    "token_count": 12800,
    "started_at": "2026-04-10T10:00:00+00:00",
    "ended_at": "2026-04-10T10:00:15+00:00"
  }
}
```

**job status 값:**

| 값 | 의미 |
|----|------|
| `PENDING` | 대기 중 |
| `RUNNING` | 처리 중 |
| `COMPLETED` | 완료 |
| `FAILED` | 실패 (error_message 참조) |

---

### 1.3 벡터화 현황 조회 (Admin)

```
GET /api/v1/admin/vectorization/stats
Authorization: Bearer <admin-token>
```

문서별 벡터화 상태, 최근 작업 목록, 토큰 사용량 통계 반환.

---

### 1.4 청킹 전략

문서 유형별로 청킹 전략이 다르게 적용된다 (플러그인 시스템).

| DocumentType | 청킹 전략 | max_tokens | overlap |
|-------------|-----------|------------|---------|
| POLICY | 조항 단위 | 512 | 50 |
| MANUAL | 절차 단위 | 400 | 30 |
| REPORT | 섹션 단위 | 600 | 100 |
| FAQ | Q&A 쌍 | 256 | 0 |

---

## 2. RAG (Retrieval-Augmented Generation)

Base URL: `/api/v1/rag`

문서 기반 질의응답 API. 벡터 검색으로 관련 청크를 찾고 LLM으로 답변을 생성한다.

### 2.1 RAG 질의 (스트리밍)

```
POST /api/v1/rag/query
Authorization: Bearer <token>
Content-Type: application/json

{
  "query": "비밀번호 정책의 최소 길이 기준은 무엇인가요?",
  "document_type": "POLICY",
  "session_id": "session-001",
  "history": [
    {
      "role": "user",
      "content": "이전 질문"
    },
    {
      "role": "assistant",
      "content": "이전 답변"
    }
  ],
  "stream": true
}
```

**요청 파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `query` | string | **필수** | 질의 텍스트 |
| `document_type` | string | - | 특정 유형 문서만 검색 |
| `document_ids` | UUID[] | - | 특정 문서 내에서만 검색 |
| `session_id` | string | - | 대화 세션 ID (멀티턴) |
| `history` | array | `[]` | 이전 대화 이력 |
| `stream` | boolean | `true` | SSE 스트리밍 여부 |

---

### 2.2 스트리밍 응답 (SSE)

`stream: true` 시 `text/event-stream` 형식으로 응답.

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"type": "token", "content": "비밀번"}

data: {"type": "token", "content": "호"}

data: {"type": "token", "content": " 정책에"}

data: {"type": "citation", "citations": [
  {
    "chunk_id": "880e8400-...",
    "document_id": "550e8400-...",
    "document_title": "정보보안 정책",
    "node_path": ["1장 목적", "1.2 비밀번호 정책"],
    "source_text": "비밀번호는 최소 8자 이상이어야 하며...",
    "score": 0.923
  }
]}

data: {"type": "done", "session_id": "session-001"}
```

**이벤트 타입:**

| type | 설명 |
|------|------|
| `token` | LLM 생성 토큰 (스트리밍) |
| `citation` | 근거 청크 목록 |
| `error` | 오류 발생 |
| `done` | 스트리밍 완료 |

---

### 2.3 비스트리밍 응답

`stream: false` 시 완료된 전체 응답 반환.

```json
{
  "status": "success",
  "data": {
    "answer": "비밀번호 정책에 따르면 최소 8자 이상이어야 하며, 대문자·소문자·숫자·특수문자를 각 1개 이상 포함해야 합니다.",
    "citations": [
      {
        "chunk_id": "880e8400-...",
        "document_title": "정보보안 정책",
        "node_path": ["1장", "1.2 비밀번호"],
        "source_text": "비밀번호는 최소 8자...",
        "score": 0.923
      }
    ],
    "session_id": "session-001",
    "model": "gpt-4o",
    "tokens_used": 450
  }
}
```

---

### 2.4 RAG 파이프라인 내부 흐름

```
1. query 임베딩 생성
        │
        ▼
2. Retriever: document_chunks에서 코사인 유사도 Top-K=20 검색
   (권한 필터: accessible_roles, is_public)
        │
        ▼
3. Reranker: 유사도 점수 기반 정렬, Top-N=5 선택
   (rag_reranker_threshold 이상만 통과)
        │
        ▼
4. ContextBuilder: 청크 + 인용 조합 (최대 6000 tokens)
        │
        ▼
5. LLM 호출 (시스템 프롬프트 + 히스토리 + 컨텍스트 + 질문)
        │
        ▼
6. SSE 스트리밍 or 단일 응답
```

---

### 2.5 RAG 세션 관리

멀티턴 대화는 `session_id`로 관리된다.

```
GET /api/v1/rag/sessions/{session_id}
Authorization: Bearer <token>
```

세션의 대화 이력 및 마지막 인용 청크 반환.

```
DELETE /api/v1/rag/sessions/{session_id}
Authorization: Bearer <token>
```

세션 삭제.

---

### 2.6 LLM 설정

`settings.py`에서 제공자와 모델을 설정한다.

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `LLM_PROVIDER` | `openai` | `openai` 또는 `anthropic` |
| `LLM_MODEL` | `gpt-4o` | 모델명 |
| `RAG_TOP_K` | `20` | Retriever Top-K |
| `RAG_TOP_N` | `5` | Reranker Top-N |
| `RAG_MAX_CONTEXT_TOKENS` | `6000` | 컨텍스트 최대 토큰 |
| `RAG_MAX_HISTORY_TURNS` | `10` | 멀티턴 최대 이력 수 |

Anthropic 사용 시: `LLM_PROVIDER=anthropic`, `LLM_MODEL=claude-sonnet-4-6`
