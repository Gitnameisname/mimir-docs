# Task4_api_versioning_strategy.md
# API 버전 관리 전략

---

## 1. 문서 목적

이 문서는 Mimir 플랫폼 API가 장기적으로 진화하면서도 계약 안정성을 유지하기 위한 **API 버전 관리 전략 기준**을 확정한다.

단순히 URL에 `/v1`을 붙일지 말지를 정하는 것이 아니다. 이 문서는 다음을 정의한다:
- API 변경 시 버전을 올리는 기준
- 소비자 호환성을 유지하는 원칙
- 실험/베타 기능의 관리 방식
- 폐기 예정 기능의 운영 절차
- 이벤트, 웹훅, AI 연계 영역을 포함한 일관된 버전 철학

이 문서는 Task 3-5 이후 모든 API 설계 결정과 구현 Phase 라우터 구조에 적용된다.

---

## 2. API 버전 관리가 필요한 이유

### 2-1. 플랫폼 API는 장기 계약이다

Mimir API는 다음 소비자가 동시에 사용한다:
- User UI / Admin UI
- 외부 애플리케이션
- AI / RAG / Agent 도구
- 내부 서비스

이 소비자들은 각기 다른 릴리즈 주기를 가진다. 외부 통합 시스템은 플랫폼 업데이트와 무관하게 수개월~수년 이상 동일 API 계약을 사용할 수 있다. AI 도구는 구조적으로 고정된 인터페이스를 기대한다.

**버전 관리 없이는** 내부 구현 개선이 기존 소비자를 언제든지 파괴할 수 있다.

### 2-2. 리소스와 필드는 시간이 지나며 반드시 확장된다

문서, 버전, 노드, 권한, 감사 로그, 검색, 이벤트, 웹훅, AI 연계 — 이 모두가 시간이 지나며 필드와 동작이 변한다. 버전 정책이 없으면 각 변경이 구현자 재량으로 흩어지고, 계약 일관성이 붕괴된다.

### 2-3. 버전 관리는 "있으면 좋은 것"이 아니다

버전 정책이 없을 때 발생하는 실제 문제:
- 소비자가 언제 어떤 변화가 생겼는지 알 수 없음
- 호환성이 깨졌을 때 원인 추적이 어려움
- 변경 고지 없이 외부 시스템이 장애를 겪음
- AI 소비자가 파싱 오류를 경험함

**버전 관리는 계약 안정성을 위한 필수 운영 원칙이다.**

---

## 3. 버전 관리의 상위 원칙

### 원칙 1. Explicit Versioning Policy

**의미**: 버전 정책은 암묵적 관행이 아니라 명시된 규칙이다.

**왜**: 암묵적 정책은 팀마다 다르게 해석된다.

**설계 판단**: 모든 API 변경에 앞서 "이 변경이 breaking인가?"를 체크리스트로 확인한다. 정책 문서(이 문서)가 판단 기준이 된다.

---

### 원칙 2. Backward Compatibility First

**의미**: 새 버전을 올리기 전에, 기존 계약을 유지한 채 확장할 방법을 먼저 탐색한다.

**왜**: 버전을 올리면 소비자에게 마이그레이션 부담이 생긴다.

**설계 판단**: breaking change를 피할 수 있다면 피한다. 피할 수 없을 때만 새 major version을 올린다.

---

### 원칙 3. Minimize Breaking Changes

**의미**: breaking change는 최소화한다. 특히 필드 제거나 의미 변경은 극히 제한적으로만 허용한다.

**왜**: 소비자의 대응 비용은 예상보다 크다.

**설계 판단**: 필드를 제거하고 싶다면 먼저 deprecated로 표시하고 충분한 이행 기간을 둔다.

---

### 원칙 4. Additive Evolution Preferred

**의미**: 기존 필드를 제거하거나 의미를 바꾸기보다, 새 optional 필드를 추가하는 방향을 우선한다.

**왜**: 필드 추가는 기존 소비자가 무시하면 그만이다. 필드 제거는 기존 소비자를 즉시 파괴한다.

