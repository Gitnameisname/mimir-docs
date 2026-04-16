# Phase 2 개발계획서
## Grounded Retrieval v2 + Citation Contract

---

## 1. 단계 개요

### 단계명
Phase 2. Grounded Retrieval v2 + Citation Contract

### 목적
검색 응답의 citation을 "검증 가능한 좌표"로 표준화하여, 챗봇과 에이전트가 근거를 신뢰할 수 있도록 한다. 동시에 Retriever/Reranker를 플러그인 구조로 재정리하고, 멀티턴 대화를 위한 쿼리 재작성 및 컨텍스트 압축을 구현한다.

### 선행 조건
- Phase 0 완료 (S2 원칙, 설계 결정 확정)
- Phase 1 완료 (LLM, Embedding, Prompt Registry 추상화)
- S1 Phase 11 RAG 구현 분석 (기존 검색 응답 포맷 이해)
- S1 Phase 10 벡터화 파이프라인 기초 (선택, Phase 2와 병행 가능)

### 기대 결과
- 검색 응답에 citation 5-tuple `{document_id, version_id, node_id, span_offset, content_hash}` 필수 포함
- 범위를 벗어나지 않은 증거(grounded) 검색 결과만 반환
- Retriever/Reranker를 플러그인으로 교체 가능
- 멀티턴 대화에서 쿼리 재작성 지원
- S1 Phase 11 RAG 응답과의 하위호환성 유지
- 챗봇 인계 메모 v2 산출 (citation 계약 반영)

---

## 2. Phase 2의 역할

Phase 2는 "근거 기반 검색(Grounded Retrieval)"을 정의한다. S1 Phase 11의 RAG는 "벡터 유사도로 가장 비슷한 청크를 반환"하는 수준이었다면, Phase 2는 다음을 추가한다:

1. **Citation 계약**: 각 검색 결과가 "이 위치의 이 내용"임을 검증 가능하게 좌표 기입
2. **Retriever/Reranker 플러그인화**: DocumentType별로 다른 검색 전략(FTS, Vector, Hybrid) 선택 가능
3. **멀티턴 지원**: 단발 쿼리 → 멀티턴 대화의 follow-up 질의를 자립적으로 재작성
4. **컨텍스트 압축**: 이전 대화의 citation을 재활용하여 불필요한 재검색 회피

이후 Phase 3 (Conversation), Phase 4 (Agent Interface), Phase 7 (평가) 등이 이 Phase의 citation 계약을 기준으로 동작한다.

---

## 3. Feature Group 상세

### FG2.1 — Citation 5-tuple 계약 고정

#### 목적
검색 응답의 근거를 검증 가능한 좌표로 표준화한다.

#### 주요 작업 항목

1. **Citation 5-tuple 정의 및 검증**
   - 구조:
     ```python
     Citation = {
         document_id: UUID,      # 문서 ID
         version_id: UUID,       # 버전 ID (같은 문서의 개정 이력 추적)
         node_id: UUID,          # 노드 ID (문서 내 섹션)
         span_offset: int | None,# 청크 내 문자 오프셋 (부분 매칭 시 사용, 단락 단위는 생략)
         content_hash: str       # SHA-256(청크 원문) (내용 변조 감지)
     }
     ```
   - 각 필드의 의미:
     - `document_id`: 근거 문서 특정
     - `version_id`: "이 인용은 v3 문서 기준입니다" 명시 → 문서 개정 후에도 원본 확인 가능
     - `node_id`: 검색 결과가 문서의 어느 섹션인지 위치 추적
     - `span_offset`: 긴 섹션(500자+)에서 "정확히 이 부분"만 인용 가능
     - `content_hash`: 청크 내용이 변하지 않았음을 검증 (해시 재계산해서 일치 확인)

