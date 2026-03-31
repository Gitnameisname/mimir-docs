Task I-1. API skeleton 및 버전 라우팅 구조 구현

Claude Code 작업 계획서

1. 작업 목표

플랫폼 API의 실제 구현을 시작할 수 있도록, 버전 기반 라우팅 골격과 도메인별 라우터 분리 구조를 코드베이스에 구축한다.

이 단계에서는 비즈니스 로직 완성보다 다음을 우선한다.
	•	/api/v1 기준의 최상위 API 진입점 확립
	•	domain router 분리 구조 확립
	•	documents / versions / nodes / system 계열의 placeholder 라우트 연결
	•	향후 admin / operations / webhooks / retrieval 확장을 수용할 수 있는 패키지 구조 확보
	•	이후 Task에서 공통 response/error/auth/list-query 계층을 자연스럽게 얹을 수 있도록 기반 정리

즉, 이번 작업은 API의 실질적 “뼈대”를 세우는 작업이다.

⸻

2. 작업 범위

포함 범위
	•	/api/v1 prefix 라우팅 구조 구현
	•	API router entrypoint 구성
	•	domain별 router module 분리
	•	최소 placeholder endpoint 연결
	•	documents, versions, nodes, system 라우터 연결
	•	health 또는 base metadata endpoint 검토 후 최소 1개 제공
	•	향후 확장용 router/module boundary 마련

제외 범위
	•	실제 documents CRUD 비즈니스 로직 구현
	•	공통 response envelope 완성
	•	공통 error handler 완성
	•	auth/authz 실제 enforcement
	•	pagination/filter/sort 구현
	•	idempotency 실제 구현
	•	webhook/event/AI retrieval 실제 구현

⸻

3. 구현 의도

이 작업의 핵심은 “지금 필요한 라우트 몇 개”를 급히 만드는 것이 아니라,
앞으로 플랫폼 API가 커져도 구조가 무너지지 않도록 초기 구조를 올바르게 잡는 것이다.

따라서 아래 원칙을 반드시 반영한다.
	•	버전은 URL prefix 레벨에서 명확히 분리할 것
	•	router는 domain 단위로 분리할 것
	•	각 router는 얇게 유지할 것
	•	placeholder endpoint는 구조 검증용이며, 나중에 교체 가능해야 할 것
	•	admin / operations / webhooks / retrieval은 아직 비어 있어도 패키지 경계는 고려할 것

⸻

4. Claude Code에게 요청할 작업

아래 작업을 순서대로 수행하라.

Step 1. 현재 코드베이스의 API 진입 구조 파악

먼저 현재 프로젝트에서 FastAPI app, router 등록 방식, 디렉터리 구조를 점검하라.

확인할 항목:
	•	FastAPI app 생성 위치
	•	기존 router include 방식
	•	API 관련 package 구조
	•	dependency injection 관련 공통 위치
	•	settings/config 관련 위치
	•	middleware / exception handler 등록 위치

이 단계에서는 현재 구조를 존중하되,
향후 /api/v1 중심 구조로 정리 가능한 최소 변경 방향을 선택하라.

⸻

Step 2. API 버전 패키지 구조 설계 및 반영

다음 요구를 만족하는 구조를 제안하고 실제 코드에 반영하라.

권장 예시는 아래와 같지만, 프로젝트 현실에 맞게 약간 조정 가능하다.

app/
  api/
    __init__.py
    router.py                # 최상위 api router 집계
    v1/
      __init__.py
      router.py              # /api/v1 집계
      documents.py
      versions.py
      nodes.py
      system.py
      admin.py               # placeholder 가능
      operations.py          # placeholder 가능
      webhooks.py            # placeholder 가능
      retrieval.py           # placeholder 가능

중요 원칙:
	•	/api 와 /api/v1 경계를 코드 레벨에서 명확히 둘 것
	•	v1/router.py가 version-specific domain router 집계점이 되게 할 것
	•	개별 domain 파일은 router 객체만 우선 정의하고 endpoint는 최소화할 것

⸻

Step 3. 최상위 API router 조립

다음과 같은 계층을 구현하라.
	•	application main app
	•	include /api
	•	include /v1
	•	include domain routers

예시 개념:
	•	app.api.router → /api
	•	app.api.v1.router → /v1
	•	각 domain router → /documents, /versions, /nodes, /system

중요:
	•	prefix 중복 또는 혼란이 없도록 할 것
	•	tags는 domain 기준으로 부여할 것
	•	추후 OpenAPI 문서에서 domain별로 식별 가능하도록 할 것

⸻

Step 4. 최소 placeholder endpoint 구현

