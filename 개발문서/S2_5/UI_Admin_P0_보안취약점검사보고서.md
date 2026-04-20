# Admin UI P0 보안 취약점 검사 보고서

- 작성일: 2026-04-20
- 대상: 본 세션에서 변경된 파일 10종 — `DataTable.tsx`, `app/admin/jobs/page.tsx`, `app/admin/jobs/JobsTabs.tsx`, 6개 bespoke `<tr>` 페이지, 그리고 직전 세션의 `P0-1/P0-3` 팔레트 스왑 누적
- 목적: 본 UI 변경이 인증/인가, XSS, CSRF, PII, ACL, scope, 의존성 취약점, CSP/SSR 보안에 미친 영향을 평가

---

## 1. 변경 요약 (공격 표면 관점)

| 변경 | 공격 표면 변화 |
| --- | --- |
| DataTable `<tr>` 에 `role="button"`, `tabIndex`, `onKeyDown` 추가 | 없음 — 클라이언트 이벤트 핸들러에 외부 문자열 주입 경로 없음. `onRowClick` 은 페이지가 제공한 클로저 |
| 페이지별 bespoke `<tr>` 에 동일 패턴 추가 | 없음 — 동일 |
| 색상 팔레트(red → blue) 치환 | 없음 — 시각 토큰만 |
| `/admin/jobs/page.tsx` 서버 컴포넌트 분리 + `JobsTabs.tsx` 신설 | 오히려 메타데이터 누출/ SSR 일관성 회복에 기여 |

---

## 2. OWASP Top 10 (2021) 매핑

### A01 Broken Access Control

- 본 세션 변경은 **클라이언트 렌더링 레이어 한정**. ACL/Scope Profile 결정 로직은 변경 없음.
- `DataTable` 에 scope 문자열 하드코딩 없음 — S2 ⑤ 부합.
- `onRowClick` 이 호출하는 페이지별 setSelected(...) 는 UI 상태 전환일 뿐, 권한 상승을 유발하지 않음. 실제 API 호출/mutation 은 상세 드로어 내 별도 버튼이 수행.
- 결론: **영향 없음**.

### A02 Cryptographic Failures
- 키/토큰/자격증명 값 로깅 없음. 변경 없음.

### A03 Injection (XSS 포함)

- 신규 코드는 모두 **JSX 표현식 바인딩** (`aria-label={`...${p.name}...`}`) — React 의 기본 이스케이프로 보호됨.
- `dangerouslySetInnerHTML` 도입 없음.
- 기존 `DOMPurify` 사용처 미변경.
- `document.title` 은 Next 16 의 `metadata` 로 정적 문자열만 주입 — 사용자 입력 경로 없음.
- 결론: **영향 없음**.

### A04 Insecure Design

- 상호작용 row 의 키보드 핸들러가 `Enter` / `Space` 에만 반응하고, Proposals 에서는 중첩 `INPUT`/`BUTTON` 은 행 열기 액션을 스킵 — 중복 동작으로 인한 우발적 상태 변경 방지.
- 결론: **영향 없음(오히려 강화)**.

### A05 Security Misconfiguration

- CSP/CORS/서버 헤더 변경 없음.
- Next metadata 복구로 per-route title 이 정상화 → 정보 누출 감소는 아니나, `<title>` 일관성 회복.
- 결론: **영향 없음**.

### A06 Vulnerable & Outdated Components

- 신규 의존성 추가 없음. `package.json` 변경 0건.
- React 19, Next 16.2.4 유지. 둘 다 최신 LTS 계열.
- 결론: **영향 없음**.

### A07 Identification & Authentication Failures

- `/admin/*` 접근 제어(`ORG_ADMIN` AuthGuard) 는 `app/admin/layout.tsx` → `AdminLayout` 에서 처리. 본 변경은 그 내부 렌더링만 수정.
- 결론: **영향 없음**.

### A08 Software & Data Integrity Failures
- SRI/서명된 번들 관련 변경 없음. 결론: 영향 없음.

