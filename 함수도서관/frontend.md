# 함수도서관 — 프론트엔드 (TypeScript)

> 배치 규약: 본 카탈로그의 모든 신규 유틸은 `frontend/src/lib/utils/` 하위에 도메인별 모듈로 둔다.
> 기존 `frontend/src/lib/utils.ts` 는 유지해 기존 import 호환성을 보존한다.

---

## 0. 기존 유틸 (이미 사용 중)

### `@/lib/utils`

- `cn(...inputs: ClassValue[]) -> string` ✅
  - purpose: clsx + tailwind-merge 조합. Tailwind 클래스 병합의 표준.
  - effects: none
  - notes: 모든 동적 className 조합은 이 함수를 통한다. 수동 문자열 연결 금지.

- `buildQueryString(params: Record<string, string | number | boolean | undefined | null>) -> string` ✅
  - purpose: `undefined`/`null`/빈 문자열을 제외한 query string 생성 (`"?page=1&q=hello"` 형태).
  - effects: none
  - notes: 본 유틸이 존재함에도 일부 API 클라이언트(`tags.ts`, `proposals.ts`, `collections.ts` 등 10곳)는 `new URLSearchParams()` 를 직접 조립한다 — 후속 라운드에서 이 유틸로 통일한다.

- `formatDate(iso: string) -> string` ✅
  - purpose: `YYYY. MM. DD.` 포맷 (ko-KR).
  - effects: none
  - notes: `app/account/sessions/page.tsx:36`, `features/admin/golden-sets/AdminGoldenSetsPage.tsx:47` 등에 **로컬 `formatDate` 재선언 4건**이 별도로 존재 — 후속 라운드에서 일괄 `@/lib/utils` 의존으로 치환.

- `relativeTime(iso: string) -> string` ✅
  - purpose: `"3일 전"`, `"방금 전"` 등 한국어 상대 시간 표기.
  - effects: none

### `@/lib/logger`

- `logger` ✅
  - purpose: 프로덕션 빌드에서 `console.debug`/`console.info` 억제, `error`/`warn` 유지.
  - notes: `console.log` 를 직접 쓰지 말고 `logger.*` 를 쓴다.

### `@/hooks/useFocusTrap`, `useMutationWithToast`, `useTheme`, `useUserPreferences`, `useAuthz`, `useExtractionActions`, `useTags`(각 feature별) ✅

- 각 훅의 단일 책임·시그니처는 해당 파일 상단 JSDoc 참조. 본 도서관에서는 시그니처만 간단히 나열한다.

---

## 1. 유틸 (✅ Existing / 🟡 Proposed)

### 1.1 `@/lib/utils/date`

- `formatDateTime(iso: string | null | undefined) -> string` ✅
  - status: ✅ Existing (2026-04-25 F1 승격)
  - purpose: 날짜 + 시간(`YYYY-MM-DD HH:mm`, 로컬 시간) 포맷. `null`/`undefined`/빈 문자열/invalid 입력은 `"-"` 로 방어적 반환.
  - effects: none
  - errors: none (던지지 않음 — 잘못된 입력은 `"-"`)
  - source: `frontend/src/lib/utils/date.ts`
  - tests: `frontend/tests/FormatDateTimeF1.test.ts` (16 cases, `process.env.TZ="UTC"` 로 타임존 결정론)
  - replaced in:
    - `features/admin/api-keys/AdminApiKeysPage.tsx` (로컬 `formatDateTime` 제거 — 기존 `toLocaleString("ko")` → 신규 ISO식)
    - `features/admin/evaluations/helpers.ts` (export 제거 → 3 importer 가 `@/lib/utils/date` 직접 import)
    - `features/admin/evaluations/AdminEvaluation{s,Detail,Compare}Page.tsx` (import 경로 이관)
    - `app/account/sessions/page.tsx` (로컬 `formatDate` 제거 — 호출지도 `formatDateTime` 으로 리네임)
  - excluded by design:
    - `features/admin/golden-sets/AdminGoldenSetsPage.tsx:47` — 이 로컬 함수는 date-only (`"2026. 4. 25."`) 이라 의미가 다름. 별도 null-safe `formatDate` 유틸(1.1a 후보) 도입 시 처리.
  - notes:
    - 출력 포맷이 가시적으로 변경됨: `"2026. 4. 25. 오전 11:30"` → `"2026-04-25 11:30"`. 2026-04-25 철균 지시로 ISO식 표기 채택 — 로케일 독립적이고 정렬 가능.
    - `@/lib/utils` 의 기존 `formatDate(iso: string) -> "YYYY. MM. DD."` 는 건드리지 않음 (하위 호환).
    - 시간은 런타임 로컬 타임존으로 렌더링. UTC 고정이 필요하면 별도 유틸 도입 (현재 요구 없음).

