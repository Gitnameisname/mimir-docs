# Phase 2 - Task 2-6. 행위 추적(Activity Trace) 모델 설계

---

## 1. 목표

감사 로그(Audit Log)가 보안/컴플라이언스 목적의 단건 이벤트 증적이라면, Activity Trace는 **사용자와 시스템이 리소스를 어떻게 탐색하고, 조회하고, 작업 흐름을 이어가며 무엇을 했는지**의 연결된 맥락을 추적하는 별도 모델이다.

- Activity Trace의 목적과 Audit Log와의 경계를 명확히 정의한다
- Trace / Session / Event / Span 개념 관계를 정리한다
- Activity Event 범주와 데이터 구조를 설계한다
- 흐름 중심 모델링 원칙과 저장/집계 단위를 결정한다
- 프라이버시/비용/보존 원칙을 정립한다
- Task 2-7 정책 예외/우선순위 설계에 넘길 분석 입력 기준을 정리한다

---

## 2. 설계 전제

| # | 전제 |
|---|---|
| 1 | Task 2-5 결과: Audit Log는 명시적 비즈니스 행위 이벤트 중심. Activity Trace는 흐름/맥락/이용 패턴 중심으로 분리됨 |
| 2 | API-first 플랫폼. 웹 UI뿐 아니라 관리자 UI, 외부 API, 서비스 계정, AI 도구의 행위 흐름도 추적 대상 |
| 3 | 모든 세부 클릭/스크롤/hover를 저장하지 않는다. 의미 있는 작업 단위 이벤트부터 시작 |
| 4 | trace_id, correlation_id를 통해 Audit Log / API Request Log / Background Job과 연결 가능해야 함 |
| 5 | 사람 사용자와 비인간 주체(서비스 계정, AI 에이전트)를 동일 모델로 다룬다 |
| 6 | 프라이버시와 저장 비용을 동시에 고려. 분석 가치가 낮은 고빈도 이벤트는 억제 |

---

## 3. Activity Trace의 역할과 범위

### 3-1. Audit Log와의 비교

| 기준 | Audit Log | Activity Trace |
|---|---|---|
| **목적** | 보안/컴플라이언스 증적. "무슨 일이 있었는가" | 흐름/맥락/이용 패턴. "어떻게 작업했는가" |
| **기록 단위** | 명시적 비즈니스 행위 단건 이벤트 | 탐색 흐름, 작업 세션, 상호작용 연속체 |
| **read 포함** | 민감 리소스 read만 | 모든 의미 있는 조회/탐색 포함 |
| **무결성** | Append-only 필수. 변조 불가 | 권장. 분석 목적이므로 엄격하지 않음 |
| **보존** | 수년 (법적 기준) | 수개월 (분석 목적) |
| **접근 권한** | SecurityAuditor, PlatformAdmin 전용 | 운영팀, 분석팀, OrganizationAdmin(범위 내) |
| **활용** | 침해 조사, 감사, 권한 남용 탐지 | UX 개선, 협업 패턴 분석, AI 행위 이해, 이상 탐지 입력 |

### 3-2. 운영/분석 로그와의 관계

```
로그 계층:
  ┌─────────────────────────────────────────┐
  │  감사 로그 (Audit Log)                   │  ← 보안/컴플라이언스 전용
  │  - 명시적 행위 이벤트                    │
  │  - 장기 보존, append-only               │
  ├─────────────────────────────────────────┤
  │  Activity Trace                         │  ← 흐름/맥락 분석
  │  - 탐색/조회/작업 세션                   │
  │  - 중기 보존, trace 연결                 │
  ├─────────────────────────────────────────┤
  │  API Request Log                        │  ← 기술적 디버깅/성능
  │  - 모든 HTTP 요청                        │
  │  - 단기 보존, 구조적 쿼리 불필요          │
  ├─────────────────────────────────────────┤
  │  애플리케이션/디버그 로그                 │  ← 개발/운영 디버깅
  └─────────────────────────────────────────┘
```

### 3-3. 왜 단순 event log만으로는 부족한가

단순 이벤트 목록(`document.opened`, `search.executed` 등)만으로는 아래를 파악할 수 없다:
- 사용자가 검색 후 어떤 문서를 선택했는지 (검색 → 문서 연결)
- 문서를 열고 실제로 편집했는지 vs 읽기만 했는지
- 특정 작업을 완료했는지 중단했는지
- AI 에이전트가 어떤 흐름으로 여러 문서를 처리했는지

**흐름(trace)**이 있어야 "어떻게 작업했는가"를 이해할 수 있다.

