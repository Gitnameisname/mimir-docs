# Task 10-11. 성능 검증 및 Phase 11 연계 포인트 설계

## 1. 작업 목적

Phase 10 벡터화 파이프라인의 성능을 검증하고, Phase 11 RAG 파이프라인이 활용할 수 있는 인터페이스를 확정한다.

이 작업의 목표는 다음과 같다.

- 벡터 검색 응답 시간 측정 및 목표 달성 확인
- HNSW 인덱스 파라미터 튜닝 (ef_search)
- 하이브리드 검색 품질 평가
- Phase 11 RAG 연계 인터페이스 설계

---

## 2. 작업 범위

### 포함 범위

- 벡터 검색 성능 측정 (latency, recall@k)
- HNSW ef_search 튜닝
- 하이브리드 검색 품질 평가 지표 설계
- Phase 11 연계 인터페이스 정의

### 제외 범위

- RAG 파이프라인 구현 (Phase 11에서 다룸)
- 인프라 스케일링 (운영 환경 이슈)

---

## 3. 선행 조건

- Task 10-6 (벡터화 파이프라인) 완료 — document_chunks 데이터 존재
- Task 10-9 (하이브리드 검색) 완료 — VectorSearchProvider, HybridSearchProvider 동작
- 테스트용 문서 데이터셋 준비

---

## 4. 주요 구현 대상

### 4-1. 성능 목표

| 지표 | 목표값 | 측정 방법 |
|------|--------|---------|
| 벡터 검색 latency (P95) | < 200ms | 1000건 쿼리 실행 |
| 하이브리드 검색 latency (P95) | < 500ms | 1000건 쿼리 실행 |
| Recall@10 | > 0.8 | 평가 데이터셋 기반 |
| 동시 검색 처리 | 50 RPS | 부하 테스트 |

---

### 4-2. HNSW ef_search 튜닝

Task 10-3 기본값(ef_search=40)을 기준으로 성능/정확도 트레이드오프 측정:

| ef_search | 예상 latency | 예상 recall |
|-----------|------------|-----------|
| 20 | 빠름 | 낮음 |
| 40 | 중간 (기본) | 중간 |
| 80 | 느림 | 높음 |

측정 후 목표값을 달성하는 최소 ef_search 값을 기본값으로 확정:

```sql
-- 쿼리 레벨에서 ef_search 설정
SET hnsw.ef_search = 40;
```

---

### 4-3. 검색 품질 평가

#### 평가 데이터셋 구성

- 테스트 쿼리 100건 이상
- 각 쿼리에 대한 정답 문서/청크 레이블
- 다양한 DocumentType 포함

#### 평가 지표

| 지표 | 설명 |
|------|------|
| Precision@k | 상위 k개 중 정답 비율 |
| Recall@k | 전체 정답 중 상위 k개에 포함된 비율 |
| MRR (Mean Reciprocal Rank) | 정답이 처음 등장하는 순위의 역수 평균 |

#### 비교 대상

| 모드 | 설명 |
|------|------|
| keyword only | Phase 8 FTS 기준선 |
| semantic only | 벡터 검색만 |
| hybrid (RRF) | FTS + 벡터 병합 |

---

### 4-4. Phase 11 RAG 연계 인터페이스

Phase 11 RAG 파이프라인이 사용할 검색 인터페이스 확정:

#### RAGContextProvider 인터페이스

```
RAGContextProvider
  .retrieve(query: RAGQuery) → RAGContext

RAGQuery
  text: str               (사용자 질문)
  user_roles: list[str]
  user_org_ids: list[str]
  document_type: str | None
  top_k: int = 5          (컨텍스트 청크 수)
  mode: str = "hybrid"

RAGContext
  chunks: RetrievedChunk[]
  total_tokens: int
  search_mode: str

RetrievedChunk
  chunk_id: str
  document_id: str
  document_title: str
  node_path: list[str]
  context_text: str       (임베딩에 사용된 부모 컨텍스트 포함 텍스트)
  similarity_score: float
  fts_score: float | None
  hybrid_score: float
```

---

### 4-5. 청크 컨텍스트 조합 전략 (Phase 11 준비)

RAG에서 LLM에 전달할 컨텍스트 구성 시 주의사항:

- `context_text`를 LLM 프롬프트에 직접 전달 (부모 컨텍스트 포함)
- 상위 k개 청크의 총 토큰 수가 LLM 컨텍스트 한도를 초과하지 않도록 조정
- 동일 문서에서 연속된 청크는 병합하여 중복 컨텍스트 제거
- 출처 정보 (`document_title`, `node_path`) 를 LLM 응답 인용에 활용

---

### 4-6. 성능 검증 보고서 항목

| 항목 | 내용 |
|------|------|
| 측정 환경 | DB 서버 사양, 데이터 규모 |
| HNSW 최종 파라미터 | m, ef_construction, ef_search |
| 검색 latency 결과 | P50, P95, P99 |
| 검색 품질 결과 | Precision@5, Recall@10, MRR |
| 권장 설정 | ef_search 최종값, 기본 검색 모드 |
| Phase 11 준비 완료 | RAGContextProvider 인터페이스 확정 여부 |

---

## 5. 산출물

1. 벡터 검색 성능 측정 결과 보고서
2. HNSW 파라미터 최종값 결정 문서
3. 하이브리드 검색 품질 평가 결과
4. RAGContextProvider 인터페이스 정의
5. Phase 11 연계 포인트 문서

---

## 6. 완료 기준

- 벡터 검색 P95 latency가 200ms 이하를 달성하거나, 미달 시 원인과 대안이 문서화된다
- 하이브리드 검색이 keyword-only 대비 Recall@10을 개선함이 확인된다
- ef_search 최종값이 결정되어 코드에 반영된다
- RAGContextProvider 인터페이스가 Phase 11 팀이 구현 가능한 수준으로 정의된다

---

## 7. Codex 작업 지침

- 성능 목표 미달 시 즉각 튜닝하지 않고 원인을 분석하여 Task 10-11 보고서에 기록한다
- Phase 11 RAGContextProvider 인터페이스는 Phase 12 플러그인 구조와도 호환 가능하도록 DocumentType에 독립적으로 설계한다
- 검색 품질 평가 데이터셋은 실제 도메인 데이터(정책, 매뉴얼 등)로 구성하여 현실적인 수치를 확보한다
- ef_search는 설정 파일로 관리하여 운영 중 재배포 없이 조정 가능하게 한다
