# Task 14-2. 비밀번호 인증 구현

## 1. 작업 목적

이메일/비밀번호 기반의 자체 인증 시스템을 구현한다.

이 작업의 목표는 다음과 같다.

- 회원가입 API 구현 (이메일/비밀번호, bcrypt 해싱)
- 로그인 API 구현 (비밀번호 검증, 로그인 시도 제한)
- 사용자 테이블 확장 (password_hash, auth_provider 등)
- 비밀번호 복잡도 검증 로직 구현
- 로그인 시도 횟수 제한 (Brute Force 방어)

---

## 2. 작업 범위

### 포함 범위

- `users` 테이블 ALTER (password_hash, auth_provider, failed_login_count, locked_until 등)
- 회원가입 엔드포인트 (`POST /api/v1/auth/register`)
- 로그인 엔드포인트 (`POST /api/v1/auth/login`)
- bcrypt 해싱 유틸리티
- 비밀번호 복잡도 검증
- Valkey 기반 로그인 시도 제한

### 제외 범위

- JWT 토큰 발급 (Task 14-3에서 구현, 이 Task에서는 인증 성공 여부만 반환)
- OAuth 로그인 (Task 14-4에서 구현)
- 이메일 인증/비밀번호 재설정 (Task 14-5에서 구현)

---

## 3. 선행 조건

- Task 14-1 인증 아키텍처 설계 완료
- `users` 테이블 구조 이해 (현재: id, email, display_name, status, role_name, last_login_at)
- `users_repository.py` CRUD 메서드 이해
- Valkey 연결 (`session.py`) 이해

---

## 4. 주요 구현 대상

### 4-1. users 테이블 확장

```sql
ALTER TABLE users
  ADD COLUMN password_hash      VARCHAR(255),
  ADD COLUMN auth_provider      VARCHAR(50) DEFAULT 'local',
  ADD COLUMN email_verified     BOOLEAN DEFAULT FALSE,
  ADD COLUMN email_verified_at  TIMESTAMPTZ,
  ADD COLUMN failed_login_count INTEGER DEFAULT 0,
  ADD COLUMN locked_until       TIMESTAMPTZ,
  ADD COLUMN avatar_url         VARCHAR(500);
```

`connection.py`의 `_ensure_schema()` DDL도 함께 업데이트.

---

### 4-2. 비밀번호 해싱 유틸리티

```python
# app/api/auth/password.py

from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto", bcrypt__rounds=12)

def hash_password(plain: str) -> str:
    return pwd_context.hash(plain)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

---

### 4-3. 비밀번호 복잡도 검증

```python
# app/api/auth/validators.py

import re

def validate_password_strength(password: str) -> list[str]:
    errors = []
    if len(password) < 8:
        errors.append("비밀번호는 최소 8자 이상이어야 합니다")

    categories = 0
    if re.search(r"[a-zA-Z]", password):
        categories += 1
    if re.search(r"[0-9]", password):
        categories += 1
    if re.search(r"[!@#$%^&*(),.?\":{}|<>]", password):
        categories += 1

    if categories < 2:
        errors.append("영문, 숫자, 특수문자 중 2종 이상 포함해야 합니다")

    return errors
```

---

### 4-4. 로그인 시도 제한 (Brute Force 방어)

```python
# app/api/auth/rate_limit.py
# Valkey 기반 로그인 시도 횟수 관리

LOGIN_ATTEMPT_PREFIX = "login_attempts:"
MAX_ATTEMPTS = 5
LOCKOUT_SECONDS = 900  # 15분

async def check_login_allowed(valkey, email: str) -> bool:
    """로그인 가능 여부 확인 (잠금 상태 검사)"""
    key = f"{LOGIN_ATTEMPT_PREFIX}{email}"
    count = await valkey.get(key)
    return count is None or int(count) < MAX_ATTEMPTS

