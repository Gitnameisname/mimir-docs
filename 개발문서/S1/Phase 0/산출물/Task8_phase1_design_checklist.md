# Phase 0 → Phase 1 도메인 모델 설계 체크리스트

## 사용 목적

이 체크리스트는 Phase 1 도메인 모델 설계 리뷰에서 Phase 0 원칙 준수 여부를 확인하는 도구다. 설계안을 검토할 때 항목별로 확인하고, 통과하지 못한 항목은 수정하거나 예외 사유를 명시한다.

---

## 공통 (모든 엔티티 공통 적용)

- [ ] 이 엔티티는 특정 문서 유형에 종속된 구조를 가지고 있지 않은가?
- [ ] 이 엔티티는 DB 테이블 컬럼 구조와 동일하지 않은가? (저장 방식이 모델을 지배하고 있지 않은가?)
- [ ] 이 엔티티는 API 리소스(`/resource/{id}`)로 자연스럽게 노출될 수 있는 구조인가?
- [ ] 이 엔티티에 외부 시스템의 식별자(SSO OID, 외부 DB ID 등)가 코어 필드로 포함되어 있지 않은가?
- [ ] 이 엔티티에 저장소 특화 필드(파티션 키, row_version 컬럼 등)가 포함되어 있지 않은가?
- [ ] User가 보는 이 엔티티와 Admin이 보는 이 엔티티의 관점 차이가 고려되었는가?
- [ ] 이 엔티티의 변경이 감사 로그와 연결될 수 있는 구조인가?

---

## Document

- [ ] Document는 파일 레코드가 아니라 독립적인 지식 자산 단위로 정의되어 있는가?
- [ ] Document 엔티티 내부에 문서 본문 텍스트가 직접 포함되어 있지 않은가? (본문은 Version 또는 오브젝트 스토리지 참조여야 함)
- [ ] Document가 DocumentType에 의존하지만, 코어 Document 구조 자체는 특정 유형에 종속되지 않는가?
- [ ] Document는 Version과 연결될 수 있는 구조인가? (currentVersionId 또는 동등한 참조)
- [ ] Document는 Node/Structure와 연결될 수 있는 구조인가?
- [ ] Document는 Metadata와 연결될 수 있는 구조인가?
- [ ] Document는 Status를 가지는가? (단순 boolean이 아닌 상태 모델 연결)
- [ ] Document와 첨부 파일의 관계는 참조(AttachmentId) 방식인가? (파일 원본을 Document가 직접 포함하지 않는가?)
- [ ] Document는 검색 대상으로 식별된 필드를 가지는가?
- [ ] Document 삭제 시 관련 Version·Node·Metadata의 처리 방식이 정의되어 있는가?

---

## Version

- [ ] Version은 Document와 별도의 독립 엔티티로 설계되어 있는가? (Document의 컬럼이 아닌가?)
- [ ] 새로운 Version은 항상 새 레코드 생성인가? (기존 Version 본문 덮어쓰기 구조가 아닌가?)
- [ ] Document는 "현재 Version을 가리키는 포인터"를 가지는가? (Document 자체에 본문이 없는가?)
- [ ] Version은 불변 엔티티로 설계되어 있는가? (생성 후 본문 수정 불가)
- [ ] Version 이력과 Audit 로그가 구분되어 있는가? (같은 저장소에 혼재하지 않는가?)
- [ ] Version은 작성자(createdBy)와 생성 시각(createdAt)을 포함하는가?
- [ ] API에서 특정 버전을 직접 조회할 수 있는 구조인가? (`/documents/{id}/versions/{versionId}`)
- [ ] Version의 범위(Document 전체 vs Node 수준)가 명시적으로 결정되어 있는가?

---

## Node / Structure

- [ ] Node는 Document 내부 텍스트 블록이 아닌 독립 엔티티로 설계되어 있는가?
- [ ] Node는 부모-자식 관계와 순서 정보를 표현할 수 있는가?
- [ ] Node는 특정 문서 포맷(Markdown, HTML 등)에 종속된 구조를 가지고 있지 않은가?
- [ ] 문서 구조(Node 트리)와 Node 본문 내용이 개념적으로 구분되어 있는가?
- [ ] Node는 단독으로 검색·조회 가능한 단위인가?
- [ ] Node 단위 API 접근이 가능한 구조인가? (`/documents/{id}/nodes/{nodeId}`)
- [ ] 트리 표현 기법이 명시적으로 선택되어 있는가? (인접 리스트, 경로 열거 등)
- [ ] Node 변경과 버전 생성의 관계가 명시되어 있는가? (어떤 경우에 새 Version이 생성되는가?)

