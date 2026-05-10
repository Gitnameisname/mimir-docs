# task5-4 — DocumentDetailPage 우측 사이드바 본격 도입 (FG 5-4)

**작성일**: 2026-05-10
**Phase / FG**: S3 Phase 5 / FG 5-4
**상태**: 착수 대기 (FG 5-2 의 AnnotationsPanel props 확장 + AnnotationGutter 산출 후 통합 권장. 단, 본 FG 의 사이드바 골격은 FG 5-2 와 병행 가능)
**담당**: 설계 Claude / 구현 Claude (또는 Codex — frontend 비중 큼) / 리뷰 Codex (extended — UI 재구조화 회귀 영향 큼)
**예상 소요**: 4~5일 (사이드바 골격 + 마운트 회귀 + 4 viewport UI 리뷰 ≥ 2회)
**선행 산출물**:
- FG 5-2 의 AnnotationsPanel props 확장 (`selectedAnnotationId` / `onSelectAnnotation`) — 본 FG 의 사이드바가 이 props 를 통과시키므로
- (선택) FG 5-3 MentionPopup — 본문 위에 floating popup 이 사이드바 z-index 와 충돌 가능성 사전 확인
**후행**: 본 FG 종결 시 Phase 5 1라운드 게이트 (Phase 5 §5.1 — 4 viewport UI 리뷰 통과 항목)

---

## 1. 작업 목적

현재 DocumentDetailPage 의 우측 영역은 collapsible 카드들이 단순 수직 stack 으로 배열되어 있다 (Pre-flight 실측 — `DocumentDetailPage.tsx:225–242`):

```
우측 stack (현재):
├─ TagChipsEditor (FG 2-2)
├─ ContributorsPanel (FG 3-1)
└─ AnnotationsPanel (FG 3-3)
```

추가로 본문 하단/외부에 분산:
- VectorizationPanel (header 우측 compact)
- RagPanel (별 영역)
- AgentProposalsTab (본문 하단)
- 메타데이터 / 워크플로 이력 섹션 (본문 하단)

본 FG 가 닫는 문제:
1. **카드들이 좁은 화면에서 본문을 가린다** (Phase 5 §7 R-04)
2. **AnnotationsPanel 이 자주 사용되는데 fixed 위치 부재로 스크롤 시 사라짐**
3. **panel 간 일관성 부재** — collapsible 동작 / 헤더 스타일 / 빈 상태 처리 등

본 FG 산출물:
1. **`DocumentSidebar`** 신설 — 우측 고정 panel + 탭 네비 (Annotations / Contributors / Tags / Vectorization / Agent Proposals / 기타 메타)
2. **4 viewport 동작** — desktop (>= 1280px) 고정 사이드바 / narrow (1024–1279px) 축소 사이드바 / tablet (768–1023px) collapsible / mobile (< 768px) bottom-sheet 또는 hidden + FAB
3. **모든 기존 위젯 회귀** — 기존 5 위젯 (TagChipsEditor / ContributorsPanel / AnnotationsPanel / VectorizationPanel / DocumentAssignControls) 동작 보존 (Phase 5 §7 R-06)

---

## 2. 작업 범위

### 2.1 포함

#### 2.1.0 Pre-flight — `산출물/FG5-4_Pre-flight.md`

##### a. 현재 위젯 inventory

`DocumentDetailPage.tsx` 전수 + 마운트 컴포넌트 inventory:

| 위젯 | 현 위치 | props 의존 | FG 매핑 |
|------|--------|---------|--------|
| TagChipsEditor | 우측 stack | `documentId`, `tags` | FG 2-2 |
| ContributorsPanel | 우측 stack | `documentId`, `currentUserId` | FG 3-1 |
| AnnotationsPanel | 우측 stack | `documentId`, `defaultNodeId`, (FG 5-2 추가) `selectedAnnotationId`, `onSelectAnnotation` | FG 3-3 + FG 5-2 |
| VectorizationPanel | header 우측 compact | `documentId` | (S2 또는 별 FG) |
| RagPanel | 별 영역 | (확인) | (별 FG) |
| AgentProposalsTab | 본문 하단 | `documentId` | (별 FG) |
| DocumentAssignControls | (확인) | (확인) | (별 FG) |
| 메타데이터 섹션 | 본문 하단 | (현 컴포넌트화 여부 확인) | — |
| 워크플로 이력 | 본문 하단 | — | (별 FG) |

