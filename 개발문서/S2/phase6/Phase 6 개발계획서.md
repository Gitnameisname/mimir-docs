# Phase 6 개발계획서
## 관리자 기능 통합

---

## 1. 단계 개요

### 단계명
Phase 6. 관리자 기능 통합

### 목적
Phase 1~5에서 구축된 모든 백엔드 API(모델·프롬프트·에이전트·Scope Profile·제안 큐·에이전트 감시 등)를 하나의 통합 관리 콘솔로 수렴한다. S1 Admin 기능(사용자/역할/설정/감사/API키)과 S2 Admin 기능을 일관된 네비게이션과 UX로 통합하여, 관리자가 이 시스템의 모든 구성 요소를 중앙에서 조작하고 모니터링할 수 있게 한다.

### 선행 조건
- Phase 0~5 완료 및 모든 FG 검수 완료
- Phase 1~5의 백엔드 API 정상 작동 확인
- S1 Admin UI 기존 구현물 상태 파악
- Phase 0에서 정의한 UI 5회 리뷰 체크리스트 준비

### 기대 결과
- 통합 Admin 대시보드 구축 (좌측 네비게이션 통일)
- FG6.1: AI 플랫폼 관리 UI (모델/프롬프트/Capabilities/비용·사용량)
- FG6.2: 에이전트·Scope 관리 UI (에이전트 목록/Scope Profile/제안 큐/활동 대시보드)
- FG6.3: 평가·추출 관리 UI (골든셋/평가 결과/추출 스키마/검토 큐) — 스켈레톤 + Phase 7~8 후 최종 연결
- 모든 Admin UI 컴포넌트 데스크톱·웹 호환 및 최소 5회 리뷰 완료
- Phase 7~8 최종 통합 준비 상태

---

## 1-1. 이전 Phase 이월 항목

Phase 3·4에서 이월된 항목 중 Phase 6에서 처리할 사항.

| ID | 항목 | 원래 Phase | 우선순위 | 처리 방침 |
|----|------|-----------|---------|-----------|
| PH3-CARRY-002 | **대화 전문 FTS (tsvector)** — 대화 제목 LIKE 검색 → 전문 Full-Text Search 전환 | Phase 3 | 낮음 | Phase 6 검색 고도화 작업(FG6.x)에서 `conversations` 테이블에 `tsvector` 컬럼 추가 및 GIN 인덱스 생성, 검색 API 업데이트 |
| DOCS-001 | **외부 클라이언트용 MCP 통합 가이드** — LangChain·n8n 등 외부 에이전트가 Mimir MCP Server를 도구로 연결하는 단독 가이드 문서 미산출 | Phase 4 | 중간 | FG6.2 에이전트 관리 UI 구축 시 함께 작성. 엔드포인트 목록·인증(API Key + `expires_at` 필수)·Tool Schema·연동 예시 코드 포함 → Task 6-5 산출물로 추가 |

> **PH3-CARRY-002 상세**: 현재 `ConversationRepository.list()`의 제목 검색은 `LIKE '%{query}%'` 방식이다. 대화 수가 증가하면 성능 저하가 발생한다. `tsvector` 컬럼 + GIN 인덱스 기반 FTS로 전환하고, 검색 API(`GET /api/v1/conversations?search=`)도 함께 업데이트한다.

> **DOCS-001 상세**: Phase 4 전체 검수에서 발견된 미산출 항목 (Phase4_전체검수보고서.md §7 DOCS-001). MCP Server 자체는 완성되었으나 외부 개발자를 위한 통합 가이드가 없다. FG4.1 검수보고서의 엔드포인트 표가 현행 대체 중이며, 에이전트 관리 UI 완성 시점에 맞춰 실제 연동 예시와 함께 완성하는 것이 효율적이다. **REC-4.3 영향**: Agent API Key는 반드시 `expires_at`을 명시해야 하며(NULL 시 인증 거부), 가이드에 이 제약을 필수로 기술해야 한다.

