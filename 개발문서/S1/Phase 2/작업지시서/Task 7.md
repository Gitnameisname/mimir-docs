Task 2-7 작업 지시서

주제

정책 적용 우선순위와 예외 규칙 정리

작업 목적

Task 2-1부터 2-6까지를 통해 권한 주체, 보호 대상 리소스, ACL 기반 권한 모델, API enforcement, 감사 로그, 행위 추적 모델을 정리했다.
이번 단계에서는 그 위에, 실제 시스템에서 권한/정책/상태/예외가 충돌할 때 어떤 순서로 판단할지 기준선을 세운다.

즉, 이번 작업의 목적은 단순히 “권한이 있다/없다”를 넘어서, 아래와 같은 상황에서 무엇이 우선하는지를 설계 문서로 확정하는 것이다.
	•	조직 정책과 개별 ACL이 충돌하는 경우
	•	역할 권한과 리소스 상태 제약이 충돌하는 경우
	•	시스템 관리자 권한과 조직 경계가 만나는 경우
	•	soft delete / archived / locked / draft 상태에서 허용 범위가 바뀌는 경우
	•	향후 approval workflow나 고위험 작업 제한 정책과 연결될 경우

이 결과는 이후 구현 단계에서 권한 평가 순서, 예외 규칙 처리, 정책 엔진 확장 포인트의 기준이 된다.

⸻

이번 작업의 범위

이번 작업에서는 아래를 다룬다.
	1.	정책 평가 우선순위 정의
	•	인증/주체 유효성
	•	시스템 수준 제한
	•	조직 경계
	•	역할 기반 권한
	•	ACL 예외
	•	리소스 상태 기반 제한
	•	고위험 작업/관리자 전용 작업 제한
	2.	예외 규칙 체계 정리
	•	soft delete
	•	archived
	•	locked
	•	draft
	•	pending approval
	•	external share / public access
	•	read-only 상태
	3.	정책 충돌 해소 원칙 정의
	•	allow vs deny
	•	role allow vs state restriction
	•	admin privilege vs tenant boundary
	•	inherited permission vs direct exception
	•	organization policy vs per-resource ACL
	4.	상태 기반 권한 변화 규칙 정리
	•	리소스 lifecycle state에 따라 허용되는 행위가 달라지는 기준 정리
	5.	향후 정책 엔진 연결 포인트 정리
	•	approval workflow
	•	고위험 액션 승인 요구
	•	외부 공유 제한
	•	시간/도메인/도구 기반 제한 확장 가능성

⸻

이번 작업에서 하지 말 것

아래는 이번 단계에서 다루지 않는다.
	•	실제 policy engine 구현
	•	approval workflow 상세 설계
	•	ABAC 전체 설계
	•	risk scoring 모델 구현
	•	상태 머신 코드 작성
	•	UI 정책 설정 화면 설계
	•	운영자 콘솔 기능 구현

즉, 이번 단계는 정책 평가 순서와 예외 처리 원칙 설계 단계다.

⸻

설계 전제

아래 전제를 반영하여 설계할 것.
	1.	플랫폼은 범용 문서 플랫폼이며, 조직 단위 경계와 리소스 단위 예외를 모두 가진다.
	2.	기본 권한 모델은 RBAC + resource-scoped ACL 구조다.
	3.	권한이 있다고 해도, 리소스 상태나 시스템 정책 때문에 실제 실행이 제한될 수 있다.
	4.	시스템 관리자 권한이 존재하더라도, 모든 테넌트/조직 데이터에 무제한 접근하도록 설계하는 것은 위험하다.
	5.	향후 approval workflow, external share, service account, AI tool, integration action까지 정책 확장이 가능해야 한다.
	6.	감사 로그와 행위 추적을 통해 사후 설명 가능한 정책 체계를 선호한다.
	7.	MVP에서는 과도한 정책 복잡도를 피하되, 나중에 깨지지 않는 우선순위 구조를 선택해야 한다.

⸻

Claude Code가 수행해야 할 작업

아래 순서대로 정리할 것.

1. 정책 적용 계층 식별

최소한 아래 정책 계층을 후보로 검토할 것.
	•	Authentication validity
	•	Principal status validity
