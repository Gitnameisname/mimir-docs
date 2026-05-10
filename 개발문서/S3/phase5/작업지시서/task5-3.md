# task5-3 — 멘션 typeahead 자동완성 + 사용자 검색 API (FG 5-3)

**작성일**: 2026-05-10
**Phase / FG**: S3 Phase 5 / FG 5-3
**상태**: 착수 대기 (FG 5-1 ADR 완료 권장 — mark 우선순위 결정 후, 단 typeahead 는 mark 가 아니라 popup 이라 schema 충돌 무관)
**담당**: 설계 Claude / 구현 Claude (또는 Codex) / 리뷰 Codex (extended — 신규 외부 API + ACL)
**예상 소요**: 4~5일 (Pre-flight + 신규 API + TipTap suggestion + ACL 회귀)
**선행 산출물**:
- Phase 4 FG 4-1 의 envelope 표준 (있으면 mention 결과 응답에 적용 — 없으면 본 FG 는 envelope 없이 진행하고 후속 patch)
- FG 5-1 ADR (mark 충돌 무관이지만 markNames 단일 정본 활용)
**후행**: task5-4 (사이드바 재구조화) 와 독립

---

## 1. 작업 목적

Phase 3 FG 3-3 가 도입한 mention (`@username`) 은 **수동 입력만** 가능하고, 사용자 검색 popup 이 미구현이었다 (FG 3-3 종결보고서 §"잔존 작업"). 본 FG 가 다음을 닫는다:

1. **`@` 입력 시 floating typeahead popup** — TipTap Suggestion API 활용. debounce 150ms.
2. **viewer-scoped 사용자 검색 API 신설** — `GET /api/v1/users?q=&limit=20`. **viewer 의 organization scope 안 사용자만 반환** (R-A4).
3. **선택 → mention 삽입** — 본문에 `@display_name` 텍스트 + 잠재적으로 별 mark 또는 단순 텍스트 (Pre-flight 결정).
4. **키보드 네비** — 화살표 위/아래로 후보 이동, Enter 로 선택, Esc 로 닫기.

**R-A4 절대 규칙**: 사용자 검색 결과가 **다른 organization 의 user.display_name 을 절대 노출하지 않음**. ACL 누설은 본 FG 의 가장 큰 위험 — 검수보고서 + 보안보고서 모두 R-A4 회귀 결과 첨부 의무.

---

## 2. 작업 범위

### 2.1 포함

#### 2.1.0 Pre-flight 실측 — `산출물/FG5-3_Pre-flight.md`

본 FG 진입 전 다음을 실측해 보고서로 남긴다 (Phase 5 §6.2 보류 결정 — 신설 vs 기존 admin users API 재사용).

##### a. 기존 admin users API 분석

- **엔드포인트**: `GET /admin/users?search=...` (`backend/app/api/v1/admin.py:291–362`)
- **ACL**: `require_admin_access()` → `authorization_service.authorize(... action="admin.read" ...)` — ORG_ADMIN + SUPER_ADMIN 만
- **응답 스키마**: 확인 후 보고서에 명시 (`AdminUserSummary` 또는 동등)
- **결론**: viewer (AUTHOR / REVIEWER) 가 typeahead 호출 시 admin.read 권한 부재 → **재사용 불가**. 신설 필요.

##### b. 사용자 모델 / 스코프 분석

- `users` 테이블의 `organization_id` 컬럼 (또는 동등) 확인
- `ActorContext.tenant_id` 또는 `organization_id` 가 viewer scope 의 정본인지 확인 (`backend/app/api/auth/models.py:49–80`)
- 사용자 명 / 이메일 / 표시명 컬럼 — `display_name` / `email` 중 어느 것이 typeahead 검색 대상인지 결정

##### c. mention 영속화 결정

Phase 3 의 mention 은 본문에 `@username` 텍스트로 저장되고, 별도 `annotation_mentions` 테이블에 `mentioned_user_id` 가 들어간다 (FG 3-3). typeahead 가 추가하는 mention 은:
- (A) **plain text** — 본문에 `@display_name` 만 삽입. annotation 레벨에서만 user_id 결합 (현행 FG 3-3 패턴 유지)
- (B) **MentionMark** — TipTap mark 로 `data-user-id` 부여. 직렬화 안정성 + click 시 user 프로필 라우팅
- (C) **Suggestion 만, 텍스트는 plain** — typeahead 는 popup 만 제공, 본문은 plain `@username` (B 보다 가벼움)

