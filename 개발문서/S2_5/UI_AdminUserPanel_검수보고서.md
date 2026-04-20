# Admin 관리자 패널 재배치 — 검수 보고서

- 작성일: 2026-04-20
- 범위: `/admin/*` 영역 헤더·사이드바의 관리자 패널 위치 및 내비게이션 통합
- 관련 규칙: CLAUDE.md S1 ①~④, S2 ⑤~⑦, UI 규칙(데스크탑·웹 호환, 리뷰 ≥ 5회)

## 1. 변경 파일 목록

| 경로 | 성격 | 설명 |
| --- | --- | --- |
| `frontend/src/components/admin/layout/AdminUserPanel.tsx` | 신규 | Admin 사이드바 하단 전용 사용자 패널. Expanded: 아바타+이름+역할+chevron / Rail: 아바타만. 메뉴(프로필 / 계정 설정 / 일반 화면으로 이동 / 로그아웃) 팝업 — Expanded는 위쪽, Rail은 오른쪽 |
| `frontend/src/components/admin/layout/AdminSidebar.tsx` | 리팩터 | aside `overflow-y-auto` → `nav` 내부 스크롤로 변경하여 하단 패널을 뷰포트에 고정. 독립 링크였던 "일반 화면으로 이동" 제거하고 `AdminUserPanel` 메뉴로 통합. 하단에 `<AdminUserPanel />` 추가 |
| `frontend/src/components/admin/layout/AdminHeader.tsx` | 리팩터 | 사용자 정보 블록(아바타+이름+로그아웃) 제거. 우측 "User UI" 복귀 링크도 제거(`AdminUserPanel` 메뉴로 통합). `useAuth` 의존성 제거로 헤더는 presentational |

## 2. S1/S2 원칙 준수 확인

| 원칙 | 확인 항목 | 결과 |
| --- | --- | --- |
| ① 문서 타입 하드코딩 금지 | 해당 없음 (UI 이동만) | N/A |
| ② generic + config | Admin 내비 항목은 기존 `ADMIN_NAV_ITEMS` 배열 config 유지 | OK |
| ③ schema 관리 | 신규 DB/JSON 필드 없음 | N/A |
| ④ type-aware | 서비스 레이어 분기 변경 없음. UI 계층만 | OK |
| ⑤ scope 하드코딩 금지 | `hasMinimumRole("ORG_ADMIN")` 는 `AdminLayout` 의 `AuthGuard` 가 담당. 패널은 단순히 `isAuthenticated && user` 만 체크 | OK |
| ⑥ 에이전트=사람 대등 API 소비자 | 인증/로그아웃은 기존 `useAuth().logout()` 을 경유 → API 계층 변화 없음 | OK |
| ⑦ 폐쇄망 | 외부 폰트/CDN 추가 없음. CSS 클래스(tailwind) + SVG 인라인만 사용 | OK |

## 3. UI 규칙 준수

| 규칙 | 준수 | 근거 |
| --- | --- | --- |
| UI 리뷰 ≥ 5회 | ✅ | Chrome 으로 7회 (목록은 §5) |
| 데스크탑·웹 호환 | ✅ | 기존 `lg:` breakpoint 기반 사이드바 Expanded/Rail/Mobile-overlay 3-mode 유지. 데스크탑 앱의 창 resize 에도 즉시 대응 |

## 4. 설계 요점

- **중복 제거**: "일반 화면으로 이동" 은 사이드바 하단 독립 링크와 헤더 우측 "User UI" 버튼 양쪽에 존재하여 사용자 혼란·시각적 중복을 유발 → 관리자 패널 메뉴 1개 항목으로 통합.
- **Scroll 분리**: 기존 `aside[overflow-y-auto]` 는 nav 항목 22+개가 긴 경우 하단 패널까지 함께 스크롤되어 `AdminUserPanel` 이 뷰포트 바닥에 고정되지 않는 문제가 있었다. `nav[flex-1 min-h-0 overflow-y-auto]` 로 이동시켜 사용자 패널을 sticky 하게 유지.
- **aria-label 유일성**: 헤더 모바일 햄버거가 이미 "관리자 메뉴 열기" 를 사용하므로, 관리자 패널 버튼은 "관리자 계정 메뉴 열기" / 메뉴는 "관리자 계정 메뉴" 로 구별 — 스크린 리더 혼동 방지.
- **팔레트 일관성**: Admin 영역은 기존 red-600 계열 테마를 유지하므로 `AdminUserPanel` 도 red-100/red-700 아바타, red-500 focus-ring, role 표시에 red-600 bold 적용. 사용자 UI(`SidebarUserPanel`)의 brand 토큰과는 의도적으로 구분.
- **메뉴 구성 단순화**: Admin 에서는 이미 관리자 모드이므로 "관리자 설정" 항목 불필요. 프로필 / 계정 설정 / 일반 화면으로 이동 / 로그아웃 4개 항목으로 정돈.

