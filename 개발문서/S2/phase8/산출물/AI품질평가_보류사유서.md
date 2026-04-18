# AI 품질평가 보류 사유서 — Phase 8 (Structured Extraction)

**작성일**: 2026-04-18
**작성자**: Claude Opus 4.7 (S2 전체 재검수 후속)
**대상 Phase**: S2 Phase 8 — Structured Extraction
**상위 근거**: `docs/개발문서/S2_종결보정서_20260418.md` §2 (F-02 AI 품질 기준 달성 선언 철회)
**상태**: **보류 (Deferred to S3 P0)**

---

## 1. 본 문서의 위치

본 문서는 「AI 품질평가 **보고서**」가 아니다. Phase 8 은 문서에서 구조화 필드를 LLM 보조로 추출하는 기능을 구축했다. 추출 품질은 본질적으로 "field-level precision / recall / faithfulness" 로 측정되나, 측정을 위한 자원이 미확보이므로 본 문서는 **왜 Phase 8 AI 품질을 지금 실측하지 않았는지**, **언제·어떻게 측정할 것인지** 를 공식 기록한다.

---

## 2. Phase 8 의 AI 관련 기능 범위

| 기능 | 코드 위치 | 평가가 필요한 품질 축 |
|------|-----------|----------------------|
| DocumentType metadata schema → 추출 타겟 매핑 | `backend/app/services/extraction/target_schema.py` | 스키마 매핑 정합성(기 검증), 필드별 instruction 적용 여부 |
| LLM 보조 추출 파이프라인 | `backend/app/services/extraction/extractor.py` | **Field Precision**, **Field Recall**, **Field-level Faithfulness** |
| 원문 span 역참조 (provenance) | `backend/app/services/extraction/provenance.py` | **Span 정확도**(추출값이 지시한 span 이 실제로 해당 값을 포함하는가) |
| 인간 승인 플로우 | `backend/app/api/v1/extraction_router.py` | **Human-Approval Rate**, **Edit Distance**(승인 전/후 값 차이) |
| 재실행 가능성 (idempotency) | `backend/app/services/extraction/rerun_service.py` | 동일 입력에서 재현성, 버전 차이 시 의도된 diff 정확도 |

---

## 3. 보류 사유

### 3.1 필수 전제 자원 미확보

1. **추출 골든셋 미구축**: Phase 8 평가의 핵심은 (문서, DocumentType, 기대 필드값, 기대 span) 의 4-튜플 골든셋이다. S2 종결 시점에 이 데이터는 **0건** 이다.
2. **추출 전용 LLM run 로그 부재**: 추출 파이프라인이 closed-network 환경에서 실제 LLM 과 왕복하여 생산한 run 로그 기록이 0건이다. Phase 7 의 `evaluation_reports/` 아래에도 extraction 전용 run 은 부재하다.
3. **DocumentType 별 instruction 프롬프트의 A/B 비교 데이터 없음**: 추출 품질은 instruction 문구에 매우 민감한데, 현재 저장소 내 instruction 은 초기 버전 1종만 존재하며 A/B 측정 이력이 없다.
4. **Phase 7 평가 러너가 extraction 전용 지표를 아직 지원하지 않음**: `evaluation_thresholds.yaml` 은 RAG 지표(Faithfulness / Citation-present 등) 위주로 구성되어 있어, **Field Precision / Field Recall / Span 정확도** 를 지원하는 확장이 필요하다. 본 확장 자체가 S3 P0 이월 과제이다.

### 3.2 현재 시점에서 **검증된 것** (단위/통합 테스트 범위)

| 검증 항목 | 수단 | 상태 |
|-----------|------|------|
| DocumentType metadata schema 로드 및 required 필드 검증 | 단위 테스트 | ✅ |
| 추출 결과의 span 좌표가 스키마 상의 원문 범위 내에 위치 | 단위 테스트 (경계 조건 포함) | ✅ |
| 인간 승인 플로우의 상태 전이 (`pending → approved / rejected`) | 통합 테스트 | ✅ |
| 재실행 시 이전 추출 결과와 diff 를 보존 | 단위 테스트 | ✅ |
| 추출 이벤트의 감사 기록 `actor_type`(F-07 시정 완료) | 타입 레벨 | ✅ |

이들은 **기능적 정합성** 이지 **LLM 추출 품질 지표** 가 아니다.

### 3.3 Phase 8 특유 지표의 임계값 부재

Phase 8 고유 지표에 대해 `evaluation_thresholds.yaml` 에 임계값이 설정되어 있지 않다. 기대 임계값 후보(S3 초기 논의용):

- **Field Precision** ≥ 0.85 (필드별 평균)
- **Field Recall** ≥ 0.80 (필드별 평균)
- **Span 정확도** ≥ 0.95 (추출값이 가리키는 span 에 실제로 해당 값이 포함되는 비율)
- **Human-Approval Rate** ≥ 0.70 (추출 직후, 수정 없이 승인된 비율)

위 임계값의 공식 편입 자체가 S3 P0 이월 과제이다.

---

## 4. 실측 계획 (S3 P0 이월)

| 단계 | 내용 | 산출물 | 담당 | 기한 |
|------|------|--------|------|------|
| 1 | Phase 8 전용 추출 골든셋 30건 구축 (주요 DocumentType 최소 3종 × 10건) | `golden_sets/phase8_extraction.json` | 지식팀 + 개발팀 | S3 착수 + 3주 |
| 2 | `evaluation_thresholds.yaml` 에 Field Precision / Recall / Span / HAR 네 축 추가 | YAML 반영 + 문서화 | 개발팀 | 1단계와 병렬 |
| 3 | Phase 7 평가 러너 확장 — extraction adapter 추가(`scripts/evaluate_extraction.py`) | 스크립트 | 개발팀 | 2단계 완료 직후 |
| 4 | closed-network LLM 으로 E2E run 1회 → `evaluation_reports/phase8_extraction_<date>.json` | 메타데이터 + 생성된 markdown | 개발팀 | 3단계 직후 |
| 5 | 본 문서를 **AI품질평가보고서_Phase8.md** 로 승격 (수치 채움) | Phase 8 산출물 폴더 | 개발팀 | 4단계 직후 |
| 6 | 미달 시 instruction 프롬프트 A/B 또는 LLM 모델 교체 후 재측정 | 동일 | 개발팀 | 5단계 결과에 따라 |

---

## 5. 수치 선언 금지 사항

본 보류가 해제되기 전까지 다음 문구는 **어떤 문서에도 기재되어서는 안 된다**.

- "Phase 8 추출 필드 Precision 은 X.XX 이다"
- "Phase 8 Span 정확도는 X.XX 를 달성했다"
- "Phase 8 은 AI 품질 기준을 충족한다"

S2 종결보고서의 동등한 선언은 S2_종결보정서_20260418.md §2.3 에서 **공식 철회** 되었다.

---

## 6. 서명

| 역할 | 이름 | 날짜 |
|------|------|------|
| 재검수 주관 | Claude Opus 4.7 | 2026-04-18 |
| 기술 리드 | (서명 대기 — S3 P0 착수 시) | |
