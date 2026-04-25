# task1-2 — 일반 에디터 뷰 + 블록 뷰 + 토글 (TipTap 이식)

**작성일**: 2026-04-24
**Phase / FG**: S3 Phase 1 / FG 1-2
**상태**: 착수 대기 (FG 1-1 블로커)
**담당**: 프런트엔드
**예상 소요**: 3~5일
**선행 산출물**: `task1-1.md` 완료 (저장 모델 단일화 + 프런트 `saveDraft` PUT 전환)
**사용자 확정 결정**: D2 — TipTap + 커스텀 `node_id` extension

---

## 1. 작업 목적

Pre-flight 실측(§3.3) 에서 확인된 바와 같이 현재 에디터(`DocumentEditPage.tsx`, 401 LOC) 는 `DocumentNode[]` 를 `<input>`/`<textarea>` 로 편집하는 DIY 블록 편집기이며 **ProseMirror/TipTap 을 사용하지 않는다** (dependency 는 설치되어 있으나 import 0 건).

Phase 1 개발계획서 §1.2 가 요구하는 "단일 ProseMirror 트리 위에서 블록 뷰 ↔ 일반 뷰 토글" 을 만족하려면 **편집기 자체를 TipTap 위에 재구축** 해야 한다. 동시에 CLAUDE.md §4 의 "UI 리뷰 ≥ 5회 + 데스크탑/웹 호환" 요구, S2 원칙 ⑤(에이전트도 동등 소비자) 원칙과 충돌하지 않는 구조를 확보한다.

---

## 2. 작업 범위

### 2.1 포함

1. **TipTap Editor 컴포넌트 신설**
   - `frontend/src/features/editor/tiptap/DocumentTipTapEditor.tsx`
   - `@tiptap/react` + `@tiptap/starter-kit` + `@tiptap/extension-placeholder` 구성 (이미 설치됨)
   - 커스텀 `node_id` global extension — 모든 block-level node 에 `node_id` attribute 을 강제 (없으면 `crypto.randomUUID()` 로 생성, 있으면 보존)
2. **블록 뷰 구현** (현재 DIY UI 감성 유지)
   - TipTap `NodeViewRenderer` 로 heading/paragraph/list/code_block 각각을 "블록 카드" 로 래핑
   - 우측 hover 시 나타나는 이동(▲▼)/삭제(×) 버튼 유지
   - Block Add 툴바 유지(+ 섹션/+ 단락/+ 제목/+ 목록/+ 코드)
3. **일반 뷰 구현**
   - 동일 `EditorState`, `NodeView` 없이 TipTap 기본 렌더
   - 블록 카드 테두리 제거, 흐르는 리치텍스트
4. **뷰 토글 UI**
   - 헤더에 세그먼트 버튼 (블록 / 일반)
   - 전환 시 `editor.setOptions({ nodeViews: ... })` 또는 `dispatch({ decorations })` 로 재마운트 없이 적용
   - **`editor.getJSON()` 결과와 `node_id` attrs 는 토글 전후 동일** (FG 1-3 의 핵심 불변식)
5. **저장 경로 연결**
   - `editor.getJSON()` → FG 1-1 에서 도입된 `versionsApi.saveDraft(documentId, { content_snapshot, title })` PUT 호출
   - FG 1-1 의 임시 어댑터(`nodesToProseMirror.ts`) 는 본 FG 완료 시 **삭제**
6. **초기 로드 경로**
   - 우선: `versionsApi.getLatest` 응답의 `content_snapshot` 을 그대로 `editor.commands.setContent(snapshot)`
   - 폴백: `content_snapshot` 이 비표준(과거 `{type:"text"}`) 또는 null 이면 `nodesApi.list` 결과를 FG 1-1 의 `nodes_from_prosemirror` 역변환 유틸로 변환 (이 폴백은 로그 warning 과 함께)
7. **접근성 / 키보드**
   - 기본 TipTap 단축키(Ctrl/Cmd+B, I, ... ) 유지
   - 뷰 토글 단축키 `Cmd/Ctrl+Shift+V` (블록 ↔ 일반)
   - ARIA role/label: 세그먼트 버튼, 툴바 버튼 모두 aria-label 지정
   - 스크린 리더: 블록 카드의 drag handle/action button 은 `aria-hidden` 또는 대체 label
