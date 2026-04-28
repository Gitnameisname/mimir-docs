# S3 Phase 4 — 외부 Chatbot 연결(MCP 표면) 개선

**작성일**: 2026-04-27
**Phase 상태**: 초안 — 사용자 합의(`uploads/Mimir API : MCP 운영 논의 요약`, 2026-04-27) 직후 킥오프 문서
**선행 조건**: S3 Phase 3 게이트 통과 (Contributors 패널 + 인라인 주석)
**후행 조건**: S3 1라운드 종결 또는 S3 2라운드 진입 판단
**Handoff Level**: `extended` — 보안(prompt injection)·정책(도구 등급화)·외부 I/O(에이전트 인터페이스)·DB(citation_basis 컬럼) 모두 포함
**Approver**: `@최철균` (P1 — Meta-1, 제45조)

---

## 0. 명명 정정 (Phase 번호의 재정의)

본 Phase는 `Season3_개발계획.md §3.1`이 "이월"로 적시했던 **외부 SaaS 커넥터(Confluence/Notion/SharePoint Import)** 와는 **다른 작업**이다. 사용자 요청(2026-04-27)에 따라 S3 Phase 4를 **"외부 Chatbot 연결(MCP 표면) 개선"** 으로 재정의하며, 외부 SaaS 커넥터는 별도 Phase(S3 2라운드 또는 S4)로 이월한다.

또한 코드베이스 일부(예: `backend/app/mcp/tools.py` 모듈 docstring의 "Phase 4 FG4.1")는 **S2의 Phase 4** 를 가리킨다. 본 문서가 다루는 것은 **S3 Phase 4** 이며, 두 Phase 번호는 시즌이 다르므로 충돌하지 않는다. 산출물 경로(`docs/개발문서/S3/phase4/`)와 task 식별자(`task4-N`)로 구분된다.

---

## 1. Phase 개요

### 1.1 목적

외부 Chatbot(에이전트)이 Mimir 문서를 **안전하게 검색·인용·검증**할 수 있도록 MCP 표면을 표준화한다. 업로드 요약(2026-04-27)에서 합의된 다음 4축을 실행 가능한 형태로 정착시킨다.

1. **REST/MCP 역할 분리** — REST는 시스템 운영 API(상태 변화 권한), MCP는 에이전트 전용 읽기 인터페이스. 1:1 복제 금지.
2. **도구 등급화(L0~L4)** — 각 도구는 위험도 등급과 manifest(`risk_tier`/`maturity`/`status`)를 부착. **L4 도구(reindex/change_schema/delete_document)는 MCP 노출 금지.**
3. **응답 메타데이터 표준** — 모든 read 도구·리소스가 `content_role`/`instruction_authority`/`trust_level`/`detected_risks`/`source.uri`를 envelope로 반환. AI가 문서 내용을 명령으로 오해하지 않도록 한다.
4. **인용 정밀화** — `citation_basis` 필수, `latest` 인용 거부(pinned 강제), span·hash·존재성 5중 검증.

### 1.2 절대 규칙 (반복 강조)

S3 1라운드 공통 규칙(저장 모델 단일성·뷰 ≠ 권한·node_id 안정성·에이전트 동등성)은 그대로 유지하면서, 본 Phase에서는 다음을 **추가 절대 규칙**으로 둔다.

- **R1. L4 도구 MCP 금지** — `risk_tier=L4` 또는 `maturity=forbidden` 도구는 MCP `tools/list`에 출현하지 않고, `tools/call`이 강제 거부한다. 이 규칙의 회귀 테스트(FG 4-4)는 CI 게이트로 등록한다.
- **R2. ACL 단일 결정점** — Scope Profile만이 ACL을 결정한다(S1 ④, S2 ⑥). MCP 도구는 자체 권한 계층을 도입하지 않으며, REST와 동일한 도메인 코어(권한·검증·정책)를 호출해야 한다.
- **R3. pinned citation 강제** — 인용·감사 로그에는 `latest`가 남으면 안 된다. `verify_citation`은 `version_id`가 `latest`로 들어오면 거부하거나, REST 측에서 미리 `vN` 으로 resolve된 값만을 받는다.
- **R4. 에이전트 응답에 명령 권한 부여 금지** — read 응답의 `instruction_authority`는 **항상 `none`** 으로 시작한다. 어떠한 `system`/`user` 권한도 자동 부여하지 않는다(헌법 제17·18조 — Untrusted / Provenance).
- **R5. REST와 1:1 복제 금지** — MCP 도구는 에이전트의 작업 의도(검색·검증·노드 탐색·렌더 조회)를 단위로 설계한다. REST 엔드포인트의 wrapper로 만들지 않는다.

