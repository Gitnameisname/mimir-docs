Claude Code 작업 지시서

Phase 1 - Task 1-4. Version 구조 설계

1. 작업명

범용 문서 플랫폼의 Version 구조 설계

2. 작업 목적

Phase 1의 앞선 작업에서 정리한 요구사항, 핵심 엔티티 책임, Document 구조 초안을 바탕으로,
범용 문서 플랫폼에서 특정 시점의 문서 상태를 표현하는 Version 엔티티의 구조를 상세 설계한다.

이 작업의 목적은 Document와 Version의 책임을 명확히 분리한 상태에서,
Version이 어떤 정보를 보관하고 어떤 규칙을 따라야 하는지를 정리하여
이후 Node 구조, 상태 모델, 저장 구조, 변경 이력 기능 설계의 기준을 만드는 것이다.

즉, 이번 작업은 “버전이 있다는 사실”을 넘어서,
Version이 무엇을 스냅샷으로 보존하고, 어떤 불변성과 참조 구조를 가져야 하는가를 확정하는 단계다.

⸻

3. 작업 배경

범용 문서 플랫폼에서 Document는 문서의 지속적인 정체성을 나타내고,
Version은 특정 시점의 문서 내용과 속성 상태를 나타낸다.

이 분리가 필요한 이유는 다음과 같다.
	•	문서는 시간이 지나도 같은 문서로 남아야 한다.
	•	문서 내용은 여러 번 수정될 수 있다.
	•	과거 상태를 추적, 비교, 복원할 수 있어야 한다.
	•	승인, 배포, 검토, 임시 작업 등은 버전 단위로 확장될 가능성이 높다.

따라서 Version은 단순 보조 테이블이 아니라,
문서 이력성과 재현 가능성을 보장하는 핵심 엔티티다.

이번 작업에서는 특히 아래를 명확히 해야 한다.
	•	Version이 표현하는 범위는 어디까지인가
	•	Version은 무엇을 스냅샷으로 가져야 하는가
	•	Version과 Node의 관계는 어떻게 되는가
	•	Version은 불변 엔티티로 볼 것인가
	•	Draft/Published 개념을 이번 단계에서 어디까지 반영할 것인가

⸻

4. 작업 범위

포함 범위
	•	Version의 역할 정의
	•	Version 필드 후보 도출
	•	필수/선택 필드 구분
	•	Version이 스냅샷으로 보관해야 하는 정보 범위 정의
	•	Document와의 관계 정리
	•	Node와의 관계 정리
	•	버전 불변성 원칙 정리
	•	current/latest/published/draft 관련 개념 검토
	•	후속 Task 입력값 정리

제외 범위
	•	Node 상세 구조 설계
	•	DB 스키마/ORM 구현
	•	diff 알고리즘 구현
	•	버전 비교 UI 설계
	•	승인 워크플로 상세 설계
	•	API 스펙 작성

즉, 이번 작업은 Version 개념 모델 명세서 작성이다.

⸻

5. Claude Code가 해야 할 일

아래 순서대로 수행하라.

Step 1. Version의 역할 정의

먼저 Version이 무엇을 대표하는지 명확히 써라.

반드시 아래 관점을 포함한다.
	•	Version은 특정 시점의 문서 상태를 나타낸다.
	•	Version은 문서의 본체가 아니라 Document에 종속된 시점 단위다.
	•	Version은 문서 내용의 재현 가능성을 보장해야 한다.
	•	Version은 Node 트리의 기준 단위가 된다.
	•	Version은 변경 이력 관리의 기본 단위다.

그리고 아래도 함께 정리한다.
	•	Version이 책임하는 것
	•	Version이 책임하지 않는 것
	•	Document가 책임해야 하는 것
	•	Node가 책임해야 하는 것

⸻

Step 2. Version 필드 후보 도출

Version이 가질 수 있는 필드 후보를 체계적으로 정리하라.

최소 아래 후보를 검토한다.
	•	id
	•	document_id
	•	version_no 또는 revision_no
	•	title_snapshot
	•	summary_snapshot
	•	metadata_snapshot
	•	root_node_id 또는 content_root_ref
	•	change_summary
	•	created_by
	•	created_at
	•	base_version_id
	•	status 또는 lifecycle marker
	•	published_at
	•	is_current
	•	is_published
	•	checksum / hash
	•	source_of_change

각 필드에 대해 다음을 검토한다.
	•	의미
	•	필수 여부
	•	MVP 포함 여부
	•	Document에 있어야 하는지 Version에 있어야 하는지
	•	후속 확장 필드인지

