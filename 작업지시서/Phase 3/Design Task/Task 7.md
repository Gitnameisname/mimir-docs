Task 3-7. Idempotency 및 재시도 안전성 설계

Claude Code 작업 지시서

1. 작업명

Phase 3 - Task 3-7. Idempotency 및 재시도 안전성 설계

⸻

2. 작업 목적

플랫폼 API에서 중복 요청, 네트워크 재시도, 클라이언트 타임아웃, 외부 시스템 재전송 상황이 발생해도 안전하게 동작할 수 있도록, idempotency 및 retry safety 설계 기준을 문서로 정리하라.

이번 작업의 목적은 단순히 Idempotency-Key 헤더를 쓰자는 수준이 아니다.
핵심은 다음이다.
	•	어떤 API가 중복 실행에 취약한지 식별
	•	어떤 요청에 idempotency를 요구하거나 권장할지 기준 수립
	•	같은 요청의 재시도와 “유사하지만 다른 요청”을 어떻게 구분할지 정리
	•	생성, 상태 전이, bulk 작업, 비동기 job 시작, 외부 연동 호출 등에서 중복 부작용을 방지
	•	내부 UI, 외부 시스템, AI agent, automation workflow가 모두 예측 가능한 재시도 의미를 갖게 함

즉, 이 문서는 플랫폼 API의 재시도 안전성 계약 문서여야 한다.

⸻

3. 반드시 반영할 배경

이 플랫폼은 다음 성격을 가진다.
	•	범용 문서 플랫폼
	•	REST 중심 API 구조
	•	문서/버전/노드/권한/감사 추적/이벤트/웹훅/jobs/AI 연계 기능 존재
	•	User/Admin UI, 외부 시스템, AI/Agent, 내부 서비스가 모두 API 소비자
	•	네트워크 장애, 타임아웃, 중복 클릭, 재전송, automation retry가 현실적으로 발생할 수 있음
	•	감사 추적과 행위 추적이 중요
	•	일부 작업은 비동기 job 또는 상태 전이 형태를 가질 수 있음

이전 Task에서 이미 다음이 정해졌다고 가정하라.
	•	플랫폼 API는 계약 계층이다.
	•	리소스 중심 REST 구조와 action endpoint 기준이 존재한다.
	•	인증/인가 보안 문맥이 있다.
	•	버전 전략이 있다.
	•	공통 응답 포맷과 오류 응답 구조가 있다.
	•	목록 조회 규약이 정해졌다.

따라서 이번 작업은 단순 백엔드 내부 구현 문제가 아니라,
API 계약 차원에서 재시도 의미를 명확히 하는 작업이어야 한다.

⸻

4. 이번 작업의 핵심 질문

이번 문서에서는 최소한 아래 질문들에 답할 수 있어야 한다.
	1.	이 플랫폼에서 어떤 요청이 중복 실행 위험이 큰가?
	2.	HTTP method 자체의 의미와 실제 비즈니스 idempotency는 어떻게 구분해야 하는가?
	3.	POST 요청 중 어떤 것은 idempotency가 필요하고, 어떤 것은 덜 중요한가?
	4.	Idempotency-Key는 어떤 요청에서 요구 또는 권장되어야 하는가?
	5.	같은 키로 다른 payload를 보내면 어떻게 처리해야 하는가?
	6.	응답 재사용은 어디까지 허용할 것인가?
	7.	비동기 job 시작, publish/archive 같은 action endpoint, bulk 작업은 어떻게 다뤄야 하는가?
	8.	외부 시스템, 웹훅 재전송, AI agent automation의 재시도 패턴을 어떻게 수용할 것인가?
	9.	idempotency와 optimistic concurrency/version check는 어떤 관계인가?
	10.	감사 로그, request tracing, security context와 idempotency는 어떻게 연결되어야 하는가?

⸻

5. 상세 작업 요구사항

5-1. 문서 작성

아래 산출물을 작성하라.

필수 산출물
	•	Task7_idempotency_strategy.md