이 5개 규칙은 본 Phase 모든 FG의 검수보고서에 **"R1~R5 준수 확인"** 섹션으로 의무 기재한다.

### 1.3 선행 조건

- S3 Phase 3 완료(또는 사용자가 명시적으로 Phase 3 완료 전 본 Phase 착수를 승인). Phase 3의 `actor_type=agent` 일관성·인라인 주석 좌표는 본 Phase의 메타데이터 표준에 영향이 없으므로, 병행 가능 여부는 Pre-flight에서 판단한다.
- 기존 MCP 자산: `backend/app/mcp/{tools.py, resources.py, prompts.py, scope_filter.py, errors.py}` + `backend/app/api/v1/mcp_router.py` + `backend/app/schemas/mcp.py` (S2 종결 자산) — Pre-flight에서 인벤토리 확정.
- 헌법 제5조(Agent-Facing Contract), 제16~18조(Trust/Untrusted/Provenance), 제29조(Handoff Metadata Scales with Risk).

### 1.4 기대 결과

- 모든 MCP read 도구·리소스가 표준 envelope을 반환한다.
- 신규 read 도구 3종(`search_nodes`, `read_document_render`, `resolve_document_reference`)이 외부 Chatbot에 노출된다.
- `verify_citation`이 5중 검증으로 강화되며, `citation_basis`가 필수 입력이 된다.
- REST/MCP contract drift 4종 테스트가 CI 게이트로 등록되어 지속적으로 일치성을 보호한다.
- 도구 manifest(`risk_tier`/`maturity`/`status`)가 모든 노출/비노출 도구에 부착되어, Tool 등급화 정책이 코드로 강제된다.

### 1.5 헌법/원칙 정합성

| 원칙 | 본 Phase에서의 적용 |
|------|---------------------|
| S1 ① API-first | 모든 신규 도구는 도메인 코어(REST와 동일 함수)를 호출. MCP는 외피일 뿐. |
| S1 ④ 권한은 모든 계층에 적용 | Scope Profile ACL을 search_nodes/render/resolve 모두 통과. |
| S2 ⑤ AI는 1급 사용자 | 응답 메타데이터로 AI가 문서를 안전하게 다룰 수 있게 함 — "1급 사용자"의 안전성 확장. |
| S2 ⑥ 접근 범위 하드코딩 금지 | 도구 등급화는 manifest로, ACL은 Scope Profile로 — 코드에 권한 매핑 인라인 금지. |
| S2 ⑦ 폐쇄망 동등성 | 신규 도구의 외부 의존(예: LLM 기반 의미 매칭) off 가능. resolve_document_reference의 의미 매칭 단계는 폐쇄망에서 FTS fallback으로 degrade. |
| 헌법 제5조 | 모든 신규 도구는 Agent-Facing Contract(description/input·output schema/권한/부작용/실패 코드)로 정의. |
| 헌법 제17·18조 | content_role/instruction_authority/trust_level로 Untrusted/Provenance 명시. |
| 헌법 제29조 | extended Handoff Level — 설계→구현→리뷰→human approval. |

---

## 2. Feature Group (FG) 요약

본 Phase는 **1라운드 5개 FG(4-0 ~ 4-4)** 로 구성한다. 후속 2개(4-5, 4-6)는 **이월/조건부**이며, 1라운드 게이트 통과 후 별도 Pre-flight를 거쳐 진행 여부를 결정한다.

### 2.1 1라운드 (즉시 착수 대상)

