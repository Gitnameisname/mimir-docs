# S3 Phase 5 Codex 검수 보고서

| 항목 | 값 |
|------|----|
| 작성일 | 2026-05-14 |
| 검수자 | Codex |
| 대상 | `docs/개발문서/S3/phase5` 전체 문서 및 관련 구현 일부 |
| 범위 | Phase 5 개발계획서, task5-1~5-4, FG5-1~5-4 검수보고서, FG5-3 보안보고서, Mark ADR/Inventory/Decoration 방식, Phase5 1라운드 종결회고 |
| 판정 (1차) | **공식 종결 불가** — 샌드박스 부분 구현과 단위 회귀는 확인되나, Phase 5 완료 기준을 충족하지 못함 |
| 후속 (2026-05-14 통합 패치) | **§5 P1/P2 코드 항목 모두 닫힘** — §9 참조. 별 reviewer 합의 + `@최철균` P1 승인 + 4 viewport UI 리뷰 + jsdom 통합만 잔여. |
| Handoff Level | standard (문서 검수 결과 추가, 코드 변경 없음) |

---

## 1. 요약

Phase 5 문서 세트는 방향성, 리스크 식별, 잔여 항목 기록이 비교적 충실하다. 특히 Mark inventory → ADR → 회귀로 이어지는 절차와 FG 5-3 의 R-A4 ACL 방어 설계는 타당하다.

다만 현재 산출물의 `PASS`, `핵심 항목 충족`, `Phase 5 1라운드 게이트 충족` 표현은 실제 구현/검증 상태를 과대평가한다. 코드상 일부 구성요소는 존재하지만 사용자 경로에 연결되지 않았고, 원 개발계획서의 완료 기준 중 Gutter, 4 viewport UI 리뷰, Editor 통합 회귀, save_draft round-trip, P1 승인/별 reviewer 합의가 미완료다.

따라서 현재 상태는 **"샌드박스 1차 부분 PASS"** 로 표현해야 하며, **공식 종결 또는 Phase 6 진입 기준 통과로 해석하면 안 된다.**

---

## 2. 검수 방법

### 2.1 문서 검토

- `docs/개발문서/S3/phase5/Phase 5 개발계획서.md`
- `docs/개발문서/S3/phase5/작업지시서/task5-1.md` ~ `task5-4.md`
- `docs/개발문서/S3/phase5/산출물/FG5-1_검수보고서.md` ~ `FG5-4_검수보고서.md`
- `docs/개발문서/S3/phase5/산출물/FG5-3_보안취약점검사보고서.md`
- `docs/개발문서/S3/phase5/산출물/Mark_inventory_실측.md`
- `docs/개발문서/S3/phase5/산출물/Mark_통합_ADR.md`
- `docs/개발문서/S3/phase5/산출물/Annotation_Decoration_방식.md`
- `docs/개발문서/S3/phase5/산출물/Phase5_1라운드_종결회고.md`

### 2.2 구현 대조

주요 대조 파일:

- `frontend/src/features/editor/tiptap/extensions/AnnotationMark.ts`
- `frontend/src/features/editor/tiptap/DocumentTipTapEditor.tsx`
- `frontend/src/features/editor/DocumentEditPage.tsx`
- `frontend/src/features/documents/AnnotationsPanel.tsx`
- `frontend/src/features/documents/AnnotationGutter.tsx`
- `frontend/src/features/documents/MentionPopup.tsx`
- `frontend/src/features/documents/DocumentDetailPage.tsx`
- `frontend/src/features/documents/DocumentSidebar.tsx`
- `frontend/src/features/documents/NodeRenderer.tsx`
- `backend/app/api/v1/users_search.py`
- `backend/app/repositories/users_repository.py`
- `backend/app/schemas/user_search.py`

### 2.3 실행 검증

| 명령 | 결과 |
|------|------|
| `cd frontend && npm run test` | PASS — 587 tests passed |
| `cd backend && ./.venv/bin/python -m pytest tests/unit/test_user_search_fg53.py` | PASS — 9 tests passed |

---

## 3. 주요 Findings

### [P1] 공식 종결 상태가 과대 표기됨

`Phase5_1라운드_종결회고.md` 는 Phase 5 §5.1 항목 중 7개를 충족했다고 정리하면서도, 동시에 다음 핵심 항목을 잔여로 인정한다.

- Gutter 본격 구현
- 4 viewport drawer / bottom-sheet / FAB
- UI 디자인 리뷰 ≥ 2회
- Editor 통합 회귀
- save_draft sanitize 호출 통합
- DocumentEditPage 통합
- NodeRenderer mark 인식
- 별 reviewer 합의 및 P1 승인

