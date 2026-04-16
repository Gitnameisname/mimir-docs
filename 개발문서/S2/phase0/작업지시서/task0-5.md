# Task 0-5. [Critical] 사용자 메뉴 및 네비게이션 구축

## 1. 작업 목적

사용자가 **로그아웃**할 수 있도록 헤더에 사용자 메뉴를 구축하고, 사용자 UI와 Admin UI 간의 **역할 기반 네비게이션**을 정상화한다.

현재 상태:
- Header.tsx의 아바타는 placeholder "U" 파란 원형 (실제 사용자 정보 미표시)
- 드롭다운 메뉴 없음
- 로그아웃 기능 접근 불가
- Admin UI로의 링크 없음
- User UI로 돌아오는 경로 불명확

이 작업은 사용자 경험의 **핵심 경로**(로그인 → 사용 → 로그아웃)의 마지막 부분을 완성한다.

---

## 2. 작업 범위

### 포함 범위

1. Header 컴포넌트 개선
   - `/frontend/src/components/layout/Header.tsx` 수정
   - placeholder 아바타 → 실제 사용자 이름/아이콘 표시
   - 드롭다운 메뉴 구현 (클릭 시 하위 메뉴 펼쳐짐)
   - 메뉴 항목: 프로필, 사용자 설정, 로그아웃
   - Admin 역할 시에만 "관리자 설정" 메뉴 항목 표시

2. Admin UI 헤더 개선
   - `/frontend/src/components/admin/layout/AdminHeader.tsx` 수정
   - 상단 우측에 "User UI로 돌아가기" 링크 추가
   - 네비게이션 breadcrumb 또는 모드 표시

3. 라우팅 연결
   - `/app/account/profile` → 프로필 보기 페이지
   - `/app/account/settings` → 사용자 설정 페이지
   - `/admin/users` → 사용자 관리
   - `/admin/roles` → 역할 관리
   - `/admin/api-keys` → API 키 관리
   - `/admin/settings` → 시스템 설정
   - `/admin/monitoring` → 모니터링 대시보드
   - `/admin/audit-logs` → 감사 로그

4. Zustand store 활용
   - `userStore` 또는 `authStore`에서 `user.role` 기반 조건부 렌더링
   - 로그아웃 시 세션 스토어 초기화

5. UI 5회 리뷰 실시
   - 최소 5회 이상 디자인 리뷰 수행
   - Task 0-10에서 정의할 체크리스트 활용

### 제외 범위

- 새로운 Account/Admin 페이지 생성 (S1 Phase 7/14에서 구축됨, 여기서는 연결만)
- 권한 검증 로직 (기존 auth 미들웨어 활용)
- 세션 저장소 설계 (기존 Zustand 패턴 활용)

---

## 3. 선행 조건

- S1 Phase 7/14 완료로 Admin UI 페이지 존재 (users, roles, api-keys, settings 등)
- `/frontend/src/lib/api/auth.ts`에서 logout 엔드포인트 존재
- `/frontend/src/stores/authStore.ts` (또는 유사) Zustand 스토어 존재 및 user 정보 포함
- Header.tsx, AdminHeader.tsx 파일 편집 가능 상태

---

## 4. 주요 작업 항목

### 4-1. Header.tsx 리팩토링

**파일:** `/frontend/src/components/layout/Header.tsx`

