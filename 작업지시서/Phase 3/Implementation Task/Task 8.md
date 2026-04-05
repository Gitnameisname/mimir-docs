Task I-8. versions / nodes API 구현

Claude Code 작업 계획서

1. 작업 목표

문서의 버전성(versioning) 과 구조성(structured document tree) 을 실제 API로 노출하기 위해, versions 및 nodes 하위 리소스 API를 구현한다.

이번 단계의 목적은 모든 버전 관리 기능을 완성하는 것이 아니라, 앞서 구현한 documents API 위에 다음 핵심 흐름을 안정적으로 세우는 것이다.
	•	문서별 버전 목록 조회
	•	문서에 새 버전 생성
	•	특정 버전 조회
	•	특정 버전의 노드 목록/조회
	•	document → version → node 관계 검증
	•	canonical subresource path 확립
	•	AI/RAG가 활용 가능한 구조적 조회 기반 확보

즉, 이번 Task는 범용 문서 도메인 모델의 구조성과 버전성을 실제 API 계층으로 드러내는 첫 구현 단계다.

⸻

2. 작업 범위

포함 범위
	•	versions API contract 구체화
	•	nodes API contract 구체화
	•	version list / create / get 구현
	•	node list / get 구현
	•	version / node request-response schema 정의
	•	document-version-node 관계 검증
	•	service 계층 구현
	•	persistence 접근 경계 연결
	•	공통 response / error / context / auth / list-query 계층 적용
	•	citation-friendly / structure-aware read 확장 가능성 확보

제외 범위
	•	full restore / rollback 기능
	•	full diff API
	•	version compare API
	•	node edit / node patch / tree mutation full 구현
	•	publish workflow 완성
	•	merge/conflict resolution
	•	collaborative editing
	•	retrieval/search/indexing full 구현
	•	AI 전용 별도 endpoint 본격 구현

⸻

3. 구현 의도

documents API만 있고 version/node API가 없으면, 플랫폼은 결국 “문서 메타데이터 CRUD” 수준에 머물게 된다.
반대로 versions/nodes를 성급하게 구현하면 다음 문제가 생긴다.
	•	document와 version의 책임 경계가 붕괴
	•	node가 document에 직접 붙어 구조가 꼬임
	•	path 설계가 일관되지 않음
	•	관계 검증이 빠져 잘못된 리소스를 읽게 됨
	•	AI/RAG 확장을 위한 구조적 식별자가 불안정해짐

따라서 이번 작업의 목적은 다음과 같다.
	•	문서의 상태/식별 책임은 document
	•	문서 내용의 특정 시점 스냅샷 책임은 version
	•	실제 구조적 본문 단위 책임은 node

이 3계층을 API 수준에서 명확히 드러내는 것이다.

⸻

4. Claude Code에게 요청할 작업

아래 작업을 순서대로 수행하라.

⸻

Step 1. 기존 도메인 모델 및 documents 구현 상태 점검

먼저 현재 코드베이스에서 versions/nodes 구현에 활용할 수 있는 구조를 점검하라.

확인할 항목:
	•	Phase 1에서 설계한 Document / Version / Node 모델 반영 흔적
	•	Task I-7에서 만든 documents 모델/서비스/리포지토리 구조
	•	version/node 관련 ORM 또는 schema 존재 여부
	•	document와 version/node의 FK 또는 관계 설계 상태
	•	placeholder versions/nodes router가 현재 어떤 path를 갖는지
	•	node tree 표현 방식(부모-자식, 순서, type 등) 설계 흔적 존재 여부

목적:
	•	기존 구조를 존중하면서 최소 변경으로 API를 현실적으로 세우기 위함이다.

⸻

Step 2. versions / nodes API contract 구체화

이번 단계에서 구현할 endpoint 계약을 명확히 정리하라.

권장 최소 endpoint:

Versions
	•	GET /api/v1/documents/{document_id}/versions
	•	POST /api/v1/documents/{document_id}/versions
	•	GET /api/v1/versions/{version_id}

Nodes
	•	GET /api/v1/versions/{version_id}/nodes
	•	GET /api/v1/versions/{version_id}/nodes/{node_id}

선택적으로 검토 가능:
	•	GET /api/v1/documents/{document_id}/versions/{version_id} 같은 canonical alias 여부
	•	하지만 이번 단계에서는 지나친 path 중복은 피하라

