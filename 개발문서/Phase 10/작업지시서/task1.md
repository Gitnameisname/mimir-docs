# Task 10-1. 벡터화 파이프라인 아키텍처 설계

## 1. 작업 목적

Phase 10 전체 벡터화 파이프라인의 기술 방향과 구조를 확정한다.

이 작업의 목표는 다음과 같다.

- 청킹 → 임베딩 → 저장 → 갱신의 전체 파이프라인 흐름 설계
- pgvector 선택 근거 및 대안 비교 문서화
- 임베딩 모델 선택 근거 (OpenAI text-embedding-3-small)
- Phase 7 백그라운드 작업 시스템과의 연계 구조 설계
- 권한 메타데이터 반영 방식 결정

---

## 2. 작업 범위

### 포함 범위

- 전체 파이프라인 흐름 및 컴포넌트 정의
- 기술 선택 근거 (벡터 저장소, 임베딩 모델)
- Phase 7 백그라운드 작업 연계 구조
- 권한 메타데이터 반영 전략 결정
- Phase 8 하이브리드 검색 연계 구조 개요

### 제외 범위

- 청킹 전략 상세 설계 (Task 10-2에서 다룸)
- DB 스키마 상세 (Task 10-3에서 다룸)
- 구현 코드 작성

---

## 3. 선행 조건

- Phase 1~5: 문서/버전/권한/워크플로 구조 이해
- Phase 7: 백그라운드 작업 시스템 이해
- Phase 8: FTS 검색 레이어 및 SearchService 추상화 구조 이해

---

## 4. 주요 설계 대상

### 4-1. 전체 파이프라인 흐름

```
[트리거]
  문서 Published 전이
  수동 재색인 요청 (Admin)
        ↓
[VectorizationJob 생성]
  Phase 7 Background Job Queue에 등록
        ↓
[ChunkingService]
  DocumentType chunking_config 로드
  Node 트리 → Chunk[] 변환
  권한 메타데이터 스냅샷 포함
        ↓
[EmbeddingService]
  Chunk[] 배치 처리
  EmbeddingProvider.embed(texts) 호출
  토큰 사용량 기록
        ↓
[VectorStoreService]
  document_chunks 테이블에 upsert
  기존 청크 is_current=false 처리
  HNSW 인덱스 자동 갱신
        ↓
[완료 이벤트]
  Admin 대시보드 상태 갱신
  감사 이벤트 기록
```

---

### 4-2. 벡터 저장소 기술 선택

#### 선택: pgvector (PostgreSQL 확장)

| 항목 | 내용 |
|------|------|
| 선택 이유 | 기존 PostgreSQL 인프라 그대로 활용, ACL JOIN이 동일 DB에서 처리, 운영 복잡도 최소화 |
| 확장 방법 | `CREATE EXTENSION IF NOT EXISTS vector;` |
| 인덱스 방식 | HNSW (`vector_cosine_ops`) |
| 한계 | 수억 건 이상 규모에서는 전용 벡터 DB(Pinecone, Qdrant 등)가 유리 |

**인지된 대안과 비교:**

| 옵션 | 장점 | 단점 |
|------|------|------|
| pgvector | 기존 인프라, ACL 통합 쉬움 | 초대규모 성능 한계 |
| Qdrant | 벡터 특화, 고성능 | 별도 인프라, ACL 연계 복잡 |
| Pinecone | 관리형, SaaS | 비용, 데이터 외부 전송 |
| Weaviate | 스키마 풍부 | 운영 복잡도 증가 |

---

### 4-3. 임베딩 모델 선택

#### 선택: OpenAI text-embedding-3-small

| 항목 | 내용 |
|------|------|
| 차원 | 1536 (기본) / 256, 512로 축소 가능 |
| 한국어 성능 | 우수 |
| 비용 | $0.02 / 1M tokens (2025년 기준, 매우 저렴) |
| 선택 이유 | 한/영 혼용 문서에서 안정적 성능, 비용 효율 |

**추상화 구조로 모델 교체 가능:**

```
EmbeddingProvider (인터페이스)
  ├── OpenAIEmbeddingProvider (기본)
  └── LocalEmbeddingProvider (비용 절감용 Reserved)
```

---

### 4-4. Phase 7 백그라운드 작업 연계 구조

벡터화는 시간이 걸리는 작업이므로 Phase 7 Background Job 시스템에서 비동기 실행:

| 작업 타입 | 설명 |
|----------|------|
| `VECTORIZE_DOCUMENT` | 단일 문서 벡터화 |
| `VECTORIZE_BATCH` | DocumentType별 일괄 벡터화 |
| `VECTORIZE_ALL` | 전체 플랫폼 재벡터화 |
| `CLEANUP_STALE_CHUNKS` | is_current=false 청크 정리 |

- 작업 상태: Phase 7 Job 모니터링 화면에서 조회 가능
- 실패 시 재시도: Phase 7 재시도 정책 그대로 활용

---

### 4-5. 권한 메타데이터 반영 전략

각 청크가 생성될 때 해당 문서의 ACL을 스냅샷으로 포함:

- `accessible_roles[]`: 접근 가능한 역할 목록
- `accessible_org_ids[]`: 접근 가능한 조직 목록
- `is_public`: 모든 인증 사용자 접근 가능 여부

**권한 변경 시 처리:**
- 벡터 재계산 없이 청크의 권한 필드만 UPDATE (비용 효율)
- ACL 변경 이벤트 → 해당 문서 청크 일괄 권한 메타데이터 갱신

**벡터 검색 시 권한 필터:**
```sql
WHERE (accessible_roles && :user_roles OR is_public = true)
  AND embedding <=> :query_vector < :threshold
```

---

### 4-6. Phase 8 하이브리드 검색 연계 개요

Phase 8에서 예약한 HybridSearchProvider를 이 단계에서 완성:

- SearchService Interface는 변경 없음 (Phase 8 API 계약 유지)
- Provider 선택: `SEARCH_MODE` 설정 또는 `mode` 파라미터로 분기
  - `mode=keyword` → FTSSearchProvider (Phase 8, 기존 동작)
  - `mode=semantic` → VectorSearchProvider (신규)
  - `mode=hybrid` → HybridSearchProvider (신규, 기본값 전환 후보)

---

## 5. 산출물

1. 벡터화 파이프라인 아키텍처 문서 (전체 흐름 다이어그램 포함)
2. 기술 선택 근거 문서 (pgvector, OpenAI 선택 이유 및 대안 비교)
3. Phase 7 백그라운드 작업 연계 구조 문서
4. 권한 메타데이터 전략 문서
5. Phase 8 하이브리드 검색 연계 개요

---

## 6. 완료 기준

- 전체 파이프라인 흐름이 다이어그램으로 정의되어 있다
- pgvector와 OpenAI 모델 선택 근거가 문서화되어 있다
- Phase 7 백그라운드 작업 연계 구조가 명확히 정의되어 있다
- 권한 메타데이터 반영 방식이 결정되어 있다
- 이후 Task(10-2~10-11)가 이 문서를 기준으로 진행 가능하다

---

## 7. Codex 작업 지침

- 실제 코드 구현은 하지 않는다
- 기술 선택의 근거와 트레이드오프를 명확히 기술한다
- 권한 필터링은 벡터 저장 및 검색의 모든 단계에서 적용되어야 함을 설계 원칙으로 명시한다
- Phase 11 RAG를 위해 청크 구조가 어떻게 활용될지를 미리 고려한다
