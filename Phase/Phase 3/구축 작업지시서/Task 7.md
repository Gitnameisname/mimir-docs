Task I-7. documents 핵심 API 구현

Claude Code 작업 계획서

1. 작업 목표

플랫폼 API의 첫 번째 핵심 도메인 리소스인 documents API를 실제로 동작하는 수준까지 구현한다.

이번 단계의 목적은 documents 기능을 전부 완성하는 것이 아니라, 앞서 만든 공통 기반 위에서 문서 리소스의 최소 CRUD/read/list 흐름을 안정적으로 세우는 것이다.

이번 작업으로 확보하려는 것은 다음이다.
	•	document create
	•	document get
	•	document list
	•	document update 기본 구조
	•	문서 상태 필드와 metadata의 최소 반영
	•	공통 response/error/context/auth/list-query 계층의 실제 적용
	•	service 계층 중심의 documents 처리 패턴 확립
	•	이후 versions / nodes / idempotency / audit / retrieval 확장과 충돌하지 않는 documents 리소스 기반 확보

즉, 이번 Task는 플랫폼 API가 실제 문서 리소스를 다루기 시작하는 첫 구현 단계다.

⸻

2. 작업 범위

포함 범위
	•	documents 리소스의 API contract 구체화
	•	create / get / list / update endpoint 구현
	•	request/response schema 정의
	•	documents service 계층 구현
	•	repository 또는 persistence access 경계 연결
	•	status / metadata 최소 반영
	•	공통 response helper 적용
	•	공통 error handling 적용
	•	auth context / authorization hook 연결
	•	list query parser/validator 적용
	•	idempotency 적용 예정 write endpoint 연결 준비

제외 범위
	•	versions / nodes 실제 구현
	•	publish workflow 완성
	•	restore / archive / delete full lifecycle
	•	document permission matrix 완성
	•	document search/full-text retrieval
	•	audit logging 완성
	•	idempotency full replay implementation
	•	concurrency control full implementation
	•	optimistic locking 완성
	•	AI/RAG 전용 endpoint 구현

⸻

3. 구현 의도

이 단계에서 documents API를 대충 만들면 이후 모든 하위 리소스와 운영 구조가 흔들리게 된다.

예를 들어:
	•	라우터에서 DB 직접 접근
	•	create/update마다 서로 다른 payload 구조
	•	metadata를 제각각 허용
	•	status 필드 의미가 불분명
	•	authz/action naming이 일관되지 않음
	•	list 규약이 documents에서만 따로 굴러감

이런 상태가 되면 versions, nodes, workflow, retrieval, audit, idempotency를 붙일 때 큰 재작업이 발생한다.

따라서 이번 작업의 목적은 다음과 같다.
	•	documents 리소스를 플랫폼 규약에 맞게 처음부터 구현
	•	router → service → persistence의 책임 분리 확립
	•	문서 상태와 metadata의 최소 계약 고정
	•	후속 Phase에서 versions / nodes / publish / AI 확장을 자연스럽게 붙일 수 있는 기반 확보

⸻

4. Claude Code에게 요청할 작업

아래 작업을 순서대로 수행하라.

⸻

Step 1. 현재 documents 관련 코드/모델/설계 자산 점검

먼저 현재 코드베이스에서 documents 구현에 활용할 수 있는 자산을 점검하라.

확인할 항목:
	•	기존 Document 관련 모델/엔티티 존재 여부
	•	Phase 1에서 설계한 Document / Version / Node 구조 반영 흔적
	•	DB 모델 또는 ORM 스키마 존재 여부
	•	repository / service / schema 계층이 이미 있는지
	•	placeholder documents router가 현재 어떤 path/contract를 쓰는지
	•	status / metadata 관련 기존 필드가 있는지

목적:
	•	기존 구조를 존중하면서, 최소 변경으로 실제 documents API를 세울 수 있는 구현 경로를 정하는 것

