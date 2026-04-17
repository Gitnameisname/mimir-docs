# FG5.1 보안취약점검사보고서 — 에이전트 쓰기 = Draft 제안 전용

| 항목 | 내용 |
|---|---|
| 기능 그룹 | FG5.1 |
| 검사일 | 2026-04-17 |
| 검사자 | 개발팀 |
| 검사 결과 | **이상 없음** |

---

## 1. 검사 범위

| 파일 | 역할 |
|---|---|
| `backend/app/services/agent_proposal_service.py` | 제안 생성·승인·반려·철회 비즈니스 로직 |
| `backend/app/api/v1/scope_profiles.py` | `/approve`, `/reject`, proposals 엔드포인트 |
| `backend/app/db/migrations/` | `agent_proposals`, `mcp_tasks` 테이블 DDL |
| `backend/app/models/document.py` | `WorkflowStatus.PROPOSED`, `WITHDRAWN` enum 추가 |

---

## 2. 취약점 검사 결과

### 2-1. 상태 전이 무결성

| 공격 시나리오 | 방어 로직 | 결과 |
|---|---|---|
| 에이전트가 `DRAFT` 상태로 직접 저장 시도 | `propose_draft()`에서 상태를 `PROPOSED`로 강제 설정 | ✅ 방어됨 |
| 에이전트가 자신의 제안을 직접 `approve_draft()` 호출 | `reviewer_role` 파라미터 필수 + 호출자 인증 확인 | ✅ 방어됨 |
| 이미 승인된 제안을 재승인 시도 | `PROPOSED` 상태가 아닌 경우 예외 발생 | ✅ 방어됨 |
| `WITHDRAWN` 제안을 승인 시도 | 상태 체크로 거부 | ✅ 방어됨 |

### 2-2. 에이전트 인가 (Authorization)

| 검사 항목 | 결과 | 비고 |
|---|---|---|
| Kill Switch 활성화 시 제안 차단 | ✅ | `test_kill_switch_blocks_all` |
| 다른 에이전트의 제안 철회 불가 | ✅ | `agent_id` 검증 |
| Admin 전용 제안 큐(`/admin/proposals`) 일반 사용자 접근 차단 | ✅ | `require_admin` 의존성 |
| 에이전트가 인간 전용 엔드포인트(`/approve`) 직접 호출 가능 여부 | 가능하나 `reviewer_role` 검증 | ⚠️ 권고-1 참조 |

### 2-3. 감사 로그 무결성

| 검사 항목 | 결과 |
|---|---|
| 모든 에이전트 액션에 `actor_type='agent'` 기록 | ✅ |
| `acting_on_behalf_of` NULL 허용 (독립 에이전트) | ✅ |
| 감사 로그 삭제·수정 API 미노출 | ✅ |

### 2-4. SQL Injection

| 입력 경로 | 처리 방식 | 결과 |
|---|---|---|
| `propose_draft()` 파라미터 | SQLAlchemy ORM / asyncpg parameterized query | ✅ |
| `agent_id` URL 파라미터 | FastAPI 타입 검증 (UUID) | ✅ |
| `rejection_reason` 텍스트 | Parameterized INSERT | ✅ |

### 2-5. OWASP LLM Top 10 대응

| OWASP ID | 위협 | 대응 상태 |
|---|---|---|
| LLM08 | Excessive Agency — 에이전트가 직접 문서 발행 | ✅ PROPOSED 전용으로 강제 격리 |
| LLM09 | Overreliance — 자동 승인 없이 인간 검토 필수 | ✅ 승인 API 인간 호출 필수 |

---

## 3. 권고 사항

| # | 등급 | 내용 | 대응 우선순위 |
|---|---|---|---|
| 권고-1 | Low | 에이전트 Principal이 `/approve` 엔드포인트를 기술적으로 호출할 수 있음. `reviewer_role` 검증으로 막고 있으나, Agent Principal에 대한 명시적 거부 조건 추가 권장 | Phase 6 |

---

## 4. 결론

FG5.1 범위에서 Critical, High, Medium 등급의 보안 취약점은 발견되지 않았습니다.  
Low 등급 권고 1건은 Phase 6에서 에이전트 Principal 타입 검증 강화로 처리 예정입니다.

**FG5.1 보안취약점검사 결과: ✅ 이상 없음 (권고 1건)**
