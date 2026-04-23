# task0-4 — AI 품질 실측 (골든셋 50+)

**작성일**: 2026-04-22 (2026-04-23 업데이트)
**Phase / FG**: S3 Phase 0 / FG 0-4
**상태**: 착수 대기
**담당**: 백엔드 + AI 품질
**예상 소요**: 3~4일

---

## 1. 작업 목적

S2의 AI 품질 목표(Faithfulness ≥ 0.80, Citation-present ≥ 0.90)는 종결 시점까지 **실측값으로 확인되지 않았다**. S2-5 RAG 스모크(21건)는 `citation_present_rate` 등 구조적 지표만 확인했으며, Faithfulness는 판정 LLM이 없는 폐쇄망 환경에서 계산되지 않았다.

본 FG는:
1. 골든셋을 **50+건**으로 확장
2. **판정 LLM 어댑터**를 폐쇄망 호환(로컬 모델)으로 추가
3. **Faithfulness / Citation-present / Answer-relevance** 실측값 확보
4. AI 품질 평가 보고서 제출

---

## 2. 작업 범위

### 2.1 포함

- 골든셋 `golden_queries.py` 확장 (21 → ≥ 50)
- 샘플 문서 `sample_documents.py` 확장 (16 → ≥ 25)
- 판정 LLM 어댑터 `scripts/rag_smoke/judge.py` 신설 (OpenAI + 로컬 2종)
- Faithfulness / Answer-relevance 지표 계산 로직 추가
- `rag_smoke.py` 확장 (또는 `rag_quality.py` 분리)
- 실측 실행 및 JSON 리포트 제출
- AI 품질 평가 보고서 · 검수 · 보안 보고서

### 2.2 제외

- 판정 LLM 자체 파인튜닝
- retrieval 알고리즘 개선 (본 FG는 **측정**이 목적; 미달 시 원인 진단까지만)
- 답변 스타일 개선 (프롬프트 튜닝은 별도 FG)

---

## 3. 선행 조건

- `scripts/rag_smoke/` 기존 산출물 존재 (`rag_smoke.py`, `golden_queries.py`, `sample_documents.py`)
- 로컬 LLM 실행 환경 (Ollama 또는 vLLM — 사내 폐쇄망 표준 선택)
- FG 0-1 / 0-3 완료 여부는 병행 가능. 단, CI 통합은 FG 0-1 후

---

## 4. 주요 작업 항목

### 4.1 골든셋 확장

기존 21건 + 신규 29건 이상 (목표 ≥ 50, 권장 60):

| 카테고리 | 기존 | 신규 | 합계 | 설명 |
|---------|------|------|------|------|
| simple | 8 | 4 | 12 | 단일 문서 keyword 명확 |
| ambiguous | 4 | 4 | 8 | 재랭커/하이브리드 효과 관찰 |
| typed | 3 | 4 | 7 | document_type 필터 |
| multihop | 3 | 4 | 7 | 한 문서 내 여러 섹션 |
| negative | 3 | 4 | 7 | 환각 방지 |
| **domain-shift** | 0 | 5 | 5 | 새 테마 (예: 브랜딩·해외사업) |
| **adversarial** | 0 | 5 | 5 | 저자 중복·시기 애매·긴 본문 |

샘플 문서도 신규 테마용으로 9건 이상 추가.

### 4.2 판정 LLM 어댑터

신규 파일: `scripts/rag_smoke/judge.py`

```
class JudgeAdapter(Protocol):
    def judge_faithfulness(
        self, question: str, answer: str, citations: list[dict]
    ) -> FaithfulnessResult: ...

    def judge_relevance(
        self, question: str, answer: str
    ) -> float: ...


class OpenAIJudge(JudgeAdapter): ...
class OllamaJudge(JudgeAdapter): ...
class VLLMJudge(JudgeAdapter): ...
```

환경변수:
- `RAG_JUDGE_PROVIDER`: `openai` | `ollama` | `vllm`
- `RAG_JUDGE_MODEL`: 모델명
- `RAG_JUDGE_BASE_URL`: 로컬 LLM의 경우 base URL
- `RAG_JUDGE_API_KEY`: OpenAI의 경우만

규칙:
- 판정 실패 시 해당 질의는 Faithfulness 계산에서 제외 (NaN 기록)
- 판정 LLM 응답 파싱은 JSON schema 기반 (`{score: 0~1, reasoning: str}`)
- 프롬프트 템플릿은 `rag_smoke/judge_prompts.py` 분리
- 단일 판정 타임아웃 30초, 전체 타임아웃은 argparse로 제어

### 4.3 Faithfulness 지표 정의

답변의 각 문장 또는 주장 단위로:
- 해당 주장이 citation snippet에 근거하는가? (0 또는 1)
- 근거가 명확하지 않지만 합리적 추론인가? (0.5)

최종 Faithfulness = 주장별 점수의 평균. 전체 Faithfulness는 쿼리별 평균의 평균(macro).

단순화 옵션: 답변 전체를 하나의 단위로 판정 (0~1 연속). **초기 버전은 단순화 옵션으로 시작, 필요 시 세분화**.

### 4.4 Answer-relevance 지표

