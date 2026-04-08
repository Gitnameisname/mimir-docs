# Phase 0 — UI 분리와 API 경계 요약

## 문서 목적

이 문서는 User/Admin UI 분리 원칙이 API 설계, 권한 모델, 감사 요구사항과 어떻게 연결되는지 정리한다. UI 분리는 화면 구조의 문제가 아니라 API 경계와 권한 통제 구조의 문제이기도 하다. Phase 1 도메인 모델과 Phase 2 API 설계에 반영할 핵심 입력을 제공한다.

---

## 1. UI 분리와 API 경계의 관계

### 1-1. UI 분리는 API 경계 분리를 요구한다

User UI와 Admin UI가 같은 API 엔드포인트를 공유하더라도, 역할에 따라 노출되는 필드와 허용되는 동작이 달라야 한다. 단순히 UI에서 기능을 숨기는 것은 API 경계 분리가 아니다.

```
User UI     ──→  /api/v1/documents/{id}        (읽기·편집 범위)
Admin UI    ──→  /api/v1/documents/{id}        (위와 동일 + 운영 필드 포함)
            ──→  /api/v1/admin/documents/{id}  (상태 전이, 권한 변경 등 운영 액션)
```

### 1-2. API 경계 구분 방식

| 구분 | 경로 패턴 | 접근 조건 | 감사 요구 |
|------|----------|----------|----------|
| **사용자 도메인 API** | `/api/v1/{resource}` | 인증된 일반 사용자 | 선택적 (주요 변경 시) |
| **관리자 도메인 API** | `/api/v1/admin/{resource}` | 관리자 권한 필수 | 필수 (모든 변경) |
| **시스템 연동 API** | `/api/v1/integration/*` | 서비스 계정 | 필수 |

### 1-3. 같은 리소스의 역할별 응답 차이

동일한 `GET /api/v1/documents/{id}` 요청이더라도, 호출자의 역할에 따라 응답 범위가 다를 수 있다.

```json
// User 역할 응답
{
  "id": "...",
  "title": "...",
  "content": "...",
  "status": "published",
  "version": { "number": 3, "updatedAt": "..." }
}

// Admin 역할 응답 (추가 필드 포함)
{
  "id": "...",
  "title": "...",
  "content": "...",
  "status": "published",
  "version": { "number": 3, "updatedAt": "...", "updatedBy": "...", "approvedBy": "..." },
  "permissions": { "ownerId": "...", "accessPolicy": "..." },
  "audit": { "lastAuditAt": "...", "policyViolations": [] }
}
```

역할별 응답 필드 분리는 API 레이어에서 처리한다. UI에서 조건부로 필드를 숨기는 방식으로 대체하지 않는다.

---

## 2. 권한·감사 요구사항

### 2-1. User UI에서 발생하는 API 호출

| 행위 | API 예시 | 권한 | 감사 기록 |
|------|---------|------|---------|
| 문서 조회 | `GET /documents/{id}` | 일반 사용자 | 불필요 |
| 문서 작성 | `POST /documents` | 일반 사용자 | 필요 (생성 이벤트) |
| 문서 편집 | `PATCH /documents/{id}` | 편집 권한 보유자 | 필요 (변경 이벤트 + 버전 생성) |
| 검토 요청 | `POST /documents/{id}/review-requests` | 일반 사용자 | 필요 (상태 이벤트) |
| 첨부 업로드 | `POST /documents/{id}/attachments` | 편집 권한 보유자 | 필요 |

### 2-2. Admin UI에서 발생하는 API 호출

| 행위 | API 예시 | 권한 | 감사 기록 |
|------|---------|------|---------|
| 문서 승인 | `POST /admin/documents/{id}/approve` | 관리자 | 필수 (승인자·시간·사유 포함) |
| 상태 강제 변경 | `POST /admin/documents/{id}/status` | 관리자 | 필수 |
| 문서 영구 삭제 | `DELETE /admin/documents/{id}` | 관리자 | 필수 |
| 권한 변경 | `PUT /admin/documents/{id}/permissions` | 관리자 | 필수 |
| 문서 유형 수정 | `PUT /admin/document-types/{id}` | 관리자 | 필수 |
| 사용자 역할 변경 | `PUT /admin/users/{id}/roles` | 관리자 | 필수 |

