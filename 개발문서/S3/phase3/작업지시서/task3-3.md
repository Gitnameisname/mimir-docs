# task3-3 — 인라인 주석 (node_id anchoring + 멘션 + MCP)

**작성일**: 2026-04-27
**Phase / FG**: S3 Phase 3 / FG 3-3
**상태**: 착수 대기 (FG 3-1 완료 후 / FG 3-2 와는 병행 가능)
**담당**: 백엔드 + 프런트 + MCP
**예상 소요**: 4~5일 (Phase 3 의 최대 규모)
**선행 산출물**:
- `task3-1.md` (Contributors 패널의 `actor_type` 표시 규약 재사용)
- Phase 1 NodeId 안정성 회귀 테스트 (`frontend/tests/NodeIdStableFg13.test.tsx`) 녹색 유지
- S2 Phase 4 MCP Tool 표면 (`backend/app/mcp/tools.py`, `backend/app/api/v1/mcp_router.py`)

---

## 1. 작업 목적

문서 본문의 **특정 노드(블록 또는 인라인 범위)** 에 주석을 달아 토론을 가능하게 한다. 좌표는 **node_id (UUID)** + 선택적 문자 오프셋 범위. 버전 diff 후에도 같은 node_id 가 살아있다면 주석은 **그대로 anchor 된다** (Phase 1 의 node_id 안정성 위에 얹힘). node_id 가 사라지면 **고아 주석(orphan)** 으로 표시한다.

부가 기능: 해결/미해결 상태, 답글 스레드, `@user` 멘션 → 알림. 에이전트가 주석을 읽을 수 있도록 **MCP Tool 표면**도 추가 (S2 ⑤ 일관성).

---

## 2. 작업 범위

### 2.1 포함

1. **DB 스키마 (Alembic revision `s3_p3_annotations`)**
   ```sql
   CREATE TABLE annotations (
       id              UUID PRIMARY KEY,
       document_id     UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
       version_id      UUID NULL REFERENCES versions(id) ON DELETE SET NULL,
       node_id         UUID NOT NULL,
       span_start      INT NULL,    -- 노드 텍스트 내 문자 오프셋 (NULL = 노드 전체)
       span_end        INT NULL,    -- exclusive
       author_id       VARCHAR(255) NOT NULL,
       actor_type      VARCHAR(32) NOT NULL DEFAULT 'user',  -- user | agent | system
       content         TEXT NOT NULL CHECK (length(content) BETWEEN 1 AND 10000),
       status          VARCHAR(16) NOT NULL DEFAULT 'open',  -- open | resolved
       resolved_at     TIMESTAMPTZ NULL,
       resolved_by     VARCHAR(255) NULL,
       parent_id       UUID NULL REFERENCES annotations(id) ON DELETE CASCADE,  -- 답글
       is_orphan       BOOLEAN NOT NULL DEFAULT false,
       orphaned_at     TIMESTAMPTZ NULL,
       created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
       updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
   );
   CREATE INDEX idx_annotations_document_id ON annotations(document_id);
   CREATE INDEX idx_annotations_node_id ON annotations(node_id);
   CREATE INDEX idx_annotations_parent_id ON annotations(parent_id) WHERE parent_id IS NOT NULL;
   CREATE INDEX idx_annotations_status_open ON annotations(document_id, status) WHERE status='open';

   CREATE TABLE annotation_mentions (
       annotation_id       UUID NOT NULL REFERENCES annotations(id) ON DELETE CASCADE,
       mentioned_user_id   VARCHAR(255) NOT NULL,
       notified_at         TIMESTAMPTZ NULL,
       PRIMARY KEY (annotation_id, mentioned_user_id)
   );
   ```
   - `span_start < span_end` CHECK (둘 다 NOT NULL 일 때)
   - ACL 은 **document.scope_profile_id** (FG 2-0) 가 결정. annotations 자체는 ACL 무관 — 단 annotation 조회 시 document join 필터 자동 적용

2. **백엔드 — 모델 / Repository**
   - `app/models/annotation.py` — `Annotation`, `AnnotationMention` dataclass
   - `app/repositories/annotations_repository.py` — CRUD + `list_for_document(document_id, include_resolved, include_orphans)`
   - `app/repositories/annotation_mentions_repository.py` — bulk insert / list

