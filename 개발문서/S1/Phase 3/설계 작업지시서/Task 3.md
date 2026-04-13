좋아. 아래는 Task 3-3. 인증/인가 구조의 API 계층 반영 설계에 대한 Claude Code 작업 지시서다.

⸻

Task 3-3. 인증/인가 구조의 API 계층 반영 설계

Claude Code 작업 지시서

1. 작업명

Phase 3 - Task 3-3. 인증/인가 구조의 API 계층 반영 설계

⸻

2. 작업 목적

플랫폼 API 계층에서 인증(Authentication)과 인가(Authorization)가 어떤 방식으로 반영되어야 하는지 설계 문서로 정리하라.

이번 작업의 목적은 로그인 기능을 구현하거나 토큰 포맷을 확정하는 것이 아니다.
핵심은 다음이다.
	•	API 요청이 어떤 보안 문맥을 가져야 하는지 정의
	•	인증된 주체와 조직/테넌트 스코프를 API 계층에서 어떻게 다룰지 정리
	•	실제 권한 판단이 어느 계층에서 수행되어야 하는지 경계 설정
	•	User/Admin/External/Service/AI 소비자별 인증/인가 요구를 분리
	•	이후 ACL enforcement, 감사 추적, 외부 시스템 연동, API 키/서비스 토큰, 웹훅, 비동기 작업까지 수용 가능한 기준선 마련

즉, 이 문서는 API 보안 문맥 설계 기준 문서여야 하며,
Phase 2의 권한 모델과 Phase 3의 REST 구조를 API 레벨에서 연결하는 역할을 해야 한다.

⸻

3. 반드시 반영할 배경

이 플랫폼은 다음 특성을 가진다.
	•	범용 문서 플랫폼
	•	조직/사용자/역할/ACL 구조 존재
	•	API-first 원칙
	•	User UI / Admin UI 분리
	•	외부 애플리케이션 연동 필요
	•	AI/RAG 도구도 공식 API 소비자
	•	감사 추적과 행위 추적 필요
	•	이후 웹훅, 비동기 작업, 검색, RAG ingestion, automation까지 확장 예정

이전 단계에서 이미 다음이 정리되었다고 가정하라.
	•	플랫폼 API는 공식 계약 계층이다.
	•	리소스 중심 REST 구조를 우선한다.
	•	보안은 UI가 아니라 API 레벨에서 강제되어야 한다.
	•	AI도 비공식 우회자가 아니라 정식 소비자다.
	•	운영 추적성과 auditability가 중요하다.

또한 Phase 2에서 다음이 이미 설계되어 있다고 가정하고 연결하라.
	•	사용자/조직/역할 모델
	•	ACL 기반 권한 구조
	•	API 레벨 권한 enforcement 필요
	•	감사 로그 체계
	•	행위 추적 모델

⸻

4. 이번 작업의 핵심 질문

이번 문서에서는 최소한 아래 질문들에 답할 수 있어야 한다.
	1.	API 요청은 어떤 인증 주체를 가질 수 있는가?
	2.	사람 사용자, 관리자, 외부 시스템, 내부 서비스, AI agent는 동일한 인증 모델을 공유하는가?
	3.	인증과 인가는 어디서 분리되어야 하는가?
	4.	API 계층은 무엇을 확인하고, 권한 엔진/서비스 계층은 무엇을 판단해야 하는가?
	5.	조직/테넌트 문맥은 path, token, request context 중 어디서 어떻게 다뤄야 하는가?
	6.	리소스 단위 ACL과 API 단위 접근 통제는 어떤 관계를 가지는가?
	7.	관리자 API, 내부 시스템 API, 외부 공개 API를 같은 구조 안에서 어떻게 안전하게 구분할 것인가?
	8.	감사 추적을 위해 API 요청에 어떤 보안/행위 메타데이터가 남아야 하는가?
	9.	웹훅, job, background worker, service-to-service 요청은 어떤 신원 모델로 처리해야 하는가?
	10.	향후 실제 인증 방식이 바뀌어도 API 보안 구조가 유지되려면 어떤 추상화가 필요한가?

⸻

5. 상세 작업 요구사항

5-1. 문서 작성

아래 산출물을 작성하라.

필수 산출물
	•	Task_authn_authz_api_design.md

