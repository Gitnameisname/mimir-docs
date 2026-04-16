# Phase 3 종결 보고서

**작성일**: 2026-04-17  
**최종 갱신**: 2026-04-17 (이월 항목 인계 완료 + Phase 2 TD-003 수정 반영)  
**작성자**: Claude Sonnet 4.6  
**Phase**: 3 — Conversation 도메인  
**종결 판정**: **완료**

---

## 1. 계획 대비 실적

| 항목 | 계획 | 실적 | 편차 | 비고 |
|------|------|------|------|------|
| FG 개수 | 3 | 3 | 0 | FG3.1, FG3.2, FG3.3 |
| Task 개수 | 10 | 10 | 0 | task3-1 ~ task3-10 |
| Critical 이슈 | 0 | 0 | 0 | |
| Major 이슈 발생 | - | 2 | - | 즉시 수정 완료 |
| 단위 테스트 | - | 47/47 (100%) | - | context_window 29 + citation_reuse 18 |
| UI 리뷰 | 5회 | 5회 | 0 | WCAG 2.1 AA 완료 |
| 의무 산출물 | 검수+보안+AI평가+종결 | ✅ 전부 | 0 | |
| PII 배치 잡 | 구현 | 이월 → 인계 완료 | -1 | Phase 4 task4-4 작업지시서에 정식 편입 |
| 대화 FTS | 구현 | 이월 → 인계 완료 | - | Phase 6 task6-1 작업지시서에 정식 편입 |
| Phase 2 TD-003 수정 | - | 완료 | - | `_convert_s1_citations()` nil UUID → chunk version_id |

**종합 평가**: 핵심 기능(도메인 모델, 세션 기반 RAG API, 채팅 UI) 전부 완료. 이월 항목 2건은 후행 Phase 작업지시서에 정식 인계 완료(PH3-CARRY-001 → task4-4, PH3-CARRY-002 → task6-1). Phase 2 이월 기술 부채 TD-003도 이번 Phase에서 추가 수정 완료.

---

## 2. Feature Group별 산출물 현황

| FG | 작업지시서 | 구현 | 검수 | 보안 | AI평가 | 판정 |
|----|----------|------|------|------|-------|------|
| FG3.1 (도메인 모델 + API) | ✅ | ✅ | ✅ | ✅ | - | **승인** (이월 2건 후행 Phase 인계 완료) |
| FG3.2 (세션 기반 RAG API) | ✅ | ✅ | ✅ | ✅ | ✅ | **조건부 승인** (골든셋 측정 Phase 4 이후) |
| FG3.3 (채팅 UI) | ✅ | ✅ | ✅ | ✅ | - | **승인** |

> FG3.1, FG3.3: 직접적인 LLM 생성 기능 없음 → AI 품질 평가 해당 없음 또는 FG3.2 평가에 포함

---

## 3. 신규 파일 목록

### 백엔드

| 파일 | 설명 |
|------|------|
| `app/models/conversation.py` | Conversation / Turn / Message dataclass |
| `app/repositories/conversation_repository.py` | Conversation CRUD (psycopg2) |
| `app/repositories/turn_repository.py` | Turn CRUD (psycopg2) |
| `app/services/context_window_manager.py` | 컨텍스트 윈도우 + ACL + 토큰 관리 |
| `app/services/prompt_builder.py` | 멀티턴 프롬프트 빌더 |
| `app/services/citation_reuse_service.py` | Citation 재활용 보너스 가중치 |
| `app/cache/rag_cache.py` | RAG 전용 캐시 (Valkey → LRU fallback) |
| `app/api/v1/conversations.py` | Conversation CRUD API 엔드포인트 |
| `app/api/v1/rag.py` (확장) | `/rag/answer` + `conversation_id` 지원 |
| `tests/unit/test_context_window.py` | ContextWindowManager 단위 테스트 (29개) |
| `tests/unit/test_citation_reuse.py` | CitationReuseService + RAGCache 단위 테스트 (18개) |
| `alembic/versions/xxx_add_conversations.py` | conversations / turns / messages 테이블 마이그레이션 |

### 프론트엔드

