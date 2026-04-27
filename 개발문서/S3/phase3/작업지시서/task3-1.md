# task3-1 — Contributors 패널

**작성일**: 2026-04-27
**Phase / FG**: S3 Phase 3 / FG 3-1
**상태**: 착수 대기 (Phase 2 게이트 통과 후)
**담당**: 백엔드 + 프런트
**예상 소요**: 2~3일
**선행 산출물**:
- `docs/개발문서/S3/phase3/Phase 3 개발계획서.md`
- Phase 2 종결 보고 (FG 2-1 ~ 2-6 게이트 통과)
- S2 Phase 3 `audit_events.actor_type` 마이그레이션 적용 상태 (`backend/app/db/connection.py` `_AUDIT_EVENTS_ACTOR_TYPE_MIGRATION_DDL`)

---

## 1. 작업 목적

문서의 **작성자 / 편집자 / 승인자 / 최근 열람자** 4 카테고리를 한 사이드 패널에 모아, "누가 이 문서에 관여했는가" 를 한눈에 보게 한다. 정보 출처는 기존 자료의 **재조합**이고, 신규 추적 로그는 도입하지 않는다. 에이전트 액터(S2 ⑤ 일관성)도 사람과 동일하게 표시하되 `actor_type` 뱃지로 구분한다.

> 열람자 노출의 on/off 는 **FG 3-2 의 책임**이다. 본 FG 는 API 가 `include_viewers` 파라미터를 받아 응답에서 viewer 섹션을 포함/생략하는 **메커니즘**까지만 담당하고, **정책 게이트**는 FG 3-2 가 Scope Profile `expose_viewers` 를 읽어 본 API 호출의 `include_viewers` 를 강제로 false 로 덮어씌우는 형태로 결합한다.

---

## 2. 작업 범위

### 2.1 포함

1. **데이터 출처 매핑 (스키마 변경 없음)**
   - 작성자: `documents.created_by` (단일 user_id 문자열, 없으면 system)
   - 편집자: `audit_events` 중 `event_type ∈ {'document.draft.saved','document.published','document.metadata.updated','annotation.created'(FG 3-3 이후)}` 의 `actor_user_id` distinct
   - 승인자: `workflow_history` 중 `to_status='published'` 또는 `action='approve'` 의 `actor_id` distinct
   - 최근 열람자: `audit_events` 중 `event_type='document.viewed'` 의 `actor_user_id` distinct + `MAX(occurred_at)`
   - 모든 카테고리는 `actor_type` 컬럼(S2 Phase 3 마이그레이션) 함께 SELECT — `'user' | 'agent' | 'system'`

2. **이벤트 타입 정의 (재확인 / 신규 등록)**
   - `audit_events` 테이블은 `event_type` 에 enum 제약이 없으므로(VARCHAR 100) **레지스트리 문서**만 갱신:
     - `backend/app/audit/event_types.md` (없으면 신규) 에 본 FG 가 의존하는 4 종 명시
     - `document.viewed` 이벤트가 기존 `document_view_service` 에서 emit 되는지 점검 — 누락이면 본 FG 에서 emit 추가 (단일 곳, 중복 emit 금지)
   - 새 이벤트 emit 은 `backend/app/audit/emitter.py` 의 `emit_for_actor()` 패턴 그대로

3. **백엔드 — 서비스**
   - `backend/app/services/contributors_service.py` 신설:
     ```python
     def get_contributors(
         conn,
         document_id: UUID,
         viewer_actor: ActorContext,
         since: datetime | None = None,
         include_viewers: bool = False,
         limit_per_section: int = 50,
     ) -> ContributorsBundle:
         """4 카테고리 집계. since 가 None 이면 viewers 는 최근 30일 default."""
     ```
   - 반환: `ContributorsBundle(creator, editors[], approvers[], viewers[])`. 각 항목 = `(user_id, display_name, actor_type, last_activity_at, role_badge?)`
   - 사용자 표시 정보는 `users_repository.get_many(user_ids)` 1 회 batch 조회로 N+1 방지
   - **ACL**: viewer 가 해당 document 를 못 본다(scope filter 통과 실패) → 404. 본 API 자체가 새 권한 게이트 추가 안 함. FG 2-0 의 `documents.scope_profile_id` 필터를 호출 직전 통과 확인

