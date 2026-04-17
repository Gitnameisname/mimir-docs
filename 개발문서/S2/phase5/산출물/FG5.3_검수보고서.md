# FG5.3 검수보고서
## 에이전트 감사 + 회귀 테스트

---

**작성일**: 2026-04-17  
**작성자**: 개발팀  
**검수 대상**: FG5.3 — 에이전트 감사 조회 API, 통계 API, Rate Limit, 회귀 테스트  
**검수 결과**: ✅ **통과** (전체 35개 테스트 통과, 신규 API 3개 구현 완료)

---

## 1. 검수 항목 및 결과

| 검수 항목 | 기준 | 결과 | 비고 |
|----------|------|------|------|
| 에이전트 감사 이력 API | GET /admin/agents/{id}/audit 정상 작동 | ✅ 통과 | 날짜 범위, 이벤트 타입 필터 지원 |
| 에이전트 통계 API | GET /admin/agents/{id}/statistics 정상 작동 | ✅ 통과 | 승인율, 평균 검토 시간, 반려 사유 분석 |
| Rate Limit 현황 API | GET /admin/agents/{id}/rate-limit 정상 작동 | ✅ 통과 | Valkey 장애 시 빈 목록 반환 (안전 처리) |
| 에이전트별 Rate Limit | Valkey 기반 per-agent 카운터 구현 | ✅ 통과 | REC-4.1 이월 완료 |
| fail-open 보장 | Valkey 연결 실패 시 통과 처리 | ✅ 통과 | S2 원칙 ⑦ (폐쇄망 환경) |
| 통합 e2e 시나리오 | 제안→승인→감사로그 전체 플로우 | ✅ 통과 | 16개 시나리오 |
| 에이전트 회귀 테스트 | tests/agents/default_agent_regression.py | ✅ 통과 | 6개 회귀 케이스 |
| 성능 테스트 | 인젝션 탐지 <100ms, 배치 <200ms | ✅ 통과 | 실측 <1ms (정규식 기반) |
| 감사 로그 actor_type 기록 | 모든 에이전트 활동 actor_type=agent | ✅ 통과 | FG5.1 구현 포함 |
| 통계 계산 정확성 | 승인율 = approved/total, 정밀도 0.0001 | ✅ 통과 | 5개 파라미터 케이스 검증 |

---

## 2. 신규 구현 파일

| 파일 | 역할 |
|------|------|
| `backend/app/api/v1/scope_profiles.py` | GET /admin/agents/{id}/audit, /statistics, /rate-limit 엔드포인트 추가 |
| `backend/app/api/v1/mcp_router.py` | `_check_agent_rate_limit()` 함수 추가, tool_call/stream에 per-agent 체크 적용 |
| `backend/tests/test_agent_end_to_end.py` | 통합 e2e 시나리오 16개 테스트 |
| `backend/tests/agents/default_agent_regression.py` | 기본 에이전트 회귀 테스트 6개 |
| `backend/tests/test_agent_performance.py` | 성능 테스트 9개 (latency, fail-open, 통계) |

---

## 3. API 명세

### GET /api/v1/admin/agents/{agent_id}/audit

**요청 파라미터**:
- `start_date` (optional): ISO8601 시작 날짜
- `end_date` (optional): ISO8601 종료 날짜
- `action_type` (optional): 이벤트 타입 필터 (예: `agent.draft.proposed`)
- `page` (default: 1): 페이지 번호
- `page_size` (default: 20, max: 100): 페이지 크기

**응답**:
```json
{
  "items": [
    {
      "id": "uuid",
      "event_type": "agent.draft.proposed",
      "occurred_at": "ISO8601",
      "actor_id": "agent-uuid",
      "actor_type": "agent",
      "acting_on_behalf_of": "user-uuid",
      "resource_type": "document",
      "resource_id": "doc-uuid",
      "previous_state": null,
      "new_state": "proposed",
      "action_result": "success",
      "reason": "자동 검토 완료"
    }
  ],
  "total": 100,
  "page": 1,
  "page_size": 20
}
```

### GET /api/v1/admin/agents/{agent_id}/statistics

**응답**:
```json
{
  "agent_id": "uuid",
  "agent_name": "에이전트명",
  "total_proposals": 100,
  "approved_count": 70,
  "rejected_count": 20,
  "withdrawn_count": 10,
  "approval_rate": 0.7,
  "average_review_time_minutes": 45.3,
  "last_activity": "ISO8601",
  "rejection_reasons": [
    {"reason": "부정확한 정보", "count": 8},
    {"reason": "양식 오류", "count": 5}
  ]
}
```

### GET /api/v1/admin/agents/{agent_id}/rate-limit

**응답**:
```json
{
  "agent_id": "uuid",
  "endpoints": [
    {
      "endpoint": "tool_call",
      "current_count": 5,
      "ttl_seconds": 43
    }
  ]
}
```

---

## 4. 에이전트별 Rate Limit 구현 (REC-4.1 이월 완료)

### 설계
- **Valkey 키 패턴**: `agent:{agent_id}:rate:{endpoint}`
- **윈도우**: 60초 슬라이딩
- **한도**: tool_call=20, stream=10, read=60, init=30 (분당)
- **fail-open**: Valkey 연결 실패 시 통과 (S2 원칙 ⑦)

### 적용 엔드포인트
- `POST /mcp/tools/call` — tool_call 엔드포인트
- `POST /mcp/tools/call/stream` — stream 엔드포인트

---

## 5. 테스트 전체 결과 요약

| 테스트 파일 | 케이스 수 | 통과 | 실패 |
|------------|---------|------|------|
| test_fg5_1_agent_proposals.py | 21 | 21 | 0 |
| test_prompt_injection_regression.py | 13 | 13 | 0 |
| test_agent_end_to_end.py | 16 | 16 | 0 |
| agents/default_agent_regression.py | 6 | 6 | 0 |
| test_agent_performance.py | 9 | 9 | 0 |
| **합계** | **65** | **65** | **0** |

---

## 6. 검수 기준 충족 여부

| 검수 기준 | 충족 여부 |
|----------|---------|
| 감사 로그에 모든 에이전트 활동이 actor_type=agent로 기록되는가 | ✅ |
| 에이전트별 통계가 정확하게 계산되는가 (승인율, 평균 검토 시간) | ✅ |
| 회귀 테스트가 CI 파이프라인에서 자동 실행 가능한가 | ✅ |
| 통합 e2e 시나리오가 모든 Phase 5 기능을 검증하는가 | ✅ |
| 성능 테스트 결과: 인젝션 탐지 <100ms | ✅ (실측 <1ms) |
| Rate Limit이 에이전트 ID 단위로 적용되는가 | ✅ |
| Valkey 장애 시 서비스가 fail-open으로 동작하는가 | ✅ |

---

## 7. 잔여 사항 (Phase 6 이관)

- Admin UI에서 감사 로그 및 통계 대시보드 구축 (FG6.2 예정)
- 에이전트별 Rate Limit 한도를 ScopeProfile에서 동적으로 설정하는 기능 (Phase 6 이관)
- 성능 부하 테스트 (1000개 이상 동시 제안) — Phase 6 배포 전 실행 권장
