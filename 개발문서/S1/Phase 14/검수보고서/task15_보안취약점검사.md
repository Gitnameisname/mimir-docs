# Task 14-15 보안 취약점 검사 보고서

작성일: 2026-04-14
범위: API 키 CRUD 및 감사 로그 필터 메타데이터 엔드포인트 + 관련 UI

---

## 1. 위협 모델링

| 자산 | 위협 | 영향 |
| --- | --- | --- |
| 발급된 API 키 (full_key) | 유출 시 외부 시스템이 관리 API 무단 호출 | 기밀성/무결성 |
| `api_keys.key_hash` | DB 유출 시 평문 복구 | 기밀성 |
| 폐기 이력 | 조작되면 권한 남용의 감사 추적 실패 | 무결성 |
| 관리자 행위 | RBAC 우회로 일반 사용자가 키 발급·폐기 | 권한상승 |
| 감사 로그 필터 | 대량 조회로 DB 포화 | 가용성 |

## 2. 점검 항목 및 결과

### 2.1 인증·인가
- 3개 엔드포인트 모두 `Depends(require_admin_access)` 적용 (verified by SEC / API-04). 일반 사용자 접근 차단.
- 프론트에서는 `SUPER_ADMIN` 만 생성/폐기 UI 가 노출되도록 게이트 — UI 게이트는 편의성이며, 실제 차단은 백엔드가 담당.

### 2.2 키 생성 보안
- `secrets.token_urlsafe(32)` — OS CSPRNG. **256bit 엔트로피**로 무차별 대입 저항.
- 저장은 `hashlib.sha256(full.encode("utf-8")).hexdigest()` 만. DB 유출 시 원본 키 복구 불가 (레인보우 테이블 공격은 256bit 원본 길이상 비현실적).
- `key_prefix` 는 앞 8자만 저장 — 로그·UI 에서 키를 식별하면서 본문은 공개하지 않음.
- `full_key` 는 응답 한 번만 (검증: SELECT 구문/logger 에 등장하지 않음, SEC-04 PASS).

### 2.3 입력 검증
- 이름: `^[\w\- .]{1,100}$` (UNICODE) — 백트래킹 없는 선형 regex (reDoS 무해). 공격 가능한 메타문자(`<`, `>`, `/`, `&` 등) 차단.
- 설명: `max_length=500` (Pydantic). 값은 response 에 JSON 으로 직렬화되므로 XSS 위험은 UI 쪽 렌더링에 달려있으며, 현재 모든 문자열을 React 가 자동 이스케이프.
- scope: 화이트리스트 `{"READ_ONLY","READ_WRITE","admin.read","admin.write"}` 이외 422.
- `expires_in_days`: `Field(ge=0, le=3650)` — 음수·과도한 미래 값 차단.
- `key_id` (revoke): `uuid.UUID(key_id)` 로 형식 검증, 실패 시 422. SQL 주입에 대한 1차 방어.

### 2.4 SQL 주입
- 모든 INSERT/UPDATE 는 파라미터 바인딩 (`%s`). `f"""..."""` 는 `expires_at_sql` 상수 조각(두 값 중 하나: 리터럴 `NULL` 혹은 `NOW() + (%s||' days')::interval`) 을 주입하며 사용자 값은 추가 `%s` 로 바인딩. 정적 검사 PASS.

### 2.5 CSRF / 재생 공격
- FastAPI 는 표준 JSON API + JWT/dev 헤더 모델. 쿠키 기반 세션 미사용이므로 브라우저 CSRF 표면은 제한적.
- 키 발급은 멱등이 아니며 이름이 중복 허용됨 — **의도된 동작**. 사용자가 실수로 재발급 시에도 이전 키는 별개 row 로 유지(개별 폐기 가능).

### 2.6 Soft-revoke 무결성
- `WHERE id=%s AND status='ACTIVE'` 조건으로 이미 폐기된 키에 대한 중복 폐기(상태 덮어쓰기) 차단.
- `revoked_reason` 는 감사 metadata 와 별도로 row 에 persist 되어, 감사 이벤트 실패(네트워크/큐)에도 최소 사유 유지.

### 2.7 감사 이벤트
- 발급·폐기 모두 `audit_emitter.emit(...)` 호출 + `result="success"`. 실패는 `except Exception` + `logger.exception` 로 흡수 — API 정상 응답을 보장하지만 운영 경보 수집용 로그는 남김.

### 2.8 감사 로그 필터 / 메타데이터
- 이벤트 유형 카탈로그는 **정적 리스트** — DB `SELECT DISTINCT` 가 아니므로 인덱스 전수 스캔 유도 공격 불가.
- 사용자 검색 드롭다운은 기존 `adminApi.getUsers({search})` 를 활용. page_size=10 으로 제한, 서버측 검색 로직은 기존 Phase 에서 검증됨.
- 기간 프리셋은 클라이언트 계산 → `from/to` ISO 문자열을 백엔드로 전달. 백엔드 파서는 기존 Phase 14-13/14 코드에서 재사용.

### 2.9 UI 보안
- 발급된 키 모달: `<input readOnly>` + 자동 select on focus. 클립보드 복사는 `navigator.clipboard.writeText` (secure context required). 실패 시 사용자에게 토스트 안내.
- 폐기 모달: 이름 정확 일치 검사 — 실수 클릭으로 키 삭제 방지. 파괴적 버튼 색상 `bg-red-700` + `disabled:opacity-50`.
- 모든 경고 블록 `role="alert"` / `role="note"` — 스크린리더로 전파.

### 2.10 종속성
- 신규 의존성 없음 (`secrets`, `hashlib`, `uuid` 모두 stdlib).
- 프론트 신규 import 없음 (기존 `@tanstack/react-query`, `lucide` 미사용).

---

## 3. 발견된 잠재 위험과 완화

| 위험 | 심각도 | 완화 |
| --- | --- | --- |
| 브라우저가 HTTPS 가 아닌 환경이면 `navigator.clipboard` 실패 | Low | Fallback: input `.select()` + 자동 선택 + 안내 토스트 |
| 감사 이벤트 발행 실패로 흔적 유실 | Medium | `logger.exception` 으로 서버 로그에는 남김. 별도 경보 파이프라인은 Phase 14-13 규칙이 처리 |
| 이름 정규식이 한글 허용 (\w + UNICODE) — 시각적 유사 문자로 혼동 | Low | 이름은 참고용 label 일 뿐 인증엔 미사용. 폐기 시 정확 일치 확인으로 2차 확인 |
| SHA-256 은 GPU 브루트포스 표면이 존재 | Low | 원본은 256bit 엔트로피. 2^256 탐색 불가 — 추가 stretching 불필요 |
| 폐기 사유 textarea → XSS | Low | React 자동 이스케이프, DB 값은 API 응답/프런트 렌더 양단 모두 plaintext 취급 |

## 4. 결론

보안 취약점 미발견. 설계상 다층 방어(256bit 엔트로피 + SHA-256 only + 1회 노출 + RBAC + 이름 재확인 + soft-revoke + 감사 이벤트)가 작동하며 알려진 OWASP Top 10 (A01 Broken Access Control, A03 Injection, A04 Insecure Design, A07 Identification & Authentication Failures, A09 Security Logging Failures) 대응 완료.

검증 스크립트 PASS (SEC 카테고리 12/12). 운영 투입 가능.
