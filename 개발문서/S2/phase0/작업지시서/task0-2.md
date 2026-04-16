# Task 0-2. 핵심 설계 결정(3.1~3.6) 상세 검토 및 확정

## 1. 작업 목적

Season 2 개발계획.md §3에 제시된 6가지 핵심 설계 결정(3.1~3.6)을 팀 전체 리뷰를 통해 상세히 검토하고, Phase 1 착수 전 공식 확정함으로써 이후 모든 기술 결정의 기반을 마련한다.

각 결정에 대해:
- 배경 및 근거 명확화
- 대안 비교 및 트레이드오프 문서화
- Phase별 구현 시점과 책임 명시
- 팀 전체 동의 및 서명

---

## 2. 작업 범위

### 포함 범위

1. **3.1 에이전트 인증 — OAuth client-credentials + access_context**
   - 챗봇 ↔ Mimir 서버 간 인증 방식 확정
   - client-credentials 플로우의 보안 검토
   - access_context 검증 정책 (재검증 vs 신뢰)
   - Phase 4 구현 기준 정의

2. **3.2 Scope Profile + FilterExpression + $ctx.* 변수 모델**
   - Scope Profile의 설계 구조 (1대다 관계, 변경 전파)
   - FilterExpression의 연산자, 필드 범위 (S2 수위)
   - $ctx.* 변수의 바인딩, 주입 공격 방어
   - Phase 4 구현 기준 정의

3. **3.3 MCP 스펙 2025-11-25 핀 확정**
   - 이 스펙 선택 근거 (기존 스펙 vs 2025-11-25)
   - Mimir가 활용하는 MCP 기능 목록 (Tools, Resources, Prompts, Tasks, OAuth)
   - 호환 정책 (스펙 업그레이드 규칙)
   - Phase 4 구현 기준 정의

4. **3.4 Citation 5-tuple {document_id, version_id, node_id, span_offset, content_hash} 계약**
   - 각 필드의 검증 가능성 설명
   - span_offset의 생략 규칙
   - content_hash 알고리즘 선택 (SHA-256 vs MD5)
   - Phase 2에서의 구현 기준 정의

5. **3.5 Agent Principal 모델 (독립 주체, delegation, kill switch)**
   - Agent를 사용자와 별개의 Principal로 모델링하는 이유
   - Delegation의 정의 및 ACL 적용 방식
   - Kill switch의 트리거 조건 및 차단 지연 시간
   - Phase 4~5에서의 구현 기준 정의

6. **3.6 평가용 판정 전략 (LLM 의존도 낮은 것부터 시작)**
   - Citation-present check (규칙 기반)
   - 근거 문장 ↔ 원문 매칭 (content_hash / 유사도)
   - Answer relevance (embedding 기반)
   - Faithfulness, Context precision/recall (LLM 판정, 모델 스왑 가능)
   - Phase 7에서의 구현 기준 정의

### 제외 범위

- 실제 구현 코드 작성
- Phase 1 이후 상세 설계 (이 Task는 고수위 확정만 수행)
- 외부 이해관계자 리뷰 (팀 내부만)

---

## 3. 선행 조건

- Season 2 개발계획.md 전체 검토 완료
- S1 완수 보고서 및 회고 검토
- MCP Specification 2025-11-25 최소 1회 읽음
- S1 인증/권한/검색 구현 구조 이해

---

## 4. 주요 작업 항목

### 4-1. 3.1 OAuth client-credentials + access_context 검토·확정

**§ 배경**
- S1 회고에서 "AI 에이전트 인증 계획 부재"로 지적
- 챗봇(외부 프로젝트)이 1차 소비자로서 서버-투-서버 호출 필요
- OAuth 2.0 client-credentials: 서버 간 인증의 표준

**§ 설계 원칙**
- Mimir는 access_context를 그대로 신뢰하지 않는다
- 요청에 포함된 `user_id`를 DB 조회하여 실제 권한을 재검증
- API Key의 scope에 `delegate:search`, `delegate:write` 명시

**§ 인증 흐름**
```
[1] 챗봇 서버: client_id + client_secret으로 토큰 요청
    POST /oauth/token (grant_type=client_credentials)
    ↓ (Mimir가 검증)
[2] Mimir: 유효한 API Key인가? → 토큰 발급
    ↓
[3] 챗봇 서버: 토큰 + access_context를 body에 포함하여 검색 요청
    POST /api/v1/search {query, access_context: {user_id, org_id, permissions}}
    ↓ (Mimir가 재검증)
[4] Mimir: 
    - 토큰의 delegate:search 권한 확인
    - access_context.user_id → DB 조회 → 실제 ACL 적용
    - 검색 결과에 ACL 필터 적용
```

