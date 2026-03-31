Claude Code 작업 지시서

Phase 1 - Task 1-5. Node 구조 설계

1. 작업명

범용 문서 플랫폼의 Node 구조 설계

2. 작업 목적

Phase 1의 앞선 작업에서 정리한 요구사항, 핵심 엔티티 책임, Document 구조, Version 구조를 바탕으로,
범용 문서 플랫폼에서 문서 내부 내용을 표현하는 최소 구조 단위인 Node의 구조를 상세 설계한다.

이 작업의 목적은 단순 텍스트 저장이 아니라,
다양한 문서 유형을 공통된 방식으로 표현할 수 있도록 트리 기반 문서 표현의 핵심 모델을 정립하는 것이다.

즉, 이번 작업은 다음을 확정하는 단계다.
	•	문서 본문을 어떤 구조 단위로 표현할 것인가
	•	Node는 어떤 속성을 가져야 하는가
	•	트리 구조를 어떻게 다룰 것인가
	•	어떤 Node type들이 필요한가
	•	구조 노드와 콘텐츠 노드를 어떻게 구분할 것인가
	•	Version과 Node의 경계를 어떻게 둘 것인가

⸻

3. 작업 배경

범용 문서 플랫폼은 단순한 “본문 문자열”만 저장해서는 확장성이 부족하다.
규정 문서, 기술 문서, 보고서, 회의록, 매뉴얼은 모두 구조를 가진다.

예:
	•	제목 / 절 / 항 / 문단
	•	리스트 / 리스트 아이템
	•	표 / 행 / 셀
	•	이미지 / 첨부 참조 / 인용 / 코드 블록
	•	템플릿 기반 블록
	•	추후 확장될 임베디드 콘텐츠

따라서 문서 본문은 트리 기반 Node 구조로 표현하는 것이 유력하다.
이때 Node는 단순 텍스트 조각이 아니라, 문서의 구조와 내용을 함께 표현하는 공통 단위여야 한다.

이번 작업에서는 특히 아래를 명확히 해야 한다.
	•	Node의 최소 책임은 무엇인가
	•	Node는 Version에 종속되는가
	•	Node는 어떤 트리 규칙을 가져야 하는가
	•	Node type 시스템은 얼마나 세분화할 것인가
	•	본문과 메타를 어디까지 Node에 둘 것인가

⸻

4. 작업 범위

포함 범위
	•	Node의 역할 정의
	•	Node 필드 후보 도출
	•	필수/선택 필드 구분
	•	트리 구조 표현 방식 검토
	•	Node type 체계 초안 설계
	•	구조 노드 / 콘텐츠 노드 구분 검토
	•	Node metadata 범위 검토
	•	Version과의 관계 정리
	•	최소 권장 Node 구조안 제시
	•	후속 Task 입력값 정리

제외 범위
	•	리치 텍스트 에디터 구현
	•	Markdown parser 설계
	•	UI 렌더러 설계
	•	표 편집기 설계
	•	DB 스키마/ORM 구현
	•	검색 인덱스 상세 설계

즉, 이번 작업은 문서 내부 구조 표현을 위한 Node 개념 모델 명세서 작성이다.

⸻

5. Claude Code가 해야 할 일

아래 순서대로 수행하라.

Step 1. Node의 역할 정의

먼저 Node가 무엇을 대표하는지 명확히 써라.

반드시 아래 관점을 포함한다.
	•	Node는 문서 내부 구조를 표현하는 기본 단위다.
	•	Node는 Version에 종속되는 문서 구성 요소다.
	•	Node는 구조와 내용을 함께 담을 수 있어야 한다.
	•	Node는 트리 기반 계층 구조를 형성할 수 있어야 한다.
	•	Node는 다양한 문서 유형을 공통 형식으로 수용해야 한다.

그리고 아래도 함께 정리한다.
	•	Node가 책임하는 것
	•	Node가 책임하지 않는 것
	•	Version이 책임해야 하는 것
	•	Document가 책임해야 하는 것

⸻

Step 2. Node 필드 후보 도출

Node가 가질 수 있는 필드 후보를 체계적으로 정리하라.

최소 아래 후보를 검토한다.
	•	id
	•	version_id
	•	parent_id
	•	order / position
	•	node_type
	•	title
	•	content
	•	content_format
	•	metadata
	•	depth
	•	path
	•	ref_key
	•	is_required
	•	is_hidden
	•	label
	•	attributes
	•	created_at
	•	updated_at

각 필드에 대해 다음을 검토한다.
	•	의미
	•	필수 여부
	•	MVP 포함 여부
	•	구조 표현에 필요한지
	•	후속 확장 필드인지

주의:
	•	모든 필드를 무조건 채택하지 말고,
	•	최소 코어 구조와 확장 후보를 구분하라.

⸻

Step 3. Node 필드 분류 체계 정리

Node 필드를 아래 범주로 나누어 정리하라.