---

## 2. Phase 6의 역할

Phase 6은 "관리 경험의 통합" Phase이다. 다음을 담당한다:

1. **통합 Admin 콘솔 아키텍처 구축**: S1 Admin UI 기존 네비게이션 위에, S2의 새로운 관리 메뉴(AI 플랫폼, 에이전트, 평가)를 계층적으로 배치
2. **즉시 구축 가능한 FG 선행**: FG6.1과 FG6.2는 Phase 1~5의 API가 모두 준비되었으므로 이 Phase에서 완성
3. **스켈레톤 기반 FG6.3**: 골든셋·평가·추출 관리의 UI 레이아웃과 라우팅만 먼저 구축. 데이터 바인딩과 기능 상세화는 Phase 7~8에서 단계적으로 추가
4. **UI 품질의 엄격한 통제**: 모든 Admin UI는 5회 이상의 리뷰를 거쳐, 접근성·반응형·상태 관리·오류 처리를 일관되게 구현

---

## 3. Feature Group 상세

### FG6.1 — AI 플랫폼 관리 대시보드

#### 목적
모델 프로바이더, 프롬프트 레지스트리, 시스템 capabilities, 비용·사용량 메트릭을 한곳에서 조작하고 모니터링할 수 있는 관리 인터페이스 제공.

#### 주요 작업 항목

1. **모델/프로바이더 관리 페이지**
   - 등록된 LLM/임베딩 프로바이더 목록 표시 (테이블 형식)
   - 각 프로바이더별 메타데이터: 이름, 타입(LLM/Embedding), 상태(활성/비활성/오류), 마지막 테스트 시각
   - "연결 테스트" 버튼: 선택한 프로바이더에 즉시 health check 요청, 결과 모달 표시
   - "기본 모델 지정" UI: LLM/Embedding 타입별로 기본값으로 사용할 모델 선택 (드롭다운)
   - 프로바이더 설정 변경(예: API key 갱신) 시 재연결 필요 알림

2. **프롬프트 관리 페이지**
   - Phase 1 Prompt Registry API를 UI로 노출
   - 프롬프트 목록: 이름, 설명, 현재 활성 버전, 생성일, 수정일
   - "프롬프트 생성" 버튼: 모달에서 이름, 설명, 초기 내용 입력 → POST /api/v1/prompts
   - 프롬프트 상세 보기: 버전 이력 표시 (timeline 형식)
     - 각 버전: 버전 번호, 생성일, 수정자, 내용 미리보기, "이 버전 활성화" 버튼
   - "프롬프트 편집" 모달: 현재 버전의 내용 수정 → 새 버전 생성 (PUT /api/v1/prompts/{id}/new-version)
   - A/B 테스트 설정: 특정 프롬프트 버전을 A/B 비율로 라우팅 (메타데이터 저장)

3. **Capabilities 대시보드**
   - `/system/capabilities` 응답을 시각화
   - 카드 레이아웃: 각 기능별 on/off 상태
     - RAG 가용: Yes / No (pgvector 상태)
     - Embedding 모델: 현재 활성 모델명
     - LLM 프로바이더: 등록된 프로바이더 개수 및 활성 상태
     - 문서 Chunking: 활성/비활성
   - Degrade 상태 경고: 일부 기능이 비활성이면 "시스템이 제한된 모드로 동작 중입니다" 배너 표시
   - Refresh 버튼: 수동으로 capabilities 재조회 (캐시 무효화)

4. **비용·사용량 현황**
   - 시계열 그래프 (최근 30일 기준):
     - 모델별 토큰 사용량 (stacked bar)
     - 프로바이더별 호출 수 (line chart)
     - 총 예상 비용 추이
   - 모델별 상세 테이블:
     - 모델명, 누적 토큰(input/output), 호출 수, 평균 지연(ms), 예상 비용
   - 필터: 기간 선택(7일/30일/90일), 프로바이더 필터
   - Export: CSV 다운로드

