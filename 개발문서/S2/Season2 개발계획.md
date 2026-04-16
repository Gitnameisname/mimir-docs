# Mimir Season 2 개발계획
## Agent-Native Knowledge Platform

**작성일**: 2026-04-16
**기반**: S1 완수 보고서 · S1 회고 · ChatGPT 논의(Ready for AI SW) · MCP 스펙 분석 · 챗봇 연동 인계 메모
**상태**: 초안 — Phase 0 착수 전 합의 문서

---

## 0. S2 정체성

### 한 줄 정의

> **S2 = Mimir를 "AI 에이전트가 1급 시민으로 소비·기록·행동하는 지식 인프라"로 진화시키는 스프린트**

S1이 "사람이 쓰는 문서 플랫폼의 0→1 구축"이었다면, S2는 "AI 소프트웨어가 이 플랫폼을 자기 지식 백본으로 쓸 수 있게 만드는" 전환이다.

### S1에서 S2로의 전환 근거

S1 종료 시점의 Mimir는 인증·RBAC·문서 CRUD·워크플로·버전/diff·검색(FTS + pgvector)·RAG(SSE 스트리밍)·문서유형 플러그인·관리 대시보드·감사/알림/배치/API 키 관리를 갖춘 완결된 제품이다. 그러나 "AI SW Ready" 관점에서 보면 다음이 빠져 있다:

1. AI 에이전트가 Mimir를 도구로 호출할 표준 계약(MCP/Tool schema)이 없다
2. 검색 응답의 citation이 검증 가능한 좌표(버전·노드·해시)를 갖추지 않았다
3. RAG 품질의 회귀를 측정하는 평가 인프라가 없다
4. 에이전트가 문서를 안전하게 쓰거나 워크플로를 전이하는 표면이 없다
5. 멀티턴 대화(챗봇 세션)가 도메인 객체로 존재하지 않는다
6. 모델/프롬프트가 하드코딩에 가깝고, 폐쇄망 배포를 구조적으로 지원하지 않는다

S2는 이 6가지를 해소한다.

---

## 1. 설계 원칙

### S1 4대 원칙 (유지)

| # | 원칙 | 요약 |
|---|------|------|
| ① | API-first | 모든 기능은 API로 먼저 정의. UI는 클라이언트 중 하나 |
| ② | Dual UI | User UI / Admin UI 분리 |
| ③ | Document = 구조, Type = 행동 | Document는 generic, DocumentType이 동작 정의 |
| ④ | 권한은 모든 계층에 적용 | API · 검색 · 벡터 · RAG |

### S2 추가 원칙

| # | 원칙 | 요약 |
|---|------|------|
| ⑤ | **AI는 1급 사용자다 (AI-as-first-class-consumer)** | 모든 기능은 사람 UI 이전에 AI 에이전트가 안전하게 호출·검증·기록할 수 있도록 정의된다 |
| ⑥ | **접근 범위도 하드코딩 금지** | 외부 소비자의 scope 어휘는 Scope Profile로 관리자가 설정. Mimir 코드에 특정 scope 문자열이 등장하지 않는다 |
| ⑦ | **폐쇄망 동등성** | 모든 기능은 외부 네트워크 없이 동작 가능해야 한다. 외부 의존(OpenAI, SaaS 트레이싱 등)은 환경변수로 off 가능하고, off 시에도 기능이 degrade될 뿐 실패하지 않는다 |

---

## 2. 핵심 제약 조건

### 2.1 1차 소비자: 챗봇

S2의 첫 번째 외부 소비자는 별도 프로젝트로 개발 중인 챗봇이다. 이 챗봇의 특성:

- 독립된 백엔드 프로세스에서 HTTP로 Mimir를 호출한다 (서버-투-서버)
- 외부 AI 모델(OpenAI, Anthropic 등) 또는 내부 폐쇄망 모델(vLLM, Ollama 등)을 사용한다
- 멀티소스 RAG 구조 — Mimir는 여러 retrieval layer 중 하나
- `rag_orchestrator`가 소스 합산·dedup·partial failure를 처리한다

참조: `챗봇 S2-2 Mimir 연동 인계 메모` (별도 문서)

### 2.2 폐쇄망 배포

- 모든 외부 의존(OpenAI API, LangSmith, OTel Cloud 등)은 환경변수로 off 가능
- 모델 아티팩트/프롬프트 템플릿은 Mimir DB에 저장 (외부 MLflow 의존 없음)
- 평가용 판정 LLM도 스왑 가능 — 폐쇄망에서는 자체 호스팅 모델이 판정
- 임베딩 모델은 로컬(BGE-M3, E5 등)이 1급

