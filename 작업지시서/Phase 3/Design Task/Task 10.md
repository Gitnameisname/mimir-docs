Task 3-10. API 오류 모델 및 운영 관측성 규약 설계

Claude Code 작업 지시서

1. 작업명

Phase 3 - Task 3-10. API 오류 모델 및 운영 관측성 규약 설계

⸻

2. 작업 목적

플랫폼 API 전반에서 일관되게 적용할 오류 모델(error model) 과 운영 관측성(observability) 규약을 설계 문서로 정리하라.

이번 작업의 목적은 단순히 HTTP 상태 코드를 나열하는 것이 아니다.
핵심은 다음이다.
	•	API 오류를 사람이 읽기 쉽고, 동시에 기계적으로 처리 가능한 형태로 표준화
	•	인증/인가 실패, validation 실패, conflict, not found, rate limit, async failure 등 다양한 실패 유형을 일관되게 분류
	•	request_id, trace_id, actor context, operation reference, webhook delivery reference 같은 운영 추적 정보를 API 계약 차원에서 반영
	•	UI, 외부 시스템, AI agent, 내부 운영 도구가 문제를 진단하고 재시도/분기 처리할 수 있도록 기준 마련
	•	감사 추적, 보안, idempotency, async operation, AI/RAG workflow와 연결되는 장애/오류 처리 철학 정립

즉, 이 문서는 플랫폼 API의 실패 계약과 운영 추적 기준 문서여야 한다.

⸻

3. 반드시 반영할 배경

이 플랫폼은 다음 성격을 가진다.
	•	범용 문서 플랫폼
	•	REST 중심 API 구조
	•	문서/버전/노드/권한/감사/검색/AI 연계/이벤트/웹훅/비동기 작업 존재
	•	User/Admin UI, 외부 시스템, AI/Agent, 내부 서비스가 모두 API 소비자
	•	권한 제어와 감사 추적이 중요
	•	idempotency 및 retry safety를 고려해야 함
	•	async operations / events / webhooks / retrieval/search까지 확장됨

이전 Task에서 이미 다음이 정해졌다고 가정하라.
	•	플랫폼 API는 계약 계층이다.
	•	리소스 구조와 명명 규칙이 있다.
	•	인증/인가 구조 및 request security context가 있다.
	•	버전 전략이 있다.
	•	공통 요청/응답 포맷이 있다.
	•	pagination/filter/sort 규약이 있다.
	•	idempotency 전략이 있다.
	•	async/event/webhook 확장 모델이 있다.
	•	AI/RAG 연계 인터페이스 방향이 있다.

따라서 이번 작업은 “예외 처리 구현 가이드”가 아니라,
실패 의미와 운영 가시성의 플랫폼 표준을 정의하는 작업이어야 한다.

⸻

4. 이번 작업의 핵심 질문

이번 문서에서는 최소한 아래 질문들에 답할 수 있어야 한다.
	1.	플랫폼 API에서 오류는 어떤 수준으로 구조화되어야 하는가?
	2.	HTTP status code와 business error code는 어떤 관계를 가져야 하는가?
	3.	validation, auth, permission, not found, conflict, rate limit, async failure, external integration failure를 어떻게 구분할 것인가?
	4.	사람이 읽는 메시지와 기계 처리용 코드는 어떻게 분리할 것인가?
	5.	보안상 숨겨야 할 정보와 운영상 필요한 정보의 경계를 어떻게 정할 것인가?
	6.	request_id / trace_id / correlation 정보는 응답에 어떻게 반영할 것인가?
	7.	async operation, webhook delivery, idempotency replay, AI retrieval failure 같은 확장 시나리오를 오류 모델에 어떻게 포함할 것인가?
	8.	어떤 실패는 재시도 가능하고, 어떤 실패는 즉시 수정이 필요한지 어떻게 표현할 것인가?
	9.	감사 로그, 운영 로그, API 오류 응답은 어떤 관계를 가져야 하는가?
	10.	구현 Phase에서 exception handler, middleware, audit pipeline, observability stack에 어떤 기준으로 연결되어야 하는가?

