# Task7_idempotency_strategy.md
# Idempotency 및 재시도 안전성 설계

---

## 1. 문서 목적

이 문서는 Mimir 플랫폼 API에서 중복 요청, 네트워크 재시도, 클라이언트 타임아웃, 외부 시스템 재전송 상황이 발생해도 안전하게 동작할 수 있는 **idempotency 및 retry safety 설계 기준**을 정의한다.

이 문서는 다음을 확정한다:
- 어떤 API가 중복 실행에 취약한지 식별과 분류
- Idempotency-Key 사용 원칙 및 적용 범위
- 동일 키 + 상이 payload 충돌 처리 원칙
- 생성 / 상태 전이 / bulk / async job 별 전략
- 외부 시스템 / 웹훅 / AI agent / automation 재시도 관점
- replay 응답 원칙 및 감사 추적 연계

이 문서는 Task 3-5(공통 응답 포맷)의 `meta.idempotency` 섹션, Task 3-8(async/event/webhook), Task 3-10(오류 모델)의 기준이 된다.

---

## 2. Idempotency가 필요한 이유

### 2-1. 운영 환경의 불확실성

다음은 실제 운영에서 반드시 발생하는 상황이다:

| 상황 | 결과 |
|---|---|
| 네트워크 타임아웃 | 클라이언트가 요청 성공 여부를 모름 → 재시도 |
| 사용자 버튼 중복 클릭 | 동일 요청이 짧은 간격으로 복수 발송 |
| 외부 시스템 재전송 | 동일 이벤트 또는 동기화 요청을 at-least-once로 재전송 |
| Webhook delivery 재시도 | 수신 확인 실패 시 동일 webhook payload 재발송 |
| Job trigger 재시도 | automation이 job 시작 요청을 실패 후 재시도 |
| AI agent 재호출 | tool invocation 결과 불확실 시 동일 작업 재실행 |

### 2-2. 중복 실행의 영향

중복 요청이 안전하지 않으면:
- 문서/버전 중복 생성
- 상태 전이(publish, archive)가 이미 완료된 상태에서 재실행 → 오류 또는 의도치 않은 동작
- 감사 이벤트 중복 발생 → 감사 로그 오염
- 외부 연동 중복 호출 → 외부 시스템 부작용

**idempotency는 부가 기능이 아니라 운영 환경의 불확실성을 견디기 위한 필수 안전장치다.**

---

## 3. 상위 설계 원칙

### 원칙 1. Explicit Retry Safety

**의미**: 각 API는 재시도 안전 여부를 명시적으로 계약에 표현한다.

**왜**: 암묵적으로 "아마도 안전할 것"이라는 가정은 운영 장애의 원인이 된다.

**설계 판단**: API 문서에 각 endpoint의 idempotency 속성을 명시한다.

---

### 원칙 2. Side-Effect Awareness

**의미**: HTTP method의 일반 의미만 믿지 않는다. 비즈니스 부수 효과(이벤트 발행, 감사 기록, 외부 연동)도 함께 고려한다.

**왜**: DELETE는 HTTP 스펙상 idempotent이지만, 삭제 이벤트가 외부로 발행되는 구조라면 중복 부수 효과가 발생한다.

**설계 판단**: method semantics와 side-effect semantics를 함께 평가한다.

---

### 원칙 3. Same Intent, Same Result

**의미**: 동일한 의도를 가진 재시도는 가능한 한 같은 결과로 수렴해야 한다.

**왜**: 소비자는 재시도 후 상태가 예측 가능해야 한다.

**설계 판단**: 동일 Idempotency-Key를 가진 재요청은 동일한 응답(또는 동일한 최종 상태)을 보장한다.

---

### 원칙 4. Default Caution for Non-idempotent Operations

**의미**: idempotency가 보장되지 않는 작업은 기본적으로 주의가 필요하다고 간주한다.

**왜**: 안전하다고 입증되지 않은 작업은 보수적으로 다룬다.

**설계 판단**: 생성 계열 POST는 Idempotency-Key를 기본 권장한다.

---

### 원칙 5. Client-Assisted Idempotency for Unsafe Writes

