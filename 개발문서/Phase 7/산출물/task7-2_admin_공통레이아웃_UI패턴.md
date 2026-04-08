# Task 7-2 산출물: Admin UI 공통 레이아웃 구조 및 UI 패턴

---

## 1. Admin 공통 레이아웃 구조 정의서

### 1-1. 전체 레이아웃 구성

```
┌──────────────────────────────────────────────────────────────┐
│  AdminHeader  (h:56px, position: fixed, z-index: 50)         │
│  [🔐 Mimir Admin]  [PROD 배지]    [⚠ 알림 N]  [관리자 ▾]     │
├──────────────┬───────────────────────────────────────────────┤
│              │  BreadcrumbBar  (h:40px)                       │
│  AdminSidebar│  Admin > Users > {user_name}                   │
│  (w:240px,   ├───────────────────────────────────────────────┤
│  fixed,      │                                               │
│  overflow-y: │  MainContentArea                              │
│  auto)       │  (flex-1, overflow-y: auto)                   │
│              │                                               │
│  [그룹/메뉴] │  ┌─────────────────────────────────────────┐  │
│              │  │  PageHeader  (제목 + 액션 버튼)          │  │
│              │  ├─────────────────────────────────────────┤  │
│              │  │  FilterBar  (검색바 + 필터)              │  │
│              │  ├─────────────────────────────────────────┤  │
│              │  │  DataTable  (본문 목록)                  │  │
│              │  └─────────────────────────────────────────┘  │
│              │                                               │
│              │  [선택] RightSidePanel (w:400px, overlay)     │
└──────────────┴───────────────────────────────────────────────┘
```

### 1-2. 레이아웃 변형 타입

| 타입 | 사용 화면 | SidePanel |
|------|-----------|-----------|
| `list` | 목록 화면 (Users, Jobs, AuditLog 등) | 선택 시 우측 패널 |
| `detail` | 상세 화면 (User Detail, Permission Detail 등) | 없음 (풀 화면) |
| `dashboard` | 대시보드 | 없음 (카드 그리드) |

### 1-3. 반응형 기준

| 환경 | 너비 | 처리 방식 |
|------|------|-----------|
| Desktop (Primary) | ≥ 1280px | Sidebar 고정 표시 |
| Narrow Desktop | 1024px ~ 1279px | Sidebar 접힘 (토글) |
| 그 이하 | < 1024px | 지원 불필요 (운영 콘솔) |

---

## 2. 공통 컴포넌트 목록 및 패턴 가이드

### 2-1. Layout 컴포넌트

| 컴포넌트 | 역할 | 핵심 Props |
|---------|------|-----------|
| `AdminLayout` | 전체 레이아웃 조립 | `children` |
| `AdminHeader` | 상단 헤더 | `env: 'production' \| 'staging'` |
| `AdminSidebar` | 좌측 네비게이션 | `currentPath, userRole` |
| `Breadcrumb` | 현재 위치 안내 | `items: {label, href}[]` |
| `PageHeader` | 페이지 제목 + 액션 | `title, description?, actions?` |
| `RightSidePanel` | 상세 정보 패널 | `open, onClose, children` |

### 2-2. Data Display 컴포넌트

| 컴포넌트 | 역할 | 핵심 Props |
|---------|------|-----------|
| `DataTable` | 운영 목록 테이블 | `columns, data, onRowClick, loading` |
| `StatusBadge` | 상태 표시 배지 | `status, variant` |
| `MetricCard` | 지표 카드 (대시보드) | `title, value, status, href` |
| `ComponentHealth` | 컴포넌트 상태 표시 | `name, status` |

### 2-3. Controls 컴포넌트

| 컴포넌트 | 역할 | 핵심 Props |
|---------|------|-----------|
| `SearchBar` | 키워드 검색 입력 | `value, onChange, placeholder` |
| `FilterPanel` | 다중 조건 필터 | `filters, values, onChange` |
| `ActiveFilters` | 활성 필터 태그 표시 | `filters, onRemove, onClear` |
| `Pagination` | 페이지 네비게이션 | `page, totalPages, onChange` |
| `ConfirmDialog` | 일반 확인 다이얼로그 | `open, title, message, onConfirm, onCancel` |
| `DangerDialog` | 위험 액션 확인 다이얼로그 | `open, title, target, impact?, requireReason, onConfirm` |

### 2-4. Feedback 컴포넌트

| 컴포넌트 | 역할 | 핵심 Props |
|---------|------|-----------|
| `LoadingSkeleton` | 로딩 중 스켈레톤 | `type: 'table' \| 'card' \| 'detail'` |
| `EmptyState` | 데이터 없음 상태 | `message, action?` |
| `ErrorState` | 오류 상태 | `message, onRetry?` |
| `PermissionDenied` | 권한 없음 안내 | `backHref?` |
| `Toast` | 작업 결과 알림 | `type: 'success' \| 'error' \| 'warning'` |

---

## 3. 상태 배지 및 경고 표시 규칙

### 3-1. StatusBadge 색상 규칙

| 상태값 | 색상 | 배지 스타일 | 대상 |
|--------|------|-------------|------|
| `ACTIVE`, `SUCCESS`, `COMPLETED`, `HEALTHY` | 녹색 | 초록 배경 + 짙은 녹색 텍스트 | 정상/활성 상태 |
| `INACTIVE`, `DISABLED`, `CANCELLED` | 회색 | 회색 배경 + 회색 텍스트 | 비활성/중단 |
| `PENDING`, `RUNNING` | 파란색 | 파란 배경 + 짙은 파란 텍스트 | 진행 중 (RUNNING은 펄스 애니메이션) |
| `WARNING`, `RETRYING`, `DEGRADED`, `SKIPPED` | 노란색 | 노란 배경 + 갈색 텍스트 | 경고/재시도 |
| `FAILED`, `ERROR`, `REVOKED`, `DOWN`, `CRITICAL` | 빨간색 | 빨간 배경 + 짙은 빨간 텍스트 | 실패/오류 |
| `EXPIRED`, `UNKNOWN` | 회색 (점선) | 회색 배경 + 이탤릭 텍스트 | 만료/미확인 |

