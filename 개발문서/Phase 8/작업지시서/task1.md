# Task 8-1. 검색 시스템 아키텍처 설계

## 1. 작업 목적

Phase 8 전체 검색 시스템의 기술 방향과 구조를 확정한다.

이 작업의 목표는 다음과 같다.

- 검색 엔진 기술 선택과 그 근거를 명확히 정리
- 검색 대상 엔티티와 계층 구조 정의
- 검색 서비스 레이어를 추상화하여 Phase 10 벡터 검색 확장 대비
- 인덱스 갱신 전략과 권한 필터링 연계 방식 확정

---

## 2. 작업 범위

### 포함 범위

- 검색 엔진 기술 선택 및 근거 문서화
- 검색 대상 엔티티 정의 (Document / Version / Node)
- 검색 서비스 레이어 추상화 구조 설계
- 인덱스 갱신 전략 정의
- 권한 필터링 연계 방식 결정
- Phase 10 확장 가능성 검토

### 제외 범위

- 실제 DB 스키마 설계 (Task 8-2에서 다룸)
- API 엔드포인트 설계 (Task 8-3에서 다룸)
- 구현 코드 작성

---

## 3. 선행 조건

- Phase 1: Document / Version / Node 구조 이해
- Phase 2: ACL 기반 권한 모델 이해
- Phase 3: 공통 API 규약 이해
- Phase 4~5: 문서/버전 데이터 구조 이해

---

## 4. 주요 설계 대상

### 4-1. 검색 엔진 기술 선택

#### 선택: PostgreSQL Full-Text Search (FTS)

Phase 8에서는 별도 검색 엔진 없이 PostgreSQL FTS를 기반으로 구현한다.

**선택 근거:**

| 항목 | 내용 |
|------|------|
| 인프라 단순성 | 기존 PostgreSQL 인프라 활용, 별도 시스템 불필요 |
| 권한 필터링 | ACL JOIN이 동일 DB 내에서 처리되어 구현 단순 |
| 확장성 | Phase 10에서 pgvector 추가 시 하이브리드 검색으로 자연 확장 |
| 기능 충분성 | tsvector/tsquery/ts_headline으로 Phase 8 요구사항 충족 |
| 운영 복잡도 | 검색 엔진 별도 운영/모니터링 불필요 |

**인지된 제약:**

- 대규모(수백만 건+) 환경에서 성능 한계 존재
- 한국어 형태소 분석이 기본 지원 제한적 (pg_bigm 확장 검토)
- 향후 필요 시 외부 검색 엔진 도입을 위한 추상화 레이어 필수

---

### 4-2. 검색 대상 엔티티 정의

검색 가능한 데이터 단위를 계층적으로 정의한다.

#### 계층 구조

```
Document (문서)
 └── Version (버전)
       └── Node (섹션/노드)
```

#### 엔티티별 검색 역할

| 엔티티 | 검색 목적 | 검색 가능 필드 |
|--------|----------|--------------|
| Document | 문서 단위 탐색 | 제목, 메타데이터(JSON 필드) |
| Version | 특정 버전 본문 검색 | 본문 전체 텍스트 |
| Node | 문서 내 섹션 탐색 | 노드 제목, 노드 내용 |

#### 검색 결과 기본 단위

- 사용자에게 반환되는 검색 결과의 기본 단위는 **Document**
- 노드 검색 결과는 부모 Document 컨텍스트를 포함하여 반환
- 동일 문서 내 여러 노드 매칭 시 그룹화하여 표시

---

### 4-3. 검색 서비스 레이어 추상화 구조

검색 엔진을 교체하거나 확장할 때 API 계약이 변경되지 않도록 서비스 레이어를 추상화한다.

#### 레이어 구조

```
[검색 API 엔드포인트]
        ↓
[SearchService Interface]  ← 이 계약은 Phase 10 이후에도 유지
        ↓
[FTSSearchProvider]        ← Phase 8: PostgreSQL FTS 구현체
        ↓ (Phase 10 이후)
[HybridSearchProvider]     ← Phase 10: FTS + pgvector 구현체
```

#### SearchService 인터페이스 핵심 메서드

- `search(query, filters, pagination, user_context)` → SearchResult
- `searchNodes(query, filters, pagination, user_context)` → NodeSearchResult
- `reindex(document_id)` → void

