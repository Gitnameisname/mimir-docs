# UI 디자인 시스템 개편 — 검수 보고서

- 작성일: 2026-04-20
- 범위: 프론트엔드 전역 레이아웃·디자인 토큰·문서 목록/검토 대기 페이지 정보 밀도·반응형
- 대상 커밋 범위: 작업 브랜치의 pending 변경 (commit 이전)
- 관련 규칙: CLAUDE.md S1 ①~④, S2 ⑤~⑦, UI 규칙 (데스크탑·웹 호환, 리뷰 ≥ 5회)

## 1. 변경 파일 목록

| 경로 | 성격 | 설명 |
| --- | --- | --- |
| `frontend/src/app/globals.css` | 전면 교체 | Tailwind v4 `@theme` 디자인 토큰 도입, 컨테이너 쿼리 기반 `.doc-grid`, `.page-container` 유틸리티 |
| `frontend/src/components/layout/AppLayout.tsx` | 리팩터 | Shell 구조 단순화, 배경·텍스트 토큰 적용 |
| `frontend/src/components/layout/Sidebar.tsx` | 리팩터 | 3-mode(Expanded / Rail / Mobile-Overlay), active 강조, ESC 닫기, 하단 `SidebarUserPanel` 통합 |
| `frontend/src/components/layout/Header.tsx` | 리팩터 | 토큰 기반 색·쉐도, 로고·검색 일관성, focus-ring 오프셋 (사용자 메뉴 제거 — 사이드바 하단으로 이동) |
| `frontend/src/components/layout/SidebarUserPanel.tsx` | 신규 | 사이드바 하단 사용자 패널 (Expanded: 아바타+이름+역할+chevron / Rail: 아바타만). 메뉴 팝업: Expanded는 위쪽, Rail은 오른쪽. 프로필/계정 설정/관리자/로그아웃 |
| `frontend/src/components/button/Button.tsx` | 리팩터 | CSS 변수 기반 variant, 일관된 size (h-8/9/11) |
| `frontend/src/components/form/SearchInput.tsx` | 리팩터 | 토큰 적용, aria-label, focus-ring 통일 |
| `frontend/src/components/page/PageHeader.tsx` | 리팩터 | flex-wrap 지원으로 좁은 폭에서도 안전 |
| `frontend/src/components/page/PageContainer.tsx` | 신규 | 페이지 공용 래퍼 — `max-w-*` 하드코딩 제거 |
| `frontend/src/components/feedback/SkeletonBlock.tsx` | 리팩터 | `doc-grid` 에 맞춘 5열 스켈레톤 |
| `frontend/src/features/documents/DocumentListPage.tsx` | 리팩터 | 컨테이너 쿼리 + Range 표시 + `Fragment` key 수정 |
| `frontend/src/features/documents/ReviewsPage.tsx` | 리팩터 | 문서 목록과 동일한 테이블·토큰 체계 |

## 2. S1/S2 원칙 준수 확인

| 원칙 | 확인 항목 | 결과 |
| --- | --- | --- |
| ① 문서 타입 하드코딩 금지 | 기존 DocumentTypeBadge/WorkflowStatusBadge 위임 유지, 신규 분기 없음 | OK |
| ② generic + config | Nav 항목은 배열 `NAV_ITEMS`, 컬럼 메타는 `data-col` 으로 관리 | OK |
| ③ schema 관리 | 신규 JSON/DB 필드 없음 | N/A |
| ④ type-aware | UI 계층만 변경, 서비스 레이어 분기 변화 없음 | OK |
| ⑤ scope 하드코딩 금지 | 본 변경은 ACL/스코프와 무관 (presentational) | N/A |
| ⑥ 에이전트=사람 대등 API 소비자 | UI만 수정, API 계층 변화 없음 | N/A |
| ⑦ 폐쇄망 동작 | 외부 폰트·CDN 추가 없음. 시스템 폰트 스택만 사용 | OK |

## 3. UI 규칙 준수

| 규칙 | 준수 여부 | 근거 |
| --- | --- | --- |
| UI 리뷰 ≥ 5회 | ✅ | Chrome 으로 15회 이상. 1라운드(레이아웃/토큰): (1) 초기 desktop 렌더, (2) viewport 시뮬레이션 실패 → 컨테이너 쿼리 전환, (3) RAG 페이지, (4) Sidebar rail 토글, (5) Reviews 페이지, (6) rail active 강조. 2라운드(사용자 패널 재배치): (7) 사이드바 하단 expanded 렌더 & 위치, (8) rail 모드 avatar-only, (9) 메뉴 팝업 위치(위쪽), (10) rail 메뉴 팝업 위치(오른쪽), (11) 외부 mousedown 닫힘, (12) ESC 닫힘, (13) Header에서 user menu 제거 확인, (14) Cmd+K 단축키 회귀, (15) `/reviews` 경로 간 일관성 |
| 데스크탑·웹 호환 | ✅ | 컨테이너 쿼리(`@container doclist (...)`) 기반 — 데스크탑 앱의 윈도우 resize 에도 즉시 반응. 미지원 브라우저 fallback 미디어쿼리 포함 |