### 3-2. 이상 상태 강조 규칙

- 실패/오류 건수가 0 이상인 경우 해당 행 전체에 옅은 빨간 배경 적용
- 장기 실행 작업(임계값 초과)에는 경고 아이콘(`⚠`) 함께 표시
- CRITICAL 감사 이벤트가 있는 경우 해당 행을 상단으로 정렬하거나 붉은 테두리로 강조

---

## 4. 확인 다이얼로그 UX 정의서

### 4-1. 일반 확인 다이얼로그 (ConfirmDialog)

적용 대상: 취소 가능하거나 영향이 제한적인 변경

```
┌────────────────────────────────────┐
│  사용자 비활성화                    │
│                                    │
│  {user_name}을(를) 비활성화하면    │
│  해당 사용자는 더 이상 로그인할    │
│  수 없습니다.                      │
│                                    │
│  변경 사유 (선택)                   │
│  [____________________________]    │
│                                    │
│             [취소]  [비활성화]      │
└────────────────────────────────────┘
```

### 4-2. 위험 액션 다이얼로그 (DangerDialog)

적용 대상: 취소 불가하거나 광범위한 영향을 주는 변경

```
┌────────────────────────────────────┐
│  ⚠️  위험: API 키 폐기              │
│                                    │
│  대상: {key_name} (mimir_ab12****) │
│                                    │
│  이 작업은 취소할 수 없습니다.     │
│  현재 이 키를 사용하는 시스템의    │
│  연결이 즉시 차단됩니다.           │
│                                    │
│  영향: 외부 시스템 3개 연계 중단   │
│                                    │
│  폐기 사유 (필수)                   │
│  [____________________________]    │
│                                    │
│        [취소]  [🗑 키 폐기 (위험)]  │
└────────────────────────────────────┘
```

**DangerDialog 적용 기준:**

| 액션 | 이유 표시 | 사유 입력 | 영향 표시 |
|------|:---------:|:---------:|:---------:|
| API 키 폐기 | 필수 | 필수 | 필수 |
| 권한 정책 광범위 변경 | 필수 | 필수 | 필수 (영향 사용자 수) |
| 작업 강제 종료 | 필수 | 필수 | 필수 |
| DocumentType 비활성화 | 필수 | 선택 | 필수 (영향 문서 수) |
| 사용자 비활성화 | 선택 | 선택 | 선택 |

---

## 5. 공통 DataTable 패턴 정의

### 5-1. 테이블 구성 요소

```
┌─────────────────────────────────────────────────────────────┐
│  SearchBar + FilterPanel + ActiveFilters                     │
├───────────────────────────────────────────────────────────────│
│  [선택] ☐  │ 컬럼1 ↑↓  │ 컬럼2 ↑↓  │ 상태   │ 날짜   │ 액션 │
├────────────┼───────────┼───────────┼────────┼────────┼──────┤
│  ☐         │ 값1       │ 값2       │ 🟢ACTIVE│ 날짜  │ ...  │
│  (hover 시 배경 강조, 클릭 시 RightSidePanel 열림)           │
├────────────┴───────────┴───────────┴────────┴────────┴──────┤
│  총 {N}건  ←  1  2  3  ...  →   [페이지당 20 ▾]             │
└─────────────────────────────────────────────────────────────┘
```

### 5-2. 테이블 상태 표현

| 상태 | 처리 방식 |
|------|-----------|
| 로딩 중 | `LoadingSkeleton` - 행 3~5개 스켈레톤 |
| 데이터 없음 | `EmptyState` - 아이콘 + 메시지 + 조치 유도 버튼 |
| 오류 | `ErrorState` - 오류 메시지 + 재시도 버튼 |
| 검색 결과 없음 | "'{keyword}'에 대한 결과가 없습니다" + 필터 초기화 버튼 |

### 5-3. 행 클릭 동작

| 화면 | 행 클릭 결과 |
|------|-------------|
| Users, Jobs, AuditLog, Indexing | RightSidePanel 열림 |
| Organizations, Roles, Permissions, DocumentTypes, API Keys | 상세 페이지로 이동 |

---

## 6. 검색 및 필터 패턴

### 6-1. 검색바

- 상단 위치, 디바운스 300ms 적용
- 검색 중 로딩 스피너 표시
- 엔터 또는 검색 버튼 클릭으로 실행

### 6-2. 필터 패널

- 필터 버튼 클릭 시 드롭다운 또는 인라인 확장
- 다중 선택 가능 (체크박스 방식)
- 날짜 범위: 오늘 / 최근 7일 / 최근 30일 / 직접 입력

### 6-3. 활성 필터 표시

```
[상태: FAILED ×]  [기간: 최근 7일 ×]  [전체 초기화]
```

---

## 7. AdminHeader 상세 구성

| 영역 | 요소 | 설명 |
|------|------|------|
| 좌측 | Mimir Admin 로고 | `/admin/dashboard`로 이동 |
| 좌측 | 운영 환경 배지 | `PROD` (빨간), `STAGING` (노란), `DEV` (회색) |
| 우측 | 알림 아이콘 | 미확인 알림 수 배지 표시 |
| 우측 | 관리자 정보 | 이름 + 역할 표시, 클릭 시 드롭다운 (프로필, 로그아웃) |
