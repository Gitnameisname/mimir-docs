# Task 10-8. 권한 메타데이터 반영 구현

## 1. 작업 목적

문서 ACL(권한) 변경 시 document_chunks의 권한 메타데이터를 재벡터화 없이 업데이트하는 메커니즘을 구현한다.

이 작업의 목표는 다음과 같다.

- ACL 변경 이벤트 구독 및 청크 권한 메타데이터 갱신
- 벡터 재계산 없이 권한 필드만 UPDATE
- 권한 메타데이터의 정합성 검증 로직 구현
- 권한 변경 감사 이벤트 기록

---

## 2. 작업 범위

### 포함 범위

- ACL 변경 이벤트 핸들러 구현
- 청크 권한 메타데이터 일괄 UPDATE
- 권한 정합성 검증 API
- 감사 이벤트 기록

### 제외 범위

- ACL 관리 자체 (Phase 4에서 구현됨)
- 벡터 검색 시 권한 필터 (Task 10-9에서 다룸)

---

## 3. 선행 조건

- Task 10-3 (pgvector 스키마) 완료 — accessible_roles, accessible_org_ids, is_public 컬럼 존재
- Task 10-6 (벡터화 파이프라인) 완료 — 청크 생성 시 권한 스냅샷 포함
- Phase 4: ACL 이벤트 구조 이해

---

## 4. 주요 구현 대상

### 4-1. 권한 메타데이터 갱신 서비스

```
PermissionSyncService
  .sync_document_permissions(document_id) → int  (갱신된 청크 수)
  .sync_all_stale_permissions() → int  (전체 갱신 수)
  ._get_current_acl(document_id) → ACLSnapshot
```

#### 갱신 쿼리

```sql
UPDATE document_chunks
SET
  accessible_roles = :roles,
  accessible_org_ids = :org_ids,
  is_public = :is_public,
  updated_at = now()
WHERE document_id = :document_id
  AND is_current = true;
```

임베딩(embedding) 컬럼은 변경하지 않아 재벡터화 비용 없이 처리.

---

### 4-2. ACL 변경 이벤트 핸들러

Phase 4 ACL 변경 이벤트를 구독하여 자동 갱신:

```python
@event_handler("DOCUMENT_ACL_CHANGED")
def on_acl_changed(event):
    document_id = event.payload["document_id"]
    updated_count = PermissionSyncService.sync_document_permissions(document_id)
    AuditService.record("CHUNK_PERMISSIONS_SYNCED", {
        "document_id": document_id,
        "updated_chunk_count": updated_count,
    })
```

#### 이벤트 발생 시점

| 이벤트 | 설명 |
|-------|------|
| `DOCUMENT_ACL_CHANGED` | 역할/조직 접근 권한 변경 |
| `DOCUMENT_VISIBILITY_CHANGED` | is_public 변경 |
| `DOCUMENT_ORG_TRANSFERRED` | 문서 조직 이동 |

---

### 4-3. 권한 정합성 검증 API

Admin이 청크와 원본 문서 ACL의 정합성을 확인:

```
GET /api/admin/vectorization/permission-sync-status

Response:
  {
    "total_documents_with_chunks": int,
    "stale_permission_count": int,  (청크 ACL ≠ 문서 ACL인 건 수)
    "last_checked_at": ISO8601
  }

POST /api/admin/vectorization/sync-permissions
Body:
  {
    "scope": "document" | "all",
    "document_id": str (scope=document 시 필수)
  }

Response:
  {
    "updated_chunk_count": int
  }
```

---

### 4-4. 권한 스냅샷 비교 로직

청크의 권한 메타데이터와 현재 문서 ACL을 비교하여 stale 여부 판별:

```python
def is_permission_stale(chunk, current_acl) -> bool:
    return (
        set(chunk.accessible_roles) != set(current_acl.roles)
        or set(chunk.accessible_org_ids) != set(current_acl.org_ids)
        or chunk.is_public != current_acl.is_public
    )
```

---

### 4-5. 배치 갱신 최적화

대량 문서의 권한 일괄 갱신 시 배치 처리:

```python
def sync_all_stale_permissions():
    # 정합성이 맞지 않는 document_id 목록 조회
    stale_docs = find_documents_with_stale_permissions()
    total = 0
    for doc_id in stale_docs:
        total += PermissionSyncService.sync_document_permissions(doc_id)
    return total
```

---

## 5. 산출물

1. PermissionSyncService 구현 코드
2. ACL 변경 이벤트 핸들러
3. 권한 정합성 검증 Admin API
4. 배치 권한 갱신 구현

---

## 6. 완료 기준

- 문서 ACL 변경 시 document_chunks의 권한 메타데이터가 자동 갱신된다
- 권한 갱신 시 embedding 컬럼은 변경되지 않는다 (재벡터화 없음)
- 정합성 검증 API가 stale 청크 수를 반환한다
- 권한 변경 감사 이벤트가 기록된다

---

## 7. Codex 작업 지침

- 권한 갱신은 is_current=true 청크에만 적용한다 (stale 청크는 어차피 삭제 예정)
- ACL 이벤트 핸들러 실패 시 Job Queue에 재시도 항목으로 등록한다
- 권한 갱신과 벡터화는 독립적인 경로로 유지하여 각각 실패해도 다른 경로에 영향이 없도록 한다
- accessible_roles 등 권한 필드는 배열 비교 시 순서 무관하게 처리한다
