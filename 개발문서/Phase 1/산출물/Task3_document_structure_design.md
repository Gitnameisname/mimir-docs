# Phase 1 - Task 1-3. Document 구조 설계

## 1. 작업 목적

Task 1-1, 1-2에서 확정한 요구사항과 엔티티 책임 정의를 바탕으로,
범용 문서 플랫폼에서 **문서의 정체성(identity)을 대표하는 최상위 엔티티인 Document의 구조**를 상세 설계한다.

이 문서는 "Document가 무엇을 담고, 무엇을 담지 않아야 하는지"를 확정하여
이후 Version, Node, 상태 모델, API, 저장 구조 설계의 기준점이 된다.

---

## 2. Document의 역할 정의

### 2-1. Document가 대표하는 것

Document는 문서의 **지속적인 정체성**을 대표하는 엔티티다.

- 이 문서가 시스템에 존재한다는 사실 자체를 나타낸다.
- 문서의 유형(DocumentType), 현재 상태(status), 대표 제목(title)을 보유한다.
- 현재 유효한 버전(`current_version_id`)을 가리키는 포인터 역할을 한다.
- 문서가 어느 조직/워크스페이스에 속하는지 소속 정보를 가진다.
- 문서가 생성된 시점부터 아카이브/삭제되는 시점까지의 생명주기를 표현한다.

### 2-2. Document가 책임하지 않는 것

| 책임 항목 | 올바른 귀속 엔티티 | 이유 |
|---|---|---|
| 특정 시점의 문서 본문 내용 | `Version` | 내용은 시점마다 달라지므로 Version이 스냅샷으로 보관 |
| 문서 본문의 계층 구조 | `Node` | 내부 구조 표현은 Node 트리가 담당 |
| 버전별 세부 metadata | `Version` | 특정 버전에 종속된 속성은 Version에 귀속 |
| 본문 섹션/문단/표 내용 | `Node` | 블록 단위 콘텐츠는 Node가 담당 |
| 승인자/검토자 이력 | `AuditEvent`, `Approval` (후속) | 워크플로 엔티티가 별도 담당 |

### 2-3. Document의 핵심 역할 요약

> **Document = 껍데기(identity) + 포인터(current_version) + 대표 정보(title/status/type)**
>
> 실제 내용은 항상 Version → Node 트리를 통해 접근한다.

---

## 3. Document 필드 후보 목록

### 3-1. 전체 후보 검토표

| 필드명 | 의미 | 필수 여부 | MVP 포함 | 비고 |
|---|---|---|---|---|
| `id` | 시스템 내부 고유 식별자 | 필수 | Y | UUID 권장 |
| `slug` | 사람이 읽을 수 있는 외부 노출 식별자 | 선택 | N | 후속 Phase에서 도입 |
| `title` | 문서 대표 제목 | 필수 | Y | current_version의 title과 동기화 정책 필요 |
| `summary` | 문서 요약/설명 | 선택 | Y | 검색·목록 UI에 유용 |
| `document_type` | 문서 유형 분류 | 필수 | Y | DocumentType 참조 |
| `status` | 문서 생명주기 상태 | 필수 | Y | draft/review/published/archived |
| `current_version_id` | 현재 유효한 버전 참조 | 필수 | Y | NULL 허용 (초기 생성 직후) |
| `latest_version_id` | 가장 최근 생성된 버전 참조 | 선택 | Y | current와 latest가 다를 수 있음 |
| `created_by` | 문서 최초 생성자 | 필수 | Y | 사용자 식별자 |
| `created_at` | 문서 생성 일시 | 필수 | Y | 불변 |
| `updated_at` | 문서 마지막 변경 일시 | 필수 | Y | 버전 생성 또는 메타 변경 시 갱신 |
| `archived_at` | 아카이브 처리 일시 | 선택 | Y | NULL이면 활성 상태 |
| `deleted_at` | 소프트 삭제 일시 | 선택 | Y | NULL이면 삭제되지 않은 상태 |
| `metadata` | Document 수준 확장 속성 | 선택 | Y | JSON 구조, 유형별 스키마 연결 |
| `owner_id` | 문서 소유자(개인 또는 팀) | 선택 | Y | 권한 모델 연동 |
| `workspace_id` | 소속 워크스페이스/조직 | 선택 | Y | 멀티테넌트 대비 |
| `category` | 분류 카테고리 | 선택 | N | metadata 내 처리 가능 |
| `tags` | 검색/분류용 레이블 | 선택 | N | metadata 내 처리 가능, 후속 확장 |

