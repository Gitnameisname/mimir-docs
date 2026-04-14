# 보안 위협 모델링 문서

> Phase 14 — Task 14-1 산출물 5/5
> 작성일: 2026-04-13
> 참조: OWASP Authentication Cheat Sheet, OWASP Testing Guide v4

---

## 1. 개요

Phase 14 인증 시스템에 대한 보안 위협을 STRIDE 모델과 OWASP Authentication 기준으로 분석하고, 각 위협에 대한 대응 전략을 정의한다.

---

## 2. 위협 모델링 대상

| 구성 요소 | 설명 | 신뢰 경계 |
|-----------|------|----------|
| 프론트엔드 (Next.js) | 사용자 입력, AT 메모리 저장, API 호출 | 비신뢰 (클라이언트) |
| 백엔드 API (FastAPI) | 인증/인가, 토큰 발급, 비즈니스 로직 | 신뢰 (서버) |
| PostgreSQL | 사용자, RT, OAuth 계정 데이터 | 신뢰 (내부) |
| Valkey | 세션, 로그인 시도 카운터, OAuth state | 신뢰 (내부) |
| GitLab (외부) | OAuth2/OIDC Provider | 부분 신뢰 (외부 서비스) |
| SMTP (외부) | 이메일 발송 | 부분 신뢰 (외부 서비스) |
| 브라우저 Cookie Store | RT 저장 | 비신뢰 (클라이언트) |

---

## 3. 위협 분석 및 대응

### T-1. Credential Stuffing (자격 증명 대입 공격)

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Spoofing (위장) |
| **공격 벡터** | 다른 서비스에서 유출된 이메일/비밀번호 조합으로 대량 로그인 시도 |
| **영향도** | 높음 — 계정 탈취, 개인정보 유출 |
| **발생 가능성** | 높음 — 자동화된 공격 도구가 널리 퍼져 있음 |

**대응 전략:**

```
1차 방어: 로그인 시도 제한 (Rate Limiting)
  - Valkey 기반 이메일별 카운터: login_attempts:{email}
  - 5회 실패 시 15분 잠금 (LOGIN_MAX_ATTEMPTS, LOGIN_LOCKOUT_MINUTES)
  - TTL 기반 자동 해제 (별도 해제 로직 불필요)

2차 방어: 비밀번호 정책 강화
  - 최소 8자, 2종류 이상 문자 유형 필수
  - bcrypt cost=12로 해싱 (해독 시간 증가)

3차 방어: 감사 로그
  - 모든 로그인 시도(성공/실패)를 audit_events에 기록
  - 동일 IP에서 다수 계정 실패 패턴 감지 (향후 Phase)
```

**구현 위치:** Task 14-2 (`session.py` 로그인 시도 제한), Task 14-3 (비밀번호 정책)

---

### T-2. XSS를 통한 Token Theft

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Information Disclosure (정보 노출) |
| **공격 벡터** | XSS 취약점을 통해 JavaScript 실행 → 토큰 탈취 |
| **영향도** | 높음 — 세션 하이재킹 |
| **발생 가능성** | 중간 — React의 기본 XSS 방어 존재, 단 dangerouslySetInnerHTML 등 사용 시 위험 |

**대응 전략:**

```
Access Token 보호:
  - 메모리 저장 (JavaScript 변수/React Context)
  - localStorage/sessionStorage 사용 금지
  - XSS 발생 시에도 페이지 새로고침하면 AT 소실 (피해 제한)
  - 짧은 수명 (15분)으로 탈취 시에도 제한된 시간

Refresh Token 보호:
  - HttpOnly Cookie → JavaScript 접근 불가
  - XSS로 직접 탈취 불가
  - Secure 플래그 → HTTPS만 전송
  - SameSite=Strict → 동일 사이트만 전송

추가 방어:
  - Content-Security-Policy 헤더 설정
  - React의 기본 이스케이핑 활용
  - 사용자 입력 렌더링 시 sanitization
```

**구현 위치:** Task 14-2 (Cookie 속성), Task 14-6 (프론트엔드 AT 저장), Phase 13 (CSP 헤더)

---

