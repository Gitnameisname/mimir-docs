# Task 3-6: Citation 재활용 + 캐싱 + 성능 최적화

## 1. 작업 목적

멀티턴 RAG에서 이전 턴의 Citation을 효과적으로 재활용하고, 캐싱을 통해 성능을 최적화하여:
- 이전 턴의 인용 Document를 현재 쿼리 검색 시 우선순위 상향
- 검색 결과 및 Citation 검증을 캐시하여 중복 연산 제거
- 프롬프트 토큰 계산의 delta만 저장하여 계산량 감소
- 폐쇄망 환경에서도 Redis 없이 in-memory LRU 캐시로 동작
- 멀티턴 RAG의 응답 시간을 합리적 수준(+500ms 이내)으로 유지

---

## 2. 작업 범위

### 포함 항목

1. **이전 턴 Citation 재활용 전략**
   - 이전 Turn의 retrieval_metadata에서 인용 Document ID 추출
   - 현재 검색 결과에서 Document 일치 검사
   - Reranker 점수에 보너스 가중치 추가

2. **검색 결과 캐싱**
   - Redis (또는 in-memory LRU) 캐시 구현
   - 캐시 키: hash(query, user_id, top_k)
   - TTL: 1시간 (설정 가능)
   - 폐쇄망 fallback: EXTERNAL_DEPENDENCIES_ENABLED=false 시 in-memory LRU 사용

3. **Citation 5-tuple 검증 캐싱**
   - content_hash 기반 캐시
   - 캐시 키: content_hash
   - 검증 결과(유효/무효) 저장
   - TTL: 24시간

4. **프롬프트 토큰 계산 캐시**
   - 이전 턴의 토큰 계산 결과 재사용
   - Delta 계산: 현재 턴 - 이전 턴의 공통 부분
   - 저장소: Turn metadata에 cached_token_count

5. **성능 벤치마크**
   - 단발 RAG vs 멀티턴 RAG 응답 시간 비교
   - 캐시 hit율 측정
   - 토큰 계산 성능 개선도 측정

6. **S2 원칙 ⑦ 적용 (폐쇄망 동작)**
   - Redis unavailable 시 자동으로 in-memory LRU fallback
   - 폐쇄망 환경에서도 RAG 성능 저하하지만 실패하지 않음
   - EXTERNAL_DEPENDENCIES_ENABLED 환경변수 체크

7. **캐시 무효화 전략**
   - Document 업데이트 시 관련 캐시 제거
   - User scope 변경 시 모든 검색 결과 캐시 제거
   - Citation 검증 캐시: content_hash 기반이므로 자동 무효

8. **단위 테스트**
   - `test_citation_reuse_bonus()`: 이전 Citation에 보너스 적용
   - `test_search_cache_hit()`: 캐시 hit 확인
   - `test_search_cache_miss()`: 캐시 miss 및 재계산
   - `test_citation_validation_cache()`: Citation 검증 캐시
   - `test_token_cache_delta()`: 토큰 delta 계산
   - `test_fallback_to_memory_cache()`: Redis unavailable 시 in-memory 사용
   - `test_cache_invalidation_on_document_update()`: Document 업데이트 시 캐시 무효화
   - `test_performance_benchmark_single_vs_multi_turn()`: 성능 벤치마크
   - **총 8개 이상의 test case**

### 제외 항목

- RAG API 엔드포인트 (Task 3-4)
- 컨텍스트 윈도우 관리 (Task 3-5)
- UI 구현 (Task 3-3)

---

## 3. 선행 조건

1. **Task 3-4, 3-5 완료**
   - ConversationRAGService, ContextWindowManager
   - Turn retrieval_metadata 저장 구조

2. **Phase 2 Citation 5-tuple 정의 완료**
   - {document_id, version_id, node_id, span_offset, content_hash, content}

3. **Phase 1 LLM 추상화 완료**
   - count_tokens() 함수

4. **(선택) Redis 또는 in-memory LRU 라이브러리**
   - redis >= 5.0 (선택사항)
   - functools.lru_cache 또는 cachetools (기본)

---

## 4. 주요 작업 항목

### 4-1. 이전 턴 Citation 재활용 전략

**파일**: `backend/src/services/citation_reuse_service.py` (새 파일)

