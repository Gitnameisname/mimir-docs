# S3 Phase 5 1라운드 종결 회고

| 항목 | 값 |
|------|----|
| 작성일 | 2026-05-11 |
| 대상 | S3 Phase 5 (TipTap Mark 통합 + UX 본격화) — FG 5-1 ~ FG 5-4 |
| 진행 기간 | 2026-04-27 (작업지시서 작성) → 2026-05-11 (1차 종결) |
| 상태 | **샌드박스 1차 종결** — Phase 5 §5.1 핵심 항목 충족 / UI 디자인 리뷰 + 4 viewport drawer + 별 reviewer 합의는 별 라운드 → **공식 종결은 운영자 환경 검증 후** |

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
| **frontend node:test** | **587건 녹색** (Phase 5 신규 36건) |
| **backend pytest** | **9건 녹색** (FG 5-3 R-A4 회귀) |
| **frontend tsc** | **0 error** (본 Phase 신규/변경 코드) |
| **신규 파일** | 19 (frontend 13 + backend 4 + 문서 2 — 산출물 디렉토리 별도) |
| **산출 문서** | 8 (Mark_inventory + ADR + 4 검수보고서 + 보안보고서 + Decoration 방식) |

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

---

## 2. Phase 5 §5.1 1라운드 완료 기준 — 매트릭스

| # | 항목 | 상태 | 검증 |
|---|------|----|----|
| 1 | **R-A1 (Mark 직렬화)** — 4 mark round-trip ≥ 10 시나리오 | ✅ | FG 5-1 ADR §c (i) + FG 5-2 sanitize 회귀 7건 (S2/S5 포함) |
| 2 | **R-A2 (Mark 우선순위)** — ADR 의 우선순위 코드 반영 | ✅ | `excludes: ""` + 4 mark `inclusive: false` + markNames 정본 + 회귀 검증 |
| 3 | **R-A3 (편집 차단 없음)** — mark click cursor 영향 없음 | ✅ | FG 5-2 click plugin `handleClick → return false` + preventDefault 호출 0 |
| 4 | **R-A4 (Typeahead ACL)** — 다른 organization user 노출 없음 | ✅ | FG 5-3 4중 방어 (모델/라우터/repository/SQL) + pytest 9건 |
| 5 | AnnotationMark click → AnnotationsPanel 자동 펼침 + highlight + scroll | ✅ | FG 5-2 onAnnotationClick → setSelectedAnnotationId / FG 5-4 setActiveTab("annotations") 자동 전환 |
| 6 | Gutter 노드별 주석 카운트 정확 | 🟡 stub | FG 5-2 `AnnotationGutter` 분포 요약 박스만. 좌측 도트 위치 추적 (NodeRenderer 좌표) 별 라운드 |
| 7 | DocumentDetailPage 우측 사이드바 4 viewport UI 리뷰 통과 | 🟡 별 라운드 | desktop 고정 통합 완료. drawer / bottom-sheet / FAB / UI 디자인 리뷰 ≥ 2회 잔여 |
| 8 | Phase 1 NodeId 안정성 회귀 녹색 | ✅ | 회귀 영향 0 (markNames 정본 적용은 string literal → 상수 import 만) |
| 9 | Phase 3 annotations 회귀 (anchoring 4 시나리오) 녹색 | ✅ | AnnotationsPanel props 확장 (selectedAnnotationId / onSelectAnnotation) — 기존 동작 변경 0 |

