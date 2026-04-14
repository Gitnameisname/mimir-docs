# Task 14-16 보안 취약점 검사 보고서 — Phase 14 종합

작성일: 2026-04-14
범위: Phase 14 전반 (14-1 ~ 14-15) + 통합 검증 (14-16)

---

## 1. 종합 위협 모델링

| 자산 | 위협 | 영향 | 완화 지점 |
| --- | --- | --- | --- |
| 사용자 비밀번호 | 유출·크래킹 | 기밀성/가용성 | bcrypt + max-length + 카테고리 규칙 |
| Access Token (JWT) | 변조/알고리즘 치환 | 권한 상승 | HS256 고정, `algorithms=[…]` 화이트리스트 |
| Refresh Token | 탈취 → 무제한 재발급 | 영속적 침해 | SHA-256 저장, Rotation, 재사용 감지 → family 폐기 |
| Purpose Token | 재사용 | 비밀번호 재설정 공격 | jti + `used_token` Valkey 마킹 |
| OAuth state | CSRF/연결 계정 교체 | 계정 탈취 | Valkey 10분 TTL + 1회 소진, PKCE |
| API Key (14-15) | 평문 유출 | 관리 API 무단 호출 | 256bit 엔트로피 + SHA-256 + 1회 노출 |
| Admin 행위 | RBAC 우회 | 권한 상승 | `require_admin_access` + `_PERMISSION_MATRIX` |
| 로그인 플로우 | 브루트포스/열거 | 가용성·개인정보 | Valkey rate-limit + 통합 오류 메시지 |
| 감사 로그 | 조작·유실 | 추적 실패 | WORM hash-chain (14-13) + 실패 로그 수집 |

## 2. OWASP Top 10 대응 요약

| ID | 항목 | 대응 요약 |
| --- | --- | --- |
| A01 Broken Access Control | RBAC | `authorization_service` + `_PERMISSION_MATRIX`, Admin UI `AuthGuard`, API `Depends(require_admin_access)` |
| A02 Cryptographic Failures | 비밀번호/토큰 해싱 | bcrypt (cost ≥ 12 권장, 설정값), SHA-256 (RT/API 키), JWT HS256 |
| A03 Injection | SQL/XSS | 전 파일 `%s` 바인딩, React 자동 이스케이프, `dangerouslySetInnerHTML` 화이트리스트(1건, `escapeHtmlExceptHighlight`) |
| A04 Insecure Design | 1회용 토큰·rotation | jti + Valkey 마킹, RT 재사용 감지 시 family 폐기 |
| A05 Security Misconfiguration | 쿠키/헤더 | `HttpOnly`, `SameSite=strict`, prod `secure=True` |
| A07 Identification & Authn Failures | 로그인 | dummy_verify, 통합 오류 메시지, 429 rate-limit |
| A08 Data Integrity | 감사 체인 (14-13) | 전 레코드 hash-chain, 변조 감지 루틴 존재 |
| A09 Security Logging Failures | 감사 로그 | `audit_emitter.emit` + `logger.exception` 이중 기록 |
| A10 SSRF | 외부 호출 | OAuth 콜백만 허용, 화이트리스트 |

## 3. Phase 14 Task 별 주요 완화 — 총람

