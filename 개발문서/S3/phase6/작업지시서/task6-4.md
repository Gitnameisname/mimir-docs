# task6-4 — Admin API organization_id 격리 일괄 패치

**Phase**: S3 Phase 6 / FG 6-4
**작성일**: 2026-05-18
**Handoff Level**: `extended` (보안 P0 — 권한 경계 변경)
**Approver**: @최철균

---

## 1. 의도

Phase 3 FG 3-2 §3 T5 에서 식별된 잠재 위험 — admin (ORG_ADMIN) 이 다른 조직의
scope_profile / agent / user / organization 자원을 변경 가능한 경로 — 를 일괄
차단. SUPER_ADMIN 만 횡단 허용하되, 횡단 호출은 별 `admin.cross_org_access`
event_type 으로 audit emit 의무.

## 2. 산출물

| 파일 | 변경 |
|------|------|
| `backend/app/utils/admin_org_guard.py` | 신설 — `is_super_admin` + `actor_org_ids` + `ensure_actor_can_access_org` |
| `backend/app/api/v1/scope_profiles.py` | 5 scope-profile 엔드포인트 + 5 agent 엔드포인트에 가드 통합 |
| `backend/app/api/v1/admin.py` | `assign_user_org_role` / `remove_user_org_role` / `update_organization` / `delete_organization` 에 가드 통합 |
| `backend/tests/unit/test_admin_org_guard_fg64.py` | 회귀 8건 |
| `산출물/FG6-4_Admin엔드포인트_점검표.md` | 전수 점검표 (별 산출물) |

## 3. R-O4 (Admin 격리) 준수 확인

가드 규칙 (`ensure_actor_can_access_org`):
  1. SUPER_ADMIN → 항상 통과 + audit emit (`admin.cross_org_access`).
  2. ORG_ADMIN 이고 target_org_id 가 본인 user_org_roles 에 있으면 통과.
  3. target_org_id 가 None 인데 non-SUPER_ADMIN → 거부 (fail-closed).
  4. 미인증 → 거부.

list/get/update/delete 각 엔드포인트는:
- 생성: body.organization_id 직접 검사.
- list: organization_id 미지정 시 non-SUPER_ADMIN 은 본인 첫 org 으로 강제.
- get/update/delete: 자원 조회 후 자원의 `organization_id` 로 검사.

## 4. 패치된 엔드포인트 — Phase 6 §3 표 일치

| 엔드포인트 | 메서드 | 가드 | 비고 |
|---|---|---|---|
| `/admin/scope-profiles` | POST | body.organization_id | |
| `/admin/scope-profiles` | GET (list) | non-SUPER_ADMIN auto-scope | |
| `/admin/scope-profiles/{id}` | GET | resource.organization_id | |
| `/admin/scope-profiles/{id}` | PUT (update) | resource.organization_id | |
| `/admin/scope-profiles/{id}` | DELETE | resource.organization_id | |
| `/admin/agents` | POST | body.organization_id | |
| `/admin/agents` | GET (list) | non-SUPER_ADMIN auto-scope | |
| `/admin/agents/{id}` | GET | resource.organization_id | |
| `/admin/agents/{id}` | PUT (update) | resource.organization_id | |
| `/admin/agents/{id}` | DELETE | resource.organization_id | |
| `/admin/users/{id}/org-roles` | POST | body.org_id | role mapping |
| `/admin/users/{id}/org-roles/{org_id}` | DELETE | path org_id | |
| `/admin/organizations/{org_id}` | PATCH | path org_id | |
| `/admin/organizations/{org_id}` | DELETE | path org_id | |

→ 총 14 엔드포인트. SUPER_ADMIN 횡단 시 audit_events 에 `admin.cross_org_access` emit.

## 5. SUPER_ADMIN 횡단 시 UI 추가 confirm 여부 (§6.2 보류 결정)

본 라운드 결정: 별도 UI confirm 안 함. audit emit + 별 event_type 으로 충분히
추적 가능. Admin UI 의 별 라운드에서 분리 검토.

## 6. 회귀 (`tests/unit/test_admin_org_guard_fg64.py`)

`.venv/bin/python -m pytest tests/unit/test_admin_org_guard_fg64.py` → 8 passed.

| ID | 시나리오 |
|---|---|
| G1 | `is_super_admin` SUPER_ADMIN ↔ ORG_ADMIN/VIEWER/None 분류 |
| G2 | SUPER_ADMIN 횡단 통과 + `admin.cross_org_access` emit |
| G3 | ORG_ADMIN 본인 조직 통과 |
| G4 | ORG_ADMIN 다른 조직 거부 (`ApiPermissionDeniedError`) |
| G5 | target_org_id None + non-SUPER_ADMIN 거부 |
| G6 | 미인증 actor 거부 |
| G7 | `actor_org_ids` role_filter 동작 |
| G8 | 미인증 actor → 빈 set |

## 7. Phase 4 R5 contract drift 패턴 재사용

Phase 4 의 `tests/integration/test_mcp_rest_drift.py` 가 보여준 contract drift
패턴 — "REST 의 어떤 엔드포인트가 의도된 권한 게이트를 우회 가능한가" — 를
admin 격리 점검표 (`FG6-4_Admin엔드포인트_점검표.md`) 작성에 그대로 적용.

## 8. 범위 밖

- Admin UI confirm dialog — 별 라운드.
- 다중 ORG_ADMIN org 매핑 사용자의 list 응답 통합 — 본 라운드는 첫 org 강제,
  사용자 명시 시 검증. 다중 org 동시 응답은 별 ADR.
