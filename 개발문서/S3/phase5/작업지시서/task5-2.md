# task5-2 — AnnotationMark + Gutter + Panel 연동 (FG 5-2)

**작성일**: 2026-05-10
**Phase / FG**: S3 Phase 5 / FG 5-2
**상태**: 착수 대기 (FG 5-1 ADR §a 의 AnnotationMark schema 결정 완료 후)
**담당**: 설계 Claude / 구현 Claude (또는 Codex — frontend 비중 큼) / 리뷰 Codex (extended — UI + ACL 영향)
**예상 소요**: 4~5일 (TipTap extension + Gutter UI + Panel 양방향 연동 + 회귀)
**선행 산출물**:
- `task5-1` 완료 (Mark 통합 ADR + AnnotationMark schema spec)
- `docs/개발문서/S3/phase5/산출물/Mark_통합_ADR.md`
**후행**: task5-3 (멘션 typeahead) 와 독립. task5-4 (사이드바 재구조화) 는 본 FG 산출물 위에서 panel 마운트

---

## 1. 작업 목적

Phase 3 FG 3-3 의 AnnotationsPanel 은 `defaultNodeId` 를 수동 입력해야 했고, 본문 click → panel highlight 양방향 연동이 미구현 상태였다 (FG 3-3 종결보고서 §"잔존 작업"). 본 FG 가 이 잔존을 닫는다:

1. **AnnotationMark** — TipTap inline mark 신설. 본문에 `data-annotation-id` 속성으로 인라인 주석 영역 표시.
2. **AnnotationGutter** — 좌측 여백에 노드별 주석 카운트 도트. 클릭 시 해당 node_id 의 주석으로 panel scroll.
3. **본문 ↔ Panel 양방향 연동** — mark 클릭 → panel highlight + scroll, panel 항목 클릭 → 본문 mark highlight + scroll (R-A3 — 본문 편집 차단 없음).

본 FG 는 schema 변경 없음 — annotations / annotation_mentions / notifications 테이블은 Phase 3 그대로. mark `annotation_id` 는 기존 `annotations.id` 를 참조한다.

---

## 2. 작업 범위

### 2.1 포함

#### 2.1.1 AnnotationMark TipTap extension 신설

`frontend/src/features/editor/tiptap/extensions/AnnotationMark.ts` (신규) — task5-1 ADR §d / §e 결정 적용:

```ts
import { Mark, mergeAttributes } from "@tiptap/core";
import { ANNOTATION_MARK_NAME, ANNOTATION_DATA_ATTR } from "../markNames";

export interface AnnotationMarkAttrs {
  annotation_id: string; // uuid — annotations.id
}

export const AnnotationMark = Mark.create<{}, { onAnnotationClick?: (id: string) => void }>({
  name: ANNOTATION_MARK_NAME,
  inclusive: false, // ADR §e
  excludes: "",      // 다른 mark 와 결합 가능 (ADR §b C1~C3)

  addAttributes() {
    return {
      annotation_id: {
        default: null,
        parseHTML: (el) => el.getAttribute(ANNOTATION_DATA_ATTR),
        renderHTML: (attrs) =>
          attrs.annotation_id ? { [ANNOTATION_DATA_ATTR]: attrs.annotation_id } : {},
      },
    };
  },

  parseHTML() {
    return [{ tag: `span[${ANNOTATION_DATA_ATTR}]` }];
  },

  renderHTML({ HTMLAttributes }) {
    return ["span", mergeAttributes(HTMLAttributes, { class: "annotation-mark" }), 0];
  },

  // 클릭 → panel 활성화. R-A3 — selection / cursor 영향 없음
  addProseMirrorPlugins() {
    // task5-1 ADR §b C4 결정에 따라 plugin 으로 click 이벤트만 forward.
    // selection / cursor 는 TipTap default 동작 그대로.
    // 자세한 구현은 §2.1.3.
  },
});
```

- `markNames.ts` 의 `ANNOTATION_MARK_NAME` / `ANNOTATION_DATA_ATTR` 사용 (task5-1 §2.1.2).
- `inclusive: false`: 주석 끝에서 입력한 텍스트가 mark 에 자동 흡수되지 않음.
- `class: "annotation-mark"`: CSS 시각 강조 단일 정본.

#### 2.1.2 CSS / 색상 / 시각 스타일

