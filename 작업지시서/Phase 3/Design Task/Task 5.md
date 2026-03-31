Task 3-5. 공통 요청/응답 포맷 표준화

Claude Code 작업 지시서

1. 작업명

Phase 3 - Task 3-5. 공통 요청/응답 포맷 표준화

⸻

2. 작업 목적

플랫폼 API 전반에서 일관되게 사용할 공통 요청/응답 포맷 표준을 설계 문서로 정리하라.

이번 작업의 목적은 단순히 JSON 예시 몇 개를 만드는 것이 아니다.
핵심은 다음이다.
	•	단건 조회, 목록 조회, 생성/수정/삭제, 비동기 작업 응답, 오류 응답에 대해 일관된 계약 구조를 정의
	•	User UI, Admin UI, 외부 시스템, AI/RAG 소비자, 내부 서비스가 모두 예측 가능한 응답 구조를 사용할 수 있게 함
	•	이후 pagination/filter/sort, idempotency, tracing, audit, event/webhook, AI tool integration과 자연스럽게 연결될 수 있는 공통 응답 기반을 마련
	•	구현자별 취향 차이로 응답 구조가 흩어지는 것을 방지

즉, 이 문서는 플랫폼 API 계약의 표면 형식(surface contract) 을 정하는 기준 문서여야 한다.

⸻

3. 반드시 반영할 배경

이 플랫폼은 다음 성격을 가진다.
	•	범용 문서 플랫폼
	•	API-first 구조
	•	리소스 중심 REST API 지향
	•	User/Admin UI 분리
	•	외부 시스템 연동 필요
	•	AI/RAG/Agent도 공식 API 소비자
	•	조직/권한/감사 추적 필요
	•	이후 pagination/filter/sort, idempotency, jobs, events, webhooks, search/retrieval로 확장 예정

이전 Task에서 이미 다음이 정해졌다고 가정하라.
	•	플랫폼 API는 계약 계층이다.
	•	REST 리소스 구조와 명명 규칙이 존재한다.
	•	인증/인가 보안 문맥이 API 계층에 반영된다.
	•	버전 관리 전략이 존재한다.
	•	장기 일관성과 확장성이 우선이다.

따라서 이번 작업은 “응답 JSON 보기 좋게 정리” 수준이 아니라,
플랫폼 계약 일관성의 핵심 기반 설계여야 한다.

⸻

4. 이번 작업의 핵심 질문

이번 문서에서는 최소한 아래 질문들에 답할 수 있어야 한다.
	1.	모든 성공 응답을 envelope로 감쌀 것인가, 아니면 순수 리소스만 반환할 것인가?
	2.	단건 응답과 목록 응답의 구조를 어떻게 통일할 것인가?
	3.	metadata는 어디에 위치해야 하는가?
	4.	오류 응답은 어떤 구조를 가져야 하는가?
	5.	validation error, authorization error, conflict error, async job accepted 응답은 어떻게 구분되는가?
	6.	request_id, trace_id, version, timestamp 같은 운영 메타데이터를 응답에 어떻게 반영할 것인가?
	7.	클라이언트가 기계적으로 처리하기 쉬운 구조와 사람이 읽기 쉬운 구조를 어떻게 균형 잡을 것인가?
	8.	AI tool이나 외부 시스템이 파싱하기 쉬운 응답 규약을 어떻게 마련할 것인가?
	9.	지나친 envelope 복잡성을 피하면서도 확장 가능한 구조를 어떻게 만들 것인가?
	10.	후속 Task들에서 pagination, idempotency, error model을 어떻게 자연스럽게 얹을 수 있는가?

⸻

5. 상세 작업 요구사항

5-1. 문서 작성

아래 산출물을 작성하라.

필수 산출물
	•	Task5_common_response_format.md

권장 부속 산출물
	•	Task5_response_examples.json.md 또는 문서 내 예시 섹션

문서에 반드시 포함할 섹션
	1.	문서 목적
	2.	공통 요청/응답 포맷이 필요한 이유
	3.	상위 설계 원칙
	4.	성공 응답 구조 원칙
	5.	단건 응답 규약
	6.	목록 응답 규약
	7.	메타데이터 구조 원칙
	8.	오류 응답 구조 원칙
	9.	Validation / Auth / Conflict / Not Found / Async 응답 구분
	10.	운영 추적 메타데이터 규칙
	11.	AI/외부 시스템 친화성 관점
	12.	예시 응답 모음
	13.	후속 Task에 전달할 설계 기준
	14.	결론

