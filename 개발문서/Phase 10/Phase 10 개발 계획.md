# Phase 10 개발 계획
## 문서 구조화 및 벡터화 파이프라인 구축

---

## 1. 단계 개요

### 단계명
Phase 10. 문서 구조화 및 벡터화 파이프라인 구축

### 목적
플랫폼에 축적된 문서를 지식 단위로 변환하여, Phase 11 RAG 질의응답의 기반을 구축한다.  
이 단계의 목표는 단순히 embedding을 생성하는 것이 아니라, **DocumentType별 최적 chunking 전략으로 문서를 의미 단위로 분해하고, 권한과 버전 정보가 반영된 벡터를 안정적으로 생성·저장·갱신하는 파이프라인을 확립**하는 것이다.

### 기대 결과
- DocumentType별 chunking 전략이 적용된 문서 구조화 파이프라인 가동
- 각 chunk에 대한 embedding 생성 및 pgvector 저장
- 권한/버전 메타데이터가 각 벡터에 반영되어 Phase 11 RAG에서 활용 가능
- 재색인 정책: 문서 변경 시 자동 갱신, 수동 재색인 지원
- Phase 8 FTS와 결합한 하이브리드 검색 기반 완성
- Admin UI에서 벡터화 상태 전체 조회 및 제어 가능

---

## 2. Phase 10의 역할

Phase 10은 이미 구축된 다음 요소들을 **지식 벡터 변환 파이프라인으로 연결하는 단계**다.

- Phase 1에서 정의한 Document / Version / Node 트리 구조 → chunking 단위로 활용
- Phase 2에서 정의한 ACL 권한 모델 → 각 벡터 chunk에 권한 메타데이터 반영
- Phase 4~5에서 구축한 버전/워크플로 시스템 → 어느 버전을 벡터화할지 결정 기준
- Phase 7에서 구축한 백그라운드 작업 시스템 → 벡터화 파이프라인 실행 기반
- Phase 7에서 구축한 Admin UI → 인덱싱 상태 관리 화면 확장
- Phase 8에서 구축한 FTS 검색 레이어 → 하이브리드 검색으로 확장

즉, Phase 10은 새로운 UI나 도메인 기능을 추가하는 단계가 아니라,  
**기존 문서 데이터를 RAG가 활용할 수 있는 벡터 지식 베이스로 전환하는 인프라 구축 단계**다.

---
1
## 3. 개발 목표

### 3.1 DocumentType별 청킹 전략 구축
- Node 트리 구조를 활용한 구조 단위 청킹 (가장 자연스러운 경계)
- DocumentType별 청킹 설정을 하드코딩 없이 설정 기반으로 관리
- 청크 크기, 오버랩, 분할 기준을 타입별로 다르게 설정 가능

### 3.2 임베딩 생성 파이프라인 구축
- 외부 임베딩 모델 API 연동 (OpenAI text-embedding-3-small 기본)
- 배치 처리로 API 비용 최소화
- 모델 교체 가능한 추상화 레이어 설계

### 3.3 pgvector 기반 벡터 저장
- PostgreSQL pgvector 확장으로 기존 인프라 활용
- 벡터 저장 스키마 설계 (chunk 메타데이터, 권한, 버전 정보 포함)
- IVFFlat 또는 HNSW 인덱스 설계

### 3.4 권한 반영 벡터
- 각 청크에 문서 ACL 메타데이터 포함
- 벡터 검색 시 권한 필터링이 가능한 구조
- 권한 변경 시 관련 청크의 권한 메타데이터 갱신

### 3.5 재색인 정책 수립 및 구현
- 문서 Published/Approved 상태 전이 시 자동 벡터화 트리거
- 수동 재색인 지원 (단건/문서 전체/전체 플랫폼)
- 벡터화 실패 시 재시도 정책

### 3.6 Phase 8 검색과 하이브리드 통합
- Task 8-11에서 설계한 HybridSearchProvider 구현
- FTS 점수 + 벡터 유사도를 RRF(Reciprocal Rank Fusion)로 통합
- 검색 API 계약은 Phase 8과 동일하게 유지

---

## 4. 핵심 개발 범위

### 4.1 청킹 파이프라인 설계 및 구현

**구조 단위 청킹 (기본 전략)**
- Node 트리의 각 리프 노드 또는 섹션 단위를 청크로 사용
- 청크가 너무 작으면 부모 컨텍스트와 합치기 (min_chunk_size 설정)
- 청크가 너무 크면 내부 분할 (max_chunk_size 설정)