본 FG default 는 (C). (B) 채택 시 task5-1 ADR 의 mark 우선순위 / 결합 표를 확장해야 하므로 **운영자 결정 필요**. Pre-flight 보고서에 (A)/(B)/(C) 비교 + 권장안 명시.

##### d. envelope 표준 적용 가능성

Phase 4 FG 4-1 의 `MCPReadEnvelope` 가 본 사용자 검색 API 응답에도 적용 가능한지 확인. mention typeahead 는 MCP 표면이 아니지만, prompt injection 방어 (display_name 에 `ignore previous` 같은 페이로드가 들어 있을 가능성) 차원에서 envelope 의 `detected_risks` 매핑 가능. Pre-flight 결정.

#### 2.1.1 사용자 검색 API 신설

`backend/app/api/v1/users.py` (신규) 또는 기존 적합 라우터에 추가:

##### a. 엔드포인트 명세

| 메서드 | 경로 | 인증 | 파라미터 |
|-------|------|-----|---------|
| GET | `/api/v1/users?q=<prefix>&limit=20` | 인증된 viewer (any role ≥ VIEWER) | `q`: 1~64자 prefix string, `limit`: 1~50 (default 20) |

##### b. ACL — viewer organization scope 강제

```python
@router.get("/users")
async def search_users(
    q: str = Query(..., min_length=1, max_length=64),
    limit: int = Query(default=20, ge=1, le=50),
    actor: ActorContext = Depends(require_authenticated),
) -> UserSearchResponse:
    # 1. viewer 의 organization_id 확정 — 외부에서 주입 금지
    org_id = actor.organization_id
    if not org_id:
        raise HTTPException(403, "조직 컨텍스트가 없는 사용자는 사용자 검색을 사용할 수 없습니다.")

    # 2. q 정규화 — SQL injection / wildcard 방어
    q_normalized = _normalize_search_prefix(q)

    # 3. repository 호출 — organization_id 가 keyword-only required
    users = await users_repository.search_by_display_name(
        organization_id=org_id,  # keyword-only required (S2 ⑥ Scope 하드코딩 금지)
        query=q_normalized,
        limit=limit,
    )

    # 4. 응답 — display_name + user_id (mention 영속화용) 만. email / role / status 노출 금지
    return UserSearchResponse(
        items=[UserSummary(user_id=u.id, display_name=u.display_name) for u in users],
        items_total=len(users),
        items_truncated=len(users) >= limit,
    )
```

##### c. Repository 신설

`backend/app/repositories/users_repository.py` 의 `search_by_display_name(*, organization_id, query, limit)`:
- WHERE `organization_id = :org_id` AND `display_name ILIKE :prefix || '%'` AND `status = 'active'`
- ORDER BY `display_name` ASC
- LIMIT `:limit`
- **`organization_id` 는 keyword-only required** (S2 ⑥ Scope 하드코딩 금지 — Phase 2 FG 2-2 패턴)

##### d. Rate Limiting

- viewer 별 `60req/min` (typeahead 가 빈번 호출 — debounce 150ms 와 결합)
- 공통 SlowAPI 또는 동등 미들웨어 사용. 기존 패턴 (`mcp_router.py` 의 `_MCP_TOOL_LIMIT` 등) 확인 후 결정

##### e. Audit / Observability

- 본 엔드포인트가 PII (사용자 prefix) 를 받음. audit log 에 prefix 자체는 저장 금지 (제24조 — 민감정보 장기 메모리 금지). `q_length` 와 `result_count` 만 기록.
- `trace_id` 부여 (제13·48조)

#### 2.1.2 Frontend — TipTap Suggestion 통합

`frontend/src/features/editor/tiptap/extensions/MentionSuggestion.ts` (신규):
- TipTap `@tiptap/suggestion` 패키지 활용 (Pre-flight 에서 폐쇄망 / 라이선스 확인)
- char: `@`
- pluginKey: `mentionSuggestion`
- command: 선택 시 `@<display_name> ` 삽입 (Pre-flight (C) default)
- items: debounce 150ms 후 `GET /api/v1/users?q=<prefix>` 호출

##### a. Suggestion popup 컴포넌트