3-1. 식별 및 소속 필드
예:
	•	id
	•	version_id

3-2. 트리 구조 필드
예:
	•	parent_id
	•	order / sibling_position
	•	depth
	•	path

3-3. 타입 및 역할 필드
예:
	•	node_type
	•	label

3-4. 콘텐츠 필드
예:
	•	title
	•	content
	•	content_format

3-5. 확장 속성 필드
예:
	•	metadata
	•	attributes
	•	ref_key
	•	is_required
	•	is_hidden

3-6. 추적 필드
예:
	•	created_at
	•	updated_at

각 범주가 왜 필요한지도 설명하라.

⸻

Step 4. 트리 구조 표현 방식 검토

이 단계는 반드시 별도 섹션으로 작성하라.

아래 방식들을 비교 검토하라.
	•	parent_id 기반 adjacency list
	•	path 기반 구조
	•	nested set / closure table 같은 고급 방식은 개념적으로만 언급

판단 포인트:
	•	이해 용이성
	•	범용성
	•	구현 복잡도
	•	버전 스냅샷 구조와의 궁합
	•	편집/이동/재정렬 가능성

권장 방향도 제시하라.

일반적으로 Phase 1에서는:
	•	기본은 parent_id + order
	•	필요 시 path 또는 depth는 보조 정보
로 정리하는 방향이 유력하다.

⸻

Step 5. Node type 체계 초안 설계

이 단계도 핵심이다. 반드시 별도 섹션으로 작성하라.

최소 아래 유형군을 검토하라.

구조 노드
	•	document_root
	•	section
	•	subsection
	•	container

텍스트/본문 노드
	•	paragraph
	•	heading
	•	quote
	•	code_block

리스트 계열
	•	list
	•	list_item

표 계열
	•	table
	•	table_row
	•	table_cell

미디어/참조 계열
	•	image
	•	embed
	•	reference
	•	attachment_ref

특수/확장 계열
	•	template_slot
	•	callout
	•	custom_block

각 type에 대해 아래를 검토하라.
	•	왜 필요한가
	•	공통 코어에 포함할지
	•	후속 확장으로 둘지
	•	과도한 세분화는 아닌지

또한 아래도 함께 정리한다.
	•	node_type은 enum처럼 시작할지
	•	확장 가능한 registry/문자열 타입으로 열어둘지

⸻

Step 6. 구조 노드 vs 콘텐츠 노드 구분 검토

아래 질문에 답하라.
	•	section과 paragraph는 같은 수준의 Node 모델로 처리할 수 있는가?
	•	구조를 위한 Node와 실제 콘텐츠를 담는 Node를 분리해야 하는가?
	•	표, 리스트 같은 복합 구조는 부모-자식 Node로 표현 가능한가?
	•	일부 노드는 content 없이 구조 역할만 할 수 있는가?

권장 방향을 정리하라.

예:
	•	하나의 공통 Node 모델을 사용하되,
	•	node_type에 따라 구조적 역할/콘텐츠 역할을 구분하는 방식

⸻

Step 7. Node 콘텐츠 표현 범위 검토

아래 항목별로
Node가 직접 가져야 하는지, metadata/attributes로 보내야 하는지 판단하라.

검토 대상:
	•	일반 텍스트
	•	제목 텍스트
	•	서식 정보
	•	표 셀 내용
	•	이미지 설명
	•	링크
	•	참조 키
	•	템플릿 슬롯 이름
	•	코드 블록 언어 정보

핵심 질문:
	•	Node의 공통 필드를 어디까지 일반화할 것인가?
	•	content 하나로 충분한가?
	•	content_format이 필요한가?
	•	세부 옵션은 attributes/metadata로 넘기는 것이 적절한가?

⸻

Step 8. Node metadata / attributes 범위 검토

이 단계에서 아래를 구분해서 검토하라.
	•	공통 코어 필드로 둘 것
	•	node_type별 확장 속성으로 둘 것
	•	metadata와 attributes를 분리할지 여부

예시:
	•	paragraph 정렬
	•	table cell colspan / rowspan
	•	image alt text
	•	reference target
	•	template placeholder name

권장 방향도 제시하라.

예:
	•	코어는 단순하게 유지
	•	type-specific 속성은 attributes 또는 metadata로 수용

⸻

Step 9. Version과 Node의 관계 정리

아래 관점을 포함해 정리하라.
	•	Node는 특정 Version에 귀속되는가
	•	Node 단위 수정은 새 Version 생성으로 이어지는가
	•	Node 재사용(shared node)을 허용할 것인가
	•	루트 노드가 필요한가
	•	Version이 전체 Node 트리의 기준이 되는가

권장 방향을 제시하라.

일반적으로 초기 설계에서는:
	•	Node는 특정 Version에 종속
	•	Version마다 독립된 Node 트리
	•	공유 Node는 초기에는 도입하지 않음
방향이 유력하다.

⸻

Step 10. 최소 권장 Node 구조안 제시

