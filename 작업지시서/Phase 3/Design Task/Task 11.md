Task 3-11. 대표 API 시나리오 설계

Claude Code 작업 지시서

1. 작업명

Phase 3 - Task 3-11. 대표 API 시나리오 설계

⸻

2. 작업 목적

지금까지 Phase 3에서 정리한 API 설계 원칙들이 실제 흐름에서 일관되게 작동하는지 검증하기 위해, 대표 API 시나리오 세트를 설계 문서로 정리하라.

이번 작업의 목적은 테스트 코드를 작성하는 것이 아니다.
핵심은 다음이다.
	•	앞서 정의한 API 아키텍처 원칙, REST 리소스 구조, 인증/인가, 버전 관리, 공통 응답 포맷, pagination/filter/sort, idempotency, async/event/webhook, AI/RAG 인터페이스, 오류 모델이 실제 사용 흐름에서 서로 충돌하지 않는지 검증
	•	구현 전에 “이 설계로 실제 사용자가 어떤 흐름을 수행할 수 있는가”를 구체적으로 점검
	•	User UI, Admin UI, 외부 시스템, AI agent, 운영/통합 시나리오를 균형 있게 포함
	•	이후 구현 Phase에서 API 우선순위를 정하고 통합 테스트/시퀀스 설계로 넘길 수 있는 기준선 마련

즉, 이 문서는 API 설계 검증용 대표 시나리오 문서여야 한다.

⸻

3. 반드시 반영할 배경

이 플랫폼은 다음 성격을 가진다.
	•	범용 문서 플랫폼
	•	문서 / 버전 / 노드 중심 도메인 구조
	•	사용자/조직/역할/ACL 존재
	•	감사 추적 필요
	•	REST 중심 API 구조
	•	외부 시스템 연동 필요
	•	AI/RAG/Agent도 공식 API 소비자
	•	async operation / event / webhook 확장 구조 존재
	•	공통 응답 포맷 및 오류 모델 존재
	•	idempotency 및 retry safety 고려
	•	검색/retrieval/citation을 포함한 AI-ready 인터페이스 지향

이전 Task에서 이미 다음이 정해졌다고 가정하라.
	•	Task 3-1: 플랫폼 API 아키텍처 원칙
	•	Task 3-2: 핵심 리소스 기반 REST 구조
	•	Task 3-3: 인증/인가 구조의 API 계층 반영
	•	Task 3-4: API 버전 관리 전략
	•	Task 3-5: 공통 요청/응답 포맷
	•	Task 3-6: pagination/filter/sort 규약
	•	Task 3-7: idempotency 및 retry safety
	•	Task 3-8: async/event/webhook 확장 모델
	•	Task 3-9: AI/RAG 인터페이스 방향
	•	Task 3-10: 오류 모델 및 observability 규약

따라서 이번 작업은 새로운 규칙을 만드는 단계라기보다,
기존 규칙들이 실제 사용 흐름에서 통합적으로 작동하는지 검증하는 단계여야 한다.

⸻

4. 이번 작업의 핵심 질문

이번 문서에서는 최소한 아래 질문들에 답할 수 있어야 한다.
	1.	지금까지 정의한 API 규약들이 실제 사용 흐름으로 연결되는가?
	2.	문서 생성, 버전 생성, 노드 조회 같은 기본 시나리오에서 리소스 구조와 응답 규약이 자연스러운가?
	3.	권한 부족, 잘못된 요청, 상태 충돌, idempotency 재시도 같은 예외 흐름이 일관되게 처리되는가?
	4.	async operation, event, webhook이 실제 흐름에서 어떤 식으로 연결되는가?
	5.	AI/RAG 사용 흐름에서 검색 → source fetch → citation 연결이 자연스러운가?
	6.	User/Admin/External/AI 소비자의 사용 패턴 차이가 API 구조 안에서 설명 가능한가?
	7.	구현 우선순위를 정할 때 어떤 시나리오가 MVP core인지 구분 가능한가?
	8.	운영/감사/관측성 정보가 실제 시나리오에서 추적 가능한가?
	9.	각 시나리오에서 어떤 엔드포인트군이 먼저 필요해지는가?
	10.	설계상 충돌, 애매함, 미결정 항목이 어떤 부분에서 드러나는가?

