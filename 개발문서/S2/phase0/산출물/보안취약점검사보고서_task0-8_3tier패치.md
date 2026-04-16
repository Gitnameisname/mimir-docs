# 보안 취약점 검사 보고서 — Task 0-8 3-tier 보안 분리 패치

**Phase**: 0 — S2 기반 구축  
**대상**: Task 0-8 후속 패치 (capabilities 3-tier 분리)  
**작성일**: 2026-04-17  
**검사자**: 기술 리드  
**판정**: ✅ Critical 0건 / High 0건

---

## 1. 검사 범위

| 파일 | 변경 유형 |
|------|---------|
| `backend/app/api/auth/authorization.py` | 수정 (`system.read` action 추가) |
| `backend/app/api/v1/system.py` | 수정 (인증 추가, 필드 필터링, 캐시 분리) |
| `backend/app/api/v1/admin.py` | 수정 (Tier 3 엔드포인트 추가) |
| `backend/tests/unit/test_system_capabilities.py` | 전면 재작성 |

---

## 2. 이전 보안 이슈 해소 현황

이번 패치는 FG0-2 보안 검사에서 지적된 정보 노출 리스크를 해소한다.

| 이전 ID | 이전 내용 | 해소 여부 | 설명 |
|---------|----------|----------|------|
| SEC-FG02-12 | `supported_providers`를 공개 엔드포인트에서 반환 | ✅ 해소 | Tier 3 (Admin 전용)으로 이동 |
| SEC-FG02-13 | `version` 공개 노출 | 유지 (Low) | Tier 2 (인증 필요)로 격상, 공개 정보이므로 허용 수준 |
| SEC-FG02-15 | capabilities 레이트 리밋 미적용 | 일부 완화 | 인증 필수화로 무차별 스캔 차단, 레이트 리밋은 Phase 1 예정 |

---

## 3. 취약점 검사 결과

### 3-1. authorization.py — `system.read` action 추가

| ID | 분류 | 내용 | 심각도 | 조치 |
|----|------|------|--------|------|
| SEC-T08-01 | 권한 상승 | `system.read`가 VIEWER 이상 모든 역할에 허용 — 설계 의도에 부합, 과도한 권한 아님 | - | 해당 없음 |
| SEC-T08-02 | RBAC 우회 | 기존 action 역할 집합 무변경 — 패치로 인한 권한 확장 없음 | - | 해당 없음 |

**판정**: ✅ 이슈 없음

### 3-2. system.py — Tier 2 인증 및 필드 필터링

| ID | 분류 | 내용 | 심각도 | 조치 |
|----|------|------|--------|------|
| SEC-T08-03 | 정보 노출 (해소) | `pgvector_enabled`, `supported_providers`가 더 이상 Tier 2에서 반환되지 않음 — `_TIER2_FIELDS` 화이트리스트 방식으로 필터링 | - | ✅ 기존 리스크 해소 |
| SEC-T08-04 | 필터 우회 | `_TIER2_FIELDS`는 화이트리스트(포함 목록) 방식 — `_build_capabilities()`에 새 필드가 추가되어도 명시적으로 `_TIER2_FIELDS`에 추가하지 않으면 Tier 2에 노출되지 않음 (안전한 기본값) | - | 해당 없음 |
| SEC-T08-05 | 인증 우회 | `resolve_current_actor`는 토큰 미제공 시 `ANONYMOUS` actor 반환 → `authorize()`에서 `require_authenticated=True`(기본값)로 401 raise — 우회 경로 없음 | - | 해당 없음 |
| SEC-T08-06 | 캐시 오염 | `_cap_cache`는 서버 사이드 모듈 변수, 사용자 입력으로 직접 조작 불가 — `_cap_lock`으로 race condition 방지 | - | 해당 없음 |
| SEC-T08-07 | Cache-Control | `public` → `private`로 변경 — 인증 응답이 CDN/프록시 캐시에 저장되지 않음 | - | ✅ 개선됨 |

**판정**: ✅ 이슈 없음 (기존 2건 해소)

### 3-3. admin.py — Tier 3 Admin 엔드포인트

