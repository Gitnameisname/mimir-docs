# Phase 1 - Task 1-7. metadata 확장 구조 설계

## 1. 작업 목적

Task 1-1~1-6에서 확정한 요구사항, 엔티티 책임, Document/Version/Node/DocumentType 구조를 바탕으로,
**공통 코어를 유지하면서도 문서 유형별 차이를 수용할 수 있는 metadata 확장 구조의 원칙과 설계**를 확정한다.

metadata를 단순 JSON 덩어리로 방치하지 않고, 공통 필드와 확장 필드의 경계를 정리하며,
검색/필터/검증/유형별 스키마 확장의 기반이 되는 구조 원칙을 정립한다.

---

## 2. metadata의 역할 정의

### 2-1. metadata가 대표하는 것

- metadata는 **코어 모델이 직접 고정하지 않는 부가 속성을 수용**한다. 코어를 단순하게 유지하면서 유연성을 제공하는 확장 지점이다.
- metadata는 **문서 유형별 차이를 표현하는 확장 지점**이다. DocumentType에 따라 다른 필드를 수용한다.
- metadata는 **검색/필터/검증/표시 로직의 입력**이 될 수 있다. 단순 보관용이 아니다.
- metadata는 **Document / Version / Node 각각에 서로 다른 의미로 존재**한다. 수준에 따라 목적이 다르다.
- metadata는 **코어 구조를 대체하지 않는다.** 코어가 담아야 할 속성을 metadata로 밀어넣지 않는다.

### 2-2. 책임 분리표

| 항목 | metadata 책임 | 귀속 위치 |
|---|---|---|
| 유형별 확장 속성 수용 | O | Document/Version/Node metadata |
| 공통 부가 속성 보관 | O | 공통 metadata 영역 |
| 검색/필터 보조 기준 제공 | O (일부 필드) | 공통 metadata |
| 문서 정체성 보유 | X | Document 코어 필드 |
| 버전 생명주기 관리 | X | Version 코어 필드 |
| 노드 트리 구조 표현 | X | Node 코어 필드 |
| 상태 전이 | X | Document.status |
| DocumentType 연결 | X | Document.document_type |

---

## 3. 코어 필드 vs metadata 구분 기준

### 3-1. 판단 기준표

| 기준 | 코어 필드 | metadata 확장 필드 |
|---|---|---|
| 거의 모든 문서에 공통으로 필요한가 | O | X |
| 정체성/관계/상태/버전 무결성에 직접 영향 주는가 | O | X |
| 검색/정렬/참조의 핵심 축인가 | O | 일부 공통 metadata |
| 특정 문서 유형에만 필요한가 | X | O |
| 향후 확장/변동 가능성이 큰가 | X | O |
| 코드/쿼리에서 직접 참조되는가 | O | 가능하면 코어, 아니면 metadata |

### 3-2. 속성별 귀속 판단 예시

| 속성 예시 | 귀속 | 이유 |
|---|---|---|
| `title` | 코어 필드 | 모든 문서에 공통, 검색 핵심 축 |
| `status` | 코어 필드 | 생명주기 무결성에 직접 영향 |
| `document_type` | 코어 필드 | 타입별 처리의 기준, 정체성 일부 |
| `created_at` | 코어 필드 | 감사 기준, 정렬 핵심 축 |
| `current_version_id` | 코어 필드 | 버전 무결성 참조 |
| `effective_date` | Version-level metadata | 유형별 차이, 시점 종속 |
| `meeting_attendees` | Version-level metadata | meeting_note 전용, 시점 종속 |
| `owner_department` | Document-level metadata | 운영 속성, 유형 무관 |
| `table cell colspan` | Node-level attributes | 특정 node_type 전용 구조 옵션 |
| `keywords` | Document-level metadata | 검색 보조, 코어 불필요 |
| `visibility_scope` | Document-level metadata | 운영 정책 속성 |
| `reporting_period` | Version-level metadata | report 유형 전용, 시점 종속 |

### 3-3. metadata 남용 방지 원칙

> - **검색/필터의 핵심 축이 될 속성은 코어 필드 또는 공통 metadata 표준 필드로 승격을 검토한다.**
> - **코어 필드로 가야 할 속성을 단순히 "나중에 생각하자"며 metadata로 미루지 않는다.**
> - **세 가지 이상의 DocumentType에서 공통으로 필요한 속성이 metadata에 등장하면 공통 metadata 표준 필드로 격상을 검토한다.**

---

