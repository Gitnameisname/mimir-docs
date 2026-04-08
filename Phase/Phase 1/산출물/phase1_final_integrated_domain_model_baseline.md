# Phase 1 최종 통합 도메인 모델 기준안
## phase1_final_integrated_domain_model_baseline

---

## 1. 작업 목적

이 문서는 Mimir 범용 문서 플랫폼의 Phase 1 최종 기준 문서다.

Task 1-1부터 1-9까지 개별 설계된 요구사항, 핵심 엔티티, Document/Version/Node 구조, DocumentType 시스템, metadata 확장 구조, 상태 모델, 관계/무결성/제약 조건을 하나의 일관된 기준안으로 통합한다.

이 문서의 목적은 단순 요약이 아니다. 앞선 작업 결과들 사이에 남아 있던 중복, 충돌, 불명확한 경계를 정리하여 Phase 2(저장 구조/DB 설계), Phase 3(API 설계), Phase 4(UI/편집/렌더링 설계)의 입력 기준으로 활용한다.

---

## 2. Phase 1 결과 통합 개요

### 2-1. Task별 결과 요약

| Task | 작업 내용 | 핵심 결론 |
|---|---|---|
| **Task 1-1** | 범용 문서 도메인 요구사항 정리 | 7종 문서 유형 식별. 공통 요구사항(식별·구조·버전·메타·상태·확장성) 확정. 코어 엔티티는 Document/Version/Node 3개 |
| **Task 1-2** | 핵심 엔티티 식별 및 책임 정의 | Document(정체성), Version(불변 스냅샷), Node(구조 단위) 코어 확정. DocumentType·Metadata·DocumentStatus는 보조 개념으로 분류 |
| **Task 1-3** | Document 구조 설계 | Document = 껍데기(identity) + 포인터(current_version_id) + 대표 정보. current_version_id와 latest_version_id 분리 운영 결정 |
| **Task 1-4** | Version 구조 설계 | Version은 특정 시점 불변 스냅샷. title_snapshot + metadata_snapshot 직접 보관. Draft = published_at NULL |
| **Task 1-5** | Node 구조 설계 | Node는 Version 종속 구조 단위. parent_id + order 기반 트리. 단일 공통 모델 + node_type 역할 구분 |
| **Task 1-6** | DocumentType 시스템 설계 | 8종 기본 타입(generic_doc 포함). 문자열 key 기반 레코드 방식. MVP: 권장 수준 node rule, 후속 강제 확장 |
| **Task 1-7** | metadata 확장 구조 설계 | common + extended 2영역 구조. snake_case + namespace 규칙. 검증은 서비스 레이어만 담당 |
| **Task 1-8** | 문서 상태 모델 기초 설계 | 4상태: draft → review → published → archived. Document 중심 상태 모델. deleted는 상태 아님(deleted_at 필드) |
| **Task 1-9** | 관계/무결성/제약 조건 정의 | 20개 불변조건 확정. Hard Invariant / Soft Rule / Future Constraint 3단계 분류 |

### 2-2. 전체 설계 흐름 개요

```
[요구사항 (Task1-1)]
  └─► [엔티티 책임 확정 (Task1-2)]
        ├─► [Document 구조 (Task1-3)]
        ├─► [Version 구조 (Task1-4)]
        ├─► [Node 구조 (Task1-5)]
        ├─► [DocumentType 시스템 (Task1-6)]
        ├─► [metadata 확장 (Task1-7)]
        ├─► [상태 모델 (Task1-8)]
        └─► [관계/무결성/제약 (Task1-9)]
              └─► [통합 기준안 (Task1-10) ← 이 문서]
```

---

## 3. 코어 모델 구성요소 최종 확정

### 3-1. 코어 구성요소 요약표

| 구성요소 | 분류 | 한 줄 정의 | 왜 코어인가 | 코어 바깥에 두는 것 |
|---|---|---|---|---|
| `Document` | **코어 엔티티** | 문서의 지속적 정체성을 표현하는 최상위 단위 | 모든 참조의 기준점. 문서가 "존재한다"는 사실 자체를 표현 | 본문 내용, 버전별 상세, 구조 표현 |
| `Version` | **코어 엔티티** | 특정 시점 문서 상태를 재현 가능하게 보존하는 불변 스냅샷 | 이력 관리와 재현 가능성의 핵심. Node 트리의 소유 단위 | 문서 정체성, 유형 분류, 전체 생명주기 상태 |
| `Node` | **코어 엔티티** | Version에 종속된 문서 본문의 최소 구조 단위 | 모든 본문 구조 표현이 Node 트리로 수렴. 내용과 구조를 동시에 표현 | 버전 관리, 문서 정체성, 유형 규칙 적용 |
| `DocumentType` | **코어 분류/규칙 개념** | 문서 유형을 식별하고 유형별 확장 규칙의 기준을 정의하는 독립 관리 개념 | Document의 분류 기준이자 metadata 스키마·Node 규칙의 연결 기준점 | 개별 문서 상태 관리, 본문 내용 저장 |
| `metadata` | **코어 확장 구조** | Document·Version·Node 각각에 부속되는 공통 및 유형별 확장 속성 구조 | 코어를 단순하게 유지하면서 유형별 차이를 수용하는 확장 지점 | 독립 존재(반드시 부모 엔티티에 귀속) |
| `DocumentStatus` | **코어 운영 개념** | 문서 생명주기 단계를 표현하는 상태 모델 | 가시성·수정 가능성·공식성·보관 여부의 운영 기준 | 승인 절차 상세, 권한 제어 |

