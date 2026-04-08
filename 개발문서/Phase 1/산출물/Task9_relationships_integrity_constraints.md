# Phase 1 - Task 1-9. 관계 / 무결성 / 제약 조건 정의

## 1. 작업 목적

Task 1-1~1-8에서 확정한 요구사항, 엔티티 책임, Document/Version/Node/DocumentType/metadata/상태 모델을 바탕으로,
**범용 문서 플랫폼의 핵심 도메인 모델이 실제 구현 단계에서도 흔들리지 않도록 엔티티 간 관계, 무결성 규칙, 핵심 제약 조건**을 정리한다.

이 문서는 DB 설계, API 설계, 버전 관리, 편집 로직, 상태 전이 로직에서 반드시 지켜져야 할
**도메인 수준의 불변조건(invariant)** 을 명확히 확정한다.

---

## 2. 관계 정의 대상

### 2-1. 핵심 관계 대상

| 엔티티/개념 | 관계 정의 필요 이유 |
|---|---|
| `Document` ↔ `Version` | 정체성과 시점 내용의 분리 경계 |
| `Version` ↔ `Node` | 본문 구조 귀속 단위 |
| `Document` ↔ `DocumentType` | 유형 분류 및 확장 규칙 연결 |
| `metadata` ↔ `Document/Version/Node` | 확장 속성의 귀속 수준 |
| `Document.status` ↔ `Version` | 상태와 버전 발행의 정합성 |
| `Document.current_version_id` | 현재 공식 버전 참조 정합성 |
| `Node.parent_id` | 트리 구조 정합성 |

### 2-2. 보조 개념 관계

| 개념 | 관계 |
|---|---|
| `current_version` | Document가 가리키는 현재 공식 버전 |
| `latest_version` | Document가 가리키는 최신 생성 버전 (draft 포함) |
| `root_node` | Version의 트리 진입점 Node |
| `state_transition` | Document.status의 허용 전이 경로 |
| `deletion/archive policy` | soft delete(deleted_at) vs archived 상태 구분 |

---

## 3. 핵심 엔티티 간 관계

### 3-1. 관계 요약표

| 관계 | 유형 | 핵심 의미 |
|---|---|---|
| Document : Version | 1 : N | 하나의 Document는 여러 Version을 가질 수 있다 |
| Version : Document | N : 1 | 모든 Version은 반드시 하나의 Document에 속한다 |
| Version : Node | 1 : N | 하나의 Version은 여러 Node를 가진다 |
| Node : Version | N : 1 | 모든 Node는 반드시 하나의 Version에 속한다 |
| Node : Node (parent) | N : 1 | Node는 같은 Version 내 다른 Node를 부모로 가질 수 있다 |
| Document : DocumentType | N : 1 | 모든 Document는 반드시 하나의 DocumentType을 가진다 |
| Document.current_version_id | Document → Version | 현재 공식 버전 참조. 반드시 해당 Document의 Version이어야 함 |
| Document.latest_version_id | Document → Version | 최신 생성 버전 참조. 반드시 해당 Document의 Version이어야 함 |
| Version.root_node_id | Version → Node | 트리 진입점 참조. 반드시 해당 Version의 Node이어야 함 |
| Version.base_version_id | Version → Version | 이력 체인 참조. 반드시 같은 Document의 Version이어야 함 |
| metadata | Document/Version/Node에 부속 | 독립 존재 불가. 반드시 부모 엔티티에 귀속 |

### 3-2. Document ↔ Version

- Document 1 : N Version — 하나의 문서는 생명주기 동안 여러 버전을 가진다.
- Version은 Document 없이 독립 존재할 수 없다.
- Document는 Version 없이도 존재할 수 있다. (초기 생성 직후, draft 편집 전)
- `current_version_id`는 nullable이다. (Document 생성 직후 첫 Version 저장 전)
- `latest_version_id`는 current와 다를 수 있다. (draft 편집 중)

### 3-3. Version ↔ Node

