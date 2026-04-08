# Task 10-2. DocumentType별 청킹 전략 설계

## 1. 작업 목적

DocumentType에 따라 최적화된 청킹 전략을 설계하고, 설정 기반으로 관리할 수 있는 구조를 확립한다.

이 작업의 목표는 다음과 같다.

- 구조 단위 청킹(node_based) 알고리즘 설계
- DocumentType 설정에 `chunking_config` 항목 추가 설계
- 청크 크기 조정(merge/split) 기준 정의
- 부모 컨텍스트 포함 방식 정의
- 청크 메타데이터 스키마 확정

---

## 2. 작업 범위

### 포함 범위

- chunking_config 스키마 설계 (DocumentType 설정 확장)
- node_based 청킹 알고리즘 설계
- 청크 크기 제한 및 merge/split 기준 정의
- 부모 컨텍스트 주입 방식 정의
- 청크 메타데이터 스키마 확정
- 버전별 청킹 대상 결정 정책

### 제외 범위

- 구현 코드 작성 (Task 10-5에서 다룸)
- DB 스키마 (Task 10-3에서 다룸)

---

## 3. 선행 조건

- Task 10-1 (벡터화 파이프라인 아키텍처 설계) 완료
- Phase 1: Node 트리 구조 및 node_type 이해
- Phase 8: Task 8-7의 DocumentType search_config 패턴 참고

---

## 4. 주요 설계 대상

### 4-1. chunking_config 스키마

DocumentType 설정에 추가될 청킹 설정:

```json
{
  "chunking_config": {
    "strategy": "node_based",
    "max_chunk_tokens": 512,
    "min_chunk_tokens": 50,
    "overlap_tokens": 50,
    "include_parent_context": true,
    "parent_context_depth": 2,
    "index_version_policy": "published_only",
    "exclude_node_types": ["metadata", "attachment"],
    "merge_strategy": "merge_siblings"
  }
}
```

#### 설정 항목 설명

| 항목 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `strategy` | string | `node_based` | 청킹 전략 (`node_based` / `fixed_size`) |
| `max_chunk_tokens` | int | 512 | 청크 최대 토큰 수 |
| `min_chunk_tokens` | int | 50 | 청크 최소 토큰 수 (이하면 병합) |
| `overlap_tokens` | int | 50 | 인접 청크 간 오버랩 토큰 수 |
| `include_parent_context` | bool | true | 부모 노드 제목을 청크 앞에 포함 |
| `parent_context_depth` | int | 2 | 포함할 부모 컨텍스트 깊이 |
| `index_version_policy` | string | `published_only` | 벡터화 대상 버전 정책 |
| `exclude_node_types` | string[] | [] | 청킹에서 제외할 노드 타입 |
| `merge_strategy` | string | `merge_siblings` | 소형 청크 병합 전략 |

---

### 4-2. node_based 청킹 알고리즘

Node 트리를 재귀적으로 탐색하여 청크를 생성하는 알고리즘:

#### 기본 원칙

1. 리프 노드 또는 일정 크기 이하의 섹션 → 청크 단위
2. 섹션이 `max_chunk_tokens` 초과 → 내부 하위 노드로 분할
3. 섹션이 `min_chunk_tokens` 미달 → 인접 형제 노드와 병합

#### 처리 순서

```
for each node in preorder_traversal(tree):
  text = extract_text(node)
  token_count = count_tokens(text)

  if token_count > max_chunk_tokens:
    if node has children:
      → 하위 노드들을 재귀적으로 처리 (split)
    else:
      → 고정 크기로 분할 (fixed_size fallback)
  elif token_count < min_chunk_tokens:
    → 다음 형제 노드와 병합 대상으로 표시
  else:
    → 청크 생성
```

---

### 4-3. 부모 컨텍스트 주입 방식

`include_parent_context = true`일 때 청크 앞에 부모 계층 제목을 포함:

#### 예시

