Claude Code 작업 지시서

Phase 1 - Task 1-7. metadata 확장 구조 설계

1. 작업명

범용 문서 플랫폼의 metadata 확장 구조 설계

2. 작업 목적

Phase 1의 앞선 작업에서 정리한 요구사항, 핵심 엔티티 책임, Document / Version / Node / DocumentType 설계 초안을 바탕으로,
범용 문서 플랫폼에서 공통 코어를 유지하면서도 문서 유형별 차이를 수용할 수 있는 metadata 확장 구조를 설계한다.

이 작업의 목적은 metadata를 단순 JSON 덩어리로 방치하지 않고,
공통 필드와 확장 필드의 경계를 정리하며,
검색/필터/검증/유형별 스키마 확장의 기반이 되는 구조 원칙을 확정하는 것이다.

즉, 이번 작업은 다음을 명확히 하는 단계다.
	•	metadata는 무엇을 표현하는가
	•	어떤 정보는 코어 필드로 두고, 어떤 정보는 metadata로 보낼 것인가
	•	Document / Version / Node 각각에서 metadata를 어떻게 다룰 것인가
	•	공통 metadata와 유형별 확장 metadata를 어떻게 구분할 것인가
	•	타입별 metadata template / schema 연결을 어디까지 허용할 것인가
	•	초기 MVP와 장기 확장의 균형을 어떻게 잡을 것인가

⸻

3. 작업 배경

범용 문서 플랫폼은 다양한 문서 유형을 다루기 때문에,
모든 속성을 Document / Version / Node의 고정 필드로 정의할 수 없다.

예:
	•	policy/regulation: 시행일, 개정일, 관할부서
	•	report: 작성부서, 대상기간, 승인자
	•	meeting_note: 회의일시, 참석자, 안건
	•	manual: 대상 시스템, 버전, 대상 사용자
	•	template_doc: placeholder 정의, 생성 규칙

이런 차이를 수용하려면 metadata 확장 구조가 필요하다.
하지만 metadata를 무제한 자유 필드로 풀어버리면 다음 문제가 생긴다.
	•	구조 일관성 저하
	•	검색/필터 어려움
	•	검증 규칙 부재
	•	타입별 의미 불명확
	•	후속 API/UI/저장 구조 설계 혼란

따라서 이번 작업에서는
“무엇을 코어 필드로 둘지”와 “무엇을 metadata 확장으로 수용할지”의 기준을 세워야 한다.

⸻

4. 작업 범위

포함 범위
	•	metadata의 역할 정의
	•	코어 필드 vs metadata 확장 필드 구분 기준 정리
	•	Document-level / Version-level / Node-level metadata 범위 검토
	•	공통 metadata와 유형별 확장 metadata 구분
	•	metadata 저장 형태에 대한 개념적 방향 정리
	•	metadata template / schema 연결 방식 검토
	•	검색/필터/검증 관점의 metadata 설계 기준 정리
	•	최소 권장 metadata 구조안 제시
	•	후속 Task 입력값 정리

제외 범위
	•	실제 JSON Schema 작성
	•	DB 스키마/ORM 구현
	•	metadata 검증 엔진 구현
	•	API validation 코드 작성
	•	UI 폼 자동 생성 구현
	•	검색 인덱스 구현

즉, 이번 작업은 metadata 개념 모델 및 확장 원칙 설계 문서 작성이다.

⸻

5. Claude Code가 해야 할 일

아래 순서대로 수행하라.

Step 1. metadata의 역할 정의

먼저 metadata가 무엇을 표현하는지 명확히 써라.

반드시 아래 관점을 포함한다.
	•	metadata는 코어 모델이 직접 고정하지 않는 부가 속성을 수용한다.
	•	metadata는 문서 유형별 차이를 표현하는 확장 지점이다.
	•	metadata는 검색/필터/검증/표시 로직의 입력이 될 수 있다.
	•	metadata는 Document / Version / Node 각각에 서로 다른 의미로 존재할 수 있다.
	•	metadata는 코어 구조를 대체하지 않는다.

그리고 아래도 함께 정리한다.
	•	metadata가 책임하는 것
	•	metadata가 책임하지 않는 것
	•	코어 필드가 책임해야 하는 것
	•	DocumentType과의 관계

⸻

Step 2. 코어 필드 vs metadata 구분 기준 정리

이 단계는 매우 중요하다. 반드시 별도 섹션으로 작성하라.

아래 기준으로 어떤 속성을 코어 필드로 둘지, metadata로 둘지 판단 기준을 정리하라.

판단 관점:
	•	거의 모든 문서에 공통으로 필요한가?
	•	정체성/관계/상태/버전 무결성에 직접 영향 주는가?
	•	검색/정렬/참조의 핵심 축인가?
	•	특정 문서 유형에만 필요한가?
	•	향후 확장/변동 가능성이 큰가?

예시 검토:
	•	title → 코어 필드
	•	status → 코어 필드
	•	effective_date → metadata 가능성 높음
	•	meeting_attendees → metadata 가능성 높음
	•	current_version_id → 코어 필드
	•	table cell colspan → Node attributes/metadata 가능성 높음

