

# 📘 Task 5-10 작업 지시서
## Phase 5 검수 결함 기록 및 사후 수정 내역

---

## 1. 🎯 작업 목적

Phase 5 구현 완료 후 검수 과정에서 발견된 결함을 기록하고,
각 결함에 대한 원인 분석·수정 내역·재발 방지 기준을 명시한다.

검수 일자: 2026-04-07
결함 건수: 6건 (전원 수정 완료)

---

## 2. 📌 결함 목록 요약

| # | 결함 | 심각도 | 근거 스펙 | 상태 |
|---|---|---|---|---|
| DEF-01 | actor_role 필드 누락 | High | Task 5-5 §9 | ✅ 수정 완료 |
| DEF-02 | reject 시 reason 필수 검증 누락 | High | Task 5-5 §12.1 | ✅ 수정 완료 |
| DEF-03 | review_actions 테이블 metadata 컬럼 누락 | Medium | Task 5-5 §4 | ✅ 수정 완료 |
| DEF-04 | IN_REVIEW/APPROVED/PUBLISHED 편집 차단 미구현 | High | Task 5-9 §3,§4 | ✅ 수정 완료 |
| DEF-05 | 버전 응답에 workflow_status 미포함 | Medium | Task 5-8 | ✅ 수정 완료 |
| DEF-06 | Admin override 미추적 | High | Task 5-9 §6 | ✅ 수정 완료 |

---

## 3. 🐛 DEF-01: actor_role 필드 누락

### 결함 내용

ReviewAction, WorkflowHistory, ChangeLog 모델·DB·리포지터리·응답 스키마 전반에
`actor_role` 필드가 누락되어 있었다.

행위 시점의 역할 정보를 불변으로 보존해야 한다는 Task 5-5 §9 요구사항 위반.

### 영향 범위

- `app/models/review_action.py` — actor_role 필드 없음
- `app/models/workflow_history.py` — actor_role 필드 없음
- `app/models/change_log.py` — actor_role 필드 없음
- `app/db/connection.py` — DDL에 actor_role 컬럼 없음
- `app/repositories/workflow_repository.py` — INSERT 시 actor_role 미전달
- `app/schemas/workflow.py` — WorkflowHistoryItem, ReviewActionItem에 actor_role 없음
- `app/api/v1/workflow.py` — _history_item(), _review_item() 헬퍼에서 actor_role 미전달

### 수정 내역

**모델 수정** — 세 모델에 `actor_role: Optional[str]` 필드 추가

**DB DDL 수정** — 신규 테이블 DDL에 `actor_role VARCHAR(100)` 컬럼 추가

**마이그레이션 DDL 추가** — 기존 테이블에 actor_role 컬럼 추가

```sql
ALTER TABLE review_actions ADD COLUMN IF NOT EXISTS actor_role VARCHAR(100);
ALTER TABLE workflow_history ADD COLUMN IF NOT EXISTS actor_role VARCHAR(100);
ALTER TABLE change_logs ADD COLUMN IF NOT EXISTS actor_role VARCHAR(100);
```

**리포지터리 수정** — INSERT 쿼리에 actor_role 파라미터 추가

**스키마 수정**

```python
class WorkflowHistoryItem(BaseModel):
    actor_role: Optional[str] = Field(default=None, description="행위 시점 역할 스냅샷")

class ReviewActionItem(BaseModel):
    actor_role: Optional[str] = Field(default=None, description="행위 시점 역할 스냅샷")
```

**라우터 헬퍼 수정**

```python
def _history_item(h) -> dict:
    return WorkflowHistoryItem(
        ...
        actor_role=h.actor_role,
        ...
    ).model_dump()
```

### 재발 방지 기준

- 행위 기록 테이블(review_actions, workflow_history, change_logs)은 반드시 `actor_id` + `actor_role` 쌍으로 저장해야 한다.
- 역할은 행위 시점 스냅샷이므로 나중에 변경되어도 기록은 불변이어야 한다.

---

## 4. 🐛 DEF-02: reject 시 reason 필수 검증 누락

### 결함 내용

`/workflow/reject` 엔드포인트에서 `reason` 필드를 입력하지 않아도
정상 처리되는 문제. Task 5-5 §12.1에서 반려 사유를 필수로 요구.

### 수정 내역

`app/services/workflow_service.py` perform_action() 메서드 내 정책 검증 단계(0단계) 추가:

```python
# 0. 정책 수준 검증 (reject는 reason 필수)
if action == WorkflowAction.REJECT and not reason:
    raise ApiValidationError(
        message="반려 사유(reason)는 필수입니다.",
        details=[{"field": "reason", "issue": "required for reject action"}],
    )
```

### 재발 방지 기준

- 반려(reject) 액션은 항상 reason 값을 요구한다.
- perform_action() 진입 시점에서 비즈니스 규칙 수준의 필드 검증을 수행한다.
- Pydantic 스키마의 Optional 선언이 비즈니스 필수 여부와 무관함을 주의한다.

---

## 5. 🐛 DEF-03: review_actions 테이블 metadata 컬럼 누락

### 결함 내용

Task 5-5 §4에서 ReviewAction에 `metadata` 확장 필드를 요구했으나,
DDL 및 모델에 metadata 컬럼이 누락되어 있었다.

Admin override 플래그 등 확장 정보를 저장할 수 없었음.

### 수정 내역

**DDL 수정**

```sql
-- review_actions 신규 생성 DDL
metadata JSONB NOT NULL DEFAULT '{}'

-- 마이그레이션 DDL
ALTER TABLE review_actions ADD COLUMN IF NOT EXISTS metadata JSONB NOT NULL DEFAULT '{}';
```

**모델 수정**

```python
@dataclass
class ReviewAction:
    ...
    metadata: dict = field(default_factory=dict)
```

**리포지터리 수정** — INSERT 시 `json.dumps(metadata)` 직렬화 후 저장

**스키마 수정**

```python
class ReviewActionItem(BaseModel):
    metadata: dict = Field(default_factory=dict, description="확장 정보")
```

### 재발 방지 기준

- 행위 기록 테이블에는 미래 확장을 위해 `metadata JSONB` 컬럼을 기본으로 포함한다.
- 신규 테이블 DDL 작성 시 확장 필드(metadata, extra_data 등) 포함 여부를 체크한다.

---

## 6. 🐛 DEF-04: IN_REVIEW/APPROVED/PUBLISHED 편집 차단 미구현

### 결함 내용

Task 5-9 §3(승인 중 수정 제한)·§4(게시 이후 수정 정책)에 따라,
`IN_REVIEW`, `APPROVED`, `PUBLISHED` 상태의 버전은 draft 저장이 불가해야 하지만
차단 로직이 구현되지 않았다.

### 수정 내역

`app/domain/workflow/policies.py`에 편집 허용 상태 정의 추가:

```python
EDITABLE_STATUSES: frozenset[WorkflowStatus] = frozenset({
    WorkflowStatus.DRAFT,
    WorkflowStatus.REJECTED,
})
```

`app/services/draft_service.py` save_draft() 내 편집 차단 로직 추가:

```python
# 편집 차단 검증
current_workflow_status = workflow_repository.get_workflow_status(conn, existing_draft.id)
if current_workflow_status is not None:
    try:
        ws = WorkflowStatus(current_workflow_status)
        if ws not in EDITABLE_STATUSES:
            raise ApiVersionNotEditableError(
                message=f"워크플로 상태 '{current_workflow_status}'인 버전은 편집할 수 없습니다.",
                details={"workflow_status": current_workflow_status},
            )
    except ValueError:
        pass
```

`app/api/errors/exceptions.py`에 ApiVersionNotEditableError 추가:

```python
class ApiVersionNotEditableError(ApiError):
    def __init__(self, message: str = "버전을 편집할 수 없습니다.", details=None):
        super().__init__(
            status_code=409,
            error_code="VERSION_NOT_EDITABLE",
            message=message,
            details=details,
        )
```

### 재발 방지 기준

- 편집 가능 상태는 `EDITABLE_STATUSES` 단일 출처(policies.py)에서 관리한다.
- draft 저장·수정 진입점마다 반드시 workflow_status 차단 검증을 수행한다.
- 비즈니스 상태와 워크플로 상태 검증은 서비스 레이어에서 통합 처리한다.

---

## 7. 🐛 DEF-05: 버전 응답에 workflow_status 미포함

### 결함 내용

Task 5-8(UI 연동)에서 버전 목록·상세 응답에 `workflow_status` 포함을 요구했으나,
실제 응답 스키마 및 서비스 코드에 해당 필드가 반영되지 않았다.

### 영향 범위

- `app/schemas/versions.py` — VersionResponse, VersionDetailResponse, VersionSummaryResponse
- `app/services/draft_service.py` — list_versions(), get_version_detail(), save_draft(), publish(), restore()

### 수정 내역

