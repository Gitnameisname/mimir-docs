# Task 9-4. 트리 Diff 알고리즘 구현

## 1. 작업 목적

Task 9-2에서 설계한 노드 ID 기반 트리 Diff 알고리즘을 실제로 구현한다.

이 작업의 목표는 다음과 같다.

- NodeDiffer 서비스 구현 (변경 유형 분류 및 DiffResult 생성)
- 다양한 변경 시나리오를 커버하는 단위 테스트 작성
- diff API 엔드포인트와 연결

---

## 2. 작업 범위

### 포함 범위

- NodeDiffer 서비스 구현
- 변경 유형 분류 로직 구현 (ADDED/DELETED/MODIFIED/MOVED/UNCHANGED)
- 재귀 트리 탐색 구현
- MOVE 감지 구현
- 엣지 케이스 처리 구현
- 단위 테스트 작성

### 제외 범위

- 텍스트 인라인 diff (Task 9-5에서 다룸)
- 변경 요약 생성 (Task 9-6에서 다룸)
- API 엔드포인트 구현 연결은 Task 9-3 API 설계 기반으로 처리

---

## 3. 선행 조건

- Task 9-1 (변경 비교 시스템 아키텍처 설계) 완료
- Task 9-2 (트리 Diff 알고리즘 설계) 완료
- Task 9-3 (변경 비교 API 설계) 완료

---

## 4. 주요 구현 대상

### 4-1. NodeDiffer 서비스 구조

#### 서비스 인터페이스

```
NodeDiffer
  .diff(nodes_a: Node[], nodes_b: Node[]) → NodeDiff[]
  .diff_with_summary(nodes_a, nodes_b) → { nodes: NodeDiff[], summary: DiffSummary }
```

#### 내부 처리 단계

1. `nodes_a`, `nodes_b`를 `{node_id → Node}` 맵으로 변환
2. 전체 ID 집합(union) 순회
3. 각 ID에 대해 변경 유형 분류 (`classify_change`)
4. MODIFIED 노드에 대해 위치 변경 여부 추가 확인 (복합 케이스)
5. NodeDiff 목록 반환

---

### 4-2. 변경 유형 분류 로직

```
classify_change(node_id, map_a, map_b) → NodeDiff

  node_a = map_a.get(node_id) or None
  node_b = map_b.get(node_id) or None

  if node_a is None and node_b is not None:
    return ADDED

  if node_a is not None and node_b is None:
    return DELETED

  // 양쪽 모두 존재
  content_changed = normalize(node_a.content) != normalize(node_b.content)
  position_changed = (node_a.parent_id != node_b.parent_id) or (node_a.order != node_b.order)

  if not content_changed and not position_changed:
    return UNCHANGED

  if content_changed:
    // 위치 변경 여부와 관계없이 MODIFIED (move_info 포함 가능)
    return MODIFIED with move_info if position_changed

  if position_changed and not content_changed:
    return MOVED
```

---

### 4-3. 내용 정규화 함수

두 노드의 내용을 비교하기 전에 정규화:

- JSON 직렬화 시 키 순서 정렬
- 공백/줄바꿈 후행 제거
- null과 빈 문자열 동일 취급 여부 설정 가능 (기본: 다르게 취급)

---

### 4-4. MOVE 감지 구현

MOVED 노드의 move_info 채우기:

```
MoveInfo
  old_parent_id: string | null
  new_parent_id: string | null
  old_order: int
  new_order: int
  move_type: HIERARCHY_CHANGE | REORDER
    - HIERARCHY_CHANGE: parent_id가 다른 경우
    - REORDER: 동일 parent_id에서 order만 다른 경우
```

---

### 4-5. 엣지 케이스 구현

| 케이스 | 구현 방식 |
|--------|---------|
| node_id 없는 노드 | 위치(parent_id + order) 기반 순서 매칭으로 fallback; 매칭 불가 시 ADDED/DELETED |
| 중복 node_id 존재 | 경고 로그 기록 후 첫 번째 항목 사용, diff 결과에 `has_data_issue: true` 플래그 |
| 빈 tree_a | tree_b 전체 노드를 ADDED |
| 빈 tree_b | tree_a 전체 노드를 DELETED |
| 노드 수 초과 (>10,000) | DiffTooLargeError 예외, 비동기 처리로 전환 안내 |

---

### 4-6. 단위 테스트 시나리오

다음 시나리오를 반드시 테스트:

| 시나리오 | 기대 결과 |
|---------|----------|
| 변경 없음 | 모든 노드 UNCHANGED |
| 새 노드 1개 추가 | 해당 노드 ADDED |
| 노드 1개 삭제 | 해당 노드 DELETED |
| 노드 내용 수정 | 해당 노드 MODIFIED |
| 노드 순서 변경 (동일 부모) | 해당 노드 MOVED (REORDER) |
| 노드 부모 변경 | 해당 노드 MOVED (HIERARCHY_CHANGE) |
| 내용 + 위치 동시 변경 | MODIFIED with move_info |
| 다중 변경 혼합 | 각 노드별 정확한 분류 |
| 빈 tree_a → tree_b | 전체 ADDED |
| tree_a → 빈 tree_b | 전체 DELETED |
| 대량 노드 (1000+개) | 성능 기준 이내 처리 |

---

## 5. 산출물

1. NodeDiffer 서비스 구현
2. 내용 정규화 유틸리티 구현
3. 트리 diff 단위 테스트 (모든 시나리오 커버)
4. diff API 엔드포인트 연결 (`GET /versions/{v_id}/diff` 등)

---

## 6. 완료 기준

- NodeDiffer 서비스가 구현되어 있다
- 모든 변경 유형(ADDED/DELETED/MODIFIED/MOVED/UNCHANGED)이 올바르게 분류된다
- MOVE 감지가 계층 이동과 형제 재정렬을 구분한다
- 엣지 케이스(빈 트리, 중복 ID 등)가 안전하게 처리된다
- 단위 테스트가 작성되어 있고 모두 통과한다
- diff API 엔드포인트가 NodeDiffer와 연결되어 동작한다

---

## 7. Codex 작업 지침

- Task 9-2 설계를 정확히 따라 구현한다
- 내용 정규화 함수는 재사용 가능한 유틸리티로 분리한다
- 단위 테스트는 다양한 변경 조합을 커버하며, 경계값 케이스를 반드시 포함한다
- SQL 인젝션 등 보안 취약점이 없는지 확인한다 (노드 ID를 캐시 키에 사용할 때 포함)
- 노드 수가 많은 경우의 성능을 측정하고 10,000개 이상 시 예외 처리가 올바르게 동작하는지 확인한다
