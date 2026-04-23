# RAG 스모크 스크립트 검수보고서

| 항목 | 값 |
|------|----|
| 검수일 | 2026-04-22 |
| 대상 | `scripts/rag_smoke/` (rag_smoke.py, sample_documents.py, golden_queries.py) |
| 검수자 | Claude Opus 4.7 (Mimir S2 종결 후 RAG 스모크 준비 세션) |
| 관련 계획 | `docs/개발문서/S2_5/RAG_스모크_실행가이드.md` |

---

## 1. 검수 범위

본 스모크는 Mimir S2 종결 직후 실제 인프라(PostgreSQL + Valkey + Milvus + SBERT + FastAPI)에서 RAG 파이프라인이 end-to-end 로 흐르는지 확인하기 위한 1차 기능 스모크다. 단위 테스트 479건이 통과했음에도 S2 종결 테스트에서 BUG-01~05(트랜잭션 오염, 차원 불일치 등)가 드러난 바 있어, 실 인프라 연동 검증이 필수다.

검수 대상 산출물:

- `scripts/rag_smoke/sample_documents.py` — 16개 샘플 문서 (4 theme × POLICY/MANUAL/REPORT 혼합)
- `scripts/rag_smoke/golden_queries.py` — 21개 골든 쿼리 (simple 8 / ambiguous 4 / typed 3 / multihop 3 / negative 3)
- `scripts/rag_smoke/rag_smoke.py` — 러너 (표준 라이브러리만 사용, 외부 pip 의존성 0)
- `docs/개발문서/S2_5/RAG_스모크_실행가이드.md`

---

## 2. 검수 체크리스트

### 2.1 스크립트 구조

| 항목 | 결과 | 근거 |
|------|------|------|
| py_compile 통과 | ✅ | `python3 -m py_compile rag_smoke.py sample_documents.py golden_queries.py` → OK |
| import 사이클/순환 없음 | ✅ | rag_smoke.py 만 타 모듈을 import, 타 모듈 간 상호 의존 없음 |
| 외부 의존성 없음 | ✅ | `urllib.request/error`, `json`, `dataclasses` 등 stdlib only. pip install 불필요 |
| 엔트리포인트 명시 | ✅ | `if __name__ == "__main__": sys.exit(main())` |
| argparse 일관성 | ✅ | `--base-url / --admin-email / --admin-password / --wait-vectorize / --rag-sleep / --top-k / --dry-run / --skip-seed / --report-out` |

### 2.2 샘플 문서 정합성

| 항목 | 결과 | 근거 |
|------|------|------|
| 최소 15건 이상 (사용자 요구) | ✅ | 16건 |
| DocumentType 다양성 | ✅ | POLICY 5 / MANUAL 6 / REPORT 5 |
| 주제 다양성 (재랭커 효과 관찰용) | ✅ | 4 theme (정보보안 / 인사복무 / 개발운영 / 재무) |
| `document_type` 대문자 일치 | ✅ | P7-2 대소문자 정규화와 정합 |
| content_snapshot 루트 | ✅ | `{"type":"document","content":[...]}` — `DraftSaveRequest.content_snapshot` 계약 부합 |
| 노드 스키마 | ✅ | top-level heading + paragraph 노드, `vectorization_service._parse_nodes_from_snapshot` 파싱 가능 |
| metadata 필수 필드 | ✅ | title/author/effective_date/version 등 포함 |

### 2.3 골든 쿼리 정합성

| 항목 | 결과 | 근거 |
|------|------|------|
| `expected_doc_ids` ⊆ SAMPLE_DOCUMENTS.id | ✅ | 교차 검증 스크립트로 missing=[] 확인 |
| 카테고리 커버리지 | ✅ | simple 8 / ambiguous 4 / typed 3 / multihop 3 / negative 3 |
| typed 카테고리는 `document_type` 지정 | ✅ | Q13=MANUAL, Q14=POLICY, Q15=REPORT |
| negative 카테고리는 `expect_no_answer=True` | ✅ | Q19/Q20/Q21 |
| multihop 은 `min_citations_from_doc` 지정 | ✅ | Q16/Q17/Q18 모두 2건 이상 |
| expected_keywords 는 실제 문서 본문에 존재 | ✅ (수정 후) | 정적 검증 과정에서 Q05("화요일"/"목요일") · Q15("18420"/"18,420") 2건이 원문과 표기 불일치로 검출되어 "화·목"/"배포 윈도우" · "9,600만원"으로 교정 완료. 18건 positive 쿼리 전수에서 keyword gap 0건 |

### 2.4 백엔드 API 계약 부합

| 호출 | 본 스크립트 경로·바디 | 서버 정의 위치 | 결과 |
|------|----------------------|---------------|------|
| 로그인 | `POST /api/v1/auth/login` `{identifier, password}` | `auth_router.py:129 LoginRequest` — `identifier` 필수 | ✅ |
| 문서 생성 | `POST /api/v1/documents` `{title, document_type, summary?, metadata?}` | `documents.py:120` + `DocumentCreateRequest` (summary Optional 존재) | ✅ |
| Draft 저장 | `PUT /api/v1/documents/{id}/draft` `{title?, summary?, content_snapshot}` | `documents.py:450 save_draft`, `schemas/versions.py:89 DraftSaveRequest` | ✅ |
| 워크플로 전이 | `POST /api/v1/documents/{id}/versions/{vid}/workflow/{action}` `{}` | `workflow.py:101/149/246` — comment/reason 모두 optional | ✅ |
| RAG 질의 | `POST /api/v1/rag/answer` `{query, top_k, document_type?, actor_type}` | `rag.py:62 rag_answer`, `RAGRequest` (schemas/rag.py:154) | ✅ |

