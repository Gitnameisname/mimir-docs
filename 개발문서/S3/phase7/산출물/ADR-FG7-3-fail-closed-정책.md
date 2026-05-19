# ADR — FG 7-3 `scope_policy` strict fail-closed 정책

**작성일**: 2026-05-19
**상태**: Pending Approval (Codex 2차 P1 시정 2026-05-19 후 구현 적용 — @최철균 P1 승인 잔여)
**관련 산출물**: `Phase 7 개발계획서.md` §1.2, `FG7-3_검수보고서.md` §2, `Phase7_Codex_2차_검수보고서_2026-05-19.md` [P1]
**작성자**: Claude (Design). 승인: pending (`@최철균` P1 승인 필요).
**승인 정정**: 2026-05-19 이전의 종결 승인 발화는 사용자가 다른 프로젝트 대상 오발송으로 정정했으므로, 본 ADR 또는 Mimir Phase 7 승인 기록으로 인정하지 않는다.

---

## 1. 컨텍스트

`scope_profile_policy.should_expose_viewers` 는 viewer 의 ScopeProfile 의 `expose_viewers` 정책 게이트. process-local TTL 30s 캐시 + S3 Phase 7 FG 7-3 에서 Valkey pub/sub broadcast 도입.

Phase 7 개발계획서 §1.2 (R-I2) 는 `scope_policy` 를 **fail-closed** 로 명시:

> 보안 관련 (scope_profile_policy invalidate) 은 fail-closed (cache 비움 → 다음 호출 시 DB 재조회)

그러나 Phase 7 1차 종결 (2026-05-18) 시점 구현은 다음과 같이 변형되었다:

> process-local TTL 30s 유지 + broadcast best-effort. Valkey 장애 시에도 process-local cache 정상 동작.

Codex 2차 검수 (2026-05-19) [P1] 가 이 불일치를 지적했다.

---

## 2. 제안 결정

**Strict fail-closed 정책을 채택하는 안을 제안**한다. 구현은 Codex 2차 P1 시정으로 적용되었지만, ADR의 최종 승인 상태는 아직 `Pending Approval` 이다. 구체 의미:

| 환경 | 동작 |
|------|------|
| Valkey 미설정 (`VALKEY_DISABLED=1` / `VALKEY_HOST=""`) | process-local cache 정상 동작 (single-worker 모드 — cluster-wide 문제 없음) |
| Valkey 정상 + subscriber 연결됨 | process-local cache 정상 동작 (cluster-wide invalidation 신뢰 가능) |
| **Valkey 설정됨 + subscriber 미연결** | **process-local cache 우회 — 매 호출 DB 재조회** |
| 운영자 opt-out (`VALKEY_FAIL_OPEN_FEATURES=scope_policy`) | process-local cache 정상 동작 (호환성 — Phase 7 1차 종결 동작) |

판정 게이트: `app.services.scope_profile_policy.should_bypass_cache()`.

---

## 3. 트레이드오프

### 3.1 보안 vs 가용성

- **보안 (선택)**: admin 이 `expose_viewers=false` 로 변경한 직후 Valkey/subscriber 장애 시, 다른 워커가 stale `true` 를 반환할 위험 차단. viewers 노출은 privacy gate.
- **가용성 (포기)**: Valkey/subscriber 장애 시 모든 viewers 요청이 DB 조회 → 추가 부하. 단, ScopeProfileRepository.get_by_id 는 가벼운 SELECT (인덱스 lookup) 라 부담 작음.

### 3.2 환경 분리

- 단일 워커 / 폐쇄망 환경 (`VALKEY_DISABLED=1`) 에서는 strict fail-closed 미발동 — 운영자가 명시적으로 disable 했으므로 cluster-wide 문제가 없음. 캐시 안전.
- 운영자가 가용성 우선 시 `VALKEY_FAIL_OPEN_FEATURES=scope_policy` opt-out 제공.

