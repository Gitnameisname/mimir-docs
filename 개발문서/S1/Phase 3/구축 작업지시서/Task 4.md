Task I-3. 공통 오류 모델 및 exception handling 계층 구현

Claude Code 작업 계획서

1. 작업 목표

플랫폼 API 전반에서 일관된 오류 응답 계약과 공통 예외 처리 흐름을 갖도록, 오류 모델과 exception handling 계층을 구현한다.

이번 작업의 목적은 단순히 에러가 났을 때 JSON 하나를 반환하는 것이 아니라, 앞으로 모든 API가 아래 기준을 따르도록 만드는 것이다.
	•	오류 응답 형식 표준화
	•	비즈니스 예외와 시스템 예외의 구분
	•	validation / authentication / authorization / not found / conflict / idempotency 계열 예외의 공통 표현
	•	request_id / trace_id 기반 추적 가능성 확보
	•	외부 노출 가능한 정보와 내부 로그용 정보를 분리
	•	이후 audit / observability / idempotency / async operation과 충돌하지 않는 예외 구조 확보

즉, 이번 Task는 플랫폼 차원의 오류 계약과 예외 전달 흐름을 세우는 작업이다.

⸻

2. 작업 범위

포함 범위
	•	공통 error response 모델 정의
	•	error object 구조 정의
	•	business error base class 설계
	•	대표 예외 계층 초안 구현
	•	global exception handler 구현
	•	FastAPI/Starlette 기본 예외와의 연결 정리
	•	request_id / trace_id 반영 가능한 오류 응답 구조 확보
	•	외부 노출 규칙 반영
	•	placeholder endpoint/라우터 환경에서 실제 동작 검증

제외 범위
	•	전체 error code catalog 완성
	•	모든 도메인 예외 상세 구현
	•	audit 로그 완성
	•	observability stack 완성
	•	idempotency replay full logic
	•	DB/외부 시스템 예외 번역 전부 완성
	•	다국어 에러 메시지 체계

⸻

3. 구현 의도

성공 응답 envelope이 통일되어도, 오류 응답이 엔드포인트마다 다르게 나오면 API는 여전히 불안정하다.
특히 플랫폼 API는 UI, 외부 시스템, 자동화 클라이언트, 향후 AI 에이전트까지 사용할 수 있으므로, 오류 계약이 흔들리면 재시도/분기/운영 추적이 모두 어려워진다.

따라서 이번 작업은 다음 목표를 가진다.
	•	오류 응답을 예측 가능하게 만들기
	•	HTTP status와 내부 error code를 함께 사용할 수 있는 구조 만들기
	•	라우터/서비스에서 임의의 HTTPException 남발을 줄이기
	•	내부 구현 세부사항이 외부로 새지 않도록 안전선 만들기
	•	request correlation 기반의 운영 추적 가능성 확보

⸻

4. Claude Code에게 요청할 작업

아래 작업을 순서대로 수행하라.

⸻

Step 1. 현재 예외/오류 처리 방식 점검

현재 코드베이스에서 예외가 어떻게 처리되고 있는지 먼저 점검하라.

확인할 항목:
	•	FastAPI HTTPException 사용 위치
	•	validation error가 현재 어떤 형태로 반환되는지
	•	전역 exception handler가 이미 있는지
	•	middleware 수준에서 예외를 처리하는지
	•	placeholder endpoint에서 현재 오류가 발생하면 어떤 shape로 내려가는지

목적:
	•	기존 구조를 무시하지 않고, 가장 적은 충돌로 공통 오류 계층을 삽입할 위치를 결정하는 것

⸻

Step 2. 공통 error response envelope 정의

성공 응답의 data/meta 구조와 별도로, 오류 응답의 공통 envelope을 정의하라.

권장 방향:

{
  "error": {
    "code": "resource_not_found",
    "message": "Document not found",
    "details": {...}
  },
  "meta": {
    "request_id": "...",
    "trace_id": "..."
  }
}

