# Phase 1 - Task 1-6. DocumentType 시스템 설계

## 1. 작업 목적

Task 1-1~1-5에서 확정한 요구사항, 엔티티 책임, Document/Version/Node 구조를 바탕으로,
**범용 문서 플랫폼에서 문서 유형을 정의하고 확장하는 DocumentType 시스템의 구조와 역할**을 설계한다.

이 문서는 DocumentType이 단순 분류 라벨을 넘어 플랫폼 내에서 어떤 의미를 가지며,
어디까지 강제력을 갖고, Node/metadata/상태 모델과 어떻게 연결되어야 하는지를 확정한다.

---

## 2. DocumentType의 역할 정의

### 2-1. DocumentType이 대표하는 것

- DocumentType은 **문서의 유형 분류를 표현**한다. 이 문서가 어떤 종류인지를 식별한다.
- DocumentType은 **단순 라벨을 넘어 확장 규칙의 기준**이 될 수 있다. 유형별 Node 구성, metadata 스키마, 상태 정책과 연결 가능하다.
- DocumentType은 **공통 코어 모델(Document/Version/Node)을 대체하지 않는다.** 코어 위에 유형별 차이를 얹는 레이어다.
- DocumentType은 **범용성과 특수성 사이의 균형 장치**다. 모든 유형을 동일하게 처리하지도, 완전히 별도 모델로 만들지도 않는다.

### 2-2. 책임 분리표

| 항목 | DocumentType 책임 | 귀속 위치 |
|---|---|---|
| 문서 유형 식별 | O | DocumentType (key/identifier) |
| 유형별 metadata 스키마 연결 | O (참조) | DocumentType → MetadataSchema |
| 유형별 Node 구성 규칙 연결 | O (참조) | DocumentType → NodeRule |
| 유형별 상태 정책 연결 | O (참조, 후속) | DocumentType → StateProfile |
| 문서 정체성 보유 | X | Document |
| 특정 시점 내용 보존 | X | Version |
| 본문 구조 표현 | X | Node |
| 실제 문서 내용 저장 | X | Node / Version |
| 권한/승인 워크플로 | X | 후속 Phase |

---

## 3. DocumentType의 성격 검토

### 3-1. 세 가지 관점 비교

| 관점 | 설명 | 장점 | 한계 | 적합 단계 |
|---|---|---|---|---|
| **A. 단순 enum/분류값** | `policy`, `manual`, `report` 등 고정 값 목록 | 단순, 구현 빠름, 초기 MVP에 최적 | 확장성 낮음, 유형별 규칙 연결 약함 | MVP 초기 |
| **B. 관리 가능한 타입 정의 개체** | key, display_name, metadata schema 참조, node rule 참조 등을 가진 독립 레코드 | 확장성 높음, 규칙 연결 가능, 관리 가능 | 설계 복잡도 다소 증가 | Phase 1~2 |
| **C. 플러그인형/사용자 정의 타입** | 외부 모듈로 타입 정의를 완전히 위임 | 장기적 유연성 매우 높음 | 초기 과설계 위험, 관리 복잡도 급증 | Phase 5 이후 |

### 3-2. 권장 방향

**결정: 관점 B를 기준으로 설계한다. 단, MVP 구현 시 처음에는 관점 A 수준으로 단순화해서 시작하고 B로 점진 확장한다.**

> "내장 타입을 문자열 key로 식별하되, 각 타입이 metadata schema / node rule 참조를 연결할 수 있는 구조를 처음부터 설계에 반영한다. 단, 연결 구현은 후속 Phase에서 완성한다."

이렇게 하면:
- MVP에서는 DocumentType을 key값 목록으로 단순 운영 가능하다.
- 구조는 처음부터 확장 포인트를 내포하므로, 나중에 코어를 변경하지 않고 규칙을 붙일 수 있다.

---

## 4. 초기 기본 타입 후보

### 4-1. 타입별 특성 요약

