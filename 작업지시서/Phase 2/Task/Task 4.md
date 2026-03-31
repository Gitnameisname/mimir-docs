Task 2-4 작업 지시서

주제

API 레벨 Authorization Enforcement 설계

작업 목적

Task 2-1에서 권한 주체를 정의했고, Task 2-2에서 보호 대상 리소스와 scope 구조를 정리했으며, Task 2-3에서 RBAC + ACL 기반 권한 모델과 평가 원칙을 설계했다.
이번 단계에서는 이를 실제 시스템 요청 흐름에 연결하여, API 요청이 들어왔을 때 어디서, 어떤 순서로, 어떤 책임 분리 하에 권한 검사를 수행할 것인지를 설계한다.

이 작업의 목표는 코드를 작성하는 것이 아니라, 이후 백엔드 구현 단계에서 일관되게 적용할 수 있는 authorization enforcement 구조와 계층 책임 원칙을 확정하는 것이다.

⸻

이번 작업의 범위

이번 작업에서는 아래를 다룬다.
	1.	API 요청 단위 권한 검사 흐름 정의
	•	인증 이후 authorization 수행 위치
	•	principal 식별과 권한 평가 순서
	•	resource loading 과 authorization 의 결합 방식
	2.	계층별 책임 분리 설계
	•	router/controller
	•	service/application layer
	•	repository/data access layer
	•	authorization service/policy evaluator
	3.	권한 검사 방식 정의
	•	사전 검사(pre-check)
	•	리소스 로드 후 검사(post-load check)
	•	목록 조회(list/filter) 시의 권한 처리
	•	부분 마스킹 vs 전체 거부
	4.	에러/응답 정책 정리
	•	401 / 403 / 404 처리 원칙
	•	존재 은닉 여부
	•	감사 로그와의 연결
	5.	공통 enforcement 패턴 정의
	•	create / read / update / delete / list / admin action
	•	bulk action
	•	nested resource 접근
	•	API-first 외부 클라이언트 대응

⸻

이번 작업에서 하지 말 것

아래는 이번 단계에서 다루지 않는다.
	•	실제 middleware/dependency 코드 작성
	•	framework별 decorator/guard 구현
	•	DB row-level security 구현
	•	감사 로그 persistence 상세 구현
	•	API 스펙 문서 전체 작성
	•	UI 메뉴 제어 구현

즉, 이번 단계는 API 권한 enforcement 아키텍처 설계 단계다.

⸻

설계 전제

아래 전제를 반영하여 설계할 것.
	1.	플랫폼은 API-first 범용 문서 플랫폼이다.
	2.	사용자 UI, 관리자 UI, 외부 애플리케이션, AI 도구, 서비스 계정 등이 모두 API를 통해 접근할 수 있다.
	3.	인증(authentication)과 권한(authorizaton)은 분리되지만, API에서는 연속된 요청 처리 흐름 안에서 함께 동작해야 한다.
	4.	권한 모델은 RBAC + resource-scoped ACL 구조를 따른다.
	5.	감사 가능성을 위해 “누가 어떤 리소스에 대해 어떤 permission으로 요청했고 왜 허용/거부되었는가”를 기록 가능한 구조여야 한다.
	6.	과도하게 모든 곳에 권한 로직이 흩어지지 않도록, 공통 authorization service 중심 구조를 우선 검토할 것.
	7.	향후 approval workflow, external share, service account, integration API, admin API까지 확장 가능해야 한다.

⸻

Claude Code가 수행해야 할 작업

아래 순서대로 정리할 것.

1. Authorization Enforcement의 위치 정의

먼저 아래 질문에 답할 것.
	•	인증 이후 authorization은 어디서 수행되어야 하는가
	•	router/controller에서 전부 처리하면 어떤 문제가 있는가
	•	service layer에서만 처리하면 어떤 문제가 있는가
	•	repository까지 권한 판단이 내려가면 어떤 문제가 있는가
	•	중앙 Authorization Service를 두는 방식의 장단점은 무엇인가

그리고 아래 관점으로 권장안을 제시할 것.
	•	router/controller의 책임
	•	service/application layer의 책임
	•	authorization service의 책임
	•	repository/data access layer의 책임

중요:
권한 판단 로직이 중복되지 않도록 하는 구조가 핵심이다.

⸻

2. API 요청 처리 흐름 초안 작성

다음과 같은 일반 흐름을 기준으로, 단계별 처리 순서를 설계할 것.

예시:
	1.	인증된 principal 확보
	2.	요청 컨텍스트 구성
	3.	대상 resource type / action 도출
	4.	필요한 상위 리소스 또는 소유 범위 확인
	5.	authorization service 호출
	6.	허용 시 service 로직 수행
	7.	결과 반환 및 감사 이벤트 기록

이 흐름을 아래 요청 유형별로 나누어 정리할 것.
	•	Create 요청
	•	Single resource read 요청
	•	Update 요청
	•	Delete 요청
	•	List/Search 요청
	•	Admin action 요청
	•	Nested resource 요청