## 4. metadata 적용 범위 구분

### 4-1. Document-level metadata

**목적**: 문서 자체의 소속·운영 정보. 버전과 무관하게 문서 전체 생명주기에 걸쳐 지속된다.

| 적합한 정보 유형 | 예시 |
|---|---|
| 소속 조직/부서 | `owner_department`, `division` |
| 문서 분류 | `domain_category`, `classification_code` |
| 공개 범위 | `visibility_scope` (public/internal/restricted) |
| 언어 | `language` |
| 검색 키워드 | `keywords` |
| 커스텀 분류값 | 기관별 특수 분류 필드 |

**다른 수준과의 구분**: 이 속성들은 특정 버전의 내용이 아니라, 문서 전체를 설명하는 운영 맥락이다.
버전이 바뀌어도 동일하게 유지된다.

### 4-2. Version-level metadata

**목적**: 특정 버전 시점에 종속된 속성. 버전마다 달라질 수 있으며, 과거 버전 재현에 필요하다.

| 적합한 정보 유형 | 예시 |
|---|---|
| 유형별 시점 속성 | `effective_date`, `revision_date` |
| 버전별 변경 맥락 | `revision_reason`, `applicable_scope` |
| 유형별 핵심 속성 | `reporting_period`, `approved_by`, `meeting_datetime` |
| 유형별 참여자 | `attendees`, `reviewers` |

**다른 수준과의 구분**: 이 속성들은 "이 버전에서는 이랬다"는 시점 정보다.
Document-level과 달리 버전마다 다를 수 있으며, metadata_snapshot으로 보존된다.

### 4-3. Node-level metadata / attributes

**목적**: 특정 본문 블록에 종속된 구조·표시 옵션 및 운영 정보.

| 분류 | 적합한 정보 유형 | 예시 |
|---|---|---|
| `attributes` | node_type별 구조·표시 옵션 (재현 필요) | colspan, alt text, language, level, ordered |
| `metadata` | 노드 수준 운영 정보 (재현 우선순위 낮음) | 작성자 주석, 잠금 여부, 검토 메모 |

**다른 수준과의 구분**: 문서 전체가 아닌 특정 블록에만 귀속되는 속성이다.
Document나 Version metadata에 두면 scope가 맞지 않는다.

---

## 5. 공통 metadata와 유형별 확장 metadata

### 5-1. 공통 metadata (모든 타입에서 재사용 가능)

문서 유형과 무관하게 다수 유형에서 반복적으로 등장하는 속성들이다.

| 필드명 | 설명 | 적용 수준 |
|---|---|---|
| `owner_department` | 문서 담당 부서 | Document |
| `visibility_scope` | 공개 범위 (public/internal/restricted) | Document |
| `domain_category` | 업무 도메인 분류 | Document |
| `language` | 문서 작성 언어 | Document |
| `keywords` | 검색 키워드 목록 | Document |
| `applicable_scope` | 이 버전의 적용 범위 | Version |
| `revision_reason` | 이 버전의 변경 사유 | Version |

### 5-2. 유형별 확장 metadata

특정 DocumentType에 주로 의존하는 속성들이다.

| DocumentType | 확장 필드명 | 설명 | 적용 수준 |
|---|---|---|---|
| `policy` | `effective_date` | 시행일 | Version |
| | `revision_date` | 개정일 | Version |
| | `governing_body` | 관할부서/기관 | Version |
| | `legal_basis` | 법적 근거 | Version |
| `report` | `reporting_period_start` | 보고 대상 기간 시작 | Version |
| | `reporting_period_end` | 보고 대상 기간 종료 | Version |
| | `approved_by` | 승인자 | Version |
| | `report_target` | 보고 대상자 | Version |
| `meeting_note` | `meeting_datetime` | 회의 일시 | Version |
| | `meeting_location` | 회의 장소 | Version |
| | `attendees` | 참석자 목록 | Version |
| | `agenda_items` | 안건 목록 | Version |
| `manual` | `target_system` | 대상 시스템/제품 | Version |
| | `target_version` | 적용 버전 | Version |
| | `target_audience` | 대상 독자 | Version |
| `technical_doc` | `related_system` | 연관 시스템 | Version |
| | `api_version` | API 버전 | Version |
| `template_doc` | `template_purpose` | 양식 용도 | Document |
| | `required_fields` | 필수 입력 항목 목록 | Document |

### 5-3. 공통 metadata 표준화 수준

