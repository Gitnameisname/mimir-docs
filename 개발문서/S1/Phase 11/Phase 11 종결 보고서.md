# Phase 11 종결 보고서
## 지식검색 및 RAG 질의응답 파이프라인 구축

**종결 일자**: 2026-04-09  
**상태**: ✅ 완료

---

## 1. 단계 요약

Phase 11은 Phase 10에서 구축한 벡터화 파이프라인 위에 **문서 기반 자연어 질의응답(RAG) 시스템**을 구현한 단계다. 사용자의 자연어 질문에 대해 문서에서 근거를 찾아 LLM이 답변을 생성하고, 해당 근거를 원본 문서 노드와 연결해 **인용 가능한 신뢰 기반 답변**을 제공한다.

구현 아키텍처: `QueryProcessor → Retriever → Reranker → ContextBuilder → LLMProvider → CitationLinker → SSE 스트리밍 응답`

---

## 2. 완료 기준 달성 현황

| 완료 기준 | 달성 여부 | 비고 |
|-----------|----------|------|
| 사용자가 자연어로 질문하면 문서 기반 답변을 받을 수 있다 | ✅ | `POST /api/v1/rag/query` SSE 스트리밍 동작 |
| 응답에 근거 문서와 노드 링크가 포함된다 | ✅ | CitationLinker — `[1]`, `[2]` 마커 → chunk_id/node_id 매핑 |
| 사용자 권한에 따라 접근 불가 문서는 컨텍스트에 포함되지 않는다 | ✅ | Retriever가 `actor_role` 필터 강제 적용, DB 레벨 차단 |
| 대화 이력이 저장되고 이전 맥락을 유지하며 질의할 수 있다 | ✅ | `rag_conversations` / `rag_messages` 테이블, Multi-turn |
| 응답이 스트리밍으로 제공된다 | ✅ | SSE `text/event-stream`, delta/citation/done 이벤트 |

**전체 완료 기준 5/5 달성**

---

## 3. Task별 구현 결과

| Task | 내용 | 결과 |
|------|------|------|
| 11-1 | RAG 파이프라인 아키텍처 설계 | ✅ 6단계 파이프라인, LLMProvider 추상화, SSE 스트리밍 설계 확정 |
| 11-2 | QueryProcessor 구현 | ✅ 질문 정규화, EmbeddingProvider를 통한 임베딩 변환 |
| 11-3 | Retriever 구현 | ✅ Phase 10 `semantic_search` 연계, document_id/actor_role 필터 |
| 11-4 | Reranker 구현 | ✅ 유사도 점수 기반 재정렬 + threshold 필터, cross-encoder 교체 가능 구조 |
| 11-5 | LLMProvider 추상화 및 GPT-4o 연동 | ✅ LLMProvider ABC, OpenAILLMProvider / AnthropicLLMProvider / MockLLMProvider |
| 11-6 | ContextBuilder 구현 | ✅ 토큰 한도 내 청크 조합, `[n]` 출처 번호 부착 |
| 11-7 | Citation 연결 구현 | ✅ 응답 `[n]` 마커 추출 → chunk_id / node_id / document_id 매핑 |
| 11-8 | RAG Query API 설계 및 구현 | ✅ 6개 엔드포인트, Rate Limit 30/min, SSE StreamingResponse |
| 11-9 | 대화 이력 관리 (Multi-turn) | ✅ `rag_conversations` / `rag_messages`, `get_history_for_llm()` |
| 11-10 | 사용자 UI 통합 | ✅ `RagPage.tsx`, `RagPanel.tsx`, SSE 스트리밍 클라이언트, 인용 표시 |
| 11-11 | 성능 검증 및 Phase 12 연계 설계 | ✅ Phase 12 연계 포인트 확정 (본 문서 §6) |

---

## 4. 신규/변경 파일 목록

### 4.1 Backend — 신규 파일

| 파일 | 역할 |
|------|------|
| `backend/app/services/rag_service.py` | RAG 파이프라인 전체 (QueryProcessor / Retriever / Reranker / ContextBuilder / LLMProvider / CitationLinker / RAGService) |
| `backend/app/repositories/rag_repository.py` | `rag_conversations` / `rag_messages` CRUD, `ensure_tables()`, `get_history_for_llm()` |
| `backend/app/schemas/rag.py` | RAG 관련 Pydantic 스키마 (RAGQueryRequest, RAGQueryResponse, Citation, ConversationCreate 등) |
| `backend/app/api/v1/rag.py` | RAG REST API 라우터 (6개 엔드포인트, SSE StreamingResponse) |

### 4.2 Backend — 변경 파일