2. **Citation 역참조 API**
   - `GET /api/v1/citations/{document_id}/versions/{version_id}/nodes/{node_id}/verify`
     - 요청: `{span_offset (optional), content_hash}`
     - 응답: `{verified: bool, original_text: str, modified: bool}`
     - 용도: 챗봇이 "이 citation이 여전히 유효한가?" 검증
   - `GET /api/v1/citations/{document_id}/versions/{version_id}/nodes/{node_id}/content`
     - 요청: (없음)
     - 응답: `{content: str, metadata: {...}}`
     - 용도: 사용자/에이전트가 citation을 클릭해서 원문 보기

3. **Citation 생성 로직**
   - Retriever가 청크를 반환할 때마다 자동으로 citation 5-tuple 생성
   - 청크 메타데이터에서 `document_id`, `version_id`, `node_id` 추출
   - `content_hash = SHA256(chunk.source_text)`로 계산
   - `span_offset`: 검색 결과가 청크 내 일부인 경우만 지정 (생략 가능)

4. **Citation 정확성 검증 테스트**
   - 단위 테스트: 각 청크마다 citation 생성 검증
   - 통합 테스트: 문서 수정 후 기존 citation의 `content_hash` 불일치 감지
   - 회귀 테스트: 100개 샘플 문서의 citation이 모두 검증 가능한가

5. **S1 Phase 11 호환성 유지**
   - S1 RAG 응답 포맷: `{results: [{document_id, node_id, score, snippet}]}`
   - S2 확장: 기존 필드 유지 + `citation: {document_id, version_id, node_id, span_offset, content_hash}` 추가
   - 클라이언트가 선택: citation 필드 사용 (S2+) 또는 무시 (S1 호환)
   - 마이그레이션 경로: S1 구 클라이언트는 기존대로, 신 클라이언트는 citation 활용

#### 입력 (선행 의존)
- Phase 0 완료 (설계 결정 3.4 Citation Contract 확정)
- Phase 1 완료 (PromptRegistry로 평가용 프롬프트 관리)
- S1 Phase 11 RAG 구현 분석
- S1 Phase 10 벡터화 파이프라인의 청크 메타데이터 구조 (선택)

#### 출력 (산출물)
- `Citation` 데이터 클래스 (또는 Pydantic Model)
- Citation 생성 로직 (Retriever에 통합)
- Citation 역참조 API 2개 엔드포인트
- Citation 검증 테스트 (단위/통합/회귀)
- S1 호환 마이그레이션 가이드

#### 검수 기준
- 모든 검색 결과가 citation 5-tuple 포함 여부
- `content_hash` 계산 정확성 (문서 수정 후 불일치 감지)
- Citation 역참조 API 정상 작동 (원문 조회, 검증)
- S1 호환 모드에서 기존 클라이언트 정상 작동 여부

---

### FG2.2 — Retriever/Reranker 플러그인화

#### 목적
검색 전략(FTS, Vector, Hybrid)과 재정렬(Reranking) 알고리즘을 DocumentType별로 교체 가능하게 한다.

#### 주요 작업 항목

1. **Retriever 인터페이스 정의**
   - `Retriever` (abstract base class):
     ```python
     async def retrieve(
         query: str,
         document_type: str,  # DocumentType별 전략 선택
         top_k: int = 10,
         filters: Dict = None,  # ACL, 날짜범위 등
     ) -> List[RetrievalResult]
     
     # RetrievalResult = {
     #     document_id, version_id, node_id,
     #     content, score, metadata,
     #     citation: Citation  # FG2.1에서 생성
     # }
     ```

2. **Retriever 구현체들**
   - **FTSRetriever** (Full-Text Search, S1에서 기존)
     - BM25 알고리즘 (또는 PostgreSQL의 built-in FTS)
     - 정확한 키워드 매칭에 우수
     - DocumentType별 설정: `search_weight` (제목, 본문, 메타데이터별 가중치)
   - **VectorRetriever** (Vector Similarity, Phase 10에서 pgvector 활용)
     - cosine similarity, L2 distance 등
     - 의미 기반 검색에 우수
     - DocumentType별 설정: `similarity_threshold` (최소 유사도)
   - **HybridRetriever** (FTS + Vector 통합)
     - RRF(Reciprocal Rank Fusion) 또는 weighted sum으로 통합
     - DocumentType별 설정: `fts_weight`, `vector_weight`
   - 향후 추가 가능: Semantic BM25, Colbert 등

