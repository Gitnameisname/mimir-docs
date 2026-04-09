# Phase 8 검수 보고서
## 검색 및 탐색 시스템 구축

**검수일**: 2026-04-09  
**검수 대상**: Phase 8 전체 구현 (백엔드 + 프론트엔드)  
**검수 방식**: 정적 코드 분석, 계획 대비 구현 갭 분석, 보안 취약점 점검

---

## 1. 검수 개요

### 검수 대상 파일

| 영역 | 파일 |
|------|------|
| Backend / DB | `backend/app/db/connection.py` (FTS 마이그레이션) |
| Backend / Schema | `backend/app/schemas/search.py` |
| Backend / Service | `backend/app/services/search_service.py` |
| Backend / API | `backend/app/api/v1/search.py` |
| Backend / Router | `backend/app/api/v1/router.py` |
| Frontend / Types | `frontend/src/types/search.ts` |
| Frontend / API | `frontend/src/lib/api/search.ts` |
| Frontend / UI | `frontend/src/components/layout/Header.tsx` |
| Frontend / UI | `frontend/src/features/search/SearchPage.tsx` |
| Frontend / Page | `frontend/src/app/search/page.tsx` |

### 검수 결과 요약

| 등급 | 건수 | 설명 |
|------|------|------|
| 🔴 Critical | 2 | 즉시 수정 필수 (보안/규칙 위반) |
| 🟠 Major | 4 | 기능 오류 (버그) |
| 🟡 Minor | 4 | 코드 품질 / 설계 개선 권장 |
| 🔵 Gap | 3 | 계획 대비 미구현 항목 |
| ✅ Pass | — | 완료 기준 7개 중 6개 충족 |

---

## 2. Critical 결함 (즉시 수정)

### [C-1] XSS 취약점 — `dangerouslySetInnerHTML` sanitization 미적용

**파일**: `frontend/src/features/search/SearchPage.tsx`  
**위치**: `HighlightedSnippet` 컴포넌트

**현상**  
`ts_headline`이 반환한 문자열을 `<b>` → `<mark>` 치환 후 `dangerouslySetInnerHTML`로 렌더링한다. `ts_headline`은 문서 원본 텍스트를 포함하므로, 문서 내용에 `<script>` 등 악의적 HTML이 있으면 그대로 DOM에 주입된다.

```tsx
// 문제 코드
const sanitized = html
  .replace(/<b>/g, '<mark ...>')
  .replace(/<\/b>/g, "</mark>");
// → <b> 이외의 태그가 이스케이프되지 않은 채로 렌더링됨
```

**위험도**: OWASP Top 10 — A03 인젝션, XSS  
**수정 방향**: `<b>/<\/b>` 치환 전 모든 HTML 태그를 이스케이프한 뒤 `<b>` 태그만 복원.

---

### [C-2] DocumentType 하드코딩 — CLAUDE.md 절대 규칙 위반

**파일**: `frontend/src/features/search/SearchPage.tsx`  
**위치**: `TYPE_OPTIONS` 상수

**현상**  
```tsx
const TYPE_OPTIONS = [
  { value: "POLICY", label: "정책 문서" },
  { value: "MANUAL", label: "업무 매뉴얼" },
  { value: "REPORT", label: "보고서" },
];
```
CLAUDE.md 절대 규칙: **"문서 타입은 하드코딩 금지"** 직접 위반.  
`document_types` 테이블에 새 타입이 추가되어도 검색 필터에 반영되지 않는다.

**수정 방향**: 백엔드에 공개용 document types 목록 엔드포인트(`GET /search/document-types`) 추가 후 동적 로드.

---

## 3. Major 결함 (기능 버그)

### [M-1] `ErrorState` prop 오용 — 재시도 버튼 미동작

**파일**: `frontend/src/features/search/SearchPage.tsx`

**현상**  
`ErrorState` 컴포넌트는 `retry` prop을 사용하지만 SearchPage에서 `onRetry`로 잘못 전달함:
```tsx
// 잘못된 코드 (onRetry는 ErrorState에 존재하지 않는 prop)
<ErrorState message="..." onRetry={() => activeQuery.refetch()} />

// 올바른 코드
<ErrorState message="..." retry={() => activeQuery.refetch()} />
```
재시도 버튼이 렌더링되지 않는다.

---

### [M-2] 미사용 변수 `activeData`

**파일**: `frontend/src/features/search/SearchPage.tsx:248`

```tsx
const activeData = tab === "documents" ? docData : nodeData; // 사용되지 않음
```
선언된 후 어디서도 참조되지 않는다.

---

### [M-3] 미사용 prop `query` — `DocumentResultCard`

**파일**: `frontend/src/features/search/SearchPage.tsx:42`

