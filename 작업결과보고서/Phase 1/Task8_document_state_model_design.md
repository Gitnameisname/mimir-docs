# Phase 1 - Task 1-8. 문서 상태 모델 기초 설계

## 1. 작업 목적

Task 1-1~1-7에서 확정한 요구사항, 엔티티 책임, Document/Version/Node/DocumentType/metadata 구조를 바탕으로,
**범용 문서 플랫폼에서 문서의 생명주기를 표현하는 최소 상태 모델**을 설계한다.

상태 이름을 나열하는 것이 아니라,
상태의 의미, 적용 범위, 전이 규칙의 기초를 확정하여
문서의 가시성·수정 가능성·공식성·보관 여부를 플랫폼 차원에서 일관되게 다룰 수 있도록 한다.

---

## 2. 상태 모델의 역할 정의

### 2-1. 상태 모델이 하는 것

- 상태 모델은 **문서의 생명주기 단계를 표현**한다.
- 상태는 단순 표시값이 아니라 **운영 규칙의 기준**이 될 수 있다. (수정 가능 여부, 노출 여부 등)
- 상태는 **문서의 가시성, 수정 가능성, 공식성, 보관 여부**와 연결된다.
- 상태 모델은 **Document 중심으로 시작**하되, Version과의 관계도 고려한다.
- 상태 모델은 **초기에는 단순하게 유지하고 후속 확장 가능**해야 한다.

### 2-2. 책임 분리표

| 항목 | 상태 모델 책임 | 귀속 위치 |
|---|---|---|
| 문서 생명주기 단계 표현 | O | Document.status |
| 문서 가시성 판단 기준 | O (일부) | Document.status |
| 수정 가능 여부 기준 | O (일부) | Document.status + 서비스 레이어 |
| 공식 배포 여부 표현 | O | Document.status (published) |
| 보관 표현 | O | Document.status (archived) |
| 논리 삭제 표현 | O (별도 필드) | Document.deleted_at |
| 누가 수정할 수 있는지 | X | 권한 모델 (후속 Phase) |
| 승인 절차의 흐름 | X | 승인 워크플로 (Phase 5 이후) |
| 버전 번호 관리 | X | Version 코어 |
| 특정 버전의 발행 일시 | X | Version.published_at |

### 2-3. 상태 모델 vs 권한 모델 vs 승인 프로세스

| 개념 | 역할 | 비고 |
|---|---|---|
| **상태 모델** | 문서가 현재 어떤 단계에 있는지 | "이 문서는 published 상태다" |
| **권한 모델** | 누가 이 문서를 보거나 수정할 수 있는지 | "이 부서만 수정 가능하다" |
| **승인 프로세스** | 어떤 절차를 거쳐 상태가 전이되는지 | "팀장 승인 후 published 전환" |

> 상태 모델은 결과(현재 단계)를 표현하고, 승인 프로세스는 전이의 절차를 정의한다.
> 권한은 누가 행위할 수 있는지를 결정한다. 세 개념은 독립적이다.

---

## 3. 상태 적용 단위 검토

### 3-1. 세 가지 관점 비교

| 관점 | 설명 | 장점 | 한계 |
|---|---|---|---|
| **A. Document 상태만** | Document 하나에 단일 status 보유 | 단순, MVP에 적합, 이해 쉬움 | 특정 버전 단위 검토/배포 표현이 약함 |
| **B. Version 상태만** | 각 Version이 상태를 보유 | 배포본/검토본 구분에 유리 | 문서 전체의 현재 운영 상태 표현이 어려움 |
| **C. Document + Version 혼합** | Document 전체 상태 + Version 단위 상태 | 확장성 높음, 정밀 표현 가능 | 초기 복잡도 증가, 두 상태의 정합성 관리 필요 |

### 3-2. 권장 방향

**결정: MVP에서는 Document 중심 상태 모델(관점 A)을 채택한다. Version에는 후속 확장을 위한 최소 marker만 열어둔다.**

근거:
- 대부분의 사용 시나리오에서 "이 문서가 현재 어떤 상태인가"는 Document 수준 질문이다.
- Version 단위 상태는 승인 워크플로 도입 시 자연스럽게 확장할 수 있다.
- Task 1-4에서 결정한 `Version.published_at`이 Version 단위 발행 여부를 이미 표현하고 있다.

**Version에 열어두는 최소 포인트**:
- `Version.published_at`: 이 Version이 발행된 일시 (이미 설계에 포함)
- 향후 `Version.review_status` 같은 확장 필드 추가 가능성 확보

---

## 4. 최소 상태 집합 후보