3. **백엔드 — Service**
   - `services/annotations_service.py`:
     - `create_annotation(actor, document_id, node_id, span, content, parent_id?) -> Annotation`
     - `update_annotation(actor, annotation_id, content?) -> Annotation` — 작성자 본인만
     - `resolve_annotation(actor, annotation_id) -> Annotation` — 작성자 또는 admin
     - `reopen_annotation(actor, annotation_id) -> Annotation`
     - `delete_annotation(actor, annotation_id) -> None` — 작성자 또는 admin (cascade 답글)
     - `list_for_document(actor, document_id, ...) -> list[Annotation]`
   - 멘션 파싱: `extract_mentions(content) -> list[user_id]` — 정규식 `@([\w.\-]{2,64})` + DB lookup 으로 valid user_id 만 채택. invalid 는 무시(plain text)
   - audit emit: `annotation.created` / `annotation.updated` / `annotation.resolved` / `annotation.deleted` (FG 3-1 의 contributors editors 카테고리에 자연 합류)

4. **백엔드 — node_id 보존 / 고아 검출**
   - `services/annotation_anchoring_service.py`:
     - `recompute_orphans(conn, document_id, latest_node_ids: set[UUID]) -> int` — 현재 versions 의 최신 snapshot 에서 추출한 node_id 집합과 비교, 빠진 node_id 의 annotation 을 `is_orphan=true, orphaned_at=now()` 로 표시
     - **저장 경로 결합**: `save_draft` / `publish_version` 의 트랜잭션 안에서 `rebuild_nodes_from_snapshot` 직후 호출 (Phase 2 FG 2-3 의 wikilink rebuild 와 같은 위치)
     - `is_orphan=true` 였는데 새 snapshot 에서 같은 node_id 가 다시 나타나면 `is_orphan=false, orphaned_at=null` 로 복구 (작성자가 undo 한 경우 등)
   - `latest_node_ids` 추출 util 은 Phase 1 의 NodeId 추출 로직 재사용 (`snapshot_sync_service` 의 `nodes_from_prosemirror`)

5. **백엔드 — 멘션 알림 (S2 알림 시스템 재사용 / 부재 시 최소 신설)**
   - 사전 점검: S2 에 알림 시스템이 존재하는지 확인. 부재 (코드베이스 서베이 결과: "no notification/mention system exists yet") → **본 FG 가 최소 알림 테이블 + emit 만 신설**:
     ```sql
     CREATE TABLE notifications (
         id          UUID PRIMARY KEY,
         user_id     VARCHAR(255) NOT NULL,
         kind        VARCHAR(32) NOT NULL,           -- 'annotation.mention'
         payload     JSONB NOT NULL,                 -- {annotation_id, document_id, snippet, author_id}
         read_at     TIMESTAMPTZ NULL,
         created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
     );
     CREATE INDEX idx_notifications_user_unread ON notifications(user_id, created_at DESC) WHERE read_at IS NULL;
     ```
   - `services/notifications_service.py` — `enqueue(user_id, kind, payload)`, `list_unread(user_id)`, `mark_read(user_id, ids)`
   - **rate-limit**: per (author_id, mentioned_user_id) **분당 5건** 폭주 방지 (한 사용자가 같은 사람을 짧은 시간에 도배 못 함)
   - API: `GET /api/v1/notifications?unread_only=true`, `POST /api/v1/notifications/read` body `{ids:[...]}`
   - **본 FG 의 알림은 in-app 만**. 이메일/푸시는 S4 범위
   - 향후 S2 알림 시스템이 별도 도입되면 본 모듈을 그쪽으로 흡수하는 ADR 별도

6. **백엔드 — REST API**
   - `api/v1/annotations.py`:
     | 메서드 | 경로 | 설명 |
     |-------|------|-----|
     | POST | /api/v1/documents/{id}/annotations | 본문 `{node_id, span_start?, span_end?, content, parent_id?}` |
     | GET | /api/v1/documents/{id}/annotations | `?include_resolved=&include_orphans=` |
     | GET | /api/v1/annotations/{id} | 단건 + 답글 트리 |
     | PATCH | /api/v1/annotations/{id} | 본문/상태 |
     | POST | /api/v1/annotations/{id}/resolve | |
     | POST | /api/v1/annotations/{id}/reopen | |
     | DELETE | /api/v1/annotations/{id} | cascade 답글 |
   - rate-limit: per-user 30 req/min (생성), 120 req/min (조회)
   - `router.py` 등록

