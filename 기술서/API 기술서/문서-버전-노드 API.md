# 문서 · 버전 · 노드 API

Base URL: `/api/v1`

---

## 1. 문서 (Documents)

### GET /documents — 문서 목록 조회

```
GET /api/v1/documents
Authorization: Bearer <token>
```

**쿼리 파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `page` | integer | 1 | 페이지 번호 |
| `page_size` | integer | 20 | 페이지 크기 (최대 100) |
| `sort_by` | string | `created_at` | `created_at`, `updated_at`, `title`, `status` |
| `sort_order` | string | `desc` | `asc`, `desc` |
| `filter[status]` | string | - | `draft`, `published`, `archived`, `deprecated` |
| `filter[document_type]` | string | - | 문서 유형 코드 |
| `search` | string | - | 제목/본문 키워드 검색 |

**응답:**
```json
{
  "status": "success",
  "data": [
    {
      "id": "550e8400-...",
      "title": "정보보안 정책",
      "document_type": "POLICY",
      "status": "published",
      "summary": "전사 정보보안 기준",
      "created_by": "user-001",
      "created_at": "2026-01-15T09:00:00+00:00",
      "updated_at": "2026-03-20T14:30:00+00:00"
    }
  ],
  "meta": { "total": 42, "page": 1, "page_size": 20, "total_pages": 3 }
}
```

---

### POST /documents — 문서 생성

```
POST /api/v1/documents
Authorization: Bearer <token>
Idempotency-Key: <uuid>
Content-Type: application/json

{
  "title": "신규 정보보안 정책",
  "document_type": "POLICY",
  "summary": "2026년 개정 정보보안 정책",
  "metadata": {
    "effective_date": "2026-06-01",
    "department": "IT보안팀",
    "author": "홍길동"
  }
}
```

**필수 필드:** `title`, `document_type`  
**응답:** `201 Created` + 생성된 Document

---

### GET /documents/{id} — 문서 상세 조회

```
GET /api/v1/documents/550e8400-e29b-41d4-a716-446655440000
Authorization: Bearer <token>
```

응답에 `current_draft_version_id`, `current_published_version_id` 포함.

---

### PUT /documents/{id} — 문서 메타데이터 수정

```
PUT /api/v1/documents/550e8400-...
Authorization: Bearer <token>
Content-Type: application/json

{
  "title": "수정된 제목",
  "summary": "업데이트된 요약",
  "metadata": { "department": "법무팀" }
}
```

문서 내용(Node)은 이 엔드포인트로 수정하지 않는다. 버전/노드 API를 사용.

---

### DELETE /documents/{id} — 문서 삭제

```
DELETE /api/v1/documents/550e8400-...
Authorization: Bearer <token>
```

`SUPER_ADMIN` 권한 필요. 연관된 모든 버전, 노드, 청크 삭제 (CASCADE).

---

## 2. 버전 (Versions)

### GET /documents/{id}/versions — 버전 목록

```
GET /api/v1/documents/550e8400-.../versions
Authorization: Bearer <token>
```

**쿼리 파라미터:**

| 파라미터 | 설명 |
|----------|------|
| `filter[status]` | `draft`, `published`, `archived` |
| `sort_by` | `created_at`, `version_number` |

---

### POST /documents/{id}/versions — 새 버전 생성

```
POST /api/v1/documents/550e8400-.../versions
Authorization: Bearer <token>

{
  "label": "v3.0",
  "change_summary": "전면 개정",
  "source": "manual"
}
```

새 버전은 `draft` 상태로 생성. `version_number`는 자동 증가.

---

### GET /versions/{vid} — 버전 상세 조회

```
GET /api/v1/versions/660e8400-...
Authorization: Bearer <token>
```

---

### GET /versions/{vid}/render — 버전 렌더링

```
GET /api/v1/versions/660e8400-.../render
Authorization: Bearer <token>
```

