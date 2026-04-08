# Task 7-1 산출물: Admin UI 정보 구조 및 접근 정책

---

## 1. Admin UI 정보 구조 (Information Architecture)

### 1-1. 전체 화면 맵

```
Admin UI (/admin)
├── /admin                        → /admin/dashboard 리다이렉트
├── /admin/dashboard              → 시스템 상태 대시보드 (운영자 홈)
│
├── /admin/users                  → 사용자 목록
│   └── /admin/users/[user_id]   → 사용자 상세
│
├── /admin/organizations          → 조직 목록
│   └── /admin/organizations/[org_id] → 조직 상세
│
├── /admin/roles                  → 역할 목록
│   └── /admin/roles/[role_id]   → 역할 상세
│
├── /admin/permissions            → 권한 정책 목록
│   └── /admin/permissions/[policy_id] → 정책 상세
│
├── /admin/document-types         → DocumentType 목록
│   └── /admin/document-types/[type_code] → 타입 상세
│
├── /admin/audit-logs             → 감사 로그 목록
│   └── /admin/audit-logs/[event_id] → 이벤트 상세 (선택적)
│
├── /admin/jobs                   → 백그라운드 작업 목록
│   └── /admin/jobs/[job_id]     → 작업 상세 (선택적, 패널 우선)
│
├── /admin/indexing               → 인덱싱 작업 목록
│   └── /admin/indexing/[job_id] → 인덱싱 작업 상세 (선택적)
│
└── /admin/api-keys               → API 키 목록
    └── /admin/api-keys/integrations → 외부 연계 설정
```

---

## 2. Admin UI 라우팅 구조 정의서

### 2-1. 라우팅 테이블

| Route | 화면명 | 접근 권한 | 설명 |
|-------|--------|-----------|------|
| `/admin/dashboard` | 시스템 대시보드 | ADMIN+ | 핵심 지표, 컴포넌트 상태, 최근 오류/감사 요약 |
| `/admin/users` | 사용자 목록 | ADMIN+ | 전체 사용자 조회, 검색, 필터 |
| `/admin/users/[user_id]` | 사용자 상세 | ADMIN+ | 기본 정보, 역할/조직 매핑, 상태 변경 |
| `/admin/organizations` | 조직 목록 | ADMIN+ | 조직 목록 조회 |
| `/admin/organizations/[org_id]` | 조직 상세 | ADMIN+ | 조직 정보, 소속 사용자 목록 |
| `/admin/roles` | 역할 목록 | ADMIN+ | 역할 목록, 권한 정책 연결 현황 |
| `/admin/roles/[role_id]` | 역할 상세 | ADMIN+ | 역할 정보, 연결 정책 확인 |
| `/admin/permissions` | 권한 정책 목록 | SUPER_ADMIN | 정책 목록, 역할-정책 매트릭스 |
| `/admin/permissions/[policy_id]` | 정책 상세 | SUPER_ADMIN | 정책 내용, 편집 |
| `/admin/document-types` | DocumentType 목록 | ADMIN+ | 타입 목록, 상태 |
| `/admin/document-types/[type_code]` | DocumentType 상세 | ADMIN+ | 스키마, 플러그인 설정, 상태 변경 |
| `/admin/audit-logs` | 감사 로그 | ADMIN+ | 로그 목록, 필터, 상세 패널 |
| `/admin/jobs` | 백그라운드 작업 | ADMIN+ | 작업 목록, 상태 관리 |
| `/admin/indexing` | 인덱싱 상태 | ADMIN+ | 인덱싱 작업 목록, 재시도 |
| `/admin/api-keys` | API 키 관리 | ADMIN+ | 키 목록, 발급/폐기 |
| `/admin/api-keys/integrations` | 외부 연계 설정 | ADMIN+ | 연계 시스템 목록 |

### 2-2. 라우팅 원칙

- **Admin/User 완전 분리**: Admin 라우트는 `/admin/*`, User 라우트는 `/documents/*`, `/reviews/*` 등으로 구조적으로 분리
- **Next.js App Router**: `app/admin/layout.tsx`에서 Admin 레이아웃 격리
- **서버사이드 권한 가드**: 모든 Admin 페이지는 서버 컴포넌트에서 세션 + 역할 검증 후 렌더링
- **RESTful URL**: 자원 ID는 path param, 필터/검색은 query param

### 2-3. Query Parameter 공통 규약

| Parameter | 타입 | 설명 |
|-----------|------|------|
| `page` | number | 페이지 번호 (기본 1) |
| `limit` | number | 페이지당 항목 수 (기본 20) |
| `sort` | string | 정렬 기준 필드 |
| `order` | `asc` \| `desc` | 정렬 방향 (기본 desc) |
| `search` | string | 키워드 검색 |
| `status` | string | 상태 필터 |

---

## 3. 접근 권한 정책 문서

### 3-1. 접근 주체 정의

| 역할 | 코드 | 설명 |
|------|------|------|
| 슈퍼 관리자 | `SUPER_ADMIN` | 플랫폼 전체 관리 권한 |
| 조직 관리자 | `ORG_ADMIN` | 자신이 속한 조직 범위 내 관리 |
| 일반 사용자 | `USER`, `AUTHOR`, `REVIEWER`, `APPROVER` | Admin UI 접근 불가 |

### 3-2. 접근 제어 방식

#### (1) 서버사이드 레이아웃 가드

