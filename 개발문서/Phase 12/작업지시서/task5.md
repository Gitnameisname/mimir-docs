# Task 12-5. 타입별 chunking 플러그인 (Phase 10 통합)

## 1. 작업 목적

Phase 10의 ChunkingService가 DocumentType별 chunking_config를 플러그인 레지스트리를 통해 읽도록 통합한다.

이 작업의 목표는 다음과 같다.

- ChunkingPlugin 인터페이스 상세 구현
- Phase 10 ChunkingService를 DocumentTypeRegistry 경유로 마이그레이션
- 내장 타입별 최적화된 chunking_config 정의
- 기존 직접 읽기 코드 교체 및 하위 호환성 보장

---

## 2. 작업 범위

### 포함 범위

- ChunkingPlugin 인터페이스 구현
- Phase 10 ChunkingService 통합 수정
- 내장 타입별 chunking_config 최적화
- 마이그레이션 및 테스트

### 제외 범위

- ChunkingService 내부 알고리즘 변경 (Phase 10에서 완성됨)
- 새 청킹 전략 추가 (별도 작업)

---

## 3. 선행 조건

- Task 12-2 (DocumentTypePlugin 인터페이스) 완료
- Phase 10 Task 10-5 (ChunkingService) 코드 이해

---

## 4. 주요 구현 대상

### 4-1. ChunkingPlugin 인터페이스

```python
class ChunkingPlugin:
    def get_config(self) -> ChunkingConfig:
        """타입별 ChunkingConfig 반환"""
        return ChunkingConfig()  # 기본값 사용

    def should_index(self, version_status: str) -> bool:
        """해당 버전 상태의 문서를 벡터화할지 여부"""
        policy = self.get_config().index_version_policy
        if policy == "published_only":
            return version_status == "PUBLISHED"
        elif policy == "latest":
            return True
        elif policy == "all":
            return True
        return False
```

---

### 4-2. 내장 타입별 ChunkingPlugin 설정

| 타입 | max_tokens | 특이사항 |
|------|-----------|---------|
| POLICY | 512 | 조항 번호 컨텍스트 포함, exclude: [metadata, attachment] |
| MANUAL | 400 | 절차 단계 단위, min_chunk_tokens=30 |
| REPORT | 600 | 섹션 단위, overlap_tokens=100 |
| FAQ | 256 | Q&A 쌍 단위, merge_strategy=merge_qa_pair |

```python
class PolicyChunkingPlugin(ChunkingPlugin):
    def get_config(self) -> ChunkingConfig:
        return ChunkingConfig(
            strategy="node_based",
            max_chunk_tokens=512,
            min_chunk_tokens=50,
            overlap_tokens=50,
            include_parent_context=True,
            parent_context_depth=2,
            index_version_policy="published_only",
            exclude_node_types=["metadata", "attachment"],
            merge_strategy="merge_siblings",
        )

class FAQChunkingPlugin(ChunkingPlugin):
    def get_config(self) -> ChunkingConfig:
        return ChunkingConfig(
            strategy="node_based",
            max_chunk_tokens=256,
            min_chunk_tokens=20,
            overlap_tokens=0,
            include_parent_context=False,
            parent_context_depth=0,
            index_version_policy="published_only",
            exclude_node_types=[],
            merge_strategy="merge_qa_pair",
        )
```

---

### 4-3. Phase 10 ChunkingService 마이그레이션

기존:

```python
# Phase 10 (기존)
def load_config(document_type: str) -> ChunkingConfig:
    doc_type_config = DocumentTypeRepository.get_config(document_type)
    chunking_raw = doc_type_config.get("chunking_config", {})
    return ChunkingConfig(**chunking_raw)
```

마이그레이션 후:

```python
# Phase 12 (플러그인 경유)
def load_config(document_type: str) -> ChunkingConfig:
    plugin = DocumentTypeRegistry.instance().get(document_type)
    return plugin.chunking_plugin().get_config()
```

하위 호환성: DB에 chunking_config가 있으면 ConfigurableChunkingPlugin이 DB 값을 우선 사용.

---

### 4-4. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| POLICY ChunkingPlugin | max_chunk_tokens=512 |
| FAQ ChunkingPlugin | max_chunk_tokens=256, overlap=0 |
| should_index (published_only) | PUBLISHED=true, DRAFT=false |
| DB 설정 오버라이드 | DB 값이 플러그인 기본값보다 우선 |

---

## 5. 산출물

1. ChunkingPlugin 인터페이스 구현
2. 내장 타입별 ChunkingPlugin 구현 (POLICY, MANUAL, REPORT, FAQ)
3. Phase 10 ChunkingService 마이그레이션 코드
4. 단위 테스트

---

## 6. 완료 기준

- ChunkingService가 DocumentTypeRegistry 경유로 chunking_config를 로드한다
- 내장 4개 타입의 최적화된 chunking 설정이 적용된다
- 기존 DB 기반 설정이 플러그인 기본값보다 우선 적용된다
- 단위 테스트가 통과한다

---

## 7. Codex 작업 지침

- Phase 10 ChunkingService 수정은 최소화하며 load_config() 함수 교체만 진행한다
- FAQ의 merge_strategy="merge_qa_pair"는 Phase 10에서 구현되지 않은 전략이므로, 지원되지 않으면 "merge_siblings"로 폴백하고 로그를 남긴다
- DB 기반 설정 오버라이드는 개별 필드 단위로 병합(merge)하며 누락된 필드는 플러그인 기본값 사용
