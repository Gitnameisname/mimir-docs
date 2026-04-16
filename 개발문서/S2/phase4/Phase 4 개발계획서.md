# Phase 4 개발계획서
## Agent-Facing Interface

---

## 1. 단계 개요

### 단계명
Phase 4. Agent-Facing Interface

### 목적
AI 에이전트가 Mimir를 도구로 호출할 수 있는 표준 계약을 제공한다. MCP 2025-11-25 스펙을 기반으로 읽기 도구(검색, 조회, 검증)를 노출하고, 에이전트를 독립 Principal로 모델링한다. 외부 소비자(챗봇)의 scope 접근 범위를 관리자가 설정 가능하도록 한다.

### 선행 조건
- Phase 1 (모델·프롬프트 추상화) 완료
- Phase 2 (Grounded Retrieval v2 + Citation Contract) 완료
- Phase 3 (Conversation 도메인) 완료
- MCP 2025-11-25 스펙 팀 검토 완료
- OAuth 2.0 client-credentials 인증 구조 합의 완료

### 기대 결과
- MCP Server 구현 완료 (MCP 2025-11-25 스펙 호환)
- 읽기 도구 3종 (search_documents, fetch_node, verify_citation) 구현 완료
- Agent Principal 모델 정의 및 역할 시스템에 통합 완료
- Scope Profile CRUD API 구현 완료
- 에이전트 킬스위치 API 구현 완료
- 챗봇이 MCP 또는 REST를 통해 Mimir를 도구로 호출 가능 상태
- MCP 공식 문서 ("Mimir는 MCP 2025-11-25를 구현한다") 산출

---

## 2. Phase 4의 역할

Phase 4는 "Mimir를 에이전트의 도구로 만드는" Phase이다. S1까지 Mimir는 사람이 UI를 통해 접근하는 플랫폼이었다면, S2의 Phase 4부터 AI 에이전트도 동등한 수준의 소비자가 된다. 구체적으로:

1. **MCP 표준 계약**: MCP 스펙을 준수하여 다양한 에이전트 프레임워크(LangChain, AutoGen, n8n 등)가 Mimir를 호출 가능
2. **에이전트 인증 (OAuth 2.0)**: 서버-투-서버 인증, 사용자 대행 검색 지원, 위임 권한 관리
3. **Scope Profile**: 코드에 하드코딩된 접근 범위 제거, 관리자가 동적으로 설정 가능
4. **Principal 모델 확장**: 에이전트를 사용자와 별개의 독립 주체로 모델링, 감사 추적 가능
5. **Kill Switch**: 문제 발생 시 특정 에이전트의 쓰기를 즉시 차단 가능

---

## 3. Feature Group 상세

### FG4.1 — MCP Server 노출

#### 목적
MCP 2025-11-25 스펙을 구현하여 에이전트 클라이언트가 표준 프로토콜로 Mimir와 통신할 수 있도록 한다.

#### 주요 작업 항목

1. **MCP Server 초기화 및 핸드셰이크**
   - MCP `initialize` 요청 처리:
     - 요청: client 메타데이터 (client_id, protocol_version)
     - 응답: server 메타데이터 (server_id="mimir-s2", version, capabilities)
   - 지원 MCP 기능 선언:
     - `tools`: search_documents, fetch_node, verify_citation, propose_transition (Phase 5)
     - `resources`: mimir:// URI scheme으로 문서 노드 노출
     - `prompts`: Prompt Registry 템플릿 노출
     - `tasks`: 에이전트 제안 비동기 플로우

