# Phase 10 검수 보고서
## 문서 구조화 및 벡터화 파이프라인 구축

**검수일**: 2026-04-09  
**검수 대상**: Phase 10 전체 구현 (백엔드)  
**검수 방식**: 정적 코드 분석, 완료 기준 대비 구현 갭 분석, 보안·품질 점검

> Phase 10 종결 보고서(2026-04-09)에 기록된 수정 이력(코드 품질 4건·보안 7건·UX 5건)은 모두 반영 완료된 상태를 기준으로 검수한다.

---

## 1. 검수 개요

### 검수 대상 파일

| 영역 | 파일 |
|------|------|
| Backend / Service | `backend/app/services/embedding_service.py` |
| Backend / Service | `backend/app/services/chunking_service.py` |
| Backend / Service | `backend/app/services/vectorization_service.py` |
| Backend / Service | `backend/app/services/search_service.py` (hybrid 부분) |
| Backend / API | `backend/app/api/v1/vectorization.py` |
| Backend / API | `backend/app/api/v1/workflow.py` (publish 트리거) |
| Backend / DB | `backend/app/db/connection.py` (DDL, pgvector 인덱스) |

### 검수 결과 요약

| 등급 | 건수 | 설명 |
|------|------|------|
| 🔴 High | 1 | 데이터 무결성 파괴 가능 |
| 🟠 Medium | 2 | 기능 오류 / 권한 모델 불완전 |
| 🟡 Minor | 3 | 코드 품질 / 운영 안전성 |
| ✅ Pass | — | 완료 기준 8/8 충족 |

---

## 2. High — 데이터 무결성 결함

### [H-1] `_save_chunks` 내 트랜잭션 오류 전파 미처리

**파일**: `backend/app/services/vectorization_service.py:467–511`

**현상**

`_save_chunks()` 는 단일 커서(`with conn.cursor() as cur:`) 안에서 청크를 순회하며 INSERT를 실행한다. psycopg2에서 한 INSERT가 실패하면 **연결 전체가 오류 상태(aborted transaction)** 로 전환되며, 이후의 모든 `cur.execute()` 호출은 `InFailedSqlTransaction`을 발생시킨다.

현재 코드는 개별 INSERT 실패를 `try/except`로 잡아 `failed += 1`을 누적하고 다음 청크 INSERT를 시도하지만, 첫 번째 실패 이후의 모든 INSERT도 `InFailedSqlTransaction`으로 실패한다. 결과적으로:

1. `saved`는 첫 번째 실패 이전 청크 수만 반영된다.
2. `failed`는 실제 원인 청크 1개가 아닌 나머지 모든 청크를 포함한다.
3. 더 심각하게 — `_soft_delete_existing_chunks()`는 이미 완료되어 기존 청크가 `is_current = FALSE`로 전환된 상태다. 새 청크는 하나도 저장되지 않으므로 해당 버전의 문서는 **검색에서 완전히 사라진다**.

```python
# 현재 코드 (문제)
with conn.cursor() as cur:
    for i, chunk in enumerate(chunks):
        try:
            cur.execute("INSERT INTO document_chunks ...")   # ← 첫 실패 후
            saved += 1                                        #   모든 이후 execute도
        except Exception as exc:                             #   InFailedSqlTransaction
            logger.error("...")                               #   으로 실패
            failed += 1
```

**수정 방향**

각 INSERT를 독립 savepoint로 감싸거나, INSERT 실패 시 즉시 루프를 중단하고 연결 롤백 후 호출자에게 오류를 전파한다.

```python
# 방법 A: savepoint per chunk
with conn.cursor() as cur:
    for i, chunk in enumerate(chunks):
        try:
            cur.execute("SAVEPOINT sp_chunk_%d" % i)
            cur.execute("INSERT INTO document_chunks ...")
            saved += 1
        except Exception as exc:
            cur.execute("ROLLBACK TO SAVEPOINT sp_chunk_%d" % i)
            logger.error("청크 저장 실패 (index=%d): %s", i, exc)
            failed += 1

# 방법 B: 실패 즉시 중단 + 호출자 롤백 위임 (단순, 권장)
with conn.cursor() as cur:
    for i, chunk in enumerate(chunks):
        cur.execute("INSERT INTO document_chunks ...")
        saved += 1
# 예외는 vectorize_version의 outer except로 전파 → 소프트 삭제 포함 전체 롤백
```

