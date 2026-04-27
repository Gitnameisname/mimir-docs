# S3 Phase 7 — 인프라 (Valkey + Cluster-wide Cache Invalidation)

**작성일**: 2026-04-27
**Phase 상태**: 대기 (Phase 6 종결 후 진입 권장 — rate-limit / retention 이 cluster-wide 화 대상)
**선행 조건**: Valkey 클러스터 가용 (운영 인프라), Phase 3~6 종결
**후행 조건**: S3 1라운드 공식 종결 또는 S3 2라운드 / S4 진입
**Handoff Level**: `extended` — 인프라 도입 (Valkey 클러스터) + cluster-wide 일관성 (다중 워커 캐시) + 보안 (pub/sub 권한)
**Approver**: `@최철균` + 운영자 (P1 — 인프라 변경)

---

## 0. Phase 4 / 5 / 6 와의 관계

본 Phase 는 **인프라** 라운드로, application layer 의 잔존을 cluster-wide 일관성으로 강화한다:

- **Phase 3 의 `viewed_throttle`** (per-process LRU) → cluster-wide dedup
- **Phase 3 의 `scope_profile_policy` 캐시** (process-local TTL 30s) → admin PATCH 시 cluster-wide invalidate
- **Phase 6 의 rate-limit** (워커별 in-process) → cluster-wide rate-limit (선택 — rate-limit 의 워커 분산 영향이 작으면 보류)
- **Phase 4 의 `tools/list` manifest** — manifest 는 코드 상수라 cache 무관. 영향 없음

**Phase 6 종결 후 진입 권장** 이유:
- Phase 6 의 retention cron 이 multi-worker 환경에서 정상 동작하는지 먼저 검증 (cron lock 은 본 Phase 의 Valkey 도입 후 가능)
- Phase 6 의 rate-limit 통일이 끝난 상태에서 cluster-wide 화가 자연스러움

병행 진입 가능하나, Phase 6 1라운드 종결 후가 안전.

---

## 1. Phase 개요

### 1.1 목적

S3 1라운드의 마지막 잔존 — 다중 워커 환경의 일관성 문제 — 를 한 인프라 도입으로 일괄 해결한다. Valkey (Redis OSS fork) 클러스터를 공식 의존성으로 추가하고, 캐시 invalidation / dedup / cron lock 등 cluster-wide 동기를 도입한다.

### 1.2 절대 규칙

- **R-I1. 폐쇄망 호환 (S2 ⑦)** — Valkey 가 외부 인터넷 의존이 아니어야 함. 사내 클러스터 또는 docker-compose 로 self-host. 폐쇄망에서 단일 노드 fallback 가능 (degrade)
- **R-I2. Fail-open vs Fail-closed 명시** — 각 cluster-wide 기능마다 Valkey 장애 시 동작 정책 명시. 보안 관련 (scope_profile_policy invalidate) 은 fail-closed (cache 비움 → 다음 호출 시 DB 재조회), 성능 관련 (viewed_throttle) 은 fail-open (워커별 LRU fallback)
- **R-I3. Pub/Sub 권한** — Valkey 의 pub/sub 채널은 인증 (AUTH) 강제. 채널명에 tenant prefix (`mimir:tenant:<org_id>:cache:invalidate`) 로 격리
- **R-I4. 키 namespace 격리** — Valkey 키 명명: `mimir:<env>:<feature>:<args>`. 다른 환경 / feature 충돌 방지 (env 는 dev/staging/prod)

### 1.3 선행 조건

- Phase 3~6 종결
- Valkey 클러스터 가용 (운영 인프라 측 결정 — sidecar / managed / self-hosted 중 선택)
- 폐쇄망 환경에서 docker-compose 로 단일 Valkey 인스턴스 가용 검증

### 1.4 기대 결과

- Valkey 클러스터 공식 의존성 + connection pool (`backend/app/infra/valkey.py` 또는 기존 cache 모듈 확장)
- `viewed_throttle` cluster-wide dedup (워커 간 일관)
- `scope_profile_policy` 캐시 invalidation pub/sub
- (선택) rate-limit cluster-wide
- (선택) retention cron lock (다중 워커 중복 실행 방지)
- 폐쇄망 fallback 회귀 (Valkey off 시 워커별 동작)

---

## 2. Feature Group (FG) 요약

본 Phase 는 **5개 FG (7-1 ~ 7-5)** 로 구성한다. FG 7-4, 7-5 는 조건부.

### 2.1 1라운드 (즉시 착수 대상)

