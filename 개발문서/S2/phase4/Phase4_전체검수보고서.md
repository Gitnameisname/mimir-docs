# Phase 4 전체 검수보고서 — Agent-Facing Interface

**Phase**: Phase 4 (S2)  
**검수일**: 2026-04-17  
**검수자**: Claude Sonnet 4.6 (AI 전체 검수)  
**검수 범위**: FG4.1 + FG4.2 + FG4.3 + 보안 권고 개선 (REC-4.1~4.3) + 단위 테스트

---

## 1. 검수 목적

FG별 검수보고서(FG4.1~4.3)는 각 Feature Group의 산출물을 기준으로 개별 검수한 결과다.  
이 보고서는 Phase 4 전체를 대상으로 **개발계획서 §10 완료 기준**, **S2 원칙 준수 여부**, **보안 권고 이행**, **테스트 커버리지**를 종합 평가한다.

---

## 2. 완료 기준 종합 점검 (개발계획서 §10)

| # | 완료 기준 | 상태 | 근거 |
|---|-----------|------|------|
| 1 | MCP Server 구현 완료 (MCP 2025-11-25 호환) | ✅ | `initialize` 핸드셰이크 + protocol_version 2025-11-25 |
| 2 | 도구 3종 정상 작동 | ✅ | `search_documents`, `fetch_node`, `verify_citation` — tools.py |
| 3 | `.well-known/mimir-mcp` 정상 노출 | ✅ | `main.py:80` `GET /.well-known/mimir-mcp` |
| 4 | Agent Principal Type 역할 모델 등재 | ✅ | `ActorType.AGENT = "agent"` — auth/models.py |
| 5 | Scope Profile CRUD API 정상 작동 | ✅ | 14개 엔드포인트 — scope_profiles.py |
| 6 | 동적 변수 치환 ($ctx.*) 정확하게 작동 | ✅ | `_ALLOWED_CTX_KEYS` 화이트리스트 + 테스트 5개 |
| 7 | 에이전트 킬스위치 API 정상 작동 (즉시 차단) | ✅ | 캐시 없는 직접 DB 조회 — dependencies.py |
| 8 | 응답 envelope 일관성 | ✅ | 전 엔드포인트 `MCPResponse(success/data/error/metadata)` |
| 9 | MCP Tool Schema 3종 완료 | ✅ | `TOOL_SCHEMAS` — schemas/mcp.py |
| 10 | Mimir Extension 선언 (MCP 호환) | ✅ | `mimir.citation-5tuple`, `mimir.span-backref` |
| 11 | FG 검수보고서 + 보안취약점검사보고서 | ✅ | FG4.1/4.2/4.3 + Phase4_보안취약점검사보고서 |
| 12 | 단위 테스트 통과 | ✅ | **47/47** (기존 35 + REC 개선 12) |
| 13 | MCP 공식 문서 산출 | ⚠️ | FG4.1 검수보고서의 엔드포인트 표로 대체. 외부 클라이언트용 단독 통합 가이드 부재 → Phase 6 이월 |
| 14 | Phase 4 종결 보고서 작성 완료 | ✅ | Phase4_종결보고서.md |

> **전체**: 13/14 ✅, 1건 ⚠️ 조건부 통과 (외부용 MCP 통합 가이드 미산출 — Phase 6 이월)

---

## 3. S2 원칙 준수 검증

| 원칙 | 요구 사항 | 검수 결과 |
|------|-----------|-----------|
| ⑤ 접근 범위 하드코딩 금지 | 코드에 `if scope == "team"` 등 scope 문자열 없어야 함 | ✅ — Scope Profile + FilterExpression으로 동적 처리. 코드 내 scope 분기 없음 |
| ⑥ AI 에이전트 = 1급 API 소비자 | 감사 로그에 `actor_type` 필드 필수 | ✅ — `audit_actor_type = "agent"` 감사 로그 전체 적용 |
| ⑦ 폐쇄망 전체 기능 동작 | 외부 의존 off 시 degrade하지만 실패 않음 | ✅ — Prompt Registry 폴백 `[]` 반환, 검색 SearchService 실패 시 MCPError 반환 |
| S1 ① 문서 타입 하드코딩 금지 | Phase 4 신규 코드에도 동일 | ✅ — MCP 도구는 document_type 파라미터로만 수용, 내부 분기 없음 |

---

## 4. 보안 권고 이행 현황

Phase4_보안취약점검사보고서의 잔여 권고 3건, 전부 이행 완료.

| ID | 권고 사항 | 상태 | 구현 내용 |
|----|-----------|------|-----------|
| REC-4.1 | MCP 엔드포인트 속도 제한 | ✅ **이행 완료** | 7개 엔드포인트 전체에 slowapi 적용: `initialize` 30/min, `tools/call` 20/min, `tools/call/stream` 10/min, 나머지 60/min |
| REC-4.2 | Scope Profile 변경 시 영향 에이전트 알림 | ✅ **이행 완료** | `update_scope_profile` / `delete_scope_profile` — `affected_agents` 목록을 감사 로그 metadata에 기록 |
| REC-4.3 | Agent API Key 만료 TTL 강제 | ✅ **이행 완료** | `_extract_agent_context()` — `expires_at IS NULL`이면 즉시 ANONYMOUS 반환 + 경고 로그 |

