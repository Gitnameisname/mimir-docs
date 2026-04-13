# Task 6-2 산출물: 공통 UI 프레임워크 및 레이아웃 컴포넌트

---

## 1. 레이아웃 구조 정의서

### 1-1. 기본 레이아웃 구성

```
┌──────────────────────────────────────────────────────────────┐
│  Header (높이: 56px, 고정)                                     │
│  [Mimir 로고]  [Sidebar Toggle]    [검색] [알림] [사용자 메뉴] │
├─────────────┬────────────────────────────────────────────────┤
│             │                                                  │
│  Sidebar    │  Main Content Area                              │
│  (240px,    │  (가변 너비, 스크롤 가능)                        │
│  접기 가능) │                                                  │
│             │  ┌──────────────────────────────────────────┐   │
│  - Docs     │  │  Page Header (제목 + 액션 버튼)           │   │
│  - My Tasks │  ├──────────────────────────────────────────┤   │
│  - Recents  │  │  Action Bar (검색 + 필터, 선택적)         │   │
│             │  ├──────────────────────────────────────────┤   │
│             │  │  Content Body                             │   │
│             │  └──────────────────────────────────────────┘   │
├─────────────┴──────────────────────────────────┬─────────────┤
│  (문서 상세/편집 화면에서만)                     │ Right Panel │
│  구조 트리 패널 (좌측 통합) or 인라인            │ (RAG, 메타) │
└────────────────────────────────────────────────┴─────────────┘
```

### 1-2. 레이아웃 변형

| 레이아웃 타입 | 사용 화면 | Sidebar | Right Panel |
|--------------|----------|---------|-------------|
| `default` | 문서 목록, 검토 목록 | 표시 | 없음 |
| `document-view` | 문서 상세 보기 | 표시 | 구조 트리 (토글) |
| `document-edit` | 문서 편집 | 표시 | 없음 (편집기 집중) |
| `fullscreen` | 버전 상세 비교 (Phase 9) | 접힘 | 없음 |

### 1-3. 반응형 기준점 (Breakpoints)

| 구간 | 너비 | Sidebar 처리 |
|------|------|-------------|
| Desktop | ≥ 1280px | 항상 노출 |
| Tablet | 768px ~ 1279px | 기본 접힘, 토글 가능 |
| Mobile | < 768px | 오버레이 드로어 방식 |

> **원칙**: 데스크탑 앱과 웹 앱 모두 동일 컴포넌트 사용. 반응형 처리는 CSS 수준에서만.

---

## 2. 공통 컴포넌트 목록

### 2-1. Layout 컴포넌트

| 컴포넌트 | 역할 | Props 핵심 |
|---------|------|-----------|
| `AppLayout` | 전체 레이아웃 조립 | `variant: LayoutVariant` |
| `Header` | 글로벌 헤더 | `title?, actions?` |
| `Sidebar` | 좌측 네비게이션 | `collapsed, onToggle` |
| `SidebarNav` | 메뉴 항목 렌더링 | `items: NavItem[]` |
| `RightPanel` | 우측 보조 패널 | `open, title, children` |
| `PageContainer` | 메인 콘텐츠 래퍼 | `maxWidth?, padding?` |

### 2-2. Page 공통 컴포넌트

| 컴포넌트 | 역할 | Props 핵심 |
|---------|------|-----------|
| `PageHeader` | 페이지 제목 + 설명 + 액션 | `title, description?, actions?` |
| `ActionBar` | 검색/필터/정렬 바 | `search?, filters?, sort?, actions?` |
| `Breadcrumb` | 현재 위치 표시 | `items: BreadcrumbItem[]` |
| `TabBar` | 탭 네비게이션 | `tabs, activeTab, onTabChange` |

### 2-3. 피드백 컴포넌트

| 컴포넌트 | 역할 | Props 핵심 |
|---------|------|-----------|
| `LoadingSpinner` | 인라인 로딩 표시 | `size: sm/md/lg` |
| `SkeletonBlock` | 콘텐츠 로딩 플레이스홀더 | `width, height, rows?` |
| `EmptyState` | 데이터 없음 안내 | `title, description, action?` |
| `ErrorState` | 오류 안내 | `title, description, retry?` |
| `PermissionDenied` | 접근 권한 없음 | `message?, backTo?` |
| `Toast` | 단기 알림 메시지 | `type: success/error/warning/info, message` |
| `ConfirmDialog` | 위험 액션 확인 | `title, message, onConfirm, onCancel, danger?` |

### 2-4. 상태 배지 / 태그 컴포넌트

| 컴포넌트 | 역할 | Props 핵심 |
|---------|------|-----------|
| `WorkflowStatusBadge` | 워크플로 상태 배지 | `status: WorkflowStatus` |
| `DocumentTypeBadge` | 문서 유형 태그 | `type: string` |
| `RoleBadge` | 역할 표시 | `role: UserRole` |

### 2-5. 폼 / 입력 컴포넌트

| 컴포넌트 | 역할 |
|---------|------|
| `TextInput` | 단일 라인 입력 |
| `TextArea` | 멀티라인 입력 |
| `Select` | 드롭다운 선택 |
| `SearchInput` | 검색 전용 입력 (clear 버튼 포함) |
| `FormField` | 레이블 + 입력 + 에러 메시지 래퍼 |

### 2-6. 버튼 컴포넌트

