# MCP Tool ACL 모델

ScopeProfile.allowed_tools — tool-level ACL 의 정본.

**선행 문서**: [MCP 표면 개요.md](MCP%20표면%20개요.md)
**소스 정본**: `backend/app/models/scope_profile.py::ScopeProfile.allowed_tools`

---

## 1. 두 차원의 ACL

본 모델은 **두 차원** 으로 구성:

| 차원 | 책임 | 정본 |
|---|---|---|
| **Per-tool manifest** | 도구가 시스템에 존재하는가? L0~L4 분류, 노출 가능 여부 | `TOOL_SCHEMAS` (관리자 정책) |
| **Per-(profile, tool) ACL** | 본 ScopeProfile 의 에이전트가 이 도구를 호출할 수 있는가? | `scope_profiles.allowed_tools` (운영자 화이트리스트) |

두 차원 **모두 통과** 해야 dispatch.

```
                  manifest 차원                       ACL 차원
[tools/call] -- _CURATED_TOOLS 검사 -- _check_tool_allowed --> [도구 함수]
              (R1: L4 차단)            (R2: ScopeProfile 단일 결정점)
```

---

## 2. ScopeProfile.allowed_tools 컬럼

`scope_profiles.allowed_tools` (Alembic revision `s3_p4_scope_profile_allow_tools`):

```sql
ALTER TABLE scope_profiles
    ADD COLUMN allowed_tools JSONB NOT NULL DEFAULT '[]'::jsonb;

CREATE INDEX idx_scope_profiles_allowed_tools_nonempty
    ON scope_profiles ((jsonb_array_length(allowed_tools) > 0))
    WHERE jsonb_array_length(allowed_tools) > 0;
```

**default-deny**: 빈 배열 — 운영자 명시 등록 전까지 모든 MCP tool 호출 거부.

**검증**: `_allowed_tools_validate(tools)` 가 `known_tool_names()` 외 거부 (manifest 정합 강제).

---

## 3. 진입점 게이트

모든 `tool_*` 함수 첫 줄:

```python
def tool_X(request, actor, conn):
    _check_agent_write_blocked(actor)        # 킬스위치
    _check_tool_allowed(actor, "X", conn=conn)  # ScopeProfile.allowed_tools 검사
    # ... 도메인 코어 호출
```

**`_check_tool_allowed` 분기**:

| actor.actor_type | scope_profile_id | allowed_tools 에 tool_name | 결과 |
|---|---|---|---|
| ≠ AGENT | (any) | (any) | **통과** — 사람/시스템은 본 게이트 비대상 (별 게이트가 처리) |
| AGENT | None | (any) | **거부** (default-deny) |
| AGENT | <id> | 포함 | **통과** |
| AGENT | <id> | 미포함 | **거부** — `MCPError(UNAUTHORIZED, 403)` |
| AGENT | (any) | 조회 실패 (DB 장애 등) | **거부** (fail-closed) |

`actor.can_call_tool(tool_name, conn)` 메서드가 위 분기를 구현 — `app/api/auth/models.py`.

---

## 4. tools/list per-actor 필터

`GET /api/v1/mcp/tools` 응답이 인증된 에이전트에 대해 `allowed_tools` 로 필터링:

```python
def mcp_list_tools(request, actor):
    exposed = mcp_exposed_tool_schemas()
    if actor.is_authenticated and actor.is_agent:
        allowed = _agent_allowed_tools(actor)  # ScopeProfile.allowed_tools set
        if allowed is not None:
            exposed = [s for s in exposed if s["name"] in allowed]
    return [mcp_exposed_public_view(s) for s in exposed]
```

**효과**: 에이전트의 `tools/list` 응답에는 자신이 호출 가능한 도구만 출현. 비허용 도구는 외부에서 보이지 않음.

**비-에이전트** (사람/시스템) 는 manifest 노출 도구 전체 (운영자 / 디버깅 용).

---

## 5. 운영 절차 — ScopeProfile 등록

### 5.1 신규 ScopeProfile 생성 (FG 4-5 `use_defaults` 옵션)