### 3-4. Activity Trace의 핵심 목적

| 목적 | 설명 |
|---|---|
| **사용자 작업 흐름 이해** | 검색 → 열람 → 편집 → 게시 흐름을 연속으로 파악 |
| **탐색 경로 분석** | 어떤 공간/문서를 얼마나 자주 접근하는지, 탐색 패턴 |
| **협업/운영 패턴 분석** | 팀 단위 문서 활용 패턴, 검토 흐름 |
| **UX 개선 인사이트** | 어떤 기능이 잘 쓰이고 어디서 중단되는지 |
| **AI/자동화 작업 재현성** | AI 에이전트 작업 흐름 기록. 재현 및 디버깅 |
| **이상 행동 탐지 입력** | 비정상 대량 export, 반복 접근 실패, 비통상적 탐색 패턴 감지 입력 |

---

## 4. Trace / Session / Event / Span 개념 모델

### 4-1. 개념 정의

#### Interaction Session (상호작용 세션)

| 항목 | 내용 |
|---|---|
| **정의** | 사용자(또는 비인간 주체)의 하나의 연속된 작업 컨텍스트. 로그인~로그아웃, 또는 의미 있는 작업 단위 |
| **왜 필요한가** | "이 사람이 이번 접속에서 무엇을 했는가"를 묶어볼 수 있는 경계 단위 |
| **Audit Log 공유** | session_id를 Audit Log의 request context에 포함 가능 (선택적) |
| **MVP 필요 여부** | 포함 (session_id를 통한 그룹화) |
| **생성 위치** | 인증 시 백엔드에서 session_id 발급. 프론트엔드에서도 자체 interaction_session_id 생성 가능 |

> 로그인 세션과 작업 Trace는 **같지 않다**. 로그인 세션은 인증 수명이고, Trace는 하나의 작업 맥락이다. 한 로그인 세션에서 여러 Trace가 발생한다.

#### Trace (작업 흐름 묶음)

| 항목 | 내용 |
|---|---|
| **정의** | 하나의 의미 있는 작업 흐름을 묶는 식별자. 시작~완료가 있는 목적 지향적 행동 묶음 |
| **예시** | "특정 문서 검색 → 열람 → 편집 → 저장"이 하나의 Trace |
| **Audit Log 공유** | trace_id를 Audit Log에 포함하여 "이 감사 이벤트가 어떤 작업 흐름에서 발생했는가" 연결 |
| **MVP 필요 여부** | 포함 (핵심 연결 단위) |
| **생성 위치** | 백엔드 또는 API gateway에서 요청 클러스터로 자동 생성. 프론트엔드에서 명시적 시작 선언도 가능 |

> 검색 → 문서 열람 → 댓글 작성 흐름은 **하나의 Trace**로 묶을 수 있다. 다만 trace 경계를 자동으로 구분하기는 어려우므로, 초기에는 명시적 trace 시작/종료 또는 session 내 time window 기반으로 묶는다.

#### Span / Step (하위 구간)

| 항목 | 내용 |
|---|---|
| **정의** | Trace 내부의 단계별 구간. 특정 작업의 시작~완료를 표현하는 하위 단위 |
| **예시** | Trace "문서 편집 흐름" 안에 Span "편집 세션 시작", Span "섹션 3 수정", Span "저장 완료" |
| **Audit Log 공유** | span_id로 Audit Log의 특정 이벤트와 연결 가능 |
| **MVP 필요 여부** | **확장** — MVP에서는 Event 수준으로 충분. Span은 AI 에이전트 작업 등 복잡한 흐름에서 필요 |
| **생성 위치** | 주로 백엔드 (비동기 작업, AI 에이전트 실행 추적 시 유용) |

#### Event (개별 행동 기록)

| 항목 | 내용 |
|---|---|
| **정의** | 단일 시점의 개별 행동 기록. Activity Trace의 기본 저장 단위 |
| **예시** | `document.opened`, `search.executed`, `edit.committed` |
| **Audit Log 공유** | 일부 이벤트는 Audit Log에도 동시 기록 (이중 기록 원칙) |
| **MVP 필요 여부** | 필수 |
| **생성 위치** | 백엔드(API 처리 시점). 프론트엔드 이벤트는 백엔드 API를 통해 수집 |

> **하나의 API 요청은 Event다.** 단일 HTTP 요청은 원자적 행동이므로 Event로 표현. 여러 요청이 의미 있는 흐름을 이루면 Trace로 묶는다.
> **Background job / AI agent 실행도 Trace로 묶을 수 있다.** job_id 또는 agent_session_id를 trace_id로 사용.

