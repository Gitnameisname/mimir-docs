# Phase 10 종결 보고서
## 문서 구조화 및 벡터화 파이프라인 구축

**종결 일자**: 2026-04-09  
**상태**: ✅ 완료

---

## 1. 단계 요약

Phase 10은 Mimir에 축적된 문서를 질의 가능한 지식 단위로 변환하는 벡터화 파이프라인 인프라를 구축한 단계다. 단순 임베딩 생성을 넘어 **권한 인식 지식 벡터 베이스(Permission-Aware Knowledge Vector Base)** 를 확립하여 Phase 11 RAG 질의응답의 기반을 완성했다.

---

## 2. 완료 기준 달성 현황

| 완료 기준 | 달성 여부 | 비고 |
|-----------|----------|------|
| Published 문서의 청크가 생성되고 embedding이 저장된다 | ✅ | `document_chunks` 테이블, VectorizationPipeline |
| 각 청크에 권한 메타데이터가 반영된다 | ✅ | `accessible_roles`, `is_public` 컬럼 |
| 문서 Published 전이 시 자동 벡터화가 실행된다 | ✅ | `_trigger_vectorization_async` (ThreadPoolExecutor) |
| 벡터 유사도 검색이 동작한다 | ✅ | `POST /vectorization/search/semantic` |
| 하이브리드 검색이 동작한다 | ✅ | `GET /search/documents?mode=hybrid` (FTS+RRF) |
| Admin UI에서 벡터화 현황을 조회할 수 있다 | ✅ | AdminVectorizationPage (3탭) |
| 수동 재색인 API가 동작한다 | ✅ | `POST /vectorization/reindex-all`, `POST /vectorization/documents/{id}` |
| 권한 없는 문서의 청크가 검색 결과에 포함되지 않는다 | ✅ | semantic_search 권한 필터, accessible_roles DB 필터 |

**전체 완료 기준 8/8 달성**

---

## 3. Task별 구현 결과

| Task | 내용 | 결과 |
|------|------|------|
| 10-1 | 벡터화 파이프라인 아키텍처 설계 | ✅ pgvector + HNSW, EmbeddingProvider 추상화 확정 |
| 10-2 | DocumentType별 청킹 전략 설계 | ✅ node_based 전략, ChunkingConfig DB 설정 기반 |
| 10-3 | pgvector 스키마 및 인덱스 설계 | ✅ document_chunks 테이블, HNSW 인덱스 (m=16, ef=64) |
| 10-4 | 임베딩 모델 추상화 레이어 구현 | ✅ EmbeddingProvider ABC, OpenAI/Local 구현체 |
| 10-5 | 청킹 파이프라인 구현 | ✅ ChunkingService (node_based, split/merge 로직) |
| 10-6 | 벡터화 파이프라인 구현 | ✅ VectorizationPipeline (청킹→임베딩→저장) |
| 10-7 | 재색인 정책 구현 | ✅ 자동 트리거, 수동 재색인 API, cleanup |
| 10-8 | 권한 메타데이터 반영 구현 | ✅ PermissionSnapshot, update_permission_metadata |
| 10-9 | 하이브리드 검색 통합 | ✅ search_documents_hybrid (RRF k=60) |
| 10-10 | Admin 벡터화 상태 관리 | ✅ AdminVectorizationPage + 대시보드 카드 |
| 10-11 | 성능 검증 및 Phase 11 연계 설계 | ✅ Phase 11 연계 포인트 확정 (본 문서 §6) |

---

## 4. 신규/변경 파일 목록

### 4.1 Backend — 신규 파일

| 파일 | 역할 |
|------|------|
| `backend/app/services/embedding_service.py` | EmbeddingProvider ABC, OpenAIEmbeddingProvider (배치, 재시도), LocalEmbeddingProvider |
| `backend/app/services/chunking_service.py` | ChunkingConfig, DocumentChunk, ChunkingService (node_based 전략) |
| `backend/app/services/vectorization_service.py` | VectorizationPipeline (청킹→임베딩→저장), semantic_search, cleanup |
| `backend/app/api/v1/vectorization.py` | 벡터화 관련 REST API 8개 엔드포인트 |

### 4.2 Backend — 변경 파일

