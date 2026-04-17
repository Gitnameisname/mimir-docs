# Phase 7 개발계획서
## AI 품질 평가 인프라

---

## 1. 단계 개요

### 단계명
Phase 7. AI 품질 평가 인프라

### 목적
RAG 품질 회귀를 정량적으로 측정하고, CI 게이트로 방어하는 평가 인프라를 구축한다. 골든 Q&A 셋을 도메인 모델로 관리하고, 룰 기반·LLM 기반 평가 지표를 구현하여 각 Phase의 AI 기능이 정의된 품질 기준을 충족하는지 자동 검증한다.

### 선행 조건
- Phase 0~6 완료
- Phase 1: LLM Provider 추상화 완료 (판정 LLM 선택 가능)
- Phase 2: Citation 5-tuple 계약 확정
- Phase 6 FG6.3: 평가 결과 대시보드 스켈레톤 완성

### 기대 결과
- 골든셋 도메인 모델 + CRUD API + import/export 기능
- 평가 러너 + 6가지 평가 지표 (Faithfulness, Answer Relevance, Context Precision/Recall, Citation-present, 헛소리율)
- CI 게이트 스크립트 (임계값 기반 빌드 실패)
- AI품질평가.md 자동 생성 템플릿
- Phase 6 FG6.3 대시보드와의 실시간 데이터 바인딩

---

## 2. Phase 7의 역할

Phase 7은 "평가의 자동화 및 게이트 정책 확립" Phase이다. 다음을 담당한다:

1. **평가 기준의 구체화**: Phase 0에서 "Faithfulness ≥0.80" 등으로 정한 목표를 실제 구현과 측정 메커니즘으로 실현
2. **다층 평가 전략**: LLM 판정이 불가능한 폐쇄망에서도 동작하도록, 룰 기반 지표부터 점진적으로 시작
3. **회귀 방어선 구축**: 각 Phase의 AI 기능이 추가될 때마다 품질 게이트를 통과해야만 merge 가능하도록 강제
4. **평가 데이터의 추적성**: 어느 프롬프트, 어느 모델에서 어떤 품질이 나왔는지 모두 기록하여, 프롬프트 A/B 테스트와 모델 선택의 근거 제공

---

## 3. Feature Group 상세

### FG7.1 — 골든셋 도메인 모델 + API

#### 목적
RAG 품질 평가의 기준이 되는 "골든 Q&A 셋"을 관리 가능한 도메인 객체로 정의하고, CRUD API 및 버전 관리를 제공.

#### 주요 작업 항목

1. **Golden Q&A 4-tuple 모델 정의**
   - 각 항목의 구성:
     ```
     {
       id: UUID,
       golden_set_id: UUID,
       version: integer,
       
       question: string,           // 사용자가 제시할 질문
       expected_answer: string,    // 기대하는 답변 (또는 답변 요약)
       expected_source_docs: [     // 기대하는 근거 문서 목록
         {document_id, version_id, node_id}
       ],
       expected_citations: [       // 기대하는 citation 5-tuple
         {document_id, version_id, node_id, span_offset, content_hash}
       ],
       
       notes: string,              // 평가자 주석 (왜 이것이 정답인가)
       created_at: timestamp,
       created_by: user_id,
       updated_at: timestamp
     }
     ```

2. **GoldenSet 엔티티 (컬렉션)**
   - 여러 개의 Golden Q&A를 그룹화
   - 메타데이터: 이름, 설명, 도메인(예: "규정 문서 RAG", "매뉴얼 검색"), 버전
   - 의도: 서로 다른 도메인별로 평가 기준 분리 (예: 기술 문서 vs. 정책 문서)

3. **Golden Q&A CRUD API**
   - `POST /api/v1/golden-sets` — 새로운 GoldenSet 생성
   - `GET /api/v1/golden-sets` — 전체 목록 조회 (페이지네이션)
   - `GET /api/v1/golden-sets/{id}` — 상세 조회 (모든 Q&A 항목 포함)
   - `PUT /api/v1/golden-sets/{id}` — 메타데이터 수정 (이름, 설명 등)
   - `DELETE /api/v1/golden-sets/{id}` — 삭제 (버전 이력은 남음)
   
   - `POST /api/v1/golden-sets/{id}/items` — 새로운 Q&A 항목 추가
   - `GET /api/v1/golden-sets/{id}/items` — 항목 목록 조회
   - `PUT /api/v1/golden-sets/{id}/items/{item_id}` — Q&A 항목 수정 (자동 새 버전 생성)
   - `DELETE /api/v1/golden-sets/{id}/items/{item_id}` — 항목 삭제

