# Task 7-11: PH6-CARRY-001 Batch 승인/거절 롤백 UI

> **이월 출처**: Phase 6 FG6.2 검수보고서 §미결 사항 2번  
> **이월 경로**: Phase 6 미결 (중간 우선순위) → Phase 7 처리

## 1. 작업 목적

Phase 6에서 구현된 `AdminProposalsPage`의 일괄 승인/거절(Batch approve/reject) 기능에는 되돌리기 수단이 없었다. 실수로 대량 처리 후 복구할 방법이 없다는 운영 리스크가 존재한다. 30초 내 되돌리기(Undo) UI를 추가하여 운영 사고를 방지한다.

## 2. 작업 범위

### 포함 범위

**프론트엔드**
1. `AdminProposalsPage.tsx` (`BatchToolbar`) 수정
   - 일괄 처리 직후 30초 카운트다운 타이머와 함께 "되돌리기" 버튼 표시
   - 30초 경과 후 자동으로 버튼 사라짐
   - 되돌리기 클릭 시 `batchRollbackMut.mutate()` 호출

2. 새로운 상태 관리
   - `lastBatchOp: { type: "approve"|"reject"; ids: string[]; completedAt: number } | null`
   - `undoCountdown: number` (30 → 0 카운트다운)

**백엔드**
3. 새 API 엔드포인트
   - `POST /admin/proposals/batch-rollback`
   - 요청: `{ proposal_ids: string[]; original_action: "approve" | "reject" }`
   - 동작: 해당 제안들을 `status: "pending"` 상태로 복원
   - 조건: 개별 승인/거절로 이미 처리된 제안은 롤백 불가 (별도 상태 체크)
   - 감사 로그: `action=batch_rollback`, `actor_type=user|agent`

4. 서비스 계층
   - `ProposalService.batch_rollback(proposal_ids, actor_id, actor_type)` 구현
   - 각 proposal의 `status == "approved" | "rejected"` 확인 후 → `"pending"` 복원
   - 이미 적용된 제안(committed 상태)은 롤백 불가 → 에러 반환

### 제외 범위

- 개별 승인/거절(단건) 롤백 — 별도 요구사항
- 30초 이후 서버사이드 롤백 — 타임아웃은 프론트엔드에서만 관리

## 3. 선행 조건

- Phase 6 FG6.2 완료: AdminProposalsPage BatchToolbar 구현
- Phase 6 proposals API (`POST /admin/proposals/batch-approve`, `batch-reject`) 구현
- S2 원칙 ⑤ 감사 로그 구조 확정

## 4. 주요 작업 항목

### 4-1. 프론트엔드 수정

**파일:** `frontend/src/features/admin/proposals/AdminProposalsPage.tsx`

#### 새로운 상태 추가

```tsx
// 기존 상태에 추가
const [lastBatchOp, setLastBatchOp] = useState<{
  type: "approve" | "reject";
  ids: string[];
  completedAt: number;
} | null>(null);
const [undoCountdown, setUndoCountdown] = useState(0);

// 카운트다운 타이머
useEffect(() => {
  if (!lastBatchOp) return;
  setUndoCountdown(30);

  const interval = setInterval(() => {
    setUndoCountdown((prev) => {
      if (prev <= 1) {
        clearInterval(interval);
        setLastBatchOp(null);
        return 0;
      }
      return prev - 1;
    });
  }, 1000);

  return () => clearInterval(interval);
}, [lastBatchOp]);
```

#### batchRollbackMut 추가

```tsx
const batchRollbackMut = useMutation({
  mutationFn: (ids: string[]) =>
    proposalsApi.batchRollback({
      proposal_ids: ids,
      original_action: lastBatchOp!.type,
    }),
  onSuccess: () => {
    qc.invalidateQueries({ queryKey: ["admin", "proposals"] });
    setLastBatchOp(null);
    setSelectedIds([]);
  },
});
```

#### 기존 배치 뮤테이션 onSuccess 수정

