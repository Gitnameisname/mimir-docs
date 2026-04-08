

# 📘 Task 5-2 작업 지시서
## Workflow 상태 전이(State Machine) 설계

---

## 1. 🎯 작업 목적

Task 5-1에서 정의한 `DocumentStatus`를 바탕으로,
문서 버전이 어떤 상태에서 어떤 상태로 이동할 수 있는지 **명시적 상태 전이 규칙(State Transition Rules)** 을 정의한다.

이 작업의 목적은 다음과 같다.

- 허용 가능한 상태 이동을 표준화한다.
- 불법 상태 전이를 서버 레벨에서 차단할 수 있게 한다.
- 이후 권한 제어(Task 5-3), Workflow Action API(Task 5-4), 이력 기록(Task 5-7)의 기반을 만든다.

---

## 2. 📌 설계 범위

본 작업에서는 다음을 정의한다.

- 상태 전이 맵(Transition Map)
- 전이별 의미 정의
- 허용 전이 / 금지 전이 규칙
- 서버 측 검증 정책
- 상태 전이 실패 시 에러 처리 원칙

주의:
- **역할별 전이 가능 여부는 Task 5-3에서 정의한다.**
- 본 Task는 먼저 **상태 자체의 이동 가능성**만 정의한다.

---

## 3. 🧩 기본 상태 목록

Task 5-1에서 정의된 기본 상태는 다음과 같다.

```text
DRAFT
IN_REVIEW
APPROVED
PUBLISHED
REJECTED
ARCHIVED
```

---

## 4. 🔁 허용 상태 전이 정의

기본 허용 전이는 아래와 같이 정의한다.

```text
DRAFT      -> IN_REVIEW
IN_REVIEW  -> APPROVED
IN_REVIEW  -> REJECTED
REJECTED   -> DRAFT
APPROVED   -> PUBLISHED
PUBLISHED  -> ARCHIVED
ARCHIVED   -> DRAFT      (정책 선택 사항)
```

추가 정책:

```text
PUBLISHED  -> DRAFT      (직접 전이 금지)
APPROVED   -> DRAFT      (기본적으로 금지)
IN_REVIEW  -> DRAFT      (기본적으로 금지, 필요 시 withdraw 액션으로 별도 정의 가능)
```

---

## 5. 🧠 상태 전이 의미 정의

### 5.1 DRAFT → IN_REVIEW

- 작성자가 검토 요청을 보낸 상태
- 문서가 초안 단계에서 검토 단계로 진입
- 이후 Reviewer가 검토 가능

### 5.2 IN_REVIEW → APPROVED

- 검토 또는 승인 절차가 통과된 상태
- 게시 직전 단계
- 내용 변경은 금지하는 것이 바람직함

### 5.3 IN_REVIEW → REJECTED

- 검토 결과 반려된 상태
- 반려 사유(comment/reason)를 남겨야 함
- 이후 작성자가 재작업 수행

### 5.4 REJECTED → DRAFT

- 반려된 문서를 다시 편집 가능한 상태로 복귀
- 재작업 후 다시 검토 요청 가능

### 5.5 APPROVED → PUBLISHED

- 승인 완료된 버전을 공식 게시
- 외부 조회 가능 상태로 전환
- 게시 이후 해당 Version은 immutable로 취급

### 5.6 PUBLISHED → ARCHIVED

- 더 이상 운영 대상이 아닌 게시 문서를 보관 상태로 전환
- 조회는 가능하되 운영상 비활성으로 본다

### 5.7 ARCHIVED → DRAFT (Optional)

- 보관 문서를 기반으로 재작성/재운영이 필요할 때 사용
- 단, 기존 Version을 직접 되살리는 것보다는
  **새 Version 생성 후 DRAFT 시작**이 더 권장된다.

---

## 6. ❌ 금지 상태 전이 정의

다음 전이는 기본적으로 금지한다.

```text
DRAFT      -> APPROVED
DRAFT      -> PUBLISHED
IN_REVIEW  -> PUBLISHED
REJECTED   -> PUBLISHED
REJECTED   -> APPROVED
PUBLISHED  -> APPROVED
PUBLISHED  -> REJECTED
ARCHIVED   -> PUBLISHED
```

금지 이유:

- 승인 절차 우회 방지
- 게시 후 상태 롤백의 모호성 방지
- 운영 이력 왜곡 방지
- 감사 추적 일관성 유지

---

## 7. 🏗 상태 전이 맵 구조 예시

서버 코드에서는 아래와 같은 전이 맵 형태를 권장한다.

```python
from app.domain.document.enums import DocumentStatus

ALLOWED_TRANSITIONS: dict[DocumentStatus, set[DocumentStatus]] = {
    DocumentStatus.DRAFT: {
        DocumentStatus.IN_REVIEW,
    },
    DocumentStatus.IN_REVIEW: {
        DocumentStatus.APPROVED,
        DocumentStatus.REJECTED,
    },
    DocumentStatus.REJECTED: {
        DocumentStatus.DRAFT,
    },
    DocumentStatus.APPROVED: {
        DocumentStatus.PUBLISHED,
    },
    DocumentStatus.PUBLISHED: {
        DocumentStatus.ARCHIVED,
    },
    DocumentStatus.ARCHIVED: set(),
}
```

선택적으로 `ARCHIVED -> DRAFT` 를 허용하려면 정책 확정 후 추가한다.

---

## 8. 🛡 서버 검증 정책

상태 전이는 반드시 **서버에서 검증**되어야 한다.

### 8.1 검증 원칙

- 클라이언트가 target status를 보냈더라도 서버가 직접 검증한다.
- 현재 상태를 DB에서 다시 조회한다.
- 전이 맵에 없는 이동은 거부한다.

### 8.2 검증 흐름 예시

```text
1. 현재 Version 상태 조회
2. 요청된 target 상태 확인
3. ALLOWED_TRANSITIONS[current_status] 내 포함 여부 확인
4. 허용 시 상태 전이 수행
5. 이력 기록 생성
6. 감사 로그 생성
```

---

## 9. ⚠️ 동시성 및 무결성 고려 사항

### 9.1 낙관적 락(Optimistic Lock)

동일 문서 버전에 대해 동시에 여러 사용자가 상태 전이를 시도할 수 있으므로,
버전 번호 또는 updated_at 기반 검증이 필요하다.

예:
- 클라이언트가 기대한 현재 상태와 서버의 실제 상태가 다르면 실패
- 이미 다른 사용자가 승인한 경우 재승인 요청 거부

### 9.2 Published 이후 수정 금지

`PUBLISHED` 상태의 Version은 직접 수정하지 않는다.
수정이 필요하면:

```text
새 Version 생성 -> DRAFT 시작
```

### 9.3 이력 불변성

상태 전이가 일어나면 반드시 WorkflowHistory에 남겨야 하며,
사후 수정은 금지하는 것이 바람직하다.

---

## 10. 🚨 에러 처리 원칙

허용되지 않은 상태 전이는 명확한 오류로 반환한다.

예시:

- `400 Bad Request`: 잘못된 상태 값
- `409 Conflict`: 현재 상태 기준으로 허용되지 않는 전이
- `403 Forbidden`: 상태 전이는 가능하지만 해당 사용자가 권한 없음 (Task 5-3)

예시 메시지:

```json
{
  "error": {
    "code": "INVALID_WORKFLOW_TRANSITION",
    "message": "Cannot transition document version from IN_REVIEW to PUBLISHED."
  }
}
```

---

## 11. 📦 산출물

- 상태 전이 정책 문서 (본 문서)
- `ALLOWED_TRANSITIONS` 구조 초안
- 금지 전이 목록
- 서버 검증 정책 정의
- 에러 처리 원칙

---

## 12. 🚀 다음 작업 연결

다음 Task에서 수행:

👉 Task 5-3: 역할 기반 상태 전이 제한(RBAC 연동)

다음 내용을 정의한다.

- 어떤 역할이 어떤 전이를 수행할 수 있는가
- Author / Reviewer / Approver / Admin 권한 분리
- 상태 전이 + 역할 조합 검증 정책

---

## 13. ✅ 완료 기준

- 모든 기본 상태 전이가 정의되어 있어야 한다.
- 허용 전이와 금지 전이가 명확히 구분되어야 한다.
- 서버 검증 구조가 바로 구현 가능한 수준이어야 한다.
- 이후 RBAC 연동 및 API 설계로 자연스럽게 이어질 수 있어야 한다.