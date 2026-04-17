# Mimir MCP Server 통합 가이드

> **산출물 분류**: DOCS-001 이월 처리 (Phase 4 전체검수보고서 §7)  
> **작성일**: 2026-04-17  
> **대상 독자**: 외부 에이전트 개발자 (LangChain, n8n, 직접 MCP 클라이언트 등)

---

## 1. 개요

Mimir MCP Server는 Model Context Protocol(MCP)을 구현한 에이전트 통합 엔드포인트입니다.  
외부 AI 에이전트가 Mimir의 문서, 검색, 대화, 제안 기능을 도구(Tool)로 사용할 수 있습니다.

### 기본 엔드포인트

| 항목 | 값 |
|------|-----|
| Base URL | `https://{HOST}/api/v1/mcp` |
| 프로토콜 | HTTP/1.1 + SSE (스트리밍) |
| 인증 | API Key (`Authorization: Bearer <key>`) |
| Content-Type | `application/json` |

---

## 2. 인증 (필수)

### 2.1 API Key 발급

관리자가 Mimir Admin UI → 에이전트 관리 → 에이전트 상세 → "API Key 재발급" 버튼을 통해 발급합니다.

> **⚠️ 중요**: API Key 생성 시 반드시 `expires_at`을 명시해야 합니다.  
> `expires_at`이 NULL인 키는 인증 서버에서 **즉시 거부**됩니다.  
> 권장 만료 기간: 90일 이하. 만료 전 재발급을 자동화하세요.

```json
{
  "expires_at": "2026-07-17T00:00:00Z"
}
```

### 2.2 요청 헤더

```http
Authorization: Bearer mim_sk_xxxxxxxxxxxxxxxxxxxx
Content-Type: application/json
```

### 2.3 인증 오류 응답

| 상태 코드 | 사유 |
|-----------|------|
| `401 Unauthorized` | 키 없음 또는 형식 오류 |
| `403 Forbidden` | 키 만료, 에이전트 차단(킬스위치 발동) |
| `429 Too Many Requests` | Rate Limit 초과 |

---

## 3. Rate Limit

API Key별 Rate Limit은 관리자가 에이전트 상세 페이지에서 동적으로 설정합니다.

| 작업 유형 | 기본 한도 (분당) | 헤더 |
|-----------|-----------------|------|
| `tool_call` | 20 | `X-RateLimit-ToolCall-Remaining` |
| `stream` | 10 | `X-RateLimit-Stream-Remaining` |
| `read` | 60 | `X-RateLimit-Read-Remaining` |
| `init` | 30 | `X-RateLimit-Init-Remaining` |

Rate Limit 초과 시 `429` 응답과 함께 `Retry-After` 헤더가 반환됩니다.

---

## 4. MCP Tool 목록

### 4.1 초기화 (세션 시작)

```
POST /api/v1/mcp/initialize
```

**요청 바디:**
```json
{
  "client_info": {
    "name": "my-langchain-agent",
    "version": "1.0.0"
  },
  "protocol_version": "2024-11-05"
}
```

**응답:**
```json
{
  "protocol_version": "2024-11-05",
  "capabilities": {
    "tools": {},
    "resources": {}
  },
  "server_info": {
    "name": "mimir-mcp-server",
    "version": "2.0.0"
  },
  "session_id": "sess_xxxx"
}
```

---

### 4.2 Tool 목록 조회

```
GET /api/v1/mcp/tools/list
```

**응답 (예시):**
```json
{
  "tools": [
    {
      "name": "search_documents",
      "description": "Mimir 문서를 전문 검색합니다.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "query": { "type": "string", "description": "검색 쿼리" },
          "limit": { "type": "integer", "default": 10 }
        },
        "required": ["query"]
      }
    },
    {
      "name": "get_document",
      "description": "문서 ID로 전체 내용을 조회합니다.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "document_id": { "type": "string" }
        },
        "required": ["document_id"]
      }
    },
    {
      "name": "propose_document_change",
      "description": "문서 변경을 제안합니다. 관리자 승인 후 반영됩니다.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "document_id": { "type": "string" },
          "proposed_content": { "type": "string" },
          "reason": { "type": "string" }
        },
        "required": ["document_id", "proposed_content"]
      }
    },
    {
      "name": "rag_query",
      "description": "RAG(Retrieval-Augmented Generation) 질의응답을 수행합니다.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "question": { "type": "string" },
          "max_chunks": { "type": "integer", "default": 5 }
        },
        "required": ["question"]
      }
    },
    {
      "name": "list_conversations",
      "description": "사용자의 대화 목록을 조회합니다.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "page": { "type": "integer", "default": 1 },
          "page_size": { "type": "integer", "default": 20 }
        }
      }
    },
    {
      "name": "create_conversation",
      "description": "새 대화를 시작합니다.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "title": { "type": "string" },
          "initial_message": { "type": "string" }
        }
      }
    },
    {
      "name": "send_message",
      "description": "대화에 메시지를 전송하고 RAG 기반 응답을 받습니다.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "conversation_id": { "type": "string" },
          "content": { "type": "string" }
        },
        "required": ["conversation_id", "content"]
      }
    }
  ]
}
```

