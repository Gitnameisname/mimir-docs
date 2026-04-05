# Task3_authn_authz_api_design.md
# 인증/인가 구조의 API 계층 반영 설계

---

## 1. 문서 목적

이 문서는 Mimir 플랫폼 API 계층에서 인증(Authentication)과 인가(Authorization)가 어떻게 반영되어야 하는지를 정의하는 API 보안 문맥 설계 기준 문서다.

이 문서의 목적은 로그인 플로우나 토큰 포맷을 확정하는 것이 아니다. 핵심은 다음이다:
- API 요청이 어떤 보안 문맥을 가져야 하는지 정의
- 인증된 주체와 조직/테넌트 스코프를 API 계층에서 어떻게 다룰지 정리
- 권한 판단이 어느 계층에서 수행되어야 하는지 경계 설정
- User/Admin/External/Service/AI 소비자별 인증/인가 요구 분리
- ACL enforcement, 감사 추적, 외부 연동, 비동기 작업, 웹훅, AI까지 수용 가능한 기준선 마련

이 문서는 Phase 2의 권한 모델과 Phase 3의 REST 구조를 API 레벨에서 연결하는 역할을 한다.

---

## 2. 인증과 인가의 역할 구분

### 2-1. 개념 정의

| 개념 | 정의 | 질문 |
|---|---|---|
| **Authentication (인증)** | 요청 주체가 누구인지 확인 | "당신은 누구입니까?" |
| **Authorization (인가)** | 이 주체가 특정 리소스/행위에 무엇을 할 수 있는지 판단 | "당신은 이것을 할 수 있습니까?" |

### 2-2. 핵심 구분 원칙

- **인증 성공이 곧 접근 허용을 의미하지 않는다.** 인증된 주체도 요청한 리소스에 대한 권한이 없으면 거부된다.
- **인가는 복합적이다.** role 기반, ACL 기반, 조직 스코프, 리소스 상태, 정책 조건 등을 함께 고려한다.
- **API 계층의 역할은 인증 결과를 요청 문맥으로 정리하는 것이다.** 권한 판단 그 자체를 API 코드가 전부 직접 구현하지 않는다.
- **인증/인가 혼용 금지.** 두 계층은 책임이 다르므로 명확히 분리해야 한다.

```
요청 도달
   ↓
[Authentication Layer]
 - 요청 주체 식별 (누구인가)
 - Security Context 구성
   ↓
[API Layer]
 - 입력 유효성 검증
 - Security Context를 서비스/인가 레이어에 전달
   ↓
[Authorization Layer]
 - 주체 + 리소스 + 행위 + 조건 → 허용/거부 판단
   ↓
[Service / Domain Layer]
 - 비즈니스 로직 실행
```

---

## 3. API 보안 설계의 상위 원칙

### 원칙 1. Default Deny

**의미**: 인증 또는 권한 문맥이 불충분하면 허용이 아니라 거부가 기본이다.

**왜 필요한가**: 허용 기본(default allow) 구조는 누락된 권한 검사가 보안 구멍이 된다.

**설계에서 어떤 판단을 유도하는가**:
- 공개 접근이 필요한 엔드포인트는 예외로 명시적 선언
- 권한 체크가 없는 엔드포인트는 기본 차단

---

### 원칙 2. Least Privilege

**의미**: 주체는 작업에 필요한 최소한의 권한만 가진다.

**왜 필요한가**: 과도한 권한은 침해 발생 시 피해 범위를 확대한다.

**설계에서 어떤 판단을 유도하는가**:
- 서비스 계정/API 키는 특정 리소스 범위와 행위로 제한
- background job은 원 요청자의 권한 범위를 초과할 수 없음
- AI/Agent 소비자도 명시적 scope 제한 적용

---

### 원칙 3. Explicit Actor Context

**의미**: 모든 요청은 명확한 actor, tenant, auth method, request_id 같은 문맥을 가져야 한다.