**스키마 수정** — 세 응답 스키마에 workflow_status 추가:

```python
workflow_status: Optional[str] = Field(
    default=None,
    description="워크플로 상태 (draft/in_review/approved/published/rejected/archived)",
)
```

**서비스 수정** — list_versions() 내 각 버전에 대해 workflow_status 조회:

```python
wf_status = workflow_repository.get_workflow_status(conn, v.id)
items.append(_to_summary_response(v, ..., workflow_status=wf_status))
```

**서비스 수정** — save_draft(), publish(), restore() 응답에 workflow_status 포함

### 재발 방지 기준

- 버전 관련 모든 응답 스키마는 `workflow_status` 필드를 포함한다.
- Phase 5 이후 버전을 반환하는 엔드포인트는 workflow_repository 조회를 필수로 수행한다.
- UI 상태 배지·버튼 분기는 `workflow_status` 값을 기준으로 한다.

---

## 8. 🐛 DEF-06: Admin override 미추적

### 결함 내용

Task 5-9 §6에서 Admin override 시 감사 로그에 override 여부 flag를 필수로 기록해야 하지만,
`is_admin_override` 플래그가 ReviewAction.metadata 및 audit_log에 기록되지 않았다.

### 수정 내역

`app/services/workflow_service.py` perform_action() 내 override 플래그 생성:

```python
is_admin_override = (actor_role is not None and
                     actor_role.lower() == WorkflowRole.ADMIN.value)
extra_metadata = {"is_admin_override": True} if is_admin_override else {}
```

**ReviewAction 생성 시 metadata 전달**:

```python
workflow_repository.create_review_action(
    conn,
    ...
    metadata=extra_metadata,
)
```

**Audit Log 이벤트에 override 정보 포함**:

```python
audit_emitter.emit(conn, AuditEvent(
    ...
    extra={
        "action": action.value,
        "from_status": current_status,
        "to_status": target_status.value,
        "is_admin_override": is_admin_override,
    },
))
```

### 재발 방지 기준

- ADMIN 역할로 수행된 모든 워크플로 액션은 `is_admin_override: true`를 metadata에 기록한다.
- Admin override는 ReviewAction.metadata와 audit_log.extra 양쪽에 모두 기록한다.
- 향후 UI에서 override 행위를 별도 표시할 수 있도록 필드 일관성을 유지한다.

---

## 9. 📦 수정 완료 파일 목록

| 파일 | 수정 내용 |
|---|---|
| `app/models/review_action.py` | actor_role, metadata 필드 추가 |
| `app/models/workflow_history.py` | actor_role 필드 추가 |
| `app/models/change_log.py` | actor_role 필드 추가 |
| `app/db/connection.py` | actor_role/metadata 컬럼 DDL + 마이그레이션 추가 |
| `app/repositories/workflow_repository.py` | actor_role, metadata INSERT 반영 |
| `app/services/workflow_service.py` | reject reason 검증, is_admin_override 플래그, metadata 전달 |
| `app/services/draft_service.py` | 편집 차단 로직, workflow_status 조회 및 응답 포함 |
| `app/schemas/workflow.py` | WorkflowHistoryItem, ReviewActionItem에 actor_role, metadata 추가 |
| `app/schemas/versions.py` | VersionResponse, VersionDetailResponse, VersionSummaryResponse에 workflow_status 추가 |
| `app/domain/workflow/policies.py` | EDITABLE_STATUSES 추가, APPROVER archive 권한 추가 |
| `app/api/v1/workflow.py` | _history_item(), _review_item() 헬퍼에 actor_role, metadata 반영 |
| `app/api/errors/exceptions.py` | ApiVersionNotEditableError 추가 |

---

## 10. ✅ 완료 기준

- [ ] actor_role이 ReviewAction, WorkflowHistory, ChangeLog에 정상 저장된다.
- [ ] reject 시 reason 없으면 400 validation_error를 반환한다.
- [ ] review_actions 테이블에 metadata 컬럼이 존재하며 is_admin_override 등 확장 정보를 저장할 수 있다.
- [ ] IN_REVIEW/APPROVED/PUBLISHED 상태 버전에 draft 저장 시 409 VERSION_NOT_EDITABLE을 반환한다.
- [ ] 버전 목록·상세·draft save·publish·restore 응답에 workflow_status 필드가 포함된다.
- [ ] Admin 역할로 수행된 액션은 ReviewAction.metadata와 audit_log.extra에 `is_admin_override: true`가 기록된다.