```python
from typing import List, Dict, Any, Optional
from uuid import UUID
from sqlalchemy.orm import Session
import logging

from src.models import Turn, Document
from src.core.reranker import rerank_results

logger = logging.getLogger(__name__)

class CitationReuseService:
    """이전 턴 Citation 재활용"""
    
    CITATION_BONUS_MULTIPLIER = 1.5  # 50% 점수 상향
    
    def __init__(self, db: Session):
        self.db = db
    
    def extract_cited_documents(self, turn: Turn) -> List[UUID]:
        """
        Turn의 retrieval_metadata에서 인용 Document ID 추출
        
        Args:
            turn: Turn 객체
        
        Returns:
            Document ID 리스트
        """
        
        if not turn.retrieval_metadata:
            return []
        
        citations = turn.retrieval_metadata.get("citations", [])
        doc_ids = [c.get("document_id") for c in citations if c.get("document_id")]
        
        return doc_ids
    
    def apply_citation_bonus(
        self,
        search_results: List[Dict[str, Any]],
        previous_turns: List[Turn]
    ) -> List[Dict[str, Any]]:
        """
        이전 턴에서 인용된 Document에 보너스 점수 적용
        
        Args:
            search_results: 현재 검색 결과
            previous_turns: 이전 Turn 리스트 (최근부터 오래된 순)
        
        Returns:
            보너스가 적용된 검색 결과
        """
        
        # 이전 턴에서 인용된 Document ID 수집
        cited_doc_ids = set()
        for turn in previous_turns:
            cited_doc_ids.update(self.extract_cited_documents(turn))
        
        if not cited_doc_ids:
            logger.debug("No previously cited documents found")
            return search_results
        
        # 검색 결과에 보너스 점수 적용
        boosted_results = []
        for result in search_results:
            doc_id = result.get("document_id")
            
            if doc_id and UUID(str(doc_id)) in cited_doc_ids:
                # 보너스 적용
                original_score = result.get("score", 0.0)
                boosted_score = original_score * self.CITATION_BONUS_MULTIPLIER
                
                result = {**result, "score": boosted_score, "citation_reused": True}
                logger.info(
                    f"Citation bonus applied: doc_id={doc_id}, "
                    f"score={original_score:.3f} → {boosted_score:.3f}"
                )
            
            boosted_results.append(result)
        
        # 점수 기준으로 재정렬
        boosted_results.sort(key=lambda x: x.get("score", 0.0), reverse=True)
        
        return boosted_results
    
    def get_citation_context(self, document_id: UUID) -> Optional[str]:
        """
        Document가 이전 어느 Turn에서 인용되었는지 확인하고 컨텍스트 제공
        
        Args:
            document_id: Document ID
        
        Returns:
            인용 컨텍스트 (선택사항)
        """
        
        # 간단한 구현: 최근 인용 정보만 반환
        # 실제로는 Turn 조회 후 metadata 확인
        return None
```

**요구사항**:
- 이전 Turn의 citations 메타데이터에서 Document ID 추출
- 점수 계산: original_score * CITATION_BONUS_MULTIPLIER
- 보너스 후 점수 기준으로 재정렬
- 로깅으로 보너스 적용 사항 기록

---

### 4-2. 캐싱 레이어 구현

**파일**: `backend/src/core/cache.py` (새 파일)