4. **버전 관리**
   - Document와 동일한 버전 정책: 수정 시 새 버전 자동 생성
   - `GET /api/v1/golden-sets/{id}/versions` — 버전 이력 조회
   - `GET /api/v1/golden-sets/{id}/versions/{v}` — 특정 버전 조회
   - 버전 간 diff 조회 가능

5. **import/export (JSON)**
   - `POST /api/v1/golden-sets/{id}/import` — JSON 파일 업로드 (대량 Q&A 일괄 생성)
   - `GET /api/v1/golden-sets/{id}/export` — JSON 다운로드
   - JSON 형식 정의:
     ```json
     {
       "name": "기술 문서 RAG",
       "description": "내부 기술 명세서 검색 평가",
       "items": [
         {
           "question": "Docker란 무엇인가?",
           "expected_answer": "Docker는 컨테이너 기술을 통해...",
           "expected_source_docs": [
             {"document_id": "doc-123", "version_id": 1, "node_id": "n-5"}
           ],
           "notes": "공식 문서의 정의 섹션에서 나온 답"
         }
       ]
     }
     ```

6. **권한 및 감시 (S2 원칙 ⑥ Scope Profile 기반)**
   - GoldenSet 열람: Scope Profile의 FilterExpression 기반 ACL 적용 — 사용자의 $ctx.* 변수에 따라 허용된 GoldenSet만 조회 가능 (S2 원칙 ⑥: scope 하드코딩 금지)
   - GoldenSet 수정: Admin 역할 또는 Scope Profile에 `golden_set:write` 권한이 포함된 사용자
   - 에이전트 접근: agent principal도 동일 Scope Profile ACL 적용 (S2 원칙 ⑤: AI-as-first-class-consumer)
   - 수정 감시: 감사 로그에 `action=golden_set_item_added/modified/deleted`, `actor_type=user|agent` 기록

#### 입력 (선행 의존)
- Phase 1: LLM Provider 추상화 (평가 실행 시 사용)
- Phase 2: Citation 5-tuple 정의 (expected_citations 필드)
- Document 및 Node 도메인 모델

#### 출력 (산출물)
- GoldenSet 도메인 모델 정의 (Pydantic)
- GoldenSet CRUD API (FastAPI)
- import/export 엔드포인트
- 버전 관리 로직 (Document와 공통)
- 감시 로그 (감사 로그 적분)

#### 검수 기준
- import/export JSON이 손실 없이 왕복 변환되는가
- 버전 간 diff가 정확하게 계산되는가
- 권한 기반 접근 제어가 정상 작동하는가
- expected_citations의 5-tuple이 실제 Citation과 호환되는가

---

### FG7.2 — 평가 러너 + 지표

#### 목적
골든셋을 입력으로 받아, 실제 RAG 시스템의 응답을 평가하고 정량 지표를 산출하는 엔진 구축.

#### 주요 작업 항목