권장 action mapping:
	•	version.list
	•	version.create
	•	version.read
	•	node.list
	•	node.read

중요:
	•	path와 method는 이후에도 유지 가능한 형태로 고정할 것
	•	action-style endpoint를 만들지 말 것
	•	versions와 nodes 모두 리소스 중심 설계를 유지할 것

⸻

Step 3. canonical / subresource path 정책 정리

versions와 nodes의 canonical path를 정리하라.

권장 방향:
	•	버전 생성/목록은 문서의 하위 리소스
	•	/documents/{document_id}/versions
	•	특정 버전 조회는 버전 자체 리소스
	•	/versions/{version_id}
	•	노드 조회는 버전의 하위 리소스
	•	/versions/{version_id}/nodes
	•	/versions/{version_id}/nodes/{node_id}

이 구조의 장점:
	•	문서 기준으로 버전 생성 흐름이 자연스럽다
	•	버전/노드 개별 조회도 독립 식별이 가능하다
	•	AI/RAG나 citation 시 version/node를 직접 가리키기 쉽다

주의:
	•	document_id와 version_id를 동시에 path에 넣는 중복 구조를 남발하지 말 것
	•	단, 권한/검증상 필요한 경우 내부적으로는 document-version 관계 검증을 수행해야 한다

⸻

Step 4. version request/response schema 정의

version 생성/조회/목록에 필요한 schema를 정의하라.

version create request 권장 항목
	•	label optional
	•	message 또는 change_summary optional
	•	source optional
	•	예: manual, system, import
	•	nodes 또는 version 구조 입력 payload
	•	필요 시 metadata optional

version response 권장 항목
	•	id
	•	document_id
	•	version_number 또는 동등 개념
	•	label
	•	status optional
	•	change_summary
	•	created_at
	•	created_by
	•	metadata optional

중요:
	•	create 시 version body/structure 입력을 어떻게 받을지 너무 과하게 복잡하게 만들지 말 것
	•	지금 단계에서는 “버전 생성 시 구조 스냅샷을 받는 흐름” 정도면 충분하다
	•	version_number 정책은 persistence 구조에 맞게 단순하고 예측 가능해야 한다

⸻

Step 5. node request/response schema 정의

node 목록/조회에 필요한 schema를 정의하라.

권장 최소 node response 항목:
	•	id
	•	version_id
	•	parent_id optional
	•	node_type
	•	order_index 또는 동등한 정렬 필드
	•	title optional
	•	content optional
	•	text 또는 body fragment
	•	metadata optional
	•	citation-friendly identifier 확장 가능 필드

선택적으로 검토 가능:
	•	path
	•	depth
	•	children_count
	•	ref
	•	anchor

중요:
	•	node는 이후 트리 렌더링과 AI citation에 모두 쓰일 수 있다
	•	그러나 지금 단계에서 너무 무거운 tree serialization까지 가지 말 것
	•	list 응답은 flat list 기반으로 시작해도 괜찮다

⸻

Step 6. versioning 정책 최소 정의

새 버전 생성 시 최소 versioning 정책을 정하라.

검토 포인트:
	•	version_number를 자동 증가시킬지
	•	초판을 1로 시작할지
	•	동일 문서 내에서만 유일하면 되는지
	•	새 버전 생성 시 문서 상태와 어떤 관계를 맺는지

권장 방향:
	•	문서별 순차 증가 정수 버전
	•	새 버전 생성은 해당 문서의 구조 스냅샷을 고정하는 행위
	•	publish/approval semantics는 아직 분리

중요:
	•	versioning 정책은 단순하고 예측 가능해야 한다
	•	나중에 workflow와 연결해도 깨지지 않아야 한다

⸻

Step 7. node tree 표현 정책 최소 정의

nodes를 어떤 방식으로 표현할지 최소 정책을 정하라.

권장 시작점:
	•	flat list + parent_id + order_index
	•	node_type으로 paragraph / heading / section 등 확장 가능
	•	필요 시 depth는 파생값으로 둘 수 있음

이 방식의 장점:
	•	저장과 조회가 단순하다
	•	UI 트리 렌더링 가능
	•	AI/RAG도 개별 노드 단위 참조 가능