#### Correlation ID

| 항목 | 내용 |
|---|---|
| **정의** | 시스템 전반을 가로지르는 연결용 식별자. 단일 요청에서 시작해 여러 시스템 로그에 공유됨 |
| **Audit Log 공유** | 동일 correlation_id로 Audit Log ↔ Activity Trace ↔ API Request Log 연결 |
| **MVP 필요 여부** | 필수 (Task 2-4에서 request_id로 이미 설계됨) |
| **생성 위치** | API Gateway 또는 Authentication Middleware에서 발급 |

### 4-2. 개념 계층 구조

```
Interaction Session (로그인 세션 또는 작업 컨텍스트)
  └── Trace (의미 있는 작업 흐름 묶음)
        ├── Span (복잡한 흐름의 하위 구간 — 확장)
        │     └── Event (개별 행동)
        └── Event (단순 흐름의 개별 행동)

[공유 식별자]
  correlation_id / request_id ─── Audit Log와 연결
  trace_id ──────────────────── 작업 흐름 연결
  session_id ────────────────── 세션 내 이벤트 묶음
```

---

## 5. Activity Event 범주

### 5-1. 탐색/조회 계열

| 이벤트명 | 설명 | Audit 병행 | MVP |
|---|---|:---:|:---:|
| `document.opened` | 문서 열람 시작 | 민감 문서만 | 필수 |
| `document.version.viewed` | 특정 버전 열람 | 아니오 | 필수 |
| `document.node.expanded` | 섹션/노드 펼침 | 아니오 | 확장 |
| `document.attachment.previewed` | 첨부파일 미리보기 | 아니오 | 확장 |
| `document.compared` | 두 버전 비교 | 아니오 | 확장 |
| `space.browsed` | 공간 탐색 | 아니오 | 필수 |

### 5-2. 검색/발견 계열

| 이벤트명 | 설명 | Audit 병행 | MVP |
|---|---|:---:|:---:|
| `search.executed` | 검색 쿼리 실행 | 아니오 | 필수 |
| `search.result.opened` | 검색 결과에서 문서 열람 | 아니오 | 필수 |
| `search.filter.changed` | 필터 조건 변경 | 아니오 | 확장 |
| `search.query.saved` | 검색 쿼리 저장 | 아니오 | 확장 |

### 5-3. 편집/작업 계열

| 이벤트명 | 설명 | Audit 병행 | MVP |
|---|---|:---:|:---:|
| `edit.started` | 편집 모드 진입 | 아니오 | 필수 |
| `edit.committed` | 편집 저장 완료 (Version 생성) | **예** (`document.updated`) | 필수 |
| `edit.abandoned` | 편집 중단/취소 | 아니오 | 필수 |
| `edit.autosaved` | 자동 저장 (draft) | 아니오 | **요약만** (고빈도 억제) |
| `comment.submitted` | 코멘트 제출 | **예** (`document.comment.created`) | 필수 |
| `attachment.uploaded` | 첨부파일 업로드 | **예** (`document.attachment.uploaded`) | 필수 |
| `version.restored` | 버전 복원 | **예** (`document.version.restored`) | 필수 |

### 5-4. 협업/공유 계열

| 이벤트명 | 설명 | Audit 병행 | MVP |
|---|---|:---:|:---:|
| `share.initiated` | 공유 액션 시작 | **예** (`document.share_link.created`) | 확장 |
| `review.requested` | 검토 요청 | **예** (워크플로) | 확장 |
| `approval.action.taken` | 승인/반려 | **예** (`workflow.approval.*`) | 확장 |

### 5-5. 시스템/자동화 계열

| 이벤트명 | 설명 | Audit 병행 | MVP |
|---|---|:---:|:---:|
| `api.request.grouped` | 외부 API 요청 묶음 (trace 단위) | 일부 (중요 행위만) | 필수 |
| `export.initiated` | export 작업 시작 | **예** (`document.exported`) | 필수 |
| `export.completed` | export 완료 | **예** | 필수 |
| `ai.action.started` | AI 어시스턴트 작업 시작 | 아니오 | 필수 |
| `ai.action.completed` | AI 어시스턴트 작업 완료 | 아니오 | 필수 |
| `background.job.triggered` | 백그라운드 작업 시작 | 일부 | 확장 |

---

## 6. Activity Trace 데이터 구조

### 6-1. ActivityEvent 레코드 구조

