# S3 Phase 6 Codex 3차 검수 시정 보고서

**작성일**: 2026-05-18
**대상**: `Phase6_Codex_3차_검수보고서.md` 의 잔존 P2-1 / P3-1 + §4 추가 권고
**작성자**: Claude (single-agent exception 연장)
**Handoff Level**: `extended`

---

## 1. 시정 요약

| Codex 3차 finding | 심각도 | 시정 상태 | 변경 위치 |
|---|---|---|---|
| P2-1 잔존: 산출물 표현이 실제 회귀 범위보다 강함 | P2 | ✅ 완료 — 표현 정확화 + route-level 3건 추가 | 점검표/검수보고서 + `test_admin_org_isolation_routes_fg64.py` |
| P3-1 잔존: 일부 문서에 이전 baseline (2739/39) 잔존 | P3 | ✅ 완료 | 4 FG 검수보고서 + 종결보고서 |
| §4 audit retention verify gate 대칭화 | 권고 | ✅ 완료 | `retention_job.py` + 회귀 1건 |

---

## 2. P2-1 잔존 시정 — 표현 정확화 + 실제 회귀 범위 확장

### 2.1 산출물 표현 정확화 (Codex 권고 1)

`FG6-4_Admin엔드포인트_점검표.md` §1 의 검증 방식 표현을 다음 3축 결합으로
명시:

| 축 | 회귀 | 대상 |
|---|---|---|
| (a) helper 단위 분기 | `test_admin_org_guard_fg64.py` (8건) | 4 분기 |
| (b) **대표 route-level** | `test_admin_org_isolation_routes_fg64.py` (5건) | 4 endpoint group 각각 + SUPER_ADMIN audit emit |
| (c) 코드 점검표 | 본 문서 (수동) | 14 endpoint 의 guard wired 여부 |

이전: "각 endpoint 의 4 시나리오 회귀 결과" (실제보다 강한 표현).
현재: "(a) + (b) + (c) 3축 결합" (실제 회귀 범위와 일치).

14 endpoint × 4 시나리오 전수 route-level 회귀는 §3 의 별 라운드 항목으로 이관.

### 2.2 route-level 회귀 3건 추가 (Codex 권고 2 — 부분 채택)

본 라운드에서는 권고 2 의 "14 endpoint 전수" 가 아닌, **4 endpoint group 각각의
대표 1건** 으로 보강 (helper 가 단일 진입점이므로 group 당 1건이 wired 검증에
충분).

| 신규 테스트 | 검증 |
|---|---|
| `test_route_scope_profile_delete_cross_org_reject` | scope_profiles 모듈 — ORG_ADMIN cross-org → 403 |
| `test_route_user_org_role_assign_cross_org_reject` | admin.py users — ORG_ADMIN cross-org POST → 403 + `assign_org_role` 미호출 |
| `test_route_organization_patch_cross_org_reject` | admin.py organizations — ORG_ADMIN cross-org PATCH → 403 + `update` 미호출 |

기존 (Codex 2차 P2-1 신설) + 본 라운드 추가 = **route-level 5건**:

| 파일 | 건수 | 검증 |
|---|---|---|
| `test_admin_org_isolation_routes_fg64.py` | 5 | agent / scope-profile / user-org-role / organization cross-org reject (4건) + SUPER_ADMIN cross-org audit emit (1건) |

### 2.3 실행

```
$ .venv/bin/python -m pytest tests/unit/test_admin_org_isolation_routes_fg64.py
5 passed
```

---

## 3. §4 추가 권고 — audit retention verify gate 대칭화

### 3.1 권고 내용

Codex 3차 §4 (코드 검수 결과 — Retention):
> verify SELECT가 annotation에만 적용되어 있고 audit_events에는 없다. ... 완전한
> 대칭성을 원하면 audit에도 같은 verify gate를 추가할 수 있다.

### 3.2 시정 — `backend/app/services/retention_job.py`

`_run_audit_view_retention()` 에 annotation 과 동일 패턴 verify gate 추가:

```python
if deleted > 0:
    cur.execute("""SELECT COUNT(*)::int AS c FROM (
                     SELECT id FROM audit_events_archive WHERE id = ANY(%s)
                   ) t""", ([self._row_id(r) for r in deleted_rows],))
    archived_count = ...
    if archived_count != deleted:
        self._conn.rollback()
        raise RuntimeError(
            "audit retention archive-first violation: "
            f"deleted={deleted} archived={archived_count}"
        )
```

annotation 과 audit 양쪽이 같은 무결성 정책으로 보호 — 코드 대칭성 + 운영 일관성.

### 3.3 회귀 추가 (`tests/unit/test_retention_job_fg62.py`)

| 테스트 | 검증 |
|---|---|
| `test_audit_archive_first_violation_rolls_back` | audit verify 불일치 시 `status='partial'` + `errors` 메시지 "audit retention archive-first" + `rollback()` 호출 |

`test_actual_run_invokes_commit` 및 `test_audit_retention_uses_deletable_union` 도
새 audit verify SELECT 시퀀스에 맞게 fetchone side_effect 조정.

### 3.4 실행

```
$ .venv/bin/python -m pytest tests/unit/test_retention_job_fg62.py
13 passed
```

---

## 4. P3-1 잔존 시정 — baseline 수치 통일

### 4.1 변경 전 잔존

