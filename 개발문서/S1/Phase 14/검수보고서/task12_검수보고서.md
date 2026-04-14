# Task 14-12 모니터링 대시보드 검수 보고서

**작성일**: 2026-04-14
**범위**: 백엔드 모니터링 API 3종 + 프론트 `AdminMonitoringPage` + `LineChart` 컴포넌트

---

## 1. 구현 개요

| 영역 | 산출물 | 비고 |
|------|--------|------|
| 백엔드 API | `GET /api/v1/admin/monitoring/response-times` | background_jobs (ended - started)*1000ms 기반 P50/P95/P99 |
| 백엔드 API | `GET /api/v1/admin/monitoring/error-trends` | audit_events(denied/conflict=4xx, failure=5xx) + jobs(FAILED=5xx) |
| 백엔드 API | `GET /api/v1/admin/monitoring/components` | PostgreSQL / Valkey / Vector DB / Job Runner 상태 + latency_ms |
| 프론트 컴포넌트 | `LineChart.tsx` (경량 SVG) | recharts 미도입 — 공급망 위험 차단 (CLAUDE.md §2) |
| 프론트 페이지 | `AdminMonitoringPage.tsx` | 자동 갱신(10s/30s/1m/수동) + 기간(1h/6h/24h/7d) |
| 라우트 | `/admin/monitoring/page.tsx` | AdminLayout `AuthGuard` 상속 |

### 기술 선택

- **외부 차트 라이브러리 미추가**: recharts v2 GA지만 신규 의존성 도입 시 공급망 위험이 증가하며 번들 크기에도 영향. Custom SVG LineChart (약 250줄)로 대체 — 호버 툴팁, 십자선, 범례, aria-label, 빈 데이터 폴백 구현.
- **PostgreSQL 시간 버킷**: `generate_series` + `percentile_cont(...) WITHIN GROUP`으로 앱 단 후처리 없이 P50/P95/P99 산출. Bucket 경계는 `to_timestamp(floor(epoch / N) * N)`.
- **Valkey 장애 분리**: `components` 엔드포인트 내에서 Valkey 연결 실패해도 PostgreSQL/Vector DB/Job Runner는 독립 검사 (try/except 블록별 격리).

---

## 2. UI 디자인 리뷰 (5회)

| 회차 | 개선 초점 | 반영 사항 |
|:----:|---------|----------|
| 1 | 정보 밀도 | 4개 컴포넌트 카드(그리드) + 2개 추이 차트(그리드) + 최근 에러 테이블 3 섹션 분리 |
| 2 | 자동 갱신 UX | 다음 갱신까지 초 단위 카운트다운 + `aria-live="polite"` 안내, 수동 갱신 옵션 포함 |
| 3 | 에러 복원력 | 일부 쿼리 실패 시 `placeholderData: prev`로 이전 값 유지 + 배너에 "이전 갱신 기준" 표시 |
| 4 | 반응형 | 컴포넌트 그리드 `1 → sm:2 → lg:4`, 차트 `1 → lg:2`, 헤더 `flex-wrap`, 테이블 `overflow-x-auto` |
| 5 | 접근성 | 카드 `role="group"`, 섹션 `aria-labelledby`, 탭 `aria-pressed`, 최소 터치 타깃 `min-h-[40px]`, focus-visible 링 |

---

## 3. 자동 검증 결과

`backend/tests/unit/test_admin_monitoring_phase14_12.py` 실행 결과 **78/78 PASS**.

| 범주 | 통과 | 대표 항목 |
|------|------|----------|
| API (20) | 20/20 | 라우트 3종, period 422 검증, percentile_cont 사용, RBAC 가드 |
| Security (8) | 8/8 | SQL 파라미터 바인딩, period enum, 에러 메시지 truncate, encodeURIComponent |
| Types (5) | 5/5 | `"HEALTHY" \| "DOWN" \| "UNKNOWN"` literal union, P50/P95/P99 필드 |
| FE API (4) | 4/4 | 3 API 클라이언트 함수 + period 기본값 24h |
| LineChart (9) | 9/9 | SVG viewBox, role=img, aria-live 툴팁, 빈 데이터 폴백, 외부 lib 미사용 |
| UI (14) | 14/14 | 갱신 옵션 4종, period 탭 4종, placeholderData, refetchInterval, aria-pressed |
| A11Y (8) | 8/8 | aria-labelledby, sr-only, aria-live, focus-visible, scope=col, role=status |
| Responsive (6) | 6/6 | `p-4 sm:p-6`, `grid-cols-1 sm:grid-cols-2 lg:grid-cols-4`, `text-xl sm:text-2xl` |
| Route (3) | 3/3 | AdminMonitoringPage 마운트, metadata, placeholder 제거 |
| Design (1) | 1/1 | recharts/chart.js 등 외부 차트 lib 미추가 |

---

## 4. CLAUDE.md 규칙 준수

| 규칙 | 준수 여부 | 비고 |
|------|:--------:|------|
| §1 deprecated 기능 회피 | ✅ | FastAPI/React Query 현행 API만 사용 |
| §2 critical 취약점 라이브러리 미도입 | ✅ | 신규 의존성 0개 — 기존 `@tanstack/react-query`만 활용 |
| §3 보안 고려 | ✅ | Task12 보안 보고서 참조 (0 Critical/High/Medium/Low) |
| §4 UI 리뷰 5회 이상 | ✅ | 위 2절 5회 반영 |
| 문서 타입 하드코딩 금지 | 해당 없음 | 모니터링은 시스템 메트릭이라 doc_type 의존 없음 |
| generic + config 기반 | ✅ | `_MONITORING_PERIODS` dict으로 기간 정의 — 하드코딩 분기 없음 |

---

## 5. 제약 사항 및 향후 개선

1. **실제 HTTP 응답 시간 수집 미구현**: FastAPI 미들웨어 부재로 현재 백엔드에는 HTTP 레이턴시 샘플이 쌓이지 않음. 대신 `background_jobs`의 실행 시간을 proxy 데이터로 사용. 향후 `app/middleware/request_timing.py` 도입 후 별도 테이블(예: `request_metrics`)로 교체 권고.
2. **멀티 인스턴스 latency 수집**: `components` 엔드포인트는 현재 요청을 처리하는 워커의 local latency만 반영. 진짜 프로덕션 모니터링은 Prometheus 같은 외부 시스템 연동 필요 (Phase 15 이후).
3. **차트 Y축 단위 자동 포맷**: ms/건수 단위를 props로 받지만 K/M 축약은 미구현 (현재 max가 작아 체감 이슈 없음).

이상 3건은 Phase 14-12 범위를 벗어나며 별도 태스크로 관리한다.

---

## 6. 결론

Task 14-12 **완료**. 78건 자동 검증 전원 통과, UI 5회 리뷰 반영, 신규 외부 의존성 0개, RBAC/SQL 안전성/접근성/반응형 모두 확보. 후속 Task 14-13 진행 가능.
