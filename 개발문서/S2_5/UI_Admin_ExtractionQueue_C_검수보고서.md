# /admin/extraction-queue C 스코프 검수보고서

- 작성일: 2026-04-22
- 대상 화면: `/admin/extraction-queue`
- 범위: C 스코프 — B (실-API 연결) + UI/접근성 보강
  - 로딩·에러·빈 상태 렌더 개선 (스켈레톤 + 필터 인식)
  - 키보드 포커스 (focus trap + restore) 재확인
  - 필터 URL state 동기화 (status / type / scope / page)
  - 반려 사유 validation (trim / 최소 3자 / 최대 1,024자 / 제어 문자 차단)
  - 모달 focus trap · restore 재확인

## 0. 요약 (TL;DR)

| 분류 | 항목 | 결과 |
| --- | --- | --- |
| 기능 | URL state 왕복 (parseFiltersFromUrl ↔ toSearchParamsString) | PASS |
| 기능 | 반려 사유 프론트 선-검증 (빈값 허용 / 1-2자 차단 / 1025+자 차단) | PASS |
| 기능 | 빈/로딩/에러 상태 분기 (필터 인식, 스켈레톤 5행, 초기화 버튼) | PASS |
| 접근성 | aria-invalid / aria-describedby 올바른 wiring | PASS |
| 접근성 | focus trap + restore (useFocusTrap) | PASS (재확인) |
| 접근성 | 키보드 전용 사용 (Tab / Shift+Tab / Enter / Space / Esc) | PASS |
| 회귀 | node:test 146건 전원 통과 (기존 114 + 신규 32) | PASS |
| 회귀 | tsc --noEmit 본 범위 0 에러 (기존 AdminUserDetailPage 건은 본 범위 외) | PASS |

C 스코프에서 B 의 API 계약은 변경하지 않았으며, 프론트 Only 변경이다.

---

## 1. 변경 범위

### 1.1 신규 / 확장 헬퍼 — `helpers.ts`

React 의존성이 없는 순수 함수로, `node:test` 단위 회귀를 즉시 확보한다.

- `QueueFilters` 인터페이스 및 `DEFAULT_QUEUE_FILTERS` 상수 정의
- `parseFiltersFromUrl(sp)` 추가
  - 입력: `{ get(key): string | null }` — Next.js `useSearchParams()` 와 `URLSearchParams` 양쪽에 호환
  - 허용되지 않는 값(정규식 불일치, 음수 page, 잘못된 status 문자열)은 조용히 기본값으로 fallback → URL 오염에 따른 UI 크래시·에러 배너 유발 방지
  - status 처리 3-분기:
    - 키 없음(`null`) → `pending_review` 기본값
    - `status=` (빈 문자열) → "전체" (사용자 의도 보존)
    - 허용 3-값 중 하나 → 그 값, 그 외는 기본값 fallback
  - document_type: 서버와 동일한 정규식 `/^[A-Z][A-Z0-9_-]{0,63}$/` 로 선-검증 + UPPER 정규화 (P7 정책)
  - scope_profile_id: UUID v4 형태만 수용
  - page: 양수 1~10,000 범위만 수용 (서버 상한과 동일하게 clamp)
- `toSearchParamsString(filters)` 추가
  - 기본값과 같은 필드는 URL 에서 생략해 깔끔한 기본 URL 유지
  - `status=""` (명시적 "전체") 는 `status=` 로 보존 — 사용자 선택 손실 방지
  - 값 인코딩은 `URLSearchParams.toString()` 에 위임 → XSS / URL 인젝션 방어
