# Phase 1 - Task 1-4. Version 구조 설계

## 1. 작업 목적

Task 1-1~1-3에서 확정한 요구사항, 엔티티 책임 정의, Document 구조 초안을 바탕으로,
**특정 시점의 문서 상태를 표현하는 Version 엔티티의 구조**를 상세 설계한다.

Version은 단순한 이력 번호가 아니라, 문서의 **재현 가능성(reproducibility)과 이력성(traceability)을 보장하는 핵심 엔티티**다.
이 문서는 Version이 무엇을 스냅샷으로 보존하고, 어떤 불변성과 참조 구조를 가져야 하는지를 확정한다.

---

## 2. Version의 역할 정의

### 2-1. Version이 대표하는 것

- Version은 **특정 시점에 해당 문서가 어떤 상태였는지**를 나타낸다.
- Version은 Document의 본체가 아니라 **Document에 종속된 시점 단위**다.
- Version은 해당 시점 문서 내용의 **완전한 재현 가능성**을 보장해야 한다.
- Version은 **Node 트리의 기준 단위**가 된다. 모든 Node는 특정 Version에 귀속된다.
- Version은 **변경 이력 관리의 기본 단위**다. 무엇이 언제 왜 바뀌었는지를 추적하는 단위다.

### 2-2. 책임 분리표

| 항목 | Version 책임 | 귀속 엔티티 |
|---|---|---|
| 특정 시점의 문서 제목 | O | Version (title_snapshot) |
| 특정 시점의 본문 구조 | O | Version → Node 트리 |
| 특정 시점의 메타데이터 | O | Version (metadata_snapshot) |
| 변경 설명 기록 | O | Version (change_summary) |
| 버전 생성자/생성일시 | O | Version |
| 문서의 현재 공식 버전 지정 | X | Document (current_version_id) |
| 문서 유형 분류 | X | Document (document_type) |
| 문서 전체 생명주기 상태 | X | Document (status) |
| 문서 본문 계층 구조 표현 | X (참조만) | Node 트리 |
| 본문 블록 내용 | X | Node |

---

## 3. Version 필드 후보 목록

| 필드명 | 의미 | 필수 여부 | MVP 포함 | 귀속 판단 | 비고 |
|---|---|---|---|---|---|
| `id` | Version 고유 식별자 | 필수 | Y | Version | UUID 권장 |
| `document_id` | 소속 Document 참조 | 필수 | Y | Version | 외래 참조 |
| `version_no` | 버전 순번 | 필수 | Y | Version | 정수 증가형 |
| `title_snapshot` | 해당 버전 시점의 문서 제목 복사본 | 필수 | Y | Version | 재현 가능성 보장 |
| `summary_snapshot` | 해당 버전 시점의 요약 복사본 | 선택 | Y | Version | 이력 재현 보조 |
| `metadata_snapshot` | 해당 버전 시점의 메타데이터 복사본 | 필수 | Y | Version | JSON 구조 |
| `root_node_id` | Node 트리의 루트 Node 참조 | 필수 | Y | Version | 본문 진입점 |
| `change_summary` | 이 버전에서 무엇이 바뀌었는지 설명 | 선택 | Y | Version | 사람이 작성 |
| `created_by` | 이 버전 생성자 | 필수 | Y | Version | 사용자 식별자 |
| `created_at` | 이 버전 생성 일시 | 필수 | Y | Version | 불변 |
| `base_version_id` | 이 버전이 기반한 이전 버전 참조 | 선택 | Y | Version | 이력 체인, nullable |
| `published_at` | 이 버전이 published 상태가 된 일시 | 선택 | Y | Version | NULL이면 미발행 |
| `is_published` | 이 버전이 발행된 버전인지 여부 | 선택 | Y | Version | published_at로 대체 가능 |
| `checksum` | 콘텐츠 무결성 검증용 해시 | 선택 | N | Version | 후속 Phase |
| `source_of_change` | 변경 출처 (수동 편집 / API / 마이그레이션 등) | 선택 | N | Version | 감사 기능 확장 시 |

