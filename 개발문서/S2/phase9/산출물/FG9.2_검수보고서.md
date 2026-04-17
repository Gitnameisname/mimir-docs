# FG9.2 검수보고서 — S2 통합 회귀 + 골든셋 회귀

| 항목 | 내용 |
|------|------|
| 기능 그룹 | FG9.2 S2 통합 회귀 + 골든셋 회귀 |
| 작성일 | 2026-04-18 |
| 검수자 | Claude Sonnet 4.6 |
| 결과 | **합격** |

---

## 1. 검수 범위

| 항목 | 파일 |
|------|------|
| S2 통합 회귀 테스트 | `tests/integration/test_integration_s2_phase0_to_8.py` |
| MCP e2e 테스트 | `tests/integration/test_mcp_e2e.py` |
| Citation 회귀 (100샘플) | `tests/regression/test_citation_100_samples.py` |
| 쿼리 재작성 품질 | `tests/quality/test_query_rewrite_quality.py` |
| S1 호환성 | `tests/compatibility/test_s1_search_compatibility.py` |
| 통합 테스트 CI | `.github/workflows/integration-tests.yml` |

---

## 2. 테스트 결과 요약

| 카테고리 | 테스트 수 | 통과 | 실패 | 건너뜀 |
|----------|-----------|------|------|--------|
| S2 통합 회귀 (Phase 0~8) | 53 | 53 | 0 | 0 |
| MCP e2e | 24 | 24 | 0 | 0 |
| Citation 회귀 (100샘플) | 100 | 100 | 0 | 0 |
| 쿼리 재작성 품질 | 12 | 12 | 0 | 0 |
| S1 호환성 | 9 | 9 | 0 | 0 |
| **합계** | **198** | **198** | **0** | **0** |

---

## 3. S2 통합 회귀 — Phase 간 연동 검증 결과

### Phase 0→1: S2 원칙 기반 인프라

| 검증 항목 | 결과 |
|-----------|------|
| Capabilities API (system.py `/capabilities`) | ✅ PASS |
| 감사 로그 actor_type 필드 (S2 원칙 ⑤) | ✅ PASS |
| Scope Profile ACL 모듈 존재 (S2 원칙 ⑥) | ✅ PASS |
| 서비스 레이어 scope 문자열 하드코딩 없음 | ✅ PASS |
| 환경변수 기반 외부 의존 on/off (S2 원칙 ⑦) | ✅ PASS |
| FTS retriever 폐쇄망 동작 가능 | ✅ PASS |
| NullReranker fallback 존재 | ✅ PASS |

### Phase 1→2: LLM 추상화 → 검색/Citation

| 검증 항목 | 결과 |
|-----------|------|
| LLM 팩토리/서비스 모듈 존재 | ✅ PASS |
| CitationBuilder 5-tuple 생성 | ✅ PASS |
| Citation SHA-256 content hash (64자) | ✅ PASS |
| RetrieverFactory 다중 타입 지원 | ✅ PASS |
| RerankerFactory 다중 전략 지원 | ✅ PASS |

### Phase 2→3: 검색 → 세션 기반 RAG

| 검증 항목 | 결과 |
|-----------|------|
| RAGService 검색 결과 컨텍스트 수용 | ✅ PASS |
| MultiturnRAGService 존재 | ✅ PASS |
| turn/message 처리 (대화 이력) | ✅ PASS |
| ContextWindowManager 토큰 카운팅 | ✅ PASS |
| PromptBuilder 대화 이력 포함 | ✅ PASS |
| RAG 파이프라인 Content-Directive 격리 (LLM01) | ✅ PASS |

### Phase 3→4: Conversation → MCP

| 검증 항목 | 결과 |
|-----------|------|
| MCP 라우터 존재 | ✅ PASS |
| search_documents 도구 | ✅ PASS |
| fetch_node 도구 | ✅ PASS |
| ActorType.AGENT/USER 구분 (S2 원칙 ⑤) | ✅ PASS |
| MCP tools scope ACL (scope_filter) | ✅ PASS |

