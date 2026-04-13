# Task6_list_query_conventions.md
# Pagination / Filtering / Sorting 규약 설계

---

## 1. 문서 목적

이 문서는 Mimir 플랫폼 API의 모든 collection endpoint에서 공통으로 사용할 **목록 탐색 규약(list traversal contract)**을 정의한다.

단순히 `page=1` 쿼리 파라미터를 정하는 것이 아니다. 이 문서는 다음을 확정한다:
- 문서, 버전, 노드, 감사 로그, 이벤트, 검색 결과, 웹훅 이력 등 모든 목록에 적용되는 일관된 pagination 방식
- 공통 filtering 문법과 필드 명명 규칙
- sorting 파라미터 규약과 다중 정렬 원칙
- 권한 기반 가시성 필터링의 의미 해석
- 응답 `meta.pagination` 구조와의 연결

이 문서는 Task 3-5(공통 응답 포맷)의 `meta.pagination` 섹션을 구체화하며, Task 3-8(async/event/webhook), Task 3-9(AI/RAG), Task 3-10(오류 모델)의 목록 조회 기준이 된다.

---

## 2. 목록 조회 규약이 필요한 이유

### 2-1. 목록 조회는 플랫폼 전반의 공통 패턴이다

Mimir에서 목록 조회가 필요한 리소스:
- documents, versions, nodes
- audit-logs, activities
- events, webhook-deliveries
- operations (jobs)
- searches, search results
- members, roles, permissions

이 모든 목록에서 pagination/filter/sort가 사용된다. 엔드포인트마다 다른 문법을 쓰면 UI, SDK, 외부 연동, AI agent 모두가 개별적으로 대응해야 한다.

### 2-2. 성격이 다른 목록이 공존한다

| 목록 유형 | 특성 |
|---|---|
| 문서 목록 | 중간 규모, 관리 친화적, 상태/타입 기반 필터 |
| 감사 로그 | 대용량, 시간순 append-only, 집계 불필요 |
| 이벤트 스트림 | 대용량, 실시간 갱신, cursor 필수 |
| 검색 결과 | relevance sort, 관련도 메타 포함 |
| 웹훅 전달 이력 | 중간 규모, 상태/시간 기반 |

이처럼 성격이 다른 목록이 공존하므로, **단일 규약 안에서 리소스 특성을 수용하는 유연한 설계**가 필요하다.

### 2-3. 규약은 탐색성과 운영성의 기반이다

공통 규약이 있으면:
- AI agent가 다음 페이지 토큰을 동일 방식으로 처리
- 외부 시스템이 pagination 로직을 한 번만 구현
- 테스트 코드가 모든 collection endpoint에 재사용 가능
- 잘못된 파라미터에 대한 오류 처리가 일관됨

**목록 조회 규약은 단순 편의 기능이 아니라 플랫폼 탐색성과 운영성을 좌우하는 핵심 표준이다.**

---

## 3. 상위 설계 원칙

### 원칙 1. Consistent Query Language Across Collections

**의미**: 모든 collection endpoint는 동일한 파라미터 이름과 의미를 사용한다.

**왜**: `page` vs `offset` vs `cursor`가 리소스마다 달라지면 소비자가 각각 학습해야 한다.

**설계 영향**: `page`, `page_size`, `sort`, `cursor`, `limit` 파라미터 이름을 플랫폼 전체에서 통일.

---

### 원칙 2. Stable Traversal Semantics

**의미**: 페이지를 넘길 때 결과 항목이 갑자기 중복되거나 누락되지 않아야 한다.

**왜**: 데이터가 실시간으로 추가/삭제되는 환경에서 offset 기반은 결과 불안정 문제가 있다.

**설계 영향**: 대용량 시간순 데이터는 cursor 기반 적용. offset은 안정성이 덜 요구되는 관리 목록에 사용.

---

### 원칙 3. Predictable Server Behavior

**의미**: 동일한 쿼리에 대해 서버 응답이 예측 가능해야 한다.

**왜**: 소비자가 캐싱, 재시도, 결과 비교를 할 수 있어야 한다.

**설계 영향**: 기본 정렬을 항상 명시. 동일 쿼리에서 결과 순서가 변하지 않는 tie-breaker 필드 지정.

---

