Task 2-5 작업 지시서

주제

감사 로그(Audit Log) 체계 설계

작업 목적

Task 2-1에서 권한 주체를 정의했고, Task 2-2에서 보호 대상 리소스와 scope 구조를 정리했으며, Task 2-3에서 ACL 기반 권한 모델을 설계했고, Task 2-4에서 API 레벨 authorization enforcement 구조를 정리했다.
이번 단계에서는 그 위에 누가, 언제, 어떤 자격으로, 어떤 리소스에 대해, 어떤 행위를 했고, 그 결과가 무엇이었는지를 추적할 수 있는 감사 로그 체계를 설계한다.

이 작업의 목표는 단순한 이벤트 기록이 아니라, 보안/컴플라이언스/사후 추적/운영 조사에 사용할 수 있는 감사 가능한 구조를 설계하는 것이다.

⸻

이번 작업의 범위

이번 작업에서는 아래를 다룬다.
	1.	감사 로그의 목적 정의
	•	보안 추적
	•	컴플라이언스 대응
	•	사고 조사
	•	권한 변경 이력 확인
	•	운영 행위 추적
	2.	감사 이벤트 범주 정의
	•	인증/접근 관련
	•	문서 행위 관련
	•	권한/역할 변경 관련
	•	조직/운영 설정 변경 관련
	•	보안 민감 행위 관련
	3.	Audit Log 레코드 구조 설계
	•	actor
	•	action
	•	target resource
	•	decision/result
	•	context
	•	trace/correlation 정보
	•	before/after snapshot 필요 범위
	4.	기록 원칙 정의
	•	무엇을 반드시 기록할 것인가
	•	무엇은 기록하지 말아야 하는가
	•	성공/실패/거부 이벤트 중 무엇을 남길 것인가
	•	읽기 행위까지 어디까지 기록할 것인가
	5.	무결성/보존/조회 관점 고려
	•	수정 불가성
	•	보존 기간
	•	검색 가능성
	•	관리자/감사 담당자의 조회 범위

⸻

이번 작업에서 하지 말 것

아래는 이번 단계에서 다루지 않는다.
	•	실제 AuditLog DB 테이블 구현
	•	로그 저장 파이프라인 구현
	•	SIEM 연동 구현
	•	Activity Trace 상세 모델 설계
	•	UI 감사 로그 화면 설계
	•	개인정보 마스킹 구현 코드
	•	보존 스케줄러 구현

즉, 이번 단계는 감사 로그의 구조와 정책 설계 단계다.

⸻

설계 전제

아래 전제를 반영하여 설계할 것.
	1.	플랫폼은 범용 문서 플랫폼이며, 문서/버전/노드/첨부/댓글뿐 아니라 조직, 역할, 권한, 설정, 연동 객체까지 다룬다.
	2.	API-first 구조이므로 사용자 UI뿐 아니라 관리자 UI, 외부 API 클라이언트, 서비스 계정 행위도 감사 대상이 된다.
	3.	감사 로그는 일반 운영 로그나 디버그 로그와 구분되어야 한다.
	4.	감사 로그는 “무슨 일이 있었는가” 뿐 아니라 “누가 어떤 자격으로 했는가”를 보여줄 수 있어야 한다.
	5.	authorization 결과와 연결되어, 허용/거부 판단의 사후 설명이 가능해야 한다.
	6.	감사 로그는 나중에 삭제/변조되면 안 되므로, 일반 비즈니스 데이터와 다른 취급이 필요하다.
	7.	지나치게 모든 세부 데이터를 기록해 민감 정보를 누출하거나 저장 비용을 폭증시키지 않도록 균형이 필요하다.

⸻

Claude Code가 수행해야 할 작업

아래 순서대로 정리할 것.

1. 감사 로그의 역할 정의

먼저 아래 질문에 답할 것.
	•	감사 로그는 일반 애플리케이션 로그와 무엇이 다른가
	•	Activity Trace와는 무엇이 다른가
	•	디버그 로그, 운영 로그, 분석 로그와의 경계를 어떻게 나눌 것인가
	•	왜 감사 로그가 별도 모델이어야 하는가

그리고 감사 로그의 핵심 목적을 아래 범주로 정리할 것.
	•	보안 사고 대응
	•	내부 통제
	•	컴플라이언스 증적
	•	권한 남용 추적
	•	민감 리소스 접근 이력 확인

⸻

2. 감사 대상 이벤트 범주 정의

최소한 아래 범주를 검토할 것.

인증/세션/접근 관련
	•	login success/failure
	•	token 발급/폐기
	•	service account 사용
	•	외부 API 호출
	•	권한 거부(authorization deny)

문서/콘텐츠 관련
	•	document 생성
	•	document 수정
	•	document 삭제
	•	version 생성/복원
	•	node 수정
	•	attachment 업로드/삭제
	•	comment/annotation 생성/삭제
	•	export/download
	•	share link 생성/폐기

권한/조직 관련
	•	organization 생성/변경
	•	membership 추가/삭제
	•	role 부여/회수
	•	ACL 변경
	•	permission policy 변경

