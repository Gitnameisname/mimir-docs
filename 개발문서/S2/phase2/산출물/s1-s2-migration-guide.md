# S1 → S2 Citation 마이그레이션 가이드

## 변경 요약

S2 Phase 2에서 검색 응답에 `citation` 필드가 추가되었습니다.
기존 S1 클라이언트는 이 필드를 무시하면 그대로 동작합니다.

---

## S1 클라이언트 (변경 불필요)

```json
// 기존 검색 결과 필드만 사용 — 정상 동작
{
  "id": "uuid",
  "title": "문서 제목",
  "document_type": "POLICY",
  "status": "published",
  "rank": 0.95,
  "snippets": [...]
}
```

`citation` 필드가 `null`로 내려오더라도 기존 필드 처리에는 영향이 없습니다.

---

## S2 클라이언트 (citation 활용)

```json
// citation 필드 추가 — S2 클라이언트가 활용
{
  "id": "uuid",
  "title": "문서 제목",
  "document_type": "POLICY",
  "status": "published",
  "rank": 0.95,
  "snippets": [...],
  "citation": {
    "document_id": "uuid",
    "version_id": "uuid",
    "node_id": "uuid",
    "span_offset": null,
    "content_hash": "sha256-hex-64chars"
  }
}
```

### Citation 필드 의미

| 필드 | 타입 | 설명 |
|------|------|------|
| `document_id` | UUID | 근거 문서 |
| `version_id` | UUID | 인용 시점의 문서 버전 (개정 후에도 원본 확인 가능) |
| `node_id` | UUID | 문서 내 섹션 위치 |
| `span_offset` | int \| null | 청크 내 문자 오프셋 (단락 단위 인용 시 null) |
| `content_hash` | string (64) | SHA-256(원문 UTF-8) — 변조 감지용 |

---

## Citation 검증 API

### 유효성 검증

```http
GET /api/v1/citations/{document_id}/versions/{version_id}/nodes/{node_id}/verify
  ?content_hash={sha256-hex}
  &span_offset={int}   (선택)
Authorization: Bearer {token}
```

응답:
```json
{
  "verified": true,
  "modified": false,
  "original_text": "청크 원문...",
  "citation": { ... }
}
```

- `verified: true` — 원문이 변경되지 않음
- `verified: false, modified: true` — 문서가 수정되어 원문이 변경됨

### 원문 조회

```http
GET /api/v1/citations/{document_id}/versions/{version_id}/nodes/{node_id}/content
Authorization: Bearer {token}
```

응답:
```json
{
  "content": "청크 원문 전체...",
  "metadata": { "document_type": "...", ... },
  "citation": { ... }
}
```

---

## 오류 코드

| HTTP 코드 | 의미 |
|-----------|------|
| 200 | 정상 응답 |
| 404 | Citation 없음 또는 권한 없음 (존재 여부 미노출) |
| 422 | `content_hash` 형식 오류 (64자 SHA-256 hex 필요) |

---

## 주의 사항

1. **`citation.version_id`** 는 검색 시점의 버전을 가리킵니다.
   문서가 개정되면 해당 버전의 청크가 달라져 `modified: true`가 반환됩니다.

2. **`content_hash`** 는 `SHA-256(원문.encode("utf-8"))`입니다.
   클라이언트가 직접 계산할 경우 반드시 UTF-8 인코딩을 사용하세요.

3. **ACL** : 요청자가 해당 문서에 접근 권한이 없으면 404가 반환됩니다.
   403이 아닌 404를 반환하여 문서 존재 여부를 외부에 노출하지 않습니다.

4. **S1 `Citation`** (schemas.rag.Citation) 과 **S2 `Citation`** (schemas.citation.Citation) 은
   다른 모델입니다. S1은 RAG 응답용 인덱스 기반 모델이고, S2는 검증 가능한 5-tuple입니다.