**§ 대안 비교**
| 옵션 | 장점 | 단점 | S2 선택 |
|------|------|------|--------|
| OAuth client-credentials | 표준, 확장성 | 구현 복잡도 ↑ | ✅ |
| API Key만 사용 | 간단 | 대행 권한 제어 어려움 | ✗ |
| mTLS | 보안 강함 | 인증서 관리 복잡 | 미래 고려 |

**§ 검증 정책 확정**
- access_context 바인딩: `user_id` 필수, 나머지(org_id, permissions 등)는 선택
- 재검증 규칙: user_id로 DB 조회 → 현재 활성 권한과 비교
- 권한 불일치 시: 요청 거부 (로그 기록)
- Scope 범위: API Key scope ⊇ 실제 권한 (무조건)

**§ Phase 4 구현 기준**
- FG4.1: MCP server의 OAuth 핸드셰이크
- FG4.2: Scope Profile CRUD, API Key 발급 시 바인딩
- 검수 기준: access_context 재검증 로직이 모든 쓰기/읽기 API에 적용되었는가?

### 4-2. 3.2 Scope Profile + FilterExpression + $ctx.* 변수 최종 검토

**§ 배경**
- S2 원칙 ⑥의 구체화: "접근 범위도 하드코딩 금지"
- 외부 소비자의 scope 어휘(team, org, confidential 등)가 변해도 코드 수정 불필요

**§ Scope Profile 구조 확정**
```
ScopeProfile {
  id: UUID,
  name: "기본 챗봇용",
  description: "팀 단위 접근 제어",
  ScopeDefinition[] {
    scope_name: "team",
    acl_filter: FilterExpression
  }
}
```

**§ FilterExpression 규칙 (S2 수위)**
- **허용 필드** (whitelist): 관리자가 설정
  - 기본값: `organization_id`, `team_id`, `visibility`, `classification`, `owner_id`
  - DocumentType별 metadata 필드도 추가 가능 (Phase 1에서 정의)
- **허용 연산자**:
  - 비교: `eq`, `neq`, `in`, `not_in`
  - 문자열: `starts_with`, `contains`
  - 범위: `gte`, `lte` (숫자만)
- **구조**: AND, OR 조합 가능 (최대 깊이 3단계)
- **$ctx.* 바인딩**: 요청의 access_context 변수로 동적 치환
  - 예: `{field: "organization_id", op: "eq", value: "$ctx.organization_id"}`

**§ 주입 공격 방어**
- $ctx.* 변수는 access_context 필드명만 참조 가능 (코드 실행 불가)
- FilterExpression 파싱: 정해진 문법만 인정 (SQL-like 표현식 거부)
- 필드명 검증: 먼저 whitelist에 있는가 확인

**§ 대안 비교**
| 옵션 | 장점 | 단점 | S2 선택 |
|------|------|------|--------|
| Scope Profile + FilterExpression | 관리자 설정, 유연함 | 구현 복잡 | ✅ |
| 하드코딩된 scope 문자열 | 간단 | 정책 변경 시 배포 필요 | ✗ |
| 임의 SQL 표현식 | 완전 유연 | 주입 공격 위험, S3 과제 | 미래(S3) |

**§ Phase 4 구현 기준**
- FG4.2: Scope Profile CRUD API 구현
- 검수 기준: 코드에 `if scope == "team"` 같은 하드코딩 문자열이 없는가?

### 4-3. 3.3 MCP 스펙 2025-11-25 핀 확정

**§ 선택 근거**
- 2025-11-25: OAuth client-credentials 정식 지원 (이전 버전은 기본 인증만)
- Tasks (experimental): 비동기 제안-승인 플로우 지원
- MCP Extensions: Mimir 고유 기능(citation 5-tuple, span 역참조) 선언 가능

**§ 기존 스펙 vs 2025-11-25**
| 항목 | 2025-09 | 2025-11-25 | S2 필요성 |
|------|---------|-----------|---------|
| OAuth client-credentials | 미지원 | ✅ | 필수 |
| Tasks | 미정의 | ✅ | 제안-승인 플로우 |
| Extensions | 기본 | 강화 | Mimir 기능 선언 |

**§ 호환 정책 확정**
- Phase 4 완료 시 "Mimir는 MCP 2025-11-25를 구현한다" 공식 문서화
- MCP 버전 업그레이드 규칙: 하위 호환 깨지는 변경은 별도 Phase로 취급
- 예: MCP 2026-03으로 업그레이드 필요 시 → Phase 9 또는 S3 초반

