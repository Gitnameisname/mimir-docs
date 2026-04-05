# Task 3-9. AI / RAG 연계 가능한 API 인터페이스 설계

**Phase 3 설계 기준 문서**
**산출물**: `Task9_ai_rag_api_interface.md`
**선행 Tasks**: Task 3-1 ~ 3-8 (API 아키텍처, 리소스 구조, 인증/인가, 버전 전략, 응답 포맷, 목록 조회 규약, 멱등성 전략, 비동기/이벤트/웹훅 확장 모델)

---

## 1. 문서 목적

이 문서는 Mimir 범용 문서 플랫폼의 API가 AI, RAG 파이프라인, agentic workflow, 검색 기반 어시스턴트의 **공식 기반**이 될 수 있도록 하는 인터페이스 설계 기준을 정립한다.

이 문서의 목적은 다음과 같다.

- AI를 플랫폼 API의 정식 소비자로 정의하고, 우회 경로 없이 공식 API만으로 AI 기능을 지원할 수 있는 설계 방향을 수립한다.
- 문서 원문뿐 아니라 구조, 버전, 노드, 메타데이터, 권한 맥락을 AI가 일관되게 접근할 수 있는 인터페이스 원칙을 정의한다.
- retrieval, citation, indexing, ingestion, sync status, AI processing workflow를 수용할 수 있는 확장 포인트를 명시한다.
- 기존 플랫폼 API 계약과 충돌하지 않는 AI/RAG 인터페이스 연결 규칙을 확립한다.

이 문서는 임베딩 모델 선택, 벡터 DB 선택, chunking 알고리즘, retrieval ranking 알고리즘을 다루지 않는다. 인터페이스 계약과 API 설계 원칙에 집중한다.

---

## 2. 왜 AI / RAG 친화 인터페이스가 필요한가

### 2-1. 현재 구조의 한계

AI를 사후적으로 붙이는 방식은 다음 문제를 낳는다.

**원문 전체를 덤프하는 방식의 한계**
- 대형 문서를 통째로 AI에 전달하면 토큰 비용이 폭발하고, 관련 섹션과 무관한 섹션이 섞여 응답 정확도가 떨어진다.
- "어느 문서의 어느 버전 어느 섹션에서 온 내용인가"를 인용하기 위한 식별 정보가 없다.
- 문서가 갱신되었을 때 AI가 최신 버전을 접근하고 있는지 보장할 수 없다.

**비공식 내부 API 경로의 위험**
- AI 기능이 내부 직접 DB 쿼리나 파일 시스템 접근으로 구현되면, 접근 제어와 감사 추적이 무력화된다.
- 공식 API를 우회하는 순간 권한 모델이 분리되어 보안 일관성이 무너진다.

**RAG 파이프라인의 구조적 요구**
- RAG는 단순 전문(全文) 검색이 아니다. 구조 단위(노드/섹션) 별 조회, 버전 고정 조회, 출처 식별이 가능해야 retrieval 정확도와 citation 신뢰성이 보장된다.
- "가장 관련 있는 문서 3개 반환"이 아니라 "문서 X의 버전 Y의 섹션 Z의 내용, 출처 URI 포함"이 RAG의 실제 요구다.

### 2-2. 공식 AI 인터페이스가 필요한 이유

| 관점 | 비공식 AI 우회 | 공식 AI 인터페이스 |
|------|---------------|-------------------|
| 접근 제어 | 권한 무시 가능 | 플랫폼 ACL 그대로 적용 |
| 감사 추적 | actor 기록 없음 | Security Context 전파 |
| 인용 신뢰성 | 출처 불명확 | 안정된 resource identifier |
| 버전 일관성 | 최신/과거 혼재 | 버전 고정 조회 지원 |
| 운영 가시성 | 처리 상태 불투명 | operation tracking 가능 |
| 확장성 | 기능별 임시 API 난립 | 공통 계약 위에 확장 |

**결론**: AI/RAG 친화 인터페이스는 부가 기능이 아니라, 플랫폼을 지식 기반 시스템으로 확장하기 위한 공식 API 기반이다.

---

## 3. 상위 설계 원칙

### 원칙 1. AI is a First-Class API Consumer

**의미**: AI, RAG 파이프라인, agent는 사람 사용자 및 외부 시스템과 동등한 수준의 정식 API 소비자다.

**왜 필요한지**: AI를 2등 소비자로 취급하면 내부 우회 경로가 생기고, 권한과 감사 추적이 분리된다.

**설계 결정 유도**: AI 전용 별도 API 레이어를 만들기 전에, 기존 플랫폼 REST 리소스로 먼저 수용 가능한지 검토한다. AI만을 위한 특수 인증 우회는 허용하지 않는다.

---

### 원칙 2. Retrieval Builds on Platform Contracts

**의미**: 검색/retrieval 인터페이스는 플랫폼의 기존 리소스 구조, 응답 포맷, 권한 모델, 목록 조회 규약 위에 구축한다.

**왜 필요한지**: 독립된 검색 API를 별도로 만들면 인증/응답 포맷/오류 처리가 이중화되어 유지보수 비용이 증가한다.

**설계 결정 유도**: 검색 결과는 `data` + `meta` 공통 응답 봉투를 사용한다. 검색 결과의 각 항목은 기존 리소스 식별자(document_id, version_id, node_id)를 포함한다.

---

### 원칙 3. Structured Access over Raw Dumping

**의미**: AI에게 문서 전체 문자열을 통째로 넘기기보다, 노드/섹션/메타데이터 단위 접근을 우선 설계한다.

**왜 필요한지**: 구조화된 접근이 토큰 효율성, retrieval 정확도, citation 신뢰성 모두에서 유리하다.

**설계 결정 유도**: 문서 단건 조회 외에 구조 요약 조회, 특정 노드 단건 조회, 하위 노드 목록 조회를 별도로 지원한다.

---

### 원칙 4. Citation-Ready Resource Identity

**의미**: 모든 AI 소비 가능한 리소스는 안정된 식별자를 가지며, 검색 결과 항목에서 원문 리소스로 따라갈 수 있는 참조 정보를 포함한다.

