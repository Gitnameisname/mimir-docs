# FG 3-1 UI 검수 로그

**작성일**: 2026-04-27
**대상**: ContributorsPanel (`frontend/src/features/documents/ContributorsPanel.tsx`)
**검수 기준**: CLAUDE.md 프로젝트 §5.2 — UI 리뷰 ≥ 1회 (Phase 1 완화 규정)

---

## 1. 검수 환경

- 검수자: Claude (자기 검수). 사람 최종 리뷰는 Codex / @최철균 (CONSTITUTION 제27조)
- 빌드 / 타입체크: `npm test` (tsc + node:test 533/533 PASS)
- 화면 조작 시뮬레이션: 코드 정독 + 컴포넌트 시그니처 검증
- 실 브라우저 회귀 시뮬레이션: 미수행 (별 라운드 — UI 자동화 또는 사람 운영자 점검)

---

## 2. 검수 항목

### 2.1 데스크톱 (≥ 1024px)

| 항목 | 결과 |
|-----|-----|
| 패널이 max-w-3xl 본문 안에 정상 마운트 | ✅ DocumentDetailPage L228-231 에 `<ContributorsPanel documentId={documentId} />` 추가 |
| collapsed 기본 상태 — "참여자" 헤더 + chevron 아이콘 | ✅ `useState(false)` + `aria-expanded` |
| 토글 클릭 → 펼침 + chevron 회전 | ✅ `transition-transform` + `rotate-180` 클래스 |
| 4 섹션 (작성자 / 편집자 / 승인자 / 열람자) 라벨 | ✅ `_Section` 컴포넌트 4 호출, 각 섹션에 항목 카운트 우측 표기 |
| 각 항목: 아바타 + 이름 + actor_type 뱃지 + role_badge + 상대 시간 | ✅ `_ContributorRow` 컴포넌트 |
| 기간 필터 드롭다운 (7일/30일/90일/전체) | ✅ `<select>` + SINCE_OPTIONS 4 종 |
| 열람자 표시 체크박스 | ✅ `<input type="checkbox">` 우측 정렬 (ml-auto) |
| 빈 상태 / 로딩 스켈레톤 / 에러 상태 분리 | ✅ `query.isLoading` → SkeletonBlock 6 줄 / `query.isError` → ErrorState + retry |

### 2.2 좁은 데스크톱 (≥ 1024px, 패널 너비 좁음 가정)

| 항목 | 결과 |
|-----|-----|
| 항목 행이 가로로 너무 길어지지 않는가 | ✅ `flex-wrap` 적용 (이름 / 뱃지 / role_badge 가 자동 줄바꿈) |
| 이름이 너무 길 때 truncate | ✅ `truncate` 클래스 (`min-w-0` 부모와 결합) |
| 기간 필터 / 열람자 토글이 좁은 폭에서 줄바꿈 | ✅ `flex-wrap` + `ml-auto` 가 좁아지면 자연스럽게 다음 줄로 |

### 2.3 태블릿 (768px ~ 1023px)

| 항목 | 결과 |
|-----|-----|
| 패널이 본문(max-w-3xl)에 그대로 들어감 | ✅ 본문이 좁아져도 max-w-3xl 안에 자연 fit |
| 토글 버튼 hit area 충분 (44px 가이드) | ⚠️ `py-3` (12px * 2 = 24px height) — touch target 권장 44px 미달.
                                          본 라운드 사이즈 보존 (Phase 2 의 다른 토글과 일관) |

### 2.4 모바일 (≤ 767px)

| 항목 | 결과 |
|-----|-----|
| 패널이 화면 폭 대비 100% 너비로 stretch | ✅ `border rounded-lg` + 부모 max-w-3xl 안 |
| 기간 필터 select 가 손가락으로 선택 가능 | ✅ 표준 `<select>` 사용, OS 네이티브 picker |
| 항목 행이 세로로 누적 가독성 OK | ✅ `flex items-center gap-3` + `flex-1 min-w-0` |
| 아바타 / 이름 / 뱃지 정렬 | ✅ 7px 아바타는 `flex-shrink-0` 으로 압축 안 됨 |

---

## 3. 접근성 (a11y) 점검

| 항목 | 결과 |
|-----|-----|
| 토글 버튼 `aria-expanded` / `aria-controls` | ✅ 적용 |
| 패널 본문 id (`contributors-panel-${documentId}`) | ✅ controls 와 일치 |
| 액터 뱃지 `aria-label="액터 유형: 사용자/에이전트/시스템"` | ✅ ActorTypeBadge 에서 적용 |
| 기간 필터 `aria-label="기간 필터"` | ✅ |
| 아바타 `aria-hidden="true"` (장식적) | ✅ |
| 열람자 체크박스 라벨 연결 | ✅ `<label>` 안에 `<input>` 둠 |

---

## 4. 다크모드

본 FG 는 **라이트 모드만** 적용. 다크모드는 R3 (FG-G4 도구 토큰 다크 변형) 의 admin 전면 다크 도입과 같은 라운드에서 후속.
- 토큰 사용 부분 (border, bg, text 색) 은 향후 `dark:` prefix 일괄 전환 가능하도록 cn 합성 유지.

---

## 5. 발견 / 결정

| # | 항목 | 결정 |
|---|-----|-----|
| 1 | 별도 우측 사이드바 vs 본문 안 카드 | 본문 안 collapsible 카드로 결정. 우측 사이드바는 Phase 3 후속 (DocumentDetailPage 전면 재구조화 비용 큼) |
| 2 | 열람자 체크박스 default | false (보수적). FG 3-2 정책 게이트가 결합되면 설정에 따라 강제 false 가능. UI 안내 메시지 ("조직 정책에 의해 가려짐") 포함 |
| 3 | 토글 버튼 touch target 24px (44px 미만) | 본 라운드 보존. Phase 2 의 다른 토글과 일관. 별 라운드 일괄 검토 |
| 4 | 빈 카테고리 메시지 톤 | 친화적 한국어 ("해당 기간 동안 …이 없습니다") |
| 5 | actor_type 뱃지 색상 | user=gray (강조 없음) / agent=indigo (정보) / system=amber (주의 톤) — 사람과 시스템을 명확히 구분 |

---

## 6. 회귀 영향

- DocumentDetailPage L228-231 추가만. 기존 마운트 컴포넌트 (`DocumentAssignControls`, `TagChipsEditor`, `VectorizationPanel` 등) 위치 / props 변경 없음
- `npm test` 533/533 PASS — 기존 테스트 회귀 0
- tsc 0 touched 오류

---

## 7. 사람 운영자 후속 권장

1. 실 브라우저에서 desktop / narrow / tablet / mobile 4 viewport 캡처
2. 큰 contributors (editors 30+ 명) 가 있는 문서에서 가독성 확인
3. agent / system actor 가 섞인 문서에서 뱃지 시각 검증

---

*작성: 2026-04-27 | task3-1 Step 6 산출물 (자기 검수 + 사람 후속 권장)*
