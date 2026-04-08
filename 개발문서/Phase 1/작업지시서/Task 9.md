Claude Code 작업 지시서

Phase 1 - Task 1-9. 관계 / 무결성 / 제약 조건 정의

1. 작업명

범용 문서 플랫폼의 관계 / 무결성 / 제약 조건 정의

2. 작업 목적

Phase 1의 앞선 작업에서 정리한 요구사항, 핵심 엔티티 책임, Document / Version / Node / DocumentType / metadata / 상태 모델 초안을 바탕으로,
범용 문서 플랫폼의 핵심 도메인 모델이 실제 구현 단계에서도 흔들리지 않도록 엔티티 간 관계, 무결성 규칙, 핵심 제약 조건을 정리한다.

이 작업의 목적은 단순히 “관계도”를 그리는 것이 아니라,
이후 DB 설계, API 설계, 버전 관리, 편집 로직, 상태 전이 로직에서 반드시 지켜져야 할
도메인 수준의 불변조건(invariant) 을 명확히 하는 것이다.

즉, 이번 작업은 다음을 확정하는 단계다.
	•	Document / Version / Node / DocumentType이 어떤 관계를 가지는가
	•	어떤 값이 반드시 존재해야 하는가
	•	어떤 참조는 항상 정합해야 하는가
	•	어떤 변경은 허용되고 어떤 변경은 금지되는가
	•	상태, 버전, 트리 구조, metadata가 어떤 제약을 따라야 하는가

⸻

3. 작업 배경

앞선 Task들에서 핵심 엔티티와 구조는 정의했지만,
도메인 모델은 엔티티만 있다고 성립하지 않는다.
실제 시스템에서는 다음과 같은 문제가 쉽게 발생한다.

예:
	•	Document.current_version_id가 해당 Document의 Version이 아님
	•	Node.parent_id가 다른 Version의 Node를 가리킴
	•	한 문서에 current_version이 둘 이상처럼 해석되는 상태
	•	published인데 current_version이 없음
	•	archived인데 여전히 수정 가능한 흐름
	•	metadata 구조가 타입 규칙과 충돌
	•	Version의 Node 트리가 루트 없이 끊겨 있음
	•	Node order가 중복되거나 불안정함

이런 문제를 막기 위해 이번 단계에서는
도메인 모델이 반드시 만족해야 하는 관계 및 제약 규칙을 문서화해야 한다.

⸻

4. 작업 범위

포함 범위
	•	엔티티 간 관계 정의
	•	핵심 참조 정합성 규칙 정의
	•	버전/상태/트리 구조 관련 무결성 규칙 정의
	•	metadata 적용 관련 최소 제약 정의
	•	DocumentType 연계 제약 정의
	•	수정 가능성과 상태의 관계 정리
	•	최소 불변조건 목록 정리
	•	후속 Phase 입력값 정리

제외 범위
	•	실제 DB foreign key 설계
	•	unique index 설계 상세
	•	SQL 제약 조건 구현
	•	ORM validator 코드 작성
	•	API validation 코드 작성
	•	권한 정책 상세 설계

즉, 이번 작업은 도메인 규칙 명세서 작성이다.

⸻

5. Claude Code가 해야 할 일

아래 순서대로 수행하라.

Step 1. 관계 정의 대상 정리

먼저 아래 핵심 개체들을 기준으로 관계 정의 대상을 정리하라.
	•	Document
	•	Version
	•	Node
	•	DocumentType
	•	metadata
	•	상태 모델

그리고 아래 보조 개념도 필요 시 검토한다.
	•	current_version
	•	latest_version
	•	root node
	•	type-specific metadata
	•	state transition
	•	deletion / archive policy

이 단계에서는 어떤 관계가 핵심이고,
어떤 관계가 보조적인지 먼저 구조화하라.

⸻

Step 2. 핵심 엔티티 간 관계 정의

아래 관계를 중심으로 정리하라.

2-1. Document ↔ Version
검토 항목:
	•	Document 1 : N Version 관계의 의미
	•	Version은 반드시 하나의 Document에 속해야 하는가
	•	Document는 0개 이상의 Version을 가질 수 있는가
	•	current_version_id는 optional인가 required인가
	•	latest_version_id를 둘 경우 어떤 의미를 가지는가

