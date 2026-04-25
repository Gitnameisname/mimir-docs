# task1-1 — 저장 모델 정규화 (content_snapshot 단일 정본)

**작성일**: 2026-04-24
**Phase / FG**: S3 Phase 1 / FG 1-1
**상태**: 착수 대기
**담당**: 백엔드 + 프런트 (소규모)
**예상 소요**: 2~3일
**선행 산출물**: `docs/개발문서/S3/phase1/산출물/Pre-flight_실측.md`
**사용자 확정 결정**: D1 — `content_snapshot` 단일 정본 / `nodes` 는 파생 동기화

---

## 1. 작업 목적

Pre-flight 실측(`산출물/Pre-flight_실측.md` §2) 에서 확인된 바와 같이 현재 DB 에는 **두 개의 서로 다른 저장 경로**가 상시 병존한다:

- `PUT /documents/{id}/draft` → `versions.content_snapshot` 만 갱신
- `PATCH /documents/{id}/versions/{vid}/draft` → `nodes` 테이블만 갱신

프런트 에디터는 후자만 쓰기 때문에 **편집 직후 `content_snapshot` 이 stale** 이고, `render_service` 는 `content_snapshot` 을 읽기 때문에 편집본과 렌더 결과가 어긋난다. 이것이 S2 BUG-02 의 구체적 형태이다.

본 작업은 **`content_snapshot` 을 단일 정본** 으로 확정하고, `nodes` 테이블을 **`content_snapshot` 으로부터 서버 사이드에서 자동 파생**되도록 재설계한다. 이후 Phase 1 FG 1-2 의 TipTap 기반 일반/블록 뷰 토글이 이 단일 저장 모델 위에 안전하게 얹힐 수 있도록 한다.

---

## 2. 작업 범위

### 2.1 포함

1. **백엔드 — 저장 경로 단일화**
   - `save_draft` (PUT) 를 유일한 Draft 쓰기 경로로 확정
   - `save_draft` 내부에서 `content_snapshot` 저장 직후 **nodes 파생 동기화** 호출 (현재 `vectorization_service._parse_nodes_from_snapshot` 로직을 정규 경로로 승격 → `snapshot_sync_service.rebuild_nodes_from_snapshot` 신설)
   - `save_draft_nodes` (PATCH) 는 **deprecated 표시** 하되 내부에서 nodes → ProseMirror 변환기로 `content_snapshot` 을 복원한 뒤 `save_draft` 로 위임 (과도기 하위호환)
   - `content_snapshot` 루트 형식을 `schemas/versions.py` validator 로 강제: `{type: "doc", content: list, ...}` 허용. `{type: "text", ...}` 같은 비표준 거부
2. **백엔드 — 에이전트 쓰기 경로 정규화**
   - `agent_proposal_service.py` L95 의 `{"type":"text", "content": content}` 포맷 제거 → `{type:"doc", content:[{type:"paragraph", content:[{type:"text", text: content}]}]}` 로 교정
3. **백엔드 — 마이그레이션 / 데이터 backfill**
   - Alembic revision `s3_p1_content_snapshot_backfill` 추가
   - Draft 버전 중 `content_snapshot` 이 NULL 이거나 `{type:"text"}` 비표준인 레코드를 **nodes 테이블 기반으로 재구성**해 정본 포맷으로 UPDATE
   - Published 버전은 **절대 수정 금지** (불변 원칙). 발견 시 warning 로그만 출력하고 건너뜀
   - dry-run 플래그 지원 (`SCHEMA_MIGRATION_DRY_RUN=1`) — 폐쇄망 배포 롤백 가능성 확보
4. **프런트 — 저장 클라이언트 교체**
   - `lib/api/versions.ts` 의 `saveDraft` 바디 타입을 `{title?, summary?, label?, change_summary?, content_snapshot: ProseMirrorDoc}` 로 변경, 호출 메서드를 PUT 으로
   - 기존 `nodes` 필드는 제거
   - 타입 `ProseMirrorDoc` 을 `types/prosemirror.ts` 신설 모듈에 정의 (최소 shape: `type: "doc", content: Node[]` — FG 1-2 에서 확장)
