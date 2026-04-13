Task 3-9. AI / RAG 연계 가능한 API 인터페이스 설계

Claude Code 작업 지시서

1. 작업명

Phase 3 - Task 3-9. AI / RAG 연계 가능한 API 인터페이스 설계

⸻

2. 작업 목적

범용 문서 플랫폼이 향후 AI 도구, RAG 파이프라인, agentic workflow, 검색 기반 어시스턴트의 공식 기반이 될 수 있도록, AI / RAG 연계 가능한 API 인터페이스 설계 기준을 문서로 정리하라.

이번 작업의 목적은 임베딩 모델을 고르거나 벡터 DB를 정하는 것이 아니다.
핵심은 다음이다.
	•	AI가 플랫폼의 비공식 우회 소비자가 아니라 정식 API 소비자가 되도록 인터페이스 방향 정립
	•	문서 원문뿐 아니라 문서 구조, 버전, 노드, 메타데이터, 상태, 참조 관계를 AI가 일관되게 접근할 수 있는 API 관점 정리
	•	향후 retrieval, citation, indexing, ingestion, sync status, AI processing workflow를 수용할 수 있는 확장 포인트 정의
	•	검색/RAG를 위한 특수 인터페이스가 기존 플랫폼 API 계약과 충돌하지 않도록 연결 규칙 정립
	•	이후 구현 단계에서 AI tool adapter가 아니라 플랫폼 자체가 AI-ready API가 되도록 기준선 마련

즉, 이 문서는 문서 플랫폼의 AI/RAG 친화 API 설계 기준 문서여야 한다.

⸻

3. 반드시 반영할 배경

이 플랫폼은 다음 성격을 가진다.
	•	범용 문서 플랫폼
	•	문서/버전/노드 중심 도메인 구조
	•	메타데이터 확장 구조 존재
	•	권한/ACL/감사 추적 구조 존재
	•	REST 중심 API 구조
	•	외부 시스템과 UI뿐 아니라 AI/Agent도 주요 소비자
	•	향후 검색, retrieval, citation, indexing, ingestion, AI workflow 확장 예정
	•	이벤트/웹훅/비동기 작업과도 연결될 수 있음

이전 Task에서 이미 다음이 정해졌다고 가정하라.
	•	플랫폼 API는 계약 계층이다.
	•	리소스 중심 REST 구조와 명명 규칙이 있다.
	•	인증/인가와 request security context가 있다.
	•	버전 전략이 있다.
	•	공통 응답 포맷이 있다.
	•	목록 조회 규약이 있다.
	•	idempotency 및 retry safety 기준이 있다.
	•	비동기 작업 / 이벤트 / 웹훅 확장 모델이 있다.

따라서 이번 작업은 “AI 전용 별도 땜질 API”를 만드는 것이 아니라,
플랫폼 API를 AI/RAG 활용 가능한 형태로 확장하는 설계 작업이어야 한다.

⸻

4. 이번 작업의 핵심 질문

이번 문서에서는 최소한 아래 질문들에 답할 수 있어야 한다.
	1.	AI / RAG 관점에서 이 플랫폼 API가 제공해야 할 핵심 읽기 인터페이스는 무엇인가?
	2.	AI는 문서 본문만 필요한가, 아니면 구조/버전/노드/메타데이터/상태/권한 맥락도 필요한가?
	3.	일반 REST 리소스 조회와 retrieval/search 전용 인터페이스는 어떻게 구분하고 연결해야 하는가?
	4.	citation 가능한 식별자와 참조 가능한 문서 단위는 어떤 관점에서 설계해야 하는가?
	5.	chunk, node, section, version 단위 접근은 어떻게 다뤄야 하는가?
	6.	indexing/ingestion/embedding 상태는 API에서 어느 정도 공식적으로 드러나야 하는가?
	7.	AI/Agent가 안전하게 사용할 수 있도록 어떤 응답 shape와 메타데이터가 필요한가?
	8.	권한이 있는 범위 내에서만 AI가 문서를 검색/조회하도록 어떤 원칙이 필요한가?
	9.	검색 결과와 원문 조회, 구조 조회, 인용 참조가 하나의 계약 안에서 어떻게 연결되어야 하는가?
	10.	향후 AI processing workflow, summarization, extraction, agent action이 추가되어도 API 구조가 무너지지 않으려면 어떤 기준이 필요한가?