#### 입력 (선행 의존)
- Phase 1 FG1.1: LLM Provider Abstraction API
- Phase 1 FG1.2: Embedding Model Abstraction API
- Phase 1 FG1.3: Prompt Registry API
- Phase 0 FG0.2: `/system/capabilities` 엔드포인트
- 모델 사용량 메트릭 수집 API

#### 출력 (산출물)
- Admin UI 프로바이더 관리 페이지 (TypeScript/React)
- Admin UI 프롬프트 관리 페이지 (버전 관리, A/B 테스트)
- Admin UI Capabilities 대시보드
- Admin UI 비용·사용량 대시보드
- 차트 라이브러리 통합 (recharts 또는 plotly)
- 데이터 갱신 전략 (폴링 vs. WebSocket)

#### 검수 기준
- 모든 UI 컴포넌트가 반응형 (mobile 640px, tablet 768px, desktop 1024px+)
- 프로바이더 연결 테스트 후 결과 즉시 표시 여부
- 프롬프트 버전 이력이 timeline 형식으로 명확하게 표시되는가
- Capabilities 상태가 시스템 현실과 동기화되는가
- 비용 계산이 정확한가 (Phase 1 메타데이터와 일치)
- 모든 CRUD 작업에서 낙관적 업데이트 또는 명확한 로딩 상태 표시
- 최소 5회 UI 리뷰 기록

---

### FG6.2 — 에이전트·Scope 관리 콘솔

#### 목적
Phase 4~5에서 구축한 에이전트 principal, delegation, Scope Profile, 제안 큐, 에이전트 감시를 중앙집중식으로 관리.

#### 주요 작업 항목

1. **에이전트 관리 페이지**
   - 등록된 에이전트 목록: 이름, 설명, 위임 대상 사용자 수, 상태(활성/차단), 마지막 활동
   - "에이전트 생성" 버튼: 모달에서 이름, 설명, 대행 권한 범위 설정 → POST /api/v1/agents
   - 에이전트 상세 보기:
     - 기본 정보: 이름, 설명, 생성일, 마지막 활동
     - 위임 상태: 이 에이전트가 대행하는 사용자 목록 (위임 해제 버튼 포함)
     - 권한: delegate scope 목록 (scope 추가/제거)
     - 킬스위치: "이 에이전트 차단" 버튼 (즉시 활성화, 1시간/24시간/영구 차단 옵션)
       - 차단 후 제안 큐의 모든 대기 중인 제안 자동 거절 여부 선택
   - 에이전트별 API Key 재발급: 보안상 새 key 생성, 이전 key 즉시 무효화

2. **Scope Profile 관리 페이지**
   - 정의된 Scope Profile 목록: 이름, 설명, 포함된 scope 개수, 사용 중인 API Key 수
   - "Scope Profile 생성" 버튼: 모달에서 이름, 설명 입력 → POST /api/v1/scope-profiles
   - Scope Profile 상세 편집:
     - Profile 메타데이터 (이름, 설명 수정)
     - Scope 목록 (테이블 형식):
       - scope name, ACL filter (JSON 형식 또는 시각 빌더)
       - 각 scope별 "수정" 버튼: filter 재정의
       - "Scope 추가" 버튼: 새로운 scope name + filter 정의
     - 이 Profile을 사용하는 API Key 목록 (읽기 전용)
   - Scope Profile 삭제: 사용 중인 API Key가 있으면 경고

3. **ACL 필터 시각 빌더**
   - FilterExpression을 노코드로 구성
   - 구성 요소:
     - 필드 선택 (organization_id, team_id, visibility, classification 등 드롭다운)
     - 연산자 선택 (eq, neq, in, contains 등)
     - 값 입력 또는 변수 선택 ($ctx.organization_id 등)
   - AND/OR 논리 연산 (tree 형식 드래그 가능)
   - JSON 텍스트 편집 모드 토글
   - 프리뷰: "이 필터는 문서 1000건 중 150건을 허용합니다" 표시 (추정)