**왜 필요한지**: AI 응답에서 출처를 명시하려면 "어느 문서의 어느 버전의 어느 섹션"을 기계적으로 따라갈 수 있는 안정된 참조가 필요하다.

**설계 결정 유도**: 검색 결과 항목에 `citation_ref` 구조체를 포함한다. document_id, version_id, node_id, canonical_uri를 조합한 인용 식별 정보를 표준화한다.

---

### 원칙 5. Version-Aware Knowledge Access

**의미**: AI는 최신 버전뿐 아니라 특정 시점의 버전에 접근할 수 있어야 한다. 버전이 다르면 같은 문서라도 내용이 다르며, 이를 인용 시 구분해야 한다.

**왜 필요한지**: "2025년 3월 기준 정책 X"를 인용하려면 당시 버전이 고정되어야 한다. 최신 버전만 제공하면 과거 내용의 출처를 보장할 수 없다.

**설계 결정 유도**: 버전 단위 조회 API를 명시적으로 지원한다. 검색 결과 항목은 어느 버전에서 왔는지 명시한다.

---

### 원칙 6. Permission-Preserving Retrieval

**의미**: 검색과 retrieval도 일반 리소스 접근과 동일한 보안 모델을 따른다. 검색 결과는 요청자 권한 범위 내 리소스만 포함한다.

**왜 필요한지**: 검색 인터페이스를 별도로 만들 때 ACL 검사가 누락되는 패턴이 자주 발생한다. 권한 없는 문서가 검색 결과에 섞이면 정보 누설이다.

**설계 결정 유도**: 검색/retrieval 요청에도 Request Security Context를 동일하게 전파한다. 검색 엔진 내부적으로 필터링을 적용하더라도, API 계층에서도 결과 항목 권한을 재확인하는 정책을 권장한다.

---

### 원칙 7. Minimal AI-Specific Duplication

**의미**: AI 전용 별도 API를 최소화한다. 기존 플랫폼 리소스 API로 수용 가능한 요구는 기존 API를 확장하거나 쿼리 파라미터로 수용한다.

**왜 필요한지**: AI 전용 엔드포인트가 늘어날수록 버전 관리와 보안 정책이 이중화된다.

**설계 결정 유도**: `GET /documents/{documentId}/structure`, `GET /documents/{documentId}/versions/{versionId}/nodes/{nodeId}` 같이 기존 리소스 계층을 최대한 활용한다.

---

### 원칙 8. Tool-Friendly Predictable Responses

**의미**: AI tool은 예측 가능한 키 구조와 일관된 응답 shape를 필요로 한다. 응답 필드 이름, 타입, 필수 여부가 안정적이어야 한다.

**왜 필요한지**: AI tool이 응답 파싱에 실패하면 pipeline 전체가 중단된다. 응답 구조의 예측 가능성이 AI 소비자에게 사람 소비자보다 더 중요하다.

**설계 결정 유도**: Task 3-5의 공통 응답 봉투를 AI 응답에도 그대로 적용한다. 검색 결과 항목의 필드를 안정적으로 문서화하고 breaking change 없이 추가 필드만 확장한다.

---

### 원칙 9. Async-Ready Ingestion and Indexing

**의미**: ingestion, indexing, embedding sync 같은 AI 처리 작업은 동기 응답으로 완료를 보장할 수 없으며, 비동기 operation 모델로 처리한다.

**왜 필요한지**: 대형 문서의 인덱싱은 수 초~수 분이 걸릴 수 있다. 동기 API로 강제하면 timeout 문제가 발생한다.

**설계 결정 유도**: Task 3-8의 `operations` 리소스를 AI 처리 작업에도 재사용한다. `202 Accepted` + operation_id 패턴을 ingestion/indexing 트리거에 적용한다.

---

### 원칙 10. Extensible toward Agent Workflows

**의미**: 현재는 검색과 조회 중심이더라도, 향후 summarization, extraction, agent action이 추가될 수 있도록 action endpoint 확장 지점을 열어둔다.

**왜 필요한지**: AI 활용 패턴은 빠르게 진화한다. 지금 당장 필요 없어도 확장 구조를 닫아버리면 나중에 더 큰 설계 변경이 필요하다.

**설계 결정 유도**: action endpoint 패턴(`:action` 접미사)과 operation 모델을 AI 처리 액션에도 적용할 수 있도록 일관된 확장 패턴을 유지한다.

---

## 4. AI 소비자 유형과 사용 패턴

### 4-1. 검색 / RAG 파이프라인

**특성**: 정기적 또는 온디맨드 문서 인덱싱, 사용자 쿼리에 대한 관련 문서 chunk/node 검색

**주요 API 패턴**
- 문서 목록 조회 (incremental sync, 변경된 문서 파악)
- 문서 버전별 노드 목록 조회 (구조화 ingestion)
- retrieval endpoint 호출 (쿼리 → 관련 노드/스니펫 반환)
- indexing/ingestion status 조회

**읽기 중심 여부**: 읽기 + ingestion 시작 트리거 (처리 시작형)

**구조화 데이터 필요 수준**: 높음 — 노드 단위 접근, 메타데이터, 버전 정보 필수

**권한/감사 중요 이유**: 인덱싱 대상 문서 범위가 권한 범위 내인지 확인해야 함. 검색 결과가 요청자 권한을 반영해야 함.

**동기 vs 비동기**: ingestion/indexing → 비동기 operation. 검색 쿼리 → 동기 REST.

---

### 4-2. 대화형 AI Assistant

**특성**: 사용자의 자연어 질문에 답하기 위해 실시간 문서 검색 및 특정 노드/섹션 조회

**주요 API 패턴**
- retrieval endpoint 호출 (쿼리 + 권한 필터 → 관련 노드 + citation_ref)
- 특정 노드 단건 조회 (검색 결과 후속 fetch)
- 문서 메타데이터 조회 (문서 상태, 작성일, 담당자 등 컨텍스트 파악)

**읽기 중심 여부**: 읽기 중심

**구조화 데이터 필요 수준**: 중간~높음 — citation 정보, 노드 컨텍스트 필요