**결정: 공통 metadata 필드 목록을 정의하되, 강제 적용은 서비스 레이어에서 결정한다.**

- 공통 metadata 필드는 플랫폼 수준에서 권장 키 이름을 정의한다.
- 유형별 확장 필드는 DocumentType의 `metadata_schema_ref`를 통해 참조한다.
- 키 충돌 방지를 위해 namespace 규칙을 적용한다. (Section 9 참고)

---

## 6. 저장 구조 방향 검토

### 6-1. 관점 비교

| 관점 | 설명 | 장점 | 한계 |
|---|---|---|---|
| **A. 완전 자유 JSON** | 아무 키-값이나 허용 | 유연, 빠른 시작 | 구조 혼란, 검증 불가, 검색 어려움 |
| **B. 공통 구조 + 자유 확장** | 공통 영역 + 타입별 확장 영역 분리 | 균형적, 관리 가능 | 구조 설계 필요 |
| **C. schema-linked metadata** | DocumentType별 스키마 참조 및 검증 | 검증 가능, UI 연계 쉬움 | 초기 복잡도 증가 |

### 6-2. 권장 방향

**결정: 관점 B(공통 구조 + 자유 확장)를 기준으로 설계하고, 관점 C(schema 연결)를 후속 Phase에서 도입한다.**

```
metadata 구조 (개념)
  ├── common              : 공통 metadata 영역 (표준 키 목록 기반)
  │     ├── owner_department
  │     ├── visibility_scope
  │     ├── domain_category
  │     └── keywords
  └── extended            : 유형별 확장 metadata 영역 (자유 구조, 스키마 참조 가능)
        ├── effective_date  (policy 전용)
        ├── meeting_datetime (meeting_note 전용)
        └── ...
```

- MVP에서는 `common`과 `extended` 두 영역으로 구분하여 JSON에 저장한다.
- `common` 영역의 키 목록은 플랫폼이 표준으로 관리한다.
- `extended` 영역은 DocumentType의 `metadata_schema_ref`를 참조하여 검증 가능 구조로 확장한다.

---

## 7. 검색/필터 관점 설계 기준

### 7-1. 검색/필터 활용 가능성이 높은 metadata 필드

| 필드 | 활용 시나리오 | 권장 위치 |
|---|---|---|
| `effective_date` | "2024년 이후 시행된 규정 조회" | Version-level metadata (색인 필요) |
| `owner_department` | "특정 부서의 문서 조회" | Document-level 공통 metadata |
| `visibility_scope` | "공개 문서만 조회" | Document-level 공통 metadata |
| `reporting_period` | "특정 분기 보고서 조회" | Version-level metadata |
| `keywords` | "키워드 검색" | Document-level 공통 metadata |
| `domain_category` | "특정 도메인 문서 필터" | Document-level 공통 metadata |
| `meeting_datetime` | "특정 날짜 회의록 조회" | Version-level metadata |
| `language` | "언어별 문서 필터" | Document-level 공통 metadata |

### 7-2. 검색 관점 설계 원칙

- **검색/필터에서 자주 쓰일 공통 metadata 필드는 저장 시 별도 색인 가능한 구조로 설계한다.**
- **완전 자유 extended 영역의 필드는 색인이 어렵다.** 검색에 자주 쓰이는 유형별 필드가 확인되면 공통 metadata로 승격을 검토한다.
- 전문 검색(full-text search)은 `content` 필드 중심으로 설계하고, metadata는 구조화 필터 축으로 활용한다.

---

## 8. 검증 전략 검토

### 8-1. 검증 수준별 단계

| 단계 | 검증 수준 | 적용 시점 |
|---|---|---|
| **Phase 1 (현재)** | 키 naming 규칙 준수, 공통 필드 권장 | MVP |
| **Phase 2~3** | DocumentType별 권장 필드 목록 제공, optional schema 참조 | 서비스 레이어 |
| **Phase 4~5** | 필수 필드 강제, JSON Schema 검증, 타입별 스키마 엔진 | 본격 도입 |

### 8-2. MVP 검증 원칙

- **완전 자유 키를 허용하지 않는다.** 최소한 naming 규칙(snake_case, namespace 접두사)은 적용한다.
- **공통 metadata 필드 키 목록을 플랫폼이 관리한다.** 알 수 없는 키를 `extended`에 자유롭게 허용하되, `common` 영역 키는 표준 목록을 따른다.
- **검증 책임은 서비스 레이어에 있다.** DB 레벨 강제 검증은 하지 않는다. (CLAUDE.md 원칙)
- **DocumentType별 required metadata는 Phase 3 이후 도입한다.** 현재는 권장 수준으로만 운영한다.