예: user disabled, membership suspended
	•	Global platform policy
	•	Organization boundary policy
	•	Role-based permission
	•	Resource-specific ACL
	•	Resource state restriction
	•	Sensitive operation guard
	•	Compliance / audit-required action rule
	•	External access/share rule

각 계층에 대해 아래를 정리할 것.
	•	왜 필요한가
	•	어떤 종류의 결정을 담당하는가
	•	다른 계층보다 먼저/나중에 평가되어야 하는가
	•	MVP 필수인지, 후속 확장인지

⸻

2. 권장 정책 평가 순서 제안

아래 질문에 답하면서, 최종 권장 평가 순서를 제시할 것.
	•	인증되지 않은 요청은 가장 먼저 배제해야 하는가
	•	비활성 사용자/중지된 멤버십은 role/ACL 평가 이전에 차단해야 하는가
	•	조직 경계 위반은 ACL 예외보다 우선 차단되어야 하는가
	•	role/ACL allow가 있어도 resource state restriction이 더 우선해야 하는가
	•	고위험 액션은 기본 권한이 있어도 추가 가드가 필요한가

최종적으로 아래 형태로 정리할 것.

예시:
	1.	Authentication validity check
	2.	Principal active status check
	3.	Tenant / organization boundary check
	4.	Baseline role permission check
	5.	Resource-specific ACL exception check
	6.	Resource state restriction check
	7.	Sensitive operation guard check
	8.	Final decision + audit metadata

단, 예시는 그대로 따르지 말고 검토 후 권장안을 제시할 것.

⸻

3. 정책 충돌 시 우선 원칙 정의

아래 충돌 케이스를 하나씩 검토할 것.
	•	Role은 허용하지만 resource state가 locked인 경우
	•	Organization policy는 금지하지만 resource ACL은 허용하는 경우
	•	상위 scope inherited allow가 있지만 하위 resource가 별도 제한 상태인 경우
	•	Platform admin이지만 tenant boundary를 넘는 접근인 경우
	•	Document read는 허용되지만 export는 금지된 경우
	•	Edit 권한은 있지만 pending approval 상태라 수정이 제한되는 경우

각 케이스에 대해 아래를 정리할 것.
	•	어느 정책이 우선하는가
	•	이유는 무엇인가
	•	감사 로그에는 어떤 판단 근거를 남겨야 하는가

그리고 최종적으로 충돌 해소 기본 원칙을 5~10개 정도의 명확한 규칙으로 요약할 것.

⸻

4. 상태 기반 예외 규칙 정리

최소한 아래 리소스 상태를 검토할 것.
	•	draft
	•	active / normal
	•	locked
	•	archived
	•	soft_deleted
	•	pending_approval
	•	published
	•	read_only
	•	external_shared

각 상태에 대해 아래를 정리할 것.
	•	어떤 행위가 기본적으로 허용되는가
	•	어떤 행위가 제한되는가
	•	관리자 예외가 허용되는가
	•	ACL 예외로 우회 가능한가
	•	audit 필요성이 높은가

중요:
상태는 단순 표시가 아니라, 권한 평가에 직접 영향을 주는 정책 조건으로 볼 것.

⸻

5. soft delete / archive / lock 예외 상세 원칙

이 부분은 별도로 깊게 다룰 것.

soft delete
	•	조회 가능 여부
	•	복원 가능 권한
	•	일반 사용자 숨김 여부
	•	관리자/감사자 노출 여부

archived
	•	수정 차단 여부
	•	조회 허용 여부
	•	export/download 허용 여부
	•	재활성화 권한

locked
	•	누가 잠글 수 있는가
	•	잠금 상태에서 누가 수정 가능한가
	•	lock override가 가능한가
	•	emergency override를 허용할 것인가

각 상태별로 MVP 권장안과 후속 확장 가능성을 분리해서 정리할 것.

⸻

6. 관리자 권한의 경계 정의

아래 역할을 중심으로 검토할 것.
	•	PlatformOwner / PlatformAdmin
	•	SecurityAuditor
	•	OrganizationOwner / OrganizationAdmin
	•	WorkspaceManager
	•	일반 Editor / Viewer

