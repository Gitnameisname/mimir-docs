# Phase 13 보안 취약점 검사 보고서

**작성일**: 2026-04-10  
**검사 범위**: Phase 13 신규 코드 + 전체 코드베이스 OWASP Top 10 기준  
**검사 방법**: 정적 분석 (AST 파싱, 패턴 검색), 수동 코드 리뷰  

---

## 1. 검사 결과 요약

| 분류 | 건수 |
|------|------|
| 발견 취약점 (신규) | 4건 |
| 허위 경보 (False Positive) | 1건 |
| 수정 완료 | 4건 |
| 잔존 취약점 | 0건 |

---

## 2. OWASP Top 10 항목별 검사 결과

| OWASP 항목 | 검사 결과 | 비고 |
|-----------|-----------|------|
| A01 — 접근 제어 실패 | ✅ 통과 | 모든 API에 require_admin_access / require_authenticated 적용 |
| A02 — 암호화 실패 | ✅ 통과 | HTTPS 강제(HSTS), JWT HS256 서명, PBKDF2 기반 세션 |
| A03 — 인젝션 | ✅ 통과 | AST 분석 26개 f-string 패턴 → 전원 화이트리스트 제어 Safe Pattern (상세: §4) |
| A04 — 불안전한 설계 | ✅ 통과 | API-first, 계층 분리, 역할 기반 접근 제어 |
| A05 — 보안 설정 오류 | ⚠️ 수정 | VULN-P13-03 .gitignore 없음, VULN-P13-04 포트 외부 노출 → 수정 완료 |
| A06 — 취약 컴포넌트 | ✅ 통과 | Bandit CI 스캔 통합, 의존성 버전 명시 |
| A07 — 인증 실패 | ⚠️ 수정 | VULN-P13-02 request_id 길이 미검증 → 수정 완료 |
| A08 — 소프트웨어 무결성 | ✅ 통과 | SHA-256 백업 체크섬, Docker 이미지 다이제스트 활용 |
| A09 — 로깅 실패 | ✅ 통과 | 구조화 로그, 민감 정보 마스킹, 감사 로그 |
| A10 — SSRF | ✅ 통과 | 외부 URL 요청 없음 (내부 API 전용) |

---

## 3. 발견 취약점 상세

---

### VULN-P13-01 ⚠️ Medium — X-Forwarded-For 스푸핑 → Metrics Endpoint IP 검증 우회

| 항목 | 내용 |
|------|------|
| 파일 | `backend/app/api/v1/system.py` |
| 발견 경위 | 수동 코드 리뷰 — metrics endpoint IP 필터 로직 분석 |
| OWASP | A01 (접근 제어 실패) |

**취약점 내용**

```python
# 취약 코드 (수정 전)
forwarded_for = request.headers.get("X-Forwarded-For", "").split(",")[0].strip()
effective_ip = forwarded_for or client_ip  # XFF가 있으면 client_ip 무시
```

`X-Forwarded-For` 헤더를 `client_ip`보다 우선 신뢰하는 로직은 리버스 프록시 없이 서비스가 직접 노출된 경우 공격자가 다음과 같이 IP 검증을 우회할 수 있습니다:

```
GET /api/v1/system/metrics HTTP/1.1
Host: mimir.example.com
X-Forwarded-For: 127.0.0.1  ← 조작
```

실제 `client_ip`가 `203.0.113.5` (외부 IP)여도 XFF가 `127.0.0.1`이면 내부로 판단하여 Prometheus 메트릭을 노출합니다.

**공격 영향**

Prometheus 메트릭에는 API 경로 패턴, 요청 수, 오류율, 응답 시간 분포가 포함됩니다. 공격자는 이를 통해 서비스 내부 구조, 취약한 엔드포인트, 이상 징후를 탐지하는 데 활용할 수 있습니다.

**수정 내용**

```python
# 수정 후: client_ip가 내부 IP(프록시)인 경우에만 XFF 신뢰
if any(client_ip.startswith(p) for p in _INTERNAL_PREFIXES):
    # 신뢰된 프록시 경유 — XFF 첫 번째 항목 사용
    forwarded_for = request.headers.get("X-Forwarded-For", "").split(",")[0].strip()
    effective_ip = forwarded_for or client_ip
else:
    # 직접 연결 — XFF 무시 (스푸핑 방지)
    effective_ip = client_ip
```