⸻

Step 2. documents API contract 구체화

이번 단계에서 구현할 documents endpoint 계약을 명확히 정리하라.

권장 최소 endpoint:
	•	POST /api/v1/documents
	•	GET /api/v1/documents/{document_id}
	•	GET /api/v1/documents
	•	PATCH /api/v1/documents/{document_id}

선택적으로 검토 가능:
	•	PUT 대신 PATCH만 우선 지원
	•	delete는 이번 단계 제외

권장 action mapping:
	•	document.create
	•	document.read
	•	document.list
	•	document.update

중요:
	•	이번 단계에서는 path와 method를 이후에도 유지 가능한 형태로 고정할 것
	•	action-style endpoint를 만들지 말 것
	•	versions/nodes는 문서 하위 확장으로 남겨두고 documents 자체 contract에 집중할 것

⸻

Step 3. documents request/response schema 정의

create/get/list/update에 필요한 request/response schema를 정의하라.

최소 create request 권장 항목:
	•	title
	•	document_type
	•	status 또는 초기 상태 기본값
	•	metadata (확장 가능 구조)
	•	필요 시 summary 또는 description은 optional

최소 update request 권장 항목:
	•	title optional
	•	status optional
	•	metadata optional
	•	document_type 변경 허용 여부는 신중히 검토
	•	immutable 필드는 명확히 구분

최소 response 권장 항목:
	•	id
	•	title
	•	document_type
	•	status
	•	metadata
	•	created_at
	•	updated_at
	•	created_by 또는 최소 actor reference placeholder 가능
	•	retrieval/citation-friendly identifier 확장 가능 여지

중요:
	•	response는 Task I-2의 공통 envelope 위에 올라가야 한다
	•	schema는 documents 전용이지만, naming과 확장성은 플랫폼 기준에 맞춰야 한다
	•	versions/nodes 세부 정보는 아직 과도하게 넣지 말 것

⸻

Step 4. document 상태 모델 최소 반영

문서 상태 필드의 최소 정책을 정하고 API에 반영하라.

권장 최소 상태:
	•	draft
	•	published는 optional
	•	또는 이번 단계에서는 draft 중심 + 확장 가능 enum 구조

검토 포인트:
	•	create 시 기본 상태를 draft로 둘지
	•	update에서 status 변경을 허용할지
	•	publish workflow가 아직 없으므로 published 상태를 지금 열어둘지 제한할지

권장 방향:
	•	이번 단계는 workflow 완성이 아니므로,
상태 모델은 최소 enum + 안전한 기본값 중심으로 시작하라.

중요:
	•	status 필드는 이후 Draft / Published / Archived / Review 등으로 확장 가능해야 한다
	•	지금 단계에서 workflow semantics를 과도하게 박지 말 것

⸻

Step 5. metadata 처리 정책 정리

documents의 metadata 필드를 어떻게 받을지 최소 정책을 정하라.

권장 방향:
	•	metadata는 key-value 확장 구조
	•	너무 자유로운 dump가 되지 않도록 타입/크기/형식 최소 검증 검토
	•	null / 빈 dict 허용 여부 정책 정리

주의:
	•	이번 단계에서는 metadata schema를 문서 유형별로 복잡하게 강제하지 말 것
	•	하지만 완전 무제한 raw object로 둘 경우 운영상 문제가 없는지 검토하라
	•	최소한 API contract 상 “확장용 metadata”라는 의미가 분명해야 한다

⸻

Step 6. documents service 계층 구현

라우터에 비즈니스 로직을 넣지 말고, documents 전용 service 계층을 구현하라.

최소 서비스 메서드 예:
	•	create_document(...)
	•	get_document(...)
	•	list_documents(...)
	•	update_document(...)

서비스 계층 책임:
	•	입력 검증 후 도메인/persistence 호출
	•	존재 여부 확인
	•	상태/metadata 처리
	•	authorization action/resource 흐름과 연결 가능한 지점 유지
	•	response DTO에 맞는 결과 반환