문서에 반드시 포함할 섹션
	1.	문서 목적
	2.	인증/인가의 역할 구분
	3.	API 보안 설계의 상위 원칙
	4.	API 소비자별 인증 주체 모델
	5.	Request Security Context 설계
	6.	조직/테넌트 스코프 반영 원칙
	7.	API Layer와 Authorization Layer의 책임 분리
	8.	리소스 ACL과 API 접근 통제의 연결 방식
	9.	관리자/외부/내부/서비스 API 구분 원칙
	10.	비동기 작업, 웹훅, 이벤트, 서비스 계정의 보안 문맥
	11.	감사 추적 및 보안 관측성 연계
	12.	후속 Task에 전달할 설계 기준
	13.	결론

⸻

5-2. 인증과 인가의 역할 구분

문서 초반에서 다음을 분명히 구분하라.
	•	Authentication
	•	요청 주체가 누구인지 확인
	•	사람 사용자/서비스 계정/시스템/웹훅 송신자 등 식별
	•	Authorization
	•	이 주체가 특정 리소스/행동에 대해 무엇을 할 수 있는지 판단

반드시 아래 내용을 포함하라.
	•	인증 성공이 곧 접근 허용을 의미하지 않는다.
	•	인가는 role 기반, ACL 기반, 조직 스코프, 리소스 상태, 정책 조건 등을 함께 고려할 수 있다.
	•	API 계층은 인증 결과를 요청 문맥으로 정리해야 하지만, 권한 판단 그 자체를 전부 직접 구현하지는 않는다.
	•	인증/인가를 혼동하지 않도록 계층을 분리해야 한다.

⸻

5-3. API 보안 설계의 상위 원칙 정의

문서에는 최소한 아래 원칙을 포함하라.
	•	Default deny
	•	Least privilege
	•	Explicit actor context
	•	Tenant-aware access
	•	Separation of authentication and authorization
	•	API-level enforcement
	•	No UI-only security
	•	Auditable access
	•	Service identity support
	•	Extensible credential model

각 원칙마다 다음을 설명하라.
	•	원칙 의미
	•	왜 필요한지
	•	실제 API 설계에서 어떤 판단을 유도하는지

예:
	•	Default deny → 인증 또는 권한 문맥이 불충분하면 허용보다 거부가 기본
	•	Explicit actor context → 모든 요청은 actor, tenant, auth method, request id 같은 문맥을 가져야 함
	•	Service identity support → 사람 이외 소비자도 정식 보안 모델 안에 포함

⸻

5-4. API 소비자별 인증 주체 모델 정리

다음 소비자/주체 유형을 나누어 설명하라.
	•	일반 사용자
	•	관리자 사용자
	•	외부 애플리케이션
	•	내부 서비스
	•	background worker / async job runner
	•	webhook caller / receiver 맥락
	•	AI / RAG / Agent consumer
	•	필요 시 service account / machine identity

각 유형에 대해 정리할 내용:
	•	주체의 성격
	•	사람이 직접 사용하는지 여부
	•	어떤 인증 수단이 적합한지 후보 수준에서만 기술
	•	권한 범위를 어떻게 제한해야 하는지
	•	actor context에 어떤 식별 정보가 필요할지
	•	감사 로그에 무엇이 남아야 하는지

중요:
여기서 구체 프로토콜은 확정하지 말고,
주체 모델과 요구사항을 중심으로 정리하라.

⸻

5-5. 인증 수단 후보 정리

구체 방식 확정은 아니지만, API 계층 설계를 위해 아래 인증 수단 후보들을 비교 관점으로 정리하라.
	•	session / cookie 기반
	•	bearer token 기반
	•	service token 기반
	•	API key 사용 여부
	•	signed webhook secret 방식
	•	mTLS 같은 강한 내부 통신 방식 고려 여부

각 방식마다 다음 수준만 정리하라.
	•	적합한 사용 대상
	•	장점
	•	주의점
	•	이 플랫폼에서의 적합성
	•	어떤 경우 비권장인지

중요:
최종 채택 결론보다,
이 플랫폼에서 어떤 인증 수단을 어떤 소비자에게 연결하는 것이 자연스러운지 구조적으로 정리하라.

⸻

5-6. Request Security Context 설계

이번 문서의 핵심 중 하나다.
API 요청이 통과한 뒤 내부 계층으로 전달되어야 하는 보안 문맥(Security Context) 을 설계하라.

최소한 아래 항목을 검토하라.
	•	actor_id
	•	actor_type
	•	tenant_id / organization_id
	•	role set 또는 granted authorities
	•	auth method
	•	session id 또는 credential id
	•	impersonation 여부
	•	service identity 여부
	•	request id / trace id
	•	source application/client id
	•	scope 또는 delegated permission 정보
	•	human initiated / system initiated 여부

중요:
이 문맥은 API 핸들러에서 service layer/authorization layer/audit layer로 전달될 수 있어야 한다는 점을 분명히 하라.

⸻