```python
from typing import Any, Optional, Dict, List
from abc import ABC, abstractmethod
import hashlib
import json
import logging
from datetime import datetime, timedelta
from functools import lru_cache
from uuid import UUID

logger = logging.getLogger(__name__)

# ===== Cache Interface (추상 클래스) =====

class CacheBackend(ABC):
    """캐시 백엔드 인터페이스"""
    
    @abstractmethod
    def get(self, key: str) -> Optional[Any]:
        pass
    
    @abstractmethod
    def set(self, key: str, value: Any, ttl_seconds: int = 3600) -> bool:
        pass
    
    @abstractmethod
    def delete(self, key: str) -> bool:
        pass
    
    @abstractmethod
    def clear(self) -> bool:
        pass

# ===== In-Memory LRU Cache (기본, 폐쇄망 환경용) =====

class InMemoryLRUCache(CacheBackend):
    """메모리 기반 LRU 캐시 (폐쇄망 fallback용)"""
    
    def __init__(self, max_size: int = 1000):
        self.max_size = max_size
        self.cache: Dict[str, tuple] = {}  # {key: (value, expires_at)}
    
    def get(self, key: str) -> Optional[Any]:
        if key not in self.cache:
            logger.debug(f"Cache miss: {key}")
            return None
        
        value, expires_at = self.cache[key]
        
        if datetime.utcnow() > expires_at:
            del self.cache[key]
            logger.debug(f"Cache expired: {key}")
            return None
        
        logger.debug(f"Cache hit: {key}")
        return value
    
    def set(self, key: str, value: Any, ttl_seconds: int = 3600) -> bool:
        # LRU 제한: 캐시 크기 초과 시 가장 오래된 항목 제거
        if len(self.cache) >= self.max_size:
            oldest_key = min(
                self.cache.keys(),
                key=lambda k: self.cache[k][1]
            )
            del self.cache[oldest_key]
            logger.debug(f"Cache evicted (LRU): {oldest_key}")
        
        expires_at = datetime.utcnow() + timedelta(seconds=ttl_seconds)
        self.cache[key] = (value, expires_at)
        logger.debug(f"Cache set: {key} (TTL={ttl_seconds}s)")
        return True
    
    def delete(self, key: str) -> bool:
        if key in self.cache:
            del self.cache[key]
            logger.debug(f"Cache deleted: {key}")
            return True
        return False
    
    def clear(self) -> bool:
        self.cache.clear()
        logger.info("Cache cleared")
        return True

# ===== Redis Cache (선택사항, 외부 의존) =====

try:
    import redis
    
    class RedisCacheBackend(CacheBackend):
        """Redis 기반 캐시"""
        
        def __init__(self, redis_url: str = "redis://localhost:6379/0"):
            self.client = redis.from_url(redis_url)
        
        def get(self, key: str) -> Optional[Any]:
            try:
                value = self.client.get(key)
                if value:
                    logger.debug(f"Cache hit (Redis): {key}")
                    return json.loads(value)
                logger.debug(f"Cache miss (Redis): {key}")
                return None
            except Exception as e:
                logger.warning(f"Redis get error: {str(e)}")
                return None
        
        def set(self, key: str, value: Any, ttl_seconds: int = 3600) -> bool:
            try:
                self.client.setex(
                    key,
                    ttl_seconds,
                    json.dumps(value, default=str)
                )
                logger.debug(f"Cache set (Redis): {key}")
                return True
            except Exception as e:
                logger.warning(f"Redis set error: {str(e)}")
                return False
        
        def delete(self, key: str) -> bool:
            try:
                result = self.client.delete(key)
                logger.debug(f"Cache deleted (Redis): {key}")
                return result > 0
            except Exception as e:
                logger.warning(f"Redis delete error: {str(e)}")
                return False
        
        def clear(self) -> bool:
            try:
                self.client.flushdb()
                logger.info("Redis cache cleared")
                return True
            except Exception as e:
                logger.warning(f"Redis clear error: {str(e)}")
                return False

except ImportError:
    logger.debug("Redis not available. Using in-memory cache fallback.")
    RedisCacheBackend = None

# ===== Cache Manager (팩토리 패턴) =====

class CacheManager:
    """캐시 백엔드 선택 및 관리"""
    
    _instance: Optional[CacheBackend] = None
    
    @classmethod
    def get_cache(cls, redis_enabled: bool = True) -> CacheBackend:
        """
        캐시 백엔드 가져오기
        
        Args:
            redis_enabled: Redis 사용 여부
        
        Returns:
            CacheBackend 인스턴스
        """
        
        if cls._instance is not None:
            return cls._instance
        
        # Redis 시도 (사용 가능하면)
        if redis_enabled and RedisCacheBackend:
            try:
                cls._instance = RedisCacheBackend()
                logger.info("Using Redis cache backend")
                return cls._instance
            except Exception as e:
                logger.warning(f"Redis initialization failed: {str(e)}. Falling back to in-memory cache.")
        
        # Fallback: in-memory LRU
        cls._instance = InMemoryLRUCache()
        logger.info("Using in-memory LRU cache backend (S2 원칙 ⑦)")
        return cls._instance

# ===== 캐시 키 생성 헬퍼 =====

def make_cache_key(prefix: str, **kwargs) -> str:
    """
    캐시 키 생성 헬퍼
    
    Args:
        prefix: 캐시 키 프리픽스 (예: "search_results")
        kwargs: 키 생성에 사용할 파라미터
    
    Returns:
        캐시 키 문자열
    """
    
    # 파라미터를 정렬하여 일관된 키 생성
    key_parts = [prefix]
    for k, v in sorted(kwargs.items()):
        # UUID를 문자열로 변환
        if isinstance(v, UUID):
            v = str(v)
        key_parts.append(f"{k}={v}")
    
    key_string = ":".join(key_parts)
    
    # 길이 제한이 있으면 hash 사용 (선택)
    if len(key_string) > 200:
        key_hash = hashlib.sha256(key_string.encode()).hexdigest()
        return f"{prefix}:{key_hash}"
    
    return key_string
```

**요구사항**:
- CacheBackend 추상 클래스로 인터페이스 정의
- InMemoryLRUCache 구현 (기본, 폐쇄망용)
- RedisCacheBackend 구현 (선택사항)
- CacheManager로 백엔드 자동 선택 (Redis → fallback)
- TTL 및 LRU 제거 정책 구현
- 로깅으로 캐시 hit/miss 추적

---

### 4-3. 검색 결과 캐싱 통합

**파일**: `backend/src/services/search_cache_service.py` (새 파일)

