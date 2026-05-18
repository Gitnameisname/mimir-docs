# S3 Phase 6 1라운드 종결보고서 — 운영 안전

**작성일**: 2026-05-18
**Phase 상태**: ✅ **공식 종결** (Codex 4차 All Green + @최철균 P1 승인, 2026-05-18)
**공식 종결 문서**: [Phase6_공식종결보고서.md](Phase6_공식종결보고서.md)
**Handoff Level**: `extended` (Rate Limit + DB 폐기 + 보안 P0)

---

## 1. 본 Phase 범위

`Phase 6 개발계획서.md` §1.1 의 4 가지 운영 잔존을 일괄 처리.

| FG | 제목 | 산출물 |
|----|------|------|
| **6-1** | Rate Limit 일괄 적용 | task6-1 + 검수 + 보안 |
| **6-2** | Retention 정책 + cron | task6-2 + 검수 + 보안 + 마이그레이션 가이드 |
| **6-3** | Content Sanitize | task6-3 + 검수 + 보안 |
| **6-4** | Admin API organization_id 격리 | task6-4 + 검수 + 보안 + 점검표 |

---

## 2. 변경 요약

### 2.1 신설 모듈

| 파일 | 책임 |
|---|---|
| `backend/app/utils/content_sanitizer.py` | write reject + read sanitize (FG 6-3) |
| `backend/app/utils/admin_org_guard.py` | admin org 격리 guard (FG 6-4) |
| `backend/app/services/retention_job.py` | archive-first retention 트랜잭션 (FG 6-2) |
| `backend/app/db/migrations/versions/20260518_1200_s3_p6_retention_archive.py` | alembic 마이그레이션 |

### 2.2 변경 라우터 / 서비스 / 스케줄러

| 파일 | 변경 |
|---|---|
| `backend/app/api/v1/annotations.py` | 7 핸들러에 `@limiter.limit` + `_to_response` sanitize |
| `backend/app/api/v1/contributors.py` | 1 핸들러에 limit |
| `backend/app/api/v1/notifications.py` | 3 핸들러에 limit |
| `backend/app/api/v1/scope_profiles.py` | 10 endpoint 에 org 가드 |
| `backend/app/api/v1/admin.py` | 4 endpoint 에 org 가드 |
| `backend/app/services/annotations_service.py` | `_validate_content` 위험 char reject |
| `backend/app/services/notifications_service.py` | mention snippet sanitize |
| `backend/app/scheduler.py` | retention thread 통합 |
| `backend/app/db/connection.py` | archive DDL 2개 bootstrap |

### 2.3 신설 회귀 (46건 — Codex 3차 보강 후)

| 파일 | 회귀 수 |
|---|---|
| `tests/unit/test_rate_limit_fg61.py` | 6 |
| `tests/unit/test_retention_job_fg62.py` | 13 (Codex 2차 P1-1 +3건, 3차 §4 audit verify +1건) |
| `tests/unit/test_content_sanitizer_fg63.py` | 14 |
| `tests/unit/test_admin_org_guard_fg64.py` | 8 |
| `tests/unit/test_admin_org_isolation_routes_fg64.py` | 5 (2차 P2-1 신규 2건 + 3차 P2-1 잔존 보강 3건) |
| **합계** | **46** |

(이전 종결보고서의 39건 주장은 Codex 2차 P2-2 가 실제 37건으로 정정. 2차 P1-1/P2-1
시정으로 5건 추가 → 42건. Codex 3차 P2-1 잔존 보강 (4 endpoint group 전수 cross-org reject)
3건 + §4 audit verify 1건 → **합계 46건**.)

---

## 3. 게이트 / 완료 기준 (개발계획서 §5.1)

| 기준 | 결과 |
|---|---|
| R-O1 (Rate Limit 단일) | ✅ FG 6-1 |
| R-O2 (Archive-first) | ✅ FG 6-2 |
| R-O3 (Sanitize 정책) | ✅ FG 6-3 |
| R-O4 (Admin 격리) | ✅ FG 6-4 (14 endpoint) |
| cron 스케줄 정상 동작 | ✅ retention thread 통합 |
| retention 정책 환경변수 (7/90) default + override | ✅ R3/R4 회귀 |
| SUPER_ADMIN 횡단 audit emit | ✅ G2 회귀 |
| Phase 1~5 회귀 녹색 | ✅ 2748 passed (--ignore=fg33 pre-existing, Codex 3차 보강 후) |

