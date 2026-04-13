# Task 10-5. 청킹 파이프라인 구현

## 1. 작업 목적

Task 10-2에서 설계한 청킹 전략을 실제 구현한다.

이 작업의 목표는 다음과 같다.

- ChunkingService 구현 (node_based 청킹)
- 부모 컨텍스트 주입 로직 구현
- 청크 merge/split 로직 구현
- DocumentType chunking_config 로드 구현
- 단위 테스트 작성

---

## 2. 작업 범위

### 포함 범위

- ChunkingService 클래스 구현
- node_based 청킹 알고리즘 구현
- 부모 컨텍스트 주입 구현
- 청크 크기 조정 (merge/split)
- DocumentType chunking_config 로드 및 적용
- 토큰 수 추정 유틸리티
- 단위 테스트

### 제외 범위

- 임베딩 연동 (Task 10-6에서 다룸)
- DB 저장 (Task 10-6에서 다룸)
- 실제 DocumentType 설정 변경 (해당 Phase에서 다룸)

---

## 3. 선행 조건

- Task 10-2 (청킹 전략 설계) 완료
- Task 10-3 (pgvector 스키마 설계) 완료 — Chunk 메타데이터 스키마 참조
- Phase 1: Node 트리 구조 및 Node 모델 코드 이해

---

## 4. 주요 구현 대상

### 4-1. ChunkingService 인터페이스

```
ChunkingService
  .chunk_document(document_id, version_id) → Chunk[]
  .chunk_nodes(nodes: Node[], config: ChunkingConfig) → Chunk[]
  ._build_chunks_from_node(node, config, context_stack) → Chunk[]
  ._merge_small_chunks(chunks, config) → Chunk[]
  ._split_large_chunk(chunk, config) → Chunk[]
  ._inject_parent_context(text, context_stack, config) → str
  ._estimate_tokens(text) → int
```

#### Chunk 데이터 클래스

```
Chunk
  document_id: str
  version_id: str
  node_id: str | None
  chunk_index: int
  chunk_total: int        (후처리 시 설정)
  source_text: str
  context_text: str
  token_count: int
  node_path: list[str]
  node_type: str | None
  document_type: str
  document_status: str
  accessible_roles: list[str]
  accessible_org_ids: list[str]
  is_public: bool
```

---

### 4-2. node_based 청킹 알고리즘 구현

```python
def _build_chunks_from_node(node, config, context_stack=[]):
    text = extract_text(node)
    token_count = _estimate_tokens(text)

    if node.node_type in config.exclude_node_types:
        return []

    if token_count > config.max_chunk_tokens:
        if node.children:
            # 하위 노드 재귀 처리
            new_context = context_stack + [node.title or ""]
            chunks = []
            for child in node.children:
                chunks.extend(_build_chunks_from_node(child, config, new_context))
            return chunks
        else:
            # 고정 크기 분할 fallback
            return _split_by_fixed_size(node, config, context_stack)

    context_text = _inject_parent_context(text, context_stack, config)
    return [Chunk(source_text=text, context_text=context_text, ...)]
```

---

### 4-3. 부모 컨텍스트 주입 구현

```python
def _inject_parent_context(text, context_stack, config):
    if not config.include_parent_context or not context_stack:
        return text

    depth = config.parent_context_depth
    relevant = context_stack[-depth:] if depth > 0 else []

    if not relevant:
        return text

    # 최상단 = 문서 제목, 나머지 = 섹션 경로
    doc_title = relevant[0] if len(relevant) >= 1 else ""
    section_path = " > ".join(relevant[1:]) if len(relevant) > 1 else ""

    prefix = ""
    if doc_title:
        prefix += f"[문서: {doc_title}]\n"
    if section_path:
        prefix += f"[섹션: {section_path}]\n"

    return prefix + text
```

---

### 4-4. 토큰 수 추정

OpenAI tiktoken 라이브러리 사용 (cl100k_base 인코딩):

```python
import tiktoken

_encoder = tiktoken.get_encoding("cl100k_base")

def _estimate_tokens(text: str) -> int:
    return len(_encoder.encode(text))
```

tiktoken이 없는 환경에서는 단어 수 기반 근사치 사용 (4자 = 1토큰 근사):

```python
def _estimate_tokens_fallback(text: str) -> int:
    return max(1, len(text) // 4)
```

---

### 4-5. ChunkingConfig 로드

DocumentType 설정에서 chunking_config를 로드하여 ChunkingConfig 객체로 변환:

```python
@dataclass
class ChunkingConfig:
    strategy: str = "node_based"
    max_chunk_tokens: int = 512
    min_chunk_tokens: int = 50
    overlap_tokens: int = 50
    include_parent_context: bool = True
    parent_context_depth: int = 2
    index_version_policy: str = "published_only"
    exclude_node_types: list[str] = field(default_factory=list)
    merge_strategy: str = "merge_siblings"

def load_config(document_type: str) -> ChunkingConfig:
    doc_type_config = DocumentTypeRepository.get_config(document_type)
    chunking_raw = doc_type_config.get("chunking_config", {})
    return ChunkingConfig(**chunking_raw)
```

---

### 4-6. 권한 메타데이터 스냅샷

청크 생성 시 문서의 ACL을 스냅샷으로 포함:

```python
def _snapshot_permissions(document_id) -> dict:
    acl = PermissionService.get_document_acl(document_id)
    return {
        "accessible_roles": acl.roles,
        "accessible_org_ids": acl.org_ids,
        "is_public": acl.is_public,
    }
```

---

### 4-7. 단위 테스트

Node 트리와 DocumentType 설정을 Mock하여 테스트:

| 시나리오 | 기대 결과 |
|---------|----------|
| 단일 노드 (토큰 적당) | 1개 Chunk 반환 |
| 단일 노드 (토큰 초과) | fixed_size fallback으로 분할 |
| 계층 노드 트리 | 리프 단위 Chunk[] 반환 |
| min_chunk_tokens 미달 노드 | 형제와 병합 |
| exclude_node_types 포함 노드 | 해당 노드 제외 |
| include_parent_context=true | context_text에 부모 경로 포함 |
| 권한 메타데이터 | accessible_roles 포함 확인 |

---

## 5. 산출물

1. ChunkingService 구현 코드
2. Chunk 데이터 클래스 정의
3. ChunkingConfig 데이터 클래스 및 로더
4. 토큰 추정 유틸리티
5. 단위 테스트

---

## 6. 완료 기준

- ChunkingService가 Node 트리를 Chunk[] 로 변환한다
- 부모 컨텍스트가 context_text에 올바르게 주입된다
- max_chunk_tokens 초과 시 split, min_chunk_tokens 미달 시 merge가 동작한다
- DocumentType chunking_config에서 설정을 읽어 적용한다
- 권한 메타데이터가 각 Chunk에 포함된다
- 단위 테스트가 통과한다

---

## 7. Codex 작업 지침

- chunking_config는 하드코딩 금지 — 반드시 DocumentType 설정에서 로드한다
- node_type 분기 로직을 서비스 레이어 내부에 두되 exclude_node_types 설정으로 동작한다
- tiktoken 의존성이 없는 환경을 위해 fallback 추정 로직을 함께 제공한다
- 권한 메타데이터 스냅샷은 청크 생성 시점의 ACL을 반영해야 하며, 이후 ACL 변경은 별도 업데이트 경로(Task 10-8)로 처리한다