중요 원칙:
	•	오류 응답 최상위는 error, meta
	•	error.code는 클라이언트 분기용 식별자
	•	error.message는 사람이 읽을 수 있는 안전한 설명
	•	error.details는 validation이나 field-level info 등 확장 공간
	•	성공 응답과 같은 최상위 data 구조를 섞지 말 것

주의:
	•	내부 stack trace, DB raw exception, 민감한 시스템 정보는 외부 응답에 넣지 말 것

⸻

Step 3. error object 모델 설계

공통 error object 모델을 정의하라.

권장 최소 필드:
	•	code: str
	•	message: str
	•	details: Optional[Any]

필요 시 고려 가능:
	•	target
	•	retryable
	•	category

하지만 이번 단계에서는 너무 많이 확정하지 말 것.
우선은 플랫폼 전반에서 가장 안정적으로 재사용 가능한 최소 구조를 우선한다.

권장 원칙:
	•	code는 기계 친화적
	•	message는 외부 노출 가능한 문장
	•	details는 선택적
	•	details는 validation 에러나 filter/sort/cursor 오류 등 구조적 정보 수용 가능

⸻

Step 4. business error base class 구현

서비스 계층과 라우터 계층이 공통으로 사용할 수 있는 비즈니스 예외의 베이스 클래스를 구현하라.

예시 개념:
	•	ApiError 또는 BusinessError
	•	내부 속성:
	•	http_status
	•	error_code
	•	message
	•	details
	•	safe_expose 또는 이에 준하는 개념

중요:
	•	라우터가 직접 HTTPException을 남발하지 않도록, 플랫폼 예외를 통해 표현하는 패턴을 도입할 것
	•	이후 서비스 계층이 이 예외를 raise하고, global handler가 일관된 응답으로 번역하는 구조가 되어야 한다

⸻

Step 5. 대표 예외 계층 초안 구현

아래 계열 예외를 우선 정의하라.

최소 권장 세트:
	•	ValidationError 계열 또는 API용 invalid request 예외
	•	AuthenticationError
	•	PermissionDeniedError 또는 AuthorizationError
	•	NotFoundError
	•	ConflictError
	•	IdempotencyError 또는 IdempotencyConflictError
	•	RateLimitError는 optional
	•	ServiceUnavailableError는 optional

주의:
	•	Python/FastAPI 기본 예외 클래스와 이름 충돌이 심하면 naming을 조정하라
	•	예: ApiValidationError, ApiNotFoundError
	•	모든 예외를 지금 완성하려 하지 말고, 플랫폼 골격 수준의 계층만 잡아라

⸻

Step 6. HTTP status mapping 정리

각 예외가 어떤 HTTP status로 번역되는지 기준을 정리하라.

권장 매핑:
	•	validation → 400
	•	authentication → 401
	•	authorization/permission denied → 403
	•	not found → 404
	•	conflict → 409
	•	idempotency mismatch/conflict → 409 또는 설계상 별도 정책
	•	unsupported operation → 400 또는 422
	•	unexpected internal error → 500

주의:
	•	422 사용 여부는 프로젝트 기준에 맞춰 신중히 결정하라
	•	FastAPI 기본 validation error와의 관계도 고려하라
	•	이번 단계에서는 “완벽한 HTTP semantics”보다 “일관성”이 우선이다

⸻

Step 7. global exception handler 구현

FastAPI application 레벨에서 공통 exception handler를 구현하라.

최소 처리 대상:
	•	플랫폼 business error base class
	•	FastAPI/Starlette HTTPException
	•	request validation 관련 예외
	•	예상치 못한 일반 Exception

핵심 요구:
	•	모든 경로에서 가능한 한 동일한 오류 envelope로 응답할 것
	•	내부 예외는 안전한 generic message로 변환할 것
	•	request_id / trace_id를 meta에 넣을 수 있게 설계할 것
	•	예외 종류에 따라 적절한 status code를 유지할 것

주의:
	•	일반 Exception handler는 내부 정보를 숨기되, 로그와 응답을 분리해서 다룰 수 있게 설계하라
	•	디버그 여부에 따라 메시지 노출을 달리할 수 있는 여지는 남겨도 되지만, 기본은 production-safe여야 한다