### 2.5 지표 정의

| 지표 | 정의의 올바름 | 비고 |
|------|--------------|-----|
| retrieval_hit_rate | positive 질의 중 expected_doc_ids ∩ citation.sample_id ≠ ∅ 비율 | ✅ |
| citation_present_rate_positive | positive 중 `len(citations)>0` 비율 | ✅ |
| avg_precision_at_k | citation 당 expected 포함 비율의 산술평균 | ✅ (MAP 아닌 단순 평균, 설계 의도와 일치) |
| avg_keyword_coverage | answer + snippet 합본 내 expected_keywords 매칭 비율 평균 | ✅ |
| multihop_ok | 동일 expected doc 에 대한 citation 수 ≥ min_citations_from_doc | ✅ |
| negative_ok | `citation==0` or 답변에 "모르겠/없음" 마커 포함 | ✅ (폐쇄망 placeholder LLM 환경에서도 동작) |

### 2.6 운영적 안전

| 항목 | 결과 | 근거 |
|------|------|------|
| 429 Retry-After 처리 | ✅ | `HttpClient._request`의 except HTTPError 분기에서 Retry-After 파싱 후 1회 재시도 |
| 마지막 질의 sleep 생략 | ✅ | `if i < len(GOLDEN_QUERIES) - 1: time.sleep(...)` — 불필요한 대기 방지 |
| 시딩 실패 시 조기 반환 | ✅ | `SeededDoc.errors` 누적 후 미발행 문서 건너뜀 |
| 시딩 0건 → 조기 종료 | ✅ | `return 3` 종료코드로 쿼리 단계 진입 차단 |
| 에러 발생 시 종료코드 분리 | ✅ | 0/2/3/4/5 체계 (가이드 §4.3) |
| 로그 형식 일관성 | ✅ | `logging.basicConfig` 하나로 통일, stderr 출력 |
| JSON 보고서 UTF-8 + ensure_ascii=False | ✅ | 한국어 답변/스니펫 그대로 저장 |
| report-out 경로 부모 디렉터리 자동 생성 | ✅ | `parent.mkdir(parents=True, exist_ok=True)` |

---

## 3. 실측 교차 검증 결과

본 세션은 백엔드 컨테이너가 기동된 상태가 아니므로 실제 RAG 실행은 하지 않았다. 정적 검증만 수행했으며, 검증 과정에서 `expected_keywords` 와 실제 문서 본문 간 표기 불일치 2건(Q05, Q15)을 발견·교정했다.

```
docs = 16
by type = {'POLICY': 5, 'MANUAL': 6, 'REPORT': 5}
queries = 21
by cat  = {'simple': 8, 'ambiguous': 4, 'typed': 3, 'multihop': 3, 'negative': 3}
missing expected_doc_ids = []
snapshot root OK. top-level nodes= 12
```

실행은 사용자 호스트에서 `python scripts/rag_smoke/rag_smoke.py ...` 로 수행 예정이며, 실측 JSON 은 `docs/개발문서/S2_5/RAG_스모크_실측_*.json` 경로에 보관한다.

---

## 4. 결함 / 제한사항

1. **`--skip-seed` 모드의 매핑 공백** — 이전 실행의 `document_id`를 디스크에 남기지 않아, skip-seed 에서는 citation의 document_id→sample_id 매핑이 모두 비어 `retrieval_hit_rate = 0` 으로 집계된다. 이는 설계상 의도된 제한으로, 가이드 §3.3에 명시했다.
2. **폐쇄망 모드에서 keyword_coverage** — placeholder LLM은 answer 본문을 생성하지 않거나 짧게 생성할 수 있으므로 `avg_keyword_coverage` 가 낮게 집계될 수 있다. citation snippets 도 함께 검색 대상이므로 0 으로 수렴하지는 않지만, 해석 시 주의 필요.
3. **negative 카테고리의 "모르겠다" 마커는 한국어 표현 고정** — 다른 언어 모델이나 다른 프롬프트 구성에서는 미감지 가능. 마커 리스트는 `run_query()` 내부 상수로 유지.
4. **rate limit 정책 의존** — `/rag/answer` 30/min 정책이 변경되면 `--rag-sleep` 기본값(2.2s)도 재조정 필요.
5. **실 DB CI 미포함** — 본 스모크는 수동 실행. S2 종결 테스트 회고 §5 Try 에 기록된 "S3에서 실 DB CI 도입" 항목과 연결.

---

## 5. 결론

정적 검증 기준 본 스모크 스크립트는 제시된 요구(샘플 15건 이상, Python 스크립트로 직접 호출, 폐쇄망 모드)에 부합하며 백엔드 API 계약과 정합된다. 실행은 사용자 호스트에서 수행하고, 실측 JSON 이 생성된 후 추가 분석(카테고리별 hit, 실패 쿼리 원인)을 `RAG_스모크_실측_*.json` 리포트 기반으로 이어갈 수 있다.

S2 산출물 규약(검수·보안·가이드 3종) 중 가이드와 본 검수보고서가 완료되었고, `RAG_스모크_보안취약점검사보고서.md` 를 동시 제출한다.

---

*작성: 2026-04-22 | Claude Opus 4.7*
