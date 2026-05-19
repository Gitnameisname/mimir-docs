# Task 7-1 — Valkey 인프라 정비 + 네임스페이스 + 폐쇄망 fallback

**작성일**: 2026-05-18
**FG**: 7-1
**Handoff Level**: `extended` — 인프라 변경 (cluster-wide cache 기반)
**Approver**: `@최철균` + 운영자
**선행**: Phase 6 공식 종결 (2026-05-18)
**관련 모듈**: `backend/app/cache/`, `backend/app/config.py`

---

## 1. 의도 (Intent)

S3 Phase 3~6 동안 누적된 Valkey 사용 패턴 — `response_cache`, `rate_limit` storage, admin settings cache 등 — 을 **공식 인프라 모듈**로 통합하고, 다음 두 잔존 결함을 해결한다.

- **결함 A — 키 namespace 충돌 가능성**: 현 `mimir:` prefix 만으로는 환경(dev / staging / prod) 분리가 코드 contract 수준에서 강제되지 않음. R-I4 요구.
- **결함 B — 폐쇄망 fallback 미공식화**: `VALKEY_HOST` 미설정 시 동작이 호출지마다 다름 (response_cache 만 in-memory fallback). R-I1 요구.
- **결함 C — fail-open vs fail-closed 분류 부재**: feature 별 정책이 코드에 분산. R-I2 요구.

본 Task 는 새 모듈 `app.cache.namespace` / `app.cache.policy` 를 신설하고, `app.cache.valkey` 의 비활성 모드(off-mode)를 명문화한다.

---

## 2. 범위 (Scope)

### 2.1 변경 — IN

| 파일 | 변경 |
|------|------|
| `backend/app/cache/namespace.py` (신규) | `make_key(feature, *parts)` + `make_channel(feature, *, org_id=None)` — `mimir:<env>:<feature>:<args>` 합성 |
| `backend/app/cache/policy.py` (신규) | `FailPolicy` enum (`FAIL_OPEN` / `FAIL_CLOSED`) + feature → policy 맵 + 헬퍼 |
| `backend/app/cache/valkey.py` (수정) | `is_valkey_disabled()` 노출 + `VALKEY_HOST` 비어있을 때 client `None` 반환 + `get_valkey_or_none()` 헬퍼 |
| `backend/app/config.py` (수정) | `valkey_namespace` property (`mimir:<env>:`) + `valkey_disabled` property |
| `backend/tests/unit/cache/test_namespace.py` (신규) | 키/채널 합성 + namespace 격리 단위 테스트 |
| `backend/tests/unit/cache/test_policy.py` (신규) | fail-open / fail-closed 분류 단위 테스트 |
| `backend/tests/unit/cache/test_valkey_disabled_mode.py` (신규) | `VALKEY_HOST=""` / `VALKEY_DISABLED=1` 시 client `None` 반환 + 회귀 호출자가 fallback 경로 사용 |

### 2.2 변경 — OUT (이번 Task 에서 손대지 않음)

- `viewed_throttle.py` (FG 7-2 별 Task)
- `scope_profile_policy.py` (FG 7-3 별 Task)
- `response_cache.py` (이미 fallback 있음 — 본 Task 영향 없음. namespace 만 추후 통일 옵션)
- `rate_limit.py` (slowapi storage_uri 자체 처리. 본 Task 영향 없음)

---

## 3. 설계 (Design)

### 3.1 `app.cache.namespace`

```python
def make_key(feature: str, *parts: str | int) -> str:
    """`mimir:<env>:<feature>:<part1>:<part2>...` 합성.

    - env 는 settings.environment (dev/staging/prod/test)
    - 다른 환경 키 충돌 방지 (R-I4)
    """

def make_channel(feature: str, *, org_id: Optional[str] = None) -> str:
    """`mimir:<env>:tenant:<org_id>:cache:invalidate:<feature>` 합성.

    - org_id None 이면 cluster-wide (테넌트 격리 없음)
    - org_id 있으면 tenant prefix 강제 (R-I3)
    """
```

### 3.2 `app.cache.policy`

```python
class FailPolicy(str, Enum):
    FAIL_OPEN = "fail_open"        # Valkey 장애 → 워커별 fallback (성능)
    FAIL_CLOSED = "fail_closed"    # Valkey 장애 → 캐시 비움 → DB 재조회 (보안)

# Feature 별 정책 (환경변수로 override 가능)
_DEFAULT_POLICY: dict[str, FailPolicy] = {
    "viewed_throttle": FailPolicy.FAIL_OPEN,
    "rate_limit": FailPolicy.FAIL_OPEN,
    "scope_policy": FailPolicy.FAIL_CLOSED,
    "response_cache": FailPolicy.FAIL_OPEN,
    "admin_settings": FailPolicy.FAIL_OPEN,
}

def policy_for(feature: str) -> FailPolicy: ...
def is_fail_open(feature: str) -> bool: ...
def is_fail_closed(feature: str) -> bool: ...
```