**의미**: 서버가 모든 중복을 자동 감지할 수 없으므로, 쓰기 요청에서는 클라이언트가 Idempotency-Key를 제공하는 방식으로 협력한다.

**왜**: 서버가 요청 payload만으로 중복을 판별하는 것은 비용이 크고 의미가 모호하다.

**설계 판단**: `Idempotency-Key` 헤더는 클라이언트가 생성하고 제공한다.

---

### 원칙 6. Conflict Visibility for Key Reuse Mismatch

**의미**: 같은 키에 다른 요청이 붙으면 조용히 처리하지 않고 명시적 충돌 응답을 반환한다.

**왜**: 키 재사용 불일치를 무시하면 의도하지 않은 결과가 silently 발생한다.

**설계 판단**: 동일 키 + 다른 payload → `409 Conflict` + `error.code: IDEMPOTENCY_KEY_CONFLICT`

---

### 원칙 7. Auditable Replay Behavior

**의미**: 재사용 응답인지, 최초 처리인지 추적 가능해야 한다.

**왜**: 감사 로그에서 중복 요청 패턴을 식별하거나 재시도 오남용을 탐지할 수 있어야 한다.

**설계 판단**: replay 응답에 `meta.idempotency.replayed: true` 표시. 내부 감사 로그에 replay 여부 기록.

---

### 원칙 8. Idempotency Is Not a Substitute for Authorization

**의미**: Idempotency-Key가 있다고 해서 권한 검증을 생략하지 않는다.

**왜**: 키를 탈취한 공격자가 동일 키로 재요청 시 권한 없이 응답을 얻을 수 없어야 한다.

**설계 판단**: replay 응답도 동일한 authorization 검증을 거친다. 원 요청자와 actor/tenant가 일치해야 한다.

---

### 원칙 9. Idempotency Is Not a Substitute for Concurrency Control

**의미**: idempotency는 "같은 요청의 재시도" 문제를 다룬다. 동시에 다른 사용자가 같은 리소스를 변경하는 문제는 별도의 concurrency control이 필요하다.

**왜**: 두 문제는 다른 메커니즘이 필요하다.

**설계 판단**: ETag / version check / optimistic locking은 별도 설계. idempotency와 혼용하지 않는다.

---

### 원칙 10. Minimal Surprise for Clients

**의미**: idempotency 동작이 클라이언트에게 예측 가능하고 놀랍지 않아야 한다.

**왜**: 예측 불가능한 dedupe 동작은 클라이언트 디버깅을 어렵게 한다.

**설계 판단**: replay 응답의 HTTP status는 원 응답과 동일하게 유지한다. (재시도했을 때 갑자기 200이 202로 바뀌지 않음)

---

## 4. HTTP Method 의미와 비즈니스 Idempotency의 관계

### 4-1. HTTP 스펙 차원의 Idempotent Method

| Method | HTTP 스펙 idempotent | 의미 |
|---|---|---|
| GET | 예 | 읽기 전용, 부수 효과 없음 |
| PUT | 예 | 완전 교체, 동일 요청 반복 시 같은 상태 |
| DELETE | 예 | 삭제, 이미 삭제된 리소스에 재요청 시 404 허용 |
| HEAD | 예 | GET과 동일, body 없음 |
| POST | **아니오** | 새 리소스 생성 또는 작업 실행 — 기본 non-idempotent |
| PATCH | **아니오** | 부분 수정 — 내용에 따라 다름 |

### 4-2. 비즈니스 차원의 Idempotency는 다르다

**HTTP 스펙을 신뢰하는 것만으로는 충분하지 않다.**

| 케이스 | HTTP | 비즈니스 실제 |
|---|---|---|
| DELETE /documents/{id} | idempotent | 이미 삭제됐다면 404 반환 — OK. 그러나 삭제 이벤트가 외부로 재발행될 수 있음 |
| PUT /documents/{id} | idempotent | 동일 body 반복 시 상태 같음. 그러나 audit 이벤트가 중복 기록될 수 있음 |
| POST /documents | non-idempotent | 기본적으로 중복 생성 위험 |
| PATCH /documents/{id} | non-idempotent | `title: "A"` 반복 적용은 안전. `counter: counter+1` 반복은 위험 |
| POST /documents/{id}:publish | non-idempotent (action) | 이미 published 상태라면? 상태 전이 의미와 부수 효과 함께 고려 필요 |