| key | 문서 목적 | 구조적 특징 | metadata 특징 | 상태 관리 특징 | 구분 포인트 |
|---|---|---|---|---|---|
| `policy` | 조직 규칙·정책 선언 | 조항 번호 계층, 섹션 체계 강함 | 시행일, 개정일, 관할부서, 승인자 | 높음 (공표·폐기 개념 필요) | 법적·조직적 효력. 개정 이력이 중요 |
| `manual` | 절차·방법 안내 | 단계형 순서 구조, 목차 중심 | 적용 대상, 버전, 작성부서 | 중간 | 순서 있는 절차 중심. guide보다 공식적 |
| `technical_doc` | 시스템·구조 설명 | 코드 블록, 다이어그램 참조, 복합 구조 | 연관 시스템, API 버전 | 중간 | 기술 맥락에 특화. 코드·다이어그램 포함 |
| `report` | 결과·현황 보고 | 표·서술형 혼합, 결론 강조 | 작성부서, 대상기간, 보고 대상 | 낮음 (제출 완료로 종결) | 시점 고정. 소급 수정 드묾 |
| `meeting_note` | 회의 내용 기록 | 항목 나열형, 참석자·결정사항 구조 | 회의일시, 참석자, 안건 | 낮음 (확정 후 고정) | 회의 맥락에 특화된 구조 |
| `knowledge_doc` | 지식·정보 정리 | 자유 형식, 링크·참조 구조 | 카테고리, 태그, 관련 문서 | 낮음 (게시 중심) | 백과사전형. 구조 제약 없음 |
| `template_doc` | 재사용 양식 제공 | template_slot 포함, 구조가 곧 내용 | 양식 용도, 필수 항목, 대상 부서 | 중간 (승인 후 활성화) | 문서 생성 기반 양식. 특수 노드 허용 |
| `generic_doc` | 분류 미정 문서 | 자유 형식 | 최소 공통 메타만 | 낮음 | fallback 타입. 분류 전 임시 또는 기타 |

### 4-2. policy와 regulation 통합 여부

**결정: `policy`로 통합. regulation은 policy의 하위 운영 분류로 처리한다.**

- 두 개념 모두 조직 내 규범을 정의한다는 점에서 코어 모델 수준 구조가 동일하다.
- `regulation`은 `policy` 타입 내의 metadata 분류값(sub_category 등)으로 구분 가능하다.
- 타입 수를 최소화하여 코어 단순성을 유지한다.

### 4-3. manual과 guide 통합 여부

**결정: `manual`로 통합. guide는 manual의 운영 분류로 처리한다.**

- 두 개념 모두 절차·방법 안내 문서이며 구조적 특징이 유사하다.
- 형식적 엄밀함(manual이 더 공식적)의 차이는 metadata 필드로 표현 가능하다.

---

## 5. 타입 체계 수준 정리

### 5-1. 타입 분류

| 분류 | 타입 목록 | 비고 |
|---|---|---|
| **코어 기본 타입 (MVP)** | `policy`, `manual`, `technical_doc`, `report`, `meeting_note`, `knowledge_doc`, `template_doc`, `generic_doc` | 8종으로 시작 |
| **통합 처리 (하위 분류로)** | `regulation` → `policy`, `guide` → `manual` | metadata sub_category로 구분 |
| **fallback 타입** | `generic_doc` | 분류 미정 또는 기타 문서의 기본값 |
| **후속 확장 타입** | 사용자 정의 타입, 도메인 특화 타입 | Phase 3 이후 |

### 5-2. template_doc의 성격

**결정: `template_doc`는 문서 자체의 타입으로 유지한다.**

- template_doc는 "이 문서가 다른 문서를 생성하기 위한 양식"이라는 정체성을 가진다.
- 생성 방식(템플릿 기반)을 나타내는 것이 아니라, 문서의 목적(재사용 양식)을 나타낸다.
- `template_slot` 같은 특수 Node 허용이 이 타입에만 적용된다.

---

## 6. Document와의 관계

### 6-1. 관계 원칙