**설계 판단**: 응답에 새 정보를 담아야 한다면 기존 구조에 optional 필드 추가를 먼저 검토한다.

---

### 원칙 5. Stable Contracts over Rapid Churn

**의미**: 내부 구현이 자주 바뀌어도 외부 계약은 안정적으로 유지한다.

**왜**: API 계약은 소비자와의 약속이다.

**설계 판단**: DB 컬럼명 변경, 내부 서비스 분리 등 내부 변경은 API 계약 변경 사유가 아니다.

---

### 원칙 6. Deprecation Before Removal

**의미**: 기능을 제거하기 전에 deprecated 상태로 충분히 운영한 뒤 제거한다.

**왜**: 소비자에게 마이그레이션 시간을 보장한다.

**설계 판단**: deprecated 선언 후 최소 유지 기간 없이 곧바로 제거하는 것을 금지한다.

---

### 원칙 7. Predictable Rollout

**의미**: 버전 변경 일정과 절차가 소비자에게 예측 가능하게 공지된다.

**왜**: 소비자가 마이그레이션 계획을 세울 수 있어야 한다.

**설계 판단**: major version 출시와 이전 버전 sunset 일정은 사전 공지한다.

---

### 원칙 8. Documentation-Synchronized Change Management

**의미**: API 계약 변경은 항상 OpenAPI 문서/CHANGELOG 갱신과 동기화된다.

**왜**: 문서와 구현이 다르면 소비자가 신뢰할 수 없다.

**설계 판단**: 구현 변경 없이 문서만 변경, 또는 문서 없이 구현만 변경하는 케이스 금지.

---

### 원칙 9. Consistent Version Semantics Across Resource Domains

**의미**: 문서/권한/검색/이벤트/웹훅/AI 등 모든 리소스 도메인은 동일한 버전 철학을 따른다.

**왜**: 도메인마다 버전 정책이 다르면 소비자가 혼란스럽다.

**설계 판단**: 특정 도메인만 "버전 예외 지대"로 두지 않는다.

---

## 4. 버전 표기 방식 후보 비교

### 4-1. URI Versioning

형태: `/api/v1/documents`

| 항목 | 내용 |
|---|---|
| 장점 | 눈에 바로 보임. 브라우저/curl/로그에서 즉시 식별. 라우팅이 단순. 문서화 편의성 높음. |
| 단점 | URL 구조에 버전이 노출됨. 리소스 canonical URL이 버전마다 달라짐. |
| 클라이언트 사용성 | 직관적. 코드에서 버전 교체가 명확. |
| 문서화 | OpenAPI spec 분리 용이. Swagger UI 등에서 버전별 분리 표현 쉬움. |
| 라우팅/운영 | 라우터에서 prefix로 단순 처리 가능. 디버깅 쉬움. |
| 외부 연동 | 외부 시스템이 URL만 보고 버전을 파악 가능. |
| 이 플랫폼 적합성 | **높음** |

### 4-2. Header-based Versioning

형태: `X-API-Version: 1` 또는 `Accept-Version: v1`

| 항목 | 내용 |
|---|---|
| 장점 | URL 구조가 깔끔. 리소스 canonical URL이 버전 무관. |
| 단점 | 헤더를 잊으면 기본 버전이 뭔지 불명확. 브라우저 직접 호출 불편. 캐싱 복잡. |
| 클라이언트 사용성 | 헤더 설정 필요. 실수 가능성 높음. |
| 문서화 | 버전별 문서 분리가 어렵고 혼란스러움. |
| 라우팅/운영 | 라우터가 헤더를 파싱해야 함. 로그에서 식별 어려움. |
| 외부 연동 | 헤더 기반 연동 복잡도 증가. |
| 이 플랫폼 적합성 | **낮음** (외부 연동과 AI 소비자를 고려하면 부적합) |

### 4-3. Media Type Versioning

형태: `Accept: application/vnd.mimir.v1+json`

