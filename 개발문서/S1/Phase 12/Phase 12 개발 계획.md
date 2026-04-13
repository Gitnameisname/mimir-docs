# Phase 12. 문서 유형 확장 프레임워크 구축

## 1. Phase 목적

새로운 DocumentType을 코드 변경 없이 플러그인 방식으로 추가할 수 있는 확장 프레임워크를 구축한다.

CLAUDE.md의 핵심 원칙 "문서 타입은 하드코딩 금지 — 구조는 generic + config 기반"을 프레임워크 수준에서 완성하여, 각 Phase에서 type-aware 처리를 위해 추가한 분기 로직들을 플러그인 구조로 통합한다.

---

## 2. Phase 12 범위

### 포함 범위

- DocumentType 플러그인 인터페이스 설계 및 구현
- 플러그인 레지스트리 (등록/조회 메커니즘)
- 타입별 metadata schema 관리 (JSON Schema 기반)
- 타입별 editor/renderer 플러그인
- 타입별 chunking 전략 플러그인 (Phase 10 연계)
- 타입별 RAG 프롬프트 플러그인 (Phase 11 연계)
- 타입별 검색 설정 플러그인 (Phase 8 연계)
- 내장 DocumentType 구현 (POLICY, MANUAL, REPORT, FAQ)
- Admin DocumentType 관리 UI (플러그인 설정 편집)
- 플러그인 테스트 프레임워크

### 제외 범위

- 운영/보안/배포 체계 (Phase 13에서 다룸)
- 새 DocumentType 비즈니스 로직 (플러그인으로 독립 구현)

---

## 3. 선행 조건

- Phase 1~11 완료 — Document, Versioning, Workflow, Search, Vectorization, RAG 기반 완성
- Phase 7 Admin UI — DocumentType 관리 화면 기반 존재
- Phase 8 search_config 패턴 이해
- Phase 10 chunking_config 패턴 이해
- Phase 11 PromptTemplate / DocumentTypeRAGPlugin 인터페이스 참조

---

## 4. 핵심 설계 원칙

### 4-1. 플러그인 구조

```
DocumentTypePlugin (인터페이스)
  ├── MetadataSchemaPlugin     타입별 metadata JSON Schema
  ├── EditorPlugin             타입별 편집기 동작 규칙
  ├── RendererPlugin           타입별 렌더링 규칙
  ├── ChunkingPlugin           타입별 청킹 전략 (Phase 10 확장)
  ├── RAGPlugin                타입별 RAG 프롬프트/설정 (Phase 11 확장)
  ├── SearchPlugin             타입별 검색 설정 (Phase 8 확장)
  └── WorkflowPlugin           타입별 워크플로 단계 (Phase 5 확장)
```

각 서브플러그인은 독립적으로 구현 가능하며, 구현하지 않으면 기본 구현이 사용됨.

---

### 4-2. 플러그인 레지스트리

```
DocumentTypeRegistry
  .register(plugin: DocumentTypePlugin) → None
  .get(document_type: str) → DocumentTypePlugin
  .list_all() → list[DocumentTypePlugin]
  .get_metadata_schema(document_type: str) → JSONSchema
  .get_chunking_config(document_type: str) → ChunkingConfig
  .get_rag_plugin(document_type: str) → RAGPlugin
  .get_search_config(document_type: str) → SearchConfig
```

---

### 4-3. 플러그인 발견 방식

| 방식 | 설명 | 사용 시점 |
|------|------|---------|
| 코드 등록 | 내장 타입 (POLICY 등) | 시스템 부팅 시 자동 등록 |
| DB 기반 설정 | Admin UI에서 관리하는 config | 런타임 로드 |
| 파일 기반 | 향후 외부 플러그인 | Reserved |

---

### 4-4. 내장 DocumentType

