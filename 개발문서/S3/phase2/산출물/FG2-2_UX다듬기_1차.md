# FG 2-2 UX 다듬기 1차 — 다크모드 + inline chip 본문 점프

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-25 |
| 전단계 | FG 2-2 공식 종결 (2026-04-25) — `FG2-2_종결보고서.md` §7 |
| 범위 | ① 다크모드 토큰·토글·SSR flash 방지 인프라 / ② inline readonly chip → 본문 점프 |
| 이월 근거 | `FG2-2_종결보고서.md` §4 "다음 라운드" |

---

## 0. 결정 정리

- **다크모드**: 전면 개편이 아니라 **토큰 도입 + 토글 UI + preferences 연동** 까지. 개별 페이지 다크 톤 세부 조정(예: admin UI 배경)은 후속 라운드에서 점진 개선.
- **Inline chip 점프**: 읽기 모드 본문 렌더러(NodeRenderer)가 HashtagMark span 을 직접 그리지 않으므로 **편집 모드로 이동 + `?focus_tag=<name>` 쿼리** 방식으로 설계. TipTap 이 렌더한 `span.tag-pill[data-tag]` 를 스크롤 + flash.

---

## 1. 변경 파일

### 1.1 백엔드
- `backend/app/api/v1/account_router.py`
  - `UserPreferences` / `UpdatePreferencesRequest` 에 `theme: Literal["system","light","dark"]` 추가 (extra=allow 유지)

### 1.2 프런트엔드 — 다크모드 인프라
- `frontend/src/app/globals.css`
  - `:root` 에 `--color-tag-flash` 등 추가 토큰
  - `:root[data-theme="dark"]` 오버라이드 블록 (surface/text/border/brand/shadow)
  - `@media (prefers-color-scheme: dark)` + `:root:not([data-theme="light"]):not([data-theme="dark"])` 시스템 추종
  - `.tag-pill` 대비 22% 로 상향 + `:hover` 32%
  - `@keyframes tag-pill-flash` + `.tag-pill--flash` (클릭 점프 시 강조)
- `frontend/src/lib/api/account.ts`
  - `ThemePreference = "system" | "light" | "dark"`
  - `UserPreferences.theme` + `UpdatePreferencesRequest.theme` 필드
- `frontend/src/hooks/useTheme.ts` (신규)
  - `preference` (저장된 값) / `effective` (실제 적용) 이분법
  - `prefers-color-scheme` MediaQuery 구독 (시스템 변경 감지)
  - `setPreference` → optimistic (html.dataset.theme 즉시) + PATCH
- `frontend/src/components/theme/ThemeApplier.tsx` (신규)
  - mount 시 `documentElement.dataset.theme = preference` + localStorage 저장
  - `themePreloadSnippet` export — layout `<head>` inline script 로 SSR flash 방지
- `frontend/src/components/theme/ThemeToggle.tsx` (신규)
  - 3-state 세그먼티드 컨트롤 (radiogroup + aria-checked)
  - compact 모드 원버튼 순환 (system→light→dark→system)
- `frontend/src/app/layout.tsx`
  - `<head>` 에 `<script dangerouslySetInnerHTML={{ __html: themePreloadSnippet }} />`
  - `<Providers>` 내에 `<ThemeApplier />` 편입
- `frontend/src/components/layout/SidebarUserPanel.tsx`
  - 사용자 드롭다운 메뉴 안 "테마" 섹션 + `<ThemeToggle />`

### 1.3 프런트엔드 — inline chip 본문 점프
- `frontend/src/features/tags/scrollToInlineTag.ts` (신규)
  - `span.tag-pill[data-tag="<name>"]` 첫 매치 → `scrollIntoView` + `tag-pill--flash` 1.3s
  - `CSS.escape` 로 selector 주입 방지
  - 매치 없음 시 `false` 반환 (degrade 경로를 상위가 결정)
- `frontend/src/features/tags/TagChipsEditor.tsx`
  - `onJumpToInlineTag?: (name: string) => void` prop 추가
  - `source === "inline"` chip 을 `<button>` 으로 렌더 (점프 핸들러 제공 시)
  - aria-label "본문 내 #<name> 위치로 이동"
- `frontend/src/features/documents/DocumentDetailPage.tsx`
  - `onJumpToInlineTag` → `router.push("/documents/<id>/edit?focus_tag=<name>")`
- `frontend/src/features/editor/DocumentEditPage.tsx`
  - `useSearchParams` 추가
  - `initializedRef.current` true 가 되면 (=TipTap 이 snapshot 로드 완료) `focus_tag` 쿼리 → 150ms 후 `scrollToInlineTag`
  - 실패 시 `"본문에 #<name> 을 찾지 못했습니다. frontmatter 태그만 있을 수 있습니다."` info toast

---

## 2. 테스트 (node:test)

누적 **269 passed / 53 suites / 0 failed** (UX1 신규 10).

| 파일 | 건수 | 범위 |
|------|------|------|
| `tests/ScrollToInlineTagUx1.test.tsx` | 5 | selector 포맷 / 빈 이름 skip / scrollIntoView + flash / CSS.escape / nested(ml/nlp) |
| `tests/ThemePreloadUx1.test.tsx` | 5 | preloadSnippet 이 dark / light / system / 없음 / 비인식 값 각 케이스에서 올바른 data-theme 만 세팅 |

### tsc
- `npx tsc -p tsconfig.test.json` → **0 errors**
- `npx tsc --noEmit` (앱 전체) → **1 error** (`AdminUserDetailPage.tsx:279` 기존 out-of-scope)

---

## 3. Chrome 실측 요약 (2026-04-25)