주의:
	•	이번 단계에서는 full recursive nested tree JSON을 기본 응답으로 강제하지 말 것
	•	필요하면 후속 Task에서 tree projection helper를 추가할 수 있게 여지를 남겨라

⸻

Step 8. document-version-node 관계 검증 구현

API에서 관계 검증을 반드시 넣어라.

최소 검증 항목:
	•	version 생성 시 대상 document 존재 여부
	•	version 조회 시 version 존재 여부
	•	node 조회 시 version 존재 여부
	•	node 조회 시 해당 node가 해당 version에 속하는지
	•	필요 시 version이 document에 속하는지 내부 검증 가능

중요:
	•	잘못된 조합으로 다른 문서의 version/node를 읽지 않게 해야 한다
	•	관계 검증 실패는 not found 또는 적절한 permission 흐름으로 연결되어야 한다
	•	관계 무결성은 AI/RAG citation 신뢰성에도 직결된다

⸻

Step 9. versions service 계층 구현

versions 전용 service 계층을 구현하라.

최소 메서드 예:
	•	list_versions(document_id, ...)
	•	create_version(document_id, payload, request_context, ...)
	•	get_version(version_id, ...)

서비스 책임:
	•	문서 존재 확인
	•	authorization action/resource 흐름 연결 가능 지점 유지
	•	version number 계산/부여
	•	nodes snapshot 생성/저장 흐름 조합
	•	response DTO 반환

중요:
	•	라우터에서 version 생성 로직이나 node 저장 로직을 직접 구현하지 말 것
	•	service는 raw Request에 의존하지 말 것

⸻

Step 10. nodes service 계층 구현

nodes 조회 전용 service 계층을 구현하라.

최소 메서드 예:
	•	list_nodes(version_id, ...)
	•	get_node(version_id, node_id, ...)

서비스 책임:
	•	version 존재 확인
	•	node-version 관계 검증
	•	node ordering / flat structure 반환
	•	필요 시 basic structure-aware read helper와 연결 가능성 확보

중요:
	•	node 수정 기능은 이번 단계 범위 밖이다
	•	지금은 조회 중심(read model)으로 가는 것이 맞다

⸻

Step 11. persistence/repository 경계 연결

versions/nodes persistence 접근 경계를 정리하라.

권장 최소 repository 예:
	•	versions_repository
	•	list_by_document_id
	•	get_by_id
	•	get_latest_version_number
	•	create
	•	nodes_repository
	•	bulk_create_for_version
	•	list_by_version_id
	•	get_by_id_and_version_id

중요:
	•	version 생성 시 node snapshot 저장이 필요하므로 transaction boundary를 검토하라
	•	라우터에서 ORM session/query를 직접 다루지 말 것
	•	나중에 diff, restore, audit, ingestion을 붙일 수 있도록 service와 persistence 경계를 유지하라

⸻

Step 12. version list endpoint 구현

GET /api/v1/documents/{document_id}/versions를 구현하라.

반드시 연결할 것:
	•	문서 존재 검증
	•	actor extraction
	•	authorization hook
	•	공통 list query parser/validator
	•	공통 list response helper

권장 allowed sort:
	•	created_at
	•	version_number

권장 allowed filter:
	•	created_by
	•	source
	•	status optional

중요:
	•	version 목록은 문서의 이력성을 드러내는 API다
	•	list query 규약이 documents list와 같은 방식으로 적용되어야 한다

⸻

Step 13. version create endpoint 구현

POST /api/v1/documents/{document_id}/versions를 구현하라.

최소 요구:
	•	문서 존재 검증
	•	actor extraction
	•	authorization hook
	•	request body validation
	•	새 version_number 부여
	•	version row 생성
	•	node snapshot 저장
	•	공통 success response envelope 반환

검토 포인트:
	•	nodes payload를 version create와 함께 받을지
	•	빈 version 허용 여부
	•	created_by / actor trace 최소 연결
	•	이후 idempotency hook과 충돌하지 않는 write 구조인지

중요:
	•	지금은 version create를 “문서 구조의 새 스냅샷 생성”으로 다루면 된다
	•	diff 계산, publish semantics, restore semantics까지 끌고 가지 말 것

⸻

Step 14. version get endpoint 구현

GET /api/v1/versions/{version_id}를 구현하라.

