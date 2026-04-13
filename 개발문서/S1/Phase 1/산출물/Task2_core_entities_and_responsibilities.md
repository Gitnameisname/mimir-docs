# Phase 1 - Task 1-2. 핵심 엔티티 식별 및 책임 정의

## 1. 작업 목적

Task 1-1에서 정리한 범용 문서 도메인 요구사항을 바탕으로,
플랫폼 코어를 구성할 엔티티들을 식별하고 각 엔티티의 책임과 경계를 확정한다.

이 문서의 목적은 이후 Document / Version / Node / DocumentType / Metadata / 상태 모델 상세 설계가
흔들리지 않도록 **각 엔티티가 무엇을 책임하고, 무엇은 책임하지 않는지**를 먼저 고정하는 것이다.

---

## 2. 검토 대상 엔티티 후보

### 2-1. 코어 후보

| 후보 엔티티 | 검토 필요 이유 |
|---|---|
| `Document` | 문서 정체성의 최상위 단위 |
| `Version` | 특정 시점 문서 내용의 보존 단위 |
| `Node` | 문서 내부 구조의 최소 표현 단위 |
| `DocumentType` | 유형 분류 및 확장 규칙 정의 기준 |
| `Metadata` | 공통·유형별 확장 속성 수용 구조 |
| `DocumentStatus` | 문서 생명주기 상태 표현 |

### 2-2. 부가 후보

| 후보 엔티티 | 검토 필요 이유 |
|---|---|
| `Attachment` | 문서에 첨부되는 파일/바이너리 |
| `Reference` | 문서 간 참조 관계 |
| `Relation` | 문서 간 의미 있는 연관 관계 |
| `Tag` | 분류/검색용 레이블 |
| `AuditEvent` | 변경 이력 추적 |
| `Template` | 재사용 양식 정의 |
| `Approval` | 승인 워크플로 단위 |
| `Publication` | 배포/공표 이벤트 |

---

## 3. 코어 엔티티 선정 결과

### 3-1. 선정 분류표

| 엔티티 | 분류 | 근거 |
|---|---|---|
| `Document` | **코어 엔티티** | 문서의 정체성(identity) 자체. 모든 참조의 기준점 |
| `Version` | **코어 엔티티** | 문서 내용의 불변 스냅샷. 버전 관리의 핵심 단위 |
| `Node` | **코어 엔티티** | 문서 본문 구조의 최소 단위. 트리 기반 표현의 기본 요소 |
| `DocumentType` | **보조 개념 (독립 관리 대상)** | Document의 속성이지만, 확장 규칙 정의를 위해 독립 관리 |
| `Metadata` | **확장 구조 (독립 엔티티 아님)** | Document·Version·Node에 부속되는 확장 속성 필드 묶음 |
| `DocumentStatus` | **상태 모델 (enum + 전이 규칙)** | 독립 엔티티가 아닌, Document에 속하는 상태 속성 |
| `Attachment` | **부가 엔티티 (Phase 2 이후)** | 필요하나 코어 모델 설계에 영향 없음 |
| `Reference` | **부가 엔티티 (Phase 2 이후)** | policy 유형 등에서 필요하나 코어와 독립 가능 |
| `Relation` | **후속 확장** | 문서 간 의미 관계. 검색/지식 그래프 단계에서 다룸 |
| `Tag` | **보조 개념** | Metadata 확장 구조 내에서 처리 가능 |
| `AuditEvent` | **부가 엔티티 (Phase 2 이후)** | 변경 추적 필요하나 코어 모델과 독립 |
| `Template` | **후속 확장** | `template_doc` 유형 특성. 코어 모델과 독립 |
| `Approval` | **후속 확장** | 승인 워크플로는 Phase 5 이후 |
| `Publication` | **후속 확장** | 배포/공표 이벤트는 Phase 5 이후 |

### 3-2. 요약

