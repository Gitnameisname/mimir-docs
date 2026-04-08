Task 3-12. Phase 3 통합 설계 정리 및 다음 Phase 인계 문서 작성

Claude Code 작업 지시서

1. 작업명

Phase 3 - Task 3-12. Phase 3 통합 설계 정리 및 다음 Phase 인계 문서 작성

⸻

2. 작업 목적

Phase 3에서 작성된 모든 API 설계 산출물을 하나의 일관된 기준 체계로 통합하고,
다음 구현 Phase에서 바로 사용할 수 있도록 통합 마스터 문서와 구현 인계 문서를 작성하라.

이번 작업의 목적은 단순 요약본을 만드는 것이 아니다.
핵심은 다음이다.
	•	Task 3-1 ~ 3-11에서 정리한 API 원칙, 리소스 구조, 인증/인가, 버전, 응답 포맷, 목록 조회 규약, idempotency, async/event/webhook, AI/RAG 인터페이스, 오류 모델, 대표 시나리오를 하나의 설계 체계로 정합화
	•	문서 간 중복, 충돌, 표현 차이, 암묵적 가정, 미결정 사항을 정리
	•	다음 Phase 구현팀 또는 Claude Code가 무엇부터 어떻게 구현해야 하는지 바로 이해할 수 있도록 기준선 제공
	•	“설계 문서 모음집”이 아니라, 실제 구현을 이끄는 마스터 기준 문서 + 실무 인계 문서를 완성

즉, 이 작업의 결과물은
Phase 3 전체의 최종 설계 기준선이자
Phase 4 이후 구현 작업의 출발점이어야 한다.

⸻

3. 반드시 반영할 배경

이 플랫폼은 다음 성격을 가진다.
	•	범용 문서 플랫폼
	•	문서 / 버전 / 노드 중심 도메인 구조
	•	조직 / 사용자 / 역할 / ACL 기반 권한 구조
	•	감사 추적 필요
	•	REST 중심 API-first 구조
	•	User/Admin UI 분리
	•	외부 시스템 연동 필요
	•	AI/RAG/Agent도 공식 API 소비자
	•	async operation / event / webhook 확장 구조 필요
	•	장기적 버전 관리, 오류 모델, observability까지 포함한 플랫폼 API 필요

Phase 3에서는 이미 다음 주제를 각각 문서화했다고 가정하라.
	•	플랫폼 API 아키텍처 원칙
	•	핵심 리소스 기반 REST 구조
	•	인증/인가 구조의 API 계층 반영
	•	API 버전 관리 전략
	•	공통 요청/응답 포맷
	•	pagination/filter/sort 규약
	•	idempotency 및 retry safety
	•	async/event/webhook 확장 모델
	•	AI/RAG 연계 가능한 API 인터페이스
	•	오류 모델 및 운영 관측성 규약
	•	대표 API 시나리오

이번 작업은 이들을 다시 새로 설계하는 것이 아니라,
하나의 통합된 플랫폼 API 설계 기준선으로 엮는 작업이어야 한다.

⸻

4. 이번 작업의 핵심 질문

이번 문서에서는 최소한 아래 질문들에 답할 수 있어야 한다.
	1.	Phase 3 전체 설계는 어떤 구조와 철학으로 요약될 수 있는가?
	2.	각 하위 문서는 어떤 역할을 가지며 서로 어떻게 연결되는가?
	3.	문서 간 충돌, 중복, 모호함은 무엇이고 어떻게 정리할 것인가?
	4.	구현팀이 실제로 처음부터 구현하려면 어떤 순서와 우선순위가 적절한가?
	5.	MVP 수준에서 반드시 구현할 API 범위는 어디까지인가?
	6.	어떤 항목은 바로 구현하고, 어떤 항목은 후속 Phase 또는 별도 기술 설계로 넘겨야 하는가?
	7.	공통 기반 코드(라우터 구조, auth middleware, response helper, error handler 등)는 어떤 원칙을 따라야 하는가?
	8.	외부 연동, AI 연계, async 확장을 고려하더라도 초기 구현을 어떻게 통제된 범위로 시작할 수 있는가?
	9.	이후 구현 단계에서 설계를 임의로 흔들지 않으려면 어떤 체크리스트가 필요한가?
	10.	다음 Phase 작업 지시서를 만들 때 어떤 형태로 나누는 것이 적절한가?