5. **프런트 — 에디터 어댑터 임시 계층**
   - FG 1-2 의 TipTap 이식 전까지는 현 `DocumentEditPage` 의 `DocumentNode[]` state 에서 ProseMirror JSON 으로 **변환해 내보내는 어댑터** (`features/editor/adapters/nodesToProseMirror.ts`) 를 도입
   - 저장 시: `nodes[] → ProseMirrorDoc` → PUT. 로드 시: PUT 응답의 `content_snapshot` 또는 fallback 으로 `nodesApi.list` → `DocumentNode[]` 유지 (FG 1-2 에서 TipTap 로 전환되면 이 어댑터 삭제)

### 2.2 제외 (본 FG 에서 다루지 않음)

- TipTap 기반 에디터 UI 이식 → FG 1-2
- 블록/일반 뷰 토글 → FG 1-2
- node_id stable 회귀 테스트 → FG 1-3
- 뷰 선호 저장(users.preferences) → FG 1-3
- 인라인 주석 / 댓글 → Phase 3
- `save_draft_nodes` 엔드포인트의 **완전 삭제** → Phase 2 또는 종결 시점 (본 FG 에서는 deprecated 만)

### 2.3 하드코딩 금지 재확인 (S1 / S2 원칙)

- DocumentType 별 분기 로직은 **서비스 레이어에서만**. `nodesToProseMirror` 어댑터 내부에는 `if type_code == "POLICY"` 같은 분기 금지 — 타입별 차이는 DocumentType schema 의 chunking/rendering plugin 설정에서 읽는다
- Scope 어휘 하드코딩 금지 — 본 FG 는 Scope Profile 자체를 변경하지 않으므로 해당 없음이나, deprecation 테스트에서 scope 문자열 직접 비교 금지

---

## 3. 선행 조건

- Phase 0 FG 0-3 공식 종결 (pytest `2,662 passed / 0 failed / 124 skipped`)
- `backend/app/db/migrations/versions/` 최신 head 가 머지 상태
- `Pre-flight_실측.md` 의 §2, §3 사실 관계가 유효 (본 작업 착수 직전 1회 재확인)

---

## 4. 구현 단계

### Step 1 — 변환기 레이어 신설 (선행)

- **위치**: `backend/app/services/snapshot_sync_service.py` (신설)
- **API**:
  ```python
  def prosemirror_from_nodes(nodes: list[DraftNodeItem]) -> dict
  def nodes_from_prosemirror(snapshot: dict) -> list[dict]
  def rebuild_nodes_from_snapshot(conn, version_id: str, snapshot: dict) -> None
  ```
- `vectorization_service._parse_nodes_from_snapshot` 의 핵심 로직을 이 서비스로 이동 + 재export 제공 (vectorization 은 당분간 import 경유로 호환)
- 단위 테스트: 왕복 변환(prosemirror → nodes → prosemirror) 노드 id / order / depth 보존 어서트

### Step 2 — content_snapshot validator 강화

- `backend/app/schemas/versions.py::validate_content_snapshot`
- 추가 규칙:
  - `type` 이 존재하고 `"doc"` 여야 함
  - `content` 가 list 여야 함 (빈 리스트 허용 — 빈 문서)
  - 크기 상한(기존 `_CONTENT_SNAPSHOT_MAX_BYTES`) 유지
- 단위 테스트: valid / `type:"text"` 거부 / `content:null` 거부 / 크기 초과 거부

### Step 3 — save_draft 에 파생 동기화 연결

- `backend/app/services/draft_service.py::save_draft`
- `versions_repository.create/update_content` 직후 `snapshot_sync_service.rebuild_nodes_from_snapshot(conn, version.id, request.content_snapshot)` 호출
- 트랜잭션 경계: 기존 `with get_db() as conn:` 내에 포함 (외부에서 관리)

### Step 4 — save_draft_nodes deprecation

- `backend/app/api/v1/documents.py` 의 PATCH 라우트 description 에 `[DEPRECATED]` 표기
- 서비스 `save_draft_nodes` 내부:
  - nodes → ProseMirror 변환 후 `save_draft` 로 위임
  - `logger.warning("save_draft_nodes is deprecated; use PUT /draft with content_snapshot")` 발화
  - 응답 헤더에 `Deprecation: true`, `Sunset: 2026-10-01` (예시 일자 — Phase 결과 확정 시 갱신)
