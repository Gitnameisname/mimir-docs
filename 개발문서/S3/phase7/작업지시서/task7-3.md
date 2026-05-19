# Task 7-3 — `scope_profile_policy` Cluster-wide Invalidation (pub/sub)

**작성일**: 2026-05-18
**FG**: 7-3
**Handoff Level**: `extended` — 보안 정책 (정책 변경 cluster-wide 즉시성)
**Approver**: `@최철균`
**선행**: Task 7-1 (Valkey 인프라 정비)
**관련 모듈**: `backend/app/services/scope_profile_policy.py`, `backend/app/api/v1/scope_profiles.py`

---

## 1. 의도 (Intent)

S3 Phase 3 FG 3-2 의 `should_expose_viewers` 는 process-local TTL 30s 캐시. 다중 워커 환경에서 admin 이 scope_profile `expose_viewers` 정책을 변경해도 워커별로 최대 30s 까지 stale value 가 반환됨 — FG3-2 검수 §3 가 본 라운드 잔존.

본 Task 는:

1. **process-local 캐시 유지** (성능)
2. admin PATCH `/scope-profiles/{id}` 가 **Valkey pub/sub broadcast** 발행
3. 모든 워커가 subscriber thread 로 broadcast 수신 → 해당 profile 의 process-local cache invalidate
4. **fail-closed 정책** (Valkey 장애 시 캐시 비움 → 다음 호출 시 DB 재조회)

---

## 2. 범위 (Scope)

### 2.1 변경 — IN

| 파일 | 변경 |
|------|------|
| `backend/app/cache/pubsub.py` (신규) | `publish_invalidate(feature, key)` + `Subscriber(feature, on_message)` 클래스 |
| `backend/app/services/scope_profile_policy.py` (수정) | `invalidate_cache()` 가 Valkey 브로드캐스트 + 자체 process cache 비움. 신규 `_handle_remote_invalidate(payload)` 콜백 |
| `backend/app/api/v1/scope_profiles.py` (수정) | PATCH `/scope-profiles/{id}` 가 `invalidate_cache(profile_id, broadcast=True)` 호출 (이미 invalidate_cache 호출 중 — broadcast 파라미터 추가) |
| `backend/app/main.py` (수정) | startup 이벤트에 subscriber 등록 (`scope_profile_policy.start_subscriber()`) |
| `backend/tests/unit/cache/test_pubsub.py` (신규) | publish / subscribe 단위 테스트 |
| `backend/tests/unit/test_scope_profile_policy_fg32.py` (수정) | 신규 cluster-wide 시나리오 4 case 추가 |

### 2.2 변경 — OUT

- `scope_profile_repository` / DB 스키마 변경 없음
- 다른 cache feature (response_cache 등) 의 invalidation 은 본 Task 범위 외

---

## 3. 설계 (Design)

### 3.1 pub/sub 채널

- 채널: `mimir:<env>:cache:invalidate:scope_policy` (cluster-wide — 본 정책은 tenant 격리 무관)
- 메시지 형식: JSON `{"key": "<profile_id>" | "*", "ts": <unix_ts>, "worker_id": "<pid>"}`
- `key == "*"` → 전체 cache 비움
- `worker_id == self.pid` → 자기 자신이 보낸 message 는 skip (이미 local invalidate 됨)

### 3.2 publish 흐름

```
invalidate_cache(profile_id, *, broadcast=True):
    1. local cache 비움 (기존 로직)
    2. if not broadcast: return
    3. if is_valkey_disabled(): return
    4. publish_invalidate("scope_policy", profile_id or "*")
```

publish 자체는 best-effort — Valkey 일시 장애 시 silent skip (다음 admin PATCH 가 재발행하거나, 캐시 TTL 30s 가 자연 만료).

### 3.3 subscriber 흐름

```
start_subscriber():
    1. if is_valkey_disabled(): warn + skip
    2. background thread 시작
    3. SUBSCRIBE mimir:<env>:cache:invalidate:scope_policy
    4. on message: parse JSON → invalidate_cache(key, broadcast=False) (loop 방지)
```

worker thread 는 `daemon=True` — 앱 종료 시 함께 종료.

### 3.4 fail-closed 정책 (R-I2)

`scope_profile_policy._get_expose_viewers_for_profile` 가 Valkey 장애 / disabled 일 때:

- **현재(Phase 3)**: process-local TTL 30s
- **신규**: process-local TTL 30s **유지** + 다음 invalidate 가 발행 못 하면 결국 TTL 자연 만료

엄격 fail-closed 는 정책 위반의 영향이 큰 경우만 (예: admin 이 expose_viewers off 변경 → 다음 GET /documents/{id}/contributors 가 viewers 노출하면 안 됨).

→ **결정**: TTL 30s 그대로 유지. Valkey 장애 시에도 process-local cache 는 정상 동작. broadcast 만 best-effort. fail-closed 강도는 다음 라운드에서 결정.