**권한/감사 중요 이유**: 사용자가 권한 없는 문서 내용을 AI를 통해 간접 접근하면 안 됨. AI 응답 생성을 위한 API 호출도 감사 추적 필요.

**동기 vs 비동기**: 주로 동기 REST (실시간 응답 필요)

---

### 4-3. 문서 요약 / 분석 도구

**특성**: 특정 문서 또는 버전을 대상으로 요약, 비교, 분석 작업 수행

**주요 API 패턴**
- 특정 버전 조회
- 버전 내 전체 노드 목록 조회
- 특정 섹션 노드 단건 조회
- 분석 작업 트리거 (action endpoint)

**읽기 중심 여부**: 읽기 + 처리 시작형

**구조화 데이터 필요 수준**: 높음 — 섹션 단위 접근, 구조 계층 파악 필요

**권한/감사 중요 이유**: 요약 결과가 외부로 나갈 경우 원본 접근 권한이 있었는지 audit trail 필요

**동기 vs 비동기**: 처리 작업 → 비동기 operation

---

### 4-4. 외부 AI Application

**특성**: 플랫폼 API를 Service Account 또는 API Key 기반으로 호출하는 써드파티 AI 서비스

**주요 API 패턴**
- 조직/테넌트 범위 문서 목록 조회
- 버전/노드 조회
- retrieval endpoint
- webhook 구독 (문서 변경 시 인덱스 갱신)

**읽기 중심 여부**: 읽기 중심 + 이벤트 수신형

**구조화 데이터 필요 수준**: 중간 — 메타데이터 + 노드 콘텐츠

**권한/감사 중요 이유**: 외부 application이 테넌트 경계를 넘어 접근하면 안 됨. API key 발급 scope가 접근 범위를 제한해야 함.

**동기 vs 비동기**: 폴링 또는 webhook 기반

---

### 4-5. Agentic Workflow / Tool-Using Agent

**특성**: 여러 API 도구를 순차적으로 호출하며 목표를 달성하는 자율 에이전트. 검색 → 후속 조회 → 분석 → 요약 등 체인형 호출

**주요 API 패턴**
- retrieval → citation_ref 추출 → 노드 단건 조회 체인
- action endpoint 호출 (처리 작업 시작)
- operation status polling

**읽기 중심 여부**: 읽기 + 처리 시작형 혼합

**구조화 데이터 필요 수준**: 매우 높음 — 각 API 응답이 다음 호출의 입력이 되므로 식별자가 일관되게 포함되어야 함

**권한/감사 중요 이유**: Delegated 모드 (사용자 위임) vs Autonomous 모드 (서비스 계정 독립)에 따라 권한 scope가 다름. Task 3-3의 AI actor 모델 적용.

**동기 vs 비동기**: 혼합. 조회는 동기, 처리는 비동기 operation.

---

### 4-6. 내부 Indexing / Ingestion Service

**특성**: 플랫폼 내부에서 문서 변경 이벤트를 수신하고 검색 인덱스를 갱신하는 내부 서비스

**주요 API 패턴**
- 이벤트 구독 (document.version.published 등)
- 문서 버전/노드 조회 (인덱싱 대상 콘텐츠 fetch)
- indexing operation 시작 및 완료 상태 보고

**읽기 중심 여부**: 이벤트 수신 + 읽기

**구조화 데이터 필요 수준**: 높음 — 노드 단위 콘텐츠 + 메타데이터

**권한/감사 중요 이유**: 내부 서비스라도 Service Account로 동작하며 감사 추적 대상

**동기 vs 비동기**: 이벤트 기반 (비동기)

---

### 4-7. Evaluation / Offline Analysis Consumer

**특성**: 검색 품질 평가, 벤치마크, 오프라인 분석을 위해 대량 문서 데이터를 일괄 조회

**주요 API 패턴**
- 문서 목록 조회 (페이지네이션 기반 대량 순회)
- 버전/노드 일괄 조회
- export operation 트리거 (가능한 경우)

**읽기 중심 여부**: 읽기 중심 (대량 배치)

**구조화 데이터 필요 수준**: 중간 — 메타데이터 + 콘텐츠 구조

**권한/감사 중요 이유**: 평가 목적이라도 접근 권한 범위가 제한되어야 함. 대량 조회는 rate limit 적용 대상.

**동기 vs 비동기**: cursor 기반 페이지네이션 + export operation

---

## 5. AI/RAG 관점의 핵심 API 요구사항

다음 표는 기존 플랫폼 리소스와 AI 사용 패턴의 매핑이다. "AI 전용 API를 새로 만든다"가 아니라 **기존 리소스를 어떻게 활용하고 어디서 확장하는지**를 정리한다.

| AI 요구사항 | 기존 리소스 활용 | 확장/추가 고려사항 |
|------------|----------------|-----------------|
| 문서 목록 조회 | `GET /documents` | `updated_after` 필터, `ai_status` 필터 추가 가능 |
| 문서 단건 조회 | `GET /documents/{documentId}` | 기존 API 그대로 사용 |
| 특정 버전 조회 | `GET /documents/{documentId}/versions/{versionId}` | 기존 API 그대로 사용 |
| 구조화 노드 조회 | `GET /documents/{documentId}/versions/{versionId}/nodes` | 신규 — 노드 목록 조회 |
| 특정 노드 단건 조회 | `GET /documents/{documentId}/versions/{versionId}/nodes/{nodeId}` | 신규 |
| 문서 구조 요약 조회 | `GET /documents/{documentId}/versions/{versionId}/structure` | 신규 — 구조 tree 요약 |
| 메타데이터 조회 | `GET /documents/{documentId}` (metadata 포함) | `?fields=metadata` 필터 옵션 고려 |
| 검색 / retrieval | `POST /searches` | 기존 searches 리소스. AI용 relevance context 응답 필드 추가 |
| citation용 참조 정보 | 검색 결과 내 `citation_ref` 필드 | 검색 결과 항목에 citation_ref 구조체 표준화 |
| indexing / ingestion 상태 | `GET /documents/{documentId}/ai-status` 또는 operation tracking | 신규 sub-resource 또는 operation 연결 |
| AI 처리 작업 시작 | `POST /documents/{documentId}/versions/{versionId}:reindex` | action endpoint → operation 반환 |
| 변경 이벤트 구독 | webhook subscription (Task 3-8 그대로) | document.version.published 이벤트 활용 |
| 변경 사항 polling | cursor 기반 `GET /events` 또는 `GET /documents?updated_after=` | 기존 cursor pagination 적용 |

