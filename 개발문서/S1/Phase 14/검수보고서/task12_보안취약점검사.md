# Task 14-12 보안 취약점 검사 보고서

**작성일**: 2026-04-14
**검사 대상**: Monitoring API 3종 + `AdminMonitoringPage` + `LineChart` 컴포넌트

---

## 1. 검사 항목 및 결과

### 1.1 접근 제어 (RBAC)

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 1 | 모든 monitoring 엔드포인트에 `require_admin_access` | ✅ | 3 엔드포인트 모두 `Depends(...)` 적용 |
| 2 | 민감 데이터(연결 수/메모리) 노출은 관리자 한정 | ✅ | 동일 가드로 보호 |
| 3 | `/admin/monitoring` 라우트 AdminLayout AuthGuard 상속 | ✅ | Phase 14-9 구조 재사용 |

### 1.2 SQL Injection / Input Validation

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 4 | repository/엔드포인트 SQL은 `%s` 바인딩 | ✅ | monitoring 3 쿼리 모두 tuple param |
| 5 | `period` 파라미터 enum 화이트리스트 검증 | ✅ | `_validate_period`가 dict key 아니면 422 |
| 6 | 쿼리 내 f-string 문자열 보간 없음 | ✅ | monitoring 섹션에 `cur.execute(f"` 없음 |
| 7 | 버킷 간격(`interval_seconds`)도 dict 상수 | ✅ | 사용자 입력 직접 삽입 불가 |
| 8 | 프론트 URL 파라미터 `encodeURIComponent` | ✅ | `getResponseTimeTrend`/`getErrorTrend` |

### 1.3 정보 노출 / 에러 처리

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 9 | 컴포넌트 상태 체크 예외 메시지 truncate | ✅ | `str(exc)[:200]` — stack trace 미노출 |
| 10 | Valkey `info("memory")` 실패 시 조용히 폴백 | ✅ | 중첩 try/except |
| 11 | 비밀값/패스워드/토큰 반환 금지 | ✅ | 반환 dict에 민감 키 없음 (name/status/latency_ms/metadata만) |
| 12 | `latency_ms`만 외부 노출, 상세 타이밍 측정은 서버 측 | ✅ | 타이밍 기반 사이드채널 위험 제한적 |

### 1.4 XSS / CSRF

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 13 | `dangerouslySetInnerHTML` 미사용 | ✅ | `AdminMonitoringPage` / `LineChart` 모두 없음 |
| 14 | 사용자 입력은 React 기본 이스케이프 | ✅ | `{component.name}`, `{err.error_code}` 등 |
| 15 | SVG 내 사용자 데이터 직접 텍스트 주입 안함 | ✅ | 서버가 정수/timestamp만 반환, 차트가 숫자로 좌표 계산 |
| 16 | 메트릭 조회는 GET — CSRF 영향 없음 | ✅ | 상태 변경 액션 없음 |

### 1.5 감사 로깅 / 모니터링 자체 보안

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 17 | 모니터링 조회는 감사 이벤트 미발행 | ✅ | 조회만 수행, 노이즈 방지 |
| 18 | `components` 엔드포인트가 실패 요소를 상세히 노출하지 않음 | ✅ | status=DOWN + error 200자 truncate |
| 19 | 차트 데이터는 비즈니스 민감 정보 없음 | ✅ | P50/P95/P99 숫자 + 에러 카운트만 |

### 1.6 캐시 / 동시성 / 리소스 보호

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 20 | `period` bucket 최대 개수 제한 | ✅ | `7d`=28 버킷이 최대 — OOM/DoS 방어 |
| 21 | 쿼리 시간 범위 상한 존재 | ✅ | `ended_at > NOW() - make_interval(secs => total_seconds)` |
| 22 | components 타임아웃: 각 트라이 내 `time.perf_counter` | ✅ | DB/Valkey 장애 시 latency_ms 기록 후 다음 요소로 이동 |
| 23 | Valkey 장애 시 전체 API는 200 응답 유지 | ✅ | 각 요소 try/except 격리 |