각 라우터에는 구조 검증용 최소 endpoint를 넣어라.

권장 예시:
	•	system.py
	•	GET /api/v1/system/health
	•	GET /api/v1/system/info 또는 GET /api/v1/system/meta
	•	documents.py
	•	GET /api/v1/documents
	•	POST /api/v1/documents
	•	단, 현재는 stub response 가능
	•	versions.py
	•	GET /api/v1/documents/{document_id}/versions
	•	nodes.py
	•	GET /api/v1/versions/{version_id}/nodes

주의:
	•	이 단계에서는 실제 비즈니스 구현이 아니라 route shape 검증이 목적이다
	•	endpoint body는 hardcoded/stub이어도 괜찮다
	•	다만 응답 형태는 나중에 공통 response envelope을 씌우기 쉬운 구조로 작성할 것
	•	“temporary”, “stub” 성격이 드러나도록 주석 또는 TODO를 남길 것

⸻

Step 5. base metadata/health endpoint 검토 및 반영

system 라우터에 최소 1개 이상의 운영성 endpoint를 제공하라.

최소 요구:
	•	health check endpoint 1개
	•	API version 또는 service metadata를 보여줄 수 있는 endpoint 검토

예:
	•	service name
	•	api version
	•	environment
	•	build placeholder
	•	timestamp는 굳이 넣지 않아도 됨

주의:
	•	상세 내부 정보가 과도하게 외부에 노출되지 않게 할 것
	•	production-safe 기본값을 유지할 것

⸻

Step 6. 향후 확장용 placeholder module boundary 마련

아직 구현하지 않더라도 아래 영역은 파일/라우터 경계 수준에서 고려하라.
	•	admin
	•	operations
	•	webhooks
	•	retrieval

반드시 전부 실제 include 할 필요는 없지만,
최소한 아래 둘 중 하나는 충족하라.
	1.	placeholder router 파일 생성
또는
	2.	v1/router.py에서 향후 연결 예정임이 구조적으로 드러나도록 TODO/comment 정리

권장 방식은 placeholder router 파일 생성이다.

예:
	•	admin.py → 현재는 빈 router 또는 GET /_placeholder
	•	operations.py → future async operations boundary
	•	webhooks.py → future subscriptions/delivery boundary
	•	retrieval.py → future AI/RAG access boundary

⸻

Step 7. import 관계와 의존 방향 정리

라우터 골격 단계부터 import 꼬임이 생기지 않도록 아래 원칙을 지켜라.
	•	main/app → api.router
	•	api.router → api.v1.router
	•	api.v1.router → domain routers
	•	domain router끼리 서로 직접 import 최소화
	•	domain router가 서비스 구현에 강하게 결합되지 않도록 유지

즉, 현재는 router dependency graph가 단방향으로 유지되어야 한다.

⸻

Step 8. OpenAPI 관점 검토

Swagger/OpenAPI에서 아래가 자연스럽게 보이는지 확인하라.
	•	/api/v1/... 경로가 일관적으로 보이는가
	•	tags가 domain 기준으로 적절한가
	•	placeholder endpoint도 너무 난잡하지 않은가
	•	향후 admin/operations/webhooks/retrieval이 추가되어도 문서 구조가 유지될 것 같은가

필요하면 tags naming을 약간 정리하라.

⸻

5. 구현 시 세부 요구사항

5-1. URL 설계 원칙

초기 skeleton이더라도 path는 이후 실제 구현에서 유지 가능한 형태를 사용하라.

권장:
	•	/api/v1/documents
	•	/api/v1/documents/{document_id}
	•	/api/v1/documents/{document_id}/versions
	•	/api/v1/versions/{version_id}
	•	/api/v1/versions/{version_id}/nodes
	•	/api/v1/system/health

비권장:
	•	/api/v1/getDocuments
	•	/api/v1/document/list
	•	/api/v1/versionNodes
	•	action-style 혼합 naming

⸻

5-2. Router 역할 제한

이 단계의 router는 다음 역할만 가진다.
	•	path 선언
	•	method 선언
	•	path/query/body 수신
	•	stub response 반환
	•	tags/summary/description 최소 정리

하지 말아야 할 것:
	•	서비스 로직 직접 구현
	•	권한 판단 하드코딩
	•	DB 접근 직접 작성
	•	ad-hoc한 예외 처리 삽입

⸻

5-3. Placeholder 응답 방식

아직 공통 response envelope이 없으므로, 지금은 단순 JSON 응답이어도 괜찮다.
다만 다음 Task에서 envelope 적용이 쉽도록 응답 메시지를 단순하고 예측 가능하게 유지하라.

