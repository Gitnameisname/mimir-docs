Task 2-2 작업 지시서

주제

리소스 범위와 보호 대상 식별

작업 목적

Phase 2의 권한 체계 설계에서, 이번 단계는 무엇을 보호할 것인지를 명확히 정의하는 작업이다.
Task 2-1에서 권한의 주체(User, Organization, Membership, Role 등)를 정의했다면, 이제는 그 권한이 적용될 대상 리소스와 범위(scope) 를 구조적으로 식별해야 한다.

이 작업의 결과는 이후 Task 2-3의 ACL 설계, Task 2-4의 API 권한 enforcement 설계, Task 2-5의 감사 로그 설계의 직접적인 기반이 된다.

⸻

이번 작업의 범위

이번 작업에서는 아래를 다룬다.
	1.	보호 대상 리소스 식별
	•	플랫폼에서 권한 통제가 필요한 객체 목록 정리
	•	문서 본문뿐 아니라 관리성 객체까지 포함
	2.	리소스 범위(scope) 구조 설계
	•	전역 리소스
	•	조직 단위 리소스
	•	문서 단위 리소스
	•	하위 객체 단위 리소스
	3.	리소스 계층 관계 정의
	•	상위 리소스와 하위 리소스 관계
	•	권한 상속 가능성 검토
	•	리소스 별 독립 권한 필요 여부 판단
	4.	보호 수준 분류
	•	읽기만 보호하면 되는가
	•	생성/수정/삭제/관리/공유/승인 등 별도 통제가 필요한가
	•	민감 메타데이터와 일반 데이터 구분 필요 여부
	5.	ACL 설계의 입력 자료 준비
	•	어떤 principal이 어떤 resource에 permission을 가질 수 있는지, 이후 단계에서 구조화할 수 있도록 리소스 분류 기준을 만든다.

⸻

이번 작업에서 하지 말 것

아래는 이번 단계에서 다루지 않는다.
	•	ACL 엔트리 구조 상세 설계
	•	permission 명칭 세부 확정
	•	API endpoint별 권한 체크 구현
	•	DB 테이블 작성
	•	감사 로그 필드 상세 설계
	•	상태 전이 워크플로 설계
	•	UI 화면 설계

즉, 이번 단계는 리소스 식별과 범위 모델링 단계다.

⸻

설계 전제

아래 전제를 반영하여 설계할 것.
	1.	플랫폼은 범용 문서 플랫폼이며, 단순 문서 파일 저장소가 아니다.
	2.	Phase 1에서 정의한 범용 문서 모델(Document / Version / Node / DocumentType / metadata / state)을 전제로 한다.
	3.	문서 외에도 향후 아래와 같은 부가 객체가 자연스럽게 생길 수 있다.
	•	Attachment
	•	Comment
	•	Tag/Label
	•	Link/Reference
	•	Approval/Workflow
	•	Share / External access grant
	•	Knowledge extraction artifact
	4.	API-first 구조이므로, 내부 UI뿐 아니라 외부 API 클라이언트도 리소스 접근 주체가 될 수 있다.
	5.	사용자 UI와 관리자 UI가 분리될 수 있으므로, 운영성 리소스와 사용자 콘텐츠 리소스를 구분할 필요가 있다.
	6.	향후 ACL, 정책 엔진, 감사 로그, approval workflow가 붙더라도 깨지지 않는 리소스 계층 구조가 필요하다.

⸻

Claude Code가 수행해야 할 작업

아래 순서대로 정리할 것.

1. 보호 대상 리소스 후보 식별

최소한 아래 리소스들을 후보로 검토할 것.

콘텐츠 계열
	•	Document
	•	DocumentVersion
	•	DocumentNode
	•	Attachment
	•	EmbeddedAsset
	•	Comment
	•	Annotation
	•	Tag/Label
	•	Link/Reference

관리 계열
	•	Organization
	•	Workspace / Space / Folder 필요 여부
	•	DocumentType definition
	•	Template
	•	Metadata schema
	•	Permission policy object
	•	Share link / external grant
	•	Workflow / Approval object

운영 계열
	•	AuditLog
	•	ActivityTrace
	•	API Credential / Service Account
	•	Integration configuration
	•	Webhook / Connector config

각 리소스에 대해 아래를 작성할 것.
	•	왜 보호 대상인가
	•	누가 접근할 가능성이 있는가
	•	읽기/수정/관리 수준이 구분되어야 하는가
	•	MVP 필수 대상인가, 확장 대상인가

⸻

2. 리소스 범위(scope) 레벨 정의

최소한 아래 범위 레벨을 검토할 것.
	•	Platform scope
	•	Organization scope
	•	Workspace/Container scope (필요 시)
	•	Document scope
	•	Version scope
	•	Node scope
	•	Sub-resource scope (comment, attachment 등)

각 scope에 대해 다음을 정리할 것.
	•	어떤 리소스가 속하는가
	•	어떤 principal이 접근할 수 있는가
	•	상위 scope 권한이 하위 scope에 영향을 미칠 가능성이 있는가
	•	독립 ACL이 필요한 경우는 무엇인가

⸻

3. 리소스 계층 구조 초안 제시

텍스트 기반 계층 구조를 제시할 것.

예시 형태:
	•	Platform
	•	Organization
	•	Workspace/Folder
	•	Document
	•	Version
	•	Node
	•	Attachment
	•	Comment

단, 이것을 단순 나열로 끝내지 말고 아래를 함께 설명할 것.
	•	계층형으로 보는 것이 맞는 리소스
	•	참조형으로 보는 것이 맞는 리소스
	•	상속형 권한이 어울리는 리소스
	•	독립 통제가 필요한 리소스

