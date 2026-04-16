# Task 2-3. Citation 검증 테스트 스위트 및 S1 호환성 보장

## 1. 작업 목적

FG2.1의 산출물(Citation 모델, 역참조 API)에 대한 **포괄적인 테스트 스위트**를 구축하고, S1 Phase 11 클라이언트가 기존 API 계약을 깨지 않고 동작하는지 검증한다. 회귀 테스트(100개 샘플)와 S1 호환 마이그레이션 가이드도 포함한다.

## 2. 작업 범위

### 포함 범위

1. **단위 테스트 보강**
   - `content_hash` 계산 정확성 (다양한 인코딩, 공백, 유니코드)
   - `CitationBuilder.from_retrieved_chunk()` 엣지 케이스 (node_id None 등)
   - `CitationVerifyResponse` 직렬화

2. **통합 테스트: DB 기반 검증**
   - 문서 생성 → 벡터화 → citation 조회 → verify 전체 파이프라인
   - 문서 개정(version_id 변경) 후 이전 citation의 `modified: true` 확인

3. **회귀 테스트: 100개 샘플 문서**
   - 샘플 문서 100개에 대해 citation 생성 → verify 모두 `verified: true`
   - pytest 픽스처로 샘플 생성 자동화

4. **S1 호환성 테스트**
   - S1 `/search/documents` 응답에서 `citation` 필드가 없어도 정상 파싱
   - S1 클라이언트 시뮬레이션: `citation` 필드 무시하고 기존 필드만 사용
   - S2 클라이언트 시뮬레이션: `citation` 필드 활용

5. **S1 → S2 마이그레이션 가이드** (Markdown 문서)

### 제외 범위

- Citation API 엔드포인트 구현 (Task 2-2)
- 성능 벤치마크 (Phase 7 AI품질평가)

## 3. 선행 조건

- Task 2-1, Task 2-2 완료
- pytest, pytest-asyncio 설치
- 테스트 DB 접근 가능 (또는 SQLite in-memory 대체)

## 4. 주요 작업 항목

### 4-1. 단위 테스트 보강

**파일:** `/backend/tests/unit/services/test_citation_edge_cases.py`

```python
"""Citation 엣지 케이스 단위 테스트."""

import hashlib
import pytest
from uuid import UUID, uuid4
from app.schemas.citation import Citation
from app.services.retrieval.citation_builder import CitationBuilder


# ── 인코딩 다양성 ──────────────────────────────────────────────

@pytest.mark.parametrize("text", [
    "Hello, World!",
    "안녕하세요 세계",
    "こんにちは世界",
    "Привет мир",
    "  공백 포함  ",
    "",  # 빈 문자열 (CitationBuilder는 raise, Citation.from_chunk는 hash 계산)
])
def test_sha256_matches_stdlib(text: str):
    """Citation.from_chunk의 SHA-256이 표준 라이브러리 결과와 동일해야 한다."""
    if text == "":
        with pytest.raises((ValueError, Exception)):
            CitationBuilder.build(uuid4(), uuid4(), uuid4(), text)
    else:
        c = Citation.from_chunk(uuid4(), uuid4(), uuid4(), text)
        expected = hashlib.sha256(text.encode("utf-8")).hexdigest()
        assert c.content_hash == expected


# ── span_offset ────────────────────────────────────────────────

def test_span_offset_stored():
    c = Citation.from_chunk(uuid4(), uuid4(), uuid4(), "text", span_offset=42)
    assert c.span_offset == 42


def test_span_offset_none_by_default():
    c = Citation.from_chunk(uuid4(), uuid4(), uuid4(), "text")
    assert c.span_offset is None


def test_span_offset_negative_rejected():
    with pytest.raises(Exception):
        Citation(
            document_id=uuid4(),
            version_id=uuid4(),
            node_id=uuid4(),
            span_offset=-1,
            content_hash="a" * 64,
        )


# ── node_id None 처리 ─────────────────────────────────────────

def test_citation_builder_node_id_none_raises():
    """node_id가 None인 청크는 CitationBuilder가 명시적 오류를 발생시켜야 한다."""
    from dataclasses import dataclass
    from typing import Optional

    @dataclass
    class FakeChunk:
        document_id: str = str(uuid4())
        version_id: str = str(uuid4())
        node_id: Optional[str] = None
        source_text: str = "test"

    with pytest.raises(ValueError):
        CitationBuilder.from_retrieved_chunk(FakeChunk())


# ── 직렬화 ────────────────────────────────────────────────────

def test_citation_model_dump_includes_all_fields():
    c = Citation.from_chunk(uuid4(), uuid4(), uuid4(), "text", span_offset=10)
    d = c.model_dump()
    assert all(k in d for k in ["document_id", "version_id", "node_id", "span_offset", "content_hash"])


def test_citation_hash_length_always_64():
    texts = ["a", "a" * 1000, "한글", "🎉"]
    for t in texts:
        c = Citation.from_chunk(uuid4(), uuid4(), uuid4(), t)
        assert len(c.content_hash) == 64
```

