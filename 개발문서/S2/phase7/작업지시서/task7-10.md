# Task 7-10: PH3-CARRY-002 대화 FTS 전환 (tsvector + GIN 인덱스)

> **이월 출처**: Phase 3 전체검수보고서 §이월 항목 PH3-CARRY-002  
> **이월 경로**: Phase 3 → Phase 6 미처리 → Phase 7 처리

## 1. 작업 목적

`conversations` 테이블의 제목 검색이 현재 `ILIKE '%query%'` 방식으로 구현되어 있어, 대화 수가 증가할수록 순차 스캔이 발생하고 성능이 저하된다. PostgreSQL의 **Full-Text Search(FTS)** — `tsvector` + GIN 인덱스 — 로 전환하여 1만 건 이상의 대화에서도 50ms 이내 응답을 보장한다.

## 2. 작업 범위

### 포함 범위

1. **DB 스키마 변경**
   - `conversations` 테이블에 `title_tsv tsvector` 컬럼 추가
   - GIN 인덱스 생성: `CREATE INDEX CONCURRENTLY idx_conv_title_fts ON conversations USING GIN(title_tsv)`
   - 자동 갱신 트리거: `title` 변경 시 `title_tsv` 자동 업데이트
   - 기존 데이터 마이그레이션: `UPDATE conversations SET title_tsv = to_tsvector('simple', coalesce(title, ''))`

2. **Alembic 마이그레이션 스크립트**
   - `CONCURRENTLY` 인덱스로 운영 중 무중단 적용
   - downgrade: 인덱스 + 컬럼 + 트리거 제거

3. **백엔드 검색 API 수정**
   - 파일: `backend/app/api/v1/conversations.py` (또는 관련 router)
   - `ILIKE` → `to_tsquery()` 기반 쿼리로 전환
   - `ts_rank(title_tsv, query)` 기준 정렬
   - Scope Profile 기반 ACL 필터 유지 (S2 원칙 ⑥)
   - 폐쇄망 환경: 외부 의존 없음 (S2 원칙 ⑦)

4. **단위 및 통합 테스트**
   - FTS 검색 정확도 검증
   - 한국어 + 영어 혼합 쿼리 처리 (`simple` dictionary)
   - ACL 필터가 FTS 경로에서도 동일하게 적용되는지 검증

### 제외 범위

- 프론트엔드 변경 (API 응답 형식 유지)
- 대화 내용(content) 본문 FTS — 별도 과제
- 유사도 검색 (pgvector) — Phase 10 이후 처리

## 3. 선행 조건

- Phase 3: conversations 테이블 및 기본 검색 API 구현 완료
- PostgreSQL 12 이상 (tsvector, GIN 지원)
- Alembic 마이그레이션 환경 설정 완료

## 4. 주요 작업 항목

### 4-1. Alembic 마이그레이션

**파일:** `backend/alembic/versions/phase7_carry002_conversations_fts.py`

```python
"""Add FTS tsvector column to conversations (PH3-CARRY-002).

Revision ID: ph7_carry002
Revises: <previous_revision>
Create Date: 2026-04-17
"""

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision = 'ph7_carry002'
down_revision = '<previous_revision>'
branch_labels = None
depends_on = None


def upgrade() -> None:
    # 1. tsvector 컬럼 추가
    op.add_column(
        'conversations',
        sa.Column(
            'title_tsv',
            postgresql.TSVECTOR(),
            nullable=True,
        )
    )

    # 2. 기존 데이터 채우기 (simple dictionary: 한국어/영어 혼합 지원)
    op.execute("""
        UPDATE conversations
        SET title_tsv = to_tsvector('simple', coalesce(title, ''))
    """)

    # 3. GIN 인덱스 생성 (CONCURRENTLY로 운영 중 무중단 적용)
    op.execute("""
        CREATE INDEX CONCURRENTLY idx_conv_title_fts
        ON conversations USING GIN(title_tsv)
    """)

    # 4. 자동 갱신 트리거 생성
    op.execute("""
        CREATE OR REPLACE FUNCTION update_conv_title_tsv()
        RETURNS trigger AS $$
        BEGIN
            NEW.title_tsv := to_tsvector('simple', coalesce(NEW.title, ''));
            RETURN NEW;
        END;
        $$ LANGUAGE plpgsql;
    """)

    op.execute("""
        CREATE TRIGGER trg_conv_title_tsv
        BEFORE INSERT OR UPDATE OF title
        ON conversations
        FOR EACH ROW
        EXECUTE FUNCTION update_conv_title_tsv();
    """)


def downgrade() -> None:
    op.execute("DROP TRIGGER IF EXISTS trg_conv_title_tsv ON conversations")
    op.execute("DROP FUNCTION IF EXISTS update_conv_title_tsv()")
    op.execute("DROP INDEX CONCURRENTLY IF EXISTS idx_conv_title_fts")
    op.drop_column('conversations', 'title_tsv')
```

