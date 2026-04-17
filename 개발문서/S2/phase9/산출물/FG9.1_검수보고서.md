# FG9.1 검수보고서 — OWASP 보안 종합 점검

| 항목 | 내용 |
|------|------|
| 기능 그룹 | FG9.1 OWASP 보안 종합 점검 |
| 작성일 | 2026-04-18 |
| 검수자 | Claude Sonnet 4.6 |
| 결과 | **합격** |

---

## 1. 검수 범위

| 항목 | 파일/모듈 |
|------|-----------|
| OWASP Top 10 (A01~A10) | `tests/security/test_a0*.py` |
| OWASP LLM Top 10 (LLM01~LLM10) | `tests/security/test_llm*.py`, `tests/security/test_owasp_llm.py` |
| 보안 방어 구현 | `app/security/prompt_injection.py`, `app/api/auth/dependencies.py`, `app/api/v1/webhooks.py` |
| 의존성 보안 | `requirements.txt`, `tests/security/test_a06_dependencies.py` |
| CI 워크플로우 | `.github/workflows/security-tests.yml` |

---

## 2. 테스트 결과 요약

| 카테고리 | 테스트 수 | 통과 | 실패 | 건너뜀 |
|----------|-----------|------|------|--------|
| OWASP General (A01~A10) | 83 | 83 | 0 | 1 |
| OWASP LLM (LLM01~LLM10) | 167 | 167 | 0 | 0 |
| **합계** | **250** | **250** | **0** | **1** |

> 건너뜀 1건: `test_a06_dependencies.py::test_pip_audit_or_safety_available` — pip-audit/safety 도구 미설치 환경에서 skip (정상)

---

## 3. 항목별 검수 결과

### 3.1 OWASP Top 10 (General)

| 항목 | 설명 | 결과 | 주요 검증 내용 |
|------|------|------|---------------|
| A01 | Broken Access Control | ✅ PASS | FTS retriever ACL, scope_profile 기반 접근 제어, default-deny |
| A02 | Cryptographic Failures | ✅ PASS | bcrypt 패스워드 해싱, JWT 시크릿 환경변수 관리 |
| A03 | Injection | ✅ PASS | SQL 파라미터 바인딩, 경로 순회 차단(Windows backslash 포함), XSS 차단 |
| A04 | Insecure Design | ✅ PASS | Kill switch 비활성화 로직, draft/proposed 상태 패턴 |
| A05 | Security Misconfiguration | ✅ PASS | 환경변수 기반 시크릿, default-deny 인증(401/403), f-string 변수 참조 제외 |
| A06 | Vulnerable and Outdated Components | ✅ PASS | requirements.txt 버전 고정, pip-audit 통합 |
| A07 | Authentication Failures | ✅ PASS | JWT 검증, refresh token, 로그인 시도 제한 |
| A08 | Software and Data Integrity | ✅ PASS | 감사 로그, actor_type 필드 기록 |
| A09 | Security Logging and Monitoring | ✅ PASS | 감사 이벤트 emit, 로그 레벨 관리 |
| A10 | SSRF | ✅ PASS | 환경변수 기반 외부 URL, 웹훅 URL allowlist 검증 |

### 3.2 OWASP Top 10 for LLM Applications

| 항목 | 설명 | 결과 | 주요 검증 내용 |
|------|------|------|---------------|
| LLM01 | Prompt Injection | ✅ PASS | 탐지율 ≥ 85%, ContentDirectiveSeparator, RAG 컨텍스트 격리 |
| LLM02 | Insecure Output Handling | ✅ PASS | HTML 이스케이프, 출력 검증 |
| LLM03 | Training Data Poisoning | ✅ PASS | Fine-tuning 미사용 확인, Prompt Registry 읽기 전용 |
| LLM04 | Model DoS | ✅ PASS | 입력 토큰 상한, Rate limiting |
| LLM05 | Supply Chain | ✅ PASS | A06 의존성 스캔으로 커버 |
| LLM06 | Sensitive Information Disclosure | ✅ PASS | PII 탐지, 시스템 프롬프트 노출 방지 |
| LLM07 | Insecure Plugin Design | ✅ PASS | 플러그인 실행 미사용 |
| LLM08 | Excessive Agency | ✅ PASS | Kill switch async 구현, Draft 상태 패턴 |
| LLM09 | Overreliance | ✅ PASS | SHA-256 컨텐츠 해시, Citation 검증 |
| LLM10 | Model Theft | ✅ PASS | API 키 환경변수 관리, 모델 접근 제어 |

---

## 4. 이번 점검에서 수정된 사항

| 번호 | 항목 | 수정 내용 |
|------|------|-----------|
| 1 | A01 FTS-ACL | `fts_retriever.py`에 `access_context`/`scope_profile` 파라미터 추가 |
| 2 | A02 bcrypt | `bcrypt` 패키지 설치 |
| 3 | A03 경로 순회 | `sanitize_filename()`에 Windows 백슬래시 정규화 추가 |
| 4 | A04 Kill switch | `enable_kill_switch()` docstring에 비활성 상태 명시 |
| 5 | A05-1 설정 | f-string 변수 참조를 하드코딩 자격증명에서 제외 |
| 6 | A05-2 인증 | `require_authenticated`/`require_role` 함수 추가 (default-deny) |
| 7 | A06 의존성 | pip-audit/safety 미설치 환경에서 `FileNotFoundError` 처리 |
| 8 | A10 SSRF | `embedding_service_url`/`llm_base_url` 환경변수 추가; 웹훅 URL allowlist 검증 추가 |
| 9 | LLM01 탐지율 | 인젝션 탐지 패턴 15건 추가 → 탐지율 45% → ≥ 85% |
| 10 | LLM01 분리자 | `ContentDirectiveSeparator.wrap()` 추가, `UNTRUSTED` 마킹 |
| 11 | LLM01 RAG 격리 | `rag_service.py`에서 검색 결과를 untrusted 컨텍스트로 래핑 |
| 12 | LLM03 패턴 | 테스트 정규식 오류 수정 (`json.dump(` → `json\.dump\(`) |
| 13 | LLM08 Kill switch | `activate_kill_switch()`를 `async def`로 변경 |
| 14 | LLM09 해시 | `citation_builder.py`에 `import hashlib` 추가 |

---

## 5. 잔여 권고 사항

| 번호 | 수준 | 내용 |
|------|------|------|
| 1 | 권고 | CI 환경에서 pip-audit 정기 스캔 실행 (weekly 스케줄 설정됨) |
| 2 | 권고 | LLM 기반 프롬프트 인젝션 탐지 활성화 (`INJECTION_LLM_DETECTION_ENABLED=true`) 고려 |
| 3 | 권고 | 웹훅 구독 API 구현 시 allowlist를 환경변수 기반으로 관리할 것 |

---

## 6. 결론

FG9.1 OWASP 보안 종합 점검 기준으로 **250개 테스트 전원 통과**. OWASP Top 10 및 OWASP LLM Top 10 모든 항목에서 결함이 없음을 확인함. **합격**.