| FG | 제목 | 핵심 | 산출물 |
|----|------|------|--------|
| **4-0** | Pre-flight + 도구 등급화 + manifest 표준 + **ScopeProfile tool-level ACL** | 기존 MCP 도구 인벤토리, L0~L4 매핑, manifest 최소 필드(`risk_tier`/`maturity`/`status`) 부착, L4 도구 차단 회귀, **`ScopeProfile.allowed_tools` 컬럼 + `AgentPrincipal.can_read` 헬퍼 + 모든 MCP tool 진입점 일괄 게이트 (S3 P3 정적 리뷰 P1 흡수, 2026-04-27)** | task4-0 + Pre-flight_실측.md + 인벤토리 표 + Alembic `s3_p4_scope_profile_allow_tools` 마이그레이션 |
| **4-1** | MCP read 응답 메타데이터 표준 | `content_role`/`instruction_authority`/`trust_level`/`detected_risks`/`source.uri` envelope. 기존 도구(search_documents/fetch_node) 회귀 적용. `mimir://` URI 스키마 확정. | task4-1 + 검수·보안 보고서 |
| **4-2** | 신규 read 도구 3종 추가 | `search_nodes` (노드 단위 검색) + `read_document_render` (렌더링 텍스트 조회) + `resolve_document_reference` (자연어 → document_id/version_ref 정규화 + confidence + needs_disambiguation) | task4-2 + 검수·보안 보고서 + AI 품질 평가 보고서(resolve의 confidence 정밀도) |
| **4-3** | verify_citation 강화 | 5중 검증(존재/pinned/hash/포함/span) + `citation_basis` 필수(node_content/rendered_text) + `node_content_hash` vs `render_hash` 분기 + Alembic 마이그레이션(`citations.citation_basis` 신설) | task4-3 + 검수·보안 보고서 + 데이터 마이그레이션 가이드 |
| **4-4** | REST/MCP Contract Drift 테스트 | 4종 테스트 — (1) 동일 노드 결과, (2) latest 해석 동일성, (3) 비노출 도구 검증(L4), (4) pinned citation 강제. CI 게이트 등록. | task4-4 + 검수 보고서 + CI 결과 아티팩트 |

### 2.2 이월 / 조건부 후속 (Phase 4 1라운드 게이트 통과 후 별도 결정)

| FG | 제목 | 진행 조건 | 작업지시서 |
|----|------|---------|---------|
| **4-5** | Capability manifest 확장 (`default_enabled`/`requires`/`preferred_use`/`policy_profile`/`streaming_supported`) | 1라운드 R1~R5 회귀 녹색. 사용자 승인 시 즉시 진행. | [task4-5.md](작업지시서/task4-5.md) (2026-04-28 작성) |
| **4-6** | L2 draft 쓰기 도구 실험적 노출 | idempotency·human approval·impact preview·감사 로그 4종 모두 갖춰진 후에만. **L3/L4는 본 Phase 범위에서 영구 제외**. **별도 P1 게이트** (MCP 표면 최초 쓰기 도구). | [task4-6.md](작업지시서/task4-6.md) (2026-04-28 작성) |

> ⚠️ FG 4-5/4-6는 본 Phase 1라운드 게이트의 일부가 **아니다**. 1라운드 종결 시 별도 Pre-flight + 사용자 합의를 거쳐 진행 여부를 결정한다.

---

## 3. 데이터 / 스키마 모델 (개요)

본 Phase는 신규 테이블을 **최소화**한다. 대부분의 변경은 응답 envelope(코드)·도구 정의(코드)·테스트(코드)이며, 영구 스키마 변경은 다음 1건만 수행한다.

### 3.1 신규 컬럼 / 테이블

