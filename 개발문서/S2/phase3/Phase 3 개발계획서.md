# Phase 3 개발계획서
## Conversation 도메인

---

## 1. 단계 개요

### 단계명
Phase 3. Conversation 도메인

### 목적
멀티턴 대화를 1급 도메인 객체로 승격하고, 챗봇이 Mimir를 세션 기반 지식 백엔드로 사용할 수 있는 기반을 마련한다. 사용자도 동일한 인터페이스로 대화형 RAG를 경험할 수 있도록 UI를 구축한다.

### 선행 조건
- Phase 1 (모델·프롬프트 추상화) 완료
- Phase 2 (Grounded Retrieval v2 + Citation Contract) 완료
- S1 Phase 11 RAG 엔드포인트 정상 작동
- 대화 세션의 PII 정책 및 보존 기준 합의 완료

### 기대 결과
- Conversation / Turn / Message 도메인 모델 구현 완료
- 세션 기반 RAG API (`/rag/answer` with `conversation_id`) 구현
- 사용자용 멀티턴 채팅 UI 구현 완료
- 대화 이력 조회/검색 API 및 UI 완성
- 챗봇이 Mimir를 세션 기반 지식 백엔드로 사용 가능 상태
- AI품질평가 보고서 (멀티턴 RAG 성능) 완료

---

## 2. Phase 3의 역할

Phase 3은 "대화를 도메인 객체로 만드는" Phase이다. S1에서 RAG는 단발 쿼리-응답 구조였다면, S2의 Phase 3부터 멀티턴 대화가 세션이라는 1급 객체로 존재한다. 이를 통해:

1. **대화 이력 보존**: 멀티턴 질문-응답 시퀀스를 저장하여 나중에 조회/재참조 가능
2. **컨텍스트 윈도우 관리**: 이전 턴의 정보를 압축하거나 캐시하여 후속 턴의 RAG 품질 향상
3. **PII 생명주기 관리**: 사용자 질의에 포함된 민감 정보의 자동 만료 및 수동 삭제 정책 적용
4. **세션 기반 권한 관리**: 대화 세션도 Document와 동일한 ACL 적용
5. **챗봇 백엔드로서의 역할**: 외부 챗봇이 Mimir의 세션 API를 호출하여 상태 관리 위임 가능

---

## 3. Feature Group 상세

### FG3.1 — Conversation / Turn / Message 도메인 모델

#### 목적
멀티턴 대화를 정규화된 도메인 모델로 정의하고, Document와 동일 수준의 권한, 감사, 보존 정책을 적용한다.

#### 주요 작업 항목

1. **Conversation (세션) 도메인 모델 정의**
   - 필드: `id`, `owner_id`, `organization_id`, `created_at`, `updated_at`, `title`, `metadata`
   - `title`: 사용자가 입력하거나 첫 턴의 질문으로 자동 생성
   - `metadata`: 대화 카테고리, 태그, 언어, 도메인 정보 등을 저장 가능하도록 확장 가능
   - 상태: `active` (진행 중), `archived` (보관됨), `deleted` (삭제)
   - 권한 모델: Document와 동일 ACL 적용 (owner, organization, public/private 등)
   - 감사: Conversation 생성/수정/삭제 이벤트 기록

2. **Turn (턴) 도메인 모델 정의**
   - 필드: `id`, `conversation_id`, `turn_number`, `created_at`, `user_message`, `assistant_response`, `retrieval_metadata`
   - `user_message`: 사용자의 질의 (full text, 시간, sender_id)
   - `assistant_response`: AI의 응답 (full text, 생성 시간, 모델명)
   - `retrieval_metadata`: 이번 턴의 검색 결과 (citation 5-tuple 리스트, 검색어 재작성 정보, 컨텍스트 윈도우 상태)

