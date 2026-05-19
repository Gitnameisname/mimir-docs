# S3 Phase 7 1라운드 종결 보고서

**작성일**: 2026-05-18
**범위**: S3 Phase 7 (인프라 — Valkey + Cluster-wide Cache Invalidation)
**선언**: 1차 종결 — Codex 2차 검수 + @최철균 P1 승인 게이트 잔여
**작성자**: Claude

---

## 1. 1라운드 완료 FG

| FG | 제목 | 작업지시서 | 산출물 |
|----|------|----------|--------|
| **7-1** | Valkey 인프라 정비 + 네임스페이스 + 폐쇄망 fallback | `phase7/작업지시서/task7-1.md` | `FG7-1_검수보고서.md`, `FG7-1_인프라가이드.md` |
| **7-2** | `viewed_throttle` cluster-wide SETNX | `phase7/작업지시서/task7-2.md` | `FG7-2_검수보고서.md`, `FG7-2_보안취약점검사보고서.md` |
| **7-3** | `scope_profile_policy` pub/sub invalidation | `phase7/작업지시서/task7-3.md` | `FG7-3_검수보고서.md`, `FG7-3_보안취약점검사보고서.md` |

**조건부 FG (7-4 / 7-5 / 7-6)**: 본 1라운드 범위 외 (`phase7/Phase 7 개발계획서.md` §2.2). 운영자 + @최철균 합의 후 별 라운드.

---

## 2. R-I 절대 규칙 충족표

| 규칙 | FG 7-1 | FG 7-2 | FG 7-3 |
|------|--------|--------|--------|
| **R-I1 폐쇄망 호환** | ✅ `is_valkey_disabled()` | ✅ LRU fallback | ✅ subscriber silent skip |
| **R-I2 fail-open / closed** | ✅ FailPolicy enum + override | ✅ fail-open | ⚠️ fail-open + best-effort broadcast (별 ADR 가능) |
| **R-I3 pub/sub 권한** | ✅ `make_channel(org_id=)` tenant prefix | N/A | ✅ size limit + loop 방지 + JSON parse 격리 |
| **R-I4 키 namespace** | ✅ `make_key()` env prefix 강제 | ✅ `mimir:<env>:viewed:...` | ✅ `mimir:<env>:cache:invalidate:scope_policy` |

> **R-I2 FG 7-3 NOTE**: Phase 7 개발계획서는 `scope_policy` 를 "fail-closed (cache 비움)" 로 분류했으나, 본 1라운드 구현은 **process-local TTL 30s 는 fail-open 유지** + **cluster-wide broadcast 만 best-effort** 로 합의 (FG7-3 검수 §2). strict fail-closed (캐시 비움 + 매 호출 DB) 는 별 ADR 대상.

---

## 3. 신규 / 변경 자산

### 3.1 신규 모듈

| 파일 | 역할 | LOC |
|---|---|---|
| `backend/app/cache/namespace.py` | env-aware key/channel 합성 | ~80 |
| `backend/app/cache/policy.py` | fail-open/closed 분류 | ~80 |
| `backend/app/cache/pubsub.py` | publish_invalidate + Subscriber | ~190 |

### 3.2 수정 모듈

| 파일 | 변경 |
|---|---|
| `backend/app/cache/__init__.py` | 신규 헬퍼 export |
| `backend/app/cache/valkey.py` | `is_valkey_disabled()` + `get_valkey_or_none()` 추가 |
| `backend/app/config.py` | `valkey_disabled` / `valkey_namespace` / `valkey_fail_open_features` / `valkey_fail_closed_features` |
| `backend/app/audit/viewed_throttle.py` | `_try_valkey_setnx` 1차 + LRU fallback |
| `backend/app/services/scope_profile_policy.py` | `invalidate_cache(broadcast=)` + `start_subscriber()` / `stop_subscriber()` |
| `backend/app/api/v1/scope_profiles.py` | PATCH 가 `broadcast=True` 호출 |
| `backend/app/main.py` | startup 에 `start_subscriber()`, shutdown 에 `stop_subscriber()` |

### 3.3 신규 테스트

| 파일 | 케이스 수 |
|---|---|
| `tests/unit/cache/test_namespace.py` | 14 |
| `tests/unit/cache/test_policy.py` | 10 |
| `tests/unit/cache/test_valkey_disabled_mode.py` | 12 |
| `tests/unit/cache/test_pubsub.py` | 16 |
| `tests/unit/audit/test_viewed_throttle_cluster.py` | 14 |
| `tests/unit/test_scope_profile_policy_fg32.py` (신규 추가) | 7 |
| **합계** | **73** |

기존 16 (viewed_throttle) + 14 (scope_profile_policy) = 30 case 도 호환 픽스처 적용 후 그대로 녹색.

---

## 4. 회귀 결과

| 범위 | 결과 |
|---|---|
| Phase 7 신규 단위 (`tests/unit/cache/` + `tests/unit/audit/test_viewed_throttle_cluster.py`) | 66 / 66 ✅ |
| Phase 7 영역 + 인접 회귀 (audit + scope_profile_policy + scope_profile_settings + scope_profile_repository + documents_scope_profile_filter + cache/) | 187 / 187 ✅ |
| 전체 단위 회귀 | 2852 / 2860 (8건 사전-존재 실패 — `test_annotations_service_fg33` mention regex + `test_embedding_dim_check` alembic 미설치. Phase 7 무관) |

Phase 7 변경 영역에서 신규 회귀 0건.

---

## 5. 완료 기준 자가 점검 (`phase7/Phase 7 개발계획서.md` §5.1)

