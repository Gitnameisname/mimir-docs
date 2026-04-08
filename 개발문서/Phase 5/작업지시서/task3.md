# 📘 Task 5-3 작업 지시서
## 역할 기반 상태 전이 제한 (RBAC 연동)

---

## 1. 🎯 작업 목적

Task 5-2에서 정의한 상태 전이(State Machine)에
**사용자 역할(Role)**을 결합하여,

👉 “누가 어떤 상태 전이를 수행할 수 있는가”를 명확히 정의한다.

이를 통해:

- 승인 절차의 무결성 확보
- 권한 없는 상태 변경 차단
- 조직 기반 문서 운영 정책 반영

이 가능해진다.

---

## 2. 📌 설계 범위

본 작업에서는 다음을 정의한다.

- 사용자 역할(Role) 정의
- 역할별 허용 액션 정의
- 상태 전이 + 역할 매핑
- 서버 검증 정책 (State + Role 결합)
- 예외 정책 (Admin override 등)

---

## 3. 🧩 역할 정의

기본 역할은 다음과 같이 정의한다.

```text
AUTHOR
REVIEWER
APPROVER
ADMIN
```

---

## 4. 🧠 역할별 책임 정의

### 4.1 AUTHOR

- 문서 작성 및 수정
- Draft 생성
- Review 요청
- 반려된 문서 재작업

---

### 4.2 REVIEWER

- 문서 검토 수행
- 코멘트 작성
- 반려 처리 가능

---

### 4.3 APPROVER

- 최종 승인 권한
- 승인 → Published 전 단계 결정

---

### 4.4 ADMIN

- 모든 상태 전이 수행 가능
- 시스템 운영자 권한
- 예외 상황 처리

---

## 5. 🔁 역할 기반 상태 전이 매핑

다음은 기본 정책이다.

```text
AUTHOR
- DRAFT → IN_REVIEW
- REJECTED → DRAFT

REVIEWER
- IN_REVIEW → REJECTED

APPROVER
- IN_REVIEW → APPROVED
- APPROVED → PUBLISHED

ADMIN
- 모든 전이 허용
```

---

## 6. 🏗 Permission Map 구조 예시

```python
from app.domain.document.enums import DocumentStatus
from app.domain.auth.enums import UserRole

WORKFLOW_PERMISSIONS: dict[UserRole, set[tuple[DocumentStatus, DocumentStatus]]] = {
    UserRole.AUTHOR: {
        (DocumentStatus.DRAFT, DocumentStatus.IN_REVIEW),
        (DocumentStatus.REJECTED, DocumentStatus.DRAFT),
    },
    UserRole.REVIEWER: {
        (DocumentStatus.IN_REVIEW, DocumentStatus.REJECTED),
    },
    UserRole.APPROVER: {
        (DocumentStatus.IN_REVIEW, DocumentStatus.APPROVED),
        (DocumentStatus.APPROVED, DocumentStatus.PUBLISHED),
    },
    UserRole.ADMIN: {
        # ADMIN은 모든 전이 허용 (별도 처리 권장)
    },
}
```

---

## 7. 🛡 서버 검증 정책

상태 전이는 다음 두 단계를 모두 통과해야 한다.

### 7.1 상태 전이 검증 (Task 5-2)

```text
ALLOWED_TRANSITIONS[current_status]에 target이 존재해야 한다
```

### 7.2 역할 기반 검증 (Task 5-3)

```text
(user_role, current_status → target_status) 조합이 허용되어야 한다
```

---

## 8. 🔐 검증 흐름 (중요)

```text
1. 현재 상태 조회
2. 요청된 target 상태 확인
3. 상태 전이 허용 여부 확인 (Task 5-2)
4. 사용자 역할 확인
5. 역할 기반 전이 허용 여부 확인
6. 모두 통과 시 상태 변경
7. 이력 기록
```

---

## 9. ⚠️ Admin 예외 정책

ADMIN은 다음 정책을 가진다.

- 모든 상태 전이 허용
- 단, 감사 로그 필수 기록

권장 정책:

```text
ADMIN override 사용 시 반드시 reason 입력
```

---

## 10. ⚠️ 확장 고려 사항

### 10.1 조직별 Role 커스터마이징

향후:

```text
- 팀별 Reviewer
- 문서별 Approver
- 조직 단위 ACL
```

로 확장 가능해야 한다.

---

### 10.2 다중 승인 구조

향후 확장:

```text
IN_REVIEW → (Reviewer 승인) → IN_APPROVAL → (Approver 승인)
```

---

## 11. 🚨 에러 처리

### 권한 없는 전이

```json
{
  "error": {
    "code": "FORBIDDEN_WORKFLOW_ACTION",
    "message": "User does not have permission to perform this workflow transition."
  }
}
```

---

## 12. 📦 산출물

- Role 정의
- WorkflowPermissionMap
- 상태 + 역할 결합 검증 정책
- Admin override 정책

---

## 13. 🚀 다음 작업 연결

👉 Task 5-4: Workflow Action API 설계

다음 내용을 정의한다.

- submit-review / approve / reject / publish API
- request schema
- service layer 구조

---

## 14. ✅ 완료 기준

- 역할별 전이 정책이 명확해야 한다
- 상태 전이 + 역할 검증이 분리되어야 한다
- 서버에서 강제되는 구조여야 한다
- 향후 조직/ACL 확장이 가능해야 한다