⸻

5. 상세 작업 요구사항

5-1. 문서 작성

아래 산출물을 작성하라.

필수 산출물
	•	Task10_api_error_and_observability.md

문서에 반드시 포함할 섹션
	1.	문서 목적
	2.	왜 오류 모델과 관측성 규약이 필요한가
	3.	상위 설계 원칙
	4.	오류 모델의 기본 구조
	5.	HTTP status code와 business error code 관계
	6.	오류 분류 체계
	7.	주요 오류 유형별 응답 원칙
	8.	운영 추적 메타데이터 규약
	9.	보안 노출 제한 원칙
	10.	async operation / webhook / idempotency / AI 연계 오류 관점
	11.	감사 추적 / 운영 로그 / API 오류 응답의 관계
	12.	예시 오류 응답 및 운영 메타데이터 예시
	13.	후속 Task 및 구현 Phase에 전달할 기준
	14.	결론

⸻

5-2. 왜 오류 모델과 관측성 규약이 필요한가

문서 초반에서 다음을 명확히 설명하라.
	•	성공 계약만 표준화되어서는 충분하지 않다.
	•	실제 운영에서는 실패 응답의 일관성이 성공 응답만큼 중요하다.
	•	UI는 사용자에게 적절한 메시지를 보여줘야 하고, 외부 시스템과 AI agent는 기계적으로 오류를 분기 처리해야 한다.
	•	장애 원인 분석, 보안 감사, 고객 지원, 운영 대응에는 request/trace 기준의 추적성이 필요하다.
	•	엔드포인트마다 제각각의 오류 구조를 쓰면 디버깅과 자동화가 어려워진다.

즉, 오류 모델과 관측성 규약은 부가 요소가 아니라
플랫폼 신뢰성과 운영 가능성을 결정하는 핵심 계약이라는 점이 드러나야 한다.

⸻

5-3. 상위 설계 원칙 정의

최소한 아래 원칙을 포함하라.
	•	Structured failures over ad-hoc messages
	•	HTTP semantics plus platform-specific codes
	•	Machine-readable and human-usable
	•	Consistent error shape across endpoints
	•	Traceable by default
	•	Secure disclosure
	•	Retry-aware signaling
	•	Async-aware failure modeling
	•	Domain-neutral but extensible taxonomy
	•	Observability without overexposure

각 원칙마다 다음을 설명하라.
	•	의미
	•	왜 필요한지
	•	실제 설계에서 어떤 결정을 유도하는지

예:
	•	Structured failures over ad-hoc messages → 문자열만 반환하지 않고 code/message/details 구조를 유지
	•	Retry-aware signaling → 재시도 가능 여부나 수정 필요 여부를 기계적으로 판단할 수 있어야 함
	•	Secure disclosure → 내부 스택/정책/민감 식별자는 외부 응답에 과도하게 노출하지 않음

⸻

5-4. 오류 모델의 기본 구조 정의

Task 3-5의 공통 응답 포맷과 연결하여, 오류 응답의 기본 shape를 정리하라.

최소한 다음 필드 후보를 검토하라.
	•	error.code
	•	error.message
	•	error.category 또는 error.type
	•	error.details
	•	error.target
	•	error.retryable
	•	error.docs_ref 또는 help_ref 여부
	•	meta.request_id
	•	meta.trace_id
	•	meta.timestamp
	•	meta.operation_id / meta.webhook_delivery_id / meta.idempotency_key 등 확장 메타데이터

반드시 포함할 내용:
	•	상위 error object 구조
	•	단일 오류와 다중 validation 오류 표현 방식
	•	사람이 읽는 메시지와 기계 판별용 코드 분리
	•	details는 선택적이되 구조화 가능해야 함

⸻

5-5. HTTP status code와 business error code 관계