- `formatDateOnly(iso: string | null | undefined) -> string` ✅ (FE-G1, 2026-04-25)
  - status: ✅ Existing (2026-04-25 FE-G1 승격, F1 이름 충돌 회피로 `formatDateOnly` 채택)
  - purpose: ISO 문자열 → `"YYYY-MM-DD"` (로컬 시간). null-safe `"-"`. F1 `formatDateTime` 의 date-only 버전.
  - effects: none
  - errors: none (잘못된 입력은 `"-"` 반환)
  - source: `frontend/src/lib/utils/date.ts` (F1 모듈에 함께 둠)
  - tests: `frontend/tests/FormatDateTimeF1.test.ts` `formatDateOnly` describe (8 case — UTC ISO / 시·분 무시 / 패딩 / 윤년 / offset / null·undefined·"" / 파싱 실패 / 길이·정규식 회귀)
  - replaced in:
    - `features/admin/golden-sets/AdminGoldenSetsPage.tsx` 로컬 `formatDate` — `formatDateOnly` 로 alias 위임. **가시 UX 변경**: `"2026. 4. 25."` (ko locale) → `"2026-04-25"` (ISO 식). F1 `formatDateTime` 과 표기 일관 (admin 시간 통일 정책, 2026-04-25 철균 합의).
  - excluded by design (별 라운드, 가시 UX 변경 영향 검토 후):
    - admin 인라인 `new Date(x).toLocaleDateString("ko")` 8+ 사이트 (`account/profile`, `extraction-schemas`, `ai-platform`, `orgs ×3`, `users ×3`) — 기존 표기 (`"2026. 4. 25."`) → ISO 식 변경이 가시 영향. 도메인별 검토 후 점진.
    - 기존 `@/lib/utils#formatDate(iso: string) -> "YYYY. MM. DD."` — 이름 보존 (외부 호환). 신규 호출자는 `formatDateOnly` 사용 권장.
  - notes:
    - F1 `formatDateTime` 과 동일한 방어적 시맨틱 (null/undefined/""/파싱 실패 → `"-"`).
    - 시·분이 필요한 호출자는 `formatDateTime` 사용. date-only 가 적합한 admin 테이블·"가입일" 라벨 등에 본 helper.

- ~~`formatRelativeShort(iso: string) -> string`~~ ⚠️ (도입 안 함, FE-G1 결정)
  - status: ⚠️ Not Adopted (2026-04-25 FE-G1 결정)
  - decision: 도입하지 않음.
  - reasoning:
    - 현재 호출지 0건 — 기존 `relativeTime("3분 전")` 은 한국어 표기로 일관, 짧은 영문 표기 (`"3d"`/`"just now"`) 요구가 admin 테이블에서 명확히 발견되지 않음.
    - 짧은 표기가 필요해지면 한국어 표기 (`"3일"` / `"방금"`) vs 영문 (`"3d"` / `"just now"`) 결정 필요 — 호출 사례가 나타날 때 정함.
  - alternative: 기존 `relativeTime` 사용. 짧은 표기가 필요하면 그 시점에 신설.

### 1.2 `@/lib/utils/url`

- `toQueryString(params: Record<string, QueryValue>, options?: ToQueryStringOptions) -> string` ✅
  - status: ✅ Existing (2026-04-25 §1.2 승격)
  - purpose: `buildQueryString` 의 확장형. 배열(`status=a&status=b`) · boolean(`true`/`false`) · `null` / `undefined` / `""` 를 일관 처리. NaN/Infinity 도 안전 omit.
  - effects: none (외부 I/O · DOM · 콘솔 · 전역 상태 접근 없음)
  - errors: none (잘못된 타입은 호출자가 책임 — 시그니처가 좁힘)
  - source: `frontend/src/lib/utils/url.ts`
  - tests: `frontend/tests/ToQueryStringF1_2.test.ts` (32 cases — 빈 입력 / 스칼라 / null·undefined·"" 스킵 / 배열 반복 키 / 키 순서 (sortKeys 옵션) / 인코딩 / `buildQueryString` 호환 / 멱등 / 입력 불변)
  - replaced in:
    - `lib/api/tags.ts` (autocomplete · popular — 2 사이트, `qs.size ? '?' + qs : ''` 보일러 제거)
    - `lib/api/extractions.ts` (listPending — 항상 `?` 가 붙던 빈-쿼리 케이스가 깔끔히 omit 으로 정정. backend FastAPI 동작 동일)
    - `lib/api/conversation.ts` (list — 필수 `page`/`limit` + 옵셔널 `status`/`search` 평탄화)
    - `lib/api/collections.ts` (list)
    - `lib/api/diff.ts` (getBetweenVersions · getWithPrevious — boolean 옵션 truthy-skip 시맨틱 보존을 위해 `flag || undefined` 로 좁힘)
    - `lib/api/proposals.ts` (listMine · listAdmin · statsAdmin — 3 사이트. statsAdmin 은 부수효과로 agentId URL-encoding 강화: 기존 `?agent_id=${agentId}` 직접 보간 → `URLSearchParams` 인코딩 경유)
    - `components/chat/CitationItem.tsx` (docUrl — `version` / `node` / `highlight` 합성)
  - excluded by design:
    - ~~`lib/utils.ts` `buildQueryString` — 본 유틸과 공존~~. **2026-04-25 §1.2c G-Carry 에서 일괄 마이그레이션 완료**. `lib/api/admin.ts` (12 사이트) + `lib/api/s2admin.ts` (18 사이트) + `lib/api/documents.ts` (1 사이트) 모두 `toQueryString` 으로 치환. `buildQueryString` 정의는 `@deprecated` JSDoc 추가 후 잠정 유지 (차후 PR 에서 제거).
    - `app/auth/callback/page.tsx:43` — `new URLSearchParams(hash)` 는 fragment 파싱이라 본 유틸 의미와 무관.
    - `features/documents/DocumentListPage.tsx`, `features/search/SearchPage.tsx`, `features/admin/extraction-queue/AdminExtractionQueuePage.tsx` — 기존 `searchParams.toString()` cloning + mutate 패턴은 build-from-scratch 와 의미가 다름 (사용자 결정, 2026-04-25). 별도 helper 후보로 이월.
    - `features/admin/extraction-queue/helpers.ts:213` `toSearchParamsString` — `status=""` (전체) 보존 + `page=1` 기본값 생략 같은 URL state 전용 시맨틱. 본 유틸의 일반 omit 규칙과 충돌. 별도 모듈로 유지.
  - notes:
    - `boolean false` 는 **명시 값으로 보존** (emit `"false"`) — 기존 `buildQueryString` 과 동일한 시맨틱. "false 면 omit" 이 필요한 호출자는 `flag || undefined` 로 좁혀 전달 (diff.ts 가 이 패턴).
    - 빈 객체 / 모든 값 omit-대상 → `""` 반환 (단독 `"?"` 금지).
    - 키 순서: 기본 삽입 순서 보존. `{ sortKeys: true }` 로 알파벳 정렬 — 캐시 키 · 서명 결정성에 사용.
    - 배열 원소 내부의 `null`/`undefined`/`""` 는 개별 스킵. 모두 스킵되면 키 자체가 emit 되지 않음 (`?status=` 같은 dangling 안 발생).
    - 보안: 키 · 값 인코딩은 `URLSearchParams` 위임 (RFC 3986). 사용자 입력을 키로 쓰는 호출자는 화이트리스트 검증 책임 (CLAUDE.md §4.4).

