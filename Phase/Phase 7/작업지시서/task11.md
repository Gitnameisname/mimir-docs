# Task 7-11. Admin UI API 연계 정의

## 1. 작업 목적

각 Admin 관리 화면이 어떤 API 엔드포인트와 연결되는지 명확히 정리하고, 화면별 API 연계 구조를 정의한다.

이 작업의 핵심 목표는 다음과 같다.

- 모든 Admin 화면의 조회/생성/수정/삭제 액션과 API 엔드포인트를 매핑한다
- 권한 체크 포인트와 오류 응답 처리 방침을 정의한다
- 프론트엔드-백엔드 간 협업 기준을 확립한다

---

## 2. 작업 범위

### 포함 범위

- Admin 화면별 API 엔드포인트 매핑
- 각 액션의 HTTP 메서드 및 파라미터 정의
- 권한 체크 포인트 정리
- 오류 응답 코드 및 프론트엔드 피드백 정책 정의
- API 페이지네이션/필터 파라미터 공통 규약 정의

### 제외 범위

- 실제 API 구현
- 데이터베이스 쿼리 최적화

---

## 3. 선행 조건

- Task 7-1 ~ 7-10 설계 완료 (또는 초안)
- Phase 3: 공통 API 규약 정의 완료

---

## 4. 주요 설계 대상

### 4-1. 공통 API 규약

모든 Admin API에 적용되는 공통 규약:

#### 인증/인가

- 모든 Admin API는 JWT Bearer 토큰 필요
- 관리자 권한 검증 (SUPER_ADMIN 또는 ORG_ADMIN)

#### 페이지네이션

```
GET /admin/users?page=1&limit=20&sort=created_at&order=desc
```

#### 필터 파라미터

```
GET /admin/users?status=ACTIVE&role=EDITOR&search=john
```

#### 오류 응답 형식

```json
{
  "error": {
    "code": "PERMISSION_DENIED",
    "message": "접근 권한이 없습니다.",
    "details": {}
  }
}
```

---

### 4-2. 사용자/조직/역할 관리 API 매핑

| 화면 액션 | HTTP | 엔드포인트 | 권한 |
|----------|------|-----------|------|
| 사용자 목록 조회 | GET | `/admin/users` | ADMIN |
| 사용자 상세 조회 | GET | `/admin/users/{user_id}` | ADMIN |
| 사용자 상태 변경 | PATCH | `/admin/users/{user_id}/status` | SUPER_ADMIN |
| 사용자 역할 변경 | PATCH | `/admin/users/{user_id}/roles` | SUPER_ADMIN |
| 조직 목록 조회 | GET | `/admin/organizations` | ADMIN |
| 조직 상세 조회 | GET | `/admin/organizations/{org_id}` | ADMIN |
| 역할 목록 조회 | GET | `/admin/roles` | ADMIN |
| 역할 상세 조회 | GET | `/admin/roles/{role_id}` | ADMIN |

---

### 4-3. 권한 정책 관리 API 매핑

| 화면 액션 | HTTP | 엔드포인트 | 권한 |
|----------|------|-----------|------|
| 정책 목록 조회 | GET | `/admin/permissions` | SUPER_ADMIN |
| 정책 상세 조회 | GET | `/admin/permissions/{policy_id}` | SUPER_ADMIN |
| 정책 수정 | PUT | `/admin/permissions/{policy_id}` | SUPER_ADMIN |
| 정책 상태 변경 | PATCH | `/admin/permissions/{policy_id}/status` | SUPER_ADMIN |
| 역할-정책 매핑 조회 | GET | `/admin/roles/{role_id}/permissions` | SUPER_ADMIN |

---

### 4-4. DocumentType 관리 API 매핑

| 화면 액션 | HTTP | 엔드포인트 | 권한 |
|----------|------|-----------|------|
| 타입 목록 조회 | GET | `/admin/document-types` | ADMIN |
| 타입 상세 조회 | GET | `/admin/document-types/{type_code}` | ADMIN |
| 타입 상태 변경 | PATCH | `/admin/document-types/{type_code}/status` | SUPER_ADMIN |

---

### 4-5. 감사 로그 API 매핑

| 화면 액션 | HTTP | 엔드포인트 | 권한 |
|----------|------|-----------|------|
| 감사 로그 목록 조회 | GET | `/admin/audit-logs` | ADMIN |
| 감사 로그 상세 조회 | GET | `/admin/audit-logs/{event_id}` | ADMIN |