**결론**: method semantics와 side-effect semantics를 함께 평가한다. HTTP method가 idempotent여도 비즈니스 부수 효과(이벤트, 감사, 외부 연동)를 고려해야 한다.

---

## 5. Idempotency 적용 대상 분류

### 5-1. 조회 계열 (GET / HEAD / list / search)

| 유형 | 안전성 | Key 필요 여부 | 이유 |
|---|---|---|---|
| GET 단건/목록/검색 | **안전** | 불필요 | 읽기 전용. 부수 효과 없음. |

---

### 5-2. 생성 계열 (POST create)

| 요청 | 안전성 | Key 권장 수준 | 이유 |
|---|---|---|---|
| 문서 생성 | **주의** | **필수 권장** | 중복 생성 시 동일 내용 문서가 두 개 생김 |
| 버전 생성 | **주의** | **필수 권장** | 동일 버전 중복 생성 위험 |
| 첨부파일 등록 | **주의** | 권장 | 중복 첨부 가능 |
| Webhook 등록 | **주의** | 권장 | 동일 webhook 중복 등록 가능 |
| Job(operation) 시작 요청 | **주의** | **필수 권장** | 동일 작업 중복 실행 위험 (비용, 부작용) |
| 멤버십/권한 추가 | 주의 | 권장 | 중복 부여는 보통 무해하나 감사 오염 발생 |

---

### 5-3. 변경 계열 (PUT / PATCH)

| 요청 | 안전성 | Key 권장 수준 | 이유 |
|---|---|---|---|
| 문서 단순 필드 수정 (PATCH) | **비교적 안전** | 선택 | 같은 값으로 덮어쓰는 것은 결과 동일 |
| 증분/상대 변경 PATCH (예: counter++) | **주의** | 권장 | 반복 적용 시 값이 누적됨 |
| 권한 SET 형태 변경 | **비교적 안전** | 선택 | 동일 권한으로 덮어쓰기는 무해 |
| 권한 ADD/REMOVE 형태 변경 | **주의** | 권장 | 중복 추가/제거 시 의도 불명확 |

---

### 5-4. 삭제 계열 (DELETE)

| 요청 | 안전성 | Key 권장 수준 | 이유 |
|---|---|---|---|
| 문서/리소스 삭제 | **비교적 안전** | 선택 | 두 번 삭제 시 두 번째는 404. 단, 삭제 이벤트 중복 발행 주의 |
| 멤버십 제거 | **비교적 안전** | 선택 | 두 번째 제거 요청은 404 |

---

### 5-5. Action Endpoint (POST `:action`)

| Action | 안전성 | Key 권장 수준 | 이유 |
|---|---|---|---|
| `:publish` | **주의** | 권장 | 이미 published → 충돌 또는 no-op 설계 필요 |
| `:archive` | **주의** | 권장 | 이미 archived → 충돌 또는 no-op |
| `:restore` | **주의** | 권장 | 이미 active → no-op 또는 오류 |
| `:validate` | **비교적 안전** | 선택 | 계산 전용, 부수 효과 없으면 안전 |
| `:clone` | **주의** | **필수 권장** | 중복 클론 시 의도하지 않은 사본 다수 생성 |
| `:reindex` | **주의** | 권장 | 중복 실행 가능하나 리소스 낭비 |
| `:regenerate-preview` | 주의 | 권장 | 중복 job 시작 |
| `:resend-webhook` | **주의** | 권장 | 외부 시스템에 중복 전달 위험 |
| `:cancel` (operation) | **비교적 안전** | 선택 | 이미 취소된 상태 재취소 → no-op |

---

### 5-6. Bulk / Async Job

| 요청 | 안전성 | Key 권장 수준 | 이유 |
|---|---|---|---|
| Bulk archive/delete | **주의** | **필수 권장** | 중복 실행 시 이미 처리된 항목 재처리 위험 |
| Bulk permission update | **주의** | 권장 | 중복 적용 시 의도 불명확 케이스 존재 |
| Export job 시작 | **주의** | **필수 권장** | 동일 export job 중복 시작 → 비용 낭비 |
| Reindex job 시작 | 주의 | 권장 | 중복 실행 가능하나 비용 낭비 |
| Ingestion job 시작 | **주의** | **필수 권장** | 중복 ingestion → 데이터 중복 |

