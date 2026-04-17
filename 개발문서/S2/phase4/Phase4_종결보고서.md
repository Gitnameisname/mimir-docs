# Phase 4 종결 보고서 — Agent-Facing Interface

**Phase**: Phase 4 (S2)  
**최초 완료일**: 2026-04-17  
**최종 종결일**: 2026-04-17  
**담당**: Claude Sonnet 4.6

---

## 1. 요약

Phase 4 "Agent-Facing Interface"가 최종 종결되었습니다.  
Mimir가 MCP 2025-11-25 스펙을 구현하는 MCP Server로 동작하며, AI 에이전트가 표준 프로토콜로 Mimir를 도구로 호출할 수 있습니다.

초기 구현 이후 보안 권고 3건(REC-4.1~4.3) 추가 이행, 전체 검수 완료, 단위 테스트 47/47 통과를 확인한 후 종결 처리합니다.

---

## 2. 완료 항목

### PH3-CARRY 이월 처리

| ID | 항목 | 결과 |
|----|------|------|
| PH3-CARRY-001 | PII 배치 잡 (`expires_at` 기반 자동 삭제) | **기존 구현 확인** — `expiration_batch.py` + `scheduler.py` 이미 완료 |
| PH3-CARRY-002 | 대화 전문 FTS | **Phase 6으로 재이월** — 이번 Phase는 LIKE 검색 유지 |

### FG4.1 — MCP Server 노출

- [x] MCP `initialize` 핸드셰이크 (`POST /api/v1/mcp/initialize`)
- [x] 도구 3종 — `search_documents`, `fetch_node`, `verify_citation`
- [x] Streamable HTTP / SSE (`POST /api/v1/mcp/tools/call/stream`)
- [x] Resource URI (`mimir://documents/.../versions/.../nodes/...`)
- [x] Prompt Registry MCP 노출 (`GET /api/v1/mcp/prompts`)
- [x] `.well-known/mimir-mcp` 메타데이터 엔드포인트
- [x] Curated Tool Set (쓰기 도구 미노출)
- [x] Rate Limit 전체 적용 (initialize 30/min, tools/call 20/min, SSE 10/min, 조회 60/min)

### FG4.2 — Agent Principal + Delegation + Scope Profile

- [x] `ActorType.AGENT` 추가 — 역할 모델 통합
- [x] `ActorContext` 확장 — `agent_id`, `scope_profile_id`, `acting_on_behalf_of`
- [x] `ScopeProfile` / `ScopeDefinition` 도메인 모델
- [x] `FilterExpression` 파서 + `$ctx` 동적 치환 + SQL 빌더
- [x] `AgentRepository` — CRUD + Kill Switch
- [x] `ScopeProfileRepository` — CRUD
- [x] Scope Profile CRUD API (14개 엔드포인트)
- [x] Kill Switch API (`POST/DELETE /api/v1/admin/agents/{id}/kill-switch`)
- [x] API Key → AGENT ActorContext 변환 (`is_disabled` 즉시 확인)
- [x] DB 마이그레이션 — `scope_profiles`, `scope_definitions`, `agents`, `api_keys` 확장
- [x] Scope Profile update/delete 시 영향 에이전트 목록 감사 로그 기록 (REC-4.2)
- [x] Agent API Key `expires_at` 필수 강제 — NULL이면 인증 거부 (REC-4.3)

### FG4.3 — Structured Response + Tool Schema

- [x] `MCPResponse` 표준 envelope (`success/data/error/metadata`)
- [x] MCP Tool Schema 3종 (inputSchema + 인증요구사항)
- [x] Mimir Extension 선언 (`citation-5tuple`, `span-backref`)
- [x] `MCPErrorCode` enum 8종
- [x] `metadata.trusted` 필드 (Phase 5 확장 준비)

---

## 3. 단위 테스트

```
tests/unit/test_phase4_mcp.py  — 47/47 통과
```

| 테스트 클래스 | 수 | 검증 항목 |
|---|---|---|
| TestFilterExpressionParser | 6 | 파서 정확성, 허용/비허용 op·field |
| TestCtxSubstitution | 5 | $ctx 치환, 화이트리스트 |
| TestSqlBuilder | 4 | eq/in/or SQL 생성 |
| TestResourceUri | 3 | mimir:// URI 파싱 |
| TestMCPInitializeResponse | 4 | initialize 응답 구조, Tool Schema |
| TestMCPResponseEnvelope | 3 | envelope, 에러 코드 enum |
| TestToolInputSchemas | 4 | 도구 입력 스키마 |
| TestAgentActorType | 3 | AGENT ActorContext |
| TestFilterConditionValidation | 3 | op/field 유효성 |
| TestRateLimitConstants | 4 | Rate limit 상수, 엔드포인트 등록 (REC-4.1) |
| TestAffectedAgentsQuery | 3 | 영향 에이전트 조회 (REC-4.2) |
| TestAgentKeyExpiration | 5 | expires_at 강제, kill-switch (REC-4.3) |

---

## 4. 완료 기준 체크