`frontend/src/app/globals.css` (또는 동등) 에 `.annotation-mark` 스타일 추가:
- 기본: 노란색 배경 (예: `bg-yellow-100/60`) + 하단 점선 (`border-b border-dotted border-yellow-500`)
- hover: 명도 강조 (`bg-yellow-200`)
- 활성 (panel 에서 선택된 항목): `.annotation-mark.is-active { background: bg-yellow-300; outline: 1px solid amber-500 }`
- HashtagMark 와 동시 적용 (`.tag-pill.annotation-mark` 또는 nested) 시 두 시각 모두 보존 — UI 디자인 리뷰 1회 의무 (Phase 5 §6.2 보류 결정)

> **UI 디자인 리뷰 ≥ 1회** 의무 (Phase 5 §4 산출물 규약). 4 viewport (desktop / narrow / tablet / mobile) 에서 시각 검증. 색상/스타일 결정은 본 FG 의 1차 디자인 리뷰에서 확정.

#### 2.1.3 본문 click → AnnotationsPanel 활성화 양방향 연동

##### a. mark click 이벤트 plugin

ProseMirror plugin 으로 `mousedown` / `click` 이벤트 처리:
- 클릭한 DOM 요소가 `[data-annotation-id]` 를 가지면 `annotation_id` 추출
- TipTap selection / cursor 는 **변경하지 않는다** (R-A3) — TipTap default 동작 보존
- 외부 콜백 호출: `editor.options.onAnnotationClick?.(annotation_id)` 또는 EventBus / Zustand store
- preventDefault 사용 금지 (편집 흐름 차단 금지)

##### b. DocumentTipTapEditor → AnnotationsPanel state 연결

`frontend/src/features/documents/DocumentDetailPage.tsx`:
- `selectedAnnotationId: string | null` state 추가
- `<DocumentTipTapEditor onAnnotationClick={setSelectedAnnotationId} ... />`
- `<AnnotationsPanel ... selectedAnnotationId={selectedAnnotationId} onSelectAnnotation={setSelectedAnnotationId} />` (props 확장)

##### c. AnnotationsPanel — 선택 항목 highlight + scroll

`frontend/src/features/documents/AnnotationsPanel.tsx`:
- 신규 prop `selectedAnnotationId?: string | null`
- 선택된 항목에 `ring-2 ring-amber-500` + scrollIntoView (`block: "center"`)
- panel 자체가 collapsed 상태였다면 자동 펼침 (Phase 5 §1.4 기대 결과)
- 선택 항목 클릭 시 `onSelectAnnotation(id)` 호출 → 본문 mark 도 `is-active` 로 표시

##### d. panel → 본문 mark highlight

panel 항목 클릭 시:
- 같은 `selectedAnnotationId` 가 본문 `[data-annotation-id]` 요소에 `.is-active` 클래스 부여 + `scrollIntoView`
- editor 외부에서 DOM 직접 조작은 위험 — TipTap NodeView 또는 React decoration plugin 사용 권장. 본 FG 에서 결정 후 ADR (§2.1.7) 에 명시.

#### 2.1.4 AnnotationGutter — 좌측 여백 카운트 도트

`frontend/src/features/documents/AnnotationGutter.tsx` (신규):

##### a. 데이터

- input prop: `annotationsByNode: Record<string /* node_id */, number /* count */>`
- annotations API 응답에서 node_id 별 그룹화. `useDocumentAnnotations(documentId)` 훅에서 derived state 로 노출.

##### b. 렌더링