운영/보안 관련
	•	integration 설정 변경
	•	API credential 생성/폐기
	•	webhook/connector 변경
	•	감사 로그 조회
	•	보안 민감 설정 변경

각 이벤트군에 대해 아래를 정리할 것.
	•	왜 감사 대상인가
	•	성공/실패/거부 중 무엇을 남겨야 하는가
	•	MVP 필수 대상인가, 후속 확장인가

⸻

3. Audit Event Taxonomy 초안 작성

감사 이벤트 이름을 체계적으로 설계할 것.

예시:
	•	auth.login.succeeded
	•	auth.login.failed
	•	document.created
	•	document.updated
	•	document.deleted
	•	document.exported
	•	permissions.acl.updated
	•	org.membership.added
	•	admin.integration.updated

이때 아래 원칙을 검토할 것.
	•	domain.action.result 형태가 좋은가
	•	result를 필드로 분리하고 이벤트명은 action 중심으로 둘 것인가
	•	감사 이벤트 이름 규칙을 어떻게 통일할 것인가

그리고 최종 naming convention 권장안을 제시할 것.

⸻

4. Audit Log 레코드 구조 정의

최소한 아래 필드를 포함하는 개념 구조를 제안할 것.

Actor 관련
	•	actor_type
	•	actor_id
	•	acting_membership_id 또는 acting_org_id
	•	acting_role_set 또는 evaluated_role_context
	•	authentication_method
	•	request_source (user UI / admin UI / external API / service account)

Action 관련
	•	event_type
	•	action_category
	•	result (success / denied / failed)
	•	denial_reason_category 또는 failure_reason_category

Target 관련
	•	target_resource_type
	•	target_resource_id
	•	target_parent_type / target_parent_id
	•	target_org_id
	•	target_scope

Context 관련
	•	request_id
	•	correlation_id / trace_id
	•	ip / user agent 필요 여부
	•	endpoint / method
	•	timestamp

Change 관련
	•	before_summary
	•	after_summary
	•	changed_fields
	•	policy_reason / business_reason

각 필드에 대해 아래를 정리할 것.
	•	왜 필요한가
	•	필수인지 선택인지
	•	민감도 고려가 필요한지
	•	MVP 포함 여부

⸻

5. 무엇을 기록하고 무엇을 기록하지 않을지 원칙 정리

아래 질문에 답할 것.
	•	document read도 모두 감사 로그로 남겨야 하는가
	•	list/search 조회는 어떤 수준까지 남겨야 하는가
	•	단순 페이지 진입도 감사 대상인가
	•	실패한 로그인과 권한 거부는 반드시 남겨야 하는가
	•	before/after 전체 snapshot을 저장하면 어떤 위험이 있는가
	•	민감 문서 본문을 감사 로그에 넣어도 되는가

그리고 아래 형태로 원칙을 정리할 것.
	•	반드시 기록할 행위
	•	조건부 기록할 행위
	•	기록하지 않을 행위
	•	요약만 기록할 행위
	•	콘텐츠 원문을 넣지 말아야 할 항목

⸻

6. 성공 / 실패 / 거부 이벤트 기록 원칙

아래 세 결과를 구분하여 설계할 것.
	•	success
	•	failed
	•	denied

그리고 아래 질문에 답할 것.
	•	시스템 오류로 실패한 경우와 권한 거부를 어떻게 구분할 것인가
	•	허용되었지만 후속 비즈니스 규칙으로 실패한 경우는 어떻게 남길 것인가
	•	존재 은닉을 위해 API 응답은 404를 주더라도, 내부 audit에는 denied로 남겨야 하는가
	•	create/update/delete 뿐 아니라 permission change 실패도 남겨야 하는가

최종적으로 결과 코드 체계를 제안할 것.

⸻

7. Before/After 및 변경 요약 원칙

변경 행위에 대해 어떤 수준의 diff를 남길지 정리할 것.

최소한 아래를 검토할 것.
	•	changed_fields만 저장
	•	before_summary / after_summary 저장
	•	full snapshot 저장
	•	정책 객체나 ACL은 full snapshot이 필요한가
	•	문서 본문 수정은 전체 텍스트를 감사 로그에 넣을 것인가

그리고 아래를 정리할 것.
	•	일반 콘텐츠 객체의 권장 기록 수준
	•	권한/정책 객체의 권장 기록 수준
	•	보안 민감 객체의 권장 기록 수준
	•	full snapshot을 제한해야 하는 이유

⸻

8. 무결성, 보존, 접근 통제 원칙

아래 질문에 답할 것.
	•	감사 로그는 수정/삭제 가능해야 하는가
	•	soft delete가 허용되어야 하는가
	•	보존 기간은 일반 데이터와 다르게 가야 하는가
	•	감사 로그 열람 권한은 누구에게 줄 것인가
	•	조직 관리자와 플랫폼 감사자에게 같은 범위를 보여줘도 되는가
	•	감사 로그 자체 조회도 감사 대상이 되어야 하는가

