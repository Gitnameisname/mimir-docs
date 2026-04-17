# Mimir Season 2 회고 (KPT)

| 항목 | 내용 |
|------|------|
| 작성일 | 2026-04-18 |
| 대상 | Season 2 전체 (Phase 0~9) |

---

## Keep (계속하기)

### K1. Feature Group 소분 + 독립 검수 게이트
Phase마다 FG를 명확히 분리하고 각 FG별 검수보고서·보안보고서를 의무화한 것이 품질 보증에 결정적이었다. Phase 9 진입 시점에 FG별 결함 0건을 달성한 것은 이 구조 덕분이다.

**S3 적용**: FG 단위를 더 세분화하되, 검수 자동화(테스트 게이트)로 수작업 문서 부하를 줄인다.

### K2. 테스트 우선 개발 (Test-First for Security)
보안 테스트를 Phase 9에 몰아서 작성하는 대신, Phase별로 보안 점검 항목을 테스트로 먼저 정의하면 더 효과적이었을 것이다. 그러나 Phase 9에서 일괄 작성 후 통과하는 방식도 충분히 효과적이었다.

**S3 적용**: 각 Phase 착수 전에 해당 Phase의 보안 테스트 케이스를 먼저 작성한다.

### K3. 폐쇄망 동등성 원칙 (S2 원칙 ⑦)
`embedding_service_url`, `llm_base_url` 환경변수로 외부 의존성을 완전히 격리한 설계가 테스트 환경에서도 빛났다. OpenAI API key 없이도 전체 테스트가 통과한다.

**S3 적용**: 신규 외부 의존성 추가 시 항상 환경변수 기반 on/off를 기본 설계로 채택한다.

### K4. Citation 5-tuple 계약
`(document_id, version_id, node_id, content_hash, span_offset)` 계약이 Phase 2부터 Phase 9까지 일관되게 유지되었다. SHA-256 해시 기반 무결성 검증 덕분에 Citation 회귀 테스트 100샘플이 전원 통과했다.

**S3 적용**: 외부 소스 커넥터에서도 동일한 Citation 계약을 준수하도록 어댑터를 설계한다.

### K5. MCP를 통한 에이전트-API 통합 분리
MCP 서버를 별도 라우터(`mcp_router.py`)로 분리한 것이 에이전트 API와 사용자 API의 명확한 경계를 만들었다. `_CURATED_TOOLS` allowlist 패턴은 에이전트가 호출할 수 있는 도구를 화이트리스트로 관리하는 강력한 방어선이다.

**S3 적용**: 신규 도구 추가 시 반드시 `_CURATED_TOOLS` 등록 → 코드 리뷰 → 보안 검토 프로세스를 거친다.

### K6. Scope Profile 기반 ACL (S2 원칙 ⑥)
코드에 `if scope == "team"` 같은 하드코딩이 단 한 건도 없다는 것을 서비스 레이어 전수 검사로 확인했다. Scope Profile이 데이터로 관리되므로 새로운 접근 범주를 코드 변경 없이 추가할 수 있다.

**S3 적용**: 외부 소스 커넥터의 접근 범위도 Scope Profile에 통합한다.

---

## Problem (개선하기)

### P1. 산출물 문서 작성 부하
Phase마다 검수보고서 + 보안취약점검사보고서 + (AI 기능 포함 시) AI품질평가보고서까지 요구되는 산출물이 많아 구현 대비 문서화 비율이 높았다.

**근본 원인**: S2 원칙에서 정의한 의무 산출물 규약이 엄격한 것이 원인이나, 품질 보증 관점에서 타당하다.

**개선 방안**: 검수보고서의 정형 항목을 테스트 결과에서 자동 생성하는 `reporting/`  모듈을 S3에서 완성한다. (`app/reporting/security_report_generator.py`의 확장)

### P2. 통합 테스트의 실 DB 의존성
`tests/integration/test_search_plugins.py` 등 일부 테스트가 `test_client` fixture(실 DB)에 의존하여 CI에서 선택적으로만 실행된다. Phase 0~8 통합 회귀는 mock 기반으로 작성해 100% 통과했으나, 실 DB 연동 검증은 별도 환경 필요하다.