**왜 필요한가**: 익명 요청이나 문맥이 모호한 요청은 감사 추적이 불가능하고 권한 판단이 어렵다.

**설계에서 어떤 판단을 유도하는가**:
- 모든 API 요청에 Request Security Context 구성 (섹션 5 참조)
- 내부 서비스 호출도 service identity를 명시적으로 포함

---

### 원칙 4. Tenant-Aware Access

**의미**: 모든 데이터 접근은 조직/테넌트 경계를 인식한다. Cross-tenant 접근은 기본 금지.

**왜 필요한가**: 다중 조직을 지원하는 플랫폼에서 테넌트 경계 침범은 데이터 격리 실패다.

**설계에서 어떤 판단을 유도하는가**:
- 모든 리소스 조회/변경에 tenant scope 적용
- 관리자도 특별히 허가된 경우에만 cross-tenant 접근 가능

---

### 원칙 5. Separation of Authentication and Authorization

**의미**: 인증과 인가는 서로 다른 계층에서 처리된다.

**왜 필요한가**: 두 개념을 혼용하면 API 코드에 권한 if문이 난립하고 유지보수가 불가능해진다.

**설계에서 어떤 판단을 유도하는가**:
- 인증 결과를 Security Context로 변환 → Authorization Layer에 위임
- API 핸들러에 직접 권한 로직 삽입 금지

---

### 원칙 6. API-Level Enforcement

**의미**: 권한 강제는 API 레벨에서 수행된다. UI 숨김이나 프론트엔드 제어만으로는 충분하지 않다.

**왜 필요한가**: UI는 우회 가능하다. 실제 보호는 API에서만 보장된다.

**설계에서 어떤 판단을 유도하는가**:
- 모든 쓰기/삭제/권한 변경 요청은 API에서 권한 검증
- "관리자만 볼 수 있는 UI"라도 API 자체가 비관리자를 거부해야 함

---

### 원칙 7. No UI-Only Security

**의미**: 보안 결정을 UI에만 의존하지 않는다.

**왜 필요한가**: API-first 플랫폼에서는 UI 없이 API를 직접 호출하는 경우가 많다.

---

### 원칙 8. Auditable Access

**의미**: 누가 무엇에 접근했는지 추적 가능해야 한다.

**왜 필요한가**: 컴플라이언스, 보안 감사, 장애 분석에 필요하다.

**설계에서 어떤 판단을 유도하는가**:
- actor, resource, action, result, timestamp를 연결
- 실패한 접근 시도도 기록 대상

---

### 원칙 9. Service Identity Support

**의미**: 사람 이외의 소비자(서비스, 시스템, AI 등)도 정식 보안 모델 안에 포함된다.

**왜 필요한가**: 내부 서비스나 AI agent가 사람 토큰을 공유하면 추적 불가, 최소 권한 원칙 불가.

---

### 원칙 10. Extensible Credential Model

**의미**: 인증 방식이 변경되어도 API 보안 구조가 유지될 수 있는 추상화를 갖춘다.

**왜 필요한가**: 초기엔 단순 bearer token이어도 나중에 OAuth, SAML 등으로 확장될 수 있다.

**설계에서 어떤 판단을 유도하는가**:
- 인증 수단을 Security Context와 분리 — 내부 로직은 Context만 보도록 설계
- 인증 방식 교체 시 API 핸들러 코드를 건드리지 않아도 되는 구조

---

## 4. API 소비자별 인증 주체 모델

### 4-1. 주체 유형 정의

| 주체 유형 | 설명 | 사람 직접 여부 |
|---|---|---|
| 일반 사용자 | 문서 조회, 편집 등 일반 기능 사용자 | 예 |
| 관리자 사용자 | 조직/사용자/권한 관리 등 관리 기능 수행 | 예 |
| 외부 애플리케이션 | API 키 또는 서비스 자격증명으로 연동 | 아니오 |
| 내부 서비스 | 플랫폼 내 마이크로서비스 | 아니오 |
| Background Worker / Async Job | 비동기 작업 실행 프로세스 | 아니오 |
| Webhook Caller / Receiver | 외부 → 플랫폼 수신 또는 플랫폼 → 외부 발신 | 아니오 |
| AI / RAG / Agent | 플랫폼 API를 소비하는 AI 도구 | 아니오 (또는 사용자 대리) |
| Service Account / Machine Identity | 시스템 자동화 목적 계정 | 아니오 |

