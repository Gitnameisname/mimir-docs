# task4-2 — 신규 read 도구 3종 추가 (FG 4-2)

**작성일**: 2026-04-27
**Phase / FG**: S3 Phase 4 / FG 4-2
**상태**: 착수 대기 (FG 4-1 완료 후)
**담당**: 백엔드 (설계 Claude / 구현 Codex / 리뷰 Claude — extended)
**예상 소요**: 4~5일
**선행 산출물**:
- `task4-0` 완료 (manifest 표준)
- `task4-1` 완료 (응답 envelope 표준)
- `Pre-flight_실측.md`의 REST 엔드포인트 인벤토리 (특히 `documents.render`, 노드 조회)
**후행**: task4-3 (verify_citation 강화) — 본 FG의 search/render 도구가 만들어내는 인용 후보를 검증

---

## 1. 작업 목적

외부 Chatbot이 Mimir 문서를 **노드 단위로 검색하고 정확한 근거 위치를 가리킬 수 있도록** 3개의 read 도구를 추가한다.

| 도구 | 의도 | risk_tier |
|------|------|-----------|
| `search_nodes` | 노드 단위 검색 — 문서가 아니라 노드를 직접 매칭. citation의 후보를 더 좁게 반환 | L0 |
| `read_document_render` | 문서의 렌더링된 텍스트 조회 — TipTap/ProseMirror 트리를 사람이 읽는 텍스트로 변환한 결과 | L0 |
| `resolve_document_reference` | 자연어 참조("그 영업 매뉴얼", "v2024 보안 정책")를 `document_id` + `version_ref`로 정규화. confidence + needs_disambiguation 후보 반환 | L1 |

본 FG는 Phase 4의 **체감 가치가 가장 큰** 작업이다. 외부 Chatbot이 "사용자가 말한 그 문서"를 잘못 추정해 환각을 유발하는 문제를 구조적으로 줄인다. 동시에 환각 위험이 가장 큰 도구이기도 하므로, AI 품질 평가가 의무다.

---

## 2. 작업 범위

### 2.1 포함

#### 2.1.1 `search_nodes` 도구

**의도**: 사용자가 특정 문구·개념을 가진 **노드** (블록·문장 단위)를 직접 찾고 싶을 때. `search_documents`가 문서 매칭이라면 이 도구는 노드 매칭이다.

**입력 스키마**:
```json
{
  "query": "string (1-500)",
  "scope": "string (optional)",
  "document_ids": "array<uuid> (optional, 검색 범위 제한)",
  "node_kinds": "array<string> (optional, 예: ['paragraph','heading','list_item'])",
  "top_k": "integer (1-100, default 20)",
  "access_context": "AccessContext (optional)"
}
```

**출력 스키마**:
```json
{
  "envelope": MCPReadEnvelope,
  "items": [
    {
      "envelope": MCPItemEnvelope,  // FG 4-1 표준
      "document_id": "uuid",
      "version_id": "vN or uuid",
      "node_id": "string",
      "node_kind": "string",
      "snippet": "string (matched span)",
      "score": "float",
      "content_hash": "sha-256 hex"
    }
  ],
  "total_matched": "integer",
  "truncated_at": "integer (top_k cap)"
}
```

**ACL**:
- Scope Profile 필터를 검색 단계에서 적용 (search_documents와 동일 함수 재사용 — `apply_scope_filter`)
- 노드 단위 권한이 별도로 있는 경우(예: 일부 노드만 비공개) Pre-flight에서 확인

**도메인 코어**:
- `app.services.search_service.search_nodes` 신설 (REST `documents.search`와 별도 의도 — 노드 그래뉼래리티)
- 또는 기존 `search_service.search_documents`의 결과를 노드 grain으로 확장하는 함수 추가

#### 2.1.2 `read_document_render` 도구

**의도**: ProseMirror 트리가 아닌 **렌더링된 텍스트**(사람이 읽는 형태)가 필요할 때. 인용의 `citation_basis=rendered_text` 경로(FG 4-3)에서 사용.

**입력 스키마**:
```json
{
  "document_id": "uuid",
  "version_id": "string (vN, uuid, or 'latest' — latest는 자동 resolve, R3)",
  "format": "enum: 'plain_text' | 'markdown' (default plain_text)",
  "include_node_anchors": "boolean (default true) — 결과에 node_id ↔ offset 매핑을 포함",
  "access_context": "AccessContext (optional)"
}
```