### 원칙 4. Scalability-Aware Pagination

**의미**: 대용량 데이터에서도 성능을 유지할 수 있는 방식을 선택한다.

**왜**: 수백만 건의 감사 로그를 offset으로 페이징하면 DB 성능이 급격히 저하된다.

**설계 영향**: 운영성 리소스(audit-logs, events)는 cursor 기반. page_size 상한 설정.

---

### 원칙 5. Explicit Filtering and Sorting

**의미**: 필터와 정렬 조건을 명시적 파라미터로 표현. 서버의 암묵적 필터링/정렬 최소화.

**왜**: 암묵적 동작은 소비자가 예측하기 어렵고 버전 변경 시 breaking이 발생하기 쉽다.

**설계 영향**: 기본 정렬을 문서화하고, 서버가 권한 외 이유로 결과를 무음으로 필터링하면 `meta.warnings`에 표시.

---

### 원칙 6. Authorization-Safe Listing

**의미**: 권한 때문에 보이지 않는 항목이 있어도 응답 구조는 일관되고, 이를 소비자에게 드러내지 않는다.

**왜**: 존재 자체를 숨겨야 하는 항목의 개수를 알려주면 보안 정보 누출이 될 수 있다.

**설계 영향**: authorization-aware filtering은 쿼리 파라미터와 분리된 별도 계층. total count의 의미를 명확히 정의.

---

### 원칙 7. Minimal Ambiguity

**의미**: 파라미터 이름과 동작의 의미가 모호하지 않아야 한다.

**왜**: 모호한 파라미터는 소비자마다 다르게 해석된다.

**설계 영향**: `created_after` vs `created_from` 같은 동의어를 하나로 표준화.

---

### 원칙 8. Extensible Query Model

**의미**: 현재는 단순 필터만 있어도 나중에 상태/날짜/메타데이터 필터를 추가할 수 있어야 한다.

**왜**: 초기에 과도한 DSL을 도입하면 복잡성만 증가한다. 단, 구조가 확장을 막아서도 안 된다.

**설계 영향**: 단순 key=value 필터로 시작. 복잡한 쿼리 DSL은 후속 단계에서 필요 시 추가.

---

### 원칙 9. Metadata-Backed Navigation

**의미**: 소비자는 응답의 `meta.pagination` 필드만 보고 다음 페이지 탐색 방법을 알 수 있어야 한다.

**왜**: `total`을 별도 API로 조회하거나 URL을 직접 계산해야 하면 소비자 구현이 복잡해진다.

**설계 영향**: `has_next`, `next_cursor` 또는 `has_next` + `page` 필드를 응답에 항상 포함.

---

## 4. Pagination 방식 후보 비교

### 4-1. 후보 1: Offset Pagination

형태: `?page=2&page_size=20` 또는 `?offset=40&limit=20`

| 항목 | 평가 |
|---|---|
| 구현 단순성 | 높음. SQL `LIMIT`/`OFFSET`으로 직접 연결. |
| 대용량 데이터 적합성 | 낮음. 깊은 offset에서 DB full scan 발생 가능. |
| 정렬 안정성 | 낮음. 항목 추가/삭제 시 페이지 경계에서 중복/누락 발생 가능. |
| 실시간 변경 내성 | 낮음. 데이터 변경 중 페이징 시 결과 불안정. |
| UI 사용성 | 높음. "3페이지로 이동" 같은 UX 구현 용이. |
| 외부 시스템 사용성 | 높음. 직관적. |
| 검색 결과 적합성 | 중간. 검색 엔진은 일반적으로 offset 지원. |
| 이 플랫폼 적합성 | **관리 목록(문서, 버전, 역할)에 적합** |

---

### 4-2. 후보 2: Cursor Pagination

형태: `?cursor=eyJpZCI6MTIzfQ&limit=20`

| 항목 | 평가 |
|---|---|
| 구현 단순성 | 중간. cursor 생성/파싱 로직 필요. |
| 대용량 데이터 적합성 | 높음. index seek 기반으로 깊은 페이지도 성능 일정. |
| 정렬 안정성 | 높음. cursor가 특정 위치를 가리키므로 중복/누락 방지. |
| 실시간 변경 내성 | 높음. 데이터가 추가되어도 cursor 이후 결과는 안정적. |
| UI 사용성 | 중간. "이전/다음"만 가능. 특정 페이지 번호로 이동 불가. |
| 외부 시스템 사용성 | 높음. 자동화 스크립트가 cursor 기반 순회에 적합. |
| 검색 결과 적합성 | 중간. 검색 엔진 특성에 따라 다름. |
| 이 플랫폼 적합성 | **대용량/시간순 로그/이벤트에 적합** |