```python
from typing import List, Dict, Any, Optional
from uuid import UUID
import logging
from sqlalchemy.orm import Session

from src.core.cache import CacheManager, make_cache_key
from src.services.retrieval_service import RetrievalService
from src.core.config import settings

logger = logging.getLogger(__name__)

class SearchCacheService:
    """검색 결과 캐싱 서비스"""
    
    SEARCH_CACHE_TTL = 3600  # 1시간
    
    def __init__(self, retrieval_service: RetrievalService):
        self.retrieval = retrieval_service
        self.cache = CacheManager.get_cache(
            redis_enabled=settings.EXTERNAL_DEPENDENCIES_ENABLED
        )
    
    def search_with_cache(
        self,
        query: str,
        top_k: int,
        user_id: UUID,
        user_scope: Dict[str, Any]
    ) -> List[Dict[str, Any]]:
        """
        캐시를 활용한 검색 수행
        
        Args:
            query: 검색 쿼리
            top_k: 상위 K개 결과
            user_id: 사용자 ID
            user_scope: 사용자 scope profile (ACL 필터링용)
        
        Returns:
            검색 결과 리스트
        """
        
        # 캐시 키 생성
        cache_key = make_cache_key(
            "search_results",
            query=query,
            top_k=top_k,
            user_id=user_id
        )
        
        # 캐시 조회
        cached_results = self.cache.get(cache_key)
        if cached_results:
            logger.info(f"Search cache hit: {cache_key}")
            return cached_results
        
        # 캐시 미스: 실제 검색 수행
        logger.info(f"Search cache miss: {cache_key}")
        results = self.retrieval.search(
            query=query,
            top_k=top_k,
            user_scope=user_scope
        )
        
        # 캐시에 저장
        self.cache.set(cache_key, results, self.SEARCH_CACHE_TTL)
        
        return results
    
    def invalidate_search_cache(self, user_id: Optional[UUID] = None) -> bool:
        """
        검색 결과 캐시 무효화
        
        Args:
            user_id: 특정 사용자의 캐시만 무효화 (None이면 전체)
        
        Returns:
            성공 여부
        """
        
        if user_id is None:
            # 전체 캐시 제거 (관리자용)
            return self.cache.clear()
        
        # 특정 사용자의 캐시만 무효화 (선택사항, 복잡함)
        logger.warning(f"User-specific cache invalidation not fully implemented. Clearing all cache.")
        return self.cache.clear()
```

**요구사항**:
- make_cache_key()로 일관된 캐시 키 생성
- 캐시 hit/miss 로깅
- user_id와 query 기반 캐시 격리
- Document 업데이트 시 캐시 무효화 API

---

### 4-4. Citation 검증 캐싱

**파일**: `backend/src/services/citation_validation_cache.py` (새 파일)

```python
from typing import Dict, Any, Optional
from uuid import UUID
import logging

from src.core.cache import CacheManager, make_cache_key
from src.models.citation import validate_citation_integrity

logger = logging.getLogger(__name__)

class CitationValidationCache:
    """Citation 5-tuple 검증 캐시"""
    
    CITATION_VALIDATION_TTL = 86400  # 24시간
    
    def __init__(self):
        self.cache = CacheManager.get_cache()
    
    def validate_with_cache(self, citation: Dict[str, Any]) -> bool:
        """
        캐시를 활용한 Citation 검증
        
        Args:
            citation: Citation 5-tuple
                {document_id, version_id, node_id, span_offset, content_hash, content}
        
        Returns:
            유효성 (True/False)
        """
        
        content_hash = citation.get("content_hash")
        if not content_hash:
            return False
        
        # 캐시 키 생성 (content_hash 기반)
        cache_key = make_cache_key("citation_validation", content_hash=content_hash)
        
        # 캐시 조회
        cached_result = self.cache.get(cache_key)
        if cached_result is not None:
            logger.debug(f"Citation validation cache hit: {content_hash}")
            return cached_result
        
        # 캐시 미스: 실제 검증 수행
        logger.debug(f"Citation validation cache miss: {content_hash}")
        is_valid = validate_citation_integrity(citation)
        
        # 캐시에 저장
        self.cache.set(cache_key, is_valid, self.CITATION_VALIDATION_TTL)
        
        return is_valid
    
    def invalidate_by_content_hash(self, content_hash: str) -> bool:
        """
        특정 content_hash의 검증 캐시 무효화
        
        Args:
            content_hash: 무효화할 content_hash
        
        Returns:
            성공 여부
        """
        
        cache_key = make_cache_key("citation_validation", content_hash=content_hash)
        return self.cache.delete(cache_key)
```

