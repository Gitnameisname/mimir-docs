Task I-9. write endpoint idempotency hook 구현

Claude Code 작업 계획서

1. 작업 목표

중복 생성, 네트워크 재시도, 클라이언트 타임아웃 이후 재요청 같은 상황에서도 write endpoint가 재시도 안전성을 가질 수 있도록, 플랫폼 API에 idempotency hook 구조를 구현한다.

이번 단계의 핵심은 완전한 분산 락 시스템이나 고도화된 replay 엔진을 만드는 것이 아니다.
핵심은 다음을 실제 코드 구조로 세우는 것이다.
	•	Idempotency-Key를 수용하는 공통 흐름
	•	selected write endpoint에서 idempotency hook 호출
	•	같은 키 재요청 시 replay 가능 구조
	•	같은 키인데 요청 의미가 다를 때 conflict/mismatch 처리
	•	request context / actor / resource / payload fingerprint와 연결 가능한 구조
	•	audit / observability / async operation으로 확장 가능한 기반

즉, 이번 Task는 write API의 재시도 안전성에 대한 최소 구현 기반을 세우는 작업이다.

⸻

2. 작업 범위

포함 범위
	•	idempotency key 처리 정책 정의
	•	공통 idempotency record 모델 또는 동등 구조 설계
	•	request fingerprint 또는 payload signature 계산 방식 초안
	•	selected POST/PATCH endpoint에 hook 연결
	•	same key replay 흐름 초안 구현
	•	key mismatch/conflict 처리 초안 구현
	•	공통 오류 응답과 연결
	•	request context / actor / resource correlation 반영
	•	향후 persistence/audit/operation 확장을 위한 경계 마련

제외 범위
	•	모든 write endpoint 일괄 적용
	•	분산 환경에서의 완전한 원자성 보장
	•	고도화된 상태 머신 구현
	•	long-running async operation과의 완전 통합
	•	full replay body 캐시 최적화
	•	cross-region consistency 보장
	•	idempotency retention policy 완성
	•	full admin/debug tooling

⸻

3. 구현 의도

write endpoint는 운영 환경에서 가장 쉽게 깨지는 부분 중 하나다.

예를 들어:
	•	클라이언트가 POST /documents를 보낸 뒤 응답을 못 받아 재전송
	•	POST /documents/{id}/versions 중간에 네트워크 끊김
	•	사용자가 UI에서 버튼을 두 번 누름
	•	reverse proxy나 client SDK가 자동 retry 수행

이때 idempotency가 없으면,
	•	문서가 중복 생성되고
	•	버전이 중복 생성되며
	•	감사 로그와 사용자 경험이 모두 망가지게 된다.

따라서 이번 작업의 목적은 다음이다.
	•	“같은 의도”의 재시도를 같은 결과로 돌려줄 수 있는 기반 마련
	•	write endpoint가 중복 생성에 취약하지 않게 구조화
	•	향후 async operations / webhook / audit / observability와 자연스럽게 연결
	•	idempotency logic이 라우터마다 제각각 박히지 않게 공통화

⸻

4. Claude Code에게 요청할 작업

아래 작업을 순서대로 수행하라.

⸻

Step 1. 현재 write endpoint 흐름 점검

현재 구현된 write endpoint를 먼저 점검하라.

확인 대상:
	•	POST /api/v1/documents
	•	PATCH /api/v1/documents/{document_id}
	•	POST /api/v1/documents/{document_id}/versions

확인할 항목:
	•	write 요청이 어떤 request body를 받는지
	•	actor/context/authz 흐름이 어디서 결정되는지
	•	service 진입점이 어디인지
	•	응답 shaping이 어디서 이뤄지는지
	•	idempotency hook을 넣기 가장 자연스러운 위치가 어디인지

목적:
	•	idempotency를 나중에 덧붙이는 임시 해킹이 아니라,
현재 write 흐름 안에 공통 계층으로 삽입하기 위함이다.

⸻

Step 2. idempotency 적용 대상 endpoint 선정

이번 단계에서 어떤 endpoint에 우선 적용할지 명확히 정하라.

권장 최소 대상:
	•	POST /api/v1/documents
	•	POST /api/v1/documents/{document_id}/versions

선택 적용 검토:
	•	PATCH /api/v1/documents/{document_id}

권장 판단:
	•	생성 계열 POST를 우선 적용
	•	PATCH는 멱등 의미가 상대적으로 애매할 수 있으므로,
이번 단계에서는 hook point만 두고 실제 강제는 선택적으로 해도 된다

