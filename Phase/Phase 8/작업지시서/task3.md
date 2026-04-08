# Task 8-3. 검색 API 설계

## 1. 작업 목적

UI와 외부 시스템 모두가 사용할 수 있는 검색 API 인터페이스를 설계한다.

이 작업의 목표는 다음과 같다.

- 통합 검색 및 엔티티별 검색 엔드포인트 정의
- 쿼리 파라미터, 필터, 정렬, 페이지네이션 규약 확정
- SearchResult 응답 스키마 표준화 (스니펫 포함)
- Phase 10 벡터 검색 도입 이후에도 동일하게 유지될 안정적인 API 계약 설계

---

## 2. 작업 범위

### 포함 범위

- 검색 엔드포인트 목록 및 역할 정의
- 쿼리 파라미터 표준 설계
- 요청/응답 스키마 정의 (OpenAPI 수준)
- 스니펫 응답 구조 설계
- 오류 응답 정의
- 인증/인가 적용 방식 정의

### 제외 범위

- 실제 API 구현 코드
- 검색 쿼리 내부 로직 (Task 8-5에서 다룸)
- 권한 필터링 구현 상세 (Task 8-4에서 다룸)

---

## 3. 선행 조건

- Task 8-1 (검색 시스템 아키텍처 설계) 완료
- Phase 3: 공통 API 규약 (페이지네이션, 응답 형식, 오류 코드) 이해

---

## 4. 주요 설계 대상