8. **자동 저장 / Cmd+S / dirty 표시**
   - 현행 `DocumentEditPage` 의 30초 auto-save, Ctrl+S, "저장됨 ✓/저장되지 않은 변경사항/저장 중/저장 실패" 상태 전부 이식
9. **UI 디자인 리뷰 ≥ 1회** (정책 완화: 2026-04-24)
   - CLAUDE.md 프로젝트 §5.2 "UI 검수는 최소 1회 이상" 을 Phase 1 운용 기준으로 적용
   - 각 회차의 스크린샷(또는 정적 코드 분석)과 지적사항 → 개선 내역을 로그로 기록

### 2.2 제외

- node_id stable 회귀 테스트 → FG 1-3 (본 FG 에서는 최소 smoke 1건만)
- 뷰 선호 저장(users.preferences) → FG 1-3
- 실시간 공동 편집 → S3 범위 밖
- 마크다운 단축키 확장 → Phase 2 Vault Import 와 겹치지 않게 보류
- 슬래시 커맨드 팔레트 → Phase 2 이후

### 2.3 하드코딩 금지 재확인

- 블록 타입 어휘(heading/paragraph/list/code_block/section) 는 **DocumentType schema 에서 정해진 chunking/rendering 계약** 과 1:1 대응해야 함
- 에디터 컴포넌트 내부에서 `if type === "POLICY"` 같은 DocumentType 분기 금지
- 뷰 토글 상태는 "블록" / "일반" 문자열을 코드에 직접 박지 말고 `EditorViewMode` enum (`"block" | "flow"`) 로 타입 고정

---

## 3. 선행 조건

- FG 1-1 완료:
  - `versionsApi.saveDraft` 가 PUT + `content_snapshot` 바디로 동작
  - 백엔드 `save_draft` 가 파생 동기화 호출 포함
  - `content_snapshot` validator 가 `{type:"doc",content:[...]}` 를 강제
  - Alembic backfill 완료로 기존 Draft 들의 `content_snapshot` 이 표준 포맷
- TipTap 3.x 가 React 19 strict mode 와 호환됨을 smoke 로 확인 (R4, 본 FG Step 0)

---

## 4. 구현 단계

### Step 0 — TipTap × React 19 smoke

- 최소 Editor 1개를 `/features/editor/tiptap/_smoke.tsx` 에 렌더하고 `next dev` 로 기동
- `React.StrictMode` 하에서 에러 없이 포커스/입력 가능한지 확인
- 실패 시: `@tiptap/react` 3.x 최신 패치 확인. 대안으로 `@tiptap/react` 2.x 호환 모드 없으므로 React 19 RC 이슈 트래커 검토

### Step 1 — node_id extension 신설

- `frontend/src/features/editor/tiptap/extensions/NodeId.ts`
- `Extension.create` 패턴 + `addGlobalAttributes` 로 heading/paragraph/bulletList/orderedList/codeBlock 등 block-level 노드에 `node_id` 적용
- `renderHTML` / `parseHTML` 설정 — HTML 왕복 시 `data-node-id` 보존
- 신규 노드 생성 시 `crypto.randomUUID()` (input rule 또는 `onTransaction` hook 에서 최종 보정)

### Step 2 — DocumentTipTapEditor 컴포넌트

- Props:
  ```ts
  interface Props {
    initialContent: ProseMirrorDoc;
    onChange: (doc: ProseMirrorDoc) => void;
    viewMode: "block" | "flow";
    readOnly?: boolean;
  }
  ```
- StarterKit + Placeholder + NodeId
- `viewMode === "block"` 일 때만 `nodeViews` 옵션에 BlockCardNodeView 등록
- `useEffect(() => editor?.setOptions({ nodeViews: viewMode === "block" ? blockNodeViews : {} }), [viewMode])`

### Step 3 — 블록 카드 NodeView

