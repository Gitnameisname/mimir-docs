Task 3-6. Pagination / Filtering / Sorting 규약 설계

Claude Code 작업 지시서

1. 작업명

Phase 3 - Task 3-6. Pagination / Filtering / Sorting 규약 설계

⸻

2. 작업 목적

플랫폼 API의 목록 조회 계열에서 공통적으로 사용할 pagination / filtering / sorting 규약을 설계 문서로 정리하라.

이번 작업의 목적은 단순히 page=1 같은 쿼리 파라미터를 정하는 것이 아니다.
핵심은 다음이다.
	•	문서, 버전, 노드, 감사 로그, 이벤트, 검색 결과, 웹훅 전달 이력 등 다양한 목록 조회에 적용 가능한 일관된 탐색 규약 정의
	•	User UI, Admin UI, 외부 시스템, AI/RAG 소비자, 내부 운영 도구가 같은 방식으로 목록을 조회하고 후속 페이지를 탐색할 수 있게 함
	•	대용량 데이터, 권한 기반 필터링, 시간순 정렬, 상태 기반 조회, 검색 결과 목록까지 수용 가능한 구조를 마련
	•	이후 구현 단계에서 엔드포인트마다 각기 다른 query parameter 문법이 생기는 것을 방지

즉, 이 문서는 목록 탐색 계약(list traversal contract) 의 기준 문서여야 한다.

⸻

3. 반드시 반영할 배경

이 플랫폼은 다음 성격을 가진다.
	•	범용 문서 플랫폼
	•	리소스 중심 REST API 구조
	•	조직/권한/감사 추적 필요
	•	문서/버전/노드/권한/감사 로그/이벤트/웹훅/jobs 등 다양한 리소스 존재
	•	User/Admin UI 분리
	•	외부 시스템 연동 필요
	•	AI/RAG/Agent도 목록 조회 API를 사용할 수 있음
	•	향후 검색, retrieval, activity trace, operations tracking까지 확장 예정

이전 Task에서 이미 다음이 정해졌다고 가정하라.
	•	플랫폼 API는 계약 계층이다.
	•	리소스 구조와 명명 규칙이 존재한다.
	•	인증/인가 구조가 API 계층에 반영된다.
	•	버전 전략이 존재한다.
	•	공통 요청/응답 포맷 및 metadata 배치 방향이 정해졌다.

따라서 이번 작업은 단순 UI 편의 기능이 아니라,
모든 collection endpoint의 일관된 조회 문법과 응답 의미를 정하는 작업이다.

⸻

4. 이번 작업의 핵심 질문

이번 문서에서는 최소한 아래 질문들에 답할 수 있어야 한다.
	1.	이 플랫폼의 기본 pagination 방식은 offset 기반이어야 하는가, cursor 기반이어야 하는가?
	2.	모든 목록 조회에 하나의 pagination 방식을 강제할 것인가, 리소스 특성에 따라 혼합할 것인가?
	3.	filtering 문법은 단순 key=value 수준으로 갈 것인가, 더 구조화된 문법을 둘 것인가?
	4.	상태, 날짜 범위, 작성자, 조직, 타입, 메타데이터 필터를 어떻게 일관되게 표현할 것인가?
	5.	sorting은 어떤 문법으로 표현할 것인가?
	6.	다중 정렬, 안정 정렬, cursor pagination과 정렬의 관계를 어떻게 정의할 것인가?
	7.	authorization-aware filtering이 적용될 때 결과 의미를 어떻게 해석할 것인가?
	8.	검색 결과와 일반 목록 조회는 같은 규약을 공유할 수 있는가?
	9.	대용량 audit/event/job 목록과 일반 문서 목록을 같은 원칙으로 다룰 수 있는가?
	10.	응답의 meta.pagination / filter / sort 정보를 어떤 철학으로 넣어야 하는가?

⸻

5. 상세 작업 요구사항

5-1. 문서 작성

아래 산출물을 작성하라.

필수 산출물
	•	Task6_list_query_conventions.md

문서에 반드시 포함할 섹션
	1.	문서 목적
	2.	목록 조회 규약이 필요한 이유
	3.	상위 설계 원칙
	4.	Pagination 방식 후보 비교
	5.	권장 pagination 전략
	6.	Pagination query parameter 규약
	7.	Filtering 문법 규약
	8.	Sorting 문법 규약
	9.	Pagination / Filtering / Sorting 조합 원칙
	10.	검색 결과와 일반 목록 조회의 관계
	11.	권한/가시성 필터링 관점
	12.	응답 metadata 연계 원칙
	13.	예시 query 및 응답 metadata
	14.	후속 Task에 전달할 설계 기준
	15.	결론