Pre-flight 보고서에 위 표를 실측해 채운다. 누락 / 명세 불일치 발견 시 별 항목으로 표기.

##### b. viewport 임계 결정

Phase 5 §6.2 보류 결정 — collapsible 임계 viewport. 기본 후보:
- `>= 1280px` desktop — 고정 사이드바 320px
- `1024–1279px` narrow — 고정 사이드바 280px (탭 헤더 축약)
- `768–1023px` tablet — 우측 collapsible drawer (FAB 으로 토글)
- `< 768px` mobile — bottom-sheet 또는 본문 하단 stack fallback

본 FG 의 1차 UI 디자인 리뷰 (≥ 2회 의무 — Phase 5 §4 산출물 규약) 에서 임계 확정.

##### c. z-index / floating UI 충돌 분석

- FG 5-3 의 MentionPopup (floating UI)
- AnnotationsPanel 의 reply 입력 textarea
- toast / dialog 의 기존 z-index 매트릭스

z-index 단일 정본 (`frontend/src/lib/zIndex.ts` 또는 동등 — 기존 부재 시 신설) 결정. 사이드바, popup, drawer, modal 의 stacking 순서 명시.

#### 2.1.1 DocumentSidebar 컴포넌트 신설

`frontend/src/features/documents/DocumentSidebar.tsx` (신규):

##### a. 구조

```tsx
interface DocumentSidebarProps {
  documentId: string;
  currentUserId: string;
  // FG 5-2 양방향 연동
  selectedAnnotationId: string | null;
  onSelectAnnotation: (id: string | null) => void;
  // 활성 탭 — URL 또는 state 동기화
  activeTab?: SidebarTab;
  onTabChange?: (tab: SidebarTab) => void;
}

type SidebarTab = "annotations" | "contributors" | "tags" | "vectorization" | "agent" | "meta";
```

##### b. 탭 네비 + 탭 콘텐츠 마운트

- 상단 탭 바 — 6 탭 (또는 Pre-flight 결과로 조정)
- 각 탭은 기존 위젯 마운트:
  - `annotations` → `<AnnotationsPanel />`
  - `contributors` → `<ContributorsPanel />`
  - `tags` → `<TagChipsEditor />`
  - `vectorization` → `<VectorizationPanel />`
  - `agent` → `<AgentProposalsTab />` (Pre-flight — 본문 하단에서 이동 가능 여부 확인)
  - `meta` → 메타데이터 / 워크플로 이력 (Pre-flight 에서 컴포넌트화)
- 비활성 탭은 unmount 또는 hidden (성능 vs 상태 보존 트레이드오프 — Pre-flight 결정)

##### c. URL / state 동기화

- 활성 탭은 URL query (`?tab=annotations`) 또는 React Router state 와 동기화
- 새로고침 / 공유 링크에서 같은 탭 복원
- AnnotationMark click (FG 5-2) → 자동으로 `tab=annotations` + `selectedAnnotationId` 설정

##### d. 빈 상태 / 로딩 / 에러

- 각 탭은 자체 빈 상태 / 로딩 / 에러 처리 (기존 위젯 동작 그대로 — 사이드바는 컨테이너만)
- 사이드바 자체의 ACL 부재 panel (예: VectorizationPanel 이 admin-only 이면 viewer 에 탭 숨김) — 기존 동작 패턴 유지. 사이드바는 ACL 결정점이 아님 (R2)

##### e. 접근성

- 탭 — ARIA `role="tablist"` / `role="tab"` / `aria-selected`
- 키보드 네비 — 좌/우 화살표로 탭 이동, Enter/Space 활성화
- 화면 리더 — 활성 탭 변경 시 live region 알림
- focus trap — drawer / bottom-sheet 모드에서 사이드바 내부 focus 유지 (close 시 trigger 로 복귀)