---

## 4. 필드 분류 체계

### 4-1. 식별 필드

Version을 고유하게 식별하고 Document와의 관계를 나타내는 필드다.

| 필드 | 역할 |
|---|---|
| `id` | Version 자체의 전역 고유 식별자 |
| `document_id` | 이 Version이 속한 Document 참조 |
| `version_no` | Document 내에서 순서를 나타내는 증가 번호 |

> `version_no`는 Document 범위 내에서 단조 증가한다. 전체 시스템 고유 번호가 아니다.

### 4-2. 스냅샷 필드

해당 시점 문서 상태를 복원하기 위해 Version이 직접 보관하는 복사본 필드다.

| 필드 | 역할 |
|---|---|
| `title_snapshot` | 해당 버전 생성 시점의 문서 제목 |
| `summary_snapshot` | 해당 버전 생성 시점의 요약 |
| `metadata_snapshot` | 해당 버전 생성 시점의 메타데이터 전체 |

> 스냅샷 필드는 Document의 현재 값과 다를 수 있다. 과거 버전 조회 시 Document 값이 아닌 이 값을 기준으로 재현해야 한다.

### 4-3. 내용 참조 필드

Version이 보유하는 본문 콘텐츠 진입점 필드다.

| 필드 | 역할 |
|---|---|
| `root_node_id` | 이 Version의 Node 트리 최상위 루트 Node 참조 |

> 본문 내용 자체는 Node 트리가 담당한다. Version은 루트 Node만 참조한다.

### 4-4. 변경 이력 필드

변경의 맥락과 출처를 기록하는 필드다.

| 필드 | 역할 |
|---|---|
| `change_summary` | 이 버전에서 달라진 내용에 대한 사람이 작성한 설명 |
| `base_version_id` | 이 버전이 기반한 이전 버전. 이력 체인 구성에 사용 |
| `source_of_change` | 변경 출처 식별자 (수동/API/마이그레이션 등) |

### 4-5. 추적 필드

생성 주체와 시점을 기록하는 감사 필드다.

| 필드 | 역할 |
|---|---|
| `created_by` | 이 버전을 생성한 사용자 |
| `created_at` | 이 버전이 생성된 일시 (불변) |

### 4-6. 배포/상태 관련 필드

Version의 발행 여부를 나타내는 필드다.

| 필드 | 역할 |
|---|---|
| `published_at` | 이 버전이 공식 발행된 일시. NULL이면 미발행 상태 |

> MVP에서 Version 자체 상태는 최소화한다. `published_at`의 NULL 여부로 draft/published를 구분할 수 있다.

### 4-7. 무결성 보조 필드

콘텐츠 변조 여부를 검증하기 위한 필드다.

| 필드 | 역할 |
|---|---|
| `checksum` | Node 트리 전체 내용의 해시값. 무결성 검증에 사용 |

> MVP에서는 제외. 감사·보안 요구가 높아지는 시점에 도입한다.

---

## 5. 스냅샷 범위 정의

Version이 특정 시점을 완전히 재현하려면 어떤 정보를 직접 보관해야 하는지 판단한다.

| 정보 항목 | 스냅샷 보관 필요 | 판단 근거 |
|---|---|---|
| 제목 | **필요** | Document.title은 현재 값만 보관. 과거 버전의 제목을 재현하려면 Version이 직접 보관해야 함 |
| 요약 | **필요 (선택적)** | 제목과 같은 이유. 없어도 재현 가능하지만 있으면 이력 조회 UX 개선 |
| 문서 유형 | **참조로 충분** | DocumentType은 거의 변경되지 않음. 변경 이력이 필요해지는 시점에 스냅샷 추가 검토 |
| 메타데이터 | **필요** | 시행일·개정일·보고 기간 등 버전마다 달라지는 핵심 속성 포함. 반드시 스냅샷으로 보관 |
| Node 트리 구조 | **Node 트리 자체가 Version에 종속** | Node는 Version 소속. 별도 스냅샷 불필요. root_node_id 참조만으로 재현 가능 |
| 상태 | **불필요** | 상태는 Document 수준. 발행 일시(published_at)로 이력 대체 |
| 태그 | **metadata_snapshot에 포함** | 태그는 metadata 확장 구조 내 처리. 별도 스냅샷 불필요 |
| 첨부 참조 | **참조 ID 기록 (후속)** | MVP 외 범위. 첨부 엔티티 도입 시 Version-Attachment 연결 이력 추가 |
| 외부 링크 참조 | **Node 내 처리** | 본문 내 링크는 Node의 content에 포함. 별도 스냅샷 불필요 |