중요:
	•	모든 write에 억지로 붙이지 말 것
	•	중복 생성 피해가 큰 endpoint부터 시작하라

⸻

Step 3. Idempotency-Key 처리 정책 정의

Idempotency-Key 헤더 처리 정책을 정하라.

최소 정책:
	•	헤더 이름: Idempotency-Key
	•	selected write endpoint에서 수용
	•	없을 경우:
	•	정책 A: 그냥 일반 write 수행
	•	정책 B: 특정 endpoint에서는 required
	•	이번 단계에서는 optional but supported가 현실적일 가능성이 높음

검토 항목:
	•	공백/과도한 길이/비정상 문자 검증
	•	최대 길이 제한
	•	빈 문자열 거부
	•	헤더 normalization 여부

중요:
	•	정책은 단순하고 예측 가능해야 한다
	•	클라이언트가 “같은 의도 요청에 같은 키를 재사용”할 수 있어야 한다

⸻

Step 4. idempotency record 모델 설계

idempotency 상태를 저장하거나 표현하기 위한 record 구조를 설계하라.

권장 최소 필드:
	•	idempotency_key
	•	actor_id 또는 actor identity key
	•	resource_action
	•	request_fingerprint
	•	status
	•	예: in_progress, completed, failed
	•	response_status_code optional
	•	response_body optional
	•	resource_id optional
	•	created_at
	•	updated_at
	•	request_id optional
	•	trace_id optional

검토 가능 추가 필드:
	•	tenant_id
	•	endpoint path
	•	method
	•	expires_at
	•	error_code

중요:
	•	이번 단계는 full production schema 완성이 아니라,
재시도 안전 흐름을 표현할 수 있는 최소 모델이면 된다

⸻

Step 5. request fingerprint 계산 정책 정리

같은 Idempotency-Key로 들어온 요청이 “같은 요청”인지 판단하기 위한 fingerprint 정책을 정하라.

권장 입력 요소:
	•	HTTP method
	•	canonical endpoint/action
	•	normalized request body
	•	relevant path params
	•	actor identity
	•	tenant scope(optional)

권장 결과:
	•	stable hash 문자열

중요:
	•	같은 키 + 같은 의미의 요청이면 fingerprint 동일
	•	같은 키 + 다른 body면 conflict/mismatch 처리
	•	JSON body는 key order 등으로 인해 의미는 같지만 문자열이 달라질 수 있으므로,
가능하면 정규화된 serialization을 검토하라

주의:
	•	fingerprint에 너무 많은 ephemeral 값을 넣지 말 것
	•	request_id, trace_id 같은 요청마다 바뀌는 값은 넣지 말 것

⸻

Step 6. idempotency service 또는 manager 계층 구현

라우터마다 키 검사 로직을 쓰지 말고, 공통 idempotency service/manager를 구현하라.

예시 역할:
	•	key 검증
	•	fingerprint 생성
	•	기존 record 조회
	•	same-key same-request 판정
	•	same-key different-request conflict 판정
	•	replay 가능한 경우 이전 응답 반환
	•	신규 요청이면 reservation 또는 record 생성

권장 개념:
	•	IdempotencyService
	•	IdempotencyManager
	•	IdempotencyCoordinator

중요:
	•	라우터가 직접 저장소를 뒤지지 말 것
	•	documents / versions write endpoint가 같은 패턴으로 사용 가능해야 한다

⸻

Step 7. in-progress / completed / mismatch 흐름 정의

idempotency state 전이를 최소 수준으로 정의하라.

권장 흐름:

신규 키
	•	record 없음
	•	새 record 생성
	•	in_progress
	•	write 수행
	•	성공 시 completed + response snapshot 저장

동일 키 재요청, 동일 fingerprint
	•	record completed
	•	기존 response replay

동일 키 재요청, 다른 fingerprint
	•	conflict/mismatch 오류 반환

동일 키, 아직 in_progress
	•	정책 검토:
	•	409 conflict
	•	또는 202 accepted-like
	•	이번 단계에서는 단순하게 conflict 또는 retryable error로 처리해도 된다

중요:
	•	상태 모델은 단순해야 한다
	•	하지만 향후 async operations와 연결할 수 있어야 한다

⸻

Step 8. replay response 구조 정의

이미 완료된 요청에 대해 같은 키가 다시 오면 어떤 응답을 돌려줄지 정하라.