### 3-2. 분류 기준 정리

- **코어 엔티티**: 독립적으로 존재하며 직접 저장되는 핵심 레코드 (Document, Version, Node)
- **코어 분류/규칙 개념**: 독립 관리 대상이지만 문서 자체보다 분류·규칙 정의 역할 (DocumentType)
- **코어 확장 구조**: 독립 엔티티가 아닌 부모 엔티티의 확장 속성 (metadata)
- **코어 운영 개념**: 독립 엔티티가 아닌 Document의 속성 + 전이 규칙 (DocumentStatus)

### 3-3. 코어 바깥에 두는 것

| 항목 | 분류 | 도입 시점 |
|---|---|---|
| `Attachment` | 부가 엔티티 | Phase 2 이후 |
| `Reference` | 부가 엔티티 | Phase 2~3 |
| `AuditEvent` | 부가 엔티티 | Phase 2 이후 |
| `Approval` | 후속 확장 | Phase 5 이후 |
| `Publication` | 후속 확장 | Phase 5 이후 |
| `Relation` | 후속 확장 | Phase 5 이후 |
| `Template` (별도 엔티티) | 후속 확장 | Phase 3~4 이후 |

---

## 4. 최종 책임 분리 기준안

### 4-1. 각 요소의 책임

| 구성요소 | 책임하는 것 | 책임하지 않는 것 |
|---|---|---|
| **Document** | 문서의 고유 식별, 유형 분류, 현재/최신 버전 참조, 생명주기 상태, 소속 정보, 소프트 삭제 | 특정 시점 본문 내용, 버전별 상세 속성, 본문 계층 구조 표현, 승인 절차 |
| **Version** | 특정 시점 제목·요약·metadata 스냅샷, Node 트리 루트 참조, 생성자·생성일시, 변경 설명, 이력 체인 참조 | 문서 정체성 관리, 전체 생명주기 상태, 유형 분류, 본문 구조 직접 표현 |
| **Node** | 노드 유형 식별, 부모-자식 관계 표현, 형제 순서 표현, 제목·본문 내용 보관, 노드 수준 속성 | 특정 시점 스냅샷 보장(Version 책임), 버전 번호 관리, 문서 유형 분류, 생명주기 상태 |
| **DocumentType** | 유형 식별값, metadata 스키마 참조, Node 구성 규칙 참조, 상태 프로필 참조 | 개별 문서 상태 관리, 버전 내용 저장, 구조 표현, 권한/승인 워크플로 |
| **metadata** | 유형별 확장 속성 수용, 공통 부가 속성 보관, 검색/필터 보조 기준 제공 | 코어 필드 대체, 독립 존재(부모 엔티티에 귀속 필수), 상태 전이, 버전 생명주기 관리 |
| **DocumentStatus** | draft·review·published·archived 상태 표현, 상태 전이 규칙 기준 제공, 가시성/수정 가능성 판단 기준 | 버전 내용 저장, 구조 관리, 승인 워크플로 상세, 권한 제어 |

### 4-2. 서로 겹치기 쉬운 부분의 최종 경계

#### Document vs Version

| 항목 | Document | Version |
|---|---|---|
| 문서 대표 제목 | `title` (현재 공식 제목. 목록/검색/참조에 사용) | `title_snapshot` (해당 버전 시점 제목 복사본. 이력 재현용) |
| metadata | Document-level metadata (버전 무관, 지속적 운영 속성) | `metadata_snapshot` (해당 버전 시점에 종속된 속성. 불변) |
| 상태 | `status` (전체 생명주기 상태) | `published_at` (발행 사실 타임스탬프만 기록) |
| 현재 버전 | `current_version_id` 포인터만 보유 | 실제 스냅샷 내용 보관 |

> **원칙**: Document는 "이 문서가 지금 어떤 상태인가"를 안다. Version은 "이 시점에 문서가 어떤 내용이었나"를 안다.

#### Version vs Node

