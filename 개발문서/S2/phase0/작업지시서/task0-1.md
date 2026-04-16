# Task 0-1. S2 추가 원칙(⑤⑥⑦) 공식 문서화 및 CLAUDE.md 업데이트

## 1. 작업 목적

S2의 핵심 설계 원칙 3개(AI-as-first-class-consumer, 접근 범위 하드코딩 금지, 폐쇄망 동등성)를 공식 문서화하고, 이를 프로젝트 절대 규칙으로 확정함으로써 모든 팀원이 동일한 기준 하에 개발할 수 있도록 한다.

구체적으로는:
- 원칙 ⑤, ⑥, ⑦의 함의와 구현 기준을 명확히 기술
- CLAUDE.md §2 "절대 규칙"에 새로운 조항 추가
- Season 2 개발계획.md와의 대응 관계를 명시하여 일관성 확보

---

## 2. 작업 범위

### 포함 범위

1. **원칙 ⑤ "AI는 1급 사용자다 (AI-as-first-class-consumer)"** 문서화
   - 정의: 모든 기능은 사람 UI 이전에 AI 에이전트가 안전하게 호출·검증·기록할 수 있어야 한다
   - 함의: API 우선 설계, 구조화된 응답 표준화, 감시 로그의 필수 포함
   - 구현 영역: 인증, 권한, 응답 포맷, 감사, 오류 처리

2. **원칙 ⑥ "접근 범위도 하드코딩 금지"** 문서화
   - 정의: 외부 소비자의 scope 어휘는 Scope Profile(관리 설정)으로 관리하며, Mimir 코드에 특정 scope 문자열이 등장하지 않는다
   - 함의: Scope Profile 기반 ACL 필터링, 동적 scope 매핑, 정책 변경 시 코드 수정 불필요
   - 구현 영역: API Key 관리, Scope Profile CRUD, FilterExpression 엔진

3. **원칙 ⑦ "폐쇄망 동등성"** 문서화
   - 정의: 모든 기능은 외부 네트워크 없이 동작 가능해야 한다. 외부 의존(OpenAI, SaaS 등)은 환경변수로 off 가능하고, off 시에도 기능이 degrade될 뿐 실패하지 않는다
   - 함의: 로컬 모델 1급 지원, 부분 실패 처리, 모드 문서화
   - 구현 영역: Provider 추상화, 환경변수 게이트, 회귀 테스트 매트릭스

4. **CLAUDE.md §2 절대 규칙 업데이트**
   - 기존 규칙(문서 타입 하드코딩 금지, 구조 generic+config 기반 등)과 새 규칙의 조화
   - 원칙 ⑤⑥⑦에 대한 코드 레벨 적용 규칙 추가

### 제외 범위

- 원칙의 구현 코드 작성
- Phase 1 이후 구현 작업 (문서화만 수행)
- 외부 소비자 커뮤니케이션 자료 (별도 Phase에서 담당)

---

## 3. 선행 조건

- Season 2 개발계획.md 검토 완료 (§1, §3)
- S1 완수 보고서 및 회고 검토
- Phase 0 개발계획서 검토 완료
- 프로젝트 팀 회의에서 원칙 ①~⑦에 대한 합의 기록

---

## 4. 주요 작업 항목

### 4-1. 원칙 ⑤ "AI는 1급 사용자다" 상세 문서화

**§ 정의와 배경**
- S1의 API-first 원칙을 AI 소비자 관점으로 확장
- 챗봇(1차 외부 소비자)의 요구사항 반영
- "사람과 AI 모두 동일한 API 계약에 접근 가능"

**§ 구현 기준**
- 모든 기능은 REST API 또는 MCP를 통해 먼저 정의
- 구조화된 요청/응답 스키마 필수 (OpenAPI, MCP Tool schema)
- 응답에 meta 필드 포함: `trusted`, `source`, `provenance`
- 에러 응답도 표준화 (code, message, recoverable)

**§ 적용 범위**
- 검색/조회 API (모든 FG에서)
- 문서 쓰기 API (Phase 5에서)
- 워크플로 전이 제안 API (Phase 5에서)
- 평가 실행 API (Phase 7에서)

