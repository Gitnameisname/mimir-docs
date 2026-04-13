# Phase 11. 지식검색 및 RAG 활용 계층 구축

## 1. Phase 목적

Phase 10에서 구축한 벡터화 파이프라인을 기반으로, 문서 기반 질의응답 시스템(RAG)을 구축한다.

단순한 검색을 넘어 사용자의 자연어 질문에 대해 문서에서 근거를 찾아 답변을 생성하고, 해당 근거를 원본 문서 노드와 연결하여 신뢰 가능한 지식 플랫폼으로 진화한다.

---

## 2. Phase 11 범위

### 포함 범위

- RAG 파이프라인 전체 아키텍처 설계 및 구현
- 쿼리 처리 및 임베딩 변환
- Retriever 구현 (Phase 10 VectorSearchProvider 연계)
- Reranker 구현 (검색 품질 향상)
- LLM 연동 및 응답 생성 (OpenAI GPT-4 계열)
- 근거(Citation) 연결 — 응답 문장과 원본 노드 매핑
- 버전/권한 반영 응답 보장
- RAG Query API 설계 및 구현
- 대화 이력 관리 (Multi-turn 지원)
- 사용자 UI 통합 (Phase 6 확장)
- 성능 검증 및 Phase 12 연계 포인트 설계

### 제외 범위

- DocumentType 플러그인 구조 (Phase 12에서 다룸)
- 운영/보안/배포 체계 (Phase 13에서 다룸)
- 임베딩 생성 자체 (Phase 10에서 완성됨)

---

## 3. 선행 조건

- Phase 10 완료 — document_chunks 테이블, VectorSearchProvider, HybridSearchProvider 동작
- Phase 8 SearchService 인터페이스 이해
- Phase 3 API 플랫폼 구조 이해
- Phase 6 사용자 UI 구조 이해
- Phase 4 권한/버전 구조 이해

---

## 4. 핵심 기술 결정

### 4-1. RAG 아키텍처

```
[사용자 질문]
      ↓
[QueryProcessor]
  - 질문 정규화
  - 임베딩 변환 (EmbeddingProvider)
      ↓
[Retriever]
  - HybridSearch (FTS + 벡터)
  - 권한 필터 적용
  - Top-K 청크 반환
      ↓
[Reranker]
  - Cross-encoder 기반 재랭킹
  - 최종 컨텍스트 청크 선택 (Top-N)
      ↓
[ContextBuilder]
  - 청크 → LLM 프롬프트 컨텍스트 조합
  - 토큰 한도 관리
      ↓
[LLMService]
  - OpenAI GPT-4o 연동
  - 시스템 프롬프트 + 컨텍스트 + 사용자 질문
  - 응답 스트리밍 지원
      ↓
[CitationLinker]
  - 응답 문장 → 근거 청크 매핑
  - 청크 → 원본 node_id → 문서 링크 생성
      ↓
[RAGResponse]
  - answer: str
  - citations: Citation[]
  - context_chunks: RetrievedChunk[]
  - conversation_id: str
```

---

### 4-2. LLM 선택

| 항목 | 결정 |
|------|------|
| 기본 모델 | OpenAI GPT-4o |
| 대안 | Claude claude-sonnet-4-6 (Anthropic) |
| 추상화 | LLMProvider 인터페이스로 모델 교체 가능 |
| 스트리밍 | SSE(Server-Sent Events) 지원 |
| API 키 관리 | 환경 변수, Admin UI에서 모델 설정 |

---

### 4-3. Reranker 전략

| 전략 | 설명 | 선택 이유 |
|------|------|---------|
| Cross-Encoder | 쿼리-청크 쌍을 함께 인코딩하여 관련성 점수 | 정확도 높음 |
| Score Threshold | 일정 점수 미만 청크 제거 | 노이즈 컨텍스트 방지 |
| 기본 모델 | `cross-encoder/ms-marco-MiniLM-L-6-v2` | 경량, 빠름 |

Reranker는 Retriever 이후 선택적으로 적용 (설정으로 ON/OFF).

---

### 4-4. Citation 전략