4. **백엔드 — 라우터**
   - `backend/app/api/v1/contributors.py` 신설:
     ```
     GET /api/v1/documents/{document_id}/contributors
       ?since=ISO8601
       &include_viewers=bool (기본 false)
       &limit_per_section=int (기본 50, 상한 200)
     ```
   - 응답 캐시: `Cache-Control: private, max-age=15` (열람자 변동이 빠르므로 짧게)
   - rate-limit: per-user 60 req/min
   - `router.py` 에 `include_router(contributors.router, prefix="/documents", tags=["contributors"])`

5. **프런트엔드 — 사이드 패널**
   - `frontend/src/features/documents/ContributorsPanel.tsx` — 문서 상세 우측 사이드
   - 섹션 4 개: 작성자(1), 편집자(N), 승인자(N), 최근 열람자(N) — viewers 섹션은 응답에 없으면 자동 숨김
   - 각 항목: 아바타(이니셜 fallback) + 이름 + `actor_type` 뱃지(`사용자` / `에이전트` / `시스템`) + 최근 활동 일시 (relative time)
   - 기간 필터 드롭다운: `최근 7일 / 30일 / 90일 / 전체` → `since=` 쿼리에 반영
   - 빈 상태 / 로딩 스켈레톤 / 에러 상태(권한·네트워크 분리)
   - React Query staleTime 30s, refetchOnWindowFocus false

6. **레이아웃 통합**
   - `frontend/src/app/documents/[id]/page.tsx` 의 우측 컬럼에 `ContributorsPanel` 마운트
   - 좁은 화면(< 1024px) 에서는 패널이 접기 가능한 collapsible 로 전환 (Phase 1/2 의 사이드바 패턴 재사용)

### 2.2 제외

- `expose_viewers` Scope Profile 설정 — **FG 3-2** 책임
- 인라인 주석(annotations) 작성자 표시 — **FG 3-3** 와의 결합은 본 FG 의 편집자 카테고리에 자연 포함되도록 event_type 만 합의(`annotation.created`)
- 외부 SSO display_name 갱신 — S2 사용자 모듈 그대로
- 그래프형 contributor 시각화 — S4 검토

### 2.3 하드코딩 금지 재확인

- 기간 필터 옵션, 섹션별 limit, actor_type 라벨은 i18n 키 또는 config 모듈에서 단일 정의. 라우터/컴포넌트에 직접 숫자/문자열 분산 금지
- "최근 30일" 같은 default 상수는 `services/contributors_service.py` 와 프런트가 `GET /api/v1/contributors/config` 또는 OpenAPI 스키마 default 로 합의

---

## 3. 선행 조건

- Phase 2 게이트 통과
- `audit_events.actor_type` 컬럼 존재 (S2 Phase 3 마이그레이션 적용 — 미적용이면 본 FG 는 블로커)
- FG 2-0 의 `documents.scope_profile_id` ACL 필터가 안정 (Phase 2 회귀 녹색)
- `document.viewed` audit emit 경로 확인. 없거나 누락이면 본 FG Step 1 에서 보강

---

## 4. 구현 단계

### Step 1 — Audit 이벤트 인벤토리 / 보강

1. `audit_events` 의 distinct `event_type` 실측 (`SELECT event_type, COUNT(*) FROM audit_events GROUP BY 1 ORDER BY 2 DESC`) → 산출물 `FG3-1_audit이벤트_실측.md`
2. 본 FG 가 의존하는 4 종 (`document.draft.saved`, `document.published`, `document.metadata.updated`, `document.viewed`) emit 경로 점검
3. `document.viewed` 누락 시 `document_view_service` (또는 GET /documents/{id} 라우터 내부) 에서 1 회 emit 추가 — 중복 emit 방지(같은 viewer + 같은 doc + 5분 이내 중복은 1건으로 합치는 dedup 로직 또는 emit 자체를 throttle)
4. pytest: emit 신규 추가분 unit + 기존 경로 회귀

### Step 2 — Repository 헬퍼

1. `backend/app/repositories/contributors_repository.py` 신설 (또는 `audit_events_repository` 확장)
2. 4 종 집계 쿼리 — 단일 문서 1 회 호출. 각 카테고리는 별 쿼리지만 같은 connection 안에서 순차 실행
3. `users_repository.get_many(ids)` batch fetch 사용

### Step 3 — Service / Bundle 모델

1. `backend/app/models/contributors.py` 의 dataclass: `Contributor`, `ContributorsBundle`
2. `services/contributors_service.py` 의 `get_contributors()` — Step 2 헬퍼 호출 + display_name 머지
3. pytest unit ≥ 12 건 (각 카테고리 비어있음/단일/복수, since 적용, include_viewers off)

### Step 4 — 라우터 / 스키마