3. **Reranker 인터페이스 정의**
   - `Reranker` (abstract base class):
     ```python
     async def rerank(
         query: str,
         candidates: List[RetrievalResult],  # Retriever 결과
         top_k: int = 10,
     ) -> List[RetrievalResult]  # 재정렬 결과
     ```
   - Reranker는 Retriever 결과를 다시 정렬하되, top_k개만 최종 반환

4. **Reranker 구현체들**
   - **CrossEncoderReranker** (학습된 재정렬 모델)
     - 패키지: `sentence-transformers` (cross-encoder 모델)
     - 용도: 상위 후보군을 더 정확히 재정렬
     - Phase 1의 LLMProvider로 실행 가능 (LLM 기반 reranking)
   - **RuleBasedReranker** (휴리스틱 규칙)
     - 문서 신선도, 조직 내 중요도 등 메타데이터 기반 스코어 조정
     - DocumentType별 규칙 정의 가능
   - **NullReranker** (재정렬 없음, 통과)
     - 빠른 응답이 필요한 경우 사용

5. **DocumentType 설정에 Retriever/Reranker 바인딩**
   - `DocumentType.retrieval_config`:
     ```json
     {
       "default_retriever": "hybrid",  // "fts" | "vector" | "hybrid"
       "retriever_params": {
         "fts_weight": 0.4,
         "vector_weight": 0.6,
         "similarity_threshold": 0.3
       },
       "default_reranker": "cross_encoder",  // "cross_encoder" | "rule_based" | null
       "reranker_params": {
         "model": "cross-encoder/ms-marco-MiniLM-L-6-v2"
       }
     }
     ```

6. **Retriever/Reranker Factory**
   - `RetrieverFactory.create(name: str, params: Dict) → Retriever`
   - `RerankerFactory.create(name: str, params: Dict) → Reranker`
   - 런타임에 설정 기반으로 인스턴스 생성

7. **검색 API 통합**
   - 기존 `GET /search/documents` API 유지
   - 파라미터 추가 (선택):
     - `retriever=hybrid|fts|vector` (기본: DocumentType 설정 값)
     - `reranker=cross_encoder|rule_based|none` (기본: DocumentType 설정 값)
   - 응답: 기존과 동일, citation 필드 추가 (FG2.1)

#### 입력 (선행 의존)
- Phase 0 완료
- Phase 1 완료 (LLM 기반 reranking을 위해)
- S1 Phase 8 FTS 구현 분석
- Phase 10 Vector 검색 구현 (또는 병행)

#### 출력 (산출물)
- `Retriever` 인터페이스 정의
- `FTSRetriever`, `VectorRetriever`, `HybridRetriever` 구현체
- `Reranker` 인터페이스 정의
- `CrossEncoderReranker`, `RuleBasedReranker`, `NullReranker` 구현체
- `RetrieverFactory`, `RerankerFactory`
- DocumentType 설정 스키마 업데이트 (retrieval_config 필드)
- 검색 API 통합 (retriever/reranker 파라미터)
- 플러그인 교체 시나리오 테스트

#### 검수 기준
- 각 Retriever 구현체 정상 작동 (FTS, Vector, Hybrid 모두)
- DocumentType별 Retriever/Reranker 설정 적용 확인
- Reranker 적용 후 검색 결과 순서 변경 확인 (품질 개선)
- Retriever/Reranker 동적 교체 정상 여부
- 검색 API의 `retriever`, `reranker` 파라미터 정상 작동

---