| 항목 | 내용 |
|---|---|
| 장점 | REST 이론적으로 가장 순수한 방식. 콘텐츠 협상과 통합. |
| 단점 | 구현 복잡. 학습 비용 높음. 대부분의 일반 도구에서 사용 불편. |
| 클라이언트 사용성 | 매우 불편. AI 도구와 외부 연동 모두 부담. |
| 문서화 | 현실적으로 문서화 복잡. |
| 외부 연동 | 비표준적 인식이 강함. 채택률 낮음. |
| 이 플랫폼 적합성 | **부적합** |

### 4-4. 혼합 전략

URI versioning을 기본으로 하되, 특정 목적(예: 콘텐츠 타입 협상)에 한해 제한적 헤더 사용 허용.

→ 세부 헤더 사용 케이스는 구현 Phase에서 필요시 결정.

---

## 5. 권장 버전 전략

### 5-1. 채택: URI Versioning (Major Version Only)

```
/api/v1/documents
/api/v1/versions/{versionId}
/api/v1/searches
```

**근거**:
- 모든 소비자(UI, 외부 시스템, AI)에게 명확하고 직관적
- 라우팅/운영/디버깅이 단순
- OpenAPI 문서화와 자연스럽게 연동
- 초기 플랫폼에 적합한 단순성과 장기 확장성의 균형

### 5-2. Major Version만 URI에 표기

- Minor/patch 수준 변경은 URI에 반영하지 않는다.
- `v1.1`, `v1.2` 같은 sub-version을 URL에 넣는 것은 **금지**.
- 동일 major version 내 변경은 non-breaking 방식으로만 허용한다.

```
✅ /api/v1/documents
❌ /api/v1.1/documents
❌ /api/v1/v1.2/documents
```

### 5-3. 버전 없는 엔드포인트 금지

- `/api/documents` 같은 버전 없는 path는 **금지**.
- 기본값(default version) fallback을 공개 API에 허용하지 않는다.

**이유**: 소비자가 버전을 명시하지 않으면 향후 major version 업 시 자동으로 계약이 깨진다.

### 5-4. 비권장 사용 패턴

| 패턴 | 이유 |
|---|---|
| `/api/latest/...` | "latest"는 시간이 지나며 의미가 변하므로 불안정 |
| 버전 없는 fallback | 기본값이 바뀌면 소비자 파괴 |
| sub-version URL | URI가 불필요하게 복잡해짐 |
| 도메인별 다른 버전 | 소비자 혼란 유발 |

---

## 6. Breaking vs Non-breaking Change 기준

### 6-1. 판단 기준 원칙

> **Breaking change란: 기존 소비자가 코드를 수정하지 않으면 정상 동작하지 않는 모든 변경이다.**

기존 소비자가 새 응답/요청을 그대로 무시할 수 있으면 non-breaking, 무시할 수 없으면 breaking이다.

### 6-2. Non-breaking Change (동일 major version 내 허용)

| 변경 유형 | 예시 | 조건 |
|---|---|---|
| 새 optional 응답 필드 추가 | `{ ..., "summary": "..." }` 추가 | 기존 소비자가 무시 가능해야 함 |
| 새 리소스 collection 추가 | `/api/v1/activities` 신규 추가 | 기존 경로 영향 없음 |
| 새 filter/sort 옵션 추가 | `?sort=updated_at` 옵션 추가 | 기존 파라미터 동작 유지 |
| 새 HTTP 메서드 추가 | 기존 리소스에 `PATCH` 추가 | 기존 메서드 영향 없음 |
| 기존 의미 유지한 metadata 확장 | `links`, `meta` 섹션 추가 | envelope 구조 변경 없음 |
| 에러 응답에 detail 필드 추가 | `{ "error": {..., "details": [...] } }` | 기존 필드 의미 변경 없음 |
| 더 관대한 유효성 검증 | 이전에 필수였던 필드를 optional로 완화 | 기존 소비자에게 유리한 변경 |
| 성능 개선, 내부 구현 변경 | DB 쿼리 최적화 | 외부 계약 영향 없음 |

### 6-3. Breaking Change (새 major version 필요)

