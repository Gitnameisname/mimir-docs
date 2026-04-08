# Task 6-12 산출물: 구현 우선순위 및 MVP 범위 확정

---

## 1. MVP 기능 정의서

### 1-1. MVP 포함 기능 (반드시 동작해야 하는 최소 기능)

| # | 기능 | 화면 | 우선순위 |
|---|------|------|--------|
| 1 | 문서 목록 조회 | `/documents` | P0 |
| 2 | 기본 키워드 검색 | `/documents` | P0 |
| 3 | 문서 상세 보기 | `/documents/[id]` | P0 |
| 4 | 구조 트리 탐색 | `/documents/[id]` | P0 |
| 5 | 새 문서 생성 | `/documents/new` | P0 |
| 6 | Draft 문서 편집 및 저장 | `/documents/[id]/edit` | P0 |
| 7 | 검토 요청 | `/documents/[id]` (액션) | P0 |
| 8 | 버전 목록 조회 | `/documents/[id]/versions` | P1 |
| 9 | 검토 대기 목록 | `/reviews` | P1 |
| 10 | 승인 / 반려 액션 | `/documents/[id]` (액션) | P1 |
| 11 | 공통 오류/로딩/권한 처리 | 전체 | P0 |
| 12 | 상태/역할 기반 버튼 제어 | 전체 | P0 |

### 1-2. MVP 제외 기능 (후속 Phase)

| 기능 | 후속 Phase |
|------|----------|
| 고급 검색 필터 (전문 검색) | Phase 8 |
| 버전 간 시각적 Diff | Phase 9 |
| 실시간 협업 편집 | Phase 9+ |
| RAG 질의 실제 기능 | Phase 11 |
| DocumentType별 커스텀 Editor | Phase 12 |
| 관리자 UI (조직/정책 관리) | Phase 7 |
| 고급 코멘트 / 협업 기능 | Phase 9+ |

---

## 2. 기능 분류표 (필수 / 확장)

```
MVP 범위
┌────────────────────────────────────────────────────┐
│  TIER 1 (P0 — 즉시 구현)                           │
│  - 레이아웃 + 공통 컴포넌트                         │
│  - 문서 목록 + 기본 검색                            │
│  - 문서 상세 보기 + 구조 트리                       │
│  - 문서 편집 + 저장                                 │
│  - 상태/역할 기반 UI 제어                           │
│  - 공통 피드백 UX                                   │
├────────────────────────────────────────────────────┤
│  TIER 2 (P1 — MVP 내 후속)                         │
│  - 버전 목록 조회                                   │
│  - 검토 대기 목록                                   │
│  - 승인/반려 액션                                   │
├────────────────────────────────────────────────────┤
│  TIER 3 (P2 — MVP 이후)                            │
│  - 버전 상세 보기                                   │
│  - 버전 복원                                        │
│  - RAG 패널 Stub                                   │
│  - 필터 확장                                        │
└────────────────────────────────────────────────────┘
```

---

## 3. 구현 단계별 로드맵

### Phase 6-1. 기본 구조 구축

**목표**: Next.js 프로젝트 세팅 + 레이아웃 + 공통 컴포넌트

**주요 작업**:
- Next.js 14 (App Router) 프로젝트 초기화
- TailwindCSS + 디자인 기반 설정
- TanStack Query 설정 (서버 상태 관리)
- API 클라이언트 계층 구현 (`/src/lib/api/`)
- 공통 컴포넌트 구현:
  - `AppLayout`, `Header`, `Sidebar`
  - `WorkflowStatusBadge`, `DocumentTypeBadge`
  - `Toast`, `ConfirmDialog`, `EmptyState`, `ErrorState`, `SkeletonBlock`
  - `Button`, `SearchInput`, `FormField`

**완료 기준**: `/documents` 라우트 진입 시 레이아웃 정상 표시

---

### Phase 6-2. 열람 기능 구현

