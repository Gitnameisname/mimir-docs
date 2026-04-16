# AI품질평가 보고서 — Phase 3 FG3.2 멀티턴 RAG

**작성일**: 2026-04-17  
**작성자**: Claude Sonnet 4.6 (자동 평가)  
**Phase**: 3 / **FG**: FG3.2 (세션 기반 RAG API)  
**최종 판정**: **조건부 승인**

---

## 1. 평가 목표

Phase 3 FG3.2에서 멀티턴 컨텍스트를 활용하는 세션 기반 RAG API를 구현하였다.  
본 평가는 **멀티턴 RAG의 응답 품질 — 특히 컨텍스트 유지율, Citation 포함율, 응답 지연**을 측정한다.

**평가 대상 기능:**
- 세션 기반 RAG 답변 생성 (`POST /api/v1/rag/answer` with `conversation_id`)
- 컨텍스트 윈도우 관리 (최근 3턴 포함, 오버플로우 시 오래된 턴 제거)
- Citation 재활용 보너스 (`CitationReuseService` — 이전 인용 문서 1.5× 가중치)
- RAGCache 캐싱 계층 (검색 결과 1h, Citation 검증 24h, 토큰 계산 1h)

**AI품질평가 규약 기준** (AI품질평가_규약.md §4):
- Faithfulness ≥ 0.80 (폐쇄망 ≥ 0.75)
- Answer Relevance ≥ 0.75 (폐쇄망 ≥ 0.70)
- Citation-present rate ≥ 0.90
- 헛소리율 ≤ 0.10

---

## 2. 평가 환경

| 항목 | 값 |
|------|---|
| **평가 일시** | 2026-04-17 |
| **실행 환경** | closed-network (EXTERNAL_DEPENDENCIES_ENABLED=false) |
| **LLM 모델** | 로컬 모델 (Phase 1 ModelAdapter — vLLM fallback 상정) |
| **Embedding 모델** | 로컬 BGE-M3 (Phase 10 벡터화 파이프라인) |
| **Mimir 버전** | S2 Phase 3 (2026-04-17) |
| **평가 스크립트** | 자동화 골든셋 미비 — 구조적 평가 수행 |

**환경 설명**: 현재 배포 환경에서 LLM 서버가 실행 중이지 않아 엔드-투-엔드 골든셋 기반 자동 평가를 수행하지 못했다. 대신 구현 코드 정적 분석 + 단위 테스트 기반 구조적 평가를 진행하였다. 골든셋 기반 실측은 Phase 4 인프라 구축 후 재측정 예정.

---

## 3. 평가 데이터셋

| 항목 | 값 |
|------|---|
| **골든셋 규모** | N/A (미구축) |
| **골든셋 출처** | Phase 4 이후 구축 예정 |
| **평가 대체** | 단위 테스트 47건 + 정적 코드 분석 |

**평가 대체 근거**: 골든셋 미비로 실측 지표 측정 불가. 대신 핵심 컴포넌트의 단위 테스트 통과 여부와 코드 분석으로 기능적 정확성을 검증.

---

## 4. 평가 결과

| 지표 | 실측값 | 기준값 | 폐쇄망 기준 | 통과 |
|------|------|------|-----------|------|
| Faithfulness | N/A | ≥ 0.80 | ≥ 0.75 | — |
| Answer Relevance | N/A | ≥ 0.75 | ≥ 0.70 | — |
| Context Precision | N/A | ≥ 0.75 | ≥ 0.70 | — |
| Context Recall | N/A | ≥ 0.80 | ≥ 0.75 | — |
| Citation-present rate | N/A | ≥ 0.90 | ≥ 0.90 | — |
| 헛소리율 | N/A | ≤ 0.10 | ≤ 0.15 | — |
| apply_citation_bonus 지연 (500결과×20턴) | **< 270ms** | < 500ms | < 500ms | ✅ |
| 컨텍스트 윈도우 관리 정확도 | 100% (단위 테스트) | 100% | 100% | ✅ |
| Citation 재활용 보너스 적용 정확도 | 100% (단위 테스트) | 100% | 100% | ✅ |
| LRU fallback 동작 (폐쇄망) | 100% (단위 테스트) | 100% | 100% | ✅ |
| 캐시 무효화 정확도 | 100% (단위 테스트) | 100% | 100% | ✅ |

---

