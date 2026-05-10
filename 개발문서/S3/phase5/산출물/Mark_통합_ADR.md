# Mark 통합 ADR — S3 Phase 5 FG 5-1

| 항목 | 값 |
|------|----|
| 작성일 | 2026-05-11 |
| 대상 | Mimir frontend ProseMirror schema 의 4 mark 통합 운영 규약 |
| 상태 | 1차 합의 (Claude 작성). 별 reviewer 합의 + `@최철균` P1 승인 후 종결 |
| 입력 | `Mark_inventory_실측.md` (2026-05-11) |
| 적용 범위 | `nodeId` (Global Attribute), `hashtag` / `wikilink` / `annotation` (Inline Mark) — 4 종 |

---

## §a 적용 범위 mark — 결정

| 항목 | 종류 | 현재 상태 | 본 ADR 결정 |
|------|------|---------|-----------|
| NodeId | Global Attribute | 구현 완료 (FG 1-2) | **변경 없음** — 우선순위 계산 시 노드 속성으로 취급. mark 결합 규칙과 별 |
| HashtagMark | Inline Mark | 구현 완료 (FG 2-2) | schema 명시화 (§d markNames + §e inclusive 정책 적용) |
| WikiLinkMark | Inline Mark | 구현 완료 (FG 2-3 재진입 종결, 2026-05-10) | schema 명시화 동일 |
| AnnotationMark | Inline Mark | 미구현 — task5-2 (FG 5-2) 신설 | schema spec 본 ADR §d 에 미리 명시. 실제 코드는 FG 5-2 |

**WikiLinkMark 처리 (이전 task5-1 §2.1.1 §a 의 3 옵션)**: **(a) 본 Phase 흡수 완료** ← Phase 2 FG 2-3 재진입 종결 (2026-05-10) 로 정합.

---

## §b 우선순위 / 결합 규칙

ProseMirror schema 의 mark 는 단일 텍스트 범위에 **여러 mark 가 동시 적용** 가능하다 (`excludes` 가 비어있는 한). 본 ADR 은 다음 케이스에 대해 결정된 동작을 명시한다.

| 케이스 | 본문 | 기대 동작 | 회귀 ID |
|-------|------|---------|---------|
| **C1** | HashtagMark 안에 AnnotationMark | 두 mark 모두 적용. 직렬화 시 양쪽 mark 가 동일 text run 의 `marks` 배열에 등장 | S2 |
| **C2** | AnnotationMark 안에 HashtagMark | 두 mark 모두 적용 (동일) | S3 |
| **C3** | HashtagMark InputRule 이 AnnotationMark 영역에서 발동 | InputRule 동작 보존 + 두 mark 동시 적용 | S8 |
| **C4** | AnnotationMark 클릭 | TipTap default mark 동작 — cursor / selection 변경 없음. AnnotationsPanel 활성화는 별 이벤트 (R-A3) | S10 |
| **C5** | NodeId 변경 없이 mark 만 toggle | NodeId 안정 유지 (mark 토글은 노드 속성 미변경) | S4 |
| **C6** | save_draft round-trip | 4 mark 동시 적용된 doc → JSON → DB → 다시 doc | S5 |
| **C7** | HashtagMark `inclusive:false` 와 AnnotationMark `inclusive:false` 결합 | 주석 끝 / hashtag 끝 입력 시 mark 자동 흡수 안 함 (§e) | S7 |
| **C8** | WikiLinkMark + HashtagMark 동시 | 두 mark 모두 적용. wikilink target 안에 `#tag` 가 들어가면 wikilink mark + hashtag mark 둘 다 | (잠정) |

`excludes` 정책: **모든 inline mark 의 `excludes` 는 빈 문자열** — 다른 mark 와 결합 가능. C1~C3, C8 보장.

---

## §c 직렬화 정본 (Canonical Serialization)

ProseMirror JSON 의 mark 배열 순서는 TipTap 의 default order (extension 등록 순서 → schema spec) 를 따른다.

**결정**: **(i) TipTap default 순서 그대로 — 추가 정규화 없음**.

근거:
- 옵션 (ii) 명시적 mark order 정규화는 결정성 보강에 도움이 되지만, **마이그레이션 비용** 가능성 (기존 저장 doc 의 mark 순서가 정규화 형식과 다르면 round-trip 회귀가 깨짐).
- TipTap 자체가 schema 등록 순서대로 정렬하므로 **새 mark 추가 시 등록 순서를 ADR §10 에 명시** 하면 결정성 자체는 충분히 확보됨.
- save_draft round-trip 시 mark 배열을 set 단위로 비교 (순서 무시) 하는 회귀 테스트로 보강.