- 감사 이벤트명 `draft.nodes_saved` 는 유지 (consumer 호환)

### Step 5 — Alembic backfill

- `backend/app/db/migrations/versions/s3_p1_content_snapshot_backfill.py`
- `upgrade()`:
  - Draft 상태 + `content_snapshot IS NULL OR content_snapshot->>'type' <> 'doc'` 조건으로 버전 ID 수집
  - 각 버전의 nodes → ProseMirror doc 생성 → UPDATE
  - 처리 건수 / 스킵 건수 / 에러 건수 로그
- `downgrade()`: backfill 된 스냅샷을 되돌리지 않는다 (No-op, 주석으로 명시)
- dry-run 환경변수 지원

### Step 6 — agent_proposal_service 포맷 교정

- `backend/app/services/agent_proposal_service.py::create_proposal`
- `content_snapshot = {"type":"text", ...}` → `snapshot_sync_service.prosemirror_from_text(content)` 로 대체
- 단위 테스트 갱신

### Step 7 — 프런트 API 클라이언트 교체

- `frontend/src/lib/api/versions.ts::saveDraft`
  - 바디 타입 교체, `api.patch` → `api.put`
  - URL: `/api/v1/documents/${documentId}/draft` (versionId path 제거 — `save_draft` 는 Draft 하나로 한정되므로 versionId 파라미터 불요)
- `frontend/src/types/prosemirror.ts` 신설:
  ```ts
  export type ProseMirrorNode = {
    type: string;
    attrs?: Record<string, unknown>;
    content?: ProseMirrorNode[];
    marks?: Array<{ type: string; attrs?: Record<string, unknown> }>;
    text?: string;
  };
  export type ProseMirrorDoc = { type: "doc"; content: ProseMirrorNode[] };
  ```

### Step 8 — 프런트 에디터 임시 어댑터

- `frontend/src/features/editor/adapters/nodesToProseMirror.ts` 신설
- `DocumentNode[] ↔ ProseMirrorDoc` 변환 (section/heading/paragraph/list/code_block 매핑)
- `DocumentEditPage` 의 `saveMutation`: `saveDraft(documentId, { content_snapshot: nodesToProseMirror(nodes), title })`
- **FG 1-2 종료 시 삭제 예정** 주석 명시

### Step 9 — 테스트

- **pytest (backend)**:
  - `test_draft_service_fg11.py`: save_draft → nodes 자동 동기화 / save_draft_nodes → content_snapshot 자동 동기화 / content_snapshot validator 거부 케이스
  - `test_snapshot_sync_service.py`: 왕복 변환 보존 (id/order/depth/attrs)
  - `test_agent_proposal_service_fg11.py`: ProseMirror 표준 포맷으로 저장됨
  - `test_content_snapshot_backfill.py`: Alembic backfill 대상 선별 / published 건너뜀 / dry-run 동작
  - **Phase 0 skip 복구**: `test_draft_service_fg03.py::test_restore_published_creates_new_draft` — 본 작업에서 `metadata_snapshot` 속성 추가 라인이 정리되므로 함께 해제
- **node:test (frontend)**:
  - `tests/versions_api.saveDraft.test.ts`: PUT 호출 URL / 바디 shape / content_snapshot 존재
  - `tests/nodesToProseMirror.adapter.test.ts`: 왕복 변환 케이스
- **회귀 기준선**: 본 작업 후 pytest `2,662 passed` 유지 + skip 이 125 로 늘지 않음 (복구한 1건 반영 시 123). 실패 시 차단

### Step 10 — 보안 점검

- `backend/app/schemas/versions.py` validator: 악의적 content_snapshot(대형 dict / 깊은 중첩) 방어 테스트
- 프런트: `dompurify` 이미 설치. FG 1-2 에서 HTML 렌더 시 필수이나 본 FG 에서는 JSON 만 왕복 → 해당 없음
- `npm audit --audit-level=high` 실행 결과 0건 (CLAUDE.md 전역 규칙 2)
- `pip-audit` 백엔드 결과 신규 critical 0건

---

## 5. 검수 기준 (CLAUDE.md S1 ⑤⑥ 의무)