- `readRegexString(key, regex, options?: { normalize?, fallback? }) -> FilterReader<string>` ✅
  - status: ✅ Existing (2026-04-25 §1.2c G-Carry 승격)
  - purpose: 정규식 검증 + 옵션 normalize 가 적용된 문자열 reader. `extraction-queue/parseFiltersFromUrl` 의 documentType (UPPER-SNAKE + uppercase normalize) · scopeProfileId (UUID) 같은 "값 모양 검증 + 가벼운 normalize" 패턴을 캡슐화.
  - effects: none
  - source: `frontend/src/lib/utils/url.ts`
  - tests: `frontend/tests/ParseListFilterParamsF1_2b.test.ts` (8 cases — 부재 / 빈값 / 매치 / 불일치 / normalize / fallback 명시)
  - replaced in: `features/admin/extraction-queue/helpers.ts` `parseFiltersFromUrl` (documentType + scopeProfileId 두 필드 차용)
  - notes: `filterReaders.regexString` alias 도 노출.

- `mutateSearchParams(current, mutations, options?: { prefix? }) -> string` ✅
  - status: ✅ Existing (2026-04-25 §1.2c G-Carry 승격)
  - purpose: 현재 URL params 를 cloning + 일부 키만 set/delete 후 직렬화. `null`/`undefined` 값 → delete, `string` 값 → set, `""` → 명시적 빈 값 보존. 입력 불변 (URL state 쓰기 패턴 표준화).
  - effects: none (router.replace 호출은 호출자 책임)
  - source: `frontend/src/lib/utils/url.ts`
  - tests: `frontend/tests/ParseListFilterParamsF1_2b.test.ts` (15 cases — URLSearchParams / string / null / undefined 입력 / set / delete / 빈값 보존 / 입력 불변 / Next.js ReadonlyURLSearchParams 호환 / 인코딩)
  - replaced in:
    - `features/documents/DocumentListPage.tsx` — 3 사이트 (?q 동기화 / removeUrlFilter 다중 키 삭제 / toggleIncludeSubfolders).
    - `features/search/SearchPage.tsx` — handleSearch 1 사이트.
  - excluded by design:
    - `features/admin/extraction-queue/AdminExtractionQueuePage.tsx:522` — `searchParams ?? new URLSearchParams()` 는 fallback 이지 cloning + mutate 가 아님 (비대상).
    - `app/auth/callback/page.tsx:43` — fragment 파싱.
  - notes:
    - 입력은 `URLSearchParams` / `ReadonlyURLSearchParams` (Next.js) / `string` (raw query, "?" 유무 무관) / `null` / `undefined` 모두 허용.
    - 결과 비면 `""` 반환 (`prefix:false` 무관). `prefix:true` (기본) + 결과 있을 때만 `"?"` 부착.
    - `mutations` 에 명시되지 않은 키는 보존 (touched 안 함). 호출자가 명시적으로 삭제하려면 `key: null`.

