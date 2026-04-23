# FG 0-5 UI 디자인 리뷰 로그 — 벡터화 상태 패널

| 항목 | 값 |
|------|----|
| 대상 컴포넌트 | `frontend/src/features/documents/VectorizationPanel.tsx` |
| 대상 페이지 | `frontend/src/features/documents/DocumentDetailPage.tsx` |
| 작성일 | 2026-04-23 |
| 리뷰 주체 | Claude Opus 4.7 (self-review × 5 iterations per CLAUDE.md §4) |
| 관련 | `task0-5.md` §4.5, `FG0-5_검수보고서.md` |

CLAUDE.md UI 규칙에 따라 **최소 5회 리뷰**를 수행하며 각 라운드에서 발견한 문제와 개선안을 기록한다. 본 FG 는 "2026-04-23 실사례 — RAG 가 문서를 찾지 못한 원인을 탐지/복구할 수단이 없었음" 동기에서 출발했으므로, **탐지(diagnosis) · 복구(recovery)** 두 축을 UX 로 잘 담는지를 우선 평가한다.

---

## 리뷰 1회차 — 배치 위치 결정 (Placement)

### 쟁점
task0-5 §4.5 가 제시한 2개 옵션:
- A. 문서 헤더 우측 영역에 소형 뱃지 + 아이콘 버튼
- B. 사이드 패널 독립 섹션 (Contributors/메타데이터 인근)

### 고려 사항
- **시선 흐름**: 사용자가 문서 상세에 도달했을 때 첫 0.5초에 "이 문서가 RAG 에 잡히는가?" 를 알 수 있어야 함.
- **현재 레이아웃**: `DocumentDetailPage` 는 본문 max-w-3xl 중앙 + 구조 트리 좌측 aside. 별도 "사이드 패널" 영역이 없음 → B 채택 시 신규 영역 생성 필요, 기존 레이아웃 재구성 비용 큼.
- **상태 부재 시 차지 공간**: B 는 독립 섹션이라 "indexed" 정상일 때도 큰 영역. A 는 뱃지만 있으면 되므로 공간 효율 ↑.

### 결론
**A 채택** (문서 헤더 우측 영역, `WorkflowStatusBadge` 옆).
`<div className="flex flex-col items-end gap-1">` 에 WorkflowStatusBadge 와 VectorizationPanel(compact) 두 개를 세로 정렬.

### 개선 행동
- VectorizationPanel 컴포넌트에 `compact?: boolean` prop 추가 → 헤더에 embed 시 size sm, 미래에 사이드 패널 내에서 큰 영역으로 쓸 때 `compact={false}` 로 재사용.

---

## 리뷰 2회차 — 상태 뱃지의 시각 언어 (Status Affordance)

### 쟁점
- 6가지 상태(`indexed/pending/in_progress/failed/stale/not_applicable`) 를 **색만으로** 구별하면 색약 사용자 대응 부족.
- "stale" 과 "failed" 는 둘 다 사용자에게 "뭔가 조치 필요" 신호인데, 구분이 불명확.

### 발견 사항
- 초기안: 색만 사용(green/amber/red/gray) → WCAG AA 대비율만 고려하면 충족하지만 색약 대응 실패.
- "stale" = amber + "↻" 아이콘, "failed" = red + "✕" 아이콘 → 아이콘과 색을 2중 신호화.

### 개선 행동
- 각 상태에 **단일 문자 아이콘** 추가: ✓ (indexed) / ↻ (stale) / … (pending) / ⟳ (in_progress) / ✕ (failed) / — (not_applicable).
- `title` 속성에 screen reader 용 상세 설명(`srText`). 예: "최근 발행된 버전이 아직 색인되지 않았습니다".
- 코드 레퍼런스: `VectorizationPanel.tsx` 의 `STATUS_DISPLAY` 맵.

