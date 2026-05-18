# task6-2 — Retention 정책 + cron

**Phase**: S3 Phase 6 / FG 6-2
**작성일**: 2026-05-18
**Handoff Level**: `extended` (DB 데이터 폐기 — P1)
**Approver**: @최철균

---

## 1. 의도

`audit_events.event_type='document.viewed'` 와 해결된 annotation 은 본질적으로
운영 가치가 짧고 PII 누설 가능성이 있다. archive-first 패턴으로 별 archive
테이블에 이동시킨 후 source 에서 DELETE 한다 (R-O2). cron 스케줄로 일 1회
off-peak 시간 자동 실행 + dry-run 모드 + 환경변수 기반 일수/시각 제어.

## 2. 산출물

| 파일 | 변경 |
|------|------|
| `backend/app/db/migrations/versions/20260518_1200_s3_p6_retention_archive.py` | alembic 마이그레이션 — `audit_events_archive` / `annotations_archive` 신설 |
| `backend/app/db/connection.py` | 두 archive DDL 을 init_db 시퀀스에 추가 (dev/test 환경 bootstrap) |
| `backend/app/services/retention_job.py` | `RetentionJob` + `run_retention_job` — INSERT INTO ... SELECT + DELETE 트랜잭션 |
| `backend/app/scheduler.py` | `_default_retention_job` + `get_retention_scheduler()` + `start_scheduler()` 에 retention thread 추가 |
| `backend/tests/unit/test_retention_job_fg62.py` | 회귀 9건 (dry-run, archive-first, env override) |

## 3. R-O2 (archive-first) 준수 확인

- 모든 source DELETE 는 `WITH archived AS (INSERT ... RETURNING id) DELETE FROM ...`
  단일 트랜잭션 안에서 실행. archive 가 빈 결과면 DELETE 도 0.
- `RETENTION_DRY_RUN=1` 또는 인스턴스 인자 `dry_run=True` 일 때는 COUNT 만 실행,
  `INSERT`/`DELETE` 모두 skip + `conn.commit()` 호출 0.
- annotations 의 답글은 root 의 expired set 에 포함시켜 함께 archive (cascade 답글
  보존). source DELETE 는 root id 로 `parent_id` FK ON DELETE CASCADE 가 처리.

## 4. 환경변수

| 변수 | 기본 | 의미 |
|---|---|---|
| `RETENTION_DOCUMENT_VIEWED_DAYS` | 7 | viewed audit 보존일 |
| `RETENTION_RESOLVED_ANNOTATION_DAYS` | 90 | 해결된 annotation 보존일 |
| `RETENTION_BATCH_LIMIT` | 500 | 한 회 처리 row 상한 |
| `RETENTION_DRY_RUN` | (unset) | "1"/"true" 시 dry-run |
| `RETENTION_CRON_HOUR` | 2 | 0~23. 매일 hh:00 실행 |
| `RETENTION_BATCH_ENABLED` | true | "false" 시 scheduler thread 비활성 |

## 5. 마이그레이션 가이드

→ `산출물/FG6-2_마이그레이션_가이드.md` 참고.

## 6. 회귀 (`tests/unit/test_retention_job_fg62.py`)

| ID | 시나리오 | 결과 |
|---|---|---|
| R1 | dry-run — `commit()` 호출 0, candidate 카운트만 노출 | ✅ |
| R2 | 실제 실행 — `commit()` 2회 (audit + annotations) | ✅ |
| R3 | env 변수 override (`RETENTION_DOCUMENT_VIEWED_DAYS=30`) | ✅ |
| R4 | env 변수 잘못된 값 → 기본 fallback | ✅ |
| R5 | `RETENTION_DRY_RUN=1` truthy 파싱 | ✅ |
| R6 | `_retention_schedule()` default `"0 2 * * *"` | ✅ |
| R7 | `RETENTION_CRON_HOUR=5` → `"0 5 * * *"` | ✅ |
| R8 | `RETENTION_CRON_HOUR=99` → fallback `"0 2 * * *"` | ✅ |
| R9 | 상수 `DEFAULT_VIEWED_DAYS==7 / DEFAULT_RESOLVED_ANNOTATION_DAYS==90` | ✅ |

## 7. 운영 절차

1. 첫 배포 시: `RETENTION_DRY_RUN=1` 로 1~2일 운영. log `retention_job_result`
   에서 `document_viewed.candidates`, `resolved_annotations.candidates` 확인.
2. 운영자 검토 후 dry-run 해제, archive 활성.
3. archive 자체의 sub-retention (1년 후 cold storage / 2년 PURGE) 은 별 라운드.

## 8. 범위 밖

- 외부 SIEM / Slack 알림 (audit_events) — S4.
- archive 의 hard delete (PURGE) — 별 도구 / 별 라운드.
- GDPR / 한국 개인정보법 우편 응답 — 별 Compliance.