```
app/admin/layout.tsx
  └─ getServerSession() 호출
  └─ 세션 없음 → /login?redirect=/admin 리다이렉트
  └─ 역할 없음 또는 일반 사용자 → /403 또는 /documents 리다이렉트
  └─ ADMIN+ → Admin 레이아웃 렌더링
```

#### (2) 메뉴 노출 제어 (클라이언트)

- 역할에 따라 Sidebar 메뉴 항목 조건부 렌더링
- 노출 제어는 UX 목적. 실제 접근 보호는 서버사이드 가드가 담당

#### (3) 고위험 액션 추가 확인 정책

다음 액션은 실행 전 강화된 확인 다이얼로그 표시:

| 액션 | 위험 수준 | 추가 조치 |
|------|-----------|-----------|
| API 키 폐기 | 높음 | 사유 입력 필수 + Danger 다이얼로그 |
| 권한 정책 변경 | 높음 | 영향 범위 표시 + 사유 입력 필수 |
| 사용자 비활성화 | 중간 | 확인 다이얼로그 + 사유 입력 선택 |
| DocumentType 비활성화 | 중간 | 영향 문서 수 표시 + 확인 다이얼로그 |
| 작업 강제 종료 | 높음 | 사유 입력 필수 + Danger 다이얼로그 |

---

## 4. 역할별 메뉴 접근 매트릭스

| 메뉴 | SUPER_ADMIN | ORG_ADMIN | 비고 |
|------|:-----------:|:---------:|------|
| Dashboard | ✅ | ✅ | ORG_ADMIN은 자신의 조직 범위 데이터만 |
| Users | ✅ | ✅ | ORG_ADMIN은 자신의 조직 내 사용자만 |
| Organizations | ✅ | ✅ | ORG_ADMIN은 자신의 조직만 |
| Roles | ✅ | ✅ | ORG_ADMIN은 조회만, 편집 불가 |
| Permissions | ✅ | ❌ | SUPER_ADMIN 전용 |
| Document Types | ✅ | ❌ | SUPER_ADMIN 전용 |
| Audit Log | ✅ | ✅ | ORG_ADMIN은 자신의 조직 내 이벤트만 |
| Background Jobs | ✅ | ✅ | ORG_ADMIN은 자신의 조직 관련 작업만 |
| Indexing | ✅ | ✅ | ORG_ADMIN은 자신의 조직 문서 인덱싱만 |
| API Keys | ✅ | ✅ | ORG_ADMIN은 자신이 발급한 키만 |

### 4-1. 기능 수준 접근 제한

| 기능 | SUPER_ADMIN | ORG_ADMIN |
|------|:-----------:|:---------:|
| 사용자 상태 변경 | ✅ | ✅ (조직 내) |
| 역할 매핑 변경 | ✅ | ❌ |
| 권한 정책 편집 | ✅ | ❌ |
| DocumentType 상태 변경 | ✅ | ❌ |
| API 키 발급 | ✅ | ✅ (제한 Scope) |
| API 키 폐기 | ✅ | ✅ (자신 발급분) |
| 작업 강제 종료 | ✅ | ❌ |
| 인덱싱 일괄 재시도 | ✅ | ❌ |

---

## 5. 공통 레이아웃 구조 초안

```
┌──────────────────────────────────────────────────────────┐
│  Admin Header (높이: 56px, 고정)                          │
│  [Mimir Admin]  [운영 환경 배지]       [알림] [관리자 메뉴] │
├──────────────┬───────────────────────────────────────────┤
│              │  Breadcrumb Bar                            │
│  Admin       │  Admin > Users > {user_name}               │
│  Sidebar     ├───────────────────────────────────────────┤
│  (240px,     │                                           │
│  고정)        │  Main Content Area                        │
│              │  (페이지 본문)                              │
│  ─────────   │                                           │
│  운영         │                                           │
│  ─ Dashboard │                                           │
│  ─ Users     │                                           │
│  ─ Orgs      │                                           │
│  ─ Roles     ├───────────────────────────────────────────┤
│              │  (선택) Right Side Panel                   │
│  보안         │  목록 선택 시 상세 정보 패널               │
│  ─ Perms     │                                           │
│  ─ DocTypes  │                                           │
│  ─ AuditLog  │                                           │
│              │                                           │
│  모니터링     │                                           │
│  ─ Jobs      │                                           │
│  ─ Indexing  │                                           │
│              │                                           │
│  연계         │                                           │
│  ─ API Keys  │                                           │
└──────────────┴───────────────────────────────────────────┘
```

### 5-1. Sidebar 섹션 그룹핑

| 그룹 | 메뉴 항목 |
|------|-----------|
| 운영 (Operations) | Dashboard, Users, Organizations, Roles |
| 보안 (Security) | Permissions, Document Types, Audit Log |
| 모니터링 (Monitoring) | Background Jobs, Indexing |
| 연계 (Integration) | API Keys |

### 5-2. 레이아웃 요구사항

- **User UI와 시각적 구분**: Admin은 진한 배경 + 밀도 높은 데이터 중심 UI
- **데스크탑 우선**: 최소 해상도 1280px 기준. 모바일 대응 불필요
- **고정 Sidebar**: 스크롤 시에도 Sidebar 유지 (운영 중 항상 네비게이션 접근 가능)
- **Side Panel**: 목록 화면에서 항목 클릭 시 우측에 상세 패널 표시 (페이지 이동 최소화)