`frontend/src/features/editor/MentionPopup.tsx` (신규):
- floating UI — `@floating-ui/react` 또는 동등 (이미 의존성에 있는 라이브러리 우선)
- 후보 목록 — display_name 만 표시 (email 노출 금지)
- 키보드 네비 — ↑/↓ / Enter / Esc / Tab
- 마우스 hover / click — 동일 동작
- 빈 결과 — "사용자를 찾을 수 없습니다" 메시지

##### b. ACL 누설 방어 — frontend 측

- API 응답이 user_id + display_name 만 포함하므로 frontend 가 다른 정보를 추측해 표시할 위험 없음
- 단, `organization_id` 를 query 로 강제 전송 금지 (서버가 viewer 의 ActorContext 에서 결정 — 클라이언트 신뢰 금지)

##### c. 접근성

- `role="listbox"` + `aria-activedescendant`
- 각 후보 `role="option"`
- 화면 리더 지원 — "사용자 검색 결과 5개" 같은 live region

#### 2.1.3 mention 영속화 — Phase 3 FG 3-3 패턴 유지

(Pre-flight (C) default 시):
- typeahead 선택 → 본문 plain text `@display_name` 삽입
- annotation 작성 시 — annotation API 가 `mentioned_user_ids: [user_id, ...]` 받음 (FG 3-3 그대로)
- typeahead 선택 결과를 React state 로 hold → annotation submit 시 함께 전송

> Pre-flight (B) MentionMark 채택 시 본 §2.1.3 확장 — task5-1 ADR 추가 결정 + AnnotationMark 와 동등한 mark 통합 + 별 attribute (`data-user-id`).

#### 2.1.4 회귀 / 보안 테스트 (≥ 12 시나리오)

##### a. backend pytest (`test_users_search.py`)

| ID | 시나리오 |
|----|---------|
| B1 | viewer (AUTHOR) — 자기 organization 사용자 검색 → 결과 OK |
| B2 | viewer — 다른 organization 사용자 검색 → 결과 0건 (R-A4 핵심) |
| B3 | viewer — `q=""` → 400 |
| B4 | viewer — `q` 64자 초과 → 400 |
| B5 | viewer — `q="'; DROP TABLE users;--"` → SQL injection 차단 + 결과 0건 또는 정상 escape |
| B6 | viewer — 다른 organization id 를 query 로 주입 시도 → 무시되고 viewer scope 만 사용 |
| B7 | unauthenticated → 401 |
| B8 | viewer — rate limit 초과 → 429 |
| B9 | viewer — `status=inactive` 사용자 제외 |
| B10 | viewer — display_name prefix 매칭 (case-insensitive) |

##### b. node:test (`MentionSuggestion.test.ts`)

| ID | 시나리오 |
|----|---------|
| F1 | `@` 입력 → popup 노출 |
| F2 | popup 후보 수가 응답 수와 일치 |
| F3 | Enter → 본문에 `@display_name ` 삽입 + popup 닫힘 |
| F4 | Esc → popup 닫힘 + 본문 미변경 |
| F5 | debounce 150ms — 빠른 입력 시 API 1회만 호출 |
| F6 | `mention 후 추가 텍스트 입력` → mark/text 결합 안정 |

##### c. R-A4 통합 회귀 (`test_typeahead_acl.py`)

`backend/tests/integration/test_typeahead_acl.py` (신규):
- 2 organization 시나리오 — org-A viewer 가 org-B user 의 display_name 을 어떤 prefix 로도 검색 못함 (≥ 5 prefix 시도)
- 동일 display_name 이 두 organization 에 존재 → org-A viewer 는 org-A 결과만

#### 2.1.5 함수도서관 갱신

- `docs/함수도서관/backend.md` — `users_repository.search_by_display_name` + `/api/v1/users` 엔드포인트
- `docs/함수도서관/frontend.md` — `MentionSuggestion` extension + `MentionPopup`
- 둘 다 같은 PR 에 commit (제12조)

### 2.2 제외 (이월)

- **Group / role mention** (`@team`, `@reviewers`) — 별 라운드
- **mention 알림 (notifications)** — Phase 3 FG 3-3 의 `annotation_mentions` 가 이미 처리. 본 FG 는 입력 UX 만
- **user search 의 fuzzy matching / score** — 본 FG 는 prefix 매칭만. ranking 은 별 라운드
- **`mimir://users/<id>` URI** — Phase 4 FG 4-1 결정. mention click 시 라우팅은 (B) MentionMark 채택 시에만 적용
- **한국어 username 정책 / 멘션 정규식 확장** — FG 5-5 (조건부, 본 라운드 제외 — Phase 5 §2.2)