### 미해결 이슈
- 아이콘으로 쓴 이모지/유니코드는 시스템 폰트 의존성이 있음 → iOS/Android/Windows 별 렌더 차이 존재. Phase 1+ 에서 lucide-react 아이콘으로 치환 검토.

---

## 리뷰 3회차 — 재벡터화 버튼 상태 머신 (Interaction States)

### 쟁점
버튼은 다음 **4가지 상태** 를 가짐:
1. 일반 (enabled) — 클릭 가능
2. In-flight — API 호출 중 (스피너)
3. 쿨다운 중 — 서버 429 방어 중 (`reindex_cooldown_sec > 0`)
4. 폴링 중 — 클릭 후 2초 간격 상태 폴링 (max 60s)

이들을 각각 **라벨/disabled/tooltip** 으로 구분해야 하며, 사용자는 "지금 뭐가 일어나고 있는지" 를 정확히 이해할 수 있어야 함.

### 초기안 문제
- 쿨다운 중과 폴링 중을 같은 disabled 로 취급 → 사용자가 "왜 클릭이 안 되는지" 구분 불가.
- 폴링 중인데 쿨다운이 끝나 버튼이 다시 활성되는 race → 사용자가 중복 클릭.

### 개선 행동
- 라벨을 상태별로 명시적 분기:
  - `inFlight` → "요청 중…"
  - `isPolling` → "진행 중…"
  - `cooldownActive` → "쿨다운 Ns"
  - 기본 → "재벡터화"
- `disabled = inFlight || cooldownActive || isPolling` 로 **OR 조건** — 폴링 중에도 중복 클릭 차단.
- `title` 속성으로 각 상태의 이유 설명 (hover tooltip).

### 결론
상태 머신 명확화로 "왜 지금 못 누르는가" 설명 가능. `data-status` 속성으로 E2E 테스트에서 상태 단정 가능.

---

## 리뷰 4회차 — 에러 메시지 노출 수준 (Error Disclosure)

### 쟁점
`status=failed` 일 때 `last_error` 가 유용하지만, 전체 스택 트레이스를 바로 보여주면:
- UI 공간 낭비 (90% 사용자는 짧은 요약만 원함)
- 사내 구성 정보 노출 (`MilvusConnectionError: milvus.internal:19530 ...`) — 일반 사용자에게 너무 상세

### 개선 행동
- 초기안: `last_error` 를 `<pre>` 로 항상 전개 → 부담스러움.
- 개선: **"에러 상세 보기" 토글 버튼** 도입.
  - 기본: 접힘 (버튼만)
  - 클릭: 펼침 (전체 에러 표시)
- 펼쳐진 영역은 `max-h-40 overflow-auto` 로 스크롤 + `whitespace-pre-wrap` 로 줄바꿈 보존.
- `aria-expanded` 로 접근성 확보.
- `data-testid="vec-error-details"` 로 컴포넌트 테스트 용이.

### 추가 결정
- 전체 스택은 Admin 전용으로 하려 했으나, 현재 API 가 권한 분기 없이 `last_error` 를 그대로 내림. 본 FG 범위에서는 "토글로 숨기는" 수준 방어까지. Phase 1+ 에서 API 응답 필드 수준으로 권한 분기 검토 (`FG0-5_보안취약점검사보고서.md` 참조).

---

## 리뷰 5회차 — 접근성·키보드·모바일 (A11y & Responsive)

### 점검 항목

