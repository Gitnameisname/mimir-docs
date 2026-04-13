# Phase 1 - Task 1-5. Node 구조 설계

## 1. 작업 목적

Task 1-1~1-4에서 확정한 요구사항, 엔티티 책임, Document/Version 구조를 바탕으로,
**문서 내부 내용을 표현하는 최소 구조 단위인 Node의 구조**를 상세 설계한다.

Node는 단순한 텍스트 저장 레코드가 아니라,
다양한 문서 유형을 공통된 방식으로 표현할 수 있는 **트리 기반 문서 표현의 핵심 모델**이다.
이 문서는 Node가 무엇을 담고, 어떤 트리 규칙을 따르며, 어떤 타입 체계를 가져야 하는지를 확정한다.

---

## 2. Node의 역할 정의

### 2-1. Node가 대표하는 것

- Node는 **문서 내부 구조를 표현하는 기본 단위**다.
- Node는 **Version에 종속되는 문서 구성 요소**다. Version 없이 Node는 독립 존재하지 않는다.
- Node는 **구조와 내용을 함께 담을 수 있어야 한다.** (예: section은 구조, paragraph는 내용)
- Node는 **트리 기반 계층 구조를 형성**할 수 있어야 한다. 부모-자식 관계로 연결된다.
- Node는 **다양한 문서 유형(규정, 보고서, 회의록 등)을 공통 형식으로 수용**해야 한다.

### 2-2. 책임 분리표

| 항목 | Node 책임 | 귀속 엔티티 |
|---|---|---|
| 문서 구조 단위 표현 (섹션, 문단, 표 등) | O | Node |
| 본문 텍스트/내용 보관 | O | Node (content 필드) |
| 노드 유형 식별 | O | Node (node_type) |
| 부모-자식 관계 표현 | O | Node (parent_id) |
| 형제 노드 간 순서 표현 | O | Node (order) |
| 노드 수준 확장 속성 | O | Node (attributes/metadata) |
| 특정 시점 스냅샷 보장 | X | Version (Node는 Version에 종속) |
| 버전 번호 관리 | X | Version |
| 문서 유형 분류 | X | Document |
| 문서 생명주기 상태 | X | Document |
| 문서 대표 제목 | X | Document / Version |

---

## 3. Node 필드 후보 목록

| 필드명 | 의미 | 필수 여부 | MVP 포함 | 비고 |
|---|---|---|---|---|
| `id` | Node 고유 식별자 | 필수 | Y | UUID 권장 |
| `version_id` | 소속 Version 참조 | 필수 | Y | 외래 참조 |
| `parent_id` | 부모 Node 참조 | 필수 | Y | NULL이면 루트 노드 |
| `order` | 형제 노드 간 순서 | 필수 | Y | 정수, 낮을수록 앞 |
| `node_type` | 노드 유형 식별자 | 필수 | Y | 문자열 or enum |
| `title` | 노드 제목 (섹션명 등) | 선택 | Y | node_type에 따라 사용 여부 다름 |
| `content` | 노드 본문 내용 | 선택 | Y | 텍스트, 마크다운, 구조화 JSON 등 |
| `content_format` | content 해석 방식 | 선택 | Y | plain / markdown / json |
| `attributes` | node_type별 확장 속성 | 선택 | Y | JSON, 타입별 옵션 보관 |
| `metadata` | 노드 수준 운영 메타 | 선택 | Y | 작성자 주석, 잠금 여부 등 |
| `depth` | 트리에서의 깊이 | 선택 | N | 보조 정보. 조회 최적화 시 추가 |
| `path` | 루트부터 이 노드까지 경로 | 선택 | N | 보조 정보. 탐색 최적화 시 추가 |
| `ref_key` | 외부 참조용 키 | 선택 | N | 다른 문서에서 이 노드를 참조할 때 사용 |
| `is_required` | 템플릿 슬롯 필수 여부 | 선택 | N | template_doc 유형 전용 |
| `is_hidden` | UI 표시 여부 | 선택 | N | 후속 렌더링 설계와 연동 |
| `label` | 노드에 붙이는 레이블 | 선택 | N | 편집 UI 보조 |
| `created_at` | 노드 생성 일시 | 선택 | Y | Version 생성 시 함께 기록 |

---

## 4. 필드 분류 체계

### 4-1. 식별 및 소속 필드

Node를 고유하게 식별하고 어느 Version에 속하는지 나타내는 필드다.

