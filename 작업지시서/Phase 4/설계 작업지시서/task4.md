

# Task 4-4 작업지시서

## 작업명

문서 작성/수정 API 설계

---

## 1. 작업 목적

Phase 4의 네 번째 작업으로, 문서 생성, 수정, Draft 저장, Published 전환, 버전 조회, 버전 복원, 렌더링 조회 흐름을 플랫폼 API 관점에서 일관되게 설계한다.

이번 작업의 목적은 다음과 같다.

- Task 4-1에서 정의한 문서 생명주기를 API 계약으로 구체화한다.
- Task 4-2의 Version 모델과 Task 4-3의 저장 구조를 실제 요청/응답 흐름에 연결한다.
- User UI, Admin UI, 외부 시스템, 향후 AI/RAG 연동이 공통으로 사용할 수 있는 문서 API 기준선을 정한다.
- Draft / Published / Restore 흐름에서 API 책임 경계를 명확히 하여 구현 혼선을 줄인다.
- 공통 응답 포맷, 권한 체크 포인트, 에러 계약, idempotency 관점을 반영한 API 설계 문서를 만든다.

이 문서는 이후 구현 단계의 라우트 설계, 서비스 계층 설계, 권한 적용, 테스트 케이스 작성의 기준 문서가 되어야 한다.

---

## 2. 작업 배경

앞선 작업에서 다음이 이미 정리되었다고 가정한다.

- Task 4-1: 문서 생성/개정/발행/복원의 생명주기와 상태 전이
- Task 4-2: Version 모델 상세 설계
- Task 4-3: 문서 저장 구조와 content_snapshot 규약
- Phase 2: 권한, 역할, 감사 추적 원칙
- Phase 3: API 버전 관리, 공통 응답 포맷, 인증/인가, pagination/filtering/sorting 규약

이제 필요한 것은, 이 설계들을 실제 플랫폼 API 수준으로 내리는 일이다.

즉 이번 작업은 단순히 엔드포인트 목록을 나열하는 것이 아니라, 다음을 정의하는 작업이다.

- 어떤 행위를 어떤 리소스 API로 노출할 것인가?
- 생성/수정/발행/복원 흐름을 REST적으로 어떻게 표현할 것인가?
- 어떤 요청은 동기 처리하고, 어떤 요청은 명시적 액션 API로 둘 것인가?
- 현재 문서 조회와 특정 버전 조회를 어떻게 구분할 것인가?
- UI 친화성과 플랫폼 일관성 사이에서 어떤 API 구조가 적절한가?

---

## 3. 작업 범위

### 포함 범위

- 문서 생성 API 설계
- 문서 메타데이터 수정 API 설계
- Draft 저장 API 설계
- Published 전환 API 설계
- 버전 목록 조회 API 설계
- 버전 상세 조회 API 설계
- 버전 복원 API 설계
- 렌더링용 현재 문서 조회 API 설계
- 렌더링용 특정 버전 조회 API 설계
- 요청/응답 스키마 수준 정의
- 공통 에러 응답 및 예외 시나리오 정리
- 권한 및 감사 연계 포인트 정리
- idempotency 필요 API 검토

### 제외 범위

- 실제 FastAPI 라우트 구현
- Pydantic 모델 코드 작성
- DB 쿼리 구현
- SSE / 웹소켓 실시간 동기화 설계
- 파일 첨부 업로드 API 상세 설계
- 검색 API 설계
- diff 비교 API 상세 설계

이번 작업은 **문서 작성/수정/버전 관리 API 계약 설계**에 집중한다.

---

## 4. 핵심 설계 질문

이번 작업에서는 최소한 아래 질문들에 답할 수 있어야 한다.

1. 문서 생성 시 Draft까지 함께 생성하는가, 아니면 빈 Document만 생성하는가?
2. 문서 수정은 Document 리소스를 수정하는가, Draft Version 리소스를 수정하는가?
3. Draft 저장은 `PUT`/`PATCH`로 표현하는가, 별도 action endpoint로 표현하는가?
4. Published 전환은 상태 변경으로 표현하는가, `publish` 액션으로 표현하는가?
5. 버전 복원은 과거 Version 자체를 되살리는가, `restore` 액션을 통해 새 Draft를 만드는가?
6. 현재 문서 조회와 특정 버전 조회는 어떤 URL 구조로 구분하는가?
7. 메타데이터 수정과 본문 수정 API를 분리할 것인가?
8. 부분 수정 API를 제공하더라도 내부 저장은 전체 snapshot 갱신으로 할 수 있는가?
9. 어떤 API에 idempotency key가 필요한가?
10. 권한 검사는 리소스 접근과 상태 전이 액션에서 각각 어떻게 반영되어야 하는가?

