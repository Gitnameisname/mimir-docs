# Phase 1 Pre-flight 실측 보고서

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-24 |
| 목적 | Phase 1 작업지시서 작성을 위한 현황 실측. 저장 경로·content_snapshot 소비자·MCP 쓰기 경로·프런트 에디터 스택 확인 |
| 근거 | 2026-04-24 기준 `mimir/backend`, `mimir/frontend` 트리 직접 grep/read |
| 결정 반영 | D1 = content_snapshot 단일 정본 (사용자 승인 2026-04-24) / D2 = TipTap + 커스텀 node_id extension (사용자 승인 2026-04-24) |

---

## 1. 핵심 결론 (먼저 요약)

1. **`save_draft` (PUT) 와 `save_draft_nodes` (PATCH) 는 "단순 중복" 이 아니라 서로 다른 테이블에 쓰는 이중 경로다.**
   - `PUT /documents/{id}/draft` → `versions.content_snapshot` 갱신. `nodes` 테이블 미터치.
   - `PATCH /documents/{id}/versions/{vid}/draft` → `nodes` 테이블 교체. `content_snapshot` 미터치.
2. **프런트엔드 에디터(`DocumentEditPage.tsx`)는 PATCH(`saveDraft`→save_draft_nodes) 경로만 사용한다.** PUT 경로 호출자는 프런트 코드 전수 검색상 없음.
3. 따라서 **편집 직후 `content_snapshot` 은 항상 stale** 이다. `render_service` 가 `content_snapshot` 을 읽기 때문에 편집본과 렌더 결과가 어긋날 수 있다. 이것이 BUG-02 의 구체적 형태.
4. **`vectorization_service` 는 nodes 우선, 없으면 content_snapshot 에서 파싱해 nodes 로 역 주입**한다. 덕분에 지금까지 RAG 는 "살아있는" nodes 를 참조해 왔으나, 두 포맷의 진실 기준이 경로별로 달라 불변식 위반이 이미 상시 발생 중.
5. **TipTap (`@tiptap/*`) 은 package.json 에 설치돼 있으나 실제 에디터 코드에서 import 0 건.** 현재 에디터는 `DocumentNode[]` 배열을 `<input>`/`<textarea>` 로 편집하는 DIY 블록 편집기.
6. **ProseMirror 라는 명시적 의존성은 frontend 에 없다.** (`@tiptap/starter-kit` 이 내부적으로 ProseMirror 를 래핑하지만 최상단에서는 TipTap 만 노출.)

---

## 2. 백엔드 Draft 저장 경로 실측

### 2.1 라우트 수준 (app/api/v1/documents.py)

| 엔드포인트 | 메서드 | 서비스 호출 | 쓰는 테이블 | 받는 바디 |
|-----------|-------|-------------|-----------|----------|
| `/documents/{document_id}/draft` | PUT | `draft_service.save_draft` | `versions` (content_snapshot) | `DraftSaveRequest` — `content_snapshot: dict` 포함 |
| `/documents/{document_id}/versions/{version_id}/draft` | PATCH | `draft_service.save_draft_nodes` | `nodes` (replace) + `versions.title_snapshot/summary_snapshot/label/change_summary` | `DraftNodeSaveRequest` — `nodes: list[DraftNodeItem]` |

둘 다 감사 이벤트(`draft.updated`, `draft.nodes_saved`) 는 발화되나, **상호 배타적** 이다. 호출자 관점에서 "둘 중 하나" 를 선택해야 한다.

### 2.2 서비스 수준 (app/services/draft_service.py)

**`save_draft` (L124~230)**:
- `content_snapshot=request.content_snapshot` 를 `versions_repository.create`/`update_content` 에 전달
- 이때 `nodes` 테이블은 건드리지 않음 → **content_snapshot 만 갱신**

**`save_draft_nodes` (L559~647)**:
- `nodes_repository.replace_for_version(conn, version_id, node_items)` — 기존 nodes DELETE + 새 nodes INSERT
- `versions_repository.update_content(...)` 호출하나 `content_snapshot=None`(기본값) → **nodes 만 갱신**
- 즉 프런트가 편집 후 PATCH 로 저장하면 DB 상태는 `nodes = {최신}, content_snapshot = {구판}` 로 분기.

### 2.3 content_snapshot 의 파이프라인 소비자

