

# 📘 Task 5-8 작업 지시서
## Workflow UI 연동 포인트 정의

---

## 1. 🎯 작업 목적

Phase 5에서 설계한 문서 워크플로 구조를 실제 사용자 화면과 운영자 화면에 연결하기 위해,
UI가 어떤 정보를 어떤 방식으로 표시하고 어떤 액션을 호출해야 하는지 **연동 포인트**를 정의한다.

이 작업의 목적은 다음과 같다.

- 문서 상태를 UI에서 명확히 인지할 수 있게 한다.
- 사용자 역할에 따라 가능한 액션만 노출되도록 한다.
- 검토/승인/반려/게시 흐름을 화면에서 자연스럽게 수행할 수 있게 한다.
- 상태 이력, 검토 의견, 변경 사유를 UI에서 추적 가능하게 한다.

---

## 2. 📌 설계 범위

본 작업에서는 다음을 정의한다.

- User UI 연동 포인트
- Admin / Operator UI 연동 포인트
- 상태 표시 방식
- 액션 버튼 노출 정책
- 이력 / 타임라인 / 코멘트 표시 정책
- API와 UI의 연결 구조
- 오류 및 충돌 상황의 UI 처리 원칙

주의:
- 본 Task는 **시각 디자인 작업**이 아니라,
  **UI 요구사항과 API 연결 구조를 정리하는 설계 작업**이다.

---

## 3. 🧩 주요 UI 대상 화면

Phase 5 기준으로 주요 화면은 다음과 같이 본다.

```text
1. 문서 상세 화면 (User UI)
2. 문서 편집 화면 (User UI)
3. 검토/승인 대기 목록 화면 (Operator UI)
4. 문서 워크플로 타임라인 / 이력 화면 (Operator UI)
5. 관리자 감사/운영 상세 화면 (Admin UI)
```

---

## 4. 🏷 상태 표시 UI 정의

문서 또는 문서 버전의 현재 상태는 항상 명확히 표시되어야 한다.

권장 표시 대상:

- 문서 상세 화면 상단
- 편집 화면 상단
- 목록 화면의 각 row/card
- 관리자 상세 화면 상단

권장 상태 배지 예시:

```text
[DRAFT]
[IN REVIEW]
[APPROVED]
[PUBLISHED]
[REJECTED]
[ARCHIVED]
```

표시 원칙:

- 텍스트만이 아니라 시각적으로 구분 가능한 badge 형태 권장
- 상태명은 내부 enum 그대로 노출하지 말고 사용자 친화적 라벨 사용 가능

예:

| 내부 상태 | UI 라벨 예시 |
|---|---|
| DRAFT | Draft |
| IN_REVIEW | In Review |
| APPROVED | Approved |
| PUBLISHED | Published |
| REJECTED | Rejected |
| ARCHIVED | Archived |

---

## 5. 🔘 액션 버튼 노출 정책

UI는 서버 권한 검증을 대체할 수 없지만,
사용자 경험을 위해 **가능한 액션만 우선적으로 노출**하는 것이 바람직하다.

예시:

### 5.1 AUTHOR

- DRAFT 상태에서:
  - `검토 요청` 버튼 표시
- REJECTED 상태에서:
  - `초안으로 복귀` 또는 `재작업 시작` 버튼 표시

### 5.2 REVIEWER

- IN_REVIEW 상태에서:
  - `반려` 버튼 표시
  - `의견 남기기` 버튼 표시 (선택)

### 5.3 APPROVER

- IN_REVIEW 상태에서:
  - `승인` 버튼 표시
- APPROVED 상태에서:
  - `게시` 버튼 표시

### 5.4 ADMIN

- 정책상 허용되는 모든 버튼 표시 가능
- 단, override 액션은 별도 경고/사유 입력 UI 필요

---

## 6. 🧠 버튼 클릭 시 필요한 입력 UI

일부 액션은 단순 클릭이 아니라 추가 입력이 필요하다.

### 6.1 검토 요청

권장 입력:

- comment (선택)
- reason (선택 또는 정책별 필수)

### 6.2 승인

권장 입력:

- comment (선택)
- reason (권장)

### 6.3 반려

권장 입력:

- comment (권장)
- reason (필수)

### 6.4 Admin override

필수 입력 권장:

- reason (필수)
- comment (선택)
- 확인 다이얼로그

---

## 7. 🌐 API 연동 구조

UI는 Task 5-4에서 정의한 Workflow Action API와 연결된다.

예:

```http
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/submit-review
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/approve
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/reject
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/publish
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/archive
POST /api/v1/documents/{document_id}/versions/{version_id}/workflow/return-to-draft
```

UI 요청 예시:

```json
{
  "comment": "검토 부탁드립니다.",
  "reason": "초안 작성 완료",
  "expected_current_status": "DRAFT"
}
```

---

## 8. 📜 이력 / 타임라인 표시 정책

### 8.1 문서 상세 화면

최소한 다음을 표시할 수 있어야 한다.

- 현재 상태
- 최근 WorkflowHistory 3~5건
- 최근 ReviewAction 3~5건
- 최근 ChangeLog 요약

