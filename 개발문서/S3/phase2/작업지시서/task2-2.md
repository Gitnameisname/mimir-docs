# task2-2 — 태그 동적 그룹 (`#tag` 인라인 + frontmatter 병행)

**작성일**: 2026-04-24
**Phase / FG**: S3 Phase 2 / FG 2-2
**상태**: 착수 대기 (블로커 1 결정서 승인 후)
**담당**: 백엔드 + 프런트
**예상 소요**: 2~3일
**선행 산출물**:
- `docs/개발문서/S3/phase2/산출물/Pre-flight_실측.md`
- `docs/개발문서/S3/phase2/산출물/블로커1_태그규약결정서.md` ✅
- `task2-1.md` (FG 2-0 선행 포함)

---

## 1. 작업 목적

문서에 부여된 태그로 **동적 그룹(이 태그를 가진 문서들)** 을 제공한다. 태그는 **인라인 `#tag` 와 frontmatter `tags: [...]` 두 경로** 모두에서 수집되며, 서버의 `extract_tags_from_snapshot` 가 **유일한 정본 파서**이다. 자동완성, 정규화, 동적 그룹 뷰를 포함한다.

---

## 2. 작업 범위

### 2.1 포함

1. **DB 스키마 (Alembic revision `s3_p2_tags`)**
   - `tags (id UUID pk, name_normalized VARCHAR(64) UNIQUE, created_at)` — 전역 태그 풀 (Scope 와 무관)
   - `document_tags (document_id UUID FK, tag_id UUID FK, source VARCHAR(12) CHECK source IN ('inline','frontmatter','both'), created_at, PRIMARY KEY (document_id, tag_id))`
   - 인덱스: `document_tags(tag_id)` (역방향 조회)
   - ACL 은 **documents.scope_profile_id** (FG 2-0) 로 결정. `document_tags` 자체는 ACL 무관

2. **백엔드 — 태그 파서**
   - `backend/app/services/snapshot_sync_service.py` 에 아래 신규 함수:
     ```python
     def extract_tags_from_snapshot(
         snapshot: dict | None,
         metadata: dict | None,
     ) -> list[tuple[str, str]]:
         """(name_normalized, source) 튜플 list 반환. source ∈ {'inline','frontmatter','both'}"""
     ```
   - 인라인 파서: 블로커1 결정서 §3.4 의 정규식 사용. 코드블록 노드 제외, link mark text 제외
   - frontmatter 파서: `metadata.get("tags")` 가 list[str] 일 때만 수집. 그 외는 무시
   - `normalize_tag(raw)` util 함수 — 블로커1 §3.3 스펙 그대로
   - 반환 형식: 같은 태그가 두 소스에서 오면 `source='both'` 로 집약

3. **백엔드 — 파생 동기화**
   - `rebuild_tags_for_version(conn, document_id, version_id, snapshot, metadata)` 함수 신설
   - `document_tags` 의 해당 document 분 DELETE 후 새 집합 INSERT
   - **정본 쓰기 경로(`save_draft`) 에서 `rebuild_nodes_from_snapshot` 호출 직후 연쇄 호출** (같은 트랜잭션)
   - 버전별이 아닌 **document 단위** 로 유지 (최신 Published 또는 최신 Draft 중 어느 기준을 쓸지 결정 필요 — 권장: 최신 published 기준, draft 만 있을 경우 draft. 서비스 상수 `TAG_SOURCE_PREFER="published"` 설정화)
   - **폐쇄망 고려**: PyYAML 미설치 환경에서도 frontmatter 는 `metadata.tags` (이미 JSONB) 만 읽으므로 YAML 파싱 불필요 → 의존성 추가 없음

4. **백엔드 — API**
   - `GET /tags?q=<prefix>&limit=20` — 자동완성. `name_normalized LIKE 'q%'` + 사용 빈도순
   - `GET /tags/popular?limit=50` — 전체 사용 빈도 상위
   - `GET /documents?tag=<name>` — 기존 list 필터 확장
   - 사용자가 직접 `PATCH /documents/{id}` 로 `metadata.tags` 를 수정 가능. 저장 시 `rebuild_tags_for_version` 트리거
   - 관리자 전용: `DELETE /tags/{id}` (고아 정리 / 잘못된 태그 통합)