```
ActivityEvent {

  // ─── 레코드 식별 ───────────────────────────────
  id:               string      // 고유 식별자
  event_type:       string      // "document.opened", "search.executed", ...
  event_category:   string      // "navigation" | "search" | "edit" | "collaboration" | "system"
  occurred_at:      datetime    // 발생 시각 (UTC)
  duration_ms:      int | null  // 행위 지속 시간 (편집 세션 등 구간 이벤트)

  // ─── Actor / Session ─────────────────────────
  actor_type:       string      // "user" | "service_account" | "ai_agent" | "system"
  actor_id:         string      // user_id 또는 service_account_id
  acting_org_id:    string | null
  session_id:       string | null  // 사용자 상호작용 세션 ID
  client_type:      string | null  // "web" | "mobile" | "api" | "ai_agent"

  // ─── Trace 연결 ──────────────────────────────
  trace_id:         string | null  // 작업 흐름 묶음 ID
  parent_trace_id:  string | null  // 중첩 trace (AI agent 내 sub-workflow 등)
  span_id:          string | null  // 하위 구간 ID (확장)
  correlation_id:   string         // Audit Log / API Request Log 연결용 (= request_id)

  // ─── Target ──────────────────────────────────
  resource_type:    string | null  // "document" | "space" | "version" | ...
  resource_id:      string | null  // 특정 리소스 ID
  parent_resource_type: string | null
  parent_resource_id:   string | null
  resource_scope:   string | null  // "org_001/space_xyz"와 같은 scope 경로

  // ─── 검색/쿼리 컨텍스트 ─────────────────────
  query_summary:    string | null  // 검색어 요약 (원문 저장 금지 — 개인정보)
  filter_summary:   string | null  // 적용 필터 요약

  // ─── Context ─────────────────────────────────
  source_channel:   string      // "user_ui" | "admin_ui" | "external_api" | "service_account" | "ai_agent"
  api_endpoint:     string | null
  surface:          string | null  // UI 화면 식별자 (선택. "document_detail", "search_results" 등)

  // ─── Outcome ─────────────────────────────────
  outcome_type:     string      // "completed" | "abandoned" | "failed" | "in_progress"
  result_summary:   object | null // { item_count, changed_field_count, version_id 등 }
}
```

### 6-2. 필드별 상세 설명

| 필드 그룹 | 핵심 필드 | 필수 | MVP | 민감도 | 설명 |
|---|---|:---:|:---:|---|---|
| 레코드 식별 | `id`, `event_type`, `occurred_at` | 필수 | 포함 | 낮음 | 기본 |
| 레코드 식별 | `duration_ms` | 선택 | 포함 | 낮음 | 편집 세션, export 등 구간 이벤트 |
| Actor | `actor_type`, `actor_id` | 필수 | 포함 | 중간 | 누가 |
| Actor | `acting_org_id`, `session_id` | 권장 | 포함 | 낮음 | 조직 컨텍스트, 세션 묶음 |
| Actor | `client_type` | 선택 | 포함 | 낮음 | 채널 분석 |
| Trace | `trace_id`, `correlation_id` | 권장 | 포함 | 낮음 | 흐름 연결. correlation_id는 필수 |
| Trace | `parent_trace_id`, `span_id` | 선택 | 확장 | 낮음 | 복잡한 AI/자동화 흐름 |
| Target | `resource_type`, `resource_id` | 권장 | 포함 | 낮음 | 무엇에 대해 |
| Target | `resource_scope` | 선택 | 포함 | 낮음 | 범위 분석 |
| 검색 | `query_summary` | 선택 | 포함 | **높음** | 원문 금지. 쿼리 해시 또는 분류만 |
| Context | `source_channel`, `correlation_id` | 필수 | 포함 | 낮음 | 채널 구분, Audit 연결 |
| Context | `api_endpoint`, `surface` | 선택 | 포함 | 낮음 | 디버깅/분석 |
| Outcome | `outcome_type`, `result_summary` | 권장 | 포함 | 낮음 | 완료/중단/실패 구분 |

---

## 7. 흐름 중심 모델링 원칙

### 7-1. 핵심 질문 답변

**Q. 단순 `document.opened`만 남기면 무엇이 부족한가?**
> - "열었다"는 알지만 "읽었는지, 편집했는지, 어디서 왔는지"를 알 수 없다
> - 검색에서 왔는지 직접 URL 접근인지 구분 불가 → 탐색 경로 분석 불가
> - 이후 edit, save, abandon 중 어느 것으로 이어졌는지 연결 불가