권장 방향:
	•	원래 response status code 재사용
	•	원래 success envelope 재사용
	•	meta에 replay 여부를 담을 확장 슬롯 검토

예:
	•	meta.idempotency = { replayed: true }
	•	또는 이후 확장을 위해 자리만 마련

중요:
	•	replay 응답은 가능한 한 원래 응답과 의미적으로 동일해야 한다
	•	단, 이번 단계에서는 meta에 작은 표시를 남길지 여부는 선택 가능
	•	공통 response helper와 충돌하지 않게 설계하라

⸻

Step 9. mismatch/conflict 오류 흐름 연결

같은 Idempotency-Key에 서로 다른 요청이 들어오면 공통 오류 응답으로 처리하라.

권장 error code 예:
	•	idempotency_key_conflict
	•	idempotency_key_mismatch

권장 HTTP status:
	•	409 Conflict

권장 message:
	•	안전하고 간결하게
	•	내부 fingerprint 값 자체를 그대로 노출하지는 말 것

예시 느낌:
	•	"Idempotency key was already used for a different request"

중요:
	•	Task I-3 공통 오류 모델을 사용할 것
	•	라우터에서 임의 JSON 에러를 만들지 말 것

⸻

Step 10. persistence 저장 위치 및 경계 결정

idempotency record를 어디에 저장할지 결정하라.

검토 후보:
	•	기존 DB
	•	별도 table
	•	임시 in-memory/redis-like placeholder
	•	현재 코드베이스에 맞는 최소 저장소

권장:
	•	운영적으로 replay 가능해야 하므로 process memory 단독은 지양
	•	가능하면 persistence layer에 별도 repository/table 경계 마련
	•	단, 초기 skeleton 단계라면 저장소 구현을 단순화할 수 있음

중요:
	•	이번 단계는 durability와 구조가 우선이다
	•	나중에 TTL/cleanup/retention을 붙일 수 있어야 한다

⸻

Step 11. transaction / write completion 연결 지점 검토

idempotency record와 실제 write 성공 완료를 언제 연결할지 검토하라.

최소 요구:
	•	write 시작 전 record 확보
	•	write 성공 후 completed 반영
	•	실패 시 상태 반영 정책 정리

검토 포인트:
	•	write 실패 시 record를 지울지
	•	failed 상태로 남길지
	•	retriable failure와 terminal failure를 구분할지

권장 초기 방향:
	•	구현 복잡도를 낮추기 위해 단순 정책을 택하되,
TODO로 정교화 지점을 남겨라

중요:
	•	이번 단계에서 완전한 원자성을 보장하지 못하더라도,
구조적으로 나중에 정교화 가능한 경계를 만들어야 한다

⸻

Step 12. selected write endpoint에 hook 연결

선정한 write endpoint에 실제로 idempotency hook을 연결하라.

최소 적용 권장:
	•	POST /api/v1/documents
	•	POST /api/v1/documents/{document_id}/versions

연결 방식 예:
	•	라우터에서 key 추출
	•	idempotency service 호출
	•	replay면 즉시 반환
	•	신규면 service write 수행
	•	완료 결과 기록

또는
	•	service 계층 진입 전에 reservation
	•	service 완료 후 finalize

중요:
	•	패턴이 documents와 versions create에서 같아야 한다
	•	idempotency 로직이 endpoint마다 다르게 흩어지지 않게 할 것

⸻

Step 13. RequestContext / actor / tenant / audit correlation 연결

idempotency record가 request context와 자연스럽게 연결되게 하라.

반영 권장 항목:
	•	request_id
	•	trace_id
	•	actor_id / actor_type
	•	tenant_id optional
	•	action/resource_action

목표:
	•	이후 Task I-10 audit/observability에서 상관관계를 추적 가능하게 함
	•	누가 어떤 키로 어떤 write를 재시도했는지 추적 가능하게 함

중요:
	•	이 정보가 모두 외부 응답에 노출될 필요는 없다
	•	내부 correlation 목적으로 충분하다

⸻

Step 14. response / error 계층 통합 검토

idempotency 흐름이 기존 공통 계층과 자연스럽게 붙는지 검토하라.

반드시 확인:
	•	replay 응답도 Task I-2 success envelope 사용
	•	mismatch/conflict는 Task I-3 error envelope 사용
	•	request_id/trace_id는 Task I-4 context 사용
	•	actor/resource 흐름은 Task I-5 authz 구조와 충돌하지 않음
	•	write endpoint 자체 contract를 깨지 않음

⸻

Step 15. 향후 async operations / webhook / audit 확장성 검토