---

### VULN-P13-02 ⚠️ Low — X-Request-ID 길이 미검증 → 로그 폭탄(Log Bloat)

| 항목 | 내용 |
|------|------|
| 파일 | `backend/app/api/context/middleware.py` |
| 발견 경위 | 수동 코드 리뷰 — request_id 처리 흐름 분석 |
| OWASP | A07 (인증 실패 / 입력 검증) |

**취약점 내용**

```python
# 취약 코드 (수정 전)
request_id = request.headers.get("X-Request-ID") or str(uuid4())
```

`X-Request-ID` 헤더를 길이 제한 없이 수락합니다. 공격자가 수십 KB의 긴 문자열을 `X-Request-ID`로 전송하면, 해당 값이 모든 요청 로그에 기록되어 로그 저장소를 과부하시킬 수 있습니다 (Log Bloat DoS).

**참고**: `JsonFormatter`에서 `json.dumps()`가 문자열을 이스케이프하므로 JSON 인젝션은 불가능합니다. 로그 폭탄만 가능합니다.

**수정 내용**

```python
# 수정 후: 128자 초과 시 잘라냄
_MAX_ID_LEN = 128
raw_request_id = request.headers.get("X-Request-ID", "")
request_id = raw_request_id[:_MAX_ID_LEN] if raw_request_id else str(uuid4())
```

---

### VULN-P13-03 ⚠️ Low — .gitignore 미존재 → 시크릿 파일 실수 커밋 위험

| 항목 | 내용 |
|------|------|
| 파일 | `.gitignore` (신규 생성) |
| 발견 경위 | 파일 시스템 점검 |
| OWASP | A02 (암호화 실패 / 시크릿 노출) |

**취약점 내용**

프로젝트 루트에 `.gitignore`가 존재하지 않습니다. 개발자가 실수로 `git add .`를 실행하면 다음 파일들이 원격 저장소에 커밋될 수 있습니다:

- `.env` (DB 비밀번호, JWT_SECRET, API 키 포함)
- `*.pem`, `*.key` (인증서, 개인키)
- `__pycache__/` (바이트코드)
- `node_modules/`

**수정 내용**

`.gitignore` 파일을 신규 생성하여 다음 항목을 명시적으로 제외:
- `.env`, `.env.local`, `.env.*.local`
- `*.pem`, `*.key`, `*.crt`, `secrets/`
- `__pycache__/`, `.venv/`, `node_modules/`, `.next/`
- `*.dump`, `*.dump.sha256` (DB 백업 파일)
- `*.log`

---

### VULN-P13-04 ⚠️ Low — 개발환경 DB/Valkey 포트 0.0.0.0 바인딩 → 외부 노출

| 항목 | 내용 |
|------|------|
| 파일 | `docker-compose.yml` |
| 발견 경위 | Docker Compose 포트 바인딩 설정 검토 |
| OWASP | A05 (보안 설정 오류) |

**취약점 내용**

```yaml
# 취약 설정 (수정 전)
ports:
  - "5432:5432"   # PostgreSQL — 0.0.0.0:5432 바인딩
  - "6379:6379"   # Valkey    — 0.0.0.0:6379 바인딩
```

클라우드 VM(AWS EC2, GCP 등) 또는 공유 네트워크 환경에서 개발 중일 때 방화벽 설정 실수로 PostgreSQL(5432)과 Valkey(6379)가 외부에서 접근 가능해집니다:

- **PostgreSQL**: 비밀번호가 `devpassword`인 개발 DB에 외부에서 직접 접속 가능 → 데이터 유출
- **Valkey(Redis)**: 개발환경 Valkey는 `requirepass` 없음 → 인증 없이 세션/캐시 데이터 조회, 명령 실행 가능

**수정 내용**

```yaml
# 수정 후: localhost만 바인딩
ports:
  - "127.0.0.1:5432:5432"
  - "127.0.0.1:6379:6379"
```

---

## 4. AST 정적 분석 — f-string SQL 패턴 (False Positive 26건)

**검사 대상**: `execute()` 호출에서 f-string 사용 26개

```
app/repositories/users_repository.py (6건)
app/api/v1/admin.py (13건)
app/api/v1/vectorization.py (2건)
app/services/vectorization_service.py (2건)
app/services/rag_service.py (1건)
app/services/search_service.py (2건)
```

