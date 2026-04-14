# Task 14-5. 비밀번호 재설정 및 이메일 인증

## 1. 작업 목적

비밀번호 분실 시 이메일 기반 재설정 플로우와 회원가입 시 이메일 인증 플로우를 구현한다.

이 작업의 목표는 다음과 같다.

- 비밀번호 재설정 요청 API (이메일 발송)
- 비밀번호 재설정 실행 API (토큰 검증 + 변경)
- 이메일 인증 API (가입 후 이메일 확인)
- SMTP 이메일 발송 서비스
- JWT 기반 1회용 인증 토큰 생성

---

## 2. 작업 범위

### 포함 범위

- `POST /api/v1/auth/forgot-password` — 재설정 이메일 발송
- `POST /api/v1/auth/reset-password` — 재설정 실행
- `POST /api/v1/auth/verify-email` — 이메일 인증 확인
- SMTP 이메일 발송 유틸리티
- JWT 기반 목적별 토큰 (password_reset, email_verify)
- 이메일 템플릿 (비밀번호 재설정, 이메일 인증)

### 제외 범위

- 이메일 발송 대기열 (async queue) — 동기 발송으로 시작
- HTML 이메일 고급 디자인 — 기본 텍스트 기반

---

## 3. 선행 조건

- Task 14-2 비밀번호 인증 구현 완료 (password_hash, bcrypt)
- Task 14-3 JWT 토큰 체계 완료 (토큰 생성 함수)
- SMTP 서버 접속 정보 확보
- `config.py`에 SMTP 설정 추가 (Task 14-1에서 정의)

---

## 4. 주요 구현 대상

### 4-1. 목적별 JWT 토큰

```python
# app/api/auth/purpose_tokens.py

def create_purpose_token(user_id: str, purpose: str, expire_minutes: int = 30) -> str:
    """목적별 1회용 토큰 (password_reset, email_verify)"""
    payload = {
        "sub": user_id,
        "purpose": purpose,
        "exp": datetime.utcnow() + timedelta(minutes=expire_minutes),
        "iat": datetime.utcnow(),
        "jti": str(uuid4()),
    }
    return jwt.encode(payload, settings.jwt_secret, algorithm="HS256")

def decode_purpose_token(token: str, expected_purpose: str) -> dict:
    """토큰 디코딩 + 목적 검증"""
    payload = jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])
    if payload.get("purpose") != expected_purpose:
        raise ValueError("토큰 목적이 일치하지 않습니다")
    return payload
```

1회성 보장: Valkey에 `used_token:{jti}` 키를 설정하여 재사용 방지.

---

### 4-2. SMTP 이메일 발송 서비스

```python
# app/services/email_service.py

import aiosmtplib
from email.message import EmailMessage

class EmailService:
    def __init__(self, settings):
        self.host = settings.smtp_host
        self.port = settings.smtp_port
        self.username = settings.smtp_username
        self.password = settings.smtp_password
        self.from_address = settings.smtp_from_address

    async def send_email(self, to: str, subject: str, body: str):
        msg = EmailMessage()
        msg["From"] = self.from_address
        msg["To"] = to
        msg["Subject"] = subject
        msg.set_content(body)
        await aiosmtplib.send(msg, hostname=self.host, port=self.port,
                              username=self.username, password=self.password,
                              use_tls=True)
```

---

### 4-3. 비밀번호 재설정 요청

```python
# POST /api/v1/auth/forgot-password

Request Body: { "email": "user@example.com" }

처리 흐름:
1. 이메일로 사용자 조회
2. 사용자 미존재 시에도 200 반환 (이메일 존재 여부 노출 방지)
3. 사용자 존재 시:
   a. purpose_token 생성 (purpose=password_reset, 30분)
   b. 재설정 링크 구성: {frontend_url}/reset-password?token={token}
   c. 이메일 발송
   d. 감사 이벤트 기록

Response 200: { "message": "재설정 링크가 이메일로 발송되었습니다" }
```

---

### 4-4. 비밀번호 재설정 실행

```python
# POST /api/v1/auth/reset-password

Request Body:
{
  "token": "eyJ...",
  "new_password": "NewSecureP@ss1"
}

처리 흐름:
1. purpose_token 디코딩 (expected_purpose=password_reset)
2. jti로 1회성 확인 (Valkey used_token:{jti} 존재 시 거부)
3. 새 비밀번호 복잡도 검증
4. bcrypt 해싱 → users.password_hash 업데이트
5. 해당 사용자의 모든 refresh_tokens family 폐기 (강제 재로그인)
6. Valkey에 used_token:{jti} 저장 (TTL 30분)
7. 감사 이벤트 기록

Response 200: { "message": "비밀번호가 변경되었습니다" }

에러:
  400: 토큰 만료 또는 이미 사용됨
  422: 비밀번호 복잡도 미충족
```

---

### 4-5. 이메일 인증

```python
# POST /api/v1/auth/verify-email

Request Body: { "token": "eyJ..." }

처리 흐름:
1. purpose_token 디코딩 (expected_purpose=email_verify)
2. jti 1회성 확인
3. users.email_verified = TRUE, email_verified_at = now() 업데이트
4. Valkey에 used_token:{jti} 저장
5. 감사 이벤트 기록

Response 200: { "message": "이메일 인증이 완료되었습니다" }
```

회원가입(Task 14-2) 성공 후 자동으로 인증 이메일을 발송하도록 연계.

---

## 5. 산출물

1. `app/api/auth/purpose_tokens.py` — 목적별 JWT 토큰
2. `app/services/email_service.py` — SMTP 이메일 발송
3. `POST /api/v1/auth/forgot-password` 엔드포인트
4. `POST /api/v1/auth/reset-password` 엔드포인트
5. `POST /api/v1/auth/verify-email` 엔드포인트
6. 이메일 템플릿 (비밀번호 재설정, 이메일 인증)
7. 단위 테스트 (토큰 생성/검증, 1회성 보장, 이메일 mock)

---

## 6. 완료 기준

- 비밀번호 재설정 요청 시 이메일이 발송된다
- 미등록 이메일에도 동일한 200 응답이 반환된다 (정보 노출 방지)
- 재설정 토큰으로 비밀번호를 변경할 수 있다
- 사용된 재설정 토큰은 재사용이 불가능하다
- 비밀번호 변경 후 기존 세션/토큰이 전부 폐기된다
- 이메일 인증 토큰으로 이메일 인증이 완료된다
- `aiosmtplib`를 `requirements.txt`에 추가한다

---

## 7. Codex 작업 지침

- 이메일 발송 실패가 회원가입/재설정 요청을 블로킹하지 않도록 에러 처리한다
- 재설정 토큰은 이메일 존재 여부와 무관하게 동일 시간 내 응답한다 (타이밍 공격 방지)
- 이메일 본문에 사용자 비밀번호를 절대 포함하지 않는다
- `aiosmtplib`를 `requirements.txt`에 추가한다
- 프론트엔드 URL은 환경 변수 (`FRONTEND_BASE_URL`)로 설정한다