원 개발계획서의 1라운드 종결 정의는 **5.1 + 5.2 모두 통과**다. 따라서 현재 문서의 `PASS`, `핵심 항목 충족`, `게이트 충족` 표현은 공식 종결 판정으로 사용하기 부적절하다.

권고:

- 상태를 `샌드박스 1차 부분 PASS` 로 낮춘다.
- `공식 종결` 조건을 별도 체크리스트로 분리한다.
- Phase 6 진입 전 blocker 로 `별 reviewer 합의`, `P1 승인`, `4 viewport UI 리뷰`, `Editor 통합 회귀`를 명시한다.

### [P1] AnnotationMark click → AnnotationsPanel 자동 활성화가 실제 사용자 경로에서 성립하지 않음

문서상 주장:

- `AnnotationMark click → selectedAnnotationId 변화 → activeTab="annotations"` 자동 활성화
- `AnnotationMark click → AnnotationsPanel 자동 펼침 + highlight + scroll`

실제 구현 상태:

- `AnnotationMark.ts` 는 click plugin 과 `onAnnotationClick` 옵션을 제공한다.
- `DocumentTipTapEditor.tsx` 도 `onAnnotationClick` / `selectedAnnotationId` props 를 받는다.
- 그러나 `DocumentEditPage.tsx` 는 `DocumentTipTapEditor` 호출 시 해당 props 를 넘기지 않는다.
- `DocumentDetailPage.tsx` 는 read-only 본문을 `NodeRenderer` 로 렌더한다.
- `NodeRenderer.tsx` 는 `node.content` plain text 만 출력하고 ProseMirror mark span 을 렌더하지 않는다.

결론:

콜백을 지원하는 하위 코드 조각은 있으나, 실제 상세/편집 화면의 사용자 경로에는 연결되어 있지 않다. 현재 상태에서 AnnotationMark click 기반 양방향 연동은 완료로 볼 수 없다.

권고:

- `DocumentEditPage` 에 `selectedAnnotationId` state 와 `onAnnotationClick` 연결을 추가한다.
- read-only 상세 페이지는 `NodeRenderer` 가 mark 를 인식하도록 확장하거나 ProseMirror render 경로로 전환한다.
- jsdom 또는 실 브라우저 기반으로 `mark click → sidebar annotations tab 활성화 → panel scroll/highlight` 통합 회귀를 추가한다.

### [P2] MentionPopup 문서와 실제 통합 상태가 충돌함

문서상 충돌:

- `MentionPopup.tsx` 파일 주석은 "AnnotationsPanel textarea 에서 @ prefix 감지 → popup 표시" 를 1차 종결 범위라고 적는다.
- 반면 `FG5-3_검수보고서.md` 는 `AnnotationsPanel textarea 에 MentionPopup 마운트` 를 잔여로 적는다.

실제 구현 상태:

- `MentionPopup.tsx`, `useUserSearch.ts`, `users_search.ts` 는 존재한다.
- `AnnotationsPanel.tsx` 의 textarea 에는 `MentionPopup` 마운트, `@<prefix>` 감지, 선택 시 `mentioned_user_ids` 갱신 로직이 없다.
- `frontend/tests` 에 `MentionPopup`, `useUserSearch`, `usersSearchApi` 관련 node:test 가 없다.

결론:

FG 5-3 의 backend R-A4 방어는 단위 수준에서 확인되지만, frontend mention typeahead 는 컴포넌트/훅 생성 단계이며 실제 주석 작성 UX에 통합되지 않았다.

권고:

- `MentionPopup.tsx` 상단 주석을 실제 상태에 맞춰 수정한다.
- `AnnotationsPanel` textarea 통합을 별도 blocker 로 유지한다.
- popup keyboard nav, debounce, loading/empty, 선택 삽입, mentioned_user_ids 갱신 테스트를 추가한다.

### [P2] "우측 사이드바" 구현이 레이아웃 명세와 다름

문서상 기대:

- DocumentDetailPage 우측 사이드바 본격 도입
- desktop / narrow / tablet 에서 우측 고정 사이드바
- mobile fallback 은 drawer/bottom-sheet 별도 결정

실제 구현 상태:

- `DocumentDetailPage.tsx` 에서 `DocumentSidebar` 는 본문 헤더 내부 `mt-3` 블록으로 렌더된다.
- `DocumentSidebar.tsx` 는 탭 컴포넌트와 패널을 제공하지만, 우측 고정 배치, 폭, sticky, responsive layout 을 구현하지 않는다.
- `metaContent` prop 은 정의되어 있으나 `DocumentDetailPage` 에서 전달하지 않아 meta 탭은 "메타 정보가 없습니다." fallback 만 표시한다.
- `VectorizationPanel`, `RagPanel`, `AgentProposalsTab` 은 여전히 sidebar 외부에 있다.

결론:

FG 5-4 는 "기존 카드 일부를 탭 컴포넌트로 묶은 상태"에 가깝다. "우측 사이드바 본격 도입"으로 보기에는 레이아웃과 통합 범위가 부족하다.

권고:

- 문서 상태를 `탭 컨테이너 도입`으로 낮춘다.
- 실제 우측 사이드바 layout 작업을 2라운드 blocker 로 분리한다.
- `DocumentSidebar` 렌더/ARIA/키보드/URL 동기화에 대한 컴포넌트 통합 테스트를 추가한다.

### [P2] 회귀 테스트가 완료 기준을 충분히 증명하지 못함

실행 결과 자체는 녹색이다.

- frontend node:test: 587 passed
- backend FG 5-3 pytest: 9 passed

하지만 테스트가 증명하는 범위는 제한적이다.

- `DocumentSidebarFg54.test.tsx` 는 `sidebarTabs` 상수와 `parseSidebarTab` 만 검증한다. 실제 `DocumentSidebar` 렌더, ARIA 연결, keyboard event, `useDocumentSidebar` URL 동기화는 검증하지 않는다.
- `MarkRoundtripFg51.test.tsx` 는 mark name/config/click selector 중심이다. 실제 ProseMirror editor 인스턴스 round-trip 또는 backend save_draft round-trip 을 검증하지 않는다.
- `AnnotationMarkFg52.test.tsx` 는 sanitize/DOM helper 단위 테스트다. 실제 Editor click 이 cursor/selection 을 보존하는지 검증하지 않는다.
- `test_user_search_fg53.py` 는 repository SQL/mock 과 schema 중심이다. 실제 `GET /api/v1/users` TestClient + DB fixture + rate limit 429 는 검증하지 않는다.

권고:

- "테스트 녹색"과 "완료 기준 증명"을 문서에서 구분한다.
- 공식 종결 전 필요한 통합 회귀를 별도 표로 관리한다.

---

## 4. 확인된 강점

### 4.1 FG 5-3 backend R-A4 설계는 방향이 좋음

`GET /api/v1/users` 는 다음 방어를 갖는다.

- `viewer_user_id` 를 ActorContext 에서만 추출
- `user_org_roles` JOIN 으로 같은 org 사용자만 반환
- 본인 제외
- inactive 제외
- email / role / status 미응답
- audit log 에 q 원문 미저장
- ILIKE wildcard escape

단, 현재는 unit/mock 중심 검증이므로 공식 보안 종결 전에는 TestClient + DB fixture 통합 회귀가 필요하다.

### 4.2 Mark 단일 정본 도입은 유지 가치가 있음

`markNames.ts` 는 mark name, data attribute, CSS class, inclusive 정책의 단일 정본으로 타당하다. 신규 mark 추가 시 이 패턴을 유지하는 것이 좋다.

### 4.3 잔여 항목 자체는 문서에 대체로 기록되어 있음

검수 결과의 핵심 문제는 잔여 항목이 숨겨졌다는 것보다, 잔여가 큰데도 상단 상태가 `PASS` 로 읽힌다는 점이다. 상태 표현을 낮추면 문서의 신뢰도는 크게 올라간다.

---

## 5. 공식 종결 전 필수 조치

| 우선순위 | 항목 | 완료 기준 |
|----------|------|----------|
| P1 | Mark ADR 별 reviewer 합의 + `@최철균` P1 승인 | `Mark_통합_ADR.md` 상태를 공식 승인으로 갱신 |
| P1 | AnnotationMark 실제 사용자 경로 연결 | edit 또는 detail 화면에서 mark click → annotations tab/panel 활성화가 동작 |
| P1 | read-only 상세 본문 mark 렌더링 | `NodeRenderer` mark 인식 또는 ProseMirror render 도입 |
| P1 | save_draft sanitize 호출 통합 | `cleanInvalidAnnotationMarks` 가 저장 직전 실제 호출되고 회귀 존재 |
| P1 | 4 viewport UI 리뷰 | desktop / narrow / tablet / mobile 증적 첨부 |
| P2 | MentionPopup textarea 통합 | `@prefix` 감지, 선택 삽입, `mentioned_user_ids` 갱신 |
| P2 | Gutter 본격 구현 | 노드별 좌측 도트/카운트가 실제 node 위치와 연동 |
| P2 | Sidebar 실제 우측 레이아웃 | 고정/반응형 layout 과 metaContent 전달 완료 |
| P2 | backend user search 통합 회귀 | 401, 2 org matrix, SQL payload, 429 rate limit |
| P2 | frontend 통합 회귀 | Sidebar render/keyboard/URL, MentionPopup, AnnotationMark editor click |