### 8-3. 검증 책임 위치

> **원칙: metadata 검증은 DB 레벨이 아닌 서비스 레이어에서 수행한다.**
>
> DocumentType별 분기 로직은 서비스 레이어에서만 처리한다. (CLAUDE.md 절대 규칙)

---

## 9. naming / namespace 규칙

### 9-1. 키 naming 규칙

**결정: snake_case를 기본 naming convention으로 채택한다.**

- 모든 metadata 키는 소문자 snake_case로 작성한다.
- 예: `effective_date`, `owner_department`, `meeting_datetime`

### 9-2. namespace 규칙

**결정: `common.*` + 타입별 namespace 구조를 채택한다.**

```
common.owner_department      → 공통 영역
common.visibility_scope      → 공통 영역
common.keywords              → 공통 영역

policy.effective_date        → policy 타입 전용
policy.governing_body        → policy 타입 전용
report.reporting_period_start → report 타입 전용
meeting.attendees            → meeting_note 타입 전용
```

- `common.*`: 모든 타입이 사용 가능한 표준 필드
- `{type_key}.*`: 해당 DocumentType 전용 확장 필드
- namespace 없는 키: 허용하되 `extended` 자유 영역으로 처리

### 9-3. 충돌 방지 원칙

- `common` namespace 키 목록은 플랫폼이 독점 관리한다.
- 타입별 namespace는 DocumentType의 key값과 일치시킨다.
- 사용자 정의 타입 도입 시 `custom.*` namespace를 별도 할당한다.

---

## 10. 최소 권장 metadata 구조안

### 10-1. 전체 개념 구조

```
metadata (JSON)
  ├── common
  │     └── {공통 표준 필드들}
  └── extended
        └── {DocumentType별 확장 필드들 (namespace 적용)}
```

### 10-2. Document-level metadata 예시

```
Document.metadata
  common:
    owner_department  : "법무팀"
    visibility_scope  : "internal"
    domain_category   : "regulatory"
    language          : "ko"
    keywords          : ["개인정보", "보호", "규정"]
  extended:
    template.template_purpose : "신규 직원 온보딩 양식"  (template_doc 전용)
```

### 10-3. Version-level metadata 예시

```
Version.metadata_snapshot
  common:
    applicable_scope  : "전사"
    revision_reason   : "법령 개정 반영"
  extended:
    policy.effective_date    : "2024-01-01"
    policy.revision_date     : "2024-03-15"
    policy.governing_body    : "준법지원팀"

  (또는 meeting_note 예시)
  extended:
    meeting.meeting_datetime  : "2024-03-20T14:00:00"
    meeting.meeting_location  : "본사 3층 회의실 A"
    meeting.attendees         : ["홍길동", "이영희", "김철수"]
    meeting.agenda_items      : ["Q1 실적 리뷰", "신규 프로젝트 승인"]
```

### 10-4. Node-level attributes / metadata 예시

```
Node.attributes (재현 필요 구조 옵션)
  table_cell: { colspan: 2, rowspan: 1, header: false }
  image:      { src: "...", alt: "조직도 이미지", width: 600 }
  code_block: { language: "python" }
  heading:    { level: 2 }

Node.metadata (운영 정보)
  { locked: true, author_note: "법무 검토 필요" }
```

### 10-5. MVP 포함 vs 후속 Phase

| 항목 | MVP 포함 | 비고 |
|---|---|---|
| `common` / `extended` 영역 구분 구조 | O | 저장 구조 기본 |
| 공통 metadata 표준 키 목록 정의 | O | 플랫폼 관리 |
| snake_case + namespace naming 규칙 | O | 서비스 레이어 적용 |
| Version metadata_snapshot 포함 | O | Task 1-4 결정 사항 |
| DocumentType별 권장 필드 목록 | O (참고 문서 수준) | 강제 아님 |
| DocumentType별 metadata schema 참조 | 구조만 준비 | 실제 검증은 후속 |
| 필수 metadata 강제 검증 | X | Phase 3 이후 |
| JSON Schema 기반 자동 검증 | X | Phase 4~5 |
| 검색 색인 자동화 | X | Phase 검색 단계 |

---

## 11. 설계 쟁점과 판단 결과

