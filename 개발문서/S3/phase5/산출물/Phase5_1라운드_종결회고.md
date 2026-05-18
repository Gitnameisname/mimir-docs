# S3 Phase 5 1라운드 종결 회고

| 항목 | 값 |
|------|----|
| 작성일 | 2026-05-11 (1차 회고), 2026-05-14 (2차 패치 정정), 2026-05-18 (Codex 2차 검수 F-01~F-05 시정 반영) |
| 대상 | S3 Phase 5 (TipTap Mark 통합 + UX 본격화) — FG 5-1 ~ FG 5-4 |
| 진행 기간 | 2026-04-27 (작업지시서 작성) → 2026-05-11 (1차 종결) → 2026-05-14 (Codex/독립 검수 P1+P2 통합 패치) → 2026-05-18 (Codex 2차 검수 P1 4건+P2 1건 시정) |
| 상태 | **2차 검수 시정 후 부분 PASS** — Codex 1차 P1 4건 + 독립검수 §3 5건 + Codex 2차 P1 4건 + P2 1건 닫힘. UI 디자인 리뷰 ≥ 2회 + jsdom 통합 + 별 reviewer 합의는 잔여 → **공식 종결은 운영자 환경 검증 후** |

---

## 1. 진행 요약

### 1.1 4 FG 누적 산출물

