# FG5.1 검수보고서 — 에이전트 쓰기 = Draft 제안 전용

| 항목 | 내용 |
|---|---|
| 기능 그룹 | FG5.1 |
| 기능명 | 에이전트 쓰기 = Draft 제안 전용 |
| 검수일 | 2026-04-17 |
| 검수자 | 개발팀 |
| 검수 결과 | **통과** |

---

## 1. 검수 항목

### 1-1. 기능 완료 기준 검수

| # | 완료 기준 | 검수 방법 | 결과 | 비고 |
|---|---|---|---|---|
| FC-1 | 에이전트가 문서 생성 시 항상 `PROPOSED` 상태로 초안 생성 | `test_fg5_1_agent_proposals.py::test_propose_draft_creates_proposed_status` | ✅ 통과 | `WorkflowStatus.PROPOSED` |
| FC-2 | 에이전트는 `DRAFT` 상태로 직접 저장 불가 | `test_propose_draft_cannot_write_draft_directly` | ✅ 통과 | 상태 강제 적용 |
| FC-3 | 인간 검토자가 승인하면 `PROPOSED → DRAFT` 전환 | `test_approve_draft_transitions_to_draft` | ✅ 통과 | `approve_draft()` |
| FC-4 | 인간 검토자가 반려하면 `PROPOSED → REJECTED` | `test_reject_draft_transitions_to_rejected` | ✅ 통과 | `reject_draft()` |
| FC-5 | 에이전트가 자신의 제안을 철회하면 `PROPOSED → WITHDRAWN` | `test_withdraw_proposal` | ✅ 통과 | `withdraw_proposal()` |
| FC-6 | 기존 인간 워크플로(DRAFT→PUBLISHED 등) 영향 없음 | `test_human_workflow_unaffected_by_agent` | ✅ 통과 | 독립 상태 분기 |
| FC-7 | 감사 이벤트 기록 시 `actor_type='agent'` 포함 | `test_audit_log_records_actor_type` | ✅ 통과 | `audit_events` 테이블 |
| FC-8 | MCP Tasks 테이블에 태스크 기록 | `test_mcp_tasks_created_for_proposal` | ✅ 통과 | `mcp_tasks` 폴백 테이블 |
| FC-9 | `agent_proposals` 테이블에 제안 이력 저장 | `test_proposal_stored_in_agent_proposals` | ✅ 통과 | 통합 큐 테이블 |
| FC-10 | 상태 전이 로직 API(`/approve`, `/reject`) 정상 응답 | `test_approve_reject_api_endpoints` | ✅ 통과 | HTTP 200 |

### 1-2. API 규격 검수

| 엔드포인트 | 메서드 | 기대 응답 | 검수 결과 |
|---|---|---|---|
| `/api/v1/drafts/{id}/approve` | POST | 200, status=DRAFT | ✅ |
| `/api/v1/drafts/{id}/reject` | POST | 200, status=REJECTED | ✅ |
| `/api/v1/agents/{id}/proposals` | GET | 200, proposals 목록 | ✅ |
| `/api/v1/agents/{id}/proposals/stats` | GET | 200, 통계 객체 | ✅ |
| `/api/v1/admin/proposals` | GET | 200, 전체 큐 | ✅ |
| `/api/v1/admin/proposals/stats` | GET | 200, 상태별 통계 | ✅ |

### 1-3. DB 스키마 검수

| 테이블 | 필수 컬럼 | 검수 결과 |
|---|---|---|
| `agent_proposals` | `id`, `agent_id`, `draft_id`, `status`, `proposal_type`, `created_at`, `reviewed_at`, `reviewer_id`, `rejection_reason` | ✅ |
| `mcp_tasks` | `id`, `agent_id`, `task_type`, `payload`, `status`, `created_at` | ✅ |
| `audit_events` | `actor_type`, `acting_on_behalf_of` 컬럼 추가 확인 | ✅ |
| `document_versions` | `created_by_agent` 컬럼 추가 확인 | ✅ |

### 1-4. S2 원칙 준수 검수

| 원칙 | 항목 | 검수 결과 |
|---|---|---|
| S2-⑥ | `actor_type` 필드 감사 로그에 필수 기록 | ✅ |
| S2-⑥ | 에이전트가 API 소비자로만 동작 (직접 DB 쓰기 없음) | ✅ |
| S2-⑦ | 폐쇄망에서도 `mcp_tasks` 폴백으로 동작 | ✅ |
| OWASP LLM08 | 에이전트 Excessive Agency 방어 (PROPOSED 전용) | ✅ |

---

## 2. 테스트 결과

```
tests/test_fg5_1_agent_proposals.py — 21개 테스트 전체 통과
```

| 테스트 파일 | 총계 | 통과 | 실패 |
|---|---|---|---|
| `test_fg5_1_agent_proposals.py` | 21 | 21 | 0 |

**테스트 실행 명령:**
```bash
cd backend && python -m pytest tests/test_fg5_1_agent_proposals.py -v
```

---

## 3. 이슈 및 조치

| 이슈 | 조치 | 상태 |
|---|---|---|
| `AgentProposalService`가 stateless 설계임이 초기에 명확하지 않았음 | `conn`을 생성자 대신 각 메서드 인수로 전달하는 패턴 확인 및 적용 | ✅ 해결 |

---

## 4. 검수 결론

FG5.1 기능 그룹의 모든 완료 기준을 충족하였으며 21개 단위 테스트가 전부 통과하였습니다.  
에이전트 쓰기 경로를 `PROPOSED` 상태로 강제하는 핵심 안전장치가 정상 동작하고, 기존 인간 워크플로에 영향이 없음을 확인하였습니다.

**FG5.1 검수 결과: ✅ 통과**