```tsx
const batchApproveMut = useMutation({
  mutationFn: (ids: string[]) => proposalsApi.batchApprove(ids),
  onSuccess: (_, ids) => {
    qc.invalidateQueries({ queryKey: ["admin", "proposals"] });
    setSelectedIds([]);
    setLastBatchOp({ type: "approve", ids, completedAt: Date.now() });  // 추가
  },
});

const batchRejectMut = useMutation({
  mutationFn: (ids: string[]) => proposalsApi.batchReject(ids),
  onSuccess: (_, ids) => {
    qc.invalidateQueries({ queryKey: ["admin", "proposals"] });
    setSelectedIds([]);
    setLastBatchOp({ type: "reject", ids, completedAt: Date.now() });  // 추가
  },
});
```

#### BatchToolbar에 Undo 버튼 추가

```tsx
{/* 기존 BatchToolbar 하단에 추가 */}
{lastBatchOp && (
  <div className="fixed bottom-20 left-1/2 -translate-x-1/2 z-40 flex items-center gap-3 bg-gray-900 text-white px-5 py-3 rounded-xl shadow-xl" role="status" aria-live="polite">
    <span className="text-sm">
      {lastBatchOp.ids.length}개 제안 {lastBatchOp.type === "approve" ? "승인" : "거절"} 완료
    </span>
    <button
      type="button"
      disabled={batchRollbackMut.isPending}
      onClick={() => batchRollbackMut.mutate(lastBatchOp.ids)}
      className="text-sm font-semibold text-yellow-300 hover:text-yellow-200 px-3 py-1 rounded-lg hover:bg-white/10 focus:outline-none focus:ring-2 focus:ring-yellow-400 min-h-[36px] disabled:opacity-50"
    >
      {batchRollbackMut.isPending ? "되돌리는 중..." : `되돌리기 (${undoCountdown}s)`}
    </button>
    <button
      type="button"
      onClick={() => setLastBatchOp(null)}
      className="p-1 rounded hover:bg-white/10 focus:outline-none focus:ring-2 focus:ring-white"
      aria-label="닫기"
    >
      <svg className="w-4 h-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
      </svg>
    </button>
  </div>
)}
```

### 4-2. API 클라이언트 추가

**파일:** `frontend/src/lib/api/s2admin.ts`

```typescript
// proposalsApi에 추가
batchRollback: async (body: {
  proposal_ids: string[];
  original_action: "approve" | "reject";
}) => {
  const res = await fetch(`${BASE}/admin/proposals/batch-rollback`, {
    method: "POST",
    headers: await adminHeaders(),
    body: JSON.stringify(body),
  });
  if (!res.ok) throw new Error("batch rollback failed");
  return res.json() as Promise<{ rolled_back: number; skipped: number }>;
},
```

### 4-3. 백엔드 엔드포인트

**파일:** `backend/app/api/v1/admin/proposals.py`

```python
@router.post("/proposals/batch-rollback")
async def batch_rollback_proposals(
    body: BatchRollbackRequest,
    actor: ActorPrincipal = Depends(get_actor),
    db: AsyncSession = Depends(get_db),
):
    """
    일괄 처리된 제안을 pending 상태로 복원.
    
    - 이미 committed(문서 반영된) 제안은 롤백 불가 → skipped 카운트
    - 감사 로그: action=batch_rollback, actor_type 기록 (S2 ⑤)
    """
    service = ProposalService(db)
    result = await service.batch_rollback(
        proposal_ids=body.proposal_ids,
        original_action=body.original_action,
        actor_id=actor.id,
        actor_type=actor.type,  # "user" | "agent"
    )

    await audit_log(
        db,
        actor_id=actor.id,
        actor_type=actor.type,
        event_type="batch_rollback",
        resource_type="proposal",
        resource_ids=body.proposal_ids,
        action_result="success",
        metadata={"rolled_back": result.rolled_back, "skipped": result.skipped},
    )

    return result


class BatchRollbackRequest(BaseModel):
    proposal_ids: list[str] = Field(..., min_items=1, max_items=500)
    original_action: Literal["approve", "reject"]


class BatchRollbackResponse(BaseModel):
    rolled_back: int
    skipped: int
    skipped_ids: list[str] = Field(default_factory=list)
```

