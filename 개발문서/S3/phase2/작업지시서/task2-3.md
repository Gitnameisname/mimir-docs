# task2-3 — 백링크 `[[문서명]]`

**작성일**: 2026-04-24
**Phase / FG**: S3 Phase 2 / FG 2-3
**상태**: 착수 대기
**담당**: 백엔드 + 프런트
**예상 소요**: 2~3일
**선행 산출물**:
- `docs/개발문서/S3/phase2/산출물/Pre-flight_실측.md`
- `task2-1.md` (FG 2-0 선행)
- `task2-2.md` (HashtagMark 패턴 재사용)

---

## 1. 작업 목적

본문 인라인 `[[문서명]]` 참조를 **양방향 그래프 에지** 로 파싱하고, 문서 상세 페이지에 **"이 문서를 참조하는 문서"** 섹션을 제공한다. 이름 충돌 시 제목 + 경로 + ID fallback 으로 disambiguation. FG 2-4 그래프 뷰가 이 데이터를 소비한다.

---

## 2. 작업 범위

### 2.1 포함

1. **DB 스키마 (Alembic revision `s3_p2_document_links`)**
   - `document_links (id UUID pk, from_document_id UUID FK documents, to_document_id UUID FK documents NULL, node_id UUID, raw_text VARCHAR(500), resolved_status VARCHAR(16), created_at, PRIMARY KEY (id))`
   - `resolved_status ∈ {'resolved','ambiguous','missing'}`
   - UNIQUE (from_document_id, node_id, raw_text) — 같은 block 안의 같은 원문 중복 방지
   - 인덱스: `document_links(to_document_id)` (역방향 조회), `document_links(from_document_id)`

2. **백엔드 — 파서**
   - `backend/app/services/snapshot_sync_service.py` 에 신규:
     ```python
     def extract_wikilinks_from_snapshot(
         snapshot: dict | None,
     ) -> list[tuple[str, str]]:
         """(raw_text, node_id) 튜플 list 반환"""
     ```
   - 정규식: `\[\[([^\]\n|]{1,200})(?:\|([^\]\n]{1,200}))?\]\]`
     - `[[제목]]` 또는 `[[제목|표시이름]]` 둘 다 허용
     - 코드블록 노드 제외, link mark text 내부 제외
   - 단, **mark 기반 구현도 가능** — TipTap `WikiLinkMark` 가 인라인 입력 지점에서 mark 부여. 하지만 **파서가 정본** (블로커1 결정서 §3.2 동일 원칙).

3. **백엔드 — Resolver**
   - `backend/app/services/wikilink_resolver.py`:
     ```python
     def resolve_wikilinks(
         conn, from_doc_id, links: list[tuple[str, str]], viewer_scope_ids: set[str]
     ) -> list[DocumentLinkRow]:
         """raw_text 를 documents.title 과 매칭해 to_document_id 결정"""
     ```
   - 매칭 우선순위:
     1. viewer_scope_ids 내에서 title 완전 일치 문서 1 개 → `resolved`
     2. 완전 일치 복수 → `ambiguous` (해결 안 함, UI 에서 경고 pill)
     3. 일치 없음 → `missing` (UI 에서 회색 pill)
   - **ACL 반영**: 해석 시 viewer 의 Scope Profile 범위에서만 매칭. Scope 밖 문서 제목은 **절대 노출 안 함** (존재 유출 방지)

4. **백엔드 — 파생 동기화**
   - `rebuild_wikilinks_for_document(conn, from_doc_id, snapshot, viewer_scope_ids)` 함수
   - 저장 경로(`save_draft`, `publish_version`, `agent_proposal` 쓰기) 에서 `rebuild_nodes_from_snapshot` + `rebuild_tags_for_version` 직후 호출
   - **viewer_scope_ids 는 작성자 본인의 scope set** — "문서 작성자의 Scope 에서 해석" 원칙. 다른 뷰어가 볼 때는 다시 viewer 기준으로 재해석 (읽기 시 API 에서 disambiguation)

5. **백엔드 — API**
   - `GET /documents/{id}/backlinks` — 이 문서를 참조하는 문서 목록 (to_document_id = {id} 역조회, viewer Scope 필터)
   - `GET /documents/{id}/links` — 이 문서가 내보내는 링크 (디버깅용, 관리자만)
   - `GET /documents/resolve?q=<title>` — 자동완성 (TipTap WikiLinkMark InputRule 에서 소비)

6. **프런트엔드 — TipTap WikiLinkMark**
   - `features/editor/tiptap/extensions/WikiLinkMark.ts`
   - InputRule `/\[\[([^\]]+)\]\]$/` 패턴으로 mark 자동 부여
   - 입력 중 `[[` 타이핑 시 floating 자동완성 팝업 (documents.resolve 호출, 150ms debounce)
   - 렌더: 상태별 class (`wikilink-resolved`, `wikilink-ambiguous`, `wikilink-missing`)
   - 클릭 시 resolved → `/documents/{id}`, ambiguous → disambiguation 모달, missing → "새 문서 만들기" 프롬프트 (Phase 3 이후 연결 — 이 FG 에서는 `/documents/new?title=<>` 로 연결만)

7. **프런트엔드 — 백링크 섹션**
   - `features/documents/BacklinksPanel.tsx` — 문서 상세 페이지 하단
   - "이 문서를 참조하는 N 개 문서" 목록, 각 항목 클릭 시 해당 문서 + 참조 지점(node_id anchor) 으로 이동
   - 빈 상태 / 로딩 / 에러 상태

### 2.2 제외

- 링크 그래프 레이아웃 — FG 2-4 범위
- 이름 변경 시 자동 링크 갱신 — Phase 3 이후 (resolver 는 읽기 시점 재해석이므로 어느 정도 자연 해결)
- AI 기반 유사 문서 추천 — Phase 3 이후

