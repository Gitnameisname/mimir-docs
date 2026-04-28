# FG 4-3 AI 품질 평가 보고서 — verify_citation 5중 검증 영향

**작성일**: 2026-04-28
**FG**: S3 Phase 4 / FG 4-3
**상태**: **선언적 종결** — measurement 절차 정의 완료. 실 measurement 는 **운영자 후속**.
**작성자**: Claude (Design Actor)

---

## 1. 본 보고서의 위치

본 FG (verify_citation 5중 검증) 가 도입되면 **기존 인용의 검증 통과율** 이 변할 수 있다. 변동이 일정 임계값 (≤ 0.02) 을 넘으면 release 차단 — 기존 클라이언트의 citation 신뢰성을 깨지 않기 위함.

본 보고서는 다음 두 부분으로 구성:
1. **회귀 측정 framework** — 본 세션 완료 (보고서 §3)
2. **실 measurement** — 운영자 후속 (Phase 0 FG 0-4 패턴)

---

## 2. 측정 지표

| 지표 | 정의 | 임계 변동 |
|---|---|---|
| **Citation-present rate** | RAG 응답 중 인용이 1건 이상 포함된 비율 | ≤ 0.02 |
| **Faithfulness** | 응답 본문이 인용된 노드 본문에 의해 뒷받침되는 비율 | ≤ 0.02 |
| **Verify pass rate (신규)** | 5중 검증을 통과하는 인용 비율 (신규 지표 — 본 FG 도입 후) | baseline 측정 |

본 FG 의 변경이 **읽기 전용** 이고 verify_citation 의 입출력 호환성을 보존하므로, Citation-present / Faithfulness 변동은 0 에 가까워야 한다 (이론 상). 다만 새 검사 (pinned / quoted_text / span_valid) 가 실제 데이터에서 false 를 트리거하면 변동이 발생할 수 있어 measurement 의무.

---

## 3. 측정 절차

### 3.1 사전 조건

1. Staging DB 가 production 과 같은 데이터 분포 (golden_set + 실 사용 인용 샘플)
2. FG 4-3 코드 배포 (본 PR 머지 후)
3. (선택) `MIMIR_OFFLINE=1` 폐쇄망 모드 별도 측정

### 3.2 실행

```bash
cd backend
source .venv/bin/activate

# (1) baseline 측정 — FG 4-3 적용 전 코드에서 한 번
git checkout pre-fg43
python scripts/evaluate_golden_set.py --out baseline_pre_fg43.json

# (2) 적용 후 측정
git checkout fg43-merged
python scripts/evaluate_golden_set.py --out post_fg43.json

# (3) 비교
python scripts/check_evaluation_thresholds.py \
  --baseline baseline_pre_fg43.json \
  --current post_fg43.json \
  --citation-present-delta 0.02 \
  --faithfulness-delta 0.02
```

### 3.3 임계값 미달

`Citation-present rate` 또는 `Faithfulness` 변동이 0.02 를 초과하면:
- 변동의 원인 분석 (어느 검사가 트리거되는지 — pinned / quoted_text / span_valid)
- 데이터 quality 이슈 (예: 기존 인용이 draft 버전으로 저장되어 pinned False) 인지 확인
- 필요 시 backfill 또는 호환 모드 (예: `pinned` 검사를 archived 도 통과시키는 정책 — 이미 적용) 검토

### 3.4 신규 지표 — Verify pass rate

```bash
python scripts/evaluate_golden_set.py --measure-verify-pass-rate --out verify_pass.json
```

목표: verify pass rate ≥ 0.95 (95% 이상의 기존 인용이 5중 검증 통과). 미달 시 데이터 cleanup 검토.

---

## 4. 사전 분석

### 4.1 회귀 위험 분류

| 검사 | 회귀 위험 | 대응 |
|---|---|---|
| 검사 1 (exists) | 매우 낮음 — 기존 동작과 동일 (version + chunk 조회) | 변경 없음 |
| 검사 2 (pinned) | **중간** — 기존에 draft 버전으로 저장된 인용이 있을 경우 트리거 | 데이터 분포 측정 후 결정 |
| 검사 3 (hash_matches) | 매우 낮음 — 기존 동작 (citation_basis=node_content default) |
| 검사 4 (quoted_text) | 0 — 기존 클라이언트는 quoted_text 미입력. None 반환으로 통과 처리 | 변경 없음 |
| 검사 5 (span_valid) | 0 — 기존 클라이언트는 span_length 미입력. None 반환으로 통과 처리 | 변경 없음 |

### 4.2 백워드 호환 보장

| 항목 | 상태 |
|---|---|
| `VerifyCitationData` 의 `content_snapshot` / `hash_matches` / `version_valid` 필드 유지 | ✅ |
| `citation_basis` default = `node_content` (기존 호출자 호환) | ✅ |
| `quoted_text` / `span_length` 둘 다 Optional (입력 없으면 검사 skip) | ✅ |
| `latest` 거부 — **신규 거부** (R3 강제) | ⚠️ |

`latest` 거부가 유일한 비호환 변경 — 기존 클라이언트가 `latest` 로 verify 호출하던 경우 본 FG 적용 후 400 에러. **operational impact**: 기존 클라이언트가 인용 시점에 vN resolve 후 저장하는 패턴을 따라야 함.

---

## 5. 측정 전 한계

운영자 실 measurement 가 완료되기 전까지 본 보고서는 다음 한계를 명시한다:

| # | 한계 | 의미 |
|---|---|---|
| 1 | Citation-present / Faithfulness 변동 미산출 | release gate 미통과 — 본 FG 의 적용 영향이 실 데이터에서 어떻게 분포하는지 불명 |
| 2 | Verify pass rate baseline 부재 | 신규 지표라 측정 후 모니터링 |
| 3 | 폐쇄망 (MIMIR_OFFLINE=1) 별도 측정 미수행 | rendered_text 모드는 render_service 위임 — 폐쇄망에서도 동작 (외부 LLM 호출 0). 별도 측정으로 확인 |

---

## 6. 회귀 검증 (단위 / mock 기반 — 실측 외)

본 FG 의 단위 / 통합 회귀는 알고리즘의 **정합성** + **R3 강제** 만 검증. 실 데이터 정밀도는 검증 안 함.

| 회귀 항목 | 결과 |
|---|---|
| 검사 1~5 단위 동작 | ✅ ([test_verify_citation_v2.py](../../../../backend/tests/integration/test_verify_citation_v2.py)) 27 case |
| `latest` 거부 (R3) | ✅ (TestCheck2Pinned::test_latest_input_immediately_rejected) |
| citation_basis 분기 (node vs rendered) | ✅ (TestCheck3HashMatches 5 case) |
| 백워드 호환 (기존 응답 필드 보존) | ✅ |
| envelope (FG 4-1) tool_metadata | ✅ |

---

## 7. 변경 이력

| 일자 | 변경 | 작성자 |
|---|---|---|
| 2026-04-28 | 초판 — measurement 절차 + 사전 분석 + 회귀 위험 분류. 실 measurement 는 운영자 후속. | Claude |