중요:
	•	service는 raw FastAPI Request에 직접 의존하지 말 것
	•	필요하면 RequestContext나 정규화된 actor/resource context만 받도록 설계하라

⸻

Step 7. repository/persistence access 경계 연결

documents 데이터를 실제 저장/조회할 수 있도록 persistence 계층을 연결하라.

검토 방향:
	•	이미 repository 패턴이 있다면 그것을 따를 것
	•	없으면 최소한 service와 DB 접근을 분리하는 경계를 만들 것

최소 repository 예:
	•	create
	•	get_by_id
	•	list
	•	update

중요:
	•	router에서 ORM session/query를 직접 다루지 말 것
	•	list query parser 결과를 persistence layer가 소비 가능한 형태로 넘길 수 있어야 한다
	•	이후 versions/nodes join이나 tenant scope, audit logging을 붙여도 구조가 유지되어야 한다

⸻

Step 8. create endpoint 구현

POST /api/v1/documents를 실제 동작하게 구현하라.

최소 요구:
	•	request body validation
	•	actor extraction 연결
	•	authorization hook 호출
	•	create service 호출
	•	공통 success response envelope 반환
	•	Task I-9에서 idempotency hook을 붙일 수 있게 write 흐름을 단순하고 안정적으로 유지

검토 사항:
	•	create 시 기본 status 적용
	•	metadata 저장
	•	created_by / actor trace 저장 가능 여부
	•	tenant context가 있으면 future scope 연결 여지 확보

주의:
	•	이번 단계에서 idempotency를 완성하지 말 것
	•	그러나 POST create가 idempotency 후보 endpoint라는 점이 코드 구조상 보이게 할 것

⸻

Step 9. get endpoint 구현

GET /api/v1/documents/{document_id}를 구현하라.

최소 요구:
	•	path parameter 검증
	•	actor extraction 연결
	•	authorization hook 호출
	•	존재하지 않으면 not found
	•	공통 response envelope 반환

중요:
	•	response에는 이후 citation-friendly 식별자나 structure-aware read로 확장 가능한 여지를 남길 것
	•	versions/nodes를 지금 과도하게 embed하지 말 것
	•	not found와 permission denied 흐름이 Task I-3 오류 체계와 자연스럽게 이어지게 할 것

⸻

Step 10. list endpoint 구현

GET /api/v1/documents를 구현하라.

반드시 연결할 것:
	•	Task I-6 공통 list query parser/validator
	•	documents 전용 allowed sort/filter spec
	•	authorization hook
	•	공통 list response helper
	•	meta.pagination 기반

권장 기본 allowed sort:
	•	created_at
	•	updated_at
	•	title
	•	status

권장 기본 allowed filter:
	•	status
	•	document_type
	•	created_by 또는 owner 계열이 있으면 검토

중요:
	•	list endpoint는 documents API의 규약성을 보여주는 대표 엔드포인트다
	•	list query parsing이 라우터 안에서 흩어지지 않게 할 것
	•	아직 복잡한 검색 기능으로 확장하지 말 것

⸻

Step 11. update endpoint 구현

PATCH /api/v1/documents/{document_id}를 구현하라.

최소 요구:
	•	partial update request schema
	•	actor extraction 연결
	•	authorization hook 호출
	•	존재 여부 확인
	•	mutable 필드만 업데이트
	•	공통 success response envelope 반환

검토 포인트:
	•	title 수정 허용
	•	metadata 부분 수정 정책
	•	전체 replace
	•	shallow merge
	•	명시적 replace only
	•	status 변경 허용 범위
	•	immutable 필드 보호

권장:
	•	이번 단계에서는 update semantics를 지나치게 복잡하게 만들지 말고,
예측 가능한 partial update 규칙을 정하라.

