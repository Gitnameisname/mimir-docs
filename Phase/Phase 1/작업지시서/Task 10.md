Claude Code 작업 지시서

Phase 1 - Task 1-10. 통합 모델 리뷰 및 기준안 확정

1. 작업명

범용 문서 플랫폼의 통합 모델 리뷰 및 기준안 확정

2. 작업 목적

Phase 1의 Task 1-1부터 1-9까지 정리한 요구사항, 핵심 엔티티, Document / Version / Node 구조, DocumentType 시스템, metadata 확장 구조, 상태 모델, 관계/무결성/제약 조건을 하나의 일관된 기준안으로 통합한다.

이 작업의 목적은 단순 요약이 아니다.
앞선 작업 결과들 사이에 남아 있을 수 있는 중복, 충돌, 과설계, 불명확한 경계를 정리하여,
이후 Phase 2(저장 구조/DB 설계), Phase 3(API 설계), Phase 4(UI/편집/렌더링 설계)의 기준이 되는 Phase 1 최종 기준 문서를 확정하는 것이다.

즉, 이번 작업은 다음을 수행하는 단계다.
	•	앞선 Task 결과를 하나의 통합 모델로 정리
	•	상충되는 판단을 조정
	•	MVP 기준과 후속 확장 항목을 분리
	•	코어 모델의 최종 경계를 고정
	•	다음 Phase에서 바로 사용할 수 있는 입력 기준 확정

⸻

3. 작업 배경

Phase 1에서는 다음 요소들을 개별적으로 설계했다.
	•	범용 문서 도메인 요구사항
	•	핵심 엔티티 및 책임
	•	Document 구조
	•	Version 구조
	•	Node 구조
	•	DocumentType 시스템
	•	metadata 확장 구조
	•	문서 상태 모델
	•	관계 / 무결성 / 제약 조건

하지만 실제 시스템 기준안은
이 각각의 문서가 따로 존재하는 상태로는 충분하지 않다.
최종적으로는 아래 질문들에 답할 수 있어야 한다.
	•	플랫폼의 코어 엔티티는 정확히 무엇인가
	•	각 엔티티는 무엇을 책임하는가
	•	무엇이 MVP 범위인가
	•	무엇이 확장 포인트인가
	•	후속 설계에서 반드시 유지해야 할 원칙은 무엇인가

이번 작업에서는 이 답을 하나의 기준 문서로 정리해야 한다.

⸻

4. 작업 범위

포함 범위
	•	Task 1-1 ~ 1-9 결과 통합 정리
	•	중복/충돌/불명확성 정리
	•	코어 모델 최종 기준안 확정
	•	MVP 범위와 후속 확장 범위 분리
	•	후속 Phase 입력 기준 정리
	•	최종 권장 모델 요약 정리

제외 범위
	•	새로운 엔티티 추가 설계
	•	DB 스키마 설계
	•	API 명세 작성
	•	UI 설계
	•	실제 코드 구현
	•	승인/배포 상세 워크플로 설계

즉, 이번 작업은 Phase 1 최종 기준 문서 작성이다.

⸻

5. Claude Code가 해야 할 일

아래 순서대로 수행하라.

Step 1. 앞선 Task 결과 요약 정리

Task 1-1부터 1-9까지의 결과를 짧게 요약하라.

최소 아래 항목을 포함한다.
	•	Task 1-1: 요구사항
	•	Task 1-2: 핵심 엔티티와 책임
	•	Task 1-3: Document 구조
	•	Task 1-4: Version 구조
	•	Task 1-5: Node 구조
	•	Task 1-6: DocumentType 시스템
	•	Task 1-7: metadata 확장 구조
	•	Task 1-8: 상태 모델
	•	Task 1-9: 관계/무결성/제약

이 단계의 목적은
최종 문서 초반에 “무엇을 통합하는지”를 보여주는 것이다.

⸻

Step 2. 코어 모델 구성요소 최종 확정

Phase 1 기준에서 플랫폼 코어 모델을 구성하는 요소를 최종 확정하라.

최소 아래 항목을 포함한다.
	•	Document
	•	Version
	•	Node
	•	DocumentType
	•	metadata 구조
	•	Document 상태 모델

그리고 각 항목에 대해 아래를 정리하라.
	•	한 줄 정의
	•	핵심 책임
	•	왜 코어에 포함되는가
	•	무엇은 코어 바깥에 두는가

이 단계에서 특히
엔티티 / 개념 / 정책 / 확장 포인트를 구분하라.

예:
	•	Document, Version, Node → 코어 엔티티
	•	DocumentType → 코어 분류/규칙 개념
	•	metadata → 코어 확장 구조
	•	상태 모델 → 코어 운영 개념
	•	Approval, Attachment, Publication → 후속 확장 가능성