- `frontend/src/features/editor/tiptap/nodeviews/BlockCardView.tsx`
- 카드 테두리 + hover 시 ▲▼× 버튼
- 이동/삭제는 ProseMirror transaction(`tr.move`, `tr.delete`) 로 실행
- 현재 Tailwind 클래스/감성 유지

### Step 4 — DocumentEditPage 재작성

- 기존 `DocumentEditPage.tsx` 의 헤더/툴바/상태칩/저장 로직은 유지
- 에디터 본문을 `DocumentTipTapEditor` 로 교체
- `nodes` state 제거. `doc: ProseMirrorDoc` state 로 통일
- FG 1-1 의 어댑터(`nodesToProseMirror`) 삭제

### Step 5 — 뷰 토글 UI

- 헤더 우측에 세그먼트 버튼 2개(블록 / 일반)
- `useState<EditorViewMode>` 로 로컬 상태 (FG 1-3 에서 users.preferences 와 연동 후 복원)
- Cmd/Ctrl+Shift+V 단축키
- 전환 시 `editor.getJSON()` 로그로 node_id 불변 smoke (FG 1-3 의 회귀 본체와 별개, 개발자 확인용)

### Step 6 — 접근성

- `aria-pressed` 로 세그먼트 선택 상태 노출
- 블록 카드의 ▲▼× 버튼 각각 `aria-label="위로 이동" / "아래로 이동" / "블록 삭제"`
- 모바일 breakpoint: 헤더/툴바가 `sm` 이하에서 wrap 되도록 (기능 차단 아님)

### Step 7 — UI 디자인 리뷰 ≥ 1회 (정책 완화: 2026-04-24)

각 회차에서 (a) 스크린샷 또는 정적 코드 분석 (b) 지적사항 (c) 수정 diff 요약을 `산출물/FG1-2_UI리뷰로그.md` 에 기록. 1회 수행 시 아래 5축을 한 번에 훑는 형태도 허용:

| 축 | 주 내용 | 예 |
|----|--------|----|
| 1 | 블록 카드 간격/테두리/hover 대비 | `border-gray-200` 대비 충분 여부 |
| 2 | 일반 뷰 서체/줄간격/인용문 스타일 | heading 1~3 시각 위계 |
| 3 | 뷰 토글 위치/레이블/세그먼트 활성 스타일 | 데스크탑/좁은 창 모두 |
| 4 | 접근성 검증(키보드만으로 블록 추가/이동/삭제, 토글, 저장) | aria-* 부재 잡기 |
| 5 | 저장 상태 피드백(저장 중/저장됨/실패) 명도 대비, 실패 시 재시도 UX | WCAG AA |

### Step 8 — 테스트 (node:test)

- `tests/tiptap_node_id.test.ts`: NodeId extension 이 신규 노드에 UUID 부여, 기존 id 보존
- `tests/document_tiptap_editor.render.test.ts`: viewMode=block/flow 토글 시 `editor.getJSON()` 의 `attrs.node_id` 집합이 동일
- `tests/document_edit_page.save.test.ts`: 저장 시 PUT `/draft` 호출되고 바디에 `content_snapshot` 포함 (FG 1-1 테스트와 중복 가능하나 통합 확인 의미)
- 회귀: 기존 node:test 전부 녹색 유지

### Step 9 — 보안 / 의존성 점검

- TipTap 은 입력 content 에 대해 HTML sanitize 를 수행하지 않음. **ProseMirror schema 를 통과한 JSON 만 수용** 되므로 `<script>` 태그는 자연 배제. 다만 URL/href 처리 시 `dompurify` 로 링크 sanitize (FG 1-2 범위에서는 링크 확장을 추가하지 않으므로 hook 만 준비)
- `npm audit --audit-level=high` 결과 0건 확인 (CLAUDE.md 전역 규칙 2)
- critical/high 취약점 있는 패키지는 **설치 금지** (CLAUDE.md 전역 규칙 2)

---

## 5. 검수 기준

