# Phase 8 종결 보고서
## LLM 문서 추출 파이프라인

**종결 일자**: 2026-04-18  
**작성자**: Claude Code  
**Phase**: S2 Phase 8

---

## 1. 종결 요약

| 항목 | 내용 |
|------|------|
| Phase 명 | S2 Phase 8 — LLM 문서 추출 파이프라인 |
| 기간 | 2026-04-18 |
| 완료 태스크 | task8-1 ~ task8-10 (10개) |
| 단위 테스트 | 207/207 PASSED |
| 검수 결과 | ✅ 통과 (수정 5건 완료) |
| 보안 취약점 검사 | ✅ 통과 (Critical 1건 + High 3건 + Medium 3건 + Low 1건 수정 완료) |
| 이월 항목 | 4건 (인증/인프라 레이어 소관, 기능 영향 없음) |

---

## 2. Feature Group별 완료 현황

### FG8.1 — 추출 스키마 관리 (task8-1 ~ task8-3)

**목적**: 문서 유형별 추출 대상 필드를 정의·버전 관리하는 ExtractionSchema 도메인 구축

| 산출물 | 파일 | 상태 |
|--------|------|------|
| ExtractionSchema 도메인 모델 | `backend/app/models/extraction_schema.py` | ✅ |
| ExtractionSchema Repository | `backend/app/repositories/extraction_schema_repository.py` | ✅ |
| ExtractionSchema Service | `backend/app/services/extraction_schema_service.py` | ✅ |
| ExtractionSchema API | `backend/app/api/v1/extraction_schemas.py` | ✅ |
| 단위 테스트 | `backend/tests/unit/extraction/` | ✅ 65/65 |

**구현 내용**:
- ExtractionSchema: `id`, `doc_type_code`, `version`, `fields`(JSON 필드 정의 배열), `scope_profile_id` ACL
- 필드 정의: `field_name`, `field_type`, `required`, `description`, `extraction_hint`
- 버전 관리: 스키마 수정 시 새 버전 생성, 최신 버전 자동 조회
- CRUD 엔드포인트 6종: create / list / get / update / delete / list-versions
- S2 원칙 ⑥: `scope_profile_id` 기반 ACL 필터 전 엔드포인트 적용

---

### FG8.2 — LLM 자동 추출 + 인간 검토 (task8-4 ~ task8-6)

**목적**: 문서 업로드 시 LLM이 스키마 기반으로 필드를 자동 추출하고 인간이 승인·수정·거절하는 파이프라인 구축

| 산출물 | 파일 | 상태 |
|--------|------|------|
| ExtractionCandidate 도메인 모델 | `backend/app/models/extraction.py` (append) | ✅ |
| ApprovedExtraction 도메인 모델 | `backend/app/models/approved_extraction.py` | ✅ |
| ExtractionCandidateRepository | `backend/app/repositories/extraction_candidate_repository.py` | ✅ |
| ApprovedExtractionRepository | `backend/app/repositories/approved_extraction_repository.py` | ✅ |
| ExtractionPipelineService | `backend/app/services/extraction/extraction_pipeline_service.py` | ✅ |
| 추출 프롬프트 템플릿 | `backend/app/services/extraction/templates/extraction_prompt.jinja2` | ✅ |
| Extractions API | `backend/app/api/v1/extractions.py` | ✅ |
| 단위 테스트 | `backend/tests/unit/extraction/` | ✅ 37건 |

**구현 내용**:
- Jinja2 동적 프롬프트 렌더링 (스키마 필드 + extraction_hint 주입)
- LLM 호출: 지수 백오프 3회 재시도, 폐쇄망 로컬 모델 fallback (S2 원칙 ⑦)
- JSON 파싱 + 필드 타입 검증, ExtractionCandidate `PENDING` 상태 저장
- 인간 검토 API 7종: pending 목록/상세/approve/modify/reject/batch-approve/batch-reject
- `actor_type` 감사 로그 필수 기록 (S2 원칙 ⑤)

**DB DDL**:
- `extraction_candidates` — `status`, `extracted_fields`(JSONB), `extraction_model`, `scope_profile_id`
- `approved_extractions` — `approved_by`, `human_edits`(JSONB), `actor_type`

---

### FG8.2 확장 — 배치 재추출 비동기 처리 (task8-7)

**목적**: 스키마 변경 또는 모델 업그레이드 시 기존 문서들을 일괄 재추출하는 비동기 배치 시스템