---

## 4. 필드 분류 체계

### 4-1. 식별 필드

문서를 고유하게 특정하는 필드다.

| 필드 | 역할 | 비고 |
|---|---|---|
| `id` | 시스템 전체에서 고유한 내부 식별자 | 기술적 기준점 |
| `slug` | 외부 URL, API 경로에 노출되는 사람 친화적 식별자 | MVP 이후 도입 |

> `id`와 `slug`를 분리하는 이유: 내부 식별자는 변경 불가여야 하지만, 외부 노출 식별자(slug)는 문서 제목 변경 등으로 바뀔 수 있다. 두 가지를 분리해야 안정성을 유지할 수 있다.

### 4-2. 대표 정보 필드

문서를 사람이 이해할 수 있도록 표현하는 필드다.

| 필드 | 역할 | 비고 |
|---|---|---|
| `title` | 문서 대표 제목 | 목록, 검색 결과, 참조 시 노출 |
| `summary` | 문서 요약 설명 | 선택적, 목록 UI에서 유용 |

### 4-3. 분류/유형 필드

문서를 분류하고 유형별 확장 규칙과 연결하는 필드다.

| 필드 | 역할 | 비고 |
|---|---|---|
| `document_type` | DocumentType 참조 | 유형별 메타 스키마·노드 규칙의 기준 |

> `category`, `tags`는 metadata 확장 구조 내에서 처리하며, 코어 필드로 승격하지 않는다.

### 4-4. 상태/생명주기 필드

문서의 현재 생명주기 단계를 나타내는 필드다.

| 필드 | 역할 | 비고 |
|---|---|---|
| `status` | 문서 현재 상태 (draft/review/published/archived) | 전이 규칙 동반 |
| `archived_at` | 아카이브 처리 시각 | NULL이면 활성 |
| `deleted_at` | 소프트 삭제 시각 | NULL이면 삭제 안 됨 |

### 4-5. 버전 참조 필드

Document가 Version을 가리키는 포인터 필드다.

| 필드 | 역할 | 비고 |
|---|---|---|
| `current_version_id` | 현재 공식 유효 버전 참조 | published 기준의 대표 버전 |
| `latest_version_id` | 가장 최근 생성된 버전 참조 | draft 포함, current와 다를 수 있음 |

### 4-6. 감사/추적 필드

문서 생성·변경 이력을 추적하기 위한 필드다.

| 필드 | 역할 | 비고 |
|---|---|---|
| `created_by` | 최초 생성자 식별자 | 불변 |
| `created_at` | 문서 생성 일시 | 불변 |
| `updated_at` | 마지막 변경 일시 | 버전 생성·메타 수정 시 갱신 |

### 4-7. 확장 필드

문서 유형·운영 정책에 따라 유연하게 확장되는 필드다.

| 필드 | 역할 | 비고 |
|---|---|---|
| `metadata` | Document 수준 확장 속성 (JSON) | 유형별 스키마 검증 가능 |
| `owner_id` | 문서 소유자 | 권한 모델 연동 |
| `workspace_id` | 소속 조직/워크스페이스 | 멀티테넌트 확장 대비 |

---

## 5. 필수/선택 필드 판단

| 필드명 | 필수 여부 | MVP 포함 | 판단 근거 |
|---|---|---|---|
| `id` | 필수 | Y | 모든 참조의 기준점. 없으면 시스템 성립 불가 |
| `title` | 필수 | Y | 문서 식별과 목록 표시의 최소 요건 |
| `document_type` | 필수 | Y | 유형별 처리 분기의 기준. 없으면 확장 불가 |
| `status` | 필수 | Y | 생명주기 관리의 최소 요건 |
| `created_by` | 필수 | Y | 감사·책임 추적의 최소 요건 |
| `created_at` | 필수 | Y | 시간순 정렬, 이력 추적 |
| `updated_at` | 필수 | Y | 최신성 판단 기준 |
| `current_version_id` | 필수 | Y | 현재 내용 접근의 유일한 경로 (NULL 허용: 초기 생성 직후) |
| `latest_version_id` | 선택 | Y | current와 latest가 다른 경우(draft 편집 중) 구분에 유용 |
| `summary` | 선택 | Y | 목록·검색 UX 개선에 유용. 없어도 작동 가능 |
| `metadata` | 선택 | Y | 유형별 확장 속성 수용을 위해 MVP에 포함 권장 |
| `owner_id` | 선택 | Y | 권한 모델 연동을 위해 MVP에 포함 권장 |
| `workspace_id` | 선택 | Y | 멀티테넌트 대비. MVP에서도 기본값으로 포함 권장 |
| `archived_at` | 선택 | Y | 소프트 아카이브 지원을 위해 포함 |
| `deleted_at` | 선택 | Y | 소프트 삭제 지원을 위해 포함 |
| `slug` | 선택 | N | URL 친화성. MVP 이후 도입 |
| `category` | 선택 | N | metadata로 흡수. 별도 필드 불필요 |
| `tags` | 선택 | N | metadata로 흡수. 후속 Phase에서 독립 처리 |

