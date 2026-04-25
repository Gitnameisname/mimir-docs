# task2-1 — 수동 컬렉션 + 계층 폴더 (+ FG 2-0 선행: documents.scope_profile_id)

**작성일**: 2026-04-24
**Phase / FG**: S3 Phase 2 / FG 2-1 (+ FG 2-0 선행 포함)
**상태**: 착수 대기
**담당**: 백엔드 + 프런트
**예상 소요**: 3~4일
**선행 산출물**:
- `docs/개발문서/S3/phase2/산출물/Pre-flight_실측.md`
- `docs/개발문서/S3/phase2/산출물/블로커1_태그규약결정서.md` (참조)

---

## 0. 선행 작업 (FG 2-0) — documents.scope_profile_id 신설

Pre-flight 실측 §2.1 / §6 에서 확인된 바와 같이, `documents` 테이블에는 `scope_profile_id` 컬럼이 존재하지 않는다. 이 상태에서 컬렉션/폴더/태그/백링크/그래프가 documents 를 참조하면 **"뷰 ≠ 권한" 규칙이 검증 불가능**하다. 따라서 FG 2-1 본편을 시작하기 **전**에 아래를 선결한다.

### 0.1 스키마 변경

Alembic revision `s3_p2_documents_scope_profile_id`:

```sql
ALTER TABLE documents
    ADD COLUMN scope_profile_id UUID
        REFERENCES scope_profiles(id) ON DELETE SET NULL;

CREATE INDEX idx_documents_scope_profile_id
    ON documents(scope_profile_id) WHERE scope_profile_id IS NOT NULL;
```

### 0.2 Backfill

- `created_by` → `users.scope_profile_id` 로 조회해 채움
- `created_by` 가 NULL 또는 유저 매핑 실패 시 **시스템 기본 private 프로파일** 로 채움 (S2-5 `_lookup_user_scope_profile_id` 와 동일 패턴)
- dry-run: `SCHEMA_MIGRATION_DRY_RUN=1` 지원

### 0.3 쓰기 / 조회 경로 적용

- `documents_service.create` 에서 ActorContext 의 `scope_profile_id` 를 기본값으로 주입
- `DocumentsRepository._FILTER_FIELD_MAP` 에 `scope_profile_id` 추가 + `list()` 가 **viewer 의 effective scope set** 으로 필터링
- "폴백 경로(FTS, 로컬 모델)에서도 동일한 ACL 적용" (CLAUDE.md §4.1 S2) — 관련 검색 경로 전수 확인

### 0.4 회귀 테스트

- 다른 scope 의 user A / B 각 1명. A 가 만든 문서를 B 가 `GET /documents/{id}` 호출하면 **404** (403 아님, 존재 유출 방지)
- `GET /documents?...` 목록에서도 B 는 보이지 않음
- FTS / hybrid retrieval 에서도 동일하게 필터됨 (pytest `test_documents_scope_profile_filter.py`)

---

## 1. 작업 목적

사용자가 만들고 관리하는 **임의 문서 집합(컬렉션)** 과 **계층 폴더 트리** 를 도입해, 사용자가 자기 맥락으로 문서를 묶어 본다. 두 구조 모두 **순수 뷰 레이어** — ACL 은 Scope Profile 단독이 결정.

---

## 2. 작업 범위

### 2.1 포함

1. **DB 스키마 (Alembic revision `s3_p2_collections_and_folders`)**
   - `collections (id UUID pk, owner_id UUID FK users, name VARCHAR(200), description TEXT, created_at, updated_at)`
   - `collection_documents (collection_id UUID FK, document_id UUID FK, position INT DEFAULT 0, added_at TIMESTAMPTZ, PRIMARY KEY (collection_id, document_id))`
   - `folders (id UUID pk, owner_id UUID FK users, parent_id UUID FK folders NULL, name VARCHAR(200), path LTREE or VARCHAR materialized, created_at, updated_at)` — 루트는 parent_id NULL
   - `document_folder (document_id UUID FK PK, folder_id UUID FK, assigned_at TIMESTAMPTZ)` — document 당 폴더 1개 (N:1, Optional)
   - 인덱스: `collection_documents(document_id)`, `folders(parent_id)`, `folders(owner_id)`, `document_folder(folder_id)`
   - **ACL 주의**: 이 테이블들은 **Scope Profile 과 독립**. owner_id 기반 접근 판단. 단 collection/folder 안에서 document 조회 시에는 FG 2-0 의 `documents.scope_profile_id` 필터가 자동 적용 (collection_documents → documents INNER JOIN 이 ACL 필터를 통과)

2. **백엔드 API**
   - `GET/POST /collections`, `GET/PATCH/DELETE /collections/{id}`
   - `POST /collections/{id}/documents`, `DELETE /collections/{id}/documents/{doc_id}`
   - `GET/POST /folders`, `GET/PATCH/DELETE /folders/{id}`, `POST /folders/{id}/move`
   - `PUT /documents/{id}/folder` (폴더 지정/해제)
   - `GET /documents?collection=<id>&folder=<id>` — 기존 list 에 필터 키 추가
   - 권한: **owner 만 편집**. 조회는 (a) owner 본인 또는 (b) admin. Phase 3 공유 Saved View 전까지는 조회도 owner 한정

