# FG 2-1 UX 다듬기 4차 — 실시간 검색 debounce + FTS 경로 분리 유지(연결)

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-24 |
| 범위 | DocumentListPage SearchInput 을 Enter 기반에서 **300ms debounce 자동 검색** 으로 전환 + `/documents?q=` (title ILIKE) 와 `/search` (FTS) 를 "본문까지 전체 검색 →" 링크로 연결 |
| 설계 선택 | **B + 연결 (사용자 확정)** — 두 경로의 역할 분리를 유지하되 UX 에서 자연스럽게 이어줌 |
| 결과 | Chrome 실측 3 시나리오 PASS (debounce 자동 필터 / placeholder 정돈 / /search 프리필 전환) + **node:test 209 passed** |

---

## 1. 변경 파일

### 1.1 수정

| 파일 | 변경 |
|------|------|
| `frontend/src/features/documents/DocumentListPage.tsx` | `search` state 입력 후 300ms debounce → `filters.q` 반영하는 `useEffect` 신설. Enter 는 "즉시 반영" 보조 경로로 유지 (`handleSearchSubmit`). placeholder 를 "제목·내용 검색…" → **"제목 검색…"** 으로 정돈 (실제 ILIKE 동작에 맞춤). `aria-label="문서 제목 검색"` 추가. SearchInput 오른쪽에 **"본문까지 전체 검색 →"** Link 추가 — 현재 검색어가 있으면 `/search?q=<encoded>`, 없으면 `/search` 로 이동 |

### 1.2 신규

| 파일 | 역할 |
|------|------|
| `frontend/tests/DocumentListSearchDebounceFg21Ux4.test.tsx` | debounce 타이밍 규약 2 건 (연달아 입력 시 마지막 값만 apply / cleanup 시 apply 스킵) + `/search` 링크 URL 규칙 3 건 (검색어 있음·없음·trim) |

---

## 2. 설계 판단

### 2.1 두 경로의 현 구조 (실측 요약)

| 항목 | `/documents?q=` (UX 3차) | `/search/documents` (Phase 8) |
|------|-------------------------|------------------------------|
| 대상 | `title ILIKE` (부분 일치) | `documents.search_vector` tsvector (title + summary) |
| 정렬 | updated_at / created_at / title | relevance(rank) / created_at / updated_at |
| 응답 | DocumentResponse 풀세트 | SearchResult + ts_headline 하이라이트 |
| collection / folder 필터 | ✓ | ✗ |
| scope_profile_id | ✓ | 훅만 있음 |
| paging | page + page_size | page + limit |
| 용도 | 리스트의 **빠른 제목 필터** | **전역 본문 검색** |

### 2.2 옵션 A/B/C 비교

| 옵션 | 설명 | 평가 |
|------|------|-----|
| A. 병합 | `/documents?q=` 를 FTS 로 교체 + collection/folder subquery 포팅 | 구현 중~대, sort/paging 계약 변경, list vs search 책임 혼재 |
| **B + 연결 (선택)** | 현 경로 그대로 두되 DocumentListPage 에서 `/search?q=<q>` 로 링크 이동 버튼 추가 | 변경 최소, 두 경로 역할 명확, UX 자연 전환 |
| C. 옵션 확장 | `/documents?q=&full_text=true` 플래그 | 두 쿼리 경로 공존 → 코드 복잡도, sort 규약 혼재 |

**결정 근거**: `/documents?q=` 는 컬렉션/폴더/scope 와 결합된 **리스트 필터** 가 정체성. FTS 로 바꾸면 이 결합이 깨짐. 반면 `/search` 는 이미 FTS 전용 UI (하이라이트, 문서/섹션 탭, 관련도 정렬) 가 완성되어 있음. 두 경로를 UX 링크 하나로 연결하는 게 사용자 멘탈 모델에도 맞음.

### 2.3 debounce 300ms

- 이전: Enter 를 눌러야 검색 → 키보드 사용자와 모바일에서 번거로움
- 이번: 입력 후 300ms 조용하면 자동 검색. Enter 는 **즉시 반영** 보조 경로로 남김 (빠른 사용자용)
- `filters.q` 는 trim 후 빈 문자열이면 `undefined` 로 → 서버로 안 보냄
- `useEffect` cleanup 로 이전 타이머 취소해 **마지막 값만 반영** 보장

### 2.4 placeholder 정직성

- 이전 "제목·내용 검색…" 은 실제로는 title 만 검색하므로 **사용자를 오도**
- UX 3차에 서버 q 를 연결했으니 현재 실제 동작 = "제목 ILIKE"
- 본문 검색을 원하면 옆의 "본문까지 전체 검색 →" 링크로 자연 유도

---

## 3. Chrome 실측 — 3 시나리오 PASS

| # | 단계 | 결과 |
|---|------|------|
| 1 | `/documents` 진입 | SearchInput placeholder = **"제목 검색…"** + 오른쪽에 🔎 **"본문까지 전체 검색 →"** 링크 노출 ✓ |
| 2 | `Audit` 타이핑 | 300ms 후 자동 필터 — 목록이 `Audit Test` 1건으로 좁혀지고 `검색: Audit` 필터 뱃지 생성. Enter 불필요 ✓ |
| 3 | "본문까지 전체 검색 →" 클릭 | `/search?q=Audit` 로 이동, 검색창 프리필 + **`Audit` Test** 노란색 하이라이트 + 문서/섹션 탭 FTS 결과 ✓ |

---

## 4. 자동화 검증

### 4.1 node:test

```
# tests 209   suites 41   pass 209   fail 0
```

신규 5건 `DocumentListSearchDebounceFg21Ux4`:
- 연달아 입력 시 마지막 값만 apply (debounce 의도)
- cleanup 시 apply 호출 안 됨
- `/search?q=<encoded>` (한글/공백)
- 빈/공백만 → `/search`
- 앞뒤 공백 trim

### 4.2 tsc --noEmit

FG 2-1 UX 4차 변경 파일 에러 0. 기존 `AdminUserDetailPage.tsx:279` 만 범위 밖 유지.

---

## 5. 문서화된 후속 여지

1. **search_service 의 scope_profile_id wiring** — 현재 파라미터 훅만 있고 상위 호출자가 아직 전달 안 함. FTS 쿼리도 viewer Scope 필터 받게 하려면 `/search/documents` 라우터에서 `actor.scope_profile_id` 전달
2. **search_service 에 collection/folder 필터 포팅** — 사용자가 "이 컬렉션 안에서 본문 검색" 을 원하는 시나리오는 아직 미지원
3. **DocumentListPage 검색어 → URL query 동기화** — 현재 filters 내부 state 만 변경. 북마크·뒤로가기 위해 `?q=` 를 URL 에도 반영하면 더 자연

이 세 가지는 FG 2-2 태그 착수 후 또는 별도 FG 로 다룸.

---

*작성: 2026-04-24 | FG 2-1 UX 다듬기 4차 완료, Chrome 전 시나리오 PASS + node:test 209*