### 4-1. 후보 검토표

| 상태 | 의미 | MVP 포함 | 다른 상태와 중복 여부 | 판단 |
|---|---|---|---|---|
| `draft` | 작성 중, 미확정 | O | - | 필수. 모든 문서의 시작 상태 |
| `review` | 검토/승인 대기 | O | - | 포함. 검토 흐름의 최소 표현 |
| `published` | 공식 배포/유효 | O | - | 필수. 운영의 핵심 상태 |
| `archived` | 비활성 보관 | O | deleted와 구분 필요 | 포함. 보관 정책 표현 필수 |
| `deprecated` | 구식/대체됨 | N | archived와 유사 | 후속 확장. 초기엔 archived로 흡수 |
| `superseded` | 새 버전으로 대체됨 | N | deprecated와 유사 | 후속 확장. metadata로도 표현 가능 |
| `rejected` | 검토 반려 | N | draft로 복귀로 처리 | 승인 워크플로와 연동. 후속 Phase |
| `scheduled` | 특정 시점 배포 예정 | N | - | 배포 자동화 단계에서 도입 |
| `inactive` | 비활성 (이유 불명확) | N | archived와 중복 가능 | 불필요. archived로 통합 |
| `deleted` | 삭제됨 | N (별도 정책) | archived와 다름 | 상태 아님. `deleted_at` 필드로 처리 |

### 4-2. MVP 상태 집합 확정

**`draft` → `review` → `published` → `archived`**

---

## 5. 상태별 의미 정의

### 5-1. draft

> **아직 확정되지 않은 작업 중 문서 상태다.**

- 문서가 생성되면 기본적으로 draft 상태로 시작한다.
- 내용 편집이 자유롭게 허용된다.
- 외부 사용자(독자)에게 노출되지 않는 것이 기본 정책이다.
- 하나의 Document에 동시에 여러 draft 작업이 존재할 수 있다. (각각 Version으로 관리)

### 5-2. review

> **검토 또는 승인 대기 중인 상태다.**

- draft 완료 후 내부 검토·승인 요청 시 전이한다.
- 수정 가능 여부는 운영 정책에 따라 결정한다. (검토 중 수정 금지 또는 수정 후 재요청 허용)
- 검토 결과에 따라 published로 전진하거나 draft로 복귀한다.
- MVP에서는 review 상태의 세부 승인 절차는 다루지 않는다.

### 5-3. published

> **공식적으로 유효하거나 외부에 공개 가능한 상태다.**

- 독자·사용자가 접근하는 공식 대표 상태다.
- `Document.current_version_id`가 가리키는 Version이 published 상태의 기준이 된다.
- published 상태 문서를 직접 수정하지 않는다. 새 draft → review → published 흐름으로 개정한다.
- 동시에 한 Document는 하나의 published 버전만 가진다.

### 5-4. archived

> **더 이상 활성 운영 대상은 아니지만, 이력 보존을 위해 시스템에 남아 있는 상태다.**

- published 이후 더 이상 유효하지 않게 된 문서에 적용한다.
- 기본적으로 일반 검색 결과에서 제외되나, 명시적 조회 시 열람 가능하다.
- archived 문서를 직접 수정하지 않는다. 필요 시 복원(새 draft 생성) 후 진행한다.
- 법적 보존 요건이 있는 문서(규정집 등)는 archived 상태로 장기 보관한다.

### 5-5. 개념 구분

| 비교 쌍 | 구분 기준 |
|---|---|
| **archived vs deleted** | archived = 보존 목적 비활성화. deleted = 논리/물리 삭제 처리. archived는 열람 가능, deleted는 사실상 접근 불가 |
| **published vs effective** | published = 플랫폼 운영 상태. effective = 법적·조직적 효력 발생일(policy 유형의 metadata). 두 개념은 독립적 |
| **draft vs working copy** | draft = Document의 공식 상태. working copy = 특정 Version의 편집 상태. MVP에서는 draft로 통합 표현 |
| **archived vs deprecated/superseded** | archived = 전체 문서 비활성화. deprecated/superseded = 특정 문서가 다른 문서로 대체됨. 후자는 문서 간 관계 표현이므로 후속 확장 |

---

## 6. 상태 전이 규칙 초안

### 6-1. 허용 전이 규칙