3. **Message (메시지) 도메인 모델 정의**
   - 필드: `id`, `turn_id`, `role` (user/assistant/system), `content`, `created_at`, `metadata`
   - Message는 Turn 내부의 더 세분화된 단계 (실제 프롬프트-완료 교환)
   - 예: 시스템 메시지 → 사용자 메시지 → Assistant 응답 (하나의 Turn) → Message 3개로 표현
   - metadata에 토큰 카운트, 모델명, 비용 정보 저장 가능

4. **PII 생명주기 정책 구현**
   - 대화 보존 기간 설정 가능 (조직별 기본값 예: 90일)
   - 자동 만료: `created_at + retention_period` 이후 자동으로 상태를 `expired`로 전환
   - 수동 삭제 API: 사용자가 특정 Conversation 또는 민감 Turn을 즉시 삭제 가능
   - 감사: 삭제 이벤트 기록 (삭제 사유, 삭제자, 삭제 시간)
   - 로그: PII 감지 규칙 (선택사항): 질의에서 이메일, 전화번호, SSN 등 패턴 감지 → 메타데이터에 플래그

5. **DB 스키마 설계**
   - `conversations` 테이블: id, owner_id, organization_id, created_at, updated_at, title, status, metadata (JSONB), retention_days, expires_at
   - `turns` 테이블: id, conversation_id, turn_number, user_message, assistant_response, retrieval_metadata (JSONB), created_at
   - `messages` 테이블: id, turn_id, role, content, metadata (JSONB), created_at
   - 인덱스: (conversation_id, turn_number), (owner_id, created_at), (expires_at)
   - Soft delete 지원: `deleted_at` nullable 필드 포함

6. **대화 이력 조회/검색 API**
   - `GET /api/v1/conversations`: 사용자의 대화 목록 조회 (페이지네이션, 정렬)
   - `GET /api/v1/conversations/{conversation_id}`: 특정 대화 상세 조회
   - `GET /api/v1/conversations/{conversation_id}/turns`: 대화의 모든 턴 조회
   - `GET /api/v1/conversations/{conversation_id}/turns/{turn_id}`: 특정 턴 상세 조회
   - `DELETE /api/v1/conversations/{conversation_id}`: 대화 삭제 (물리 삭제 또는 soft delete)
   - `POST /api/v1/conversations/{conversation_id}/turns/{turn_id}/redact`: 특정 턴의 민감 정보 제거

#### 입력 (선행 의존)
- Phase 1 완료 (모델 추상화)
- Phase 2 완료 (Citation Contract)
- S1 Phase 11 RAG 응답 포맷 분석
- PII 정책 및 보존 기간 합의 문서

#### 출력 (산출물)
- Conversation/Turn/Message 도메인 모델 정의서 (ER diagram 포함)
- DB 마이그레이션 스크립트 (alembic)
- 도메인 모델 구현체 (SQLAlchemy ORM 클래스)
- 대화 이력 조회/검색 API 엔드포인트 (FastAPI)
- PII 정책 설정 관리 API
- 자동 만료 배치 작업 (Celery 또는 APScheduler)
- 단위 테스트 (test_conversations_crud.py 등)

#### 검수 기준
- Conversation/Turn/Message 스키마가 정규화되었는가
- 권한 모델이 Document와 동일 수준인가
- PII 자동 만료 로직이 정확하게 작동하는가
- 수동 삭제 API가 감사 로그를 기록하는가
- 대화 검색이 full-text search를 지원하는가
- retention_days 설정이 조직별로 커스터마이징 가능한가

---

### FG3.2 — 세션 기반 RAG API

#### 목적
기존 `/rag/answer` 엔드포인트를 확장하여 `conversation_id`를 지원하고, 멀티턴 컨텍스트를 효율적으로 관리한다.

#### 주요 작업 항목

1. **세션 기반 RAG API 설계**
   - 기존 `/rag/answer` (단발 RAG) 유지
   - 새로운 `/rag/answer` 호출 시 요청 본문에 선택사항 `conversation_id` 포함
   - `conversation_id`가 없으면: 기존 단발 RAG 동작
   - `conversation_id`가 있으면: Turn 생성 후 멀티턴 모드로 동작