이 부분은 중요하다.
다음을 정리하라.
	•	HTTP status는 transport/semantic layer의 1차 신호다.
	•	business/platform error code는 더 세밀한 분기 처리를 위한 2차 신호다.
	•	같은 400 계열 안에서도 validation, bad filter syntax, invalid cursor, unsupported sort, idempotency mismatch 등이 구분될 수 있어야 한다.
	•	같은 409 계열 안에서도 version conflict, duplicate create intent, state transition conflict를 구분할 수 있어야 한다.

문서에는 최소한 다음을 포함하라.
	•	HTTP status만으로 충분하지 않은 이유
	•	platform-specific code 체계 필요성
	•	status와 code가 모순되면 안 된다는 원칙
	•	내부 exception 이름을 그대로 외부 code로 노출하지 않아야 한다는 점

⸻

5-6. 오류 분류 체계 정의

최소한 아래 범주를 포함하라.

입력/계약 오류
	•	validation_error
	•	invalid_query
	•	unsupported_filter
	•	unsupported_sort
	•	invalid_cursor
	•	malformed_request

인증/보안 오류
	•	authentication_required
	•	invalid_credentials/token/session
	•	permission_denied
	•	tenant_scope_violation
	•	forbidden_action

리소스/상태 오류
	•	not_found
	•	already_exists 또는 duplicate_intent
	•	conflict
	•	invalid_state_transition
	•	stale_version / concurrency_conflict 가능성 검토

운영/제한 오류
	•	rate_limited
	•	service_unavailable
	•	timeout
	•	dependency_failure
	•	upstream_error

비동기/통합 오류
	•	operation_failed
	•	operation_not_ready
	•	webhook_delivery_failed
	•	webhook_signature_invalid
	•	idempotency_mismatch
	•	replay_not_available 가능성 검토
	•	indexing_not_ready
	•	retrieval_unavailable

각 범주에 대해:
	•	어떤 상황에서 쓰는지
	•	어떤 상위 status와 연결되는지
	•	외부에 얼마나 자세히 노출할지

를 정리하라.

⸻

5-7. 주요 오류 유형별 응답 원칙

반드시 아래 유형을 각각 다뤄라.
	•	validation failure
	•	authentication failure
	•	authorization failure
	•	not found
	•	conflict
	•	idempotency mismatch
	•	invalid state transition
	•	rate limited
	•	async operation failed
	•	upstream/dependency failure
	•	AI/retrieval/indexing not ready or unavailable
	•	webhook delivery / webhook verification failure

각 유형에 대해 다음을 정리하라.
	•	권장 HTTP status 범주
	•	대표 error code 방향
	•	message 구성 원칙
	•	details에 무엇을 넣을 수 있는지
	•	retryable 여부 판단 방향
	•	보안상 숨겨야 할 정보 여부
	•	운영 메타데이터와 어떻게 연결되는지

⸻

5-8. 운영 추적 메타데이터 규약

다음을 문서화하라.
	•	request_id
	•	trace_id
	•	correlation_id 여부 검토
	•	actor/tenant 문맥은 응답에 전부 노출하지 않더라도 내부 추적과 연결돼야 함
	•	operation_id
	•	event_id
	•	webhook_delivery_id
	•	idempotency_key reflected 여부 검토
	•	timestamp

중요:
이 문서는 observability stack 설계 문서가 아니므로,
응답 계약과 로그 상관관계를 위해 필요한 최소 운영 메타데이터 표준에 집중하라.

또한 아래를 구분하라.
	•	항상 응답에 포함할 후보
	•	일부 오류/응답에서만 조건부 포함할 후보
	•	외부 응답에는 비노출이지만 내부 로그에는 남겨야 할 정보

⸻

5-9. 보안 노출 제한 원칙