1. `api/v1/contributors.py` + `schemas/contributors.py` (Pydantic)
2. `router.py` 등록
3. integration test ≥ 6 건 — 권한 통과/차단(404), since 파라미터, include_viewers on/off, limit, agent actor 표시

### Step 5 — 프런트 컴포넌트

1. `ContributorsPanel.tsx` — 4 섹션 + 기간 필터
2. `ContributorBadge.tsx` — actor_type 뱃지 (Phase 1/2 의 뱃지 컴포넌트 재사용 가능 여부 확인)
3. React Query hook `useContributors(documentId, { since, includeViewers })`
4. node:test ≥ 10 건 — 렌더, 빈 상태, 기간 필터, viewers 섹션 hidden, agent 뱃지

### Step 6 — UI 검수 / 반응형

- desktop / narrow desktop / tablet / mobile 4 관점
- CLAUDE.md 프로젝트 §5.2 기준 **UI 리뷰 ≥ 1회** (Phase 2 와 동일 완화 규정)
- 스크린샷: `산출물/FG3-1_UI리뷰로그.md`

### Step 7 — 검수 / 보안 보고서

- `FG3-1_검수보고서.md` + 재검수 1회
- `FG3-1_보안취약점검사보고서.md`:
  - 다른 Scope 의 viewer 가 `GET /documents/{id}/contributors` 호출 시 **404** (존재 유출 없음)
  - `include_viewers=true` 강제 호출이 가능해도, FG 3-2 결합 후에는 Scope Profile 설정에 의해 강제 false 로 덮인다는 점 — 본 FG 단계에서는 "TODO: FG 3-2 통합 후 회귀" 만 명시
  - audit_events 직접 노출 금지 (raw row 가 응답에 섞이지 않도록 schema 화이트리스트)
  - actor_user_id 가 외부 SSO ID 형식이라도 **표시 이름**만 응답에 포함, 원시 ID 는 admin 전용

---

## 5. API 계약 요약

| 메서드 | 경로 | 설명 |
|-------|------|-----|
| GET | /api/v1/documents/{id}/contributors | 4 섹션 번들. 쿼리: since, include_viewers, limit_per_section |

**응답 스키마 (요약)**:
```json
{
  "creator": { "user_id": "...", "display_name": "...", "actor_type": "user", "last_activity_at": "2026-..." },
  "editors": [ { ... } ],
  "approvers": [ { ... } ],
  "viewers": [ { ... } ]   // include_viewers=false 면 키 자체가 응답에 없음
}
```

---

## 6. 성공 기준

- [ ] `audit_events` 4 종 event_type 실측 + 누락 emit 보강 완료
- [ ] `GET /documents/{id}/contributors` 정상/권한차단/since/include_viewers/agent actor 시나리오 녹색
- [ ] 프런트 패널: 4 섹션 + 기간 필터 + 빈/로딩/에러 상태 + actor_type 뱃지
- [ ] 좁은 화면 collapsible 동작
- [ ] pytest 신규 ≥ 18 건, 베이스라인 유지
- [ ] node:test 신규 ≥ 10 건
- [ ] UI 리뷰 로그, 검수·재검수·보안 보고서 제출

---

## 7. 리스크

| 리스크 | 대응 |
|-------|-----|
| `audit_events` 가 큰 테이블 → 4 종 집계가 느림 | `document_id` 인덱스 확인 (`backend/app/db/connection.py` 의 audit_events DDL). 없으면 본 FG Step 1 에서 추가. since 필터로 범위 제한 |
| `document.viewed` emit 누락 → viewers 섹션 텅 빔 | Step 1 의 인벤토리 산출물에서 명시. 보강 emit 시 throttle 로 폭주 방지 |
| 에이전트가 대량 view → viewers 가 에이전트로 도배 | 응답에서 actor_type=agent 항목은 별도 정렬 가중 down (사람 우선). 관리자 UI 에서 `?actor_types=user` 필터 옵션 (확장점만 마련, 본 FG 는 default 동작만) |
| display_name 누락 → "사용자 ID 노출" | `Unknown` placeholder 로 마스킹 + audit_emitter 가 시스템/에이전트일 때 "Mimir 시스템" / "에이전트 (<scope_profile_name>)" 라벨 |
| FG 3-2 통합 전 viewers 노출 사고 | 본 FG 는 `include_viewers` default **false** + 응답 schema 에서 viewers 키를 옵셔널로 명시. 프런트도 default off |

---

*작성: 2026-04-27 | FG 3-1 Contributors 패널*