### 4-2. 검색 쿼리 수정

**파일:** `backend/app/repositories/conversation_repository.py` (또는 해당 위치)

**Before (ILIKE 방식):**
```python
query = select(ConversationORM).where(
    and_(
        ConversationORM.scope_id == scope_id,
        ConversationORM.title.ilike(f"%{search_term}%"),
    )
)
```

**After (FTS 방식):**
```python
from sqlalchemy import func, text

def search_conversations(
    self,
    scope_id: str,
    search_term: str,
    offset: int = 0,
    limit: int = 20,
) -> tuple[list[Conversation], int]:
    """
    FTS 기반 대화 검색 (S2 ⑥: scope_id ACL 필터 필수).
    
    'simple' dictionary 사용: 한국어/영어 혼합 쿼리 처리.
    ts_rank 기준 관련도 높은 순 정렬.
    """
    # 검색어를 tsquery로 변환 (AND 연산)
    # e.g. "도커 컨테이너" → "도커 & 컨테이너"
    ts_query = func.plainto_tsquery("simple", search_term)

    base_filters = [
        ConversationORM.scope_id == scope_id,  # S2 ⑥: 필수 ACL
        ConversationORM.is_deleted == False,
        ConversationORM.title_tsv.op("@@")(ts_query),  # FTS 매칭
    ]

    # 전체 개수
    count_q = select(func.count()).select_from(ConversationORM).where(and_(*base_filters))
    total = (await self.db.execute(count_q)).scalar() or 0

    # 관련도 정렬 조회
    rank_col = func.ts_rank(ConversationORM.title_tsv, ts_query).label("rank")
    q = (
        select(ConversationORM, rank_col)
        .where(and_(*base_filters))
        .order_by(rank_col.desc(), ConversationORM.created_at.desc())
        .offset(offset)
        .limit(limit)
    )

    result = await self.db.execute(q)
    rows = result.all()

    return [self._orm_to_domain(row.ConversationORM) for row in rows], total
```

**폴백 처리** (title_tsv가 NULL인 레코드 — 마이그레이션 전 생성된 극소수 케이스):
```python
# title_tsv IS NULL인 경우 ILIKE 폴백 (마이그레이션 완료 후 제거 예정)
base_filters_fallback = [
    ConversationORM.scope_id == scope_id,
    ConversationORM.is_deleted == False,
    ConversationORM.title_tsv.is_(None),
    ConversationORM.title.ilike(f"%{search_term}%"),
]
```

### 4-3. 단위 테스트

**파일:** `backend/tests/repositories/test_conversation_fts.py`