**출력 스키마**:
```json
{
  "envelope": MCPReadEnvelope,  // source.uri = mimir://documents/{id}/versions/{vN}/render
  "document_id": "uuid",
  "version_id": "vN (resolved)",
  "format": "plain_text | markdown",
  "rendered_text": "string",
  "render_hash": "sha-256 hex",
  "node_anchors": [
    {"node_id": "string", "offset_start": "int", "offset_end": "int"}
  ]
}
```

**도메인 코어**:
- 기존 `app.services.render_service.render_version` 재사용 (REST `/documents/{id}/render` 백엔드)
- `node_anchors`는 Phase 1의 stable node_id 위에서 빌드 — render_service가 anchor 정보를 제공하지 않으면 `render_service`에 신규 함수 추가
- `render_hash`는 rendered_text 전체의 SHA-256

**R3 (pinned) 보장**:
- `version_id="latest"` 입력 시 서버가 즉시 vN으로 resolve하고, envelope의 `source.uri`에 vN을 포함
- 응답 본문의 `version_id` 필드도 resolved 값

#### 2.1.3 `resolve_document_reference` 도구

**의도**: "그 영업 매뉴얼", "v2024 보안 정책 문서" 같은 자연어 참조를 `document_id` + `version_ref`로 정규화.

**입력 스키마**:
```json
{
  "reference": "string (1-500)",
  "scope": "string (optional)",
  "preferred_doc_types": "array<string> (optional)",
  "context": {
    "recent_document_ids": "array<uuid> (optional, 사용자의 최근 문서)",
    "conversation_id": "uuid (optional)"
  },
  "max_candidates": "integer (1-10, default 5)",
  "confidence_threshold": "float (0..1, default 0.85)",
  "access_context": "AccessContext (optional)"
}
```

**출력 스키마**:
```json
{
  "envelope": MCPReadEnvelope,
  "resolved": "boolean",
  "needs_disambiguation": "boolean",
  "best_match": {
    "document_id": "uuid",
    "version_ref": "vN or 'latest_published'",
    "title": "string",
    "confidence": "float",
    "match_kind": "exact_title | alias | semantic | recent_context"
  } | null,
  "candidates": [
    {
      "document_id": "uuid",
      "version_ref": "vN or string",
      "title": "string",
      "confidence": "float",
      "match_kind": "string",
      "envelope": MCPItemEnvelope
    }
  ]
}
```

**해소 로직 (단계적)**:
1. **정확 매칭**: 제목 정확 일치 → confidence 0.99, match_kind=exact_title
2. **별칭 매칭**: 메타데이터(별칭/이전 제목) 일치 → confidence 0.95, match_kind=alias
3. **최근 컨텍스트**: `context.recent_document_ids` 안에서 부분 일치 → confidence 0.85, match_kind=recent_context
4. **의미 매칭(semantic)**: 벡터 검색으로 top-K → 코사인 점수 정규화 → match_kind=semantic
5. 임계값(`confidence_threshold`) 미달 시 `needs_disambiguation=true`, `best_match=null`, `candidates`에 후보 반환
6. **ACL은 모든 단계에 적용** — Scope Profile 통과한 문서만 후보로

**폐쇄망 동등성 (S2 ⑦)**:
- `MIMIR_OFFLINE=1` 환경에서는 단계 4(semantic)를 **FTS fallback**으로 대체
- FTS fallback도 confidence를 산출하지만 임계값을 더 보수적으로 (예: 0.90) 적용
- 폐쇄망 회귀 테스트 의무

#### 2.1.4 도구 등록 (FG 4-0 manifest 표준 준수)

`backend/app/schemas/mcp.py`의 `TOOL_SCHEMAS`에 3개 항목 추가:

```python
{
    "name": "search_nodes",
    "risk_tier": "L0",
    "maturity": "beta",
    "status": "enabled",
    "exposure_policy": "MCP_ENABLED",
    ...
},
{
    "name": "read_document_render",
    "risk_tier": "L0",
    "maturity": "beta",
    "status": "enabled",
    "exposure_policy": "MCP_ENABLED",
    ...
},
{
    "name": "resolve_document_reference",
    "risk_tier": "L1",
    "maturity": "experimental",  # confidence 정밀도 검증 후 beta
    "status": "enabled",
    "exposure_policy": "MCP_ENABLED",
    ...
},
```

`backend/app/mcp/tools.py`에 `tool_search_nodes`, `tool_read_document_render`, `tool_resolve_document_reference` 함수 추가.

`backend/app/api/v1/mcp_router.py`의 `tools/call` 디스패처에 분기 추가.

