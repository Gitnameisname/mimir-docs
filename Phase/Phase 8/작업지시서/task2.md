# Task 8-2. 검색 인덱스 구조 설계

## 1. 작업 목적

PostgreSQL FTS 기반 검색을 위한 DB 레벨 인덱스 구조를 설계한다.

이 작업의 목표는 다음과 같다.

- Document / Version / Node 각각에 대한 tsvector 컬럼 설계
- 검색 가중치 체계 정의 (제목 > 메타데이터 > 본문 > 노드)
- GIN 인덱스 구성으로 검색 성능 확보
- 문서 변경 시 인덱스 자동 갱신 방식 정의
- 기존 데이터 일괄 인덱싱 마이그레이션 계획

---

## 2. 작업 범위

### 포함 범위

- tsvector 컬럼 설계 (엔티티별)
- 검색 가중치(weight) 체계 정의
- GIN 인덱스 설계
- 언어별 FTS 설정 전략
- 인덱스 자동 갱신 트리거 설계
- 마이그레이션 전략 (기존 데이터 일괄 인덱싱)
- 메타데이터(JSONB) 검색 지원 방안

### 제외 범위

- 실제 마이그레이션 코드 작성
- 트리거 함수 구현
- 성능 튜닝 (Task 8-10에서 다룸)

---

## 3. 선행 조건

- Task 8-1 (검색 시스템 아키텍처 설계) 완료
- Phase 1: Document / Version / Node 테이블 스키마 이해

---

## 4. 주요 설계 대상

### 4-1. Document 테이블 검색 인덱스

#### 검색 대상 컬럼

| 컬럼 | 가중치 | 설명 |
|------|--------|------|
| title | A (최고) | 문서 제목 |
| metadata 주요 필드 | B | JSON에서 추출한 핵심 메타데이터 |
| summary / description | C | 문서 요약 또는 설명 (있는 경우) |

#### tsvector 컬럼 설계

```sql
-- 개념적 설계
ALTER TABLE documents ADD COLUMN search_vector tsvector;

-- tsvector 생성 방식 (개념)
setweight(to_tsvector('simple', coalesce(title, '')), 'A') ||
setweight(to_tsvector('simple', coalesce(metadata_text, '')), 'B')
```

#### GIN 인덱스

```sql
CREATE INDEX idx_documents_search_vector ON documents USING GIN (search_vector);
```

---

### 4-2. Version 테이블 검색 인덱스

#### 검색 대상 컬럼

| 컬럼 | 가중치 | 설명 |
|------|--------|------|
| content_text | B | 버전 본문 평문 추출 텍스트 |
| version_note | C | 버전 메모/변경 사유 |

#### 설계 고려사항

- Version 테이블은 행 수가 많을 수 있어 인덱스 크기 주의
- 검색 대상 버전 범위 정책 필요:
  - 기본: 최신 Published 버전만 인덱싱
  - 선택: 모든 버전 인덱싱 (관리자 검색용)
- 본문 구조(JSON/트리)에서 평문 추출 방식 정의 필요

#### content_text 추출 방식

Node 트리 구조에서 본문 평문을 추출하여 별도 컬럼에 저장하거나, 검색 시 동적으로 추출하는 방식 중 선택:

| 방식 | 장점 | 단점 |
|------|------|------|
| 사전 추출(컬럼 저장) | 검색 속도 빠름 | 저장 공간 증가, 동기화 필요 |
| 동적 추출 | 항상 최신 | 검색 시 처리 비용 |

→ **권장: 사전 추출 방식** (검색 성능 우선)

---

### 4-3. Node 테이블 검색 인덱스

#### 검색 대상 컬럼

| 컬럼 | 가중치 | 설명 |
|------|--------|------|
| node_title | A | 노드 제목/헤딩 |
| node_content_text | B | 노드 내용 평문 |

#### 설계 고려사항

- Node는 Document/Version보다 행 수가 훨씬 많을 수 있음
- 노드 단위 검색 결과는 부모 Document ID를 항상 포함
- 삭제된 문서의 노드는 검색 결과에서 제외 (논리 삭제 시 필터)

---

### 4-4. 검색 가중치 체계

PostgreSQL FTS의 가중치(A/B/C/D)를 다음과 같이 할당한다:

| 가중치 | 적용 대상 | 랭킹 기여도 |
|--------|----------|-----------|
| A | 문서 제목, 노드 제목 | 최고 |
| B | 버전 본문, 노드 내용, 메타데이터 핵심 필드 | 높음 |
| C | 버전 메모, 메타데이터 부가 필드 | 보통 |
| D | 기타 | 낮음 |

#### 가중치 점수 설정 (ts_rank_cd 기준)

```
{D가중치, C가중치, B가중치, A가중치}
기본값: {0.1, 0.2, 0.4, 1.0}
```

---