- ~~**`citations.citation_basis`** (Alembic revision `s3_p4_citation_basis`)~~ — **N/A** (Disagreement Record 2026-04-28). citations 테이블 부재로 Alembic 미적용. citation_basis 는 `app.schemas.citation.Citation` Pydantic 모델 필드로만 도입 (in-memory). 정본 영구 저장 모델은 별 라운드.
- **(선택, FG 4-3 평가 후 결정)** `versions.render_hash` — 렌더링된 텍스트의 별도 hash. 현재 `versions.content_hash`는 ProseMirror 트리 기준이므로, rendered_text 기반 인용 검증을 위해 별도 hash가 필요할 수 있다. Pre-flight에서 결정.
- **`scope_profiles.allowed_tools`** (Alembic revision `s3_p4_scope_profile_allow_tools`, FG 4-0 §2.1.6) — `JSONB NOT NULL DEFAULT '[]'::jsonb`. 기존 행 backfill: 빈 배열 (default-deny). 운영자가 Admin UI 로 도구 등록 전까지 에이전트 도구 호출 모두 거부 — **운영 절차 의존**. S3 Phase 3 정적 리뷰 P1 흡수 (task3-3.md §[129,223–225,318]).

### 3.2 변경 없음 (확인 사항)

- `documents`, `versions`, `nodes`, `scope_profiles` 등 핵심 테이블은 본 Phase에서 변경하지 않는다.
- `actor_type=agent` 감사 로그 구조는 S2 종결 상태를 그대로 사용한다.

### 3.3 Manifest 표준 (코드 상수)

`backend/app/schemas/mcp.py`의 `TOOL_SCHEMAS`에 다음 필드를 부착한다(스키마는 코드, 영구 저장 아님):

```python
{
    "name": "...",
    "risk_tier": "L0" | "L1" | "L2" | "L3" | "L4",
    "maturity": "stable" | "beta" | "experimental" | "disabled" | "forbidden",
    "status": "enabled" | "disabled" | "not_exposed",
    "exposure_policy": "MCP_ENABLED" | "MCP_DISABLED" | "REST_ADMIN_ONLY",
    # 기존 필드 (description / inputSchema / authentication) 유지
}
```

---

## 4. 산출물 규약

`Season3_개발계획.md §4.2` 규약을 따른다. 본 Phase 한정 추가:

| 산출물 | FG 4-0 | FG 4-1 | FG 4-2 | FG 4-3 | FG 4-4 |
|--------|--------|--------|--------|--------|--------|
| 작업지시서 (`task4-N.md`) | ✅ | ✅ | ✅ | ✅ | ✅ |
| 검수보고서 (R1~R5 준수 확인 섹션 의무) | ✅ | ✅ | ✅ | ✅ | ✅ |
| 보안취약점검사보고서 | ✅ | ✅ | ✅ | ✅ | ✅ |
| AI 품질 평가 보고서 (Faithfulness/Citation-present 영향) | — | — | ✅ (resolve confidence) | ✅ (citation 강화) | — |
| 마이그레이션 가이드 | — | — | — | ✅ (citation_basis backfill) | — |
| CI 게이트 등록 증거 | — | — | — | — | ✅ |

UI 요소가 거의 없는 Phase이므로 UI 디자인 리뷰는 **FG당 0회 허용**(프로젝트 §5.2 완화 적용). 단, 관리자 화면에서 도구 등급/상태를 표시하는 화면을 추가한다면 해당 FG 한정 1회 이상 리뷰.

---

## 5. 게이트 / 완료 기준

### 5.1 1라운드 완료 기준 (FG 4-0 ~ 4-4)