| 쟁점 | 선택한 방향 | 이유 |
|---|---|---|
| metadata를 완전 자유 JSON으로 둘지 | **공통 구조 + 자유 확장 (B 방식)** | 완전 자유는 관리 불가. 공통 영역을 표준화해야 검색/필터 기반 마련 가능 |
| 공통/유형별 metadata를 분리할지 | **분리 (`common` / `extended` 영역)** | 공통 필드는 플랫폼이 표준 관리, 유형별 필드는 확장 위임. 구분 없으면 키 충돌 |
| DocumentType별 schema 연결을 초기부터 둘지 | **구조만 준비, 실제 연결은 후속** | 초기 과설계 방지. `metadata_schema_ref` 참조 구조는 설계에 포함 |
| 검색 자주 쓰는 필드를 metadata에 둘지 코어로 올릴지 | **공통 metadata 표준 필드로 유지. 사용 빈도 높으면 코어 승격 검토** | 지금은 metadata로 충분. 운영 경험 후 코어 승격 판단 |
| Node-level metadata와 attributes를 분리할지 | **분리** | attributes = 재현 필요 구조 옵션, metadata = 운영 목적. 목적 다름 |
| namespace 규칙을 둘지 | **채택 (`common.*`, `{type_key}.*`)** | 키 충돌 방지, 타입별 필드 관리, 향후 커스텀 타입 확장 대비 |
| 필수 metadata 개념을 초기부터 둘지 | **초기엔 권장 수준만. Phase 3 이후 필수 도입** | 너무 이른 강제는 초기 도입 부담 증가. 운영 경험 후 결정 |

---

## 12. 다음 Task 입력값

### Task 1-8 (문서 상태 모델 기초 설계)와의 연결 포인트

- 상태 모델은 코어 필드(Document.status)로 관리한다. metadata에 상태를 두지 않는다.
- `archived_at`, `deleted_at` 같은 생명주기 타임스탬프도 코어 필드다.
- 타입별 상태 확장(policy의 `effective`, report의 `submitted`)은 상태 모델 설계에서 결정한다.
  이 상태들이 metadata로 흡수될지, 별도 상태 필드로 승격될지를 Task 1-8에서 판단한다.

### 저장 구조 설계 시 확인해야 할 metadata 관련 쟁점

- Document.metadata, Version.metadata_snapshot, Node.attributes, Node.metadata 각각의 저장 방식 (JSON 컬럼 vs 별도 테이블).
- `common` 영역과 `extended` 영역을 하나의 JSON에 담을지, 두 컬럼으로 분리할지.
- 검색/필터에 자주 쓰이는 공통 metadata 필드의 인덱싱 전략 (`owner_department`, `effective_date`, `visibility_scope` 등).
- `metadata_snapshot`의 크기 제한 정책 (대용량 JSON 방지).

### API 설계 시 확인해야 할 metadata 처리 기준

- 문서 생성/수정 API에서 metadata를 어떻게 수신할지 (전체 교체 vs 부분 병합).
- `common` 영역과 `extended` 영역을 API 레벨에서 구분할지, 단일 JSON으로 처리할지.
- 유효하지 않은 metadata 키 수신 시 오류 처리 전략 (거부 vs 경고 후 허용).
- DocumentType별 metadata 검증을 API 레이어에서 수행할지, 서비스 레이어에서 수행할지.

### 검색/필터 설계로 넘길 핵심 판단

- `common.owner_department`, `common.visibility_scope`, `common.domain_category`, `common.keywords`는 기본 필터 축으로 설계해야 한다.
- `policy.effective_date`, `report.reporting_period`, `meeting.meeting_datetime`은 날짜 범위 필터 대상이다.
- `extended` 영역의 자유 필드는 전문 검색 대상이며, 구조화 필터로 사용하려면 사전 색인 필요하다.
- 검색 성능을 위해 자주 쓰이는 유형별 필드를 별도 색인 컬럼으로 추출하는 전략을 검색 설계 단계에서 결정한다.

### UI 문서 생성/편집 플로우에 영향을 주는 판단

- 문서 생성 시 DocumentType을 선택하면, 해당 타입의 권장 metadata 필드를 UI 폼으로 안내해야 한다.
- `common` 영역 필드는 타입 무관 공통 폼으로, `extended` 영역은 타입별 동적 폼으로 렌더링한다.
- `metadata_schema_ref` 연결이 완성되면 UI 폼 자동 생성(schema-driven form)이 가능해진다.
- `template_doc` 타입의 `required_fields` 메타는 문서 작성 가이드 UI와 직결된다.