### 4-5. 언어별 FTS 설정 전략

#### 한국어

- PostgreSQL 기본 딕셔너리(`simple`)로 기본 지원
- `simple` 딕셔너리: 소문자 변환 + 불용어 제거 정도
- 형태소 분석이 필요한 경우 `pg_bigm` 확장 검토
  - pg_bigm: 2-gram 방식으로 한국어 부분 문자열 검색 지원
  - GIN 인덱스와 함께 사용 가능

#### 영어

- `english` 딕셔너리 사용 (어근 분석, 불용어 처리 포함)

#### 다국어 혼용 문서 처리

- 기본 전략: `simple` 딕셔너리로 통일 (한영 혼용 환경 고려)
- 언어 감지 후 딕셔너리 선택 방식은 Phase 이후 고려

---

### 4-6. 인덱스 자동 갱신 트리거 설계

#### 트리거 이벤트 및 대상

| 테이블 | 이벤트 | 갱신 대상 |
|--------|--------|---------|
| documents | INSERT, UPDATE(title, metadata, status) | documents.search_vector |
| versions | INSERT, UPDATE(content, status) | versions.search_vector |
| nodes | INSERT, UPDATE(title, content), DELETE | nodes.search_vector |

#### 트리거 설계 원칙

- 각 테이블에 BEFORE INSERT OR UPDATE 트리거로 search_vector 자동 갱신
- 관련 없는 컬럼 변경 시 불필요한 재계산 방지 (변경 컬럼 감지)
- 문서 삭제(논리)/비활성화 시 search_vector를 NULL로 설정 또는 별도 is_searchable 플래그 관리

#### 트리거 함수 개념 (설계 수준)

```
FUNCTION update_document_search_vector()
  NEW.search_vector ←
    setweight(to_tsvector('simple', NEW.title), 'A') ||
    setweight(to_tsvector('simple', extract_metadata_text(NEW.metadata)), 'B')
  RETURN NEW
```

---

### 4-7. 메타데이터(JSONB) 검색 지원

Document의 metadata JSONB 컬럼에서 검색 가능한 텍스트를 추출하는 방식을 정의한다.

#### 추출 방식

1. **전체 추출**: JSONB를 text로 변환하여 FTS에 포함 (단순하지만 노이즈 가능)
2. **선택적 추출**: DocumentType별로 검색 대상 필드를 정의하여 해당 필드만 추출 (권장)

#### DocumentType별 메타데이터 검색 필드 설정

```
DocumentType 설정에 search_fields 목록을 추가:
  - POLICY 타입: ["policy_number", "department", "summary"]
  - MANUAL 타입: ["product_name", "version", "category"]
```

이 설정은 Task 8-7 (DocumentType 기반 검색 차별화)에서 상세 구현.

---

### 4-8. 초기 마이그레이션 전략

기존 데이터에 대한 일괄 인덱싱 계획:

#### 마이그레이션 단계

1. tsvector 컬럼 및 GIN 인덱스 추가 (비어있는 상태)
2. 배치 스크립트로 기존 Document/Version/Node 순서로 인덱싱
3. 인덱싱 진행 상태를 Admin UI에서 모니터링 가능 (Phase 7 연계)
4. 완료 후 트리거 활성화

#### 성능 고려

- 대량 데이터 인덱싱 시 CONCURRENTLY 옵션 활용
- 배치 처리 단위 크기 설정 (예: 100건 단위)
- 인덱싱 중 서비스 영향 최소화 방안

---

## 5. 산출물

1. 검색 인덱스 스키마 설계서 (엔티티별 tsvector 컬럼 및 GIN 인덱스 정의)
2. 검색 가중치 체계 정의서
3. 인덱스 자동 갱신 트리거 설계서
4. 초기 마이그레이션 전략 문서

---

## 6. 완료 기준

- Document / Version / Node 각각의 tsvector 컬럼 설계가 완료되어 있다
- 검색 가중치 체계가 구체적으로 정의되어 있다
- GIN 인덱스 구성이 명확히 기술되어 있다
- 인덱스 자동 갱신 트리거 이벤트와 동작이 설계되어 있다
- 메타데이터 JSONB 검색 지원 방안이 정의되어 있다
- 초기 마이그레이션 전략이 수립되어 있다

---

## 7. Codex 작업 지침

- 코드 작성 금지 (설계 중심, 개념적 SQL 예시는 구조 설명 목적으로 허용)
- PostgreSQL FTS 메커니즘(tsvector/tsquery/ts_rank)에 기반한 설계를 한다
- 노드 테이블의 행 수가 많을 수 있음을 고려하여 인덱스 크기와 갱신 비용을 설계에서 언급한다
- 메타데이터 검색은 DocumentType 설정과 연계되어야 함을 명시한다
- Task 8-7 (DocumentType 기반 검색 차별화)와의 연계 포인트를 명확히 한다