- `parseListFilterParams<S>(sp: SearchParamsLike, schema: S) -> { [K in keyof S]: ReturnType<S[K]> }` ✅
  - status: ✅ Existing (2026-04-25 §1.2b 승격)
  - purpose: `URLSearchParams`(또는 Next.js `ReadonlyURLSearchParams` / `{get(key)}` 호환 mock)에서 리스트 페이지 필터를 한 번에 파싱. schema 의 각 reader 반환 타입에서 결과 타입이 자동 추론된다 (zod 의존 없음).
  - effects: none (읽기 전용)
  - errors: none (잘못된 URL 값은 reader 가 silently fallback)
  - source: `frontend/src/lib/utils/url.ts` (§1.2 모듈에 함께 둠)
  - tests: `frontend/tests/ParseListFilterParamsF1_2b.test.ts` (40 cases — 각 reader / 러너 / 타입 추론 / mock 객체 + 실 `URLSearchParams` 양쪽 / 회귀 패턴)
  - companion exports (named — tree-shaking · grep 친화):
    - `readString(key, fallback?: string="")` — 키 부재 시 fallback. 빈 값 `""` 은 보존.
    - `readOptionalString(key)` — 키 부재 시 `undefined`. 빈 값 `""` 은 보존.
    - `readBool(key)` — case-insensitive `"true"` 만 `true`. `"1"`/`"yes"`/`"on"` 등은 `false` (엄격).
    - `readEnum(key, allowed, fallback?)` — 오버로드: fallback 있으면 `T`, 없으면 `T | undefined`.
    - `readBoundedInt(key, { min, max, fallback })` — `parseInt` + 정확 일치 검사 (`"1.5"`/`"1abc"`/`"0x10"` 거부) + 범위 clamp.
    - `filterReaders.{string,optionalString,bool,enum,boundedInt}` — 동일 함수 alias 객체.
    - `SearchParamsLike` 인터페이스 — `{ get(key: string): string | null }` 만 요구.
  - replaced in:
    - `features/documents/DocumentListPage.tsx` — 5 필드 (q · collection · folder · include_subfolders · tag) 의 `searchParams.get(...) ?? undefined` / `.toLowerCase() === "true"` 5 줄을 단일 schema 호출 1건으로 축약.
    - `features/search/SearchPage.tsx` — 4 필드 (q · collection · folder · include_subfolders). 초기 파싱과 useEffect 안의 변경 감지 파싱이 동일 schema (모듈 상수 `URL_FILTERS_SCHEMA`) 를 공유 → reader 정의가 한 번만 존재. q 는 `string("q", "")` (fallback 빈문자) 시맨틱 보존.
  - excluded by design:
    - ~~`features/admin/extraction-queue/helpers.ts:154` `parseFiltersFromUrl`~~ — **2026-04-25 §1.2c G-Carry 에서 부분 차용 완료**. documentType (regexString + uppercase normalize) · scopeProfileId (regexString) · page (boundedInt) 는 일반 reader 로 위임. status 만 bespoke sentinel (null vs `""` vs valid 3구분) 시맨틱 보존을 위해 직접 분기 유지.
    - `app/auth/callback/page.tsx:43` — fragment 파싱.
    - DocumentListPage / SearchPage 의 `searchParams.toString()` cloning + mutate (URL 쓰기 경로) 도 비대상 (반대 방향 — toQueryString 도 별도 결정).
  - notes:
    - **schema 객체 stable identity**: 컴포넌트 내에서 매 렌더 새 schema 를 만들면 reader 함수 신원이 바뀐다. 두 곳 이상에서 같은 schema 를 쓰면 모듈 상수로 추출 (SearchPage 의 `URL_FILTERS_SCHEMA` 패턴).
    - **부재 vs 빈 값 구분이 필요한 호출자**는 `readString(key) → ""` 와 `readOptionalString(key) → undefined` 를 명시적으로 골라 쓴다. SearchPage(q 는 string 입력 controlled component → `""`), DocumentListPage(filter dict 는 `undefined` 가 자연스러움) 둘 다 자기 시맨틱대로 갔다.
    - **bespoke sentinel 이 필요한 호출자**는 본 러너 대신 직접 reader 조합 또는 inline `.get()` 으로 빠질 것. 강제하지 않는다.
    - 보안: reader 가 사용자 입력 URL 값을 좁히지만 SQLi 같은 백엔드 보호는 별개 — `enum`/`boundedInt`/`regex(미제공, 호출지 직접 검증)` 로 이미 검증된 값만 backend 에 보내는 것이 권장.

### 1.3 `@/lib/utils/errors` ✅ (FE-G2, 2026-04-25)

