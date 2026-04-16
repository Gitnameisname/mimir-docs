# FG6.1 작업지시서 1: Admin UI 통합 레이아웃 + 좌측 네비게이션 + 공통 컴포넌트 라이브러리

## 1. 작업 목적
Mimir S2 프로젝트의 관리자 대시보드(Phase 6)에 필요한 통합 레이아웃, 좌측 네비게이션 구조, 및 공통 Admin UI 컴포넌트 라이브러리를 구축하여, FG6.2~FG6.4에서 재사용 가능한 기반 인프라를 제공합니다.

## 2. 작업 범위

### 포함 사항
- AdminLayout 컴포넌트: 좌측 네비게이션 + 헤더 + 메인 콘텐츠 영역 + 푸터
- 좌측 네비게이션 (SideNav) 구조:
  - S1 영역: 사용자 및 역할 관리, 시스템 설정, 감사 로그, API 키 관리
  - S2 영역: AI 플랫폼 관리, 에이전트·Scope 관리, 평가·추출 시스템
  - 접기/펼치기 토글 (collapse state)
  - 현재 경로 강조 (active nav item)
  - 마지막 방문 섹션 기억 (localStorage)
- 권한 기반 메뉴 조건부 렌더링 (admin 역할만 표시, S2 원칙 ⑥)
- 공통 Admin UI 컴포넌트 라이브러리:
  - AdminTable: TanStack Table 래퍼 (정렬, 필터, 페이지네이션)
  - AdminModal: 폼 입력, 확인 다이얼로그
  - AdminForm: React Hook Form + Zod 검증 래퍼
  - StatusBadge, LoadingSkeleton, EmptyState, ErrorState 컴포넌트
  - Toast 알림 (sonner 라이브러리)
- Admin 라우팅 정의:
  - `/admin/providers` (FG6.2)
  - `/admin/capabilities` (FG6.2)
  - `/admin/prompts` (FG6.3)
  - `/admin/usage` (FG6.4)
- Zustand adminStore: 현재 사용자, 권한 캐시, UI 상태 (접기/펼치기)
- 반응형 디자인 (640px, 768px, 1024px+)
- WCAG 2.1 AA 접근성 준수

### 제외 사항
- 개별 관리 페이지 구현 (모델/프로바이더, 프롬프트, 비용 등) → FG6.2~6.4에서 수행
- FG6.2, FG6.3 등 다른 기능 그룹의 페이지 개발
- 외부 CDN 의존 (S2 원칙 ⑦ 폐쇄망)

## 3. 선행 조건
- Phase 1~5 백엔드 API 완성 및 문서화
- Next.js 16.2.2, React 19, TypeScript 환경 구축 완료
- TanStack Table, React Hook Form, Zod, recharts, sonner 라이브러리 설치 완료
- 기존 인증/권한 시스템 (authStore, useAuth hook) 구현 완료
- API wrapper (unwrapEnvelope) 구현 완료

## 4. 주요 작업 항목

### 4-1 AdminLayout 컴포넌트 구현
**목표**: 좌측 네비게이션, 헤더, 메인 콘텐츠 영역, 푸터를 통합한 레이아웃 컴포넌트 구축

**세부 작업**:
- `app/admin/layout.tsx` 생성: 모든 admin 페이지의 부모 레이아웃
- `components/admin/AdminLayout.tsx` 구현:
  ```typescript
  interface AdminLayoutProps {
    children: React.ReactNode;
    currentSection?: string; // 현재 섹션 표시용
  }
  
  export function AdminLayout({ children, currentSection }: AdminLayoutProps) {
    return (
      <div className="flex h-screen bg-gray-50">
        {/* 좌측 네비게이션 */}
        <SideNav currentSection={currentSection} />
        
        {/* 메인 콘텐츠 영역 */}
        <div className="flex-1 flex flex-col">
          {/* 헤더 */}
          <AdminHeader />
          
          {/* 메인 콘텐츠 */}
          <main className="flex-1 overflow-auto p-6">
            {children}
          </main>
          
          {/* 푸터 */}
          <AdminFooter />
        </div>
      </div>
    );
  }
  ```