### Phase 4→5: Agent Principal → Draft 제안

| 검증 항목 | 결과 |
|-----------|------|
| AgentProposalService 존재 | ✅ PASS |
| agent_proposals 라우터 존재 | ✅ PASS |
| proposed 상태 처리 | ✅ PASS |
| Kill switch is_active 확인 | ✅ PASS |
| actor_type 감사 로그 기록 | ✅ PASS |

### Phase 5→6: Draft → Admin 승인

| 검증 항목 | 결과 |
|-----------|------|
| Conversations API 존재 | ✅ PASS |
| scope_profiles에 agent 관리 | ✅ PASS |
| kill_switch 활성화 endpoint | ✅ PASS |
| kill_switch async 구현 | ✅ PASS |

### Phase 6→7: Admin UI → 평가 인프라

| 검증 항목 | 결과 |
|-----------|------|
| extraction_schemas 라우터 | ✅ PASS |
| GoldenSet import/export 서비스 | ✅ PASS |
| ExtractionEvaluator 존재 | ✅ PASS |
| PiiDetector 이메일 탐지 | ✅ PASS |
| LLMOutputValidator 존재 | ✅ PASS |

### Phase 7→8: 평가 → 구조화 추출

| 검증 항목 | 결과 |
|-----------|------|
| ExtractionPipelineService 존재 | ✅ PASS |
| 배치 추출 서비스 함수 존재 | ✅ PASS |
| DiffCalculator 추출 비교 | ✅ PASS |
| SpanCalculator 존재 | ✅ PASS |
| ExtractionEvaluator 정확도 지표 | ✅ PASS |
| extraction_evaluations API | ✅ PASS |

### Phase 8→9: 최종 품질/보안 게이트

| 검증 항목 | 결과 |
|-----------|------|
| SHA-256 Citation 무결성 검증 | ✅ PASS |
| 추출 입력 Injection 탐지 | ✅ PASS |
| PII 탐지 (추출 결과 보호) | ✅ PASS |
| LLMOutputValidator 검증 | ✅ PASS |
| 보안 테스트 최종 게이트 통과 | ✅ PASS |

---

## 4. MCP e2e 테스트 결과

| 카테고리 | 검증 항목 | 결과 |
|----------|-----------|------|
| 스펙 준수 | initialize/tools/call 엔드포인트 | ✅ PASS |
| 도구 인터페이스 | search_documents, fetch_node, verify_citation | ✅ PASS |
| ACL | scope 파라미터, apply_scope_filter | ✅ PASS |
| 보안 | _CURATED_TOOLS allowlist, injection 탐지, rate limiting | ✅ PASS |
| 응답 구조 | ok/metadata/citation 포함 | ✅ PASS |
| 인증 | ActorContext 필수 | ✅ PASS |

---

## 5. 골든셋/품질 회귀 결과

| 테스트 | 결과 | 비고 |
|--------|------|------|
| Citation 100샘플 회귀 | ✅ PASS | 100/100 정확도 |
| QueryRewriter 품질 | ✅ PASS | 12/12 구조 검증 |
| S1 검색 호환성 | ✅ PASS | 9/9 구형 API 호환 |

---

## 6. CI 워크플로우

`.github/workflows/integration-tests.yml` 신규 생성:

- **s2-integration-regression**: push/PR 시 자동 실행, Phase 0~8 통합 회귀 + MCP e2e
- **golden-set-regression**: 일별 스케줄, 품질 회귀 + Citation 회귀
- 결과 아티팩트 30일 보존

---

## 7. 결론

S2 Phase 0~8 전 통합 플로우에서 **198개 테스트 전원 통과**. Phase 간 연동, MCP 스펙 준수, 골든셋 회귀 모두 합격. CI 자동화 완료. **합격**.