⸻

5. 상세 작업 요구사항

5-1. 문서 작성

아래 산출물을 작성하라.

필수 산출물
	•	Task11_api_scenarios.md

문서에 반드시 포함할 섹션
	1.	문서 목적
	2.	대표 시나리오 설계가 필요한 이유
	3.	시나리오 작성 원칙
	4.	시나리오 분류 체계
	5.	핵심 사용자 시나리오
	6.	권한/보안 시나리오
	7.	idempotency / conflict / 오류 처리 시나리오
	8.	async operation / event / webhook 시나리오
	9.	AI / RAG 사용 시나리오
	10.	운영/관측성/감사 추적 시나리오
	11.	시나리오별 공통 검증 포인트
	12.	시나리오별 미결정 항목 및 후속 설계 포인트
	13.	구현 우선순위 제안
	14.	결론

⸻

5-2. 대표 시나리오 설계가 필요한 이유 정리

문서 초반에서 다음을 명확히 설명하라.
	•	개별 규칙 문서만으로는 실제 사용 흐름에서의 일관성을 검증하기 어렵다.
	•	API 설계는 엔드포인트 단위가 아니라 사용자/시스템 작업 흐름 단위로 검토해야 한다.
	•	시나리오 기반 검증을 통해 누락된 리소스, 애매한 상태 전이, 과도한 복잡성, 미비한 오류 모델을 조기에 발견할 수 있다.
	•	이후 구현 단계에서 무엇부터 만들지 우선순위를 정하는 데 도움이 된다.

즉, 대표 시나리오는 단순 예시가 아니라
설계 검증과 구현 인계 사이를 잇는 실전형 문서라는 점이 드러나야 한다.

⸻

5-3. 시나리오 작성 원칙 정의

문서에는 최소한 아래 원칙을 포함하라.
	•	Scenario as contract validation, not feature fiction
	•	Happy path + failure path 모두 포함
	•	Consumer-aware scenario design
	•	Resource, security, response, error, observability를 함께 검증
	•	Minimal but representative coverage
	•	Cross-task consistency validation
	•	Implementation-oriented abstraction

각 원칙마다 아래를 설명하라.
	•	의미
	•	왜 필요한지
	•	시나리오 작성에서 어떤 결정을 유도하는지

예:
	•	Happy path + failure path 모두 포함 → 성공 흐름만 보면 설계가 실제 운영에서 무너질 수 있음
	•	Consumer-aware scenario design → User UI, Admin UI, External, AI, Ops 관점을 섞어야 함
	•	Cross-task consistency validation → 한 시나리오에서 리소스 구조, authz, 응답, idempotency, 오류 모델을 함께 검증

⸻

5-4. 시나리오 분류 체계 정의

문서에서 시나리오를 최소한 아래 범주로 분류하라.
	•	Core document lifecycle scenarios
	•	Access control / security scenarios
	•	Retry / conflict / failure scenarios
	•	Async / event / webhook scenarios
	•	AI / RAG consumption scenarios
	•	Operations / observability / audit scenarios

각 분류에 대해:
	•	왜 필요한지
	•	어떤 소비자가 주로 등장하는지
	•	어떤 기존 Task 결과를 검증하는지

를 정리하라.

⸻

5-5. 핵심 사용자 시나리오 정의

최소한 아래 시나리오를 포함하라.

시나리오 A. 문서 생성 → 초기 버전 생성 → 구조 조회
포함해야 할 요소:
	•	문서 생성 요청
	•	버전/노드 관계
	•	단건 응답 구조
	•	생성 후 조회 흐름
	•	기본 보안 문맥
	•	canonical resource reference