- 헤더: 로고, 현재 사용자 정보, 알림 아이콘, 로그아웃 버튼
- 푸터: 버전 정보, 지원 링크, 저작권
- 반응형: 768px 이상에서 좌측 네비게이션 고정, 768px 미만에서는 모바일 메뉴로 변경

### 4-2 좌측 네비게이션 (SideNav) 구현
**목표**: S1/S2 원칙을 반영한 권한 기반 네비게이션 메뉴 구축

**세부 작업**:
- `components/admin/SideNav.tsx` 구현:
  ```typescript
  interface NavSection {
    title: string;
    icon: React.ReactNode;
    items: NavItem[];
    minRoleRequired?: UserRole; // S2: 하드코딩 금지, Scope Profile 사용
  }
  
  interface NavItem {
    label: string;
    href: string;
    icon: React.ReactNode;
  }
  
  const NAV_CONFIG: NavSection[] = [
    {
      title: "S1: 기본 설정",
      items: [
        { label: "사용자 및 역할", href: "/admin/users", icon: <Users /> },
        { label: "시스템 설정", href: "/admin/settings", icon: <Settings /> },
        { label: "감사 로그", href: "/admin/audit", icon: <History /> },
        { label: "API 키", href: "/admin/api-keys", icon: <Key /> },
      ],
    },
    {
      title: "S2: AI 플랫폼 & 에이전트",
      items: [
        { label: "프로바이더 관리", href: "/admin/providers", icon: <Zap /> },
        { label: "Capabilities", href: "/admin/capabilities", icon: <Activity /> },
        { label: "프롬프트 관리", href: "/admin/prompts", icon: <FileText /> },
        { label: "사용량 현황", href: "/admin/usage", icon: <BarChart3 /> },
        { label: "에이전트 관리", href: "/admin/agents", icon: <Bot /> },
        { label: "Scope 관리", href: "/admin/scopes", icon: <Shield /> },
        { label: "평가 및 추출", href: "/admin/evaluation", icon: <CheckCircle /> },
      ],
    },
  ];
  
  export function SideNav({ currentSection }: { currentSection?: string }) {
    const [isCollapsed, setIsCollapsed] = useAdminStore(
      (state) => [state.sideNavCollapsed, state.setSideNavCollapsed]
    );
    const { user } = useAuth();
    
    // S2 원칙 ⑥: 권한 기반 메뉴 렌더링 (하드코딩 금지)
    const visibleSections = NAV_CONFIG.filter(
      (section) => !section.minRoleRequired || hasRole(user, section.minRoleRequired)
    );
    
    return (
      <nav className={`
        bg-white border-r border-gray-200 shadow-sm transition-all duration-300
        ${isCollapsed ? "w-20" : "w-64"}
      `}>
        {/* 로고 & 토글 */}
        <div className="p-4 border-b border-gray-200 flex items-center justify-between">
          {!isCollapsed && <span className="font-bold">Mimir Admin</span>}
          <button
            onClick={() => setIsCollapsed(!isCollapsed)}
            className="p-2 hover:bg-gray-100 rounded"
          >
            <ChevronLeft className={isCollapsed ? "rotate-180" : ""} />
          </button>
        </div>
        
        {/* 네비게이션 섹션 */}
        <div className="overflow-y-auto h-[calc(100vh-120px)]">
          {visibleSections.map((section) => (
            <div key={section.title} className="py-4">
              {!isCollapsed && (
                <h3 className="px-4 text-xs font-semibold text-gray-500 uppercase tracking-wider">
                  {section.title}
                </h3>
              )}
              <ul className="mt-2 space-y-1">
                {section.items.map((item) => (
                  <li key={item.href}>
                    <Link
                      href={item.href}
                      className={`
                        flex items-center gap-3 px-4 py-2 rounded-lg
                        transition-colors duration-200
                        ${
                          currentSection === item.href
                            ? "bg-blue-50 text-blue-600 border-r-4 border-blue-600"
                            : "text-gray-700 hover:bg-gray-100"
                        }
                      `}
                      title={isCollapsed ? item.label : undefined}
                    >
                      <span className="flex-shrink-0">{item.icon}</span>
                      {!isCollapsed && <span>{item.label}</span>}
                    </Link>
                  </li>
                ))}
              </ul>
            </div>
          ))}
        </div>
      </nav>
    );
  }
  ```