### T-3. Refresh Token Replay (재전송 공격)

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Spoofing + Elevation of Privilege |
| **공격 벡터** | 네트워크 스니핑, 로그, 또는 클라이언트 측 공격으로 RT를 탈취한 후 재사용 |
| **영향도** | 최상 — 장기간 세션 하이재킹 |
| **발생 가능성** | 중간 — HttpOnly Cookie로 직접 탈취 어려움, 네트워크 공격 필요 |

**대응 전략:**

```
RT Rotation + Reuse Detection:
  1. RT 사용 시 반드시 새 RT 발급 (Rotation)
  2. 기존 RT는 DB에서 used_at 기록
  3. 이미 사용된 RT가 다시 제출되면:
     a. 탈취 감지 (Reuse Detection)
     b. 해당 family 전체 폐기
     c. 정상 사용자 + 공격자 모두 재로그인 필요
     d. 감사 로그에 보안 이벤트 기록

Family 폐기 시나리오:
  공격자가 RT-1을 탈취 → RT-1으로 갱신 → RT-2 발급(공격자에게)
  정상 사용자가 RT-1로 갱신 시도 → 이미 사용됨 → Family 폐기
  → RT-2(공격자 보유)도 무효화됨
```

**구현 위치:** Task 14-2 (`refresh_tokens` 테이블, RT 검증 로직)

---

### T-4. CSRF (Cross-Site Request Forgery)

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Spoofing (위장) |
| **공격 벡터** | 악성 사이트에서 사용자 브라우저의 쿠키를 이용한 요청 위조 |
| **영향도** | 중간 — RT를 이용한 의도하지 않은 토큰 갱신 |
| **발생 가능성** | 낮음 — SameSite=Strict가 대부분의 CSRF 차단 |

**대응 전략:**

```
1차 방어: SameSite=Strict Cookie
  - RT 쿠키에 SameSite=Strict 설정
  - 외부 사이트에서의 요청에 쿠키 미전송
  - 대부분의 CSRF 공격 자동 차단

2차 방어: Origin/Referer 검증
  - POST /auth/refresh 요청의 Origin 헤더 검증
  - 허용된 도메인이 아닌 경우 거부

3차 방어 (선택): CSRF 토큰
  - Double Submit Cookie 패턴
  - 필요 시 향후 추가 (SameSite=Strict가 충분한 경우 생략)
```

**구현 위치:** Task 14-2 (Cookie SameSite), 미들웨어 (Origin 검증)

---

### T-5. OAuth State Tampering / CSRF

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Spoofing + Tampering |
| **공격 벡터** | OAuth state 파라미터를 위조하여 악의적 authorization code를 주입 |
| **영향도** | 높음 — 공격자 계정으로 로그인 유도, 세션 고정 |
| **발생 가능성** | 낮음 — state + PKCE 이중 방어 |

**대응 전략:**

```
State 파라미터:
  - 랜덤 UUID 생성 → Valkey에 저장 (TTL 10분)
  - 콜백 시 state 검증 → 일치하지 않으면 거부
  - 사용 후 즉시 삭제 (1회용)

PKCE (Proof Key for Code Exchange):
  - code_verifier: 43~128자 랜덤 문자열
  - code_challenge: SHA256(code_verifier), base64url 인코딩
  - Authorization 요청: code_challenge 전송
  - Token 교환: code_verifier 전송 → GitLab이 검증
  - Authorization Code 가로채기 공격 방지
```

**구현 위치:** Task 14-4 (OAuth 플로우 전체)

---

### T-6. Account Takeover via Password Reset

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Spoofing (위장) |
| **공격 벡터** | 비밀번호 재설정 메커니즘 악용 (토큰 추측, 링크 가로채기) |
| **영향도** | 최상 — 계정 완전 탈취 |
| **발생 가능성** | 낮음 — JWT 기반 토큰, 짧은 만료, 1회용 |

**대응 전략:**

```
재설정 토큰 보안:
  - JWT 형식 (HS256, type=password_reset)
  - 30분 만료
  - DB에 jti 기록, 사용 후 used_at 설정 (1회용)
  - 이메일 존재 여부에 관계없이 동일 응답 (열거 공격 방지)

비밀번호 변경 후 조치:
  - 해당 사용자의 모든 RT family 폐기
  - 기존 세션 강제 종료
  - 변경 확인 이메일 발송 (본인이 아닌 경우 인지)
```