### 2.3 에이전트는 읽기와 쓰기 모두 수행

- 에이전트는 검색·조회 외에 Draft 생성, 수정, 워크플로 전이 제안까지 가능
- 모든 에이전트 쓰기는 `proposed` 상태로만 진입하며, 기존 워크플로의 review/approve 게이트를 통과해야 `published`가 된다
- "에이전트는 Draft까지 만드는 사람"으로 모델링

### 2.4 평가는 필수

- Phase별 산출물 표준에 `AI품질평가.md`를 세 번째 의무 문서로 추가
  - 기존: 검수보고서 + 보안취약점검사보고서
  - S2: 검수보고서 + 보안취약점검사보고서 + **AI품질평가보고서**

### 2.5 외부 커넥터는 S3로

- S2는 "Mimir 내부에 올라온 지식"에 대해서만 AI가 1급 시민
- 외부 소스(GitLab Wiki, Confluence, Notion, SharePoint 등) 연동은 S3에서 별도의 AI-native 계획 도구 개발과 함께 진행

---

## 3. 핵심 설계 결정

S2 착수 전 합의된 설계 결정 목록이다. 각 결정은 해당 Phase의 상세 설계에서 구체화된다.

### 3.1 에이전트 인증 — OAuth client-credentials + access_context

챗봇 서버 ↔ Mimir 서버 간 인증은 MCP 2025-11-25 스펙의 **OAuth 2.0 client-credentials** 플로우를 사용한다.

- 챗봇은 자신의 client-credentials로 Mimir에 인증한다
- 사용자 대행 검색 시 request body에 `access_context`를 포함한다
- **Mimir는 access_context를 그대로 신뢰하지 않는다** — `user_id`로 DB를 조회해 실제 권한을 재검증한다
- API Key scope에 `delegate:search`, `delegate:write` 등을 추가하여 대행 권한을 명시적으로 관리한다

```
[챗봇 서버]
  ├─ 자신의 인증: OAuth client-credentials → Mimir API
  └─ 사용자 대행: access_context {user_id, org_id, permissions} → body에 포함
                                                                    ↓
[Mimir 서버]
  ├─ client-credentials 검증 → 이 agent에게 delegate 권한이 있는가?
  └─ access_context.user_id → DB 조회 → 실제 ACL 적용
```

### 3.2 Scope Profile — 관리자 설정 기반 ACL 그룹핑

외부 소비자의 scope 어휘(`team`, `org`, `confidential` 등)는 Mimir 코드에 등장하지 않는다. 대신 **Scope Profile**이라는 관리자 설정 계층을 둔다.

**개념 모델:**
```
ScopeProfile (name, description)
└─ ScopeDefinition[] (scope_name, acl_filter: FilterExpression)
```

**작동 원리:**
1. 관리자가 Admin UI에서 Scope Profile을 생성하고, 각 scope name에 ACL 필터 규칙을 정의한다
2. API Key 발급 시 어떤 Scope Profile을 사용하는지 바인딩한다
3. 외부 소비자가 `scope: "team"`으로 요청하면, Mimir는 해당 API Key의 Scope Profile에서 `"team"` → ACL 필터 규칙을 해석한다
4. 해석된 필터를 기존 ACL 엔진에 AND로 합성한다

**FilterExpression (S2 수위):**
- 미리 정의된 필터 필드(`organization_id`, `team_id`, `visibility`, `classification` 등)의 조합만 지원
- `$ctx.*` 변수로 access_context에서 동적 치환 (예: `$ctx.organization_id`)
- 임의 SQL-like 표현식은 S3에서 필요 시 확장

**예시:**
```json
{
  "name": "기본 챗봇용",
  "scopes": [
    {
      "scope_name": "team",
      "acl_filter": {
        "and": [
          {"field": "organization_id", "op": "eq", "value": "$ctx.organization_id"},
          {"field": "team_id", "op": "eq", "value": "$ctx.team_id"}
        ]
      }
    },
    {
      "scope_name": "org",
      "acl_filter": {
        "and": [
          {"field": "organization_id", "op": "eq", "value": "$ctx.organization_id"}
        ]
      }
    }
  ]
}
```

### 3.3 MCP 스펙 핀 — 2025-11-25

S2의 MCP 구현은 **MCP specification version 2025-11-25**을 기준으로 한다.

**활용하는 스펙 기능:**

