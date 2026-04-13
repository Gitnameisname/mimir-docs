# Task 12-6. 타입별 RAG 플러그인 (Phase 11 통합)

## 1. 작업 목적

Phase 11의 RAG 파이프라인이 DocumentType별 프롬프트 템플릿 및 컨텍스트 설정을 플러그인 레지스트리를 통해 읽도록 통합한다.

이 작업의 목표는 다음과 같다.

- RAGPlugin 인터페이스 상세 구현
- 내장 타입별 PromptTemplate 구현
- Phase 11 RAGQueryService를 DocumentTypeRegistry 경유로 마이그레이션
- 타입별 최적화된 RAG 설정 적용

---

## 2. 작업 범위

### 포함 범위

- RAGPlugin 인터페이스 구현
- 내장 타입별 PromptTemplate 구현
- Phase 11 ContextBuilder 통합 수정
- 타입별 RAG 동작 테스트

### 제외 범위

- LLM 자체 수정 (Phase 11에서 완성됨)
- Retriever/Reranker 수정 (Phase 11에서 완성됨)

---

## 3. 선행 조건

- Task 12-2 (DocumentTypePlugin 인터페이스) 완료
- Phase 11 Task 11-6 (ContextBuilder, PromptTemplate) 코드 이해

---

## 4. 주요 구현 대상

### 4-1. RAGPlugin 인터페이스

```python
class RAGPlugin:
    def get_prompt_template(self) -> PromptTemplate:
        return DefaultPromptTemplate()

    def get_context_config(self) -> ContextConfig:
        return ContextConfig()

    def get_reranker_config(self) -> dict:
        return {"enabled": True, "top_n": 5}
```

---

### 4-2. 내장 타입별 PromptTemplate

#### PolicyPromptTemplate

```python
class PolicyPromptTemplate(PromptTemplate):
    def render_system(self, chunks: list[RetrievedChunk], config: ContextConfig) -> str:
        return f"""
당신은 정책/규정 문서 전문 AI입니다.
아래 정책 문서 조항을 바탕으로 질문에 답하세요.

규칙:
1. 정책 조항의 원문을 가능한 한 그대로 인용하세요.
2. 조항 번호를 함께 언급하세요 (예: 제3조에 따르면...).
3. 모호한 경우 "정책 원문을 직접 확인하시기 바랍니다"라고 안내하세요.
4. 출처를 [1], [2] 형식으로 표시하세요.

=== 정책 조항 ===
{self._render_chunks(chunks)}
""".strip()
```

#### ManualPromptTemplate

```python
class ManualPromptTemplate(PromptTemplate):
    def render_system(self, chunks, config):
        return f"""
당신은 절차/매뉴얼 문서 전문 AI입니다.
아래 매뉴얼 내용을 바탕으로 절차를 단계별로 설명하세요.

규칙:
1. 절차는 번호가 있는 목록 형식으로 답변하세요.
2. 각 단계에서 주의사항이 있으면 강조하세요.
3. 출처를 [1], [2] 형식으로 표시하세요.

=== 매뉴얼 내용 ===
{self._render_chunks(chunks)}
""".strip()
```

#### FAQPromptTemplate

```python
class FAQPromptTemplate(PromptTemplate):
    def render_system(self, chunks, config):
        return f"""
당신은 FAQ 문서를 바탕으로 질문에 답하는 AI입니다.
관련 Q&A 항목을 참조하여 명확하고 친절하게 답변하세요.

=== 관련 FAQ ===
{self._render_chunks(chunks)}
""".strip()
```

---

### 4-3. 타입별 ContextConfig 커스터마이징

| 타입 | max_context_tokens | top_n | 특이사항 |
|------|-------------------|-------|---------|
| POLICY | 8000 | 7 | 조항 원문 보존을 위해 긴 컨텍스트 |
| MANUAL | 5000 | 5 | 기본값 |
| REPORT | 6000 | 5 | 기본값 |
| FAQ | 3000 | 10 | 짧은 Q&A 많이 포함 |

---

### 4-4. Phase 11 ContextBuilder 마이그레이션

기존:

```python
# Phase 11 (기존)
def get_prompt_template(document_type: str | None) -> PromptTemplate:
    if document_type == "POLICY":
        return PolicyPromptTemplate()
    return DefaultPromptTemplate()
```

마이그레이션 후:

```python
# Phase 12 (플러그인 경유)
def get_prompt_template(document_type: str | None) -> PromptTemplate:
    if document_type:
        plugin = DocumentTypeRegistry.instance().get(document_type)
        return plugin.rag_plugin().get_prompt_template()
    return DefaultPromptTemplate()
```

---

### 4-5. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| POLICY RAGPlugin | PolicyPromptTemplate 반환 |
| FAQ ContextConfig | top_n=10 |
| 미등록 타입 | DefaultPromptTemplate 반환 |
| 프롬프트 렌더링 | 조항 원문 인용 지시 포함 |

---

## 5. 산출물

1. RAGPlugin 인터페이스 구현
2. 내장 타입별 PromptTemplate 구현 (POLICY, MANUAL, REPORT, FAQ)
3. Phase 11 ContextBuilder 마이그레이션 코드
4. 단위 테스트

---

## 6. 완료 기준

- RAGPlugin이 타입별 PromptTemplate과 ContextConfig를 반환한다
- ContextBuilder가 DocumentTypeRegistry 경유로 프롬프트 설정을 로드한다
- 내장 4개 타입의 RAG 설정이 정의되어 있다
- 단위 테스트가 통과한다

---

## 7. Codex 작업 지침

- 타입별 프롬프트 템플릿은 "해당 문서 유형의 특성에 맞는 응답 스타일"을 지시하는 내용을 포함한다
- PromptTemplate 내용은 하드코딩하되, 향후 Admin UI에서 편집 가능하도록 DB 오버라이드 경로를 설계한다
- POLICY 타입의 경우 법적/규정 문서이므로 원문 인용을 강조하는 프롬프트가 중요하다