- Version 1 : N Node — 하나의 Version은 자신의 Node 트리를 소유한다.
- Node는 Version 없이 독립 존재할 수 없다.
- Version은 최소 1개의 Node(루트 노드)를 가져야 한다. (content가 있는 Version 기준)
- Version 간 Node 공유는 허용하지 않는다. (Version별 독립 Node 원칙)

### 3-4. Document ↔ DocumentType

- Document는 반드시 하나의 DocumentType을 가진다. type 없는 Document는 허용하지 않는다.
- 분류 미정 또는 기타 문서는 `generic_doc` fallback type을 사용한다.
- DocumentType은 여러 Document가 공유할 수 있다.

### 3-5. metadata 귀속 관계

- `Document.metadata`: 문서 전체 생명주기에 걸친 운영 속성. Version과 무관.
- `Version.metadata_snapshot`: 해당 버전 생성 시점에 종속된 속성. 불변.
- `Node.attributes`: node_type별 구조·표시 옵션. 해당 Node에 귀속.
- `Node.metadata`: 노드 수준 운영 메타. 해당 Node에 귀속.

### 3-6. 상태 ↔ Document / Version

- 상태(`Document.status`)는 Document 수준에서 관리한다.
- Version은 `published_at` 타임스탬프로 발행 사실만 기록한다.
- `Document.status = published`이면 `current_version_id`가 유효한 published Version을 가리켜야 한다.

---

## 4. Document 관련 무결성 규칙

| 규칙 | 내용 |
|---|---|
| D-1 | Document는 시스템 전체에서 고유한 식별자(`id`)를 가져야 한다 |
| D-2 | Document는 반드시 하나의 DocumentType을 가져야 한다. type 없는 Document는 허용하지 않는다 |
| D-3 | Document.title은 필수다. 빈 문자열은 허용하지 않는다 |
| D-4 | Document.status는 정의된 상태 집합(`draft`, `review`, `published`, `archived`) 내 값이어야 한다 |
| D-5 | Document.current_version_id가 존재한다면, 반드시 해당 Document의 Version을 가리켜야 한다 |
| D-6 | Document.latest_version_id가 존재한다면, 반드시 해당 Document의 Version을 가리켜야 한다 |
| D-7 | Document.status = `published`이면 current_version_id는 NULL이 아니어야 한다 |
| D-8 | Document.status = `archived`이면 archived_at이 설정되어 있어야 한다 |
| D-9 | Document.deleted_at이 설정된 문서는 일반 조회 결과에서 제외되어야 한다 |
| D-10 | Document는 Version 없이 초기 생성 상태로 존재할 수 있다. 이 경우 current_version_id = NULL |
| D-11 | Document의 DocumentType은 활성(`is_active = true`) 타입이어야 한다 |

---

## 5. Version 관련 무결성 규칙

| 규칙 | 내용 |
|---|---|
| V-1 | Version은 반드시 하나의 Document에 속해야 한다. 고아 Version은 허용하지 않는다 |
| V-2 | version_no는 동일 Document 내에서 중복될 수 없다 |
| V-3 | Version은 생성 후 원칙적으로 불변이다. Node 트리, metadata_snapshot, title_snapshot은 변경하지 않는다 |
| V-4 | Version.root_node_id가 존재한다면, 반드시 해당 Version의 Node를 가리켜야 한다 |
| V-5 | Version.base_version_id가 존재한다면, 반드시 같은 Document 내 Version을 가리켜야 한다 |
| V-6 | Version.created_at은 불변이다. 생성 후 수정하지 않는다 |
| V-7 | 같은 Document의 두 Version이 동시에 `Document.current_version`이 될 수 없다 |
| V-8 | Version.published_at이 설정된 Version은 Document.current_version_id의 후보가 될 수 있다 |
| V-9 | Document가 soft delete(`deleted_at` 설정)되어도 하위 Version은 물리 삭제하지 않는다. 이력 보존 원칙 |
| V-10 | version_no는 Document 범위 내에서 단조 증가해야 한다. 이전 번호 재사용은 허용하지 않는다 |

---

## 6. Node 관련 무결성 규칙