| 변경 유형 | 예시 | 이유 |
|---|---|---|
| 필드 제거 | `created_by` 삭제 | 기존 소비자가 해당 필드 참조 시 실패 |
| 필드 이름 변경 | `owner` → `author` | 기존 소비자가 `owner`를 찾을 수 없음 |
| 필드 타입 변경 | `id: int` → `id: string` | 파싱 오류 발생 |
| 필수 요청 필드 추가 | 기존 optional → required | 기존 소비자의 요청이 거부됨 |
| 응답 envelope 구조 변경 | `{ data: ... }` → `{ result: ... }` | 모든 소비자의 파싱 로직 파괴 |
| URI path 변경 | `/documents` → `/docs` | 기존 URL 404 |
| HTTP 메서드 의미 변경 | `GET`으로 상태 변경 유발 | 소비자 예측 불가 |
| 상태 코드 의미 변경 | 성공이 200에서 201로 변경 | 소비자의 분기 로직 파괴 가능 |
| 정렬/기본값 변경 | 목록 기본 정렬 순서 변경 | 소비자가 순서에 의존할 경우 파괴 |
| 인증 요구 수준 강화 | 공개 → 인증 필요로 변경 | 기존 비인증 소비자 접근 불가 |
| 리소스 관계 구조 변경 | subresource 위치 변경 | URL 변경으로 귀결 |
| 이벤트/웹훅 payload 필드 제거 | webhook body에서 `document_id` 제거 | 수신 측 파싱 실패 |

### 6-4. 판단이 어려운 케이스

| 케이스 | 판단 방향 |
|---|---|
| 필드 의미를 미묘하게 확장 | Breaking으로 간주. 명시적 소비자 공지 필요. |
| 기존에 명세에 없던 필드를 소비자가 의존 | 소비자 과실이지만, 실무적으로는 Breaking으로 취급 고려 |
| 오류 응답 구조 변경 | Breaking. 소비자가 에러 파싱을 한다고 가정. |
| 새 enum 값 추가 | Non-breaking이지만, 소비자가 enum exhaustive check를 하면 문제. 공지 필요. |
| 응답 필드 순서 변경 | Non-breaking (JSON은 순서 무관). 그러나 직렬화 방식에 따라 이슈 가능. |

---

## 7. 버전별 호환성 유지 원칙

### 7-1. Major Version 안정성 보장 범위

- 동일 major version(`v1`) 안에서는 **non-breaking change만 허용**한다.
- `v1`을 사용하는 소비자는 minor 변경에 코드 수정 없이 계속 동작해야 한다.

### 7-2. Additive Change 범위

- Optional 필드 추가는 자유롭게 허용.
- 단, **소비자는 알 수 없는 필드를 무시(ignore unknown fields)할 수 있어야 한다는 전제**가 필요. API 문서에 이 요구사항을 명시.

### 7-3. 정렬/기본값 변경의 위험성

- 목록 조회 기본 정렬 순서, 기본 page size 변경은 소비자에게 영향을 줄 수 있다.
- 이러한 변경은 non-breaking처럼 보이지만 실질적으로 소비자 로직에 영향을 줄 수 있으므로 **사전 공지 후 적용**을 원칙으로 한다.

### 7-4. 버전은 코드 릴리즈 번호가 아니다

- API `v1`은 소프트웨어 `1.0.0` 릴리즈와 같지 않다.
- 내부 코드가 `2.x`가 되어도 외부 API는 `v1`을 유지할 수 있다.
- API 버전은 **외부 계약의 안정성 단위**이며 내부 릴리즈 주기에 종속되지 않는다.

### 7-5. 전체 API vs 도메인별 버전 분리

**결정**: 플랫폼 전체를 단일 major version으로 관리한다. 도메인별 버전 분리는 허용하지 않는다.

**이유**:
- 도메인별 버전 분리는 `documents`는 `v2`, `webhooks`는 `v1` 같은 혼란을 유발한다.
- 소비자가 하나의 major version만 추적하면 되는 단순성을 유지한다.
- 단, 특정 도메인을 experimental namespace에 두는 것은 허용 (섹션 9 참조).

---

## 8. Deprecation 및 Sunset 정책

### 8-1. 용어 정의

