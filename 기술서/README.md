# Mimir 기술서

**버전**: 2.0 (S3 Phase 4 종결 기준)
**최종 수정**: 2026-04-28

Mimir 플랫폼의 전체 기술 문서 인덱스입니다.

---

## 폴더 구조

```
기술서/
├── README.md                         ← 이 파일 (전체 인덱스)
│
├── 1. 시스템 아키텍처/
│   ├── 시스템 개요.md                 ← 전체 구조, 계층, 컴포넌트 다이어그램
│   └── 기술 스택.md                  ← 사용 기술 목록 및 선택 근거
│
├── 2. 도메인 모델/
│   ├── 핵심 엔티티.md                 ← Document / Version / Node 구조
│   ├── 문서 상태와 워크플로.md         ← 상태 전이 다이어그램, 워크플로 규칙
│   └── 권한과 역할 모델.md            ← 역할 정의, ACL, 권한 계층
│
├── API 기술서/
│   ├── API 공통 규약.md               ← 응답 포맷, 에러 코드, 페이지네이션
│   ├── 인증과 인가.md                 ← JWT, API Key, 세션, 인증 흐름
│   ├── 문서-버전-노드 API.md          ← 핵심 CRUD 엔드포인트
│   ├── 워크플로 API.md                ← 승인/검토/반려 API
│   ├── 검색 API.md                   ← FTS, 벡터 검색 API
│   ├── 벡터화와 RAG API.md            ← 임베딩, RAG 질의 API
│   └── 관리자 API.md                 ← 사용자/조직/역할/설정 관리 API
│
├── MCP 기술서/                       ← S3 Phase 4 신설 — 외부 AI 에이전트 인터페이스
│   ├── MCP 표면 개요.md               ← R1~R6, 9 도구, mimir:// URI, ScopeProfile
│   ├── 도구별 명세.md                 ← 9 도구 입출력 schema + envelope + ACL
│   ├── 매니페스트 정책.md             ← 9 manifest 필드, 외부/운영자 view 분리
│   ├── Tool ACL 모델.md               ← ScopeProfile.allowed_tools 두 차원 ACL
│   └── Citation 모델.md               ← 5-tuple + citation_basis + 5중 검증
│
├── DB 기술서/
│   └── 스키마 정의.md                 ← 전체 테이블 DDL 및 설명 (S3 신규 테이블 포함)
│
├── 5. 플러그인 시스템/
│   ├── 플러그인 시스템 개요.md         ← 아키텍처, 레지스트리, 내장 타입
│   └── 플러그인 개발 가이드.md         ← 커스텀 타입 추가 절차
│
├── 6. 보안 모델/
│   └── 보안 모델.md                  ← 인증/인가/OWASP/취약점 이력 + Scope Profile + MCP R1~R6 + L0~L4 등급
│
├── 7. 운영/
│   ├── 환경 설정과 배포.md            ← 환경 변수, Docker, CI/CD
│   └── 모니터링과 로깅.md             ← Prometheus, Grafana, 구조화 로그
│
└── 사용 설명서/
    ├── 빠른 시작.md                   ← 로컬 실행부터 첫 문서 생성까지
    └── 주요 기능 안내.md              ← 핵심 기능별 사용 방법
```

---

## 빠른 탐색

| 목적 | 문서 |
|------|------|
| 시스템 전체를 한눈에 파악 | [시스템 개요](1.%20시스템%20아키텍처/시스템%20개요.md) |
| API 호출 방법 | [API 공통 규약](API%20기술서/API%20공통%20규약.md) |
| 인증 방법 | [인증과 인가](API%20기술서/인증과%20인가.md) |
| 데이터 모델 | [핵심 엔티티](2.%20도메인%20모델/핵심%20엔티티.md) |
| DB 스키마 | [스키마 정의](DB%20기술서/스키마%20정의.md) |
| 새 문서 유형 추가 | [플러그인 개발 가이드](5.%20플러그인%20시스템/플러그인%20개발%20가이드.md) |
| 배포 방법 | [환경 설정과 배포](7.%20운영/환경%20설정과%20배포.md) |
| 처음 사용 | [빠른 시작](사용%20설명서/빠른%20시작.md) |
| **외부 AI 에이전트 / MCP 통합** | [MCP 표면 개요](MCP%20기술서/MCP%20표면%20개요.md) |
| **MCP 도구 9종 명세** | [도구별 명세](MCP%20기술서/도구별%20명세.md) |
| **Citation 5중 검증** | [Citation 모델](MCP%20기술서/Citation%20모델.md) |
| **Scope Profile 기반 ACL** | [Tool ACL 모델](MCP%20기술서/Tool%20ACL%20모델.md) |
| **MCP 보안 R1~R6** | [보안 모델 §12](6.%20보안%20모델/보안%20모델.md) |

