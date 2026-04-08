Task 2-6 작업 지시서

주제

행위 추적(Activity Trace) 모델 설계

작업 목적

Task 2-5에서 감사 로그(Audit Log) 체계를 설계했다면, 이번 단계에서는 별도의 목적을 가진 행위 추적(Activity Trace) 모델을 설계한다.
이 모델은 보안/컴플라이언스 중심의 감사 로그와 달리, 사용자와 시스템이 리소스를 어떻게 탐색하고, 조회하고, 수정 흐름을 이어가며, 어떤 작업 맥락 안에서 행동했는지를 연결해서 볼 수 있도록 만드는 것이 목표다.

즉, 이번 작업은 단건 이벤트를 남기는 것이 아니라,
행동 흐름, 작업 세션, 탐색 경로, 상호작용 패턴, 운영 분석 가능성을 위한 추적 모델을 정의하는 단계다.

이 결과는 이후 운영 분석, UX 개선, 협업 흐름 이해, AI 도구 연계 분석, 이상 행위 탐지, 업무 재현성 확보의 기반이 된다.

⸻

이번 작업의 범위

이번 작업에서는 아래를 다룬다.
	1.	Activity Trace의 목적 정의
	•	사용자 행동 흐름 추적
	•	작업 세션 이해
	•	탐색 경로 및 문서 이용 패턴 분석
	•	작업 재현성 및 맥락 연결
	•	운영 분석 / UX 개선 / AI 협업 분석 기반
	2.	Trace와 Event의 관계 정의
	•	Trace
	•	Span 또는 Step 필요 여부
	•	Event
	•	Session / Interaction boundary
	3.	행위 이벤트 범주 정의
	•	탐색/조회
	•	검색/필터링
	•	편집 시작/중단/완료
	•	문서 간 이동
	•	첨부/댓글/버전 관련 상호작용
	•	API/외부 도구/자동화 행위 흐름
	4.	Activity Trace 레코드 구조 설계
	•	actor
	•	session
	•	trace id / correlation id
	•	action/event
	•	target resource
	•	client / source context
	•	timing / duration
	•	result / outcome
	5.	감사 로그와의 분리 원칙 정의
	•	무엇은 audit로
	•	무엇은 activity로
	•	무엇은 둘 다 기록해야 하는가

⸻

이번 작업에서 하지 말 것

아래는 이번 단계에서 다루지 않는다.
	•	실제 이벤트 수집 SDK 구현
	•	프론트엔드 telemetry 코드 작성
	•	클릭스트림 수집 시스템 구현
	•	analytics dashboard 구현
	•	anomaly detection 모델 구현
	•	audit log 스키마 재설계
	•	개인정보 비식별화 구현 코드

즉, 이번 단계는 행위 추적 모델과 경계 설계 단계다.

⸻

설계 전제

아래 전제를 반영하여 설계할 것.
	1.	플랫폼은 범용 문서 플랫폼이며, 문서 조회/편집뿐 아니라 검색, 비교, 검토, 승인, 협업, 외부 API 호출 등 다양한 상호작용이 존재한다.
	2.	감사 로그는 보안/통제/증적 목적이고, Activity Trace는 흐름/맥락/이용 패턴 목적이다.
	3.	API-first 구조이므로 행위 추적은 웹 UI뿐 아니라 관리자 UI, 외부 API, 서비스 계정, AI 도구 호출까지 포괄할 수 있어야 한다.
	4.	향후 UX 개선, 협업 분석, AI assistant 작업 분석, 자동화 워크플로 추적, 성능/병목 파악에 활용될 수 있어야 한다.
	5.	모든 세부 클릭을 무한정 저장하면 비용과 프라이버시 문제가 생기므로, 추적 단위와 보존 수준을 합리적으로 제한해야 한다.
	6.	trace/correlation id를 활용하여 audit log, API request log, background job, integration action과 연결 가능해야 한다.

⸻

Claude Code가 수행해야 할 작업

아래 순서대로 정리할 것.

1. Activity Trace의 역할 정의