⸻

Step 8. FastAPI validation error shape 정리

FastAPI/Pydantic의 request validation 오류가 기본 형식으로 흩어지지 않도록 공통 envelope로 감싸라.

목표:
	•	잘못된 body/query/path 파라미터가 들어왔을 때도 일관된 오류 응답을 반환
	•	field-level 정보가 있으면 details에 담을 수 있도록 설계

예시 방향:

{
  "error": {
    "code": "validation_error",
    "message": "Request validation failed",
    "details": [
      {
        "field": "page_size",
        "reason": "must be <= 100"
      }
    ]
  },
  "meta": {
    "request_id": "...",
    "trace_id": "..."
  }
}

중요:
	•	FastAPI 기본 detail payload를 그대로 노출하기보다, 플랫폼 규약에 맞춰 번역할 것
	•	다만 디버깅에 필요한 최소 정보는 남겨야 한다

⸻

Step 9. 안전한 외부 노출 규칙 반영

오류 응답에서 외부로 노출 가능한 정보와 내부 정보의 경계를 정리하라.

반드시 지킬 점:
	•	stack trace 금지
	•	raw DB 오류 메시지 금지
	•	내부 클래스명/경로/SQL 직접 노출 금지
	•	인증/권한 실패 시 민감한 내부 판단 로직 노출 금지

허용 가능한 것:
	•	안전한 요약 메시지
	•	validation field 정보
	•	request correlation ID
	•	클라이언트가 재시도/분기 판단에 쓸 수 있는 code

필요하면 내부용 로그 메시지와 외부용 message를 분리할 수 있는 구조를 마련하라.

⸻

Step 10. 공통 error/helper 코드 배치 위치 정리

오류 모델과 예외 클래스를 어디에 둘지 정리하라.

권장 예시:

app/
  api/
    errors/
      __init__.py
      models.py
      exceptions.py
      handlers.py

또는 기존 구조에 맞게:

app/common/api/errors/
app/shared/http/errors/

중요 원칙:
	•	특정 도메인(router/documents) 내부에 두지 말 것
	•	response helper와도 병렬적으로 이해 가능한 구조일 것
	•	이후 audit/observability와 연결하기 쉬울 것

⸻

Step 11. placeholder endpoint 또는 테스트 경로에서 동작 검증

최소한 몇 가지 예외를 실제로 발생시켜 공통 handler가 예상대로 응답하는지 검증하라.

권장 검증 대상:
	•	존재하지 않는 resource 요청 → not found 계열
	•	잘못된 query/body 입력 → validation 계열
	•	placeholder route에서 강제 raise → business error 계열
	•	예상치 못한 예외 → 500 generic error 계열

목표:
	•	오류 envelope이 실제로 통일되는지 확인
	•	OpenAPI/Swagger와 충돌이 없는지 확인
	•	다음 Task에서 auth context / list query validator를 연결할 준비가 되는지 확인

⸻

Step 12. OpenAPI 및 향후 확장 관점 검토

Swagger/OpenAPI와 향후 확장성을 검토하라.

확인할 점:
	•	성공 응답 구조(Task I-2)와 충돌이 없는가
	•	오류 응답도 점차 공통 response documentation을 붙이기 쉬운가
	•	향후 idempotency error, pagination/filter validation error, authz error를 같은 패턴으로 수용 가능한가
	•	async/operations/webhook/retrieval 계층에서도 재사용 가능할 것 같은가

⸻

5. 구현 시 세부 요구사항

5-1. 오류 응답 기본 원칙

오류 응답은 아래 원칙으로 고정하라.
	•	최상위는 error, meta
	•	error.code는 분기용
	•	error.message는 안전한 요약
	•	error.details는 선택적 확장
	•	meta.request_id, meta.trace_id는 가능하면 포함

권장 예시:

{
  "error": {
    "code": "not_found",
    "message": "Requested resource was not found"
  },
  "meta": {
    "request_id": "req_123",
    "trace_id": "trace_456"
  }
}


⸻

5-2. code naming 원칙

error.code는 사람이 아닌 시스템도 읽을 수 있는 안정적인 문자열이어야 한다.

권장:
	•	validation_error
	•	authentication_required
	•	permission_denied
	•	resource_not_found
	•	resource_conflict
	•	idempotency_key_conflict
	•	internal_server_error

비권장:
	•	"ERROR_001"
	•	"BadRequest"
	•	"SomethingWentWrong"

원칙:
	•	snake_case
	•	의미 명확
	•	도메인에 과도하게 종속되지 않음

⸻

5-3. message는 안전하고 간결하게

message는 외부 노출용이다.
따라서 아래를 지켜라.
	•	내부 시스템 구조를 드러내지 말 것
	•	개발자 전용 디버깅 문구를 넣지 말 것
	•	너무 장황하지 말 것
	•	클라이언트가 이해 가능한 수준으로 유지할 것

예:
	•	좋은 예: "Document not found"
	•	나쁜 예: "Document row missing in table documents_master_v3 due to failed repository lookup"

⸻

5-4. details는 선택적이며 구조적이어야 함

details는 에러가 복잡할 때만 사용하라.

대표 사용처:
	•	validation field 목록
	•	filter/sort 지원 불가 항목
	•	cursor format 오류
	•	idempotency mismatch 사유 요약

주의:
	•	자유 텍스트 dump처럼 쓰지 말 것
	•	가능하면 구조화된 데이터로 둘 것

⸻

5-5. 내부 예외와 외부 응답 분리

예기치 않은 내부 예외는 외부에는 일반화된 메시지로 내려야 한다.

권장:
	•	외부 응답: internal_server_error
	•	내부 로그: 원본 예외/traceback 보존 가능

즉, 응답과 로그를 같은 정보량으로 다루지 말 것.

⸻

5-6. 라우터에서 HTTPException 남발 금지

이번 단계 이후부터는 라우터/서비스에서 가능한 한 플랫폼 예외를 쓰는 방향으로 유도하라.

예:
	•	not found → ApiNotFoundError
	•	permission denied → ApiPermissionDeniedError

단, 기존 구조와 충돌이 크면 점진적 전환을 전제로 TODO를 남겨도 된다.

⸻

5-7. request_id / trace_id 연결 포인트 유지

Task I-4에서 request context가 본격 구현되기 전이라도, 이번 error handler는 request_id, trace_id를 추후 주입할 수 있는 인터페이스를 가져야 한다.

즉:
	•	지금 값이 없더라도 필드 구조는 유지
	•	request state 또는 context object에서 읽어올 수 있도록 확장 여지를 남길 것

⸻

6. 권장 설계 방향

Claude Code는 아래 방향을 우선 검토하라.

옵션 A. 모델 + 예외 + 핸들러 분리
	•	models.py → error response schema
	•	exceptions.py → business error classes
	•	handlers.py → exception → response mapping

장점:
	•	책임 분리 명확
	•	이후 유지보수 쉬움
	•	error code catalog 확장에 유리

옵션 B. 최소 구조 우선
	•	예외 클래스 수를 과도하게 늘리지 않고 핵심 예외만 먼저 구현
	•	핸들러도 핵심 케이스 중심으로 정리

이번 단계에서는 골격을 올바르게 만드는 것이 중요하므로,
예외 종류를 무한히 늘리기보다 “분류 체계”가 보이도록 구현하는 것을 우선하라.

⸻

7. 산출물 요구

Claude Code는 작업 후 아래 내용을 보고하라.

A. 생성/수정 파일 목록

예:
	•	app/api/errors/models.py
	•	app/api/errors/exceptions.py
	•	app/api/errors/handlers.py
	•	main.py 또는 app factory 파일
	•	일부 router 파일

B. 도입한 오류 응답 구조 요약

예:
	•	기본 error envelope
	•	validation error shape
	•	internal server error shape
	•	meta 필드 구성