중요:
	•	versions가 아직 없는 상태에서 document 본문 구조 전체를 update로 다루려 하지 말 것
	•	현재는 문서 리소스 메타 수준의 update 중심으로 시작하라

⸻

Step 12. authorization/action/resource 흐름 실제 적용

documents API 각 endpoint에 대해 action/resource 기준 authorization hook을 실제로 적용하라.

예:
	•	create → document.create
	•	get → document.read
	•	list → document.list
	•	update → document.update

resource reference 권장 예:
	•	resource_type = document
	•	resource_id = {document_id} optional
	•	tenant_id optional
	•	document_type optional if useful

중요:
	•	라우터에 권한 if문을 넣지 말 것
	•	documents API가 Task I-5 패턴을 실제로 쓰는 첫 대표 사례가 되어야 한다

⸻

Step 13. 공통 response/error/context 계층 통합

documents API가 앞선 Task들을 실제로 활용하도록 연결하라.

반드시 확인할 것:
	•	success response는 Task I-2 envelope 사용
	•	validation/auth/not found/conflict는 Task I-3 공통 오류 응답 사용
	•	request_id/trace_id는 Task I-4 context에서 연결
	•	actor는 Task I-5 dependency 통해 반영
	•	list query는 Task I-6 parser 사용

즉, documents API는 공통 기반이 제대로 작동하는지 검증하는 첫 실전 통합 지점이어야 한다.

⸻

Step 14. idempotency 적용 준비 지점 정리

Task I-9에서 POST /documents 같은 write endpoint에 idempotency hook을 붙일 수 있도록 준비하라.

최소 요구:
	•	create/update 흐름이 request body와 actor/resource 맥락을 안정적으로 해석하는 구조일 것
	•	라우터/서비스에서 idempotency hook을 삽입할 위치가 명확할 것
	•	TODO 또는 인터페이스 수준의 연결 지점을 남길 것

중요:
	•	지금 idempotency 저장소/재생 logic까지 구현하지 말 것
	•	다만 write 흐름이 “나중에 붙이기 쉬운 형태”여야 한다

⸻

Step 15. 문서 도메인과 versions/nodes 확장성 검토

이번 단계는 documents만 구현하지만, 이후 Task I-8과 충돌하지 않도록 아래를 검토하라.

확인할 점:
	•	document response가 versions summary를 강제로 품고 있지 않은가
	•	update가 실제로는 version create가 되어야 할 영역까지 침범하지 않는가
	•	document와 version/node의 책임 경계가 모호해지지 않았는가
	•	document_type / metadata / status가 버전성 확장을 막지 않는가

⸻

Step 16. OpenAPI/계약 관점 검토

Swagger/OpenAPI에서 documents API가 플랫폼 API답게 보이는지 검토하라.

확인할 점:
	•	create/get/list/update의 request/response schema가 일관적인가
	•	response envelope가 공통 형식으로 보이는가
	•	list query가 표준화되어 보이는가
	•	public/protected endpoint 성격이 자연스러운가
	•	이후 versions / nodes 추가 시 문서 구조가 유지될 것 같은가

⸻

5. 구현 시 세부 요구사항

5-1. router는 얇게, service는 중심에

이번 단계에서 documents API는 실제 구현이 들어가므로 특히 책임 분리가 중요하다.

라우터의 역할:
	•	request parsing
	•	dependency injection
	•	authz hook 연결
	•	service 호출
	•	response shaping

서비스의 역할:
	•	비즈니스 흐름 제어
	•	상태/metadata 처리
	•	persistence 호출 조합
	•	도메인 검증

DB 접근은 가능한 한 repository/persistence 계층으로 분리하라.

⸻

5-2. create/get/list/update contract는 오래 갈 수 있어야 함

이번 단계의 endpoint shape는 이후에도 유지 가능한 형태여야 한다.

권장:
	•	POST /documents
	•	GET /documents/{id}
	•	GET /documents
	•	PATCH /documents/{id}

