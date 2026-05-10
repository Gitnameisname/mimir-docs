# Mark Inventory 실측 보고서 — S3 Phase 5 FG 5-1

**작성일**: 2026-05-11
**대상**: task5-1 §2.1.0 — Mark/Extension 사실 baseline
**범위**: Mimir frontend 의 ProseMirror schema 에 등록된 모든 mark / global attribute 의 정본 사실 기반.

본 보고서는 **사실(현 코드 상태)** 을 기록한다. 결정/규범은 별도 ADR (`Mark_통합_ADR.md`).

---

## 1. 요약

| 항목 | 종류 | 상태 | 출처 FG | 본 Phase 5 영향 |
|------|------|----|--------|----------------|
| `nodeId` (Global Attribute) | 노드 속성 | ✅ 구현 완료 | Phase 1 FG 1-2 | 변경 없음 |
| `hashtag` (Inline Mark) | inline mark | ✅ 구현 완료 | Phase 2 FG 2-2 | schema 명시화 (§4) |
| `wikilink` (Inline Mark) | inline mark | ✅ 구현 완료 (Phase 2 FG 2-3 재진입, 2026-05-10) | Phase 2 FG 2-3 | schema 명시화 (§4) |
| `annotation` (Inline Mark) | inline mark | 🟡 미구현 — task5-2 (FG 5-2) 에서 신설 | Phase 5 FG 5-2 | schema spec 만 ADR 에 명시 |

> Phase 5 task5-1 작성 시점 (2026-04-27) 의 가정 — "S3 Phase 1~3 에서 도입한 4 종 mark" — 와 본 실측 결과의 정합:
>
> - WikiLinkMark 는 **2026-05-10 본 세션에서 FG 2-3 재진입으로 종결** (`docs/개발문서/S3/phase2/산출물/FG2-3_종결보고서.md`).
> - AnnotationMark 만 미구현 — task5-2 의 신설 책임.

---

## 2. NodeId — Global Attribute (Phase 1 FG 1-2)

**파일**: `frontend/src/features/editor/tiptap/extensions/NodeId.ts` (135 lines)
**종류**: TipTap `Extension` (Mark 아님 — `addGlobalAttributes` 로 노드 속성 추가)

### 2.1 attribute

| key | 값 | parseHTML | renderHTML |
|----|---|-----------|------------|
| `node_id` | UUID string (snake_case 필수) | `data-node-id` 속성 | `data-node-id` 속성 |

- `keepOnSplit: false` — Enter 로 블록 분할 시 새 블록은 id 미부여 → `appendTransaction` 이 신규 id 자동 부여.
- 적용 대상 노드 타입 (`options.types`): **`heading`, `paragraph`, `bulletList`, `orderedList`, `codeBlock`** (5 종).
- `addProseMirrorPlugins` — id 없는 블록 발견 시 transaction 으로 UUID 부여 (loop 방지 PluginKey 사용).

### 2.2 호출 흐름

- 블록 분할 / paste / `setContent` / 신규 입력 모든 경로에서 자동 id 부여.
- 서버 저장 시 `versions.content_snapshot` JSONB 의 attrs.node_id 로 직렬화. backend `snapshot_sync_service.nodes_from_prosemirror` 가 이 키를 읽어 `nodes` 테이블 동기화.

### 2.3 본 Phase 5 영향

**없음**. Phase 1 의 NodeId 안정성 회귀가 Phase 5 의 R-A1 (직렬화 round-trip) 의 baseline 으로 작용.

---

## 3. HashtagMark — Inline Mark (Phase 2 FG 2-2)

**파일**: `frontend/src/features/editor/tiptap/extensions/HashtagMark.ts` (125 lines)
**종류**: TipTap `Mark`
**관련 backend**: `app/services/tag_rules.py` (서버 파서 정본).

### 3.1 schema