7. **백엔드 — MCP Tool 표면 (S2 ⑤ 일관성)**
   - `backend/app/mcp/tools.py` 에 신규:
     ```python
     def tool_read_annotations(
         args: dict, principal: AgentPrincipal, conn,
     ) -> dict:
         """document_id 의 annotations 를 읽기 전용으로 반환. agent 의 scope 필터 통과 필수."""
     ```
   - `mcp_router.py` 의 `/mcp/tools/call` 디스패치에 등록
   - `schemas/mcp.py` 에 args/response schema 추가
   - **에이전트 권한**: `AgentPrincipal.can_read("annotations")` 체크. ScopeProfile 의 `allowed_tools` 에 `read_annotations` 가 없으면 거부 (S2 Phase 4/5 의 ScopeProfile 권한 모델 그대로)
   - 에이전트의 **쓰기**(`create_annotation`)는 본 FG 에서 MCP 표면에 추가하지 않음 — 향후 ADR 후 별도. 단, REST API 로는 actor_type=agent 가 직접 호출 가능 (Phase 1/2 의 agent_proposal 패턴과 동일 권한 검사)

8. **프런트엔드 — TipTap 통합**
   - `frontend/src/features/editor/tiptap/extensions/AnnotationMark.ts`:
     - 인라인 mark — `data-annotation-id` 속성 보유, span_start/span_end 범위 marking
     - 노드 전체 주석은 mark 대신 **gutter 마커** 로 표시 (다음 항목)
     - 클릭 시 우측 패널 열기 + 해당 주석 highlight
   - `features/editor/tiptap/AnnotationGutter.tsx` — 본문 좌측 여백에 노드별 주석 개수 점/뱃지
   - 새 주석 만들기: 본문 선택 → Floating toolbar 의 "주석" 버튼 → 우측 패널에 입력창 활성화

9. **프런트엔드 — 주석 패널**
   - `features/documents/AnnotationsPanel.tsx` — 문서 우측 패널 (Contributors 패널과 탭으로 구분)
   - 각 주석 카드: 작성자 + actor_type 뱃지(FG 3-1 재사용) + 본문 + 답글 스레드 + 해결/재오픈/삭제 버튼
   - 멘션 입력: `@` 입력 시 자동완성 팝업 (사용자 검색 API; 이미 존재하면 재사용, 없으면 `GET /api/v1/users?q=` 신설)
   - 고아 주석은 별도 섹션 "고아 주석 (anchored 노드 사라짐)" 으로 분리, 회색 배경
   - 빈/로딩/에러 상태

10. **프런트엔드 — 알림 표시**
    - 헤더 우상단 종 아이콘 + unread count 뱃지
    - 클릭 시 드롭다운 — 최근 10건, 클릭 시 해당 문서 + 주석 anchor 로 이동
    - 폴링 30s 또는 (선택) SSE — 본 FG 는 polling 으로 단순화

### 2.2 제외

- 주석의 문서 간 링크 — S4 (Phase 3 개발계획서 §6)
- 오프라인 주석 동기화 — S4
- 이메일/푸시 알림 — S4
- AI 자동 주석 추천 — Phase 3 후속
- 주석 export (PDF / Markdown) — S4

### 2.3 하드코딩 금지

- 멘션 정규식, content 길이 상한(10000), rate-limit 수치(분당 5/30/120)는 `services/annotations_config.py` 단일 모듈
- mark CSS 클래스명은 i18n / theme 토큰
- `kind='annotation.mention'` 같은 enum 문자열은 enum/Literal 로

---

## 3. 선행 조건

- task3-1 완료 — actor_type 뱃지 컴포넌트 재사용
- Phase 1 의 NodeId 안정성 회귀 테스트 녹색 유지 — node_id 가 마이그레이션/저장에서 변하지 않음
- FG 2-0 의 documents.scope_profile_id 필터 안정
- S2 Phase 4 의 MCP 라우터/AgentPrincipal 안정

---

## 4. 구현 단계

### Step 1 — 스키마 / Repository

1. Alembic revision `s3_p3_annotations` 작성 (annotations + annotation_mentions + notifications 3 테이블)
2. dataclass 모델 + repository CRUD
3. pytest unit ≥ 12 건 — 생성/조회/답글 트리/cascade/CHECK 위반

### Step 2 — Service / 멘션 파싱

1. `annotations_service.py` 의 5 액션 구현
2. `extract_mentions(content)` + valid user_id 필터
3. 권한 검증: 작성자 본인 / admin / actor_type=agent 의 ScopeProfile 권한
4. audit emit 4 종 (`annotation.created/updated/resolved/deleted`)
5. pytest unit ≥ 18 건