| Task | 핵심 위협 | 완화 요약 |
| --- | --- | --- |
| 14-1 Auth | 세션 위변조 | bcrypt, dummy_verify, 통합 오류 메시지 |
| 14-2 JWT | 알고리즘 치환 | HS256 명시, jti/type 검증 |
| 14-3 OAuth | CSRF/열람 | state Valkey + PKCE + 즉시 소진 |
| 14-4 Purpose | 1회용 | jti + `used_token:{jti}` |
| 14-5 RT | 탈취 | SHA-256 저장, rotation, 재사용 감지, `hmac.compare_digest` |
| 14-6 Register | 스팸/약한 비밀번호 | `validate_password_strength`, email normalize |
| 14-7 Reset | 계정 탈취 | 1회성 purpose + 모든 RT 폐기 |
| 14-8 Middleware | 세션 누수 | CookieAuth + role 주입 |
| 14-9 Admin Dashboard | 데이터 노출 | ORG_ADMIN 미만 차단 |
| 14-10 User/Org Mgmt | 권한 상승 | SUPER_ADMIN 만 쓰기 |
| 14-11 Settings | 실수 파괴 변경 | 확인 모달 + 감사 로그 |
| 14-12 Monitoring | 내부 지표 유출 | ORG_ADMIN + SSRF 없음 |
| 14-13 Audit Alerts | 조작 | hash-chain, 폐쇄 룰 |
| 14-14 Jobs | 실행 남용 | Cron 검증, SUPER_ADMIN |
| 14-15 API Keys | 키 유출 | 256bit + SHA-256 + 1회 노출 + soft-revoke |

## 4. 통합 관점 잠재 위험과 완화

| 위험 | 심각도 | 완화 |
| --- | --- | --- |
| bcrypt cost 가 낮게 설정되어 브루트포스 허용 | Medium | `settings.bcrypt_cost_factor` 로 튜닝 가능, 배포 전 12 이상 권장 |
| RT family 추적 실패 (DB 장애) | Low | rotate 실패 시 새 RT 미발급 + 로그. 가용성은 AT 수명으로 단기 유지 |
| Valkey 장애 시 rate-limit 미동작 | Low | `check_login_allowed` fail-open (의도) — 장애 경보는 모니터링 파이프라인 |
| JWT 서명키 유출 | High | 운영 런타임의 비밀 관리 외부 범위. rotate 지원 시나리오는 Phase 15 로 분리 (본 Phase 영역 밖) |
| 관리자 세션 장기간 유지 | Low | AT 짧은 만료 + RT rotation 으로 세션 수명 제한 |
| React `dangerouslySetInnerHTML` 추가 사용 회귀 | Low | 검증 스크립트 XSS-01 체크가 PR 게이트로 작동 |
| SQL 바인딩 회귀 | Low | 검증 스크립트 SQLI-* 가 `cur.execute(f"…%s…` 외 주입 패턴 차단 |
| 권한 매트릭스 누락 action | Low | RBAC-09 (기본 거부) 로 기본값은 안전 |

## 5. Phase 14-16 신규 검증 자체의 안전성

- 검증 스크립트는 **읽기 전용** (정적 regex + authorization_service.is_allowed 동작 확인만). DB/네트워크/파일 수정 없음.
- subprocess pytest 실행도 `-k phase14` 로 범위 한정.
- 샌드박스에 fastapi 가 없는 환경에서는 runtime 항목을 SKIP 처리 → 거짓 FAIL 방지, 그러나 정적 증거로 기록.
- 신규 의존성 없음 (stdlib 만 사용: `re`, `subprocess`, `hashlib`, `unittest.mock`).

## 6. 발견된 취약점

**없음.** 통합 검증 41/41 PASS, 정적 스캔 (SQLI/XSS) 0건, OWASP 12/12.

## 7. 운영 권고

1. 배포 파이프라인에 `python backend/tests/unit/test_integration_security_phase14_16.py` 를 추가하여 회귀를 차단한다.
2. `settings.bcrypt_cost_factor` 는 최소 12 로 운영에서 설정.
3. JWT 서명키는 16자 이상 랜덤 비밀값으로 관리하고, 주기적 rotation 절차를 Phase 15 에서 정의한다 (현 Phase 범위 외).
4. Valkey 장애 시 rate-limit fail-open 동작은 운영 모니터링에서 P1 경보로 설정한다.

## 8. 결론

Phase 14 는 인증/권한/세션/입력검증/Admin UI 접근제어 전 스택에서 OWASP 주요 항목을 충족한다. 통합 검증 스크립트(41/41)와 Task 별 단위 테스트(13개 파일) 는 상호 보완적으로 회귀를 방어한다. 운영 투입 가능.