| 항목 | Version | Node |
|---|---|---|
| 역할 | 특정 시점의 문서 상태를 재현 가능하게 보존 | 해당 시점 문서의 내부 구조를 표현 |
| 관계 | Node 트리의 루트를 참조 (`root_node_id`) | Version에 종속. Version 없이 독립 존재 불가 |
| 공유 | Version 간 Node 공유 없음 | 특정 Version에만 귀속 |

> **원칙**: Version은 컨테이너(시점 단위). Node는 그 컨테이너 안의 내용(구조 단위).

#### 코어 필드 vs metadata

| 코어 필드로 두는 기준 | metadata로 두는 기준 |
|---|---|
| 거의 모든 문서에 공통으로 필요 | 특정 유형에만 필요 |
| 정체성·관계·상태·버전 무결성에 직접 영향 | 향후 확장/변동 가능성이 큼 |
| 검색·정렬·참조의 핵심 축 | 부가 속성 수용 목적 |

#### 상태(status) vs 삭제 정책(deleted_at)

| 항목 | 상태 (status) | 삭제 정책 (deleted_at) |
|---|---|---|
| 의미 | 생명주기 단계 (draft/review/published/archived) | 데이터 처리 정책 (소프트 삭제) |
| 표현 방식 | `Document.status` enum 필드 | `Document.deleted_at` nullable 타임스탬프 |
| 열람 가능성 | archived는 명시적 필터로 조회 가능 | deleted_at 설정 시 모든 조회에서 기본 제외 |
| 복원 | archived → draft 경로로 복원 가능 | 복원은 별도 운영 정책으로 결정 |

#### DocumentType vs 템플릿/승인 정책

| 항목 | DocumentType | 템플릿/승인 정책 |
|---|---|---|
| 역할 | 문서 유형 분류 + 확장 규칙 연결 기준 | 재사용 양식 구조 (template_doc 타입의 Node 내용) |
| 범위 | 모든 Document에 적용 | template_doc 타입 문서에만 적용 |
| 승인 정책 | DocumentType의 `state_profile_ref`로 연결 참조만 | 실제 승인 워크플로는 Phase 5 이후 별도 설계 |

---

## 5. 최종 관계 구조 요약

### 5-1. 핵심 관계

```
DocumentType (1) ◄──── (N) Document (1) ────────────────► (N) Version (1) ────► (N) Node
                              │                                    │                   │
                     document_type (key)                  root_node_id           parent_id
                     current_version_id ─────────────────►│                   (같은 Version 내)
                     latest_version_id ───────────────────►│
                              │                                    │
                     Document-level metadata          metadata_snapshot (불변)
                                                              │
                                                       base_version_id ──► 이전 Version
```

### 5-2. current_version의 의미

- `Document.current_version_id`는 "현재 공식적으로 유효한 버전"을 가리키는 단일 출처(single source of truth)다.
- published 상태에서는 반드시 존재해야 한다. (NULL 불허)
- draft 상태에서는 NULL일 수 있다. (첫 버전 저장 전)
- 동시에 같은 Document에서 두 개의 current_version이 될 수 있는 Version은 없다.

### 5-3. latest_version의 의미와 current_version과의 관계

| 상황 | current_version_id | latest_version_id |
|---|---|---|
| 초기 draft 작성 중 | NULL | draft Version |
| 첫 발행 완료 | published Version | published Version (동일) |
| 발행 후 새 draft 편집 중 | 이전 published Version | 새 draft Version (다름) |
| 재발행 완료 | 새 published Version | 새 published Version (동일) |

> `latest ≠ current` 상태는 "개정 작업이 진행 중"임을 나타낸다.

### 5-4. root_node의 의미

- 모든 Version은 `node_type: document_root`인 루트 Node를 정확히 1개 가진다.
- 루트 Node의 `parent_id`는 NULL이다.
- `document_root`는 콘텐츠를 직접 담지 않으며, 하위 section들의 부모 역할만 한다.
- Version이 생성될 때 루트 Node도 반드시 함께 생성된다.

### 5-5. type-specific metadata의 위치

- DocumentType별 metadata는 Version-level `metadata_snapshot`의 `extended` 영역에 위치한다.
- 버전마다 달라지는 속성(시행일, 참석자 등)은 Version에 귀속된다.
- 버전과 무관한 문서 운영 속성(담당부서, 공개범위 등)은 Document-level metadata의 `common` 영역에 위치한다.

### 5-6. Version snapshot과 Document current 정보의 관계

- Document.title은 현재 공식 대표 제목이다. 목록/검색에 사용된다.
- Version.title_snapshot은 해당 버전 생성 시점의 제목 복사본이다. 과거 버전 재현에 사용된다.
- 두 값은 항상 같지 않다. 개정 이력 조회 시 Document.title이 아닌 Version.title_snapshot을 기준으로 재현해야 한다.