**§ 반례 및 회피 기준**
- "User UI에만 있는 기능" 금지
- 페이지에만 있는 로직 금지 (반드시 API로 추상화)
- 감사 로그에 AI 에이전트의 행동이 명확히 기록되지 않는 경우 금지

### 4-2. 원칙 ⑥ "접근 범위도 하드코딩 금지" 상세 문서화

**§ 정의와 동기**
- S1의 "문서 타입은 하드코딩 금지" 규칙을 접근 제어로 확장
- 외부 소비자의 scope 어휘가 변할 때 코드 수정 불필요하게 함
- 관리자가 설정 UI에서 직접 정책 정의 가능

**§ Scope Profile 구조**
```
ScopeProfile
  ├─ name: "기본 챗봇용" (관리자 설정)
  └─ ScopeDefinition[]
       ├─ scope_name: "team" (외부 소비자가 요청)
       └─ acl_filter: FilterExpression (동적 매핑)
```

**§ FilterExpression 규칙** (S2 수위)
- 미리 정의된 필터 필드만 사용 (whitelist)
  - `organization_id`, `team_id`, `visibility`, `classification`, `owner_id` 등
- 연산자: `eq`, `neq`, `in`, `not_in`, `starts_with` (SQL-like 표현식 불가)
- `$ctx.*` 변수로 access_context 동적 치환
- 예: `{and: [{field: "organization_id", op: "eq", value: "$ctx.organization_id"}]}`

**§ 코드 금지 목록**
- `if scope == "team"`: 금지
- `if user.org_id == "acme"`: 금지 (조직별 하드코딩)
- 환경변수에 scope 문자열 고정: 금지

**§ 의무 적용 계층**
- API 인증 레이어 (client-credentials 검증 후 Scope Profile 로드)
- ACL 필터링 (모든 검색, 조회, 쓰기 API에 AND 합성)
- RAG 파이프라인 (벡터 검색 + FTS 모두 적용)

### 4-3. 원칙 ⑦ "폐쇄망 동등성" 상세 문서화

**§ 정의와 배경**
- 모든 기능은 격리된 환경(폐쇄망, 자체 호스팅)에서 완전히 동작 가능
- 외부 의존(OpenAI, LangSmith, OTel Cloud)은 선택적
- Degrade mode: 기능이 줄지만 서비스는 계속 동작

**§ 환경변수 기반 모드 전환**
- `EXTERNAL_API_ENABLED=true/false` (기본 false)
- `OPENAI_API_KEY` 없으면 로컬 모델로 자동 전환
- `ENABLE_CLOUD_TRACING=true/false` (기본 false)

**§ 각 영역의 Degrade 정책**
- **LLM 기능**: OpenAI 불가능 → vLLM/Ollama/BGE로 자동 전환
- **임베딩**: OpenAI embedding 불가능 → BGE-M3/E5 로컬 모델 사용
- **평가**: Faithfulness (판정 LLM) 불가능 → Citation-present, 원문 매칭만 수행
- **모니터링**: Cloud trace 불가능 → 로컬 로그만 남김
- **RAG**: 벡터 db 장애 → FTS만 사용하여 계속 검색

**§ 테스트 매트릭스** (CI에서 의무)
- Matrix 1: 모든 외부 API 활성 (기본 개발 환경)
- Matrix 2: 모든 외부 API 비활성 (폐쇄망 환경)
- 양쪽 매트릭스 모두 기본 CRUD, 검색, 인증 통과 필수

**§ 문서화 기준**
- 각 Feature에 "폐쇄망 모드에서의 동작" 섹션 필수
- 예: "RAG 검색 장애 시, FTS 검색으로 자동 폴백하며 사용자에게 '제한 모드'를 알림"

### 4-4. CLAUDE.md §2 절대 규칙 추가

**기존 규칙 (유지)**
```
- 문서 타입은 하드코딩 금지
- 구조는 generic + config 기반
- JSON 필드 남용은 허용하지만 schema는 관리
- 모든 로직은 "type-aware" 해야 함
```

**S2 신규 규칙 (추가)**
```
- 접근 범위도 하드코딩 금지: Scope Profile 기반 ACL 필터링 필수
- AI 에이전트는 사람과 동등한 API 소비자 — 감사 로그에 actor_type 필수 기록
- 폐쇄망 환경에서 전체 기능 동작: 환경변수로 외부 의존 on/off 가능하고, 
  off 시에도 서비스는 degrade하지만 실패하지 않음
- 원칙 ①~④의 기존 규칙과의 모순 발생 시 S1 원칙이 우선하고, 
  S2 신규 규칙은 그 위에서 강화
```

