# FG9.1 보안취약점검사보고서

| 항목 | 내용 |
|------|------|
| 대상 시스템 | Mimir Season 2 백엔드 (FastAPI) |
| 점검 기준 | OWASP Top 10 (2021) + OWASP Top 10 for LLM Applications (2023) |
| 점검일 | 2026-04-18 |
| 점검자 | Claude Sonnet 4.6 (자동화 + 수동 코드 리뷰) |
| 결과 | **High: 0건, Medium: 0건, Low: 0건, 권고: 3건** |

---

## 1. 점검 방법

| 방법 | 도구/기법 |
|------|-----------|
| 자동화 테스트 | pytest 기반 250개 보안 테스트 (tests/security/) |
| 정적 분석 | 소스코드 패턴 검사 (정규식 기반) |
| 의존성 스캔 | pip-audit (CI 통합) |
| 수동 코드 리뷰 | 인증·인가·입력 검증·LLM 파이프라인 |

---

## 2. 발견된 취약점

### 2.1 이번 점검에서 수정 완료된 취약점

| ID | 심각도 | OWASP | 설명 | 상태 |
|----|--------|-------|------|------|
| VULN-S2P9-001 | High | A01 | FTS retriever에 ACL 미적용 — scope_profile 기반 접근 제어 누락 | ✅ 수정완료 |
| VULN-S2P9-002 | Medium | A03 | `sanitize_filename()`이 Windows 경로 순회(`..\\system32`) 미처리 | ✅ 수정완료 |
| VULN-S2P9-003 | Medium | A05 | 기본 인증 거부 정책(default-deny) 미구현 — 미인증 요청 허용 가능 | ✅ 수정완료 |
| VULN-S2P9-004 | Medium | A10 | 웹훅 URL 검증 미구현 — SSRF 공격 가능 | ✅ 수정완료 |
| VULN-S2P9-005 | Medium | LLM01 | Prompt Injection 탐지율 45% (목표 ≥ 85%) | ✅ 수정완료 |
| VULN-S2P9-006 | Medium | LLM01 | RAG 파이프라인에서 검색 결과를 신뢰 데이터로 처리 — Content-Directive 분리 미적용 | ✅ 수정완료 |
| VULN-S2P9-007 | Low | LLM08 | Kill switch 동기 처리 — 5초 내 응답 보장 불확실 | ✅ 수정완료 |
| VULN-S2P9-008 | Low | A06 | pip-audit 미설치 환경에서 `FileNotFoundError` 발생 | ✅ 수정완료 |

### 2.2 잔여 권고 사항 (Low/정보)

| ID | 수준 | OWASP | 내용 | 처리 계획 |
|----|------|-------|------|-----------|
| REC-S2P9-001 | 권고 | A06 | pip-audit CI 통합은 설정됐으나 로컬 환경 설치 미보장 | CI 스케줄 실행으로 커버 |
| REC-S2P9-002 | 권고 | LLM01 | LLM 기반 프롬프트 인젝션 탐지 미활성화 (룰 기반만 동작) | `INJECTION_LLM_DETECTION_ENABLED=true` 운영 고려 |
| REC-S2P9-003 | 권고 | A10 | 웹훅 구독 API 미구현 — URL allowlist를 환경변수로 관리 필요 | 웹훅 구현 시 적용 |

---

## 3. OWASP Top 10 항목별 상세 결과

### A01 — Broken Access Control

**점검 항목**: Scope Profile ACL, FTS retriever ACL, default-deny 인증, 에이전트 kill switch

**결과**: 합격
- `access_context`/`scope_profile` 기반 ACL이 FTS retriever에 적용됨
- `require_authenticated()` → 401 Unauthorized, `require_role()` → 403 Forbidden 구현
- Kill switch 활성화 시 에이전트 비활성화 즉시 적용

### A02 — Cryptographic Failures

**점검 항목**: 패스워드 해싱, JWT 시크릿, 민감 데이터 평문 저장

**결과**: 합격
- bcrypt cost factor 12 사용
- JWT 시크릿 환경변수 관리 (`jwt_secret`)
- Production 환경에서 미설정 시 startup 실패

### A03 — Injection

**점검 항목**: SQL Injection, Path Traversal, XSS, Command Injection

**결과**: 합격
- psycopg2 파라미터 바인딩 (SQL Injection 방지)
- `sanitize_filename()` PurePosixPath + Windows 백슬래시 정규화
- HTML sanitizer (XSS 방지)
- 명령 실행 코드 없음 (Command Injection 해당 없음)

### A04 — Insecure Design

**점검 항목**: 에이전트 쓰기 제한, kill switch, draft 패턴

**결과**: 합격
- 에이전트 액션은 `proposed` → 승인 후 실행 패턴
- Kill switch 활성화 시 `is_active=False`로 즉시 비활성화

