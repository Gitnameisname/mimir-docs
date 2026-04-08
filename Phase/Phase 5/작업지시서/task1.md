

# 📘 Task 5-1 작업 지시서
## Workflow 상태 모델 정의

---

## 1. 🎯 작업 목적

문서의 생명주기를 명확히 표현할 수 있는 **표준 상태 모델(Document Workflow Status Model)**을 정의한다.

이 상태 모델은 이후:
- 상태 전이(State Machine)
- 권한 제어(RBAC)
- 감사 로그(Audit)
- UI 표시

의 기준이 된다.

---

## 2. 📌 설계 범위

본 작업에서는 다음을 정의한다:

- Document 상태 Enum
- 상태별 의미 정의
- 상태 간 관계 (전이 규칙은 Task 5-2에서 정의)
- 상태별 정책 (수정 가능 여부, 공개 여부 등)

---

## 3. 🧩 상태 정의

다음 상태를 기본으로 한다:

```
DRAFT
IN_REVIEW
APPROVED
PUBLISHED
REJECTED
ARCHIVED (optional)
```

---

## 4. 🧠 상태 의미 정의

### 4.1 DRAFT

- 작성 중 상태
- 자유롭게 수정 가능
- 외부 사용자에게 노출되지 않음
- Author가 관리

---

### 4.2 IN_REVIEW

- 검토 요청 상태
- Reviewer가 검토 수행
- 작성자는 직접 수정 제한 (정책 선택 가능)
- 코멘트 기반 피드백 가능

---

### 4.3 APPROVED

- 승인 완료 상태
- Publish 대기 상태
- 내용 변경 금지 (immutable 권장)

---

### 4.4 PUBLISHED

- 공식 배포 상태
- 외부 조회 가능
- 수정 불가
- 변경 시 반드시 새로운 Version 생성 필요

---

### 4.5 REJECTED

- 검토/승인 과정에서 반려된 상태
- 수정 후 다시 Draft 또는 Review로 이동

---

### 4.6 ARCHIVED (Optional)

- 더 이상 사용되지 않는 문서 상태
- 조회는 가능하지만 수정/배포 불가

---

## 5. 🏗 데이터 모델 정의

### 5.1 Enum 정의

```python
from enum import Enum

class DocumentStatus(str, Enum):
    DRAFT = "DRAFT"
    IN_REVIEW = "IN_REVIEW"
    APPROVED = "APPROVED"
    PUBLISHED = "PUBLISHED"
    REJECTED = "REJECTED"
    ARCHIVED = "ARCHIVED"
```

---

### 5.2 Document / Version 연결 정책

- 상태는 **Version 단위로 관리**
- Document는 여러 Version을 가질 수 있음
- 각 Version은 독립적인 상태를 가짐

---

## 6. ⚠️ 설계 고려 사항

### 6.1 상태는 Version 기준으로 관리

❌ Document 단위 상태 관리 금지  
✔ Version 단위 상태 관리 필수

---

### 6.2 Published 불변성 (Immutable)

- Published 상태의 Version은 수정 금지
- 수정 시:
  → 새로운 Version 생성 → Draft 상태 시작

---

### 6.3 Rejected 상태의 처리

- 종료 상태가 아님
- 반드시 재작업 가능해야 함

---

### 6.4 상태 확장 가능성

향후 확장 고려:

```
IN_APPROVAL
SCHEDULED
DEPRECATED
```

---

## 7. 📦 산출물

- DocumentStatus Enum 정의
- 상태 의미 정의 문서 (본 문서)
- Version 기준 상태 관리 정책

---

## 8. 🚀 다음 작업 연결

다음 Task에서 수행:

👉 Task 5-2: 상태 전이(State Machine) 설계

- 상태 간 이동 규칙 정의
- 불법 전이 차단 정책 수립

---

## 9. ✅ 완료 기준

- 모든 상태가 명확히 정의되어야 한다
- 상태 간 의미 충돌이 없어야 한다
- Version 단위 상태 관리 정책이 확정되어야 한다
- 이후 Task에서 바로 사용할 수 있어야 한다