| 파일 | 설명 |
|------|------|
| `src/types/conversation.ts` | Conversation / Turn / Message / RAGCitationInfo 타입 |
| `src/stores/conversationStore.ts` | 채팅 메시지 + 스트리밍 Zustand 스토어 |
| `src/stores/conversationListStore.ts` | 대화 목록 + 검색/필터 Zustand 스토어 |
| `src/lib/api/conversation.ts` | conversationApi + ragAnswerApi |
| `src/lib/citationService.ts` | Citation hash 검증 (1h 캐시) |
| `src/features/chat/ChatPage.tsx` | 반응형 3/2/1 컬럼 레이아웃 |
| `src/app/chat/page.tsx` | `/chat` 라우트 |
| `src/components/chat/ChatWindow.tsx` | 채팅 메인 컨테이너 |
| `src/components/chat/ChatHeader.tsx` | 대화 헤더 (제목 + 옵션) |
| `src/components/chat/MessageHistory.tsx` | 메시지 이력 (role="log") |
| `src/components/chat/MessageBubble.tsx` | user/assistant 버블 (role="article") |
| `src/components/chat/InputArea.tsx` | 채팅 입력 (aria-describedby, 글자 수) |
| `src/components/chat/CitationBlock.tsx` | Citation collapsible 섹션 |
| `src/components/chat/CitationItem.tsx` | Citation 단일 아이템 (원문 링크 + BibTeX) |
| `src/components/chat/CitationVerificationBadge.tsx` | hash 검증 배지 |
| `src/components/chat/ConversationList.tsx` | 대화 목록 사이드바 (role="navigation") |
| `src/components/chat/ConversationItem.tsx` | 단일 대화 항목 (role="button") |
| `src/components/chat/SearchBox.tsx` | 검색 입력 (role="search", debounce 500ms) |
| `src/components/chat/FilterTabs.tsx` | 필터 탭 (role="tablist", 방향키 탐색) |
| `src/components/common/ConfirmDialog.tsx` | 삭제 확인 모달 (role="alertdialog", WCAG 2.1 AA) |

---

## 4. 핵심 설계 결정 기록

### S2 원칙 ⑥ 준수 — ACL은 소유권 기반

`ContextWindowManager.fetch_context_window`에서 `conv.owner_id != actor_id` 시 `PermissionError` 발생. scope 문자열 하드코딩 없음.

### S2 원칙 ⑦ 준수 — 폐쇄망 동등성

`RAGCache`: `EXTERNAL_DEPENDENCIES_ENABLED=false` 시 Valkey 없이 in-memory LRU로 전환. 기능 동일.  
`ContextWindowManager.count_tokens`: tiktoken 없으면 `len(text) // 4` fallback.

### JSON이 아닌 SSE 선택 이유

`/api/v1/rag/answer`는 JSON 응답(RAGResponse)으로 구현. 이유: S1 SSE 엔드포인트(`/rag/query`)와 공존하면서, S2 멀티턴 API는 conversation_id 기반 상태 저장이 필요하므로 원자적 JSON 트랜잭션이 적합. 프론트엔드는 로딩 인디케이터 + 메시지 placeholder로 UX를 구현.

### psycopg2 + dataclass 선택 이유

기존 S2 Phase 1~2와 동일한 패턴 유지. SQLAlchemy ORM 도입 없이 psycopg2 직접 사용. 개발계획서에 SQLAlchemy가 언급되었으나, 기존 코드베이스 패턴을 따라 dataclass + raw SQL로 구현.

---

## 5. 종합 평가

| 항목 | 결과 |
|------|------|
| 계획한 핵심 기능 구현 완료 | ✅ |
| 이월 항목 2건 후행 Phase 작업지시서 인계 | ✅ task4-4(배치 잡), task6-1(FTS) |
| Phase 2 TD-003 추가 수정 | ✅ `_convert_s1_citations()` version_id 정확화 |
| 보안 취약점 전체 수정 (5건/5건) | ✅ |
| 단위 테스트 47/47 (100%) | ✅ |
| UI 5회 리뷰 + WCAG 2.1 AA | ✅ |
| TypeScript 오류 (task 관련 파일) | ✅ 0건 |
| S2 원칙 ⑥⑦ 준수 확인 | ✅ |
| AI 품질 평가 — 골든셋 측정 | ⚠️ Phase 4 이후 재측정 |

---

## 6. 후행 Phase에 전달되는 전제 조건

Phase 4 (Agent-Facing Interface)가 사용할 Phase 3 산출물:
- `POST /api/v1/conversations` — 에이전트 세션 생성
- `POST /api/v1/rag/answer` with `conversation_id` + `actor_type: "agent"` — 에이전트 세션 기반 RAG
- `GET /api/v1/conversations/{id}/turns` — 에이전트 대화 이력 조회
- `ContextWindowManager` — 컨텍스트 윈도우 관리 (에이전트도 동일 인터페이스)
- `RAGCache` — 에이전트 세션 캐시 (actor_id 격리 보장)

---

## 7. 다음 Phase 인계 사항

| 사항 | 우선순위 | 인계 위치 | 상태 |
|------|----------|-----------|------|
| PII 자동 만료 배치 잡 구현 | 높음 | Phase 4 `task4-4` 작업지시서 §2 #9, §4-9 | ✅ 인계 완료 |
| 대화 전문 FTS (tsvector) | 낮음 | Phase 6 `task6-1` 작업지시서 §4-N | ✅ 인계 완료 |
| 골든셋 기반 멀티턴 RAG 품질 측정 | 높음 | Phase 4 이후 (LLM 서버 구동 후) | ⚠️ 별도 추적 필요 |
| React Testing Library 단위 테스트 | 낮음 | 프론트엔드 테스트 인프라 구축 후 | ⚠️ 별도 추적 필요 |