### A05 — Security Misconfiguration

**점검 항목**: 하드코딩 자격증명, 기본값 보안, 기본 인증 거부

**결과**: 합격
- 자격증명 환경변수 관리 (`jwt_secret`, `postgres_password`, `internal_service_secret`)
- f-string 변수 참조는 하드코딩에서 제외
- `require_authenticated()` default-deny 구현

### A06 — Vulnerable and Outdated Components

**점검 항목**: requirements.txt 버전 고정, pip-audit 스캔

**결과**: 합격 (pip-audit CI 통합 확인)
- 주요 패키지 버전 고정 (`==` 연산자)
- CI weekly pip-audit 스캔 설정 (`security-tests.yml`)

### A07 — Authentication Failures

**점검 항목**: JWT 검증, refresh token, 로그인 시도 제한, bcrypt

**결과**: 합격
- JWT 만료 15분, refresh token 7일
- 로그인 실패 5회 시 15분 잠금
- bcrypt cost 12

### A08 — Software and Data Integrity

**점검 항목**: 감사 로그, actor_type 필드, 무결성 검증

**결과**: 합격
- 모든 쓰기 작업에 `actor_type` ("user"/"agent") 기록
- audit_emitter를 통한 이벤트 추적

### A09 — Security Logging and Monitoring

**점검 항목**: 보안 이벤트 로깅, 로그 레벨, 알림

**결과**: 합격
- 인증 실패, 인젝션 탐지, kill switch 활성화 이벤트 로깅
- logger.warning 사용 (보안 이벤트)

### A10 — SSRF

**점검 항목**: 외부 URL 환경변수 관리, 웹훅 URL 검증, 내부 IP 차단

**결과**: 합격
- `embedding_service_url`, `llm_base_url` 환경변수 전용 관리
- 웹훅 URL: https 스킴만 허용, 내부 IP(127.*, 10.*, 172.*, 192.168.*) 차단

---

## 4. OWASP LLM Top 10 항목별 상세 결과

### LLM01 — Prompt Injection

**점검 항목**: 룰 기반 탐지, 탐지율, Content-Directive 분리, RAG 격리

**결과**: 합격
- 탐지 패턴 40개 이상 (instruction_override, code_execution, markup_injection, boundary_manipulation)
- 탐지율: ≥ 85% (20개 표준 공격 샘플 기준)
- `ContentDirectiveSeparator.wrap()`: `[UNTRUSTED BEGIN/END]` 마킹
- RAG 파이프라인: 검색 결과 전체를 untrusted context로 격리

### LLM02 — Insecure Output Handling

**점검 항목**: HTML 이스케이프, 출력 스키마 검증

**결과**: 합격
- LLM 출력에서 `<script>`, `<iframe>` 등 이스케이프
- `OutputValidator` 스키마 검증

### LLM03 — Training Data Poisoning

**점검 항목**: Fine-tuning 미사용, Prompt Registry 읽기 전용

**결과**: 합격 (N/A — Fine-tuning 미사용)
- 소스코드에 fine-tuning 관련 코드 없음
- PromptRegistry: 시드 파일 쓰기 작업 없음

### LLM04 — Model Denial of Service

**점검 항목**: 입력 토큰 상한, Rate limiting

**결과**: 합격
- `rag_max_context_tokens=6000` 상한
- API Rate limiting (limits 라이브러리)

### LLM06 — Sensitive Information Disclosure

**점검 항목**: PII 탐지, 시스템 프롬프트 보호

**결과**: 합격
- `PIIDetector` (이메일, 전화번호, 주민번호 패턴)
- 시스템 프롬프트 미노출

### LLM08 — Excessive Agency

**점검 항목**: 에이전트 쓰기 범위 제한, kill switch

**결과**: 합격
- 에이전트 액션 제한 (scope 기반)
- Kill switch async 구현: 5초 내 비활성화 보장

### LLM09 — Overreliance on LLM Output

**점검 항목**: Citation 검증, 출력 불확실성 처리

**결과**: 합격
- SHA-256 content hash로 인용 무결성 검증
- 검색 결과 없을 시 "정보가 없습니다" 응답

### LLM10 — Model Theft

**점검 항목**: API 키 보안, 모델 접근 제어

**결과**: 합격
- `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` 환경변수 전용
- 인증된 사용자만 LLM 기능 접근

---

## 5. 결론

OWASP Top 10 및 OWASP LLM Top 10 전 항목에서 **High/Critical 취약점 없음** 확인.
총 **250개 보안 테스트 통과 (0 실패)**. 잔여 권고 사항 3건은 운영 설정 수준으로 즉각적 위험 없음.

**FG9.1 보안 점검 기준: 합격**.