**DocumentType별 청킹 설정 (chunking_config)**
- `strategy`: `node_based` | `fixed_size` | `semantic` (기본: `node_based`)
- `max_chunk_tokens`: 청크 최대 토큰 수 (기본 512)
- `min_chunk_tokens`: 청크 최소 토큰 수 (기본 50)
- `overlap_tokens`: 청크 간 오버랩 토큰 수 (기본 50)
- `include_parent_context`: 부모 노드 제목을 청크 앞에 포함 여부
- `index_version_policy`: `published_only` | `latest` | `all`

**청크 메타데이터 구조**
- `document_id`, `version_id`, `node_id`
- `chunk_index` (문서 내 청크 순서)
- `source_text` (원본 텍스트)
- `node_path` (문서 내 위치 경로)
- `document_type`, `document_status`
- `permission_snapshot` (청크 생성 시점 권한 정보)

### 4.2 임베딩 생성 파이프라인

**임베딩 모델 추상화 레이어**
- `EmbeddingProvider` 인터페이스로 모델 교체 가능
- 기본 구현체: `OpenAIEmbeddingProvider` (text-embedding-3-small, 1536차원)
- 대안 구현체: `LocalEmbeddingProvider` (비용 절감 목적의 로컬 모델)

**배치 처리**
- 단건 처리 대신 배치(N건)로 API 호출 최소화
- 배치 크기: 기본 100건
- 실패 시 개별 재시도

**비용 추적**
- 토큰 사용량 기록 (API 비용 모니터링)
- Admin UI에서 임베딩 비용 현황 조회 가능

### 4.3 pgvector 저장 스키마

**document_chunks 테이블**
- `id` (PK)
- `document_id`, `version_id`, `node_id` (FK)
- `chunk_index` (문서 내 순서)
- `source_text` (원본 텍스트, FTS 보완용)
- `embedding vector(1536)` (pgvector)
- `embedding_model` (사용된 모델명)
- `token_count`
- `node_path text[]`
- `document_type`
- `document_status`
- `accessible_roles text[]` (권한 메타데이터)
- `accessible_user_ids text[]`
- `created_at`, `updated_at`
- `is_current` (해당 청크가 현재 유효한지)

**벡터 인덱스**
- HNSW 인덱스 (recall/speed 균형): `ivfflat` 또는 `hnsw` 중 선택
- 코사인 유사도 기준 검색

### 4.4 재색인 정책

**자동 트리거 이벤트**
- 문서 상태가 `PUBLISHED`로 전이될 때 (메인 트리거)
- 문서 메타데이터 수정 시 (제목, 주요 필드)
- 권한 변경 시 (ACL 메타데이터만 갱신)

**수동 재색인**
- 단건 문서 재색인 (Admin UI)
- DocumentType별 일괄 재색인
- 전체 플랫폼 재색인 (초기화 목적)

**재색인 시 기존 청크 처리**
- 신규 청크 생성 후 기존 청크 `is_current = false` 처리 (소프트 삭제)
- 일정 기간 후 주기적 정리(cleanup) 배치 실행

**실패 처리**
- 최대 3회 자동 재시도
- 실패 로그 기록 및 Admin UI 표시
- 임베딩 API 오류와 내부 오류 구분 처리

### 4.5 권한 반영

**권한 메타데이터 구조**
- 청크 생성 시점에 문서 ACL 스냅샷을 청크에 포함
- `accessible_roles`: 접근 가능한 역할 목록
- `accessible_org_ids`: 접근 가능한 조직 ID 목록
- `is_public`: 모든 인증 사용자 접근 가능 여부

**권한 변경 시 처리**
- ACL 변경 이벤트 수신 시 해당 문서의 청크 권한 메타데이터 갱신
- 벡터 재계산 없이 메타데이터만 업데이트 (비용 효율)

### 4.6 하이브리드 검색 통합 (Phase 8 연계)

**HybridSearchProvider 구현**
- Phase 8에서 예약한 `HybridSearchProvider` 구현체 완성
- FTS 검색 결과 + 벡터 유사도 검색 결과를 RRF로 통합
- 검색 API(`GET /search/documents`)의 파라미터에 `mode=hybrid` 추가

