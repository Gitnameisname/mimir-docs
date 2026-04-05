# Task8_event_webhook_extension.md
# 비동기 작업 / 이벤트 / 웹훅 확장 모델 설계

---

## 1. 문서 목적

이 문서는 Mimir 플랫폼 API를 동기 REST 호출에만 머무르지 않고, **비동기 작업(async operations), 플랫폼 이벤트(events), 외부 웹훅(webhooks)**으로 확장할 수 있는 상위 설계 기준을 정의한다.

이 문서는 다음을 확정한다:
- 어떤 작업이 비동기 처리 대상인지와 구분 기준
- `operations` vs `jobs` 용어 선택과 리소스 모델 방향
- 비동기 작업의 상태 모델
- 플랫폼 이벤트의 의미, 분류, 공통 속성
- 웹훅 구독(subscription)과 전달(delivery) 리소스 모델
- 실패/재시도/idempotency 정책 방향
- AI/RAG/인덱싱/대량 작업까지 포괄하는 확장 관점

이 문서는 Task 3-7(idempotency)을 기반으로 하며, Task 3-9(AI/RAG), Task 3-10(오류 모델)의 기준이 된다.

---

## 2. 왜 비동기 작업 / 이벤트 / 웹훅 확장 모델이 필요한가

### 2-1. 모든 작업이 요청-응답 한 번으로 끝나지 않는다

| 작업 예시 | 이유 |
|---|---|
| 대용량 문서 export | 수 MB~GB, 처리에 수십 초~분 소요 |
| 검색 인덱스 재생성 (reindex) | 수만 건 문서 처리, 비동기 필수 |
| AI 임베딩/ingestion | 외부 AI 서비스 연동, 처리 시간 불확실 |
| Bulk archive/delete | 수백~수천 건 처리 |
| Preview 재생성 | 외부 렌더링 서비스 의존 |
| Validation batch | 다수 문서 일괄 검증 |

동기 HTTP 응답(30초 이내)으로는 처리할 수 없거나, 처리 중 진행 상태를 소비자가 알아야 하는 경우가 있다.

### 2-2. 외부 시스템은 polling보다 event/webhook을 선호한다

- 외부 통합 시스템이 작업 완료, 문서 변경, 권한 변경 등을 polling으로 확인하면 불필요한 API 부하가 발생한다.
- event 기반 / webhook push 방식이 더 효율적이고 실시간에 가깝다.

### 2-3. UI와 운영 도구도 상태/실패 정보가 필요하다

- Admin UI: "현재 실행 중인 작업", "실패한 웹훅 delivery" 등을 보여줘야 한다.
- 개발자: 작업 실패 원인, 재시도 이력, 이벤트 수신 여부를 진단해야 한다.

**이 확장 모델은 부가 기능이 아니라 플랫폼을 실제 운영 가능한 수준으로 만드는 핵심 확장 계층이다.**

---

## 3. 상위 설계 원칙

### 원칙 1. REST-first, Async-when-needed

**의미**: 기본은 동기 REST. 오래 걸리거나 후처리가 많은 작업만 비동기 모델로 승격.

**왜**: 불필요한 복잡성을 피한다. 짧게 끝나는 작업을 비동기로 만들면 소비자 부담만 늘어난다.

**설계 판단**: 즉시 응답 가능한 작업은 동기로 유지. 처리 시간이 수 초를 넘거나 외부 의존이 있으면 `202 Accepted` + operation.

---

### 원칙 2. Explicit Operation Tracking

**의미**: 비동기 작업은 추적 가능한 operation 리소스로 표현된다.

**왜**: 소비자가 작업 상태를 폴링하거나 조회할 수 있어야 한다. 블랙박스 처리 금지.

**설계 판단**: 모든 비동기 작업 시작 시 `operation_id`를 반환하고, `GET /operations/{id}`로 상태 조회 가능.

---

### 원칙 3. Events as Platform Facts

**의미**: 이벤트는 단순 로그가 아니라 "플랫폼에서 무슨 일이 일어났는가"를 표현하는 계약적 사실 단위다.

**왜**: 이벤트가 내부 구현 세부처럼 설계되면 외부 소비자가 안정적으로 의존할 수 없다.

**설계 판단**: 외부에 노출하는 이벤트는 플랫폼 계약의 일부로 관리. 내부 구현 변경으로 이벤트 스키마가 바뀌면 버전 관리 대상.

---

### 원칙 4. Webhooks as External Delivery Mechanism

**의미**: 웹훅은 이벤트를 외부 시스템으로 push 전달하는 수단이다. 이벤트 그 자체가 아니다.

**왜**: 이벤트와 웹훅을 동일시하면 설계가 혼란스러워진다. 이벤트는 발생하는 것, 웹훅은 전달하는 것.

