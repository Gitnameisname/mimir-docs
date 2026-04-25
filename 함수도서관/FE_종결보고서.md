# 함수도서관 프론트엔드 (TypeScript) 종결보고서

> **작성일**: 2026-04-25
> **범위**: docs/함수도서관/frontend.md §1 의 전 후보 (8 그룹)
> **검증 환경**: `npm test` 495 passed / 0 fail / tsc 0 touched 신규 오류.

---

## 1. 신설 모듈 (12개)

| 모듈 | 등록 항목 | 그룹 |
|------|----------|------|
| `lib/utils/date.ts` | `formatDateTime` (F1) + `formatDateOnly` (FE-G1) | F1 + FE-G1 |
| `hooks/useDebouncedValue.ts` | `useDebouncedValue<T>` | F6 |
| `hooks/useDebouncedCallback.ts` | `useDebouncedCallback<F>` | §1.6a |
| `lib/utils/url.ts` | `toQueryString` + `parseListFilterParams` + 6 readers + `mutateSearchParams` + `SearchParamsLike` | §1.2 + §1.2b + §1.2c |
| `lib/utils/errors.ts` | `classifyApiError` + `ApiErrorClassification` | FE-G2 |
| `lib/utils/download.ts` | `downloadJsonFile` | FE-G2 |
| `lib/utils/guards.ts` | `isString` / `isNonEmptyString` / `isPlainObject` | FE-G2 |
| `lib/constants/labels/{extraction,goldenSet,evaluation}.ts` | 5 도메인 라벨 | FE-G3 |
| `lib/constants/badges/{extraction,goldenSet,evaluation}.ts` | 3 도메인 배지 | FE-G3 |
| `lib/styles/tokens.ts` | BADGE_BASE + ALERT 4종 (FE-G4) + R3 변형 7종 (BADGE_BASE_PILL / ALERT_ERROR_COMPACT / 다크모드 5종) | FE-G4 + R3 |

---

## 2. 호출지 마이그레이션 누계

| 그룹 | 사이트 수 | 비고 |
|------|----------|------|
| F1 formatDateTime | 5 (api-keys + evaluations 3 + sessions) | ISO식 통일 |
| F6 useDebouncedValue | 3 (TagChipsEditor 150ms / AddDocsModal 300ms / DocumentList 300ms) | excluded by design 3 (callback debounce) |
| §1.6a useDebouncedCallback | 3 (useUserPreferences 400ms / DocumentEdit 30s autoSave / SearchBox 500ms) | useUserPreferences cleanup 잠재 버그 부수 수정 |
| §1.2 toQueryString | 11 (7 파일) | 8 API 클라이언트 + CitationItem |
| §1.2b parseListFilterParams | 5+4×2 = **13** (DocumentList + SearchPage 모듈 상수) | URL 필터 파싱 표준화 |
| §1.2c G-Carry | (1) extraction-queue 부분 차용 + (2) URL state mutate 4 + (3) **buildQueryString 31 호출자** 일괄 toQueryString | 가장 큰 단일 마이그레이션 |
| FE-G2 시범 | 3 (mutationErrorMessage + downloadJsonFile 2) | 호출지 일괄은 별 라운드 |
| FE-G3 admin 라벨/배지 | 3 호출지 + 자동 영향 16+ 파일 | thin re-export 호환 보존 |
| FE-G4 토큰 시범 | 7 (BADGE_BASE 4 + ALERT_ERROR 3) | 변형 23+ 와 다크모드는 별 라운드 |
| FE-G1 시범 | 1 (golden-sets) + R2-A 11 (admin date) = **12** | 가시 UX: ko locale → ISO식 통일 |
| R2-B type guards | 20 (10 파일) | url.ts 1건 union narrowing 보존 |

**합계**: **약 110 사이트** (F1 5 + F6 3 + §1.6a 3 + §1.2 11 + §1.2b 13 + §1.2c 36 + FE-G2 3 + FE-G3 3 + FE-G4 7 + FE-G1 12 + R2-B 20 = 116).

---

## 3. 도입 안 함 결정 항목

| 항목 | 결정 이유 |
|------|----------|
| `formatRelativeShort` (§1.1) | 호출지 0건 + 한/영 표기 결정 미정 |
| `classifyApiError` 호출지 일괄 (R2-C) | `getApiErrorMessage` 가 401/403/409 도메인 한국어 라벨 인라인 보존. 두 helper 공존 |

---

## 4. 잔존 (점진 마이그레이션)

| 항목 | 사이트 추정 | 정책 |
|------|------------|------|
| 변형 alert / 다크모드 호출지 | 23+ | **D3 결정**: S3 완수 이후 별 작업 (admin 다크 전면 도입과 함께) |
| classifyApiError 49 호출지 | 49 | 두 helper 공존 — 호출지 마이그레이션 안 함 |
| type guards 변형 (Array/object) | 추가 신설 시점 미정 | ESLint custom rule 도입 후 점진 |

---

## 5. CONSTITUTION 준수

- **제8조 Single Responsibility**: 각 helper 단일 책임.
- **제10조 Docstring as Agent Contract**: 모든 helper 에 JSDoc + 예시.
- **제11조 Function Index**: `docs/함수도서관/frontend.md` 인덱스 동기화.
- **제15조 Bounded Change**: 그룹 단위 변환 + 테스트 동반.

---

## 6. 검증

- `npm test` (tsc + node:test): **495 passed / 0 fail**.
  - F1: 16 case
  - F6: 14 case
  - §1.6a: 22 case
  - §1.2 toQueryString: 32 case
  - §1.2b parseListFilterParams + readers: 40 case
  - §1.2c readRegexString + mutateSearchParams: 23 case
  - FE-G2: 22 case
  - FE-G3: 21 case
  - FE-G4: 14 + R3 12 = 26 case
  - FE-G1 (formatDateOnly): 8 case
- `tsc --noEmit`: touched 파일 신규 오류 0건.

---

## 7. UX 변경 (가시 영향)

| 변경 | 사이트 | 비고 |
|------|-------|------|
| 날짜 표기 ko locale → ISO 식 | 12 (golden-sets + admin 11) | 철균 합의 (admin 시간 통일) |
| autoSave 시맨틱 변경 | DocumentEdit | 진짜 debounce (30s 후 1회 → 입력마다 timer reset) |

---

## 8. 운영자 행동 항목

1. (선택) Chrome MCP 실측 — admin 페이지 날짜 표기 ISO 식 확인.
2. preferences.theme 토글 후 다크모드 토큰 적용 페이지 점검 (현재 호출지 마이그레이션 0이라 영향 없음 — admin 다크 전면 도입 후 검증).
3. 새 호출자 PR 리뷰 시 helper 사용 강제 (code-review 게이트).
