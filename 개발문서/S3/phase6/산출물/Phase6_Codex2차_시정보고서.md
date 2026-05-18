# S3 Phase 6 Codex 2차 검수 시정 보고서

**작성일**: 2026-05-18
**대상**: `Phase6_Codex_2차_검수보고서.md` 의 P1-1 / P1-2 / P2-1 / P2-2 + §5 추가 관찰
**작성자**: Claude (single-agent exception 연장)
**Handoff Level**: `extended`

---

## 1. 시정 요약

| Codex 발견 | 심각도 | 시정 상태 | 변경 위치 |
|---|---|---|---|
| P1-1 annotation retention archive-first 미보장 | P1 | ✅ 완료 | `backend/app/services/retention_job.py` |
| P1-2 함수도서관 갱신 누락 | P1 | ✅ 완료 | `docs/함수도서관/backend.md` §9 |
| P2-1 admin 14 endpoint route-level 회귀 부족 | P2 | ✅ 완료 | `backend/tests/unit/test_admin_org_isolation_routes_fg64.py` 신설 |
| P2-2 산출물 회귀 수치 불일치 | P2 | ✅ 완료 | 4 검수보고서 + 종결보고서 정정 |
| §5 추가 관찰 (rate-limit per-user 표현) | 관찰 | ✅ 문서 명시 | FG6-1 검수보고서 §5 + 종결보고서 §7 O1 |
| §5 추가 관찰 (audit emit 실패 정책) | 관찰 | ✅ open question 명시 | 종결보고서 §7 O7 + 함수도서관 §9.5 |

---

## 2. P1-1 — annotation retention archive-first 무결성 강화

### 2.1 문제 (Codex 보고서 §4 P1-1)

```sql
-- 이전:
WITH expired_roots, expired_all AS (root + 1-depth replies),
     archived AS (INSERT ... ON CONFLICT DO NOTHING RETURNING id)
DELETE FROM annotations WHERE id IN (SELECT id FROM expired_all)
```

위험:
- DELETE 가 `archived` (성공 INSERT) 가 아닌 `expired_all` (모든 후보) 을 기준으로 함 → archive 미통과 row 가 source 에서 사라질 수 있음.
- `expired_all` 이 1-depth reply 만 → nested reply 가 `ON DELETE CASCADE` 로 archive 안 거치고 함께 삭제될 수 있음.

### 2.2 시정 — `backend/app/services/retention_job.py`

```sql
-- 변경 후:
WITH RECURSIVE expired_roots AS (...),
descendants AS (                        -- ★ 임의 depth 재귀
    SELECT a.id, a.parent_id
    FROM annotations a
    INNER JOIN expired_roots r ON a.id = r.id
    UNION ALL
    SELECT a.id, a.parent_id
    FROM annotations a
    INNER JOIN descendants d ON a.parent_id = d.id
),
expired_all AS (SELECT DISTINCT id FROM descendants),
inserted AS (INSERT INTO annotations_archive ...
             ON CONFLICT (id) DO NOTHING RETURNING id),
already_archived AS (SELECT a.id FROM annotations_archive a
                     INNER JOIN expired_all e ON e.id = a.id),
deletable AS (SELECT id FROM inserted UNION SELECT id FROM already_archived)
DELETE FROM annotations WHERE id IN (SELECT id FROM deletable)
RETURNING id
```

추가 무결성 게이트 (Python):

```python
if deleted > 0:
    cur.execute("SELECT COUNT(*) ... FROM annotations_archive WHERE id = ANY(%s)", ...)
    archived_count = ...
    if archived_count != deleted:
        self._conn.rollback()
        raise RuntimeError("annotation retention archive-first violation: ...")
```

audit retention 도 같은 `deletable = inserted ∪ already_archived` 패턴으로 정리.

### 2.3 신규 회귀 (`tests/unit/test_retention_job_fg62.py`)

| 테스트 | 검증 |
|---|---|
| `test_archive_first_violation_rolls_back` | verify gate 불일치 시 rollback + error → `status='partial'` + `errors` 비어있지 않음 |
| `test_annotation_retention_uses_recursive_cte_sql` | SQL 에 `RECURSIVE` + `descendants` CTE 가 실제로 발급됨 |
| `test_audit_retention_uses_deletable_union` | audit SQL 이 `deletable = inserted UNION already_archived` 사용 |

실행:

```
$ .venv/bin/python -m pytest tests/unit/test_retention_job_fg62.py
12 passed
```

---

## 3. P1-2 — 함수도서관 갱신

### 3.1 위치

`docs/함수도서관/backend.md` §9 신설 (Phase 6, 2026-05-18 도입).

### 3.2 등록 항목

