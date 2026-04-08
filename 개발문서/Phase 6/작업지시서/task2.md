

# Task 6-2. 공통 UI 프레임워크 및 레이아웃 컴포넌트 설계

## 1. 작업 목적

User UI 전반에서 일관된 사용자 경험을 제공하기 위해 **공통 UI 프레임워크와 레이아웃 구조를 정의**한다.

이 작업의 핵심 목표는 다음과 같다.

- 모든 화면에서 재사용 가능한 레이아웃 구조 확립
- 페이지 간 일관된 UX 제공
- 이후 개별 화면(Task 6-3 ~ 6-7)의 기반이 되는 공통 컴포넌트 정의

---

## 2. 작업 범위

### 포함 범위

- 공통 레이아웃 구조 정의
- 공통 UI 컴포넌트 식별 및 설계
- 상태 표현 패턴 정의 (Loading / Error / Empty 등)
- 공통 액션 UI 패턴 정의

### 제외 범위

- 실제 UI 스타일링 (색상, 디자인 시스템)
- 개별 페이지 상세 구현

---

## 3. 선행 조건

- Task 6-1 (정보 구조 및 화면 맵 설계) 완료

---

## 4. 주요 설계 대상

### 4-1. 전체 레이아웃 구조

User UI의 기본 레이아웃을 정의한다.

#### 기본 구조

- Header
- Sidebar (Navigation)
- Main Content
- Optional Right Panel (확장 영역)

#### 요구사항

- 모든 주요 화면에서 동일한 구조 유지
- 콘텐츠 중심 레이아웃
- 확장 가능한 패널 구조 (RAG, 메타데이터, 트리 등)

---

### 4-2. Header 구성 요소

Header에 포함될 요소 정의

- 시스템 로고 / 서비스명
- 사용자 정보 (프로필, 로그아웃)
- 글로벌 액션 (검색 또는 알림 등)

---

### 4-3. Sidebar (Navigation)

Sidebar는 주요 탐색 역할을 수행한다.

#### 포함 항목

- Documents
- My Tasks
- (선택) Dashboard
- (선택) Favorites / Recent

#### 요구사항

- 현재 위치 표시 (active state)
- 메뉴 확장 가능 구조

---

### 4-4. Main Content 영역

각 페이지의 핵심 콘텐츠가 표시되는 영역

#### 요구사항

- 페이지별 독립적인 구성 가능
- 상단 Action Bar 지원
- 스크롤 관리 기준 정의

---

### 4-5. 공통 페이지 패턴

모든 페이지에서 공통적으로 사용하는 UI 패턴 정의

#### (1) Page Header

- 페이지 제목
- 설명 (선택)
- 주요 액션 버튼

#### (2) Action Bar

- 검색
- 필터
- 정렬
- 주요 액션 버튼

---

### 4-6. 상태 표현 패턴

공통적으로 사용되는 상태 UI 정의

#### (1) Loading 상태

- Skeleton 또는 Spinner

#### (2) Empty 상태

- 데이터 없음 안내
- 행동 유도 버튼

#### (3) Error 상태

- 오류 메시지
- 재시도 버튼

#### (4) Permission Denied

- 접근 권한 없음 안내

---

### 4-7. 공통 액션 UI 패턴

#### 버튼 유형

- Primary Action (저장, 승인 등)
- Secondary Action
- Danger Action (삭제, 반려 등)

#### Modal / Dialog

- Confirm Dialog
- Form Modal

#### Notification

- Toast 메시지
- 성공 / 실패 / 경고 상태

---

### 4-8. 컴포넌트 분류

공통 컴포넌트를 다음과 같이 분류한다.

#### Layout 컴포넌트

- AppLayout
- Header
- Sidebar

#### Page 컴포넌트

- PageHeader
- ActionBar

#### Feedback 컴포넌트

- Loading
- Empty
- Error

#### Control 컴포넌트

- Button
- Input
- Select

---

## 5. 산출물

1. 레이아웃 구조 정의서
2. 공통 컴포넌트 목록
3. 상태 표현 가이드
4. UI 패턴 정의 문서

---

## 6. 완료 기준

- 전체 UI 레이아웃 구조가 정의되어 있다
- 공통 컴포넌트가 명확히 식별되어 있다
- 상태 표현 방식이 일관되게 정의되어 있다
- 이후 화면 설계(Task 6-3 이상)가 이 구조를 기반으로 진행 가능하다

---

## 7. Codex 작업 지침

- 코드 작성 금지 (설계 중심)
- 재사용성과 확장성을 최우선 고려
- 컴포넌트 단위로 명확히 구분
- 불필요한 복잡성 제거