### 4-2. 주체별 상세 정의

#### 일반 사용자

| 항목 | 내용 |
|---|---|
| 적합한 인증 수단 후보 | Session/cookie, Bearer token |
| 권한 범위 제한 | 자신이 속한 조직 내, 접근 권한이 있는 리소스만 |
| Actor context 필수 필드 | actor_id, actor_type=user, tenant_id, session/token_id |
| 감사 로그 기록 | user_id, org_id, resource, action, result |

#### 관리자 사용자

| 항목 | 내용 |
|---|---|
| 적합한 인증 수단 후보 | 일반 사용자와 동일 (인증 수단 아닌 역할로 구분) |
| 권한 범위 제한 | 관리 대상 조직 범위 내. Super admin은 플랫폼 전역. |
| Actor context 필수 필드 | actor_id, actor_type=admin, tenant_id, roles |
| 감사 로그 기록 | 일반 사용자 + 관리 행위(역할 변경, 강제 조치 등) 추가 기록 |

#### 외부 애플리케이션

| 항목 | 내용 |
|---|---|
| 적합한 인증 수단 후보 | API Key, Service Token |
| 권한 범위 제한 | 등록 시 정의한 scope로 제한 |
| Actor context 필수 필드 | actor_id=app_id, actor_type=external_app, tenant_id, scope |
| 감사 로그 기록 | app_id, org_id, resource, action, result |

#### 내부 서비스

| 항목 | 내용 |
|---|---|
| 적합한 인증 수단 후보 | Service Token, mTLS (강한 내부 통신) |
| 권한 범위 제한 | 서비스별 사전 정의된 기능 범위 |
| Actor context 필수 필드 | actor_id=service_id, actor_type=internal_service, service_name |
| 감사 로그 기록 | service_id, action, resource, result |

#### Background Worker / Async Job

| 항목 | 내용 |
|---|---|
| 적합한 인증 수단 후보 | 위임 컨텍스트 (원 요청자 권한의 축약 저장), 또는 Service Token |
| 권한 범위 제한 | 원 요청자의 권한 범위 이내. 시스템 승격 금지. |
| Actor context 필수 필드 | original_actor_id, actor_type=async_job, job_id, operation_id |
| 감사 로그 기록 | original_actor + job context 모두 기록 |

#### Webhook Caller / Receiver

| 항목 | 내용 |
|---|---|
| 적합한 인증 수단 후보 | Signed secret (HMAC 서명 검증) |
| 권한 범위 제한 | 발신: 구독 등록 시 정의한 이벤트 유형만. 수신: 서명 검증 통과 여부 |
| Actor context 필수 필드 | webhook_id, actor_type=webhook, event_type, delivery_id |
| 감사 로그 기록 | webhook_id, event_type, delivery_id, result |

#### AI / RAG / Agent

| 항목 | 내용 |
|---|---|
| 적합한 인증 수단 후보 | Service Token (독립 동작), 또는 Bearer Token (사용자 대리) |
| 권한 범위 제한 | 독립 동작 시 명시적 scope 제한. 사용자 대리 시 사용자 권한 범위 이내. |
| Actor context 필수 필드 | actor_id, actor_type=ai_agent or delegated, delegating_user_id (대리 시), scope |
| 감사 로그 기록 | agent_id + delegating_user_id + resource + action 기록 |
| 주의 | delegated action vs autonomous action 구분 필요 (섹션 10 참조) |

---

## 5. Request Security Context 설계

API 요청이 처리되는 과정에서 Authentication 이후 내부 계층으로 전달되는 **Request Security Context**를 정의한다.