| # | 기준 | 근거 |
|---|------|------|
| C1 | 에디터가 TipTap 기반으로 동작 + 초기 로드가 PUT 응답 content_snapshot 을 사용 | 수동 smoke + node:test |
| C2 | 블록 / 일반 뷰 토글이 재마운트 없이 동작 | 수동 smoke (포커스 유지 확인) |
| C3 | 토글 전/후 `editor.getJSON()` 의 node_id 집합 동일 | node:test `document_tiptap_editor.render.test.ts` |
| C4 | 저장이 PUT `/documents/{id}/draft` + content_snapshot 바디로 수행 | node:test + 네트워크 로그 |
| C5 | Ctrl/Cmd+S, 30초 auto-save, dirty 표시 현행 동등 | 수동 smoke + 기존 테스트 호환 |
| C6 | UI 리뷰 ≥ 1회 로그 제출 (스크린샷 또는 정적 분석 + 지적사항 + 개선 내역) — 정책 완화 2026-04-24 | `산출물/FG1-2_UI리뷰로그.md` |
| C7 | 데스크탑 / 좁은 창(≤ 768px) 에서 레이아웃 깨짐 없음 | 수동 smoke 2종 |
| C8 | aria-label/aria-pressed 적용 및 키보드만으로 전체 동작 가능 | 수동 smoke |
| C9 | 보안: `npm audit --audit-level=high` 0건, TipTap 의존성에 critical 0건 | `산출물/FG1-2_보안취약점검사보고서.md` |
| C10 | node:test 전부 녹색, pytest 베이스라인 변동 없음 (본 FG 는 프런트 중심) | 실 실행 로그 |

---

## 6. 산출물

| 파일 | 설명 |
|------|------|
| `docs/개발문서/S3/phase1/산출물/FG1-2_검수보고서.md` | C1~C10 체크리스트 + 실 실행 로그 |
| `docs/개발문서/S3/phase1/산출물/FG1-2_보안취약점검사보고서.md` | `npm audit` 결과, TipTap/ProseMirror CVE 조사, 링크/URL sanitize 미래 대응 |
| `docs/개발문서/S3/phase1/산출물/FG1-2_UI리뷰로그.md` | 리뷰 5회 × (스크린샷 + 지적사항 + 수정) |

---

## 7. 리스크 / 열린 질문

| # | 리스크 | 대응 |
|---|-------|-----|
| R1 | TipTap 3.x × React 19 호환 문제 | Step 0 smoke 로 조기 확인. 실패 시 임시로 Compat Provider 검토 |
| R2 | node_id extension 이 list 안의 listItem 같은 nested block 에서 누락 | NodeId extension 적용 타입을 명시적으로 열거 + 테스트 커버리지 |
| R3 | NodeView(블록 카드) 전환 시 포커스 유실 | `decorations` 기반 + `NodeView` 최소 재생성 경로 연구 |
| R4 | 어댑터 삭제 타이밍 (FG 1-1 의 `nodesToProseMirror`) | 본 FG 의 Step 4 마지막에 import 경유 제거 확인 |
| R5 | 모바일 breakpoint 축소 시 블록 카드 hover 버튼 불가 | 모바일은 항상 visible 또는 long-press 대안 (Phase 1 모바일은 "차단 안 하는 수준" 까지) |
| Q1 | 일반 뷰에서도 이동/삭제 버튼을 노출할지? (현재는 블록 뷰 전용 가정) | UI 리뷰 1회차에서 결정. 기본 권장: 일반 뷰는 흐름 중심이라 미노출 |
| Q2 | 뷰 토글 단축키 충돌 여부 (`Cmd/Ctrl+Shift+V` 는 일부 OS 에서 "붙여넣기 일반 텍스트" 예약) | 단축키는 재협상 가능 — 브라우저 기본 동작 확인 후 확정 |

---

## 8. 참조

- `docs/개발문서/S3/phase1/Phase 1 개발계획서.md` §1~§2 FG 1-2
- `docs/개발문서/S3/phase1/산출물/Pre-flight_실측.md` §3, §6.2
- `CLAUDE.md` §4 UI 규칙 + 전역 규칙 2(취약점)
- TipTap 공식 문서 (Extension / NodeView / globalAttributes)
- FG 1-1 작업지시서 (`task1-1.md`)

---

*작성: 2026-04-24 | Phase 1 FG 1-2 착수 대기 (FG 1-1 블로커)*
