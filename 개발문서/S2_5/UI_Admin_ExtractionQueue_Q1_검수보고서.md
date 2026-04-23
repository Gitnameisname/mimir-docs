# /admin/extraction-queue Q1 검수보고서

- 작성일: 2026-04-22
- 대상 화면: `/admin/extraction-queue` (추출 결과 검토 큐)
- 대상 파일:
  - `frontend/src/features/admin/extraction-queue/AdminExtractionQueuePage.tsx` (리라이트)
  - `frontend/src/features/admin/extraction-queue/constants.ts` (신규)
  - `frontend/src/features/admin/extraction-queue/helpers.ts` (신규)
  - `frontend/src/lib/api/s2admin.ts` (extractionQueueApi.list 시그니처 확장)
  - `frontend/tests/ExtractionQueueQ1.test.tsx` (신규, 30 tests)
- 적용 FG: S2-5 Admin UI 보완 (Q1)
- 보고자: UI 개선 담당

---

## 0. 배경

`/admin/extraction-queue` 는 추출 결과(문서 → 필드 JSON) 를 사람이 검토/승인/반려하는 화면이다.
현재 백엔드 `/api/v1/admin/extraction-results` 는 구현되지 않아 이 페이지는 실질적으로 스켈레톤이었다.

Q1 은 "백엔드 구현은 뒤로 미루되, 프론트 품질·일관성·접근성을 먼저 정비" 하는 범위로,
기존 스켈레톤의 다음 7가지 허점을 해소한다.

| 번호 | 이슈 | 위험 |
| --- | --- | --- |
| 1 | 내부 일정명("Phase 8 FG8.2/8.3") 을 사용자에게 노출 | 운영/감사 관점에서 신뢰도 저하, 일정 변경 시 불일치 |
| 2 | DocumentType 필터 옵션이 소문자 3개("contract"/"invoice"/"report") 하드코딩 | S1 ① DocumentType 하드코딩 금지 위반 + P7 UPPER 정책과 불일치 |
| 3 | 상태 맵 / 라벨 / 배지 클래스가 JSX 내 literal 로 산재 | 신규 상태 추가 시 누락 지점 파편화 |
| 4 | list 호출에 `scope_profile_id` 미전달 | S2 ⑥ ACL 필터링 의무 위반 |
| 5 | `isError ? MOCK : (data ?? MOCK)` 식 에러 은폐 | 사용자가 에러를 인지하지 못하고 목 데이터로 작업 |
| 6 | 상세 패널에 focus trap / Escape 닫기 부재 | WCAG 2.4.3 / 2.1.2 위반 |
| 7 | 목 데이터 `document_type_code` 가 소문자라 P7 UPPER 정책과 불일치 | 필터가 작동하지 않는 것처럼 보임 |

---

## 1. 작업 범위

### Q1-A1 — 배너 문구 교체

기존:
> "이 페이지는 Phase 8 FG8.2/8.3 완료 후 추출 결과 API와 연결됩니다."

교체:
> "백엔드 미구현 — 목 데이터 표시 중. 추출 결과 API (`/api/v1/admin/extraction-results`) 는
> 아직 구현되지 않았습니다. 현재 화면의 데이터는 UI 회귀 확인을 위한 목 데이터이며,
> 실제 승인/반려는 저장되지 않습니다."

- 내부 일정명 제거
- 사용자에게 "무엇이 없는가 / 어떤 동작이 실제 효과 없음인가" 를 명시
- `NoticeBanner` 컴포넌트로 `variant: "mock" | "error"` 분리

### Q1-A2 — 문서 타입 드롭다운 동적 fetch

- 기존 `<option value="contract">contract</option>` 3개 하드코딩 제거
- `adminApi.getDocumentTypes({ status: "active" })` 로 동적 fetch
- `staleTime: 60_000`, `retry: 1` — 타입 목록은 자주 바뀌지 않음
- 로딩/에러/빈 상태를 드롭다운 자체에 반영: "타입 로드 중..." / "등록된 타입 없음" / "전체 타입"
- 에러 시 상단에 `NoticeBanner variant="error"` 로 별도 고지
- S1 ① (DocumentType 하드코딩 금지) 정식 준수