### 4-2. 회귀 테스트: 100개 샘플

**파일:** `/backend/tests/regression/test_citation_100_samples.py`

```python
"""Citation 회귀 테스트 — 100개 샘플 문서.

모든 샘플 문서에 대해 Citation 생성 후 verify가 True여야 한다.
"""
import pytest
from uuid import uuid4
from app.schemas.citation import Citation
from app.services.retrieval.citation_builder import CitationBuilder


SAMPLE_TEXTS = [
    f"샘플 문서 {i}: 이것은 테스트용 텍스트입니다. 내용 번호 {i}번."
    for i in range(100)
]


@pytest.mark.parametrize("source_text", SAMPLE_TEXTS)
def test_citation_verify_true_for_sample(source_text: str):
    """생성된 citation이 같은 텍스트에 대해 verify=True를 반환해야 한다."""
    doc_id, ver_id, node_id = uuid4(), uuid4(), uuid4()
    citation = CitationBuilder.build(doc_id, ver_id, node_id, source_text)
    assert citation.verify(source_text) is True


@pytest.mark.parametrize("source_text", SAMPLE_TEXTS[:10])
def test_citation_verify_false_after_modification(source_text: str):
    """수정된 텍스트에 대해 verify=False를 반환해야 한다."""
    doc_id, ver_id, node_id = uuid4(), uuid4(), uuid4()
    citation = CitationBuilder.build(doc_id, ver_id, node_id, source_text)
    modified = source_text + " [수정됨]"
    assert citation.verify(modified) is False
```

### 4-3. S1 호환성 테스트

**파일:** `/backend/tests/compatibility/test_s1_search_compatibility.py`

```python
"""S1 ↔ S2 검색 결과 호환성 테스트.

S1 클라이언트: citation 필드 없는 응답을 정상 파싱
S2 클라이언트: citation 필드 있는 응답을 활용
"""
import pytest
from app.schemas.search import DocumentSearchResult


# ── S1 클라이언트 시뮬레이션 ──────────────────────────────────

def test_s1_client_ignores_citation_field():
    """S1 스키마 필드만 사용해도 파싱 가능해야 한다."""
    # S2 응답 (citation 포함)
    s2_response = {
        "document_id": "00000000-0000-0000-0000-000000000001",
        "title": "테스트 문서",
        "score": 0.95,
        "snippet": "검색 스니펫",
        "citation": {
            "document_id": "00000000-0000-0000-0000-000000000001",
            "version_id": "00000000-0000-0000-0000-000000000002",
            "node_id": "00000000-0000-0000-0000-000000000003",
            "span_offset": None,
            "content_hash": "a" * 64,
        },
    }
    # S1 클라이언트는 citation을 무시하고 기존 필드만 사용
    s1_fields = {k: v for k, v in s2_response.items() if k != "citation"}
    assert s1_fields["document_id"] == "00000000-0000-0000-0000-000000000001"
    assert s1_fields["score"] == 0.95


def test_s2_client_uses_citation_field():
    """S2 클라이언트는 citation 필드를 정상 파싱해야 한다."""
    from app.schemas.citation import Citation

    raw = {
        "document_id": "00000000-0000-0000-0000-000000000001",
        "version_id": "00000000-0000-0000-0000-000000000002",
        "node_id": "00000000-0000-0000-0000-000000000003",
        "span_offset": None,
        "content_hash": "b" * 64,
    }
    citation = Citation.model_validate(raw)
    assert str(citation.document_id) == "00000000-0000-0000-0000-000000000001"


def test_search_result_without_citation_is_valid():
    """citation 필드 없는 검색 결과도 유효해야 한다 (S1 호환)."""
    result = DocumentSearchResult(
        document_id="00000000-0000-0000-0000-000000000001",
        title="제목",
        score=0.8,
        snippet="스니펫",
        # citation 생략
    )
    assert result.citation is None
```

