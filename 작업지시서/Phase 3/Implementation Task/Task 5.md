Task I-5. 인증 컨텍스트 추출 및 authorization hook point 구현

Claude Code 작업 계획서

1. 작업 목표

Task I-4에서 만든 RequestContext plumbing 위에, 인증 주체(actor)를 API 계층에서 추출하고, 이후 권한 판단을 붙일 수 있는 authorization hook point를 구현한다.

이번 단계의 핵심은 “완전한 인증/인가 시스템 완성”이 아니다.
핵심은 아래 구조를 실제 코드 흐름으로 고정하는 것이다.
	•	요청에서 actor를 추출해 RequestContext.actor에 반영
	•	anonymous / authenticated / service actor를 구분 가능한 형태로 정규화
	•	router가 직접 권한을 하드코딩하지 않고, authorization service나 policy evaluator로 연결되는 패턴 도입
	•	tenant scope enforcement를 이후 붙일 수 있는 연결 지점 확보
	•	documents / versions / nodes API가 이후 보안 구조를 해치지 않고 확장될 수 있게 함

즉, 이번 Task는 보안 구조를 망치지 않는 API 계층 패턴을 만드는 작업이다.

⸻

2. 작업 범위

포함 범위
	•	current actor 추출용 dependency 또는 동등 구조 구현
	•	anonymous / authenticated / service actor 구분 구조
	•	RequestContext.actor 갱신 흐름 구현
	•	인증 소스 추출 placeholder
	•	authorization service interface 또는 policy check hook point 정의
	•	router에서 직접 권한 체크를 하드코딩하지 않는 패턴 도입
	•	tenant scope enforcement 연결 위치 마련
	•	일부 endpoint에 시범 적용

제외 범위
	•	완전한 JWT/session/api key 인증 구현
	•	사용자 DB 조회 완성
	•	RBAC/ACL 전체 구현
	•	세부 permission matrix 구현
	•	실제 tenant isolation enforcement 완성
	•	admin 전용 정책 완성
	•	외부 IAM 연동

⸻

3. 구현 의도

지금 이 단계에서 아무 구조 없이 documents API를 계속 늘리면, 나중에 다음 문제가 생긴다.
	•	어떤 라우터는 header를 직접 파싱
	•	어떤 라우터는 dependency로 유저 조회
	•	어떤 서비스는 actor 없이 실행
	•	권한 체크가 라우터마다 if문으로 흩어짐
	•	tenant scope가 나중에 붙으면서 전체 API를 다시 뜯어야 함

따라서 이번 작업의 목적은 다음과 같다.
	•	actor 추출 경로를 표준화
	•	인증 전/후 상태를 같은 ActorContext 모델 위에서 표현
	•	authorization 판단을 별도 계층으로 연결할 수 있는 인터페이스를 만든다
	•	현재는 stub이더라도, 향후 ACL/RBAC/policy engine으로 확장 가능한 구조를 고정한다

⸻

4. Claude Code에게 요청할 작업

아래 작업을 순서대로 수행하라.

⸻

Step 1. 현재 인증 관련 코드/패턴 점검

현재 코드베이스에서 인증/사용자 식별이 어떻게 이뤄지는지 먼저 점검하라.

확인할 항목:
	•	기존 get_current_user 류 dependency 존재 여부
	•	session/cookie/bearer/api key 등 어떤 인증 방식 흔적이 있는지
	•	middleware에서 auth를 다루는지
	•	router가 직접 user를 해석하는 부분이 있는지
	•	admin/service caller를 구분하는 로직이 있는지

목적:
	•	기존 구조를 최대한 활용하되, 이번 Task의 범위에 맞게 “actor extraction + authorization hook point” 중심으로 재정렬하는 것

⸻

Step 2. actor 분류 기준 정리

요청 주체(actor)를 최소 다음 세 범주로 다룰 수 있게 정리하라.
	•	anonymous
	•	user 또는 authenticated_user
	•	service

권장 추가 값:
	•	admin_user는 지금 별도 actor_type으로 둘지, user + role로 둘지 검토 가능
	•	하지만 이번 단계에서는 지나친 세분화보다 단순한 분류가 우선

권장 구조:
	•	actor_type
	•	actor_id
	•	is_authenticated
	•	auth_method

중요:
	•	분류 기준은 이후 policy engine에서 재사용 가능해야 한다
	•	라우터에서 “로그인한 사용자냐 아니냐”를 각자 해석하지 않도록 할 것

⸻

Step 3. current actor extraction dependency 구현

요청에서 actor를 추출하여 RequestContext.actor를 세팅하거나 갱신하는 dependency를 구현하라.

권장 개념:
	•	resolve_current_actor(request) -> ActorContext
	•	또는 get_current_actor_context(...)