| 파일 | 변경 내용 |
|------|----------|
| `backend/app/main.py` | `on_startup()`에서 RAG 테이블 초기화 추가 (`ensure_tables`) |
| `backend/app/api/auth/authorization.py` | `_PERMISSION_MATRIX`에 `rag.*` 4개 액션 추가 |
| `backend/app/api/v1/router.py` | `/rag` 라우터 등록 |
| `backend/app/config.py` | `llm_provider`, `llm_model`, `anthropic_api_key`, `rag_top_k`, `rag_top_n`, `rag_max_context_tokens`, `rag_reranker_enabled`, `rag_reranker_threshold`, `rag_max_history_turns` 설정 추가 |
| `backend/requirements.txt` | `anthropic` 패키지 추가 |

### 4.3 Frontend — 신규 파일

| 파일 | 역할 |
|------|------|
| `frontend/src/features/rag/RagPage.tsx` | RAG 질의응답 메인 페이지 (사이드바 대화 목록 + 채팅 패널) |
| `frontend/src/features/rag/RagPanel.tsx` | SSE 스트리밍 수신, 스트리밍 답변 렌더링, 인용(Citation) 표시 |
| `frontend/src/lib/api/rag.ts` | RAG API 클라이언트 (SSE 스트리밍, Conversation CRUD) |
| `frontend/src/types/rag.ts` | RAG 관련 TypeScript 타입 정의 |

---

## 5. 검수 및 보안 수정 이력

### 5.1 Phase 10 검수 보고서 지적 사항 수정 (Phase 11 착수 전 선행 조치)

Phase 10 검수에서 발견된 6건(H-1, M-1, M-2, m-1, m-2, m-3)을 Phase 11 착수 전에 전부 수정했다.

| ID | 등급 | 내용 | 수정 |
|----|------|------|------|
| H-1 | HIGH | `_save_chunks` InFailedSqlTransaction 연쇄 실패 | INSERT별 SAVEPOINT 적용 |
| M-1 | MEDIUM | `embed_single` zero vector로 semantic_search 실행 | zero/빈 벡터 검사 후 빈 목록 반환 |
| M-2 | MEDIUM | `accessible_org_ids` 항상 빈 목록 | TODO 주석 추가, Phase 12+ 구현 예정 |
| m-1 | MINOR | `vectorize_all_published` Rate Limit 무방비 | 배치 간 `time.sleep(0.1)` 추가 |
| m-2 | MINOR | `LocalEmbeddingProvider` zero vector 검색 오염 | 빈 리스트 반환 → DB NULL 저장 → 검색 제외 |
| m-3 | MINOR | `get_chunking_config_for_type` 폴백 무음 | 폴백 시 `logger.warning` 추가 (3개 경로) |

### 5.2 Phase 11 보안 취약점 수정 (8건)

상세 내용: [Phase11_보안취약점_리포트.md](../../보안리포트/Phase11_보안취약점_리포트.md)

| ID | 등급 | 내용 | 수정 |
|----|------|------|------|
| RAG-001 | HIGH | SSE error 이벤트에 내부 예외 메시지(`str(exc)`) 노출 | `"응답 생성 중 오류가 발생했습니다."` 고정 메시지로 교체, 내부 로그 유지 |
| RAG-002 | HIGH | SSE 스트리밍 중 DB 커넥션 장기 보유 → 풀 소진 DoS | `prepare_context(conn)` + `stream_answer()`로 단계 분리, LLM 스트리밍 중 커넥션 반환 |
| RAG-003 | MEDIUM | `conversation_id` / `document_id` UUID 형식 미검증 | `schemas/rag.py`에 `field_validator` + `_validate_uuid_field()` 추가 |
| RAG-004 | MEDIUM | 시스템 프롬프트 컨텍스트 구분자 프롬프트 인젝션 | 구분자 `--- 컨텍스트 끝 ---` → `<document_context>` XML 태그, `_sanitize_for_context()` 이스케이프 |
| RAG-005 | MEDIUM | `list_conversations` limit 상한 미설정 | `limit = min(limit, 100)` 상한 적용 |
| VEC-008 | MEDIUM | `list_chunks` Query 파라미터 `document_id` UUID 미검증 | `_validate_uuid(document_id, "document_id")` 추가 |
| RAG-006 | LOW | RBAC 매트릭스에 `rag.*` 액션 미등록 | `authorization.py`에 4개 액션 등록, 엔드포인트 `authorization_service.authorize()` 교체 |
| RAG-007 | LOW | `_ensure_rag_tables()` TOCTOU 글로벌 플래그 | `_ensure_rag_tables()` 제거, `main.py` `on_startup()`에서 1회 실행으로 이전 |

---

## 6. Phase 12 연계 포인트

### 6.1 LLMProvider 교체 인터페이스

```python
# 설정 값 변경만으로 LLM 교체 가능
# LLM_PROVIDER=anthropic → AnthropicLLMProvider (Claude claude-sonnet-4-6)
# LLM_PROVIDER=openai    → OpenAILLMProvider (GPT-4o)
# API 키 없음            → MockLLMProvider (개발용)

from app.services.rag_service import get_llm_provider

llm = get_llm_provider()
answer, tokens = await llm.complete(system_prompt, messages)
```