- `classifyApiError(error: unknown, fallbackMessage?: string) -> ApiErrorClassification` ✅
  - status: ✅ Existing (2026-04-25 FE-G2 승격)
  - purpose: `ApiError` / `NetworkError` / `AbortError` / 일반 `Error` / 미지의 값을 단일 분류 결과 (`{ code, status?, serverCode?, message, recoverable }`) 로 변환.
  - effects: none
  - source: `frontend/src/lib/utils/errors.ts`
  - tests: `frontend/tests/UtilsErrorsDownloadGuardsFG2.test.ts` `classifyApiError` (9 case — ApiError 4xx/5xx · NetworkError · AbortError · 일반 Error · string · null/undefined · 커스텀 fallback)
  - replaced in:
    - `features/admin/extraction-queue/helpers.ts` `mutationErrorMessage` — 본 helper 위에서 도메인별 한국어 메시지 매핑으로 재구성. 시그니처 호환 유지 (string 반환).
  - excluded by design (호출지 미마이그레이션, 후속 라운드):
    - `@/lib/api/client.ts:247` `getApiErrorMessage` 와 16 파일 분산 호출자 — 도메인 helper 들이 이 위에서 점진 재구성 가능. 본 라운드는 시범 1건만.
    - `AdminGoldenSetsPage` `classifyListError` — golden-sets 도메인 시맨틱 (auth/permission/network/server 4분류) 가 본 helper 와 다른 결 — 별 라운드 검토.
  - notes:
    - `code` 카테고리: `"api_error" | "network_error" | "abort_error" | "generic_error" | "unknown_error"`.
    - `recoverable=true`: 5xx ApiError / NetworkError / AbortError. 4xx 와 알 수 없는 값은 `false` (보수적 — retry 의미 없음).
    - 백엔드 Shared Error Contract 의 `safe_retry`/`required_scope`/`suggested_next_action` 은 헤더/detail 노출 정착 후 본 결과 객체에 추가 검토.

### 1.4 `@/lib/utils/download` ✅ (FE-G2, 2026-04-25)

- `downloadJsonFile(filename: string, data: unknown, options?: { space?: 0 | 2; type?: string }) -> void` ✅
  - status: ✅ Existing (2026-04-25 FE-G2 승격)
  - purpose: `JSON.stringify` → `Blob` → `URL.createObjectURL` → `<a>` click → `revokeObjectURL` 보일러를 1줄로. SSR 안전.
  - effects: DOM (임시 `<a>` 생성/click) · external_io (브라우저 다운로드).
  - source: `frontend/src/lib/utils/download.ts`
  - tests: `frontend/tests/UtilsErrorsDownloadGuardsFG2.test.ts` `downloadJsonFile` (SSR no-op / Blob/URL/click/revoke 흐름 — globalThis stub 으로 시뮬레이션)
  - replaced in:
    - `features/admin/golden-sets/AdminGoldenSetsPage.tsx` — 로컬 `downloadJsonFile(data, filename)` 을 표준 helper 의 thin wrapper 로 교체 (호출자 인자 순서 호환 유지).
    - `components/chat/ConversationList.tsx` `handleExport` — `Blob`/`URL.createObjectURL`/`a.click`/`revokeObjectURL` 인라인 5줄 제거.
  - notes:
    - **SSR 가드**: `typeof window === "undefined" || typeof document === "undefined"` 시 no-op 후 반환 (Next.js RSC 안전).
    - **메모리 회수**: `try/finally` 로 `URL.revokeObjectURL` 호출 보장.
    - `space` 기본 2 (가독). 압축은 `space: 0`. `type` 기본 `"application/json"` — 다른 MIME 도 가능.

### 1.5 `@/lib/utils/guards` ✅ (FE-G2, 2026-04-25)

- `isString(value: unknown): value is string` ✅
- `isNonEmptyString(value: unknown): value is string` ✅
- `isPlainObject(value: unknown): value is Record<string, unknown>` ✅
  - status: ✅ Existing (2026-04-25 FE-G2 신설). 호출지 마이그레이션은 ⚠️ 별 라운드.
  - purpose: `typeof x === "string"` / `x !== null && typeof x === "object"` 패턴의 가독성 + narrowing.
  - effects: none
  - source: `frontend/src/lib/utils/guards.ts`
  - tests: `frontend/tests/UtilsErrorsDownloadGuardsFG2.test.ts` (isString 2 / isNonEmptyString 3 / isPlainObject 6 = 11 case)
  - excluded by design (호출지 미마이그레이션):
    - 51회 분산된 `typeof === "string"` / `null !== ... && typeof ... === "object"` 호출 — 의미 동등성 검증 비용 vs 일관성 이득의 균형 미정. 본 라운드는 신설만, 마이그레이션은 ESLint custom rule 도입 후 점진.
  - notes:
    - `isNonEmptyString`: 빈 문자열 `""` 와 공백만 `"   "` 모두 `false` (`trim` 후 길이 검사).
    - `isPlainObject`: `Object.prototype` 또는 `null prototype` 만 참 — Date/Map/Set/Error/사용자 클래스는 모두 `false`. cross-realm 객체는 `false` 일 수 있음.
    - 추후 `isNumber` / `isBoolean` 등 추가 가드는 호출 빈도 충족 시 도입.

### 1.6 `@/hooks/useDebouncedValue`

