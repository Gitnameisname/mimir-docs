# AI 품질평가 보류 사유서 — Phase 5 (Agent Action Plane)

**작성일**: 2026-04-18
**작성자**: Claude Opus 4.7 (S2 전체 재검수 후속)
**대상 Phase**: S2 Phase 5 — Agent Action Plane
**상위 근거**: `docs/개발문서/S2_종결보정서_20260418.md` §2 (F-02 AI 품질 기준 달성 선언 철회)
**상태**: **보류 (Deferred to S3 P0)**

---

## 1. 본 문서의 위치

본 문서는 「AI 품질평가 **보고서**」가 아니다. Phase 5 는 S2 의 하이라이트 Phase 로서 "AI 에이전트가 안전하게 쓰기를 수행하는" 계층을 구축했다. 에이전트가 만든 모든 제안의 품질은 **인간 승인 게이트** 와 맞물려 있으므로, 품질 측정은 단순한 RAG 지표로 환원되지 않는다.

그러나 `CLAUDE.md` 규약은 "AI 기능 포함 FG" 에 대해 AI품질평가보고서 제출을 의무화하고 있으며, S2 종결 시점에 Phase 5 의 정량 평가 자료는 전무하다. 본 문서는 **왜 Phase 5 AI 품질을 지금 실측하지 않았는지**, **언제·어떻게 측정할 것인지** 를 공식 기록한다.

---

## 2. Phase 5 의 AI 관련 기능 범위

품질평가가 필요한 Phase 5 의 기능은 다음과 같다.

| 기능 | 코드 위치 | 평가가 필요한 품질 축 |
|------|-----------|----------------------|
| 에이전트 쓰기 → `proposed` Draft 한정 | `backend/app/services/agents/proposal_service.py` | 제안 정확도(= 인간 승인률), 반려 사유 분포 |
| 워크플로 상태 전이 제안 API | `backend/app/api/v1/agents/` | 잘못된 전이 제안 비율 |
| 컨텐츠-지시 분리 정책 (프롬프트 인젝션 방어) | `backend/app/services/agents/prompt_isolation.py` | 주입 방어율, 오탐율(정상 지시를 거부한 비율) |
| MCP Tasks 비동기 승인 플로우 | `backend/app/services/mcp/tasks/` | 완료율, 타임아웃률, dead-letter 재처리율 |
| 에이전트 감사 API / 행동 로그 | `backend/app/api/v1/agent_audit_router.py` | 감사 이벤트 누락률, `actor_type` 기록률 |

---

## 3. 보류 사유

### 3.1 필수 전제 자원 미확보

1. **에이전트 제안 골든셋 미구축**: Phase 5 평가의 핵심은 "에이전트가 제안한 Draft 가 인간 기준으로 수용 가능한가" 이다. 이는 (입력 지시, 소스 문서, 기대 Draft) 의 3-튜플 골든셋을 요구하며 현재 **0건** 이다.
2. **LLM 판정 backend 미상시 가동**: 제안 품질의 자동 판정은 LLM-as-a-Judge 를 사용하며, closed-network fallback 모델의 실운영 채널이 확보되지 않은 상태이다.
3. **프롬프트 인젝션 공격 시나리오 셋 미구축**: 분리 정책의 방어율을 측정하려면 표준 공격 프롬프트(Prompt Injection Benchmark 계열) 가 필요한데, 저장소 내 `tests/security/prompt_injection/` 에 공식 공격 셋이 0건이다.
4. **실운영 제안·승인·반려 로그 부재**: S2 종결 시점 기준 `audit_events` 테이블의 agent 관련 row 는 개발 smoke test 수준이다. 통계적으로 유의미한 샘플이 누적되지 않았다.

### 3.2 현재 시점에서 **검증된 것** (단위/통합 테스트 범위)

