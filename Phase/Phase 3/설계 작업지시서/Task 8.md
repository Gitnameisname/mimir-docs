Task 3-8. 비동기 작업 / 이벤트 / 웹훅 확장 모델 설계

Claude Code 작업 지시서

1. 작업명

Phase 3 - Task 3-8. 비동기 작업 / 이벤트 / 웹훅 확장 모델 설계

⸻

2. 작업 목적

플랫폼 API를 단순 동기식 REST 호출에 머무르지 않고, 비동기 작업(async operations), 이벤트(event), 웹훅(webhook) 으로 확장할 수 있도록 상위 설계 기준을 문서로 정리하라.

이번 작업의 목적은 메시지 브로커를 고르거나 웹훅 전송 코드를 작성하는 것이 아니다.
핵심은 다음이다.
	•	동기 REST 응답만으로 처리하기 어려운 작업을 어떤 구조로 API에 연결할지 정의
	•	장시간 실행 작업, 재색인, export, preview 생성, ingestion, bulk 작업 등에서 사용할 비동기 작업 모델 정립
	•	문서 변경, 권한 변경, 상태 전이, 인덱싱, 작업 완료 등에서 사용할 이벤트 모델 정립
	•	외부 시스템 연동을 위한 웹훅 구독 및 전달 모델 정립
	•	이후 구현 Phase에서 jobs/operations/events/webhooks를 실제로 만들 수 있도록, API 계약과 확장 포인트 기준선을 마련

즉, 이 문서는 플랫폼 API의 비동기/이벤트 확장 아키텍처 기준 문서여야 한다.

⸻

3. 반드시 반영할 배경

이 플랫폼은 다음 성격을 가진다.
	•	범용 문서 플랫폼
	•	REST 중심 API 구조
	•	문서/버전/노드/권한/감사 추적/검색/AI 연계 기능 존재
	•	User/Admin UI, 외부 시스템, AI/Agent, 내부 서비스가 모두 API 소비자
	•	일부 작업은 즉시 끝나지 않거나, 외부 통지/후처리/재시도가 필요함
	•	향후 검색 인덱싱, AI ingestion, RAG sync, export, 대량 작업, workflow agent 연계까지 고려해야 함

이전 Task에서 이미 다음이 정해졌다고 가정하라.
	•	플랫폼 API는 계약 계층이다.
	•	REST 리소스 구조가 존재한다.
	•	인증/인가 구조와 request security context가 있다.
	•	버전 전략이 있다.
	•	공통 응답 포맷이 있다.
	•	목록 조회 규약이 있다.
	•	idempotency 및 retry safety 기준이 있다.

따라서 이번 작업은 REST를 버리는 것이 아니라,
REST 기반 플랫폼 위에 비동기/이벤트/웹훅 확장 모델을 얹는 작업이어야 한다.

⸻

4. 이번 작업의 핵심 질문

이번 문서에서는 최소한 아래 질문들에 답할 수 있어야 한다.
	1.	어떤 작업은 동기 처리보다 비동기 작업 리소스로 표현하는 것이 적절한가?
	2.	jobs와 operations 중 어떤 개념이 더 적절한가, 혹은 둘 다 필요한가?
	3.	비동기 작업은 REST API와 어떻게 연결되는가?
	4.	이벤트는 내부 구현 세부가 아니라 API/플랫폼 계약 차원에서 어떤 의미를 가져야 하는가?
	5.	어떤 종류의 도메인 이벤트를 정의해야 하는가?
	6.	웹훅은 어떤 이벤트를 외부로 전달하기 위한 수단인가?
	7.	웹훅 구독 리소스와 웹훅 전달 이력(delivery)은 어떤 구조를 가져야 하는가?
	8.	실패, 재시도, 중복 전달, 보안 검증은 어떤 원칙으로 다뤄야 하는가?
	9.	비동기 작업 상태 조회, 이벤트 추적, 웹훅 전달 조회는 어떤 운영 리소스로 노출해야 하는가?
	10.	AI/RAG ingestion이나 indexing workflow도 같은 확장 모델 안에 편입할 수 있는가?

⸻

5. 상세 작업 요구사항

5-1. 문서 작성

아래 산출물을 작성하라.

