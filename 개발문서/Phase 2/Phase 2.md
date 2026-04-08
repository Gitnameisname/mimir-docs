Phase 2 하위 Task 구성안

Task 2-1. 권한/역할/조직 도메인 모델 정의

목표:
	•	사용자, 조직, 팀, 역할의 기본 관계를 정의
	•	플랫폼에서 어떤 단위로 권한을 부여할지 확정
	•	전역 역할과 문서/공간 단위 역할을 분리할지 결정

주요 산출물:
	•	User / Organization / Membership / Role 도메인 초안
	•	Role 계층 구조 및 책임 범위 정의
	•	권한 부여 단위 정의서

⸻

Task 2-2. 리소스 범위와 보호 대상 식별

목표:
	•	어떤 객체에 권한이 걸리는지 식별
	•	Document, Version, Node, Attachment, Comment, API endpoint 등 보호 대상을 정리
	•	리소스 계층과 상속 여부를 정의

주요 산출물:
	•	보호 대상 리소스 목록
	•	Resource Scope 계층 정의
	•	권한 상속/전파 규칙 초안

⸻

Task 2-3. ACL 기반 권한 모델 설계

목표:
	•	ACL 구조를 문서 플랫폼에 맞게 설계
	•	principal, resource, permission, effect(allow/deny), scope를 체계화
	•	기본 정책과 예외 정책을 함께 정의

주요 산출물:
	•	ACL 엔트리 구조 정의
	•	Permission taxonomy
	•	Allow/Deny 평가 우선순위 규칙

⸻

Task 2-4. API 레벨 Authorization Enforcement 설계

목표:
	•	API 요청이 들어왔을 때 어떤 단계에서 권한을 검사할지 정리
	•	인증 이후 권한 체크, 리소스 조회, 마스킹/거절 흐름 설계
	•	백엔드 전역 enforcement 패턴 정의

주요 산출물:
	•	API authorization flow
	•	endpoint/service/repository 계층별 책임 분리안
	•	권한 검사 실패 응답 규칙

⸻

Task 2-5. 감사 로그(Audit Log) 체계 설계

목표:
	•	누가 언제 무엇을 했는지 추적 가능한 구조 설계
	•	읽기/수정/삭제/권한 변경/배포/승인 등 주요 이벤트 분류
	•	보안 및 컴플라이언스 대응 가능한 감사 로그 필드 정의

주요 산출물:
	•	Audit Event taxonomy
	•	AuditLog 스키마 초안
	•	보존 정책 및 무결성 고려사항

⸻

Task 2-6. 행위 추적(Activity Trace) 모델 설계

목표:
	•	감사 로그와 별도로 사용자 활동 흐름을 분석할 수 있는 행위 추적 모델 설계
	•	문서 조회, 탐색, 검색, 편집 흐름을 연결 가능하게 구조화
	•	운영 분석과 보안 분석의 경계를 정리

주요 산출물:
	•	Activity/Event 모델 정의
	•	감사 로그와 activity 로그의 분리 원칙
	•	trace/correlation id 전략

⸻

Task 2-7. 정책 적용 우선순위와 예외 규칙 정리

목표:
	•	조직 정책, 역할 정책, ACL 예외, 시스템 관리자 권한의 충돌 규칙 정리
	•	soft delete, archived, locked, draft 상태에서의 권한 변화 정의
	•	향후 approval workflow와 연결 가능한 정책 기반 확보

주요 산출물:
	•	정책 평가 순서 문서
	•	예외 케이스 목록
	•	상태 기반 권한 변화 규칙

⸻

Task 2-8. Phase 2 통합 설계서 정리

목표:
	•	앞선 Task 결과를 하나의 일관된 설계 문서로 통합
	•	이후 구현 단계에서 바로 사용할 수 있도록 기준선 문서화
	•	Phase 1 문서 도메인과 연결점 명확화

주요 산출물:
	•	Phase 2 통합 설계서
	•	오픈 이슈 / 후속 Phase 연계 포인트
	•	구현 전 체크리스트