---

### 4-3. 후보 3: 혼합 전략

- 관리 목록 → offset
- 대용량/스트림 목록 → cursor

| 항목 | 평가 |
|---|---|
| 구현 복잡성 | 높음. 두 방식 모두 구현 필요. |
| 소비자 부담 | 리소스마다 다른 방식을 알아야 함. |
| 이 플랫폼 적합성 | **리소스 특성이 명확히 다르므로 필요** |

---

### 4-4. 결정: 혼합 전략 (Offset 기본, Cursor 선택 적용)

**기본**: Offset pagination (`page` + `page_size`)

**이유**: 초기 구현 단순성, 관리 UI 친화성, 대부분의 문서 관련 목록에서 데이터 규모가 관리 가능.

**Cursor 적용**: 다음 리소스에는 cursor pagination 적용

| 리소스 | 이유 |
|---|---|
| `audit-logs` | 대용량, append-only, 시간순 탐색 |
| `events` | 실시간 증가, 스트림 특성 |
| `webhook-deliveries` | 대용량 가능, 시간순 조회 위주 |
| `operations` (결과 목록) | 장기 운영 데이터 |

**파라미터 혼용 방지**: 동일 endpoint에서 offset + cursor를 동시에 허용하지 않는다. 리소스별로 지원 방식을 명시하고 다른 방식 요청 시 `400 Bad Request`.

---

## 5. 권장 Pagination 전략

### 5-1. Offset 기반 (기본)

- 파라미터: `page` (1-based), `page_size`
- 기본값: `page=1`, `page_size=20`
- 최대 허용: `page_size <= 100`
- `page_size=0` 또는 음수: `400 Bad Request`
- `page_size > 100`: `400 Bad Request` (또는 100으로 clamp — 비권장)

**권장 리소스**: documents, versions, nodes, roles, members, permissions, searches

### 5-2. Cursor 기반

- 파라미터: `cursor` (불투명 토큰), `limit`
- 기본값: `limit=20`
- 최대 허용: `limit <= 100`
- 만료/무효 cursor: `400 Bad Request` + `error.code: INVALID_CURSOR`
- 첫 페이지: `cursor` 생략
- cursor는 불투명(opaque)하게 취급 — 소비자가 내부 구조를 파싱하거나 생성해서는 안 됨

**권장 리소스**: audit-logs, events, webhook-deliveries, operations

### 5-3. 페이지네이션 미지정 시 기본 동작

- `page`/`cursor` 미지정: 첫 페이지(`page=1`) 반환
- `page_size`/`limit` 미지정: 기본값(20) 적용
- 파라미터 없는 목록 요청은 항상 첫 페이지를 반환 (전체 결과 반환 금지)

### 5-4. 비권장 패턴

| 패턴 | 이유 |
|---|---|
| `page_size=0` | 의미 불명확. 금지. |
| `page_size=-1` (전체 조회) | 대용량 데이터에서 위험. 금지. |
| offset + cursor 혼용 | 동일 endpoint에서 두 방식 동시 사용 금지 |
| cursor 내부 구조 파싱 | cursor는 불투명 토큰. 소비자가 파싱 시도 비권장. |
| 매우 깊은 offset | `page > 500` 등 성능 경고 또는 제한 (구현 Phase에서 결정) |

---

## 6. Pagination Query Parameter 규약

### 6-1. Offset 기반 파라미터

| 파라미터 | 타입 | 기본값 | 최대값 | 설명 |
|---|---|---|---|---|
| `page` | integer (1-based) | 1 | — | 조회할 페이지 번호 |
| `page_size` | integer | 20 | 100 | 페이지당 항목 수 |

**`page` vs `offset` 선택**: `page` + `page_size` 채택. `offset` + `limit` 방식은 비권장.
- 이유: `page` 방식이 UI 친화적이고 직관적. `offset`은 계산이 필요하여 실수 유발 가능.