2. **읽기 도구 구현 (Tool interface)**

   **도구 1: search_documents**
   - 목적: 전문 검색 (FTS + Vector)
   - 입력 매개변수:
     ```json
     {
       "query": "string (필수)",
       "scope": "string (선택, 기본값: 'default')",
       "document_types": ["string"] (선택),
       "top_k": "int (기본: 5)",
       "conversation_id": "uuid (선택)"
     }
     ```
   - 출력:
     ```json
     {
       "results": [
         {
           "document_id": "uuid",
           "document_title": "string",
           "version_id": "uuid",
           "node_id": "string",
           "content": "string (요약 또는 전문)",
           "citation": {
             "document_id": "uuid",
             "version_id": "uuid",
             "node_id": "string",
             "span_offset": "int",
             "content_hash": "string"
           },
           "relevance_score": "float",
           "retrieval_time_ms": "int"
         }
       ],
       "total_count": "int",
       "retrieval_method": "fts|vector|hybrid"
     }
     ```

   **도구 2: fetch_node**
   - 목적: 특정 문서 노드의 전문 조회
   - 입력 매개변수:
     ```json
     {
       "document_id": "uuid (필수)",
       "version_id": "uuid (선택, 기본값: latest)",
       "node_id": "string (필수)"
     }
     ```
   - 출력:
     ```json
     {
       "document_id": "uuid",
       "document_title": "string",
       "version_id": "uuid",
       "node_id": "string",
       "content": "string (full text)",
       "metadata": { ... },
       "relationships": { "parent_node_id": "...", "children": [...] }
     }
     ```

   **도구 3: verify_citation**
   - 목적: Citation 5-tuple 검증 (content_hash 확인)
   - 입력 매개변수:
     ```json
     {
       "document_id": "uuid (필수)",
       "version_id": "uuid (필수)",
       "node_id": "string (필수)",
       "content_hash": "string (필수)",
       "span_offset": "int (선택)"
     }
     ```
   - 출력:
     ```json
     {
       "verified": "bool",
       "current_hash": "string",
       "hash_matches": "bool",
       "content_snapshot": "string",
       "version_valid": "bool",
       "message": "string (검증 결과 메시지)"
     }
     ```

3. **Resource 노출 (MCP Resources)**
   - URI scheme: `mimir://documents/{document_id}/versions/{version_id}/nodes/{node_id}`
   - 리소스 메타데이터:
     - mime_type: text/plain
     - description: {document_title} - {node_id}
   - 리소스 조회: URI → fetch_node와 동일한 응답

4. **Prompt Registry 노출 (MCP Prompts)**
   - Phase 1에서 구축한 Prompt Registry의 활성 프롬프트를 MCP로 노출
   - 프롬프트 이름: `mimir-{prompt_id}`
   - 프롬프트 본문: 템플릿 + 변수 목록

5. **Streamable HTTP + SSE 지원**
   - 도구 응답이 크거나 스트리밍 가능한 경우 (예: 긴 문서 fetch):
     - HTTP 200 → chunked transfer encoding
     - 또는 Server-Sent Events (SSE) 사용
   - 클라이언트가 부분 결과부터 처리 가능하도록 설계

6. **`.well-known` 메타데이터 노출**
   - 엔드포인트: `/.well-known/mimir-mcp`
   - 응답:
     ```json
     {
       "mcp_version": "2025-11-25",
       "server_id": "mimir-s2",
       "capabilities": {
         "tools": ["search_documents", "fetch_node", "verify_citation"],
         "resources": true,
         "prompts": true,
         "tasks": false (Phase 5에서 true로 변경)
       },
       "authentication": "oauth2_client_credentials",
       "scope_profile_required": true,
       "documentation_url": "https://mimir.example.com/docs/mcp"
     }
     ```

#### 입력 (선행 의존)
- Phase 2 완료 (Citation 5-tuple 계약)
- Phase 3 완료 (Conversation API)
- MCP 2025-11-25 스펙 분석 완료
- OAuth 2.0 client-credentials 구현 (S1에서 기존)

#### 출력 (산출물)
- MCP Server 구현체 (Python 클래스)
- 도구 3종 구현 (search_documents, fetch_node, verify_citation)
- Resource URI 해석 및 조회 로직
- Prompt Registry MCP 노출 로직
- `.well-known` 메타데이터 엔드포인트
- MCP 핸드셰이크 및 초기화 로직
- 단위 테스트 (MCP 프로토콜 정합성 검증)
- MCP 공식 문서 (Mimir 구현 설명)

#### 검수 기준
- MCP 2025-11-25 스펙 호환성 검증
- 도구 3종 입출력 스키마 정확성
- `.well-known` 응답이 클라이언트 자동 발견 가능
- Resource URI가 유효하고 fetch_node와 동등
- Prompt 노출이 정확한 변수 목록 제공
- Streamable HTTP 또는 SSE가 정상 작동
- 에러 응답이 MCP 스펙 준수 (error code, message)