### 8.2 Operator UI 타임라인

권장 타임라인 항목:

```text
- 초안 생성
- 검토 요청
- 반려
- 재작성
- 승인
- 게시
- 보관
```

각 항목에 포함할 정보:

- action
- from_status / to_status
- actor
- created_at
- comment/reason 요약

---

## 9. 🧾 검토 의견 / 반려 사유 표시 정책

ReviewAction은 별도 섹션으로 표시하는 것이 좋다.

예:

```text
[검토 의견]
- 2026-04-06 / Reviewer A / "근거 문장을 보완해 주세요"
- 2026-04-07 / Approver B / "정책 기준 충족"
```

반려 사유는 특별히 강조해서 보여야 한다.

권장 이유:

- Author가 다음 액션을 이해해야 함
- 반려 후 재작업 효율 향상

---

## 10. 📝 변경 사유(ChangeLog) 표시 정책

문서 편집/상세 화면에서 다음 중 하나를 제공하는 것을 권장한다.

### 옵션 A: 최근 변경 요약

```text
최근 변경 사유: 정책 문구 최신화
```

### 옵션 B: 별도 변경 이력 탭

- 변경 시각
- 변경자
- reason
- diff summary

MVP에서는 옵션 A 또는 간단 목록부터 시작 가능하다.

---

## 11. 📋 대기 목록 화면 요구사항

Operator UI에서는 다음 목록이 필요하다.

### 11.1 검토 대기 목록

필드 예시:

- document title
- version
- current status
- requested_by
- requested_at
- last updated at

### 11.2 승인 대기 목록

필드 예시:

- document title
- version
- approved candidate status
- reviewer/author info
- latest comment summary

---

## 12. ⚠️ 오류 및 충돌 UI 처리 원칙

### 12.1 권한 없음

- 버튼 비노출 우선
- 서버에서 403 반환 시 명확한 오류 메시지 표시

예:

```text
이 워크플로 액션을 수행할 권한이 없습니다.
```

### 12.2 상태 충돌

예:
- 사용자가 승인 버튼을 눌렀으나,
- 다른 사용자가 먼저 반려 처리함

UI 처리 원칙:

- `409 Conflict` 수신 시 현재 상태 재조회
- 사용자에게 상태가 변경되었음을 안내
- 화면 새로고침 또는 상태 동기화 수행

예시 메시지:

```text
문서 상태가 다른 사용자에 의해 변경되었습니다. 최신 상태를 다시 불러옵니다.
```

### 12.3 필수 사유 누락

- reject 시 reason 미입력
- UI에서 사전 검증
- 서버 검증 실패 메시지도 표시 가능해야 함

---

## 13. 🔐 역할 기반 UI 분리 원칙

중요 원칙:

> UI는 역할에 따라 다르게 보일 수 있지만,
> 최종 권한 검증은 반드시 서버가 수행한다.

즉:

- UI는 편의상 버튼을 숨길 수 있음
- 서버는 항상 상태/권한 검증 수행
- UI 로직만으로 보안을 구현하지 않음

---

## 14. 📊 관리자/운영자 화면 확장 포인트

향후 다음 기능으로 확장 가능해야 한다.

- 승인/반려 반복 문서 식별
- 병목 단계 분석
- 특정 역할별 처리 시간 통계
- 문서 유형별 워크플로 성능 분석

이를 위해 UI는 나중에 다음 API를 추가할 수 있도록 구조를 열어둔다.

```text
GET /workflow/history
GET /review-actions
GET /change-logs
GET /workflow/queues
```

---

## 15. 🧭 화면별 요약 매핑

| 화면 | 표시 정보 | 주요 액션 |
|---|---|---|
| 문서 상세 | 상태, 최근 이력, 최근 의견 | 게시/승인 여부 확인 |
| 문서 편집 | 현재 상태, 최근 반려 사유 | 재작업, 검토 요청 |
| 검토 대기 목록 | 제목, 요청 시각, 상태 | 문서 열기, 반려/의견 |
| 승인 대기 목록 | 제목, 상태, 최신 코멘트 | 승인, 게시 |
| 관리자 상세 | 전체 타임라인, 변경 로그, 감사 정보 | override, 감사 확인 |

---

## 16. 📦 산출물

- 상태 표시 UI 정책
- 액션 버튼 노출 정책
- 이력/코멘트 표시 정책
- 대기 목록 요구사항
- API ↔ UI 연동 포인트 정의
- 오류/충돌 처리 원칙

---

## 17. 🚀 다음 작업 연결

👉 Task 5-9: 예외 및 운영 정책 정의

다음 내용을 정리한다.

- 승인 중 수정 방지
- 게시 후 수정 정책
- 동시 승인 충돌 대응
- 운영 예외 상황 처리 원칙

---

## 18. ✅ 완료 기준

- UI가 어떤 상태와 액션을 보여야 하는지 명확해야 한다.
- API 연동 구조가 바로 구현 가능한 수준이어야 한다.
- 이력/코멘트/변경 사유가 화면에서 추적 가능해야 한다.
- User UI / Operator UI / Admin UI가 역할에 맞게 구분되어야 한다.
