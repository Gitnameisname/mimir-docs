# Task 0-8 영향성 검토 보고서

> 작성일: 2026-04-17
> 검토 대상: `/api/v1/system/capabilities` 3-tier 분리 코드 수정
> 전제: Phase 2 개발 완료 후 수행

---

## 1. 변경 개요

현재 `GET /api/v1/system/capabilities`는 인증 없이 `pgvector_enabled`, `supported_providers` 등 내부 구성 정보를 모두 노출하는 단일 public 엔드포인트이다.

이를 3-tier로 분리한다:
- **Tier 1** `GET /api/v1/system/health` — 인증 불필요, `status` + `version`만 반환
- **Tier 2** `GET /api/v1/system/capabilities` — 인증 필요, `rag_available` + `chunking_enabled`만 반환
- **Tier 3** `GET /api/v1/admin/system/capabilities` — Admin 전용, 전체 정보 반환

---

## 2. 영향 받는 파일 목록

### 2-1. 직접 수정 대상 (필수)

| 파일 | 변경 내용 | 영향도 |
|------|-----------|--------|
| `backend/app/api/v1/system.py` | 3-tier 라우터 분리, 인증 의존성 추가, 캐시 분리 | **High** |
| `backend/app/api/v1/router.py` | admin_router 등록 추가 (33행 근처) | **Medium** |
| `backend/tests/unit/test_system_capabilities.py` | 전면 재작성 (3-tier 인증/정보격리 검증) | **High** |

### 2-2. 간접 영향 (확인 완료, 변경 불필요)

| 파일 | 현황 | 변경 필요 |
|------|------|-----------|
| `backend/tests/integration/test_system_endpoints.py` | `/health`만 호출 — Tier 1로 이동해도 경로 동일 | **없음** |
| `backend/tests/smoke/fresh_boot.py` | capabilities 미호출, `/health`만 사용 | **없음** |
| `scripts/loadtest/locustfile.py` | `/health`만 호출 | **없음** |
| `backend/tests/unit/test_auth.py` | capabilities 미참조 | **없음** |
| `backend/tests/unit/test_security_headers.py` | capabilities 미참조 | **없음** |
| `backend/tests/unit/test_metrics.py` | capabilities 미참조 | **없음** |
| `backend/tests/unit/test_input_validation.py` | capabilities 미참조 | **없음** |
| **프론트엔드 전체** | capabilities 호출 코드 없음 (아직 미연동) | **없음** |
| **scripts/ 전체** | capabilities 미참조 | **없음** |

---

## 3. 인증 체계 호환성 분석

### 현재 인증 패턴

프로젝트의 인증은 `resolve_current_actor()` (FastAPI Depends) + `authorization_service.authorize()` RBAC 패턴을 사용한다.

- `resolve_current_actor()` → `ActorContext` 반환 (anonymous 포함)
- `authorization_service.authorize(actor, action, resource)` → 권한 검사, 실패 시 401/403 raise
- Admin 엔드포인트는 `admin.read` / `admin.write` action으로 RBAC 제어

### 작업지시서 vs 실제 코드 괴리

작업지시서(task0-8.md)에서 가정한 `get_current_user` / `get_current_admin_user`는 **실제 코드에 존재하지 않는다.** 코드 수정 시 다음과 같이 매핑해야 한다:

| 작업지시서 가정 | 실제 코드 패턴 |
|----------------|---------------|
| `Depends(get_current_user)` | `Depends(resolve_current_actor)` + `authorization_service.authorize(actor, "system.read", ..., require_authenticated=True)` |
| `Depends(get_current_admin_user)` | `Depends(resolve_current_actor)` + `authorization_service.authorize(actor, "admin.read", ...)` |

### 필요 조치

1. `_PERMISSION_MATRIX`에 `"system.read"` action 추가 필요 (현재 미정의)
   - 허용 역할: 모든 인증된 역할 (`VIEWER` 이상)
2. Tier 3는 기존 `"admin.read"` action 재사용 가능

---

## 4. system.py 구체 변경 계획

### 4-1. 기존 health 엔드포인트 (변경 없음)

`GET /health`는 현재도 인증 불필요. Tier 1 역할을 이미 수행 중이므로 그대로 유지한다. 다만 응답에 `status` 필드 추가를 고려한다 (현재는 `{"healthy": true}`만 반환).

### 4-2. capabilities 엔드포인트 변경 (Tier 2)

```python
# 변경 전 (현재)
@router.get("/capabilities")
def get_capabilities(response: Response) -> SuccessResponse:
    # 인증 없이 모든 정보 반환

# 변경 후
@router.get("/capabilities")
def get_capabilities(
    response: Response,
    actor: ActorContext = Depends(resolve_current_actor),
) -> SuccessResponse:
    authorization_service.authorize(
        actor, "system.read", ResourceRef(resource_type="system"),
        require_authenticated=True,
    )
    # rag_available, chunking_enabled만 반환
    # pgvector_enabled, supported_providers 제외
```