- **Document는 하나의 DocumentType을 반드시 가져야 한다.** 타입 없는 문서는 허용하지 않는다. (최소 `generic_doc` 사용)
- DocumentType은 `Document.document_type` 필드에 key 값으로 저장된다.
- DocumentType은 여러 Document가 공유하는 분류 기준이다.

### 6-2. type 변경 허용 여부

**결정: type 변경은 제한적으로 허용하되, 운영 정책에서 엄격히 관리한다.**

| 상황 | 처리 방식 |
|---|---|
| draft 상태의 문서 type 변경 | 허용 |
| published 문서의 type 변경 | 원칙적으로 불허. 예외 시 새 버전 생성 + 변경 이유 기록 |
| 타입이 다른 두 문서 병합 | 후속 Phase에서 정책 결정 |

### 6-3. Version별 type 스냅샷 여부

**결정: Version에 type_snapshot을 두지 않는다. (MVP 기준)**

- DocumentType은 문서의 정체성에 가까운 속성이므로 Document에 귀속한다.
- type이 변경된다면 그 시점부터 새 Document 또는 새 Version으로 관리한다.
- 재현 필요성이 높아지는 시점에 Version.type_snapshot을 추가할 수 있다.

---

## 7. Node 규칙과의 연결

### 7-1. 연결 방식 검토

| 강도 | 설명 | 장점 | 단점 |
|---|---|---|---|
| **강제 (hard constraint)** | 허용하지 않는 node_type 사용 시 저장 거부 | 구조 일관성 보장 | 초기 유연성 감소, 구현 복잡 |
| **권장 (soft guidance)** | 유형별 권장 node_type 목록 제공, 위반 시 경고만 | 유연성 높음, 초기 구현 단순 | 구조 일관성 보장 약함 |
| **없음** | 모든 타입이 모든 node_type 사용 가능 | 최대 유연성 | 타입별 구조 차별화 불가 |

**결정: MVP에서는 권장(soft guidance) 수준으로 시작한다. 강제 검증은 후속 Phase로 미룬다.**

### 7-2. 타입별 Node 구성 가이드 (권장 수준)

| DocumentType | 권장 Node 구성 | 특수 허용 Node |
|---|---|---|
| `policy` | section(조) → section(항) → paragraph, reference | - |
| `manual` | section → list, list_item, paragraph, image | - |
| `technical_doc` | section → paragraph, code_block, table, image | - |
| `report` | section → paragraph, table, heading | - |
| `meeting_note` | section(안건) → list_item(결정사항), paragraph | - |
| `knowledge_doc` | section → paragraph, list, callout, reference | - |
| `template_doc` | section → template_slot, paragraph | `template_slot` |
| `generic_doc` | 제한 없음 | 제한 없음 |

### 7-3. Node 규칙 연결 구조 (설계 예비)

DocumentType이 Node 규칙을 참조할 수 있도록 구조에 `node_rule_ref`를 포함한다.
MVP에서는 이 참조가 NULL이거나 비어있어도 되며, 후속 Phase에서 규칙 엔진과 연결한다.

---

## 8. metadata 템플릿과의 연결

### 8-1. 연결 방식

**결정: 타입별 metadata template 참조 구조를 설계에 포함하되, 실제 강한 검증은 후속 Phase로 미룬다.**

- 각 DocumentType은 `metadata_schema_ref`를 통해 권장 metadata 스키마를 참조할 수 있다.
- MVP에서는 이 참조가 없어도 동작한다.
- 서비스 레이어에서 DocumentType을 기반으로 metadata 검증을 수행하는 구조를 준비한다.

### 8-2. 타입별 권장 metadata 필드

