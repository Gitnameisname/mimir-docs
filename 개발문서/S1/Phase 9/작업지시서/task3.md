# Task 9-3. 변경 비교 API 설계

## 1. 작업 목적

UI와 외부 시스템 모두가 사용할 수 있는 변경 비교 API 인터페이스를 설계한다.

이 작업의 목표는 다음과 같다.

- diff 엔드포인트 목록과 파라미터 정의
- DiffResult 응답 스키마 표준화
- 변경 요약(summary) 전용 경량 응답 구조 설계
- 캐싱 정책 및 Cache-Control 헤더 정의
- 권한 검증 방식 명시

---

## 2. 작업 범위

### 포함 범위

- diff 엔드포인트 목록 및 역할 정의
- 쿼리 파라미터 표준 설계
- DiffResult 전체 응답 스키마 정의 (OpenAPI 수준)
- DiffSummary 경량 응답 스키마 정의
- 오류 응답 정의
- 캐싱 전략 및 Cache-Control 정책

### 제외 범위

- 구현 코드 작성
- 알고리즘 상세 (Task 9-2에서 다룸)

---

## 3. 선행 조건

- Task 9-1 (변경 비교 시스템 아키텍처 설계) 완료
- Task 9-2 (트리 Diff 알고리즘 설계) 완료
- Phase 3: 공통 API 규약 이해

---

## 4. 주요 설계 대상

