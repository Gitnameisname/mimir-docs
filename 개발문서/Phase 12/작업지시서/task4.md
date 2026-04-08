# Task 12-4. 타입별 editor/renderer 플러그인

## 1. 작업 목적

DocumentType별로 편집기 동작 규칙과 렌더링 방식을 커스터마이징하는 플러그인을 구현한다.

이 작업의 목표는 다음과 같다.

- EditorPlugin 인터페이스 구현 (허용 노드 타입, 기본 구조 템플릿)
- RendererPlugin 인터페이스 구현 (노드 렌더링 설정)
- 내장 타입별 Editor/Renderer 설정 정의
- Phase 6 편집기/뷰어와 통합

---

## 2. 작업 범위

### 포함 범위

- EditorPlugin 인터페이스 및 내장 타입 구현
- RendererPlugin 인터페이스 및 내장 타입 구현
- 편집기 노드 팔레트 설정 (허용 노드 타입 목록)
- 문서 생성 시 기본 구조 템플릿

### 제외 범위

- 편집기 UI 자체 수정 (Phase 6에서 완성됨)
- 렌더링 엔진 수정 (Phase 6에서 완성됨)

---

## 3. 선행 조건

- Task 12-2 (DocumentTypePlugin 인터페이스) 완료
- Phase 6 편집기 구조 이해 — 노드 타입 팔레트 구조 파악

---

## 4. 주요 구현 대상

### 4-1. EditorPlugin 인터페이스

```python
class EditorPlugin:
    def get_allowed_node_types(self) -> list[str]:
        """편집기에서 추가 가능한 노드 타입 목록. 빈 리스트 = 모두 허용"""
        return []

    def get_default_structure(self) -> list[dict]:
        """신규 문서 생성 시 기본 노드 구조 템플릿"""
        return []

    def get_editor_config(self) -> dict:
        """편집기 추가 설정 (툴바, 단축키 등)"""
        return {}
```

#### POLICY EditorPlugin 예시

```python
class PolicyEditorPlugin(EditorPlugin):
    def get_allowed_node_types(self):
        return ["heading1", "heading2", "heading3", "paragraph",
                "article",    # 정책 조항
                "table", "list", "note"]

    def get_default_structure(self):
        return [
            {"type": "heading1", "content": "1. 목적"},
            {"type": "paragraph", "content": ""},
            {"type": "heading1", "content": "2. 적용 범위"},
            {"type": "paragraph", "content": ""},
            {"type": "heading1", "content": "3. 정책 본문"},
            {"type": "article",  "content": "제1조 (기본 원칙)"},
        ]
```

#### FAQ EditorPlugin 예시

```python
class FAQEditorPlugin(EditorPlugin):
    def get_allowed_node_types(self):
        return ["faq_question", "faq_answer", "paragraph"]

    def get_default_structure(self):
        return [
            {"type": "faq_question", "content": "Q: 질문을 입력하세요"},
            {"type": "faq_answer",   "content": "A: 답변을 입력하세요"},
        ]
```

---

### 4-2. RendererPlugin 인터페이스

```python
class RendererPlugin:
    def get_render_config(self) -> dict:
        """노드 타입별 렌더링 규칙"""
        return {}

    def get_node_css_class(self, node_type: str) -> str | None:
        """노드 타입에 추가할 CSS 클래스"""
        return None

    def get_toc_config(self) -> dict:
        """목차(TOC) 구성 설정"""
        return {"enabled": True, "depth": 3}
```

#### POLICY RendererPlugin 예시

```python
class PolicyRendererPlugin(RendererPlugin):
    def get_render_config(self):
        return {
            "article": {
                "prefix": "제",
                "suffix": "조",
                "numbered": True,
            }
        }

    def get_toc_config(self):
        return {"enabled": True, "depth": 2, "label": "목차"}
```

---

### 4-3. Phase 6 편집기 통합

편집기 노드 팔레트가 EditorPlugin 설정을 읽어 허용 노드만 표시:

```typescript
// 편집기 초기화 시
const plugin = await DocumentTypeRegistry.get(documentType);
const editorConfig = plugin.editorPlugin.getEditorConfig();
const allowedNodeTypes = editorConfig.allowedNodeTypes;

// 팔레트에서 허용된 노드 타입만 표시
renderNodePalette(allowedNodeTypes);
```

---

### 4-4. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| POLICY 기본 구조 | 6개 기본 노드 반환 |
| FAQ 허용 노드 타입 | faq_question, faq_answer 포함 |
| 기본 플러그인 허용 노드 | 빈 리스트 (모두 허용) |
| 렌더 설정 조회 | 타입별 렌더 규칙 반환 |

---

## 5. 산출물

1. EditorPlugin 인터페이스 및 내장 타입 구현 (POLICY, MANUAL, REPORT, FAQ)
2. RendererPlugin 인터페이스 및 내장 타입 구현
3. Phase 6 편집기 통합 코드
4. 단위 테스트

---

## 6. 완료 기준

- EditorPlugin이 타입별 허용 노드 및 기본 구조를 반환한다
- RendererPlugin이 타입별 렌더링 설정을 반환한다
- Phase 6 편집기가 EditorPlugin 설정을 읽어 노드 팔레트를 구성한다
- 내장 4개 타입의 Editor/Renderer 설정이 정의되어 있다

---

## 7. Codex 작업 지침

- 허용 노드 타입은 CLAUDE.md 원칙에 따라 하드코딩 금지 — 항상 플러그인에서 읽는다
- 기본 구조 템플릿은 최소한의 뼈대만 제공하며 빈 내용으로 시작한다
- RendererPlugin은 편집기 UI가 아닌 뷰어 렌더링에만 영향을 준다