---

## 6. 최소 권장 코어 모델 기준안

### 6-1. Document 최소 기준안

Document는 문서의 지속적 정체성을 표현한다. 내용을 직접 보관하지 않으며, 버전 참조를 통해 내용에 접근한다.

**필수 필드 (MVP 코어)**

| 필드 | 설명 |
|---|---|
| `id` | 시스템 전체에서 고유한 식별자 (UUID) |
| `title` | 문서 대표 제목 (필수, 빈 문자열 불허) |
| `document_type` | DocumentType key 참조 (필수, 활성 타입만) |
| `status` | 문서 생명주기 상태 (draft/review/published/archived) |
| `current_version_id` | 현재 공식 유효 버전 참조 (nullable: 초기 생성 직후) |
| `created_by` | 문서 최초 생성자 식별자 |
| `created_at` | 문서 생성 일시 (불변) |
| `updated_at` | 마지막 변경 일시 |

**권장 필드 (MVP 포함 권장)**

| 필드 | 설명 |
|---|---|
| `latest_version_id` | 최신 생성 버전 참조 (draft 구분용, nullable) |
| `summary` | 문서 요약 설명 (목록 UI, 검색 결과용) |
| `metadata` | Document-level 확장 속성 JSON (common + extended) |
| `owner_id` | 문서 소유자 식별자 |
| `workspace_id` | 소속 워크스페이스/조직 |
| `archived_at` | 아카이브 처리 일시 (nullable) |
| `deleted_at` | 소프트 삭제 일시 (nullable) |

### 6-2. Version 최소 기준안

Version은 특정 시점의 문서 상태를 재현 가능하게 보존한다. 한 번 생성된 Version은 변경되지 않는다.

**필수 필드 (MVP 코어)**

| 필드 | 설명 |
|---|---|
| `id` | Version 고유 식별자 (UUID) |
| `document_id` | 소속 Document 참조 (외래 참조) |
| `version_no` | Document 내 순번 (1부터 단조 증가) |
| `title_snapshot` | 해당 버전 시점의 문서 제목 복사본 (불변) |
| `metadata_snapshot` | 해당 버전 시점의 메타데이터 복사본 JSON (불변) |
| `root_node_id` | Node 트리 루트 참조 |
| `created_by` | 이 버전 생성자 |
| `created_at` | 이 버전 생성 일시 (불변) |

**권장 필드 (MVP 포함 권장)**

| 필드 | 설명 |
|---|---|
| `summary_snapshot` | 이 버전 시점의 요약 복사본 |
| `change_summary` | 변경 설명 (사람이 작성) |
| `base_version_id` | 기반한 이전 버전 참조 (이력 체인, nullable) |
| `published_at` | 발행 일시 (nullable, NULL이면 draft 상태) |

### 6-3. Node 최소 기준안

Node는 Version에 종속된 문서 구조 단위다. parent_id + order 기반의 트리를 형성한다.

**필수 필드 (MVP 코어)**

| 필드 | 설명 |
|---|---|
| `id` | Node 고유 식별자 (UUID) |
| `version_id` | 소속 Version 참조 (생명 범위 결정) |
| `parent_id` | 부모 Node 참조 (NULL이면 루트 노드) |
| `order` | 형제 노드 간 순서 (정수, 낮을수록 앞) |
| `node_type` | 노드 유형 식별자 (문자열) |

**권장 필드 (MVP 포함 권장)**

| 필드 | 설명 |
|---|---|
| `title` | 노드 제목 (섹션명 등, nullable) |
| `content` | 노드 본문 내용 (nullable) |
| `content_format` | content 해석 방식 (plain/markdown/json) |
| `attributes` | node_type별 확장 속성 JSON (nullable) |
| `created_at` | 노드 생성 일시 |

**MVP 기본 node_type 목록**

| 분류 | node_type |
|---|---|
| 구조 노드 | `document_root`, `section`, `subsection` |
| 텍스트/본문 노드 | `paragraph`, `heading`, `quote`, `code_block` |
| 리스트 계열 | `list`, `list_item` |
| 표 계열 | `table`, `table_row`, `table_cell` |
| 미디어/참조 계열 | `image`, `reference` |
| 특수 계열 | `callout` |

### 6-4. DocumentType 최소 기준안

DocumentType은 문서를 분류하고 유형별 확장 규칙의 기준 역할을 한다. 단순 enum이 아닌 레코드 방식으로 관리한다.

**필수 필드 (MVP 코어)**

