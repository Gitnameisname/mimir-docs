# Pre-flight 실측 — S3 Phase 4 FG 4-0

**작성일**: 2026-04-28
**대상**: 본 Phase 진입 전 MCP 표면 사실 확정 (task4-0 §2.1.1)
**기준 코드**: `backend/app/mcp/`, `backend/app/api/v1/mcp_router.py`, `backend/app/schemas/mcp.py` 현 상태 (head: `s3_p2_tags`, S3 P3 boot DDL 적용 후)
**작성자**: Claude (Design+Implementation Actor — dual-agent §3.3 extended)

---

## 1. MCP 도구 인벤토리 (TOOL_SCHEMAS ↔ tool_* 함수 1:1 대응)

`backend/app/schemas/mcp.py:TOOL_SCHEMAS` 와 `backend/app/mcp/tools.py` 의 함수가 다음과 같이 대응한다 — **누락 없음**:

| # | TOOL_SCHEMAS.name | 함수 (위치) | 의도 | 도입 시점 |
|---|---|---|---|---|
| 1 | `search_documents` | `tool_search_documents` ([tools.py:42](../../../../backend/app/mcp/tools.py#L42)) | FTS/Hybrid 검색 + Scope ACL | S2 Phase 4 |
| 2 | `fetch_node` | `tool_fetch_node` ([tools.py:154](../../../../backend/app/mcp/tools.py#L154)) | 노드 전문 조회 | S2 Phase 4 |
| 3 | `verify_citation` | `tool_verify_citation` ([tools.py:241](../../../../backend/app/mcp/tools.py#L241)) | Citation 5-tuple 검증 (해시) | S2 Phase 4 |
| 4 | `mimir.vectorization.status` | `tool_vectorization_status` ([tools.py:455](../../../../backend/app/mcp/tools.py#L455)) | 문서 벡터화 상태 조회 | S3 Phase 0 FG 0-5 |
| 5 | `read_annotations` | `tool_read_annotations` ([tools.py:550](../../../../backend/app/mcp/tools.py#L550)) | 인라인 주석 조회 | S3 Phase 3 FG 3-3 |

**대응 검증**: `_CURATED_TOOLS = {s["name"] for s in TOOL_SCHEMAS}` ([mcp_router.py:109](../../../../backend/app/api/v1/mcp_router.py#L109)) 가 단일 정본. `_dispatch_tool` ([mcp_router.py:263](../../../../backend/app/api/v1/mcp_router.py#L263)) 가 5 케이스를 분기. 누락·고아 함수 없음.

**확장 선언** (`MIMIR_EXTENSIONS`, [mcp.py:366](../../../../backend/app/schemas/mcp.py#L366)): `mimir.citation-5tuple`, `mimir.span-backref` 2 capability — 단순 메타 (호출 가능 도구 아님).

---

## 2. 리소스 인벤토리 ([resources.py](../../../../backend/app/mcp/resources.py))

처리 가능 URI 패턴은 정규식 1건으로 단일화:

```
mimir://documents/{document_id}/versions/{version_id}/nodes/{node_id}
```

- 정규식: `r"^mimir://documents/([^/]+)/versions/([^/]+)/nodes/([^/]+)$"` ([resources.py:11](../../../../backend/app/mcp/resources.py#L11))
- 노출 엔드포인트: `GET /mcp/resources/read?uri=...` — 내부적으로 `tool_fetch_node` 위임 ([mcp_router.py:432](../../../../backend/app/api/v1/mcp_router.py#L432))
- 리스트 엔드포인트: `GET /mcp/resources?document_id=...&limit=...` — `nodes/versions/documents` 직접 SELECT ([mcp_router.py:356](../../../../backend/app/api/v1/mcp_router.py#L356))

**경고**: 리스트 엔드포인트는 `tool_fetch_node` 의 ACL 필터(`_resolve_acl_filter` + `_ensure_document_allowed`) 를 거치지 **않는다**. 직접 SQL JOIN 으로 nodes/versions/documents 조회 → Scope Profile ACL 우회 가능성 있음. **FG 4-1 (envelope 적용 시) 또는 별도 ACL 보강 권고** — 본 Pre-flight 단계에서는 사실 기록만, 본 FG 범위 외.

---

## 3. 프롬프트 인벤토리 ([prompts.py](../../../../backend/app/mcp/prompts.py))

- 함수: `list_mcp_prompts(conn) -> list[dict]` ([prompts.py:15](../../../../backend/app/mcp/prompts.py#L15))
- 데이터 소스: `prompt_templates` 테이블, `WHERE is_active = TRUE`, `LIMIT 100`
- 노출 엔드포인트: `GET /mcp/prompts` ([mcp_router.py:455](../../../../backend/app/api/v1/mcp_router.py#L455))
- 폐쇄망 안전: 테이블 부재 시 빈 목록 반환 (try/except 로 fail-soft)
- 형식: `{"name": "mimir-{id}", "description": ..., "arguments": [{name, description, required}]}`

**관찰**: `is_active = TRUE` 외 다른 ACL 게이트 없음. prompts 가 organization-scoped 또는 user-scoped 어휘를 포함하면 누구나 조회 가능. 운영 정책상 prompt_templates 의 콘텐츠가 비밀이 아닌 가정 — 이 가정의 명문화는 본 Pre-flight 범위 외.

---

## 4. REST 엔드포인트 인벤토리 (관련 부분)

본 Phase 가 다룰 신규 도구의 도메인 코어가 REST 와 wrapper 가 아님을 확인하기 위한 매핑.

| REST 엔드포인트 (예상) | 도메인 코어 함수 | MCP 신규 도구 매핑 (FG 4-2) | wrapper 관계 |
|---|---|---|---|
| `GET /api/v1/documents/{id}/render` | `app.services.render_service.render_document` | `read_document_render` (신규) | **non-wrapper** — render 결과를 envelope 화 + content_role 부여. 도메인 코어 호출만 공유 |
| `GET /api/v1/documents/{id}/versions/{vid}/render` | 동일 service | `read_document_render` (vid 명시 케이스) | non-wrapper |
| `GET /api/v1/documents/{id}/nodes/{nid}` | `app.repositories.nodes_repository` | `fetch_node` (기존) | **non-wrapper** — fetch_node 는 ACL + version resolve + children 까지 합성. REST 는 raw 노드 1건 |
| `GET /api/v1/citations/{id}/verify` | `app.services.citations_service.verify` | `verify_citation` (기존, FG 4-3 강화) | non-wrapper — REST 는 citation_id 기반, MCP 는 5-tuple 직접 |
| `GET /api/v1/search?q=` (FTS) | `app.services.search_service.SearchService.search_documents` | `search_documents` (기존) | non-wrapper — 결과 형태 다름 (REST 는 사람용, MCP 는 envelope+citation) |
| `POST /api/v1/documents/{id}/resolve-reference` (예상 신규) | (FG 4-2 에서 신설) | `resolve_document_reference` (신규) | non-wrapper — REST 와 동일 도메인 함수 호출만 공유 |

검수보고서 §R5 에서 함수-호출 그래프로 재증명 (FG 4-2 산출물).

---

## 5. 현재 manifest 필드 상태

`TOOL_SCHEMAS` 의 5 도구 모두 다음 필드를 **갖고 있지 않다**:

- ❌ `risk_tier` (L0~L4)
- ❌ `maturity` (stable/beta/experimental/disabled/forbidden)
- ❌ `status` (enabled/disabled/not_exposed)
- ❌ `exposure_policy` (MCP_ENABLED/MCP_DISABLED/REST_ADMIN_ONLY)

가지고 있는 필드: `name`, `description`, `authentication.{method, scope_profile_required, delegation}`, `inputSchema.{type, properties, required}`.

§2.1.3 에서 4 신규 manifest 필드 부착 + §2.1.6 에서 ScopeProfile.allowed_tools 도입 양쪽이 본 FG 범위.

---

## 6. Rate Limit 현황

`mcp_router.py` 상단에 4 한도 정의:

| 상수 | 값 | 적용 엔드포인트 |
|---|---|---|
| `_MCP_TOOL_LIMIT` | `20/minute` | `POST /mcp/tools/call` ([mcp_router.py:228](../../../../backend/app/api/v1/mcp_router.py#L228)) |
| `_MCP_STREAM_LIMIT` | `10/minute` | `POST /mcp/tools/call/stream` ([mcp_router.py:295](../../../../backend/app/api/v1/mcp_router.py#L295)) |
| `_MCP_READ_LIMIT` | `60/minute` | `GET /mcp/resources*`, `GET /mcp/prompts`, `GET /mcp/tools` |
| `_MCP_INIT_LIMIT` | `30/minute` | `POST /mcp/initialize` |

이는 **IP 기반 limiter** (`app.api.rate_limit.limiter` 데코레이터). 별도로 **에이전트별 Valkey 기반 한도** ([mcp_router.py:78](../../../../backend/app/api/v1/mcp_router.py#L78) `_AGENT_RATE_LIMITS`) 가 `_check_agent_rate_limit` 로 적용. Valkey 실패 시 fail-open (S2 ⑦ 폐쇄망 degrade).

본 FG 범위 외이지만, `종결회고 §4.1 #3` 의 per-user 30 req/min annotations rate-limit 후속과 같은 구조 (별 라운드).

---

## 7. prompt_injection_detector 사용 위치

`app.security.prompt_injection` 의 `prompt_injection_detector` 와 `content_directive_separator` 이 두 곳에서 사용:

1. **`mcp_tool_call`** ([mcp_router.py:251](../../../../backend/app/api/v1/mcp_router.py#L251)): `tool_name in ("search_documents", "fetch_node")` 케이스에 `_run_injection_detection(raw)` + 결과 `results[]` 에 `content_directive_separator.annotate_results` 적용. envelope `metadata.injection_risk` / `metadata.injection_patterns_detected` 로 노출.
2. **`mcp_tool_call_stream`** ([mcp_router.py:322](../../../../backend/app/api/v1/mcp_router.py#L322)): 동일 로직, SSE 스트림 청크에 annotate_result 적용.

**관찰**:
- `verify_citation` 도 `content_snapshot` 에 본문 일부를 반환하나 인젝션 탐지 적용 **안 됨**. FG 4-1 envelope 적용 시 함께 검토.
- `read_annotations` 도 본문 텍스트(주석 content) 를 반환하나 인젝션 탐지 적용 **안 됨**. 동일.
- `mimir.vectorization.status` 는 본문 텍스트 미반환 — 적용 불필요.

FG 4-1 (envelope 표준화) 에서 detector 적용 정책 통일 필요. 본 Pre-flight 단계는 사실 기록만.

---

## 8. scope_filter 적용 경로

`apply_scope_filter` ([scope_filter.py:26](../../../../backend/app/mcp/scope_filter.py#L26)) 호출자:

| 호출 위치 | 함수 | 적용 시점 |
|---|---|---|
| [tools.py:345](../../../../backend/app/mcp/tools.py#L345) | `_resolve_acl_filter` (헬퍼) | tool 진입점에서 `actor.is_agent and actor.scope_profile_id` 일 때만 |

`_resolve_acl_filter` 가 호출되는 tool_* 함수:

| 도구 | ACL 패턴 | 호출 위치 |
|---|---|---|
| `tool_search_documents` | `_resolve_acl_filter` + `_fetch_allowed_doc_ids` 후처리 (scope_filter 경로) | [tools.py:68](../../../../backend/app/mcp/tools.py#L68) |
| `tool_fetch_node` | `_resolve_acl_filter` + `_ensure_document_allowed` + `_fetch_accessible_chunk` (scope_filter 경로) | [tools.py:161](../../../../backend/app/mcp/tools.py#L161) |
| `tool_verify_citation` | 동일 (scope_filter 경로) | [tools.py:248](../../../../backend/app/mcp/tools.py#L248) |
| `tool_vectorization_status` | `authorization_service.authorize(document.read, ...)` (직접 authorize 경로, F05-02 P1 수정 후) | [tools.py:484-499](../../../../backend/app/mcp/tools.py#L484-L499) |
| `tool_read_annotations` | `authorization_service.authorize(document.read, ...)` + `annotations_service.list_for_document(actor=...)` (직접 authorize 경로) | [tools.py:582-606](../../../../backend/app/mcp/tools.py#L582-L606) |

**관찰**:
- 두 패턴 공존 — S2 Phase 4 의 search/fetch/verify 는 `scope_filter` 경유, S3 의 vectorization_status (FG 0-5) / read_annotations (FG 3-3) 는 `authorization_service.authorize` 직접 경유. 양쪽 모두 ScopeProfile ACL 을 거쳐 결과는 동일하나 **아키텍처 일관성 부재** — 본 FG 범위 외 (별 ADR 권고).
- `read_annotations` 은 `documents_service.get_document(actor=...)` 가 동일 ScopeProfile ACL 을 적용 — 의미적으로 동등 (FG 3-3 종결보고서 §3 게이트 46 ⚠️ 정정 후의 사실).
- 그러나 task3-3 의 **tool-level allow 게이트** (`ScopeProfile.allowed_tools`) 는 어떤 도구에도 미적용. 본 FG §2.1.6 에서 도입 — 5 도구 일괄 진입점 게이트로 두 패턴을 모두 감싼다.

---

## 9. ActorContext / ScopeProfile 현 상태

### 9.1 ActorContext ([auth/models.py](../../../../backend/app/api/auth/models.py))

```
@dataclass
class ActorContext:
    actor_type: ActorType        # anonymous / user / agent / service
    actor_id: Optional[str]
    is_authenticated: bool
    auth_method: Optional[AuthMethod]
    tenant_id: Optional[str]
    role: Optional[str] = None
    agent_id: Optional[str] = field(default=None)
    scope_profile_id: Optional[str] = field(default=None)  # 에이전트 + 사용자 모두 사용
    acting_on_behalf_of: Optional[str] = field(default=None)
```

- `is_agent` property 있음 (`actor_type == AGENT`)
- `can_read(tool_name)` 메서드 **없음** — §2.1.6.b 에서 신설
- ScopeProfile 직접 조회 로직 없음 — `scope_profile_id` 만 가지고 있음. 헬퍼 추가 시 lazy load 필요

### 9.2 ScopeProfile ([models/scope_profile.py](../../../../backend/app/models/scope_profile.py))

```
@dataclass
class ScopeProfile:
    id, name, description, organization_id, created_at, updated_at,
    scopes: list[ScopeDefinition],
    settings: ScopeProfileSettings,  # FG 3-2 추가
```

- `allowed_tools` 필드 **없음** — §2.1.6.a 에서 DB 컬럼 + dataclass 필드 추가
- `ScopeProfileSettings` 와는 분리해서 추가 권고 — settings 는 운영 옵션, allowed_tools 는 ACL (의미 다름, 쿼리 패턴 다름)

### 9.3 ScopeProfileRepository ([scope_profile_repository.py](../../../../backend/app/repositories/scope_profile_repository.py))

- `_row_to_profile` 가 row → ScopeProfile 매핑 단일 정본. `allowed_tools` 추가 시 본 메서드 + 모든 SELECT 쿼리에 컬럼 포함 필요.
- create/update 메서드의 INSERT/UPDATE SQL 에 `allowed_tools` 추가 필요.
- `_KNOWN_SETTINGS_KEYS` 패턴 (forward-compat) 처럼 `allowed_tools` 도 알려진 도구 화이트리스트로 관리할지 결정 — **권고**: TOOL_SCHEMAS 의 name 만 허용 (manifest 와 정합).

---

## 10. Step 1 결론

**모든 사실 확정 완료**. 본 Pre-flight 가 후속 Step 2~7 의 사실 기반.

특기 사항:
1. `/mcp/resources` 리스트 엔드포인트의 ACL 우회 (§2 끝) — **본 FG 범위 외, 별 라운드 검토 권고**
2. `verify_citation` / `read_annotations` 응답에 인젝션 탐지 미적용 (§7) — **FG 4-1 envelope 적용 시 통일**
3. `read_annotations` 의 `_resolve_acl_filter` 우회는 동등 결과 (§8) — 이슈 아님
4. `read_annotations` 가 `_check_agent_write_blocked` 호출 외에 `_check_tool_allowed` 게이트 미보유 (§8 끝) — **본 FG §2.1.6 에서 일괄 도입**

운영자 검토 후 Step 2 (등급화 매핑 합의) 진행.

---

## 11. 변경 이력

| 일자 | 변경 | 작성자 |
|---|---|---|
| 2026-04-28 | 초판 — task4-0 §2.1.1 사실 확정 | Claude |