| 필드 | 역할 |
|---|---|
| `id` | 시스템 전체에서 고유한 Node 식별자 |
| `version_id` | 이 Node가 속한 Version 참조. Node의 존재 범위를 확정 |

> Node는 Version 없이 독립 조회·참조되지 않는다. `version_id`는 Node의 생명 범위를 결정한다.

### 4-2. 트리 구조 필드

Node 간 계층 관계와 순서를 표현하는 필드다.

| 필드 | 역할 | 비고 |
|---|---|---|
| `parent_id` | 부모 Node 참조. NULL이면 이 Version의 루트 노드 | 필수 |
| `order` | 같은 부모를 가진 형제 노드들 사이의 순서 | 정수, 낮을수록 앞 |
| `depth` | 루트로부터의 깊이 (0 = 루트) | 보조 정보, MVP 이후 |
| `path` | 루트부터 이 노드까지 id 경로 문자열 | 탐색 최적화 보조, MVP 이후 |

> 기본 트리는 `parent_id + order`로 표현한다. `depth`와 `path`는 조회 성능 최적화 목적으로 추후 추가한다.

### 4-3. 타입 및 역할 필드

Node가 어떤 역할을 하는지 식별하는 필드다.

| 필드 | 역할 |
|---|---|
| `node_type` | 이 Node의 구조적·의미적 유형 (section, paragraph, table 등) |
| `label` | 사람이 붙이는 노드 레이블. 편집 UI 보조용 |

### 4-4. 콘텐츠 필드

Node가 직접 담는 내용 관련 필드다.

| 필드 | 역할 |
|---|---|
| `title` | 섹션 제목, 표 제목 등 노드 수준 제목 텍스트 |
| `content` | 본문 내용. 텍스트, 마크다운, 구조화 JSON 형태 |
| `content_format` | content 해석 방식 (`plain`, `markdown`, `json`) |

### 4-5. 확장 속성 필드

node_type별 추가 옵션을 수용하는 필드다.

| 필드 | 역할 |
|---|---|
| `attributes` | node_type별 구조·표시 옵션 (colspan, alt text, 정렬 등) |
| `metadata` | 노드 수준 운영 메타 (주석, 잠금, 작성자 표시 등) |
| `ref_key` | 외부 문서에서 이 노드를 직접 참조할 때 사용하는 키 |
| `is_required` | 템플릿 슬롯의 필수 입력 여부 |
| `is_hidden` | UI 렌더링 시 숨김 여부 |

### 4-6. 추적 필드

| 필드 | 역할 |
|---|---|
| `created_at` | 이 Node가 생성된 일시. Version 생성과 동시에 기록 |

> Node는 Version에 종속되므로 `updated_at`은 원칙적으로 불필요하다. Node 수정 = 새 Version 생성.

---

## 5. 트리 구조 표현 방식 검토

### 5-1. 방식 비교

| 방식 | 설명 | 장점 | 단점 |
|---|---|---|---|
| **Adjacency List** (`parent_id + order`) | 각 Node가 부모 id와 순서만 보유 | 단순, 이해 쉬움, 노드 이동·삽입 용이 | 전체 트리 조회 시 재귀 쿼리 필요 |
| **Path Enumeration** (`path`) | 루트부터 현재까지 id 경로 문자열 보유 | 특정 경로의 모든 자손 조회 빠름 | 노드 이동 시 경로 업데이트 필요 |
| **Nested Set** | 좌/우 경계값 보유 | 서브트리 조회 매우 빠름 | 삽입/이동 시 대규모 업데이트 필요, 복잡 |
| **Closure Table** | 모든 조상-자손 쌍을 별도 테이블에 저장 | 유연하고 빠른 조회 | 저장 공간 증가, 관리 복잡 |

### 5-2. 권장 방향

**결정: 기본은 `parent_id + order` (Adjacency List). 보조 정보로 `depth`, `path`는 후속 최적화 시 추가.**

근거:
- Version별 독립 Node 방식에서는 트리 전체를 한 번에 로딩하는 경우가 많아 재귀 쿼리 부담이 제한적이다.
- 노드 이동·재정렬·삽입이 단순하다.
- 구현 복잡도가 낮아 초기 단계에 적합하다.
- 성능 이슈가 생기는 시점에 `path` 또는 `depth` 보조 필드를 추가하면 된다.

---

## 6. Node type 체계 초안

### 6-1. Node type 분류표