문서에 반드시 포함할 섹션
	1.	문서 목적
	2.	Idempotency가 필요한 이유
	3.	상위 설계 원칙
	4.	HTTP method 의미와 비즈니스 idempotency의 관계
	5.	Idempotency 적용 대상 분류
	6.	Idempotency-Key 사용 원칙
	7.	동일 요청 / 상이 요청 충돌 처리 원칙
	8.	리소스 생성 / 상태 전이 / bulk / async job에 대한 적용 전략
	9.	외부 시스템 / 웹훅 / automation / AI agent 재시도 관점
	10.	응답 재사용 및 replay 응답 원칙
	11.	감사 추적 / request context / observability 연계
	12.	concurrency control과의 관계
	13.	후속 Task 및 구현 Phase에 전달할 기준
	14.	결론

⸻

5-2. Idempotency가 필요한 이유 정리

문서 초반에서 다음을 명확히 설명하라.
	•	네트워크 타임아웃 후 클라이언트는 요청 성공 여부를 모를 수 있다.
	•	사용자는 버튼을 여러 번 누를 수 있다.
	•	외부 시스템은 동일 이벤트를 재전송할 수 있다.
	•	webhook delivery나 job trigger는 재시도될 수 있다.
	•	AI agent/automation은 실패 시 자동 재시도 전략을 쓸 수 있다.
	•	중복 요청이 문서 중복 생성, 버전 중복 생성, 상태 전이 중복 실행, 중복 감사 이벤트 발생을 유발할 수 있다.

즉, idempotency는 부가 기능이 아니라
실제 운영 환경의 불확실성을 견디기 위한 필수 안전장치라는 점이 드러나야 한다.

⸻

5-3. 상위 설계 원칙 정의

최소한 아래 원칙을 포함하라.
	•	Explicit retry safety
	•	Side-effect awareness
	•	Same intent, same result
	•	Default caution for non-idempotent operations
	•	Client-assisted idempotency for unsafe writes
	•	Conflict visibility for key reuse mismatch
	•	Auditable replay behavior
	•	Idempotency is not a substitute for authorization
	•	Idempotency is not a substitute for concurrency control
	•	Minimal surprise for clients

각 원칙마다 아래를 설명하라.
	•	의미
	•	왜 필요한지
	•	실제 API 설계에서 어떤 결정을 유도하는지

예:
	•	Same intent, same result → 동일한 의도를 가진 재시도는 가능한 한 같은 결과로 수렴해야 함
	•	Conflict visibility for key reuse mismatch → 같은 키에 다른 요청이 붙으면 조용히 처리하지 말고 명시적으로 충돌 처리
	•	Auditable replay behavior → 재사용 응답인지, 최초 처리인지 추적 가능해야 함

⸻

5-4. HTTP method 의미와 비즈니스 idempotency 관계 정리

반드시 아래를 구분해서 설명하라.
	•	HTTP 스펙 차원의 idempotent method
	•	GET
	•	PUT
	•	DELETE
	•	HEAD 등
	•	비즈니스 차원의 idempotent behavior
	•	POST도 설계에 따라 idempotent하게 만들 수 있다는 점
	•	PATCH는 부분 변경이라도 비즈니스 의미상 중복 적용 위험이 다를 수 있다는 점

문서에 포함해야 할 핵심 내용:
	•	HTTP method의 일반 의미만 믿고 충분하다고 보면 안 된다.
	•	예를 들어 DELETE는 두 번 호출해도 같은 상태에 도달할 수 있지만, 감사/이벤트 관점에서는 중복 부작용이 생길 수 있다.
	•	PUT/PATCH도 내부 구현에 따라 부작용이 달라질 수 있다.
	•	따라서 method semantics와 side-effect semantics를 함께 봐야 한다.

⸻

5-5. Idempotency 적용 대상 분류

플랫폼 API의 요청 유형을 분류하고, 각 유형에 대해 idempotency 필요성을 평가하라.

최소한 다음 유형을 검토하라.