⸻

5. 상세 작업 요구사항

5-1. 문서 작성

아래 산출물을 작성하라.

필수 산출물
	•	Task9_ai_rag_api_interface.md

문서에 반드시 포함할 섹션
	1.	문서 목적
	2.	왜 AI / RAG 친화 인터페이스가 필요한가
	3.	상위 설계 원칙
	4.	AI 소비자 유형과 사용 패턴
	5.	AI/RAG 관점의 핵심 API 요구사항
	6.	문서 구조 조회 인터페이스 방향
	7.	버전/노드/메타데이터 접근 방향
	8.	검색 / retrieval 인터페이스 방향
	9.	citation / reference identifier 설계 관점
	10.	indexing / ingestion / AI processing 상태 노출 방향
	11.	권한/보안/감사 관점
	12.	응답 shape와 tool-friendly 설계 관점
	13.	비동기 작업 / 이벤트 / 웹훅과의 연결
	14.	대표 사용 시나리오
	15.	후속 Task 및 구현 Phase에 전달할 기준
	16.	결론

⸻

5-2. 왜 AI / RAG 친화 인터페이스가 필요한가

문서 초반에서 다음을 명확히 설명하라.
	•	이 플랫폼은 사람이 UI로 읽는 것만이 아니라, AI가 검색·요약·인용·분석·연계 작업을 수행할 기반이 되어야 한다.
	•	AI는 일반 사용자보다 더 구조화된 데이터 접근을 필요로 한다.
	•	문서 원문 전체만 던지는 방식은 비용, 정확성, 권한 통제, 인용 정확성 측면에서 한계가 있다.
	•	RAG는 단순 full-text search가 아니라, 구조 단위/버전 단위/출처 식별이 가능한 조회 모델이 필요하다.
	•	나중에 AI 기능을 억지로 붙이면 비공식 내부 API나 우회 경로가 늘어나고, 보안과 일관성이 무너진다.

즉, AI/RAG 친화 인터페이스는 부가 기능이 아니라
플랫폼을 지식 기반 시스템으로 확장하기 위한 공식 API 기반이라는 점이 드러나야 한다.

⸻

5-3. 상위 설계 원칙 정의

최소한 아래 원칙을 포함하라.
	•	AI is a first-class API consumer
	•	Retrieval builds on platform contracts
	•	Structured access over raw dumping
	•	Citation-ready resource identity
	•	Version-aware knowledge access
	•	Permission-preserving retrieval
	•	Minimal AI-specific duplication
	•	Tool-friendly predictable responses
	•	Async-ready ingestion and indexing
	•	Extensible toward agent workflows

각 원칙마다 다음을 설명하라.
	•	의미
	•	왜 필요한지
	•	실제 API 설계에서 어떤 결정을 유도하는지

예:
	•	Structured access over raw dumping → AI에게 문서 전체 문자열만 넘기기보다 노드/섹션/메타데이터 단위 접근을 우선 고려
	•	Citation-ready resource identity → 검색 결과가 어느 문서/버전/노드에서 왔는지 명확히 참조 가능해야 함
	•	Permission-preserving retrieval → 검색과 retrieval도 일반 리소스 접근과 동일한 보안 모델을 따라야 함

⸻

5-4. AI 소비자 유형과 사용 패턴 정리

다음 소비자를 구분해서 설명하라.
	•	검색/RAG 파이프라인
	•	대화형 AI assistant
	•	문서 요약/분석 도구
	•	외부 AI application
	•	agentic workflow / tool-using agent
	•	내부 indexing / ingestion service
	•	evaluation / offline analysis consumer 가능성

각 유형에 대해 정리할 내용:
	•	필요한 주요 API 패턴
	•	읽기 중심인지, 처리 시작형인지
	•	어떤 수준의 구조화 데이터가 필요한지
	•	권한/감사 추적이 왜 중요한지
	•	동기 REST와 비동기 operation 중 무엇을 더 자주 사용할지

⸻

5-5. AI/RAG 관점의 핵심 API 요구사항 정의