- **Phase 1 코어 엔티티**: `Document`, `Version`, `Node`
- **Phase 1 보조 개념**: `DocumentType`(독립 관리 대상), `Metadata`(확장 구조), `DocumentStatus`(상태 모델)
- **Phase 2 이후 부가 엔티티**: `Attachment`, `Reference`, `AuditEvent`
- **Phase 5 이후 확장**: `Relation`, `Template`, `Approval`, `Publication`

---

## 4. 엔티티별 정의와 책임

### 4-1. 책임 분리표

| 엔티티 | 한 줄 정의 | 핵심 책임 | 책임하지 않는 것 | 주요 연관 | 설계 주의사항 |
|---|---|---|---|---|---|
| `Document` | 문서의 정체성과 대표 정보를 관리하는 최상위 단위 | 문서 고유 식별, 유형 분류, 현재 버전 참조, 상태 보유, 소속 정보 | 문서 내용 직접 보관, 버전별 본문 상세, 구조 표현 | `Version`, `DocumentType` | 내용은 Version 위임. Document는 껍데기+포인터 |
| `Version` | 특정 시점 문서 내용의 불변 스냅샷 | 노드 트리 루트 참조, 버전 번호, 변경 설명, 생성자·생성일시, 메타데이터 스냅샷 보관 | 문서 정체성 관리, 상태 결정, 유형 분류 | `Document`, `Node` | 한 번 생성된 Version은 수정 불가(불변성 원칙) |
| `Node` | 문서 본문의 계층 구조를 표현하는 최소 단위 | 노드 유형 식별, 부모-자식 관계, 형제 순서, 제목·본문 내용 보관, 노드 수준 메타데이터 | 문서 정체성, 버전 관리, 유형 규칙 적용 | `Version` | 노드는 Version에 종속. Version 없이 독립 존재 불가 |
| `DocumentType` | 문서 유형을 식별하고 유형별 확장 규칙의 기준을 정의하는 개념 | 유형 식별값, 유형별 허용 노드 규칙, 유형별 메타데이터 스키마 연결 | 개별 문서의 상태 관리, 버전 내용 저장, 구조 표현 | `Document`, `Metadata` | 유형은 고정 enum이 아닌, 독립 관리 가능 레코드로 설계 |
| `Metadata` | Document·Version·Node에 부속되는 공통 및 유형별 확장 속성 구조 | 공통 메타(작성자, 날짜 등) 보관, 유형별 확장 필드 수용, 검색·필터 기준 제공 | 독립 생존(반드시 부모 엔티티에 귀속), 상태·버전 관리 | `Document`, `Version`, `Node` | 독립 엔티티가 아닌 확장 구조. JSON 또는 스키마 연결 방식 |
| `DocumentStatus` | 문서 생명주기를 나타내는 상태 속성 | draft·review·published·archived 상태 표현, 상태 전이 규칙 적용 기준 | 버전 내용 저장, 구조 관리, 승인 워크플로 상세 | `Document` | 독립 엔티티가 아닌 Document의 상태 속성(enum + 전이 규칙) |

### 4-2. 각 엔티티 상세 정의

#### Document

- **Document는 문서의 정체성(identity)을 대표한다.**
- 문서가 어떤 유형인지, 현재 어떤 버전이 유효한지, 어떤 상태인지를 안다.
- 문서의 실제 내용(본문, 구조)은 Version을 통해 접근한다.
- Document를 삭제하거나 아카이브하면 연결된 모든 Version이 논리적으로 비활성화된다.
- Document는 여러 Version을 가질 수 있지만, 언제나 하나의 `current_version`을 명확히 가리킨다.

#### Version

- **Version은 특정 시점 문서 내용의 불변 스냅샷이다.**
- 버전이 생성되면 그 내용은 이후 절대 변경되지 않는다.
- 문서를 수정한다는 것은 기존 Version을 수정하는 것이 아니라, 새 Version을 생성하는 것이다.
- Version은 Document의 `current_version`으로 지정되거나, 과거 이력으로 보존된다.
- Version은 자신에게 속한 Node 트리의 루트를 참조한다.