### FG2.3 — 쿼리 재작성 + 멀티턴 컨텍스트 압축

#### 목적
멀티턴 대화에서 follow-up 질의(예: "더 자세히 설명해줄래?")를 자립적이고 정확한 단발 쿼리로 변환하여, 검색 성공률을 높인다.

#### 주요 작업 항목

1. **쿼리 재작성 서비스**
   - `QueryRewriter` 클래스:
     ```python
     async def rewrite_query(
         original_query: str,
         conversation_history: List[Message],  # 이전 턴의 Q&A
         mode: str = "standalone"  # "standalone" | "contextual"
     ) -> str
     ```
   - 용도:
     - 원본 쿼리: "이게 뭐야?"
     - 대화 맥락: 이전에 "Kubernetes deployment"를 다뤘음
     - 재작성 결과: "Kubernetes deployment scaling 전략은 무엇인가?"
   
2. **재작성 프롬프트 (Prompt Registry 사용)**
   - 기본 프롬프트 (FG1.3에서 제공):
     ```
     당신은 검색 질의 재작성 전문가입니다.
     
     원본 질의: {original_query}
     이전 대화 맥락:
     {conversation_history}
     
     위 맥락을 고려하여, 원본 질의를 더 자립적이고 명확한 단발 검색 쿼리로 다시 작성하세요.
     검색 엔진이 이해할 수 있도록 주요 키워드를 포함하세요.
     ```
   - Prompt Registry에서 런타임에 로드 (A/B 테스트 가능)

3. **대화 이력 압축 (summarization)**
   - 길어진 대화 맥락을 요약하여 컨텍스트 윈도우 절약
   - `ConversationCompressor` 클래스:
     ```python
     async def compress(
         messages: List[Message],
         max_tokens: int = 1000
     ) -> str  # 요약된 맥락
     ```
   - 전략:
     - 슬라이딩 윈도우: 최근 N개 턴만 유지
     - 요약: LLM으로 대화 내용 압축 (Phase 1의 LLMProvider 사용)
     - 인라인 요약: 핵심 citation만 추출해서 참조

4. **Citation 캐시 및 재활용**
   - 상황: 턴 1에서 "Kubernetes" 관련 문서 검색 → citation 획득
   - 턴 2: "더 자세히 설명해줄래?" → 동일 문서 재검색 필요 없음 (citation 재사용)
   - 구현:
     - 각 턴의 검색 결과(citation)를 대화 이력에 기록
     - 재작성 시 "이전 citation 참고: doc_id=123, version_id=456" 포함
     - Retriever가 기존 citation 재사용 여부 판단

5. **멀티턴 RAG 엔드포인트**
   - 기존: `POST /api/v1/rag/answer` (단발)
   - 신규: `POST /api/v1/rag/answer` (conversation_id 포함 시 멀티턴)
   - 요청:
     ```json
     {
       "query": "이게 뭐야?",
       "conversation_id": "uuid",  // 이전 턴 명시 → 멀티턴 모드
       "top_k": 10
     }
     ```
   - 응답:
     ```json
     {
       "answer": "...",
       "citations": [...],
       "rewritten_query": "Kubernetes deployment scaling 전략은 무엇인가?",  // 재작성된 쿼리 투명성
       "context_compressed": true,  // 대화 요약 사용 여부
       "turn_number": 2
     }
     ```

6. **쿼리 재작성 품질 평가**
   - 평가 항목 (Phase 7에서 상세 구현):
     - 원본 쿼리의 정보 보존: 재작성 후 핵심 정보가 남아있는가
     - 명확성 증가: 재작성 후 검색 엔진이 더 이해하기 쉬운가
     - 맥락 통합: 대화 맥락이 적절히 포함되었는가
   - 테스트 세트: 3~5턴 멀티턴 대화 샘플 10개 (수동 검수)

