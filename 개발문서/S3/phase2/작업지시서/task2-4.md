# task2-4 — 그래프 뷰 + 4종 레이아웃 전환 (cytoscape.js)

**작성일**: 2026-04-24
**Phase / FG**: S3 Phase 2 / FG 2-4
**상태**: 착수 대기 (블로커 2 결정서 승인 후)
**담당**: 프런트 (주) + 백엔드 (그래프 조회 API)
**예상 소요**: 3~4일
**선행 산출물**:
- `docs/개발문서/S3/phase2/산출물/Pre-flight_실측.md`
- `docs/개발문서/S3/phase2/산출물/블로커2_그래프라이브러리결정서.md` ✅ (cytoscape.js 채택)
- `task2-1.md` ~ `task2-3.md` (컬렉션·폴더·태그·백링크 데이터 소스)

---

## 1. 작업 목적

문서 목록 화면에 **리스트 / 트리 / 카드 / 그래프 4종 레이아웃** 토글을 제공. 그래프 뷰는 문서를 노드, 백링크를 엣지로 표현하며, 선택적으로 태그/컬렉션을 메타노드로 포함. 폐쇄망 환경에서 cytoscape.js 로컬 번들로 동작. 뷰포트 컬링·단계별 렌더·초기 노드 수 상한으로 대규모 성능 확보.

---

## 2. 작업 범위

### 2.1 포함

1. **백엔드 — 그래프 데이터 API**
   - `GET /documents/graph?scope=mine&limit=500` — viewer Scope 에 포함된 documents + 관련 edges
     - **response** (JSON):
       ```json
       {
         "nodes": [{"id":"<uuid>","type":"document","title":"...","document_type":"..."}, ...],
         "edges": [{"source":"<uuid>","target":"<uuid>","type":"backlink"}, ...],
         "truncated": true,
         "total_nodes": 1234
       }
       ```
     - `type ∈ {"document","tag","collection"}`, edge `type ∈ {"backlink","tagged","in_collection"}`
   - 필터 쿼리: `?collection=<id>`, `?folder=<id>`, `?tag=<name>`, `?include_tag_nodes=true` 등
   - **ACL**: FG 2-0 documents.scope_profile_id 필터 필수. viewer Scope 밖 노드는 반환 안 함
   - 노드 수 상한 500 기본, max 2000. truncated=true 표시

2. **프런트엔드 — 레이아웃 토글 컴포넌트**
   - `features/documents/LayoutToggle.tsx` — 4 버튼 (list / tree / cards / graph)
   - 단축키: `⌘/Ctrl + Alt + 1/2/3/4`
   - URL state: `?layout=list|tree|cards|graph`
   - user preferences (`users.preferences.documents.default_layout`) 에 마지막 선택 저장 — `useUserPreferences` 훅 재사용 (Phase 1 FG 1-3 산출물)

3. **프런트엔드 — 4 종 레이아웃 본체**
   - **list**: 기존 `/documents` 리스트 (TanStack Table 등)
   - **tree**: 폴더 계층 기반 트리 (FG 2-1 폴더 데이터 소비) — cytoscape **없음**, 리액트 트리 컴포넌트
   - **cards**: CSS Grid 카드 뷰 (제목/요약/수정일/태그 뱃지)
   - **graph**: cytoscape.js + dynamic import

4. **프런트엔드 — 그래프 뷰**
   - `features/documents/GraphView.tsx` — `next/dynamic(() => import('./GraphView.impl'))` 로 청크 분리
   - `GraphView.impl.tsx`:
     - `cytoscape` + `cytoscape-cose-bilkent` + `cytoscape-dagre` 레이아웃 등록
     - 레이아웃 선택 드롭다운: `cose-bilkent` (기본) / `dagre` / `breadthfirst` / `concentric`
     - 노드 스타일: document 는 원형, tag 는 사각형, collection 은 다이아몬드
     - 엣지 스타일: backlink=실선, tagged=점선, in_collection=굵은 점선
     - 클릭 이벤트: document → `/documents/{id}`, tag → `?tag=<>`, collection → `?collection=<id>`
     - 줌/팬/선택. 선택 시 우측 미니 패널에 상세
   - 초기 렌더: cytoscape 의 뷰포트 culling 기본 동작 활용
   - "더 불러오기" 버튼 → limit 을 500 씩 증가 (max 2000), 재 fetch

5. **프런트엔드 — 번들 분리**
   - cytoscape 관련 패키지는 **그래프 페이지 진입 시에만 로드** (dynamic import)
   - 빌드 시 `next build` 출력의 chunk 분리 확인 + `docs/개발문서/S3/phase2/산출물/FG2-4_번들검증.md` 에 기록
   - 폐쇄망 검증: `npm install --prefer-offline` + `next build` 성공 확인, 네트워크 차단 상태에서 그래프 페이지 정상 렌더

6. **보안**
   - `npm audit --production --audit-level=critical` exit 0 필수 (블로커 2 결정서 §4.3)
   - CLAUDE.md 전역 §2 "critical 취약점 있는 라이브러리 사용 금지" 준수

### 2.2 제외

- 그래프 뷰에서 노드 편집 / 드래그 저장 — Phase 3 이후
- AI 기반 클러스터링 / 유사도 하이라이트 — Phase 3 이후
- 태그 co-occurrence 전용 뷰 — 본 FG 에 통합 (태그 노드 포함 옵션)

### 2.3 하드코딩 금지

- 레이아웃 이름 `"list"|"tree"|"cards"|"graph"` 는 상수 ES module 1 곳 정의 후 frontend 전역 참조
- 노드/엣지 타입 문자열도 동일

---

## 3. 선행 조건

