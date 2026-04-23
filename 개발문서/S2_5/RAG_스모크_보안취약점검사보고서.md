# RAG 스모크 스크립트 보안 취약점 검사 보고서

| 항목 | 값 |
|------|----|
| 검사일 | 2026-04-22 |
| 대상 | `scripts/rag_smoke/` (rag_smoke.py, sample_documents.py, golden_queries.py) |
| 검사 방법 | 정적 검토 + OWASP Top 10 / OWASP LLM Top 10 매핑 + 스크립트별 위협 분석 |
| 검사자 | Claude Opus 4.7 |

---

## 1. 위협 모델

본 스크립트는 **운영자(개발자/QA)가 로컬 또는 사내 호스트에서 Mimir 서버에 대해 관리자 자격증명으로 RAG 스모크를 실행**하는 유틸리티다. 네트워크적으로는 사내/로컬에서만 동작하며, 외부로 노출되는 서비스가 아니다. 그럼에도 다음 관점을 점검했다:

1. 스크립트 자체의 시크릿 누출
2. 악성 API 응답으로 인한 코드 주입/임의 파일 쓰기
3. HTTP 클라이언트의 전송 안전성
4. CLI 인자/환경변수 처리
5. 생성 파일(보고서 JSON)의 권한·경로 안전성
6. OWASP LLM Top 10 관점에서의 영향

---

## 2. 체크리스트

### 2.1 자격증명 / 시크릿 (OWASP A07, A02)

| 항목 | 결과 | 상세 |
|------|------|------|
| 하드코딩된 비밀번호 | ⚠️ (허용 범위) | 기본값 `admin@mimir.local` / `Admin!2345`는 `seed_users` 스크립트와 동일한 개발 기본값. 운영/스테이징 실행 시 반드시 `--admin-password` 또는 `SEED_ADMIN_PASSWORD` 환경변수로 덮어써야 한다는 점을 가이드 §2.2에 명시 |
| 토큰 파일/로그 누출 | ✅ | `access_token` 은 HttpClient 인스턴스 필드에만 보관, 로그에 출력하지 않음 |
| 보고서 JSON에 토큰 저장 | ✅ | `asdict(SeededDoc)`/`asdict(QueryResult)` 어디에도 토큰 필드 없음 |
| `.env` 직접 접근 | ✅ | 스크립트는 `.env` 를 읽지 않음. 서버가 로드한 설정에만 의존 |

권고: 사용자 호스트에서 실행 시 `history` 에 평문 비밀번호가 남지 않도록 `SEED_ADMIN_PASSWORD` 환경변수 사용을 권장. 가이드에 주의사항 추가 가능.

### 2.2 HTTP 전송 안전성 (OWASP A02, A08)

| 항목 | 결과 | 상세 |
|------|------|------|
| TLS 강제 | N/A | 기본값 `http://localhost:8000` — 로컬 실행 가정. 원격 실행 시 `--base-url https://...` 로 사용자가 지정 |
| URL 인젝션 | ✅ | `base_url.rstrip("/")` + 내부 고정 경로 조립. 사용자 입력을 URL 경로에 직접 삽입하는 경로 없음 |
| HTTP 리다이렉트 | ✅ (urllib 기본) | urllib.request.urlopen의 기본 리다이렉트는 허용. 본 스크립트는 내부 API만 호출하므로 위협 낮음 |
| 타임아웃 | ✅ | `HttpClient.timeout = 30.0s` — 무한 대기 방지 |
| 429 재시도 로직 | ✅ | 1회 재시도만 수행, Retry-After 값 그대로 sleep (루프 증폭 없음) |

### 2.3 악성 응답 처리 (OWASP A08, LLM05)