**핵심 원칙**: 기존 플랫폼 리소스를 최대한 재사용하고, AI 전용 인터페이스는 기존 구조로 수용 불가능한 경우에만 추가한다.

---

## 6. 문서 구조 조회 인터페이스 방향

### 6-1. 원문 전체 덤프의 한계

단순히 `GET /documents/{documentId}`로 전체 본문을 반환하는 방식은 다음 한계를 가진다.

- **토큰 비용**: 수백 페이지 문서를 통째로 AI에 전달하면 토큰 비용이 폭발한다.
- **관련성 희석**: 관련 없는 섹션이 섞여 retrieval 정확도가 떨어진다.
- **인용 불가능**: "이 내용이 문서의 어느 부분에 있는가"를 특정할 수 없다.
- **변경 추적 불가**: 어느 섹션이 변경되었는지 파악하지 못한다.

### 6-2. 구조 접근 API의 필요성

AI/RAG 파이프라인은 문서를 구조 단위로 접근해야 한다. 이를 위해 다음 수준의 접근이 필요하다.

**레벨 1 — 문서 메타데이터 조회**
```
GET /documents/{documentId}
→ 문서 기본 정보, 상태, 메타데이터 (본문 제외)
```

**레벨 2 — 버전 구조 요약 조회**
```
GET /documents/{documentId}/versions/{versionId}/structure
→ 노드 tree 구조 (id, type, title, depth, parent_node_id, child_count)
→ 본문 콘텐츠 없이 구조만 반환
```

**레벨 3 — 노드 목록 조회**
```
GET /documents/{documentId}/versions/{versionId}/nodes
?depth=1 (직접 하위만), ?node_type=section
→ 페이지네이션 적용
→ 각 노드: id, type, title, depth, parent_node_id, content_length, has_children
```

**레벨 4 — 특정 노드 단건 조회**
```
GET /documents/{documentId}/versions/{versionId}/nodes/{nodeId}
?include=content,context,siblings
→ 노드 본문 콘텐츠
→ 선택적으로 parent/sibling context 포함 가능
```

### 6-3. 노드 식별자 안정성

노드 식별자는 다음 조건을 만족해야 한다.

- **버전 내 불변**: 같은 버전 내에서 노드 id는 변경되지 않는다.
- **버전 간 추적 가능**: 구조 변경이 없는 노드는 버전이 바뀌어도 같은 id 또는 stable anchor를 유지하는 것을 권장한다. (후속 설계에서 확정)
- **RAG chunk와의 관계**: internal chunking이 노드보다 세분화되더라도, AI에게 노출되는 최소 주소 단위는 node_id다. chunk는 내부 구현 개념이며 외부 API에 직접 노출하지 않는다.

### 6-4. 컨텍스트 윈도우를 고려한 조회

노드 단건 조회 시 `?include=context` 옵션으로 주변 노드(parent, prev_sibling, next_sibling)의 요약 또는 제목을 포함할 수 있다. 이는 AI가 단일 노드를 문서 구조 맥락 안에서 이해하는 데 도움이 된다.

---

## 7. 버전 / 노드 / 메타데이터 접근 방향

### 7-1. 버전 관점

**최신 버전 vs 고정 버전**
- `GET /documents/{documentId}/versions/latest` — 현재 활성 버전으로 redirect 또는 직접 반환
- `GET /documents/{documentId}/versions/{versionId}` — 특정 버전 고정 조회

AI가 인용을 위해 특정 시점 내용을 조회할 때는 반드시 버전 고정 URI를 사용해야 한다. "최신 문서"를 인용하면 나중에 내용이 바뀌어도 같은 URI가 다른 내용을 반환하게 된다.

**버전 목록 조회**: `GET /documents/{documentId}/versions` — 증분 인덱싱 시 어느 버전까지 처리했는지 추적 가능

**버전 비교/diff**: 현재 범위 밖. 후속 확장 포인트로 남긴다.

### 7-2. 노드 관점

| 노드 접근 수준 | 용도 |
|------------|------|
| 노드 목록 (타이틀 + id만) | 구조 파악, 인덱싱 계획 수립 |
| 노드 단건 (본문 포함) | retrieval용 chunk 추출, 요약 입력 |
| 노드 + 컨텍스트 | 주변 맥락이 필요한 요약/분석 |
| 섹션 하위 전체 | 섹션 단위 처리 |

노드 type 예시: `section`, `subsection`, `paragraph`, `table`, `list`, `code_block`, `figure` — 구체 type 목록은 구현 Phase에서 확정.

### 7-3. 메타데이터 관점

AI/RAG에서 메타데이터의 역할:

| 메타데이터 필드 | AI 활용 목적 |
|-------------|------------|
| `document_type` | 문서 종류 기반 필터링 (정책/절차/규정 구분) |
| `status` | 폐기된 문서를 검색 결과에서 제외 |
| `author_id`, `owner_org_id` | 출처 신뢰도 판단, citation 설명 |
| `tags`, `categories` | 의미 기반 필터링, 검색 후처리 |
| `effective_date`, `expiry_date` | 시점 기반 조회 (특정 날짜 기준 유효 문서) |
| `custom_metadata` | 플랫폼 확장 메타데이터, AI 특화 분류 가능 |

메타데이터는 검색 결과 ranking 보정, retrieval 후처리, citation 설명 생성에 모두 활용 가능하므로 조회 응답에 항상 포함한다.

---

## 8. 검색 / Retrieval 인터페이스 방향

### 8-1. 일반 목록 조회 vs Retrieval Query의 구분

| 구분 | 일반 목록 조회 | Retrieval Query |
|------|-------------|----------------|
| 엔드포인트 | `GET /documents` | `POST /searches` |
| 입력 | 필터 파라미터 | 자연어 쿼리 또는 구조화 쿼리 |
| 결과 단위 | 문서 리소스 | 관련 노드/스니펫 + 점수 + citation_ref |
| 정렬 기준 | 필드 기반 정렬 | relevance score |
| 주용도 | 목록 UI, 배치 처리 | AI 답변 생성, RAG retrieval |

