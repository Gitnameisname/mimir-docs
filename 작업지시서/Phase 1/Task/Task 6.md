Claude Code 작업 지시서

Phase 1 - Task 1-6. DocumentType 시스템 설계

1. 작업명

범용 문서 플랫폼의 DocumentType 시스템 설계

2. 작업 목적

Phase 1의 앞선 작업에서 정리한 요구사항, 핵심 엔티티 책임, Document / Version / Node 구조 초안을 바탕으로,
범용 문서 플랫폼에서 문서 유형을 정의하고 확장하는 DocumentType 시스템의 구조와 역할을 설계한다.

이 작업의 목적은 단순히 문서 종류 이름을 나열하는 것이 아니라,
DocumentType이 플랫폼 내에서 어떤 의미를 가지며,
어디까지 강제력을 갖고, 어떤 확장 지점을 제공해야 하는지를 정리하는 것이다.

즉, 이번 작업은 다음을 확정하는 단계다.
	•	DocumentType은 단순 분류값인지, 규칙 시스템의 기준인지
	•	DocumentType은 Document와 어떤 관계를 가지는지
	•	유형별로 어떤 차이를 표현해야 하는지
	•	Node 구조, metadata, 상태 모델과 어떻게 연결되는지
	•	초기 MVP에서 어디까지 지원하고, 이후 어떻게 확장할지

⸻

3. 작업 배경

범용 문서 플랫폼은 하나의 공통 코어 모델 위에서
여러 문서 유형을 수용해야 한다.

예:
	•	policy / regulation
	•	manual / guide
	•	technical document
	•	report
	•	meeting note
	•	generic knowledge document
	•	template-based document

이때 모든 문서를 완전히 동일하게 취급하면 범용성은 높지만 실용성이 떨어질 수 있다.
반대로 문서 유형별로 완전히 별도 모델을 만들면 플랫폼이 파편화된다.

따라서 DocumentType 시스템은
공통 코어 모델을 유지하면서도 유형별 차이와 확장 규칙을 수용하는 중간 계층이 되어야 한다.

이번 작업에서는 특히 아래를 명확히 해야 한다.
	•	DocumentType의 정체는 무엇인가
	•	DocumentType은 어디까지 플랫폼 동작에 영향을 주는가
	•	초기 타입 집합은 무엇으로 둘 것인가
	•	유형별 metadata 템플릿, Node 규칙, 상태 확장과 연결할지 여부
	•	향후 사용자 정의 타입/플러그인형 확장을 열어둘지

⸻

4. 작업 범위

포함 범위
	•	DocumentType의 역할 정의
	•	DocumentType을 단순 enum으로 볼지, 확장 가능한 시스템으로 볼지 검토
	•	초기 기본 타입 목록 설계
	•	유형별 차이를 어떤 방식으로 반영할지 정리
	•	Node 규칙과의 연결 가능성 검토
	•	metadata 템플릿과의 연결 가능성 검토
	•	상태 모델과의 연결 가능성 검토
	•	확장 전략 정리
	•	최소 권장 DocumentType 구조안 제시
	•	후속 Task 입력값 정리

제외 범위
	•	실제 코드 구현
	•	DB 스키마/ORM 설계
	•	UI 폼 생성 로직 구현
	•	검증 엔진 구현
	•	권한/승인 워크플로 구현
	•	플러그인 시스템 구현

즉, 이번 작업은 DocumentType 개념 모델 및 시스템 설계 문서 작성이다.

⸻

5. Claude Code가 해야 할 일

아래 순서대로 수행하라.

Step 1. DocumentType의 역할 정의

먼저 DocumentType이 무엇을 대표하는지 명확히 써라.

반드시 아래 관점을 포함한다.
	•	DocumentType은 문서의 유형 분류를 표현한다.
	•	DocumentType은 단순 라벨을 넘어서 확장 규칙의 기준이 될 수 있다.
	•	DocumentType은 공통 코어 모델(Document / Version / Node)을 대체하지 않는다.
	•	DocumentType은 유형별 metadata, Node 구성, 상태 정책과 연결될 수 있다.
	•	DocumentType은 범용성과 특수성 사이의 균형 장치다.

