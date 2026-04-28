# task4-0 — Pre-flight + 도구 등급화 + manifest 표준 (FG 4-0 선행)

**작성일**: 2026-04-27
**Phase / FG**: S3 Phase 4 / FG 4-0 (선행)
**상태**: 착수 대기
**담당**: 백엔드 (설계 Claude / 구현 Codex / 리뷰 Claude — extended)
**예상 소요**: 3~4일 (§2.1.6 ScopeProfile ACL 흡수로 +1.5~2일)
**선행 산출물**: 없음 (본 FG가 Phase 4 진입 게이트)
**후행**: task4-1 (read 응답 메타데이터 표준)이 본 FG 산출물 위에서 진행

---

## 1. 작업 목적

S3 Phase 4 본편(FG 4-1~4-4) 진입 전, **MCP 도구 표면의 사실 관계를 코드로 확정**한다. 구체적으로 다음 5가지를 산출한다.

1. **Pre-flight 실측** — 현재 코드베이스의 MCP 도구·리소스·프롬프트 표면을 inventory로 만든다.
2. **L0~L4 등급화 매핑** — 각 도구를 헌법 제20조(Typed Action Risk)에 따라 등급 분류.
3. **manifest 표준 부착** — `TOOL_SCHEMAS`에 `risk_tier`/`maturity`/`status`/`exposure_policy` 필드 추가.
4. **L4 차단 회귀 테스트** — `risk_tier=L4` 도구가 절대 MCP 표면에 노출되지 않음을 입증.
5. **ScopeProfile tool-level ACL 도입** — `ScopeProfile.allowed_tools` 필드 + `AgentPrincipal.can_read(tool_name)` 헬퍼 + 모든 MCP tool 진입점 일괄 게이트 (S3 Phase 3 정적 리뷰 P1 흡수, 2026-04-27 — `종결회고.md §4.1 #11` / `FG3-3 §5 #9` 잔존부채 등재). per-tool manifest(§3) 와 함께 R2 (ACL 단일 결정점) 의 tool-level 차원을 닫는다.

본 FG는 (5)의 추가로 코드 변경이 단순히 *적지만*에서 **DB 마이그레이션 + 모델 + 시드 default + Admin UI 토글**까지 포함하는 **extended Handoff Level** 이다. 예상 소요는 1.5~2일 → **3~4일** 로 갱신.

---

## 2. 작업 범위

### 2.1 포함

#### 2.1.1 Pre-flight 실측 (`산출물/Pre-flight_실측.md`)

다음 항목을 실측해 보고서로 남긴다.

- **MCP 도구 인벤토리**: `backend/app/schemas/mcp.py`의 `TOOL_SCHEMAS` + `backend/app/mcp/tools.py`의 `tool_*` 함수 목록을 1:1 대응표로 작성. 누락(스키마만 있고 함수 없음 / 함수만 있고 스키마 없음) 여부 확인.
- **리소스 인벤토리**: `backend/app/mcp/resources.py`의 `parse_resource_uri` 처리 가능 URI 패턴 전수.
- **프롬프트 인벤토리**: `backend/app/mcp/prompts.py`의 `list_mcp_prompts` 결과 전수.
- **REST 엔드포인트 inventory (관련 부분)**: `documents.render`, `documents.versions.render`, `documents.{id}` 노드 조회, `citations` 검증 — REST 측 핵심 6개 엔드포인트의 함수 호출 그래프를 그려 MCP 신규 도구가 wrapper가 아닌 별도 의도임을 사전 확인.
- **현재 manifest 필드 상태**: `risk_tier`/`maturity`/`status`가 빠져 있음을 확인 (배경 명시).
- **Rate Limit 현황**: `mcp_router.py`의 `_MCP_TOOL_LIMIT`/`_MCP_STREAM_LIMIT`/`_MCP_READ_LIMIT`/`_MCP_INIT_LIMIT` + `_AGENT_RATE_LIMITS` 표.
- **prompt_injection_detector 사용 위치**: `mcp_router.py`에서 어느 경로가 detector를 호출하는지 추적. 결과를 응답에 어떻게 반영하는지 확인 (FG 4-1의 `detected_risks` 매핑 입력).
- **scope_filter 적용 경로**: `apply_scope_filter`가 호출되는 모든 도구 확인.