5-7. 조직/테넌트 스코프 반영 원칙

다음을 정리하라.
	•	이 플랫폼이 tenant-aware한 API 구조를 가져야 하는 이유
	•	조직/테넌트 문맥을 path에 강제할지, credential/context에서 해석할지, 혼합할지
	•	actor가 여러 조직에 속할 수 있는 경우 처리 방향
	•	조직 범위를 벗어나는 접근을 어떻게 차단해야 하는지
	•	cross-tenant access는 기본 금지인지 여부
	•	관리자 또는 시스템 계정이 예외적으로 넓은 범위를 가질 수 있는지

문서에서는 구체 엔드포인트 설계보다는 정책 방향을 정리하라.

⸻

5-8. API Layer와 Authorization Layer의 책임 분리

반드시 이 부분을 명확히 쓰라.

API Layer가 해야 할 일
	•	인증 정보 파싱/검증 결과 수용
	•	request security context 구성
	•	최소한의 스코프/형식/요청 조건 검증
	•	적절한 authorization layer 호출
	•	결과에 따라 허용/거부 응답 생성
	•	감사/추적용 메타데이터 전달

Authorization Layer가 해야 할 일
	•	role / membership / ACL / resource ownership / policy 조건 기반 판단
	•	액션 단위 허용 여부 계산
	•	리소스 상태나 관계까지 고려한 정책 적용
	•	정책 변경 가능성을 API 코드에서 분리

중요:
“API 코드 내부에 권한 if문이 흩어지는 구조”를 지양하는 방향을 명시하라.

⸻

5-9. 리소스 ACL과 API 접근 통제 연결 방식

Phase 2의 ACL 모델과 연결하라.

다음을 문서화하라.
	•	API 접근 통제는 endpoint 자체 접근과 리소스별 접근 두 층위를 가질 수 있음
	•	예: /documents/{id} 접근 가능 여부와, 그 문서에서 edit/share/publish 가능한지는 다를 수 있음
	•	역할 기반 기본 권한 + 리소스별 ACL override 조합 가능성
	•	문서/버전/노드/첨부/감사 로그 등 리소스별 정책 차이
	•	목록 조회에서도 “보이는 리소스만 반환” 정책이 필요할 수 있음
	•	UI에서 숨기는 것으로 끝나지 않고 API가 실제 enforcement를 해야 함

즉, endpoint guard와 resource authorization의 관계를 정리하라.

⸻

5-10. 관리자 API / 외부 API / 내부 API 구분 원칙

다음을 검토하라.
	•	관리자 기능을 같은 플랫폼 API 안에서 권한으로만 구분할지
	•	/admin 같은 별도 namespace를 둘지
	•	내부 시스템 간 API는 공개 API와 같은 표준을 따를지
	•	외부 통합용 API와 내부 UI용 API를 완전히 분리할지, 공통 API를 우선할지
	•	민감 기능은 authz 뿐 아니라 별도 exposure control이 필요한지

권장 방향:
	•	공통 플랫폼 API를 우선
	•	다만 운영성/위험도/노출 범위가 크게 다른 경우 namespace 또는 exposure boundary를 둘 수 있음

이 기준을 문서화하라.

⸻

5-11. 비동기 작업, 웹훅, 이벤트, 서비스 계정의 보안 문맥

이번 Phase에서 중요한 확장 포인트다.
다음을 반드시 다뤄라.

비동기 작업
	•	누가 작업을 시작했는지
	•	background job이 누구의 권한으로 동작하는지
	•	원요청 actor context를 위임/축약 저장할지
	•	시스템 승격 권한을 어떻게 제한할지

웹훅
	•	외부로 보내는 웹훅과 외부에서 받는 웹훅의 신뢰 모델 차이
	•	수신 검증 방식 원칙
	•	발신자 검증 실패 시 처리 방향

내부 서비스
	•	service-to-service identity 필요성
	•	사람 사용자 토큰 재사용보다 서비스 신원 분리 원칙

AI / Agent
	•	사용자를 대신해 동작하는지
	•	독립 서비스 계정처럼 동작하는지
	•	delegated action과 autonomous action 구분 필요성

⸻

5-12. 권한 실패 및 보안 오류 응답 원칙

세부 에러 포맷은 Task 3-10에서 더 다루겠지만, 이번 문서에서도 방향을 정리하라.

반드시 담을 내용:
	•	인증 실패와 권한 부족을 구분해야 함
	•	401 vs 403 개념 구분
	•	리소스 존재 여부를 숨겨야 하는 경우 고려
	•	과도한 내부 정책 정보를 노출하지 않기
	•	감사 로그에는 충분한 정보가 남아야 하지만 외부 응답은 최소화해야 함