⸻

5. 상세 작업 요구사항

5-1. 문서 작성

아래 산출물을 작성하라.

필수 산출물
	•	phase3_platform_api_foundation_master.md
	•	phase3_implementation_handoff.md

두 문서는 서로 연결되어야 하지만, 목적은 구분하라.

phase3_platform_api_foundation_master.md
	•	Phase 3 전체 설계의 통합 기준 문서
	•	각 주제의 핵심 결론을 구조적으로 정리
	•	문서 간 관계와 전체 설계 철학을 드러냄
	•	구현자가 “왜 이렇게 설계되었는가”를 이해할 수 있어야 함

phase3_implementation_handoff.md
	•	구현 Phase로 넘기기 위한 실무형 인계 문서
	•	무엇부터 구현할지
	•	어떤 순서로 구현할지
	•	어떤 파일/레이어/구성요소가 먼저 필요한지
	•	무엇은 구현 범위에서 제외해야 하는지
	•	구현 체크리스트와 주의사항

⸻

5-2. 통합 문서의 목적 정리

phase3_platform_api_foundation_master.md 문서 초반에서 다음을 명확히 설명하라.
	•	이 문서는 Phase 3의 개별 설계 문서를 요약한 문서가 아니다.
	•	플랫폼 API 설계의 최종 기준선(master baseline) 이다.
	•	이후 구현과 후속 설계는 이 문서를 기준으로 일관성을 유지해야 한다.
	•	세부 규칙이 필요할 때는 개별 Task 문서로 내려가되, 전체 방향 판단은 이 문서를 따른다.

즉, 이 문서가 최상위 API 아키텍처 기준 문서라는 점을 분명히 하라.

⸻

5-3. 통합 문서에 반드시 포함할 섹션

phase3_platform_api_foundation_master.md에는 최소한 아래 섹션을 포함하라.
	1.	문서 목적
	2.	Phase 3의 전체 목표와 범위
	3.	플랫폼 API의 핵심 철학
	4.	핵심 소비자와 사용 패턴
	5.	리소스 중심 API 구조 요약
	6.	인증/인가와 보안 문맥 요약
	7.	버전 관리 전략 요약
	8.	공통 요청/응답 및 오류 모델 요약
	9.	목록 조회 규약 요약
	10.	idempotency 및 retry safety 요약
	11.	async / event / webhook 확장 구조 요약
	12.	AI / RAG 친화 인터페이스 요약
	13.	observability / audit / traceability 요약
	14.	대표 API 시나리오를 통한 설계 검증 요약
	15.	Phase 3에서 확정된 사항
	16.	의도적으로 미결정으로 남긴 사항
	17.	구현 시 반드시 지켜야 할 기준
	18.	결론

⸻

5-4. 각 하위 Task 결과 통합 요약

통합 문서에서는 Task 3-1 ~ 3-11을 하나씩 단순 나열하지 말고,
다음 관점으로 핵심 결론만 재구성하라.
	•	아키텍처 원칙
	•	리소스 구조
	•	보안 구조
	•	계약 구조(응답/오류)
	•	목록 탐색 구조
	•	변경 안정성(idempotency/versioning)
	•	확장 구조(async/event/webhook)
	•	AI 연계 구조
	•	운영성(observability/audit)
	•	실제 검증 시나리오

중요:
개별 Task 문서를 압축 요약하되,
서로 겹치는 내용을 정리하고 표현을 통일하라.

⸻

5-5. 문서 간 충돌 / 중복 / 모호성 정리

