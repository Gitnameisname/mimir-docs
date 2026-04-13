

# 📘 Task 5-7 작업 지시서
## Workflow 이력 관리 시스템 (WorkflowHistory) 설계

---

## 1. 🎯 작업 목적

문서의 상태 전이(State Transition)를 **시간 순으로 추적 가능한 형태로 기록**하는
WorkflowHistory 시스템을 설계한다.

이 작업의 목적은 다음과 같다.

- 문서가 어떤 과정을 거쳐 게시되었는지 **전체 흐름 추적**
- 승인/반려/재작업 이력의 **감사 가능성 확보**
- 운영 프로세스의 병목 및 패턴 분석 기반 제공

---

## 2. 📌 설계 범위

본 작업에서는 다음을 정의한다.

- WorkflowHistory 개념 정의
- 데이터 모델 구조
- 상태 전이 기록 정책
- ReviewAction / ChangeLog와의 관계
- 조회 API 설계
- 서비스 처리 흐름

---

## 3. 🧩 WorkflowHistory 개념 정의

WorkflowHistory는 **문서 Version의 상태가 변경된 모든 이벤트**를 기록한다.

포함 대상:

```text
- DRAFT → IN_REVIEW
- IN_REVIEW → APPROVED
- IN_REVIEW → REJECTED
- APPROVED → PUBLISHED
- PUBLISHED → ARCHIVED
```

제외 대상:

```text
- 단순 코멘트 (→ ReviewAction)
- 내용 변경 (→ ChangeLog)
```

---

## 4. 🏗 데이터 모델 설계

권장 모델:

```text
WorkflowHistory
- id
- document_id
- version_id
- from_status
- to_status
- action
- actor_id
- actor_role
- comment (optional)
- reason (optional)
- created_at
- metadata (optional)
```

---

## 5. 🧠 필드 설명

- `from_status`: 이전 상태
- `to_status`: 변경된 상태
- `action`: 수행된 WorkflowAction
- `actor_role`: 당시 역할 스냅샷
- `comment`: 사용자 코멘트 (선택)
- `reason`: 상태 변경 사유

---

## 6. 🔁 상태 전이 기록 정책

핵심 원칙:

> 상태가 바뀌는 모든 순간에 반드시 WorkflowHistory를 남긴다.

예:

| 이전 상태 | 액션 | 이후 상태 |
|---|---|---|
| DRAFT | submit_review | IN_REVIEW |
| IN_REVIEW | approve | APPROVED |
| IN_REVIEW | reject | REJECTED |

---

## 7. 🔄 ReviewAction / ChangeLog와의 관계

| 구조 | 역할 |
|---|---|
| WorkflowHistory | 상태 변화 기록 |
| ReviewAction | 사람의 판단/코멘트 |
| ChangeLog | 내용 변경 |

연계 규칙:

- 상태 전이 시:
  - WorkflowHistory 생성 (필수)
  - ReviewAction 생성 (필요 시)

---

## 8. 🛡 서비스 처리 흐름

```text
1. Workflow Action 요청
2. 상태 전이 검증
3. 권한 검증
4. 상태 변경
5. WorkflowHistory 생성
6. (선택) ReviewAction 생성
7. Audit Log 기록
8. 응답 반환
```

---

## 9. 🌐 조회 API 설계

### 9.1 전체 이력 조회

```http
GET /api/v1/documents/{document_id}/versions/{version_id}/workflow/history
```

응답 예시:

```json
{
  "data": [
    {
      "from_status": "DRAFT",
      "to_status": "IN_REVIEW",
      "action": "submit_review",
      "actor_id": "user_001",
      "actor_role": "AUTHOR",
      "created_at": "2026-04-06T10:00:00Z"
    }
  ]
}
```

---

## 10. 📊 UI 활용

### 10.1 타임라인 뷰

```text
[작성] → [검토 요청] → [반려] → [재작성] → [승인] → [게시]
```

### 10.2 관리자 감사 화면

- 상태 전이 흐름 확인
- 특정 사용자 행동 분석

---

## 11. ⚠️ 동시성 고려

- 상태 변경 시 WorkflowHistory는 반드시 동일 트랜잭션으로 기록
- 상태 변경 성공 후 기록 실패 상황 방지

---

## 12. 🚨 오류 정책

- 상태 변경 실패 시 WorkflowHistory 생성 금지
- 중복 기록 방지

---

## 13. 🧾 예시 데이터

```json
{
  "id": "wh_001",
  "document_id": "doc_123",
  "version_id": "ver_5",
  "from_status": "IN_REVIEW",
  "to_status": "APPROVED",
  "action": "approve",
  "actor_id": "user_010",
  "actor_role": "APPROVER",
  "created_at": "2026-04-06T12:30:00Z"
}
```

---

## 14. 📦 산출물

- WorkflowHistory 모델
- 상태 전이 기록 정책
- 조회 API 설계
- 서비스 처리 흐름

---

## 15. 🚀 다음 작업 연결

👉 Task 5-8: UI 연동 포인트 정의

---

## 16. ✅ 완료 기준

- 모든 상태 전이가 추적 가능해야 한다.
- ReviewAction / ChangeLog와 명확히 구분되어야 한다.
- UI 및 감사 기능으로 확장 가능해야 한다.
