# FG5.3 보안취약점검사보고서
## 에이전트 감사 API / Rate Limit / 회귀 테스트

---

**작성일**: 2026-04-17  
**작성자**: 개발팀  
**검사 대상**: FG5.3 신규 구현 (감사 조회 API, 통계 API, Rate Limit)  
**검사 기준**: OWASP Top 10 Web (2021), OWASP LLM Top 10 (2025)  
**최종 등급**: 🟢 **PASS** — High/Critical 취약점 없음, 권고사항 3건

---

## 1. 검사 요약

| 심각도 | 건수 |
|--------|------|
| Critical | 0 |
| High | 0 |
| Medium | 0 |
| Low | 0 |
| 권고 (Info) | 3 |

---

## 2. 권고사항

### REC-5.3.1 [권고] 감사 로그 조회 결과 페이지 크기 상한 확인

**위치**: `GET /admin/agents/{id}/audit` — `page_size` 파라미터  
**현황**: 최대 100으로 제한 (`page_size: int = Query(20, ge=1, le=100)`)  
**평가**: 현재 구현은 안전 (le=100).  
**권고**: 대용량 조회 시 응답 크기가 커질 수 있으므로, 향후 응답 압축(gzip) 적용 고려.  
**조치**: 향후 Phase 6 배포 시 nginx gzip 설정 검토.

---

### REC-5.3.2 [권고] Rate Limit 카운터 키에 agent_id 직접 사용

**위치**: `mcp_router.py` — `_check_agent_rate_limit()`  
**현황**: Valkey 키 패턴: `agent:{agent_id}:rate:{endpoint}`  
**평가**: agent_id가 UUID이므로 충분한 엔트로피 보유. 직접 키 사용은 허용 가능.  
**권고**: agent_id가 외부에서 통제 가능한 값이라면 키 해시 적용 고려 (`SHA256(agent_id)`). 현재 UUID는 서버에서 생성하므로 위험 없음.  
**조치**: 현재 안전. UUID 유지.

---

### REC-5.3.3 [권고] 에이전트 통계 API — 에이전트 존재하지 않을 때 빈 응답 반환

**위치**: `GET /admin/agents/{id}/statistics`  
**현황**: 에이전트가 DB에 없어도 통계 쿼리 수행 후 0 값 반환.  
**평가**: 정보 노출 위험 없음 (카운트가 0이므로). 단, 에이전트 존재 유무를 유추 가능.  
**권고**: 없는 에이전트 조회 시 404 반환 여부를 정책적으로 결정할 것. 현재 구현은 0 반환으로 정보 노출이 최소화되어 있어 허용 가능.  
**조치**: 현재 구현 유지. 향후 정책 결정 후 변경 가능.

---

## 3. 기존 보안 통제 확인

| 통제 항목 | 구현 여부 | 비고 |
|----------|---------|------|
| 모든 API admin 역할 필수 (`_require_admin`) | ✅ | ORG_ADMIN / SUPER_ADMIN만 접근 |
| JWT 인증 필수 (`resolve_current_actor`) | ✅ | Bearer JWT 또는 X-API-Key |
| SQL Injection 방지 (파라미터화된 쿼리) | ✅ | `%s` 바인딩 사용 |
| Rate Limit fail-open (Valkey 장애 시 통과) | ✅ | S2 원칙 ⑦ — 폐쇄망 환경 동작 |
| 감사 로그 actor_type 기록 | ✅ | S2 원칙 ⑥ |
| Valkey 키 TTL 60초 자동 만료 | ✅ | 메모리 누수 방지 |
| 에이전트 ID 유효성 검사 (UUID 형식 강제 아님) | ⚠️ | 경로 파라미터 문자열 — DB 쿼리에서 안전하게 처리됨 |

---

## 4. OWASP LLM Top 10 추가 검사 (FG5.2 포함)

| OWASP LLM 항목 | 상태 |
|----------------|------|
| LLM01: Prompt Injection | ✅ 방어 구현 완료 (FG5.2 규칙 기반 탐지 + 콘텐츠-지시 분리) |
| LLM02: Insecure Output Handling | ✅ 에이전트 응답은 항상 `trusted=false`, `injection_risk` 명시 |
| LLM06: Sensitive Information Disclosure | ✅ 감사 로그에 토큰/PII 미포함, 최소 메타데이터만 기록 |
| LLM08: Excessive Agency | ✅ 에이전트는 proposed 상태로만 쓰기 가능, 인간 승인 필수 |
| LLM09: Overreliance | ✅ UI에서 proposed 상태임을 명확히 표시, 인간 검토 강제 |

---

## 5. 결론

FG5.3 구현에서 Critical/High/Medium 취약점은 발견되지 않았습니다.  
권고사항 3건은 모두 정책 결정이 필요한 낮은 우선순위 사항으로, 현재 구현은 안전합니다.

Phase 5 (FG5.1 + FG5.2 + FG5.3) 전체 보안 검사를 통과하였으며, Phase 6 개발을 진행할 수 있습니다.