**필터 파라미터**: `from`, `to`, `actor_id`, `resource_type`, `event_type`, `severity`, `result`

---

### 4-6. 시스템 대시보드 API 매핑

| 화면 섹션 | HTTP | 엔드포인트 | 권한 |
|----------|------|-----------|------|
| 핵심 지표 카드 | GET | `/admin/dashboard/metrics` | ADMIN |
| 컴포넌트 상태 | GET | `/admin/dashboard/health` | ADMIN |
| 최근 오류 요약 | GET | `/admin/dashboard/errors` | ADMIN |
| 최근 감사 이벤트 | GET | `/admin/dashboard/recent-audit-logs` | ADMIN |

---

### 4-7. 인덱싱 상태 API 매핑

| 화면 액션 | HTTP | 엔드포인트 | 권한 |
|----------|------|-----------|------|
| 인덱싱 목록 조회 | GET | `/admin/indexing/jobs` | ADMIN |
| 인덱싱 상세 조회 | GET | `/admin/indexing/jobs/{job_id}` | ADMIN |
| 인덱싱 재시도 | POST | `/admin/indexing/jobs/{job_id}/retry` | ADMIN |
| 일괄 재시도 | POST | `/admin/indexing/jobs/retry-failed` | SUPER_ADMIN |

---

### 4-8. API 키 관리 API 매핑

| 화면 액션 | HTTP | 엔드포인트 | 권한 |
|----------|------|-----------|------|
| API 키 목록 조회 | GET | `/admin/api-keys` | ADMIN |
| API 키 발급 | POST | `/admin/api-keys` | SUPER_ADMIN |
| API 키 상세 조회 | GET | `/admin/api-keys/{key_id}` | ADMIN |
| API 키 폐기 | DELETE | `/admin/api-keys/{key_id}` | SUPER_ADMIN |
| 외부 연계 목록 조회 | GET | `/admin/integrations` | ADMIN |
| 외부 연계 상세 조회 | GET | `/admin/integrations/{integration_id}` | ADMIN |

---

### 4-9. 백그라운드 작업 API 매핑

| 화면 액션 | HTTP | 엔드포인트 | 권한 |
|----------|------|-----------|------|
| 작업 목록 조회 | GET | `/admin/jobs` | ADMIN |
| 작업 상세 조회 | GET | `/admin/jobs/{job_id}` | ADMIN |
| 작업 취소 | POST | `/admin/jobs/{job_id}/cancel` | ADMIN |
| 작업 강제 종료 | POST | `/admin/jobs/{job_id}/force-stop` | SUPER_ADMIN |
| 작업 재시도 | POST | `/admin/jobs/{job_id}/retry` | ADMIN |

---

### 4-10. 오류 응답 처리 정책

| HTTP 코드 | 코드 | 프론트엔드 처리 |
|-----------|------|---------------|
| 400 | VALIDATION_ERROR | 폼 필드 오류 표시 |
| 401 | UNAUTHORIZED | 로그인 페이지 리다이렉트 |
| 403 | PERMISSION_DENIED | 권한 없음 안내 Toast |
| 404 | NOT_FOUND | "항목을 찾을 수 없습니다" 안내 |
| 409 | CONFLICT | 충돌 내용 상세 안내 |
| 500 | INTERNAL_ERROR | 일반 오류 Toast + 재시도 안내 |

---

## 5. 산출물

1. Admin UI API 연계 매핑 표 (화면별 엔드포인트 정리)
2. 공통 API 규약 문서 (페이지네이션, 필터, 오류 형식)
3. 권한 체크 포인트 정리서
4. 오류 응답 처리 정책 문서

---

## 6. 완료 기준

- 모든 Admin 화면의 API 엔드포인트 매핑이 정리되어 있다
- 각 엔드포인트의 HTTP 메서드, 파라미터, 권한이 명확히 기술되어 있다
- 오류 응답 코드와 프론트엔드 처리 방침이 정의되어 있다
- 공통 규약(페이지네이션, 필터)이 일관되게 정의되어 있다

---

## 7. Codex 작업 지침

- 코드 작성 금지 (설계 중심)
- API 매핑 표는 명확하고 일관된 형식으로 정리한다
- 권한 수준은 ADMIN (ORG_ADMIN 포함)과 SUPER_ADMIN으로 구분하여 명시한다
- 오류 응답 처리는 운영자 UX를 기준으로 정의한다 (기술적 오류보다 이해 가능한 안내)
- Phase 3의 공통 API 규약과 일관성을 유지한다