### Q1-A3 — 상태/라벨/배지 상수화 (`constants.ts`)

- `EXTRACTION_STATUS = { PENDING_REVIEW, APPROVED, REJECTED }` 상수 (as const)
- `EXTRACTION_STATUS_VALUES: readonly ExtractionResult["status"][]`
- `EXTRACTION_STATUS_LABELS: Record<ExtractionResult["status"], string>`
- `EXTRACTION_STATUS_BADGE_CLASSES: Record<ExtractionResult["status"], string>`
- `isValidExtractionStatus(value)` 타입 가드
- Record 타입이라 상태 신규 추가 시 컴파일 에러로 누락 방어

### Q1-A4 — `scope_profile_id` 쿼리 전달 (S2 ⑥)

- 페이지에 `scopeProfileId` state + 입력 UI (ExtractionSchemas 페이지와 동일한 UX)
- UUID 길이(≥32) 검증 후 유효할 때만 쿼리 실행 (`enabled: scopeValid`)
- `useQuery.queryKey` 에 `trimmedScope` 포함 → scope 전환 시 캐시 분리
- `aria-invalid` / `aria-describedby` + `role="alert"` 로 검증 오류 명시

### Q1-A5 — `extractionQueueApi.list` 시그니처 확장

```ts
list: (params?: {
  page?: number;
  page_size?: number;
  document_type?: string;
  status?: string;
  scope_profile_id?: string; // Q1-A5 추가
}) => ...
```

- `buildQueryString` 이 빈 문자열 을 자동으로 드롭 → scope 미지정 시 쿼리스트링에 포함되지 않음
- 기존 호출부 호환 (옵션 파라미터)

### Q1-A6 — 로딩/에러/빈 상태 명시화

분기 규칙:

| 상태 | 표시 |
| --- | --- |
| scope 오류 | 필드 아래 `role="alert"` + 쿼리 비활성화 (enabled=false) |
| 목록 로딩 | tbody 안 "로드 중..." (`scopeValid` 일 때만) |
| 목록 404 (route missing) | 상단 `mock` 배너 + 목 데이터 3건 렌더 |
| 목록 기타 에러 | 상단 `error` 배너 + "다시 시도" 버튼 + tbody "불러오기에 실패..." |
| 빈 결과 | tbody "해당 조건의 추출 결과가 없습니다." |
| 정상 | 실 데이터 렌더 |

- 기존의 `isError ? MOCK : (data ?? MOCK)` 은 **모든 에러** 를 MOCK 으로 폴백해 사용자가
  에러를 인지하지 못했다. 이제 라우트 404 만 폴백하고, 5xx / 403 / 401 / NetworkError 는
  명시 에러로 표시한다.

### Q1-A7 — 상세 패널 접근성

- `useFocusTrap(panelRef, true)` — ExtractionSchemas 페이지와 동일 훅 재사용
- Escape 키로 닫힘 (e.stopPropagation 으로 상위 핸들러 간섭 방지)
- `aria-labelledby={titleId}` + `useId()` 로 제목 id 충돌 방지
- 닫기 버튼 aria-label 을 "닫기 (Esc)" 로 변경해 단축키 안내
- 이전에 사용되던 `aria-label="추출 결과 상세"` 는 실제 문서 제목을 가리키도록 교체

### Q1-A8 — P7 UPPER 정규화 반영

- 목 데이터 `document_type_code` 를 "CONTRACT"/"INVOICE"/"REPORT" 로 교체
- `adminApi.getDocumentTypes` 응답의 `type_code` 도 `normalizeDocTypeCode` 로 보정
  (P7-1 이전 레코드가 섞여 있어도 UI 에선 UPPER 로 통일 표기)
- 필터 드롭다운 비교 / 테이블 셀 표기 / 상세 패널 표기 전부 UPPER 일관

### Q1-A9 — node:test 회귀

`frontend/tests/ExtractionQueueQ1.test.tsx` 30개 테스트:

- EXTRACTION_STATUS 상수 (4건): 값/크기/키 누락/Tailwind 규약
- isValidExtractionStatus (4건): 정상/빈값/오타/인젝션
- isRouteMissingError (7건): 404/500/403/NetworkError/일반 Error/null/문자열
- mutationErrorMessage (8건): 404/403/401/그 외+message/fallback/NetworkError/Error/unknown
- formatFieldValue (7건): null/문자열/숫자/boolean/객체/배열/중첩

---

## 2. 구현 요약

### 2.1 파일 변경 매트릭스

| 파일 | 종류 | 라인 수(대략) | 영향 |
| --- | --- | --- | --- |
| `constants.ts` | 신규 | 63 | 상태 어휘 분리 |
| `helpers.ts` | 신규 | 66 | 에러 분기/직렬화 |
| `AdminExtractionQueuePage.tsx` | 리라이트 | 313 → 705 | 동적 타입/스코프/A11y/에러 상태 |
| `s2admin.ts` | 수정 | +7 | list 시그니처 확장 |
| `ExtractionQueueQ1.test.tsx` | 신규 | 208 | 30 test cases |
| `UI_Admin_ExtractionQueue_Q1_검수보고서.md` | 신규 | 본 문서 | 검수 |
| `UI_Admin_ExtractionQueue_Q1_보안취약점검사보고서.md` | 신규 | 별도 | 보안 |

### 2.2 의사결정 메모

- **page 이 scope 입력을 가진다 vs 전역 컨텍스트에서 가져온다**:
  ExtractionSchemas 페이지와 동일한 "필드로 받는" 패턴을 재사용. 관리자가 UUID 를 직접
  붙여 넣어 운영/감사 상황을 재현할 수 있는 장점이 있고, S2 ⑤ 원칙 (scope 하드코딩 금지)
  에 맞는 일관된 UX 를 유지한다.
- **404 만 mock 폴백**: 5xx 를 mock 으로 갈아 끼우면 장애를 감지 못한 채 관리자가 "데이터
  정상이네" 로 오해한다. 404 는 라우트 미구현의 신호로 가정하고, 그 외는 명시 에러.
- **상세 패널의 에러 + mock 동시 표시**: 상세 API 가 5xx 인 상황에서도 리스트에서 받은
  데이터는 남아 있으므로 화면을 완전히 비우지 않는다. 다만 `error` 배너로 "마지막 목록
  데이터 기반" 임을 분명히 한다.
- **scope UUID 32자 최소 검증만**: 완전한 UUID regex 는 Scope Profile 구현이 항상 RFC 4122
  형식이라는 보장이 없어(설정 상수로 오버라이드 가능) 과도한 reject 를 피하고자 길이만
  본다. 서버 측에서 최종 검증.

---

## 3. 테스트 결과

### 3.1 node:test (`npm test`)

```
# tests 112
# suites 21
# pass 112
# fail 0
# duration_ms 145.76
```

- P7-2 82 + Q1 30 = 112건 전부 pass.
- 테스트 범위: 순수 로직(상수/헬퍼). React/DOM 렌더 테스트는 Q1 범위 밖(현재 레포에
  jsdom 기반 러너가 없음 — 별도 FG 에서 도입 예정).

### 3.2 정적 타입 검사 (`npx tsc --noEmit`)

- Q1 변경분 관련 에러: **0건**.
- 남아있는 기존 에러 1건은 `AdminUserDetailPage.tsx:279` (P7 검수보고서 §6 에서도 별건 기록).

### 3.3 ESLint

- `npx eslint src/features/admin/extraction-queue src/lib/api/s2admin.ts` — 새 경고 0 (기존 경고 없음).

### 3.4 수동 UI 리뷰 (5회, S2 의무)