| 전이 | 허용 여부 | 사용 시나리오 | 주의사항 |
|---|---|---|---|
| `draft → review` | O | 작성 완료 후 검토 요청 | 최소 하나의 Version이 있어야 함 |
| `review → draft` | O | 검토 반려, 수정 후 재요청 | 반려 사유 기록 권장 |
| `review → published` | O | 검토 통과, 공식 배포 | `current_version_id` 갱신 필요 |
| `published → archived` | O | 문서 폐기, 신규 버전으로 대체 | `archived_at` 기록 |
| `draft → archived` | O (예외) | 작성 취소, 폐기 결정 | 드문 케이스. 명시적 이유 기록 권장 |
| `archived → draft` | O (복원) | 보관 문서 재활성화 | 새 Version 생성으로 처리. 원본 보존 |
| `published → draft` | 제한적 허용 | 긴급 수정 개시 | 직접 draft로 전환보다 새 draft Version 생성 권장. Document 상태는 published 유지 가능 |
| `archived → published` | X | - | 보관 문서 직접 재발행 불허. draft → review → published 경유 |
| `review → archived` | X | - | 검토 중 폐기 시 review → draft → archived 경유 |

### 6-2. 전이 원칙

- **제어된 전이만 허용한다.** 임의 상태 간 자유 전이는 허용하지 않는다.
- **상태 전이는 서비스 레이어에서 검증한다.** DB 레벨 강제가 아닌 로직 레이어 검증.
- **복원 시나리오는 archived → draft 경로를 사용한다.** 원본 Version은 보존된다.
- **상태 전이 이력 기록은 후속 Phase(AuditEvent 도입 시)에서 완성한다.**

### 6-3. 상태 전이 다이어그램 (텍스트 표현)

```
              ┌──────────────────┐
              ▼                  │
[draft] ──► [review] ──► [published] ──► [archived]
  ▲            │                             │
  └────────────┘                             │
  (반려)                               (복원: 새 draft)
              ▲──────────────────────────────┘
```

---

## 7. 수정 가능성과의 관계

### 7-1. 상태별 수정 정책

| 상태 | 직접 수정 | 새 Version 생성 | 비고 |
|---|---|---|---|
| `draft` | O | O | 자유 편집 허용 |
| `review` | 제한적 | O (재요청 필요) | 수정 시 review → draft 복귀 권장 |
| `published` | X | O (새 draft로) | published 문서는 직접 덮어쓰기 불허 |
| `archived` | X | O (복원 후) | archived → draft 복원 후 수정 |

### 7-2. published 문서 개정 흐름

```
published 문서 개정 시:
  1. 새 draft Version 생성 (Document 상태는 published 유지 가능)
  2. draft 편집
  3. review 요청 → review 상태 전이
  4. 검토 통과 → published 전이 + current_version_id 갱신
  5. 이전 published Version은 이력으로 보존
```

> **핵심**: published 상태 = "현재 독자에게 제공되는 공식 버전이 있다"는 의미.
> 동시에 새 draft 작업이 진행 중일 수 있으며, 이때 Document 상태 관리 정책은 두 가지 방향이 가능하다.
>
> - **방식 A**: Document.status = published 유지, latest_version은 draft Version 참조
> - **방식 B**: Document.status를 latest Version의 상태로 동기화
>
> **결정: 방식 A 채택.** Document.status는 current_version 기준, latest_version으로 draft 작업 추적.

---

## 8. Version 정책과의 관계

### 8-1. published의 소속

**결정: `published`는 Document의 상태다. 단, Version.published_at이 특정 Version의 발행 사실을 기록한다.**

| 개념 | 표현 방식 |
|---|---|
| "이 문서가 현재 공식 배포 상태다" | `Document.status = published` |
| "이 Version이 발행된 시점" | `Version.published_at` (nullable) |
| "현재 공식 버전이 무엇인가" | `Document.current_version_id` |

### 8-2. published 상태와 current_version의 정합성 규칙

- `Document.status = published` 이면 `Document.current_version_id`가 유효한 Version을 가리켜야 한다.
- `Document.current_version_id`가 가리키는 Version은 `published_at`이 설정되어 있어야 한다.
- 두 값의 정합성은 서비스 레이어에서 보장한다.

### 8-3. 개정 중 Document 상태

- Document가 published 상태이면서 동시에 `latest_version_id`가 draft Version을 가리킬 수 있다.
- 이 경우 Document.status는 여전히 published다.
- "개정 중"은 상태가 아니라 latest ≠ current인 상황으로 판단한다.

---

## 9. archived / deleted / deprecated 구분

### 9-1. archived

> **운영 대상에서 제외되었지만 이력 보존을 위해 시스템에 남아 있는 문서 상태다.**

- Document.status = `archived`
- `Document.archived_at` 타임스탬프가 함께 기록된다.
- 기본 검색에서 제외되나 명시적 필터로 조회 가능하다.
- 열람 가능하다. 법적 보존 요건 대응에 적합하다.

