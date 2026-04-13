# Task 11-7. Citation(근거) 연결 구현

## 1. 작업 목적

LLM 응답에 포함된 출처 마킹(`[1]`, `[2]`)을 파싱하여 원본 문서 노드와 연결한다.

이 작업의 목표는 다음과 같다.

- LLM 응답 내 Citation 마킹 파싱
- 청크 → node_id → 문서 링크 매핑
- Citation 구조 정의 및 API 응답에 포함
- 스트리밍 응답에서 Citation 처리

---

## 2. 작업 범위

### 포함 범위

- CitationLinker 구현
- Citation 데이터 구조 정의
- 문서 노드 링크 생성 (node_id → URL)
- 스트리밍 완료 후 Citation 후처리

### 제외 범위

- UI에서 Citation 표시 (Task 11-10에서 다룸)
- RAG Query API (Task 11-8에서 다룸)

---

## 3. 선행 조건

- Task 11-6 (ContextBuilder) 완료 — chunk_mapping 확정
- Task 11-5 (LLMProvider) 완료 — LLMResponse 구조 확정

---

## 4. 주요 구현 대상

### 4-1. Citation 구조

```
Citation
  number: int               ([1], [2] 번호)
  chunk_id: str
  document_id: str
  document_title: str
  node_id: str | None
  node_path: list[str]      (["정보보안 정책", "2. 정책 본문", "2.3 접근통제"])
  source_text: str          (근거 청크 원문)
  document_url: str         (/documents/{document_id}?node={node_id})
  version_id: str
```

---

### 4-2. CitationLinker 구현

```
CitationLinker
  .link(response_text: str, chunk_mapping: dict[int, RetrievedChunk]) → CitationResult

CitationResult
  answer: str               (citation 마킹이 포함된 응답 원문)
  citations: Citation[]     (번호별 근거 정보)
  cited_numbers: list[int]  (실제 응답에 등장한 번호 목록)
```

#### Citation 파싱 로직

```python
import re

def link(self, response_text: str, chunk_mapping: dict) -> CitationResult:
    # "[1]", "[2]" 형식 파싱
    pattern = r'\[(\d+)\]'
    found_numbers = sorted(set(int(m) for m in re.findall(pattern, response_text)))

    citations = []
    for num in found_numbers:
        if num in chunk_mapping:
            chunk = chunk_mapping[num]
            citations.append(Citation(
                number=num,
                chunk_id=chunk.chunk_id,
                document_id=chunk.document_id,
                document_title=chunk.document_title,
                node_id=chunk.node_id,
                node_path=chunk.node_path,
                source_text=chunk.source_text,
                document_url=self._build_url(chunk),
                version_id=chunk.version_id,
            ))

    return CitationResult(
        answer=response_text,
        citations=citations,
        cited_numbers=found_numbers,
    )
```

---

### 4-3. 문서 링크 생성

```python
def _build_url(self, chunk: RetrievedChunk) -> str:
    base = f"/documents/{chunk.document_id}"
    if chunk.node_id:
        return f"{base}?node={chunk.node_id}"
    return base
```

---

### 4-4. 스트리밍 응답에서 Citation 처리

스트리밍 중에는 텍스트 청크를 그대로 전송하고, 완료 후 Citation을 별도 이벤트로 전송:

```
스트리밍 흐름:
  1. LLM 텍스트 청크 → SSE "chunk" 이벤트로 전송 (실시간)
  2. 스트리밍 완료 → 전체 응답 텍스트 수집
  3. CitationLinker.link() 실행
  4. SSE "citations" 이벤트로 Citation[] 전송
  5. SSE "done" 이벤트로 conversation_id, message_id 전송
```

---

### 4-5. 미참조 청크 처리

LLM이 일부 청크를 응답에 참조하지 않은 경우에도 컨텍스트 청크 목록은 API 응답에 포함:

```
RAGResponse
  answer: str
  citations: Citation[]         (실제 참조된 출처)
  context_chunks: RetrievedChunk[]  (검색된 전체 청크 - 참조 여부 무관)
  conversation_id: str
  message_id: str
```

---

### 4-6. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| "[1], [2] 참조" 응답 | citations 2개 반환 |
| 번호 미참조 응답 | citations 빈 리스트 |
| 존재하지 않는 번호 "[5]" | 해당 번호 무시 |
| node_id 없는 청크 | document_url = /documents/{id} |

---

## 5. 산출물

1. CitationLinker 구현 코드
2. Citation / CitationResult 데이터 클래스
3. 문서 링크 생성 유틸리티
4. 단위 테스트

---

## 6. 완료 기준

- CitationLinker가 응답 텍스트에서 `[1]`, `[2]` 패턴을 파싱한다
- 각 번호가 원본 청크 및 문서 노드 링크와 매핑된다
- 스트리밍 완료 후 Citation이 별도 이벤트로 전송된다
- 단위 테스트가 통과한다

---

## 7. Codex 작업 지침

- Citation 번호 파싱은 `[1]` 형식만 처리하며, `(1)`, `1.` 등 다른 형식은 무시한다
- LLM이 응답에 없는 번호를 생성하는 경우(hallucination) chunk_mapping에 없는 번호는 조용히 무시한다
- document_url은 상대 경로로 생성하여 배포 환경에 독립적으로 유지한다