### 2.3 하드코딩 금지 재확인

- typeahead 의 `organization_id` 는 항상 ActorContext 에서 추출. query / body 주입 금지
- `users_repository.search_by_display_name` 의 `organization_id` keyword-only required — 위치 인자 금지 (S2 ⑥)
- rate limit 임계값 / debounce 시간은 환경 변수 또는 설정 모듈 단일 정본

---

## 3. 선행 조건

- Phase 5 §1.2 R-A4 운영자 승인
- Pre-flight (§2.1.0) 완료 + 운영자 합의 — (A)/(B)/(C) mention 영속화 결정
- `users` 테이블의 `organization_id` 컬럼 / `ActorContext.organization_id` 또는 `tenant_id` 의 정본 확인
- 기존 admin users API 의 ACL 패턴 분석 — 동일 패턴 재사용 가능 여부

---

## 4. 구현 단계

### Step 1 — Pre-flight 실측 (§2.1.0)

1. admin users API 분석 + 사용자 모델 / 스코프 분석
2. mention 영속화 (A)/(B)/(C) 비교 보고서
3. envelope 표준 적용 가능성 검토
4. 운영자 합의 — mention 영속화 결정 (default (C))

### Step 2 — backend 사용자 검색 API (§2.1.1)

1. `users_repository.search_by_display_name` 신설
2. `/api/v1/users` 라우터 신설
3. rate limit 적용
4. audit / trace_id 부여
5. 단위 테스트 — repository ≥ 5 + router ≥ 5

### Step 3 — frontend MentionSuggestion (§2.1.2)

1. `@tiptap/suggestion` 의존성 확인 (없으면 추가 — package.json + 폐쇄망 미러 체크)
2. `MentionSuggestion.ts` extension 신설
3. `MentionPopup.tsx` 컴포넌트 + floating UI
4. `DocumentTipTapEditor` extensions 배열에 등록
5. 단위 테스트 ≥ 6 (§2.1.4 b)

### Step 4 — annotation submit 통합 (§2.1.3)

1. typeahead 선택된 user_id 를 annotation submit body 의 `mentioned_user_ids` 에 포함
2. AnnotationsPanel 의 annotation 작성 path 확인 — typeahead 결과를 흡수하는 경로 명시
3. 회귀 테스트 — typeahead 선택 → annotation 생성 → mention 알림 (Phase 3 FG 3-3 회귀 녹색)

### Step 5 — R-A4 회귀 (§2.1.4 c)

1. `test_typeahead_acl.py` 신설 — 2 organization 매트릭스
2. 동일 display_name + 다른 organization 시나리오
3. organization_id query 주입 차단 회귀

### Step 6 — 함수도서관 갱신 (§2.1.5)

1. backend.md / frontend.md 동시 commit

### Step 7 — 검수 / 보안 보고서

- `FG5-3_검수보고서.md` — R-A4 준수 확인 + 회귀 결과 첨부
- `FG5-3_보안취약점검사보고서.md`:
  - SQL injection (B5 시나리오 결과)
  - rate limiting 회귀
  - prompt injection (display_name 페이로드)
  - PII audit log (q prefix 미저장 확인)
  - organization_id 주입 차단 회귀
  - timing attack — q 길이별 응답 시간 (다른 org 사용자 존재 여부 누설)
- UI 디자인 리뷰 ≥ 1회 (Phase 5 §4 산출물 규약) — popup 의 4 viewport 동작

---

## 5. API 계약 변경 요약

| 메서드 | 경로 | 변경 |
|-------|------|------|
| GET | `/api/v1/users` | **신설** — `q` prefix 검색 + viewer organization scope. 응답: `{ items: [{ user_id, display_name }], items_total, items_truncated }` |

응답 envelope (Phase 4 FG 4-1) 는 본 FG 에서 적용 안 함 (MCP 표면 외) — Pre-flight 결정 시 적용 가능.

---

## 6. 데이터 모델 주의사항

- DB 스키마 변경 없음. 기존 `users` 테이블 read-only 사용.
- (B) MentionMark 채택 시 ProseMirror JSON 에 mark 추가됨 — 마이그레이션 불필요 (기존 doc 에 부재 = plain text mention 유지).