최소 요구:
	•	version 존재 검증
	•	authorization hook
	•	version metadata 반환
	•	필요 시 document_id 포함
	•	node 전체를 기본적으로 embed할지 여부 신중 검토

권장:
	•	기본 version get은 버전 메타 중심
	•	node 전체는 별도 nodes list endpoint로 조회

이유:
	•	큰 구조 문서에서 응답이 과도하게 무거워지는 것을 피함
	•	read contract를 더 안정적으로 유지할 수 있음

⸻

Step 15. nodes list endpoint 구현

GET /api/v1/versions/{version_id}/nodes를 구현하라.

최소 요구:
	•	version 존재 검증
	•	authorization hook
	•	flat list 또는 structure-aware projection 반환
	•	공통 list response helper 적용 가능 여부 검토
	•	node ordering 보장

검토 포인트:
	•	기본은 전체 flat list
	•	추후 view=tree 같은 projection 확장 여지
	•	AI/RAG에서 chunking/citation-friendly 활용 가능성

중요:
	•	지금 단계에서 tree rendering용 중첩 JSON을 강제하지 말 것
	•	flat but structured 형태가 초기엔 더 안정적이다

⸻

Step 16. node get endpoint 구현

GET /api/v1/versions/{version_id}/nodes/{node_id}를 구현하라.

최소 요구:
	•	version 존재 검증
	•	node-version 관계 검증
	•	authorization hook
	•	단건 node response 반환

권장 추가 필드 검토:
	•	parent_id
	•	order_index
	•	anchor/path
	•	citation reference

중요:
	•	node 단건 조회는 AI/RAG 및 deep-linking의 기초가 될 수 있다
	•	구조 식별자가 일관되게 유지되도록 하라

⸻

Step 17. authz/action/resource 흐름 실제 적용

versions/nodes API에 Task I-5 패턴을 실제 적용하라.

권장 action:
	•	version.list
	•	version.create
	•	version.read
	•	node.list
	•	node.read

권장 resource reference:
	•	resource_type = version / node
	•	resource_id
	•	parent_document_id optional
	•	version_id optional
	•	tenant_id optional

중요:
	•	라우터에 권한 if문을 넣지 말 것
	•	document-version-node 관계와 authorization 흐름이 자연스럽게 이어져야 한다

⸻

Step 18. 공통 response / error / context / list-query 계층 통합

versions/nodes API가 앞선 Task들의 공통 기반을 실제로 쓰도록 연결하라.

반드시 확인:
	•	성공 응답은 Task I-2 envelope 사용
	•	관계 검증/validation/not found/auth 오류는 Task I-3 공통 오류 응답 사용
	•	request_id/trace_id는 Task I-4 context에서 연결
	•	actor는 Task I-5 dependency 통해 반영
	•	list endpoint는 Task I-6 query parser 사용

즉, versions/nodes API도 documents와 동일한 플랫폼 규약 위에 있어야 한다.

⸻

Step 19. AI/RAG 확장성 관점 검토

이번 단계의 versions/nodes API는 이후 Task I-12의 AI/RAG read-model 준비와 직접 연결된다.
따라서 아래를 검토하라.

확인할 점:
	•	node에 citation-friendly identifier를 붙일 수 있는가
	•	version/node response가 구조적 참조에 충분한가
	•	AI 전용 우회 API 없이 공통 리소스를 재사용할 수 있을까
	•	노드 응답이 후속 indexing / ingestion status 확장을 막지 않는가

권장:
	•	id 외에도 향후 참조용 ref, anchor, path 류 확장 슬롯을 염두에 둘 것
	•	지금 당장 모두 구현할 필요는 없다

⸻

Step 20. documents와의 책임 경계 재검토

Task I-7의 documents API와 이번 versions/nodes API 사이의 책임 경계를 다시 점검하라.

반드시 확인:
	•	documents update가 실제로 version create가 되어야 할 영역을 침범하지 않는가
	•	version get이 documents 메타데이터를 과도하게 중복하지 않는가
	•	nodes가 document 소속처럼 직접 노출되지 않는가
	•	3계층 역할이 명확한가

권장 정리:
	•	document = 리소스 정체성/상태/메타
	•	version = 시점 스냅샷
	•	node = 구조 단위

⸻