2. **멀티턴 RAG 컨텍스트 윈도우 관리**
   - 이전 Turn의 검색 결과 캐시: 같은 쿼리면 재검색 대신 캐시 활용
   - 컨텍스트 압축 전략:
     - 최근 N개 Turn만 프롬프트에 포함 (기본값: 최근 3턴)
     - 오래된 Turn은 요약 또는 완전 제거
   - 토큰 카운팅: Phase 1의 모델 추상화 사용하여 프롬프트 토큰 계산
   - 컨텍스트 윈도우 오버플로우 시: 가장 오래된 Turn부터 제거하거나 요약

3. **쿼리 재작성 로직 통합 (Phase 2에서 구축한 것 활용)**
   - 멀티턴 대화에서 follow-up query를 자립적 쿼리로 재작성 (Phase 2 FG2.3)
   - 재작성된 쿼리로 검색 수행
   - 원문 쿼리와 재작성 쿼리 모두 Turn의 metadata에 저장

4. **이전 턴 Citation 재활용 전략**
   - 이전 Turn에서 인용된 Document가 현재 쿼리에도 관련 있으면 우선 순위 상향
   - 예: Turn 1에서 document#10 인용 → Turn 2 검색 시 document#10을 1순위로 고려
   - Reranker 점수에 보너스 가중치 추가

5. **세션 기반 RAG 요청/응답 스키마**
   - Request:
     ```json
     {
       "query": "string (필수)",
       "conversation_id": "uuid (선택)",
       "user_id": "uuid (필수)",
       "top_k": "int (기본: 5)",
       "context_window_size": "int (기본: 최근 3턴)",
       "model": "string (기본: Phase 1에서 설정된 기본 모델)"
     }
     ```
   - Response:
     ```json
     {
       "answer": "string",
       "citations": [
         {
           "document_id": "uuid",
           "version_id": "uuid",
           "node_id": "string",
           "span_offset": "int (optional)",
           "content_hash": "string",
           "content": "string"
         }
       ],
       "turn_id": "uuid (새 턴 생성 시)",
       "retrieval_metadata": {
         "query_rewritten": "string",
         "retrieval_time_ms": "int",
         "context_window_turns": ["turn_id1", "turn_id2"]
       }
     }
     ```

6. **성능 최적화**
   - 이전 Turn의 검색 결과 캐시 (Redis 또는 in-memory)
   - Citation 5-tuple 검증 캐시
   - 프롬프트 토큰 계산 캐시 (같은 conversation이면 delta만 계산)

#### 입력 (선행 의존)
- Phase 2 완료 (Citation 5-tuple, 쿼리 재작성)
- FG3.1 완료 (Conversation/Turn/Message 도메인)
- Phase 1 모델 추상화 (LLM 선택, 토큰 카운팅)

#### 출력 (산출물)
- 세션 기반 RAG API 엔드포인트 업데이트 (FastAPI)
- 컨텍스트 윈도우 관리 로직 (Python 클래스)
- 쿼리 재작성 통합 (Phase 2에서 구축한 것을 RAG 계층에 통합)
- Citation 캐싱 전략 구현
- 단위 테스트 (test_conversation_rag.py 등)
- 성능 벤치마크 보고서 (단발 vs 멀티턴 응답 시간)

#### 검수 기준
- `/rag/answer`에 `conversation_id` 매개변수 추가 및 하위호환성 유지
- 멀티턴 Turn 생성 시 conversation_id 자동 할당
- 컨텍스트 윈도우가 토큰 제한 내에서 작동하는가
- 쿼리 재작성 결과가 Turn metadata에 저장되는가
- Citation 캐시가 정확하게 작동하는가
- 응답 시간이 단발 RAG 대비 합리적인 수준인가 (예: <500ms 추가 소비)

---

### FG3.3 — User UI 채팅 인터페이스