⸻

Step 3. 최종 책임 분리 기준안 작성

이 단계는 매우 중요하다. 반드시 별도 섹션으로 작성하라.

아래 질문에 답할 수 있도록 정리하라.
	•	Document는 무엇을 책임하는가
	•	Version은 무엇을 책임하는가
	•	Node는 무엇을 책임하는가
	•	DocumentType은 무엇을 책임하는가
	•	metadata는 무엇을 책임하는가
	•	상태 모델은 무엇을 책임하는가

그리고 반드시 아래도 정리하라.
	•	각 요소가 책임하지 않는 것
	•	서로 겹치기 쉬운 부분의 최종 경계
	•	Document vs Version
	•	Version vs Node
	•	코어 필드 vs metadata
	•	상태 vs 삭제 정책
	•	DocumentType vs 템플릿/승인정책

⸻

Step 4. 최종 관계 구조 요약

아래 핵심 관계를 기준으로 최종 관계 구조를 정리하라.
	•	Document 1 : N Version
	•	Version 1 : N Node
	•	Document N : 1 DocumentType
	•	metadata는 Document / Version / Node 수준에서 부속 구조로 존재
	•	상태는 초기에는 Document 중심으로 적용

이 단계에서는 아래도 함께 정리하라.
	•	current_version의 의미
	•	root node의 의미
	•	type-specific metadata의 위치
	•	Version snapshot과 Document current 정보의 관계

⸻

Step 5. 최소 권장 코어 모델 기준안 제시

위 통합 내용을 바탕으로,
Phase 1 기준의 최소 권장 코어 모델 기준안을 제시하라.

아래 항목을 포함한다.

5-1. Document 최소 기준안
	•	필수 필드 개념
	•	대표 정보
	•	버전 참조
	•	상태
	•	타입

5-2. Version 최소 기준안
	•	문서 시점 단위
	•	snapshot 정보
	•	Node 트리 기준
	•	불변성 원칙

5-3. Node 최소 기준안
	•	Version 종속 구조 단위
	•	트리 구조
	•	node_type
	•	콘텐츠/확장 속성

5-4. DocumentType 최소 기준안
	•	초기 기본 타입 집합
	•	확장 가능 key 구조
	•	강제보다 권장 중심

5-5. metadata 최소 기준안
	•	공통 metadata + 유형별 확장 metadata
	•	레벨별(Document / Version / Node) 구분

5-6. 상태 모델 최소 기준안
	•	draft / review / published / archived
	•	deleted는 상태보다 삭제 정책 쪽으로 분리

⸻

Step 6. 최소 불변조건 최종 요약

Task 1-9에서 정리한 내용을 바탕으로,
플랫폼 코어 모델이 반드시 만족해야 할 최종 핵심 불변조건을 요약하라.

최소 아래 성격을 포함한다.
	•	참조 정합성
	•	버전 정합성
	•	트리 정합성
	•	상태 정합성
	•	metadata 최소 원칙

이 단계에서는 너무 많은 세부 규칙을 재나열하기보다,
후속 Phase의 기준이 되는 핵심 invariant 세트를 압축 정리하라.

⸻

Step 7. MVP 범위와 후속 확장 범위 분리

이 단계는 반드시 별도 섹션으로 작성하라.

아래 두 범주로 나누어 정리하라.

MVP 범위
Phase 2~4에서 바로 구현 기준으로 삼을 최소 범위

예:
	•	Document / Version / Node 기본 구조
	•	parent_id + order 기반 트리
	•	기본 DocumentType 집합
	•	공통 상태 모델
	•	metadata 확장 구조의 기본 틀
	•	current_version 중심 구조

후속 확장 범위
지금은 개념만 열어두고 구현은 미루는 항목

예:
	•	type별 강한 구조 검증
	•	schema 기반 metadata validation
	•	Version별 published marker 강화
	•	Approval / Publication 엔티티
	•	custom DocumentType 동적 생성
	•	shared node
	•	advanced diff model
	•	deprecated/superseded 상태 확장

이 단계의 목적은
후속 구현에서 범위가 번지는 것을 막는 것이다.

⸻

Step 8. 상충 판단 및 최종 선택 기록

앞선 Task들 사이에서 충돌하거나 애매할 수 있는 항목들을 다시 모아
최종 선택을 기록하라.