4. **에이전트 제안 큐**
   - 전체 제안 목록: 에이전트명, 문서명, 제안 내용 요약, 제안 시각, 상태(대기/승인/거절)
   - 필터: 에이전트별, 문서별, 상태별, 기간별
   - 제안 상세 보기:
     - 제안 메타데이터: 에이전트, 문서, 목표 상태, 제안 사유, 타임스탬프
     - 변경 내용 상세: Diff 뷰 (기존 content vs. 제안된 content, line-by-line)
     - 승인/거절 버튼 + 피드백 텍스트 필드
   - 일괄 작업: "선택된 제안 모두 승인" / "모두 거절" (체크박스 다중 선택)
   - 실시간 상태 업데이트 (WebSocket 또는 polling)

5. **에이전트 활동 대시보드**
   - 시계열 요약:
     - 에이전트별 제안 수 (막대 그래프, 최근 30일)
     - 승인률 추이 (line chart, %로 표시)
     - 평균 승인까지 소요 시간 (bar chart)
   - 에이전트별 상세 통계:
     - 총 제안 수, 승인 수, 거절 수, 승인률(%)
     - 거절 사유 상위 5개 (텍스트 분석)
     - 가장 자주 제안되는 문서 타입
   - 이상 행동 알림:
     - 24시간 내 거절률 > 50%인 에이전트 강조
     - 킬스위치 발동 히스토리 표시
   - Export: CSV 다운로드

#### 입력 (선행 의존)
- Phase 4 FG4.2: Agent Principal + Delegation + Scope Profile CRUD API
- Phase 5 FG5.1: 에이전트 쓰기 + Draft 제안 API
- Phase 5 FG5.2: 에이전트 제안 큐 관리 API
- Phase 5 FG5.3: 에이전트 감사 뷰 API

#### 출력 (산출물)
- Admin UI 에이전트 관리 페이지
- Admin UI Scope Profile 관리 페이지 + ACL 필터 시각 빌더
- Admin UI 에이전트 제안 큐 (상세 diff 뷰 포함)
- Admin UI 에이전트 활동 대시보드
- 차트 라이브러리 통합
- 실시간 업데이트 메커니즘
- **MCP 통합 가이드 문서** (`docs/개발문서/S2/mcp_integration_guide.md`) ← *DOCS-001 이월 처리*

#### 검수 기준
- 에이전트 킬스위치 발동 후 5초 이내 신규 제안 차단되는가
- Scope Profile 수정 후 기존 API Key의 동작이 즉시 반영되는가
- 제안 큐에서 Diff가 명확하게 표시되는가 (라인 넘버, 색상 구분)
- 에이전트별 승인률이 정확하게 계산되는가
- 모든 UI가 반응형이고 최소 5회 리뷰 완료
- 일괄 작업(승인/거절) 후 롤백 방법 명확한가
- MCP 통합 가이드에 `expires_at` 필수 제약 및 엔드포인트 7종이 모두 기술되어 있는가

---

### FG6.3 — 평가·추출 관리

#### 목적
Phase 7~8의 평가 및 추출 기능의 관리자 인터페이스를 스켈레톤 형태로 먼저 구축. Phase 7·8 완료 후 백엔드와 연결하여 최종 완성.

#### 주요 작업 항목

