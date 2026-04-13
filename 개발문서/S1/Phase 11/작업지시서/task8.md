# Task 11-8. RAG Query API 설계 및 구현

## 1. 작업 목적

사용자가 자연어로 질문하면 RAG 파이프라인 전체가 실행되어 응답을 반환하는 API를 설계하고 구현한다.

이 작업의 목표는 다음과 같다.

- POST /api/rag/query 엔드포인트 구현 (단건 + 스트리밍)
- 대화 세션 API 구현 (생성/조회/삭제)
- 권한 검증 포함
- API 응답 스키마 정의

---

## 2. 작업 범위

### 포함 범위

- RAG Query API 구현
- 대화 세션 CRUD API
- 스트리밍(SSE) 응답 구현
- 권한 검증

### 제외 범위

- UI 통합 (Task 11-10에서 다룸)
- 대화 이력 관리 내부 로직 (Task 11-9에서 다룸)

---

## 3. 선행 조건

- Task 11-2~11-7 완료 — 파이프라인 전체 구현
- Phase 3 API 플랫폼 인증/인가 구조 이해

---

## 4. 주요 구현 대상

### 4-1. RAG Query API

#### 단건 질의 (스트리밍)

```
POST /api/rag/query
Authorization: Bearer {token}

Request Body:
{
  "question": "정보보안 정책에서 접근통제 기준을 알려주세요",
  "conversation_id": "uuid (선택, 이어지는 대화)",
  "document_type": "POLICY (선택)",
  "mode": "hybrid (선택, 기본값)",
  "stream": true (선택, 기본 true)
}

Response (stream=true, SSE):
  data: {"type": "chunk", "content": "접근통제는..."}
  data: {"type": "chunk", "content": " 역할 기반으로..."}
  data: {"type": "citations", "citations": [...]}
  data: {"type": "done", "conversation_id": "...", "message_id": "..."}

Response (stream=false):
{
  "answer": "접근통제는 역할 기반으로...",
  "citations": [...],
  "context_chunks": [...],
  "conversation_id": "...",
  "message_id": "...",
  "token_used": 1234
}
```

---

### 4-2. 대화 세션 API

```
POST /api/rag/conversations
Response: {"id": "...", "title": null, "created_at": "..."}

GET /api/rag/conversations?page=1&page_size=20
Response: {"items": [...], "total": int, "page": int}

GET /api/rag/conversations/{id}
Response: {"id": "...", "title": "...", "messages": [...]}

GET /api/rag/conversations/{id}/messages
Response: {"items": [...], "total": int}

DELETE /api/rag/conversations/{id}
Response: 204 No Content
```

---

### 4-3. RAGQueryService 흐름

```python
async def query(self, request: RAGQueryRequest, user: User) -> AsyncIterator[SSEEvent]:
    # 1. 대화 세션 조회 또는 생성
    conversation = await get_or_create_conversation(request.conversation_id, user)

    # 2. 대화 이력 로드 (Multi-turn)
    history = await load_conversation_history(conversation.id)

    # 3. QueryProcessor
    processed = QueryProcessor().process(UserQuery(
        text=request.question,
        conversation_history=history,
        user_roles=user.roles,
        user_org_ids=user.org_ids,
    ))

    # 4. Retriever
    context = RAGContextProvider().retrieve(RAGQuery(
        processed_query=processed,
        document_type=request.document_type,
        mode=request.mode,
    ))

    # 5. Reranker
    reranked = get_reranker().rerank(processed.standalone_text, context.chunks)

    # 6. ContextBuilder
    built = ContextBuilder().build(processed.standalone_text, reranked, ContextConfig())

    # 7. LLM 스트리밍
    full_response = ""
    async for chunk in LLMProvider().stream(built.prompt):
        full_response += chunk
        yield SSEEvent(type="chunk", content=chunk)

    # 8. Citation 연결
    citation_result = CitationLinker().link(full_response, built.chunk_mapping)
    yield SSEEvent(type="citations", citations=citation_result.citations)

    # 9. 메시지 저장
    message = await save_message(conversation.id, request.question, citation_result)
    yield SSEEvent(type="done", conversation_id=str(conversation.id), message_id=str(message.id))
```

---

### 4-4. 권한 검증

- 인증된 사용자만 RAG Query 가능 (Bearer token)
- 자신의 대화 세션만 조회/삭제 가능
- Retriever 단계에서 권한 필터 자동 적용 (별도 체크 불필요)

---

### 4-5. 에러 응답

| 상황 | 응답 |
|------|------|
| 인증 없음 | 401 Unauthorized |
| 타인 대화 접근 | 403 Forbidden |
| 빈 question | 400 Bad Request |
| LLM 오류 | SSE "error" 이벤트 또는 500 |
| 검색 결과 없음 | 응답 생성 ("정보를 찾을 수 없습니다") |

---

## 5. 산출물

1. POST /api/rag/query 구현 (스트리밍 + 비스트리밍)
2. 대화 세션 CRUD API 구현
3. RAGQueryService 구현
4. SSE 이벤트 스키마 정의

---

## 6. 완료 기준

- POST /api/rag/query가 스트리밍 응답을 반환한다
- 대화 세션 API가 동작한다
- 권한 없는 사용자는 401/403을 받는다
- 스트리밍 완료 후 citations 이벤트가 전송된다
- 대화 메시지가 DB에 저장된다

---

## 7. Codex 작업 지침

- 스트리밍은 FastAPI의 StreamingResponse + Server-Sent Events 방식으로 구현한다
- stream=false 요청은 내부적으로 스트리밍을 수행하되 결과를 모아서 반환한다
- 대화 제목은 첫 질문을 기반으로 자동 생성한다 (50자 제한)
- RAG 질의당 토큰 사용량을 LLMUsageLog에 conversation_id와 함께 기록한다
