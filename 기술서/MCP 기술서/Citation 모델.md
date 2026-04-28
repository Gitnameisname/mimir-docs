# MCP Citation 모델

5-tuple 좌표 + citation_basis + 5중 검증 — 인용의 시간 후 변조 감지.

**선행 문서**: [MCP 표면 개요.md](MCP%20표면%20개요.md)
**소스 정본**: `backend/app/schemas/citation.py::Citation`
**Disagreement Record**: [docs/disagreements/2026-04-28-fg43-citations-table-absence.md](../../disagreements/2026-04-28-fg43-citations-table-absence.md) — citations 테이블 부재로 작업지시서 §2.1.1 의 Alembic 마이그레이션 미적용. citation_basis 는 Pydantic 모델 + JSONB 임베드 표면.

---

## 1. Citation 5-tuple

```python
class Citation(BaseModel):
    document_id: UUID
    version_id: UUID
    node_id: UUID
    span_offset: Optional[int]      # 청크 내 문자 오프셋
    content_hash: str               # SHA-256(정본 텍스트) — 64 hex
    citation_basis: Literal["node_content", "rendered_text"] = "node_content"
```

**5-tuple 의 의미**:
- `(document_id, version_id, node_id, span_offset)` — 시간 불변 좌표 (R3: pinned vN)
- `content_hash` — 좌표가 가리키는 텍스트의 SHA-256 — 시간 후 재검증 가능

---

## 2. citation_basis (FG 4-3)

| 값 | 정본 텍스트 | hash 대상 | 사용 시점 |
|---|---|---|---|
| `node_content` (default) | `document_chunks.source_text` | SHA-256(source_text) | 청크 단위 인용 — S2 이전 동작 그대로 |
| `rendered_text` | `RenderService.render_version` + `_walk_blocks_for_text(plain_text)` | SHA-256(rendered_text) | FG 4-2 `read_document_render` 출력 기반 인용 |

---

## 3. mimir:// URI ↔ Citation

Citation 의 좌표는 mimir:// URI 로도 표현 가능:

| Citation 필드 | URI |
|---|---|
| 노드 단위 인용 | `mimir://documents/{document_id}/versions/{version_id}/nodes/{node_id}` |
| 렌더 텍스트 인용 | `mimir://documents/{document_id}/versions/{version_id}/render` |

응답 envelope 의 `source.uri` 가 본 URI — 빌더 정본은 `app.mcp.uri_builder`.

**R3 강제**: URI 어디에도 `latest` 단독 미출현. 빌더가 입력 시점에 `ValueError` 거부.

---

## 4. verify_citation 5중 검증 (FG 4-3)

`verify_citation` 도구가 다음 5 검사 모두 수행:

| # | 검사 | 입력 | 통과 조건 |
|---|---|---|---|
| 1 | **exists** | `document_id`, `version_id`, `node_id` | 모두 존재 |
| 2 | **pinned** | `version_id` | `"latest"` 즉시 거부 (도구 진입점) + `versions.status` ∈ `{published, archived}` (draft 거부) |
| 3 | **hash_matches** | `content_hash`, `citation_basis` | citation_basis 별 분기로 정본 hash 와 비교 |
| 4 | **quoted_text_in_content** | `quoted_text` (선택) | 입력 시 정본 텍스트에 포함 / 미입력 시 None (skip) |
| 5 | **span_valid** | `span_offset`, `span_length` (선택) | 둘 다 입력 + offset≥0 + length>0 + offset+length≤len(text) / 미입력 시 None |

종합:
```
verified = exists AND pinned AND hash_matches
        AND (quoted_in is not False)   # None 시 통과
        AND (span_valid is not False)  # None 시 통과
```

**응답 (`VerifyCitationData`)**:
```json
{
  "verified": true,
  "checks": {
    "exists": true, "pinned": true, "hash_matches": true,
    "quoted_text_in_content": true, "span_valid": true
  },
  "current_hash": "...",
  "rendered_snapshot": null,
  "node_snapshot": "...",
  "message": "검증 성공."
}
```

---

## 5. R3 강제 (pinned citation)

세 곳에서 `latest` 차단:

1. **URI 빌더** (`build_version_uri` / `build_node_uri` / `build_render_uri`)
   - `version_id="latest"` 입력 → `ValueError`
2. **`verify_citation` 도구 진입점**
   - `version_id="latest"` 입력 → `MCPError(INVALID_REQUEST, 400)` 즉시 거부
3. **`resolve_document_reference` 후보**
   - `_Candidate.version_ref` default = `"latest_published"` (단독 `"latest"` 절대 미반환)

Citation 응답의 모든 `version_id` / `version_ref` 는 구체값 (vN 또는 UUID 또는 `"latest_published"`).

---

## 6. citations 테이블 부재 — Disagreement Record 적응

`task4-3.md` §2.1.1 은 `ALTER TABLE citations ADD COLUMN citation_basis` 를 가정했으나, **Mimir 에는 `citations` 테이블이 존재하지 않음** (Disagreement Record).

**Mimir 의 실제 citation 모델**:
1. **Pydantic 모델** (`app.schemas.citation.Citation`) — in-memory 5-tuple
2. **JSONB 임베드** — 다른 테이블의 컬럼 (예: `golden_set_items.expected_citations`, `evaluation_runs.citations`) 에 직렬화
3. **검증 정본** (`document_chunks.source_text`) — 시간 후 재검증의 hash 계산 대상

따라서 citation_basis 는:
- ✅ Pydantic Citation 모델 필드 (Literal 강제)
- ✅ JSONB 임베드 키 (운영 정착 시 별 라운드 — 현재 일부 위치만)
- ❌ DB 컬럼 (citations 테이블 자체 부재)

---

## 7. 인용 라이프사이클

```
[검색]
  search_documents / search_nodes
    → 결과에 citation 포함 (content_hash 채움)
        ↓
[인용 저장]
  외부 클라이언트 (예: golden_set / conversation answer) 에서 citation 5-tuple JSONB 로 보존
        ↓
[시간 흐름] — 문서 변경, 새 vN 발행 등
        ↓
[재검증]
  verify_citation(document_id, version_id, node_id, content_hash, citation_basis, ...)
    → 5중 검사 통과 시 verified=True
    → 미통과 시 checks 딕셔너리로 어느 검사가 실패했는지 디버깅 가시성
```

**핵심**: 인용 저장 시점의 vN 이 시간이 지나도 그대로 보존되어야 검증 가능 (R3). 저장 시점에 `latest` 면 후일 다른 vN 의 본문이 정본이 되어 검증 깨짐.

---

## 8. 백워드 호환

`VerifyCitationData` 의 호환 필드 (FG 4-3):

| 필드 | 의미 | FG 4-3 권장 |
|---|---|---|
| `content_snapshot` | 호환 — `node_snapshot` 또는 `rendered_snapshot` 의 alias | 신규 클라이언트는 `node_snapshot` / `rendered_snapshot` 사용 |
| `hash_matches` | 호환 — `checks.hash_matches` 의 alias | 신규는 `checks` 사용 |
| `version_valid` | 호환 — `checks.exists` 의 alias | 신규는 `checks` 사용 |

호환 기간 관찰 후 별 라운드에서 제거.

---

## 9. 변경 이력

| 일자 | 변경 |
|---|---|
| 2026-04-28 | 초판 — Citation 5-tuple + citation_basis + 5중 검증 + R3 강제 + Disagreement Record |