### 3.1 다크모드 토글
- 사용자 메뉴 → 테마 섹션 → "다크" 클릭
- `data-theme="dark"` 즉시 적용 / body bg `rgb(17,24,39)` / text `rgb(226,232,240)` 확인 ✅
- "라이트" / "시스템" 도 정상 순환 ✅
- `useUserPreferences` 를 통해 서버 PATCH 까지 반영

### 3.2 Inline chip 본문 점프
- 편집 모드에서 본문에 `and #docjump now` 타이핑 → 저장 → 문서 상세 복귀
- TagChipsEditor 에 `#docjump` 가 **readonly inline chip(회색)** 로 등장 ✅
- 하단 안내문 "회색 칩은 본문 #태그 에서 자동 감지된 태그입니다" ✅
- chip 클릭 → URL 이 `/documents/<id>/edit?focus_tag=docjump` 로 정상 전이 ✅ (find 결과에서 button 역할과 aria-label "본문 내 #docjump 위치로 이동" 확인)
- 에디터 마운트 후 scrollToInlineTag 실행 → 매치 시 flash

### 3.3 Hydration mismatch 버그 → 수정 (2026-04-25)

**증상**: 초기 배포 후 콘솔에 `"A tree hydrated but some attributes of the server rendered HTML didn't match the client properties"` + diff `-data-theme="system"` — SSR HTML 은 `<html>` 에 data-theme 없이 나오는데 클라이언트 preload snippet 이 "system" 을 setAttribute 해버려서 속성값이 달라짐.

**원인**: `:root:not([data-theme="light"]):not([data-theme="dark"])` CSS 조건이 이미 system = "속성 없음 == prefers 추종" 으로 설계됨. preload 가 "system" 을 명시 설정할 필요가 없고, 오히려 설정하면 서버 HTML 과 달라져 hydration 경고.

**수정**:
- `themePreloadSnippet` 이 "light" / "dark" 만 `setAttribute`, "system" 은 **no-op**
- `ThemeApplier` / `useTheme.setPreference` 도 "system" 선택 시 `delete dataset.theme` 로 속성 제거
- `app/layout.tsx` 의 `<html>` 에 `suppressHydrationWarning` 추가 (Next 권장 패턴, 초기 paint 전 inline script 가 속성을 만졌을 가능성을 허용)
- `ThemePreloadUx1.test.tsx` 의 system 케이스를 "속성 미설정" 로 업데이트

**Chrome 재검증**:
- `localStorage='system'` + reload → `data-theme=null` + 콘솔 에러 **0** ✅
- `ThemeToggle → 다크` 후 reload → preload 가 `data-theme="dark"` 를 pre-paint 적용 + bg `rgb(17,24,39)` 유지 + 콘솔 에러 **0** ✅
- `ThemeToggle → 시스템` 으로 복귀 → `data-theme=null` + 라이트 복원 ✅

### 3.4 범위 밖 관찰 (별건)
- 편집 페이지 reload 시 draft snapshot 초기 로드 race 이슈 (본문이 순간 빈 상태) — 본 라운드 범위 밖. 기존 `/documents/{id}/edit` 에 대한 별건 (Phase2 인수문서 §6 의 잔존 이슈 연장)
- 화면 우하단 "1 Issue" 배지 — dev 환경의 React Query DevTools 또는 sentry 연동 배지로 추정. UX1 범위 밖.

---

## 4. "뷰 ≠ 권한" 재확인

- `PATCH /account/preferences` 는 자기 자신 user_id 에만 적용 (기존 `_normalize_actor_to_user_id` 경유, S2 ⑤)
- preferences.theme 은 순수 UI 취향 — 서버 응답 직렬화에서 다른 사용자 preferences 노출 경로 없음
- 다크모드 토큰 변경은 viewport 수준 CSS 오버라이드이며 API 응답 필터에 무관

---

## 5. 이월 / 알려진 제약

| 항목 | 상태 |
|------|------|
| admin/테이블/특정 페이지의 다크 스타일 세부 조정 | 후속 라운드 (토큰 도입 완료 기반) |
| 편집 페이지 draft snapshot 초기 로드 race | 기존 별건 (Phase2 인수문서 §6 연장) |
| 읽기 모드 본문에서도 `#tag` 를 pill 로 렌더 (NodeRenderer 확장) | FG 2-3 백링크와 함께 검토 (`[[문서명]]` 렌더와 규약 통일) |
| 시스템 prefers 변경 애니메이션 | 즉시 스위치 — 전환 애니메이션은 UX 개선 후보 |
| 다크모드 `.tag-pill` 대비 추가 튜닝 | 실사용 피드백 후 필요 시 |

---

## 6. 산출물 위치

- 본 문서: `docs/개발문서/S3/phase2/산출물/FG2-2_UX다듬기_1차.md`
- 백엔드: `backend/app/api/v1/account_router.py`
- 프런트 신규: `src/hooks/useTheme.ts`, `src/components/theme/ThemeApplier.tsx`, `src/components/theme/ThemeToggle.tsx`, `src/features/tags/scrollToInlineTag.ts`
- 프런트 확장: `app/layout.tsx`, `app/globals.css`, `components/layout/SidebarUserPanel.tsx`, `lib/api/account.ts`, `features/tags/TagChipsEditor.tsx`, `features/documents/DocumentDetailPage.tsx`, `features/editor/DocumentEditPage.tsx`
- 테스트: `frontend/tests/ScrollToInlineTagUx1.test.tsx`, `frontend/tests/ThemePreloadUx1.test.tsx`

---

*작성: 2026-04-25 | FG 2-2 UX 다듬기 1차 — 다크모드 인프라 + inline chip 점프*