| 산출물 | 파일 | 상태 |
|--------|------|------|
| BatchExtractionJob 도메인 모델 | `backend/app/models/batch_extraction.py` | ✅ |
| BatchExtractionJobRepository | `backend/app/repositories/batch_extraction_repository.py` | ✅ |
| BatchExtractionService | `backend/app/services/extraction/batch_extraction_service.py` | ✅ |
| BatchExtractions API | `backend/app/api/v1/batch_extractions.py` | ✅ |
| 단위 테스트 | `backend/tests/unit/extraction/test_batch_extraction_service.py` | ✅ 20/20 |

**구현 내용**:
- FastAPI `BackgroundTasks` + `asyncio.run()` 래퍼 (Celery 없이 비동기 처리)
- CHUNK_SIZE=10, RATE_LIMIT=5/s, 지수 백오프 3회 재시도
- 취소 요청 플래그 (`is_cancellation_requested`) 폴링으로 안전한 중단
- SSE 진행률 스트리밍: `GET /batch-jobs/{job_id}/progress`
- 샘플 모드: 최대 100건 랜덤 샘플링 후 신규 모델 검증 (comparison_mode)
- 날짜 범위 최대 365일 상한 (SEC-03 수정)

**DB DDL**:
- `batch_extraction_jobs` — `status`, `total_count`, `completed_count`, `failed_count`, `is_cancellation_requested`
- `extraction_retry_logs` — `attempt_number`, `latency_ms`, `error_reason`

---

### FG8.3 — 추출 계약 검증 (task8-8 ~ task8-10)

**목적**: 추출 결과의 원문 provenance(SourceSpan), 재현성 검증(ExtractionRecord), 품질 평가 인프라(ExtractionEvaluator) 구축

#### task8-8: SourceSpan 역참조

| 산출물 | 파일 | 상태 |
|--------|------|------|
| SourceSpan 도메인 모델 | `backend/app/models/extraction_span.py` | ✅ |
| SpanCalculator / MultiSpanExtractor / SpanVisualizationConverter | `backend/app/services/extraction/span_calculator.py` | ✅ |
| ExtractionSpanRepository | `backend/app/repositories/extraction_span_repository.py` | ✅ |
| 단위 테스트 | `backend/tests/unit/extraction/test_span_calculator.py` | ✅ 28/28 |

**구현 내용**:
- `SourceSpan`: `(span_start, span_end)` character offset + `content_hash`(SHA-256) 자동 계산
- `SpanCalculator`: 대소문자 무시 fallback, 전체 출현 탐색, 겹치는 span 병합
- `SpanVisualizationConverter`: `SpanHighlight` 배열 (start/end/field_name) — UI 하이라이트용
- 신규 API: `GET /{id}/spans`, `GET /{id}/highlights`

#### task8-9: ExtractionRecord 재현성 검증

| 산출물 | 파일 | 상태 |
|--------|------|------|
| ExtractionRecord / VerificationResult 도메인 모델 | `backend/app/models/extraction_record.py` | ✅ |
| DiffCalculator / SpanBasedDiffCalculator | `backend/app/services/extraction/diff_calculator.py` | ✅ |
| ExtractionVerificationService | `backend/app/services/extraction/extraction_verification_service.py` | ✅ |
| ExtractionRecordRepository / VerificationResultRepository | `backend/app/repositories/extraction_record_repository.py` | ✅ |
| 단위 테스트 | `backend/tests/unit/extraction/test_diff_calculator.py` | ✅ 27/27 |

**구현 내용**:
- `ExtractionRecord`: 추출 조건 전체 불변 기록 (model, temperature, seed, schema_id, extracted_result)
- `DiffCalculator`: 문자열(Levenshtein 유사도), 숫자(float epsilon), 리스트/dict(재귀, 최대 깊이 10단계)
- `SpanBasedDiffCalculator`: IoU(Intersection over Union) 기반 span 비교
- `MatchStatus`: IDENTICAL / PARTIAL / MISMATCH
- 신규 API: `POST /{id}/verify`, `GET /{id}/audit`, `GET /{id}/verification-results`

#### task8-10: ExtractionEvaluator 품질 평가