그리고 아래도 함께 정리한다.
	•	DocumentType이 책임하는 것
	•	DocumentType이 책임하지 않는 것
	•	Document가 책임해야 하는 것
	•	metadata/상태/Node와의 관계

⸻

Step 2. DocumentType의 성격 검토

아래 세 가지 관점을 비교 검토하라.

관점 A. 단순 enum / 분류값
예:
	•	policy
	•	manual
	•	report
	•	meeting_note

장점:
	•	단순함
	•	구현 쉬움
	•	초기 MVP에 적합

한계:
	•	확장성 낮음
	•	유형별 규칙 연결이 약함

관점 B. 관리 가능한 타입 정의 개체
예:
	•	type key
	•	display name
	•	description
	•	allowed rules reference
	•	metadata schema reference

장점:
	•	확장성 높음
	•	규칙 연결 가능

한계:
	•	설계 복잡도 증가

관점 C. 플러그인형/사용자 정의 타입 시스템
장점:
	•	장기적 유연성 매우 높음

한계:
	•	초기 과설계 위험 큼

이 세 관점을 비교하고,
Phase 1 기준 권장 방향을 제안하라.

⸻

Step 3. 초기 기본 DocumentType 후보 설계

최소 아래 타입들을 검토 대상으로 포함하라.
	•	policy
	•	regulation
	•	manual
	•	guide
	•	technical_doc
	•	report
	•	meeting_note
	•	generic_doc
	•	template_doc

각 타입에 대해 아래를 간단히 정리하라.
	•	문서 목적
	•	구조적 특징
	•	metadata 특징
	•	상태 관리 특징
	•	다른 타입과 구분되는 포인트

주의:
	•	너무 세밀한 하위 분류로 과도하게 쪼개지 말 것
	•	policy와 regulation처럼 유사 타입은 분리 필요성을 검토할 것

⸻

Step 4. 타입 체계 수준 정리

이 단계에서는 초기 타입 체계를 아래와 같이 정리하라.
	•	코어 기본 타입
	•	별도 분리 필요성이 낮은 타입
	•	후속 확장으로 미룰 타입
	•	custom/user-defined 가능성이 있는 타입

예시 질문:
	•	manual과 guide는 분리할 것인가?
	•	policy와 regulation은 코어에서 통합할 것인가?
	•	generic_doc는 fallback type으로 둘 것인가?
	•	template_doc는 문서 자체의 타입인지, 생성 방식의 속성인지?

이 단계에서 타입 개수 최소화 원칙을 유지하라.

⸻

Step 5. DocumentType과 Document의 관계 정리

아래 질문에 답할 수 있도록 정리하라.
	•	Document는 하나의 DocumentType을 반드시 가져야 하는가?
	•	type 변경이 허용되는가?
	•	DocumentType은 Document의 필드인가, 별도 관리 대상인가?
	•	Version별로 type이 달라질 수 있는가?
	•	type은 현재 문서 정체성에 속하는가, 시점별 snapshot 대상인가?

권장 방향도 제시하라.

일반적으로는:
	•	Document는 하나의 현재 DocumentType을 가진다.
	•	Type 변경은 제한적으로 허용 가능하나 신중해야 한다.
	•	Version에는 필요 시 type snapshot을 둘 수 있다.
방향이 유력하다.

⸻

Step 6. DocumentType과 Node 규칙 연결 검토

이 단계는 매우 중요하다. 반드시 별도 섹션으로 작성하라.

아래를 검토하라.
	•	특정 DocumentType에서 특정 Node type만 허용할 것인가?
	•	필수 구조를 요구할 것인가?
	•	예: meeting_note는 참석자/안건/결정사항 구조를 권장할 수 있음
	•	policy는 조/항 구조를 가질 수 있음
	•	이 제약을 강제할지, 권장 수준으로 둘지
	•	MVP에서는 어디까지 반영할지

