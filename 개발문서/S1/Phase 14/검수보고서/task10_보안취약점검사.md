# Task 14-10 보안 취약점 검사 보고서

**작성일**: 2026-04-14
**검사 대상**: 사용자/조직/역할 관리 UI + 권한 매트릭스 API/뷰

---

## 1. 검사 항목 및 결과

### 1.1 접근 제어 (Backend RBAC)

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 1 | 매트릭스 API에 require_admin_access 적용 | ✅ 통과 | ORG_ADMIN 이상만 조회 가능 |
| 2 | admin.write 권한 SUPER_ADMIN 전용 유지 | ✅ 통과 | `_PERMISSION_MATRIX["admin.write"]` |
| 3 | 사용자/조직/역할 삭제 백엔드 SUPER_ADMIN 검증 | ✅ 통과 | Phase 7에서 구현, 변경 없음 |
| 4 | 권한 매트릭스 API는 GET 전용 (조회) | ✅ 통과 | 매트릭스 변경은 코드 배포로만 가능 |

### 1.2 접근 제어 (Frontend 가드)

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 5 | 사용자 삭제 버튼 SUPER_ADMIN 가드 | ✅ 통과 | `useAuth().hasRole("SUPER_ADMIN")` |
| 6 | 조직 삭제 버튼 SUPER_ADMIN 가드 | ✅ 통과 | 동일 패턴 |
| 7 | UI 가드 우회 시에도 백엔드가 차단 | ✅ 통과 | 이중 검증 (UI 보조 + 백엔드 진짜 권한) |
| 8 | 모든 /admin/* 페이지 AuthGuard ORG_ADMIN | ✅ 통과 | Task 14-9 AdminLayout 상속 |

### 1.3 SQL Injection / Input Validation

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 9 | list_users role 파라미터 파라미터 바인딩 | ✅ 통과 | psycopg `%s` 플레이스홀더 사용 |
| 10 | search/status/role 모두 prepared statement | ✅ 통과 | f-string은 WHERE 골격만 사용, 값은 params |
| 11 | role_id 경로 파라미터 검증 | ✅ 통과 | UUID 변환 시 ValueError 핸들링 (Phase 7) |

### 1.4 XSS 방어

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 12 | PermissionMatrix dangerouslySetInnerHTML 미사용 | ✅ 통과 | grep 검증 |
| 13 | action 문자열 JSX 바인딩 (이스케이프) | ✅ 통과 | `{item.action}` |
| 14 | 그룹 라벨은 화이트리스트 매핑 | ✅ 통과 | `GROUP_LABELS[name] ?? name` |
| 15 | 사용자 입력은 React 기본 이스케이프 | ✅ 통과 | display_name, email 등 |

### 1.5 정보 노출 / 권한 분리

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 16 | 권한 매트릭스 노출의 안전성 검토 | ✅ 통과 | RBAC 매트릭스는 운영 정보가 아닌 정책 정보 (관리자 대상 한정 조회) |
| 17 | 매트릭스에 사용자 데이터 미포함 | ✅ 통과 | action/role 문자열만 |
| 18 | 토큰/시크릿 하드코딩 없음 | ✅ 통과 | grep 검증 |

### 1.6 CSRF / 상태 변경

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 19 | 매트릭스 API는 GET only (CSRF 무관) | ✅ 통과 | |
| 20 | 사용자/조직/역할 변경 API는 Phase 7 패턴 유지 | ✅ 통과 | Bearer 토큰 인증 |
| 21 | 삭제 동작에 확인 다이얼로그 | ✅ 통과 | deleteConfirm 모달 (Phase 7) |

### 1.7 클라이언트 사이드 보안

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 22 | 매트릭스 데이터를 localStorage에 저장 안 함 | ✅ 통과 | React Query in-memory cache만 사용 |
| 23 | URL 파라미터에 민감 정보 없음 | ✅ 통과 | role/status는 enum 값만 |
| 24 | 외부 링크 없음 (open redirect 방어) | ✅ 통과 | next/link 내부 경로만 |

### 1.8 접근성 보안

| # | 검사 항목 | 결과 | 비고 |
|:-:|---------|:----:|------|
| 25 | 키보드로 매트릭스 토글 가능 (button + aria-expanded) | ✅ 통과 | 마우스/터치 없이도 사용 가능 |
| 26 | 스크린리더에 매트릭스 의미 전달 | ✅ 통과 | `aria-label="역할-권한 매트릭스"`, 셀별 aria-label |

---

## 2. 취약점 요약

| 등급 | 수량 | 상세 |
|:----:|:----:|------|
| Critical | 0 | - |
| High | 0 | - |
| Medium | 0 | - |
| Low | 0 | - |
| Info | 2 | 아래 참조 |

### Info 항목

**INFO-1: 권한 매트릭스 정보 노출 가능성**
- 위치: `GET /api/v1/admin/roles/permissions/matrix`
- 설명: ORG_ADMIN 이상 사용자에게 시스템 전체 RBAC 매트릭스(action 목록 + 역할 매핑)가 노출됨.
- 위험도: 정보 (관리자 권한자만 접근 가능, 코드 베이스에도 동일 정보 존재 — `authorization.py`)
- 권고: 향후 SUPER_ADMIN 전용으로 좁히는 것을 고려할 수 있으나, "어떤 역할이 무엇을 할 수 있는가"는 ORG_ADMIN의 정상 업무 정보이므로 현재 적정.

**INFO-2: UI 삭제 가드는 보조 통제**
- 위치: `AdminUserDetailPage.tsx`, `AdminOrgsPage.tsx`
- 설명: SUPER_ADMIN 가드는 UI에서만 적용. 직접 API 호출 시 백엔드가 차단함.
- 위험도: 정보 (이중 검증 — 백엔드의 admin.write SUPER_ADMIN 체크가 진짜 권한 검사)
- 권고: 현재 구조 적절. UI는 ORG_ADMIN의 사용성 개선 목적.

---

## 3. 외부 라이브러리 취약점

Task 14-10에서 **신규 추가된 의존성 없음**. 기존 라이브러리(`@tanstack/react-query`)만 사용. CVE 검사 결과 critical/high 0건.

---

## 4. 결론

**Critical/High/Medium/Low 취약점 0건**. 권한 매트릭스 API는 ORG_ADMIN 가드, SUPER_ADMIN-only 삭제 동작은 UI/백엔드 이중 가드, SQL 파라미터 바인딩, React 기본 XSS 방어, dangerouslySetInnerHTML 미사용 등 보안 통제 모두 충족. Info 2건은 정상 운영 정보 / 보조 통제 설계에 따른 의도된 사항임.