2-2. Version ↔ Node
검토 항목:
	•	Version 1 : N Node 관계의 의미
	•	Node는 반드시 하나의 Version에 속해야 하는가
	•	Version은 1개 이상의 Node를 가져야 하는가
	•	root node 존재가 필수인가

2-3. Document ↔ DocumentType
검토 항목:
	•	Document는 정확히 하나의 DocumentType을 가져야 하는가
	•	type 없는 문서를 허용할 것인가
	•	generic_doc fallback type이 필요한가

2-4. metadata ↔ Document / Version / Node
검토 항목:
	•	metadata는 각 수준에 부속되는가
	•	메타데이터의 귀속 레벨이 섞이지 않도록 어떤 기준이 필요한가

2-5. 상태 ↔ Document / Version
검토 항목:
	•	상태는 Document 중심으로 관리하는가
	•	Version에는 보조 marker만 허용하는가

⸻

Step 3. Document 관련 무결성 규칙 정의

이 단계에서는 Document가 반드시 만족해야 할 규칙을 정리하라.

최소 아래를 포함하라.
	•	Document는 고유 식별자를 가져야 한다.
	•	Document는 하나의 DocumentType을 가져야 한다.
	•	Document.current_version_id가 존재한다면, 반드시 해당 Document의 Version을 가리켜야 한다.
	•	Document.status는 정의된 상태 집합 내 값이어야 한다.
	•	archived 상태 문서는 운영 정책상 비활성 문서로 취급되어야 한다.
	•	deleted는 상태가 아니라 별도 삭제 정책으로 분리할지 검토 결과를 반영하라.

또한 아래도 검토하라.
	•	title 필수 여부
	•	Document 생성 시 Version 없이 존재 가능한지
	•	generic_doc fallback 필요성

⸻

Step 4. Version 관련 무결성 규칙 정의

Version이 반드시 만족해야 할 규칙을 정리하라.

최소 아래를 포함하라.
	•	Version은 반드시 하나의 Document에 속해야 한다.
	•	Version은 동일 Document 내에서 식별 가능한 version_no를 가져야 한다.
	•	Version은 생성 시점의 스냅샷으로 취급되어야 한다.
	•	Version은 원칙적으로 불변이어야 한다.
	•	Version이 root node reference를 가진다면 해당 Version의 Node를 가리켜야 한다.
	•	base_version_id가 존재한다면 같은 Document 내 Version을 가리켜야 한다.

또한 아래도 검토하라.
	•	version_no 중복 허용 여부
	•	current_version / latest_version / published_version 관계
	•	orphan version 허용 여부

⸻

Step 5. Node 관련 무결성 규칙 정의

이 단계는 매우 중요하다. 반드시 별도 섹션으로 작성하라.

Node가 반드시 만족해야 할 규칙을 정리하라.

최소 아래를 포함하라.
	•	Node는 반드시 하나의 Version에 속해야 한다.
	•	Node.parent_id가 존재한다면 같은 Version의 Node를 가리켜야 한다.
	•	하나의 Node는 자기 자신을 parent로 가질 수 없다.
	•	Node 트리는 순환(cycle)을 가져서는 안 된다.
	•	root node는 parent_id가 없어야 한다.
	•	동일 부모 아래 sibling order는 일관되어야 한다.
	•	node_type은 정의된 허용 타입 집합에 속해야 한다.

또한 아래도 검토하라.
	•	Version당 root node는 정확히 1개여야 하는가
	•	content 없는 구조 노드를 허용할 것인가
	•	leaf node / container node 구분이 필요한가
	•	path/depth를 둘 경우 정합성 규칙 필요 여부

⸻

Step 6. 상태 관련 무결성 및 전이 제약 정리

상태 모델이 도메인 규칙과 어떻게 연결되는지 정리하라.

최소 아래를 포함하라.
	•	Document.status는 정의된 전이 규칙을 따라야 한다.
	•	published 상태 문서는 current_version이 존재해야 한다.
	•	archived 상태 문서는 새 편집 시작 시 복원 또는 새 작업 흐름을 거쳐야 하는지 검토하라.
	•	review 상태에서 허용되는 수정 정책은 후속 운영 정책 포인트로 남기되, 최소 제약은 기록하라.
	•	deleted는 상태 전이 대상인지 여부를 명확히 하라.