### 2.3 하드코딩 금지

- disambiguation 표시 문구 ("해석 불가", "문서 없음") 는 i18n 키로. 지금은 한국어만 지원이지만 키 분리 원칙 적용.

---

## 3. 선행 조건

- task2-1 (FG 2-0) 완료 — documents.scope_profile_id 필수
- task2-2 HashtagMark 패턴 참고 (WikiLinkMark 가 구조상 유사)

---

## 4. 구현 단계

### Step 1 — 파서 유닛

1. `extract_wikilinks_from_snapshot` 구현 + pytest
2. 테스트: 단순 `[[제목]]`, pipe `[[제목|별명]]`, 코드블록 내 제외, link mark 내 제외, 중첩 `[[[[foo]]]]` 거부, 빈 `[[]]` 거부, 개행 포함 거부

### Step 2 — DB / Repository

1. Alembic revision `s3_p2_document_links`
2. `app/models/document_link.py` + `DocumentLinksRepository`
3. `replace_for_document(conn, from_doc_id, rows)` 구현

### Step 3 — Resolver

1. `wikilink_resolver.py` 구현 — `resolve_wikilinks(conn, from_doc_id, links, viewer_scope_ids)`
2. documents.title 인덱스 확인 (기존 `idx_documents_title` 존재 — Pre-flight §2.1)
3. 복수 후보 정렬: 우선 **최근 수정순**, 동점 시 제목 길이 오름차순
4. pytest: 단일 일치, 복수 일치, 없음, scope 밖 문서 제외

### Step 4 — 파생 동기화 연결

1. `rebuild_wikilinks_for_document` — save_draft / publish / agent 경로에서 호출
2. 트랜잭션 경계: nodes / tags / wikilinks 셋을 **하나의 `with get_db() as conn:` 블록** 안에서 순차 호출
3. integration: draft 에 `[[다른 문서]]` 포함 저장 → document_links 에 1건 resolved

### Step 5 — API

1. `GET /documents/{id}/backlinks` 라우터 — paginated, viewer Scope 필터
2. `GET /documents/resolve?q=` — prefix 매칭 상위 10 + ambiguous 정렬
3. 응답 캐시: 30s

### Step 6 — TipTap WikiLinkMark

1. `extensions/WikiLinkMark.ts`
2. floating 자동완성 팝업 (TipTap Floating UI 또는 커스텀)
3. 키보드 네비 ↑ ↓ Enter Esc
4. node:test: 입력 시 mark 부여, 자동완성 선택, 저장 후 JSON 에 mark 존재

### Step 7 — 백링크 패널

1. `BacklinksPanel.tsx` — React Query fetch
2. 클릭 시 해당 문서 node_id anchor 로 이동 (`/documents/{id}#node-<uuid>`)
3. 빈 상태 친절 메시지

### Step 8 — UI 검수

- UI 리뷰 ≥ 1회
- Mark 색상 (resolved/ambiguous/missing) 3 상태 대비 충분한가

### Step 9 — 검수 / 보안 보고서

- 일반 보고서 + 재검수
- 보안 보고서: **"뷰 ≠ 권한 준수 확인"** — A 가 B 의 문서 제목을 추측해 `[[<B의 문서 제목>]]` 입력 시 A 의 Scope 에서 resolved 안 됨 (missing 표시) → B 의 존재 유출 없음 검증

---

## 5. 주의사항

- Resolver 는 **모든 호출자 viewer 컨텍스트** 기준 재해석. 저장 시점의 resolved_status 는 힌트일 뿐, 읽기 시 viewer Scope 로 필터/재매칭 가능
- 이름 충돌 시 UI 는 resolved_status 만 보고 3 상태 pill 렌더. 구체 후보 list 는 disambiguation 모달에서만 fetch (초기 렌더 비용 절감)
- 순환 링크 (`A → B → A`) 허용 (그래프는 유향 그래프이므로 구조상 자연)

---

## 6. API 계약 요약

| 메서드 | 경로 | 설명 |
|-------|------|-----|
| GET | /documents/{id}/backlinks?page=&page_size= | 역참조 목록 |
| GET | /documents/{id}/links | admin 전용, 정방향 링크 |
| GET | /documents/resolve?q= | 자동완성 |

---

## 7. 성공 기준

- [ ] 파서 유닛 ≥ 15 건
- [ ] resolver 유닛 ≥ 10 건 (단일/복수/없음/ACL 필터)
- [ ] integration ≥ 5 건 (저장 → document_links 반영)
- [ ] API p95 < 150ms
- [ ] WikiLinkMark 입력/자동완성 동작
- [ ] 백링크 패널 빈/정상/에러 상태
- [ ] ACL 회귀 녹색 — B 의 문서 제목 유출 없음
- [ ] pytest 신규 ≥ 30 건
- [ ] node:test 신규 ≥ 20 건

---

## 8. 리스크

| 리스크 | 대응 |
|-------|-----|
| documents.title 중복 흔함 | fuzzy matching 대신 완전 일치 + ambiguous 표시로 명시. 사용자가 `[[제목|id:<uuid>]]` 형식도 허용해 강제 지정 가능하게 → v2 기능으로 이월 |
| 대형 문서에서 link 수백 개 | rebuild 배치 크기 200 + UNIQUE 제약으로 중복 흡수 |
| WikiLinkMark floating popup 접근성 | ARIA role=listbox, 키보드 네비 필수 |
| 이름에 특수문자 (Unicode) | title 은 VARCHAR(500) NOT NULL, 서버 정규화 없이 원문 비교. NFC 정규화 적용으로 한글 조합형/완성형 차이 흡수 |

---

*작성: 2026-04-24 | FG 2-3 백링크*
