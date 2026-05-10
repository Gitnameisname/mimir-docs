# Annotation Decoration 방식 — Mark 통합 ADR §2.1.7 부록

**작성일**: 2026-05-11
**대상**: task5-2 §2.1.7 (Decoration 방식 결정서)
**상위 ADR**: `Mark_통합_ADR.md`

---

## 1. 결정

**panel → 본문 highlight (`.is-active` 클래스 토글) 방식: (B') 외부 DOM 조작**.

ADR §2.1.7 의 두 후보 (A: schema attribute / B: ProseMirror Decoration plugin) 중 **B 의 변형 — 외부 DOM 직접 조작** 채택.

```
A. schema attribute       → 직렬화 누설 위험. 거부.
B. Decoration plugin      → ProseMirror 정통. 채택 가능.
B'. 외부 DOM 조작 (채택)  → React useEffect 에서 syncAnnotationActiveDom() 호출.
                           Decoration plugin 보다 React state 통합이 단순.
```

## 2. 채택 사유

1. **직렬화 누설 위험 0** — `.is-active` 는 어디까지나 **DOM 클래스**. ProseMirror state / JSON 직렬화에 미반영. round-trip 안전.
2. **React state 와의 통합 단순** — `selectedAnnotationId` state 가 변하면 `useEffect` 에서 `syncAnnotationActiveDom(wrapperRef.current, selectedAnnotationId)` 호출. PluginKey 별 state 관리 불필요.
3. **R-A3 보존** — TipTap selection / cursor 동작에 영향 없음. Decoration 도 영향 없으나, 외부 DOM 조작은 더 단순.
4. **테스트 용이성** — 단순 querySelectorAll + classList 토글이라 jsdom 미설치 환경에서도 mock DOM 으로 단위 회귀 가능 (`AnnotationMarkFg52.test.tsx`).

## 3. 단점 / 한계

1. **ProseMirror 정통은 Decoration** — 본 변형은 외부 DOM 의존. ProseMirror state mutation 시 (예: 빠른 입력 + setContent 동시 발생) 에 race 가능성 — 단 `.is-active` 는 시각만이라 실해 영향 미미.
2. **wrapper ref 의존** — DocumentTipTapEditor 가 wrapper `<div ref>` 를 노출해야 sync 가능. 다른 컴포넌트가 같은 본문을 표시하면 별 sync 호출 필요.

## 4. 구현 위치

- `frontend/src/features/editor/tiptap/extensions/AnnotationMark.ts::syncAnnotationActiveDom(root, selectedId)` — 핵심 helper
- `frontend/src/features/editor/tiptap/DocumentTipTapEditor.tsx::useEffect` — `selectedAnnotationId` 변화 시 호출
- `frontend/tests/AnnotationMarkFg52.test.tsx::syncAnnotationActiveDom — DOM 조작 패턴` — 3 회귀 (선택 / null / 같은 id 여러 위치)

## 5. 향후 변경 가이드

- **Decoration plugin 으로 전환 필요 시점**: 본문 mark 의 시각 상태가 ProseMirror state 와 강한 결합이 필요해질 때 (예: 다중 선택 / 키보드 네비). 그 시점에 본 변형을 Decoration plugin 으로 마이그레이션. 검수보고서에서 명시.
- **현재 (FG 5-2)**: 단일 선택만 — 본 변형이 충분.

---

*작성: 2026-05-11 | Mark 통합 ADR 부록 — Annotation Decoration 방식*
