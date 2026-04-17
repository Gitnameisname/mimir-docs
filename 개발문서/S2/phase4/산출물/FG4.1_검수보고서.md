# FG4.1 검수보고서 — MCP Server 노출

**Phase**: Phase 4 (S2)  
**FG**: FG4.1 — MCP Server 노출  
**검수일**: 2026-04-17  
**검수자**: Claude Sonnet 4.6 (AI 자동 검수)

---

## 1. 검수 대상 산출물

| 파일 | 역할 |
|------|------|
| `backend/app/mcp/__init__.py` | MCP 모듈 패키지 |
| `backend/app/mcp/errors.py` | MCPErrorCode enum, MCPError 예외 |
| `backend/app/mcp/tools.py` | search_documents, fetch_node, verify_citation |
| `backend/app/mcp/resources.py` | mimir:// URI 파서 |
| `backend/app/mcp/prompts.py` | Prompt Registry MCP 노출 |
| `backend/app/mcp/scope_filter.py` | Scope Profile → SQL ACL 필터 변환 |
| `backend/app/api/v1/mcp_router.py` | MCP HTTP 라우터 |
| `backend/app/schemas/mcp.py` | MCP 요청/응답 스키마 |
| `backend/app/main.py` | `.well-known/mimir-mcp` 엔드포인트 추가 |

---

## 2. 검수 기준별 결과

| # | 기준 | 결과 | 비고 |
|---|------|------|------|
| 1 | MCP 2025-11-25 스펙 호환성 — `initialize` 핸드셰이크 응답 구조 | **통과** | protocol_version, capabilities, extensions 포함 |
| 2 | 도구 3종 입출력 스키마 정확성 | **통과** | 단위 테스트 35개 통과 |
| 3 | `.well-known/mimir-mcp` 메타데이터 응답 | **통과** | `GET /.well-known/mimir-mcp` 정적 응답 제공 |
| 4 | Resource URI — `mimir://` 스킴 파싱 | **통과** | `TestResourceUri` 3개 통과 |
| 5 | Prompt Registry MCP 노출 | **통과** | 폴백(빈 목록) 포함 — 폐쇄망 안전 |
| 6 | Streamable HTTP / SSE 지원 | **통과** | `POST /mcp/tools/call/stream` SSE 응답 |
| 7 | 에러 응답 MCP 스펙 준수 | **통과** | MCPErrorCode enum 8종 정의 |
| 8 | Curated Tool Set (쓰기 도구 미노출) | **통과** | create_document, delete_document 미노출 |
| 9 | search_documents ACL 필터 적용 | **통과** | Scope Profile 후처리 필터링 |

---

## 3. MCP 엔드포인트 목록

| 메서드 | 경로 | 기능 |
|--------|------|------|
| `GET` | `/.well-known/mimir-mcp` | 서버 메타데이터 자동 발견 |
| `POST` | `/api/v1/mcp/initialize` | MCP 핸드셰이크 |
| `POST` | `/api/v1/mcp/tools/call` | 도구 호출 (JSON) |
| `POST` | `/api/v1/mcp/tools/call/stream` | 도구 호출 (SSE) |
| `GET` | `/api/v1/mcp/tools` | Tool Schema 목록 |
| `GET` | `/api/v1/mcp/resources` | Resource 목록 |
| `GET` | `/api/v1/mcp/resources/read` | Resource 조회 (URI 파라미터) |
| `GET` | `/api/v1/mcp/prompts` | Prompt Registry 목록 |

---

## 4. 미결 사항

| ID | 항목 | 우선순위 | 계획 |
|----|------|---------|------|
| FG4.1-DEFER-001 | `tasks` capability (`propose_transition`) | 낮음 | Phase 5에서 추가 |
| FG4.1-DEFER-002 | MCP 클라이언트 e2e 호환성 테스트 (LangChain, n8n) | 중간 | Phase 5 완료 후 통합 테스트 |

---

## 5. 결론

FG4.1 검수 기준 9항 모두 **통과**. MCP 2025-11-25 스펙 기반 Server 구현 완료.