#### Node

- **Node는 문서 본문의 계층 구조를 표현하는 최소 단위다.**
- 각 Node는 `node_type`(section, paragraph, list, table 등)을 가지며, 부모-자식 관계로 트리를 형성한다.
- Node를 단순 텍스트 조각으로 보면 안 된다. Node는 의미 있는 구조 단위다.
- Node는 Version에 종속된다. Version 없이 Node는 독립 존재할 수 없다.
- 문서의 모든 구조 표현(섹션, 본문, 표, 리스트 등)은 Node 트리로 수렴한다.

#### DocumentType

- **DocumentType은 문서를 분류하고, 유형별 확장 규칙의 기준 역할을 한다.**
- 단순 enum 값으로만 쓰이면 확장성이 떨어진다. 독립 관리 레코드로 설계하면 유형별 메타데이터 스키마·노드 규칙을 연결할 수 있다.
- DocumentType은 Document에 1:1로 귀속되나, DocumentType 자체는 여러 Document가 공유한다.

#### Metadata

- **Metadata는 Document·Version·Node에 부속되는 확장 속성 구조다.**
- 독립 엔티티가 아니다. 부모 엔티티 없이 독립 존재하지 않는다.
- 공통 메타(생성일, 작성자 등)는 부모 엔티티에 직접 필드로 보관한다.
- 유형별 확장 메타(시행일, 참석자 목록 등)는 JSON 또는 스키마 연결 구조로 수용한다.

#### DocumentStatus

- **DocumentStatus는 문서 생명주기를 나타내는 상태 속성이다.**
- 독립 엔티티가 아닌 Document의 속성(enum)이며, 상태 전이 규칙을 동반한다.
- MVP 기준 상태: `draft` → `review` → `published` → `archived`
- 승인 워크플로가 도입되면 Version 수준 상태를 별도로 확장한다.

---

## 5. 엔티티 간 관계와 경계

### 5-1. 개념 관계 정리

```
Document (1) ──────────── (N) Version
    │                          │
    │  current_version_id       │
    └──────────────────────────►│
                                │
                         Version (1) ──── (N) Node
                                              │
                                          Node (1) ──── (N) Node (자식)

Document (N) ──── (1) DocumentType

Metadata ──── 부속 ──── Document | Version | Node (각각에 귀속)
DocumentStatus ──── 속성 ──── Document
```

### 5-2. 핵심 경계 질문 답변

**Q. Document와 Version은 어떻게 다른가?**

> Document는 "이 문서가 존재한다"는 사실과 정체성(유형, 상태, 소속)을 보유한다.
> Version은 "이 시점에 이 문서는 이런 내용이었다"는 스냅샷을 보유한다.
> 내용을 알고 싶으면 Document가 가리키는 `current_version`을 통해 접근한다.

**Q. Version과 Node는 어떻게 연결되는가?**

> Version은 하나의 루트 Node를 참조하고, 해당 루트 Node를 시작으로 전체 트리가 구성된다.
> Node는 Version에 종속되며, 특정 Version의 Node는 다른 Version과 공유되지 않는다.

**Q. DocumentType은 Document의 속성인가, 독립 규칙 단위인가?**

> 둘 다다. Document 입장에서는 "나의 유형 분류값"이지만,
> 플랫폼 입장에서는 "이 유형에 어떤 메타데이터 스키마와 노드 규칙이 연결되는가"를 정의하는 독립 관리 대상이다.
> 따라서 DocumentType은 Document에 외래 참조로 귀속되되, 독립적으로 관리된다.

**Q. Metadata는 어디에 귀속되는가?**

> Metadata는 독립 엔티티가 아니다. Document·Version·Node 각각에 확장 속성 구조로 귀속된다.
> 공통 메타는 각 엔티티의 직접 필드로, 유형별 확장 메타는 JSON 또는 스키마 연결 구조로 보관한다.

**Q. 상태는 Document 수준인가, Version 수준인가?**