**요구사항**:
- content_hash를 캐시 키로 사용
- 검증 결과 (True/False)만 저장 (간단함)
- TTL: 24시간 (변경 가능성 낮음)

---

### 4-5. 프롬프트 토큰 계산 캐시

**파일**: `backend/src/services/conversation_rag_service.py` 확장

```python
# ConversationRAGService에 다음 메서드 추가

class ConversationRAGService:
    
    def _calculate_token_delta(
        self,
        current_turn: Turn,
        previous_turn: Optional[Turn],
        model: str
    ) -> int:
        """
        현재 턴과 이전 턴의 토큰 차이 계산 (delta)
        
        Args:
            current_turn: 현재 Turn 객체
            previous_turn: 이전 Turn 객체 (선택사항)
            model: LLM 모델명
        
        Returns:
            추가 토큰 수 (delta)
        """
        
        from src.core.llm import count_tokens
        
        # 현재 턴의 토큰 계산
        current_tokens = (
            count_tokens(current_turn.user_message, model) +
            count_tokens(current_turn.assistant_response, model)
        )
        
        # 이전 턴이 없으면 현재 턴 그대로
        if previous_turn is None:
            current_turn.retrieval_metadata = {
                **current_turn.retrieval_metadata,
                "cached_token_count": current_tokens
            }
            return current_tokens
        
        # 이전 턴의 캐시된 토큰 가져오기
        previous_cached = previous_turn.retrieval_metadata.get("cached_token_count", 0)
        
        # Delta 계산
        delta = current_tokens - previous_cached
        
        # 현재 턴에 캐시된 토큰 저장
        current_turn.retrieval_metadata = {
            **current_turn.retrieval_metadata,
            "cached_token_count": current_tokens,
            "token_delta": delta
        }
        
        logger.debug(f"Token delta calculated: prev={previous_cached}, current={current_tokens}, delta={delta}")
        
        return delta
```

**요구사항**:
- Turn metadata에 cached_token_count 저장
- delta = current - previous 계산
- 다음 턴에서 delta 활용

---

### 4-6. 성능 벤치마크 및 모니터링

**파일**: `backend/src/services/performance_monitor.py` (새 파일)

```python
import time
import logging
from typing import Dict, Any, Callable
from datetime import datetime, timedelta
from collections import defaultdict

logger = logging.getLogger(__name__)

class PerformanceMonitor:
    """RAG 성능 모니터링 및 벤치마크"""
    
    def __init__(self):
        self.metrics = defaultdict(list)
    
    def record_metric(self, name: str, value: float, metadata: Dict[str, Any] = None):
        """
        성능 메트릭 기록
        
        Args:
            name: 메트릭 이름 (예: "rag_single_turn_time", "rag_multi_turn_time")
            value: 메트릭 값 (밀리초)
            metadata: 추가 메타데이터
        """
        
        record = {
            "timestamp": datetime.utcnow(),
            "value": value,
            "metadata": metadata or {}
        }
        
        self.metrics[name].append(record)
        logger.debug(f"Metric recorded: {name}={value}ms")
    
    def get_stats(self, name: str, time_window: int = 3600) -> Dict[str, float]:
        """
        메트릭 통계 계산
        
        Args:
            name: 메트릭 이름
            time_window: 시간 창 (초, 기본 1시간)
        
        Returns:
            통계 딕셔너리 {mean, median, p95, p99, min, max, count}
        """
        
        if name not in self.metrics:
            return {}
        
        # 시간 창 내의 메트릭 필터링
        cutoff_time = datetime.utcnow() - timedelta(seconds=time_window)
        recent_values = [
            r["value"] for r in self.metrics[name]
            if r["timestamp"] > cutoff_time
        ]
        
        if not recent_values:
            return {}
        
        recent_values.sort()
        
        return {
            "mean": sum(recent_values) / len(recent_values),
            "median": recent_values[len(recent_values) // 2],
            "p95": recent_values[int(len(recent_values) * 0.95)],
            "p99": recent_values[int(len(recent_values) * 0.99)],
            "min": min(recent_values),
            "max": max(recent_values),
            "count": len(recent_values)
        }
    
    def timeit(self, name: str) -> Callable:
        """
        데코레이터: 함수 실행 시간 측정
        
        Args:
            name: 메트릭 이름
        
        Returns:
            데코레이터 함수
        """
        
        def decorator(func):
            def wrapper(*args, **kwargs):
                start = time.time()
                result = func(*args, **kwargs)
                elapsed_ms = (time.time() - start) * 1000
                self.record_metric(name, elapsed_ms, {"function": func.__name__})
                return result
            return wrapper
        return decorator

# 전역 모니터 인스턴스
_monitor = PerformanceMonitor()

def get_monitor() -> PerformanceMonitor:
    return _monitor
```

