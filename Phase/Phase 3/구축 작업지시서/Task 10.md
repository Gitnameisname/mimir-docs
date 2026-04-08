Task I-10. audit / observability 연결 포인트 구현

Claude Code 작업 계획서

1. 작업 목표

플랫폼 API 요청과 결과를 운영적으로 추적하고, 이후 감사 추적(audit), 보안 분석, 장애 분석, 성능 관찰성을 확장할 수 있도록 audit / observability의 최소 연결 포인트를 구현한다.

이번 단계의 핵심은 완전한 로깅 플랫폼이나 SIEM 연동을 만드는 것이 아니다.
핵심은 다음 구조를 실제 코드 흐름으로 세우는 것이다.
	•	request_id / trace_id 중심의 요청 추적
	•	actor / tenant / resource / action 기준의 이벤트 shape 정리
	•	성공 / 실패 흐름에 대한 공통 로그 포인트 마련
	•	감사 이벤트 emit hook point 마련
	•	향후 operations / webhook / idempotency / authz와 correlation 가능한 구조 확보
	•	router마다 제각각 print/logging을 하지 않도록 공통화

즉, 이번 Task는 운영 추적과 감사 추적의 공통 관측 지점을 코드에 심는 단계다.

⸻

2. 작업 범위

포함 범위
	•	request logging baseline 설계
	•	structured log shape 초안 정의
	•	actor / tenant / resource / action correlation field 설계
	•	성공 / 실패 요청에 대한 공통 logging hook 구현
	•	audit event emit interface 또는 hook point 정의
	•	service/authz/idempotency와 연결 가능한 observability 경계 마련
	•	최소한의 duration/status/result 관측 포인트 마련
	•	selected endpoint 또는 middleware 레벨에 시범 적용

제외 범위
	•	full centralized logging stack 구축
	•	ELK / OpenSearch / Datadog / Grafana 연동 완성
	•	metrics backend 완성
	•	tracing backend 완성
	•	audit persistence 완성
	•	보안 탐지 규칙 구현
	•	대시보드 구축
	•	알림/경보 체계 구축

⸻

3. 구현 의도

지금까지 API skeleton, response/error, request context, auth hook, list query, documents/versions/nodes, idempotency hook을 만들었지만,
운영 관측 지점이 없으면 실제 서비스에서는 다음 문제가 생긴다.
	•	어떤 요청이 어떤 actor에 의해 발생했는지 모름
	•	같은 오류가 어디서 반복되는지 추적 어려움
	•	문서 생성/버전 생성 재시도 문제를 audit/logging으로 상관 분석할 수 없음
	•	permission denied / validation error / internal error를 통계적으로 구분하기 어려움
	•	향후 webhook/operation/AI retrieval 확장 시 correlation이 깨짐

따라서 이번 작업의 목적은 다음과 같다.
	•	request lifecycle의 공통 관측 포인트를 만든다
	•	actor / tenant / action / resource / request_id를 하나의 관측 키로 묶는다
	•	감사 이벤트를 이후 영속화/전송할 수 있는 hook point를 확보한다
	•	현재는 최소 logging 수준이지만, 미래 확장 시 재작업이 적도록 구조를 고정한다

⸻

4. Claude Code에게 요청할 작업

아래 작업을 순서대로 수행하라.

⸻

Step 1. 현재 logging/추적 방식 점검

현재 코드베이스에 이미 존재하는 logging/관측 관련 구조를 먼저 점검하라.

확인할 항목:
	•	FastAPI app 레벨 logging 방식
	•	middleware에서 request logging이 있는지
	•	exception handler에서 logging을 하는지
	•	service/repository에서 개별 logging이 흩어져 있는지
	•	request_id / trace_id를 로그에 남기는 구조가 있는지
	•	auth/idempotency/documents/versions write 흐름에서 현재 관측 가능한 정보가 무엇인지

목적:
	•	기존 구조를 무시하지 않고, 공통 관측 포인트를 가장 낮은 마찰로 삽입하기 위함이다.

⸻

Step 2. observability baseline event shape 정의

요청/응답/오류/감사 이벤트에서 재사용할 수 있는 공통 event shape를 정리하라.

