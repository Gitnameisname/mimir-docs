Task I-2. 공통 응답 envelope 및 response helper 구현

Claude Code 작업 계획서

1. 작업 목표

플랫폼 API의 모든 성공 응답이 앞으로 일관된 계약(contract) 위에서 반환되도록, 공통 응답 envelope과 helper 계층을 구현한다.

이번 작업의 핵심은 단순히 응답 JSON을 예쁘게 꾸미는 것이 아니라, 이후 모든 도메인 API가 다음 특성을 공유하도록 만드는 것이다.
	•	성공 응답 포맷 일관화
	•	단건 / 목록 / 비동기 수락 응답의 공통 패턴 확보
	•	meta 확장 위치 확보
	•	request_id / trace_id 연결 가능 구조 확보
	•	list query / pagination / idempotency / audit / async operation 확장을 수용하는 응답 기반 마련

즉, 이번 Task는 응답 스키마의 공용 기반을 세우는 작업이다.

⸻

2. 작업 범위

포함 범위
	•	공통 success response envelope 정의
	•	single resource response helper
	•	list response helper
	•	meta object 구조 정의
	•	request_id / trace_id 반영 가능한 응답 구조
	•	async accepted response helper 초안
	•	response serialization 일관화
	•	documents / versions / nodes / system placeholder endpoint 일부에 시범 적용 가능 구조 마련

제외 범위
	•	공통 error response 구현 완료
	•	exception handler 구현
	•	실제 pagination 계산 로직 구현 완료
	•	auth context 실제 응답 반영
	•	operation tracking 실제 구현
	•	webhook/event 실제 payload 정의
	•	전체 API 일괄 마이그레이션 완료

⸻

3. 구현 의도

지금 이 단계에서 공통 응답 구조를 잡아두지 않으면, 이후 documents/versions/nodes API가 제각각 다른 JSON 형태를 내보내게 되고, 나중에 통합 비용이 급격히 커진다.

따라서 이번 작업은 다음 목적을 가진다.
	•	라우터마다 임의의 응답 구조가 생기는 것을 방지
	•	클라이언트/UI/외부 시스템이 예측 가능한 응답을 받도록 기반 통일
	•	향후 확장 필드(pagination, links, operation, correlation, idempotency-replay)를 넣을 위치 확보
	•	비즈니스 로직과 응답 shaping을 분리하는 패턴 도입

⸻

4. Claude Code에게 요청할 작업

아래 작업을 순서대로 수행하라.

⸻

Step 1. 현재 placeholder 응답 상태 점검

Task I-1에서 만든 placeholder endpoint들의 현재 응답 형태를 점검하라.

확인할 항목:
	•	endpoint별 반환 형식이 제각각인지
	•	현재 FastAPI에서 dict 직접 반환 중인지
	•	Pydantic response model을 이미 일부 사용 중인지
	•	어떤 수준까지 공통 helper 적용이 자연스러운지

이 단계의 목적은, 기존 구조를 무시하지 말고 가장 낮은 마찰로 공통 응답 계층을 삽입하는 방법을 찾는 것이다.

⸻

Step 2. 공통 success envelope 모델 정의

성공 응답의 최상위 envelope를 위한 공통 모델을 정의하라.

권장 개념은 아래와 같다.

{
  "data": ...,
  "meta": {
    "request_id": "...",
    "trace_id": "..."
  }
}

혹은 필요 시 아래에 가까운 구조도 가능하다.

{
  "data": ...,
  "meta": { ... }
}

중요 원칙:
	•	성공 응답은 기본적으로 data와 meta를 가진다
	•	meta는 없애지 말고, 비어 있더라도 확장 가능한 위치로 유지할 것
	•	status, message 같은 필드를 남발하지 말 것
	•	성공 응답에서는 구조를 최대한 단순하고 예측 가능하게 유지할 것

권장 필드:
	•	meta.request_id
	•	meta.trace_id
	•	이후 확장을 위한 공간 (pagination, links, operation 등)

⸻

Step 3. meta 모델 구조 설계

meta 객체의 최소 공통 모델을 설계하라.

초기 단계에서 권장되는 최소 구조:
	•	request_id: Optional[str]
	•	trace_id: Optional[str]

추가 가능하지만 아직 과하게 확정하지 말아야 할 것:
	•	pagination
	•	links
	•	operation
	•	warnings