- [x] **R1 (L4 차단)** — FG 4-4 contract drift 테스트 11건 ✅ (TestNoForbiddenInToolSchemas + TestFakeL4ToolBlocked + TestRestAdminOnlyToolsNotInToolSchemas) + FG 4-0 16건 = **27건 ✅**
- [x] **R2 (ACL 단일)** — FG 4-0 §2.1.6 ScopeProfile.allowed_tools + FG 4-2 신규 3 도구 진입점 게이트 ✅
- [x] **R3 (pinned)** — FG 4-3 verify_citation `latest` 거부 + FG 4-4 contract drift 8건 ✅ (citations 테이블 부재 — Disagreement Record 적응안)
- [x] **R4 (instruction_authority=none)** — FG 4-1 envelope `Literal["none"]` 강제 + FG 4-4 contract drift `_build_envelope` 회귀 ✅
- [x] **R5 (REST 1:1 복제 금지)** — FG 4-4 정본 source_text / hash / VersionsRepository 공유 입증 ✅ (TestSameNodeContent + TestLatestResolveSharedRepository)
- [x] ~~Alembic revision `s3_p4_citation_basis` 적용 + backfill~~ — **Disagreement Record 적응안 (2026-04-28)**: citations 테이블 부재. Pydantic 모델만 갱신.
- [x] **Alembic revision `s3_p4_scope_profile_allow_tools` 적용 + 운영자 Admin UI 도구 등록 완료** (FG 4-0 §2.1.6 — S3 P3 정적 리뷰 P1 흡수) ✅ (2026-04-28 — L0 4개 bootstrap)
- [x] **모든 MCP `tool_*` 진입점에 `_check_tool_allowed` 게이트 적용 + 회귀 ≥ 6 시나리오 녹색** ✅ (FG 4-0 — 5 도구 + FG 4-2 신규 3 도구 = 8 도구 모두 적용)
- [x] **`종결회고.md §4.1 #11` / `FG3-3 §5 #9` 잔존부채 항목 종결 마크** ✅ (2026-04-28)
- [x] `TOOL_SCHEMAS` 모든 항목에 `risk_tier`/`maturity`/`status` 부착 ✅ (FG 4-0 + FG 4-2 — 8 도구 모두)
- [x] Contract drift 4종 테스트 작성 — CI 게이트 등록은 운영자 후속 ✅ (FG 4-4)
- [ ] 각 FG의 검수·보안 보고서 제출
- [ ] (FG 4-2) 폐쇄망 모드(`MIMIR_OFFLINE=1`)에서 resolve_document_reference가 FTS fallback으로 동작함을 입증

### 5.2 회귀 게이트 (이전 Phase 보호)

- [ ] Phase 0~3에서 통과한 모든 회귀 테스트 녹색 유지 (특히 Phase 1의 node_id 안정성 / Phase 2의 뷰 ≠ 권한 / Phase 3의 인라인 주석 좌표 보존)
- [ ] pytest 전체 녹색 (현재 베이스라인 — 운영자 macOS 실측치 기반으로 갱신)
- [ ] `coverage_baseline.py --check` 게이트 통과 (services ≥ 80% / repositories ≥ 80%)
- [ ] node:test 녹색

### 5.3 1라운드 종결 정의

위 5.1 + 5.2 모두 통과 시 **S3 Phase 4 1라운드 공식 종결**. 1라운드 종결 후 FG 4-5/4-6 진행 여부는 사용자 합의로 별도 결정한다.

### 5.4 1라운드 종결 상태 (2026-04-28)

| 게이트 | 상태 |
|---|---|
| §5.1 R1~R5 자동 보호 | ✅ 5/5 통과 (FG 4-4 contract drift 매트릭스) |
| §5.1 Alembic + 도구 등록 + 회귀 + 잔존부채 + manifest + drift 테스트 | ✅ 6/6 통과 (위 §5.1 체크리스트) |
| §5.2 회귀 게이트 | ✅ 전체 회귀 3495 / 0 fail (단, services/repositories 80%, overall 67% — Phase1 베이스라인 유지) |
| 외부 Codex 정적 리뷰 | ⏳ S3 전체 종료 시점으로 이연 (사용자 결정 2026-04-28) |
| AI 품질 실 measurement | ⏳ 운영자 후속 (FG 4-2 골든셋 / FG 4-3 회귀 측정) |
| CI 게이트 등록 | ⏳ 운영자 후속 (FG 4-4 §3) |

**Phase 4 1라운드 코드 측 종결 완료**. 운영자 후속 (Codex / AI 품질 / CI 게이트) 대기 중. FG 4-5/4-6 진입 여부는 별도 사용자 합의.

---

## 6. 범위 밖 / 이월

### 6.1 본 Phase 1라운드 명시적 제외