| 용어 | 의미 |
|---|---|
| **Deprecated** | 더 이상 권장하지 않으며 향후 제거될 예정임을 공지한 상태. 아직 동작함. |
| **Sunset** | 특정 날짜 이후 더 이상 지원하지 않는다고 예고된 상태. |
| **Removed** | 실제로 엔드포인트/필드가 제거된 상태. |

### 8-2. Deprecated 표시 방법

**응답 헤더**:
```
Deprecation: true
Sunset: Sat, 31 Dec 2026 00:00:00 GMT
Link: <https://docs.mimir.io/migration/v1-to-v2>; rel="successor-version"
```

**OpenAPI 문서**:
- 해당 엔드포인트/필드에 `deprecated: true` 표시

**API 응답 본문**:
- 필드 수준 deprecation은 응답의 `meta.warnings` 배열에 포함 (공통 응답 포맷 Task 3-5에서 확정)

**변경 로그(CHANGELOG)**:
- deprecated 선언 일자, 대체 방법, sunset 예정일 기록

### 8-3. 최소 유지 기간 정책 방향

| 소비자 유형 | 권장 최소 유지 기간 |
|---|---|
| 외부 공개 API | 최소 6개월 이상 |
| 내부 UI | 최소 1 릴리즈 사이클 이상 |
| 실험/베타 기능 | 안정성 보장 없음 — 단기 제거 가능 |

*정확한 기간은 구현 Phase 운영 정책에서 확정. 단, 외부 소비자 없이 운영 초기에는 유동적으로 적용.*

### 8-4. Deprecated 상태에서 허용/금지 변경

| 구분 | 허용 여부 |
|---|---|
| 보안 패치 | 허용 |
| 버그 수정 | 허용 |
| 새 기능 추가 | 금지 (deprecated 상태에서 확장하지 않음) |
| 이미 deprecated된 기능의 동작 변경 | 금지 |

### 8-5. Sunset 이후 제거 절차

1. Sunset 날짜 전 최종 공지
2. Sunset 날짜 이후 해당 엔드포인트 → `410 Gone` 응답 반환
3. 일정 기간 후 라우터에서 완전 제거

**중요**: Sunset 이후 즉시 제거하지 않고 `410 Gone`으로 일정 기간 유지하여 소비자가 오류를 인식할 시간을 준다.

### 8-6. 내부 소비자 vs 외부 소비자 대응 차이

| 소비자 | 대응 방식 |
|---|---|
| 외부 공개 소비자 | 공식 공지, 충분한 이행 기간, 응답 헤더 경고 |
| 내부 UI/서비스 | 직접 협의 가능. 이행 기간 단축 가능. 단, 원칙은 동일 적용. |

---

## 9. Beta / Experimental API 운영 원칙

### 9-1. 용어 구분

| 구분 | 설명 | 안정성 보장 |
|---|---|---|
| **Stable** (`/api/v1/`) | 정식 안정 API | breaking change 금지, deprecation 정책 적용 |
| **Beta** (`/api/beta/`) | 안정화 전 기능 | 계약이 변경될 수 있음. 프로덕션 사용 비권장 |
| **Experimental** (`/api/experimental/`) | 탐색 단계 기능 | 언제든지 변경/제거 가능. 고지 없이 제거 가능. |

### 9-2. 안정 버전과 분리 원칙

- 실험/베타 기능을 `/api/v1/` 계약 안에 섞는 것은 **금지**.
- 이유: `/api/v1/`의 stability 보장이 전체적으로 희석된다.
- 실험 기능이 충분히 안정화되면 `/api/v1/`로 **승격(promote)**하는 절차를 밟는다.

### 9-3. Namespace 방식

```
/api/v1/...            → 안정 API
/api/beta/...          → 베타 API (계약 변경 가능)
/api/experimental/...  → 실험 API (언제든 변경/제거)
```

### 9-4. 실험/베타 API의 공통 규약

실험 기능이라도 다음은 반드시 따른다:
- 동일한 보안/인가 원칙
- 동일한 기본 응답 envelope (Task 3-5에서 확정)
- 동일한 오류 응답 구조
- 감사 추적 연결