### 9-2. deleted

> **deleted는 상태가 아닌 삭제 처리 정책이다.**

- `Document.deleted_at` 필드로 표현한다. (soft delete)
- deleted_at이 설정된 문서는 모든 일반 조회에서 제외된다.
- archived와 달리 명시적 열람도 기본 차단된다.
- 물리 삭제는 별도 정책(데이터 보존 기간 만료 등)에 따라 결정한다.

**archived vs deleted 핵심 차이**:

| | archived | deleted |
|---|---|---|
| 상태 표현 | Document.status = archived | Document.deleted_at IS NOT NULL |
| 열람 가능성 | 명시적 필터로 가능 | 원칙적 불가 |
| 목적 | 이력 보존, 비활성화 | 데이터 제거 처리 |
| 복원 | archived → draft 경로로 복원 | 복원은 정책에 따라 결정 |

### 9-3. deprecated / superseded

> **이 문서가 더 이상 권장되지 않거나 다른 문서로 대체되었음을 나타내는 개념이다.**

- MVP에서는 `archived` 상태로 흡수하거나 metadata로 표현한다.
- 문서 간 관계(이 문서가 어떤 문서로 대체되었는지)는 `Reference` 엔티티 도입 이후 처리한다.
- 후속 확장 시 `Document.status`에 `deprecated` 추가 가능성을 열어둔다.

---

## 10. DocumentType과의 관계

### 10-1. 공통 상태 모델 유지

**결정: MVP에서는 모든 DocumentType이 동일한 공통 상태 모델(`draft → review → published → archived`)을 공유한다.**

### 10-2. 타입별 상태 확장 필요성 검토

| DocumentType | 추가 상태 필요성 | 후속 확장 방향 |
|---|---|---|
| `policy` | 중간~높음 | `effective`(시행 중), `obsolete`(효력 상실) 추가 가능. 단, effective는 metadata(effective_date)로 1차 표현 가능 |
| `report` | 낮음 | `submitted` 추가 가능. MVP에서는 published로 통합 |
| `meeting_note` | 낮음 | 공통 모델로 충분 |
| `template_doc` | 낮음~중간 | `active`/`inactive` 구분. archived로 대체 가능 |
| 나머지 | 낮음 | 공통 모델로 충분 |

### 10-3. DocumentType별 상태 확장 전략

- DocumentType의 `state_profile_ref` (Task 1-6에서 설계)를 통해 후속 Phase에서 타입별 상태 집합을 연결한다.
- 타입별 상태는 공통 상태 모델의 **확장**이지 **대체**가 아니다.
- 예: `policy` 타입의 `effective` 상태는 published의 하위 개념으로 설계한다.

---

## 11. 최소 권장 상태 모델

### 11-1. MVP 상태 집합

| 상태 | 의미 요약 | 편집 가능 | 외부 노출 |
|---|---|---|---|
| `draft` | 작성 중, 미확정 | O | X (기본) |
| `review` | 검토/승인 대기 | 제한적 | X (기본) |
| `published` | 공식 유효/공개 | X (직접 수정 불허) | O |
| `archived` | 비활성 보관 | X (복원 후 가능) | X (기본, 명시적 조회 가능) |

### 11-2. 기본 전이 규칙 요약

```
draft ──────────► review ──────────► published ──────────► archived
  ▲                  │                                         │
  └──────────────────┘                                         │
     (반려)                                              (복원: 새 draft)
  ▲────────────────────────────────────────────────────────────┘

draft ──────────────────────────────────────────────────────► archived
  (예외: 작성 취소)
```

### 11-3. 상태 적용 단위 권장안

| 엔티티 | 상태 표현 방식 | 비고 |
|---|---|---|
| `Document` | `status` 필드 (draft/review/published/archived) | 핵심 상태 단위 |
| `Document` | `archived_at` 필드 | archived 전이 시 기록 |
| `Document` | `deleted_at` 필드 | soft delete, 상태 아님 |
| `Version` | `published_at` 필드 | 발행 사실 기록. 상태가 아닌 타임스탬프 |
| `Version` | `review_status` (후속) | 승인 워크플로 도입 시 추가 |

### 11-4. 후속 확장 가능 상태

| 상태 | 도입 시점 | 비고 |
|---|---|---|
| `deprecated` | Phase 3~4 | policy 유형 등에서 필요 |
| `superseded` | Phase 3~4 | 문서 간 대체 관계와 함께 도입 |
| `rejected` | Phase 5 | 승인 워크플로 도입 시 |
| `scheduled` | Phase 5 | 예약 배포 기능 도입 시 |
| `effective` | Phase 3~4 | policy 타입 전용 상태 확장 |

