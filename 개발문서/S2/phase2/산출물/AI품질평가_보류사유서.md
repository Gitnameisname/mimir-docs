# AI 품질평가 보류 사유서 — Phase 2 (Grounded Retrieval v2 + Citation Contract)

**작성일**: 2026-04-18
**작성자**: Claude Opus 4.7 (S2 전체 재검수 후속)
**대상 Phase**: S2 Phase 2 — Grounded Retrieval v2 + Citation Contract
**상위 근거**: `docs/개발문서/S2_종결보정서_20260418.md` §2 (F-02 AI 품질 기준 달성 선언 철회)
**상태**: **보류 (Deferred to S3 P0)**

---

## 1. 본 문서의 위치

본 문서는 「AI 품질평가 **보고서**」가 아니다. S2 Phase 2 의 AI 관련 기능(Grounded Retrieval, Citation Contract, 쿼리 재작성) 에 대해 `CLAUDE.md` 규약이 요구하는 정량 품질 지표(Faithfulness ≥0.80 · Citation-present ≥0.90 · Precision ≥0.85 등) 를 **왜 지금 실측하지 않고 보류했는지**, **언제·어떤 방법으로 측정할 것인지** 를 공식 기록하는 문서이다.

S2 종결보정서 F-02 에서 확인된 바와 같이, S2 종결 시점에 다음이 모두 **부재** 하다.

- 골든셋(Golden Q&A) 데이터: **0건**
- 평가 run 메타데이터(`evaluation_reports/*.json`): **0건**
- LLM 판정 backend 실운영 기록: **0건**

따라서 Phase 2 의 AI 품질에 대해 수치를 산출하는 행위는 **근거 없는 수치 생성(= 허위 보고)** 에 해당하므로 본 문서로 보류를 명시하고 실측 시점까지 수치 선언을 금지한다.

---

## 2. Phase 2 의 AI 관련 기능 범위

품질평가가 필요한 Phase 2 의 기능은 다음과 같다.

| 기능 | 코드 위치 | 평가가 필요한 품질 축 |
|------|-----------|----------------------|
| Citation 5-tuple 계약 (document_id / version_id / node_id / span_offset / content_hash) | `backend/app/services/retrieval/citation_*` | Citation-present rate, Citation 정합성(해시 검증), span 정확도 |
| Retriever 플러그인 (FTS / Vector / Hybrid) 선택 | `backend/app/services/retrieval/retrievers/` | Context Precision / Context Recall, DocumentType 별 선택 정확도 |
| Reranker 플러그인 | `backend/app/services/retrieval/rerankers/` | Context Precision 개선폭(Before/After) |
| 멀티턴 쿼리 재작성 | `backend/app/services/retrieval/query_rewrite.py` | 의도 보존율, Follow-up 재작성 후 retrieval 품질 |
| Grounded Retrieval 응답(범위 이탈 금지) | `backend/app/api/v1/search_router.py` | 헛소리율(환각률), Faithfulness(LLM 답변 부착 시) |

---

## 3. 보류 사유

### 3.1 필수 전제 자원 미확보

1. **골든셋 Q&A 데이터**: Phase 7 FG7.1 에서 골든셋 CRUD API·import/export API 인프라는 완성되었으나, **데이터 자체가 0건** 이다. Citation-present rate / Context Precision / Context Recall 은 모두 "정답 citation 집합" 에 대한 비교 연산이므로, 골든셋 없이는 수치화 불가능하다.
2. **LLM 판정 backend**: Faithfulness·Answer Relevance 는 LLM-as-a-Judge 방식으로 산출된다. S2 종결 시점의 실운영 환경에서는 closed-network fallback (로컬 vLLM) 이 상시 가동되지 않는 상태이다.
3. **재현 가능한 retrieval run**: `scripts/evaluate_golden_set.py` 인프라는 완성되었으나 실행 로그(`evaluation_reports/`) 가 0건이므로 "측정된 적 없음" 이 사실이다.

### 3.2 구조적으로 충족 가능한 부분 (보류 사유가 아닌 진술)

Phase 2 구현은 골든셋 기반 측정 시 기준을 충족 **할 수 있도록** 설계되었다 — 단, 이는 "현재 충족했다" 가 아니다.

- Citation 5-tuple 은 검색 응답 계약에 필드로 **강제** 되어 있으므로, retrieval 결과에 citation 이 누락될 물리적 경로가 없다 → Citation-present rate 은 측정 시 ≥0.90 에 근접할 것으로 예상되나 **예상일 뿐 실측 아님**.
- content_hash 재검증 로직(`/citations/.../verify`) 이 있으므로 citation 정합성은 API 레벨에서 검증 가능 — 이 역시 "API 존재" 일 뿐 "품질 수치" 아님.

### 3.3 현재 시점에서 **검증된 것** (단위 테스트 범위)

Phase 2 의 단위 테스트·통합 테스트 범위에서는 다음이 확인되어 있다(본 문서는 이들을 "품질 지표 달성" 으로 바꿔치기하지 않는다).

| 검증 항목 | 수단 | 상태 |
|-----------|------|------|
| Citation 필드 5종이 응답에 항상 포함됨 | 통합 테스트 | ✅ |
| content_hash 재계산 후 일치 확인 API | 단위 테스트 | ✅ |
| Hybrid Retriever 의 FTS + Vector 합산 점수 재정렬 | 단위 테스트 | ✅ |
| 쿼리 재작성 후 빈 쿼리·주입 패턴 거부 | 단위 테스트 | ✅ |

이들은 **기능적 정합성** 이지 **LLM 품질 지표** 가 아니다.

---

## 4. 실측 계획 (S3 P0 이월)

S2 종결보정서 §2.4 의 이월 과제와 연결된다. Phase 2 범위의 실측은 다음 순서로 수행한다.

| 단계 | 내용 | 산출물 | 담당 | 기한 |
|------|------|--------|------|------|
| 1 | Phase 2 전용 골든셋 20건 이상 구축 (Citation 을 요구하는 질문 위주) | `golden_sets/phase2_retrieval.json` | 지식팀 + 개발팀 | S3 착수 + 2주 |
| 2 | `scripts/evaluate_golden_set.py --closed-network --set phase2_retrieval` 1회 실행 | `evaluation_reports/phase2_retrieval_<date>.json` | 개발팀 | 1단계 완료 직후 |
| 3 | 본 문서를 **AI품질평가보고서_Phase2.md** 로 승격 (수치 채움) | Phase 2 산출물 폴더 | 개발팀 | 2단계 직후 |
| 4 | 미달 시 PromptBuilder / Retriever 파라미터 재조정 후 재측정 | 동일 | 개발팀 | 3단계 평가 결과에 따라 |

---

## 5. 수치 선언 금지 사항

본 보류가 해제되기 전까지 다음 문구는 **어떤 문서에도 기재되어서는 안 된다**.

- "Phase 2 는 Faithfulness X.XX 를 달성했다"
- "Phase 2 Citation-present rate 은 X.XX 이다"
- "Phase 2 는 AI 품질 기준을 충족한다"

S2 종결보고서의 동등한 선언은 S2_종결보정서_20260418.md §2.3 에서 **공식 철회** 되었다. 본 Phase 2 보류 사유서는 그 철회 결정을 Phase 2 수준에서 구체화한 것이다.

---

## 6. 서명

| 역할 | 이름 | 날짜 |
|------|------|------|
| 재검수 주관 | Claude Opus 4.7 | 2026-04-18 |
| 기술 리드 | (서명 대기 — S3 P0 착수 시) | |