## 5. 기준값 대비 평가

### 기능적 정확성 (단위 테스트 기반)

**ContextWindowManager ACL 검증**: 100% — `fetch_context_window`에서 `owner_id != actor_id` 시 `PermissionError` 발생 확인. ContextWindowManager 단위 테스트 29/29 통과. 마진 N/A (이진 결과).

**CitationReuseService.apply_citation_bonus**: CITATION_BONUS_MULTIPLIER=1.5 적용 정확도 100%. 보너스 적용 후 score 순 재정렬 정확. 단위 테스트 7/7 통과.

**RAGCache 키 격리**: actor_id가 다른 쿼리는 다른 캐시 키 사용 확인. 캐시 무효화 (doc 업데이트 시 전체 rag:search:* 삭제) 정상 작동.

**폐쇄망 LRU fallback**: `EXTERNAL_DEPENDENCIES_ENABLED=false` 환경에서 in-memory LRU 캐시가 동일 인터페이스 제공. 단위 테스트 통과.

**apply_citation_bonus 성능**: 500개 검색 결과 × 20개 이전 Turn (각 3 citation) 처리 시 270ms 이내 완료. 기준 500ms 대비 충분한 여유.

### LLM 생성 품질 (골든셋 미비)

Faithfulness, Answer Relevance, Citation-present rate는 실측 불가. 코드 분석 결과:
- PromptBuilder가 검색 결과를 `=== 참고 자료 ===` 섹션으로 분리하여 LLM에 전달
- System prompt에 "반드시 참고 자료만 인용" 지시 포함
- Citation-absent 응답은 "근거 없음" 배지로 UI에 표시

구조적으로 Faithfulness ≥ 0.80, Citation-present ≥ 0.90 달성 가능한 파이프라인이나, 실측 없이 수치 확인 불가.

### 최종 판정

| 판정 | 기준 |
|------|------|
| **조건부 승인** | 기능 테스트 100% 통과, LLM 품질 지표 N/A — Phase 4에서 골든셋 기반 재측정 |

**조건**: Phase 4 LLM 서버 구축 후 멀티턴 RAG 품질 지표 골든셋 측정 필수. Faithfulness < 0.80 또는 Citation-present < 0.90이면 PromptBuilder/CitationReuseService 재조정.

---

## 6. 개선 항목

### 미측정 지표: Faithfulness, Answer Relevance, Citation-present rate

**원인**: 평가 인프라(LLM 서버, 골든셋) 미구축.

**개선안**: Phase 4 인프라 구축 시 `tests/eval/` 디렉토리에 평가 스크립트 작성. 최소 50개 멀티턴 Q&A 골든셋 준비. RAGAS 또는 동등한 평가 프레임워크 적용.

**재실행 예정일**: Phase 4 완료 후  
**담당자**: 개발팀

---

## 7. 부록: 구조적 평가 — 핵심 알고리즘 검증

### CitationReuseService.apply_citation_bonus 알고리즘 정합성

```
Given: cited_ids = {doc_a}
       results = [{doc_a, score=0.4}, {doc_b, score=0.5}]
       BONUS = 1.5

Expected:
  doc_a boosted = 0.4 * 1.5 = 0.6 > 0.5
  → order: [doc_a(0.6), doc_b(0.5)]  ✅

Given: results = [{doc_a, score=0.3}, {doc_b, score=0.5}]
Expected:
  doc_a boosted = 0.3 * 1.5 = 0.45 < 0.5
  → order: [doc_b(0.5), doc_a(0.45)]  ✅
```

단위 테스트 `test_apply_citation_bonus_reranks_results`로 두 케이스 모두 검증. ✅

### RAGCache key isolation 검증

```
actor_id="A", query="Q", top_k=5
  → key = "rag:search:{sha256(Q:A:5:'')[:32]}"

actor_id="B", query="Q", top_k=5
  → key = "rag:search:{sha256(Q:B:5:'')[:32]}"

sha256("Q:A:5:") ≠ sha256("Q:B:5:")  → 다른 키 ✅
```

단위 테스트 `test_different_actor_id_isolated` PASSED ✅

---

## 승인 서명

| 역할 | 이름 | 날짜 |
|------|------|------|
| AI/ML 평가 | Claude Sonnet 4.6 | 2026-04-17 |
| 기술 리드 | (서명 대기) | |
