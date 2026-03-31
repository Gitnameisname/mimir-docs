# Phase 0 — 시스템 컨텍스트 요약

## 문서 목적

이 문서는 Mimir 플랫폼의 시스템 컨텍스트를 정리한다. 플랫폼과 상호작용하는 주요 참여자, 외부 시스템, 연결 방식을 설명하고, Phase 1/2 설계에 주는 경계 관련 시사점을 제공한다.

---

## 1. 시스템 컨텍스트 다이어그램

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         외부 참여자 및 시스템                                 │
│                                                                          │
│  [End User]              [Admin/Operator]                                │ 
│      │                        │                                          │ 
│      │ 탐색·조회·작성·편집        │ 운영·정책·감사·관리                           │
│      ▼                        ▼                                          │
│  [User UI]              [Admin UI]                                       │
│      │  API 호출               │  Admin API 호출                           │
└──────┼─────────────────────────┼─────────────────────────────────────────┘
       │                         │
       ▼                         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                      Mimir Knowledge Platform Core                       │
│                                                                          │
│  ┌─────────────────────────────────────────────────────┐                 │
│  │  API / Application Layer                            │                 │
│  │  (라우팅, 인증 미들웨어, 직렬화, 에러 처리)                  │                 │
│  └──────────────────────┬──────────────────────────────┘                 │
│                         │                                                │
│  ┌──────────────────────▼──────────────────────────────┐                 │
│  │  Core Domain                                        │                 │
│  │  Document · Version · Node · Metadata               │                 │
│  │  DocumentType · Status · Permission · Audit         │                 │
│  └──────────────────────┬──────────────────────────────┘                 │
│                         │                                                │
│  ┌──────────┬───────────┼───────────┬─────────────────┐                  │
│  │          │           │           │                 │                  │
│  ▼          ▼           ▼           ▼                 ▼                  │
│ [Search   [File/     [Auth/      [Audit/         [Async Job/             │
│  Index    Resource   Permission  Operations      Event Layer]            │
│  Layer]   Layer]     Boundary]   Layer]                                  │
└───┬───────────┬──────────┬───────────┬─────────────────┬─────────────────┘
    │           │          │           │                 │
    │           │          │           │        [Integration/Connector Layer]
    ▼           ▼          ▼           ▼                 │
[Search    [Object   [Identity   [Log/SIEM           ┌───┴─────────────────┐
 Engine]   Storage]  / SSO]      System]             │ External Systems:   │
                                                     │ · BPM/Workflow      │
                                                     │ · AI/NLP Services   │
                                                     │ · Business Apps     │
                                                     │ · Doc Source Conn.  │
                                                     │ · Notification Sys  │
                                                     └─────────────────────┘
         ▲
