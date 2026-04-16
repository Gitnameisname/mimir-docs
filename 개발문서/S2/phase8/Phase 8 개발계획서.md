# Phase 8 개발계획서
## Structured Extraction

---

## 1. 단계 개요

### 단계명
Phase 8. Structured Extraction

### 목적
문서에서 구조화된 데이터를 LLM 보조로 자동 추출하고, 인간 승인 플로우를 거쳐 저장하는 시스템을 구축한다. 이를 통해 Mimir를 "지식 그래프의 씨앗"으로 진화시키고, S3의 AI-native 계획 도구가 활용할 수 있는 정형 데이터 기반을 제공한다.

### 선행 조건
- Phase 0~7 완료
- Phase 1: LLM Provider 추상화 완료 (추출 LLM 선택 가능)
- Phase 2: Citation 5-tuple 계약 확정
- Phase 7: 평가 러너 완료 (FG7.2)
- DocumentType 메타데이터 스키마 정의 완료

### 기대 결과
- DocumentType metadata schema → 추출 타겟 스키마 매핑
- LLM 보조 추출 + 인간 승인 플로우
- 원문 span 역참조 및 재실행 가능성
- Phase 7의 평가 패턴을 추출 품질 평가에 적용
- Phase 6 FG6.3 추출 관리 UI와의 완전한 통합

---

## 2. Phase 8의 역할

Phase 8은 "구조화 데이터 추출 및 검증" Phase이다. 다음을 담당한다:

1. **추출 타겟 정의의 체계화**: DocumentType의 metadata schema를 추출 기준으로 재해석
2. **자동→인간 검증의 연쇄**: 문서 업로드/개정 시 자동 추출, 그 후 인간의 승인/수정 게이트 통과
3. **추출 결과의 검증 가능성**: 원문의 어느 부분에서 추출했는가를 항상 기록
4. **품질 평가 통합**: Phase 7의 골든셋 패턴을 추출에도 적용하여, 추출 정확도 회귀 방지

---

## 3. Feature Group 상세

### FG8.1 — DocumentType metadata schema → 추출 타겟 스키마

#### 목적
각 DocumentType의 metadata schema를 기반으로, 실제 추출 시 어떤 필드를 어떻게 추출할 것인지 정의하는 매핑 계층 제공.

#### 주요 작업 항목

1. **추출 타겟 스키마 모델 정의**
   - DocumentType의 metadata schema와 1:1 매칭
   - 확장: 각 필드별 추출 지침(instruction)을 문서화
   - 예시:
     ```python
     DocumentType: "규정"
     metadata schema: {
       조항번호: string,        # 원문의 조항 번호 필드
       책임자: string,         # "담당: [부서명]" 패턴에서 추출
       발효일: date,           # "발효: YYYY-MM-DD" 형식으로 파싱
       요약: string,           # 해당 조항의 요약 (50 단어 이내)
       관련규정: array(string) # "참고: [규정명]" 또는 "관련: [규정ID]"
     }
     
     ExtractionTargetSchema: {
       조항번호: {
         field_type: "string",
         required: true,
         pattern: "^\\d+\\.\\d+(\\.\\d+)?$",  # 1.2.3 형식
         instruction: "원문의 조항 번호를 정확히 추출. 원문에 없으면 NULL.",
         examples: ["1.2", "1.2.3"]
       },
       책임자: {
         field_type: "string",
         required: true,
         instruction: "\"담당: [부서명]\" 문구에서 부서명 추출. 여러 부서면 쉼표로 구분.",
         examples: ["기술팀", "기술팀, 운영팀"]
       },
       발효일: {
         field_type: "date",
         required: false,
         format: "YYYY-MM-DD",
         instruction: "\"발효: YYYY-MM-DD\" 형식으로 파싱. 없으면 NULL.",
         examples: ["2024-01-01"]
       },
       요약: {
         field_type: "string",
         required: true,
         max_length: 100,
         instruction: "해당 조항의 핵심 내용을 한국어로 50~100 단어로 요약.",
         examples: ["이 조항은 ...을 규정합니다."]
       },
       관련규정: {
         field_type: "array(string)",
         required: false,
         instruction: "\"참고: [규정명]\" 또는 \"[규정ID]\" 형식의 참조를 모두 추출. 배열로 반환.",
         examples: [["규정A", "규정B"]]
       }
     }
     ```

