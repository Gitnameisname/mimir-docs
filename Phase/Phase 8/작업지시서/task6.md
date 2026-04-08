# Task 8-6. 노드 단위 검색 구현

## 1. 작업 목적

문서 내 특정 섹션(Node) 수준까지 검색이 가능하도록 노드 단위 검색 기능을 구현한다.

이 작업의 목표는 다음과 같다.

- Node 테이블에 FTS 인덱스를 적용하여 섹션 단위 검색 지원
- 노드 검색 결과에 문서 컨텍스트와 경로(breadcrumb)를 포함하여 반환
- 부모 문서의 권한을 상속하여 권한 기반 필터링 적용
- 검색 결과에서 해당 노드 위치로 직접 이동할 수 있는 앵커 정보 제공

---

## 2. 작업 범위

### 포함 범위

- Node 테이블 FTS 인덱스 구현
- 노드 단위 검색 서비스 로직 구현
- 노드 검색 결과 응답 스키마 구현 (문서 컨텍스트, 경로 포함)
- 부모 문서 권한 상속 검증 구현
- 동일 문서 내 다중 매칭 노드 그룹화 로직

### 제외 범위

- 검색 결과 UI (Task 8-8에서 다룸)
- 성능 튜닝 (Task 8-10에서 다룸)

---

## 3. 선행 조건

- Task 8-2 (검색 인덱스 구조 설계) 완료
- Task 8-4 (권한 기반 검색 필터링 설계) 완료
- Task 8-5 (전문 검색 구현) 완료 (FTSSearchProvider 기반 위에 구현)
- Phase 1: Node 트리 구조 이해

---

## 4. 주요 구현 대상

### 4-1. Node 테이블 FTS 인덱스

#### 컬럼 추가

- `content_text text`: 노드 내용 평문 추출 저장
- `search_vector tsvector`: FTS 인덱스 컬럼

#### GIN 인덱스

```sql
CREATE INDEX idx_nodes_search_vector ON nodes USING GIN (search_vector);
```

#### 트리거 설계

- 노드 제목(title/heading) 변경 시 search_vector 갱신 (가중치 A)
- 노드 내용(content) 변경 시 content_text 추출 후 search_vector 갱신 (가중치 B)
- 노드 삭제 또는 부모 버전/문서 삭제 시 search_vector NULL 처리

---

### 4-2. 노드 경로(Breadcrumb) 생성

노드 검색 결과에서 문서 내 위치를 나타내는 경로를 생성한다.

#### 경로 구성

```
["1. 개요", "2. 정책 본문", "2.3 접근통제 정책"]
```

#### 구현 방식

- Node 트리는 부모-자식 관계로 구성되어 있으므로 재귀 쿼리(WITH RECURSIVE)로 경로 추출
- 루트 노드부터 현재 노드까지의 제목을 배열로 구성
- 경로 깊이 제한 설정 (기본: 최대 5단계)

#### 앵커(Anchor) 정보

UI에서 해당 노드 위치로 직접 이동하기 위한 앵커 정보 포함:
- `node_id`: 노드 고유 ID
- `anchor`: URL 앵커용 해시 (`#node-{id}` 또는 노드 제목 기반 슬러그)

---

### 4-3. 부모 문서 권한 상속 검증

노드 검색 결과에서 부모 Document의 접근 권한을 검증한다:

#### 쿼리 구조 개념

```
Node 검색 결과
  JOIN versions ON nodes.version_id = versions.id
  JOIN documents ON versions.document_id = documents.id
  AND documents 접근 권한 조건 (ACL)
  AND documents.status IN (허용된 상태 목록)
```

- 노드 자체의 권한이 아닌 부모 문서 권한을 기준으로 필터링
- 여러 버전의 노드가 있을 때 검색 대상 버전 범위 적용 (최신 Published 버전 우선)

---

### 4-4. 동일 문서 내 다중 매칭 노드 그룹화

하나의 문서에서 여러 노드가 매칭될 경우의 처리:

#### `GET /search/nodes` 응답 (기본)

- 노드 단위 개별 결과 반환 (그룹화 없음)
- 동일 문서의 여러 노드가 각각 독립적인 결과로 표시

#### `GET /search/documents`에서의 노드 매칭 정보

- `matched_nodes_count`: 해당 문서에서 매칭된 노드 수 포함
- 상위 N개 매칭 노드의 스니펫 포함 (선택적)

#### UI 그룹화 (Task 8-8과 연계)

- 통합 검색 결과에서 동일 문서의 노드 매칭은 해당 문서 결과 항목 아래 펼침(expand)으로 표시

---

### 4-5. 노드 검색 결과 응답 구현

Task 8-3에서 설계한 NodeSearchResult 스키마를 실제로 구현:

```json
{
  "node_id": "node_xyz789",
  "node_title": "2.3 접근통제 정책",
  "node_path": ["1. 개요", "2. 정책 본문", "2.3 접근통제 정책"],
  "anchor": "#node-xyz789",
  "node_type": "section",
  "snippet": "...시스템 <mark>접근통제</mark>는 역할 기반으로...",
  "score": 0.88,
  "document": {
    "id": "doc_abc123",
    "title": "정보보안 기본 정책",
    "type": "POLICY",
    "status": "PUBLISHED",
    "version": "2.1"
  }
}
```

---

### 4-6. 특정 문서 내 노드 검색

`GET /search/nodes?document_id={id}` 파라미터를 통해 특정 문서 내에서만 노드 검색:

- 문서 상세 화면에서 "이 문서 내 검색" 기능에 활용
- 해당 문서 접근 권한 확인 후 처리
- 결과 수가 적으므로 페이지네이션 없이 전체 반환 가능 (상한 설정 필요)

---

## 5. 산출물

1. Node 테이블 DB 마이그레이션 (content_text, search_vector, GIN 인덱스)
2. 노드 인덱스 갱신 트리거 구현
3. 노드 검색 서비스 구현 (SearchService.searchNodes)
4. 노드 경로(Breadcrumb) 생성 로직 구현
5. `GET /search/nodes` API 엔드포인트 구현

---

## 6. 완료 기준

- Node 테이블에 search_vector 컬럼과 GIN 인덱스가 적용되어 있다
- 노드 내용 변경 시 search_vector가 자동 갱신된다
- `GET /search/nodes` API가 동작하며 결과에 경로와 스니펫이 포함된다
- 권한 없는 문서의 노드가 검색 결과에 포함되지 않는다
- 특정 문서 내 노드 검색이 동작한다
- 검색 결과에서 노드 위치로 이동할 수 있는 앵커 정보가 포함된다

---

## 7. Codex 작업 지침

- Task 8-5 (전문 검색 구현)의 FTSSearchProvider 구조 위에 구현한다
- 노드 경로 생성 시 재귀 쿼리의 최대 깊이를 제한하여 무한 루프를 방지한다
- 부모 문서 권한 상속 검증은 생략하거나 약화시킬 수 없다
- 노드 수는 문서 수보다 훨씬 많을 수 있으므로 인덱스 크기와 갱신 비용을 항상 고려한다
- content_text 평문 추출 로직은 Task 8-5와 동일한 방식을 사용하여 일관성을 유지한다
