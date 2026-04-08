

# 📘 Task 5-4 작업 지시서
## Workflow Action API 설계

---

## 1. 🎯 작업 목적

Task 5-1 ~ 5-3에서 정의한 내용을 바탕으로,
문서 워크플로를 실제 시스템에서 실행할 수 있도록 **Workflow Action API**를 설계한다.

이 API는 다음 목적을 가진다.

- 문서 상태 전이를 HTTP API로 표준화한다.
- UI / 외부 시스템 / 자동화 도구가 동일한 계약으로 워크플로를 제어할 수 있게 한다.
- 상태 전이 검증, 권한 검증, 이력 기록을 일관된 요청 흐름 안에 묶는다.

---

## 2. 📌 설계 범위

본 작업에서는 다음을 정의한다.

- Workflow 액션 단위 API 목록
- 각 API의 요청/응답 규격
- 상태 전이 검증 흐름
- RBAC 검증 연결 지점
- 서비스 레이어 책임 범위
- 오류 처리 원칙
- 감사 / 이력 기록 연계 포인트

주의:
- 실제 DB 모델 확장은 후속 Task에서 구체화할 수 있다.
- 본 Task는 먼저 **API 계약과 처리 흐름**을 확정하는 데 집중한다.

---

## 3. 🧩 기본 액션 정의

기본 Workflow 액션은 다음과 같이 정의한다.

```text
submit_review
approve
reject
publish
archive
return_to_draft   (선택적)
```

각 액션은 내부적으로 특정 상태 전이와 매핑된다.

| Action | From | To |
|---|---|---|
| submit_review | DRAFT | IN_REVIEW |
| approve | IN_REVIEW | APPROVED |
| reject | IN_REVIEW | REJECTED |
| publish | APPROVED | PUBLISHED |
| archive | PUBLISHED | ARCHIVED |
| return_to_draft | REJECTED | DRAFT |

---

## 4. 🌐 API 엔드포인트 설계

권장 API 구조는 다음과 같다.

```http
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/submit-review
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/approve
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/reject
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/publish
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/archive
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/return-to-draft
```

설계 이유:

- workflow는 **Document가 아니라 Version 단위**로 동작해야 한다.
- 액션별 엔드포인트를 두면 UI/권한/감사 연결이 직관적이다.
- 이후 OpenAPI 문서화 및 외부 API 연동이 쉬워진다.

---

## 5. 📨 요청 스키마 정의

공통 요청 바디는 다음 구조를 권장한다.

```json
{
  "comment": "검토 의견 또는 처리 메모",
  "reason": "상태 변경 사유",
  "expected_current_status": "IN_REVIEW"
}
```

### 필드 정의

- `comment`: 선택 입력. 검토/승인/반려 시 남기는 메모
- `reason`: 정책상 필요 시 필수 입력
- `expected_current_status`: 낙관적 락/동시성 검증 보조용 필드

---

## 6. 📤 응답 스키마 정의

성공 응답 예시는 다음과 같다.

```json
{
  "data": {
    "document_id": "doc_123",
    "version_id": "ver_5",
    "previous_status": "IN_REVIEW",
    "current_status": "APPROVED",
    "action": "approve",
    "acted_by": "user_001",
    "acted_at": "2026-04-06T10:30:00Z"
  },
  "meta": {
    "request_id": "req_abc123"
  }
}
```

응답에는 최소한 다음이 포함되어야 한다.

- 이전 상태
- 현재 상태
- 수행된 액션
- 수행자
- 수행 시각

---

## 7. 🛡 서버 처리 흐름 설계

각 Workflow Action API는 아래 순서로 처리한다.

```text
1. 문서/버전 존재 여부 확인
2. 현재 Version 상태 조회
3. expected_current_status 일치 여부 확인 (선택적이지만 권장)
4. 액션 -> target status 매핑
5. 상태 전이 허용 여부 검증 (Task 5-2)
6. 사용자 역할 및 권한 검증 (Task 5-3)
7. 상태 전이 수행
8. WorkflowHistory 기록
9. Audit Log 기록
10. 응답 반환
```

---

## 8. 🧠 Action → Status 매핑 구조 예시

```python
from app.domain.document.enums import DocumentStatus
from app.domain.workflow.enums import WorkflowAction

ACTION_TO_TARGET_STATUS: dict[WorkflowAction, DocumentStatus] = {
    WorkflowAction.SUBMIT_REVIEW: DocumentStatus.IN_REVIEW,
    WorkflowAction.APPROVE: DocumentStatus.APPROVED,
    WorkflowAction.REJECT: DocumentStatus.REJECTED,
    WorkflowAction.PUBLISH: DocumentStatus.PUBLISHED,
    WorkflowAction.ARCHIVE: DocumentStatus.ARCHIVED,
    WorkflowAction.RETURN_TO_DRAFT: DocumentStatus.DRAFT,
}
```

별도 Enum도 정의하는 것을 권장한다.

