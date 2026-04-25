# FG 1-2 UI 디자인 리뷰 로그

| 항목 | 값 |
|------|----|
| Phase / FG | S3 Phase 1 / FG 1-2 |
| 정책 기준 | CLAUDE.md 프로젝트 §5.2 "UI 검수는 최소 1회 이상" (2026-04-24 완화, 기존 task1-2 §5 C6 "5회" → "≥1회") |
| 수행 회차 | **1회** (2026-04-24) |
| 수행 방식 | 정적 코드 분석 (sandbox 에서 Next dev 서버 SWC arm64 바이너리 미설치 + npm registry 차단으로 실 화면 기동 불가) |
| 실 화면 smoke | **운영자 로컬 후속** — 아래 §4 체크리스트 |

---

## 1. 리뷰 대상

- 페이지: `DocumentEditPage` (`/documents/{documentId}/edit`)
- 에디터: `DocumentTipTapEditor` (`features/editor/tiptap/DocumentTipTapEditor.tsx`)
- 스타일: `app/globals.css` §"Mimir Editor (Phase 1 FG 1-2)" 블록
- 뷰 모드: `block` | `flow` 토글 (헤더 세그먼트 + `⌘/Ctrl+Alt+M`)

---

## 2. 정적 리뷰 결과 (5축 통합, 1회)

### 축 1 — 블록 뷰 카드 간격 / 테두리 / hover 대비