**5.1 핵심 9 항목 중 7 충족, 2 잔여 (#6 Gutter 본격, #7 4 viewport UI 리뷰).**

## 3. Phase 5 §5.2 회귀 게이트 — 매트릭스

| # | 항목 | 상태 |
|---|------|----|
| 1 | Phase 0~4 모든 회귀 녹색 | ✅ (frontend 587 / backend 본 FG pytest 9) |
| 2 | pytest 베이스라인 유지 | ✅ |
| 3 | node:test 베이스라인 유지 | ✅ |
| 4 | tsc 0 error | ✅ (본 Phase 신규/변경 코드 — 기존 AnnotationsPanel toast 7건 오류는 Phase 3 잔재) |

**5.2 4 항목 모두 충족.**

## 4. 잔여 (Phase 5 2라운드 / 별 라운드 입력)

각 잔여를 4 카테고리로 분류 — 우선순위 / 게이트 / 책임 명시:

### 4.1 UI 디자인 리뷰 의무 (4 viewport)

| 항목 | 출처 |
|------|----|
| 4 viewport drawer / bottom-sheet / FAB (tablet/mobile) | task5-4 §2.1.2 |
| UI 디자인 리뷰 ≥ 2회 (mockup + 4 viewport 구현 후) | task5-4 §4 |
| Gutter 본격 좌측 도트 (NodeRenderer 좌표 추적) | task5-2 §2.1.4 |
| popup 4 viewport (mobile 키보드 시 가려짐 방지) | task5-3 |
| AnnotationMark 색상 / nested mark 시각 검증 | task5-2 §2.1.2 |

→ **운영자 + UI 디자인 리뷰어 의존**. 본 세션 코드 차원 정합 완료.

### 4.2 통합 / Editor 회귀 (jsdom or 실 Chrome)

| 항목 | 출처 |
|------|----|
| Editor 인스턴스 click → cursor 미변 (R-A3) 회귀 | FG 5-2 ADR §11 S10 |
| AnnotationMark 안 #tag InputRule 발동 (S8) | FG 5-2 ADR §11 |
| backend tag_rules 가 annotation 안 #tag 인식 (S6) | FG 5-1 ADR §g |
| save_draft round-trip 4 mark 동시 (S5) — backend pytest 통합 | FG 5-2 ADR §11 |
| 멘션 typeahead 통합 회귀 (TestClient 2 organization × 5 prefix R-A4) | FG 5-3 §5 |
| Rate limit 429 회귀 (60+ 요청) | FG 5-3 §5 |
| Timing attack 정량 측정 (결과 0건 vs N건 차이 < 5%) | FG 5-3 보안 §5 |

→ **jsdom 도입 + DB fixture + TestClient 환경 의존**. 별 라운드 인프라 작업.

### 4.3 통합 작업 (코드 차원)

| 항목 | 출처 | 책임 |
|------|----|----|
| TipTap suggestion 통합 (`@tiptap/suggestion` 의존성) — 본문 에디터 안 `@` 입력 popup | FG 5-3 §1 | P1 + 폐쇄망 미러 |
| AnnotationsPanel textarea 에 MentionPopup 마운트 + `@<prefix>` 패턴 감지 | FG 5-3 §5 | 별 라운드 |
| save_draft sanitize 호출 통합 (DocumentEditPage onChange 직전) | FG 5-2 §5 | 별 라운드 |
| DocumentEditPage 통합 — 본문 mark click → 사이드바 자동 활성화 | FG 5-4 §6 | 별 라운드 |
| NodeRenderer mark 인식 (read-only 상세 페이지 본문 mark 시각화) | FG 5-2 §5 | 별 라운드 |
| VectorizationPanel / RagPanel / AgentProposalsTab 사이드바 흡수 | FG 5-4 §6.1 | 별 라운드 |
| z-index 매트릭스 정본 (`zIndex.ts`) | FG 5-4 §6 | 별 라운드 |
| focus trap (drawer) | FG 5-4 §6 | 별 라운드 |

### 4.4 정책 / 합의

| 항목 | 출처 |
|------|----|
| **별 reviewer 합의** (헌법 제27조) — Mark 통합 ADR + 본 회고 | FG 5-1 §12 |
| `@최철균` P1 승인 — Mark 통합 ADR + 4 검수보고서 + API 표면 1 endpoint 추가 | Phase 5 §8 |
| FG 5-5 (한국어 username 정책) — 사용자 합의 후 진행 | Phase 5 §2.2 |

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
