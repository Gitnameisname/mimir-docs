# Task 10-6. 벡터화 파이프라인 구현 (청킹 + 임베딩 + 저장)

## 1. 작업 목적

ChunkingService, EmbeddingService, VectorStoreService를 연결하여 완전한 벡터화 파이프라인을 구현한다.

이 작업의 목표는 다음과 같다.

- EmbeddingService 구현 (ChunkingService + EmbeddingProvider 연결)
- VectorStoreService 구현 (document_chunks 테이블 upsert)
- VectorizationJob 구현 (Phase 7 Background Job 연계)
- 파이프라인 전체 흐름 통합 테스트

---

## 2. 작업 범위

### 포함 범위

- EmbeddingService 구현
- VectorStoreService 구현 (upsert, is_current 처리)
- VectorizationJob 구현 (VECTORIZE_DOCUMENT, VECTORIZE_BATCH)
- 파이프라인 통합 테스트

### 제외 범위

- CLEANUP_STALE_CHUNKS Job (Task 10-7에서 다룸)
- 권한 메타데이터 갱신 (Task 10-8에서 다룸)
- Admin UI (Task 10-10에서 다룸)

---

## 3. 선행 조건

- Task 10-3 (pgvector 스키마) 완료 — document_chunks 테이블 존재
- Task 10-4 (EmbeddingProvider) 완료 — OpenAIEmbeddingProvider 사용 가능
- Task 10-5 (ChunkingService) 완료 — Chunk[] 반환 가능
- Phase 7 Background Job 시스템 이해

---

## 4. 주요 구현 대상

### 4-1. EmbeddingService

ChunkingService와 EmbeddingProvider를 연결하여 Chunk[]를 임베딩 포함 Chunk[]로 변환:

```
EmbeddingService
  .embed_chunks(chunks: Chunk[]) → EmbeddedChunk[]
  .embed_document(document_id, version_id) → EmbeddedChunk[]
```

#### 처리 흐름

```
1. chunks = ChunkingService.chunk_document(document_id, version_id)
2. texts = [chunk.context_text for chunk in chunks]
3. results = EmbeddingProvider.embed(texts)
4. for i, chunk in enumerate(chunks):
     chunk.embedding = results[i].embedding
     chunk.token_count = results[i].token_count
     chunk.embedding_model = EmbeddingProvider.get_model_name()
5. return embedded_chunks
```

---

### 4-2. VectorStoreService

document_chunks 테이블에 청크를 저장하고 기존 청크를 소프트 삭제:

```
VectorStoreService
  .store_chunks(chunks: EmbeddedChunk[]) → int  (저장된 청크 수)
  .deactivate_previous_chunks(document_id, version_id) → int  (비활성화 수)
  .get_chunk_count(document_id) → int
```

#### upsert 처리 순서

```
1. 새 청크 배치 INSERT (is_current=true)
2. 동일 document_id + version_id의 이전 청크를 is_current=false로 UPDATE
   (단, 방금 INSERT한 청크 제외)
3. 저장 수 반환
```

#### 배치 INSERT 최적화

한 번에 최대 500건씩 배치 INSERT:

```python
def store_chunks(chunks):
    for batch in chunks(chunks, 500):
        db.execute_many(
            "INSERT INTO document_chunks (...) VALUES ...",
            [chunk.to_dict() for chunk in batch]
        )
```

---

### 4-3. VectorizationJob 구현

Phase 7 Background Job 시스템에 등록할 Job 핸들러:

#### VECTORIZE_DOCUMENT

```python
class VectorizeDocumentJob:
    job_type = "VECTORIZE_DOCUMENT"

    def execute(self, payload):
        document_id = payload["document_id"]
        version_id = payload["version_id"]

        # 1. 청크 생성 + 임베딩
        embedded_chunks = EmbeddingService.embed_document(document_id, version_id)

        # 2. 저장 (새 청크 INSERT + 이전 청크 비활성화)
        count = VectorStoreService.store_chunks(embedded_chunks)
        VectorStoreService.deactivate_previous_chunks(document_id, version_id)

        # 3. 완료 이벤트
        AuditService.record("VECTORIZATION_COMPLETE", {
            "document_id": document_id,
            "chunk_count": count,
        })
        return {"chunk_count": count}
```

#### VECTORIZE_BATCH

```python
class VectorizeBatchJob:
    job_type = "VECTORIZE_BATCH"

    def execute(self, payload):
        document_type = payload.get("document_type")  # None이면 전체
        documents = DocumentRepository.find_published(document_type=document_type)

        results = []
        for doc in documents:
            sub_job = JobQueue.enqueue("VECTORIZE_DOCUMENT", {
                "document_id": doc.id,
                "version_id": doc.current_version_id,
            })
            results.append(sub_job.id)

        return {"enqueued": len(results), "job_ids": results}
```

---

### 4-4. 트리거: Published 전이 연동

문서가 Published 상태로 전이될 때 자동으로 벡터화 Job 생성:

```python
# WorkflowService 또는 DocumentService의 상태 전이 핸들러에 추가
def on_document_published(document_id, version_id):
    JobQueue.enqueue("VECTORIZE_DOCUMENT", {
        "document_id": document_id,
        "version_id": version_id,
    })
```

---

### 4-5. 통합 테스트

실제 DB(pgvector)와 Mock EmbeddingProvider를 사용한 통합 테스트:

| 시나리오 | 기대 결과 |
|---------|----------|
| 단일 문서 벡터화 | document_chunks에 청크 저장 |
| 재벡터화 시 기존 청크 처리 | 이전 청크 is_current=false |
| VECTORIZE_BATCH 실행 | VECTORIZE_DOCUMENT Job N개 생성 |
| 임베딩 실패 시 Job 상태 | FAILED로 기록, 재시도 큐 등록 |

---

## 5. 산출물

1. EmbeddingService 구현 코드
2. VectorStoreService 구현 코드
3. VectorizationJob 핸들러 (VECTORIZE_DOCUMENT, VECTORIZE_BATCH)
4. Published 전이 트리거 연동 코드
5. 통합 테스트

---

## 6. 완료 기준

- 문서 Published 전이 시 자동으로 벡터화 Job이 생성된다
- VectorizationJob 실행 시 document_chunks에 청크가 저장된다
- 재벡터화 시 이전 청크가 is_current=false로 처리된다
- 임베딩 실패 시 Job이 FAILED 상태로 기록되고 재시도된다
- 통합 테스트가 통과한다

---

## 7. Codex 작업 지침

- VectorStoreService의 INSERT와 이전 청크 비활성화는 트랜잭션으로 묶는다 (원자성 보장)
- VECTORIZE_BATCH는 개별 문서를 서브 Job으로 분산하여 단일 Job 타임아웃을 방지한다
- Published 전이 트리거는 워크플로 로직과 강하게 결합하지 않도록 이벤트 기반으로 구현한다
- 배치 INSERT 크기(500)는 설정으로 조정 가능하게 한다
