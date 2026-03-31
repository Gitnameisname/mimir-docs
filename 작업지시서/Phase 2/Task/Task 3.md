Task 2-3 작업 지시서

주제

ACL 기반 권한 모델 설계

작업 목적

Task 2-1에서 권한의 주체(User, Organization, Membership, Role 등)를 정리했고, Task 2-2에서 보호 대상 리소스와 scope 구조를 식별했다.
이번 단계에서는 이 둘을 연결하여, 누가(principal) 어떤 리소스(resource)에 대해 어떤 행위(permission)를 어떤 효과(effect)로 수행할 수 있는지를 표현하는 ACL 기반 권한 모델을 설계한다.

이 작업의 목표는 구현 코드 작성이 아니라, 이후 API enforcement와 감사 로그, 정책 엔진으로 자연스럽게 확장 가능한 권한 표현 구조와 평가 원칙의 기준선을 확정하는 것이다.

⸻

이번 작업의 범위

이번 작업에서는 아래를 다룬다.
	1.	ACL 엔트리 개념 모델 정의
	•	principal
	•	resource
	•	permission
	•	effect (allow / deny)
	•	scope
	•	inheritance / override 개념
	2.	권한 표현 단위 설계
	•	역할(Role)을 통한 간접 권한
	•	ACL을 통한 직접 권한
	•	예외 권한 부여 구조
	3.	permission taxonomy 초안
	•	읽기 / 작성 / 수정 / 삭제 / 관리 / 공유 / 승인 등
	•	콘텐츠 리소스와 운영 리소스에 대한 권한 구분
	4.	권한 평가 원칙 설계
	•	allow / deny 충돌 규칙
	•	상속된 권한과 직접 권한의 우선순위
	•	role 기반 권한과 ACL 예외의 관계
	5.	기본 정책과 예외 정책 구조화
	•	기본적으로는 RBAC 중심
	•	예외 케이스는 ACL로 보완
	•	과도하게 복잡한 정책 엔진으로 가지 않도록 제한

⸻

이번 작업에서 하지 말 것

아래는 이번 단계에서 다루지 않는다.
	•	실제 authorization middleware 구현
	•	API endpoint별 권한 체크 코드 작성
	•	DB migration 작성
	•	ORM 모델 구현
	•	감사 로그 저장 구조 상세 구현
	•	정책 엔진 전체 구현
	•	UI 메뉴 권한 설정 화면 설계

즉, 이번 단계는 ACL 표현 구조와 평가 규칙 설계 단계다.

⸻

설계 전제

아래 전제를 반영하여 설계할 것.
	1.	플랫폼은 범용 문서 플랫폼이며, 문서/버전/노드/첨부/댓글/운영 객체 등 다양한 리소스를 가진다.
	2.	기본 권한 모델은 RBAC 중심으로 시작하되, 문서나 특정 리소스에 대한 예외 권한을 표현하기 위해 ACL이 필요하다.
	3.	ACL은 RBAC를 대체하는 것이 아니라, RBAC 위에 얹는 예외/세분화 계층으로 설계해야 한다.
	4.	향후 approval workflow, external share, service account, integration, policy engine으로 확장 가능해야 한다.
	5.	API-first 구조이므로 권한은 UI 메뉴 기준이 아니라 리소스 행위 기준으로 정의해야 한다.
	6.	감사 추적을 위해 “누가 어떤 자격으로 어떤 리소스에 어떤 권한으로 접근했는가”를 해석할 수 있어야 한다.

⸻

Claude Code가 수행해야 할 작업

아래 순서대로 정리할 것.

1. ACL 모델의 역할 정의

먼저 아래 질문에 답하면서 ACL의 위치를 정리할 것.
	•	이 플랫폼에서 ACL은 왜 필요한가
	•	RBAC만으로 부족한 이유는 무엇인가
	•	ACL은 어떤 상황에서만 사용되어야 하는가
	•	ACL을 기본 권한 체계로 삼지 않고 보조 계층으로 두는 이유는 무엇인가

그리고 아래 원칙을 문장으로 정리할 것.
	•	기본 권한은 Role 기반으로 부여
	•	예외 권한은 ACL로 부여
	•	운영 복잡도가 급증하지 않도록 ACL 사용 범위를 제한

⸻

2. ACL 엔트리 구성 요소 정의

최소한 아래 요소를 포함하는 ACL 엔트리 개념 구조를 정의할 것.
	•	subject_type 또는 principal_type
	•	subject_id 또는 principal_id
	•	resource_type
	•	resource_id
	•	permission
	•	effect
	•	scope 또는 inheritance flag
	•	condition 필요 여부
	•	created_by / reason / source 같은 감사 친화 필드 필요 여부

