# FG 3-2 — viewer 정보 노출 경로 전수 점검표

**작성일**: 2026-04-27
**대상**: `task3-2 Step 5` — `audit_events.event_type='document.viewed'` 정보가 정책 게이트(`should_expose_viewers`) 우회 없이 노출되는지 전수 검증

---

## 1. 점검 방법

```bash
grep -rn "document\.viewed\|FROM audit_events\|audit_events WHERE" backend/app --include='*.py'
grep -rn "viewer\|viewed_by" backend/app --include='*.py'
```

검색 후 각 호출자가:
- (a) viewer 정보를 응답에 포함하는가
- (b) 호출자 권한 (admin / user / agent / anonymous) 이 무엇인가
- (c) FG 3-2 정책 게이트 통과가 필요한가

---

## 2. 발견 경로

| # | 위치 | 호출자 권한 | viewer 정보 노출? | 정책 게이트 필요? | 결과 |
|---|-----|-----------|----------------|----------------|-----|
| 1 | `api/v1/contributors.py` `get_contributors` | 인증 사용자 | ✅ Y (viewers 섹션) | ✅ Y | **적용 완료** — `contributors_service.get_contributors` 가 `should_expose_viewers(viewer_actor)` 호출, 거짓이면 viewers 미반환 + 헤더 사유 |
| 2 | `api/v1/admin.py:266` `list recent audit events` | **admin** (require_admin) | ✅ Y (event_type=document.viewed 가 그대로 표시) | ❌ N | **의도된 admin oversight**. 정책 게이트 무관 (조직 정책으로 admin 은 모든 audit 접근). 산출물 §4 참조. |
| 3 | `api/v1/admin.py:394` `user activity` | **admin** (require_admin) | ✅ Y (user 의 최근 audit) | ❌ N | 동일 — admin 전용 |
| 4 | `api/v1/admin.py:675-696` `audit_events list paginated` | **admin** (require_admin) | ✅ Y (event_type 무관 모든 row) | ❌ N | 동일 — admin 전용 |
| 5 | `api/v1/admin.py:715` `audit_events get by id` | **admin** (require_admin) | ✅ Y | ❌ N | 동일 — admin 전용 |
| 6 | `api/v1/admin.py:2173` (조사 — RAG quality 통계) | admin | ❌ N (event_type 통계 집계) | ❌ N | 비식별 통계, viewer 식별 정보 노출 없음 |
| 7 | `api/v1/scope_profiles.py:597-616` `audit log per profile` | **admin** (require_admin) | ❌ N (scope profile 변경 audit 만 — `event_type LIKE 'scope_profile.%'`) | ❌ N | viewer event_type 미포함 |
| 8 | `services/vectorization_status_service.py:130` | service (내부) | ❌ N (vectorization audit only) | ❌ N | 비대상 |
| 9 | `services/alert_evaluator.py:101` | service (배치) | ❌ N (alert rule 평가 — viewer 식별 안 함) | ❌ N | 비대상 |
| 10 | `api/v1/contributors.py` `include_viewers` 쿼리 강제 | 인증 사용자 | (위 #1 의 입력) | (위 #1 의 게이트) | ✅ 게이트가 입력 의사를 덮어쓰므로 우회 불가 |

---

## 3. 결론 — 우회 가능 경로 0

**FG 3-2 정책 게이트가 보호해야 할 사용자-facing 노출 경로는 §2 의 #1 단 한 곳이다.**

- #2~5 는 **admin 전용** — `require_admin` 가드가 통과해야 도달. 조직 정책상 admin 은 모든 audit_events 조회 권한이 있다 (조사 / 사고 추적 / 컴플라이언스).
- #6~9 는 viewer 식별 정보를 응답 / 외부 인터페이스에 포함하지 않는다.
- #1 은 `contributors_service.get_contributors` 가 `should_expose_viewers(viewer_actor)` 를 호출해 정책에 의해 viewers 응답을 차단한다. fail-closed 보장.

**우회 경로 없음.** FG 3-2 의 정책 게이트가 비-admin 사용자에 대한 viewer 노출을 정확히 통제한다.

---

## 4. admin 노출의 정책적 정당성

### 4.1 admin 의 audit_events 접근 정책

조직 정책상 admin 은 audit_events 전수 접근 권한을 가진다:
- 사고 조사 (예: 권한 오용, 데이터 유출 추적)
- 컴플라이언스 보고 (감사 요구)
- Scope Profile 정책 변경 추적

따라서 admin 이 `document.viewed` 이벤트를 보는 것은 **정책상 의도된 동작** 이다.

### 4.2 본 FG 의 적용 범위

- `should_expose_viewers` 는 **일반 사용자** 가 다른 사용자의 view 흔적을 보는 것을 통제한다.
- admin 의 audit_events 직접 조회는 본 FG 의 정책 게이트와 별개의 admin 권한 게이트로 통제된다.
- 향후 admin 도 viewer 식별 정보를 가리는 정책이 필요해지면 별 라운드에서 admin-side `expose_viewers` 게이트 도입 검토.

---

## 5. 별 라운드 후속

| # | 항목 | 우선순위 |
|---|-----|--------|
| 1 | admin 조회 시에도 viewer 식별 정보를 마스킹하는 옵션 | 낮 (조직별 결정) |
| 2 | `document.viewed` event_type 을 audit_events retention 정책에서 더 짧은 보존 (예: 7일) 적용 | 중 (PII 최소화) |
| 3 | `should_expose_viewers` 게이트의 캐시 invalidate 자동화 (settings PATCH 시) | 중 (`scope_profiles_router` PATCH 에서 호출) |

---

*작성: 2026-04-27 | task3-2 Step 5 산출물*