이번 단계는 idempotency hook만 구현하지만, 나중에 다음과 연결될 수 있어야 한다.

검토 항목:
	•	long-running operation의 accepted response와 연결 가능한가
	•	webhook delivery 중복 방지와 개념적으로 충돌하지 않는가
	•	audit 로그와 동일 correlation field를 공유할 수 있는가
	•	operation_id / resource_id / request_id 연계 여지를 남겼는가

중요:
	•	지금 다 구현할 필요는 없다
	•	하지만 구조가 막혀 있으면 안 된다

⸻

Step 16. 테스트/검증 시나리오 정리

최소한 아래 시나리오를 검증 가능하게 정리하라.

필수 시나리오:
	1.	같은 key + 같은 body → 두 번째 요청은 replay
	2.	같은 key + 다른 body → conflict
	3.	key 없음 → 일반 write 수행
	4.	invalid key 형식 → validation error
	5.	in-progress 중복 요청 → 정책대로 처리
	6.	documents create와 versions create가 같은 패턴으로 동작

⸻

5. 구현 시 세부 요구사항

5-1. idempotency는 write retry safety 용도

이번 단계의 idempotency는 “클라이언트 재시도 안전성”을 위한 것이다.
권한, 중복 비즈니스 판단, optimistic locking과 혼동하지 말 것.

즉:
	•	business uniqueness와는 다르다
	•	concurrency control 전체와도 다르다
	•	retry safety에 초점을 맞춰라

⸻

5-2. Idempotency-Key는 요청 의도의 식별자

키 그 자체만 보면 안 되고, 같은 키가 같은 요청 의미와 연결되는지를 fingerprint로 함께 봐야 한다.

따라서:
	•	키만 같다고 무조건 replay 금지
	•	body가 다르면 conflict 처리
	•	actor/resource/action 맥락도 고려하라

⸻

5-3. replay는 원래 응답 의미를 유지

replay 응답은 “대충 성공”이 아니라, 원래 성공 응답과 의미적으로 동일해야 한다.

따라서:
	•	status code 유지
	•	data envelope 유지
	•	가능하면 동일 resource id 유지

⸻

5-4. 라우터 하드코딩 금지

idempotency 검사/저장/재생 로직을 라우터마다 복붙하지 말 것.

반드시:
	•	service 또는 manager 계층으로 공통화
	•	documents create와 versions create가 같은 패턴 사용

⸻

5-5. fingerprint는 안정적이어야 함

fingerprint는 요청마다 달라지는 값을 섞으면 안 된다.

넣지 말 것 예:
	•	request_id
	•	trace_id
	•	timestamp
	•	random ordering 영향을 받는 raw dict string

가능하면:
	•	normalized body
	•	stable route/action identity
	•	actor/resource scope

⸻

5-6. 실패 정책은 단순하게 시작

실패한 요청에 대한 재처리 정책은 복잡하다.
이번 단계에서는 지나치게 고도화하지 말고, 구조를 먼저 잡아라.

예:
	•	failed 상태 저장
	•	in_progress conflict
	•	completed replay

이 정도로 시작해도 충분하다.

⸻

5-7. PATCH 적용은 신중히

PATCH는 HTTP semantics상 본질적으로 idempotent하게 설계될 수도 있지만,
현실 구현에서는 partial update semantics와 충돌할 수 있다.

따라서:
	•	이번 단계는 POST 생성류 중심
	•	PATCH는 hook point만 남기거나 선택 적용

이 방향이 더 안전하다.

⸻

6. 권장 설계 방향

Claude Code는 아래 방향을 우선 검토하라.

옵션 A. IdempotencyService + Repository + Response Snapshot
	•	idempotency_service.py
	•	idempotency_repository.py
	•	별도 record 모델
	•	write 전 reserve, write 후 finalize, replay 시 snapshot 반환

장점:
	•	구조가 명확
	•	향후 확장 용이
	•	운영 추적 가능성 높음

옵션 B. 최소 manager + 기존 persistence 임시 재사용

장점:
	•	빠르게 붙일 수 있음
단점:
	•	나중에 정리 비용이 커질 수 있음

가능하면 별도 idempotency 경계를 만드는 방향을 우선 검토하라.

⸻

7. 산출물 요구

Claude Code는 작업 후 아래 내용을 보고하라.

A. 생성/수정 파일 목록

예:
	•	app/services/idempotency_service.py
	•	app/repositories/idempotency_repository.py
	•	app/models/idempotency_record.py
	•	app/api/v1/documents.py
	•	app/api/v1/versions.py

