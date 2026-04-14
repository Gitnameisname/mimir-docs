# Task 14-14 보안 취약점 검사 — 배치 작업 관리

작성일: 2026-04-14
작성자: Mimir Team

---

## 1. 요약

Phase 14-14 (배치 작업 관리) 의 구현 전반을 대상으로 OWASP Top 10 관점의 정적 검사를 수행했다. **Critical / High 위험 없음**. cron 표현식 파싱은 외부 라이브러리 취약점을 피하기 위해 순수 stdlib 로 구현되었으며, `eval` / `exec` 호출을 AST 기반으로 검증(세 파일 모두 통과) 했다.

| 심각도 | 건수 | 비고 |
|---|---|---|
| Critical | 0 | — |
| High | 0 | — |
| Medium | 0 | — |
| Low / Info | 2 | 권고 사항 (§4) |

---

## 2. 위협 모델 & 통제

| 위협 | 완화 통제 |
|---|---|
| **A01 접근 통제 실패** | 모든 엔드포인트 `Depends(require_admin_access)` — `admin.read` / `admin.write` 스코프. UI 측에서도 `SUPER_ADMIN` 외 실행·편집·취소 버튼 비활성화 (서버가 최종 방어). |
| **A02 암호화/민감 데이터** | `job_schedules` 에 저장하는 값은 cron 표현식·이름·설명뿐. 비밀번호/토큰 미포함. |
| **A03 인젝션 (SQL)** | repository 의 모든 `cur.execute` 는 `%s` 바인딩. 동적 UPDATE 는 **허용 컬럼 화이트리스트 `{schedule, enabled, next_run_at}`** 만 f-string 에 포함. 사용자 입력은 값으로만 바인딩. |
| **A03 인젝션 (Cron RCE)** | cron 문자 화이트리스트 정규식 `^[0-9*,/\-]+$` 로 각 필드 검사 → `;`, `$(...)`, `` ` `` 등 셸 메타문자 전면 차단. cron 파싱은 `eval`/`exec`/`compile` 미사용 (순수 문자열 split + 정수 변환). |
| **A04 설계 결함 (DoS)** | cron `next_run` 은 **4년 탐색 상한**(무효 표현식으로 인한 무한 루프 방지), 표현식 길이 120자 상한. 5초 폴링은 실행 중일 때만 — idle 시 폴링 없음. |
| **A05 보안 설정** | DDL 은 `CREATE TABLE IF NOT EXISTS`, 시드는 `ON CONFLICT DO NOTHING` 으로 멱등. `TIMESTAMPTZ` 로 타임존 명시. |
| **A06 취약 의존성** | `cronstrue` / `croniter` 등 외부 라이브러리 **미사용** (CLAUDE.md §2). 순수 stdlib 만 사용. |
| **A07 인증 실패** | 엔드포인트는 `require_admin_access` 의존성으로 API 키 스코프 검증 + JWT / 세션 검증. 스케줄 ID 는 `_SCHEDULE_ID_RE = ^[a-z0-9_]{1,100}$` 로 화이트리스트. |
| **A08 무결성 실패** | 수동 실행(`POST /run`) 시 **실행 중이면 409** — 중복 enqueue 방지. 취소는 graceful 플래그만 설정(runner 가 안전 시점 종료). 감사 이벤트(`JOB_SCHEDULE_RUN` / `_UPDATED` / `_CANCELLED`) 로 모든 변경 추적. |
| **A09 로깅** | 감사 로그에 `actor_id` · 대상 `schedule_id` · 결과 기록. 실패 시 `error_code` / `error_message` 기록. |
| **A10 SSRF** | 본 기능은 외부 네트워크 호출 없음 (cron 해석/다음 시각 계산만). 해당 없음. |

---

## 3. 코드 레벨 정적 검사

### 3.1 eval / exec 직접 호출

AST 기반 검사 (`_has_eval_or_exec_call`) — 주석·문자열에 "eval" 포함되어도 오탐 없음.

| 파일 | 결과 |
|---|---|
| `backend/app/services/cron_util.py` | ✅ 없음 |
| `backend/app/api/v1/admin.py` | ✅ 없음 |
| `backend/app/repositories/job_schedule_repository.py` | ✅ 없음 |

### 3.2 SQL 인젝션 검사

- `f"INSERT"`, `f"UPDATE job_schedules"` 형태의 값 삽입 f-string **없음** (`grep` + 수동 확인).
- 동적 컬럼명은 오직 `allowed` 집합(`{schedule, enabled, next_run_at}`)의 키로만 삽입 — 외부 입력 키는 `if k not in allowed: continue` 로 폐기.
- `background_jobs` 로의 enqueue/cancel 쿼리도 `%s` 바인딩.

### 3.3 입력 검증

- `schedule_id` URL path 매개변수: `^[a-z0-9_]{1,100}$` → 미스매치 시 422.
- `UpdateJobScheduleBody` Pydantic v2 모델: `schedule` Optional, `enabled` Optional. 추가 필드 무시(Pydantic 기본).
- cron 표현식은 반드시 `cron_util.validate()` 거친 뒤 저장 — 잘못된 입력 시 `ValueError` → 400.

### 3.4 ReDoS / 정규식 복잡도

`_FIELD_PATTERN = ^[0-9*,/\-]+$` — 단일 문자 클래스, 백트래킹 없음. `_SCHEDULE_ID_RE` 도 단순 charset — ReDoS 불가.

### 3.5 권한 경계 (Frontend)

UI 의 `canEdit = hasRole("SUPER_ADMIN")` 는 UX 보조용이며, 서버 측 `require_admin_access` 가 최종 방어. 권한 없는 사용자가 직접 API 를 호출해도 403.

### 3.6 인코딩 (XSS / Path injection)

- 프론트 API 클라이언트: 경로 매개변수 `encodeURIComponent(jobId)` 적용.
- 서버 응답은 JSON — HTML 렌더 시 React 자동 escape.

---

## 4. 권고 사항 (Low / Info)

1. **Info — cron 다음 실행 계산 한도 모니터링**: `next_run` 의 4년 상한은 충분하지만, 추후 `@monthly` 같은 Unix 확장을 지원하게 되면 상한 재검토.
2. **Low — graceful cancel 후속 감사**: 현재는 `CANCELLED` 로 상태 변경하는 시점에 감사 로그가 남지만, runner 가 실제로 중단 완료했을 때의 완료 이벤트는 별도로 남지 않는다. 운영 상 필요 시 runner 측에서 보완 이벤트 발행 검토.

---

## 5. 검사 결과

```
$ python3 backend/tests/unit/test_admin_jobs_phase14_14.py
[SEC] 3/3
  ✅ SEC-01 cron_util 에 eval/exec 호출 없음 (AST)
  ✅ SEC-02 admin.py Phase 14-14 섹션에 eval/exec 호출 없음 (AST)
  ✅ SEC-03 repository 에 eval/exec 호출 없음 (AST)

합계: 78/78 PASS  (0 FAIL)
```

Critical / High 위험 없음. 운영 승인 가능.