반드시 포함할 것:
	•	개별 Task 문서 사이에서 표현이나 결론이 미묘하게 달라질 수 있는 부분을 점검하고 정리
	•	예를 들어:
	•	jobs vs operations 용어
	•	admin namespace 처리 방향
	•	envelope 강도
	•	search vs searches resource 표현
	•	idempotency mandatory 범위
	•	total count 제공 범위
	•	citation identifier 구체성 수준
	•	충돌이 있으면 어느 방향을 master 기준으로 채택할지 명시
	•	아직 결정하지 않은 것은 미결정 항목으로 분리

즉, 이 작업은 “문서들을 묶는다”가 아니라
정합성을 정리해서 하나의 기준으로 고정하는 작업이다.

⸻

5-6. Phase 3에서 확정된 사항 정리

통합 문서에 별도 섹션으로 아래를 정리하라.

예시 범주:
	•	확정된 아키텍처 원칙
	•	확정된 URI/리소스 구조 방향
	•	확정된 응답/오류 구조 방향
	•	확정된 auth/authz 계층 원칙
	•	확정된 pagination/filter/sort 방향
	•	확정된 idempotency 원칙
	•	확정된 async/event/webhook 확장 방향
	•	확정된 AI/RAG 인터페이스 원칙
	•	확정된 observability/audit 연결 원칙

중요:
이 섹션은 이후 구현자가 “여기서부터는 바꾸지 말아야 할 기준”으로 읽을 수 있어야 한다.

⸻

5-7. 의도적으로 미결정으로 남긴 사항 정리

통합 문서에 별도 섹션으로,
Phase 3에서 일부러 확정하지 않은 항목을 정리하라.

예:
	•	구체 인증 프로토콜 채택
	•	JWT/세션 세부 스키마
	•	구체 OpenAPI schema
	•	exact endpoint catalog
	•	operation payload schema 세부
	•	webhook signature algorithm
	•	error code full catalog
	•	citation identifier 정확한 포맷
	•	검색 알고리즘 / 벡터 DB / 임베딩 모델
	•	concurrency control 세부
	•	rate limit 수치
	•	구현체별 middleware 구조

중요:
미결정은 설계 부족이 아니라
의도적 범위 통제라는 점이 드러나야 한다.

⸻

5-8. 구현 인계 문서의 목적 정리

phase3_implementation_handoff.md 문서 초반에서 다음을 명확히 설명하라.
	•	이 문서는 설계 이론서가 아니라 구현 시작 문서다.
	•	구현자 또는 Claude Code가 API 기초 계층을 만들 때 어떤 순서로 무엇을 해야 하는지 안내한다.
	•	“모든 기능을 한 번에 구현”하지 않고, 기반 계층부터 단계적으로 올리는 방향을 제시한다.
	•	API-first 원칙을 지키면서도 MVP 범위를 통제할 수 있도록 돕는다.

⸻

5-9. 구현 인계 문서에 반드시 포함할 섹션

phase3_implementation_handoff.md에는 최소한 아래 섹션을 포함하라.
	1.	문서 목적
	2.	구현 시작 전 전제 조건
	3.	구현 범위와 비범위
	4.	권장 구현 순서
	5.	Priority 1: API 기초 골격
	6.	Priority 2: 공통 계약 계층
	7.	Priority 3: 보안/권한 연결
	8.	Priority 4: 핵심 문서 리소스 API
	9.	Priority 5: 안정성/운영성 보강
	10.	Priority 6: async/event/webhook 기반 추가
	11.	Priority 7: AI/RAG 확장 포인트 준비
	12.	구현 시 공통 체크리스트
	13.	구현 시 금지사항 / 주의사항
	14.	다음 Phase로 넘길 항목
	15.	결론

⸻

5-10. 권장 구현 순서 제안

구현 인계 문서에서는 아래와 같은 수준으로 구현 순서를 제안하라.

1단계 — API 골격과 공통 계약
	•	버전 prefix 구조
	•	라우터/모듈 구조
	•	공통 응답 envelope/helper 방향
	•	공통 오류 처리 골격
	•	request_id/trace_id 전달 골격