---

### FG4.2 — Agent Principal + Delegation + Scope Profile

#### 목적
에이전트를 독립 Principal로 모델링하고, 위임 메커니즘과 scope 관리를 구현한다.

#### 주요 작업 항목

1. **Agent Principal Type 역할 모델에 추가**
   - S1 Phase 2의 역할 모델 확장:
     - 기존: `user`, `admin`, `system`
     - 추가: `agent`
   - Agent 특성:
     - 독립 인증 (OAuth client-credentials)
     - 사용자를 대행하여 호출 가능
     - 킬스위치로 차단 가능
     - 에이전트별 감사 추적

2. **Delegation (위임) 모델 정의**
   - Agent가 User를 대행하는 구조:
     ```
     API 요청: Agent A (client-credentials) + access_context {user_id=U1, org_id=O1, permissions=[read]}
     
     Mimir 서버:
       1. Agent A의 OAuth credentials 검증 → 이 agent에게 delegate 권한이 있는가?
       2. access_context.user_id=U1로 DB 조회 → User U1의 실제 ACL 확인
       3. 검색/조회 권한 부여 또는 거부
       4. 감사 로그: actor_type=agent, actor_id=A, acting_on_behalf_of=U1
     ```
   - API Key scope에 `delegate:search`, `delegate:write` 추가
     - `delegate:search`: 사용자를 대행하여 검색 가능
     - `delegate:write`: 사용자를 대행하여 Draft 생성 가능 (Phase 5)

3. **Scope Profile 도메인 모델**
   - Scope Profile: 관리자가 정의하는 ACL 필터 템플릿
   - 구조:
     ```
     ScopeProfile {
       id: uuid,
       name: string (예: "챗봇-기본설정"),
       description: string,
       organization_id: uuid,
       created_at, updated_at: timestamp,
       scopes: ScopeDefinition[] (아래 참조)
     }
     
     ScopeDefinition {
       id: uuid,
       scope_name: string (예: "team", "org", "public"),
       acl_filter: FilterExpression (아래 참조),
       description: string
     }
     
     FilterExpression {
       // S2 수위: 미리 정의된 필드만 지원
       and: [FilterCondition]? 
       or: [FilterCondition]?
     }
     
     FilterCondition {
       field: string (예: "organization_id", "team_id", "visibility", "classification"),
       op: "eq" | "neq" | "in" | "not_in" | "contains",
       value: any | "$ctx.{variable}" (동적 치환)
     }
     ```

4. **Scope Profile 활용 파이프라인**
   - API 요청 시:
     1. OAuth credentials 검증 → 해당 agent에 바인딩된 Scope Profile 조회
     2. 요청에 `scope` 매개변수 포함 (예: `scope="team"`)
     3. Scope Profile에서 `scope_name="team"`에 해당하는 ACL filter 추출
     4. access_context의 변수로 동적 치환 (예: `$ctx.organization_id` → actual org_id)
     5. 해석된 필터를 기존 ACL 엔진에 AND로 합성
     6. 검색/조회 권한 최종 결정

   - 예시 (실제 구동 로직):
     ```python
     # API 요청
     POST /api/v1/rag/search
     Authorization: Bearer {oauth_token}
     Body: {
       "query": "규정 변경사항",
       "scope": "team",
       "access_context": {
         "user_id": "user-123",
         "organization_id": "org-456",
         "team_id": "team-789",
         "permissions": ["read"]
       }
     }
     
     # Mimir 서버 처리
     1. OAuth token 검증 → agent-456 확인
     2. agent-456의 Scope Profile 조회 → "챗봇-기본"
     3. scope_name="team"의 ACL filter 조회:
        {
          "and": [
            {"field": "organization_id", "op": "eq", "value": "$ctx.organization_id"},
            {"field": "team_id", "op": "eq", "value": "$ctx.team_id"}
          ]
        }
     4. $ctx 치환: organization_id="org-456", team_id="team-789"
     5. 최종 필터: organization_id=org-456 AND team_id=team-789
     6. 기존 ACL과 AND: user_id=user-123의 권한 AND 위의 필터
     7. 검색 실행
     ```

