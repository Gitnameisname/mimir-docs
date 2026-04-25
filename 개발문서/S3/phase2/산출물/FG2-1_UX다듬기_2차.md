# FG 2-1 UX 다듬기 2차 — 현재 폴더/컬렉션 노출 + 빈 컬렉션 CTA

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-24 |
| 범위 | `GET /documents/{id}` 응답에 `folder_id` + `in_collection_ids` 포함 → DocumentAssignControls 가 현재 상태를 표시하고, 빈 컬렉션 맥락에서 "기존 문서 추가" 모달 + "새 문서 만들기" CTA 제공 |
| 결과 | Chrome 실측 전 단계 PASS (빈 CTA → 모달 → 문서 추가 → 목록 갱신 → 문서 상세 chip 표시 → X 제거 → 사이드바 뱃지 감소) + **node:test 198 passed** |

---

## 1. 변경 / 신규 파일

### 1.1 백엔드

| 파일 | 변경 |
|------|------|
| `backend/app/models/document.py` | Document dataclass 에 `folder_id: Optional[str]` + `in_collection_ids: list[str]` 필드 추가 |
| `backend/app/schemas/documents.py` | DocumentResponse 에 동일 필드 추가 — 문서 상세 응답 전용, 리스트 조회는 기본 `null/[]` |
| `backend/app/repositories/collections_repository.py` | `list_collection_ids_for_document(conn, document_id, owner_id)` 메서드 신설. **owner 범위 필터** 필수 — 타 owner 컬렉션은 반환 안 함 |
| `backend/app/services/documents_service.py` | `get_document` 이 actor 전달 시 `folders_repository.get_folder_of_document` + `collections_repository.list_collection_ids_for_document` 조합해 응답. actor=None 레거시 경로는 조회 생략 (성능) |
| `backend/tests/unit/test_documents_service_get_with_assignments.py` | 회귀 테스트 4건 (정상 / 빈 상태 / actor=None / Scope 밖 404 선결) |

### 1.2 프런트엔드

| 파일 | 변경 |
|------|------|
| `frontend/src/types/document.ts` | `Document` 타입에 `folder_id?: string \| null` + `in_collection_ids?: string[]` |
| `frontend/src/lib/api/documents.ts` | `adaptDocument` 를 **export** 로 승격 + 두 필드 보존 + 레거시 응답 호환 (기본 null/[]) |
| `frontend/src/features/explore/DocumentAssignControls.tsx` | `currentFolderId` / `currentCollectionIds` prop 추가. 폴더 드롭다운 기본값 주입 + 현재 포함된 컬렉션을 **chip 행**으로 렌더 (📋 + 이름 + X 버튼). 컬렉션 추가 드롭다운에서는 이미 포함된 항목 제외 |
| `frontend/src/features/documents/DocumentDetailPage.tsx` | `DocumentAssignControls` 호출에 `doc.folder_id` + `doc.in_collection_ids` 전달 |
| `frontend/src/features/explore/hooks/useCollections.ts` | `useAddDocumentsToCollection.onSuccess` / `useRemoveDocumentFromCollection.onSuccess` 에 **`["document", id]` + `["documents"]`** 무효화 추가 — 문서 상세와 리스트 양쪽 동시 갱신 |
| `frontend/src/features/explore/hooks/useFolders.ts` | `useSetDocumentFolder.onSuccess` 도 같은 이중 invalidate |
| `frontend/src/features/explore/AddDocumentsToCollectionModal.tsx` | 신규 — 최근 문서 50건 + 제목 검색 + 체크박스 multi-select + "N개 추가" 버튼. 이미 포함된 문서는 disabled + "이미 포함됨" 라벨. Scope 밖은 백엔드가 조용히 reject |
| `frontend/src/features/documents/DocumentListPage.tsx` | `urlCollection && activeCollection && 빈 결과` 때 기본 EmptyState 대신 전용 CTA: `"<name>" 컬렉션이 비어 있습니다` + `[기존 문서 추가]` / `[+ 새 문서 만들기]`. 모달 상태 `addDocsOpen` + PageContainer 하단에 모달 mount |
| `frontend/tests/DocumentsGetAdapterFg21Polish.test.tsx` | `adaptDocument` 5 시나리오 — 필드 보존 / null 유지 / 레거시 호환 / envelope 없는 평평 / 비배열 → 빈배열 정규화 |

---

## 2. 설계 판단

### 2.1 배치 상태는 **문서 상세 응답에만**

리스트 API 의 100건 각각에 폴더/컬렉션 조회를 붙이면 N+1 비용이 폭발. 상세 응답(`GET /documents/{id}`) 에서만 조합해 반환하고, 리스트는 기존처럼 가벼운 row 만. 프런트 `Document.folder_id` / `in_collection_ids` 는 상세 페이지에서만 채워짐을 주석으로 명시.