async def record_failed_attempt(valkey, email: str) -> int:
    """실패 횟수 증가, TTL 설정"""
    key = f"{LOGIN_ATTEMPT_PREFIX}{email}"
    count = await valkey.incr(key)
    if count == 1:
        await valkey.expire(key, LOCKOUT_SECONDS)
    return count

async def clear_attempts(valkey, email: str):
    """로그인 성공 시 카운터 초기화"""
    await valkey.delete(f"{LOGIN_ATTEMPT_PREFIX}{email}")
```

---

### 4-5. 회원가입 API

```python
# POST /api/v1/auth/register

Request Body:
{
  "email": "user@example.com",
  "password": "SecureP@ss1",
  "display_name": "홍길동"
}

처리 흐름:
1. 이메일 중복 검사 (users_repository.get_by_email)
2. 비밀번호 복잡도 검증
3. bcrypt 해싱
4. users 레코드 생성 (status=ACTIVE, role_name=VIEWER, auth_provider=local)
5. 감사 이벤트 기록

Response 201:
{
  "id": "uuid",
  "email": "user@example.com",
  "display_name": "홍길동",
  "status": "ACTIVE"
}

에러:
  409: 이메일 중복
  422: 비밀번호 복잡도 미충족
```

---

### 4-6. 로그인 API

```python
# POST /api/v1/auth/login

Request Body:
{
  "email": "user@example.com",
  "password": "SecureP@ss1"
}

처리 흐름:
1. 로그인 시도 제한 확인 (Valkey)
2. 이메일로 사용자 조회
3. 계정 상태 확인 (ACTIVE만 로그인 허용, SUSPENDED/INACTIVE 거부)
4. locked_until 확인 (잠금 상태면 거부)
5. bcrypt 비밀번호 검증
6. 실패 시: failed_login_count 증가, 감사 이벤트 기록
7. 성공 시: failed_login_count 초기화, last_login_at 갱신, 감사 이벤트 기록
8. (토큰 발급은 Task 14-3에서 통합)

에러:
  401: 이메일 또는 비밀번호 불일치 (구체적 원인 미노출)
  403: 계정 비활성 또는 정지
  429: 로그인 시도 초과 (잠금)
```

---

## 5. 산출물

1. `users` 테이블 ALTER 마이그레이션
2. `app/api/auth/password.py` — bcrypt 해싱 유틸리티
3. `app/api/auth/validators.py` — 비밀번호 복잡도 검증
4. `app/api/auth/rate_limit.py` — Valkey 기반 로그인 시도 제한
5. `POST /api/v1/auth/register` 엔드포인트
6. `POST /api/v1/auth/login` 엔드포인트 (토큰 발급 전 단계)
7. 단위 테스트 (비밀번호 해싱, 복잡도 검증, 시도 제한)

---

## 6. 완료 기준

- 회원가입 시 bcrypt(cost 12)로 비밀번호가 해싱 저장된다
- 이메일 중복 가입이 409 에러로 거부된다
- 비밀번호 복잡도 미충족 시 422 에러가 반환된다
- 로그인 5회 연속 실패 시 15분간 해당 이메일이 잠금된다
- 로그인 성공 시 failed_login_count가 0으로 초기화된다
- 비활성/정지 계정은 로그인이 거부된다
- 로그인 실패 시 "이메일 또는 비밀번호가 올바르지 않습니다"만 반환한다 (정보 누출 방지)

---

## 7. Codex 작업 지침

- `passlib[bcrypt]`를 `requirements.txt`에 추가한다
- 비밀번호 평문은 로그에 절대 기록하지 않는다 (보안 최우선)
- 로그인 실패 응답에 "이메일이 존재하지 않습니다" 등 구체적 원인을 노출하지 않는다
- `users_repository.py`의 기존 CRUD 메서드를 확장하여 새 컬럼을 처리한다
- Valkey 키에 TTL을 반드시 설정하여 메모리 누수를 방지한다
- 감사 이벤트는 기존 `audit_events` 테이블에 기록한다
