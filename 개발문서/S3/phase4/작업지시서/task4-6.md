# task4-6 — L2 Draft 쓰기 도구 실험적 노출 (FG 4-6)

**작성일**: 2026-04-28
**Phase / FG**: S3 Phase 4 / FG 4-6 (이월 — 조건부)
**상태**: **착수 대기 — 사용자 합의 필요** (Phase 4 1라운드 종결 + FG 4-5 권장 후행)
**담당**: 백엔드 (설계 Claude / 구현 Claude / 리뷰 Codex — extended)
**예상 소요**: 4~5일 (4 사전 조건 모두 갖춰진 후)
**Approver**: `@최철균` (P1 — Meta-1, 제45조). 본 FG 는 **MCP 표면 최초의 쓰기 도구** — 별도 P1 게이트로 격상.
**선행 산출물**:
- Phase 4 1라운드 (FG 4-0 ~ 4-4) 모두 종결
- FG 4-5 (capability manifest 확장) 권장 종결 — `policy_profile="write_audited"` 사용
- Phase 3 FG 1 (agent_proposal_service) 안정 — human approval 패턴 재사용
- 1라운드 R1~R5 회귀 모두 녹색

---

## 1. 작업 목적

MCP 표면에 **최초의 쓰기 도구** (`save_draft`) 를 실험적으로 노출한다. L1 (read + 결정성) 위에 L2 (단일 사용자 의도의 가역 쓰기) 를 추가하면서, 4 사전 조건을 갖춰 위험을 통제한다:

1. **idempotency** — 같은 idempotency_key 재시도 시 같은 결과 (네트워크 재시도 안전)
2. **human approval** — 자동 머지 금지. 사람 reviewer 승인 후에만 본 draft 가 적용 (S2 Phase 5 의 `agent_proposal_service` 패턴 재사용)
3. **impact preview** — 적용 전 영향 (변경 노드 수, 글자 수, 기존 draft 덮어씀 여부) 사전 노출
4. **감사 로그 4종** — request / approval / merge / rollback 각각 별도 event_type

본 FG 는 **L3/L4 영구 제외** (R1) — `publish` / `reindex` / `change_schema` / `delete` 는 본 FG 범위가 아니라 헌법 제20조의 영구 비대상.

> ⚠️ **본 FG 는 보안 영향이 큰 변경**이다. 각 사전 조건 마다 회귀 테스트 + 운영자 합의 단계 거침. 사전 조건 미충족 시 본 FG 미진행.

---

## 2. 작업 범위

### 2.1 포함

#### 2.1.1 4 사전 조건의 코드 측 충족

본 FG 가 진행 가능한지 사전 점검:

| 조건 | 현 상태 | 본 FG 에서 충족 |
|---|---|---|
| idempotency | `agent_proposal_service.propose_draft` 가 idempotency_key 미지원 | ✅ 신설 — proposal 테이블에 idempotency_key 컬럼 + 단위 테스트 |
| human approval | ✅ 이미 존재 — `agent_proposal_service.{propose,approve,reject}_draft` (S2 Phase 5) | 재사용 |
| impact preview | ❌ 부재 — 본 FG 신설 | ✅ `_compute_draft_impact(conn, doc_id, snapshot)` — diff 계산 |
| 감사 로그 4종 | `agent_proposal.*` event_type 일부 존재 | ✅ 4종 표준화 (`agent_proposal.requested` / `.approved` / `.merged` / `.rolled_back`) |

#### 2.1.2 idempotency

`agent_proposals` 테이블 + repository 에 `idempotency_key VARCHAR(128)` 컬럼 추가 (Alembic revision `s3_p4_agent_proposal_idempotency`):

```sql
ALTER TABLE agent_proposals
    ADD COLUMN idempotency_key VARCHAR(128);

-- 동일 (agent_id, idempotency_key) 중복 INSERT 차단
CREATE UNIQUE INDEX idx_agent_proposals_agent_idempotency
    ON agent_proposals(agent_id, idempotency_key)
    WHERE idempotency_key IS NOT NULL;
```

`AgentProposalService.propose_draft` 갱신:
- 입력 `idempotency_key` 가 주어지고 동일 (agent_id, key) 가 이미 존재하면 → 기존 proposal 재사용 (새 INSERT 안 함)
- 동일성 보장: 같은 (agent, key) → 같은 (proposal_id, status, version)

#### 2.1.3 impact preview

`agent_proposal_service` 에 신설:

```python
@dataclass
class DraftImpactPreview:
    document_id: str
    target_version_id: Optional[str]  # None = 신규 문서
    overwrites_existing_draft: bool
    nodes_added: int
    nodes_modified: int
    nodes_deleted: int
    chars_added: int
    chars_removed: int
    summary: str  # 자연어 요약 (운영자 합의)


def compute_draft_impact(
    conn, *, document_id: str, snapshot: dict,
) -> DraftImpactPreview:
    """save_draft 적용 시 발생할 영향을 사전 계산.

    실제 DB 변경 없이 diff 만 산출. snapshot_sync_service 의 diff 로직 위임.
    """
    ...
```

#### 2.1.4 감사 로그 4종 표준화

`event_types.md` 에 등재:

| event_type | 시점 | metadata |
|---|---|---|
| `agent_proposal.requested` | tool_save_draft 호출 직후 (proposal 생성) | `{document_id, idempotency_key, impact: {...}}` |
| `agent_proposal.approved` | reviewer approve | `{proposal_id, approver_id, version_id_after}` |
| `agent_proposal.merged` | approved → committed (실 draft 반영) | `{proposal_id, version_id, nodes_changed}` |
| `agent_proposal.rolled_back` | approved 후 롤백 | `{proposal_id, reason, by_actor_id}` |

본 FG 의 핵심 보안 가치 — 모든 쓰기 시점이 audit trail 로 추적 가능.

#### 2.1.5 `tool_save_draft` MCP 도구

`backend/app/mcp/tools.py`:

```python
def tool_save_draft(request: SaveDraftRequest, actor: ActorContext, conn) -> SaveDraftData:
    """L2 draft 쓰기 — propose 만, merge 는 reviewer approval 후."""
    _check_agent_write_blocked(actor)              # 킬스위치 적용
    _check_tool_allowed(actor, "save_draft", conn=conn)

    # ACL 게이트 — 다른 scope 의 문서에 draft 못 만듦
    acl_extra = _resolve_acl_filter(actor, request.scope, request.access_context, conn)
    _ensure_document_allowed(conn, request.document_id, acl_extra)

    # impact preview 사전 계산
    impact = compute_draft_impact(
        conn, document_id=request.document_id, snapshot=request.content_snapshot
    )

    # idempotent propose
    proposal = agent_proposal_service.propose_draft(
        conn,
        agent_id=actor.agent_id,
        document_id=request.document_id,
        content_snapshot=request.content_snapshot,
        idempotency_key=request.idempotency_key,
        impact=impact,  # audit metadata 부착
    )

    # 감사 로그 1: requested
    _emit_audit(
        event_type="agent_proposal.requested",
        action="mcp.save_draft",
        actor=actor,
        metadata={
            "document_id": request.document_id,
            "proposal_id": proposal.id,
            "idempotency_key": request.idempotency_key,
            "impact": asdict(impact),
        },
    )

    return SaveDraftData(
        proposal_id=proposal.id,
        status=proposal.status,  # 'proposed' — reviewer approval 대기
        impact=impact,
        message="draft proposed — reviewer approval 후 적용됩니다.",
    )
```

**주의**: `tool_save_draft` 는 직접 draft 를 적용하지 **않음**. propose 만 — reviewer approve 시점에 `agent_proposal_service.approve_draft` 가 별도 트리거되어 실 draft 반영.

#### 2.1.6 write 응답 envelope

FG 4-1 의 read envelope 와 별 — write envelope 신설:

```python
class MCPWriteEnvelope(BaseModel):
    """L2 write 도구 응답 envelope (FG 4-6)."""
    content_role: Literal["mutation_proposed"] = "mutation_proposed"
    instruction_authority: Literal["none"] = "none"  # R4 그대로
    impact: Optional[DraftImpactPreview] = None
    proposal_id: Optional[str] = None
    requires_human_approval: bool = True
    audit_chain: list[str] = Field(default_factory=list)  # ['agent_proposal.requested', ...]
```

`MCPResponse` 에 `write_envelope: Optional[MCPWriteEnvelope]` 별 필드 추가 — read envelope (`envelope`) 와 분리.

#### 2.1.7 TOOL_SCHEMAS 등재 (experimental)