**RRF 통합 방식**
- FTS 상위 K개 + 벡터 상위 K개 수집
- `RRF_score = Σ 1/(k + rank_i)` (k=60)
- 통합 랭킹 기준 상위 결과 반환
- 권한 필터링은 통합 결과에도 동일하게 적용

---

## 5. 하위 Task 분해

### Task 10-1. 벡터화 파이프라인 아키텍처 설계
**목적** Phase 10 전체 파이프라인의 기술 방향과 구조를 확정한다.

**주요 작업**
- 청킹 → 임베딩 → 저장 전체 파이프라인 흐름 설계
- pgvector 선택 근거 및 대안 비교
- 임베딩 모델 선택 근거 (OpenAI text-embedding-3-small)
- Phase 7 백그라운드 작업 시스템과의 연계 구조 설계
- 권한 메타데이터 반영 방식 결정

**산출물**
- 벡터화 파이프라인 아키텍처 문서
- 기술 선택 근거 문서

---

### Task 10-2. DocumentType별 청킹 전략 설계
**목적** DocumentType에 따라 최적화된 청킹 전략을 설계한다.

**주요 작업**
- 구조 단위 청킹(node_based) 알고리즘 설계
- DocumentType 설정에 `chunking_config` 항목 추가 설계
- 청크 크기 계산 및 분할 기준 정의
- 부모 컨텍스트 포함 방식 정의
- 청크 메타데이터 스키마 확정

**산출물**
- 청킹 전략 설계서
- chunking_config 스키마 정의서

---

### Task 10-3. pgvector 스키마 및 인덱스 설계
**목적** 벡터 저장을 위한 DB 스키마와 검색 인덱스를 설계한다.

**주요 작업**
- `document_chunks` 테이블 스키마 설계
- 권한/버전/타입 메타데이터 컬럼 설계
- HNSW 인덱스 설계 및 파라미터 선정
- 소프트 삭제(is_current) 방식 설계
- 마이그레이션 계획

**산출물**
- document_chunks 테이블 스키마 설계서
- 벡터 인덱스 설계서

---

### Task 10-4. 임베딩 모델 추상화 레이어 설계 및 구현
**목적** 임베딩 모델 교체가 가능한 추상화 레이어를 설계하고 기본 구현체를 제공한다.

**주요 작업**
- EmbeddingProvider 인터페이스 설계
- OpenAIEmbeddingProvider 구현 (text-embedding-3-small)
- 배치 처리 로직 구현
- 토큰 사용량 추적 구현
- 오류 처리 및 재시도 로직

**산출물**
- EmbeddingProvider 인터페이스 및 구현체
- 배치 임베딩 생성 서비스

---

### Task 10-5. 청킹 파이프라인 구현
**목적** 문서를 DocumentType 설정에 따라 청크로 분해하는 파이프라인을 구현한다.

**주요 작업**
- ChunkingService 구현 (DocumentType별 전략 적용)
- Node 트리 기반 청크 분해 로직 구현
- 청크 크기 조정(merge/split) 로직 구현
- 부모 컨텍스트 주입 로직 구현
- 청크 메타데이터 생성 구현

**산출물**
- ChunkingService 구현
- 청킹 단위 테스트

---

### Task 10-6. 벡터화 파이프라인 구현 (청킹 + 임베딩 + 저장)
**목적** 청킹 → 임베딩 → 저장의 전체 파이프라인을 구현한다.

**주요 작업**
- VectorizationPipeline 서비스 구현
- Phase 7 백그라운드 작업과 연계하여 비동기 실행
- 진행 상태 추적 및 로깅
- 실패 처리 및 재시도 구현
- 기존 데이터 일괄 벡터화 배치

**산출물**
- VectorizationPipeline 서비스 구현
- 일괄 벡터화 배치 스크립트

---

### Task 10-7. 재색인 정책 구현
**목적** 문서 변경 이벤트에 따른 자동/수동 재색인을 구현한다.

**주요 작업**
- 문서 상태 변경 이벤트 수신 및 벡터화 트리거 구현
- 소프트 삭제 및 청크 갱신 로직 구현
- 수동 재색인 API 구현 (단건/일괄/전체)
- 청크 cleanup 배치 구현

**산출물**
- 재색인 트리거 이벤트 핸들러 구현
- 수동 재색인 API 구현
- 청크 cleanup 배치

---

### Task 10-8. 권한 메타데이터 반영 구현
**목적** 각 청크에 권한 정보를 포함하여 RAG 단계에서 권한 필터링이 가능하게 한다.