중요:
	•	이번 단계에서는 meta를 확장 가능하되 과도하게 복잡하게 만들지 말 것
	•	다음 Task들에서 쉽게 확장 가능해야 한다
	•	pagination 정보는 Step 5의 list helper에서 자연스럽게 넣을 수 있어야 한다

⸻

Step 4. single response helper 구현

단건 또는 일반 성공 응답을 만들기 위한 helper를 구현하라.

예시 개념:
	•	build_response(data=..., meta=...)
	•	또는 success_response(...)

반드시 고려할 점:
	•	helper는 라우터에서 쉽게 사용할 수 있어야 한다
	•	라우터가 envelope shape를 직접 매번 dict로 조립하지 않게 할 것
	•	request context가 아직 완성되지 않았더라도, 나중에 request_id, trace_id를 주입할 수 있게 인터페이스를 설계할 것

권장 사용 예시 느낌:

return success_response(
    data=document_payload,
    request_id=request_id,
    trace_id=trace_id,
)

혹은

return build_single_response(data=payload, meta=meta)

핵심은 라우터에서 응답 shaping 중복을 줄이는 것이다.

⸻

Step 5. list response helper 구현

목록 응답을 위한 helper를 구현하라.

권장 shape:

{
  "data": [ ... ],
  "meta": {
    "request_id": "...",
    "trace_id": "...",
    "pagination": {
      ...
    }
  }
}

이 단계에서 중요한 점:
	•	pagination이 아직 완성되지 않았더라도, 넣을 자리와 인터페이스를 먼저 확보할 것
	•	data는 목록이어야 하며, 목록 응답과 단건 응답의 구분이 자연스러워야 한다
	•	list endpoint에서 items를 최상위에 둘지, data에 넣을지 혼란이 없게 할 것

권장:
	•	최상위는 무조건 data
	•	목록일 경우 data에 배열
	•	pagination 정보는 meta.pagination

즉:
	•	단건: data: {...}
	•	목록: data: [{...}, {...}]

⸻

Step 6. async accepted response helper 초안 구현

향후 operation/event/webhook/async 처리에 대비해, 202 Accepted 응답을 위한 helper 초안을 만들어라.

권장 목적:
	•	비동기 작업 시작 시 일관된 accepted 응답을 줄 수 있게 하기 위함
	•	아직 실제 operation resource는 없어도, 응답 형태를 위한 공통 자리를 만든다

예시 방향:

{
  "data": {
    "status": "accepted",
    "operation_id": "...",
    "resource": "..."
  },
  "meta": {
    "request_id": "...",
    "trace_id": "..."
  }
}

주의:
	•	아직 operation model을 과도하게 고정하지 말 것
	•	단지 “accepted response helper” 수준의 초안만 마련할 것
	•	Task I-11에서 본격 확장 가능하도록 너무 세부적인 스키마를 박지 말 것

⸻

Step 7. response model / helper 배치 위치 정리

공통 응답 관련 코드의 위치를 정리하라.

권장 예시:

app/
  api/
    responses/
      __init__.py
      models.py
      helpers.py

또는 프로젝트 구조에 따라:

app/common/api/
app/shared/http/
app/platform/api/

중요 원칙:
	•	응답 모델과 helper가 특정 domain router 안에 들어가면 안 된다
	•	documents 전용 구조처럼 보이지 않게 해야 한다
	•	error model과도 이후 자연스럽게 나란히 놓일 수 있어야 한다

⸻

Step 8. placeholder endpoint 일부에 시범 적용

Task I-1에서 만든 endpoint들 중 일부에 공통 response helper를 연결하라.

최소 적용 권장 대상:
	•	GET /api/v1/system/health
	•	GET /api/v1/system/info 또는 meta endpoint
	•	GET /api/v1/documents
	•	POST /api/v1/documents

목표:
	•	helper가 실제 라우터에서 잘 작동하는지 검증
	•	envelope 구조가 지나치게 번거롭지 않은지 확인
	•	이후 Task I-7 / I-8에서 전면 적용이 가능한 패턴 확정

주의:
	•	모든 endpoint를 한 번에 다 바꾸려 하지 말 것
	•	먼저 대표 endpoint에 안정적으로 적용하는 것이 목적이다

⸻

Step 9. serialization 기준 정리