**구현 위치:** Task 14-3 (비밀번호 재설정 API)

---

### T-7. SQL Injection

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Tampering (변조) |
| **공격 벡터** | 로그인 폼, 검색 필터 등 사용자 입력에 SQL 주입 |
| **영향도** | 최상 — 데이터 유출, 데이터 변조, 인증 우회 |
| **발생 가능성** | 매우 낮음 — SQLAlchemy ORM + 파라미터 바인딩 사용 |

**대응 전략:**

```
기본 방어: SQLAlchemy ORM
  - 파라미터 바인딩으로 SQL Injection 자동 방어
  - Raw SQL 사용 금지 (코드 리뷰에서 차단)

추가 방어:
  - 입력값 길이 제한 (Pydantic 모델 max_length)
  - 이메일 형식 검증 (EmailStr)
  - 비밀번호 길이 제한 (max 128자)
```

**검증 위치:** Task 14-16 (통합 테스트에서 SQL Injection 테스트 수행)

---

### T-8. JWT Algorithm Confusion

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Tampering (변조) |
| **공격 벡터** | JWT 헤더의 alg를 "none"으로 변경하여 서명 검증 우회 |
| **영향도** | 최상 — 임의의 JWT 생성 가능 |
| **발생 가능성** | 매우 낮음 — PyJWT의 기본 방어 존재 |

**대응 전략:**

```
PyJWT 설정:
  - algorithms=["HS256"]으로 명시적 지정
  - "none" 알고리즘 자동 거부
  - RS256/ES256 등 비대칭 키 혼동 불가 (HS256만 허용)

검증 코드:
  jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])
  # algorithms 파라미터가 화이트리스트 역할
```

**검증 위치:** Task 14-16 (alg:none 토큰 전송 테스트)

---

### T-9. Timing Attack (타이밍 공격)

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Information Disclosure (정보 노출) |
| **공격 벡터** | 로그인 응답 시간 차이로 이메일 존재 여부 추론 |
| **영향도** | 낮음 — 이메일 열거만 가능 |
| **발생 가능성** | 중간 — 정밀 측정 시 차이 감지 가능 |

**대응 전략:**

```
일정한 응답 시간 유지:
  - 사용자 미존재 시에도 bcrypt 더미 해싱 수행
  - 결과에 관계없이 유사한 처리 시간 보장

  예시:
  if user is None:
      bcrypt.verify("dummy", DUMMY_HASH)  # 더미 해싱으로 시간 일정화
      return 401
  if not bcrypt.verify(password, user.password_hash):
      return 401
```

**구현 위치:** Task 14-2 (로그인 API)

---

### T-10. Brute Force on Password Reset Token

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Spoofing (위장) |
| **공격 벡터** | 비밀번호 재설정 토큰 추측 시도 |
| **영향도** | 높음 — 계정 탈취 |
| **발생 가능성** | 매우 낮음 — JWT 토큰의 엔트로피가 충분 |

**대응 전략:**

```
토큰 엔트로피:
  - JWT jti에 UUID4 사용 (122비트 엔트로피)
  - 추측 불가능한 수준

추가 방어:
  - 30분 만료 (공격 시간 제한)
  - 1회용 (사용 후 DB에서 무효화)
  - 재설정 요청 Rate Limiting (동일 이메일 3분 간격)
```

**구현 위치:** Task 14-3

---

### T-11. Session Fixation

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Spoofing (위장) |
| **공격 벡터** | 공격자가 알고 있는 세션 ID를 피해자에게 강제 설정 |
| **영향도** | 높음 — 세션 하이재킹 |
| **발생 가능성** | 매우 낮음 — JWT 기반이므로 세션 ID 고정 불가 |

**대응 전략:**

```
JWT 기반 인증:
  - 로그인 시 새 JWT 발급 (예측 불가)
  - 서버 측 세션 ID 없음 (JWT가 곧 인증 정보)
  - 로그인 전후 동일 토큰 유지 불가능

추가 확인:
  - OAuth 콜백 후 새 토큰 발급 (기존 세션 무효화)
```

---

### T-12. OAuth Token Exposure in Database