### 2-3. UI 숨김과 API 권한 통제의 역할 구분

| 구분 | 역할 | 비고 |
|------|------|------|
| UI에서 버튼 숨김 | UX 편의 | 보안 수단이 아님 |
| API 레이어 역할 검사 | 실제 접근 통제 | 필수. UI 숨김과 독립적으로 반드시 존재 |
| 감사 로그 기록 | 추적 가능성 확보 | Admin 액션은 모두 기록 |

---

## 3. User / Admin API 설계 시 고려사항

### 3-1. 공통 도메인 API + 역할별 액션 API 조합

- 도메인 리소스 CRUD는 공통 API로 제공하되, 역할별 응답 범위를 다르게 한다.
- 운영 전용 액션(승인, 강제 상태 변경, 일괄 처리)은 `/admin/*` 경로로 분리한다.
- 사용자가 수행할 수 없는 액션은 API 레이어에서 `403 Forbidden`을 반환한다.

### 3-2. 코어 도메인 규칙의 단일화

- User UI와 Admin UI는 서로 다른 도메인 로직을 직접 구현하지 않는다.
- 두 UI 모두 동일한 API를 통해 도메인 서비스에 접근한다.
- 상태 전이 규칙, 버전 생성 정책, 메타데이터 검증 등은 API 레이어(도메인 서비스)에서 일관되게 적용된다.

### 3-3. Admin 액션의 강화된 감사 요구

- Admin UI에서 발생하는 모든 변경 요청은 감사 로그에 기록되어야 한다.
- 감사 로그에는 최소한 `actorId`, `action`, `resourceType`, `resourceId`, `timestamp`, `requestId`가 포함된다.
- Admin API 응답에 감사 추적 ID를 포함하여 이후 조회가 가능하게 한다.

### 3-4. 데이터 노출 정책

- User API 응답에는 운영·감사 관련 필드를 포함하지 않는다.
- Admin API 응답에는 운영 필드가 추가로 포함된다.
- 이 차이는 API 레이어의 직렬화(serialization) 레이어에서 처리한다. 동일 도메인 엔티티를 역할에 따라 다른 뷰로 직렬화한다.

---

## 4. Phase 1/2로 전달할 핵심 입력 요약

### Phase 1 (도메인 모델 설계)

- **Document 엔티티**는 사용자 관점 필드(title, content, status 표시용)와 운영 관점 필드(ownerHistory, policyViolations, approvalHistory)를 모두 수용할 수 있어야 한다.
- **Status 모델**은 사용자에게 보여줄 단순 상태 레이블과, 운영자에게 보여줄 전이 이력 및 정책 적합성 정보를 함께 포함해야 한다.
- **감사 필드** (`createdBy`, `updatedBy`, `approvedBy`, `deletedBy`)는 도메인 엔티티 수준에서 정의되어야 한다.
- **Version 엔티티**는 변경자·승인자 정보를 포함해야 하며, 이 정보의 역할별 노출 범위는 API에서 결정한다.

### Phase 2 (API·서비스 설계)

- 도메인 API 응답 직렬화 시, 호출자 역할에 따라 다른 응답 뷰를 제공하는 직렬화 전략을 정의한다.
- `/admin/*` 경로의 API는 공통 미들웨어에서 관리자 권한 검사와 감사 로그 기록을 자동으로 처리한다.
- UI 분리 원칙은 프론트엔드 라우팅 아키텍처(`/app/*` vs `/admin/*`)에 반영하고, 두 영역 간 직접 링크를 두지 않는다.
- 같은 도메인 서비스를 User API와 Admin API가 공유하되, 서비스 레이어 호출 시 호출자 컨텍스트(역할, 요청 ID)를 함께 전달한다.
