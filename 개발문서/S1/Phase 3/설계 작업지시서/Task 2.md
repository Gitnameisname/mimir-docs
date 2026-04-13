Task 3-2. 핵심 리소스 기반 REST API 구조 설계

Claude Code 작업 지시서

1. 작업명

Phase 3 - Task 3-2. 핵심 리소스 기반 REST API 구조 설계

⸻

2. 작업 목적

플랫폼 API의 핵심 리소스를 식별하고, 이를 기준으로 REST API의 상위 구조를 설계하라.

이번 작업의 목적은 단순히 엔드포인트를 많이 나열하는 것이 아니다.
문서 플랫폼의 도메인 모델과 운영 모델을 반영하여, 앞으로 구현될 API들이 일관된 방식으로 확장될 수 있도록 리소스 중심의 URL 구조, 계층 관계, 명명 규칙, 서브리소스 규칙, 액션 엔드포인트 허용 기준을 정의하는 것이다.

이 작업은 이후의 다음 설계에 직접 연결된다.
	•	인증/인가 구조 반영
	•	공통 응답 포맷
	•	pagination/filter/sort 규약
	•	idempotency 적용 위치
	•	이벤트/웹훅 모델
	•	AI/RAG 연계 API
	•	실제 FastAPI/백엔드 라우터 구조 설계

즉, 이번 문서는 플랫폼 API의 리소스 지도(resource map) 와 REST 구조 기준선을 만드는 작업이다.

⸻

3. 반드시 반영할 배경

이 플랫폼은 단순 문서 CRUD 시스템이 아니라 다음을 포함하는 범용 문서 플랫폼이다.
	•	다양한 문서 유형 수용
	•	Document / Version / Node 중심 구조
	•	사용자/조직/역할/권한 모델 존재
	•	감사 추적 필요
	•	검색/이벤트/웹훅/AI/RAG 연계 예정
	•	User/Admin UI 분리
	•	외부 시스템 및 AI 도구 연계 가능해야 함

또한 이전 Task 3-1에서 다음 원칙이 이미 정해졌다고 가정하라.
	•	API-first
	•	Resource-oriented
	•	Consistency over convenience
	•	Secure by default
	•	Extensible by design
	•	AI-integratable
	•	Explicit contracts over implicit behavior

이번 작업은 이 원칙을 실제 REST 리소스 구조로 번역하는 단계다.

⸻

4. 이번 작업의 핵심 질문

이번 문서에서는 최소한 아래 질문들에 답할 수 있어야 한다.
	1.	이 플랫폼에서 API 리소스로 다뤄야 할 핵심 객체는 무엇인가?
	2.	어떤 것은 최상위 리소스이고, 어떤 것은 하위 리소스인가?
	3.	문서, 버전, 노드, 권한, 감사 로그, 검색, 이벤트, 웹훅, AI 연계 항목은 어떤 구조로 배치되어야 하는가?
	4.	nested resource는 어디까지 허용해야 하는가?
	5.	REST로 표현하기 어려운 작업성 기능은 어떤 기준으로 action endpoint로 허용할 것인가?
	6.	bulk operation이나 비동기 작업은 REST 구조 안에서 어떻게 표현할 것인가?
	7.	미래에 문서 유형과 기능이 늘어나도 URL 구조가 무너지지 않으려면 어떤 규칙이 필요한가?

⸻

5. 상세 작업 요구사항

5-1. 문서 작성

아래 산출물을 작성하라.

필수 산출물
	•	Task2_rest_resource_map.md
	•	Task2_endpoint_naming_rules.md

두 문서는 서로 연결되어야 하지만, 역할은 분리하라.
	•	Task2_rest_resource_map.md
	•	어떤 리소스가 존재하는지
	•	리소스 간 관계가 어떤지
	•	상위 URL 구조가 어떻게 잡히는지
	•	collection / item / subresource의 구조가 어떤지
	•	Task2_endpoint_naming_rules.md
	•	URI 명명 원칙
	•	복수형/단수형 규칙
	•	nested path 규칙
	•	action endpoint 허용 기준
	•	bulk endpoint 표기 방식
	•	검색/조회/상태/작업 계열 네이밍 가이드