#### 입력 (선행 의존)
- Phase 0, 1 완료
- FG2.1, FG2.2 완료 (Citation, Retriever)
- Phase 3 Conversation 도메인 기초 (선택, Phase 2와 병행 가능)

#### 출력 (산출물)
- `QueryRewriter` 클래스 (LLM 기반)
- `ConversationCompressor` 클래스 (요약/슬라이딩 윈도우)
- 쿼리 재작성 프롬프트 (Prompt Registry에 시드)
- 멀티턴 RAG 엔드포인트 (`POST /api/v1/rag/answer` 확장)
- Citation 캐시 로직
- 쿼리 재작성 품질 평가 (테스트 세트 포함)

#### 검수 기준
- 단발 쿼리 모드 (conversation_id 없음) 정상 작동 (기존과 동일)
- 멀티턴 쿼리 모드 (conversation_id 있음) 정상 작동
- 재작성된 쿼리가 검색 결과 개선 (정성적 평가)
- Citation 재사용 시 불필요한 재검색 회피
- 대화 요약으로 컨텍스트 토큰 감소 확인

---

## 4. 기술 설계 요약

### 4.1 Citation 흐름도
```
Retriever (e.g., HybridRetriever)
    ↓ [청크 조회]
document_chunks 테이블
    ↓ [메타데이터 추출]
{document_id, version_id, node_id, source_text}
    ↓ [Citation 생성]
Citation = {
    document_id, version_id, node_id,
    span_offset, content_hash=SHA256(source_text)
}
    ↓
검색 응답에 포함
```

### 4.2 Retriever/Reranker 플러그인 아키텍처
```
SearchService
    ├─ RetrieverFactory.create(config.retriever_type)
    │   ├─ FTSRetriever
    │   ├─ VectorRetriever
    │   └─ HybridRetriever
    │
    ├─ RerankerFactory.create(config.reranker_type)
    │   ├─ CrossEncoderReranker
    │   ├─ RuleBasedReranker
    │   └─ NullReranker
    │
    └─ [apply pipeline]
        Retriever.retrieve() → RerankerList.rerank() → 최종 결과
```

### 4.3 멀티턴 RAG 상태 관리
```
Conversation (도메인 객체, Phase 3에서 정의)
├─ Turn 1
│   ├─ query: "Kubernetes"
│   ├─ rewritten_query: (동일, 첫 턴)
│   ├─ retrieved_citations: [citation1, citation2, ...]
│   └─ answer: "..."
│
├─ Turn 2
│   ├─ query: "이게 뭐야?"
│   ├─ rewritten_query: "Kubernetes deployment scaling 전략은?"
│   ├─ retrieved_citations: (Turn 1의 citation 재사용?)
│   └─ answer: "..."
│
└─ Turn 3
    └─ ...
```

### 4.4 성능 고려사항
- Citation 생성: 청크 조회 시 인라인 (추가 지연 < 10ms)
- Reranker 적용: top 100 후보군에만 적용 (CPU 비용 절약)
- 쿼리 재작성: LLM 호출 (지연 ~1초), 폐쇄망 모델 사용 시 ~2초
- Citation 캐시: 대화 이력 메모리 보관 (메모리 사용 주의)

---

## 5. 의존 관계

### 선행 Phase
- **Phase 0**: S2 설계 결정 (Citation Contract)
- **Phase 1**: LLM, Prompt Registry (쿼리 재작성, Reranker)
- Phase 10 (선택): Vector 검색 구현 (HybridRetriever 위해)

### 후행 Phase (이 Phase의 산출물을 소비하는 Phase)
- **Phase 3 (Conversation)**: 멀티턴 RAG API, Citation, 컨텍스트 압축 활용
- **Phase 4 (Agent Interface)**: Citation 5-tuple을 MCP Tools로 노출
- **Phase 7 (평가)**: Citation 검증, 쿼리 재작성 품질 평가 (골든셋)

---

## 6. 검수 기준 종합