| MCP 기능 | Mimir 적용 |
|----------|-----------|
| Tools | `search_documents`, `fetch_node`, `create_draft`, `propose_transition` 등 |
| Resources | `mimir://documents/{id}/versions/{v}/nodes/{n}` 형태로 문서 조각 노출 |
| Prompts | Prompt Registry의 템플릿을 MCP 프롬프트로 노출 |
| Tasks (experimental) | 에이전트 쓰기의 비동기 승인 플로우 (proposed → approved → completed) |
| OAuth client-credentials | 서버-투-서버 인증 |
| Extensions | Mimir-specific 기능(citation 5-tuple, span 역참조)을 Mimir extension으로 선언 |

**호환 정책:**
- Phase 4 종결 시 "Mimir는 MCP 2025-11-25을 구현한다"를 공식 문서화
- 스펙 하위 호환이 깨지는 MCP 버전 업그레이드는 별도 Phase로 취급

### 3.4 Citation Contract — 5-tuple

Phase 2에서 검색 응답의 citation을 아래 5-tuple로 고정한다:

```
{document_id, version_id, node_id, span_offset (optional), content_hash}
```

- `version_id`: 문서 개정 후에도 citation이 유효한지 검증 가능
- `content_hash`: 청크 내용이 원문과 동일한지 검증 가능
- `span_offset`: 긴 노드의 부분 매칭에서 사용, 단락 단위 청크에서는 생략 가능
- 챗봇 인계 메모 v2는 Phase 2 종결 시 이 계약을 반영하여 산출

### 3.5 에이전트 Principal 모델

에이전트는 사용자와 별개의 **독립 주체(Principal)**로 모델링한다.

- S1 Phase 2의 역할 모델에 `agent` principal type 추가
- 사용자가 에이전트에게 **위임(delegation)**하는 형태
- 감사로그에 `actor_type=agent, acting_on_behalf_of=<user_id>` 기록
- 에이전트 단위 킬스위치: 문제 발생 시 특정 에이전트의 쓰기를 즉시 차단 가능

### 3.6 평가용 판정 전략

폐쇄망에서 "판정 LLM 품질 = 평가 지표 품질"이 된다. S2에서는 **LLM 의존도를 낮춘 판정부터 시작**하고 점진 확장한다.

| 판정 유형 | LLM 필요 | S2 도입 |
|----------|---------|--------|
| Citation-present check (응답에 근거가 있는가) | 불필요 | ✅ |
| 근거 문장 ↔ 원문 매칭 (content_hash / 유사도) | 불필요 | ✅ |
| Answer relevance (질문-답변 의미 유사도) | embedding만 | ✅ |
| Faithfulness (답변이 근거에 충실한가) | LLM 판정 | ✅ (스왑 가능 LLM) |
| Context precision / recall | LLM 판정 | ✅ (스왑 가능 LLM) |

판정 LLM은 Phase 1의 모델 추상화 계층을 그대로 사용하여, 외부망에서는 GPT-4/Claude, 폐쇄망에서는 로컬 모델로 자동 전환된다.

---

## 4. Phase 구조

### 운영 규약: Feature Group 소분

S1 회고 §4.4에 따라, 각 Phase를 2~3개 **Feature Group(FG)**으로 분할한다. 각 FG는 독립적인 검수 게이트를 가진다.

**Phase별 의무 산출물:**
- Phase 개발계획서
- FG별 작업지시서
- FG별 검수보고서
- FG별 보안취약점검사보고서
- **FG별 AI품질평가보고서** (AI 기능이 포함된 FG에 한함)
- Phase 종결 보고서

### S1 부채 선행 흡수: Phase 0

S1 회고 §7의 7개 Action Item은 S2 Phase 0에서 선행 처리한다.

---

### Phase 0. S2 플랫폼 원칙 수립 + S1 부채 청산

**목적**: S2 설계 기준 확정 및 S1에서 이월된 기술 부채 해소

#### FG0.1 — S2 원칙 및 규약 확정
- 제⑤⑥⑦ 원칙 공식 문서화
- `AI품질평가.md` 산출물 규약 확정 (평가 항목, 기준값, 보고서 포맷)
- Feature Group 소분 운영 규약 확정
- CLAUDE.md §2 절대 규칙에 "접근 범위도 하드코딩 금지" 추가
- 챗봇 연동 인계 메모를 참조 문서로 등록

#### FG0.2 — S1 기술 부채 청산
- **[Critical]** 사용자 메뉴 및 네비게이션 구축 — 현재 로그아웃·사용자 설정·관리자 설정 진입 경로가 전무함
  - User UI 헤더에 사용자 메뉴 (프로필, 설정, 로그아웃)
  - Admin 역할 사용자에게 "관리자 설정" 진입점 노출
  - User UI ↔ Admin UI 간 전환 네비게이션
  - S1 Phase 7/14에서 구축된 Admin 페이지들(대시보드, 사용자/역할 관리, 시스템 설정, 모니터링, 감사, API 키)로의 라우팅 연결