**서비스 계층:**

```python
async def batch_rollback(
    self,
    proposal_ids: list[str],
    original_action: str,
    actor_id: str,
    actor_type: str,
) -> BatchRollbackResponse:
    """
    제안들을 pending 상태로 롤백.
    
    - status가 approved/rejected인 것만 롤백 가능
    - committed 상태인 것은 skipped
    """
    rolled_back = 0
    skipped = 0
    skipped_ids = []

    for proposal_id in proposal_ids:
        proposal = await self.repo.get_by_id(proposal_id)
        if proposal is None:
            skipped += 1
            skipped_ids.append(proposal_id)
            continue

        # committed 상태(문서에 반영됨)는 롤백 불가
        if proposal.status == "committed":
            skipped += 1
            skipped_ids.append(proposal_id)
            continue

        # approved/rejected → pending 복원
        if proposal.status in ("approved", "rejected"):
            await self.repo.update_status(
                proposal_id,
                status="pending",
                updated_by=actor_id,
            )
            rolled_back += 1
        else:
            skipped += 1
            skipped_ids.append(proposal_id)

    return BatchRollbackResponse(
        rolled_back=rolled_back,
        skipped=skipped,
        skipped_ids=skipped_ids,
    )
```

## 5. 산출물

1. **프론트엔드 수정** (`AdminProposalsPage.tsx`)
   - `lastBatchOp` 상태 관리
   - 30초 카운트다운 Undo 토스트 UI
   - `batchRollbackMut` 뮤테이션

2. **API 클라이언트 추가** (`s2admin.ts`)
   - `proposalsApi.batchRollback()` 함수

3. **백엔드 엔드포인트** (`backend/app/api/v1/admin/proposals.py`)
   - `POST /admin/proposals/batch-rollback`
   - `BatchRollbackRequest`, `BatchRollbackResponse` 모델

4. **서비스 계층** (`backend/app/services/proposal_service.py`)
   - `batch_rollback()` 메서드

## 6. 완료 기준

1. 일괄 승인/거절 직후 30초 카운트다운과 함께 "되돌리기" 버튼이 표시되는가?
2. 30초 경과 후 버튼이 자동으로 사라지는가?
3. "되돌리기" 클릭 시 해당 제안들이 `pending` 상태로 복원되는가?
4. `committed` 상태 제안은 롤백이 거부되고 `skipped` 카운트에 포함되는가?
5. 감사 로그에 `action=batch_rollback`, `actor_type` 필드가 기록되는가?
6. 에이전트(`actor_type=agent`)도 API 통해 롤백 호출이 가능한가? (S2 ⑤)
7. 카운트다운 중 페이지 이동 시 타이머가 정리(cleanup)되는가?
8. 되돌리기 후 제안 목록이 자동으로 갱신되는가?

## 7. 작업 지침

### 지침 7-11-1. 30초 타임아웃은 프론트엔드만
서버에서 시간 기반 롤백 만료를 강제하지 않는다. 30초는 UX 편의를 위한 클라이언트 타이머다. 30초 이후에도 API는 받아들이지만, UI 버튼이 사라져 UX상 불가능하게 보일 뿐이다.

### 지침 7-11-2. committed 상태 정의
`committed` = 해당 제안이 승인되어 실제 문서에 반영(커밋)된 상태. 이 상태에서는 단순 status 복원으로 되돌릴 수 없으므로 롤백 불가 처리한다.

### 지침 7-11-3. 접근성
Undo 토스트는 `role="status" aria-live="polite"`를 사용하여 스크린 리더에게 상태 변화를 알린다. 카운트다운 숫자는 매 초 갱신되어 시각적으로 표시된다.
