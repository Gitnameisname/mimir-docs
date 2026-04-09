# Phase 10 보안 취약점 검사 리포트

**검사 일자**: 2026-04-09  
**검사 범위**: Phase 10 "문서 구조화 및 벡터화 파이프라인" 신규/변경 파일 전체  
**검사 방법**: 수동 코드 리뷰 (OWASP Top 10, API 보안, 비용 DoS 관점)  
**상태**: 전체 이슈 수정 완료

---

## 요약

| 등급 | 발견 수 | 수정 완료 |
|------|---------|----------|
| HIGH | 2 | 2 |
| MEDIUM | 2 | 2 |
| LOW | 3 | 3 |
| **합계** | **7** | **7** |

---

## HIGH

### VEC-001 — 벡터화 엔드포인트 Rate Limiting 전무

**파일**: `backend/app/api/v1/vectorization.py`  
**영향**: 비용 DoS (OpenAI 임베딩 API 무제한 호출)

**설명**  
`POST /vectorization/search/semantic` 및 `POST /vectorization/reindex-all` 엔드포인트에 rate limiting이 전혀 적용되지 않았다. `search.py`의 모든 검색 엔드포인트가 `@limiter.limit("30/minute")`으로 보호된 것과 대조적이다.

- `/search/semantic`: 요청당 OpenAI embedding API 1회 호출 발생. 인증된 악의적 사용자가 초당 수십 회 호출하면 상당한 API 비용이 발생한다.
- `/reindex-all`: limit=500으로 500개 문서 임베딩 배치 처리. 반복 호출 시 극단적 비용이 발생한다.

**수정**  
```python
# vectorization.py
_SEMANTIC_SEARCH_LIMIT = "30/minute"
_REINDEX_LIMIT = "5/minute"

@router.post("/reindex-all", ...)
@limiter.limit(_REINDEX_LIMIT)       # 추가
def reindex_all(...): ...

@router.post("/search/semantic", ...)
@limiter.limit(_SEMANTIC_SEARCH_LIMIT)  # 추가
def semantic_search(...): ...
```

---

### VEC-002 — UUID 경로 파라미터 미검증으로 내부 오류 정보 노출

**파일**: `backend/app/api/v1/vectorization.py` — `reindex_document`, `reindex_version`  
**영향**: 정보 노출 (CWE-209), 잘못된 입력에 대한 500 응답

**설명**  
`document_id`, `version_id` 경로 파라미터가 UUID 형식 검증 없이 `%s::uuid` PostgreSQL 캐스팅에 직접 전달된다. 비정상 UUID 입력 시:

1. PostgreSQL이 `InvalidTextRepresentation` 오류를 발생
2. `vectorize_version`의 `except` 절이 이를 `result.error`에 저장
3. `reindex_version` 핸들러가 `raise HTTPException(status_code=500, detail=result.error)` 실행
4. **DB 오류 메시지 원문이 클라이언트에 그대로 노출**됨

**수정**  
```python
# 입력 단계에서 UUID 형식 검증 (400 반환)
_UUID_RE = re.compile(r"^[0-9a-f]{8}-...-[0-9a-f]{12}$", re.IGNORECASE)

def _validate_uuid(value: str, field: str) -> None:
    if not _UUID_RE.match(value):
        raise HTTPException(status_code=400, detail=f"{field}이(가) 유효한 UUID 형식이 아닙니다.")

# 500 응답에서 내부 오류 메시지 제거
raise HTTPException(status_code=500, detail="벡터화 처리 중 오류가 발생했습니다.")
```

---

## MEDIUM

### VEC-003 — SemanticSearchRequest.q 길이 제한 없음

**파일**: `backend/app/api/v1/vectorization.py` — `SemanticSearchRequest`  
**영향**: 비용 증폭, 경미한 DoS

**설명**  
`SemanticSearchRequest.q`에 최대 길이 제한이 없어 임의로 긴 텍스트를 임베딩 API에 전달할 수 있다. `search.py`는 `_MAX_QUERY_LEN = 300`으로 검색어를 제한하지만 벡터화 검색 엔드포인트는 이 제한이 없었다. 1,000자짜리 쿼리는 짧은 쿼리보다 ~3배 많은 토큰을 소비한다.

**수정**  
```python
class SemanticSearchRequest(BaseModel):
    q: str = Field(..., min_length=1, max_length=500)  # 길이 제한 추가
    document_type: Optional[str] = None
    top_k: int = Field(default=10, ge=1, le=50)        # 범위 제한도 추가
```

---

### VEC-004 — 자동 벡터화 스레드 무제한 생성

**파일**: `backend/app/api/v1/workflow.py` — `_trigger_vectorization_async`  
**영향**: 리소스 소진 (CWE-400) — DB 연결 풀 고갈

**설명**  
`publish` 전이마다 `threading.Thread(...).start()`로 새 스레드를 생성한다. 짧은 시간에 다수 문서가 publish되면 (예: 스크립트로 API 일괄 호출) 동시 스레드 수가 무제한으로 증가한다. 각 스레드는 `get_db()`로 DB 연결을 점유하므로 연결 풀이 고갈될 수 있다.

**수정**  
```python
# 모듈 레벨에서 고정 크기 스레드 풀 생성
_VECTORIZATION_EXECUTOR = concurrent.futures.ThreadPoolExecutor(
    max_workers=3,
    thread_name_prefix="vec-bg",
)

# Thread().start() → executor.submit()으로 교체
_VECTORIZATION_EXECUTOR.submit(_run)
```