**§ Mimir가 활용하는 MCP 기능**
- **Tools**: search_documents, fetch_node, create_draft, propose_transition
- **Resources**: mimir://documents/{id}/versions/{v}/nodes/{n}
- **Prompts**: Prompt Registry의 템플릿을 MCP 프롬프트로 노출
- **Tasks**: 제안(working) → 승인 대기(input_required) → 완료(completed)
- **OAuth**: client-credentials 플로우
- **Extensions**: citation 5-tuple, span 역참조, Scope Profile 기능

**§ Phase 4 구현 기준**
- FG4.1: MCP Server 기본 구현 (Tools, Resources)
- FG4.3: MCP Tool schema 선택적 변환
- 검수 기준: `.well-known/mimir-capabilities` 엔드포인트가 MCP 2025-11-25 호환성을 선언하는가?

### 4-4. 3.4 Citation 5-tuple 계약 검수

**§ 필드 정의 확정**
```
Citation {
  document_id: UUID,      # 어떤 문서인가
  version_id: UUID,       # 몇 번째 개정본인가 (개정 후에도 citation 유효성 검증 가능)
  node_id: UUID,          # 문서 내 어떤 노드인가
  span_offset: int,       # [선택] 노드 내 어디서부터인가 (단락 단위 청크에서는 생략)
  content_hash: string    # SHA-256(노드 또는 청크 내용) — 내용 변조 여부 검증
}
```

**§ 각 필드의 검증 가능성**
- `document_id + version_id`: 개정 후에도 어떤 버전인지 명확 (변조 체크 가능)
- `content_hash`: 청크 내용이 원문과 동일한지 즉시 검증 가능
- `span_offset`: 긴 노드(예: 조항)에서 정확한 위치 지정 (선택적)

**§ content_hash 알고리즘 선택**
| 알고리즘 | 성능 | 충돌 위험 | S2 선택 |
|---------|------|---------|--------|
| SHA-256 | 중간 | 무시할 수준 | ✅ |
| MD5 | 빠름 | 충돌 위험 있음 | ✗ |
| Blake3 | 빠름 | 무시할 수준 | 미래(성능 최적) |

**§ span_offset 생략 규칙**
- 생략 가능: 단락/절 단위 청크, 짧은 문서
- 필수: 100 단어 이상의 긴 노드에서 부분 청크

**§ Phase 2 구현 기준**
- FG2.1: Citation 5-tuple 계약 고정
- span → 원문 역참조 API 구현
- 검수 기준: 모든 검색 응답에 {document_id, version_id, node_id, content_hash} 필드가 있는가?

### 4-5. 3.5 Agent Principal 모델 최종 정의

**§ 주체 모델**
```
Principal {
  type: "user" | "agent" | "system",
  id: UUID,
  acting_on_behalf_of: UUID (agent 또는 system인 경우만)
}
```

**§ Agent를 별개 주체로 모델링하는 이유**
1. 감시 및 추적: 에이전트별 행동 이력을 명확히 기록 가능
2. 킬스위치: 특정 에이전트만 즉시 차단 가능
3. 위임: 에이전트가 사용자를 명시적으로 대행하는 구조

**§ Delegation 정의**
- 에이전트가 사용자 대행 시: `acting_on_behalf_of: <user_id>`를 감시로그에 기록
- 권한: 에이전트 자신의 권한 ∩ 사용자의 권한 (교집합, 더 제한적)
- Scope: 에이전트의 API Key scope에 `delegate:search`, `delegate:write` 필요

**§ Kill Switch 정의 및 실행**
- **트리거**: 에이전트의 이상 행동 감지 (관리자 수동 또는 자동 규칙)
  - 예: 반려율 > 80%, 제안 생성 > 100/시간
- **동작**: 특정 에이전트의 쓰기(draft 생성, 워크플로 전이 제안) 즉시 차단
- **차단 지연 시간**: < 5초 (설정 반영 및 캐시 무효화)
- **해제**: 관리자 수동 해제만 가능 (자동 해제 없음)

**§ 감시로그 형식**
```
{
  timestamp: datetime,
  actor_type: "user" | "agent",
  actor_id: UUID,
  acting_on_behalf_of: UUID (nullable),
  action: "search" | "create_draft" | "propose_transition",
  resource_id: UUID,
  result: "success" | "failed",
  reason: string (실패 사유)
}
```

**§ Phase 4~5 구현 기준**
- FG4.2: Agent principal type 추가, Delegation ACL 적용
- FG5.3: 킬스위치 API, 감시 대시보드
- 검수 기준: 에이전트의 모든 쓰기가 감시로그에 `actor_type=agent` 기록되는가?

### 4-6. 3.6 평가용 판정 전략 최종 승인

**§ LLM 의존도별 판정 전략**