1. **평가 지표 정의 및 구현**

   **1a. Faithfulness (충실성)**
   - 정의: 생성된 답변이 검색된 근거에 충실한가 (hallucination 여부)
   - 구현: LLM 판정 (Phase 1의 판정 LLM 사용)
   - 프롬프트:
     ```
     주어진 근거 텍스트와 질문, 답변이 주어졌을 때,
     답변이 근거에만 기반하여 생성되었고, 근거에 없는 정보를 추가하지 않았다면 1점,
     근거를 왜곡하거나 근거에 없는 정보를 추가했다면 0점.
     [근거]
     {retrieved_text}
     [질문]
     {question}
     [답변]
     {answer}
     평가: (0 또는 1)
     ```
   - 스코어: 0.0~1.0 (여러 근거가 있으면 평균)
   - 외부 모델 불가 환경: 규칙 기반 대체 (답변이 근거에서 추출된 텍스트와 일정 비율 이상 매칭되는가)

   **1b. Answer Relevance (답변 관련성)**
   - 정의: 생성된 답변이 사용자 질문과 의미적으로 관련 있는가
   - 구현: Embedding 기반 (cosine similarity)
   - 프롬프트 필요 없음, Phase 1의 embedding model 사용
   - 스코어: 질문 embedding과 답변 embedding의 cosine similarity (0.0~1.0)

   **1c. Context Precision (컨텍스트 정밀도)**
   - 정의: 검색된 청크 중 답변에 실제로 사용된 비율
   - 구현: LLM 판정 (각 검색 청크가 답변 생성에 기여했는가)
   - 프롬프트:
     ```
     다음 검색 결과가 주어진 답변을 생성하는 데 얼마나 도움이 되었는가?
     [검색 결과 1]
     [검색 결과 2]
     ...
     [답변]
     {answer}
     각 검색 결과에 대해 도움 정도(0~1)를 평가하고 평균을 계산.
     ```
   - 스코어: (도움이 된 청크 수) / (검색된 총 청크 수)

   **1d. Context Recall (컨텍스트 회수율)**
   - 정의: 답변에 필요한 청크 중 실제로 검색된 비율
   - 구현: 골든셋의 expected_source_docs와 실제 검색 결과 비교
   - 계산: (실제 검색된 expected_source_docs) / (모든 expected_source_docs)
   - 스코어: 0.0~1.0

   **1e. Citation-present Rate (근거 제시율)**
   - 정의: 생성된 답변이 검증 가능한 citation을 포함하는가
   - 구현: 규칙 기반 (응답에 5-tuple citation이 포함되었는가)
   - 계산: (citation 포함된 문장 수) / (총 문장 수)
   - 스코어: 0.0~1.0

   **1f. Hallucination Rate (헛소리율)**
   - 정의: 답변이 근거에 없는 정보를 포함하는가
   - 구현: 규칙 기반 (citation 없는 문장의 비율)
   - 계산: (citation 없는 문장 수) / (총 문장 수)
   - 스코어: 0.0~1.0 (낮을수록 좋음)

   **1g. 성능 지표 (Latency, Tokens, Cost)**
   - Latency: 질문→답변 총 소요 시간 (ms)
   - Input tokens: RAG 리트리브 + LLM generate 입력 토큰 합계
   - Output tokens: LLM 생성 토큰
   - Estimated cost: Phase 1의 모델 메타데이터에서 제공된 단가로 계산

2. **평가 러너 (Evaluator 클래스)**
   ```python
   class Evaluator:
       def __init__(self, model_config: ModelConfig, embedding_model: str):
           self.judge_llm = get_llm_by_config(model_config)  # Phase 1
           self.embedder = get_embedder_by_name(embedding_model)  # Phase 1
       
       def evaluate_golden_item(self, 
           golden_item: GoldenItem,
           retrieved_docs: List[RetrievedDoc],
           generated_answer: str,
           latency_ms: float,
           tokens: TokenMetrics
       ) -> EvaluationResult:
           """각 단일 Golden Q&A에 대한 평가 수행"""
           return EvaluationResult(
               faithfulness=self._compute_faithfulness(retrieved_docs, generated_answer),
               answer_relevance=self._compute_answer_relevance(golden_item.question, generated_answer),
               context_precision=self._compute_context_precision(retrieved_docs, generated_answer),
               context_recall=self._compute_context_recall(golden_item.expected_source_docs, retrieved_docs),
               citation_present_rate=self._compute_citation_present_rate(generated_answer),
               hallucination_rate=self._compute_hallucination_rate(generated_answer),
               latency_ms=latency_ms,
               tokens=tokens,
               estimated_cost=self._compute_cost(tokens)
           )
       
       def evaluate_golden_set(self,
           golden_set: GoldenSet,
           rag_system: RAGSystem,
           prompt_version: str,
           model_name: str
       ) -> EvaluationReport:
           """전체 Golden Set에 대한 평가 실행"""
           results = []
           for item in golden_set.items:
               answer, docs, metrics = rag_system.answer_with_metrics(item.question, prompt_version)
               result = self.evaluate_golden_item(item, docs, answer, metrics)
               results.append(result)
           
           return EvaluationReport(
               golden_set_id=golden_set.id,
               golden_set_version=golden_set.version,
               prompt_version=prompt_version,
               model_name=model_name,
               evaluated_at=now(),
               results=results,
               summary=self._summarize(results)
           )
   ```

