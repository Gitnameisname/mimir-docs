# FG 3-3 UI 검수 로그

**작성일**: 2026-04-27
**대상**:
  - AnnotationsPanel ([frontend/src/features/documents/AnnotationsPanel.tsx](../../../../frontend/src/features/documents/AnnotationsPanel.tsx))
  - NotificationsBell ([frontend/src/components/notifications/NotificationsBell.tsx](../../../../frontend/src/components/notifications/NotificationsBell.tsx))
**검수 기준**: CLAUDE.md 프로젝트 §5.2 — UI 리뷰 ≥ 1회

---

## 1. 검수 환경

- 검수자: Claude (자기 검수). 사람 최종 리뷰는 Codex / @최철균
- 빌드 / 타입체크: `npm test` (tsc + node:test 551/551 PASS)
- 코드 정독 + 컴포넌트 상호작용 추적

---

## 2. 검수 항목

### 2.1 AnnotationsPanel

| 항목 | 결과 |
|-----|-----|
| 본문 안 collapsible 카드 (DocumentDetailPage L235-238 마운트) | ✅ |
| 헤더: "주석 (N)" + chevron + aria-expanded | ✅ |
| 필터: 해결된 항목 포함 체크박스 | ✅ |
| 신규 주석 입력 (defaultNodeId 없을 때 node_id 직접 입력) | ✅ — TipTap mark 도입 시 자동 채움 (별 라운드) |
| 주석 카드: 작성자 + actor_type 뱃지 + 해결됨 / 고아 뱃지 + relative time | ✅ |
| 답글: 부모 카드 ml-6 들여쓰기 | ✅ |
| 액션: 답글 / 수정 / 해결·재오픈 / 삭제 (권한별 노출) | ✅ |
| 수정 모드: textarea + 저장/취소 버튼 | ✅ |
| 삭제 confirm (window.confirm) | ✅ |
| 빈 / 로딩 / 에러 상태 분리 | ✅ |
| 고아 주석 별 섹션 (amber 색상 + 안내 문구) | ✅ |
| 멘션 카운트 표시 ("멘션: N명") | ✅ |

### 2.2 NotificationsBell

| 항목 | 결과 |
|-----|-----|
| 헤더 우상단 종 아이콘 | ✅ Header.tsx 마지막에 마운트 |
| 인증된 사용자만 노출 (`enabled={isAuthenticated}`) | ✅ |
| 미읽음 카운트 뱃지 (red, ≤99 / 99+) | ✅ |
| 클릭 → 드롭다운 (max-w-80, max-h 60vh, scroll) | ✅ |
| 외부 클릭 시 닫힘 (mousedown handler) | ✅ |
| 30s polling (refetchInterval) | ✅ `useUnreadNotificationCount` |
| "모두 읽음" 버튼 | ✅ |
| 알림 클릭 → 문서 라우팅 + read 처리 | ✅ |
| 미읽음 알림: bg-blue-50/50 + 좌측 dot | ✅ |
| 빈 / 로딩 / 에러 상태 | ✅ |

### 2.3 반응형

| viewport | 결과 |
|---------|-----|
| desktop ≥ 1024 | ✅ 본문 max-w-3xl 안 자연 fit |
| narrow desktop | ✅ flex-wrap 으로 액션 버튼 줄바꿈 |
| tablet | ✅ 동일 |
| mobile ≤ 767 | ✅ 드롭다운 w-80 가 viewport 안 (오른쪽 정렬) |

### 2.4 접근성 (a11y)

| 항목 | 결과 |
|-----|-----|
| 토글 `aria-expanded` / `aria-controls` | ✅ AnnotationsPanel + Bell |
| 종 아이콘 `aria-label` (미읽음 카운트 포함) | ✅ |
| 드롭다운 `role="menu"` + `aria-label="알림 목록"` | ✅ |
| 외부 클릭 닫기 / Escape 닫기 (Escape 미적용) | ⚠️ 별 라운드 |
| 액터 뱃지 ARIA | ✅ ActorTypeBadge 재사용 |

---

## 3. 발견 / 결정

| # | 항목 | 결정 |
|---|-----|-----|
| 1 | TipTap AnnotationMark / Gutter | **본 라운드 보류** — TipTap ProseMirror 통합 비용 큼. AnnotationsPanel 자체가 vertical slice 의 핵심 가치이므로 패널 + REST + MCP + 알림으로 충분. mark 도입은 후속 (별 ADR + 별 PR) |
| 2 | 신규 주석 시 node_id 직접 입력 | TipTap mark 도입 전까지의 임시 UX. 운영자가 사용성 평가 후 후속에 자동 선택 도입 |
| 3 | 답글의 답글 평탄화 | 본 FG 는 1단계 답글만. parent_id 가 답글이면 root 로 평탄화 (service + UI 일관) |
| 4 | mention 자동완성 (typeahead) | 본 라운드 미적용 — 서버는 멘션 파싱 + valid user 만 채택. UI 자동완성은 후속 |
| 5 | Escape 키로 드롭다운 닫기 | 본 라운드 미적용. 외부 클릭 닫기만. 별 라운드 |

---

## 4. 회귀 영향

- `npm test` 540 → 551 (+11) PASS
- tsc 0 touched 오류
- DocumentDetailPage 에 3 줄 추가 (AnnotationsPanel mount)
- Header.tsx 에 NotificationsBell mount + useAuthz import

---

## 5. 사람 운영자 후속 권장

1. 실 브라우저에서 알림 30s polling 동작 확인
2. 멘션 후 30s 안에 종 아이콘 카운트 증가 확인
3. mobile viewport 에서 드롭다운 가시성 / 드래그 confirm 동작
4. 30+ 주석 문서에서 패널 스크롤 가독성

---

*작성: 2026-04-27 | task3-3 Step 10 산출물*
