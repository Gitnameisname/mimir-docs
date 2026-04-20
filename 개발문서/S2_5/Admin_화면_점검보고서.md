# 관리자 설정 화면 점검 보고서 (25페이지 전체 훑기)

- 작성일: 2026-04-20
- 범위: `/admin/*` 전체 — `ADMIN_NAV_ITEMS` 23개 + `/admin`, `/admin/document-types/[type_code]`, `/admin/users/[user_id]`
- 방법: 정적 분석(grep across `features/admin/**/*Page.tsx`, `components/admin/DataTable.tsx`) + Chrome 런타임 점검(대시보드·사용자·API 키·스케줄·Scope Profile·역할)
- 관점: ① 시각 일관성 ② 접근성/키보드 ③ 정보구조/UX 흐름 ④ 정보 밀도/레이아웃

---

## 요약 — 심각도별 이슈 개수

| 심각도 | 시각 일관성 | 접근성/키보드 | 정보구조/UX | 정보 밀도/레이아웃 | 합계 |
| --- | --- | --- | --- | --- | --- |
| P0 (즉시) | 2 | 2 | 1 | 0 | **5** |
| P1 (단기) | 3 | 3 | 2 | 2 | **10** |
| P2 (중기) | 2 | 2 | 2 | 2 | **8** |

핵심: Admin 영역 전반에 **4~5가지 서로 다른 "페이지 템플릿 스타일"이 공존**한다. 2025년 이전/이후에 추가된 페이지가 서로 다른 convention 을 따르며 리팩토링되지 않아, "같은 메뉴 그룹인데 생김새는 다른" 경험을 준다. 가장 큰 영향은 **시각 일관성**과 **키보드 접근성** 두 축이다.

---

## ① 시각 일관성 (Visual Consistency)

### VI-1 [P0] 페이지 루트 래퍼가 3가지로 분기

`features/admin/**/Admin*Page.tsx` 의 최상위 `<div>` className 을 수집한 결과:

| 패턴 | 페이지 수 | 페이지 |
| --- | --- | --- |
| **A. `p-3 sm:p-6 space-y-6 max-w-7xl`** (신규) | 13 | dashboard, proposals, usage, scope-profiles, prompts, capabilities(`max-w-5xl`), providers, extraction-queue, agent-activity, golden-sets, extraction-schemas, agents, evaluations |
| **B. `p-4 sm:p-6 space-y-5`** (중기) | 7 | monitoring, users, api-keys, alerts, roles, settings, jobs/schedules |
| **C. `p-6 space-y-5` / `p-6 space-y-6` (responsive 없음)** | 4 | document-types, document-types/[code] 탭, vectorization, users/[id] |
| **D. `flex h-full` split-pane** | 3 | orgs, audit-logs, jobs/runs |

증상:
- 같은 Admin 영역인데 페이지마다 좌우 padding(12px/16px/24px)과 컨텐츠 max-width(`7xl`/`5xl`/없음) 가 달라, 사이드바 토글 시 **내용 왼쪽 정렬선이 통통 튄다**.
- `space-y-5` vs `space-y-6` 혼재 → 섹션 간 여백이 8px 단위로 들쭉날쭉.

처방(P0):
- 공용 `AdminPageShell` 컴포넌트 1개 도입. 기본: `p-3 sm:p-6 space-y-6 max-w-7xl`. split-pane 은 `variant="split"` 옵션. 모든 `Admin*Page.tsx` 를 이 shell 로 감싸기.

### VI-2 [P0] Admin 영역 primary 색이 빨강/파랑 혼재

`features/admin/**/*.tsx` 기준:

| 토큰 | red 계열 사용 | blue 계열 사용 |
| --- | --- | --- |
| `bg-*-600` primary 버튼 | 약 40+ (대세) | **AdminApiKeysPage** 2건 |
| `focus-visible:ring-*-500` | red-500 95건 | **AdminApiKeysPage** 3건 |
| 탭 active 배경 | red 0건 | **app/admin/jobs/page.tsx** `bg-blue-600 text-white` |