**"실험적"이라는 이유로 보안/감사/기본 계약을 생략하지 않는다.**

### 9-5. 정식 승격 기준

실험/베타 API가 `/api/v1/`로 승격되려면:
- 충분한 내부 사용 테스트 완료
- 계약 구조 안정화 (큰 변경 없이 일정 기간 유지)
- 문서화 완료
- 보안/감사 연결 검증

---

## 10. 내부 API와 외부 API의 버전 정책 관계

### 10-1. 공통 계약 철학 유지

**방향**: 내부 UI(User/Admin)와 외부 공개 API는 **동일한 공통 플랫폼 API**를 사용한다. 별도 분리하지 않는다.

**이유**:
- 내부 전용 API와 외부 API를 분리하면 동기화 부담이 두 배가 된다.
- 내부 API가 외부보다 훨씬 빠르게 변하면 내부 UI가 테스트베드가 되어버리고, 결국 외부에 안 좋은 API가 나온다.

### 10-2. 노출 범위와 안정성 요구 차이

| 구분 | 안정성 요구 | 관리 수준 |
|---|---|---|
| 외부 공개 API (External) | 매우 높음 | deprecation 정책 엄격 적용 |
| 내부 서비스 간 API | 높음 | 직접 협의로 이행 기간 단축 가능 |
| Admin UI용 | 높음 | 동일 계약. Admin 전용 기능은 인가로 구분 |
| 실험/베타 | 낮음 | `/api/beta/` 또는 `/api/experimental/` 사용 |

### 10-3. 관리자 기능의 버전 생애주기

- 관리자 전용 기능도 동일한 major version 체계를 따른다.
- `/api/v1/admin/...` 형태를 사용하는 경우도 same versioning 적용.
- 관리자 기능이라도 외부 서비스 계정(API Key)이 사용하는 경우 외부 공개 API 수준의 안정성을 보장한다.

---

## 11. 이벤트 / 웹훅 / AI 연계 API의 버전 관점

### 11-1. 이벤트 / 웹훅 Payload도 버전 관리 대상이다

웹훅 payload는 외부 시스템이 수신하는 계약이다. REST API와 동일하게 버전 관리가 필요하다.

**원칙**:
- 웹훅 payload도 non-breaking 변경(optional 필드 추가)은 허용
- 필드 제거/이름 변경은 breaking → 버전 업 또는 migration 공지 필요
- 이벤트 스키마 버전과 REST API major version은 **원칙적으로 일치**시킨다

**Payload에 버전 정보 포함**:
```json
{
  "event_type": "document.published",
  "schema_version": "v1",
  "payload": { ... }
}
```

### 11-2. REST API 버전과 이벤트 스키마 버전의 관계

- 동일 major version(`v1`) 안에서는 이벤트 payload도 additive evolution만 허용.
- delivery endpoint와 payload schema는 **같은 버전을 따른다** (분리 금지).
- 이유: delivery endpoint만 v2로 올리고 payload는 v1으로 유지하면 소비자가 혼란스러움.

### 11-3. AI / RAG 연계 API

- AI 소비자는 일반 소비자보다 더 엄격한 계약 안정성을 필요로 할 수 있다.
  - 이유: AI 파이프라인은 한 번 구성되면 자주 업데이트되지 않는 경우가 많다.
- retrieval API, citation identifier, search API도 동일한 major version 체계를 따른다.
- 실험적 AI 기능(예: 새 검색 알고리즘)은 `/api/experimental/`에서 운영 후 안정화되면 `/api/v1/`로 승격.
- citation identifier는 특히 안정성이 중요하다 — 한 번 외부에 노출된 citation은 쉽게 변경할 수 없다. 구체 포맷은 Task 3-9에서 정의하되, 변경 불가 기준을 명시해야 한다.

---

## 12. 후속 Task 및 구현 Phase에 전달할 기준