- 블로커 2 결정서 승인 (cytoscape.js)
- task2-1 ~ 2-3 완료 — 컬렉션/폴더/태그/백링크 데이터가 쿼리 가능해야 그래프 노드·엣지 생성
- `users.preferences` (Phase 1 FG 1-3) 의 `documents` 하위 키 확장 — `default_layout: "list"|"tree"|"cards"|"graph"`

---

## 4. 구현 단계

### Step 1 — 그래프 데이터 API

1. `backend/app/services/graph_service.py` 신설
2. `GET /documents/graph` 라우터 (FastAPI `api/v1/documents.py` 확장 또는 별도 파일)
3. 쿼리: documents (scope_profile_id 필터) + document_links + document_tags + collection_documents
4. pytest: ACL 필터 준수, 상한 500/2000, truncated 플래그

### Step 2 — 레이아웃 토글

1. `LayoutToggle.tsx` — 4 버튼 세그먼트
2. `useSearchParams` 로 URL state + `useUserPreferences` 로 마지막 선택 저장
3. 단축키 등록 (`useEffect` + keydown)
4. node:test: 클릭/단축키/URL/preferences 모두 일관

### Step 3 — 4 종 레이아웃 본체

1. list: 기존 리스트 유지
2. cards: CSS Grid — 반응형 (desktop 4열, tablet 2열, mobile 1열)
3. tree: 폴더 트리 + 문서 leaf. 빈 폴더도 표시
4. graph: GraphView 스텁 (다음 단계에서 본체)

### Step 4 — cytoscape 설치 / 번들 분리

1. `npm install cytoscape @types/cytoscape cytoscape-cose-bilkent cytoscape-dagre dagre`
2. `npm audit --production --audit-level=critical` 녹색 확인 → `FG2-4_번들검증.md` 첫 섹션 기록
3. `GraphView.tsx` 는 shell 만, `GraphView.impl.tsx` 에 cytoscape import 집중
4. `next/dynamic` 으로 dynamic import

### Step 5 — GraphView 본체

1. cytoscape instance 생성 + `elements`, `style`, `layout` 설정
2. 데이터 fetch → cytoscape 에 주입
3. 레이아웃 전환: 드롭다운 선택 시 `cy.layout({name: '...'}).run()`
4. 클릭/더블클릭/hover 이벤트 바인딩
5. 우측 상세 패널 (선택된 노드 metadata)

### Step 6 — 성능 / 뷰포트

1. 500 노드 / 2000 엣지 기준 초기 렌더 5 초 이내 확인
2. cytoscape layout worker 옵션 활성화 (`cose-bilkent` 는 지원)
3. `layout.stop()` 으로 5 초 타임아웃 강제 종료

### Step 7 — UI 검수 / 반응형

- UI 리뷰 ≥ 1회
- desktop / narrow / tablet / mobile — tablet/mobile 에서 그래프 뷰는 축소 모드 (줌 in/out 컨트롤 큰 버튼)

### Step 8 — 검수 / 보안 보고서

- 일반 + 재검수
- 보안: "뷰 ≠ 권한 준수 확인" 섹션
  - A 가 그래프 뷰 열 때 B 의 문서 노드는 반환 안 됨
  - 태그 노드 클릭 시 필터도 Scope 적용
  - API 의 truncated 플래그 신뢰 (악의적 큰 limit 요청 시 서버가 상한 강제)

---

## 5. 폐쇄망 검증 체크리스트 (블로커 2 결정서 §4.2)

- [ ] `npm install --prefer-offline` 성공
- [ ] `next build` 후 `.next/static/chunks/` 에 cytoscape 청크 존재
- [ ] 빌드 output 에 외부 CDN fetch 없음 (cytoscape icon/font 포함 여부 확인)
- [ ] 네트워크 차단 상태에서 그래프 페이지 정상 렌더
- [ ] CSP 위반 경고 없음 (브라우저 콘솔 확인)

---

## 6. API 계약 요약

| 메서드 | 경로 | 설명 |
|-------|------|-----|
| GET | /documents/graph?limit=&include_tag_nodes=&collection=&folder=&tag= | 그래프 데이터 |

---

## 7. 성공 기준

- [ ] 4 종 레이아웃 전환 동작 (UI/URL/preferences 일관)
- [ ] 그래프 뷰 500 노드 / 2000 엣지 5초 이내 렌더
- [ ] 레이아웃 4종 (cose-bilkent/dagre/breadthfirst/concentric) 전환 성공
- [ ] 폐쇄망 번들 검증 체크리스트 녹색
- [ ] `npm audit` critical 없음
- [ ] ACL 회귀 녹색 (그래프가 권한 밖 문서를 드러내지 않음)
- [ ] pytest 신규 ≥ 15 건 (graph API)
- [ ] node:test 신규 ≥ 20 건 (레이아웃 토글 + 그래프 뷰 핵심 동작)

---

## 8. 리스크

| 리스크 | 대응 |
|-------|-----|
| 초기 번들 크기 증가 | dynamic import + tree-shaking. 측정 기록 |
| 대형 그래프 layout 타임아웃 | `layout.stop()` 5 초 + 단계별 렌더 (incremental) |
| cytoscape-dagre → dagre CVE | 설치 시점 npm audit 로 차단. 발견 시 본 FG 중단 후 재검토 |
| 그래프 UI 모바일 사용성 | 축소 모드 + 큰 줌 버튼 + touch pan 지원 (cytoscape 기본) |
| 그래프 내 노드가 많을 때 클릭 오판 | hitbox 최소 20px + 선택 시 하이라이트로 시각 피드백 |

---

*작성: 2026-04-24 | FG 2-4 그래프 뷰 + 4 레이아웃*