**수정 사항:**
```typescript
// Before: placeholder 아바타만 있음
// <div className="bg-blue-300 w-8 h-8 rounded-full flex items-center justify-center">
//   <span className="text-white">U</span>
// </div>

// After: 실제 사용자 메뉴
import { useAuthStore } from '@/stores/authStore';
import { useState, useRef } from 'react';
import { useRouter } from 'next/navigation';

export function Header() {
  const { user, logout } = useAuthStore();
  const [isMenuOpen, setIsMenuOpen] = useState(false);
  const menuRef = useRef<HTMLDivElement>(null);
  const router = useRouter();

  const handleLogout = async () => {
    await logout();
    router.push('/login');
  };

  const handleMenuClick = (path: string) => {
    router.push(path);
    setIsMenuOpen(false);
  };

  return (
    <header className="border-b bg-white">
      <div className="flex justify-between items-center p-4">
        <h1 className="font-semibold">Mimir</h1>
        
        <div className="relative" ref={menuRef}>
          <button
            onClick={() => setIsMenuOpen(!isMenuOpen)}
            className="flex items-center gap-2 p-2 rounded hover:bg-gray-100"
          >
            <div className="bg-blue-300 w-8 h-8 rounded-full flex items-center justify-center">
              <span className="text-white text-sm font-bold">
                {user?.name?.[0]?.toUpperCase() || 'U'}
              </span>
            </div>
            <span className="text-sm">{user?.name || 'User'}</span>
          </button>

          {isMenuOpen && (
            <div className="absolute right-0 mt-2 w-48 bg-white border rounded shadow-lg z-10">
              <button
                onClick={() => handleMenuClick('/account/profile')}
                className="w-full text-left px-4 py-2 hover:bg-gray-50 text-sm"
              >
                프로필 보기
              </button>
              <button
                onClick={() => handleMenuClick('/account/settings')}
                className="w-full text-left px-4 py-2 hover:bg-gray-50 text-sm"
              >
                사용자 설정
              </button>
              
              {user?.role === 'admin' && (
                <>
                  <hr className="my-1" />
                  <button
                    onClick={() => handleMenuClick('/admin')}
                    className="w-full text-left px-4 py-2 hover:bg-gray-50 text-sm font-semibold"
                  >
                    관리자 설정 →
                  </button>
                </>
              )}
              
              <hr className="my-1" />
              <button
                onClick={handleLogout}
                className="w-full text-left px-4 py-2 hover:bg-red-50 text-sm text-red-600"
              >
                로그아웃
              </button>
            </div>
          )}
        </div>
      </div>
    </header>
  );
}
```

**완료 기준:**
- 실제 사용자 이름/이니셜 표시됨
- 드롭다운 메뉴 열기/닫기 동작
- "프로필", "사용자 설정", "로그아웃" 메뉴 항목 표시
- Admin 역할 시에만 "관리자 설정" 표시
- 메뉴 항목 클릭 시 해당 페이지로 라우팅

### 4-2. AdminHeader.tsx 개선

**파일:** `/frontend/src/components/admin/layout/AdminHeader.tsx`

**수정 사항:**
```typescript
// AdminHeader에 "User UI로 돌아가기" 링크 추가
import { useRouter } from 'next/navigation';

export function AdminHeader() {
  const router = useRouter();

  return (
    <header className="border-b bg-white">
      <div className="flex justify-between items-center p-4">
        <div className="flex items-center gap-4">
          <span className="text-red-600 font-bold text-xs bg-red-50 px-2 py-1 rounded">
            관리자 모드
          </span>
          <h1 className="font-semibold">관리자 설정</h1>
        </div>

        <button
          onClick={() => router.push('/')}
          className="px-3 py-1 text-sm bg-blue-50 text-blue-600 rounded hover:bg-blue-100"
        >
          ← User UI로 돌아가기
        </button>
      </div>
    </header>
  );
}
```

### 4-3. 라우팅 연결 확인

**파일:** `/frontend/src/app/` (기존 페이지들)

**확인 작업:**
- `/app/account/profile` 페이지 존재 확인
- `/app/account/settings` 페이지 존재 확인
- `/admin/users`, `/admin/roles`, `/admin/api-keys`, `/admin/settings`, `/admin/monitoring`, `/admin/audit-logs` 페이지 존재 확인
- 각 페이지의 Admin 레이아웃 적용 확인

**필요 시 추가:**
- 라우팅 guard: Admin 페이지 접근 시 user.role === 'admin' 검증
- 예시:
```typescript
// /frontend/src/app/admin/layout.tsx
'use client';
import { useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { useAuthStore } from '@/stores/authStore';
import { AdminLayout } from '@/components/admin/layout/AdminLayout';

export default function AdminRootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const { user } = useAuthStore();
  const router = useRouter();

  useEffect(() => {
    if (user && user.role !== 'admin') {
      router.push('/');
    }
  }, [user]);

  return <AdminLayout>{children}</AdminLayout>;
}
```

### 4-4. Zustand store 활용

**파일:** `/frontend/src/stores/authStore.ts` (기존)

**확인 사항:**
- `useAuthStore()` 훅 제공 여부
- `user` 객체에 `name`, `role` 필드 포함 여부
- `logout()` 메서드 존재 여부
- 로그아웃 후 `user` 상태 초기화 여부

**필요 시 추가:**
```typescript
interface AuthStore {
  user: { name: string; role: 'user' | 'admin' } | null;
  logout: () => Promise<void>;
}
```

### 4-5. UI 5회 리뷰 실시

