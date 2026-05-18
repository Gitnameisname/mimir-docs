# task6-1 — Rate Limit 일괄 적용

**Phase**: S3 Phase 6 / FG 6-1
**작성일**: 2026-05-18
**Handoff Level**: `extended` (외부 I/O 흐름 — 보호 layer 추가)
**Approver**: @최철균

---

## 1. 의도

annotations / contributors / notifications 라우터는 Phase 3 진행 중 rate-limit
보호가 누락된 채 머지되었다. citations / mcp / search / rag / users_search 와
같은 `@limiter.limit(...)` dependency 를 일괄 적용해 외부 abuse 표면을 줄인다.

## 2. 산출물

| 파일 | 변경 |
|------|------|
| `backend/app/api/v1/annotations.py` | 7 엔드포인트 (`create/list/get/patch/resolve/reopen/delete`) 에 `@limiter.limit` 추가 |
| `backend/app/api/v1/contributors.py` | `GET /documents/{id}/contributors` 에 `@limiter.limit("60/minute")` 추가 |
| `backend/app/api/v1/notifications.py` | `list / unread-count / mark-read` 에 polling 120/min · write 60/min 추가 |
| `backend/tests/unit/test_rate_limit_fg61.py` | 회귀 9건 (decorator 적용 확인 + limit 상수 일치) |

## 3. 결정한 limit 값 (Phase 계획 §2 표 일치)

| 엔드포인트 | limit | 근거 |
|---|---|---|
| `POST /documents/{id}/annotations` | 30/min | 사람 쓰기 빈도 보호 |
| `PATCH /annotations/{id}` | 30/min | 동일 |
| `POST /annotations/{id}/resolve` | 30/min | 동일 |
| `POST /annotations/{id}/reopen` | 30/min | 동일 |
| `DELETE /annotations/{id}` | 30/min | 동일 |
| `GET /documents/{id}/annotations` | 60/min | 문서 열람 시 1회 ± thread 갱신 |
| `GET /annotations/{id}` | 60/min | 단건 조회 (개별) |
| `GET /documents/{id}/contributors` | 60/min | cache-control private 15s 도 같이 적용 |
| `GET /notifications` | 120/min | polling (≈ 1회/0.5초 상한) |
| `GET /notifications/unread-count` | 120/min | polling |
| `POST /notifications/read` | 60/min | 쓰기 |

## 4. R-O1 (Rate Limit 단일) 준수 확인

- 모든 새 적용은 `app.api.rate_limit.limiter` 의 동일 dependency 사용.
- 라우터 안 별도 `slowapi`/`asyncio.Semaphore` 등 별 구현 0.
- citations / mcp / search / rag / users_search 의 기존 패턴 그대로 답습 (`@limiter.limit(_LIMIT_CONST)`).

## 5. 회귀 (`tests/unit/test_rate_limit_fg61.py`)

| ID | 시나리오 | 결과 |
|---|---|---|
| L1 | annotations 7 핸들러가 `__wrapped__` 보유 | ✅ |
| L2 | contributors 핸들러가 wrapped | ✅ |
| L3 | notifications 3 핸들러가 wrapped | ✅ |
| L4 | `_ANNOTATION_WRITE_LIMIT == "30/minute"` | ✅ |
| L5 | `_ANNOTATION_READ_LIMIT == "60/minute"` | ✅ |
| L6 | `_NOTIFICATIONS_POLL_LIMIT == "120/minute"` | ✅ |
| L7 | `_CONTRIBUTORS_LIMIT == "60/minute"` | ✅ |

`.venv/bin/python -m pytest tests/unit/test_rate_limit_fg61.py` → 9 passed.

## 6. 범위 밖

- Cluster-wide rate-limit (다중 워커 동기) — Phase 7 (Valkey 정합).
- per-user (vs per-IP) keying — 현재 `slowapi` 의 `_get_client_ip` 사용. 향후 별 ADR.