---

## 6. Idempotency-Key 사용 원칙

### 6-1. 위치

- **HTTP 요청 헤더**: `Idempotency-Key: <client-generated UUID>`
- 헤더 방식을 표준으로 채택. request body에 포함하는 방식은 비권장.
- 이유: body는 변경될 수 있고, 헤더는 미들웨어에서 일관되게 처리 가능.

```
POST /api/v1/documents
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{ "title": "새 문서", "document_type_id": "dt_001" }
```

### 6-2. 키 생성 주체

- **클라이언트**가 생성한다. 서버는 키를 부여하지 않는다.
- 형태: UUID v4 권장. 클라이언트가 보장하는 고유성.
- 키는 "요청 동일성"이 아니라 **"사용자 의도 단위"**를 나타낸다.
  - 예: "사용자가 '문서 생성' 버튼을 클릭한 1회 의도" = 하나의 키

### 6-3. 적용 수준

| 수준 | 대상 |
|---|---|
| **필수 권장** | 문서/버전 생성, clone, export/ingestion job 시작, bulk 위험 작업 |
| **권장** | 일반 생성 POST, 상태 전이 action, webhook 등록, reindex job |
| **선택** | 단순 필드 수정 PATCH, 삭제, 이미 멱등한 PUT |
| **불필요** | GET / HEAD / list / search |

### 6-4. 키의 스코프

키는 다음 범위 내에서 고유성을 가진다:
- `(actor_id, tenant_id, endpoint, idempotency_key)` 조합

즉, 같은 키라도 **다른 actor 또는 다른 tenant의 요청과 격리**된다. 보안상 스코프를 섞지 않는다.

### 6-5. Replay Window (TTL)

- 키는 영구 보관하지 않는다. **replay window** 개념으로 제한된 기간만 보관한다.
- 권장 방향: 24시간 ~ 7일 (구현 Phase에서 확정).
- replay window 만료 후 동일 키 재사용: 새 요청으로 처리 (중복 방지 효과 없음).
- window 내 키 만료 여부는 `meta.idempotency.expired: true` 로 표시 가능 (선택).

---

## 7. 동일 요청 / 상이 요청 충돌 처리 원칙

### 7-1. 같은 키 + 같은 Payload (정상 재시도)

| 시나리오 | 처리 방식 |
|---|---|
| 원 요청이 성공 처리됨 | 최초 응답 재사용. HTTP status 동일 유지. `meta.idempotency.replayed: true` 표시. |
| 원 요청이 처리 중 | `202 Accepted` 또는 원 요청 상태 응답. |
| 원 요청이 실패 처리됨 | 실패 응답 재반환. 클라이언트가 오류를 수정 후 새 키로 재시도해야 함. |

```json
// 재시도 성공 응답 예시
{
  "data": { "id": "doc_123", ... },
  "meta": {
    "request_id": "req_new456",
    "timestamp": "2026-04-02T14:00:00Z",
    "idempotency": {
      "key": "550e8400-...",
      "replayed": true,
      "original_request_id": "req_abc123",
      "processed_at": "2026-04-02T13:55:00Z"
    }
  }
}
```

### 7-2. 같은 키 + 다른 Payload (충돌)

**반드시 충돌로 처리한다. 조용히 덮어쓰거나 무시하지 않는다.**

이유:
- 소비자가 실수로 같은 키를 다른 요청에 재사용했을 가능성
- 키 충돌을 silently 처리하면 의도하지 않은 동작이 발생

처리:
- HTTP `409 Conflict`
- `error.code: IDEMPOTENCY_KEY_CONFLICT`
- 응답 본문에 원 요청 정보(처리 시각, original_request_id 등) 포함

```json
{
  "error": {
    "code": "IDEMPOTENCY_KEY_CONFLICT",
    "message": "이 Idempotency-Key는 이미 다른 요청에 사용되었습니다.",
    "type": "conflict",
    "retryable": false
  },
  "meta": {
    "request_id": "req_conflict",
    "idempotency": {
      "key": "550e8400-...",
      "original_request_id": "req_abc123",
      "processed_at": "2026-04-02T13:55:00Z"
    }
  }
}
```