**개선 방안**: S3에서 테스트 DB (Docker Compose, testcontainers) 인프라를 CI에 통합한다.

### P3. Prompt Injection 탐지 false positive 관리
룰 기반 탐지 40개 패턴이 공격을 잘 잡지만, 일부 정상 텍스트(예: 기술 문서에 "ignore formatting")에서 false positive가 발생할 수 있다. 현재 false positive rate 기준을 5% 미만으로 설정했으나, 운영 데이터 기반으로 검증이 필요하다.

**개선 방안**: 운영 로그에서 탐지 이벤트를 수집하고, 월별 패턴 재검토 프로세스를 도입한다.

### P4. UI 검수 5회 리뷰 부담
CLAUDE.md의 "UI 리뷰 최소 5회" 규칙이 품질을 높이지만, Admin UI와 User UI가 복잡해질수록 리뷰 비용이 증가한다.

**개선 방안**: Storybook + Chromatic을 이용한 시각적 회귀 테스트를 도입하여 수동 리뷰 회차를 줄인다.

### P5. Phase 간 직렬 의존성
Phase 4(MCP)가 Phase 3(Conversation)에 의존하고, Phase 5(Agent)가 Phase 4에 의존하는 직렬 구조로 인해 병렬 개발이 어려웠다. 특히 Phase 7(평가)과 Phase 8(추출)은 독립적으로 병렬 진행 가능했으나 직렬로 처리됐다.

**개선 방안**: S3 착수 전에 Phase 간 의존성 DAG를 명시적으로 작성하고, 독립 Phase는 병렬 착수한다.

---

## Try (시도하기)

### T1. 보안 테스트 Phase 분산 작성
Phase 9에 보안 테스트를 일괄 작성하는 대신, 각 Phase 개발 중에 해당 보안 케이스를 작성한다. 예: Phase 5(Agent Proposal) 개발 시 LLM08(Excessive Agency) 테스트를 함께 작성.

### T2. AI 품질 지표 사전 예산 확정
각 Phase 착수 시 해당 Phase의 AI 기능에 대한 품질 지표 임계값(`faithfulness ≥ 0.80` 등)을 먼저 정의하고, 개발 완료 후 측정한다. S2는 이 과정이 Phase 7에서 사후 정의되었다.

### T3. 추출 스키마 자동 제안 도구
ExtractionSchema를 사람이 수동 정의하는 것이 병목이다. LLM에게 문서 샘플을 주고 스키마를 초안 생성하는 `schema_suggester` 도구를 개발한다.

### T4. 에이전트 거절 사유 분석 자동화
Kill Switch 발동 및 에이전트 제안 거절 사유를 수집하고, 반복 패턴을 분석하여 프롬프트 개선으로 피드백한다. 현재는 감사 로그에만 기록되고 분석 파이프라인이 없다.

### T5. 폐쇄망 모델 품질 정기 벤치마크
`llm_base_url`로 연결하는 로컬 모델(vLLM, Ollama)의 품질이 OpenAI 대비 얼마나 차이 나는지를 GoldenSet으로 정기적으로 측정한다. 현재는 기능 동작 여부만 확인하고 품질 차이는 미측정이다.

### T6. Mimir API 버전 관리 (v1 안정화)
S3에서 외부 커넥터가 Mimir API를 호출할 때 하위 호환성을 보장하기 위해 `/api/v1` 엔드포인트를 안정 버전으로 선언하고, breaking change는 `/api/v2`로 격상한다.

---

## 종합 평가

S2는 **계획 대비 100% 달성**의 스프린트였다. 특히 보안(OWASP 20/20)과 S2 원칙 준수(⑤⑥⑦ 전면 적용)는 예상보다 높은 수준의 완성도를 보였다. 개선 영역은 주로 **문서화 자동화**와 **실 DB 통합 테스트 인프라**로, S3에서 우선적으로 해결할 과제로 이관한다.
