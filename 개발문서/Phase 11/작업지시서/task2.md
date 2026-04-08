# Task 11-2. QueryProcessor 구현

## 1. 작업 목적

사용자의 자연어 질문을 RAG 파이프라인이 처리할 수 있는 형태로 변환하는 QueryProcessor를 구현한다.

이 작업의 목표는 다음과 같다.

- 쿼리 정규화 및 전처리
- 임베딩 변환 (Phase 10 EmbeddingProvider 재사용)
- 대화 이력 컨텍스트 압축 (Multi-turn 지원)
- 단위 테스트 작성

---

## 2. 작업 범위

### 포함 범위

- QueryProcessor 클래스 구현
- 쿼리 정규화 로직
- 대화 이력 컨텍스트 주입 (Standalone Question 생성)
- 임베딩 변환 연동

### 제외 범위

- Retriever 구현 (Task 11-3에서 다룸)
- LLM 연동 (Task 11-5에서 다룸)

---

## 3. 선행 조건

- Task 11-1 (아키텍처 설계) 완료
- Phase 10 Task 10-4 (EmbeddingProvider) 완료 — embed_single() 사용

---

## 4. 주요 구현 대상

### 4-1. QueryProcessor 인터페이스

```
QueryProcessor
  .process(query: UserQuery) → ProcessedQuery

UserQuery
  text: str                          (사용자 입력)
  conversation_history: Message[]    (이전 대화, Multi-turn 시)
  user_roles: list[str]
  user_org_ids: list[str]

ProcessedQuery
  original_text: str
  standalone_text: str               (대화 이력 컨텍스트 반영 후 독립 질문)
  embedding: float[]
  user_roles: list[str]
  user_org_ids: list[str]
```

---

### 4-2. 쿼리 정규화

```python
def _normalize(text: str) -> str:
    # 1. 앞뒤 공백 제거
    text = text.strip()
    # 2. 연속 공백 단일화
    text = re.sub(r'\s+', ' ', text)
    # 3. 최대 길이 제한 (1000자)
    if len(text) > 1000:
        text = text[:1000]
        logger.warning("Query truncated to 1000 chars")
    return text
```

---

### 4-3. Standalone Question 생성 (Multi-turn)

대화 이력이 있을 때 LLM을 활용하여 이전 맥락이 포함된 독립 질문 생성:

```
이전 대화:
  User: 정보보안 정책이 뭐야?
  Assistant: 정보보안 정책은 ...

현재 질문: "거기서 접근통제 부분만 다시 설명해줘"

→ Standalone Question: "정보보안 정책에서 접근통제 부분을 설명해주세요"
```

#### Standalone 생성 프롬프트

```
시스템: 다음 대화 이력과 후속 질문이 주어졌을 때, 대화 이력 없이도 이해할 수 있는 독립적인 질문으로 재작성하세요. 질문만 출력하세요.

대화 이력: {history}
후속 질문: {question}
독립 질문:
```

대화 이력이 없거나 첫 질문이면 원본 질문을 standalone_text로 사용.

---

### 4-4. 임베딩 변환

standalone_text를 EmbeddingProvider로 임베딩 변환:

```python
def _embed(text: str) -> list[float]:
    result = EmbeddingProvider.embed_single(text)
    return result.embedding
```

---

### 4-5. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| 단순 질문 (대화 이력 없음) | standalone_text = 정규화된 원본 질문 |
| 대화 이력 있는 후속 질문 | standalone_text = 독립적인 질문으로 재작성 |
| 빈 질문 | 예외 발생 |
| 1000자 초과 질문 | 자르기 후 처리, 경고 로그 |
| 임베딩 변환 | embedding 필드 길이 = 1536 |

---

## 5. 산출물

1. QueryProcessor 구현 코드
2. UserQuery / ProcessedQuery 데이터 클래스
3. 단위 테스트

---

## 6. 완료 기준

- QueryProcessor가 UserQuery를 ProcessedQuery로 변환한다
- 대화 이력이 있을 때 Standalone Question이 생성된다
- 임베딩이 ProcessedQuery에 포함된다
- 단위 테스트가 통과한다

---

## 7. Codex 작업 지침

- Standalone Question 생성에 사용하는 LLM 호출은 가볍고 빠른 모델(GPT-4o-mini 등)을 사용하여 비용을 최소화한다
- 대화 이력의 최대 포함 턴 수는 설정으로 관리한다 (기본 5턴)
- 쿼리 길이 제한 및 정규화 기준은 설정으로 조정 가능하게 한다
