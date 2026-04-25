# FG 2-1 UX 다듬기 3차 — list API `q` 서버 지원 + NewDocumentPage 자동 연결 완결

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-24 |
| 범위 | `GET /documents?q=` 서버 측 ILIKE 제목 검색 + 모달 서버 검색 debounce + `/documents/new?collection=<id>` 수신 시 생성 후 자동 추가 |
| 결과 | Chrome 실측 전 시나리오 PASS + **node:test 204 passed** + `_build_list_query` q 스모크 8/8 PASS |

---

## 1. 변경 / 신규 파일

### 1.1 백엔드

| 파일 | 변경 |
|------|------|
| `backend/app/repositories/documents_repository.py` | `_build_list_query` 가 `q` 를 special key 로 꺼내 `title ILIKE %s ESCAPE '\\'` 조건으로 변환. `%` / `_` / `\` 를 사전 이스케이프해 LIKE 메타 문자 안전 처리. 공백만/빈 문자열/비문자열은 skip |
| `backend/app/api/v1/documents.py` | `_DOCUMENTS_SPEC.allowed_filter_fields` 에 `FilterFieldSpec(name="q")` 추가 |
| `backend/tests/unit/test_documents_list_q_filter.py` | 신규 — 7 건 (평범 키워드 / 공백 skip / 빈 skip / % 이스케이프 / _ 이스케이프 / \ 이스케이프 / 다른 필터와 결합 / 비문자열 무시) |

### 1.2 프런트엔드

| 파일 | 변경 |
|------|------|
| `frontend/src/lib/api/documents.ts` | `buildDocumentListParams` 가 `filters.q` 를 trim 후 `params.q` 로 전파. 기존 "q 미지원 → 전송 제외" 주석 제거 |
| `frontend/src/features/explore/AddDocumentsToCollectionModal.tsx` | 클라이언트 측 `filtered` 계산 제거 → **서버 q + 300ms debounce**. `useQuery` key 에 `listParams` (q 포함) 반영. 검색 중 `isFetching` 표시 라벨 "검색 중…" 추가. `items` 가 이미 서버 필터링 결과 |
| `frontend/src/features/documents/NewDocumentPage.tsx` | `useSearchParams()` 로 `collection=<id>` 읽고 `collectionsApi.get` 으로 이름 prefetch → 페이지 상단에 **📋 배너** ("생성 후 X 컬렉션에 자동으로 추가됩니다"). create 성공 후 `useAddDocumentsToCollection` 으로 자동 mutateAsync. 실패해도 문서는 생성됐으므로 편집 화면 이동 |
| `frontend/tests/DocumentsListQParamFg21Polish2.test.tsx` | 신규 — 6 건 (q URL 전송 / 공백 skip / 빈 skip / 없을 때 파라미터 없음 / collection/folder 와 조합 / trim 동작) |

---

## 2. 설계 판단

### 2.1 ILIKE + ESCAPE — 단순하지만 충분

FG 2-1 스코프에서는 제목 부분 일치가 목표이므로 full-text 인덱스까지는 과잉. 기존 `documents(title)` 인덱스로 **선두 매칭**은 이미 빠르고, 중간 매칭도 수만 건 이하에서 문제 없음. 다만 사용자 입력에 `%` / `_` 가 있으면 LIKE 메타 문자로 해석되어 의도치 않은 전체 매칭/단일 문자 매칭이 발생하므로 `ESCAPE '\\'` 와 사전 치환으로 **wildcard 주입 방어**.

### 2.2 q 는 "special key"

기존 `_FILTER_FIELD_MAP` 은 `col = %s` 단순 매핑용. 서브쿼리(collection/folder) 와 같은 자리에 q 도 올려 **generic loop 에서 제외** + 전용 분기로 처리. 보안상 type annotation (`isinstance(q_raw, str)`) 으로 비문자열 거부 — query spec 단에서도 검증되지만 repository 가 마지막 방어선.

### 2.3 모달은 debounce 300ms + isFetching 라벨

서버 호출을 키 입력마다 내지 않고 입력이 멈춘 후 300ms 기다림. 응답 도중에는 검색 input 우측에 `검색 중…` aria-live 폴라이트 라벨. 검색어가 비어지면 즉시 기본 최근 50건으로 복귀 (queryKey 에 listParams 통째로 들어가므로 React Query 가 캐시 분리).

### 2.4 NewDocumentPage 자동 연결의 실패 정책

생성 이후 컬렉션 연결이 실패해도 **문서 자체는 이미 생성**되어 있다. 문서를 롤백하지 않고 편집 화면으로 이동. 훅이 이미 실패 토스트를 띄우므로 사용자는 상세 페이지의 DocumentAssignControls 에서 수동 재연결 가능. "배너 확인 → 원하는 컬렉션 미 연결 시 사용자가 직접 조치" 경로 보장.

### 2.5 `useAddDocumentsToCollection` 이 invalidate 는 이미 2차에서 정비됨

- `["collections","list"]` document_count 증가 반영
- `["document", id]` 상세 캐시 stale
- `["documents"]` 목록 갱신 (빈 컬렉션 CTA 가 즉시 사라지도록)

---

## 3. Chrome 실측 — 전 시나리오 PASS

| # | 단계 | 결과 |
|---|------|------|
| 1 | 빈 컬렉션 "검색테스트" 진입 | 전용 CTA (`"검색테스트" 컬렉션이 비어 있습니다` + 두 버튼) 렌더 ✓ |
| 2 | `[기존 문서 추가]` → 모달 | 최근 문서 목록 표시 ✓ |
| 3 | `Audit` 검색어 입력 | debounce 후 서버 결과 **`Audit Test` 1건만** 노출 (이전 클라이언트 filter 대비 서버 ILIKE 동작 확인) ✓ |
| 4 | 모달 취소 → `[+ 새 문서 만들기]` | `/documents/new?collection=<d8bdc3fb...>` 로 이동 + 상단 배너 **`📋 생성 후 "검색테스트" 컬렉션에 자동으로 추가됩니다.`** ✓ |
| 5 | 제목 `자동 연결 테스트` 입력 → 생성 | 사이드바 `검색테스트` 뱃지 즉시 `1` 로 갱신 + 컬렉션 복귀 시 목록에 `자동 연결 테스트` 1건 표시 ✓ |

(edit 페이지 에러는 기존 `/documents/{id}/edit` 렌더 단에서 발생한 별건으로 FG 2-1 범위 밖.)

---

## 4. 자동화 검증

### 4.1 node:test

```
# tests 204   suites 39   pass 204   fail 0
```

신규 6건 `DocumentsListQParamFg21Polish2`:
- q URL 에 포함
- 공백만 → 전송 안 함
- 빈 문자열 → 전송 안 함
- 파라미터 없음 → URL 에 q 없음
- collection/folder 와 조합
- trim 동작 (`%20` 이 URL 에 실리지 않음)

### 4.2 백엔드 q 필터 스모크

`_build_list_query(q=...)` 를 샌드박스 stub 으로 실행:
- plain / whitespace skip / empty skip / `%` 이스케이프 / `_` 이스케이프 / `\` 이스케이프 / status + scope 조합 / 비문자열 무시 → **8/8 PASS**

### 4.3 tsc --noEmit

FG 2-1 UX 3차 변경 파일 에러 0. 기존 `AdminUserDetailPage.tsx:279` 만 범위 밖.

---

## 5. 운영자 실 pytest 대기

```
pytest tests/unit/test_documents_list_q_filter.py -v
```

예상: 8 건 모두 녹색. 회귀로 기존 q 관련 테스트와 충돌 없음.

---

## 6. 남은 후속 작업

1. **DocumentListPage 상단 SearchInput → 서버 q** — DocumentListPage 의 q 는 이미 `buildDocumentListParams` 를 통해 서버로 나감. 단 SearchInput UX (디바운스 / 포커스) 개선은 추후
2. **본문 검색** — 현재 q 는 제목 ILIKE 만. 본문 full-text 는 `search_service` 의 검색 API 와 병합 or FG 2-2 뒤 검토
3. **FG 2-2 태그 착수**

---

*작성: 2026-04-24 | FG 2-1 UX 다듬기 3차 완료, Chrome 전 시나리오 PASS + node:test 204 녹색 + q 스모크 8/8*
