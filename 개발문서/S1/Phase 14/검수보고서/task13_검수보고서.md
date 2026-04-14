# Task 14-13 검수 보고서 — 알림 관리 시스템

작성일: 2026-04-14
작성자: Mimir Team

---

## 1. 요약

Phase 14-13 (알림 관리 시스템)의 백엔드·프론트엔드 구현과 UI 리뷰 5회, 그리고 정적 검증 스크립트(103개 항목) 전체 PASS 를 달성했다. 알림 규칙 CRUD, 이력 조회/확인, 수동 평가 API 및 `/admin/alerts` 관리 화면(규칙/이력 탭)이 연결되었으며, 권한 분리 (SUPER_ADMIN), RBAC, SSRF 방어, `eval()`-free 조건 평가 등 CLAUDE.md 보안 원칙을 전부 준수한다.

| 영역 | 결과 |
|---|---|
| DDL (alert_rules / alert_history) | 8/8 PASS |
| Repository (CRUD / history 전이) | 9/9 PASS |
| Evaluator (연산자·메트릭 화이트리스트) | 11/11 PASS |
| Notifier (이메일 / 웹훅 SSRF) | 11/11 PASS |
| API (9 라우트, 권한·검증) | 18/18 PASS |
| 프론트 타입 / API 클라이언트 / 라우팅 | 5/5 + 2/2 + 3/3 |
| UI (탭·모달·스위치·a11y) | 14/14 PASS |
| 반응형 (p-4 sm:p-6 등) | 6/6 PASS |
| 보안(SEC) / 구문(SYNTAX) | 13/13 + 3/3 |
| **합계** | **103/103 PASS** |

---

## 2. 구현 내용

### 2.1 DB 스키마 (backend/app/db/connection.py)

- `alert_rules(id UUID PK, name, description, metric_name, condition JSONB, severity, channels JSONB, channel_config JSONB, enabled, created_by FK, created_at, updated_at)`
- `alert_history(id UUID PK, rule_id FK ON DELETE CASCADE, triggered_at, resolved_at, acknowledged_at, acknowledged_by FK, status, metric_value NUMERIC, message, notified_channels JSONB)`
- 인덱스
  - `idx_alert_rules_enabled_metric (enabled, metric_name)` — 평가 루프 성능
  - `idx_alert_history_status (status, triggered_at DESC)` — 필터링
  - `idx_alert_history_rule (rule_id)` — FK 조인
  - `idx_alert_history_one_firing_per_rule ON (rule_id) WHERE status = 'firing'` — **firing 중복 방지(부분 Unique)**

### 2.2 Repository (backend/app/repositories/alert_repository.py)

- `list_rules / get_rule / create_rule / update_rule / delete_rule`
- `list_history / get_firing_history / insert_firing / resolve_firing / acknowledge`
- 업데이트 필드 화이트리스트(`allowed = {...}`) → 사용자 입력으로부터 SQL 컬럼 주입 방지
- JSONB 컬럼은 `%s::jsonb` 캐스트 + `json.dumps(...)`
- 모든 SQL 은 파라미터 바인딩(`%s`) 사용, f-string 은 화이트리스트 컬럼명에만 한정

### 2.3 Evaluator (backend/app/services/alert_evaluator.py)

- **`eval()` / `exec()` 절대 미사용** — `operator.gt/ge/lt/le/eq/ne` 을 dict 로 매핑한 화이트리스트로 분기
- 메트릭 또한 `_METRIC_LABELS` 화이트리스트로 6종 지원 (`api.response_time_p95`, `api.error_rate_5xx`, `db.connection_pool_usage`, `valkey.memory_usage`, `job.queue_length`, `search.index_lag`)
- firing 전이 시 `get_firing_history` 로 기존 firing 확인 → DB Unique 인덱스와 2중 방어
- 통계 반환: `{evaluated, fired, resolved, skipped}`
- 수동 평가는 POST `/admin/alerts/evaluate` — 추후 스케줄러(1분 간격) 연동용 훅 존재

### 2.4 Notifier (backend/app/services/alert_notifier.py)

- 채널: `email` / `webhook` (문자열 키 — 추후 slack/teams 확장 용이)
- **SSRF 방어**
  - 허용 스킴 화이트리스트 `{https, http}`
  - 내부/루프백 차단 `localhost / 127. / 0. / 169.254. / 10. / 172. / 192.168.`
  - `_is_safe_webhook_url()` 진입 검증
- **DoS 방어**
  - 웹훅 타임아웃 10 초 고정 (`_WEBHOOK_TIMEOUT_SECONDS = 10`)
  - 이메일 수신자 최대 10명 제한 (`recipients[:10]`)
- TLS: `ssl.create_default_context()` — 시스템 CA 체인 사용
- 외부 HTTP 라이브러리(httpx/requests) 미도입 — `urllib.request` 로 의존성 최소화 (CLAUDE.md §2)
- 발송 실패는 예외 격리 — 한 채널 실패가 다른 채널/전체 평가를 중단하지 않음

