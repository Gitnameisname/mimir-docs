Task 2-8 작업 지시서

주제

Phase 2 통합 설계서 정리

작업 목적

Task 2-1부터 2-7까지를 통해 다음 내용을 각각 설계했다.
	•	권한 주체(User / Organization / Membership / Role 등)
	•	보호 대상 리소스와 scope 구조
	•	RBAC + ACL 기반 권한 모델
	•	API 레벨 authorization enforcement 구조
	•	Audit Log 체계
	•	Activity Trace 모델
	•	정책 적용 우선순위와 예외 규칙

이번 단계의 목적은 이 개별 설계 결과들을 하나의 일관된 문서로 통합하여,
이후 구현 단계에서 흔들리지 않는 Phase 2의 기준선 설계 문서를 만드는 것이다.

즉, 이번 작업은 새로운 세부 설계를 확장하는 것이 아니라,
이미 정의한 내용을 정리, 통합, 정합성 검토, 충돌 정리, 후속 과제 명시의 관점에서 재구성하는 단계다.

이 결과물은 이후 개발자가 구현에 들어가기 전에 반드시 참고해야 하는 보안/권한/감사/추적의 기준 문서가 된다.

⸻

이번 작업의 범위

이번 작업에서는 아래를 다룬다.
	1.	Task 2-1 ~ 2-7 결과 통합
	•	중복 제거
	•	용어 통일
	•	설계 충돌 정리
	•	기준 원칙 재정리
	2.	Phase 2 전체 구조 문서화
	•	주체(principal)
	•	보호 대상(resource)
	•	권한(permission)
	•	정책(policy)
	•	감사(audit)
	•	행위 추적(activity)
	•	enforcement 흐름
	3.	MVP 기준선 명확화
	•	지금 당장 구현해야 하는 것
	•	후속 확장으로 남길 것
	•	MVP 단순화 원칙
	4.	후속 Phase 연결 포인트 정리
	•	approval/workflow
	•	external share
	•	admin UI / user UI
	•	정책 엔진 확장
	•	sensitivity labeling
	•	integration / service account
	5.	구현 전 체크리스트 작성
	•	구현 착수 전에 반드시 확인해야 할 항목
	•	구현 중 흔들리기 쉬운 부분 명시

⸻

이번 작업에서 하지 말 것

아래는 이번 단계에서 다루지 않는다.
	•	실제 코드 작성
	•	DB 스키마 확정
	•	API 명세 전체 작성
	•	UI 화면 설계
	•	policy engine 구현
	•	audit/activity 수집 인프라 구현
	•	approval workflow 상세 설계

즉, 이번 단계는 통합 설계 문서 정리 단계다.

⸻

설계 전제

아래 전제를 반영하여 통합 문서를 정리할 것.
	1.	플랫폼은 범용 문서 플랫폼이다.
	2.	문서 모델은 Phase 1의 범용 문서 구조(Document / Version / Node / DocumentType / metadata / state)를 따른다.
	3.	권한 모델은 RBAC 중심 + 제한적 ACL 예외 구조를 채택한다.
	4.	조직 경계와 principal 유효성은 최상위 보호 경계로 취급한다.
	5.	authorization enforcement는 API-first 구조 하에서 공통 Authorization Service 중심으로 정리한다.
	6.	Audit Log와 Activity Trace는 목적이 다르므로 분리 설계한다.
	7.	정책 우선순위와 상태 기반 제한은 권한 평가의 핵심 일부로 포함한다.
	8.	MVP에서는 예측 가능성과 단순성을 우선하되, 후속 확장을 막지 않는 방향을 유지한다.

⸻

Claude Code가 수행해야 할 작업

아래 순서대로 정리할 것.

1. Phase 2 전체 목표 재정리

먼저 아래 질문에 답하면서 Phase 2의 목적을 짧고 명확하게 재정의할 것.
	•	왜 이 플랫폼에 권한/감사/행위추적 체계가 필요한가
	•	단순 RBAC만으로 충분하지 않은 이유는 무엇인가
	•	감사와 activity를 왜 동시에 설계했는가
	•	구현 전에 왜 통합 설계 문서가 필요한가

그리고 Phase 2의 핵심 목표 5~8개를 요약할 것.

⸻

2. 핵심 개념 사전(Glossary) 통합

Task 2-1 ~ 2-7에서 등장한 핵심 용어를 통일해서 정리할 것.

최소한 아래 용어를 포함할 것.
	•	Principal
	•	User
	•	Organization
	•	Membership
	•	Role
	•	Team / Group
	•	Service Account
	•	Resource
	•	Scope
	•	Permission
	•	ACL
	•	Policy
	•	Authorization
	•	Audit Log
	•	Activity Trace
	•	Trace ID
	•	Correlation ID
	•	Resource State
	•	Sensitive Action
	•	Break-glass