응답 serialization 기준을 점검하라.

검토 항목:
	•	None 필드 처리 방침
	•	빈 meta 허용 여부
	•	UUID / datetime 등 직렬화 확장 가능성
	•	Pydantic model 기반 응답 vs dict 반환 중 어떤 방식을 기본으로 둘지

권장:
	•	가능한 한 Pydantic response model + helper 조합을 우선 검토
	•	다만 현재 코드베이스 구조상 dict helper가 더 자연스러우면, 과도하게 무리하지 말 것
	•	중요한 것은 “일관된 응답 shape”이지, 특정 기술 선택 자체가 아니다

⸻

Step 10. OpenAPI 관점 검토

Swagger/OpenAPI에서 응답 구조가 너무 난잡하지 않은지 검토하라.

확인할 점:
	•	success response shape가 endpoint마다 일관적인가
	•	단건/목록 응답 설명이 자연스러운가
	•	나중에 에러 응답 모델(Task I-3)을 붙여도 충돌이 적은가
	•	accepted response helper가 미래 확장에 방해되지 않는가

⸻

5. 구현 시 세부 요구사항

5-1. 응답 envelope 기본 원칙

이번 단계에서는 성공 응답의 기본 형태를 아래 원칙으로 고정하라.
	•	최상위 응답은 data, meta
	•	실제 리소스 payload는 data
	•	부가 정보는 meta
	•	임의의 루트 필드 남발 금지

권장 예시:

{
  "data": {
    "id": "doc_123",
    "title": "Sample"
  },
  "meta": {
    "request_id": "req_abc",
    "trace_id": "trace_xyz"
  }
}

목록 예시:

{
  "data": [
    {
      "id": "doc_1"
    },
    {
      "id": "doc_2"
    }
  ],
  "meta": {
    "request_id": "req_abc",
    "trace_id": "trace_xyz",
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 2
    }
  }
}


⸻

5-2. message/status 남용 금지

성공 응답에서 다음 필드를 습관적으로 넣지 말 것.
	•	message: "success"
	•	status: "ok"
	•	code: 200

이유:
	•	성공 응답은 envelope 자체로 충분히 표현 가능하다
	•	상태 코드는 HTTP가 표현한다
	•	message 남용은 API 계약을 불필요하게 흐린다

단, 202 Accepted helper처럼 비동기 흐름의 의미 전달이 필요한 경우에는 data.status 수준에서 제한적으로 허용 가능하다.

⸻

5-3. meta는 확장 가능해야 함

meta는 단순 현재 필드만 담는 곳이 아니라, 이후 아래 정보를 수용할 수 있어야 한다.
	•	pagination
	•	links
	•	operation
	•	warnings
	•	idempotency replay metadata
	•	correlation identifiers

하지만 이번 단계에서 이를 모두 구현하지는 말 것.
확장 자리만 자연스럽게 확보하면 된다.

⸻

5-4. 라우터에서 직접 envelope 조립 최소화

router 함수마다 아래처럼 직접 응답을 만들지 않게 하는 것이 목표다.

비권장:

return {
    "data": payload,
    "meta": {
        "request_id": req_id,
        "trace_id": trace_id,
    },
}

권장:

return success_response(
    data=payload,
    request_id=req_id,
    trace_id=trace_id,
)

즉, 공통 helper를 통해 응답 shaping 중복을 줄여라.

⸻

5-5. 이후 Task와의 연결성 고려

이번 구현은 아래 Task와 충돌하면 안 된다.
	•	Task I-3: 공통 오류 모델/예외 처리
	•	Task I-4: request context / trace context
	•	Task I-6: pagination/filter/sort validator
	•	Task I-9: idempotency replay metadata
	•	Task I-11: async accepted / operations
	•	Task I-12: AI/RAG citation-friendly read model

즉, envelope 구조는 지금 단순해야 하지만 미래 확장을 막지 않아야 한다.

⸻

6. 권장 설계 방향

Claude Code는 아래와 같은 설계 방향을 우선 검토하라.

옵션 A. Pydantic 모델 + helper
	•	ResponseMeta
	•	SuccessResponse[T]
	•	ListResponse[T] 또는 SuccessResponse[list[T]]
	•	helper가 이 모델을 반환

장점:
	•	명시적 계약
	•	OpenAPI 문서화에 유리
	•	향후 정적 검증에 좋음

