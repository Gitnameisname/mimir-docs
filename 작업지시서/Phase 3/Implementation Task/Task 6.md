Task I-6. list query parser 및 pagination / filter / sort validator 구현

Claude Code 작업 계획서

1. 작업 목표

목록 조회 엔드포인트에서 공통으로 사용할 수 있는 list query parsing / validation 계층을 구현한다.

이번 단계의 핵심은 documents API 하나를 임시로 처리하는 것이 아니라, 앞으로 documents / versions / nodes / operations / admin listing 등 여러 목록형 API가 같은 규약을 따르도록 만드는 것이다.

이번 작업으로 확보하려는 것은 다음이다.
	•	page 기반 pagination의 공통 모델
	•	cursor 기반 pagination으로 확장 가능한 구조
	•	sort parameter의 공통 parser/validator
	•	filter query의 최소 공통 parsing 구조
	•	invalid query에 대한 일관된 validation 오류 연결
	•	meta.pagination 생성에 재사용 가능한 기반
	•	라우터마다 query parsing 로직이 흩어지지 않는 구조

즉, 이번 Task는 목록 조회 API의 공통 입력 계약을 코드로 고정하는 작업이다.

⸻

2. 작업 범위

포함 범위
	•	list query 공통 모델 정의
	•	page / page_size 기반 pagination parser
	•	cursor / limit 확장 가능 구조 설계
	•	sort parser 및 허용 필드 검증
	•	filter parameter parsing의 최소 구조
	•	invalid cursor / unsupported sort / invalid filter validation 연결
	•	meta.pagination 생성용 공통 모델 또는 helper 기반 마련
	•	documents list 등 대표 endpoint에 시범 적용

제외 범위
	•	복잡한 검색 DSL 구현
	•	고급 filter language 구현
	•	full-text search 통합
	•	DB query builder 완성
	•	cursor pagination full implementation
	•	모든 resource별 filter catalog 완성
	•	OpenSearch / Elasticsearch / vector search 통합

⸻

3. 구현 의도

목록 API는 초기에 대충 구현하면 가장 빨리 일관성이 무너지는 영역이다.

예를 들어:
	•	어떤 API는 page/pageSize
	•	어떤 API는 limit/offset
	•	어떤 API는 sortBy/order
	•	어떤 API는 filter_status
	•	어떤 API는 status=...

이런 식으로 흩어지면, 이후 UI/외부 시스템/SDK/AI agent 모두가 endpoint마다 별도 코드를 가져야 한다.

따라서 이번 작업의 목적은 다음과 같다.
	•	목록 조회 규약을 공통화
	•	parser/validator를 재사용 가능한 컴포넌트로 분리
	•	잘못된 query에 대해 공통 오류 응답을 반환
	•	pagination / sort / filter가 response meta와 자연스럽게 연결되게 함
	•	future cursor pagination으로 확장 가능한 구조를 초기에 확보

⸻

4. Claude Code에게 요청할 작업

아래 작업을 순서대로 수행하라.

⸻

Step 1. 현재 목록 endpoint와 query 처리 방식 점검

현재 코드베이스에서 목록성 endpoint가 어떻게 query를 다루는지 먼저 점검하라.

확인할 항목:
	•	이미 page, limit, offset, sort 같은 쿼리 파라미터를 쓰는 부분이 있는지
	•	documents placeholder list endpoint가 어떤 입력을 받는지
	•	FastAPI dependency 기반 query parsing 패턴이 있는지
	•	validation error가 Task I-3의 공통 오류 응답과 자연스럽게 연결되는지

목적:
	•	현재 구조를 무시하지 않고, 공통 parser/validator가 가장 자연스럽게 삽입될 위치를 정하는 것

⸻

Step 2. 공통 list query 모델 정의

목록 조회 엔드포인트가 공통으로 사용할 수 있는 list query 모델을 정의하라.

최소 권장 포함 요소:
	•	page: Optional[int]
	•	page_size: Optional[int]
	•	cursor: Optional[str]
	•	limit: Optional[int]
	•	sort: Optional[str]
	•	filters: ... 또는 filter parsing용 구조