환경변수 override 형식:
- `VALKEY_FAIL_OPEN_FEATURES=viewed_throttle,rate_limit` — comma-separated
- `VALKEY_FAIL_CLOSED_FEATURES=scope_policy`
- override 가 있으면 default 무시. 둘 다 지정된 feature 는 **fail-closed 우선** (보안 보수).

### 3.3 `app.cache.valkey` 비활성 모드

```python
def is_valkey_disabled() -> bool:
    """다음 중 하나면 True:
    - VALKEY_DISABLED=1 (명시적)
    - VALKEY_HOST="" (폐쇄망 / 단일 노드 환경)
    """

def get_valkey_or_none() -> redis.Redis | None:
    """disabled 모드 또는 init 실패 시 None.
    호출자는 fallback 경로를 명시적으로 처리.
    """
```

### 3.4 환경변수

```
VALKEY_HOST=valkey            # 비어있으면 disabled mode
VALKEY_PORT=6379
VALKEY_PASSWORD=
VALKEY_DB=0
VALKEY_DISABLED=              # "1" 이면 강제 disable (폐쇄망 명시)
VALKEY_NAMESPACE=             # 미설정 시 mimir:<environment>
VALKEY_FAIL_OPEN_FEATURES=    # comma-separated override
VALKEY_FAIL_CLOSED_FEATURES=
```

---

## 4. 절대 규칙 (R-I) 매핑

| 규칙 | 본 Task 대응 |
|------|------|
| **R-I1 폐쇄망 호환** | `is_valkey_disabled()` 공식화. 호출자가 명시적 fallback 경로 보유 |
| **R-I2 fail-open / fail-closed** | `FailPolicy` enum + 분류 맵 + env override |
| **R-I3 pub/sub 권한** | `make_channel(org_id=...)` 에 tenant prefix 강제 — pub/sub 구현은 FG 7-3 |
| **R-I4 키 namespace** | `make_key()` 가 `mimir:<env>:` prefix 강제. namespace 격리 회귀 |

---

## 5. 회귀 / 검증

### 5.1 단위 테스트 (신규)

- `test_namespace.py`:
  - `mimir:test:viewed:user-A:doc-1` 형태로 합성
  - 다른 환경 (`dev` vs `prod`) 충돌 없음
  - org_id 지정 시 채널에 tenant prefix 강제
  - org_id 미지정 시 cluster-wide 채널
- `test_policy.py`:
  - 기본 분류 확인
  - env override 동작
  - 둘 다 지정 시 fail-closed 우선
  - 미등록 feature → fail-closed (보수 default)
- `test_valkey_disabled_mode.py`:
  - `VALKEY_DISABLED=1` 시 `get_valkey_or_none()` → None
  - `VALKEY_HOST=""` 시 `get_valkey_or_none()` → None
  - 정상 모드 시 redis.Redis 인스턴스 반환 (mock)

### 5.2 회귀 게이트

- 기존 `response_cache` / `rate_limit` / admin settings cache 회귀 모두 녹색 유지
- `pytest backend/tests/unit/` 전체 베이스라인 유지

---

## 6. 산출물

| 산출물 | 위치 |
|--------|------|
| 코드 변경 | `backend/app/cache/` + `backend/app/config.py` |
| 단위 테스트 | `backend/tests/unit/cache/` (신규 디렉토리) |
| 검수보고서 | `docs/개발문서/S3/phase7/산출물/FG7-1_검수보고서.md` |
| 인프라 가이드 | `docs/개발문서/S3/phase7/산출물/FG7-1_인프라가이드.md` |
| 함수도서관 등록 | `docs/함수도서관/backend.md` §1.11-fg71 |

---

## 7. 위험

- **R-1** 기존 호출자가 `get_valkey()` 가 항상 instance 반환을 가정 → 별 helper `get_valkey_or_none()` 신설로 기존 API 호환 보존. 단 disabled 모드 시 `get_valkey()` 는 여전히 instance 반환 (호출 시 connection error).
- **R-2** namespace 변경이 운영중 기존 키와 충돌 → 본 Task 는 `mimir:` prefix 유지 + `mimir:<env>:` 강제는 **신규 호출자만**. 기존 키 패턴은 그대로 유지.

---

## 8. 완료 기준

- [ ] `app.cache.namespace` / `app.cache.policy` 모듈 생성
- [ ] `app.cache.valkey.is_valkey_disabled()` / `get_valkey_or_none()` 노출
- [ ] 단위 테스트 신규 (≥ 12 건) 녹색
- [ ] 기존 pytest 베이스라인 녹색
- [ ] 함수도서관 §1.11-fg71 등록
- [ ] FG7-1_검수보고서.md + FG7-1_인프라가이드.md 작성

---

*Owner: Claude (Design + Implementation). Review: Codex 또는 사람. Final approval: @최철균 + 운영자.*
