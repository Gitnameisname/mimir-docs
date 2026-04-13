# Task 11-11. 성능 검증 및 Phase 12 연계 포인트 설계

## 1. 작업 목적

Phase 11 RAG 파이프라인의 성능과 응답 품질을 검증하고, Phase 12 DocumentType 플러그인이 RAG를 확장할 수 있는 연계 포인트를 설계한다.

이 작업의 목표는 다음과 같다.

- RAG 파이프라인 End-to-End 성능 측정
- 응답 품질 평가 (Faithfulness, Relevancy)
- LLM 비용 모니터링 현황 검토
- Phase 12 DocumentType 플러그인 RAG 확장 포인트 정의

---

## 2. 작업 범위

### 포함 범위

- E2E 성능 측정 (latency 목표 달성 확인)
- 응답 품질 평가 지표 및 방법론
- LLM / 임베딩 비용 집계 보고
- Phase 12 RAG 확장 포인트 설계
- Admin RAG 모니터링 API 구현

### 제외 범위

- DocumentType 플러그인 구현 (Phase 12에서 다룸)
- 인프라 스케일링 (Phase 13에서 다룸)

---

## 3. 선행 조건

- Task 11-1~11-10 모두 완료
- Phase 10 Task 10-11 (벡터 검색 성능 검증) 결과 참조

---

## 4. 주요 구현 대상

### 4-1. E2E 성능 목표

| 지표 | 목표값 | 측정 방법 |
|------|--------|---------|
| RAG 응답 첫 토큰 도달 (TTFT) | < 2초 | 100건 질의 실행 |
| 전체 응답 완료 (P95) | < 15초 | 100건 질의 실행 |
| Retriever 단독 latency | < 500ms | Phase 10 기준 |
| Reranker 단독 latency | < 300ms | 20개 청크 기준 |

---

### 4-2. 응답 품질 평가

#### 평가 지표

| 지표 | 설명 | 측정 방법 |
|------|------|---------|
| Faithfulness | 응답이 컨텍스트 내용에 충실한 비율 | LLM 기반 평가 |
| Answer Relevancy | 응답이 질문에 적절한 비율 | LLM 기반 평가 |
| Citation Accuracy | 출처 번호가 올바른 청크를 참조하는 비율 | 자동 검증 |
| Context Recall | 정답에 필요한 청크가 검색된 비율 | 평가 데이터셋 기반 |

#### 평가 데이터셋

- 질문 50건 이상 (DocumentType별 균형 배분)
- 각 질문에 대한 기대 답변 및 근거 문서 레이블
- 평가 도구: RAGAS 라이브러리 (선택) 또는 자체 LLM 판정

---

### 4-3. LLM 비용 집계 Admin API

```
GET /api/admin/rag/costs?period=daily|weekly|monthly

Response:
{
  "period": "daily",
  "items": [
    {
      "date": "2026-04-08",
      "llm_prompt_tokens": 1234567,
      "llm_completion_tokens": 234567,
      "llm_estimated_cost_usd": 8.23,
      "embedding_tokens": 345678,
      "embedding_estimated_cost_usd": 0.007,
      "total_estimated_cost_usd": 8.237,
      "query_count": 1234
    }
  ]
}
```

---

### 4-4. Phase 12 RAG 확장 포인트

Phase 12 DocumentType 플러그인이 RAG를 확장할 수 있는 인터페이스:

#### 확장 포인트 목록

| 확장 포인트 | 인터페이스 | 설명 |
|------------|----------|------|
| PromptTemplate | `get_prompt_template(doc_type)` | 타입별 시스템 프롬프트 커스터마이징 |
| Reranker | `get_reranker(doc_type)` | 타입별 재랭킹 전략 |
| ContextConfig | `get_context_config(doc_type)` | 타입별 max_tokens, top_n 등 |
| CitationFormat | `format_citation(chunk)` | 타입별 출처 표시 형식 |

#### 기본 구현 (Phase 11)

모든 확장 포인트는 Phase 11에서 기본 구현체를 제공하며, Phase 12에서 타입별 구현체로 오버라이드 가능:

```python
class DocumentTypeRAGPlugin:
    def get_prompt_template(self) -> PromptTemplate:
        return DefaultPromptTemplate()

    def get_reranker(self) -> Reranker:
        return get_reranker()

    def get_context_config(self) -> ContextConfig:
        return ContextConfig()
```

---

### 4-5. 검증 보고서 항목

| 항목 | 내용 |
|------|------|
| 측정 환경 | 서버 사양, 데이터 규모, 모델 버전 |
| TTFT / 전체 응답 latency | P50, P95 |
| 응답 품질 점수 | Faithfulness, Relevancy, Citation Accuracy |
| LLM 비용 예측 | 일 평균 질의 수 기반 월 예상 비용 |
| Phase 12 준비 완료 확인 | 확장 포인트 인터페이스 확정 여부 |
| 미해결 이슈 | 목표 미달 지표 및 개선 방안 |

---

## 5. 산출물

1. E2E 성능 측정 결과 보고서
2. 응답 품질 평가 결과
3. LLM 비용 집계 Admin API
4. Phase 12 RAG 확장 포인트 인터페이스 정의
5. DocumentTypeRAGPlugin 기본 구현

---

## 6. 완료 기준

- TTFT P95가 2초 이하를 달성하거나, 미달 시 원인과 대안이 문서화된다
- Faithfulness 및 Answer Relevancy 점수가 0.8 이상을 목표로 한다
- LLM 비용 집계 Admin API가 동작한다
- Phase 12 확장 포인트 인터페이스가 정의되어 있다

---

## 7. Codex 작업 지침

- 성능 목표 미달 시 즉각 튜닝보다 원인 분석을 우선하여 보고서에 기록한다
- Phase 12 확장 포인트는 문서 타입을 하드코딩하지 않고 플러그인 등록 방식으로 설계한다 (CLAUDE.md 원칙 준수)
- 비용 모니터링은 LLMUsageLog와 EmbeddingUsageLog 두 테이블을 조인하여 집계한다
- 응답 품질 평가는 자동화 가능한 지표(Citation Accuracy)부터 측정하고, Faithfulness 등 LLM 판정 지표는 샘플 기반으로 측정한다