- localStorage에 collapse 상태 저장: `adminStore.setSideNavCollapsed()`
- 현재 경로 강조 (usePathname() 활용)
- 권한 기반 조건부 렌더링: Scope Profile에서 권한 검색 (하드코딩 금지)

### 4-3 공통 Admin UI 컴포넌트 라이브러리 구현
**목표**: FG6.2~6.4에서 재사용 가능한 컴포넌트 모음 구축

**세부 작업**:

#### 4-3-1 AdminTable 컴포넌트
- `components/admin/AdminTable.tsx`: TanStack Table 래퍼
  ```typescript
  interface AdminTableProps<T> {
    columns: ColumnDef<T>[];
    data: T[];
    isLoading?: boolean;
    pageSize?: number;
    onRowClick?: (row: T) => void;
    sortable?: boolean;
    filterable?: boolean;
  }
  
  export function AdminTable<T>({
    columns,
    data,
    isLoading = false,
    pageSize = 10,
    sortable = true,
    filterable = true,
  }: AdminTableProps<T>) {
    const [sorting, setSorting] = useState<SortingState>([]);
    const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([]);
    const [pagination, setPagination] = useState({ pageIndex: 0, pageSize });
    
    const table = useReactTable({
      data,
      columns,
      getCoreRowModel: getCoreRowModel(),
      getSortedRowModel: getSortedRowModel(),
      getFilteredRowModel: getFilteredRowModel(),
      getPaginationRowModel: getPaginationRowModel(),
      onSortingChange: setSorting,
      onColumnFiltersChange: setColumnFilters,
      onPaginationChange: setPagination,
      state: { sorting, columnFilters, pagination },
    });
    
    if (isLoading) return <LoadingSkeleton rows={pageSize} />;
    if (data.length === 0) return <EmptyState message="데이터가 없습니다." />;
    
    return (
      <div className="space-y-4">
        {/* 필터 바 (선택) */}
        {filterable && (
          <div className="flex gap-4">
            {/* 컬럼별 필터 UI */}
          </div>
        )}
        
        {/* 테이블 */}
        <div className="border border-gray-200 rounded-lg overflow-hidden">
          <table className="w-full">
            <thead className="bg-gray-50 border-b border-gray-200">
              {table.getHeaderGroups().map((headerGroup) => (
                <tr key={headerGroup.id}>
                  {headerGroup.headers.map((header) => (
                    <th key={header.id} className="px-6 py-3 text-left text-sm font-semibold text-gray-700">
                      {header.isPlaceholder ? null : (
                        <div
                          onClick={header.column.getToggleSortingHandler()}
                          className={sortable ? "cursor-pointer select-none" : ""}
                        >
                          {flexRender(header.column.columnDef.header, header.getContext())}
                        </div>
                      )}
                    </th>
                  ))}
                </tr>
              ))}
            </thead>
            <tbody>
              {table.getRowModel().rows.map((row) => (
                <tr
                  key={row.id}
                  className="border-b border-gray-200 hover:bg-gray-50 transition-colors"
                >
                  {row.getVisibleCells().map((cell) => (
                    <td key={cell.id} className="px-6 py-4 text-sm text-gray-700">
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </td>
                  ))}
                </tr>
              ))}
            </tbody>
          </table>
        </div>
        
        {/* 페이지네이션 */}
        <div className="flex items-center justify-between">
          <p className="text-sm text-gray-600">
            {pagination.pageIndex + 1} / {table.getPageCount()} 페이지
          </p>
          <div className="flex gap-2">
            <button
              onClick={() => table.previousPage()}
              disabled={!table.getCanPreviousPage()}
              className="px-4 py-2 border border-gray-300 rounded-lg text-sm font-medium text-gray-700 hover:bg-gray-50 disabled:opacity-50"
            >
              이전
            </button>
            <button
              onClick={() => table.nextPage()}
              disabled={!table.getCanNextPage()}
              className="px-4 py-2 border border-gray-300 rounded-lg text-sm font-medium text-gray-700 hover:bg-gray-50 disabled:opacity-50"
            >
              다음
            </button>
          </div>
        </div>
      </div>
    );
  }
  ```