문서에서 최소한 아래 요구를 정리하라.
	•	문서 목록 조회
	•	문서 단건 조회
	•	특정 버전 조회
	•	구조화된 노드/섹션 조회
	•	메타데이터 조회
	•	검색 / retrieval 결과 조회
	•	citation용 참조 정보 획득
	•	indexing / ingestion 상태 조회
	•	대량 AI 처리 작업 시작 및 추적 가능성
	•	변경 이벤트 구독 또는 polling 가능성

중요:
이 요구사항은 “AI 전용 API를 많이 만든다”가 아니라,
기존 플랫폼 리소스와 AI용 조회 패턴을 어떻게 연결할지 관점에서 정리하라.

⸻

5-6. 문서 구조 조회 인터페이스 방향

이 부분은 핵심이다.
다음을 정리하라.
	•	AI가 문서 전체 raw body만 받는 것의 한계
	•	문서 구조(tree / node / section / block)에 접근 가능한 API 필요성
	•	문서와 버전, 버전과 노드의 관계를 AI가 탐색할 수 있어야 함
	•	상위 문서 metadata와 하위 노드 content/context를 함께 활용할 수 있어야 함
	•	전체 문서 조회, 구조 요약 조회, 특정 노드 조회를 구분할 필요
	•	chunk가 내부 구현 개념이어도, AI 관점에서 유사한 주소 단위가 필요할 수 있음

반드시 포함할 것:
	•	구조 조회는 일반 UI용이기도 하지만 AI/RAG에 특히 중요하다는 점
	•	노드 식별자와 버전 식별자의 안정성 관점
	•	parent/child/sibling/context window를 고려한 조회 가능성 검토

⸻

5-7. 버전 / 노드 / 메타데이터 접근 방향

다음을 문서화하라.

버전 관점
	•	AI는 최신 버전만이 아니라 특정 시점 버전이 필요할 수 있음
	•	버전별 인용 가능성
	•	버전 비교 / diff는 후속 확장 포인트가 될 수 있음

노드 관점
	•	section/subsection/paragraph/block 수준 접근이 retrieval 정확도에 유리할 수 있음
	•	노드 단위 식별자와 문맥 연결 정보 필요성

메타데이터 관점
	•	document_type
	•	상태
	•	작성자/관리자/조직 맥락
	•	태그/분류
	•	시간 정보
	•	사용자 정의 metadata

중요:
메타데이터는 필터링과 ranking, retrieval post-processing, citation 설명에 중요하다는 점을 반영하라.

⸻

5-8. 검색 / retrieval 인터페이스 방향

다음을 다뤄라.
	•	일반 collection listing과 retrieval query의 차이
	•	q 기반 단순 검색과 별도 retrieval endpoint/resource의 차이
	•	검색 결과는 단순 문서 목록이 아니라 relevance와 source context를 가질 수 있음
	•	retrieval 결과가 문서/버전/노드/스니펫/점수/근거를 포함할 가능성
	•	full-text / semantic / hybrid는 내부 구현 차이지만, API는 결과 계약을 안정적으로 보여줘야 함
	•	검색은 플랫폼 공통 규약을 최대한 재사용하되, relevance context를 추가로 표현할 수 있어야 함

중요:
이번 문서에서는 검색 알고리즘이 아니라
플랫폼 API에서 retrieval을 어떤 성격의 인터페이스로 볼지를 정리하라.

⸻

5-9. citation / reference identifier 설계 관점

이 부분은 매우 중요하다.
문서에는 반드시 다음을 포함하라.
	•	AI 응답에서 출처를 명시하려면 stable reference가 필요함
	•	citation 단위는 문서 전체일 수도 있고 버전/노드/섹션일 수도 있음
	•	“문서 제목만 반환”은 불충분할 수 있음
	•	어떤 리소스에서 온 정보인지 기계적으로 따라갈 수 있어야 함
	•	citation identifier는 사람이 읽는 제목과 별개로, 안정된 resource reference를 가져야 함
	•	버전 고정 citation과 최신 문서 citation은 의미가 다를 수 있음

검토할 관점:
	•	document_id
	•	version_id
	•	node_id
	•	canonical URI
	•	human-friendly label
	•	optional anchor/path info

중요:
구체 포맷을 확정하지 않아도 되지만,
citation 가능한 플랫폼 식별 체계가 필요하다는 원칙을 명확히 하라.

