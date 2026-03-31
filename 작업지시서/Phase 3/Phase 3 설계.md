Phase 3-Implementation. 플랫폼 API 기초 계층 실제 구현

1. 구현 단계 목적

Phase 3에서 정의한 설계 기준을 바탕으로, 실제 코드베이스에 플랫폼 API의 기초 골격을 구현한다.

이 단계의 목표는 모든 기능을 한 번에 완성하는 것이 아니라, 다음을 우선 확보하는 것이다.
	•	버전 있는 API 라우팅 골격
	•	공통 요청/응답 계약 계층
	•	공통 오류 처리 계층
	•	인증 컨텍스트 주입 구조
	•	권한 검사용 연결 지점
	•	핵심 문서 리소스 API의 최소 골격
	•	목록 조회 규약 적용 기반
	•	idempotency 적용 가능한 write 구조
	•	async/event/webhook/AI 확장을 막지 않는 구조

즉, 이 구현 단계는 기능 완성이 아니라
플랫폼 API의 구현 가능한 뼈대와 핵심 흐름을 실제 코드로 세우는 단계다.

⸻

2. 구현 원칙

2-1. 설계 준수 우선

편의상 ad-hoc endpoint를 만들지 말고, Phase 3 설계 기준을 먼저 코드 구조에 반영한다.

2-2. 공통 계층 먼저

documents API부터 급하게 만들기보다, 공통 응답/오류/auth context/list query 같은 기반을 먼저 구현한다.

2-3. MVP 범위 통제

초기에는 다음만 우선한다.
	•	documents
	•	versions
	•	nodes
	•	공통 response/error
	•	auth context plumbing
	•	기본 list query 처리
	•	최소한의 idempotent write 구조

2-4. 확장 가능 구조 유지

async/webhook/AI retrieval을 당장 다 구현하지 않더라도, 나중에 붙일 수 있게 인터페이스와 훅 포인트를 남긴다.

2-5. 라우터에 비즈니스 로직 과적재 금지

라우터는 request parsing / response shaping / auth context 연결에 집중하고, 서비스 계층과 권한 계층으로 분리한다.

⸻

3. 구현 범위

초기 구현 포함
	•	API 버전 prefix 및 라우터 골격
	•	공통 response envelope/helper
	•	공통 error model/helper/handler
	•	request_id / trace_id 골격
	•	auth context extraction 구조
	•	authorization hook point
	•	documents / versions / nodes 핵심 API 골격
	•	list query parsing/validation 기본 구조
	•	idempotency hook for selected writes
	•	operation/event/webhook/AI용 placeholder extension point

초기 구현 제외
	•	전체 webhook delivery engine
	•	full event bus integration
	•	full AI retrieval engine
	•	vector/embedding/indexing pipeline
	•	full bulk API suite
	•	full admin API suite
	•	full error code catalog
	•	concurrency control full implementation

⸻

4. 하위 구현 Task 분해

Task I-1. API skeleton 및 버전 라우팅 구조 구현

목적

플랫폼 API의 최상위 라우팅 구조와 모듈 분리를 구현한다.

주요 내용
	•	/api/v1 기준 라우터 구조
	•	domain router 분리 기준
	•	documents / versions / nodes / system routes placeholder
	•	health / base metadata endpoint 여부 검토
	•	향후 admin / webhooks / operations / retrieval 확장을 고려한 패키지 구조

결과
	•	API 골격이 일관된 버전 구조 위에서 시작 가능

⸻

Task I-2. 공통 응답 envelope 및 response helper 구현

목적

모든 성공 응답이 일관된 포맷을 사용하도록 공통 응답 계층을 구현한다.

주요 내용
	•	단건 응답 helper
	•	목록 응답 helper
	•	meta 구조 주입
	•	request_id / trace_id 반영 구조
	•	async accepted 응답용 helper 초안
	•	response serialization 기준 정리

결과
	•	엔드포인트별 응답 구조 편차 방지

⸻

Task I-3. 공통 오류 모델 및 exception handling 계층 구현

목적

플랫폼 전반에서 일관된 오류 응답과 예외 처리 구조를 구현한다.

주요 내용
	•	공통 error object 모델
	•	business error base class 계층
	•	validation/auth/permission/conflict/not_found/idempotency 계열 예외 초안
	•	global exception handler
	•	request_id / trace_id 포함
	•	안전한 외부 노출 규칙 반영

결과
	•	오류 응답 표준화 및 운영 추적 기반 확보

⸻

Task I-4. request context / trace context / auth context plumbing 구현

목적

모든 요청이 공통 request context를 가지도록 기반 구조를 만든다.

주요 내용
	•	request_id 생성/전달
	•	trace_id 연결 포인트
	•	actor context container
	•	tenant/org context container
	•	auth method / client context placeholder
	•	downstream service/audit layer에 전달 가능한 구조

결과
	•	이후 authz, audit, idempotency, observability 구현의 공통 기반 확보

⸻

Task I-5. 인증 컨텍스트 추출 및 authorization hook point 구현

목적

실제 상세 auth 구현 전이라도, API 계층에서 인증 주체와 권한 판단 연결 구조를 구현한다.

주요 내용
	•	current actor 추출 dependency/middleware 골격
	•	anonymous / authenticated / service actor 구분 구조
	•	authorization service 호출 인터페이스
	•	router에서 policy check를 직접 하드코딩하지 않는 패턴 도입
	•	tenant scope enforcement hook point

결과
	•	보안 구조를 해치지 않는 API 구현 시작 가능

