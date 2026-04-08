

# Task 4-2 작업지시서

## 작업명

버전 모델 상세 설계

---

## 1. 작업 목적

Phase 4의 두 번째 작업으로, 문서 플랫폼에서 문서의 변경 이력을 안정적으로 관리하기 위한 **Version 모델**을 상세 설계한다.

이번 작업의 목적은 다음과 같다.

- Document와 Version의 관계를 실제 구현 가능한 수준으로 구체화한다.
- Draft / Published 흐름에서 Version이 어떤 역할을 수행하는지 명확히 한다.
- 버전 식별자, 버전 번호, 상태, 스냅샷 범위를 일관된 규칙으로 정의한다.
- 이후 API 설계, DB 설계, 조회/복원 설계가 Version 모델 위에서 흔들리지 않도록 기준선을 확정한다.

이 문서는 이후 Task 4-4, 4-5, 4-6, 그리고 구현 단계의 데이터 모델 설계의 선행 기준 문서가 되어야 한다.

---

## 2. 작업 배경

Task 4-1에서 문서 작성/개정 생명주기와 Draft / Published 흐름이 정의되었다고 가정한다.

이제 그 흐름을 실제 데이터 모델 수준으로 내리기 위해, Version을 다음 질문에 답할 수 있는 구조로 정리해야 한다.

- Version은 무엇을 스냅샷으로 보존하는가?
- Draft와 Published는 같은 Version 모델을 공유하는가?
- 현재 문서가 참조하는 최신 Draft / 최신 Published는 어떻게 연결되는가?
- 복원 시 어떤 Version이 새롭게 생성되는가?
- 버전 번호와 내부 식별자는 어떤 관계를 가지는가?

즉, 이번 작업은 단순히 "버전 테이블 필드 나열"이 아니라, **문서 이력 관리의 중심 엔티티를 설계하는 작업**이다.

---

## 3. 작업 범위

### 포함 범위

- Version 엔티티 정의
- Version과 Document의 관계 정의
- Version 식별 전략 정의
- version_number / revision_id 정책 정의
- Draft / Published 상태 표현 방식 정의
- current draft / current published 포인터 연결 방식 정의
- 스냅샷 포함 범위 정의
- 부모 버전 / 파생 버전 관계 정의
- 복원 시 Version 생성 규칙 반영
- 조회와 복원을 고려한 필드 설계 원칙 정리

### 제외 범위

- 실제 DB 마이그레이션 작성
- ORM 코드 작성
- API 엔드포인트 구현
- diff 저장 구조 구현
- 대용량 성능 튜닝 구현
- 감사 로그 테이블 상세 설계

이번 작업은 **Version 모델 자체의 상세 설계**에 집중한다.

---

## 4. 핵심 설계 질문

이번 작업에서는 최소한 아래 질문들에 답할 수 있어야 한다.

1. Version은 Draft와 Published를 모두 표현할 수 있는 단일 모델인가?
2. Document는 Version들을 어떻게 참조하는가?
3. 버전 번호는 사용자에게 보여주는 번호와 내부 식별자를 분리해야 하는가?
4. Version은 문서 본문만 저장하는가, 메타데이터도 함께 스냅샷으로 가지는가?
5. 문서 구조(Node)는 Version에 종속되는가, Document에 종속되는가?
6. Published 전환 시 기존 Draft Version을 상태만 변경하는가, 별도 Published Version을 새로 생성하는가?
7. 복원은 과거 Version을 그대로 활성화하는가, 아니면 새 Version을 생성하는가?
8. 부모 버전(parent_version_id) 또는 기반 버전(base_version_id)이 필요한가?
9. 현재 최신 Draft와 현재 최신 Published를 Document가 별도 포인터로 가져야 하는가?
10. 이후 diff, 비교, 승인, 감사 추적 확장을 고려할 때 어떤 필드가 미리 필요하거나 고려되어야 하는가?

---

## 5. 작업 지시

아래 항목을 순서대로 설계 문서로 정리한다.

### 5-1. Version 모델의 역할 정의

먼저 Version이 시스템에서 무엇을 담당하는지 명확히 정리한다.

반드시 아래를 포함한다.

- Version의 정의
- Document와의 관계
- Draft / Published와의 관계
- 조회 시점에서의 역할
- 복원 시점에서의 역할
- 감사 추적과의 관계

여기서 중요한 점은 Version을 단순 보조 엔티티가 아니라, **문서 이력의 공식 기록 단위**로 정의하는 것이다.

---

### 5-2. Version 식별 전략 정의

