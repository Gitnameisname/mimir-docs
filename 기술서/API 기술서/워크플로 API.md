# 워크플로 API

Base URL: `/api/v1/documents/{document_id}/versions/{version_id}/workflow`

---

## 1. 워크플로 상태 전이 요약

```
draft → in_review → approved → published
                ↘             ↗
               rejected (→ draft)
```

전이 상세는 [문서 상태와 워크플로](../2.%20도메인%20모델/문서%20상태와%20워크플로.md) 참조.

---

## 2. 검토 제출 (Submit)

```
POST /api/v1/documents/{did}/versions/{vid}/workflow/submit
Authorization: Bearer <token>
Content-Type: application/json

{
  "comment": "검토 부탁드립니다."
}
```

- `draft` → `in_review` 전이
- 필요 권한: AUTHOR 이상
- `comment` 선택 입력

**응답:**
```json
{
  "status": "success",
  "data": {
    "version_id": "660e8400-...",
    "workflow_status": "in_review",
    "action": "submit",
    "actor_id": "user-001",
    "comment": "검토 부탁드립니다.",
    "created_at": "2026-04-10T10:00:00+00:00"
  }
}
```

---

## 3. 승인 (Approve)

```
POST /api/v1/documents/{did}/versions/{vid}/workflow/approve
Authorization: Bearer <token>
Content-Type: application/json

{
  "comment": "내용 확인 완료. 승인합니다."
}
```

- `in_review` → `approved` 전이
- 필요 권한: APPROVER 이상

---

## 4. 반려 (Reject)

```
POST /api/v1/documents/{did}/versions/{vid}/workflow/reject
Authorization: Bearer <token>
Content-Type: application/json

{
  "reason": "3.2절 내용 보완 필요",
  "comment": "구체적인 절차 명시 후 재제출 바랍니다."
}
```

- `in_review` → `draft` (rejected 경유) 전이
- 필요 권한: REVIEWER 이상
- `reason` 필수 입력

---

## 5. 게시 (Publish)

```
POST /api/v1/documents/{did}/versions/{vid}/workflow/publish
Authorization: Bearer <token>
Content-Type: application/json

{
  "comment": "공식 게시합니다."
}
```

- `approved` → `published` 전이
- 필요 권한: APPROVER 이상
- 이전 Published 버전이 있으면 자동으로 `archived` 처리
- `Document.status` → `published` 자동 업데이트

---

## 6. 보관 (Archive)

```
POST /api/v1/documents/{did}/versions/{vid}/workflow/archive
Authorization: Bearer <token>
Content-Type: application/json

{
  "reason": "신규 버전으로 교체"
}
```

- `published` → `archived` 전이
- 필요 권한: ORG_ADMIN 이상

---

## 7. 검토 취소 (Withdraw)

```
POST /api/v1/documents/{did}/versions/{vid}/workflow/withdraw
Authorization: Bearer <token>
Content-Type: application/json

{
  "reason": "수정 사항 발생"
}
```

- `in_review` → `draft` 전이 (제출자 본인 또는 ORG_ADMIN)

---

## 8. 워크플로 이력 조회

```
GET /api/v1/documents/{did}/versions/{vid}/workflow/history
Authorization: Bearer <token>
```

**응답:**
```json
{
  "status": "success",
  "data": [
    {
      "id": "...",
      "from_status": "draft",
      "to_status": "in_review",
      "action": "submit",
      "actor_id": "user-001",
      "actor_role": "AUTHOR",
      "comment": "검토 부탁드립니다.",
      "created_at": "2026-04-10T10:00:00+00:00"
    },
    {
      "id": "...",
      "from_status": "in_review",
      "to_status": "approved",
      "action": "approve",
      "actor_id": "user-002",
      "actor_role": "APPROVER",
      "comment": "승인합니다.",
      "created_at": "2026-04-10T14:30:00+00:00"
    }
  ]
}
```

---

## 9. 검토 대기 문서 목록 조회

```
GET /api/v1/documents?filter[workflow_status]=in_review
Authorization: Bearer <token>
```

REVIEWER/APPROVER 역할 사용자의 검토 대기 문서 목록.

---

## 10. 오류 응답

| 상황 | 코드 | 설명 |
|------|------|------|
| 잘못된 상태 전이 | 409 CONFLICT | 현재 상태에서 해당 액션 불가 |
| 권한 부족 | 403 PERMISSION_DENIED | 해당 역할 미충족 |
| 버전 없음 | 404 NOT_FOUND | 버전 ID 불일치 |

```json
{
  "status": "error",
  "error": {
    "code": "CONFLICT",
    "message": "Cannot submit: version is not in draft status",
    "details": {
      "current_status": "in_review",
      "required_status": "draft"
    }
  }
}
```