Step 21. OpenAPI/계약 관점 검토

Swagger/OpenAPI에서 versions/nodes API가 플랫폼스럽게 보이는지 검토하라.

확인할 점:
	•	/documents/{document_id}/versions
	•	/versions/{version_id}
	•	/versions/{version_id}/nodes
	•	/versions/{version_id}/nodes/{node_id}

이 구조가 일관되게 보이는가
응답 envelope가 공통 형식인가
list query 규약이 재사용되는 것이 보이는가
후속 diff/restore/retrieval 확장을 붙여도 path 구조가 유지될 것 같은가

⸻

5. 구현 시 세부 요구사항

5-1. document / version / node 책임을 섞지 말 것

이번 단계의 최우선 원칙이다.
	•	document: 정체성, 상태, 메타데이터
	•	version: 스냅샷, 생성 시점, 변경 요약
	•	node: 실제 구조 단위

이 셋을 응답과 서비스 로직에서 섞지 말 것.

⸻

5-2. version create는 스냅샷 생성 행위로 볼 것

이번 단계에서 version create는 “변경 diff 계산”보다 “새 구조 스냅샷 저장”에 가깝게 접근하라.

이 방식의 장점:
	•	구현 단순성
	•	RAG/AI read model과 정합성
	•	복원/비교 기능을 후속 단계로 미루기 쉬움

⸻

5-3. node 응답은 flat structured 방식 우선

처음부터 nested recursive tree JSON을 기본 응답으로 강제하지 말 것.

권장:
	•	flat list
	•	parent_id
	•	order_index
	•	node_type

이 구조면 UI, API, AI 모두에 무난하다.

⸻

5-4. version get과 nodes list의 책임 분리

GET /versions/{id} 가 모든 node를 항상 embed하면 응답이 과도해질 수 있다.

권장:
	•	version get = 버전 메타 중심
	•	nodes list = 구조 본문 중심

필요하면 나중에 projection 옵션을 추가할 수 있다.

⸻

5-5. relationship 검증은 반드시 넣을 것

이번 단계에서 관계 검증을 빼면 잘못된 경로로 다른 버전/노드를 읽는 위험이 생긴다.

반드시 검증:
	•	document ↔ version
	•	version ↔ node

⸻

5-6. 이후 idempotency와 충돌하지 않게 write 구조 유지

POST /documents/{document_id}/versions 역시 idempotency 후보 write endpoint다.

이번 단계에서는 full idempotency를 구현하지 않더라도,
	•	request body
	•	actor/resource context
	•	write service 진입점

이 분명히 보이게 구조를 유지하라.

⸻

5-7. AI/RAG를 위한 읽기 친화성 유지

이번 단계는 AI 기능 구현이 아니지만,
응답 구조가 향후 구조적 읽기와 citation을 방해하면 안 된다.

예:
	•	stable node id
	•	version id
	•	document id
	•	parent-child 관계
	•	order 정보

이 정도는 안정적으로 유지되어야 한다.

⸻

6. 권장 설계 방향

Claude Code는 아래 방향을 우선 검토하라.

옵션 A. documents와 대칭적인 Router + Service + Repository
	•	v1/versions.py
	•	v1/nodes.py
	•	versions_service.py
	•	nodes_service.py
	•	versions_repository.py
	•	nodes_repository.py

장점:
	•	documents와 일관된 구조
	•	책임 분리 명확
	•	이후 확장(diff/restore/retrieval) 용이

옵션 B. version service 중심 + node read helper 분리

장점:
	•	초기에 구조가 단순할 수 있음
단점:
	•	node 책임이 version 서비스에 과적재될 수 있음

가능하면 versions와 nodes를 각각 독립 책임으로 유지하는 방향을 우선 검토하라.

⸻

7. 산출물 요구

Claude Code는 작업 후 아래 내용을 보고하라.

A. 생성/수정 파일 목록

예:
	•	app/api/v1/versions.py
	•	app/api/v1/nodes.py
	•	app/services/versions_service.py
	•	app/services/nodes_service.py
	•	app/repositories/versions_repository.py
	•	app/repositories/nodes_repository.py
	•	app/schemas/versions.py
	•	app/schemas/nodes.py

B. 구현 endpoint 요약