| 항목 | 내용 |
|------|------|
| **STRIDE 분류** | Information Disclosure (정보 노출) |
| **공격 벡터** | DB 접근 권한을 가진 공격자가 OAuth 토큰을 평문으로 획득 |
| **영향도** | 높음 — GitLab 계정 접근 가능 |
| **발생 가능성** | 낮음 — DB 접근 필요 |

**대응 전략:**

```
AES-256-GCM 암호화:
  - oauth_accounts 테이블의 access_token, refresh_token 필드
  - OAUTH_TOKEN_ENCRYPTION_KEY로 암호화 후 저장
  - 조회 시 복호화
  - DB 유출 시에도 암호화 키 없이는 토큰 사용 불가

키 관리:
  - 암호화 키는 환경 변수 또는 Secret Manager에 저장
  - 키 로테이션은 향후 지원 (이중 키 방식)
```

**구현 위치:** Task 14-4 (oauth_accounts 저장 시 암호화)

---

## 4. OWASP Authentication Cheat Sheet 점검표

| # | 점검 항목 | 설계 상태 | 대응 |
|---|----------|----------|------|
| 1 | 비밀번호 저장 시 해싱 | O | bcrypt (cost=12) |
| 2 | 비밀번호가 로그에 노출되지 않음 | O | Pydantic SecretStr + 로깅 필터 |
| 3 | 인증 실패 메시지가 동일함 | O | "이메일 또는 비밀번호가 올바르지 않습니다" |
| 4 | Brute Force 방어 | O | 5회 실패 → 15분 잠금 |
| 5 | 비밀번호 복잡도 요구사항 | O | 8자 이상, 2종류 이상 문자 |
| 6 | 비밀번호 최대 길이 제한 | O | 128자 (bcrypt 72바이트 제한 고려) |
| 7 | HTTPS 강제 | O | Secure Cookie, 프로덕션 HTTPS 필수 |
| 8 | 세션 ID가 URL에 노출되지 않음 | O | JWT는 Authorization 헤더, RT는 Cookie |
| 9 | 로그인 후 세션 재생성 | O | 매 로그인 시 새 AT/RT 발급 |
| 10 | 비밀번호 변경 시 기존 세션 폐기 | O | 모든 RT family revoke |
| 11 | 적절한 세션 만료 | O | AT 15분, RT 7일 |
| 12 | 로그아웃 기능 존재 | O | POST /auth/logout → RT 폐기 + Cookie 삭제 |

---

## 5. 위험도 요약 매트릭스

```
영향도
  최상 │  T-7       T-3, T-6    T-8
       │  (SQL Inj)  (RT Replay, (JWT Alg)
       │              Pwd Reset)
  높음 │             T-1, T-2    T-12
       │             (Cred Stuff, (OAuth DB)
       │              XSS Token)
  중간 │                          T-4
       │                          (CSRF)
  낮음 │                          T-9
       │                          (Timing)
       └──────────────────────────────────
          매우 낮음     낮음      중간     높음
                      발생 가능성
```

**최우선 대응 항목:** T-1 (Credential Stuffing), T-3 (RT Replay), T-6 (Password Reset Abuse)
이 세 위협은 영향도와 발생 가능성 모두 높은 편이므로, 해당 대응 전략을 반드시 구현 초기부터 적용한다.

---

## 6. 구현 Task별 보안 체크리스트

| Task | 보안 항목 |
|------|----------|
| 14-2 (백엔드 인증) | T-1 로그인 제한, T-2 Cookie 속성, T-3 RT Rotation, T-4 SameSite, T-8 알고리즘 제한, T-9 타이밍 방어 |
| 14-3 (회원가입/재설정) | T-6 재설정 토큰 보안, T-10 토큰 엔트로피, T-7 입력값 검증 |
| 14-4 (GitLab OAuth) | T-5 State/PKCE, T-12 토큰 암호화 |
| 14-5 (이메일 서비스) | T-6 재설정 링크 보안 |
| 14-6~8 (프론트엔드) | T-2 AT 메모리 저장, T-4 CSRF 방어, T-11 세션 고정 방지 |
| 14-16 (통합 테스트) | 모든 위협에 대한 검증 테스트 |
