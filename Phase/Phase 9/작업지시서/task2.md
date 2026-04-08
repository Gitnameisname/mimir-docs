# Task 9-2. 트리 Diff 알고리즘 설계

## 1. 작업 목적

Node 트리 구조의 변경을 정확하게 감지하는 알고리즘을 설계한다.

이 작업의 목표는 다음과 같다.

- 노드 ID 기반 매칭 알고리즘으로 구조적 변경 정확하게 분류
- 변경 유형(ADDED/DELETED/MODIFIED/MOVED/UNCHANGED) 분류 기준 명확화
- MOVE 감지 방식 정의 (동일 ID 노드의 위치 변경)
- 엣지 케이스 처리 방침 결정

---

## 2. 작업 범위

### 포함 범위

- 노드 ID 기반 매칭 알고리즘 설계
- 변경 유형 분류 기준 정의
- 트리 재귀 탐색 방식 설계
- MOVE 감지 알고리즘 설계
- 엣지 케이스 처리 기준

### 제외 범위

- 텍스트 인라인 diff 알고리즘 (Task 9-5에서 다룸)
- 구현 코드 작성

---

## 3. 선행 조건

- Task 9-1 (변경 비교 시스템 아키텍처 설계) 완료
- Phase 1: Node 트리 구조 상세 이해 (node_id, parent_id, order, content 필드)

---

## 4. 주요 설계 대상

### 4-1. 알고리즘 입력/출력 정의

#### 입력

```
TreeDiff(tree_a: Node[], tree_b: Node[]) → NodeDiff[]
```

- `tree_a`: Version A의 전체 Node 목록 (flat array, 트리 관계는 parent_id로)
- `tree_b`: Version B의 전체 Node 목록

#### 출력

```
NodeDiff[]
  각 항목:
    node_id: string
    change_type: ADDED | DELETED | MODIFIED | MOVED | UNCHANGED
    before: NodeSnapshot | null
    after: NodeSnapshot | null
    move_info: { old_parent_id, new_parent_id, old_order, new_order } | null
```

---

### 4-2. 노드 ID 기반 매칭 방식

#### 기본 원칙

- 동일한 `node_id`를 가진 노드는 "같은 노드"로 취급
- A에만 있는 ID → **DELETED**
- B에만 있는 ID → **ADDED**
- 양쪽에 있는 ID → 내용 및 위치 비교 후 MODIFIED / MOVED / UNCHANGED 분류

#### 매칭 맵 생성

```
map_a = { node_id → Node } from tree_a
map_b = { node_id → Node } from tree_b

all_ids = union(keys(map_a), keys(map_b))
```

---

### 4-3. 변경 유형 분류 기준

동일 ID를 가진 노드 쌍에 대해 다음 순서로 분류:

#### UNCHANGED 조건

- 내용(content) 동일
- 부모 ID 동일
- 순서(order) 동일

→ 세 조건 모두 만족 시 UNCHANGED

#### MODIFIED 조건

- 내용(content)이 다름
- 위치(부모 ID, 순서)는 동일 또는 다를 수 있음

#### MOVED 조건

- 내용(content)은 동일
- 위치(부모 ID 또는 순서)가 다름

#### MODIFIED + MOVED (복합)

- 내용도 다르고 위치도 다른 경우
- 변경 유형: **MODIFIED** (내용 변경이 우선)
- move_info에 위치 변경 정보 추가 포함

---

### 4-4. 내용 비교 방식

노드 내용 동일 여부를 판단하는 방식:

| 비교 대상 | 방식 |
|----------|------|
| content (텍스트/JSON) | 정규화된 JSON 직렬화 후 문자열 비교 |
| node_type | 문자열 비교 |
| title / heading | 문자열 비교 |
| metadata | JSON 직렬화 후 비교 |

#### 정규화 원칙

- 키 순서 정렬된 JSON 직렬화 사용 (키 순서 차이로 인한 오탐 방지)
- 공백/줄바꿈 정규화 포함 여부 설정 가능 (기본: 정규화 없음)

---

### 4-5. 위치(MOVE) 비교 방식

