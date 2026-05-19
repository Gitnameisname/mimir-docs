# S3 Phase 7 Codex 2차 시정보고서

**작성일**: 2026-05-19
**대상**: `Phase7_Codex_2차_검수보고서_2026-05-19.md` 의 발견 사항 시정
**작성자**: Claude (구현). 별 reviewer (Codex 3차) + @최철균 P1 승인 잔여.

---

## 1. 시정 범위

Codex 2차 검수의 4건 발견 + 1건 권고에 대해 다음과 같이 시정한다.

| # | 등급 | 발견 | 시정 |
|---|---|---|---|
| 1 | **P1** | FG 7-3 fail-closed 요구와 구현 불일치 | **strict fail-closed 구현** + `ADR-FG7-3-fail-closed-정책.md` 작성 |
| 2 | **P1** | 공식 게이트가 실제 통합/chaos 검증 없이 부분 충족 | 운영자 chaos test 영역 — 본 보고서는 단위 회귀 강화로 보강. chaos 자체는 잔여 (O-7-1/O-7-2 — 운영자 합의) |
| 3 | **P2** | subscriber 자동 재구독 부재 | **supervisor thread + backoff 재구독** 구현 |
| 4 | **P3** | namespace segment `:` 처리 불일치 | `_quote_part()` 가 `:` 도 sanitize + 회귀 추가 |
| 5 | **권고** | 전체 unit 4건 실패 (annotation mention) | Phase 5 FG 5-5 (2026-05-14) 변경 반영한 테스트 갱신 |

---

## 2. P1 — FG 7-3 strict fail-closed 구현

### 2.1 시정 내용

`backend/app/services/scope_profile_policy.py` 의 `_get_expose_viewers_for_profile` 에 strict fail-closed gate 도입.

```python
def should_bypass_cache() -> bool:
    """strict fail-closed gate.

    1. is_fail_open("scope_policy") True (운영자 opt-out) → bypass False
    2. is_valkey_disabled() True (단일 워커 모드) → bypass False (안전)
    3. Valkey 설정됨 + subscriber 미연결 → bypass True (strict fail-closed)
    4. 정상 → bypass False
    """
```

`bypass=True` 시 process-local cache lookup + store 모두 skip — 매 호출 DB 재조회.

### 2.2 ADR

`docs/개발문서/S3/phase7/산출물/ADR-FG7-3-fail-closed-정책.md` 작성 — 보안 vs 가용성 trade-off, 단일워커/다중워커 환경 분리, 대안 검토, 위험.

### 2.3 회귀

`tests/unit/test_scope_profile_policy_fg32.py::TestClusterWideInvalidation` 신규 4 case:
- `test_strict_fail_closed_when_valkey_set_but_subscriber_down` — bypass=True, 2회 호출 = 2회 DB hit
- `test_strict_fail_closed_disabled_when_valkey_off` — single-worker → bypass False
- `test_strict_fail_closed_disabled_via_env_override` — operator opt-out → bypass False
- `test_strict_fail_closed_off_when_subscriber_connected` — 정상 → bypass False

기존 14 case 는 본 파일의 autouse fixture 에 `valkey_disabled="1"` 강제 — single-worker 모드 검증.

---

## 3. P1 — 공식 게이트 chaos 검증

본 시정은 **단위 회귀로 강화**하고, 컨테이너 chaos / latency / AUTH 실측은 **운영자 합의 후 별 산출물** 로 유지.

| 게이트 | 본 시정 후 상태 |
|---|---|
| R-I1 폐쇄망 — disabled 시 fallback | 단위 회귀 12 case + viewed_throttle cluster 14 case |
| R-I2 fail-open / closed | strict fail-closed 구현 + ADR + 회귀 4 신규 |
| R-I3 pub/sub 권한 — AUTH 거부 실측 | **운영자 영역** (FG 7-1 인프라가이드 §3.3) — 본 코드는 ACL 없이도 안전 동작 |
| AUTH 없는 client 거부 | **운영자 영역** (Valkey requirepass / ACL 설정) |
| admin PATCH < 1초 invalidate | 단위 publish/dispatch + supervisor reconnect 회귀 |
| Phase 1~6 회귀 녹색 | annotation 4건 시정 — 본 보고서 §5 |