권장 최소 필드:
	•	event_type
	•	request_id
	•	trace_id
	•	actor_id
	•	actor_type
	•	tenant_id
	•	org_id
	•	client_type
	•	http_method
	•	path
	•	action
	•	resource_type
	•	resource_id
	•	status_code
	•	result
	•	예: success / failure / denied / validation_error
	•	duration_ms
	•	error_code optional

선택 필드:
	•	idempotency_key
	•	operation_id
	•	version_id
	•	document_id

중요:
	•	이번 단계에서는 모든 필드를 항상 채우지 않아도 된다
	•	필드 이름과 shape를 먼저 안정화하는 것이 목적이다

⸻

Step 3. request lifecycle logging 포인트 정의

요청 lifecycle 어디에서 로그를 남길지 기준을 정하라.

권장 최소 포인트:
	•	request start
	•	request completion
	•	request failure
	•	authorization denied
	•	idempotency replay / conflict
	•	write success (selected endpoints)
	•	audit candidate event emit

권장 방향:
	•	request start는 너무 시끄럽다면 optional
	•	request completion / failure는 기본
	•	민감한 write action은 별도 audit candidate 이벤트를 발생시킬 수 있게 하라

중요:
	•	모든 레이어가 중복으로 똑같은 로그를 찍지 않게 할 것
	•	“운영 로그”와 “감사 이벤트”의 역할을 구분할 것

⸻

Step 4. structured logger/helper 또는 공통 logging utility 구현

각 레이어가 문자열 로그를 제각각 만들지 않도록, structured logging helper를 구현하라.

예시 개념:
	•	log_api_event(...)
	•	log_request_completion(...)
	•	log_audit_candidate(...)

핵심 요구:
	•	request context에서 request_id/trace_id/actor/tenant를 쉽게 주입할 수 있어야 한다
	•	event shape가 일관되어야 한다
	•	단순 문자열 concatenation보다 구조화된 dict/log record 형태를 우선 검토하라

중요:
	•	documents / versions / idempotency / authz가 같은 helper를 사용할 수 있어야 한다

⸻

Step 5. middleware 또는 동등 위치에 request completion logging 연결

요청 성공/실패와 duration을 중앙에서 관찰할 수 있도록 middleware 또는 동등 구조에 공통 logging을 연결하라.

최소 요구:
	•	request 시작 시각 기록
	•	response 완료 시 duration 계산
	•	request_id / trace_id / path / method / status_code / actor 기본 정보 로그 반영
	•	예외 발생 시 failure 결과 반영

중요:
	•	middleware가 너무 많은 비즈니스 의미를 알 필요는 없다
	•	resource_id/action 같은 도메인 정보는 이후 deeper layer에서 보강될 수 있다

⸻

Step 6. exception handler와 failure logging 연결

Task I-3의 공통 exception handler와 logging이 자연스럽게 연결되게 하라.

반드시 반영:
	•	error code
	•	status code
	•	request_id / trace_id
	•	path / method
	•	actor / tenant 기본 정보
	•	internal error와 business error 구분 가능성

중요:
	•	외부 응답은 안전하게 유지하되, 내부 로그는 운영 추적에 필요한 최소 맥락을 남겨야 한다
	•	stack trace 노출은 응답이 아니라 내부 로그에서만 관리하라

⸻

Step 7. action / resource correlation field 연결

Task I-5에서 도입한 action / resource reference 흐름과 observability를 연결하라.

권장 연결 정보:
	•	action
	•	예: document.create, document.read, version.create
	•	resource_type
	•	document / version / node
	•	resource_id
	•	parent_document_id optional
	•	tenant_id optional

목표:
	•	단순 HTTP 로그가 아니라 “무슨 비즈니스 액션이 수행되었는지”까지 추적 가능하게 함
	•	audit/event/webhook/AI operation과의 상관관계를 확보함

중요:
	•	라우터나 service가 action/resource를 이미 알고 있다면 그 정보를 공통 logging helper에 전달할 수 있어야 한다

⸻

Step 8. audit event emit hook interface 정의

지금 당장 감사 이벤트를 영속화하지 않더라도, emit interface는 이번 단계에서 정의하라.

