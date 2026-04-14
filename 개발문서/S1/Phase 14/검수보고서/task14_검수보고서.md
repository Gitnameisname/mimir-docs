# Task 14-14 검수 보고서 — 배치 작업 관리

작성일: 2026-04-14
작성자: Mimir Team

---

## 1. 요약

Phase 14-14 (배치 작업 관리) 의 백엔드 · 프론트엔드 구현, UI 리뷰 5회, 그리고 정적 검증 스크립트(78 항목) 전체 PASS 를 달성했다. `/admin/jobs` 가 **스케줄** / **실행 이력** 2개 탭으로 분리되어, 새로 추가한 `AdminJobSchedulesPage` 가 기본 탭으로 노출된다. cron 표현식은 외부 라이브러리 없이 순수 stdlib 으로 파싱 · 설명 · 다음 실행 예측을 제공하며(CLAUDE.md §2), 실행 중인 스케줄이 있을 때만 5초 폴링된다. 수동 실행 · 스케줄 변경 · graceful 취소는 `SUPER_ADMIN` 으로 제한되고 감사 이벤트가 발행된다.

| 영역 | 결과 |
|---|---|
| DDL (`job_schedules` + 시드) | 8/8 PASS |
| Repository (CRUD / runs / enqueue / cancel) | 10/10 PASS |
| Cron util (validate / describe_ko / next_run) | 12/12 PASS |
| 보안 SEC (AST 기반 eval/exec 스캔) | 3/3 PASS |
| API (6 라우트, RBAC, 감사 이벤트) | 14/14 PASS |
| 프론트 타입 / API 클라이언트 | 5/5 + 3/3 PASS |
| 라우팅 (탭) | 3/3 PASS |
| UI (상태 색상 · 모달 · a11y · 폴링) | 13/13 PASS |
| 반응형 | 5/5 PASS |
| 구문(SYNTAX) | 2/2 PASS |
| **합계** | **78/78 PASS** |

---

## 2. 구현 내용

### 2.1 DB 스키마 (`backend/app/db/connection.py`)

- `job_schedules(id VARCHAR(100) PK, name, description, schedule VARCHAR(100), enabled BOOL, last_run_at, last_run_duration_ms INT, last_run_result, last_run_id FK→background_jobs ON DELETE SET NULL, next_run_at, created_at, updated_at)`
- 인덱스 `idx_job_schedules_enabled`
- 시드(멱등, `ON CONFLICT (id) DO NOTHING`): `reindex_all`, `vector_sync`, `audit_cleanup`, `token_cleanup`
- 실행 이력은 기존 `background_jobs` 재사용 (`job_type = schedule.id`) — 스키마 중복 방지

### 2.2 Repository (`backend/app/repositories/job_schedule_repository.py`)

- `list_schedules` / `get_schedule` / `update_schedule` (컬럼 화이트리스트 `{schedule, enabled, next_run_at}`)
- `list_recent_runs` — `background_jobs` 조인, 소요 시간 계산(`CAST(EPOCH * 1000 AS INTEGER)`)
- `get_running_run` — 상태 판정용 (PENDING / RUNNING)
- `enqueue_manual_run` — `INSERT INTO background_jobs (...PENDING...)`, runner 가 픽업
- `mark_cancel_requested` — graceful: 상태를 `CANCELLED` 로 표시, runner 가 인지

### 2.3 Cron util (`backend/app/services/cron_util.py`, 외부 의존성 0)

5-field (`m h dom mon dow`) 만 지원, 문자 화이트리스트 정규식 (`^[0-9*,/\-]+$`) 으로 안전 검증:
- `validate(expr)` — 필드 개수·범위·문자 검사, 길이 120자 상한
- `describe_ko(expr)` — 매일 / 매주 월요일 등 한국어 설명
- `next_run(expr, now)` — DOM+DOW OR 시맨틱, 4년 탐색 상한 (무한 루프 방지)

### 2.4 API (`backend/app/api/v1/admin.py`, Phase 14-14 섹션)