Version을 식별하는 방식은 사용자 친화적 식별자와 시스템 내부 식별자를 분리해서 검토한다.

반드시 다음 항목을 다룬다.

- 내부 고유 식별자(id)
- 문서 내 버전 순번(version_number)
- 사용자 표시용 revision_label 또는 revision_code 필요 여부
- Draft 상태의 버전 번호 부여 시점
- Published 기준 버전 번호와 Draft 기준 버전 번호를 동일 체계로 운영할지 여부

예시 관점:

- `id`: 시스템 내부 UUID
- `version_number`: 문서 내 순차 증가 정수
- `revision_code`: 선택적 표시값 (예: v3, rev-2026-001)

단, 최종안은 단순성과 확장성 균형을 우선한다.

---

### 5-3. Version 상태 표현 방식 정의

Version이 어떤 상태를 가질 수 있는지 정의한다.

예시 후보:

- draft
- published
- archived
- superseded
- restored-derived

하지만 상태가 과도하게 많아지면 운영 복잡도가 올라가므로, 아래 두 가지 방식 중 어떤 것이 적절한지 비교 검토한다.

1. **최소 상태 방식**
   - draft / published 중심
2. **확장 상태 방식**
   - draft / published / archived / restored 등 세분화

각 방식의 장단점을 적고, 이번 Phase 기준 권장안을 제시한다.

---

### 5-4. 스냅샷 범위 정의

Version이 저장하는 스냅샷의 범위를 정의한다.

반드시 아래를 구분해서 검토한다.

- 문서 본문 구조(Node tree)
- 문서 메타데이터(title, summary, tags 등)
- 문서 타입 관련 정보(document_type)
- 상태 관련 정보
- 작성자/수정자 정보
- 렌더링용 파생 데이터 포함 여부

여기서 중요한 것은, Version이 **복원 가능성**을 보장할 수 있을 만큼 충분한 정보를 가져야 한다는 점이다.

단, 캐시성 데이터나 재생성 가능한 파생 데이터까지 과도하게 스냅샷에 포함하지 않도록 주의한다.

---

### 5-5. Document와 Version 연결 방식 정의

Document가 Version들을 어떤 방식으로 참조하는지 정의한다.

반드시 다음 내용을 다룬다.

- Document : Version = 1 : N 관계 정의
- current_draft_version_id 필요 여부
- current_published_version_id 필요 여부
- latest_version_id 필요 여부
- Draft와 Published 공존 시 조회 기준
- Version 목록 정렬 기준

여기서는 조회 성능, 구현 단순성, 운영 명확성을 함께 고려한다.

---

### 5-6. 부모/파생 관계 정의

Version 간 계보(lineage)가 필요한지 검토하고, 필요하다면 어떤 필드가 필요한지 정의한다.

검토 대상:

- parent_version_id
- base_version_id
- restored_from_version_id
- published_from_version_id

예를 들어 다음 흐름을 설명할 수 있어야 한다.

- Published v2를 기준으로 Draft v3 생성
- Published v1을 기반으로 복원 Draft v4 생성
- 기존 Draft를 수정하여 같은 Draft를 덮는지, 새 Draft Version을 생성하는지

최종안에서는 최소한 **복원 출처**와 **일반 개정 출처**를 설명할 수 있어야 한다.

---

### 5-7. 복원 친화적 모델 정의

Version 모델은 조회뿐 아니라 복원도 지원해야 한다.

다음 항목을 정리한다.

- 복원 시 원본 Version과 새 Version 관계
- 복원된 Version을 표시하는 방식
- 복원 사유/메모를 Version에 직접 둘지 별도 이벤트로 둘지 검토
- 복원 후 Published 즉시 전환 가능 여부와 별개로, 기본 모델은 무엇을 생성하는지 정의

권장 방향은, 복원 요청 시 **과거 Version을 다시 현재 활성 상태로 지정하는 것이 아니라, 새 Draft Version을 생성하는 방식**이다.

---

### 5-8. 필드 후보 목록 작성

Version 엔티티에 필요한 필드 후보를 정리한다.

최소한 아래 항목들은 검토 대상에 포함한다.

- id
- document_id
- version_number
- revision_label 또는 revision_code
- status
- title_snapshot
- metadata_snapshot
- content_snapshot
- document_type_snapshot
- parent_version_id
- restored_from_version_id
- created_by
- created_at
- published_at
- published_by
- change_summary
- checksum 또는 content_hash 필요 여부

각 필드에 대해 아래를 정리한다.

- 필드명
- 목적
- 필수 여부
- 생성 시점
- 변경 가능 여부

---