⸻

5-2. 공통 요청/응답 포맷이 필요한 이유 정리

문서 초반에서 다음을 명확히 설명하라.
	•	플랫폼 API는 다수 소비자가 공유하므로 응답 구조의 일관성이 중요하다.
	•	같은 종류의 응답은 어느 리소스에서든 유사한 구조를 가져야 한다.
	•	응답 구조가 엔드포인트마다 달라지면 UI와 외부 연동, SDK, 테스트, AI tool integration 비용이 급격히 증가한다.
	•	request/response contract는 장기 운영의 안정성에 직접 연결된다.
	•	오류 응답도 성공 응답만큼 표준화되어야 한다.

즉, 공통 포맷은 단지 보기 좋은 JSON이 아니라
계약 비용을 낮추는 핵심 표준이라는 점이 드러나야 한다.

⸻

5-3. 상위 설계 원칙 정의

최소한 아래 원칙을 포함하라.
	•	Consistent shape across endpoints
	•	Predictable parsing
	•	Machine-readable first, human-readable also
	•	Minimal but extensible envelope
	•	Separation of resource data and metadata
	•	Explicit error contracts
	•	Traceable responses
	•	Version-compatible structure
	•	AI/tool-friendly representation

각 원칙마다 아래를 설명하라.
	•	의미
	•	왜 필요한지
	•	실제 응답 구조 설계에 어떤 영향을 주는지

예:
	•	Predictable parsing → 성공/실패/목록/비동기 응답의 상위 키 구조를 일관되게 유지
	•	Minimal but extensible envelope → 초기부터 과도한 중첩은 피하되, pagination/tracing 등을 넣을 수 있는 공간은 확보
	•	Explicit error contracts → 단순 문자열 대신 code/message/details 구조 사용

⸻

5-4. 성공 응답 envelope 방향 결정

중요한 설계 선택이다.
다음을 비교하고 권장 방향을 제시하라.

후보 A
리소스 자체를 바로 반환
예: { "id": "...", "title": "..." }

후보 B
상위 envelope 아래에 data 배치
예: { "data": { ... }, "meta": { ... } }

후보 C
상황별로 다르게 사용
예: 단건은 리소스 직접, 목록은 wrapper

각 후보에 대해:
	•	장점
	•	단점
	•	파서 일관성
	•	확장성
	•	외부 시스템/AI 소비 관점
	•	이 플랫폼에서의 적합성

그리고 최종 권장안을 제시하라.

중요:
초기 구현 단순성과 장기 확장성을 함께 고려하라.
플랫폼 규모상, metadata/tracing/pagination/error consistency를 다루기 쉬운 구조가 더 적합한지 검토해야 한다.

⸻

5-5. 단건 응답 규약 정의

다음을 정리하라.
	•	단건 조회 성공 응답 구조
	•	생성 성공 응답 구조
	•	수정 성공 응답 구조
	•	삭제 성공 응답 구조
	•	상태 전이(action endpoint) 성공 응답 구조
	•	비동기 작업 시작 응답과의 차이

반드시 포함할 것:
	•	data 영역 사용 여부
	•	meta 영역 사용 여부
	•	생성 후 반환 리소스 범위
	•	삭제 응답에서 body를 둘지 여부
	•	action 응답도 리소스 계약의 일부로 일관성 있게 다뤄야 한다는 점

⸻

5-6. 목록 응답 규약 정의

다음을 문서화하라.
	•	목록 응답의 상위 구조
	•	items 배열 위치
	•	total count 제공 여부
	•	pagination metadata 위치
	•	filter/sort 반영 정보를 메타에 넣을지 여부
	•	빈 목록일 때의 응답 원칙
	•	authorization-aware filtering이 적용되어도 구조는 같아야 한다는 점

예를 들어 아래 질문에 답할 수 있어야 한다.
	•	data 아래에 items를 둘지
	•	meta.pagination 같은 구조를 둘지
	•	links를 둘지
	•	total은 항상 주는지, 조건부로 주는지
	•	cursor 기반과 offset 기반의 공통 표현이 가능한지

세부 문법은 Task 3-6에서 정하지만,
이번 문서에서는 응답 틀을 정하라.

⸻

5-7. 메타데이터 구조 원칙