### 6-2. Cursor 기반 파라미터

| 파라미터 | 타입 | 기본값 | 최대값 | 설명 |
|---|---|---|---|---|
| `cursor` | string (opaque) | — | — | 이전 응답의 `next_cursor` 값 |
| `limit` | integer | 20 | 100 | 반환할 항목 수 |

### 6-3. 응답 meta.pagination 연결

**Offset 기반 응답**:
```json
"pagination": {
  "page": 2,
  "page_size": 20,
  "total": 87,
  "has_next": true,
  "has_prev": true
}
```

**Cursor 기반 응답**:
```json
"pagination": {
  "limit": 20,
  "has_next": true,
  "has_prev": false,
  "next_cursor": "eyJpZCI6NTAsInRzIjoiMjAyNi0wNC0wMVQxMjowMDowMFoifQ",
  "prev_cursor": null
}
```

---

## 7. Filtering 문법 규약

### 7-1. 기본 단일 값 필터

형태: `?{field}={value}`

```
GET /api/v1/documents?status=published
GET /api/v1/documents?document_type_id=dt_001
GET /api/v1/audit-logs?actor_type=user
```

- 파라미터 이름은 리소스 필드명과 일치시킨다 (일관성 원칙)
- 대소문자: 파라미터명은 snake_case

### 7-2. 다중 값 필터

형태: `?{field}={v1},{v2},{v3}` (comma-separated)

```
GET /api/v1/documents?status=published,draft
GET /api/v1/documents?document_type_id=dt_001,dt_002
```

- OR 의미: 지정된 값 중 하나에 해당하는 항목 반환
- 반복 파라미터 방식 (`?status=published&status=draft`) 허용하나 **비권장** — comma-separated 방식을 표준으로 채택

### 7-3. 범위 필터

**날짜 범위**: `{field}_after` / `{field}_before` suffix 방식

```
GET /api/v1/documents?created_after=2026-01-01
GET /api/v1/documents?created_before=2026-03-31
GET /api/v1/audit-logs?occurred_after=2026-04-01T00:00:00Z
```

- 날짜 포맷: ISO 8601 (`YYYY-MM-DD` 또는 `YYYY-MM-DDTHH:MM:SSZ`)
- `_after`는 inclusive 또는 exclusive — 문서에 명시. **권장: exclusive** (`after` = 해당 시각 이후)
- `_before`는 exclusive

**숫자 범위**: `{field}_min` / `{field}_max` suffix 방식

```
GET /api/v1/documents?version_count_min=3
```

**비채택**: `?filter[created_at][gte]=...` 같은 bracket 문법 — 과도한 복잡성, 초기 단계 비적합.

### 7-4. Boolean 필터

형태: `?{field}=true` / `?{field}=false`

```
GET /api/v1/documents?has_attachments=true
GET /api/v1/documents?is_archived=false
```

### 7-5. 텍스트 검색 파라미터

형태: `?q={text}`

```
GET /api/v1/documents?q=개인정보
```

- `q`는 자유 텍스트 검색 파라미터 (키워드 기반)
- 일반 필터 파라미터(`status`, `type` 등)와 조합 가능
- 구체적 검색 동작(full-text, 부분 일치, 형태소 분석 등)은 리소스별로 다를 수 있음 — 문서화 필요
- 고급 시맨틱 검색은 `POST /api/v1/searches` 리소스를 통해 처리 (Task 3-9)

### 7-6. Metadata 필터

메타데이터(JSON 확장 필드) 기반 필터는 초기 단계에서는 지원하지 않는다.
필요 시 구현 Phase에서 `meta.{key}={value}` 방식 또는 별도 필터 문법으로 추가. (Extensible query model 원칙)

---

## 8. Filtering 필드 명명 규칙

### 8-1. 표준 필드명