### Step 3 — Anchoring / Orphan

1. `annotation_anchoring_service.recompute_orphans` 구현
2. `save_draft` / `publish_version` 트랜잭션에 결합 (FG 2-3 wikilink rebuild 와 같은 지점)
3. integration test ≥ 8 건:
   - 노드 삭제 시 해당 주석 → orphan
   - 같은 node_id 복구 시 → orphan 해제
   - 노드 텍스트만 변하고 node_id 유지 → 주석 그대로 anchored
   - span_start/end 가 노드 새 텍스트 길이 초과 시 → 주석은 그대로 두되 응답에 `span_clamped=true` 플래그 (UI 가 안전 표시)
   - 답글 스레드는 부모와 함께 orphan
   - 동시 저장 race (advisory lock 또는 트랜잭션 격리 확인)
   - 다중 버전 publish 후에도 anchoring 유지
   - 마이그레이션 직후 backfill 시나리오 (기존 주석 0 건 가정 → 무영향)

### Step 4 — 알림 시스템 (최소)

1. notifications 테이블 + service + API
2. `enqueue` 호출 위치: `create_annotation` 의 멘션 처리 직후
3. rate-limit 적용
4. unit ≥ 8 건 — 폭주 차단, read 처리, 권한(타인 알림 못 읽음)

### Step 5 — REST API

1. `api/v1/annotations.py` 7 엔드포인트
2. `api/v1/notifications.py` 2 엔드포인트
3. integration ≥ 12 건 — 권한 / 답글 / 해결 / 삭제 cascade / orphan 응답 / include 옵션 / rate-limit

### Step 6 — MCP Tool

1. `mcp/tools.py` 에 `tool_read_annotations` 추가
2. `mcp_router.py` 디스패치 등록
3. ScopeProfile `allowed_tools` 에 `read_annotations` 키 추가 (관리자 UI 에서 토글 가능 — S2 Phase 4 패턴)
4. integration ≥ 5 건 — 권한 통과/거부, scope 밖 문서 차단, 응답 포맷, agent_principal 검증

### Step 7 — TipTap AnnotationMark / Gutter

1. `extensions/AnnotationMark.ts` — span 렌더, 클릭 이벤트
2. `AnnotationGutter.tsx` — 노드별 점 표시
3. node:test ≥ 12 건 — mark 부여/제거, 클릭 라우팅, gutter 카운트, NodeId 와의 호환

### Step 8 — 주석 패널 / 멘션 자동완성

1. `AnnotationsPanel.tsx` — 카드 / 답글 / 액션 버튼
2. 멘션 자동완성 — `GET /api/v1/users?q=` (없으면 본 FG 에서 최소 신설, admin/본인 조직 한정)
3. 고아 주석 분리 섹션
4. node:test ≥ 12 건

### Step 9 — 알림 드롭다운

1. 헤더 종 아이콘 + 폴링 hook
2. 드롭다운 / 클릭 라우팅
3. node:test ≥ 6 건

### Step 10 — UI 검수 / 반응형

- desktop / narrow / tablet / mobile
- UI 리뷰 ≥ 1회 (CLAUDE.md 프로젝트 §5.2 완화 규정)
- 산출물 `FG3-3_UI리뷰로그.md`

### Step 11 — 검수 / 보안 보고서

- `FG3-3_검수보고서.md` + 재검수 1회
- `FG3-3_보안취약점검사보고서.md`:
  - **"뷰 ≠ 권한"**: 다른 Scope 의 사용자가 annotation_id 직접 호출 시 404
  - 멘션을 통한 정보 누출 — 존재하지 않는 user_id 멘션 시 invalid 처리, 존재 유출 없음
  - content XSS — 본문 sanitization 정책 (서버 저장은 raw, 렌더 시 escape — TipTap 기본 동작 확인)
  - rate-limit 회귀 (멘션 폭주 차단)
  - MCP 권한 — `allowed_tools` 미허용 시 거부, scope 밖 문서 거부
  - cascade 삭제 — 답글이 부모와 함께 삭제, 다른 사용자 답글 삭제 권한 (작성자 + admin 만)
  - **node_id 안정성 회귀** — Phase 1 테스트 녹색 유지 + anchoring 8 시나리오 녹색

### Step 12 — Phase 3 / S3 1라운드 종결 인계