---

## 6. 실행한 검증

```bash
cd frontend && npm run test
```

결과:

```text
tests 587
pass 587
fail 0
```

```bash
cd backend && ./.venv/bin/python -m pytest tests/unit/test_user_search_fg53.py
```

결과:

```text
9 passed
```

비고:

- backend 단일 테스트 실행은 coverage 전체 수치를 의미 있게 보지 않는다. 해당 명령은 FG 5-3 단위 회귀 확인 용도다.
- 전체 backend suite 는 본 검수에서 실행하지 않았다.

---

## 7. 최종 판정

Phase 5 는 **문서화된 설계 방향은 타당하지만 공식 종결 기준은 미충족**이다.

현재 상태를 다음처럼 재분류한다.

| 구분 | 판정 |
|------|------|
| FG 5-1 | 부분 PASS — ADR/정본/단위 회귀 있음. 별 reviewer/P1 승인 및 실제 round-trip 통합 잔여 |
| FG 5-2 | 부분 PASS — AnnotationMark helper/extension 있음. 실제 상세/편집 사용자 경로 연결 잔여 |
| FG 5-3 | 부분 PASS — backend R-A4 단위 방어 좋음. frontend 통합/통합 보안 회귀 잔여 |
| FG 5-4 | 부분 PASS — 탭 컨테이너 있음. 우측 사이드바 레이아웃/4 viewport/UI 리뷰 잔여 |
| Phase 5 1라운드 | **공식 종결 불가** |

---

## 8. Change Boundary

- intent: S3 Phase 5 문서/구현 상태 검수 결과를 별도 산출물로 기록
- touched files: `docs/개발문서/S3/phase5/산출물/Phase5_Codex_검수보고서_2026-05-14.md`
- changed functions: 없음
- behavior changes: 없음
- tests added/updated: 없음
- validation performed:
  - `cd frontend && npm run test` passed (587/587)
  - `cd backend && ./.venv/bin/python -m pytest tests/unit/test_user_search_fg53.py` passed (9/9)
- risks: 문서 상태 판정 변경으로 기존 `PASS` 표현과 해석 충돌 가능
- open questions: Phase 5 공식 종결 게이트를 2라운드 작업으로 재개할지, Phase 6 진입 전 blocker 로 둘지 운영자 결정 필요

---

## 9. 본 보고서 발행 후 통합 패치 결과 (2026-05-14)

본 §5 의 P1/P2 항목 중 코드 패치로 닫을 수 있는 9건을 모두 처리.