핵심 질문:
	•	DocumentType이 Node 구조를 “강제”하는가, “가이드”하는가?

권장 방향:
	•	초기에는 강한 강제보다 권장 규칙 / validation hint 수준으로 시작
	•	후속 Phase에서 구조 검증으로 확장 가능하게 설계

⸻

Step 7. DocumentType과 metadata 템플릿 연결 검토

이 단계도 별도 섹션으로 작성하라.

아래를 검토하라.
	•	각 DocumentType별 권장 metadata 필드가 필요한가?
	•	metadata schema/template를 타입별로 연결할 것인가?
	•	필수/권장 메타 필드 구분이 필요한가?
	•	metadata 검증을 타입 시스템에 연결할 것인가?

예:
	•	policy/regulation: 시행일, 개정일, 관할부서
	•	report: 작성부서, 보고대상기간, 승인자
	•	meeting_note: 회의일시, 참석자, 안건

권장 방향:
	•	초기에는 타입별 metadata template 정의 가능성을 열어두되
	•	실제 강한 검증은 후속 Phase로 미룬다

⸻

Step 8. DocumentType과 상태 모델 연결 검토

아래를 검토하라.
	•	모든 DocumentType이 동일 상태 모델을 공유하는가?
	•	특정 타입은 추가 상태가 필요한가?
	•	예: report는 submitted 상태 필요 가능
	•	policy는 effective / deprecated 개념 필요 가능
	•	초기에는 공통 상태 모델을 유지할지
	•	타입별 상태 확장은 언제 도입할지

권장 방향:
	•	초기에는 공통 상태 모델 유지
	•	타입별 상태 확장은 후속 확장 포인트로만 남김

⸻

Step 9. DocumentType 확장 전략 정리

아래 확장 시나리오를 검토하라.
	•	기본 내장 타입만 제공
	•	관리자 정의 타입 추가 가능
	•	사용자 정의 타입 가능
	•	외부 시스템 연동형 타입 가능

그리고 아래를 판단하라.
	•	Phase 1에서 어디까지 열어둘 것인가
	•	어떤 확장은 후속 Phase로 미룰 것인가
	•	타입 식별자는 문자열 key 기반이 적절한가
	•	registry 방식으로 가는 것이 적절한가

권장 방향:
	•	초기에는 내장 타입 + 확장 가능한 key 구조
	•	실제 동적 타입 생성은 후속 Phase 검토

⸻

Step 10. 최소 권장 DocumentType 구조안 제시

위 검토를 바탕으로,
Phase 1 기준의 최소 권장 DocumentType 구조안을 제안하라.

다음 항목을 검토 대상으로 포함하라.
	•	key
	•	display_name
	•	description
	•	category/group
	•	metadata template reference
	•	node rule reference
	•	state profile reference
	•	is_builtin
	•	is_active

단, 구현 코드가 아니라 개념 모델 명세 형태로 작성하라.

그리고 아래를 구분하라.
	•	MVP에 필요한 요소
	•	권장 확장 요소
	•	후속 Phase로 미룰 요소

⸻

Step 11. 설계 쟁점 및 판단 기록

아래 쟁점에 대해 판단을 남겨라.
	•	DocumentType을 enum 수준으로 시작할지
	•	policy와 regulation을 분리할지
	•	manual과 guide를 분리할지
	•	template_doc를 type으로 둘지
	•	type별 Node 규칙을 강제할지
	•	type별 metadata 검증을 코어에 넣을지
	•	custom type을 초기부터 허용할지

각 항목에 대해:
	•	쟁점
	•	선택한 방향
	•	이유
를 기록하라.

⸻

Step 12. 다음 Task 입력값 정리