예: /documents/{id}/comments, /documents/{id}/versions/{version_id}

각 요청 유형에서 아래를 설명할 것.
	•	어떤 시점에 권한 검사가 필요한가
	•	어떤 resource/action으로 매핑되는가
	•	pre-check와 post-load check 중 어느 방식이 적절한가
	•	감사 로그에 어떤 decision context를 남겨야 하는가

⸻

3. Pre-check / Post-load check 원칙 설계

다음 두 방식을 비교 검토할 것.
	•	Pre-check
	•	리소스 생성 전
	•	상위 컨테이너 접근 권한 기반
	•	존재 여부와 무관하게 권한 판단 가능
	•	Post-load check
	•	특정 리소스를 실제로 로드한 뒤
	•	ownership, state, resource-specific ACL 반영 가능

그리고 아래 질문에 답할 것.
	•	create는 어떤 기준으로 체크해야 하는가
	•	document read/update/delete는 언제 post-load가 필요한가
	•	nested resource는 상위 리소스와 하위 리소스를 모두 봐야 하는가
	•	resource state(draft, locked, archived 등)가 권한 판단에 영향을 주면 어느 계층에서 반영해야 하는가

최종적으로 아래 형태로 정리할 것.
	•	pre-check가 적합한 요청 유형
	•	post-load check가 필요한 요청 유형
	•	두 방식을 함께 써야 하는 요청 유형
	•	남용 시 문제점

⸻

4. 목록 조회(List/Search) 권한 처리 원칙

이 부분은 매우 중요하다. 아래 질문에 답할 것.
	•	list API에서 권한 없는 리소스를 어떻게 처리할 것인가
	•	필터링된 결과만 보여줄 것인가
	•	카운트(total count)에 권한 없는 항목을 포함할 것인가
	•	search 결과에서 snippet/metadata만 일부 보이는 것이 허용되는가
	•	부분 마스킹과 완전 제외 중 어떤 원칙이 적절한가

그리고 아래 시나리오를 검토할 것.
	•	사용자가 접근 가능한 문서 목록 조회
	•	조직 관리자 전용 목록 조회
	•	감사 담당자의 감사 로그 목록 조회
	•	검색 결과에서 제목만 보이고 본문은 숨기는 경우의 허용 여부

최종적으로 아래를 제시할 것.
	•	list endpoint 기본 원칙
	•	search endpoint 기본 원칙
	•	count / aggregate 응답 원칙
	•	부분 공개 허용 여부 및 조건

⸻

5. 오류 응답 정책 정의

아래 상태 코드를 중심으로 정책을 설계할 것.
	•	401 Unauthorized
	•	403 Forbidden
	•	404 Not Found

그리고 아래 질문에 답할 것.
	•	인증 실패와 권한 부족은 어떻게 구분할 것인가
	•	존재하지만 접근 권한이 없는 리소스에 대해 403을 줄지 404를 줄지
	•	민감 리소스에서는 존재 은닉이 필요한가
	•	admin API와 일반 API의 응답 정책을 다르게 둘 필요가 있는가
	•	오류 응답에서 어느 정도까지 이유를 설명할 것인가

최종적으로 아래를 정리할 것.
	•	일반 콘텐츠 리소스 응답 원칙
	•	보안 민감 리소스 응답 원칙
	•	운영 리소스 응답 원칙
	•	감사 기록 시 내부적으로 남겨야 할 실제 거부 사유

⸻

6. 공통 Authorization Context 모델 정의

Authorization Service가 판단하기 위해 어떤 입력을 받아야 하는지 정리할 것.

최소한 아래 요소를 검토할 것.
	•	authenticated principal
	•	acting membership / organization context
	•	resource type
	•	resource id
	•	parent resource 정보
	•	action / permission
	•	request source (UI / external API / service account)
	•	resource state
	•	optional target subject (예: 권한 변경의 대상 사용자)

그리고 아래를 정리할 것.
	•	필수 입력
	•	선택 입력
	•	request context에서 자동 추론 가능한 것
	•	service layer가 채워야 하는 것
	•	감사 로그와 공유할 수 있는 decision metadata

⸻

7. 계층별 책임 분리안 제시

아래 계층별로 역할을 분리해서 설명할 것.

Router / Controller
	•	입력 해석
	•	path parameter 기반 resource identity 수집
	•	인증된 사용자 컨텍스트 확보
	•	필요한 service 호출

Service / Application Layer
	•	비즈니스 유스케이스 실행
	•	authorization service 호출 지점 관리
	•	resource loading 및 state 반영
	•	감사 이벤트 생성 트리거

Authorization Service
	•	RBAC + ACL 평가
	•	inherited scope / exception rule 해석
	•	allow / deny + reason 반환

Repository / Data Access
	•	권한 판단 최소화
	•	필요한 데이터 조회
	•	list filtering 지원 필요 여부 검토