⸻

5-2. 핵심 리소스 식별

최소한 아래 후보들을 검토하고, 각각에 대해 최상위 리소스인지 / 서브리소스인지 / 별도 관리 리소스인지 판단하라.

검토 대상 리소스
	•	organizations
	•	users
	•	memberships
	•	roles
	•	documents
	•	document-types
	•	document-schemas 또는 document-type-definitions
	•	versions
	•	nodes
	•	attachments
	•	permissions / acl entries
	•	audit-logs
	•	activities or action-traces
	•	searches
	•	search-results 여부
	•	events
	•	webhooks
	•	webhook-deliveries
	•	jobs / operations
	•	ai or rag-related resources
	•	indexes / ingestion-status 여부
	•	comments / annotations 여부
	•	tags / labels 여부

중요:
	•	위 목록을 무조건 다 채택하지는 말고,
	•	플랫폼 관점에서 핵심 리소스 / 파생 리소스 / 내부 구현 개념을 구분해서 판단하라.

즉, API 리소스로 공개해야 할 것과 내부 구현에 남겨야 할 것을 구분해야 한다.

⸻

5-3. 리소스 분류 체계 정의

문서에서 리소스를 다음 정도로 분류하라.
	•	Core domain resources
	•	Identity / access resources
	•	Governance / audit resources
	•	Integration / event resources
	•	Search / AI extension resources
	•	Operational resources

각 분류별로:
	•	왜 필요한지
	•	외부에 직접 노출할 가치가 있는지
	•	어떤 소비자가 주로 사용할지
	•	공개 범위가 어디까지 적절한지

를 정리하라.

⸻

5-4. 상위 URL 구조 초안 설계

문서에는 플랫폼의 상위 URL 구조 방향을 정의하라.

예를 들어 다음 같은 수준의 논의가 필요하다.
	•	/api/v1/documents
	•	/api/v1/document-types
	•	/api/v1/organizations
	•	/api/v1/users
	•	/api/v1/searches
	•	/api/v1/webhooks
	•	/api/v1/events
	•	/api/v1/jobs

하지만 이번 작업에서는 모든 세부 path를 exhaustive하게 만들 필요는 없다.
대신 다음을 분명히 해야 한다.
	•	어떤 리소스를 top-level collection으로 둘 것인가
	•	어떤 관계는 subresource로 둘 것인가
	•	조직 스코프를 path에 포함할지, 컨텍스트로 처리할지, 혼합할지
	•	리소스 간 참조는 path nesting보다 식별자 참조를 우선할지 여부
	•	지나치게 깊은 nesting을 피하는 기준

⸻

5-5. Collection / Item / Subresource 규칙

다음을 명확히 문서화하라.
	•	collection path 규칙
	•	item path 규칙
	•	subresource path 규칙
	•	relation-like resource를 어떻게 표현할지
	•	같은 리소스를 여러 부모 아래 중복 노출할지 여부
	•	canonical path 개념 도입 여부

예를 들어 다음 같은 질문에 답할 수 있어야 한다.
	•	version은 /documents/{documentId}/versions 아래만 둘 것인가?
	•	node는 /documents/{documentId}/versions/{versionId}/nodes 구조만 허용할 것인가?
	•	ACL은 /documents/{documentId}/permissions 형태로 둘 것인가, 혹은 별도 /permissions 리소스로도 조회 가능한가?
	•	webhook delivery는 /webhooks/{webhookId}/deliveries 식이 적절한가?
	•	audit log는 개별 도메인 아래 둘지, 전역 관리 리소스로 둘지?

⸻

5-6. Nested resource 설계 원칙