운영자 chaos 합의 전까지 잔여 (O-7-1 / O-7-2) 그대로 유지.

---

## 4. P2 — Subscriber 자동 재구독

### 4.1 시정 내용

`backend/app/cache/pubsub.py::Subscriber` 에 supervisor thread 도입:

- `start()` → supervisor thread 시작 (subscribe 즉시 아님)
- supervisor 가 `_try_subscribe()` 시도 → 실패 시 backoff 후 재시도
- subscribe 성공 → message loop 진입
- loop 가 connection error 로 break → supervisor 가 재구독 사이클
- backoff: 1s → 2s → 4s → 8s → 16s → max 30s (회복 시 1s 리셋)
- `stop()` → supervisor + loop 모두 종료

### 4.2 회귀

`tests/unit/cache/test_pubsub.py::TestSubscriberAutoReconnect` 신규 3 case:
- `test_supervisor_retries_after_failure` — 2회 실패 → 3회째 성공 검증
- `test_supervisor_reconnects_after_loop_error` — message loop error 후 재구독 검증
- `test_stop_clears_state` — stop 후 supervisor 종료 + connected=False

### 4.3 운영 인지 노출

`Subscriber.is_connected()` public method 추가 — 운영자 / admin endpoint 가 연결 상태 조회 가능.

향후 admin health endpoint 에 추가 (별 라운드).

---

## 5. P3 — namespace `:` segment escape

### 5.1 시정 내용

`backend/app/cache/namespace.py::_quote_part()` 의 정규식에 `:` 추가:

```python
# 기존: r"[\s\r\n\x00]"
# 시정: r"[\s\r\n\x00:]"
_INVALID_KEY_CHARS = re.compile(r"[\s\r\n\x00:]")
```

주석도 갱신 — `:` segment 구분자 차단 사유 명시.

### 5.2 회귀

`tests/unit/cache/test_namespace.py` 신규 2 case:
- `test_colon_in_part_is_sanitized` — `user:1` → `user_1`, 키 충돌 회피 검증
- `test_channel_colon_in_org_id_sanitized` — org_id 의 `:` 도 sanitize

---

## 6. 권고 — annotation mention 4건

### 6.1 원인

Phase 5 FG 5-5 (2026-05-14) 가 mention regex 를 한국어 display_name 매칭 지원으로 변경했다:
- 기존: `[a-z][a-z0-9._-]{1,62}` 류 영문 패턴
- 신규: `[^\s@,;.!?…\(\)\[\]{}<>"']{1,64}` — 공백·구두점 외 모든 문자

Phase 3 FG 3-3 의 4 case 는 옛 패턴 기준으로 작성되어 있어 Phase 5 변경 시 함께 갱신했어야 하지만 누락됨.

### 6.2 시정

`tests/unit/test_annotations_service_fg33.py` 의 4 case 를 현재 regex 동작에 맞춰 갱신:

| 옛 case | 신 case (FG 5-5 반영) |
|---|---|
| `test_dot_dash_underscore_allowed` (`first.last` 매치 기대) | `test_dash_underscore_allowed_dot_terminates` (`.` 가 token 종료) |
| `test_must_start_with_letter` (숫자/`_` 시작 거부 기대) | `test_first_char_no_letter_restriction` (한국어 매칭 위해 제약 제거) |
| `test_min_length_2` (1글자 거부 기대) | `test_min_length_1` (1글자 허용 + 호출자 검증 책임) |

`_mock_users` fixture 에 `find_by_display_name_in_viewer_orgs` mock 추가 — FG 5-5 에서 추가된 fallback path 가 mock 미설정 시 truthy MagicMock 반환하여 `test_create_with_mention_unknown_user_skipped` 가 실패하던 문제 해결.

### 6.3 회귀

`tests/unit/test_annotations_service_fg33.py` 전체 41 / 41 ✅

---

## 7. 종합 회귀

| 범위 | 결과 |
|---|---|
| Phase 7 신규 (cache + audit cluster + scope_policy 신규) | 113 / 113 ✅ (이전 101 → +12 신규) |
| Phase 7 영역 + 인접 | 199 / 199 ✅ |
| annotation_service_fg33 | 41 / 41 ✅ (이전 37 / 41 → +4 시정) |
| 전체 unit | 2865 passed / 4 failed (모두 `test_embedding_dim_check.py` — `ModuleNotFoundError: alembic`. 환경 의존, Phase 7 무관) / 13 skipped |