```python
{
    "name": "save_draft",
    "description": (
        "Propose a draft change to a document. "
        "Requires human reviewer approval before merge — agent cannot self-approve. "
        "Idempotent via idempotency_key."
    ),
    "risk_tier": "L2",
    "maturity": "experimental",  # AI 품질 + 운영 안정성 검증 후 beta 승격
    "status": "enabled",
    "exposure_policy": "MCP_ENABLED",
    # FG 4-5 5 필드 (권장 선행)
    "default_enabled": False,    # 명시 등록 필수 — write 자동 등록 금지
    "requires": ["fetch_node"],  # save_draft 는 일반적으로 fetch_node 와 함께 사용
    "preferred_use": "에이전트가 사람 reviewer 의 승인을 받아 draft 변경을 제안할 때",
    "policy_profile": "write_audited",
    "streaming_supported": False,
    "authentication": {...},
    "inputSchema": {
        "type": "object",
        "properties": {
            "document_id": {"type": "string", "format": "uuid"},
            "content_snapshot": {"type": "object", "description": "ProseMirror 트리"},
            "idempotency_key": {"type": "string", "minLength": 1, "maxLength": 128},
            "scope": {"type": "string"},
            "access_context": {"type": "object"},
        },
        "required": ["document_id", "content_snapshot", "idempotency_key"],
    },
}
```

**`default_enabled=False`** — 운영자가 명시 등록한 ScopeProfile 만 본 도구 사용 가능 (FG 4-5 §2.1.5).

#### 2.1.8 ScopeProfile.allowed_tools — write_audited 그룹 격리

신규 도구 등록 시 운영 가이드:

> ⚠️ `save_draft` 는 `write_audited` 정책 그룹. 운영자가 ScopeProfile.allowed_tools 에 등록할 때 별도 합의 + audit 의무. read 도구와 별 ScopeProfile 분리 권고.

#### 2.1.9 회귀 테스트

`backend/tests/integration/test_save_draft_fg46.py`:

**idempotency**:
- 같은 (agent_id, idempotency_key) 두 번 propose → 같은 proposal_id 반환, 새 INSERT 0
- 다른 idempotency_key → 새 proposal
- idempotency_key NULL → 매번 새 proposal (기존 동작 보존)

**human approval**:
- 도구 응답이 `status="proposed"` (자동 approved 아님)
- 반환된 proposal_id 로 `approve_draft` 호출 → status=approved, draft 적용
- agent 가 자기 proposal 을 self-approve 시도 → 거부 (기존 agent_proposal_service 검사 그대로)

**impact preview**:
- 신규 노드 N개 추가 → impact.nodes_added==N
- 기존 draft 덮어쓰기 → overwrites_existing_draft=True
- chars_added / chars_removed 정확성 (3 시나리오)

**감사 로그 4종**:
- propose → `agent_proposal.requested` event 출현
- approve → `agent_proposal.approved` event
- merge → `agent_proposal.merged` event
- rollback → `agent_proposal.rolled_back` event
- 4 event 의 metadata 에 proposal_id 가 모두 동일 (chain 추적 가능)

**ACL**:
- 다른 scope 의 document_id → propose 거부
- ScopeProfile.allowed_tools 에 save_draft 미등록 → MCPError(UNAUTHORIZED, 403)

**write envelope (FG 4-6 §2.1.6)**:
- 응답에 `write_envelope.requires_human_approval=True`
- `write_envelope.audit_chain` 이 ['agent_proposal.requested'] 시작

**L3/L4 영구 제외 (R1 강화)**:
- TOOL_SCHEMAS 에 `publish_document` / `reindex_document` / `change_schema` / `delete_document` 등재 0 — 본 FG 후에도 그대로 보장 (회귀)
- `is_tool_mcp_exposed` 가 `risk_tier="L3"` 거부 (정의 단계 안전망)

#### 2.1.10 마이그레이션 가이드 + 운영자 승인

`산출물/FG4-6_마이그레이션_가이드.md`:
- Alembic revision `s3_p4_agent_proposal_idempotency` 적용 절차
- TOOL_SCHEMAS 에 save_draft 등재 시점 — **`@최철균` P1 승인 의무**
- ScopeProfile 에 save_draft 등록은 운영자가 명시 — bootstrap 스크립트는 본 도구 자동 추가 안 함
- 모니터링 항목: `agent_proposal.requested` event 폭증 / `requires_human_approval=True` 응답 수
- 롤백 절차: TOOL_SCHEMAS 에서 `status="not_exposed"` 로 변경 → 즉시 비노출

### 2.2 제외 (영구 또는 별 라운드)

- **L3 도구** (publish_document / approve_workflow_transition) — 본 FG 영구 제외 (R1)
- **L4 도구** (reindex / change_schema / delete) — 본 FG 영구 제외 (R1)
- **multi-document batch save_draft** — 별 라운드 (단일 docs 만 본 FG)
- **draft 직접 수정** (PATCH 방식 — 부분 변경) — 별 라운드. 본 FG 는 PUT (전체 snapshot) 만
- **자동 approve 정책** (high-confidence draft 는 자동 머지) — 영구 제외. human approval 절대 의무

