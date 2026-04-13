# Task 13-1. 보안 강화

## 1. 작업 목적

Mimir 플랫폼 전체의 보안 취약점을 점검하고 OWASP Top 10 기준으로 강화한다.

이 작업의 목표는 다음과 같다.

- OWASP Top 10 취약점 전수 검토 및 수정
- 인증/인가 정책 최종 검토
- API 보안 헤더 및 Rate Limiting 적용
- 민감 정보(API 키, PII) 처리 정책 검토
- 의존성 취약점 스캔 자동화

---

## 2. 작업 범위

### 포함 범위

- OWASP Top 10 취약점 점검 (A01~A10)
- 인증/인가 누락 엔드포인트 점검
- HTTP 보안 헤더 설정
- Rate Limiting 구현
- 의존성 취약점 스캔 (CI 통합)
- 환경 변수 및 시크릿 관리 점검

### 제외 범위

- 인프라 네트워크 보안 설정 (환경별 별도)
- 침투 테스트 (별도 외부 감사)

---

## 3. 선행 조건

- Phase 3 인증/인가 구조 이해
- Phase 2 권한/역할 모델 이해
- 코드베이스 전체 접근 가능

---

## 4. 주요 구현 대상

### 4-1. OWASP Top 10 점검 체크리스트

| # | 항목 | 점검 내용 | Mimir 관련 위험 |
|---|------|---------|--------------|
| A01 | Broken Access Control | 모든 API에 인가 검사 | 문서/청크 권한 누락 |
| A02 | Cryptographic Failures | 비밀번호 해싱, HTTPS 강제 | 사용자 자격증명 |
| A03 | Injection | SQL Injection, NoSQL Injection | 검색 쿼리, 필터 |
| A04 | Insecure Design | 설계 수준 취약점 | 워크플로 우회 |
| A05 | Security Misconfiguration | 디버그 모드, 기본 자격증명 | 개발 설정 운영 노출 |
| A06 | Vulnerable Components | 의존성 취약점 | npm/pip 패키지 |
| A07 | Auth Failures | 세션 관리, 브루트포스 | 로그인 시도 제한 |
| A08 | Software Integrity Failures | CI/CD 파이프라인 무결성 | 빌드 아티팩트 |
| A09 | Logging Failures | 보안 이벤트 로그 | 인증 실패 미기록 |
| A10 | SSRF | 외부 URL 요청 | 웹훅, 임베딩 API 호출 |

---

### 4-2. API 인증/인가 전수 점검

모든 API 엔드포인트에 대해 인증/인가 적용 여부를 테이블로 문서화:

```
점검 항목:
  - 인증 없이 접근 가능한 엔드포인트 목록 (의도적 공개 vs 누락)
  - 권한 검사 없이 타인 리소스 접근 가능한 엔드포인트
  - Admin 전용 엔드포인트에 일반 사용자 접근 가능 여부

수정 우선순위:
  1. 인증 누락 (즉시 수정)
  2. 인가 누락 (즉시 수정)
  3. 권한 우회 가능 (즉시 수정)
```

---

### 4-3. HTTP 보안 헤더

FastAPI 미들웨어로 보안 헤더 일괄 적용:

```python
@app.middleware("http")
async def add_security_headers(request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    response.headers["Content-Security-Policy"] = "default-src 'self'"
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    return response
```

---

### 4-4. Rate Limiting

```python
# slowapi 또는 fastapi-limiter 사용

# 로그인 엔드포인트: 1분 10회
@limiter.limit("10/minute")
async def login(request: Request): ...

# RAG 쿼리: 1분 20회
@limiter.limit("20/minute")
async def rag_query(request: Request): ...

# 일반 API: 1분 100회 (기본)
```

---

### 4-5. SQL Injection 방어

ORM(SQLAlchemy) 파라미터 바인딩 사용 여부 전수 확인:

```python
# 위험: 문자열 포맷팅으로 쿼리 조합
query = f"SELECT * FROM documents WHERE title = '{user_input}'"  # ❌

# 안전: 파라미터 바인딩
query = "SELECT * FROM documents WHERE title = :title"
db.execute(query, {"title": user_input})  # ✅
```

검색 및 필터 API를 우선 점검.

---

### 4-6. 의존성 취약점 스캔 CI 통합

```yaml
# .github/workflows/security.yml
- name: Python 취약점 스캔
  run: pip install safety && safety check

- name: JS 취약점 스캔
  run: npm audit --audit-level=high

- name: 정적 분석 (Bandit)
  run: bandit -r app/ -ll
```

---

### 4-7. 민감 정보 처리 점검

| 민감 정보 | 점검 내용 | 허용 기준 |
|---------|---------|---------|
| API 키 (OpenAI 등) | 로그 출력 금지, 환경 변수 저장 | 코드 내 0건 |
| 비밀번호 | bcrypt 해싱, 평문 저장 금지 | bcrypt rounds ≥ 12 |
| JWT 비밀키 | 환경 변수 저장, 최소 256비트 | 코드 내 0건 |
| 개인정보 (이메일) | 로그 마스킹 | 로그에 마스킹 적용 |

---

## 5. 산출물

1. OWASP Top 10 점검 결과 보고서 (취약점 및 수정 내역)
2. API 인증/인가 전수 점검 표
3. HTTP 보안 헤더 미들웨어
4. Rate Limiting 구현
5. 의존성 취약점 스캔 CI 설정
6. 민감 정보 처리 점검 결과

---

## 6. 완료 기준

- OWASP Top 10 점검 결과 Critical/High 취약점이 없다
- 모든 비공개 API에 인증이 적용되어 있다
- HTTP 보안 헤더가 모든 응답에 포함된다
- Rate Limiting이 로그인 및 RAG 쿼리에 적용된다
- 의존성 취약점 스캔이 CI에서 자동 실행된다

---

## 7. Codex 작업 지침

- OWASP 점검은 코드 리뷰 수준에서 수행하며, 취약점 발견 즉시 수정 후 테스트한다
- API 키가 로그에 노출되는 코드는 즉시 수정한다 (보안 최우선)
- Rate Limiting 임계값은 설정 파일에서 조정 가능하게 구현한다
- 의존성 취약점 스캔에서 High 이상 취약점은 PR 블로킹으로 설정한다