**주요 작업**
- ACL 스냅샷을 청크 메타데이터에 포함하는 로직 구현
- 권한 변경 이벤트 수신 시 청크 메타데이터만 갱신하는 로직 구현
- 권한 메타데이터 기반 벡터 검색 필터링 구현
- 권한 메타데이터 정확성 검증 테스트

**산출물**
- 권한 메타데이터 반영 구현
- 권한 필터링 벡터 검색 구현

---

### Task 10-9. 하이브리드 검색 통합 (Phase 8 HybridSearchProvider)
**목적** Phase 8 FTS와 벡터 검색을 RRF로 통합하는 HybridSearchProvider를 구현한다.

**주요 작업**
- HybridSearchProvider 구현 (FTS + pgvector)
- RRF 통합 랭킹 로직 구현
- 검색 API에 `mode=hybrid` 파라미터 추가
- 하이브리드 검색 결과 품질 평가
- Phase 8 API 계약 유지 확인

**산출물**
- HybridSearchProvider 구현
- 하이브리드 검색 통합 테스트

---

### Task 10-10. Admin 벡터화 상태 관리 (Phase 7 확장)
**목적** Phase 7 Admin UI에 벡터화 파이프라인 상태 관리 기능을 추가한다.

**주요 작업**
- 벡터화 현황 대시보드 카드 추가
- 청크 목록 및 상태 조회 화면
- 수동 재색인 트리거 UI
- 임베딩 비용 현황 표시
- 실패 청크 확인 및 재시도 UI

**산출물**
- Admin 벡터화 상태 관리 화면 구현
- Phase 7 대시보드 카드 확장

---

### Task 10-11. 성능 검증 및 Phase 11 연계 포인트 설계
**목적** 벡터화 파이프라인 성능을 검증하고 Phase 11 RAG와의 연계 구조를 설계한다.

**주요 작업**
- 벡터화 처리 속도 측정 (건당 처리 시간, 배치 처리량)
- 벡터 검색 응답 시간 측정
- 하이브리드 검색 품질 평가 (FTS 단독 vs. 하이브리드)
- Phase 11 Retriever가 사용할 청크 조회 API 설계
- RAG 파이프라인을 위한 청크 컨텍스트 조회 구조 Reserved

**산출물**
- 성능 검증 보고서
- Phase 11 연계 포인트 설계서

---

## 6. 구현 원칙

### 6.1 권한 우선 벡터화
- 벡터 청크는 생성 시점의 권한 정보를 반드시 포함한다
- 벡터 검색 결과는 검색 시점의 요청자 권한으로 필터링된다
- 권한 없는 문서의 청크가 RAG 응답에 포함되어서는 안 된다

### 6.2 DocumentType 인식 청킹
- 청킹 전략은 DocumentType 설정에서 읽어오며 하드코딩하지 않는다
- 타입별로 최적 청크 크기와 경계가 다를 수 있음을 항상 고려한다

### 6.3 버전 기반 벡터화
- 벡터화 대상 버전을 문서 상태 정책으로 명확히 정의한다 (기본: Published 버전)
- 버전별로 벡터를 독립적으로 관리하여 특정 버전 검색 가능

### 6.4 모델 교체 가능성 확보
- EmbeddingProvider 추상화로 모델을 언제든 교체 가능하게 한다
- 모델 변경 시 전체 재색인이 필요함을 운영 정책에 명시한다

### 6.5 API 계약 유지 (Phase 8 연계)
- 하이브리드 검색 도입 후에도 Phase 8의 검색 API 계약은 동일하게 유지한다
- `mode` 파라미터를 추가하되 기본값은 기존 FTS 동작을 유지한다

---

## 7. 기술 선택 방향

### 7.1 벡터 저장소: pgvector

| 항목 | 내용 |
|------|------|
| 선택 이유 | 기존 PostgreSQL 인프라 그대로 활용, ACL JOIN이 동일 DB에서 처리 |
| 인덱스 방식 | HNSW (높은 recall + 빠른 검색 속도 균형) |
| 벡터 차원 | 1536 (text-embedding-3-small 기본) |
| 인지된 제약 | 수억 건 규모에서는 전용 벡터 DB가 유리. 전환을 위한 추상화 유지 |

### 7.2 임베딩 모델: OpenAI text-embedding-3-small