| FG | 제목 | 핵심 | 산출물 |
|----|------|------|--------|
| **7-1** | Valkey 인프라 도입 + connection pool + 폐쇄망 fallback | docker-compose / 사내 클러스터 + Python client (`valkey-py` 또는 `redis-py`). 폐쇄망 fallback 검증 | task7-1 + 인프라 가이드 + 운영자 합의 기록 |
| **7-2** | `viewed_throttle` cluster-wide dedup | 현 in-process LRU → Valkey SET / EXPIRE. fail-open 정책 (Valkey 장애 시 워커별 LRU fallback) | task7-2 + 검수·보안 보고서 |
| **7-3** | `scope_profile_policy` 캐시 cluster-wide invalidation | admin PATCH `/scope-profiles/{id}` 가 Valkey pub/sub broadcast. 모든 워커가 subscribe + 즉시 invalidate. fail-closed 정책 (Valkey 장애 시 캐시 비움) | task7-3 + 검수·보안 보고서 |

### 2.2 조건부 (게이트 통과 후 사용자 합의)

| FG | 제목 | 진행 조건 |
|----|------|---------|
| **7-4** | Cluster-wide rate-limit | 다중 워커 환경에서 rate-limit 우회가 운영 영향 (실측). 영향 작으면 보류 |
| **7-5** | Retention cron lock + notification dedup | (a) 다중 워커가 같은 cron 동시 실행 방지 (Valkey SETNX lock). (b) annotation 멘션 폭주 시 cluster-wide dedup |

> ⚠️ FG 7-4 / 7-5 는 본 Phase 1라운드 게이트의 일부가 **아니다**.

---

## 3. 데이터 / 스키마 모델 (개요)

본 Phase 는 영구 DB 스키마 변경 **없음**. Valkey 키 / 채널만 추가.

### 3.1 Valkey 키 명명

```
mimir:<env>:viewed:<actor_id>:<document_id>           # value=ts (FG 7-2)
mimir:<env>:scope_policy:<scope_profile_id>           # value=expose_viewers bool (FG 7-3)
mimir:<env>:rate:<actor_id>:<endpoint>                # value=count (FG 7-4 조건부)
mimir:<env>:cron:retention:lock                       # value=worker_id, EXPIRE 1h (FG 7-5 조건부)
```

### 3.2 Valkey 채널 명명

```
mimir:<env>:tenant:<org_id>:cache:invalidate:<feature>  # FG 7-3 등 invalidation 브로드캐스트
```

### 3.3 환경변수

```
VALKEY_URL=valkey://valkey-0:6379    # 또는 redis:// 호환
VALKEY_AUTH=...                       # AUTH 토큰
VALKEY_NAMESPACE=mimir:prod
VALKEY_FAIL_OPEN_FEATURES=viewed_throttle,rate_limit
VALKEY_FAIL_CLOSED_FEATURES=scope_policy
```

---

## 4. 산출물 규약

| 산출물 | FG 7-1 | FG 7-2 | FG 7-3 | FG 7-4 | FG 7-5 |
|--------|--------|--------|--------|--------|--------|
| 작업지시서 (`task7-N.md`) | ✅ | ✅ | ✅ | (조건부) | (조건부) |
| 인프라 가이드 (운영자 대상) | ✅ | — | — | — | — |
| 검수보고서 (R-I1~I4 준수 확인) | ✅ | ✅ | ✅ | (조건부) | (조건부) |
| 보안취약점검사보고서 (pub/sub 권한) | — | ✅ | ✅ | (조건부) | (조건부) |
| 운영자 합의 기록 | ✅ | — | — | — | — |
| 폐쇄망 회귀 결과 | ✅ | ✅ | ✅ | — | — |

---

## 5. 게이트 / 완료 기준

### 5.1 1라운드 완료 기준 (FG 7-1 ~ 7-3)

- [ ] **R-I1 (폐쇄망)** — Valkey off (`VALKEY_URL=`) 환경에서 모든 기능이 fallback 모드로 정상 동작. 회귀 ≥ 5 시나리오
- [ ] **R-I2 (fail-open / fail-closed)** — Valkey 장애 시 각 기능별 정책에 맞게 동작 (chaos 테스트 — 의도적 connection drop)
- [ ] **R-I3 (pub/sub 권한)** — AUTH 없는 클라이언트가 pub/sub 채널 구독 시도 시 거부. 다른 tenant prefix 의 채널은 본 tenant 워커가 구독 안 함
- [ ] **R-I4 (키 namespace)** — `mimir:dev:` / `mimir:prod:` 키가 충돌 없음 (회귀 테스트)
- [ ] `viewed_throttle` 가 다중 워커 환경에서 정확히 1건만 emit (회귀 — 4 워커 동시 호출 시)
- [ ] `scope_profile_policy` 가 admin PATCH 후 < 1초 안에 모든 워커에서 invalidate (회귀)
- [ ] Phase 1~6 모든 회귀 녹색