#### 2.1.2 4 viewport 동작 (§2.1.0 b 결정 적용)

##### a. desktop / narrow — 고정 사이드바

- CSS grid / flex 로 본문 + 사이드바 동시 표시
- 사이드바 폭 결정값 적용

##### b. tablet — collapsible drawer

- 우측에서 슬라이드 인 drawer
- FAB (floating action button) 또는 헤더 토글 버튼
- backdrop + Esc 닫기

##### c. mobile — bottom-sheet 또는 hidden

- 하단에서 올라오는 sheet (예: `vaul` 또는 동등 라이브러리 — 기존 의존성 우선 확인)
- 또는 본문 하단 stack fallback (현 동작 유지)
- Pre-flight 의 디자인 리뷰에서 결정

##### d. 매체별 회귀 시나리오

- 기존 위젯 5 종 모두 4 viewport 에서 동작 회귀 (Phase 5 §7 R-06)
- 예: AnnotationsPanel 의 reply 입력이 mobile bottom-sheet 에서 키보드로 가려지지 않음

#### 2.1.3 위젯 회귀 보호 (R-06)

기존 위젯들이 사이드바 마운트 후에도 동일 동작을 유지함을 입증.

##### a. 기존 회귀 보존

- TagChipsEditor: 태그 추가/제거 (FG 2-2 회귀)
- ContributorsPanel: 기여자 표시 (FG 3-1 회귀)
- AnnotationsPanel: 주석 CRUD + 1단계 답글 + mention (FG 3-3 회귀)
- VectorizationPanel: 벡터화 트리거 + 상태 표시 (별 FG 회귀)
- DocumentAssignControls: assign / unassign (Pre-flight 에서 위치 확인)
- AgentProposalsTab: agent proposal CRUD (Pre-flight 에서 이동 영향 확인)

##### b. 위젯 마운트 회귀 테스트

`frontend/src/features/documents/__tests__/DocumentSidebar.test.tsx` (신규):
- 각 탭이 올바른 위젯을 마운트
- 탭 전환 시 기존 위젯 unmount/mount 또는 hidden 토글 (Pre-flight 결정에 따라)
- props 통과 회귀 — `documentId` / `currentUserId` 가 모든 탭에 전달

#### 2.1.4 FG 5-2 의 본문 click → annotations 탭 자동 활성화

- `AnnotationMark` click (FG 5-2 §2.1.3) → DocumentDetailPage state `selectedAnnotationId` 설정
- 본 FG 가 동일 click 으로 사이드바의 `activeTab="annotations"` 도 설정 + 사이드바가 collapsed 였다면 자동 펼침
- `AnnotationGutter` 의 도트 click 도 같은 동작

#### 2.1.5 함수도서관 갱신

- `docs/함수도서관/frontend.md` — `DocumentSidebar` + `SidebarTab` 타입 + viewport hook (있으면) 등록
- 기존 위젯들은 기존 그대로 (DocumentSidebar 가 새 컨테이너)

### 2.2 제외 (이월)

- **사이드바 사용자 커스터마이징** (탭 순서 변경 / 숨김) — 별 라운드. 본 FG 는 default 탭 순서 고정.
- **사이드바의 반응형 폭 사용자 조절** (drag handle) — 별 라운드.
- **RagPanel 의 사이드바 통합** — RagPanel 은 별 영역 유지. Pre-flight 에서 통합 가능성 평가 후 결과만 보고서에 기록.
- **AgentProposalsTab 의 본문 하단 → 사이드바 탭 이동** — Pre-flight 결과에 따라 본 FG 또는 별 라운드.

### 2.3 하드코딩 금지 재확인

- viewport 임계값은 단일 정본 (예: `frontend/src/lib/breakpoints.ts` 또는 Tailwind config). 컴포넌트 인라인 금지.
- z-index 매트릭스는 `frontend/src/lib/zIndex.ts` 단일 정본.
- 탭 정의 (`SidebarTab` 타입 + 라벨) 는 단일 정본.

---

## 3. 선행 조건