1. **골든셋 관리 페이지 (스켈레톤)**
   - 골든셋 목록: 이름, 설명, 문항 수, 최신 버전, 생성일
   - "골든셋 생성" 버튼: 모달에서 이름, 설명 입력 → 백엔드 스켈레톤 API
   - 골든셋 상세 보기:
     - 기본 정보 (이름, 설명, 버전)
     - 질문-답변 항목 테이블 (스켈레톤: 열 정의만. 데이터는 Phase 7 FG7.1 후 연결)
       - 질문, 기대 답변, 기대 근거 문서, 기대 citation
     - "항목 추가" 버튼 (Phase 7 후 활성화)
     - "버전 이력" 탭 (timeline)
     - "Import/Export" 버튼 (JSON 형식)
   - 평가 실행 이력 (Phase 7 후 연결)

2. **평가 결과 대시보드 (스켈레톤)**
   - 대시보드 레이아웃:
     - 상단: 최근 평가 실행 요약 (카드: 실행 시간, 평가 항목 수, 전체 평균 스코어)
     - 좌측: 지표별 현황 (Faithfulness, Answer Relevance, Citation-present 등)
       - 각 지표: 최신 값, 기준값(threshold), 통과/미통과 상태
     - 우측: 추이 그래프 (30일 시계열)
   - 필터: 기간 선택, 프롬프트 버전 필터, 모델 필터
   - CI 게이트 상태 표시: "마지막 CI 실행: 통과" 또는 "실패" (Phase 7 후 연결)
   - 상세 실행 결과 조회: 평가 실행 ID 선택 → 개별 Q&A별 스코어 테이블

3. **추출 타겟 스키마 관리 (스켈레톤)**
   - DocumentType별 추출 스키마 정의
   - 스키마 목록: DocumentType명, 정의된 필드 수, 마지막 수정일
   - 스키마 편집:
     - 선택한 DocumentType의 metadata schema 표시 (Phase 8 FG8.1 후 연결)
     - 스키마 필드 테이블:
       - 필드명, 타입(string/number/boolean/array 등), 필수 여부
       - "필드 추가" 버튼 (Phase 8 후 활성화)
     - 추출 모드 설정: "결정론적" (동일 입력→동일 출력) 또는 "확률적"

4. **추출 결과 검토 큐 (스켈레톤)**
   - 미승인 추출 결과 목록:
     - 문서명, DocumentType, 추출 시각, 상태(검토 대기)
     - 필터: DocumentType별, 상태별
   - 검토 상세 보기:
     - 원본 문서 미리보기
     - 추출된 필드 값 테이블 (span 역참조 표시, Phase 8 FG8.3 후 연결)
     - 각 필드별 승인/수정 버튼
     - "모두 승인" / "반려" (이유 텍스트)
   - 승인 후 자동으로 Document metadata에 저장 (Phase 8 FG8.2 후 연결)

#### 입력 (선행 의존)
- Phase 7 FG7.1: 골든셋 도메인 모델 + API (추후)
- Phase 7 FG7.2: 평가 러너 + 지표 API (추후)
- Phase 8 FG8.1: 추출 타겟 스키마 정의 API (추후)
- Phase 8 FG8.2: 추출 결과 API (추후)

#### 출력 (산출물)
- Admin UI 골든셋 관리 페이지 (스켈레톤)
- Admin UI 평가 결과 대시보드 (스켈레톤)
- Admin UI 추출 스키마 관리 페이지 (스켈레톤)
- Admin UI 추출 결과 검토 큐 (스켈레톤)
- 라우팅 정의 (URL path: /admin/golden-sets, /admin/evaluations, 등)
- 좌측 네비게이션 메뉴 항목 추가

#### 검수 기준
- 모든 UI 레이아웃이 반응형이고 일관된 디자인을 유지하는가
- Phase 7·8 완료 후 API 연결이 최소한의 추가 작업으로 가능한가
- 스켈레톤 상태에서도 접근성 기준(WCAG 2.1 AA) 충족되는가
- 라우팅이 명확하고 네비게이션이 자연스러운가

#### 특수 고려 사항
- **Phase 7·8과의 핸드오프**: FG6.3 완료 시 "API 연결 가능 상태" 명시. Phase 7·8 팀은 FG6.3의 구조를 학습하고, API 완료 후 바인딩만 수행
- **데이터 시뮬레이션**: 스켈레톤 상태에서 목 데이터(mock data)를 표시하여 UX를 검증 가능하게 구성