이 부분은 반드시 명확히 써라.
	•	내부 stack trace, DB 오류, 정책 엔진 상세, credential 정보는 외부 응답에 노출하지 않는다.
	•	not found와 forbidden 사이에서 리소스 존재 노출을 제한해야 하는 경우가 있다.
	•	보안 오류 메시지는 지나치게 상세하면 안 된다.
	•	외부 응답에는 최소 충분 정보만 제공하고, 내부 로그/감사 로그에는 더 풍부한 정보를 남긴다.
	•	AI consumer라고 해서 더 많은 내부 오류 상세를 주어서는 안 된다.

즉,
운영 가능성과 보안 최소 노출 사이의 경계를 문서화하라.

⸻

5-10. async operation / webhook / idempotency / AI 연계 오류 관점

이전 Task들과 연결하여 다음을 반드시 다뤄라.

Async operation
	•	작업 시작은 성공했지만 결과는 실패한 경우
	•	operation status endpoint와 immediate response의 오류 의미 차이
	•	아직 준비되지 않은 결과(not_ready) 표현 가능성

Webhook
	•	delivery 실패는 API 요청 자체 실패와 다름
	•	subscription 검증 실패와 delivery 실패를 구분해야 함

Idempotency
	•	key 누락
	•	key 충돌
	•	same key / different payload mismatch
	•	replay response unavailable

AI / RAG
	•	indexing not ready
	•	retrieval unavailable
	•	citation target not found
	•	permission-preserving retrieval failure
	•	ingestion in progress

중요:
이 확장 시나리오들이 기존 오류 체계 바깥에 별도 예외로 떠다니지 않도록
플랫폼 오류 모델 안으로 편입시켜라.

⸻

5-11. 감사 추적 / 운영 로그 / API 오류 응답의 관계

다음을 명확히 구분하라.
	•	API 오류 응답
	•	클라이언트에게 전달되는 계약
	•	운영 로그
	•	진단, 장애 분석, 성능 추적, trace correlation용
	•	감사 로그
	•	누가 무엇을 시도했고 어떤 결과가 났는지에 대한 보안/컴플라이언스용 기록

반드시 포함할 것:
	•	같은 실패라도 세 층위에 담기는 정보가 다를 수 있음
	•	API 응답은 최소 충분 정보
	•	운영 로그는 진단 중심
	•	감사 로그는 actor/action/resource/result 중심
	•	request_id/trace_id로 이 셋이 연결 가능해야 한다는 점

⸻

5-12. 예시 오류 응답 및 운영 메타데이터 예시 작성

문서에 최소한 아래 예시를 포함하라.

오류 응답 예시
	1.	validation error
	2.	permission denied
	3.	not found
	4.	conflict
	5.	idempotency mismatch
	6.	rate limited
	7.	operation failed
	8.	retrieval not ready
	9.	webhook signature invalid
	10.	dependency unavailable

예시는 JSON 형태로 제시하되,
각 예시가 왜 그런 구조를 갖는지 짧게 설명하라.

또한 가능하다면 아래도 포함하라.
	•	같은 HTTP status 안에서 code가 다른 사례 비교
	•	retryable=true/false 차이 예시
	•	외부 응답과 내부 로그 수준 차이 예시 개념 설명

⸻

5-13. error code 체계 작성 방향

전체 catalog를 완성할 필요는 없지만,
다음을 문서화하라.
	•	error code 네이밍 원칙
	•	짧고 안정적이며 기계 친화적이어야 함
	•	내부 예외 클래스명 의존 금지
	•	도메인 특화 코드와 공통 코드의 경계
	•	향후 확장 가능한 taxonomy 필요

예:
	•	validation.invalid_field
	•	auth.authentication_required
	•	auth.permission_denied
	•	resource.not_found
	•	state.invalid_transition
	•	idempotency.key_mismatch
	•	operation.failed
	•	retrieval.not_ready

중요:
최종 카탈로그를 다 만들지는 않아도,
코드 체계가 어떻게 자라야 하는지 기준은 정리하라.

⸻

5-14. 후속 Task 및 구현 Phase에 전달할 기준

문서 마지막에 다음 연결 포인트를 정리하라.

