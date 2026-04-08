# Task 9-6. 변경 요약 생성 구현

## 1. 작업 목적

DiffResult를 입력으로 받아 사람이 읽기 쉬운 변경 요약을 생성한다.

이 작업의 목표는 다음과 같다.

- 변경 규모를 정량화 (추가/삭제/수정 건수, 변경 텍스트 양)
- 자연어 요약 텍스트 생성 (예: "섹션 3개 수정, 섹션 1개 추가")
- 변경이 발생한 핵심 섹션 식별 (최상위 헤딩 기준)
- 워크플로 검토 화면과 버전 이력 타임라인에서 활용 가능한 경량 요약 제공

---

## 2. 작업 범위

### 포함 범위

- DiffSummaryGenerator 서비스 구현
- 변경 규모 계산 로직 구현
- 자연어 요약 텍스트 생성 로직 구현
- 핵심 변경 섹션 식별 로직 구현
- 단위 테스트 작성

### 제외 범위

- UI 렌더링 (Task 9-7에서 다룸)
- LLM 기반 요약 생성 (Phase 11 이후 고려)

---

## 3. 선행 조건

- Task 9-4 (트리 Diff 알고리즘 구현) 완료
- Task 9-5 (텍스트 인라인 Diff 구현) 완료 (텍스트 변경량 계산에 활용)

---

## 4. 주요 구현 대상

### 4-1. DiffSummaryGenerator 서비스 인터페이스

```
DiffSummaryGenerator
  .generate(node_diffs: NodeDiff[], nodes_a: Node[], nodes_b: Node[]) → DiffSummary
```

#### DiffSummary 구조

```json
{
  "total_added": 2,
  "total_deleted": 1,
  "total_modified": 3,
  "total_moved": 0,
  "total_unchanged": 12,
  "changed_characters": 450,
  "description": "섹션 2개 추가, 섹션 1개 삭제, 섹션 3개 수정됨",
  "changed_sections": [
    {
      "node_id": "node_s2",
      "title": "2. 정책 본문",
      "change_type": "MODIFIED",
      "sub_changes": 3
    },
    {
      "node_id": "node_new1",
      "title": "4. 신규 정책",
      "change_type": "ADDED",
      "sub_changes": 0
    }
  ]
}
```

---

### 4-2. 변경 규모 계산 로직

#### 노드 단위 카운트

```
total_added    = count(nodes where change_type == ADDED)
total_deleted  = count(nodes where change_type == DELETED)
total_modified = count(nodes where change_type == MODIFIED)
total_moved    = count(nodes where change_type == MOVED)
total_unchanged = count(nodes where change_type == UNCHANGED)
```

#### 텍스트 변경량 계산

MODIFIED/ADDED/DELETED 노드의 텍스트 변경량 합산:

```
changed_characters = Σ abs(len(after.content_text) - len(before.content_text))
                   + Σ len(content) for ADDED nodes
                   + Σ len(content) for DELETED nodes
```

인라인 diff가 있는 경우 실제 변경 문자수를 계산:
```
inline_changed = Σ len(token.text) for tokens where type in ('added', 'deleted')
```

---

### 4-3. 자연어 요약 텍스트 생성

변경 카운트를 기반으로 한국어 자연어 요약 생성:

#### 패턴 정의

| 조건 | 요약 텍스트 |
|------|-----------|
| 변경 없음 | "변경 사항 없음" |
| 추가만 | "섹션 {N}개 추가됨" |
| 삭제만 | "섹션 {N}개 삭제됨" |
| 수정만 | "섹션 {N}개 수정됨" |
| 혼합 | "섹션 {A}개 추가, {D}개 삭제, {M}개 수정됨" |
| 이동 포함 | 위 + ", {V}개 이동됨" |

#### 노드 유형별 명칭

노드의 `node_type`에 따라 더 구체적인 명칭 사용:
- section → "섹션"
- paragraph → "단락"
- list → "목록"
- table → "표"
- 기타 → "항목"

최상위 섹션 기준으로 요약하여 더 의미 있는 표현 제공:
```
"3. 보안 정책 섹션 수정, 4. 신규 접근 제어 정책 추가"
```

---

### 4-4. 핵심 변경 섹션 식별

변경이 발생한 최상위 섹션(루트 노드에 가까운 section 노드) 목록 생성:

#### 식별 알고리즘

1. ADDED/DELETED/MODIFIED/MOVED 노드 목록 추출
2. 각 변경 노드의 최상위 section 조상 노드 찾기 (재귀적 부모 탐색)
3. 중복 제거하여 변경된 최상위 섹션 목록 생성
4. 각 섹션의 `sub_changes` (해당 섹션 내 변경 노드 수) 카운트

#### 최대 표시 섹션 수

- `changed_sections` 목록: 최대 10개
- 초과 시 "외 N개 섹션 변경됨" 형태로 요약

---

### 4-5. 변경 심각도 분류 (선택)

관리자 및 검토자를 위한 변경 심각도 힌트:

| 심각도 | 기준 |
|--------|------|
| MAJOR | 전체 노드의 30% 이상 변경, 또는 최상위 섹션 절반 이상 변경 |
| MINOR | 전체 노드의 10~30% 변경 |
| TRIVIAL | 전체 노드의 10% 미만 변경 (오탈자 수준) |

이 필드는 `summary.severity`로 포함 (선택적).

---

### 4-6. 단위 테스트 시나리오

| 시나리오 | 기대 결과 |
|---------|----------|
| 변경 없음 | description: "변경 사항 없음", 모든 카운트 0 |
| 노드 1개 추가 | total_added: 1, description에 "1개 추가" 포함 |
| 노드 여러 개 혼합 변경 | 각 카운트 정확, description 올바름 |
| 깊이 중첩된 노드 변경 | 최상위 섹션이 changed_sections에 표시됨 |
| 전체 문서 교체 | MAJOR 심각도, 모든 기존 노드 DELETED, 신규 노드 ADDED |

---

## 5. 산출물

1. DiffSummaryGenerator 서비스 구현
2. 자연어 요약 텍스트 생성 로직 구현
3. 핵심 변경 섹션 식별 로직 구현
4. DiffSummary 스키마 구현
5. 단위 테스트

---

## 6. 완료 기준

- DiffSummaryGenerator가 구현되어 있다
- 변경 카운트가 정확하게 계산된다
- 자연어 요약 텍스트가 한국어로 생성된다
- changed_sections가 최상위 섹션 기준으로 정확히 식별된다
- 단위 테스트가 통과한다
- Task 9-3 API의 `summary` 필드가 이 서비스로 채워진다

---

## 7. Codex 작업 지침

- LLM 기반 요약 생성은 이 단계에서 구현하지 않는다 (Phase 11 이후 고려)
- 한국어 자연어 요약 패턴은 하드코딩 템플릿으로 구현한다 (단순하고 안정적)
- 요약에서 사용하는 노드 유형 명칭은 DocumentType 설정과 연계 가능한 구조로 설계한다 (Phase 12 대비)
- changed_sections 식별 시 재귀 부모 탐색의 무한 루프를 방지하는 깊이 제한을 반드시 구현한다
