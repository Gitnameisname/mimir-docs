# Task 14-13 보안 취약점 검사 보고서 — 알림 관리 시스템

작성일: 2026-04-14
작성자: Mimir Team
대상 범위: Phase 14-13 (알림 규칙/이력 CRUD, 평가기, 알림 채널)

---

## 1. 검사 개요

OWASP Top 10 및 Mimir 내부 보안 원칙(CLAUDE.md §2~3)에 기반하여 본 Phase 에서 도입·수정된 파일을 정적 검사로 점검했다. 외부 의존성은 추가되지 않았으며(`urllib`/`ssl` stdlib 사용), 검증 스크립트의 SEC 영역 13개 항목 모두 PASS 했다.

| ID | 위협 | 결과 |
|---|---|---|
| SEC-01 | 코드 실행 (eval/exec) | ✅ 금지 (AST 스캔 확인) |
| SEC-02 | 입력 검증 우회 (metric) | ✅ 화이트리스트 + 422 |
| SEC-03 | 입력 검증 우회 (operator) | ✅ 화이트리스트 + 422 |
| SEC-04 | 입력 검증 우회 (channel) | ✅ 화이트리스트 + 422 |
| SEC-05 | 입력 검증 우회 (severity) | ✅ 화이트리스트 + 422 |
| SEC-06 | 경로 파라미터 주입 | ✅ UUID 정규식 2중 검증 |
| SEC-07 | SSRF (웹훅) | ✅ 스킴+호스트 화이트리스트 |
| SEC-08 | SQL Injection | ✅ 파라미터 바인딩 (`%s`) |
| SEC-09 | Stored XSS (JSONB) | ✅ `json.dumps` 경유 |
| SEC-10 | 권한 회피 | ✅ require_admin_access RBAC |
| SEC-11 | DoS (웹훅 행잉) | ✅ 10 초 타임아웃 |
| SEC-12 | DoS (이메일 대량) | ✅ 수신자 10명 상한 |
| SEC-13 | URL 파라미터 인코딩 | ✅ encodeURIComponent |

---

## 2. 주요 취약점 대응

### 2.1 임의 코드 실행 (Arbitrary Code Execution)

- 알림 조건(`condition.operator`)을 문자열로 받아 연산자를 결정해야 하므로 Python `eval()` 을 사용하고 싶은 유혹이 존재. 이를 원천 차단하기 위해 `operator.gt/ge/lt/le/eq/ne` 을 dict 로 매핑한 화이트리스트(`_OPS`)만 허용.
- `ast.walk` 로 `alert_evaluator.py`, `alert_repository.py`, `alert_notifier.py` 의 실제 호출 노드를 검사하여 `eval/exec Call` 노드 0개 확인.

### 2.2 SSRF (Server-Side Request Forgery)

- 웹훅 URL 입력을 받을 때 악성 사용자가 `http://169.254.169.254/latest/meta-data/` (AWS IMDS) 또는 `http://localhost:5432` (내부 DB) 를 지정해 내부망 접근을 유도할 수 있다.
- 대응
  - 허용 스킴 `{http, https}` 만 통과 (`file://`, `gopher://`, `ftp://` 차단)
  - 내부 호스트 접두어 6종 차단 (`localhost`, `127.`, `0.`, `169.254.`, `10.`, `172.`, `192.168.`)
  - `_is_safe_webhook_url()` 진입 검증
  - 프론트 모달에도 "내부/루프백 주소는 차단됩니다 (SSRF 방어)" 고지 (사용자 오해 방지)
- 참고: `172.` 는 CGNAT 등 공개망에 속하는 일부 블록도 포함하므로 향후 RFC1918 정확한 판정(`172.16.0.0/12`)으로 정교화 여지 있음 — 현 단계는 보수적 차단을 우선.

### 2.3 SQL Injection

- 모든 CRUD/history SQL 은 `%s` 파라미터 바인딩 사용 (15개 이상의 `%s` 플레이스홀더 확인).
- `update_rule()` 의 `UPDATE ... SET` 부분은 f-string 을 쓰지만 **컬럼명 화이트리스트(allowed set)** 통과 값만 포함, 사용자 값은 `params` 튜플로만 전달.
- JSONB 저장 시 `%s::jsonb` 캐스트 + `json.dumps(...)` — 직렬화 단계에서 실행 가능한 payload 차단.

