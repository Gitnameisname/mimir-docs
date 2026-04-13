# Task 9-1. 변경 비교 시스템 아키텍처 설계

## 1. 작업 목적

Phase 9 전체 diff 시스템의 기술 방향과 구조를 확정한다.

이 작업의 목표는 다음과 같다.

- diff 대상 엔티티와 비교 단위를 명확히 정의
- DiffResult 핵심 데이터 모델 확정
- diff 결과 저장/캐싱 전략 결정
- Phase 5 워크플로, Phase 6 User UI와의 통합 지점 정의

---

## 2. 작업 범위

### 포함 범위

- diff 대상 엔티티 정의
- diff 알고리즘 선택 근거 문서화
- DiffResult 최상위 데이터 모델 정의
- diff 결과 저장/캐싱 전략 결정
- 각 Phase와의 통합 지점 정의
- 권한 검증 방식 결정

### 제외 범위

- 알고리즘 상세 설계 (Task 9-2에서 다룸)
- API 설계 상세 (Task 9-3에서 다룸)
- 구현 코드 작성

---

## 3. 선행 조건

- Phase 1: Document / Version / Node 트리 구조 이해
- Phase 4: 버전 저장 구조 이해
- Phase 5: 워크플로 상태 전이 이해
- Phase 6: User UI 구조 이해

---

## 4. 주요 설계 대상

### 4-1. diff 대상 엔티티 정의

Phase 9의 diff는 **두 Version의 Node 트리**를 비교 단위로 한다.

#### 비교 단위 계층

```
Document
  └── Version A  ←── diff ──►  Version B
        └── Node 트리                └── Node 트리
```

#### 비교 가능한 조합

| 비교 유형 | 설명 | API 접근 |
|----------|------|---------|
| 연속 버전 비교 | 버전 N과 버전 N-1 비교 | `GET /versions/{v}/diff` |
| 임의 버전 비교 | 두 버전을 명시적으로 선택 | `GET /versions/{v1}/diff/{v2}` |
| 최신 vs. 직전 Published | 검토 흐름에서 자동 선택 | 워크플로 컨텍스트로 처리 |

---

### 4-2. 알고리즘 선택 근거

#### 선택: 노드 ID 기반 매칭 + Myers 텍스트 diff

| 레이어 | 알고리즘 | 근거 |
|--------|---------|------|
| 트리 구조 diff | 노드 ID 기반 매칭 | Node에 고유 ID가 있으므로 LCS보다 단순하고 정확한 매칭 가능 |
| 텍스트 인라인 diff | Myers diff (diff-match-patch) | LCS 기반으로 최소 편집 거리를 계산, 한/영 모두 지원 |

#### 트리 diff 처리 순서

1. 두 트리의 모든 노드를 ID를 키로 하는 맵으로 변환
2. A에만 있는 ID → ADDED
3. B에만 있는 ID → DELETED
4. 양쪽에 있는 ID → 위치 비교 후 MOVED, 내용 비교 후 MODIFIED/UNCHANGED 분류

---

### 4-3. DiffResult 최상위 데이터 모델

```
DiffResult
  ├── document_id: string
  ├── version_a: VersionRef (id, number, created_at)
  ├── version_b: VersionRef (id, number, created_at)
  ├── summary: DiffSummary
  │     ├── total_added: int
  │     ├── total_deleted: int
  │     ├── total_modified: int
  │     ├── total_moved: int
  │     ├── total_unchanged: int
  │     └── description: string (자연어 요약)
  └── nodes: NodeDiff[]
        ├── node_id: string
        ├── change_type: ADDED|DELETED|MODIFIED|MOVED|UNCHANGED
        ├── node_before: NodeSnapshot | null
        ├── node_after: NodeSnapshot | null
        ├── inline_diff: InlineDiffToken[] | null  (MODIFIED 시)
        └── move_info: MoveInfo | null  (MOVED 시)
```

---

### 4-4. diff 결과 저장/캐싱 전략

#### 전략 선택: 온디맨드 계산 + 캐싱

**배경:** 버전은 한번 생성되면 변경되지 않으므로, 동일 버전 쌍의 diff 결과는 항상 동일하다.

| 옵션 | 장점 | 단점 |
|------|------|------|
| DB 사전 계산/저장 | 항상 빠른 응답 | 저장 공간 증가, 불필요한 사전 계산 |
| 온디맨드 + 캐싱 | 필요한 것만 계산, 구현 단순 | 첫 요청 지연 |
| 온디맨드만 | 구현 단순, 저장 불필요 | 반복 요청 시 매번 계산 비용 |

→ **권장: 온디맨드 계산 + 캐싱**

#### 캐시 키 설계

```
diff:{document_id}:{version_id_a}:{version_id_b}
```

- 버전 ID 순서를 정규화 (작은 ID가 앞으로) 하여 방향 무관 캐시 가능
- TTL: 무제한 (버전 불변성 활용) 또는 문서 삭제 시 캐시 무효화

---

### 4-5. Phase별 통합 지점

#### Phase 5 워크플로 통합

- **검토 화면**: 검토 요청 시 직전 버전 대비 diff 자동 제공
- **변경 사유**: 버전 생성 시 입력한 변경 사유를 diff 응답에 포함
- 통합 위치: 검토 요청 API 응답에 diff 요약 포함 또는 별도 API 호출

#### Phase 6 User UI 통합

- **문서 상세 화면**: 버전 히스토리 탭에서 버전 선택 후 diff 진입
- **버전 비교 페이지**: `/documents/{id}/versions/{v1}/compare/{v2}`
- **diff 뷰어 컴포넌트**: Task 9-7에서 구현

#### Phase 2 감사 로그 연계

- diff 응답에 버전 관련 감사 이벤트(변경 사유, 승인자) 링크 포함 가능
- diff 자체를 조회하는 행위도 감사 이벤트로 기록 여부 결정 (민감 문서의 경우)

---

### 4-6. 권한 검증 방식

- diff 요청 시 **version_a와 version_b 양쪽 모두**에 대한 접근 권한 확인
- 하나라도 접근 불가하면 diff 전체 거부 (403 반환)
- 권한 확인은 부모 Document 레벨의 ACL을 기준으로 함

---

## 5. 산출물

1. 변경 비교 시스템 아키텍처 문서
2. DiffResult 최상위 데이터 모델 정의서
3. 알고리즘 선택 근거 문서
4. diff 결과 캐싱 전략 문서
5. Phase별 통합 지점 정의서

---

## 6. 완료 기준

- diff 대상 엔티티와 비교 단위가 명확히 정의되어 있다
- DiffResult 최상위 데이터 모델이 확정되어 있다
- diff 결과 저장/캐싱 전략이 결정되어 있다
- Phase 5, 6, 2와의 통합 지점이 명확히 정의되어 있다
- 권한 검증 방식이 결정되어 있다
- 이후 Task(9-2~9-10)가 이 문서를 기준으로 진행 가능하다

---

## 7. Codex 작업 지침

- 실제 코드 구현은 하지 않는다
- 설계 문서 중심으로 작성한다
- 버전 불변성(immutability)을 캐싱 전략의 핵심 근거로 활용한다
- 권한 검증은 양쪽 버전 모두에 적용되어야 함을 설계 원칙으로 명시한다
- Phase 5 워크플로와의 통합이 이 시스템의 핵심 사용 사례임을 항상 고려한다