```python
# 옵트인 — default_enabled=True 도구 자동 등록
ScopeProfileRepository.create(
    name="my_profile",
    use_defaults=True,    # FG 4-5 옵션 (기본 False)
)
# 결과 allowed_tools = ['fetch_node', 'read_annotations', 'read_document_render',
#                       'search_documents', 'search_nodes', 'verify_citation']
```

`use_defaults=False` (기본) 또는 `allowed_tools` 명시 입력 시 default-deny 보존.

### 5.2 기존 ScopeProfile 갱신 (Admin UI)

`AdminScopeProfilesPage.tsx` → `AllowedToolsToggleGroup`:
- `/api/v1/admin/mcp/manifest` 동적 fetch
- `is_mcp_exposed=true` 도구만 토글 후보
- `policy_profile` 색상 (read_safe=green, write_audited=orange, experimental=yellow)
- 토글 → PUT `/api/v1/admin/scope-profiles/{id}` body: `{"allowed_tools": [...]}`

### 5.3 Bootstrap 스크립트 (FG 4-0 시드)

`backend/scripts/migrate_scope_profile_allowed_tools.py`:

```bash
# Option A (default-deny 유지)
python scripts/migrate_scope_profile_allowed_tools.py

# Option B (L0 4개 자동 등록)
python scripts/migrate_scope_profile_allowed_tools.py --bootstrap-with-l0 --apply
```

---

## 6. Audit (FG 4-0 §2.1.6.g)

`scope_profile.allowed_tools.changed` event_type — `PUT /admin/scope-profiles/{id}` 가 allowed_tools 를 변경할 때 emit:

```json
{
  "event_type": "scope_profile.allowed_tools.changed",
  "actor_id": "<admin_user>",
  "resource_type": "scope_profile",
  "resource_id": "<sp_id>",
  "metadata": {
    "before": ["search_documents", ...],
    "after": ["search_documents", "save_draft", ...],
    "affected_agents": [{"id": "...", "name": "..."}]
  }
}
```

보안 정책 변경의 audit trail. 운영자가 누가 / 언제 / 어떤 권한을 변경했는지 추적 가능.

---

## 7. 위협 모델 (FG 4-0 §보안 보고서)

| # | 위협 | 본 모델의 대응 |
|---|---|---|
| T1 | 에이전트가 ScopeProfile 외 도구 호출 | `_check_tool_allowed` 진입점 게이트 + default-deny |
| T2 | manifest 와 ScopeProfile drift (잘못된 도구 이름 등록) | `_allowed_tools_validate` 가 `known_tool_names()` 외 거부 |
| T3 | DB 조회 실패 시 게이트 무력화 (fail-open) | `can_call_tool` 의 catch-all `except Exception: return False` (fail-closed) |
| T4 | scope_profile_id None 인 에이전트 → 게이트 우회 | 첫 분기 — `not self.scope_profile_id: return False` |
| T5 | 사람/시스템 actor 가 본 게이트로 차단 | `actor_type != AGENT` 시 통과 (사람은 별 게이트가 처리) |
| T6 | per-actor 필터의 캐시 stale → ScopeProfile 변경 후 stale 응답 | 캐시 미적용 — 매 호출마다 ScopeProfile lookup (성능 vs 일관성, 일관성 우선) |

---

## 8. FG 4-6 영향 — write 도구 (`save_draft`)

`save_draft` 가 L2 write 도구로 추가 (FG 4-6) 하면서:

- `default_enabled=False` — 자동 등록 절대 금지 (write 도구 보안 정책)
- `policy_profile="write_audited"` — Admin UI 가 orange 색상 + 경고 표시
- 운영자가 ScopeProfile.allowed_tools 에 `save_draft` 명시 등록 의무
- `_check_tool_allowed("save_draft")` 첫 줄 적용 — 다른 도구와 동일 게이트

---

## 9. 변경 이력

| 일자 | 변경 |
|---|---|
| 2026-04-28 | 초판 — FG 4-0 ScopeProfile.allowed_tools + FG 4-5 use_defaults + FG 4-6 write 도구 영향 |