### 2.3 하드코딩 금지 재확인

- `requires_human_approval=True` 가 envelope 의 default — 도구 함수에서 False 로 override 금지
- idempotency_key 의 length 제약 (128) 은 schema 정본 — DB constraint 와 동기화
- audit event_type 4종은 `event_types.md` 가 정본 — 코드 인라인 금지
- L3/L4 영구 금지 정책은 `is_tool_mcp_exposed` 가 정본 — TOOL_SCHEMAS 추가 시 차단

---

## 3. 선행 조건

- Phase 4 1라운드 + FG 4-5 종결
- 운영자 `@최철균` **별도 P1 승인** — 본 FG 는 MCP 표면 최초 쓰기 도구
- `agent_proposal_service` 안정 (S2 Phase 5 자산) + 단위 테스트 녹색
- `snapshot_sync_service` 의 diff 로직 안정 (FG 1-1 자산) — impact preview 가 위임
- ScopeProfile.allowed_tools (FG 4-0 §2.1.6) 동작 — save_draft 등록 가능 상태

---

## 4. 구현 단계

### Step 1 — Pre-flight 추가 점검

1. agent_proposal_service 의 propose_draft / approve_draft 호환성 확인
2. snapshot_sync_service 의 diff 로직 — impact preview 위임 가능 여부
3. event_types.md 에 4 신규 event 추가 합의
4. 운영자 합의: save_draft 의 idempotency_key 정책 (필수 vs 선택), approve UI 위치

### Step 2 — 4 사전 조건 충족

1. **idempotency**: Alembic revision + repository 갱신 + 단위 테스트
2. **human approval**: agent_proposal_service 재사용 — 신규 코드 0
3. **impact preview**: `compute_draft_impact` 신설 + 단위 테스트
4. **감사 로그 4종**: event_types.md 등재 + emit 4 시점 표준화

### Step 3 — write envelope 신설

1. `MCPWriteEnvelope` 모델
2. `MCPResponse.write_envelope: Optional[MCPWriteEnvelope]`
3. read envelope 과 분리 — content_role Literal 별

### Step 4 — tool_save_draft 신설

1. SaveDraftRequest / SaveDraftData schema
2. tool_save_draft 함수 (idempotency + impact + propose + audit)
3. mcp_router._dispatch_tool 분기 추가
4. _build_envelope 갱신 — save_draft 는 write_envelope 사용

### Step 5 — TOOL_SCHEMAS 등재

1. save_draft entry (L2, experimental, default_enabled=False)
2. dump_mcp_manifest 재생성 + commit
3. drift 게이트 통과 확인

### Step 6 — Admin UI 갱신

1. KNOWN_MCP_TOOLS (FG 4-5 동적 fetch) 가 자동으로 save_draft 표시
2. write_audited 정책 그룹 색상 + 경고 텍스트 추가
3. 토글 시 confirm 다이얼로그 ("이 도구는 쓰기 권한입니다. 등록하시겠습니까?")
4. UI 리뷰 ≥ 1회

### Step 7 — 회귀 테스트

1. test_save_draft_fg46.py 작성 (≥ 30 시나리오)
2. contract drift 테스트 갱신 — L3/L4 영구 제외 회귀

### Step 8 — AI 품질 평가 (실험적 도구이므로 경량)

`scripts/rag_smoke/golden_save_draft.py`:
- agent 가 자연어 요청을 받아 draft 를 propose 하는 시나리오 ≥ 20 케이스
- 지표: **propose 정확성** (요청한 내용이 draft 에 반영됨), **impact 일관성** (preview 와 실제 변경 일치)
- 임계: propose 정확성 ≥ 0.80, impact 일관성 ≥ 0.95

운영자 staging measurement (Phase 0 FG 0-4 패턴).

### Step 9 — 검수 / 보안 / 마이그레이션 / AI 품질 보고서

- `FG4-6_검수보고서.md` — 4 사전 조건 충족 + R1 (L3/L4 영구 제외) 회귀
- `FG4-6_보안취약점검사보고서.md` — write 도구 위협 모델 (T1: agent self-approval / T2: idempotency 위조 / T3: impact 위조 / T4: 감사 로그 누락 / T5: ACL 우회 등)
- `FG4-6_마이그레이션_가이드.md`
- `FG4-6_AI품질평가보고서.md`

---

## 5. API 계약 변경 요약