시나리오 B. 문서 목록 조회 with filter / sort / pagination
포함해야 할 요소:
	•	목록 query
	•	pagination metadata
	•	filter/sort 규약
	•	권한 기반 가시성 반영
	•	빈 목록 처리 가능성

시나리오 C. 문서 수정 → 새 버전 생성 또는 상태 갱신 흐름
포함해야 할 요소:
	•	update vs versioned change 관점
	•	응답 구조
	•	상태 전이(action endpoint) 가능성
	•	observability metadata 연결

중요:
이 시나리오들은 Phase 3 API의 최소 핵심 흐름으로 다뤄라.

⸻

5-6. 권한 / 보안 시나리오 정의

최소한 아래 시나리오를 포함하라.

시나리오 D. 권한 없는 사용자의 문서 조회 시도
포함 요소:
	•	authentication vs authorization 구분
	•	401/403/404 계열 판단 방향
	•	error shape
	•	과도한 정보 노출 방지

시나리오 E. 다른 조직 문서 접근 시도
포함 요소:
	•	tenant scope violation
	•	조직/스코프 enforcement
	•	감사 추적 필요성

시나리오 F. 관리자 전용 리소스 접근
포함 요소:
	•	관리자 API 또는 관리자 권한 시나리오
	•	일반 사용자와 다른 결과
	•	exposure boundary 검토

중요:
보안 시나리오는 UI 기준이 아니라
API 레벨 enforcement 기준으로 설명하라.

⸻

5-7. Idempotency / conflict / 오류 처리 시나리오 정의

최소한 아래 시나리오를 포함하라.

시나리오 G. 문서 생성 요청 타임아웃 후 동일 Idempotency-Key로 재시도
포함 요소:
	•	최초 요청 성공 여부 불명
	•	같은 의도 재시도
	•	replay 응답 가능성
	•	observability/audit 연결

시나리오 H. 같은 Idempotency-Key로 다른 payload 전송
포함 요소:
	•	key mismatch
	•	conflict/error code
	•	안전한 실패 처리

시나리오 I. 상태 전이 충돌 또는 invalid transition
예:
	•	이미 archived 상태인 문서에 archive 재요청
	•	publish 불가능 상태에서 publish 요청

포함 요소:
	•	action endpoint 의미
	•	invalid_state_transition / conflict 구분
	•	재시도 가능성 판단

시나리오 J. 잘못된 filter / invalid cursor / unsupported sort 요청
포함 요소:
	•	query contract violation
	•	structured error
	•	machine-readable code

⸻

5-8. Async operation / event / webhook 시나리오 정의

최소한 아래 시나리오를 포함하라.

시나리오 K. export 요청 → async operation 생성 → 상태 polling
포함 요소:
	•	sync vs async 경계
	•	operation resource
	•	상태 전이
	•	완료 후 결과 참조

시나리오 L. document.published 이벤트 발생 → webhook delivery
포함 요소:
	•	도메인 이벤트와 웹훅 전달 구분
	•	subscription 개념
	•	delivery 상태
	•	재시도 가능성

시나리오 M. webhook delivery 실패 → delivery 조회 → 재전송
포함 요소:
	•	failed delivery observability
	•	resend action 가능성
	•	운영 API 관점

시나리오 N. reindex / ingestion 시작 → 동일 idempotency key 재요청
포함 요소:
	•	async operation과 idempotency 결합
	•	duplicate job 방지
	•	operation reference 재사용 가능성

⸻

5-9. AI / RAG 사용 시나리오 정의

최소한 아래 시나리오를 포함하라.

시나리오 O. 대화형 assistant가 사용자 권한 범위에서 검색 → 노드 조회 → citation 생성
포함 요소:
	•	retrieval query
	•	source resource lookup
	•	citation-ready identifier
	•	permission-preserving retrieval