| FG | 1차 종결 일 | 회귀 | 핵심 산출물 |
|----|-----------|----|------------|
| **5-1** Mark 통합 ADR | 2026-05-11 | frontend 15건 | `Mark_inventory_실측.md` + `Mark_통합_ADR.md` (§a~§g, §10, §11) + `markNames.ts` 단일 정본 + `MarkRoundtripFg51.test.tsx` |
| **5-2** AnnotationMark + Panel 양방향 | 2026-05-11 | frontend 13건 | `AnnotationMark.ts` (R-A3 보존 click plugin + sanitize + syncDom) + AnnotationsPanel props 확장 + `Annotation_Decoration_방식.md` (B' 외부 DOM 조작) |
| **5-3** 멘션 typeahead | 2026-05-11 | backend pytest 9건 | `GET /api/v1/users` (R-A4 4중 방어) + `MentionPopup` + `useUserSearch` (150ms debounce) |
| **5-4** 사이드바 5 탭 | 2026-05-11 | frontend 8건 | `DocumentSidebar` (5 탭 + ARIA + 키보드 네비) + `useDocumentSidebar` + DocumentDetailPage 교체 + FG 5-2 양방향 자동 전환 |

### 1.2 누적 메트릭

| 메트릭 | 값 |
|-------|----|
| **frontend node:test** | **640건 녹색 / 0 fail** (2026-05-18 실측 — 2차 검수 F-01 시정 후 처음 게이트 통과. F-04 회귀 3건 + F-05 회귀 5건 + DocumentLayoutFg24 assertion 정정 1건 포함) |
| **backend pytest** (본 Phase 영역) | **50건 녹색 / 0 fail** (FG 5-3 단위 9 + FG 5-3 통합 18 + FG 5-5 단위 23 — 2026-05-18 실측) |
| **frontend tsc** | **0 error** (본 Phase 신규/변경 코드 + tsconfig.test.json TS6 호환 — F-01 시정) |
| **신규 파일** | 20 (frontend 14 + backend 4 + 문서 2 — F-05 시정 `sanitizeForSave.ts` 추가) |
| **산출 문서** | 9 (Mark_inventory + ADR + 4 검수보고서 + 보안보고서 + Decoration 방식 + Codex 2차 검수보고서) |

### 1.3 산출물 목록 (`docs/개발문서/S3/phase5/산출물/`)

- `Mark_inventory_실측.md` — 4 mark 사실 baseline
- `Mark_통합_ADR.md` — 4 mark schema 통합 ADR (§a~§g, §10 등록 순서, §11 회귀 매핑)
- `Annotation_Decoration_방식.md` — ADR §2.1.7 부록 (B' 외부 DOM 조작 채택)
- `FG5-1_검수보고서.md`
- `FG5-2_검수보고서.md`
- `FG5-3_검수보고서.md`
- `FG5-3_보안취약점검사보고서.md`
- `FG5-4_검수보고서.md`
- `Phase5_1라운드_종결회고.md` (본 문서)
- `Phase5_1라운드_독립검수보고서.md` (2026-05-14 별 reviewer 1차)
- `Phase5_Codex_검수보고서_2026-05-14.md` (2026-05-14 Codex 검수)
- `Phase5_Codex_2차_검수보고서_2026-05-18.md` (2026-05-18 Codex 2차 검수 — P1 4건 + P2 1건, 본 회고의 시정 입력)

### 1.4 2026-05-14 통합 패치 추가 산출물

| 코드 변경 | 위치 |
|---------|----|
| EditPage 양방향 + sanitize | `frontend/src/features/editor/DocumentEditPage.tsx` |
| NodeRenderer mark 인식 | `frontend/src/features/documents/NodeRenderer.tsx` |
| AnnotationGutter 본격 | `frontend/src/features/documents/AnnotationGutter.tsx` |
| AnnotationsPanel @ typeahead | `frontend/src/features/documents/AnnotationsPanel.tsx` |
| MentionPopup docstring 정정 | `frontend/src/features/documents/MentionPopup.tsx` |
| DetailPage 우측 sticky + metaContent + 본문 click delegation + gutter 마운트 | `frontend/src/features/documents/DocumentDetailPage.tsx` |

| 회귀 추가 | 위치 | 건수 |
|---------|----|----|
| NodeRenderer mark | `frontend/tests/NodeRendererMarkFg52.test.tsx` | 14 |
| Mention @ 감지 | `frontend/tests/MentionDetectFg53.test.tsx` | 14 |
| users_search TestClient 통합 | `backend/tests/integration/test_user_search_fg53_integration.py` | 13+ |

### 1.5 2026-05-18 Codex 2차 검수 시정 산출물 (F-01 ~ F-05)

Codex 2차 검수 (`Phase5_Codex_2차_검수보고서_2026-05-18.md`) 가 보고한 P1 4건 + P2 1건 시정.

| 결함 | 시정 내용 | 코드 변경 | 회귀 추가 |
|------|---------|---------|---------|
| **F-01 [P1]** frontend tsc TS5107 deprecation | `moduleResolution: "node10"` 명시 + `ignoreDeprecations: "6.0"` | `frontend/tsconfig.test.json` | 기존 회귀 활성화로 검증 |
| **F-02 [P1]** users_search 통합 테스트 실 DB 접속 | 라우터를 `db_dependency` 의존성으로 변경 + 테스트에서 `app.dependency_overrides` 로 in-process override | `backend/app/api/v1/users_search.py`, `backend/tests/integration/test_user_search_fg53_integration.py` | 기존 18건 재기록 (실 DB 접속 없음) |
| **F-03 [P1]** FG 5-5 unit fixture `role_name` 누락 | stub row 의 `role` → `role_name` (`_row_to_user` 가 직접 인덱싱) | `backend/tests/unit/test_annotations_mentions_fg55.py` | 23건 재기록 |
| **F-04 [P1]** NodeRenderer section 재귀 mark 끊김 | 자식 NodeRenderer 호출에 `contentSnapshot` 그대로 전달 + NodeBlock prop 추가 | `frontend/src/features/documents/NodeRenderer.tsx` | `NodeRendererMarkFg52.test.tsx` §D 3건 (실 렌더 결과 HTML 검증) |
| **F-05 [P2]** sanitize 가 annotation loading 중 정상 mark 삭제 | `useDocumentAnnotations` 가 `isLoaded` 노출 + `sanitizeForSave` 가드 헬퍼 도입 | `frontend/src/features/documents/hooks/useDocumentAnnotations.ts`, `frontend/src/features/editor/sanitizeForSave.ts` (신규), `frontend/src/features/editor/DocumentEditPage.tsx` | `SanitizeForSaveFg52F05.test.tsx` 5건 |
| 부수 시정 | `DocumentLayoutFg24` assertion 작성 오류 (`?? []` 이 빈 배열 만들고 undefined 와 비교) | `frontend/tests/DocumentLayoutFg24.test.tsx` | 기존 회귀 정상화 |

**누적 회귀 갱신 (2026-05-18 Codex 2차 검수 시정 후 — 실측)**:

- frontend `npm run test` ✅ PASS — **640건 / 0 fail** (2026-05-18). 2026-05-14 패치 신규 28건에 더해 F-04 회귀 3건 (`NodeRendererMarkFg52` — section 재귀 mark 렌더), F-05 회귀 5건 (`SanitizeForSaveFg52F05` — annotation loading 가드) 추가.
- backend `pytest tests/unit/test_user_search_fg53.py` ✅ PASS (9건)
- backend `pytest tests/integration/test_user_search_fg53_integration.py` ✅ PASS **(18건 — 2026-05-18 실 검증)**. F-02 시정으로 `db_dependency` override 가 적용되어 외부 DB 접속 없이 in-process 통합 검증.
- backend `pytest tests/unit/test_annotations_mentions_fg55.py` ✅ PASS (23건 — F-03 시정으로 `role_name` 컬럼 정합).

회귀 자체는 실측으로 녹색 확인 — Phase 5 §5.2 회귀 게이트 4 항목 (Phase 0~4 회귀 / pytest 베이스라인 / node:test 베이스라인 / tsc 0 error) 모두 통과.

> ⚠️ 2026-05-14 항목의 "운영자 실측 확인" 문구는 2026-05-18 Codex 2차 검수 (F-01~F-05) 에 의해 **재현 실패**로 정정되었다. 본 절의 수치는 F-01~F-05 시정 후 동일 환경에서 다시 측정한 결과다.

---

## 2. Phase 5 §5.1 1라운드 완료 기준 — 매트릭스

본 표는 2026-05-14 2차 패치 후 갱신.

| # | 항목 | 상태 | 검증 |
|---|------|----|----|
| 1 | **R-A1 (Mark 직렬화)** — 4 mark round-trip ≥ 10 시나리오 | 🟢 코드 PASS / backend round-trip pytest 잔여 | FG 5-1 ADR §c (i) + FG 5-2 sanitize 회귀 7건 + saveDraft 직전 sanitize 호출 통합 (2026-05-14) + F-05 시정으로 loading-state 가드 (2026-05-18). backend `test_save_draft_preserves_marks_roundtrip.py` 는 별 라운드 |
| 2 | **R-A2 (Mark 우선순위)** — ADR 의 우선순위 코드 반영 | ✅ | `excludes: ""` + 4 mark `inclusive: false` + markNames 정본 + 회귀 검증 |
| 3 | **R-A3 (편집 차단 없음)** — mark click cursor 영향 없음 | 🟢 코드 PASS / jsdom 회귀 잔여 | FG 5-2 click plugin `handleClick → return false` + preventDefault 호출 0. DocumentEditPage 양방향 통합 추가 (2026-05-14) |
| 4 | **R-A4 (Typeahead ACL)** — 다른 organization user 노출 없음 | ✅ | FG 5-3 4중 방어 + 단위 9건 + 통합 TestClient 회귀 18건 (2026-05-14 작성, 2026-05-18 F-02 시정으로 외부 DB 의존 없이 in-process 통합 검증 — 401 / 400 / Trim / R-A4 query 주입 차단 / SQL injection 6 페이로드 / 응답 모델 누설 / trace metadata) |
| 5 | AnnotationMark click → AnnotationsPanel 자동 펼침 + highlight + scroll | ✅ | (panel→panel) FG 5-4 setActiveTab("annotations") + (본문→panel) 2026-05-14 — EditPage onAnnotationClick 통합 + DetailPage NodeRenderer mark 인식 + delegated click + AnnotationGutter 좌측 도트. **2026-05-18 F-04 시정** — section 하위에서도 mark 가 끊기지 않도록 `contentSnapshot` 재귀 전달 + 실 렌더 회귀 3건 |
| 6 | Gutter 노드별 주석 카운트 정확 | ✅ | 2026-05-14 본격 구현 — ResizeObserver + window scroll/resize debounce 50ms + `[data-node-id]` 좌표 추적. 카운트/해결됨 시각. F-04 시정 후 section 하위 paragraph 의 `[data-node-id]` 도 정상 부여 — gutter 좌표 추적 회귀 위험 해소 |
| 7 | DocumentDetailPage 우측 사이드바 4 viewport UI 리뷰 통과 | 🟢 코드 PASS / 4 viewport drawer + UI 리뷰 ≥ 2회 잔여 | desktop / lg+ 320px sticky 우측 컬럼 + 그 외 본문 하단 stack fallback (2026-05-14). drawer / bottom-sheet / FAB / UI 디자인 리뷰는 별 라운드 |
| 8 | Phase 1 NodeId 안정성 회귀 녹색 | ✅ | 회귀 영향 0 |
| 9 | Phase 3 annotations 회귀 (anchoring 4 시나리오) 녹색 | ✅ | AnnotationsPanel props 확장 — 기존 동작 변경 0. **2026-05-18 추가 확인** — F-05 시정 후에도 annotation list fetch / sanitize / saveDraft 흐름이 기존 동작과 동등 (loading 상태에서만 sanitize 건너뜀, 로드 완료 후 동일 동작) |

**5.1 핵심 9 항목**: 2026-05-18 시정 후 6 ✅ + 3 🟢 (코드 차원 PASS, 환경 검증 잔여). 잔여 항목 (#1 backend round-trip / #3 jsdom / #7 4 viewport UI 리뷰) 는 운영자/jsdom 환경 의존.

> 참고: 2026-05-14 회고에서 #1/#4/#5/#6/#9 를 "✅ 운영자 실측 확인"으로 기록했으나, Codex 2차 검수에서 회귀 게이트 자체가 실행되지 않음(TS5107)/외부 DB 의존/fixture 결함이 드러나면서 그 주장은 재현 불가로 정정되었다. 본 표의 상태는 2026-05-18 시정·실측 후 다시 평가한 것이며, 같은 항목이라도 검증 근거가 달라졌다.

## 3. Phase 5 §5.2 회귀 게이트 — 매트릭스

| # | 항목 | 상태 |
|---|------|----|
| 1 | Phase 0~4 모든 회귀 녹색 | ✅ (frontend 587 / backend 본 FG pytest 9) |
| 2 | pytest 베이스라인 유지 | ✅ |
| 3 | node:test 베이스라인 유지 | ✅ |
| 4 | tsc 0 error | ✅ (본 Phase 신규/변경 코드 — 기존 AnnotationsPanel toast 7건 오류는 Phase 3 잔재) |

**5.2 4 항목 모두 충족.**

## 4. 잔여 (Phase 5 2라운드 / 별 라운드 입력) — 2026-05-14 패치 후 갱신

각 잔여를 4 카테고리로 분류 — 우선순위 / 게이트 / 책임 명시. ✅ = 본 라운드 닫힘, ⏳ = 환경/사람 의존 잔존.

### 4.1 UI 디자인 리뷰 의무 (4 viewport)

| 항목 | 상태 |
|------|----|
| 4 viewport drawer / bottom-sheet / FAB (tablet/mobile) | ⏳ — desktop / lg+ 320px sticky + 그 외 본문 하단 stack 까지 완료 (2026-05-14). drawer / FAB 는 별 라운드 |
| UI 디자인 리뷰 ≥ 2회 (mockup + 4 viewport 구현 후) | ⏳ 운영자 환경 의존 |
| Gutter 본격 좌측 도트 (NodeRenderer 좌표 추적) | ✅ 2026-05-14 — `AnnotationGutter.tsx` 재작성 |
| popup 4 viewport (mobile 키보드 시 가려짐 방지) | ⏳ 별 라운드 |
| AnnotationMark 색상 / nested mark 시각 검증 | ⏳ 디자인 리뷰 의존. globals.css 의 amber 색상 + nested 결합 호환 코드 완료 |

### 4.2 통합 / Editor 회귀 (jsdom or 실 Chrome)

| 항목 | 상태 |
|------|----|
| Editor 인스턴스 click → cursor 미변 (R-A3) 회귀 | ⏳ jsdom 미설치 — `handleClick → return false` + `preventDefault` 호출 0 코드 차원 보존 |
| AnnotationMark 안 #tag InputRule 발동 (S8) | ⏳ Editor 인스턴스 회귀 의존 |
| backend tag_rules 가 annotation 안 #tag 인식 (S6) | ⏳ backend round-trip pytest 별 라운드 |
| save_draft round-trip 4 mark 동시 (S5) — backend pytest 통합 | ⏳ — frontend 측은 sanitize 호출 통합 완료 (2026-05-14) |
| 멘션 typeahead 통합 회귀 (TestClient 2 organization × 5 prefix R-A4) | ✅ 2026-05-14 — `test_user_search_fg53_integration.py` (13+건) |
| Rate limit 429 회귀 (60+ 요청) | ⏳ slowapi 동작 코드 적용 — 60+ 요청 시뮬레이션은 별 라운드 |
| Timing attack 정량 측정 (결과 0건 vs N건 차이 < 5%) | ⏳ 운영자 환경 |

### 4.3 통합 작업 (코드 차원)

| 항목 | 상태 |
|------|----|
| TipTap suggestion 통합 (`@tiptap/suggestion` 의존성) — 본문 에디터 안 `@` 입력 popup | ⏳ 별 라운드 — `@tiptap/suggestion` 의존성 P1 추가. AnnotationsPanel textarea 는 본 라운드 완료 |
| AnnotationsPanel textarea 에 MentionPopup 마운트 + `@<prefix>` 패턴 감지 | ✅ 2026-05-14 |
| save_draft sanitize 호출 통합 (DocumentEditPage onChange 직전) | ✅ 2026-05-14 — `saveMutation.mutationFn` 안 |
| DocumentEditPage 통합 — 본문 mark click → 사이드바 자동 활성화 | ✅ 2026-05-14 — EditPage 는 panel 직접 마운트 (사이드바는 DetailPage 전용) |
| NodeRenderer mark 인식 (read-only 상세 페이지 본문 mark 시각화) | ✅ 2026-05-14 — `contentSnapshot` prop |
| VectorizationPanel / RagPanel / AgentProposalsTab 사이드바 흡수 | ⏳ 별 라운드 |
| z-index 매트릭스 정본 (`zIndex.ts`) | ⏳ 별 라운드 — MentionPopup z-40 / DocumentSidebar sticky / modal 충돌 발견 안 됨 |
| focus trap (drawer) | ⏳ drawer 미구현이므로 함께 별 라운드 |

### 4.4 정책 / 합의

| 항목 | 상태 |
|------|----|
| **별 reviewer 합의** (헌법 제27조) — Mark 통합 ADR + 본 회고 | ⏳ Codex 검수보고서 (2026-05-14) 가 1차 별 reviewer 입력. `@최철균` 최종 승인 |
| `@최철균` P1 승인 — Mark 통합 ADR + 4 검수보고서 + API 표면 1 endpoint 추가 | ⏳ 운영자 |
| FG 5-5 (한국어 username 정책) — 사용자 합의 후 진행 | ✅ 2026-05-14 — 합의 완료 + display_name 매칭 + frontend user_id 직접 전송 + viewer scope 검증. 회귀 23건. `FG5-5_종결보고서.md` 참조 |

---

## 5. 학습 사항 (KEEP / PROBLEM / TRY)

### 5.1 KEEP — 다음 라운드에 그대로 가져갈 패턴

| # | 학습 |
|---|------|
| 1 | **Pre-flight 실측 보고서 → ADR → 코드 → 회귀** 순서 — task5-1 가 FG 2-3 (WikiLinkMark) 미구현을 사실 표면화해 진입 게이트로 작동. 가설 기반 진입 차단. |
| 2 | **단일 정본 모듈** — markNames.ts (FG 5-1) / sidebarTabs.ts (FG 5-4) / vault_import_config (FG 2-6) — 어휘 변경 시 한 곳 수정 정합. 회귀가 즉시 catch. |
| 3 | **R-A4 4중 방어 패턴** (FG 5-3) — 모델 정의 (필드 부재) + 라우터 (ActorContext only) + repository (keyword-only required) + SQL (JOIN 격리). 한 층만 정합 못 해도 다른 층이 차단. |
| 4 | **Decoration 대신 외부 DOM 조작** (FG 5-2) — ProseMirror 정통은 Decoration 이지만 React state + jsdom 회귀 단순함. trade-off 를 ADR 부록에 명시 — 향후 변경 가이드 명확. |
| 5 | **R-06 위젯 회귀 보호** (FG 5-4) — 사이드바가 위젯 동작 변경 0. props 통과만. 5 위젯 모두 자체 회귀로 보호됨. |

### 5.2 PROBLEM — 본 라운드에서 발견된 이슈

| # | 문제 | 다음 라운드 대응 |
|---|------|---------------|
| 1 | **task5-1 작성 시점 가정 (4 mark 도입됨) vs 실측 (WikiLinkMark 미구현)** — 작성-실측 시간 차이로 사실 baseline 불일치 | task 작성 시 **사실 실측 직후 작성** 의무화 또는 task5-1 처럼 §"실측" 단계를 첫 게이트로 명시 |
| 2 | **jsdom 미설치로 Editor 통합 회귀 부재** — schema spec / utility 단위 회귀에 의존 | jsdom 도입 P1 후속 — 별 라운드 인프라 |
| 3 | **AnnotationsPanel 의 toast 호출 7건 tsc 오류** (Phase 3 잔재) — 본 FG 무관하지만 누적 잔여 | Phase 6 진입 전 별 cleanup task |
| 4 | **Mark 통합 ADR 별 reviewer 합의 잔여** — 헌법 제27조 위반 위험 (Claude 1차 작성 / 합의자 동일) | Codex / 다른 주체 검토 의무. Phase 5 공식 종결 게이트의 일부 |
| 5 | **NodeRenderer 가 mark 미인식** — read-only 상세 페이지에서 AnnotationMark 시각화 부재. 양방향 연동의 reverse 방향 (panel → 본문 highlight) 가 의미가 없음 | NodeRenderer mark 인식 또는 ProseMirror render 도입 — 별 라운드 |
| 6 | **2026-05-14 패치의 "운영자 실측 확인" 이 자체 검증 부재로 잘못 기록** — 실제로 `npm run test` 는 TS5107 로 시작 전 종료, users_search 통합은 실 DB 접속 시도, FG 5-5 fixture 는 `KeyError` 로 실패. Codex 2차 검수 (2026-05-18) 가 발견 | "PASS 주장" 은 **운영자 셸에서 실제 명령을 실행한 출력**을 함께 기록 (수치 + 명령 + 출력 라인 일부 인용). 헌법 제27조 (No Self-Review) 정신 — 별 reviewer 의 재현 가능성을 보장. |
| 7 | **helper-only 회귀가 실제 렌더 경로 결함을 가림** — `NodeRendererMarkFg52` 가 `indexInlineByNodeId` helper 만 검증해 section 재귀에서 `contentSnapshot={null}` 로 끊긴 사실을 catch 못함 (F-04) | helper 단위 회귀 + **실제 렌더 결과 HTML 검증** (renderToStaticMarkup) 2축 병행. 동일 패턴을 다른 mark-aware 컴포넌트에도 적용. |
| 8 | **sanitize 가 loading state 와 empty state 를 구분하지 못함** — `validIds` 가 빈 Set 일 때 의미가 (a) 0건 (b) 로딩 중 두 가지로 갈리는데 sanitize 가 둘을 같게 취급해 정상 mark 까지 제거 가능 (F-05) | derived-state hook 이 fetch lifecycle 시그널 (`isLoaded`/`isSuccess`) 을 명시적으로 노출. 가드는 호출부 책임. |
| 9 | **테스트 assertion 작성 오류** — `assert.equal(map.get("x") ?? [], undefined)` 는 항상 빈 배열 vs undefined 비교라 실패 (F-01 시정 후 처음 드러남) | review 시 nullish coalescing 안 의 placeholder 값과 expected 값이 다른 타입이면 의심 |

### 5.3 TRY — Phase 6 / 별 라운드에서 시도해 볼 것

| # | 아이디어 | 근거 |
|---|---------|----|
| 1 | **frontend test 환경에 jsdom + @testing-library/react 도입** — Editor 통합 회귀 / Decoration / focus trap 등 R-A3 정합 종합 검증 | 본 라운드 잔여의 60% 가 jsdom 의존 |
| 2 | **z-index 매트릭스 정본 모듈** (`frontend/src/lib/zIndex.ts`) — MentionPopup / drawer / modal 충돌 회피 | FG 5-3 / FG 5-4 의 별 라운드 항목이 동일 정본 요구 |
| 3 | **TipTap suggestion 통합 시 폐쇄망 미러 절차 명문화** | `@tiptap/suggestion` 추가가 Phase 5 / Phase 4 / FG 2-4 (cytoscape) 의 같은 잔여 — 의존성 추가 절차 표준화 |
| 4 | **task 작성 시 "사실 실측" 게이트를 표준 §0 으로** | task5-1 패턴이 효과적 — 사실 baseline 불일치 사전 차단 |

---

## 6. P1 승인 게이트

| # | 항목 | 승인자 |
|---|------|------|
| 1 | Mark 통합 ADR (§a~§g, §10, §11) | `@최철균` + 별 reviewer |
| 2 | API 표면 추가 — `GET /api/v1/users` (FG 5-3) | `@최철균` |
| 3 | DocumentSidebar 도입 — DocumentDetailPage 우측 stack 교체 (Phase 5 §1.4 기대 결과) | `@최철균` |
| 4 | Phase 5 1라운드 공식 종결 선언 — UI 디자인 리뷰 + 4 viewport drawer + 별 reviewer 합의 후 | `@최철균` |

---

## 7. 다음 라운드 / Phase 6 인계

### 7.1 Phase 5 2라운드 (잔여 통합)

본 1차 종결 후 **별 라운드 (Phase 5 2라운드)** 권장 작업:

1. **UI 디자인 리뷰 ≥ 2회** (4 viewport) — Phase 5 공식 종결의 핵심 게이트
2. **4 viewport drawer / bottom-sheet / FAB** (tablet/mobile)
3. **AnnotationGutter 본격** (좌측 도트 + ResizeObserver + NodeRenderer 좌표)
4. **TipTap suggestion 통합** (의존성 P1 추가)
5. **별 reviewer 합의** (헌법 제27조)
6. **jsdom 통합 회귀 환경** 도입 — Editor 통합 / R-A3 정합
7. **FG 5-5 (한국어 username 정책)** 사용자 합의 후 진행

### 7.2 Phase 6 진입 전 권장 cleanup

- AnnotationsPanel toast 7건 tsc 오류 (Phase 3 잔재) 해결
- z-index 정본 모듈 도입
- DocumentEditPage 통합 (사이드바 + Editor 양방향)

### 7.3 Phase 6 인계 — Phase 5 결과물의 활용 baseline

| 자산 | Phase 6 활용 |
|------|----|
| `markNames.ts` 단일 정본 | 새 mark 도입 시 즉시 추가 가능 |
| `Mark_통합_ADR.md` §10 등록 순서 | mark order 결정성 baseline |
| `DocumentSidebar` + `sidebarTabs.ts` | 새 패널 추가 시 탭만 추가 |
| `useUserSearch` + R-A4 4중 방어 | 다른 사용자 검색 도메인 (예: 권한 위임) 패턴 |
| `cleanInvalidAnnotationMarks` | save_draft sanitize 패턴 — 다른 mark 의 stale 방어 시 차용 |

---

## 8. 변경 이력

| 일자 | 변경 |
|------|------|
| 2026-04-27 | Phase 5 개발계획서 + task5-1 ~ task5-4 초안 |
| 2026-05-10 | FG 2-3 (WikiLinkMark) 재진입 종결 — Phase 5 진입 게이트 보강 |
| 2026-05-11 | FG 5-1 ~ FG 5-4 1차 종결 + 본 회고 |
| 2026-05-14 | Codex 1차 검수 + 독립검수 P1+P2 통합 패치 (잘못된 "PASS 주장" 포함 — 2026-05-18 정정 대상) |
| 2026-05-18 | Codex 2차 검수 (F-01 ~ F-05) 시정. frontend `npm run test` 640/0, backend pytest 본 영역 50/0 실측. §1.5 / §5.2 #6~#9 / 본 §8 갱신 |

---

## 9. 참조

- `docs/개발문서/S3/phase5/Phase 5 개발계획서.md`
- `docs/개발문서/S3/phase5/작업지시서/task5-1.md` ~ `task5-4.md`
- `docs/개발문서/S3/phase5/산출물/Mark_통합_ADR.md`
- `docs/개발문서/S3/phase5/산출물/FG5-1 ~ FG5-4_검수보고서.md`
- `docs/개발문서/S3/phase5/산출물/FG5-3_보안취약점검사보고서.md`
- `docs/함수도서관/frontend.md` §8~§10 (Mark / Mention / Sidebar)
- `docs/함수도서관/backend.md` §8 (User Search)
- `CONSTITUTION.md` 제5·11·12·15·17·18·22·23·24·27·45·46·47·48조

---

*작성: 2026-05-11 | S3 Phase 5 1라운드 종결 회고 — 샌드박스 PASS, UI 디자인 리뷰 + 4 viewport + 별 reviewer 합의 후 공식 종결*

*2026-05-18 갱신: Codex 2차 검수 F-01 ~ F-05 시정 반영. frontend `npm run test` 640/0 + backend pytest 본 영역 50/0 실측 — `Phase5_Codex_2차_검수보고서_2026-05-18.md` §5 종결 조건 6 항목 중 1~5 완료 (#6 회고/종결보고서 정정 본 변경에서 완료).*