### 5-1. Security Context 필드

| 필드 | 타입 | 필수 여부 | 설명 |
|---|---|---|---|
| `actor_id` | string | 필수 | 요청 주체 식별자 (user_id, service_id, app_id 등) |
| `actor_type` | enum | 필수 | user / admin / external_app / internal_service / async_job / ai_agent / webhook |
| `tenant_id` | string | 필수 | 조직/테넌트 식별자 |
| `roles` | string[] | 조건부 | 주체에게 부여된 역할 집합 |
| `auth_method` | enum | 필수 | session / bearer_token / api_key / service_token / signed_secret |
| `credential_id` | string | 권장 | 사용된 자격증명 식별자 (세션 ID, 토큰 ID 등) |
| `request_id` | string | 필수 | 이 요청 고유 식별자 |
| `trace_id` | string | 권장 | 분산 추적 식별자 |
| `is_impersonation` | boolean | 조건부 | 관리자가 다른 사용자를 대리하는 경우 |
| `impersonated_actor_id` | string | 조건부 | impersonation 대상 |
| `is_service_identity` | boolean | 조건부 | 사람이 아닌 주체 여부 |
| `original_actor_id` | string | 조건부 | async job / delegate 시 원 요청자 |
| `delegating_user_id` | string | 조건부 | AI/Agent가 사용자 대리 시 위임자 |
| `scope` | string[] | 조건부 | 외부 앱/AI agent의 허가된 기능 범위 |
| `source_client_id` | string | 권장 | 요청 출처 애플리케이션 |
| `human_initiated` | boolean | 권장 | 사람이 직접 트리거한 요청인지 여부 |
| `issued_at` | timestamp | 권장 | 자격증명 발급 시각 |

### 5-2. 전달 원칙

- Request Security Context는 **API 핸들러에서 구성**된다.
- 구성된 Context는 **Service Layer, Authorization Layer, Audit Layer로 전달**된다.
- 각 계층은 HTTP 요청 객체에 직접 접근하지 않고 Context를 통해서만 보안 정보를 읽는다.
- Context는 **불변(immutable)**이어야 한다 — 서비스 계층이 Context를 수정해서는 안 된다.

```
HTTP Request
   ↓
Authentication Middleware
   ↓ (Security Context 구성)
API Handler
   ├→ Authorization Layer (Context 전달)
   ├→ Service Layer (Context 전달)
   └→ Audit Layer (Context 기록)
```

---

## 6. 조직/테넌트 스코프 반영 원칙

### 6-1. Tenant-Aware 설계가 필요한 이유

Mimir는 여러 조직이 동일 플랫폼을 사용하는 multi-tenant 구조를 지원한다. 테넌트 경계를 API 레벨에서 강제하지 않으면 데이터 격리가 깨진다.

### 6-2. 조직 스코프 처리 방향

**채택**: 조직 스코프는 **path가 아닌 Security Context에서 처리**한다.

이유:
- path에 `orgId`를 포함하면 모든 URL이 깊어지고 소비자 부담 증가
- 인증된 주체는 이미 org에 귀속되어 있으므로, Security Context의 `tenant_id`로 처리가 자연스럽다
- 관리자가 여러 조직을 관리해야 하는 경우, `organization_id` 쿼리 파라미터를 선택적으로 허용

### 6-3. Cross-Tenant 접근 정책

| 케이스 | 정책 |
|---|---|
| 일반 사용자 | Cross-tenant 접근 금지. tenant_id 외 리소스 접근 시 403 반환 |
| 관리자 | 자신이 관리하는 조직 범위 내에서만 허용 |
| Super Admin | 플랫폼 전역 접근 허용. 단, 모든 cross-tenant 행위는 별도 감사 기록 |
| 내부 서비스 | 명시적 cross-tenant 필요 시 서비스 identity로 예외 허용. 제한적으로 적용 |

### 6-4. 복수 조직 소속 사용자 처리