#### 2.1.2 L0~L4 등급화 매핑 (`산출물/도구등급화_매핑.md`)

업로드 요약 §3.1 표 기준으로 현재 + 신규 도구를 매핑한다. **본 FG에서는 매핑만 확정하고, 코드 부착은 §2.1.3에서 한다.**

| 도구 (현재 또는 예정) | 의도 | risk_tier | maturity | status | exposure_policy |
|----------------------|------|-----------|----------|--------|-----------------|
| `search_documents` | 문서 검색 | L0 | stable | enabled | MCP_ENABLED |
| `fetch_node` | 노드 조회 | L0 | stable | enabled | MCP_ENABLED |
| `verify_citation` (현재) | 인용 검증 | L1 | stable | enabled | MCP_ENABLED |
| `verify_citation` (FG 4-3 강화 후) | 5중 검증 | L1 | stable | enabled | MCP_ENABLED |
| `mimir.vectorization.status` | 벡터화 상태 조회 | L0 | beta | enabled | MCP_ENABLED |
| `search_nodes` (FG 4-2 신설) | 노드 단위 검색 | L0 | beta | enabled | MCP_ENABLED |
| `read_document_render` (FG 4-2 신설) | 렌더링 텍스트 조회 | L0 | beta | enabled | MCP_ENABLED |
| `resolve_document_reference` (FG 4-2 신설) | 자연어 참조 정규화 | L1 | experimental | enabled | MCP_ENABLED |
| `save_draft` (S2 자산, 노출 여부 확인) | draft 쓰기 | L2 | beta | (Pre-flight에서 확인) | (FG 4-6 결정 후 확정) |
| `publish_document` (REST에만 존재) | 워크플로 전이 | L3 | stable | not_exposed | REST_ADMIN_ONLY |
| `reindex_document` (REST에만 존재) | 인덱스 재구성 | L4 | forbidden | not_exposed | REST_ADMIN_ONLY |
| `change_schema` (REST 관리자) | 스키마 변경 | L4 | forbidden | not_exposed | REST_ADMIN_ONLY |
| `delete_document` (REST 관리자) | 문서 삭제 | L4 | forbidden | not_exposed | REST_ADMIN_ONLY |

> Pre-flight에서 각 도구의 실제 코드 경로를 확인 후 위 표를 확정한다. **L4 도구는 모두 `not_exposed`** 가 강제되어야 한다(R1).

#### 2.1.3 manifest 표준 코드 부착

`backend/app/schemas/mcp.py`에 다음을 추가한다.

```python
# 새 enum (Literal 또는 Enum) 정의
RiskTier = Literal["L0", "L1", "L2", "L3", "L4"]
Maturity = Literal["stable", "beta", "experimental", "disabled", "forbidden"]
ToolStatus = Literal["enabled", "disabled", "not_exposed"]
ExposurePolicy = Literal["MCP_ENABLED", "MCP_DISABLED", "REST_ADMIN_ONLY"]

# TOOL_SCHEMAS 각 항목에 다음 키 추가
{
    "name": "...",
    "description": "...",
    "risk_tier": "L0",            # 추가 (필수)
    "maturity": "stable",         # 추가 (필수)
    "status": "enabled",          # 추가 (필수)
    "exposure_policy": "MCP_ENABLED",  # 추가 (선택, 기본 MCP_ENABLED)
    "authentication": {...},
    "inputSchema": {...},
}
```

또한 `mcp_router.py`의 `tools/list` 응답 디스패처가 `status` 필드를 읽어 다음을 강제한다:

- `status == "not_exposed"` 또는 `maturity == "forbidden"` 인 도구는 **목록에서 제외**.
- 위 도구를 `tools/call`로 호출하면 `MCPErrorCode.METHOD_NOT_FOUND` (또는 동등)로 거부.

#### 2.1.4 L4 차단 회귀 테스트

`backend/tests/integration/test_mcp_l4_blocked.py` (신규):

1. 가짜 L4 도구 정의를 `TOOL_SCHEMAS`에 임시 등록한 fixture로:
   - `tools/list` 응답에 해당 도구가 포함되지 **않음**을 검증.
   - `tools/call` 호출 시 표준 거부 에러를 반환함을 검증.
2. 정의된 모든 도구를 순회하며 `risk_tier in {"L4"}`인 항목이 있다면 즉시 fail (정의 단계 안전망).
3. `maturity == "forbidden"` 인 항목도 동일.

#### 2.1.5 manifest 자동 추출 스크립트

`backend/scripts/dump_mcp_manifest.py` (신규):

- `TOOL_SCHEMAS`를 읽어 `docs/개발문서/S3/phase4/산출물/MCP_도구_매니페스트.json` 출력.
- CI에서 이 파일을 commit 상태와 비교해 drift 감지(다음 PR 작업).
- 본 FG 단계에서는 추출만 작성하고, drift gate는 FG 4-4에서 통합.

#### 2.1.6 ScopeProfile tool-level ACL 도입 (S3 P3 정적 리뷰 P1 흡수)

> **배경**: S3 Phase 3 task3-3.md §[129,223–225,318] 가 요구한 `ScopeProfile.allowed_tools` 및 `AgentPrincipal.can_read("annotations")` tool-level 게이트가 read_annotations 단일이 아니라 **MCP 표면 전체**에서 미구현으로 확인됨 (2026-04-27 Codex 정적 리뷰 P1). 본 FG 의 manifest 표준화와 같은 시점에 진행하는 것이 R2 (ACL 단일 결정점) 정합·운영 비용 측면에서 자연스럽다.
>
> **per-tool manifest (§2.1.3) vs per-profile ACL (본 절)**: manifest 는 "도구 X 가 시스템에 존재하고 노출되는가" (운영자 정책), 본 절은 "프로파일 P 가 도구 X 를 호출할 수 있는가" (에이전트 권한). 두 축이 모두 통과해야 dispatch 한다.

##### a. DB 스키마 변경 (Alembic revision `s3_p4_scope_profile_allow_tools`)

```sql
ALTER TABLE scope_profiles
  ADD COLUMN allowed_tools JSONB NOT NULL DEFAULT '[]'::jsonb;
-- backfill: 기존 행 모두 빈 배열 (default-deny). 운영자 명시 등록 전까지 모든 도구 거부.
-- 마이그레이션 후 운영자가 Admin UI 또는 `scope_profiles_seed.sql` 로 도구 등록.
```

기존 `scope_profiles.settings_json` (FG 3-2 도입) 와는 별 컬럼. 운영 의미가 다르고 (settings 는 운영 옵션, allowed_tools 는 ACL), 쿼리 빈도/패턴이 달라 분리 유지.

##### b. 모델 / 헬퍼 변경

- `backend/app/models/scope_profile.py` `ScopeProfile` dataclass 에 `allowed_tools: list[str] = field(default_factory=list)` 추가
- `backend/app/repositories/scope_profile_repository.py` row → ScopeProfile 매핑·CRUD 에 `allowed_tools` 반영
- `backend/app/auth/agent_principal.py` (또는 ActorContext 가 사는 모듈) 에 `can_read(tool_name: str) -> bool` 헬퍼 추가
  - 시그니처: `def can_read(self, tool_name: str) -> bool` — `self.scope_profile.allowed_tools` 확인
  - actor_type ≠ agent 인 경우 또는 scope_profile 없는 경우는 `True` (사람 / 시스템 호출은 본 게이트 비대상; 별 게이트로 판단)
  - tool_name 정규화 (예: `read_annotations`, `search_documents` — `TOOL_SCHEMAS.name` 와 정확히 일치)