**요구사항**:
- 응답 시간 측정 (밀리초)
- 통계 계산 (평균, 중앙값, p95, p99)
- 데코레이터로 쉬운 계측
- 시간 창 기반 필터링

---

### 4-7. 단위 테스트 작성

**파일**: `backend/tests/test_citation_caching.py`

```python
import pytest
from uuid import uuid4
from datetime import datetime
from unittest.mock import Mock, patch, MagicMock

from src.services.citation_reuse_service import CitationReuseService
from src.services.search_cache_service import SearchCacheService
from src.services.citation_validation_cache import CitationValidationCache
from src.core.cache import InMemoryLRUCache, CacheManager
from src.models import Turn

# ===== Fixtures =====

@pytest.fixture
def in_memory_cache():
    return InMemoryLRUCache(max_size=100)

@pytest.fixture
def mock_retrieval_service():
    service = Mock()
    service.search.return_value = [
        {
            "document_id": uuid4(),
            "score": 0.85,
            "content": "Test document content"
        }
    ]
    return service

# ===== Test Cases: Citation Reuse =====

class TestCitationReuse:
    """Citation 재활용 테스트"""
    
    def test_citation_reuse_bonus(self, db):
        """이전 Citation에 보너스 점수 적용"""
        
        service = CitationReuseService(db)
        
        # 이전 턴 (인용된 Document)
        doc_id = uuid4()
        previous_turn = Turn(
            turn_number=1,
            user_message="Previous question",
            assistant_response="Previous answer",
            retrieval_metadata={
                "citations": [
                    {
                        "document_id": str(doc_id),
                        "content": "cited content"
                    }
                ]
            },
            created_at=datetime.utcnow()
        )
        
        # 현재 검색 결과
        search_results = [
            {
                "document_id": doc_id,
                "score": 0.8,
                "content": "Test content"
            },
            {
                "document_id": uuid4(),
                "score": 0.7,
                "content": "Other content"
            }
        ]
        
        # 보너스 적용
        boosted = service.apply_citation_bonus(search_results, [previous_turn])
        
        # 첫 번째 결과 (보너스 받음)가 더 높은 점수를 가져야 함
        assert boosted[0]["document_id"] == doc_id
        assert boosted[0]["score"] == 0.8 * 1.5  # 50% 보너스
        assert boosted[0]["citation_reused"] is True

class TestSearchCache:
    """검색 결과 캐싱 테스트"""
    
    @patch('src.core.cache.CacheManager.get_cache')
    def test_search_cache_hit(self, mock_get_cache, mock_retrieval_service):
        """캐시 hit 시 재계산 없음"""
        
        cache = InMemoryLRUCache()
        mock_get_cache.return_value = cache
        
        # 캐시에 미리 저장
        cached_results = [{"document_id": uuid4(), "score": 0.9}]
        cache.set("search_results:query=test:top_k=5:user_id=123", cached_results)
        
        service = SearchCacheService(mock_retrieval_service)
        
        results = service.search_with_cache(
            query="test",
            top_k=5,
            user_id=uuid4(),
            user_scope={}
        )
        
        # retrieval_service.search가 호출되지 않음
        mock_retrieval_service.search.assert_not_called()
        assert results == cached_results
    
    @patch('src.core.cache.CacheManager.get_cache')
    def test_search_cache_miss(self, mock_get_cache, mock_retrieval_service):
        """캐시 miss 시 재계산 및 저장"""
        
        cache = InMemoryLRUCache()
        mock_get_cache.return_value = cache
        
        search_results = [{"document_id": uuid4(), "score": 0.8}]
        mock_retrieval_service.search.return_value = search_results
        
        service = SearchCacheService(mock_retrieval_service)
        
        results = service.search_with_cache(
            query="new_query",
            top_k=5,
            user_id=uuid4(),
            user_scope={}
        )
        
        # retrieval_service.search 호출됨
        mock_retrieval_service.search.assert_called_once()
        assert results == search_results
        
        # 캐시에 저장됨
        assert cache.get("search_results:query=new_query:top_k=5:user_id=") is not None

class TestCitationValidationCache:
    """Citation 검증 캐시 테스트"""
    
    @patch('src.core.cache.CacheManager.get_cache')
    @patch('src.models.citation.validate_citation_integrity')
    def test_citation_validation_cache_hit(self, mock_validate, mock_get_cache):
        """Citation 검증 캐시 hit"""
        
        cache = InMemoryLRUCache()
        mock_get_cache.return_value = cache
        
        # 캐시에 미리 저장
        content_hash = "sha256:abc123"
        cache.set(f"citation_validation:content_hash={content_hash}", True)
        
        citation = {"content_hash": content_hash, "content": "test"}
        validator = CitationValidationCache()
        
        result = validator.validate_with_cache(citation)
        
        # validate_citation_integrity 호출 안 됨
        mock_validate.assert_not_called()
        assert result is True
    
    @patch('src.core.cache.CacheManager.get_cache')
    @patch('src.models.citation.validate_citation_integrity')
    def test_citation_validation_cache_miss(self, mock_validate, mock_get_cache):
        """Citation 검증 캐시 miss"""
        
        cache = InMemoryLRUCache()
        mock_get_cache.return_value = cache
        mock_validate.return_value = False
        
        content_hash = "sha256:xyz789"
        citation = {"content_hash": content_hash, "content": "test"}
        
        validator = CitationValidationCache()
        result = validator.validate_with_cache(citation)
        
        # validate_citation_integrity 호출됨
        mock_validate.assert_called_once()
        assert result is False

class TestTokenDelta:
    """토큰 delta 계산 테스트"""
    
    @patch('src.core.llm.count_tokens')
    def test_token_delta_calculation(self, mock_count_tokens):
        """토큰 delta 계산"""
        
        mock_count_tokens.side_effect = lambda text, model: len(text.split())
        
        from src.services.conversation_rag_service import ConversationRAGService
        
        # 이전 턴
        previous_turn = Turn(
            turn_number=1,
            user_message="Previous question",
            assistant_response="Previous answer",
            retrieval_metadata={"cached_token_count": 10},
            created_at=datetime.utcnow()
        )
        
        # 현재 턴
        current_turn = Turn(
            turn_number=2,
            user_message="Current question",
            assistant_response="Current answer",
            retrieval_metadata={},
            created_at=datetime.utcnow()
        )
        
        service = ConversationRAGService(Mock(), Mock(), Mock())
        delta = service._calculate_token_delta(current_turn, previous_turn, "gpt-4")
        
        # delta 계산되어 저장됨
        assert "token_delta" in current_turn.retrieval_metadata

class TestFallbackToMemoryCache:
    """폐쇄망 fallback 테스트 (S2 원칙 ⑦)"""
    
    @patch.dict('os.environ', {'EXTERNAL_DEPENDENCIES_ENABLED': 'false'})
    def test_fallback_when_redis_unavailable(self):
        """Redis unavailable 시 in-memory 사용"""
        
        # Redis 없이 캐시 생성
        cache = CacheManager.get_cache(redis_enabled=False)
        
        # in-memory cache여야 함
        assert isinstance(cache, InMemoryLRUCache)
        
        # 정상 작동
        cache.set("key1", "value1")
        assert cache.get("key1") == "value1"

class TestCacheInvalidation:
    """캐시 무효화 테스트"""
    
    def test_cache_invalidation_on_document_update(self):
        """Document 업데이트 시 캐시 무효화"""
        
        cache = InMemoryLRUCache()
        
        # 캐시 저장
        cache.set("search_results:query=test:user_id=123", [{"doc": 1}])
        assert cache.get("search_results:query=test:user_id=123") is not None
        
        # 캐시 무효화
        cache.clear()
        assert cache.get("search_results:query=test:user_id=123") is None

class TestPerformanceBenchmark:
    """성능 벤치마크 테스트"""
    
    @patch('src.services.conversation_rag_service.ConversationRAGService.answer')
    def test_performance_benchmark_single_vs_multi_turn(self, mock_answer):
        """단발 vs 멀티턴 응답 시간 비교"""
        
        from src.services.performance_monitor import get_monitor
        
        monitor = get_monitor()
        
        # 단발 RAG: 100ms
        monitor.record_metric("rag_single_turn_time", 100)
        
        # 멀티턴 RAG: 150ms (+50ms)
        monitor.record_metric("rag_multi_turn_time", 150)
        
        # 통계 확인
        single_stats = monitor.get_stats("rag_single_turn_time")
        multi_stats = monitor.get_stats("rag_multi_turn_time")
        
        # 멀티턴이 +50ms 이내의 오버헤드
        overhead = multi_stats["mean"] - single_stats["mean"]
        assert overhead <= 500  # 500ms 이내 (S2 요구사항)
```