⸻

5-2. 목록 조회 규약이 필요한 이유 정리

문서 초반에서 다음을 명확히 설명하라.
	•	목록 조회는 플랫폼 전반에서 매우 자주 사용되는 공통 패턴이다.
	•	엔드포인트마다 paging/filter/sort 방식이 다르면 UI, SDK, 외부 연동, 테스트, AI agent의 구현 비용이 급증한다.
	•	문서 플랫폼은 단순 문서 목록뿐 아니라 감사 로그, 이벤트, 검색 결과, 작업 이력 등 성격이 다른 목록이 많기 때문에 공통 규약이 더 중요하다.
	•	목록 조회 규약은 성능, 사용성, 확장성, 보안, 일관성에 모두 영향을 준다.

즉, 이 규약은 단순 편의 기능이 아니라
플랫폼 탐색성과 운영성을 좌우하는 핵심 표준이라는 점이 드러나야 한다.

⸻

5-3. 상위 설계 원칙 정의

최소한 아래 원칙을 포함하라.
	•	Consistent query language across collections
	•	Stable traversal semantics
	•	Predictable server behavior
	•	Scalability-aware pagination
	•	Explicit filtering and sorting
	•	Authorization-safe listing
	•	Minimal ambiguity
	•	Extensible query model
	•	Metadata-backed navigation

각 원칙마다 아래를 설명하라.
	•	의미
	•	왜 필요한지
	•	실제 목록 조회 설계에 어떤 영향을 주는지

예:
	•	Stable traversal semantics → 페이지를 넘길 때 결과가 갑자기 중복/누락되지 않도록 정렬과 pagination 관계를 명확히 해야 함
	•	Authorization-safe listing → 권한 때문에 보이지 않는 항목이 있어도 응답 구조는 일관되어야 함
	•	Extensible query model → 지금은 간단한 필터만 있어도 나중에 상태/날짜/메타데이터 필터를 확장할 수 있어야 함

⸻

5-4. Pagination 방식 후보 비교

다음 pagination 방식을 비교하라.

후보 1. Offset pagination
예:
	•	?page=1&page_size=20
	•	?offset=40&limit=20

후보 2. Cursor pagination
예:
	•	?cursor=...&limit=20

후보 3. 혼합 전략
예:
	•	일반 관리 목록은 offset
	•	대용량/시간순 목록은 cursor

각 방식에 대해 다음을 비교하라.
	•	구현 단순성
	•	대용량 데이터 적합성
	•	정렬 안정성
	•	실시간 변경 데이터에 대한 내성
	•	UI 사용성
	•	외부 시스템 사용성
	•	검색 결과와의 적합성
	•	이 플랫폼에서의 적합성

중요:
최종적으로 기본 권장 전략을 제시하라.
문서 플랫폼의 특성을 고려하면, 모든 목록에 단일 방식을 강제할지, 리소스 특성에 따라 혼합할지 정책 방향을 분명히 해야 한다.

⸻

5-5. 권장 pagination 전략 제안

문서에서는 최소한 아래를 정리하라.
	•	기본(default) pagination 방식
	•	어떤 컬렉션에서 offset이 적절한지
	•	어떤 컬렉션에서 cursor가 더 적절한지
	•	page size / limit 기본값 및 최대값을 둘지 여부
	•	대량 데이터 보호를 위한 상한 정책 방향
	•	페이지네이션 미지정 시 기본 동작
	•	비권장 패턴

검토해야 할 리소스 예시:
	•	documents
	•	versions
	•	nodes
	•	audit-logs
	•	events
	•	webhook-deliveries
	•	jobs
	•	search results

중요:
초기 구현 난이도와 장기 확장성을 함께 고려하라.
예를 들어 관리 화면 친화성과 대규모 로그 탐색 요구가 서로 다를 수 있다는 점을 반영하라.

⸻

5-6. Pagination query parameter 규약

아래를 정리하라.
	•	page, page_size를 쓸지
	•	limit, offset을 쓸지
	•	cursor, limit을 쓸지
	•	파라미터 명칭 표준화
	•	최대 허용 개수
	•	음수/0/과대 요청 처리 방향
	•	잘못된 cursor 처리 원칙
	•	first page / next page / previous page 표현 방향
	•	응답 metadata 또는 link와의 연결 방식

중요:
파라미터 명칭은 후속 구현자가 바로 쓸 수 있도록 충분히 명확해야 한다.

⸻

5-7. Filtering 문법 규약

이 부분은 매우 중요하다.
플랫폼에서 공통적으로 사용할 필터 표현 원칙을 정하라.

