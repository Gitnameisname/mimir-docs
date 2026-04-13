

# 📘 Task 5-5 작업 지시서
## 검토 / 승인 / 반려 기능 구현 설계

---

## 1. 🎯 작업 목적

Task 5-4에서 정의한 Workflow Action API를 실제 운영 가능한 수준으로 만들기 위해,
검토(review), 승인(approve), 반려(reject) 행위에 대한 **데이터 구조와 처리 로직**을 설계한다.

이 작업의 목적은 다음과 같다.

- 검토/승인/반려 행위를 단순 상태 변경이 아니라 **의미 있는 운영 기록**으로 남긴다.
- comment / reason / actor / timestamp를 구조적으로 저장한다.
- 이후 이력 조회, 감사 추적, UI 타임라인, 통계 기능의 기반을 만든다.

---

## 2. 📌 설계 범위

본 작업에서는 다음을 정의한다.

- ReviewAction 개념 정의
- 검토/승인/반려 데이터 모델 구조
- comment / reason 저장 정책
- 상태 전이와 ReviewAction 기록의 관계
- 서비스 레이어 처리 흐름
- UI 및 이력 연계 포인트

주의:
- 본 Task는 Workflow 전체 이력 테이블(Task 5-7)과 별도로,
  **행위(Action) 자체의 상세 기록**을 다루는 데 초점을 둔다.

---

## 3. 🧩 ReviewAction 개념 정의

문서 버전에 대해 사용자가 수행하는 검토 관련 행위를 `ReviewAction`으로 정의한다.

포함되는 행위:

```text
submit_review
approve
reject
comment_only   (선택적)
```

설계 원칙:

- 상태 전이와 별개로 “누가 무엇을 말하며 어떤 판단을 내렸는가”를 기록한다.
- 특히 `approve` 와 `reject` 는 반드시 기록되어야 한다.
- `submit_review`도 요청 이력으로 남기는 것을 권장한다.

---

## 4. 🏗 데이터 모델 설계

권장 모델 예시는 다음과 같다.

```text
ReviewAction
- id
- document_id
- version_id
- action_type
- actor_id
- actor_role
- comment
- reason
- created_at
- metadata (optional)
```

### 필드 설명

- `id`: ReviewAction 고유 식별자
- `document_id`: 대상 문서 ID
- `version_id`: 대상 문서 버전 ID
- `action_type`: submit_review / approve / reject / comment_only
- `actor_id`: 수행 사용자 ID
- `actor_role`: 수행 당시 역할 스냅샷
- `comment`: 검토 코멘트
- `reason`: 승인/반려/변경의 근거 사유
- `created_at`: 수행 시각
- `metadata`: 확장 정보(JSON), 예: 조직 정보, 정책 태그, 첨부 여부

---

## 5. 🧠 action_type 정의

권장 Enum:

```python
from enum import Enum

class ReviewActionType(str, Enum):
    SUBMIT_REVIEW = "submit_review"
    APPROVE = "approve"
    REJECT = "reject"
    COMMENT_ONLY = "comment_only"
```

설계 원칙:

- WorkflowAction과 유사하지만 완전히 동일할 필요는 없다.
- ReviewAction은 **운영 기록 관점**, WorkflowAction은 **상태 제어 관점**으로 분리해도 된다.

---

## 6. 📝 comment / reason 저장 정책

### 6.1 comment

- 검토 의견, 승인 메모, 반려 코멘트
- 사람이 읽기 위한 설명 중심
- 선택 입력 가능

예:

```text
- 표현이 불명확하므로 수정 필요
- 정책 문구 검토 완료
- 게시 전 제목 정렬 권장
```

### 6.2 reason

- 상태 변경의 근거
- 정책적/운영상 의미를 더 강하게 가짐
- 특정 액션에서는 필수화 권장

권장 정책:

| action_type | comment | reason |
|---|---|---|
| submit_review | optional | optional |
| approve | optional | recommended |
| reject | recommended | required |
| comment_only | required or recommended | optional |

---

## 7. 🔁 상태 전이와 ReviewAction의 관계

중요 원칙:

> 상태 전이가 발생하면, 필요한 경우 해당 상태 전이에 대응하는 ReviewAction도 함께 남긴다.

예:

- `DRAFT -> IN_REVIEW`
  → `ReviewActionType.SUBMIT_REVIEW`
- `IN_REVIEW -> APPROVED`
  → `ReviewActionType.APPROVE`
- `IN_REVIEW -> REJECTED`
  → `ReviewActionType.REJECT`

즉:

- 상태 전이는 시스템적 결과
- ReviewAction은 사람의 행위와 판단 기록

둘은 연결되지만 같은 개념은 아니다.

---

## 8. 🛡 서비스 레이어 처리 흐름