3. **평가 실행 API**
   - `POST /api/v1/evaluations/run` — Golden Set 평가 실행
     - 입력: golden_set_id, prompt_version, model_name, (optional) filter
     - 출력: evaluation_id (비동기 태스크)
   - `GET /api/v1/evaluations/{eval_id}` — 평가 결과 조회
   - `GET /api/v1/evaluations` — 평가 실행 이력 목록
   - `GET /api/v1/evaluations/{eval_id}/compare` — 두 평가 결과 비교

4. **평가 결과 저장 (TimeSeries DB 고려)**
   - 각 평가 실행은 평가 시점, 프롬프트 버전, 모델 선택을 기록
   - 시계열 쿼리 가능: "지난 30일 Faithfulness 추이"
   - 선택 사항: InfluxDB 또는 Prometheus에 메트릭 export (Phase 6 대시보드와 연계)

#### 입력 (선행 의존)
- Phase 1 FG1.1: LLM Provider (판정 LLM)
- Phase 1 FG1.2: Embedding Model
- Phase 2 FG2.1: Citation 5-tuple 계약
- FG7.1: GoldenSet 도메인 모델
- RAG 시스템의 `answer_with_metrics()` 메서드 (Phase 2/3 확장)

#### 출력 (산출물)
- Evaluator 클래스 (각 지표별 구현)
- EvaluationResult, EvaluationReport 데이터모델
- 평가 실행 API
- 지표별 단위 테스트 (특히 LLM 불가 환경에서의 폴백)

#### 검수 기준
- 각 지표가 0.0~1.0 범위 내에 있는가
- LLM 판정이 없을 때 규칙 기반 대체 지표가 동작하는가
- Faithfulness 점수가 실제 hallucination 여부와 상관관계 확인
- Context Recall이 expected_source_docs와 실제 검색 결과를 정확히 비교하는가
- Citation-present rate가 5-tuple 구조 인식
- 성능 지표(latency, tokens, cost)가 정확하게 수집되는가

---

### FG7.3 — CI 게이트

#### 목적
CI 파이프라인에 골든셋 회귀 테스트 잡을 추가하여, 품질 기준 미만의 커밋이 merge되는 것을 방지.

#### 주요 작업 항목

1. **CI 잡 정의 (GitHub Actions)**
   ```yaml
   name: Golden Set Regression Test
   on: [pull_request, push]
   
   jobs:
     evaluation:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         - name: Set up Python
           uses: actions/setup-python@v4
           with:
             python-version: '3.11'
         - name: Install dependencies
           run: pip install -r requirements.txt
         - name: Start Mimir backend
           run: docker-compose up -d
         - name: Wait for backend ready
           run: sleep 10 && curl http://localhost:8000/api/v1/health
         - name: Run golden set evaluation
           run: python -m pytest tests/evaluation/test_golden_set.py -v --tb=short
         - name: Check thresholds
           run: python scripts/check_evaluation_thresholds.py
         - name: Report results
           if: always()
           run: python scripts/generate_evaluation_report.md
         - name: Comment on PR
           if: failure()
           uses: actions/github-script@v6
           with:
             script: |
               github.rest.issues.createComment({
                 issue_number: context.issue.number,
                 owner: context.repo.owner,
                 repo: context.repo.repo,
                 body: '평가 게이트 실패: AI품질평가.md 참고'
               })
   ```