2. **추출 타겟 스키마 CRUD API**
   - `POST /api/v1/extraction-schemas` — 새로운 스키마 정의
   - `GET /api/v1/extraction-schemas/{doc_type}` — DocumentType별 스키마 조회
   - `PUT /api/v1/extraction-schemas/{doc_type}` — 스키마 수정 (새 버전 생성)
   - `DELETE /api/v1/extraction-schemas/{doc_type}` — 스키마 삭제 (기존 추출 결과는 유지)
   - `GET /api/v1/extraction-schemas/{doc_type}/versions` — 버전 이력
   
3. **스키마 정의 보조 도구**
   - UI (Phase 6 FG6.3): DocumentType 선택 → metadata schema 자동 로드 → 각 필드별 추출 지침 편집
   - 시각 빌더: 필드 타입 선택, required/optional, 예시 추가
   - 텍스트 편집 모드: JSON으로 직접 편집

4. **스키마 버전 관리**
   - ExtractionTargetSchema도 Document처럼 versioned 리소스
   - 스키마 변경 시 이전 버전의 추출 결과와 비교 가능
   - 마이그레이션 로직: 기존 추출 결과를 새 스키마로 변환할 때의 매핑

#### 입력 (선행 의존)
- S1에서 정의한 DocumentType 메타데이터 스키마
- Phase 1 LLM Provider (선택사항: 스키마 정의 보조)

#### 출력 (산출물)
- ExtractionTargetSchema 도메인 모델 (Pydantic)
- CRUD API (FastAPI)
- 스키마 버전 관리 로직
- 스키마 검증 (각 필드별 constraints)

#### 검수 기준
- 각 DocumentType의 metadata schema가 추출 타겟으로 정확히 매핑되는가
- 스키마 버전 변경 후 기존 추출 결과와의 호환성이 명확한가
- 추출 지침이 LLM이 이해 가능할 정도로 명확한가

---

### FG8.2 — LLM 보조 추출 + 인간 승인

#### 목적
문서가 업로드/개정되면 자동으로 추출을 시도하고, 인간이 결과를 검토 후 승인하거나 수정하는 엔드-투-엔드 플로우 구축.

#### 주요 작업 항목

1. **자동 추출 파이프라인**
   - 트리거: 문서 생성/개정 시 (`POST /documents` 또는 `PUT /documents/{id}`)
   - 처리 흐름:
     ```
     Document 업로드
         ↓
     DocumentType 판정 (메타데이터의 type 필드)
         ↓
     ExtractionTargetSchema 로드
         ↓
     LLM 프롬프트 구성 + 호출 (Phase 1 LLM Provider)
         ↓
     추출 결과 생성 (ExtractionCandidate)
         ↓
     큐(pending review)에 저장
         ↓
     사용자 알림 (메뉴: "내 문서에 대한 추출 결과 검토 대기중")
     ```

2. **LLM 추출 프롬프트**
   - 구조화:
     ```
     당신은 정보 추출 AI입니다.
     
     다음 스키마에 따라 문서에서 정보를 추출하세요:
     
     스키마:
     {extraction_target_schema_json}
     
     지침:
     - 각 필드는 원문에서만 추출하세요. 추론하지 마세요.
     - 필드가 필수(required)인데 원문에 없으면 null을 반환하세요.
     - 날짜 필드는 {format} 형식으로 변환하세요.
     - 배열 필드는 JSON 배열로 반환하세요.
     
     원문:
     {document_text}
     
     JSON 포맷으로 추출 결과를 반환하세요:
     ```
   - Phase 1의 모델 추상화 사용 (gpt-4o / Anthropic Claude / 폐쇄망 모델)