#### 4-3-2 AdminModal 컴포넌트
- `components/admin/AdminModal.tsx`:
  ```typescript
  interface AdminModalProps {
    isOpen: boolean;
    onClose: () => void;
    title: string;
    description?: string;
    children: React.ReactNode;
    onSubmit?: () => void;
    submitLabel?: string;
    cancelLabel?: string;
    isLoading?: boolean;
  }
  
  export function AdminModal({
    isOpen,
    onClose,
    title,
    description,
    children,
    onSubmit,
    submitLabel = "저장",
    cancelLabel = "취소",
    isLoading = false,
  }: AdminModalProps) {
    if (!isOpen) return null;
    
    return (
      <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
        <div className="bg-white rounded-lg shadow-xl max-w-md w-full mx-4">
          {/* 헤더 */}
          <div className="border-b border-gray-200 px-6 py-4">
            <h2 className="text-lg font-semibold text-gray-900">{title}</h2>
            {description && <p className="text-sm text-gray-600 mt-1">{description}</p>}
          </div>
          
          {/* 콘텐츠 */}
          <div className="px-6 py-4">{children}</div>
          
          {/* 푸터 */}
          <div className="border-t border-gray-200 px-6 py-4 flex justify-end gap-3">
            <button
              onClick={onClose}
              disabled={isLoading}
              className="px-4 py-2 border border-gray-300 rounded-lg text-sm font-medium text-gray-700 hover:bg-gray-50 disabled:opacity-50"
            >
              {cancelLabel}
            </button>
            {onSubmit && (
              <button
                onClick={onSubmit}
                disabled={isLoading}
                className="px-4 py-2 bg-blue-600 text-white rounded-lg text-sm font-medium hover:bg-blue-700 disabled:opacity-50 flex items-center gap-2"
              >
                {isLoading && <Loader className="animate-spin w-4 h-4" />}
                {submitLabel}
              </button>
            )}
          </div>
        </div>
      </div>
    );
  }
  ```

#### 4-3-3 AdminForm 컴포넌트
- `components/admin/AdminForm.tsx`: React Hook Form + Zod 래퍼
  ```typescript
  interface AdminFormProps<T extends FieldValues> {
    schema: ZodSchema;
    onSubmit: (data: T) => Promise<void> | void;
    children: (form: UseFormReturn<T>) => React.ReactNode;
    defaultValues?: T;
    isLoading?: boolean;
  }
  
  export function AdminForm<T extends FieldValues>({
    schema,
    onSubmit,
    children,
    defaultValues,
    isLoading = false,
  }: AdminFormProps<T>) {
    const form = useForm<T>({
      resolver: zodResolver(schema),
      defaultValues,
    });
    
    return (
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        {children(form)}
      </form>
    );
  }
  ```

#### 4-3-4 상태 표시 컴포넌트들
- `components/admin/StatusBadge.tsx`:
  ```typescript
  interface StatusBadgeProps {
    status: "active" | "inactive" | "error" | "pending";
    label: string;
  }
  
  export function StatusBadge({ status, label }: StatusBadgeProps) {
    const statusConfig = {
      active: "bg-green-100 text-green-800",
      inactive: "bg-gray-100 text-gray-800",
      error: "bg-red-100 text-red-800",
      pending: "bg-yellow-100 text-yellow-800",
    };
    
    return (
      <span className={`px-3 py-1 rounded-full text-sm font-medium ${statusConfig[status]}`}>
        {label}
      </span>
    );
  }
  ```

- `components/admin/LoadingSkeleton.tsx`: 테이블 행 로딩 상태
- `components/admin/EmptyState.tsx`: 데이터 없음 상태
- `components/admin/ErrorState.tsx`: 오류 상태

#### 4-3-5 Toast 알림 통합
- `hooks/useAdminToast.ts`: sonner를 활용한 toast 래퍼
  ```typescript
  export function useAdminToast() {
    return {
      success: (message: string) => toast.success(message),
      error: (message: string) => toast.error(message),
      loading: (message: string) => toast.loading(message),
      promise: <T,>(promise: Promise<T>, messages: ToastMessages) =>
        toast.promise(promise, messages),
    };
  }
  ```

### 4-4 Admin 라우팅 정의
**목표**: Admin 섹션의 라우팅 구조 정의