2. **임계값 설정 (Phase 0 기준값 인용)**
   ```python
   EVALUATION_THRESHOLDS = {
       "faithfulness": 0.80,        # ≥0.80
       "answer_relevance": 0.75,    # ≥0.75
       "context_precision": 0.75,   # ≥0.75
       "context_recall": 0.75,      # ≥0.75
       "citation_present_rate": 0.90,  # ≥0.90
       "hallucination_rate": 0.10,  # ≤0.10 (낮을수록 좋음)
   }
   
   def check_thresholds(evaluation_report: EvaluationReport) -> Tuple[bool, List[str]]:
       """평가 결과가 임계값을 충족하는가?"""
       failures = []
       for metric, threshold in EVALUATION_THRESHOLDS.items():
           value = evaluation_report.summary[metric]
           if metric in ["hallucination_rate"]:
               # 낮을수록 좋은 지표
               if value > threshold:
                   failures.append(f"{metric}: {value:.2%} (임계값: {threshold:.2%})")
           else:
               # 높을수록 좋은 지표
               if value < threshold:
                   failures.append(f"{metric}: {value:.2%} (임계값: {threshold:.2%})")
       
       return len(failures) == 0, failures
   ```

3. **실패 동작**
   - 임계값 미충족 시 빌드 실패
   - 실패 원인 명시 (어떤 지표가 임계값 이하인가)
   - PR 코멘트로 자동 피드백
   - 오버라이드 가능 조건:
     - Maintainer의 명시적 approval (라벨: `evaluation-override`)
     - 평가 결과가 "성능 개선" 단계 (이전 버전 대비 상승)

4. **AI품질평가.md 자동 생성**
   - 평가 완료 후 markdown 보고서 자동 생성
   - 포맷:
     ```markdown
     # AI 품질 평가 보고서
     
     **평가 대상**: Golden Set v{version}, {golden_set_name}
     **평가 시점**: {timestamp}
     **프롬프트**: {prompt_version}
     **모델**: {model_name}
     
     ## 지표 요약
     
     | 지표 | 실측값 | 임계값 | 상태 |
     |------|--------|--------|------|
     | Faithfulness | 0.82 | ≥0.80 | ✓ PASS |
     | Answer Relevance | 0.78 | ≥0.75 | ✓ PASS |
     | Citation-present | 0.92 | ≥0.90 | ✓ PASS |
     
     ## 항목별 결과
     
     [각 Golden Item 평가 상세...]
     
     ## 결론
     
     모든 임계값을 충족합니다.
     ```
   - 템플릿 경로: `templates/ai_quality_evaluation.md.jinja2`

5. **평가 스크립트 (수동 실행 가능)**
   - `scripts/evaluate_golden_set.py`
   - 용도: 로컬에서 평가 결행, 개발 중 지표 확인
   - 사용법:
     ```bash
     python scripts/evaluate_golden_set.py \
       --golden-set-id {id} \
       --prompt-version {version} \
       --model gpt-4o \
       --output-dir reports/
     ```

#### 입력 (선행 의존)
- FG7.2: 평가 러너 및 API
- GitHub Actions 인프라
- Phase 1: 판정 LLM (CI 환경에서 접근 가능)

#### 출력 (산출물)
- GitHub Actions workflow (YAML)
- 임계값 정의 및 검사 로직 (Python)
- AI품질평가.md 생성 스크립트 (Jinja2 템플릿)
- 수동 평가 스크립트

#### 검수 기준
- CI 잡이 정상 완료/실패 판단하는가
- 실패 시 PR 코멘트가 명확한가
- 생성된 AI품질평가.md가 정해진 포맷을 따르는가
- 폐쇄망 환경(external LLM 없음)에서도 규칙 기반 지표가 평가되는가

---

## 3.5 이월 항목 (Phase 6 미결 사항)

Phase 6 종결 시 처리되지 않은 항목으로, Phase 7에서 반드시 완료해야 한다.

| ID | 원 Phase | 항목 | 우선순위 | 담당 Task |
|----|---------|------|---------|----------|
| PH6-CARRY-001 | Phase 6 FG6.2 | Batch 승인/거절 롤백 UI (AdminProposalsPage) | 중간 | task7-11 |
| PH3-CARRY-002 | Phase 3 FG3.x | 대화 제목 검색 FTS 전환 (tsvector + GIN) | 낮음 | task7-10 |
| PH5-CARRY-003 | Phase 5 FG5.x | 1000개 이상 제안 동시 부하 테스트 | 낮음 | task7-12 |

