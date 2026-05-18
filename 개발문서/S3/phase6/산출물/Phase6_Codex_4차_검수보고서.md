# S3 Phase 6 Codex 4차 검수보고서 — 3차 시정분 확인

**작성일**: 2026-05-18
**검수자**: Codex
**대상**: `Phase6_Codex3차_시정보고서.md` 기반 Claude 시정분
**Handoff Level**: extended (retention DB 폐기 + admin 권한 경계)
**검수 판정**: 통과 — P1 승인 대기

---

## 1. 검수 범위

- `backend/app/services/retention_job.py`
- `backend/tests/unit/test_retention_job_fg62.py`
- `backend/tests/unit/test_admin_org_isolation_routes_fg64.py`
- `docs/함수도서관/backend.md`
- `docs/개발문서/S3/phase6/산출물/Phase6_Codex3차_시정보고서.md`
- `docs/개발문서/S3/phase6/산출물/Phase6_1라운드_종결보고서.md`
- `docs/개발문서/S3/phase6/산출물/FG6-{1,2,3,4}_검수보고서.md`
- `docs/개발문서/S3/phase6/산출물/FG6-4_Admin엔드포인트_점검표.md`

## 2. 실행 검증

### Phase 6 신규/수정 회귀

```bash
cd backend
./.venv/bin/python -m pytest \
  tests/unit/test_rate_limit_fg61.py \
  tests/unit/test_retention_job_fg62.py \
  tests/unit/test_content_sanitizer_fg63.py \
  tests/unit/test_admin_org_guard_fg64.py \
  tests/unit/test_admin_org_isolation_routes_fg64.py \
  --no-cov
```

결과:

```text
46 passed in 1.20s
```

### Backend unit baseline

```bash
cd backend
./.venv/bin/python -m pytest tests/unit/ \
  --ignore=tests/unit/test_annotations_service_fg33.py \
  --no-cov -q
```

결과:

```text
2748 passed, 13 skipped, 1 warning in 189.88s
```

### Security baseline

```bash
cd backend
./.venv/bin/python -m pytest tests/security/ --no-cov -q
```

결과:

```text
250 passed, 1 skipped in 3.28s
```

## 3. 3차 Finding 시정 확인

| 3차 Finding / 권고 | 4차 판정 | 근거 |
|---|---|---|
| P2-1 admin endpoint 검증 범위와 산출물 표현 불일치 | 해소 | 점검표가 3축 결합 검증으로 표현을 낮췄고, route-level 테스트가 5건으로 보강됨 |
| P3-1 이전 baseline 수치 잔존 | 해소 | 최신 종결보고서와 FG 검수보고서가 46건 / 2748 passed 기준으로 정리됨 |
| audit retention verify gate 대칭화 권고 | 해소 | audit retention에도 archive 존재성 verify + rollback + RuntimeError 추가 |

과거 `Phase6_Codex_2차_검수보고서.md`, `Phase6_Codex_3차_검수보고서.md`, `Phase6_Codex2차_시정보고서.md` 안의 37/39/42/2739/2744 수치는 당시 finding 또는 변화 이력으로 남아 있는 기록이다. 최신 산출물 정합성 판단 대상은 종결보고서, FG 검수보고서, 점검표, 함수도서관이며 이 문서들은 현재 46/2748 기준으로 맞춰져 있다.

## 4. 코드 검수 결과

### Retention

`RetentionJob` 의 archive-first 정책은 annotation과 audit 양쪽 모두 같은 구조로 정리됐다.

- `inserted ∪ already_archived` 로 DELETE 대상을 제한한다.
- annotation은 `WITH RECURSIVE descendants` 로 nested reply를 포함한다.
- audit과 annotation 모두 DELETE 후 archive 존재성 verify를 수행한다.
- verify 불일치 시 rollback 후 `RuntimeError` 로 배치 결과가 `partial` 이 된다.

4차 검수에서 추가 blocker는 발견하지 못했다.

### Admin org isolation

Admin org 격리는 다음 3축으로 검증 범위가 정리됐다.

- helper 단위 분기 8건.
- 대표 route-level 5건: agent / scope-profile / user-org-role / organization cross-org reject, SUPER_ADMIN cross-org audit emit.
- 14 endpoint guard wired 여부 코드 점검표.

14 endpoint × 4 시나리오 전수 route-level 회귀는 별 라운드로 명시되어 있으며, 현재 Phase 6 종결 blocker로 보지 않는다. 단일 helper 경로와 endpoint group 대표 회귀가 확보되어 실질적 회귀 방어는 충분하다고 판단한다.

## 5. 문서 검수 결과

- `Phase6_1라운드_종결보고서.md`: 신규 회귀 46건, unit baseline 2748 passed, security 250 passed로 정리됨.
- `FG6-4_Admin엔드포인트_점검표.md`: 이전의 "각 endpoint 4시나리오 회귀" 표현이 제거되고 3축 결합 검증으로 정확화됨.
- `FG6-1~4_검수보고서.md`: 최신 회귀 수와 baseline 수치가 반영됨.
- `docs/함수도서관/backend.md`: Phase 6 신규 함수와 회귀 카운트 46건이 반영됨.

## 6. 잔존 리스크 / 이월

| 항목 | 판정 | 비고 |
|---|---|---|
| 14 endpoint × 4 시나리오 전수 route-level 회귀 | 이월 | 현재는 대표 route-level + helper + 코드 점검으로 승인 가능. 운영자가 요구하면 별 라운드 |
| SUPER_ADMIN cross-org audit emit 실패 시 차단 여부 | 이월 | 현재 warning 후 통과. 보안 강화 정책은 별도 승인 사안 |
| archive 자체 sub-retention / PURGE | 이월 | Phase 6 범위 밖 |
| Phase 5 FG 5-5 stale assertion (`test_annotations_service_fg33.py`) | 이월 | 본 Phase 영향 아님. baseline에서는 명시적으로 ignore |

## 7. 결론

Codex 2차와 3차에서 지적한 P1/P2/P3 항목은 현재 코드와 최신 산출물 기준으로 해소됐다.

- **코드 판정**: 통과
- **테스트 판정**: 통과
- **문서 판정**: 통과
- **종결 조건**: 충족

**최종 판정: S3 Phase 6 1라운드 종결 가능. @최철균 P1 승인 후 공식 종결 처리.**

## 8. Change Boundary

- intent: S3 Phase 6 3차 시정분에 대한 Codex 4차 검수 결과 기록
- touched files: `docs/개발문서/S3/phase6/산출물/Phase6_Codex_4차_검수보고서.md`
- changed functions: 없음
- behavior changes: 없음
- tests added/updated: 없음
- validation performed:
  - `pytest tests/unit/test_rate_limit_fg61.py tests/unit/test_retention_job_fg62.py tests/unit/test_content_sanitizer_fg63.py tests/unit/test_admin_org_guard_fg64.py tests/unit/test_admin_org_isolation_routes_fg64.py --no-cov` → 46 passed
  - `pytest tests/unit/ --ignore=tests/unit/test_annotations_service_fg33.py --no-cov -q` → 2748 passed, 13 skipped
  - `pytest tests/security/ --no-cov -q` → 250 passed, 1 skipped
- risks: 본 보고서는 문서 추가만 수행. 남은 항목은 명시적 이월이며 Phase 6 종결 blocker 아님
- open questions: @최철균 P1 승인 기록을 어느 산출물에 최종 반영할지 결정 필요