### 7-3. 같은 키 + 같은 Payload + 다른 Actor/Tenant

- 키 스코프는 `(actor_id, tenant_id, endpoint, key)` 조합이므로, 다른 actor/tenant의 동일 키는 **별도 요청으로 취급**한다.
- 보안상 다른 actor의 idempotency 응답을 재사용하지 않는다.
- Cross-tenant 키 공유는 발생할 수 없음 (Security Context에서 tenant_id 분리 보장).

---

## 8. 리소스 생성에 대한 적용 전략

### 8-1. 기본 원칙

문서 생성 같은 POST create는 대표적인 idempotency 적용 대상이다.

시나리오:
1. 클라이언트가 `POST /documents` + `Idempotency-Key: key_A` 전송
2. 서버가 문서를 생성하고 응답 전송 중 네트워크 단절
3. 클라이언트가 성공 여부 모른 채 동일 요청 재전송
4. 서버가 `key_A`를 인식하고 최초 생성된 `doc_123`을 재반환

**결과**: 새 문서가 중복 생성되지 않음. 클라이언트는 `doc_123`을 받아 후속 작업 진행.

### 8-2. 자연키 vs Idempotency-Key 구분

| 개념 | 설명 |
|---|---|
| **Natural Key** | 비즈니스 의미를 가진 고유 식별자 (예: `external_system_id`) |
| **Idempotency-Key** | 클라이언트 요청 세션의 고유 의도 식별자 (UUID) |

- Idempotency-Key가 Natural Key를 대체하지 않는다.
- `create-if-not-exists` 패턴(natural key 기반 중복 방지)과 Idempotency-Key(재시도 안전성)는 별개의 메커니즘이다.

### 8-3. 생성 replay 시 동일 리소스 ID 반환

- 동일 Idempotency-Key의 재시도는 최초 생성된 리소스 ID(`doc_123`)를 재반환한다.
- 새 ID를 생성하지 않는다.
- 클라이언트는 replay 응답을 받아도 동일한 ID로 후속 작업을 진행할 수 있다.

---

## 9. 상태 전이 / Action Endpoint 적용 전략

### 9-1. 상태 전이형 Action의 idempotency 판단 기준

상태 전이 action은 두 가지 관점을 함께 고려한다:

| 관점 | 설명 |
|---|---|
| **목표 상태 도달** | 이미 목표 상태에 있으면 no-op으로 처리 가능 |
| **부수 효과 중복** | 이미 목표 상태여도 이벤트/알림/외부 연동이 재실행되면 문제 |

### 9-2. Action별 전략

| Action | 전략 |
|---|---|
| `:publish` | 이미 published → `409 Conflict` 반환 (상태 전이 불가). Idempotency-Key 있으면 replay 응답 반환. |
| `:archive` | 이미 archived → `409 Conflict`. Key 있으면 replay. |
| `:restore` | 이미 active → no-op 또는 `409`. Key 있으면 replay. |
| `:validate` | 계산 전용, 부수 효과 없음 → Key 없어도 재시도 안전. |
| `:clone` | **Key 필수 권장**. 중복 clone은 동일 원본에서 복수 사본 생성. 재시도 시 동일 clone 결과 반환. |
| `:reindex` | 중복 실행 가능하나 비용 낭비. Key 있으면 동일 operation 반환. |
| `:resend-webhook` | **Key 권장**. 중복 재전송은 외부 시스템 영향. 같은 delivery를 재전송. |
| `:cancel` | 이미 취소된 operation → no-op 가능. |

### 9-3. "At-most-once Intent" 필요 케이스

일부 action은 **strict idempotent**보다 "at-most-once intent"가 더 적합하다.

- 예: `:publish` — 한 번만 실행되어야 한다는 의미. Idempotency-Key로 중복 보호.
- 예: `:resend-webhook` — 외부 시스템에 재전송되므로 한 번만 발신.

이 케이스들은 Idempotency-Key 없이 재시도하면 중복 실행이 가능하다. **Key 필수 권장 대상**이다.