| 규칙 | 내용 |
|---|---|
| N-1 | Node는 반드시 하나의 Version에 속해야 한다. 고아 Node는 허용하지 않는다 |
| N-2 | Node.parent_id가 존재한다면, 반드시 같은 Version 내 Node를 가리켜야 한다. 다른 Version의 Node를 참조할 수 없다 |
| N-3 | Node는 자기 자신을 parent_id로 가질 수 없다 (자기 참조 금지) |
| N-4 | Node 트리는 순환(cycle)을 가져서는 안 된다. 모든 Node는 루트까지 도달 가능해야 한다 |
| N-5 | 루트 Node(`document_root`)는 parent_id가 NULL이어야 한다 |
| N-6 | 하나의 Version에는 `document_root` 타입 Node가 정확히 1개 존재해야 한다 |
| N-7 | 동일 부모를 가진 형제 Node들 사이에서 order 값은 일관된 정렬 기준을 제공해야 한다 |
| N-8 | node_type은 플랫폼이 정의한 허용 타입 집합에 속하거나, `custom_block` fallback 처리 대상이어야 한다 |
| N-9 | `document_root` 타입 Node는 content를 직접 가지지 않는다. 구조 역할만 담당한다 |
| N-10 | `depth` 또는 `path` 보조 필드를 사용하는 경우, 해당 값은 실제 트리 구조와 정합해야 한다 |
| N-11 | Node가 속한 Version이 soft delete된 경우, 해당 Node도 함께 비활성 처리되어야 한다 |

---

## 7. 상태 관련 제약

### 7-1. 상태 전이 제약

| 전이 | 허용 여부 | 조건 |
|---|---|---|
| `draft → review` | O | 최소 하나의 Version이 존재해야 함 |
| `review → draft` | O | 반려. 제약 없음 |
| `review → published` | O | current_version_id 갱신 필요. 해당 Version에 published_at 설정 필요 |
| `published → archived` | O | archived_at 설정 필요 |
| `draft → archived` | O (예외) | 작성 취소 목적. 명시적 요청만 허용 |
| `archived → draft` | O (복원) | 새 Version 생성으로 처리. 원본 보존 |
| `published → draft` | 제한적 | Document.status = published 유지한 채 새 draft Version 생성 권장 |
| `archived → published` | X | draft → review → published 경유 필요 |
| 자유 전이 (임의 상태 변경) | X | 허용되지 않는 전이는 서비스 레이어에서 거부 |

### 7-2. 상태별 핵심 제약

- `published` 상태 문서는 `current_version_id`가 반드시 존재해야 한다.
- `published` 상태 문서의 직접 내용 수정은 허용하지 않는다. 새 draft Version 생성 흐름을 사용한다.
- `archived` 상태 문서는 신규 편집 시작 전 복원(새 draft Version 생성) 절차를 거친다.
- `deleted_at`이 설정된 문서는 상태 전이 대상이 아니다. 삭제는 상태가 아닌 별도 처리다.
- `review` 상태에서 허용되는 수정 정책은 운영 정책에서 결정하며, 최소 제약은 "수정 시 review → draft 복귀"다.

---

## 8. DocumentType 관련 제약

| 규칙 | 내용 |
|---|---|
| DT-1 | Document의 document_type은 반드시 존재하는 유효한 DocumentType key여야 한다 |
| DT-2 | 비활성(`is_active = false`) DocumentType으로 새 Document를 생성할 수 없다 |
| DT-3 | 분류 미정 또는 기타 문서는 `generic_doc` fallback type을 사용한다. type 없는 Document는 허용하지 않는다 |
| DT-4 | DocumentType별 Node 구성 규칙은 MVP에서 권장 수준이다. 위반 시 경고만 발생하며 저장을 거부하지 않는다 |
| DT-5 | DocumentType별 metadata template 준수는 MVP에서 권장 수준이다. 검증은 서비스 레이어에서 담당한다 |
| DT-6 | `generic_doc` 타입은 Node 구성 제약을 갖지 않는다. 모든 node_type 허용 |

---

## 9. metadata 관련 최소 제약