- **[High]** 프런트 `unwrapEnvelope<T>()` 도입 및 모든 `*Api` 리팩토링
- **[High]** Fresh-boot 스모크 CI 잡 (pgvector 있음/없음 매트릭스)
- **[High]** `/api/v1/system/capabilities` 엔드포인트 신설 (`.well-known` 규약 정합, 최소 노출: 사용 가능 provider 목록, pgvector 활성 여부, RAG 가용 여부 — Phase 1에서 provider 목록을 확장)
- **[Mid]** `seed_users.py` production 가드 + 감사 이벤트
- **[Mid]** UI 5회 리뷰의 여정 기반 체크리스트 표준화
- **[Low]** 회귀 게이트 스크립트 FG 단위 복제 준비

**결과**: S2 이후 모든 설계의 기준 문서 완성 + S1 잔존 부채 0건 + 사용자/관리자 네비게이션 정상 작동

---

### Phase 1. 모델·프롬프트 추상화 계층

**목적**: 외부/폐쇄망 모델을 동등하게 사용할 수 있는 기반층 구축. 이후 모든 Phase가 이 추상화 위에서 동작한다.

#### FG1.1 — LLM Provider Abstraction
- Provider 인터페이스 정의 (generate, stream, embed 통합)
- OpenAI / Anthropic / vLLM / Ollama / llama.cpp 어댑터 구현
- 비용·지연 메타데이터 포함, 라우팅 정책 기초
- 판정 LLM도 동일 추상화 사용
- `system/capabilities`에 사용 가능 provider 목록 노출

#### FG1.2 — Embedding Model Abstraction
- OpenAI Embedding / BGE-M3 / E5 등 로컬 모델 어댑터
- 재임베딩 마이그레이션 파이프라인 (모델 변경 시 기존 벡터 재생성)
- 임베딩 차원 불일치 방어 (모델 변경 시 pgvector 인덱스 재구축 절차)

#### FG1.3 — Prompt Registry
- 프롬프트 템플릿을 DB에 저장 (Versioned)
- 런타임 스왑 (A/B 테스트 가능)
- 프롬프트 CRUD + 버전 이력 조회 API (Admin UI는 Phase 6 FG6.1에서 구축)
- MCP Prompts 원시와의 연결 고리 준비

**결과**: `gpt-4o` 사고류의 구조적 차단. 폐쇄망에서 로컬 모델만으로 전 기능 동작 가능.

---

### Phase 2. Grounded Retrieval v2 + Citation Contract

**목적**: 검색 응답을 "검증 가능한 근거 좌표"로 고정하고, Retriever/Reranker를 플러그인 구조로 재정리

#### FG2.1 — Citation 5-tuple 계약 고정
- 검색 응답에 `{document_id, version_id, node_id, span_offset, content_hash}` 필수 포함
- span → 원문 역참조 API (`GET /api/v1/citations/{hash}/verify`)
- 기존 S1 Phase 11 RAG 응답 포맷의 하위호환 마이그레이션

#### FG2.2 — Retriever/Reranker 플러그인화
- Retriever 인터페이스 분리 (FTS, Vector, Hybrid)
- Reranker 인터페이스 분리 (cross-encoder, 룰 기반)
- DocumentType별 retriever/reranker 설정 가능
- Phase 1의 모델 추상화 위에서 동작

#### FG2.3 — 쿼리 재작성 + 멀티턴 컨텍스트 압축
- 단발 RAG → 멀티턴 대화에서 follow-up query를 자립적 쿼리로 재작성
- 이전 턴의 citation 재활용 전략
- Phase 1의 LLM 추상화를 통해 재작성 모델 선택

**결과**: 검증 가능한 citation 계약 확보. 챗봇 인계 메모 v2 산출 (version_id, content_hash 반영).
**특별 산출물**: Phase 2 종결 시 `챗봇 연동 인계 메모 v2`를 챗봇 프로젝트에 전달. Phase별 의무 산출물 외 추가 산출.

---

### Phase 3. Conversation 도메인

**목적**: 멀티턴 대화를 1급 도메인 객체로 승격. 챗봇의 세션 관리를 Mimir가 지원하는 구조.