3. **서비스 계층**
   - `collections_service.py`, `folders_service.py` 신설
   - 컬렉션 CRUD + 문서 추가/제거 (N:M)
   - 폴더 CRUD + 이동 (parent_id 변경 시 하위 path 재계산)
   - **폴더 이동이 ACL 에 영향 없음** — 문서 접근 판정은 `documents.scope_profile_id` 단독

4. **프런트엔드**
   - `features/explore/CollectionsTree.tsx` — 컬렉션 목록
   - `features/explore/FoldersTree.tsx` — 계층 트리 (접기/펼치기, drag & drop 재배치는 Phase 2 후속 또는 Phase 3)
   - `components/layout/Sidebar.tsx` — 중간 "탐색" 섹션에 위 2 컴포넌트 주입
   - `/documents` 페이지 — 쿼리 파라미터 `collection`, `folder` 지원 + 필터 뱃지 표시
   - 문서 상세 페이지 — "폴더 지정" 드롭다운 + "컬렉션에 추가" 버튼
   - 빈 상태 / 로딩 스켈레톤 / 필터인식 empty state (추출 결과 검토 큐 C 스코프 패턴 재사용)

### 2.2 제외

- 드래그-드롭 재배치 (Phase 2 후속 또는 Phase 3)
- 팀 공유 컬렉션 (S4 범위)
- 폴더 권한 상속 — **금지**. 폴더는 순수 뷰
- 스마트 컬렉션 (필터 저장) — FG 2-5 Saved View 범위

### 2.3 하드코딩 금지 재확인

- 컬렉션/폴더 "기본" 개수·이름 하드코딩 금지. 빈 상태는 `(비어있음)` 빈 배열.
- folder.path 계산은 DB trigger 또는 application 서비스 공용 util 로 — 각 API 라우터에 분산 금지

---

## 3. 선행 조건

- Phase 1 전체 공식 종결 (2,718 passed / services ≥ 80% / repositories ≥ 80%)
- 본 문서의 **§0 FG 2-0 선행** 완료
- 블로커 1 결정서 (태그) 승인 — FG 2-1 자체는 직접 참조 없지만 FG 2-2 착수 전 필수이므로 본 FG 진행 중 병행 확정

---

## 4. 구현 단계

### Step 1 — FG 2-0 선행 (documents.scope_profile_id)

1. Alembic revision `s3_p2_documents_scope_profile_id` 작성 + 로컬 upgrade 성공 확인
2. Backfill 쿼리 dry-run 으로 건수 출력 → 운영자 승인 후 실행
3. `DocumentsRepository.list` WHERE 에 scope_profile_id IN 필터 추가 + 단위 테스트
4. ActorContext 경로 4곳(S2-5 `_lookup_user_scope_profile_id` 위치) 에서 생성 시 기본값 주입 확인
5. FTS / hybrid retrieval 경로 점검 — `backend/app/services/retrieval/` 의 `fts_retriever.py`, `hybrid_retriever.py`, `vector_retriever.py` 에서 scope_profile_id 필터가 documents 조인에 포함되는지 확인
6. 회귀 테스트 `test_documents_scope_profile_filter.py` 작성 (최소 4 시나리오)

### Step 2 — 컬렉션 CRUD

1. Alembic revision `s3_p2_collections_and_folders` 로 4 테이블 일괄 생성
2. `app/models/collection.py`, `app/models/folder.py` dataclass 추가
3. `app/repositories/collections_repository.py`, `folders_repository.py` 신설 (CRUD + filter by owner)
4. `app/services/collections_service.py`, `folders_service.py` 신설
5. `app/api/v1/collections.py`, `folders.py` 라우터 신설 — FastAPI
6. pytest 유닛 + integration 테스트

### Step 3 — 폴더 계층 / 이동

1. `folders.path` 컬럼 (materialized, `/` 구분자 예: `/루트/하위/`) 또는 PostgreSQL `ltree` 확장 — 둘 다 검토해 **DB 확장 없는 materialized path** 채택 (ltree 는 DB 확장 의존 생성)
2. 이동 시 자신의 path + 모든 하위 path 재계산 (재귀 CTE 또는 서비스 계층 순회)
3. 순환 참조 방지 (자신 또는 하위를 새 부모로 지정 금지)
4. 이동 테스트 — 3 레벨 깊이 트리에서 중간 노드 이동 시 손자 path 까지 갱신 확인

### Step 4 — 문서 ↔ 컬렉션 / 폴더 연결

1. `POST /collections/{id}/documents` — body: `{document_ids: [...]}`. 이미 들어있는 문서는 idempotent 처리 (ON CONFLICT DO NOTHING)
2. `PUT /documents/{id}/folder` — body: `{folder_id: UUID | null}`
3. 문서 목록 API 필터 키 추가 — `GET /documents?collection=<id>`, `GET /documents?folder=<id>&include_subfolders=true`
4. `include_subfolders=true` 는 `folders.path` prefix 매칭으로 구현