| 판정 유형 | LLM 필요 | 특징 | S2 도입 시점 |
|----------|---------|------|-----------|
| Citation-present check | 불필요 | 응답에 근거가 있는가 (규칙 기반) | Phase 7 FG7.2 |
| 근거 문장 ↔ 원문 매칭 | 불필요 | content_hash / 코사인 유사도 | Phase 7 FG7.2 |
| Answer relevance | embedding만 | 질문-답변 의미 유사도 | Phase 7 FG7.2 |
| Faithfulness | LLM 필수 | 답변이 근거에 충실한가 | Phase 7 FG7.2 (스왑 가능 LLM) |
| Context precision | LLM 필수 | 검색된 청크 중 관련 있는 비율 | Phase 7 FG7.2 (스왑 가능 LLM) |
| Context recall | LLM 필수 | 관련 청크 중 검색된 비율 | Phase 7 FG7.2 (스왑 가능 LLM) |

**§ 각 판정의 정확한 정의**
- **Citation-present rate**: 응답의 각 문장이 근거(citation with content_hash)를 가지는 비율
- **Faithfulness**: "응답 내용이 근거 문서와 모순하는가?" (LLM이 판정, 0~1 점수)
- **Answer Relevance**: embedding으로 질문-답변 코사인 유사도 (LLM 불필요)
- **Context Precision**: 검색 결과 중 "실제로 질문 답변에 기여하는" 청크의 비율 (LLM이 판정)
- **Context Recall**: "질문 답변에 필요한 모든 청크"를 검색했는가? (골든셋 vs 검색결과 비교)

**§ 폐쇄망에서의 판정**
- Citation-present, 근거 매칭: 항상 가능 (LLM 불필요)
- Answer relevance: 임베딩 모델이 있으면 가능 (로컬 모델 사용)
- Faithfulness, Context precision: 로컬 LLM(vLLM, Ollama) 사용 (GPT-4 대체 가능)

**§ Phase 7 구현 기준**
- FG7.2: 6개 지표 러너 구현, 각각의 정의 명시
- FG7.3: CI 게이트에 기준값 설정 (예: Faithfulness ≥0.80)
- 검수 기준: 폐쇄망 환경에서도 6개 지표 중 최소 4개(citation-present, relevance, precision, recall) 측정 가능한가?

---

## 5. 산출물

1. **핵심 설계 결정 확정서** (`S2_핵심설계결정_3.1~3.6_확정서.md`)
   - 각 결정 5개 섹션: 정의, 근거, 대안 비교, 트레이드오프, Phase 구현 기준

2. **대안 비교 매트릭스**
   - 각 결정마다 3~5개 대안 제시 및 장단점 정리

3. **Phase별 의존 관계 맵**
   - 3.1~3.6 각 결정이 어떤 Phase/FG에서 구현되는가

4. **검수 체크리스트**
   - 각 결정의 구현 완료를 확인하는 구체적 기준

5. **팀 합의 기록**
   - 최소 2명(기술 리드, 보안 담당)의 사인/날짜

---

## 6. 완료 기준

1. 3.1~3.6 각 결정에 대해 300단어 이상의 상세 기술 완료
2. 각 결정의 "대안"을 최소 2~3개 제시하고 선택 이유 명시
3. Phase별 구현 기준이 명확하여 해당 Phase FG에서 체크리스트로 즉시 사용 가능
4. 팀 전체 리뷰 완료 및 최소 2명의 서명 확보
5. Season 2 개발계획과의 대응 관계 표로 정리
6. 폐쇄망 환경에서의 동작을 각 결정마다 명시 (원칙 ⑦과의 정합)

---

## 7. 작업 지침

### 지침 7-1. 구체성 확보

- 추상적 설명 최소화
- "Phase 몇 FG몇에서 어떻게 구현되는가" 명시
- "검수 기준: ~이 구현되었는가?" 형태의 구체적 기준 제시

### 지침 7-2. 팀 리뷰 진행

1. 작성 완료 후 Slack/메일로 팀 전체에 공지 (48시간 리뷰 기간)
2. 기술 리드, 보안 담당자의 리뷰 필수
3. 의견 수렴 후 최종본 확정 및 사인
4. 사인날짜를 문서 상단에 명시

### 지침 7-3. 이후 Phase의 활용

- Phase 1 착수 시, 이 문서를 "기본 설계 기준 문서"로 배포
- 각 Phase FG의 작업지시서에서 해당 결정을 참조
- 예: Task 4-2에서 "3.5 Agent Principal 모델 참조" 명시

---

## 8. 참고 자료

- Season 2 개발계획.md §3 (핵심 설계 결정 전문)
- MCP Specification 2025-11-25 (§3.3 근거)
- S1 회고 §4 (원칙 ①④의 구현 경험)
- OAuth 2.0 RFC 6749 (§3.1 근거)
- OWASP Top 10 for LLM Applications (§3.6 판정 전략 근거)