5. **Scope Profile CRUD API (Admin UI는 Phase 6 FG6.2에서 구축)**
   - `POST /api/v1/admin/scope-profiles`: 새 Scope Profile 생성
   - `GET /api/v1/admin/scope-profiles`: Scope Profile 목록 조회
   - `GET /api/v1/admin/scope-profiles/{profile_id}`: 상세 조회
   - `PUT /api/v1/admin/scope-profiles/{profile_id}`: 수정
   - `DELETE /api/v1/admin/scope-profiles/{profile_id}`: 삭제
   - 각 엔드포인트는 admin 역할 필수

6. **API Key Scope Profile 바인딩**
   - S1 API Key 모델 확장:
     - 기존: `key`, `secret`, `permissions` (array of strings)
     - 추가: `scope_profile_id`, `agent_principal_type` (optional)
   - API Key 발급 시:
     - Agent용이면: scope_profile_id 필수 지정
     - User용이면: scope_profile_id 무시 (사용자 ACL 그대로 적용)

7. **에이전트 킬스위치 API**
   - `POST /api/v1/admin/agents/{agent_id}/kill-switch`: 특정 에이전트의 쓰기 즉시 차단
   - `DELETE /api/v1/admin/agents/{agent_id}/kill-switch`: 킬스위치 해제
   - 킬스위치 상태 플래그: `agents.is_disabled` (boolean)
   - 검증: 모든 쓰기 요청 시 해당 에이전트의 `is_disabled` 확인 → true이면 거부

#### 입력 (선행 의존)
- Phase 3 완료 (API 구조)
- S1 역할 모델 및 ACL 엔진 (기존)
- 관리자가 정의하고자 하는 Scope Profile 예시 (비즈니스 요구사항)

#### 출력 (산출물)
- Agent Principal Type 역할 모델 확장 (Python)
- Scope Profile 도메인 모델 (SQLAlchemy ORM)
- FilterExpression 파서 및 적용 로직 (Python)
- Scope Profile CRUD API (FastAPI)
- API Key Scope Profile 바인딩 로직
- 에이전트 킬스위치 API
- 동적 변수 치환 (`$ctx.*`) 로직
- DB 마이그레이션 (agents 테이블, scope_profiles 테이블)
- 단위 테스트 (Scope Profile 필터링, 변수 치환)

#### 검수 기준
- Agent Principal Type이 역할 모델에 정상 등재되었는가
- Scope Profile이 관리자 설정으로 동적 생성 가능한가
- FilterExpression 파서가 정확하게 조건을 해석하는가
- $ctx 동적 치환이 정확하게 작동하는가
- API Key가 Scope Profile에 바인딩되고 검증되는가
- 킬스위치 활성화 시 해당 에이전트의 쓰기가 즉시 거부되는가
- 감사 로그에 agent 작업이 정확하게 기록되는가

---

### FG4.3 — Structured Response + Tool Schema

#### 목적
에이전트용 응답을 구조화하고, MCP Tool schema를 정의하며, Mimir 고유 기능(Citation 5-tuple, span 역참조)을 확장으로 선언한다.

#### 주요 작업 항목

1. **에이전트용 구조화 응답 포맷**
   - 모든 에이전트 응답에 표준 envelope:
     ```json
     {
       "success": bool,
       "data": {
         // 도구별 응답 본문
       },
       "error": {
         "code": "string",
         "message": "string"
       },
       "metadata": {
         "request_id": "uuid",
         "timestamp": "ISO8601",
         "agent_id": "uuid",
         "execution_time_ms": "int",
         "trusted": false // Phase 5에서 활용
       }
     }
     ```

2. **MCP Tool Schema 정의**
   - OpenAPI 스펙 → MCP Tool schema로 변환 (수동 큐레이션)
   - 모든 도구에 대해:
     - 도구명, 설명, 입력 매개변수, 출력 스키마
     - 예시:
       ```json
       {
         "name": "search_documents",
         "description": "Search Mimir documents using full-text or vector search",
         "inputSchema": {
           "type": "object",
           "properties": {
             "query": {
               "type": "string",
               "description": "Search query"
             },
             "scope": {
               "type": "string",
               "description": "Access scope (e.g., 'team', 'org')"
             },
             "top_k": {
               "type": "integer",
               "minimum": 1,
               "maximum": 50
             }
           },
           "required": ["query"]
         }
       }
       ```

