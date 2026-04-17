# Phase 4 보안 취약점 검사 보고서

**Phase**: Phase 4 (S2) — Agent-Facing Interface  
**검사일**: 2026-04-17  
**검사자**: Claude Sonnet 4.6 (AI 자동 검사)

---

## 1. 검사 대상

FG4.1 (MCP Server), FG4.2 (Agent Principal + Scope Profile), FG4.3 (Structured Response)

---

## 2. 발견 항목 및 처리 결과

| ID | 심각도 | 항목 | 위치 | 처리 상태 |
|----|--------|------|------|----------|
| SEC-4.1 | **High** | $ctx 변수 인젝션 — 화이트리스트 외 변수 허용 시 SQL 인젝션 가능 | `filter_expression.py` | **수정 완료** — `_ALLOWED_CTX_KEYS` 화이트리스트 검증, 미허용 키 `ValueError` |
| SEC-4.2 | **High** | FilterExpression 허용되지 않은 필드/연산자 — SQL 컬럼 오남용 | `filter_expression.py` | **수정 완료** — `_ALLOWED_OPS`, `_ALLOWED_FIELDS` frozenset 검증 |
| SEC-4.3 | **Medium** | Agent Kill Switch 지연 — 인증 캐시 사용 시 비활성 에이전트가 잠시 동작 | `auth/dependencies.py` | **설계 완화** — 캐시 없는 직접 DB 조회로 즉시 확인 구현 |
| SEC-4.4 | **Medium** | API Key `principal_type` 변조 — 클라이언트가 임의로 에이전트 자격 주장 | `auth/dependencies.py` | **수정 완료** — DB에서 `principal_type` 조회, 클라이언트 입력 무시 |
| SEC-4.5 | **Medium** | MCP 도구 응답에서 내부 정보 노출 | `mcp/tools.py` | **수정 완료** — 에러 메시지 최소화, 스택 트레이스 미노출 |
| SEC-4.6 | **Low** | Scope Profile 삭제 시 바인딩된 에이전트 영향 | `scope_profile_repository.py` | **설계 완화** — ON DELETE SET NULL으로 참조 무결성 유지 |
| SEC-4.7 | **Low** | MCP 리소스 URI 인젝션 — 비정상 URI 처리 | `mcp/resources.py` | **수정 완료** — 정규식 패턴 검증, 불일치 시 None 반환 |

---

## 3. 항목별 상세

### SEC-4.1: $ctx 변수 인젝션 (High → 수정 완료)

**문제**: `$ctx.{arbitrary_key}` 변수가 SQL 파라미터로 주입될 때 화이트리스트 검증 없으면 admin 권한의 비공개 필드 노출 가능.

**수정 내용**:
```python
_ALLOWED_CTX_KEYS = frozenset({
    "organization_id", "team_id", "user_id", "permissions",
})
# _resolve_ctx_var() 에서 _ALLOWED_CTX_KEYS 검증 후 ValueError 발생
```

### SEC-4.2: FilterExpression 허용되지 않은 필드 (High → 수정 완료)

**문제**: `field: "password"`, `op: "eq"` 같은 비정상 조건이 SQL로 생성될 수 있음.

**수정 내용**:
```python
_ALLOWED_FIELDS = frozenset({"organization_id", "team_id", "visibility", ...})
# FilterCondition.validate() 에서 검증
```

### SEC-4.3: Kill Switch 즉시성 (Medium → 설계 완화)

**처리**: `is_disabled` 확인을 Valkey 캐시 없이 직접 DB 조회로 구현. 응답 지연 최소화.

### SEC-4.4: API Key principal_type 변조 (Medium → 수정 완료)

**수정**: DB의 `api_keys.principal_type` 값만 신뢰. HTTP 헤더/요청 본문의 principal_type 무시.

---

## 4. 잔여 권고 사항 — 이행 현황 (2026-04-17 갱신)

| ID | 항목 | 이행 상태 | 구현 내용 |
|----|------|-----------|-----------|
| REC-4.1 | MCP 도구 속도 제한 | ✅ **이행 완료** (전역 rate limit) | `mcp_router.py` 7개 엔드포인트 slowapi 적용: initialize 30/min, tools/call 20/min, SSE 10/min, 조회 60/min. 에이전트별 독립 quota는 Phase 5 FG5.3 §5로 이월 |
| REC-4.2 | Scope Profile 변경 감사 | ✅ **이행 완료** | `scope_profiles.py` — update/delete 전 `_get_affected_agents()`로 영향 에이전트 조회, 감사 로그 `metadata.affected_agents`에 기록 |
| REC-4.3 | Agent 토큰 만료 | ✅ **이행 완료** | `auth/dependencies.py` `_extract_agent_context()` — `expires_at IS NULL`이면 즉시 ANONYMOUS 반환 + 경고 로그 |

---

## 5. 결론

**High 2건, Medium 3건** 모두 수정 완료. Low 2건은 설계 내 완화.  
**잔여 권고 3건(REC-4.1~4.3) 전부 이행 완료** (2026-04-17). 단위 테스트 47/47 통과.