**Q. search → open → edit → save → export 흐름을 어떻게 연결할 것인가?**
> 동일 `trace_id`를 공유한 이벤트 체인으로 연결한다.
> trace 시작: 검색 쿼리 실행 시 trace_id 발급
> trace 종료: export 완료 또는 일정 시간 inactivity 후 trace 종결

**Q. 검색어 변경을 각각 event로 볼지, 하나의 search session으로 묶을지?**
> **동일 trace 내 search session으로 묶는다.** 검색어 변경은 개별 이벤트로 저장하되 동일 trace_id를 유지. 단, 고빈도 타이핑 중 발생하는 임시 쿼리는 억제하고 최종 쿼리만 기록.

**Q. autosave 같은 고빈도 이벤트는 모두 저장해야 하는가?**
> 아니다. autosave는 대표 이벤트만 샘플링하거나 세션 종료 시 총 autosave 횟수 요약으로 처리. 개별 autosave 이벤트 전량 저장은 저장 비용 낭비.

**Q. 사용자의 "작업 의도"를 어떤 수준까지 추론 가능한 구조로 남겨야 하는가?**
> 의도를 직접 저장하지 않고, **흐름 이벤트를 연결(trace)**하면 의도를 사후 추론할 수 있다. 예: search → open → edit.committed 흐름 = "검색으로 찾아 수정한 작업"으로 해석 가능.

### 7-2. 원칙 정리

**Trace로 묶어야 하는 행동**
- 검색 → 문서 열람 → 편집 → 저장 (목적 지향적 연속 작업)
- AI 에이전트의 다문서 검색 → 요약 생성 흐름
- 외부 API 클라이언트의 batch 요청 흐름
- Export job 시작 → 처리 → 완료

**단일 Event로 충분한 행동**
- 독립적인 문서 열람 (trace 연결 없는 단순 조회)
- 공간 탐색
- 설정 화면 진입

**Span/Step이 필요한 흐름**
- AI 에이전트 멀티스텝 작업 (Phase 10+ AI 연계 시)
- 복잡한 approval workflow (Phase 5+)
- Background job 내 단계별 처리 (확장 단계)

**샘플링 또는 요약이 필요한 고빈도 이벤트**
- autosave: 세션 내 횟수 요약만
- 검색어 입력 중 임시 쿼리: 최종 쿼리만 기록
- 스크롤/hover: 기록하지 않음
- typing: 기록하지 않음

---

## 8. Audit Log와의 분리 및 연계 원칙

### 8-1. 이벤트별 기록 분류

| 이벤트 | Audit only | Activity only | 이중 기록 |
|---|:---:|:---:|:---:|
| 로그인 성공/실패 | ● | | |
| Authorization denied | ● | | |
| 권한/역할/ACL 변경 | ● | | |
| API Credential 발급/폐기 | ● | | |
| 감사 로그 조회 | ● | | |
| 문서 열람 (`document.opened`) | | ● | |
| 검색 실행 (`search.executed`) | | ● | |
| 편집 시작/중단 | | ● | |
| 공간 탐색 | | ● | |
| 문서 수정 (저장 완료) | | | ● |
| 문서 생성/삭제/게시 | | | ● |
| 문서 export | | | ● |
| 코멘트 제출 | | | ● (Audit: 확장) |
| 첨부파일 업로드/삭제 | | | ● |
| 버전 복원 | | | ● |
| AI 에이전트 작업 시작/완료 | | ● | |
| 외부 API 요청 흐름 | | ● | |

### 8-2. 이중 기록 원칙

이중 기록이 필요한 이벤트는 두 시스템에 **별도 레코드**로 저장한다. 동일 레코드를 공유하지 않는다.

- Audit Log: 보안/컴플라이언스 필드 중심 (`actor`, `target`, `decision`, `before/after`)
- Activity Trace: 흐름/맥락 필드 중심 (`trace_id`, `session_id`, `outcome`, `duration_ms`)

**연결 키(공유 식별자):**

| 식별자 | 역할 |
|---|---|
| `correlation_id` (= `request_id`) | 단일 API 요청에서 발생한 Audit Event와 Activity Event를 연결 |
| `trace_id` | 하나의 작업 흐름 내 여러 이벤트 연결 (Activity 전용. Audit에는 선택적 포함) |
| `actor_id` | 동일 사용자의 Audit/Activity 이벤트 연결 |
| `resource_id` | 동일 리소스에 대한 Audit/Activity 이벤트 연결 |
| `session_id` | 세션 내 전체 이벤트 연결 |

### 8-3. 예시별 분류