`alembic` 모듈 미설치 4건은 venv 환경 이슈 — 운영 CI 에서는 alembic 설치되어 통과 전망. 별 환경 셋업 확인 잔여 (`backend/requirements.txt` 에 alembic 명시 여부 — 별 단순 확인 후 마감).

---

## 8. 함수도서관 갱신

`docs/함수도서관/backend.md` §1.11-fg73 에 다음 추가 (별 turn — 본 보고서 마감 후):
- `app.cache.pubsub.Subscriber.is_connected()` 노출
- `app.cache.pubsub.Subscriber` 의 supervisor / 재구독 동작 명문화
- `app.cache.namespace._quote_part` 의 `:` sanitize 정책

`app.services.scope_profile_policy.should_bypass_cache()` 도 신규 등록.

---

## 9. 변경 파일 요약

| 파일 | 변경 |
|------|------|
| `backend/app/cache/namespace.py` | `_INVALID_KEY_CHARS` 에 `:` 추가 + 주석 |
| `backend/app/cache/pubsub.py` | Subscriber 클래스에 supervisor + `_try_subscribe` + `_message_loop` + `is_connected()` 추가 |
| `backend/app/services/scope_profile_policy.py` | `should_bypass_cache()` 신규 + `_get_expose_viewers_for_profile` 가 bypass 게이트 사용 |
| `backend/tests/unit/cache/test_namespace.py` | `:` sanitize 2 신규 case |
| `backend/tests/unit/cache/test_pubsub.py` | TestSubscriberAutoReconnect 3 신규 case + helper |
| `backend/tests/unit/test_scope_profile_policy_fg32.py` | strict fail-closed 4 신규 case + autouse fixture single-worker 모드 |
| `backend/tests/unit/test_annotations_service_fg33.py` | mention 4 case 갱신 + `_mock_users` fixture 보강 |
| `docs/개발문서/S3/phase7/산출물/ADR-FG7-3-fail-closed-정책.md` | 신규 ADR |
| `docs/개발문서/S3/phase7/산출물/Phase7_Codex2차_시정보고서.md` | 본 보고서 |

---

## 10. Change Boundary

- **intent**: Codex 2차 검수 4건 발견 + 1건 권고 시정.
- **handoff_level**: extended — P1/보안/정책/외부 I/O (Valkey pub/sub).
- **changed functions**: `_quote_part`, `Subscriber.start/_supervisor/_try_subscribe/_message_loop/is_connected`, `should_bypass_cache`, `_get_expose_viewers_for_profile`.
- **behavior changes**:
  - `scope_policy` strict fail-closed 도입 (default — Valkey 설정됨 + subscriber 미연결 시 캐시 우회). 운영자 env override 로 opt-out 가능.
  - `Subscriber` 자동 재구독 — 이전엔 start 1회 실패 시 영구 종료. 이제 supervisor 가 backoff 재시도.
- **tests added/updated**: 신규 12 (cache + scope_policy), 갱신 5 (annotation + scope_policy autouse fixture).
- **validation**:
  - Phase 7 영역 199 / 199 ✅
  - annotation_service_fg33 41 / 41 ✅
  - 전체 unit 2865 passed (Phase 7 무관 alembic 4건 잔여)
- **risks**:
  - Valkey 장애 시 DB 부하 증가 — ADR §6 위험표 명시. 운영자 모니터링 권고.
  - subscriber reconnect 가 backoff 동안 invalidate 누락 → process-local TTL 30s 자연 만료 backstop.
- **open questions**:
  - chaos test 실제 환경 (운영자 합의 필요)
  - admin health endpoint 에 subscriber 상태 노출 (별 라운드)
  - alembic 모듈 환경 셋업 확인 (별 단순 확인)
- **final review by**: Codex 3차 + @최철균 P1 (FG 7-3 정책 변경 + ADR 승인)

---

*Codex 2차 시정 완료. 별 reviewer 3차 검수 + @최철균 P1 승인 후 공식 종결 가능.*