**세부 작업**:
- `app/admin/layout.tsx`: Admin 레이아웃 적용
- `app/admin/page.tsx`: Admin 대시보드 홈 (리디렉트 또는 개요)
- `app/admin/providers/page.tsx`: FG6.2에서 구현 예정
- `app/admin/capabilities/page.tsx`: FG6.2에서 구현 예정
- `app/admin/prompts/page.tsx`: FG6.3에서 구현 예정
- `app/admin/usage/page.tsx`: FG6.4에서 구현 예정
- 라우팅 미들웨어: `/admin/*` 접근 시 admin 역할 확인

### 4-5 Zustand adminStore 구현
**목표**: Admin UI 상태 관리

**세부 작업**:
- `store/adminStore.ts`:
  ```typescript
  interface AdminStore {
    // UI 상태
    sideNavCollapsed: boolean;
    setSideNavCollapsed: (collapsed: boolean) => void;
    
    // 사용자 캐시
    currentUser: User | null;
    setCurrentUser: (user: User) => void;
    
    // 권한 캐시
    userPermissions: Permission[];
    setUserPermissions: (permissions: Permission[]) => void;
    
    // 마지막 방문 섹션
    lastSection: string;
    setLastSection: (section: string) => void;
  }
  
  export const useAdminStore = create<AdminStore>((set) => ({
    sideNavCollapsed: localStorage.getItem("adminSideNavCollapsed") === "true",
    setSideNavCollapsed: (collapsed) => {
      localStorage.setItem("adminSideNavCollapsed", String(collapsed));
      set({ sideNavCollapsed: collapsed });
    },
    
    currentUser: null,
    setCurrentUser: (user) => set({ currentUser: user }),
    
    userPermissions: [],
    setUserPermissions: (permissions) => set({ userPermissions: permissions }),
    
    lastSection: localStorage.getItem("adminLastSection") || "/admin/providers",
    setLastSection: (section) => {
      localStorage.setItem("adminLastSection", section);
      set({ lastSection: section });
    },
  }));
  ```

### 4-6 반응형 디자인 및 접근성
**목표**: 데스크탑/태블릿/모바일 호환성 및 WCAG 2.1 AA 준수

**세부 작업**:
- Tailwind CSS 반응형 클래스 적용 (640px, 768px, 1024px 브레이크포인트)
- 좌측 네비게이션: 768px 미만에서 햄버거 메뉴로 변환
- aria-label, aria-expanded 등 시맨틱 속성 추가
- 키보드 네비게이션 지원 (Tab, Enter, ESC)
- 색상 대비 확인 (WCAG AA 최소 4.5:1)
- 폰트 크기 최소 16px

## 5. 산출물

### 필수 산출물
1. AdminLayout, SideNav, 공통 컴포넌트 소스 코드
   - `app/admin/layout.tsx`
   - `components/admin/AdminLayout.tsx`
   - `components/admin/SideNav.tsx`
   - `components/admin/AdminTable.tsx`
   - `components/admin/AdminModal.tsx`
   - `components/admin/AdminForm.tsx`
   - `components/admin/StatusBadge.tsx`, `LoadingSkeleton.tsx`, `EmptyState.tsx`, `ErrorState.tsx`
   - `hooks/useAdminToast.ts`
   - `store/adminStore.ts`

2. Admin 라우팅 정의 및 미들웨어
   - `middleware.ts` (admin 역할 확인)
   - 라우팅 구조 문서

3. 컴포넌트 사용 예제 및 스토리북 (선택)
   - `components/admin/AdminTable.stories.tsx` 등

4. 단위 테스트
   - `components/admin/__tests__/AdminLayout.test.tsx`
   - `components/admin/__tests__/SideNav.test.tsx`
   - `components/admin/__tests__/AdminTable.test.tsx`

5. 검수보고서 (5회 UI 리뷰)
   - UI 리뷰 기록 (데스크탑/태블릿/모바일)
   - 개선 내용 기록

6. 보안취약점검사보고서
   - 인증/권한 검사
   - XSS/CSRF 취약점 검사
   - localStorage 보안 검사

## 6. 완료 기준