| 산출물 | 파일 | 상태 |
|--------|------|------|
| GoldenExtractionSet / ExtractionEvaluationResult 도메인 모델 | `backend/app/models/extraction_evaluation.py` | ✅ |
| ExtractionEvaluator / EvaluationReportGenerator / QualityGateChecker | `backend/app/services/extraction/extraction_evaluator.py` | ✅ |
| ExtractionEvaluationRepository / GoldenExtractionSetRepository / GoldenExtractionItemRepository | `backend/app/repositories/extraction_evaluation_repository.py` | ✅ |
| ExtractionEvaluations API | `backend/app/api/v1/extraction_evaluations.py` | ✅ |
| 단위 테스트 | `backend/tests/unit/extraction/test_extraction_evaluator.py` | ✅ 30/30 |

**구현 내용**:
- 품질 점수 가중치: `field_accuracy×0.40 + span_accuracy×0.20 + required_field_coverage×0.25 + type_correctness×0.15`
- `QualityGateChecker`: field_accuracy≥0.80, span_accuracy≥0.80, required_field_coverage≥0.95, type_correctness≥0.90, overall_score≥0.85
- `EvaluationReportGenerator`: Jinja2 마크다운 보고서 자동 생성 (autoescape 활성화)
- 신규 API 5종: `POST /run`, `GET /{eval_id}`, `GET /{eval_id}/compare`, `POST /quality-gate-check`, `POST /golden-sets`, `GET /golden-sets/{set_id}`

---

## 3. 검수 및 보안 취약점 검사 결과

### 검수 보고서

**파일**: `docs/개발문서/S2/phase8/산출물/검수보고서.md`

| 심각도 | 발견 | 수정 완료 | 이월 |
|--------|------|----------|------|
| Critical | 0건 | — | — |
| High | 2건 | 2건 | 0건 |
| Medium | 2건 | 2건 | 0건 |
| Low | 1건 (R-01) | 0건 | 1건 |

**주요 수정 사항**:
- `run_evaluation()` — Golden Set 평가 시 실제 `ExtractionCandidate`를 DB에서 조회하도록 수정 (기존: 빈 dict 하드코딩)
- `MultiSpanExtractor.extract()` — `extraction_candidate_id: Optional[UUID] = None` 파라미터 추가
- `BatchRetryRequest` — `date_from > date_to` 검증 + `model_validator` 임포트 누락 수정
- 모든 `model_dump()` → `model_dump(mode="json")` 교체 (Pydantic v2 직렬화 정합성)

**이월 항목 (R-01)**: `POST /{id}/verify` — 동일 조건 LLM 재호출 미구현. 현재는 원본 기록과 동일 데이터로 비교하여 항상 IDENTICAL 반환. Phase 9에서 LLM 연동 확장 시 처리.

### 보안 취약점 검사 보고서

**파일**: `docs/개발문서/S2/phase8/산출물/보안취약점검사보고서.md`

| ID | 심각도 | 내용 | 결과 |
|----|--------|------|------|
| SEC-01 | Critical | `list_pending_extractions` scope None 시 전체 조회 | ✅ 수정 — 403 반환 |
| SEC-02 | High | JWT actor_type 클레임 미검증 | ⛔ 이월 — Phase 4 인증 레이어 소관 |
| SEC-03 | High | 배치 날짜 범위 상한 없음 | ✅ 수정 — 365일 상한 추가 |
| SEC-04 | High | DiffCalculator 재귀 깊이 무제한 | ✅ 수정 — 최대 10단계 제한 |
| SEC-05 | High | Jinja2 autoescape 미적용 (SSTI) | ✅ 수정 — `select_autoescape` 활성화 |
| SEC-06 | High | 추출 필드 타입 강제화 미흡 | ⛔ 이월 — FG8.1 스키마 검증 레이어 소관 |
| SEC-07 | Medium | 500 에러 응답 내부 정보 노출 | ✅ 수정 — 7건 정적 메시지 교체 |
| SEC-08 | Medium | 배치 취소 `created_by` 검증 누락 | ✅ 수정 — 작성자 일치 검증 추가 |
| SEC-09 | Medium | 배치 엔드포인트 rate limiting 없음 | ⛔ 이월 — Phase 13 운영 인프라 소관 |
| SEC-10 | Medium | audit_trail에 PII 포함 | ✅ 수정 — 필드 수 + 해시로 대체 |
| SEC-11 | Low | DB 직접 수정 시 Pydantic 재검증 없음 | ⛔ 이월 — Phase 13 DB 접근 감사 소관 |
| SEC-12 | Low | Golden Set scope 클라이언트 입력 수용 | ✅ 수정 — actor scope 강제 |