### PH6-CARRY-001: Batch 승인/거절 롤백 UI
- **내용**: AdminProposalsPage의 일괄 승인/거절 실행 후 일정 시간(30초) 내 되돌리기 가능한 Undo UI
- **요구사항**: 
  - BatchToolbar에 "되돌리기" 버튼 추가 (작업 완료 후 30초 표시)
  - 백엔드 `POST /admin/proposals/batch-rollback` 엔드포인트 필요
  - 이미 개별 처리된 제안은 롤백 불가 (상태 기반 판단)
- **완료 기준**: 일괄 작업 후 30초 내 롤백 가능, 이후 버튼 사라짐

### PH3-CARRY-002: 대화 FTS 전환
- **내용**: `conversations` 테이블의 title 검색을 `ILIKE '%query%'` → `tsvector` + GIN 인덱스로 전환
- **요구사항**:
  - `conversations.title_tsv` tsvector 컬럼 추가 (자동 생성 트리거)
  - GIN 인덱스: `CREATE INDEX CONCURRENTLY idx_conv_title_fts ON conversations USING GIN(title_tsv)`
  - 검색 API: `to_tsquery()` 기반 쿼리로 전환, `ts_rank` 정렬
  - 기존 API 응답 형식 변경 없음 (프론트엔드 수정 불필요)
- **완료 기준**: 1만 건 이상 대화에서 검색 응답 50ms 이하

### PH5-CARRY-003: 1000개 이상 제안 부하 테스트
- **내용**: 배포 직전 실행하는 성능 검증 테스트 스크립트
- **요구사항**:
  - Locust 또는 k6 기반 스크립트 (`scripts/load_test_proposals.py`)
  - 시나리오: 1000개 제안 동시 제출, P95 응답시간 ≤ 2s 확인
  - 결과 리포트 자동 생성
- **완료 기준**: 1000개 동시 제안에서 P95 ≤ 2s, 에러율 ≤ 1%

---

## 4. 기술 설계 요약

### 4.1 평가 엔진 아키텍처
```
GoldenSet (평가 기준)
    │
    ├─ Golden Item 1 (질문, 기대 답변, 기대 근거)
    ├─ Golden Item 2
    └─ ...
    
         ↓ (평가 실행)
         
Evaluator (평가 엔진)
    ├─ Faithfulness: LLM / Rule-based fallback
    ├─ Answer Relevance: Embedding
    ├─ Context Precision: LLM / Rule-based
    ├─ Context Recall: Rule-based (expected_source_docs 비교)
    ├─ Citation-present: Rule-based (5-tuple 탐지)
    └─ Hallucination Rate: Rule-based (citation 없는 부분)
    
         ↓ (결과)
         
EvaluationReport
    ├─ 각 항목별 스코어
    ├─ 요약 통계
    ├─ 프롬프트/모델 메타데이터
    └─ 타임스탬프
```

### 4.2 평가 실행 플로우
1. 사용자가 `POST /api/v1/evaluations/run` 호출
2. 평가 엔진이 각 Golden Item에 대해:
   - 질문을 RAG 시스템에 제시
   - 답변 및 메트릭(지연, 토큰) 수집
   - 각 지표 계산
3. 평가 결과 저장 (DB + TimeSeries)
4. CI 게이트에서 임계값 검사
5. 실패 시 PR 코멘트 및 빌드 실패
6. 성공 시 AI품질평가.md 자동 생성

### 4.3 LLM 불가 환경에서의 폴백
- Faithfulness: 근거와 답변의 텍스트 유사도 (embedding) + keyword overlap
- Context Precision: 근거 문장과 답변 문장의 매칭도
- 결과: 모든 지표를 "보수적"으로 계산. LLM 판정보다 점수가 낮을 수 있음