---

## 12. 설계 쟁점과 판단 결과

| 쟁점 | 선택한 방향 | 이유 |
|---|---|---|
| 상태를 Document에만 둘지 | **Document 중심 상태 모델 채택** | 단순성과 실용성. "이 문서의 현재 상태"는 Document 수준 질문 |
| Version 상태를 초기부터 둘지 | **`published_at` 타임스탬프만 유지, 상태 전이는 후속** | Version은 불변이므로 상태 전이 개념이 약함. 발행 사실만 기록 |
| `review` 상태를 코어에 포함할지 | **포함** | 검토 흐름 없이 draft → published 직행은 너무 단순. 최소 검토 단계 표현 필요 |
| `archived`를 상태로 둘지 | **상태로 포함** | 비활성화와 삭제를 구분해야 이력 보존 정책 표현 가능 |
| `deleted`를 상태로 둘지 | **상태 아님. `deleted_at` 필드로 분리** | 삭제는 생명주기 단계가 아닌 데이터 처리 정책. 상태와 혼용하면 조회 로직 복잡 |
| `published` 문서를 직접 수정 가능하게 둘지 | **직접 수정 불허. 새 draft Version 생성으로 개정** | 불변성 원칙 + 이력 추적 가능성 보장. 직접 수정은 이력 단절 위험 |
| `deprecated`/`superseded`를 초기 상태로 넣을지 | **초기 제외. archived로 흡수 또는 후속 확장** | 초기 과설계 방지. 문서 간 대체 관계는 Reference 엔티티와 함께 도입 |

---

## 13. 다음 Task 입력값

### Task 1-9 (관계/무결성/제약 조건 정의)로 넘길 포인트

- `Document.status = published`이면 `Document.current_version_id`가 NULL이 아니어야 한다.
- `Document.current_version_id`가 가리키는 Version은 `published_at`이 설정되어 있어야 한다.
- `Document.archived_at` IS NOT NULL이면 `Document.status = archived`여야 한다.
- `Document.deleted_at` IS NOT NULL이면 모든 조회에서 제외해야 한다.
- 상태 전이는 허용된 경로만 가능하다. (서비스 레이어 검증 규칙으로 명시)
- archived 문서의 Version은 삭제하지 않는다. (이력 보존 무결성)

### 저장 구조 설계 시 확인해야 할 상태 관련 쟁점

- `Document.status`를 문자열로 저장할지 정수 코드로 저장할지.
- `archived_at`, `deleted_at`을 별도 컬럼으로 유지할지, status와 통합할지.
- status별 문서 조회 성능을 위한 인덱스 전략 (`status`, `deleted_at` 조합 인덱스).
- soft delete 문서를 물리 삭제로 전환하는 정책과 시점.

### API 설계 시 필요한 상태 전이 제약 포인트

- 상태 전이 API는 직접 status 값을 교체하는 방식보다 전이 액션 기반(예: `POST /documents/{id}/publish`)으로 설계하는 것이 바람직하다.
- 허용되지 않은 전이 요청 시 명확한 오류 응답이 필요하다.
- `deleted_at` 설정 API는 별도 삭제 엔드포인트로 분리한다.
- `archived → draft` 복원 API는 새 Version 생성과 함께 처리된다.

### UI 문서 목록/상세/편집 플로우에 영향을 줄 판단

- 기본 문서 목록은 `deleted_at IS NULL AND status != archived` 조건으로 표시한다.
- 아카이브 문서 조회는 별도 필터(archived 포함)로 접근한다.
- 문서 상세 화면에서 현재 status에 따라 편집 버튼 활성화 여부가 결정된다.
- `published` 상태 문서의 편집 버튼은 "새 버전으로 개정" 흐름을 시작한다.
- `review` 상태 문서는 편집 제한 안내와 함께 "검토 반려 후 수정" 경로를 안내한다.

### 승인/배포 워크플로 확장 시 남겨둘 포인트

- `review` 상태의 내부 세부 단계(검토자 지정, 검토 의견, 승인/반려)는 Phase 5 이후 `Approval` 엔티티와 함께 설계한다.
- `Version.review_status` 필드 추가 가능성을 열어둔다. (Version 단위 승인 이력)
- DocumentType별 상태 확장(`policy.effective`, `report.submitted`)은 `state_profile_ref` 연결로 도입한다.
- 승인 워크플로 도입 시 `review → published` 전이가 자동 승인에서 명시적 승인으로 강화된다.
