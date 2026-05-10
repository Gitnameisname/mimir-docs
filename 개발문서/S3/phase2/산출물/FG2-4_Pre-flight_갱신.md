# FG 2-4 Pre-flight 갱신 메모 (재진입 전 사실 baseline)

**작성일**: 2026-05-10
**대상 작업지시서**: `docs/개발문서/S3/phase2/작업지시서/task2-4.md` (작성일 2026-04-24)
**목적**: task2-4.md 작성 후 Phase 3/4 + FG 2-3 (2026-05-10 종결) 가 코드베이스를 진화시켰다. 본 메모는 사실 baseline 을 2026-05-10 시점으로 갱신해 구현 진입 시 충돌을 차단한다.

---

## 1. 진행 상태 실측

### 1.1 미구현 확인 (2026-05-10)

- ❌ `app/services/graph_service.py` — 부재
- ❌ `GET /documents/graph` 라우터 — 부재
- ❌ `frontend/src/features/documents/LayoutToggle.tsx` — 부재
- ❌ `frontend/src/features/documents/GraphView.tsx` / `GraphView.impl.tsx` — 부재
- ❌ frontend `package.json` 에 cytoscape / cytoscape-cose-bilkent / cytoscape-dagre / dagre 의존성 부재
- ❌ `users.preferences.documents.default_layout` — 명시적 schema 부재 (단 `UserPreferences.extra: allow` 라 키 자체는 허용)

### 1.2 선행 조건 (FG 2-3 종결 후 충족)

- ✅ FG 2-1 (collections / folders) — 종결 (2026-04-24)
- ✅ FG 2-2 (tags / document_tags) — 종결 (2026-04-25)
- ✅ FG 2-3 (백링크 / document_links) — **2026-05-10 종결** (재진입 흡수). 그래프의 백링크 엣지 데이터 가용.
- ✅ users.preferences (Phase 1 FG 1-3) — `UserPreferences` model + `/account/preferences` GET/PATCH 가용
- ✅ 블로커2 결정서 (cytoscape.js) 합의 완료

### 1.3 의도가 코드 docstring 에 박혀있는 흔적

`backend/app/api/v1/account_router.py:87` (`UserPreferences` docstring):

```
공식 필드:
  - editor_view_mode: "block" | "flow"
  - theme: "system" | "light" | "dark"
extra 는 allow — 향후 language 등 확장 허용.
```

→ `documents.default_layout` 키는 `extra: allow` 위에 추가 가능. 단 task2-4.md 의 명시적 Literal 등록이 안전 — 본 FG 에서 결정 (§2.1).

---

## 2. 갱신 항목

### 2.1 UserPreferences schema — `documents.default_layout` 처리 결정

**task2-4.md §3 의 가정**: "users.preferences (Phase 1 FG 1-3) 의 documents 하위 키 확장 — default_layout: list|tree|cards|graph"

**갱신 결정**: **(A) `extra: allow` 활용 — 명시적 schema 변경 없음**.

근거:
- `UserPreferences` 가 이미 `extra: allow` 로 임의 키 허용
- 명시적 Literal 추가 시 backend pydantic + frontend 타입 + PATCH 요청 schema 모두 동시 변경 (변경 비용 큼)
- frontend `useUserPreferences` 훅이 이미 `unknown` 으로 들어오는 임의 키를 처리 (확인 필요)
- tradeoff: 타입 안전성 약함 — 본 FG 의 LayoutToggle 이 default_layout 값 검증 + invalid 시 "list" fallback

본 결정은 검수보고서에 명시 (FG 2-2 UX1 의 `theme` 추가가 schema 명시였던 것과 대비). 향후 language / locale 추가 시 한 번에 schema 화 권장.

### 2.2 LayoutToggle 의 데이터 흐름 — 단일 정본

```
URL ?layout=<v>  (1차)
   ↓
React state currentLayout
   ↓
사용자 토글
   ↓
URL 갱신 + preferences PATCH (debounced 400ms — useUserPreferences 표준)
```

- URL 이 1차 정본 (공유 가능, 새로고침 복원)
- preferences 는 fallback (URL 부재 시 마지막 선택 복원)
- preferences 가 없으면 default `"list"`

### 2.3 그래프 데이터 API — 데이터 소스 정합

`graph_service.py` 가 합쳐야 할 4 데이터 소스 (현 상태 확인):

| 소스 | 테이블 | FG | scope 필터 |
|------|--------|----|-----------|
| 노드: document | `documents` (`scope_profile_id`) | FG 2-0 | viewer scope IN |
| 엣지: backlink | `document_links` (`resolved_status='resolved'`) | FG 2-3 (2026-05-10 신설) | from/to 양쪽 documents 가 viewer scope 안 |
| 엣지: tagged + 노드: tag | `document_tags` + `tags` | FG 2-2 | document scope 통과 후 그 documents 의 tag 만 |
| 엣지: in_collection + 노드: collection | `collection_documents` + `collections` | FG 2-1 | document scope 통과 후 그 documents 의 collection 만 |

**ACL 정책**:
- documents 는 viewer scope IN 강제 (정본)
- tags / collections 메타노드는 documents 통과 후 자연 필터 (어떤 document 도 통과 못하면 메타노드도 0)
- `?include_tag_nodes=true` 옵션 — 기본 false (그래프가 polluted 되지 않도록)
- `?include_collection_nodes=true` 옵션 — 기본 false