| 소비자 | 읽는 소스 | 비고 |
|-------|---------|------|
| `render_service.py` (L201, 209) | `version.content_snapshot` | **빈 스냅샷이면 빈 문서 반환**. 즉 신규 문서를 nodes 로만 저장하면 렌더에서 빈 문서로 보임 |
| `vectorization_service._fetch_nodes` (L192 → L499~594) | `nodes` 우선, 없으면 `content_snapshot` 을 파싱해 nodes 로 역 주입 | 지금까지 RAG 가 "살아있는" nodes 를 본 이유. 그러나 이는 **fallback** 이므로 주 경로가 아니라는 점에 유의 |
| `diff_service.py` (L773) | 버전 간 `content_snapshot` 비교 SQL | diff 는 content_snapshot 기준 → nodes 편집은 diff 에 반영 안 됨 |
| `agent_proposal_service.py` (L95~121) | `content_snapshot = {"type": "text", "content": content}` 로 자체 포맷 생성 | **ProseMirror 표준 포맷이 아님** — 별도 이슈. FG 1-1 범위에 포함 검토 |
| MCP tools.py (L265, 285, 309) | `content_snapshot=None` / `content_snapshot=snapshot` 은 `VerifyCitationData` 응답 필드. DB 의 `content_snapshot` 과는 이름만 같고 의미가 다름(span_offset 기반 스니펫) | 혼동 주의. 쓰기 경로 아님 |

### 2.4 nodes 의 파이프라인 소비자

| 소비자 | 용도 |
|-------|------|
| `vectorization_service` | 벡터화 입력의 주 경로 |
| `versions_service` / `nodes_service` | 버전 상세/노드 단건 조회 API |
| `diff_service` | L773 주변 보조 |
| `draft_service.save_draft_nodes` | 현재 편집 쓰기 경로 |
| 프런트 `nodesApi.list` | `DocumentEditPage` 초기 로드 + `DocumentDetailPage` 렌더 대체 데이터 |

---

## 3. 프런트엔드 저장 경로 실측

### 3.1 호출자 (grep `versionsApi` / `saveDraft`)

| 위치 | 호출 | 메서드 |
|------|------|-------|
| `features/editor/DocumentEditPage.tsx:72` | `versionsApi.saveDraft(documentId, versionId, { title, nodes })` | PATCH — **save_draft_nodes 경로** |
| `features/versions/VersionsPage.tsx:147` | `versionsApi.list` | GET |
| `features/versions/VersionDetailPage.tsx:21` | `versionsApi.get` | GET |
| `features/documents/DocumentDetailPage.tsx:42` | `versionsApi.getLatest` | GET |
| `features/documents/NewDocumentPage.tsx:30` | `versionsApi.create` | POST (신규 버전) |
| `features/diff/VersionComparePage.tsx:33` | `versionsApi.list` | GET |

**결론**: 쓰기 경로는 `DocumentEditPage` 한 곳이며, **PATCH(nodes) 만 쓰고 있다**. PUT(content_snapshot) 호출자 0건.

### 3.2 versions API 클라이언트 (lib/api/versions.ts)

- `saveDraft` 만 노출. 바디: `{ title?, summary?, label?, change_summary?, nodes? }` → **content_snapshot 필드 자체가 프런트 타입에 없다**.
- 즉 프런트 타입시스템도 "nodes 만이 저장 모델" 이라는 전제로 맞춰져 있다.

### 3.3 에디터 스택

- `DocumentEditPage.tsx` (401 LOC): `DocumentNode[]` state → `<input>` (heading/section), `<textarea>` (paragraph/list/code) DIY 렌더.
- `grep -r "tiptap|prosemirror|ProseMirror"` 결과 frontend/src 내 2 건 — 둘 다 타입 필드 선언(`content_snapshot?: unknown` in proposals.ts) 수준이며 에디터 구현 아님.
- `package.json` dependencies: `@tiptap/extension-placeholder ^3.22.3`, `@tiptap/react ^3.22.3`, `@tiptap/starter-kit ^3.22.3` — 설치됐으나 **미사용**. dead weight.
- 테스트: `node:test` (`tsc -p tsconfig.test.json && node --test "dist-tests/tests/*.test.js"`).

### 3.4 node 모델 (types)

- `DocumentNode` 필드: `id`, `version_id`, `parent_id`, `node_type ∈ {section, paragraph, heading, list, code_block}`, `order`, `title?`, `content?`, `metadata?`.
- backend `DraftNodeItem` (schemas/versions.py L158 근방) 과 대응.
- **`node_id` 는 프런트에서 `crypto.randomUUID()` 로 생성** (`DocumentEditPage.tsx:129`) → 저장 시 그대로 DB 로. Phase 1 에서 이 UUID 가 ProseMirror node attribute 로 이식될 때 **stable 규약이 유지돼야 한다** (FG 1-3).

---

## 4. ProseMirror 스키마 가정 (현재 아직 실재 스키마 없음)

Phase 1 개발계획서 §1.3 은 "`content_snapshot` 루트가 `{type: "document", content: [...]}` 임을 확인" 을 선행조건으로 걸었으나, 본 Pre-flight 단계에서는:

- `backend/app/schemas/versions.py` 의 `content_snapshot` 타입은 `dict[str, Any]` 이며 validator 는 "dict 여야 함 / 크기 상한" 만 검증한다. **구조 validator 없음.**
- 저장된 실 데이터의 루트 shape 은 경로별로 갈라진다:
  - `save_draft` 경로로 들어온 문서: 호출자가 보낸 대로 (이론상 ProseMirror `{type:"document",content:[...]}`).
  - `agent_proposal_service` 경로로 들어온 문서: `{type:"text", content: "문자열"}` — **단일 노드, ProseMirror 표준 아님**.
  - `save_draft_nodes` 경로로만 편집된 문서: `content_snapshot` 은 버전 생성 직후 값 그대로 (대체로 None 또는 `vectorization_service` 가 역 주입한 흔적).

즉 **"단일 ProseMirror 트리" 는 선언일 뿐 실 DB 는 혼재** 상태다. FG 1-1 작업 범위에 **샘플 실측 + 마이그레이션 배치** 를 추가해야 한다 (개발계획서 §2 FG 1-1 마지막 항목 "기존 published 문서의 content_snapshot 정합성 검증 배치").

---

## 5. MCP Tool 쓰기 경로

- `mcp/tools.py` 의 `content_snapshot` 키(L265/285/309)는 모두 `VerifyCitationData` 응답 필드의 의미. DB 의 `versions.content_snapshot` 과 네이밍만 같음.
- `agent_proposal_service.py` 가 실질적인 에이전트 쓰기 경로:
  - `content_snapshot = {"type": "text", "content": content}` 를 직접 구성해 INSERT (L95~121).
  - **ProseMirror 표준 아님** — heading/paragraph/list 같은 구조가 통째로 무시된다.
  - 개선: `type="doc"` 루트 + `paragraph` 노드 최소 1개 감싸는 변환기를 도입. FG 1-1 서브태스크 후보.
- `draft_service.save_draft` 는 에이전트가 호출할 수 있으나 현재 MCP Tool 표면에 명시적 바인딩 없음 (확인: `mcp/tools.py` 에 `save_draft` 직접 호출 코드 없음).

**결론**: 에이전트도 사람과 동등한 API 로 `save_draft` 를 쓸 수 있어야 한다는 S2 원칙 ⑤ 기준으로, 에이전트가 `content_snapshot` 표준(ProseMirror `{type:"doc"}`)을 만들어 `save_draft` 를 호출하도록 가이드를 포함해야 한다 — FG 1-1 의 `agent_proposal_service` 포맷 교정과 묶어서 처리.

---

## 6. Phase 1 FG 로의 인풋 (실측 근거로 확정)

### 6.1 FG 1-1 에 들어가야 할 구체 작업

1. **저장 경로 단일화**: PATCH `save_draft_nodes` 폐지 후보 → Phase 1 내에서는 `deprecated` 표시 + 내부에서 `save_draft` 로 위임. Phase 2 말미에 실제 삭제.
2. **프런트 전환**: `versionsApi.saveDraft` 바디 타입을 `{title?, summary?, content_snapshot: ProseMirrorDoc}` 로 변경 + 호출 메서드를 PUT 으로. 기존 `nodes` 필드는 제거.
3. **`save_draft` 서비스 측 파생 동기화**: `content_snapshot` 저장 시 ProseMirror → nodes 변환기 동작시켜 nodes 테이블도 같이 재생성 (vectorization fallback 로직을 정규 경로로 승격).
4. **content_snapshot validator 강화**: `schemas/versions.py` 의 `validate_content_snapshot` 에 `type=="doc"` 루트 / `content: list` 존재 검증 추가. `agent_proposal_service` 의 `{"type":"text"}` 포맷을 제거.
5. **마이그레이션 배치**: 기존 DB 의 content_snapshot 을 nodes 로부터 재구성하는 일회성 backfill 스크립트 (drat 전용, published 는 별도). Alembic data migration 경로.
6. **회귀**:
   - pytest: `save_draft` 호출 후 `nodes` 테이블도 동기화됐는지 어서트 (신규).
   - pytest: `save_draft_nodes` (deprecated) 호출 시 `content_snapshot` 도 갱신되는지 어서트 (과도기).
   - `test_draft_service_fg03.py` 의 skip 8건 중 `test_restore_published_creates_new_draft` 는 본 작업과 스키마가 겹치므로 함께 복구.

### 6.2 FG 1-2 에 들어가야 할 구체 작업