- `useDebouncedValue<T>(value: T, delay: number): T` ✅
  - status: ✅ Existing (2026-04-25 F6 승격)
  - purpose: `value` 가 `delay` ms 동안 안정될 때까지 갱신을 늦춰 반환. 자동완성·검색 입력의 rapid typing 보일러플레이트 제거.
  - effects: none (`useEffect` cleanup 으로 타이머 정리)
  - errors: none (음수·NaN delay 는 React state 갱신을 일으키지 않으므로 호출 측에서 정상 범위 보장)
  - source: `frontend/src/hooks/useDebouncedValue.ts`
  - tests: `frontend/tests/UseDebouncedValueF6.test.ts` (14 cases — 시그니처 / 첫 렌더 / debounce 전이 / rapid 변경 / delay 변경 / 같은 value 재렌더 / unmount cleanup. `require.cache` React 스텁 + `node:test` `mock.timers` 결정론)
  - replaced in:
    - `features/tags/TagChipsEditor.tsx` — 자동완성 150ms (`AUTOCOMPLETE_DEBOUNCE_MS` 상수 유지). `addTagByName` 의 직접 `setDebouncedQuery("")` 호출은 제거 — `setOpen(false)` 가 dropdown 을 닫고 `useTagAutocomplete` 가 `staleTime=30s` 캐시이므로 사용자 가시 영향 없음.
    - `features/explore/AddDocumentsToCollectionModal.tsx` — 검색 300ms. `if (!open) return` 게이팅은 hook 통일을 위해 제거. 모달 닫힘 시 useQuery `enabled: open` 으로 fetch 차단되므로 정상 사용 시 영향 없음. 닫고 즉시 (~300ms 이내) 다시 여는 케이스에서만 직전 검색어가 한 번 잔존해 stale fetch 1회 가능 (queryKey 가 안정되면 자동 refetch).
    - `features/documents/DocumentListPage.tsx` — 제목 ILIKE 300ms. debounce 와 부수효과(setFilters / `router.replace`) 를 분리: hook 이 `debouncedSearch` 를 반환하고 별도 `useEffect` 가 deps `[debouncedSearch, router, searchParams]` 로 URL 동기화. 가시 동작 동일.
  - excluded by design:
    - `hooks/useUserPreferences.ts` (`PATCH_DEBOUNCE_MS = 400`) — debounce 대상이 *값* 이 아니라 *PATCH 호출 (side-effect)*. `useDebouncedValue` 의미와 다름. 별도 `useDebouncedCallback` 후보 (1.6a) 로 후속.
    - `features/editor/DocumentEditPage.tsx:170` — 30,000ms autoSave 도 함수 호출 debounce. 동일 사유.
    - `components/chat/SearchBox.tsx` — 500ms debounce 가 onChange 콜백 호출 (input → 부모 state). 단순 hook 치환은 의미가 다르므로 보류.
  - notes:
    - delay 는 호출자가 준다 (의미에 맞춰 150 / 300 / 1000 등).
    - SSR 안전: `useEffect` 만 사용하므로 Next.js 16 RSC 와 양립.
    - `value` 비교는 `Object.is` (React deps 표준). 객체/배열은 호출자가 stable reference 또는 primitive 로 좁혀 전달.

- `useDebouncedCallback<F>(fn: F, delay: number): readonly [debounced: F, flush: () => void, cancel: () => void]` ✅
  - status: ✅ Existing (2026-04-25 §1.6a 승격)
  - purpose: 함수 호출 자체를 debounce. `useDebouncedValue` 와 달리 *값* 이 아니라 *부수효과(PATCH·저장·콜백)* 를 늦춰서 1회만 발화한다.
  - effects: none (timer 만 사용. 실 부수효과는 호출자의 `fn` 가 책임)
  - errors: none (음수·NaN·Infinity delay 는 setTimeout 기본 동작에 위임)
  - source: `frontend/src/hooks/useDebouncedCallback.ts`
  - tests: `frontend/tests/UseDebouncedCallbackF1_6a.test.ts` (22 cases — 시그니처 / 기본 debounce / last-call-wins / flush / cancel / unmount auto-cancel / fn 최신성 / delay 변경 / delay=0 / 튜플 identity. F6 와 동일한 `require.cache` React 스텁 + `node:test` `mock.timers` 결정론.)
  - replaced in:
    - `hooks/useUserPreferences.ts` — PATCH 400ms. merge 시맨틱(여러 patch 누적)은 호출자의 `pendingRef` 로 유지하고, 본 훅은 timer 메커니즘만 책임. `flushPending()` 으로 즉시 발사 분기 단순화. **부수 발견·수정**: 기존 unmount cleanup 의 deps `[flush]` 가 react-query mutation 객체 신원 변동에 따라 *중간* 에도 cleanup 을 트리거해 pending PATCH 를 즉시 발사하던 잠재 버그를 ref-based + `[]` deps 로 고정.
    - `features/editor/DocumentEditPage.tsx` — autoSave 30s. **시맨틱 변경 (2026-04-25 철균 합의)**: 기존은 "isDirty=true 진입 시점부터 30s 후 1회" (deps `[isDirty, handleSave]`) 였으나, 진짜 debounce 로 변경 — `markDirty()` 시점(편집/제목 변경)마다 timer reset, 마지막 입력 후 30s 경과 시 1회 저장. 명시 저장(Cmd+S/버튼)은 `cancelAutoSave()` 후 mutate.
    - `components/chat/SearchBox.tsx` — onChange 500ms. 기존 `useEffect(setTimeout)` 패턴 제거. `handleInputChange` 안에서 `setInput(next) + debouncedOnChange(next)` 직접 호출. `onBlur` 에서 `flushOnChange()`, clear 버튼에서 `cancelOnChange()` 후 즉시 onChange("").
  - notes:
    - **마운트 시 `[debounced, flush, cancel]` 튜플은 stable identity**. useEffect deps 에 안전하게 넣을 수 있고, React.memo 자식의 무효화도 일으키지 않는다 (useCallback `[]` + useMemo).
    - **fn 은 호출 시점에 stable 일 필요 없음** — 매 렌더 ref 로 최신 클로저를 잡는다. delay 도 마찬가지.
    - **Unmount 시 자동 cancel** (실행하지 않음). flush-on-unmount 가 필요한 호출자(useUserPreferences 의 사용자 의도 보존)는 별도 `useEffect cleanup` 으로 `flush()` (또는 직접 fn 호출) 를 부른다. **그 cleanup 의 deps 는 반드시 ref 우회 + `[]`** — 그렇지 않으면 fn 신원 변동 시 cleanup 이 mid-render 에 트리거되며 의도와 무관하게 발사된다 (실측 회귀로 발견).
    - **Chrome 실측 (2026-04-25)**: SearchBox 5 글자 빠른 입력 / clear / blur flush 모두 정상, 콘솔 에러 0. 뷰 모드 토글 4 회 빠른 클릭 → 마지막 호출 + 400ms 만에 1회 PATCH.

