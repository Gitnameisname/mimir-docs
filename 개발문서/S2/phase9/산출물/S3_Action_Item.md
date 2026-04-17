# S3 Action Item — Mimir Season 3 착수 준비

| 항목 | 내용 |
|------|------|
| 작성일 | 2026-04-18 |
| 기반 | S2 완수 보고서 + 회고 KPT |

---

## 우선순위 분류 기준

| 등급 | 기준 |
|------|------|
| P0 — Critical | S3 착수 전 반드시 해결해야 하는 항목 (S2 기반 안정화) |
| P1 — High | S3 초기(Phase 1~3)에 구현해야 하는 핵심 기능 |
| P2 — Medium | S3 중반(Phase 4~7)에 구현하는 확장 기능 |
| P3 — Low | S3 후반 또는 선택적 구현 항목 |

---

## P0 — Critical (S3 착수 전 해결)

### P0-1. 실 DB 통합 테스트 인프라 구축
**배경**: `tests/integration/test_search_plugins.py` 등이 실 DB 없이 실행 불가. CI에서 DB 연동 테스트가 누락되어 있다.

**Action**:
- Docker Compose 기반 테스트 DB 환경 구성 (`docker-compose.test.yml`)
- GitHub Actions에 PostgreSQL + Valkey 서비스 컨테이너 추가
- `test_client` fixture를 실 DB에 연결하는 conftest 작성

**완료 기준**: `pytest -m integration` 통과 (CI 포함)

### P0-2. Mimir API v1 안정 버전 선언
**배경**: S3 외부 커넥터 및 챗봇이 `/api/v1` 엔드포인트를 호출한다. Breaking change 없이 안정성을 보장해야 한다.

**Action**:
- `/api/v1` 엔드포인트 전수 목록 및 OpenAPI 스펙 Export
- Breaking change 정책 문서 작성 (semver 준용)
- 하위 호환성 테스트 스위트 추가 (`tests/compatibility/`)

**완료 기준**: OpenAPI 스펙 `docs/api/v1-spec.yaml` 생성 + 호환성 테스트 통과

---

## P1 — High (S3 Phase 1~3 핵심)

### P1-1. 외부 소스 커넥터 프레임워크
**배경**: S2는 Mimir 내부 문서만 처리. S3에서는 외부 지식 소스를 수집해야 한다.

**Action**:
- `app/connectors/` 디렉터리 신설, 플러그인 인터페이스 정의
- 우선 구현 순서: **GitLab Wiki** → **Confluence** → **Notion** → SharePoint
- 각 커넥터: OAuth/API Key 기반 인증, 페이지 → 청크 변환, Citation 5-tuple 생성

**완료 기준**: GitLab Wiki 커넥터 동작, Citation 무결성 검증 통과

### P1-2. 멀티테넌시 지원
**배경**: S2는 단일 조직 가정. S3에서는 여러 조직이 Mimir를 공유한다.

**Action**:
- `organization_id` 기반 데이터 격리 (Row-Level Security)
- 테넌트별 Scope Profile, 테넌트별 LLM 설정
- 테넌트 간 데이터 누수 방지 테스트

**완료 기준**: 2개 테넌트 동시 운영 시 데이터 격리 확인

### P1-3. 산출물 보고서 자동 생성 완성
**배경**: `app/reporting/security_report_generator.py`가 시작되었으나 부분 구현.

**Action**:
- 검수보고서 자동 생성: 테스트 결과 JSON → Markdown 변환
- CI에서 PR마다 보고서 자동 생성 + 아티팩트 업로드
- 보안취약점검사보고서도 템플릿 기반 자동화

**완료 기준**: CI에서 Markdown 보고서 자동 생성 확인

### P1-4. 챗봇 MCP 클라이언트 개발
**배경**: S2에서 MCP 서버를 구현했으나, S3 챗봇이 이를 소비하는 클라이언트가 필요하다.

**Action**:
- MCP 2025-11-25 스펙 기반 Python 클라이언트 라이브러리
- `initialize` → `tools/list` → `tools/call` 플로우 구현
- 에러 처리 + 재시도 (exponential backoff)
- 폐쇄망 환경 테스트

**완료 기준**: 챗봇이 `search_documents` 호출 후 Citation 포함 응답 수신

---

## P2 — Medium (S3 Phase 4~7 확장)