---

## 5. 산출물

1. **원칙 ⑤ 상세 문서** (`S2_원칙_⑤_AI는_1급_사용자다.md`)
   - 200~250 단어, 정의, 함의, 구현 기준, 적용 범위, 반례

2. **원칙 ⑥ 상세 문서** (`S2_원칙_⑥_접근범위_하드코딩금지.md`)
   - 250~300 단어, Scope Profile 구조, FilterExpression 규칙, 코드 금지 목록

3. **원칙 ⑦ 상세 문서** (`S2_원칙_⑦_폐쇄망_동등성.md`)
   - 250~300 단어, 환경변수 모드, 각 영역 degrade 정책, 테스트 매트릭스

4. **CLAUDE.md 업데이트본** 
   - 기존 내용 유지, §2 절대 규칙에 신규 규칙 4개 추가
   - Season 2 개발계획 §3.1~3.6과의 대응 명시

5. **원칙 정합성 매트릭스** 
   - S1 원칙 ①④와 S2 원칙 ⑤⑦의 중복, 모순, 강화 관계 정리

---

## 6. 완료 기준

1. 원칙 ⑤⑥⑦ 각각에 대해 100 단어 이상의 상세 설명 작성
2. CLAUDE.md에 신규 규칙 4개 추가되고, 기존 규칙과의 관계 명시
3. Season 2 개발계획.md §3.1~3.6과의 대응 관계 표로 정리
4. 팀 전체 리뷰 완료 및 동의 확인 (최소 2명 리뷰)
5. 각 규칙에 대해 "이 규칙을 위반하는 코드 예시"를 1~2개씩 제시
6. Phase 1~9의 각 FG에서 이 규칙을 적용하는 체크포인트 정의

---

## 7. 작업 지침

### 지침 7-1. 문서 작성 스타일

- 한국어로 작성, 간결하고 구체적인 표현 사용
- 추상적 설명보다는 "코드에서 금지되는 패턴", "명시적 체크포인트" 중심
- 각 원칙마다 "현장에서의 의문" FAQ 섹션 추가

### 지침 7-2. 대응 관계 문서화

Season 2 개발계획.md와 이 Task의 산출물 간 대응:

| Season 2 개발계획 섹션 | 이 Task의 산출물 |
|----------------------|-----------------|
| §3.1 OAuth + access_context | 원칙 ⑤ (1급 소비자) |
| §3.2 Scope Profile | 원칙 ⑥ (범위 하드코딩 금지) |
| §3.3 MCP 2025-11-25 | 원칙 ⑤ (API-first) |
| §3.5 Agent Principal | 원칙 ⑤ (감사 로그) |
| §3.6 평가용 판정 전략 | 원칙 ⑦ (폐쇄망 판정 LLM) |
| §2.2 폐쇄망 배포 | 원칙 ⑦ (폐쇄망 동등성) |

### 지침 7-3. 검토 체크리스트

완성 전 다음 항목 확인:

- [ ] 원칙 ⑤⑥⑦이 상호 배타적인가? (각자의 영역이 명확한가)
- [ ] CLAUDE.md의 신규 규칙이 S1 원칙과 모순하지 않는가?
- [ ] 각 원칙에 대해 "Phase 몇에서 이 규칙이 검수되는가" 명시됨?
- [ ] Degrade mode 정책이 명확하여 Phase 7의 평가 기준과 일관되는가?
- [ ] Scope Profile과 FilterExpression의 설명이 Phase 4 FG4.2와 정합되는가?

---

## 8. 참고 자료

- Season 2 개발계획.md §1 (설계 원칙), §3 (핵심 설계 결정)
- Phase 0 개발계획서 §2 (Phase 0의 역할)
- S1 회고 (원칙 ①④의 구현 경험)
- MCP Specification 2025-11-25 (원칙 ⑤의 근거, OAuth client-credentials)
- OWASP Top 10 for LLM Applications (원칙 ⑦의 prompt injection 방어 관련)