2단계 — 인증/인가 연결
	•	auth context 추출
	•	request security context
	•	authorization layer 연계 지점 확보
	•	tenant scope enforcement 연결 포인트

3단계 — 핵심 문서 리소스
	•	documents
	•	versions
	•	nodes
	•	기본 CRUD/read/list
	•	filter/sort/pagination 적용

4단계 — 안정성/운영성
	•	idempotency 적용이 필요한 write endpoint
	•	audit 연결
	•	observability/logging
	•	structured error codes

5단계 — 확장 구조
	•	operations/jobs 기본 모델
	•	event publication hook point
	•	webhook subscription/delivery 모델 골격

6단계 — AI-ready 인터페이스
	•	구조 조회 강화
	•	버전/노드 기반 read
	•	retrieval/citation 확장 포인트
	•	indexing status/operation 연결 포인트

중요:
이 순서는 하나의 권장 흐름이며,
실제 구현 단위를 Claude Code 작업으로 쪼갤 수 있을 정도로 실무적이어야 한다.

⸻

5-11. 구현 범위와 비범위 명확화

구현 인계 문서에는 반드시 다음을 구분하라.

초기 구현 범위 예시
	•	API version prefix
	•	documents/versions/nodes 핵심 read/write
	•	auth context plumbing
	•	공통 response/error skeleton
	•	list query conventions
	•	request tracing 기본값
	•	일부 idempotent write
	•	감사/운영 연결 포인트

초기 구현 비범위 예시
	•	전체 webhook delivery engine 완성
	•	full AI retrieval engine 완성
	•	벡터 인덱스/임베딩 파이프라인 완성
	•	full error catalog 완성
	•	모든 admin API 완성
	•	모든 bulk/async endpoint 완성

중요:
구현자가 처음부터 과도하게 넓은 범위를 잡지 않게 해야 한다.

⸻

5-12. 구현 시 공통 체크리스트 작성

구현 인계 문서에 아래와 같은 공통 체크리스트를 넣어라.

예시:
	•	path naming 규칙을 따르는가?
	•	response envelope 규약을 지키는가?
	•	error code/message 구조를 따르는가?
	•	authz를 API if문으로 흩뿌리지 않았는가?
	•	request_id/trace_id가 전달되는가?
	•	list endpoints가 공통 query 규약을 따르는가?
	•	idempotency 필요한 write에 고려가 들어갔는가?
	•	actor/tenant context가 감사와 연결되는가?
	•	async/AI 확장 포인트를 막는 구조가 아닌가?

이 체크리스트는 다음 Phase 작업 지시서에서도 재사용 가능해야 한다.

⸻

5-13. 구현 시 금지사항 / 주의사항 정리

반드시 포함할 것:
	•	엔드포인트마다 응답 구조를 제각각 만들지 말 것
	•	authz 로직을 라우터마다 중복 하드코딩하지 말 것
	•	AI 전용 우회 API를 따로 만들지 말 것
	•	내부 구현 편의를 위해 versioning 원칙을 깨지 말 것
	•	list query 문법을 리소스마다 다르게 만들지 말 것
	•	operation/webhook/error 구조를 ad-hoc하게 만들지 말 것
	•	MVP라는 이유로 observability와 error shape를 생략하지 말 것

즉, 구현 과정에서 설계를 훼손하기 쉬운 지점을 경고하라.

⸻

5-14. 다음 Phase로 넘길 항목 정리

구현 인계 문서 말미에서 다음을 정리하라.
	•	Phase 4 이후 실제 구현 Task로 분해할 수 있는 작업군
	•	별도 기술 설계가 필요한 주제
	•	운영 인프라/보안/검색/AI 파이프라인과 연결되는 후속 항목
	•	API 기초 계층 이후 바로 이어질 추천 개발 순서

예:
	•	공통 API skeleton 구현
	•	문서 핵심 CRUD API 구현
	•	authz enforcement layer 구현
	•	audit integration 구현
	•	operation/event/webhook 모델 구현
	•	retrieval/citation/read model 확장 구현