#### FG3.1 — Conversation / Turn / Message 도메인 모델
- `Conversation` (세션) → `Turn` (한 번의 질의-응답) → `Message` (사용자/시스템/에이전트 메시지)
- Document와 대칭적 권한·감사·보존 정책 적용
- 대화 이력 보존 기간 및 삭제 정책 설계 (개인정보 포함 가능성 고려 — 사용자 질의에 민감 정보가 포함될 수 있으므로 자동 만료/수동 삭제 API 필수)
- 대화 이력 저장 및 조회 API

#### FG3.2 — 세션 기반 RAG API
- 기존 단발 `/rag/answer` 의 상위호환: `conversation_id` 포함 시 멀티턴 모드
- 턴 간 컨텍스트 윈도우 관리 (요약/압축 전략)
- 이전 turn의 retrieval 결과 캐시 및 재활용

#### FG3.3 — User UI 챗 인터페이스
- 대화형 RAG UI (User UI 내)
- 턴별 citation 표시 + 원문 참조 링크
- 대화 이력 목록/검색

**결과**: 챗봇이 Mimir를 세션 기반 지식 백엔드로 사용 가능. 사람도 동일 인터페이스 사용 가능.

---

### Phase 4. Agent-Facing Interface

**목적**: AI 에이전트가 Mimir를 도구로 호출할 수 있는 표준 계약 제공

#### FG4.1 — MCP Server 노출
- MCP 2025-11-25 기반 서버 구현
- Read tools: `search_documents`, `fetch_node`, `verify_citation`
- Streamable HTTP 전송
- `initialize` 핸드셰이크 시 사용 가능한 scope 목록을 capability로 선언
- `.well-known` 메타데이터로 서버 capability 사전 발견 가능

#### FG4.2 — Agent Principal + Delegation + Scope Profile
- Agent principal type을 역할 모델에 추가
- delegation 위임 구조: agent는 사용자를 대행하여 호출
- API Key 발급 시 Scope Profile 바인딩
- **Scope Profile CRUD API**: scope name → ACL filter 매핑 관리 (Admin UI는 Phase 6 FG6.2에서 구축)
- API Key scope에 `delegate:search`, `delegate:write` 추가
- 에이전트 킬스위치 API: 특정 에이전트의 쓰기 즉시 차단 (Admin UI는 Phase 6 FG6.2에서 구축)

#### FG4.3 — Structured Response + Tool Schema
- OpenAPI → MCP Tool schema 선택적 변환 (전체 자동 변환은 지양, 큐레이션된 tool set 제공)
- 에이전트용 구조화 응답 포맷 표준화
- Mimir Extension 선언 (citation 5-tuple, span 역참조 등)

**결과**: 챗봇이 MCP 또는 REST로 Mimir를 도구로 호출 가능. 인증·권한·scope가 표준 계약으로 정리.

---

### Phase 5. Agent Action Plane

**목적**: 에이전트가 문서를 안전하게 쓰고, 워크플로를 전이하는 계층 구축. **S2의 하이라이트 Phase.**

#### FG5.1 — 에이전트 쓰기 = Draft 제안 전용
- 에이전트의 문서 생성/수정은 `proposed` 상태의 Draft로만 진입
- 워크플로 전이 제안 API: `propose_transition(document_id, target_state, reason)`
- MCP Tasks 활용: 제안 → 인간 승인 → 확정의 비동기 플로우
  - `working` → `input_required`(승인 대기) → `completed`(확정) / `failed`(반려)
- 모든 에이전트 쓰기는 감사로그에 `actor_type=agent, acting_on_behalf_of=<user_id>` 기록

#### FG5.2 — 에이전트 제안 큐 + 프롬프트 인젝션 정책
- 에이전트 제안 큐 API: 승인/반려/회수 관리 (Admin UI는 Phase 6 FG6.2에서 구축)
- User UI "내 문서에 대한 에이전트 제안" 뷰
- **컨텐츠-지시 분리 정책**: 검색된 텍스트는 항상 untrusted로 프레이밍
  - 에이전트에게 전달되는 검색 결과에 명확한 delimiter + 메타데이터
  - 공격성 instruction 문구 탐지 (룰 기반 + 선택적 LLM 판정)
  - injection regression test 추가
- Mimir 응답에 `trusted: false` 메타 플래그 포함

#### FG5.3 — 에이전트 감사 뷰 + 회귀 테스트
- 에이전트 활동 감사 API: 에이전트별 행동 이력, 승인률, 반려 사유 통계 (Admin UI는 Phase 6 FG6.2에서 구축)
- 에이전트별 회귀 테스트 스크립트
- 에이전트 쓰기 시나리오 통합 검증