- 응답 생성 시 LLM에 출처 마킹 지시 (`[1]`, `[2]` 형식)
- 각 출처 번호를 청크의 `node_id` + `document_id`와 매핑
- UI에서 출처 번호 클릭 시 원본 문서 노드로 이동

---

### 4-5. 권한 반영 원칙

- Retriever 단계에서 사용자 권한 필터 적용 (Phase 10 동일 구조)
- 응답에 포함된 근거 청크는 모두 사용자가 접근 권한이 있는 문서에서만 추출
- 접근 불가 문서가 컨텍스트에 포함되지 않도록 쿼리 레벨에서 차단

---

## 5. Task 목록

| Task | 이름 | 주요 내용 |
|------|------|---------|
| 11-1 | RAG 파이프라인 아키텍처 설계 | 전체 흐름, 기술 선택, Phase 10 연계 구조 |
| 11-2 | QueryProcessor 구현 | 쿼리 정규화, 임베딩 변환, 의도 분석 |
| 11-3 | Retriever 구현 | RAGContextProvider, HybridSearch 연계 |
| 11-4 | Reranker 구현 | Cross-encoder 재랭킹, 컨텍스트 필터링 |
| 11-5 | LLMProvider 추상화 및 GPT-4o 연동 | LLMProvider 인터페이스, 스트리밍, 프롬프트 설계 |
| 11-6 | ContextBuilder 구현 | 청크 조합, 토큰 한도 관리, 프롬프트 구조화 |
| 11-7 | Citation(근거) 연결 구현 | 응답-청크 매핑, 문서 노드 링크 생성 |
| 11-8 | RAG Query API 설계 및 구현 | /api/rag/query, 스트리밍 SSE, 대화 이력 |
| 11-9 | 대화 이력 관리 (Multi-turn) | Conversation 세션, 이전 컨텍스트 유지 |
| 11-10 | 사용자 UI 통합 (Phase 6 확장) | RAG 질의 UI, 근거 표시, 스트리밍 응답 |
| 11-11 | 성능 검증 및 Phase 12 연계 포인트 설계 | Latency, 품질 평가, Phase 12 확장 인터페이스 |

---

## 6. 데이터 구조

### RAGConversation 테이블

```sql
CREATE TABLE rag_conversations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  title           TEXT,
  created_at      TIMESTAMP WITH TIME ZONE DEFAULT now(),
  updated_at      TIMESTAMP WITH TIME ZONE DEFAULT now()
);
```

### RAGMessage 테이블

```sql
CREATE TABLE rag_messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES rag_conversations(id) ON DELETE CASCADE,
  role            VARCHAR(20) NOT NULL,  -- 'user' | 'assistant'
  content         TEXT NOT NULL,
  citations       JSONB,                 -- Citation[]
  context_chunks  JSONB,                 -- RetrievedChunk[] (요약)
  token_used      INTEGER,
  model           VARCHAR(100),
  created_at      TIMESTAMP WITH TIME ZONE DEFAULT now()
);
```

---

## 7. API 구조 개요

```
POST /api/rag/query                 -- 단건 RAG 질의 (스트리밍 지원)
POST /api/rag/conversations         -- 대화 세션 생성
GET  /api/rag/conversations         -- 대화 목록 조회
GET  /api/rag/conversations/:id     -- 대화 상세 조회
GET  /api/rag/conversations/:id/messages  -- 메시지 목록
DELETE /api/rag/conversations/:id   -- 대화 삭제
```

---

## 8. Phase 12 연계 고려사항

- LLMProvider 인터페이스는 DocumentType-independent하게 설계
- Phase 12 DocumentType 플러그인에서 타입별 RAG 프롬프트 커스터마이징 가능하도록 PromptTemplate 추상화 준비
- Reranker도 DocumentType별 설정 가능하도록 config 기반 구조 유지

---

## 9. 완료 기준

- 사용자가 자연어로 질문하면 문서 기반 답변을 받을 수 있다
- 응답에 근거 문서와 노드 링크가 포함된다
- 사용자 권한에 따라 접근 불가 문서는 컨텍스트에 포함되지 않는다
- 대화 이력이 저장되고 이전 맥락을 유지하며 질의할 수 있다
- 응답이 스트리밍으로 제공된다