| 항목 | 기준 |
|------|------|
| **FG2.1 - Citation** | 모든 검색 결과가 citation 5-tuple 포함 / `content_hash` 정확성 (문서 수정 후 불일치 감지) / Citation 역참조 API 정상 / S1 호환성 유지 |
| **FG2.2 - Retriever/Reranker** | 모든 Retriever 구현체(FTS, Vector, Hybrid) 정상 / DocumentType별 설정 적용 / Reranker 적용 후 순서 변경 확인 / 동적 교체 가능 |
| **FG2.3 - 쿼리 재작성** | 단발/멀티턴 쿼리 모드 모두 정상 / 재작성 쿼리가 검색 개선 / Citation 재사용 로직 / 대화 요약 컨텍스트 감소 |
| **AI품질평가** | 검색 결과의 relevance, precision 개선 측정 (Phase 7 골든셋) / 쿼리 재작성이 검색 성공률 개선하는가 |
| **종합** | S1 Phase 11 호환성 유지 / 챗봇이 Citation을 신뢰하고 사용 가능한가 |

---

## 7. 예상 산출물 목록

| 산출물 | 형태 | 설명 |
|--------|------|------|
| Citation 데이터 모델 | Pydantic Model | 5-tuple 정의 |
| Citation 생성 로직 | Python | Retriever에 통합 |
| Citation 역참조 API | FastAPI endpoints | 2개 엔드포인트 |
| Retriever 인터페이스 | Python (abc.ABC) | 추상 베이스 클래스 |
| FTSRetriever / VectorRetriever / HybridRetriever | Python | 구현체 3개 |
| Reranker 인터페이스 | Python (abc.ABC) | 추상 베이스 클래스 |
| CrossEncoderReranker / RuleBasedReranker / NullReranker | Python | 구현체 3개 |
| RetrieverFactory / RerankerFactory | Python | 팩토리 클래스 |
| DocumentType retrieval_config 스키마 | JSON Schema | 설정 정의 |
| 검색 API 통합 | FastAPI endpoint | `GET /search/documents` 확장 |
| QueryRewriter | Python | LLM 기반 쿼리 재작성 |
| ConversationCompressor | Python | 대화 요약/압축 |
| 멀티턴 RAG 엔드포인트 | FastAPI endpoint | `POST /api/v1/rag/answer` 확장 |
| Citation 캐시 로직 | Python | 대화 이력에 citation 기록 |
| 단위/통합/회귀 테스트 | pytest | Citation, Retriever, Reranker, 쿼리 재작성 |
| 쿼리 재작성 프롬프트 | JSON (Prompt Registry) | 기본 템플릿 |
| S1 호환 마이그레이션 가이드 | Markdown | 기존 클라이언트 지원 |
| **FG2.1 검수보고서** | Markdown | Citation 검수 |
| **FG2.1 보안취약점검사보고서** | Markdown | Citation 검증 보안 (hash collision, 위조 등) |
| **FG2.2 검수보고서** | Markdown | Retriever/Reranker 검수 |
| **FG2.2 보안취약점검사보고서** | Markdown | DocumentType 설정 injection 방어 |
| **FG2.3 검수보고서** | Markdown | 쿼리 재작성, 컨텍스트 압축 |
| **FG2.3 보안취약점검사보고서** | Markdown | 프롬프트 injection (쿼리 재작성) 방어 |
| **FG2.3 AI품질평가** | Markdown | 쿼리 재작성 품질, 검색 개선도 |
| **챗봇 연동 인계 메모 v2** | Markdown | Citation 계약 반영 (특별 산출물) |
| **Phase 2 종결 보고서** | Markdown | Phase 2 완료 보고 |

---

## 8. 리스크 및 대응

