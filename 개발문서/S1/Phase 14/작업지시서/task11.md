# Task 14-11. 시스템 설정 관리 API 및 UI

## 1. 작업 목적

관리자가 Mimir 플랫폼의 시스템 설정을 Admin UI에서 조회·변경할 수 있는 설정 관리 시스템을 구현한다.

이 작업의 목표는 다음과 같다.

- `system_settings` 테이블 생성
- 시스템 설정 CRUD API 구현
- 카테고리별 설정 관리 UI
- 설정 변경 시 감사 로그 기록
- DocumentType 설정 관리 연동

---

## 2. 작업 범위

### 포함 범위

- `system_settings` 테이블 생성
- `GET /api/v1/admin/settings` — 전체 설정 조회
- `GET /api/v1/admin/settings/{category}` — 카테고리별 설정 조회
- `PATCH /api/v1/admin/settings/{category}/{key}` — 설정 변경
- `/admin/settings` UI 페이지
- 초기 설정 시드 데이터
- 설정 변경 감사 이벤트

### 제외 범위

- 설정 변경의 실시간 반영 (서버 재시작 또는 캐시 TTL 경과 후 반영)
- 환경 변수 기반 설정 오버라이드 (기존 `config.py` 유지)

---

## 3. 선행 조건

- Task 14-9 Admin 레이아웃 완료
- 기존 `config.py` 설정 구조 이해
- Phase 12 `document_type_configs` 테이블 이해

---

## 4. 주요 구현 대상

### 4-1. system_settings 테이블

```sql
CREATE TABLE system_settings (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  category    VARCHAR(100) NOT NULL,
  key         VARCHAR(255) NOT NULL,
  value       JSONB NOT NULL,
  description TEXT,
  updated_by  UUID REFERENCES users(id),
  updated_at  TIMESTAMPTZ DEFAULT now(),
  UNIQUE (category, key)
);

CREATE INDEX idx_system_settings_category ON system_settings(category);
```

---

### 4-2. 초기 시드 데이터

```python
INITIAL_SETTINGS = [
    # 인증 설정
    {"category": "auth", "key": "session_timeout_minutes", "value": 120,
     "description": "세션 타임아웃 (분)"},
    {"category": "auth", "key": "max_login_attempts", "value": 5,
     "description": "최대 로그인 시도 횟수"},
    {"category": "auth", "key": "lockout_duration_minutes", "value": 15,
     "description": "로그인 잠금 시간 (분)"},
    {"category": "auth", "key": "password_min_length", "value": 8,
     "description": "최소 비밀번호 길이"},
    {"category": "auth", "key": "auto_create_gitlab_users", "value": True,
     "description": "GitLab 최초 로그인 시 자동 계정 생성"},

    # 시스템 설정
    {"category": "system", "key": "platform_name", "value": "Mimir",
     "description": "플랫폼 표시 이름"},
    {"category": "system", "key": "default_user_role", "value": "VIEWER",
     "description": "신규 사용자 기본 역할"},
    {"category": "system", "key": "maintenance_mode", "value": False,
     "description": "유지보수 모드 활성화"},

    # 알림 설정
    {"category": "notification", "key": "email_enabled", "value": True,
     "description": "이메일 알림 활성화"},
    {"category": "notification", "key": "webhook_enabled", "value": False,
     "description": "웹훅 알림 활성화"},

    # 보안 설정
    {"category": "security", "key": "api_rate_limit_per_minute", "value": 100,
     "description": "API 분당 요청 제한"},
    {"category": "security", "key": "require_email_verification", "value": False,
     "description": "이메일 인증 필수 여부"},
]
```

---

### 4-3. Settings API

```python
# GET /api/v1/admin/settings
# 전체 설정을 카테고리별 그룹으로 반환

Response 200:
{
  "auth": [
    { "key": "session_timeout_minutes", "value": 120, "description": "세션 타임아웃 (분)", "updated_at": "..." },
    ...
  ],
  "system": [...],
  "notification": [...],
  "security": [...]
}

# PATCH /api/v1/admin/settings/{category}/{key}
Request Body: { "value": 10 }

처리:
1. 카테고리/키 존재 확인
2. 값 타입 검증 (기존 값과 동일 타입)
3. 업데이트 + updated_by 기록
4. 감사 이벤트: 이전 값 → 새 값 기록

Response 200: { "key": "max_login_attempts", "value": 10, "updated_at": "..." }
```

---

### 4-4. 설정 관리 UI

```
시스템 설정

[탭: 인증 | 시스템 | 알림 | 보안]

인증 설정
┌────────────────────┬──────┬────────────────────────────┐
│ 설정 항목            │ 현재 값│ 설명                       │
├────────────────────┼──────┼────────────────────────────┤
│ 세션 타임아웃 (분)    │ [120]│ 비활동 시 세션 만료 시간      │
│ 최대 로그인 시도      │ [ 5 ]│ 초과 시 잠금                │
│ 잠금 시간 (분)       │ [15] │ 로그인 시도 초과 후 잠금 시간  │
│ 최소 비밀번호 길이    │ [ 8 ]│ 회원가입 비밀번호 최소 길이    │
│ GitLab 자동 가입     │ [✓]  │ GitLab 첫 로그인 시 자동 생성 │
└────────────────────┴──────┴────────────────────────────┘
                                               [저장]

변경 시 확인 다이얼로그: "설정을 변경하시겠습니까?"
성공 토스트: "설정이 저장되었습니다"
```

---

## 5. 산출물

1. `system_settings` 테이블 DDL + 시드 데이터
2. `app/repositories/settings_repository.py`
3. Settings API 엔드포인트 3종
4. `/admin/settings` UI 페이지
5. 설정 변경 감사 이벤트 로깅

---

## 6. 완료 기준

- 시스템 설정이 카테고리별로 조회된다
- 관리자가 UI에서 설정 값을 변경할 수 있다
- 설정 변경이 감사 로그에 이전 값/새 값과 함께 기록된다
- 잘못된 타입의 값을 입력하면 에러가 반환된다
- SUPER_ADMIN만 설정을 변경할 수 있다

---

## 7. Codex 작업 지침

- 설정 값은 JSONB로 저장하되 UI에서는 타입에 맞는 입력 컴포넌트를 표시한다 (숫자→number input, boolean→toggle)
- `config.py`의 환경 변수 설정은 `system_settings` DB 설정보다 우선한다
- 설정 캐싱은 Valkey에 5분 TTL로 관리한다
- 유지보수 모드(`maintenance_mode`) 활성화 시 Admin 외 API가 503을 반환하도록 향후 미들웨어에서 참조할 수 있게 설계한다
