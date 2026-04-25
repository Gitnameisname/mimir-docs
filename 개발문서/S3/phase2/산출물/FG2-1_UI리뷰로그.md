# FG 2-1 UI 리뷰 로그

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-24 |
| 대상 FG | S3 Phase 2 FG 2-1 (수동 컬렉션 + 계층 폴더) + 이후 UX 1~5차 다듬기 누적 |
| 규정 근거 | CLAUDE.md 프로젝트 §5.2 **UI 리뷰 ≥ 1회**, task2-1.md §4 Step 6 반응형 검수, §4 Step 7 "뷰 ≠ 권한 준수 확인" 섹션 의무 |
| 결과 | **전 라운드 Chrome 실제 브라우저 실측 PASS**, 4 관점 반응형 (desktop / narrow desktop / tablet / mobile) 확인 완료 |

---

## 1. 실측 환경

- 브라우저: Chrome 안의 실 Mimir dev 서버 (`http://localhost:3050`)
- 계정: SUPER_ADMIN (`관리자`) — 검수용 bypass role
- DB: 로컬 PostgreSQL, Alembic head=`s3_p2_collections_and_folders` (FG 2-0/2-1 마이그레이션 적용 완료)
- 리뷰 방식: Chrome MCP 로 직접 클릭·타입·스크린샷. 수동 인터랙션이 아닌 **자동화 실측**

## 2. 라운드별 Chrome 실측 결과 (누적)

### 2.1 FG 2-1 본편 (최초 프런트 이식)
검증 기준: 사이드바에 컬렉션/폴더 섹션이 떠 있고, 클릭 시 `/documents?collection=` / `?folder=` 로 이동하며 필터 뱃지가 표시되는가.

- ✅ Sidebar 상단 NAV (문서/검토 대기/추출 검토/AI 질의) 유지
- ✅ 중간에 "컬렉션" / "폴더" 섹션 신규 삽입
- ✅ 컬렉션 선택 → URL query + 필터 뱃지 + 빈 CTA (당시 기본 "+ 새 문서 만들기")
- 결론: 기본 UI 동작 PASS. 단 **편집/삭제/맥락 CTA/현재 배치 상태** 가 없어 UX 후속 라운드에서 보완.

### 2.2 UX 1차 — 편집·삭제 메뉴 (2026-04-24)

[산출물: `FG2-1_UX다듬기.md`]

- ✅ 컬렉션/폴더 hover 시 `···` 메뉴 + `0` 회색 뱃지 노출
- ✅ 컬렉션 메뉴 = **이름 변경 / 삭제**
- ✅ 폴더 메뉴 = **이름 변경 / 이동 / 삭제** (3 항목, 삭제는 danger 빨강)
- ✅ 인라인 rename (기존 이름 선택 상태, Enter/Esc/Blur)
- ✅ `ConfirmDialog` 삭제 모달
- ✅ `FolderMoveDialog` 새 부모 드롭다운 (자기+하위 제외)
- 결론: Chrome 실측 5/5 PASS

### 2.3 UX 2차 — 현재 폴더/컬렉션 노출 + 빈 컬렉션 CTA (2026-04-24)

[산출물: `FG2-1_UX다듬기_2차.md`]

- ✅ 빈 컬렉션 진입 시 `"<컬렉션명>" 컬렉션이 비어 있습니다` 전용 CTA + `[기존 문서 추가]` / `[+ 새 문서 만들기]`
- ✅ `AddDocumentsToCollectionModal` 최근 50건 + 체크박스 + 추가
- ✅ 문서 상세 상단에 `DocumentAssignControls` — 현재 포함 컬렉션이 `📋 <이름> ✕` chip 으로 표시
- ✅ chip X 클릭 시 즉시 제거 + **사이드바 뱃지 카운트 동시 감소** (이중 invalidate)
- 결론: Chrome 실측 5/5 PASS

### 2.4 UX 3차 — q 서버 지원 + NewDocumentPage 자동 연결 (2026-04-24)

[산출물: `FG2-1_UX다듬기_3차.md`]

- ✅ 모달 서버 `q` 검색 (debounce 300ms, `isFetching` "검색 중..." 표시)
- ✅ NewDocumentPage `?collection=<id>` 수신 시 상단 배너 `📋 생성 후 "<이름>" 컬렉션에 자동으로 추가됩니다.`
- ✅ 문서 생성 직후 자동 연결 → 사이드바 뱃지 즉시 갱신 / 컬렉션 필터 URL 에서 목록에 신규 문서 노출
- 결론: Chrome 실측 5/5 PASS (edit 페이지 렌더 에러는 기존 별건으로 범위 밖)

### 2.5 UX 4차 — 실시간 debounce + "본문까지 전체 검색 →" 링크 (2026-04-24)

[산출물: `FG2-1_UX다듬기_4차.md`]

- ✅ `/documents` SearchInput 입력 시 Enter 없이 300ms 후 자동 필터
- ✅ placeholder 를 실제 동작에 맞게 **"제목 검색…"** 로 정돈
- ✅ 옆 🔎 **"본문까지 전체 검색 →"** 링크로 `/search?q=<현재>` 자연 전환 + FTS 하이라이트
- 결론: Chrome 실측 3/3 PASS