```
document.opened
  → Activity Trace만 (단순 열람. 보안/컴플라이언스 증적 불필요)

search.executed
  → Activity Trace만 (탐색 패턴 분석. 감사 불필요)

document.updated (편집 저장)
  → Audit Log: { event_type: "document.updated", result: "success",
                 before_summary: {version: "ver_001"}, after_summary: {version: "ver_002"} }
  → Activity Trace: { event_type: "edit.committed", trace_id: "trace_xyz",
                      duration_ms: 840000, outcome_type: "completed" }
  → correlation_id로 두 레코드 연결

permission.acl.created
  → Audit Log만 (보안 전용. 활동 흐름 분석 불필요)

login.failed
  → Audit Log만 (보안 사고 탐지. Activity에 불필요)

export.initiated
  → Audit Log: { event_type: "document.exported" }
  → Activity Trace: { event_type: "export.initiated", trace_id: "trace_export_001" }

ai.action.started
  → Activity Trace만 (AI 흐름 추적. 일반 감사 대상 아님)
```

---

## 9. 저장 단위와 집계 단위

### 9-1. Raw Event 저장 구조

기본 저장 단위는 개별 `ActivityEvent` 레코드. trace_id로 흐름을 쿼리로 재구성한다.

```
// Raw Event 예시
[
  { id: "evt_001", event_type: "search.executed", trace_id: "trace_xyz",
    query_summary: "category:policy", occurred_at: "T10:00:00" },
  { id: "evt_002", event_type: "document.opened", trace_id: "trace_xyz",
    resource_id: "doc_123", occurred_at: "T10:00:05" },
  { id: "evt_003", event_type: "edit.started", trace_id: "trace_xyz",
    resource_id: "doc_123", occurred_at: "T10:00:10" },
  { id: "evt_004", event_type: "edit.committed", trace_id: "trace_xyz",
    resource_id: "doc_123", duration_ms: 600000, occurred_at: "T10:10:10" }
]
```

### 9-2. Trace Summary 구조

긴 작업 흐름이 완료되면 Trace Summary를 생성하여 집계 분석에 활용한다.

```
TraceSession {
  trace_id:         string
  actor_id:         string
  acting_org_id:    string
  started_at:       datetime
  ended_at:         datetime | null
  duration_ms:      int
  event_count:      int
  event_types:      string[]   // 포함된 event_type 목록
  touched_resources: [{ resource_type, resource_id }]  // 접근한 리소스 목록
  outcome:          string     // "completed" | "abandoned" | "in_progress"
  source_channel:   string
}
```

Trace Summary는 Raw Event보다 오래 보존하고, 집계 쿼리에 사용한다.

### 9-3. Session 요약 구조

```
InteractionSessionSummary {
  session_id:       string
  actor_id:         string
  started_at:       datetime
  ended_at:         datetime
  trace_count:      int
  document_count:   int        // 접근한 문서 수
  edit_count:       int        // 편집 완료 수
  search_count:     int        // 검색 실행 수
  export_count:     int        // export 수
  source_channel:   string
}
```

### 9-4. 운영 분석 파이프라인으로 넘기기 좋은 구조

- Raw Event → Trace Summary → Session Summary 계층으로 집계
- Raw Event는 짧게 보존 (3~6개월), Summary는 더 오래 (1~2년)
- 이벤트 스트림(Kafka, 이벤트 큐)으로 내보내어 별도 분석 파이프라인 구축 가능
- `correlation_id`를 통해 외부 APM/분산 추적 도구와 연결 가능

---

## 10. 프라이버시 / 비용 / 보존 원칙

### 10-1. 최소 수집 원칙

- 분석에 의미 있는 이벤트만 수집한다
- "모든 것을 수집하고 나중에 필터링"이 아닌 "필요한 것만 수집"이 기본
- UI 미세 동작(hover, scroll, 클릭)은 수집하지 않는다
- 초기 MVP에서는 의미 있는 작업 단위 이벤트 14종만 수집

### 10-2. 민감 필드 처리 원칙

| 필드 | 처리 원칙 |
|---|---|
| 검색어 (`query_text`) | **원문 저장 금지.** 해시 또는 카테고리 분류만 저장. 예: `query_summary: "category:policy length:medium"` |
| 문서 본문 내용 | **저장 금지.** resource_id와 resource_type만 |
| IP 주소 | 선택적 포함. 마스킹(끝자리 제거) 또는 저장 안 함 |
| 사용자 이름/이메일 | actor_id(UUID)만 저장. 이름/이메일은 저장 안 함 |
| 코멘트 내용 | 저장 금지. event_type과 resource_id만 |

