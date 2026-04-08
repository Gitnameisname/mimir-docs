# Task 13-8. 성능 최적화 및 부하 테스트

## 1. 작업 목적

서비스 운영 전 성능 병목을 분석하고 SLO 목표를 달성하도록 최적화한다.

이 작업의 목표는 다음과 같다.

- 핵심 API 엔드포인트 성능 측정
- DB 쿼리 최적화 (N+1, 인덱스 누락)
- 캐싱 전략 구현 (자주 읽는 데이터)
- 부하 테스트로 SLO 달성 확인 (50 동시 사용자)
- 병목 식별 및 수정

---

## 2. 작업 범위

### 포함 범위

- API latency 측정 및 병목 분석
- DB 쿼리 최적화
- 애플리케이션 레벨 캐싱 구현
- Locust 부하 테스트 실행
- 최적화 결과 보고서

### 제외 범위

- 인프라 스케일 업/아웃 (운영 환경 별도)
- CDN 설정 (프론트엔드 정적 파일)

---

## 3. 주요 구현 대상

### 3-1. 성능 목표 (SLO)

| 엔드포인트 | P95 목표 |
|----------|---------|
| GET /api/documents | < 100ms |
| GET /api/documents/{id} | < 150ms |
| POST /api/search | < 300ms |
| POST /api/rag/query (TTFT) | < 2,000ms |
| Admin API 전반 | < 300ms |

---

### 3-2. DB 쿼리 최적화

#### N+1 쿼리 탐지 및 수정

```python
# 탐지: SQLAlchemy 이벤트 리스너로 쿼리 카운트
from sqlalchemy import event

query_count = 0

@event.listens_for(Engine, "before_cursor_execute")
def count_queries(conn, cursor, statement, parameters, context, executemany):
    global query_count
    query_count += 1

# 수정: eager loading 적용
# 기존 (N+1)
documents = db.query(Document).all()
for doc in documents:
    _ = doc.current_version  # N번 추가 쿼리 발생

# 수정 (1+1)
documents = db.query(Document).options(
    joinedload(Document.current_version)
).all()
```

#### 인덱스 누락 점검

```sql
-- 슬로우 쿼리 조회 (pg_stat_statements)
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 100
ORDER BY mean_exec_time DESC
LIMIT 20;

-- 인덱스 미사용 테이블 점검
SELECT schemaname, tablename, attname
FROM pg_stats
WHERE n_distinct > 1000
  AND tablename NOT IN (
    SELECT tablename FROM pg_indexes WHERE tablename = pg_stats.tablename
  );
```

---

### 3-3. 캐싱 전략

#### 캐싱 대상 및 TTL

| 데이터 | TTL | 캐싱 방식 |
|-------|-----|---------|
| DocumentType 설정 | 5분 | 메모리 캐시 |
| 사용자 권한 (ACL) | 1분 | Redis (선택) |
| 문서 목록 (필터별) | 30초 | Redis (선택) |
| Static 리소스 | 영구 | CDN/HTTP 캐시 헤더 |

#### 인메모리 캐시 구현 (간단한 경우)

```python
from functools import lru_cache
from datetime import datetime, timedelta

_cache: dict = {}
_cache_ttl: dict = {}

def cached(ttl_seconds: int):
    def decorator(func):
        def wrapper(*args, **kwargs):
            key = f"{func.__name__}:{args}:{kwargs}"
            if key in _cache and datetime.now() < _cache_ttl[key]:
                return _cache[key]
            result = func(*args, **kwargs)
            _cache[key] = result
            _cache_ttl[key] = datetime.now() + timedelta(seconds=ttl_seconds)
            return result
        return wrapper
    return decorator

@cached(ttl_seconds=300)
def get_document_type_config(document_type: str) -> dict:
    return DocumentTypeRepository.get_config(document_type)
```

---

### 3-4. 부하 테스트 결과 기준

Locust 50 동시 사용자 기준:

| 지표 | 목표 | 측정 방법 |
|------|------|---------|
| 문서 조회 P95 | < 200ms | Locust 리포트 |
| 검색 P95 | < 500ms | Locust 리포트 |
| 오류율 | < 0.1% | Locust 통계 |
| 최대 RPS | 50 RPS | Locust 통계 |

---

### 3-5. 최적화 체크리스트

```
DB 쿼리:
  □ N+1 쿼리 제거
  □ 슬로우 쿼리 인덱스 추가
  □ 불필요한 SELECT * → 필요 컬럼만 조회

애플리케이션:
  □ DocumentType 설정 캐싱 (5분 TTL)
  □ 비동기 처리 (async/await) 누락 없음
  □ 연결 풀 설정 (DB connection pool)

API:
  □ 페이지네이션 강제 (limit 없는 쿼리 금지)
  □ 응답 압축 (gzip) 활성화
  □ 불필요한 응답 필드 제거
```

---

## 4. 산출물

1. 성능 측정 결과 보고서 (최적화 전/후 비교)
2. DB 쿼리 최적화 커밋 목록
3. 캐싱 구현 코드
4. 부하 테스트 결과 보고서 (Locust)
5. SLO 달성 여부 확인서

---

## 5. 완료 기준

- 핵심 API P95 latency가 SLO 목표 내에 있다
- N+1 쿼리가 없다
- 50 동시 사용자 부하 테스트에서 오류율 < 0.1%
- DocumentType 설정이 캐싱된다

---

## 6. Codex 작업 지침

- 최적화 전 반드시 측정 후 수정하며, 측정 없이 추측으로 최적화하지 않는다
- 캐싱 TTL은 설정으로 조정 가능하게 하여 데이터 정합성 요구사항에 따라 조정한다
- 부하 테스트는 스테이징 환경에서 실행하며 운영 DB를 대상으로 절대 실행하지 않는다
- DB 연결 풀 크기는 환경 변수로 설정하며 기본값은 10으로 한다