---

## DocumentType

- [ ] 코어 Document/Version/Node 모델이 특정 DocumentType 없이도 의미가 성립하는가?
- [ ] DocumentType은 코어 위에 올라타는 확장 계층으로 설계되어 있는가?
- [ ] 새 DocumentType 추가 시 코드 변경이 필요하지 않은 구조인가?
- [ ] DocumentType은 해당 유형의 메타데이터 스키마를 정의할 수 있는가?
- [ ] DocumentType은 Admin UI에서만 관리 가능한가? (User UI에서 수정 불가)
- [ ] DocumentType과 Document의 관계가 명확한가? (Document가 하나의 DocumentType을 가지는가?)
- [ ] DocumentType별 상태 모델 차이 허용 여부가 명시되어 있는가?

---

## Metadata

- [ ] 공통 메타데이터와 DocumentType별 확장 메타데이터가 구분되어 있는가?
- [ ] 확장 메타데이터는 완전 자유 필드가 아니라 DocumentType이 정의한 스키마에 따라 검증되는가?
- [ ] 메타데이터 스키마 정의(어떤 필드가 허용되는가)와 메타데이터 값(실제 데이터)이 별도 엔티티로 분리되어 있는가?
- [ ] 검색·필터링에 사용될 메타데이터 필드가 식별되어 있는가?
- [ ] User가 볼 수 있는 메타데이터와 Admin만 볼 수 있는 메타데이터가 구분되어 있는가?
- [ ] AI 생성 메타데이터(자동 추출 키워드 등)가 원본 메타데이터와 분리 관리될 수 있는 구조인가?
- [ ] 메타데이터 변경 시 검색 인덱스 갱신과 연결될 수 있는 구조인가?

---

## Status / Lifecycle

- [ ] Document는 상태(Status)를 가지는가? (단순 삭제 여부 boolean이 아닌가?)
- [ ] 현재 상태값(Status 컬럼)과 상태 전이 이력(별도 이력)이 분리되어 있는가?
- [ ] 상태 전이 규칙이 도메인 서비스 계층에 명시적으로 정의되어 있는가? (각 API 핸들러에 분산되지 않았는가?)
- [ ] Version과 Status가 구분되어 있는가? (상태 변경 ≠ 버전 생성)
- [ ] 상태 전이 시 감사 로그와 연결되는 구조인가?
- [ ] User에게 보이는 상태 정보와 Admin에게 보이는 상태 정보의 차이가 고려되었는가?
- [ ] 상태 기반 API 접근 제어(예: 게시된 문서만 외부 접근 허용)가 가능한 구조인가?
- [ ] 최소한 DRAFT → PUBLISHED의 전이 경로가 정의되어 있는가?

---

## Phase 2 연결 가능성 (최종 검토)

- [ ] 각 코어 엔티티가 API 리소스로 드러날 수 있는 구조인가? (Document, Version, Node, Metadata, DocumentType, Status)
- [ ] 도메인 이벤트(`DocumentCreated`, `VersionCreated`, `StatusChanged` 등)가 정의되어 있거나 정의 가능한 구조인가?
- [ ] 비동기 처리(검색 인덱싱, 감사 로그 기록)가 도메인 레이어와 분리될 수 있는 구조인가?
- [ ] Phase 2에서 서비스 계층이 이 도메인 모델을 기반으로 유스케이스를 구현할 수 있는가?

---

## 예외 처리 기록 템플릿

체크리스트 항목을 통과하지 못하는 경우 아래 형식으로 기록한다.

```
- 미통과 항목: [항목 내용]
- 해당 엔티티: [Document / Version / Node / ...]
- 예외 사유: [왜 이 원칙을 지키기 어려운가]
- 대안 또는 후속 계획: [어떤 방식으로 해결할 것인가]
- 임시 여부: [임시 / 영구]
```
