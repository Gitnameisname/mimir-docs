# Task 14-11 보안 취약점 검사 보고서

**작성일**: 2026-04-14
**검사 대상**: system_settings 테이블 + Settings API 3종 + AdminSettingsPage UI

---

## 1. 검사 항목 및 결과

### 1.1 접근 제어 (RBAC)

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 1 | 모든 settings 엔드포인트에 `require_admin_access` | ✅ 통과 | GET/PATCH 모두 의존성 적용 |
| 2 | PATCH는 `admin.write` (SUPER_ADMIN 전용) | ✅ 통과 | `require_admin_access`가 메서드별 분기 |
| 3 | GET은 `admin.read` (ORG_ADMIN 이상) | ✅ 통과 | 동일 |
| 4 | UI `canEdit` 가드는 보조 통제 | ✅ 통과 | 백엔드가 진짜 권한 검증 |
| 5 | `/admin/settings` 라우트 AdminLayout `AuthGuard` 상속 | ✅ 통과 | Phase 14-9 |

### 1.2 SQL Injection / Input Validation

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 6 | repository SQL은 `%s` 파라미터 바인딩 | ✅ 통과 | f-string 미사용, 모든 값은 params |
| 7 | path parameter `category` 정규식 검증 | ✅ 통과 | `^[a-z][a-z0-9_]{0,99}$` |
| 8 | path parameter `key` 정규식 검증 | ✅ 통과 | `^[a-z][a-z0-9_]{0,254}$` |
| 9 | JSONB value는 `json.dumps` 후 `%s::jsonb` 캐스트 | ✅ 통과 | 임의 객체도 안전하게 직렬화 |
| 10 | 프론트 URL path는 `encodeURIComponent` | ✅ 통과 | URL injection 방어 |

### 1.3 타입 검증 / 무결성

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 11 | 새 값과 기존 값의 타입 일치 강제 (422) | ✅ 통과 | `_type_signature` 비교 |
| 12 | bool은 int와 별도 시그니처 처리 | ✅ 통과 | `True`로 정수 필드 덮어쓰기 방지 |
| 13 | maintenance_mode(bool) → 숫자/문자 차단 | ✅ 통과 | 위 타입 검증으로 자동 보호 |
| 14 | 존재하지 않는 키 → 404 | ✅ 통과 | get_one None 분기 |

### 1.4 XSS / CSRF / 정보 노출

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 15 | UI에 `dangerouslySetInnerHTML` 미사용 | ✅ 통과 | grep 검증 |
| 16 | 사용자 입력은 React 기본 이스케이프 | ✅ 통과 | `{item.value}`, `{label}` 등 |
| 17 | PATCH는 Bearer 토큰 인증 (Phase 14 패턴) | ✅ 통과 | 변경 시 인증 필수 |
| 18 | 에러 메시지에 내부 정보 미노출 | ✅ 통과 | "찾을 수 없습니다" / "타입이 일치하지 않습니다" 등 메시지 한정 |

### 1.5 감사 로깅

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 19 | 변경 시 `SETTING_CHANGED` 이벤트 emit | ✅ 통과 | actor_id/role 포함 |
| 20 | metadata에 `old_value`, `new_value` 기록 | ✅ 통과 | 추적 가능 |
| 21 | `previous_state`/`new_state` 500자 truncate | ✅ 통과 | DoS 방어 (거대 JSON 차단) |
| 22 | 감사 실패가 응답을 막지 않음 | ✅ 통과 | try/except로 격리 |

### 1.6 캐시 / 동시성

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 23 | Valkey 캐시 TTL 5분 (300초) | ✅ 통과 | `_SETTINGS_CACHE_TTL` |
| 24 | PATCH 후 캐시 무효화 | ✅ 통과 | `_invalidate_settings_cache()` |
| 25 | Valkey 장애 시 DB 폴백 | ✅ 통과 | try/except, 무중단 |
| 26 | 캐시 페이로드에 민감 정보 없음 | ✅ 통과 | 설정 값만 (관리자 한정 정보) |

### 1.7 클라이언트 사이드 보안

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 27 | 설정 값을 localStorage에 저장 안 함 | ✅ 통과 | React Query in-memory |
| 28 | URL 파라미터에 민감 정보 없음 | ✅ 통과 | category/key는 enum-like |
| 29 | 외부 링크 / open redirect 없음 | ✅ 통과 | 내부 라우트만 |

### 1.8 접근성 보안

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 30 | 키보드로 모든 컨트롤 조작 가능 | ✅ 통과 | button/input + focus-visible 링 |
| 31 | 토글 `role="switch"` + `aria-checked` | ✅ 통과 | 스크린리더 상태 전달 |
| 32 | 변경 카운트 `aria-live=polite` | ✅ 통과 | 변경 감지 알림 |

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

**INFO-1: maintenance_mode 활성화 시 즉시 효과 미반영**
- 위치: `system_settings` → `system.maintenance_mode`
- 설명: 미들웨어 미구현 상태이므로 값을 켜도 비관리자 차단이 자동 적용되지 않음. 향후 `app/middleware/maintenance_guard.py`에서 settings 캐시 조회 후 503 반환 예정.
- 위험도: 정보 (운영 절차로 관리)
- 권고: Phase 14 후속 작업에서 미들웨어 추가.

**INFO-2: Valkey 캐시 TTL 5분 — 변경 즉시 반영 지연 가능**
- 위치: `_SETTINGS_CACHE_KEY_ALL`
- 설명: PATCH 직후 캐시는 무효화하지만, 멀티 인스턴스에서 다른 워커의 in-process 호출이 stale dict을 사용할 수 있음. 단 Valkey가 분산 캐시이므로 모든 워커가 동일 캐시 키를 공유 — 실제로는 즉시 일관.
- 위험도: 정보
- 권고: 현재 구조 적정.

**INFO-3: 설정 값 변경 이력 페이지 미제공**
- 설명: 감사 로그에는 변경이 기록되지만 settings 페이지에는 "최근 변경자/시각"만 표시. 향후 `audit_logs` 화면에서 `event_type=SETTING_CHANGED` 필터로 조회 가능 (Phase 14-7 완료).
- 위험도: 정보
- 권고: 현재 구조 적정.

---

## 3. 외부 라이브러리 취약점

Task 14-11에서 **신규 추가된 의존성 없음**. 기존 `psycopg2`, `redis-py`, `fastapi`, `pydantic`, `@tanstack/react-query`만 사용. CVE 검사 결과 critical/high 0건.

---

## 4. 결론

**Critical/High/Medium/Low 취약점 0건**. Settings API는 RBAC 가드(`admin.write` = SUPER_ADMIN) + path 정규식 검증 + 파라미터 바인딩 + 타입 일치 검증의 다층 통제를 갖춤. 변경은 감사 이벤트로 추적되며, 캐시는 PATCH 시 즉시 무효화. UI는 권한 없는 사용자에게 모든 입력을 disable 처리하고 보조 안내. Info 3건은 향후 작업 범위(미들웨어 / 변경 이력 뷰)에 해당.