**결과**: 에이전트가 안전하게 문서를 제안하고, 인간이 승인하는 구조 완성. 감사·회수 체계 확보.

---

### Phase 6. 관리자 기능 통합

**목적**: S2에서 추가된 모든 관리 기능(모델·프롬프트·에이전트·Scope Profile·골든셋·추출 스키마)을 하나의 통합 관리 콘솔로 구축. Phase 0에서 복구한 기본 네비게이션 위에, S2 전용 관리 경험을 얹는다.

> Phase 1~5에서는 각 기능의 **백엔드 API**를 구현한다. Phase 6에서는 그 API 위에 **관리자가 실제로 조작하는 UI와 운영 도구**를 일괄 구축한다. 이렇게 분리하는 이유: 관리 UI는 여러 도메인을 가로지르는 통합 경험이며, 각 Phase에 분산시키면 네비게이션·레이아웃·권한 체크가 제각각이 된다.

#### FG6.1 — AI 플랫폼 관리 대시보드
- **모델/프로바이더 관리**: 등록된 LLM·임베딩 프로바이더 목록, 상태(활성/비활성/오류), 연결 테스트, 기본 모델 지정
- **프롬프트 관리**: Phase 1에서 구축한 Prompt Registry의 Admin UI — 프롬프트 CRUD, 버전 이력, A/B 설정, 활성 버전 스왑
- **Capabilities 대시보드**: `/system/capabilities` 데이터를 시각화 — 현재 가용한 기능(RAG, 벡터, 임베딩, LLM), degrade 상태 모니터링
- **비용·사용량 현황**: 모델별 토큰 사용량, 비용 추이, 프로바이더별 응답 지연 (Phase 1에서 수집한 메타데이터 기반)

#### FG6.2 — 에이전트·Scope 관리 콘솔
- **에이전트 관리**: 등록된 에이전트 목록, 상태(활성/차단), 권한(delegate scope), 킬스위치 조작
- **Scope Profile 관리**: Phase 4에서 구축한 Scope Profile CRUD의 Admin UI — 프로필 생성/수정/삭제, scope name ↔ ACL filter 매핑 편집기, API Key 바인딩 관리
- **에이전트 제안 큐**: Phase 5에서 구축한 제안 API의 Admin UI — 전체 제안 목록, 승인/반려/회수 일괄 조작, 필터(에이전트별/문서별/상태별)
- **에이전트 활동 대시보드**: 에이전트별 행동 이력, 승인률 추이, 반려 사유 분석, 이상 행동 알림

#### FG6.3 — 평가·추출 관리
- **골든셋 관리**: 골든 Q&A Set CRUD, 버전 이력, import/export (JSON), 평가 실행 이력
- **평가 결과 대시보드**: 최근 평가 실행 결과 요약, 지표 추이(faithfulness/relevance/precision/recall), CI 게이트 상태
- **추출 타겟 스키마 관리**: DocumentType별 추출 스키마 정의/편집, 추출 결과 승인 현황
- **추출 결과 검토 큐**: 미승인 추출 결과 목록, 승인/수정/반려

**결과**: S2에서 추가된 모든 관리 기능이 하나의 통합 콘솔에서 조작 가능. S1 Admin 기능(사용자/역할/설정/감사/API키)과 S2 Admin 기능(모델/에이전트/Scope/평가/추출)이 일관된 네비게이션으로 통합.

---

### Phase 7. AI 품질 평가 인프라

**목적**: RAG 품질 회귀를 정량적으로 측정하고, CI 게이트로 방어하는 체계 구축

#### FG7.1 — 골든셋 도메인 모델 + API
- Golden Q&A Set: (질문, 기대 답변, 기대 근거 문서, 기대 citation) 4-tuple
- Versioned 리소스로 관리 (Document와 유사한 생명주기)
- 골든셋 CRUD API + import/export (JSON)
- Admin UI는 Phase 6 FG6.3에서 구축

#### FG7.2 — 평가 러너 + 지표
- 평가 지표:
  - **Faithfulness**: 답변이 근거에 충실한가 (LLM 판정, 스왑 가능)
  - **Answer Relevance**: 질문-답변 의미 유사도 (embedding 기반)
  - **Context Precision**: 검색된 청크 중 관련 있는 비율
  - **Context Recall**: 관련 청크 중 검색된 비율
  - **Citation-present rate**: 응답에 검증 가능한 근거가 있는 비율 (룰 기반)
  - **헛소리율**: citation 없는 답변 비율 (룰 기반)
  - **지연·토큰·비용**: 모델/프롬프트 버전별 측정