- 사용자가 여러 조직에 속할 수 있는 경우, 요청 시 active org context를 명시해야 한다.
- 방식: 인증 토큰에 active_org claim 포함, 또는 요청 헤더 `X-Organization-Id` 제공.
- 세부 처리 방식은 실제 인증 시스템 구현 시 확정.

---

## 7. API Layer와 Authorization Layer의 책임 분리

### 7-1. API Layer의 책임

| 역할 | 설명 |
|---|---|
| 인증 정보 파싱 | 요청 헤더/쿠키에서 자격증명 추출 |
| Security Context 구성 | 인증 미들웨어 결과를 Context로 변환 |
| 입력 유효성 검증 | 요청 형식, 필수 필드 등 검증 |
| Authorization Layer 호출 | 권한 판단 위임 |
| 응답 생성 | 허용/거부 결과에 따라 응답 구성 |
| 메타데이터 전달 | 감사/추적 레이어로 Context 전달 |

**API Layer가 하지 않아야 할 것**:
- 직접 권한 if문 작성 (예: `if user.role == "admin": ...`)
- DB에서 직접 권한 조회
- 비즈니스 규칙 기반 접근 판단

### 7-2. Authorization Layer의 책임

| 역할 | 설명 |
|---|---|
| Role 기반 판단 | 주체의 역할이 해당 행위를 허용하는지 |
| ACL 기반 판단 | 리소스별 접근 제어 항목 확인 |
| Resource Ownership 확인 | 리소스 소유자 여부 |
| 리소스 상태/조건 고려 | 문서 상태(draft/published/archived)에 따른 접근 제어 |
| 정책 변경 격리 | 권한 정책이 바뀌어도 API 코드를 건드리지 않음 |

### 7-3. 분리 원칙

```
❌ 금지 패턴:
API Handler 내부에서:
  if context.actor_type == "admin":
    allow()
  elif context.actor_id == document.owner_id:
    allow()
  else:
    deny()

✅ 권장 패턴:
API Handler에서:
  result = authz.check(context, resource="document", action="edit", resource_id=documentId)
  if not result.allowed:
    raise PermissionDeniedError(result.reason)
```

**원칙: API 코드 내부에 권한 if문이 흩어지는 구조를 지양한다.** 권한 판단은 Authorization Layer에 위임하고, API는 결과만 받는다.

---

## 8. 리소스 ACL과 API 접근 통제의 연결 방식

### 8-1. 두 층위의 접근 통제

| 층위 | 설명 | 예시 |
|---|---|---|
| Endpoint Guard | API 엔드포인트 자체에 접근 가능한지 | `/api/v1/audit-logs`는 Admin만 접근 |
| Resource Authorization | 특정 리소스에 대한 특정 행위 가능 여부 | 문서 A에 대해 edit/share/publish 가능한지 |

**이 두 층위는 독립적으로 동작하며 모두 통과해야 접근이 허용된다.**

### 8-2. 역할 기반 기본 권한 + ACL Override

```
기본 권한 흐름:
  1. 역할(role) 기반 기본 권한 확인
     → 역할이 해당 행위를 전혀 허용하지 않으면 즉시 거부
  2. 리소스별 ACL 확인
     → 역할로는 허용되어도 리소스 ACL이 거부하면 최종 거부
     → 역할로는 불허여도 명시적 ACL grant가 있으면 허용 가능
```

### 8-3. 리소스 유형별 접근 정책 예시

| 리소스 | 접근 통제 특징 |
|---|---|
| documents | 소유자/공유자/조직 멤버별 권한 차이. ACL 필수. |
| versions | 상위 document 권한을 상속하되, 버전별 락/삭제 권한 별도 가능 |
| nodes | 버전 권한 상속. 노드 단위 fine-grained 접근은 구현 Phase에서 결정 |
| audit-logs | 기본 읽기 전용. Admin 전용. |
| permissions | 리소스 소유자 + Admin만 변경 가능 |

### 8-4. 목록 조회의 Authorization-Aware Filtering