> MVP 단계에서는 Document 수준이다.
> "이 문서는 현재 published 상태다"처럼 문서 전체의 대표 상태를 Document가 보유한다.
> 이후 승인 워크플로 도입 시, Version 수준의 검토·승인 상태를 별도 확장한다.

---

## 6. 부가 엔티티 후보 검토

| 엔티티 | 왜 필요한가 | 지금 코어에 포함해야 하는가 | 판단 |
|---|---|---|---|
| `Attachment` | 문서에 첨부 파일(PDF, 이미지 등)이 필요함 | 아니오 — 코어 모델 구조에 영향 없음 | Phase 2 이후 부가 엔티티로 추가 |
| `Reference` | policy 등에서 조항 간 참조, 문서 간 인용 필요 | 아니오 — Node의 `reference` 타입으로 우선 흡수 가능 | Phase 2 이후 관계 모델로 분리 |
| `Relation` | 문서 간 연관(후속 문서, 대체 문서, 관련 문서) | 아니오 — 검색·지식그래프 단계에서 다룸 | Phase 5 이후 |
| `Tag` | 분류·검색 레이블 | 아니오 — Document Metadata 확장 구조 내에서 처리 가능 | 보조 개념으로 흡수 |
| `AuditEvent` | 누가 언제 무엇을 변경했는지 추적 | 아니오 — 중요하지만 코어 모델 구조에 영향 없음 | Phase 2 이후 부가 엔티티 |
| `Template` | `template_doc` 유형의 양식 재사용 | 아니오 — `template_doc`을 DocumentType으로 분류하고 노드 구조로 표현 | Phase 3~4 이후 |
| `Approval` | 문서 승인 프로세스 | 아니오 — 승인 워크플로는 Phase 5 이후 | Phase 5 이후 |
| `Publication` | 배포·공표 이벤트 기록 | 아니오 — 배포는 상태 전이의 결과이므로 지금 불필요 | Phase 5 이후 |

---

## 7. 최종 결론 및 다음 Task 입력값

### 7-1. Phase 1 코어 엔티티 확정

| 엔티티 | 핵심 책임 요약 |
|---|---|
| `Document` | 문서의 정체성·유형·상태·현재 버전 참조를 보유하는 최상위 단위 |
| `Version` | 특정 시점 문서 내용의 불변 스냅샷. 노드 트리의 루트 참조 보유 |
| `Node` | 문서 본문의 계층 구조를 표현하는 최소 단위. 부모-자식 트리 형성 |

### 7-2. 보조 개념 처리 결정

| 개념 | 처리 방식 |
|---|---|
| `DocumentType` | Document의 외래 참조 + 독립 관리 레코드 (유형별 규칙 연결 기준) |
| `Metadata` | 독립 엔티티 아님. Document·Version·Node 각각의 확장 속성 구조 |
| `DocumentStatus` | 독립 엔티티 아님. Document의 상태 속성(enum + 전이 규칙) |
| `Tag` | Metadata 확장 구조 내 처리 |

### 7-3. 후속 Task 입력값

| Task | 상세 설계 대상 | 핵심 설계 포인트 |
|---|---|---|
| **Task 1-3** | Document 구조 상세화 | 필수 필드, Version 참조 방식, 식별자 정책, 소프트 삭제 전략 |
| **Task 1-4** | Version 구조 상세화 | 불변성 보장 방식, 스냅샷 vs diff, draft/published 버전 구분 |
| **Task 1-5** | Node 트리 구조 상세화 | node_type 분류, 트리 순서 관리, 콘텐츠 노드 vs 구조 노드 |
| **Task 1-6** | DocumentType 시스템 설계 | 확장 규칙 연결 방식, 커스텀 타입 전략 |
| **Task 1-7** | Metadata 확장 구조 설계 | 공통 필드 최소셋, 유형별 스키마 연결 방식, 검증 전략 |
| **Task 1-8** | 상태 모델 기초 설계 | 상태 전이 규칙, Document vs Version 상태 적용 범위 |