먼저 아래 질문에 답할 것.
	•	Activity Trace는 감사 로그와 무엇이 다른가
	•	운영 로그/분석 로그/제품 사용성 로그와는 어떤 관계를 가지는가
	•	왜 단순 event log만으로는 부족한가
	•	“흐름”을 보기 위해 어떤 모델이 필요한가

그리고 Activity Trace의 핵심 목적을 아래 범주로 정리할 것.
	•	사용자 작업 흐름 이해
	•	탐색 경로 분석
	•	협업/운영 패턴 분석
	•	UX 개선 인사이트 확보
	•	AI/자동화 작업 재현성 확보
	•	이상 행동 탐지의 입력 데이터 확보

⸻

2. Trace / Session / Event / Span 개념 관계 정리

최소한 아래 개념을 검토할 것.
	•	Interaction Session
	•	사용자의 작업 단위 세션
	•	Trace
	•	하나의 작업 흐름을 묶는 식별자
	•	Span / Step
	•	trace 내부의 하위 구간 또는 단계
	•	Event
	•	개별 행동 기록
	•	Correlation ID
	•	시스템 전반 연결용 식별자

각 개념에 대해 아래를 정리할 것.
	•	왜 필요한가
	•	audit log와 공유 가능한가
	•	MVP에서 반드시 필요한가
	•	프론트엔드/백엔드/API 어디서 생성되는 것이 자연스러운가

그리고 아래 질문에도 답할 것.
	•	사용자 로그인 세션과 작업 trace는 같은 것인가
	•	하나의 검색 → 문서 열람 → 댓글 작성 흐름은 하나의 trace인가
	•	하나의 API 요청은 event인가 span인가
	•	background job이나 AI agent 실행도 trace로 묶을 수 있는가

⸻

3. Activity Event 범주 정의

최소한 아래 이벤트군을 검토할 것.

탐색/조회 계열
	•	document opened
	•	version viewed
	•	node expanded / collapsed
	•	attachment previewed
	•	comment panel opened
	•	document compared

검색/발견 계열
	•	search executed
	•	filter changed
	•	search result opened
	•	saved query used
	•	recommendation clicked

편집/작업 계열
	•	edit started
	•	draft autosaved
	•	edit committed
	•	edit abandoned
	•	comment drafted
	•	comment submitted
	•	attachment uploaded
	•	version restored

협업/공유 계열
	•	share action initiated
	•	mention used
	•	review requested
	•	approval action taken
	•	external access used

시스템/자동화 계열
	•	api client request grouped
	•	service account workflow executed
	•	AI-assisted action started/completed
	•	connector sync triggered
	•	export job initiated/completed

각 이벤트군에 대해 아래를 정리할 것.
	•	왜 activity 추적 대상인가
	•	감사 로그에도 함께 남겨야 하는가
	•	MVP 필수인지, 후속 확장인지

⸻

4. Activity Trace 데이터 구조 초안

최소한 아래 필드를 포함하는 개념 구조를 설계할 것.

Actor / Session
	•	actor_type
	•	actor_id
	•	acting_org_id
	•	session_id 또는 interaction_session_id
	•	client_id / device_id 필요 여부

Trace / Event
	•	trace_id
	•	parent_trace_id 필요 여부
	•	span_id 또는 step_id 필요 여부
	•	event_type
	•	event_category
	•	occurred_at
	•	duration_ms 필요 여부

Target
	•	resource_type
	•	resource_id
	•	parent_resource_type / id
	•	resource_scope
	•	query_text 또는 filter_summary 필요 여부

Context
	•	request_id
	•	correlation_id
	•	source_channel (user UI / admin UI / api / service account / AI agent)
	•	route / endpoint
	•	surface or screen identifier 필요 여부

Outcome
	•	outcome_type
	•	success/failure/cancelled
	•	result_summary
	•	item_count / changed_field_count 같은 요약치 필요 여부

각 필드에 대해 아래를 정리할 것.
	•	왜 필요한가
	•	필수인지 선택인지
	•	민감도 주의가 필요한지
	•	MVP 포함 여부

⸻

5. 흐름 중심 모델링 원칙 정리