증상:
- Admin 로고/뱃지/헤더는 red 팔레트인데 `/admin/api-keys` 만 blue 버튼·blue focus-ring → 해당 페이지에서 "다른 영역으로 넘어온 느낌".
- `/admin/jobs` 의 스케줄/실행이력 탭은 파란색 active — 동일 페이지 내부의 빨강 버튼들과 대비가 깨짐.

처방(P0):
- `AdminApiKeysPage.tsx`: `bg-blue-600 hover:bg-blue-700 ring-blue-500` → `bg-red-600 hover:bg-red-700 ring-red-500` 일괄 치환(총 5건).
- `app/admin/jobs/page.tsx` 탭: active 배경 `bg-blue-600` → `bg-red-600`, focus-ring 도 `red-500` 로.

### VI-3 [P1] h1 타이포가 3종 혼재

동일 수준의 "페이지 제목" 에 세 가지 클래스가 공존:

| 클래스 | 페이지 수 |
| --- | --- |
| `text-2xl sm:text-3xl font-bold` | 13 (신규 그룹) |
| `text-xl sm:text-2xl font-semibold` | 8 |
| `text-xl font-semibold` (responsive 없음) | 4 (audit-logs, vectorization, jobs runs, doc-types 2종) |

처방(P1):
- `AdminPageHeader` 컴포넌트로 통일. 기본 `text-2xl sm:text-3xl font-bold text-gray-900`.

### VI-4 [P1] `amber-50/amber-200` notice 박스가 일회성 처방처럼 흩어져 있음

`AdminEvaluationsPage`, `AdminExtractionQueuePage`, `AdminAgentActivityPage` 에서 같은 "경량 경고 박스" 를 각각 인라인으로 재구현 (`rounded-xl bg-amber-50 border border-amber-200 p-4 flex items-start gap-3`). 동일 구조를 3군데서 복붙.

처방(P1):
- `<Notice variant="warning" | "info">` 컴포넌트 1개로 추출.

### VI-5 [P1] 대시보드 KPI 카드의 `-` placeholder

/admin/dashboard 에서 "총 사용자 -", "총 문서 -", "실행 중 작업 -", "실패 작업 -" 네 카드가 모두 `-` 만 표기. (Chrome 실측)

처방(P1):
- 로딩 중이면 DataTable 과 동일한 skeleton 처리, 값이 정말 0 이면 `0` 숫자로 표기. 의미 없는 `-` 는 제거.

### VI-6 [P2] `bg-black/40` 드로어/모달 오버레이가 페이지마다 개별 구현

5개 페이지(scope-profiles, prompts, providers, proposals, doc-types-detail) 에서 모달/드로어를 각자 `fixed inset-0 z-50 bg-black/40` 로 재구현. ESC 닫기·포커스 트랩·스크롤 락 구현은 페이지별로 다르거나 누락.

처방(P2):
- `<Drawer>` / `<Modal>` 재사용 컴포넌트로 통합.

### VI-7 [P2] 버튼 높이 토큰 혼재 (`min-h-[32px]` / `[36px]` / `[40px]` / `[44px]`)

104건의 `min-h-[…]` 인라인 하드코딩. 같은 "작은 액션 버튼" 이 페이지마다 32/36 사이에서 다르게 지정.

처방(P2):
- `button` 공용 컴포넌트 3-size(sm/md/lg) 로 통일.

---

## ② 접근성 / 키보드 (Accessibility)

### AX-1 [P0] 6개 페이지에서 `<tr onClick>` 이 키보드 접근 불가

`scope-profiles`, `ai-platform/prompts`, `extraction-queue`, `golden-sets`, `extraction-schemas`, `agents` 가 전부 동일 패턴:

```tsx
<tr key={…} className="hover:bg-gray-50 transition-colors cursor-pointer" onClick={() => setSelected(p)}>
```

- 마우스 사용자만 상세 드로어를 열 수 있음.
- `tabIndex`, `role="button"`, `onKeyDown(Enter|Space)` 모두 부재 → 키보드·스크린리더 사용자는 상세 조회 불가.

