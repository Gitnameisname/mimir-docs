# Task 14-12. 모니터링 대시보드 UI 구현

## 1. 작업 목적

시스템 상태, 에러 추이, 응답 시간 등을 시각적으로 보여주는 모니터링 대시보드를 구현한다.

이 작업의 목표는 다음과 같다.

- 시스템 상태 상세 뷰 (DB, Valkey, 벡터 DB, Job Runner)
- 에러 추이 차트 (시간대별 에러 발생 수)
- API 응답 시간 그래프 (P50, P95, P99)
- 리소스 사용량 표시
- 자동 갱신 (30초 간격)

---

## 2. 작업 범위

### 포함 범위

- `/admin/monitoring` UI 페이지
- 시스템 상태 상세 카드 (각 구성 요소별 상태/지표)
- 에러 추이 라인 차트 (최근 24시간)
- API 응답 시간 라인 차트
- 기존 `GET /admin/dashboard/health`, `/admin/dashboard/errors`, `/admin/dashboard/metrics` API 활용
- Recharts 라이브러리를 이용한 차트 렌더링

### 제외 범위

- 알림 규칙 설정 (Task 14-13에서 구현)
- APM 상세 (외부 도구 연계)
- 인프라 메트릭 (CPU, 메모리 등 — Phase 13 모니터링 스택 연계)

---

## 3. 선행 조건

- Task 14-9 Admin 레이아웃 완료
- Phase 13 모니터링 체계 (메트릭 수집, `/metrics` 엔드포인트)
- 기존 Admin Dashboard API 이해

---

## 4. 주요 구현 대상

### 4-1. 모니터링 페이지 레이아웃

```
모니터링 대시보드                              [자동 갱신: 30초 ▼]  [새로고침]

┌─ 시스템 상태 ──────────────────────────────────────────────────────┐
│ ● PostgreSQL  정상  12ms  │ ● Valkey  정상  2ms                   │
│ ● 벡터 DB     정상  45ms  │ ● Jobs    정상  대기 0                 │
└────────────────────────────────────────────────────────────────────┘

┌─ API 응답 시간 (최근 24시간) ─────────┬─ 에러 추이 (최근 24시간) ──────────┐
│                                      │                                   │
│  ms                                  │  건수                              │
│  500 ┤                ╭─╮            │   20 ┤                             │
│  300 ┤        ╭───╮  │ │            │   10 ┤    ╭─╮      ╭╮             │
│  100 ┤────────╯   ╰──╯ ╰───        │    0 ┤────╯ ╰──────╯╰──          │
│      └──────────────────────        │      └──────────────────────       │
│       00   06   12   18   24        │       00   06   12   18   24       │
│  ── P50  ── P95  ── P99            │  ── 4xx  ── 5xx                    │
└──────────────────────────────────────┴───────────────────────────────────┘

┌─ 최근 에러 상세 ───────────────────────────────────────────────────────┐
│ 시간      상태  메서드  경로                  에러                       │
│ 14:20    500   POST   /api/v1/rag/query    TimeoutError               │
│ 12:45    502   GET    /api/v1/search       ConnectionRefused          │
│ 11:30    500   POST   /api/v1/documents    IntegrityError             │
└────────────────────────────────────────────────────────────────────────┘
```

---

### 4-2. 모니터링 메트릭 API 확장

기존 API로 부족한 경우 아래 API를 추가:

```python
# GET /api/v1/admin/monitoring/response-times?period=24h
# 시간대별 API 응답 시간 집계

Response 200:
{
  "period": "24h",
  "interval": "1h",
  "data": [
    { "timestamp": "2026-04-13T00:00:00Z", "p50": 45, "p95": 180, "p99": 450 },
    { "timestamp": "2026-04-13T01:00:00Z", "p50": 42, "p95": 160, "p99": 380 },
    ...
  ]
}

# GET /api/v1/admin/monitoring/error-trends?period=24h
# 시간대별 에러 발생 수 집계

Response 200:
{
  "period": "24h",
  "interval": "1h",
  "data": [
    { "timestamp": "2026-04-13T00:00:00Z", "4xx": 3, "5xx": 0 },
    ...
  ]
}
```

---

### 4-3. 차트 컴포넌트

```typescript
// Recharts 사용

import { LineChart, Line, XAxis, YAxis, Tooltip, Legend, ResponsiveContainer } from "recharts";

// 응답 시간 차트
<ResponsiveContainer width="100%" height={300}>
  <LineChart data={responseTimeData}>
    <XAxis dataKey="timestamp" tickFormatter={formatHour} />
    <YAxis unit="ms" />
    <Tooltip />
    <Legend />
    <Line dataKey="p50" stroke="#22c55e" name="P50" />
    <Line dataKey="p95" stroke="#f59e0b" name="P95" />
    <Line dataKey="p99" stroke="#ef4444" name="P99" />
  </LineChart>
</ResponsiveContainer>
```

---

### 4-4. 자동 갱신

```typescript
// 30초 간격 자동 갱신 (SWR 또는 setInterval)

const { data, mutate } = useSWR("/admin/monitoring/...", fetcher, {
  refreshInterval: 30000,  // 30초
});

// 사용자가 갱신 주기 변경 가능: 10초 / 30초 / 1분 / 수동
```

---

## 5. 산출물

1. `/admin/monitoring` UI 페이지
2. 시스템 상태 상세 카드 컴포넌트
3. 응답 시간 차트 컴포넌트
4. 에러 추이 차트 컴포넌트
5. 최근 에러 상세 테이블
6. 모니터링 API 확장 (response-times, error-trends) — 필요 시

---

## 6. 완료 기준

- 시스템 구성 요소별 상태와 응답 시간이 표시된다
- 에러 추이가 시간대별 라인 차트로 표시된다
- API 응답 시간(P50/P95/P99)이 라인 차트로 표시된다
- 최근 에러 목록이 상세 정보와 함께 표시된다
- 30초 간격 자동 갱신이 동작한다
- 데스크탑/모바일 반응형이 동작한다
- UI 디자인 리뷰를 최소 5회 수행한다

---

## 7. Codex 작업 지침

- Recharts 라이브러리를 사용한다 (프로젝트 기존 의존성 확인)
- 차트 데이터가 없는 경우 "데이터 수집 중" 안내를 표시한다
- 시간축 레이블은 사용자 로컬 시간대로 표시한다
- 차트 라인 색상은 일관된 색상 팔레트를 사용한다
- 자동 갱신 중 에러 발생 시 이전 데이터를 유지하고 에러 배너만 표시한다