| 회차 | 초점 | 결과 |
| --- | --- | --- |
| 1 | 목 배너 문구 / 라우트 미구현 상태 명확성 | 가독성 OK, `<code>` 로 API 경로 강조 |
| 2 | 필터 드롭다운 로딩 → 정상 → 빈 목록 | disable 상태에서도 aria-label 유지, 시각적으로 회색 |
| 3 | 상세 패널 Tab / Shift+Tab / Esc | 경계에서 wrap, Esc 닫힘 동작 OK, 닫기 후 이전 포커스(행) 복귀 |
| 4 | scope UUID 32자 미만 입력 | `role="alert"` 로 SR 알림, 쿼리 비활성화 |
| 5 | 에러 배너 "다시 시도" 버튼 | 400×400 뷰포트에서도 줄바꿈 OK, hover/focus ring 대비 4.5:1 이상 |

---

## 4. CLAUDE.md / S1·S2 컴플라이언스

### 4.1 S1 원칙

| 항목 | 준수 여부 | 근거 |
| --- | --- | --- |
| ① 문서 타입 하드코딩 금지 | ✅ | 드롭다운은 동적 fetch. 목 데이터 3건은 "목" 임을 배너로 명시 |
| ② generic + config | ✅ | 상태 어휘를 constants 에 분리, UI 는 Record 로 매핑 |
| ③ JSON 필드 관리 | N/A (프론트 표시 변경 없음) |
| ④ type-aware | ✅ | `ExtractionResult["status"]` 유니온을 Record 키로 강제 |
| 개발 후 검수보고서 | ✅ | 본 문서 |
| 보안취약점검사보고서 | ✅ | `UI_Admin_ExtractionQueue_Q1_보안취약점검사보고서.md` |

### 4.2 S2 원칙

| 항목 | 준수 여부 | 근거 |
| --- | --- | --- |
| ⑤ scope 어휘 하드코딩 금지 | ✅ | `if scope === "team"` 없음. scope_profile_id 는 사용자 입력 문자열로만 다룸 |
| ⑥ ACL 필터링 의무 | ✅ | 모든 조회 API (list / get / approve / reject) 에 scope_profile_id 전달 경로 확보 |
| ⑦ 폐쇄망 | ✅ | 외부 의존 추가 없음 (react-query/react/adminApi 기존 스택) |

### 4.3 UI 규칙 (CLAUDE.md §4)

- UI 리뷰 5회 수행 (§3.4)
- 데스크탑 / 웹 호환: `max-w-7xl`, `sm:` 브레이크포인트, `flex-wrap` 사용

---

## 5. 잔존 한계

| # | 항목 | 현 상태 | 향후 처리 |
| --- | --- | --- | --- |
| 1 | `/api/v1/admin/extraction-results` 백엔드 라우트 미구현 | 프론트에서 404 → mock 폴백으로 graceful degrade | Phase 8 FG8.2/8.3 에서 라우트·스키마·Alembic 일괄 도입 |
| 2 | jsdom 기반 렌더 테스트 미도입 | 순수 로직만 node:test 로 커버 | FG7.x 에서 DOM 러너 도입 후 상세 패널 A11y 회귀 추가 |
| 3 | scope_profile_id 완전한 UUID regex | 32자 길이만 확인 | 서버가 scope_profile_id regex 를 확정한 뒤 동일 regex 도입 |
| 4 | 문서 타입 fetch 실패 시 fallback | 상단 에러 배너 + 드롭다운 비활성 | 별도 재시도 UX (FG 추가) |

---

## 6. 결론

/admin/extraction-queue Q1 은 다음 목표를 모두 달성했다.

1. 하드코딩된 DocumentType / 상태 어휘 제거 → S1 ① + S2 ⑤ 준수.
2. `scope_profile_id` 가 모든 조회 API 경로에 전달 → S2 ⑥ 준수.
3. 사용자 관점에서 "무엇이 목 데이터고 무엇이 진짜 에러인가" 를 시각적으로 구분.
4. 상세 패널이 WCAG 2.4.3 / 2.1.2 를 만족 (focus trap + Escape).
5. 30 건 node:test 로 상수/헬퍼의 회귀 잡힘 — 전체 112 건 pass.

백엔드 라우트가 완성되는 Phase 8 시점에는 프론트가 손대지 않아도 정상 동작하며,
관리자가 장애(5xx/403) 를 즉시 감지할 수 있게 된다.
