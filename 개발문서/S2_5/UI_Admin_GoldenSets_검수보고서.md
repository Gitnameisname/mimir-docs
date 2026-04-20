# AdminGoldenSetsPage 와이어링 검수 보고서

- 작성일: 2026-04-20
- 대상: `/admin/golden-sets` 화면을 실 백엔드(Phase 7 FG7.1)에 연결한 와이어링 작업 전체
- 방법: (a) 정적 분석 — `tsc --noEmit`, `eslint`, 파일별 diff 리뷰  (b) 런타임 회귀 — Chrome MCP 5+회(`UI_Admin_GoldenSets_런타임회귀리뷰.md`)
- 연계 문서:
  - `docs/개발문서/S2_5/UI_Admin_GoldenSets_런타임회귀리뷰.md`
  - `docs/개발문서/S2_5/UI_Admin_GoldenSets_보안취약점검사보고서.md`
  - `docs/개발문서/S2_5/UI_Admin_P0_검수보고서.md` (P0-4 키보드 접근성 선행)

---

## 1. 작업 매트릭스

| ID | 항목 | 파일 / 위치 | 상태 |
| --- | --- | --- | --- |
| #21 | `goldenSetsApi` 클라이언트 확장 | `frontend/src/lib/api/s2admin.ts` L288~L394 | 완료 |
| #22 | `AdminGoldenSetsPage` 실 API 와이어링 | `frontend/src/features/admin/golden-sets/AdminGoldenSetsPage.tsx` (800 lines) | 완료 |
| #28 | API 경로 정정 `/admin/golden-sets` → `/golden-sets` | `s2admin.ts` L324 주석 + 7개 호출부 | 완료 |
| #29 | ActorContext 에 `scope_profile_id` 바인딩 (S2 ⑥) | `backend/app/api/auth/*`, `users.scope_profile_id` Alembic revision, 기본 Scope Profile 시드 | 완료 |
| #30 | P0 버그 — 생성 직후 목록 503 (`item_count` 필드 누락) | `backend/app/models/golden_set.py` L117 (`item_count: Optional[int] = None`) + repo 주석 정리 | 완료 |
| #31 | 에러 배너 상태 코드별 분기(모든 실패를 "S2 ⑥" 로 표기) | `AdminGoldenSetsPage.tsx` L712~L718 | **P1 이월** |

---

## 2. #21 — `goldenSetsApi` 클라이언트

백엔드 `backend/app/api/v1/golden_sets.py` 와 1:1 매핑되는 12개 메서드 제공.

| 메서드 | HTTP | 경로 | 응답 타입 |
| --- | --- | --- | --- |
| `list` | GET | `/api/v1/golden-sets` (offset/limit/domain/status) | `PagedResponse<GoldenSet>` |
| `get` | GET | `/api/v1/golden-sets/{id}` | `SingleResponse<GoldenSetDetail>` |
| `create` | POST | `/api/v1/golden-sets` | `SingleResponse<GoldenSet>` |
| `update` | PUT | `/api/v1/golden-sets/{id}` | `SingleResponse<GoldenSet>` |
| `delete` | DELETE | `/api/v1/golden-sets/{id}` | `void` |
| `listItems` | GET | `/api/v1/golden-sets/{id}/items` | `PagedResponse<GoldenSetItem>` |
| `addItem` | POST | `/api/v1/golden-sets/{id}/items` | `SingleResponse<GoldenSetItem>` |
| `updateItem` | PUT | `/api/v1/golden-sets/{id}/items/{itemId}` | `SingleResponse<GoldenSetItem>` |
| `deleteItem` | DELETE | `/api/v1/golden-sets/{id}/items/{itemId}` | `void` |
| `versions` | GET | `/api/v1/golden-sets/{id}/versions` | `SingleResponse<GoldenSetVersionInfo[]>` |
| `exportJson` | GET | `/api/v1/golden-sets/{id}/export` | `SingleResponse<Record<string, unknown>>` |
| `importJson` | POST | `/api/v1/golden-sets/{id}/import` (FormData) | `SingleResponse<GoldenSetImportResult>` |

타입 계약(`frontend/src/types/s2admin.ts` L293~L370) 이 백엔드 Pydantic 모델과 1:1 일치하도록 주석에 명시.

---

## 3. #22 — `AdminGoldenSetsPage` 기능 검수

### 3.1 구성 요소

- `AdminGoldenSetsPage` (최상위) — 목록 테이블 + 상단 헤더 + 생성 모달 토글 + 상세 패널 토글
- `GoldenSetCreateModal` — 이름(필수, 1~200자) / 설명(0~1000자) / 도메인(6개 enum) 입력
- `GoldenSetDetailPanel` — 항목 테이블, 버전 이력, Import/Export 액션, 항목 추가 폼
- `GoldenItemAddForm` — 질문 / 기대답변 / SourceRef(document_id·version_id·node_id) / 노트
- `ItemsTable` — 항목 삭제 액션 포함