생성 계열
	•	문서 생성
	•	버전 생성
	•	첨부 업로드 등록
	•	webhook 등록
	•	job 시작 요청

변경 계열
	•	문서 수정
	•	권한 변경
	•	태그/라벨 변경
	•	상태 전이
	•	publish / archive / restore / validate / clone 등 action endpoint

삭제 계열
	•	삭제 / detach / revoke

조회 계열
	•	GET / list / search

비동기 시작 계열
	•	reindex
	•	ingestion start
	•	export start
	•	regenerate preview
	•	retry job
	•	resend webhook

각 항목에 대해 다음을 정리하라.
	•	기본적으로 안전한지 / 주의가 필요한지
	•	Idempotency-Key가 필요한지 / 권장인지 / 불필요한지
	•	왜 그런지

⸻

5-6. Idempotency-Key 사용 원칙

이 부분은 핵심이다.
문서에서는 최소한 아래를 정리하라.
	•	Idempotency-Key를 어떤 위치에 둘지
	•	일반적으로 헤더를 권장할지
	•	어떤 요청에서 필수인지
	•	어떤 요청에서 권장인지
	•	어떤 요청에서는 오히려 불필요한지
	•	키는 클라이언트가 생성하는지
	•	키의 의미는 “요청 동일성”이 아니라 “사용자 의도 단위”에 가깝다는 점
	•	키 범위를 actor/tenant/endpoint 단위와 어떻게 연결해 생각할지
	•	키 보관 기간(TTL) 개념을 둘지 여부
	•	영구 보관이 아니라 replay window를 두는 방향 검토

중요:
구현 세부 저장소 설계는 하지 말되,
API 계약과 운영 원칙 수준에서는 충분히 구체적으로 써라.

⸻

5-7. 동일 요청 / 상이 요청 충돌 처리 원칙

반드시 다음을 다뤄라.

같은 key + 같은 intent/payload
	•	동일 응답 재사용 가능 여부
	•	최초 처리 결과를 재반환할지
	•	replay indicator를 제공할지

같은 key + 다른 payload
	•	왜 위험한지
	•	충돌로 간주해야 하는지
	•	조용히 덮어쓰지 말아야 하는 이유
	•	어떤 오류 범주로 처리할지 방향 제시

같은 key + 동일 payload지만 다른 actor/tenant
	•	별도 범위로 취급해야 하는지
	•	보안상 섞이면 안 된다는 점

중요:
“같은 키면 무조건 같은 결과”가 아니라
키의 스코프와 payload 일관성 검증이 필요하다는 점을 문서화하라.

⸻

5-8. 리소스 생성에 대한 적용 전략

다음을 반드시 정리하라.
	•	문서 생성과 같은 POST create는 대표적인 idempotency 적용 대상
	•	중복 생성 방지 관점에서 강한 필요성이 있음
	•	생성 성공 후 클라이언트가 응답을 받지 못했을 때 재시도 가능한 구조가 중요함
	•	생성 결과로 새 resource id가 나오는 경우 replay 시 동일 resource를 참조하도록 해야 하는지 검토
	•	클라이언트 제공 natural key와 idempotency key의 차이
	•	create-if-not-exists 패턴과 idempotency를 혼동하지 않도록 설명

⸻

5-9. 상태 전이 / action endpoint 적용 전략

다음을 다뤄라.
	•	publish
	•	archive
	•	restore
	•	validate
	•	reindex
	•	regenerate-preview
	•	clone
	•	resend-webhook
	•	retry-job

각 항목에 대해 검토할 것:
	•	상태 전이형 action은 이미 목표 상태에 도달해 있으면 어떻게 볼지
	•	부수 효과가 있는 action은 단순 상태 비교만으로 충분한지
	•	같은 action 재시도 시 동일 job/resource를 재사용할 수 있는지
	•	어떤 action은 strict idempotent보다 “at-most-once intent”가 더 중요할 수 있다는 점

중요:
action endpoint를 CRUD와 동일하게 다루지 말고,
상태 전이 + 부수 효과 관점으로 따로 정리하라.