##### c. MCP 진입점 일괄 게이트

`backend/app/mcp/tools.py` 의 모든 `tool_*` 함수 (현재 4개: search_documents / fetch_node / verify_citation / vectorization_status / read_annotations) 에 표준 헬퍼 호출 추가:

```python
def _check_tool_allowed(actor: ActorContext, tool_name: str) -> None:
    """ScopeProfile.allowed_tools 게이트 — 에이전트가 본 도구를 허용받았는가."""
    if not actor.can_read(tool_name):
        raise MCPError(
            MCPErrorCode.UNAUTHORIZED,
            f"본 ScopeProfile 은 도구 '{tool_name}' 을 허용하지 않습니다.",
            403,
        )
```

각 `tool_*` 의 첫 줄(`_check_agent_write_blocked` 옆) 에 `_check_tool_allowed(actor, "tool_name")` 추가. 또한 `mcp_router.py` 의 `tools/list` 응답 생성기에 **per-actor 필터** 추가 (allowed_tools 외 도구 응답 목록에서 제외 — manifest 의 `not_exposed` 필터와 AND 결합).

##### d. Admin UI 토글

`frontend/src/pages/admin/AdminScopeProfilesPage.tsx` (FG 3-2 에서 도입) 에 `allowed_tools` 체크박스 그룹 추가:

- `TOOL_SCHEMAS` 에서 `status != "not_exposed"` 도구만 토글 후보로 노출
- 저장 시 PATCH `/admin/scope-profiles/{id}` body 에 `allowed_tools: string[]` 포함
- 기본 빈 배열로 도입 (default-deny) — 운영자가 명시 등록

##### e. 시드 / 운영 가이드

`backend/scripts/migrate_scope_profile_allowed_tools.py` (신규, 일회성):

- 옵션 A (기본): 기존 모든 ScopeProfile 의 `allowed_tools` 를 빈 배열 유지 (default-deny). 운영자가 후속 단계에서 Admin UI 로 등록 — **권장**.
- 옵션 B (운영자 선택): `--bootstrap-with-l0` 플래그 시 `risk_tier == "L0"` 도구를 모든 기존 ScopeProfile 에 등록. 신규 ScopeProfile 에는 적용 안 함.

산출물 `Pre-flight_실측.md` 에 본 옵션 결정을 운영자에게 합의 받을 것.

##### f. 회귀 테스트 (≥ 5 시나리오)

`backend/tests/integration/test_mcp_tool_acl.py` (신규):

1. 빈 `allowed_tools` 인 프로파일 → `read_annotations` 호출 시 403 거부
2. `allowed_tools=["read_annotations"]` 인 프로파일 → 호출 통과
3. `allowed_tools=["search_documents"]` 만 있는 프로파일 → `read_annotations` 거부 + `search_documents` 통과
4. `tools/list` 응답이 `allowed_tools` 로 필터링됨 (다른 도구는 보이지 않음)
5. actor_type=user 또는 actor_type=system → 본 게이트 미적용 (다른 게이트 통과)
6. `allowed_tools` 와 `manifest.status=not_exposed` 가 동시 적용 — `not_exposed` 가 우선 (manifest 외 도구는 allowed_tools 등록되어 있어도 거부)

##### g. 검수보고서 §R2 의무 기재

FG 4-0 검수보고서의 "R1~R5 준수 확인" 섹션 중 **R2 (ACL 단일 결정점)** 항목에서 본 게이트의 도입을 명시. S3 Phase 3 정적 리뷰 P1 종결을 함께 기록.

### 2.2 제외 (이월)

- `default_enabled`/`requires`/`preferred_use`/`policy_profile`/`streaming_supported` 등 확장 manifest 필드 — FG 4-5(이월)
- `tools/list` 응답에 `risk_tier`를 외부에 그대로 노출할지 여부의 정책 결정 — Pre-flight에서 후보 결정 후 본 FG 마지막에 확정. (잠정: 외부에는 `maturity`/`description`만 노출, 운영자용 별도 엔드포인트에서 risk_tier 노출 검토.)
- 신규 도구 함수 구현 — FG 4-2