### 2.5 API (backend/app/api/v1/admin.py — Phase 14-13 섹션)

9개 라우트:

| 메서드 | 경로 | 권한 | 설명 |
|---|---|---|---|
| GET | /alerts/metrics | admin.read | 지원 메트릭 목록 |
| GET | /alerts/rules | admin.read | 규칙 목록 (enabled_only) |
| POST | /alerts/rules | admin.write | 규칙 생성 |
| GET | /alerts/rules/{id} | admin.read | 규칙 상세 |
| PATCH | /alerts/rules/{id} | admin.write | 규칙 부분 수정 |
| DELETE | /alerts/rules/{id} | admin.write | 규칙 삭제 |
| GET | /alerts/history | admin.read | 이력 조회(filter+pagination) |
| POST | /alerts/history/{id}/acknowledge | admin.write | 알림 확인 |
| POST | /alerts/evaluate | admin.write | 즉시 평가 (수동) |

- Pydantic v2 모델 `CreateAlertRuleBody` / `UpdateAlertRuleBody` / `AlertConditionBody`
- 공통 검증 `_validate_rule_payload()` — metric/operator/severity/channels 화이트리스트
- UUID 경로 파라미터는 `re.match(r"^[0-9a-f\-]{36}$")` 로 2중 검증
- audit emit: `ALERT_RULE_CREATED / ALERT_RULE_UPDATED / ALERT_RULE_DELETED / ALERT_ACKNOWLEDGED`

### 2.6 프론트엔드

- Types (`frontend/src/types/admin.ts`): AlertSeverity/Status/Operator/Channel 타입, AlertRule/Condition/HistoryItem/HistoryResponse 인터페이스
- Client (`frontend/src/lib/api/admin.ts`): 9개 `adminApi.*` 메소드 — 경로 파라미터는 `encodeURIComponent`
- Page (`frontend/src/features/admin/alerts/AdminAlertsPage.tsx`) ~735줄
  - `<AdminAlertsPage>` (탭 구조)
  - `<RulesTab>` (테이블, switch 토글, 편집/삭제, 즉시 평가)
  - `<RuleEditorModal>` (이름/설명/메트릭/연산자/임계값/심각도/채널/수신자/웹훅URL)
  - `<HistoryTab>` (상태·심각도 필터, firing/resolved 카운트 aria-live, 확인 버튼)
- 라우팅 `frontend/src/app/admin/alerts/page.tsx` — metadata + `<AdminAlertsPage />`

---

## 3. UI 리뷰 5회 (CLAUDE.md §4 준수)

| 라운드 | 포커스 | 개선 사항 |
|---|---|---|
| 1차 | 정보 아키텍처 | 규칙/이력 두 탭으로 분리 (단일 페이지 과밀 방지) |
| 2차 | 권한 시각화 | `SUPER_ADMIN` 이 아닌 경우 편집/삭제/추가 버튼 비가시화, 토글 비활성 |
| 3차 | 안전성 | 삭제는 Modal + destructive 확인, 웹훅 URL 필드에 SSRF 경고 문구 |
| 4차 | 접근성 | `role=tablist/tab/switch/status/alert`, `aria-selected/checked/live`, `sr-only` 필터 라벨, `focus-visible:ring` 10+ 포인트 |
| 5차 | 반응형 | `p-4 sm:p-6`, `text-xl sm:text-2xl`, `overflow-x-auto`, `flex-wrap`, 최소 터치 타깃 `min-h-[36px]` |

Severity 시각화는 블루(info) / 앰버(warning) / 레드(critical) 세 단계로 통일하였고, 이력 탭 상단 firing/resolved 카운트는 `aria-live="polite"` 로 스크린리더에 변경을 전달한다.

---

## 4. 검증 결과

```
============================================================
Phase 14-13 알림 관리 시스템 검증
============================================================

[API] 18/18    [API-FE] 2/2   [DDL] 8/8
[EVAL] 11/11   [NOTIFY] 11/11 [REPO] 9/9
[RESP] 6/6     [ROUTE] 3/3    [SEC] 13/13
[SYNTAX] 3/3   [TYPES] 5/5    [UI] 14/14

합계: 103/103 PASS  (0 FAIL)
```

검증 스크립트: `backend/tests/unit/test_admin_alerts_phase14_13.py`

---

## 5. 잔여 / 차후 과제

- 주기적 평가 스케줄러 연동(1분 간격) — 현재는 `POST /alerts/evaluate` 수동 트리거 + 코어 훅 제공. Task 14-14 ~ 14-16 에서 worker 등록 예정.
- Valkey 메모리 사용률 메트릭은 proxy 0 반환 — 실제 `INFO memory` 연결은 차후 Phase 에서 구현.
- 채널 확장(slack/teams)은 `_ALERT_CHANNELS` 화이트리스트 + Notifier 분기 추가만으로 가능하도록 설계됨.