---

### 4.3 Tool 호출

```
POST /api/v1/mcp/tools/call
```

**요청 바디:**
```json
{
  "name": "search_documents",
  "arguments": {
    "query": "계약서 작성 방법",
    "limit": 5
  }
}
```

**응답:**
```json
{
  "content": [
    {
      "type": "text",
      "text": "[{\"id\": \"doc-123\", \"title\": \"계약서 작성 가이드\", \"score\": 0.92, ...}]"
    }
  ],
  "isError": false
}
```

---

### 4.4 스트리밍 응답 (SSE)

`rag_query` 또는 `send_message` 호출 시 스트리밍을 사용하려면 `Accept: text/event-stream` 헤더를 추가합니다.

```http
POST /api/v1/mcp/tools/call
Accept: text/event-stream
Content-Type: application/json
Authorization: Bearer mim_sk_xxxx

{
  "name": "rag_query",
  "arguments": { "question": "계약서 보존 기간은?" }
}
```

**SSE 스트림 형식:**
```
data: {"type":"text","text":"계약서"}
data: {"type":"text","text":"의 보존"}
data: {"type":"text","text":" 기간은"}
data: {"type":"citations","citations":[{"doc_id":"doc-123","title":"법인 문서 관리 규정","span":"§4.2"}]}
data: [DONE]
```

---

### 4.5 리소스 목록 조회

```
GET /api/v1/mcp/resources/list
```

리소스는 에이전트가 읽을 수 있는 정적 컨텍스트입니다. Scope Profile 기반 ACL이 적용됩니다.

---

## 5. Scope Profile 기반 ACL

에이전트 API Key에는 Scope Profile이 연결됩니다.  
모든 도구 호출은 해당 Profile의 ACL 필터를 통과해야 합니다.

- `search_documents`: Scope `read:documents` 필요
- `propose_document_change`: Scope `write:proposals` 필요
- `rag_query`: Scope `read:documents`, `read:chunks` 필요

Scope가 부족하면 `403 Forbidden`이 반환됩니다.

---

## 6. 감사 로그

모든 도구 호출은 `audit_events` 테이블에 기록됩니다.

```json
{
  "actor_type": "agent",
  "actor_id": "agent-uuid",
  "event_type": "tool_call",
  "resource_type": "document",
  "resource_id": "doc-123",
  "action_result": "success"
}
```

`actor_type: "agent"` 필드가 반드시 기록됩니다 (S2 원칙 ⑥).

---

## 7. LangChain 연동 예시

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({
    "mimir": {
        "url": "https://mimir.example.com/api/v1/mcp",
        "transport": "http",
        "headers": {
            "Authorization": "Bearer mim_sk_xxxxxxxxxxxxxxxxxxxx"
        }
    }
})

# 도구 목록 로드
tools = await client.get_tools()

# LangGraph 에이전트와 통합
from langgraph.prebuilt import create_react_agent
agent = create_react_agent("claude-sonnet-4-6", tools)

result = await agent.ainvoke({
    "messages": [{"role": "user", "content": "계약서 보존 기간을 알려줘"}]
})
```

---

## 8. n8n 연동 예시

n8n의 **HTTP Request** 노드를 사용합니다.

| 항목 | 값 |
|------|-----|
| Method | `POST` |
| URL | `https://mimir.example.com/api/v1/mcp/tools/call` |
| Authentication | Header Auth (`Authorization: Bearer mim_sk_xxx`) |
| Body (JSON) | `{"name": "search_documents", "arguments": {"query": "{{$json.query}}"}}` |

---

## 9. 폐쇄망 환경

Mimir는 외부 LLM SaaS(OpenAI 등)를 환경변수로 비활성화할 수 있습니다.  
폐쇄망 환경에서는:

- `rag_query` → 로컬 LLM(Ollama 등)으로 폴백  
- `search_documents` → FTS(전문 검색) 폴백  
- ACL 필터는 폴백 경로에서도 **동일하게 적용**됩니다

`/api/v1/system/capabilities` 응답으로 현재 가용 기능을 확인하세요.

---

## 10. 오류 코드 요약

| 코드 | 의미 | 처리 방법 |
|------|------|-----------|
| `400` | 잘못된 요청 (파라미터 오류) | 요청 바디 검토 |
| `401` | 인증 실패 | API Key 확인 |
| `403` | 권한 부족 또는 에이전트 차단 | Scope Profile 또는 킬스위치 상태 확인 |
| `404` | 리소스 없음 | ID 확인 |
| `429` | Rate Limit 초과 | `Retry-After` 헤더 준수 |
| `500` | 서버 내부 오류 | 재시도 또는 운영팀 문의 |
| `503` | 서비스 일시 불가 (폴백 모드) | `/system/capabilities` 확인 |

---

*최종 수정: 2026-04-17 · Phase 6 FG6.2 산출물 (DOCS-001 이월 처리)*