---

## 6. Document와 Version의 책임 경계

### 6-1. title의 위치

**결정: Document에 대표 title을 두고, Version에도 snapshot title을 둔다.**

| 관점 | 내용 |
|---|---|
| Document.title | 현재 문서를 대표하는 제목. 목록·검색·참조에 사용 |
| Version.title_snapshot | 해당 버전 생성 시점의 제목. 이력 복원·비교에 사용 |
| 동기화 정책 | 새 버전이 current_version이 될 때 Document.title을 갱신할 수 있음 |

### 6-2. metadata의 분리

| 수준 | 내용 | 예시 |
|---|---|---|
| Document-level metadata | 문서 자체의 소속·운영 정보. 버전과 무관하게 지속 | 소유 조직, 공개 범위, 문서 담당 부서 |
| Version-level metadata | 해당 버전 시점에 종속된 속성 | 시행일, 개정 번호, 검토 완료일, 보고 대상 기간 |
| Node-level metadata | 특정 본문 블록에 종속된 속성 | 섹션 작성자, 강조 스타일, 블록 주석 |

### 6-3. 상태의 위치

**결정: MVP에서는 Document 수준 상태로 운영한다.**

- `Document.status`는 문서 전체의 현재 공식 상태를 나타낸다.
- 특정 Version에 대한 검토·승인 상태는 향후 Version 수준 상태 또는 Approval 엔티티로 확장한다.
- 예: "이 문서는 published 상태다" → Document.status
- 예: "이 버전은 검토 중이다" → 후속 Phase에서 Version.review_status 확장

### 6-4. current_version_id vs latest_version_id

**결정: 둘 다 Document에 보유한다.**

| 필드 | 의미 | 상황 예시 |
|---|---|---|
| `current_version_id` | 현재 공식 유효 버전 (published 기준) | 독자·외부 API가 접근하는 기준 버전 |
| `latest_version_id` | 가장 최근 생성된 버전 (draft 포함) | 편집 중인 버전. current와 다를 수 있음 |

> 두 값이 일치할 때: 편집 중인 draft 없음, current가 최신  
> 두 값이 다를 때: latest는 draft 편집 중, current는 이전 published 버전

### 6-5. updated_at의 기준

**결정: Document.updated_at은 Document 수준 변경(메타 수정, 새 버전 생성) 시 갱신한다.**

- 새 Version이 생성되면 Document.updated_at 갱신
- Document.title, status, metadata 변경 시 Document.updated_at 갱신
- Version 내부 Node 변경은 해당 Version 생성 시점에 반영됨 (Version은 불변이므로)

---

## 7. Document-level Metadata 범위

Document 수준 metadata는 **버전과 무관하게 문서 자체에 지속적으로 귀속되는 속성**이다.

### 7-1. Document metadata에 적합한 항목

| 항목 | 설명 | 비고 |
|---|---|---|
| 소유 조직 / 담당 부서 | 문서가 어느 조직 단위에 속하는지 | 권한 모델 연동 |
| 공개 범위 | 전체 공개 / 조직 내 / 특정 그룹 | 접근 제어 기반 |
| 문서 분류 코드 | 기관 내부 분류 체계 코드 | 유형별 확장 |
| 언어 | 문서 작성 언어 | 다국어 지원 대비 |
| 태그 / 키워드 | 검색·분류용 레이블 | metadata 내 배열로 처리 |
| 커스텀 분류값 | 기관별 특수 분류 필드 | 유형별 스키마로 검증 |

### 7-2. Document metadata에 적합하지 않은 항목

| 항목 | 이유 |
|---|---|
| 시행일, 개정일 | 버전 시점에 종속 → Version-level metadata |
| 참석자, 안건 목록 | 버전 시점에 종속 → Version-level metadata |
| 본문 섹션 주석 | 블록 단위 속성 → Node-level metadata |

---

## 8. 최소 권장 Document 구조안