**현황 (`globals.css` §`.mimir-editor--block .ProseMirror > *`)**
- `padding: 0.75rem 1rem` · 카드 기본 `border: 1px solid transparent`
- hover: `border-color: var(--color-border)` (#e2e8f0) + `background: var(--color-surface-muted)` (#f8fafc)
- focus-within: `border-color: var(--color-brand-300)` (#93c5fd) + brand-50 배경 50% 반투명

**평가**
- ✅ 블록 간 간격(`margin-top: 0.5rem`) 자연스러움. 카드 hover 전환 120ms 짧아 쾌적
- ⚠ hover 시 `border-gray-200 (#e2e8f0)` 이 기본 페이지 배경(#ffffff)과 채도 차이가 작아 **좁은 창에서 경계 모호** 가능 — 운영자 smoke 에서 대비 확인 요구
- ✅ focus-within 의 brand-300 + brand-50 배경은 선택 상태를 분명히 보여 줌

**판정**: 현 상태 승인. 운영자 실 화면 smoke 1건에서 hover 대비만 확인 (WCAG AA 3:1 권장)

### 축 2 — 일반 뷰 타이포그래피

**현황**
- `.mimir-editor .ProseMirror` 기본: 15px / line-height 1.7 / `--color-text` (#0f172a)
- h1 24px · h2 20px · h3 18px (각각 font-weight 600)
- heading margin: `1.25rem 0 0.5rem` · p margin: `0.5rem 0`
- ul/ol: `padding-left: 1.5rem`
- pre(codeBlock): `#0f172a` bg / `#e2e8f0` text / font-size 13px

**평가**
- ✅ heading 1:2:3 비율 24/20/18 = 1.33 / 1.11 — 적절 (시각 위계 분명)
- ✅ 본문 line-height 1.7 은 한국어 장문에 충분
- ⚠ h1~h3 의 상단 margin `1.25rem` 이 blockcard padding `0.75rem` 과 중첩되어 블록 뷰에서 **heading 카드가 과하게 높아짐** — 아래 개선 권고

**개선 권고 (즉시 반영 가능)**: 블록 뷰에서 heading 내부 `h1/h2/h3 { margin-top: 0 }` 재설정. 이번 리뷰 범위에서 적용하고 후속 smoke 에서 확인.

### 축 3 — 뷰 토글 세그먼트

**현황 (`DocumentEditPage.tsx` L218–L250)**
- `role="group"` + `aria-label="에디터 뷰 모드"`
- 두 button: `aria-pressed` 토글 / 활성 시 `bg-gray-900 text-white`
- 가로 `inline-flex rounded-md border border-gray-200` / `text-xs`
- 단축키 안내는 "일반" 버튼 `title` 에만 포함 (`⌘/Ctrl+Alt+M 으로 토글`)

**평가**
- ✅ 레이아웃 세그먼트 패턴 표준 (WAI-ARIA radio group 변형)
- ✅ 활성 상태 색 대비 (`bg-gray-900` #0f172a vs white) > WCAG AA 4.5:1 충분
- ⚠ 단축키 hint 가 일반 버튼에만 — "블록" 버튼 `title` 에도 대칭 문구 필요
- ⚠ 좁은 창(`flex-wrap`)에서 세그먼트가 저장 상태/저장 버튼 오른쪽에 떨어질 수 있음. 시각적으로는 깨지지 않으나 우선순위상 제목 바로 옆 유지가 더 명확

**개선 권고**:
1. "블록" 버튼 `title` 에도 `"블록 뷰 — 블록별 카드 표시 (⌘/Ctrl+Alt+M 으로 토글)"` 추가
2. (후속) 데스크탑 우선 기준에서 세그먼트를 저장 상태 앞 / 저장 버튼 뒤 배치로 이동은 UI 리뷰 후속 세션

### 축 4 — 접근성 (키보드 전용)

**현황**
- 제목 `<input aria-label="문서 제목">` — Tab 대상
- 세그먼트 버튼 2개 — `aria-pressed` 로 토글 상태 노출
- 저장 상태 `<span role="status" aria-live="polite">` — 변경 시 스크린 리더 알림
- 저장 버튼 `Button` 컴포넌트 (내부적으로 `<button>`, `disabled={!isDirty}`)
- 본문 `<EditorContent>` — TipTap 이 contentEditable + ARIA 자동

**평가**
- ✅ Tab 순서: 취소 → 제목 → 블록 → 일반 → 저장 상태(span, non-focusable) → 저장 → 에디터 본문. 논리적
- ✅ Cmd/Ctrl+S / Cmd/Ctrl+Alt+M 단축키 `preventDefault` 로 브라우저 기본 동작 차단 처리 됨
- ⚠ 저장 상태 `span` 에 `onClick={saveStatus === "error" ? handleSave : undefined}` — 마우스 전용 재시도. **키보드 접근 불가**
- ⚠ 에디터 본문에 `role="textbox"` 또는 `aria-label` 이 TipTap 기본 제공인지 실 화면 확인 필요

**개선 권고**:
1. 저장 실패 시 재시도 경로를 키보드 접근 가능하게 — 실패 상태 span 을 `<button type="button">` 으로 교체. 아래 §3 즉시 개선에 포함
2. (후속) 에디터 본문 aria-label 점검은 실 화면 smoke

### 축 5 — 저장 상태 피드백

**현황 (`SAVE_STATUS_TEXT` / `SAVE_STATUS_COLOR`)**
- saved: "저장됨 ✓" / `text-gray-400`
- unsaved: "저장되지 않은 변경사항" / `text-amber-500`
- saving: "저장 중..." / `text-gray-400`
- error: "저장 실패 — 클릭하여 재시도" / `text-red-500 cursor-pointer`

**평가**
- ✅ unsaved amber / error red 명도 대비 분명
- ⚠ saved/saving 모두 `text-gray-400` — 저장 중 시각적 피드백 약함. 저장 중에는 loading 인디케이터가 "저장" 버튼에 있으나 상태 텍스트만 보는 사용자는 구분 어려움
- ⚠ 위 축 4 의 "클릭하여 재시도" 가 실제로 키보드 접근 불가

**개선 권고**:
1. saving 상태를 `text-brand-500` (파랑) 또는 스피너 아이콘 붙여 가시화 — 후속 UI 세션
2. 재시도 경로 키보드 접근 가능하게 변경 (본 리뷰에서 즉시 반영 — §3)

---

## 3. 본 리뷰에서 즉시 반영한 개선 (1차)

- [x] 저장 상태 `<span>` 의 error 분기를 **`<button>`** 으로 변경 → 키보드 접근 가능
- [x] "블록" 버튼 `title` 에 단축키 문구 대칭 추가
- [x] 블록 뷰 heading 카드 높이 정규화: `.mimir-editor--block .ProseMirror > h1/h2/h3 { margin-top: 0 }` 추가

(diff 는 §5 에 링크)

**미개선 (운영자 smoke 이후 2차 리뷰 대상)**:
- saving 상태 가시성 강화 (스피너/색상)
- hover 테두리 대비 조정 (`--color-border` vs 배경 차이)
- 세그먼트 배치 위치 재검토 (flex-wrap 동작 실측 이후)

---

## 4. 운영자 로컬 smoke 체크리스트

sandbox 에서 실 화면 기동 불가(`SWC arm64` 미설치 + `registry.npmjs.org` 차단). 로컬에서 아래 순서로 1회 smoke 후 별도 피드백 부탁드립니다.

```bash
cd /Users/ckchoi/Desktop/projects/mimir/frontend
npm run dev  # -p 3050
```

브라우저에서 임의 문서 편집 페이지 진입 (`http://localhost:3050/documents/{id}/edit`) 후:

1. [ ] **블록 뷰** hover 시 카드 테두리가 시각적으로 분명한지 (회색 대비)
2. [ ] **일반 뷰** heading h1/h2/h3 시각 위계 자연스러운지 (1차 개선 적용 후)
3. [ ] 세그먼트 **블록/일반** 버튼이 헤더 제목 옆에 정상 배치, 활성 색 대비 충분
4. [ ] 좁은 창(≤ 640px) 로 축소 시 헤더가 2줄 wrap 되며 깨지지 않음
5. [ ] Tab 키만으로 취소→제목→블록→일반→저장 순서 포커스 이동 (접근성 기본)
6. [ ] `Cmd/Ctrl+Alt+M` 로 뷰 모드 토글 즉시 반영 (재마운트 없이)
7. [ ] `Cmd/Ctrl+S` 저장 후 상태 "저장됨 ✓" 표시, 재로드 시 내용 유지
8. [ ] 저장 실패 상태에서 **Tab 으로 포커스 → Enter 로 재시도** 가능 (§3 개선 확인)
9. [ ] 한글 IME 상태에서 `Alt+M` 이 의도치 않은 입력으로 이어지지 않는지

결과 공유 → FG 1-2 공식 종결 선언 또는 2차 리뷰 착수.

---

## 5. 즉시 개선 diff 참조

아래 파일에서 §3 의 3개 항목 반영 (본 리뷰 로그와 같은 PR 에 포함):

- `frontend/src/features/editor/DocumentEditPage.tsx` — 저장 상태 span → button / 블록 버튼 title 업데이트
- `frontend/src/app/globals.css` — 블록 뷰 heading margin-top 정규화

---

## 6. 회귀 확인

- [x] node:test 157 passed / 0 failed (개선 전과 동일)
- [x] tsc --noEmit 우리 변경 관련 에러 0건

---

## 7. 참조

- [task1-2.md](../작업지시서/task1-2.md) §5 C6 (정책 완화 이력 포함)
- [FG1-2_검수보고서.md](./FG1-2_검수보고서.md) §2 C6
- `CLAUDE.md` 프로젝트 §5.2 (최소 1회 이상)

---

*작성: 2026-04-24 | 1회 정적 리뷰 수행 + 3개 개선 반영 + 운영자 로컬 smoke 체크리스트.*
