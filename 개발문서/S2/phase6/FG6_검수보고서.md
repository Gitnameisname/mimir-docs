# Phase 6 FG6.1/6.2/6.3 검수보고서

**검수일**: 2026-04-17  
**검수자**: Claude Sonnet 4.6 (AI 검수)  
**대상**: Phase 6 관리자 기능 통합 UI 전체

---

## 1. 검수 범위

| Feature Group | 주요 페이지 | 상태 |
|---------------|-------------|------|
| **FG6.1** | AdminProvidersPage, AdminPromptsPage, AdminCapabilitiesPage, AdminUsagePage | ✅ 완료 |
| **FG6.2** | AdminAgentsPage, AdminScopeProfilesPage, AdminProposalsPage, AdminAgentActivityPage | ✅ 완료 |
| **FG6.3** | AdminGoldenSetsPage, AdminEvaluationsPage, AdminExtractionSchemasPage, AdminExtractionQueuePage | ✅ 완료 (스켈레톤) |
| **공통** | AdminSidebar S2 확장, S2 Admin API 타입/클라이언트, MCP 가이드 | ✅ 완료 |

---

## 2. UI 리뷰 기록 (5회)

### 1차 리뷰 — 와이어프레임 (레이아웃 구조)

| 점검 항목 | 결과 | 비고 |
|-----------|------|------|
| S1/S2 섹션 구분 (네비게이션) | ✅ | "S1 관리" / "AI 플랫폼" / "에이전트·Scope" / "평가·추출" 4구간 명시 |
| 페이지별 h1 태그 | ✅ | 모든 페이지 단일 h1 사용 |
| 모달 role/aria-modal | ✅ | 모든 모달 `role="dialog" aria-modal="true"` |
| 반응형 그리드 | ✅ | `grid-cols-1 sm:grid-cols-2 xl:grid-cols-3` 패턴 |

**결론**: 레이아웃 구조 기본 요건 충족.

### 2차 리뷰 — 프로토타입 (기본 레이아웃)

| 점검 항목 | 결과 | 비고 |
|-----------|------|------|
| 테이블 헤더 `uppercase tracking-wide` | ✅ | 일관 적용 |
| 로딩 스켈레톤 | ✅ | 모든 페이지 animate-pulse 구현 |
| 빈 상태 메시지 | ✅ | 테이블 emptyMessage 처리 |
| 에러 상태 (retry 버튼) | ✅ | `isError` 분기 + refetch() 버튼 |

**결론**: 기본 레이아웃 일관성 확인. 

### 3차 리뷰 — 기능 통합 (API 바인딩)

| 점검 항목 | 결과 | 비고 |
|-----------|------|------|
| React Query useQuery 패턴 | ✅ | 기존 adminApi 패턴과 동일 |
| useMutation + onSuccess invalidate | ✅ | 모든 뮤테이션 쿼리 캐시 무효화 |
| 낙관적 업데이트 vs 안전 업데이트 | ✅ | Delete는 confirm 필수, POST는 즉시 |
| FG6.3 스켈레톤 목 데이터 | ✅ | MOCK_* 상수로 UX 검증 가능 |

**결론**: API 바인딩 구조 검증 완료.

### 4차 리뷰 — 반응형·접근성

| 점검 항목 | 결과 | 비고 |
|-----------|------|------|
| 모바일(640px) 레이아웃 | ✅ | `sm:` breakpoint 적용 |
| 태블릿(768px) | ✅ | `md:` 일부, `sm:` 기준 충족 |
| 데스크탑(1024px+) | ✅ | `xl:` 확장 그리드 |
| 최소 터치 영역 44px | ✅ | `min-h-[44px]` 버튼 전체 적용 |
| aria-label 버튼 | ✅ | 아이콘 전용 버튼 모두 aria-label |
| 키보드 포커스 링 | ✅ | `focus:outline-none focus:ring-2` 패턴 |
| aria-live (실시간 갱신) | ✅ | refetch 인터벌 데이터 영역 |
| 색 대비 (WCAG AA) | ⚠️ | 일부 `text-gray-400` 대비 검토 필요 |