| 위치 | 이전 수치 |
|---|---|
| `Phase6_1라운드_종결보고서.md` §4 회귀 게이트 표 | `2739 passed` |
| `Phase6_1라운드_종결보고서.md` §8 종결 정의 | `신규 회귀 39건 녹색` |
| `FG6-1_검수보고서.md` §4 | `2739 passed` |
| `FG6-2_검수보고서.md` §6 | `2739 passed` |
| `FG6-3_검수보고서.md` §6 | `2739 passed` |
| `FG6-4_검수보고서.md` §5 | `2739 passed` |

### 4.2 변경 후 (Codex 3차 보강 반영)

| 위치 | 변경 후 |
|---|---|
| 종결보고서 §2.3 | 신규 회귀 합계 **46건** + 표 갱신 |
| 종결보고서 §4 회귀 게이트 | **2748 passed, 13 skipped** |
| 종결보고서 §8 | "신규 회귀 46건 녹색" / "베이스라인 2748 passed" |
| 4 FG 검수보고서 | **2747~2748 passed** (실측치) |
| `docs/함수도서관/backend.md` §9.6 | 합계 46건 + baseline 2748 |

### 4.3 회귀 카운트 진화

| 시점 | 회귀 수 | 비고 |
|---|---|---|
| 초안 (이전 종결보고서) | 39 | 주장값 — pytest 결과와 불일치 |
| Codex 2차 측정 | 37 | 실제 pytest collection |
| Codex 2차 시정 후 | 42 | P1-1 +3, P2-1 신설 +2 |
| Codex 3차 시정 후 | **46** | P2-1 잔존 +3 (route-level 그룹 보강), §4 audit verify +1 |

---

## 5. 회귀 게이트 (3차 시정 후)

```
$ .venv/bin/python -m pytest \
    tests/unit/test_rate_limit_fg61.py \
    tests/unit/test_retention_job_fg62.py \
    tests/unit/test_content_sanitizer_fg63.py \
    tests/unit/test_admin_org_guard_fg64.py \
    tests/unit/test_admin_org_isolation_routes_fg64.py --no-cov
46 passed

$ .venv/bin/python -m pytest tests/unit/ \
    --ignore=tests/unit/test_annotations_service_fg33.py --no-cov -q
2748 passed, 13 skipped

$ .venv/bin/python -m pytest tests/security/ --no-cov -q
250 passed, 1 skipped
```

사전 실패 4건 (`test_annotations_service_fg33.py`) 은 Phase 5 FG 5-5 stale —
본 Phase 영향 0.

---

## 6. Open question 처리 (Codex 3차 §7)

Codex 3차 §7 open question:
> admin 14 endpoint route-level 전수 회귀를 이번 Phase 종결 조건으로 볼지,
> 대표 route-level 회귀 + 코드 점검으로 승인할지 운영자 결정 필요

본 라운드 권고 (Claude 측):
- **대표 route-level + 코드 점검** 으로 충분 — 14 endpoint 가 단일 helper
  (`ensure_actor_can_access_org`) 의 wrapper 이므로 분기 회귀 (helper 8건) + 4
  endpoint group 각각의 route-level (5건) + 코드 점검표 (수동) 의 3축 결합이
  실질적 보장.
- 14 endpoint 전수는 운영자가 요구 시 별 라운드 추가 가능 — `점검표.md` §3 에
  "다음 라운드 점검 권고" 로 이관.

최종 결정은 @최철균 P1 승인 단계에서 결정.

---

## 7. Change Boundary (3차 시정 라운드)

- **intent**: Codex 3차 검수의 잔존 P2-1 / P3-1 + §4 권고 시정.
- **handoff level**: `extended`.
- **touched files**:
  - `backend/app/services/retention_job.py` (§4 — audit verify gate 추가)
  - `backend/tests/unit/test_retention_job_fg62.py` (§4 — 회귀 +1건, 시퀀스 조정)
  - `backend/tests/unit/test_admin_org_isolation_routes_fg64.py` (P2-1 — 회귀 +3건)
  - `docs/함수도서관/backend.md` (§9.6 카운트 갱신)
  - `docs/개발문서/S3/phase6/산출물/FG6-{1,2,3,4}_검수보고서.md` (baseline 정정)
  - `docs/개발문서/S3/phase6/산출물/FG6-4_Admin엔드포인트_점검표.md` (표현 정확화)
  - `docs/개발문서/S3/phase6/산출물/Phase6_1라운드_종결보고서.md` (전체 정정)
- **changed functions**: `RetentionJob._run_audit_view_retention` — verify gate 추가.
- **behavior changes**: audit retention 도 archive 무결성 verify (annotation 과 대칭).
  실패 시 rollback + RuntimeError.
- **tests added**: 4건 (route-level 3 + audit verify 1).
- **validation performed**: 위 §5.
- **risks**: audit verify SELECT 1회 추가 — 운영 성능 영향 미미. deleted=0 이면 skip.
- **open questions**: 14 endpoint 전수 route-level 회귀 — §6 권고 + 운영자 결정.

---

## 8. 결론

Codex 3차 검수의 잔존 P2-1 / P3-1 + §4 권고 모두 시정.

- **코드**: archive-first 무결성이 annotation + audit 양쪽 대칭으로 보장됨.
- **회귀**: 46건 (이전 42 + 4) 모두 ✅. unit 베이스라인 2748 passed.
- **문서**: 산출물의 표현이 실제 회귀 범위와 1:1 일치. 카운트 통일.
- **open question**: 14 endpoint 전수 route-level 은 별 라운드 (P1 승인 시 운영자 결정).

**판정 (Claude 측)**: 종결 조건 충족.

**다음 단계**: Codex 4차 검수 (필요 시) + @최철균 P1 승인.