---

## 4. 기술 설계 요약

### 4.1 Admin UI 아키텍처
```
Admin Router
├─ S1 영역
│  ├─ 사용자/역할 관리 (기존)
│  ├─ 시스템 설정 (기존)
│  ├─ 감사 로그 (기존)
│  └─ API 키 관리 (기존)
└─ S2 영역
   ├─ AI 플랫폼 (FG6.1)
   │  ├─ 모델/프로바이더 관리
   │  ├─ 프롬프트 관리
   │  ├─ Capabilities 대시보드
   │  └─ 비용·사용량
   ├─ 에이전트·Scope (FG6.2)
   │  ├─ 에이전트 관리
   │  ├─ Scope Profile 관리
   │  ├─ 제안 큐
   │  └─ 활동 대시보드
   └─ 평가·추출 (FG6.3, 스켈레톤)
      ├─ 골든셋 관리
      ├─ 평가 결과 대시보드
      ├─ 추출 스키마 관리
      └─ 추출 결과 검토 큐
```

### 4.2 좌측 네비게이션 구조
- 접으면/펼치면 토글 가능 (desktop 호환)
- 섹션별 그룹 (S1 Admin, S2 Admin)
- 현재 경로 강조 (active state)
- 페이지 로드 시 마지막 방문 섹션 열어둠

### 4.3 UI 컴포넌트 라이브러리
- 테이블: 정렬, 필터, 페이지네이션 (headless UI: TanStack Table)
- 모달: 폼 입력, 확인 다이얼로그
- 차트: recharts (시계열, 막대, 선형)
- 폼: React Hook Form + Zod 검증
- 알림: toast 메시지 (sonner 또는 react-toastify)

### 4.4 상태 관리
- 전역 에이전트 상태 (Zustand): 현재 로그인 사용자, 권한 캐시
- 페이지별 로컬 상태: 폼 입력, 필터 선택, UI 전개/축소 상태
- React Query: API 캐싱 및 백그라운드 동기화

### 4.5 에러 처리
- API 에러 응답 공통 처리 (unwrapEnvelope 활용)
- 사용자 피드백: 에러 메시지 모달 또는 inline 경고
- 낙관적 업데이트(optimistic update) vs. 안전한 업데이트 전략 (Delete는 확인 필수)

---

## 5. 의존 관계

### 선행 Phase
- **Phase 0**: UI 5회 리뷰 체크리스트, 네비게이션 원칙
- **Phase 1**: LLM Provider, Embedding, Prompt Registry API
- **Phase 2**: (Citation 계약, 실질적 의존 없음)
- **Phase 3**: Conversation 도메인 — FG6.1 비용·사용량 대시보드에서 대화 세션별 모델 호출 메트릭 포함 검토 필요
- **Phase 4**: Agent Principal, Delegation, Scope Profile API
- **Phase 5**: 에이전트 제안 큐, 감사 API

### 후행 Phase (이 Phase의 산출물을 소비하는 Phase)
- **Phase 7**: FG6.3의 평가 대시보드 스켈레톤을 활용하여 실제 평가 결과 바인딩
- **Phase 8**: FG6.3의 추출 관리 스켈레톤을 활용하여 추출 스키마·결과 바인딩
- **Phase 9**: Phase 6 FG6.3 최종 연결 검증

### 주요 핸드오프 지점
- FG6.1 → Phase 7: "프롬프트 A/B 테스트 결과를 비용·사용량 대시보드에 반영" (정성적 요청)
- FG6.3 → Phase 7·8: API 스텁 → 실제 구현 바인딩

---

## 6. 검수 기준 종합