처방(P0):
- `<tr>` 에 `tabIndex={0} role="button" aria-label={…} onKeyDown={(e)=>{ if(e.key==="Enter"||e.key===" ") onRowClick(); }}` 추가 — 공용 `DataTable.tsx` 에 일괄 반영하여 기존 6페이지 전환(아래 AX-2 와 함께).

### AX-2 [P0] `DataTable` 공용 컴포넌트 자체에도 동일 결함

`components/admin/DataTable.tsx` L66-80:
```tsx
<tr key={rowKey(row)} onClick={onRowClick ? () => onRowClick(row) : undefined} ...>
```
→ `onRowClick` 사용 중인 users, orgs, doc-types, audit-logs, jobs-runs 까지 전부 키보드 차단 상태.

처방(P0):
- `DataTable` 에 `onKeyDown` + `tabIndex` + `role="button"` 추가. 별도 prop 필요 없이 `onRowClick` 있으면 자동 활성화.

### AX-3 [P1] `<th>` 에 `scope="col"` 미지정

`features/admin` 전체에서 `scope="col"` 0건. `<caption>` 도 0건. 스크린리더가 열 헤더·표 목적을 읽지 못함.

처방(P1):
- `DataTable` 의 `<th>` 에 `scope="col"` 추가. 20개 raw table 들도 점진적으로 `DataTable` 로 전환.

### AX-4 [P1] 정렬 가능한 컬럼 표기 없음

users, audit-logs, agents, prompts, proposals 등 정렬 가능해야 자연스러운 화면에서 컬럼 정렬 기능·UI 가 전무. 또한 정렬 상태를 표현할 `aria-sort` 도 없음.

처방(P1):
- `DataTable.Column<T>` 에 `sortable?: boolean` 옵션 추가, 헤더 클릭 시 `aria-sort="ascending|descending"` 토글.

### AX-5 [P1] 필터 영역이 레이블 없이 `select` 단독

`AdminUsersPage` (실측): "역할/상태" 필터 `<select>` 에 `<label>` 이 없음. placeholder 역할의 `<option value="">전체</option>` 만 존재. 유사 패턴이 alerts, audit-logs, api-keys, proposals 등에 반복.

처방(P1):
- `<label htmlFor="">` + `sr-only` 또는 눈에 보이는 레이블 추가. select 의 `aria-label` 최소.

### AX-6 [P2] `focus:outline-none` 을 남발하며 `focus-visible` 로만 대체

165건 중 상당수가 `focus:outline-none focus-visible:ring-2 focus-visible:ring-*-500`. 정상 동작은 하지만 Safari 의 `:focus-visible` 엣지 케이스에서 **완전히 포커스가 보이지 않는 구간**이 생김.

처방(P2):
- Safari 대응으로 `focus:ring-…` 도 함께 유지하거나, Tailwind v4 `@theme` 토큰에서 `--ring-admin` 으로 집중.

### AX-7 [P2] 대시보드 "Invalid Date" 표기 (Chrome 실측)

/admin/dashboard 의 "최근 감사 이벤트" 표 `시간` 컬럼이 `Invalid Date` 로 렌더링. 날짜 포맷 실패를 사용자에게 날것 그대로 노출.

처방(P2):
- `formatDateTime(input)` 공용 유틸: 파싱 실패 시 `—` 또는 `방금` 같은 graceful fallback 반환.

---

## ③ 정보 구조 / UX 흐름

### IA-1 [P0] `/admin/jobs` 의 페이지 title 이 "Mimir" 로 남음

`app/admin/jobs/page.tsx` 는 `"use client"` 이므로 `export const metadata` 가 먹지 않음 → 브라우저 탭 제목이 "Mimir" 로 남고 AdminSidebar 의 `aria-current` 와 혼동. (Chrome 실측)

처방(P0):
- `page.tsx` 에서 client 로직을 `<JobsTabs />` 에 위임하고 page.tsx 는 server component 로 되돌려 `metadata` 복원. 또는 클라이언트에서 `useEffect(() => { document.title = "백그라운드 작업 — Mimir Admin" }, [])` 로 보정.