---

## 5. 작업 지시

아래 항목을 순서대로 설계 문서로 정리한다.

### 5-1. API 리소스 모델 정의

먼저 문서 도메인을 API 리소스 관점에서 어떻게 노출할지 정리한다.

최소 검토 대상:

- Document
- Draft Version
- Published Version
- Version Collection
- Rendered Document View

다음 질문에 답한다.

- 외부에 1차적으로 노출되는 주 리소스는 무엇인가?
- Version은 하위 리소스인가, 독립 리소스인가?
- Draft와 Published를 별도 리소스로 노출할지, Document 아래 상태 기반 서브리소스로 다룰지
- UI 친화성과 API 일관성 중 어디에 무게를 둘지

권장 방향은, `Document`를 최상위 리소스로 두고 `Version`을 하위 또는 연관 리소스로 두는 방식이다. 다만 실제 채택안은 비교 검토 후 명시한다.

---

### 5-2. 엔드포인트 후보 목록 작성

초기 Phase에서 필요한 최소 엔드포인트 목록을 정의한다.

최소 검토 대상은 아래를 포함한다.

- `POST /api/v1/documents`
- `GET /api/v1/documents/{document_id}`
- `PATCH /api/v1/documents/{document_id}`
- `GET /api/v1/documents/{document_id}/draft`
- `PUT or PATCH /api/v1/documents/{document_id}/draft`
- `POST /api/v1/documents/{document_id}/publish`
- `GET /api/v1/documents/{document_id}/versions`
- `GET /api/v1/documents/{document_id}/versions/{version_id}`
- `POST /api/v1/documents/{document_id}/versions/{version_id}/restore`
- `GET /api/v1/documents/{document_id}/render`
- `GET /api/v1/documents/{document_id}/versions/{version_id}/render`

필요 시 다음도 검토한다.

- `POST /api/v1/documents/{document_id}/drafts` 형태의 새 Draft 생성 API
- `DELETE /api/v1/documents/{document_id}/draft` 또는 draft discard API
- `GET /api/v1/documents/{document_id}/current-version`

각 엔드포인트에 대해 아래를 정리한다.

- 목적
- HTTP method
- path
- 주요 요청 필드
- 주요 응답 필드
- 권한 체크 포인트
- 감사 이벤트 필요 여부

---

### 5-3. 문서 생성 API 설계

문서 생성 API의 계약을 정의한다.

반드시 다음을 검토한다.

- 생성 요청 시 필수 입력값
- 문서 생성과 초기 Draft 생성의 결합 여부
- 빈 본문 허용 여부
- 문서 타입 지정 방식
- 초기 metadata 입력 범위
- 응답에 포함할 값(Document, current_draft, links 등)
- idempotency 필요 여부

예시 관점:

- 문서 생성 시점에 Document와 초기 Draft Version을 함께 생성하면 UI 흐름이 단순해질 수 있다.
- 하지만 아주 빈 Document 생성이 필요할 수도 있으므로 두 방식의 장단점을 비교한다.

이번 Phase 기준 권장안을 문서에 명시한다.

---

### 5-4. 메타데이터 수정 API와 본문 수정 API 관계 정의

다음 두 방향을 비교 검토한다.

#### 안 A. 문서 메타데이터와 Draft 본문 수정 API 분리

- Document metadata는 `PATCH /documents/{id}`
- Draft content는 `PATCH or PUT /documents/{id}/draft`

#### 안 B. 단일 저장 API 중심

- Draft 저장 시 metadata와 content를 함께 업데이트

각 방식의 장단점을 작성하고, 다음 관점을 포함한다.

- 명확한 책임 분리
- UI 사용 편의성
- 버전 증가 기준과의 정합성
- 감사 추적 단순성
- 구현 복잡도

이번 Phase 기준 권장안을 제시한다.

---

### 5-5. Draft 저장 API 설계

Draft 저장 API의 계약을 상세히 정의한다.

반드시 다음을 다룬다.

- 전체 content tree 저장인지, 부분 patch 허용인지
- `PUT`과 `PATCH` 중 어떤 의미가 적절한지
- content와 metadata 동시 수정 허용 여부
- change_summary 입력 여부
- optimistic locking 또는 revision check 필요 여부 검토
- 응답에 반환할 Draft Version 정보
- 저장 시 실제 Version 번호 증가 정책과 API 표현 방식 연결

중요: 사용자 편집은 부분 수정처럼 보여도, 내부적으로는 전체 snapshot 갱신일 수 있다는 점을 API 설계 문서에 명시한다.

---

### 5-6. Published 전환 API 설계

Publish 행위는 일반 수정과 다르므로 별도 액션 API 여부를 검토한다.

반드시 다음을 정리한다.