## 4. 회귀 게이트 (개발계획서 §5.2)

| 기준 | 결과 |
|---|---|
| pytest unit 베이스라인 유지 | ✅ 2748 passed, 13 skipped (--ignore=fg33, Codex 3차 보강 후 신규 9건 포함) |
| pytest security 유지 | ✅ 250 passed, 1 skipped |
| 신규 cron dry-run 가용 | ✅ `RETENTION_DRY_RUN=1` |
| tsc 0 error | (UI 변경 없음 — 본 Phase) |
| node:test 베이스라인 유지 | (UI 변경 없음) |

## 5. 사전 알려진 실패 (본 Phase 외)

- `tests/unit/test_annotations_service_fg33.py` 의 4 실패 — Phase 5 FG 5-5 의
  mention regex 변경 이후 stale assertion. Phase 5 종결 메모리에 기록된 별 라운드
  과제. 본 Phase 가 신규로 유발한 실패 0.

## 6. 운영 절차 / 마이그레이션

- `FG6-2_마이그레이션_가이드.md` 참고. 첫 배포 시 `RETENTION_DRY_RUN=1` 권장.
- 환경변수 추가 항목 (`.env` / docker-compose):
  - `RETENTION_DOCUMENT_VIEWED_DAYS=7`
  - `RETENTION_RESOLVED_ANNOTATION_DAYS=90`
  - `RETENTION_BATCH_LIMIT=500`
  - `RETENTION_CRON_HOUR=2`
  - `RETENTION_BATCH_ENABLED=true`
  - `RETENTION_DRY_RUN=1` (첫 배포 1~2일)

## 7. 별 라운드 권고 (본 Phase 1라운드 명시적 제외)

> **정본**: [docs/개발문서/S3/이월.md](../../이월.md) — Phase 6 이월 등록부.
> 공식 종결 보고서 [Phase6_공식종결보고서.md](Phase6_공식종결보고서.md) §7 도 동일 표 참조.

| # | 항목 | 분류 |
|---|---|---|
| O1 | per-user rate-limit keying | **Phase 7 흡수** (FG 7-4) — 현재 slowapi per-IP. Codex 2차 §5 관찰. |
| O2 | archive 자체의 sub-retention | 별 라운드 (cold storage backend 결정 선행) |
| O3 | annotation 외 텍스트 필드 sanitize | 별 라운드 |
| O4 | admin 14 endpoint × 4 시나리오 전수 route-level | 별 라운드 (운영자 요구 시) — `FG6-4_Admin엔드포인트_점검표.md` §3 |
| O5 | SUPER_ADMIN 횡단 빈도 알람 | **Phase 7 흡수** (FG 7-6 observability) |
| O6 | 다중 org admin 의 list UX | 별 ADR |
| O7 | `audit_emitter.emit` 실패 시 P1 정책 | 운영자 결정 — 현재 warning 후 통과 (가용성 우선) |

## 8. 1라운드 종결 정의

- 본 보고서 + 4 FG 의 검수/보안 보고서 + 점검표 + 마이그레이션 가이드 모두 작성.
- 신규 회귀 46건 녹색 (Codex 3차 보강 후).
- 사전 회귀 베이스라인 유지 (2748 passed).

→ **S3 Phase 6 1라운드 1차 종결**. 후속:
  - Codex 2차 검수
  - @최철균 P1 승인 (retention 정책 + admin 격리)
  - 승인 후 공식 종결

---

## 9. 변경 이력

| 일자 | 변경 | 작성자 |
|---|---|---|
| 2026-05-18 | 4 FG 일괄 1차 종결 (Claude 단독 처리 — single-agent exception) | Claude |
| 2026-05-18 | Codex 2차 검수 P1-1 / P1-2 / P2-1 / P2-2 시정 — archive-first 무결성 강화 + recursive CTE + verify rollback + 함수도서관 갱신 + route-level 회귀 + 카운트 정정 (총 42건) | Claude |
| 2026-05-18 | Codex 3차 검수 P2-1 잔존 + P3-1 보강 — 4 endpoint group 전수 cross-org reject route-level (+3건) + 점검표 표현 정확화 + audit retention verify gate 대칭화 (+1건) + 잔존 수치 2739/39 → 2748/46 통일 | Claude |