최소 아래 쟁점을 포함하라.
	•	Document 생성 시 Version 없는 상태 허용 여부
	•	current_version_id optional 여부
	•	title을 Document와 Version 모두에 둘지
	•	Version을 완전 불변으로 볼지
	•	Node path/depth를 코어에 포함할지
	•	DocumentType을 enum 수준으로 시작할지
	•	metadata schema 연결을 초기부터 강제할지
	•	상태를 Document 중심으로 둘지
	•	deleted를 상태로 둘지 정책으로 둘지

각 항목에 대해:
	•	쟁점
	•	최종 선택
	•	이유
를 기록하라.

⸻

Step 9. 후속 Phase 입력 기준 정리

문서 마지막에는 반드시
다음 Phase에서 이 문서를 어떻게 사용할지 정리하라.

최소 아래를 포함한다.

Phase 2(저장 구조/DB 설계) 입력값
	•	테이블/컬렉션 분해 기준
	•	참조 관계 설계 기준
	•	불변조건 반영 포인트

Phase 3(API 설계) 입력값
	•	CRUD 단위 기준
	•	버전 생성 규칙
	•	상태 전이 API 제약 포인트
	•	metadata 입력/검증 기본 원칙

Phase 4(UI/편집/렌더링) 입력값
	•	Document/Version/Node 분리 인식
	•	Node 트리 편집 기준
	•	상태/타입/metadata UI 노출 기준

⸻

Step 10. 최종 기준 문서 작성

최종 결과는 아래 목차를 따르는 Markdown 문서로 작성하라.

권장 문서명:
phase1_final_integrated_domain_model_baseline.md

권장 목차:
	1.	작업 목적
	2.	Phase 1 결과 통합 개요
	3.	코어 모델 구성요소 최종 확정
	4.	최종 책임 분리 기준안
	5.	최종 관계 구조 요약
	6.	최소 권장 코어 모델 기준안
	7.	핵심 불변조건 요약
	8.	MVP 범위와 후속 확장 범위
	9.	상충 판단과 최종 선택
	10.	후속 Phase 입력 기준
	11.	최종 결론

⸻

6. 출력 형식 요구사항

반드시 Markdown(.md) 문서로 작성

권장 문서명:
phase1_final_integrated_domain_model_baseline.md

표를 적극 활용

최소 아래 표를 포함하는 것이 좋다.
	•	코어 구성요소 요약표
	•	책임 분리표
	•	MVP vs 후속 확장 비교표
	•	상충 판단 / 최종 선택 표

문장형 기준안으로 작성

좋은 예:
	•	“Document는 문서의 지속적 정체성을 표현한다.”
	•	“Version은 특정 시점의 문서 상태를 재현 가능하게 보존한다.”
	•	“Node는 Version에 종속된 문서 구조 단위다.”
	•	“초기 상태 모델은 Document 중심으로 적용한다.”

좋지 않은 예:
	•	“Document = 문서”
	•	“Version = 버전”
	•	“Node = 내용”

⸻

7. 산출물

필수 산출물
	1.	phase1_final_integrated_domain_model_baseline.md

문서 내 반드시 포함되어야 할 내용
	•	Phase 1 통합 결과 요약
	•	코어 모델 최종 확정안
	•	책임 분리 기준안
	•	관계 구조 요약
	•	MVP / 후속 확장 분리
	•	후속 Phase 입력 기준

⸻

8. 완료 기준

아래를 모두 만족하면 완료로 본다.
	•	Task 1-1 ~ 1-9 결과가 하나의 기준안으로 통합되어 있다.
	•	코어 엔티티와 개념의 경계가 최종 확정되어 있다.
	•	MVP 범위와 후속 확장 범위가 명확히 구분되어 있다.
	•	이후 DB / API / UI 설계가 이 문서를 기준으로 진행 가능하다.
	•	상충 판단이 정리되어 후속 혼선이 줄어든다.

⸻

9. 작업 시 주의사항
	•	새로운 범위를 무리하게 추가하지 마라.
	•	앞선 Task 결과를 무시하고 다시 처음부터 설계하지 마라.
	•	구현 세부사항으로 내려가지 마라.
	•	과도한 미래 기능을 코어 기준안에 섞지 마라.
	•	MVP 범위를 흐리지 마라.
	•	개념 문서와 구현 문서를 혼동하지 마라.

⸻

10. Claude Code에게 전달할 한 줄 지시

“Mimir 프로젝트의 Phase 1 - Task 1-10으로서, Task 1-1부터 1-9까지의 결과를 통합하여 범용 문서 플랫폼의 최종 도메인 모델 기준안을 Markdown 문서로 작성하라. 코어 모델 구성요소, 책임 분리, 관계 구조, 최소 권장 기준안, 핵심 불변조건, MVP 범위와 후속 확장 범위, 후속 Phase 입력 기준까지 포함하되 구현은 하지 마라.”