| 필터 대상 | 파라미터명 | 예시 |
|---|---|---|
| 상태 | `status` | `?status=published` |
| 문서 타입 | `document_type_id` | `?document_type_id=dt_001` |
| 작성자 | `created_by` | `?created_by=user_123` |
| 최종 수정자 | `updated_by` | `?updated_by=user_456` |
| 조직 | `organization_id` | `?organization_id=org_001` |
| 생성일 이후 | `created_after` | `?created_after=2026-01-01` |
| 생성일 이전 | `created_before` | `?created_before=2026-03-31` |
| 수정일 이후 | `updated_after` | `?updated_after=2026-04-01` |
| 수정일 이전 | `updated_before` | `?updated_before=2026-04-02` |
| 첨부파일 여부 | `has_attachments` | `?has_attachments=true` |
| 가시성 | `visibility` | `?visibility=public` |
| 태그 | `tag` | `?tag=security,compliance` |
| 자유 검색 | `q` | `?q=개인정보` |

### 8-2. 명명 충돌 원칙

- 동일 의미의 필터명은 리소스 전체에서 통일한다.
- `created_at_from` / `created_from` / `created_after` 중 `created_after`로 표준화.
- 리소스 특화 필터는 리소스 도메인 필드명을 따르되, 범용 필터(날짜, 상태 등)는 공통 명명 규칙 우선.

### 8-3. 비권장 필터명 예시

| 비권장 | 권장 | 이유 |
|---|---|---|
| `from_date` | `created_after` | 모호. 어떤 날짜인지 불명확 |
| `begin` / `end` | `created_after` / `created_before` | 비표준 |
| `filter[status]` | `status` | Bracket 문법 비채택 |
| `createdBy` | `created_by` | camelCase 금지 (쿼리 파라미터도 snake_case) |

---

## 9. Sorting 문법 규약

### 9-1. 기본 sort 파라미터

```
?sort={field}          → 오름차순 (ascending)
?sort=-{field}         → 내림차순 (descending, minus prefix)
?sort={f1},-{f2}       → 다중 정렬 (comma-separated)
```

**예시**:
```
?sort=created_at           → 생성일 오름차순
?sort=-created_at          → 생성일 내림차순 (최신순)
?sort=status,-updated_at   → 상태 오름차순, 그 다음 수정일 내림차순
```

### 9-2. 다중 정렬

- 쉼표로 구분한 다중 정렬 허용
- 왼쪽에서 오른쪽 순서로 우선순위 적용
- 최대 지원 정렬 필드 수: **3개** (구현 Phase에서 조정 가능)

### 9-3. 기본 정렬

- 명시하지 않을 경우 서버가 기본 정렬을 적용
- **원칙**: 기본 정렬은 항상 문서화된다. 암묵적 기본 정렬 금지.
- 권장 기본 정렬: `sort=-updated_at` (최근 수정 순, 대부분의 목록에서 직관적)
- 리소스별 기본 정렬이 다른 경우 해당 리소스 API 문서에 명시

### 9-4. 안정 정렬 (Stable Sort) 보장

- 동일한 sort field 값을 가진 항목의 순서가 페이지마다 달라지면 안 된다.
- **Tie-breaker 원칙**: 모든 sort에는 암묵적으로 `id` 또는 `created_at` + `id` tie-breaker를 추가한다.
- cursor pagination에서 특히 중요 — cursor가 가리키는 위치가 명확해야 함.

### 9-5. Sort와 Cursor Pagination의 제약

- cursor 방식 리소스에서는 **서버가 지원하는 정렬 필드만** 허용
- 예: audit-logs는 `sort=-occurred_at` (시간 내림차순)만 기본 허용
- 지원하지 않는 정렬 필드 요청: `400 Bad Request` + `error.code: UNSUPPORTED_SORT_FIELD`

### 9-6. 지원하지 않는 정렬 필드 처리

| 케이스 | 처리 |
|---|---|
| 존재하지 않는 필드 | `400 Bad Request` |
| 정렬 불가 필드 (예: JSON blob) | `400 Bad Request` |
| cursor 방식에서 지원 안 되는 필드 | `400 Bad Request` |

---

## 10. Pagination / Filtering / Sorting 조합 원칙

### 10-1. 기본 실행 순서

```
1. Authorization-aware filtering (보안 레이어, 소비자 지정 아님)
   ↓
2. User-specified filtering (쿼리 파라미터 필터 적용)
   ↓
3. User-specified sorting
   ↓
4. Pagination (count 계산 후 페이지 추출)
```

### 10-2. Cursor Pagination과 Sorting 조합 제약

