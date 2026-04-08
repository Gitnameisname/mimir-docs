# Task 12-2. DocumentTypePlugin 인터페이스 구현 및 레지스트리

## 1. 작업 목적

Task 12-1에서 설계한 플러그인 인터페이스와 레지스트리를 실제 코드로 구현한다.

이 작업의 목표는 다음과 같다.

- DocumentTypePlugin 루트 인터페이스 구현
- 모든 서브플러그인 기본 인터페이스 구현
- DefaultDocumentTypePlugin 구현 (폴백)
- DocumentTypeRegistry 구현
- 단위 테스트 작성

---

## 2. 작업 범위

### 포함 범위

- DocumentTypePlugin 인터페이스 및 기본 구현
- 7개 서브플러그인 인터페이스 및 기본 구현
- DocumentTypeRegistry 구현 (캐싱 포함)
- 플러그인 등록 검증 로직
- 단위 테스트

### 제외 범위

- 각 서브플러그인 상세 구현 (Task 12-3~12-7에서 다룸)
- 내장 타입 구현 (Task 12-8에서 다룸)

---

## 3. 선행 조건

- Task 12-1 (아키텍처 설계) 완료

---

## 4. 주요 구현 대상

### 4-1. DocumentTypePlugin 루트 인터페이스

```python
from abc import ABC, abstractmethod

class DocumentTypePlugin(ABC):
    @abstractmethod
    def get_type_name(self) -> str:
        """영문 대문자, 밑줄만 허용 (예: POLICY)"""
        pass

    @abstractmethod
    def get_display_name(self) -> str:
        """UI 표시 이름 (예: 정책/규정)"""
        pass

    def get_description(self) -> str:
        return ""

    # 서브플러그인 접근자 (기본 구현 제공)
    def metadata_schema_plugin(self) -> "MetadataSchemaPlugin":
        return DefaultMetadataSchemaPlugin()

    def editor_plugin(self) -> "EditorPlugin":
        return DefaultEditorPlugin()

    def renderer_plugin(self) -> "RendererPlugin":
        return DefaultRendererPlugin()

    def chunking_plugin(self) -> "ChunkingPlugin":
        return DefaultChunkingPlugin()

    def rag_plugin(self) -> "RAGPlugin":
        return DefaultRAGPlugin()

    def search_plugin(self) -> "SearchPlugin":
        return DefaultSearchPlugin()

    def workflow_plugin(self) -> "WorkflowPlugin":
        return DefaultWorkflowPlugin()
```

---

### 4-2. 서브플러그인 기본 인터페이스

각 서브플러그인은 최소 1개의 추상 메서드 없이 기본 구현만 제공:

```python
class ChunkingPlugin:
    def get_config(self) -> dict:
        return {
            "strategy": "node_based",
            "max_chunk_tokens": 512,
            "min_chunk_tokens": 50,
            "overlap_tokens": 50,
            "include_parent_context": True,
            "parent_context_depth": 2,
            "index_version_policy": "published_only",
            "exclude_node_types": [],
            "merge_strategy": "merge_siblings",
        }

class RAGPlugin:
    def get_prompt_template(self) -> "PromptTemplate":
        return DefaultPromptTemplate()

    def get_context_config(self) -> dict:
        return {"max_context_tokens": 6000, "top_n": 5}

class SearchPlugin:
    def get_config(self) -> dict:
        return {}  # Phase 8 기본값 사용

class WorkflowPlugin:
    def get_allowed_transitions(self) -> dict:
        return {}  # Phase 5 기본 워크플로 사용

class MetadataSchemaPlugin:
    def get_schema(self) -> dict:
        return {}  # metadata 제약 없음

class EditorPlugin:
    def get_allowed_node_types(self) -> list[str]:
        return []  # 모든 노드 타입 허용

class RendererPlugin:
    def get_render_config(self) -> dict:
        return {}  # 기본 렌더링
```

---

### 4-3. DocumentTypeRegistry

```python
class DocumentTypeRegistry:
    _instance: "DocumentTypeRegistry | None" = None
    _plugins: dict[str, DocumentTypePlugin] = {}

    @classmethod
    def instance(cls) -> "DocumentTypeRegistry":
        if cls._instance is None:
            cls._instance = cls()
        return cls._instance

    def register(self, plugin: DocumentTypePlugin) -> None:
        type_name = plugin.get_type_name()
        # 검증
        if not re.match(r'^[A-Z][A-Z0-9_]*$', type_name):
            raise ValueError(f"Invalid type_name: {type_name}")
        if type_name in self._plugins:
            raise ValueError(f"Plugin already registered: {type_name}")
        self._plugins[type_name] = plugin

    def get(self, document_type: str) -> DocumentTypePlugin:
        if document_type in self._plugins:
            return self._plugins[document_type]
        # DB 기반 설정 조회 시도
        db_config = self._load_from_db(document_type)
        if db_config:
            return ConfigurableDocumentTypePlugin(document_type, db_config)
        # 폴백
        return DefaultDocumentTypePlugin(document_type)

    def list_all(self) -> list[DocumentTypePlugin]:
        return list(self._plugins.values())

    @classmethod
    def reset(cls) -> None:
        """테스트용 레지스트리 초기화"""
        cls._instance = None
```

---

### 4-4. ConfigurableDocumentTypePlugin

DB에서 로드한 설정으로 동작하는 플러그인:

```python
class ConfigurableDocumentTypePlugin(DocumentTypePlugin):
    def __init__(self, type_name: str, config: dict):
        self._type_name = type_name
        self._config = config

    def get_type_name(self) -> str:
        return self._type_name

    def get_display_name(self) -> str:
        return self._config.get("display_name", self._type_name)

    def chunking_plugin(self) -> ChunkingPlugin:
        config = self._config.get("chunking_config", {})
        return ConfigurableChunkingPlugin(config)

    # ... 나머지 서브플러그인도 동일 패턴
```

---

### 4-5. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| 플러그인 등록 및 조회 | 등록한 플러그인 반환 |
| 중복 등록 | ValueError 발생 |
| 잘못된 type_name | ValueError 발생 |
| 미등록 타입 조회 | DefaultDocumentTypePlugin 반환 |
| 레지스트리 reset() | 빈 상태로 초기화 |

---

## 5. 산출물

1. DocumentTypePlugin 루트 인터페이스 구현
2. 7개 서브플러그인 기본 인터페이스 구현
3. DocumentTypeRegistry 구현
4. ConfigurableDocumentTypePlugin 구현
5. 단위 테스트

---

## 6. 완료 기준

- DocumentTypePlugin 인터페이스가 구현되어 있다
- DocumentTypeRegistry에 플러그인을 등록하고 조회할 수 있다
- 미등록 타입 조회 시 DefaultDocumentTypePlugin이 반환된다
- Registry.reset()으로 테스트 격리가 가능하다
- 단위 테스트가 통과한다

---

## 7. Codex 작업 지침

- 서브플러그인 기본 구현(Default*)은 기존 Phase 8/10/11의 기본값을 그대로 반환하여 하위 호환성을 유지한다
- DocumentTypeRegistry는 싱글턴이지만 테스트에서 reset()으로 격리할 수 있도록 한다
- type_name 정규식 검증은 느슨하게 시작하여 필요 시 강화한다 (현재: 영문 대문자 + 숫자 + 밑줄)