| DocumentType | 권장 metadata 필드 | 필수 여부 (초기) |
|---|---|---|
| `policy` | 시행일, 개정일, 관할부서, 법적 근거, 승인자 | 선택 → 후속 강화 |
| `manual` | 적용 대상, 작성부서, 적용 버전 | 선택 |
| `technical_doc` | 연관 시스템, API 버전 | 선택 |
| `report` | 작성부서, 대상기간, 보고 대상자 | 선택 |
| `meeting_note` | 회의일시, 참석자, 장소, 안건 목록 | 선택 |
| `knowledge_doc` | 카테고리, 태그, 관련 문서 | 선택 |
| `template_doc` | 양식 용도, 필수 항목, 대상 부서 | 선택 |
| `generic_doc` | 없음 (자유) | - |

---

## 9. 상태 모델과의 연결

### 9-1. 공통 상태 모델 유지

**결정: MVP에서는 모든 DocumentType이 공통 상태 모델을 공유한다.**

공통 상태: `draft → review → published → archived`

### 9-2. 타입별 상태 확장 필요성 검토

| DocumentType | 추가 상태 필요성 | 후속 확장 방향 |
|---|---|---|
| `policy` | 높음 | `effective`(시행 중), `deprecated`(폐기) 상태 추가 가능 |
| `report` | 중간 | `submitted`(제출 완료) 상태 추가 가능 |
| `meeting_note` | 낮음 | `confirmed`(확정) 정도 추가 가능 |
| `template_doc` | 중간 | `active`(활성 양식) 상태 추가 가능 |
| 나머지 | 낮음 | 공통 모델로 충분 |

### 9-3. 상태 확장 연결 구조

DocumentType에 `state_profile_ref`를 포함하여, 후속 Phase에서 타입별 상태 프로필을 연결할 수 있도록 구조를 열어둔다.

---

## 10. 확장 전략

### 10-1. 확장 시나리오별 판단

| 확장 시나리오 | Phase 1 지원 여부 | 비고 |
|---|---|---|
| 내장 기본 타입 8종 제공 | O | MVP 코어 |
| 관리자 정의 타입 추가 | 구조만 준비 | key 기반 레코드 추가 가능하도록 설계 |
| 사용자 정의 타입 | X | Phase 3 이후 |
| 외부 시스템 연동형 타입 | X | Phase 5 이후 |
| 타입 간 상속/확장 | X | 후속 Phase |

### 10-2. 확장을 위한 구조 원칙

- DocumentType은 **문자열 key 기반**으로 식별한다. 코드에 hardcoding된 enum이 아닌, 레코드로 관리한다.
- 새 타입 추가는 레코드 삽입으로 가능해야 한다. 코드 변경 없이 타입을 늘릴 수 있어야 한다.
- 알 수 없는 타입을 처리할 때는 `generic_doc` 동작을 기본으로 한다.
- `is_builtin` 플래그로 내장 타입과 커스텀 타입을 구분한다.

---

## 11. 최소 권장 DocumentType 구조안

### 11-1. DocumentType 개념 모델

#### 필수 필드 (MVP 코어)

```
DocumentType
  - key             : 타입 고유 식별자 (문자열, 예: "policy", "manual")
  - display_name    : 사람이 읽을 수 있는 타입 이름
  - is_builtin      : 내장 타입 여부 (true/false)
  - is_active       : 활성 여부. 비활성 타입으로 새 문서 생성 불가
```

#### 권장 필드 (MVP 포함 권장)

```
  - description     : 타입 설명
  - group           : 타입 그룹 분류 (예: "regulatory", "technical", "operational")
```

#### 확장 참조 필드 (후속 Phase 연결)

```
  - metadata_schema_ref   : 이 타입의 권장 metadata 스키마 참조
  - node_rule_ref         : 이 타입의 Node 구성 규칙 참조
  - state_profile_ref     : 이 타입의 상태 프로필 참조
```

### 11-2. 초기 내장 타입 목록

| key | display_name | group | 비고 |
|---|---|---|---|
| `policy` | 규정·정책 문서 | regulatory | regulation 포함 |
| `manual` | 매뉴얼·가이드 | operational | guide 포함 |
| `technical_doc` | 기술 문서 | technical | |
| `report` | 보고서 | operational | |
| `meeting_note` | 회의록 | operational | |
| `knowledge_doc` | 지식 문서 | knowledge | |
| `template_doc` | 템플릿 문서 | template | |
| `generic_doc` | 일반 문서 | general | fallback 타입 |