- cursor는 **특정 sort 조건 하의 위치**를 나타낸다.
- filter 또는 sort가 변경되면 기존 cursor는 **무효화**된다.
- 무효화된 cursor 사용 시: `400 Bad Request` + `error.code: INVALID_CURSOR`
- 소비자는 filter/sort가 바뀌면 cursor를 재사용하지 말고 첫 페이지부터 다시 시작해야 한다.

### 10-3. Total Count의 의미와 한계

| 상황 | total 의미 |
|---|---|
| 일반 offset 목록 | 현재 filter 조건 + authorization filter 적용 후 총 건수 |
| cursor 기반 목록 | total 미제공 권장 (`has_next` + `limit`으로 충분) |
| 대용량 목록 | `estimated_total` 제공 가능 (정확한 count 대신 추정치) |

- cursor 기반 응답에서는 `total`을 기본 제공하지 않는다. (DB count 쿼리 비용 최소화)
- offset 기반에서도 `total`이 성능상 문제가 되는 경우, `estimated_total`을 제공할 수 있음 (구현 Phase에서 결정).

### 10-4. 허용하지 않는 조합

| 케이스 | 처리 |
|---|---|
| cursor + page 동시 사용 | `400 Bad Request` |
| cursor 방식 리소스에 지원 안 되는 sort | `400 Bad Request` |
| page_size > 100 | `400 Bad Request` |
| 잘못된 날짜 포맷 | `400 Bad Request` |
| 존재하지 않는 filter 필드 | `400 Bad Request` (엄격) 또는 무시 (관대) — **권장: 엄격** |

---

## 11. 검색 결과와 일반 목록 조회의 관계

### 11-1. 두 가지 검색 경로

| 경로 | 사용 케이스 | 비고 |
|---|---|---|
| `GET /api/v1/documents?q=키워드` | 단순 키워드 필터 | 일반 목록 조회에 `q` 파라미터 추가 |
| `POST /api/v1/searches` | 구조화된 검색, 시맨틱 검색, 검색 히스토리 | 별도 리소스, 검색 세션 개념 |

### 11-2. 공통 Pagination 규약 공유

- `POST /api/v1/searches` 결과도 동일한 `data` 배열 + `meta.pagination` 구조를 따른다.
- pagination 파라미터(`page`, `page_size` 또는 `cursor`, `limit`)도 동일하게 사용.

### 11-3. Relevance Sort와 일반 정렬의 차이

- 검색 결과는 `sort=relevance` 또는 sort 미지정 시 관련도 내림차순 적용 가능.
- `sort=relevance`는 일반 목록에서는 허용하지 않고 search 결과에서만 허용.
- `meta.pagination`에 검색 관련 추가 정보(query, total_score 등)를 선택적으로 포함.

### 11-4. 검색 결과 특화 metadata

일반 목록과 달리 검색 응답 `meta`에 추가될 수 있는 필드:
```json
"search": {
  "query": "개인정보",
  "total": 12,
  "took_ms": 45
}
```

세부 구조는 Task 3-9에서 결정.

---

## 12. 권한/가시성 필터링 관점

### 12-1. Authorization-Aware Filtering은 쿼리 파라미터가 아니다

- 소비자가 `?visibility=mine` 같은 방식으로 권한 필터를 직접 지정하는 것은 허용하되,
- **근본적인 권한 기반 필터링은 서버가 자동 적용**하며 소비자 지정 파라미터와는 별개다.

| 구분 | 설명 |
|---|---|
| Authorization filtering | 보안 레이어. 소비자가 볼 수 없는 항목은 쿼리 결과에서 자동 제외. |
| User-specified filtering | `?status=published` 같은 소비자 명시 조건 |

### 12-2. Total Count의 의미

- `meta.pagination.total`은 **현재 소비자가 볼 수 있는 항목 중 filter 조건에 맞는 건수**다.
- 전체 시스템의 총 건수(숨겨진 항목 포함)를 알려주지 않는다.
- 이는 보안 정보 보호 원칙이며, 소비자에게 혼란을 줄 수 있으므로 API 문서에 명시한다.

### 12-3. 숨겨진 결과 처리

- 권한 때문에 제외된 항목이 있어도 응답 구조는 동일.
- "일부 결과가 숨겨졌습니다" 같은 메시지는 기본적으로 노출하지 않는다.
- 단, 관리자에게는 `meta.warnings`에 선택적으로 표시 가능 (구현 Phase에서 결정).