이 dependency는 다음 책임만 가진다.
	•	인증 관련 입력(header/cookie/token placeholder) 확인
	•	actor를 anonymous/user/service 중 하나로 정규화
	•	RequestContext.actor에 반영
	•	downstream에서 사용할 ActorContext 반환 가능

중요:
	•	아직 완전한 검증 로직이 없어도 괜찮다
	•	지금은 “actor를 공통 모델로 정규화하는 entrypoint”를 만드는 것이 목적이다

⸻

Step 4. 인증 입력 소스 placeholder 정리

지원할 인증 입력 소스를 placeholder 수준으로 정리하라.

예:
	•	session/cookie
	•	bearer token
	•	api key
	•	internal service header

이번 단계에서 중요한 점:
	•	전부 완성하지 않아도 된다
	•	어떤 입력 소스를 어떤 우선순위로 볼지 구조만 잡아라
	•	auth_method에 기록 가능한 형태로 정규화하라

예시:
	•	auth_method = "session"
	•	auth_method = "bearer"
	•	auth_method = "api_key"
	•	auth_method = "internal_service"
	•	인증 실패 또는 없음 → anonymous

주의:
	•	이 단계에서 보안 민감한 임시 우회 로직을 넣지 말 것
	•	검증이 미완성인 경우, 명확히 TODO/stub 처리할 것

⸻

Step 5. RequestContext.actor 갱신 규칙 구현

Task I-4에서 anonymous 기본 actor를 넣어두었다면, 이번 단계에서는 인증 dependency가 그것을 업데이트할 수 있는 규칙을 구현하라.

예:
	•	기본값: anonymous
	•	인증 dependency 실행 후:
	•	user 확인 시 actor_type="user"
	•	service caller 확인 시 actor_type="service"
	•	없으면 그대로 anonymous

중요:
	•	actor context 갱신 지점은 일관되어야 한다
	•	router나 service가 임의로 request.state를 수정하지 않게 할 것

⸻

Step 6. authorization service interface 정의

실제 permission 판단을 지금 완성하지 않더라도, authorization hook point는 이번 단계에서 반드시 만든다.

권장 개념:
	•	AuthorizationService
	•	PolicyEvaluator
	•	PermissionChecker

최소 인터페이스 예시:
	•	authorize(actor, action, resource, tenant_scope=None)
	•	또는
	•	check_permission(context, action, resource_ref)

핵심:
	•	라우터는 “이 요청이 어떤 action/resource에 해당하는지”만 선언하고
	•	실제 허용/거부 판단은 별도 계층에 위임하는 패턴을 만들 것

중요:
	•	이 인터페이스는 지금 stub이어도 된다
	•	하지만 추후 ACL/RBAC/ABAC/policy engine이 들어갈 수 있어야 한다

⸻

Step 7. router에서 권한 체크 하드코딩 금지 패턴 도입

라우터 내부에서 아래와 같은 패턴을 피하라.

비권장:

if not user:
    raise ...
if user.role != "admin":
    raise ...

권장:

actor = resolve_current_actor(...)
authorization_service.authorize(
    actor=actor,
    action="document.read",
    resource=...,
)

즉, 이번 단계에서는 권한 판단이 라우터 로직에 박히지 않게 만드는 것이 핵심이다.

⸻

Step 8. action / resource 식별 방식 초안 정리

authorization service에 넘길 action, resource 표현 방식을 정리하라.

권장 예시:
	•	document.read
	•	document.create
	•	document.update
	•	version.read
	•	node.read

resource 표현은 다음 중 하나를 검토하라.
	•	단순 resource type + optional id
	•	lightweight resource reference object
	•	tenant/document/version/node 식별자를 담는 구조체

이번 단계에서는 너무 무겁지 않은 방향이 좋다.
핵심은 다음이다.
	•	action naming이 일관적일 것
	•	resource 참조 방식이 이후 확장 가능할 것
	•	documents/versions/nodes에 바로 적용 가능할 것

⸻

Step 9. tenant scope enforcement hook point 마련

멀티테넌시 enforcement는 아직 완성하지 않더라도, authorization 단계에서 tenant scope를 고려할 수 있는 연결점을 마련하라.

예:
	•	authorize(..., tenant_scope=...)
	•	resource_ref.tenant_id
	•	RequestContext.tenant

중요:
	•	지금 tenant enforcement를 실제로 강제하지 않아도 된다
	•	하지만 나중에 “정책 판단 시 tenant 스코프를 고려할 수 있는 구조”는 있어야 한다

⸻

Step 10. 일부 endpoint에 시범 적용

documents/system 등의 대표 endpoint 몇 개에 current actor extraction 및 authorization hook point를 시범 적용하라.

