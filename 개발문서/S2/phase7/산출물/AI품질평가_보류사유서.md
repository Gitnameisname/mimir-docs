# AI 품질평가 보류 사유서 — Phase 7 (AI 품질 평가 인프라)

**작성일**: 2026-04-18
**작성자**: Claude Opus 4.7 (S2 전체 재검수 후속)
**대상 Phase**: S2 Phase 7 — AI 품질 평가 인프라
**상위 근거**: `docs/개발문서/S2_종결보정서_20260418.md` §2 (F-02 AI 품질 기준 달성 선언 철회)
**상태**: **보류 (Deferred to S3 P0)**

---

## 1. 본 문서의 위치와 특수성

Phase 7 은 "AI 품질을 **측정하는 인프라** 를 만드는 Phase" 이다. 따라서 "Phase 7 자체에 대한 AI 품질평가" 는 메타적 관계에 놓인다 — 측정 도구가 맞게 동작하는지의 평가이다. 본 문서는 이 특수성을 명시하고, **왜 지금 실측 run 을 수행하지 않았는지**, **언제·어떻게 측정할 것인지** 를 기록한다.

Phase 7 의 **인프라 구축** 자체는 종결되었다(아래 §3 참조). 그러나 "인프라가 맞게 동작함" 을 End-to-End 실측으로 확인한 run 기록이 **0건** 이다. 이것이 본 문서가 요구되는 본질적 이유이다.

---

## 2. Phase 7 의 AI 관련 기능 범위

| 기능 | 코드 위치 | 평가가 필요한 품질 축 |
|------|-----------|----------------------|
| 6-메트릭 임계값 시스템 | `backend/config/evaluation_thresholds.yaml` | 임계값 로드 정확도, 환경별 override 정확도 |
| CI Gate 서비스 | `backend/app/services/evaluation/ci_gate.py` | exit code 정확도, PR 스킵 룰 정확도 |
| 골든셋 평가 실행 클라이언트 | `backend/scripts/evaluate_golden_set.py` | End-to-End run 정상성, 폐쇄망 모드 degrade 정상성 |
| 평가 보고서 생성기 | `backend/scripts/generate_evaluation_report.py` + Jinja2 템플릿 | 수치-메트릭 매칭 정확도, 이전 run 비교 정확도 |
| 골든셋 CRUD / import·export | `backend/app/api/v1/golden_sets_router.py` | API 계약 정합성 (기 검증 — 단위 테스트 범위) |

---

## 3. 현재 시점에서 **검증된 것 vs. 보류 사항**

### 3.1 인프라 레벨 ✅ (단위 테스트 범위 — 본 보류 사유서가 부정하지 않음)

Phase 7 종결보고서에 기록된 바와 같이 단위 테스트 74/74 PASS 로 다음이 검증되어 있다.

- 임계값 YAML 파싱 및 환경별 override (26/26 test)
- 보고서 생성기의 Jinja2 렌더링 정합성 (19/19 test)
- 골든셋 CRUD API 계약 (FastAPI 테스트 클라이언트)
- CI Gate 의 PASS/FAIL exit code
- 폐쇄망 모드 `fallback_base_scores` 진입 경로

이들은 "인프라가 명세대로 동작한다" 는 **구조적 정합성** 증거이다.

### 3.2 E2E 실측 run ❌ (보류)

그러나 다음은 아직 수행되지 않았다 — 본 보류 사유서의 핵심 대상이다.

1. **실제 골든셋을 넣은 end-to-end run**: 골든셋 데이터 자체가 0건이므로, 평가 러너가 "실제 사용 환경에서 의미 있는 숫자를 생산하는지" 가 측정된 적이 없다.
2. **판정 LLM 왕복 검증**: Faithfulness·Answer Relevance 산출 경로(LLM-as-a-Judge) 가 실제 closed-network LLM 과 왕복하여 정상 응답·정상 파싱을 반환하는지의 실측 로그가 0건이다.
3. **CI Gate 의 실제 PR 트리거**: `.github/workflows/golden-set-evaluation.yml` 은 작성되어 있으나 실제 PR 에서 트리거되어 exit 0/1 을 반환한 기록이 없다.
4. **보고서 생성기의 이전 run 비교**: 비교 대상이 될 "이전 run" 자체가 없어 📈📉➡️ 변화 표시 로직은 실 데이터에서 검증된 적이 없다.

### 3.3 보류의 본질

Phase 7 은 **도구** 를 만든 Phase 이다. 도구가 **있다** 는 것과 도구가 **쓸 만하다** 는 것은 다르다. 본 사유서는 이 구분을 깨뜨리지 않기 위해, 현 시점에 "Phase 7 의 평가 인프라가 실측 품질 기준을 충족한다" 라고 선언하지 않는다.

---

## 4. 실측 계획 (S3 P0 이월)

| 단계 | 내용 | 산출물 | 담당 | 기한 |
|------|------|--------|------|------|
| 1 | 골든셋 최소 50건 구축 (Phase 2·Phase 5·Phase 8 공용 셋 포함) | `golden_sets/seed_v1.json` | 지식팀 | S3 착수 + 2주 |
| 2 | closed-network 판정 LLM 상시 가동 (로컬 vLLM or 동등 모델) | 배포 문서 + 헬스체크 | 인프라팀 | S3 착수 + 1주 |
| 3 | `scripts/evaluate_golden_set.py --closed-network --set seed_v1` E2E run 1회 | `evaluation_reports/seed_v1_<date>.json` + 생성된 markdown 보고서 | 개발팀 | 1+2 단계 완료 직후 |
| 4 | `generate_evaluation_report.py` 의 이전 run 비교 로직이 두 번째 run 에서 정상 동작하는지 확인 | 2회차 run + diff 렌더링 확인 | 개발팀 | 3단계 이후 |
| 5 | CI Gate 를 실제 PR 에 트리거 → exit 0 / exit 1 양쪽 케이스 재현 | GitHub Actions run URL 2건 | 개발팀 | 4단계 이후 |
| 6 | 본 문서를 **AI품질평가보고서_Phase7.md** 로 승격 (인프라 자체의 동작 수치 기입) | Phase 7 산출물 폴더 | 개발팀 | 5단계 직후 |

---

## 5. 수치 선언 금지 사항

본 보류가 해제되기 전까지 다음 문구는 **어떤 문서에도 기재되어서는 안 된다**.

- "Phase 7 의 평가 인프라는 Faithfulness ≥0.80 을 달성한다" — Phase 7 은 측정 도구이므로 자체가 Faithfulness 를 가질 수 없다. 이런 표현이 있다면 삭제 대상.
- "Phase 7 평가 러너는 실측에서 정상 동작이 확인되었다" — 실측 run 0건 상태.
- "S2 는 AI 품질 기준을 충족한다" — Phase 7 단일로는 이 선언의 근거가 될 수 없다.

---

## 6. 서명

| 역할 | 이름 | 날짜 |
|------|------|------|
| 재검수 주관 | Claude Opus 4.7 | 2026-04-18 |
| 기술 리드 | (서명 대기 — S3 P0 착수 시) | |