- 목록 API(`GET /documents`)는 요청자가 볼 수 없는 리소스를 응답에 포함해서는 안 된다.
- 전체 목록을 반환하고 프론트엔드에서 필터링하는 방식은 금지.
- **Authorization-aware query**: DB 쿼리 단계에서 접근 가능한 리소스만 조회하거나, 결과를 권한 필터로 후처리.
- 세부 구현은 Task 3-6(목록 조회)에서 결정.

---

## 9. 관리자 API / 외부 API / 내부 API 구분 원칙

### 9-1. 기본 방향

**원칙: 공통 플랫폼 API를 우선한다. 분리는 예외다.**

동일한 리소스 path에서 요청자의 역할에 따라 접근 가능 범위가 달라지는 것을 기본으로 한다.

### 9-2. Admin Namespace 허용 케이스

다음 경우에 `/api/v1/admin/` prefix를 선택적으로 허용한다:
- 일반 사용자에게 절대 노출해서는 안 되는 시스템 제어 기능
- 시스템 설정, 전역 정책 관리, 사용자 강제 조치 등

```
/api/v1/documents         → 공통 (역할로 접근 범위 결정)
/api/v1/audit-logs        → 공통 (Admin만 인가)
/api/v1/admin/system-config      → Admin namespace (예외)
/api/v1/admin/users/{id}:force-deactivate  → Admin namespace (예외)
```

**주의**: `/admin/` namespace는 인가의 대체 수단이 아니다. namespace에도 인가 검증은 반드시 적용.

### 9-3. 내부 API vs 외부 공개 API

| 구분 | 처리 방향 |
|---|---|
| 내부 서비스 간 통신 | Service identity로 인증. 동일 인가 원칙 적용. |
| 외부 공개 API | 공통 플랫폼 API. API Key/Service Token으로 인증. scope 제한 |
| User UI용 API | 공통 플랫폼 API. 별도 분리 불필요. |

---

## 10. 비동기 작업, 웹훅, 이벤트, 서비스 계정의 보안 문맥

### 10-1. 비동기 작업(Async Job)

**핵심 질문**: background job은 누구의 권한으로 동작하는가?

**원칙**:
- 비동기 작업은 원 요청자(original actor)의 권한 범위 내에서만 동작한다.
- 시스템 승격(escalate privilege) 금지 — job runner가 원 요청자보다 넓은 권한을 갖지 않는다.
- 원 요청 시점의 Security Context(actor_id, tenant_id, scope)를 **위임 컨텍스트**로 job 생성 시 저장한다.
- Job이 실행될 때 저장된 위임 컨텍스트를 복원하여 권한 판단에 사용한다.

```
사용자 요청 → operation 생성 (위임 컨텍스트 저장)
   ↓
Background Job 실행 (저장된 컨텍스트로 권한 판단)
   ↓
결과를 operation 리소스에 기록
```

### 10-2. 웹훅

**발신 (플랫폼 → 외부)**:
- 플랫폼이 외부 시스템으로 이벤트를 전달
- 발신 자체는 시스템 service identity로 동작
- 감사 로그: webhook_id, event_type, delivery_id, target_url, result

**수신 (외부 → 플랫폼)**:
- 외부 시스템이 플랫폼에 이벤트/데이터를 전송하는 경우
- 수신 검증 원칙: HMAC 서명 검증 (구체 알고리즘은 구현 Phase에서 결정)
- 서명 검증 실패 시: 즉시 거부 (400 또는 401), 상세 실패 이유 외부 노출 금지
- 수신 처리도 Security Context 구성 후 동일 인가 흐름 적용

### 10-3. 내부 서비스

- 서비스 간 통신에 사람 사용자 토큰 재사용 금지
- 각 내부 서비스는 **고유한 service identity**를 가진다
- Service identity도 Security Context의 `actor_type=internal_service`로 표현
- 서비스별 허용 행위 범위를 사전 정의

### 10-4. AI / Agent 소비자

두 가지 동작 모드를 구분한다:

| 모드 | 설명 | 보안 처리 |
|---|---|---|
| **Delegated Action** | 사용자를 대리하여 동작 | Security Context에 `delegating_user_id` 포함. 사용자 권한 범위 이내. |
| **Autonomous Action** | 독립 service account로 동작 | Service Token으로 인증. 명시적 scope 제한. |

- AI는 `delegating_user_id`가 가진 권한을 초과할 수 없다.
- Autonomous AI agent는 최소 권한 원칙을 적용한 service account로 등록.

---

## 11. 권한 실패 및 보안 오류 응답 원칙

*(세부 오류 포맷은 Task 3-10에서 확정. 여기서는 방향만 정리.)*

### 11-1. 인증 실패 vs 권한 부족 구분

| 상황 | HTTP Status | 설명 |
|---|---|---|
| 인증 정보 없음/만료 | 401 Unauthorized | "당신이 누구인지 모른다" |
| 인증 성공, 권한 없음 | 403 Forbidden | "당신을 알지만 허용하지 않는다" |
| 리소스 존재 여부를 숨겨야 하는 경우 | 404 Not Found | 보안상 존재 자체를 노출하지 않아야 하는 경우 (예: 다른 테넌트 리소스) |

### 11-2. 보안 오류 응답 원칙

- **외부 응답은 최소화**: 내부 정책 구조, 역할 이름, 권한 로직 상세를 외부 응답에 포함하지 않는다.
- **내부 추적은 충분히**: 실패 요청도 `request_id`, `actor_id`, `resource`, `action`, `reason(내부용)` 감사 로그에 기록.
- **일관된 오류 구조**: 인증/인가 오류도 플랫폼 공통 오류 포맷을 따른다 (Task 3-10에서 확정).

---

## 12. 감사 추적 및 보안 관측성 연계

### 12-1. 감사 기록 대상

| 구분 | 기록 여부 |
|---|---|
| 성공한 쓰기/변경 요청 | 필수 |
| 성공한 읽기 요청 (민감 리소스) | 권장 |
| 실패한 접근 시도 | 권장 (보안 감사 목적) |
| 권한 변경 행위 | 필수 |
| 관리자 행위 | 필수 |
| impersonation / delegation | 필수 (고위험 문맥) |
| 서비스 계정 행위 | 필수 |

### 12-2. 감사 로그 핵심 필드

| 필드 | 설명 |
|---|---|
| `actor_id` | 행위 주체 |
| `actor_type` | 주체 유형 |
| `tenant_id` | 조직 스코프 |
| `auth_method` | 사용된 인증 수단 |
| `action` | 수행한 행위 |
| `resource_type` | 대상 리소스 유형 |
| `resource_id` | 대상 리소스 식별자 |
| `result` | 성공/실패 |
| `request_id` | 요청 추적 ID |
| `timestamp` | 발생 시각 |
| `original_actor_id` | delegation/async 시 원 요청자 |
| `ip_address` | 요청 출처 (사람 요청 시) |

### 12-3. 보안 로그 vs 비즈니스 감사 로그 분리

| 구분 | 보안 로그 | 비즈니스 감사 로그 |
|---|---|---|
| 목적 | 침해 탐지, 이상 접근 감지 | 누가 무엇을 했는지 증적 |
| 포함 내용 | 실패 접근, 인증 오류, 비정상 패턴 | 성공한 중요 행위 |
| 소비자 | 보안팀, SIEM | 감사팀, 컴플라이언스 |
| 보존 기간 | 단~중기 | 장기 |

---

## 13. 후속 Task에 전달할 설계 기준

