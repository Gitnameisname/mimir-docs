# Phase 2 Pre-flight 실측 보고서

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-24 |
| 목적 | Phase 2 작업지시서 작성을 위한 현황 실측. 컬렉션·폴더·태그·백링크·그래프·Saved View·Vault Import 도입 시 ACL 필터와 `snapshot_sync_service` 확장 지점 확인 |
| 근거 | 2026-04-24 기준 `mimir/backend`, `mimir/frontend` 트리 직접 grep / read |
| 상위 규약 확인 | `docs/규칙/` 경로는 실제로 비어있음 — CLAUDE.md §0 참조는 공란. AGENTS.md + CLAUDE.md 프로젝트 §5.2 (UI 리뷰 ≥ 1회) 를 상위 규약으로 적용 |

---

## 1. 핵심 결론 (먼저 요약)

1. **`documents` 테이블에 `scope_profile_id` 컬럼이 존재하지 않는다.** 현재 `DocumentsRepository._FILTER_FIELD_MAP` 은 `status`, `document_type`, `owner_id(=created_by)` 만 허용. 즉 **documents 조회 API 는 현재 Scope Profile 기반 ACL 을 적용하지 않는다**.
2. Scope Profile 바인딩은 `extraction_candidate`, `extraction_record`, `conversations`, `rag_*` 등 **개별 도메인 테이블** 에만 `scope_profile_id` 컬럼으로 분산되어 있다. `users.scope_profile_id` (S2-5) 는 **Actor 소속** 을 가리키는 것이지 documents 에 기록된 Scope 가 아니다.
3. **Phase 2 의 "뷰 ≠ 권한" 절대 규칙은 이 갭을 가리지 못한다.** 컬렉션·폴더·태그·백링크가 documents 를 참조할 때 "어떤 Scope Profile 에 속한 document 인가" 를 판단할 수 있어야 컬렉션/폴더/태그에 ACL 필터가 걸린다. 현재는 그 전제가 없음.
4. **결론:** Phase 2 FG 2-1 착수 전에 **FG 2-0 (선행)** 으로 `documents.scope_profile_id` 컬럼 신설 + backfill + Repository/Service 필터 적용이 필요하다. 또는 FG 2-1 의 범위를 확장해 그 안에 포함시킨다. 이 결정을 본 Pre-flight 의 §6 에서 제안한다.
5. **`snapshot_sync_service`** 는 ProseMirror ↔ flat nodes 양방향 변환을 이미 제공. FG 2-2 `#tag` 파싱과 FG 2-3 `[[백링크]]` 파싱은 **`_collect_text`/`nodes_from_prosemirror` 의 text run 순회 지점에 텍스트 후킹 함수를 추가** 하는 형태로 얹을 수 있다 — 이 패턴이 Phase 1 의 "단일 쓰기 경로에서 파생 동기화" 를 그대로 재사용.
6. **TipTap 3.22 + `NodeId` extension** 이 Phase 1 에서 이식되어 있으므로, 태그/백링크를 **TipTap Mark** (인라인) + 서버측 파생 테이블 두 층으로 구현하는 설계가 자연스럽다. ProseMirror mark 는 `snapshot_sync_service` 가 이미 순회하므로 추가 테이블로 파생하기 쉬움.
7. **`frontend/src/components/layout/Sidebar.tsx`** 는 하드코딩된 `NAV_ITEMS` 배열 기반 단순 링크 네비. 컬렉션/폴더/태그 트리를 주입하려면 사이드바를 **상단 "앱 네비" + 하단 "탐색 트리 패널"** 2-영역으로 재구조화해야 한다. 기존 `NAV_ITEMS` 보존 + 하단에 동적 트리 삽입.
8. **그래프/사이그마/사이토스케이프** 패키지는 `frontend/package.json` 에 없음 → FG 2-4 착수 시 전량 신규 설치. 폐쇄망 번들 확인 필수.

---

## 2. documents 테이블 실측

