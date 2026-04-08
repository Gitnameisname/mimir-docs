# Task 10-7. 재색인 정책 및 Cleanup 구현

## 1. 작업 목적

문서 재벡터화 시 기존 청크의 소프트 삭제 및 물리 삭제 정책을 구현한다.

이 작업의 목표는 다음과 같다.

- 재색인 트리거 조건 정의 및 구현
- is_current=false 청크 물리 삭제 Job 구현 (CLEANUP_STALE_CHUNKS)
- 재색인 중 검색 서비스 중단 없이 전환 보장
- 관리자 수동 재색인 API 구현

---

## 2. 작업 범위

### 포함 범위

- 재색인 트리거 조건 정의
- CLEANUP_STALE_CHUNKS Job 구현
- 수동 재색인 Admin API
- 재색인 상태 추적

### 제외 범위

- Admin UI (Task 10-10에서 다룸)
- 성능 튜닝 (Task 10-11에서 다룸)

---

## 3. 선행 조건

- Task 10-6 (벡터화 파이프라인) 완료 — VECTORIZE_DOCUMENT, VECTORIZE_BATCH 사용 가능
- Task 10-3 (pgvector 스키마) 완료 — is_current 컬럼 존재

---

## 4. 주요 구현 대상

### 4-1. 재색인 트리거 조건

| 트리거 | 설명 | 처리 방식 |
|--------|------|---------|
| 문서 Published 전이 | 새 버전 Published | VECTORIZE_DOCUMENT 자동 생성 |
| 수동 재색인 (Admin) | 관리자 수동 요청 | VECTORIZE_DOCUMENT 즉시 생성 |
| 임베딩 모델 변경 | 모델 교체 후 전체 재색인 필요 | VECTORIZE_ALL 생성 |
| DocumentType 청킹 설정 변경 | chunking_config 변경 | 해당 타입 VECTORIZE_BATCH 생성 |

---

### 4-2. 소프트 삭제 전략 (재확인)

Task 10-3 설계를 구현:

```
재색인 시:
1. 새 청크 INSERT (is_current=true, created_at=now)
2. 동일 document_id의 이전 청크 UPDATE is_current=false
   (단, version_id가 동일하면서 방금 INSERT된 청크 제외)
3. is_current=false 청크는 정리 배치가 도달할 때까지 DB에 잔류
```

이 방식으로 재색인 도중에도 이전 청크가 검색 서비스에서 계속 사용됨.

---

### 4-3. CLEANUP_STALE_CHUNKS Job

```python
class CleanupStaleChunksJob:
    job_type = "CLEANUP_STALE_CHUNKS"
    default_retention_days = 7

    def execute(self, payload):
        retention_days = payload.get("retention_days", self.default_retention_days)
        cutoff = now() - timedelta(days=retention_days)

        deleted = db.execute("""
            DELETE FROM document_chunks
            WHERE is_current = false
              AND updated_at < :cutoff
            RETURNING id
        """, {"cutoff": cutoff})

        AuditService.record("STALE_CHUNKS_CLEANED", {
            "deleted_count": len(deleted),
            "cutoff": cutoff.isoformat(),
        })
        return {"deleted_count": len(deleted)}
```

#### 스케줄 등록

Phase 7 Background Job 시스템의 Cron 기능 활용:

```
스케줄: 매일 새벽 3시 (0 3 * * *)
Job 타입: CLEANUP_STALE_CHUNKS
파라미터: {"retention_days": 7}
```

---

### 4-4. 수동 재색인 Admin API

```
POST /api/admin/vectorization/reindex
Body:
  {
    "scope": "document" | "document_type" | "all",
    "document_id": str (scope=document 시 필수),
    "document_type": str (scope=document_type 시 필수)
  }

Response:
  {
    "job_id": str,
    "job_type": str,
    "enqueued_at": ISO8601
  }
```

#### scope별 처리

| scope | 동작 |
|-------|------|
| `document` | VECTORIZE_DOCUMENT 1건 생성 |
| `document_type` | VECTORIZE_BATCH (해당 타입) 생성 |
| `all` | VECTORIZE_BATCH (전체) 생성 |

---

### 4-5. 재색인 상태 조회 API

```
GET /api/admin/vectorization/status?document_id=:id

Response:
  {
    "document_id": str,
    "current_chunk_count": int,
    "stale_chunk_count": int,
    "last_vectorized_at": ISO8601 | null,
    "last_job_id": str | null,
    "last_job_status": "PENDING" | "RUNNING" | "COMPLETED" | "FAILED" | null
  }
```

---

### 4-6. VECTORIZE_ALL Job

```python
class VectorizeAllJob:
    job_type = "VECTORIZE_ALL"

    def execute(self, payload):
        documents = DocumentRepository.find_all_published()
        for doc in documents:
            JobQueue.enqueue("VECTORIZE_DOCUMENT", {
                "document_id": doc.id,
                "version_id": doc.current_version_id,
            })
        return {"enqueued": len(documents)}
```

임베딩 모델 교체 후 전체 재색인 시 사용.

---

## 5. 산출물

1. 재색인 트리거 조건 구현 (DocumentType 설정 변경 연동 포함)
2. CLEANUP_STALE_CHUNKS Job 구현
3. Cron 스케줄 등록 코드
4. 수동 재색인 Admin API
5. 재색인 상태 조회 API
6. VECTORIZE_ALL Job 구현

---

## 6. 완료 기준

- 문서 Published 전이 시 자동 재색인이 트리거된다
- CLEANUP_STALE_CHUNKS Job이 7일 이상 경과한 stale 청크를 삭제한다
- 수동 재색인 Admin API가 동작한다
- 재색인 상태 조회 API가 current/stale 청크 수를 반환한다
- 재색인 중 검색 서비스 중단이 없다 (is_current=true 청크 유지)

---

## 7. Codex 작업 지침

- CLEANUP_STALE_CHUNKS의 retention_days는 설정 가능하게 하며, 기본값은 7일이다
- VECTORIZE_ALL은 대규모 작업이므로 Admin 권한 확인 후 실행한다
- 재색인 API는 멱등성을 보장한다 — 동일 요청을 여러 번 해도 중복 Job이 쌓이지 않도록 처리한다
- DocumentType 설정 변경 트리거는 설정 변경 이벤트를 구독하는 방식으로 구현하여 결합도를 낮춘다