⸻

5-15. Claude Code 작업 분해 관점 반영

가능하면 구현 인계 문서에는
나중에 Claude Code 작업 지시서로 쪼개기 쉬운 작업 묶음도 제안하라.

예:
	•	API skeleton setup
	•	common response/error layer
	•	auth context and dependency plumbing
	•	documents router + service contract
	•	versions/nodes router
	•	list query parser/validator
	•	idempotency middleware/hook
	•	operation resource skeleton
	•	webhook subscription skeleton
	•	AI read-model extension skeleton

중요:
지금 구현 지시서를 만드는 것은 아니지만,
다음 대화에서 곧바로 하위 구현 Task로 쪼갤 수 있는 수준의 묶음이면 좋다.

⸻

6. 산출물

아래 두 파일을 작성하라.

필수 산출물
	•	phase3_platform_api_foundation_master.md
	•	phase3_implementation_handoff.md

문서 성격
	•	하나는 통합 설계 기준 문서
	•	하나는 구현 인계 문서
	•	구현 코드 금지
	•	설계와 구현 사이의 연결 문서 성격

⸻

7. 제외 범위

이번 작업에서는 아래는 하지 마라.
	•	실제 코드 작성
	•	OpenAPI 전체 통합 스펙 작성
	•	DB schema 통합 설계
	•	인증 제품 선택
	•	검색엔진/벡터DB 제품 선택
	•	구현 일정 산정
	•	인력 계획 작성
	•	인프라 배포 문서 작성
	•	UI 설계 문서 작성

즉, 이번 작업은 Phase 3 설계 통합과 구현 인계에 집중하라.

⸻

8. 작성 스타일 가이드

문서는 아래 스타일을 따르라.
	•	통합 문서는 상위 기준 문서답게 구조적이고 정합성 있게 작성
	•	인계 문서는 실무형이고 실행 가능하게 작성
	•	중복 설명보다 정리와 판정이 중요
	•	확정 / 미결정 / 구현 권장 / 구현 비범위 / 주의사항을 분명히 구분
	•	이후 Claude Code 작업 분해에 바로 활용 가능해야 함
	•	지나치게 추상적이지 않게, 구현자가 바로 움직일 수 있는 수준으로 작성

⸻

9. 검토 체크리스트

작업 완료 전에 아래를 스스로 점검하라.
	•	Phase 3 전체 설계가 통합 기준선으로 정리되었는가?
	•	각 하위 문서의 핵심 결론이 정합적으로 묶였는가?
	•	충돌/중복/모호성이 정리되었는가?
	•	확정 사항과 미결정 사항이 분리되었는가?
	•	구현 인계 문서가 실제로 구현 시작에 도움이 되는가?
	•	구현 순서와 범위 통제가 실무적으로 제시되었는가?
	•	공통 체크리스트와 금지사항이 포함되었는가?
	•	다음 Phase 하위 구현 Task로 분해하기 쉬운 형태인가?

⸻

10. Claude Code에 대한 최종 지시

위 요구사항을 반영하여 다음 두 문서를 작성하라.
	•	phase3_platform_api_foundation_master.md
	•	phase3_implementation_handoff.md

문서들은 각각 다음 역할을 충실히 수행해야 한다.
	•	phase3_platform_api_foundation_master.md
→ Phase 3 전체의 통합 API 설계 기준선
	•	phase3_implementation_handoff.md
→ 다음 구현 Phase에서 바로 활용 가능한 실무형 인계 문서

문서 작성 후에는 아래 형식의 자체 점검 요약도 함께 정리하라.

자체 점검 요약
	•	통합 문서에서 확정한 핵심 기준 요약
	•	통합 과정에서 정리한 주요 충돌/모호성 요약
	•	미결정으로 남긴 핵심 항목 요약
	•	구현 인계 문서의 권장 구현 순서 요약
	•	초기 구현 범위 / 비범위 요약
	•	Claude Code 작업 분해 가능 영역 요약
	•	다음 Phase 연결 포인트 요약