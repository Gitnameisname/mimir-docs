# MCP 표면 개요

**Mimir Control Protocol** — 외부 AI 에이전트 (Chatbot 등) 가 Mimir 문서를 안전하게 검색·인용·검증·제안할 수 있도록 표준화된 인터페이스.

**버전**: v2.0.0-s3-phase4 (2026-04-28)
**프로토콜**: MCP 2025-11-25 호환 + Mimir Extension

---

## 1. 설계 철학

### 1.1 5 절대 규칙 (R1~R5)

| R | 규칙 | 보장 방식 |
|---|---|---|
| **R1** | L4 도구 MCP 절대 금지 | TOOL_SCHEMAS 에 L4 등재 0 + `is_tool_mcp_exposed` 차단 + contract drift 게이트 |
| **R2** | ACL 단일 결정점 (Scope Profile) | 모든 도구 진입점에 `_check_tool_allowed` + `_ensure_document_allowed` 게이트 |
| **R3** | pinned citation 강제 | URI 빌더가 `latest` 거부 + `verify_citation` 이 `latest` 입력 즉시 거부 |
| **R4** | `instruction_authority=none` | envelope 의 Literal 단일값 강제 (read + write) — Pydantic 422 |
| **R5** | REST/MCP 1:1 복제 금지 | 도메인 코어 (services/repositories) 공유 — wrapper 가 아닌 envelope/ACL/audit 통합 |

추가 정책:
- **R6** — `detected_risks` 는 **차단이 아닌 경고** (응답 본문 보존, 클라이언트가 결정)

### 1.2 도구 등급화 (L0~L4)

| 등급 | 정의 | 예시 | MCP 노출 |
|---|---|---|---|
| **L0** | 순수 읽기, 부작용 없음, 가역 | search / fetch / status | ✅ 가능 |
| **L1** | 검증·해시 등 결정성 검사 | verify_citation, resolve_reference | ✅ 가능 |
| **L2** | 단일 사용자 의도의 가역 쓰기 | save_draft (FG 4-6) | 조건부 (4 사전 조건 충족) |
| **L3** | 워크플로 전이, 사람 승인 필요 | publish, approve | ❌ MCP 영구 금지 |
| **L4** | 비가역·파괴적·데이터 영향 | delete, reindex, change_schema | ❌ R1 영구 금지 |

---

## 2. 9 도구 매니페스트 (2026-04-28 기준)

| # | 도구 | risk_tier | maturity | default_enabled | policy_profile |
|---|---|---|---|---|---|
| 1 | `search_documents` | L0 | stable | ✅ | read_safe |
| 2 | `fetch_node` | L0 | stable | ✅ | read_safe |
| 3 | `verify_citation` | L1 | stable | ✅ | read_safe |
| 4 | `read_annotations` | L0 | beta | ✅ | read_safe |
| 5 | `search_nodes` | L0 | beta | ✅ | read_safe |
| 6 | `read_document_render` | L0 | beta | ✅ | read_safe |
| 7 | `mimir.vectorization.status` | L0 | beta | ❌ | read_safe |
| 8 | `resolve_document_reference` | L1 | experimental | ❌ | experimental |
| 9 | `save_draft` (write) | **L2** | experimental | ❌ | **write_audited** |

각 도구의 상세 입출력은 [도구별 명세.md](도구별%20명세.md) 참조.

---

## 3. 표면 구조

### 3.1 HTTP 엔드포인트

| 메서드 | 경로 | 의도 | Rate Limit |
|---|---|---|---|
| POST | `/api/v1/mcp/initialize` | 핸드셰이크 (capability 협상) | 30/min |
| POST | `/api/v1/mcp/tools/call?tool_name=<name>` | 도구 호출 (JSON 응답) | 20/min |
| POST | `/api/v1/mcp/tools/call/stream?tool_name=<name>` | 도구 호출 (SSE 스트림) | 10/min |
| GET  | `/api/v1/mcp/tools` | 노출 도구 목록 (외부 view) | 60/min |
| GET  | `/api/v1/mcp/resources` | mimir:// 리소스 목록 | 60/min |
| GET  | `/api/v1/mcp/resources/read?uri=<mimir://...>` | 리소스 본문 조회 (fetch_node 위임) | 60/min |
| GET  | `/api/v1/mcp/prompts` | Prompt Registry 목록 | 60/min |
| GET  | `/api/v1/admin/mcp/manifest` | **운영자 전용** 전체 manifest | (admin) |

추가로 에이전트별 Valkey 기반 rate-limit (`_AGENT_RATE_LIMITS`) 가 IP rate-limit 와 별 적용.

### 3.2 표준 응답 envelope (FG 4-1 / FG 4-6)

모든 응답은 다음 구조:

```json
{
  "success": true,
  "data": { ... 도구별 결과 ... },
  "metadata": {
    "request_id": "...", "timestamp": "...", "agent_id": "...",
    "execution_time_ms": 123,
    "trusted": false,
    "injection_risk": false,
    "injection_patterns_detected": [],
    "source_data_untrusted": true
  },
  "envelope": { ... read 도구 envelope (있을 때) ... },
  "write_envelope": { ... write 도구 envelope (있을 때) ... }
}
```