### 12-4. 관리자 vs 일반 사용자의 결과 차이

- 동일한 쿼리(`GET /api/v1/documents?status=published`)라도 관리자는 더 많은 항목을 볼 수 있다.
- 이는 정상 동작이며 Authorization Layer가 담당.
- 응답 구조는 동일 — 차이는 데이터 내용에만 있다.

---

## 13. 응답 Metadata 연계 원칙

### 13-1. Offset 기반 meta.pagination

```json
"pagination": {
  "page": 2,
  "page_size": 20,
  "total": 87,
  "has_next": true,
  "has_prev": true
}
```

| 필드 | 필수 여부 | 설명 |
|---|---|---|
| `page` | 필수 | 현재 페이지 번호 |
| `page_size` | 필수 | 페이지당 항목 수 |
| `total` | 권장 | 전체 항목 수 (인가 필터 적용 후) |
| `has_next` | 필수 | 다음 페이지 존재 여부 |
| `has_prev` | 필수 | 이전 페이지 존재 여부 |

### 13-2. Cursor 기반 meta.pagination

```json
"pagination": {
  "limit": 20,
  "has_next": true,
  "has_prev": false,
  "next_cursor": "eyJpZCI6NTAsInRzIjoiMjAyNi0wNC0wMVQxMjowMDowMFoifQ",
  "prev_cursor": null
}
```

| 필드 | 필수 여부 | 설명 |
|---|---|---|
| `limit` | 필수 | 반환 항목 수 |
| `has_next` | 필수 | 다음 cursor 존재 여부 |
| `has_prev` | 조건부 | 이전 cursor 존재 여부 (backward 지원 시) |
| `next_cursor` | 조건부 | `has_next=true`일 때 포함 |
| `prev_cursor` | 조건부 | backward 지원 시 포함 |

### 13-3. Applied Filters / Sort 반영 여부

- `meta.pagination`에 적용된 filter 조건 요약을 반영하지 않는다 (응답 비대화 방지).
- 소비자는 본인이 보낸 요청 파라미터를 그대로 기록하면 됨.
- 단, `meta.warnings`에 무시된 필터 파라미터 경고는 포함 가능.

---

## 14. 예시 Query 및 응답 Metadata

### Offset Pagination 기본

```
GET /api/v1/documents?page=2&page_size=10
```
```json
"meta": {
  "request_id": "req_abc",
  "pagination": { "page": 2, "page_size": 10, "total": 87, "has_next": true, "has_prev": true }
}
```

### Cursor Pagination (audit-logs)

```
GET /api/v1/audit-logs?limit=20&cursor=eyJpZCI6MTIwfQ
```
```json
"meta": {
  "request_id": "req_def",
  "pagination": { "limit": 20, "has_next": true, "next_cursor": "eyJpZCI6MTQwfQ", "prev_cursor": null }
}
```

### 상태 필터

```
GET /api/v1/documents?status=published,draft
```

### 날짜 범위 필터

```
GET /api/v1/documents?created_after=2026-01-01&created_before=2026-03-31
GET /api/v1/audit-logs?occurred_after=2026-04-01T00:00:00Z&limit=50
```

### 다중 값 필터

```
GET /api/v1/documents?status=published,draft&document_type_id=dt_001,dt_002
```

### Boolean / Tag 필터

```
GET /api/v1/documents?has_attachments=true&tag=security,compliance
```

### 단일 정렬

```
GET /api/v1/documents?sort=-created_at
```

### 다중 정렬

```
GET /api/v1/documents?sort=status,-updated_at
```

### Filter + Sort + Pagination 조합

```
GET /api/v1/documents?status=published&created_after=2026-01-01&sort=-updated_at&page=1&page_size=20
```

### Search + Sort + Cursor 조합

```
POST /api/v1/searches
{
  "query": "개인정보",
  "filters": { "status": "published" },
  "sort": "-relevance",
  "limit": 10
}
```

### Audit Log 목록 조합 예시

