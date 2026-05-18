# S3 Phase 6 Codex 3차 검수보고서 — 2차 시정분 확인

**작성일**: 2026-05-18
**검수자**: Codex
**대상**: `Phase6_Codex_2차_검수보고서.md` 기반 Claude 시정분
**Handoff Level**: extended (retention DB 폐기 + admin 권한 경계)
**검수 판정**: 코드 P1 해소, 문서 정합성 일부 보완 필요

---

## 1. 검수 범위

- `backend/app/services/retention_job.py`
- `backend/tests/unit/test_retention_job_fg62.py`
- `backend/tests/unit/test_admin_org_isolation_routes_fg64.py`
- `docs/함수도서관/backend.md`
- `docs/개발문서/S3/phase6/산출물/Phase6_Codex2차_시정보고서.md`
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
42 passed in 1.16s
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
2744 passed, 13 skipped, 1 warning in 190.01s
```

### Security baseline

```bash
cd backend
./.venv/bin/python -m pytest tests/security/ --no-cov -q
```

결과:

```text
250 passed, 1 skipped in 3.31s
```

## 3. 2차 Finding 시정 확인

| 2차 Finding | 3차 판정 | 근거 |
|---|---|---|
| P1-1 annotation retention archive-first 미보장 | 해소 | DELETE 기준이 `inserted ∪ already_archived` 로 제한됨. annotation은 recursive CTE로 descendants 수집. verify mismatch 시 rollback + RuntimeError |
| P1-2 함수도서관 갱신 누락 | 해소 | `docs/함수도서관/backend.md` §9에 content sanitizer, admin org guard, retention job, rate-limit routing 등록 |
| P2-1 admin route-level 회귀 부족 | 부분 해소 | route-level 2건 추가. 단, 14 endpoint 전수 route-level 회귀는 여전히 별 라운드로 이관됨 |
| P2-2 산출물 회귀 수치 불일치 | 부분 해소 | 종결보고서 주요 표는 42건/2744건으로 정정. 일부 문서 구간에는 39건/2739건 잔존 |

## 4. 코드 검수 결과

### Retention

`RetentionJob._run_annotations_retention()` 은 2차 P1 이슈를 실질적으로 막는다.

- expired root를 기준으로 `WITH RECURSIVE descendants` 를 구성해 nested reply를 포함한다.
- `INSERT ... ON CONFLICT DO NOTHING RETURNING id` 결과와 이미 archive에 존재하는 id를 `deletable` 로 합친다.
- DELETE는 `deletable` 에 포함된 id에만 수행된다.
- DELETE 후 `annotations_archive WHERE id = ANY(deleted_ids)` count를 확인하고, 불일치 시 rollback한다.

남은 판단 사항:
- 이미 archive에 존재하는 source row를 삭제하는 idempotency 정책은 합리적이다.
- verify SELECT가 annotation에만 적용되어 있고 audit_events에는 없다. audit_events는 FK cascade 계층이 없어 위험이 낮지만, 완전한 대칭성을 원하면 audit에도 같은 verify gate를 추가할 수 있다. 현재는 blocking finding으로 보지 않는다.

### Admin org isolation

신규 route-level 테스트는 실제 ASGI stack 위에서 다음을 확인한다.

- ORG_ADMIN이 다른 조직 agent 삭제 시 403.
- SUPER_ADMIN이 다른 조직 agent 삭제 시 204 + `admin.cross_org_access` emit.

이는 helper 단위 테스트만 있던 2차 상태보다 개선이다. 다만 Phase 6 산출물이 계속 "14 endpoint 격리 완료"를 주장하므로, endpoint별 guard 호출/target_org_id 오배선을 자동으로 잡는 수준까지는 아직 아니다.

## 5. 잔존 Finding

### P2-1. Admin endpoint 전수 route-level 검증과 산출물 표현이 여전히 맞지 않음

- 위치:
  - `docs/개발문서/S3/phase6/산출물/FG6-4_Admin엔드포인트_점검표.md`
  - `docs/개발문서/S3/phase6/산출물/FG6-4_검수보고서.md`
- 문제:
  - 점검표는 "각 endpoint 의 4 시나리오 회귀 결과"라고 표현한다.
  - 실제 route-level 회귀는 agent delete 2건만 추가됐다.
  - 나머지 endpoint는 helper 단위 분기와 코드 점검에 의존한다.
- 영향:
  - 코드상 guard가 들어간 것은 확인되지만, 산출물의 "각 endpoint 회귀" 주장은 실제 테스트 범위보다 강하다.
- 권고:
  - 둘 중 하나를 선택한다.
    - 산출물 표현을 "helper 단위 + 대표 route-level 회귀 + 코드 점검"으로 낮춘다.
    - 또는 14 endpoint별 최소 cross-org reject route-level 테스트를 추가한다.

### P3-1. 산출물 일부에 이전 baseline 수치가 잔존

- 위치:
  - `Phase6_1라운드_종결보고서.md` §8: "신규 회귀 39건 녹색" 잔존.
  - `FG6-1_검수보고서.md`, `FG6-2_검수보고서.md`, `FG6-3_검수보고서.md`, `FG6-4_검수보고서.md`: baseline `2739 passed` 잔존.
- 실제:
  - 신규/수정 회귀: 42 passed.
  - unit baseline: 2744 passed, 13 skipped.
- 영향:
  - 코드 품질에는 영향 없음.
  - 승인 문서의 정합성 측면에서 정정 필요.
- 권고:
  - 위 문서들의 잔존 수치를 42 / 2744 기준으로 통일한다.

## 6. 결론

2차 검수의 P1 코드 이슈는 해소됐다. 특히 retention archive-first 데이터 손실 위험은 현재 구현과 회귀 테스트 기준으로 차단됐다.

공식 종결 전 남은 것은 코드 blocker가 아니라 문서/검증 범위 정합성이다.

- **코드 판정**: 통과
- **테스트 판정**: 통과
- **문서 판정**: P2/P3 보완 필요
- **P1 승인 전 조건**: 위 P2-1 산출물 표현 또는 route-level 회귀 범위 중 하나를 정리하고, 잔존 수치를 정정할 것

## 7. Change Boundary

- intent: S3 Phase 6 2차 시정분에 대한 Codex 3차 검수 결과 기록
- touched files: `docs/개발문서/S3/phase6/산출물/Phase6_Codex_3차_검수보고서.md`
- changed functions: 없음
- behavior changes: 없음
- tests added/updated: 없음
- validation performed:
  - `pytest tests/unit/test_rate_limit_fg61.py tests/unit/test_retention_job_fg62.py tests/unit/test_content_sanitizer_fg63.py tests/unit/test_admin_org_guard_fg64.py tests/unit/test_admin_org_isolation_routes_fg64.py --no-cov` → 42 passed
  - `pytest tests/unit/ --ignore=tests/unit/test_annotations_service_fg33.py --no-cov -q` → 2744 passed, 13 skipped
  - `pytest tests/security/ --no-cov -q` → 250 passed, 1 skipped
- risks: 본 보고서는 문서 추가만 수행. 잔존 P2/P3는 산출물 정합성 이슈이며 코드 동작 회귀는 발견하지 못함
- open questions: admin 14 endpoint route-level 전수 회귀를 이번 Phase 종결 조건으로 볼지, 대표 route-level 회귀 + 코드 점검으로 승인할지 운영자 결정 필요