반드시 다음 내용을 포함하라.
	•	nested resource는 관계를 표현할 때 유용하지만 과도하면 URL이 복잡해짐
	•	1~2단계 nesting을 기본 권장 범위로 둘지 검토
	•	깊은 탐색이 필요한 경우 query parameter나 direct resource lookup을 우선할지 검토
	•	부모 없이는 존재 의미가 약한 리소스는 subresource로 표현 가능
	•	독립적 식별/조회 가치가 높은 리소스는 top-level 또는 canonical endpoint를 둘 수 있음

즉, nesting을 허용하되 남용하지 않는 기준을 제시하라.

⸻

5-7. Action endpoint 허용 기준

이 플랫폼은 리소스 중심 원칙을 따르지만, 일부 기능은 순수 CRUD로 표현하기 어렵다.
문서에서 action endpoint를 언제 허용하는지 기준을 정의하라.

반드시 다뤄야 할 예시:
	•	publish
	•	archive
	•	restore
	•	reindex
	•	validate
	•	compare
	•	clone
	•	regenerate-preview
	•	resend-webhook
	•	retry-job

다음 관점을 정리하라.
	•	상태 전이를 action으로 둘지, PATCH로 풀지
	•	계산/검증/비동기 트리거는 action endpoint가 적절한지
	•	action endpoint 명명 규칙
	•	action endpoint 남발 방지 기준
	•	action endpoint도 리소스 계약의 일부로 관리해야 한다는 점

중요:
문서는 “action endpoint를 금지”가 아니라
허용하되, 엄격한 기준 아래 제한적으로 사용하는 방향으로 작성하라.

⸻

5-8. Bulk operation 설계 방향

다음을 정리하라.
	•	bulk create / update / delete / permission change / tag update 같은 작업이 필요한지
	•	bulk 작업을 별도 리소스로 볼지, action endpoint로 볼지
	•	동기 처리 vs 비동기 job 처리 기준
	•	partial success가 가능한 경우 응답 계약을 어떻게 바라봐야 하는지
	•	bulk API를 초기부터 열지, 제한적으로 설계할지

이번 작업에서는 상세 응답 JSON까지는 가지 말고, 구조 원칙만 정의하라.

⸻

5-9. 검색 / 조회 / AI 연계 리소스 방향

플랫폼은 문서 CRUD만 있는 것이 아니므로, 다음 항목을 리소스 구조 관점에서 정리하라.
	•	검색은 /search 같은 RPC성 엔드포인트로 둘지
	•	/searches 라는 리소스 개념으로 둘지
	•	retrieval/AI workflow를 위한 조회 계층을 어떻게 둘지
	•	AI용 composite read 모델이 필요한 경우 어떻게 다룰지
	•	문서 구조 조회, 버전 조회, 노드 조회, citation-friendly identifier를 어떤 리소스 전략으로 수용할지

여기서는 최종 확정보다는 방향을 정리하되,
REST 리소스 체계와 충돌하지 않도록 설명하라.

⸻

5-10. 운영/거버넌스 리소스 방향

다음 항목도 검토하라.
	•	audit-logs
	•	activities / traces
	•	jobs / operations
	•	events
	•	webhooks
	•	webhook-deliveries

각각에 대해:
	•	일반 사용자 API인지
	•	관리자/운영 API인지
	•	외부 통합 API인지
	•	전역 리소스인지
	•	특정 부모 아래의 subresource인지

를 정리하라.

⸻

5-11. 리소스 관계도 또는 표 작성

문서에는 최소한 하나 이상의 구조화된 정리물이 있어야 한다.

권장 예시:
	•	리소스 분류 표
	•	리소스별 canonical path 후보 표
	•	subresource 관계 표
	•	공개 여부/소비자 유형 표

즉, prose만 쓰지 말고 후속 작업자가 빠르게 볼 수 있는 정리표를 포함하라.

⸻

5-12. 엔드포인트 명명 규칙 문서 작성

Task2_endpoint_naming_rules.md에는 최소한 아래를 포함하라.