옵션 B. dict helper + 최소 모델
	•	meta만 모델화
	•	envelope는 helper dict 반환

장점:
	•	빠르게 적용 가능
	•	초기 마이그레이션 비용이 낮음

우선은 프로젝트 현실에 맞게 가장 자연스럽고 유지보수 가능한 쪽을 선택하되,
가능하면 타입 안정성과 문서화를 위해 모델 기반 접근을 우선 검토하라.

⸻

7. 산출물 요구

Claude Code는 작업 후 아래 내용을 보고하라.

A. 생성/수정 파일 목록

예:
	•	app/api/responses/models.py
	•	app/api/responses/helpers.py
	•	app/api/v1/system.py
	•	app/api/v1/documents.py

B. 도입한 응답 구조 요약

예:
	•	성공 응답 기본 구조
	•	목록 응답 구조
	•	accepted response 구조 초안
	•	meta 필드 구성

C. helper 사용 예시

간단히:
	•	system health endpoint에서 어떻게 사용했는지
	•	documents list endpoint에서 어떻게 사용했는지

D. 설계 판단 근거

짧게 정리:
	•	왜 data + meta 구조를 선택했는지
	•	왜 meta.pagination 구조를 선택했는지
	•	왜 accepted helper를 이 수준으로만 정의했는지

E. 남겨둔 TODO

예:
	•	request context에서 request_id/trace_id 자동 주입 예정
	•	error response envelope 연결 예정
	•	list query validator와 pagination meta 연계 예정
	•	idempotency replay meta 연계 예정

⸻

8. 완료 조건

아래를 만족하면 완료로 본다.
	•	성공 응답의 공통 envelope이 정의되어 있다.
	•	single response helper가 존재한다.
	•	list response helper가 존재한다.
	•	async accepted response helper 초안이 존재한다.
	•	meta에 request_id / trace_id를 담을 수 있는 구조가 있다.
	•	일부 placeholder endpoint에 실제 적용되어 있다.
	•	응답 구조가 이후 error/context/pagination/idempotency 확장을 막지 않는다.

⸻

9. 금지 사항

이번 작업에서는 다음을 하지 마라.
	•	error response까지 같이 확정하려고 하지 말 것
	•	pagination 계산 로직까지 과하게 구현하지 말 것
	•	documents 전용 응답 구조를 만들지 말 것
	•	성공 응답에 message/status/code를 남발하지 말 것
	•	router마다 제각각 다른 envelope helper를 만들지 말 것
	•	accepted response를 실제 operation engine처럼 과도하게 설계하지 말 것

⸻

10. Claude Code 최종 지시문

아래 지시문을 그대로 사용해도 된다.

Task I-2를 수행하라.

목표:
- 플랫폼 API의 공통 성공 응답 envelope 및 response helper를 구현한다.
- 모든 성공 응답이 앞으로 일관된 계약 위에서 반환될 수 있도록 기반을 만든다.
- 단건 응답, 목록 응답, async accepted 응답의 공통 패턴을 정의한다.
- meta에 request_id / trace_id / pagination 등 확장 가능한 위치를 확보한다.

반드시 지킬 원칙:
- 이번 단계는 성공 응답 계약의 공통 기반을 세우는 작업이다.
- error response까지 과도하게 확장하지 마라.
- 성공 응답에서 message/status/code를 남발하지 마라.
- router에서 직접 envelope를 반복 조립하지 않게 helper 중심으로 구현하라.
- 이후 Task I-3, I-4, I-6, I-9, I-11, I-12와 충돌하지 않도록 확장 가능성을 유지하라.

수행 항목:
1. 현재 placeholder endpoint들의 응답 형태를 점검한다.
2. 공통 success envelope 모델을 정의한다.
3. meta 모델을 정의한다.
4. single response helper를 구현한다.
5. list response helper를 구현한다.
6. async accepted response helper 초안을 구현한다.
7. 공통 response 관련 코드의 배치 위치를 정리한다.
8. 일부 placeholder endpoint에 helper를 시범 적용한다.
9. serialization/OpenAPI 관점에서 구조를 점검한다.

산출물:
- 생성/수정 파일 목록
- 성공 응답 구조 요약
- helper 사용 예시
- 설계 판단 근거
- 남겨둔 TODO 목록