---

## Mimir 한 줄 요약

> **Mimir = 범용 문서/지식 플랫폼 + API/MCP 기반 인프라**
>
> 모든 문서 유형을 Generic 구조로 수용하고, DocumentType 플러그인이 타입별 동작을 정의한다.
> API-first 설계로 UI·외부 시스템·**외부 AI 에이전트(MCP)** 모두 동일한 정본을 통해 플랫폼을 활용한다.

---

## S3 Phase 4 (2026-04-17 ~ 2026-04-28) 주요 변경 요약

S3 Phase 4 — **Agent-Facing Interface 통합**:

1. **MCP 서버 표면 (FG 4-0/4-1)** — MCP 2025-11-25 호환 dispatcher (`POST /api/v1/mcp`), 9 도구, mimir:// URI 4 패턴, 표준 envelope (`source` / `scope` / `audit` / `detected_risks`)
2. **읽기 도구 확장 (FG 4-2)** — `read_document_render`, `resolve_document_reference`, `read_annotations`, `search_nodes`
3. **Citation 5중 검증 (FG 4-3)** — 5-tuple + `citation_basis` (`node_content` / `rendered_text`), `verify_citation` 5중 검사 (`exists` / `pinned` / `hash_matches` / `quoted_text_in_content` / `span_valid`). citations 테이블 부재로 [Disagreement Record](../disagreements/2026-04-28-fg43-citations-table-absence.md) 적용 — Pydantic 모델 + JSONB 임베드 표면으로 적응
4. **Tool ACL (FG 4-4)** — `scope_profiles.allowed_tools` (JSONB) 추가, `agent_principal.can_call_tool()` 두 차원 검사 (manifest exposure ∧ profile allow-list)
5. **매니페스트 정책 (FG 4-5)** — 9 manifest 필드 (`risk_tier`, `maturity`, `status`, `exposure_policy`, `default_enabled`, `requires`, `preferred_use`, `policy_profile`, `streaming_supported`), 외부 (`/mcp/manifest`) vs 운영자 (`/admin/mcp/manifest`) view 분리, drift 게이트
6. **첫 L2 쓰기 도구 (FG 4-6)** — `save_draft` 4 사전 조건 적용 (idempotency / `requires_human_approval=True` 강제 / `compute_draft_impact` preview / 4 종 audit chain)

**5 절대 규칙**:
- **R1**: L4 도구 MCP 영구 금지 (`exposure_policy="never"` 단일 결정점)
- **R2**: ACL 단일 결정점 (`ScopeProfile.allowed_tools` ∩ manifest exposure)
- **R3**: pinned citation 강제 (`latest` 단독 — URI 빌더/`verify_citation`/`resolve_document_reference` 3 곳에서 차단)
- **R4**: `instruction_authority="none"` (Literal 강제 — agent 출력의 권한 상승 차단)
- **R5**: REST/MCP 1:1 복제 금지 (MCP는 read-or-propose; L3/L4는 REST 전용)
- **R6**: `detected_risks` 는 차단이 아닌 경고 (Constitution Meta-2 사용자 의도 우선)

---

## 변경 이력

| 일자 | 버전 | 주요 변경 |
|---|---|---|
| 2026-04-10 | 1.0 | Phase 13 종결 — 기술서 폴더 8 개 신설 |
| 2026-04-28 | 2.0 | S3 Phase 4 종결 — **MCP 기술서/** 폴더 5 docs 신설, DB 스키마에 S3 신규 테이블 (`scope_profiles`, `agent_proposals`, `tags`, `document_tags`, `collections`, `collection_documents`, `folders`, `document_folders`, `annotations`, `annotation_mentions`, `notifications`, `prompt_templates`) DDL 추가, 보안 모델에 §11 Scope Profile + §12 MCP R1~R6 + §13 도구 등급 L0~L4 + 4 사전 조건 추가 |