**설계 판단**: 이벤트 모델과 웹훅 전달 모델을 분리 설계. 웹훅은 이벤트 구독 필터를 가진 전달 채널.

---

### 원칙 5. Decoupled but Observable

**의미**: 비동기 작업, 이벤트, 웹훅은 느슨하게 결합되지만 연결 가능하고 추적 가능해야 한다.

**왜**: 느슨한 결합이 유지보수를 쉽게 하지만, 추적이 불가능하면 운영이 어렵다.

**설계 판단**: `request_id`, `operation_id`, `event_id`, `delivery_id`를 연결하여 하나의 요청에서 발생한 모든 비동기 결과를 추적 가능.

---

### 원칙 6. Idempotent-friendly Delivery

**의미**: 비동기 작업 트리거와 웹훅 전달은 at-least-once를 기본으로 설계. 수신자는 idempotent consumer를 지향.

**왜**: exactly-once를 보장하는 것은 매우 어렵다. 현실적인 설계는 중복을 허용하되 소비자가 안전하게 처리하도록 지원.

**설계 판단**: 이벤트와 delivery에 고유 ID 부여. 수신자가 중복 처리를 방지할 수 있게 `event_id`/`delivery_id` 제공.

---

### 원칙 7. Secure by Default

**의미**: 비동기 작업의 actor context, 이벤트의 tenant scope, 웹훅 서명 검증은 기본 요구사항이다.

**왜**: 비동기 경로가 보안 통제를 우회하는 경로가 되어서는 안 된다.

**설계 판단**: operation은 시작한 actor context 보존. 이벤트는 tenant 경계를 넘지 않음. 웹훅 발신에 서명 포함.

---

### 원칙 8. Auditable Lifecycle

**의미**: operation 시작, 완료, 실패; 이벤트 발행; 웹훅 delivery 시도 — 모두 감사 추적 대상이다.

**왜**: 비동기 처리는 추적하지 않으면 사후 분석이 불가능하다.

**설계 판단**: 각 단계의 주요 상태 변화를 감사 로그에 연결.

---

### 원칙 9. Consistent Resource Modeling

**의미**: operations, events, webhooks, deliveries 모두 플랫폼 공통 리소스 모델(Task 3-2, 3-5)을 따른다.

**왜**: 비동기 확장이 별도 API 규약을 가지면 소비자 학습 비용이 증가한다.

**설계 판단**: 공통 응답 envelope(`data`+`meta`), 공통 pagination, 공통 오류 구조를 그대로 적용.

---

### 원칙 10. Extensible to AI and Automation Workflows

**의미**: ingestion, embedding, retrieval sync, AI processing 같은 AI 관련 작업도 동일한 operation/event 모델로 수용.

**왜**: AI 연계를 별도 예외 영역으로 두면 일관성이 무너진다.

**설계 판단**: AI/RAG 관련 작업도 operation 리소스로 표현. AI 이벤트도 플랫폼 이벤트 체계에 포함.

---

## 4. 비동기 작업 모델 설계

### 4-1. 동기 vs 비동기 처리 기준

| 기준 | 동기 처리 | 비동기 처리 |
|---|---|---|
| 처리 시간 | 수 초 이내 확정 | 수 초 이상 또는 불확실 |
| 외부 의존 | 없거나 즉각 완료 | 외부 서비스/큐 의존 |
| 부분 처리 | 없음 | 수천 건 중 일부만 먼저 처리 |
| 상태 추적 필요 | 불필요 | 진행률/완료/실패 추적 필요 |
| 실패 재시도 | 즉시 오류 반환 | 백그라운드 재시도 필요 |

### 4-2. 비동기 처리 대상 예시

| 작업 | 이유 |
|---|---|
| Export (PDF/ZIP) | 처리 시간 불확실, 대용량 |
| Reindex (검색 인덱스 재생성) | 수만 건 문서 처리 |
| AI Ingestion/Embedding | 외부 AI 서비스 의존 |
| AI Retrieval Index Sync | 대규모 벡터 인덱스 업데이트 |
| Bulk archive/delete | 수백~수천 건 |
| Bulk permission update | 다수 리소스 변경 |
| Preview 재생성 | 외부 렌더링 의존 |
| Validation batch | 다수 문서 일괄 검증 |

### 4-3. 비동기 작업 시작 계약

```
1. 클라이언트 → POST /documents/doc_123:reindex
   Headers: Idempotency-Key: key_A

2. 서버 → 202 Accepted
   Body: {
     "data": {
       "operation_id": "op_xyz",
       "type": "reindex",
       "status": "accepted",
       "resource_type": "document",
       "resource_id": "doc_123"
     },
     "meta": {
       "request_id": "req_001",
       "links": { "operation": "/api/v1/operations/op_xyz" }
     }
   }

3. 클라이언트 → GET /api/v1/operations/op_xyz (폴링)
4. 서버 → 200 OK, status: "running" → "succeeded"
```