```python
from enum import Enum

class WorkflowAction(str, Enum):
    SUBMIT_REVIEW = "submit_review"
    APPROVE = "approve"
    REJECT = "reject"
    PUBLISH = "publish"
    ARCHIVE = "archive"
    RETURN_TO_DRAFT = "return_to_draft"
```

---

## 9. 🏗 서비스 레이어 구조 권장안

API 라우터에서 직접 모든 로직을 처리하지 말고,
Workflow 전용 서비스 레이어를 둔다.

예시:

```text
app/
  api/
    v1/
      document_workflow.py
  services/
    workflow_service.py
  domain/
    workflow/
      enums.py
      policies.py
```

### 서비스 레이어 책임

`workflow_service.py`는 다음을 담당한다.

- 현재 상태 조회
- 액션 -> 상태 매핑
- 상태 전이 허용 여부 검증
- 역할 기반 권한 확인
- 상태 변경 트랜잭션 수행
- 이력 / 감사 로그 생성

즉, 라우터는 얇게 유지하고,
비즈니스 로직은 서비스 계층에 집중시킨다.

---

## 10. 🔐 권한 검증 연결 지점

Task 5-3의 Role 기반 정책은 API 내부에서 다음처럼 적용한다.

```text
can_perform_workflow_action(
    user=user,
    current_status=current_status,
    target_status=target_status,
    action=action,
    document=document,
    version=version,
)
```

권장 사항:

- 단순 Role뿐 아니라 향후 ACL/조직/문서 소유자 정책까지 확장 가능해야 한다.
- Author 본인만 submit-review 가능하게 할지 여부는 정책으로 분리할 수 있다.

---

## 11. ⚠️ 동시성 및 무결성 고려

### 11.1 expected_current_status

클라이언트가 마지막으로 인식한 상태를 함께 보내고,
서버의 현재 상태와 다르면 `409 Conflict`를 반환한다.

예:

- 사용자는 `IN_REVIEW`라고 보고 approve 요청
- 그러나 이미 다른 사용자가 reject 처리함
- 서버는 충돌로 간주하고 거부

### 11.2 DB 트랜잭션

다음 작업은 하나의 트랜잭션으로 묶는 것이 바람직하다.

- Version 상태 변경
- WorkflowHistory 생성
- Audit Log 생성

---

## 12. 🚨 오류 처리 원칙

### 12.1 존재하지 않는 문서/버전

- `404 Not Found`

### 12.2 허용되지 않은 상태 전이

- `409 Conflict`

```json
{
  "error": {
    "code": "INVALID_WORKFLOW_TRANSITION",
    "message": "Cannot transition document version from REJECTED to PUBLISHED."
  }
}
```

### 12.3 권한 없음

- `403 Forbidden`

```json
{
  "error": {
    "code": "FORBIDDEN_WORKFLOW_ACTION",
    "message": "User does not have permission to perform this workflow action."
  }
}
```

### 12.4 상태 충돌

- `409 Conflict`

```json
{
  "error": {
    "code": "WORKFLOW_STATE_CONFLICT",
    "message": "The document version status has changed since it was loaded."
  }
}
```

---

## 13. 🧾 감사 / 이력 기록 연계

각 API 성공 시 반드시 다음을 남긴다.

### 13.1 WorkflowHistory

- document_id
- version_id
- previous_status
- current_status
- action
- actor_id
- comment
- reason
- created_at

### 13.2 Audit Log

- event_type: `document.workflow.transition`
- actor_id
- target_resource
- request_id
- result

---

## 14. 🔄 대안 API 설계 검토

대안으로 단일 엔드포인트도 고려할 수 있다.

예:

```http
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/actions
```

```json
{
  "action": "approve",
  "comment": "검토 완료",
  "reason": "정책 충족"
}
```

하지만 Phase 5에서는 다음 이유로 **액션별 엔드포인트 방식**을 우선 권장한다.

- OpenAPI 문서가 명확함
- 권한 정책을 액션별로 설명하기 쉬움
- 감사 이벤트를 액션별로 추적하기 쉬움
- UI 버튼과 API가 1:1 대응됨

단, 내부 서비스 계층은 공통 함수로 재사용 가능하게 설계한다.

---

## 15. 📦 산출물

- Workflow Action API 목록
- 요청/응답 스키마 정의
- 서비스 레이어 구조 권장안
- 검증 흐름 설계
- 오류 처리 정책
- 감사/이력 연계 포인트 정의

---

## 16. 🚀 다음 작업 연결

👉 Task 5-5: 검토 / 승인 / 반려 기능 구현 설계

다음 내용을 구체화한다.

- ReviewAction 모델
- comment / reason 저장 구조
- 검토/승인/반려 데이터 기록 방식
- 워크플로 액션과 데이터 모델 연결

---

## 17. ✅ 완료 기준

- 액션별 API 계약이 명확히 정의되어 있어야 한다.
- 상태 전이 및 권한 검증 흐름이 API 수준에서 연결되어 있어야 한다.
- 서비스 레이어 분리가 반영되어 있어야 한다.
- 이후 구현 단계에서 바로 라우터/서비스 코드를 작성할 수 있어야 한다.