### 5.2 회귀 게이트

- [ ] pytest 베이스라인 유지
- [ ] node:test 베이스라인 유지
- [ ] tsc 0 error
- [ ] Valkey off 모드 회귀 (폐쇄망)
- [ ] Valkey 장애 chaos 테스트 (connection drop)

### 5.3 1라운드 종결 정의

위 5.1 + 5.2 모두 통과 시 **S3 Phase 7 1라운드 공식 종결** = **S3 1라운드 전체 종결**.

S3 2라운드 또는 S4 진입 결정은 별 회고에서.

---

## 6. 범위 밖 / 이월

### 6.1 본 Phase 1라운드 명시적 제외

- **Valkey 클러스터 sharding** — 본 Phase 는 단일 노드 또는 sentinel 모드. sharding 은 별 인프라 라운드
- **Cross-region replication** — 별 작업
- **Valkey 외 다른 캐시 backend** (Memcached / Hazelcast) — 본 Phase 는 Valkey 만
- **WebSocket / SSE 알림 push** — Phase 3 의 polling 그대로. push 는 S4
- **외부 SaaS 커넥터** — 여전히 S3 2라운드 / S4 이월 (Phase 4 와 동일)

### 6.2 보류 결정 (FG 진행 중 결정)

- Valkey 클러스터 sentinel vs cluster mode → FG 7-1 운영자 합의
- pub/sub 채널의 메시지 형식 (JSON vs MessagePack) → FG 7-1 결정
- cluster-wide rate-limit 도입 여부 → FG 7-4 (실측 후)
- retention cron lock 도입 여부 → FG 7-5 (다중 워커 영향 측정 후)

---

## 7. 리스크

| # | 리스크 | 대응 |
|---|-------|-----|
| R-01 | Valkey 클러스터 장애 시 application 전체 5xx | fail-open / fail-closed 정책 명문화 + chaos 테스트 의무 |
| R-02 | 폐쇄망 환경에서 Valkey 운영 부담 | docker-compose 단일 노드 가이드 + fallback 모드 (워커별 LRU) 보존 |
| R-03 | pub/sub 메시지 유실로 invalidate 누락 | TTL 30s 기본값을 그대로 둠 (메시지 유실해도 30초 안에 자연 만료). pub/sub 은 즉시성 보강 layer |
| R-04 | 다른 tenant 의 캐시 키 노출 | namespace prefix `mimir:<env>:tenant:<org_id>:` 강제 + 회귀 테스트 |
| R-05 | Valkey 클라이언트 connection pool 누수 | psycopg2 pool 패턴 그대로 적용. 명시적 close + finally |
| R-06 | Phase 4 / 5 / 6 의 다른 잔존 (예: archive cold storage) 와 본 Phase 작업 충돌 | 인프라 변경 (Valkey) 이 다른 작업과 독립. 충돌 없음 검증 |
| R-07 | 운영자가 Valkey 인프라 도입 거부 (보안 / 비용) | FG 7-1 진입 전 운영자 합의 게이트. 거부 시 본 Phase 전체 보류 |

---

## 8. 협업 / 라우팅

`AGENT_MODE.md` §3.3 (extended):
- Design: Claude (인프라 결정은 운영자 합의 후)
- Implementation: Codex (또는 Claude — 인프라 코드는 둘 다 가능)
- Review: Codex
- Human Approval: @최철균 + 운영자 (인프라 변경은 운영자 게이트 의무)

---

## 9. 참조

- `docs/개발문서/S3/phase3/산출물/FG3-1_검수보고서.md` §3 (viewed_throttle 다중 워커 한계)
- `docs/개발문서/S3/phase3/산출물/FG3-2_검수보고서.md` §3 (scope_profile_policy 다중 워커 한계)
- `docs/개발문서/S3/phase3/산출물/FG3-3_검수보고서.md` §3 (notification rate-limit cluster-wide 잔존)
- `docs/개발문서/S3/phase6/Phase 6 개발계획서.md` (rate-limit / retention cron — Phase 6 의 단일 워커 구조 위에 cluster 화)
- `docs/규칙/CONSTITUTION.md` 제16조 (Trust Level), 제24조 (Long-term Memory) — Valkey 가 long-term memory 가 아님 명시 의무

---

## 10. 변경 이력

| 일자 | 변경 | 작성자 |
|------|------|-------|
| 2026-04-27 | 초안 — Phase 4 (MCP) 신설 후 재정렬. 기존 "Phase 6 (Valkey)" 안을 Phase 7 으로 이동. S3 1라운드 종결 Phase | Claude |

---

*작성: 2026-04-27 | 인프라 — S3 Phase 7 킥오프 (S3 1라운드 마지막 Phase)*