3. **Mimir Extension 선언 (MCP Extensions)**
   - MCP 스펙의 Extensions 메커니즘을 사용하여 Mimir 고유 기능 선언:
     ```json
     {
       "extensions": [
         {
           "name": "mimir.citation-5tuple",
           "version": "1.0",
           "description": "Citation verification using 5-tuple (document_id, version_id, node_id, span_offset, content_hash)",
           "capabilities": {
             "verify_citation": true,
             "cite_format": "5-tuple"
           }
         },
         {
           "name": "mimir.span-backref",
           "version": "1.0",
           "description": "Span-level back-reference to original content",
           "capabilities": {
             "span_verification": true,
             "position_tracking": true
           }
         }
       ]
     }
     ```

4. **Curated Tool Set**
   - 모든 OpenAPI 엔드포인트를 자동으로 도구화하지 않음 (과도한 노출)
   - Phase 4 수위에서 에이전트에 노출할 도구 목록을 명시적으로 정의:
     - ✅ search_documents (읽기)
     - ✅ fetch_node (읽기)
     - ✅ verify_citation (읽기)
     - ❌ create_document (쓰기, Phase 5에서 제안 전용)
     - ❌ delete_document (민감한 쓰기, 관리자만)
   - 각 도구마다 인증 요구사항 명시:
     - oauth2_client_credentials 필수
     - scope_profile 필수
     - delegation 권한 필요

5. **에러 응답 표준화**
   - 도구 호출 실패 시 MCP 호환 오류 응답:
     ```json
     {
       "success": false,
       "error": {
         "code": "UNAUTHORIZED",
         "message": "User does not have permission to access this document"
       }
     }
     ```
   - 오류 코드 정의 (enum):
     - `UNAUTHORIZED`: 권한 부족
     - `NOT_FOUND`: 리소스 없음
     - `INVALID_SCOPE`: Scope 지정 오류
     - `INVALID_CITATION`: Citation 검증 실패
     - `RATE_LIMIT`: 속도 제한
     - `INTERNAL_ERROR`: 서버 오류

#### 입력 (선행 의존)
- FG4.1 완료 (MCP Server 구현)
- FG4.2 완료 (Scope Profile, Principal 모델)
- Phase 2 Citation Contract 정의
- OpenAPI 스펙 (기존)

#### 출력 (산출물)
- 에이전트용 구조화 응답 포맷 정의 (JSON Schema)
- MCP Tool Schema 정의서 (3종 도구)
- Mimir Extension 선언 문서
- Curated Tool Set 명시 (도구명, 인증요구사항)
- 오류 코드 정의 및 매핑 (Python enum)
- 단위 테스트 (도구 입출력 검증)

#### 검수 기준
- 응답 envelope이 모든 에이전트 엔드포인트에서 일관되게 적용되는가
- MCP Tool Schema가 정확하고 예시가 명확한가
- Extension 선언이 MCP 스펙 준수하는가
- Curated Tool Set이 보안과 기능성 균형을 맞추는가
- 오류 응답이 MCP 호환이고 에이전트 클라이언트가 처리 가능한가

---

## 4. 기술 설계 요약

### 4.1 MCP Server 아키텍스처
```
Agent Client (HTTP POST)
    ↓
MCP Initialize Request
    ↓
Mimir MCP Server
  ├─ OAuth credentials 검증 (client-credentials)
  ├─ Tool dispatch
  │  ├─ search_documents → Retrieval layer
  │  ├─ fetch_node → Document layer
  │  └─ verify_citation → Citation validation
  └─ Response formatting (envelope)
    ↓
MCP Response (JSON)
    ↓
Agent Client (처리)
```

### 4.2 Scope Profile 필터 파이프라인
```
API 요청 (scope="team", access_context={org_id, team_id})
    ↓
Scope Profile 조회 (agent의 바인딩된 profile)
    ↓
scope_name → ACL filter 추출
    ↓
$ctx 동적 치환 (access_context 변수로)
    ↓
FilterExpression 파서 (and/or 조건 해석)
    ↓
기존 ACL 엔진과 AND 합성
    ↓
검색/조회 실행 (최종 필터 적용)
```