⸻

Step 3. Version 필드 분류 체계 정리

Version 필드를 아래 범주로 나누어 정리하라.

3-1. 식별 필드
예:
	•	id
	•	document_id
	•	version_no

3-2. 스냅샷 필드
예:
	•	title_snapshot
	•	summary_snapshot
	•	metadata_snapshot

3-3. 내용 참조 필드
예:
	•	root_node_id
	•	content_root_ref

3-4. 변경 이력 필드
예:
	•	change_summary
	•	base_version_id
	•	source_of_change

3-5. 추적 필드
예:
	•	created_by
	•	created_at

3-6. 배포/상태 관련 필드
예:
	•	published_at
	•	lifecycle marker

3-7. 무결성 보조 필드
예:
	•	checksum / hash

각 범주가 왜 필요한지 설명도 함께 적어라.

⸻

Step 4. Version이 보존해야 하는 스냅샷 범위 정의

이 단계는 핵심이다. 반드시 별도 섹션으로 작성하라.

아래 항목별로
“Version이 스냅샷으로 가져야 하는가 / 참조만 하면 되는가 / 후속으로 미뤄도 되는가”를 판단하라.

검토 대상:
	•	제목
	•	요약
	•	문서 유형
	•	metadata
	•	Node 트리 구조
	•	상태
	•	태그
	•	첨부 참조
	•	외부 링크 참조

핵심 질문:
	•	어떤 정보가 있어야 과거 버전을 정확히 재현할 수 있는가?
	•	어떤 정보는 Document의 현재 대표 정보만 있으면 충분한가?
	•	어떤 정보는 Version snapshot으로 남기는 것이 안전한가?

⸻

Step 5. Version과 Document의 관계 정리

아래 질문에 답할 수 있도록 정리하라.
	•	Document 1 : N Version 관계는 어떤 의미인가?
	•	Document는 current_version_id만 가지면 충분한가?
	•	latest_version과 published_version은 분리 가능한가?
	•	Version 쪽에 is_current 같은 중복 상태를 둘 필요가 있는가?
	•	문서 제목 변경 시 Document.title과 Version.title_snapshot은 어떻게 관계 맺는가?

이 단계에서는
Version이 Document의 하위 엔티티이되, 재현 가능성을 위해 충분히 독립적인 스냅샷을 가져야 한다는 점을 정리해야 한다.

⸻

Step 6. Version과 Node의 관계 정리

아래 관점을 포함해 정리하라.
	•	Node는 특정 Version에 종속되는가?
	•	하나의 Node가 여러 Version에서 재사용되는가, 아니면 버전별로 고정되는가?
	•	Version은 Node 트리 전체의 루트를 참조하는가?
	•	Node 수정은 새 Version 생성을 유도하는가?

권장 방향도 함께 제시하라.

일반적으로는 초기 설계에서:
	•	Version 1 : N Node
	•	Node는 특정 Version 소속
	•	버전 생성 시 Node 트리도 해당 Version 기준으로 고정
이라는 방향이 유력하다.

하지만 대안도 짧게 검토하라.

⸻

Step 7. 버전 불변성 원칙 정의

이 단계도 별도 섹션으로 작성하라.

반드시 아래 내용을 다룬다.
	•	생성된 Version은 수정 가능한가?
	•	오탈자 수정 같은 예외를 허용할 것인가?
	•	Node 변경은 기존 Version 수정이 아니라 새 Version 생성으로 보는가?
	•	metadata 변경도 새 Version을 만들어야 하는가?
	•	완전 불변 모델 vs 제한적 수정 모델 중 어떤 방향이 적절한가?

권장 방향:
	•	원칙적으로 Version은 불변
	•	수정은 새 Version 생성
	•	예외적 시스템 보정은 감사 로그 하에서만 허용 가능

이렇게 설계 판단을 문서화하라.

⸻

Step 8. 버전 관리 정책 초안 작성

아래 항목을 검토하여 초안을 제시하라.
	•	버전 번호 부여 방식
	•	정수 증가형
	•	major/minor 가능성
	•	초안(draft) 버전을 Version으로 볼지 여부
	•	published version과 working version의 분리 필요성
	•	복원 시 새 Version 생성 여부
	•	비교(diff)는 Phase 1 범위 밖이지만 고려 포인트만 기록

목적은 후속 Phase에서 버전 정책을 설계할 수 있도록 기본 판단을 남기는 것이다.

⸻

Step 9. 최소 권장 Version 구조안 제시

위 검토를 바탕으로,
Phase 1 기준의 최소 권장 Version 구조안을 제안하라.