### 1.7 공급망 / 의존성 보안

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 24 | 신규 외부 라이브러리 추가 없음 | ✅ | recharts/chart.js 등 미도입 — package.json 미변경 |
| 25 | 차트는 자체 SVG 구현 | ✅ | `LineChart.tsx` 250줄 자체 구현 |
| 26 | 기존 의존성 CVE 검사 | ✅ | `@tanstack/react-query` / `react` / `psycopg2` / `fastapi` — critical/high 0건 (npm audit / pip-audit 기준) |

### 1.8 클라이언트 사이드 보안

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 27 | 메트릭 값 localStorage 저장 안함 | ✅ | React Query in-memory |
| 28 | URL 파라미터에 민감 정보 없음 | ✅ | `period` enum-like만 |
| 29 | 외부 링크 / open redirect 없음 | ✅ | 내부 경로만 |

### 1.9 접근성 관련 보안

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 30 | 자동 갱신 안내 `aria-live="polite"` | ✅ | 과도한 알림 방지 (`assertive` 아님) |
| 31 | 모든 컨트롤 키보드 접근 가능 | ✅ | `<button>` / `<select>` + focus-visible |
| 32 | 모션 기반 장식 없음 (WCAG 2.3.3 대응) | ✅ | SVG 애니메이션 부재 |

---

## 2. 취약점 요약

| 등급 | 수량 | 상세 |
|:----:|:----:|------|
| Critical | 0 | - |
| High | 0 | - |
| Medium | 0 | - |
| Low | 0 | - |
| Info | 3 | 아래 참조 |

### Info 항목

**INFO-1: 실제 HTTP 응답시간 수집 미구현**
- 위치: `/monitoring/response-times` 엔드포인트
- 설명: 현재 background_jobs 실행 시간을 proxy로 사용. 실제 API 엔드포인트 latency는 FastAPI 미들웨어 추가 후 별도 테이블로 수집 예정.
- 위험도: 정보 (운영 지표의 정확도 이슈이며 보안 취약점 아님)
- 권고: Phase 15에서 `app/middleware/request_timing.py` 도입.

**INFO-2: pg_stat_activity 조회 권한**
- 위치: `components` 엔드포인트의 `SELECT count(*) FROM pg_stat_activity WHERE state='active'`
- 설명: 일반 앱 유저가 다른 세션의 query 문자열에는 접근하지 못하지만, 카운트는 읽힌다. 본 작업은 카운트만 노출 (query 내용 미노출) — 안전.
- 위험도: 정보
- 권고: 현 구조 유지.

**INFO-3: 멀티 인스턴스 latency 왜곡 가능성**
- 설명: `components` latency_ms는 요청을 처리한 인스턴스 기준. 여러 인스턴스를 로드밸런싱하는 프로덕션에서는 해당 요청을 처리한 노드의 건강 상태만 노출될 수 있음.
- 위험도: 정보 (보안보단 관측 편향)
- 권고: Phase 15 이후 Prometheus/Grafana 같은 외부 모니터링 도입 검토.

---

## 3. 외부 라이브러리 취약점

Task 14-12에서 **신규 추가된 의존성 0건**. 기존 `psycopg2-binary`, `fastapi`, `redis-py`, `pydantic`, `@tanstack/react-query`, `react` 18만 사용. 현행 버전 기준 critical/high CVE 0건.

특히 recharts/chart.js 등 차트 라이브러리 도입을 의도적으로 회피하여 공급망 공격 표면을 늘리지 않음 (CLAUDE.md §2 준수).

---

## 4. 결론

**Critical/High/Medium/Low 취약점 0건**. Monitoring API 3종은 RBAC 가드(`admin.read`) + period 화이트리스트 + 파라미터 바인딩 + 구성 요소별 try/except 격리로 다층 통제를 갖춤. 조회 전용이라 CSRF 영향 없음. 신규 의존성 미추가로 공급망 위험 증가 없음. Info 3건은 Phase 15 이후 개선 범위이며 현재 보안성에는 영향 없음.