비권장:
	•	POST /documents/create
	•	POST /documents/{id}/update
	•	action-heavy path

⸻

5-3. metadata는 확장 가능하되 무질서하지 않게

metadata는 문서 유형별 확장을 위해 중요하지만, 지금부터 완전 자유형으로 두면 나중에 제어가 어렵다.

따라서:
	•	구조는 열어두되
	•	최소 타입 검증
	•	지나치게 큰/복잡한 payload 제한 가능성 검토
	•	문서화는 명확히

⸻

5-4. update semantics는 단순하고 예측 가능하게

이번 단계에서는 PATCH semantics를 과도하게 복잡하게 만들지 말 것.

권장:
	•	명시된 필드만 수정
	•	immutable 필드는 거부
	•	metadata는 replace 또는 shallow merge 중 하나를 명확히 고정

핵심은 “예측 가능성”이다.

⸻

5-5. not found / permission denied / validation 흐름 구분

documents API는 사용자 체감이 큰 리소스이므로, 오류 흐름이 명확해야 한다.

반드시 구분:
	•	문서가 없음 → not found
	•	인증 필요 → authentication required
	•	권한 없음 → permission denied
	•	입력 잘못됨 → validation error

이 흐름이 Task I-3 공통 오류 모델을 통해 일관되게 나와야 한다.

⸻

5-6. list 규약은 documents가 첫 실전 적용 사례

documents list는 Task I-6 규약의 대표 사례다.

따라서:
	•	page/page_size
	•	sort
	•	filter
	•	pagination meta

가 실제로 잘 적용되는지 확인해야 한다.

documents에서 ad-hoc 예외를 두면 이후 다른 리소스가 다 흐트러진다.

⸻

5-7. created_by / actor trace는 최소 연결만 고려

감사 로그를 지금 완성할 필요는 없지만, create/update 시 actor trace를 남길 자리를 검토하라.

예:
	•	created_by
	•	updated_by
	•	actor_id placeholder

다만:
	•	audit 시스템을 지금 다 구현하려 하지 말 것
	•	최소 연결 가능성만 확보하면 된다

⸻

6. 권장 설계 방향

Claude Code는 아래 방향을 우선 검토하라.

옵션 A. Router + Service + Repository
	•	documents.py router
	•	documents_service.py
	•	documents_repository.py

장점:
	•	책임 분리가 명확
	•	이후 versions/nodes/audit/idempotency 확장에 유리
	•	플랫폼 API 구조와 잘 맞음

옵션 B. Router + Service, repository는 보류

장점:
	•	현재 코드베이스가 단순하면 빠르게 적용 가능
단점:
	•	service에 persistence 세부가 과도하게 들어갈 수 있음

이번 단계에서는 가능하면 service와 persistence 경계를 분리하는 방향을 우선 검토하라.
다만 현재 구조상 repository가 과하면, 최소한 라우터와 service는 반드시 분리하라.

⸻

7. 산출물 요구

Claude Code는 작업 후 아래 내용을 보고하라.

A. 생성/수정 파일 목록

예:
	•	app/api/v1/documents.py
	•	app/services/documents_service.py
	•	app/repositories/documents_repository.py
	•	app/schemas/documents.py
	•	app/models/document.py
	•	관련 dependency / auth / response 파일

B. 구현한 endpoint 요약

예:
	•	POST /api/v1/documents
	•	GET /api/v1/documents/{document_id}
	•	GET /api/v1/documents
	•	PATCH /api/v1/documents/{document_id}

C. request/response schema 요약

예:
	•	create request 필드
	•	update request 필드
	•	document response 필드
	•	status / metadata 처리 방식

D. authz/list/common-layer 적용 요약

간단히:
	•	어떤 action naming을 사용했는지
	•	documents list에서 어떤 sort/filter spec을 허용했는지
	•	response/error/context가 어떻게 연결됐는지