- **Capability manifest 확장 필드**(`default_enabled`/`requires`/`preferred_use`/`policy_profile`/`streaming_supported`) — FG 4-5(이월)
- **L2 이상의 쓰기 도구** — FG 4-6(이월·조건부)
- **L3/L4 쓰기 도구** — 본 Phase 영구 제외(헌법·R1)
- **외부 SaaS 커넥터(Confluence/Notion/SharePoint)** — Season3_개발계획.md의 원안 Phase 4 작업. S3 2라운드 또는 S4로 별도 이월
- **MCP 클라이언트 SDK 배포** — 본 Phase는 서버 측 표면 정비. 클라이언트 SDK는 별도 작업
- **에이전트별 Rate Limit 정밀화** — `mcp_router.py`의 `_AGENT_RATE_LIMITS`는 FG5.3 이월(S2 결정) 그대로 유지

### 6.2 보류 결정 사항 (FG 진행 중 결정)

- `versions.render_hash` 신설 여부 → FG 4-3 Pre-flight에서 결정
- `resolve_document_reference`의 의미 매칭 LLM 모델 선택 → FG 4-2 Pre-flight에서 결정 (폐쇄망 fallback은 FTS로 고정)
- `tools/list` 응답이 `risk_tier`를 외부에 노출할지 여부 → FG 4-0에서 결정 (외부 노출 시 운영 정보 누출 위험 vs 클라이언트 결정 도움)

---

## 7. 리스크

| # | 리스크 | 영향 | 대응 방향 |
|---|--------|------|----------|
| R-01 | L4 도구가 코드 변경 중 실수로 MCP에 노출됨 | 보안 사고(데이터 삭제·인덱스 파괴) | FG 4-4 contract drift 테스트의 (3) 비노출 도구 검증을 CI 게이트로 등록 + L4 도구 정의에 `not_exposed` 강제 + `tool_call` 디스패처에서 manifest의 `status`로 라우팅 |
| R-02 | 응답 메타데이터 추가가 기존 클라이언트(외부 Chatbot) 호환을 깸 | 외부 통합 회귀 | envelope 추가는 **순수 추가형**(기존 필드 유지). 기존 클라이언트가 envelope을 무시해도 동작. 변경 후 기존 통합 테스트 회귀 수행 |
| R-03 | `resolve_document_reference`가 잘못된 문서를 자신있게 반환(low confidence를 high로 표시) | 인용 오류 → 환각 | confidence 임계값(예: 0.85)을 기준으로 `needs_disambiguation` 강제. AI 품질 평가에 정밀도/재현율 측정 의무 |
| R-04 | `citation_basis` 필수화로 기존 인용 데이터가 무효 | 기존 답변 검증 실패 | Alembic backfill로 모든 기존 행을 `node_content`로 채움. backfill 후 NOT NULL 추가. 단계적 마이그레이션 |
| R-05 | pinned 강제로 인해 "최신 답변" UX가 깨짐 | 사용자가 항상 vN을 입력해야 함 | REST 측에서 `latest` → `vN` resolve를 자동 수행한 후 인용을 저장. MCP 외부 인터페이스에서는 사용자가 `latest`를 보낼 수 있지만, 서버 저장 시점에 resolve 강제 |
| R-06 | prompt injection 탐지 false positive로 정상 문서가 거부됨 | 사용자 차단 | `detected_risks`는 **차단이 아닌 경고**. 응답은 정상 반환하되 risk 목록만 첨부. 차단 정책은 클라이언트가 결정 |
| R-07 | manifest 필드가 코드/문서 사이에 drift | L4 도구 누락 가능성 | manifest는 `TOOL_SCHEMAS` 코드 상수가 단일 정본. 문서는 코드에서 자동 추출(스크립트) — FG 4-0에서 추출 스크립트 작성 |
| R-08 | REST/MCP 도메인 코어 공유가 깨져 권한 이중 구현 | ACL 우회 가능성 | 신규 도구 구현 시 반드시 기존 service 함수 호출. 검수보고서에서 함수-호출 그래프로 증명. R5 회귀 테스트로 protect |
| R-09 | search_nodes의 결과량이 토큰 폭발을 일으킴 | 에이전트 비용 폭증 | `top_k` 상한(50) + 응답에 `truncated_at` 명시. envelope에 결과 수 포함 |
| R-10 | 폐쇄망에서 의미 매칭이 동작하지 않아 resolve가 무용지물 | 폐쇄망 환경 회귀 | FG 4-2의 폐쇄망 fallback(FTS) 회귀 테스트 의무. `MIMIR_OFFLINE=1` 환경변수로 분기 |