권장 개념:
	•	AuditEventEmitter
	•	emit_audit_event(...)
	•	publish_audit_candidate(...)

권장 이벤트 필드:
	•	event_type
	•	action
	•	actor
	•	tenant
	•	resource
	•	result
	•	request_id
	•	trace_id
	•	timestamp
	•	metadata optional

중요:
	•	현재는 stub/no-op 구현이어도 괜찮다
	•	하지만 나중에 DB 저장, message bus 발행, webhook 발행으로 자연스럽게 확장 가능해야 한다

⸻

Step 9. 어떤 이벤트를 audit candidate로 볼지 최소 정책 정리

모든 요청을 감사 이벤트로 저장할 필요는 없으므로, 최소 정책을 정하라.

권장 audit candidate 예:
	•	document create
	•	document update
	•	version create
	•	authorization denied on protected resource
	•	idempotency conflict
	•	sensitive metadata change optional

비권장:
	•	health check
	•	단순 public info read
	•	너무 잦은 low-value read 로그 전부

중요:
	•	이번 단계는 “무엇을 emit hook에 올릴지” 기준을 만드는 것이 목적이다
	•	full audit catalog는 이후 확장 가능하다

⸻

Step 10. idempotency / authz / documents / versions 흐름과 관측 포인트 연결

앞서 만든 핵심 기능들과 observability를 실제로 연결하라.

최소 연결 권장:
	•	documents create success
	•	documents update success
	•	versions create success
	•	authorization denied
	•	idempotency replay
	•	idempotency conflict

목표:
	•	공통 event shape가 실제 endpoint 흐름에서 사용 가능한지 검증
	•	write/action 중심 관측이 가능해지는지 확인

⸻

Step 11. request_id / trace_id / actor / tenant 상관관계 일관성 검토

Task I-4의 RequestContext와 실제 logging/audit hook이 같은 정보를 참조하는지 확인하라.

반드시 확인:
	•	request completion log의 request_id가 response meta와 같은가
	•	error log의 request_id가 error response meta와 같은가
	•	actor/tenant/client 정보가 같은 context에서 읽히는가
	•	idempotency record / audit event / API log가 동일 correlation field를 공유할 수 있는가

⸻

Step 12. 민감 정보 로깅 안전선 정리

observability를 강화한다고 해서 민감 정보를 과도하게 로깅하면 안 된다.
따라서 로깅 안전선을 정하라.

반드시 주의:
	•	raw auth token 금지
	•	session secret 금지
	•	전체 request body 덤프 금지
	•	민감 metadata 전체 덤프 금지
	•	개인식별 정보 과도한 노출 금지

허용 가능한 방향:
	•	actor_id 같은 정규화된 식별자
	•	resource_id
	•	action/result/status/error_code
	•	sanitized summary
	•	payload fingerprint 또는 size 정도

중요:
	•	감사와 운영 관측은 정보량이 많아질수록 좋지 않다
	•	안전하고 구조적인 최소 로깅이 원칙이다

⸻

Step 13. success/failure result taxonomy 최소 정의

로그/감사 이벤트에서 결과 분류를 최소한 정리하라.

권장 예:
	•	success
	•	failure
	•	denied
	•	validation_error
	•	conflict
	•	replayed

목표:
	•	대시보드나 후속 통계에서 분류가 쉬워지게 함
	•	HTTP status만으로는 부족한 의미를 보완

중요:
	•	taxonomy를 너무 많이 늘리지 말 것
	•	운영적으로 의미 있는 최소 집합이면 충분하다

⸻

Step 14. selected endpoint 또는 service에 시범 적용

공통 logging/audit hook을 대표적인 endpoint 흐름에 시범 적용하라.

권장 대상:
	•	POST /api/v1/documents
	•	PATCH /api/v1/documents/{document_id}
	•	POST /api/v1/documents/{document_id}/versions
	•	GET /api/v1/documents optional
	•	denied/conflict 흐름 1~2개

목표:
	•	create/update/version write 흐름에서 audit candidate emit이 자연스러운지 확인
	•	read endpoint에는 너무 무겁지 않게 적용되는지 확인

⸻

Step 15. future operations / webhook / retrieval 확장성과의 연결 검토