| 규칙 | 내용 |
|---|---|
| M-1 | metadata는 반드시 부모 엔티티(Document/Version/Node)에 귀속된다. 독립 존재 불가 |
| M-2 | 코어 필드(`title`, `status`, `document_type` 등)와 동일한 의미의 값을 metadata에 중복 저장하지 않는다 |
| M-3 | metadata key는 snake_case 명명 규칙을 따른다 |
| M-4 | `common.*` namespace의 키는 플랫폼이 정의한 표준 키 목록을 따른다. 임의 추가를 허용하지 않는다 |
| M-5 | DocumentType별 확장 metadata는 해당 타입의 namespace(`{type_key}.*`)를 사용한다 |
| M-6 | Version.metadata_snapshot은 Version 생성 시점에 고정되며 이후 변경하지 않는다 (불변 원칙) |
| M-7 | Node.attributes는 node_type의 구조·표시 옵션에 해당하는 값만 포함한다. 문서 수준 정보를 Node attributes에 넣지 않는다 |
| M-8 | DocumentType별 metadata template 준수 여부 검증은 서비스 레이어에서만 수행한다. DB 레벨 강제는 하지 않는다 |

---

## 10. 최소 불변조건(Invariants) 목록

> 이 목록은 플랫폼 코어 모델이 어떤 상태에서도 항상 만족해야 하는 조건이다.

1. 모든 Version은 정확히 하나의 Document에 속해야 한다.
2. 모든 Node는 정확히 하나의 Version에 속해야 한다.
3. Document.current_version_id는 해당 Document의 Version만 가리킬 수 있다.
4. Document.latest_version_id는 해당 Document의 Version만 가리킬 수 있다.
5. Version.root_node_id는 해당 Version의 Node만 가리킬 수 있다.
6. Version.base_version_id는 같은 Document 내 Version만 가리킬 수 있다.
7. Node.parent_id는 같은 Version 내 Node만 가리킬 수 있다.
8. Node는 자기 자신을 parent_id로 가질 수 없다.
9. Node 트리는 순환(cycle)을 포함할 수 없다.
10. 모든 Version에는 `document_root` 타입 루트 Node가 정확히 1개 존재해야 한다.
11. `document_root` Node의 parent_id는 NULL이어야 한다.
12. Document.status = `published`이면 current_version_id는 NULL이 아니어야 한다.
13. Document.current_version_id가 가리키는 Version의 published_at은 NULL이 아니어야 한다.
14. Document.status = `archived`이면 archived_at이 설정되어 있어야 한다.
15. Document는 반드시 하나의 DocumentType을 가져야 한다.
16. Document의 DocumentType은 활성(`is_active = true`) 타입이어야 한다.
17. version_no는 동일 Document 내에서 중복될 수 없다.
18. Version은 생성 후 Node 트리, metadata_snapshot, title_snapshot을 변경할 수 없다.
19. Document가 soft delete되어도 하위 Version과 Node는 물리 삭제하지 않는다.
20. 상태 전이는 정의된 허용 경로만 따른다. 임의 전이는 허용하지 않는다.

---

## 11. 제약 강도 분류

### 11-1. Hard Invariant (항상 반드시 지켜져야 하는 규칙)

| 규칙 번호 | 내용 요약 |
|---|---|
| 1, 2 | Version과 Node의 단일 부모 엔티티 귀속 |
| 3, 4, 5, 6, 7 | 모든 참조 정합성 규칙 |
| 8, 9 | Node 트리 순환 금지 |
| 10, 11 | 루트 Node 유일성 및 parent_id 규칙 |
| 12, 13 | published 상태의 current_version 정합성 |
| 17 | version_no 유일성 |
| 18 | Version 불변성 |

### 11-2. Soft Rule / Policy Hint (운영 정책에 따라 달라질 수 있는 규칙)

