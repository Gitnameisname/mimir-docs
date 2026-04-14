# Task 14-3. JWT 이중 토큰 체계 구현

## 1. 작업 목적

기존 Access Token 전용 JWT 시스템을 Access/Refresh Token 이중 체계로 확장한다.

이 작업의 목표는 다음과 같다.

- Refresh Token 발급·검증·폐기 구현
- Refresh Token Rotation (사용 시 새 RT 발급) 구현
- Family 기반 탈취 감지 및 일괄 폐기 구현
- `refresh_tokens` 테이블 생성
- 기존 `tokens.py` 확장 (하위 호환 유지)
- 토큰 갱신 API (`POST /auth/refresh`) 구현
- 로그아웃 API (`POST /auth/logout`) 구현

---

## 2. 작업 범위

### 포함 범위

- `refresh_tokens` 테이블 생성
- Refresh Token 발급 (SHA-256 해시 저장, family_id 부여)
- Refresh Token Rotation 로직
- Family 단위 폐기 (탈취 감지)
- `POST /api/v1/auth/refresh` 엔드포인트
- `POST /api/v1/auth/logout` 엔드포인트
- 기존 `tokens.py`에 jti claim 추가
- Task 14-2의 로그인 API와 통합 (로그인 성공 → AT+RT 발급)

### 제외 범위

- 프론트엔드 자동 갱신 (Task 14-8에서 구현)
- OAuth 로그인 토큰 발급 (Task 14-4에서 통합)

---

## 3. 선행 조건

- Task 14-1 인증 아키텍처 설계 완료
- Task 14-2 비밀번호 인증 구현 완료
- `tokens.py` — `create_access_token()`, `decode_access_token()` 이해
- `session.py` — Valkey 연결 이해

---

## 4. 주요 구현 대상

### 4-1. refresh_tokens 테이블 생성

```sql
CREATE TABLE refresh_tokens (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash  VARCHAR(255) NOT NULL UNIQUE,
  family_id   UUID NOT NULL,
  expires_at  TIMESTAMPTZ NOT NULL,
  revoked     BOOLEAN DEFAULT FALSE,
  revoked_at  TIMESTAMPTZ,
  created_at  TIMESTAMPTZ DEFAULT now(),
  ip_address  VARCHAR(50),
  user_agent  TEXT
);

CREATE INDEX idx_refresh_tokens_user   ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_family ON refresh_tokens(family_id);
CREATE INDEX idx_refresh_tokens_hash   ON refresh_tokens(token_hash);
```

---

### 4-2. tokens.py 확장

기존 `create_access_token()` 확장 및 Refresh Token 함수 추가:

```python
# 기존 함수 확장
def create_access_token(actor_id: str, role: str) -> str:
    payload = {
        "sub": actor_id,
        "role": role,
        "exp": datetime.utcnow() + timedelta(minutes=settings.jwt_access_expire_minutes),
        "iat": datetime.utcnow(),
        "jti": str(uuid4()),       # 추가: 고유 토큰 ID
        "type": "access",          # 추가: 토큰 타입 구분
    }
    return jwt.encode(payload, settings.jwt_secret, algorithm="HS256")

# 신규 함수
def create_refresh_token(actor_id: str) -> tuple[str, str]:
    """Refresh Token 생성, (raw_token, token_hash) 반환"""
    raw_token = secrets.token_urlsafe(64)
    token_hash = hashlib.sha256(raw_token.encode()).hexdigest()
    return raw_token, token_hash

def verify_refresh_token_hash(raw_token: str, stored_hash: str) -> bool:
    computed = hashlib.sha256(raw_token.encode()).hexdigest()
    return hmac.compare_digest(computed, stored_hash)
```

---

### 4-3. Refresh Token Rotation 흐름