### IA-2 [P1] "백그라운드 작업" 과 "배치 작업 관리" 의 Navigation 혼선

사이드바에 `백그라운드 작업 /admin/jobs` 이 있고, 그 페이지 내부 탭이 "스케줄 / 실행 이력" 이며 스케줄 탭 h1 은 "배치 작업 관리" 이다. **사이드바 이름 ≠ h1 이름** → 현재 위치를 네 번 다르게 부르는 셈.

처방(P1):
- 사이드바 항목 명칭을 `배치 작업` 으로 통일, 탭도 `스케줄 / 실행` 유지하고 스케줄 탭 내부 h1 제거(h1 은 페이지 1개).

### IA-3 [P1] 감사 이벤트 표에 actor 가 UUID 로 노출 (Chrome 실측)

대시보드 "최근 감사 이벤트" 에 actor `558cd010-9f8a-4906-a069-82a863b4ef42` 가 그대로 표시. 감사 이벤트 자체는 의도적으로 식별자 보존이나 UI 에서는 `display_name`/`email` 로 대체, hover/링크로 full ID 접근 가능해야.

처방(P1):
- `/api/admin/audit-logs` 응답에 `actor_display_name` include 또는 FE 에서 users 캐시 join.

### IA-4 [P2] Empty state 문구가 페이지마다 미묘하게 다름

20개 이상 변형: "…이 없습니다.", "등록된 …이 없습니다.", "데이터가 없습니다.", "…수집 중…", "—". 정보 위계가 흐려짐.

처방(P2):
- 공용 `EmptyState` 컴포넌트: 아이콘 + 주제어 + "{동작}" CTA 3요소. `DataTable` 에도 내장.

### IA-5 [P2] 위험 액션(삭제) 의 confirm 이 페이지마다 다른 형태

삭제 확인이 어떤 페이지는 `confirm()` 네이티브, 어떤 페이지는 `<Modal>`, 어떤 페이지는 인라인 "정말 삭제?" 토글로 처리. 일관성 없음.

처방(P2):
- `<ConfirmDialog>` 공용 컴포넌트 + 위험 액션 규약(주황 경고 + 파괴적 뉘앙스 문구 + 2차 확인).

---

## ④ 정보 밀도 / 레이아웃

### DL-1 [P1] 필터 줄이 모바일에서 줄바꿈 후 간격이 붕괴

`AdminUsersPage`, `AdminAlertsPage`, `AdminAuditLogsPage` 에서 `flex flex-wrap gap-2` 로 검색+필터를 한 줄에 올림. 모바일에서 wrap 되면 "검색 버튼" 이 단독 줄에 떨어짐.

처방(P1):
- `grid grid-cols-1 sm:grid-cols-[1fr_auto_auto_auto]` 로 변경하거나, 모바일에서 "필터 접기" 버튼으로 드로어 처리.

### DL-2 [P1] split-pane 3페이지 가 `min-h-0` 누락 의심

`orgs`, `audit-logs`, `jobs/runs` 의 `<div className="flex h-full">` 내부 child 에 `min-w-0/min-h-0` 체인이 불완전하면 긴 셀이 오른쪽 detail pane 을 뚫고 나옴.

처방(P1):
- split-pane 공용 `<SplitPane left right />` 로 추출하고 `min-w-0` 체인 강제.

### DL-3 [P2] `max-w-7xl` vs `max-w-5xl` 혼재 이유 불명

capabilities 만 `max-w-5xl`. 의도인지 오타인지 불명.

처방(P2):
- 페이지 카테고리별(리스트/상세/대시보드) 명시 규칙 수립 후 일괄.

### DL-4 [P2] 테이블 가로 스크롤 wrapper 가 반쪽만 적용

13페이지에 `overflow-x-auto`, 나머지는 컬럼이 많아도 wrapper 없음 → 모바일에서 가로 스크롤 깨짐.

처방(P2):
- `DataTable` 내장, raw table 은 커스텀 wrapper 추가.

---

## 부록 A. 페이지별 스냅샷 (정적 분석 + Chrome)