### 8-1. 필수 필드 (MVP 코어)

```
Document
  - id                  : 전역 고유 식별자
  - title               : 문서 대표 제목
  - document_type       : 문서 유형 (DocumentType 참조)
  - status              : 문서 상태 (draft | review | published | archived)
  - current_version_id  : 현재 유효 버전 참조 (nullable: 초기 생성 시)
  - created_by          : 생성자 식별자
  - created_at          : 생성 일시 (불변)
  - updated_at          : 마지막 변경 일시
```

### 8-2. 권장 필드 (MVP 포함 권장)

```
  - latest_version_id   : 최근 생성 버전 참조 (draft 편집 구분)
  - summary             : 문서 요약 설명
  - metadata            : Document-level 확장 속성 (JSON)
  - owner_id            : 문서 소유자
  - workspace_id        : 소속 워크스페이스/조직
  - archived_at         : 아카이브 일시 (nullable)
  - deleted_at          : 소프트 삭제 일시 (nullable)
```

### 8-3. 선택 확장 필드 (후속 Phase)

```
  - slug                : 외부 노출용 식별자 (Phase 2~3)
  - category            : metadata로 흡수 또는 후속 확장
  - tags                : metadata 내 처리 또는 후속 Tag 엔티티
```

---

## 9. 설계 쟁점과 판단 결과

| 쟁점 | 선택한 방향 | 이유 |
|---|---|---|
| `current_version_id`와 `latest_version_id`를 둘 다 둘 것인가 | **둘 다 보유** | 편집 중인 draft와 공식 published 버전을 구분해야 실용적. 두 값이 같으면 draft 없는 상태 |
| `summary`를 Document에 둘 것인가 | **포함 (선택 필드)** | 목록 UI, 검색 결과에서 요약 없이 제목만으로는 부족. 단, 필수는 아님 |
| `tags`를 코어에 포함할 것인가 | **포함하지 않음, metadata로 처리** | tag는 metadata.tags 배열로 충분. 별도 필드/엔티티 승격은 후속 Phase |
| 삭제 표현을 soft delete로 열어둘 것인가 | **soft delete (`deleted_at`) 채택** | 문서는 영구 삭제보다 이력 보존이 중요. 복원 가능성 유지 |
| `slug`/`external_id`를 초기에 둘 것인가 | **MVP에서 제외, 후속 도입** | MVP에서는 `id`로 충분. slug는 URL 설계와 함께 결정 필요 |
| `title`을 Document와 Version 중 어디에 둘지 | **둘 다 보유 (동기화 정책 적용)** | Document.title은 현재 대표 제목, Version.title_snapshot은 해당 시점 이력 보존 |
| 상태를 Document/Version 어디에 둘지 | **MVP: Document 수준, 이후 Version 확장** | 초기엔 "이 문서의 현재 상태"로 단순화. 승인 흐름 도입 시 Version 상태 분리 |

---

## 10. 다음 Task 입력값

### Task 1-4 (Version 구조 설계)로 전달할 항목

- Document는 `current_version_id`와 `latest_version_id`를 통해 Version을 참조한다.
- Version은 Document.title의 시점 스냅샷(`title_snapshot`)을 보유해야 한다.
- Version은 해당 시점에 종속된 metadata(시행일, 개정 번호 등)를 보유해야 한다.
- Version은 불변이어야 한다. 내용 수정 = 새 Version 생성.
- `current_version_id`가 가리키는 Version이 published 기준인지, draft 포함 가능한지 명확히 결정 필요.
- draft_version과 published_version의 분리 방식을 결정해야 함.

### Task 1-5 (Node 트리 구조 설계)와의 연결 포인트

- Document 자체는 Node를 직접 보유하지 않는다.
- Node는 반드시 Version에 종속되며, Document → current_version_id → Version → Node 트리로 접근한다.
- Document-level metadata와 Node-level metadata의 경계를 Node 설계 시 명확히 확인해야 한다.

### Task 1-8 (상태 모델 설계)와의 연결 포인트

- `Document.status`는 `draft → review → published → archived` 전이를 가진다.
- 상태 전이 규칙(어떤 상태에서 어떤 상태로 전이 가능한지)은 Task 1-8에서 확정한다.
- `archived_at`, `deleted_at`의 상태 모델과의 관계(아카이브 = status 변경인지, 별도 필드인지)를 Task 1-8에서 결정한다.
- 향후 Version 수준 승인 상태 확장 시 Document.status와의 우선순위 규칙이 필요하다.