**부가 결정**: round-trip 회귀 비교 시 mark 배열을 **`{type, attrs}` set** 으로 비교 (순서 무시). 시퀀스 비교가 필요한 경우는 별 회귀로.

---

## §d 어휘 단일 정본 — `markNames.ts`

mark name / attribute key / data-attribute / CSS class 의 string literal 인라인 사용을 차단. 단일 정본 모듈 신설:

**파일**: `frontend/src/features/editor/tiptap/markNames.ts` (FG 5-1 Step 3 에서 신설)

```ts
// MARK NAMES (ProseMirror schema name)
export const HASHTAG_MARK_NAME = "hashtag" as const;
export const WIKILINK_MARK_NAME = "wikilink" as const;
export const ANNOTATION_MARK_NAME = "annotation" as const;

// DATA ATTRIBUTES (HTML 직렬화 키)
export const HASHTAG_DATA_ATTR = "data-tag" as const;
export const WIKILINK_DATA_ATTR = "data-wikilink-target" as const;
export const ANNOTATION_DATA_ATTR = "data-annotation-id" as const;
export const NODE_ID_DATA_ATTR = "data-node-id" as const;

// CSS CLASSES
export const HASHTAG_CLASS = "tag-pill" as const;
export const WIKILINK_CLASS = "wikilink" as const;
export const ANNOTATION_CLASS = "annotation-mark" as const;
```

**적용**: HashtagMark / WikiLinkMark 의 schema spec, parseHTML, renderHTML 이 본 모듈 import. AnnotationMark (task5-2 신설 시) 도 동일.

**cross-stack 정합**: backend `tag_rules.py` / `wikilink_rules.py` 가 직접 HTML 파싱 안 하므로 (text run 만 봄) data-attr 정합은 frontend 단에서만 강제. 단 frontend `markdown_to_prosemirror` (FG 2-6) 가 `attrs.tag` / `attrs.target` 을 부여하는 경로는 본 markNames 와 일관 (이미 정합).

---

## §e inclusive / excludes 정책

| mark | `inclusive` | `excludes` | 근거 |
|------|----|----|----|
| HashtagMark | `false` | (빈 문자열) | BUG-FG22-02 회귀 — mark 끝 입력 자동 흡수 차단 |
| WikiLinkMark | `false` | (빈 문자열) | 동일 정신 |
| AnnotationMark | `false` (결정) | (빈 문자열) | 주석 끝 입력 시 주석 영역 자동 확장 차단 |
| NodeId | (mark 아님) | — | — |

→ **4 mark 모두 `inclusive: false`**. `excludes` 는 빈 문자열로 통일 — 다른 mark 와 결합 가능 (§b C1~C3 보장).

---

## §f attribute 키 / data-attribute / CSS 네임스페이스 (단일 정본 표)

§d markNames 의 정본 표를 반영한 마스터 매핑:

| mark | attribute | HTML data attr | CSS class |
|------|-----------|----------------|----------|
| HashtagMark | `tag` (string, normalized lowercase) | `data-tag` | `tag-pill` |
| WikiLinkMark | `target` (string, NFC 정규화는 backend 단계) | `data-wikilink-target` | `wikilink` |
| AnnotationMark | `annotation_id` (uuid — annotations.id) | `data-annotation-id` | `annotation-mark` |
| NodeId (Global Attr) | `node_id` (uuid) | `data-node-id` | — |

**변경 시 절차**: 본 표를 변경하면 (a) `markNames.ts` (b) 각 mark schema (c) `globals.css` (d) backend 파서 (해당 시) (e) 본 ADR 표 5 곳 동시 수정. 한 곳만 빠지면 round-trip 회귀 깨짐.

---

## §g ProseMirror 변환 / Backend 영향

- `backend/app/services/snapshot_sync_service.py` 의 `rebuild_nodes_from_snapshot` 는 mark 정보를 무시하고 텍스트 + node_id 만 활용 (현행 유지).
- `backend/app/services/tag_rules.py::extract_tags_from_snapshot` 는 mark 안의 `#tag` 를 인식 (link mark text 내부 제외 패턴). 본 ADR 변경이 이 파서의 동작에 영향 없음.
- `backend/app/services/wikilink_rules.py::extract_wikilinks_from_snapshot` 는 mark 안의 `[[...]]` 를 인식 (codeBlock / link mark text 내부 제외).
- **회귀**: AnnotationMark 안의 `#tag` / `[[...]]` 가 backend 파서로 정상 인식되는지 회귀 1건 추가 (FG 5-2 단계에서 검증).

---