#### 목적
사용자가 Mimir와 멀티턴 대화를 나누는 친화적 UI를 제공한다. 턴별 Citation이 명확하게 표시되고 원문 참조 가능해야 한다.

#### 주요 작업 항목

1. **채팅 UI 기본 구조**
   - 화면 레이아웃: 좌측 "대화 목록", 중앙 "현재 대화", 우측 "참고 자료" (선택사항)
   - 상단: 현재 대화 제목, 내보내기, 삭제 버튼
   - 중앙: 턴별 메시지 시퀀스 (user bubble, assistant bubble)
   - 하단: 질문 입력 필드, 전송 버튼, 로딩 상태

2. **턴별 메시지 렌더링**
   - 사용자 메시지 (user bubble): 질문 텍스트
   - 어시스턴트 응답 (assistant bubble): 답변 텍스트 (마크다운 지원)
   - 스트리밍 응답 지원: Phase 1 LLM 추상화에서 Streamable HTTP 또는 SSE 사용

3. **턴별 Citation 표시**
   - 답변 하단에 "인용 출처" 섹션 추가
   - 각 Citation마다:
     - 문서명, 버전, 노드 ID 표시
     - "원문 보기" 링크 → 클릭 시 Document의 해당 노드로 이동
     - span_offset이 있으면, 원문의 정확한 위치에 하이라이트 처리
     - content_hash를 시각적 신뢰도 표시 (체크마크 또는 배지)
   - Citation이 없는 답변: 경고 표시 ("근거 없음" 배지)

4. **대화 이력 목록**
   - 좌측 사이드바: 이전 대화 목록 (최근순)
   - 각 항목: 제목, 생성 날짜, 마지막 업데이트 시간
   - 검색: 대화 제목/내용으로 검색 가능 (전체 텍스트 검색)
   - 필터: 전체, 지난 7일, 지난 30일, 보관함, 삭제됨 (soft delete 조회)

5. **새 대화 시작**
   - "새로운 대화" 버튼 → 새 Conversation 생성
   - 첫 질문 입력 시 자동으로 제목 생성 (첫 질문의 일부를 사용 또는 AI 요약)
   - 빈 대화 상태: "질문을 입력하여 시작하세요" 프롬프트

6. **대화 관리 기능**
   - 대화 제목 수정
   - 대화 아카이빙 (soft delete)
   - 대화 영구 삭제 (재확인 필요)
   - 대화 공유 (Phase 6에서 공유 링크 생성 기능 추가 가능)
   - 대화 내보내기 (JSON 또는 Markdown 포맷)

7. **반응형 디자인 및 접근성**
   - 데스크톱: 3-컬럼 레이아웃 (대화목록 / 채팅 / 참고자료)
   - 태블릿: 2-컬럼 (대화목록 / 채팅)
   - 모바일: 1-컬럼 (대화목록은 오프캔버스 메뉴, 또는 탭으로 전환)
   - WCAG 2.1 AA 준수 (색상 대비, 키보드 네비게이션, 스크린 리더 지원)
   - 키보드 단축키: Enter 전송, Shift+Enter 줄바꿈

8. **UI 5회 리뷰 프로세스**
   - 리뷰 1: 와이어프레임 및 정보 아키텍처 (구조가 타당한가)
   - 리뷰 2: 시각 디자인 및 색상 스키마 (브랜드와 부합하는가)
   - 리뷰 3: Citation 렌더링 및 원문 링크 정확성 (UX가 명확한가)
   - 리뷰 4: 반응형 breakpoint 및 모바일 경험 (모든 화면에서 동작하는가)
   - 리뷰 5: 접근성 및 오류 처리 (장애인도 사용할 수 있는가)

#### 입력 (선행 의존)
- FG3.1 완료 (Conversation/Turn API)
- FG3.2 완료 (세션 기반 RAG API)
- Phase 2 Citation Contract 정의 (5-tuple)
- 프론트엔드 프레임워크 (기존 React 스택 사용)