**리뷰 항목 (Task 0-10에서 정의):**
1. 접근성 (키보드 네비게이션, ARIA 라벨)
2. 반응형 (모바일, 태블릿, 데스크톱)
3. 오류 메시지 명확성
4. 네비게이션 경로 명확성
5. 데이터 로딩 상태 표시

**리뷰 회차별 개선 기록:**
- 1회: 초안 리뷰 (UI 배치, 기본 동작)
- 2회: 접근성 리뷰 (키보드 네비게이션, 포커스 표시)
- 3회: 반응형 리뷰 (다양한 화면 크기)
- 4회: 사용성 리뷰 (메뉴 항목 순서, 로그아웃 확인)
- 5회: 최종 리뷰 (통합 테스트, 문서화)

---

## 5. 산출물

1. **Header.tsx** (수정본)
   - 사용자 메뉴 드롭다운 구현
   - 프로필, 설정, 로그아웃 메뉴 항목
   - Admin 역할 조건부 렌더링

2. **AdminHeader.tsx** (수정본)
   - "User UI로 돌아가기" 링크
   - 관리자 모드 표시

3. **라우팅 가드 (선택)** 
   - `/admin/layout.tsx`의 역할 기반 접근 제어

4. **UI 리뷰 기록**
   - 5회 리뷰 결과 문서
   - 각 회차별 개선 사항

---

## 6. 완료 기준

1. Header.tsx 드롭다운 메뉴 구현 및 동작 확인
2. 로그아웃 클릭 시 세션 종료 및 login 페이지 리다이렉트 확인
3. Admin 역할 사용자에게만 "관리자 설정" 메뉴 표시 확인
4. Admin UI → User UI 전환 정상 작동 확인
5. 모든 라우팅 경로 정상 작동 (404 없음)
6. 키보드 네비게이션 (Tab, Enter) 작동 확인
7. 모바일/태블릿/데스크톱 반응형 정상 동작
8. UI 5회 리뷰 완료 및 개선 사항 반영
9. 코드 스타일 (TypeScript, ESLint) 통과
10. 기존 기능 회귀 테스트 통과 (다른 헤더 기능 영향 없음)

---

## 7. 작업 지침

### 지침 7-1. 외부 클릭 시 메뉴 닫기

드롭다운 메뉴의 접근성을 위해 다음을 고려:

```typescript
useEffect(() => {
  const handleClickOutside = (e: MouseEvent) => {
    if (menuRef.current && !menuRef.current.contains(e.target as Node)) {
      setIsMenuOpen(false);
    }
  };

  if (isMenuOpen) {
    document.addEventListener('mousedown', handleClickOutside);
  }

  return () => {
    document.removeEventListener('mousedown', handleClickOutside);
  };
}, [isMenuOpen]);
```

### 지침 7-2. 로그아웃 확인 다이얼로그 (선택)

중요한 작업이므로 확인 메시지 추가 고려:

```typescript
const handleLogout = async () => {
  if (confirm('정말 로그아웃 하시겠습니까?')) {
    await logout();
    router.push('/login');
  }
};
```

### 지침 7-3. 로딩 상태 표시

로그아웃 중 버튼 비활성화:

```typescript
const [isLoggingOut, setIsLoggingOut] = useState(false);

const handleLogout = async () => {
  setIsLoggingOut(true);
  try {
    await logout();
    router.push('/login');
  } finally {
    setIsLoggingOut(false);
  }
};

<button
  onClick={handleLogout}
  disabled={isLoggingOut}
  className="... disabled:opacity-50"
>
  {isLoggingOut ? '로그아웃 중...' : '로그아웃'}
</button>
```

### 지침 7-4. 권장 CSS 프레임워크

기존 프로젝트 스타일과 일관성 유지:
- Tailwind CSS 클래스명 따르기
- 색상 팔레트 통일 (기존 Header 스타일 참고)
- 다크 모드 대응 (선택, 필요 시)

### 지침 7-5. 테스트 시나리오

수동 테스트:
1. 로그인 → 사용자 이름 표시 확인
2. 메뉴 버튼 클릭 → 드롭다운 펼쳐짐
3. "프로필" 클릭 → `/account/profile` 이동
4. "사용자 설정" 클릭 → `/account/settings` 이동
5. (Admin) "관리자 설정" 클릭 → `/admin` 이동
6. (Admin) "User UI로 돌아가기" 클릭 → `/` 이동
7. "로그아웃" 클릭 → 세션 종료, login 페이지 리다이렉트
8. 메뉴 열린 상태에서 외부 클릭 → 메뉴 닫힘