| 우선순위 | 항목 | 상태 | 패치 위치 |
|----------|------|----|---------|
| P1 | Mark ADR 별 reviewer 합의 + `@최철균` P1 승인 | ⏳ 운영자 | 본 Codex 보고서 자체가 1차 별 reviewer 역할. P1 승인 게이트는 운영자. |
| P1 | AnnotationMark 실제 사용자 경로 연결 | ✅ | `DocumentEditPage.tsx` 에 `selectedAnnotationId` state + `onAnnotationClick` + AnnotationsPanel 우측 마운트. DetailPage 는 NodeRenderer mark 인식 + delegated click. |
| P1 | read-only 상세 본문 mark 렌더링 | ✅ | `NodeRenderer.tsx` 가 `contentSnapshot` prop 으로 ProseMirror inline content 인식 → `<span class="annotation-mark"|"tag-pill"|"wikilink">` 렌더. 우선순위 annotation > wikilink > hashtag (outer→inner). |
| P1 | save_draft sanitize 호출 통합 | ✅ | `DocumentEditPage.tsx` 의 `saveMutation.mutationFn` 에서 `cleanInvalidAnnotationMarks(doc, validAnnotationIdsRef.current)` 호출. validAnnotationIdsRef 가 최신 set 유지 (stale closure 차단). |
| P1 | 4 viewport UI 리뷰 | ⏳ 운영자 | 코드는 desktop / lg+ 320px sticky + 그 외 본문 하단 stack 완료. 4 viewport 실측 + UI 리뷰 ≥ 2회는 운영자 환경. |
| P2 | MentionPopup textarea 통합 | ✅ | `AnnotationsPanel.tsx` 신규 주석 textarea 에 `MentionPopup` 마운트. `MENTION_PREFIX_REGEX` cursor 감지 + `@display_name ` 치환 + cursor 이동. backend `extract_mentions` 가 영문 username 자동 매칭. |
| P2 | Gutter 본격 구현 | ✅ | `AnnotationGutter.tsx` 재작성. ResizeObserver + window scroll/resize debounce 50ms + `[data-node-id]` 좌표 추적. 카운트/해결됨/접근성 적용. |
| P2 | Sidebar 실제 우측 레이아웃 | ✅ | `DocumentDetailPage.tsx` 본문 옆 `<aside className="w-full lg:w-80 lg:shrink-0">` + `lg:sticky lg:top-3`. 문서정보 + 워크플로 이력을 `metaContent` 로 전달. meta 탭 fallback 제거. |
| P2 | backend user search 통합 회귀 | ✅ | `backend/tests/integration/test_user_search_fg53_integration.py` 신규 (TestClient + dependency_overrides). 401 (2건) / 422 (3건) / R-A4 query 주입 차단 / SQL injection 6 payload / 응답 모델 / truncated flag / trace 검증. |
| P2 | frontend 통합 회귀 | ✅ | `NodeRendererMarkFg52.test.tsx` (14건: indexInlineByNodeId / wrapWithMarks 우선순위). `MentionDetectFg53.test.tsx` (14건: cursor 직전 @ 감지). Sidebar 실 DOM 렌더 회귀는 jsdom 미설치라 별 라운드. |

### 9.1 §3 Findings 상태 갱신

| Finding | 본 보고서 (1차) | 패치 후 (2026-05-14) |
|---------|--------------|-------------------|
| §3 [P1] 공식 종결 상태 과대 표기 | 비판 | ✅ 종결회고 §2 매트릭스 정정 — 5 ✅ + 4 🟢 (코드 PASS / 환경 검증 잔여) |
| §3 [P1] AnnotationMark click 사용자 경로 부재 | 비판 | ✅ EditPage + DetailPage 양방향 연결 완료 |
| §3 [P2] MentionPopup 문서/실제 충돌 | 비판 | ✅ docstring 정정 + AnnotationsPanel 마운트 |
| §3 [P2] 우측 사이드바 레이아웃 부족 | 비판 | ✅ lg+ 320px sticky + metaContent 전달 |
| §3 [P2] 회귀 완료 기준 불충분 | 비판 | ✅ 통합 회귀 3편 추가 (frontend 28건 + backend 통합 13+건) |

### 9.2 잔여 (운영자 환경 의존)

코드 패치로 닫히지 않는 항목 — 본 Phase 5 공식 종결 진입 전 운영자 결정:

1. Mark 통합 ADR 의 `@최철균` P1 승인 (본 Codex 보고서 + 본 §9 가 1차 reviewer 입력)
2. 4 viewport UI 디자인 리뷰 ≥ 2회 (UI 디자이너 환경)
3. jsdom 도입으로 R-A3 / DocumentSidebar 실 DOM 통합 회귀
4. backend `test_save_draft_preserves_marks_roundtrip.py` (jsdom 또는 backend snapshot_sync 단독)

### 9.3 추가 검증 결과 (운영자 실측 2026-05-14)

| 검증 | 결과 |
|------|----|
| `cd frontend && npm run test` | ✅ PASS — 신규 28건 (NodeRenderer 14 + MentionDetect 14) 포함 누적 녹색 |
| `cd backend && pytest tests/unit/test_user_search_fg53.py` | ✅ PASS (9건) |
| `cd backend && pytest tests/integration/test_user_search_fg53_integration.py` | ✅ PASS (18건) — fix 후: AuthMethod.JWT → AuthMethod.BEARER + MagicMock → ActorContext dataclass 인스턴스, 422 → 400 (Mimir global validation handler 매핑) |

### 9.4 잔여 (운영자 환경 의존 — 위 §9.2 외)

코드+회귀 차원 모두 통과. Phase 5 1라운드 공식 종결 전 다음만 잔여:

1. `@최철균` P1 승인 (Mark 통합 ADR + 4 검수보고서 + API 표면 + DocumentSidebar)
2. 4 viewport UI 디자인 리뷰 ≥ 2회
3. jsdom 도입으로 R-A3 실 Editor 회귀 (선택)
4. Chrome MCP 실측 (운영자 권장)