최소한 아래 항목을 다뤄라.

기본 단일 필터
	•	?status=active
	•	?document_type=policy

다중 값 필터
	•	?status=active,archived
	•	반복 파라미터 방식 허용 여부 검토

범위 필터
	•	날짜 범위
	•	숫자 범위
	•	예: created_after / created_before 같은 방식 또는 gte/lte 문법

메타데이터 필터
	•	metadata 확장 구조에 대한 필터링 방향
	•	지나치게 복잡한 질의문법을 초기부터 도입할지 여부

텍스트 검색성 필터
	•	q 같은 자유 검색 파라미터를 일반 필터와 어떻게 구분할지

중요:
초기 단계에서 과도하게 복잡한 DSL을 도입하지 않도록 하되,
향후 확장 가능성을 열어두는 방향으로 정리하라.

⸻

5-8. Filtering 필드 명명 및 의미 규칙

다음을 문서화하라.
	•	상태값 필터 표기
	•	작성자/수정자 필터 표기
	•	조직/owner 관련 필터 표기
	•	날짜 범위 필터 표기
	•	boolean 필터 표기
	•	태그/라벨 필터 표기
	•	metadata field filter 확장 방식
	•	리소스마다 이름이 달라지는 것을 어느 정도 허용할지

예:
	•	created_at_from vs created_after
	•	updated_before
	•	has_attachments=true
	•	tag=...
	•	visibility=...

중요:
필터 이름 자체도 플랫폼 공통 언어가 되어야 한다는 관점으로 작성하라.

⸻

5-9. Sorting 문법 규약

다음을 정리하라.
	•	정렬 파라미터 이름
	•	오름차순/내림차순 표기 방식
	•	다중 정렬 허용 여부
	•	기본 정렬 규칙
	•	안정 정렬 보장을 위한 tie-breaker 필요성
	•	지원하지 않는 정렬 필드 요청 처리 방식

예:
	•	?sort=created_at
	•	?sort=-created_at
	•	?sort=type,-updated_at

중요:
sorting은 pagination과 강하게 연결되므로,
특히 cursor 방식에서 어떤 정렬만 허용할지 방향을 설명하라.

⸻

5-10. Pagination / Filtering / Sorting 조합 원칙

반드시 아래를 정리하라.
	•	filter 적용 후 sort 후 paginate 하는 기본 개념
	•	cursor pagination과 sorting 조합 제약
	•	total count가 비싸거나 부정확할 수 있는 경우 처리 방향
	•	filter/sort가 바뀌면 cursor 재사용이 무효화될 수 있음을 어떻게 볼지
	•	서버가 허용하지 않는 조합 요청 시 어떤 방향으로 실패시킬지

즉, 각각을 따로 정의하는 데 그치지 말고
셋이 함께 작동하는 규칙을 문서화하라.

⸻

5-11. 검색 결과와 일반 목록 조회의 관계

플랫폼은 일반 collection listing과 search/retrieval 결과를 모두 다룬다.
다음을 검토하라.
	•	일반 목록 조회와 검색 결과가 같은 pagination 규약을 공유할 수 있는지
	•	q 파라미터 기반 단순 검색과 별도 /searches 리소스의 역할 차이
	•	relevance sort와 일반 정렬의 차이
	•	search result metadata에 추가로 필요한 정보가 무엇인지
	•	검색 결과가 일반 목록 응답 envelope를 어느 정도 공유할 수 있는지

중요:
검색은 특수성이 있지만, 가능한 한 플랫폼 공통 규약과 충돌하지 않게 설계해야 한다.

⸻

5-12. 권한/가시성 필터링 관점

다음을 반드시 다뤄라.
	•	목록 조회는 authorization-aware해야 함
	•	사용자가 접근 불가능한 항목은 결과에서 제외될 수 있음
	•	이 때문에 total count 의미가 달라질 수 있음
	•	“숨겨진 결과”를 응답에 직접 노출하지 않는 것이 원칙인지 검토
	•	관리자와 일반 사용자에서 같은 filter query라도 결과가 다를 수 있음
	•	권한 필터링은 filtering 문법의 일부가 아니라 보안 enforcement라는 점을 명확히 할 것

중요:
권한 필터링과 사용자 지정 filtering을 개념적으로 구분하라.

⸻

5-13. 응답 metadata 연계 원칙

Task 3-5와 연결하여 다음을 정리하라.
	•	pagination metadata에 어떤 필드를 둘지
	•	applied filters를 반영할지
	•	applied sort를 반영할지
	•	next_cursor / prev_cursor / has_next 등 표현 방향
	•	total / estimated_total / count 같은 표현 구분 필요성
	•	warnings나 truncation 정보 필요 여부