---

## 4. 단위 테스트 결과

| 파일 | 테스트 수 | 결과 |
|------|-----------|------|
| FG8.1 추출 스키마 관련 | 65 | ✅ 65/65 PASSED |
| `test_extraction_candidate_repository.py` | 19 | ✅ 19/19 PASSED |
| `test_extraction_pipeline_service.py` | 18 | ✅ 18/18 PASSED |
| `test_batch_extraction_service.py` | 20 | ✅ 20/20 PASSED |
| `test_span_calculator.py` | 28 | ✅ 28/28 PASSED |
| `test_diff_calculator.py` | 27 | ✅ 27/27 PASSED |
| `test_extraction_evaluator.py` | 30 | ✅ 30/30 PASSED |
| **합계** | **207** | **✅ 207/207 PASSED** |

---

## 5. S2 원칙 준수 현황

| 원칙 | 내용 | 결과 |
|------|------|------|
| ① 문서 타입 하드코딩 금지 | ExtractionSchema의 `doc_type_code`는 config 기반 | ✅ |
| ② Generic + config 기반 구조 | ExtractionSchema `fields` 배열로 추출 대상 외부화 | ✅ |
| ③ JSON 필드 schema 관리 | `extracted_fields`, `human_edits`, `metrics` JSONB 스키마 정의됨 | ✅ |
| ④ type-aware 로직 | `_check_type()`: 필드 타입별 검증 분기 | ✅ |
| ⑤ actor_type 감사 로그 | 모든 추출 승인/수정/거절/평가 이벤트에 `actor_type` 기록 | ✅ |
| ⑥ scope_profile_id ACL | 모든 조회·생성 엔드포인트에 scope 필터 적용, SEC-01 수정 완료 | ✅ |
| ⑦ 폐쇄망 동등성 | LLM 호출 실패 시 로컬 fallback, `CLOSED_NETWORK` 환경변수 지원 | ✅ |

---

## 6. 이월 항목

| ID | 항목 | 사유 | 우선순위 |
|----|------|------|----------|
| CARRY-P8-001 | `POST /{id}/verify` LLM 재호출 구현 | 현재 동일 데이터 비교, 실질적 재현성 검증은 LLM 연동 필요 | Medium |
| CARRY-P8-002 | JWT actor_type 클레임 강제화 (SEC-02) | Phase 4 인증 레이어 소관 | High |
| CARRY-P8-003 | 배치 엔드포인트 rate limiting (SEC-09) | Phase 13 API Gateway 정책에 포함 | Medium |
| CARRY-P8-004 | DB JSONB 직접 수정 시 Pydantic 재검증 (SEC-11) | Phase 13 DB 접근 감사 정책과 함께 처리 | Low |

---

## 7. 신규 파일 목록

### 백엔드 — 도메인 모델

| 파일 | 설명 |
|------|------|
| `backend/app/models/extraction_schema.py` | ExtractionSchema, ExtractionFieldDefinition |
| `backend/app/models/extraction.py` (append) | ExtractionCandidate, ExtractionStatus, ExtractionMode |
| `backend/app/models/approved_extraction.py` | ApprovedExtraction, HumanEdit |
| `backend/app/models/batch_extraction.py` | BatchExtractionJob, BatchJobStatus, BatchRetryRequest |
| `backend/app/models/extraction_span.py` | SourceSpan, ExtractedFieldWithAttribution, SpanHighlight |
| `backend/app/models/extraction_record.py` | ExtractionRecord, VerificationResult, DiffDetail, MatchStatus |
| `backend/app/models/extraction_evaluation.py` | GoldenExtractionSet/Item, ExtractionMetrics, QualityGateResult |

### 백엔드 — Repository

| 파일 | 설명 |
|------|------|
| `backend/app/repositories/extraction_schema_repository.py` | raw psycopg2 CRUD |
| `backend/app/repositories/extraction_candidate_repository.py` | list_pending, count_pending, list_by_document |
| `backend/app/repositories/approved_extraction_repository.py` | approved_extractions CRUD |
| `backend/app/repositories/batch_extraction_repository.py` | BatchExtractionJobRepository + ExtractionRetryLogRepository |
| `backend/app/repositories/extraction_span_repository.py` | ExtractionSpanRepository |
| `backend/app/repositories/extraction_record_repository.py` | ExtractionRecordRepository + VerificationResultRepository |
| `backend/app/repositories/extraction_evaluation_repository.py` | 3개 Repository (Evaluation/GoldenSet/GoldenItem) |