그리고 아래를 정리할 것.
	•	append-only 원칙
	•	보존 정책 초안
	•	열람 권한 원칙
	•	감사 로그 조회의 재감사 필요 여부

⸻

9. Authorization Decision과 Audit Log의 연결 방식

Task 2-4와 연결해서 아래를 정리할 것.
	•	authorization service가 반환한 decision metadata 중 무엇을 audit에 포함할 것인가
	•	evaluated principal context를 남겨야 하는가
	•	어떤 permission으로 평가했는지 남겨야 하는가
	•	inherited allow인지, direct ACL allow인지까지 남겨야 하는가
	•	deny reason category는 어떻게 표준화할 것인가

즉, “왜 허용/거부되었는가”를 사후 해석할 수 있도록 필요한 연결 필드를 정의할 것.

⸻

10. MVP 감사 로그 범위 제한안 제시

MVP에서 반드시 구현해야 하는 감사 로그 범위를 제안할 것.

최소한 아래를 검토할 것.

MVP 필수
	•	login success/failure
	•	document create/update/delete
	•	document export/download
	•	permission/role/ACL 변경
	•	organization/membership 변경
	•	admin 설정 변경
	•	authorization denied on sensitive resources

후속 확장
	•	모든 read 이벤트
	•	세밀한 list/search query audit
	•	integration payload detail
	•	workflow action detail
	•	anomaly detection용 확장 필드

중요:
초기에는 반드시 필요한 보안/컴플라이언스 이벤트 중심으로 제한할 것.

⸻

11. 다음 단계 입력 자료 형태로 정리

Task 2-6 행위 추적(Activity Trace) 모델 설계로 넘기기 위해 아래를 정리할 것.
	•	감사 로그와 activity trace의 차이
	•	감사 로그는 어떤 이벤트를 강하게 남기고
	•	activity trace는 어떤 흐름/탐색/사용성 정보를 남기는지
	•	두 시스템이 공유할 수 있는 correlation id / actor / resource 식별자

⸻

산출물 형식

이번 작업 결과물은 설계 문서 초안이어야 하며, 아래 구조를 따를 것.

Phase 2 - Task 2-5

1. 목표

2. 설계 전제

3. 감사 로그의 역할과 범위

4. 감사 대상 이벤트 범주

5. Audit Event Taxonomy 초안

6. Audit Log 레코드 구조

7. 기록 대상 / 비대상 원칙

8. success / failed / denied 기록 원칙

9. before/after 및 변경 요약 원칙

10. 무결성 / 보존 / 접근 통제 원칙

11. Authorization Decision 연계 방식

12. MVP 감사 로그 범위 제한안

13. 다음 Task로 넘길 결정사항

14. 오픈 이슈

⸻

의사결정 원칙

설계 중 아래 원칙을 반드시 지킬 것.
	1.	감사 로그는 별도 목적을 가진다
	•	디버깅이나 분석용 로그와 혼합하지 말 것
	2.	누가 어떤 자격으로 했는지 남길 것
	•	actor identity뿐 아니라 acting context를 중요하게 볼 것
	3.	콘텐츠 원문 저장 최소화
	•	감사 목적상 필요한 요약은 남기되, 민감 정보 과다 저장은 피할 것
	4.	거부와 실패를 구분할 것
	•	denied와 failed를 섞지 말 것
	5.	감사 로그 자체도 보호 대상
	•	append-only와 열람 통제가 중요함
	6.	MVP는 필수 이벤트 중심
	•	모든 것을 다 남기려 하지 말고, 보안/컴플라이언스 핵심 이벤트부터 시작할 것

⸻

기대 결과

이 작업이 끝나면 최소한 아래가 확정되어 있어야 한다.
	•	감사 로그가 다뤄야 하는 이벤트 범주
	•	audit event naming 규칙
	•	audit record의 핵심 필드 구조
	•	success / failed / denied 기록 기준
	•	before/after 및 diff 기록 수준 원칙
	•	감사 로그 무결성/보존/열람 원칙
	•	authorization decision과의 연결 방식
	•	Task 2-6과 구분되는 감사 로그의 역할

⸻

금지 사항
	•	일반 애플리케이션 로그와 감사 로그를 혼합 설계하지 말 것
	•	문서 본문 전체를 감사 로그에 기본 저장하려 하지 말 것
	•	Activity Trace까지 이번 단계에서 같이 설계 완료하려 하지 말 것
	•	SIEM/외부 보안 솔루션 구현으로 넘어가지 말 것
	•	UI 화면 중심으로 이벤트를 정의하지 말 것

⸻

작업 완료 판단 기준

아래 조건을 만족하면 완료로 본다.
	•	감사 로그의 역할과 Activity Trace와의 차이가 설명되어 있다
	•	감사 대상 이벤트 범주와 naming 규칙이 정리되어 있다
	•	audit record 구조가 actor/action/target/context/result 기준으로 정의되어 있다
	•	기록 대상과 비대상 원칙이 구분되어 있다
	•	무결성/보존/열람 원칙이 제시되어 있다
	•	Task 2-6으로 넘길 수 있는 경계와 연계점이 정리되어 있다