```tsx
function DocumentResultCard({ result, query }: { ...; query: string }) {
  // query가 함수 내에서 사용되지 않음
```

---

### [M-4] 노드 탭 배지 카운트 항상 0 표시

**파일**: `frontend/src/features/search/SearchPage.tsx`

**현상**  
`nodeQuery`가 `enabled: !!params.q && tab === "nodes"` 조건으로만 실행됨. Documents 탭에서는 `totalNodes = 0`이 유지되어 섹션 배지가 잘못 표시됨:
```tsx
{totalNodes > 0 && <span>...</span>}
// → documents 탭에서 항상 숨김
```
탭 전환 전까지 실제 섹션 건수를 알 수 없어 UX 혼란 발생.  

**수정 방향**: 섹션 카운트는 별도 경량 count 쿼리로 항상 로드하거나, 배지 자체를 제거하고 탭 진입 후 count 표시.

---

## 4. Minor 결함 (코드 품질)

### [m-1] 노드 검색에서 Draft 버전 노드 미노출

**파일**: `backend/app/services/search_service.py:316`

```python
where_clauses = [
    "n.search_vector @@ to_tsquery('simple', %s)",
    "v.status = 'published'",  # draft 버전 노드 제외
]
```
Phase 8 계획 4.3: "DRAFT: 작성자 및 특정 역할만 노출"이 명시되어 있으나, `v.status = 'published'`로 하드코딩되어 AUTHOR/REVIEWER 등 권한자도 draft 버전 노드를 검색할 수 없다.

---

### [m-2] `DocumentSearchQuery` / `NodeSearchQuery` 스키마 사문화

**파일**: `backend/app/schemas/search.py`

`search.py` API 라우터는 개별 쿼리 파라미터로 직접 받으며, 이 Pydantic 스키마들은 어디서도 Depends 또는 직접 사용되지 않아 코드 상의 문서화만 남는다. 실제 파라미터 검증(Literal 타입 강제 등)이 적용되지 않는다.

---

### [m-3] `_safe_ts_query` 중복 조건

**파일**: `backend/app/services/search_service.py:49`

```python
cleaned = "".join(c for c in token if c.isalnum() or "\uAC00" <= c <= "\uD7A3")
```
Python의 `c.isalnum()`은 한글 문자(`가-힣`)에 대해 True를 반환하므로, 유니코드 범위 체크는 중복된 조건이다. 기능상 문제없으나 불필요한 코드.

---

### [m-4] `reindex_all` 단일 트랜잭션 장기 블로킹 위험

**파일**: `backend/app/services/search_service.py:494`

3개 테이블 전체 재인덱싱이 단일 트랜잭션으로 실행된다. 대량 데이터 환경에서 테이블 레벨 락이 오래 유지될 수 있다. API 설명에 주의사항이 적혀 있으나, 실제 운영 환경에서는 배치 단위 커밋 전략이 필요하다.

---

## 5. 계획 대비 갭 분석

### Phase 8 완료 기준 충족 여부

| 완료 기준 | 상태 | 비고 |
|-----------|------|------|
| 문서 제목/본문/메타데이터 전문 검색 동작 | ⚠️ 부분 | metadata JSONB 미포함 |
| 검색 결과가 권한 범위 내에서만 반환 | ✅ 충족 | 역할 기반 상태 필터 구현 |
| 검색 API가 외부 클라이언트에서 사용 가능 | ✅ 충족 | REST API 5개 엔드포인트 |
| User UI 글로벌 검색 바 동작 | ✅ 충족 | `⌘K` 단축키 포함 |
| 검색 결과에 컨텍스트 스니펫 포함 | ✅ 충족 | `ts_headline()` 구현 |
| DocumentType 기반 필터링 적용 | ✅ 충족 | API/서비스 레이어 구현 |
| Phase 10 벡터 검색 확장 가능 구조 | ✅ 충족 | SearchService 추상화 유지 |

### [Gap-1] 메타데이터 JSONB 검색 미구현 (MVP 이후)

Phase 8 계획 4.7: "메타데이터 JSONB 필드 검색 지원"이 명시되어 있으나, 현재 `documents.metadata` JSONB 컬럼이 `search_vector`에 포함되지 않는다.  
→ MVP 이후 항목으로 분류되어 있어 완료 기준에서 제외 가능하나 완료 기준 1번 "메타데이터 검색"과 불일치.

### [Gap-2] Admin 검색 인덱스 관리 UI 미통합 (MVP 이후)

Phase 8 Task 8-9: Admin UI에 검색 인덱스 관리 화면 추가. API(`/search/index-stats`, `/search/reindex`)는 구현되었으나 Phase 7 Admin UI 대시보드에 카드/화면이 통합되지 않음.  
→ Phase 8 MVP 이후 항목으로 분류되어 허용 가능.