| 속성 | 값 |
|------|----|
| `name` | `"hashtag"` |
| `inclusive` | `false` |
| `excludes` | (미지정 → default 빈 문자열) |
| attribute `tag` | string. `data-tag` 로 직렬화 |
| HTML 직렬화 | `<span class="tag-pill" data-tag="<name>">#<name></span>` |
| InputRule | `(?:^|[^\w#])(#[\p{L}\p{N}_/-]{1,64})\s$/u` — 공백/구두점으로 끝났을 때 매칭 |
| PasteRule | 동일 패턴 (global) |

### 3.2 mark 캡처 그룹 규약 (BUG-FG22-02 회귀, 2026-04-25)

`markInputRule` 의 last capture group 의 텍스트만 mark 대상으로 보존. 따라서 group 1 = `#word` 전체 (`#` 접두사 보존).

### 3.3 실측 inclusive 동작

`inclusive: false` — `#ai world` 입력 시 `ai` 만 mark, ` world` 는 일반 text. mark 끝에 입력한 텍스트가 자동 흡수되지 않음.

---

## 4. WikiLinkMark — Inline Mark (Phase 2 FG 2-3, 2026-05-10 종결)

**파일**: `frontend/src/features/editor/tiptap/extensions/WikiLinkMark.ts`
**종류**: TipTap `Mark`
**관련 backend**: `app/services/wikilink_rules.py` (서버 파서 정본).

### 4.1 schema

| 속성 | 값 |
|------|----|
| `name` | `"wikilink"` |
| `inclusive` | `false` |
| `excludes` | (미지정 → default 빈 문자열) |
| attribute `target` | string. `data-wikilink-target` 으로 직렬화. NFC 정규화는 backend resolver 단계 |
| HTML 직렬화 | `<span class="wikilink" data-wikilink-target="<title>">[[<title>]]</span>` |
| InputRule | `/(?:^|[^\[])(\[\[(?:[^\]\n|]{1,200})(?:\|[^\]\n]{1,200})?\]\])$/u` |
| PasteRule | 동일 패턴 (global) |

### 4.2 alias 처리

`[[제목|표시이름]]` — alias 부분은 mark attribute 에 저장 안 함. resolver 가 무시하고 group 1 (target) 만 사용.

### 4.3 status modifier

`resolved` / `ambiguous` / `missing` 시각 차이는 mark attribute 가 아닌 **CSS class modifier** 로 표현 (mount 후 fetch 결과로 추가). 이유: status 는 시간 따라 변하므로 본문 영속화 시 stale 위험.

modifier 클래스: `wikilink--resolved` / `wikilink--ambiguous` / `wikilink--missing` (`globals.css` 정의).

---

## 5. AnnotationMark — 미구현 (task5-2 에서 신설 예정)

task5-2 §2.1.1 의 schema spec 을 baseline 으로 본 ADR 에 명시.

| 속성 | 결정 (잠정 — task5-1 ADR 에서 확정) |
|------|----|
| `name` | `"annotation"` |
| `inclusive` | `false` (HashtagMark / WikiLinkMark 와 동일 — mark 끝 입력 흡수 차단) |
| attribute | `annotation_id: string` (uuid — `annotations.id`) |
| HTML 직렬화 | `<span class="annotation-mark" data-annotation-id="<uuid>"></span>` |
| InputRule / PasteRule | 없음 — UI 패널에서 명시 적용 |
| 클릭 핸들러 | `attachAnnotationClickHandler` (계획) — TipTap default selection / cursor 동작 보존 (R-A3) |

---

## 6. ProseMirror schema 우선순위 — TipTap default

TipTap 은 mark 등록 순서대로 ProseMirror schema spec 의 `marks` order 를 결정. 현재 등록 순서 (`DocumentTipTapEditor.tsx:60-77`):

```
1. StarterKit (default marks: bold / italic / strike / code / link)
2. Placeholder
3. NodeId (Extension — mark 아님)
4. HashtagMark
5. WikiLinkMark
(6. AnnotationMark — task5-2 신설 시 추가)
```

ProseMirror schema 의 `marks` order 가 직렬화 시 mark 배열 순서를 결정. 본 ADR §c 결정에 따라:
- (i) TipTap default 순서 그대로 — 추가 정규화 없음
- (ii) 저장 직전 명시적 mark order 정렬 — 결정성 보강

→ ADR 결정 후 적용.