### 2.1 DDL 현황 (`backend/app/db/connection.py` L30~67)

```sql
CREATE TABLE IF NOT EXISTS documents (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title       VARCHAR(500) NOT NULL,
    document_type VARCHAR(100) NOT NULL,
    status      VARCHAR(50) NOT NULL DEFAULT 'draft',
    metadata    JSONB NOT NULL DEFAULT '{}',
    summary     TEXT,
    created_by  VARCHAR(255),
    updated_by  VARCHAR(255),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- + Phase 4: current_draft_version_id, current_published_version_id
```

**`scope_profile_id` 컬럼 없음.** Alembic migrations/versions 전수 검색에서 documents 스키마 변경 이력에도 없음.

### 2.2 현재 ACL 경로

| 요소 | scope_profile_id 컬럼 |
|------|----|
| `users` | ✓ (S2-5 `20260420_1330_s2_5_users_scope_profile_binding`) |
| `scope_profiles`, `scope_definitions` | ✓ (본체) |
| `extraction_candidate`, `extraction_record`, `extraction_evaluation`, `golden_set` | ✓ |
| `conversations` | ✓ (RAG 대화) |
| `documents` | **✗ (없음)** |
| `versions`, `nodes` | ✗ (documents 를 따라가면 암묵 결정) |

### 2.3 DocumentsRepository.list 필터 (`documents_repository.py` L68~122)

```python
_FILTER_FIELD_MAP = {
    "status": "status",
    "document_type": "document_type",
    "owner_id": "created_by",  # owner_id query param → created_by 컬럼
}
```

`scope_profile_id` 필터 키 없음. WHERE 조건에 포함되지 않음.

---

## 3. snapshot_sync_service 확장 지점

### 3.1 구조 (`backend/app/services/snapshot_sync_service.py`)

| 함수 | 역할 |
|------|------|
| `is_valid_prosemirror_doc(snapshot)` | `type=="doc"` + `content: list` 검증 |
| `prosemirror_from_nodes(nodes)` | flat nodes → PM doc |
| `nodes_from_prosemirror(snapshot)` | PM doc → flat nodes (`_walk` 재귀) |
| `_collect_text(pm_node)` | PM 노드의 text 자식 이어붙이기 — **text run 순회 지점** |
| `_pm_block_to_flat(pm, parent_id, order)` | 블록 1 개를 flat dict 로 |
| `rebuild_nodes_from_snapshot(conn, version_id, snapshot)` | nodes 테이블 replace 호출 |

### 3.2 FG 2-2 #tag 파싱 얹는 위치

두 가지 후보:

**A. 서버 파생 (권장)**
- `snapshot_sync_service` 에 `extract_tags_from_snapshot(snapshot) -> list[str]` 신설
- `nodes_from_prosemirror` 순회와 동일 로직으로 text run 수집 → `#\w+` 정규식 매칭
- `rebuild_nodes_from_snapshot` 와 같은 트랜잭션에서 `rebuild_tags_for_version` 호출
- 장점: 서버에서 단일 규약 일관 강제. frontmatter 병행도 동일 함수 내에서 처리.
- 단점: 인라인 hashtag 를 사용자에게 **시각적으로 tag 처럼** 보여주려면 TipTap mark 가 별도로 필요.

**B. TipTap Mark (클라이언트 파싱)**
- `HashtagMark` TipTap extension 신설 → 저장된 PM doc 에 `marks: [{type: "hashtag", attrs: {name}}]` 저장
- 서버는 mark 만 수집하면 끝 (정규식 불필요)
- 장점: 사용자 입력 시 바로 시각화. 단점: TipTap 없는 import 경로(Vault Import) 에서 별도 파싱이 또 필요.

**결정 근거는 §블로커 1 결정서** 에서 확정. 선택에 따라 FG 2-2 설계가 달라진다.

### 3.3 FG 2-3 [[백링크]] 파싱 얹는 위치