예:
	•	GET /api/v1/documents/{document_id}/versions
	•	POST /api/v1/documents/{document_id}/versions
	•	GET /api/v1/versions/{version_id}
	•	GET /api/v1/versions/{version_id}/nodes
	•	GET /api/v1/versions/{version_id}/nodes/{node_id}

C. version / node schema 요약

예:
	•	version create request 필드
	•	version response 필드
	•	node response 필드
	•	node tree 표현 방식

D. 관계 검증 및 authz 적용 요약

간단히:
	•	어떤 관계 검증을 넣었는지
	•	어떤 action naming을 사용했는지
	•	어떤 리소스 참조를 authz에 넘겼는지

E. 설계 판단 근거

짧게 정리:
	•	왜 canonical path를 이렇게 정했는지
	•	왜 version get과 nodes list를 분리했는지
	•	왜 node를 flat structured 방식으로 시작했는지
	•	왜 version create를 snapshot 생성으로 본 것인지

F. 남겨둔 TODO

예:
	•	diff/compare API 추후 구현
	•	restore/rollback 추후 구현
	•	node tree projection helper 추후 구현
	•	Task I-9 idempotency hook 연결 예정
	•	Task I-12 citation/read-model 강화 예정

⸻

8. 완료 조건

아래를 만족하면 완료로 본다.
	•	version list/create/get API가 동작한다.
	•	node list/get API가 동작한다.
	•	request/response schema가 정의되어 있다.
	•	document-version-node 관계 검증이 존재한다.
	•	공통 response/error/context/auth/list-query 계층이 실제로 적용되어 있다.
	•	versions/nodes service 계층이 존재한다.
	•	persistence 접근 경계가 정리되어 있다.
	•	AI/RAG 확장을 막지 않는 구조적 식별성과 read 친화성이 확보되어 있다.

⸻

9. 금지 사항

이번 작업에서는 다음을 하지 마라.
	•	diff/restore/merge까지 한 번에 끌고 가지 말 것
	•	node edit/tree mutation 전체를 지금 구현하려 하지 말 것
	•	version get에 node 전체를 무조건 embed하지 말 것
	•	documents와 versions의 책임을 섞지 말 것
	•	document 직속 node API를 성급히 만들지 말 것
	•	retrieval 전용 우회 API를 먼저 만들지 말 것

⸻

10. Claude Code 최종 지시문

아래 지시문을 그대로 사용해도 된다.

Task I-8을 수행하라.

목표:
- 문서의 버전성과 구조성을 실제 API로 노출하기 위해 versions / nodes API를 구현한다.
- version list/create/get 및 node list/get의 최소 흐름을 구현한다.
- document → version → node 관계를 API 계층에서 명확히 드러내고 검증한다.
- 공통 response/error/context/auth/list-query 계층을 versions/nodes API에도 실제로 적용한다.
- 이후 diff/restore/idempotency/AI-RAG 확장을 막지 않는 구조를 만든다.

반드시 지킬 원칙:
- 이번 단계는 full version management 완성이 아니라 version/node 리소스의 최소 API 구현 단계다.
- document, version, node의 책임을 섞지 마라.
- version create는 새 구조 스냅샷 생성으로 접근하라.
- node 응답은 flat structured 방식으로 시작하라.
- version get과 nodes list의 책임을 분리하라.
- 관계 검증과 authz hook을 반드시 넣어라.

수행 항목:
1. 기존 document/version/node 구조를 점검한다.
2. versions / nodes API contract를 구체화한다.
3. canonical/subresource path 정책을 정한다.
4. version request/response schema를 정의한다.
5. node request/response schema를 정의한다.
6. versioning 정책과 node tree 표현 정책을 최소 수준으로 정한다.
7. document-version-node 관계 검증을 구현한다.
8. versions service 및 nodes service를 구현한다.
9. persistence/repository 경계를 연결한다.
10. version list/create/get 및 node list/get endpoint를 구현한다.
11. authz action/resource 흐름을 적용한다.
12. 공통 response/error/context/list-query 계층을 실제로 연결한다.
13. AI/RAG 확장을 막지 않는 구조인지 검토한다.

산출물:
- 생성/수정 파일 목록
- 구현 endpoint 요약
- version/node schema 요약
- 관계 검증 및 authz 적용 요약
- 설계 판단 근거
- 남겨둔 TODO 목록