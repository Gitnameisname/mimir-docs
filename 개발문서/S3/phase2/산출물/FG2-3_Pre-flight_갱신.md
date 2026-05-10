# FG 2-3 Pre-flight 갱신 메모 (재진입 전 사실 baseline)

**작성일**: 2026-05-10
**대상 작업지시서**: `docs/개발문서/S3/phase2/작업지시서/task2-3.md` (작성일 2026-04-24)
**목적**: 작업지시서 작성 후 Phase 3 ~ Phase 4 가 코드베이스를 진화시켰다. 본 메모는 task2-3.md 의 사실 baseline 을 2026-05-10 시점으로 갱신해 구현 진입 시 충돌을 차단한다.

---

## 1. 진행 상태 실측

### 1.1 미구현 확인

- ❌ `document_links` 테이블 (Alembic revision `s3_p2_document_links`) — 부재
- ❌ `app/services/snapshot_sync_service.py::extract_wikilinks_from_snapshot` — 부재
- ❌ `app/services/wikilink_resolver.py` — 부재
- ❌ `app/repositories/document_links_repository.py` (또는 동등) — 부재
- ❌ `app/models/document_link.py` — 부재
- ❌ `GET /documents/{id}/backlinks` / `GET /documents/{id}/links` / `GET /documents/resolve` — 부재
- ❌ `frontend/src/features/editor/tiptap/extensions/WikiLinkMark.ts` — 부재
- ❌ `frontend/src/features/documents/BacklinksPanel.tsx` — 부재

### 1.2 의도가 코드 docstring 에 박혀있는 흔적

`backend/app/services/snapshot_sync_service.py:425-431` (`rebuild_tags_for_document` docstring):

```
작업지시서 상호검증 §7 트랜잭션 규약:
  1) rebuild_nodes_from_snapshot  (FG 1-1)
  2) rebuild_tags_for_document     (FG 2-2, 이 함수)
  3) rebuild_wikilinks_for_document (FG 2-3, 후속)
세 함수 모두 동일 with get_db() as conn: 블록에서 호출되어야 한다.
```

→ FG 2-3 의 함수 시그니처와 호출 순서가 이미 약속되어 있음. 본 작업이 이 약속을 충족.

---

## 2. 갱신 항목

### 2.1 Alembic head — `s3_p2_document_links` 의 down_revision 변경

**task2-3.md 의 가정**: down_revision 미명시 (작성 시점 head = `s3_p2_tags`)

**현 head (실측)**: `s3_p4_agent_prop_idempotency` (2026-04-28)

**Migration chain (현재)**:
```
s3_p4_agent_prop_idempotency        (2026-04-28, head)
└─ s3_p4_scope_profile_allow_tools  (2026-04-28)
   └─ s3_p2_tags                    (2026-04-25, FG 2-2)
      └─ s3_p2_collections_and_folders  (FG 2-1)
         └─ s3_p2_documents_scope_profile (FG 2-0)
            └─ s3_p1_users_preferences
               └─ s3_p1_content_snapshot_backfill
                  └─ s3_p0_embedding_dim_check
```

**갱신 결정**:
- 새 revision `s3_p2_document_links` 의 `down_revision = "s3_p4_agent_prop_idempotency"` (현 head 위에 쌓는다)
- revision id 는 `s3_p2_document_links` 그대로 (의미 명확). VARCHAR(32) 한도 OK (24 chars)
- 파일명: `20260510_HHMM_s3_p2_document_links.py` (날짜 prefix는 작성 시각)

> 운영자 마이그레이션 적용 직전 **`@최철균` P1 승인 필수** (Phase 5 §8 / AGENT_MODE.md §3.3).

### 2.2 저장 경로 통합 지점 — 3곳 (task2-3.md 의 "save_draft / publish / agent" 약식 표현 정밀화)

`rebuild_wikilinks_for_document` 호출이 필요한 지점:

| # | 파일 | 라인 | 호출 chain (현재) |
|---|------|-----|-----------------|
| 1 | `backend/app/services/draft_service.py` | 230~247 | `rebuild_nodes_from_snapshot` → `rebuild_tags_for_document` → `rebuild_annotation_anchoring` |
| 2 | `backend/app/services/agent_proposal_service.py` | 152~167 | 동일 chain |
| 3 | `backend/app/services/documents_service.py` | 290~302 | `rebuild_tags_for_document` 만 호출 (publish 경로 — `metadata` 갱신 시 태그 재계산만) |