### 2.4 노드 / 엣지 상한

- **nodes**: default 500, max 2000 (task2-4.md §2.1 line 39)
- **edges**: 별 상한 없음 — nodes 가 bounded 면 자연 bounded
- truncated 플래그 응답에 포함

### 2.5 cytoscape 의존성 추가 — P1 게이트

task2-4.md §4 Step 4:
```
npm install cytoscape @types/cytoscape cytoscape-cose-bilkent cytoscape-dagre dagre
```

**본 세션 결정**: 코드는 작성하되 **`npm install` 자체는 운영자 환경에서 실행** (P1 게이트).

근거:
- 의존성 추가는 빌드 / 폐쇄망 / npm audit 영향 — 외부 시스템 영향
- task2-4.md §6 보안 / §5 폐쇄망 검증 모두 운영자 환경 의존
- Claude 세션이 npm install 실행 시 운영자 검증 없이 의존성이 추가되어 위험

**진행 방식**:
- 코드는 cytoscape import 가정으로 작성 (`@ts-expect-error` 또는 type stub)
- `package.json` 의 dependencies 섹션 수정만 — 운영자가 `npm install` 실행
- frontend 빌드 / type check 는 의존성 설치 후 운영자가 실측

### 2.6 폐쇄망 검증 — 운영자 환경 의존

task2-4.md §5 체크리스트는 모두 운영자 환경에서 실행 필수:

- `npm install --prefer-offline` 성공
- `next build` chunk 분리 검증
- 외부 CDN fetch 없음 검증
- 네트워크 차단 상태 그래프 페이지 정상 렌더
- CSP 위반 없음

→ 본 세션 산출물은 코드만, 검증은 별 라운드 (운영자 환경).

### 2.7 npm audit critical 검증 — 운영자 환경 의존

- 코드에 의존성 declaration 만 추가
- 운영자가 `npm install` 직후 `npm audit --production --audit-level=critical` 실행
- 결과는 별 라운드의 `FG2-4_번들검증.md` 에 기록

### 2.8 task2-4.md 의 Step 별 본 세션 진행 범위

| Step | 내용 | 본 세션 진행 |
|------|------|-----------|
| Step 1 | 그래프 데이터 API | ✅ 진행 |
| Step 2 | 레이아웃 토글 | ✅ 진행 |
| Step 3 | 4 종 레이아웃 본체 | ✅ 진행 (graph 는 stub) |
| Step 4 | cytoscape 설치 / 번들 분리 | 코드만 (npm install 보류) |
| Step 5 | GraphView 본체 | ✅ 진행 (의존성 설치 후 동작) |
| Step 6 | 성능 / 뷰포트 | 운영자 환경 측정 — 별 라운드 |
| Step 7 | UI 검수 / 반응형 | UI 디자인 리뷰 — 별 라운드 |
| Step 8 | 검수 / 보안 보고서 | ✅ 진행 (잔여 명시) |

### 2.9 라우터 등록 위치

`/documents/graph` path 가 documents 라우터 안의 `/{document_id}` 와 충돌 가능 — FG 2-3 에서 `/resolve` 와 동일 패턴.

해결: documents.py 라우터 내부에 `/graph` 를 `/{document_id}` 보다 **먼저** 등록 OR FG 2-3 처럼 `app/api/v1/document_links.py` 와 같은 별 모듈 (`app/api/v1/document_graph.py`) 신설 + `/documents` prefix 공유 + documents 보다 앞에 include.

후자가 일관성 측면에서 정합 (FG 2-3 에서 확립한 패턴). 본 FG 채택.

---

## 3. P1 승인 게이트

본 FG 종결 시 다음 항목 운영자 P1 승인 필요:

1. **cytoscape 의존성 추가** — `npm install` 실행 + `npm audit` 검증
2. **GET /documents/graph API 표면 추가** — 외부 API 표면 변경
3. **종결 선언** — Phase 2 FG 2-4 공식 종결

승인자: `@최철균` (Phase 5 §8 / AGENT_MODE.md §3.3)

---

## 4. Phase 5 정합 — task5-1 ADR

본 FG 는 ProseMirror schema / TipTap mark 와 무관 (그래프 뷰는 별 표면). task5-1 ADR §a 에 명시 변경 없음.

---

## 5. 알려진 잔여 / 후속

본 세션에서 진행 못 하는 항목:

- npm install 실제 실행 + 의존성 lock 갱신 (P1 후 운영자)
- `npm audit --production --audit-level=critical` 실측 (운영자)
- 폐쇄망 번들 검증 — `next build` + chunk 분석 + 네트워크 차단 테스트 (운영자)
- 500 노드 / 2000 엣지 5초 이내 렌더 측정 (운영자)
- UI 디자인 리뷰 1회 (4 viewport — desktop / narrow / tablet / mobile)
- node:test ≥ 20건 (테스트는 의존성 설치 후 실제 cytoscape API 동작 검증 가능)
- frontend tsc 0 error 검증 (의존성 설치 후)

---

*작성: 2026-05-10 | FG 2-4 그래프 뷰 + 4 레이아웃 — 재진입 전 Pre-flight 갱신*