**핵심 원칙:**
> 과거 버전을 정확히 재현하는 데 필요한 정보는 Version이 직접 보관한다.
> Document의 현재 값은 언제든 바뀔 수 있으므로, 재현에 필요한 값은 반드시 스냅샷으로 고정한다.

---

## 6. Document와의 관계

### 6-1. Document 1 : N Version 관계의 의미

- 하나의 Document는 생명주기 동안 여러 Version을 가진다.
- Document는 `current_version_id`(공식 버전)와 `latest_version_id`(최신 버전)를 통해 Version을 참조한다.
- Document를 삭제(soft delete)하면 모든 하위 Version도 논리적으로 비활성화된다.

### 6-2. current_version과 latest_version의 분리

| 상황 | current_version_id | latest_version_id |
|---|---|---|
| 초기 draft 작성 중 | NULL (아직 없음) | draft version |
| 첫 발행 완료 | published version | published version |
| 발행 후 새 draft 편집 중 | 이전 published version | 새 draft version |
| 재발행 완료 | 새 published version | 새 published version |

> 두 값이 다를 때 = 현재 draft 편집이 진행 중인 상태.

### 6-3. Version에 is_current 플래그를 두어야 하는가?

**결정: MVP에서는 두지 않는다.**

- `is_current`를 Version에 두면 Document.current_version_id와 이중 관리가 발생한다.
- 정합성 유지 책임이 늘어나 버그 위험이 증가한다.
- Document.current_version_id 단일 출처(single source of truth) 원칙을 유지한다.

### 6-4. 제목 변경 시 관계

- Document.title 변경 → 새 Version 생성 여부를 결정해야 한다.
- **결정**: 제목 변경은 새 Version을 생성하는 것으로 처리한다. (메타 수정도 버전으로 관리)
- 단, 제목만 바꾸는 경미한 수정에도 version_no가 증가하는 것이 적절한지는 운영 정책에서 결정.
- 대안: 제목·요약 등 메타 수정은 Document-level 변경으로 처리하고 version_no는 본문 변경 시에만 증가하는 방식 → 이는 후속 Phase에서 결정.

---

## 7. Node와의 관계

### 7-1. 기본 원칙

- **Node는 특정 Version에 종속된다.** 특정 Version 없이 독립 존재하는 Node는 없다.
- **하나의 Node는 하나의 Version에만 귀속된다.** 버전 간 Node 공유는 하지 않는다.
- Version은 `root_node_id`를 통해 Node 트리의 진입점을 참조한다.
- 문서 본문 수정 = 새 Version 생성 + 새 Node 트리 구성.

```
Document
  └─ Version (v1)
       └─ Node (root)
            ├─ Node (section 1)
            │    └─ Node (paragraph 1-1)
            └─ Node (section 2)
                 └─ Node (paragraph 2-1)

  └─ Version (v2)  ← 본문 수정 시 새 Version + 새 Node 트리
       └─ Node (root)
            ├─ Node (section 1)  ← 새로 생성
            │    └─ Node (paragraph 1-1)
            └─ Node (section 2, 수정됨)
```

### 7-2. Node 공유 방식 검토 (대안)