**갱신 결정**:
- 지점 1, 2: `rebuild_tags_for_document` **직후** + `rebuild_annotation_anchoring` 직전에 `rebuild_wikilinks_for_document` 추가 (snapshot_sync_service docstring 의 약속 순서 그대로)
- 지점 3 (publish 경로): publish 가 content_snapshot 자체를 변경하지 않는 경우 wikilink rebuild 불필요. **content_snapshot 변경 동반 publish 가 가능한 경로인지** Step 4 진입 시 확인 후 결정 (변경 동반이면 추가, 아니면 생략 + 보고서에 사유 기록)

### 2.3 viewer_scope_profile_ids 헬퍼 정본

**task2-3.md 의 표현**: "`viewer_scope_ids` 는 작성자 본인의 scope set" (line 61)

**현 정본 (실측)**: `backend/app/services/documents_service.py:40` 의 `_resolve_viewer_scope_profile_ids(actor)` 헬퍼

**갱신 결정**:
- `wikilink_resolver.resolve_wikilinks(*, conn, from_doc_id, links, viewer_scope_profile_ids)` 시그니처 채택 — keyword-only required (S2 ⑥ Scope 하드코딩 금지, FG 2-2 의 `document_ids_for_tag` 패턴 동일)
- 호출자가 `_resolve_viewer_scope_profile_ids(actor)` 결과를 그대로 전달
- 쓰기 경로 (save_draft / agent_proposal) 에서는 **작성자 본인의 scope set** 을 전달 (task2-3.md §2.1 line 61 의 "작성자 본인의 scope set" 원칙 유지)
- 읽기 경로 (`/backlinks`, `/resolve`) 에서는 **viewer 의 scope set** 을 전달 (재해석 — task2-3.md §5 line 158 "Resolver 는 모든 호출자 viewer 컨텍스트 기준 재해석")

### 2.4 documents.title 인덱스 — 이미 존재

**task2-3.md Step 3.2 의 가정**: "기존 idx_documents_title 존재 — Pre-flight §2.1"

**실측**: `backend/app/db/connection.py:52` 에 `CREATE INDEX IF NOT EXISTS idx_documents_title` 존재. 갱신 불필요.

> 단, prefix 매칭 (`LIKE 'q%'`) 의 인덱스 활용 가능성은 Step 5 의 `/documents/resolve` 구현 시 EXPLAIN 으로 검증.

### 2.5 ScopeProfile 컬럼 변동 (Phase 3 FG 3-2 + Phase 4 FG 4-0)

`scope_profiles` 테이블이 task2-3.md 작성 후 다음 컬럼이 추가됨:
- `settings_json` — Phase 3 FG 3-2 (boot-time DDL `IF NOT EXISTS`, Alembic 아님)
- `allowed_tools` — Phase 4 FG 4-0 (Alembic `s3_p4_scope_profile_allow_tools`, JSONB NOT NULL DEFAULT '[]')

**갱신 결정**: 본 FG 의 wikilink resolver 는 `scope_profiles` 의 위 두 컬럼과 **무관**. resolver 는 `documents.scope_profile_id` 와 viewer 의 `scope_profile_ids` 만 사용. 충돌 없음.

### 2.6 한국어 NFC 정규화

**task2-3.md §8 R-04**: "title 은 VARCHAR(500) NOT NULL, 서버 정규화 없이 원문 비교. NFC 정규화 적용으로 한글 조합형/완성형 차이 흡수"

**갱신 결정**:
- Python 표준 `unicodedata.normalize("NFC", s)` 적용 — 신설 모듈 대신 `app/utils/strings.py` (이미 존재 — 실측 시 확인) 에 `normalize_title(s) -> str` 추가 또는 인라인.
- documents.title 저장 시점에는 정규화 강제하지 않음 (기존 데이터 유지). resolver 가 비교 시점에 양쪽 NFC 변환 후 비교.

### 2.7 documents.metadata 의 frontmatter wikilinks 무시

**task2-3.md 명시 안 됨**: HashtagMark (FG 2-2) 는 frontmatter `tags:[]` 와 인라인 `#tag` 둘 다 정본화했다. 본 FG 는 `[[...]]` 인라인만 정본 — frontmatter 에 wikilinks 필드는 도입 안 함.

**갱신 결정**: 본 메모에 명시. `extract_wikilinks_from_snapshot(snapshot)` 만 사용, `metadata` 인자 받지 않음. 시그니처가 `rebuild_tags_for_document(conn, document_id, snapshot, metadata)` 와 다른 이유.

### 2.8 응답 캐시 30s

**task2-3.md Step 5.3**: "응답 캐시: 30s"

**갱신 결정**:
- 현재 backend 의 캐시 정책 (Phase 3/4 에서 valkey / scope_profile_policy 캐시 등 도입) 과 정합.
- 본 FG 의 `/backlinks` / `/resolve` 응답 캐시는 **viewer 별 캐시키** 필수 (R-A4 정합 — viewer scope 누설 방지). Step 5 진입 시 정확한 키 설계 (e.g., `wikilinks:backlinks:{doc_id}:{scope_hash}`).
- 캐시 도입은 1차 구현 후 결정 가능. Step 5 1차 구현은 캐시 없이 → p95 측정 → 필요 시 추가.

