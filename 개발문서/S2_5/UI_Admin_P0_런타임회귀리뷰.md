# Admin UI P0 런타임 회귀 리뷰 보고서 (Chrome 5+회)

- 작성일: 2026-04-20
- 대상: P0-1 ~ P0-5 수정 결과의 Chrome 런타임 시각·키보드 회귀
- 방법: Chrome MCP(localhost:3050, ORG_ADMIN=`admin@mimir.local` 세션) 로 9개 admin 경로를 순회하며 스크린샷·키보드 조작 수행
- 원 점검 보고서: `docs/개발문서/S2_5/Admin_화면_점검보고서.md`
- 원 수정 검수: `docs/개발문서/S2_5/UI_Admin_P0_검수보고서.md`

---

## 리뷰 요약

| 라운드 | 경로 | 초점 | 결과 |
| --- | --- | --- | --- |
| R1 | `/admin/roles` | 사이드바/헤더 chrome, primary 버튼 | ✅ blue chrome, "역할 추가" blue primary, 권한 매트릭스 체크는 green(의미) |
| R2 | `/admin/dashboard` | chrome, KPI, 시스템 상태 | ✅ blue chrome, "오류" 상태 dot/text 는 red(semantic 유지), "상세 관리/모니터링" 링크 blue |
| R3 | `/admin/api-keys` | 기존 blue outlier 확인 | ✅ "+ API 키 생성" blue 유지(기존 정책과 일치), destructive 모달은 코드상 red 유지 |
| R4 | `/admin/jobs` | 브라우저 탭 title (P0-5) | ✅ tab title = "백그라운드 작업 — Mimir Admin" (이전 "Mimir" 에서 정상화). 탭 active=blue |
| R5 | `/admin/scope-profiles` | 리스트 페이지 chrome (P0-3) | ✅ "+ Scope Profile 생성" blue, 사이드바 active blue |
| R6 | `/admin/users` (DataTable 핵심 회귀) | **P0-2 키보드 접근성** | ✅ Tab 키로 첫 행 포커스 → `focus-visible:bg-blue-50 ring-2 ring-blue-500` 가시화 → Enter 키로 `/admin/users/{id}` 상세 이동 |
| R7 | `/admin/users/[id]` | destructive red 보존 | ✅ 우측 상단 "삭제" 버튼 red 유지, 조직 역할 행의 "제거" red 유지, "+ 조직 역할 부여" blue |
| R8 | `/admin/agents` | 신규 페이지 chrome | ✅ "+ 에이전트 생성" blue primary |
| R9 | `/admin/settings` | 탭 active, toggle, 저장 | ✅ "인증" 탭 blue underline, 토글 on=blue bar, "저장" blue primary |

**5+회 목표(#19) 초과 달성(9회)**. P0-1 ~ P0-5 모두 브라우저에서 정상 렌더링.

---

## 키 검증 포인트

### 1. P0-5 `document.title` 복원

`/admin/jobs` 로 네비게이션 직후 Chrome 탭 메타:
- **이전**: `"Mimir"` (client-only page 에서 metadata 가 무시됨)
- **현재**: `"백그라운드 작업 — Mimir Admin"` ← Next metadata 가 server component 로부터 정상 주입됨을 확인

### 2. P0-2 DataTable 키보드 접근성

`/admin/users` 에서 `<input 이메일/이름 검색>` 클릭 후 `Tab × 4` 로 포커스 이동:
- 첫 번째 사용자 행(`cck1835@naver.com / 최철균 / VIEWER`) 에 **blue 2px ring + 연한 blue 배경** 이 즉시 나타남.
- 줌 샷(`(148,200)-(1500,280)`) 으로 `focus-visible:bg-blue-50 focus-visible:ring-inset focus-visible:ring-blue-500` 가 의도대로 그려지는 것을 육안 확인.
- 이어서 `Enter` → 행이 활성화되어 `/admin/users/bfd8965d-bc5f-4580-9a02-33e13ab48856` 로 정상 이동 (onKeyDown 핸들러 동작).

즉 `DataTable.tsx` 의 `role="button" + tabIndex=0 + onKeyDown(Enter|Space)` 계약이 마우스 사용자와 동일한 결과를 만든다.

### 3. P0-3 팔레트 스왑

9개 경로에서 공통 확인:
- primary 액션 버튼(`+ 생성`, `저장`, `+ 추가`): 전부 `bg-blue-600` 계열
- 탭 active: blue (underline 또는 filled)
- 토글 ON: blue
- 인라인 링크("상세 관리", "모니터링", "다시 시도", "+ 조직 역할 부여"): blue

보존된 red(예상과 일치):
- 대시보드 "오류" 상태 dot·텍스트(semantic)
- 사용자 상세 "삭제" / "제거" 버튼(destructive)
- 코드상 KillSwitchModal, `폐기` 확인 모달, `거절` 확인 모달(destructive) — 본 리뷰 시점 트리거 없어 스크린샷은 없으나 정적 리뷰에서 red 로 유지됨을 확인 완료

---

## 본 리뷰에서 추가로 관찰된 기존 결함 (P0 범위 밖)

원 점검 보고서에서 이미 P1/P2 로 분류된 항목 중, 이번 런타임 리뷰로 재확인된 건:

- **VI-5 / KPI `-` placeholder** (`/admin/dashboard` 총 사용자·총 문서·실행 중·실패 24H) — 해소되지 않음(P1)
- **IA-3 / 감사 이벤트 actor 가 UUID** (`/admin/dashboard` 최근 감사 이벤트) — 해소되지 않음(P1)
- **AX-7 / `Invalid Date`** (`/admin/dashboard` 감사 이벤트 시간 컬럼) — 해소되지 않음(P2)
- **DL-? / `/admin/agents` h1 description 부재** — 타 페이지 대비 설명 문구가 없어 정보 밀도가 낮음(P2 — 원 보고서에도 유사 기술)

이번 P0 수정 범위에 해당하지 않으므로 별도 후속 태스크에서 처리.

---

## 결론

- **P0-1 ~ P0-5 전 항목 런타임 회귀 통과.**
- DataTable 키보드 접근성은 정적 코드뿐 아니라 브라우저에서도 실제 Tab/Enter 동작이 확인됨.
- 브랜드 팔레트는 blue 로 완전히 수렴했고, destructive/semantic red 는 의도대로 남아 있음.
- **후속 작업**: `Admin_화면_점검보고서.md` 의 P1 트랙 — AdminPageShell 도입(VI-1), KPI placeholder 교체(VI-5), actor 표시(IA-3), DataTable sortable/ariaLabel 확산(AX-3/AX-4).
