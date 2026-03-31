Claude Code 작업 지시서

Phase 1 - Task 1-3. Document 구조 설계

1. 작업명

범용 문서 플랫폼의 Document 구조 설계

2. 작업 목적

Phase 1 - Task 1-1, 1-2에서 정리한 요구사항과 핵심 엔티티 책임 정의를 바탕으로,
범용 문서 플랫폼에서 문서 자체의 정체성을 대표하는 최상위 엔티티인 Document의 구조를 상세 설계한다.

이 작업의 목적은 Document가 무엇을 담고, 무엇을 담지 않아야 하는지 명확히 정리하여
이후 Version, Node, 상태 모델, API, 저장 구조 설계의 기준점을 만드는 것이다.

즉, 이번 작업은 “문서의 본문을 어떻게 저장할까”보다는
Document라는 엔티티가 플랫폼에서 어떤 역할을 하는가를 구조적으로 고정하는 단계다.

⸻

3. 작업 배경

범용 문서 플랫폼에서 Document는 단순한 파일 레코드가 아니다.
Document는 하나의 문서를 식별하고, 그 문서의 유형, 대표 제목, 현재 상태, 현재 버전 참조 등
문서의 지속적인 정체성(identity) 을 표현하는 중심 개념이다.

반면 실제 본문 구조나 특정 시점의 문서 내용은 Version / Node가 담당해야 한다.
따라서 이번 작업에서는 특히 다음을 명확히 해야 한다.
	•	Document가 직접 가지는 정보는 무엇인가
	•	Version과 분리해야 할 정보는 무엇인가
	•	문서 정체성과 현재 시점 요약 정보는 어디까지 포함할 것인가
	•	상태와 버전 참조를 어떻게 표현할 것인가

⸻

4. 작업 범위

포함 범위
	•	Document 엔티티의 역할 재정의
	•	Document 필드 후보 도출
	•	필수 필드와 선택 필드 구분
	•	Version과의 책임 분리 정리
	•	상태, 유형, 대표 정보, 참조 정보 구조 정리
	•	Document 수준 metadata 포함 범위 검토
	•	후속 Task를 위한 입력값 정의

제외 범위
	•	Version 상세 설계
	•	Node 상세 설계
	•	DB 스키마/ORM 코드 작성
	•	API 요청/응답 설계
	•	권한/승인/배포 상세 설계
	•	실제 구현 코드 작성

이번 작업은 Document 도메인 구조 명세서 작성이다.

⸻

5. Claude Code가 해야 할 일

아래 순서대로 수행하라.

Step 1. Document의 역할 재정리

먼저 Document가 무엇을 대표하는지 다시 명확히 써라.

반드시 아래 관점을 포함한다.
	•	Document는 문서의 지속적인 정체성을 대표한다.
	•	Document는 특정 버전의 내용 전체를 직접 담는 엔티티가 아니다.
	•	Document는 현재 유효한 버전을 가리킬 수 있다.
	•	Document는 문서 유형과 상태를 가진다.
	•	Document는 대표 제목/요약/식별 정보를 가질 수 있다.

그리고 아래도 함께 정리한다.
	•	Document가 책임하지 않는 것
	•	Version이 책임해야 하는 것
	•	Node가 책임해야 하는 것

⸻

Step 2. Document 필드 후보 도출

Document가 가질 수 있는 필드 후보를 최대한 체계적으로 나열하고 분류한다.

최소 아래 후보를 검토한다.
	•	id
	•	external_id 또는 slug
	•	title
	•	summary 또는 description
	•	document_type
	•	status
	•	current_version_id
	•	latest_version_id
	•	created_by
	•	created_at
	•	updated_at
	•	archived_at
	•	deleted_at 또는 is_deleted
	•	metadata
	•	tags 또는 분류 정보
	•	owner / organization / workspace scope

필드별로 아래를 판단한다.
	•	필수 여부
	•	선택 여부
	•	MVP 포함 여부
	•	후속 확장으로 미룰지 여부

⸻

Step 3. 필드 분류 체계 정리

Document 필드를 아래 범주로 나누어 정리한다.

3-1. 식별 필드
예:
	•	id
	•	external_id / slug

3-2. 대표 정보 필드
예:
	•	title
	•	summary

3-3. 분류/유형 필드
예:
	•	document_type
	•	category
	•	tags

3-4. 상태/생명주기 필드
예:
	•	status
	•	archived_at
	•	deleted flag

3-5. 버전 참조 필드
예:
	•	current_version_id
	•	latest_version_id

3-6. 감사/추적 필드
예:
	•	created_by
	•	created_at
	•	updated_at

3-7. 확장 필드
예:
	•	metadata
	•	tenant/workspace scope

이 단계에서는 각 범주가 왜 필요한지도 함께 설명해야 한다.

⸻

Step 4. 필수 필드 vs 선택 필드 구분

각 필드에 대해 다음을 정리하라.
	•	왜 반드시 필요한가
	•	왜 선택 사항이어야 하는가
	•	지금 MVP에 포함해야 하는가
	•	후속 단계로 미뤄도 되는가

권장 출력 형식:

필드명	의미	필수 여부	MVP 포함 여부	비고



⸻

Step 5. Document와 Version의 책임 경계 명확화

이 단계는 매우 중요하다. 반드시 별도 섹션으로 작성하라.

아래 질문에 답하는 식으로 정리한다.
	•	title은 Document에 둘지 Version에 둘지, 또는 둘 다 둘지?
	•	metadata는 Document 수준과 Version 수준을 어떻게 나눌지?
	•	상태는 Document의 상태인가, 특정 Version의 상태인가?
	•	current_version_id와 latest_version_id를 분리해야 하는가?
	•	수정 시 updated_at은 Document 기준인지 Version 기준인지?