권장 대상:
	•	GET /api/v1/documents
	•	POST /api/v1/documents
	•	GET /api/v1/documents/{document_id} placeholder가 있으면 거기
	•	GET /api/v1/system/info는 policy가 필요 없는 공개 endpoint로 둘지 검토 가능

목표:
	•	actor extraction dependency가 실제로 동작하는지 확인
	•	anonymous 요청과 authenticated/service 요청이 같은 구조 위에서 표현되는지 확인
	•	router가 authorization interface를 호출하는 패턴이 자리잡는지 확인

주의:
	•	system health 같은 공개 endpoint에는 authz를 강제하지 않아도 된다
	•	어떤 endpoint가 public이고 어떤 endpoint가 actor-aware인지 구분이 보이게 하라

⸻

Step 11. unauthorized / forbidden 흐름 연결

authorization hook point가 거부 시 어떤 예외를 던질지 정리하라.

권장:
	•	인증 필요하지만 없음 → authentication_required
	•	인증은 되었지만 권한 없음 → permission_denied

이 흐름이 Task I-3의 공통 오류 응답과 자연스럽게 연결되게 하라.

중요:
	•	401과 403의 구분 기준을 가능한 한 일관되게 유지할 것
	•	라우터에서 ad-hoc JSON error를 만들지 말 것

⸻

Step 12. 향후 auth 구현과의 연결 TODO 정리

이번 단계는 골격 구현이므로, 반드시 TODO를 남겨라.

예:
	•	session resolver 구체화
	•	bearer token verifier 연결
	•	internal service authentication 연결
	•	tenant membership resolution
	•	ACL/RBAC policy backend 연결

중요:
	•	지금 stub인 부분과 이후 실제 구현 대상이 명확히 구분되어야 한다

⸻

5. 구현 시 세부 요구사항

5-1. actor extraction과 authorization는 분리

이번 단계에서 가장 중요한 설계 원칙 중 하나다.
	•	actor extraction: “누가 요청했는가”를 정규화
	•	authorization: “이 actor가 이 action/resource를 할 수 있는가”를 판단

이 둘을 하나의 거대한 dependency로 섞지 말 것.

⸻

5-2. anonymous도 정상 actor 상태

anonymous는 실패 상태가 아니라 정상적인 actor 상태다.

즉:
	•	public endpoint는 anonymous 허용 가능
	•	protected endpoint는 authorization/policy에서 거절 가능
	•	actor 모델 자체는 동일

이 원칙을 지켜야 public/protected 혼합 API를 일관되게 다룰 수 있다.

⸻

5-3. auth method는 기록하되 과신하지 말 것

auth_method는 유용하지만, 이번 단계에서는 “어떤 입력으로 식별되었는가”를 기록하는 수준이면 충분하다.

예:
	•	session
	•	bearer
	•	api_key
	•	internal_service

검증이 불완전한 상태에서 auth_method만으로 강한 보안 의미를 부여하지 말 것.

⸻

5-4. policy action naming 일관화

action 값은 이후 permission matrix, audit 로그, operation tracking에도 재사용될 수 있으므로, 초기에 naming 규칙을 잡아라.

권장:
	•	<resource>.<verb>
	•	예: document.read, document.create, version.read

비권장:
	•	canReadDocument
	•	READ_DOC
	•	documents:get

⸻

5-5. resource reference는 lightweight 하게 시작

resource reference는 너무 무거운 도메인 객체일 필요 없다.

이번 단계에서는 예를 들어:
	•	resource_type
	•	resource_id
	•	parent_resource_id
	•	tenant_id(optional)

정도면 충분하다.

핵심은 authorization에 필요한 최소 맥락을 전달하는 것이다.

⸻

5-6. raw user model을 라우터 전반에 퍼뜨리지 말 것

기존에 사용자 모델이 있더라도, 라우터/서비스가 직접 ORM user 객체를 들고 다니기보다,
우선은 ActorContext와 authorization interface로 수렴시키는 방향을 검토하라.

이유:
	•	보안 경계 명확화
	•	테스트 용이성
	•	나중에 service actor나 external caller도 같은 구조로 수용 가능

⸻

6. 권장 설계 방향

Claude Code는 아래 방향을 우선 검토하라.

옵션 A. dependency 기반 actor resolution + service stub
	•	get_current_actor_context(request)
	•	authorization_service.authorize(...)

장점:
	•	FastAPI 구조와 자연스럽게 맞음
	•	router에서 패턴이 명확
	•	현재 단계에 가장 적합

옵션 B. middleware에서 actor 추정 + dependency는 보정

장점:
	•	일부 공통 처리 중앙화 가능