### 2.4 Cross-Site Request 방어 / 권한 회피

- 전 라우트(9개)에 `Depends(require_admin_access)` 가드 적용.
- `require_admin_access` 는 HTTP 메서드별로 `admin.read`(GET) / `admin.write`(POST/PATCH/DELETE) 권한을 분기 → SUPER_ADMIN 만 규칙 생성/수정/삭제/평가/확인 수행.
- 프론트 `canEdit = hasRole("SUPER_ADMIN")` 체크와 백엔드 RBAC 가 2중 방어.

### 2.5 입력 검증 (Input Validation)

- Pydantic v2 `BaseModel` + `Field(max_length=...)` 로 길이 상한 강제 (name 255, description 2000, metric 255).
- 화이트리스트 세트 4종 (`_ALERT_SEVERITIES / _ALERT_CHANNELS / _ALERT_STATUSES / _ALERT_OPERATORS`) 으로 enum-like 값 강제.
- metric 는 evaluator 의 `_METRIC_LABELS` dict 를 직접 참조 — 실제 조회 가능한 값만 허용 (오탐 영역 없음).
- 422 반환으로 FastAPI 전역 예외 핸들러와 일관된 오류 포맷 유지.

### 2.6 DoS (Denial of Service)

- 웹훅은 `urllib.request.urlopen(..., timeout=10)` 으로 연결/읽기 행잉 차단.
- 이메일은 `recipients[:10]` — 단일 알림이 수십~수백 명에게 브로드캐스트되어 메일 큐를 범람시키는 것을 방지. UI/백엔드 공동 상한.
- 이력 페이지네이션 `page_size: Query(ge=1, le=500)` — 과도한 결과 요청 차단.

### 2.7 민감정보 노출

- alert_rules 의 `channel_config` 는 수신자 이메일·웹훅 URL 을 포함 → `admin.read` 권한자만 조회 가능.
- audit 이벤트 metadata 는 `name`/`severity`/`changed_fields` 만 기록 — 실제 수신자/웹훅 URL 은 포함하지 않음.
- 프론트 에러 메시지는 서버 예외 원본을 노출하지 않음(상한 검증 + 사용자 친화 메시지).

### 2.8 의존성

- 외부 HTTP 클라이언트(`httpx`, `requests`)를 추가하지 않고 stdlib `urllib.request` 사용 → Phase 14-13 으로 인한 신규 외부 의존성 0건 (CLAUDE.md §2 "critical 취약점 라이브러리 회피" 원칙 준수).
- frontend 역시 recharts/chart.js 등 신규 의존성 없이 기존 Tailwind + 내장 Modal 만 사용.

---

## 3. 잔여 리스크 / 모니터링 제언

1. **172.x 세분화**: 현재 `172.` 전체 차단은 과차단. 장기적으로 `172.16.0.0/12` 정확 매칭 + IPv6 `::1`, `fc00::/7`, `fe80::/10` 추가 차단 권고.
2. **DNS Rebinding**: URL hostname 이 공개 IP 로 해석되었다가 내부 IP 로 재해석되는 TOCTOU 공격. `urllib` 단에서 직접 방어 불가 — 향후 HTTP proxy 경유 또는 `resolve → bind` 분리로 대응 가능.
3. **웹훅 응답 본문 Exfiltration**: 현재 응답 본문 내용을 로그/저장하지 않아 데이터 반출 경로 없음. 미래 확장 시 본문 저장 피하고 상태코드만 기록하도록 유지.
4. **스케줄러 연동 후 Rate Limiting**: 1분 간격 평가가 도입되면 웹훅/이메일 발송 Rate Limit 이 필요. 규칙별/채널별 쿨다운 카운터를 `alert_history` 의 `acknowledged_at`/`resolved_at` 과 연동해 차후 구현 권고.

---

## 4. 결론

Phase 14-13 에서 도입된 코드는 CLAUDE.md §2/§3 보안 규칙, OWASP Top 10 (A01/A03/A05/A08/A10) 주요 항목을 모두 준수한다. 정적 검증 103개 항목 전체 PASS, 신규 외부 의존성 0건, 알려진 CVE 영향 없음. 본 구현을 Phase 14 의 다음 단계(14-14)로 진행 가능.