1. TipTap 기반 에디터로 `DocumentEditPage` 를 이식. `@tiptap/starter-kit` + 커스텀 `node_id` extension (전역 attribute, `addGlobalAttributes` 패턴).
2. 두 뷰 구현:
   - **블록 뷰** (현재 UI 감성 유지): TipTap `NodeViewRenderer` 로 heading/paragraph/list/code 를 블록 카드로 래핑. 좌측 드래그 핸들 / 우측 이동·삭제 버튼 유지.
   - **일반 뷰**: 동일 doc, TipTap 기본 렌더 (흐르는 리치텍스트).
3. **토글 UI**: 헤더에 세그먼트 버튼 (블록 / 일반). 전환은 즉시, doc 재마운트 없음 (같은 EditorState 유지, prop 으로 decoration/nodeView 만 교체).
4. **저장**: `editor.getJSON()` → `content_snapshot` → PUT 호출 (FG 1-1 의 새 API 계약).
5. **접근성**: 키보드 단축키 (`Cmd/Ctrl+Shift+V` 뷰 토글 후보), screen reader 라벨.
6. **UI 리뷰 ≥ 5회** 로그를 `산출물/FG1-2_UI리뷰로그.md` 로 제출.
7. **폐쇄망 체크**: TipTap 모두 OSS, CDN 참조 없음 — `npm audit` 로 critical 0건 확인 (CLAUDE.md 전역 규칙 2).

### 6.3 FG 1-3 에 들어가야 할 구체 작업

1. **node_id stable 회귀**: 블록 뷰 → 일반 뷰 → 블록 뷰 왕복 후 `editor.getJSON()` 의 모든 노드 `attrs.node_id` 가 불변임을 어서트 (node:test).
2. **편집 중 node_id 보존**: 중간 문단 추가/삭제 후에도 다른 노드의 node_id 가 변하지 않음을 어서트.
3. **citation anchoring 기반**: 서버 측 `citation_builder` 가 기대하는 node_id 포맷(UUID) 을 에디터가 유지하는지 pytest 단에서 간접 검증.
4. **뷰 선호 저장**: users.preferences JSON 컬럼 (Alembic revision) + `GET/PATCH /me/preferences` API + 프런트 저장/복원.
5. **다음 단계 훅**: 인라인 주석은 Phase 3 범위이나, node_id 가 anchor key 이므로 본 FG 에서 "주석 앵커 키 = node_id" 계약을 문서화.

---

## 7. 리스크 / 열린 질문

| # | 리스크 | 대응 |
|---|-------|-----|
| R1 | PATCH 경로를 deprecate 하면 기존 draft 를 열 때 하위호환 문제 | Phase 1 기간 내 PATCH 는 내부적으로 PUT 으로 위임. 바디 형식 둘 다 수용하는 변환기 |
| R2 | 프런트 이식 후 기존 draft(비표준 content_snapshot)가 에디터에서 깨짐 | `content_snapshot` 로드 시 fallback 변환기 (nodes → ProseMirror) 보유 |
| R3 | `agent_proposal_service` 의 `{"type":"text"}` 포맷이 기존 proposal 에 DB 상 남아 있음 | 읽기 쪽에서도 변환기 적용 + 마이그레이션으로 재작성 |
| R4 | TipTap 버전 3.x 가 React 19 strict mode 와 충돌 여부 | Phase 1 착수 직후 smoke 확인 (개별 리스크 아이템) |
| R5 | Phase 0 skip 8건 중 스키마 충돌 | FG 1-1 작업 시 `test_draft_service_fg03.py::test_restore_published_creates_new_draft` 는 본 작업과 함께 복구 |
| Q1 | 뷰 선호 저장을 users 테이블 컬럼 vs 별도 user_preferences 테이블 어느 쪽이 적절? | FG 1-3 세부 결정. 단일 필드면 컬럼 JSON, 추후 확장 고려면 테이블 |
| Q2 | PATCH 경로의 감사 이벤트(`draft.nodes_saved`) 는 유지 vs PUT 으로 합치기 | 감사 consumer 영향 조사 후 결정 |

---

## 8. 참조

- `docs/개발문서/S3/phase1/Phase 1 개발계획서.md` §1~§7
- `docs/개발문서/S3/phase1/Phase1_인수문서.md` §3~§4
- `docs/개발문서/S3/phase0/산출물/FG0-3_종결보고서.md`
- `CLAUDE.md` (S1 ①~④ + S2 ⑤~⑦)
- 코드: `backend/app/api/v1/documents.py`, `backend/app/services/draft_service.py`, `backend/app/services/vectorization_service.py`, `backend/app/services/render_service.py`, `backend/app/services/agent_proposal_service.py`, `backend/app/mcp/tools.py`, `frontend/src/features/editor/DocumentEditPage.tsx`, `frontend/src/lib/api/versions.ts`

---

*작성: 2026-04-24 | Phase 1 착수 전 선행 실측. 작업지시서 task1-1 ~ task1-3 의 기초 자료.*