5. **프런트엔드 — 태그 편집 UI**
   - `features/tags/TagChipsEditor.tsx` — 문서 상세 상단 칩 위젯
     - `Tab` / `,` / `Enter` 로 칩 확정, 백스페이스로 마지막 칩 제거
     - React Query 로 자동완성 fetch (150ms debounce)
     - 저장 시 `PATCH /documents/{id}` (optimistic + rollback)
   - `features/editor/tiptap/extensions/HashtagMark.ts` — TipTap InputRule/PasteRule 기반 인라인 mark
     - 입력 중 `#word ` 패턴 인식 → mark 자동 부여
     - Mark 스타일: 반전 배경 / primary color / 클릭 시 `/documents?tag=<name>`
   - `features/tags/TagPill.tsx` — 본문 내 hashtag 렌더
   - Sidebar 중간 "탐색" 에 **태그 섹션** 추가: 상위 20 개 popular tag 표시, 클릭 시 `/documents?tag=<>`

### 2.2 제외

- AI 기반 태그 자동 추천 — Phase 3 이후
- 태그 그래프 (태그-태그 co-occurrence) — FG 2-4 그래프 뷰에 포함 검토
- 팀 공유 태그 / 권한별 태그 — S4 범위

### 2.3 하드코딩 금지

- 허용 문자 집합을 라우터/프런트에 각각 하드코딩하지 말고 `backend/app/services/tag_rules.py` 의 단일 source 를 읽어 `GET /tags/rules` 로 프런트에 제공 (또는 config 상수 ES 모듈 공유 → 빌드 타임 sync)

---

## 3. 선행 조건

- task2-1 의 FG 2-0 완료 (documents.scope_profile_id)
- 블로커1 결정서 승인 완료
- `snapshot_sync_service` 의 `_collect_text` / `nodes_from_prosemirror` 구조 숙지

---

## 4. 구현 단계

### Step 1 — 파서 / 정규화 util

1. `backend/app/services/tag_rules.py` — `normalize_tag`, `INLINE_HASHTAG_RE`, `MAX_LEN=64`
2. `extract_tags_from_snapshot(snapshot, metadata)` — PM doc 순회 + frontmatter 수집, 중복/대소문자 정규화
3. pytest 유닛: 인라인 단순, 중첩 `#ai/ml`, 코드블록 내 `#foo` 제외, link text 내 `#foo` 제외, `##` 제외, 한글 `#기획`, 공격 입력(초장문 `#` 반복, 유니코드 변종)

### Step 2 — DB 스키마 / Repository

1. Alembic revision `s3_p2_tags`
2. `app/models/tag.py` dataclass + `TagsRepository` / `DocumentTagsRepository`
3. `TagsRepository.upsert_many(conn, names_normalized)` — ON CONFLICT DO NOTHING
4. `DocumentTagsRepository.replace_for_document(conn, document_id, assignments)` — assignments = list[(tag_id, source)]

### Step 3 — 파생 동기화 연결

1. `save_draft` 흐름에 `rebuild_tags_for_version` 추가 (같은 트랜잭션)
2. `publish_version` 흐름에도 호출 (published 가 정본이 되는 경우)
3. `agent_proposal_service` 쓰기 경로도 포함 — agent 가 만든 draft 에서도 hashtag 수집
4. pytest integration: 인라인 태그를 포함한 draft 를 저장 → document_tags 에 반영 확인

### Step 4 — 자동완성 API

1. `/tags?q=` 엔드포인트
2. 사용 빈도 순으로 정렬 — `document_tags` GROUP BY count
3. 응답 캐시: `Cache-Control: public, max-age=30`
4. rate-limit: per-user 60 req/min (FastAPI dependency)