| 방식 | 설명 | 장점 | 단점 |
|---|---|---|---|
| **Version별 독립 Node** (권장) | 각 Version이 자신만의 Node 트리를 가짐 | 완전한 재현 가능성, 단순한 구조 | 저장 공간 증가 |
| Node 공유 + diff 방식 | 변경된 Node만 새로 생성, 나머지는 공유 | 저장 효율 | 복잡한 참조 관리, 재현 로직 복잡화 |

> MVP에서는 **Version별 독립 Node 방식**을 채택한다. 저장 효율은 후속 Phase에서 최적화한다.

---

## 8. 버전 불변성 원칙

### 8-1. 원칙 선언

> **생성된 Version은 원칙적으로 불변(immutable)이다.**

- Version이 한 번 생성되면 그 내용(Node 트리, metadata_snapshot, title_snapshot 등)은 변경되지 않는다.
- 문서 수정은 기존 Version을 변경하는 것이 아니라 **새 Version을 생성하는 것**이다.
- 이 원칙이 재현 가능성과 이력 신뢰성의 기반이다.

### 8-2. 예외 처리 방침

| 상황 | 처리 방식 |
|---|---|
| 오탈자 등 경미한 수정 | 원칙적으로 새 Version 생성. 예외 없음 |
| 시스템 오류로 인한 보정 | 감사 로그(AuditEvent) 하에서만 허용. MVP 이후 도입 |
| metadata만 변경 | 새 Version 생성 또는 Document-level 변경으로 처리 (운영 정책 결정 필요) |
| Version 삭제 | 허용하지 않는 것을 원칙으로 함. 비활성화(soft delete) 방식 검토 |

### 8-3. 불변성이 필요한 이유

- **감사/법적 요건**: 특히 규정 문서(policy)는 특정 시점의 내용이 법적 증거가 될 수 있다.
- **재현 가능성**: "당시 문서는 이랬다"를 신뢰할 수 있어야 한다.
- **승인 이력**: 특정 버전이 승인된 사실은 해당 버전의 불변성이 보장되어야 유효하다.

---

## 9. 버전 관리 정책 초안

### 9-1. 버전 번호 부여 방식

**결정: MVP에서는 Document 범위 내 단조 증가 정수를 사용한다.**

- `version_no`는 Document별로 1부터 시작해 생성될 때마다 1씩 증가한다.
- major/minor 방식(1.0, 1.1, 2.0 등)은 운영 정책 도입 시 추가 검토한다.
- 버전 번호는 생성 순서를 나타내며, 논리적 중요도를 나타내지 않는다.

### 9-2. Draft 버전을 Version 엔티티로 볼 것인가?

**결정: Draft도 Version 엔티티로 생성한다.**

- Draft 상태의 편집 내용도 Version으로 저장한다.
- `published_at`이 NULL인 Version = draft (미발행) 상태.
- 이 방식으로 draft → published 전환이 Version 내 상태 변화로 처리된다.

### 9-3. Published/Working(Draft) Version 분리

| 개념 | 의미 | 식별 방법 |
|---|---|---|
| Published Version | 공식 발행된 버전. 독자가 보는 버전 | `published_at` IS NOT NULL |
| Draft Version | 편집 중인 미발행 버전 | `published_at` IS NULL |
| Current Version | Document의 현재 공식 버전 | Document.current_version_id 참조 |
| Latest Version | 가장 최근 생성된 버전 | Document.latest_version_id 참조 |

### 9-4. 복원 시 새 Version 생성 여부

**결정: 복원도 새 Version 생성으로 처리한다.**

- 과거 버전을 복원하면 해당 버전의 내용을 복사한 새 Version을 생성한다.
- 복원된 Version은 `base_version_id`에 복원 기준이 된 과거 Version을 기록한다.
- 원본 과거 Version은 변경하지 않는다 (불변성 원칙).

### 9-5. diff/비교 기능 (범위 외 메모)

- Phase 1 범위 밖이나, Version이 독립적인 스냅샷 구조를 가지므로 diff 비교가 구조적으로 가능하다.
- diff는 Node 트리 비교 알고리즘으로 구현한다 (후속 Phase).
- `base_version_id`를 통해 이력 체인을 따라 변경 흐름을 추적할 수 있다.