또한 아래도 검토하라.
	•	상태 전이는 자유 전이가 아니라 제한된 전이여야 하는가
	•	published → draft 복귀 허용 방식
	•	archived → published 재활성화 허용 여부

⸻

Step 7. DocumentType 관련 제약 정리

아래를 검토하라.
	•	DocumentType은 반드시 유효한 타입 키여야 한다.
	•	비활성 타입 사용 허용 여부
	•	특정 타입에 필요한 metadata 힌트가 있을 경우, 이를 강제 규칙으로 둘지 권장 규칙으로 둘지
	•	특정 타입에서 특정 Node type 사용을 제한할지 여부
	•	generic_doc를 fallback으로 둘 경우 무결성 관점에서 어떤 장점이 있는지

권장 방향:
	•	Phase 1에서는 강한 구조 강제보다 유효 타입 참조 + 확장 가능 규칙 연결 포인트 수준으로 유지

⸻

Step 8. metadata 관련 최소 제약 정리

metadata에 대해 아래 최소 제약을 검토하라.
	•	metadata는 귀속 수준(Document / Version / Node)에 맞는 의미를 가져야 한다.
	•	코어 필드와 중복되는 핵심 값은 metadata에 무분별하게 넣지 않도록 원칙을 정리하라.
	•	key naming / namespace 규칙이 있다면 이를 반영하라.
	•	공통 metadata와 유형별 metadata를 어떻게 구분할지 최소 기준을 정리하라.
	•	타입별 metadata template은 강제인지 권장인지 정리하라.

핵심은
metadata를 너무 자유롭게 풀었을 때 발생할 수 있는 무결성 혼란을 최소화하는 것이다.

⸻

Step 9. 최소 불변조건(invariants) 목록 작성

이 단계는 핵심 산출물이다. 반드시 별도 섹션으로 작성하라.

플랫폼 코어 모델이 항상 만족해야 하는
최소 불변조건 목록을 번호 매겨 정리하라.

예시 형식:
	1.	모든 Version은 정확히 하나의 Document에 속해야 한다.
	2.	모든 Node는 정확히 하나의 Version에 속해야 한다.
	3.	Node.parent_id는 같은 Version 내 Node만 가리킬 수 있다.
	4.	Document.current_version_id는 해당 Document의 Version만 가리킬 수 있다.
	5.	published 상태 Document는 current_version을 가져야 한다.

최소 12개 이상 도출하라.

⸻

Step 10. 제약 강도 분류

도출한 규칙들을 아래처럼 분류하라.
	•	Hard invariant
	•	반드시 항상 지켜져야 하는 규칙
	•	Soft rule / policy hint
	•	운영 정책이나 후속 확장에 따라 달라질 수 있는 규칙
	•	Future constraint
	•	지금은 강제하지 않지만 후속 Phase에서 제약 가능성이 큰 규칙

이 분류를 통해
초기 구현 시 무엇을 강하게 보장하고,
무엇을 유연하게 둘지 구분할 수 있게 하라.

⸻

Step 11. 최소 관계/제약 기준안 제시

위 검토를 바탕으로,
Phase 1 기준의 최소 관계/무결성/제약 기준안을 제안하라.

아래를 포함하라.
	•	핵심 관계 요약
	•	필수 정합성 규칙
	•	상태 관련 핵심 제약
	•	트리 구조 핵심 제약
	•	metadata 관련 최소 원칙
	•	후속 확장 포인트

구현 코드가 아니라 도메인 규칙 명세 형태로 작성하라.

⸻

Step 12. 설계 쟁점 및 판단 기록

아래 쟁점에 대해 판단을 남겨라.
	•	Document 생성 시 Version 없이 존재 가능하게 둘지
	•	current_version_id를 optional로 둘지
	•	Version당 root node를 정확히 1개로 강제할지
	•	generic_doc fallback을 강제할지
	•	type별 Node 제약을 초기부터 강하게 둘지
	•	metadata template 준수를 강제할지
	•	archived 문서 수정 경로를 어떻게 볼지
	•	deleted를 상태에서 제외할지

