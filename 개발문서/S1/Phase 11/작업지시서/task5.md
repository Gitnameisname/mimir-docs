# Task 11-5. LLMProvider 추상화 및 GPT-4o 연동

## 1. 작업 목적

LLM 모델을 교체 가능한 LLMProvider 추상화 레이어를 설계하고, OpenAI GPT-4o 기본 구현체를 제공한다.

이 작업의 목표는 다음과 같다.

- LLMProvider 인터페이스 설계
- OpenAILLMProvider 구현 (스트리밍 지원)
- 토큰 사용량 추적 및 비용 모니터링
- 오류 처리 및 재시도 로직 구현

---

## 2. 작업 범위

### 포함 범위

- LLMProvider 인터페이스 설계
- OpenAILLMProvider 구현
- 스트리밍 응답 (SSE)
- 토큰 사용량 기록
- API 오류 처리 및 재시도 로직
- 단위 테스트 (Mock 기반)

### 제외 범위

- ContextBuilder / 프롬프트 조합 (Task 11-6에서 다룸)
- 실제 RAG 흐름 통합 (Task 11-8에서 다룸)

---

## 3. 선행 조건

- Task 11-1 (아키텍처 설계) 완료 — LLMProvider 인터페이스 초안 참조
- Phase 10 Task 10-4 (EmbeddingProvider) — 오류 처리 패턴 참조

---

## 4. 주요 구현 대상

### 4-1. LLMProvider 인터페이스

```
LLMProvider (인터페이스)
  .complete(prompt: LLMPrompt) → LLMResponse
  .stream(prompt: LLMPrompt) → AsyncIterator[str]
  .get_model_name() → str
  .get_context_limit() → int

LLMPrompt
  system: str
  messages: LLMMessage[]
  max_tokens: int = 2048
  temperature: float = 0.1

LLMMessage
  role: "user" | "assistant"
  content: str

LLMResponse
  content: str
  prompt_tokens: int
  completion_tokens: int
  total_tokens: int
  model: str
  finish_reason: str
```

---

### 4-2. OpenAILLMProvider 구현

#### 설정

```
model: "gpt-4o"  (기본)
max_tokens: 2048
temperature: 0.1
max_retries: 3
retry_delay_seconds: 1.0
```

#### 비스트리밍 complete()

```python
def complete(self, prompt: LLMPrompt) -> LLMResponse:
    response = openai.chat.completions.create(
        model=self.model,
        messages=self._build_messages(prompt),
        max_tokens=prompt.max_tokens,
        temperature=prompt.temperature,
    )
    self.record_usage(response.usage)
    return LLMResponse(
        content=response.choices[0].message.content,
        prompt_tokens=response.usage.prompt_tokens,
        completion_tokens=response.usage.completion_tokens,
        total_tokens=response.usage.total_tokens,
        model=self.model,
        finish_reason=response.choices[0].finish_reason,
    )
```

#### 스트리밍 stream()

```python
async def stream(self, prompt: LLMPrompt) -> AsyncIterator[str]:
    response = await openai.chat.completions.create(
        model=self.model,
        messages=self._build_messages(prompt),
        max_tokens=prompt.max_tokens,
        temperature=prompt.temperature,
        stream=True,
    )
    async for chunk in response:
        delta = chunk.choices[0].delta
        if delta.content:
            yield delta.content
```

---

### 4-3. 토큰 사용량 추적

```
LLMUsageLog
  id
  conversation_id (선택)
  model: str
  prompt_tokens: int
  completion_tokens: int
  total_tokens: int
  estimated_cost_usd: float
  created_at
```

#### 비용 계산 기준 (2025년 기준)

| 모델 | 입력 | 출력 |
|------|------|------|
| gpt-4o | $5.00 / 1M tokens | $15.00 / 1M tokens |
| gpt-4o-mini | $0.15 / 1M tokens | $0.60 / 1M tokens |

---

### 4-4. 오류 처리

| 오류 유형 | 처리 방식 |
|----------|----------|
| RateLimitError | Exponential backoff 재시도 (최대 3회) |
| APIConnectionError | 즉시 재시도 후 실패 처리 |
| ContextWindowExceededError | 컨텍스트 축소 후 재시도 |
| AuthenticationError | 즉시 실패 (재시도 불가) |
| 기타 서버 오류 | 재시도 후 실패 처리 |

---

### 4-5. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| 기본 complete() | LLMResponse 반환 |
| 스트리밍 stream() | 문자열 청크 순차 반환 |
| RateLimitError 발생 | 재시도 후 성공 |
| AuthenticationError | 즉시 예외 발생 |
| 토큰 사용량 기록 | LLMUsageLog DB 저장 |

---

## 5. 산출물

1. LLMProvider 인터페이스 구현
2. OpenAILLMProvider 구현 (스트리밍 포함)
3. LLMUsageLog 저장 구현
4. 단위 테스트 (Mock API 기반)

---

## 6. 완료 기준

- LLMProvider 인터페이스가 정의되어 있다
- OpenAILLMProvider가 비스트리밍 / 스트리밍 모두 동작한다
- 토큰 사용량이 LLMUsageLog에 기록된다
- API 오류 시 재시도 로직이 동작한다
- 실제 OpenAI API 없이 Mock으로 테스트 가능하다

---

## 7. Codex 작업 지침

- OpenAI API 키는 환경 변수로 관리하며 코드에 하드코딩하지 않는다
- API 키가 로그에 출력되지 않도록 반드시 확인한다 (보안 취약점 방지)
- temperature 기본값을 0.1로 낮게 설정하여 일관되고 사실 기반 응답을 유도한다
- Phase 10 EmbeddingProvider의 오류 처리 패턴을 참조하여 일관성을 유지한다