| 항목 | 기준 |
|------|------|
| **FG6.1** | 모든 모델 관리 CRUD 정상 작동 / 프롬프트 버전 관리 및 A/B 설정 가능 / Capabilities 실시간 반영 / 비용 계산 정확성 확인 |
| **FG6.2** | 에이전트 킬스위치 발동 후 5초 내 신규 제안 차단 / Scope Profile 필터 실시간 적용 / 제안 큐 Diff 명확 표시 / 승인률 통계 정확성 |
| **FG6.3** | 모든 스켈레톤 레이아웃 반응형 / Phase 7·8 API 연결 가능성 사전 검증 / 접근성 기준 준수 |
| **통합** | S1 Admin UI와 S2 Admin UI 네비게이션 일관성 / 모든 Admin 페이지 최소 5회 리뷰 완료 / desktop+web 호환성 검증 / 권한 기반 메뉴 조건부 렌더링 |

---

## 7. 예상 산출물 목록

| 산출물 | 형태 | 설명 |
|--------|------|------|
| Admin UI 통합 레이아웃 컴포넌트 | TypeScript/React | 좌측 네비게이션, 헤더, 푸터 |
| FG6.1 모델·프롬프트·Capabilities 관리 UI | TypeScript/React | 3개 페이지 + 모달 |
| FG6.1 비용·사용량 대시보드 | TypeScript/React + 차트 라이브러리 |  |
| FG6.2 에이전트 관리 페이지 | TypeScript/React |  |
| FG6.2 Scope Profile 관리 + ACL 필터 빌더 | TypeScript/React |  |
| FG6.2 제안 큐 + Diff 뷰 | TypeScript/React |  |
| FG6.2 에이전트 활동 대시보드 | TypeScript/React + 차트 |  |
| **MCP 통합 가이드** (`mcp_integration_guide.md`) | Markdown | 외부 에이전트 연동 가이드 — DOCS-001 이월 처리 |
| FG6.3 골든셋·평가·추출 관리 UI (스켈레톤) | TypeScript/React | 4개 페이지, 목 데이터 포함 |
| Admin 라우팅 정의 및 네비게이션 | TypeScript | Route 설정, 권한 체크 |
| 공통 Admin UI 컴포넌트 라이브러리 | TypeScript/React | 테이블, 모달, 폼, 차트 유틸 |
| **FG6.1 검수보고서** | Markdown |  |
| **FG6.1 보안취약점검사보고서** | Markdown | 프롬프트 저장/조회 보안, API 인증 확인 |
| **FG6.2 검수보고서** | Markdown |  |
| **FG6.2 보안취약점검사보고서** | Markdown | 에이전트 권한 검증, Scope Profile 필터 우회 방지 |
| **FG6.3 검수보고서** | Markdown |  |
| **FG6.3 보안취약점검사보고서** | Markdown |  |
| **Phase 6 종결 보고서** | Markdown | 통합 콘솔 완성, Phase 7·8 핸드오프 준비 |
| UI 5회 리뷰 기록 | Markdown | 각 FG별 리뷰 회차, 피드백, 개선사항 |

---

## 8. 리스크 및 대응

| 리스크 | 영향도 | 대응 방안 |
|--------|--------|----------|
| FG6.3 스켈레톤 상태에서 Phase 7·8과의 연결 혼동 | 중간 | 명확한 API 계약 문서 사전 정의, Phase 7·8과 주간 동기화 |
| Admin UI 성능 저하 (대량 에이전트·제안 조회) | 중간 | 테이블 가상화(virtualization), 페이지네이션 기본 설정 |
| 5회 리뷰로 인한 일정 압박 | 높음 | 리뷰 팀 사전 구성, 평행 처리 (FG별 병행 리뷰) |
| Admin 권한 검증 누락으로 보안 이슈 | 높음 | 모든 Admin 페이지 접근 시 권한 체크 자동화, 감사 로그 필수 |
| 차트 라이브러리 성능 (많은 데이터 포인트) | 낮음~중간 | 데이터 다운샘플링, 차트 가상화 고려 |