- `POST /documents/{id}/publish` 형태의 action endpoint 적절성
- 요청 바디에 포함될 값(예: publish_note, expected_draft_version_id)
- 현재 Draft가 없을 때 처리 방식
- 현재 Published와 새 Published의 관계
- publish 권한 체크 포인트
- publish 성공 시 응답 구조
- 감사 이벤트 기록 항목

권장 방향은, publish는 일반 `PATCH`가 아니라 **명시적 상태 전이 액션 API**로 두는 것이다. 실제 채택안과 근거를 문서에 적는다.

---

### 5-7. 버전 조회 API 설계

버전 히스토리 탐색을 위한 API를 설계한다.

반드시 아래를 포함한다.

- 버전 목록 조회 API
- 정렬 기준
- 상태 필터링 가능 여부
- pagination 적용 방식
- 버전 상세 조회 API
- 현재 Draft / 현재 Published를 응답에 어떻게 표시할지
- 사용자가 version_number 기준으로 조회할 필요가 있는지 검토

목록 응답은 UI와 운영자 도구 모두 사용할 수 있도록 충분한 요약 정보를 포함해야 한다.

예시 필드:

- version_id
- version_number
- status
- created_at
- created_by
- published_at
- restored_from_version_id
- change_summary

---

### 5-8. 버전 복원 API 설계

복원은 상태 변경이 아니라 새 작업 상태를 만드는 액션으로 설계하는 방향을 우선 검토한다.

반드시 다음을 정리한다.

- `POST /documents/{id}/versions/{version_id}/restore` 구조 적절성
- 요청 시 필요한 값(예: restore_note)
- 응답으로 새 Draft를 돌려줄지 여부
- 기존 Draft 존재 시 정책
- 복원 대상 제약
- 권한 체크 포인트
- 감사 로그 항목

중요: 복원 API는 과거 버전을 수정하는 API가 아니라, **과거 Version을 기반으로 새 Draft를 만드는 명시적 액션**이라는 점을 문서에 분명히 적는다.

---

### 5-9. 렌더링 조회 API 설계

렌더링 목적의 조회 API를 별도 관점으로 설계한다.

다음 두 방향을 비교 검토한다.

#### 안 A. 문서 조회 응답에 렌더링 가능한 구조를 그대로 포함

- API 수 감소
- 응답이 무거워질 수 있음

#### 안 B. 렌더링 전용 ViewModel API 별도 제공

- 역할 분리 가능
- 변환 비용/엔드포인트 수 증가

최소한 아래를 정리한다.

- 현재 문서 렌더링 조회 API
- 특정 버전 렌더링 조회 API
- 일반 도메인 조회와 렌더링 조회의 차이
- 저장 구조와 렌더링 구조 분리 정책과의 연결

Task 4-7과 충돌하지 않도록, 이번 문서에서는 API 관점의 책임만 정의한다.

---

### 5-10. 요청/응답 스키마 초안 정의

각 핵심 API에 대해 요청/응답 스키마의 핵심 필드를 정리한다.

최소 대상 API:

- 문서 생성
- 문서 메타데이터 수정
- Draft 저장
- Publish
- 버전 목록 조회
- 버전 상세 조회
- Restore
- 렌더링 조회

각 API에 대해 다음을 작성한다.

- 요청 필드 목록
- 필수/선택 여부
- 응답 최상위 구조
- 핵심 data 필드
- meta 포함 여부
- links 또는 related resource 정보 필요 여부

공통 응답 포맷은 Phase 3의 원칙을 따른다.

---

### 5-11. 에러 계약 및 예외 시나리오 정리

각 API에서 발생 가능한 주요 예외 상황을 정리한다.

최소 검토 대상:

- 문서 없음
- Draft 없음
- Publish 가능한 상태 아님
- Restore 불가능한 버전
- 권한 없음
- validation 실패
- optimistic lock 충돌 또는 stale update
- 이미 폐기된 문서
- 지원하지 않는 document_type

각 예외에 대해 다음을 정리한다.

- 상황 설명
- 권장 HTTP status
- 에러 코드
- 사용자 메시지/개발자 메시지 분리 필요 여부
- 재시도 가능 여부

---

### 5-12. 권한, 감사, idempotency 포인트 정리

각 API에 대해 다음 세 가지 관점을 함께 정리한다.

#### 권한

- 누가 조회 가능한가
- 누가 Draft 수정 가능한가
- 누가 Publish 가능한가
- 누가 Restore 가능한가

#### 감사

- 어떤 API 호출이 audit event를 남겨야 하는가
- 어떤 필드를 이벤트에 포함해야 하는가

#### Idempotency

- 문서 생성
- Publish
- Restore