### 1.7 `@/lib/constants/{labels,badges}` ✅ (FE-G3, 2026-04-25)

- labels (5종): `EXTRACTION_STATUS_LABELS` · `GOLDEN_SET_STATUS_LABELS` · `GOLDEN_SET_DOMAIN_LABELS` · `EVALUATION_RUN_STATUS_LABELS` · `EVALUATION_METRIC_LABELS` ✅
- badges (3종): `EXTRACTION_STATUS_BADGE_CLASSES` · `GOLDEN_SET_STATUS_BADGE_CLASSES` · `EVALUATION_RUN_STATUS_BADGE_CLASSES` ✅
  - status: ✅ Existing (2026-04-25 FE-G3 승격, 8 객체 = 5 도메인 라벨 + 3 도메인 배지)
  - purpose: admin 페이지 전반에 흩어진 상태/도메인 한글 라벨과 Tailwind 배지 className 맵을 중앙화.
  - effects: none
  - source:
    - `frontend/src/lib/constants/labels/{extraction,goldenSet,evaluation}.ts` + `index.ts`
    - `frontend/src/lib/constants/badges/{extraction,goldenSet,evaluation}.ts` + `index.ts`
  - tests: `frontend/tests/ConstantsLabelsBadgesFG3.test.ts` (21 case — 라벨 비공백 / 배지 Tailwind 토큰 / 키 일치 / 색상 정책 회귀 / thin re-export 일관성)
  - replaced in:
    - `features/admin/extraction-queue/constants.ts` — 라벨/배지 인라인 정의 제거 후 thin re-export (외부 import path 호환 보존). `EXTRACTION_STATUS` enum 과 `isValidExtractionStatus` 가드는 도메인 정책이라 잔존.
    - `features/admin/golden-sets/AdminGoldenSetsPage.tsx` — 인라인 `DOMAIN_LABELS` / `STATUS_LABELS` / `STATUS_BADGE_STYLE` 3 객체를 신규 import 의 로컬 별칭으로 교체 (호출자 줄 수 변경 0).
    - `features/admin/evaluations/helpers.ts` — `STATUS_LABEL` / `STATUS_BADGE_STYLE` / `METRIC_LABELS` 3 객체를 thin re-export. 정책 상수 (`METRIC_THRESHOLDS` / `METRIC_LOWER_IS_BETTER` / `scorePasses`) 는 도메인 정책이라 helper 잔존.
  - notes:
    - **labels / badges 분리** — i18n 도입 시 labels 만 i18n 파일로 이관 (badge className 은 그대로).
    - **thin re-export 보존** — 기존 호출지 16+ 파일이 `STATUS_LABEL` / `STATUS_BADGE_STYLE` 같은 짧은 이름을 import 하므로 해당 모듈에서 그대로 export 유지. 신규 호출자는 `lib/constants/{labels,badges}` 의 명시적 이름을 사용.
    - **GoldenSetDomain 은 라벨만** — 상태가 아닌 카테고리라 배지 색 매핑이 도메인 정책상 부적합 (사용자 정의 도메인이 추가될 수 있음).
    - **EvaluationMetricKey 도 라벨만** — 메트릭 표시명만 중앙화. threshold/inverted 정책은 helper 잔존.

### 1.8 `@/lib/styles/tokens.ts` (className 토큰) ✅ (FE-G4, 2026-04-25)

