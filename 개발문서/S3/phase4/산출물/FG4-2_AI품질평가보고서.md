# FG 4-2 AI 품질 평가 보고서 — `resolve_document_reference`

**작성일**: 2026-04-28
**FG**: S3 Phase 4 / FG 4-2 §2.1.5
**상태**: **선언적 종결** — 골든셋 + 평가 framework 작성 완료. 실 measurement 는 **운영자 후속**. (Phase 0 FG 0-4 동일 패턴)
**작성자**: Claude (Design Actor)

---

## 1. 본 보고서의 위치

`resolve_document_reference` 도구는 자연어 참조를 `document_id` + `version_ref` 로 정규화한다. 환각 (잘못된 문서를 high confidence 로 반환) 위험이 본 FG 의 가장 큰 보안 가치이며, **AI 품질 평가가 release gate**다 (task4-2 §7).

본 보고서는 다음 두 부분으로 구성된다:
1. 골든셋 정의 + 평가 framework — **이번 세션 완료**
2. 실 measurement (Precision@1 / Recall@5 / Disambiguation correctness) — **운영자 후속**

---

## 2. 골든셋 (50 케이스)

### 2.1 카테고리별 분포

| 카테고리 | 케이스 수 | 의도 |
|---|---|---|
| `exact_title` | 20 | 제목 정확 일치 — 가장 명확한 케이스 |
| `alias` | 10 | metadata.aliases / previous_titles 매칭 |
| `semantic` | 15 | 의미 매칭 — 자연어 표현이 제목과 다름 |
| `disambiguation` | 5 | 모호 — `needs_disambiguation=true` 트리거 기대 |
| **합계** | **50** | |

### 2.2 정본 위치

[backend/scripts/rag_smoke/golden_resolve.py](../../../../backend/scripts/rag_smoke/golden_resolve.py) `GOLDEN_CASES` 정본 리스트.

각 케이스 필드:
- `case_id` (E01~E20 / A01~A10 / S01~S15 / D01~D05)
- `category`
- `reference` (자연어)
- `expected_document_id` (placeholder `<seed:xxx>` — 운영자 staging seed 시 실제 UUID 로 대체)
- `expected_disambiguation` (boolean)
- `notes`

### 2.3 시드 placeholder

골든셋의 `expected_document_id` 는 `<seed:sales_manual_2024>` 등 placeholder 형식. 운영자가 staging DB 에 골든셋용 문서를 시드할 때 매핑을 채워야 한다.

운영자 시드 스크립트는 본 FG 범위 외 — 별도 작업 (`scripts/seed_golden_resolve.py` 권고).

---

## 3. 지표 정의

| 지표 | 계산 | 임계값 |
|---|---|---|
| **Precision@1** | `expected_document_id 가 있는 케이스 중 best_match.document_id 가 일치하는 비율` | ≥ 0.85 |
| **Recall@5** | `expected_document_id 가 candidates[0..5] 안에 포함된 비율` | ≥ 0.95 |
| **Disambiguation correctness** | `actual_disambiguation == expected_disambiguation 인 비율` (전체 케이스 대상) | ≥ 0.80 |

`disambiguation` 카테고리 5 케이스는 `expected_document_id=None` + `expected_disambiguation=True` 로, Precision@1 / Recall@5 모집단 (`expected_document_id is not None`) 에서 자연 제외된다.

---

## 4. 측정 절차 (운영자)

### 4.1 사전 조건

1. Staging DB 가 깨끗한 상태 (또는 별도 평가 schema)
2. 50 케이스의 `expected_document_id` placeholder → 실 UUID 매핑 완료
3. (선택) alias 메타 시드: `documents.metadata.aliases` / `previous_titles` 채움 — `alias` 카테고리 평가에 필요
4. 폐쇄망 모드 평가 시 `export MIMIR_OFFLINE=1`

### 4.2 실행

```bash
cd backend
source .venv/bin/activate
python scripts/rag_smoke/golden_resolve.py --run \
  --out docs/개발문서/S3/phase4/산출물/FG4-2_AI품질평가_<날짜>.json
```

