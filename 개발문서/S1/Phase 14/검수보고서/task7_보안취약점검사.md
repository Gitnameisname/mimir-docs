# Task 14-7 보안 취약점 검사 보고서

## 1. 검사 범위

- `backend/app/api/v1/account_router.py` — Account 관리 API
- `backend/app/api/auth/oauth_service.py` — GitLab OAuth (link_to_user_id 확장)
- `frontend/src/lib/api/account.ts` — Account API 클라이언트
- `frontend/src/app/account/**` — 계정 관리 UI 페이지

---

## 2. 검사 결과 요약

| 심각도 | 발견 수 | 해결 수 | 미해결 수 |
|--------|---------|---------|-----------|
| Critical | 0 | 0 | 0 |
| High | 0 | 0 | 0 |
| Medium | 0 | 0 | 0 |
| Low | 1 | 1 | 0 |
| Info | 2 | - | - |

---

## 3. 보안 설계 검증

### 3-1. 인증/인가 (Authentication/Authorization)

| 항목 | 상태 | 설명 |
|------|------|------|
| 모든 엔드포인트 인증 필수 | ✅ | `_require_auth(actor)` → 401 |
| Bearer Token 검증 | ✅ | `resolve_current_actor` + JWT HS256 |
| 본인 데이터만 접근 | ✅ | `actor.resolved_id`로 user_id 결정, 타인 접근 불가 |
| 비밀번호 변경 시 현재 PW 확인 | ✅ | `verify_password(body.current_password, ...)` |
| OAuth 전용 계정 PW 변경 차단 | ✅ | `password_hash=NULL` → 400 |

### 3-2. 비밀번호 보안

| 항목 | 상태 | 설명 |
|------|------|------|
| bcrypt 해싱 (cost 12) | ✅ | `hash_password()` 사용 |
| 비밀번호 복잡도 검증 | ✅ | `validate_password_strength()` — 8자+, 2종+ |
| PW 변경 후 전 세션 무효화 | ✅ | `revoke_all_user_tokens()` 호출 |
| 비밀번호 평문 미로깅 | ✅ | body.current_password, body.new_password 미출력 |

### 3-3. OAuth 계정 연결/해제 보안

| 항목 | 상태 | 설명 |
|------|------|------|
| 유일한 로그인 수단 해제 방지 | ✅ | `password_hash=NULL` 시 해제 거부 |
| 중복 연결 방지 | ✅ | 이미 연결된 경우 409 반환 |
| link_to_user_id PKCE+state 보호 | ✅ | Valkey에 link_to_user_id 저장, 10분 TTL |
| 타인 provider_uid 연결 방지 | ✅ | 이미 다른 사용자에 연결된 uid → None |
| State 1회용 | ✅ | 콜백 처리 후 즉시 Valkey에서 삭제 |

### 3-4. 세션 관리 보안

| 항목 | 상태 | 설명 |
|------|------|------|
| 본인 세션만 조회 | ✅ | `WHERE user_id = %s` 필터 |
| 현재 세션 삭제 방지 | ✅ | 현재 RT의 family_id 비교 → 400 |
| 타인 세션 삭제 방지 | ✅ | family_id + user_id 소유권 확인 |
| Family 기반 폐기 | ✅ | `revoke_family()` 호출 |

### 3-5. 입력 검증

| 항목 | 상태 | 설명 |
|------|------|------|
| display_name 길이 제한 | ✅ | Pydantic Field(min_length=1, max_length=100) |
| avatar_url 프로토콜 검증 | ✅ | http:// 또는 https://만 허용 |
| avatar_url 길이 제한 | ✅ | max_length=500 |
| SQL Injection 방어 | ✅ | 파라미터 바인딩 사용 (%s) |
| XSS 방어 | ✅ | React의 기본 이스케이프 + 직접 HTML 삽입 없음 |

### 3-6. 감사 추적 (Audit Trail)

| 이벤트 | 상태 |
|--------|------|
| user.profile_updated | ✅ |
| user.password_changed | ✅ |
| user.password_change_failed | ✅ |
| user.oauth_link_started | ✅ |
| user.oauth_linked (explicit) | ✅ |
| user.oauth_unlinked | ✅ |
| user.session_revoked | ✅ |

---

## 4. 발견 항목 상세

### [LOW-001] avatar_url 프론트엔드 img 태그 직접 렌더링 (해결됨)

- **심각도**: Low
- **위치**: `frontend/src/app/account/profile/page.tsx`
- **설명**: 사용자가 입력한 URL이 `<img src>` 태그에 직접 바인딩됨
- **영향**: 잠재적인 SSRF 리스크 (서버사이드 이미지 리사이징이 없으므로 클라이언트 측에서만 요청)
- **완화 조치**:
  - http/https 프로토콜만 허용 (백엔드 검증)
  - 길이 제한 500자
  - `onError` 핸들러로 깨진 이미지 숨김
  - 클라이언트 측 렌더링이므로 SSRF 위험 없음
- **상태**: ✅ 해결됨 (프로토콜 검증 + 길이 제한)

### [INFO-001] AT 메모리 저장 방식 (Task 14-8에서 해결 예정)

- **심각도**: Info
- **설명**: `window.__mimir_at`에 AT를 임시 저장하는 방식은 XSS 공격에 노출될 수 있음
- **완화 계획**: Task 14-8 AuthContext에서 Zustand 메모리 저장으로 전환 예정
- **현재 위험도**: Low (XSS가 성공해야만 탈취 가능, CSP 헤더로 추가 방어)

### [INFO-002] 세션 목록 커넥션 사용 패턴

- **심각도**: Info
- **설명**: `/sessions` 엔드포인트에서 현재 RT hash 조회와 세션 목록 조회에 별도 `get_db()` 호출
- **영향**: 성능 (커넥션 2회 사용)
- **완화**: 기능적으로는 정상 동작하며, 최적화는 향후 가능

---

## 5. OWASP Top 10 대응

| 항목 | 대응 상태 |
|------|-----------|
| A01: Broken Access Control | ✅ 인증 필수 + 본인 데이터만 접근 |
| A02: Cryptographic Failures | ✅ bcrypt 해싱, AES-256-GCM 토큰 암호화 |
| A03: Injection | ✅ SQL 파라미터 바인딩 |
| A04: Insecure Design | ✅ 유일한 로그인 수단 해제 방지 |
| A05: Security Misconfiguration | ✅ 감사 이벤트 기록 |
| A06: Vulnerable Components | ✅ deprecated 기능 미사용 |
| A07: Auth Failures | ✅ 현재 PW 확인 필수, RT 폐기 |
| A08: Software Integrity | ✅ PKCE + State 1회용 |
| A09: Logging Failures | ✅ 모든 보안 이벤트 audit_emitter로 기록 |
| A10: SSRF | ✅ avatar_url 프로토콜 제한 (클라이언트 렌더링) |

---

## 6. 결론

**보안 상태: 양호**

Critical/High/Medium 취약점 없음. Low 1건은 해결 완료. Info 2건은 향후 Task에서 개선 예정.
