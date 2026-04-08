# Task 11-1. RAG 파이프라인 아키텍처 설계

## 1. 작업 목적

Phase 11 전체 RAG 파이프라인의 기술 방향과 컴포넌트 구조를 확정한다.

이 작업의 목표는 다음과 같다.

- 전체 RAG 흐름 설계 (QueryProcessor → Retriever → Reranker → LLM → Citation)
- Phase 10 VectorSearchProvider / HybridSearchProvider 연계 구조 확정
- LLM Provider 추상화 구조 설계
- 권한/버전 반영 원칙 확정
- Phase 8 SearchService와의 역할 구분 명확화

---

## 2. 작업 범위

### 포함 범위

- RAG 전체 컴포넌트 정의 및 역할 분리
- 각 컴포넌트 인터페이스 초안 정의
- Phase 10 연계 구조 확정
- 권한 반영 전략 확정
- 스트리밍 응답 구조 설계
- 대화 세션 데이터 구조 설계

### 제외 범위

- 각 컴포넌트 실제 구현 (Task 11-2~11-10에서 다룸)

---

## 3. 선행 조건

- Phase 10 Task 10-9 (HybridSearchProvider) 완료 — RAGContextProvider 인터페이스 확정
- Phase 10 Task 10-11 (성능 검증) — ef_search 최종값, 기본 검색 모드 확정

---

## 4. 주요 설계 대상

### 4-1. 전체 파이프라인 흐름

```
[사용자 질문 + 대화 이력 + 권한 컨텍스트]
            ↓
    [QueryProcessor]
      - 질문 정규화
      - 대화 이력 컨텍스트 압축
      - 임베딩 변환 (EmbeddingProvider)
            ↓
      [Retriever]
        - HybridSearchProvider 호출 (권한 필터 포함)
        - Top-K 청크 반환 (기본 K=20)
            ↓
      [Reranker] (선택적)
        - Cross-encoder 재랭킹
        - Top-N 선택 (기본 N=5)
            ↓
    [ContextBuilder]
      - 청크 → 프롬프트 컨텍스트 조합
      - 토큰 한도 관리
            ↓
      [LLMService]
        - LLMProvider 호출
        - 스트리밍 응답
            ↓
    [CitationLinker]
      - 응답 내 출처 마킹 파싱
      - 청크 → node_id → 문서 링크 매핑
            ↓
    [RAGResponse]
      answer / citations / context_chunks
```

---

### 4-2. 컴포넌트 역할 구분

| 컴포넌트 | 역할 | Phase 10 연계 |
|---------|------|-------------|
| QueryProcessor | 쿼리 전처리, 임베딩 변환 | EmbeddingProvider 재사용 |
| Retriever | 문서 청크 검색 | RAGContextProvider (Task 10-11 인터페이스) 구현 |
| Reranker | 검색 결과 재정렬 | 독립 컴포넌트 |
| ContextBuilder | LLM 프롬프트 조합 | 독립 컴포넌트 |
| LLMService | 텍스트 생성 | LLMProvider 추상화 |
| CitationLinker | 출처 매핑 | document_chunks.node_id 참조 |

---

### 4-3. Phase 8 SearchService와의 역할 구분

| 기능 | SearchService (Phase 8) | RAG Retriever (Phase 11) |
|------|------------------------|------------------------|
| 목적 | 문서 검색 결과 반환 | LLM 컨텍스트용 청크 반환 |
| 출력 | SearchResult[] (문서 목록) | RetrievedChunk[] (청크 목록) |
| 권한 | 동일 (is_current + ACL 필터) | 동일 |
| UI 연결 | 검색 결과 페이지 | RAG 응답 생성 |

두 경로는 동일한 HybridSearchProvider를 내부적으로 재사용하되, 출력 형태와 목적이 다름.

---

### 4-4. 스트리밍 응답 구조

LLM 응답을 SSE(Server-Sent Events)로 스트리밍:

```
이벤트 종류:
  data: {"type": "chunk", "content": "안녕하세요..."}
  data: {"type": "chunk", "content": " 해당 정책에"}
  ...
  data: {"type": "citations", "citations": [...]}
  data: {"type": "done", "conversation_id": "...", "message_id": "..."}
  data: {"type": "error", "message": "..."}
```

---

### 4-5. 대화 세션 구조

```
RAGConversation
  id: UUID
  user_id: UUID
  title: str (첫 질문 자동 생성)
  messages: RAGMessage[]
  created_at

RAGMessage
  id: UUID
  conversation_id: UUID
  role: "user" | "assistant"
  content: str
  citations: Citation[]
  context_chunks: RetrievedChunk[]  (요약 저장)
  token_used: int
  model: str
  created_at
```

---

### 4-6. LLMProvider 추상화

```
LLMProvider (인터페이스)
  .complete(prompt: LLMPrompt) → LLMResponse
  .stream(prompt: LLMPrompt) → AsyncIterator[str]
  .get_model_name() → str
  .get_context_limit() → int

LLMPrompt
  system: str
  messages: LLMMessage[]
  max_tokens: int
  temperature: float

구현체:
  OpenAILLMProvider (GPT-4o)
  AnthropicLLMProvider (Claude, Reserved)
```

---

## 5. 산출물

1. RAG 파이프라인 전체 흐름 다이어그램
2. 컴포넌트 역할 정의서
3. Phase 10 연계 구조 문서
4. LLMProvider 인터페이스 초안
5. 스트리밍 응답 이벤트 명세
6. 대화 세션 데이터 구조

---

## 6. 완료 기준

- RAG 전체 컴포넌트가 역할과 인터페이스 수준으로 정의되어 있다
- Phase 10 VectorSearchProvider/HybridSearchProvider와의 연계 구조가 명확하다
- LLMProvider 추상화로 GPT-4o 이외 모델 교체가 가능한 구조다
- 권한 필터가 Retriever 단계에서 적용되는 원칙이 명시되어 있다
- 이후 Task(11-2~11-11)가 이 문서를 기준으로 진행 가능하다

---

## 7. Codex 작업 지침

- 실제 코드 구현은 하지 않는다 (설계 중심)
- LLM API 키는 환경 변수로 관리하며 코드에 하드코딩하지 않는다
- 권한 필터는 Retriever 내부에서만 처리하며, LLM이 권한 외 문서를 볼 수 없도록 설계 원칙을 명시한다
- Phase 12 DocumentType 플러그인이 RAG 프롬프트 커스터마이징을 지원할 수 있도록 PromptTemplate 확장 포인트를 설계 원칙에 포함한다