---

## 10. Bulk 작업 및 비동기 Job 시작 적용 전략

### 10-1. Bulk 작업의 복잡성

Bulk 작업은 일반 단건 요청보다 idempotency 처리가 복잡하다:
- 부분 성공/실패가 섞일 수 있음
- 일부 항목만 처리된 상태에서 재시도 시 이미 처리된 항목과 미처리 항목 혼재

### 10-2. Bulk 작업 Idempotency 전략

| 방식 | 설명 |
|---|---|
| **전체 intent 단위 Key** | bulk 요청 전체를 하나의 Key로 묶어 처리. 재시도 시 동일 bulk 결과 반환. |
| **항목별 Key** | 각 항목에 별도 Key. 구현 복잡도 높음. |

**권장**: 전체 intent 단위 Key 방식.
- `Idempotency-Key` 하나로 bulk 전체를 식별.
- 재시도 시 최초 처리 결과(부분 성공 포함)를 그대로 반환.
- 새로 처리하지 않는다.

### 10-3. 비동기 Job 시작 전략

| 원칙 | 내용 |
|---|---|
| 중복 job 생성 방지 | Key 있는 재시도 시 최초 생성된 `operation_id`를 재반환 |
| 동일 operation_id | 클라이언트가 동일 ID로 상태를 폴링 가능 |
| Job 중복 실행 방지 | replay 처리이므로 job이 다시 실행되지 않음 |

```
// 최초 요청
POST /api/v1/documents/doc_123:reindex
Idempotency-Key: key_B
→ 202 Accepted, operation_id: op_xyz

// 재시도 (동일 Key)
POST /api/v1/documents/doc_123:reindex
Idempotency-Key: key_B
→ 202 Accepted (replay), operation_id: op_xyz (동일)
   meta.idempotency.replayed: true
```

---

## 11. 외부 시스템 / 웹훅 / Automation / AI Agent 관점

### 11-1. 외부 시스템

- 외부 시스템은 네트워크 불안정 때문에 재시도가 흔하다.
- 문서 생성/동기화/상태 업데이트 요청 모두 Idempotency-Key를 제공하도록 권장.
- 외부 시스템 연동 문서에 idempotency 사용 가이드를 포함한다.

### 11-2. Webhook 수신 (외부 → 플랫폼)

- 동일 이벤트가 재전송될 수 있다 (at-least-once delivery).
- **Webhook 수신 idempotency 원칙**:
  - 외부 webhook에 포함된 `event_id` 또는 `delivery_id`를 idempotency key로 활용.
  - 동일 `event_id` 재수신 시 처리 결과 재반환. 재처리하지 않음.
  - 구현 방식은 구현 Phase에서 결정 (Idempotency-Key 헤더 방식과 event_id 기반 방식 중 선택).

### 11-3. Automation / Workflow Agent

- 자동화 시스템은 실패 시 자동 재시도 전략을 사용한다.
- 사람이 아닌 소비자는 더 엄격한 retry safety 요구.
- **권고**: 자동화 시스템은 모든 비안전 요청에 Idempotency-Key를 필수로 포함한다.
- workflow engine이 키를 자동 생성/관리하도록 권장.

### 11-4. AI / Agent Tool Invocation

- AI agent는 tool invocation 결과가 불확실할 때 동일 작업을 재호출할 수 있다.
- **idempotent-friendly API는 AI integration을 안전하게 만든다.**
- AI 소비자를 위한 원칙:
  - 생성/변경 요청 시 Idempotency-Key 포함.
  - replay 응답의 `meta.idempotency.replayed: true`를 통해 중복 처리 여부 확인.
  - AI agent는 replayed 응답을 원본 처리 결과와 동일하게 취급한다.

---

## 12. 응답 재사용 및 Replay 응답 원칙

### 12-1. Replay 응답 기본 원칙

| 항목 | 규칙 |
|---|---|
| HTTP status | 원 응답과 동일하게 유지 |
| response body (`data`) | 원 응답의 `data` 재사용 |
| `meta.request_id` | 새 요청의 request_id 사용 (추적 가능성) |
| `meta.idempotency.replayed` | `true` |
| `meta.idempotency.original_request_id` | 최초 요청의 request_id |
| `meta.idempotency.processed_at` | 최초 처리 시각 |

