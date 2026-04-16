# Phase 7 FG7.2 작업 문서

**페이즈**: Phase 7 FG7.2 (평가 러너 + 지표)  
**목적**: 골든셋을 입력으로 받아, 실제 RAG 시스템의 응답을 평가하고 정량 지표를 산출하는 엔진 구축.

## 작업 구성

총 4개 Task로 구성되며, 순차적으로 진행:

### [task7-4: 규칙 기반 평가 지표 구현](작업지시서/task7-4.md)
- **예상 소요시간**: 5~6일
- **산출물 크기**: 40KB (1,191줄)
- **주요 내용**:
  - Citation 5-tuple 형식 정의 및 감지
  - 문장 분할 유틸리티
  - CitationPresentMetric, HallucinationRateMetric, ContextRecallMetric 구현
  - 단위 테스트 (8개 이상)
- **완료 기준**:
  - 3가지 지표 모두 0.0~1.0 정규화 점수 반환
  - 엣지 케이스 처리 (공백, 마크다운, 특수문자)
  - 모든 테스트 통과 (80%+ 커버리지)

### [task7-5: LLM 기반 평가 지표 + 규칙 기반 폴백](작업지시서/task7-5.md)
- **예상 소요시간**: 6~7일
- **산출물 크기**: 44KB (1,288줄)
- **주요 내용**:
  - Faithfulness 평가 (LLM 판정자)
  - Context Precision 평가 (LLM 판정자)
  - 규칙 기반 폴백 (텍스트 오버랩, 키워드 매칭, TF-IDF)
  - 환경변수 `EVAL_LLM_ENABLED` 기반 전환
  - Prompt 템플릿 (영어, 한국어)
- **S2 원칙**:
  - ⑦ 폐쇄망 환경: LLM 불가 시 폴백으로 완전 동작

### [task7-6: Answer Relevance + Evaluator 클래스 통합](작업지시서/task7-6.md)
- **예상 소요시간**: 7~8일
- **산출물 크기**: 40KB (1,156줄)
- **주요 내용**:
  - Answer Relevance 지표 (임베딩 기반)
  - Evaluator 클래스: 모든 6가지 지표 통합
  - 성능 지표 (Latency, Token Count, Cost)
  - Pydantic 데이터 모델 (TokenMetrics, LatencyMetrics, CostMetrics, EvaluationResult, EvaluationReport)
  - 비동기 배치 평가 (asyncio.gather, 동시성 제어)
- **완료 기준**:
  - evaluate_golden_item() 동기 + 비동기
  - evaluate_golden_set() 배치 평가 (세마포어 제한)
  - 통계 계산 (min, max, mean, median, std)

### [task7-7: 평가 실행 API + 결과 저장](작업지시서/task7-7.md)
- **예상 소요시간**: 8~9일
- **산출물 크기**: 39KB (1,166줄)
- **주요 내용**:
  - REST API 엔드포인트 (FastAPI)
    - POST /api/v1/evaluations/run
    - GET /api/v1/evaluations/{eval_id}
    - GET /api/v1/evaluations (목록, 페이지네이션)
    - GET /api/v1/evaluations/{eval_id}/compare
  - 데이터베이스 모델 (EvaluationRun, EvaluationResultRecord)
  - Repository 패턴 (CRUD)
  - 배경 작업 (비동기 처리)
- **S2 원칙**:
  - ⑤ Actor Type 감사 로그 ("user" vs "agent")
  - ⑥ Scope Profile 기반 ACL (모든 엔드포인트)

## 6가지 평가 지표

| 지표 | 구현 방식 | 폴백 | 범위 | Task |
|------|----------|------|------|------|
| Faithfulness | LLM 판정 | 텍스트 오버랩 | 0.0~1.0 | task7-5 |
| Answer Relevance | 임베딩 코사인 유사도 | 키워드 오버랩 | 0.0~1.0 | task7-6 |
| Context Precision | LLM 판정 | 키워드 매칭 | 0.0~1.0 | task7-5 |
| Context Recall | 규칙 기반 (소스 매칭) | N/A | 0.0~1.0 | task7-4 |
| Citation-present | 규칙 기반 (정규식) | N/A | 0.0~1.0 | task7-4 |
| Hallucination Rate | 규칙 기반 (반수 계산) | N/A | 0.0~1.0 | task7-4 |

## 의존성 관계

```
task7-4 (규칙 기반)
    ↓
task7-5 (LLM + 폴백) [task7-4에 의존]
    ↓
task7-6 (Answer Relevance + Evaluator) [task7-4, task7-5에 의존]
    ↓
task7-7 (API + 저장) [task7-6에 의존]
```

