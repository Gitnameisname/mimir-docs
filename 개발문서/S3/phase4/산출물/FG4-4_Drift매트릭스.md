# FG 4-4 Contract Drift 매트릭스

**작성일**: 2026-04-28
**FG**: S3 Phase 4 / FG 4-4
**작성자**: Claude

본 매트릭스는 4종 contract drift 테스트 + manifest drift 게이트의 통과/실패 추적용. CI 등록 후
매번 갱신.

---

## 1. 통과 매트릭스 (2026-04-28 로컬 실행 기준)

| # | 테스트 | 시나리오 | 통과 | 마지막 실행 | CI run |
|---|---|---|---|---|---|
| 1 | 동일 노드 결과 일치성 | 5 | ✅ | 2026-04-28 | (CI 미등록) |
| 2 | latest 해석 동일성 | 7 | ✅ | 2026-04-28 | (CI 미등록) |
| 3 | 비노출 도구 (R1) | 11 | ✅ | 2026-04-28 | (CI 미등록) |
| 4 | pinned citation (R3) | 8 | ✅ | 2026-04-28 | (CI 미등록) |
| 5 | manifest drift 게이트 | 6 | ✅ | 2026-04-28 | (CI 미등록) |
| **합계** | — | **37** | **37/37 ✅** | | |

---

## 2. 시나리오 분포 (R1/R3/R5 매핑)

### R1 (L4 차단)

| 시나리오 | 위치 |
|---|---|
| TOOL_SCHEMAS L4 0건 | test_not_exposed_tools.py::test_no_l4_tools |
| TOOL_SCHEMAS forbidden 0건 | test_not_exposed_tools.py::test_no_forbidden_tools |
| TOOL_SCHEMAS not_exposed 0건 | test_not_exposed_tools.py::test_no_not_exposed_tools |
| L4 함의 검증 (정의 0개여도 통과) | test_not_exposed_tools.py::test_l4_implies_not_exposed_and_forbidden |
| 가짜 L4 거부 | test_not_exposed_tools.py::test_is_tool_mcp_exposed_blocks_fake_l4 |
| forbidden 단독 거부 | test_not_exposed_tools.py::test_fake_forbidden_blocked_even_if_status_enabled |
| not_exposed 단독 거부 | test_not_exposed_tools.py::test_fake_not_exposed_blocked_even_if_l0 |
| 임시 주입 → curated 제외 | test_not_exposed_tools.py::test_curated_tools_excludes_filtered_after_fake_injection |
| REST 관리자 도구 부재 | test_not_exposed_tools.py::test_rest_admin_tools_absent |
| dispatcher unknown 거부 | test_not_exposed_tools.py::test_dispatch_unknown_tool_path_returns_invalid_request |
| _CURATED_TOOLS 정합 | test_not_exposed_tools.py::test_curated_tools_matches_exposed_schemas |
| manifest 의 L4 0건 | test_manifest_drift.py::test_manifest_no_l4_tools |
| manifest 의 is_mcp_exposed=True | test_manifest_drift.py::test_manifest_all_mcp_exposed_true |

### R3 (pinned citation)

| 시나리오 | 위치 |
|---|---|
| latest 즉시 거부 (verify_citation) | test_pinned_citation.py::test_latest_input_immediate_400 |
| draft pinned=False | test_pinned_citation.py::test_draft_returns_pinned_false |
| citation_basis default = node_content | test_pinned_citation.py::test_request_default_node_content / test_citation_model_default_node_content |
| _Candidate.version_ref = latest_published | test_pinned_citation.py::test_resolve_candidate_default_latest_published |
| ReadDocumentRenderData.version_id resolved | test_pinned_citation.py::test_render_response_version_id_resolved |
| URI 빌더 'latest' 거부 | test_latest_resolution.py::test_uri_builder_rejects_latest |
| Disagreement Record 존재 | test_pinned_citation.py::test_disagreement_record_exists |
| citations 테이블 부재 (Disagreement) | test_pinned_citation.py::test_no_citations_table_in_codebase |

### R5 (REST 1:1 복제 금지)

| 시나리오 | 위치 |
|---|---|
| 정본 source_text 공유 | test_same_node_result.py::test_mcp_and_rest_share_chunk_source_text_as_canonical |
| MCP/REST hash 일치 | test_same_node_result.py::test_canonical_content_hash_consistency |
| ApiNotFoundError → MCPError(NOT_FOUND) | test_same_node_result.py::test_mcp_rejects_when_document_not_allowed |
| ACL 모듈 공유 | test_same_node_result.py::test_acl_filter_function_shared_between_rest_and_mcp |
| projection 차이 허용 (envelope) | test_same_node_result.py::test_mcp_response_includes_envelope |
| VersionsRepository 정본 공유 | test_latest_resolution.py::test_mcp_uri_builder_resolve_uses_versions_repository |
| REST resolve 동일 모듈 | test_latest_resolution.py::test_rest_resolve_uses_same_versions_repository |
| 결정성 (같은 시점 같은 vN) | test_latest_resolution.py::test_resolve_version_id_returns_same_concrete |
| 구체 vN 통과 | test_latest_resolution.py::test_concrete_version_passthrough |
| published 부재 시 None | test_latest_resolution.py::test_no_published_returns_none |

---

## 3. 누적 회귀

| 카테고리 | 케이스 |
|---|---|
| 본 FG 신규 (FG 4-4) | 37 |
| 누적 MCP (4-0/4-1/4-2/4-3 + e2e + 본 FG) | **213 / 213 ✅** |
| 전체 회귀 | **3495 / 0 fail** |

---

## 4. CI 등록 후 갱신 절차

운영자가 CI 게이트 등록 후 본 매트릭스의 §1 마지막 컬럼 (`CI run`) 에 URL 추가. 매 main pipeline 후 갱신:

```bash
# CI 통과 시
git diff --no-index <(echo "✅") <(echo "✅")  # nop
# 실패 시 — 매트릭스에 ❌ + 원인 메모
```

---

## 5. 변경 이력

| 일자 | 변경 | 작성자 |
|---|---|---|
| 2026-04-28 | 초판 — 37 시나리오 R1/R3/R5 매핑. CI 미등록 — 운영자 후속. | Claude |