- Phase 3 개발계획서 §8 "S3 1라운드 종결" 에 따라 본 FG 가 마지막. 종결 회고 작성:
  - `docs/개발문서/S3/종결회고.md`
  - S1/S2/S3 원칙 준수 자기 검증
  - 잔존 부채 / 이월 항목
  - S3 2라운드 또는 S4 킥오프 기초 자료

---

## 5. API 계약 요약

| 메서드 | 경로 | 설명 |
|-------|------|-----|
| POST | /api/v1/documents/{id}/annotations | 신규 주석 / 답글 |
| GET | /api/v1/documents/{id}/annotations | `?include_resolved=&include_orphans=` |
| GET | /api/v1/annotations/{id} | 단건 + 답글 |
| PATCH | /api/v1/annotations/{id} | 본문 수정 |
| POST | /api/v1/annotations/{id}/resolve | 해결 |
| POST | /api/v1/annotations/{id}/reopen | 재오픈 |
| DELETE | /api/v1/annotations/{id} | 삭제 (cascade 답글) |
| GET | /api/v1/notifications | `?unread_only=` |
| POST | /api/v1/notifications/read | `{ids:[...]}` |

MCP:
- `read_annotations` — args `{document_id}`, response `{annotations:[...]}`

---

## 6. 성공 기준

- [ ] annotations + annotation_mentions + notifications 3 테이블 생성
- [ ] node_id 기반 anchoring 8 시나리오 녹색 (특히 노드 삭제→orphan→복구→해제)
- [ ] 답글 스레드 cascade, 해결/재오픈, 삭제 권한 회귀
- [ ] 멘션 알림 enqueue + rate-limit + 읽음 처리
- [ ] MCP `read_annotations` 권한 통과/거부 회귀
- [ ] 프런트 mark + gutter + panel + 멘션 자동완성 + 알림 드롭다운
- [ ] 고아 주석 표시 케이스
- [ ] **Phase 1 node_id 안정성 회귀 녹색 유지**
- [ ] pytest 신규 ≥ 60 건 (service 18 + integration 8 + API 12 + MCP 5 + notifications 8 + repository 12 등)
- [ ] node:test 신규 ≥ 30 건
- [ ] 검수·재검수·보안 보고서 + UI 리뷰 로그 제출
- [ ] S3 1라운드 종결 회고 초안 제출

---

## 7. 리스크

| 리스크 | 대응 |
|-------|-----|
| node_id 안정성 위반 → anchoring drift | Phase 1 회귀 테스트 유지 + 본 FG 의 anchoring 시나리오 ≥ 10. 위반 발견 즉시 본 FG 게이트 미통과 |
| span_start/end 가 텍스트 변경에 깨짐 | span_clamped 플래그로 안전 표시. 사용자가 수동 재배치하거나 노드 단위 주석으로 폴백 |
| 멘션 알림 폭주 (DDoS / 도배) | per (author, mentioned_user) 분당 5건 + per-user 생성 30 req/min |
| 에이전트가 대량 주석 생성 | actor_type=agent 필터 + 관리자 UI 에서 일괄 숨김 옵션 (Phase 3 개발계획서 §7). 본 FG 는 필터 토글만 마련, UI 적용은 후속 |
| MCP 권한 누락 → agent 가 ACL 우회 | AgentPrincipal.can_read("annotations") + scope 필터 둘 다 통과 필수. integration 회귀 5 시나리오 |
| 알림 시스템 재발명이 S4 통합과 충돌 | 본 FG 는 최소 in-app 알림만. 후속 ADR 로 흡수 가능하도록 enqueue API 1 곳에 격리 |
| content XSS | 서버 저장은 raw. 렌더는 React + escape (dangerouslySetInnerHTML 금지). 보안 보고서에 회귀 케이스 |
| 마이그레이션 후 기존 데이터 호환 | annotations 신규 테이블이라 backfill 불필요. notifications 동일 |

---

## 8. 폐쇄망 / 운영 고려

- 외부 fetch 없음 — 멘션 자동완성은 내부 user 검색만
- 알림 폴링 30s — SSE 가 폐쇄망 프록시 이슈 가능성 있어 본 FG 는 polling 채택
- audit emit 폭주 (주석 생성/해결/삭제) → 기존 emitter 의 batch 정책 그대로

---

*작성: 2026-04-27 | FG 3-3 인라인 주석 + 멘션 + MCP*