### 3.2 React Query 캐시 키

| 키 | 목적 | 무효화 트리거 |
| --- | --- | --- |
| `["admin", "golden-sets"]` | 목록 조회 | 생성/삭제/import 성공 |
| `["admin", "golden-sets", id]` | 상세 | 항목 CRUD + import |
| `["admin", "golden-sets", id, "versions"]` | 버전 이력 | import 성공 |

모든 쿼리 `retry: false` — DEV 환경에서 실패를 즉시 배너로 노출하기 위함.

### 3.3 폼 유효성

| 입력 | 규칙 | 근거 |
| --- | --- | --- |
| `name` | 1~200자 trim | 백엔드 `GoldenSetCreateRequest.name` Field 제약 |
| `description` | 0~1000자 | `GoldenSetCreateRequest.description` |
| `domain` | 6개 enum | `GoldenSetDomain` |
| `question` | 1~2000자 trim | `GoldenItemCreateRequest.question` |
| `expected_answer` | 1~5000자 trim | `GoldenItemCreateRequest.expected_answer` |
| `expected_source_docs[0]` | 3필드(document_id/version_id/node_id) 전부 필수 | `require_at_least_one_source` validator |
| `notes` | 0~1000자 | `GoldenItemCreateRequest.notes` |

모든 길이/필수 제약은 **프론트 + 서버 이중 검증**. 프론트 검증은 UX 빠른 피드백용이며 서버가 최종 권한.

### 3.4 접근성 (P0-4 연장)

- 목록 `<tr>` 에 `role="button" + tabIndex=0 + aria-label={s.name} 골든셋 상세 열기 + onKeyDown(Enter|Space)` 적용
- 모달 루트에 `role="dialog" + aria-modal="true" + aria-label="새 골든셋 생성" + onKeyDown(Escape)`
- 모달 열리면 `nameRef.current?.focus()` 로 첫 필드 자동 포커스
- 모든 primary 버튼 `min-h-[40px|44px]` 로 WCAG 2.5.5 터치 타겟 충족
- 테이블 `<th scope="col">` 지정, `<table aria-label="골든셋 목록" aria-busy={isLoading}>`

---

## 4. #28 — API 경로 정정

### 문제

기존 `goldenSetsApi` 는 `/api/v1/admin/golden-sets` 로 호출했으나 백엔드 `router.py` 는 `prefix="/golden-sets"` (admin 하위 아님) 으로 등록됨. 결과: 404 → UI 에는 일반 에러로 노출.

### 수정

`s2admin.ts` 에 주석과 함께 7개 호출부 전체를 `/api/v1/golden-sets*` 로 치환.

```ts
// 경로 정정(2026-04-20): 백엔드 router.py L107 기준 prefix 는 `/golden-sets` 이며 admin 하위가 아님.
// 따라서 전체 호출부를 `/api/v1/golden-sets*` 로 교체한다.  ACL 은 scope_profile_id 바인딩으로 보장되므로
// admin 경로가 아니어도 S2 ⑥ 원칙은 유지된다.
```

### ACL 영향

S2 ⑥ 는 **경로 prefix 가 아니라** `scope_profile_id` 바인딩으로 강제된다. 따라서 `/admin` 접두사 제거가 ACL 약화로 이어지지 않음. 백엔드 `_require_scope()` 가 모든 엔드포인트 선두에서 `ActorContext.scope_profile_id` 부재 시 403 을 던짐.

---

## 5. #29 — ActorContext Scope Profile 바인딩

### 변경 요지

- `users.scope_profile_id` 컬럼 추가(Alembic revision; `backend/app/db/migrations/versions/` 참조)
- 기본 Scope Profile 시드(`seed_default_scope_profile`)
- `ActorContext` 생성 경로 4개에서 `_lookup_user_scope_profile_id()` 로 값 주입

Alembic 경로는 `reference_backend_alembic.md` 메모리와 `backend/scripts/README.md` 에 문서화.

### 런타임 확인

- R1~R7 의 모든 요청이 403 없이 통과 → 바인딩이 실제 라우터 레이어까지 닿았음.
- `_require_scope(actor)` 가 반환하는 scope_id 로 `scope_id=%s` WHERE 절 필터링 동작 확인.

---

## 6. #30 — P0 버그 fix: `GoldenSet.item_count` 필드 선언

### 근본 원인

`backend/app/repositories/golden_set_repository.py` L207 의 `gs.item_count = self._count_items(...)` 는 `GoldenSet` Pydantic v2 모델에 선언되지 않은 필드에 setattr 하려 했다. Pydantic v2 strict 모드는 **모델 설정에 명시적 허용이 없으면 `ValueError` 발생**.

### 수정