중요 원칙:
	•	page 기반과 cursor 기반을 동시에 수용할 수 있는 구조를 먼저 만든다
	•	하지만 이번 단계에서는 실제 기본 적용은 page/page_size 중심으로 시작해도 괜찮다
	•	둘을 동시에 받았을 때의 처리 정책도 정리해야 한다

권장 방향:
	•	공통 request model 또는 dependency input model
	•	이후 resource별 allowed sort/filter spec을 주입 가능하게 설계

⸻

Step 3. pagination 모드 정책 정리

pagination 입력 정책을 명확히 정리하라.

권장 초기 정책:
	•	기본은 page + page_size
	•	future expansion을 위해 cursor + limit 슬롯 확보
	•	page/page_size와 cursor/limit를 동시에 보내면 validation error

권장 검증 규칙 예:
	•	page >= 1
	•	page_size >= 1
	•	page_size <= max_page_size
	•	limit >= 1
	•	limit <= max_limit
	•	unsupported 혼합 입력 금지

중요:
	•	이번 단계에서는 cursor pagination을 완성하지 않아도 된다
	•	다만 입력 계약과 충돌 방지 규칙은 먼저 정의해야 한다

⸻

Step 4. page/page_size parser 및 validator 구현

page 기반 pagination parser를 구현하라.

최소 요구:
	•	page 기본값 정책
	•	page_size 기본값 정책
	•	최대 허용치 제한
	•	음수/0/과대값 validation
	•	validation 실패 시 Task I-3 공통 오류 응답으로 연결

권장 예시 정책:
	•	page: default 1
	•	page_size: default 20
	•	page_size max: 100

중요:
	•	숫자 파라미터 변환/검증이 endpoint마다 반복되지 않게 할 것
	•	documents list 외의 다른 list endpoint도 재사용 가능해야 한다

⸻

Step 5. cursor/limit 확장 포인트 마련

cursor pagination이 아직 미구현이어도, 입력 파싱과 validation 관점에서 확장 포인트를 마련하라.

최소 요구:
	•	cursor 문자열 슬롯
	•	limit 검증
	•	page/page_size와의 혼합 금지
	•	invalid cursor 형식에 대한 placeholder validation 훅

예:
	•	현재는 cursor 자체 해석을 하지 않더라도,
validate_cursor(cursor) 또는 future decoder 연결 지점을 둘 수 있음

중요:
	•	이번 단계에서는 “cursor pagination ready”가 아니라
cursor-compatible input contract를 만드는 것이 목적이다

⸻

Step 6. sort parser 설계

정렬 파라미터를 공통 규약으로 파싱하는 구조를 구현하라.

권장 입력 예시:
	•	sort=created_at
	•	sort=-created_at
	•	sort=title,-updated_at

권장 파싱 결과 개념:
	•	field: created_at, direction: asc
	•	field: updated_at, direction: desc

핵심 요구:
	•	다중 sort 가능 여부를 정책으로 정할 것
	•	허용되지 않은 필드면 validation error
	•	정렬 방향 표현 규칙을 일관되게 유지할 것

권장 방향:
	•	leading - = descending
	•	no prefix = ascending

주의:
	•	sort_by, order, descending=true 같은 여러 방식 혼용 금지
	•	이번 단계부터 규약을 하나로 고정하는 것이 목적이다

⸻

Step 7. allowed sort field validation 구조 구현

각 리소스별로 허용 sort field를 지정할 수 있는 구조를 만들라.

예:
	•	documents list 허용 필드:
	•	created_at
	•	updated_at
	•	title
	•	status

권장 설계:
	•	공통 sort parser는 범용
	•	resource router 또는 service가 allowed field 목록을 전달
	•	parser/validator가 이를 검증

핵심:
	•	sort 규약은 공통화하되, 허용 필드는 resource-specific 이어야 한다
	•	하드코딩이 라우터 곳곳에 흩어지지 않게 할 것

⸻

Step 8. filter parsing 최소 구조 구현

필터는 이번 단계에서 과하게 복잡하게 만들지 말고, 기본 구조만 구현하라.

권장 방향 중 하나:
	•	단순 key-value filter
	•	허용된 filter 필드만 수용
	•	반복 파라미터 또는 comma-separated 처리 여부를 정책화