- [ ] AdminLayout 컴포넌트가 모든 admin 페이지에 적용됨
- [ ] SideNav가 S1/S2 섹션을 모두 표시하고 권한 기반 필터링 동작
- [ ] 공통 컴포넌트(Table, Modal, Form, Status, Loading, Empty, Error)가 구현되고 재사용 가능
- [ ] 접기/펼치기 토글이 localStorage에 저장 및 복원됨
- [ ] 라우팅 미들웨어가 admin 역할 확인
- [ ] 반응형 디자인: 640px, 768px, 1024px 브레이크포인트에서 정상 동작
- [ ] WCAG 2.1 AA 준수 (키보드 네비게이션, 색상 대비, aria 속성)
- [ ] 단위 테스트 커버리지 > 80%
- [ ] 5회 UI 리뷰 완료 및 개선 사항 반영
- [ ] 보안취약점검사 완료 및 결과 보고서 작성

## 7. 작업 지침

### 7-1 S1/S2 원칙 준수
- **원칙 ①**: DocumentType별 분기는 서비스 레이어에서만 → Admin UI는 type-aware하지 않음 (데이터 표시만)
- **원칙 ②**: 구조는 generic + config 기반 → NAV_CONFIG를 중심으로 설정 관리
- **원칙 ④**: 모든 로직은 type-aware → 권한 확인 로직은 Scope Profile 조회 기반
- **원칙 ⑤**: AI 에이전트 동등성 → Admin 메뉴에 에이전트·Scope 관리 섹션 포함
- **원칙 ⑥**: Scope Profile로 권한 관리 → 하드코딩 금지 (if scope == "admin" 등)
- **원칙 ⑦**: 폐쇄망 환경 → 외부 CDN 최소화, 로컬 라이브러리만 사용

### 7-2 UI 리뷰 프로세스 (5회 이상)
**리뷰 체크리스트**:
1. **1차 리뷰**: 레이아웃 구조 (좌측 네비, 헤더, 메인 영역 배치)
2. **2차 리뷰**: 반응형 (640px, 768px, 1024px 각 브레이크포인트)
3. **3차 리뷰**: 색상/타이포그래피 (WCAG AA, 폰트 크기)
4. **4차 리뷰**: 인터랙션 (마우스/키보드, 토글, 호버 상태)
5. **5차 리뷰**: 접근성 (스크린 리더, aria 속성, 키보드 네비게이션)

각 리뷰마다 개선사항을 기록하고 반영할 것.

### 7-3 코드 스타일 및 컨벤션
- TypeScript strict mode 활성화
- 컴포넌트명: PascalCase, 훅/함수: camelCase
- Props 인터페이스는 컴포넌트 위에 정의
- 컴포넌트 폴더 구조:
  ```
  components/admin/
  ├── AdminLayout.tsx
  ├── SideNav.tsx
  ├── AdminHeader.tsx
  ├── AdminFooter.tsx
  ├── AdminTable.tsx
  ├── AdminModal.tsx
  ├── AdminForm.tsx
  ├── StatusBadge.tsx
  ├── LoadingSkeleton.tsx
  ├── EmptyState.tsx
  ├── ErrorState.tsx
  ├── __tests__/
  └── index.ts (배럴 내보내기)
  ```

### 7-4 테스트 작성
- React Testing Library 사용
- 각 컴포넌트의 렌더링, 상호작용, 상태 변화 테스트
- `userEvent` 사용 (click, keyboard 이벤트)
- localStorage 모킹 (MockStorage)

### 7-5 타입 안정성
- 모든 props, state, API 응답에 TypeScript 타입 정의
- `as const`로 문자열 리터럴 타입 정의 (e.g., status)
- 제네릭 활용 (AdminTable<T>, AdminForm<T>)

### 7-6 성능 최적화
- React.memo 사용 (자주 리렌더링되는 컴포넌트)
- useCallback으로 콜백 메모이제이션
- 이미지 최적화 (next/image)
- 번들 크기 모니터링

### 7-7 에러 처리 및 로깅
- API 호출 실패 시 ErrorState 표시
- 사용자 친화적 에러 메시지
- 콘솔 경고 대신 에러 로그 기록

### 7-8 문서화
- 컴포넌트 JSDoc 주석 추가
- PropTypes 또는 TypeScript 인터페이스로 props 문서화
- 라우팅 구조도 작성
- 사용 예제 (README 또는 Storybook)

---

**작업 예상 소요 시간**: 5~7일 (UI 리뷰, 테스트 포함)
**의존 관계**: FG6.1 완료 후 FG6.2~6.4 진행 가능