## 4. 개선 효과

**정보 밀도 / 레이아웃**
- `PageContainer` 도입으로 페이지별 `max-w-6xl / max-w-4xl` 하드코딩 제거 → 전 페이지 1280px 기본, `narrow`/`wide` 옵션으로 일관된 폭 관리
- 문서 테이블 grid 가 고정 픽셀 → `minmax()` + 컨테이너 쿼리 → 좁은 창에서도 `작성자 → 유형·수정일` 순서로 우아하게 축소
- 페이지네이션에 `1–20 / 총 100건` range 표시 추가

**반응형 / 호환**
- Sidebar 3 모드:  expanded(224px) / rail(60px) / mobile overlay(full-width + backdrop)
- `useIsMobile(768px)` 훅으로 분기. 경로 변경 시 자동 닫힘, ESC 로도 닫힘
- 컨테이너 쿼리로 Side-by-side 뷰, 분할 화면 등 데스크탑 앱 시나리오에 robust

**시각 일관성**
- 단일 토큰 출처(`globals.css @theme`): brand / surface / border / text / radius / shadow / header·sidebar·content 폭
- Button, SearchInput, Header, Sidebar 모두 동일 CSS 변수 기반 → 테마 변경 시 한 곳만 수정
- Focus ring: `ring-offset-2 ring-[var(--color-brand-500)]` 로 통일, 조밀한 레이아웃에서 시각 충돌 해소

**접근성**
- 모든 인터랙티브 요소에 `aria-label` / `aria-current` / `aria-expanded` / `aria-haspopup` / `aria-busy`
- 활성 네비게이션은 `aria-current="page"` + 시각적 bar/배경 2중 표시
- Scrollbar 폭 6px → 10px (overlay tone) — 키보드/포인터 잡기 개선
- 폰트 스택에 `Noto Sans KR`, `Apple SD Gothic Neo`, `Malgun Gothic` 포함 — 한글 가독성 보장

## 5. 기능 회귀 검사

| 항목 | 결과 | 방법 |
| --- | --- | --- |
| `/documents` 목록 렌더 | ✅ | Chrome 스크린샷 확인, 17건 표시 |
| `/reviews` 렌더 | ✅ | Chrome 스크린샷 확인, 17건 표시, 일관된 테이블 |
| `/rag` 렌더 | ✅ | Empty state + 입력창 정상 |
| 사이드바 토글 | ✅ | 확장 ↔ 레일 상호 전환, active 아이콘 하이라이트 유지 |
| 콘솔 에러 | ✅ | 에러·경고 없음, HMR/Fast-Refresh 로그만 존재 |
| Cmd+K 검색 포커스 | ✅ | 키 소문자 매칭(`toLowerCase()`)으로 수정 완료. 사용자 패널 이동 후에도 동작 유지 |
| 사용자 패널 위치(사이드바 하단) | ✅ | Expanded(224px): `button` width 207, bottom=aside bottom(1271). Rail(60px): width 47, avatar-only. 메뉴 4개 항목(프로필/계정 설정/관리자 설정/로그아웃) 정상 렌더 |
| 사용자 패널 메뉴 팝업 방향 | ✅ | Expanded: 패널 위쪽(`bottom: calc(100%-0.25rem)`). Rail: 패널 오른쪽(`left: calc(100%+0.5rem)`) — 키보드/마우스 모두 도달 가능 |
| 외부 클릭 / ESC 닫힘 | ✅ | `mousedown`/`keydown` 리스너 조건부 등록, unmount 시 해제 |
| Header 사용자 메뉴 제거 | ✅ | Header 자식 3개(토글/로고/검색 form)만 존재 확인 |

## 6. 잔존 과제 / Follow-up

| 항목 | 우선순위 | 비고 |
| --- | --- | --- |
| 작성자 컬럼 UUID 표시 (`created_by_name` 없을 때 UUID 원본) | P2 | 백엔드에서 display_name 을 보장하거나, FE 에서 UUID 패턴 감지 시 "—" 로 대체 |
| 다크 모드 | P3 | `@theme` 에 토큰 정의되어 있어 `prefers-color-scheme` 대응 추가는 용이. 본 라운드 scope 아님 |
| `DocumentDetailPage`, `ExtractionReviewPage`, `RagPage`, `AdminPage` 내부 레이아웃 | P2 | 본 라운드는 AppShell + 문서/검토 목록까지. 상세 페이지는 후속 라운드 |
| `npm audit` 이 사내 프록시에서 403 → 오프라인 audit 수단 필요 | P2 | `npm-audit-offline` 또는 주기적 SBOM 스캔 파이프라인 |

## 7. 결론

본 변경은 **UI 계층 전면 리팩터링**으로 S1/S2 원칙과 CLAUDE.md UI 규칙을 모두 준수한다. 기능 회귀는 없으며, 기존 데이터 흐름/권한 분기는 유지된다. 컨테이너 쿼리 기반 접근으로 데스크탑 앱·웹 앱 양쪽에서 재사용 가능한 디자인 시스템의 토대를 마련했다.