C. 예외 계층 요약

예:
	•	base business error
	•	not found / conflict / auth / validation 계열
	•	HTTP status mapping 원칙

D. handler 동작 예시

간단히:
	•	validation 실패 시 어떤 응답이 내려가는지
	•	not found raise 시 어떤 응답이 내려가는지
	•	unexpected exception 시 어떤 응답이 내려가는지

E. 설계 판단 근거

짧게 정리:
	•	왜 error + meta 구조를 선택했는지
	•	왜 HTTPException 직접 사용보다 플랫폼 예외 계층을 도입했는지
	•	왜 details를 선택적 구조로 뒀는지

F. 남겨둔 TODO

예:
	•	request context와 request_id/trace_id 자동 주입 연결 예정
	•	error code catalog 확장 예정
	•	audit logging과 exception correlation 연결 예정
	•	domain-specific 예외 세분화 예정

⸻

8. 완료 조건

아래를 만족하면 완료로 본다.
	•	공통 error response envelope이 정의되어 있다.
	•	error object 모델이 정의되어 있다.
	•	business error base class가 존재한다.
	•	주요 예외 계층(validation/auth/permission/not_found/conflict/idempotency 초안)이 존재한다.
	•	global exception handler가 등록되어 있다.
	•	validation/HTTPException/unexpected Exception이 공통 규약으로 변환된다.
	•	외부 노출이 production-safe하다.
	•	request_id/trace_id를 추후 연결할 수 있는 meta 구조가 있다.

⸻

9. 금지 사항

이번 작업에서는 다음을 하지 마라.
	•	전체 에러 카탈로그를 완성하려 하지 말 것
	•	도메인별 세부 예외를 과하게 늘리지 말 것
	•	stack trace나 내부 오류를 외부 응답에 그대로 넣지 말 것
	•	성공 응답 envelope과 오류 응답 envelope을 섞지 말 것
	•	라우터마다 ad-hoc error JSON을 직접 만들지 말 것
	•	auth/context/idempotency를 아직 완성하지 않았는데 그것까지 확정하려 하지 말 것

⸻

10. Claude Code 최종 지시문

아래 지시문을 그대로 사용해도 된다.

Task I-3를 수행하라.

목표:
- 플랫폼 API의 공통 오류 모델 및 exception handling 계층을 구현한다.
- 모든 API 오류가 일관된 계약(error + meta)으로 반환되도록 기반을 만든다.
- business error base class와 대표 예외 계층을 정의한다.
- global exception handler를 통해 validation/auth/not_found/conflict/internal error 등을 공통 형식으로 변환한다.

반드시 지킬 원칙:
- 이번 단계는 오류 응답 계약과 예외 처리 골격을 세우는 작업이다.
- stack trace, raw DB error 등 내부 정보는 외부에 노출하지 마라.
- 성공 응답과 오류 응답의 최상위 구조를 섞지 마라.
- 라우터에서 ad-hoc HTTPException 남발 패턴을 줄이고, 플랫폼 예외 계층으로 수렴할 수 있게 하라.
- 이후 Task I-4, I-6, I-9, I-10, I-11과 충돌하지 않도록 확장 가능성을 유지하라.

수행 항목:
1. 현재 예외/오류 처리 방식을 점검한다.
2. 공통 error response envelope을 정의한다.
3. error object 모델을 정의한다.
4. business error base class를 구현한다.
5. validation/auth/permission/not_found/conflict/idempotency 계열 예외 초안을 만든다.
6. HTTP status mapping 기준을 정리한다.
7. global exception handler를 구현하고 앱에 등록한다.
8. FastAPI validation error를 공통 envelope로 변환한다.
9. 일반 Exception을 production-safe한 internal error 응답으로 변환한다.
10. 일부 경로에서 실제 동작을 검증한다.

산출물:
- 생성/수정 파일 목록
- 오류 응답 구조 요약
- 예외 계층 요약
- handler 동작 예시
- 설계 판단 근거
- 남겨둔 TODO 목록