### 4-1. 엔드포인트 목록

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/search` | 통합 검색 (Document + Node 혼합) |
| GET | `/search/documents` | 문서 단위 검색 |
| GET | `/search/nodes` | 노드(섹션) 단위 검색 |
| POST | `/search/reindex` | 수동 재인덱싱 트리거 (Admin 전용) |

---

### 4-2. 공통 쿼리 파라미터

모든 검색 엔드포인트에 공통으로 적용되는 파라미터:

| 파라미터 | 타입 | 필수 | 설명 | 예시 |
|---------|------|------|------|------|
| `q` | string | Y | 검색어 (최소 1자) | `q=보안정책` |
| `type` | string[] | N | DocumentType 필터 (다중) | `type=POLICY&type=MANUAL` |
| `status` | string[] | N | 문서 상태 필터 | `status=PUBLISHED` |
| `from` | date | N | 생성일 시작 범위 | `from=2025-01-01` |
| `to` | date | N | 생성일 종료 범위 | `to=2025-12-31` |
| `sort` | string | N | 정렬 기준 | `sort=relevance` (기본) |
| `order` | string | N | 정렬 방향 | `order=desc` (기본) |
| `page` | integer | N | 페이지 번호 (1부터) | `page=1` |
| `limit` | integer | N | 페이지당 결과 수 (기본 20, 최대 100) | `limit=20` |
| `snippet` | boolean | N | 스니펫 포함 여부 (기본 true) | `snippet=true` |

#### `sort` 가능 값

| 값 | 설명 |
|----|------|
| `relevance` | 검색 관련도 순 (기본) |
| `created_at` | 생성일 순 |
| `updated_at` | 수정일 순 |
| `title` | 제목 알파벳 순 |

---

### 4-3. 통합 검색 엔드포인트 (`GET /search`)

문서와 노드를 함께 검색하여 혼합 결과를 반환한다.

#### 추가 파라미터

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| `include` | string[] | 포함할 엔티티 타입 (`documents`, `nodes`) |

#### 응답 스키마

```json
{
  "query": "보안정책",
  "total": 42,
  "page": 1,
  "limit": 20,
  "results": [
    {
      "type": "document",
      "document": { ... },
      "snippet": "...",
      "score": 0.85
    },
    {
      "type": "node",
      "node": { ... },
      "document": { ... },
      "snippet": "...",
      "score": 0.72
    }
  ]
}
```

---

### 4-4. 문서 단위 검색 엔드포인트 (`GET /search/documents`)

#### 응답 스키마

```json
{
  "query": "보안정책",
  "total": 35,
  "page": 1,
  "limit": 20,
  "results": [
    {
      "id": "doc_abc123",
      "title": "정보보안 기본 정책",
      "type": "POLICY",
      "status": "PUBLISHED",
      "version": "2.1",
      "created_at": "2025-03-01T00:00:00Z",
      "updated_at": "2025-04-01T00:00:00Z",
      "metadata": {
        "department": "보안팀",
        "tags": ["보안", "정책"]
      },
      "snippet": "...본 정책은 <mark>보안</mark>을 위한 기본 <mark>정책</mark>을 정의하며...",
      "score": 0.91,
      "matched_nodes_count": 3
    }
  ]
}
```

#### 핵심 응답 필드 정의

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | string | 문서 ID |
| `title` | string | 문서 제목 |
| `type` | string | DocumentType |
| `status` | string | 문서 상태 |
| `snippet` | string | 매칭 컨텍스트 스니펫 (HTML, `<mark>` 태그로 하이라이팅) |
| `score` | float | 검색 관련도 점수 (0~1) |
| `matched_nodes_count` | integer | 이 문서에서 매칭된 노드 수 |

---

### 4-5. 노드 단위 검색 엔드포인트 (`GET /search/nodes`)

#### 추가 파라미터

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| `document_id` | string | 특정 문서 내 노드만 검색 |

#### 응답 스키마

```json
{
  "query": "접근통제",
  "total": 18,
  "page": 1,
  "limit": 20,
  "results": [
    {
      "node_id": "node_xyz789",
      "node_title": "3.2 접근통제 정책",
      "node_path": ["1. 개요", "3. 정책 본문", "3.2 접근통제 정책"],
      "node_type": "section",
      "snippet": "...시스템 <mark>접근통제</mark>는 역할 기반으로...",
      "score": 0.88,
      "document": {
        "id": "doc_abc123",
        "title": "정보보안 기본 정책",
        "type": "POLICY",
        "status": "PUBLISHED"
      }
    }
  ]
}
```

#### 핵심 응답 필드 정의

| 필드 | 타입 | 설명 |
|------|------|------|
| `node_id` | string | 노드 ID |
| `node_title` | string | 노드 제목/헤딩 |
| `node_path` | string[] | 문서 내 경로 (breadcrumb) |
| `snippet` | string | 노드 내 매칭 스니펫 |
| `document` | object | 부모 문서 요약 정보 |

---

### 4-6. 스니펫 응답 설계

#### 스니펫 생성 방식

- PostgreSQL `ts_headline()` 함수 활용
- 매칭 키워드 전후 컨텍스트 포함
- 키워드는 `<mark>` 태그로 감싸서 반환

#### 스니펫 설정

| 설정 | 기본값 | 설명 |
|------|--------|------|
| 최대 길이 | 200자 | 스니펫 최대 문자 수 |
| 최대 단어 수 | 30개 | 스니펫 최대 단어 수 |
| 생략 기호 | `...` | 생략 시 사용할 문자 |

#### 스니펫 미포함 응답

- `snippet=false` 파라미터로 스니펫 제외 가능
- 목록 화면 등 대량 조회 시 성능 최적화 목적

---

### 4-7. 오류 응답 정의

| HTTP 코드 | 오류 코드 | 상황 |
|-----------|----------|------|
| 400 | `INVALID_QUERY` | 검색어가 비어있거나 유효하지 않음 |
| 400 | `INVALID_PARAMETER` | 파라미터 형식 오류 |
| 401 | `UNAUTHORIZED` | 인증되지 않은 요청 |
| 403 | `FORBIDDEN` | 검색 권한 없음 |
| 422 | `QUERY_TOO_LONG` | 검색어 최대 길이 초과 (기본 500자) |
| 500 | `SEARCH_ENGINE_ERROR` | 검색 엔진 내부 오류 |

---

### 4-8. 인증/인가 적용

- 모든 검색 API는 JWT Bearer 토큰 필수
- 인증된 사용자의 권한에 따라 검색 결과 자동 필터링
- 권한 필터링 상세는 Task 8-4에서 정의

---

### 4-9. API 사용 예시 (설계 수준)

#### 기본 문서 검색

```
GET /search/documents?q=보안정책&type=POLICY&status=PUBLISHED&page=1&limit=20
Authorization: Bearer {token}
```

#### 노드 단위 검색

```
GET /search/nodes?q=접근통제&document_id=doc_abc123
Authorization: Bearer {token}
```

#### 통합 검색

```
GET /search?q=보안&include=documents&include=nodes&sort=relevance
Authorization: Bearer {token}
```

---

## 5. 산출물

1. 검색 API 명세서 (OpenAPI 3.0 기반 YAML/JSON)
2. SearchResult 스키마 정의서 (Document / Node / 통합)
3. 스니펫 응답 구조 정의서
4. 오류 응답 코드 정의서

---

## 6. 완료 기준

- 검색 엔드포인트 목록과 역할이 명확히 정의되어 있다
- 모든 쿼리 파라미터가 타입/필수 여부/설명과 함께 정의되어 있다
- Document / Node / 통합 검색 응답 스키마가 정의되어 있다
- 스니펫 포함 응답 구조가 구체적으로 기술되어 있다
- 오류 응답이 정의되어 있다
- Phase 10 이후에도 API 계약이 유지될 수 있는 구조임이 확인되어 있다

---

## 7. Codex 작업 지침

- 코드 작성 금지 (설계 중심, JSON 스키마 예시는 구조 설명 목적으로 허용)
- Phase 3의 공통 API 규약(페이지네이션, 응답 형식)과 일관성을 유지한다
- 스니펫 HTML 마크업(`<mark>`)은 XSS 방지를 위해 반드시 서버 측에서 안전하게 생성함을 명시한다
- API 계약의 안정성을 최우선으로 하여 Phase 10 이후에도 변경되지 않을 구조를 설계한다
- 외부 AI 에이전트가 사용할 수 있는 API임을 고려하여 명확하고 예측 가능한 응답 구조를 설계한다