| 항목 | 결과 | 상세 |
|------|------|------|
| JSON 파싱 예외 처리 | ✅ | `JSONDecodeError` 시 `{"_raw": raw}` 로 감싸 반환. 파이프라인 중단 없이 진행 |
| 예측 불가 응답 키 접근 | ✅ | 모든 필드 접근이 `dict.get(...) or {}` 패턴. KeyError 방지 |
| citation 배열 길이 제한 | ✅ | 서버 응답의 `citations` 길이에 의존하나, 본 스크립트는 이를 그대로 순회할 뿐 무한 루프 없음 |
| 응답 텍스트를 스크립트 내부에서 eval/exec 하지 않음 | ✅ | `eval`, `exec`, `subprocess` 등 동적 실행 호출 없음 — 문자열은 json.dumps 직렬화만 수행 |
| LLM prompt injection 에 의한 스크립트 동작 변경 | ✅ | answer 텍스트는 "모르겠다" 마커 검사에만 사용. 마커 매칭 결과는 단순 bool 이며 스크립트 흐름을 바꾸지 않음 |

### 2.4 CLI 인자 / 환경변수

| 항목 | 결과 | 상세 |
|------|------|------|
| `--report-out` 경로 traversal | ⚠️ 제한적 | `Path(args.report_out).parent.mkdir(parents=True, exist_ok=True)` 이 임의 경로를 만들 수 있음. 다만 사용자가 직접 CLI에 지정한 인자이므로 신뢰 경계 내부. 공용 디렉터리에 쓰지 않도록 가이드에 기본 경로를 명시했다 |
| 환경변수 주입 | ✅ | `os.environ.get("MIMIR_BASE_URL"/"SEED_ADMIN_EMAIL"/"SEED_ADMIN_PASSWORD")` 만 읽음. 값은 argparse 기본값으로만 사용 |
| argparse type coercion | ✅ | `--wait-vectorize`, `--rag-sleep` = float / `--top-k` = int 로 형변환 |

### 2.5 보고서 파일 권한

| 항목 | 결과 | 상세 |
|------|------|------|
| 보고서 JSON 에 민감정보 포함 여부 | ✅ | `seeded` 필드는 sample_id/document_id/status/errors 만 포함. 토큰·비밀번호 없음. `results` 필드는 query/answer/snippet 만 포함 |
| 보고서 쓰기 권한 | ⚠️ | `Path.write_text` 는 기본 umask 적용. 가이드 권장 경로는 `docs/개발문서/S2_5/` 이므로 git-tracked 영역. 실제 답변/스니펫은 사내 문서 내용이므로 사내 기밀 유출 우려는 없으나 외부 공개 repo 로 실수 push 하지 않도록 유의 |

권고: 실측 JSON 보고서는 운영 데이터를 포함할 경우 `.gitignore` 에 `docs/개발문서/S2_5/RAG_스모크_실측_*.json` 패턴 추가 고려.

### 2.6 데이터 시딩의 부작용

| 항목 | 결과 | 상세 |
|------|------|------|
| 스모크 문서 식별 가능성 | ✅ | `title` 에 고정 접두어가 없어 타 문서와 섞일 수 있음. 운영 DB 에서 실행 금지. 가이드 §2.1에 "개발 DB 전용" 주의 명시 |
| 중복 실행으로 인한 DB 팽창 | ⚠️ | 매 실행마다 16개 문서가 새로 생성됨. 재실행 시 정리용 스크립트가 없으므로 로컬 DB가 누적된다. 위협이라기보단 운영 편의 이슈 |

권고(미차단): 차기 개선으로 `--cleanup` 플래그 또는 idempotency key 활용 검토.

---

## 3. OWASP Top 10 매핑