각 필드에 대해 아래를 설명할 것.
	•	왜 필요한가
	•	필수인지 선택인지
	•	단순 ACL MVP에 포함할지, 후속 확장으로 둘지

중요:
여기서는 실제 DB 컬럼 설계를 확정하는 것이 아니라, ACL이 어떤 정보를 가져야 하는지 개념적으로 정의하는 것이다.

⸻

3. Principal 유형 정의

ACL의 subject/principal 후보를 정리할 것. 최소한 아래를 검토할 것.
	•	User
	•	Membership
	•	Team/Group
	•	Role
	•	ServiceAccount / API Client
	•	ExternalSharePrincipal 필요 여부

각 principal 유형에 대해 다음을 정리할 것.
	•	ACL의 직접 대상이 될 수 있는가
	•	MVP에서 지원해야 하는가
	•	운영 복잡성을 높이는가
	•	감사 추적에 어떤 장단점이 있는가

그리고 최종 권장안을 제시할 것.
예를 들어:
	•	MVP에서는 Membership + Team 정도까지만 허용
	•	Role 자체를 ACL principal로 볼지 여부
	•	ServiceAccount는 후속 확장인지 여부

⸻

4. Permission taxonomy 초안 작성

리소스 행위 기준으로 permission 범주를 정의할 것.

최소한 아래 축을 검토할 것.

콘텐츠 리소스 권한
	•	read
	•	create
	•	update
	•	delete
	•	comment
	•	annotate
	•	attach
	•	version_view
	•	version_create
	•	restore
	•	export
	•	share

관리 리소스 권한
	•	manage_metadata
	•	manage_document_type
	•	manage_template
	•	manage_permissions
	•	manage_workflow
	•	approve
	•	publish
	•	archive
	•	lock

운영 리소스 권한
	•	view_audit_log
	•	manage_org
	•	manage_members
	•	manage_roles
	•	manage_integrations
	•	manage_api_credentials
	•	system_admin

중요:
permission을 너무 세분화해서 폭발시키지 말고,
MVP permission set 과 후속 확장 permission set 으로 나누어 정리할 것.

또한 아래도 답할 것.
	•	permission은 action verb 중심이 좋은가
	•	CRUD 기반과 domain-specific action을 어떻게 혼합할 것인가
	•	permission naming rule은 어떻게 가져갈 것인가

⸻

5. Effect 및 충돌 규칙 정의

ACL에서 effect를 어떻게 둘지 검토할 것.

최소한 아래를 비교할 것.
	•	allow only
	•	allow + deny

그리고 아래 질문에 답할 것.
	•	deny를 지원하면 어떤 장점이 있는가
	•	deny를 지원하면 평가 복잡도가 얼마나 증가하는가
	•	MVP에서 deny를 지원해야 하는가, 아니면 후속 단계로 미뤄야 하는가

그 후 최종 권장안을 제시할 것.
예:
	•	MVP는 allow 중심 + 명시적 deny 미지원
	•	고급 정책 단계에서 deny 도입 가능
	•	또는 deny를 지원하되 범위를 엄격히 제한

⸻

6. 상속과 override 원칙 정의

Task 2-2에서 정리한 resource scope를 전제로, ACL이 상속되는 방식을 설계할 것.

최소한 아래 질문에 답할 것.
	•	Organization level ACL이 Document까지 내려갈 수 있는가
	•	Document ACL이 Node / Comment / Attachment에 상속되는가
	•	Version은 상속 대상인가 독립 리소스인가
	•	상위 allow와 하위 예외 ACL이 충돌하면 어떻게 되는가
	•	direct assignment와 inherited assignment 중 무엇이 우선하는가

그리고 아래 형태로 정리할 것.
	•	기본 상속 규칙
	•	하위 override 허용 여부
	•	예외를 허용할 리소스
	•	override 남용 방지 원칙

⸻

7. RBAC와 ACL 결합 방식 설계

이번 Task의 핵심이다. 아래 구조를 비교 검토하고 추천안을 제시할 것.

방식 A
	•	Role 자체가 permission bundle
	•	ACL은 user/membership 단위 예외 부여

방식 B
	•	Role도 ACL principal처럼 동작
	•	모든 권한 부여를 ACL 엔트리로 통일

방식 C
	•	기본은 role-permission mapping
	•	resource-specific exception만 별도 ACL로 추가