**목표**: 문서 목록 → 상세 → 구조 탐색 흐름 완성

**주요 작업**:
- `/documents` 페이지 구현 (목록, 검색, 필터, 정렬, 페이지네이션)
- `/documents/[id]` 페이지 구현 (상세 보기, 메타데이터, 액션 버튼)
- `DocumentTree` 컴포넌트 (구조 트리 + 스크롤 동기화)
- Node 렌더링 엔진 (기본 타입: heading, paragraph, list)
- `WorkflowStatusBadge` 기반 상태 표시

**완료 기준**: 문서 목록에서 문서를 선택하여 본문과 구조 트리를 읽을 수 있음

---

### Phase 6-3. 편집 기능 구현

**목표**: Draft 문서 작성 및 편집 완성

**주요 작업**:
- `/documents/new` 페이지 구현 (생성 폼)
- `/documents/[id]/edit` 페이지 구현
- Node 기반 편집기 구현:
  - 인라인 텍스트 편집
  - Node 추가/삭제/순서 변경
- 저장 상태 관리 (명시적 저장 + 자동 저장)
- Unsaved Changes 이탈 경고

**완료 기준**: 새 문서 생성 → 편집 → 저장 → 검토 요청 흐름 동작

---

### Phase 6-4. 버전 및 워크플로

**목표**: 버전 이력 조회 + 승인/반려 흐름 완성

**주요 작업**:
- `/documents/[id]/versions` 페이지 구현 (타임라인)
- `/reviews` 페이지 구현 (검토 대기 목록)
- 승인 / 반려 액션 모달 구현
- 워크플로 이력 표시 (문서 상세 하단)

**완료 기준**: 검토자가 검토 대기 목록에서 문서를 승인/반려할 수 있음

---

### Phase 6-5. 공통 UX 및 안정화

**목표**: 전체 UX 일관성 확보 + MVP 완성

**주요 작업**:
- Route Guard 적용 (전 화면)
- 전체 상태/역할 기반 버튼 제어 검토
- 공통 피드백 UX 전면 적용 (Toast, 오류, 로딩)
- 반응형 처리 (Tablet 기준)
- API 오류 처리 전체 점검
- RAG 패널 Stub 추가 (버튼만 노출, Phase 11 연계 포인트)

**완료 기준**: Phase 6 완료 기준 8개 항목 모두 충족

---

## 4. API 의존성 매핑 문서

| UI 기능 | 의존 API | Phase 5 완료 여부 |
|--------|---------|----------------|
| 문서 목록 | `GET /api/v1/documents` | ✓ |
| 문서 상세 | `GET /api/v1/documents/{id}` | ✓ |
| 버전 상세 | `GET /api/v1/documents/{id}/versions/{vid}` | ✓ |
| Node 목록 | `GET /api/v1/documents/{id}/nodes` | ✓ |
| 문서 생성 | `POST /api/v1/documents` | ✓ |
| Draft 저장 | `PATCH /api/v1/documents/{id}/versions/{vid}/draft` | ✓ |
| 검토 요청 | `POST .../workflow/submit-review` | ✓ |
| 승인 | `POST .../workflow/approve` | ✓ |
| 반려 | `POST .../workflow/reject` | ✓ |
| 발행 | `POST .../workflow/publish` | ✓ |
| 워크플로 이력 | `GET .../workflow/history` | ✓ |
| 버전 복원 | `POST .../versions/{vid}/restore` | 확인 필요 |
| 검토 액션 목록 | `GET .../workflow/review-actions` | ✓ |
| RAG 질의 | `POST /api/v1/rag/query` | Phase 11 |

> **주의**: 버전 복원 API 존재 여부 확인 후 Phase 6-4에서 구현 여부 결정

---

## 5. 기술 스택 확정