| # | 기준 | 상태 |
|---|------|------|
| 1 | MCP Server 구현 완료 (MCP 2025-11-25 호환) | ✅ |
| 2 | 도구 3종 정상 작동 | ✅ |
| 3 | `.well-known/mimir-mcp` 정상 노출 | ✅ |
| 4 | Agent Principal Type 역할 모델 등재 | ✅ |
| 5 | Scope Profile CRUD API 정상 작동 | ✅ |
| 6 | 동적 변수 치환 ($ctx.*) | ✅ |
| 7 | Kill Switch API (즉시 차단) | ✅ |
| 8 | 응답 envelope 일관성 | ✅ |
| 9 | MCP Tool Schema 3종 | ✅ |
| 10 | Mimir Extension 선언 | ✅ |
| 11 | FG 검수보고서 + 보안취약점검사보고서 | ✅ |
| 12 | 단위 테스트 통과 | ✅ (47/47) |
| 13 | 전체 검수보고서 | ✅ |
| 14 | Phase 4 종결 보고서 | ✅ |

> **e2e 테스트**: DB 없이 단위 테스트 수준으로 검증 완료. 실제 챗봇 e2e는 Docker 환경에서 통합 테스트 권장.

---

## 5. 보안 권고 최종 이행 현황

| ID | 항목 | 상태 |
|----|------|------|
| SEC-4.1 | $ctx 변수 인젝션 방어 | ✅ 수정 완료 |
| SEC-4.2 | FilterExpression 필드/연산자 화이트리스트 | ✅ 수정 완료 |
| SEC-4.3 | Kill Switch 즉시성 (캐시 없는 직접 DB 조회) | ✅ 수정 완료 |
| SEC-4.4 | API Key principal_type 변조 방지 | ✅ 수정 완료 |
| SEC-4.5 | MCP 도구 응답 내부 정보 최소화 | ✅ 수정 완료 |
| SEC-4.6 | Scope Profile 삭제 시 ON DELETE SET NULL | ✅ 설계 완화 |
| SEC-4.7 | MCP 리소스 URI 인젝션 방어 | ✅ 수정 완료 |
| REC-4.1 | MCP 엔드포인트 Rate Limit | ✅ 이행 완료 |
| REC-4.2 | Scope Profile 변경 영향 에이전트 감사 기록 | ✅ 이행 완료 |
| REC-4.3 | Agent API Key expires_at 필수 강제 | ✅ 이행 완료 |

---

## 6. 신규 파일 목록

### Backend

```
backend/app/mcp/__init__.py
backend/app/mcp/errors.py
backend/app/mcp/tools.py
backend/app/mcp/resources.py
backend/app/mcp/prompts.py
backend/app/mcp/scope_filter.py
backend/app/models/agent.py
backend/app/models/scope_profile.py
backend/app/repositories/agent_repository.py
backend/app/repositories/scope_profile_repository.py
backend/app/schemas/agent.py
backend/app/schemas/mcp.py
backend/app/services/filter_expression.py
backend/app/api/v1/mcp_router.py
backend/app/api/v1/scope_profiles.py
```

### 수정 파일

```
backend/app/api/auth/models.py       — ActorType.AGENT, ActorContext 확장
backend/app/api/auth/dependencies.py — AGENT ActorContext 변환 + REC-4.3 expires_at 강제
backend/app/api/v1/router.py         — MCP + Scope Profile 라우터 등록
backend/app/api/v1/mcp_router.py     — REC-4.1 Rate Limit 7개 엔드포인트 적용
backend/app/api/v1/scope_profiles.py — REC-4.2 영향 에이전트 감사 로그
backend/app/db/connection.py         — Phase 4 DDL + init_db 확장
backend/app/main.py                  — .well-known/mimir-mcp 엔드포인트
```

### 산출 문서

```
docs/개발문서/S2/phase4/산출물/FG4.1_검수보고서.md
docs/개발문서/S2/phase4/산출물/FG4.2_검수보고서.md
docs/개발문서/S2/phase4/산출물/FG4.3_검수보고서.md
docs/개발문서/S2/phase4/Phase4_보안취약점검사보고서.md  — REC-4.1~4.3 이행 현황 갱신
docs/개발문서/S2/phase4/Phase4_전체검수보고서.md       — 신규
docs/개발문서/S2/phase4/Phase4_종결보고서.md
```

### 테스트

```
backend/tests/unit/test_phase4_mcp.py  — 47개 단위 테스트
```

---

## 7. 이월 항목

| ID | 항목 | 이월 대상 | 비고 |
|----|------|-----------|------|
| FG4.1-DEFER-001 | `tasks` capability 활성화 (`propose_transition`) | Phase 5 | MCP `tasks: false → true` |
| FG4.1-DEFER-002 | MCP 클라이언트 e2e 호환성 테스트 (LangChain, n8n) | Phase 5 완료 후 | Docker 통합 테스트 시 수행 |
| REC-4.1-PART | 에이전트별 독립 rate limit (per-agent quota) | Phase 5 FG5.3 §5 | Phase 5 개발계획서에 메모 등록 완료 |
| DOCS-001 | 외부 클라이언트용 MCP 통합 가이드 | Phase 6 FG6.2 / Task 6-5 | Phase 6 개발계획서·task6-5에 추가 완료 |

---

## 8. Phase 5 선행 조건

Phase 4 종결로 Phase 5 "Agent Action Plane" 진행 가능합니다.

- Phase 4의 Agent Principal 모델 위에서 `propose_transition` 도구 구현
- MCP `tasks` capability (`false` → `true`로 전환)
- `delegate:write` scope 구현