#### 출력 (산출물)
- 채팅 UI 컴포넌트 (React/TypeScript)
  - `<ChatWindow>` (main chat area)
  - `<ConversationList>` (left sidebar)
  - `<MessageBubble>` (user/assistant messages)
  - `<CitationBlock>` (inlined citations per turn)
  - `<CitationVerificationBadge>` (content_hash 검증 표시)
- API 호출 계층 (unwrapEnvelope 패턴 사용)
- 반응형 스타일 (CSS-in-JS 또는 Tailwind)
- 접근성 테스트 (axe, WAVE, 키보드 테스트)
- UI 리뷰 기록 및 개선 사항 문서
- 단위 테스트 (React Testing Library)
- 스토리북 컴포넌트 카탈로그 (선택사항)

#### 검수 기준
- 채팅 UI에서 질문 입력 및 응답 조회가 정상 작동하는가
- Citation이 턴별로 명확하게 표시되는가
- "원문 보기" 링크가 올바른 Document로 이동하는가
- 대화 목록 검색이 정확하게 작동하는가
- 반응형 디자인이 모바일/태블릿에서 정상 작동하는가
- 최소 5회 UI 리뷰 기록 및 개선 사항이 반영되었는가
- WCAG 2.1 AA 접근성 기준 충족하는가
- 장시간 대화(100+ 턴)에서 성능 저하가 없는가

---

## 4. 기술 설계 요약

### 4.1 Conversation 도메인 아키텍처
```
Conversation (1)
  ├── Turn (N)
  │   ├── user_message: str
  │   ├── assistant_response: str
  │   └── retrieval_metadata: {citation_5tuple[], query_rewritten, context_window_turns}
  └── Message (N*M) [세분화된 프롬프트 교환]
      ├── role: user | assistant | system
      └── content: str
```

### 4.2 멀티턴 RAG 파이프라인
```
User Query
  ├─ Phase 2 쿼리 재작성 (conversation_id 있으면 이전 턴 컨텍스트 고려)
  ├─ Retrieval (캐시 확인 → FTS/Vector)
  ├─ Reranking (이전 Citation 가중치 적용)
  ├─ Prompt 구성 (컨텍스트 윈도우 최근 N턴 + 현재 검색 결과)
  └─ LLM 생성 (Phase 1 모델 추상화) → Turn 저장 → Response
```

### 4.3 PII 생명주기 관리
```
대화 생성 (expires_at = created_at + retention_days)
  ↓
배치 작업 (매 시간): expires_at < now() 인 대화를 상태 'expired'로 전환
  ↓
수동 삭제 API: 사용자 또는 관리자가 특정 대화/턴 즉시 삭제 가능
  ↓
감사 로그: 삭제 이벤트 기록
```

### 4.4 UI 아키텍처 (React)
```
App
  ├─ ConversationList (좌측 사이드바)
  │   ├─ SearchBox
  │   ├─ FilterTabs (전체/지난7일/보관함)
  │   └─ ConversationItem (N)
  └─ ChatWindow (중앙)
      ├─ ChatHeader (제목, 옵션 메뉴)
      ├─ MessageHistory
      │   └─ MessageBubble (N)
      │       ├─ role: user | assistant
      │       ├─ content (마크다운)
      │       └─ CitationBlock [assistant만]
      │           └─ Citation (5-tuple) (N)
      └─ InputArea
          ├─ TextInput (multiline)
          └─ SendButton (streaming 지원)
```

---

## 5. 의존 관계

### 선행 Phase
- **Phase 1** (모델·프롬프트 추상화): LLM 선택, 토큰 카운팅
- **Phase 2** (Grounded Retrieval v2): Citation 5-tuple, 쿼리 재작성

### 후행 Phase (이 Phase의 산출물을 소비하는 Phase)
- **Phase 4** (Agent-Facing Interface): 에이전트는 Conversation API를 통해 세션 관리 위임받음
- **Phase 5** (Agent Action Plane): 에이전트가 Draft 제안 시 대화 컨텍스트 참조 가능
- **Phase 6** (관리자 기능 통합): 대화 분석 대시보드, 대화 메타데이터 관리 UI
- **Phase 7** (AI 품질 평가): 멀티턴 RAG 품질 평가 (faithfulness, answer relevance 등)