3. **ExtractionCandidate 도메인 모델**
   ```python
   class ExtractionCandidate(BaseModel):
       id: UUID
       document_id: UUID
       document_version: int
       extraction_schema_version: int
       
       extracted_fields: Dict[str, Any]  # 추출된 값 {field_name: value}
       confidence_scores: Dict[str, float]  # 각 필드별 신뢰도 (선택)
       
       status: Enum["pending", "approved", "rejected", "modified"]
       
       created_at: datetime
       extraction_model: str  # 사용한 모델명
       extraction_latency_ms: int
       
       # 인간의 수정/승인
       reviewed_by: Optional[user_id]
       reviewed_at: Optional[datetime]
       human_feedback: Optional[str]
       approved_fields: Optional[Dict[str, Any]]  # 인간이 수정한 부분
   ```

4. **추출 결과 검토 플로우**
   - 사용자는 "내 문서에 대한 추출 결과 검토" 뷰 접근
   - 문서별 미승인 추출 후보 목록 표시
   - 각 후보에 대해:
     - 원본 문서 미리보기 (읽기 전용)
     - 추출된 필드 테이블 (자동 추출값, 신뢰도)
     - 각 필드별 편집 버튼:
       - 자동 추출값 유지: "승인"
       - 수정: 값 변경 후 "수정하여 승인"
       - 삭제: "거절" (또는 필드별 삭제)
   - "모두 승인" / "모두 거절" (일괄 작업)

5. **추출 결과 저장**
   - 승인 후:
     - ExtractionCandidate를 최종 상태(`approved` / `modified`)로 저장
     - 추출 결과는 **별도 artifact**로 관리: `ApprovedExtraction` 테이블에 `document_id + version_id + extraction_schema_version` 복합 키로 저장
       - Document.metadata에는 직접 통합하지 않음 (DB 모델 generic 유지 원칙 준수)
       - 필요 시 **projection / materialized view**로만 metadata에 반영
     - 추출 스키마 버전, 모델 버전, 인간 수정 이력을 독립 추적 가능하도록 artifact에 다음 필드 포함:
       - `extraction_schema_version`: 사용된 스키마 버전
       - `extraction_model`: 추출에 사용된 모델명 및 버전
       - `human_edits`: 인간이 수정한 필드 목록과 수정 전/후 값
       - `approved_at`, `approved_by`
     - 감사 로그: `action=extraction_approved, approved_by={user_id}, artifact_id={approved_extraction_id}`
   - 거절 후:
     - ExtractionCandidate를 `rejected` 상태로 표시
     - 사용자가 수동 입력 옵션 제공

6. **추출 재실행**
   - 스키마 변경 후 기존 문서들의 추출 재실행 (배치)
   - 새 모델 배포 후 샘플 재추출 (비교 평가)
   - API: `POST /api/v1/extractions/batch-retry` — 특정 DocumentType의 모든 승인된 문서 재추출

#### 입력 (선행 의존)
- Phase 1 FG1.1: LLM Provider Abstraction
- FG8.1: ExtractionTargetSchema
- Document 및 DocumentType 도메인 모델
- User UI 페이지 (추출 결과 검토 뷰)

#### 출력 (산출물)
- 자동 추출 파이프라인 (비동기 task queue)
- LLM 추출 프롬프트 템플릿 (Jinja2)
- ExtractionCandidate 도메인 모델
- 추출 결과 검토 API (승인/거절/수정)
- 추출 결과 검토 UI (User UI 페이지)
- 추출 재실행 배치 스크립트