B. 도입한 idempotency 정책 요약

예:
	•	어떤 endpoint에 우선 적용했는지
	•	key가 optional인지 required인지
	•	key 형식/길이 검증 규칙
	•	replay / conflict / in-progress 처리 정책

C. record / fingerprint 구조 요약

예:
	•	어떤 필드를 저장하는지
	•	fingerprint에 무엇을 포함하는지
	•	actor/resource/context와 어떻게 연결하는지

D. endpoint 적용 예시

간단히:
	•	documents create에서 어떻게 동작하는지
	•	versions create에서 어떻게 동작하는지
	•	replay 응답이 어떤 형태인지

E. 설계 판단 근거

짧게 정리:
	•	왜 POST 생성류부터 적용했는지
	•	왜 same-key different-body를 conflict로 처리했는지
	•	왜 fingerprint에 이런 요소를 넣었는지
	•	왜 별도 service/repository 경계를 두었는지

F. 남겨둔 TODO

예:
	•	PATCH/write action 확대 적용 예정
	•	TTL/cleanup 정책 예정
	•	stronger transaction guarantee 예정
	•	async operation 연계 예정
	•	audit/logging 고도화 예정

⸻

8. 완료 조건

아래를 만족하면 완료로 본다.
	•	Idempotency-Key를 처리하는 공통 hook 구조가 존재한다.
	•	selected write endpoint에 실제 적용되어 있다.
	•	same-key same-request replay 흐름이 있다.
	•	same-key different-request conflict 흐름이 있다.
	•	request fingerprint 정책이 정의되어 있다.
	•	idempotency record 또는 동등한 persistence 구조가 존재한다.
	•	공통 response/error/context/auth 계층과 충돌 없이 연결된다.
	•	향후 audit/async/webhook 확장을 막지 않는 구조다.

⸻

9. 금지 사항

이번 작업에서는 다음을 하지 마라.
	•	모든 write endpoint를 한 번에 다 적용하려 하지 말 것
	•	distributed lock/full atomicity를 이번 단계에 완성하려 하지 말 것
	•	idempotency를 business uniqueness 검증과 혼동하지 말 것
	•	라우터에 replay/conflict 로직을 복붙하지 말 것
	•	같은 키인데 다른 요청도 억지로 replay하지 말 것
	•	request_id 같은 변동 값을 fingerprint에 넣지 말 것

⸻

10. Claude Code 최종 지시문

아래 지시문을 그대로 사용해도 된다.

Task I-9를 수행하라.

목표:
- 플랫폼 API의 selected write endpoint에 idempotency hook 구조를 구현한다.
- Idempotency-Key를 통해 retry-safe write 흐름의 최소 기반을 만든다.
- same-key same-request replay, same-key different-request conflict 처리 구조를 구현한다.
- request context / actor / resource / payload fingerprint와 연결 가능한 idempotency 계층을 만든다.

반드시 지킬 원칙:
- 이번 단계는 full distributed idempotency engine 구현이 아니라 retry-safe write hook의 최소 구현 단계다.
- 중복 생성 피해가 큰 POST 생성류 endpoint부터 우선 적용하라.
- 라우터마다 개별 로직을 넣지 말고 공통 service/manager 계층으로 수렴시켜라.
- same-key different-request는 replay하지 말고 conflict로 처리하라.
- request_id 같은 변동 값은 fingerprint에 넣지 마라.
- 이후 audit/observability/async operation 확장을 막지 않도록 구조를 잡아라.

수행 항목:
1. 현재 write endpoint 흐름을 점검한다.
2. idempotency 적용 대상 endpoint를 선정한다.
3. Idempotency-Key 처리 정책을 정한다.
4. idempotency record 모델 또는 동등 구조를 설계한다.
5. request fingerprint 정책을 정한다.
6. IdempotencyService/Manager를 구현한다.
7. 신규/in-progress/completed/mismatch 상태 흐름을 정의한다.
8. replay response와 conflict error 흐름을 공통 response/error 계층에 연결한다.
9. selected write endpoint에 실제 hook을 연결한다.
10. request context / actor / resource correlation을 반영한다.
11. documents create와 versions create에서 패턴이 동일하게 동작하는지 검증한다.

산출물:
- 생성/수정 파일 목록
- idempotency 정책 요약
- record/fingerprint 구조 요약
- endpoint 적용 예시
- 설계 판단 근거
- 남겨둔 TODO 목록