- FG 5-2 의 AnnotationsPanel props 확장 (`selectedAnnotationId` / `onSelectAnnotation`) 완료 — 본 FG 의 양방향 연동 가능
- Pre-flight 실측 (§2.1.0) 완료 + 운영자 합의 (viewport 임계 + 탭 순서)
- 기존 5 위젯 회귀 베이스라인 — 각 FG 의 회귀 테스트 녹색 확인

---

## 4. 구현 단계

### Step 1 — Pre-flight (§2.1.0)

1. `DocumentDetailPage.tsx` 전수 + 위젯 inventory
2. viewport 임계 + z-index 매트릭스 결정
3. 디자인 리뷰 1회차 (총 ≥ 2회 중 1차) — 사이드바 mockup 검토 + 운영자 합의

### Step 2 — DocumentSidebar 골격 (§2.1.1)

1. 탭 바 + 탭 콘텐츠 컨테이너 신설
2. 각 탭에 기존 위젯 마운트 (props 통과)
3. URL / state 동기화 (활성 탭)
4. 접근성 — ARIA / 키보드 / focus trap
5. 단위 테스트 ≥ 8 — 탭 전환 / props 통과 / URL 동기화 / 키보드 네비

### Step 3 — 4 viewport 동작 (§2.1.2)

1. desktop / narrow — 고정 사이드바 CSS
2. tablet — drawer + FAB
3. mobile — bottom-sheet 또는 fallback
4. 디자인 리뷰 2회차 — 4 viewport 시각 검증
5. 회귀 테스트 ≥ 4 — 각 viewport 별 1 시나리오

### Step 4 — 기존 위젯 회귀 보호 (§2.1.3)

1. 5 위젯 회귀 테스트 — 각 FG 의 기존 회귀가 사이드바 안에서도 녹색
2. AnnotationsPanel 의 reply 입력 + mention typeahead (FG 5-3 와 통합) 회귀
3. node:test 신규 ≥ 5

### Step 5 — FG 5-2 양방향 연동 (§2.1.4)

1. AnnotationMark click → activeTab="annotations" + selectedAnnotationId
2. AnnotationGutter 도트 click → 동일 동작
3. 회귀 테스트 ≥ 3

### Step 6 — 함수도서관 / 검수 / 보안 / UI 디자인 리뷰

1. `docs/함수도서관/frontend.md` 갱신
2. `FG5-4_검수보고서.md` — R-06 (위젯 회귀 보호) + 4 viewport 동작 명시
3. `FG5-4_보안취약점검사보고서.md`:
   - 사이드바가 ACL 결정점이 아님을 정적 분석 (각 위젯 자체 ACL 정본 유지)
   - drawer / bottom-sheet 가 다른 페이지 / 위젯 의 데이터를 누설하지 않음
   - z-index 매트릭스가 modal / dialog 와 충돌하지 않아 모달 콘텐츠가 사이드바에 가려지지 않음
4. **UI 디자인 리뷰 ≥ 2회** (Phase 5 §4 산출물 규약 — 본 FG 만 ≥ 2회 의무)

### Step 7 — Phase 5 1라운드 게이트 통합 회귀

1. Phase 5 §5.1 항목 전체 회귀 — task5-1 / task5-2 / task5-3 의 회귀 통합 녹색 확인
2. node:test / pytest / tsc 베이스라인 유지

---

## 5. API 계약 변경 요약

| 메서드 | 경로 | 변경 |
|-------|------|------|
| (없음) | (없음) | 본 FG 는 frontend 전용. backend API 변경 없음. |

---

## 6. 데이터 모델 주의사항

- DB 스키마 변경 없음.
- URL query (`?tab=annotations`) 추가 — 기존 query 와 충돌 없는지 Pre-flight 에서 확인. 충돌 시 prefix (`sidebarTab=`) 채택.

---

## 7. 성공 기준

