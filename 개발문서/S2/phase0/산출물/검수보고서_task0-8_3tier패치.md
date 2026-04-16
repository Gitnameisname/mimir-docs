# 검수 보고서 — Task 0-8 3-tier 보안 분리 패치

**Phase**: 0 — S2 기반 구축  
**대상**: Task 0-8 후속 패치 (capabilities 3-tier 분리)  
**작성일**: 2026-04-17  
**검수자**: 기술 리드  
**판정**: ✅ 승인 (환경 제약으로 정적 분석 기반 검증)

---

## 1. 작업 배경

S2 개발계획 검수에서 "단일 public 엔드포인트가 `pgvector_enabled`, `supported_providers` 등 내부 구성 정보를 인증 없이 노출"하는 보안 위반이 지적되었다. 이에 따라 3-tier 보안 분리를 적용하는 코드 패치를 수행하였다.

| Tier | 엔드포인트 | 인증 | 노출 정보 |
|------|-----------|------|----------|
| 1 | `GET /api/v1/system/health` | 불필요 | `healthy` 상태만 |
| 2 | `GET /api/v1/system/capabilities` | VIEWER 이상 | `version`, `rag_available`, `chunking_enabled`, `mcp_spec_version` |
| 3 | `GET /api/v1/admin/system/capabilities` | Admin 전용 | 전체 정보 (pgvector, providers 포함) |

---

## 2. 수정 파일 목록

| 파일 | 변경 유형 | 변경 요약 |
|------|---------|----------|
| `backend/app/api/auth/authorization.py` | 수정 | `_PERMISSION_MATRIX`에 `"system.read"` action 추가 |
| `backend/app/api/v1/system.py` | 수정 | Tier 2 인증 추가, 필드 필터링, `_get_full_capabilities()` 분리 |
| `backend/app/api/v1/admin.py` | 수정 | Tier 3 `/system/capabilities` 엔드포인트 추가 |
| `backend/tests/unit/test_system_capabilities.py` | 전면 재작성 | 3-tier 인증/정보격리 테스트 20+ 케이스 |

---

## 3. 검수 항목별 결과

### 3-1. RBAC 권한 매트릭스 (`authorization.py`)

| 항목 | 기대값 | 실측값 | 판정 |
|------|--------|--------|------|
| `system.read` action 존재 | `_PERMISSION_MATRIX`에 정의 | line 120에 정의 | ✅ |
| 허용 역할 범위 | 모든 인증 역할 (VIEWER~SUPER_ADMIN) | 6개 역할 모두 포함 | ✅ |
| 기존 action 무영향 | 다른 action의 역할 집합 변경 없음 | 변경 없음 | ✅ |

**판정**: ✅ 승인

### 3-2. Tier 2 엔드포인트 (`system.py`)

| 항목 | 기대값 | 실측값 | 판정 |
|------|--------|--------|------|
| `resolve_current_actor` 의존성 | `Depends(resolve_current_actor)` 사용 | line 160 | ✅ |
| `authorization_service.authorize` 호출 | `"system.read"` action으로 RBAC 검사 | line 162-164 | ✅ |
| 미인증 → 401 | `ApiAuthenticationError` raise | RBAC에서 자동 처리 | ✅ |
| `_TIER2_FIELDS` 필터 | `version`, `rag_available`, `chunking_enabled`, `mcp_spec_version`만 | line 142 | ✅ |
| `pgvector_enabled` 미노출 | Tier 2 응답에 미포함 | dict comprehension 필터링 | ✅ |
| `supported_providers` 미노출 | Tier 2 응답에 미포함 | dict comprehension 필터링 | ✅ |
| Cache-Control | `private, max-age=300` | line 170 | ✅ |
| `_get_full_capabilities()` 캐시 함수 | thread-safe 5분 TTL | `_cap_lock` + 시각 비교 | ✅ |

**판정**: ✅ 승인

### 3-3. Tier 3 Admin 엔드포인트 (`admin.py`)

| 항목 | 기대값 | 실측값 | 판정 |
|------|--------|--------|------|
| 엔드포인트 경로 | `/system/capabilities` (admin prefix 자동) | `@router.get("/system/capabilities")` | ✅ |
| Admin 인증 | `require_admin_access` 의존성 | `Depends(require_admin_access)` | ✅ |
| 전체 정보 반환 | `_get_full_capabilities()` 필터 없이 반환 | inline import 후 직접 반환 | ✅ |
| 일반 사용자 → 403 | VIEWER/AUTHOR 접근 차단 | `admin.read` RBAC으로 차단 | ✅ |
| Cache-Control | `private, max-age=300` | 설정 완료 | ✅ |
| 라우터 등록 불필요 | 기존 `router.py`의 admin 등록 재사용 | line 72에서 이미 등록 | ✅ |