**개선 사항**: `text-gray-400` → `text-gray-500` 전환 권고 (대비 비율 개선).

### 5차 리뷰 — 최종 폴리시 (일관성·오류 처리)

| 점검 항목 | 결과 | 비고 |
|-----------|------|------|
| 에러 메시지 role="alert" | ✅ | 폼 에러 및 API 에러 적용 |
| 삭제 확인 (2단계) | ✅ | confirmDelete 패턴 구현 |
| 킬스위치 확인 모달 | ✅ | KillSwitchModal 별도 컴포넌트 |
| 일괄 작업 실행취소 방법 | ⚠️ | Batch 승인/거절 후 롤백 UI 미구현 (Phase 7 과제) |
| FG6.3 스켈레톤 명확한 안내 | ✅ | 황색 배너로 "Phase X 후 활성화" 명시 |
| S1/S2 네비게이션 일관성 | ✅ | 동일 AdminSidebar 컴포넌트 사용 |

**결론**: 전체 5회 리뷰 완료. 주요 이슈 2건은 개선 권고 수준.

---

## 3. 기능 검수 결과

### FG6.1

| 검수 항목 | 결과 | 비고 |
|-----------|------|------|
| 모델 CRUD 정상 작동 | ✅ | POST/PATCH/DELETE API 바인딩 완료 |
| "연결 테스트" 후 결과 모달 표시 | ✅ | TestResultModal 즉시 표시 |
| 프롬프트 버전 타임라인 | ✅ | VersionTimeline 컴포넌트 구현 |
| "이 버전 활성화" 버튼 | ✅ | activateVersion mutation 연동 |
| A/B 테스트 설정 표시 | ✅ | ab_test_config 배지 표시 |
| Capabilities 실시간 갱신 | ✅ | 60초 refetchInterval + 수동 버튼 |
| 저하 상태 배너 | ✅ | DegradedBanner 컴포넌트 구현 |
| 비용 시계열 그래프 (stacked bar) | ✅ | recharts BarChart 구현 |
| 모델별 상세 테이블 | ✅ | ModelSummaryTable 구현 |
| CSV 다운로드 | ✅ | ExportButton href 링크 구현 |

### FG6.2

| 검수 항목 | 결과 | 비고 |
|-----------|------|------|
| 에이전트 킬스위치 발동 모달 | ✅ | KillSwitchModal (1h/24h/permanent) |
| 대기 중인 제안 자동 거절 옵션 | ✅ | reject_pending 체크박스 |
| Scope Profile ACL 필터 시각 빌더 | ✅ | FilterBuilder (visual/JSON 모드) |
| 제안 큐 Diff 뷰 | ✅ | DiffViewer (라인별 색상 구분) |
| 일괄 승인/거절 (체크박스) | ✅ | BatchToolbar + batchApprove/Reject |
| 에이전트 감사 이력 탭 | ✅ | PH5-CARRY-001 이월 처리 완료 |
| Rate Limit 동적 설정 UI | ✅ | PH5-CARRY-002 이월 처리 완료 |
| 에이전트 활동 차트 | ✅ | BarChart + LineChart 구현 |
| 이상 행동 감지 배너 | ✅ | AnomalyBanner 컴포넌트 |
| MCP 통합 가이드 문서 | ✅ | DOCS-001 이월 처리 완료 |

### FG6.3

| 검수 항목 | 결과 | 비고 |
|-----------|------|------|
| 모든 스켈레톤 레이아웃 반응형 | ✅ | 동일 Tailwind 패턴 |
| Phase 7·8 API 연결 가능성 | ✅ | retry: false + 폴백 MOCK_* |
| WCAG 2.1 AA 접근성 | ✅ (⚠️) | gray-400 개선 권고 외 충족 |
| 라우팅 명확성 | ✅ | `/admin/golden-sets`, `/admin/evaluations` 등 |
| 스켈레톤 상태 안내 배너 | ✅ | 황색 배너 일관 적용 |