응답에 포함될 수 있는 metadata를 정리하라.

검토 대상 예시:
	•	request_id
	•	trace_id
	•	api_version
	•	timestamp
	•	pagination
	•	filter summary
	•	sort summary
	•	idempotency replay indicator 여부
	•	deprecation info 여부
	•	warnings
	•	partial result 여부
	•	source context 여부

다음을 반드시 정리하라.
	•	어떤 메타데이터는 전역 meta에 두고,
	•	어떤 것은 특정 응답 유형에서만 등장하는지
	•	resource data와 operational metadata를 분리해야 하는 이유
	•	metadata가 지나치게 비대해지지 않도록 제한하는 원칙

⸻

5-8. 오류 응답 구조 원칙

이 부분은 매우 중요하다.
오류 응답에 대해 공통 구조를 정의하라.

최소한 검토할 필드:
	•	error.code
	•	error.message
	•	error.type 또는 category
	•	error.details
	•	error.target / field
	•	request_id
	•	trace_id
	•	timestamp
	•	retryable 여부
	•	docs link 여부
	•	multiple errors 표현 방식

문서에는 다음을 반영하라.
	•	오류 응답은 문자열 하나가 아니라 구조화된 객체여야 한다.
	•	사람 친화 메시지와 기계 판별 코드를 함께 가져야 한다.
	•	내부 시스템 정보는 과도하게 노출하지 않는다.
	•	동일 계열 오류는 어느 엔드포인트에서든 유사한 구조를 유지한다.

⸻

5-9. Validation / Auth / Conflict / Not Found / Async 응답 구분

반드시 아래 유형을 각각 구분해서 정리하라.
	•	validation error
	•	authentication failure
	•	authorization failure
	•	not found
	•	conflict
	•	rate limit 여부 검토 가능
	•	async accepted
	•	idempotent replay response 여부 검토 가능

각 항목에 대해:
	•	어떤 HTTP status 범주와 연결되는지
	•	응답 구조에서 어떤 필드가 중요해지는지
	•	사용자 메시지와 기계 처리용 정보가 어떻게 구분되는지
	•	후속 Task에서 더 구체화할 영역이 무엇인지

중요:
Task 3-10에서 오류 모델을 더 깊게 다루겠지만,
이번 문서에서도 응답 shape 수준의 기준은 잡아야 한다.

⸻

5-10. 운영 추적 메타데이터 규칙

다음을 정리하라.
	•	request_id / trace_id 노출 여부
	•	어느 응답에 기본 포함할지
	•	success와 error 모두에서 일관되게 다룰지
	•	timestamp 포함 여부
	•	correlation에 필요한 최소 메타데이터
	•	디버깅에 유용하지만 민감할 수 있는 정보는 어디까지 노출할지

이 문서는 observability 세부 정책 문서가 아니므로,
응답 계약에서 어떤 운영 메타데이터를 위한 자리를 확보할지에 집중하라.

⸻

5-11. AI / 외부 시스템 친화성 관점

이 플랫폼은 AI와 외부 시스템도 공식 소비자다.
다음을 반드시 다뤄라.
	•	응답 구조가 너무 인간 중심 서술형이면 안 됨
	•	상위 키가 예측 가능해야 함
	•	리소스 식별자 위치가 일관되어야 함
	•	오류 코드가 기계적으로 분기 가능해야 함
	•	metadata와 resource payload가 섞이지 않아야 함
	•	AI tool이 citation/lookup/follow-up action을 수행하기 쉬운 구조가 바람직함

즉, UI 렌더링 편의만이 아니라
자동화 소비자 친화성도 응답 포맷 설계 기준에 포함하라.

⸻

5-12. 요청 포맷 관점도 최소 반영

이번 Task는 응답 중심이지만, 요청 포맷에 대해서도 최소 기준을 정리하라.

예:
	•	JSON body 기반 요청 원칙
	•	top-level body shape 일관성 여부
	•	bulk/action/search 요청에서 wrapper를 둘지 여부
	•	partial update 시 body shape 방향
	•	metadata성 입력과 resource mutation 입력 구분 필요성

중요:
요청 포맷은 깊게 들어가지 말고,
응답 포맷과 일관된 계약 철학만 정리하라.

⸻

5-13. 예시 응답 모음 작성

문서에 최소한 아래 예시를 포함하라.

성공 응답 예시
	•	단건 조회
	•	생성 성공
	•	목록 조회
	•	빈 목록
	•	action 성공
	•	async accepted