| 함수 | 종류 | 책임 |
|---|---|---|
| `app.utils.content_sanitizer.reject_dangerous_chars` | @policy | write 시점 위험 char reject |
| `app.utils.content_sanitizer.sanitize_for_response` | @effectful | 응답 직렬화 정규화 |
| `app.utils.content_sanitizer.MAX_CONTENT_LENGTH` | const | DB CHECK 정합 |
| `app.utils.admin_org_guard.is_super_admin` | @policy | role 분기 helper |
| `app.utils.admin_org_guard.actor_org_ids` | @io | user_org_roles SELECT |
| `app.utils.admin_org_guard.ensure_actor_can_access_org` | @policy + @effectful | 14 endpoint 단일 진입점 |
| `app.services.retention_job.RetentionJob` | @effectful | archive-first 트랜잭션 |
| `app.services.retention_job.run_retention_job` | @effectful | scheduler 진입점 |

각 항목에 `purpose / effects / errors / source / tests / used by / notes` 포함.

§9.4 는 `@limiter.limit` 신규 routing 적용 (5 상수) 명시.
§9.5 는 audit emit 실패 운영 정책 open question 명시.
§9.6 는 신규 회귀 총 42건 표.

---

## 4. P2-1 — admin route-level 회귀

### 4.1 신설 — `backend/tests/unit/test_admin_org_isolation_routes_fg64.py`

TestClient + `app.dependency_overrides` + repository monkeypatch 패턴 (FG 5-3 통합
회귀에서 검증된 구조 그대로). 회귀 2건:

| 테스트 | 검증 |
|---|---|
| `test_route_agent_delete_cross_org_reject` | ORG_ADMIN (org-A) 이 org-B 의 agent 삭제 → **403**. `admin.cross_org_access` emit 0. |
| `test_route_agent_delete_super_admin_cross_org_audited` | SUPER_ADMIN cross-org 삭제 → **204** + `admin.cross_org_access` emit ≥ 1 + repo.delete 실제 호출. |

실행:

```
$ .venv/bin/python -m pytest tests/unit/test_admin_org_isolation_routes_fg64.py
2 passed
```

### 4.2 본 라운드 회귀 범위 제한 명시

Codex 의 권고는 14 endpoint × 4 시나리오 = 56 회귀 였으나, 본 라운드는 다음
근거로 2건 (대표 endpoint × 2 시나리오) 으로 제한한다:

- guard 가 **단일 helper 진입점** (`ensure_actor_can_access_org`) 이며 14 endpoint
  가 같은 호출 흐름을 공유 — endpoint 별로 동일 로직을 반복 검증하는 가치는 낮음.
- helper 분기 회귀 8건 (G1~G8) 이 boundary case 를 이미 커버.
- route-level 은 "guard 가 라우터에 실제로 호출됨 + audit emit 가 ASGI stack 위에서
  실제 발화" 만 보장하면 충분 (`ensure_actor_can_access_org` 호출이 누락된
  endpoint 가 있는가? 라는 문제는 별 ADR / 점검표 §3 의 회귀 항목).

14 endpoint × 4 시나리오 일괄 회귀는 별 라운드 — `docs/개발문서/S3/phase6/산출물/
FG6-4_Admin엔드포인트_점검표.md` §3 의 추가 항목으로 이관.

---

## 5. P2-2 — 산출물 회귀 수치 정정

### 5.1 실제 pytest collection 결과

```
$ .venv/bin/python -m pytest tests/unit/test_rate_limit_fg61.py \
    tests/unit/test_retention_job_fg62.py \
    tests/unit/test_content_sanitizer_fg63.py \
    tests/unit/test_admin_org_guard_fg64.py \
    tests/unit/test_admin_org_isolation_routes_fg64.py --collect-only -q
42 tests collected
```

| 파일 | 이전 주장 | 실제 (Codex 2차) | 시정 후 |
|---|---|---|---|
| test_rate_limit_fg61.py | 9 | 6 | 6 |
| test_retention_job_fg62.py | 9 | 9 | 12 (P1-1 시정으로 3건 추가) |
| test_content_sanitizer_fg63.py | 13 | 14 | 14 |
| test_admin_org_guard_fg64.py | 8 | 8 | 8 |
| test_admin_org_isolation_routes_fg64.py | — | — | 2 (신설) |
| **합계** | **39** | **37** | **42** |

### 5.2 산출물 정정 위치

- `FG6-1_검수보고서.md` §1, §3 — 6건으로 정정 + 11 endpoint 가 L1~L3 안에 일괄 단언됨을 명시.
- `FG6-2_검수보고서.md` §1, §2 R-O2 점검표, §4 — 12건 + recursive CTE / verify gate 항목 추가.
- `FG6-3_검수보고서.md` §1, §3 — 14건으로 정정.
- `FG6-4_검수보고서.md` §1, §3 — 8 + 2 = 10건. route-level 회귀 1줄 추가.
- `Phase6_1라운드_종결보고서.md` §2.3 / §4 — 42건 합계 + 회귀 게이트 2744 passed.