## 5. Chrome 기반 UI 리뷰 (7회)

| # | 시나리오 | 결과 |
| --- | --- | --- |
| 1 | Expanded 하단 렌더·크기 | aside width 224, userBtn width 207, bottom 1263 ≈ aside bottom 1271 (8px padding) ✓ |
| 2 | Expanded 메뉴 위쪽 팝업 | menu rect top 1002 / bottom 1210.5 (btn.top 1214.5 위) ✓. 4개 항목 렌더 |
| 3 | Rail(collapsed) 모드 | aside width 64, userBtn width 51, title="관리자", textContent="관" (이니셜만) ✓ |
| 4 | Rail 메뉴 오른쪽 팝업 | menu left 71 > rail right 64 (gap 7px), width 224 ✓ |
| 5 | 외부 mousedown / ESC 닫기 | 둘 다 menu 언마운트 확인 ✓ |
| 6 | "일반 화면으로 이동" 메뉴 통합 & 중복 제거 | 사이드바 하단 독립 링크 없음, 헤더 "User UI" 없음, 메뉴에 `일반 화면으로 이동` 항목 포함 ✓ |
| 7 | 메뉴 항목 클릭 → 실제 navigation | `일반 화면으로 이동` 클릭 → `/documents` 로 이동, `/admin` 컨텍스트 벗어남 ✓ |

## 6. 기능 회귀 검사

| 항목 | 결과 | 비고 |
| --- | --- | --- |
| 관리자 페이지 렌더(`/admin/dashboard`, `/admin/users`) | ✅ | 네비 active 하이라이트 유지 (`사용자 관리` aria-current 확인) |
| 사이드바 collapse 토글 | ✅ | 헤더 버튼 click → aside 224 ↔ 64 전환 |
| 로그아웃 플로우 | ✅ | `logout()` → `/login` 리다이렉트 기존 흐름 그대로 |
| 사용자 UI 복귀 | ✅ | 메뉴의 `일반 화면으로 이동` 으로 `/documents` 이동 |
| 콘솔 에러 | ✅ | Hydration/Fast-refresh 외 에러 없음 |

## 7. 잔존 과제 / Follow-up

| 항목 | 우선순위 | 비고 |
| --- | --- | --- |
| 모바일 overlay 드로어에서 패널 하단 고정 동작 | P2 | 현재 `flex flex-col` + `min-h-0 overflow-y-auto` 로 큰 문제 없어 보이나 실기기 확인 권장 |
| Admin 테마를 `@theme` 토큰화(red brand) | P3 | 기존 raw 클래스(`bg-red-100`) 그대로. S2.5 후속 라운드에서 토큰 도입 고려 |
| User UI 패널(`SidebarUserPanel`) 도 동일한 "관리자 화면으로" 항목 검토 | P3 | 현재 `관리자 설정 → /admin` 으로 이동 가능하므로 별도 불필요 |

## 8. 결론

본 변경은 헤더/사이드바에 **중복 배치되어 있던 "일반 화면으로 이동"을 관리자 계정 메뉴 1곳으로 통합**하고, 사이드바 하단에 `AdminUserPanel` 을 배치하여 User UI 와 동일한 "계정 드롭다운은 사이드바 하단" UX 컨벤션을 준수하도록 정리했다. 기능 회귀 없음, 접근성(aria) 및 키보드 조작(ESC 닫기, focus-ring) 유지.

**판정: PASS.**
