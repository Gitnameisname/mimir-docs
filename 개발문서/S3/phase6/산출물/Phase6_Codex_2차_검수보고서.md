# S3 Phase 6 Codex 2차 검수보고서 — 운영 안전

**작성일**: 2026-05-18
**검수자**: Codex
**Phase**: S3 / Phase 6
**Handoff Level**: extended (Rate Limit + DB 폐기 + 보안 P0)
**검수 판정**: 공식 종결 보류

---

## 1. 검수 범위

Phase 6 개발계획서와 작업지시서, 산출물, 실제 backend 구현을 대조했다.

- 계획/지시서
  - `docs/개발문서/S3/phase6/Phase 6 개발계획서.md`
  - `docs/개발문서/S3/phase6/작업지시서/task6-1.md`
  - `docs/개발문서/S3/phase6/작업지시서/task6-2.md`
  - `docs/개발문서/S3/phase6/작업지시서/task6-3.md`
  - `docs/개발문서/S3/phase6/작업지시서/task6-4.md`
- 주요 구현
  - `backend/app/api/v1/annotations.py`
  - `backend/app/api/v1/contributors.py`
  - `backend/app/api/v1/notifications.py`
  - `backend/app/services/retention_job.py`
  - `backend/app/utils/content_sanitizer.py`
  - `backend/app/utils/admin_org_guard.py`
  - `backend/app/api/v1/scope_profiles.py`
  - `backend/app/api/v1/admin.py`
- 신규 회귀
  - `backend/tests/unit/test_rate_limit_fg61.py`
  - `backend/tests/unit/test_retention_job_fg62.py`
  - `backend/tests/unit/test_content_sanitizer_fg63.py`
  - `backend/tests/unit/test_admin_org_guard_fg64.py`

## 2. 실행 검증

실행 명령:

```bash
cd backend
./.venv/bin/python -m pytest \
  tests/unit/test_rate_limit_fg61.py \
  tests/unit/test_retention_job_fg62.py \
  tests/unit/test_content_sanitizer_fg63.py \
  tests/unit/test_admin_org_guard_fg64.py
```

결과:

```text
37 passed in 6.19s
```

주의: 기존 산출물 일부는 신규 회귀를 39건으로 기재했으나, 실제 수집/실행된 테스트는 37건이다.

## 3. 종합 판정

| 항목 | 판정 | 근거 |
|---|---|---|
| R-O1 Rate Limit 단일 진입점 | 조건부 통과 | annotations / contributors / notifications 핸들러에 `@limiter.limit` 적용 확인 |
| R-O2 Archive-first retention | 불합격 | annotation retention DELETE 기준이 archive 성공 집합이 아님 |
| R-O3 Content sanitize | 조건부 통과 | write reject / read sanitize 단위 회귀 통과 |
| R-O4 Admin organization 격리 | 보완 필요 | guard 단위 회귀는 있으나 14 endpoint route-level 검증 부족 |
| 함수도서관 동시 갱신 | 불합격 | 신규 public/effectful/policy 함수 등록 누락 |
| 산출물 정합성 | 보완 필요 | 테스트 건수 및 일부 검수 결론이 실제 검수와 불일치 |

따라서 Phase 6는 개발 완료 상태로는 볼 수 있으나, P1 승인 및 공식 종결은 보류한다.

## 4. 발견 이슈

### P1-1. annotation retention 이 archive-first 보장을 깨뜨릴 수 있음

- 위치: `backend/app/services/retention_job.py`
- 관련 코드: `_run_annotations_retention()`
- 문제:
  - `annotations_archive` INSERT 는 `ON CONFLICT (id) DO NOTHING RETURNING id` 를 사용한다.
  - 그러나 DELETE 는 `archived` CTE가 아니라 `expired_all` 후보 집합 기준으로 수행한다.
  - archive row가 이미 존재하거나 일부 row가 archive되지 않은 경우에도 source row가 삭제될 수 있다.
- 영향:
  - Phase 6 R-O2 "archive-first" 위반.
  - 운영 데이터 소실 가능성.
- 추가 위험:
  - `expired_all` 은 root + 직속 답글만 선택한다. nested reply가 존재하면 archive되지 않은 채 root 삭제의 FK cascade로 함께 삭제될 수 있다.
- 수정 요구:
  - DELETE 기준을 `archived RETURNING id` 로 제한한다.
  - 답글 수집은 recursive CTE로 전체 descendant를 포함하거나, 명시적으로 "1-depth reply만 지원" 제약을 서비스/DB/테스트로 고정한다.
  - archive 성공 row 수와 deleted row 수 불일치 시 rollback 또는 error로 처리하는 회귀를 추가한다.

### P1-2. 함수도서관 갱신 누락

- 위치: `docs/함수도서관/backend.md`
- 누락 대상:
  - `app.utils.content_sanitizer.reject_dangerous_chars`
  - `app.utils.content_sanitizer.sanitize_for_response`
  - `app.utils.admin_org_guard.is_super_admin`
  - `app.utils.admin_org_guard.actor_org_ids`
  - `app.utils.admin_org_guard.ensure_actor_can_access_org`
  - `app.services.retention_job.RetentionJob`
  - `app.services.retention_job.run_retention_job`
- 문제:
  - AGENTS.md / CONSTITUTION 제12조는 public/tool/effectful/io/policy 함수 추가 또는 변경 시 중앙 함수도서관 동시 갱신을 요구한다.
  - 현재 `rg "content_sanitizer|admin_org_guard|retention_job" docs/함수도서관/backend.md` 결과가 없다.