---

## 7. cross-stack 정합 (frontend ↔ backend)

| 항목 | frontend | backend | 정합 여부 |
|------|---------|---------|---------|
| HashtagMark 정규식 | `INPUT_REGEX = /(?:^|[^\w#])(#[\p{L}\p{N}_/-]{1,64})\s$/u` | `INLINE_HASHTAG_PATTERN = re.compile(r"...")` (`tag_rules.py`) | ✅ 동일 정신 (frontend 는 InputRule, backend 는 finditer) |
| WikiLinkMark 정규식 | `INPUT_REGEX = /...\[\[(?:[^\]\n|]{1,200})(?:\|[^\]\n]{1,200})?\]\].../` | `WIKILINK_PATTERN = re.compile(r"\[\[([^\]\n|]{1,200})(?:\|([^\]\n]{1,200}))?\]\]")` (`wikilink_rules.py`) | ✅ 동일 |
| HashtagMark data-attr | `data-tag` | `tag_rules` 가 직접 HTML 파싱 안 함 (text run 만 봄) | ✅ |
| WikiLinkMark data-attr | `data-wikilink-target` | `wikilink_rules` 가 직접 HTML 파싱 안 함 | ✅ |
| NodeId | `data-node-id` (frontend) / `attrs.node_id` (JSON) | `nodes_from_prosemirror` 가 attrs.node_id 사용 | ✅ |
| markdown→ProseMirror 변환 (FG 2-6) | (해당 없음 — server-side 변환) | `markdown_to_prosemirror.py` 가 wikilink mark `attrs.target`, hashtag mark `attrs.tag` 부여 | ✅ |

→ cross-stack 정합 양호. 본 ADR 의 §d (markNames 단일 정본) 가 도입되면 string literal 인라인 위험도 해소.

---

## 8. 실측 결론

본 보고서 시점 (2026-05-11) Mimir 의 ProseMirror schema 는:
- **Mark 종류**: 4 (StarterKit default 제외) — `nodeId` (global attr), `hashtag`, `wikilink` 구현 완료. `annotation` 신설 예정.
- **WikiLinkMark 처리 결정 (task5-1 ADR §a 표)**: **(a) 본 Phase 흡수 완료** — Phase 2 FG 2-3 재진입 종결로 정합.
- **AnnotationMark schema spec**: 본 ADR 에서 확정, 실제 코드는 task5-2 (FG 5-2) 에서.

---

## 9. ADR 작성 시 입력 결정 항목

본 보고서가 ADR 작성에 필요한 baseline 을 제공한다. ADR 에서 결정해야 할 사항 (task5-1 §2.1.1):

1. **§a 4 mark 확정** — 4 mark 모두 본 ADR 가 규율 (WikiLinkMark 흡수 확인됨).
2. **§b 우선순위/결합 규칙** — HashtagMark + AnnotationMark 같은 범위 적용 시 동작.
3. **§c 직렬화 정본** — (i) TipTap default vs (ii) 명시적 mark order 정규화.
4. **§d markNames 단일 정본 모듈** — `frontend/src/features/editor/tiptap/markNames.ts` 신설.
5. **§e inclusive 정책** — 4 mark 모두 `inclusive: false` 권장.
6. **§f backend 파서 정합** — 본 §7 cross-stack 표가 baseline.

---

## 10. 참조

- `docs/개발문서/S3/phase5/작업지시서/task5-1.md` §2.1.0 (본 보고서 정의)
- `docs/개발문서/S3/phase2/산출물/FG2-3_종결보고서.md` (WikiLinkMark 종결)
- `frontend/src/features/editor/tiptap/extensions/NodeId.ts`
- `frontend/src/features/editor/tiptap/extensions/HashtagMark.ts`
- `frontend/src/features/editor/tiptap/extensions/WikiLinkMark.ts`
- `frontend/src/features/editor/tiptap/DocumentTipTapEditor.tsx`
- `backend/app/services/tag_rules.py`, `backend/app/services/wikilink_rules.py`

---

*작성: 2026-05-11 | FG 5-1 Mark inventory 실측 — ADR 작성 baseline*