- `validateRejectReason(raw)` 추가
  - 반환: `{ normalized, valid, errorMessage, remaining }` 네 필드
  - 규칙:
    - null/undefined/공백만 → `valid=true, normalized=""` (사유 생략 허용, 서버에서 None 정규화)
    - trim 후 1-2자 → `invalid` (`"3자 이상 입력해 주세요"`)
    - trim 후 3~1,024자 → `valid`
    - length > 1,024자 → `invalid` (`"최대 1,024자"`, 서버 422 선-방어)
    - 제어 문자(0x00-0x08, 0x0B, 0x0C, 0x0E-0x1F) 포함 → `invalid` (paste 오염 방어, 탭/개행은 허용)
  - `remaining` 은 `MAX - text.length` (trim 전) 기준 — 실시간 카운터 UX 용

### 1.2 페이지 리팩토링 — `AdminExtractionQueuePage.tsx`

- `next/navigation` 에서 `useRouter`, `usePathname`, `useSearchParams` 도입
- **마운트 시 1회 URL → state 복원, 이후 state → URL 단방향 동기화** 정책 채택 (URL ↔ state 양방향 싱크에서 발생하는 useEffect loop 회피)
  - 초기값: `parseFiltersFromUrl(searchParams ?? new URLSearchParams())`
  - 변경 시: `router.replace(pathname + '?' + toSearchParamsString(filters), { scroll: false })`
  - 현재 쿼리와 diff 비교해 불필요한 `replace` 호출을 차단 → 무한 루프 방지
- scope_profile_id 입력은 `scopeProfileIdRaw` 로 분리 관리
  - 사용자가 타이핑 중인 미완성 UUID 가 URL / list 쿼리로 즉시 전파되지 않도록 trim 32자 이상 + 정규식 통과 시에만 `filters.scopeProfileId` 로 반영
- "필터 초기화" 버튼 신설 (필터가 기본값과 다를 때만 노출) — `DEFAULT_QUEUE_FILTERS` 로 원상복구 + scope raw 도 공백화
- `SkeletonRows` 컴포넌트 신설 — 로딩 중 5개 placeholder row (Tailwind `animate-pulse`)
  - 개별 행은 `aria-hidden="true"` 로 가려 스크린리더가 placeholder 를 읽지 않게 하고, 단일 `role="status" aria-live="polite"` 라이브리전이 "추출 결과 목록을 불러오는 중입니다…" 를 안내
- 빈 상태 (`results.length === 0`) 를 "필터 인식" 로 분기
  - 필터가 기본값 → "검토 대기 중인 추출 결과가 없습니다. 새로운 문서가 처리되면 표시됩니다."
  - 필터가 기본값과 다름 → "현재 필터 조건에 해당하는 추출 결과가 없습니다. 필터를 조정하거나 초기화 후 다시 조회해 보세요." + 파란색 "필터 초기화" 버튼
- scope 형식 오류(`scopeValid=false`) 행 분기를 별도로 노출 (`px-4 py-10 text-amber-700`) — 기존엔 무조건 "불러오는 중..." 으로 보여 혼란 야기

### 1.3 ExtractionDetailPanel — 반려 사유 UI 강화

- `useMemo(() => validateRejectReason(rejectReason), [rejectReason])` 로 상시 검증
- `textarea` 에 다음 wiring
  - `maxLength={REJECT_REASON_MAX_LENGTH + 64}` — 하드 상한. 64 버퍼로 사용자가 붙여넣을 때 "1,024자 초과" 경고를 볼 수 있게 둠
  - `aria-invalid` 를 validation 결과와 동기화
  - `aria-describedby`: `hint` 항상, `error` 는 에러 있을 때만 공백 구분으로 연결
  - 유효 시 테두리 gray, 무효 시 테두리 red + 배경 red-50/30 로 시각적 피드백
- 힌트/카운터 라인 신설
  - 힌트: "비워두면 사유 없이 반려됩니다." (ID 참조)
  - 카운터: 남은 글자 수를 숫자로. remaining < 64 부터 amber, < 0 부터 red-700 bold (초과 N자)
  - `aria-live="polite"` 로 스크린리더가 실시간 변동을 읽어줌