Version의 모든 Node를 트리 구조로 조합하여 렌더링된 문서 반환.

---

### POST /documents/{id}/versions/{vid}/restore — 버전 복원

```
POST /api/v1/documents/550e8400-.../versions/660e8400-.../restore
Authorization: Bearer <token>

{
  "change_summary": "v2.1로 롤백"
}
```

지정 버전의 내용 스냅샷으로 새 Draft 버전 생성.

---

### POST /versions/{vid}/draft/save — Draft 저장

```
POST /api/v1/versions/660e8400-.../draft/save
Authorization: Bearer <token>

{
  "title": "업데이트된 제목",
  "summary": "업데이트된 요약",
  "nodes": [...]
}
```

Draft 버전의 메타데이터와 노드 트리를 일괄 저장.

---

## 3. 노드 (Nodes)

### GET /versions/{vid}/nodes — 노드 목록 (트리)

```
GET /api/v1/versions/660e8400-.../nodes
Authorization: Bearer <token>
```

해당 버전의 모든 노드를 평면 목록으로 반환. 트리 구조 재구성은 클라이언트가 담당.

---

### POST /versions/{vid}/nodes — 노드 생성

```
POST /api/v1/versions/660e8400-.../nodes
Authorization: Bearer <token>

{
  "node_type": "paragraph",
  "parent_id": "770e8400-...",
  "order_index": 2,
  "title": null,
  "content": "이 단락은 정보보안의 기본 원칙을 설명합니다.",
  "metadata": {}
}
```

---

### PUT /versions/{vid}/nodes/{nid} — 노드 수정

```
PUT /api/v1/versions/660e8400-.../nodes/770e8400-...
Authorization: Bearer <token>

{
  "content": "수정된 내용",
  "order_index": 3
}
```

---

### DELETE /versions/{vid}/nodes/{nid} — 노드 삭제

```
DELETE /api/v1/versions/660e8400-.../nodes/770e8400-...
Authorization: Bearer <token>
```

하위 노드(children)도 함께 삭제 (`parent_id SET NULL`이 아닌 CASCADE).

---

## 4. Diff (버전 비교)

### GET /documents/{id}/versions/{vid}/diff/{vid2} — 두 버전 비교

```
GET /api/v1/documents/550e8400-.../versions/v1id.../diff/v2id...
Authorization: Bearer <token>
```

두 버전 간 노드 단위 변경 사항 (추가/삭제/수정) 반환.

### GET /documents/{id}/versions/{vid}/diff/summary — 변경 요약

```
GET /api/v1/documents/550e8400-.../versions/v2id.../diff/summary
Authorization: Bearer <token>
```

이전 버전 대비 변경 요약 (추가 노드 수, 삭제 노드 수, 수정 노드 수).

---

## 5. 공통 응답 필드

### Document

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | UUID | 문서 고유 ID |
| `title` | string | 문서 제목 |
| `document_type` | string | 문서 유형 코드 |
| `status` | string | `draft` \| `published` \| `archived` \| `deprecated` |
| `metadata` | object | 타입별 확장 필드 |
| `summary` | string? | 문서 요약 |
| `current_draft_version_id` | UUID? | 현재 Draft 버전 ID |
| `current_published_version_id` | UUID? | 현재 Published 버전 ID |
| `created_by` | string? | 생성자 ID |
| `created_at` | datetime | 생성 시각 (ISO 8601) |
| `updated_at` | datetime | 수정 시각 (ISO 8601) |

### Version

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | UUID | 버전 고유 ID |
| `document_id` | UUID | 소속 문서 ID |
| `version_number` | integer | 순차 버전 번호 |
| `label` | string? | 버전 레이블 |
| `status` | string | `draft` \| `published` \| `archived` |
| `workflow_status` | string | 워크플로 진행 상태 |
| `change_summary` | string? | 변경 요약 |
| `created_by` | string? | 작성자 ID |
| `created_at` | datetime | 생성 시각 |