각 용어에 대해 아래를 정리할 것.
	•	이 문서에서의 정의
	•	다른 유사 용어와의 차이
	•	MVP 필수 개념인지, 확장 개념인지

중요:
통합 문서에서 같은 개념이 다른 표현으로 흔들리지 않도록 용어를 고정할 것.

⸻

3. 전체 아키텍처 관점 요약

Phase 2의 설계를 아래 큰 축으로 나눠 요약할 것.
	•	주체(principal) 계층
	•	보호 대상(resource) 계층
	•	권한 모델(RBAC + ACL)
	•	정책 계층(policy layers)
	•	enforcement 흐름
	•	감사 로그 계층
	•	activity trace 계층

각 축에 대해 아래를 정리할 것.
	•	핵심 역할
	•	다른 축과의 관계
	•	MVP 기준 구현 범위
	•	후속 확장 포인트

⸻

4. 권한 구조 통합 정리

아래 내용을 하나의 일관된 구조로 요약할 것.
	•	어떤 principal이 존재하는가
	•	어떤 resource가 보호 대상인가
	•	permission은 어떤 방식으로 부여되는가
	•	role과 ACL은 어떻게 결합되는가
	•	어떤 정책이 예외보다 우선하는가
	•	resource state는 권한 평가에 어떻게 반영되는가

이 섹션에서는 Task 2-1, 2-2, 2-3, 2-7의 핵심 결론을 통합해서 보여줄 것.

⸻

5. API Enforcement 구조 통합 정리

Task 2-4의 핵심을 반영하여 아래를 정리할 것.
	•	인증 이후 authorization 흐름
	•	router / service / authorization service / repository의 책임
	•	pre-check / post-load check 원칙
	•	list/search 처리 원칙
	•	오류 응답 원칙 (401 / 403 / 404)
	•	authorization decision metadata 개요

중요:
이 섹션은 나중에 백엔드 구현자가 바로 참고할 수 있도록 구조적이고 명확하게 정리할 것.

⸻

6. Audit Log / Activity Trace 분리 통합 정리

Task 2-5, 2-6의 핵심을 비교 방식으로 정리할 것.

최소한 아래 비교 축을 포함할 것.
	•	목적
	•	주요 사용자
	•	기록 대상
	•	저장 단위
	•	보존 성격
	•	민감도
	•	대표 이벤트
	•	correlation 방식
	•	어떤 경우 둘 다 남기는가

그리고 최종적으로 아래를 명시할 것.
	•	audit 전용 이벤트
	•	activity 전용 이벤트
	•	이중 기록 이벤트
	•	두 시스템 간 공유 필드

⸻

7. 정책 우선순위와 예외 규칙 통합 정리

Task 2-7의 핵심 결론을 문서화할 것.

최소한 아래를 포함할 것.
	•	권장 정책 평가 순서
	•	충돌 해소 기본 규칙
	•	상태 기반 제한 원칙
	•	관리자 권한 경계
	•	ACL 예외 허용 한계
	•	고위험 액션의 추가 가드 원칙

이 섹션은 구현자와 운영자가 모두 읽어도 흔들리지 않는 규칙 문장으로 정리할 것.

⸻

8. MVP 기준선 명시

이 부분은 매우 중요하다.
Phase 2 전체 설계 중에서 MVP에 반드시 포함해야 할 것과 후속 확장으로 남길 것을 명확히 나눌 것.

최소한 아래 관점으로 정리할 것.

MVP 필수
	•	Membership 중심 역할 구조
	•	Organization boundary enforcement
	•	RBAC 기반 기본 권한
	•	제한적 resource-scoped ACL
	•	공통 authorization service
	•	핵심 audit log
	•	핵심 activity trace
	•	locked / archived / soft_deleted 상태 제한
	•	고위험 액션 강화 audit

후속 확장
	•	deny 기반 세밀 정책
	•	Team/Group 고도화
	•	Service Account 정교화
	•	sensitivity label 기반 제어
	•	time/rate-based policy
	•	break-glass 자동 승인/만료
	•	fine-grained workflow/approval gate
	•	advanced UI telemetry

그리고 왜 그렇게 나누는지 설명할 것.

⸻

9. 구현 전 체크리스트 작성

실제 구현 전에 검토해야 할 체크리스트를 작성할 것.

최소한 아래를 포함할 것.
	•	용어 정의가 코드/문서/API에서 일치하는가
	•	principal 식별 방식이 확정되었는가
	•	resource scope 모델이 API path와 정합한가
	•	permission naming rule이 일관적인가
	•	authorization service 입력 모델이 정의되었는가
	•	audit/event taxonomy가 정리되었는가
	•	activity trace 수집 범위가 과도하지 않은가
	•	상태 기반 제한 규칙이 resource model과 연결 가능한가
	•	관리자 권한 범위가 과도하지 않은가
	•	ACL 예외 범위가 제한되어 있는가

가능하면 아래 두 단계로 나눌 것.
	•	구현 착수 전 체크리스트
	•	구현 리뷰 시 체크리스트