### 4.3 Principal 모델 확장
```
Principal
  ├─ User (S1)
  │  ├─ user_id, email, roles[]
  │  └─ ACL: document-level
  ├─ Agent (S2 Phase 4 새추가)
  │  ├─ agent_id, api_key_id, organization_id
  │  ├─ scope_profile_id (ACL 템플릿)
  │  ├─ is_disabled (킬스위치)
  │  └─ acting_on_behalf_of: User (위임 대상)
  └─ System (S1)
     └─ system (배치, 관리 작업)
```

---

## 5. 의존 관계

### 선행 Phase
- **Phase 1** (모델·프롬프트 추상화)
- **Phase 2** (Grounded Retrieval v2 + Citation Contract)
- **Phase 3** (Conversation 도메인)

### 후행 Phase (이 Phase의 산출물을 소비하는 Phase)
- **Phase 5** (Agent Action Plane): 에이전트 제안 API는 Phase 4의 Principal/Scope 모델 위에서 구축
- **Phase 6** (관리자 기능 통합): Scope Profile 관리 UI (FG6.2)
- **Phase 7** (AI 품질 평가): 에이전트 기반 평가 데이터 수집 (에이전트 활동 감시)

---

## 6. 검수 기준 종합

| 항목 | 기준 |
|------|------|
| **FG4.1 - MCP 호환성** | MCP 2025-11-25 스펙을 완전히 구현했는가 / 도구 3종 입출력 스키마 정확 |
| **FG4.1 - Tool 정확성** | search_documents, fetch_node, verify_citation이 정상 작동 |
| **FG4.1 - Streamable HTTP** | 스트리밍 또는 SSE가 정상 작동 |
| **FG4.1 - .well-known** | `/.well-known/mimir-mcp` 메타데이터가 정확하고 클라이언트 발견 가능 |
| **FG4.2 - Agent Principal** | Agent type이 역할 모델에 정상 등재되고 감사 추적 가능 |
| **FG4.2 - Scope Profile** | 관리자가 동적으로 생성/수정 가능하고 필터가 정확히 적용 |
| **FG4.2 - $ctx 치환** | 동적 변수 치환이 정확하고 보안상 문제 없음 |
| **FG4.2 - Kill Switch** | 킬스위치 활성화 시 해당 에이전트의 쓰기가 즉시 거부 |
| **FG4.3 - 응답 포맷** | 모든 에이전트 응답이 표준 envelope 사용 |
| **FG4.3 - Tool Schema** | MCP Tool schema가 정확하고 인증요구사항 명시 |
| **FG4.3 - Extension** | Mimir Extension이 MCP 스펙 준수 |
| **종합** | 챗봇이 MCP를 통해 Mimir를 도구로 호출 가능 |

---

## 7. 예상 산출물 목록

| 산출물 | 형태 | 설명 |
|--------|------|------|
| MCP Server 구현체 | Python 클래스 | MCP 핸드셰이크, 도구 dispatch |
| Tool 구현 (3종) | Python 함수 | search_documents, fetch_node, verify_citation |
| Resource URI 해석 로직 | Python 클래스 | mimir:// URI 파싱 및 조회 |
| `.well-known` 엔드포인트 | FastAPI 라우터 | 메타데이터 노출 |
| Agent Principal 역할 | Python 클래스 | Agent type 정의 |
| Scope Profile 도메인 모델 | SQLAlchemy ORM | ScopeProfile, ScopeDefinition 테이블 |
| FilterExpression 파서 | Python 클래스 | and/or/조건 해석 로직 |
| Scope Profile CRUD API | FastAPI 라우터 | 생성/조회/수정/삭제 |
| 에이전트 킬스위치 API | FastAPI 라우터 | 활성화/해제 |
| API Key Scope Profile 바인딩 | Python 로직 | API Key 발급 시 바인딩 |
| 응답 포맷 정의 | JSON Schema | envelope 구조 |
| Tool Schema 정의서 | JSON | MCP Tool 명세 |
| Mimir Extension 선언 | JSON | Extension 메타데이터 |
| 오류 코드 매핑 | Python enum | 오류 코드 정의 |
| DB 마이그레이션 | Python (alembic) | agents, scope_profiles 테이블 |
| 단위 테스트 | pytest | MCP, Scope Profile, 도구 테스트 |
| MCP 공식 문서 | Markdown | Mimir MCP 2025-11-25 구현 설명 |
| **FG4.1 검수보고서** | Markdown | MCP 호환성, 도구 정확성 검수 |
| **FG4.1 보안취약점검사보고서** | Markdown | Tool 입출력 검증, 인증 보안 |
| **FG4.2 검수보고서** | Markdown | Principal, Scope Profile, 위임 검수 |
| **FG4.2 보안취약점검사보고서** | Markdown | FilterExpression 인젝션 방어, $ctx 보안 |
| **FG4.3 검수보고서** | Markdown | 응답 포맷, Tool Schema, Extension 검수 |
| **FG4.3 보안취약점검사보고서** | Markdown | 응답 데이터 노출 검토, 오류 처리 보안 |
| **Phase 4 종결 보고서** | Markdown | Phase 4 전체 완료 보고 |