### 4.4 데이터 모델 통합
```python
class GoldenItem(BaseModel):
    id: UUID
    question: str
    expected_answer: str
    expected_source_docs: List[SourceRef]
    expected_citations: List[Citation5Tuple]  # Phase 2와 동일 구조

class EvaluationResult(BaseModel):
    golden_item_id: UUID
    faithfulness: float
    answer_relevance: float
    context_precision: float
    context_recall: float
    citation_present_rate: float
    hallucination_rate: float
    latency_ms: float
    tokens: TokenMetrics
    estimated_cost: float

class EvaluationReport(BaseModel):
    golden_set_id: UUID
    golden_set_version: int
    prompt_version: str
    model_name: str
    evaluated_at: datetime
    results: List[EvaluationResult]
    summary: Dict[str, float]  # 각 지표의 평균값
```

---

## 5. 의존 관계

### 선행 Phase
- **Phase 0**: 평가 기준값 정의 (Faithfulness ≥0.80 등)
- **Phase 1**: LLM Provider 추상화 (판정 LLM, embedding 모델)
- **Phase 2**: Citation 5-tuple 정의 (expected_citations 필드)
- **Phase 6**: 평가 결과 대시보드 스켈레톤

### 후행 Phase (이 Phase의 산출물을 소비하는 Phase)
- **Phase 8**: 추출 품질 평가 (FG7.2의 평가 러너를 재사용)
- **Phase 9**: S2 통합 회귀 (FG7.3의 CI 게이트 및 AI품질평가.md 템플릿)

### 소급 적용
- Phase 2~5의 AI 기능에도 Phase 7 평가 지표를 소급 적용 가능
- Phase별 AI품질평가.md 자동 생성 가능

---

## 6. 검수 기준 종합

| 항목 | 기준 |
|------|------|
| **FG7.1** | Golden Set CRUD API 정상 작동 / import/export JSON 손실 없음 / 버전 관리 정확 |
| **FG7.2** | 6가지 지표 모두 0.0~1.0 범위 / LLM 판정 불가 환경에서 규칙 기반 대체 동작 / Faithfulness 점수 vs. 실제 hallucination 상관도 검증 / Citation-present rate가 5-tuple 구조 인식 |
| **FG7.3** | CI 게이트 임계값 검사 정상 작동 / PR 코멘트 자동 생성 / AI품질평가.md 포맷 정확 |
| **통합** | 지난 30일 지표 추이가 시각화 가능 (Phase 6 대시보드와 연계) / Golden Set 버전 변경 후 평가 재실행 결과 기록 |

---

## 7. 예상 산출물 목록

| 산출물 | 형태 | 설명 |
|--------|------|------|
| GoldenSet 도메인 모델 | Pydantic | GoldenSet, GoldenItem, 버전 관리 |
| GoldenSet CRUD API | FastAPI | 6개 엔드포인트 |
| import/export 엔드포인트 | FastAPI | JSON 직렬화/역직렬화 |
| Evaluator 클래스 | Python | 6가지 지표 계산 로직 |
| EvaluationResult, Report 모델 | Pydantic | 평가 결과 데이터 구조 |
| 평가 실행 API | FastAPI | run, get, list, compare 엔드포인트 |
| GitHub Actions 워크플로우 | YAML | CI 게이트 설정 |
| 임계값 정의 및 검사 로직 | Python | EVALUATION_THRESHOLDS |
| AI품질평가.md 생성 스크립트 | Python + Jinja2 | 템플릿 및 렌더링 |
| 수동 평가 스크립트 | Python | scripts/evaluate_golden_set.py |
| Phase 7별 평가 지표 계산 테스트 | pytest | 각 지표별 단위 테스트 |
| **FG7.1 검수보고서** | Markdown |  |
| **FG7.1 보안취약점검사보고서** | Markdown | GoldenSet 접근 제어, 버전 무결성 |
| **FG7.2 검수보고서** | Markdown |  |
| **FG7.2 보안취약점검사보고서** | Markdown | 판정 LLM 프롬프트 주입 방지, 메트릭 조작 방지 |
| **FG7.3 검수보고서** | Markdown |  |
| **FG7.3 보안취약점검사보고서** | Markdown | CI 게이트 우회 불가능성 확인 |
| **Phase 7 종결 보고서** | Markdown | 평가 인프라 완성, Phase 6/8/9와의 통합 준비 |

---

## 8. 리스크 및 대응