| # | 기준 | 근거 |
|---|------|------|
| C1 | save_draft 호출 후 nodes 테이블이 content_snapshot 과 정합 | pytest `test_save_draft_syncs_nodes` |
| C2 | save_draft_nodes 호출 후에도 content_snapshot 이 동기화됨 (과도기) | pytest `test_save_draft_nodes_delegates_to_save_draft` |
| C3 | content_snapshot validator 가 비표준 포맷 거부 | pytest 3종 |
| C4 | agent_proposal_service 가 ProseMirror 표준 포맷으로 저장 | pytest `test_agent_proposal_uses_prosemirror_doc` |
| C5 | Alembic backfill 이 draft 만 수정, published 건너뜀 | pytest `test_backfill_skips_published` |
| C6 | 프런트 저장 클라이언트가 PUT + content_snapshot 바디로 호출 | node:test `tests/versions_api.saveDraft.test.ts` |
| C7 | pytest 베이스라인 `2,662 passed` 유지 (skip 복구 반영하면 skip 123) | 실 실행 로그 |
| C8 | node:test 전부 녹색 | 실 실행 로그 |
| C9 | 검수보고서 / 보안취약점검사보고서 제출 | `산출물/FG1-1_*.md` |

---

## 6. 산출물

| 파일 | 설명 |
|------|------|
| `docs/개발문서/S3/phase1/산출물/FG1-1_검수보고서.md` | C1~C9 체크리스트 + pytest/node:test 실 실행 로그 |
| `docs/개발문서/S3/phase1/산출물/FG1-1_보안취약점검사보고서.md` | `pip-audit` + `npm audit` 결과, validator 악성 입력 방어 케이스 표 |
| (선택) `docs/개발문서/S3/phase1/산출물/FG1-1_백필_dry-run_로그.md` | 운영 DB 대상 dry-run 결과 (운영자 실행 후 첨부) |

---

## 7. 리스크 / 열린 질문

| # | 리스크 | 대응 |
|---|-------|-----|
| R1 | backfill 에서 `content_snapshot` 이 이미 `{type:"text"}` 인 레코드를 건드리며 데이터 손실 가능 | dry-run 필수 + 원본을 `metadata_snapshot.content_snapshot_before_backfill` 로 보존하는 옵션 도입 |
| R2 | 프런트 어댑터(`nodesToProseMirror`) 가 FG 1-2 에서 폐기됨에도 오래 살아남으면 혼란 | 파일 상단 주석 + `@deprecated` JSDoc + 종결 기한(2026-05-15) 명시 |
| R3 | save_draft_nodes deprecation 이 외부 API 소비자(에이전트/스크립트)에 영향 | `Deprecation`/`Sunset` 헤더 + 변경 로그에 명시. Phase 2 착수 시 재확인 후 제거 |
| R4 | vectorization_service 가 현재 `_parse_nodes_from_snapshot` 로 content_snapshot 을 nodes 로 역 주입하는 fallback 에 의존. 정규 경로 승격 후 이중 실행 가능 | save_draft 경로에서 재구성된 경우 `vectorization_service` 의 fallback 은 no-op 이 되도록 가드 |
| Q1 | content_snapshot 의 `version` 필드(스키마 버전 번호) 를 이번에 넣을지? | 권장: yes. `{"type":"doc","schema_version":1,"content":[...]}` — 향후 마이그레이션에 유용 |
| Q2 | 에이전트가 save_draft 를 MCP Tool 로 호출 가능하게 노출해야 하는가? (S2 ⑤) | 본 FG 에서는 기존 `agent_proposal_service` 경로 유지. Phase 3 의 에이전트 쓰기 확장에서 재검토 |

---

## 8. 참조

- `docs/개발문서/S3/phase1/Phase 1 개발계획서.md` §1, §2 FG 1-1
- `docs/개발문서/S3/phase1/산출물/Pre-flight_실측.md` §2, §4, §6.1
- `CLAUDE.md` S1 ①③④ + S2 ⑤⑦
- Phase 0 `FG0-3_종결보고서.md` — 베이스라인 수치

---

*작성: 2026-04-24 | Phase 1 FG 1-1 착수 대기*