⸻

5-13. 감사 추적 및 보안 관측성 연계

다음을 정리하라.
	•	어떤 요청이 누구에 의해 수행되었는지 추적 가능해야 함
	•	actor, tenant, action, target resource, auth method, result, request id 등을 연결할 필요
	•	보안 로그와 비즈니스 감사 로그의 역할 차이
	•	실패한 접근 시도도 일부 기록 대상이 될 수 있음
	•	impersonation, delegated action, service action 같은 고위험 문맥은 추가 추적 필요

⸻

5-14. 후속 Task에 전달할 설계 기준

문서 마지막에 아래 연결 포인트를 정리하라.

예:
	•	Task 3-4에서는 인증 변화에도 버전 전략이 무너지지 않아야 함
	•	Task 3-5에서는 security context와 trace metadata를 응답 규약과 연결해야 함
	•	Task 3-6에서는 목록 조회 시 authorization-aware filtering을 고려해야 함
	•	Task 3-7에서는 idempotency와 actor context 연결이 필요함
	•	Task 3-8에서는 webhook/job/event 보안 문맥을 구체화해야 함
	•	Phase 4 이후 실제 구현에서 auth middleware/dependency와 policy engine 경계를 반영해야 함

⸻

6. 산출물

아래 파일을 작성하라.

필수 산출물
	•	Task4_authn_authz_api_design.md

문서 성격
	•	설계 기준 문서
	•	구현 코드 금지
	•	토큰 스펙/JWT claim 상세 설계 금지
	•	특정 인증 제품 종속 설명 최소화
	•	장기적으로 유지 가능한 API 보안 구조 관점 우선

⸻

7. 제외 범위

이번 작업에서는 아래는 하지 마라.
	•	실제 로그인 플로우 설계
	•	OAuth/OIDC/SAML 상세 프로토콜 설계
	•	JWT 클레임 스키마 확정
	•	세션 저장소 설계
	•	RBAC/ACL DB 스키마 설계
	•	실제 middleware / FastAPI dependency 코드 작성
	•	세부 policy language 설계
	•	permission matrix 상세 구현
	•	webhook signature algorithm 상세 선택
	•	암호키 회전 운영 절차 상세 설계

즉, 이번 작업은 API 계층에 인증/인가 구조를 반영하는 원칙 설계에 집중하라.

⸻

8. 작성 스타일 가이드

문서는 아래 스타일을 따르라.
	•	인증과 인가를 명확히 분리해서 설명
	•	기술 선택보다 설계 원칙과 책임 경계를 우선
	•	사람/서비스/AI/외부 시스템 주체 차이를 구조적으로 설명
	•	“권장 / 허용 / 비권장 / 금지 / 후속 Task에서 구체화”를 분명히 구분
	•	보안 문맥 설계를 추상적으로만 쓰지 말고 실제 후속 구현이 가능한 수준으로 정리
	•	표나 매트릭스를 적절히 활용 가능

⸻

9. 검토 체크리스트

작업 완료 전에 아래를 스스로 점검하라.
	•	인증과 인가의 역할 차이가 명확한가?
	•	다양한 actor 유형이 빠지지 않았는가?
	•	request security context 개념이 충분히 정의되었는가?
	•	tenant/organization scope 처리 방향이 정리되었는가?
	•	API Layer와 Authorization Layer 책임이 분리되었는가?
	•	ACL 기반 리소스 권한과 endpoint guard 관계가 설명되었는가?
	•	async job / webhook / internal service / AI agent 문맥이 포함되었는가?
	•	감사 추적과 연결되는가?
	•	후속 Task가 이 문서를 기준선으로 삼을 수 있는가?

⸻

10. Claude Code에 대한 최종 지시

위 요구사항을 반영하여 Task3_authn_authz_api_design.md 문서를 작성하라.

문서는 문서 플랫폼 API의 인증/인가 구조를 단순 로그인 설명이 아니라,
API 보안 문맥, 권한 판단 경계, 조직 스코프, 서비스 신원, 감사 추적 연계까지 포함하는 상위 설계 기준 문서로 정리해야 한다.

문서 작성 후에는 아래 형식의 자체 점검 요약도 함께 정리하라.

자체 점검 요약
	•	구분한 actor 유형 목록
	•	보안 상위 원칙 요약
	•	request security context 핵심 필드 요약
	•	API Layer / Authorization Layer 책임 분리 요약
	•	async/webhook/service/AI 관련 보안 관점 요약
	•	후속 Task 연결 포인트
	•	의도적으로 미결정으로 남긴 항목