| ID | 분류 | 내용 | 심각도 | 조치 |
|----|------|------|--------|------|
| SEC-T08-08 | 권한 검증 | `require_admin_access` 의존성이 `admin.read` action으로 RBAC 검사 — VIEWER/AUTHOR는 403 | - | 해당 없음 |
| SEC-T08-09 | 정보 노출 | 전체 capabilities를 Admin에게만 반환 — `pgvector_enabled`, `supported_providers` 등은 관리자 수준 정보로 적절한 접근 제어 | - | 해당 없음 |
| SEC-T08-10 | inline import | `_get_full_capabilities`를 함수 내부에서 import — 순환 참조 방지 목적, 보안 영향 없음 | - | 해당 없음 |
| SEC-T08-11 | API 키 간접 노출 | `supported_providers`에 `"openai"`, `"anthropic"` 문자열만 포함 — 키 값 자체는 노출하지 않음, Admin 접근 범위에서 허용 수준 | - | 해당 없음 |

**판정**: ✅ 이슈 없음

### 3-4. test_system_capabilities.py — 테스트 코드

| ID | 분류 | 내용 | 심각도 | 조치 |
|----|------|------|--------|------|
| SEC-T08-12 | 테스트 인증 | `X-Actor-Id`, `X-Actor-Role` 헤더는 `DEBUG=True` 환경에서만 유효 — production에서 JWT 인증으로 자동 전환 | - | 해당 없음 |
| SEC-T08-13 | 테스트 데이터 | `patch.dict("os.environ", {"OPENAI_API_KEY": "sk-test"})` — 테스트 전용 더미 값, 실제 키 아님 | - | 해당 없음 |

**판정**: ✅ 이슈 없음

---

## 4. 보안 개선 효과 분석

이번 패치의 핵심 보안 개선 사항을 정리한다.

### 4-1. 정보 노출 최소화 (Principle of Least Privilege)

| 변경 전 | 변경 후 |
|---------|---------|
| 비인증 사용자가 `pgvector_enabled` 확인 가능 | Admin만 확인 가능 |
| 비인증 사용자가 `supported_providers` 확인 가능 | Admin만 확인 가능 |
| 비인증 사용자가 내부 인프라 구성 유추 가능 | 인증 없이는 health 상태만 확인 가능 |

### 4-2. 캐시 보안

| 변경 전 | 변경 후 |
|---------|---------|
| `Cache-Control: public, max-age=300` | `Cache-Control: private, max-age=300` |
| CDN/프록시가 응답을 캐시하여 타 사용자에게 전달 가능 | 브라우저 로컬 캐시만 허용 |

### 4-3. 인증 강화

| 변경 전 | 변경 후 |
|---------|---------|
| `/capabilities` 공개 접근 | 인증 필수 (401 for anonymous) |
| 무차별 정보 수집 가능 | 인증 토큰 필요, 감사 로그 기록 가능 |

---

## 5. 잔여 리스크 및 권고

| ID | 분류 | 내용 | 심각도 | 권고 |
|----|------|------|--------|------|
| SEC-T08-R01 | 레이트 리밋 | Tier 2 capabilities에 레이트 리밋 미적용 — 인증 필수화로 무차별 스캔은 차단되나, 인증된 사용자의 과도한 호출은 여전히 가능 | Low | Phase 1에서 slowapi 레이트 리밋 적용 |
| SEC-T08-R02 | 감사 로그 | capabilities 조회에 대한 감사 로그 미기록 — 현재 접근 통제만 적용 | Low | Phase 3 감사 로그 구현 시 포함 |

---

## 6. 전체 요약

| 심각도 | 건수 | 처리 |
|--------|------|------|
| Critical | 0 | — |
| High | 0 | — |
| Medium | 0 | — |
| Low | 2 | 잔여 리스크로 후속 Phase에서 해소 예정 |
| **해소된 기존 리스크** | **2** | SEC-FG02-12 (providers 공개 노출), SEC-FG02-15 (무차별 스캔 일부 완화) |

**최종 판정**: ✅ 승인 — Critical/High 없음, 기존 정보 노출 리스크 2건 해소
