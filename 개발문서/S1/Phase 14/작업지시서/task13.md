# Task 14-13. 알림 관리 시스템 구축

## 1. 작업 목적

시스템 이상 감지 시 관리자에게 알림을 발생시키는 알림 관리 시스템을 구축한다.

이 작업의 목표는 다음과 같다.

- `alert_rules`, `alert_history` 테이블 생성
- 알림 규칙 CRUD API 구현
- 알림 이력 조회 API 구현
- 알림 채널 설정 (이메일, 웹훅)
- 알림 관리 UI 구현
- 알림 평가 로직 (메트릭 기반 조건 판단)

---

## 2. 작업 범위

### 포함 범위

- `alert_rules` 테이블 생성
- `alert_history` 테이블 생성
- 알림 규칙 CRUD API (`/admin/alerts/rules/*`)
- 알림 이력 조회 API (`/admin/alerts/history`)
- 알림 확인(acknowledge) API
- 알림 평가 서비스 (주기적 메트릭 확인)
- 알림 채널: 이메일 발송, 웹훅 호출 (설정만)
- `/admin/alerts` UI 페이지

### 제외 범위

- Slack/PagerDuty 등 외부 서비스 실제 연동 (웹훅 URL 설정만 제공)
- 복합 조건 알림 (AND/OR 조합 — 향후 확장)

---

## 3. 선행 조건

- Task 14-9 Admin 레이아웃 완료
- Task 14-11 시스템 설정 완료 (알림 설정 참조)
- Phase 13 모니터링 체계 (메트릭 수집) 이해

---

## 4. 주요 구현 대상

### 4-1. alert_rules / alert_history 테이블

```sql
CREATE TABLE alert_rules (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name         VARCHAR(255) NOT NULL,
  description  TEXT,
  metric_name  VARCHAR(255) NOT NULL,
  condition    JSONB NOT NULL,
  severity     VARCHAR(50) NOT NULL,
  channels     JSONB DEFAULT '[]',
  enabled      BOOLEAN DEFAULT TRUE,
  created_by   UUID REFERENCES users(id),
  created_at   TIMESTAMPTZ DEFAULT now(),
  updated_at   TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE alert_history (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  rule_id           UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
  triggered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at       TIMESTAMPTZ,
  status            VARCHAR(50) NOT NULL,
  metric_value      NUMERIC,
  message           TEXT,
  notified_channels JSONB DEFAULT '[]'
);

CREATE INDEX idx_alert_history_rule   ON alert_history(rule_id);
CREATE INDEX idx_alert_history_status ON alert_history(status, triggered_at DESC);
```

---

### 4-2. 알림 조건(condition) 스키마

```json
{
  "operator": "gt",          // gt, gte, lt, lte, eq, ne
  "threshold": 500,          // 임계값
  "duration_seconds": 300    // 조건 지속 시간 (선택)
}
```

지원 메트릭 예시:

| metric_name | 설명 | 단위 |
|------------|------|------|
| `api.response_time_p95` | API 응답 시간 P95 | ms |
| `api.error_rate_5xx` | 5xx 에러율 | % |
| `db.connection_pool_usage` | DB 커넥션 풀 사용률 | % |
| `valkey.memory_usage` | Valkey 메모리 사용률 | % |
| `job.queue_length` | 작업 대기열 길이 | 건 |
| `search.index_lag` | 검색 인덱스 지연 | 초 |

---

### 4-3. 알림 평가 서비스

```python
# app/services/alert_evaluator.py

class AlertEvaluator:
    async def evaluate_all(self):
        """모든 활성 규칙을 순회하며 조건 판단"""
        rules = await self.repo.list_enabled_rules()
        for rule in rules:
            current_value = await self.get_metric_value(rule.metric_name)
            if self._check_condition(current_value, rule.condition):
                await self._fire_alert(rule, current_value)
            else:
                await self._resolve_if_firing(rule)

    def _check_condition(self, value, condition):
        ops = {"gt": ">", "gte": ">=", "lt": "<", "lte": "<=", "eq": "==", "ne": "!="}
        return eval(f"{value} {ops[condition['operator']]} {condition['threshold']}")

    async def _fire_alert(self, rule, value):
        # 이미 firing 상태면 무시
        # 신규 → alert_history INSERT + 알림 채널 발송
        ...

    async def _resolve_if_firing(self, rule):
        # firing → resolved 상태 전환
        ...
```