이 단계의 목적은
**“metadata를 남용하지 않도록 기준을 세우는 것”**이다.

⸻

Step 3. metadata 적용 범위 구분

아래 세 수준으로 구분해서 정리하라.

3-1. Document-level metadata
예:
	•	소속 조직
	•	문서 분류
	•	담당 부서
	•	공개 범위
	•	업무 도메인 분류

3-2. Version-level metadata
예:
	•	시행일
	•	개정 사유
	•	적용 범위
	•	특정 시점 스냅샷 속성

3-3. Node-level metadata / attributes
예:
	•	표 셀 병합 정보
	•	이미지 대체 텍스트
	•	코드 블록 언어
	•	참조 링크 대상
	•	템플릿 placeholder 정보

각 수준에 대해 아래를 정리하라.
	•	어떤 종류의 정보가 적합한가
	•	왜 그 수준에 두는 것이 맞는가
	•	다른 수준과 어떻게 구분되는가

⸻

Step 4. 공통 metadata와 유형별 확장 metadata 구분

아래 두 층으로 나누어 정리하라.

공통 metadata
문서 유형과 무관하게 자주 쓰이거나
여러 유형에서 재사용 가능한 필드들

예:
	•	owner_department
	•	visibility_scope
	•	domain_category
	•	language
	•	keywords

유형별 확장 metadata
특정 DocumentType에 주로 의존하는 필드들

예:
	•	policy/regulation: effective_date, revision_date, governing_body
	•	report: reporting_period, approved_by
	•	meeting_note: meeting_datetime, attendees, agenda_items

이 단계에서는 아래도 함께 정리하라.
	•	공통 metadata는 어느 수준까지 표준화할 것인가
	•	유형별 필드는 namespace처럼 분리할지
	•	key naming 규칙이 필요한지

⸻

Step 5. metadata 저장 구조 방향 검토

이 단계는 개념적 수준에서 정리하라. 구현하지 마라.

아래 관점을 비교 검토하라.

관점 A. 완전 자유 JSON
장점:
	•	유연함
	•	빠른 시작 가능

한계:
	•	혼란
	•	검증 어려움
	•	검색/정렬 어려움

관점 B. 공통 구조 + 자유 확장 필드
예:
	•	common metadata 영역
	•	type-specific metadata 영역

장점:
	•	균형적
	•	관리 가능

관점 C. schema-linked metadata
예:
	•	DocumentType별 metadata template/schema 참조

장점:
	•	검증 가능성 높음
	•	UI 연계 쉬움

한계:
	•	초기 복잡도 증가

Phase 1 기준 권장 방향을 제안하라.

권장 방향 예시:
	•	초기에는 공통 metadata + 유형별 확장 metadata 구조
	•	schema 연결 가능성은 열어두되 강제 검증은 후속으로 미룸

⸻

Step 6. 검색/필터 관점의 metadata 설계 기준 정리

metadata가 단순 보관용이 아니라,
향후 검색과 필터링에 활용될 수 있어야 한다는 관점에서 정리하라.

아래를 검토하라.
	•	어떤 metadata는 검색/필터 주요 축이 될 수 있는가?
	•	어떤 필드는 표준화가 필요한가?
	•	자유 텍스트 metadata와 구조화 metadata를 어떻게 구분할 것인가?
	•	검색에 중요한 필드는 코어 또는 공통 metadata로 올릴 필요가 있는가?

예:
	•	effective_date
	•	department
	•	reporting_period
	•	visibility_scope
	•	attendees

핵심 질문:
	•	나중에 검색/필터에서 자주 쓸 필드는 어디에 두는 것이 적절한가?

⸻

Step 7. metadata 검증 전략 검토

이 단계도 별도 섹션으로 작성하라.

아래를 검토하라.
	•	metadata를 아무 키나 허용할 것인가?
	•	최소한 key naming 규칙은 둘 것인가?
	•	DocumentType별 권장/필수 metadata 필드를 둘 것인가?
	•	schema validation은 언제 도입할 것인가?
	•	초기에는 validation hint 수준으로 둘 것인가?

권장 방향:
	•	초기에는 강제 검증보다 권장 template / optional schema 연결 가능성 중심
	•	후속 Phase에서 stricter validation 도입 가능하게 설계

⸻

Step 8. metadata naming / namespace 규칙 검토

아래를 검토하라.
	•	metadata key naming convention 필요 여부
	•	snake_case / camelCase 중 방향성
	•	유형별 namespace 예:
	•	common.*
	•	policy.*
	•	report.*
	•	meeting.*
	•	충돌 방지 전략 필요 여부

목적은
나중에 필드가 난립하는 것을 막기 위한 최소 기준을 정리하는 것이다.

⸻

Step 9. 최소 권장 metadata 구조안 제시

위 검토를 바탕으로,
Phase 1 기준의 최소 권장 metadata 구조안을 제안하라.

아래를 구분하라.
	•	공통 metadata 영역
	•	유형별 확장 metadata 영역
	•	Document-level metadata 예시
	•	Version-level metadata 예시
	•	Node-level metadata/attributes 예시
	•	MVP에 필요한 요소
	•	후속 Phase로 미룰 요소