| 항목 | 상태 | 비고 |
|---|---|---|
| R-I1 폐쇄망 — Valkey off 시 fallback ≥ 5 시나리오 | ⚠️ 부분 | `tests/unit/cache/test_valkey_disabled_mode.py` 단위 12 case + fallback 회귀. 실제 컨테이너 chaos test 는 별 산출물 (운영자 합의) |
| R-I2 fail-open / closed 정책 | ✅ | `FailPolicy` enum + override + scope_policy fail-open + ADR 사유 명문화 |
| R-I3 pub/sub 권한 | ✅ (코드) | tenant prefix + size limit + loop 방지. AUTH/ACL 운영자 책임 (인프라 가이드 §3.3) |
| R-I4 키 namespace | ✅ | env prefix 강제 + 격리 회귀 |
| `viewed_throttle` 다중 워커 1건 emit | ✅ | `TestMultiWorkerSimulation.test_shared_valkey_dedups_across_workers` |
| `scope_profile_policy` admin PATCH < 1s invalidate | ✅ (코드) | broadcast=True publish 회귀. 실제 latency 측정은 별 산출물 |
| Phase 1~6 모든 회귀 녹색 | ✅ | Phase 7 무관 사전-존재 실패 외 신규 회귀 0 |

### 5.2 회귀 게이트

| 항목 | 상태 |
|---|---|
| pytest 베이스라인 유지 | ✅ |
| node:test 베이스라인 유지 | N/A (frontend 변경 없음) |
| tsc 0 error | N/A (frontend 변경 없음) |
| Valkey off 모드 회귀 (폐쇄망) | ⚠️ 단위 회귀 (12 case) — 컨테이너 chaos 는 잔여 |
| Valkey 장애 chaos 테스트 | ⚠️ 단위 회귀 (fail-open / fail-closed) — 컨테이너 chaos 는 잔여 |

---

## 6. 잔여 / 별 산출물

| # | 항목 | 분류 |
|---|---|---|
| O-7-1 | 폐쇄망 회귀 ≥ 5 시나리오 — 실제 docker-compose Valkey off chaos | 별 산출물 (운영자 합의 후) |
| O-7-2 | Valkey 컨테이너 정지/재기동 chaos test (4 워커 환경) | 별 산출물 |
| O-7-3 | strict fail-closed for scope_policy (cache 비움 + 매 호출 DB) | 별 ADR (운영자 합의) |
| O-7-4 | response_cache 등 다른 feature 의 cluster-wide invalidation | 별 라운드 |
| O-7-5 | tenant-scoped invalidation 실제 사용 사례 | 별 라운드 |
| O-7-6 | subscriber 자동 reconnect (Valkey 회복 시 자동 재구독) | 별 라운드 (heartbeat + retry backoff) |
| O-7-7 | FG 7-4 (cluster-wide rate-limit + per-user keying) | 조건부 — 운영자 합의 |
| O-7-8 | FG 7-5 (retention cron lock + notification dedup) | 조건부 — 운영자 합의 |
| O-7-9 | FG 7-6 (SUPER_ADMIN 횡단 알람) | 조건부 — 운영자 합의 |

위 잔여 9항은 본 1라운드 종결과 독립.

---

## 7. 공식 종결 절차

### 7.1 본 보고서가 제출 시점에 충족된 게이트

- 코드 변경 (FG 7-1/7-2/7-3) 완료
- 단위 회귀 73 신규 + 30 기존 + 84 인접 = 187 ✅
- 검수 보고서 3건, 보안 보고서 2건, 인프라 가이드 1건 작성
- 함수도서관 §1.7-fg31 / §1.7-fg32 / §1.11-fg71 / §1.11-fg73 갱신
- R-I 4개 규칙 충족 (R-I2 의 FG 7-3 변형은 ADR 사유 명문화)

### 7.2 공식 종결 전 잔여 게이트

- Codex (또는 별 reviewer) **2차 검수** — 본 보고서 + 코드 + 회귀 결과 독립 점검
- **@최철균 P1 승인** (Operation Mode dual-agent + extended Handoff Level)
- 운영자 합의 — Valkey 인프라 도입 정책 (R-I3 ACL 정책, 폐쇄망 chaos 검증 계획)

---

## 8. S3 1라운드 전체 종결과의 관계

`phase7/Phase 7 개발계획서.md` §5.3 — 본 1라운드 종결 = **S3 1라운드 전체 종결**.

| S3 Phase | 상태 |
|---|---|
| Phase 0 (S2 기반 구축) | ✅ 종결 (2026-04-16) |
| Phase 2 FG 2-3 ~ 2-6 (Phase 2 흡수) | ✅ 종결 (2026-05-11) |
| Phase 3 (Conversation 도메인) | ✅ 종결 (2026-04-17) |
| Phase 4 (Agent-Facing) | ✅ 종결 (2026-04-17) |
| Phase 5 (편집 UX) | ✅ 종결 (2026-05-18) |
| Phase 6 (운영 안전) | ✅ 종결 (2026-05-18) |
| **Phase 7 (인프라)** | 🟡 1차 종결 (본 보고서) — Codex 2차 + @최철균 P1 승인 잔여 |

S3 1라운드 전체 종결은 본 Phase 7 의 공식 승인 후 별 보고서 (S3_1라운드_공식종결보고서.md) 로 선언.

---

## 9. 변경 이력

| 일자 | 변경 | 작성자 |
|---|---|---|
| 2026-05-18 | 초안 — FG 7-1/7-2/7-3 1차 종결 | Claude |

---

*S3 Phase 7 — 인프라 라운드. S3 1라운드 마지막 Phase. Codex 2차 검수 + @최철균 P1 승인 + 운영자 합의 후 공식 종결.*