---

## 8. 협업 / 라우팅

`AGENT_MODE.md` §3.3 (extended) 적용:

- **Design**: Claude (본 문서 + task4-0~4-4 작성)
- **Implementation**: Codex (FG별 코드 변경)
- **Review**: Claude (Codex 산출물 리뷰, R1~R5 준수 확인)
- **Human Approval**: `@최철균` — 다음 시점에 의무
  - Phase 4 개발계획서 승인 (본 문서)
  - Alembic revision `s3_p4_citation_basis` 적용 직전
  - **Alembic revision `s3_p4_scope_profile_allow_tools` 적용 직전** (FG 4-0 §2.1.6 — default-deny 라 운영자 도구 등록 작업과 같은 운영 창에 배치 필수)
  - 1라운드 종결 선언 직전
  - FG 4-5/4-6 진입 결정 시점
- **Validation**: unit + contract + policy + eval + recovery

Disagreement Record(헌법 제34조)는 본 Phase 진행 중 어느 쪽이든 발견 시 즉시 작성한다. 특히 다음 결정점에서 이견 가능성이 높다 — manifest의 외부 노출 범위 / render_hash 신설 여부 / resolve confidence 임계값.

---

## 9. 참조

### 9.1 본 Phase 직접 입력
- `uploads/Mimir API : MCP 운영 논의 요약` (2026-04-27, 사용자 합의 정본)

### 9.2 헌법·운영 문서
- `CONSTITUTION.md` 제5·16·17·18·26·27·29·45조
- `CLAUDE.md` §1.1, §3, §6
- `AGENT_MODE.md` §3.3 (extended), §5 (P1 Approval Gate)
- `AUTOMATION.md` (Dispatcher 추천 해석)

### 9.3 S3 컨텍스트
- `docs/개발문서/S3/Season3_개발계획.md` §3 (Phase 구조), §6 (게이트 정의)
- `docs/개발문서/S3/phase0/` ~ `phase3/` (선행 게이트)
- `docs/개발문서/S2/` (MCP S2 종결 상태 — `tools.py`/`mcp_router.py`/`schemas/mcp.py` 기반)

### 9.4 코드베이스 (현 상태)
- `backend/app/mcp/` — `tools.py` (search/fetch/verify/vectorization_status), `resources.py`, `prompts.py`, `scope_filter.py`, `errors.py`
- `backend/app/api/v1/mcp_router.py` — HTTP 표면 (`/mcp/initialize`, `/mcp/tools/call`, `/mcp/resources`, `/mcp/prompts`)
- `backend/app/schemas/mcp.py` — `TOOL_SCHEMAS`, request/response 모델
- `backend/app/security/prompt_injection.py` — `PromptInjectionDetector`, `ContentDirectiveSeparator`
- `backend/app/services/render_service.py` — REST `/documents/{id}/render` 백엔드 (FG 4-2 read_document_render 도메인 코어)

---

## 10. 변경 이력