#### 검수 기준
- 추출 결과가 ExtractionTargetSchema 필드에 모두 매핑되는가
- 추출 재실행 후 신뢰도 변화가 추적되는가
- 사용자 수정 후 ApprovedExtraction artifact가 정확히 저장되는가 (Document.metadata 직접 통합 아님)
- projection/materialized view 경유 시 metadata 반영이 정확한가
- 동시성 문제 없음 (여러 사용자가 동일 문서 추출 결과 수정 시)

---

### FG8.3 — 추출 결과의 검증 가능 계약

#### 목적
추출된 데이터가 원문의 어디에서 왔는지 추적 가능하고, 동일 모델 + 동일 프롬프트 + 동일 문서 입력 시 재현 가능하도록 설계.

#### 주요 작업 항목

1. **원문 Span 역참조 (Source Attribution)**
   - 각 추출된 필드에 대해 원문의 위치 기록:
     ```python
     class ExtractedFieldWithAttribution(BaseModel):
         field_name: str
         extracted_value: Any
         source_spans: List[SourceSpan] = []  # 복수 span 가능
     
     class SourceSpan(BaseModel):
         document_id: UUID
         version_id: int
         node_id: UUID
         span_offset: Tuple[int, int]  # (start, end) character offset
         source_text: str  # 실제 추출된 텍스트
         content_hash: str  # 원본 무결성 검증
     ```
   - LLM 추출 프롬프트에 "추출한 텍스트의 위치(단락 번호 또는 키)를 반드시 명시하세요" 지침 추가
   - LLM이 반환한 위치 정보를 기반으로 자동으로 span 계산

2. **재실행 가능성 (Deterministic Extraction Mode)**
   - 추출 조건을 모두 기록:
     ```python
     class ExtractionRecord(BaseModel):
         document_id: UUID
         document_version: int
         document_content_hash: str
         
         extraction_schema_id: str
         extraction_schema_version: int
         
         extraction_model: str  # 모델명
         model_version: str  # 모델 버전 (예: gpt-4o-2024-11-20)
         extraction_prompt_version: str  # 프롬프트 template version
         
         extraction_mode: Enum["deterministic", "probabilistic"]
         temperature: float = 0.0 if deterministic else 0.7
         
         extracted_result: Dict[str, Any]
         extracted_timestamp: datetime
     ```
   - 동일 조건으로 재실행하면 동일 결과 재현 (deterministic mode에서)
   - 평가 시 이 정보 활용: "이 추출이 정말 재현 가능한가?" 검증

3. **추출 품질 평가 (Phase 7 패턴 적용)**
   - 평가용 GoldenSet: (문서, 기대 추출값, 기대 span)의 tuple
   - 평가 지표:
     - **Field Accuracy**: 각 필드의 추출값이 기대값과 일치하는 비율
     - **Span Accuracy**: 추출된 span이 기대 span과 일치하는 비율 (또는 overlap IoU)
     - **Required Field Coverage**: 필수 필드가 모두 추출된 비율
     - **Type Correctness**: 추출된 값의 타입이 스키마와 일치하는 비율
     - **Latency, Tokens, Cost**: Phase 7과 동일
   - 평가 러너: Phase 7의 Evaluator 클래스 확장
     ```python
     class ExtractionEvaluator(Evaluator):
         def evaluate_extraction(self, golden_item: GoldenExtractionItem,
                                extracted: ExtractionResult) -> ExtractionEvaluationResult:
             return ExtractionEvaluationResult(
                 field_accuracy=self._compute_field_accuracy(golden_item, extracted),
                 span_accuracy=self._compute_span_accuracy(golden_item, extracted),
                 coverage=self._compute_required_field_coverage(extracted),
                 type_correctness=self._compute_type_correctness(extracted),
                 latency_ms=extracted.latency_ms,
                 tokens=extracted.tokens,
                 estimated_cost=self._compute_cost(extracted.tokens)
             )
     ```