| 대상 | 전달 기준 |
|---|---|
| Task 3-5 (공통 응답 포맷) | 응답 envelope 구조는 `v1` 계약이므로, 이후 변경 시 breaking change 검토 필요. `meta.warnings`에 deprecation 경고 포함 방식 설계. |
| Task 3-6 (목록 조회) | pagination/filter/sort 파라미터 추가는 non-breaking으로 설계. 기본값 변경은 사전 공지 필요. |
| Task 3-7 (idempotency) | idempotency 키 구조는 major version 간 의미가 달라지지 않도록 설계. |
| Task 3-8 (async/event/webhook) | 이벤트/웹훅 payload에 `schema_version` 필드 포함. delivery endpoint와 payload 버전 일치 원칙 구체화. |
| Task 3-9 (AI/RAG) | citation identifier의 불변성 기준 정의. AI용 실험 endpoint의 experimental namespace 운영 방침. |
| 구현 Phase | 라우터 prefix `/api/v1/` 구조 적용. OpenAPI YAML에 deprecated 필드 표시. `Deprecation`/`Sunset` 응답 헤더 미들웨어 구현. CHANGELOG 관리 정책 수립. |

---

## 13. 결론

Mimir 플랫폼 API의 버전 관리 전략은 다음으로 요약된다:

1. **URI Versioning (Major only)**: `/api/v1/...` 방식. 단순하고 명확. 모든 소비자에게 직관적.
2. **Backward Compatibility First**: Major version 안에서는 additive evolution만 허용. breaking change는 새 major version.
3. **Deprecation Before Removal**: 제거 전 충분한 공지와 이행 기간.
4. **실험 분리**: `/api/experimental/` → `/api/beta/` → `/api/v1/` 승격 경로.
5. **일관성**: REST, 이벤트, 웹훅, AI — 모든 인터페이스에 동일한 버전 철학.

버전 전략은 개발 편의가 아니라 **소비자와의 약속을 관리하는 운영 정책**이다.

---

## 자체 점검 요약

### 권장 버전 표기 방식

URI Versioning, Major version만 표기. `/api/v1/...` 형태. Sub-version URL 금지. Default fallback 금지.

### Breaking / Non-breaking 기준 요약

- **Non-breaking**: optional 필드 추가, 새 리소스/옵션 추가, 내부 구현 변경, 더 관대한 검증
- **Breaking**: 필드 제거/이름변경/타입변경, 필수 필드 추가, envelope 구조 변경, URI 변경, 상태코드 의미 변경, 인증 강화

### Deprecation / Sunset 정책 요약

Deprecated → 응답 헤더 경고 + OpenAPI 표시 + CHANGELOG 기록 → Sunset 날짜 공지 → `410 Gone` → 완전 제거. 외부 공개 API 최소 6개월 유지.

### Beta / Experimental 운영 요약

`/api/experimental/` → `/api/beta/` → `/api/v1/` 승격 경로. 실험 기능도 보안/감사/기본 계약 적용. `/api/v1/`에 실험 기능 혼용 금지.

### 내부/외부/웹훅/AI API 버전 관점 요약

- 내부/외부: 공통 플랫폼 API. 외부는 더 엄격한 deprecation 적용.
- 이벤트/웹훅: payload에 `schema_version` 포함. REST API와 버전 일치.
- AI/RAG: citation identifier 불변성 중요. 실험 AI는 experimental namespace.

### 후속 Task 연결 포인트

Task 3-5(envelope 구조 안정성), 3-6(파라미터 additive 확장), 3-7(idempotency 버전 중립), 3-8(이벤트 schema_version), 3-9(AI citation 불변성 및 experimental 분리), 구현 Phase(라우터 prefix, OpenAPI, deprecation 헤더)

### 의도적으로 미결정으로 남긴 항목

- 정확한 deprecation 최소 유지 기간 수치 (운영 정책에서 확정)
- `410 Gone` 유지 기간 수치 (구현 Phase에서 결정)
- Beta/Experimental namespace의 정확한 path 최종 확정 (구현 Phase에서 결정)
- 이벤트 `schema_version` 필드의 정확한 포맷 (Task 3-8에서 결정)
- AI citation identifier 불변성 기준 구체화 (Task 3-9에서 결정)
- OpenAPI 버전별 spec 분리 운영 방식 (구현 Phase에서 결정)