예:
	•	status=draft
	•	document_type=policy
	•	owner_id=...

또는 확장 가능한 구조:
	•	filter spec 정의
	•	parser가 query params에서 allowed filter만 추출

중요:
	•	이번 단계에서 고급 표현식(status[in]=..., or, and, nested`)까지 가지 말 것
	•	기본적인 equality-style filter 중심으로 시작하라

⸻

Step 9. allowed filter spec 구조 설계

리소스별 허용 filter 필드를 정의할 수 있는 구조를 만들라.

예:
	•	documents list 허용 필드:
	•	status
	•	document_type
	•	owner_id

필요 시 각 필드에 대해 다음 정도를 둘 수 있다.
	•	field name
	•	type
	•	optional coercion
	•	optional allowed values

중요:
	•	filter spec은 documents 전용 구조로 굳지 않게 할 것
	•	versions / nodes / operations에도 재사용 가능해야 한다

⸻

Step 10. invalid filter / unsupported sort / invalid pagination 오류 연결

잘못된 목록 query 입력이 들어왔을 때, 공통 오류 응답으로 자연스럽게 연결되게 하라.

반드시 다룰 항목:
	•	unsupported sort field
	•	malformed sort syntax
	•	invalid page/page_size/limit
	•	page + cursor 혼합
	•	unsupported filter field
	•	invalid filter value 형식

권장 error code 예:
	•	validation_error
	•	또는 더 구체적으로
	•	invalid_pagination
	•	unsupported_sort
	•	invalid_filter

이 부분은 Task I-3와 충돌하지 않게, 세부 code를 너무 많이 늘리지 않으면서도 구조적 details를 담을 수 있게 하라.

⸻

Step 11. pagination meta 모델/헬퍼 기반 마련

목록 응답의 meta.pagination에 넣을 수 있는 공통 모델 또는 helper 기반을 만들라.

최소 권장 필드:
	•	page
	•	page_size
	•	total (optional)
	•	has_next (optional)
	•	next_cursor (optional placeholder)

주의:
	•	지금 총 개수(total)를 항상 계산하지 않아도 된다
	•	cursor pagination이 아직 없어도 next_cursor 필드 자리는 둘 수 있다
	•	이번 단계에서는 “meta.pagination에 무엇을 담을지”의 기본 틀을 만드는 것이 목적이다

⸻

Step 12. documents list endpoint에 시범 적용

documents list endpoint에 공통 list query parser/validator를 연결하라.

최소 적용 항목:
	•	page/page_size
	•	sort
	•	기본 filter 몇 개
	•	validation 오류 처리 연결
	•	response meta.pagination 자리 반영 가능 여부 검토

목표:
	•	parser/validator 구조가 실제 router에서 재사용 가능한지 검증
	•	documents 외 다른 리소스에도 쉽게 붙일 수 있는 패턴인지 확인

주의:
	•	실제 DB query 최적화나 복잡한 검색까지 가지 말 것
	•	지금은 API contract와 공통 parsing 계층이 목적이다

⸻

Step 13. versions / nodes 목록 endpoint에 재사용 가능성 검토

versions/nodes 목록 endpoint에도 같은 규약을 재사용할 수 있는지 검토하라.

이번 단계에서 실제로 다 붙이지 않아도 되지만 최소한 아래는 정리하라.
	•	documents와 같은 dependency를 그대로 쓸 수 있는지
	•	resource-specific allowed sort/filter spec만 바꾸면 되는지
	•	path/resource 관계 검증과 query 규약이 충돌하지 않는지

⸻

Step 14. OpenAPI 문서 관점 검토

Swagger/OpenAPI에서 목록 query가 일관되게 보이는지 검토하라.

확인할 점:
	•	page, page_size, sort 설명이 자연스러운가
	•	unsupported한 파라미터 이름들이 노출되지 않는가
	•	문서/버전/노드 목록 API가 같은 규약을 공유하는 것이 보이는가
	•	향후 cursor pagination이 추가되어도 문서 구조를 유지할 수 있는가

⸻

5. 구현 시 세부 요구사항

5-1. 목록 조회 규약은 공통화, 허용 필드는 리소스별 분리

이번 Task의 핵심 원칙이다.
	•	pagination/sort/filter 형식은 공통
	•	어떤 필드를 허용할지는 리소스별

즉:
	•	documents, versions, nodes가 같은 파라미터 규약을 사용
	•	허용 field 목록만 각자 다를 수 있음

⸻

5-2. 파라미터 naming 규칙 고정

이번 단계부터 목록 query naming은 일관되게 고정하라.

권장:
	•	page
	•	page_size
	•	cursor
	•	limit
	•	sort
	•	filter field는 직접 노출 (status=...)

비권장:
	•	pageSize
	•	offset
	•	sortBy
	•	order
	•	filter_status

⸻

5-3. sort 표현은 단일 규약 사용

정렬 방식은 하나로 통일하라.

권장:
	•	sort=field,-other_field

이 규약을 택했다면,
	•	no prefix = asc
	•	- prefix = desc

를 고정하라.

⸻

5-4. filter는 최소 구조로 시작

이번 단계에서는 filter 기능을 과하게 욕심내지 말 것.

우선은:
	•	equality 기반
	•	허용 필드만
	•	기본 타입 검증

정도로 시작하고, 복합 조건/범위 조건은 이후 Phase로 미뤄라.

⸻

5-5. page 기반을 기본, cursor 기반은 확장 슬롯

이번 MVP 단계에서는 page 기반이 더 단순하므로 기본값으로 삼아도 된다.

다만:
	•	cursor 기반이 미래에 들어올 것을 막지 말 것
	•	입력 contract와 meta.pagination 설계에서 cursor-compatible 구조를 유지할 것

⸻

5-6. validation은 공통 오류 계층으로 연결

목록 query 오류가 들어왔을 때 라우터마다 제각각 다른 400 JSON을 만들지 말 것.

반드시:
	•	Task I-3의 공통 예외/오류 응답 흐름으로 연결
	•	details에 어떤 필드가 왜 잘못됐는지 구조적으로 담을 수 있게 할 것

⸻

5-7. 라우터는 parsing 조립만, 의미 해석은 분리

라우터의 역할은 다음 정도로 제한하라.
	•	list query dependency 주입
	•	allowed sort/filter spec 전달
	•	service에 정규화된 query object 전달
	•	response shaping

하지 말아야 할 것:
	•	라우터마다 query string 수동 파싱
	•	sort/filter 규칙 직접 if문 하드코딩
	•	query 검증을 라우터 함수 안에서 반복 작성

⸻

6. 권장 설계 방향

Claude Code는 아래 방향을 우선 검토하라.

옵션 A. 공통 dependency + resource spec 주입

예:
	•	parse_list_query(...)
	•	ListQuerySpec(allowed_sort_fields=[...], allowed_filters=[...])

장점:
	•	가장 재사용성이 높음
	•	documents/versions/nodes에 쉽게 확장 가능
	•	라우터는 spec만 넘기면 됨

옵션 B. 공통 모델 + resource-specific wrapper

예:
	•	공통 parser는 존재
	•	documents용 dependency, versions용 dependency가 wrapper 역할

장점:
	•	OpenAPI 문서 설명을 resource별로 조금 더 친절하게 줄 수 있음
	•	초기 실무 적용이 편할 수 있음

이번 단계에서는 공통 parser + resource spec 주입 방식을 우선 검토하되,
문서화나 FastAPI dependency ergonomics상 필요하면 wrapper를 허용하라.

⸻

7. 산출물 요구

Claude Code는 작업 후 아래 내용을 보고하라.

A. 생성/수정 파일 목록

예:
	•	app/api/query/models.py
	•	app/api/query/pagination.py
	•	app/api/query/sorting.py
	•	app/api/query/filtering.py
	•	app/api/query/dependencies.py
	•	app/api/responses/models.py
	•	app/api/v1/documents.py

B. 도입한 list query 구조 요약

예:
	•	공통 list query 모델
	•	page/page_size 정책
	•	cursor/limit 확장 슬롯
	•	sort 표현 방식
	•	filter 구조

C. validation 정책 요약

예:
	•	page/page_size 제한
	•	page/cursor 혼합 금지
	•	unsupported sort 처리 방식
	•	invalid filter 처리 방식

D. endpoint 적용 예시

간단히:
	•	documents list에서 어떤 allowed sort/filter spec을 사용했는지
	•	어떤 validation 오류가 어떻게 응답되는지

E. 설계 판단 근거

짧게 정리:
	•	왜 page 기반을 기본으로 두었는지
	•	왜 sort=-field 규약을 택했는지
	•	왜 filter를 최소 구조로 시작했는지
	•	왜 parser와 resource spec을 분리했는지

F. 남겨둔 TODO

예:
	•	cursor decode/encode 실제 구현 예정
	•	total count 최적화 정책 예정
	•	range/search/full-text filter 확장 예정
	•	resource별 filter spec 확장 예정

⸻

8. 완료 조건

아래를 만족하면 완료로 본다.
	•	공통 list query parser/validator 구조가 존재한다.
	•	page/page_size 기반 pagination 입력이 처리된다.
	•	cursor/limit 확장 슬롯이 있다.
	•	sort parser와 allowed sort field 검증이 존재한다.
	•	filter parsing 최소 구조가 존재한다.
	•	invalid pagination/sort/filter가 공통 오류 응답으로 연결된다.
	•	meta.pagination 생성용 공통 모델/헬퍼 기반이 있다.
	•	documents list 같은 대표 endpoint에 시범 적용되어 있다.

⸻

9. 금지 사항

이번 작업에서는 다음을 하지 마라.
	•	복잡한 검색 DSL까지 구현하려 하지 말 것
	•	DB query builder를 과도하게 완성하려 하지 말 것
	•	documents 전용 query 규약을 만들지 말 것
	•	page/offset/sortBy/order 등 여러 스타일을 혼합하지 말 것
	•	cursor pagination을 억지로 완성하려 하지 말 것
	•	validation JSON을 라우터에서 ad-hoc하게 만들지 말 것

⸻

10. Claude Code 최종 지시문

아래 지시문을 그대로 사용해도 된다.

Task I-6를 수행하라.

목표:
- 플랫폼 API의 목록 조회에 공통으로 사용할 list query parser 및 pagination/filter/sort validator를 구현한다.
- documents/versions/nodes 등 여러 목록형 API가 같은 query 규약을 사용할 수 있도록 기반을 만든다.
- page/page_size 기반 pagination을 우선 지원하고, cursor/limit 확장 슬롯을 확보한다.
- sort parser와 filter parser를 공통 구조로 만들고, resource별 allowed spec으로 검증할 수 있게 한다.

반드시 지킬 원칙:
- 이번 단계는 목록 조회 query contract의 공통 기반을 세우는 작업이다.
- pagination/sort/filter 형식은 공통화하고, 허용 필드는 resource별로 분리하라.
- 라우터에서 query parsing/validation을 직접 반복 구현하지 말고 공통 parser/dependency로 수렴시켜라.
- invalid pagination/sort/filter는 Task I-3의 공통 오류 응답 흐름으로 연결하라.
- cursor pagination은 지금 완성하지 말고 확장 가능한 슬롯만 마련하라.

수행 항목:
1. 현재 목록 endpoint의 query 처리 방식을 점검한다.
2. 공통 list query 모델을 정의한다.
3. page/page_size 기본 정책과 cursor/limit 확장 정책을 정리한다.
4. pagination parser/validator를 구현한다.
5. sort parser와 allowed sort field 검증 구조를 구현한다.
6. filter parsing 최소 구조와 allowed filter spec 구조를 구현한다.
7. invalid pagination/sort/filter 오류를 공통 오류 응답으로 연결한다.
8. meta.pagination 생성용 공통 모델 또는 helper 기반을 만든다.
9. documents list endpoint에 시범 적용해 패턴을 검증한다.
10. versions/nodes 목록 API로 재사용 가능한 구조인지 검토한다.

산출물:
- 생성/수정 파일 목록
- list query 구조 요약
- validation 정책 요약
- endpoint 적용 예시
- 설계 판단 근거
- 남겨둔 TODO 목록