### 5-9. 최소 모델안과 확장 모델안 비교

최종 설계 전에 아래 두 안을 비교한다.

#### 안 A. 단순 단일 Version 모델

- Draft와 Published를 같은 Version 엔티티에서 상태값으로 구분
- Document는 current_draft_version_id / current_published_version_id만 가진다
- 복원은 새 Draft Version 생성

#### 안 B. 더 풍부한 계보형 Version 모델

- parent / restored_from / published lineage를 명시적으로 관리
- 상태와 관계를 더 세분화
- 추후 diff/승인 확장성이 높음

각 안의 장단점을 정리하고, **이번 Phase에서 채택할 권장안**을 명시한다.

---

## 6. 산출물 요구사항

이번 작업의 최종 산출물은 하나의 상세 설계 문서다.

문서에는 반드시 아래 항목이 포함되어야 한다.

1. 작업 목적
2. Version 모델 개요
3. Version 식별 전략
4. Version 상태 모델
5. 스냅샷 범위 정의
6. Document-Version 관계 정의
7. Version lineage 정의
8. 복원 관점 설계
9. 필드 후보 표
10. 권장 모델안 및 선택 근거
11. 후속 작업 영향도

가능하면 아래 시각화 초안도 포함한다.

- Mermaid ER 다이어그램 초안
- Mermaid 상태 또는 관계 다이어그램 초안

---

## 7. 산출물 작성 원칙

- 설계 문서 작성에 집중하고 구현 코드는 작성하지 않는다.
- 필드 나열에 그치지 말고, 각 필드가 왜 필요한지 설명한다.
- Task 4-1의 생명주기 정의와 충돌하지 않도록 용어를 일관되게 사용한다.
- 지나치게 무거운 모델은 피하되, 향후 diff / 승인 / 감사 / AI 개정 보조로 확장 가능한 구조를 고려한다.
- “현재 구현 최소안”과 “향후 확장 포인트”를 구분해서 작성한다.

---

## 8. 권장 방향

아래 방향을 우선안으로 검토한다.

### 우선안 A

- Document와 Version은 1:N 관계다.
- Version은 Draft와 Published를 공통 표현하는 단일 모델이다.
- Document는 `current_draft_version_id`와 `current_published_version_id`를 가진다.
- Version은 본문 구조와 핵심 메타데이터의 스냅샷을 가진다.
- Published 이후 수정은 기존 Published를 바꾸지 않고 새 Draft Version을 생성한다.
- 복원은 과거 Version을 그대로 재활성화하지 않고 새 Draft Version 생성으로 처리한다.
- lineage는 최소한 `parent_version_id`와 `restored_from_version_id` 정도를 우선 검토한다.

이 우선안은 다음 장점이 있다.

- 구현이 비교적 단순하다.
- Draft / Published 흐름을 명확히 유지할 수 있다.
- 복원과 감사 추적이 안전하다.
- 이후 diff, 승인, AI 개정 제안 기능으로 확장하기 쉽다.

---

## 9. 완료 기준

다음 조건을 만족하면 이번 작업이 완료된 것으로 본다.

- Version의 역할과 책임이 문서로 정리되어 있다.
- Document-Version 관계가 명확하게 정의되어 있다.
- 버전 식별 전략과 버전 번호 정책이 정리되어 있다.
- 스냅샷 범위가 복원 가능성을 보장하는 수준으로 정의되어 있다.
- 최소 1개의 권장 모델안이 선택 근거와 함께 제시되어 있다.
- 필드 후보 표가 후속 DB/API 설계에 활용 가능할 정도로 구체적이다.
- Task 4-4, 4-5, 4-6이 이 문서를 바탕으로 진행 가능하다.

---

## 10. 금지사항

이번 작업에서는 아래를 하지 않는다.

- 실제 ORM 모델 코드 작성
- 실제 DB 스키마 생성
- 마이그레이션 파일 작성
- API 응답 JSON 예시까지 상세 구현
- diff 알고리즘 설계
- 승인 워크플로 상세 설계

범위를 넘기지 말고, Version 모델 설계에 집중한다.

---

## 11. 후속 연계 작업

이 작업이 끝나면 다음 작업들이 이어진다.

- Task 4-4: 작성/수정 API 설계
- Task 4-5: Draft / Published 상태 관리 규칙 설계
- Task 4-6: 버전 조회, 이력 탐색, 복원 흐름 설계
- 구현 단계: Document / Version DB 모델 및 서비스 로직 설계

즉, 이번 문서는 Phase 4에서 **버전 관리의 중심 기준 문서** 역할을 한다.