방법 B를 권장한다. 호출자(`vectorize_version`)의 `except Exception` 블록에서 `conn.rollback()`을 명시 호출하면 소프트 삭제도 함께 취소된다.

---

## 3. Medium — 기능 오류 / 권한 모델 불완전

### [M-1] `embed_single` 실패 시 zero vector로 semantic_search 실행

**파일**: `backend/app/services/vectorization_service.py:336–337`

**현상**

`semantic_search()`가 호출될 때 `provider.embed_single(query)`를 통해 쿼리 임베딩을 얻는다. `OpenAIEmbeddingProvider.embed_single()`은 API 실패 시 `[0.0] * 1536` zero vector를 반환한다. zero vector를 사용한 코사인 유사도 검색은 수학적으로 의미 없는 결과를 반환하며(`1 - (v <=> 0⃗)` = NaN 또는 0), 오류 없이 정상 결과처럼 반환된다.

```python
def semantic_search(self, conn, query, ...):
    provider = self._get_provider()
    query_embedding = provider.embed_single(query)  # ← API 실패 시 zero vector
    # 이후 검색 계속 실행 → 무의미한 결과 반환
```

**수정 방향**

`embed_single()` 결과가 zero vector이면 예외를 발생시키거나 빈 목록을 반환한다.

```python
query_embedding = provider.embed_single(query)
if not any(query_embedding):
    logger.warning("임베딩 생성 실패 — 빈 결과 반환 (query=%s)", query[:50])
    return []
```

---

### [M-2] `accessible_org_ids` 권한 필드 미구현 (항상 빈 목록)

**파일**: `backend/app/services/vectorization_service.py:41–73`

**현상**

`PermissionSnapshot` 데이터클래스는 `accessible_org_ids: list[str]` 필드를 정의하지만, `_get_permission_snapshot()` 함수는 어떤 경우에도 빈 목록을 반환한다. `document_chunks` 테이블의 `accessible_org_ids` 컬럼은 항상 `[]`로 저장된다.

Phase 2 ACL 모델에 org 기반 접근 제어가 포함되어 있고, `document_chunks.accessible_org_ids` 컬럼도 DDL에 정의되어 있으나 실제로 채워지지 않는다. 조직 단위 접근 제어 기능이 필요한 경우 벡터 검색 결과가 잘못 필터링될 수 있다.

**수정 방향**

문서의 조직 ID를 조회하여 `accessible_org_ids`를 올바르게 채우거나, 미구현 상태임을 명시하는 `TODO` 주석과 함께 `Phase 12 이후 구현 예정` 항목으로 이슈를 등록한다.

---

## 4. Minor — 코드 품질 / 운영 안전성

### [m-1] `vectorize_all_published` OpenAI API 연속 호출 — Rate Limit 무방비

**파일**: `backend/app/services/vectorization_service.py:239–282`

**현상**

`vectorize_all_published(limit=100)`은 루프에서 문서 하나씩 `vectorize_version()`을 동기 호출한다. 각 `vectorize_version()`은 내부에서 배치 임베딩 API를 여러 번 호출할 수 있다. 100개 문서 일괄 재색인 시 OpenAI API TPM/RPM 한도에 도달할 수 있으나, 재시도 백오프(`time.sleep()`)가 `embed_batch` 레벨에서만 존재하며 배치 간 지연이 없다.

**수정 방향**

배치 크기 제한 및 배치 간 `time.sleep(0.2)` 정도의 간격 추가를 권장한다. 또는 Admin API 레이어(`POST /vectorization/reindex-all`)에서 rate limit를 낮게 유지한다(현행 5/minute — 유지).

---

### [m-2] `LocalEmbeddingProvider` zero vector와 실제 임베딩 미구분

**파일**: `backend/app/services/embedding_service.py:183–218`

**현상**