```python
"""대화 FTS 검색 단위 테스트 (PH3-CARRY-002)."""

import pytest

@pytest.mark.asyncio
class TestConversationFTS:
    
    async def test_fts_basic_search(self, async_db_session):
        """기본 FTS 검색: 정확한 단어 매칭."""
        repo = ConversationRepository(async_db_session)
        
        # 데이터 생성
        await repo.create(scope_id="scope-1", title="도커 컨테이너 설정 가이드", ...)
        await repo.create(scope_id="scope-1", title="Kubernetes 클러스터 구성", ...)
        await repo.create(scope_id="scope-1", title="전혀 관련 없는 문서", ...)
        
        # "도커" 검색
        results, total = await repo.search_conversations(
            scope_id="scope-1",
            search_term="도커",
        )
        
        assert total == 1
        assert results[0].title == "도커 컨테이너 설정 가이드"
    
    async def test_fts_scope_acl(self, async_db_session):
        """S2 ⑥: scope_id ACL이 FTS 경로에서도 적용됨."""
        repo = ConversationRepository(async_db_session)
        
        await repo.create(scope_id="scope-A", title="도커 가이드", ...)
        await repo.create(scope_id="scope-B", title="도커 설정", ...)
        
        # scope-A로만 조회
        results, total = await repo.search_conversations(
            scope_id="scope-A",
            search_term="도커",
        )
        
        assert total == 1
        assert results[0].title == "도커 가이드"
    
    async def test_fts_korean_english_mixed(self, async_db_session):
        """한국어+영어 혼합 검색."""
        repo = ConversationRepository(async_db_session)
        
        await repo.create(scope_id="scope-1", title="Docker 컨테이너 배포 방법", ...)
        
        results, _ = await repo.search_conversations(
            scope_id="scope-1",
            search_term="Docker 컨테이너",
        )
        
        assert len(results) == 1
    
    async def test_fts_empty_result(self, async_db_session):
        """검색 결과 없음."""
        repo = ConversationRepository(async_db_session)
        
        results, total = await repo.search_conversations(
            scope_id="scope-1",
            search_term="존재하지않는검색어xyz",
        )
        
        assert total == 0
        assert len(results) == 0
    
    async def test_fts_rank_ordering(self, async_db_session):
        """ts_rank 기반 정렬: 더 관련도 높은 결과가 먼저."""
        repo = ConversationRepository(async_db_session)
        
        # title에 "도커"가 더 많이 포함된 문서가 높은 rank
        await repo.create(scope_id="scope-1", title="도커 도커 설정", ...)
        await repo.create(scope_id="scope-1", title="도커 입문", ...)
        
        results, _ = await repo.search_conversations(
            scope_id="scope-1",
            search_term="도커",
        )
        
        assert results[0].title == "도커 도커 설정"  # rank 더 높음
```

## 5. 산출물

1. **Alembic 마이그레이션** (`backend/alembic/versions/phase7_carry002_conversations_fts.py`)
   - tsvector 컬럼, GIN 인덱스, 자동 갱신 트리거

2. **Repository 수정** (`backend/app/repositories/conversation_repository.py`)
   - `search_conversations()` 메서드 FTS 전환

3. **단위 테스트** (`backend/tests/repositories/test_conversation_fts.py`)
   - 5개 이상 테스트 케이스

## 6. 완료 기준

1. `conversations` 테이블에 `title_tsv` 컬럼 및 GIN 인덱스가 존재하는가?
2. 자동 갱신 트리거가 INSERT/UPDATE 시 `title_tsv`를 갱신하는가?
3. 검색 API가 `ILIKE` 대신 `@@` (tsquery match) 연산자를 사용하는가?
4. S2 ⑥: scope_id ACL 필터가 FTS 경로에서도 동일하게 적용되는가?
5. 한국어 + 영어 혼합 검색이 정상 동작하는가? (simple dictionary)
6. 1만 건 대화 기준 검색 응답이 50ms 이내인가?
7. 기존 API 응답 형식이 변경되지 않아 프론트엔드 수정이 불필요한가?
8. downgrade 마이그레이션이 정상 롤백되는가?
9. 모든 단위 테스트가 통과하는가?

## 7. 작업 지침

### 지침 7-10-1. simple dictionary 선택 이유
`korean`, `english` 등 언어 특화 dictionary는 별도 설치가 필요하고 폐쇄망에서 의존성이 생긴다. `simple` dictionary는 PostgreSQL 기본 제공으로 의존성 없이 한국어/영어 혼합 검색을 지원한다 (형태소 분석 없이 공백 기준 토큰화).

### 지침 7-10-2. CONCURRENTLY 인덱스
`CREATE INDEX CONCURRENTLY`는 테이블 락 없이 인덱스를 생성하므로 운영 중 무중단 마이그레이션이 가능하다. 단, 트랜잭션 블록 내에서는 사용 불가하므로 Alembic에서 별도 `op.execute()`로 실행한다.

### 지침 7-10-3. 프론트엔드 영향 없음
API 응답 형식(`conversations[]` 배열)은 변경하지 않는다. 정렬 순서가 `created_at DESC` → `ts_rank DESC, created_at DESC`로 변경되지만, 이는 검색 품질 개선이므로 허용된다.

### 지침 7-10-4. 폐쇄망 환경 (S2 ⑦)
FTS는 PostgreSQL 내장 기능이므로 외부 의존 없음. 폐쇄망에서도 동일하게 동작한다.