### A09 Security Logging & Monitoring Failures
- 감사 로그(`actor_type` 필수 기록, S2 ⑥)는 서버 레이어. UI 변경은 이에 영향 없음.

### A10 Server-Side Request Forgery
- 서버측 fetch 변경 없음. 결론: 영향 없음.

---

## 3. 보조 체크리스트

### 3.1 Deprecated API 사용 (CLAUDE.md #1)

| API | 상태 |
| --- | --- |
| `document.title` 수동 조작 | 사용 안 함 (Next 16 `metadata` 대체) |
| 인라인 `onclick` 속성 | 사용 안 함 |
| React `findDOMNode` 등 legacy | 사용 안 함 |
| Next `getServerSideProps` | 해당 없음 (App Router) |

### 3.2 Critical CVE 의존성 (CLAUDE.md #2)

- `package.json` 변경 없음 → 공격 표면 추가 없음.
- 참고: 기존 의존성 중 `dompurify@^3.4.0`, `zod@^4.3.6`, `react-hook-form@^7.72.1`, `next@^16.2.4` 모두 최신 안정. (변경이 없으므로 본 PR 에서 재점검 불필요.)

### 3.3 보안 일반(CLAUDE.md #3)

| 항목 | 검토 |
| --- | --- |
| PII 저장/노출 증가 | 없음 — `aria-label` 에 사용자 표시명을 포함하나, 이미 화면에 렌더되던 정보 |
| 인증 우회 경로 | 없음 |
| CSRF 토큰 다루는 폼 변경 | 없음 |
| 외부 리소스 로드 | 추가 없음 |
| localStorage/sessionStorage 신규 사용 | 없음 |
| 외부 iframe/embed | 없음 |

### 3.4 폐쇄망(S2 ⑦) 영향

- Blue 팔레트는 Tailwind 기본 토큰(이미 번들에 포함)으로 렌더 — 외부 네트워크 의존 없음.
- Next metadata 경로는 런타임 외부 호출 없음.
- 결론: 폐쇄망에서도 동일 동작.

---

## 4. 접근성 변경이 간접적으로 만드는 보안 영향

새 `role="button"` + `tabIndex={0}` 은 자동화 도구(스크린리더/키보드 스크립트)의 도달 범위를 넓힌다. 이 자체는 정상이며 **권한이 없는 사용자가 열람할 수 없는 데이터에 접근하는 경로가 새로 생기지 않는다** — 상세 드로어는 여전히 상위 API(`/api/admin/...`) 로부터 권한 체크된 데이터만 로드하기 때문이다. 즉, "접근 가능 범위 확장" 은 UI 발견 가능성(discoverability)만 늘리고, 실제 권한 결정은 서버 레이어에서 동일하게 유지된다.

---

## 5. 잔존 고위험 항목 (참고)

본 보고서 범위 밖이지만, S2 종결 선언 후에도 잔존하는 P0(오토메모리 `project_s2_closure_gaps.md` 참조) 중 UI 와 간접적으로 연관되는 항목:

- `compose` 기본 시크릿 잔존 → UI 변경과 무관. 서버 환경 구성으로 분리 조치 필요.
- AI 품질 실측 공백 → UI 범위 밖.
- 테스트 커버리지 35% → 본 UI 변경은 회귀/접근성 단위테스트가 없음. 추후 `DataTable` 키보드 이벤트 단위테스트 추가 권고.

---

## 6. 결론

- 본 세션의 UI P0 변경은 **신규 취약점 도입 0건**.
- OWASP Top 10 전 카테고리에서 "영향 없음" 또는 "간접적 개선".
- 의존성 추가 없음, 네트워크 표면 확장 없음, 인증/인가 경로 미변경.
- 권고 후속: (1) `DataTable` 의 키보드 이벤트 회귀 단위테스트 추가 (2) `DataTable` 사용 페이지에 `ariaLabel` prop 전파하여 스크린리더 경험 개선.