⸻

5-10. Bulk 작업 및 비동기 job 시작 적용 전략

다음을 반드시 포함하라.
	•	bulk permission update
	•	bulk tag update
	•	bulk archive
	•	bulk delete
	•	export job start
	•	index/reindex job start
	•	ingestion job start

정리할 내용:
	•	bulk 요청은 부분 성공/실패가 섞일 수 있어 더 복잡함
	•	idempotency를 bulk 전체 intent에 대해 적용할지
	•	응답 재사용 시 부분 성공 결과를 그대로 재반환할 수 있는지
	•	비동기 job은 중복 생성 방지가 핵심인지
	•	replay 시 같은 job id를 재사용할 수 있는지 방향 제시
	•	일부 bulk 작업은 key 없이 재시도하면 위험하다는 점

⸻

5-11. 외부 시스템 / 웹훅 / automation / AI agent 관점

이 플랫폼 특성상 중요하다.
다음을 반드시 다뤄라.

외부 시스템
	•	네트워크 불안정 때문에 재시도가 흔함
	•	문서 생성/동기화/상태 업데이트 요청에서 idempotency 중요

웹훅 수신
	•	동일 이벤트 재전송 가능성
	•	이벤트 id와 idempotency를 어떻게 관계 지을지 원칙 수준에서 설명

automation / workflow agent
	•	자동 재시도 로직이 붙을 수 있음
	•	사람이 아닌 소비자는 더 엄격한 retry safety가 필요할 수 있음

AI / Agent
	•	동일 작업을 재호출할 가능성
	•	tool invocation 결과가 불확실할 때 중복 호출 위험 존재
	•	idempotent-friendly API가 agent integration에 유리함

중요:
이 문서는 AI/automation을 특수 예외가 아니라
주요 API 소비자로 간주해야 한다.

⸻

5-12. 응답 재사용 및 replay 응답 원칙

Task 3-5와 연결해서 다음을 정리하라.
	•	재시도 요청이 성공적으로 deduplicate 되면 최초 응답을 재사용할 수 있는지
	•	status code를 그대로 유지할지
	•	meta에 replay 여부를 표시할지
	•	header에 replay indicator를 둘지
	•	응답이 너무 오래된 경우 처리 방향
	•	일부 작업은 “결과 동일 + 메타만 갱신 없음”이 적절한지

중요:
클라이언트 입장에서는 replay 여부를 알 수 있으면 유용할 수 있지만,
계약을 지나치게 복잡하게 만들지 않도록 균형 있게 정리하라.

⸻

5-13. 감사 추적 / request context / observability 연계

다음을 정리하라.
	•	idempotency key는 request context 일부로 추적 가능해야 함
	•	actor, tenant, endpoint, request_id, idempotency_key를 함께 남길 필요
	•	replay 처리 여부도 감사 또는 운영 로그에서 구분 가능해야 함
	•	중복 요청 공격/오용 탐지에도 도움이 될 수 있음
	•	보안상 외부 응답에 과도한 내부 정보를 노출하지 않아야 함

즉, idempotency는 단순 dedupe 기능이 아니라
운영 추적성과 연결된 계약 요소라는 점을 반영하라.

⸻

5-14. Concurrency control과의 관계

이 부분을 반드시 분리해서 설명하라.
	•	idempotency는 “같은 요청의 재시도 안전성” 문제
	•	concurrency control은 “동시에 다른 변경이 충돌하는 문제”
	•	optimistic locking / version check / ETag / precondition 계열은 별도 관심사
	•	둘은 함께 필요할 수 있지만 서로 대체재가 아님

예:
	•	같은 create 요청 재시도는 idempotency로 다룸
	•	같은 문서를 두 사용자가 동시에 수정하는 문제는 concurrency control 문제

이 구분을 명확히 써라.

⸻

5-15. 후속 Task 및 구현 Phase에 전달할 기준

문서 마지막에 다음 연결 포인트를 정리하라.