구현 코드가 아니라 개념 모델 명세 형태로 작성하라.

⸻

Step 10. 설계 쟁점 및 판단 기록

아래 쟁점에 대해 판단을 남겨라.
	•	metadata를 완전 자유 JSON으로 둘지
	•	공통/유형별 metadata를 분리할지
	•	DocumentType별 schema 연결을 초기부터 둘지
	•	검색 자주 쓰는 필드를 metadata에 둘지 코어로 올릴지
	•	Node-level metadata와 attributes를 분리할지
	•	namespace 규칙을 둘지
	•	필수 metadata 개념을 초기부터 둘지

각 항목에 대해:
	•	쟁점
	•	선택한 방향
	•	이유
를 기록하라.

⸻

Step 11. 다음 Task 입력값 정리

문서 마지막에는 반드시 아래를 정리하라.
	•	Task 1-8(문서 상태 모델 기초 설계)와 연결되는 포인트
	•	저장 구조 설계 시 확인해야 할 metadata 관련 쟁점
	•	API 설계 시 확인해야 할 metadata 처리 기준
	•	검색/필터 설계로 넘길 핵심 판단
	•	UI 문서 생성/편집 플로우에 영향을 주는 판단

⸻

6. 출력 형식 요구사항

반드시 Markdown(.md) 문서로 작성

권장 문서명:
Task7_metadata_extension_design.md

권장 목차
	1.	작업 목적
	2.	metadata의 역할 정의
	3.	코어 필드 vs metadata 구분 기준
	4.	metadata 적용 범위 구분
	5.	공통 metadata와 유형별 확장 metadata
	6.	저장 구조 방향 검토
	7.	검색/필터 관점 설계 기준
	8.	검증 전략 검토
	9.	naming / namespace 규칙
	10.	최소 권장 metadata 구조안
	11.	설계 쟁점과 판단 결과
	12.	다음 Task 입력값

⸻

7. 작성 방식 요구사항

1) 표를 적극 활용

최소 아래 표를 포함하는 것이 좋다.
	•	코어 필드 vs metadata 판단표
	•	수준별 metadata 적용 범위 표
	•	유형별 metadata 예시 표
	•	설계 쟁점 판단표

2) 구현 세부사항으로 내려가지 말 것

좋은 예:
	•	“metadata는 코어 필드로 고정하기 어려운 확장 속성을 수용해야 한다.”
	•	“검색과 필터에서 자주 쓰일 필드는 공통 metadata 또는 코어 필드 승격을 검토해야 한다.”
	•	“초기에는 schema 연결 가능성을 열어두되 강한 검증은 후속으로 미룬다.”

좋지 않은 예:
	•	JSON Schema 코드 작성
	•	DB JSONB 컬럼 설계
	•	API validation 코드 작성

3) 유연성과 통제의 균형을 유지할 것

metadata를 너무 자유롭게 풀지 말고,
반대로 너무 이른 단계에서 엄격한 검증 시스템으로 과설계하지도 마라.

⸻

8. 산출물

필수 산출물
	1.	Task7_metadata_extension_design.md

문서 내 반드시 포함되어야 할 내용
	•	metadata 역할 정의
	•	코어 필드와 metadata 구분 기준
	•	Document/Version/Node 수준별 metadata 범위
	•	공통/유형별 metadata 구분
	•	검증 및 확장 방향
	•	최소 권장 metadata 구조안
	•	후속 Task 입력값

⸻

9. 완료 기준

아래를 모두 만족하면 완료로 본다.
	•	metadata의 역할과 한계가 명확히 정의되어 있다.
	•	코어 필드와 metadata의 경계가 정리되어 있다.
	•	수준별 metadata 적용 범위가 구분되어 있다.
	•	공통/유형별 metadata 구조 방향이 정리되어 있다.
	•	검색/필터/검증 확장 관점이 반영되어 있다.
	•	Task 1-8, 저장 구조, API 설계로 자연스럽게 이어질 수 있다.

⸻

10. 작업 시 주의사항
	•	metadata를 만능 저장소처럼 쓰지 마라.
	•	코어 필드로 가야 할 속성을 무조건 metadata로 밀어 넣지 마라.
	•	특정 문서 유형 하나에 최적화하지 마라.
	•	검증 시스템을 초기부터 과도하게 강제하지 마라.
	•	검색/필터에서 자주 쓰일 필드를 간과하지 마라.
	•	Node-level metadata와 Document-level metadata를 혼동하지 마라.
	•	MVP와 장기 확장을 구분하라.

⸻

11. Claude Code에게 전달할 한 줄 지시

“Mimir 프로젝트의 Phase 1 - Task 1-7로서, 범용 문서 플랫폼의 metadata 확장 구조를 설계하는 Markdown 문서를 작성하라. metadata의 역할, 코어 필드와 metadata 구분 기준, Document/Version/Node 수준별 metadata 범위, 공통/유형별 metadata 구조, 검증 및 확장 방향, 최소 권장 구조안, 후속 Task 입력값까지 포함하되 구현은 하지 마라.”