| 검증 항목 | 수단 | 상태 |
|-----------|------|------|
| 에이전트가 생성한 Draft 는 반드시 `proposed` 로 진입 | 통합 테스트 (status 검증) | ✅ |
| `proposed → published` 직접 전이 금지 (`approved` 경유 필수) | 통합 테스트 | ✅ |
| 감사 이벤트 `actor_type="agent"` 필수 기록 (F-07 시정 완료) | 타입 레벨 + 단위 테스트 | ✅ |
| 검색 결과(untrusted) 가 system prompt 와 분리된 user content 로만 삽입 | 코드 정적 분석 + 단위 테스트 | ✅ |
| MCP Tasks dead-letter 재처리 경로 | 단위 테스트 | ✅ |

이들은 **구조적 안전성** 이지 **LLM 품질 지표(Faithfulness / 승인률 등)** 가 아니다. 본 보류 사유서는 두 개념을 섞지 않는다.

### 3.3 Phase 5 특유의 평가 지표 미정의

Phase 5 는 일반적 RAG 지표(Faithfulness 등) 만으로는 충분히 평가되지 않는다. 다음 고유 지표가 정의·임계값화 되어야 하나 `evaluation_thresholds.yaml` 에 아직 반영되지 않았다.

- **Human-Approval Rate (HAR)**: 승인 / (승인 + 반려) — 기대 임계값 후보: ≥ 0.75
- **Injection Defense Rate**: 방어 / (방어 + 누락) — 기대 임계값 후보: ≥ 0.95
- **Mis-route Rate**: 잘못된 상태 전이 제안 / 전체 전이 제안 — 기대 임계값 후보: ≤ 0.05
- **Agent Audit Coverage**: agent actor_type 기록률 — F-07 후 ≥ 0.99 가 목표

위 지표의 공식 편입 자체도 S3 P0 이월 과제이다.

---

## 4. 실측 계획 (S3 P0 이월)

| 단계 | 내용 | 산출물 | 담당 | 기한 |
|------|------|--------|------|------|
| 1 | Phase 5 전용 평가 셋 구축 — 제안 골든셋 30건 + injection 공격 셋 50건 | `golden_sets/phase5_agent_proposal.json`, `tests/security/prompt_injection/*.yaml` | 지식팀 + 보안팀 | S3 착수 + 3주 |
| 2 | `evaluation_thresholds.yaml` 에 HAR / Defense / Mis-route / Audit Coverage 네 축 추가 | YAML 반영 + 문서화 | 개발팀 | 1단계와 병렬 |
| 3 | 평가 러너 확장(`scripts/evaluate_agent_proposals.py`) — Phase 7 의 golden set runner 재사용 | 스크립트 | 개발팀 | 2단계 완료 직후 |
| 4 | 실측 run 1회 → `evaluation_reports/phase5_agent_<date>.json` | 메타데이터 + 본 문서를 **AI품질평가보고서_Phase5.md** 로 승격 | 개발팀 | 3단계 직후 |
| 5 | 미달 지표에 대해 PromptBuilder / 분리 정책 / 워크플로 룰 조정 후 재측정 | 동일 | 개발팀 | 4단계 결과에 따라 |

---

## 5. 수치 선언 금지 사항

본 보류가 해제되기 전까지 다음 문구는 **어떤 문서에도 기재되어서는 안 된다**.

- "Phase 5 는 에이전트 제안 승인률 X.XX 를 달성했다"
- "Phase 5 는 프롬프트 인젝션 방어율 X.XX 이다"
- "Phase 5 는 AI 품질 기준을 충족한다"

S2 종결보고서의 동등한 선언은 S2_종결보정서_20260418.md §2.3 에서 **공식 철회** 되었다.

---

## 6. 서명

| 역할 | 이름 | 날짜 |
|------|------|------|
| 재검수 주관 | Claude Opus 4.7 | 2026-04-18 |
| 기술 리드 | (서명 대기 — S3 P0 착수 시) | |