필수 산출물
	•	Task8_event_webhook_extension.md

문서에 반드시 포함할 섹션
	1.	문서 목적
	2.	왜 비동기 작업 / 이벤트 / 웹훅 확장 모델이 필요한가
	3.	상위 설계 원칙
	4.	비동기 작업 모델 설계
	5.	작업 리소스(jobs 또는 operations) 방향
	6.	이벤트 모델 설계
	7.	이벤트 분류 체계
	8.	웹훅 구독 모델 설계
	9.	웹훅 전달(delivery) 모델 설계
	10.	실패 / 재시도 / 중복 / idempotency 관점
	11.	보안 및 인증/서명 관점
	12.	운영 관측성 및 감사 추적 연계
	13.	AI/RAG/인덱싱/대량 작업 확장 관점
	14.	후속 Task 및 구현 Phase에 전달할 기준
	15.	결론

⸻

5-2. 왜 비동기 작업 / 이벤트 / 웹훅 확장 모델이 필요한가

문서 초반에서 다음을 명확히 설명하라.
	•	모든 작업이 요청-응답 한 번으로 끝나지 않는다.
	•	일부 작업은 오래 걸리고, 상태를 추적해야 한다.
	•	일부 작업은 내부 후처리나 외부 시스템 통지가 필요하다.
	•	문서 플랫폼은 CRUD를 넘어 인덱싱, export, validation, preview 생성, AI ingestion, bulk 변경 같은 작업을 다루게 된다.
	•	외부 통합 시스템은 polling보다 event/webhook 기반 연계를 선호할 수 있다.
	•	UI와 운영 도구도 작업 상태/실패 원인/재시도 여부를 알아야 한다.

즉, 이 확장 모델은 부가 기능이 아니라
플랫폼을 실제 운영 가능한 수준으로 만드는 핵심 확장 계층이라는 점이 드러나야 한다.

⸻

5-3. 상위 설계 원칙 정의

최소한 아래 원칙을 포함하라.
	•	REST-first, async-when-needed
	•	Explicit operation tracking
	•	Events as platform facts
	•	Webhooks as external delivery mechanism
	•	Decoupled but observable
	•	Idempotent-friendly delivery
	•	Secure by default
	•	Auditable lifecycle
	•	Consistent resource modeling
	•	Extensible to AI and automation workflows

각 원칙마다 다음을 설명하라.
	•	의미
	•	왜 필요한지
	•	실제 설계에서 어떤 결정을 유도하는지

예:
	•	REST-first, async-when-needed → 기본은 동기 REST지만 오래 걸리거나 후처리가 많은 작업은 비동기 모델로 승격
	•	Events as platform facts → 이벤트는 단순 로그가 아니라 “무슨 일이 일어났는지”를 표현하는 플랫폼 사실 단위
	•	Webhooks as external delivery mechanism → 웹훅은 이벤트 자체가 아니라 외부 전달 수단이라는 점을 구분

⸻

5-4. 비동기 작업 모델 설계

다음을 정리하라.
	•	어떤 작업을 비동기로 다루는 것이 적절한지
	•	동기 처리와 비동기 처리의 구분 기준
	•	클라이언트가 작업 시작 요청을 보내면 어떤 계약을 기대해야 하는지
	•	작업 시작 직후 즉시 응답과 이후 상태 조회 흐름
	•	작업 상태(state) 개념
	•	작업 완료 결과(result)와 작업 리소스 자체의 차이
	•	작업 실패/부분 실패/취소 가능성

검토 대상 예시:
	•	export 시작
	•	reindex 시작
	•	ingestion 시작
	•	regenerate preview
	•	대량 archive
	•	bulk permission update
	•	AI embedding/indexing
	•	validation batch 실행

중요:
비동기 작업 모델은 이후 jobs 또는 operations 리소스로 연결될 수 있어야 한다.

⸻

5-5. 작업 리소스(jobs 또는 operations) 방향

문서에서 다음을 비교하고 권장 방향을 제시하라.

후보 1. jobs
	•	실행 단위 느낌이 강함
	•	배치/백그라운드 작업과 잘 맞음