### Step 5 — 프런트엔드 Sidebar 확장

1. `Sidebar.tsx` 를 상/중/하 3 영역으로 재구조화 (기존 NAV_ITEMS 보존)
2. `features/explore/SidebarExploreTree.tsx` — 컬렉션 섹션 + 폴더 트리 섹션 세로 배치
3. 각 섹션은 React Query 로 데이터 fetch (staleTime 30s)
4. URL query 와 연동 (선택 시 `?collection=` 또는 `?folder=` 로 이동)

### Step 6 — UI 검수 / 반응형

- desktop / narrow desktop / tablet / mobile 4 관점 확인
- CLAUDE.md 프로젝트 §5.2 기준 **UI 리뷰 ≥ 1회** (Phase 1 종결 시 완화된 규정. 전역 5회보다 프로젝트 규정 우선)
- 스크린샷 저장: `산출물/FG2-1_UI리뷰로그.md`

### Step 7 — 검수 보고서 / 보안 보고서

- 검수 보고서 `FG2-1_검수보고서.md` + 재검수 1회
- 보안 보고서 `FG2-1_보안취약점검사보고서.md`
- **"뷰 ≠ 권한 준수 확인"** 섹션 필수 — 아래 회귀 테스트 결과 첨부
  - A 가 만든 컬렉션에 A 의 문서 + B 만 접근 가능한 문서를 "강제 추가" 시도 → FG 2-0 필터로 거부 확인
  - 폴더 이동이 문서 접근성 변화 없음
  - 컬렉션 공유 URL 을 복사해 권한 없는 C 가 열어도 문서 목록은 본인 Scope 필터로 비어 보임

---

## 5. 데이터 모델 주의사항

- 신규 테이블은 **필드 추가 수준**. 기존 documents 스키마는 §0 컬럼 1 개 추가 외 변경 없음
- documents.metadata JSONB 에 `tags` 는 FG 2-2 가 추가. 본 FG 는 metadata 를 건드리지 않음
- folder.path 유니크는 **owner_id + path** 조합

---

## 6. API 계약 요약

| 메서드 | 경로 | 설명 |
|-------|------|-----|
| GET | /collections | owner 본인의 컬렉션 |
| POST | /collections | {name, description?} |
| GET | /collections/{id} | |
| PATCH | /collections/{id} | |
| DELETE | /collections/{id} | |
| POST | /collections/{id}/documents | {document_ids} |
| DELETE | /collections/{id}/documents/{doc_id} | |
| GET | /folders | 계층 전체 (root-first, depth ≤ 10 상한) |
| POST | /folders | {parent_id?, name} |
| PATCH | /folders/{id} | {name} |
| POST | /folders/{id}/move | {new_parent_id} |
| DELETE | /folders/{id} | 하위 없을 때만 허용 (cascade 금지) |
| PUT | /documents/{id}/folder | {folder_id: null\|UUID} |
| GET | /documents?collection=&folder=&include_subfolders= | 기존 list 확장 |

모든 응답은 Scope Profile ACL 통과 후 반환. 쓰기 API 는 `require_admin` 또는 owner 검증.

---

## 7. 성공 기준

- [ ] FG 2-0 선결 완료 — `documents.scope_profile_id` 컬럼 + backfill + list 필터 + 회귀 테스트 녹색
- [ ] 컬렉션 / 폴더 CRUD API 전 경로 녹색
- [ ] 프런트 Sidebar 에 컬렉션 + 폴더 트리 표시, 선택 시 `/documents` 가 필터됨
- [ ] 폴더 이동 테스트 (3 레벨 깊이) 녹색
- [ ] **뷰 ≠ 권한** 회귀 테스트 녹색
- [ ] pytest 베이스라인 (2,718+) 유지, 신규 테스트 ≥ 30 건
- [ ] frontend node:test 신규 ≥ 15 건, 전체 녹색
- [ ] UI 리뷰 로그 제출
- [ ] 검수·재검수·보안 보고서 제출

---

## 8. 리스크

| 리스크 | 대응 |
|-------|-----|
| FG 2-0 backfill 중 용량 큰 documents 테이블 lock | 배치 크기 1000 rows + LIMIT/OFFSET, 야간 배포 |
| folder.path 재계산 누락 | 이동 시 하위 전체 재계산 대상 테스트 3 레벨 깊이까지 |
| 순환 참조 폴더 | parent_id 체인 순회 체크 서비스 계층에서 강제 |
| `/documents?include_subfolders=true` 성능 | path prefix 인덱스 (`LIKE '/A/B/%'`) + 깊이 상한 10 |
| ACL 회귀 | FG 2-0 회귀 테스트가 본 FG 의 진입 게이트 |

---

*작성: 2026-04-24 | FG 2-0 선행 + FG 2-1 본편*