### 4-4. S1 → S2 마이그레이션 가이드

**파일:** `/docs/개발문서/S2/phase2/산출물/s1-s2-migration-guide.md`

```markdown
# S1 → S2 Citation 마이그레이션 가이드

## 변경 요약

S2 Phase 2에서 검색 응답에 `citation` 필드가 추가되었습니다.  
기존 S1 클라이언트는 이 필드를 무시하면 그대로 동작합니다.

## S1 클라이언트 (변경 불필요)

```json
// 기존 검색 결과 필드만 사용 — 정상 동작
{
  "document_id": "...",
  "title": "문서 제목",
  "score": 0.95,
  "snippet": "..."
}
```

## S2 클라이언트 (citation 활용)

```json
// citation 필드 추가 — S2 클라이언트가 활용
{
  "document_id": "...",
  "title": "문서 제목",
  "score": 0.95,
  "snippet": "...",
  "citation": {
    "document_id": "...",
    "version_id": "...",
    "node_id": "...",
    "span_offset": null,
    "content_hash": "sha256hex..."
  }
}
```

## Citation 검증 방법

```http
GET /api/v1/citations/{document_id}/versions/{version_id}/nodes/{node_id}/verify
  ?content_hash={content_hash}
```

응답:
```json
{ "verified": true, "modified": false, "original_text": "...", "citation": {...} }
```

## 원문 조회

```http
GET /api/v1/citations/{document_id}/versions/{version_id}/nodes/{node_id}/content
```

## 주의 사항

- `citation.version_id`는 검색 시점의 버전을 가리킵니다.
  문서가 개정되면 `modified: true`가 반환됩니다.
- `content_hash`는 SHA-256(UTF-8 인코딩 원문)입니다.
```

## 5. 산출물

1. **단위 테스트 보강** (`/backend/tests/unit/services/test_citation_edge_cases.py`)
2. **회귀 테스트 100개** (`/backend/tests/regression/test_citation_100_samples.py`)
3. **S1 호환성 테스트** (`/backend/tests/compatibility/test_s1_search_compatibility.py`)
4. **마이그레이션 가이드** (`/docs/개발문서/S2/phase2/산출물/s1-s2-migration-guide.md`)

## 6. 완료 기준

1. 단위 테스트 전체 통과 (엣지 케이스 포함)
2. 회귀 테스트 100개 샘플 모두 `verified: true`
3. 수정된 텍스트에 대해 `verified: false` 감지
4. S1 클라이언트 시뮬레이션 테스트 통과 (citation 없이도 정상 파싱)
5. S2 클라이언트 시뮬레이션 테스트 통과 (citation 활용)
6. 마이그레이션 가이드 문서 작성 완료
7. 테스트 커버리지 `citation.*` 모듈 ≥ 90%

## 7. 작업 지침

### 지침 7-1. 픽스처 재사용

`test_citation.py` (Task 2-1)의 픽스처(`DOC_ID`, `SOURCE` 등)를 `conftest.py`로 이동하여 재사용한다. 중복 픽스처 정의를 피한다.

### 지침 7-2. 회귀 테스트 데이터 자동 생성

100개 샘플은 하드코딩하지 않고 list comprehension으로 생성한다. 향후 샘플 수를 늘리거나 실제 DB 문서를 활용할 때 교체하기 쉽게 설계한다.

### 지침 7-3. 호환성 테스트 독립성

S1 호환성 테스트는 실제 DB 없이 동작해야 한다. Pydantic 스키마 수준의 파싱 테스트이므로 DB 의존성을 제거한다.