---

## 8. 리스크 및 대응

| 리스크 | 영향도 | 대응 방안 |
|--------|--------|----------|
| MCP 스펙 버전 불일치 (2025-11-25 vs 실제 라이브러리) | 중간 | 스펙 버전 명시, 호환성 테스트 강화 |
| Scope Profile FilterExpression의 복잡한 조건으로 오성능 | 중간 | 필터 캐싱, 사전 컴파일 |
| FilterExpression에서 $ctx 변수 누출 (보안) | 높음 | 인젝션 방어 강화, 화이트리스트 검증 |
| 에이전트 킬스위치 지연으로 악성 에이전트가 잠시 계속 동작 | 중간 | 캐시 TTL 짧게 (1분 이하), 실시간 체크 |
| 관리자가 의도치 않게 높은 권한의 Scope Profile 생성 | 중간 | 권한 정책 문서화, 템플릿 제공 |
| MCP 클라이언트 호환성 부족 (LangChain, n8n 등) | 낮음~중간 | 다양한 클라이언트로 테스트, 문서 상세화 |
| Principal 모델 확장으로 인한 기존 코드 변경 | 낮음 | 하위호환성 유지, 마이그레이션 스크립트 |

---

## 9. 선행 조건

Phase 4를 시작하려면:
- Phase 1, 2, 3 완료
- MCP 2025-11-25 스펙 팀 리뷰 완료
- OAuth 2.0 client-credentials 인증 구조 확정
- 관리자가 원하는 Scope Profile 예시 (비즈니스 요구사항) 수집
- MCP 클라이언트 테스트 환경 준비 (챗봇 서버 또는 테스트 클라이언트)

---

## 10. 완료 기준

Phase 4 완료로 판단하는 조건:

1. MCP Server 구현 완료 (MCP 2025-11-25 호환)
2. 도구 3종 (search_documents, fetch_node, verify_citation) 정상 작동
3. `.well-known/mimir-mcp` 메타데이터 정상 노출
4. Agent Principal Type 역할 모델에 정상 등재
5. Scope Profile CRUD API 정상 작동
6. 동적 변수 치환 ($ctx.*) 정확하게 작동
7. 에이전트 킬스위치 API 정상 작동 (즉시 차단)
8. 응답 envelope이 모든 에이전트 엔드포인트에서 일관됨
9. MCP Tool Schema 정의 완료 (3종)
10. Mimir Extension 선언 완료 및 MCP 호환
11. 모든 FG 검수보고서 및 보안취약점검사보고서 승인
12. 챗봇이 MCP를 통해 Mimir를 도구로 호출 가능 (e2e 테스트 통과)
13. MCP 공식 문서 (Mimir 구현) 산출 완료
14. Phase 4 종결 보고서 작성 완료

---

## 11. 예상 투입 규모

- **MCP 서버 구현**: 1.5주 (초기화, 도구 dispatch)
- **도구 3종 구현**: 1주 (search, fetch, verify)
- **Principal/Scope 모델**: 1.5주 (도메인 모델, API, 필터 파서)
- **테스트 및 검수**: 1주 (단위 테스트, MCP 호환성, 보안)
- **문서화 및 마무리**: 0.5주 (MCP 공식 문서, 종결 보고서)
- **총 투입**: 약 5.5주 (병행 가능 부분은 압축 가능)