### 3.3 다른 정책 키 확장

본 ADR 은 `expose_viewers` 단일 정책에 적용. 향후 `scope_policy` feature 에 추가 정책 (예: `allow_agent_actions`) 추가 시 동일 strict fail-closed 게이트 적용.

---

## 4. 대안 검토

### A. (기각) Phase 7 1차 동작 유지 (best-effort)

- TTL 30s 자연 만료에 의존 → 최악 30s stale.
- viewers 노출 정책의 보안 영향이 적다면 채택 가능했으나, privacy gate 인 점 + admin 변경의 의도성 (즉시 적용 기대) 고려 시 부적합.

### B. (기각) Strict fail-closed 무조건 (env opt-out 없음)

- 가장 보안 보수적이지만, 다음 시나리오에서 부담:
  - 운영자가 Valkey 가용성을 잠시 포기하고 가용성 우선 모드로 운영하고 싶을 때 강제 DB 호출 부하.
- env override 로 운영자 선택권 보존 (보안 default + 명시적 opt-out).

### C. (제안 채택안) 환경 인지 strict fail-closed

본 ADR 의 제안 채택안. R-I2 의 정신 (보안 우선 default) + 운영 유연성 (env override + 단일 워커 모드 인지) 균형.

---

## 5. 구현

`backend/app/services/scope_profile_policy.py`:

```python
def should_bypass_cache() -> bool:
    if is_fail_open("scope_policy"):
        return False  # 운영자 opt-out
    if is_valkey_disabled():
        return False  # 단일 워커 모드 — cache 안전
    sub = _subscriber
    if sub is None or not sub.is_connected():
        return True  # strict fail-closed
    return False
```

`_get_expose_viewers_for_profile()` 가 위 게이트로 캐시 lookup / store 모두 우회.

회귀: `tests/unit/test_scope_profile_policy_fg32.py::TestClusterWideInvalidation` 의 4 신규 case (`test_strict_fail_closed_*`).

---

## 6. 위험 / 모니터링

| 위험 | 대응 |
|------|------|
| Valkey 장애 시 DB 부하 폭증 | ScopeProfileRepository.get_by_id 는 lightweight SELECT. PostgreSQL 인덱스 활용. 운영자가 부하 모니터링. 별 산출물 |
| subscriber 연결 상태 모니터링 부재 | `is_connected()` 가 admin API 또는 health endpoint 에 노출되어야 — 별 라운드 |
| operator 가 fail-open override 후 Valkey 장애 인지 못함 | env 변경은 audit 로 남기는 정책 권고 — 운영자 책임 |
| 단일 호스트 다중 워커 (uvicorn `--workers 4`) — Valkey 미설정 | 운영자가 다중 워커 + Valkey 미설정 시 명시 경고. 본 ADR 은 `VALKEY_DISABLED=1` 을 "single-worker" 신호로 신뢰 |

---

## 7. 승인 / 변경 관리

본 ADR 의 변경은 **P1 변경** (정책 / 보안). `CLAUDE.md` §1.1 의 extended Handoff Level. @최철균 + 운영자 합의 필요.

현재 상태:
- 구현 적용: 완료 (Codex 2차 P1 시정)
- Codex 검수: 3차에서 승인 표기 불일치 지적, 본 문서에서 정정
- 최종 승인: pending (`@최철균` P1 승인 필요)

---

## 8. 변경 이력

| 일자 | 변경 | 작성자 |
|------|------|-------|
| 2026-05-19 | 초안 — Codex 2차 P1 시정안 작성 | Claude |
| 2026-05-19 | 승인 오발송 정정 반영 — 상태를 `Pending Approval` 로 수정 | Codex |

---

*ADR — S3 Phase 7 FG 7-3 의 fail-closed 정책 명문화. 향후 다른 정책 키 추가 시 본 ADR 의 분기를 재사용.*
