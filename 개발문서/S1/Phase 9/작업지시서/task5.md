# Task 9-5. 텍스트 인라인 Diff 구현

## 1. 작업 목적

MODIFIED 노드 내에서 텍스트 수준의 세밀한 변경을 추적하여 인라인 diff를 생성한다.

이 작업의 목표는 다음과 같다.

- Myers diff 또는 LCS 기반 알고리즘으로 단어/문자 단위 텍스트 diff 계산
- diff 결과를 `{type, text}` 토큰 배열로 표현하여 UI 렌더링에 활용
- 마크다운 구조를 보존하면서 의미 있는 diff 단위로 처리
- 노드 콘텐츠 타입(텍스트, 리스트, 테이블)별 diff 처리 방식 정의

---

## 2. 작업 범위

### 포함 범위

- TextDiffer 서비스 구현
- 단어 단위 diff 처리 구현
- diff 토큰 배열 생성 구현
- 노드 콘텐츠 타입별 diff 처리 방식 구현
- 단위 테스트 작성

### 제외 범위

- 트리 구조 diff (Task 9-4에서 완료)
- UI 렌더링 (Task 9-7에서 다룸)
- 성능 최적화 (Task 9-10에서 다룸)

---

## 3. 선행 조건

- Task 9-4 (트리 Diff 알고리즘 구현) 완료
- Phase 1: Node 콘텐츠 구조(텍스트, 마크다운 등) 이해

---

## 4. 주요 구현 대상

### 4-1. TextDiffer 서비스 인터페이스

```
TextDiffer
  .diff(text_a: str, text_b: str, granularity: 'word'|'char') → InlineDiffToken[]
  .diff_content(content_a: NodeContent, content_b: NodeContent) → InlineDiffToken[]
```

#### InlineDiffToken 스키마

```json
[
  { "type": "unchanged", "text": "본 정책은 " },
  { "type": "deleted",   "text": "기존" },
  { "type": "added",     "text": "변경된" },
  { "type": "unchanged", "text": " 내용을 정의한다." }
]
```

| 필드 | 값 | 설명 |
|------|-----|------|
| `type` | `added` | 새 버전에서 추가된 텍스트 |
| `type` | `deleted` | 이전 버전에서 삭제된 텍스트 |
| `type` | `unchanged` | 변경 없는 텍스트 |

---

### 4-2. 사용 라이브러리 선택

#### diff-match-patch (권장)

- Google이 개발한 LCS 기반 diff 라이브러리
- Python (`diff-match-patch`), JavaScript (`diff-match-patch`) 모두 지원
- 문자 단위 또는 단어 단위 diff 가능
- 성숙한 라이브러리, 한국어 텍스트에서도 안정적으로 동작

#### 단어 단위 diff 전처리 방식

diff-match-patch의 기본 단위는 문자이므로, 단어 단위 diff를 위한 전처리:

1. 텍스트를 단어로 분리하여 각 단어를 단일 Unicode 문자로 인코딩
2. 인코딩된 문자열로 diff 계산
3. 결과를 원래 단어로 디코딩

→ diff-match-patch의 `diff_wordsToChars` 방식 활용

---

### 4-3. 단어 단위 diff가 권장되는 이유

한국어 텍스트에서 문자 단위 diff의 문제:

```
기존: "정보 보안 정책"
변경: "정보보안 정책"

문자 단위 diff:
  unchanged: "정보"
  deleted: " "       ← 공백 하나 변경이 노이즈
  unchanged: "보안 정책"

단어 단위 diff:
  deleted: "정보 보안"
  added: "정보보안"
  unchanged: " 정책"
```

단어 단위가 훨씬 가독성 높은 diff 결과 제공.

---

### 4-4. 노드 콘텐츠 타입별 처리

노드의 콘텐츠 타입에 따라 diff 처리 방식을 구분:

#### 일반 텍스트 / 단락

- 단어 단위 diff 직접 적용

#### 마크다운 텍스트

- 마크다운 렌더링 전 원문 텍스트로 diff 계산
- 마크다운 기호(**, *, #, - 등)도 단어로 취급
- 마크다운 구조 자체가 변경된 경우(# 추가/제거)도 토큰으로 표현

#### 리스트 항목

- 리스트를 항목 단위로 분리
- 항목별로 추가/삭제/수정 분류
- 수정된 항목 내에서 단어 단위 diff 적용

```
before: ["항목 A", "항목 B", "항목 C"]
after:  ["항목 A", "항목 B 수정됨", "항목 D"]

결과:
  unchanged: "항목 A"
  modified:  "항목 B" → "항목 B 수정됨"
  deleted:   "항목 C"
  added:     "항목 D"
```

#### 테이블

- 행(row) 단위로 추가/삭제/수정 분류
- 수정된 행의 각 셀에서 단어 단위 diff 적용
- 컬럼 추가/삭제는 구조 변경으로 처리

#### 구조화 JSON (기타 콘텐츠)

- JSON 평문 추출 후 단어 단위 diff 적용
- 또는 JSON diff 전용 처리 (Phase 이후 확장 포인트)

---

### 4-5. 텍스트 길이 제한 처리

`max_inline_length` 초과 시:

- 인라인 diff 생성 건너뜀
- 응답에 `inline_diff: null` 및 `inline_diff_skipped: true` 포함
- 이유: 매우 긴 텍스트의 diff는 계산 비용이 크고 UI 렌더링도 느려짐

---

### 4-6. 단위 테스트 시나리오

| 시나리오 | 기대 결과 |
|---------|----------|
| 동일 텍스트 | 전체 unchanged 토큰 |
| 단어 하나 수정 | deleted + added 토큰 |
| 문장 추가 | added 토큰 |
| 문장 삭제 | deleted 토큰 |
| 한국어 단어 수정 | 올바른 단어 경계로 분리 |
| 마크다운 헤딩 변경 | `#` 기호 포함하여 올바른 diff |
| 빈 문자열 → 텍스트 | 전체 added |
| 텍스트 → 빈 문자열 | 전체 deleted |
| 매우 긴 텍스트 | max_inline_length 초과 시 skip 처리 |

---

## 5. 산출물

1. TextDiffer 서비스 구현
2. 노드 콘텐츠 타입별 diff 처리 구현
3. InlineDiffToken 스키마 구현
4. 텍스트 인라인 diff 단위 테스트

---

## 6. 완료 기준

- TextDiffer 서비스가 구현되어 있다
- 단어 단위 diff가 한국어 텍스트에서 올바르게 동작한다
- 리스트 항목 단위 diff가 동작한다
- max_inline_length 초과 시 안전하게 스킵 처리된다
- 단위 테스트가 작성되어 있고 모두 통과한다
- Task 9-4의 NodeDiffer와 연결되어 MODIFIED 노드에 inline_diff가 채워진다

---

## 7. Codex 작업 지침

- diff-match-patch 라이브러리를 활용하며, 단어 단위 diff 전처리를 구현한다
- 한국어 텍스트에서 단어 경계 처리가 올바르게 동작하는지 테스트를 통해 반드시 검증한다
- 노드 콘텐츠 타입별 처리 로직은 플러그인/전략 패턴으로 구조화하여 새 콘텐츠 타입 추가가 쉽도록 한다 (Phase 12 확장 대비)
- 매우 긴 텍스트 처리 시 메모리 사용량을 고려한다
