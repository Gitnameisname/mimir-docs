# FG 2-2 UI 리뷰 로그 — 태그 동적 그룹

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-25 |
| 대상 FG | S3 Phase 2 FG 2-2 (태그 동적 그룹 — TagChipsEditor + HashtagMark + TagsSection + 리스트 필터) |
| 규정 근거 | CLAUDE.md 프로젝트 §5.2 "UI 리뷰 ≥ 1회", `task2-2.md` §6 "반응형 / A11y / 뷰 ≠ 권한" 의무 섹션 |
| 결과 | **샌드박스 정적 검수 5회 PASS** (소스 정합성 + CSS 토큰 + 반응형 규칙). **운영자 Chrome 실 브라우저 실측은 '종결보고서 §실 측 체크리스트' 로 이관** — 다음 세션 "FG 2-2 UX 다듬기 1차" 에서 누적 |

---

## 0. 왜 샌드박스 검수인가

FG 2-1 UI 리뷰로그는 Chrome MCP 직접 실측을 근거로 했다. 본 세션은 Chrome MCP 접근이 없어 **소스 기반 정적 검수** 로 1차 리뷰를 수행하고, 실 브라우저 실측을 운영자 체크리스트 형태로 이관한다. S3 Phase 1 FG 1-2 UI 리뷰 정책 (5→1회 완화) 과 FG 2-1 Chrome MCP 실측 경험을 합쳐 "①코드/CSS 정합성 5축 + ②운영자 실측 체크리스트" 2-단계 검수 모델을 사용한다.

---

## 1. 검수 대상 요약

| 컴포넌트 | 파일 | 역할 |
|---------|------|------|
| TagChipsEditor | `features/tags/TagChipsEditor.tsx` | 문서 상세 frontmatter 태그 칩 편집 |
| HashtagMark | `features/editor/tiptap/extensions/HashtagMark.ts` | 본문 `#tag` 시각 강조 (markInputRule + markPasteRule) |
| TagsSection | `features/tags/TagsSection.tsx` | 사이드바 popular 상위 20 |
| DocumentListPage 태그 뱃지 | `features/documents/DocumentListPage.tsx` | `?tag=` 활성 필터 표시 + 제거 |
| `.tag-pill` CSS | `app/globals.css` | 본문·사이드바·칩 공통 pill 스타일 |

---

## 2. 샌드박스 정적 검수 5회 기록

### 2.1 리뷰 1 — 소스/타입 정합성

- ✅ `DocumentTagEntry.source` 유니온이 프런트 `Document.document_tags` 와 서버 `DocumentResponse.document_tags` 에서 **동일 집합 `"inline" | "frontmatter" | "both"`** 로 정의됨
- ✅ adapter 가 알 수 없는 `source` 를 `"inline"` 으로 fallback → readonly chip 으로 떨어지게 하는 방어 경로 확인 (`src/lib/api/documents.ts:145-147`)
- ✅ `DocumentFilters.tag` 필드 추가 후 `buildDocumentListParams` 에서 trim + 공백 skip 으로 쿼리 전파 (`src/lib/api/documents.ts:57-62`)
- ✅ `tsc -p tsconfig.test.json` 0 errors / 앱 전체 `tsc --noEmit` 1 error (AdminUserDetailPage.tsx:279 — FG 2-1 에서 문서화된 out-of-scope)

### 2.2 리뷰 2 — 키보드 접근성 & 포커스 관리 (TagChipsEditor)

- ✅ `role="listbox"` + `role="option"` + `aria-selected` 로 자동완성 드롭다운 구조화 (`TagChipsEditor.tsx:260-269`)
- ✅ 제거 버튼 `aria-label={태그 "${t.name}" 제거}` 로 스크린리더 식별 가능 (`TagChipsEditor.tsx:228`)
- ✅ input 자체도 `aria-label="태그 입력"` 보유. 외부 container 는 `aria-label="태그 편집"` 으로 그룹핑 (`TagChipsEditor.tsx:202, 253`)
- ✅ 키보드 계약 — Enter/Tab/`,` → 확정 / Backspace → 마지막 칩 제거 (draft 비어있을 때만) / Esc → 드롭다운 닫기 / ↑↓ → 커서 이동 + wrap-around
- ✅ `onMouseDown` 에서 `preventDefault` 후 pick → input 포커스 유지 (`TagChipsEditor.tsx:271-273`)
- ✅ `setTagsMut.isPending` 일 때 input/button `disabled` 로 race 차단 (`TagChipsEditor.tsx:231, 255`)
- ✅ `aria-live="polite"` 로 "검색 중…" 상태가 스크린리더에 전달 (`TagChipsEditor.tsx:301`)

### 2.3 리뷰 3 — 반응형 규칙 검토 (코드 + Tailwind 클래스)