### 4-1. 엔드포인트 목록

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/documents/{doc_id}/versions/{v_id}/diff` | 직전 버전 대비 diff |
| GET | `/documents/{doc_id}/versions/{v1_id}/diff/{v2_id}` | 두 버전 간 diff |
| GET | `/documents/{doc_id}/versions/{v_id}/diff/summary` | 직전 버전 대비 변경 요약만 |
| GET | `/documents/{doc_id}/versions/{v1_id}/diff/{v2_id}/summary` | 두 버전 간 변경 요약만 |

---

### 4-2. 쿼리 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|---------|------|--------|------|
| `inline_diff` | boolean | false | MODIFIED 노드에 텍스트 인라인 diff 포함 여부 |
| `include_unchanged` | boolean | false | UNCHANGED 노드 포함 여부 |
| `max_inline_length` | integer | 5000 | 인라인 diff 생성 최대 텍스트 길이 |

---

### 4-3. DiffResult 전체 응답 스키마

```json
{
  "document_id": "doc_abc123",
  "version_a": {
    "id": "ver_001",
    "number": "1.0",
    "created_at": "2025-03-01T00:00:00Z",
    "created_by": "user_kim",
    "note": "초안 작성"
  },
  "version_b": {
    "id": "ver_002",
    "number": "1.1",
    "created_at": "2025-03-15T00:00:00Z",
    "created_by": "user_park",
    "note": "보안 정책 강화 반영"
  },
  "summary": {
    "total_added": 2,
    "total_deleted": 1,
    "total_modified": 3,
    "total_moved": 0,
    "total_unchanged": 12,
    "description": "섹션 2개 추가, 섹션 1개 삭제, 섹션 3개 수정됨"
  },
  "nodes": [
    {
      "node_id": "node_xyz",
      "change_type": "MODIFIED",
      "before": {
        "node_id": "node_xyz",
        "node_type": "paragraph",
        "title": null,
        "content": "기존 내용입니다.",
        "parent_id": "node_abc",
        "order": 2
      },
      "after": {
        "node_id": "node_xyz",
        "node_type": "paragraph",
        "title": null,
        "content": "변경된 내용입니다.",
        "parent_id": "node_abc",
        "order": 2
      },
      "inline_diff": [
        { "type": "unchanged", "text": "" },
        { "type": "deleted", "text": "기존" },
        { "type": "added", "text": "변경된" },
        { "type": "unchanged", "text": " 내용입니다." }
      ],
      "move_info": null
    },
    {
      "node_id": "node_new1",
      "change_type": "ADDED",
      "before": null,
      "after": {
        "node_id": "node_new1",
        "node_type": "section",
        "title": "4. 신규 정책",
        "content": "...",
        "parent_id": null,
        "order": 4
      },
      "inline_diff": null,
      "move_info": null
    }
  ]
}
```

---

### 4-4. DiffSummary 경량 응답 스키마 (`/diff/summary`)

전체 diff 없이 변경 요약만 필요한 경우 (목록 화면, 워크플로 배너 등):

```json
{
  "document_id": "doc_abc123",
  "version_a": { "id": "ver_001", "number": "1.0" },
  "version_b": { "id": "ver_002", "number": "1.1" },
  "summary": {
    "total_added": 2,
    "total_deleted": 1,
    "total_modified": 3,
    "total_moved": 0,
    "total_unchanged": 12,
    "description": "섹션 2개 추가, 섹션 1개 삭제, 섹션 3개 수정됨",
    "changed_sections": [
      { "node_id": "node_s2", "title": "2. 정책 본문", "change_type": "MODIFIED" },
      { "node_id": "node_new1", "title": "4. 신규 정책", "change_type": "ADDED" }
    ]
  }
}
```

---

### 4-5. 오류 응답 정의

| HTTP 코드 | 오류 코드 | 상황 |
|-----------|----------|------|
| 400 | `SAME_VERSION` | v1_id와 v2_id가 동일 |
| 403 | `VERSION_ACCESS_DENIED` | 버전 중 하나 이상에 접근 권한 없음 |
| 404 | `VERSION_NOT_FOUND` | 지정한 버전 ID가 존재하지 않음 |
| 404 | `NO_PREVIOUS_VERSION` | 직전 버전이 없는 첫 버전에서 diff 요청 |
| 422 | `DIFF_TOO_LARGE` | 노드 수 초과로 diff 불가 (비동기 처리 안내) |
| 500 | `DIFF_COMPUTATION_ERROR` | diff 계산 중 내부 오류 |

---

### 4-6. 캐싱 전략

#### HTTP 캐싱 헤더

버전은 불변이므로 diff 결과는 영속 캐시 가능:

```
Cache-Control: public, max-age=86400, immutable
ETag: "{version_a_id}-{version_b_id}"
```

단, 버전 접근 권한이 사용자마다 다를 수 있으므로:
- `Cache-Control: private, max-age=3600` 사용 (개인 캐시)
- 또는 ETag 기반 조건부 요청으로 서버 처리 최소화

#### 서버 사이드 캐싱

- 캐시 키: `diff:{min(v1,v2)}:{max(v1,v2)}`
- TTL: 영속 (버전 불변) 또는 문서 삭제/보존 정책에 따라 설정
- 캐시 저장소: Redis 또는 인메모리 (요청 빈도에 따라 결정)

---

### 4-7. 인라인 diff 포함 여부 결정

`inline_diff=true` 파라미터:

- 기본값: false (diff 계산 비용 절약)
- true 시: MODIFIED 노드의 `inline_diff` 필드 채워짐
- 텍스트 길이가 `max_inline_length` 초과 시 인라인 diff 생략 + 경고 포함

---

## 5. 산출물

1. 변경 비교 API 명세서 (OpenAPI 3.0 기반)
2. DiffResult 전체 스키마 정의서
3. DiffSummary 경량 스키마 정의서
4. 오류 응답 코드 정의서
5. 캐싱 전략 문서

---

## 6. 완료 기준

- diff 엔드포인트 목록과 역할이 명확히 정의되어 있다
- DiffResult 전체 응답 스키마가 정의되어 있다
- DiffSummary 경량 응답 스키마가 정의되어 있다
- 인라인 diff 포함 파라미터와 동작이 정의되어 있다
- 캐싱 전략이 버전 불변성을 활용하여 설계되어 있다
- 오류 응답이 모든 케이스에 대해 정의되어 있다

---

## 7. Codex 작업 지침

- 코드 작성 금지 (설계 중심, JSON 스키마 예시는 구조 설명 목적으로 허용)
- Phase 3의 공통 API 규약과 일관성을 유지한다
- DiffSummary 경량 응답은 워크플로 검토 화면에서 빠른 렌더링을 위한 것임을 설계 목적으로 명시한다
- 인라인 diff는 선택적으로 제공하여 불필요한 계산 비용을 피하는 설계임을 명시한다
- 캐싱 전략에서 접근 권한이 사용자마다 다를 수 있음을 반드시 고려한다