#### 2.1.5 AI 품질 평가 (resolve 정밀도)

`backend/scripts/rag_smoke/golden_resolve.py` 신설 — 50건 이상의 자연어 참조 골든셋:
- 정확 매칭 케이스: 20건
- 별칭 매칭 케이스: 10건
- 의미 매칭 케이스 (모호): 15건
- 명시적 disambiguation 필요 케이스: 5건+

지표:
- **Precision@1** (best_match가 정답일 확률) ≥ 0.85
- **Recall@5** (정답이 candidates 안에 포함될 확률) ≥ 0.95
- **Disambiguation triggered correctly** (모호 케이스에서 needs_disambiguation=true) ≥ 0.80

`산출물/FG4-2_AI품질평가보고서.md`에 결과 첨부.

### 2.2 제외 (이월)

- LLM 기반 의미 매칭 (단순 벡터 검색 외) — 별도 Phase
- resolve 결과를 사용자에게 disambiguation UI로 표시 — 클라이언트 영역
- search_nodes에 grouped result (문서별 그룹화) — 향후 검토
- read_document_render의 PDF/DOCX 형식 — 본 FG는 plain_text/markdown만

### 2.3 하드코딩 금지 재확인

- confidence 임계값(0.85)은 입력 파라미터 + 환경변수(`MCP_RESOLVE_CONFIDENCE_THRESHOLD`)로 외부화. 코드 인라인 금지.
- node_kinds 화이트리스트는 `app.constants.node_kinds`에서 import. tool에서 인라인 금지.
- mimir:// URI 빌드는 FG 4-1의 `uri_builder` 단일 모듈 경유.

---

## 3. 선행 조건

- FG 4-0 / 4-1 완료
- Phase 1의 stable node_id 회귀 녹색 (read_document_render의 node_anchors가 의존)
- `render_service.render_version`이 stable (S2 종결 자산)
- `search_service`의 ACL 필터 함수 안정

---

## 4. 구현 단계

### Step 1 — search_nodes 도메인 코어

1. `app/services/search_service.py`에 `search_nodes(conn, query, actor, top_k, document_ids, node_kinds)` 신설
2. 기존 vector/FTS retriever 재사용 — 노드 grain으로 좁힘
3. ACL 필터 동일 적용
4. 단위 테스트: 결과 grain이 노드인지 / score 정렬 / top_k cap

### Step 2 — search_nodes MCP 도구

1. `app/mcp/tools.py`에 `tool_search_nodes` 추가
2. envelope 채우기 (FG 4-1 표준)
3. 항목별 envelope에 `source.uri = mimir://documents/{id}/versions/{vN}/nodes/{node_id}`
4. 회귀 테스트: ACL 통과 / envelope 표준 / truncated_at 동작

### Step 3 — read_document_render 도구

1. `render_service`에 `node_anchors` 출력 추가 (없으면)
2. `tool_read_document_render` 추가
3. `latest` resolve + `render_hash` 계산
4. envelope에 `source.uri = mimir://.../render`
5. 회귀: latest → vN resolve / node_anchors 정확성 / 폐쇄망 모드 동작

### Step 4 — resolve_document_reference 도메인 코어

1. `app/services/document_resolver_service.py` 신설 (REST 측에서 재사용 가능하도록)
2. 5단계 해소 로직 구현 (§2.1.3)
3. 각 단계에 ACL 적용
4. 폐쇄망 분기 (`MIMIR_OFFLINE=1`)
5. 단위 테스트: 단계별 confidence 산출 / 임계값 동작 / disambiguation 트리거

### Step 5 — resolve_document_reference MCP 도구

1. `tool_resolve_document_reference` 추가
2. envelope + 항목별 envelope 채움
3. `version_ref`는 `vN` 또는 `latest_published`. **`latest` 단독 문자열은 반환 금지** (R3 — 인용으로 흘러갈 가능성)
4. 회귀: 5 단계 케이스 / 임계값 미달 → disambiguation / ACL 누락 시 후보 비공개

### Step 6 — manifest 등록 + 디스패처 분기

1. `TOOL_SCHEMAS`에 3 항목 추가 (FG 4-0 manifest 형식 준수)
2. `mcp_router.py`의 `tools/call` 분기 추가
3. `tools/list` 응답에서 3 도구가 노출됨을 회귀

### Step 7 — AI 품질 평가