이번 구조가 이후 확장을 막지 않는지 검토하라.

확인할 점:
	•	operation_id를 나중에 event shape에 넣기 쉬운가
	•	webhook delivery result correlation을 붙이기 쉬운가
	•	retrieval/search/read-model 이벤트도 같은 shape로 일부 재사용 가능한가
	•	AI action trace와 request trace가 충돌하지 않는가

중요:
	•	지금 구현 범위 밖이지만, event shape가 너무 HTTP 중심으로 굳어지면 안 된다

⸻

Step 16. 최소 검증 시나리오 정리

아래 시나리오를 최소한 검증 가능하게 정리하라.

필수 시나리오:
	1.	documents create 성공 시 request completion log + audit candidate emit
	2.	versions create 성공 시 correlation field 포함 log 발생
	3.	authz denied 시 denied result 로그 발생
	4.	validation error 시 structured failure log 발생
	5.	idempotency replay/conflict 시 별도 result 분류 가능
	6.	request_id / trace_id가 response/error/meta/log에서 일관됨

⸻

5. 구현 시 세부 요구사항

5-1. 운영 로그와 감사 이벤트를 구분할 것

둘은 비슷하지만 목적이 다르다.
	•	운영 로그: 성능/오류/추적 관점
	•	감사 이벤트: 누가 무엇을 했는지의 변경 추적 관점

이번 단계에서는 둘을 완전히 분리 저장하지 않아도 되지만,
구조적 역할 구분은 반드시 유지하라.

⸻

5-2. structured logging 우선

문자열 위주의 ad-hoc 로그는 이후 필터링과 분석이 어렵다.
가능하면 structured dict 기반 또는 동일 키 집합을 갖는 logger helper를 우선하라.

⸻

5-3. correlation fields를 중심에 둘 것

이번 단계의 핵심은 “많이 로그 남기기”가 아니라,
	•	request_id
	•	trace_id
	•	actor_id
	•	tenant_id
	•	action
	•	resource_id

이런 correlation field를 중심에 두는 것이다.

⸻

5-4. middleware는 HTTP baseline, service는 business meaning 보강

권장 역할 분담:
	•	middleware: method/path/status/duration/request_id 등 HTTP baseline
	•	service/authz/idempotency: action/resource/result 등 business meaning 보강

이렇게 나누면 구조가 깔끔해진다.

⸻

5-5. request body 전체 로깅 금지

특히 documents/version payload는 커질 수 있고 민감할 수 있다.
전체 payload dump는 피하라.

대안:
	•	payload size
	•	changed fields 목록
	•	fingerprint
	•	sanitized summary

⸻

5-6. result taxonomy는 단순하게 유지

success/failure/denied/replayed 정도로도 충분한 경우가 많다.
초기 단계부터 세밀한 taxonomy를 과도하게 늘리지 말 것.

⸻

5-7. no-op audit emitter를 허용

감사 영속화 backend가 아직 없다면, emit interface + stub/no-op 구현은 괜찮다.
중요한 것은 emit 지점과 계약을 먼저 고정하는 것이다.

⸻

6. 권장 설계 방향

Claude Code는 아래 방향을 우선 검토하라.

옵션 A. Logging helper + Audit emitter interface + middleware baseline
	•	observability/logging.py
	•	audit/emitter.py
	•	request middleware에서 baseline log
	•	service/write flow에서 audit candidate emit

장점:
	•	책임 분리가 명확
	•	이후 확장 용이
	•	현재 단계에 적합

옵션 B. 단일 observability service로 통합

장점:
	•	초기 구현이 단순할 수 있음
단점:
	•	운영 로그와 감사 이벤트의 역할이 섞일 수 있음

가능하면 운영 로그 helper와 audit emitter hook을 분리하는 방향을 우선 검토하라.

⸻

7. 산출물 요구

Claude Code는 작업 후 아래 내용을 보고하라.

A. 생성/수정 파일 목록

예:
	•	app/observability/logging.py
	•	app/audit/emitter.py
	•	app/api/context/middleware.py
	•	app/api/errors/handlers.py
	•	app/api/v1/documents.py
	•	app/api/v1/versions.py

