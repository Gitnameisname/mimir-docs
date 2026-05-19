# Task 7-2 — `viewed_throttle` Cluster-wide Dedup

**작성일**: 2026-05-18
**FG**: 7-2
**Handoff Level**: `extended` — 인프라 정책 (다중 워커 dedup 정합)
**Approver**: `@최철균`
**선행**: Task 7-1 (Valkey 인프라 정비)
**관련 모듈**: `backend/app/audit/viewed_throttle.py`

---

## 1. 의도 (Intent)

S3 Phase 3 FG 3-1 의 `should_emit_view` 는 process-local LRU 로 dedup 한다. 다중 워커 환경에서는 같은 (actor, document) view 가 워커 수 N 만큼 emit 될 수 있음 — Phase 3 FG3-1_검수보고서 §3 가 본 라운드 잔존으로 명시.

본 Task 는 Valkey 의 `SET NX EX` 원자성을 사용해 cluster-wide 단일 emit 을 보장한다. fail-open 정책 (Valkey 장애 시 워커별 LRU fallback) 으로 가용성 우선.

---

## 2. 범위 (Scope)

### 2.1 변경 — IN

| 파일 | 변경 |
|------|------|
| `backend/app/audit/viewed_throttle.py` (수정) | `should_emit_view()` 가 Valkey `SET NX EX` 시도 → 실패/disabled 시 in-process LRU fallback |
| `backend/tests/unit/audit/test_viewed_throttle.py` (수정) | 기존 16 case 유지 + 신규 6 case (Valkey hit/miss/error/disabled, namespace 격리) |
| `backend/tests/unit/audit/test_viewed_throttle_cluster.py` (신규) | cluster-wide dedup 시나리오 (Valkey mock + multi-call) |

### 2.2 변경 — OUT

- emit 자체는 호출지(`app/api/v1/documents.py`)에 그대로 — 본 Task 는 dedup 게이트만 변경.
- `audit_events` 테이블 / event_types 변경 없음.

---

## 3. 설계 (Design)

### 3.1 호출 흐름

```
should_emit_view(actor_id, document_id):
    1. 입력 검증 (None / "" → False)
    2. window_sec = _window_seconds()
    3. if window_sec == 0: return True  (dedup off)
    4. Valkey 시도:
       client = get_valkey_or_none()
       if client and not disabled:
           key = make_key("viewed", actor_id, document_id)
           try:
               # SETNX(key, ts, EX=window_sec)
               # NX: 이미 있으면 set 안 함. 반환 None.
               # NX 성공 (반환 True) → 신규 → emit
               # NX 실패 (반환 None) → 이미 있음 → skip
               set_result = client.set(key, str(now_ts), nx=True, ex=window_sec)
               if set_result is True:
                   return True
               if set_result is None:
                   return False
           except Exception:
               # fail-open: LRU fallback
               pass
    5. in-process LRU (기존 로직 — fallback)
```

### 3.2 키 / TTL

- 키: `mimir:<env>:viewed:<actor_id>:<document_id>`
- TTL: `AUDIT_VIEWED_DEDUP_WINDOW_SEC` (기본 300)
- 값: `str(now_ts)` (디버깅 용, 실제 사용 안 함)

### 3.3 fail-open 정책 (R-I2)

- Valkey 장애 (ConnectionError, TimeoutError, RedisError) → in-process LRU fallback
- `is_valkey_disabled()` True → in-process LRU 만 사용 (silent — 정상 폐쇄망 동작)
- 로그: ERROR 가 아닌 DEBUG 로 (운영자가 의도한 fallback 가능)

### 3.4 동시성

- Valkey `SET NX EX` 는 원자적. 다중 워커에서 정확히 1건만 True 반환.
- in-process LRU 는 기존 `threading.Lock` 유지 (fallback 경로).

---

## 4. 절대 규칙 (R-I) 매핑

| 규칙 | 본 Task 대응 |
|------|------|
| **R-I1 폐쇄망** | `is_valkey_disabled()` 시 LRU fallback. 회귀 보장 |
| **R-I2 fail-open** | Valkey 장애 시 LRU fallback. policy `viewed_throttle=FAIL_OPEN` 등록 |
| **R-I3 pub/sub 권한** | 본 Task 는 pub/sub 미사용. N/A |
| **R-I4 키 namespace** | `make_key("viewed", actor_id, doc_id)` 사용 |

---

## 5. 회귀 / 검증

### 5.1 기존 단위 테스트 (16 case)

기존 테스트는 in-process LRU 동작을 검증하므로, **Valkey disabled 모드** 에서 그대로 통과해야 한다.

테스트 픽스처에 `VALKEY_DISABLED=1` 또는 `monkeypatch` 로 `get_valkey_or_none` → None 강제.

### 5.2 신규 단위 테스트 (≥ 6 case)

- `test_valkey_setnx_dedup_first_call_emits`
- `test_valkey_setnx_dedup_within_window_skips`
- `test_valkey_failure_falls_back_to_lru`
- `test_valkey_disabled_uses_lru`
- `test_namespace_includes_environment`
- `test_concurrent_multi_worker_simulation` (Valkey mock 으로 워커별 cache 분리한 후 동일 SETNX 시 1건만 emit)

### 5.3 회귀 게이트

- pytest backend/tests/ 베이스라인 유지
- 본 Task 이전 16 case 전부 녹색

---

## 6. 산출물

| 산출물 | 위치 |
|--------|------|
| 코드 변경 | `backend/app/audit/viewed_throttle.py` |
| 단위 테스트 | `backend/tests/unit/audit/` |
| 검수보고서 | `docs/개발문서/S3/phase7/산출물/FG7-2_검수보고서.md` |
| 보안 보고서 | `docs/개발문서/S3/phase7/산출물/FG7-2_보안취약점검사보고서.md` |
| 함수도서관 | `docs/함수도서관/backend.md` §1.7-fg31 갱신 (cluster-wide 명시) |

---

## 7. 위험

- **R-1** 다른 tenant 의 viewed key 가 collide → 본 Task 는 key 에 tenant prefix 없음. actor_id 가 글로벌 유일 (UUID) 이므로 OK. 회귀에서 namespace 격리만 검증.
- **R-2** TTL 만료 직후 race → SETNX 가 원자적이므로 정확히 1건만 emit. 윈도우 정의에 맞음.

---

## 8. 완료 기준

- [ ] `should_emit_view` cluster-wide 동작 (Valkey 정상)
- [ ] `should_emit_view` fail-open fallback (Valkey 장애)
- [ ] 단위 테스트 ≥ 22 건 (16 기존 + 6 신규) 녹색
- [ ] 함수도서관 갱신
- [ ] FG7-2 검수·보안 보고서 작성