### 10-3. 고빈도 이벤트 억제 원칙

| 이벤트 | 억제 방식 |
|---|---|
| autosave | 세션당 첫 autosave만 기록 + 세션 종료 시 총 횟수 요약 |
| 검색어 입력 (debounce 전) | 최종 confirmed 쿼리만 기록 |
| 탭/패널 전환 | 초 단위 연속 전환은 마지막 상태만 기록 |
| 스크롤/hover | 수집 안 함 |
| typing (에디터) | 수집 안 함 |

### 10-4. 보존 기간 초안

| 데이터 유형 | 보존 기간 | 이유 |
|---|---|---|
| Raw ActivityEvent | 3~6개월 | 저장 비용 제어. 단기 분석 목적 |
| TraceSession Summary | 1~2년 | 중기 패턴 분석 |
| InteractionSessionSummary | 1~2년 | 사용자 활동 패턴 추세 분석 |
| 집계 통계 (일별/월별) | 3~5년 | 장기 운영 지표 |

> Audit Log(수년)보다 훨씬 짧다. Activity Trace는 분석 목적이므로 장기 보존 필요 없음.

### 10-5. 비용 통제 원칙

- Raw Event 보존 3~6개월 후 Trace Summary로 집계하고 raw 삭제
- 세션 종료 시 Session Summary 생성 후 해당 세션의 raw event는 요약 기간 후 삭제 가능
- 고빈도 이벤트(autosave 등) 억제로 저장 용량 제어
- 검색/탐색 이벤트는 sample rate를 도입하여 일부만 수집 가능 (분석 정확도 허용 범위 내)

---

## 11. 외부 API / AI 도구 / 자동화 반영 원칙

### 11-1. 외부 API 클라이언트의 다수 요청 묶음

```
// 외부 클라이언트가 여러 문서를 batch 조회하는 경우
// 클라이언트가 X-Trace-Id 헤더로 trace_id를 직접 제공하거나
// API Gateway에서 자동으로 session 기반 trace_id 발급

ActivityEvent {
  event_type:    "api.request.grouped",
  actor_type:    "service_account",
  trace_id:      "trace_api_batch_001",
  source_channel: "external_api",
  result_summary: { request_count: 12, resource_type: "document" }
}
```

### 11-2. 서비스 계정 배치 작업 (session 없는 경우)

- session_id = null 허용. trace_id는 job_id 또는 배치 실행 ID로 대체
- 배치 작업 시작 / 완료 이벤트로 trace 범위를 명시적으로 표현

```
ActivityEvent { event_type: "background.job.triggered", trace_id: "job_cleanup_20260401",
                actor_type: "system", session_id: null }
```

### 11-3. AI 에이전트의 다문서 처리 흐름

```
// AI 에이전트가 여러 문서를 검색/읽기/요약하는 흐름
Trace: trace_id = "ai_session_abc"
  Event 1: { event_type: "ai.action.started", actor_type: "ai_agent" }
  Event 2: { event_type: "search.executed", trace_id: "ai_session_abc" }
  Event 3: { event_type: "document.opened", resource_id: "doc_001", trace_id: "ai_session_abc" }
  Event 4: { event_type: "document.opened", resource_id: "doc_002", trace_id: "ai_session_abc" }
  Event 5: { event_type: "ai.action.completed", result_summary: { docs_processed: 2 } }
```

### 11-4. Background Job vs 사용자 Initiated 작업 구분

| actor_type | 설명 |
|---|---|
| `user` | 사람 사용자의 직접 행위 |
| `service_account` | 외부 API 클라이언트, 자동화 파이프라인 |
| `ai_agent` | AI 어시스턴트, LLM 기반 에이전트 |
| `system` | 플랫폼 내부 배치 작업, 스케줄러 |

### 11-5. 사람과 비인간 주체의 동일 모델 처리

동일한 `ActivityEvent` 스키마를 모든 actor_type에 적용한다. `actor_type` 필드로 구분한다.

- 분석 쿼리 시: `WHERE actor_type = 'user'` vs `WHERE actor_type = 'ai_agent'`
- 이상 행동 탐지 시: 비인간 주체의 비정상 패턴(비통상적 대량 접근 등)도 동일 구조로 감지 가능

### 11-6. MVP 반영 수준

- **MVP에서 반드시**: user actor의 기본 이벤트 (탐색, 검색, 편집, export)
- **MVP에서 부분 반영**: service_account의 api.request.grouped (trace_id 전달)
- **확장 단계**: ai_agent 흐름 추적, background job 상세 추적, cross-system workflow lineage