| 리스크 | 영향도 | 대응 방안 |
|--------|--------|----------|
| LLM 판정의 주관성 (모델마다 다른 평가) | 중간 | 판정 LLM 고정 (예: gpt-4o), 여러 번 실행 후 평균 취득 |
| CI 게이트 임계값 과도로 높게 설정 | 높음 | Phase 0에서 현실적인 목표값 사전 협의, 점진적 상향 전략 |
| 평가 실행 시간 과다 (모든 Golden Item마다 LLM 호출) | 중간 | 캐싱 (동일 질문→동일 답변 caching), 배치 처리, 비동기 실행 |
| 평가 데이터 저장소 부족 | 낮음~중간 | TimeSeries DB 활용 (InfluxDB) 또는 데이터 압축 |
| Golden Set 항목 수 폭증 (관리 부담) | 낮음 | 도메인별 분할 GoldenSet, import 기능으로 대량 관리 |

---

## 9. 선행 조건

Phase 7 착수 전:
- Phase 6 FG6.3 평가 결과 대시보드 스켈레톤 완성
- Phase 1 LLM Provider 추상화 안정화
- Phase 2 Citation 5-tuple 계약 최종 확정
- Golden Set 형식 및 평가 지표에 대한 팀 합의

---

## 10. 완료 기준

Phase 7 완료로 판단하는 조건:

1. GoldenSet 도메인 모델 및 CRUD API 완성
2. 6가지 평가 지표(Faithfulness, Answer Relevance, Context Precision/Recall, Citation-present, Hallucination) 구현 완료
3. LLM 불가 환경에서 규칙 기반 지표 동작 확인
4. 평가 실행 API (run, get, compare) 정상 작동
5. CI 게이트 워크플로우 구축 및 임계값 검사 작동
6. AI품질평가.md 자동 생성 템플릿 작성 및 테스트
7. Phase 6 FG6.3 대시보드와의 데이터 바인딩 완성
8. 평가 성능 시계열 데이터 수집 확인
9. **이월 항목 완료**:
   - PH3-CARRY-002: 대화 FTS 전환 완료 (ILIKE → tsvector)
   - PH6-CARRY-001: Batch 롤백 UI 완료 (30초 Undo)
   - PH5-CARRY-003: 부하 테스트 스크립트 작성 및 결과 보고
10. 모든 FG + 이월 항목 검수보고서 및 보안취약점검사보고서 승인
11. Phase 7 종결 보고서 작성 완료

---

## 11. 권장 투입 순서

1. FG7.1 GoldenSet 모델 정의 및 CRUD API 구축 (선행)
2. import/export 기능 추가 및 테스트
3. FG7.2 평가 지표 구현 (병행: 각 지표별 독립 구현)
   - Answer Relevance (embedding 기반, 의존 최소)
   - Citation-present, Hallucination (규칙 기반)
   - Context Recall (expected_source_docs 비교)
   - Context Precision, Faithfulness (LLM 기반, 폴백 포함)
4. Evaluator 클래스 통합 및 단위 테스트
5. 평가 실행 API 구축
6. FG7.3 CI 게이트 구축
7. AI품질평가.md 생성 스크립트
8. Phase 6 FG6.3 대시보드와의 데이터 바인딩
9. **이월 항목 처리** (FG 완료 후 병행)
   - PH3-CARRY-002: 대화 FTS 전환 (task7-10)
   - PH6-CARRY-001: Batch 롤백 UI (task7-11)
   - PH5-CARRY-003: 부하 테스트 스크립트 작성 (task7-12)
10. 모든 FG + 이월 항목 검수 및 보안 검사
11. Phase 7 종결 보고서 작성

---

## 12. 기대 효과

Phase 7 완료 시:
- RAG 품질이 정량적으로 측정되고, 회귀가 자동으로 탐지됨
- CI 게이트로 인한 품질 보증: 커밋마다 평가 자동 실행
- 프롬프트 A/B 테스트의 성과를 수치로 비교 가능
- 모델 업그레이드의 영향을 사전 평가 가능
- 폐쇄망 배포에서도 (LLM 판정 없이) 품질 게이트 작동
- Phase 6의 평가 대시보드가 실제 데이터로 가동
