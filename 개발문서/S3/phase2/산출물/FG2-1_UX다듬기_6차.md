# FG 2-1 UX 다듬기 6차 — 남은 이월 2건 해결

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-24 |
| 범위 | 컬렉션 description 편집 UI + SearchPage UI 에 collection/folder/include_subfolders 필터 노출 |
| 결과 | Chrome 실측 2/2 PASS (EditCollectionModal 저장 / /search 컬렉션 필터 AND 결합) + **node:test 229 passed** |

---

## 1. 변경 / 신규 파일

### 1.1 신규

| 파일 | 역할 |
|------|------|
| `frontend/src/features/explore/EditCollectionModal.tsx` | 이름 + 설명 풀 편집 모달. 기본값 = 현재 컬렉션 값. Trim + 1-200자 (이름) / 2000자 (설명) 상한. 저장 payload 계산 `computeUpdateBody` 를 `__test__` 대신 정규 export 로 공개 |
| `frontend/tests/EditCollectionAndSearchFilterFg21Ux6.test.tsx` | 13 건 (EditCollectionModal payload 9 + searchApi.documents URL 조합 4) |

### 1.2 수정

| 파일 | 변경 |
|------|------|
| `frontend/src/features/explore/CollectionsTree.tsx` | RowMenu 에 **"편집..."** 추가 (기존 인라인 rename 은 빠른 경로 유지). `editTarget` state + `<EditCollectionModal>` mount |
| `frontend/src/types/search.ts` | `DocumentSearchParams` 에 `collection?` / `folder?` / `include_subfolders?` 추가 |
| `frontend/src/features/search/SearchPage.tsx` | 초기값에 URL 의 `collection`/`folder`/`include_subfolders` 읽기 + URL→state 동기화 useEffect 확장 + `useCollections` / `useFolders` 훅으로 드롭다운 옵션 채움 + 필터 바에 2 select + "하위 폴더 포함" 체크박스 + "필터 초기화" 버튼 확장 + handleSearch 가 기존 URL params 보존 |
| `frontend/tsconfig.test.json` | test compile include 에 `EditCollectionModal` + `lib/api/search.ts` 추가 |

---

## 2. 설계 판단

### 2.1 "이름 변경" 인라인 + "편집..." 모달 **공존**

인라인 rename 은 빠른 경로로 유지 (키보드만으로 한 번 클릭 → 입력 → Enter). 설명까지 포함한 **풀 편집** 은 모달로 분리. 이렇게 하면 사용자 선택지 2 개:
- 이름만 빠르게: 인라인
- 이름 + 설명 한꺼번에: 편집...

### 2.2 `computeUpdateBody` 순수 함수 추출

저장 버튼이 PATCH 에 보낼 payload 조합 규약을 순수 함수로 뽑아 단위 테스트 가능. 변경 없는 필드는 payload 에 담지 않아 서버가 부분 업데이트로 처리. 설명을 빈 문자열로 지우면 **null** 로 전송 (PostgreSQL TEXT 컬럼 nullable).

### 2.3 SearchPage 필터는 **컬렉션·폴더·하위** 3요소

- 컬렉션 select — 전체 / 각 컬렉션. "검색테스트" 등 owner 소유만 (서버 `list_by_owner`)
- 폴더 select — path 를 `›` 구분자로 표시
- 폴더가 선택되면 **"하위 폴더 포함" 체크박스** 조건부 노출 (폴더 해제 시 자동 사라짐)
- 필터 초기화 버튼은 기존 type/status 외에 collection/folder 도 함께 리셋

### 2.4 URL params 보존 검색

`handleSearch` 가 `router.push(`/search?q=${q}`)` 로 덮어쓰면 collection/folder 가 사라짐. 이번 라운드에서 `URLSearchParams(searchParams.toString()).set("q", q)` 로 **다른 파라미터 보존**.

### 2.5 URL → state 동기화 범위 확장

기존엔 `q` 만 감지해서 state 업데이트했으나 이제 4 키 (`q`, `collection`, `folder`, `include_subfolders`) 전부 관찰. prev ref 4개를 둬서 외부 URL 변경이 state 에 반영되도록.

---

## 3. Chrome 실측

| # | 단계 | 결과 |
|---|------|------|
| 1 | 컬렉션 `···` 메뉴 | "이름 변경 / **편집...** / 삭제" 3 항목 노출 ✓ |
| 2 | "편집..." 클릭 | `컬렉션 편집` 모달 오픈, 이름 input 프리필 + 설명 textarea 빈 + 0/2000 카운터 + 취소/저장 버튼 ✓ |
| 3 | 설명 입력 후 "저장" | 토스트 `컬렉션이 수정되었습니다` + 모달 닫힘 ✓ |
| 4 | `/search?q=Test` 진입 | 필터 바에 **관련도순 / 전체 유형 / 전체 상태 / 전체 컬렉션 / 전체 폴더** 5 드롭다운 ✓ |
| 5 | 컬렉션 드롭다운에서 "검색테스트" 선택 | `Test` AND `collection=<검색테스트>` 로 **결과 0건** (해당 컬렉션에 Test 문자 있는 문서 없음 — 정확히 의도된 AND 결합 동작) + "필터 초기화" 버튼 노출 ✓ |

---

## 4. 자동화 검증

### 4.1 node:test

```
$ npm test
# tests 229   suites 44   pass 229   fail 0   duration_ms ~760
```

신규 13건 `EditCollectionAndSearchFilterFg21Ux6`:

**EditCollectionModal.computeUpdateBody** (9)
- 이름만 변경 / 설명만 추가 / 동시 변경 / 설명 빈 문자열 → null / 앞뒤 공백 trim / 변경 없음 → 빈 객체 / 이름 빈 → null / 이름 201자 → null / 설명 2001자 → null

**searchApi.documents** (4)
- collection 전송 / folder + include_subfolders 전송 / 세 필터 + q 전송 / 필터 없으면 기본 q 만

### 4.2 TypeScript 타입 체크

`tsc --noEmit` — FG 2-1 UX 6차 변경 파일 에러 0. 기존 `AdminUserDetailPage.tsx:279` 만 범위 밖.

---

## 5. 검수 보고서 §7.1 후속

`FG2-1_검수보고서.md §7.1` 의 이월 4건 중 **2건 해결** (취소선 마킹):
1. ~~컬렉션 description 편집 UI~~ → **해결**
2. ~~SearchPage UI 에 collection/folder 필터~~ → **해결**
3. search_documents_hybrid pytest — 잔존
4. `/documents/{id}/edit` 기존 별건 — 잔존 (범위 밖)

---

## 6. 남은 후속

1. **search_documents_hybrid 경로 pytest** (작은 범위)
2. FG 2-2 태그 착수
3. 운영자 macOS 실 pytest 실행 → FG 2-1 공식 종결 선언

---

*작성: 2026-04-24 | FG 2-1 UX 다듬기 6차 완료, Chrome 2/2 PASS + node:test 229*