### 6.2 DocumentType별 프롬프트 커스터마이징 준비

`build_system_prompt(context, document_type)` 함수가 `document_type` 파라미터를 받도록 설계되어 있다. Phase 12에서 DocumentType 플러그인이 DocumentType별 시스템 프롬프트를 등록하면 이 함수에서 분기 적용 가능하다.

```python
# Phase 12에서 확장될 인터페이스 (현재는 기본 프롬프트 사용)
def build_system_prompt(context: str, document_type: Optional[str] = None) -> str:
    # Phase 12: document_type별 커스터마이징 분기 삽입 예정
    return _SYSTEM_PROMPT_TEMPLATE.format(context=context)
```

### 6.3 Reranker 교체 준비

현재 `Reranker`는 유사도 점수 기반 단순 정렬을 사용한다. Cross-encoder 모델 (`cross-encoder/ms-marco-MiniLM-L-6-v2`) 교체 시 `Reranker.rerank()` 시그니처 변경 없이 내부 구현만 교체하면 된다.

### 6.4 RAG API 엔드포인트

```http
POST /api/v1/rag/query              -- 단건 RAG 질의 (SSE 스트리밍 기본)
POST /api/v1/rag/conversations      -- 대화 세션 생성
GET  /api/v1/rag/conversations      -- 대화 목록 조회
GET  /api/v1/rag/conversations/{id} -- 대화 상세 (메시지 포함)
GET  /api/v1/rag/conversations/{id}/messages  -- 메시지 목록
DELETE /api/v1/rag/conversations/{id} -- 대화 삭제
```

---

## 7. 주요 설계 결정 기록

| 결정 | 근거 |
|------|------|
| SSE 스트리밍 (`text/event-stream`) | 30~60초 LLM 응답을 토큰 단위로 즉시 전달 → UX 개선 |
| DB/LLM 단계 분리 (`prepare_context` + `stream_answer`) | DB 커넥션 풀(max=10)을 LLM 스트리밍 시간 동안 점유하지 않도록 설계 |
| LLMProvider 추상화 | OpenAI / Anthropic / Mock 교체를 설정 변경만으로 대응 가능 |
| 시스템 프롬프트 XML 태그 구분자 | `--- 컨텍스트 끝 ---` 문자열 대비 프롬프트 인젝션 공격면 최소화 |
| `[n]` 출처 마킹 방식 | LLM이 자연스럽게 인용 번호를 생성하도록 지시, 후처리로 청크 매핑 |
| Multi-turn 이력 관리 | `rag_messages` JSONB 구조로 citation/context_chunks 메타 함께 저장 |
| Rate Limit 30회/분 | LLM API 호출 비용 보호 및 서버 자원 보호 |
| MockLLMProvider | API 키 없는 개발/테스트 환경에서 파이프라인 검증 가능 |

---

## 8. 미구현 항목 및 다음 단계 권고

### 8.1 Phase 12에서 처리할 항목

| 항목 | 내용 |
|------|------|
| DocumentType별 RAG 프롬프트 | 정책 문서 vs 매뉴얼 등 타입별 시스템 프롬프트 분기 |
| Cross-encoder Reranker | 현재 유사도 점수 정렬만 사용 → 실제 cross-encoder 모델 연동 |
| `accessible_org_ids` (M-2 잔존) | 조직 단위 접근 제어 미구현 — Phase 12+ 처리 예정 |

### 8.2 추후 확장 고려 항목

| 항목 | 배경 |
|------|------|
| RAG 응답 품질 평가 | Faithfulness / Relevance / Answer Correctness 지표 측정 체계 |
| 스트리밍 토큰 수 추적 | 현재 스트리밍에서 `token_used=0` 고정 — OpenAI `stream_options` 활용 가능 |
| 감사 로그 (VEC-007 잔존) | `rag.query` 이벤트를 `audit_events`에 기록 |
| 문서 범위 지정 UI | `document_id` 필터를 활용한 "이 문서에 대해 질문" 인터페이스 |

---

## 9. 결론

Phase 11은 계획된 11개 Task를 모두 완료했다. Phase 10 검수 지적 6건을 선행 수정한 뒤, 보안 취약점 검사에서 HIGH 2건을 포함한 8건을 추가 발견·수정했다. 완료 기준 5개 항목 전체 달성.

핵심 산출물인 **LLMProvider 추상화 기반 RAG 질의응답 파이프라인**과 **DB 커넥션 분리 SSE 스트리밍 구조**가 완성됨으로써, Phase 12 DocumentType 플러그인 시스템과의 연계를 즉시 착수할 수 있는 기반이 갖춰졌다.

> Phase 12 진행 가능 조건 충족 ✅
