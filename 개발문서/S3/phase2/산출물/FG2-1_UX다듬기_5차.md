# FG 2-1 UX 다듬기 5차 — /search ACL/필터 포팅 + DocumentListPage URL 동기화

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-24 |
| 범위 | search_service 에 scope_profile_id wiring + collection/folder/include_subfolders 필터 포팅 + DocumentListPage 의 `?q=` URL 동기화 (북마크/뒤로가기) |
| 결과 | Chrome 실측 (URL 동기화 2/2 + 복합 URL 복원 1/1 + /search scope wire 1/1) PASS + **node:test 216 passed** + search_service smoke 7/7 |

---

## 1. 변경 / 신규 파일

### 1.1 백엔드

| 파일 | 변경 |
|------|------|
| `backend/app/services/search_service.py` | `_build_document_query` + `search_documents` + `search_documents_hybrid` 에 `collection_id` / `folder_id` / `include_subfolders` 3 파라미터 추가. documents_repository 와 **동일 subquery 규약**으로 `d.id IN (SELECT document_id FROM collection_documents ...)` / `JOIN folders f ON f.path LIKE ...` 서브쿼리 구성. 기존 `viewer_scope_profile_ids` 는 그대로 재사용 |
| `backend/app/api/v1/search.py` | `_resolve_viewer_scope_profile_ids(actor)` import + 상단에서 계산 후 `search_documents` / `search_documents_hybrid` 로 전달. 라우터 query params 에 `collection`, `folder`, `include_subfolders` 추가 |
| `backend/tests/unit/test_search_service_scope_and_assignment_filters.py` | 신규 10건 — scope 필터 3 (None/[]/[ids]) + collection/folder leaf/subfolders + 완전 결합 |

### 1.2 프런트엔드

| 파일 | 변경 |
|------|------|
| `frontend/src/features/documents/DocumentListPage.tsx` | - URL `?q=` 를 초기값으로 읽어 `search` / `filters.q` 초기화 <br>- debounce 후 `filters.q` 반영과 **동시에** `router.replace("/documents?...")` 로 URL 에 `q=` 반영 (다른 파라미터 보존) <br>- 뒤로가기/외부 URL 변경으로 `?q=` 가 바뀌면 input state 동기화하는 useEffect 추가 <br>- "필터 초기화" 가 `q` 도 URL 에서 제거 |
| `frontend/tests/DocumentListUrlSyncFg21Ux5.test.tsx` | 신규 7건 — `mergeSearchQuery` 규약 (빈→주입 / 기존 파라미터 보존 / 빈 q 제거 / 공백만·undefined 제거 / trim 후 URL 반영 / include_subfolders 보존) |

---

## 2. 설계 판단

### 2.1 search_service 가 documents_repository 와 규약을 맞춤

리스트 `?q=` 에서 쓰는 collection/folder subquery 구문을 search_service 에 **문자 단위 동일**하게 포팅. `d.id IN (SELECT document_id FROM collection_documents WHERE collection_id = %s)` / `JOIN folders ... f.path LIKE (SELECT path || '%%' FROM folders WHERE id = %s)`. 이렇게 두 경로가 **같은 SQL 의미** 를 갖게 해 "컬렉션 안 본문 검색" 과 "컬렉션 안 제목 검색" 이 일관된 결과 set 위에서 작동함.

### 2.2 `_resolve_viewer_scope_profile_ids` 재사용

documents_service 의 이미 검증된 Scope 해석 함수를 search 라우터에서도 호출. **admin role bypass / 일반 user [id] / scope 없음 [] 차단** 의 규약이 한 곳에 고정되어 세 경로 (documents 리스트 / documents 단건 / search) 가 같은 ACL 정책을 공유.

### 2.3 URL 동기화는 `replace` — history 스팸 방지

`router.push` 를 쓰면 키 입력마다 history entry 가 쌓임. `router.replace` 로 덮어써 **하나의 검색 세션 = 하나의 URL 상태**. 뒤로가기는 검색 직전 페이지로 자연 이동.

### 2.4 외부 URL 변경 반영

사용자가 뒤로가기/앞으로가기 또는 외부에서 `?q=` 포함 URL 로 진입하면 `urlQ` 가 바뀜. 이를 감지하는 useEffect 에서 `search` state 와 `filters.q` 를 동시 업데이트 → input 과 결과가 동기화.

### 2.5 /search UI 는 URL 만으로도 `q` 프리필

SearchPage 가 이미 `useSearchParams().get("q")` 로 초기값 사용. 이번 라운드에서 DocumentListPage 의 "본문까지 전체 검색 →" 링크가 현재 검색어를 prefill 하는 기반이 그대로 유효. **collection/folder 를 SearchPage UI 에도 노출하는 건 별도 라운드** — `DocumentSearchParams` 타입 확장 + 필터 UI 추가 필요.

---

## 3. Chrome 실측

| # | 단계 | 결과 |
|---|------|------|
| 1 | `/documents` 에서 `Audit` 입력 | 300ms 뒤 URL 이 `/documents?q=Audit` 로 자동 갱신 ✓ |
| 2 | 직접 URL `/documents?q=Audit&collection=<id>` 진입 | SearchInput 에 `Audit` 프리필 + 필터 뱃지 2개 (`검색: Audit`, `컬렉션: 검색테스트`) + AND 필터 결과 ✓ |
| 3 | 뒤로가기 | 이전 `/search?q=Audit` 로 복귀 ✓ |
| 4 | `/search?q=Audit` | FTS 결과 1건 `**Audit** Test` 하이라이트 (기존 scope 필터가 admin bypass 로 동작 — wiring 정상) ✓ |

---

## 4. 자동화 검증

### 4.1 node:test (frontend)

```
# tests 216   suites 42   pass 216   fail 0
```

신규 7건 `DocumentListUrlSyncFg21Ux5`:
- 빈 쿼리에 q 주입
- collection / folder 보존
- 빈 q 이면 URL 에서 제거
- 공백만 이면 제거
- undefined 이면 제거
- 앞뒤 공백 trim 후 URL 반영
- include_subfolders 등 다른 파라미터 완전 보존

### 4.2 search_service smoke (샌드박스 stub 7/7 PASS)

- scope None skip
- scope [] blocks (`1 = 0`)
- scope ids → `d.scope_profile_id IN (%s, %s)`
- collection_id → `collection_documents` subquery
- folder_id leaf → `document_folder` subquery
- folder_id + include_subfolders → `path LIKE` prefix subquery
- fully combined: status + scope + collection + folder+subfolders 전부 AND 결합

### 4.3 tsc --noEmit

FG 2-1 UX 5차 변경 파일 에러 0. 기존 `AdminUserDetailPage.tsx:279` 만 범위 밖.

### 4.4 운영자 실 pytest 대기

```
pytest tests/unit/test_search_service_scope_and_assignment_filters.py -v
```

예상 10건 전부 녹색.

---

## 5. 남은 후속

1. **SearchPage UI 에 collection/folder 필터 노출** — 백엔드는 완결, `DocumentSearchParams` 타입 확장 + 필터 컴포넌트만 붙이면 됨
2. **search_documents_hybrid 라우터 wiring 확인** — 훅은 넣었으나 hybrid 경로의 상위 호출자가 별도 테스트 필요
3. **FG 2-2 태그 백엔드 착수**

---

*작성: 2026-04-24 | FG 2-1 UX 다듬기 5차 완료, Chrome 실측 PASS + node:test 216 + search_service smoke 7/7*
