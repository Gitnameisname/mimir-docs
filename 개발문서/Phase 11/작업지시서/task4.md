# Task 11-4. Reranker 구현

## 1. 작업 목적

Retriever가 반환한 Top-K 청크를 Cross-encoder 기반으로 재랭킹하여 LLM에 전달할 컨텍스트 품질을 높인다.

이 작업의 목표는 다음과 같다.

- Reranker 인터페이스 설계
- CrossEncoderReranker 구현
- 점수 임계값 기반 청크 필터링
- 설정으로 Reranker ON/OFF 및 모델 교체 지원

---

## 2. 작업 범위

### 포함 범위

- Reranker 인터페이스 설계
- CrossEncoderReranker 구현
- 점수 임계값 필터링
- 단위 테스트

### 제외 범위

- LLM 연동 (Task 11-5에서 다룸)
- ContextBuilder (Task 11-6에서 다룸)

---

## 3. 선행 조건

- Task 11-3 (Retriever) 완료 — RetrievedChunk 구조 확정

---

## 4. 주요 구현 대상

### 4-1. Reranker 인터페이스

```
Reranker (인터페이스)
  .rerank(query: str, chunks: RetrievedChunk[], top_n: int) → RetrievedChunk[]

구현체:
  CrossEncoderReranker   (기본)
  PassthroughReranker    (Reranker 비활성화 시 원본 순서 유지)
```

---

### 4-2. CrossEncoderReranker 구현

```python
from sentence_transformers import CrossEncoder

class CrossEncoderReranker(Reranker):
    model_name = "cross-encoder/ms-marco-MiniLM-L-6-v2"

    def __init__(self):
        self._model = CrossEncoder(self.model_name)

    def rerank(self, query: str, chunks: list[RetrievedChunk], top_n: int = 5) -> list[RetrievedChunk]:
        if not chunks:
            return []

        pairs = [(query, chunk.context_text) for chunk in chunks]
        scores = self._model.predict(pairs)

        for chunk, score in zip(chunks, scores):
            chunk.rerank_score = float(score)

        sorted_chunks = sorted(chunks, key=lambda c: c.rerank_score, reverse=True)

        # 임계값 필터링
        threshold = settings.RERANKER_SCORE_THRESHOLD  # 기본 0.0
        filtered = [c for c in sorted_chunks if c.rerank_score >= threshold]

        return filtered[:top_n]
```

---

### 4-3. RetrievedChunk rerank_score 필드 추가

```
RetrievedChunk (확장)
  ...기존 필드...
  rerank_score: float | None = None
```

---

### 4-4. PassthroughReranker

Reranker가 비활성화된 경우 원본 순서 유지:

```python
class PassthroughReranker(Reranker):
    def rerank(self, query: str, chunks: list[RetrievedChunk], top_n: int = 5) -> list[RetrievedChunk]:
        return chunks[:top_n]
```

---

### 4-5. Reranker 팩토리

설정 기반 Reranker 선택:

```python
def get_reranker() -> Reranker:
    if settings.RERANKER_ENABLED:
        return CrossEncoderReranker()
    return PassthroughReranker()
```

---

### 4-6. 설정

```
RERANKER_ENABLED = true          (기본 활성화)
RERANKER_MODEL = "cross-encoder/ms-marco-MiniLM-L-6-v2"
RERANKER_TOP_N = 5               (재랭킹 후 선택 청크 수)
RERANKER_SCORE_THRESHOLD = 0.0   (기본 임계값 없음)
```

---

### 4-7. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| 정상 재랭킹 | top_n개 청크 반환, rerank_score 내림차순 |
| 빈 청크 입력 | 빈 리스트 반환 |
| RERANKER_ENABLED=false | PassthroughReranker 사용, 원본 순서 유지 |
| 임계값 0.5 설정 | 0.5 미만 청크 제거 |

---

## 5. 산출물

1. Reranker 인터페이스 및 구현 코드
2. CrossEncoderReranker 구현
3. PassthroughReranker 구현
4. Reranker 팩토리
5. 단위 테스트

---

## 6. 완료 기준

- CrossEncoderReranker가 쿼리-청크 쌍을 재랭킹하고 top_n개를 반환한다
- RERANKER_ENABLED=false 시 원본 순서를 유지한다
- rerank_score가 RetrievedChunk에 기록된다
- 단위 테스트가 통과한다

---

## 7. Codex 작업 지침

- sentence-transformers 모델은 첫 호출 시 로컬 캐시에 다운로드된다 — 운영 환경에서 사전 다운로드 방안을 검토한다
- CrossEncoder 모델 로딩은 싱글턴으로 처리하여 요청마다 재로딩하지 않는다
- 한국어 성능이 필요하면 한/영 모델로 교체 가능하도록 RERANKER_MODEL 설정을 사용한다