| 필드 | 설명 |
|---|---|
| `key` | 타입 고유 식별자 문자열 (예: "policy", "manual") |
| `display_name` | 사람이 읽을 수 있는 타입 이름 |
| `is_builtin` | 내장 타입 여부 |
| `is_active` | 활성 여부 (비활성 타입으로 신규 문서 생성 불가) |

**초기 기본 타입 집합 (8종)**

| key | display_name | group |
|---|---|---|
| `policy` | 규정·정책 문서 | regulatory |
| `manual` | 매뉴얼·가이드 | operational |
| `technical_doc` | 기술 문서 | technical |
| `report` | 보고서 | operational |
| `meeting_note` | 회의록 | operational |
| `knowledge_doc` | 지식 문서 | knowledge |
| `template_doc` | 템플릿 문서 | template |
| `generic_doc` | 일반 문서 (fallback) | general |

**확장 참조 필드 (후속 Phase 연결 구조)**

| 필드 | 설명 |
|---|---|
| `metadata_schema_ref` | 유형별 metadata 스키마 참조 |
| `node_rule_ref` | 유형별 Node 구성 규칙 참조 |
| `state_profile_ref` | 유형별 상태 프로필 참조 |

> MVP에서 확장 참조 필드는 NULL 허용. 구조만 열어두고 실제 연결은 후속 Phase에서 완성한다.

### 6-5. metadata 최소 기준안

metadata는 Document/Version/Node 각각에 부속되는 확장 구조다. 독립 엔티티가 아니며 부모 없이 존재하지 않는다.

**저장 구조 (공통 원칙)**

```
metadata (JSON)
  ├── common             : 공통 표준 metadata 영역 (플랫폼 표준 키 목록)
  │     ├── owner_department
  │     ├── visibility_scope
  │     ├── domain_category
  │     ├── language
  │     └── keywords
  └── extended           : 유형별 확장 metadata 영역 (namespace 적용)
        ├── policy.effective_date
        ├── meeting.attendees
        └── ...
```

**수준별 구분**

| 수준 | 목적 | 예시 |
|---|---|---|
| Document-level | 버전 무관 문서 운영 속성 | owner_department, visibility_scope |
| Version-level | 해당 버전 시점 종속 속성 | effective_date, meeting_datetime, reporting_period |
| Node-level attributes | node_type별 구조·표시 옵션 | colspan, language, level, ordered |
| Node-level metadata | 노드 수준 운영 메타 | 작성자 주석, 잠금 여부 |

**Naming 규칙**: 모든 metadata key는 `snake_case`. namespace는 `common.*` 또는 `{type_key}.*` 형태.

### 6-6. 상태 모델 최소 기준안

초기 상태 모델은 Document 중심으로 적용한다.

**4가지 상태**

| 상태 | 의미 | 편집 가능 | 외부 노출 |
|---|---|---|---|
| `draft` | 작성 중, 미확정 | O | X (기본) |
| `review` | 검토/승인 대기 | 제한적 | X (기본) |
| `published` | 공식 유효/공개 | X (직접 수정 불허) | O |
| `archived` | 비활성 보관 | X (복원 후 가능) | X (기본, 명시적 조회 가능) |

**기본 전이 규칙**

```
draft ──────────► review ──────────► published ──────────► archived
  ▲                  │                                         │
  └──────────────────┘                              (복원: 새 draft 생성)
     (반려)                                  ▲────────────────┘

draft ─────────────────────────────────────────────────────► archived
  (예외: 작성 취소)
```

**deleted는 상태가 아닌 별도 정책**: `Document.deleted_at` 필드로 표현. 모든 일반 조회에서 기본 제외.

---

## 7. 핵심 불변조건 요약

> 이 조건들은 플랫폼 코어 모델이 어떤 상태에서도 항상 만족해야 하는 조건이다. DB 설계, API 로직, 편집기 동작 모두에서 이 조건을 위반해서는 안 된다.

### 7-1. 참조 정합성 불변조건 (Hard Invariant)

| # | 조건 |
|---|---|
| RI-1 | 모든 Version은 정확히 하나의 Document에 속해야 한다 (고아 Version 불허) |
| RI-2 | 모든 Node는 정확히 하나의 Version에 속해야 한다 (고아 Node 불허) |
| RI-3 | `Document.current_version_id`는 해당 Document의 Version만 가리킬 수 있다 |
| RI-4 | `Document.latest_version_id`는 해당 Document의 Version만 가리킬 수 있다 |
| RI-5 | `Version.root_node_id`는 해당 Version의 Node만 가리킬 수 있다 |
| RI-6 | `Version.base_version_id`는 같은 Document 내 Version만 가리킬 수 있다 |
| RI-7 | `Node.parent_id`는 같은 Version 내 Node만 가리킬 수 있다 (다른 Version 참조 불가) |

### 7-2. 버전 정합성 불변조건 (Hard Invariant)