### Step 5 — 프런트 TipTap HashtagMark

1. `extensions/HashtagMark.ts` — InputRule `/(?:^|\s)#([\w/-]+)$/` 기반 mark 자동 부여
2. PasteRule 도 동일 패턴
3. 렌더: `<span class="tag-pill" data-tag="...">#...</span>`, 클릭 이벤트 전파
4. NodeId extension 과 호환 확인 (mark 만 추가하므로 node_id 불변)
5. node:test: 입력 시 mark 부여, 저장된 JSON 에 mark 포함, 클릭 시 router push

### Step 6 — 프런트 칩 위젯 / 사이드바 태그 섹션

1. `TagChipsEditor.tsx` — 자동완성 드롭다운, 키보드 네비 (↑ ↓ Enter Esc)
2. `Sidebar` "탐색" 에 태그 섹션 추가 — popular top 20
3. `/documents?tag=<>` 진입 시 필터 뱃지 표시 (X 로 제거)
4. node:test: 칩 입력/제거/자동완성 선택, 태그 클릭 시 라우팅

### Step 7 — UI 검수 / 반응형

- UI 리뷰 ≥ 1회 (desktop / narrow / tablet / mobile)
- 칩 위젯이 좁은 화면에서 줄바꿈 되는지, 본문 내 pill 이 clickable area 충분한지

### Step 8 — 검수 / 보안 보고서

- `FG2-2_검수보고서.md` + 재검수 1회
- `FG2-2_보안취약점검사보고서.md` — 입력 정규식 DoS, 유니코드 호모글립, `metadata.tags` 가 list 가 아닐 때 방어, rate-limit 통과 여부
- **"뷰 ≠ 권한 준수 확인"** 섹션: 태그 필터가 ACL 우회 수단이 되지 않는가 (B 가 A 의 태그를 알고 `/documents?tag=<>` 호출해도 B 의 Scope 밖 문서는 안 보임)

---

## 5. API 계약 요약

| 메서드 | 경로 | 설명 |
|-------|------|-----|
| GET | /tags?q=&limit= | 자동완성 |
| GET | /tags/popular?limit= | 인기 태그 |
| GET | /documents?tag=<name> | 태그 필터 |
| PATCH | /documents/{id} | `metadata.tags` 변경 허용 |
| DELETE | /tags/{id} | admin 전용, 연관 document_tags cascade |

---

## 6. 성공 기준

- [ ] 파서 유닛 ≥ 15 건 녹색 (일반/코드블록/link/heading/한글/공격 포함)
- [ ] integration: draft 저장 → document_tags 갱신 ≥ 5 시나리오
- [ ] 자동완성 API 응답 p95 < 100ms (1만 태그 가정)
- [ ] TipTap HashtagMark 입력/paste 동작
- [ ] 칩 위젯: 추가/제거/자동완성 선택
- [ ] Sidebar 태그 섹션: popular 20 표시 + 필터 연동
- [ ] ACL 회귀 ("태그가 ACL 우회 안 됨") 녹색
- [ ] pytest 베이스라인 유지, 신규 ≥ 25 건
- [ ] node:test 신규 ≥ 18 건

---

## 7. 리스크

| 리스크 | 대응 |
|-------|-----|
| 정규식 DoS | 길이 상한 문자열 사전 검사 + non-backtracking 정규식 구성 |
| Mark 와 서버 파서 불일치 | 서버 파서가 정본. 저장된 doc 에서 재파싱해 대소문자 / 정규화 일관 유지 |
| 자동완성 테이블 스캔 비용 | `tags(name_normalized text_pattern_ops)` 인덱스 + LIMIT 20 |
| frontmatter 의 malformed tags | list[str] 외 모두 무시 + 경고 로그 |
| hashtag 사용자 개인정보 누출 | popular 목록에 최소 3 문서 이상에서 쓰인 태그만 노출 (k-anonymity=3) |

---

*작성: 2026-04-24 | FG 2-2 태그 동적 그룹*