동일 논리. **권장은 하이브리드**: TipTap 입력 편의는 Mark 로, 저장 후 백링크 그래프 인덱스는 서버에서 PM doc 순회로 재계산. Mark + 서버 파싱이 둘 다 돌면 정합성 이슈가 있어 **서버가 정본** 원칙을 명시.

---

## 4. Frontend Sidebar 확장 지점

### 4.1 현재 구조 (`frontend/src/components/layout/Sidebar.tsx`)

```tsx
const NAV_ITEMS: NavItem[] = [
  { href: "/documents", label: "문서", icon: ... },
  { href: "/reviews",   label: "검토 대기", icon: ... },
  { href: "/extractions", label: "추출 검토", icon: ... },
  { href: "/rag",       label: "AI 질의", icon: ... },
];
```

단순 링크 리스트. 계층 트리 구조 없음.

### 4.2 Phase 2 요구 구조

```
Sidebar
├─ 상단: 기존 NAV_ITEMS (전역 앱 네비)
├─ 중간: "탐색" 섹션
│   ├─ 컬렉션 (FG 2-1)
│   ├─ 폴더 트리 (FG 2-1)
│   ├─ 태그 (FG 2-2)
│   └─ Saved Views (FG 2-5)
└─ 하단: SidebarUserPanel (기존)
```

기존 `Sidebar.tsx` 는 그대로 두고, 새 컴포넌트 `SidebarExploreTree.tsx` 를 중간 영역에 주입. URL query 와 연동해 선택 상태를 유지.

### 4.3 접근 경로

- 컬렉션/폴더/태그 모두 `/documents?collection=X` / `?folder=Y` / `?tag=Z` 형태 query 로 필터링. Saved View 는 `/documents?view=<id>` 로 요약.
- URL state 는 이미 추출 결과 검토 큐 C 스코프에서 검증된 패턴 (URL state status/type/scope/page) 재사용.

---

## 5. 라이브러리 / 의존성 실측

### 5.1 Frontend package.json 관련 항목

```
"@tiptap/extension-placeholder": "^3.22.3",
"@tiptap/react":                 "^3.22.3",
"@tiptap/starter-kit":           "^3.22.3",
```

그래프 라이브러리 후보 **전무** — sigma.js / cytoscape.js / vis-network / dagre / cola / d3-force 모두 미설치. FG 2-4 착수 시 신규 설치 + 폐쇄망 번들 확인 필요.

### 5.2 Backend Python

- `psycopg2` 기반. 새 테이블 DDL 은 Alembic revision 으로만 추가.
- YAML frontmatter 파싱은 `PyYAML` 필요 — 현재 설치 여부 별도 확인 (FG 2-6 블로커 아님, 구현 시 확인).
- zip 안전 처리는 Python 표준 `zipfile` + 제한값 설정으로 충분. FG 2-6 zip bomb 방어는 `defusedxml` 범주가 아닌 자체 상한 검사.

---

## 6. FG 2-0 (선행) 제안 — documents.scope_profile_id 신설

### 6.1 필요성

Phase 2 의 핵심 규칙은 **"컬렉션·폴더·태그·Saved View 가 ACL 에 영향 없음"**. 이는 "**ACL 은 Scope Profile 단독**" 이 전제된 규칙. 현재 documents 는 Scope Profile 과 연결이 없으므로, 컬렉션/폴더/태그가 documents 를 참조할 때 뷰어의 Scope 로 필터링을 걸 수가 없다. 즉 **ACL 이 애초에 없는 상태를 "뷰가 ACL 을 건드리지 않는다" 로 오해** 할 위험.

### 6.2 최소 범위 제안