| # | 조건 |
|---|---|
| VC-1 | Version은 생성 후 Node 트리·metadata_snapshot·title_snapshot을 변경할 수 없다 (불변성) |
| VC-2 | `version_no`는 동일 Document 내에서 중복될 수 없다 |
| VC-3 | `version_no`는 Document 범위 내에서 단조 증가해야 한다 (이전 번호 재사용 불허) |
| VC-4 | Document.status = `published`이면 `current_version_id`는 NULL이 아니어야 한다 |
| VC-5 | `Document.current_version_id`가 가리키는 Version의 `published_at`은 NULL이 아니어야 한다 |

### 7-3. 트리 정합성 불변조건 (Hard Invariant)

| # | 조건 |
|---|---|
| TC-1 | 모든 Version에는 `document_root` 타입 루트 Node가 정확히 1개 존재해야 한다 |
| TC-2 | `document_root` Node의 `parent_id`는 NULL이어야 한다 |
| TC-3 | Node는 자기 자신을 `parent_id`로 가질 수 없다 (자기 참조 금지) |
| TC-4 | Node 트리는 순환(cycle)을 포함할 수 없다 |
| TC-5 | Version 간 Node 공유는 허용하지 않는다 |

### 7-4. 상태 정합성 불변조건 (Hard Invariant)

| # | 조건 |
|---|---|
| SC-1 | 상태 전이는 정의된 허용 경로만 따른다 (임의 전이 불허) |
| SC-2 | `Document.status = archived`이면 `archived_at`이 설정되어 있어야 한다 |
| SC-3 | `Document.deleted_at`이 설정된 문서는 모든 일반 조회에서 제외되어야 한다 |
| SC-4 | Document soft delete 시 하위 Version과 Node는 물리 삭제하지 않는다 (이력 보존) |

### 7-5. DocumentType 정합성 불변조건 (Hard Invariant)

| # | 조건 |
|---|---|
| DT-1 | 모든 Document는 반드시 하나의 DocumentType을 가져야 한다 (type 없는 Document 불허) |
| DT-2 | Document의 DocumentType은 활성(`is_active = true`) 타입이어야 한다 |
| DT-3 | type 미분류 문서는 `generic_doc` fallback type을 사용한다 |

### 7-6. metadata 최소 원칙 (Soft Rule → Future Constraint)

| # | 조건 |
|---|---|
| M-1 | metadata는 반드시 부모 엔티티에 귀속된다 (독립 존재 불가) |
| M-2 | 코어 필드와 동일한 의미의 값을 metadata에 중복 저장하지 않는다 |
| M-3 | metadata key는 snake_case 명명 규칙을 따른다 |
| M-4 | `common.*` namespace 키는 플랫폼 표준 키 목록을 따른다 |
| M-5 | `Version.metadata_snapshot`은 Version 생성 후 변경하지 않는다 |

---

## 8. MVP 범위와 후속 확장 범위

### 8-1. 비교표

| 항목 | MVP 범위 | 후속 확장 범위 |
|---|---|---|
| **엔티티** | Document, Version, Node, DocumentType (기본 구조) | Attachment, Reference, AuditEvent, Approval, Publication, Relation |
| **트리 구조** | parent_id + order 기반 adjacency list | depth/path 보조 필드, shared node (검토), closure table |
| **DocumentType** | 8종 기본 타입 + generic_doc fallback | 관리자 정의 타입 (Phase 3), 사용자 정의 타입 (Phase 5 이후) |
| **Node 구성 규칙** | 권장 수준 (위반 시 경고만) | 강제 검증 (Phase 3~4) |
| **상태 모델** | draft/review/published/archived (Document 중심) | deprecated/superseded/rejected/scheduled, Version 단위 승인 상태 (Phase 5) |
| **metadata 검증** | 서비스 레이어 권장 수준 | DocumentType별 필수 metadata 강제 (Phase 3), JSON Schema 자동 검증 (Phase 4~5) |
| **metadata schema 연결** | 구조(참조 필드)만 준비 | 실제 schema-linked 검증 연결 (Phase 3~4) |
| **Version 비교(diff)** | 구조는 가능하나 기능 미구현 | diff 비교 알고리즘 (후속 Phase) |
| **승인 워크플로** | review 상태로 흐름만 표현 | Approval 엔티티 + 세부 승인 절차 (Phase 5) |
| **version_no 방식** | 단순 정수 증가형 | major/minor 방식 (운영 정책 도입 시) |
| **타입별 상태 확장** | 공통 상태 모델 공유 | policy.effective, report.submitted 등 타입별 확장 (Phase 3~4) |

### 8-2. MVP 범위 정의

Phase 2~4에서 구현 기준으로 삼을 최소 범위:

- Document / Version / Node 기본 구조 및 필드
- parent_id + order 기반 트리
- 8종 기본 DocumentType 집합 (레코드 방식, 권장 수준 규칙)
- draft/review/published/archived 공통 상태 모델 (Document 중심)
- metadata common + extended 구조의 기본 틀 (검증은 서비스 레이어 권장 수준)
- current_version_id 중심 버전 참조 구조
- 20개 Hard Invariant 전면 적용

### 8-3. 후속 확장 범위 (지금은 개념만 열어두고 구현은 미룬다)

- DocumentType별 Node 구성 강제 검증
- JSON Schema 기반 metadata 자동 검증
- Approval / Publication 엔티티
- 문서 간 Reference / Relation 관계
- custom DocumentType 동적 생성
- shared node (저장 효율화)
- advanced diff model
- deprecated/superseded/rejected 상태 확장
- Version 단위 승인 상태 (review_status)
- 예약 배포 (scheduled 상태)

---

## 9. 상충 판단과 최종 선택

앞선 Task들에서 상충하거나 판단이 필요했던 쟁점들의 최종 선택을 기록한다.

| # | 쟁점 | 최종 선택 | 이유 |
|---|---|---|---|
| 1 | Document 생성 시 Version 없는 상태 허용 여부 | **허용** | 문서 초기 생성 직후 빈 Document 상태 필요. `current_version_id = NULL`로 표현 |
| 2 | `current_version_id` optional 여부 | **Optional (nullable)** | 초기 생성 후 첫 Version 저장 전 상태를 표현해야 함. published 상태에서만 Non-null 강제 |
| 3 | `title`을 Document와 Version 모두에 둘지 | **둘 다 보유** | Document.title = 현재 대표 제목(목록/검색용). Version.title_snapshot = 시점 이력 재현용. 목적이 다름 |
| 4 | Version을 완전 불변으로 볼지 | **완전 불변** | 재현 가능성과 감사/법적 신뢰성의 핵심 요건. 수정 = 새 Version 생성 |
| 5 | Node `depth`/`path`를 코어에 포함할지 | **MVP 제외, 후속 추가** | parent_id + order로 충분. 성능 이슈 발생 시 보조 필드 추가 |
| 6 | DocumentType을 enum 수준으로 시작할지 | **문자열 key 기반 레코드** | enum은 코드 변경 없이 타입 추가 불가. 레코드 방식으로 MVP는 단순하되 확장 가능 |
| 7 | metadata schema 연결을 초기부터 강제할지 | **구조만 준비, 실제 강제는 후속** | 초기 과설계 방지. `metadata_schema_ref` 참조 구조를 먼저 설계에 포함 |
| 8 | 상태를 Document 중심으로 둘지 | **Document 중심 (MVP)** | "이 문서가 현재 어떤 상태인가"는 Document 수준 질문. Version 단위 상태는 후속 확장 |
| 9 | `deleted`를 상태로 둘지 정책으로 둘지 | **정책(deleted_at 필드)** | 삭제는 생명주기 단계가 아닌 데이터 처리 정책. 상태와 혼용 시 조회 로직 복잡화 |

---

## 10. 후속 Phase 입력 기준

### 10-1. Phase 2 (저장 구조/DB 설계) 입력값

**테이블/컬렉션 분해 기준**:
- 코어 엔티티 3개 (documents, versions, nodes)는 독립 테이블로 분해
- DocumentType은 별도 테이블 (document_types) 또는 정적 설정 파일로 관리
- metadata는 JSON 컬럼 인라인 저장 (common/extended 분리 또는 단일 JSON)

**참조 관계 설계 기준**:
- `versions.document_id` → `documents.id` (FK, NOT NULL)
- `nodes.version_id` → `versions.id` (FK, NOT NULL)
- `nodes.parent_id` → `nodes.id` (FK, nullable, 같은 version_id 조건 필요)
- `documents.current_version_id` → `versions.id` (FK, nullable)
- `documents.latest_version_id` → `versions.id` (FK, nullable)
- `versions.root_node_id` → `nodes.id` (FK, nullable)
- `(versions.document_id, versions.version_no)` 복합 유니크 제약 필요

**불변조건 반영 포인트**:
- `documents.status` 컬럼에 정의된 값 목록 제약
- `documents.deleted_at` 기반 soft delete 필터 인덱스
- `documents.status + documents.deleted_at` 조합 인덱스 (목록 조회 성능)
- Node 트리 재귀 조회 성능을 위한 인덱스 전략 (`version_id + parent_id`)

### 10-2. Phase 3 (API 설계) 입력값

**CRUD 단위 기준**:
- Document CRUD: 문서 정체성 관리. 내용은 Version/Node 경로로 접근
- Version 생성: 새 버전 생성 시 document_root Node를 반드시 함께 생성
- Node CRUD: 특정 Version의 Node 트리 조작. parent_id 검증 필수