이 단계에서 중요한 것은
Document를 너무 비대하게 만들지 않으면서도 실용성을 유지하는 것이다.

필요하다면 아래와 같이 정리한다.
	•	Document는 “현재 대표 정보”를 가질 수 있다.
	•	실제 시점별 본문/세부 metadata는 Version이 가진다.
	•	Document의 title은 현재 대표 제목으로 두고, Version에도 snapshot title을 둘 수 있다.

⸻

Step 6. Document metadata 범위 검토

metadata를 무조건 하나로 보지 말고 아래처럼 나누어 검토한다.
	•	Document-level metadata
	•	문서 자체의 소속/분류/운영 정보
	•	Version-level metadata
	•	특정 버전에 종속되는 속성
	•	Node-level metadata
	•	특정 본문 블록/섹션에 종속되는 속성

그리고 Document 수준 metadata에는 어떤 것이 적합한지 정리한다.

예:
	•	소유 조직
	•	문서 담당 부서
	•	문서 분류
	•	공개 범위
	•	커스텀 분류값

⸻

Step 7. 최소 권장 Document 구조안 제시

위 검토를 바탕으로,
Phase 1 시점에서 권장하는 최소 Document 구조안을 제안한다.

형식은 예시적으로 아래처럼 정리할 수 있다.
	•	필수 필드
	•	권장 필드
	•	선택 확장 필드
	•	후속 Phase로 미룰 필드

이때 구현 코드가 아니라 개념 모델 명세 형태로 작성한다.

⸻

Step 8. 설계 쟁점 및 판단 기록

아래 쟁점에 대해 판단을 남겨라.
	•	current_version_id와 latest_version_id를 둘 다 둘 것인가
	•	summary를 Document에 둘 것인가
	•	tags를 코어에 포함할 것인가
	•	삭제 표현을 soft delete로 열어둘 것인가
	•	external_id/slug를 초기에 둘 것인가

각 항목에 대해:
	•	쟁점
	•	선택한 방향
	•	이유
를 남긴다.

⸻

Step 9. 다음 Task 입력값 정리

문서 마지막에는 반드시 아래를 정리한다.
	•	Task 1-4(Version 구조 설계)로 넘길 쟁점
	•	Task 1-5(Node 구조 설계)와 연결되는 포인트
	•	Document 상태 모델이 Task 1-8과 연결되는 방식

⸻

6. 출력 형식 요구사항

반드시 Markdown(.md) 문서로 작성

권장 문서명:
Task3_document_structure_design.md

권장 목차
	1.	작업 목적
	2.	Document의 역할 정의
	3.	Document 필드 후보 목록
	4.	필드 분류 체계
	5.	필수/선택 필드 판단
	6.	Document와 Version의 책임 경계
	7.	Document-level metadata 범위
	8.	최소 권장 Document 구조안
	9.	설계 쟁점과 판단 결과
	10.	다음 Task 입력값

⸻

7. 작성 방식 요구사항

1) 표를 적극 활용

최소 아래 표를 포함하는 것이 좋다.
	•	필드 후보 분류표
	•	필수/선택 필드 판단표
	•	설계 쟁점 판단표

2) 구현 세부사항으로 내려가지 말 것

좋은 예:
	•	“Document는 current_version_id를 통해 현재 대표 버전을 참조할 수 있어야 한다.”
	•	“Document는 대표 제목을 보유할 수 있다.”
	•	“시점별 상세 내용은 Version이 담당한다.”

좋지 않은 예:
	•	SQL DDL 작성
	•	ORM 클래스 코드 작성
	•	API JSON 스키마 작성

3) 책임 경계를 우선할 것

필드 나열보다 더 중요한 것은
Document가 무엇을 담당하고 무엇을 담당하지 않는지 명확히 하는 것이다.

⸻

8. 산출물

필수 산출물
	1.	Task3_document_structure_design.md

문서 내 반드시 포함되어야 할 내용
	•	Document 역할 정의
	•	Document 필드 후보 목록
	•	필드 필수/선택 판단
	•	Document와 Version의 책임 경계
	•	최소 권장 Document 구조안
	•	후속 Task 입력값

⸻

9. 완료 기준

아래를 모두 만족하면 완료로 본다.
	•	Document의 정체성과 역할이 명확하게 정의되어 있다.
	•	Document 필드가 체계적으로 분류되어 있다.
	•	필수 필드와 선택 필드가 구분되어 있다.
	•	Version과 책임이 섞이지 않도록 경계가 정리되어 있다.
	•	metadata 범위가 구분되어 있다.
	•	Task 1-4, 1-5, 1-8로 자연스럽게 이어질 수 있다.

⸻

10. 작업 시 주의사항
	•	Document를 “본문까지 다 담는 거대한 엔티티”로 만들지 마라.
	•	Version과 역할이 겹치지 않게 하라.
	•	Node 구조 상세로 미리 넘어가지 마라.
	•	tags, attachment, approval 등을 무조건 코어에 넣지 마라.
	•	MVP와 장기 확장을 구분하라.
	•	현재 대표 정보와 시점별 스냅샷 정보를 혼동하지 마라.

⸻

11. Claude Code에게 전달할 한 줄 지시

“Mimir 프로젝트의 Phase 1 - Task 1-3으로서, 범용 문서 플랫폼의 최상위 엔티티인 Document의 구조를 설계하는 Markdown 문서를 작성하라. Document의 역할, 필드 후보, 필수/선택 판단, Version과의 책임 경계, metadata 범위, 최소 권장 구조안, 후속 Task 입력값까지 포함하되 구현은 하지 마라.”