---

## 7. 성공 기준

- [ ] Pre-flight 보고서 제출 + mention 영속화 (A/B/C) 결정 + 운영자 합의
- [ ] `/api/v1/users` 엔드포인트 신설 + 단위 테스트 ≥ 10 녹색
- [ ] viewer organization scope 강제 (`organization_id` keyword-only required)
- [ ] **R-A4 회귀** — 2 organization 매트릭스 ≥ 5 prefix 시나리오 녹색
- [ ] MentionSuggestion extension + MentionPopup 컴포넌트 + 단위 테스트 ≥ 6 녹색
- [ ] debounce 150ms 동작 회귀
- [ ] 키보드 네비 (↑/↓/Enter/Esc) 회귀
- [ ] annotation submit 통합 — typeahead 선택 → mention 알림 (Phase 3 FG 3-3 회귀 녹색)
- [ ] rate limit 회귀 (429 반환)
- [ ] SQL injection / prompt injection 회귀
- [ ] audit log 에 q prefix 미저장 확인
- [ ] node:test 신규 ≥ 6 / 베이스라인 유지
- [ ] pytest 신규 ≥ 15 / 베이스라인 유지
- [ ] tsc 0 error
- [ ] `docs/함수도서관/backend.md` + `frontend.md` 동시 갱신
- [ ] `FG5-3_검수보고서.md` + `FG5-3_보안취약점검사보고서.md` + UI 디자인 리뷰 1회 통과

---

## 8. 리스크

| # | 리스크 | 대응 |
|---|-------|-----|
| R-01 | viewer 가 `organization_id` query 로 다른 org 의 사용자 노출 | repository 의 keyword-only `organization_id` + ActorContext 에서만 주입 + B6 회귀 |
| R-02 | display_name 충돌로 잘못된 user_id 영속화 | typeahead 결과는 `(user_id, display_name)` 페어 — 본문 텍스트가 아닌 user_id 가 mention 영속화 정본 |
| R-03 | rate limit 부재 시 user enumeration 공격 | viewer 별 60req/min + 1자 prefix 도 rate limit 적용 + B8 회귀 |
| R-04 | timing attack — 다른 org 사용자 존재 여부 누설 | 응답 시간 정규화 (의미 있는 수준에서) + 검수보고서에 분석 결과 |
| R-05 | display_name 에 prompt injection 페이로드 | typeahead 응답에 envelope 적용 (Pre-flight (d) 결정 시) 또는 frontend 측 sanitize. mention 본문 삽입 시 plain text 만 |
| R-06 | (B) MentionMark 채택 시 mark 통합 ADR (task5-1) 재합의 필요 | Pre-flight 에서 default (C) 채택 권장. (B) 선택 시 task5-1 ADR §a 표 확장 + 별 reviewer 합의 |
| R-07 | `@tiptap/suggestion` 의존성이 폐쇄망 미러에 부재 | Pre-flight 에서 dependencies 확인 + 폐쇄망 미러 등록 절차 운영자 합의 |
| R-08 | mention 후 본문 mark/text 결합이 task5-1 round-trip 회귀 깸 | F6 시나리오 + task5-1 round-trip S5 회귀 녹색 의무 |
| R-09 | audit log 가 q prefix 를 저장 → PII 누설 | log schema 명시적으로 prefix 제외 + 정적 분석 1회 (검수보고서) |

---

## 9. 참조

- `docs/개발문서/S3/phase5/Phase 5 개발계획서.md` §1.2 (R-A4), §2.1 (FG 5-3), §6.2 (보류 결정 — 신설 vs 재사용)
- `docs/개발문서/S3/phase3/산출물/FG3-3_종결보고서.md` (mention typeahead 잔존 작업)
- `docs/개발문서/S3/phase4/작업지시서/task4-1.md` (envelope 표준 — 적용 가능성)
- `backend/app/api/v1/admin.py:291–362` (admin users API — 패턴 참고)
- `backend/app/api/auth/models.py:49–80` (ActorContext)
- `backend/app/api/auth/authorization.py` (역할 계층)
- `frontend/src/features/editor/tiptap/DocumentTipTapEditor.tsx`
- `CONSTITUTION.md` 제5·11·12·17·18·19·20·24·25조

---

*작성: 2026-05-10 | FG 5-3 — 멘션 typeahead + 사용자 검색 API + R-A4 ACL 누설 차단*