**분석 결과: 전원 안전 (False Positive)**

모든 사례가 다음 두 가지 패턴 중 하나에 해당합니다:

**패턴 A — Safe Dynamic WHERE**
```python
conditions = []
params = []
if search:
    conditions.append("(display_name ILIKE %s)")  # 고정 문자열
    params.append(f"%{search}%")                   # 사용자 입력은 파라미터로
where = ("WHERE " + " AND ".join(conditions)) if conditions else ""
cur.execute(f"SELECT * FROM users {where}", params)  # WHERE절은 고정 조각들의 조합
```

**패턴 B — Safe Dynamic SET (UPDATE)**
```python
fields = []
params = []
if display_name is not None:
    fields.append("display_name = %s")   # 고정 문자열
    params.append(display_name)           # 사용자 입력은 파라미터로
cur.execute(f"UPDATE users SET {', '.join(fields)} WHERE id = %s", params)
```

**패턴 C — Safe IN Clause**
```python
placeholders = ",".join(["%s::uuid"] * len(doc_ids))  # 사용자 입력 제외
cur.execute(f"SELECT * FROM documents WHERE id IN ({placeholders})", doc_ids)
```

**결론**: f-string으로 삽입되는 내용은 모두 하드코딩된 SQL 조각이며, 사용자 입력은 항상 파라미터(`params`)로 전달됩니다. SQL 인젝션 위험 없음.

---

## 5. 추가 보안 검사 항목

| 항목 | 검사 결과 | 내용 |
|------|-----------|------|
| 하드코딩 자격증명 | ✅ 없음 | 모든 시크릿은 `settings.xxx` 환경 변수 경유 |
| HTML 응답 XSS | ✅ 없음 | HTMLResponse 사용 없음, API 전용 JSON/텍스트 응답 |
| 열린 리다이렉트 | ✅ 없음 | RedirectResponse 사용 없음 |
| 파일 업로드 취약점 | ✅ 없음 | UploadFile 사용 없음 |
| eval/exec 사용 | ✅ 없음 | 코드 인젝션 위험 없음 |
| pickle 역직렬화 | ✅ 없음 | pickle 사용 없음 |
| subprocess shell=True | ✅ 없음 | 명령 인젝션 위험 없음 |
| 세션 토큰 생성 | ✅ 안전 | `secrets.token_urlsafe(32)` 사용 |
| JWT 서명 알고리즘 | ✅ 안전 | HS256, algorithms 명시적 지정 |
| CORS 설정 | ✅ 안전 | 와일드카드 제거, 명시적 origins |
| Rate Limiting | ✅ 적용 | slowapi 기반 엔드포인트별 제한 |
| Redis SCAN 사용 | ✅ 안전 | `scan_iter()` 사용 (KEYS 명령 대신) |

---

## 6. 수정 완료 파일 목록

| 파일 | 수정 내용 |
|------|----------|
| `backend/app/api/v1/system.py` | VULN-P13-01: XFF 스푸핑 방지 — client_ip 우선 확인 후 프록시 경유 시에만 XFF 신뢰 |
| `backend/app/api/context/middleware.py` | VULN-P13-02: X-Request-ID 128자 길이 제한 추가 |
| `.gitignore` | VULN-P13-03: .env, 시크릿, 캐시, 로그 파일 git 제외 규칙 |
| `docker-compose.yml` | VULN-P13-04: PostgreSQL/Valkey 포트를 127.0.0.1로 바인딩 제한 |

---

## 7. 결론

Phase 13 보안 취약점 검사에서 **4건의 취약점**이 발견되었으며, 모두 이번 검사에서 즉시 수정되었습니다.

- AST 정적 분석에서 발견된 f-string SQL 패턴 26건은 전원 안전한 패턴으로 확인되어 False Positive 처리되었습니다.
- OWASP Top 10 전 항목을 검사한 결과, 수정 후 기준 취약점 0건입니다.

**Phase 0~13 전체 보안 취약점 누적 수정 현황:**

| Phase | 보안 수정 건수 |
|-------|---------------|
| Phase 3~7 | 15건 (VULN-001~015) |
| Phase 8 | 10건 |
| Phase 9 | 3건 |
| Phase 10 | 7건 |
| Phase 11 | 8건 |
| Phase 13 검수 | 1건 |
| Phase 13 보안검사 | 4건 |
| **누계** | **48건** |