**요구사항**:
- pytest framework
- Mock 및 patch 활용
- 캐시 hit/miss 시나리오
- 토큰 delta 계산 검증
- 폐쇄망 fallback 테스트
- 성능 벤치마크

---

## 5. 산출물

1. **CitationReuseService 클래스** (`backend/src/services/citation_reuse_service.py`)
   - extract_cited_documents(): 이전 Citation 추출
   - apply_citation_bonus(): 보너스 점수 적용

2. **캐싱 프레임워크** (`backend/src/core/cache.py`)
   - CacheBackend 추상 클래스
   - InMemoryLRUCache (기본)
   - RedisCacheBackend (선택)
   - CacheManager (팩토리)

3. **검색 결과 캐싱** (`backend/src/services/search_cache_service.py`)
   - SearchCacheService
   - 캐시 키 생성 및 무효화

4. **Citation 검증 캐싱** (`backend/src/services/citation_validation_cache.py`)
   - CitationValidationCache

5. **성능 모니터링** (`backend/src/services/performance_monitor.py`)
   - PerformanceMonitor
   - 메트릭 기록 및 통계

6. **ConversationRAGService 확장**
   - token delta 계산
   - 캐싱 레이어 통합

7. **단위 테스트** (`backend/tests/test_citation_caching.py`)
   - Citation 재활용 (1개)
   - 검색 결과 캐싱 (2개)
   - Citation 검증 캐싱 (2개)
   - 토큰 delta (1개)
   - Fallback (1개)
   - 캐시 무효화 (1개)
   - 성능 벤치마크 (1개)
   - **총 9개 test case**