```
GET /api/v1/audit-logs?actor_type=user&occurred_after=2026-04-01T00:00:00Z&limit=50
```
```json
"meta": {
  "request_id": "req_ghi",
  "pagination": {
    "limit": 50,
    "has_next": true,
    "next_cursor": "eyJ0cyI6IjIwMjYtMDQtMDFUMTI6MDA6MDBaIiwiaWQiOiI5OTkifQ"
  }
}
```

---

## 15. 후속 Task에 전달할 설계 기준

| Task | 전달 기준 |
|---|---|
| Task 3-7 (idempotency) | bulk 작업 결과 조회 시 operations 리소스에 cursor pagination 적용. |
| Task 3-8 (async/event/webhook) | events, webhook-deliveries에는 cursor pagination 적용. operations 목록은 offset 또는 cursor 선택. |
| Task 3-9 (AI/RAG 검색) | 검색 결과도 `data` 배열 + `meta.pagination` 공통 구조 사용. `sort=relevance` 허용. `meta.search` 섹션 추가. |
| Task 3-10 (오류 모델) | 잘못된 pagination/filter/sort 파라미터 오류 코드 정의. `INVALID_CURSOR`, `UNSUPPORTED_SORT_FIELD`, `INVALID_FILTER_VALUE` 등. |
| 구현 Phase | query parameter parser, validator, DB query builder에 이 규약 직접 반영. OpenAPI parameter 명세 작성. cursor 생성/파싱 로직 구현. |

---

## 16. 결론

Mimir 플랫폼의 목록 조회 규약은 다음으로 요약된다:

**Pagination**: Offset 기본(`page` + `page_size`), 대용량/시간순 리소스는 Cursor(`cursor` + `limit`)

**Filtering**: `?field=value` 단순 문법. 다중 값은 comma-separated. 날짜 범위는 `_after`/`_before` suffix.

**Sorting**: `?sort=field` (오름차순), `?sort=-field` (내림차순). 다중 정렬 쉼표 구분. Tie-breaker 필수.

**Metadata**: `meta.pagination`에 `has_next`, `total`(offset), `next_cursor`(cursor) 포함.

이 규약은 모든 collection endpoint에 예외 없이 적용되며, 구현자는 이 기준에서 임의로 벗어날 수 없다.

---

## 자체 점검 요약

### 권장 Pagination 전략

Offset 기본(`page`+`page_size`, 기본 20, 최대 100). Cursor는 audit-logs/events/webhook-deliveries에 적용. 동일 endpoint에서 두 방식 혼용 금지.

### Filtering 문법 핵심 요약

- 단일 값: `?status=published`
- 다중 값: `?status=published,draft` (comma-separated)
- 날짜 범위: `?created_after=2026-01-01&created_before=2026-03-31`
- Boolean: `?has_attachments=true`
- 자유 텍스트: `?q=키워드`
- 파라미터명: snake_case 표준화

### Sorting 문법 핵심 요약

`?sort=field`(오름차순), `?sort=-field`(내림차순), 다중: `?sort=f1,-f2`. Tie-breaker 필수. cursor 방식에서 지원 필드 제한.

### Search/List 관계 요약

`?q=키워드`는 일반 목록 필터. 구조화/시맨틱 검색은 `POST /api/v1/searches`. 검색 결과도 공통 pagination 구조 사용.

### 권한 기반 Listing 관점 요약

Authorization filtering은 보안 레이어 자동 적용. `total`은 소비자 권한 기준 건수. 숨겨진 항목 개수 노출 금지. 동일 쿼리라도 역할에 따라 결과 다름 — 정상 동작.

### 후속 Task 연결 포인트

3-7(bulk 결과 조회), 3-8(events/webhook cursor), 3-9(검색 결과 envelope + relevance sort), 3-10(잘못된 파라미터 오류 코드)

### 의도적으로 후속 Task로 남긴 미결정 사항

- cursor 내부 포맷 및 만료 정책 → 구현 Phase
- `estimated_total` 제공 여부 및 기준 → 구현 Phase
- 깊은 offset 성능 제한 기준(`page > 500` 등) → 구현 Phase
- metadata filter 상세 DSL → 구현 Phase
- 검색 결과 `meta.search` 상세 구조 → Task 3-9
- 오류 코드 카탈로그 (`INVALID_CURSOR` 등) → Task 3-10
- 관리자 대상 숨겨진 항목 경고 노출 여부 → 구현 Phase