**read envelope** (`MCPReadEnvelope`):
- `content_role`: `retrieved_evidence` / `tool_metadata` / `system_status`
- `instruction_authority`: `"none"` (Literal 강제 — R4)
- `trust_level`: `source_document` / `agent_generated` / `synthetic` / `unknown`
- `detected_risks`: prompt injection / url obfuscation / secret leak 등 — 차단이 아닌 경고 (R6)
- `source`: mimir:// URI 컨테이너
- `items_total` / `items_truncated` (검색류)

**write envelope** (`MCPWriteEnvelope`, FG 4-6):
- `content_role`: `"mutation_proposed"` (Literal)
- `instruction_authority`: `"none"` (R4)
- `impact`: DraftImpactPreview (chars/nodes 차이)
- `proposal_id`: 후속 approve/reject 추적용
- `requires_human_approval`: **항상 True** (자동 머지 0)
- `audit_chain`: emit 된 event_type 목록

---

## 4. mimir:// URI 4 패턴 (FG 4-1)

| 패턴 | URI | 사용 |
|---|---|---|
| 문서 | `mimir://documents/{document_id}` | resolve / 일반 참조 |
| 버전 | `mimir://documents/{document_id}/versions/{version_id}` | 버전 단위 참조 |
| 노드 | `mimir://documents/{document_id}/versions/{version_id}/nodes/{node_id}` | citation source |
| 렌더 | `mimir://documents/{document_id}/versions/{version_id}/render` | rendered_text citation |

R3 강제: URI 어디에도 `latest` 단독 미출현. `version_id` 는 항상 구체 vN. `latest` 입력 시 서버가 즉시 `VersionsRepository.get_current_published` 로 resolve.

빌더: `app.mcp.uri_builder.{build_doc_uri / build_version_uri / build_node_uri / build_render_uri / parse_uri / resolve_latest_version}`.

---

## 5. ScopeProfile tool-level ACL (FG 4-0 §2.1.6)

`scope_profiles.allowed_tools` (JSONB list[str]) — 에이전트가 호출 가능한 MCP 도구 화이트리스트.

- **default-deny**: 빈 리스트 (운영자 명시 등록 전까지 모든 도구 거부)
- **per-actor 필터**: `tools/list` 응답이 actor 의 `allowed_tools` 로 좁혀짐
- **검증**: `_allowed_tools_validate` 가 `known_tool_names()` 외 거부 (manifest 정합 강제)
- **Admin UI**: `/admin/scope-profiles/{id}` 의 토글 그룹 — 동적 manifest fetch 로 도구 표시
- **Audit**: `scope_profile.allowed_tools.changed` event_type — 변경 시 before/after 기록

---

## 6. 인증 / 인가 흐름

```
[Agent / Chatbot] --POST/api/v1/mcp/initialize--> [Mimir]
                                                    |
                                          resolve_current_actor()
                                                    |
                                                    v
                                         actor.is_authenticated?
                                                    |
                                          Y (agent / user / system)
                                                    |
                                                    v
                                    POST /tools/call?tool_name=X
                                                    |
                                                    v
                                          _check_agent_write_blocked   (kill switch)
                                                    |
                                                    v
                                          _check_tool_allowed("X")     (R2 — ScopeProfile.allowed_tools)
                                                    |
                                                    v
                                          _ensure_document_allowed     (document.read 권한)
                                                    |
                                                    v
                                          dispatch → tool_X(request, actor, conn)
                                                    |
                                                    v
                                          _build_envelope (or write)   (R4 — instruction_authority=none)
                                                    |
                                                    v
                                          _emit_audit                  (event_type=mcp.tool.X.<...>)
                                                    |
                                                    v
                                          MCPResponse { data, envelope, metadata }
```

Kill switch (`_check_agent_write_blocked`) 는 현재 read 도구에서 no-op — 인증 단계에서 비활성 agent 가 이미 차단됨. Write 도구 (save_draft) 가 추가되면서 본 게이트가 의미를 가질 예정.

---

## 7. 폐쇄망 동등성 (S2 ⑦)

- 모든 룰 기반 탐지 (prompt_injection_detector) 가 외부 LLM 의존 없음
- `resolve_document_reference` 의 semantic 단계 → `MIMIR_OFFLINE=1` 에서 FTS fallback 자동 전환
- `read_document_render` 의 render 가 외부 의존 0 (render_service 위임)
- Valkey 장애 시 fail-soft (rate limit / 캐시) — 보안 게이트는 fail-closed

---

## 8. 관련 문서

- [도구별 명세.md](도구별%20명세.md) — 9 도구의 입출력 schema + 동작 상세
- [매니페스트 정책.md](매니페스트%20정책.md) — manifest 5 + 5 필드 + 외부/운영자 view 분리 + drift 게이트
- [Tool ACL 모델.md](Tool%20ACL%20모델.md) — ScopeProfile.allowed_tools + 진입점 게이트 + audit
- [Citation 모델.md](Citation%20모델.md) — 5-tuple + citation_basis + 5중 검증
- 관련 API: [API 기술서/MCP API.md](../API%20기술서/MCP%20API.md) (Phase 2 작성 예정)
- 관련 보안: [6. 보안 모델/보안 모델.md](../6.%20보안%20모델/보안%20모델.md) §11 MCP 표면 보안

---

## 9. 변경 이력

| 일자 | 변경 |
|---|---|
| 2026-04-28 | 초판 — S3 Phase 4 1라운드 (FG 4-0 ~ 4-4) + 이월 (FG 4-5 / 4-6) 모두 종결 시점 반영 |