#### 설계 원칙

- SearchService는 엔진 구현 상세를 감추며, 상위 레이어는 Provider를 직접 알지 못한다
- user_context를 통해 권한 정보를 서비스 레이어에 전달
- 모든 권한 필터링은 Provider 구현 내부에서 처리

---

### 4-4. 인덱스 갱신 전략

검색 인덱스를 최신 상태로 유지하는 전략을 정의한다.

#### 갱신 트리거 이벤트

| 이벤트 | 인덱스 갱신 대상 | 갱신 방식 |
|--------|---------------|---------|
| 문서 생성 | Document 인덱스 | 즉시 (동기) |
| 문서 메타데이터 수정 | Document 인덱스 | 즉시 (동기) |
| 버전 저장/발행 | Version 인덱스 | 즉시 (동기) |
| 노드 내용 수정 | Node 인덱스 | 즉시 (동기) |
| 문서 상태 변경 | Document 인덱스 | 즉시 (동기) |
| 문서 삭제/비활성화 | 모든 관련 인덱스 | 즉시 (동기) |

#### 초기 인덱싱 (마이그레이션)

- 기존 데이터 일괄 인덱싱 배치 작업 필요
- 진행 상태 Admin UI에서 조회 가능해야 함 (Phase 7 연계)

#### 갱신 지연 허용 기준

- 정상적인 경우: 문서 저장 후 즉시 검색 가능
- 배치 재인덱싱: 완료까지 검색 결과 지연 허용 (운영자 안내 필요)

---

### 4-5. 권한 필터링 연계 방식

검색 쿼리에서 ACL을 어떻게 적용할지 결정한다.

#### 기본 방식: 검색 쿼리 내 ACL JOIN

```sql
-- 개념적 예시
SELECT d.*, ts_rank(d.search_vector, query) as rank
FROM documents d
JOIN document_permissions dp ON d.id = dp.document_id
WHERE d.search_vector @@ query
  AND dp.user_id = :user_id
  AND d.status IN ('PUBLISHED', ...)
ORDER BY rank DESC
```

#### 권한 적용 범위

- 문서 접근 권한 (열람 권한 없는 문서 제외)
- 문서 상태 필터 (Draft는 작성자/관리자만)
- 조직 범위 제한 (조직 기반 권한인 경우)

#### 성능 고려

- 권한 JOIN이 추가됨으로 인한 성능 영향 분석 필요
- 사용자별 접근 가능 문서 ID 캐싱 전략 검토 (Phase 이후)

---

### 4-6. Phase 10 벡터 검색 확장 고려

현재 FTS 구조가 향후 벡터 검색과 공존할 수 있도록 설계한다.

| 항목 | Phase 8 | Phase 10 이후 |
|------|---------|---------------|
| 검색 엔진 | PostgreSQL FTS | FTS + pgvector (하이브리드) |
| 쿼리 방식 | tsquery | tsquery + embedding similarity |
| API 계약 | 동일 유지 | 동일 유지 |
| 랭킹 | ts_rank | FTS rank + cosine similarity 결합 |

---

## 5. 산출물

1. 검색 시스템 아키텍처 문서
2. 기술 선택 근거 문서 (PostgreSQL FTS 선택 이유 및 제약 정리)
3. 검색 서비스 레이어 추상화 구조 다이어그램
4. 인덱스 갱신 전략 문서

---

## 6. 완료 기준

- 검색 엔진 기술 선택이 근거와 함께 확정되어 있다
- 검색 대상 엔티티와 계층 구조가 명확히 정의되어 있다
- 검색 서비스 레이어 추상화 구조가 설계되어 있다
- 인덱스 갱신 전략이 이벤트별로 정의되어 있다
- 권한 필터링 연계 방식이 결정되어 있다
- 이후 Task(8-2~8-11)가 이 문서를 기준으로 진행 가능하다

---

## 7. Codex 작업 지침

- 실제 코드 구현은 하지 않는다
- 설계 문서 중심으로 작성한다
- 기술 선택의 근거와 트레이드오프를 명확히 기술한다
- Phase 10 확장 가능성을 설계의 핵심 제약으로 반영한다
- 추상적 표현보다 명확한 구조 정의와 표를 활용한다