| 항목 | 내용 |
|------|------|
| 컬럼 | `documents.scope_profile_id UUID NULL REFERENCES scope_profiles(id) ON DELETE SET NULL` |
| 기본값 | 생성 시 `created_by` 의 `users.scope_profile_id` 를 복사 (기본 private 프로파일) |
| Backfill | 기존 documents 전체 — `created_by` 의 `users.scope_profile_id` 로 채움. `created_by` 가 NULL 이면 시스템 기본 private 프로파일 |
| 필터 적용 | `DocumentsRepository.list` WHERE 절에 `scope_profile_id IN (<viewer 의 effective profile set>)` 추가 |
| 쓰기 규정 | 문서 생성 시 scope_profile_id 결정 규약을 `ActorContext` 소유의 기본값으로 통일 (S2-5 `_lookup_user_scope_profile_id` 패턴 재사용) |
| 회귀 테스트 | "scope 가 다른 사용자가 같은 document_id 로 조회 시 404" 최소 1건 |

### 6.3 처리 위치 선택

| 선택지 | 내용 | 평가 |
|--------|------|-----|
| (a) FG 2-0 신설 | Phase 2 본편 6 FG 앞에 0 번을 넣어 이 작업만 분리 | 범위 명확. 단 Phase 개발계획서 §2 에 없던 FG 라 사용자 승인 필요 |
| (b) FG 2-1 확장 | FG 2-1 (컬렉션+폴더) 내부에서 선행 단계로 처리 | Phase 개발계획서 수정 없이 진행 가능. 단 FG 2-1 범위 비대 |
| (c) Phase 0 잔존 이월로 처리 | Phase 0 FG 0-3 "api.v1 overall 커버리지 gap" 과 같은 수준의 이월 항목 | 관리 모호. 비권장 |

**권장: (b)** — FG 2-1 의 첫 단계로 포함. 작업지시서 `task2-1.md` §0 "선행" 에서 명시.

---

## 7. 실측 근거 요약표

| 관찰 | 파일 / 위치 |
|------|----|
| documents DDL | `backend/app/db/connection.py` L30~68 |
| DocumentsRepository 필터 매핑 | `backend/app/repositories/documents_repository.py` L28~42 |
| `_build_list_query` | 같은 파일 L68~122 |
| Alembic head 목록 | `backend/app/db/migrations/versions/` — 최신 2 건: `20260424_1100_s3_p1_content_snapshot_backfill`, `20260424_1400_s3_p1_users_preferences` |
| scope_profile 사용처 | `extraction_candidate_repository.py`, `extraction_evaluation_repository.py`, `conversation_repository.py`, `api/v1/scope_profiles.py` |
| users.scope_profile_id 바인딩 | `20260420_1330_s2_5_users_scope_profile_binding.py` |
| snapshot_sync_service 주요 함수 | `backend/app/services/snapshot_sync_service.py` L78, 204, 266, 349, 386, 416 |
| NodeId extension | `frontend/src/features/editor/tiptap/extensions/NodeId.ts` |
| Sidebar | `frontend/src/components/layout/Sidebar.tsx` L16~73 (NAV_ITEMS) |
| 그래프 / hashtag 관련 패키지 부재 | `frontend/package.json` grep — sigma/cytoscape/vis-network/dagre/cola/d3-force/hashtag 0 건 |

---

## 8. FG 별 블로커 요약

| FG | 블로커 |
|----|--------|
| 2-0 (선행 제안) | 사용자 승인 — documents.scope_profile_id 신설 범위 확정 |
| 2-1 컬렉션 + 폴더 | FG 2-0 선결 후 진행 |
| 2-2 태그 | **블로커 1: 태그 규약** (인라인 `#tag` / frontmatter / 둘 다) |
| 2-3 백링크 | (블로커 2-2 결정 이후 자연 해결) + disambiguation 정책 |
| 2-4 그래프 | **블로커 2: 그래프 라이브러리** (sigma.js / cytoscape.js) + 폐쇄망 번들 |
| 2-5 Saved View | 뷰 json schema 정의 (FG 2-1~2-4 필터 키 집합 확정 이후) |
| 2-6 Vault Import | zip bomb 상한 (파일 수 / 크기 / 압축 비율) + PII 스캔 규약 |

---

*작성: 2026-04-24 | Phase 2 착수 Pre-flight — FG 2-0 선행 제안 포함*