⸻

5-10. indexing / ingestion / AI processing 상태 노출 방향

다음을 정리하라.
	•	플랫폼이 AI-ready하려면 ingestion/indexing 상태가 완전히 블랙박스여서는 안 됨
	•	어떤 문서/버전이 검색 인덱스에 반영되었는지, 처리 중인지, 실패했는지 운영상 중요
	•	ingestion/indexing 자체는 내부 구현이더라도, 상태 조회와 operation tracking은 외부에 노출 가치가 있음
	•	AI 관련 처리 상태는 jobs/operations 모델과 연결될 수 있음
	•	전용 리소스 또는 서브리소스로 상태를 드러낼지 검토 가능

검토 대상 예시:
	•	indexing status
	•	ingestion status
	•	embedding sync status
	•	last indexed version
	•	processing errors summary

중요:
AI/RAG 기능을 위해 필요한 운영 visibility를 API 설계에 반영하라.

⸻

5-11. 권한 / 보안 / 감사 관점

AI/RAG에서도 반드시 다음 원칙을 반영하라.
	•	AI는 권한 우회자가 아님
	•	retrieval/search 결과도 사용자 또는 서비스 계정 권한 범위 내에서만 반환되어야 함
	•	조직/테넌트 경계를 넘는 검색은 기본 금지
	•	citation 정보도 권한 범위를 넘는 누설이 되지 않도록 주의
	•	AI processing 요청도 actor context와 감사 추적이 필요
	•	외부 AI application은 일반 API 보안 모델과 동일한 기준을 따라야 함

중요:
“AI라서 예외”가 아니라
기존 플랫폼 보안 모델을 그대로 계승하는 AI 인터페이스라는 점을 문서화하라.

⸻

5-12. 응답 shape와 tool-friendly 설계 관점

Task 3-5와 연결하여 다음을 정리하라.
	•	AI tool은 예측 가능한 키 구조를 선호함
	•	resource data, metadata, citation info, retrieval score, snippet/context는 명확히 분리되어야 함
	•	지나치게 인간 중심 서술형 응답을 피해야 함
	•	follow-up fetch가 가능하도록 식별자가 일관되게 포함되어야 함
	•	search result와 source resource 간 연결 정보가 분명해야 함
	•	tool invocation chaining에 적합한 최소 필드 집합 개념을 검토할 수 있음

중요:
응답을 AI가 직접 학습용으로 쓰는 것이 아니라,
도구 호출·검색·후속 조회·인용 연결에 적합해야 한다는 관점으로 작성하라.

⸻

5-13. 비동기 작업 / 이벤트 / 웹훅과의 연결

Task 3-8과 연결하여 다음을 정리하라.
	•	ingestion/indexing/export/summarization 같은 작업은 async operation으로 모델링 가능
	•	문서 변경 시 AI 인덱스 갱신 이벤트가 필요할 수 있음
	•	외부 AI 시스템과의 연동에는 webhook 또는 polling이 필요할 수 있음
	•	대규모 AI 처리 파이프라인은 operation status + event + webhook 조합으로 운영될 수 있음
	•	AI workflow도 플랫폼 공통 확장 모델을 재사용하는 것이 바람직한지 검토

즉, AI/RAG 인터페이스가 별도 섬이 아니라
기존 operation/event/webhook 확장 모델과 연결되어야 한다는 점을 반영하라.

⸻

5-14. 대표 사용 시나리오 포함

문서에 최소한 아래 시나리오를 포함하라.

시나리오 예시
	1.	대화형 assistant가 사용자 권한 범위 내 문서를 검색하고, 관련 노드와 citation 정보를 조회
	2.	외부 RAG 서비스가 특정 조직 범위 문서의 구조와 최신 버전을 가져와 인덱싱
	3.	문서 수정 발생 → indexing operation 시작 → 완료 후 retrieval 가능 상태 반영
	4.	AI 분석 도구가 특정 버전의 특정 섹션만 조회하여 요약 생성
	5.	agent가 검색 결과에서 나온 citation reference를 이용해 원문 노드를 후속 조회

예시는 구현 명세가 아니라 API 소비 흐름 예시 수준으로 작성하라.