| Task | 전달 기준 |
|---|---|
| Task 3-4 (버전 관리) | 인증 방식이 변경되어도 API 버전 전략이 무너지지 않도록. Security Context 추상화로 인증 교체 격리. |
| Task 3-5 (요청/응답 포맷) | request_id/trace_id를 응답 envelope에 포함. Security Context의 메타데이터를 응답 규약과 연결. |
| Task 3-6 (목록 조회) | authorization-aware filtering 필수. 권한 없는 리소스는 목록에도 포함하지 않음. |
| Task 3-7 (idempotency) | idempotency key는 actor context와 함께 저장. 다른 actor가 동일 key로 중복 처리하는 케이스 정의 필요. |
| Task 3-8 (async/event/webhook) | 비동기 작업의 위임 컨텍스트 저장 방식, 웹훅 서명 검증 원칙 구체화. |
| Task 3-10 (오류 모델) | 401/403/404 보안 오류 응답 구조. 내부 최소화 원칙과 공통 오류 포맷 연결. |
| Phase 4 이후 구현 | auth middleware / dependency 구조, Authorization Layer와 policy engine 경계, ACL DB 쿼리 최적화. |

---

## 14. 결론

Mimir 플랫폼 API의 보안 구조는 다음 세 가지를 기반으로 한다.

1. **Request Security Context**: 인증 결과를 구조화된 Context로 변환하여 모든 내부 계층에 전달
2. **계층 분리**: API Layer는 Context 구성과 위임, Authorization Layer는 권한 판단 전담
3. **모든 주체 포용**: 사람, 서비스, AI, 비동기 작업, 웹훅 — 모두 동일한 보안 모델 안에서 처리

이 구조는 인증 방식이 바뀌어도 API 보안 로직이 유지되고, 새로운 소비자 유형이 추가되어도 확장 가능한 기반이다.

---

## 자체 점검 요약

### 구분한 Actor 유형 목록

일반 사용자 / 관리자 사용자 / 외부 애플리케이션 / 내부 서비스 / Background Worker / Webhook / AI Agent / Service Account

### 보안 상위 원칙 요약

Default Deny / Least Privilege / Explicit Actor Context / Tenant-Aware Access / Separation of AuthN & AuthZ / API-Level Enforcement / No UI-Only Security / Auditable Access / Service Identity Support / Extensible Credential Model

### Request Security Context 핵심 필드 요약

`actor_id`, `actor_type`, `tenant_id`, `roles`, `auth_method`, `request_id`, `trace_id`, `is_impersonation`, `original_actor_id`, `delegating_user_id`, `scope`, `human_initiated`

### API Layer / Authorization Layer 책임 분리 요약

- API Layer: Context 구성, 입력 검증, 인가 레이어 위임, 응답 생성
- Authorization Layer: role/ACL/리소스 상태 기반 판단 전담
- API 코드 내 권한 if문 금지

### Async/Webhook/Service/AI 보안 관점 요약

- Async Job: 원 요청자 위임 컨텍스트 저장, 권한 초과 금지
- Webhook 수신: HMAC 서명 검증, 실패 시 즉시 거부
- 내부 서비스: 고유 service identity, 사람 토큰 재사용 금지
- AI Agent: Delegated vs Autonomous 모드 구분, 각각 권한 제한 적용

### 후속 Task 연결 포인트

Task 3-4(인증 교체 격리), 3-5(request_id 응답 포함), 3-6(authorization-aware filtering), 3-7(actor context + idempotency), 3-8(위임 컨텍스트 및 webhook 서명), 3-10(보안 오류 응답 구조)

### 의도적으로 미결정으로 남긴 항목

- 구체 인증 프로토콜 채택 (JWT/OAuth/SAML/OIDC 등) — 구현 Phase에서 결정
- JWT claim 스키마 상세 — 구현 Phase에서 결정
- HMAC 서명 알고리즘 선택 — Task 3-8 또는 구현 Phase에서 결정
- ACL DB 스키마 및 쿼리 최적화 — 구현 Phase에서 결정
- Active org context 전달 방식 (token claim vs 헤더) — 구현 Phase에서 결정
- Policy engine 제품/라이브러리 선택 — 구현 Phase에서 결정
- Authorization-aware filtering 구체 구현 방식 — Task 3-6에서 결정
- Admin namespace 범위 최종 확정 — Task 3-3 기준 구현 Phase에서 적용