이번 Task는 query 문법 중심이지만,
응답 메타와의 연결 방식도 최소한 문서화하라.

⸻

5-14. 예시 query 및 응답 metadata 작성

문서에 최소한 아래 예시를 포함하라.

pagination 예시
	•	offset/page 기반 예시
	•	cursor 기반 예시

filtering 예시
	•	상태 필터
	•	날짜 범위 필터
	•	다중 값 필터
	•	태그/boolean 필터

sorting 예시
	•	단일 정렬
	•	다중 정렬
	•	내림차순 예시

조합 예시
	•	filter + sort + pagination
	•	search + sort + cursor
	•	audit log listing 예시

응답 예시는 full JSON이 아니라도 되지만,
meta.pagination 구조 예시는 꼭 포함하라.

⸻

5-15. 후속 Task에 전달할 설계 기준

문서 마지막에 다음 연결 포인트를 정리하라.

예:
	•	Task 3-7에서는 idempotency와 직접 연결은 적지만, 대규모 bulk 작업 결과 조회와의 연결을 고려할 수 있음
	•	Task 3-8에서는 jobs/events/webhooks 같은 운영성 리소스에 어떤 pagination 전략을 적용할지 구체화해야 함
	•	Task 3-9에서는 retrieval/search 결과가 이 규약을 얼마나 재사용할지 정리해야 함
	•	Task 3-10에서는 잘못된 pagination/filter/sort 요청에 대한 error model을 구체화해야 함
	•	구현 Phase에서는 query parser, validation, repository query builder, OpenAPI 문서화에 직접 반영해야 함

⸻

6. 산출물

아래 파일을 작성하라.

필수 산출물
	•	Task6_list_query_conventions.md

문서 성격
	•	설계 기준 문서
	•	구현 코드 금지
	•	ORM/DB 의존 설명 최소화
	•	장기적으로 재사용 가능한 플랫폼 query 규약 문서

⸻

7. 제외 범위

이번 작업에서는 아래는 하지 마라.
	•	실제 query parser 코드 작성
	•	SQL/ORM query builder 구현
	•	검색 엔진 DSL 상세 설계
	•	full text search ranking algorithm 설계
	•	OpenAPI parameter schema 전체 작성
	•	응답 JSON 스키마 전체 작성
	•	permission filtering 로직 구현
	•	DB index 설계 상세화
	•	metadata filtering DSL 상세 설계
	•	query language를 완전한 mini-language로 확장하는 작업

즉, 이번 작업은 목록 조회용 pagination/filtering/sorting 규약의 상위 기준 수립에 집중하라.

⸻

8. 작성 스타일 가이드

문서는 아래 스타일을 따르라.
	•	실용적인 규약 문서로 작성
	•	단순함과 확장성의 균형을 고려
	•	후보 비교 후 권장안을 명확히 제시
	•	권장 / 허용 / 비권장 / 금지 / 후속 Task에서 구체화를 구분
	•	문서, 로그, 이벤트, 검색 등 서로 성격이 다른 목록을 함께 고려
	•	예시 query를 충분히 제공하되 문서가 예시 모음집이 되지 않도록 할 것

⸻

9. 검토 체크리스트

작업 완료 전에 아래를 스스로 점검하라.
	•	pagination 방식 비교와 권장안이 명확한가?
	•	filtering 문법이 과도하게 복잡하지 않으면서 확장 가능하게 정리되었는가?
	•	sorting 문법과 안정 정렬 원칙이 있는가?
	•	pagination/filter/sort 조합 규칙이 설명되었는가?
	•	검색 결과와 일반 목록의 관계가 정리되었는가?
	•	권한 기반 가시성 필터링 관점이 포함되었는가?
	•	응답 metadata와 연결되는가?
	•	후속 Task 및 구현 단계가 이 문서를 기준으로 이어질 수 있는가?

⸻

10. Claude Code에 대한 최종 지시

위 요구사항을 반영하여 Task6_list_query_conventions.md 문서를 작성하라.

문서는 범용 문서 플랫폼 API의 모든 collection endpoint에서 재사용 가능한
pagination / filtering / sorting의 공통 규약 문서여야 한다.

문서 작성 후에는 아래 형식의 자체 점검 요약도 함께 정리하라.

자체 점검 요약
	•	권장 pagination 전략
	•	filtering 문법 핵심 요약
	•	sorting 문법 핵심 요약
	•	search/list 관계 요약
	•	권한 기반 listing 관점 요약
	•	후속 Task 연결 포인트
	•	의도적으로 후속 Task로 남긴 미결정 사항