동일 ID 노드의 위치 변경 감지:

#### 위치 정의

```
position = (parent_id, order)
  - parent_id: 부모 노드 ID (최상위 노드는 null)
  - order: 형제 노드 중 순서 (0부터 시작)
```

#### MOVE 판단 기준

- `parent_id` 변경 → 부모가 바뀐 이동 (계층 이동)
- `order` 변경 → 동일 부모 내 순서 변경 (형제 재정렬)

#### 형제 재정렬 처리

동일 부모 내에서 여러 노드의 순서가 바뀐 경우:
- 각 노드를 개별적으로 MOVED로 표시
- 이동 전/후 순서 정보 포함

---

### 4-6. 트리 재귀 탐색 방식

DiffResult에서 노드 변경 목록을 어떤 순서로 제공할지 결정:

#### 탐색 순서 옵션

| 옵션 | 설명 |
|------|------|
| 변경된 노드만 (기본) | ADDED/DELETED/MODIFIED/MOVED 노드만 포함 |
| 전체 노드 | UNCHANGED 포함 |
| 최상위 섹션 기준 그룹화 | 변경이 있는 최상위 섹션별로 그룹화 |

기본 응답은 "변경된 노드만" 반환하고, `include_unchanged=true` 파라미터로 전체 포함 가능.

---

### 4-7. 엣지 케이스 처리

| 케이스 | 처리 방침 |
|--------|---------|
| node_id가 없는 노드 (레거시 데이터) | 위치(parent+order) 기반 순서 매칭으로 fallback |
| 동일 ID가 두 개 이상 존재 | 오류 처리 (데이터 무결성 문제로 diff 불가) |
| 빈 트리 (신규 버전이 첫 버전) | 모든 노드를 ADDED로 처리 |
| 빈 트리 (문서 내용 전체 삭제) | 모든 노드를 DELETED로 처리 |
| 트리 깊이 제한 초과 | 최대 깊이(기본 20) 이상 노드는 경고 후 처리 |
| 노드 수 매우 많은 경우 | 최대 노드 수(기본 10,000) 초과 시 비동기 처리로 전환 |

---

### 4-8. 알고리즘 복잡도 분석

| 단계 | 시간 복잡도 | 설명 |
|------|-----------|------|
| 맵 생성 | O(n + m) | n: tree_a 노드 수, m: tree_b 노드 수 |
| ID 매칭 | O(n + m) | 해시맵 기반 O(1) 조회 |
| 내용 비교 | O(k) | k: 수정된 노드의 내용 평균 길이 |
| 전체 | O(n + m + Σk) | 실질적으로 선형에 근접 |

Myers diff 트리 알고리즘(O(n²))과 비교하여 노드 ID 기반 방식이 훨씬 효율적.

---

## 5. 산출물

1. 트리 Diff 알고리즘 설계서 (매칭 방식, 분류 기준, 탐색 방식 포함)
2. 변경 유형 분류 기준 결정 트리 (flowchart)
3. 엣지 케이스 처리 기준 문서
4. 알고리즘 복잡도 분석서

---

## 6. 완료 기준

- 노드 ID 기반 매칭 알고리즘이 명확히 설계되어 있다
- 변경 유형(ADDED/DELETED/MODIFIED/MOVED/UNCHANGED) 분류 기준이 구체적으로 정의되어 있다
- MOVE 감지 방식(계층 이동 vs. 형제 재정렬)이 구분되어 있다
- 모든 엣지 케이스에 대한 처리 방침이 결정되어 있다
- 알고리즘 복잡도가 분석되어 있다

---

## 7. Codex 작업 지침

- 코드 작성 금지 (설계 중심, 의사코드 수준은 구조 설명 목적으로 허용)
- MODIFIED와 MOVED의 복합 케이스 처리 방침을 명확히 결정한다
- 레거시 데이터(node_id 없는 경우) 처리 방침을 반드시 포함한다
- 노드 수가 많은 대용량 문서의 처리 방식도 설계에 포함한다
- 이 알고리즘은 Task 9-4에서 구현될 것이므로 구현 가능한 수준으로 구체적으로 설계한다