### 8-2. 검색 인터페이스 설계 방향

검색 요청은 `POST /searches` (Task 3-2 endpoint naming 규약 준수).

**요청 구조 (설계 방향)**:
```json
{
  "query": "규정 위반 처리 절차",
  "scope": {
    "document_types": ["policy", "procedure"],
    "statuses": ["published"],
    "org_ids": ["org-abc"]
  },
  "retrieval_unit": "node",
  "max_results": 10
}
```

- `scope`: 권한 범위 내에서 추가 필터링. 권한 모델이 scope를 override한다.
- `retrieval_unit`: `document` | `node` — AI는 주로 `node` 단위 결과를 사용
- 검색 알고리즘(full-text/semantic/hybrid)은 내부 구현. API 계약은 결과 구조만 정의한다.

**응답 결과 항목 구조 (설계 방향)**:
```json
{
  "data": [
    {
      "score": 0.91,
      "snippet": "...",
      "citation_ref": {
        "document_id": "doc-123",
        "version_id": "ver-456",
        "node_id": "node-789",
        "canonical_uri": "/documents/doc-123/versions/ver-456/nodes/node-789"
      },
      "context": {
        "document_title": "...",
        "version_label": "v3.1",
        "node_title": "3.2 위반 처리 절차",
        "node_type": "section"
      }
    }
  ],
  "meta": {
    "request_id": "...",
    "total_matches": 42,
    "search_id": "srch-abc"
  }
}
```

### 8-3. 검색 결과와 원문 리소스의 연결

검색 결과 항목의 `citation_ref.canonical_uri`로 후속 GET 요청을 통해 원문 노드를 조회할 수 있다. 이로써 검색 → 원문 fetch 체인이 완성된다.

### 8-4. 검색 결과 식별자 (search_id)

검색 결과 전체에 `search_id`를 부여한다. 이는 감사 추적과 디버깅에 활용되며, 특정 검색 결과를 나중에 참조할 때 사용할 수 있다.

---

## 9. Citation / Reference Identifier 설계 관점

### 9-1. 왜 안정된 Citation 식별자가 필요한가

AI가 "문서 X의 내용에 따르면..."이라고 답할 때, 그 출처가 검증 가능하려면 다음이 필요하다.

1. 어느 문서인지 — document_id
2. 어느 버전인지 — version_id (버전 고정 인용)
3. 어느 섹션인지 — node_id (가능한 경우)
4. 사람이 읽을 수 있는 설명 — 문서 제목, 섹션 제목, 버전 label

단순히 "문서 제목만 반환"하는 것은 불충분하다. 제목이 같은 문서가 여러 버전 존재할 수 있고, 제목은 변경될 수 있다.

### 9-2. Citation Reference 구조체

```
citation_ref:
  document_id: 플랫폼 내 문서 고유 식별자 (불변)
  version_id: 버전 고유 식별자 (버전 고정 인용)
  node_id: 섹션/노드 식별자 (있는 경우)
  canonical_uri: 플랫폼 내 resource URI (기계 접근 가능)
  human_label: 사람이 읽는 출처 설명 (e.g. "개인정보 처리방침 v2.3 §3.2")
  anchor: 선택적 anchor 또는 section path (e.g. "#3.2")
```

### 9-3. 버전 고정 인용 vs 최신 버전 인용

| 유형 | URI 형태 | 의미 |
|------|---------|------|
| 버전 고정 인용 | `/documents/{docId}/versions/{verId}/nodes/{nodeId}` | 특정 시점 내용 고정 |
| 최신 버전 인용 | `/documents/{docId}/versions/latest/nodes/{nodeId}` | 항상 최신 내용 참조 |

규정/정책 문서의 경우 버전 고정 인용이 원칙이다. "당시 적용된 정책"을 인용할 때는 반드시 version_id가 포함된 URI를 사용한다.

### 9-4. 포맷 확정은 후속 Phase에서

citation_ref의 구체적인 포맷(예: JSON 직렬화 방식, 외부 URL 포함 여부, anchor 표준)은 구현 Phase에서 확정한다. 이 문서에서는 **citation 가능한 플랫폼 식별 체계가 필요하다는 원칙**을 확립한다.

---

## 10. Indexing / Ingestion / AI Processing 상태 노출 방향

### 10-1. 왜 상태를 노출해야 하는가

AI-ready 플랫폼이 되려면 다음 질문에 API로 답할 수 있어야 한다.

- "이 문서의 최신 버전이 검색 인덱스에 반영되어 있는가?"
- "인덱싱이 진행 중인가, 실패했는가?"
- "마지막으로 인덱싱된 버전은 무엇인가?"

인덱싱 상태가 완전히 블랙박스이면, AI 서비스가 오래된 데이터를 조회하고 있는지, 아직 처리 중인지 알 수 없다.

### 10-2. AI 처리 상태 노출 방향

**방향 1 — 문서 단위 AI 상태 sub-resource**

```
GET /documents/{documentId}/ai-status
→ {
    "indexing": {
      "status": "indexed" | "pending" | "processing" | "failed" | "not_indexed",
      "last_indexed_version_id": "ver-456",
      "last_indexed_at": "2025-03-15T10:00:00Z",
      "error_summary": null
    },
    "embedding_sync": {
      "status": "synced" | "stale" | "pending",
      "synced_version_id": "ver-456"
    }
  }
```

**방향 2 — Operation 연결**

인덱싱 작업 트리거 시 operation_id를 반환하고, `GET /operations/{operationId}`로 진행 상태 추적.

```
POST /documents/{documentId}/versions/{versionId}:reindex
→ 202 Accepted
  { "data": { "operation_id": "op-xyz", ... } }

GET /operations/op-xyz
→ status: running → succeeded
```

**권장 방향**: 두 가지를 함께 사용한다. `ai-status`는 현재 상태 snapshot (polling 가능), `operation`은 특정 작업 추적. 구체적인 sub-resource 이름은 구현 Phase에서 확정한다.