- [ ] Pre-flight 보고서 + viewport 임계 + 탭 순서 운영자 합의
- [ ] DocumentSidebar 컴포넌트 신설 + 6 탭 마운트
- [ ] URL / state 활성 탭 동기화
- [ ] ARIA / 키보드 / focus trap 접근성 통과
- [ ] desktop / narrow / tablet / mobile 4 viewport 동작 + 회귀 ≥ 4 시나리오
- [ ] 기존 5 위젯 회귀 ≥ 5 시나리오 (Phase 5 §7 R-06)
- [ ] AnnotationMark click → activeTab="annotations" + 사이드바 자동 펼침 회귀 ≥ 3 시나리오
- [ ] AnnotationGutter 도트 click → 동일 동작 회귀
- [ ] z-index / floating UI 충돌 회귀 (MentionPopup + 사이드바 + modal)
- [ ] **UI 디자인 리뷰 ≥ 2회** 통과 (Phase 5 §4 산출물 규약)
- [ ] node:test 신규 ≥ 12 / 베이스라인 유지
- [ ] tsc 0 error
- [ ] `docs/함수도서관/frontend.md` 갱신
- [ ] `FG5-4_검수보고서.md` + `FG5-4_보안취약점검사보고서.md` 제출
- [ ] Phase 5 §5.1 1라운드 완료 기준 통합 회귀 녹색

---

## 8. 리스크

| # | 리스크 | 대응 |
|---|-------|-----|
| R-01 | 기존 위젯이 사이드바 안에서 깨짐 (R-06) | 각 FG 의 기존 회귀 ≥ 5 시나리오 보존 + 마운트 회귀 별도 추가 |
| R-02 | 좁은 화면에서 사이드바가 본문 가림 | 4 viewport UI 디자인 리뷰 ≥ 2회 의무 + tablet 이하는 collapsible/drawer/bottom-sheet 채택 |
| R-03 | URL query 활성 탭이 기존 query 와 충돌 | Pre-flight 에서 충돌 검토 + prefix 채택 |
| R-04 | drawer focus trap 버그 — Esc 시 trigger 로 복귀 안 됨 | focus trap 라이브러리 (예: @floating-ui/react-dom-interactions) 또는 기존 hook 재사용 + 회귀 |
| R-05 | mobile bottom-sheet 가 키보드로 가려져 reply 입력 불가 | viewport-aware safe-area 적용 + 회귀 |
| R-06 | z-index 충돌로 MentionPopup 이 사이드바에 가림 | `zIndex.ts` 단일 정본 + 회귀 시나리오 |
| R-07 | 비활성 탭 unmount 시 위젯 상태 손실 (예: AnnotationsPanel reply 입력 중) | Pre-flight 결정 — 비활성 탭은 hidden (state 보존) 또는 unmount + state 외부화. 기본은 hidden 권장 |
| R-08 | 디자인 리뷰 ≥ 2회 일정 지연 | 1차는 mockup 단계 (구현 시작 전), 2차는 4 viewport 구현 후. 일정 분리로 병행 가능 |
| R-09 | RagPanel / AgentProposalsTab 이 사이드바 외 영역 유지 → UI 일관성 깨짐 | Pre-flight 에서 통합 가능성 평가 + 본 FG 외 결정은 별 라운드 명시 |

---

## 9. 참조

- `docs/개발문서/S3/phase5/Phase 5 개발계획서.md` §1.4 (기대 결과 — 사이드바), §6.2 (보류 결정 — viewport 임계), §7 R-04 / R-06
- `docs/개발문서/S3/phase5/작업지시서/task5-2.md` §2.1.3 (양방향 연동 — 사이드바가 통과시킬 props)
- `docs/개발문서/S3/phase5/작업지시서/task5-3.md` §2.1.2 (MentionPopup z-index)
- `frontend/src/features/documents/DocumentDetailPage.tsx`
- `frontend/src/features/documents/AnnotationsPanel.tsx`
- `frontend/src/features/documents/ContributorsPanel.tsx`
- `frontend/src/features/documents/VectorizationPanel.tsx`
- `frontend/src/features/documents/AgentProposalsTab.tsx`
- `frontend/src/features/tags/TagChipsEditor.tsx`
- `CONSTITUTION.md` 제5·9·11·12·15조

---

*작성: 2026-05-10 | FG 5-4 — DocumentDetailPage 우측 사이드바 본격 도입 + 4 viewport + UI 디자인 리뷰 ≥ 2회*