| 규칙 | 내용 요약 |
|---|---|
| D-3 (title 필수) | 빈 title을 허용하는 임시 저장 기능 도입 시 완화 가능 |
| D-8 (archived_at) | 상태 전이 자동화 도입 시 정책 조정 가능 |
| V-9 (삭제 시 보존) | 데이터 보존 정책에 따라 TTL 기반 물리 삭제 허용 가능 |
| N-7 (order 일관성) | 순서 정렬 방식 변경 시 조정 가능 |
| 20 (상태 전이 제한) | 관리자 권한 등 특수 케이스에서 일부 예외 허용 가능 |
| M-5 (namespace 규칙) | 초기 운영 중 네이밍 체계 정립 전까지 경고 수준 |

### 11-3. Future Constraint (후속 Phase에서 강제 가능성이 큰 규칙)

| 규칙 | 내용 요약 | 강제 예상 시점 |
|---|---|---|
| DT-4 (Node 구성 규칙) | DocumentType별 허용 Node 강제 검증 | Phase 3~4 |
| DT-5 (metadata 검증) | DocumentType별 필수 metadata 강제 | Phase 3 |
| M-4 (common namespace 표준 키) | 표준 키 외 common 키 거부 | Phase 2~3 |
| review 상태 수정 금지 | review 상태에서 직접 수정 거부 | 승인 워크플로 도입 시 |
| Version 소프트 삭제 정책 | archived/deleted Version 처리 정책 명확화 | Phase 2 |

---

## 12. 최소 관계/제약 기준안

### 12-1. 핵심 관계 요약

```
DocumentType (1) ◄──── (N) Document (1) ────► (N) Version (1) ────► (N) Node
                             │                      │                    │
                    current_version_id         root_node_id         parent_id
                    latest_version_id          base_version_id      (같은 Version 내)
                             │                      │
                         metadata              metadata_snapshot
                                               (불변 스냅샷)
```

### 12-2. 필수 정합성 규칙

- 모든 참조(current_version_id, root_node_id, parent_id 등)는 정의된 범위 내 엔티티만 가리킬 수 있다.
- Version은 Document에, Node는 Version에 종속된다. 역방향 종속은 없다.
- 루트 Node는 Version당 정확히 1개이며 parent_id = NULL이다.

### 12-3. 상태 관련 핵심 제약

- published 상태 = current_version_id 존재 + 해당 Version.published_at 설정.
- 상태 전이는 허용 경로만 따른다. 서비스 레이어에서 검증한다.
- deleted_at은 상태가 아닌 별도 소프트 삭제 필드다.

### 12-4. 트리 구조 핵심 제약

- Node.parent_id는 같은 Version 내 Node만 참조한다.
- 순환 없음. 루트 노드 유일. 자기 참조 금지.
- Version 간 Node 공유 없음.

### 12-5. metadata 관련 최소 원칙

- 수준별 귀속 원칙: Document-level은 운영 속성, Version-level은 시점 속성, Node-level은 블록 속성.
- 코어 필드 중복 금지.
- snake_case + namespace 규칙 적용.
- 검증은 서비스 레이어 담당.

### 12-6. 후속 확장 포인트

- DocumentType별 Node 구성 강제 검증 (Phase 3~4)
- DocumentType별 metadata 필수 필드 강제 (Phase 3)
- Version 단위 승인 상태 확장 (Phase 5)
- 문서 간 참조/대체 관계(Reference 엔티티) 도입 (Phase 2~3)

---

## 13. 설계 쟁점과 판단 결과