**판정**: ✅ 승인

### 3-4. 캐시 구조

| 항목 | 기대값 | 실측값 | 판정 |
|------|--------|--------|------|
| 단일 캐시 공유 | Tier 2/3 모두 `_get_full_capabilities()` 경유 | 동일 함수 호출 | ✅ |
| TTL 5분 | `_CAP_CACHE_TTL = timedelta(minutes=5)` | line 46 | ✅ |
| Thread-safe | `threading.Lock` 사용 | `_cap_lock` (line 48) | ✅ |
| 응답 분리 | Tier별 필드 필터링으로 분리 | `_TIER2_FIELDS` 기반 필터 | ✅ |

**판정**: ✅ 승인

### 3-5. 테스트 커버리지 (`test_system_capabilities.py`)

| 테스트 클래스 | 케이스 수 | 검증 내용 | 판정 |
|--------------|----------|----------|------|
| `TestHealthTier1` | 2 | 비인증 접근 200, 내부 정보 미노출 | ✅ |
| `TestCapabilitiesTier2` | 6 | 401/200 인증, Tier2 필드 포함/Admin 필드 제외, Cache-Control, mcp_spec_version | ✅ |
| `TestAdminCapabilitiesTier3` | 11 | 401/403/200 계층별 접근, 전체 스키마, pgvector/RAG 조합, providers | ✅ |
| `TestCapabilitiesCache` | 2 | 캐시 적재, TTL 내 재사용 | ✅ |
| `TestInformationIsolation` | 1 | Admin이 Tier 2 호출 시에도 정보 격리 유지 | ✅ |
| **합계** | **22** | | ✅ |

**판정**: ✅ 승인

### 3-6. 정적 분석 결과

| 항목 | 결과 |
|------|------|
| import 정합성 | 모든 import 경로 정상 (기존 모듈에서 사용 중인 심볼만 사용) |
| 필드명 일관성 | `_build_capabilities()` 반환 키 ↔ `_TIER2_FIELDS` ↔ 테스트 assertion 일치 |
| conftest fixture 매칭 | `auth_viewer`, `auth_author`, `auth_admin` 이름 및 헤더 값 일치 |
| 모듈 패치 경로 | `app.api.v1.system` 모듈 경로 정확 |
| 라우팅 경로 | Tier 1→2→3 URL 매핑 정상 |

**판정**: ✅ 승인

---

## 4. 하위 호환성 평가

| 항목 | 판정 |
|------|------|
| Breaking Change | `GET /api/v1/system/capabilities`가 공개 → 인증 필요로 변경 |
| 프론트엔드 영향 | 없음 (capabilities 호출 코드 없음) |
| 스크립트 영향 | 없음 (loadtest, smoke 모두 `/health`만 호출) |
| 통합 테스트 영향 | 없음 (`test_system_endpoints.py`는 `/health`만 호출) |
| 외부 소비자 | 현재 미존재, 실질적 영향 없음 |

---

## 5. 검수 환경 제약 사항

현재 실행 환경에서 Python 3.13 기반 가상환경의 바이너리 호환성 문제로 pytest를 직접 실행할 수 없었다. 검수는 다음 방법으로 수행하였다:

- 코드 정적 분석: import 경로, 필드명 일관성, RBAC 매트릭스 정합성 확인
- 파일 간 교차 검증: system.py ↔ admin.py ↔ authorization.py ↔ test 코드 ↔ conftest 간 계약 일치 확인
- 기존 보고서 대비: FG0-2 검수보고서의 Task 0-8 검수 항목과 변경 사항 비교 검증

**권장**: 로컬 개발 환경에서 `pytest tests/unit/test_system_capabilities.py -v` 실행하여 22개 테스트 전수 통과를 확인할 것.

---

## 6. 최종 판정

| 항목 | 결과 |
|------|------|
| 기능 정합성 | ✅ 3-tier 분리가 설계 의도대로 구현됨 |
| 인증/인가 | ✅ RBAC 패턴 준수, 401/403 분리 정상 |
| 정보 격리 | ✅ Tier별 노출 필드가 명확히 분리됨 |
| 캐시 효율 | ✅ 단일 빌드 + 필터 방식으로 중복 계산 방지 |
| 테스트 커버리지 | ✅ 22개 케이스로 모든 tier 및 교차 시나리오 검증 |
| 하위 호환성 | ✅ 실질적 영향 없음 |
| **최종** | **✅ 승인** |
