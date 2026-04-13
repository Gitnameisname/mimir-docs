# Task 10-3. pgvector 스키마 및 인덱스 설계

## 1. 작업 목적

벡터 저장을 위한 PostgreSQL pgvector 기반 DB 스키마와 검색 인덱스를 설계한다.

이 작업의 목표는 다음과 같다.

- `document_chunks` 테이블 스키마 설계 (벡터 + 메타데이터)
- 권한/버전/타입 메타데이터 컬럼 설계
- HNSW 인덱스 설계 및 파라미터 선정
- 소프트 삭제(is_current) 방식 설계
- 마이그레이션 계획

---

## 2. 작업 범위

### 포함 범위

- pgvector 확장 설치 계획
- `document_chunks` 테이블 스키마 설계
- 메타데이터 컬럼 설계 (권한, 버전, 타입)
- 벡터 인덱스 설계 (HNSW 파라미터)
- 조회 쿼리 성능을 위한 보조 인덱스 설계
- 소프트 삭제 전략
- 마이그레이션 전략

### 제외 범위

- 실제 마이그레이션 코드 작성
- 성능 튜닝 (Task 10-11에서 다룸)

---

## 3. 선행 조건

- Task 10-1 (벡터화 파이프라인 아키텍처 설계) 완료
- Task 10-2 (DocumentType별 청킹 전략 설계) 완료

---

## 4. 주요 설계 대상

### 4-1. pgvector 확장 설치

```sql
-- 개념적 설계
CREATE EXTENSION IF NOT EXISTS vector;
```

#### 요구 버전
- PostgreSQL 14 이상
- pgvector 0.5.0 이상 (HNSW 인덱스 지원)

---

### 4-2. document_chunks 테이블 스키마

```sql
-- 개념적 설계
CREATE TABLE document_chunks (
  -- 식별자
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- 소스 참조
  document_id     UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  version_id      UUID NOT NULL REFERENCES versions(id) ON DELETE CASCADE,
  node_id         UUID REFERENCES nodes(id) ON DELETE SET NULL,
  
  -- 청킹 정보
  chunk_index     INTEGER NOT NULL,
  chunk_total     INTEGER NOT NULL,
  
  -- 텍스트
  source_text     TEXT NOT NULL,
  context_text    TEXT NOT NULL,  -- 부모 컨텍스트 포함 텍스트
  token_count     INTEGER NOT NULL,
  
  -- 임베딩
  embedding       vector(1536),   -- pgvector
  embedding_model VARCHAR(100),
  
  -- 문서 메타데이터
  document_type   VARCHAR(50) NOT NULL,
  document_status VARCHAR(50) NOT NULL,
  node_path       TEXT[],
  node_type       VARCHAR(50),
  
  -- 권한 메타데이터 (스냅샷)
  accessible_roles    TEXT[] NOT NULL DEFAULT '{}',
  accessible_org_ids  TEXT[] NOT NULL DEFAULT '{}',
  is_public           BOOLEAN NOT NULL DEFAULT false,
  
  -- 상태 관리
  is_current      BOOLEAN NOT NULL DEFAULT true,
  
  -- 타임스탬프
  created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
  updated_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now()
);
```

---

### 4-3. 벡터 인덱스 설계 (HNSW)

#### 인덱스 생성

```sql
-- 코사인 유사도 기반 HNSW 인덱스
CREATE INDEX idx_chunks_embedding_hnsw
  ON document_chunks
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);
```

#### HNSW 파라미터 설명

| 파라미터 | 값 | 설명 |
|---------|-----|------|
| `m` | 16 | 각 노드의 연결 수 (높을수록 정확, 메모리 증가) |
| `ef_construction` | 64 | 인덱스 구축 시 탐색 범위 (높을수록 정확, 구축 시간 증가) |
| 검색 시 `ef_search` | 40 | 검색 시 탐색 범위 (쿼리 파라미터로 조정 가능) |

#### is_current 부분 인덱스

현재 유효한 청크만 인덱싱하여 성능 최적화:

```sql
CREATE INDEX idx_chunks_embedding_current
  ON document_chunks
  USING hnsw (embedding vector_cosine_ops)
  WHERE is_current = true;
```

---

### 4-4. 보조 인덱스 설계

벡터 검색 외 조회 및 필터링을 위한 인덱스:

```sql
-- 문서별 청크 조회 (재색인 시 기존 청크 찾기)
CREATE INDEX idx_chunks_document_id ON document_chunks(document_id);
CREATE INDEX idx_chunks_version_id ON document_chunks(version_id);

-- 권한 필터링 (GIN 인덱스로 배열 연산 최적화)
CREATE INDEX idx_chunks_accessible_roles ON document_chunks USING GIN(accessible_roles);
CREATE INDEX idx_chunks_accessible_org_ids ON document_chunks USING GIN(accessible_org_ids);

-- 현재 유효 청크 필터
CREATE INDEX idx_chunks_is_current ON document_chunks(is_current) WHERE is_current = true;

-- 타입/상태 필터
CREATE INDEX idx_chunks_type_status ON document_chunks(document_type, document_status);
```

---

### 4-5. 소프트 삭제 전략

재색인 시 기존 청크를 즉시 삭제하지 않고 소프트 삭제:

#### 처리 흐름

1. 새 청크 배치 생성 (is_current = true)
2. 동일 `document_id + version_id`의 기존 청크를 `is_current = false`로 업데이트
3. 일정 기간(기본 7일) 후 `is_current = false` 청크 물리 삭제 (cleanup 배치)

#### 이유

- 재색인 도중 기존 검색 서비스 중단 없이 전환 가능
- 롤백이 필요할 때 이전 청크로 복원 가능

---

### 4-6. 벡터 검색 쿼리 구조 (개념)

권한 필터가 포함된 벡터 유사도 검색:

```sql
-- 개념적 예시
SELECT
  id, document_id, version_id, node_id,
  source_text, context_text, node_path,
  1 - (embedding <=> :query_vector) AS similarity
FROM document_chunks
WHERE is_current = true
  AND (
    accessible_roles && :user_roles
    OR is_public = true
  )
  AND document_status = 'PUBLISHED'
ORDER BY embedding <=> :query_vector
LIMIT :k;
```

---

### 4-7. 마이그레이션 전략

#### 단계

1. pgvector 확장 활성화
2. `document_chunks` 테이블 생성 (embedding 컬럼은 NULL 허용)
3. 보조 인덱스 생성 (CONCURRENTLY 옵션)
4. HNSW 인덱스 생성 (데이터 없는 상태이므로 빠름)
5. 벡터화 파이프라인 가동 → 청크 및 임베딩 채우기
6. 임베딩 충분히 채워진 후 HNSW 인덱스 재구성 (선택)

---

## 5. 산출물

1. document_chunks 테이블 스키마 정의서
2. HNSW 벡터 인덱스 설계서 (파라미터 근거 포함)
3. 보조 인덱스 설계서
4. 소프트 삭제 전략 문서
5. 마이그레이션 전략 문서

---

## 6. 완료 기준

- document_chunks 테이블 스키마가 권한/버전/타입 메타데이터를 포함하여 설계되어 있다
- HNSW 인덱스 파라미터가 선정 근거와 함께 정의되어 있다
- 권한 필터링을 위한 GIN 인덱스가 설계되어 있다
- 소프트 삭제 전략이 구체적으로 기술되어 있다
- 마이그레이션 계획이 단계적으로 수립되어 있다

---

## 7. Codex 작업 지침

- 코드 작성 금지 (설계 중심, 개념적 SQL은 구조 설명 목적으로 허용)
- `is_current = true` 조건을 모든 벡터 검색 쿼리의 필수 조건으로 명시한다
- 권한 필터링(GIN 인덱스)은 벡터 검색 성능에 영향을 미치므로 인덱스 설계에서 반드시 다룬다
- HNSW vs. IVFFlat 선택 근거를 설계서에 명시한다 (HNSW: 동적 삽입에 유리, IVFFlat: 정적 데이터에 유리)