**버전 생성 규칙**:
- published 상태 문서의 수정은 새 draft Version 생성 흐름으로 유도
- draft Version 저장 시 `Document.latest_version_id` 갱신
- `review → published` 전이 시 `current_version_id` 갱신 + `Version.published_at` 설정을 원자적으로 처리

**상태 전이 API 제약 포인트**:
- 상태 직접 수정(PATCH status) 방식보다 전이 액션 기반(`POST /documents/{id}/publish`) 권장
- 비허용 전이 요청 시 명확한 오류 응답
- `deleted_at` 설정은 별도 삭제 엔드포인트

**metadata 입력/검증 기본 원칙**:
- 문서 생성/수정 API에서 metadata를 common + extended 구조로 수신
- DocumentType key 검증: 활성 DocumentType만 허용
- metadata 검증 책임은 서비스 레이어 (DB 레벨 강제 아님)
- Phase 3에서 DocumentType별 권장 필드 목록 기반 검증 도입 시작

### 10-3. Phase 4 (UI/편집/렌더링 설계) 입력값

**Document/Version/Node 분리 인식**:
- 문서 목록은 Document 기준으로 표시 (title, status, updated_at)
- 문서 열람 화면은 current_version_id → Version → Node 트리 경로로 렌더링
- 버전 이력은 Document에 속한 Version 목록으로 표시

**Node 트리 편집 기준**:
- Node 편집은 현재 Version을 직접 수정하지 않음. 새 Version 생성 흐름
- `document_root` Node는 에디터에서 삭제 불가
- Node 이동(parent_id 변경) 시 같은 Version 내 Node만 참조 가능한지 검증 필요
- 형제 Node 재정렬 시 `order` 값 일관성 유지

**상태/타입/metadata UI 노출 기준**:
- `draft` 상태: 편집 버튼 활성화
- `review` 상태: 편집 제한 안내, "검토 반려 후 수정" 경로 안내
- `published` 상태: 편집 버튼 → "새 버전으로 개정" 흐름 시작
- `archived` 상태: 읽기 전용, "복원" 버튼 제공
- 문서 생성 시 DocumentType 선택 필수, 선택된 타입에 따라 metadata 입력 폼 동적 변화
- `common` metadata는 공통 폼으로, `extended` metadata는 타입별 동적 폼으로 렌더링

---

## 11. 최종 결론

### 11-1. Phase 1에서 확정한 것

Phase 1은 Mimir 범용 문서 플랫폼의 도메인 모델 기반을 확립했다.

**확정된 핵심 결정사항**:
1. 코어 엔티티는 Document·Version·Node 3개다.
2. Document는 정체성과 포인터만 보유하며, 내용은 Version → Node 경로로 접근한다.
3. Version은 한 번 생성되면 불변이다. 수정 = 새 Version 생성.
4. Node는 Version에 종속되며, 모든 본문 구조가 Node 트리로 수렴한다.
5. DocumentType은 문자열 key 기반 레코드로 관리하며, 확장 규칙 연결 구조를 처음부터 설계에 포함한다.
6. metadata는 common + extended 2영역 구조를 채택하며, 검증은 서비스 레이어가 담당한다.
7. 상태 모델은 Document 중심 4상태(draft/review/published/archived)로 시작한다.
8. deleted는 상태가 아닌 soft delete 정책(deleted_at)으로 분리한다.
9. 20개의 Hard Invariant가 모든 후속 구현의 불변 기준이 된다.

### 11-2. 후속 Phase에 넘기는 것

- **구현**: DB 스키마, API 명세, UI 컴포넌트
- **강제 검증**: DocumentType별 Node 구성 강제, metadata 필수 필드 강제
- **확장 엔티티**: Attachment, Reference, AuditEvent, Approval
- **상태 확장**: deprecated, superseded, type별 상태 프로필

### 11-3. 이 문서의 사용 방법

> 이 문서는 Phase 2(DB), Phase 3(API), Phase 4(UI) 설계의 입력 기준으로 사용한다.
>
> - 새로운 필드·엔티티·상태를 추가하려 할 때: 이 문서의 책임 분리 기준안과 MVP/확장 범위 분리표를 먼저 확인한다.
> - 구현 로직에서 경계 판단이 필요할 때: 섹션 4(최종 책임 분리 기준안)를 참조한다.
> - DB 설계 시 참조 무결성 규칙이 필요할 때: 섹션 7(핵심 불변조건 요약)을 참조한다.
> - Phase 2~4에서 "이걸 지금 해야 하나?"를 판단할 때: 섹션 8(MVP 범위와 후속 확장 범위)을 참조한다.