4. **추출 재현성 검증 API**
   - `POST /api/v1/extractions/{extraction_id}/verify` — 추출 재현성 검증
     - 동일 문서, 동일 모델, 동일 프롬프트로 재추출
     - 이전 추출과 비교
     - 결과: "일치", "부분일치", "불일치" + 상세 diff
   - `GET /api/v1/extractions/{extraction_id}/audit` — 추출 감시 정보 (모델, 프롬프트, 타임스탐프)

#### 입력 (선행 의존)
- Phase 2 FG2.1: Citation 5-tuple (span 역참조와 동일 개념)
- Phase 7 FG7.2: 평가 러너 (평가 로직 재사용)
- FG8.2: 추출 결과 및 LLM 출력
- Document Node 및 Chunking 정보

#### 출력 (산출물)
- SourceSpan 도메인 모델
- ExtractedFieldWithAttribution 모델 (span 정보 포함)
- ExtractionRecord (추출 조건 전체 기록)
- 추출 재현성 검증 API
- GoldenExtractionItem 모델 (평가용)
- ExtractionEvaluator 클래스 (Phase 7 확장)
- 추출 품질 평가 보고서 생성 로직

#### 검수 기준
- 추출된 필드의 span이 원문과 정확히 매칭되는가
- 동일 조건으로 재추출 시 결과가 재현되는가 (deterministic mode)
- 추출 품질 평가가 실제 정확도와 상관도를 가지는가
- 복수 span 필드 (예: 관련규정 배열)의 attribution이 정확한가

---

## 4. 기술 설계 요약

### 4.1 추출 시스템 아키텍처
```
DocumentType
    │
    ├─ metadata schema (S1 정의)
    │
    ▼
ExtractionTargetSchema (이 Phase에서 정의)
    │
    ├─ field1: {type, required, instruction, ...}
    ├─ field2: {...}
    └─ ...
    
         ↓ (문서 업로드 시)
         
Document
    │
    ├─ content
    └─ type → ExtractionTargetSchema 로드
    
         ↓ (LLM 호출)
         
LLM (Phase 1 Provider)
    │
    ├─ prompt: "다음 스키마에 따라 추출하세요..."
    ├─ input: document_text + schema_json
    └─ output: json 형식 추출 결과
    
         ↓
         
ExtractionCandidate (자동 추출값)
    │
    ├─ extracted_fields: {field1: value1, ...}
    ├─ source_spans: [span1, span2, ...]
    └─ status: pending
    
         ↓ (인간 검토)
         
사용자 (검토 UI)
    │
    ├─ 원문 미리보기
    ├─ 자동 추출값 확인
    └─ 승인 / 수정 / 거절
    
         ↓
         
ApprovedExtraction artifact (최종 저장, Document.metadata와 분리)
    ├─ {field1: approved_value1, ...}
    ├─ extraction_schema_version, extraction_model, human_edits
    └─ projection/materialized view로 metadata 반영 가능
```

### 4.2 추출 프롬프트 엔지니어링
- 기본 프롬프트: 스키마 정의 + 지침 + 예시
- 최적화: few-shot 예시 (GoldenExtractionItem 활용)
- 프롬프트 버전 관리: Phase 1 Prompt Registry와 동일

### 4.3 비동기 처리
- 문서 업로드 시 자동 추출은 비동기 task로 실행
- Task queue (Celery 또는 Python asyncio) 사용
- 사용자에게 "추출 진행 중" 알림 표시

### 4.4 스키마 마이그레이션
- 스키마 버전 변경 시:
  1. 기존 추출 결과는 구 스키마 버전과 매핑된 상태로 유지
  2. 배치 재추출 옵션: 기존 문서들을 새 스키마로 재추출
  3. 필드 매핑 도구: 구 필드 → 신 필드 자동 변환 규칙 정의

---

## 5. 의존 관계