### P2-1. 다국어 지원
**배경**: 현재 PII 탐지, 임베딩, Prompt Injection 탐지가 한국어/영어 중심.

**Action**:
- 다국어 임베딩 모델로 전환 (`multilingual-e5-large` 등)
- PII 패턴을 다국어로 확장 (일본어 주민번호, 유럽 IBAN 등)
- Prompt Injection 탐지 패턴 다국어 추가

### P2-2. AI-Native 계획 도구 개발
**배경**: S3 에이전트가 Mimir를 이용해 복잡한 지식 관리 계획을 세우려면 계획 전용 도구가 필요하다.

**Action**:
- `plan_knowledge_graph`: 문서 간 관계 분석 → 계획 제안
- `summarize_project_state`: 프로젝트 문서 상태 요약
- `draft_action_plan`: 목표 기반 액션 플랜 초안 생성 (Draft 제안으로 등록)

### P2-3. 추출 스키마 자동 제안 도구
**배경**: P3에서 식별한 병목 — 스키마 수동 정의.

**Action**:
- LLM에게 문서 샘플 10개를 주면 ExtractionSchema 초안을 JSON으로 생성
- 관리자 UI에서 초안 검토 → 확정 플로우
- GoldenSet 기반 스키마 품질 자동 평가

### P2-4. 지식 그래프 시각화
**배경**: 문서 간 인용·참조 관계가 복잡해질 경우 시각화가 필요하다.

**Action**:
- D3.js 또는 React Flow 기반 노드 관계도
- 문서 → Citation → 다른 문서 연결 그래프
- 관리자 UI에 "지식 맵" 탭 추가

### P2-5. 폐쇄망 모델 품질 정기 벤치마크
**배경**: `llm_base_url` 로컬 모델의 품질이 미측정.

**Action**:
- GoldenSet 기반 OpenAI vs 로컬 모델 품질 비교 CI 잡
- 월 1회 자동 벤치마크 + Slack/이메일 알림
- 품질 임계값(`faithfulness ≥ 0.80`) 미달 시 경고

---

## P3 — Low (S3 후반 / 선택적)

### P3-1. 공동 편집 (Collaborative Editing)
- WebSocket 기반 실시간 동시 편집
- Operational Transform 또는 CRDT 기반 충돌 해결

### P3-2. 고급 FilterExpression (SQL-like)
- 현재: 기본 필터(문서 타입, 날짜 범위)
- 목표: `status = 'published' AND created_by IN ('user-1', 'user-2')`

### P3-3. 에이전트 거절 사유 자동 분석
- Kill Switch 발동 / 제안 거절 로그 → 패턴 분석 파이프라인
- 주간 리포트로 프롬프트 개선 제안

### P3-4. SharePoint / Jira 커넥터
- P1-1의 GitLab/Confluence 이후 추가 커넥터

---

## S3 착수 전 체크리스트

```
[ ] P0-1: Docker Compose 테스트 DB 인프라 구성
[ ] P0-2: API v1 OpenAPI 스펙 Export + 호환성 테스트
[ ] 챗봇 팀에 인계 메모 v3 전달
[ ] MCP 운영 가이드 작성 (mcp_integration_guide.md 업데이트)
[ ] S2 완수 보고서 팀 공유 + 회고 미팅
[ ] S3 Phase 1 개발계획서 작성
```

---

## S3 예상 Phase 구성 (초안)

| Phase | 제목 | 핵심 기능 |
|-------|------|-----------|
| S3-0 | 인프라 안정화 | 실 DB 통합 테스트, API v1 안정화, 보고서 자동화 |
| S3-1 | 외부 커넥터 | GitLab Wiki, Confluence 커넥터 |
| S3-2 | 멀티테넌시 | 조직 격리, 테넌트별 설정 |
| S3-3 | MCP 클라이언트 | 챗봇 연동, AI-native 계획 도구 |
| S3-4 | 다국어 지원 | 임베딩, PII, Injection 탐지 다국어 확장 |
| S3-5 | 고급 UI | 지식 그래프 시각화, 공동 편집 |
| S3-6 | 운영 최적화 | 벤치마크 자동화, 폐쇄망 모델 품질 관리 |
| S3-7 | 통합·종결 게이트 | S3 OWASP 재점검, 통합 회귀, S4 인계 |