오류 응답 예시
	•	validation error
	•	auth failure
	•	permission denied
	•	not found
	•	conflict

예시는 JSON 형태로 보여주되,
구조의 의미를 짧게 설명하라.

중요:
예시는 후속 구현자가 바로 참조 가능한 수준이어야 한다.

⸻

5-14. 후속 Task에 전달할 설계 기준

문서 마지막에 다음 연결 포인트를 정리하라.

예:
	•	Task 3-6에서는 pagination/filter/sort metadata를 이 응답 envelope에 얹어야 함
	•	Task 3-7에서는 idempotency replay 정보가 meta나 header와 어떻게 연결될지 검토해야 함
	•	Task 3-8에서는 async job/webhook/event 응답 형식도 이 공통 계약을 가능한 한 따르도록 해야 함
	•	Task 3-9에서는 AI/RAG retrieval/search 결과 형식이 공통 envelope를 활용할지 정리해야 함
	•	Task 3-10에서는 error code taxonomy와 observability metadata를 더 구체화해야 함
	•	구현 Phase에서는 serializer/exception handler/response helper 설계에 직접 반영해야 함

⸻

6. 산출물

아래 파일을 작성하라.

필수 산출물
	•	Task5_common_response_format.md

권장 부속 산출물
	•	문서 내부 JSON 예시 섹션 또는 Task5_response_examples.json.md

문서 성격
	•	설계 기준 문서
	•	구현 코드 금지
	•	프레임워크 응답 객체 예시 최소화
	•	장기 계약 구조 중심

⸻

7. 제외 범위

이번 작업에서는 아래는 하지 마라.
	•	실제 serializer/helper 코드 작성
	•	OpenAPI schema 전체 작성
	•	HTTP status code taxonomy 전체 확정
	•	pagination 세부 문법 확정
	•	filtering/sorting query syntax 확정
	•	idempotency header 명세 확정
	•	error code catalog 전체 작성
	•	webhook payload schema 상세 설계
	•	search/retrieval 응답 전용 스키마 확정
	•	국제화(i18n) 메시지 체계 상세 설계

즉, 이번 작업은 공통 요청/응답 포맷의 상위 표준 수립에 집중하라.

⸻

8. 작성 스타일 가이드

문서는 아래 스타일을 따르라.
	•	선언적이고 기준 중심으로 작성
	•	envelope 채택 여부는 장단점을 비교한 뒤 명확히 권장
	•	예시는 충분히 실용적으로 작성
	•	“권장 / 허용 / 비권장 / 금지 / 후속 Task에서 구체화”를 구분
	•	사람이 읽기 쉬우면서도 기계 처리 가능한 구조라는 균형을 유지
	•	UI 전용 응답 형태가 아니라 플랫폼 공통 계약 형태로 서술

⸻

9. 검토 체크리스트

작업 완료 전에 아래를 스스로 점검하라.
	•	성공/오류 응답 구조가 모두 정리되었는가?
	•	단건/목록/비동기/오류 응답 차이가 설명되었는가?
	•	metadata 구조 원칙이 있는가?
	•	envelope 전략이 명확히 결정되었는가?
	•	외부 시스템과 AI 소비자 친화성이 반영되었는가?
	•	운영 추적 메타데이터 자리가 확보되었는가?
	•	후속 Task 3-6, 3-7, 3-8, 3-10과 자연스럽게 연결되는가?

⸻

10. Claude Code에 대한 최종 지시

위 요구사항을 반영하여 Task5_common_response_format.md 문서를 작성하라.

문서는 범용 문서 플랫폼 API에서 공통적으로 사용할
성공/목록/오류/비동기 응답 구조와 메타데이터 배치 원칙을 정의하는 기준 문서여야 한다.

가능하면 실용적인 JSON 예시를 충분히 포함하라.
다만 예시가 문서의 원칙 설명을 대체해서는 안 된다.

문서 작성 후에는 아래 형식의 자체 점검 요약도 함께 정리하라.

자체 점검 요약
	•	권장 응답 envelope 구조
	•	단건/목록/오류/비동기 응답 핵심 shape 요약
	•	공통 metadata 핵심 필드 요약
	•	AI/외부 시스템 친화성 관점 요약
	•	후속 Task 연결 포인트
	•	의도적으로 후속 Task로 남긴 미결정 사항