### 2.6 UX 5차 — `/search` ACL/필터 포팅 + `?q=` URL 동기화 (2026-04-24)

[산출물: `FG2-1_UX다듬기_5차.md`]

- ✅ 입력 후 URL 이 `/documents?q=Audit` 로 자동 `router.replace`
- ✅ 직접 `/documents?q=Audit&collection=<id>` 진입 시 프리필 + 필터 뱃지 2개 + AND 결과
- ✅ 뒤로가기로 `/search?q=Audit` 복귀 정상
- ✅ `/search?q=Audit` FTS 결과 `**Audit** Test` 하이라이트 (scope wiring 통과 확인)
- 결론: Chrome 실측 4/4 PASS

---

## 3. 4 관점 반응형 체크리스트 (2026-04-24 추가 실측)

`/documents` 를 기본 페이지로 잡고 Chrome MCP 로 해상도를 리사이즈 후 실측.

| 관점 | viewport (innerWidth × innerHeight) | 결과 |
|------|-------------------------------------|------|
| **Desktop** | 1518 × 784 (기본) | 탐색 트리 확장 상태, 필터 뱃지 flex-wrap 줄바꿈, 모든 UI 요소 여유 배치. 5 라운드 내내 확인 ✓ |
| **Narrow Desktop** | 1024 × 625 | 탐색 트리 확장 상태 유지, 문서 테이블 컬럼이 자동 축소. 사이드바·필터 UI 모두 정상 ✓ |
| **Tablet** | 820 × 757 | Sidebar 확장 상태. 테이블 컬럼 2열 (제목 + 상태) 축소 모드. 필터 뱃지 자동 줄바꿈 ✓ |
| **Mobile** | 606 × 701 (Chrome 창 최소치) | Sidebar 는 숨김 + 햄버거(`≡`) 버튼 노출. 햄버거 클릭 → **overlay 사이드바**에 컬렉션/폴더 섹션 포함 전체 네비 노출. 경로 이동 시 자동 닫힘 ✓ |

**세부 관찰**

- **Sidebar 레이아웃** — `SidebarNav shrink-0 + SidebarExploreTree flex-1 min-h-0 overflow-y-auto + SidebarUserPanel` 3-layer 구조가 모든 해상도에서 안정. 탐색 트리가 많아져도 자체 스크롤
- **모바일 햄버거 overlay** — 컬렉션/폴더 섹션이 내부에 포함. backdrop 탭하면 닫힘. ESC 키도 close (코드 확인)
- **필터 뱃지 wrap** — `flex flex-wrap gap-2` 로 좁은 폭에서도 자연 줄바꿈
- **DocumentAssignControls** — 데스크탑에서 한 줄 + flex-wrap 로 좁은 폭에서 세로 배치

---

## 4. "뷰 ≠ 권한 준수 확인" 시각 검증

`/documents?collection=<id>` 진입 시 권한 밖 문서가 노출되지 않는지 **UI 수준에서도** 점검.

- ✅ 사이드바의 `document_count` 뱃지는 **자기 owner 컬렉션** 의 문서 수만 반영. 타 owner 컬렉션은 사이드바에 나타나지도 않음 (CollectionsRepository `list_by_owner`)
- ✅ `AddDocumentsToCollectionModal` 에서 Scope 밖 문서를 체크 후 추가 시도 시 백엔드 `add_documents.rejected` 카운트만 반환 → `"${inserted}개 추가됨. 접근 권한이 없어 ${rejected}개는 제외되었습니다."` **info** 토스트. 구체 id 노출 없음
- ✅ `DocumentAssignControls` 의 컬렉션 chip 은 `in_collection_ids` (요청자 owner 범위만) 에만 의존. 다른 사용자의 동일 문서는 자기 컬렉션 chip 만 보임
- ✅ `/search?q=<>` 에서도 viewer Scope 필터가 활성 (UX 5차 scope wiring). `AdminUserDetailPage` 같은 별건 제외 FG 2-1 변경 범위 내에서 타 scope 문서 유출 없음

---

## 5. UI 리뷰 규정 충족 선언

| 규정 | 충족 상태 |
|------|----------|
| CLAUDE.md 프로젝트 §5.2 "UI 리뷰 ≥ 1회" | **5회+ 수행** — UX 1~5차 각 라운드에서 Chrome MCP 실측 완료 |
| task2-1.md §4 Step 6 "desktop/narrow desktop/tablet/mobile" | **4 관점 완결** — §3 표에 해상도별 실측 결과 기재 |
| task2-1.md §4 Step 7 "뷰 ≠ 권한 준수 확인" 섹션 | **§4 에 시각 검증 4 항목** — 회귀 테스트 결과는 `FG2-1_검수보고서.md` §6 참조 |

---

## 6. 남은 시각 UX 이월 항목

`FG2-1_검수보고서.md` §7 과 동일:

- 드래그-드롭 재배치 (폴더/컬렉션) — Phase 3 이월
- 컬렉션 description 편집 UI — 별도 소규모 FG
- SearchPage UI 에 collection/folder 필터 노출 — 백엔드 완결, 타입+UI 만 추가

---

*작성: 2026-04-24 | FG 2-1 UI 리뷰 로그 — Chrome 자동화 실측 5 라운드 + 4 관점 반응형 확정*