### 10-3. 노출 대상 상태 항목 (검토 대상)

| 상태 항목 | 설명 | 노출 우선순위 |
|----------|-----|------------|
| `indexing.status` | 검색 인덱스 반영 상태 | 높음 |
| `last_indexed_version_id` | 마지막 인덱싱 완료 버전 | 높음 |
| `embedding_sync.status` | 임베딩 벡터 동기화 상태 | 중간 |
| `processing_errors` | 처리 실패 요약 | 중간 |
| `ingestion.status` | 원문 수집 완료 상태 | 낮음 (내부 처리에 따라) |

구체 필드 목록은 구현 Phase에서 플랫폼 AI 처리 파이프라인 설계와 함께 확정한다.

---

## 11. 권한 / 보안 / 감사 관점

### 11-1. AI는 권한 우회자가 아니다

AI 소비자도 일반 API 소비자와 동일한 보안 모델을 따른다.

- AI assistant가 대신 조회해도 원래 사용자가 접근 불가한 문서는 반환하지 않는다.
- RAG 파이프라인이 인덱싱하는 문서 범위는 Service Account의 권한 범위로 제한된다.
- AI agent가 여러 조직의 문서를 동시에 검색하려면 명시적 다중 테넌트 권한 구성이 필요하다.

### 11-2. Retrieval / Search의 권한 적용

- 검색 요청 시 Request Security Context (Task 3-3)가 동일하게 전파된다.
- 검색 엔진 내부 필터링 + API 계층 결과 필터링 이중 적용을 권장한다.
- 검색 결과 항목에 권한 없는 리소스가 포함되면 해당 항목을 제거하거나 접근 불가 표시를 한다.

### 11-3. Citation 정보 누설 방지

- citation_ref를 통해 문서 제목, 섹션 제목 등 메타데이터가 포함될 수 있다.
- 권한 없는 문서의 제목이라도 검색 결과에 노출되면 정보 누설이다.
- 권한 없는 리소스는 citation_ref 자체를 반환하지 않는다.

### 11-4. AI 처리 요청의 감사 추적

- AI 처리 작업(indexing, summarization 등)을 시작하는 API 호출도 actor_id, tenant_id, request_id가 기록된다.
- 비동기 operation에도 시작 actor 정보가 포함된다.
- AI Delegated 모드: 사용자 위임 시 원본 사용자 context 전파
- AI Autonomous 모드: Service Account 독립 actor로 기록

### 11-5. 테넌트 경계

- 조직/테넌트 간 검색은 기본 금지된다.
- 명시적으로 다중 테넌트 범위 접근이 허용된 경우에만 scope 확장 가능하며, 별도 권한 설정 필요.

---

## 12. 응답 Shape와 Tool-Friendly 설계 관점

### 12-1. AI Tool이 응답에 요구하는 것

AI tool은 다음을 필요로 한다.

- **예측 가능한 키 이름**: 매 응답마다 다른 필드 이름이 나오면 파싱 실패
- **일관된 타입**: 때로는 문자열, 때로는 null, 때로는 누락 — 방어 코드 없이 파이프라인 중단
- **follow-up fetch를 위한 식별자**: 모든 결과 항목에 후속 조회 URI 또는 id 포함
- **구조적 분리**: content / metadata / citation_ref / score가 섞이지 않고 명확히 구분

### 12-2. Tool-Friendly 응답 구조 원칙

Task 3-5의 공통 응답 봉투(`data` + `meta`)를 AI 응답에도 그대로 적용한다.

검색 결과 항목의 필드 분리 원칙:

```
result_item:
  score: relevance score (검색 결과만)
  snippet: 해당 부분 발췌 (검색 결과만)
  citation_ref: { document_id, version_id, node_id, canonical_uri, human_label }
  context: { document_title, version_label, node_title, node_type }
  resource_uri: 원문 리소스 URI (follow-up fetch용)
```

- `citation_ref`: 인용 정보 — AI 응답 생성용
- `context`: 사람이 읽는 설명 — 표시용
- `resource_uri`: 후속 GET 요청용 — tool chaining용
- 세 가지를 하나의 필드에 섞지 않는다.

### 12-3. 최소 필드 집합 개념

AI tool이 pipeline 내에서 다음 단계로 넘겨야 할 최소 정보:
1. 원문 접근을 위한 식별자 (`resource_uri` 또는 `citation_ref.canonical_uri`)
2. 인용을 위한 메타데이터 (`citation_ref`)
3. 다음 단계 결정을 위한 컨텍스트 (`context.node_type`, `score`)

불필요한 전체 본문을 목록 응답에 포함하지 않는다. 본문은 노드 단건 조회로 fetch한다.

### 12-4. 응답 확장 정책

AI 소비자를 위해 새 필드를 추가할 때 breaking change 없이 추가 가능하도록 Task 3-4 versioning 정책을 적용한다. AI tool은 미지의 필드를 무시하도록 설계되어야 한다.

---

## 13. 비동기 작업 / 이벤트 / 웹훅과의 연결

### 13-1. AI 처리 작업과 Operations 모델

Task 3-8에서 정의된 `operations` 리소스와 비동기 처리 모델을 AI 작업에 그대로 재사용한다.

| AI 작업 | Operations 연결 |
|--------|---------------|
| 문서 인덱싱 트리거 | `POST /:reindex` → `202 Accepted` + operation_id |
| 임베딩 동기화 | `POST /:sync-embeddings` → operation |
| 대량 export | `POST /documents:export` → operation |
| AI 분석 작업 시작 | action endpoint → operation |

모든 AI 처리 작업은 `operations` 모델을 통해 추적한다. AI 전용 별도 job 시스템을 만들지 않는다.

### 13-2. 이벤트 기반 AI 인덱스 갱신

문서 변경 → AI 인덱스 갱신의 흐름:

```
document.version.published 이벤트 발생
  → 내부 indexing service가 이벤트 수신
  → GET /documents/{documentId}/versions/{versionId}/nodes 로 콘텐츠 fetch
  → indexing operation 시작
  → indexing 완료 시 document.ai.indexed 이벤트 발행 (선택적)
```