각 방식에 대해 아래를 비교할 것.
	•	설계 단순성
	•	운영 난이도
	•	감사 가능성
	•	API enforcement 구현 용이성
	•	향후 정책 엔진 연결성

그리고 최종 권장안을 명확히 제시할 것.

⸻

8. 권한 평가 흐름 초안 작성

실제 코드가 아니라 논리 흐름 수준으로 정리할 것.

예시 질문:
	•	요청자가 누구인지 식별
	•	어떤 membership/role/principal 자격으로 요청하는지 결정
	•	대상 resource와 scope 확인
	•	role 기반 기본 권한 평가
	•	resource-specific ACL 확인
	•	예외/override 적용
	•	최종 allow/deny 결론 도출

이 흐름을 단계별로 설명하고, 아래도 포함할 것.
	•	evaluation input
	•	evaluation output
	•	감사 로그에 남겨야 할 decision context

⸻

9. MVP용 ACL 범위 제한안 제시

MVP에서 ACL을 어디까지 허용할지 제한할 것.

예시 검토 포인트:
	•	어떤 resource에만 ACL 허용할 것인가
	•	어떤 principal에만 ACL 허용할 것인가
	•	어떤 permission만 ACL 예외 부여 가능할 것인가
	•	ACL 생성/수정 권한은 누구에게 줄 것인가

즉, “ACL은 유연하지만 위험하므로, 초기에는 좁게 시작한다”는 관점에서 제한안을 제시할 것.

⸻

10. 다음 단계 입력 자료 형태로 정리

Task 2-4 API enforcement 설계에 넘기기 위해 아래를 정리할 것.
	•	권한 평가에 필요한 입력 요소
	•	API 계층에서 반드시 알아야 하는 principal/resource/permission 매핑
	•	공통 authorization service가 판단해야 할 정보
	•	감사 로그와 연결할 수 있는 decision metadata

⸻

산출물 형식

이번 작업 결과물은 설계 문서 초안이어야 하며, 아래 구조를 따를 것.

Phase 2 - Task 2-3

1. 목표

2. 설계 전제

3. ACL 도입 목적과 역할

4. ACL 엔트리 개념 모델

5. Principal 유형 정의

6. Permission taxonomy 초안

7. Effect 및 충돌 규칙

8. 상속 및 override 원칙

9. RBAC와 ACL 결합 권장안

10. 권한 평가 흐름 초안

11. MVP ACL 제한안

12. 다음 Task로 넘길 결정사항

13. 오픈 이슈

⸻

의사결정 원칙

설계 중 아래 원칙을 반드시 지킬 것.
	1.	RBAC 우선, ACL 보완
	•	ACL이 기본 체계가 되면 운영 복잡도가 너무 커진다
	•	기본은 역할 기반, 예외만 ACL로 처리할 것
	2.	리소스 행위 기준으로 권한 정의
	•	메뉴 권한이나 화면 권한이 아니라 resource + action 기준으로 설계할 것
	3.	감사 가능성 유지
	•	나중에 “왜 허용되었는가”를 설명할 수 있는 구조를 선호할 것
	4.	MVP는 단순하게
	•	deny, condition, attribute-based rule 등은 신중히 도입할 것
	5.	상속은 기본, override는 제한
	•	예외는 가능하되 무제한 허용하지 말 것

⸻

기대 결과

이 작업이 끝나면 최소한 아래가 확정되어 있어야 한다.
	•	ACL이 이 플랫폼에서 맡는 역할
	•	ACL 엔트리의 핵심 구성 요소
	•	principal / resource / permission / effect 구조
	•	permission taxonomy의 초기 분류
	•	RBAC와 ACL을 결합하는 권장 방식
	•	권한 평가 흐름의 논리 구조
	•	MVP에서 허용할 ACL 범위 제한안

⸻

금지 사항
	•	실제 authorization 코드를 작성하지 말 것
	•	permission matrix를 모든 리소스 조합으로 과도하게 확장하지 말 것
	•	attribute-based policy engine까지 이번 단계에 섞지 말 것
	•	UI 권한 체크 중심으로 설계하지 말 것
	•	deny/condition을 무리해서 도입하지 말 것

⸻

작업 완료 판단 기준

아래 조건을 만족하면 완료로 본다.
	•	ACL의 역할이 RBAC 대비 명확히 설명되어 있다
	•	ACL 엔트리 개념 구조가 정의되어 있다
	•	permission taxonomy가 MVP/확장 관점으로 정리되어 있다
	•	RBAC + ACL 결합 방식의 권장안이 제시되어 있다
	•	권한 평가 흐름이 API enforcement 단계로 넘길 수 있을 만큼 정리되어 있다