---

## 3. 작업 순서 갱신

task2-3.md §4 Step 1~9 그대로 진행하되, 각 Step 진입 시 본 메모의 §2.X 갱신 사항 적용.

| Step | task2-3.md | 본 Pre-flight 갱신 적용 |
|------|-----------|----------------------|
| Step 1 (파서) | 그대로 | §2.7 (metadata 미사용) |
| Step 2 (DB / Repository) | 그대로 | §2.1 (down_revision = head), §2.5 (scope_profiles 무관) |
| Step 3 (Resolver) | 그대로 | §2.3 (viewer_scope_profile_ids keyword-only), §2.6 (NFC 정규화) |
| Step 4 (파생 동기화) | 그대로 | §2.2 (3 지점 정밀 매핑) |
| Step 5 (API) | 그대로 | §2.4 (title 인덱스 활용), §2.8 (캐시 viewer key) |
| Step 6 (TipTap WikiLinkMark) | 그대로 | task5-1 ADR §a 표 와 정합 |
| Step 7 (BacklinksPanel) | 그대로 | DocumentDetailPage 사이드바 (FG 5-4 와 정합 — 본 FG 는 기존 stack 에 추가, FG 5-4 가 사이드바로 흡수) |
| Step 8 (UI 리뷰) | 그대로 | — |
| Step 9 (보고서) | 그대로 | — |

---

## 4. P1 승인 게이트

다음 시점에서 **`@최철균` 승인 필수** (헌법 제45조 / Phase 5 §8 / AGENT_MODE.md §3.3):

1. **Step 2 의 Alembic 적용 직전** — `alembic upgrade head` 실행 전. `--sql` dry-run 결과 첨부 후 승인.
2. **Step 5 의 API 표면 추가** — REST 엔드포인트 3종 신설. 사실상 Step 2 와 같은 PR 일 수 있으나, 별 항목으로 승인 기록.
3. **Step 9 의 종결 선언** — Phase 2 FG 2-3 공식 종결 + Phase 5 task5-1 ADR §a 표 (a) 옵션 충족 확인.

---

## 5. Phase 5 정합

본 FG 종결 시:
- `docs/개발문서/S3/phase5/작업지시서/task5-1.md` §2.1.1 ADR §a 표의 WikiLinkMark 행을 **(a) 본 단계에서 흡수 완료** 로 갱신
- task5-1 ADR §a 표의 직렬화 / 우선순위 / 결합 규칙은 본 FG 의 WikiLinkMark schema (Step 6) 결과를 baseline 으로 task5-1 ADR 작성 시 흡수
- 본 FG 의 `WikiLinkMark.ts` 가 task5-1 의 markNames 단일 정본 모듈 (`frontend/src/features/editor/tiptap/markNames.ts`) 의 입력이 됨

---

## 6. 검수 / 보안 보고서 추가 항목

task2-3.md §9 가 명시한 항목 외에 본 메모로 추가:

- **R2 (ACL 단일 결정점) 정합**: wikilink_resolver 가 `documents_service.get_document` 의 ACL 결정과 일관 — resolver 가 자체 ACL 판단 금지, viewer scope set 으로 필터만.
- **존재 유출 검증**: `[[B의 비공개 문서]]` 입력 시 A 가 missing pill 만 보고 B 의 제목 / 메타데이터 / 작성자 정보 일체 누설 없음. 통합 회귀 ≥ 3 시나리오.
- **NFC 정규화 회귀**: 한글 조합형 / 완성형 동일 입력 시 같은 문서로 resolved. ≥ 2 시나리오.
- **순환 링크**: A → B → A 저장 시 양방향 backlink 모두 정상 (task2-3.md §5 line 161).

---

## 7. 알려진 잔여 / 후속

- **이름 변경 시 자동 링크 갱신** — Phase 3 이후 별 라운드 (task2-3.md §2.2)
- **`[[제목|id:<uuid>]]` 강제 ID 지정 형식** — v2 기능 이월 (task2-3.md §8 R-01)
- **AI 기반 유사 문서 추천** — Phase 3 이후 별 라운드 (task2-3.md §2.2)
- **링크 그래프 레이아웃** — Phase 2 FG 2-4 미구현 (실측 결과 미진입). 본 FG 의 `document_links` 데이터를 소비할 후속 FG 로 명시.

---

*작성: 2026-05-10 | FG 2-3 재진입 전 Pre-flight 갱신*