각 항목에 대해:
	•	쟁점
	•	선택한 방향
	•	이유
를 기록하라.

⸻

Step 13. 다음 Task 입력값 정리

문서 마지막에는 반드시 아래를 정리하라.
	•	Task 1-10(통합 모델 리뷰 및 기준안 확정)로 넘길 핵심 포인트
	•	저장 구조/DB 설계 시 바로 반영해야 할 규칙
	•	API 설계 시 유효성 검증 포인트
	•	에디터/문서 편집 플로우에서 강하게 의식해야 할 제약
	•	상태 전이/버전 생성 로직에서 꼭 지켜야 할 불변조건

⸻

6. 출력 형식 요구사항

반드시 Markdown(.md) 문서로 작성

권장 문서명:
Task9_relationships_integrity_constraints.md

권장 목차
	1.	작업 목적
	2.	관계 정의 대상
	3.	핵심 엔티티 간 관계
	4.	Document 관련 무결성 규칙
	5.	Version 관련 무결성 규칙
	6.	Node 관련 무결성 규칙
	7.	상태 관련 제약
	8.	DocumentType 관련 제약
	9.	metadata 관련 최소 제약
	10.	최소 불변조건 목록
	11.	제약 강도 분류
	12.	최소 관계/제약 기준안
	13.	설계 쟁점과 판단 결과
	14.	다음 Task 입력값

⸻

7. 작성 방식 요구사항

1) 표를 적극 활용

최소 아래 표를 포함하는 것이 좋다.
	•	엔티티 관계 요약표
	•	무결성 규칙 표
	•	제약 강도 분류표
	•	설계 쟁점 판단표

2) 구현 세부사항으로 내려가지 말 것

좋은 예:
	•	“Document.current_version_id는 반드시 동일 Document의 Version을 가리켜야 한다.”
	•	“Node.parent_id는 같은 Version 내 Node만 참조할 수 있다.”
	•	“published 상태 문서는 current_version을 가져야 한다.”

좋지 않은 예:
	•	SQL foreign key 문 작성
	•	ORM validator 코드 작성
	•	API validation 코드 작성

3) 도메인 불변조건 중심으로 작성할 것

이번 문서의 핵심은
“이 플랫폼 모델이 항상 어떤 조건을 만족해야 하는가”를 정리하는 것이다.

⸻

8. 산출물

필수 산출물
	1.	Task9_relationships_integrity_constraints.md

문서 내 반드시 포함되어야 할 내용
	•	핵심 엔티티 관계
	•	무결성 규칙
	•	최소 불변조건 목록
	•	제약 강도 분류
	•	최소 관계/제약 기준안
	•	후속 Task 입력값

⸻

9. 완료 기준

아래를 모두 만족하면 완료로 본다.
	•	Document / Version / Node / DocumentType 간 관계가 명확하다.
	•	핵심 정합성 규칙이 정리되어 있다.
	•	상태, 트리, metadata 관련 최소 제약이 있다.
	•	최소 불변조건 목록이 체계적으로 정리되어 있다.
	•	Hard invariant와 Soft rule이 구분되어 있다.
	•	Task 1-10, 저장 구조, API 설계로 자연스럽게 이어질 수 있다.

⸻

10. 작업 시 주의사항
	•	DB 구현 세부로 바로 내려가지 마라.
	•	너무 많은 운영 정책을 강한 제약으로 고정하지 마라.
	•	반대로 핵심 참조 정합성은 느슨하게 두지 마라.
	•	상태와 권한, 상태와 삭제 정책을 혼동하지 마라.
	•	Node 트리 무결성을 가볍게 보지 마라.
	•	metadata를 무제한 자유 영역으로 보지 마라.
	•	MVP 제약과 장기 확장을 구분하라.

⸻

11. Claude Code에게 전달할 한 줄 지시

“Mimir 프로젝트의 Phase 1 - Task 1-9로서, 범용 문서 플랫폼의 관계 / 무결성 / 제약 조건을 정리한 Markdown 문서를 작성하라. Document / Version / Node / DocumentType / metadata / 상태 모델 간 관계, 핵심 정합성 규칙, 최소 불변조건 목록, 제약 강도 분류, 최소 기준안, 후속 Task 입력값까지 포함하되 구현은 하지 마라.”