시나리오 P. 외부 RAG 시스템이 문서 구조와 특정 버전을 조회하여 인덱싱
포함 요소:
	•	version-aware access
	•	structured content access
	•	metadata 활용
	•	service identity 또는 external integration 관점

시나리오 Q. 문서 변경 → indexing operation 시작 → indexing 완료 후 retrieval 가능
포함 요소:
	•	async operation
	•	이벤트 또는 상태 조회
	•	indexing visibility
	•	AI-ready lifecycle 연결

시나리오 R. AI tool이 citation identifier를 따라 원문 노드 재조회
포함 요소:
	•	stable reference
	•	follow-up fetch
	•	응답 shape의 tool-friendliness

⸻

5-10. 운영 / 관측성 / 감사 추적 시나리오 정의

최소한 아래 시나리오를 포함하라.

시나리오 S. 실패한 요청을 request_id로 추적
포함 요소:
	•	error response metadata
	•	운영 로그 correlation
	•	감사 로그와의 관계

시나리오 T. 누가 문서를 publish 했는지 추적
포함 요소:
	•	actor context
	•	domain event
	•	audit log
	•	operation/event/request 연결

시나리오 U. 관리자 또는 운영자가 실패한 webhook delivery와 관련 이벤트를 조사
포함 요소:
	•	webhook delivery id
	•	event id
	•	request trace
	•	운영 리소스 조회 흐름

중요:
이 시나리오들은 “API가 성공적으로 일한다”보다
운영자가 문제를 이해할 수 있는가를 검증하는 관점으로 작성하라.

⸻

5-11. 각 시나리오의 서술 형식 표준화

모든 시나리오는 가능한 한 같은 형식으로 정리하라.

권장 형식:
	1.	시나리오 이름
	2.	주요 소비자
	3.	목적
	4.	전제 조건
	5.	주요 리소스
	6.	단계별 흐름
	7.	핵심 API 규약 검증 포인트
	8.	예상 성공 응답/오류 응답 포인트
	9.	관측성/감사 추적 포인트
	10.	남아 있는 미결정 사항

중요:
시나리오가 산문식 설명으로 흩어지지 않게,
구조화된 검토 문서처럼 작성하라.

⸻

5-12. 시나리오별 공통 검증 포인트 정의

문서 어딘가에 아래와 같은 공통 체크 항목을 정리하라.
	•	리소스 구조가 자연스러운가?
	•	actor/tenant/security context가 반영되는가?
	•	요청/응답 포맷이 일관적인가?
	•	pagination/filter/sort가 적절한가?
	•	idempotency 의미가 분명한가?
	•	error model이 충분히 구조화되어 있는가?
	•	request_id/trace_id/operation_id/event_id 등 추적 포인트가 있는가?
	•	AI 소비자도 같은 계약을 재사용할 수 있는가?

이 체크 포인트는 후속 검토와 구현 인계 시 바로 쓸 수 있어야 한다.

⸻

5-13. 시나리오별 미결정 항목 및 후속 설계 포인트 정리

각 시나리오마다 또는 문서 말미에서 다음을 정리하라.
	•	어떤 부분은 이미 Phase 3 기준으로 충분히 결정되었는지
	•	어떤 부분은 Phase 4 이후 실제 구현 설계에서 더 정해야 하는지
	•	어떤 시나리오는 MVP에서 축소 가능하고, 어떤 것은 초기부터 반드시 반영해야 하는지

예:
	•	citation identifier 구체 포맷
	•	operation payload schema
	•	webhook signature 상세
	•	indexing status resource 구체 구조
	•	concurrency control 상세 등

⸻

5-14. 구현 우선순위 제안

문서 마지막에 시나리오 기준 구현 우선순위를 제안하라.

예를 들면 다음처럼 묶을 수 있다.