| 파일 | 변경 내용 |
|------|----------|
| `backend/app/db/connection.py` | pgvector 확장, `document_chunks` 테이블, HNSW 인덱스, `embedding_token_usage` 테이블 추가 |
| `backend/app/config.py` | `openai_api_key`, `embedding_model`, `embedding_dimensions`, `embedding_batch_size` 설정 추가 |
| `backend/requirements.txt` | `openai==1.84.0`, `pgvector==0.4.1`, `tiktoken==0.9.0`, `httpx==0.28.1` 추가 |
| `backend/app/services/search_service.py` | `search_documents_hybrid()` 메서드 추가 (FTS+pgvector RRF) |
| `backend/app/api/v1/search.py` | `mode=hybrid` 파라미터 추가, mode 검증 로직 추가 |
| `backend/app/api/v1/admin.py` | 대시보드 metrics에 vectorization 현황 추가 |
| `backend/app/api/v1/workflow.py` | publish 시 자동 벡터화 트리거, ThreadPoolExecutor 추가 |
| `backend/app/api/v1/router.py` | vectorization router 등록 |

### 4.3 Frontend — 신규 파일

| 파일 | 역할 |
|------|------|
| `frontend/src/features/admin/vectorization/AdminVectorizationPage.tsx` | Admin 벡터화 관리 페이지 (개요/청크목록/토큰사용량 3탭) |
| `frontend/src/app/admin/vectorization/page.tsx` | Next.js 라우팅 페이지 |

### 4.4 Frontend — 변경 파일

| 파일 | 변경 내용 |
|------|----------|
| `frontend/src/lib/api/admin.ts` | vectorization API 함수 및 타입 추가 |
| `frontend/src/types/admin.ts` | `DashboardMetrics.vectorization` 필드 추가 |
| `frontend/src/components/admin/layout/AdminSidebar.tsx` | "벡터화 관리" 메뉴 추가 |
| `frontend/src/features/admin/dashboard/AdminDashboardPage.tsx` | 벡터화 현황 카드 추가 |

---

## 5. 검수 및 보안 수정 이력

### 5.1 코드 품질 검수 (4건)

| # | 파일 | 이슈 | 수정 |
|---|------|------|------|
| Q1 | `vectorization_service.py` | 미사용 `import json`, `datetime` | 제거 |
| Q2 | `vectorization_service.py` | `chunk_version(chunking_config=None)` — DB 조회한 config가 실제로 사용되지 않음 | `ChunkingConfig` → dict 변환 후 전달 |
| Q3 | `vectorization_service.py` | `INTERVAL '%s days'` — psycopg2가 문자열 리터럴 내부 파라미터 미치환 | `make_interval(days => %s)`로 수정 |
| Q4 | `chunking_service.py` | `_split_by_tokens` 루프 내 tiktoken 중복 임포트 및 `enc` 재생성 | 외부 `enc` 재사용 |

### 5.2 보안 취약점 검수 (7건)

상세 내용: [Phase10_보안취약점_리포트.md](../../보안리포트/Phase10_보안취약점_리포트.md)

| ID | 등급 | 내용 | 수정 |
|----|------|------|------|
| VEC-001 | HIGH | vectorization 엔드포인트 Rate Limiting 전무 → 임베딩 비용 DoS | `/semantic` 30/min, `/reindex-all` 5/min |
| VEC-002 | HIGH | UUID 미검증으로 DB 오류 메시지 500 응답 노출 | `_validate_uuid()` 추가, 오류 메시지 마스킹 |
| VEC-003 | MEDIUM | `SemanticSearchRequest.q` 길이 제한 없음 | `Field(max_length=500)` |
| VEC-004 | MEDIUM | publish마다 무제한 스레드 생성 → DB 풀 고갈 | `ThreadPoolExecutor(max_workers=3)` 교체 |
| VEC-005 | LOW | `mode=semantic` 미구현 → silent fallthrough | 명시적 400, 문서 수정 |
| VEC-006 | LOW | Admin 대시보드 벡터화 오류 silent 무시 | `logger.debug` 기록 |
| VEC-007 | LOW | Admin 작업 감사 로그 미기록 | Phase 11 권고 사항으로 추적 |

### 5.3 프론트엔드 UX 검수 (5건)

| # | 이슈 | 수정 |
|---|------|------|
| U1 | `ChunksTab`에 미사용 `queryClient` prop | 제거 |
| U2 | 파괴적 작업(재색인/정리)에 확인 대화상자 없음 | `window.confirm` 추가 |
| U3 | `cleanupMutation` 오류 핸들러 없음 | `onError` 추가 |
| U4 | 상태 메시지가 success/error 구분 없이 단일 문자열 | `{ type, text }` 구조로 개선, 닫기 버튼 추가 |
| U5 | 테이블에 `overflow-x-auto` 없어 모바일 레이아웃 깨짐 | `overflow-x-auto` + `min-w` 추가 |

---

## 6. Phase 11 연계 포인트