---

## 5. 작업 리소스: Operations vs Jobs

### 5-1. 후보 비교

**후보 1: `jobs`**

| 항목 | 평가 |
|---|---|
| 장점 | 배치/백그라운드 작업과 잘 맞는 용어. 친숙함. |
| 단점 | "시스템 레벨 배치"의 느낌이 강함. API 요청 파생 작업과 어울리지 않을 수 있음. |
| 외부 API 적합성 | 중간. 소비자에게 내부 구현 느낌을 줄 수 있음. |
| 운영 추적성 | 적합. |
| AI/automation 연계 | 적합. |

**후보 2: `operations`**

| 항목 | 평가 |
|---|---|
| 장점 | API 요청으로부터 파생된 장기 실행 작업 표현에 적합. Google API Design Guide, Stripe 등 업계 표준. |
| 단점 | 일부 소비자에게 덜 친숙할 수 있음. |
| 외부 API 적합성 | 높음. REST 관행과 잘 맞음. |
| 운영 추적성 | 적합. |
| AI/automation 연계 | 높음. LLM tool spec에서도 operation 개념이 자연스러움. |

**후보 3: 역할 분리 (외부 operations + 내부 jobs)**

| 항목 | 평가 |
|---|---|
| 장점 | 외부 계약과 내부 구현을 분리. |
| 단점 | 구현 복잡도 증가. 초기 단계에는 과도함. |

### 5-2. 채택: `operations` (외부 API 계약 용어)

**이유**:
- REST 업계 관행에 부합 (Google Cloud Operations, Stripe API 등)
- "API 요청으로부터 파생된 장기 실행 작업"의 의미를 정확히 표현
- AI/automation 소비자에게도 명확한 개념

**내부 구현 용어**(`jobs`)는 별도. 외부 API 계약에는 `operations`만 노출.

### 5-3. Operation 리소스 핵심 속성

```json
{
  "id": "op_xyz",
  "type": "reindex",
  "status": "running",
  "resource_type": "document",
  "resource_id": "doc_123",
  "actor_id": "user_001",
  "tenant_id": "org_001",
  "created_at": "2026-04-02T13:00:00Z",
  "updated_at": "2026-04-02T13:01:00Z",
  "started_at": "2026-04-02T13:00:05Z",
  "completed_at": null,
  "progress": {
    "processed": 1200,
    "total": 5000,
    "percent": 24
  },
  "result": null,
  "error": null
}
```

---

## 6. 비동기 작업 상태 모델

### 6-1. 상태 정의

| 상태 | 의미 | 중간/최종 |
|---|---|---|
| `accepted` | 요청 수락됨, 큐 대기 전 | 중간 |
| `queued` | 실행 대기 중 | 중간 |
| `running` | 현재 실행 중 | 중간 |
| `succeeded` | 성공 완료 | **최종** |
| `failed` | 실패 완료 | **최종** |
| `canceled` | 취소됨 | **최종** |
| `partial_success` | 일부 성공, 일부 실패 (bulk 작업) | **최종** |

### 6-2. 상태 전이 원칙

```
accepted → queued → running → succeeded
                            ↘ failed
                            ↘ partial_success
                 ↘ canceled (중간 상태에서 취소 가능)
```

- 최종 상태(`succeeded`, `failed`, `canceled`, `partial_success`)에서는 상태 변경 없음.
- `running` 상태의 operation은 `:cancel` action 가능 (지원 여부는 operation type별로 다름).

### 6-3. 실패 상태 정보

실패 시 `error` 필드에 구조화된 정보 포함:
```json
"error": {
  "code": "REINDEX_FAILED",
  "message": "인덱싱 서버 연결 실패로 작업이 중단되었습니다.",
  "type": "server",
  "retryable": true
}
```

### 6-4. 진행률 필드

- `progress.processed`, `progress.total`, `progress.percent` — bulk/ingestion 등 진행 추적 가능한 작업에서 제공.
- 모든 operation type에서 필수가 아님. 지원 여부를 operation type별 문서에 명시.

### 6-5. 결과 리소스 참조

완료된 operation은 결과 리소스를 참조:
```json
"result": {
  "resource_type": "export",
  "resource_id": "export_abc",
  "download_url": "/api/v1/exports/export_abc/download"
}
```

---

## 7. 이벤트 모델 설계

### 7-1. 이벤트의 의미