그리고 아래 질문에 답할 것.
	•	플랫폼 관리자는 모든 조직 데이터를 무조건 읽을 수 있어야 하는가
	•	보안 감사자는 콘텐츠 원문까지 봐야 하는가, 메타데이터만 보면 되는가
	•	조직 관리자는 soft-deleted 문서나 감사 로그까지 볼 수 있는가
	•	운영 지원 역할은 실제 문서 본문 접근 없이 문제를 재현할 수 있어야 하는가
	•	break-glass 접근이 필요한가, 있다면 어떻게 제한해야 하는가

최종적으로 아래를 정리할 것.
	•	관리자 권한의 기본 원칙
	•	콘텐츠 접근과 운영 권한의 분리 원칙
	•	break-glass 도입 여부와 조건
	•	감사 강도가 높아야 하는 관리자 행위 목록

⸻

7. ACL 예외 허용 범위 제한안

ACL이 너무 강하면 정책 체계가 무너질 수 있으므로 아래를 검토할 것.
	•	ACL로 조직 경계를 넘는 권한을 줄 수 있는가
	•	ACL로 locked/archived 상태를 우회할 수 있는가
	•	ACL로 admin-only action을 일반 사용자에게 줄 수 있는가
	•	ACL로 export/share만 별도 허용하는 것이 가능한가
	•	ACL 예외는 어떤 리소스/행위에만 허용하는 것이 적절한가

그리고 아래를 정리할 것.
	•	ACL로 허용 가능한 예외
	•	ACL로 허용하면 안 되는 예외
	•	정책적으로 더 상위에 두어야 하는 제한
	•	MVP ACL 예외 허용 범위

⸻

8. 고위험 액션 정책 초안

최소한 아래 행위를 고위험 액션 후보로 검토할 것.
	•	권한/ACL 변경
	•	대량 export
	•	external share 생성
	•	integration credential 생성/회수
	•	organization owner 변경
	•	soft-deleted 문서 영구 삭제
	•	locked resource override
	•	break-glass access
	•	감사 로그 열람/내보내기

각 액션에 대해 아래를 정리할 것.
	•	왜 고위험인가
	•	기본 role/ACL 외에 추가 가드가 필요한가
	•	추가 가드 예시
예: 재인증, 2단계 승인, 사유 입력, 강화 audit
	•	MVP에서 필수 반영할지 여부

⸻

9. 향후 정책 엔진 연결 포인트 정리

이번 단계는 구현이 아니므로, 아래 확장 포인트만 구조적으로 정리할 것.
	•	approval workflow와의 결합
	•	time-based restriction
예: 특정 시간대 외부 공유 금지
	•	environment/source-based restriction
예: service account는 특정 API만 허용
	•	sensitivity-based restriction
예: 민감 문서는 export 금지
	•	rate / volume-based restriction
예: 짧은 시간 내 과다 다운로드 제한
	•	emotional / behavioral risk gate와 같은 미래 확장 가능성은 별도 주석으로만 언급

중요:
여기서는 구현하지 말고, 현재 정책 우선순위 구조에 어디에 훅을 걸 수 있는지만 정리할 것.

⸻

10. 감사 로그 및 Activity Trace와의 연계 기준

Task 2-5, 2-6과 연결해서 아래를 정리할 것.
	•	정책 거부/예외 허용 시 audit에 반드시 남겨야 할 정보
	•	상태 기반 제한으로 막힌 경우 어떤 denial category를 쓸 것인가
	•	break-glass나 override는 activity trace에도 남겨야 하는가
	•	비정상 시도 반복은 activity trace와 결합해 어떻게 해석 가능한가

⸻

11. MVP 정책 우선순위/예외 규칙 제한안 제시

MVP에서 반드시 고정해야 할 정책 규칙과, 후속 단계로 넘길 것을 분리할 것.

예시 관점:

MVP 필수
	•	인증/비활성 주체 선차단
	•	조직 경계 우선 보호
	•	role 기반 기본 권한
	•	제한적 ACL 예외
	•	locked/archived/soft_deleted 상태 제한
	•	admin action 강화 audit
	•	고위험 액션 사유 기록

