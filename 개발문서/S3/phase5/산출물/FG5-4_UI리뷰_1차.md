# FG 5-4 UI 디자인 리뷰 1차 — 4 viewport 실측

| 항목 | 값 |
|------|----|
| 작성일 | 2026-05-14 |
| 대상 | DocumentDetailPage 우측 사이드바 (DocumentSidebar) + AnnotationGutter + NodeRenderer mark 시각화 |
| 환경 | Chrome MCP 실측 — localhost:3050 / SUPER_ADMIN 계정 / 테스트 문서 `a01b6070-fbeb-4322-a090-bd3ded9eb8ce` |
| Phase 5 §4 산출물 규약 | UI 디자인 리뷰 ≥ 2회 — 본 보고서가 1차. 2차는 잔여 이슈 해소 후 |

---

## 1. viewport 별 결과

| viewport | 폭 | 사이드바 동작 | 본문 가독성 | 판정 |
|---------|----|----------|----------|----|
| desktop | 1440 | 우측 320px 고정 sticky | ✅ 충분 | ✅ PASS |
| narrow | 1100 | 본문 하단 stack (xl 임계 미만) | ✅ 충분 (fix 후) | ✅ PASS (fix 후) |
| tablet | 900 | 본문 하단 stack | ✅ 충분 | ✅ PASS |
| mobile | 480 | 본문 하단 stack | 🟡 "문서 구조" 트리 여전 — 본 패치 외 이슈 | 🟡 별 라운드 |

---

## 2. 본 라운드 발견 + 즉시 수정 (2026-05-14 fix)

### 2.1 ❌ AnnotationGutter 무한루프 — Maximum update depth exceeded

**증상**: DocumentDetailPage 진입 직후 Next.js console error
```
Maximum update depth exceeded.
src/features/documents/AnnotationGutter.tsx (73:7) @ useCallback[compute]
```

**원인**: `compute` 가 `useCallback([annotationsByNode, containerRef])` 로 메모이즈됐으나, useEffect deps 에 `compute` 가 들어 있어 매 render 마다 setDots 가 새 array 를 반환 → re-render → useCallback identity 변경 → useEffect 재실행 → 무한 루프.

**Fix**:
- `compute` 를 useEffect 내부 인라인 (useCallback 제거)
- `setDots` 호출에 얕은 비교 (`_dotsEqual`) 적용 — 동일하면 state 갱신 skip
- useEffect deps 를 `[annotationsByNode]` 로 단순화

### 2.2 ❌ narrow (1100px) 에서 본문 가독성 저하

**증상**: 좌측 메뉴 (224px) + 문서 구조 (224px) + 본문 + 우측 사이드바 (320px) 가 4컬럼 차지 → 본문 ~332px → 타이틀/작성자/액션 버튼 모두 wrap.

**원인**: `lg:flex-row` + `lg:w-80 lg:shrink-0 lg:sticky` (lg 임계 = 1024px) 가 너무 낮음. 1024–1279 narrow 구간에서 4컬럼이 모두 동시에 표시되어 본문이 좁아짐.

**Fix**: 임계를 `xl:` (1280+) 로 상향
- `DocumentDetailPage.tsx`: `lg:flex-row` → `xl:flex-row` / `lg:w-80 lg:shrink-0` → `xl:w-80 xl:shrink-0` / `lg:sticky lg:top-3` → `xl:sticky xl:top-3`
- `DocumentEditPage.tsx`: 동일 3 위치

**결과**: narrow 에서 좌측 메뉴 자동 collapse + DocumentSidebar 본문 하단 stack fallback — 본문 가독성 확보.

---

## 3. 본 라운드 확인된 PASS 항목

- ✅ **NodeRenderer mark 인식** — 본문 "`and #docjump now`" 의 `#docjump` 가 노란 `tag-pill` span 으로 시각화. Codex P1 #3 의 실제 동작 입증.
- ✅ DocumentSidebar 5 탭 정합 (주석/태그·배치/기여자/백링크/정보) + URL 동기화
- ✅ AnnotationsPanel collapsible 정상 동작 (헤더 클릭 → 펼침 + "해결된 항목 포함" 체크박스 + "+ 새 주석" 버튼)
- ✅ 좌측 ".prose pl-7" + AnnotationGutter absolute positioning (주석 없을 때 null 반환 정상)
- ✅ 4 viewport 별 layout breakdown 동작 (xl 이상 / lg 이하 분기)

---

## 4. 별 라운드 잔존 (본 패치 외)

| # | 이슈 | 분류 |
|---|------|----|
| 1 | mobile (480px) 에서 "문서 구조" 트리 여전 — 본문 폭 ~280px 로 타이틀 wrap | 별 이슈 (`DocumentTree` 의 모바일 collapse 별 라운드) |
| 2 | annotations API 응답 "주석을 불러올 수 없습니다 / 권한이 없거나 일시적인 오류" — SUPER_ADMIN 도 차단 | 본 패치 외 (Phase 3 기존 이슈) |
| 3 | AnnotationsPanel default-collapsed — annotations 탭 진입 후 다시 펼치기 필요 | UX 별 라운드 결정 |
| 4 | "AI 에이전트 제안" 섹션 사이드바 외부 잔존 | 회고 §4.3 #6 잔여 |
| 5 | 우하단 야자수 일러스트가 로그인 페이지 + 일부 화면에 노출 — 톤 일관성 검토 | 본 패치 외 |

---

## 5. 다음 (2차 UI 리뷰 입력)

1. 위 §4 잔여 #1, #3 결정 후 2차 리뷰
2. annotation 이 있는 문서로 mark 클릭 → panel highlight → AnnotationGutter 도트 클릭 → activeTab 자동 전환 흐름 Chrome 실측
3. EditPage 에서 본문 mark click → AnnotationsPanel 양방향 + @ typeahead Chrome 실측
4. 모바일 (480px) 에서 DocumentTree collapse 또는 hamburger 아래 토글 결정

본 1차 리뷰는 desktop/narrow/tablet 3 viewport PASS, mobile 은 본 패치 외 별 이슈로 합리적 분기. **§5.1 R-04 (좁은 화면에서 사이드바가 본문 가림 방지) 1차 확인**.