1. `scripts/rag_smoke/golden_resolve.py` 작성 (50+ 케이스)
2. 평가 실행 → JSON 결과 + `FG4-2_AI품질평가보고서.md`
3. 임계값 미달 시 confidence_threshold 또는 후보 알고리즘 조정 후 재실행

### Step 8 — 검수 / 보안 보고서

- `FG4-2_검수보고서.md` — R1~R5 준수 + R3 사전 보장(latest 미반환) 확인 + 도메인 코어 함수-호출 그래프(R5)
- `FG4-2_보안취약점검사보고서.md` — query/reference 입력의 prompt injection 페이로드 5+종 결과 + ACL 우회 시도 결과
- `FG4-2_AI품질평가보고서.md` — 위 §2.1.5 지표 + 실측 JSON

---

## 5. API 계약 변경 요약

| 메서드 | 도구 / 경로 | 변경 |
|-------|-------------|------|
| POST | /mcp/tools/call (search_nodes) | 신규 도구 |
| POST | /mcp/tools/call (read_document_render) | 신규 도구 |
| POST | /mcp/tools/call (resolve_document_reference) | 신규 도구 |

기존 `tools/call` 응답 envelope은 FG 4-1 표준을 그대로 사용.

---

## 6. 데이터 모델 주의사항

- DB 스키마 변경 없음 (citation_basis는 FG 4-3)
- 기존 `nodes` 테이블의 `node_id`/`kind`/`content_hash` 필드 활용
- `versions.content_hash`는 ProseMirror 트리 hash. `read_document_render`가 내는 `render_hash`는 별도(렌더 텍스트 기준) — FG 4-3에서 어느 hash로 검증할지 분기

---

## 7. 성공 기준

- [ ] 3개 도구 manifest 등록 완료 (`risk_tier`/`maturity`/`status` 부착)
- [ ] `tools/list`에 3개 도구 노출 + `tools/call`로 호출 가능
- [ ] 모든 응답이 FG 4-1 envelope 표준 통과 (R1~R5)
- [ ] **R3 사전 보장**: 어느 도구도 응답에 단독 `latest` 문자열을 반환하지 않음
- [ ] Scope Profile ACL 회귀: 다른 scope의 사용자는 결과를 받지 못함
- [ ] 폐쇄망 모드(`MIMIR_OFFLINE=1`)에서 resolve 도구가 FTS fallback으로 동작
- [ ] AI 품질 평가: Precision@1 ≥ 0.85 / Recall@5 ≥ 0.95 / Disambiguation correctness ≥ 0.80
- [ ] pytest 신규 ≥ 40 / 전체 베이스라인 유지
- [ ] 검수·보안·AI 품질 보고서 제출

---

## 8. 리스크

| 리스크 | 대응 |
|-------|-----|
| resolve가 잘못된 문서를 high confidence로 반환 → 환각 | 임계값 + AI 품질 평가 의무. Precision@1 미달 시 release 차단 |
| search_nodes의 결과량 폭발 | top_k 상한 100 + truncated_at 응답 + envelope.items_truncated |
| read_document_render의 render_hash가 비결정적(공백·줄바꿈 변동) | render 출력 normalization 함수 단일화 + 결정성 회귀 테스트 |
| 폐쇄망 fallback이 동작하지 않음 | `MIMIR_OFFLINE=1` 회귀 의무 + CI에서 양쪽 모드 모두 실행 |
| ACL 우회 (resolve 후보가 ACL을 거치지 않음) | 후보 생성 모든 단계에서 `apply_scope_filter` 호출 단위 테스트 |
| 신규 도구 함수가 REST 도메인 코어를 우회 (R5 위배) | 검수보고서 §"R5 준수 확인"에 함수-호출 그래프 첨부 의무 |
| latest resolve의 race condition | resolve 트랜잭션 격리 회귀 + 본 FG 도구도 FG 4-1 표준 사용 |

---

## 9. 참조

- `Phase 4 개발계획서.md` §1.2 (R1~R5), §2.1 (FG 4-2)
- `uploads/Mimir API : MCP 운영 논의 요약` §4 (resolve_document_reference), §7.1 (Phase 1A)
- `task4-1.md` (envelope 표준 + mimir:// URI)
- `CONSTITUTION.md` 제5·17·18조
- `backend/app/services/render_service.py`, `backend/app/services/search_service.py`
- `backend/app/api/v1/documents.py` (REST `/documents/{id}/render`)
- `scripts/rag_smoke/` (AI 품질 평가 베이스라인 — Phase 0 FG 0-4 자산)

---

*작성: 2026-04-27 | FG 4-2 — 신규 read 도구 3종*