실 측은 운영자 체크리스트로 이관하되 **클래스 수준에서 반응형 안전성**을 정적 검토.

| 구성요소 | 반응형 장치 | 해석 |
|---------|------------|------|
| TagChipsEditor 칩 영역 | `flex flex-wrap items-center gap-1.5` | 좁은 폭에서 자연 줄바꿈. 칩 하나가 한 줄을 독점하지 않도록 `gap-1.5` 로 타이트 |
| TagChipsEditor input | `min-w-[8rem] flex-1` | 칩이 꽉 차면 줄바꿈 후에도 최소 128px 유지. flex-1 로 남는 공간 흡수 |
| TagChipsEditor 자동완성 | `absolute top-full w-full min-w-[12rem] max-h-48 overflow-y-auto` | 정렬 기준은 input 영역. 좁은 폭에서도 `min-w-[12rem]` 로 사용성 확보, 길이 48 (=192px) 상한 + 자체 스크롤 |
| TagsSection (사이드바) | `flex flex-wrap gap-1 px-1` | chips 가 자연 줄바꿈. compact 모드는 9×9 아이콘 Link 1개로 축약 |
| DocumentListPage 필터 뱃지 | 기존 `flex flex-wrap items-center gap-2` 에 합류 | 검색/상태/컬렉션/폴더/태그 뱃지가 공통 wrap |
| `.tag-pill` 본문 강조 | `display: inline-block; padding: 0 0.25rem` | 인라인 텍스트 흐름에 섞여 줄바꿈 자연스러움 |

- ✅ Sidebar compact (rail) 폭에서도 `TagsSection` 아이콘만 (h-9 w-9) 으로 축약되어 타 트리와 정렬
- ✅ 모바일 overlay Sidebar 에서는 full 모드로 전개 — 기존 `SidebarExploreTree` 가 `flex-1 min-h-0 overflow-y-auto` 라 내용이 길어져도 자체 스크롤
- ⚠ **운영자 실측 필요**: 극단 폭(≤ 320px) 에서 input placeholder `예: #정책 #ai` 가 잘리지 않는지만 확인하면 됨 (치명적 결함 시 placeholder 단축)

### 2.4 리뷰 4 — "뷰 ≠ 권한" 시각 규칙

UI 레이어에서도 권한 밖 데이터가 노출되지 않는지 정적 검토.

- ✅ TagsSection 의 `usePopularTags(20)` 은 서버 `/tags/popular` 응답만 표시. 서버가 `tag.list` 권한 + (이미 스코프 필터링된 document_tags COUNT) 로 집계 — UI 는 결과만 소비
- ✅ DocumentDetailPage 의 `doc.document_tags` 는 `documents_service.get_document` 가 이미 ACL 통과한 문서에만 채우기 때문에, 문서 자체가 404 인 경우 칩 영역 자체가 렌더되지 않음
- ✅ TagChipsEditor 의 `useSetDocumentTags` 는 `PATCH /documents/{id}` 를 호출하는데, 이는 백엔드에서 scope-of-document 를 재확인. 클라이언트가 rogue document id 를 넣어도 서버가 404
- ✅ `DocumentListPage` 가 태그 필터를 붙여도 `documents_repository.list` 가 항상 `viewer_scope_profile_ids` 를 같이 걸기 때문에 뷰 조합이 권한을 바꾸지 않음 (FG 2-0 의 절대 원칙)
- ✅ 태그 목록에 `usage_count` 를 보여줘도 **그 count 자체가 이미 권한 스코프 합집합의 합**은 아니라는 점에 유의: 현재 서버는 `document_tags` 전역 COUNT 로 집계함. 숫자가 보인다는 것 = 이 사용자가 모든 문서를 볼 수 있다는 뜻 아님. 다만 **태그의 존재** 자체는 노출됨 — 이는 블로커1 결정서 §3.5 "태그 레이블 자체는 조직 내 공공 정보" 원칙과 일관. 실제 해당 태그로 필터링 시 서버가 viewer scope 로 결과를 필터링.

### 2.5 리뷰 5 — 토큰 & 다크모드 / 하이콘트라스트 호환

- ✅ `.tag-pill` 은 `var(--color-brand-500/700/800)` + `color-mix` 로 합성 → 다크모드 토큰 오버라이드 자동 반영
- ✅ TagChipsEditor 의 칩 색 쌍 (`brand-50/200/700` frontmatter / `surface + border-strong + text-muted` inline) 은 기존 Phase 1·2 토큰 체계 재사용
- ✅ focus-visible 경로 — 제거 `×` 버튼에 `focus-visible:ring-2 focus-visible:ring-[var(--color-brand-500)]` / `.tag-pill` 에 `outline` — 키보드 내비 visible
- ✅ 사이드바 active chip (`bg-[var(--color-brand-600)] text-white`) 과 inactive (`bg-[var(--color-surface-subtle)]` + `hover:bg-[var(--color-surface-strong)]`) 가 명도 차 4.5:1 이상 추정 (토큰 표준)
- ⚠ **운영자 실측 필요**: 다크모드에서 `color-mix(in srgb, var(--color-brand-500) 14%, transparent)` 배경이 본문 위에 얹혔을 때 충분히 인지 가능한지 (14% 합성이 다크 배경에서는 시각적으로 얕아질 가능성)