| 타입 | 설명 | 특수 동작 |
|------|------|---------|
| POLICY | 정책/규정 | 조항 단위 청킹, 조항 번호 컨텍스트 |
| MANUAL | 매뉴얼/절차서 | 절차 단계 청킹, 단계 번호 강조 |
| REPORT | 보고서 | 섹션 단위 청킹, 요약 섹션 우선 |
| FAQ | FAQ | Q&A 쌍 단위 청킹 (소형) |

---

## 5. Task 목록

| Task | 이름 | 주요 내용 |
|------|------|---------|
| 12-1 | 플러그인 아키텍처 설계 | 전체 플러그인 구조, 레지스트리, 발견 메커니즘 |
| 12-2 | DocumentTypePlugin 인터페이스 구현 | 기반 인터페이스, 기본 구현체, 레지스트리 |
| 12-3 | 타입별 metadata schema 플러그인 | JSON Schema 기반 metadata 검증/편집 |
| 12-4 | 타입별 editor/renderer 플러그인 | 편집기 동작 규칙, 렌더링 커스터마이징 |
| 12-5 | 타입별 chunking 플러그인 (Phase 10 통합) | ChunkingPlugin, Phase 10 ChunkingService 연계 |
| 12-6 | 타입별 RAG 플러그인 (Phase 11 통합) | RAGPlugin, PromptTemplate, ContextConfig |
| 12-7 | 타입별 검색 플러그인 (Phase 8 통합) | SearchPlugin, search_config 통합 |
| 12-8 | 내장 DocumentType 구현 (POLICY, MANUAL, REPORT, FAQ) | 4개 내장 타입 플러그인 구현 |
| 12-9 | Admin DocumentType 관리 UI 확장 | 플러그인 설정 편집, 스키마 관리 |
| 12-10 | 플러그인 테스트 프레임워크 및 검증 | 플러그인 규격 준수 검증, 통합 테스트 |

---

## 6. 데이터 구조

### document_type_configs 테이블 (기존 확장)

```sql
ALTER TABLE document_type_configs
  ADD COLUMN chunking_config    JSONB DEFAULT '{}',
  ADD COLUMN rag_config         JSONB DEFAULT '{}',
  ADD COLUMN search_config      JSONB DEFAULT '{}',  -- 기존 Phase 8
  ADD COLUMN editor_config      JSONB DEFAULT '{}',
  ADD COLUMN renderer_config    JSONB DEFAULT '{}',
  ADD COLUMN metadata_schema    JSONB DEFAULT '{}',
  ADD COLUMN workflow_config    JSONB DEFAULT '{}';
```

각 config 컬럼은 해당 플러그인의 설정을 JSON으로 저장하며, Admin UI에서 편집 가능.

---

## 7. 기존 Phase 코드와의 통합

| Phase | 기존 패턴 | Phase 12 통합 방식 |
|-------|---------|-----------------|
| Phase 8 | search_config 직접 읽기 | SearchPlugin.get_config() 경유 |
| Phase 10 | chunking_config 직접 읽기 | ChunkingPlugin.get_config() 경유 |
| Phase 11 | PromptTemplate 팩토리 | RAGPlugin.get_prompt_template() 경유 |
| Phase 5 | 워크플로 단계 설정 | WorkflowPlugin.get_steps() 경유 |

기존 직접 읽기 코드는 하위 호환성 유지하며 점진적으로 플러그인 경유로 마이그레이션.

---

## 8. Phase 13 연계 고려사항

- 플러그인 코드는 테스트 커버리지 보고 대상에 포함
- 플러그인 레지스트리 초기화는 애플리케이션 부팅 시 검증
- 플러그인 설정 변경 시 감사 이벤트 기록 (Phase 7 감사 로그 연계)

---

## 9. 완료 기준

- 새 DocumentType을 코드 변경 없이 플러그인 등록만으로 추가할 수 있다
- 기존 POLICY, MANUAL, REPORT, FAQ 타입이 플러그인으로 동작한다
- Admin UI에서 타입별 설정(metadata schema, chunking, RAG, 검색)을 편집할 수 있다
- 모든 type-aware 로직이 플러그인 레지스트리를 경유한다