⸻

5-15. 후속 Task 및 구현 Phase에 전달할 기준

문서 마지막에 다음 연결 포인트를 정리하라.

예:
	•	Task 3-10에서는 retrieval/search/operation 관련 error model과 observability metadata를 더 구체화해야 함
	•	Task 3-11에서는 AI/RAG 사용 시나리오를 대표 API 시나리오로 검증해야 함
	•	구현 Phase에서는 document/version/node read API, retrieval endpoint, citation identifier, indexing status, operation tracking 설계로 이어져야 함
	•	이후 검색 엔진/벡터 인덱스/랭킹 알고리즘은 별도 기술 설계에서 다뤄야 함
	•	AI workflow actions가 추가되더라도 공통 API 계약을 우선 재사용해야 함

⸻

6. 산출물

아래 파일을 작성하라.

필수 산출물
	•	Task9_ai_rag_api_interface.md

문서 성격
	•	설계 기준 문서
	•	구현 코드 금지
	•	검색엔진/벡터DB/모델 선택 금지
	•	알고리즘 상세 설계 금지
	•	API 계약과 인터페이스 원칙 중심

⸻

7. 제외 범위

이번 작업에서는 아래는 하지 마라.
	•	임베딩 모델 선택
	•	벡터 DB 선택
	•	retrieval ranking algorithm 설계
	•	chunking algorithm 상세 설계
	•	prompt 설계
	•	AI summarization workflow 구현
	•	OpenAPI 전체 작성
	•	실제 search engine DSL 설계
	•	citation formatting UI 설계
	•	indexing pipeline 인프라 상세 설계

즉, 이번 작업은 AI / RAG 친화 API 인터페이스의 상위 기준 수립에 집중하라.

⸻

8. 작성 스타일 가이드

문서는 아래 스타일을 따르라.
	•	AI를 특수 부가기능이 아니라 공식 API 소비자로 다룰 것
	•	raw text dumping보다 구조화 접근의 장점을 강조
	•	문서/버전/노드/metadata/citation/retrieval 관계를 분명히 설명
	•	권장 / 허용 / 비권장 / 금지 / 후속 Task에서 구체화를 구분
	•	검색 알고리즘이 아니라 인터페이스 계약에 집중
	•	보안/권한/감사 관점을 반드시 포함
	•	operation/event/webhook 확장과 연결 지을 것

⸻

9. 검토 체크리스트

작업 완료 전에 아래를 스스로 점검하라.
	•	왜 AI/RAG 친화 인터페이스가 필요한지 충분히 설명했는가?
	•	AI 소비자 유형과 사용 패턴이 정리되었는가?
	•	문서 구조/버전/노드/metadata 접근 필요성이 설명되었는가?
	•	검색/retrieval 인터페이스 방향이 정리되었는가?
	•	citation/reference identifier 관점이 충분히 다뤄졌는가?
	•	indexing/ingestion 상태 노출 방향이 있는가?
	•	권한/보안/감사 관점이 포함되었는가?
	•	tool-friendly 응답 shape 관점이 반영되었는가?
	•	async/event/webhook 확장과 연결되었는가?
	•	후속 Task와 구현 단계가 이 문서를 기준으로 이어질 수 있는가?

⸻

10. Claude Code에 대한 최종 지시

위 요구사항을 반영하여 Task9_ai_rag_api_interface.md 문서를 작성하라.

문서는 범용 문서 플랫폼 API가 향후 검색, retrieval, citation, indexing, AI assistant, agent workflow의 기반이 될 수 있도록 하는
AI / RAG 연계 가능한 API 인터페이스의 상위 설계 기준 문서여야 한다.

문서 작성 후에는 아래 형식의 자체 점검 요약도 함께 정리하라.

자체 점검 요약
	•	AI 소비자 유형 요약
	•	핵심 AI/RAG API 요구사항 요약
	•	문서 구조/버전/노드 접근 관점 요약
	•	검색/retrieval/citation 관점 요약
	•	indexing/ingestion 상태 노출 관점 요약
	•	권한/보안/감사 관점 요약
	•	operation/event/webhook 연결 관점 요약
	•	후속 Task 연결 포인트
	•	의도적으로 후속 Task로 남긴 미결정 사항