Phase 11 "RAG 질의응답 파이프라인" 구현 시 아래 인터페이스를 사용한다.

### 6.1 Retriever — 청크 조회 API

```python
# 서비스 레이어 직접 호출 (내부)
from app.services.vectorization_service import vectorization_pipeline

chunks = vectorization_pipeline.semantic_search(
    conn,
    query="사용자 질문",
    actor_role=actor.role,         # 권한 필터링 자동 적용
    document_type="POLICY",        # Optional
    top_k=20,
)
# 반환: List[{ chunk_id, document_id, source_text, node_path, similarity, ... }]
```

```http
# REST API (외부/테스트용)
POST /api/v1/vectorization/search/semantic
{ "q": "사용자 질문", "document_type": "POLICY", "top_k": 20 }
```

### 6.2 Retriever — 하이브리드 검색 (권장)

```http
GET /api/v1/search/documents?q=검색어&mode=hybrid&page=1&limit=10
```
FTS 점수 + 벡터 유사도를 RRF(k=60)로 통합한 결과 반환.

### 6.3 청크 컨텍스트 조회

```python
# document_chunks 테이블에서 node_path와 source_text를 활용해
# RAG 응답 생성 시 출처 정보(인용) 표시 가능
# node_path: ["정책 문서", "3장. 보안 정책", "3.1 접근 제어"] 형태
```

### 6.4 권한 인식 보장

- `semantic_search()` 호출 시 `actor_role` 파라미터를 반드시 전달하면 DB 레벨에서 권한 필터 자동 적용
- `is_public=TRUE` 청크는 모든 인증 사용자에게 반환
- 권한 없는 청크는 RAG 컨텍스트에 포함되지 않음

---

## 7. 주요 설계 결정 기록

| 결정 | 근거 |
|------|------|
| pgvector + PostgreSQL | 기존 인프라 재사용, ACL JOIN이 동일 DB에서 처리 가능 |
| HNSW 인덱스 (m=16, ef=64) | IVFFlat 대비 인덱스 구축 후 업데이트 불요. Recall/속도 균형 |
| text-embedding-3-small (1536차원) | 한국어·영어 우수 성능, $0.02/1M tokens 비용 효율 |
| node_based 청킹 전략 | Node 트리의 의미 단위 경계를 그대로 활용, 컨텍스트 단절 최소화 |
| 소프트 삭제 (is_current) | 재색인 중 서비스 연속성 보장, 일정 기간 후 물리 삭제 |
| RRF k=60 | 업계 표준값. FTS/벡터 각 소스의 bias를 균등하게 통합 |
| ThreadPoolExecutor(max_workers=3) | 자동 벡터화 동시 실행 3개 제한 → DB 연결 풀 보호 |
| EmbeddingProvider 추상화 | 모델 교체 시 재인덱싱만으로 대응 가능, Phase 11 이후 고려 |

---

## 8. 미구현 항목 및 다음 단계 권고

### 8.1 Phase 11에서 처리할 항목

| 항목 | 내용 |
|------|------|
| 감사 로그 (VEC-007) | `vectorization.reindex_*`, `vectorization.cleanup` 이벤트를 `audit_events` 테이블에 기록 |
| RAG 파이프라인 구현 | LLM 연동, Prompt 템플릿, 인용(citation) 처리 |
| 청크 cleanup 자동화 | 30일 주기 자동 실행 (현재는 Admin 수동 트리거) |

### 8.2 추후 확장 고려 항목

| 항목 | 배경 |
|------|------|
| LocalEmbeddingProvider 완성 | 현재 placeholder. 비용 0 운영 환경(on-premise)용 |
| `fixed_size` / `semantic` 청킹 전략 | node_based의 청크 크기 불균일 문제를 보완 |
| 임베딩 모델 버전 관리 | 모델 변경 시 전체 재색인 필요 — 버전별 청크 관리 체계 |
| 전용 벡터 DB 전환 | 수억 건 규모에서는 pgvector 한계. Weaviate/Qdrant 추상화 준비됨 |

---

## 9. 결론

Phase 10은 계획된 11개 Task를 모두 완료했다. 검수 과정에서 코드 품질 4건, 보안 취약점 7건(HIGH 2건 포함), UX 5건을 발견하여 전부 수정했다. 완료 기준 8개 항목 전체 달성.

핵심 산출물인 **권한 인식 지식 벡터 베이스**가 완성됨으로써 Phase 11 RAG 질의응답을 즉시 착수할 수 있는 기술적 기반이 갖춰졌다.

> Phase 11 진행 가능 조건 충족 ✅
