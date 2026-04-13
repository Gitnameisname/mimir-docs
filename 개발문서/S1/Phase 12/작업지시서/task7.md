# Task 12-7. 타입별 검색 플러그인 (Phase 8 통합)

## 1. 작업 목적

Phase 8의 SearchService가 DocumentType별 search_config를 플러그인 레지스트리를 통해 읽도록 통합한다.

이 작업의 목표는 다음과 같다.

- SearchPlugin 인터페이스 상세 구현
- 내장 타입별 search_config 최적화
- Phase 8 SearchService를 DocumentTypeRegistry 경유로 마이그레이션
- 타입별 검색 가중치 및 부스트 설정

---

## 2. 작업 범위

### 포함 범위

- SearchPlugin 인터페이스 구현
- 내장 타입별 SearchPlugin 구현
- Phase 8 SearchService 통합 수정
- 검색 가중치 설정 (제목/본문/metadata)

### 제외 범위

- FTS/벡터 검색 로직 자체 (Phase 8/10에서 완성됨)
- 검색 UI (Phase 8에서 완성됨)

---

## 3. 선행 조건

- Task 12-2 (DocumentTypePlugin 인터페이스) 완료
- Phase 8 SearchService 및 search_config 패턴 코드 이해

---

## 4. 주요 구현 대상

### 4-1. SearchPlugin 인터페이스

```python
class SearchPlugin:
    def get_config(self) -> dict:
        """Phase 8 search_config 형식 반환"""
        return {}

    def get_boost_config(self) -> dict:
        """검색 필드별 가중치"""
        return {
            "title": 2.0,
            "content": 1.0,
            "metadata": 0.5,
        }

    def get_searchable_node_types(self) -> list[str]:
        """검색 대상 노드 타입. 빈 리스트 = 전체"""
        return []

    def get_snippet_config(self) -> dict:
        """스니펫 생성 설정"""
        return {"max_length": 200, "highlight": True}
```

---

### 4-2. 내장 타입별 SearchPlugin

#### POLICY SearchPlugin

```python
class PolicySearchPlugin(SearchPlugin):
    def get_boost_config(self):
        return {
            "title": 3.0,           # 정책 제목 가중 부스트
            "content": 1.0,
            "metadata.policy_number": 5.0,  # 정책 번호로 검색 시 최우선
        }

    def get_searchable_node_types(self):
        return ["heading1", "heading2", "heading3", "article", "paragraph"]

    def get_snippet_config(self):
        return {"max_length": 300, "highlight": True}
```

#### FAQ SearchPlugin

```python
class FAQSearchPlugin(SearchPlugin):
    def get_boost_config(self):
        return {
            "title": 2.0,
            "content": 1.5,         # FAQ는 본문 가중치 높임
        }

    def get_searchable_node_types(self):
        return ["faq_question", "faq_answer"]

    def get_snippet_config(self):
        return {"max_length": 150, "highlight": True}  # FAQ는 짧게
```

---

### 4-3. Phase 8 SearchService 마이그레이션

기존:

```python
# Phase 8 (기존)
def build_search_query(text, document_type):
    config = DocumentTypeRepository.get_search_config(document_type)
    boost = config.get("boost", {})
    # ...
```

마이그레이션 후:

```python
# Phase 12 (플러그인 경유)
def build_search_query(text, document_type):
    plugin = DocumentTypeRegistry.instance().get(document_type)
    boost = plugin.search_plugin().get_boost_config()
    searchable_types = plugin.search_plugin().get_searchable_node_types()
    snippet_config = plugin.search_plugin().get_snippet_config()
    # ...
```

---

### 4-4. 워크플로 플러그인 (WorkflowPlugin)

Phase 5 워크플로와 연계하여 타입별 워크플로 단계를 커스터마이징:

```python
class WorkflowPlugin:
    def get_allowed_transitions(self) -> dict:
        """상태 전이 제약. 빈 딕셔너리 = Phase 5 기본 워크플로 그대로"""
        return {}

    def requires_approval(self) -> bool:
        """승인 단계 필수 여부"""
        return True

    def get_review_roles(self) -> list[str]:
        """검토 가능한 역할 목록. 빈 리스트 = 모든 역할"""
        return []
```

#### POLICY WorkflowPlugin 예시

```python
class PolicyWorkflowPlugin(WorkflowPlugin):
    def requires_approval(self):
        return True  # 정책 문서는 반드시 승인 필요

    def get_review_roles(self):
        return ["POLICY_REVIEWER", "ADMIN"]  # 정책 검토자만 검토 가능
```

---

### 4-5. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| POLICY SearchPlugin | policy_number 부스트 5.0 |
| FAQ 검색 노드 타입 | faq_question, faq_answer |
| 기본 SearchPlugin | 빈 config 반환 |
| WorkflowPlugin POLICY | requires_approval=true |

---

## 5. 산출물

1. SearchPlugin 인터페이스 구현
2. WorkflowPlugin 인터페이스 구현
3. 내장 타입별 SearchPlugin / WorkflowPlugin 구현
4. Phase 8 SearchService 마이그레이션 코드
5. 단위 테스트

---

## 6. 완료 기준

- SearchPlugin이 타입별 검색 가중치와 설정을 반환한다
- SearchService가 DocumentTypeRegistry 경유로 검색 설정을 로드한다
- POLICY 타입에서 policy_number 검색이 우선 순위를 갖는다
- 단위 테스트가 통과한다

---

## 7. Codex 작업 지침

- 검색 가중치(boost) 값은 FTS 쿼리의 `ts_rank`와 직접 연동되므로, Phase 8 FTS 구현 코드와 boost 적용 방식을 먼저 확인한다
- WorkflowPlugin은 Phase 5 기본 워크플로를 완전히 교체하지 않고 제약을 추가하는 방식으로 구현한다
- metadata 필드 부스트(예: metadata.policy_number)는 Phase 8 FTS에서 지원되지 않을 수 있으므로, 지원 여부를 확인 후 구현한다