⸻

10. 설계 리스크와 흔들리기 쉬운 지점 정리

아래 유형의 리스크를 분석할 것.
	•	RBAC와 ACL 경계가 흐려지는 리스크
	•	감사 로그와 activity trace가 섞이는 리스크
	•	관리자 권한이 과도해지는 리스크
	•	상태 기반 정책이 나중에 누락되는 리스크
	•	list/search에서 정보 노출이 발생하는 리스크
	•	후속 workflow 정책이 기존 모델과 충돌하는 리스크

각 리스크에 대해 아래를 정리할 것.
	•	왜 생기는가
	•	어떤 문제가 발생하는가
	•	지금 설계에서 어떻게 예방하는가
	•	구현 시 어떤 점을 주의해야 하는가

⸻

11. 후속 Phase 연결 포인트 정리

최소한 아래와의 연결 포인트를 정리할 것.
	•	approval / workflow phase
	•	external share / public access phase
	•	admin UI phase
	•	user collaboration phase
	•	integration / service account phase
	•	policy engine / governance phase
	•	knowledge / AI tool integration phase

각 항목에 대해 아래를 정리할 것.
	•	이번 Phase 2의 어떤 결정이 기반이 되는가
	•	아직 열어 둔 확장 포인트는 무엇인가
	•	후속 단계에서 다시 결정해야 할 것은 무엇인가

⸻

12. 최종 통합 설계 요약 작성

문서 맨 마지막에, 아래 형식의 요약을 작성할 것.

Phase 2 핵심 결론 요약
	•	10개 내외

구현 시 절대 흔들리면 안 되는 원칙
	•	5~10개

다음 Phase에 넘기는 핵심 결정사항
	•	5~10개

이 요약은 실제 프로젝트 리더가 빠르게 훑어볼 수 있는 수준으로 명확하게 작성할 것.

⸻

산출물 형식

이번 작업 결과물은 Phase 2 통합 설계 문서 초안이어야 하며, 아래 구조를 따를 것.

Phase 2. 권한, 역할, 감사 추적 체계 설계

1. 문서 목적

2. Phase 2 목표 요약

3. 핵심 개념 사전

4. 전체 아키텍처 관점 요약

5. 권한 구조 통합 정리

6. API Authorization Enforcement 구조

7. Audit Log / Activity Trace 통합 정리

8. 정책 우선순위와 예외 규칙

9. MVP 기준선

10. 구현 전 체크리스트

11. 설계 리스크 및 주의점

12. 후속 Phase 연결 포인트

13. 최종 요약

⸻

의사결정 원칙

통합 문서 정리 시 아래 원칙을 반드시 지킬 것.
	1.	새로운 설계를 무리하게 추가하지 말 것
	•	기존 Task 결과를 정리하고 일관성 있게 통합하는 데 집중할 것
	2.	중복보다 기준선
	•	설명이 길어지더라도, 구현자가 흔들리지 않게 기준을 명확히 적을 것
	3.	MVP와 확장을 분리할 것
	•	지금 당장 구현해야 하는 것과 나중에 확장할 것을 혼합하지 말 것
	4.	정합성 우선
	•	principal, resource, permission, policy, audit, activity 개념이 서로 충돌하지 않도록 정리할 것
	5.	구현 친화성 유지
	•	이 문서는 이론 문서가 아니라 구현 직전 기준 문서다
	6.	감사와 추적의 차이를 끝까지 유지
	•	두 체계를 다시 하나로 합치지 말 것

⸻

기대 결과

이 작업이 끝나면 최소한 아래가 확정되어 있어야 한다.
	•	Phase 2 전체 설계가 하나의 문서로 통합되어 있다
	•	용어와 구조가 일관되게 정리되어 있다
	•	MVP 기준선이 명확하다
	•	구현 전 체크리스트가 있다
	•	후속 Phase로 넘길 결정사항이 정리되어 있다
	•	구현자가 이 문서를 보고 Phase 2 구현에 착수할 수 있다

⸻

금지 사항
	•	새로운 세부 기능 설계를 끝없이 추가하지 말 것
	•	구현 코드/DDL/API 세부명세로 내려가지 말 것
	•	아직 하지 않은 후속 Phase 내용을 이번 문서에 섞어 완료된 것처럼 쓰지 말 것
	•	audit/activity/policy 개념을 다시 혼합하지 말 것
	•	관리자 권한을 모호하게 표현하지 말 것

⸻

작업 완료 판단 기준

아래 조건을 만족하면 완료로 본다.
	•	Task 2-1 ~ 2-7 결과가 일관된 문서로 통합되어 있다
	•	용어 사전과 핵심 원칙이 정리되어 있다
	•	MVP 기준선과 후속 확장 구분이 명확하다
	•	구현 체크리스트와 리스크가 포함되어 있다
	•	다음 Phase와 연결할 수 있는 결정사항이 정리되어 있다