---

## 12. MVP Activity Trace 범위 제한안

### 12-1. MVP 필수 수집 이벤트

| 이벤트명 | 이유 |
|---|---|
| `document.opened` | 문서 활용 패턴 분석의 기본 |
| `document.version.viewed` | 버전 열람 패턴 |
| `space.browsed` | 탐색 경로 분석 |
| `search.executed` | 검색 이용 패턴 |
| `search.result.opened` | 검색 → 열람 전환율 분석 |
| `edit.started` | 편집 진입 추적 |
| `edit.committed` | 편집 완료 (Audit도 병행) |
| `edit.abandoned` | 편집 중단률 분석 |
| `comment.submitted` | 협업 활동 추적 |
| `attachment.uploaded` | 첨부 활용 추적 |
| `export.initiated` | export 행위 추적 (Audit도 병행) |
| `export.completed` | export 완료 (Audit도 병행) |
| `api.request.grouped` | 외부 API 클라이언트 흐름 |
| `ai.action.started` / `ai.action.completed` | AI 에이전트 작업 추적 |

### 12-2. 확장 단계 이벤트

| 이벤트 | 도입 조건 |
|---|---|
| `document.node.expanded` | 상세 탐색 분석 필요 시 |
| `search.filter.changed` | 검색 필터 이용 분석 필요 시 |
| `document.compared` | 버전 비교 기능 도입 시 |
| `share.initiated`, `review.requested` | 워크플로 기능 도입 시 |
| `background.job.*` 상세 | Job 분석 필요 시 |

---

## 13. 다음 Task로 넘길 결정사항

### Task 2-7 (정책 적용 우선순위와 예외 규칙) 에 전달

**Activity Trace가 정책 위반 탐지에 주는 힌트:**

| 패턴 | 감지 가능 흐름 |
|---|---|
| 반복된 접근 거부 후 성공 | `auth.access.denied` (Audit) + `document.opened` (Activity) 연속 → 권한 우회 의심 |
| 비정상 대량 export | `export.initiated` 이벤트가 짧은 시간 내 비정상적으로 많은 경우 → 데이터 유출 의심 |
| 비통상적 탐색 패턴 | 새벽 시간대 대량 문서 열람, 평소와 다른 공간 접근 패턴 |
| AI 에이전트의 과도한 접근 | `ai.action.started` 후 `document.opened` 이벤트 수가 임계치 초과 |
| 권한 변경 직후 즉각 행위 | `permission.acl.created` (Audit) 직후 같은 actor의 이전에 못 하던 행위 시도 |

**Audit + Activity 조합 분석으로 가능한 판단:**
- Audit의 `authorization_path`와 Activity의 trace_id를 연결 → "어떤 작업 흐름에서 권한 예외가 발생했는가" 파악
- Activity의 이상 패턴이 감지되면 Audit Log에서 해당 시간대 권한 변경 이력 확인
- 정책 예외 케이스(Task 2-7) 설계 시 "어떤 행동 패턴이 예외 정책 대상인가" 근거 제공

---

## 14. 오픈 이슈

| # | 이슈 | 설명 | 결정 필요 시점 |
|---|---|---|---|
| OI-1 | Trace 경계 자동 감지 방법 | 사용자가 명시적으로 trace를 시작/종료하지 않을 때, time window 기반으로 trace를 자동 분리하는 기준 (예: 30분 inactivity → trace 종료) | Phase 3 구현 시 |
| OI-2 | 검색어 요약 방식 | 원문 저장 금지 원칙 하에, 어느 수준의 요약이 분석에 유용한가. 길이/언어/도메인 분류 vs 해시 | Phase 8 검색 기능 설계 시 |
| OI-3 | client_type 자동 감지 | "web", "api", "ai_agent" 구분을 User-Agent 또는 API 파라미터로 자동 감지할 수 있는가 | Phase 3 API 구현 시 |
| OI-4 | GDPR 삭제 요청 시 Activity Trace 처리 | 사용자 삭제 요청 시 Activity Trace의 actor_id 익명화 방식. Audit Log와 달리 삭제 가능한지 | Phase 13 법적 검토 |
| OI-5 | Trace Summary 생성 시점 | trace 종료가 명시적이지 않을 때 Summary를 언제 생성할 것인가. 실시간 vs 배치 집계 | Phase 3 이후 분석 파이프라인 설계 시 |

---

*작성일: 2026-04-01*
*Phase: 2 / Task: 2-6*
*상태: 설계 초안 완료*