| 항목 | 내용 |
|------|------|
| 선택 이유 | 한국어/영어 모두 우수한 성능, 비용 효율적 |
| 차원 | 1536 (또는 256/512로 축소 가능) |
| 비용 | $0.02 / 1M tokens (2025년 기준) |
| 대안 | text-embedding-3-large (고품질, 고비용), 로컬 모델 (비용 무료, 품질 낮음) |

### 7.3 청킹 기본 전략: 구조 단위 (node_based)

Node 트리의 자연스러운 경계를 청킹 단위로 사용:
- 장점: 의미 단위가 보존됨, 청크 간 컨텍스트 단절 최소화
- 단점: 청크 크기가 불균일할 수 있음 (크기 조정 로직 필요)

---

## 8. MVP 범위 제안

### MVP에 포함할 항목
- node_based 청킹 전략 구현
- OpenAIEmbeddingProvider 구현 (배치 처리)
- document_chunks 테이블 및 pgvector HNSW 인덱스
- Published 버전 자동 벡터화 트리거
- 권한 메타데이터 반영
- Admin 벡터화 현황 대시보드 카드
- 수동 재색인 API

### MVP 이후 확장 항목
- HybridSearchProvider (FTS + 벡터 통합)
- DocumentType별 청킹 설정 고도화
- 임베딩 비용 상세 분석 화면
- 로컬 임베딩 모델 지원
- 청크 cleanup 자동화

---

## 9. 선행 조건

Phase 10을 진행하려면 다음이 최소한 정리되어 있어야 한다.

- Phase 1: Document / Version / Node 구조 확정
- Phase 2: ACL 권한 모델 확정
- Phase 4~5: Published 버전 상태 전이 구현 완료
- Phase 7: 백그라운드 작업 시스템 구현 완료 (벡터화 파이프라인 실행 기반)
- Phase 7: Admin 검색 인덱싱 관리 화면 (벡터화 상태 관리 확장 대상)
- Phase 8: FTS 검색 레이어 구현 완료 (하이브리드 검색 통합 대상)
- pgvector PostgreSQL 확장 설치

---

## 10. 완료 기준

다음 조건을 만족하면 Phase 10 완료로 본다.

- Published 문서의 청크가 생성되고 embedding이 저장되어 있다
- 각 청크에 권한 메타데이터가 반영되어 있다
- 문서 Published 전이 시 자동 벡터화가 실행된다
- 벡터 유사도 검색이 동작한다 (`GET /search/documents?mode=hybrid` 또는 `mode=semantic`)
- Admin UI에서 벡터화 현황을 조회할 수 있다
- 수동 재색인 API가 동작한다
- 권한 없는 문서의 청크가 벡터 검색 결과에 포함되지 않는다

---

## 11. 권장 투입 순서

1. Task 10-1 벡터화 파이프라인 아키텍처 설계
2. Task 10-2 DocumentType별 청킹 전략 설계
3. Task 10-3 pgvector 스키마 및 인덱스 설계
4. Task 10-4 임베딩 모델 추상화 레이어 설계 및 구현
5. Task 10-5 청킹 파이프라인 구현
6. Task 10-6 벡터화 파이프라인 구현 (청킹 + 임베딩 + 저장)
7. Task 10-7 재색인 정책 구현
8. Task 10-8 권한 메타데이터 반영 구현
9. Task 10-9 하이브리드 검색 통합 (HybridSearchProvider)
10. Task 10-10 Admin 벡터화 상태 관리
11. Task 10-11 성능 검증 및 Phase 11 연계 포인트 설계

---

## 12. 최종 정리

Phase 10은 벡터화 파이프라인을 만드는 단계이지만, 본질적으로는  
**Mimir를 "문서 저장소"에서 "질의 가능한 지식 베이스"로 전환하는 전환점**이다.

이 단계에서 중요한 것은 임베딩 생성 자체가 아니라,

- 권한이 반영된 벡터가 생성되는가
- DocumentType별 최적 청킹으로 의미 단위가 보존되는가
- 문서 변경 시 벡터가 자동으로 갱신되는가
- Phase 11 RAG가 안전하게 활용할 수 있는 청크 구조인가

를 구조적으로 보장하는 것이다.

Phase 10의 산출물은 단순한 임베딩 테이블이 아니라  
**권한 인식 지식 벡터 베이스(Permission-Aware Knowledge Vector Base)**이어야 한다.