[External API Consumers]
(AI Agents / Automation / External Apps)
```

---

## 2. 주요 참여자

| 참여자 | 역할 | API 접근 경로 | 권한 수준 |
|--------|------|-------------|---------|
| **End User** | 문서 탐색·조회·작성·편집 | User UI → `/api/v1/*` | 인증된 일반 사용자 |
| **Admin / Operator** | 운영·정책·감사·관리 | Admin UI → `/api/v1/admin/*` | 관리자 권한 |
| **External Application** | 업무 시스템 연동, 데이터 수집 | `/api/v1/*`, `/api/v1/integration/*` | 서비스 계정 |
| **AI Agent / Automation** | 지식 자산 수집·처리·활용 | `/api/v1/*` (읽기 중심) | 읽기 전용 또는 제한 쓰기 |

---

## 3. 주요 외부 시스템과 연결 방식

| 외부 시스템 | 연결 목적 | 연결 방식 | 데이터 소유권 | 격리 원칙 |
|-----------|---------|---------|------------|---------|
| **Identity / SSO** | 사용자 인증 토큰 검증 | OIDC / SAML (Auth Boundary) | 외부 IdP | 인증 실패 시 플랫폼 코어 영향 최소화 |
| **Object Storage** | 첨부 파일·버전 본문 저장 | S3 호환 API (File/Resource Layer 어댑터) | 외부 스토리지 | 어댑터 추상화로 공급자 교체 가능 |
| **Search Engine** | 전문 검색·메타데이터 필터 | HTTP/REST (Search Index Layer 어댑터) | 플랫폼 (인덱스 파생) | 엔진 장애 시 기본 조회는 RDB로 유지 |
| **Notification / Messaging** | 문서 상태 변경 알림 발송 | 이벤트 발행 → 외부 시스템 구독 | 외부 시스템 | 알림 실패가 플랫폼 기능에 영향 없음 |
| **BPM / Workflow** | 복잡한 승인 프로세스 처리 | 이벤트 발행 → BPM 소비 (비동기) | 외부 BPM | BPM 장애가 플랫폼 상태 저장에 영향 없음 |
| **AI / NLP Services** | 문서 요약·분류·임베딩 생성 | REST API (Integration/Connector Layer) | AI 서비스 생성 → 플랫폼 보조 저장 | AI 결과는 원본 Metadata와 분리 저장 |
| **External Business Apps** | 지식 자산 조회·동기화 | 플랫폼 API 소비, 이벤트 구독 | 플랫폼 (원본) | 외부 앱 요청이 플랫폼 도메인 모델 변경 유발 금지 |
| **Document Source Connectors** | 기존 시스템 문서 마이그레이션 | 커넥터 → 플랫폼 API 투입 | 플랫폼 (변환본) | 소스 시스템 종속 로직이 코어에 진입 금지 |
| **Observability / Logging** | 운영 로그·메트릭 수집 | 구조화 로그 출력, 메트릭 엔드포인트 | 플랫폼 (원본 이벤트) | 모니터링 시스템 장애가 플랫폼 기능에 영향 없음 |

---

## 4. 연결 방식 요약

| 연결 방식 | 사용 대상 |
|---------|---------|
| **동기 REST API** | User UI, Admin UI, External App, AI Agent의 직접 기능 요청 |
| **비동기 이벤트 발행** | 감사 로그 기록, 검색 인덱스 갱신, 알림 발송 트리거, BPM 연동 |
| **어댑터/커넥터 계층** | 오브젝트 스토리지, 검색 엔진, AI 서비스, 외부 문서 소스 |
| **이벤트 구독** | 외부 시스템이 플랫폼 이벤트를 소비하는 방식 (Webhook, 메시지 브로커) |

---

## 5. Phase 1/2 설계 시사점

### Phase 1 (도메인 모델 설계)

**외부 시스템 구조를 도메인 모델에 반영하지 않는다.**

- User 엔티티는 플랫폼 내부 식별자와 속성만 포함한다. 외부 SSO 클레임 구조(Azure AD OID, LDAP DN 등)는 별도 매핑 테이블로 분리한다.
- Document 상태 모델은 플랫폼의 지식 자산 라이프사이클을 기준으로 정의한다. 외부 BPM의 태스크 개념을 상태 모델에 반영하지 않는다.
- Attachment 엔티티는 파일 참조 경로만 가진다. 오브젝트 스토리지 공급자 전용 필드(예: S3 bucket, GCS bucket)를 도메인 엔티티에 직접 포함하지 않는다.

**경계가 명확한 도메인 이벤트를 설계한다.**

- `DocumentCreated`, `VersionCreated`, `StatusChanged`, `DocumentDeleted` 같은 도메인 이벤트를 도메인 모델에서 정의한다.
- 이 이벤트들이 외부 시스템 연계의 트리거가 된다. 외부 시스템의 반응은 플랫폼 코어와 무관하다.

### Phase 2 (API·서비스 설계)

**내부 책임 API와 외부 연계 API를 명확히 분리한다.**

- `/api/v1/*` (코어 기능) — 모든 소비자 대상, 인증 필수
- `/api/v1/admin/*` (관리자 기능) — 관리자 권한 필수
- `/api/v1/integration/*` (외부 연계) — 서비스 계정 인증, 연계 전용 엔드포인트

**커넥터 계층을 코어 서비스와 분리하여 설계한다.**

- 오브젝트 스토리지 어댑터, 검색 엔진 어댑터, AI 서비스 클라이언트는 인프라 계층의 독립 모듈로 설계한다.
- 코어 도메인 서비스는 이 어댑터들의 구체적 구현에 의존하지 않고, 인터페이스에만 의존한다.

**Admin 기능에서 코어 운영과 외부 통합 운영을 구분한다.**

- Admin UI에서 제공하는 기능 중 "플랫폼 내부 운영"(DocumentType 관리, 권한 설정, 감사 조회)과 "외부 통합 운영"(Webhook 설정, 외부 커넥터 상태 확인)을 별도 영역으로 설계한다.
- 외부 통합 설정이 코어 운영 화면과 혼재하지 않도록 Admin UI 내에서도 영역을 분리한다.

---

## 참조 문서

- [Task7_system_boundary_definition.md](Task7_system_boundary_definition.md) — 코어 책임 정의, 내부 서브시스템, 외부 시스템 분류, 위반 사례
- [Task7_internal_external_responsibility_matrix.md](Task7_internal_external_responsibility_matrix.md) — 기능별 내부/외부 책임 매트릭스