```
1. 클라이언트: POST /auth/refresh (Cookie: refresh_token=<raw_token>)
2. 서버:
   a. raw_token → SHA-256 해시 → DB 조회
   b. 조회된 레코드 검증:
      - revoked == True? → 탈취 감지! family 전체 폐기 → 401
      - expires_at < now()? → 만료 → 401
   c. 현재 RT를 revoked=True로 마킹
   d. 같은 family_id로 새 RT 발급 → DB 저장
   e. 새 Access Token 발급
   f. 응답: Set-Cookie(새 RT) + Body(새 AT)
```

---

### 4-4. 탈취 감지 및 Family 폐기

```python
async def revoke_token_family(pool, family_id: UUID):
    """family_id에 속한 모든 RT를 일괄 폐기"""
    await pool.execute("""
        UPDATE refresh_tokens
        SET revoked = TRUE, revoked_at = now()
        WHERE family_id = $1 AND revoked = FALSE
    """, family_id)

# 이미 revoked된 RT가 사용된 경우 → 탈취 의심
# 해당 family 전체를 폐기하여 공격자와 정상 사용자 모두 재로그인 유도
```

---

### 4-5. 로그아웃 API

```python
# POST /api/v1/auth/logout

처리 흐름:
1. Cookie에서 refresh_token 추출
2. raw_token → SHA-256 해시 → DB 조회
3. 해당 RT의 family_id로 family 전체 폐기
4. Cookie 삭제 (Set-Cookie: refresh_token=; Max-Age=0)
5. 기존 Valkey 세션도 삭제 (session.delete_session)
6. 감사 이벤트 기록

Response 200: { "message": "로그아웃 완료" }
```

---

### 4-6. 토큰 갱신 API

```python
# POST /api/v1/auth/refresh

처리 흐름:
1. Cookie에서 refresh_token 추출 (없으면 401)
2. SHA-256 해시 → DB 조회
3. 유효성 검증 (revoked, expires_at, 사용자 상태)
4. Rotation 실행 (4-3 참조)
5. 새 Access Token + 새 Refresh Token 반환

Response 200:
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 900
}
+ Set-Cookie: refresh_token=<new_rt>; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth; Max-Age=604800
```

---

### 4-7. 로그인 API 통합

Task 14-2의 로그인 성공 후 토큰 발급을 연결:

```python
# POST /api/v1/auth/login 최종 응답

로그인 성공 시:
1. Access Token 발급
2. Refresh Token 발급 (새 family_id 생성)
3. refresh_tokens 테이블에 저장
4. 응답: Body(AT) + Set-Cookie(RT)
```

---

## 5. 산출물

1. `refresh_tokens` 테이블 생성 DDL
2. `tokens.py` 확장 (jti, type claim, RT 생성/검증)
3. `app/api/auth/refresh_service.py` — RT Rotation, family 폐기
4. `POST /api/v1/auth/refresh` 엔드포인트
5. `POST /api/v1/auth/logout` 엔드포인트
6. Task 14-2 로그인 API 토큰 통합
7. 단위 테스트 (토큰 발급/갱신/폐기, Rotation, 탈취 감지)

---

## 6. 완료 기준

- 로그인 성공 시 Access Token(15분) + Refresh Token(7일)이 발급된다
- Refresh Token이 HttpOnly Secure Cookie로 전송된다
- POST /auth/refresh 요청 시 새 AT + 새 RT가 발급된다 (Rotation)
- 이미 사용된 RT로 갱신 요청 시 family 전체가 폐기된다
- 로그아웃 시 해당 family의 모든 RT가 폐기된다
- 만료된 RT로 갱신 요청 시 401이 반환된다
- 기존 Access Token 검증 로직이 하위 호환된다

---

## 7. Codex 작업 지침

- Refresh Token은 DB에 평문이 아닌 SHA-256 해시로 저장한다
- RT Cookie의 Path를 `/api/v1/auth`로 제한하여 불필요한 전송을 방지한다
- `hmac.compare_digest()`를 사용하여 timing attack을 방어한다
- 기존 `create_access_token()`의 시그니처를 변경할 경우 기존 호출부를 모두 업데이트한다
- 만료된 refresh_tokens 레코드는 배치로 정리할 수 있도록 created_at 인덱스를 활용한다
