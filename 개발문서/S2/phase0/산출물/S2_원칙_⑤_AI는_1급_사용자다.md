# S2 원칙 ⑤ — AI는 1급 사용자다 (AI-as-first-class-consumer)

**확정일**: 2026-04-16
**근거**: Season 2 개발계획.md §1, §3.1, §3.3, §3.5

---

## 1. 정의

모든 기능은 사람 UI 이전에 AI 에이전트가 **안전하게 호출·검증·기록**할 수 있도록 정의된다.  
"AI 에이전트도 사람과 동일한 API 계약으로 Mimir를 소비할 수 있다"가 이 원칙의 핵심이다.

---

## 2. 배경

S1의 원칙 ①(API-first)은 "UI는 클라이언트 중 하나"라는 명제였다.  
S2에서는 이 명제를 AI 소비자까지 확장한다.  
1차 외부 소비자인 챗봇은 독립 백엔드 프로세스로 Mimir를 서버-투-서버 호출한다.  
챗봇이 요구하는 API 계약이 사람 UI보다 더 엄격하다 — 구조화 응답, 재현 가능한 citation, 명시적 감사 로그.

---

## 3. 구현 기준

| 구현 영역 | 구체적 기준 |
|----------|-----------|
| **API 우선 설계** | 모든 기능은 REST API 또는 MCP Tool로 먼저 정의된다. UI는 그 API를 소비한다. |
| **구조화 응답** | 모든 검색·조회 응답에 `{success, data, error}` envelope 적용. AI가 파싱 가능한 JSON 보장. |
| **응답 메타 필드** | 응답에 `trusted`, `source`, `provenance` 포함. AI 에이전트가 근거 신뢰도를 판단할 수 있게 함. |
| **오류 표준화** | 오류 응답: `{code, message, recoverable}`. AI가 자동 재시도 여부를 판단할 수 있게 함. |
| **감사 로그** | 모든 API 호출에 `actor_type` 필드 필수 기록. `"user"` 또는 `"agent"`. |

---

## 4. 적용 범위 (Phase별)

| Phase | 적용 사항 |
|-------|---------|
| Phase 0 | API 응답 envelope 정규화 (`unwrapEnvelope<T>()`), 감사 로그 `actor_type` 필드 |
| Phase 2 | Citation 5-tuple — AI가 검색 결과의 근거를 검증 가능한 좌표로 수신 |
| Phase 3 | 세션 기반 RAG API — AI 에이전트가 멀티턴 대화 상태를 Mimir에 위임 |
| Phase 4 | MCP Tool 노출 — `search_documents`, `fetch_node`, `create_draft`, `propose_transition` |
| Phase 5 | 에이전트 쓰기 API — `proposed` 상태 Draft 생성, MCP Tasks 비동기 승인 플로우 |

---

## 5. 반례 (위반 패턴)

```python
# ❌ 금지: UI 전용 로직 (API 없이 페이지에만 존재)
@router.get("/ui/documents", include_in_schema=False)
def get_documents_for_ui():
    # AI 에이전트가 호출할 수 없음
    ...

# ❌ 금지: 감사 로그에 actor_type 누락
audit_log(action="create_draft", actor_id=user.id)  # actor_type 없음

# ✅ 올바른 패턴
audit_log(action="create_draft", actor_id=actor.id, actor_type=actor.type)
```

---

## 6. FAQ — 현장에서의 의문

**Q. "UI에만 있는 기능"은 정말 금지인가?**  
A. 시각화·인터랙션 등 UI 고유 행동은 제외. 하지만 그 기능의 *데이터 조회·변경 로직*은 반드시 API로 추상화해야 한다.

**Q. 기존 S1 API에 `trusted` 필드를 소급 추가해야 하나?**  
A. Phase 0에서는 `unwrapEnvelope` 정규화가 우선. `trusted` 메타 필드는 Phase 4(Agent Interface)에서 AI가 실제로 소비할 시점에 추가한다.

**Q. 감사 로그 `actor_type` 필드가 없는 기존 로그는 어떻게 하나?**  
A. 기존 로그는 `actor_type="user"` 로 기본값을 설정하는 DB 마이그레이션을 Phase 4 착수 전에 수행한다.

---

## 7. 이 원칙이 검수되는 Phase

| Phase | 검수 항목 |
|-------|---------|
| Phase 0 FG0.2 | 모든 API 호출이 `unwrapEnvelope<T>()` 패턴 사용하는가? |
| Phase 4 FG4.1 | MCP Tool schema가 OpenAPI 스펙과 일치하는가? |
| Phase 5 FG5.1 | 에이전트 쓰기가 감사 로그에 `actor_type=agent`로 기록되는가? |
| Phase 9 FG9.1 | OWASP LLM Top 10 — Excessive Agency (LLM08) 점검 포함 |

---

## 8. S1 원칙 ①(API-first)과의 관계

| 측면 | S1 원칙 ① | S2 원칙 ⑤ |
|------|----------|----------|
| 대상 소비자 | 사람 (UI 클라이언트) | 사람 + AI 에이전트 |
| 응답 포맷 | JSON, envelope 권장 | JSON envelope 필수 + 메타 필드 |
| 감사 | `actor_id` 기록 | `actor_id` + `actor_type` 필수 |
| 오류 처리 | HTTP 상태 코드 | `{code, message, recoverable}` 구조화 |

S2 원칙 ⑤는 S1 원칙 ①을 무효화하지 않는다. S1 원칙 위에서 AI 소비자 요건을 강화한다.
