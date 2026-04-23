# AdminGoldenSetsPage 목록 에러 배너 상태 코드별 분기 (#31) 보고서

- 작성일: 2026-04-21
- 범위: `frontend/src/features/admin/golden-sets/AdminGoldenSetsPage.tsx`
- 원 이슈: `docs/개발문서/S2_5/UI_Admin_GoldenSets_런타임회귀리뷰.md` §"본 리뷰에서 추가로 관찰된 파생 이슈" 1번 (#31)
- 연계 선행 작업: #30 (`item_count` 필드 누락 503) 해결 후 S2 ⑥ 오귀인 빈도가 급감한 상태에서 UX 디테일로 후속 처리

---

## 1. 문제 정의

기존 `AdminGoldenSetsPage` 는 `listQuery.isError` 일 때 다음 한 가지 문구만 출력했다.

> "골든셋 목록을 불러오지 못했습니다." + "Scope Profile 바인딩 또는 권한을 확인해 주세요. (S2 ⑥)"

이 결과로 발생한 UX 문제:

| 실제 원인 | 사용자 인지 | 유도되는 행동 | 실제로 필요한 행동 |
| --- | --- | --- | --- |
| 네트워크 단절 | "권한 문제" | 관리자 문의 | 네트워크/서버 가용성 확인 |
| 5xx (서버 장애) | "권한 문제" | 관리자 문의 → 관리자 혼란 | 잠시 후 재시도 |
| 429 (레이트 리밋) | "권한 문제" | 관리자 문의 | 잠시 후 재시도 |
| 404 (경로 오류) | "권한 문제" | Scope Profile 확인 | 배포/라우팅 확인 |
| 401 (세션 만료) | "권한 문제" | 역할 확인 | 재로그인 |
| **403 (진짜 ACL 거부)** | "권한 문제" ✅ | Scope Profile 확인 ✅ | Scope Profile 확인 ✅ |

즉, S2 ⑥ 안내는 **단 한 가지 경우(403)** 에만 정답이고 나머지 모든 케이스에서 오진을 유도했다. #30 수정 직전 재현했던 503 케이스에서도 운영자는 S2 ⑥ 메시지만 보고 "권한 설정에 문제가 있다" 고 오판했던 전력이 있음.

---

## 2. 설계

`classifyListError(err: unknown): ListErrorInfo` 를 도입해 HTTP 상태와 에러 타입을 기준으로 분기.

```ts
interface ListErrorInfo {
  title: string;     // 배너 제목 (bold)
  body: string;      // 주 설명
  hint?: string;     // 행동 가이드 (optional)
  canRetry: boolean; // "다시 시도" 버튼 노출 여부
}
```

분기 규칙 요약:

| 에러 | title | hint | canRetry |
| --- | --- | --- | :---: |
| `ApiError` status 401 | 세션이 만료되었습니다 | — | ✗ |
| `ApiError` status 403 | 이 목록을 볼 권한이 없습니다 | **Scope Profile 바인딩 또는 역할을 확인해 주세요. (S2 ⑥)** | ✗ |
| `ApiError` status 404 | 목록 엔드포인트를 찾을 수 없습니다 | 배포 버전 또는 API 경로 설정을 확인해 주세요 | ✗ |
| `ApiError` status 429 | 요청이 너무 잦습니다 | — | ✓ |
| `ApiError` status ≥ 500 | 서버에서 일시적 오류가 발생했습니다 | 잠시 후 다시 시도해 주세요… | ✓ |
| `ApiError` 그 외 4xx | 목록을 불러오지 못했습니다 | — | ✓ |
| `TypeError` (Failed to fetch) | 서버에 연결하지 못했습니다 | (body에 포함) 네트워크 상태 또는 서버 가용성을 확인해 주세요 | ✓ |
| 기타 `unknown` | 목록을 불러오지 못했습니다 | — | ✓ |

설계 원칙:

- **S2 ⑥ 은 오직 403 에만.** 다른 어느 분기에도 "S2 ⑥" 문자열을 쓰지 않는다.
- **재시도 가능성(canRetry)** 은 사용자가 즉시 버튼으로 해소할 수 있는지 여부. 401/403/404 는 재시도로 풀리지 않으므로 버튼을 숨겨 불필요한 연타를 막는다.
- **body 는 서버 메시지를 우선 노출** (`getApiErrorMessage(err, fallback)`) 하여 백엔드가 내려주는 세부 정보가 있으면 그대로 보여준다. Fallback 은 generic "골든셋 목록을 불러오지 못했습니다."
- **시각 계층** 은 기존 빨강 경고 박스를 유지(변화가 과하지 않게). title = bold, body/hint = 작은 글씨 2줄.

---

## 3. 구현

### 3.1 변경 파일

- `frontend/src/features/admin/golden-sets/AdminGoldenSetsPage.tsx`
  - `ApiError` import 추가: `import { ApiError, getApiErrorMessage } from "@/lib/api/client";`
  - `classifyListError` 헬퍼 추가 (§2 표의 구현체, 주석으로 분기 정책을 본문 위에 명시)
  - 기존 `errorMessage` 문자열 state → `errorInfo` 로 전환:
    ```tsx
    const errorInfo = listQuery.isError ? classifyListError(listQuery.error) : null;
    ```
  - 배너 JSX 를 title / body / hint / retry 버튼으로 재구성:
    ```tsx
    {errorInfo && (
      <div className="rounded-xl bg-red-50 border border-red-200 p-4" role="alert">
        <p className="text-sm font-bold text-red-800">{errorInfo.title}</p>
        <p className="text-xs text-red-700 mt-1">{errorInfo.body}</p>
        {errorInfo.hint && <p className="text-xs text-red-700 mt-1">{errorInfo.hint}</p>}
        {errorInfo.canRetry && (
          <button type="button" onClick={() => listQuery.refetch()}
            className="mt-2 px-3 py-1 text-xs font-semibold rounded-lg bg-white border border-red-300 text-red-700 hover:bg-red-100 focus:outline-none focus-visible:ring-2 focus-visible:ring-red-500 min-h-[32px]">
            다시 시도
          </button>
        )}
      </div>
    )}
    ```
  - 빈 상태 가드: `!errorMessage` → `!errorInfo` 로 교체하여 에러 시에 빈 상태 메시지가 중복 출력되지 않게 유지.

### 3.2 변경하지 않은 것

- API 클라이언트(`client.ts`)·`goldenSetsApi` 서비스·React Query 키 / invalidate / 캐시 전략은 불변.
- 상세 패널(오른쪽)·모달·아이템 추가 폼 등 다른 에러 경로는 모두 그대로 유지. 본 PR 은 **목록 조회 배너 한 블록** 에만 영향.
- 백엔드 (스키마·응답 포맷·HTTP 코드) 무변경. 503 → 500 매핑 정상화는 여전히 별건 후속 과제.

---

## 4. 정적 검증

- `pnpm tsc --noEmit` (frontend/): 본 파일에는 신규 오류 없음. 리포 전체에는 선행 PR 에서 유입된 `AdminUserDetailPage.tsx:279` Date 생성자 관련 pre-existing 에러가 남아 있으나 본 PR 범위 외.
- `pnpm eslint frontend/src/features/admin/golden-sets/AdminGoldenSetsPage.tsx`: **경고 0건**.
- 사용처: `classifyListError` 는 같은 파일 내 `AdminGoldenSetsPage` 컴포넌트에서만 호출. 외부 export 없음.

---

## 5. 런타임 검증 (Chrome MCP, localhost:3050, `admin@mimir.local` / SUPER_ADMIN 세션)

`window.fetch` 를 모드 스위치(`window.__mimirMockMode`) 로 재작성하여 `/api/v1/golden-sets` 목록 호출에만 특정 상태를 주입하고, 사이드바 SPA 내비게이션(`대시보드 ↔ 골든셋 관리`) 으로 React Query 가 새 fetch 를 발생시키도록 유도. 상세/생성/아이템/버전/import/export 경로는 모크에서 제외하여 나머지 기능은 건드리지 않음.

| 시나리오 | 주입 | 관측한 배너 | 재시도 버튼 | S2 ⑥ 언급 | 결과 |
| --- | --- | --- | :---: | :---: | :---: |
| S1 | 네트워크 실패 (`TypeError: Failed to fetch`) | **서버에 연결하지 못했습니다** / 네트워크 상태 또는 서버 가용성을 확인해 주세요 | ✓ "다시 시도" | ✗ 없음 | ✅ |
| S2 | 403 `{ detail: "Forbidden by Scope Profile" }` | **이 목록을 볼 권한이 없습니다** / 이 작업을 수행할 권한이 없습니다 / **Scope Profile 바인딩 또는 역할을 확인해 주세요. (S2 ⑥)** | ✗ 없음 | ✓ 있음 | ✅ |
| S3 | 503 `{ detail: "Backend temporarily unavailable" }` | **서버에서 일시적 오류가 발생했습니다** / 잠시 후 다시 시도해 주세요. 문제가 지속되면 관리자에게 문의해 주세요. | ✓ "다시 시도" | ✗ 없음 | ✅ |

- S1~S3 에서 `"S2 ⑥"` 문자열이 의도대로 오직 S2(403) 에만 등장함을 DOM 관찰로 재확인.
- S1, S3 의 "다시 시도" 버튼 클릭 시 `listQuery.refetch()` 가 발화 → 모크가 계속 걸려 있으면 같은 에러, 모크 해제 후에는 정상 200 응답으로 복귀.
- 모크 해제 후 페이지 재진입에서 "S2-5 회귀 테스트 골든셋" 행과 정상 목록이 복원됨을 확인(regression 확인).

---

## 6. 접근성 / 시각

- `role="alert"` 는 기존과 동일 유지 → 스크린리더가 title + body + hint 를 하나의 alert 로 읽음.
- title 은 `text-sm font-bold`, body/hint 는 `text-xs` 로 시각 계층이 명확. 빨강 톤(`bg-red-50` / `border-red-200` / `text-red-700/800`) 는 기존 디자인 언어와 일치하며 blue 팔레트 교체(#14) 취지와도 충돌하지 않음 (에러는 여전히 빨강 톤 사용).
- "다시 시도" 버튼은 `min-h-[32px]` 로 P0-4 키보드/터치 접근성 기준(32px 이상) 을 충족. `focus-visible:ring-2 focus-visible:ring-red-500` 가시화.
- 대비(contrast): `text-red-800 on bg-red-50` ≈ 11:1 (AA 준수).

---

## 7. 보안 영향 (추가 공격 표면 평가)

| 항목 | 영향 |
| --- | --- |
| 인증/인가 경로 | 변경 없음 — 서버 응답을 UI 가 어떻게 분류하느냐만 변경 |
| XSS | 모든 표시 문자열은 JSX 표현식 바인딩 → React 자동 이스케이프. `dangerouslySetInnerHTML` 미사용 |
| 정보 누출 | body 에 서버 메시지(`detail`) 를 그대로 노출. 기존 동작과 동일하며, 백엔드가 민감 정보를 `detail` 에 내리지 않는다는 전제를 유지 |
| CSRF/PII/의존성 | 신규 의존성 0건, 네트워크 호출 0건 추가, 폼 추가 0건 |
| 감사 로그 | 로깅 로직 미변경 (서버 레이어) |

OWASP Top 10 (2021) 전 카테고리에서 **영향 없음**. 신규 취약점 도입 0건. 오히려 5xx/네트워크 실패를 잘못 S2 ⑥ 로 귀인하던 오보(OWASP A04 Insecure Design 관점) 를 해소.

---

## 8. 잔존 후속

- 503 → 500 매핑 정상화: 백엔드 `unhandled_exception_handler` + slowapi 상호작용 조사 (별건, `project_s2_closure_gaps.md` 트래킹).
- "다시 시도" 버튼 클릭 후 실패가 연속되면 지수 백오프 안내를 추가할지 여부 — 현재 사용 패턴(저빈도, 운영자 수동) 에선 불필요. 운영 데이터가 쌓이면 재평가.
- 테스트 row "S2-5 회귀 테스트 골든셋" 정리(선행 리뷰의 잔존 항목).
- `classifyListError` 에 대한 단위테스트 추가 — `S2 종결 커버리지 갭` 해소의 일부로 벤더(예: `@testing-library` + `vitest`) 표준 세팅 정비 후 일괄 추가 예정.

---

## 9. 결론

- 단일 목록 에러 배너에서 **"S2 ⑥" 오귀인이 제거**되었고, 운영자는 이제 에러 유형에 맞는 행동(재로그인 / 권한 점검 / 재시도 / 네트워크 확인 / 배포 확인) 을 배너만 보고도 결정할 수 있다.
- 정적 검증(tsc, eslint) + 런타임 검증(3 시나리오) 모두 통과. 신규 취약점 0건.
- #31 은 본 보고서로 **해결(resolved)** 상태로 전환.
