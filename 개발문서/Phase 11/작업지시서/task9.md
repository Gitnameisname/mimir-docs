# Task 11-9. 대화 이력 관리 (Multi-turn)

## 1. 작업 목적

RAG 질의에서 이전 대화 맥락을 유지하여 자연스러운 Multi-turn 대화를 지원한다.

이 작업의 목표는 다음과 같다.

- rag_conversations / rag_messages 테이블 구현
- 대화 이력 로드 및 컨텍스트 압축 로직
- 대화 세션 수명 주기 관리
- 대화 이력 기반 Standalone Question 생성 연동

---

## 2. 작업 범위

### 포함 범위

- rag_conversations 테이블 마이그레이션
- rag_messages 테이블 마이그레이션
- ConversationService 구현
- 대화 이력 컨텍스트 압축 로직

### 제외 범위

- RAG Query API (Task 11-8에서 다룸)
- UI 통합 (Task 11-10에서 다룸)

---

## 3. 선행 조건

- Task 11-1 (아키텍처 설계) 완료 — 대화 세션 데이터 구조 확정
- Task 11-2 (QueryProcessor) 완료 — conversation_history 입력 형식 확정

---

## 4. 주요 구현 대상

### 4-1. DB 스키마

```sql
CREATE TABLE rag_conversations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title           VARCHAR(200),
  created_at      TIMESTAMP WITH TIME ZONE DEFAULT now(),
  updated_at      TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE INDEX idx_rag_conv_user ON rag_conversations(user_id);

CREATE TABLE rag_messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES rag_conversations(id) ON DELETE CASCADE,
  role            VARCHAR(20) NOT NULL CHECK (role IN ('user', 'assistant')),
  content         TEXT NOT NULL,
  citations       JSONB DEFAULT '[]',
  context_chunks  JSONB DEFAULT '[]',
  token_used      INTEGER,
  model           VARCHAR(100),
  created_at      TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE INDEX idx_rag_msg_conv ON rag_messages(conversation_id, created_at);
```

---

### 4-2. ConversationService

```
ConversationService
  .get_or_create(conversation_id: UUID | None, user_id: UUID) → Conversation
  .load_history(conversation_id: UUID, max_turns: int) → Message[]
  .save_message(conversation_id: UUID, role: str, content: str, ...) → Message
  .update_title(conversation_id: UUID, title: str) → None
  .list_conversations(user_id: UUID, page, page_size) → Page[Conversation]
  .delete_conversation(conversation_id: UUID, user_id: UUID) → None
```

---

### 4-3. 대화 이력 로드

```python
def load_history(self, conversation_id, max_turns=5) -> list[Message]:
    messages = db.query("""
        SELECT role, content FROM rag_messages
        WHERE conversation_id = :id
        ORDER BY created_at DESC
        LIMIT :limit
    """, {"id": conversation_id, "limit": max_turns * 2})

    # 최신순 → 시간순 역정렬
    return list(reversed(messages))
```

max_turns=5이면 최근 5턴(user+assistant 쌍 5개 = 최대 10개 메시지) 로드.

---

### 4-4. 대화 제목 자동 생성

첫 번째 사용자 메시지 저장 시 대화 제목 자동 생성:

```python
def _generate_title(first_question: str) -> str:
    # 첫 질문 50자로 자르기
    title = first_question.strip()
    if len(title) > 50:
        title = title[:47] + "..."
    return title
```

---

### 4-5. 대화 세션 수명 주기

| 이벤트 | 처리 |
|-------|------|
| 새 질문 (conversation_id 없음) | 새 Conversation 생성 |
| 이어서 질문 (conversation_id 있음) | 기존 Conversation에 Message 추가 |
| 사용자 삭제 요청 | Conversation + Messages CASCADE 삭제 |
| 사용자 계정 삭제 | ON DELETE CASCADE로 자동 삭제 |

---

### 4-6. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| 신규 대화 생성 | Conversation 생성, 제목=None |
| 첫 메시지 저장 | 제목 자동 생성 |
| 이력 로드 (max_turns=2) | 최근 2턴 (4개 메시지) |
| 타인 대화 삭제 시도 | 403 예외 |

---

## 5. 산출물

1. rag_conversations / rag_messages DB 마이그레이션
2. ConversationService 구현
3. 단위 테스트

---

## 6. 완료 기준

- rag_conversations, rag_messages 테이블이 생성된다
- 대화 이력이 저장되고 조회된다
- max_turns 설정으로 이력 깊이를 조절할 수 있다
- 대화 제목이 첫 질문으로 자동 생성된다
- 타인의 대화 세션에 접근 불가하다

---

## 7. Codex 작업 지침

- max_turns 기본값(5)은 설정으로 조정 가능하게 한다
- 대화 이력이 길어질수록 LLM 입력 토큰이 증가하므로, QueryProcessor의 Standalone Question 생성으로 압축하는 것이 핵심이다
- citations 및 context_chunks는 JSONB로 저장하되, 청크 전체 텍스트는 제외하고 참조 정보(id, title, node_path 등)만 저장하여 DB 용량을 절약한다