### 12-2. Replay 응답에서 header 표시

```
HTTP/1.1 201 Created
Idempotency-Key: 550e8400-...
X-Idempotency-Replayed: true
```

### 12-3. 응답이 너무 오래된 경우

- replay window 내에서는 항상 저장된 응답을 반환.
- replay window 만료 후: 새 요청으로 처리 (중복 방지 효과 없음).
- 만료 여부를 클라이언트에 알릴 필요는 없으나, 선택적으로 `meta.idempotency.key_expired: true` 표시 가능.

### 12-4. 원 요청이 실패한 경우의 재시도

- 원 요청이 `400 Bad Request`로 실패한 경우: 동일 실패 응답 재반환.
- 클라이언트가 payload를 수정하고 싶다면 **새 Idempotency-Key로 재시도**해야 한다.
- 실패 응답도 replay 가능하다는 점을 문서화.

---

## 13. 감사 추적 / Request Context / Observability 연계

### 13-1. 감사 로그에서 idempotency 추적

감사 로그에 기록해야 할 idempotency 관련 정보:

| 필드 | 설명 |
|---|---|
| `idempotency_key` | 요청에 사용된 키 |
| `is_replayed` | replay 처리 여부 |
| `original_request_id` | 최초 처리 요청 ID (replay 시) |
| `actor_id` | 요청 주체 |
| `tenant_id` | 조직 스코프 |

### 13-2. Request Security Context 연계

Idempotency-Key는 Request Security Context의 일부로 전달된다:
```
SecurityContext {
  actor_id: "user_123",
  tenant_id: "org_001",
  idempotency_key: "550e8400-...",
  ...
}
```

### 13-3. 중복 요청 오남용 탐지

- 동일 키로 다른 payload를 반복 전송하는 패턴 → 보안 이상 감지 대상
- 단기간에 동일 actor가 수백 개의 키를 발급하는 패턴 → rate limit 또는 알람 대상
- 내부 보안 로그에 기록. 외부 응답에는 노출하지 않음.

---

## 14. Concurrency Control과의 관계

### 14-1. 두 문제는 다르다

| 구분 | 문제 | 해결 수단 |
|---|---|---|
| **Idempotency** | 같은 클라이언트가 같은 요청을 여러 번 보내는 문제 | Idempotency-Key |
| **Concurrency Control** | 다른 클라이언트가 같은 리소스를 동시에 변경하는 문제 | ETag, version check, optimistic locking |

### 14-2. 예시로 구분

```
Idempotency 문제:
  사용자 A가 "문서 생성" 버튼을 두 번 클릭 → 같은 요청 두 번 전송
  → Idempotency-Key로 중복 방지

Concurrency Control 문제:
  사용자 A와 B가 동시에 같은 문서를 수정 → 나중에 저장된 것이 앞의 것을 덮어씀
  → ETag 또는 version check로 충돌 감지
```

### 14-3. 함께 사용 가능

두 메커니즘은 독립적이며 함께 필요할 수 있다:
- 같은 요청 재시도 방지: Idempotency-Key
- 동시 편집 충돌 방지: ETag / version 필드

둘을 혼용하거나 한쪽으로 대체하지 않는다.

### 14-4. Concurrency Control 설계

Optimistic locking / ETag / version check 상세 설계는 이 문서의 범위가 아니다. 별도 설계 문서에서 다룬다.

---

## 15. 후속 Task 및 구현 Phase에 전달할 기준

| 대상 | 전달 기준 |
|---|---|
| Task 3-8 (async/event/webhook) | Webhook 수신 event_id 기반 dedup 원칙. Job 중복 시작 방지를 위한 Idempotency-Key + operation_id 재사용 전략 구체화. |
| Task 3-9 (AI/RAG) | AI tool invocation용 retry-safe 인터페이스 설계 시 Idempotency-Key 포함 권장. replayed 응답의 의미를 AI 소비자가 이해할 수 있도록 문서화. |
| Task 3-10 (오류 모델) | `IDEMPOTENCY_KEY_CONFLICT` 오류 코드 확정. replay window 만료 처리 오류 코드. 원 요청 실패 replay 시 오류 응답 구조. |
| 구현 Phase | Idempotency-Key 저장소(TTL 기반 캐시/DB). request middleware에서 Key 추출/검증/저장 흐름. response serializer에 `meta.idempotency` 자동 주입. 감사 로그에 `idempotency_key` + `is_replayed` 기록. |
| 별도 설계 | Concurrency control(ETag/version check/optimistic locking) 상세 설계. |