| 구분 | 설명 |
|---|---|
| **이벤트** | 플랫폼에서 "무슨 일이 발생했는가"를 표현하는 계약적 사실 단위 |
| **감사 로그** | 누가 무엇을 했는지 추적. 모든 요청을 기록. 컴플라이언스 목적. |
| **내부 브로커 메시지** | 구현 세부. 외부 계약 아님. |

이 세 가지는 유사하지만 목적과 소비자가 다르다.

- 모든 내부 상태 변화가 외부 이벤트가 되는 것은 아니다.
- 외부에 노출하는 이벤트는 플랫폼 계약의 일부로 안정적으로 관리해야 한다.

### 7-2. 이벤트 공통 속성

| 필드 | 타입 | 필수 여부 | 설명 |
|---|---|---|---|
| `event_id` | string (UUID) | **필수** | 이벤트 고유 식별자. 중복 처리 방지용. |
| `event_type` | string | **필수** | 이벤트 종류. `{domain}.{action}` 형태. (예: `document.published`) |
| `occurred_at` | ISO 8601 | **필수** | 이벤트 발생 시각 |
| `schema_version` | string | **필수** | 이벤트 payload 스키마 버전 (예: `v1`) |
| `tenant_id` | string | **필수** | 이벤트 발생 조직 |
| `actor_id` | string | 조건부 | 행위 주체 (시스템 이벤트는 없을 수 있음) |
| `actor_type` | string | 조건부 | 주체 유형 (user/service/system) |
| `subject_type` | string | **필수** | 이벤트 대상 리소스 유형 |
| `subject_id` | string | **필수** | 이벤트 대상 리소스 ID |
| `correlation_id` | string | 권장 | 연결된 request_id 또는 operation_id |
| `payload` | object | 조건부 | 이벤트 관련 데이터 요약 |

```json
{
  "event_id": "evt_abc123",
  "event_type": "document.published",
  "occurred_at": "2026-04-02T13:00:00Z",
  "schema_version": "v1",
  "tenant_id": "org_001",
  "actor_id": "user_001",
  "actor_type": "user",
  "subject_type": "document",
  "subject_id": "doc_123",
  "correlation_id": "req_xyz",
  "payload": {
    "document_id": "doc_123",
    "version_id": "ver_456",
    "document_type": "regulation"
  }
}
```

---

## 8. 이벤트 분류 체계

### 8-1. 도메인 이벤트 (Domain Events)

비즈니스 도메인에서 발생하는 의미 있는 사건.

| 이벤트 | 설명 | 외부 공개 | 웹훅 구독 |
|---|---|---|---|
| `document.created` | 문서 생성됨 | ✅ | ✅ |
| `document.updated` | 문서 내용 변경됨 | ✅ | ✅ |
| `document.published` | 문서 게시됨 | ✅ | ✅ |
| `document.archived` | 문서 보관됨 | ✅ | ✅ |
| `document.deleted` | 문서 삭제됨 | ✅ | ✅ |
| `version.created` | 새 버전 생성됨 | ✅ | ✅ |
| `node.updated` | 노드 내용 변경됨 | 조건부 | 조건부 |
| `permission.changed` | 권한 변경됨 | ✅ | ✅ |
| `attachment.added` | 첨부파일 추가됨 | ✅ | ✅ |
| `member.added` | 조직 멤버 추가됨 | Admin | Admin |
| `role.changed` | 역할 변경됨 | Admin | Admin |

### 8-2. 운영 이벤트 (Operational Events)

플랫폼 작업/시스템 운영 관련 사건.

| 이벤트 | 설명 | 외부 공개 | 웹훅 구독 |
|---|---|---|---|
| `operation.started` | 비동기 작업 시작됨 | ✅ | 조건부 |
| `operation.completed` | 비동기 작업 완료됨 | ✅ | ✅ |
| `operation.failed` | 비동기 작업 실패됨 | ✅ | ✅ |
| `webhook.delivery.failed` | 웹훅 전달 실패 | 내부/Admin | ❌ |
| `webhook.delivery.succeeded` | 웹훅 전달 성공 | 내부/Admin | ❌ |

### 8-3. 통합/AI 관련 이벤트 (Integration/AI Events)

AI 파이프라인, 인덱싱, 외부 연동 관련 사건.

| 이벤트 | 설명 | 외부 공개 | 웹훅 구독 |
|---|---|---|---|
| `ingestion.started` | AI ingestion 작업 시작 | ✅ | ✅ |
| `ingestion.completed` | AI ingestion 완료 | ✅ | ✅ |
| `ingestion.failed` | AI ingestion 실패 | ✅ | ✅ |
| `retrieval-index.updated` | 검색 인덱스 갱신 완료 | ✅ | 조건부 |
| `ai-processing.failed` | AI 처리 실패 | ✅ | ✅ |
| `export.completed` | Export 작업 완료 | ✅ | ✅ |

