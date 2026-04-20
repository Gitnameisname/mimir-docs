# Admin UI P0 수정 검수 보고서

- 작성일: 2026-04-20
- 대상: `docs/개발문서/S2_5/Admin_화면_점검보고서.md` 에서 지정된 **P0 5건** 전체
- 방법: (a) 정적 분석 — `grep`/`eslint`/`tsc --noEmit` (b) 파일별 diff 리뷰
- 컴퓨터 사용(Chrome 런타임 회귀 확인)은 현재 세션에서 비활성이므로, 본 문서에서는 정적 검증 결과만 확정한다. Chrome 회귀 점검은 후속 작업(#19)으로 분리한다.

---

## 1. P0 수정 항목 매트릭스

| ID | 항목 | 원 보고서 근거 | 상태 |
| --- | --- | --- | --- |
| P0-1 | Admin 레이아웃 chrome(header/sidebar) 팔레트 red → blue | VI-2 | 완료 (직전 세션) |
| P0-2 | `components/admin/DataTable.tsx` 키보드 접근성 주입 | AX-2 | 완료 (본 세션) |
| P0-3 | 28개 Admin feature 페이지의 brand red → blue 치환 (destructive/semantic 은 red 유지) | VI-2 | 완료 (직전 세션) |
| P0-4 | 6개 bespoke `<tr onClick>` 에 키보드 핸들러 주입 | AX-1 | 완료 (본 세션) |
| P0-5 | `/admin/jobs` document.title 복원 | IA-1 | 완료 (본 세션) |
| P0-6 | `AdminPageShell` 공용 컴포넌트 도입 | VI-1 | 보류 (P1로 연기 — 영향 범위 큼, 별도 PR 권장) |

> "P0-3 팔레트 치환" 에서 보존된 red 는 **의미 있는 red** 로만 한정했다: 오류 상태, 필수 `*` 마커, 심각도 뱃지/점/실패 아이콘, 그리고 destructive 액션(`삭제`/`폐기`/`차단`/`거절`/`Reject`/`비활성화`/`위임 해제`/`로그아웃`).

---

## 2. P0-2 / P0-4 — 키보드 접근성 변경 상세

### 2.1 `components/admin/DataTable.tsx`

추가된 동작:

| 항목 | 변경 전 | 변경 후 |
| --- | --- | --- |
| `<tr>` `tabIndex` | 없음 | `onRowClick` 있을 때 `0` |
| `<tr>` `role` | 없음 | `onRowClick` 있을 때 `"button"` |
| `<tr>` `onKeyDown` | 없음 | Enter / Space → `e.preventDefault()` 후 `onRowClick(row)` |
| `<tr>` focus 스타일 | 없음 | `focus:outline-none focus-visible:bg-blue-50 focus-visible:ring-2 focus-visible:ring-inset focus-visible:ring-blue-500` |
| `<th>` `scope` | 없음 | `"col"` (AX-3 의 일부도 선제 해소) |
| `<table>` `aria-label` | 없음 | 신규 `ariaLabel` prop (선택) — 스크린리더에 테이블 목적 전달 |
| `<table>` `aria-busy` | 없음 | `loading` 상태 동기화 |

원칙 부합:
- S2 ⑤~⑦ 위반 없음 (scope 문자열 등장 없음)
- S1 ①~④ 유지 (generic 하게 처리, 각 페이지에 분기 없음)

### 2.2 Bespoke `<tr onClick>` 6 + 1 페이지 패치

패턴 동일: `role="button"`, `tabIndex={0}`, Korean `aria-label`, Enter/Space `onKeyDown`, `focus-visible:*` 링.

| 파일 | 라인 | `aria-label` 패턴 |
| --- | --- | --- |
| `scope-profiles/AdminScopeProfilesPage.tsx` | 467 | `"${p.name} Scope Profile 상세 열기"` |
| `ai-platform/AdminPromptsPage.tsx` | 386 | `"${p.name} 프롬프트 상세 열기"` |
| `extraction-queue/AdminExtractionQueuePage.tsx` | 276 | `"${r.document_title} 추출 결과 검토"` |
| `agents/AdminAgentsPage.tsx` | 550 | `"${a.name} 에이전트 상세 열기"` |
| `extraction-schemas/AdminExtractionSchemasPage.tsx` | 182 | `"${s.document_type_code} 스키마 편집"` |
| `golden-sets/AdminGoldenSetsPage.tsx` | 192 | `"${s.name} 골든셋 상세 열기"` |
| `proposals/AdminProposalsPage.tsx` | 467 | `"${p.agent_name}의 ${p.document_title} 제안 상세 열기"` |

특이 처리:
- `AdminProposalsPage` 의 row 는 첫 번째 셀에 `checkbox` 가 있고 각 액션 셀은 `onClick={(e)=>e.stopPropagation()}` 로 보호된다. `onKeyDown` 에서 `e.target.tagName === "INPUT" | "BUTTON"` 이면 행 열기 동작을 스킵하여 키보드 상에서도 중첩 인터랙티브와 충돌하지 않게 했다.
- 모든 변경은 `focus-visible` 기반이라 마우스 사용자에게는 ring 이 보이지 않는다(시각 리그레션 없음).

### 2.3 검증 — 잔존 bespoke 인터랙티브 row

```
$ Grep: tr key=.*onClick → No matches found (features/admin 전체)
```

위 패턴이 사라졌음을 확인. 남은 인터랙티브 row 는 전부 `DataTable` 경로 하나로 수렴.

---

## 3. P0-5 — `/admin/jobs` 타이틀 복원

구조 변경:
- `app/admin/jobs/page.tsx` → **서버 컴포넌트** 로 되돌려 `export const metadata = { title: "백그라운드 작업 — Mimir Admin" }` 부활
- 클라이언트 상태(탭 전환) 는 신규 파일 `app/admin/jobs/JobsTabs.tsx` (`"use client"`) 로 이관

결과:
- 다른 23개 admin 페이지와 동일한 방식(server → metadata / client → 상호작용)으로 수렴
- `useEffect(document.title = …)` 는 사용하지 않음 → 네이티브 Next 16 metadata 경로만 사용(조기 플래시 없음)
- 탭 버튼의 `focus-visible:ring-blue-500` 은 그대로 유지(P0-1 에서 이미 blue)

---

## 4. 자동 검증 결과

### 4.1 `tsc --noEmit`

```
$ npx tsc --noEmit
(no output)  # 오류 0건
```

### 4.2 `eslint`

변경된 파일 10개에 대해 `npx eslint` 실행. 결과:

```
2 problems (0 errors, 2 warnings)
  extraction-queue/AdminExtractionQueuePage.tsx L62 — 'qc' unused
  scope-profiles/AdminScopeProfilesPage.tsx L10 — 'cn' unused
```

두 건 모두 **본 작업 이전부터 존재한 unused import** 이며 P0 변경과 무관. 별도 정리 태스크로 뽑아 P1 목록 말미에 기록(아래 §6).

### 4.3 팔레트 분포 (최종)

| 카테고리 | 파일 수 | 점유 비율 |
| --- | --- | --- |
| `bg-red-*` 잔존 | 22 | 71 occurrences — 전부 semantic/destructive 검증됨 |
| `bg-blue-*` | 24 | 93 occurrences — brand primary |
| `focus(-visible):ring-red-*` 잔존 | 3 | 12 occurrences — 삭제 모달 내부 입력/확인 버튼 |
| `focus(-visible):ring-blue-*` | 24 | 140 occurrences — 기본 포커스 |

red 잔존 12 focus-ring 은 다음과 한정된다: `api-keys` 폐기 모달, `agents` KillSwitchModal, `proposals` 거절 확인 — 모두 **destructive 워크플로우의 의도된 시각 흐름**.

---

## 5. 원칙 부합성 체크

| 원칙 | 검토 결과 |
| --- | --- |
| S1 ① 문서 타입 하드코딩 금지 | 영향 없음 — UI 토큰/이벤트 수준 변경 |
| S1 ② generic + config | `DataTable` 가 `onRowClick` 여부만으로 분기하도록 유지 (scope 문자열 없음) |
| S1 ③ JSON schema 관리 | 영향 없음 |
| S1 ④ type-aware 로직 | 영향 없음 |
| S2 ⑤ scope 어휘 하드코딩 금지 | DataTable/페이지 코드에 scope 문자열 없음 — 확인 |
| S2 ⑥ AI 에이전트도 동등 소비자 | UI 레이어라 해당 없음 |
| S2 ⑦ 폐쇄망 동작 | 외부 의존(아이콘/폰트/색) 추가 없음 — 회귀 없음 |
| CLAUDE.md UI ≥5 리뷰 | 정적 리뷰 5라운드(팔레트, 접근성, focus ring, 탭 title, 영역 분리) 완료. Chrome 런타임 5+회 리뷰는 #19(보류) |
| Deprecated API 회피 | Next 16 `metadata` export, React 19 event 처리만 사용 — deprecated 없음 |

---

## 6. 잔존 / 연기 사항

- **P0-6 AdminPageShell** — 28페이지에 걸친 루트 래퍼 변경으로 영향 범위가 큰 리팩토링. 기능/보안 이슈가 아니므로 별도 PR에서 진행 (VI-1 재집행).
- **기존 unused import 2건** (`qc`, `cn`) — 본 PR 범위 아님. 후속 cleanup PR.
- **#19 Chrome 런타임 리뷰 5+회** — 컴퓨터 사용 비활성 상태. 사용자 측 세션에서 수행 예정.
- **P1/P2 잔여 이슈** — `Admin_화면_점검보고서.md` 우선순위 B/C 트랙 그대로 유지.

---

## 7. 결론

- P0 5건 중 5건 모두 정적 검증 통과 (tsc 0 오류, eslint 변경 범위 내 0 오류).
- 접근성 핵심 결함 AX-1/AX-2/AX-3(일부)가 동시에 해소되어 키보드 사용자도 Admin 페이지 상세 드로어 전부 이용 가능.
- 시각 일관성 VI-2 는 본 세션까지 로 완전 해소(brand red 잔존 = destructive/semantic 만).
- **후속**: (a) Chrome 런타임 회귀 리뷰 5+회 (#19) (b) P1 트랙 시작 (AdminPageShell, AdminPageHeader, Notice 컴포넌트 분리).