### 2.3 하드코딩 금지 재확인

- 도구 등급 매핑은 `TOOL_SCHEMAS` 코드 상수가 **단일 정본**. 다른 모듈에서 risk_tier를 hardcode 금지.
- `tools/list` 디스패처는 `TOOL_SCHEMAS`를 순회해 필터. tool 이름을 if/else로 분기 금지.

---

## 3. 선행 조건

- S3 Phase 3 게이트 통과 (또는 사용자 명시 승인)
- `Phase 4 개발계획서.md` 사용자 승인 — **P1 변경 — `@최철균` 승인 필수**
- 코드베이스 현 상태 깨끗 (`git status` clean)
- **현 Alembic head = `s3_p2_tags`** (운영자 `alembic upgrade head` 적용 완료, 2026-04-28). S3 Phase 3 스키마 (`scope_profiles.settings_json` + `annotations` / `annotation_mentions` / `notifications`) 는 Alembic 이 아니라 [`db/connection.py`](../../../../../../backend/app/db/connection.py) boot-time `IF NOT EXISTS` DDL 로 적용 — 백엔드 부팅 시 자동 반영. §2.1.6 의 신규 revision `s3_p4_scope_profile_allow_tools` 는 본 FG 구현 단계에서 새로 작성하여 추가 적용 필요 (Step 7).

---

## 4. 구현 단계

### Step 1 — Pre-flight 실측

1. `backend/app/mcp/` + `backend/app/api/v1/mcp_router.py` + `backend/app/schemas/mcp.py` 전수 읽기
2. `Pre-flight_실측.md` 작성 (§2.1.1)
3. 운영자 검토 — 본 보고서가 후속 FG의 사실 기반이므로 1차 리뷰 의무

### Step 2 — 등급화 매핑 합의

1. `도구등급화_매핑.md` 표 작성 (§2.1.2)
2. 각 도구의 등급에 대한 짧은 근거 메모 첨부
3. **L4 도구 = MCP 절대 금지** 규칙을 별도 강조 박스로 표기
4. 사용자 승인 — 등급 매핑은 정책 변경이므로 **extended approval 필수**

### Step 3 — manifest 코드 부착

1. `schemas/mcp.py` Literal 타입 정의
2. `TOOL_SCHEMAS` 4개 항목에 필드 추가 (search_documents/fetch_node/verify_citation/mimir.vectorization.status)
3. 단위 테스트: `TOOL_SCHEMAS`의 모든 항목이 새 필수 필드를 가짐을 schema validation으로 검증

### Step 4 — `tools/list` / `tools/call` 디스패처 갱신

1. `mcp_router.py`의 `tools/list` 응답 생성기에서 `status != "not_exposed"` AND `maturity != "forbidden"` 필터 추가
2. `tools/call`에서 동일 필터 + 거부 시 표준 에러 반환
3. 회귀 테스트: 기존 4개 도구 모두 노출되고 호출 가능함을 회귀 (변경 회귀 보호)

### Step 5 — L4 차단 회귀 테스트

1. `test_mcp_l4_blocked.py` 작성 (§2.1.4)
2. CI에서 자동 실행 확인

### Step 6 — manifest 추출 스크립트

1. `scripts/dump_mcp_manifest.py` 작성
2. 첫 출력 결과를 `산출물/MCP_도구_매니페스트.json`으로 commit

### Step 7 — ScopeProfile.allowed_tools 도입 (§2.1.6 흡수)