아래 질문에 답할 것.
	•	단순 document.view 이벤트만 남기면 무엇이 부족한가
	•	search → open → edit → save → export 같은 흐름을 어떻게 연결할 것인가
	•	검색어 변경을 각각 event로 볼지, 하나의 search session으로 묶을지
	•	autosave 같은 고빈도 이벤트는 모두 저장해야 하는가
	•	사용자의 “작업 의도”를 어떤 수준까지 추론 가능한 구조로 남겨야 하는가

그리고 아래 형태로 원칙을 정리할 것.
	•	trace로 묶어야 하는 행동
	•	단일 event로 충분한 행동
	•	span/step이 필요한 흐름
	•	샘플링 또는 요약이 필요한 고빈도 이벤트

⸻

6. 감사 로그와의 분리 및 중복 기록 원칙

Task 2-5와 연결해서 아래를 정리할 것.
	•	어떤 이벤트는 audit만 필요하고 activity는 불필요한가
	•	어떤 이벤트는 activity만 필요하고 audit는 불필요한가
	•	어떤 이벤트는 둘 다 남겨야 하는가
	•	동일 행위를 두 시스템에 남길 때 어떤 필드를 공유해야 하는가
	•	trace_id, correlation_id, actor, resource를 어떻게 맞출 것인가

예시를 검토할 것.
	•	document opened
	•	search executed
	•	document updated
	•	permission changed
	•	login failed
	•	export initiated
	•	AI agent가 문서를 읽고 요약 생성

그리고 최종적으로 아래를 정리할 것.
	•	audit 전용 이벤트
	•	activity 전용 이벤트
	•	이중 기록 이벤트
	•	공유 식별자 규칙

⸻

7. 저장 단위와 집계 단위 설계

아래 질문에 답할 것.
	•	Activity Trace는 event 단위 저장이 기본인가
	•	trace summary를 별도로 가져야 하는가
	•	긴 작업 흐름은 종료 시점 summary를 만들 필요가 있는가
	•	list/search 같은 반복 행위는 raw event와 aggregate 둘 다 필요한가
	•	운영 분석용 집계와 원시 추적 데이터를 분리해야 하는가

그리고 아래를 제안할 것.
	•	raw event 구조
	•	trace summary 구조
	•	세션 요약 구조 필요 여부
	•	향후 분석 파이프라인으로 넘기기 좋은 구조

⸻

8. 프라이버시 / 비용 / 보존 원칙

아래 질문에 답할 것.
	•	query_text나 검색어를 원문 저장해도 되는가
	•	문서 제목/경로/식별자는 어느 수준까지 저장할 것인가
	•	화면 이동, 클릭, 스크롤까지 전부 수집해야 하는가
	•	autosave, typing, hover 같은 이벤트는 과한가
	•	activity trace 보존 기간은 audit log와 같아야 하는가
	•	집계본과 원시본의 보존 정책을 다르게 둘 수 있는가

그리고 아래를 정리할 것.
	•	최소 수집 원칙
	•	민감 필드 처리 원칙
	•	고빈도 이벤트 억제 원칙
	•	보존 기간 초안
	•	비용 통제 원칙

⸻

9. 외부 API / AI 도구 / 자동화 흐름 반영

API-first 플랫폼이므로 아래를 검토할 것.
	•	외부 API 클라이언트의 다수 요청을 하나의 trace로 묶을 수 있는가
	•	서비스 계정 배치 작업은 session 개념이 없는 경우 어떻게 처리할 것인가
	•	AI assistant가 여러 문서를 검색/읽기/요약하는 흐름은 어떻게 표현할 것인가
	•	background job과 사용자 initiated 작업을 어떻게 구분할 것인가
	•	사람 사용자와 비인간 주체의 activity trace를 동일 모델로 다룰 수 있는가

그리고 MVP에서 어느 수준까지 반영할지 정리할 것.

⸻

10. MVP Activity Trace 범위 제한안 제시

MVP에서 반드시 수집할 행위와 후속 확장으로 넘길 것을 나눌 것.

예시 관점:

MVP 필수
	•	search executed
	•	document opened
	•	version viewed
	•	edit started / committed
	•	comment submitted
	•	export initiated/completed
	•	API request grouped by trace
	•	AI-assisted flow 시작/종료

후속 확장
	•	세밀한 UI 패널 열기/닫기
	•	recommendation interaction
	•	hover/scroll telemetry
	•	typing/autosave 세부 이벤트
	•	advanced collaboration event graph
	•	cross-system workflow lineage