예:
	•	DocumentVersion은 Document 하위지만, 별도 접근 통제가 필요할 수 있는가
	•	Node는 Document의 일부로만 볼지, 독립 수정 권한을 둘지
	•	Attachment는 상위 문서 권한을 따르는지, 예외 공유가 가능한지
	•	Comment는 문서 조회 권한과 별개로 작성/삭제 통제가 필요한지

⸻

4. 보호 수준 분류

리소스를 아래 관점으로 분류할 것.
	•	Read-sensitive
	•	Write-sensitive
	•	Admin-sensitive
	•	Security-sensitive
	•	Compliance-sensitive

예를 들어:
	•	Document 본문은 read/write-sensitive
	•	Permission policy object는 admin-sensitive
	•	AuditLog는 compliance-sensitive
	•	API credential은 security-sensitive

각 리소스별로 어떤 보호 수준을 가지는지 분류하고,
왜 그렇게 분류했는지 설명할 것.

⸻

5. 권한 상속과 독립 통제 원칙 정리

아래 질문에 답할 것.
	•	상위 리소스 권한이 하위 리소스에 자동 상속되어야 하는가
	•	모든 하위 리소스가 상속을 따르게 하면 어떤 문제가 생기는가
	•	예외 ACL이 자주 필요한 리소스는 무엇인가
	•	문서 버전은 문서와 동일 권한으로 볼지, 별도 보호가 필요한가
	•	댓글/첨부/주석은 상위 문서 권한으로 충분한가
	•	운영 리소스는 콘텐츠 리소스와 분리된 권한 체계가 필요한가

그리고 아래 형태로 원칙을 정리할 것.
	•	기본 상속 원칙
	•	예외 부여 가능 리소스
	•	독립 관리 리소스
	•	보안상 별도 분리 리소스

⸻

6. MVP 기준 보호 대상 우선순위 도출

MVP에서 반드시 보호 대상으로 봐야 할 것과, 후속 확장으로 넘길 것을 나눌 것.

예시 관점:

MVP 우선
	•	Organization
	•	Document
	•	DocumentVersion
	•	DocumentNode
	•	Comment
	•	Attachment
	•	Permission policy object
	•	AuditLog

후속 확장
	•	Share link
	•	Workflow
	•	External integration config
	•	Template
	•	Metadata schema

단, 이 분류는 검토 후 조정 가능하다.
중요한 것은 초기 시스템에서 반드시 권한 통제가 들어가야 하는 핵심 리소스를 분명히 하는 것이다.

⸻

7. 다음 단계 입력 자료 형태로 정리

Task 2-3 ACL 설계로 넘기기 위해, 리소스를 아래 기준으로 재정리할 것.
	•	principal이 직접 접근하는 리소스
	•	상위 리소스 권한을 상속받는 리소스
	•	예외 ACL 후보 리소스
	•	운영자 전용 리소스
	•	감사 대상 중요 리소스

즉, 단순 목록이 아니라 ACL 모델 설계 입력 자료가 되도록 재구성할 것.

⸻

산출물 형식

이번 작업 결과물은 설계 문서 초안이어야 하며, 아래 구조를 따를 것.

Phase 2 - Task 2-2

1. 목표

2. 설계 전제

3. 보호 대상 리소스 후보 분석

4. 리소스 범위(scope) 정의

5. 리소스 계층 구조 초안

6. 보호 수준 분류

7. 권한 상속 및 독립 통제 원칙

8. MVP 우선 보호 대상

9. 다음 Task로 넘길 결정사항

10. 오픈 이슈

⸻

의사결정 원칙

설계 중 아래 원칙을 반드시 지킬 것.
	1.	문서만 보호 대상으로 보지 말 것
	•	범용 문서 플랫폼은 부가 객체와 운영 객체까지 통제해야 한다
	2.	상속은 기본, 예외는 명시
	•	권한 체계는 단순해야 하므로 상속을 기본값으로 두되, 예외가 필요한 리소스를 분리할 것
	3.	콘텐츠 리소스와 운영 리소스 분리
	•	문서 콘텐츠 접근 권한과 플랫폼 운영 권한은 혼합하지 말 것
	4.	API-first 관점 유지
	•	UI에서만 보이는 객체가 아니라 API에서 접근 가능한 리소스 단위로 생각할 것
	5.	감사 친화성 확보
	•	중요한 리소스는 누가 접근/변경했는지 추적 가능한 단위로 식별되어야 한다

⸻

기대 결과

이 작업이 끝나면 최소한 아래가 확정되어 있어야 한다.
	•	권한 통제가 필요한 리소스 목록
	•	리소스의 scope 레벨 구조
	•	상위/하위 리소스 관계
	•	상속 권한이 적절한 리소스와 독립 통제가 필요한 리소스 구분
	•	MVP에서 우선 보호해야 할 핵심 리소스
	•	다음 ACL 설계에서 활용 가능한 리소스 분류 기준

⸻

금지 사항
	•	ACL rule을 미리 상세 정의하지 말 것
	•	permission action matrix를 이번 단계에서 완성하려 하지 말 것
	•	DB 테이블 설계로 내려가지 말 것
	•	UI 단위 메뉴 권한 중심으로 사고하지 말 것
	•	아직 확정되지 않은 workflow 전체를 과도하게 가정하지 말 것

⸻

작업 완료 판단 기준

아래 조건을 만족하면 완료로 본다.
	•	보호 대상 리소스가 콘텐츠/관리/운영 관점으로 정리되어 있다
	•	scope 레벨이 구조적으로 설명되어 있다
	•	계층 구조와 예외 통제 지점이 문서화되어 있다
	•	MVP 보호 우선순위가 제시되어 있다
	•	Task 2-3 ACL 설계에 넘길 수 있는 입력 기준이 정리되어 있다