---

## 9. UI 리뷰 전략

Phase 0 FG0.2에서 정의한 체크리스트를 따르되, Admin UI 특화 항목 추가:

### 리뷰 항목 (5회 이상)
1. **네비게이션**: 좌측 메뉴 구조 명확성, 현재 위치 표시, S1/S2 영역 구분
2. **반응형**: mobile(640px), tablet(768px), desktop(1024px+) 모두 정상
3. **폼 입력**: placeholder, 필수 필드 표시, 유효성 검사 피드백
4. **데이터 로딩**: 스켈레톤 로더, 빈 상태 메시지, 오류 상태
5. **접근성**: 키보드 네비게이션, 스크린 리더 호환(aria labels)
6. **시각 계층**: 중요한 작업(승인/반려) vs. 부가 작업(Export) 구분
7. **오류 처리**: 네트워크 오류, API 타임아웃 시 명확한 메시지
8. **성능**: 페이지 로딩 <2초, 상호작용 응답성 <100ms

### 리뷰 회차 일정
- 1회: 와이어프레임 리뷰 (로우파이)
- 2회: 프로토타입 리뷰 (기본 레이아웃)
- 3회: 기능 통합 리뷰 (API 바인딩)
- 4회: 반응형·접근성 리뷰
- 5회: 최종 폴리시 리뷰 (일관성, 오류 처리)

---

## 10. 선행 조건

Phase 6 착수 전:
- Phase 0~5 모든 FG 검수 완료
- Phase 1~5 백엔드 API 안정성 확인 (자동 테스트 통과)
- Admin UI 컴포넌트 라이브러리 초안 준비
- S1 Admin UI 기존 코드베이스 분석 완료

---

## 11. 완료 기준

Phase 6 완료로 판단하는 조건:

1. FG6.1: 모델/프롬프트/Capabilities/비용 관리 UI 완성 및 Phase 1 API 연동
2. FG6.2: 에이전트/Scope/제안 큐/활동 관리 UI 완성 및 Phase 4-5 API 연동
3. FG6.3: 평가·추출 관리 UI 스켈레톤 완성 (목 데이터 포함)
4. Admin 라우팅 및 좌측 네비게이션 통합
5. 모든 FG 최소 5회 UI 리뷰 완료 및 개선사항 반영
6. desktop+web 반응형 호환성 확인
7. 모든 FG 검수보고서 및 보안취약점검사보고서 승인
8. Phase 7·8과의 API 연결 계약 문서 확정
9. Phase 6 종결 보고서 작성 완료

---

## 12. 권장 투입 순서

1. Admin UI 공통 컴포넌트 라이브러리 구축 (병행)
2. S1 Admin UI 기존 구조 리팩토링 (네비게이션 통합)
3. FG6.1 구축 (모델·프롬프트 관리) → 1차 리뷰
4. FG6.1 비용 대시보드 추가 → 2차 리뷰
5. FG6.2 구축 (에이전트·Scope) → 3차 리뷰
6. FG6.2 제안 큐 상세 (Diff 뷰) 추가 → 4차 리뷰
7. FG6.3 스켈레톤 구축 → 5차 리뷰
8. 반응형·접근성 개선
9. 모든 FG 검수 및 보안 검사
10. Phase 6 종결 보고서 작성

---

## 13. 기대 효과

Phase 6 완료 시:
- 관리자가 Mimir의 모든 AI·에이전트·평가 기능을 중앙에서 조작 가능
- S1 Admin 경험과 S2 Admin 경험의 일관된 네비게이션 확보
- Phase 7·8과의 명확한 API 계약으로 후속 통합 위험 최소화
- 5회 리뷰를 통해 상용 수준의 Admin UX 품질 확보
- 폐쇄망·다중 모델 환경에서도 관리 가능한 기반 마련