| 페이지 | 루트 패턴 | h1 | Primary | 비고 |
| --- | --- | --- | --- | --- |
| /admin/dashboard | A | 2xl bold | — | KPI `-` 다수, Invalid Date (실측) |
| /admin/users | B | xl semibold | red | `<tr onClick>` 3건 (실측), label-less select |
| /admin/organizations | D | (split-pane) | red | |
| /admin/roles | B | xl semibold | red | 시스템/커스텀 분리 양호 |
| /admin/settings | B | xl semibold | red | Toggle switch 자체구현 |
| /admin/monitoring | B | xl semibold | red | |
| /admin/alerts | B | xl semibold | red | label-less filters |
| /admin/jobs (runs) | D | (split-pane) | red | 탭 active=blue, title 미설정 (실측) |
| /admin/jobs (schedules) | B | xl semibold | red | `min-h-[32px]` 남발 |
| /admin/audit-logs | D | xl no-resp | red | actor UUID 노출 |
| /admin/api-keys | B | xl semibold | **blue** | 빨강 원칙 위반 |
| /admin/ai-platform/providers | A | 2xl bold | red | 드로어 자체구현 |
| /admin/ai-platform/prompts | A | 2xl bold | red | `<tr onClick>` |
| /admin/ai-platform/capabilities | A (`max-w-5xl`) | 2xl bold | — | |
| /admin/ai-platform/usage | A | 2xl bold | — | |
| /admin/agents | A | 2xl bold | red | `<tr onClick>` |
| /admin/scope-profiles | A | 2xl bold | red | `<tr onClick>` |
| /admin/proposals | A | 2xl bold | red | 드로어 자체구현 |
| /admin/agent-activity | A | 2xl bold | red | amber notice |
| /admin/golden-sets | A | 2xl bold | red | `<tr onClick>` |
| /admin/evaluations | A | 2xl bold | red | amber notice |
| /admin/extraction-schemas | A | 2xl bold | red | `<tr onClick>` |
| /admin/extraction-queue | A | 2xl bold | red | `<tr onClick>`, amber notice |
| /admin/vectorization | C | xl no-resp | red | |
| /admin/document-types | C | xl no-resp | red | DataTable 사용 |
| /admin/document-types/[code] | C | xl | red | 긴 탭/여러 섹션 |

---

## 부록 B. 우선순위별 수정 플랜 (제안)

**1차 (P0, 1개 PR 권장)**
1. `DataTable` 에 키보드 접근성 주입 (AX-2) — 6페이지 부작용 없이 즉시 해소
2. `AdminApiKeysPage` blue → red 일괄 치환 (VI-2)
3. `app/admin/jobs/page.tsx` 탭 blue → red, `useEffect` 로 document.title 복원 (VI-2, IA-1)
4. 6개 페이지의 bespoke `<tr onClick>` 을 `DataTable` 또는 인라인 키보드 핸들러로 교체 (AX-1)
5. `AdminPageShell` 컴포넌트 도입 + 전 페이지 감싸기 (VI-1)

**2차 (P1)**
- VI-3 AdminPageHeader 통일 / VI-4 Notice 추출 / VI-5 KPI placeholder 교체
- AX-3 `scope="col"` / AX-4 정렬 UI / AX-5 필터 레이블
- IA-2 네이밍 통일 / IA-3 actor 표시

**3차 (P2)**
- VI-6 Modal/Drawer 추출 / VI-7 버튼 사이즈 토큰
- AX-6 Safari focus / AX-7 date formatter
- IA-4 EmptyState / IA-5 ConfirmDialog
- DL-1~DL-4 레이아웃 리파인

---

## 결론

- **Critical 2건**: blue→red (api-keys/jobs 탭), `<tr>` 키보드 접근 — 즉시 수정 시작 권장
- **High 3건**: 루트 레이아웃 shell 통일, 탭 title/label 혼선, th scope
- 기타 자잘한 일관성 이슈는 shell/Table/Header/Notice 4개 공용 컴포넌트 정리로 대부분 해소 가능

다음 단계: **1차(P0) 수정 PR 먼저 진행**하고, 수정 후 동일한 기준으로 회귀 점검 보고서 작성.