| 메서드 | 도구 / 경로 | 변경 |
|---|---|---|
| POST | /mcp/tools/call (save_draft) | **신규** L2 도구. propose 만, 자동 머지 없음 |
| POST | /mcp/tools/call | 응답에 `write_envelope: MCPWriteEnvelope` (write 도구 한정) — 기존 envelope 와 분리 |
| (DDL) | agent_proposals 테이블 | `idempotency_key VARCHAR(128)` 컬럼 + UNIQUE INDEX |

---

## 6. 데이터 모델 주의사항

- Alembic revision `s3_p4_agent_proposal_idempotency` (revision id 31자 이내 — FG 4-0 §2.1.6 lesson)
- DROP COLUMN downgrade — 기존 idempotency_key 정보 손실 명시
- backfill 불필요 (NULL 허용)
- agent_proposals 테이블 행 수 큰 경우 ALTER TABLE lock 시간 — 야간 적용 권고

---

## 7. 성공 기준

- [ ] 4 사전 조건 모두 충족 (idempotency / human approval / impact preview / 감사 4종)
- [ ] tool_save_draft 가 propose 만 — 자동 머지 0
- [ ] L3/L4 영구 제외 회귀 (FG 4-4 contract drift 강화)
- [ ] save_draft 응답이 `requires_human_approval=True` 항상
- [ ] 같은 (agent, idempotency_key) → 같은 proposal (idempotent)
- [ ] 4 audit event 가 chain 으로 추적 가능
- [ ] ScopeProfile.allowed_tools 미등록 시 거부 + default_enabled=False
- [ ] AI 품질: propose 정확성 ≥ 0.80, impact 일관성 ≥ 0.95
- [ ] pytest 신규 ≥ 30 / 전체 베이스라인 유지
- [ ] 검수·보안·마이그레이션·AI 품질 보고서 제출

---

## 8. 리스크

| 리스크 | 대응 |
|---|---|
| agent 가 자기 proposal 을 self-approve | agent_proposal_service.approve_draft 가 actor.role 검사 — agent 본인 거부 (S2 Phase 5 자산 그대로) |
| idempotency_key 위조 (다른 agent 의 key 재사용) | UNIQUE (agent_id, idempotency_key) — 같은 agent 만 idempotent. 다른 agent 는 새 INSERT |
| impact preview 가 실제 적용 결과와 차이 | preview 와 actual diff 를 audit metadata 에 양쪽 기록 — drift 발견 즉시 본 도구 status="disabled" 로 즉시 차단 |
| reviewer approval UI 가 missing → propose 가 영구 pending | proposal pending 시간 모니터링 + 운영자 알림 (별 작업) |
| save_draft 가 외부 클라이언트에 노출되어 자동화 폭주 | default_enabled=False + ScopeProfile 명시 등록 의무 + maturity=experimental — 운영자가 ROI 검증 후 활성 |
| MCPWriteEnvelope 가 read envelope 와 혼용되어 클라이언트 혼란 | 응답 schema 가 둘을 분리 (`envelope` vs `write_envelope`). 동시 출현 0 |
| L3/L4 도구가 본 FG 작업 중 실수로 노출 | FG 4-4 contract drift 테스트가 머지 차단. 본 FG 도 회귀 강화 |
| Alembic 적용 후 idempotency_key 관련 race condition | INSERT ON CONFLICT (agent_id, idempotency_key) DO NOTHING + RETURNING — 트랜잭션 격리 |

---

## 9. 참조

- `Phase 4 개발계획서.md` §1.2 (R1 — L4 영구 금지), §2.2 (FG 4-6 이월 — 4 사전 조건)
- `task4-5.md` (capability manifest 확장 — `policy_profile="write_audited"` 사용)
- `task4-4.md` (contract drift 테스트 — L3/L4 영구 제외 회귀 강화)
- `uploads/Mimir API : MCP 운영 논의 요약` §6 (write 도구 / human approval)
- `CONSTITUTION.md` 제20조 (Typed Action Risk), 제45조 (Human Approval Gates), 제46조 (Approval Binds to Change Set)
- `backend/app/services/agent_proposal_service.py` (S2 Phase 5 — propose/approve/reject)
- `backend/app/services/draft_service.py` (실 draft 적용 — approve 후)
- `backend/app/services/snapshot_sync_service.py` (impact preview 의 diff 로직 위임)

---

*작성: 2026-04-28 | FG 4-6 — L2 Draft 쓰기 도구 실험적 노출 (이월 / 조건부 / P1 별도 게이트)*