- `BADGE_BASE` ✅
- `ALERT_ERROR` · `ALERT_WARNING` · `ALERT_INFO` · `ALERT_SUCCESS` ✅
  - status: ✅ Existing (2026-04-25 FE-G4 승격, 5 토큰)
  - purpose: 배지 베이스 (`inline-flex items-center px-2 py-0.5 rounded text-xs font-semibold`) + 4 색상 alert 박스 (`rounded-lg bg-*-50 border border-*-200 px-4 py-3 text-sm text-*-700|800 font-medium`) 의 반복을 토큰화.
  - effects: none (순수 문자열 상수)
  - source: `frontend/src/lib/styles/tokens.ts`
  - tests: `frontend/tests/StylesTokensFG4.test.ts` (14 case — BADGE_BASE 핵심 토큰 / 색상 미포함 / cn 합성 / ALERT 4종 색상 매핑 / 4 토큰 유니크 / 비공백)
  - replaced in (시범):
    - `features/admin/extraction-queue/AdminExtractionQueuePage.tsx` `StatusBadge` — `cn(BADGE_BASE, EXTRACTION_STATUS_BADGE_CLASSES[status])`.
    - `features/admin/evaluations/AdminEvaluationsPage.tsx` `StatusBadge` — 동일 패턴.
    - `features/admin/evaluations/AdminEvaluationDetailPage.tsx` — 동일.
    - `features/admin/evaluations/AdminEvaluationComparePage.tsx` — 동일.
    - `app/login/page.tsx` — `${ALERT_ERROR} flex items-start gap-3` (cn 으로 layout 추가 합성).
    - `app/register/page.tsx` — 단독 `ALERT_ERROR`.
    - `app/reset-password/page.tsx` — 단독 `ALERT_ERROR`.
  - excluded by design (점진 후속):
    - 변형 (`rounded-full` 배지, `px-3 sm:px-4` 같은 responsive padding) 23+ 사이트 — 정확히 같은 className 인 곳만 본 라운드 시범, 변형 사이트는 별 라운드.
    - 다크모드 토큰 (`dark:bg-red-950 dark:text-red-200` 등) — preferences.theme 이미 도입되었으나 본 라운드 토큰은 light 만. 다크 변형은 별 라운드 (admin 페이지 다크모드 전면 도입과 함께).
    - `text-red-600` 변형 (account/security, account/sessions 등) — 색상 강도가 다름 (700 vs 600). 별 토큰 후보.
    - Toast / Modal 컨테이너 토큰 — 별 도메인.
  - notes:
    - **사용 패턴**: 단독 `className={ALERT_ERROR}` 또는 `className={cn(BADGE_BASE, "bg-amber-100 text-amber-800")}` (tailwind-merge 가 충돌 해소).
    - **base 에 색상 미포함** — 호출자가 색상 토큰 합성. 회귀 테스트로 base 에 `bg-*-` / `text-(red|green|blue|amber|...)-` 가 없음을 강제.
    - **다크모드 후속**: 본 모듈에 `dark:` 변형을 추가할지 또는 별 모듈 (`tokens.dark.ts`) 로 둘지 다음 라운드 결정.

---

## 2. 도입 보류 / 주의 후보 (⚠️ 검토 필요)

### 2.1 `invalidateQueries` 래퍼 ⚠️
- 79회 등장하지만 각 호출의 queryKey 조합이 도메인별로 달라 통합이 과도한 추상화가 된다.
- **결정**: 도입하지 않는다. 대신 `useMutationWithToast` 의 JSDoc 에 "성공 시 relevant queryKey 를 invalidate 하라" 가이드를 명시한다.

### 2.2 Dialog/Modal 공통 래퍼 ⚠️
- `useFocusTrap` 훅은 이미 존재하나 일부 모달(`FolderMoveDialog`, 일부 admin 페이지)에서 미적용이다.
- **결정**: 새 유틸을 만들지 않고, 미적용 컴포넌트를 후속 라운드에서 `useFocusTrap` 로 일괄 전환한다.

### 2.3 `scrollY` 저장/복원 헬퍼 ⚠️
- 등장 2회 수준이라 threshold 경계. 관용구가 굳어지지 않아 일반화가 어렵다.
- **결정**: 당장 도입하지 않고, 3회 초과 시 재검토.

---

## 3. 감사 메타

- 후보 선정 근거: `docs/함수도서관/감사보고서_2026-04-25.md` §4
- 스팟 체크 결과 요약:
  - `toLocaleDateString` 17회, `new URLSearchParams` 18회, `URL.createObjectURL` 2회, `invalidateQueries` 79회, local `formatDate` 재선언 4건 (`@/lib/utils` 원본 제외)
  - debounce `setTimeout(..., 300)` 직접 매치는 0이었으나, `SEARCH_DEBOUNCE_MS` / `AUTOCOMPLETE_DEBOUNCE_MS` 상수로 이름이 달라져 매칭되지 않은 것. 실사용은 3곳 이상 확인 완료.
- 후속 구현은 1.1 `formatDateTime` → 1.6 `useDebouncedValue` → 1.2 `toQueryString` 순으로 소규모 PR 분할 권장 (CONSTITUTION 제32조 Reviewable PRs).
