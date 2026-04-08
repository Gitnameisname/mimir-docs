# Task 12-8. 내장 DocumentType 구현 (POLICY, MANUAL, REPORT, FAQ)

## 1. 작업 목적

Task 12-3~12-7에서 구현한 서브플러그인들을 조합하여 4개 내장 DocumentType 플러그인을 완성하고 레지스트리에 등록한다.

이 작업의 목표는 다음과 같다.

- POLICY, MANUAL, REPORT, FAQ 플러그인 클래스 통합 구현
- 시스템 부팅 시 자동 등록 메커니즘 구현
- 각 타입의 전체 동작 통합 테스트
- 기존 타입별 분기 코드 제거 (하드코딩 제거 검증)

---

## 2. 작업 범위

### 포함 범위

- 4개 내장 DocumentType 플러그인 클래스 구현
- 레지스트리 자동 등록 코드
- 하드코딩된 타입 분기 코드 제거 및 플러그인 경유 교체
- 통합 테스트

### 제외 범위

- Admin UI (Task 12-9에서 다룸)
- 새 타입 추가 (플러그인 구조 완성 후 독립 진행)

---

## 3. 선행 조건

- Task 12-3 (MetadataSchemaPlugin) 완료
- Task 12-4 (EditorPlugin, RendererPlugin) 완료
- Task 12-5 (ChunkingPlugin) 완료
- Task 12-6 (RAGPlugin) 완료
- Task 12-7 (SearchPlugin, WorkflowPlugin) 완료

---

## 4. 주요 구현 대상

### 4-1. POLICYPlugin 클래스

```python
class POLICYPlugin(DocumentTypePlugin):
    def get_type_name(self) -> str:
        return "POLICY"

    def get_display_name(self) -> str:
        return "정책/규정"

    def get_description(self) -> str:
        return "조직 정책, 규정, 지침 문서"

    def metadata_schema_plugin(self):
        return PolicyMetadataSchemaPlugin()

    def editor_plugin(self):
        return PolicyEditorPlugin()

    def renderer_plugin(self):
        return PolicyRendererPlugin()

    def chunking_plugin(self):
        return PolicyChunkingPlugin()

    def rag_plugin(self):
        return PolicyRAGPlugin()

    def search_plugin(self):
        return PolicySearchPlugin()

    def workflow_plugin(self):
        return PolicyWorkflowPlugin()
```

나머지 3개 타입 (MANUALPlugin, REPORTPlugin, FAQPlugin)도 동일 패턴으로 구현.

---

### 4-2. 자동 등록 메커니즘

애플리케이션 부팅 시 내장 플러그인 자동 등록:

```python
# app/plugins/builtin.py

BUILTIN_PLUGINS = [
    POLICYPlugin(),
    MANUALPlugin(),
    REPORTPlugin(),
    FAQPlugin(),
]

def register_builtin_plugins():
    registry = DocumentTypeRegistry.instance()
    for plugin in BUILTIN_PLUGINS:
        registry.register(plugin)

# app/main.py (FastAPI 앱 시작 시)
@app.on_event("startup")
async def startup():
    register_builtin_plugins()
```

---

### 4-3. 하드코딩 제거 체크리스트

이전 Phase에서 타입별 분기 코드를 모두 플러그인 경유로 교체:

| 위치 | 기존 코드 | 교체 후 |
|------|---------|--------|
| ChunkingService.load_config() | if doc_type == "POLICY": ... | plugin.chunking_plugin().get_config() |
| ContextBuilder.get_prompt_template() | if doc_type == "POLICY": ... | plugin.rag_plugin().get_prompt_template() |
| SearchService.build_query() | if doc_type == "FAQ": ... | plugin.search_plugin().get_boost_config() |
| WorkflowService.can_transition() | if doc_type == "POLICY": requires_approval | plugin.workflow_plugin().requires_approval() |

---

### 4-4. 타입별 동작 요약

| 항목 | POLICY | MANUAL | REPORT | FAQ |
|------|--------|--------|--------|-----|
| max_chunk_tokens | 512 | 400 | 600 | 256 |
| parent_context | O (depth=2) | O (depth=2) | O (depth=1) | X |
| requires_approval | O | O | O | X |
| RAG 스타일 | 조항 원문 인용 | 단계별 절차 | 섹션 요약 | Q&A 형식 |
| 대표 metadata | policy_number, effective_date | version_tag, target_audience | - | category, tags |

---

### 4-5. 통합 테스트

각 타입의 전체 동작 확인:

| 시나리오 | POLICY | FAQ |
|---------|--------|-----|
| chunking_config 로드 | max_tokens=512 | max_tokens=256 |
| RAG 프롬프트 | 조항 인용 지시 포함 | Q&A 형식 지시 포함 |
| 검색 가중치 | policy_number=5.0 | content=1.5 |
| 승인 필요 여부 | true | false |
| metadata 검증 | policy_number 형식 | category 문자열 |

---

## 5. 산출물

1. POLICYPlugin, MANUALPlugin, REPORTPlugin, FAQPlugin 클래스
2. 자동 등록 코드 (register_builtin_plugins)
3. 하드코딩된 타입 분기 코드 제거 커밋
4. 통합 테스트

---

## 6. 완료 기준

- 4개 내장 타입 플러그인이 레지스트리에 자동 등록된다
- 각 타입의 chunking, RAG, 검색, 워크플로 설정이 플러그인에서 반환된다
- 기존 하드코딩된 타입 분기 코드가 제거되었다
- 통합 테스트가 통과한다

---

## 7. Codex 작업 지침

- 내장 플러그인 4개는 각각 별도 파일로 구성하여 독립적으로 수정 가능하게 한다
- 하드코딩 제거 시 기존 동작과 동일하게 동작하는지 반드시 테스트 후 커밋한다
- CLAUDE.md 원칙 "문서 타입은 하드코딩 금지"를 이 Task 완료 시점에서 코드베이스 전체에서 검증한다
