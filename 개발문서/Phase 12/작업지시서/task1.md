# Task 12-1. 플러그인 아키텍처 설계

## 1. 작업 목적

Phase 12 전체 DocumentType 플러그인 시스템의 기술 방향과 구조를 확정한다.

이 작업의 목표는 다음과 같다.

- 플러그인 인터페이스 계층 구조 설계
- 플러그인 레지스트리 설계
- 플러그인 발견 및 등록 메커니즘 설계
- 기존 Phase 8/10/11 코드와의 통합 전략 설계
- 하위 호환성 유지 전략 확정

---

## 2. 작업 범위

### 포함 범위

- 플러그인 계층 구조 및 역할 분리 설계
- DocumentTypeRegistry 설계
- 플러그인 발견 방식 결정 (코드 등록 vs DB 기반)
- 기존 코드 마이그레이션 전략
- 플러그인 규격 정의

### 제외 범위

- 실제 인터페이스/레지스트리 구현 (Task 12-2에서 다룸)
- 내장 타입 구현 (Task 12-8에서 다룸)

---

## 3. 선행 조건

- Phase 10 Task 10-2 (chunking_config 패턴) 이해
- Phase 11 Task 11-6 (PromptTemplate 추상화) 이해
- Phase 7 Task 7-5 (DocumentType Admin 관리) 이해
- Phase 8 search_config 패턴 이해

---

## 4. 주요 설계 대상

### 4-1. 플러그인 계층 구조

```
DocumentTypePlugin (루트 인터페이스)
  .get_type_name() → str
  .get_display_name() → str
  .get_description() → str

  서브플러그인 접근자:
  .metadata_schema_plugin() → MetadataSchemaPlugin
  .editor_plugin() → EditorPlugin
  .renderer_plugin() → RendererPlugin
  .chunking_plugin() → ChunkingPlugin
  .rag_plugin() → RAGPlugin
  .search_plugin() → SearchPlugin
  .workflow_plugin() → WorkflowPlugin
```

각 접근자는 기본 구현체를 반환하며, 서브클래스에서 오버라이드하여 커스터마이징.

---

### 4-2. 서브플러그인 역할 분리

| 서브플러그인 | 역할 | 기존 연계 |
|------------|------|---------|
| MetadataSchemaPlugin | 타입별 metadata JSON Schema 반환 | 새로운 기능 |
| EditorPlugin | 편집기 동작 규칙 (허용 노드 타입 등) | Phase 4/6 |
| RendererPlugin | 노드 렌더링 커스터마이징 | Phase 6 |
| ChunkingPlugin | chunking_config 반환 | Phase 10 |
| RAGPlugin | PromptTemplate, ContextConfig 반환 | Phase 11 |
| SearchPlugin | search_config 반환 | Phase 8 |
| WorkflowPlugin | 워크플로 단계/규칙 반환 | Phase 5 |

---

### 4-3. DocumentTypeRegistry 설계

```
싱글턴 레지스트리
  - 내장 플러그인: 시스템 부팅 시 코드로 등록
  - DB 기반 설정: 런타임에 document_type_configs 테이블에서 로드
  - 캐시: DB 기반 설정은 메모리 캐시 (TTL 5분)

조회 우선순위:
  1. 코드 등록 플러그인 (내장 타입)
  2. DB 기반 설정 (Admin UI 편집 내용으로 오버라이드)
  3. DefaultDocumentTypePlugin (폴백)
```

---

### 4-4. 기존 코드 마이그레이션 전략

하위 호환성을 유지하며 점진적 마이그레이션:

| 단계 | 내용 |
|------|------|
| 1단계 (Phase 12) | 플러그인 레지스트리 구현, 기존 코드는 레지스트리 경유로 리다이렉트 |
| 2단계 (Phase 12) | 기존 직접 읽기 코드를 플러그인 경유 코드로 교체 |
| 3단계 (유지보수) | 신규 타입은 처음부터 플러그인으로만 추가 |

기존 코드 패턴 → 플러그인 패턴:

```
# 기존 (Phase 10)
chunking_config = DocumentTypeRepository.get_config(doc_type)["chunking_config"]

# 플러그인 경유 (Phase 12 이후)
chunking_config = DocumentTypeRegistry.get(doc_type).chunking_plugin().get_config()
```

---

### 4-5. 플러그인 규격 정의

플러그인이 준수해야 할 최소 규격:

| 항목 | 요구사항 |
|------|---------|
| type_name | 영문 대문자, 밑줄만 허용 (예: POLICY, WORK_ORDER) |
| 등록 중복 | 동일 type_name 재등록 시 오류 |
| 서브플러그인 미구현 | 기본 구현체 자동 사용 |
| 설정 JSON 검증 | 플러그인 로드 시 schema 검증 |

---

## 5. 산출물

1. 플러그인 계층 구조 설계서
2. DocumentTypeRegistry 설계서
3. 서브플러그인 역할 분리 문서
4. 기존 코드 마이그레이션 전략 문서
5. 플러그인 규격 정의서

---

## 6. 완료 기준

- 플러그인 계층 구조가 서브플러그인 역할 분리와 함께 설계되어 있다
- DocumentTypeRegistry 발견 및 조회 메커니즘이 명확히 정의되어 있다
- 기존 Phase 8/10/11 코드와의 하위 호환 마이그레이션 전략이 수립되어 있다
- 이후 Task(12-2~12-10)가 이 문서를 기준으로 진행 가능하다

---

## 7. Codex 작업 지침

- 실제 코드 구현은 하지 않는다 (설계 중심)
- CLAUDE.md 원칙 "문서 타입은 하드코딩 금지"를 플러그인 아키텍처의 설계 근거로 명시한다
- 플러그인 레지스트리는 싱글턴으로 설계하되 테스트에서 재설정(reset) 가능하도록 한다
- 외부 플러그인 파일 로딩(Dynamic Loading)은 이 단계에서 Reserved로만 표시하고 구현하지 않는다