---

## 6. 검수 기준 종합

| 항목 | 기준 |
|------|------|
| **FG3.1 - 도메인 모델** | Conversation/Turn/Message 스키마가 정규화되고 권한 모델이 Document와 동등한가 |
| **FG3.1 - PII 정책** | 자동 만료가 정확하게 작동하는가 / 수동 삭제 API가 감사 로그를 기록하는가 |
| **FG3.1 - API** | 대화 조회/검색 API가 페이지네이션 및 필터를 지원하는가 |
| **FG3.2 - 세션 기반 RAG** | `/rag/answer`에 `conversation_id`가 추가되고 하위호환성이 유지되는가 |
| **FG3.2 - 컨텍스트 관리** | 토큰 오버플로우 시 자동으로 오래된 Turn을 제거하는가 |
| **FG3.2 - 성능** | 멀티턴 응답 시간이 합리적인가 (<500ms 추가) |
| **FG3.3 - 채팅 UI** | 사용자가 질문 입력 및 응답 조회를 정상적으로 수행할 수 있는가 |
| **FG3.3 - Citation 표시** | 각 Turn의 Citation이 명확하게 표시되고 원문 링크가 정확한가 |
| **FG3.3 - 반응형** | 모바일/태블릿/데스크톱 모든 화면에서 정상 작동하는가 |
| **FG3.3 - 접근성** | WCAG 2.1 AA 준수하고 키보드 네비게이션 가능한가 |
| **FG3.3 - UI 리뷰** | 최소 5회 리뷰 기록 및 개선 사항 반영되었는가 |
| **종합** | 챗봇이 Conversation API를 통해 세션을 관리할 수 있는가 |

---

## 7. 예상 산출물 목록

| 산출물 | 형태 | 설명 |
|--------|------|------|
| Conversation/Turn/Message 도메인 모델 정의서 | Markdown (ER diagram 포함) | 스키마 명세 |
| DB 마이그레이션 스크립트 | Python (alembic) | 테이블 생성, 인덱스 정의 |
| ORM 클래스 | Python (SQLAlchemy) | Conversation, Turn, Message 모델 |
| 대화 CRUD API | FastAPI 라우터 | GET/POST/DELETE /conversations, /turns |
| 세션 기반 RAG API | FastAPI 라우터 | PUT /rag/answer with conversation_id |
| 컨텍스트 윈도우 관리 로직 | Python 클래스 | 토큰 카운팅, 윈도우 압축 전략 |
| 배치 작업 (PII 만료) | Python (APScheduler) | 자동 만료 스케줄 |
| 프론트엔드 컴포넌트 | React/TypeScript | ChatWindow, ConversationList, MessageBubble 등 |
| API 호출 계층 | TypeScript | conversationApi.ts, ragApi.ts |
| 반응형 스타일 | CSS-in-JS (Tailwind) | 모바일/태블릿/데스크톱 breakpoint |
| 접근성 테스트 결과 | 보고서 | WCAG 2.1 AA 준수 확인 |
| 단위 테스트 | pytest, React Testing Library | test_conversations_crud.py, ChatWindow.test.tsx |
| UI 리뷰 기록 | Markdown | 5회 리뷰 항목, 개선 사항 |
| 성능 벤치마크 | 보고서 | 단발 vs 멀티턴 응답 시간 비교 |
| **FG3.1 검수보고서** | Markdown | 도메인 모델, API, PII 정책 검수 |
| **FG3.1 보안취약점검사보고서** | Markdown | PII 생명주기, 권한 모델 보안 검토 |
| **FG3.2 검수보고서** | Markdown | 세션 기반 RAG, 컨텍스트 관리 검수 |
| **FG3.2 보안취약점검사보고서** | Markdown | 멀티턴 프롬프트 인젝션 방어, 캐시 보안 |
| **FG3.3 검수보고서** | Markdown | UI 기능, Citation 표시, 반응형 검수 |
| **FG3.3 보안취약점검사보고서** | Markdown | XSS (마크다운), 접근성 보안 검토 |
| **FG3.1/3.2/3.3 AI품질평가보고서** | Markdown | 멀티턴 RAG faithfulness, answer relevance, citation-present rate 측정 |
| **Phase 3 종결 보고서** | Markdown | Phase 3 전체 완료 보고 |