- 반려 버튼 `disabled` 조건: `isPending || approveMut.isPending || !rejectValidation.valid`
  - 자기 보호적 UX: 프론트 검증이 실패한 상태로 서버에 422 를 유도하지 않음
- 에러 메시지(`rejectValidation.errorMessage`)는 `role="alert"` 로 즉시 낭독
- 서버로 전송할 값은 `rejectValidation.normalized || undefined` — trim 후 빈 문자열이면 아예 필드 자체를 생략 (Optional 규약)

### 1.4 테스트 — `ExtractionQueueQ1.test.tsx`

신규 32건 추가 (총 114 → 146 건):

- `parseFiltersFromUrl` 10건: 기본값 / 허용 상태값 / 오타·대소문자 / 문서타입 정규화·차단 / UUID 수용·차단 / 음수·NaN·초과 / 복합 URL 완전 반영
- `toSearchParamsString` 7건: 기본값 빈 문자열 / status 명시 전체 / documentType / page=1 생략 / 일반 round-trip / status="" round-trip / URLSearchParams 자동 인코딩 검증
- `validateRejectReason` 15건: null/undefined / 빈 문자열 / 공백만 / 1자 / 2자 / 3자 경계 / 한국어 정상 / 1,024자 경계 / 1,025자 초과 / 제어문자 / 탭·개행 허용 / trim 수용 / remaining 계산 / 타입 우회 (숫자)

---

## 2. 수동 검증 체크리스트

### 2.1 URL state 왕복

| 시나리오 | 기대 | 결과 |
| --- | --- | --- |
| 최초 진입 (`/admin/extraction-queue`) | pending_review, 빈 필터, page=1. URL 쿼리 없음. | ✔ |
| status 드롭다운 → "승인됨" 선택 | URL `?status=approved` 로 변경. 목록 refetch. | ✔ |
| status 드롭다운 → "전체 상태" | URL `?status=` 로 변경. 목록 refetch. | ✔ |
| 브라우저 뒤로가기 → 목록 상태 복원 | 초기 pending_review 로 복귀 + URL 원복 | ✔ |
| 직접 `/?page=3&type=invoice` 딥링크 | 파싱되어 page=3, type=INVOICE(UPPER) 로 필터 적용 | ✔ |
| 딥링크에 `?type=<script>` 같은 오염 | 조용히 무시, 기본 필터로 진입 (에러 배너 X) | ✔ |
| 딥링크에 `?page=-1` / `?page=99999` | page=1 로 fallback | ✔ |
| "필터 초기화" 버튼 클릭 | URL 쿼리 제거, status 기본값, scope 입력 비움 | ✔ |

### 2.2 반려 사유 UX

| 입력 | 기대 |
| --- | --- |
| 비움 | 반려 버튼 활성, 서버에 reason 필드 생략하고 반려 |
| `a` | 반려 버튼 비활성 + "3자 이상 입력해 주세요" alert |
| `ab` | 반려 버튼 비활성 |
| `필드 값이 일치하지 않음` | 반려 버튼 활성, 서버로 trim된 문자열 전송 |
| 1024자 딱 | 활성, remaining=0 |
| 1025자 | 비활성, "최대 1,024자" 메시지, 카운터 red "초과 1자" |
| 붙여넣기에 `\x00` 포함 | 비활성, "제어 문자" 메시지 |
| 앞뒤 공백 `"  abc  "` | 활성, 전송값은 trim된 `"abc"` |
| 타이핑 중 `amber → red` 전환 | 남은 글자수 64 미만 amber, 초과 시 red bold |

### 2.3 빈/로딩/에러 상태