### 4.3 출력

```json
{
  "metrics": {
    "total": 50,
    "precision_at_1": 0.xx,
    "recall_at_5": 0.xx,
    "disambiguation_correctness": 0.xx,
    "by_category": {
      "exact_title": {"n": 20, "precision_at_1": 0.xx, ...},
      "alias": {"n": 10, ...},
      "semantic": {"n": 15, ...},
      "disambiguation": {"n": 5, ...}
    }
  },
  "results": [
    {"case_id": "E01", "category": "exact_title", ..., "pass_at_1": true, ...},
    ...
  ]
}
```

### 4.4 재실행 / 임계값 미달

`Precision@1 < 0.85` 또는 `Recall@5 < 0.95` 또는 `Disambiguation correctness < 0.80` 인 경우:
- `confidence_threshold` 조정 검토 (현재 default 0.85)
- 후보 알고리즘 조정 (단계별 confidence 가중치)
- alias 메타 시드 보강
- 의미 단계의 fallback 정책 검토

조정 후 재실행 → 동일 procedure.

---

## 5. 사전 분석 — 골든셋의 자체 검증

골든셋 정본을 실 DB 없이 자체 점검한다 (운영자 measurement 전 검증):

| 점검 | 결과 |
|---|---|
| 50 케이스 카테고리 분포 task4-2 §2.1.5 일치 (20 / 10 / 15 / 5) | ✅ |
| disambiguation 케이스의 `expected_disambiguation=True` 명시 | ✅ |
| disambiguation 외 케이스의 `expected_document_id` placeholder 존재 | ✅ |
| reference 텍스트 길이 1~500 (Pydantic 제약) 만족 | ✅ |
| reference 가 한국어 / 영어 혼합 (실 환경 반영) | ✅ — `영업 매뉴얼 2024` / `Sales Playbook` / `사고 발생 시 절차` 등 |

---

## 6. measurement 전 한계

운영자 실 measurement 가 완료되기 전까지 본 보고서는 다음 한계를 명시한다:

| # | 한계 | 의미 |
|---|---|---|
| 1 | Precision@1 / Recall@5 / Disambiguation 수치 미산출 | release gate 미통과 — `resolve_document_reference` 의 `experimental` maturity 가 본 measurement 통과 후 `beta` 로 승격 가능 |
| 2 | 실 데이터 / 실 인프라 (벡터 검색) 미접근 | 단계 4 (semantic) 의 정밀도는 벡터 인프라 품질에 좌우 — staging 환경에서 검증 필요 |
| 3 | alias 메타 시드 누락 시 alias 카테고리 0% | 운영자가 시드 절차 명시 — 실측 시 alias 시드 여부 명기 |

---

## 7. 통합 / 회귀 검증 (실측 외)

본 FG 의 회귀 테스트는 알고리즘의 **결정성** + **ACL** + **폐쇄망 fallback** 만 검증 — 정밀도는 검증 안 함.

| 회귀 테스트 (mock 기반) | 결과 |
|---|---|
| `_stage_exact_title` confidence 0.99 | ✅ ([TestResolverStages::test_exact_title_high_confidence](../../../../backend/tests/integration/test_mcp_tools_fg42.py)) |
| `_stage_recent_context` 부분 일치 confidence 0.85 | ✅ |
| ACL post-filter 차단 | ✅ |
| 폐쇄망 모드 (`MIMIR_OFFLINE=1`) FTS fallback | ✅ |
| 임계값 미달 시 `needs_disambiguation=true` | ✅ |
| `version_ref` 가 단독 `"latest"` 가 아닌 `"latest_published"` 또는 `vN` | ✅ (R3) |

---

## 8. 변경 이력

| 일자 | 변경 | 작성자 |
|---|---|---|
| 2026-04-28 | 초판 — 50 케이스 골든셋 + 평가 framework + 측정 절차. 실 measurement 는 운영자 후속. | Claude |