> **참고**: REC-4.1의 에이전트별 독립 rate limit (per-agent quota)은 Phase 5 이월. Phase 5 개발계획서 FG5.3 §5에 메모 추가 완료.

---

## 5. 엔드포인트 전수 점검

### MCP 엔드포인트 (7개)

| 메서드 | 경로 | Rate Limit | 인증 | 상태 |
|--------|------|-----------|------|------|
| `POST` | `/api/v1/mcp/initialize` | 30/min | Bearer/API Key | ✅ |
| `POST` | `/api/v1/mcp/tools/call` | 20/min | Bearer/API Key | ✅ |
| `POST` | `/api/v1/mcp/tools/call/stream` | 10/min | Bearer/API Key | ✅ |
| `GET` | `/api/v1/mcp/resources` | 60/min | Bearer/API Key | ✅ |
| `GET` | `/api/v1/mcp/resources/read` | 60/min | Bearer/API Key | ✅ |
| `GET` | `/api/v1/mcp/prompts` | 60/min | Bearer/API Key | ✅ |
| `GET` | `/api/v1/mcp/tools` | 60/min | Bearer/API Key | ✅ |
| `GET` | `/.well-known/mimir-mcp` | — | 공개 | ✅ |

### Admin 엔드포인트 (14개)

| 메서드 | 경로 | 역할 요구 |
|--------|------|----------|
| `POST/GET/GET/PUT/DELETE` | `/api/v1/admin/scope-profiles` | ORG_ADMIN 이상 |
| `POST/DELETE` | `/api/v1/admin/scope-profiles/{id}/scopes/{name}` | ORG_ADMIN 이상 |
| `POST/GET/GET/PUT/DELETE` | `/api/v1/admin/agents` | ORG_ADMIN 이상 |
| `POST/DELETE` | `/api/v1/admin/agents/{id}/kill-switch` | ORG_ADMIN 이상 |

---

## 6. 단위 테스트 커버리지 요약

**총 47/47 통과** (0 실패, 0 오류)

| 테스트 클래스 | 테스트 수 | 검증 항목 |
|---|---|---|
| `TestFilterExpressionParser` | 6 | 파서 정확성, 허용/비허용 op·field |
| `TestCtxSubstitution` | 5 | $ctx 치환, 화이트리스트, 리스트 치환 |
| `TestSqlBuilder` | 4 | eq/in/or SQL 생성 |
| `TestResourceUri` | 3 | mimir:// URI 파싱, 라운드트립 |
| `TestMCPInitializeResponse` | 4 | initialize 응답 구조, Tool Schema 완전성 |
| `TestMCPResponseEnvelope` | 3 | success/error envelope, 에러 코드 enum |
| `TestToolInputSchemas` | 4 | 도구 입력 스키마 유효성 |
| `TestAgentActorType` | 3 | AGENT ActorContext, is_agent, audit_actor_type |
| `TestFilterConditionValidation` | 3 | op/field 유효성 검증 |
| `TestRateLimitConstants` | 4 | Rate limit 상수, 엔드포인트 7개 등록, 엄격도 순서 |
| `TestAffectedAgentsQuery` | 3 | 영향 에이전트 목록 조회, UUID→str 변환 |
| `TestAgentKeyExpiration` | 5 | expires_at 필수 강제, kill-switch 검증, scope_profile 우선순위 |

---

## 7. 미결 이월 사항

| ID | 항목 | 이월 대상 | 비고 |
|----|------|-----------|------|
| FG4.1-DEFER-001 | `tasks` capability 활성화 (`propose_transition`) | Phase 5 | Phase 5 완료 후 `tasks: false → true` |
| FG4.1-DEFER-002 | MCP 클라이언트 e2e 호환성 테스트 (LangChain, n8n) | Phase 5 완료 후 | Docker 환경 통합 테스트 시 수행 |
| REC-4.1-PART | 에이전트별 독립 rate limit (per-agent quota) | Phase 5 FG5.3 | Phase 5 개발계획서 §FG5.3 §5에 메모 등록 완료 |
| DOCS-001 | 외부 클라이언트용 MCP 통합 가이드 (단독 문서) | Phase 6 | FG4.1 검수보고서 엔드포인트 표로 현행 대체 |

---

## 8. 종합 판정

| 항목 | 결과 |
|------|------|
| 완료 기준 (14항) | 13 ✅, 1 ⚠️ |
| S2 원칙 (4항) | 4 ✅ |
| 보안 권고 이행 (3건) | 3 ✅ |
| 단위 테스트 | 47/47 ✅ |
| 이월 항목 | 4건 (Phase 5~6 예정) |

**Phase 4는 핵심 완료 기준을 충족하며 Phase 5 진행 가능 상태입니다.**  
⚠️ 항목(외부 MCP 통합 가이드)은 Phase 6 Admin UI 구축 시 함께 작성 예정.