| 컴포넌트 | 역할 | variant |
|---------|------|---------|
| `Button` | 범용 버튼 | `primary / secondary / ghost / danger` |
| `IconButton` | 아이콘 전용 버튼 | `size: sm/md/lg` |
| `ActionMenu` | 드롭다운 액션 묶음 | `items: ActionItem[]` |

---

## 3. 상태 표현 가이드

### 3-1. 페이지 레벨 상태

| 상태 | 컴포넌트 | 적용 범위 |
|------|---------|----------|
| 초기 로딩 | `SkeletonBlock` | 목록, 상세 페이지 첫 로드 |
| 데이터 없음 | `EmptyState` | 목록 결과 0건, 버전 없음 등 |
| 오류 발생 | `ErrorState` | API 호출 실패, 네트워크 오류 |
| 권한 없음 | `PermissionDenied` | 403 / 권한 부족 |
| 찾을 수 없음 | `ErrorState` (404 변형) | 존재하지 않는 문서 접근 |

### 3-2. 인라인 상태

| 상태 | 처리 방식 |
|------|----------|
| 버튼 클릭 후 처리 중 | 버튼 내 `LoadingSpinner` + 비활성화 |
| 저장 중 | Action Bar에 "저장 중..." 텍스트 |
| 저장 완료 | Toast (success, 2초) |
| 저장 실패 | Toast (error, 수동 닫기) |
| 폼 유효성 오류 | 필드 하단 인라인 오류 메시지 |

### 3-3. WorkflowStatus 배지 색상 체계

| 상태 | 레이블 | 색상 계열 |
|------|--------|----------|
| `DRAFT` | 초안 | Gray |
| `IN_REVIEW` | 검토 중 | Blue |
| `APPROVED` | 승인됨 | Green |
| `PUBLISHED` | 발행됨 | Teal |
| `REJECTED` | 반려됨 | Red |
| `ARCHIVED` | 보관됨 | Dark Gray |

---

## 4. UI 패턴 정의 문서

### 4-1. Confirm Dialog 패턴

```
위험 액션(삭제, 반려, 강제 전이 등) 수행 전 반드시 ConfirmDialog 표시

구성:
  - 제목: "[액션명] 하시겠습니까?"
  - 설명: 수행 결과와 되돌릴 수 없음 안내
  - 버튼: [취소] [확인 (danger 스타일)]

예외: 자동 저장, 검토 요청 등 저위험 액션은 Dialog 불필요
```

### 4-2. Unsaved Changes 경고 패턴

```
편집 화면에서 저장하지 않고 이탈 시도 시:
  - 브라우저 beforeunload 이벤트 처리
  - Next.js router.beforePopState 처리
  - ConfirmDialog: "저장하지 않은 변경 사항이 있습니다. 떠나시겠습니까?"
  - 버튼: [계속 편집] [저장 없이 이탈]
```

### 4-3. Toast 알림 패턴

```
위치: 화면 우측 하단
표시 시간:
  - success: 2초 후 자동 닫힘
  - info: 3초 후 자동 닫힘
  - warning: 수동 닫기 (또는 5초)
  - error: 수동 닫기

동시 최대: 3개 (초과 시 오래된 것 제거)
```

### 4-4. Page Transition 패턴

```
페이지 전환 시:
  - Main Content 영역만 교체 (Header/Sidebar 유지)
  - Next.js App Router의 loading.tsx 활용
  - 최초 로드: SkeletonBlock으로 콘텐츠 플레이스홀더 표시
  - 이후 로드: 이전 데이터 유지 + 새 데이터 교체 (SWR/TanStack Query 활용)
```

### 4-5. Action Button 배치 원칙

```
주요 액션 버튼 위치:
  - PageHeader 우측: 해당 페이지의 주요 액션 (새 문서, 편집 완료 등)
  - 문서 상세 Header: 문서 상태 전이 액션 (검토 요청, 승인, 반려 등)
  - 목록 행 우측: 빠른 접근 액션 (상세 보기, 편집 등) — hover 시 노출 가능

버튼 우선순위 순서 (좌→우):
  Secondary Action → Primary Action
  (Danger Action은 별도 위치 또는 ActionMenu 하위)
```

---

## 5. 컴포넌트 디렉토리 구조 제안

```
frontend/
└── src/
    └── components/
        ├── layout/
        │   ├── AppLayout.tsx
        │   ├── Header.tsx
        │   ├── Sidebar.tsx
        │   └── RightPanel.tsx
        ├── page/
        │   ├── PageHeader.tsx
        │   ├── ActionBar.tsx
        │   └── Breadcrumb.tsx
        ├── feedback/
        │   ├── LoadingSpinner.tsx
        │   ├── SkeletonBlock.tsx
        │   ├── EmptyState.tsx
        │   ├── ErrorState.tsx
        │   ├── PermissionDenied.tsx
        │   ├── Toast.tsx
        │   └── ConfirmDialog.tsx
        ├── badge/
        │   ├── WorkflowStatusBadge.tsx
        │   └── DocumentTypeBadge.tsx
        ├── form/
        │   ├── TextInput.tsx
        │   ├── TextArea.tsx
        │   ├── Select.tsx
        │   ├── SearchInput.tsx
        │   └── FormField.tsx
        └── button/
            ├── Button.tsx
            ├── IconButton.tsx
            └── ActionMenu.tsx
```