이 흐름은 기존 platform event 모델 위에서 동작한다. AI 전용 이벤트 시스템이 아니다.

### 13-3. 외부 AI 시스템과의 연동 패턴

외부 RAG 서비스가 문서 변경을 실시간 반영하려면:

**Option A — Webhook 기반**:
- `document.version.published` 이벤트 webhook 구독
- 변경 발생 시 webhook 수신 → 해당 문서 fetch → 인덱스 갱신

**Option B — Polling 기반**:
- `GET /documents?updated_after={timestamp}` 으로 주기적 변경 감지
- cursor 기반 pagination으로 대량 처리 가능

두 방식 모두 기존 플랫폼 API 위에서 동작한다.

### 13-4. AI Event Category

Task 3-8의 이벤트 분류에서 AI 관련 이벤트 예시:

| 이벤트 타입 | 의미 |
|-----------|-----|
| `document.ai.indexed` | 문서 버전 인덱싱 완료 |
| `document.ai.index_failed` | 인덱싱 실패 |
| `document.ai.embedding_synced` | 임베딩 동기화 완료 |

이 이벤트 타입 목록은 확정이 아니라 방향 예시이다. 구현 Phase에서 확정한다.

---

## 14. 대표 사용 시나리오

### 시나리오 1 — 대화형 Assistant의 권한 범위 내 문서 검색 및 인용

```
1. User: "휴가 사용 관련 규정을 알려줘"
   → Assistant가 대신 API 호출 (Delegated 모드, user_id=U1 context)

2. POST /searches
   { "query": "휴가 사용 규정", "retrieval_unit": "node", "max_results": 5 }
   Authorization: Bearer {user_token}

3. 응답: node 5개, 각 항목에 score, snippet, citation_ref, context 포함
   결과 중 user_id=U1 권한 없는 노드는 자동 제외됨

4. citation_ref.canonical_uri로 상위 1개 노드 단건 조회
   GET /documents/doc-456/versions/ver-12/nodes/node-78
   → 전체 노드 본문 수신

5. Assistant 응답 생성 + 출처 표시:
   "인사규정 v2.3 §4.1 휴가 사용 기준에 따르면..."
   citation_ref: { document_id: "doc-456", version_id: "ver-12", node_id: "node-78" }
```

---

### 시나리오 2 — 외부 RAG 서비스의 조직 범위 문서 구조 수집 및 인덱싱

```
1. RAG 서비스가 Service Account (sa-rag@example.com)로 인증

2. 변경된 문서 파악:
   GET /documents?org_id=org-abc&updated_after=2025-03-01T00:00:00Z&page_size=50
   → cursor 기반으로 순회

3. 각 문서의 최신 버전 구조 조회:
   GET /documents/doc-X/versions/latest/structure
   → 노드 tree 확인 (콘텐츠 없이 구조만)

4. 필요한 노드 콘텐츠 수집:
   GET /documents/doc-X/versions/ver-Y/nodes?node_type=section
   → 페이지네이션으로 전체 섹션 노드 수집

5. 인덱싱 상태 확인:
   GET /documents/doc-X/ai-status
   → "indexing.status: stale" 확인 → 인덱싱 필요

6. 인덱싱 작업 완료 후 내부 인덱스 갱신
```

---

### 시나리오 3 — 문서 수정 → 인덱싱 Operation → Retrieval 가능 상태 반영

```
1. 편집자가 문서 버전 발행:
   POST /documents/doc-123/versions/ver-45:publish

2. 플랫폼이 document.version.published 이벤트 발행

3. 내부 indexing service가 이벤트 수신 → 인덱싱 시작:
   POST /documents/doc-123/versions/ver-45:reindex
   → 202 Accepted, operation_id: op-789

4. GET /operations/op-789 polling → status: running → succeeded

5. GET /documents/doc-123/ai-status
   → indexing.status: "indexed", last_indexed_version_id: "ver-45"

6. 이제 POST /searches 에서 ver-45 내용이 검색 결과에 반영됨
```

---

### 시나리오 4 — AI 분석 도구의 특정 버전/섹션 요약 생성

```
1. 분석 도구가 특정 문서의 v3.0과 v4.0의 3장을 비교 분석 요청

2. v3.0의 3장 노드 목록 조회:
   GET /documents/doc-99/versions/ver-30/nodes?parent_section=chapter-3
   → 3장 하위 섹션 노드 목록

3. v4.0의 동일 섹션 노드 목록 조회:
   GET /documents/doc-99/versions/ver-40/nodes?parent_section=chapter-3

4. 각 버전 노드 단건 조회로 본문 수집:
   GET /documents/doc-99/versions/ver-30/nodes/node-chapter3-1
   GET /documents/doc-99/versions/ver-40/nodes/node-chapter3-1

5. 수집된 콘텐츠 + 메타데이터로 요약 생성
   인용: ver-30 기준 변경 전 내용, ver-40 기준 변경 후 내용
```

---

### 시나리오 5 — Agent의 Citation Reference를 이용한 원문 노드 후속 조회

```
1. Agent가 검색 실행:
   POST /searches { "query": "화재 비상 대피 절차" }

2. 검색 결과 중 score 최상위 항목:
   {
     "score": 0.95,
     "snippet": "비상 대피 시 엘리베이터 사용 금지...",
     "citation_ref": {
       "canonical_uri": "/documents/doc-55/versions/ver-8/nodes/node-201"
     }
   }

3. Agent가 citation_ref.canonical_uri로 원문 노드 조회:
   GET /documents/doc-55/versions/ver-8/nodes/node-201?include=context

4. 전체 노드 본문 + 주변 컨텍스트 수신

5. Agent 응답 생성 + 인용 포함:
   "안전관리규정 v2.0 §5.3 비상 대피 절차에 따르면..."
```

---

## 15. 후속 Task 및 구현 Phase에 전달할 기준

### Task 3-10 (오류 모델 및 운영 관측성)으로 전달

- retrieval 실패 오류 코드 정의 — `RETRIEVAL_SCOPE_EMPTY`, `INDEXING_NOT_READY`, `NODE_NOT_FOUND`
- AI processing operation 실패 시 오류 응답 구조
- 검색 요청의 observability 메타데이터 — latency, matched_count, filtered_count
- indexing/ingestion 실패 알림 방식