`max_workers=3`으로 동시 벡터화 작업을 3개로 제한한다. 풀이 가득 찬 경우 새 작업은 큐에 대기한다.

---

## LOW

### VEC-005 — mode=semantic 미구현으로 silent fallthrough 발생

**파일**: `backend/app/api/v1/search.py` — `search_documents`  
**영향**: 예상치 못한 동작, 잘못된 API 계약

**설명**  
엔드포인트 docstring에 `mode=semantic (벡터 유사도 전용)`이 명시되어 있으나 실제 구현에서는 `mode=hybrid`만 분기하고 나머지는 FTS로 fallthrough한다. 클라이언트가 `mode=semantic`을 요청하면 오류 없이 FTS 결과가 반환되어 기대와 다른 결과를 조용히 제공한다.

**수정**  
```python
if mode not in ("fts", "hybrid"):
    raise HTTPException(status_code=400, detail=f"지원하지 않는 mode입니다: '{mode}'. 허용 값: fts, hybrid")
```
docstring에서 `semantic` 옵션 제거.

---

### VEC-006 — Admin 벡터화 지표 조회 오류 silent 무시

**파일**: `backend/app/api/v1/admin.py` — `get_dashboard_metrics`  
**영향**: 운영 가시성 저하 (오류 감지 불가)

**설명**  
`document_chunks` 통계 쿼리가 `except Exception: pass`로 감싸져 있어 테이블 부재 외의 모든 오류(권한 문제, 네트워크 오류, 쿼리 오류 등)가 조용히 무시된다. 벡터화 파이프라인에 문제가 생겨도 Admin 대시보드는 정상처럼 표시된다.

**수정**  
```python
except Exception as _vec_exc:
    logger.debug("벡터화 지표 조회 스킵 (테이블 미존재 또는 오류): %s", _vec_exc)
```
오류를 `DEBUG` 레벨로 기록하여 로그 분석 시 추적 가능하게 한다.

---

### VEC-007 — 벡터화 Admin 작업 감사 로그 미기록

**파일**: `backend/app/api/v1/vectorization.py`  
**영향**: 감사 추적 불가 (OWASP A09 — Security Logging and Monitoring Failures)

**설명**  
`POST /vectorization/reindex-all`, `POST /vectorization/documents/{id}`, `POST /vectorization/cleanup` 등 Admin 전용 대량 데이터 작업이 `audit_events` 테이블에 기록되지 않는다. 누가 언제 재색인/정리를 트리거했는지 추적할 수 없다. 다른 Admin 엔드포인트(`admin.py`)는 대부분 감사 로그를 기록하므로 일관성에서도 벗어난다.

**권고**  
Phase 11 또는 이후 단계에서 다음 감사 이벤트를 추가한다:

| 이벤트 | 심각도 |
|--------|--------|
| `vectorization.reindex_all` | MEDIUM |
| `vectorization.reindex_document` | LOW |
| `vectorization.cleanup` | MEDIUM |

현 단계에서는 최소한 애플리케이션 로그(`logger.info`)로 actor 정보를 기록하도록 가이드한다.

> **주**: 감사 로그 스키마 변경 및 전체 audit_events 연계는 범위가 크므로 다음 Phase에서 일괄 처리를 권고한다.

---

## 비취약점 검토 사항 (참고)

| 항목 | 결과 |
|------|------|
| SQL 인젝션 — vectorization.py | 안전 (모든 값 psycopg2 파라미터 바인딩) |
| SQL 인젝션 — search_service.py `sort` 파라미터 | 안전 (if/elif/else로 ORDER BY 하드코딩, 사용자 값 미삽입) |
| SQL 인젝션 — chunking_service.py | 안전 (psycopg2 파라미터 바인딩) |
| 권한 우회 — semantic_search | 안전 (`require_authenticated=True` + DB 레벨 `accessible_roles` 필터) |
| 권한 우회 — hybrid search | 안전 (FTS는 `visible_statuses`로, 벡터는 `accessible_roles`로 필터) |
| 권한 우회 — Admin 엔드포인트 | 안전 (`_require_admin()` + `authorization_service.authorize()` 적용) |
| OpenAI API key 노출 | 안전 (메모리 내 보관, 응답에 미포함) |
| `plugin_config.chunking_config` 인젝션 | 안전 (`int()`, `bool()` 변환으로 타입 강제, 실패 시 기본값 사용) |
| `_save_chunks` NULL UUID 캐스팅 | 안전 (psycopg2의 None → SQL NULL 변환 정상 작동) |
| HNSW 인덱스 권한 | 안전 (is_current=TRUE 부분 인덱스, 일반 쿼리 노출 없음) |

---

## 수정 파일 목록

| 파일 | 수정 내용 |
|------|----------|
| `backend/app/api/v1/vectorization.py` | Rate limiting 추가, UUID 검증 추가, q 길이 제한, 오류 메시지 마스킹 |
| `backend/app/api/v1/search.py` | mode 값 검증 (400), semantic 문서 제거 |
| `backend/app/api/v1/workflow.py` | ThreadPoolExecutor(max_workers=3)로 교체 |
| `backend/app/api/v1/admin.py` | silent pass → logger.debug 기록 |