| 일자 | 변경 | 작성자 |
|------|------|--------|
| 2026-04-27 | 초안 작성. 사용자 업로드 합의(2026-04-27) 기반. FG 4-0~4-4 1라운드 + 4-5/4-6 이월 구조 확정. | Claude (Design Actor) |
| 2026-04-28 | FG 4-0 에 ScopeProfile tool-level ACL (S3 Phase 3 Codex 정적 리뷰 P1) 흡수. `scope_profiles.allowed_tools` 컬럼 + `AgentPrincipal.can_read` + 모든 MCP tool 진입점 일괄 게이트 + Admin UI 토글. task4-0 §2.1.6 신설. 5.1 게이트 4건 추가. R2 의 tool-level 차원 종결. | Claude (Design Actor) |
| 2026-04-28 | **FG 4-0 종결** — alembic 적용 + L0 4개 도구 bootstrap 등록 + 운영자 후속 1~4·6 완료. 외부 Codex 정적 리뷰는 S3 종결 시점으로 이연. | Claude |
| 2026-04-28 | **FG 4-1 종결** — read 응답 envelope 표준 (MCPReadEnvelope / MCPItemEnvelope / MCPSourceRef / MCPDetectedRisk) + mimir:// URI 4 패턴 + risk_mapper + 5 도구 envelope 적용 + 항목별 envelope. 53 신규 회귀. R3 (pinned) / R4 (instruction_authority=none) / R6 (경고만) 코드 강제. | Claude |
| 2026-04-28 | **FG 4-2 종결** — 신규 read 도구 3종 (search_nodes / read_document_render / resolve_document_reference) + document_resolver_service (5단계 + 폐쇄망 fallback) + AI 품질 골든셋 50 케이스. TOOL_SCHEMAS 5 → 8. 37 신규 회귀 (누적 MCP 149/149). R1~R6 모두 준수. AI 품질 실 measurement 는 운영자 후속 (Phase 0 FG 0-4 패턴). | Claude |
| 2026-04-28 | **FG 4-3 종결** (Disagreement Record 적응안) — verify_citation 5중 검증 + citation_basis 분기 + R3 강제 (`latest` 도구 진입점 거부). citations 테이블 부재로 Alembic 미적용 (`docs/disagreements/2026-04-28-fg43-citations-table-absence.md`). 27 신규 회귀 (누적 MCP 176/176). 백워드 호환 필드 보존. | Claude |
| 2026-04-28 | **FG 4-4 종결 — Phase 4 1라운드 게이트 모두 통과** — REST/MCP Contract Drift 4종 + manifest drift 게이트 (37 신규 회귀). R1/R3/R5 자동 보호 입증 (가짜 L4 차단 + DB 'latest' 부재 + 정본 공유). 누적 MCP 213/213. 전체 3495 / 0 fail. CI 게이트 YAML 스니펫은 검수보고서 §3 — 운영자 후속 적용. **Phase 4 1라운드 공식 종결 후보**. | Claude |
| 2026-04-28 | **이월 작업지시서 작성** — task4-5 (Capability manifest 확장: 5 신규 필드 + admin manifest endpoint + Admin UI 동적 fetch) + task4-6 (L2 draft 쓰기 도구: 4 사전 조건 + tool_save_draft + write envelope + L3/L4 영구 제외). 진행 여부는 사용자 합의 후 결정. | Claude |
| 2026-04-28 | **FG 4-5 종결** — Capability manifest 5 신규 필드 (`default_enabled` / `requires` / `preferred_use` / `policy_profile` / `streaming_supported`) + 헬퍼 2종 + `GET /admin/mcp/manifest` endpoint + `ScopeProfileRepository.create(use_defaults)` 옵션 + Admin UI 동적 fetch (`KNOWN_MCP_TOOLS` 정적 상수 제거). dump_mcp_manifest schema_version 1.0 → 1.1. 27 신규 회귀 (누적 MCP 240/240). R1~R6 영향 0 — 메타데이터 표면화만. | Claude |
| 2026-04-28 | **FG 4-6 종결** (별도 P1 게이트 — MCP 표면 최초 쓰기 도구) — L2 `save_draft` 도구 + 4 사전 조건 (idempotency / human approval / impact preview / 감사 4종). Alembic `s3_p4_agent_prop_idempotency` (agent_proposals.idempotency_key + UNIQUE INDEX). MCPWriteEnvelope 신설 (read 와 분리). TOOL_SCHEMAS 9 도구. L3/L4 영구 제외 회귀 강화. 33 신규 회귀 (누적 MCP 310/310). 자동 머지 0 — `requires_human_approval=True` 강제. | Claude |

---

*작성: 2026-04-27 | 외부 Chatbot 연결 표면(MCP) 개선 — S3 Phase 4 1라운드 킥오프 문서*