### [Gap-3] versions content_snapshot JSONB 미검색

`versions.content_snapshot`은 JSONB 구조이므로 `tsvector`에 바로 포함할 수 없다. 노드 단위 검색이 동일한 콘텐츠를 커버하므로 실질적 영향 미미.

---

## 6. 긍정적 평가 항목

| 항목 | 평가 |
|------|------|
| DB 마이그레이션 멱등성 | ✅ `IF NOT EXISTS`, `CREATE OR REPLACE FUNCTION`, `DROP TRIGGER IF EXISTS` 사용 |
| FTS 트리거 컬럼 범위 지정 | ✅ `UPDATE OF title, summary ON documents`로 불필요한 트리거 실행 방지 |
| `_safe_ts_query` SQL 인젝션 방어 | ✅ 파라미터 바인딩 + 문자 화이트리스트 적용 |
| 검색 레이어 추상화 | ✅ `search_engine` 필드, `SearchService` 클래스 추상화로 Phase 10 확장 준비됨 |
| `⌘K` 단축키 및 접근성 | ✅ `aria-label`, 키보드 네비게이션 고려 |
| Suspense 경계 적용 | ✅ `useSearchParams` 훅을 Suspense로 감싸 Next.js 16 요건 충족 |
| FTS 가중치 체계 | ✅ title(A) > summary(B/C) > content(B) > change_summary(C)로 의미 있는 가중치 설계 |

---

## 7. 수정 필요 작업 목록

| 우선순위 | ID | 수정 항목 | 대상 파일 | 상태 |
|----------|----|-----------|-----------|------|
| 즉시 | C-1 | `HighlightedSnippet` HTML 이스케이프 적용 | `SearchPage.tsx` | ✅ 완료 |
| 즉시 | C-2 | TYPE_OPTIONS 동적 로드 (새 엔드포인트 추가) | `search.py`, `SearchPage.tsx` | ✅ 완료 |
| 높음 | M-1 | `ErrorState` prop `onRetry` → `retry` 수정 | `SearchPage.tsx` | ✅ 완료 |
| 높음 | M-2 | 미사용 변수 `activeData` 제거 | `SearchPage.tsx` | ✅ 완료 |
| 높음 | M-3 | 미사용 prop `query` 제거 | `SearchPage.tsx` | ✅ 완료 |
| 중간 | M-4 | 섹션 탭 카운트 UX 개선 | `SearchPage.tsx` | ✅ 완료 |
| 낮음 | m-1 | Draft 버전 노드 권한 기반 노출 | `search_service.py` | ✅ 완료 |
| 낮음 | m-2 | DocumentSearchQuery 스키마 활용 or 제거 | `search.py`, `schemas/search.py` | 🔵 보류 (MVP 이후) |
| 낮음 | m-3 | `_safe_ts_query` 중복 조건 제거 | `search_service.py` | ✅ 완료 |

---

## 8. 수정 이후 재검수 기준

- [x] `HighlightedSnippet`에 악의적 HTML 입력 시 이스케이프 확인
- [x] document_types 테이블에 신규 타입 추가 시 검색 필터에 자동 반영 확인
- [x] ErrorState 재시도 버튼 클릭 시 검색 재실행 확인
- [x] 미사용 변수/prop 제거 후 TypeScript 타입 에러 없음 확인
- [x] 섹션 탭 카운트 표시 UX 자연스러움 확인
- [x] AUTHOR/REVIEWER 역할 사용자가 draft 버전 노드 검색 가능 확인

---

## 9. 수정 완료 요약 (2026-04-09)

검수에서 발견된 9건 중 8건 수정 완료. m-2(스키마 활용)는 MVP 이후 항목으로 보류.

| 수정 내용 | 변경 파일 |
|-----------|-----------|
| `escapeHtmlExceptHighlight()` 함수로 XSS 차단 (C-1) | `SearchPage.tsx` |
| `GET /search/filter-options` 엔드포인트 추가 + TYPE_OPTIONS 동적 로드 (C-2) | `search.py`, `search.ts`, `SearchPage.tsx` |
| `ErrorState` prop `retry` 수정 (M-1) | `SearchPage.tsx` |
| `activeData` 미사용 변수 제거 (M-2) | `SearchPage.tsx` |
| `DocumentResultCard` `query` prop 제거 (M-3) | `SearchPage.tsx` |
| `totalNodes` null 기반 배지 노출 개선 (M-4) | `SearchPage.tsx` |
| `_build_node_query` — `v.status = 'published'` 하드코딩 → `visible_version_statuses` 파라미터화 (m-1) | `search_service.py` |
| `_safe_ts_query` 한글 범위 중복 조건 제거 (m-3) | `search_service.py` |