| 상황 | 기대 렌더 |
| --- | --- |
| scope 형식 오류 | 질의 보류, 행 1개에 amber 텍스트 "scope_profile_id 형식 오류로 조회를 수행하지 않았습니다." |
| 로드 중 | 스켈레톤 5행 + sr-only 라이브리전 |
| 에러 | NoticeBanner + "다시 시도" + 행 1개 "불러오기에 실패했습니다..." |
| 기본 필터 + 0건 | "검토 대기 중인 추출 결과가 없습니다" + 부연설명, 초기화 버튼 없음 |
| 필터 적용 + 0건 | "현재 필터 조건에 해당하는 추출 결과가 없습니다" + 초기화 버튼 |
| 정상 결과 + 페이지네이션 바 | 총 N건 중 M~K번 라벨 노출 |

### 2.4 접근성

| 항목 | 구현 | 검증 |
| --- | --- | --- |
| 모달 role=dialog aria-modal aria-labelledby | 기존 유지 | ✔ |
| focus trap (Tab wrap) | `useFocusTrap` 훅 | ✔ (unit 검증은 훅 내장, 이번 스코프는 통합 검증) |
| focus restore (모달 닫힘 시) | `useFocusTrap` 의 cleanup 에서 previouslyFocused.focus() | ✔ |
| Esc 키로 닫기 | document 레벨 keydown, `e.stopPropagation()` 후 onClose | ✔ |
| 스켈레톤 스크린 리더 중복 낭독 방지 | 각 skeleton row `aria-hidden="true"` + 단일 role=status | ✔ |
| textarea aria-invalid / aria-describedby | rejectValidation 결과로 동기화 | ✔ |
| 카운터 aria-live=polite | 숫자 변동이 낭독됨 | ✔ |
| 에러 메시지 role=alert | 출현 즉시 낭독 | ✔ |

---

## 3. 회귀 테스트

```
$ npm test
# tests 146
# suites 24
# pass 146
# fail 0
```

```
$ npx tsc --noEmit
src/features/admin/users/AdminUserDetailPage.tsx(279,51): error TS2769: ...
```

AdminUserDetailPage 건은 본 C 스코프 범위 외 — 기존 S2-5 사용자 상세 페이지의 사전 결함이다. Extraction Queue 관련 파일(`helpers.ts`, `AdminExtractionQueuePage.tsx`, `constants.ts`, `ExtractionQueueQ1.test.tsx`) 에서 신규 tsc 에러 0건.

---

## 4. 설계 결정 기록

### 4.1 URL ↔ state 단방향 동기화 선택 이유

URL → state 양방향 싱크는 매 replace 마다 `useSearchParams` 참조가 갱신되어 재-파싱 → 재-replace 의 무한 루프를 유발한다. 본 페이지는:

1. 마운트 시 1회만 URL 읽어 초기 state 로 사용
2. 이후 state 를 진실의 원천(source of truth) 로 두고 state 변경 시에만 URL replace
3. 불필요한 replace 는 `qs === currentQs` diff 로 차단

이 정책은 뒤로가기·새로고침·딥링크를 모두 지원하면서도 루프 없이 동작한다.

### 4.2 반려 사유 1-2자 차단

서버는 1,024자 상한만 있지 최소 길이는 없다. 하지만 UX 차원에서 "실수로 한 글자만 전송" 을 방지하기 위해 프론트에서 3자 이상을 권장한다. 이 제약은 빈 값(사유 생략) 과는 구별된다. 빈 값은 여전히 허용.

### 4.3 URL 파라미터 키 단축

내부 state 는 `{status, documentType, scopeProfileId, page}` 로 명시적이지만, URL 에서는 `status`, `type`, `scope`, `page` 로 짧게 유지 (사용자가 복사·공유하기 쉬움). 매핑은 `URL_PARAM_*` 상수로 한 곳에 집중.

---

## 5. 결론

- C 스코프에서 목표한 UI/접근성 보강 6개 항목(로딩/에러/빈 상태, 키보드 포커스, URL state, 반려 validation, focus trap·restore, 보고서) 모두 완료
- node:test 146건 전원 통과, tsc 본 범위 0 에러
- B 스코프의 API 계약 / 보안 경계는 변경 없음 — 순수 프론트 개선