- 본문 좌측 ~24px 여백에 절대 위치
- 각 노드 (heading / paragraph 등) 의 top offset 을 `data-node-id` 요소의 `getBoundingClientRect()` 로 계산
- node_id 에 주석이 있으면 도트 (• ` size, 채워진 색) 표시. 카운트 ≥ 2 면 도트 + 숫자
- 도트 클릭 → `onSelectAnnotation(첫 주석 id)` (panel highlight + scroll)
- 해결됨 주석만 있으면 흐린 도트 (opacity 50%)

##### c. 스크롤 / 리사이즈 동기화

- 본문 스크롤 또는 윈도우 리사이즈 시 도트 위치 재계산. `ResizeObserver` + `IntersectionObserver` 조합
- 빈번한 reflow 방지 — debounce 50ms

##### d. 접근성

- 각 도트는 `<button>` (`aria-label="이 노드 주석 N개 — 펼치기"`)
- 키보드 포커스 가능, Enter/Space 로 활성화

#### 2.1.5 노드 단위 주석 카운트 derived state

`frontend/src/features/documents/hooks/useDocumentAnnotations.ts` (신규 또는 기존 확장):
- 기존 `useAnnotations(documentId)` 훅 (FG 3-3 산출) 위에 derived selector 추가
- `annotationsByNode: Record<string, AnnotationSummary[]>`
- AnnotationsPanel + AnnotationGutter 가 같은 source 사용 (단일 정본)

#### 2.1.6 AnnotationMark attribute 영속화 — save_draft 경로

- TipTap doc 에 AnnotationMark 가 들어가면 ProseMirror JSON 의 mark 배열에 `{ type: "annotation", attrs: { annotation_id: "..." } }` 가 직렬화됨
- `save_draft` 가 이 mark 를 그대로 저장 — backend 변경 없음 (content_snapshot 에 자연 저장)
- 단, mark 가 가리키는 `annotation_id` 가 annotations 테이블에 실존하는지 검증 — **클라이언트 측 검증** (저장 전 invalid id 제거)
  - 신설 함수 `cleanInvalidAnnotationMarks(doc, validIds): doc` — `frontend/src/features/editor/tiptap/sanitize.ts` (신규)
  - 호출 지점: save_draft 직전 (markOrder 정규화와 동시 — task5-1 §2.1.3 (ii) 채택 시 같은 모듈)
- backend `snapshot_sync_service.py` 는 mark 무관하게 동작 (현행 유지). 단, 별 회귀 1건 — `test_save_draft_preserves_annotation_marks.py` (신규)

#### 2.1.7 작은 ADR — Decoration 방식 선택

panel → 본문 mark highlight (`.is-active`) 적용 방식이 두 가지:
- (A) `editor.commands.updateAttributes('annotation', { is_active: true })` — schema attribute 로 영속화. 단, `is_active` 가 직렬화에 흘러가면 안 됨 → DB 저장 위험
- (B) ProseMirror `Decoration.inline` plugin — 임시 시각 마킹, 직렬화 안 됨 (권장)

본 FG 진입 시 (B) 채택을 default 로 하되, 1차 구현 결과를 ADR `Annotation_Decoration_방식.md` 로 짧게 기록 (1쪽 이내). task5-1 ADR 의 부록으로 첨부.

### 2.2 제외 (이월)

- **Span-level annotation drag-to-select** — Phase 5 §6.1 명시적 제외. 본 FG 는 `span_start/end` 가 이미 들어 있는 주석을 mark 로 시각화만.
- **Annotation 본문 rich text** — Phase 5 §6.1 명시적 제외.
- **답글 깊이 2+ (스레드 nesting)** — Phase 5 §6.1.
- **Notifications grouping / digest** — 별 라운드.
- **AnnotationMark 의 `mimir://` URI 라우팅** — task5-3 / Phase 4 FG 4-1 산출물과 연동 (별 task — Phase 5 §0).

### 2.3 하드코딩 금지 재확인

- mark name / data attribute / CSS class 는 task5-1 의 `markNames.ts` 단일 정본
- 색상 / 스타일 token 은 globals.css 의 CSS variable 또는 Tailwind config 에 단일 정본 — 컴포넌트 인라인 색상 금지

---

## 3. 선행 조건

- task5-1 의 ADR §a / §d / §e 가 AnnotationMark schema 결정 완료
- `docs/함수도서관/frontend.md` 의 TipTap extension / annotations 섹션 위치 확인
- Phase 3 annotations API (`/api/v1/documents/{id}/annotations`) 안정 — FG 3-3 종결보고서 회귀 녹색 유지
- AnnotationsPanel (`AnnotationsPanel.tsx`, 410줄) 가 `defaultNodeId` 외에 `selectedAnnotationId` / `onSelectAnnotation` props 를 받을 수 있도록 확장 가능

---

## 4. 구현 단계

### Step 1 — AnnotationMark extension 작성

1. `extensions/AnnotationMark.ts` 신설 (§2.1.1)
2. `markNames.ts` 의 상수 등록 (task5-1 산출물에 추가)
3. `DocumentTipTapEditor.tsx` 의 extensions 배열에 등록
4. 단위 테스트 — 기본 schema, attribute parse/render, inclusive 동작 ≥ 5 시나리오
5. tsc 0 error

### Step 2 — CSS / 시각 스타일 (§2.1.2)

1. `globals.css` 에 `.annotation-mark` 스타일
2. UI 디자인 리뷰 1회 — 4 viewport 검증 + HashtagMark 와 동시 적용 시각 검증
3. 색상 / 스타일 결정 후 ADR (§2.1.7 의 부록 또는 별 디자인 결정 메모) 기록

### Step 3 — 본문 click → panel 양방향 (§2.1.3)

1. ProseMirror plugin — click 이벤트 forward
2. `DocumentDetailPage.tsx` state 연결
3. `AnnotationsPanel.tsx` props 확장 (`selectedAnnotationId` / `onSelectAnnotation`)
4. panel → 본문 highlight (Decoration plugin §2.1.7 (B))
5. 단위 테스트 — click → callback ≥ 4 시나리오 (mark click / outside click / hover / 다중 mark)

### Step 4 — AnnotationGutter (§2.1.4)

1. `AnnotationGutter.tsx` 신설 + 좌측 여백 도트 렌더링
2. `useDocumentAnnotations` 훅 derived state 추가 (§2.1.5)
3. ResizeObserver / IntersectionObserver 연결
4. 접근성 — 키보드 / aria-label
5. 단위 테스트 — 카운트 정확성 + 클릭 → onSelectAnnotation ≥ 5 시나리오
6. UI 디자인 리뷰 1회 — 좁은 화면 fallback (모바일 에서 gutter 가 본문 가리지 않도록)

### Step 5 — save_draft 경로 보호 (§2.1.6)

1. `sanitize.ts` 신설 — `cleanInvalidAnnotationMarks`
2. save_draft 직전 호출 + 단위 테스트 ≥ 4 시나리오 (valid only / invalid only / mixed / empty)
3. backend 회귀 — `test_save_draft_preserves_annotation_marks.py` 신설 (≥ 2 시나리오)

### Step 6 — Decoration 방식 ADR 부록 (§2.1.7)

1. 1쪽 이내 결정서 — `산출물/Annotation_Decoration_방식.md`
2. task5-1 ADR 본문에서 본 부록을 cross-link

### Step 7 — 회귀 / 베이스라인 / 함수도서관

1. Phase 1 NodeId 안정성 회귀 녹색
2. Phase 3 annotations 회귀 (anchoring 4 시나리오) 녹색
3. task5-1 의 round-trip 회귀 (S5 / S10) 녹색
4. node:test 신규 ≥ 18 + 베이스라인 유지
5. pytest 신규 ≥ 2 + 베이스라인 유지
6. tsc 0 error
7. `docs/함수도서관/frontend.md` 갱신 — AnnotationMark / AnnotationGutter / sanitize 항목 추가

### Step 8 — 검수 / 보안 보고서

- `FG5-2_검수보고서.md` — R-A1 / R-A2 / R-A3 준수 확인 (특히 R-A3 — mark 클릭이 cursor 영향 없음 회귀)
- `FG5-2_보안취약점검사보고서.md`:
  - mark `annotation_id` 가 다른 organization 의 주석을 가리킬 때 차단 (annotations API 의 ACL 이 정본이지만, mark 영속화 시 stale id 위험)
  - XSS — `data-annotation-id` 가 사용자 입력 경로로 들어올 때 sanitize 검증
  - DOM injection — Decoration plugin 이 사용자 제공 HTML 을 마운트하지 않는지 검증

---

## 5. API 계약 변경 요약

| 메서드 | 경로 | 변경 |
|-------|------|------|
| (없음) | (없음) | 본 FG 는 frontend 전용. backend annotations API 변경 없음. |

단, save_draft 응답 본문 round-trip 동등은 task5-1 회귀로 보장.

---

## 6. 데이터 모델 주의사항

- DB 스키마 변경 없음.
- ProseMirror JSON 의 mark 영속화에 `{ type: "annotation", attrs: { annotation_id } }` 추가됨 — 기존 doc 에는 부재. 즉 마이그레이션 불필요 (mark 부재 = panel 만 사용 = 후방 호환).
- `annotation_id` 는 `annotations.id` 의 외부 참조이지만 DB 외부 (JSONB) 에 위치 — 무결성은 클라이언트 sanitize (§2.1.6) 로 보장. annotation 삭제 시 본문 mark 가 stale 상태 가능 → save_draft 경로에서 정리.

---

## 7. 성공 기준

- [ ] AnnotationMark extension 등록 + tsc 0 error
- [ ] CSS / 시각 스타일 + UI 디자인 리뷰 1회 통과 (4 viewport)
- [ ] 본문 mark click → AnnotationsPanel 자동 펼침 + 해당 주석 highlight + scroll
- [ ] panel 항목 click → 본문 mark `.is-active` highlight + scroll (Decoration plugin)
- [ ] AnnotationGutter 노드별 카운트 정확 + 도트 click → panel highlight
- [ ] **R-A3** — mark click 이 cursor / selection / 본문 편집 차단하지 않음 (회귀 ≥ 4 시나리오)
- [ ] **R-A1** — AnnotationMark 가 task5-1 round-trip 회귀 S5 녹색
- [ ] save_draft sanitize — invalid annotation_id 제거 ≥ 4 시나리오 녹색
- [ ] Phase 1 NodeId 안정성 회귀 녹색
- [ ] Phase 3 annotations anchoring 4 시나리오 녹색
- [ ] node:test 신규 ≥ 18 / 베이스라인 유지
- [ ] pytest 신규 ≥ 2 / 베이스라인 유지
- [ ] `docs/함수도서관/frontend.md` 갱신
- [ ] `FG5-2_검수보고서.md` + `FG5-2_보안취약점검사보고서.md` 제출
- [ ] `Annotation_Decoration_방식.md` ADR 부록 제출

---

## 8. 리스크

| # | 리스크 | 대응 |
|---|-------|-----|
| R-01 | mark click plugin 이 cursor 위치 변경 (R-A3 위반) | TipTap default mark 동작 보존 — preventDefault 금지. 회귀 ≥ 4 시나리오 |
| R-02 | Decoration plugin 이 mark 와 충돌 → 본문 깜빡임 | (B) 방식 선택 (§2.1.7) + 1차 구현 후 시각 회귀 1회. 깜빡임 발견 시 React state 의 `is-active` 를 CSS-only 로 처리 |
| R-03 | AnnotationGutter 의 ResizeObserver 가 모든 paragraph 에서 발화 → 성능 저하 | debounce 50ms + 가시 영역 (IntersectionObserver) 만 추적 |
| R-04 | annotation 삭제 후 본문 mark 가 stale → click 시 404 | sanitize §2.1.6 + panel UI 에서 stale mark 시각 강조 (예: 점선만 남김) |
| R-05 | Decoration plugin 이 직렬화에 누설 → DB 에 `is_active` 저장 | (B) 방식은 직렬화 무관. 단위 테스트 — Decoration 적용 후 save_draft → JSON 비교 동등 |
| R-06 | HashtagMark 와 AnnotationMark 동시 적용 시 시각 깨짐 | UI 디자인 리뷰 1회 — nested 시각 검증. CSS 우선순위는 task5-1 ADR §d 표 |
| R-07 | annotation_id 가 다른 organization 의 주석을 가리킴 (URL 공유 / 악의적 입력) | annotations GET API 의 ACL 이 정본 — mark 클릭 시 fetch 가 403/404 반환하면 panel 에서 "접근 불가" 메시지. mark 자체가 권한 우회 안 함 |
| R-08 | 좁은 화면에서 gutter 가 본문 가림 | UI 디자인 리뷰 4 viewport 의무. 모바일 fallback — gutter 숨김 + panel 의 node_id 텍스트로 대체 |

---

## 9. 참조

- `docs/개발문서/S3/phase5/Phase 5 개발계획서.md` §1.2 (R-A3), §2.1 (FG 5-2), §6.2 (보류 결정 — 색상/스타일)
- `docs/개발문서/S3/phase5/작업지시서/task5-1.md` §2.1.1 §2.1.2 (markNames / schema)
- `docs/개발문서/S3/phase3/산출물/FG3-3_종결보고서.md` (annotations API + AnnotationsPanel)
- `frontend/src/features/documents/AnnotationsPanel.tsx`
- `frontend/src/features/documents/DocumentDetailPage.tsx`
- `frontend/src/features/editor/tiptap/extensions/HashtagMark.ts` (mark 패턴 참고)
- `frontend/src/features/editor/tiptap/DocumentTipTapEditor.tsx`
- `backend/app/api/v1/annotations.py`
- `CONSTITUTION.md` 제5·9·11·12·15조

---

*작성: 2026-05-10 | FG 5-2 — AnnotationMark + Gutter + Panel 양방향 연동*