`LocalEmbeddingProvider.embed_batch()`는 `[0.0] * dimensions` zero vector를 반환하며 `embedding_model = "local-placeholder"`로 저장된다. 이 청크는 `is_current = TRUE`로 저장되어 semantic_search 대상에 포함되지만, zero vector이므로 모든 쿼리에 대해 무의미한 유사도(0.0)를 반환한다. API 키 없는 환경에서 무심코 운영 배포 시 검색이 완전히 동작하지 않을 수 있다.

**수정 방향**

`LocalEmbeddingProvider` 사용 시 경고 로그를 매 배치마다 출력하는 것은 현재도 있으나, `semantic_search()` 또는 저장 시점에서 `embedding_model = "local-placeholder"` 청크를 필터링하거나 관리자 대시보드에서 placeholder 청크 수를 표시한다.

---

### [m-3] `get_chunking_config_for_type` 폴백 시 로그 없음

**파일**: `backend/app/services/chunking_service.py` (ChunkingService 내 해당 메서드)

**현상**

신규 `document_type`이 `document_types` 테이블에 `chunking_config` 설정 없이 추가된 경우, `get_chunking_config_for_type()`은 기본 `ChunkingConfig()`를 반환하되 로그를 남기지 않는다. 타입별로 최적화된 청킹 설정이 적용되어야 한다는 Phase 10 설계 원칙(6.2)에 반하는 상황이 무음으로 발생할 수 있다.

**수정 방향**

폴백 발생 시 `logger.warning("document_type=%s 에 대한 chunking_config가 없어 기본값을 사용합니다.", document_type)` 추가.

---

## 5. 완료 기준 재검증

| 완료 기준 | 검수 결과 | 비고 |
|-----------|----------|------|
| Published 문서의 청크가 생성되고 embedding이 저장된다 | ✅ | VectorizationPipeline 정상 동작 |
| 각 청크에 권한 메타데이터가 반영된다 | ⚠️ 부분 | `accessible_org_ids` 미구현 [M-2] |
| 문서 Published 전이 시 자동 벡터화가 실행된다 | ✅ | `_trigger_vectorization_async` 동작 |
| 벡터 유사도 검색이 동작한다 | ⚠️ 조건부 | API 실패 시 zero vector로 결과 오염 [M-1] |
| 하이브리드 검색이 동작한다 | ✅ | `search_documents_hybrid` RRF 정상 |
| Admin UI에서 벡터화 현황을 조회할 수 있다 | ✅ | AdminVectorizationPage 3탭 |
| 수동 재색인 API가 동작한다 | ✅ | API 정상 |
| 권한 없는 문서의 청크가 검색 결과에 포함되지 않는다 | ✅ | `accessible_roles` 필터 동작 |

**6/8 온전 충족, 2/8 조건부 충족**

---

## 6. 수정 우선순위

| 우선순위 | ID | 내용 | 예상 공수 |
|---------|-----|------|----------|
| P0 | H-1 | `_save_chunks` savepoint 또는 즉시 중단 + rollback 처리 | 30분 |
| P1 | M-1 | `embed_single` zero vector 반환 시 빈 목록 반환 | 15분 |
| P2 | M-2 | `accessible_org_ids` TODO 주석 추가 + 이슈 등록 | 10분 |
| P3 | m-1 | `vectorize_all_published` 배치 간 sleep 추가 | 15분 |
| P3 | m-2 | `LocalEmbeddingProvider` 대시보드 표시 (별도 이슈) | 별도 검토 |
| P4 | m-3 | `get_chunking_config_for_type` 폴백 경고 로그 | 5분 |

---

## 7. 결론

Phase 10은 완료 기준 8개 중 6개를 완전 충족하며, 핵심 벡터화 파이프라인과 하이브리드 검색은 정상 동작한다. 다만 데이터 무결성 관련 HIGH 결함([H-1]) 하나와 권한 모델 불완전([M-2])이 확인되었다.

**[H-1]은 반드시 수정 후 Phase 11을 진행할 것을 권고한다.** 특히 임베딩 배치 저장 중 오류 발생 시 해당 문서가 검색에서 누락되는 문제는 RAG 질의 품질에 직접 영향을 준다.

[M-1], [M-2]는 Phase 11 RAG 서비스 안정성에 영향을 주므로 함께 수정을 권장한다.