- `GET    /admin/jobs/schedules` — 목록 + 상태 파생(idle/running/failed)
- `GET    /admin/jobs/schedules/{job_id}` — 상세 + 최근 실행 10건
- `POST   /admin/jobs/schedules/{job_id}/run` (202) — 409 if running, 감사 `JOB_SCHEDULE_RUN`
- `PATCH  /admin/jobs/schedules/{job_id}` — cron · enabled 수정, cron validate, next_run_at 자동 계산, 감사 `JOB_SCHEDULE_UPDATED`
- `POST   /admin/jobs/schedules/{job_id}/cancel` — graceful, 감사 `JOB_SCHEDULE_CANCELLED`
- `POST   /admin/jobs/schedules/cron/preview` — 설명 + 3 next runs

모든 엔드포인트는 `Depends(require_admin_access)` 로 RBAC. `schedule_id` 는 `^[a-z0-9_]{1,100}$` 화이트리스트.

### 2.5 프론트엔드

- 타입 (`frontend/src/types/admin.ts`): `JobScheduleStatus`, `JobRunResult`, `JobSchedule`, `JobScheduleDetail`, `JobScheduleRun`, `CronPreviewResponse`.
- API 클라이언트 (`frontend/src/lib/api/admin.ts`): 6개 메소드 (`getJobSchedules`, `getJobSchedule`, `runJobSchedule`, `updateJobSchedule`, `cancelJobSchedule`, `previewCron`) — 모두 `encodeURIComponent` 적용.
- 페이지 (`frontend/src/features/admin/jobs/AdminJobSchedulesPage.tsx`)
  - 상태 색상: 대기(회색) / 실행 중(파란 pulse) / 실패(빨강)
  - 행 확장 상세 패널 — 최근 실행 10건 테이블
  - 수동 실행 · 취소 확인 모달 · 스케줄 편집 모달(cron 미리보기 포함)
  - `refetchInterval`: running 존재 시 5000ms, 아니면 `false`
  - 권한: `SUPER_ADMIN` 만 `canEdit` — 실행 / 취소 / 저장 버튼 비활성화
- 라우트 (`frontend/src/app/admin/jobs/page.tsx`): `role="tablist"` 탭으로 **스케줄** 기본 표시, 기존 실행 이력 보기(`AdminJobsPage`) 는 두 번째 탭으로 보존.

---

## 3. UI 디자인 리뷰 (5회)

| 회차 | 주제 | 반영 내용 |
|---|---|---|
| 1 | **상태 시각화** | dot + pulse(실행 중만), 색상 3단계(gray/blue/red), 상태 텍스트 `aria-live` |
| 2 | **권한 분리** | `SUPER_ADMIN` 외에는 실행/편집/취소 버튼 비활성화, 시각적 disabled 스타일 유지 |
| 3 | **안전성** | 수동 실행 확인 모달(작업명 굵게 표시), 취소는 graceful 고지("요청이 접수되면…"), 저장 시 cron 미리보기 검증 |
| 4 | **반응형** | `p-4 sm:p-6`, `text-xl sm:text-2xl`, 테이블 `overflow-x-auto`, 헤더 `flex-wrap`, 모달 `max-w-md` |
| 5 | **접근성** | `scope="col"`, `role="status"`/`role="alert"`, `aria-expanded` 행 확장, `aria-live`, `sr-only` 라벨, `focus-visible:ring` |

---

## 4. 기존 설계와의 차이 (의도적 일탈)

- **네임스페이스 분리**: 과제에 명시된 `/admin/jobs/{job_id}` 는 기존에 UUID 실행 기록에 사용 중이라, 충돌 방지를 위해 스케줄 관리는 `/admin/jobs/schedules/*` 로 분리. UI 는 동일한 `/admin/jobs` 페이지에서 탭으로 통합.
- **`cronstrue` 미사용**: CLAUDE.md §2 (외부 라이브러리 취약점 회피) 에 따라 stdlib 기반 자체 구현 (`app/services/cron_util.py`).
- **실행 이력 테이블 미신설**: 기존 `background_jobs` 재사용 (`job_type = schedule.id`) — 저장 공간 · 복잡도 절감.

---

## 5. 검증 결과

```
$ python3 backend/tests/unit/test_admin_jobs_phase14_14.py
합계: 78/78 PASS  (0 FAIL)
```

전 항목(DDL · REPO · CRON · SEC · API · TYPES · API-FE · ROUTE · UI · RESP · SYNTAX) PASS.