- 영향:
  - CI 또는 PR 리뷰 게이트에서 차단되어야 하는 규칙 위반.
  - 후속 에이전트가 기존 함수를 재사용하지 못하고 중복 구현할 위험.
- 수정 요구:
  - `docs/함수도서관/backend.md` 에 Phase 6 신규 유틸/서비스 항목을 추가한다.
  - 각 항목에 purpose, effects, errors, source, tests, 사용지, 운영 주의사항을 포함한다.

### P2-1. Admin 14 endpoint 격리 검증이 route-level 로 입증되지 않음

- 위치:
  - `backend/tests/unit/test_admin_org_guard_fg64.py`
  - `docs/개발문서/S3/phase6/산출물/FG6-4_Admin엔드포인트_점검표.md`
- 문제:
  - 점검표는 14 endpoint 각각의 4 시나리오를 검증했다고 서술한다.
  - 실제 테스트는 `ensure_actor_can_access_org()` 단위 분기만 검증한다.
  - 라우터별 guard 호출 누락, 잘못된 `target_org_id`, list auto-scope, dependency/auth 경로 오류를 잡지 못한다.
- 영향:
  - R-O4 "Admin 은 자기 조직만"의 endpoint-level 보장이 부족하다.
- 수정 요구:
  - 최소한 다음 route-level regression을 추가한다.
    - scope profile create/list/get/update/delete
    - agent create/list/get/update/delete
    - user org-role assign/remove
    - organization patch/delete
  - 각 그룹에서 ORG_ADMIN same-org, ORG_ADMIN cross-org deny, SUPER_ADMIN cross-org audit emit을 검증한다.

### P2-2. 산출물의 회귀 수치와 실제 테스트 수 불일치

- 위치:
  - `docs/개발문서/S3/phase6/산출물/Phase6_1라운드_종결보고서.md`
  - `docs/개발문서/S3/phase6/산출물/FG6-1_검수보고서.md`
  - `docs/개발문서/S3/phase6/산출물/FG6-3_검수보고서.md`
- 문제:
  - 종결보고서는 신규 회귀 39건을 주장한다.
  - 실제 실행 결과는 37건이다.
  - `test_rate_limit_fg61.py` 는 6건, `test_content_sanitizer_fg63.py` 는 14건으로 문서 표와 맞지 않는다.
- 영향:
  - 검수 산출물 신뢰도 저하.
  - 승인자가 실제 검증 범위를 오판할 수 있다.
- 수정 요구:
  - 산출물의 테스트 건수와 실행 결과를 실제 pytest 결과 기준으로 정정한다.

## 5. 추가 관찰

- Rate limit 적용 자체는 문서의 limit 값과 대체로 일치한다. 다만 Phase 계획의 기대 결과는 "per-user rate-limit"라고 표현하지만 실제 구현은 기존 slowapi client IP 기반 패턴을 따른다. task6-1에서 범위 밖으로 명시했으므로 이번 라운드에서는 문서 표현을 "기존 limiter 기반"으로 정리하는 편이 정확하다.
- Content sanitize는 byte/control layer로 분리되어 있고, annotation response 직렬화와 mention snippet 저장 경로에 연결되어 있다. HTML/XSS sanitize와는 별 레이어라는 설계도 문서와 일치한다.
- SUPER_ADMIN cross-org audit emit은 helper 단위에서 mock 검증된다. 다만 실제 `audit_emitter.emit` 실패를 warning 후 통과시키므로, "감사 로그 의무"를 강제 요건으로 볼지 운영 정책 결정이 필요하다.

## 6. 종결 전 필수 조치

1. `RetentionJob._run_annotations_retention()` 의 DELETE 기준을 archive 성공 row 기준으로 수정한다.
2. nested reply archive/delete 정책을 명시하고 회귀를 추가한다.
3. `docs/함수도서관/backend.md` 를 Phase 6 신규 함수 기준으로 갱신한다.
4. admin 14 endpoint 중 핵심 경로에 route-level org isolation 회귀를 추가한다.
5. Phase 6 산출물의 테스트 건수와 판정을 실제 결과에 맞게 정정한다.
6. 위 수정 후 신규 회귀와 관련 backend unit/security baseline을 재실행한다.

## 7. Change Boundary

- intent: S3 Phase 6 구현 완료분에 대한 Codex 2차 검수 결과를 산출물로 기록
- touched files: `docs/개발문서/S3/phase6/산출물/Phase6_Codex_2차_검수보고서.md`
- changed functions: 없음
- behavior changes: 없음 (문서 추가만)
- tests added/updated: 없음
- validation performed: `./.venv/bin/python -m pytest tests/unit/test_rate_limit_fg61.py tests/unit/test_retention_job_fg62.py tests/unit/test_content_sanitizer_fg63.py tests/unit/test_admin_org_guard_fg64.py` → 37 passed
- risks: 본 보고서는 코드 수정 없이 검수 판정만 기록한다. P1 이슈 수정 전 Phase 6 공식 종결 승인 금지
- open questions: SUPER_ADMIN audit emit 실패를 경고 후 통과로 둘지, P1 감사 의무 위반으로 실패 처리할지 승인자 결정 필요

## 8. 결론

Phase 6는 신규 단위 회귀가 통과했지만, retention 데이터 소실 가능성과 함수도서관 규칙 위반이 있어 `extended` 게이트를 통과하지 못한다.

**판정: 공식 종결 보류. P1-1 / P1-2 수정 후 재검수 필요.**