1. **DB 마이그레이션 작성** — Alembic revision `s3_p4_scope_profile_allow_tools` (§2.1.6.a). `--sql` dry-run 검증 후 운영자 승인.
2. **모델·Repository 갱신** — ScopeProfile dataclass + repository CRUD + AgentPrincipal/ActorContext 의 `can_read(tool_name)` 헬퍼 (§2.1.6.b)
3. **MCP 진입점 일괄 게이트** — `_check_tool_allowed` 헬퍼 + 5 `tool_*` 함수 첫 줄 추가 + `tools/list` per-actor 필터 (§2.1.6.c)
4. **Admin UI 토글** — `AdminScopeProfilesPage.tsx` 체크박스 그룹 + PATCH body 확장 (§2.1.6.d). UI 리뷰 1회 의무.
5. **시드 / 마이그레이션 스크립트** — `migrate_scope_profile_allowed_tools.py` (§2.1.6.e). 옵션 A/B 운영자 결정 후 적용.
6. **회귀 테스트** — `test_mcp_tool_acl.py` 6 시나리오 (§2.1.6.f). pytest 녹색 + 기존 4 도구 회귀 영향 0.
7. **운영자 인계** — Admin UI 로 기존 ScopeProfile 의 `allowed_tools` 명시 등록 (default-deny 시작이라 등록 전엔 모든 에이전트 호출 거부 — 운영 가이드에 명시).

### Step 8 — 검수 / 보안 보고서

- `FG4-0_검수보고서.md` — R1~R5 준수 확인 섹션 의무. 특히 R1(L4 차단)은 회귀 테스트 결과 첨부.
- `FG4-0_보안취약점검사보고서.md` — manifest의 외부 노출 범위(risk_tier 노출 시 운영 정보 누출 위험) 분석.

---

## 5. API 계약 변경 요약

| 메서드 | 경로 | 변경 |
|-------|------|------|
| POST | /mcp/tools/list | 응답 필터링 — `status=not_exposed`/`maturity=forbidden` 도구 제외 |
| POST | /mcp/tools/call | 거부 정책 — 위 도구 호출 시 `METHOD_NOT_FOUND` 반환 |

응답 envelope 자체는 본 FG에서 변경하지 않는다(FG 4-1 범위).

---

## 6. 데이터 모델 주의사항

본 FG 의 manifest 표준화 부분(§2.1.1~§2.1.5)은 **DB 스키마 변경 없음** — 코드 상수(`TOOL_SCHEMAS`) + 디스패처 + 테스트만.

§2.1.6 ScopeProfile.allowed_tools 도입은 **DB 마이그레이션 1건**:

- Alembic revision `s3_p4_scope_profile_allow_tools` — `scope_profiles.allowed_tools` 컬럼 (`JSONB NOT NULL DEFAULT '[]'::jsonb`)
- backfill: 기존 행 모두 빈 배열 유지 (default-deny). 운영자 후속 등록 전까지 에이전트 도구 호출 모두 거부.
- 운영자 마이그레이션 적용 직전 **`@최철균` 승인 필수** (Phase 4 개발계획서 §8 P1 게이트).
- 마이그레이션 적용 직후 운영자가 Admin UI 로 기존 ScopeProfile 의 도구 등록을 수행해야 정상 운영 재개. 본 의존을 §10 변경 이력 / 운영 가이드에 강조.

---

## 7. 성공 기준

### 7.1 manifest 표준화 (§2.1.1~§2.1.5)

- [ ] Pre-flight 실측 보고서 제출
- [ ] 등급화 매핑 표 작성 + 사용자 승인
- [ ] `TOOL_SCHEMAS` 모든 항목에 `risk_tier`/`maturity`/`status` 부착
- [ ] `tools/list` 디스패처가 `not_exposed`/`forbidden` 필터링
- [ ] `tools/call` 디스패처가 동일 필터로 거부
- [ ] L4 차단 회귀 테스트 녹색 (3 시나리오 이상)
- [ ] manifest 추출 스크립트 + 첫 JSON 산출
- [ ] 기존 4개 도구의 `tools/list`·`tools/call` 동작 회귀 녹색

### 7.2 ScopeProfile tool-level ACL (§2.1.6 — S3 P3 정적 리뷰 P1 흡수)