---

## 4. 이월 항목 처리 현황

| ID | 항목 | 처리 결과 |
|----|------|-----------|
| PH5-CARRY-001 | 에이전트 감사 이력 UI | ✅ Task 6-8 AdminAgentsPage 감사 이력 탭으로 완료 |
| PH5-CARRY-002 | Rate Limit 동적 설정 | ✅ AdminAgentsPage Rate Limit 섹션 + 전용 모달로 완료 |
| PH5-CARRY-003 | 1000개 이상 부하 테스트 | ⏳ 배포 직전 수행 예정 (Phase 6 배포 전) |
| DOCS-001 | MCP 통합 가이드 | ✅ `docs/개발문서/S2/mcp_integration_guide.md` 완료 |
| PH3-CARRY-002 | 대화 FTS 전환 | ⏳ Phase 6 검색 고도화 시 처리 예정 |

---

## 5. 미결 사항

| 번호 | 항목 | 우선순위 | 처리 시점 | 처리 결과 |
|------|------|---------|-----------|-----------|
| 1 | text-gray-400 색 대비 개선 (WCAG AA) | 낮음 | Phase 6 최종 배포 전 | ✅ 완료 — body text 10개소 text-gray-500 전환 |
| 2 | Batch 승인/거절 롤백 UI | 중간 | Phase 7 | ⏳ Phase 7 이월 → task7-11 |
| 3 | PH5-CARRY-003 부하 테스트 | 낮음 | 배포 직전 | ⏳ Phase 7 이월 → task7-12 |
| 4 | PH3-CARRY-002 대화 FTS 전환 | 낮음 | Phase 7 | ⏳ Phase 7 이월 → task7-10 |

### 미결 사항 처리 상세

**1번 (text-gray-400 → text-gray-500) — ✅ 2026-04-17 완료**

Phase 6 신규 파일에서 정보 전달 body text에 해당하는 `text-gray-400` 10개소를 `text-gray-500`으로 수정:
- `AdminAgentsPage.tsx`: Rate Limit 오류문, 위임 없음 메시지, 감사이벤트 없음, 에이전트 없음 (4개소)
- `AdminProposalsPage.tsx`: 제안 없음 메시지 (1개소)
- `AdminScopeProfilesPage.tsx`: Scope 없음, Scope Profile 없음 (2개소)
- `AdminProvidersPage.tsx`: 프로바이더 없음 (1개소)
- `AdminPromptsPage.tsx`: 버전 없음, 프롬프트 없음 (2개소)
- `AdminUsagePage.tsx`: 사용 데이터 없음 (1개소)
- `AdminAgentActivityPage.tsx`: 활동 없음 (1개소)
- `AdminGoldenSetsPage.tsx`: Phase 7 연결 안내 (1개소)

비고: 아이콘 색상(`w-5 h-5 text-gray-400`)과 비활성 버튼(`cursor-not-allowed text-gray-400`)은 의도적 처리로 변경 제외.

**2, 3, 4번 — Phase 7 이월 완료**

Phase 7 개발계획서 §3.5에 이월 항목 섹션 추가 완료. 각 항목별 작업지시서 신규 생성:
- `docs/개발문서/S2/phase7/작업지시서/task7-10.md` (PH3-CARRY-002)
- `docs/개발문서/S2/phase7/작업지시서/task7-11.md` (PH6-CARRY-001)
- `docs/개발문서/S2/phase7/작업지시서/task7-12.md` (PH5-CARRY-003)

---

**검수 결론**: FG6.1, FG6.2, FG6.3 모두 기능 구현 및 5회 UI 리뷰 완료. Phase 7·8 API 연결을 위한 스켈레톤 상태 준비 완료. 미결 사항 4건은 모두 낮음~중간 우선순위로, Phase 7 진입 가능 판정.