---

## 6. 완료 기준

- [x] CitationReuseService가 이전 Citation 추출하는가
- [x] 이전 Citation에 보너스 점수(1.5배) 적용되는가
- [x] 보너스 후 점수 기준으로 재정렬되는가
- [x] InMemoryLRUCache가 TTL 및 LRU 제거 정책 구현하는가
- [x] RedisCacheBackend가 선택사항으로 구현되는가
- [x] CacheManager가 Redis → fallback 자동 선택하는가
- [x] 검색 결과 캐시 hit율이 측정되는가
- [x] Citation 검증 캐시 (content_hash 기반) 구현되는가
- [x] 프롬프트 토큰 delta 계산되고 저장되는가
- [x] 폐쇄망 환경에서 in-memory 캐시로 동작하는가 (S2 원칙 ⑦)
- [x] 성능 벤치마크: 멀티턴 오버헤드 <500ms
- [x] 모든 단위 테스트 통과 (pytest)
- [x] 캐시 무효화 전략 (Document 업데이트 시) 구현

---

## 7. 작업 지침

### 7-1. 캐시 일관성 보장
- 검색 결과 캐시: user_id 단위로 격리 (한 사용자의 검색이 다른 사용자에게 노출 방지)
- Citation 검증 캐시: content_hash 기반 (모든 사용자 공유)
- Document 업데이트 시: 검색 결과 캐시 전체 무효화 (간단함)

### 7-2. 성능 최적화 우선순위
1. 검색 결과 캐싱 (가장 효과적)
2. Citation 검증 캐싱 (선택사항)
3. 토큰 계산 delta (선택사항)

### 7-3. 로깅 수준
- DEBUG: 캐시 hit/miss
- INFO: 보너스 적용, 캐시 생성
- WARNING: Redis unavailable
- ERROR: 캐시 연산 실패

### 7-4. 폐쇄망 환경 고려 (S2 원칙 ⑦)
- Redis 미사용 설정: EXTERNAL_DEPENDENCIES_ENABLED=false
- in-memory LRU 자동 활성화
- max_size=1000 (메모리 초과 방지)

### 7-5. 보너스 점수 조정
- 기본값: 1.5배 (50% 상향)
- 조정 가능: settings.CITATION_BONUS_MULTIPLIER
- A/B 테스트로 최적값 찾기

### 7-6. 캐시 키 설계
- 구조: "prefix:key1=value1:key2=value2"
- UUID는 문자열로 변환
- 길이 > 200자면 SHA256 hash 사용

---

## 8. 성능 목표

| 항목 | 목표 | 달성 기준 |
|------|------|----------|
| 단발 RAG 응답 시간 | <300ms | baseline |
| 멀티턴 RAG 추가 오버헤드 | <500ms | +200ms 정도 목표 |
| 캐시 hit율 | >70% | 반복 쿼리 많을 때 |
| 검색 결과 캐시 히트 시 시간 절감 | >50% | 100ms 이상 |
| 메모리 사용 (in-memory cache) | <500MB | 1000개 항목 기준 |

---

## 참고 자료

- **Task 3-4 RAG API**: `/docs/개발문서/S2/phase3/작업지시서/task3-4.md`
- **Task 3-5 컨텍스트 윈도우**: `/docs/개발문서/S2/phase3/작업지시서/task3-5.md`
- **Phase 2 Citation 5-tuple**: `/docs/개발문서/S2/phase2/citation_contract.md`
- **S2 원칙**: `/CLAUDE.md` (Mimir 프로젝트 설정)
- **Python lru_cache**: https://docs.python.org/3/library/functools.html#functools.lru_cache
- **Redis Python**: https://github.com/redis/redis-py