E. 설계 판단 근거

짧게 정리:
	•	왜 PATCH를 선택했는지
	•	왜 status를 이 수준으로만 열어뒀는지
	•	왜 metadata를 이런 방식으로 처리했는지
	•	왜 versions/nodes를 아직 embed하지 않았는지

F. 남겨둔 TODO

예:
	•	Task I-8에서 versions/nodes 연결 예정
	•	Task I-9에서 create write idempotency hook 연결 예정
	•	created_by/updated_by audit 강화 예정
	•	document body/version separation 고도화 예정

⸻

8. 완료 조건

아래를 만족하면 완료로 본다.
	•	documents create/get/list/update endpoint가 동작한다.
	•	request/response schema가 정의되어 있다.
	•	공통 response/error/context/auth/list-query 계층이 실제로 적용되어 있다.
	•	documents service 계층이 존재한다.
	•	persistence access 경계가 정리되어 있다.
	•	status와 metadata의 최소 정책이 반영되어 있다.
	•	create/update가 이후 idempotency와 audit 확장을 막지 않는 구조다.
	•	versions/nodes 확장과 충돌하지 않는 documents 책임 경계가 유지된다.

⸻

9. 금지 사항

이번 작업에서는 다음을 하지 마라.
	•	versions/nodes 구현까지 한 번에 끌고 가지 말 것
	•	publish workflow를 과도하게 완성하려 하지 말 것
	•	라우터에서 DB를 직접 조작하지 말 것
	•	documents 전용 예외/응답 규약을 따로 만들지 말 것
	•	metadata를 완전 무제한 덤프로 방치하지 말 것
	•	update에서 versioning이 필요한 영역까지 섣불리 덮어쓰지 말 것
	•	idempotency 전체를 이번 단계에서 미리 구현하려 하지 말 것

⸻

10. Claude Code 최종 지시문

아래 지시문을 그대로 사용해도 된다.

Task I-7을 수행하라.

목표:
- 플랫폼 API의 첫 번째 핵심 리소스인 documents API를 실제 동작 수준으로 구현한다.
- create/get/list/update의 최소 문서 리소스 흐름을 구현한다.
- 앞서 만든 공통 response/error/context/auth/list-query 계층을 documents API에 실제로 적용한다.
- 이후 versions/nodes/idempotency/audit/retrieval 확장을 막지 않는 구조로 documents 책임 경계를 세운다.

반드시 지킬 원칙:
- 이번 단계는 documents 기능을 전부 완성하는 것이 아니라 핵심 리소스 API의 최소 구현을 세우는 작업이다.
- router는 얇게 유지하고, 비즈니스 로직은 service 계층으로 분리하라.
- persistence 접근도 가능한 한 service와 분리하라.
- documents API가 공통 플랫폼 규약의 첫 실전 적용 사례가 되게 하라.
- versions/nodes/workflow/idempotency를 과도하게 선구현하지 말고, 붙일 수 있는 구조만 남겨라.

수행 항목:
1. 현재 documents 관련 모델/구조를 점검한다.
2. create/get/list/update API contract를 구체화한다.
3. request/response schema를 정의한다.
4. status와 metadata의 최소 정책을 정한다.
5. documents service 계층을 구현한다.
6. repository 또는 persistence access 경계를 연결한다.
7. POST /documents, GET /documents/{id}, GET /documents, PATCH /documents/{id}를 구현한다.
8. authorization action/resource 흐름을 적용한다.
9. 공통 response/error/context/list-query 계층을 실제로 연결한다.
10. Task I-9 idempotency와 Task I-8 versions/nodes 확장을 막지 않는지 검토한다.

산출물:
- 생성/수정 파일 목록
- 구현 endpoint 요약
- request/response schema 요약
- authz/list/common-layer 적용 요약
- 설계 판단 근거
- 남겨둔 TODO 목록