예시 느낌:

{
  "message": "stub",
  "resource": "documents",
  "version": "v1"
}

혹은

{
  "status": "ok"
}

단, endpoint마다 제각각 응답 형태가 지나치게 다르지 않게 할 것.

⸻

5-4. 문서화 주석

각 router 파일 상단이나 endpoint 주석에 아래를 간단히 남겨라.
	•	이 라우터의 역할
	•	현재 stub 상태인지 여부
	•	후속 Task에서 무엇이 들어올지

예:
	•	documents router: Task I-7에서 실제 리소스 API 확장 예정
	•	versions/nodes router: Task I-8에서 관계 검증 및 read model 확장 예정
	•	system router: health/meta 역할

⸻

6. 산출물 요구

Claude Code는 작업 후 아래 산출물을 정리해서 보고하라.

A. 생성/수정 파일 목록

예:
	•	app/api/router.py
	•	app/api/v1/router.py
	•	app/api/v1/documents.py
	•	app/api/v1/versions.py
	•	app/api/v1/nodes.py
	•	app/api/v1/system.py
	•	필요 시 main.py 또는 app factory 파일

B. 라우팅 구조 요약

예:
	•	/api → 최상위 API router
	•	/api/v1 → versioned router
	•	/api/v1/documents
	•	/api/v1/system/health

C. 구조 설계 판단 근거

짧게 정리:
	•	왜 이렇게 패키지를 나눴는지
	•	admin/operations/webhooks/retrieval 확장을 어떻게 고려했는지
	•	추후 Task I-2 ~ I-4를 어떻게 얹기 쉬운지

D. 남겨둔 TODO

예:
	•	공통 response envelope 적용 예정
	•	공통 error handling 연결 예정
	•	auth context dependency 주입 예정
	•	list query validator 연결 예정

⸻

7. 완료 조건

아래를 만족하면 완료로 본다.
	•	/api/v1 기반 라우팅 구조가 실제 코드에 존재한다.
	•	documents / versions / nodes / system 라우터가 분리되어 있다.
	•	최소 placeholder endpoint가 동작한다.
	•	health 또는 metadata endpoint가 있다.
	•	향후 admin / operations / webhooks / retrieval 확장을 수용할 구조가 보인다.
	•	router 계층이 과도한 비즈니스 로직 없이 얇게 유지된다.
	•	다음 Task에서 response/error/context 계층을 붙이기 쉬운 구조다.

⸻

8. 금지 사항

이번 작업에서는 다음을 하지 마라.
	•	documents CRUD를 완성하려고 하지 말 것
	•	DB 모델/리포지토리까지 연결하려고 하지 말 것
	•	auth를 임시 하드코딩하지 말 것
	•	ad-hoc endpoint를 만들지 말 것
	•	공통 응답/오류 체계를 미리 제멋대로 확정하지 말 것
	•	미래 기능을 추정해서 과도하게 구현하지 말 것

⸻

9. Claude Code 최종 지시문

아래 지시문을 그대로 사용해도 된다.

Task I-1을 수행하라.

목표:
- 플랫폼 API의 버전 기반 라우팅 skeleton을 실제 코드베이스에 구현한다.
- /api/v1 구조를 기준으로 domain router를 분리한다.
- documents / versions / nodes / system 라우터를 placeholder 수준으로 연결한다.
- 향후 admin / operations / webhooks / retrieval 확장을 막지 않는 패키지 구조를 만든다.

반드시 지킬 원칙:
- 지금은 기능 완성이 아니라 API 골격 구현 단계다.
- router는 얇게 유지하고 비즈니스 로직을 넣지 마라.
- ad-hoc endpoint를 만들지 마라.
- 이후 Task I-2, I-3, I-4가 자연스럽게 붙을 수 있게 구조를 잡아라.

수행 항목:
1. 현재 FastAPI app/router 구조를 점검한다.
2. /api 및 /api/v1 라우팅 집계 구조를 만든다.
3. documents.py / versions.py / nodes.py / system.py 라우터를 분리한다.
4. 최소 placeholder endpoint를 각 라우터에 만든다.
5. system 라우터에 health 또는 metadata endpoint를 제공한다.
6. admin / operations / webhooks / retrieval 확장을 고려한 placeholder module boundary를 만든다.
7. import dependency가 단방향으로 유지되게 정리한다.
8. OpenAPI 상에서 path/tag 구조가 일관적인지 검토한다.

산출물:
- 생성/수정 파일 목록
- 최종 라우팅 구조 요약
- 구조 설계 판단 근거
- 남겨둔 TODO 목록