### 8-4. 이벤트 공개 범위 원칙

| 범주 | 기준 |
|---|---|
| **외부 공개** | 외부 시스템 연동, AI agent, 자동화 워크플로우에 유용한 이벤트 |
| **Admin 전용** | 조직/사용자 관리, 보안 관련 이벤트 (일반 사용자 불필요) |
| **내부 전용** | 구현 세부, webhook delivery 상태 (플랫폼 내부 운영) |

---

## 9. 웹훅 구독 모델 설계

### 9-1. 웹훅 구독 리소스 (`webhooks`)

웹훅 구독은 `POST /api/v1/webhooks`로 등록하는 독립 리소스.

```json
{
  "id": "wh_001",
  "target_url": "https://external.example.com/hooks/mimir",
  "subscribed_events": [
    "document.published",
    "document.archived",
    "operation.completed"
  ],
  "status": "active",
  "filters": {
    "document_type": ["regulation", "policy"]
  },
  "created_by": "user_001",
  "tenant_id": "org_001",
  "created_at": "2026-04-01T10:00:00Z",
  "updated_at": "2026-04-01T10:00:00Z",
  "secret_reference": "wh_secret_ref_001"
}
```

### 9-2. 구독 리소스 핵심 속성

| 필드 | 설명 |
|---|---|
| `id` | 구독 고유 ID |
| `target_url` | 이벤트를 전달할 외부 엔드포인트 |
| `subscribed_events` | 구독할 이벤트 타입 목록 |
| `status` | `active` / `inactive` / `suspended` |
| `filters` | 이벤트 필터 조건 (선택). 특정 document_type, organization 등 |
| `secret_reference` | 서명 secret의 참조 ID (secret 값 자체는 노출하지 않음) |
| `created_by` | 구독을 생성한 actor |
| `tenant_id` | 구독이 속한 조직. 조직 범위를 넘는 이벤트 전달 금지. |

### 9-3. 구독 범위 원칙

- 구독은 항상 특정 `tenant_id` 범위 내 이벤트만 전달한다.
- 플랫폼 전역 구독(모든 tenant 이벤트 수신)은 Super Admin에게만 허용.
- 리소스 단위 구독 (`document_id` 기준)은 초기 단계에서는 지원하지 않고, `subscribed_events` + `filters`로 처리.

### 9-4. 검증 흐름 (선택)

- 구독 등록 시 `target_url`에 검증 요청(challenge/verify)을 보내는 방식 검토.
- 구현 Phase에서 결정. 초기에는 등록 후 즉시 active 처리 가능.

---

## 10. 웹훅 전달(Delivery) 모델 설계

### 10-1. Delivery는 Subscription과 별개의 운영 리소스

- `webhook-deliveries`는 특정 구독에 대한 개별 전달 시도를 추적하는 리소스.
- 하나의 이벤트 → 구독한 모든 webhook → 각각 별도 delivery 생성.

```json
{
  "id": "wdl_xyz",
  "webhook_id": "wh_001",
  "event_id": "evt_abc123",
  "event_type": "document.published",
  "status": "failed",
  "attempt_count": 3,
  "first_attempt_at": "2026-04-02T13:00:05Z",
  "last_attempt_at": "2026-04-02T13:15:00Z",
  "next_retry_at": "2026-04-02T14:15:00Z",
  "response_status": 503,
  "response_body_summary": "Service Unavailable",
  "permanently_failed": false
}
```

### 10-2. Delivery 상태

| 상태 | 의미 |
|---|---|
| `pending` | 전달 대기 중 |
| `sending` | 전달 시도 중 |
| `succeeded` | 성공적으로 전달됨 (2xx 응답) |
| `failed` | 전달 실패 (재시도 예정) |
| `permanently_failed` | 최대 재시도 횟수 초과, 전달 포기 |

### 10-3. 전달 조회 API

- `GET /api/v1/webhooks/{webhookId}/deliveries` — 구독별 전달 이력 조회
- `POST /api/v1/webhooks/{webhookId}/deliveries/{deliveryId}:resend` — 수동 재전송

### 10-4. 노출 범위

| 소비자 | 노출 범위 |
|---|---|
| 구독 소유자 | 본인 구독의 delivery 조회/재전송 |
| Admin | 전체 delivery 조회 및 관리 |
| 일반 사용자 | 불필요 |

---

## 11. 실패 / 재시도 / 중복 / Idempotency 관점

### 11-1. 비동기 작업의 중복 생성 방지

- 작업 시작 endpoint에 `Idempotency-Key` 적용 (Task 3-7).
- 동일 키 재요청 시 동일 `operation_id` 반환. 중복 job 생성 없음.