---

## 12. 설계 쟁점과 판단 결과

| 쟁점 | 선택한 방향 | 이유 |
|---|---|---|
| DocumentType을 enum 수준으로 시작할지 | **문자열 key 기반 레코드로 설계. 초기는 내장 타입만 운영** | enum은 코드 변경 없이 타입 추가가 불가. 레코드 방식으로 설계하면 MVP는 단순하되 확장 가능 |
| policy와 regulation을 분리할지 | **`policy`로 통합. regulation은 metadata 분류값** | 코어 구조가 동일함. 타입 수 최소화 원칙 |
| manual과 guide를 분리할지 | **`manual`로 통합. guide는 metadata 분류값** | 구조적 특징이 동일함. 운영 정책 차이는 metadata로 처리 |
| template_doc를 type으로 둘지 | **type으로 유지** | 문서의 목적이 "재사용 양식 정의"로 고유함. template_slot 같은 특수 Node와 연결 필요 |
| type별 Node 규칙을 강제할지 | **MVP: 권장 수준. 후속: 강제로 확장 가능** | 초기 유연성 유지. 구조 강제는 실제 운영 경험 후 결정 |
| type별 metadata 검증을 코어에 넣을지 | **코어에 넣지 않음. 서비스 레이어에서 담당** | CLAUDE.md 원칙: DocumentType별 분기 로직은 서비스 레이어에서만 |
| custom type을 초기부터 허용할지 | **구조는 열어두되, 초기에는 내장 타입만 활성화** | 과설계 방지. 관리자 정의 타입은 Phase 3 이후 도입 |

---

## 13. 다음 Task 입력값

### Task 1-7 (metadata 확장 구조 설계)로 넘길 포인트

- DocumentType별 권장 metadata 필드 목록이 확정되었다. (8종 타입별 권장 필드 참고)
- metadata 스키마를 DocumentType과 연결하는 `metadata_schema_ref` 구조를 설계에 반영해야 한다.
- Document-level metadata와 Version-level metadata의 타입별 분리 기준을 Task 1-7에서 확정한다.
- metadata 검증 책임은 서비스 레이어에 있음을 Task 1-7에서 원칙으로 명시해야 한다.

### Task 1-8 (문서 상태 모델 기초 설계)와의 연결 포인트

- 공통 상태 모델(`draft → review → published → archived`)은 모든 타입이 공유한다.
- `policy`의 `effective`/`deprecated`, `report`의 `submitted`, `template_doc`의 `active` 같은 타입별 상태 확장을 언제 도입할지 Task 1-8에서 결정한다.
- DocumentType의 `state_profile_ref` 연결 구조를 Task 1-8의 상태 모델 설계와 함께 정의한다.

### 저장 구조/API 설계에서 확인해야 할 DocumentType 관련 쟁점

- DocumentType 레코드의 저장 방식: 별도 테이블 vs 코드 내 정적 설정 파일.
- Document.document_type 필드의 타입: 문자열 key 직접 저장 vs DocumentType 테이블 외래 키.
- 알 수 없는 DocumentType key 처리 정책: 오류 vs generic_doc fallback.
- DocumentType 비활성화 시 기존 문서의 처리 방식.

### UI/문서 생성 플로우에서 영향을 받을 판단

- 문서 생성 UI에서 타입 선택이 필수이며, 타입에 따라 metadata 입력 폼이 달라진다.
- `template_doc` 타입은 일반 문서 생성과 다른 편집 UI가 필요하다 (template_slot 편집 모드).
- DocumentType의 `group` 분류로 UI에서 타입을 카테고리별로 정렬 표시할 수 있다.
- 타입 변경 기능은 UI에서 특별한 경고와 확인을 요구해야 한다 (published 문서의 경우).