B. 도입한 event/log shape 요약

예:
	•	어떤 필드를 공통으로 남기는지
	•	request completion log 구조
	•	audit candidate event 구조
	•	result taxonomy

C. 연결 포인트 요약

예:
	•	middleware에서 무엇을 남기는지
	•	exception handler에서 무엇을 남기는지
	•	documents/versions/idempotency/authz에서 어떤 보강 정보를 남기는지

D. endpoint 적용 예시

간단히:
	•	documents create 성공 시 로그/감사 이벤트
	•	version create 성공 시 로그/감사 이벤트
	•	denied/conflict/error 시 어떤 로그가 남는지

E. 설계 판단 근거

짧게 정리:
	•	왜 structured logging helper를 썼는지
	•	왜 audit emitter를 분리했는지
	•	왜 request body 전체를 로그에 남기지 않았는지
	•	왜 result taxonomy를 이 수준으로 제한했는지

F. 남겨둔 TODO

예:
	•	audit persistence backend 연결 예정
	•	metrics/tracing backend 연결 예정
	•	operation_id/webhook correlation 확장 예정
	•	대시보드/집계 규칙 정의 예정

⸻

8. 완료 조건

아래를 만족하면 완료로 본다.
	•	request completion/failure에 대한 공통 logging baseline이 존재한다.
	•	structured log/event shape가 정의되어 있다.
	•	actor / tenant / resource / action correlation field를 담을 수 있다.
	•	exception handler와 observability가 연결되어 있다.
	•	audit event emit hook point가 존재한다.
	•	documents/versions/idempotency/authz 흐름 일부에 실제 연결되어 있다.
	•	민감 정보 로깅 안전선이 정리되어 있다.
	•	이후 operations/webhook/metrics/tracing 확장을 막지 않는 구조다.

⸻

9. 금지 사항

이번 작업에서는 다음을 하지 마라.
	•	full logging platform 구축까지 하려 하지 말 것
	•	request body 전체를 로깅하지 말 것
	•	auth token/secret을 로그에 남기지 말 것
	•	라우터마다 print/logging을 흩뿌리지 말 것
	•	운영 로그와 감사 이벤트를 완전히 같은 개념으로 취급하지 말 것
	•	metrics/tracing backend까지 이번 단계에 억지로 완성하려 하지 말 것

⸻

10. Claude Code 최종 지시문

아래 지시문을 그대로 사용해도 된다.

Task I-10을 수행하라.

목표:
- 플랫폼 API에 audit / observability의 최소 연결 포인트를 구현한다.
- request_id / trace_id / actor / tenant / action / resource 중심의 공통 관측 구조를 만든다.
- request completion/failure logging baseline과 audit event emit hook point를 마련한다.
- documents/versions/idempotency/authz 흐름 일부에 실제로 연결해 운영 추적 가능성을 확보한다.

반드시 지킬 원칙:
- 이번 단계는 full logging/audit platform 구현이 아니라 공통 관측 포인트를 세우는 단계다.
- structured logging을 우선하고, ad-hoc 문자열 로그를 흩뿌리지 마라.
- request body 전체, auth token, 민감 metadata를 로그에 남기지 마라.
- 운영 로그와 감사 이벤트의 역할을 구분하라.
- 이후 metrics/tracing/operations/webhook 확장을 막지 않도록 correlation field 중심으로 설계하라.

수행 항목:
1. 현재 logging/추적 방식을 점검한다.
2. 공통 event/log shape를 정의한다.
3. request lifecycle logging 포인트를 정한다.
4. structured logger/helper를 구현한다.
5. middleware 또는 동등 위치에 request completion/failure logging을 연결한다.
6. exception handler와 failure logging을 연결한다.
7. action/resource correlation field를 반영한다.
8. audit event emit interface 또는 hook point를 정의한다.
9. documents/versions/idempotency/authz 흐름 일부에 시범 적용한다.
10. 민감 정보 로깅 안전선을 정리하고, 최소 검증 시나리오를 점검한다.

산출물:
- 생성/수정 파일 목록
- event/log shape 요약
- 연결 포인트 요약
- endpoint 적용 예시
- 설계 판단 근거
- 남겨둔 TODO 목록