원본 노드 (node_path: `["정보보안 기본 정책", "2. 정책 본문", "2.3 접근통제"]`):
```
본 시스템 접근은 역할 기반으로 처리된다.
```

부모 컨텍스트 주입 후:
```
[문서: 정보보안 기본 정책]
[섹션: 2. 정책 본문 > 2.3 접근통제]
본 시스템 접근은 역할 기반으로 처리된다.
```

이 방식으로 청크만 검색해도 어디서 나온 내용인지 임베딩에 반영됨.

#### parent_context_depth

- `1`: 직접 부모 제목만 포함
- `2`: 부모 + 조부모 제목 포함 (기본 권장)
- `0`: 부모 컨텍스트 없음

---

### 4-4. 버전별 청킹 대상 정책 (index_version_policy)

| 정책 값 | 설명 | 사용 사례 |
|--------|------|---------|
| `published_only` | Published 상태 버전만 벡터화 (기본) | 안정된 콘텐츠만 RAG에 활용 |
| `latest` | 최신 버전 무조건 벡터화 | 초안도 검색 대상에 포함 |
| `all` | 모든 버전 벡터화 | 버전별 검색 필요 시 |

기본값 `published_only`로 RAG 품질 보장.

---

### 4-5. DocumentType별 권장 설정 예시

| DocumentType | 전략 | max_tokens | 특이사항 |
|-------------|------|-----------|---------|
| POLICY (정책) | node_based | 512 | 조항 단위로 청킹, 조항 번호를 컨텍스트에 포함 |
| MANUAL (매뉴얼) | node_based | 400 | 절차 단계별 청킹 |
| REPORT (보고서) | node_based | 600 | 섹션 단위 청킹 |
| FAQ | node_based | 256 | Q&A 쌍 단위 청킹 (작은 크기) |

이 설정은 기본값이며 Admin UI에서 조정 가능.

---

### 4-6. 청크 메타데이터 스키마

각 청크에 포함할 메타데이터:

```json
{
  "document_id": "doc_abc123",
  "version_id": "ver_002",
  "node_id": "node_xyz",
  "chunk_index": 5,
  "chunk_total": 23,
  "source_text": "본 시스템 접근은...",
  "context_text": "[문서: ...][섹션: ...]본 시스템 접근은...",
  "node_path": ["정보보안 기본 정책", "2. 정책 본문", "2.3 접근통제"],
  "node_type": "paragraph",
  "document_type": "POLICY",
  "document_status": "PUBLISHED",
  "token_count": 87,
  "accessible_roles": ["ADMIN", "VIEWER"],
  "accessible_org_ids": ["org_001"],
  "is_public": false
}
```

---

## 5. 산출물

1. chunking_config 스키마 정의서
2. node_based 청킹 알고리즘 설계서
3. 부모 컨텍스트 주입 방식 정의서
4. 버전별 청킹 대상 정책 문서
5. 청크 메타데이터 스키마 정의서

---

## 6. 완료 기준

- chunking_config 스키마가 DocumentType 설정 확장으로 정의되어 있다
- node_based 청킹 알고리즘이 merge/split 기준과 함께 설계되어 있다
- 부모 컨텍스트 주입 방식이 구체적으로 기술되어 있다
- 버전별 청킹 대상 정책이 정의되어 있다
- 청크 메타데이터 스키마가 확정되어 있다

---

## 7. Codex 작업 지침

- 청킹 설정은 하드코딩 금지 — DocumentType 설정(chunking_config)에서 읽어오는 구조로 설계한다
- CLAUDE.md의 "문서 타입은 하드코딩 금지" 원칙을 청킹 전략에서도 철저히 준수한다
- 부모 컨텍스트 포함으로 청크의 임베딩 품질이 향상된다는 근거를 설계서에 명시한다
- `published_only` 정책이 기본값인 이유 (RAG 품질 보장)를 명시한다
- Phase 12 (DocumentType 플러그인)에서 새 타입 추가 시 chunking_config도 확장 가능함을 명시한다