### 11-2. 이벤트의 고유 식별

- 모든 이벤트는 `event_id`(UUID)를 가진다.
- 소비자는 `event_id`를 기준으로 중복 이벤트를 감지할 수 있다.
- 같은 `event_id`의 재전달은 동일 이벤트로 처리 (idempotent consumer 책임).

### 11-3. 웹훅 Delivery: At-least-once 정책

- 웹훅은 **at-least-once delivery**를 기본으로 한다.
- exactly-once를 보장하지 않는다. 이를 문서에 명시한다.
- 수신자(외부 시스템)는 **idempotent consumer**로 동작해야 한다:
  - `event_id` 또는 `delivery_id`를 저장하고 중복 수신 시 무시.

### 11-4. 재시도 원칙 (방향 수준)

- 실패한 delivery는 지수 백오프(exponential backoff) 방식으로 재시도.
- 최대 재시도 횟수 초과 시 `permanently_failed` 상태로 전환.
- 재시도 간격, 최대 횟수 수치는 구현 Phase에서 결정.

### 11-5. 동일 Delivery 재시도 vs 새 Delivery 구분

- 재시도(`attempt_count` 증가)는 동일 `delivery_id`를 유지한다.
- 수동 재전송(`:resend` action)은 새 `delivery_id`를 생성한다.
- 수신자는 `event_id`로 중복을 감지한다 (`delivery_id`와 무관하게).

---

## 12. 보안 및 인증/서명 관점

### 12-1. 비동기 작업의 Actor Context

- 모든 operation은 시작한 actor의 Security Context를 보존한다.
- background worker가 실행할 때 원 요청자의 권한 범위를 초과하지 않는다 (Task 3-3 원칙).

### 12-2. 이벤트의 Tenant 격리

- 이벤트는 항상 생성된 `tenant_id` 스코프 내에서만 전달된다.
- 다른 tenant의 이벤트를 웹훅으로 수신하는 것은 허용하지 않는다.

### 12-3. 웹훅 발신 서명 원칙

- 플랫폼이 외부로 웹훅을 발신할 때 서명을 포함한다.
- **목적**: 수신자가 발신자(플랫폼)를 검증할 수 있도록.
- **서명 원칙**: HMAC 기반 서명. 구체 알고리즘(SHA-256 등)은 구현 Phase에서 결정.
- 서명은 요청 헤더에 포함: `X-Mimir-Signature: ...`
- secret은 구독 생성 시 일회 반환하고 이후 API로 조회 불가 (`secret_reference`만 노출).

### 12-4. 재전송 공격 방지

- 서명에 timestamp를 포함하여 오래된 payload 재사용 방지.
- 수신자는 timestamp가 허용 범위(예: 5분) 이내인지 검증해야 한다.
- 구체 검증 방식은 구현 Phase에서 문서화.

---

## 13. 운영 관측성 및 감사 추적 연계

### 13-1. 추적 연결 체계

하나의 사용자 요청에서 발생하는 모든 비동기 결과를 추적할 수 있어야 한다:

```
request_id → operation_id → event_id → delivery_id
```

| 연결 | 방법 |
|---|---|
| request_id → operation_id | operation 생성 시 `correlation_id: request_id` |
| operation_id → event_id | operation 완료 이벤트에 `correlation_id: operation_id` |
| event_id → delivery_id | delivery 생성 시 `event_id` 참조 |

### 13-2. 감사 로그 연계

| 이벤트 | 감사 로그 기록 |
|---|---|
| Operation 시작 | actor, operation_id, type, resource 기록 |
| Operation 완료/실패 | 상태 변경, 결과 기록 |
| 이벤트 발행 | event_id, event_type, subject 기록 |
| Webhook delivery 시도 | delivery_id, attempt, response 기록 |
| Webhook 재전송 (수동) | actor, delivery_id, operation 기록 |

### 13-3. 운영 도구 지원

- `GET /api/v1/operations?status=failed` → 실패한 작업 목록
- `GET /api/v1/webhooks/{id}/deliveries?status=permanently_failed` → 전달 실패 이력
- `GET /api/v1/events?occurred_after=...` → 이벤트 조회 (Admin)
- 실패 원인, 재시도 이력, 진행률이 운영 도구에서 조회 가능해야 함

---

## 14. AI / RAG / 인덱싱 / 대량 작업 확장 관점

### 14-1. AI 관련 작업을 Operation 모델로 수용

AI/RAG 관련 작업은 모두 동일한 `operations` 리소스로 표현한다.

| AI 작업 | Operation Type |
|---|---|
| AI 문서 ingestion 시작 | `ingestion` |
| 임베딩 생성/갱신 | `embedding_update` |
| 검색 인덱스 동기화 | `retrieval_index_sync` |
| AI 처리 결과 검증 | `ai_validation` |

