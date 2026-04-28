# FG 4-6 AI 품질 평가 보고서 — `save_draft` propose 정확성 + impact 일관성

**작성일**: 2026-04-28
**FG**: S3 Phase 4 / FG 4-6
**상태**: **선언적 종결** — measurement framework 정의. 실 measurement 는 운영자 후속 (Phase 0 FG 0-4 패턴).
**작성자**: Claude

---

## 1. 본 보고서의 위치

`save_draft` 는 MCP 표면 최초 쓰기 도구 — 환각 위험이 가장 큰 구간. 두 지표:
1. **propose 정확성** — agent 가 자연어 요청을 받아 의도된 draft 를 propose 했는가?
2. **impact 일관성** — preview (compute_draft_impact) 가 실제 적용 결과와 일치하는가?

본 FG 는 1라운드 **experimental** maturity — 본 measurement 통과 후 `beta` 로 승격 가능.

---

## 2. 측정 지표 + 임계값

| 지표 | 정의 | 임계값 |
|---|---|---|
| **Propose 정확성** | agent 의 자연어 요청에 대해 propose 된 draft 가 정답 (운영자 평가) 와 일치하는 비율 | ≥ 0.80 |
| **Impact 일관성** | preview 의 (chars_added, chars_removed, nodes_added, overwrites_existing_draft) 가 실제 적용 결과와 ±10% 이내 | ≥ 0.95 |
| **Idempotent Replay 정확성** | 같은 (agent, key) 재호출 시 같은 proposal 반환 | = 1.0 (절대) |
| **Human Approval 비율** | 자동 머지된 proposal 비율 | = 0.0 (절대 — 자동 머지 영구 금지) |

---

## 3. 골든셋 정의 (≥ 20 케이스, 운영자 후속 작성)

`backend/scripts/rag_smoke/golden_save_draft.py` (별도 작성):

| 카테고리 | 케이스 수 | 설명 |
|---|---|---|
| 신규 문서 propose | 5 | 자연어 요청 → 새 문서 생성 |
| 기존 draft 수정 | 5 | 자연어 요청 → 기존 draft 갱신 (overwrites_existing_draft=True 검증) |
| 부분 추가 | 5 | 기존 본문에 새 섹션 추가 |
| 부분 삭제 | 3 | 기존 본문 일부 삭제 |
| Idempotent replay | 2 | 같은 key 두 번 호출 → idempotent_replay=True |

각 케이스의 expected:
- expected_overwrites_existing_draft (bool)
- expected_chars_delta (int range)
- expected_nodes_delta (int range)
- expected_summary_keywords (list[str] — summary 텍스트에 포함되어야 할 단어)

---

## 4. 측정 절차 (운영자)

### 4.1 사전 조건

1. Staging DB 가 production 과 같은 데이터 분포
2. FG 4-6 코드 배포 + Alembic 적용 (`s3_p4_agent_prop_idempotency`)
3. ScopeProfile 에 save_draft 등록 (테스트 전용 ScopeProfile 신설 권고)
4. golden_save_draft.py 작성 + 골든셋 시드

### 4.2 실행

```bash
cd backend
source .venv/bin/activate
python scripts/rag_smoke/golden_save_draft.py --run \
  --out docs/개발문서/S3/phase4/산출물/FG4-6_AI품질평가_<날짜>.json
```

### 4.3 임계값 미달 시

- **Propose 정확성 < 0.80**: agent 가 사용한 LLM 의 출력 품질 문제. content_snapshot 형식 가이드 (preferred_use 문구) 보강 검토.
- **Impact 일관성 < 0.95**: compute_draft_impact 의 단순화 한계 — 정밀 NodeDiff 도입 우선 (별 라운드).
- **Idempotent Replay ≠ 1.0**: 즉시 release 차단. UNIQUE INDEX / `_lookup_idempotent_proposal` 의 race condition 가능성 점검.
- **Human Approval Bypass > 0**: **즉시 보안 사고**. 본 FG 의 핵심 가정 위반 — 도구 즉시 status="disabled" 비노출 + Postmortem.

---

## 5. 사전 분석 (단위 / mock 기반 — 실측 외)

본 FG 의 회귀 테스트가 알고리즘 정합성을 검증 — 정밀도는 검증 안 함:

| 회귀 항목 | 결과 |
|---|---|
| compute_draft_impact 신규/덮어쓰기/길이 차이 정확성 | ✅ ([TestComputeDraftImpact](../../../../backend/tests/integration/test_save_draft_fg46.py), 3 case) |
| Idempotent replay (lookup → 기존 proposal 재사용) | ✅ ([TestRepositoryIdempotency](../../../../backend/tests/integration/test_save_draft_fg46.py), 2 case) |
| `requires_human_approval=True` 강제 | ✅ ([TestSchemas / TestToolSaveDraft](../../../../backend/tests/integration/test_save_draft_fg46.py)) |
| audit chain 4 events 등재 | ✅ (event_types.md) |
| L3/L4 영구 제외 회귀 | ✅ ([TestL3L4PermanentExclusion](../../../../backend/tests/integration/test_save_draft_fg46.py), 4 case) |

---

## 6. 한계

운영자 실 measurement 가 완료되기 전까지:

| # | 한계 | 의미 |
|---|---|---|
| 1 | Propose 정확성 미산출 | agent → save_draft 호출 시점의 실 정밀도 측정 안 됨 |
| 2 | Impact 일관성 미산출 | preview vs actual diff 의 차이 운영 데이터 부재 |
| 3 | golden_save_draft.py 미작성 | 운영자 후속 작업 |
| 4 | reviewer approval UI 영향 미검증 | reviewer 가 actual diff 를 검토하는 UX 가 별 작업 — UX 이슈 시 본 FG 의 가치가 줄어들 수 있음 |

---

## 7. 변경 이력

| 일자 | 변경 | 작성자 |
|---|---|---|
| 2026-04-28 | 초판 — measurement 절차 + 사전 분석 + 4 지표 임계값. 골든셋 + 실측은 운영자 후속. | Claude |