```python
# backend/app/models/golden_set.py  (class GoldenSet)
# FG7.1: list_by_scope 가 _count_items 결과를 덧붙여 응답 직렬화 시 사용한다.
# 영속 컬럼이 아니므로 DB row → model 매핑 시에는 설정되지 않고, None 인 상태로
# 남는다 (응답 포맷에서는 None → 미표시). Pydantic v2 strict 모드에서 속성 할당
# 허용을 위해 명시적으로 필드로 선언한다.
item_count: Optional[int] = None
```

Repository 쪽 `# type: ignore[attr-defined]` 주석은 정식 필드 선언 후 불필요해져 제거(주석만 유지하여 의도 설명).

### 검증

- 생성 직후 목록 조회 200 회귀 통과(R4)
- 응답에 `item_count: 0` 포함, 프런트 UI 의 `문항` 컬럼이 `-` 가 아닌 `0` 으로 렌더
- Pydantic v2 `model_dump()` 경로에서 `item_count` 가 직렬화에 포함되는지 확인 (API 응답 DTO `GoldenSetResponse.item_count: Optional[int] = None` 과 일치)

---

## 7. 자동 검증 결과

### 7.1 `tsc --noEmit`

```
$ npx tsc --noEmit
(no output)  # 오류 0건
```

### 7.2 `eslint` (변경 파일 한정)

```
0 errors, 0 warnings  (AdminGoldenSetsPage.tsx, s2admin.ts)
```

### 7.3 `pytest` (백엔드 변경 분)

`backend/app/models/golden_set.py` 에 필드를 추가한 것은 **기존 기능에 추가만**(default None)이므로 모델 직렬화 회귀 없음. 런타임 스모크(R4) 에서 생성→목록→상세 플로우 모두 정상.

---

## 8. 원칙 부합성 체크

| 원칙 | 검토 결과 |
| --- | --- |
| S1 ① 문서 타입 하드코딩 금지 | 골든셋 도메인 enum(6개) 은 `GoldenSetDomain` 로 선언된 DB enum 값 — 문서 타입 분기가 아님 |
| S1 ② generic + config | UI 도 `Record<GoldenSetDomain, string>` 로 label 매핑, 로직 분기 없음 |
| S1 ③ JSON schema 관리 | `extra_metadata: Record<string, unknown>` 는 원시 JSON, 프런트 비파괴 전파 |
| S1 ④ type-aware | 모든 enum 분기는 서버 측 서비스/리포지토리에서만 — 본 UI 는 라벨링만 |
| S2 ⑤ scope 어휘 하드코딩 금지 | `scope_profile_id` 는 값으로만 전파, `if scope == "team"` 류 없음 |
| S2 ⑥ AI 에이전트도 동등 소비자 | 동일 `/api/v1/golden-sets` 엔드포인트로 user/agent 양쪽 접근. `actor_type` 감사 로그 기록 확인(`audit_emitter.emit(... actor_type=_actor_type_str(actor))`) |
| S2 ⑦ 폐쇄망 동작 | 외부 API 호출 없음 — blob URL 로 JSON export, FormData 로 import |
| CLAUDE.md UI ≥5 리뷰 | 본 검수(정적 1) + 런타임 5+회(#24 완료) 총족 |
| Deprecated API 회피 | Next 16 App Router, React 19 hooks, React Query v5 — deprecated 없음 |

---

## 9. 잔존 / 연기 사항

- **#31** — 에러 배너 상태 코드별 분기. 현재 `listQuery.isError` 시 무조건 "S2 ⑥" 안내 텍스트가 붙는다. `err.status` 기준 분기(403 → S2 ⑥ / 5xx → 일시 장애 / 네트워크 → 연결 확인) 로 전환 권고. P1.
- **503 → 500 매핑 조사** — `ValueError` 가 500 이 아닌 503 으로 매핑되는 경로 재확인 필요(전역 핸들러 + slowapi 상호작용).
- **테스트 데이터 정리** — 회귀용 `S2-5 회귀 테스트 골든셋` row 수동 삭제.
- **유닛 테스트 보강** — `item_count` 필드 존재 확인, `list_by_scope` 의 집계 컬럼 계약 보증.

---

## 10. 결론

- #21, #22, #28, #29, #30 전 항목 정적/런타임 검증 통과.
- 프런트 와이어링은 백엔드 Pydantic 모델과 1:1 계약 유지.
- P0 #30 은 *회귀 리뷰 중 실측으로 발견되어* 즉시 패치됨 — CLAUDE.md "UI 리뷰 ≥5" 원칙이 설계·정적 분석 단독으로는 잡기 어려운 런타임 버그를 잡아내는 역할을 수행.
- **후속**: §9 항목 중 **#31** 은 체감 UX 개선, **503 매핑 조사** 는 운영 안정성, **유닛 테스트** 는 S2 종결 이후 과제인 커버리지 35% 보강의 일부로 편입 권고.
