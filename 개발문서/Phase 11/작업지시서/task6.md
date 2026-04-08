# Task 11-6. ContextBuilder 구현

## 1. 작업 목적

Reranker가 선택한 청크들을 LLM에 전달할 프롬프트로 조합하는 ContextBuilder를 구현한다.

이 작업의 목표는 다음과 같다.

- 청크 → LLM 시스템 프롬프트 조합
- 토큰 한도 관리 (LLM 컨텍스트 윈도우 초과 방지)
- Citation 마킹 지시 포함 프롬프트 설계
- DocumentType별 프롬프트 커스터마이징 지원 (Phase 12 준비)

---

## 2. 작업 범위

### 포함 범위

- ContextBuilder 클래스 구현
- 시스템 프롬프트 템플릿 설계
- 청크 조합 및 토큰 한도 관리
- Citation 마킹 지시 포함
- DocumentType별 PromptTemplate 구조 준비

### 제외 범위

- LLM 호출 자체 (Task 11-5에서 다룸)
- Citation 파싱 (Task 11-7에서 다룸)

---

## 3. 선행 조건

- Task 11-4 (Reranker) 완료 — RetrievedChunk[] 입력
- Task 11-5 (LLMProvider) 완료 — LLMPrompt 구조 확정

---

## 4. 주요 구현 대상

### 4-1. ContextBuilder 인터페이스

```
ContextBuilder
  .build(query: str, chunks: RetrievedChunk[], config: ContextConfig) → LLMPrompt

ContextConfig
  max_context_tokens: int = 6000     (컨텍스트 최대 토큰)
  max_response_tokens: int = 2048    (응답 최대 토큰)
  include_citations: bool = True
  language: str = "ko"
  document_type: str | None = None   (타입별 프롬프트 분기)
```

---

### 4-2. 시스템 프롬프트 기본 템플릿

```
당신은 문서 기반 질의응답 AI입니다.
아래에 제공된 문서 컨텍스트만을 바탕으로 질문에 답하세요.

규칙:
1. 제공된 컨텍스트 외의 정보를 추가하지 마세요.
2. 답변할 근거가 없으면 "제공된 문서에서 해당 정보를 찾을 수 없습니다."라고 답하세요.
3. 각 주장의 출처를 [1], [2] 형식으로 표시하세요.
4. 한국어로 답변하세요.

=== 문서 컨텍스트 ===
[1] {문서제목} - {노드경로}
{context_text}

[2] {문서제목} - {노드경로}
{context_text}
...
```

---

### 4-3. 청크 조합 및 토큰 관리

```python
def build(self, query: str, chunks: list[RetrievedChunk], config: ContextConfig) -> LLMPrompt:
    # 1. 각 청크의 토큰 수 추정
    # 2. 최대 컨텍스트 토큰 한도 내에서 청크 선택
    # 3. 선택된 청크를 번호 붙여 시스템 프롬프트에 조합
    # 4. LLMPrompt 반환

    selected_chunks = []
    total_tokens = _estimate_system_prompt_tokens()

    for chunk in chunks:
        chunk_tokens = _estimate_tokens(chunk.context_text)
        if total_tokens + chunk_tokens > config.max_context_tokens:
            break
        selected_chunks.append(chunk)
        total_tokens += chunk_tokens

    system_prompt = _render_system_prompt(selected_chunks, config)

    return LLMPrompt(
        system=system_prompt,
        messages=[LLMMessage(role="user", content=query)],
        max_tokens=config.max_response_tokens,
    )
```

---

### 4-4. Citation 마킹 지시

청크 번호(`[1]`, `[2]`)와 청크 mapping 정보를 함께 반환하여 CitationLinker가 매핑에 사용:

```python
@dataclass
class BuiltContext:
    prompt: LLMPrompt
    chunk_mapping: dict[int, RetrievedChunk]  # 번호 → 청크 매핑
    selected_chunks: list[RetrievedChunk]
```

---

### 4-5. PromptTemplate 추상화 (Phase 12 준비)

DocumentType별로 프롬프트를 커스터마이징할 수 있는 구조 준비:

```
PromptTemplate (인터페이스)
  .render_system(chunks, config) → str

DefaultPromptTemplate (기본 구현)
PolicyPromptTemplate (POLICY 타입용, Phase 12에서 추가)
ManualPromptTemplate (MANUAL 타입용, Phase 12에서 추가)

팩토리:
  get_prompt_template(document_type: str | None) → PromptTemplate
```

---

### 4-6. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| 청크 5개 조합 | system에 [1]~[5] 포함 |
| 토큰 한도 초과 | 한도 내 청크만 선택 |
| 청크 없음 | "컨텍스트 없음" 시스템 프롬프트 |
| chunk_mapping | 번호와 청크가 정확히 매핑 |

---

## 5. 산출물

1. ContextBuilder 구현 코드
2. BuiltContext 데이터 클래스
3. PromptTemplate 인터페이스 및 DefaultPromptTemplate
4. 단위 테스트

---

## 6. 완료 기준

- ContextBuilder가 청크를 LLMPrompt로 조합한다
- 토큰 한도 초과 시 초과 청크를 제외한다
- chunk_mapping으로 번호와 청크가 매핑된다
- PromptTemplate이 교체 가능한 구조다
- 단위 테스트가 통과한다

---

## 7. Codex 작업 지침

- 시스템 프롬프트 템플릿은 하드코딩 금지 — 파일이나 설정에서 로드하는 구조로 설계한다
- 토큰 추정은 Phase 10 ChunkingService에서 사용한 tiktoken 유틸리티를 재사용한다
- 컨텍스트 청크 번호(`[1]`, `[2]`)는 rerank_score 내림차순으로 정렬하여 가장 관련성 높은 청크가 [1]이 되도록 한다