후보 2. operations
	•	API 요청으로부터 파생된 장기 실행 작업 느낌
	•	더 일반적인 operation-tracking에 적합할 수 있음

후보 3. 둘을 역할 분리해서 병행
	•	외부 API는 operations
	•	내부 실행 엔진은 jobs

각 후보에 대해:
	•	장점
	•	단점
	•	외부 API 계약 적합성
	•	운영 추적성
	•	AI/automation 연계 적합성
	•	이 플랫폼에서의 적합성

최종적으로 어떤 방향이 더 적절한지 제안하라.

중요:
이번 문서에서는 구현 테이블이 아니라 플랫폼 계약 용어를 정하는 것이 핵심이다.

⸻

5-6. 비동기 작업 상태 모델 정의

다음을 문서화하라.
	•	queued
	•	accepted
	•	running
	•	succeeded
	•	failed
	•	canceled
	•	partial_success 여부 검토
	•	retrying 여부 검토 가능

각 상태의 의미를 정리하고,
다음도 포함하라.
	•	작업 상태 전이 원칙
	•	최종 상태와 중간 상태 구분
	•	실패 상태에 어떤 수준의 오류 정보가 필요한지
	•	progress percentage / processed count / ETA 같은 운영 필드의 위치 방향
	•	작업 대상 리소스와 결과 리소스를 어떻게 참조할지

⸻

5-7. 이벤트 모델 설계

이벤트를 다음 관점에서 설명하라.
	•	이벤트는 내부 브로커 메시지와 동일 개념이 아님
	•	플랫폼 차원에서 “무슨 일이 발생했는가”를 나타내는 계약적 사실 단위
	•	이벤트는 audit log와 유사하지만 동일하지 않음
	•	모든 내부 상태 변화가 외부 이벤트가 되는 것은 아님
	•	도메인 이벤트와 운영 이벤트를 구분할 필요가 있음

반드시 포함할 것:
	•	이벤트 최소 공통 속성 후보
	•	event_id
	•	event_type
	•	occurred_at
	•	actor context 여부
	•	subject/resource reference
	•	tenant/org context
	•	correlation/request/operation reference
	•	payload summary
	•	이벤트는 플랫폼 내부와 외부 통합 모두에서 의미가 있어야 함

⸻

5-8. 이벤트 분류 체계 정의

최소한 아래 범주를 검토하라.

도메인 이벤트
	•	document.created
	•	document.updated
	•	document.published
	•	version.created
	•	node.updated
	•	permission.changed
	•	attachment.added

운영 이벤트
	•	operation.started
	•	operation.completed
	•	operation.failed
	•	webhook.delivery.failed
	•	index.sync.completed

통합/AI 관련 이벤트
	•	ingestion.started
	•	ingestion.completed
	•	retrieval-index.updated
	•	ai-processing.failed

각 범주에 대해:
	•	외부 공개 가치가 있는지
	•	내부 전용인지
	•	웹훅 구독 대상으로 적절한지
	•	감사 추적과 어떤 관계가 있는지

⸻

5-9. 웹훅 구독 모델 설계

다음을 정리하라.
	•	웹훅은 외부 시스템에 이벤트를 push delivery하기 위한 메커니즘
	•	웹훅 구독(subscription) 자체를 API 리소스로 둘 것인지
	•	구독 리소스의 핵심 속성 후보
	•	id
	•	target_url
	•	subscribed_events
	•	secret/reference
	•	status
	•	created_by
	•	tenant scope
	•	filters 여부
	•	활성/비활성 상태
	•	이벤트 타입별 구독
	•	조직 범위나 리소스 범위 구독 가능성
	•	test webhook / verification 흐름 고려 여부

중요:
이번 작업은 구현이 아니라 구독 리소스 모델과 계약 방향을 정하는 것이다.

⸻

5-10. 웹훅 전달(delivery) 모델 설계

다음을 정리하라.
	•	delivery는 subscription과 별개의 운영 리소스인지
	•	delivery 상태 추적 필요성
	•	핵심 속성 후보
	•	delivery_id
	•	webhook_id
	•	event_id
	•	delivery_status
	•	attempt_count
	•	first_attempt_at
	•	last_attempt_at
	•	response_status
	•	next_retry_at
	•	성공/실패/재시도 상태
	•	dead-letter 또는 permanently_failed 개념 검토 가능
	•	delivery 조회 API의 필요성
	•	관리자/운영자와 일반 사용자 노출 범위 차이