| 영역 | 기술 선택 | 근거 |
|------|---------|------|
| 프레임워크 | Next.js 14 (App Router) | 기존 계획, SSR + CSR 혼합 가능 |
| 스타일링 | TailwindCSS | 빠른 개발, 유틸리티 기반 |
| 서버 상태 | TanStack Query (React Query) | 캐시 + 재조회 + 낙관적 업데이트 |
| 클라이언트 상태 | Zustand | 간단한 글로벌 상태 (Toast, 패널 열림 등) |
| 폼 관리 | React Hook Form + Zod | 타입 안전 + 유효성 검사 |
| 에디터 | Tiptap (ProseMirror 기반) | Node 구조 편집 + 확장성 |
| 드래그 | dnd-kit | Node 순서 변경 |
| API 클라이언트 | fetch + custom wrapper | 인증 헤더, 에러 처리 통합 |

---

## 6. 프론트엔드 디렉토리 구조

```
frontend/
├── src/
│   ├── app/                     # Next.js App Router
│   │   ├── layout.tsx            # Root Layout
│   │   ├── documents/
│   │   │   ├── page.tsx          # /documents (목록)
│   │   │   ├── new/
│   │   │   │   └── page.tsx      # /documents/new
│   │   │   └── [id]/
│   │   │       ├── page.tsx      # /documents/[id] (상세)
│   │   │       ├── edit/
│   │   │       │   └── page.tsx  # /documents/[id]/edit
│   │   │       └── versions/
│   │   │           ├── page.tsx  # /documents/[id]/versions
│   │   │           └── [vid]/
│   │   │               └── page.tsx
│   │   └── reviews/
│   │       └── page.tsx          # /reviews
│   ├── components/               # 공통 컴포넌트 (Task 6-2 기준)
│   ├── features/                 # 기능별 컴포넌트
│   │   ├── documents/            # 문서 관련 컴포넌트
│   │   ├── editor/               # 편집기 컴포넌트
│   │   ├── versions/             # 버전 관련 컴포넌트
│   │   ├── workflow/             # 워크플로 액션 컴포넌트
│   │   └── rag/                  # RAG 패널 (Stub)
│   ├── lib/
│   │   ├── api/                  # API 클라이언트 계층
│   │   │   ├── client.ts         # fetch wrapper + 인증
│   │   │   ├── documents.ts      # 문서 API 함수
│   │   │   ├── versions.ts
│   │   │   ├── nodes.ts
│   │   │   └── workflow.ts
│   │   └── utils/               # 유틸리티 함수
│   ├── hooks/                    # 커스텀 훅
│   │   ├── useDocuments.ts       # TanStack Query 훅
│   │   ├── useWorkflow.ts
│   │   └── usePermission.ts      # 역할/상태 기반 권한 훅
│   ├── stores/                   # Zustand 스토어
│   │   └── uiStore.ts            # Toast, 패널 상태 등
│   └── types/                    # TypeScript 타입 정의
│       ├── document.ts
│       ├── version.ts
│       ├── workflow.ts
│       └── user.ts
├── package.json
└── tailwind.config.ts
```

---

## 7. MVP 완료 기준 체크리스트

- [ ] 일반 사용자가 문서를 목록에서 찾고 상세에서 읽을 수 있다
- [ ] 구조 트리를 통해 문서를 구조적으로 탐색할 수 있다
- [ ] 권한 있는 사용자가 Draft 문서를 편집하고 저장할 수 있다
- [ ] 사용자가 문서의 버전 목록과 주요 상태를 확인할 수 있다
- [ ] 검토자가 Review 상태 문서에 대해 검토/승인/반려 액션을 수행할 수 있다
- [ ] 역할과 상태에 따라 UI 액션이 올바르게 제한된다
- [ ] 공통 오류/로딩/권한 부족 상태가 일관되게 처리된다
- [ ] 후반 Phase의 Search/Diff/RAG 기능을 붙일 수 있는 UI 확장 포인트가 확보된다