후속 확장
	•	deny 기반 세밀 정책
	•	time-based policy
	•	rate-based policy
	•	sensitivity label 기반 세밀 제어
	•	multi-stage approval gate
	•	break-glass 자동 만료 정책

중요:
초기 시스템은 예측 가능해야 하므로, 정책 계층을 너무 많이 늘리지 말 것.

⸻

12. 다음 단계 입력 자료 형태로 정리

Task 2-8 Phase 2 통합 설계서 정리로 넘기기 위해 아래를 정리할 것.
	•	정책 평가 순서 최종안
	•	상태 기반 제한 핵심 규칙
	•	관리자 권한 경계 규칙
	•	ACL 예외 허용 한계
	•	고위험 액션 가드 원칙
	•	감사/행위추적과 연결되는 정책 metadata

⸻

산출물 형식

이번 작업 결과물은 설계 문서 초안이어야 하며, 아래 구조를 따를 것.

Phase 2 - Task 2-7

1. 목표

2. 설계 전제

3. 정책 계층 후보 분석

4. 권장 정책 평가 순서

5. 정책 충돌 해소 원칙

6. 상태 기반 예외 규칙

7. soft delete / archived / locked 상세 원칙

8. 관리자 권한 경계 정의

9. ACL 예외 허용 범위 제한안

10. 고위험 액션 정책 초안

11. 향후 정책 엔진 연결 포인트

12. Audit / Activity 연계 기준

13. MVP 정책 제한안

14. 다음 Task로 넘길 결정사항

15. 오픈 이슈

⸻

의사결정 원칙

설계 중 아래 원칙을 반드시 지킬 것.
	1.	상위 보호 경계는 예외보다 우선
	•	조직 경계, 주체 유효성, 시스템 제한은 하위 ACL 예외보다 우선해야 함
	2.	상태는 권한 평가의 일부
	•	locked, archived, pending approval 같은 상태는 단순 표시값이 아니라 실행 제한 규칙으로 볼 것
	3.	관리자도 무제한이 아니다
	•	운영 권한과 콘텐츠 접근 권한을 분리하고, 필요 시 break-glass처럼 통제된 우회만 허용할 것
	4.	ACL은 제한적으로
	•	ACL이 모든 정책을 뒤엎지 못하게 할 것
	5.	고위험 액션은 추가 가드
	•	기본 권한 외에 사유, 강화 감사, 재확인 같은 추가 보호 장치를 둘 것
	6.	감사 가능성을 유지
	•	정책 거부/예외 허용/override는 나중에 설명 가능해야 함
	7.	MVP는 예측 가능해야 한다
	•	너무 복잡한 동적 정책보다 명시적이고 일관된 규칙을 우선할 것

⸻

기대 결과

이 작업이 끝나면 최소한 아래가 확정되어 있어야 한다.
	•	정책 평가 순서의 기준선
	•	충돌 시 어떤 정책이 우선하는지에 대한 원칙
	•	상태 기반 제한 규칙
	•	관리자 권한과 tenant boundary의 경계
	•	ACL 예외가 허용되는 범위와 금지 범위
	•	고위험 액션에 대한 추가 가드 방향
	•	Phase 2 통합 설계서에 넣을 정책 결정사항

⸻

금지 사항
	•	full policy engine 설계로 확장하지 말 것
	•	approval workflow 전체 구현을 이번 단계에 섞지 말 것
	•	상태 머신 상세 코드 작성으로 내려가지 말 것
	•	관리자에게 무조건 전체 우회 권한을 주는 식으로 단순화하지 말 것
	•	ACL 예외를 만능처럼 설계하지 말 것

⸻

작업 완료 판단 기준

아래 조건을 만족하면 완료로 본다.
	•	정책 평가 순서가 단계적으로 정리되어 있다
	•	충돌 해소 원칙이 명확한 규칙으로 제시되어 있다
	•	상태 기반 권한 변화 규칙이 설명되어 있다
	•	관리자 권한 경계와 break-glass 여부가 검토되어 있다
	•	ACL 예외 허용 범위가 제한적으로 정의되어 있다
	•	Task 2-8 통합 설계서에 넣을 결정사항이 정리되어 있다