| 쟁점 | 선택한 방향 | 이유 |
|---|---|---|
| Document 생성 시 Version 없이 존재 가능하게 둘지 | **허용** | 문서 초기 생성 시 빈 Document 상태 존재 가능. current_version_id = NULL로 표현 |
| current_version_id를 optional로 둘지 | **Optional (nullable)** | 초기 생성 직후, draft 저장 전 상태를 표현해야 함. published 상태에서는 Non-null 강제 |
| Version당 root node를 정확히 1개로 강제할지 | **강제** | root가 0개이면 트리 진입 불가. 2개 이상이면 트리 정의 불가. 구조 무결성의 핵심 |
| generic_doc fallback을 강제할지 | **강제** | type 없는 Document를 허용하면 DocumentType 기반 확장 규칙 전체가 흔들림 |
| type별 Node 제약을 초기부터 강하게 둘지 | **MVP: 권장 수준. 후속 강제** | 초기 유연성 유지. 운영 경험 후 필요한 타입만 선별 강제 |
| metadata template 준수를 강제할지 | **MVP: 서비스 레이어 권장 수준** | DB 레벨 강제는 불필요. 서비스 레이어에서 타입별 분기 처리 (CLAUDE.md 원칙) |
| archived 문서 수정 경로 | **archived → 새 draft Version 생성 (복원)** | 직접 수정 불허. 불변성 + 이력 보존 원칙 |
| deleted를 상태에서 제외할지 | **제외. deleted_at 필드로 분리** | 삭제는 생명주기 단계가 아닌 데이터 처리 정책. 상태와 혼용 시 조회 로직 복잡화 |

---

## 14. 다음 Task 입력값

### Task 1-10 (통합 모델 리뷰 및 기준안 확정)으로 넘길 핵심 포인트

- 20개 불변조건 목록이 통합 모델 검증의 기준이 된다.
- Hard Invariant / Soft Rule / Future Constraint 분류를 기반으로 MVP 구현 범위를 확정한다.
- 각 Task에서 내린 설계 판단 중 상충되는 규칙이 있는지 통합 검토가 필요하다.
- DocumentType ↔ Node 규칙 연결, metadata schema 연결의 설계 예비 구조가 통합 모델에 포함되어야 한다.

### 저장 구조/DB 설계 시 바로 반영해야 할 규칙

- `Document.current_version_id`, `Document.latest_version_id`: 같은 Document의 Version ID만 허용. FK 제약 또는 서비스 레이어 검증.
- `Version.root_node_id`: 같은 Version의 Node ID만 허용.
- `Node.parent_id`: 같은 Version 내 Node ID만 허용. 다른 Version 참조 불가.
- `version_no`: Document 범위 내 유일성 보장. 복합 유니크 제약(`document_id + version_no`).
- `Document.status`: 정의된 값 목록 제약.
- soft delete(`deleted_at`) 컬럼은 기본 조회 필터에 포함.

### API 설계 시 유효성 검증 포인트

- 문서 생성 API: `document_type`은 활성 DocumentType key여야 함.
- Version 생성 API: 해당 Document 존재 여부, 불변 필드 수정 시도 거부.
- Node 생성 API: `parent_id`가 같은 Version의 Node인지 검증. 순환 참조 방지 로직 필요.
- 상태 전이 API: 허용된 전이 경로인지 검증. 비허용 전이 요청 시 명확한 오류 반환.
- `published` 전이 API: `current_version_id` 갱신 + `Version.published_at` 설정 원자적 처리.

### 에디터/문서 편집 플로우에서 강하게 의식해야 할 제약

- `published` 상태 문서의 편집 시도는 새 draft Version 생성 흐름으로 안내해야 한다.
- Node 편집(추가/수정/삭제)은 현재 Version의 Node를 수정하는 것이 아니라 새 Version 생성을 유도한다.
- `Node.parent_id`는 항상 같은 Version 내 Node만 참조해야 한다. 에디터에서 Node 이동 시 검증 필요.
- `document_root` Node는 에디터에서 삭제할 수 없어야 한다.
- 형제 Node 재정렬 시 `order` 값의 일관성을 유지해야 한다.

### 상태 전이/버전 생성 로직에서 꼭 지켜야 할 불변조건

- `review → published` 전이 시: `Document.current_version_id` 갱신 + `Version.published_at` 설정을 원자적으로 처리해야 한다.
- Version 생성 시: `document_root` 타입 루트 Node를 반드시 함께 생성해야 한다.
- 복원(archived → draft) 시: 원본 Version은 변경하지 않고 새 Version을 생성한다.
- Document soft delete 시: 하위 Version과 Node는 물리 삭제하지 않는다.
- `published` 상태 전이 시: `current_version_id`가 가리키는 Version의 `published_at`이 설정되어 있는지 반드시 검증한다.