### 2.2 Owner 범위만 노출

`in_collection_ids` 는 **요청자 소유 컬렉션** 만 반환. 타인이 자기 컬렉션에 담았더라도 응답에 나타나지 않음. 같은 문서가 여러 뷰에서 독립적으로 조직화될 수 있다는 "뷰 레이어" 원칙을 응답 수준에서도 지킴. 테스트 `list_collection_ids_for_document` 의 쿼리가 `JOIN collections c WHERE c.owner_id = %s` 로 강제.

### 2.3 이중 invalidate

컬렉션/폴더 mutation 훅이 기존엔 `["documents"]` (리스트) + `[...collectionDocuments, id]` 만 무효화 → 문서 상세 `["document", id]` 캐시가 stale 로 남아 DocumentAssignControls 가 예전 상태 표시. 이번 라운드에 **세 지점 모두** invalidate:

- `["documents"]` (리스트)
- `["document", <document_id>]` (상세 — 새 invalidation)
- `[...collectionDocuments, <collection_id>]` (컬렉션 내 문서 id 목록)

### 2.4 빈 컬렉션 CTA 의 "새 문서 만들기" 는 쿼리 전달만

`/documents/new?collection=<id>` 로 링크만 연결. 실제 "생성 후 자동 컬렉션 연결" 로직은 NewDocumentPage 수정이 필요해 후속 라운드로. 이번 라운드에서도 일단 CTA 는 떠있고, 사용자가 신규 작성 후 상세에서 수동으로 컬렉션 드롭다운에서 추가할 수 있음.

### 2.5 모달의 검색은 클라이언트 측 필터

백엔드 list API 가 `q` 미지원. 최근 50건 fetch 후 client-side `title.toLowerCase().includes(q)` 필터. 50건 이상 컬렉션 모달 탐색 UX 는 서버 `q` 파라미터 지원 후 개선 예정 (FG 2-2 태그 백엔드와 병행 가능).

---

## 3. 검증

### 3.1 Chrome 실측 — 전 단계 PASS

| # | 단계 | 확인 |
|---|------|------|
| 1 | `?collection=<test>` 빈 상태 진입 | `"test" 컬렉션이 비어 있습니다` + `기존 문서 추가` / `+ 새 문서 만들기` 두 버튼 렌더 ✓ |
| 2 | `기존 문서 추가` 클릭 | `"test" 에 문서 추가` 모달 오픈, 최근 50건 목록 + 검색 + 체크박스 + "문서를 선택하세요" 힌트 ✓ |
| 3 | 문서 2건 체크 → `2개 추가` | 모달 닫힘 + 목록이 2건으로 갱신 + **사이드바 뱃지 `2`** 즉시 반영 ✓ |
| 4 | 해당 문서 상세 진입 | DocumentAssignControls 에 **`📋 test ✕`** chip 표시, 컬렉션 추가 드롭다운에서 "test" 제외 ✓ |
| 5 | chip 의 `✕` 클릭 | chip 사라짐 + **사이드바 뱃지 `1`** 로 즉시 감소 (문서 상세 `["document", id]` 캐시 invalidate 동작) ✓ |

### 3.2 자동화

```text
$ npm test
# tests 198   suites 38   pass 198   fail 0
```

신규 5건 `DocumentsGetAdapterFg21Polish`:
- envelope `{ data }` 에서 필드 보존
- `folder_id=null` 유지
- 필드 없는 레거시 응답 → 기본값
- envelope 없는 평평 응답 호환
- `in_collection_ids` 가 비배열 → `[]` 정규화

tsc --noEmit: FG 2-1 UX 2차 변경 파일 에러 0. 기존 `AdminUserDetailPage.tsx:279` 만 범위 밖으로 남음.

### 3.3 파이썬 회귀 (운영자 실 pytest 대기)

`test_documents_service_get_with_assignments.py` 4건 — 샌드박스에서 구조 py_compile 확인, 실 pytest 는 macOS 로컬 실행 대기.

---

## 4. 남은 후속 작업

1. **NewDocumentPage 가 `?collection=<id>` 쿼리 수신 시 생성 성공 후 자동 추가** — CTA 완결
2. **list API `q` 서버 지원** — 대량 문서에서 모달 검색 성능
3. **문서 상세에서 "편집" 모드 전환 시에도 chip 갱신 확인** — 편집/저장 왕복 회귀
4. **FG 2-2 태그 백엔드 착수** — `extract_tags_from_snapshot` + HashtagMark

---

*작성: 2026-04-24 | FG 2-1 UX 다듬기 2차 완료, Chrome 전 단계 실측 + node:test 198 녹색*