---

## 10. 최소 권장 Version 구조안

### 10-1. 필수 필드 (MVP 코어)

```
Version
  - id                  : Version 고유 식별자
  - document_id         : 소속 Document 참조
  - version_no          : Document 내 순번 (1부터 증가)
  - title_snapshot      : 이 버전 시점의 문서 제목 복사본
  - metadata_snapshot   : 이 버전 시점의 메타데이터 복사본 (JSON)
  - root_node_id        : Node 트리 루트 참조
  - created_by          : 생성자
  - created_at          : 생성 일시 (불변)
```

### 10-2. 권장 필드 (MVP 포함 권장)

```
  - summary_snapshot    : 이 버전 시점의 요약 복사본
  - change_summary      : 변경 설명 (사람이 작성)
  - base_version_id     : 기반한 이전 버전 참조 (nullable)
  - published_at        : 발행 일시 (nullable, NULL이면 draft)
```

### 10-3. 선택 확장 필드 (후속 Phase)

```
  - source_of_change    : 변경 출처 식별 (수동/API/마이그레이션)
  - checksum            : 콘텐츠 무결성 해시
```

---

## 11. 설계 쟁점과 판단 결과

| 쟁점 | 선택한 방향 | 이유 |
|---|---|---|
| `version_no`를 단순 정수로 시작할지 | **단순 정수 증가형** | MVP 단순화. major/minor는 운영 정책 도입 시 확장 |
| `title_snapshot`을 둘지 | **보유** | Document.title은 현재값만 보관. 과거 재현을 위해 Version이 직접 보관 필요 |
| `metadata_snapshot`을 어디까지 저장할지 | **Version 생성 시점 metadata 전체 저장** | 유형별 핵심 속성(시행일 등)이 포함됨. 부분 저장은 재현 불완전 |
| `is_current`/`is_published`를 Version에 둘지 | **두지 않음** | Document.current_version_id 단일 출처 원칙. 이중 관리 방지 |
| `base_version_id`를 코어에 포함할지 | **MVP에 포함 (선택 필드)** | 복원 이력 추적에 필수. nullable이므로 부담 없음 |
| draft를 Version 엔티티로 즉시 포함할지 | **포함** | draft도 Version으로 관리. `published_at` NULL로 구분 |

---

## 12. 다음 Task 입력값

### Task 1-5 (Node 트리 구조 설계)로 전달할 항목

- Node는 반드시 특정 Version에 종속되어야 한다.
- Version은 `root_node_id`를 통해 Node 트리의 진입점을 참조한다.
- MVP에서는 Version별 독립 Node 방식을 채택한다 (Node 공유 없음).
- Node 수정은 새 Version 생성을 전제로 설계해야 한다.
- 루트 Node의 정의와 역할을 Task 1-5에서 명확히 해야 한다.

### Task 1-8 (상태 모델 기초 설계)와의 연결 포인트

- MVP 기준 Version 자체 상태는 `published_at` NULL 여부로 draft/published를 구분한다.
- Document.status와 Version의 발행 상태 간 관계 규칙이 필요하다.
  - 예: Document가 published가 되려면 최소 하나의 published Version이 있어야 한다.
- 향후 승인 워크플로 도입 시 Version 수준 상태(review_status 등) 확장 방식을 Task 1-8에서 결정한다.
- `archived_at`, `deleted_at`을 Version 수준에도 둘지 여부를 Task 1-8에서 결정한다.

### 저장 구조 설계(Phase 2) 시 확인해야 할 Version 관련 쟁점

- Version별 독립 Node 방식은 데이터 증가량이 크다. 저장 최적화 전략 필요.
- `metadata_snapshot`을 별도 테이블로 분리할지 JSON 컬럼으로 인라인할지 결정 필요.
- `root_node_id` 참조와 Node 트리의 cascade delete/archive 정책 결정 필요.
- 버전 수가 많아질 때 성능(특히 current_version 조회)을 위한 인덱스 전략 필요.