⸻

Task I-6. list query parser 및 pagination/filter/sort validator 구현

목적

목록 조회 엔드포인트에서 공통 query 규약을 재사용할 수 있도록 기반 컴포넌트를 구현한다.

주요 내용
	•	page/page_size 또는 cursor/limit 모델 구조
	•	sort parser
	•	filter parameter parsing 기본기
	•	invalid cursor / unsupported sort / invalid filter validation 연결
	•	meta.pagination 생성용 공통 모델

결과
	•	documents 등 목록 API에 동일 규약 적용 가능

⸻

Task I-7. documents 핵심 API 구현

목적

핵심 문서 리소스의 최소 CRUD/read/list 골격을 구현한다.

주요 내용
	•	document create
	•	document get
	•	document list
	•	document update 기본 구조
	•	상태 필드/metadata 기본 반영
	•	공통 응답/오류/auth/list query 구조 적용
	•	idempotency 적용 대상 write 연결 준비

결과
	•	플랫폼 API의 첫 번째 핵심 리소스 확보

⸻

Task I-8. versions / nodes API 구현

목적

문서의 구조성과 버전성을 반영하는 하위 리소스 API를 구현한다.

주요 내용
	•	document versions list/create/get
	•	version nodes list/get
	•	canonical/subresource path 반영
	•	문서-버전-노드 관계 검증
	•	AI/RAG용 구조 조회를 방해하지 않는 모델 유지

결과
	•	범용 문서 도메인 모델이 실제 API로 노출되기 시작함

⸻

Task I-9. write endpoint idempotency hook 구현

목적

중복 생성/재시도에 취약한 write endpoint에 idempotency 적용 기반을 넣는다.

주요 내용
	•	Idempotency-Key 처리 골격
	•	selected POST/action endpoint 연결
	•	same key / mismatch flow 초안
	•	replay response structure 연결
	•	request context 및 audit 연계 포인트 확보

결과
	•	재시도 안전성의 최소 구현 기반 확보

⸻

Task I-10. audit / observability 연결 포인트 구현

목적

API 요청과 결과를 추적할 수 있도록 운영성과 감사 추적의 최소 연결 포인트를 구현한다.

주요 내용
	•	request_id 기반 로깅
	•	actor / tenant / resource / action logging shape 초안
	•	감사 이벤트 emit hook point
	•	operation/event/webhook 확장을 고려한 correlation field 설계 반영
	•	success/failure logging baseline

결과
	•	운영 및 보안 추적의 초기 기반 확보

⸻

Task I-11. operation/event/webhook 확장 skeleton 구현

목적

후속 확장을 위해 async/event/webhook 구조를 막지 않는 최소 skeleton을 구현한다.

주요 내용
	•	operations resource placeholder 또는 skeleton
	•	async accepted response flow
	•	internal event publish interface placeholder
	•	webhook subscription/delivery placeholder route or module boundary
	•	현재는 stub 수준이어도 구조는 고정

결과
	•	이후 Phase에서 async/integration 기능을 자연스럽게 붙일 수 있음

⸻

Task I-12. AI/RAG 확장 read-model 준비

목적

AI/RAG가 사용할 구조적 조회 인터페이스를 위한 최소 read-model 준비를 한다.

주요 내용
	•	document/version/node 조회 응답이 citation-friendly identifier를 담을 수 있는 구조 확보
	•	structure-aware read helper
	•	retrieval endpoint placeholder boundary
	•	indexing status / ingestion status를 붙일 수 있는 확장 위치 설계 반영
	•	AI 전용 우회 API 없이 공통 리소스를 재사용하는 패턴 고정

결과
	•	후속 AI/RAG 확장이 기존 API 위에서 가능해짐

⸻

5. 권장 구현 순서

가장 추천하는 순서는 아래와 같다.

1차 구현 묶음: 공통 기반
	•	Task I-1 API skeleton 및 버전 라우팅 구조 구현
	•	Task I-2 공통 응답 envelope 및 response helper 구현
	•	Task I-3 공통 오류 모델 및 exception handling 계층 구현
	•	Task I-4 request context / trace context / auth context plumbing 구현

2차 구현 묶음: 보안 및 목록 규약
	•	Task I-5 인증 컨텍스트 추출 및 authorization hook point 구현
	•	Task I-6 list query parser 및 pagination/filter/sort validator 구현

3차 구현 묶음: 핵심 도메인 API
	•	Task I-7 documents 핵심 API 구현
	•	Task I-8 versions / nodes API 구현

4차 구현 묶음: 안정성 및 운영성
	•	Task I-9 write endpoint idempotency hook 구현
	•	Task I-10 audit / observability 연결 포인트 구현

5차 구현 묶음: 확장 기반
	•	Task I-11 operation/event/webhook 확장 skeleton 구현
	•	Task I-12 AI/RAG 확장 read-model 준비

⸻

6. 구현 완료 기준

이 구현 단계는 아래를 만족하면 1차 완료로 본다.
	•	/api/v1 기반 라우팅 구조가 존재한다.
	•	공통 응답/오류 처리 계층이 작동한다.
	•	request_id/trace/auth context 기반이 존재한다.
	•	documents / versions / nodes 핵심 API가 최소 수준으로 동작한다.
	•	목록 조회 규약이 공통 parser/validator로 연결된다.
	•	일부 write endpoint에서 idempotency hook이 동작한다.
	•	audit/observability 연결 포인트가 있다.
	•	async/webhook/AI 확장을 막지 않는 skeleton이 존재한다.