중요:
초기에는 의미 있는 작업 흐름만 잡고, 잡음이 많은 미세 UI telemetry는 제외하는 방향을 우선 검토할 것.

⸻

11. 다음 단계 입력 자료 형태로 정리

Task 2-7 정책 적용 우선순위와 예외 규칙 정리로 넘기기 위해 아래를 정리할 것.
	•	activity trace가 정책 위반 탐지에 어떤 힌트를 줄 수 있는가
	•	반복된 접근 시도, 비정상 탐색 패턴, 대량 export 흐름 등을 어떻게 포착할 수 있는가
	•	audit와 activity를 조합해 어떤 정책 기반 판단이 가능해지는가

⸻

산출물 형식

이번 작업 결과물은 설계 문서 초안이어야 하며, 아래 구조를 따를 것.

Phase 2 - Task 2-6

1. 목표

2. 설계 전제

3. Activity Trace의 역할과 범위

4. Trace / Session / Event / Span 개념 모델

5. Activity Event 범주

6. Activity Trace 데이터 구조

7. 흐름 중심 모델링 원칙

8. Audit Log와의 분리 및 연계 원칙

9. 저장 단위와 집계 단위

10. 프라이버시 / 비용 / 보존 원칙

11. 외부 API / AI 도구 / 자동화 반영 원칙

12. MVP Activity Trace 범위 제한안

13. 다음 Task로 넘길 결정사항

14. 오픈 이슈

⸻

의사결정 원칙

설계 중 아래 원칙을 반드시 지킬 것.
	1.	감사와 행위 추적을 분리할 것
	•	audit는 통제/증적
	•	activity는 흐름/맥락/분석
	2.	흐름이 보이도록 설계할 것
	•	event를 흩뿌리는 것이 아니라 trace로 연결 가능한 구조를 우선할 것
	3.	의미 있는 행동만 우선 수집
	•	모든 UI 미세 동작을 다 담지 말고 작업 의미가 있는 이벤트부터 시작할 것
	4.	API-first / 사람+비인간 주체 모두 지원
	•	웹 사용자뿐 아니라 서비스 계정, AI agent, background job도 다룰 수 있어야 함
	5.	프라이버시와 비용을 같이 고려
	•	분석 가치가 높지 않은 고빈도 이벤트는 최소화할 것
	6.	정책/이상행동 분석으로 확장 가능하게
	•	trace_id, correlation_id, actor/resource 식별자를 일관되게 유지할 것

⸻

기대 결과

이 작업이 끝나면 최소한 아래가 확정되어 있어야 한다.
	•	Activity Trace가 감사 로그와 구분되는 목적
	•	trace / session / event / span의 개념 관계
	•	activity event taxonomy 초안
	•	흐름 중심의 activity data 구조
	•	audit와 activity의 중복 기록 원칙
	•	프라이버시/비용/보존 기준
	•	사람 사용자, API 클라이언트, AI 도구를 포괄하는 추적 방향
	•	Task 2-7 정책 예외/우선순위 설계에 넘길 분석 입력 기준

⸻

금지 사항
	•	클릭스트림 분석 시스템 구현으로 넘어가지 말 것
	•	audit log와 activity trace를 다시 하나로 합치지 말 것
	•	전 UI 행동을 무조건 수집하는 방향으로 설계하지 말 것
	•	제품 분석 도구 전체 아키텍처까지 확장하지 말 것
	•	anomaly detection 로직 자체를 이번 단계에서 구현하려 하지 말 것

⸻

작업 완료 판단 기준

아래 조건을 만족하면 완료로 본다.
	•	Activity Trace의 역할과 Audit Log와의 차이가 명확히 설명되어 있다
	•	trace/session/event/span 개념이 정리되어 있다
	•	activity event 범주와 데이터 구조가 제시되어 있다
	•	흐름 중심 모델링 원칙과 저장/집계 원칙이 정리되어 있다
	•	프라이버시/비용/보존 원칙이 포함되어 있다
	•	Task 2-7로 넘길 수 있는 정책 분석 연계 포인트가 정리되어 있다