---

## 16. 결론

Mimir 플랫폼 API의 idempotency 전략은 다음으로 요약된다:

1. **클라이언트 제공 Idempotency-Key**: 헤더 방식, UUID. 생성/action/bulk/job 시작에 권장/필수.
2. **스코프**: `(actor_id, tenant_id, endpoint, key)` 조합. 다른 actor/tenant와 격리.
3. **Replay Window**: 제한된 기간(TTL) 내에서만 dedup 보장.
4. **충돌 처리**: 같은 키 + 다른 payload → `409 IDEMPOTENCY_KEY_CONFLICT`. 조용한 덮어쓰기 금지.
5. **Replay 표시**: `meta.idempotency.replayed: true`. HTTP status 원본 유지.
6. **감사 연계**: `idempotency_key`, `is_replayed`를 감사 로그에 기록.
7. **Concurrency Control 분리**: idempotency와 별개. 동시 편집 충돌은 별도 메커니즘.

---

## 자체 점검 요약

### Idempotency 적용이 중요한 요청 유형 요약

필수 권장: 문서/버전 생성, clone, export/ingestion job 시작, bulk 위험 작업
권장: 일반 생성 POST, 상태 전이 action(:publish/:archive/:clone), webhook 등록, reindex
선택: 단순 필드 PATCH, DELETE
불필요: GET/HEAD/list/search

### Idempotency-Key 사용 원칙 요약

- 위치: HTTP 헤더 `Idempotency-Key`
- 생성 주체: 클라이언트 (UUID v4)
- 스코프: `(actor_id, tenant_id, endpoint, key)` 조합
- Replay Window: TTL 기반 제한 (24시간~7일, 구현 Phase 확정)
- 키 의미: "사용자 의도 단위" (요청 동일성이 아님)

### Same Key / Different Payload 충돌 처리 요약

`409 Conflict` + `error.code: IDEMPOTENCY_KEY_CONFLICT`. 조용한 처리 금지. 응답에 원 요청 정보 포함.

### Create / Action / Bulk / Async Job 적용 관점 요약

- Create: 재시도 시 동일 resource_id 반환
- Action: 목표 상태 + 부수 효과 함께 고려. at-most-once intent 개념 적용
- Bulk: 전체 intent 단위 Key. 부분 성공 결과도 재반환
- Async Job: 동일 operation_id 재반환. job 중복 실행 방지

### 외부 시스템 / Webhook / AI Agent 관련 요약

- 외부 시스템: 모든 비안전 요청에 Key 포함 권장
- Webhook 수신: event_id 기반 dedup 원칙
- Automation: Key 필수 포함 권고
- AI Agent: replayed 응답 구분 가능. idempotent-friendly API가 안전한 AI integration 기반

### Concurrency Control과의 차이 요약

- Idempotency: 같은 요청 재시도 안전성 → Idempotency-Key
- Concurrency Control: 동시 다른 변경 충돌 → ETag/version check (별도 설계)
- 둘은 독립적. 혼용/대체 금지

### 후속 Task 연결 포인트

Task 3-8(webhook event_id dedup, job operation_id 재사용), 3-9(AI tool retry-safe), 3-10(충돌/replay 오류 코드)

### 의도적으로 후속 Task로 남긴 미결정 사항

- Replay Window TTL 정확한 수치 → 구현 Phase
- Idempotency-Key 저장소 구현 방식(Redis vs DB) → 구현 Phase
- Webhook event_id 기반 dedup 상세 → Task 3-8
- Bulk 부분 성공 replay 응답 상세 구조 → Task 3-10
- Concurrency control 상세 설계 → 별도 설계 문서
- Idempotency 오류 코드 카탈로그 → Task 3-10