| 항목 | 결과 | 근거 |
|------|------|------|
| `aria-live="polite"` (상태 변경 시 스크린 리더 알림) | ✅ | `<div role="status" aria-live="polite">` |
| 버튼 `aria-label="재벡터화 실행"` | ✅ | `Button` 의 `aria-label` prop 설정 |
| 키보드 네비게이션 — Tab 순서 | ✅ | 헤더 내 Button 은 기본 tabindex=0, `WorkflowStatusBadge` 이후 자연 순서 |
| Enter/Space 로 버튼 활성 | ✅ | 기본 `<button>` 동작. 추가 처리 불필요 |
| 포커스 링 | ✅ | `Button` 컴포넌트 기본 focus-visible ring |
| 색 대비 (WCAG AA 4.5:1) | ✅ | `bg-green-100 text-green-800`, `bg-red-100 text-red-800` 등 Tailwind 기본 pair 모두 AA 통과 |
| 모바일 breakpoint | △ | `flex-col items-end gap-1` 이 좁은 화면(320px)에서 WorkflowStatusBadge 와 VectorizationPanel 세로 정렬 유지. 헤더 title 과 겹칠 수 있음 → title `flex-1` 가 있어 wrap 되므로 OK. 실제 UA 에서 최종 확인 필요 |
| 다크 모드 | ❌ | 현재 color pair 는 light only. 프로젝트 전반이 light 전용이라 본 FG 에선 범위 밖 |

### 개선 행동
- 토글 버튼 (에러 상세) 에 `aria-expanded` 명시적 설정.
- compact mode 에서도 충분한 라벨 공간 유지 (chunk_count 는 opacity-70 로 덜 두드러지게).
- 모바일 좁은 화면에서 버튼 텍스트 "재벡터화" 가 잘리면 아이콘만 보이도록 개선하는 것은 Phase 1+ 검토 (현재는 text 유지).

### 남은 개선 후보 (Phase 1+ )
- `failed` 상태 에러 스택의 Admin-only 재노출
- lucide-react 아이콘으로 이모지 치환
- 다크 모드 pair
- 폴링 종료 시점의 토스트 (예: "색인 완료됐습니다")

---

## 최종 결론

5회 리뷰를 통해 다음이 확정되었다:

1. **배치**: 문서 헤더 우측, WorkflowStatusBadge 와 세로 정렬. compact prop 으로 미래 재배치 가능.
2. **시각 언어**: 6상태 각각 색 + 단일 문자 아이콘 + srText 3중 신호.
3. **버튼 상태 머신**: in-flight / cooldown / polling 3상황을 각각 라벨·disabled·tooltip 으로 분기.
4. **에러 노출**: 토글 방식으로 기본 접힘, 필요 시 펼침.
5. **접근성**: aria-live, aria-label, aria-expanded, 키보드 탭 순서 준수.

본 UI 는 **탐지(진단)** 에 충분하고 **복구(재벡터화)** 에도 실수 차단(쿨다운 · 확인 모달) 이 걸려있다. 2026-04-23 사용자 불만(원인 진단 수단 부재) 을 직접적으로 해소한다.

---

## 부록 A — 컴포넌트 테스트 체크리스트 (FG 0-5 F 에서 사용)

| # | 시나리오 | 기대 |
|---|---------|------|
| 1 | `status=indexed` | 뱃지 "✓ 색인됨" / 재벡터화 버튼 렌더 (can_reindex=true 시) |
| 2 | `status=stale` | 뱃지 "↻ 최신 미반영" / amber |
| 3 | `status=failed + last_error` | 에러 토글 렌더, 클릭 시 펼쳐짐 |
| 4 | `can_reindex=false` | 버튼 비렌더 |
| 5 | `reindex_cooldown_sec=7` | 버튼 disabled + "쿨다운 7s" |
| 6 | 버튼 클릭 → 확인 모달 → 확인 | mutate 호출 + 폴링 시작 |
| 7 | 폴링 중 → `indexed` 도달 | 폴링 중지 |
| 8 | 폴링 60초 상한 | 자동 중지 |
| 9 | `status=not_applicable` | 뱃지 "— 해당 없음" / 버튼 숨김 |
| 10 | 로딩 중 | "벡터화 상태 조회 중…" 뱃지 |

---

*5회 리뷰 완료 | CLAUDE.md UI 규칙 준수 | 최종 본: `VectorizationPanel.tsx`*