---

## 8. 리스크 및 대응

| 리스크 | 영향도 | 대응 방안 |
|--------|--------|----------|
| PII 자동 만료 배치 오류로 대화 데이터 손실 | 높음 | 배치 잡 dry-run 모드 추가, 삭제 전 백업 |
| 장시간 대화(100+ 턴)에서 컨텍스트 윈도우 오버플로우 | 중간 | 토큰 카운팅 정확도 향상, 윈도우 크기 동적 조정 |
| Citation 캐시 과다 메모리 사용 | 중간 | LRU 캐시로 제한, TTL 설정 (1시간) |
| 멀티턴 응답 시간 급증 (검색 N번 수행) | 중간 | 이전 결과 캐시, 검색 병렬화, 쿼리 샘플링 |
| UI 5회 리뷰 일정 지연 | 낮음~중간 | 리뷰 팀 확대, 병행 리뷰 |
| 반응형 디자인 모바일 경험 저하 | 낮음 | 모바일 우선 개발, 조기 테스트 |
| Citation 원문 링크 404 (문서 이동/삭제) | 낮음 | 404 페이지 안내, 버전 기반 링크 사용 |
| 사용자 대화 데이터 노출 (권한 모델 버그) | 높음 | 권한 테스트 강화, 감사 로그 분석 |

---

## 9. 선행 조건

Phase 3을 시작하려면:
- Phase 1 완료 (LLM 추상화, 토큰 카운팅)
- Phase 2 완료 (Citation 5-tuple, 쿼리 재작성)
- PII 정책 및 보존 기간 합의 (보안 팀, 법무 팀 검토)
- S1 Phase 11 RAG API 검증 완료
- 프론트엔드 환경 설정 (React, TypeScript, 상태 관리)

---

## 10. 완료 기준

Phase 3 완료로 판단하는 조건:

1. Conversation/Turn/Message 도메인 모델 완전 구현
2. 대화 CRUD API 모두 정상 작동
3. `/rag/answer`에 `conversation_id` 매개변수 추가 및 하위호환성 유지
4. 멀티턴 Turn 생성 및 조회 정상 작동
5. PII 자동 만료 배치 잡 정상 작동
6. 채팅 UI에서 질문 입력 및 응답 조회 정상 작동
7. Citation이 턴별로 명확하게 표시되고 원문 링크 정확
8. 반응형 디자인 모바일/태블릿/데스크톱 정상 작동
9. 최소 5회 UI 리뷰 기록 및 개선 반영
10. WCAG 2.1 AA 접근성 기준 충족
11. 성능 벤치마크 통과 (멀티턴 응답 <500ms 추가 소비)
12. 모든 FG 검수보고서 및 보안취약점검사보고서 승인
13. AI품질평가보고서 (멀티턴 RAG 품질 지표 측정) 완료
14. Phase 3 종결 보고서 작성 완료

---

## 11. 예상 투입 규모

- **백엔드 개발**: 2주 (DB 스키마 + API + 배치 작업)
- **프론트엔드 개발**: 2주 (UI 컴포넌트 + API 호출)
- **테스트 및 검수**: 1주 (단위/통합 테스트 + UI 리뷰 5회)
- **AI 품질 평가**: 1주 (골든셋 기반 멀티턴 RAG 평가)
- **총 투입**: 약 6주 (병행 가능 부분은 압축 가능)