- 평가 러너는 Phase 1의 모델 추상화를 그대로 사용
- 판정 LLM이 없는 환경에서도 룰 기반 지표(citation-present, 원문 매칭)는 동작

#### FG7.3 — CI 게이트
- CI 파이프라인에 "골든셋 회귀 테스트" 잡 추가
- 임계값 이하 시 빌드 실패 (예: faithfulness < 0.8)
- Grafana 연동은 Phase 6 FG6.3의 평가 결과 대시보드에서 처리
- Phase별 `AI품질평가.md` 자동 생성 템플릿

**결과**: AI 기능의 품질이 "검수/보안 보고서"와 동급의 게이트로 방어됨.

---

### Phase 8. Structured Extraction

**목적**: 문서에서 구조화 데이터를 LLM 보조로 추출하여, Mimir를 "지식 그래프의 씨앗"으로 진화

#### FG8.1 — DocumentType metadata schema → 추출 타겟 스키마
- DocumentType의 metadata schema를 추출 타겟으로 재사용
  - 예: 규정 → `{조항번호, 책임자, 발효일, 요약}`
  - 예: 매뉴얼 → `{절차명, 선행조건, 수행자, 산출물}`
- 추출 타겟 스키마 정의 API (Admin UI는 Phase 6 FG6.3에서 구축)

#### FG8.2 — LLM 보조 추출 + 인간 승인
- 추출 파이프라인: 문서 업로드/개정 → 자동 추출 후보 생성 → 인간 승인/수정
- Phase 1의 모델 추상화 사용
- 추출 결과는 Document의 metadata 필드에 저장 (generic + config 원칙 유지)
- User UI "추출 결과 검토" 뷰

#### FG8.3 — 추출 결과의 검증 가능 계약
- 추출된 각 필드에 원문 span 역참조 (어디서 추출했는가)
- 재실행 가능성: 동일 모델 + 동일 프롬프트 + 동일 문서 → 동일 결과 (결정론적 추출 모드)
- 추출 품질 평가: Phase 7의 골든셋 패턴을 추출에도 적용

**결과**: 문서 → 구조화 데이터 자동 추출 체계. 지식 그래프·S3 AI-native 계획 도구의 데이터 기반.

---

### Phase 9. S2 통합·보안·평가 종결 게이트

**목적**: S2 전체 회귀 방어선 구축 및 종결

#### FG9.1 — OWASP Top 10 for LLM Applications 전 항목 점검
- 일반 OWASP Top 10 (A01~A10) 재검증 (S1 수준 유지)
- **OWASP Top 10 for LLM Applications** 전 항목 추가 점검:
  - LLM01: Prompt Injection
  - LLM02: Insecure Output Handling
  - LLM03: Training Data Poisoning (해당 시)
  - LLM06: Sensitive Information Disclosure
  - LLM08: Excessive Agency
  - 기타 해당 항목

#### FG9.2 — S2 통합 회귀 + 골든셋 회귀
- S1 `test_integration_security_phase14_16.py` 확장
- Phase별 FG 회귀 스크립트 통합
- 골든셋 기반 RAG 품질 회귀 게이트
- MCP 연동 e2e 테스트
- S3 드라이런: MCP 스펙을 S3 AI-native 계획 도구와 시험 연결

#### FG9.3 — S2 완수 보고서 / 회고 / S3 Action Item
- S2 완수 보고서 (S1 완수 보고서와 대칭)
- S2 회고 (KPT + 정량·정성)
- S3 Action Item 정리
- 챗봇 인계 메모 v3 산출 (S2 전 기능 반영)

**결과**: S2 종결. 운영 투입 가능 상태로 마감. S3 착수 기반 확보.

---

## 5. Phase 의존 관계

```
Phase 0 (원칙 + 부채)
    │
    ▼
Phase 1 (모델/프롬프트 추상화) ──── 기반층, 이후 모든 Phase가 의존
    │
    ├──▶ Phase 2 (Grounded Retrieval v2) ──▶ Phase 3 (Conversation)
    │                                              │
    │                                              ▼
    └──▶ Phase 4 (Agent Interface) ◀──────────────┘
              │
              ▼
         Phase 5 (Agent Action Plane)
              │
              ▼
         Phase 6 (관리자 기능 통합) ── FG6.1~6.2: Phase 1~5 API 위에 구축
              │                         FG6.3: Phase 7~8 완료 후 최종 연결
              ▼
         Phase 7 (AI 품질 평가) ── 평가는 Phase 2~5 산출물에 소급 적용 가능
              │
              ▼
         Phase 8 (Structured Extraction) ── Phase 7의 골든셋 패턴을 추출 평가에도 적용
              │
              ▼
         Phase 9 (통합 종결) ── Phase 6 FG6.3 최종 검증 포함
```