중요:
웹훅 delivery는 단순 내부 로그가 아니라,
운영성과 외부 연동 디버깅에 필요한 API 리소스가 될 수 있다는 점을 다뤄라.

⸻

5-11. 실패 / 재시도 / 중복 / idempotency 관점

Task 3-7과 연결하여 반드시 다음을 다뤄라.

비동기 작업
	•	같은 요청이 중복으로 작업을 여러 개 만들지 않도록 할 필요
	•	작업 시작 endpoint에 idempotency 적용 가능성

이벤트
	•	동일 사건이 여러 소비자에 의해 중복 관찰될 수 있음
	•	이벤트 고유 식별자의 필요성

웹훅
	•	at-least-once delivery가 일반적일 수 있음
	•	따라서 수신자도 idempotent consumer가 되어야 할 수 있음
	•	동일 delivery 재시도와 새로운 delivery를 구분할 필요

문서에 다음을 반영하라.
	•	정확히 한 번(exactly-once)을 쉽게 약속하지 말 것
	•	대신 at-least-once + idempotent consumer 친화 설계를 우선 검토
	•	replay/retry/deduplication에 대한 현실적 정책 방향 제시

⸻

5-12. 보안 및 인증/서명 관점

다음을 정리하라.

비동기 작업
	•	누가 시작했는지 actor context 필요
	•	background worker가 누구 권한으로 동작하는지 추적 가능해야 함

이벤트
	•	민감한 payload 외부 노출 통제 필요
	•	tenant scope를 넘는 전달 금지

웹훅
	•	웹훅 수신자 검증 또는 발신 서명 필요성
	•	secret 관리 개념
	•	재전송 공격/위조 요청 방지 고려
	•	test endpoint와 production endpoint 구분 검토 가능

중요:
구체 서명 알고리즘을 고르지 말고,
보안 원칙과 요구사항을 정리하라.

⸻

5-13. 운영 관측성 및 감사 추적 연계

다음을 문서화하라.
	•	operation/job은 상태 조회와 진단이 가능해야 함
	•	request_id / trace_id / actor / tenant / event_id / webhook delivery id 간 연결 가능해야 함
	•	감사 로그와 operation/event/webhook delivery의 관계
	•	사용자가 수행한 요청이 어떤 operation과 어떤 이벤트를 낳았는지 추적 가능해야 함
	•	실패 원인과 재시도 이력이 운영 도구에서 보일 수 있어야 함

즉, 이 확장 계층도 observability-first 관점으로 설계해야 한다.

⸻

5-14. AI / RAG / 인덱싱 / 대량 작업 확장 관점

이 플랫폼 특성상 중요하다.
다음을 반드시 다뤄라.
	•	ingestion, embedding, indexing, retrieval sync 같은 작업은 전형적인 async operation 후보
	•	AI processing 상태 추적은 운영상 중요
	•	AI/RAG 관련 이벤트도 플랫폼 이벤트 체계 안에 들어와야 함
	•	대량 작업(bulk change, export, sync)도 동일한 operation 모델을 재사용하는 것이 바람직한지 검토
	•	Agentic workflow가 operation status polling / event subscription / webhook consumption을 활용할 수 있어야 함

즉, 이 문서는 전통 CRUD 시스템이 아니라
AI 확장형 문서 플랫폼을 염두에 두고 작성되어야 한다.

⸻

5-15. 예시 시나리오 포함

문서에 최소한 아래 시나리오를 포함하라.

시나리오 예시
	1.	문서 export 시작 → operation 생성 → 완료 상태 조회
	2.	document.published 이벤트 발생 → 웹훅 구독자에게 전달
	3.	webhook delivery 실패 → 재시도 → delivery 상태 조회
	4.	reindex 요청 → 동일 idempotency key 재요청 → 동일 operation 참조
	5.	AI ingestion 시작 → operation 진행률 업데이트 → 완료 이벤트 발행

예시는 상세 구현이 아니라 계약 흐름 예시 수준으로 작성하라.

⸻