| 분류 | node_type | 용도 | MVP 포함 | 비고 |
|---|---|---|---|---|
| **구조 노드** | `document_root` | Version의 트리 최상위 루트 | Y | 모든 Version에 반드시 1개 |
| | `section` | 문서의 주요 구획 단위 | Y | 하위에 다른 노드 포함 |
| | `subsection` | section의 하위 구획 | Y | section과 동일 모델, depth로 구분 가능 |
| | `container` | 논리적 그룹화 컨테이너 | N | 구조 목적만. 후속 확장 |
| **텍스트/본문 노드** | `paragraph` | 일반 텍스트 문단 | Y | 기본 본문 단위 |
| | `heading` | 제목 텍스트 (h1~h6) | Y | level은 attributes로 |
| | `quote` | 인용문 | Y | 블록 인용 |
| | `code_block` | 코드 블록 | Y | 언어 정보는 attributes로 |
| **리스트 계열** | `list` | 리스트 컨테이너 | Y | ordered/unordered는 attributes로 |
| | `list_item` | 리스트 개별 항목 | Y | list의 자식 노드 |
| **표 계열** | `table` | 표 컨테이너 | Y | |
| | `table_row` | 표의 행 | Y | table의 자식 |
| | `table_cell` | 표의 셀 | Y | table_row의 자식. colspan 등은 attributes로 |
| **미디어/참조 계열** | `image` | 이미지 | Y | src, alt는 attributes로 |
| | `reference` | 다른 문서·조항 참조 | Y | ref_key 활용 |
| | `embed` | 외부 콘텐츠 임베드 | N | 후속 확장 |
| | `attachment_ref` | 첨부 파일 참조 | N | Attachment 엔티티 도입 후 |
| **특수/확장 계열** | `callout` | 강조 박스 (주의, 팁 등) | Y | 유형은 attributes로 |
| | `template_slot` | 템플릿 빈 칸 | N | template_doc 유형 전용 |
| | `custom_block` | 사용자 정의 블록 | N | 플러그인 확장 시 |

### 6-2. node_type 확장 방식

**결정: 초기에는 닫힌 문자열 목록(enum 수준)으로 시작하되, 확장 가능한 문자열 타입으로 열어둔다.**

- `document_root`, `section`, `paragraph` 등 기본 타입은 플랫폼이 직접 정의한다.
- 향후 플러그인이나 사용자 정의 타입 추가를 위해 문자열 타입으로 저장한다.
- 알 수 없는 타입은 `custom_block`으로 fallback 처리한다.

---

## 7. 구조 노드와 콘텐츠 노드 구분

### 7-1. 구분 방식 검토

| 구분 방식 | 설명 | 장점 | 단점 |
|---|---|---|---|
| **공통 모델 + node_type으로 역할 구분** (권장) | 하나의 Node 모델을 사용하고 node_type이 구조/콘텐츠 역할 결정 | 단순한 구조, 확장 용이 | node_type별 필드 사용 규칙을 별도 관리 필요 |
| 구조 노드 / 콘텐츠 노드 별도 모델 | 두 종류의 엔티티를 분리 | 타입 안전성 높음 | 모델 복잡도 증가, 혼합 구조 처리 어려움 |

**결정: 하나의 공통 Node 모델을 사용하되, node_type에 따라 구조적 역할과 콘텐츠 역할을 구분한다.**

### 7-2. 구조 노드와 콘텐츠 노드의 특징

| 분류 | 예시 | content 유무 | 자식 노드 가능 여부 |
|---|---|---|---|
| 구조 노드 | `document_root`, `section`, `table`, `list` | 없거나 선택적 | O (자식 포함) |
| 콘텐츠 노드 | `paragraph`, `heading`, `table_cell`, `list_item` | 있음 | 부분적으로 가능 |
| 복합 노드 | `callout`, `quote` | 있을 수 있음 | O |

> 구조 노드는 `content`가 없어도 되며, 자식 Node로 구조를 표현한다.
> 콘텐츠 노드는 `content` 필드에 실제 텍스트·데이터를 담는다.

---

## 8. Node 콘텐츠 표현 범위

### 8-1. 항목별 처리 방식 판단

