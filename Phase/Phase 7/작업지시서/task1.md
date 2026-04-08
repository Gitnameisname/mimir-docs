# Task 7-1. Admin UI 정보 구조 및 접근 정책 설계

## 1. 작업 목적

관리자 UI(Admin UI)의 전체 구조를 정의하고, 운영자 메뉴 체계와 접근 정책을 설계한다.

이 작업의 목표는 다음과 같다.

- 운영자가 시스템을 어떻게 탐색하고 통제하는지 명확히 정의
- User UI와 구조적으로 분리된 Admin UI 정보 구조(IA) 확립
- 이후 모든 Admin 화면 설계의 기준이 되는 라우팅/접근 정책 확정

---

## 2. 작업 범위

### 포함 범위

- Admin UI 전체 메뉴 구조 정의
- 주요 페이지 라우팅 설계 (Next.js 기준)
- 관리자 접근 권한 정책 정의
- 역할 기반 메뉴 노출 제어 방침
- 공통 레이아웃 구조 초안

### 제외 범위

- 실제 UI 디자인 (스타일, 색상 등)
- 컴포넌트 상세 설계
- API 연동 구현

---

## 3. 선행 조건

- Phase 2: 역할(Role) 및 권한 모델 정의 완료
- Phase 3: 공통 API 규약 정의 완료
- 관리자 권한 판별 방식 확정

---

## 4. 주요 설계 대상

### 4-1. Admin UI 최상위 메뉴 구조

Admin UI에서 제공할 주요 메뉴를 정의한다.

예시:

- Dashboard (운영 상태 요약)
- Users (사용자/조직/역할 관리)
- Permissions (권한 정책 관리)
- Document Types (DocumentType 관리)
- Audit Log (감사 로그 조회)
- Background Jobs (백그라운드 작업 모니터링)
- Indexing (벡터화/인덱싱 상태)
- API Keys (API 키 및 외부 연계)

각 메뉴의 목적과 포함 기능을 명확히 정의한다.

---

### 4-2. 라우팅 구조 (Route Map)

Next.js 기준으로 Admin URL 구조를 설계한다.

예시:

- `/admin`
- `/admin/dashboard`
- `/admin/users`
- `/admin/users/{user_id}`
- `/admin/organizations`
- `/admin/roles`
- `/admin/permissions`
- `/admin/document-types`
- `/admin/audit-logs`
- `/admin/jobs`
- `/admin/indexing`
- `/admin/api-keys`

요구사항:

- User UI 라우트(`/app/*`)와 완전히 분리
- RESTful하고 일관된 URL 구조
- 확장 가능한 계층 구조

---

### 4-3. 접근 권한 정책

Admin UI 접근 제어 방침을 정의한다.

#### (1) 접근 주체

- SUPER_ADMIN: 모든 Admin 메뉴 접근 가능
- ORG_ADMIN: 자신의 조직 범위 내 관리 기능 접근
- 일반 사용자: Admin UI 접근 불가

#### (2) 접근 제어 방식

- 미인증 또는 권한 부족 시 리다이렉트 규칙 정의
- 메뉴 노출 제어 vs. 라우트 보호 구분
- 서버 사이드 권한 검사 기준

#### (3) 민감 기능 보호

- API 키 발급/폐기, 권한 정책 변경 등 고위험 액션의 추가 확인 정책

---

### 4-4. 역할 기반 메뉴 노출 제어

역할별로 노출할 메뉴 항목을 정의한다.

| 메뉴 | SUPER_ADMIN | ORG_ADMIN |
|------|-------------|-----------|
| Dashboard | O | O (범위 제한) |
| Users | O | O (조직 내) |
| Permissions | O | X |
| Document Types | O | X |
| Audit Log | O | O (조직 내) |
| Background Jobs | O | O |
| Indexing | O | O |
| API Keys | O | O (자신 발급분) |

---

### 4-5. 공통 레이아웃 구조 초안

Admin UI의 기본 레이아웃을 정의한다.

구성 요소:

- Top Header (시스템명, 관리자 정보, 글로벌 알림)
- Left Sidebar (Admin 네비게이션)
- Main Content (관리 화면 본문)
- Breadcrumb (현재 위치 안내)

레이아웃 요구사항:

- User UI 레이아웃과 명확히 구별되는 시각적 구조
- 운영자 중심의 정보 밀도 (데이터 테이블 중심)
- 모바일 대응보다 데스크탑 운영 환경 우선

---

## 5. 산출물

다음 문서를 작성한다.

1. Admin UI 정보 구조(IA) 문서
2. Admin UI 라우팅 구조 정의서
3. 접근 권한 정책 문서
4. 역할별 메뉴 접근 매트릭스

---

## 6. 완료 기준

다음 조건을 만족하면 완료로 본다.

- Admin UI 전체 메뉴 구조가 명확히 정의되어 있다
- User UI와 분리된 라우팅 구조가 확정되어 있다
- 접근 권한 정책이 역할 기반으로 정의되어 있다
- 이후 Task(7-2~7-12)가 이 문서를 기준으로 설계 가능하다

---

## 7. Codex 작업 지침

- 실제 코드 구현은 하지 않는다
- 설계 문서 중심으로 작성한다
- 구조화된 Markdown, 표, 계층 리스트를 적극 활용한다
- User UI와의 분리 원칙을 모든 설계에서 명확히 반영한다
- 추상적 표현보다 명확한 구조 정의를 우선한다