`workflow_service.py` 또는 `review_service.py`에서 아래 흐름을 따른다.

```text
1. 요청된 workflow action 확인
2. 현재 상태 확인
3. 상태 전이 허용 여부 검증
4. 역할/권한 검증
5. 상태 변경 수행
6. ReviewAction 생성
7. WorkflowHistory 생성
8. Audit Log 생성
9. 응답 반환
```

권장 사항:

- 상태 변경과 ReviewAction 기록은 하나의 트랜잭션으로 처리한다.
- 상태는 바뀌었는데 ReviewAction이 누락되는 상황을 방지해야 한다.

---

## 9. 🔐 actor_role 스냅샷 저장 이유

`actor_role`은 조회 시점이 아니라 **행위 시점의 역할**을 저장하는 것이 중요하다.

이유:

- 나중에 사용자 역할이 바뀌어도 당시 승인 권한이 있었는지 추적 가능
- 감사/Audit 시 운영 근거 확보 가능

예:

```text
현재는 REVIEWER가 아니더라도,
당시 APPROVER 역할로 승인했음을 기록해야 한다.
```

---

## 10. ⚠️ comment_only 액션에 대한 정책

선택적으로,
상태를 바꾸지 않고 검토 의견만 남기는 액션도 지원할 수 있다.

예:

```text
IN_REVIEW 상태에서 Reviewer가 중간 피드백만 남김
```

이 경우:

- Workflow 상태는 그대로 유지
- ReviewAction만 생성
- WorkflowHistory는 생성하지 않음 (정책 선택 가능)

권장:

- Phase 5 MVP에서는 optional로 두고,
- 구조만 열어두는 것이 적절하다.

---

## 11. 📊 조회/표시 연계 포인트

ReviewAction은 이후 다음 UI와 연결된다.

### 11.1 문서 상세 화면

- 승인/반려 이력 표시
- 최신 검토 의견 표시

### 11.2 워크플로 타임라인

- `submit_review`
- `approve`
- `reject`
- `comment_only`

를 시간순으로 시각화

### 11.3 관리자 감사 화면

- 누가 어떤 근거로 승인/반려했는지 조회
- 반복 반려 문서 식별
- 승인 병목 구간 분석

---

## 12. 🚨 검증 및 오류 정책

### 12.1 reject 시 reason 필수

예:

```json
{
  "error": {
    "code": "REASON_REQUIRED",
    "message": "A rejection reason is required for reject action."
  }
}
```

### 12.2 approve / reject / submit_review 외 action_type 금지

- 정의되지 않은 ReviewActionType은 허용하지 않는다.

### 12.3 actor_id 누락 금지

- ReviewAction은 반드시 사용자 기반 행위여야 한다.
- 시스템 자동 액션이 있다면 별도 system actor 정책 필요

---

## 13. 🔄 WorkflowHistory와의 차이 정리

두 기록 구조는 다음처럼 구분한다.

### ReviewAction

- 사람이 수행한 검토 관련 행위 기록
- comment / reason 포함
- 운영 판단 중심

### WorkflowHistory

- 상태 변화 자체의 이력
- from_status / to_status 포함
- 시스템 상태 추적 중심

권장 정책:

- approve / reject / submit_review 시 둘 다 남긴다.
- comment_only 시 ReviewAction만 남긴다.

---

## 14. 🧾 예시 데이터

```json
{
  "id": "ra_001",
  "document_id": "doc_123",
  "version_id": "ver_5",
  "action_type": "reject",
  "actor_id": "user_007",
  "actor_role": "REVIEWER",
  "comment": "근거 문장이 부족합니다.",
  "reason": "정책 문구 검토 기준 미충족",
  "created_at": "2026-04-06T11:10:00Z",
  "metadata": {
    "team": "compliance"
  }
}
```

---

## 15. 📦 산출물

- ReviewAction 개념 정의
- ReviewActionType Enum 설계
- 데이터 모델 초안
- comment / reason 저장 정책
- 상태 전이와의 연결 원칙
- 서비스 레이어 처리 흐름

---

## 16. 🚀 다음 작업 연결

👉 Task 5-6: 변경 사유 기록 구조 설계

다음 내용을 정의한다.

- 일반 수정(change)와 workflow action의 사유 기록 구분
- ChangeLog 구조
- 편집 사유와 승인/반려 사유의 관계 정리

---

## 17. ✅ 완료 기준

- approve / reject / submit_review 행위가 구조적으로 기록 가능해야 한다.
- comment / reason 저장 정책이 명확해야 한다.
- 상태 전이와 ReviewAction의 관계가 분리되면서도 연결되어 있어야 한다.
- 이후 이력 조회/UI 타임라인/감사 기능으로 확장 가능해야 한다.