| 콘텐츠 항목 | 처리 방식 | 비고 |
|---|---|---|
| 일반 텍스트 | `content` 필드 직접 | `content_format: plain` |
| 마크다운 텍스트 | `content` 필드 직접 | `content_format: markdown` |
| 제목 텍스트 | `title` 또는 `content` 필드 | node_type: heading |
| 서식 정보 (굵기, 기울임 등) | `content` 내 인라인 마크업 | 서식을 필드로 분리하지 않음 |
| 표 셀 내용 | `content` 필드 | node_type: table_cell |
| 이미지 설명 (alt) | `attributes.alt` | content가 아닌 attributes로 |
| 이미지 경로 (src) | `attributes.src` | |
| 링크 URL | `attributes.href` 또는 content 내 인라인 | |
| 참조 키 | `ref_key` 필드 | 외부 참조용 |
| 템플릿 슬롯 이름 | `attributes.slot_name` | template_slot 전용 |
| 코드 블록 언어 | `attributes.language` | node_type: code_block |
| 리스트 순서 여부 | `attributes.ordered` | node_type: list |

### 8-2. content 단일화 원칙

**결정: `content` 하나로 텍스트 내용을 담고, `content_format`으로 해석 방식을 지정한다.**

- `content_format: plain` — 순수 텍스트
- `content_format: markdown` — 마크다운 인라인 포맷 포함
- `content_format: json` — 구조화된 리치 텍스트 표현 (이후 블록 에디터 연동 시)

> type-specific 세부 옵션은 `attributes` JSON으로 수용해 content 필드를 단순하게 유지한다.

---

## 9. metadata / attributes 범위

### 9-1. attributes vs metadata 구분

| 항목 | attributes | metadata |
|---|---|---|
| 목적 | node_type별 구조·표시 옵션 | 노드 수준 운영·관리 정보 |
| 예시 | colspan, alt, language, ordered, level, href | 작성자 주석, 잠금 여부, 검토 메모 |
| 변경 빈도 | 콘텐츠와 함께 관리 | 운영 중 별도 관리 가능 |
| 재현 필요성 | Version 재현에 필요 | 운영 정보, 재현 우선순위 낮음 |

**결정: `attributes`와 `metadata`를 분리한다.**

- `attributes`: node_type에 따른 구조·표시 속성. 콘텐츠 재현에 필요.
- `metadata`: 노드 수준 운영 메타. 재현보다 운영 목적.

### 9-2. node_type별 attributes 예시

| node_type | attributes 예시 |
|---|---|
| `heading` | `{ "level": 2 }` |
| `list` | `{ "ordered": true }` |
| `table_cell` | `{ "colspan": 2, "rowspan": 1, "header": false }` |
| `image` | `{ "src": "...", "alt": "...", "width": 400 }` |
| `code_block` | `{ "language": "python" }` |
| `callout` | `{ "type": "warning" }` |
| `reference` | `{ "target_doc_id": "...", "target_node_ref_key": "..." }` |

---

## 10. Version과의 관계

### 10-1. 기본 원칙

- **Node는 특정 Version에 귀속된다.** Version이 없으면 Node는 존재할 수 없다.
- **Version마다 독립된 Node 트리를 구성한다.** 다른 Version의 Node를 공유하지 않는다.
- **Node 수정은 새 Version 생성으로 이어진다.** 기존 Version의 Node는 변경하지 않는다.
- **모든 Version의 트리는 단 하나의 루트 노드(`document_root`)로 시작한다.**
- Version은 `root_node_id`를 통해 Node 트리의 진입점을 참조한다.

### 10-2. 루트 노드 정의

- 모든 Version은 `node_type: document_root`인 노드를 하나 가진다.
- 이 루트 노드의 `parent_id`는 NULL이다.
- `document_root`는 직접적인 콘텐츠를 담지 않으며, 하위 section들의 부모 역할만 한다.

### 10-3. Node 공유 방식 검토 (대안)

| 방식 | 설명 | 채택 여부 |
|---|---|---|
| **Version별 독립 Node** (채택) | 각 Version이 완전히 독립된 Node 트리를 가짐 | O |
| Node 공유 + delta | 변경된 Node만 새로 생성하고 나머지는 공유 | X (MVP 이후 검토) |

> MVP에서는 독립 Node 방식을 채택한다. 데이터 증가는 저장 최적화로 해결한다.

---

## 11. 최소 권장 Node 구조안

### 11-1. 필수 필드 (MVP 코어)

```
Node
  - id              : Node 고유 식별자
  - version_id      : 소속 Version 참조
  - parent_id       : 부모 Node 참조 (NULL이면 루트)
  - order           : 형제 노드 간 순서 (정수)
  - node_type       : 노드 유형 (문자열)
```

### 11-2. 권장 필드 (MVP 포함 권장)