## §10 등록 순서 (직렬화 결정성의 baseline)

`DocumentTipTapEditor.tsx::useEditor` 의 extensions 배열 순서. 본 순서가 TipTap schema spec 의 mark order 결정.

```
1. StarterKit (default marks: bold / italic / strike / code / link)
2. Placeholder
3. NodeId (Extension — Global Attribute, mark 아님)
4. HashtagMark
5. WikiLinkMark
6. AnnotationMark  ← task5-2 (FG 5-2) 에서 추가
```

**규약**: 신규 mark 추가 시 본 §10 에 추가 + `Mark_inventory_실측.md` §6 갱신.

---

## §11 회귀 시나리오 매핑 (round-trip)

`frontend/src/features/editor/tiptap/__tests__/markRoundtrip.test.ts` (FG 5-1 Step 4 에서 신설) 에서 다음 시나리오 회귀.

| ID | 시나리오 | 매핑 |
|----|---------|------|
| S1 | NodeId + HashtagMark 단독 — text → JSON → text round-trip | C5 |
| S2 | HashtagMark 안에 AnnotationMark | C1 |
| S3 | AnnotationMark 안에 HashtagMark | C2 |
| S4 | mark 토글 후 NodeId 안정 (3 회 toggle 동안 node_id 불변) | C5 |
| S5 | save_draft round-trip — 4 mark 동시 적용된 doc | C6 |
| S6 | backend `tag_rules.py` 가 AnnotationMark 안의 `#tag` 도 인식 | §g |
| S7 | mark inclusive — 주석 / hashtag 끝에서 입력 시 mark 미확장 | §e / C7 |
| S8 | InputRule — 주석 mark 안에서 `#word ` 입력 시 HashtagMark 적용 | C3 |
| S9 | (생략 — 본 ADR 결정 (i) 채택으로 mark order 정규화 없음) | §c |
| S10 | mark click 동작 — AnnotationMark click 이 cursor / selection 정상 | C4 |
| S11 | WikiLinkMark + HashtagMark 동시 (wikilink target 안 hashtag 미적용 — wikilink mark text 는 inline 으로 hashtag InputRule 발동 안 됨이 자연스러움) | C8 |

> S6 / S5 일부는 backend round-trip 의존. **FG 5-2 (AnnotationMark 신설) 단계에서 실제 회귀 코드 작성**. FG 5-1 에서는 frontend-only 시나리오 (S1, S4, S7, S11) 위주로 작성하고 AnnotationMark 의존 시나리오 (S2, S3, S5, S6, S8, S10) 는 task5-2 의 회귀에 포함.

---

## §12 결정 합의 정합 (헌법 제27조)

본 ADR 은 Claude 1차 작성. 헌법 제27조 (Self-Review 금지) 정합:
- **별 reviewer 합의 필요** — Codex 또는 다른 주체. 본 세션에서는 작성자 = 합의자가 같으므로 **잠정 합의** 상태로 분류.
- **`@최철균` P1 승인** — Phase 5 §8 / AGENT_MODE.md §3.3 의 P1 게이트. ADR 변경은 본 게이트로.

본 ADR 의 결정이 후속 task5-2/5-3/5-4 에서 잠정으로 사용됨. P1 승인 후 정식 종결.

---

## §13 변경 이력

| 일자 | 변경 |
|------|------|
| 2026-04-27 | task5-1 작업지시서 작성 시점 — 4 mark 가정 (WikiLinkMark 미구현 사실 표면화) |
| 2026-05-10 | FG 2-3 재진입 종결 — WikiLinkMark 흡수 완료 |
| 2026-05-11 | 본 ADR 1차 작성 — Mark inventory 실측 + 결정 §a~§g + §10 등록 순서 + §11 회귀 매핑 |

---

## §14 참조

- `docs/개발문서/S3/phase5/작업지시서/task5-1.md`
- `docs/개발문서/S3/phase5/산출물/Mark_inventory_실측.md`
- `docs/개발문서/S3/phase2/산출물/FG2-3_종결보고서.md` (WikiLinkMark 종결)
- `frontend/src/features/editor/tiptap/extensions/{NodeId,HashtagMark,WikiLinkMark}.ts`
- `frontend/src/features/editor/tiptap/DocumentTipTapEditor.tsx`
- `backend/app/services/{tag_rules,wikilink_rules,snapshot_sync_service}.py`
- `CONSTITUTION.md` 제5·11·12·15·27·48조

---

*작성: 2026-05-11 | Mark 통합 ADR (S3 Phase 5 FG 5-1) — 1차 합의, 별 reviewer + P1 승인 후 종결*