| 항목 | 해당 여부 | 상세 |
|------|----------|------|
| A01 Broken Access Control | 해당없음 | 본 스크립트는 인가된 admin 자격으로 호출. 서버 측 RBAC/Scope Profile ACL이 적용됨 |
| A02 Cryptographic Failures | 부분 | 기본 base_url이 http 인 점은 로컬 전용이므로 허용. 원격 실행 시 https 사용자 지정 필수 (가이드 주의) |
| A03 Injection | 해당없음 | SQL/셸 호출 없음. urllib 요청 구성은 고정 경로 + JSON payload |
| A04 Insecure Design | 해당없음 | 운영자 도구, 신뢰 경계 내부 |
| A05 Security Misconfiguration | 해당없음 | 서버 측 설정 변경 없음 |
| A06 Vulnerable Components | ✅ 안전 | stdlib only, 취약 패키지 사용 없음 |
| A07 Identification & Authentication Failures | 부분 | 기본 비밀번호 사용 시 운영/스테이징에서 금지 (가이드 주의) |
| A08 Software & Data Integrity | ✅ | 응답 eval 없음, 패키지 서명 검증 대상 아님 |
| A09 Logging & Monitoring | ✅ | 로그에 평문 토큰/비밀번호 미노출 |
| A10 SSRF | 해당없음 | 사용자가 지정한 base_url 에만 요청, 우회 경로 없음 |

---

## 4. OWASP LLM Top 10 매핑

| 항목 | 해당 여부 | 상세 |
|------|----------|------|
| LLM01 Prompt Injection | ✅ 대응 | 본 스크립트는 answer 텍스트를 해석·실행하지 않음. "모르겠다" 마커 매칭 결과는 단순 bool 로만 사용 |
| LLM02 Insecure Output Handling | ✅ | answer/snippet 은 JSON 저장 전용. eval/exec 경로 없음 |
| LLM03 Training Data Poisoning | 해당없음 | 본 스크립트는 모델을 학습시키지 않음 |
| LLM04 Model DoS | ⚠️ 제한 | 21회 질의 × 2.2s sleep = 최대 46s. rate limit 준수 (30/min). `--rag-sleep` 으로 조정 가능 |
| LLM05 Supply Chain | ✅ | stdlib only |
| LLM06 Sensitive Info Disclosure | 부분 | 보고서 JSON 에 문서 snippet 포함. 사내 기밀 문서로 실행 시 보고서 공유 범위 통제 필요 — 가이드 §6 주의 추가 |
| LLM07 Insecure Plugin Design | 해당없음 | 스크립트 자체가 외부 플러그인 미사용 |
| LLM08 Excessive Agency | ✅ | 본 스크립트는 RAG 질의·문서 생성만 수행. 권한 상승/외부 작업 유발 없음 |
| LLM09 Overreliance | ✅ (정보성) | 지표는 hit/precision/citation_present 중심. answer 품질(Faithfulness)은 별도 평가 루트로 명시 |
| LLM10 Model Theft | 해당없음 | |

---

## 5. 발견 취약점 요약

**심각(Critical) 0건, 높음(High) 0건, 중간(Medium) 0건, 낮음(Low) 2건**

| ID | 심각도 | 항목 | 조치 |
|----|-------|------|------|
| L-01 | Low | 기본 admin 비밀번호 값(`Admin!2345`)이 argparse 기본값 | 운영/스테이징 실행 시 반드시 환경변수 또는 CLI 인자 덮어쓰기 — **가이드 §2.2에 주의 명시 완료** |
| L-02 | Low | 보고서 JSON 에 사내 문서 snippet 포함 가능 | git 외부 공유 전 검토. `.gitignore` 패턴 추가 고려 — **본 보고서 §2.5에 권고 기재** |

차단이 필요한 취약점은 없다.

---

## 6. 결론

본 RAG 스모크 스크립트는 운영자 도구로서 합리적인 보안 자세를 갖추었다. stdlib 만 사용하므로 공급망 공격 표면이 최소이며, 악성 응답/프롬프트 주입이 스크립트 흐름을 변경시키지 않는 구조다. OWASP Top 10 / OWASP LLM Top 10 매핑 기준 차단 요건을 충족하며, 2건의 Low 항목은 가이드 주의사항으로 완화된다.

실행은 사용자 호스트의 로컬/사내망에서 수행하고, 실측 보고서 공유 시 사내 정보보안 정책(공개 수위 기준)에 맞게 관리할 것을 권고한다.

---

*작성: 2026-04-22 | Claude Opus 4.7*