그리고 아래 질문도 답할 것.
	•	repository에서 권한 필터 쿼리를 같이 넣어야 하는가
	•	service layer에서 후처리 필터링하면 어떤 문제가 있는가
	•	list API에서 authorization-aware query가 필요한가

⸻

8. 공통 Enforcement 패턴 카탈로그 작성

최소한 아래 패턴별로 enforcement 방법을 정리할 것.
	•	Create under parent
	•	Read single resource
	•	Update mutable resource
	•	Delete resource
	•	List resources in scope
	•	Manage permissions on resource
	•	Perform admin-only operation
	•	Bulk update/delete
	•	Export / publish / approve
	•	Nested child resource management

각 패턴에 대해 아래를 작성할 것.
	•	대표 API 예시
	•	필요한 principal/resource/action 정보
	•	권장 검사 시점
	•	주의할 점
	•	감사 포인트

⸻

9. 외부 API / 서비스 계정 대응 원칙

API-first 플랫폼이므로 아래를 검토할 것.
	•	서비스 계정 요청은 일반 사용자 요청과 무엇이 다른가
	•	organization scoped token/client는 어떻게 해석해야 하는가
	•	외부 integration이 요청할 때 acting principal을 어떻게 남겨야 하는가
	•	user delegated access와 service account access를 구분해야 하는가

그리고 MVP 기준에서 어느 수준까지 고려할지 정리할 것.

⸻

10. 다음 단계 입력 자료 형태로 정리

Task 2-5 감사 로그 설계로 넘기기 위해 아래를 정리할 것.
	•	authorization decision 시 남겨야 할 핵심 metadata
	•	누가 / 어떤 자격으로 / 어떤 resource / 어떤 action / 어떤 결과
	•	denial reason category
	•	correlation id / request id 필요 여부
	•	post-event audit와 runtime authorization decision log의 차이

⸻

산출물 형식

이번 작업 결과물은 설계 문서 초안이어야 하며, 아래 구조를 따를 것.

Phase 2 - Task 2-4

1. 목표

2. 설계 전제

3. Authorization Enforcement 위치와 책임

4. API 요청 처리 흐름 초안

5. Pre-check / Post-load check 원칙

6. List/Search 권한 처리 원칙

7. 오류 응답 정책

8. Authorization Context 모델

9. 계층별 책임 분리안

10. 공통 Enforcement 패턴 카탈로그

11. 외부 API / 서비스 계정 대응 원칙

12. 다음 Task로 넘길 결정사항

13. 오픈 이슈

⸻

의사결정 원칙

설계 중 아래 원칙을 반드시 지킬 것.
	1.	권한 로직 중앙화
	•	권한 판단 기준은 최대한 Authorization Service에 모을 것
	2.	비즈니스 로직과 권한 로직 분리
	•	서비스 레이어는 orchestration을 담당하되, 정책 자체는 별도 계층에서 판단할 것
	3.	목록 조회는 필터링 우선
	•	권한 없는 리소스는 기본적으로 결과에서 제외하는 방향을 우선 검토할 것
	4.	민감 리소스는 존재 은닉 고려
	•	보안 민감 리소스는 403보다 404가 더 적절할 수 있음
	5.	감사 가능성 확보
	•	allow/deny 결정은 사후 설명 가능한 metadata를 남길 수 있어야 함
	6.	API-first 일관성 유지
	•	UI, 외부 앱, AI 도구, 서비스 계정 모두 동일한 authorization 원칙 아래에서 동작해야 함

⸻

기대 결과

이 작업이 끝나면 최소한 아래가 확정되어 있어야 한다.
	•	API 레벨에서 권한 검사를 어디서 수행할지
	•	pre-check와 post-load check를 어떻게 나눌지
	•	list/search에서 권한 없는 리소스를 어떻게 처리할지
	•	401/403/404 응답 원칙
	•	authorization service가 받아야 하는 context 구조
	•	계층별 책임 분리 원칙
	•	다음 감사 로그 설계에 넘길 authorization decision metadata 기준

⸻

금지 사항
	•	실제 middleware/decorator/dependency 코드를 작성하지 말 것
	•	모든 endpoint를 일일이 구현 수준으로 나열하지 말 것
	•	DB row-level security로 바로 점프하지 말 것
	•	UI 전용 예외 규칙을 먼저 만들지 말 것
	•	감사 로그 상세 스키마를 이번 단계에서 완성하려 하지 말 것

⸻

작업 완료 판단 기준

아래 조건을 만족하면 완료로 본다.
	•	authorization enforcement 위치와 책임이 문서로 설명되어 있다
	•	API 요청 유형별 검사 흐름이 정리되어 있다
	•	list/search와 오류 응답 정책이 정의되어 있다
	•	authorization context와 계층별 책임 분리가 제시되어 있다
	•	Task 2-5 감사 로그 설계에 필요한 decision metadata 기준이 정리되어 있다
