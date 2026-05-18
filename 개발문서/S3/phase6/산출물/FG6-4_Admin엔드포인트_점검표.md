# FG 6-4 Admin 엔드포인트 점검표

**작성일**: 2026-05-18
**대상**: `/api/v1/admin/*` + `/api/v1/admin/scope-profiles/*` + `/api/v1/admin/agents/*`
**Phase 4 R5 contract drift 패턴 재사용**: 본 점검은 "어떤 endpoint 가 의도된
권한 게이트를 우회 가능한가" 검증 형태로 수행한다.

---

## 1. 본 라운드 격리 적용 endpoint (14개)

| # | Endpoint | Method | 가드 위치 | 보호 대상 org_id 출처 |
|---|---|---|---|---|
| 1 | `/admin/scope-profiles` | POST | `create_scope_profile` | body.organization_id |
| 2 | `/admin/scope-profiles` | GET | `list_scope_profiles` | non-SUPER_ADMIN auto-scope |
| 3 | `/admin/scope-profiles/{id}` | GET | `get_scope_profile` | resource.organization_id |
| 4 | `/admin/scope-profiles/{id}` | PUT | `update_scope_profile` | resource.organization_id |
| 5 | `/admin/scope-profiles/{id}` | DELETE | `delete_scope_profile` | resource.organization_id |
| 6 | `/admin/agents` | POST | `create_agent` | body.organization_id |
| 7 | `/admin/agents` | GET | `list_agents` | non-SUPER_ADMIN auto-scope |
| 8 | `/admin/agents/{id}` | GET | `get_agent` | resource.organization_id |
| 9 | `/admin/agents/{id}` | PUT | `update_agent` | resource.organization_id |
| 10 | `/admin/agents/{id}` | DELETE | `delete_agent` | resource.organization_id |
| 11 | `/admin/users/{id}/org-roles` | POST | `assign_user_org_role` | body.org_id |
| 12 | `/admin/users/{id}/org-roles/{org_id}` | DELETE | `remove_user_org_role` | path org_id |
| 13 | `/admin/organizations/{org_id}` | PATCH | `update_organization` | path org_id |
| 14 | `/admin/organizations/{org_id}` | DELETE | `delete_organization` | path org_id |

**검증 방식** (Codex 3차 P2-1 잔존 시정 — 표현 정확화):

본 라운드는 **3축 결합 회귀** 로 검증한다 — endpoint × 시나리오 행렬 전수가 아님.

| 축 | 회귀 | 대상 |
|---|---|---|
| (a) **helper 단위 분기** | `tests/unit/test_admin_org_guard_fg64.py` (8건, G1~G8) | `ensure_actor_can_access_org` 의 4 분기 (SUPER_ADMIN 횡단 + audit emit / ORG_ADMIN 본인 조직 / ORG_ADMIN 다른 조직 reject / target_org_id None reject) |
| (b) **대표 route-level (ASGI stack)** | `tests/unit/test_admin_org_isolation_routes_fg64.py` (5건) | 4 endpoint group 각각의 cross-org reject 1건 (agent / scope-profile / user-org-role / organization) + SUPER_ADMIN cross-org audit emit 1건 (agent) |
| (c) **코드 점검표** (본 문서) | (수동) | 14 endpoint 각각에 `ensure_actor_can_access_org` 호출 또는 `is_super_admin`/`actor_org_ids` 호출이 wired 됨을 코드 review 로 확인 |

각 축의 의미:
- (a) 가 guard 의 **분기 정확성** 을 보장.
- (b) 가 guard 가 ASGI stack 위에서 **실제 wired** 임을 보장 (4 endpoint group 모두 대표 1건 포함).
- (c) 가 14 endpoint 중 어느 하나라도 guard 호출이 누락되지 않았음을 review-time 에 확인.

**14 endpoint × 4 시나리오 전수 route-level 회귀**는 본 라운드 범위 밖 — §3 의 "다음 라운드 점검 권고" 로 이관 (별 라운드).

---

## 2. 본 라운드 격리 미적용 endpoint — 사유

다음 endpoint 는 본 라운드 격리 미적용. 사유 명시:

| Endpoint | 사유 |
|---|---|
| `/admin/dashboard/*` | 운영 메트릭 — organization 무관 또는 SUPER_ADMIN 전용 가정. 별 라운드 검증 권장 |
| `/admin/users` (LIST/GET/POST/PATCH/DELETE) | user 자체는 multi-org 매핑 가능 → org 경계로 표현 불가. 별 ADR (`다중 org user`) |
| `/admin/roles/*` | role 정의는 글로벌 (organization 무관) — 격리 불필요 |
| `/admin/document-types/*` | document_type 은 글로벌 정의 — 격리 불필요 |
| `/admin/audit-logs/*` | SUPER_ADMIN 만 접근 가능 (require_admin_access action 분기) — 본 라운드 미점검, 별 라운드 |
| `/admin/jobs/*`, `/admin/indexing/*` | 작업 큐 — organization 표현 부재. 별 라운드 |
| `/admin/api-keys/*` | API key 는 issuer_id 로 격리됨. 별 라운드 검증 권장 |
| `/admin/settings/*` | system_settings — 글로벌 |
| `/admin/alerts/*` | 운영 알람 — 글로벌 또는 SUPER_ADMIN 전용 |
| `/admin/usage*`, `/admin/agent-activity*` | 메트릭 — 본 라운드 미점검 |
| `/admin/providers/*` | LLM provider — 글로벌 |
| `/admin/prompts/*` | prompt template — 글로벌 |
| `/admin/job-schedules/*` | scheduler 메트릭 — 글로벌 |

---

## 3. 다음 라운드 점검 권고

본 라운드는 보안 영향이 가장 큰 14 endpoint 우선. 별 라운드 권고:

1. `admin_extraction_results.py` — 추출 결과의 org 격리.
2. `admin_alerts` — 알람 정의의 org scope.
3. `api_keys` — issuer_id ↔ organization 매핑 검증.
4. `audit-logs` 조회 — ORG_ADMIN 이 다른 조직의 audit 조회 가능 여부.
5. `users` 자체 — 다중 org admin 이 본인 첫 org 외 user 조작 가능성.

---

## 4. SUPER_ADMIN 횡단 audit 점검 절차

운영 점검 (월 1회 권장):

```sql
SELECT actor_user_id, COUNT(*) AS cnt, MAX(occurred_at) AS last_seen
FROM audit_events
WHERE event_type = 'admin.cross_org_access'
  AND occurred_at > NOW() - INTERVAL '30 days'
GROUP BY actor_user_id
ORDER BY cnt DESC;
```

비정상적으로 잦은 SUPER_ADMIN 횡단은 운영자 검토 대상.

---

## 5. 결론

본 라운드 14 endpoint 격리 완료. 미점검 endpoint 는 별 라운드.
SUPER_ADMIN 횡단은 audit emit 으로 추적.