```
  - title           : 노드 제목 (섹션명 등, nullable)
  - content         : 노드 본문 내용 (nullable)
  - content_format  : content 해석 방식 (plain / markdown / json)
  - attributes      : node_type별 확장 속성 (JSON, nullable)
  - created_at      : 노드 생성 일시
```

### 11-3. 선택 확장 필드 (후속 Phase)

```
  - metadata        : 노드 수준 운영 메타 (JSON)
  - depth           : 트리 깊이 (조회 최적화 보조)
  - path            : 루트부터의 경로 문자열 (탐색 최적화 보조)
  - ref_key         : 외부 참조용 키
  - is_required     : 템플릿 슬롯 필수 여부
  - is_hidden       : UI 표시 숨김 여부
```

---

## 12. 설계 쟁점과 판단 결과

| 쟁점 | 선택한 방향 | 이유 |
|---|---|---|
| `depth`/`path`를 코어에 둘지 | **MVP 제외, 후속 추가** | parent_id + order로 충분. 성능 이슈 시 추가 |
| `title`과 `content`를 분리할지 | **분리 (별도 필드)** | 섹션 제목(title)과 본문(content)은 의미가 다름. 제목 없는 노드, 내용 없는 구조 노드 모두 표현 가능 |
| `content_format`을 둘지 | **포함** | plain/markdown/json 세 가지 해석 방식을 구분해야 에디터·렌더러 연동 가능 |
| `table_row` / `table_cell`을 별도 type으로 둘지 | **별도 type 유지** | table 구조를 단일 content 필드로만 표현하면 셀 단위 편집·참조가 불가. Node 트리로 표현해야 범용성 확보 |
| `custom_block`을 초기부터 허용할지 | **node_type 문자열로 열어두되, 초기 정의 타입만 공식 지원** | 무분별한 custom 허용보다 정의된 타입 우선 사용. 알 수 없는 타입은 custom_block fallback |
| `metadata`와 `attributes`를 분리할지 | **분리** | attributes는 콘텐츠 재현에 필요한 구조 옵션, metadata는 운영 목적. 목적과 변경 주기가 다름 |
| shared node를 허용할지 | **MVP에서 불허** | 재현 가능성과 불변성 원칙 유지. 저장 효율은 후속 최적화로 |

---

## 13. 다음 Task 입력값

### Task 1-6 (DocumentType 시스템 설계)와의 연결 포인트

- DocumentType은 해당 유형 문서에서 **허용하는 node_type 목록**을 정의할 수 있어야 한다.
  - 예: `meeting_note` 유형은 `agenda_item`, `decision`, `action_item` 같은 특수 노드 허용
  - 예: `template_doc` 유형은 `template_slot` 노드 허용
- DocumentType이 node_type 제약을 강제하는 방식(허용 목록 vs 권장 목록)을 Task 1-6에서 결정한다.
- `custom_block` 허용 범위도 DocumentType 정책에 연동할 수 있다.

### Task 1-7 (metadata 확장 구조 설계)로 넘길 포인트

- Node-level `attributes`와 `metadata`의 정확한 경계 기준을 Task 1-7에서 확정한다.
- 노드 수준 metadata 스키마 검증 방식(자유 JSON vs 타입별 스키마)을 결정해야 한다.
- Document-level / Version-level / Node-level metadata 간 중복 방지 원칙을 정리해야 한다.

### 저장 구조 설계(Phase 2) 시 확인해야 할 Node 관련 쟁점

- Version별 독립 Node 방식은 버전 생성마다 전체 Node 트리를 복사한다. 대규모 문서에서 저장 공간 최적화 전략 필요.
- `parent_id` 기반 트리의 재귀 조회 성능. 깊이가 깊어지면 `path` 또는 `depth` 보조 인덱스 추가 검토.
- `attributes` JSON 필드의 인덱싱 전략 (일부 attributes는 검색 기준이 될 수 있음).
- `document_root` 노드 보장 방식 (Version 생성 시 자동 생성 규칙 필요).

### 렌더링/편집기 설계(Phase 4) 시 영향을 주는 구조적 판단

- `content_format` 필드가 렌더러 선택의 기준이 된다. 렌더러는 format에 따라 다르게 처리해야 한다.
- `node_type + attributes` 조합이 렌더링 컴포넌트 선택의 기준이 된다.
- Node 트리의 `parent_id + order` 구조를 기반으로 에디터의 블록 삽입·이동·삭제 동작을 설계해야 한다.
- `is_hidden`, `is_required` 같은 확장 필드는 에디터 UI 동작과 직결되므로 도입 시 렌더링 설계와 함께 결정한다.