평가 주기: 1분 간격 (백그라운드 Job으로 실행).

---

### 4-4. 알림 채널 발송

```python
# app/services/alert_notifier.py

class AlertNotifier:
    async def notify(self, rule, alert_event):
        for channel in rule.channels:
            if channel == "email":
                await self._send_email(rule, alert_event)
            elif channel == "webhook":
                await self._send_webhook(rule, alert_event)

    async def _send_email(self, rule, event):
        # EmailService 활용 (Task 14-5)
        subject = f"[Mimir Alert] [{rule.severity}] {rule.name}"
        body = f"메트릭: {rule.metric_name}\n현재 값: {event.metric_value}\n시간: {event.triggered_at}"
        await email_service.send_email(admin_emails, subject, body)

    async def _send_webhook(self, rule, event):
        # 설정된 웹훅 URL로 POST
        payload = {"rule": rule.name, "severity": rule.severity, ...}
        async with httpx.AsyncClient() as client:
            await client.post(webhook_url, json=payload, timeout=10)
```

---

### 4-5. 알림 관리 UI

```
알림 관리

[탭: 규칙 | 이력]

── 규칙 탭 ──
[+ 규칙 추가]

┌──────────┬──────────────┬───────┬────────┬──────┬────────┐
│ 이름      │ 메트릭        │ 조건   │ 심각도  │ 활성  │ 작업   │
├──────────┼──────────────┼───────┼────────┼──────┼────────┤
│ API 지연  │ response_p95 │ > 500 │ warning│  ✓   │ [편집] │
│ 에러율    │ error_5xx    │ > 1%  │critical│  ✓   │ [편집] │
│ DB 풀    │ db_pool      │ > 80% │ warning│  ✓   │ [편집] │
└──────────┴──────────────┴───────┴────────┴──────┴────────┘

── 이력 탭 ──
[필터: 상태 ▼]  [필터: 심각도 ▼]  [기간: 최근 7일 ▼]

┌──────────┬──────────┬────────┬──────────┬────────┬────────┐
│ 시간      │ 규칙 이름 │ 심각도  │ 메트릭 값 │ 상태   │ 작업   │
├──────────┼──────────┼────────┼──────────┼────────┼────────┤
│ 14:20    │ API 지연  │ warning│ 650ms   │ firing │ [확인] │
│ 12:45    │ 에러율    │critical│ 2.3%    │resolved│        │
└──────────┴──────────┴────────┴──────────┴────────┴────────┘
```

---

## 5. 산출물

1. `alert_rules`, `alert_history` 테이블 DDL
2. `app/repositories/alert_repository.py`
3. `app/services/alert_evaluator.py` — 알림 평가
4. `app/services/alert_notifier.py` — 알림 발송
5. 알림 규칙 CRUD API + 이력 API
6. `/admin/alerts` UI 페이지 (규칙 + 이력 탭)
7. 알림 평가 백그라운드 Job 등록

---

## 6. 완료 기준

- 알림 규칙 CRUD가 API와 UI에서 동작한다
- 메트릭이 조건을 충족하면 알림이 자동 발생한다
- 조건이 해소되면 알림이 자동 resolved 상태로 전환된다
- 알림 이력이 시간순으로 조회된다
- 이메일 알림이 발송된다
- 웹훅 URL이 설정되고 호출된다
- UI 디자인 리뷰를 최소 5회 수행한다

---

## 7. Codex 작업 지침

- 알림 평가는 `eval()`을 사용하지 않고 안전한 비교 함수를 구현한다
- 웹훅 호출 시 타임아웃을 10초로 제한하고 실패를 graceful하게 처리한다
- 알림 평가 주기(1분)는 설정 가능하게 한다
- 동일 규칙의 중복 알림을 방지한다 (이미 firing이면 재발생하지 않음)
- 알림 채널 종류는 향후 확장을 위해 enum이 아닌 문자열로 관리한다