### Task 3-11 (대표 API 시나리오)으로 전달

- 이 문서의 시나리오 1~5를 API 레벨 시나리오로 구체화
- 각 시나리오의 실제 HTTP 요청/응답 형식 검증
- AI 소비자 권한 검증 시나리오 포함

### 구현 Phase로 전달

- `GET /documents/{documentId}/versions/{versionId}/structure` 신규 API 설계 확정
- `GET /documents/{documentId}/versions/{versionId}/nodes` 신규 API 설계 확정
- `GET /documents/{documentId}/versions/{versionId}/nodes/{nodeId}` 신규 API 설계 확정
- `GET /documents/{documentId}/ai-status` 신규 API 설계 확정
- `POST /searches` 응답에 `citation_ref` 및 `context` 구조체 표준화
- `POST /documents/{documentId}/versions/{versionId}:reindex` action endpoint 설계
- citation_ref 포맷 확정 (JSON 구조, anchor 표현 방식)
- AI actor 등록 및 Service Account 권한 설정 방식
- document.ai.indexed 이벤트 타입 목록 확정

### 별도 기술 설계 Phase로 전달

- 검색 엔진/벡터 인덱스 기술 선택
- embedding 모델 선택 및 chunking 전략
- retrieval ranking 알고리즘
- indexing pipeline 인프라 설계
- AI workflow orchestration (내부 구현)

---

## 16. 결론

이 문서에서 확립한 핵심 방향은 다음과 같다.

**AI는 정식 API 소비자다.** 내부 우회 경로 없이 공식 플랫폼 REST API를 통해 문서 구조, 버전, 노드, 메타데이터, 검색 결과, 상태를 접근한다.

**구조화 접근을 원칙으로 한다.** 원문 전체 덤프 대신, 노드/섹션 단위 접근으로 토큰 효율성, retrieval 정확도, citation 신뢰성을 높인다.

**기존 플랫폼 계약 위에 확장한다.** 공통 응답 포맷, 권한 모델, 비동기 operation, 이벤트/웹훅 확장 모델을 AI 인터페이스에도 그대로 적용한다. AI 전용 별도 시스템을 최소화한다.

**Citation 가능한 식별 체계가 필요하다.** document_id + version_id + node_id + canonical_uri 조합으로 안정된 인용 참조를 보장한다.

**권한과 감사는 AI에게도 예외 없이 적용한다.** AI assistant를 통한 간접 접근도 원래 사용자 권한 범위를 따른다.

이 기준이 충족되면 Mimir 플랫폼은 별도의 AI 어댑터 없이도 RAG 파이프라인, 대화형 assistant, agentic workflow의 공식 기반이 될 수 있다.

---

## 자체 점검 요약

### AI 소비자 유형 요약
- 7가지 유형 정의: 검색/RAG 파이프라인, 대화형 assistant, 문서 요약/분석, 외부 AI application, agentic workflow, 내부 indexing 서비스, evaluation 소비자
- 각 유형별 주요 API 패턴, 읽기/처리 구분, 권한/감사 이유, 동기/비동기 패턴 정리

### 핵심 AI/RAG API 요구사항 요약
- 기존 플랫폼 리소스 최대한 재활용 원칙
- 신규 필요 API: `/structure`, `/nodes`, `/nodes/{nodeId}`, `/ai-status`, `:reindex` action
- 기존 `POST /searches`에 citation_ref + context 구조체 추가

### 문서 구조/버전/노드 접근 관점 요약
- 4 레벨 접근: 메타데이터 → 구조 요약 → 노드 목록 → 노드 단건(본문)
- 노드 식별자 안정성 원칙 (버전 내 불변, 버전 간 추적 가능 지향)
- chunk는 내부 개념, node_id가 외부 노출 최소 주소 단위

### 검색/retrieval/citation 관점 요약
- `POST /searches` 기존 리소스 활용, AI용 응답 필드 확장
- 검색 결과 항목: score + snippet + citation_ref + context + resource_uri 명확 분리
- citation_ref: document_id + version_id + node_id + canonical_uri + human_label
- 버전 고정 인용 원칙

### indexing/ingestion 상태 노출 관점 요약
- `GET /documents/{documentId}/ai-status` sub-resource (방향)
- operation 연결: `:reindex` → 202 → operation_id → 상태 추적
- 노출 항목: indexing.status, last_indexed_version_id, embedding_sync.status

### 권한/보안/감사 관점 요약
- AI는 권한 우회자가 아님 — 동일 Security Context 전파
- 검색 결과에 권한 없는 항목 제거 (API 계층 + 검색 엔진 이중 필터)
- citation 정보도 권한 범위 내에서만 반환
- AI 처리 작업도 actor 기록 및 audit trail

### operation/event/webhook 연결 관점 요약
- AI 처리 작업 → 기존 operations 모델 재사용 (전용 job 시스템 불필요)
- 문서 변경 → `document.version.published` 이벤트 → indexing 서비스
- 외부 AI 시스템 연동: webhook 구독 또는 polling (`updated_after` 필터)
- AI 전용 이벤트 타입 예시 제시 (확정은 구현 Phase)

### 후속 Task 연결 포인트
- **Task 3-10**: retrieval 오류 코드, AI operation 실패 오류, observability metadata
- **Task 3-11**: 이 문서 시나리오의 HTTP 레벨 API 시나리오 검증
- **구현 Phase**: structure/nodes/ai-status API 확정, citation_ref 포맷 확정, AI actor 설정

### 의도적으로 후속 Task로 남긴 미결정 사항
- citation_ref 포맷 확정 (JSON 직렬화 방식, external URL 포함 여부)
- 노드 타입 목록 확정 (section/paragraph/table 등)
- `/ai-status` 구체 필드 목록 확정
- AI 관련 이벤트 타입 목록 확정
- 버전 간 노드 id 추적 방식 확정
- 검색 알고리즘 내부 구현 선택 (full-text/semantic/hybrid)
- AI processing action endpoint 목록 확정
- rate limit 정책 (AI 소비자 별도 quota 여부)