Priority 1 — MVP 핵심
	•	문서 생성/조회/목록
	•	기본 auth/authz
	•	공통 응답/오류 구조
	•	기본 filter/sort/pagination

Priority 2 — 안정성/운영성
	•	idempotency
	•	observability
	•	감사 추적 연결
	•	conflict/invalid transition 처리

Priority 3 — 확장성
	•	async operation
	•	event/webhook
	•	external integration support

Priority 4 — AI-ready 확장
	•	구조 조회
	•	retrieval/citation
	•	indexing status
	•	AI workflow support

중요:
이 우선순위는 구현 일정이 아니라
Phase 3 산출물을 다음 Phase 구현에 어떻게 연결할지 보여주는 인계 프레임이어야 한다.

⸻

6. 산출물

아래 파일을 작성하라.

필수 산출물
	•	Task11_api_scenarios.md

문서 성격
	•	설계 검증 문서
	•	구현 코드 금지
	•	테스트 코드 금지
	•	OpenAPI 전체 작성 금지
	•	시퀀스/흐름 중심 설계 검토 문서

⸻

7. 제외 범위

이번 작업에서는 아래는 하지 마라.
	•	실제 테스트 케이스 코드 작성
	•	Postman collection 작성
	•	시퀀스 다이어그램 이미지 생성
	•	OpenAPI 예제 전체 세트 작성
	•	성능 테스트 시나리오 상세화
	•	SLA/SLO 운영 문서 작성
	•	세부 DB/큐/인덱스 구현 설계
	•	UI 화면 설계

즉, 이번 작업은 대표 API 사용 흐름을 통한 설계 검증에 집중하라.

⸻

8. 작성 스타일 가이드

문서는 아래 스타일을 따르라.
	•	규약 문서를 실제 흐름으로 연결하는 검증 문서처럼 작성
	•	happy path와 failure path를 균형 있게 포함
	•	User / Admin / External / AI / Ops 관점을 빠짐없이 고려
	•	각 시나리오를 구조화된 템플릿으로 작성
	•	“권장 / 검증 필요 / 미결정 / 후속 구현에서 구체화”를 구분
	•	예시는 실전적이어야 하지만 구현 상세로 너무 내려가지 않게 할 것

⸻

9. 검토 체크리스트

작업 완료 전에 아래를 스스로 점검하라.
	•	핵심 사용자 시나리오가 포함되었는가?
	•	권한/조직/보안 시나리오가 포함되었는가?
	•	idempotency/conflict/query 오류 시나리오가 포함되었는가?
	•	async/event/webhook 시나리오가 포함되었는가?
	•	AI/RAG 사용 시나리오가 포함되었는가?
	•	observability/audit 시나리오가 포함되었는가?
	•	각 시나리오가 구조화된 형식으로 작성되었는가?
	•	공통 검증 포인트와 미결정 항목이 정리되었는가?
	•	구현 우선순위 제안이 포함되었는가?

⸻

10. Claude Code에 대한 최종 지시

위 요구사항을 반영하여 Task11_api_scenarios.md 문서를 작성하라.

문서는 지금까지 정의한 Phase 3 API 설계 원칙들이 실제 사용자/시스템/AI/운영 흐름에서 일관되게 작동하는지 검증할 수 있게 하는
대표 API 시나리오 설계 및 검증 기준 문서여야 한다.

문서 작성 후에는 아래 형식의 자체 점검 요약도 함께 정리하라.

자체 점검 요약
	•	포함한 시나리오 범주 요약
	•	MVP 핵심 시나리오 요약
	•	보안/오류/idempotency 검증 시나리오 요약
	•	async/event/webhook 시나리오 요약
	•	AI/RAG 시나리오 요약
	•	observability/audit 시나리오 요약
	•	구현 우선순위 제안 요약
	•	후속 Task 연결 포인트
	•	의도적으로 후속 단계로 남긴 미결정 사항