| 리스크 | 영향도 | 대응 방안 |
|--------|--------|----------|
| S1 호환성 깨짐 (기존 클라이언트 오류) | 높음 | citation 필드 선택적 추가, 기본값 설정 |
| Reranker 지연 증가 (LLM 호출) | 중간 | top-k 후보군만 재정렬, cross-encoder 모델 경량화 |
| 쿼리 재작성 잘못 (LLM 판정 오류) | 중간 | 기본 프롬프트 A/B 테스트, 폴백 (원본 쿼리 사용) |
| Content hash 충돌 (매우 낮은 확률) | 낮음 | SHA-256 사용 (충돌 확률 무시), monitoring |
| Citation 캐시 메모리 증가 | 중간 | 슬라이딩 윈도우(최근 10턴만), 주기적 정리 |
| HybridRetriever 성능 (FTS+Vector 모두 실행) | 중간 | 병렬 실행, 캐싱 활용, top-k 제한 |

---

## 9. 선행 조건

Phase 2를 시작하려면:
- Phase 0, 1 완료
- S1 Phase 11 RAG 구현 상세 분석
- 기본 검색 쿼리 셋 확보 (테스트/평가용)
- S1 Phase 10 벡터화 준비 상태 (또는 병행 가능)

---

## 10. 완료 기준

Phase 2 완료로 판단하는 조건:

1. 모든 검색 결과가 citation 5-tuple 필수 포함
2. `content_hash` 정확성 검증 (문서 수정 시 불일치 감지)
3. Citation 역참조 API (`/verify`, `/content`) 정상 작동
4. S1 Phase 11 호환 모드에서 기존 클라이언트 정상 작동
5. FTSRetriever, VectorRetriever, HybridRetriever 모두 정상 작동
6. DocumentType별 Retriever/Reranker 설정 적용 확인
7. Reranker 적용 후 검색 결과 순서 변경 (정성적 개선 확인)
8. QueryRewriter 정상 작동 (멀티턴 쿼리 재작성)
9. ConversationCompressor 정상 작동 (대화 요약)
10. 멀티턴 RAG 엔드포인트 정상 작동 (conversation_id 파라미터)
11. Citation 캐시 메모리 사용량 모니터링 기록
12. 단위/통합 테스트 커버리지 ≥85%
13. 모든 FG 검수보고서 및 보안취약점검사보고서 승인
14. FG2.3 AI품질평가 완료 (쿼리 재작성 품질 기록)
15. 챗봇 인계 메모 v2 작성 완료 (Citation 계약 반영)
16. Phase 2 종결 보고서 작성 완료

---

## 11. 권장 투입 순서

1. FG2.1 Citation 5-tuple 계약 고정
   - Citation 데이터 모델 정의
   - Citation 생성 로직 구현
   - Citation 역참조 API 구현
   - S1 호환성 테스트
2. FG2.2 Retriever/Reranker 플러그인화
   - Retriever 인터페이스 및 구현체 (FTS, Vector, Hybrid)
   - Reranker 인터페이스 및 구현체 (CrossEncoder, RuleBased, Null)
   - DocumentType 설정 통합
   - 검색 API 확장
3. FG2.3 쿼리 재작성 + 컨텍스트 압축
   - QueryRewriter 구현
   - ConversationCompressor 구현
   - 멀티턴 RAG 엔드포인트 구현
   - Citation 캐시 로직
4. 통합 테스트 및 성능 검증
5. AI품질평가 (쿼리 재작성 품질)
6. 모든 검수/보안 검사
7. 챗봇 인계 메모 v2 작성
8. Phase 2 종결 보고서

---

## 12. 기대 효과

Phase 2 완료 시:
- 검색 결과가 "검증 가능한 근거"로 표준화 → 챗봇/에이전트가 citation 신뢰 가능
- Retriever/Reranker 교체로 검색 품질 지속적 개선 가능
- 멀티턴 대화 지원으로 챗봇 사용성 대폭 향상
- 쿼리 재작성으로 follow-up 질의 성공률 증가
- 챗봇 S2-2 연동의 데이터 계약 확정 (Citation, 멀티턴 API)