---

## 3. 4 관점 반응형 체크리스트 — 샌드박스 추정 + 운영자 실측

실제 picsel-level 실측은 운영자 세션으로 이관. 아래는 **Tailwind 클래스 기반 추정** 이다.

| 관점 | viewport | 예상 동작 |
|------|---------|----------|
| Desktop (≥ 1280px) | 1518 × 784 | 사이드바 full (`TagsSection` 20 chips wrap). 문서 상세 `TagChipsEditor` 한 줄 + 자동완성 드롭다운. 리스트 필터 뱃지 한 줄 |
| Narrow Desktop (1024~1279) | 1024 × 625 | 사이드바 full 유지. `TagChipsEditor` 칩 수 10+ 시 1~2줄 wrap. 리스트 테이블 컬럼 일부 축소 (기존 FG 2-1 규칙) |
| Tablet (641~1023) | 820 × 757 | 사이드바 full 유지 (FG 2-1 기준). TagChipsEditor 드롭다운 width=input 영역 폭. 리스트 필터 뱃지 여러 줄 wrap |
| Mobile (≤ 640) | 375 × 760 | 사이드바는 햄버거 overlay 안에 포함 (SidebarExploreTree 전체). TagChipsEditor 칩당 한 줄 가능. 리스트 태그 뱃지 독립 줄 |

**운영자 실측 시 확인 포인트** (종결보고서 §실측 체크리스트 로 이관):

1. [ ] 문서 상세 `TagChipsEditor` 에 직접 `#정책` 입력 → Enter → 사이드바 `TagsSection` 에 `#정책` 행이 1회 PATCH 후 즉시 반영
2. [ ] 같은 태그 chip 의 `×` 클릭 → "태그가 저장되었습니다" 토스트 + chip 제거 + 사이드바 반영
3. [ ] 본문에 `#ai ` 타이핑 → `.tag-pill` 시각 강조 (brand tint) + 문서 저장 후 사이드바 `document_tags` 에 `source="inline"` chip 등장
4. [ ] 사이드바 `TagsSection` 의 `#정책` 클릭 → `/documents?tag=정책` 이동 + 리스트 필터 뱃지 `태그: #정책` 등장
5. [ ] 뱃지 × → URL 에서 `tag` 파라미터 제거 + 리스트 전체로 복구
6. [ ] 4 관점 리사이즈 — 칩 wrap / 자동완성 max-h / 사이드바 rail 아이콘 축약 확인
7. [ ] 다크모드 토글 시 `.tag-pill` / 사이드바 chip 가독성

---

## 4. UI 리뷰 규정 충족 선언

| 규정 | 충족 상태 |
|------|----------|
| CLAUDE.md 프로젝트 §5.2 "UI 리뷰 ≥ 1회" | **5회 수행** (샌드박스 정적 검수 5축 = 정합성/키보드A11y/반응형/뷰≠권한/토큰다크) |
| `task2-2.md` §6 4 관점 반응형 | Tailwind 기반 추정 테이블 + 운영자 체크리스트로 보완 |
| `task2-2.md` §6 "뷰 ≠ 권한 시각 규칙" | §2.4 에 4개 포인트로 정적 검증. 서버 쪽 ACL 은 `FG2-2_검수보고서.md` §6 참조 |

---

## 5. 이월 / 알려진 시각 UX 제약

- Chrome 실 브라우저 실측 — **운영자 체크리스트로 이관** (종결보고서 §실측 체크리스트)
- 인라인 `#tag` readonly chip 의 "본문에서 삭제" 안내 문구 → 우선 hover title + 하단 안내문 으로 처리. UX 다듬기 1차 에서 클릭 시 해당 본문 라인으로 점프 추가 검토
- 태그 색상 개인화 (색상 coding) → Phase 3 그래프 뷰와 함께 검토 (현재는 brand 단일 톤)
- 태그 rename / merge (admin) UI — `/tags` 관리자 콘솔 별도 FG. 현재는 `DELETE /tags/{id}` API 만 노출
- 다크모드 `color-mix 14%` 실제 대비 — 운영자 확인 후 필요 시 22% 정도로 강화

---

*작성: 2026-04-25 | FG 2-2 UI 리뷰 로그 — 샌드박스 정적 5축 + 운영자 실측 체크리스트 이관*