질의와 답변의 의미적 연관도를 판정 LLM에 묻는다. 0~1 score.

### 4.5 `rag_quality.py` 러너

기존 `rag_smoke.py`와 분리하는 것을 권장 (기능 집중):
- `python scripts/rag_smoke/rag_quality.py --golden-set ./golden_queries.py --judge-provider ollama --judge-model llama3.1:8b --out ./report.json`
- 시딩 단계는 `rag_smoke.py` 재사용
- 질의 단계 + 판정 단계 분리
- 결과 JSON 스키마:
  ```jsonc
  {
    "run_at": "...",
    "golden_set_version": "...",
    "judge": {"provider": "ollama", "model": "llama3.1:8b"},
    "summary": {
      "faithfulness_macro": 0.82,
      "citation_present_rate": 0.93,
      "answer_relevance_macro": 0.88,
      "retrieval_hit_rate": 0.91,
      "by_category": {...}
    },
    "results": [{"id": "Q01", "faithfulness": 0.9, "relevance": 0.85, ...}]
  }
  ```

### 4.6 실측 실행

- 최소 2회 실측 (랜덤성 완화)
- 폐쇄망 모드(로컬 LLM)에서 1회 + 외부 모드(OpenAI, 선택적 — 키 있으면)에서 1회
- 결과 JSON 저장: `docs/개발문서/S3/phase0/산출물/FG0-4_실측_YYYYMMDD_HHMM.json`

### 4.7 AI 품질 평가 보고서

`docs/개발문서/S3/phase0/산출물/FG0-4_AI품질평가보고서.md`:
- 골든셋 구성 요약
- 판정 LLM 설정 (프로바이더·모델·온도 등)
- 실측 결과 표 (카테고리별 Faithfulness / Citation-present / Relevance / Retrieval-hit)
- 목표 대비 달성도: Faithfulness ≥ 0.80, Citation-present ≥ 0.90
- 미달 시 원인 진단 (retriever / reranker / prompt / 판정 어댑터 안정성 등)
- 폐쇄망 vs 외부 판정 LLM 결과 차이 분석

### 4.8 산출물

- `scripts/rag_smoke/golden_queries.py` (확장)
- `scripts/rag_smoke/sample_documents.py` (확장)
- `scripts/rag_smoke/judge.py` + `judge_prompts.py`
- `scripts/rag_smoke/rag_quality.py` (또는 통합)
- `docs/개발문서/S3/phase0/산출물/FG0-4_실측_*.json` (최소 2회분)
- `docs/개발문서/S3/phase0/산출물/FG0-4_AI품질평가보고서.md`
- `docs/개발문서/S3/phase0/산출물/FG0-4_검수보고서.md`
- `docs/개발문서/S3/phase0/산출물/FG0-4_보안취약점검사보고서.md`

---

## 5. 완료 기준

- [ ] 골든셋 ≥ 50건
- [ ] 판정 어댑터 2종 이상 (로컬 ≥ 1, 외부 ≥ 1) 실행 성공
- [ ] **실측** Faithfulness ≥ 0.80 (폐쇄망 모드 기준)
- [ ] **실측** Citation-present ≥ 0.90
- [ ] AI 품질 평가 보고서 제출
- [ ] 검수 · 보안 보고서 제출

---

## 6. 리스크와 대처

| 리스크 | 대처 |
|--------|------|
| 로컬 LLM 판정 일관성 낮음 | 판정을 3회 반복 후 median 채점 옵션 제공 |
| 폐쇄망에서 모델 크기 제약 | 8B급 모델에서도 판정 프롬프트가 안정적으로 parsing 되도록 JSON schema + strict retry |
| Faithfulness 0.80 미달 | 원인 진단 섹션에서 retriever vs generator 분리 분석 후 개선 티켓을 Phase 1 이후로 생성 |
| 실측 중 RAG 서비스 rate limit | 기존 `--rag-sleep` 유지, 판정은 별도 호출이므로 RAG 쪽 rate limit에는 영향 없음 |
| 외부 OpenAI 키 공개 | 보안보고서에서 `.env` 및 CI secret 관리 경로 명시, 기본 어댑터를 ollama로 |

---

## 7. 보안 주의

- 판정 LLM에 질의/답변/citation snippet이 전달됨. **폐쇄망 사내 문서가 외부 API로 유출되지 않도록** `RAG_JUDGE_PROVIDER=openai` 사용 시 경고 + 사용자 확인 프롬프트
- 로그에 판정 LLM API key 출력 금지
- 판정 결과 JSON에는 snippet 포함. git push 전 `.gitignore` 확인 (S2-5 보안보고서 권고 재확인)

---

## 8. 참조

- `docs/개발문서/S3/phase0/Phase 0 개발계획서.md` §2 FG 0-4
- `docs/개발문서/S2_5/RAG_스모크_실행가이드.md`
- `docs/개발문서/S2_5/RAG_스모크_검수보고서.md`
- `docs/개발문서/S2_5/RAG_스모크_보안취약점검사보고서.md` §6 LLM06 주의
- `CLAUDE.md` S1 ⑤ ⑥ + S2 ⑦ (폐쇄망)

---

*작성: 2026-04-22 | 착수 대기*