- [ ] Alembic revision `s3_p4_scope_profile_allow_tools` 작성 + dry-run 검증 + 운영자 승인 + 적용
- [ ] `ScopeProfile` 모델 / repository / AgentPrincipal `can_read` 헬퍼 신설
- [ ] 모든 `tool_*` 함수에 `_check_tool_allowed` 진입점 게이트 추가 (5 도구: search_documents / fetch_node / verify_citation / vectorization_status / read_annotations)
- [ ] `tools/list` per-actor 필터 적용 (manifest `not_exposed` 필터와 AND 결합)
- [ ] `AdminScopeProfilesPage.tsx` `allowed_tools` 토글 UI + PATCH body 확장 + UI 리뷰 1회 통과
- [ ] `migrate_scope_profile_allowed_tools.py` 시드 스크립트 + 운영자 옵션 결정 (A: default-deny / B: L0 bootstrap)
- [ ] `test_mcp_tool_acl.py` ≥ 6 시나리오 녹색 (§2.1.6.f)
- [ ] FG 4-0 검수보고서 §R2 (ACL 단일 결정점) 항목에 본 게이트 명시 + S3 P3 정적 리뷰 P1 종결 기록
- [ ] `종결회고.md §4.1 #11` / `FG3-3 §5 #9` 잔존부채 항목을 종결로 마크 + 본 task 링크 첨부

### 7.3 공통

- [ ] pytest 신규 ≥ 14 (manifest 8 + ACL 6) / 전체 베이스라인 유지
- [ ] 검수·보안 보고서 제출

---

## 8. 리스크

| 리스크 | 대응 |
|-------|-----|
| `TOOL_SCHEMAS` 변경이 외부 클라이언트의 schema validation을 깸 | 추가형 변경 (기존 키 유지). 외부 호환성 회귀 테스트 |
| 디스패처 필터 우회(직접 함수 호출) | tool dispatch는 `mcp_router.py`의 단일 진입점. unit test로 진입점 우회 케이스 차단 |
| 등급 매핑 오판 (L1을 L0로 놓음 등) | Pre-flight 보고서에 근거 메모 + 사용자 승인 단계 |
| 추출 스크립트가 staging/prod에서 다른 결과 | 추출은 코드 상수 기준 → 환경 무관. 환경별 차이가 발견되면 즉시 정책 위반으로 처리 |
| **`allowed_tools` default-deny 적용 후 운영자 등록 전까지 모든 에이전트 호출 거부** | 마이그레이션 적용 시점을 운영자 등록 작업과 같은 운영 창에 배치. 운영 가이드에 절차 명시 (마이그 적용 → Admin UI 등록 → 검증). 옵션 B (L0 bootstrap) 사용 시 비즈니스 영향 최소화 가능하나 보안 트레이드오프 명시 |
| `can_read` 헬퍼가 actor_type 분기를 잘못 처리 → 사람 호출 차단 | `can_read` 단위 테스트에 actor_type 매트릭스 (agent / user / system × scope_profile 유무) ≥ 4 시나리오 의무 |
| `tools/list` per-actor 필터가 캐시되어 ScopeProfile 변경 후에도 stale 응답 | per-actor 필터는 응답 시점 평가 (캐시 금지). FG 3-2 의 `scope_profile_policy` 캐시와 별 — 본 데이터는 actor 별 응답 형성에 직접 들어가므로 결과를 캐시하지 않는다. 회귀 테스트에 ScopeProfile 변경 → 즉시 반영 시나리오 추가 |

---

## 9. 참조

- `Phase 4 개발계획서.md` §1.2 (R1~R5), §2.1 (FG 4-0)
- `uploads/Mimir API : MCP 운영 논의 요약` §3 (도구 등급화 / manifest)
- `CONSTITUTION.md` 제5·20·23·45조
- `backend/app/schemas/mcp.py`, `backend/app/api/v1/mcp_router.py`, `backend/app/mcp/tools.py`

---

*작성: 2026-04-27 | FG 4-0 — Phase 4 진입 게이트*