단점:
	•	인증 로직이 middleware에 과도하게 들어갈 수 있음

이번 단계는 dependency 기반 actor resolution + authorization service hook을 우선 검토하라.
middleware는 request context plumbing까지만 두고, actor 해석은 dependency에서 하는 편이 구조상 깔끔하다.

⸻

7. 산출물 요구

Claude Code는 작업 후 아래 내용을 보고하라.

A. 생성/수정 파일 목록

예:
	•	app/api/auth/dependencies.py
	•	app/api/auth/models.py
	•	app/api/auth/authorization.py
	•	app/api/context/models.py
	•	일부 v1 router 파일

B. actor extraction 구조 요약

예:
	•	anonymous / user / service 구분 기준
	•	어떤 입력 소스를 보는지
	•	RequestContext.actor를 어떻게 갱신하는지

C. authorization hook 구조 요약

예:
	•	authorize 인터페이스
	•	action naming 방식
	•	resource reference 방식
	•	tenant scope 연결 위치

D. endpoint 적용 예시

간단히:
	•	documents list/create에서 어떻게 연결했는지
	•	public endpoint와 protected endpoint를 어떻게 구분했는지

E. 설계 판단 근거

짧게 정리:
	•	왜 actor extraction과 authorization를 분리했는지
	•	왜 router 하드코딩 대신 service hook을 둔 것인지
	•	왜 anonymous를 정상 actor 상태로 본 것인지

F. 남겨둔 TODO

예:
	•	session/bearer/api_key 실제 검증 연결 예정
	•	tenant membership resolution 예정
	•	ACL/RBAC 정책 엔진 연결 예정
	•	admin/service caller 세분화 예정

⸻

8. 완료 조건

아래를 만족하면 완료로 본다.
	•	current actor extraction dependency 또는 동등 구조가 존재한다.
	•	anonymous / authenticated / service actor를 구분할 수 있다.
	•	RequestContext.actor 갱신 흐름이 존재한다.
	•	authorization service/policy hook point가 존재한다.
	•	router가 직접 권한 판단을 하드코딩하지 않는 패턴이 도입된다.
	•	tenant scope enforcement를 붙일 수 있는 연결 지점이 있다.
	•	일부 endpoint에 시범 적용되어 동작 흐름이 보인다.

⸻

9. 금지 사항

이번 작업에서는 다음을 하지 마라.
	•	완전한 인증 시스템을 끝내려 하지 말 것
	•	JWT/session/api key 전체를 다 구현하려 하지 말 것
	•	라우터마다 개별 auth if문을 넣지 말 것
	•	권한 판단을 ORM user.role 하드코딩으로 박지 말 것
	•	tenant enforcement를 성급히 강제하지 말 것
	•	documents 전용 auth 구조를 만들지 말 것

⸻

10. Claude Code 최종 지시문

아래 지시문을 그대로 사용해도 된다.

Task I-5를 수행하라.

목표:
- 플랫폼 API에서 current actor를 추출하고, authorization hook point를 구현한다.
- anonymous / authenticated / service actor를 공통 ActorContext 구조로 정규화한다.
- router가 직접 권한을 하드코딩하지 않고 authorization service/policy evaluator를 호출하는 패턴을 도입한다.
- tenant scope enforcement를 이후 붙일 수 있는 연결 지점을 마련한다.

반드시 지킬 원칙:
- 이번 단계는 완전한 인증/인가 구현이 아니라 actor extraction + authorization hook 골격 구현 단계다.
- actor extraction과 authorization 판단은 분리하라.
- anonymous도 정상적인 actor 상태로 취급하라.
- router에 권한 if문을 하드코딩하지 말고, action/resource 기반 authorization interface로 수렴시켜라.
- 이후 ACL/RBAC/tenant enforcement 확장을 막지 않도록 설계하라.

수행 항목:
1. 현재 인증 관련 구조를 점검한다.
2. anonymous / user / service actor 분류 기준을 정리한다.
3. current actor extraction dependency를 구현한다.
4. 인증 입력 소스(session/bearer/api_key/internal_service placeholder)를 정리한다.
5. RequestContext.actor 갱신 규칙을 만든다.
6. authorization service 또는 policy hook interface를 정의한다.
7. action/resource 표현 방식을 정한다.
8. tenant scope enforcement를 붙일 수 있는 연결 지점을 마련한다.
9. 일부 endpoint에 시범 적용해 패턴을 검증한다.
10. 공통 오류 응답과 401/403 흐름이 자연스럽게 연결되게 한다.

산출물:
- 생성/수정 파일 목록
- actor extraction 구조 요약
- authorization hook 구조 요약
- endpoint 적용 예시
- 설계 판단 근거
- 남겨둔 TODO 목록