## 각 작업지시서 구성

모든 작업지시서는 7개 섹션으로 구성:

1. **작업 목적**: Why this task exists
2. **작업 범위**: 포함 범위 (세부 항목) + 제외 범위
3. **선행 조건**: Prerequisites
4. **주요 작업 항목**: 상세 구현 (코드 예시 포함)
5. **산출물**: Deliverables with file paths
6. **완료 기준**: 수용 기준 (체크리스트)
7. **작업 지침**: 구현 시 가이드라인 (지침 7-1, 7-2, ...)

## 파일 구조

```
/backend/app/
├── services/evaluation/
│   ├── metrics/
│   │   ├── citation.py                      # task7-4
│   │   ├── sentence_splitter.py             # task7-4
│   │   ├── rule_based.py                    # task7-4
│   │   ├── fallback.py                      # task7-5
│   │   ├── llm_based.py                     # task7-5
│   │   ├── embedding_based.py               # task7-6
│   │   └── __init__.py
│   ├── prompts/
│   │   └── judge_prompts.py                 # task7-5
│   ├── evaluator.py                         # task7-6
│   ├── background_tasks.py                  # task7-7
│   └── models.py                            # task7-6
├── models/
│   └── evaluation.py                        # task7-7
├── repositories/
│   └── evaluation_repository.py              # task7-7
└── api/v1/
    └── evaluations.py                       # task7-7

/backend/tests/
├── services/evaluation/metrics/
│   ├── test_rule_based.py                   # task7-4
│   └── test_llm_based.py                    # task7-5
├── services/evaluation/
│   └── test_evaluator.py                    # task7-6
└── api/
    └── test_evaluations.py                  # task7-7
```

## 크로스컷(Cross-cutting) 원칙

### S2 원칙 ⑤: AI 에이전트는 사람과 동등한 API 소비자
- 모든 API 호출에 `actor_type` 필드 기록
- 감사 로그: `{"actor": "user_id", "actor_type": "user|agent", ...}`
- 추후 에이전트별 성능 분석 가능

### S2 원칙 ⑥: 폐쇄망 환경에서 접근 제어
- Scope Profile 기반 ACL 필터링
- 모든 검색/조회/쓰기 API에 적용
- 폴백 경로(내부 DB)에서도 동일한 ACL 적용

### S2 원칙 ⑦: 폐쇄망 환경에서 전체 기능 동작
- 환경변수 `EVAL_LLM_ENABLED=false` 시
- LLM 기반 지표는 규칙 기반 폴백으로 자동 전환
- 서비스 실패 안 함, 정확도 저하만 발생

## 산출물 체크리스트

### 코드 산출물
- [ ] task7-4: citation.py, sentence_splitter.py, rule_based.py, __init__.py
- [ ] task7-5: fallback.py, llm_based.py, judge_prompts.py
- [ ] task7-6: embedding_based.py, evaluator.py, models.py
- [ ] task7-7: evaluation.py (models), evaluation_repository.py, background_tasks.py, evaluations.py (api)

### 테스트 산출물
- [ ] test_rule_based.py (8+ test cases)
- [ ] test_llm_based.py (8+ test cases)
- [ ] test_evaluator.py (8+ test cases)
- [ ] test_evaluations.py (6+ test cases)

### 문서 산출물
- [ ] task7-4-검수보고서.md
- [ ] task7-5-검수보고서.md
- [ ] task7-6-검수보고서.md
- [ ] task7-7-검수보고서.md
- [ ] task7-4~7-보안검사보고서.md (선택)

## 예상 타임라인

| Task | 예상 기간 | 검수 기간 | 총 소요 |
|------|---------|---------|--------|
| task7-4 | 5~6일 | 1일 | 6~7일 |
| task7-5 | 6~7일 | 1일 | 7~8일 |
| task7-6 | 7~8일 | 1일 | 8~9일 |
| task7-7 | 8~9일 | 1일 | 9~10일 |
| **전체** | **26~30일** | **4일** | **30~34일** |

## 검수 및 품질 보증

각 Task 완료 후:

1. **검수 보고서** 작성
   - 완료 기준 체크리스트 (7가지 각 10+ 항목)
   - 코드 리뷰 결과
   - 테스트 커버리지 (80%+ 목표)

2. **보안 취약점 검사** (선택)
   - OWASP Top 10 체크
   - Dependency 취약점 스캔
   - 감사 로그 기록 확인

3. **AI 품질 평가** (task7-5, 7-6 포함 시)
   - Faithfulness ≥ 0.80
   - Citation-present ≥ 0.90

---

**최종 수정**: 2026-04-17  
**담당자**: [TBD]