### 선행 Phase
- **Phase 0**: S2 원칙 정의
- **Phase 1**: LLM Provider 추상화 (추출 모델)
- **Phase 2**: Citation 5-tuple (span 역참조와 유사)
- **Phase 7**: 평가 러너 (추출 품질 평가)

### 후행 Phase (이 Phase의 산출물을 소비하는 Phase)
- **Phase 9**: 추출 품질 평가를 S2 통합 회귀에 포함
- **S3**: 추출된 구조화 데이터를 AI-native 계획 도구의 입력으로 활용

---

## 6. 검수 기준 종합

| 항목 | 기준 |
|------|------|
| **FG8.1** | 모든 DocumentType의 metadata schema가 ExtractionTargetSchema로 정확히 매핑되는가 / 스키마 버전 관리가 Document와 동일하게 작동하는가 |
| **FG8.2** | 문서 업로드 시 자동 추출이 정상 실행되는가 / 추출 결과가 pending 상태로 사용자 큐에 들어가는가 / 사용자 승인 후 ApprovedExtraction artifact에 정확히 저장되는가 (Document.metadata 직접 통합 아님) / projection/materialized view 경유 시 metadata 반영이 정확한가 / 사용자가 자동 추출값을 수정 후 승인 가능한가 |
| **FG8.3** | 추출된 필드의 span이 원문과 정확히 매칭되는가 / deterministic mode에서 재추출 시 동일 결과 재현되는가 / 추출 품질 평가 지표가 실제 정확도와 상관 / 복수 span 필드 처리 정확성 |
| **통합** | Phase 6 FG6.3 추출 관리 UI와 완전히 연동되는가 / Phase 7 평가 인프라와 통합되는가 |

---

## 7. 예상 산출물 목록

| 산출물 | 형태 | 설명 |
|--------|------|------|
| ExtractionTargetSchema 도메인 모델 | Pydantic | DocumentType별 추출 스키마 정의 |
| ExtractionTargetSchema CRUD API | FastAPI | 스키마 CRUD 및 버전 관리 |
| LLM 추출 프롬프트 템플릿 | Jinja2 | 스키마 기반 동적 프롬프트 |
| 자동 추출 파이프라인 | Python (비동기) | 문서 업로드 시 트리거, LLM 호출 |
| ExtractionCandidate 도메인 모델 | Pydantic | 추출 결과 및 검토 상태 |
| SourceSpan 도메인 모델 | Pydantic | 원문 역참조 (span 좌표 + content_hash) |
| 추출 결과 검토 API | FastAPI | 승인/거절/수정 엔드포인트 |
| 추출 결과 검토 UI | TypeScript/React | User UI 페이지 |
| 추출 재실행 배치 스크립트 | Python | 스키마 변경 후 대량 재추출 |
| ExtractionRecord (감시 정보) | Pydantic | 모델, 프롬프트, 타임스탐프 기록 |
| 추출 재현성 검증 API | FastAPI | verify, audit 엔드포인트 |
| GoldenExtractionItem 모델 | Pydantic | 평가용 Golden Set (문서, 기대 추출값) |
| ExtractionEvaluator 클래스 | Python | Phase 7 Evaluator 확장, 추출 지표 계산 |
| 추출 품질 평가 보고서 생성 | Python + Jinja2 | 마크다운 보고서 자동 생성 |
| **FG8.1 검수보고서** | Markdown |  |
| **FG8.1 보안취약점검사보고서** | Markdown | 스키마 정의 보안, 버전 무결성 |
| **FG8.2 검수보고서** | Markdown |  |
| **FG8.2 보안취약점검사보고서** | Markdown | LLM 프롬프트 주입 방지, 동시성 문제 |
| **FG8.3 검수보고서** | Markdown |  |
| **FG8.3 보안취약점검사보고서** | Markdown | span 역참조 무결성, 재현성 보증 |
| **Phase 8 종결 보고서** | Markdown | 구조화 추출 완성, S3 인계 준비 |