### 4-3. admin capabilities 신규 추가 (Tier 3)

두 가지 방안:

- **방안 A**: `system.py`에 admin_router를 별도 정의하고 `router.py`에서 등록
- **방안 B**: 기존 `admin.py`에 admin/system/capabilities 엔드포인트 추가

기존 admin 라우터가 이미 `/admin` prefix로 등록되어 있으므로 **방안 B**가 라우팅 구조와 일관성이 높다.

```python
# admin.py에 추가
@router.get("/system/capabilities", summary="전체 시스템 정보 조회 (Admin)")
def get_admin_capabilities(
    response: Response,
    actor: ActorContext = Depends(resolve_current_actor),
) -> SuccessResponse:
    authorization_service.authorize(actor, "admin.read", ResourceRef(resource_type="admin"))
    # 전체 정보 반환 (pgvector, providers, deployment_type 등)
```

### 4-4. 캐시 구조

- Tier 2용 캐시와 Tier 3용 캐시를 분리 (반환 필드가 다르므로)
- 또는 내부적으로 `_build_capabilities()`는 동일하게 호출하고, 응답 시 필드를 필터링
- **권장**: 빌드 함수는 하나, 응답 필터만 tier별 분리 (캐시 효율)

---

## 5. 테스트 변경 계획

### 5-1. `test_system_capabilities.py` 전면 재작성

| 기존 테스트 | 변경 |
|------------|------|
| `test_accessible_without_auth` → 200 | Tier 2: → **401** (인증 필요) |
| `test_response_shape` (pgvector, providers 포함) | Tier 2: pgvector, providers **미포함** 검증 |
| `test_pgvector_*` 시리즈 | Tier 3(admin 인증)으로 이동 |
| `test_supported_providers_*` 시리즈 | Tier 3(admin 인증)으로 이동 |
| `test_cache_*` 시리즈 | Tier 2: `private`, Tier 1: `public` 분리 검증 |

### 5-2. 추가 테스트 케이스

- Tier 1: `/health` 비인증 접근 200, 내부 정보 미노출
- Tier 2: 비인증 → 401, 인증 → 200 + pgvector/providers 미노출
- Tier 3: 비인증 → 401, 일반 사용자 → 403, Admin → 200 + 전체 정보
- 정보 격리: 각 tier에서 다른 tier의 필드가 노출되지 않는지 교차 검증

---

## 6. 리스크 및 주의사항

### 6-1. 하위 호환성 (Breaking Change)

`GET /api/v1/system/capabilities`가 인증 불필요 → 인증 필요로 변경되므로 **breaking change**이다.

**영향 범위**: 프론트엔드가 아직 capabilities를 호출하지 않으므로 실질적 영향은 없다. 다만 외부 에이전트나 스크립트가 이미 사용 중이라면 인증 헤더 추가가 필요하다.

**완화 조치**: 변경 사항을 CHANGELOG에 기록하고, API 문서(Swagger)에서 인증 요구사항을 명시한다.

### 6-2. Permission Matrix 확장

`_PERMISSION_MATRIX`에 `"system.read"` action이 현재 없다. 이를 추가할 때 기존 RBAC 테스트(`test_auth.py`)에 영향이 없는지 확인 필요.

### 6-3. Phase 2 산출물과의 충돌 가능성

Phase 2에서 `system.py`, `router.py`, 인증 관련 파일을 수정했다면 merge conflict 가능성이 있다. Phase 2 최종 코드 상태에서 작업을 시작해야 한다.

---

## 7. 작업 순서 권장

1. `_PERMISSION_MATRIX`에 `"system.read"` action 추가 (모든 인증 역할 허용)
2. `system.py` 수정: capabilities에 `resolve_current_actor` + `authorize` 추가, 응답 필드 축소
3. `admin.py` 수정: `/system/capabilities` admin 엔드포인트 추가
4. `test_system_capabilities.py` 전면 재작성
5. 기존 테스트 전체 실행 → 회귀 확인
6. health 엔드포인트 `status` 필드 추가 (선택)

---

## 8. 결론

| 항목 | 판정 |
|------|------|
| 영향 범위 | **좁음** — 백엔드 3파일 수정, 프론트엔드 영향 없음 |
| 하위 호환성 | Breaking change이나 실질적 소비자 없음 |
| 인증 체계 호환 | 작업지시서와 실제 코드 패턴 괴리 있음 → 코드 수정 시 `resolve_current_actor` + `authorization_service` 패턴 사용 |
| 작업 난이도 | 중 (인증 의존성 추가 + 테스트 재작성) |
| 예상 소요 | 코드 수정 + 테스트 = 약 1~2시간 |
| 리스크 | 낮음 |