**핵심 의존:**
- Phase 1은 선행 필수. 나머지 모든 Phase가 모델 추상화에 의존한다.
- Phase 3(Conversation)이 Phase 4(Agent Interface) 앞에 온다. 챗봇이 1차 소비자이므로, 에이전트 인터페이스가 세션 위에 얹히는 것이 자연스럽다.
- Phase 6(관리자 기능 통합)은 Phase 1~5의 백엔드 API를 전제로 한다. FG6.1(AI 플랫폼 대시보드)과 FG6.2(에이전트·Scope 콘솔)는 Phase 5 완료 시점에 구축 가능하다. FG6.3(평가·추출 관리)은 스켈레톤을 Phase 6에서 만들되, Phase 7·8의 백엔드 완료 후 최종 연결한다.
- Phase 7(평가) → Phase 8(Structured Extraction)은 순차 의존. Phase 8이 Phase 7의 골든셋 패턴(평가 러너, 품질 지표)을 추출 품질 평가에 재사용하기 때문이다.
- Phase 7(평가)은 Phase 6 이후 배치하지만, 평가 기준은 Phase 0에서 확정하므로 **각 Phase의 AI품질평가는 Phase 7 이전에도 수동으로 수행 가능**.
- Phase 9의 S3 드라이런은 S3 AI-native 계획 도구와의 연결 가능성을 사전 검증한다. Phase 6 FG6.3의 최종 연결 검증도 이 Phase에서 마무리한다.

---

## 6. S3와의 경계

S2가 만들어주는 S3의 전제 조건:

| S2 산출물 | S3에서의 역할 |
|----------|-------------|
| MCP Server (Phase 4) | S3 AI-native 계획 도구가 Mimir를 소비하는 1차 인터페이스 |
| Scope Profile (Phase 4) | S3 외부 커넥터가 새로운 scope 어휘를 가져올 때 관리자가 매핑 |
| Citation 5-tuple (Phase 2) | S3의 크로스-소스 citation에서 Mimir 측 앵커 |
| Structured Extraction (Phase 8) | S3 AI-native 계획 도구의 데이터 기반 (추출된 구조화 지식) |
| 평가 인프라 (Phase 7) | S3 기능 추가 시에도 동일 품질 게이트 사용 |

S3 범위 (S2에서 제외된 것):
- 외부 소스 커넥터 (GitLab Wiki, Confluence, Notion, SharePoint, Jira)
- AI-native 계획 도구 개발
- FilterExpression 수위 2 (임의 SQL-like 표현식)
- 멀티테넌시 / SaaS 확장 (필요 시)

---

## 7. 정량 목표

| 지표 | S1 결과 | S2 목표 |
|------|--------|--------|
| 잔존 결함 / Phase | 0 | 0 (유지) |
| OWASP Top 10 커버 | 10/10 | 10/10 + LLM Top 10 |
| 단위 테스트 커버리지 | ≥80% | ≥85% |
| 사인오프 후 발견 이슈 | 5건 | ≤1건 (fresh-boot 방어) |
| RAG faithfulness (골든셋) | 미측정 | ≥0.80 |
| RAG answer relevance | 미측정 | ≥0.75 |
| Citation-present rate | 미측정 | ≥0.90 |
| 프런트 envelope 산발 | 다수 | 0건 |
| 폐쇄망 전 기능 동작 | 미검증 | 100% (CI 매트릭스) |
| 에이전트 제안 승인률 | 미측정 | ≥0.70 (목표, 반려 사유 분석 포함) |
| 프롬프트 인젝션 탐지율 | 미측정 | ≥0.95 (룰 기반 + 선택적 LLM) |
| 에이전트 킬스위치 발동→차단 지연 | 미측정 | <5초 |

---

## 8. 참조 문서

| 문서 | 위치 / 출처 |
|------|-----------|
| S1 완수 보고서 | `docs/개발문서/S1/S1 완수 보고서.md` |
| S1 회고 | `docs/개발문서/S1/S1 회고.md` |
| Project description v2 | `docs/Project description.md` |
| 챗봇 S2-2 Mimir 연동 인계 메모 | 챗봇 프로젝트 별도 관리 |
| MCP Specification 2025-11-25 | `https://modelcontextprotocol.io/specification/2025-11-25` |
| MCP 2026 Roadmap | `https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/` |
| OWASP Top 10 for LLM Applications | `https://owasp.org/www-project-top-10-for-large-language-model-applications/` |