---

## 8. 리스크 및 대응

| 리스크 | 영향도 | 대응 방안 |
|--------|--------|----------|
| LLM 추출의 일관성 부족 (모델마다 다른 추출) | 중간~높음 | few-shot 예시 추가, temperature=0 (deterministic), 재현성 검증 API로 모니터링 |
| 인간 승인 병목 (대량 문서 추출 시 검토 시간) | 중간 | 신뢰도 스코어 기반 자동 승인 임계값 제공, 배치 승인 UI |
| Span 계산 오류 (OCR 왜곡, 특수문자 등) | 낮음~중간 | 추출 후 span 검증, 사용자 수정 시 span도 함께 갱신 |
| 스키마 버전 변경 시 기존 추출 호환성 | 중간 | 명확한 마이그레이션 경로 정의, 구 버전 추출 유지 |
| 추출 성능 (문서 크기, 모델 호출) | 낮음~중간 | 청크 단위 추출, 캐싱, 비동기 처리로 UI 블로킹 방지 |

---

## 9. 선행 조건

Phase 8 착수 전:
- Phase 7 완료 (평가 러너 필요)
- 모든 DocumentType의 metadata schema 최종 확정
- Phase 6 FG6.3 추출 관리 UI 스켈레톤 완성
- LLM 추출 프롬프트 엔지니어링 사전 연구

---

## 10. 완료 기준

Phase 8 완료로 판단하는 조건:

1. ExtractionTargetSchema 모델 및 CRUD API 완성
2. 모든 DocumentType에 대해 추출 스키마 정의 완료
3. LLM 기반 자동 추출 파이프라인 정상 작동
4. 문서 업로드 시 자동 추출 트리거 및 비동기 처리
5. 사용자 검토 UI (승인/수정/거절) 완성
6. 사용자 수정 후 Document metadata 저장 확인
7. SourceSpan 역참조 정보 완벽히 기록
8. deterministic mode에서 추출 재현성 검증 성공
9. 추출 품질 평가 지표 구현 완료 (Phase 7 확장)
10. Phase 6 FG6.3 추출 관리 UI와의 완전한 데이터 바인딩
11. 모든 FG 검수보고서 및 보안취약점검사보고서 승인
12. Phase 8 종결 보고서 작성 완료

---

## 11. 권장 투입 순서

1. FG8.1 ExtractionTargetSchema 모델 정의 및 CRUD API (선행)
2. 모든 DocumentType에 대한 추출 스키마 정의 (병행: 도메인 전문가 참여)
3. FG8.2 LLM 추출 프롬프트 개발 및 테스트
4. 자동 추출 파이프라인 구축 (비동기 처리)
5. ExtractionCandidate 모델 및 검토 큐 API
6. 추출 결과 검토 UI (User UI) 구축
7. FG8.3 SourceSpan 역참조 정보 수집 및 저장
8. 추출 재현성 검증 API 구축
9. 추출 품질 평가 (GoldenExtractionItem, ExtractionEvaluator)
10. Phase 6 FG6.3 추출 관리 UI 최종 연결
11. 모든 FG 검수 및 보안 검사
12. Phase 8 종결 보고서 작성

---

## 12. 기대 효과

Phase 8 완료 시:
- 문서에서 구조화 데이터가 자동 추출되고 인간 검증을 거침
- 추출 결과가 원문과 추적 가능하고 재현 가능 (검증 가능한 계약)
- Mimir가 지식 그래프의 씨앗이 되어, S3 AI-native 계획 도구의 기반 제공
- 추출 품질이 Phase 7의 평가 인프라로 모니터링되고, 회귀 방지
- 폐쇄망 환경에서도 로컬 LLM으로 추출 가능 (모델 swappable)
- 새로운 DocumentType 추가 시 추출 스키마만 정의하면 자동 파이프라인 적용 가능 (generic + config 원칙)
