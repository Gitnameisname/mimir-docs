# Task 10-4. 임베딩 모델 추상화 레이어 설계 및 구현

## 1. 작업 목적

임베딩 모델 교체가 가능한 추상화 레이어를 설계하고, OpenAI 기반 기본 구현체를 제공한다.

이 작업의 목표는 다음과 같다.

- EmbeddingProvider 인터페이스 설계 및 구현
- OpenAIEmbeddingProvider 구현 (배치 처리 포함)
- 토큰 사용량 추적 및 비용 모니터링
- 오류 처리 및 재시도 로직 구현

---

## 2. 작업 범위

### 포함 범위

- EmbeddingProvider 인터페이스 설계
- OpenAIEmbeddingProvider 구현
- 배치 처리 로직 구현
- 토큰 사용량 기록 구현
- API 오류 처리 및 재시도 로직
- 단위 테스트 (Mock 기반)

### 제외 범위

- 로컬 임베딩 모델 구현 (Reserved)
- 청킹 파이프라인 연결 (Task 10-5에서 다룸)

---

## 3. 선행 조건

- Task 10-1 (벡터화 파이프라인 아키텍처 설계) 완료

---

## 4. 주요 구현 대상

### 4-1. EmbeddingProvider 인터페이스

```
EmbeddingProvider (인터페이스)
  .embed(texts: str[]) → EmbeddingResult[]
  .embed_single(text: str) → EmbeddingResult
  .get_model_name() → str
  .get_dimensions() → int
  .estimate_tokens(text: str) → int
```

#### EmbeddingResult 구조

```
EmbeddingResult
  text: str           (입력 텍스트)
  embedding: float[]  (벡터, 길이 = dimensions)
  token_count: int    (실제 사용 토큰 수)
  model: str          (사용된 모델명)
```

---

### 4-2. OpenAIEmbeddingProvider 구현

#### 설정

```
model: "text-embedding-3-small"  (기본)
dimensions: 1536
batch_size: 100  (한 번 API 호출당 최대 텍스트 수)
max_retries: 3
retry_delay_seconds: 1.0
```

#### 배치 처리 로직

```
embed(texts: str[]) → EmbeddingResult[]:
  결과 = []
  for batch in chunks(texts, batch_size=100):
    응답 = openai.embeddings.create(input=batch, model=self.model)
    for i, data in enumerate(응답.data):
      결과.append(EmbeddingResult(
        text=batch[i],
        embedding=data.embedding,
        token_count=응답.usage.total_tokens / len(batch),  // 근사치
        model=self.model
      ))
    self.record_usage(응답.usage)  // 비용 추적
  return 결과
```

---

### 4-3. 토큰 사용량 추적

임베딩 API 호출 시 사용된 토큰을 기록하여 비용 모니터링에 활용:

#### 기록 대상

```
EmbeddingUsageLog
  id
  document_id (선택)
  model: str
  prompt_tokens: int
  total_tokens: int
  estimated_cost_usd: float
  created_at
```

#### 비용 계산 기준 (2025년 기준)

| 모델 | 비용 |
|------|------|
| text-embedding-3-small | $0.02 / 1M tokens |
| text-embedding-3-large | $0.13 / 1M tokens |

---

### 4-4. 오류 처리 및 재시도 로직

| 오류 유형 | 처리 방식 |
|----------|----------|
| RateLimitError | Exponential backoff로 재시도 (최대 3회) |
| APIConnectionError | 즉시 재시도 후 실패 처리 |
| InvalidRequestError | 텍스트 길이 초과 → 텍스트 자르기 후 재시도 |
| AuthenticationError | 즉시 실패 (재시도 불가) |
| 기타 서버 오류 | 재시도 후 실패 처리 |

#### 텍스트 길이 초과 처리

```
최대 토큰 수(8191) 초과 시:
  1. 텍스트를 최대 길이로 자르기 (뒷부분 제거)
  2. 경고 로그 기록
  3. 잘린 텍스트로 임베딩 생성
```

---

### 4-5. 단위 테스트

OpenAI API를 Mock하여 의존성 없이 테스트:

| 시나리오 | 기대 결과 |
|---------|----------|
| 단건 텍스트 임베딩 | EmbeddingResult 반환 |
| 배치(100건) 임베딩 | 100개의 EmbeddingResult 반환 |
| 100건 초과 배치 | 자동으로 나눠서 API 호출 |
| RateLimitError 발생 | 재시도 후 성공 |
| 텍스트 길이 초과 | 자르기 후 성공, 경고 로그 |
| AuthenticationError | 즉시 예외 발생 |

---

## 5. 산출물

1. EmbeddingProvider 인터페이스 구현
2. OpenAIEmbeddingProvider 구현 (배치 처리, 오류 처리 포함)
3. EmbeddingUsageLog 저장 구현
4. 단위 테스트 (Mock API 기반)

---

## 6. 완료 기준

- EmbeddingProvider 인터페이스가 정의되어 있다
- OpenAIEmbeddingProvider가 배치 처리를 지원하며 동작한다
- 토큰 사용량이 기록된다
- API 오류 시 재시도 로직이 동작한다
- 단위 테스트가 통과한다
- 실제 OpenAI API 없이 Mock으로 테스트 가능하다

---

## 7. Codex 작업 지침

- OpenAI API 키는 환경 변수로 관리하며 코드에 하드코딩하지 않는다
- API 키가 로그에 출력되지 않도록 반드시 확인한다 (보안 취약점 방지)
- EmbeddingProvider 인터페이스를 통해 모델 교체가 가능한 구조를 유지한다
- 배치 크기는 설정으로 조정 가능하게 한다
- 비용 추적은 운영에서 중요하므로 누락 없이 기록되어야 한다