반드시 포함할 항목
	1.	path는 명사 중심
	2.	collection은 복수형 사용 원칙
	3.	동사형 path 지양 원칙
	4.	kebab-case / snake_case / camelCase 중 path 표기 선택
	5.	path parameter naming 규칙
	6.	subresource naming 규칙
	7.	action endpoint suffix/pattern 규칙
	8.	bulk endpoint naming 규칙
	9.	admin 전용/시스템 전용 API 네이밍 구분 방향
	10.	실험적 API 네이밍 또는 격리 전략
	11.	금지 예시 / 권장 예시

예를 들어 다음 같은 판단이 가능해야 한다.
	•	/documents/{documentId}/publish vs /documents/{documentId}:publish vs /publish-document
	•	/documentTypes 같은 camelCase path를 허용할지
	•	/documents/{id}/acl 와 /documents/{id}/permissions 중 어떤 방향이 더 적절한지
	•	/admin/... 를 둘지, 권한은 path가 아니라 auth로만 구분할지
	•	/search vs /searches
	•	/jobs/{jobId}/retry 식 표현 허용 여부

⸻

6. 산출물

아래 두 파일을 작성하라.

필수 산출물
	•	Task2_rest_resource_map.md
	•	Task2_endpoint_naming_rules.md

문서 성격
	•	설계 문서
	•	구현 코드 금지
	•	특정 프레임워크/라우터 라이브러리 의존 설명 최소화
	•	실제 구현 전에 기준선으로 쓸 수 있는 수준의 명확성 필요

⸻

7. 제외 범위

이번 작업에서는 아래는 하지 마라.
	•	OpenAPI 스펙 전체 작성
	•	실제 FastAPI router 코드 작성
	•	인증/인가 정책 상세 설계
	•	응답 envelope 상세 정의
	•	pagination/filter/sort 구체 문법 정의
	•	idempotency 설계 상세화
	•	webhook payload schema 작성
	•	AI/RAG endpoint 세부 계약 확정
	•	DB schema 설계
	•	내부 서비스 클래스 설계

즉, 이번 작업은 REST 리소스 구조와 명명 규칙의 기준선 정의에 집중하라.

⸻

8. 작성 스타일 가이드

문서는 아래 스타일을 따르라.
	•	추상 원칙과 실제 구조 예시를 균형 있게 작성
	•	리소스 선정 이유를 설명
	•	path 예시는 넣되, exhaustive endpoint catalog처럼 만들지 말 것
	•	“권장 / 허용 / 비권장 / 금지”를 구분해서 쓸 것
	•	향후 Task에서 구체화할 부분은 명시적으로 남길 것
	•	문서 전체가 후속 설계자가 바로 참조 가능한 기준 문서여야 함

⸻

9. 검토 체크리스트

작업 완료 전에 아래를 스스로 점검하라.
	•	핵심 리소스와 파생 리소스가 구분되었는가?
	•	top-level / subresource / canonical path 개념이 정리되었는가?
	•	nested resource 남용 방지 기준이 있는가?
	•	action endpoint 허용 기준이 분명한가?
	•	bulk / job / webhook / search / AI 연계 항목이 빠지지 않았는가?
	•	endpoint naming 규칙이 충분히 명확한가?
	•	이후 Task 3-3, 3-5, 3-6, 3-8, 3-9가 이 결과를 기반으로 이어질 수 있는가?

⸻

10. Claude Code에 대한 최종 지시

위 요구사항을 반영하여 다음 두 문서를 작성하라.
	•	Task2_rest_resource_map.md
	•	Task2_endpoint_naming_rules.md

문서는 범용 문서 플랫폼의 REST API 구조를 장기적으로 안정화하기 위한 기준 문서여야 한다.
리소스 중심 설계 원칙을 유지하되, 검색, 이벤트, 비동기 작업, AI/RAG 연계 같은 확장 요구도 수용할 수 있게 정리하라.

문서 작성 후에는 아래 형식의 자체 점검 요약도 함께 정리하라.

자체 점검 요약
	•	식별한 핵심 리소스 목록
	•	top-level / subresource 설계 기준
	•	action endpoint 허용 기준 요약
	•	naming 규칙 핵심 요약
	•	후속 Task 연결 포인트
	•	의도적으로 후속 Task로 넘긴 미결정 사항