형식 예시:
	•	필수 필드
	•	권장 필드
	•	선택 확장 필드
	•	후속 Phase로 미룰 필드

구현 코드가 아니라 개념 모델 명세 형태로 작성하라.

⸻

Step 10. 설계 쟁점 및 판단 기록

아래 쟁점에 대해 판단을 남겨라.
	•	version_no를 단순 정수로 시작할지
	•	title_snapshot을 둘지
	•	metadata_snapshot을 어디까지 저장할지
	•	is_current / is_published를 Version에 둘지
	•	base_version_id를 코어에 포함할지
	•	draft를 Version 엔티티로 바로 포함할지

각 항목에 대해:
	•	쟁점
	•	선택한 방향
	•	이유
를 기록하라.

⸻

Step 11. 다음 Task 입력값 정리

문서 마지막에는 반드시 아래를 정리한다.
	•	Task 1-5(Node 구조 설계)로 넘길 핵심 포인트
	•	Task 1-8(상태 모델 기초 설계)와 연결되는 포인트
	•	저장 구조 설계 시 확인해야 할 Version 관련 쟁점

⸻

6. 출력 형식 요구사항

반드시 Markdown(.md) 문서로 작성

권장 문서명:
Task4_version_structure_design.md

권장 목차
	1.	작업 목적
	2.	Version의 역할 정의
	3.	Version 필드 후보 목록
	4.	필드 분류 체계
	5.	스냅샷 범위 정의
	6.	Document와의 관계
	7.	Node와의 관계
	8.	버전 불변성 원칙
	9.	버전 관리 정책 초안
	10.	최소 권장 Version 구조안
	11.	설계 쟁점과 판단 결과
	12.	다음 Task 입력값

⸻

7. 작성 방식 요구사항

1) 표를 적극 활용

최소 아래 표를 포함하는 것이 좋다.
	•	필드 후보 분류표
	•	스냅샷 포함 여부 판단표
	•	설계 쟁점 판단표

2) 구현 세부사항으로 내려가지 말 것

좋은 예:
	•	“Version은 특정 시점의 문서 내용을 재현할 수 있어야 한다.”
	•	“Version은 Node 트리의 기준 단위가 되어야 한다.”
	•	“Version은 원칙적으로 불변이어야 한다.”

좋지 않은 예:
	•	ORM 클래스 코드
	•	SQL 테이블 설계
	•	diff 알고리즘 구현

3) 재현 가능성과 책임 분리를 우선할 것

필드 나열보다 중요한 것은
Version이 왜 필요한지, 무엇을 보존해야 하는지, Document와 어떻게 다른지를 분명히 하는 것이다.

⸻

8. 산출물

필수 산출물
	1.	Task4_version_structure_design.md

문서 내 반드시 포함되어야 할 내용
	•	Version 역할 정의
	•	Version 필드 후보 목록
	•	스냅샷 범위 판단
	•	Document/Node와의 관계
	•	버전 불변성 원칙
	•	최소 권장 Version 구조안
	•	후속 Task 입력값

⸻

9. 완료 기준

아래를 모두 만족하면 완료로 본다.
	•	Version의 역할이 Document와 구분되어 명확하다.
	•	Version 필드가 체계적으로 정리되어 있다.
	•	Version이 무엇을 스냅샷으로 가져야 하는지 정의되어 있다.
	•	Node와의 관계가 정리되어 있다.
	•	불변성 원칙이 명확하다.
	•	Task 1-5, 1-8, 저장 구조 설계로 자연스럽게 이어질 수 있다.

⸻

10. 작업 시 주의사항
	•	Version을 단순 “이력 번호” 정도로 축소하지 마라.
	•	Document와 역할이 섞이지 않게 하라.
	•	Node 구조 상세로 너무 깊게 들어가지 마라.
	•	현재 버전 참조와 버전 자체 상태를 혼동하지 마라.
	•	draft/published 개념을 너무 복잡하게 만들지 마라.
	•	MVP와 장기 확장을 구분하라.
	•	Version의 불변성 원칙을 흐리지 마라.

⸻

11. Claude Code에게 전달할 한 줄 지시

“Mimir 프로젝트의 Phase 1 - Task 1-4로서, 범용 문서 플랫폼의 Version 엔티티 구조를 설계하는 Markdown 문서를 작성하라. Version의 역할, 필드 후보, 스냅샷 범위, Document/Node와의 관계, 버전 불변성 원칙, 최소 권장 구조안, 후속 Task 입력값까지 포함하되 구현은 하지 마라.”