예:
	•	Task 3-8에서는 async job / webhook delivery / event trigger에 대한 중복 방지 구조를 더 구체화해야 함
	•	Task 3-9에서는 AI/agent tool invocation에 적합한 retry-safe 인터페이스 관점을 반영해야 함
	•	Task 3-10에서는 idempotency mismatch/replay/conflict에 대한 error model을 구체화해야 함
	•	구현 Phase에서는 request middleware, persistence layer, response meta, audit logging에 직접 반영해야 함
	•	이후 concurrency control 설계가 별도로 필요하다면 분리해서 다뤄야 함

⸻

6. 산출물

아래 파일을 작성하라.

필수 산출물
	•	Task7_idempotency_strategy.md

문서 성격
	•	설계 기준 문서
	•	구현 코드 금지
	•	저장소/DB 상세 설계 금지
	•	프레임워크 종속 설명 최소화
	•	API 계약과 운영 원칙 중심

⸻

7. 제외 범위

이번 작업에서는 아래는 하지 마라.
	•	실제 Idempotency-Key 저장 테이블 설계
	•	Redis/DB/캐시 구현 상세
	•	middleware/decorator 코드 작성
	•	해시 알고리즘 상세 선택
	•	payload canonicalization 세부 알고리즘 구현
	•	optimistic locking 상세 설계
	•	webhook dedupe 저장소 구현
	•	retry/backoff 알고리즘 상세 설계
	•	bulk partial success 응답 스키마 상세 구현
	•	보안 위협 모델 전체 작성

즉, 이번 작업은 플랫폼 API의 idempotency 및 retry safety 상위 기준 수립에 집중하라.

⸻

8. 작성 스타일 가이드

문서는 아래 스타일을 따르라.
	•	운영 현실을 반영한 실용적 문서로 작성
	•	HTTP method 의미와 비즈니스 안전성을 구분해서 설명
	•	create/action/bulk/async/job/webhook/AI agent를 각각 충분히 고려
	•	권장 / 필수 / 선택 / 비권장 / 금지 / 후속 Task에서 구체화를 구분
	•	구현 세부 대신 계약 의미와 정책 기준에 집중
	•	예시는 넣되 문서가 구현 명세서가 되지 않도록 할 것

⸻

9. 검토 체크리스트

작업 완료 전에 아래를 스스로 점검하라.
	•	왜 idempotency가 필요한지 충분히 설명했는가?
	•	HTTP method와 비즈니스 idempotency를 구분했는가?
	•	어떤 요청에 key가 필요한지 분류했는가?
	•	같은 key + 다른 payload 충돌 처리가 명확한가?
	•	create / action / bulk / async job 관점이 모두 포함되었는가?
	•	외부 시스템 / 웹훅 / automation / AI agent 관점이 포함되었는가?
	•	replay 응답과 observability 연계가 설명되었는가?
	•	concurrency control과의 차이를 분명히 했는가?
	•	후속 Task와 구현 단계가 이 문서를 기준으로 이어질 수 있는가?

⸻

10. Claude Code에 대한 최종 지시

위 요구사항을 반영하여 Task7_idempotency_strategy.md 문서를 작성하라.

문서는 범용 문서 플랫폼 API에서 발생할 수 있는 중복 요청, 재시도, 타임아웃, 자동화 재호출 상황을 안전하게 처리하기 위한
idempotency 및 retry safety의 상위 설계 기준 문서여야 한다.

문서 작성 후에는 아래 형식의 자체 점검 요약도 함께 정리하라.

자체 점검 요약
	•	idempotency 적용이 중요한 요청 유형 요약
	•	Idempotency-Key 사용 원칙 요약
	•	same key / different payload 충돌 처리 요약
	•	create / action / bulk / async job 적용 관점 요약
	•	외부 시스템 / 웹훅 / AI agent 관련 관점 요약
	•	concurrency control과의 차이 요약
	•	후속 Task 연결 포인트
	•	의도적으로 후속 Task로 남긴 미결정 사항