### 14-2. AI/RAG 관련 이벤트도 플랫폼 이벤트 체계에 포함

- `ingestion.completed`, `retrieval-index.updated` 등이 플랫폼 이벤트로 발행됨.
- AI agent는 이 이벤트를 웹훅 구독으로 수신하거나 `GET /api/v1/events`로 폴링 가능.

### 14-3. Agentic Workflow 지원

AI agent/automation이 이 확장 모델을 활용하는 방식:

```
1. AI agent → POST /documents:reindex (Idempotency-Key 포함)
2. 서버 → 202 Accepted, operation_id: op_xyz
3. AI agent → 이벤트 웹훅 구독 (operation.completed)
4. 플랫폼 → ingestion 완료 시 webhook push
5. AI agent → 이벤트 수신, retrieval pipeline 갱신
```

또는 polling 방식:
```
3. AI agent → GET /operations/op_xyz (반복 폴링)
4. 서버 → status: "succeeded"
5. AI agent → 후속 처리 실행
```

### 14-4. 대량 작업도 동일 Operation 모델 재사용

- Bulk archive, bulk permission update 등도 `operations` 리소스로 표현.
- `progress.processed`, `progress.total`로 진행률 추적.
- `partial_success` 상태를 통해 일부 성공/실패 구분.

---

## 15. 예시 시나리오

### 시나리오 1: 문서 Export 시작 → Operation 추적

```
// 1. Export 시작
POST /api/v1/documents/doc_123:export
Idempotency-Key: key_export_01
Body: { "format": "pdf" }

→ 202 Accepted
{
  "data": {
    "operation_id": "op_exp001",
    "type": "export",
    "status": "accepted",
    "resource_type": "document",
    "resource_id": "doc_123"
  },
  "meta": { "links": { "operation": "/api/v1/operations/op_exp001" } }
}

// 2. 상태 폴링
GET /api/v1/operations/op_exp001
→ { "data": { "status": "running", "progress": { "percent": 60 } } }

// 3. 완료
GET /api/v1/operations/op_exp001
→ {
    "data": {
      "status": "succeeded",
      "result": { "resource_type": "export", "download_url": "/api/v1/exports/exp_abc/download" }
    }
  }
```

### 시나리오 2: document.published 이벤트 → 웹훅 전달

```
// 이벤트 발생: doc_123 게시됨
// 이벤트: { event_id: "evt_001", event_type: "document.published", subject_id: "doc_123" }

// 구독 중인 wh_001의 target_url로 POST 전달:
POST https://external.example.com/hooks/mimir
X-Mimir-Signature: hmac_signature
X-Mimir-Delivery-Id: wdl_xyz

{
  "event_id": "evt_001",
  "event_type": "document.published",
  "occurred_at": "2026-04-02T13:00:00Z",
  "schema_version": "v1",
  "tenant_id": "org_001",
  "subject_type": "document",
  "subject_id": "doc_123",
  "payload": { "document_id": "doc_123", "version_id": "ver_456" }
}
```

### 시나리오 3: Webhook Delivery 실패 → 재시도 → 상태 조회

```
// 전달 실패 (503 응답)
// 플랫폼이 exponential backoff로 재시도

// 상태 조회
GET /api/v1/webhooks/wh_001/deliveries/wdl_xyz
→ {
    "data": {
      "status": "failed",
      "attempt_count": 3,
      "next_retry_at": "2026-04-02T14:00:00Z",
      "response_status": 503
    }
  }

// 수동 재전송
POST /api/v1/webhooks/wh_001/deliveries/wdl_xyz:resend
→ 202 Accepted, new delivery_id: wdl_xyz2
```

### 시나리오 4: Reindex + 동일 Idempotency-Key 재요청

```
// 최초 요청
POST /api/v1/documents/doc_123:reindex
Idempotency-Key: key_reindex_01
→ 202 Accepted, operation_id: op_ri001

// 네트워크 오류 후 재시도 (동일 Key)
POST /api/v1/documents/doc_123:reindex
Idempotency-Key: key_reindex_01
→ 202 Accepted (replay), operation_id: op_ri001 (동일)
   meta.idempotency.replayed: true
```

### 시나리오 5: AI Ingestion → 진행률 업데이트 → 완료 이벤트

```
// 1. Ingestion 시작
POST /api/v1/documents/doc_123:ingest
→ 202 Accepted, operation_id: op_ing001

// 2. 진행률 업데이트
GET /api/v1/operations/op_ing001
→ { "status": "running", "progress": { "processed": 500, "total": 2000, "percent": 25 } }

// 3. 완료
→ { "status": "succeeded", "result": { "indexed_chunks": 2000 } }

// 4. 이벤트 발행: ingestion.completed
// → 구독한 외부 시스템 / AI agent에 webhook 전달
```