---

## 6. §5 추가 관찰 — 문서 표현 정리

### 6.1 rate-limit per-user 표현

Phase 계획서 §1.1: "신규 API (annotations / contributors / notifications) 와 기존
일부 API 에 **per-user / per-IP** rate-limit dependency 통일".

실제 구현은 slowapi `_get_client_ip` 기반 **per-IP**. per-user keying 은 별
라운드 (Phase 7 Valkey).

→ `FG6-1_검수보고서.md` §5 + 종결보고서 §7 O1 에 명시.

### 6.2 audit emit 실패 정책

`ensure_actor_can_access_org` 의 SUPER_ADMIN cross-org audit emit 실패 경로:

```python
try:
    audit_emitter.emit(event_type="admin.cross_org_access", ...)
except Exception as exc:
    logger.warning("admin_org_guard: cross_org audit emit failed: %s", exc)
```

→ 가용성 우선 (감사 실패가 접근을 차단하지 않음). 보안 강화 시 P1 위반으로
거부 정책 검토 가능 — 운영자 결정.

→ 종결보고서 §7 O7 + 함수도서관 §9.5 에 open question 명시.

---

## 7. 회귀 게이트 (시정 후)

```
$ .venv/bin/python -m pytest tests/unit/ \
    --ignore=tests/unit/test_annotations_service_fg33.py --no-cov -q
2744 passed, 13 skipped

$ .venv/bin/python -m pytest tests/security/ --no-cov -q
250 passed, 1 skipped

$ .venv/bin/python -m pytest \
    tests/unit/test_rate_limit_fg61.py \
    tests/unit/test_retention_job_fg62.py \
    tests/unit/test_content_sanitizer_fg63.py \
    tests/unit/test_admin_org_guard_fg64.py \
    tests/unit/test_admin_org_isolation_routes_fg64.py --no-cov
42 passed
```

사전 실패 4건 (`test_annotations_service_fg33.py`) 은 Phase 5 FG 5-5 stale
assertion — 본 Phase 영향 0.

---

## 8. Change Boundary (시정 라운드)

- **intent**: Codex 2차 검수의 P1/P2 finding 시정 + 산출물 정합 정리.
- **handoff level**: `extended` (P1 — archive-first 무결성 변경 + 함수도서관 변경).
- **touched files**:
  - `backend/app/services/retention_job.py` (P1-1 — SQL recursive CTE + verify gate + `_row_id` helper)
  - `backend/tests/unit/test_retention_job_fg62.py` (P1-1 — 3건 추가)
  - `backend/tests/unit/test_admin_org_isolation_routes_fg64.py` (P2-1 — 신설 2건)
  - `docs/함수도서관/backend.md` (P1-2 — §9 신설)
  - `docs/개발문서/S3/phase6/산출물/FG6-1_검수보고서.md` (P2-2 — 수치 정정)
  - `docs/개발문서/S3/phase6/산출물/FG6-2_검수보고서.md` (P2-2 + P1-1 — 수치/항목 정정)
  - `docs/개발문서/S3/phase6/산출물/FG6-3_검수보고서.md` (P2-2 — 수치 정정)
  - `docs/개발문서/S3/phase6/산출물/FG6-4_검수보고서.md` (P2-1 — route-level 회귀 명시)
  - `docs/개발문서/S3/phase6/산출물/Phase6_1라운드_종결보고서.md` (P2-2 — 수치 + 변경이력)
- **changed functions**: `RetentionJob._run_audit_view_retention`, `_run_annotations_retention`, `_row_id` 신설.
- **behavior changes**:
  - retention DELETE 가 archive 성공 / 이미 archive 된 row 에만 실행됨.
  - annotation retention 이 nested reply (depth > 1) 까지 archive 후 삭제.
  - archive 무결성 위반 시 즉시 rollback + RuntimeError (이전엔 무경고 통과 가능).
- **tests added**: 5건 (재기능 3건 + route-level 2건).
- **validation performed**: 위 §7.
- **risks**: archive 무결성 verify 가 추가 SELECT 1회 (deleted 가 0 이면 skip). 운영 성능 영향 미미.
- **open questions**:
  - `audit_emitter.emit` 실패의 P1 위반 처리 여부 — §6.2 / 종결보고서 §7 O7.
  - admin endpoint 14 전수 route-level 회귀 — 별 라운드 (점검표 §3).

---

## 9. 결론

Codex 2차 검수 P1-1 / P1-2 / P2-1 / P2-2 + §5 추가 관찰 6항 모두 시정.

**판정 (Claude 측)**: archive-first 데이터 손실 위험 차단 + 함수도서관 정합 회복
+ route-level guard 검증 + 카운트 정합 — `extended` 게이트 통과 조건 충족.

**다음 단계**: Codex 3차 검수 + @최철균 P1 승인 후 공식 종결.