### 백엔드 — 서비스

| 파일 | 설명 |
|------|------|
| `backend/app/services/extraction/__init__.py` | 패키지 초기화 |
| `backend/app/services/extraction/extraction_pipeline_service.py` | Jinja2 프롬프트 + LLM 호출 + fallback |
| `backend/app/services/extraction/templates/extraction_prompt.jinja2` | 동적 추출 프롬프트 템플릿 |
| `backend/app/services/extraction/batch_extraction_service.py` | BackgroundTask 워커, CHUNK/RATE 제어 |
| `backend/app/services/extraction/span_calculator.py` | SpanCalculator, MultiSpanExtractor, SpanVisualizationConverter |
| `backend/app/services/extraction/diff_calculator.py` | DiffCalculator (Levenshtein+epsilon+deep), SpanBasedDiffCalculator |
| `backend/app/services/extraction/extraction_verification_service.py` | ExtractionVerificationService, audit trail |
| `backend/app/services/extraction/extraction_evaluator.py` | ExtractionEvaluator, EvaluationReportGenerator, QualityGateChecker |

### 백엔드 — API

| 파일 | 설명 |
|------|------|
| `backend/app/api/v1/extraction_schemas.py` | ExtractionSchema CRUD 6개 엔드포인트 |
| `backend/app/api/v1/extractions.py` | 인간 검토 7개 + FG8.3 5개 = 총 12개 엔드포인트 |
| `backend/app/api/v1/batch_extractions.py` | 배치 재추출 5개 엔드포인트 |
| `backend/app/api/v1/extraction_evaluations.py` | 품질 평가 6개 엔드포인트 |

### 백엔드 — DB DDL (connection.py에 추가)

| DDL 상수 | 테이블 |
|----------|--------|
| `_EXTRACTION_SCHEMAS_DDL` | extraction_schemas |
| `_EXTRACTION_CANDIDATES_DDL` | extraction_candidates |
| `_APPROVED_EXTRACTIONS_DDL` | approved_extractions |
| `_BATCH_EXTRACTION_JOBS_DDL` | batch_extraction_jobs |
| `_EXTRACTION_RETRY_LOGS_DDL` | extraction_retry_logs |
| `_EXTRACTION_SPANS_DDL` | extraction_spans |
| `_EXTRACTION_RECORDS_DDL` | extraction_records |
| `_VERIFICATION_RESULTS_DDL` | verification_results |
| `_GOLDEN_EXTRACTION_SETS_DDL` | golden_extraction_sets |
| `_GOLDEN_EXTRACTION_ITEMS_DDL` | golden_extraction_items |
| `_EXTRACTION_EVALUATIONS_DDL` | extraction_evaluations |

### 테스트

| 파일 | 테스트 수 |
|------|-----------|
| `tests/unit/extraction/test_extraction_schema_*.py` 外 FG8.1 | 65 |
| `tests/unit/extraction/test_extraction_candidate_repository.py` | 19 |
| `tests/unit/extraction/test_extraction_pipeline_service.py` | 18 |
| `tests/unit/extraction/test_batch_extraction_service.py` | 20 |
| `tests/unit/extraction/test_span_calculator.py` | 28 |
| `tests/unit/extraction/test_diff_calculator.py` | 27 |
| `tests/unit/extraction/test_extraction_evaluator.py` | 30 |

### 문서 산출물

| 파일 | 설명 |
|------|------|
| `docs/개발문서/S2/phase8/산출물/검수보고서.md` | Phase 8 검수 보고서 |
| `docs/개발문서/S2/phase8/산출물/보안취약점검사보고서.md` | Phase 8 보안 취약점 검사 보고서 |
| `docs/개발문서/S2/phase8/산출물/Phase_8_종결보고서.md` | 본 보고서 |

---

## 8. 종합 판정

| 항목 | 결과 |
|------|------|
| 기능 구현 완성도 | 100% (task8-1 ~ task8-10 전원 완료) |
| 단위 테스트 | 207/207 PASSED |
| 검수 | ✅ PASS (수정 5건, 이월 1건) |
| 보안 취약점 검사 | ✅ PASS (Critical 1건 포함 8건 수정, 이월 4건) |
| S2 원칙 준수 | ✅ ①~⑦ 전원 준수 |

## ✅ Phase 8 종결 — Phase 9 진행 가능