> **NOTE**: 본 Task 의 fail-closed 정책은 "cache invalidate 실패가 보안 문제가 되는 경우만 캐시 비움" 이 아니라, "**process-local cache 자체는 fail-open 으로 유지**하되, **cluster-wide 즉시성은 best-effort**" 로 합의한다. 별 라운드에서 strict fail-closed 가 필요하면 ADR 작성.

### 3.5 메시지 검증 (R-I3 pub/sub 권한)

- subscriber 는 자기 채널만 SUBSCRIBE — 다른 tenant prefix 채널 접근 안 함
- 받은 메시지의 `worker_id == self.pid` 면 skip (loop 방지)
- 메시지 JSON 파싱 실패 시 warn + skip (DoS-resistant)
- 메시지 크기 상한 1KB (parse 전 check)

---

## 4. 절대 규칙 (R-I) 매핑

| 규칙 | 본 Task 대응 |
|------|------|
| **R-I1 폐쇄망** | `is_valkey_disabled()` 시 process-local cache 단독 동작. 회귀 |
| **R-I2 fail-open vs closed** | scope_policy 는 process-local fail-open + broadcast best-effort. 별 ADR 가능 |
| **R-I3 pub/sub 권한** | 자기 채널만 구독, message 크기 제한, loop 방지, JSON parse 실패 격리 |
| **R-I4 키 namespace** | `make_channel("scope_policy")` 사용 — `mimir:<env>:cache:invalidate:scope_policy` |

---

## 5. 회귀 / 검증

### 5.1 기존 단위 테스트 (14 case)

- monkeypatch 로 `is_valkey_disabled()` → True 강제
- 모든 기존 case 그대로 녹색

### 5.2 신규 단위 테스트

#### `test_pubsub.py` (신규)
- `publish_invalidate("scope_policy", "sp-1")` 호출 시 Valkey `publish()` 호출 확인
- channel 명이 `mimir:<env>:cache:invalidate:scope_policy` 형식
- message JSON 에 key/ts/worker_id 포함
- `is_valkey_disabled()` True 시 publish skip
- `Subscriber.dispatch(message)` 가 `on_message` 콜백 호출
- worker_id 자기자신 → skip (loop 방지)
- message size > 1024 → skip
- 잘못된 JSON → skip + warn

#### `test_scope_profile_policy_fg32.py` 추가 case
- `invalidate_cache(profile_id, broadcast=True)` 가 publish_invalidate 호출 확인
- broadcast=False 일 때 publish 호출 안 함 (loop 방지)
- 외부 message 수신 시 local cache 비움

### 5.3 회귀 게이트

- pytest backend/tests/ 베이스라인 유지
- 기존 FG 3-2 14 case 녹색

---

## 6. 산출물

| 산출물 | 위치 |
|--------|------|
| 코드 변경 | `backend/app/cache/pubsub.py`, `backend/app/services/scope_profile_policy.py`, `backend/app/api/v1/scope_profiles.py`, `backend/app/main.py` |
| 단위 테스트 | `backend/tests/unit/cache/test_pubsub.py`, `backend/tests/unit/test_scope_profile_policy_fg32.py` |
| 검수보고서 | `docs/개발문서/S3/phase7/산출물/FG7-3_검수보고서.md` |
| 보안 보고서 | `docs/개발문서/S3/phase7/산출물/FG7-3_보안취약점검사보고서.md` |
| 함수도서관 | `docs/함수도서관/backend.md` §1.7-fg32 갱신 + §1.11-fg73 신설 (pubsub) |

---

## 7. 위험

- **R-1 subscriber thread 누수** — daemon=True + 명시적 stop 메서드. 단위 테스트에서 thread 누수 검증.
- **R-2 pub/sub loop** — `worker_id == self.pid` skip + `broadcast=False` 파라미터. 회귀에서 loop 미발생 검증.
- **R-3 메시지 폭주 (DoS)** — message size 1KB 상한 + JSON parse 실패 silent skip. unbounded queue 방지.
- **R-4 redis-py PubSub thread safety** — `pubsub_thread.run_in_thread()` 패턴 사용. redis-py 5.2 공식 지원.

---

## 8. 완료 기준

- [ ] `app.cache.pubsub` 모듈 + 단위 테스트 ≥ 8 case
- [ ] `scope_profile_policy.invalidate_cache(broadcast=True)` 동작
- [ ] startup subscriber 등록 + daemon thread
- [ ] FG 3-2 기존 14 case + 신규 ≥ 4 case 녹색
- [ ] 함수도서관 §1.7-fg32 갱신 + §1.11-fg73 신설
- [ ] FG7-3 검수·보안 보고서 작성

---

*Owner: Claude. Review: Codex 또는 사람. Final approval: @최철균.*