위 검토를 바탕으로,
Phase 1 기준의 최소 권장 Node 구조안을 제안하라.

형식 예시:
	•	필수 필드
	•	권장 필드
	•	선택 확장 필드
	•	후속 Phase로 미룰 필드

구현 코드가 아니라 개념 모델 명세 형태로 작성하라.

⸻

Step 11. 설계 쟁점 및 판단 기록

아래 쟁점에 대해 판단을 남겨라.
	•	depth/path를 코어에 둘지
	•	title과 content를 분리할지
	•	content_format을 둘지
	•	table_row / table_cell을 별도 type으로 둘지
	•	custom_block을 초기부터 허용할지
	•	metadata와 attributes를 분리할지
	•	shared node를 허용할지

각 항목에 대해:
	•	쟁점
	•	선택한 방향
	•	이유
를 기록하라.

⸻

Step 12. 다음 Task 입력값 정리

문서 마지막에는 반드시 아래를 정리한다.
	•	Task 1-6(DocumentType 시스템 설계)와 연결되는 포인트
	•	Task 1-7(metadata 확장 구조 설계)로 넘길 포인트
	•	저장 구조 설계 시 확인해야 할 Node 관련 쟁점
	•	렌더링/편집기 설계 시 영향을 주는 구조적 판단

⸻

6. 출력 형식 요구사항

반드시 Markdown(.md) 문서로 작성

권장 문서명:
Task5_node_structure_design.md

권장 목차
	1.	작업 목적
	2.	Node의 역할 정의
	3.	Node 필드 후보 목록
	4.	필드 분류 체계
	5.	트리 구조 표현 방식 검토
	6.	Node type 체계 초안
	7.	구조 노드와 콘텐츠 노드 구분
	8.	Node 콘텐츠 표현 범위
	9.	metadata / attributes 범위
	10.	Version과의 관계
	11.	최소 권장 Node 구조안
	12.	설계 쟁점과 판단 결과
	13.	다음 Task 입력값

⸻

7. 작성 방식 요구사항

1) 표를 적극 활용

최소 아래 표를 포함하는 것이 좋다.
	•	필드 후보 분류표
	•	Node type 후보 표
	•	설계 쟁점 판단표

2) 구현 세부사항으로 내려가지 말 것

좋은 예:
	•	“Node는 Version에 종속되는 문서 구조 단위여야 한다.”
	•	“트리 구조는 기본적으로 parent_id + order 방식으로 표현할 수 있어야 한다.”
	•	“type-specific 속성은 확장 필드로 수용하는 것이 바람직하다.”

좋지 않은 예:
	•	ORM 코드
	•	SQL 테이블 DDL
	•	에디터 컴포넌트 코드

3) 범용성과 단순성의 균형을 유지할 것

Node를 지나치게 복잡하게 만들지 말고,
동시에 다양한 문서 구조를 수용할 수 있도록 설계하라.

⸻

8. 산출물

필수 산출물
	1.	Task5_node_structure_design.md

문서 내 반드시 포함되어야 할 내용
	•	Node 역할 정의
	•	Node 필드 후보 목록
	•	트리 구조 표현 방식 판단
	•	Node type 체계 초안
	•	구조 노드/콘텐츠 노드 구분 검토
	•	최소 권장 Node 구조안
	•	후속 Task 입력값

⸻

9. 완료 기준

아래를 모두 만족하면 완료로 본다.
	•	Node의 역할이 Document, Version과 구분되어 명확하다.
	•	Node 필드가 체계적으로 정리되어 있다.
	•	트리 구조 표현 방식이 정리되어 있다.
	•	Node type 체계 초안이 마련되어 있다.
	•	구조 노드와 콘텐츠 노드 처리 방식이 정리되어 있다.
	•	Task 1-6, 1-7, 저장 구조 설계로 자연스럽게 이어질 수 있다.

⸻

10. 작업 시 주의사항
	•	Node를 단순 문자열 레코드로 축소하지 마라.
	•	반대로 너무 복잡한 블록 에디터 전용 모델로 치우치지 마라.
	•	특정 렌더러/에디터 구현을 전제로 고정하지 마라.
	•	table, list 같은 복합 구조를 무리하게 단일 content 필드로만 해결하려 하지 마라.
	•	shared node, advanced tree model 등은 초기 코어에 과도하게 넣지 마라.
	•	MVP와 장기 확장을 구분하라.

⸻

11. Claude Code에게 전달할 한 줄 지시

“Mimir 프로젝트의 Phase 1 - Task 1-5로서, 범용 문서 플랫폼의 Node 엔티티 구조를 설계하는 Markdown 문서를 작성하라. Node의 역할, 필드 후보, 트리 구조 표현 방식, Node type 체계, 구조 노드/콘텐츠 노드 구분, metadata/attributes 범위, 최소 권장 구조안, 후속 Task 입력값까지 포함하되 구현은 하지 마라.”