5-16. 후속 Task 및 구현 Phase에 전달할 기준

문서 마지막에 다음 연결 포인트를 정리하라.

예:
	•	Task 3-9에서는 AI/RAG 인터페이스가 operation/event 모델을 어떻게 소비할지 구체화해야 함
	•	Task 3-10에서는 async/webhook/event 관련 error model과 observability metadata를 더 구체화해야 함
	•	구현 Phase에서는 operation store, event publisher, webhook dispatcher, delivery tracker 설계로 이어져야 함
	•	이후 scheduler/retry policy/dead-letter 처리 전략은 별도 구현 설계에서 다뤄야 함
	•	보안 서명/비밀 관리/권한 범위 검증은 auth/security 구현 단계와 연결되어야 함

⸻

6. 산출물

아래 파일을 작성하라.

필수 산출물
	•	Task8_event_webhook_extension.md

문서 성격
	•	설계 기준 문서
	•	구현 코드 금지
	•	메시지 브로커/큐 제품 선택 금지
	•	웹훅 전송 코드/재시도 알고리즘 구현 금지
	•	API 계약과 확장 모델 중심

⸻

7. 제외 범위

이번 작업에서는 아래는 하지 마라.
	•	Kafka, RabbitMQ, SQS 등 특정 기술 채택 결정
	•	background worker 구현 상세
	•	cron/scheduler 설계 상세
	•	webhook signature algorithm 확정
	•	dead-letter queue 구현 설계
	•	retry backoff 수치 결정
	•	event payload schema 상세 확정
	•	operation persistence schema 상세 설계
	•	SSE/WebSocket 실시간 채널 상세 설계
	•	OpenAPI 전체 작성

즉, 이번 작업은 비동기 작업 / 이벤트 / 웹훅 확장 모델의 상위 기준 수립에 집중하라.

⸻

8. 작성 스타일 가이드

문서는 아래 스타일을 따르라.
	•	REST와 비동기 확장을 대립시키지 말고 연결된 구조로 설명
	•	이벤트, 감사 로그, 웹훅 delivery의 차이를 명확히 구분
	•	작업 리소스와 이벤트 리소스를 혼동하지 않도록 서술
	•	권장 / 허용 / 비권장 / 금지 / 후속 Task에서 구체화를 구분
	•	운영 현실(실패, 재시도, 중복, 진단)을 충분히 반영
	•	AI/RAG/인덱싱/대량 작업 확장성을 반드시 고려

⸻

9. 검토 체크리스트

작업 완료 전에 아래를 스스로 점검하라.
	•	왜 async/event/webhook 확장이 필요한지 충분히 설명했는가?
	•	jobs vs operations 방향이 비교되고 권장안이 제시되었는가?
	•	작업 상태 모델이 정리되었는가?
	•	이벤트의 의미와 분류 체계가 있는가?
	•	웹훅 subscription과 delivery가 구분되었는가?
	•	실패/재시도/idempotency 관점이 반영되었는가?
	•	보안/감사/관측성 관점이 포함되었는가?
	•	AI/RAG/인덱싱/대량 작업까지 고려되었는가?
	•	후속 Task와 구현 단계가 이 문서를 기준으로 이어질 수 있는가?

⸻

10. Claude Code에 대한 최종 지시

위 요구사항을 반영하여 Task8_event_webhook_extension.md 문서를 작성하라.

문서는 범용 문서 플랫폼 API를 동기 REST 호출에만 머무르지 않고,
비동기 작업 추적, 플랫폼 이벤트, 외부 웹훅 통지까지 확장할 수 있게 하는 상위 설계 기준 문서여야 한다.

문서 작성 후에는 아래 형식의 자체 점검 요약도 함께 정리하라.

자체 점검 요약
	•	비동기 작업 모델 핵심 요약
	•	jobs/operations 용어 선택 방향 요약
	•	이벤트 분류 체계 요약
	•	웹훅 subscription/delivery 모델 요약
	•	실패/재시도/idempotency 관점 요약
	•	보안/관측성/감사 연계 요약
	•	AI/RAG/인덱싱 확장 관점 요약
	•	후속 Task 연결 포인트
	•	의도적으로 후속 Task로 남긴 미결정 사항