문서 마지막에는 반드시 아래를 정리한다.
	•	Task 1-7(metadata 확장 구조 설계)로 넘길 포인트
	•	Task 1-8(문서 상태 모델 기초 설계)와 연결되는 포인트
	•	저장 구조/API 설계에서 확인해야 할 DocumentType 관련 쟁점
	•	UI/문서 생성 플로우에서 영향을 받을 판단

⸻

6. 출력 형식 요구사항

반드시 Markdown(.md) 문서로 작성

권장 문서명:
Task6_document_type_system_design.md

권장 목차
	1.	작업 목적
	2.	DocumentType의 역할 정의
	3.	DocumentType의 성격 검토
	4.	초기 기본 타입 후보
	5.	타입 체계 수준 정리
	6.	Document와의 관계
	7.	Node 규칙과의 연결
	8.	metadata 템플릿과의 연결
	9.	상태 모델과의 연결
	10.	확장 전략
	11.	최소 권장 DocumentType 구조안
	12.	설계 쟁점과 판단 결과
	13.	다음 Task 입력값

⸻

7. 작성 방식 요구사항

1) 표를 적극 활용

최소 아래 표를 포함하는 것이 좋다.
	•	기본 타입 후보 비교표
	•	타입 체계 분류표
	•	설계 쟁점 판단표

2) 구현 세부사항으로 내려가지 말 것

좋은 예:
	•	“DocumentType은 공통 코어 모델 위에 유형별 차이를 얹는 분류 및 확장 기준이어야 한다.”
	•	“초기에는 강한 구조 강제보다 권장 규칙 수준으로 시작하는 것이 적절하다.”
	•	“타입별 metadata template 연결 가능성을 열어두되 강한 검증은 후속으로 미룬다.”

좋지 않은 예:
	•	enum 코드 작성
	•	DB 테이블 설계
	•	타입 검증 엔진 구현

3) 범용성과 확장성의 균형을 유지할 것

타입 체계를 너무 세밀하게 만들지 말고,
동시에 후속 Phase에서 확장 가능한 구조를 남겨라.

⸻

8. 산출물

필수 산출물
	1.	Task6_document_type_system_design.md

문서 내 반드시 포함되어야 할 내용
	•	DocumentType 역할 정의
	•	성격 검토(enum vs 관리형 타입 vs 확장형)
	•	초기 기본 타입 후보
	•	Node/metadata/상태 모델과의 연결 검토
	•	최소 권장 DocumentType 구조안
	•	후속 Task 입력값

⸻

9. 완료 기준

아래를 모두 만족하면 완료로 본다.
	•	DocumentType의 역할이 명확히 정의되어 있다.
	•	초기 타입 집합이 과도하지 않게 정리되어 있다.
	•	DocumentType과 Document/Node/metadata/상태 모델의 관계가 정리되어 있다.
	•	초기 MVP와 장기 확장 방향이 구분되어 있다.
	•	Task 1-7, 1-8, 저장 구조 설계로 자연스럽게 이어질 수 있다.

⸻

10. 작업 시 주의사항
	•	DocumentType을 단순 UI 분류값으로만 축소하지 마라.
	•	반대로 초기부터 과도한 플러그인 시스템으로 과설계하지 마라.
	•	특정 문서 유형 하나에 끌려가지 마라.
	•	타입 수를 지나치게 늘리지 마라.
	•	Node 강제 규칙과 metadata 검증을 초기에 너무 강하게 넣지 마라.
	•	MVP와 장기 확장을 구분하라.

⸻

11. Claude Code에게 전달할 한 줄 지시

“Mimir 프로젝트의 Phase 1 - Task 1-6으로서, 범용 문서 플랫폼의 DocumentType 시스템을 설계하는 Markdown 문서를 작성하라. DocumentType의 역할, enum/관리형 타입/확장형 관점 비교, 초기 기본 타입 후보, Node/metadata/상태 모델과의 연결, 확장 전략, 최소 권장 구조안, 후속 Task 입력값까지 포함하되 구현은 하지 마라.”