---

## 16. 후속 Task 및 구현 Phase에 전달할 기준

| 대상 | 전달 기준 |
|---|---|
| Task 3-9 (AI/RAG) | AI ingestion, embedding, retrieval sync는 `operations` 리소스로 표현. AI agent가 operation 폴링 또는 이벤트 구독으로 상태 확인. `ingestion.completed` 등 AI 이벤트 체계 구체화. |
| Task 3-10 (오류 모델) | operation 실패 error 구조. webhook delivery 실패 오류 코드. `permanently_failed` 이후 처리 정책. delivery 오류 응답 구조. |
| 구현 Phase | operation store(status 추적 DB). event publisher(이벤트 발행 파이프라인). webhook dispatcher(전달 엔진). delivery tracker(재시도 스케줄러). 서명 생성/검증 미들웨어. |
| 별도 설계 | 메시지 브로커 선택(Kafka/Redis/SQS 등). retry backoff 수치. dead-letter 처리. SSE/WebSocket 실시간 채널(선택적 확장). 서명 알고리즘 최종 선택. |

---

## 17. 결론

Mimir 플랫폼 API의 비동기/이벤트/웹훅 확장 모델은 다음으로 요약된다:

1. **Operations**: 비동기 작업의 외부 계약 단위. `202 Accepted` + `operation_id` + 상태 폴링.
2. **Events**: 플랫폼 사실 단위. `event_id` + `event_type` + actor/tenant/subject. 도메인/운영/AI 세 범주.
3. **Webhooks**: 이벤트를 외부로 push하는 전달 채널. subscription 리소스 + delivery 추적.
4. **At-least-once**: exactly-once 미보장. `event_id`/`delivery_id`로 idempotent consumer 지원.
5. **Security**: actor context 보존, tenant 격리, HMAC 서명, secret 비노출.
6. **AI 포함**: AI 관련 작업도 동일한 operation/event 모델 안에서 수용.

---

## 자체 점검 요약

### 비동기 작업 모델 핵심 요약

처리 시간/외부 의존/상태 추적 필요 기준으로 비동기 결정. `202 Accepted` + `operation_id` 반환. 상태 폴링으로 추적.

### Jobs/Operations 용어 선택 방향 요약

외부 API 계약 용어: `operations` 채택. 내부 구현은 `jobs` 가능. 이유: REST 업계 표준, API 요청 파생 작업 표현에 적합.

### 이벤트 분류 체계 요약

도메인 이벤트(document.published 등), 운영 이벤트(operation.completed 등), AI/통합 이벤트(ingestion.completed 등). 모두 `event_id` + `schema_version` 포함.

### 웹훅 Subscription/Delivery 모델 요약

- Subscription: target_url + subscribed_events + filters + secret_reference + status
- Delivery: delivery_id + webhook_id + event_id + status + attempt_count + next_retry_at
- 재시도: 동일 delivery_id 유지. 수동 재전송은 새 delivery_id.

### 실패/재시도/Idempotency 관점 요약

비동기 작업: Idempotency-Key로 중복 operation 방지. 이벤트: event_id로 중복 감지. 웹훅: at-least-once + idempotent consumer. exactly-once 미보장 명시.

### 보안/관측성/감사 연계 요약

actor context 보존(operation). tenant 격리(이벤트). HMAC 서명 + timestamp(웹훅). request_id → operation_id → event_id → delivery_id 추적 체인.

### AI/RAG/인덱싱 확장 관점 요약

ingestion/embedding/retrieval sync 모두 operations 리소스로 수용. AI 이벤트(ingestion.completed 등) 플랫폼 이벤트 체계 포함. Agentic workflow는 operation 폴링 또는 이벤트 구독으로 상태 확인.

### 후속 Task 연결 포인트

Task 3-9(AI operation/event 소비 구체화), Task 3-10(operation/delivery 오류 모델), 구현 Phase(operation store, event publisher, webhook dispatcher)

### 의도적으로 후속 Task로 남긴 미결정 사항

- 메시지 브로커 제품 선택 → 구현 Phase
- Retry backoff 수치/최대 재시도 횟수 → 구현 Phase
- 웹훅 서명 알고리즘 확정 → 구현 Phase
- Dead-letter 처리 전략 → 구현 Phase
- SSE/WebSocket 실시간 채널 → 선택적 확장 설계
- 구독 검증(challenge) 흐름 → 구현 Phase
- event payload schema 상세 → Task 3-9 및 구현 Phase
- operation 보존 기간(TTL) → 구현 Phase