위 액션들에 idempotency key가 필요한지 검토하고, 필요한 경우 어떤 요청에 적용할지 정리한다.

---

## 6. 산출물 요구사항

이번 작업의 최종 산출물은 하나의 상세 API 설계 문서다.

문서에는 반드시 아래 항목이 포함되어야 한다.

1. 작업 목적
2. API 리소스 모델 정의
3. 엔드포인트 후보 목록
4. 문서 생성 API 설계
5. 메타데이터 수정 / 본문 수정 API 관계
6. Draft 저장 API 설계
7. Publish API 설계
8. 버전 조회 API 설계
9. Restore API 설계
10. 렌더링 조회 API 설계
11. 요청/응답 스키마 초안
12. 에러 계약 및 예외 시나리오
13. 권한 / 감사 / idempotency 포인트
14. 권장 API 구조안 및 선택 근거
15. 후속 작업 영향도

가능하면 아래 시각화 초안도 포함한다.

- Mermaid API 흐름 시퀀스 다이어그램
- 리소스 관계 다이어그램 초안

---

## 7. 산출물 작성 원칙

- 구현 코드가 아니라 설계 문서 작성에 집중한다.
- REST 원칙을 참고하되, 상태 전이 액션은 필요 시 명시적 action endpoint를 허용한다.
- Task 4-1, 4-2, 4-3의 용어와 충돌하지 않도록 작성한다.
- 사용자 UI 편의만 보지 말고, 외부 시스템과 운영 도구 재사용성도 고려한다.
- 요청/응답 예시는 실제 구현 참고용 수준으로 충분히 구체적으로 작성한다.
- 저장 모델, 도메인 모델, 렌더링 뷰 모델을 혼동하지 않도록 구분해서 서술한다.

---

## 8. 권장 방향

아래 방향을 우선안으로 검토한다.

### 우선안 A

- 최상위 리소스는 `Document`다.
- `Version`은 `Document`에 종속된 하위 조회/액션 리소스로 노출한다.
- 문서 생성 시 초기 Draft를 함께 생성하는 방향을 우선 검토한다.
- 문서 메타데이터 수정과 Draft content 저장 API는 분리하되, 필요 시 한 요청에서 일부 공통 필드를 허용할 수 있게 설계한다.
- Publish와 Restore는 일반 수정이 아니라 **명시적 action endpoint**로 설계한다.
- 현재 문서 조회와 특정 버전 조회를 분리한다.
- 렌더링 조회는 일반 도메인 조회와 별도 ViewModel API로 분리 가능한 구조를 우선 고려한다.
- 문서 생성, Publish, Restore에는 idempotency 적용 가능성을 검토한다.

이 우선안의 장점은 다음과 같다.

- 문서 생명주기와 API 구조가 자연스럽게 대응된다.
- 권한과 감사 포인트가 명확해진다.
- UI와 외부 시스템이 공통 계약을 재사용하기 쉽다.
- Publish/Restore 같은 중요 액션을 일반 수정과 분리하여 안전하게 다룰 수 있다.

---

## 9. 완료 기준

다음 조건을 만족하면 이번 작업이 완료된 것으로 본다.

- 문서 작성/수정/발행/복원 흐름에 대한 API 구조가 문서화되어 있다.
- 주요 엔드포인트와 HTTP method가 정리되어 있다.
- 최소 핵심 API들에 대한 요청/응답 스키마 초안이 포함되어 있다.
- Publish와 Restore의 action endpoint 여부가 선택 근거와 함께 명시되어 있다.
- 에러 계약과 주요 예외 시나리오가 정리되어 있다.
- 권한, 감사, idempotency 포인트가 API별로 검토되어 있다.
- 구현 단계에서 라우트/서비스/Pydantic 모델 설계를 진행할 수 있을 정도로 충분히 구체적이다.

---

## 10. 금지사항

이번 작업에서는 아래를 하지 않는다.

- 실제 FastAPI 라우터 코드 작성
- Pydantic request/response 모델 구현
- DB CRUD 구현
- SSE 스트리밍 설계 확장
- diff API 상세 설계
- 파일 업로드 API 설계 확장

범위를 넘기지 말고, 작성/수정/버전 관리 API 설계에 집중한다.

---

## 11. 후속 연계 작업

이 작업이 끝나면 다음 작업들이 이어진다.

- Task 4-5: Draft / Published 상태 관리 규칙 설계
- Task 4-6: 버전 조회, 이력 탐색, 복원 흐름 설계
- Task 4-7: 렌더링 파이프라인 기초 설계
- 구현 단계: FastAPI route 설계, request/response schema 설계, service layer 구현

즉, 이번 문서는 Phase 4에서 **문서 행위 API의 기준 문서** 역할을 한다.