예:
	•	Task 3-11에서는 대표 API 시나리오에서 오류 응답과 추적 메타데이터가 일관적으로 적용되는지 검증해야 함
	•	Task 3-12에서는 전체 Phase 3 통합 문서에 error/observability 규약을 반영해야 함
	•	구현 Phase에서는 exception handler, request context middleware, logging correlation, audit pipeline, operation/webhook failure handling에 직접 연결되어야 함
	•	이후 rate limiting, concurrency control, resilience policy는 별도 구현 설계로 확장 가능

⸻

6. 산출물

아래 파일을 작성하라.

필수 산출물
	•	Task10_api_error_and_observability.md

문서 성격
	•	설계 기준 문서
	•	구현 코드 금지
	•	특정 로깅/모니터링 제품 종속 금지
	•	예시는 가능하나, 전체 catalog/스키마 구현 수준까지는 가지 말 것

⸻

7. 제외 범위

이번 작업에서는 아래는 하지 마라.
	•	실제 exception handler 코드 작성
	•	OpenAPI 전체 error schema 작성
	•	full error code catalog 완성
	•	로깅 시스템 제품 선택
	•	tracing backend 선택
	•	SLO/SLA 수치 설계
	•	alerting rule 설계
	•	rate limit 정책 상세 수치 결정
	•	concurrency control 상세 설계
	•	보안 사고 대응 프로세스 문서화

즉, 이번 작업은 오류 모델과 관측성 규약의 상위 기준 수립에 집중하라.

⸻

8. 작성 스타일 가이드

문서는 아래 스타일을 따르라.
	•	HTTP status와 business error code를 명확히 구분해서 설명
	•	성공 계약만큼 실패 계약도 중요하다는 관점을 유지
	•	보안 최소 노출과 운영 추적성의 균형을 분명히 설명
	•	권장 / 허용 / 비권장 / 금지 / 후속 Task에서 구체화를 구분
	•	async / webhook / idempotency / AI retrieval 같은 확장 시나리오를 빠뜨리지 말 것
	•	예시는 충분히 실용적으로 작성하되 문서가 구현 스펙으로 과도하게 내려가지 않게 할 것

⸻

9. 검토 체크리스트

작업 완료 전에 아래를 스스로 점검하라.
	•	왜 오류 모델과 관측성 규약이 필요한지 충분히 설명했는가?
	•	오류 기본 구조와 metadata 구조가 정리되었는가?
	•	HTTP status와 business error code 관계가 명확한가?
	•	주요 오류 범주와 유형이 충분히 분류되었는가?
	•	보안 노출 제한 원칙이 포함되었는가?
	•	async / webhook / idempotency / AI 연계 오류가 포함되었는가?
	•	API 오류 응답 / 운영 로그 / 감사 로그 관계가 설명되었는가?
	•	예시 오류 응답이 실용적인가?
	•	후속 Task와 구현 단계가 이 문서를 기준으로 이어질 수 있는가?

⸻

10. Claude Code에 대한 최종 지시

위 요구사항을 반영하여 Task10_api_error_and_observability.md 문서를 작성하라.

문서는 범용 문서 플랫폼 API의 실패 상황을 일관되게 표현하고,
운영·보안·감사·디버깅을 위해 필요한 추적 정보를 계약 차원에서 반영할 수 있게 하는
API 오류 모델 및 운영 관측성 규약의 상위 설계 기준 문서여야 한다.

문서 작성 후에는 아래 형식의 자체 점검 요약도 함께 정리하라.

자체 점검 요약
	•	오류 모델 기본 shape 요약
	•	HTTP status / business error code 관계 요약
	•	주요 오류 범주 요약
	•	운영 추적 메타데이터 요약
	•	보안 노출 제한 원칙 요약
	•	async/webhook/idempotency/AI 연계 오류 관점 요약
	•	API 오류 응답 / 운영 로그 / 감사 로그 관계 요약
	•	후속 Task 연결 포인트
	•	의도적으로 후속 Task로 남긴 미결정 사항