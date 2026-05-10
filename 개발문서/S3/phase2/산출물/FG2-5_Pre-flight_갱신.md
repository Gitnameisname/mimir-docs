# FG 2-5 Pre-flight 갱신 메모 (재진입 전 사실 baseline)

**작성일**: 2026-05-10
**대상 작업지시서**: `docs/개발문서/S3/phase2/작업지시서/task2-5.md` (작성일 2026-04-24)

---

## 1. 진행 상태 실측

### 1.1 미구현 확인 (2026-05-10)

- ❌ Alembic `s3_p2_saved_views` revision — 부재
- ❌ `app/models/saved_view.py` / `saved_views_repository.py` / `saved_views_service.py` — 부재
- ❌ `app/schemas/saved_views.py` / `app/api/v1/saved_views.py` — 부재
- ❌ `frontend/src/features/documents/SaveViewButton.tsx` / `SavedViewsMenu.tsx` — 부재
- ❌ `frontend/src/lib/api/saved_views.ts` — 부재

### 1.2 선행 조건 (FG 2-1~2-4 종결 후 충족)

- ✅ FG 2-1 (collections / folders) — 종결 (2026-04-24)
- ✅ FG 2-2 (tags) — 종결 (2026-04-25)
- ✅ FG 2-3 (백링크) — 본 세션 종결 (2026-05-10)
- ✅ FG 2-4 (그래프 + 4 layout) — 본 세션 1차 종결 (2026-05-10)
- ✅ DocumentLayout 어휘 정본 — `frontend/src/features/documents/layout.ts` (FG 2-4)

---

## 2. 갱신 항목

### 2.1 Alembic head — `s3_p2_saved_views` 의 down_revision

**현 head (FG 2-3 추가 후)**: `s3_p2_document_links` (2026-05-10)

**갱신 결정**: 새 revision `s3_p2_saved_views` 의 `down_revision = "s3_p2_document_links"` (head 위에 쌓는다).

### 2.2 필터 키 집합 — FG 2-1~2-4 누적 baseline

`SavedViewFilter` Pydantic 스키마 (extra="forbid") 가 받아야 할 필드:

| 키 | 타입 | 출처 FG | 비고 |
|---|------|--------|----|
| `q` | str | FG 2-1 UX 3차 | 제목 부분 일치 (ILIKE) |
| `document_type` | list[str] | (기존) | DocumentTypeBadge 의 값 |
| `status` | list[WorkflowStatus] | (기존) | DRAFT / IN_REVIEW / APPROVED / PUBLISHED / REJECTED / ARCHIVED |
| `tag` | list[str] | FG 2-2 | 정규화된 태그 이름 (소문자) |
| `collection` | list[UUID] | FG 2-1 | UUID 검증 |
| `folder` | UUID? | FG 2-1 | 단일 폴더 (재귀는 별 키) |
| `include_subfolders` | bool? | FG 2-1 UX 4차 | folder 와 결합 |
| `created_from` | ISODate? | (기존) | 기간 시작 |
| `created_to` | ISODate? | (기존) | 기간 끝 |
| `owner_id` | UUID? | (기존) | 작성자 필터 |

> **제외 필드 (whitelist 기반 거부)**: `tag_name_normalized` 형태의 backend 내부 키 / `viewer_scope_profile_ids` / `scope` 직접 노출 — 모두 Pydantic extra="forbid" 로 거부.

**Sort 키**:
- `field`: Literal["created_at", "updated_at", "title"]
- `direction`: Literal["asc", "desc"]
- multi-sort 배열 — task2-5.md §2.1 (3) 명시

**Layout**:
- `Literal["list", "tree", "cards", "graph"]` — FG 2-4 의 `DOCUMENT_LAYOUTS` 어휘와 정합 (frontend layout.ts 단일 정본)

### 2.3 ACL 정책 명시화 (task2-5.md §2.1 (5))

| 엔드포인트 | 인증 요구 | 응답 |
|----------|---------|----|
| `GET /saved-views` | 인증 필요 | owner 본인 목록 |
| `POST /saved-views` | 인증 필요 | 정의 저장 (owner_id = actor) |
| `GET /saved-views/{id}` | 인증 필요 | 정의 (`name/filter/sort/layout/include_tag_nodes/created_at/updated_at`) — **`owner_id` 마스킹** (응답에서 제거) |
| `PATCH /saved-views/{id}` | 인증 + owner 검증 | 본인 view 만 |
| `DELETE /saved-views/{id}` | 인증 + owner 검증 | 본인 view 만 |

**핵심 정책**:
- 공유 URL `/documents?view=<id>` 는 다른 사용자가 열어도 정의 fetch 통과 (인증된 사용자) → `/documents` 가 **viewer 의 ScopeProfile** 로 ACL 재필터
- 응답에 `owner_id` 자체가 마스킹 — owner 식별 불가

### 2.4 owner 당 상한 + 이름 UNIQUE

- 사용자당 최대 50 개 (task2-5.md §7 R-03)
- UNIQUE (owner_id, name) — 같은 owner 가 같은 이름으로 중복 저장 불가 (R-04)
- 초과 / 중복 시 409 Conflict

### 2.5 frontend layout 어휘와 정합

backend `SavedViewLayout` Literal 의 4 값과 frontend `DocumentLayout` (FG 2-4) 의 4 값이 정확히 같아야 한다 — 한쪽 변경 시 양쪽 동시 수정 의무. 검수보고서 §"R2" 에 명시.

### 2.6 view 정의 → URL 변환 정책

frontend `viewToQueryString(view): string` 신설 — 정의를 URL query 로 풀어 router.push.

`/documents` 페이지의 우선순위:
1. `?view=<id>` 가 있으면 → fetch view 정의 → URL 의 다른 query 와 **병합 안 함** (view 가 정의한 키만 사용. 다른 query 는 무시)
2. 없으면 기존 query 그대로

이 우선순위로 인해 user 가 `/documents?view=X&q=hello` 같이 혼합 호출해도 view 정의가 우선. 단, view 적용 후 사용자가 toolbar 에서 필터 조작하면 새 query 로 replace (view 자동 분리).

### 2.7 본 세션 진행 범위

| Step | 내용 | 본 세션 |
|------|------|--------|
| 1 | DB / Repository | ✅ |
| 2 | Pydantic Schema | ✅ |
| 3 | Service + 라우터 + 단위 회귀 | ✅ |
| 4 | SaveViewButton + 모달 | ✅ |
| 5 | SavedViewsMenu + viewToQueryString | ✅ |
| 6 | 공유 URL 복사 + 안내 배너 | ✅ |
| 7 | UI 디자인 리뷰 | 🟡 운영자 잔여 |
| 8 | 검수 / 보안 보고서 | ✅ (종결보고서에 통합) |
| 9 | 종결보고서 + 함수도서관 | ✅ |

---

## 3. P1 승인 게이트

1. **Alembic `s3_p2_saved_views` 적용** — 운영자 환경 `alembic upgrade head`
2. **API 표면 5종 추가** — `/api/v1/saved-views/...`
3. **종결 선언** — UI 디자인 리뷰 + P1 승인 후 공식 종결

승인자: `@최철균`

---

## 4. Phase 5 정합

본 FG 는 ProseMirror schema / mark 와 무관. task5-1 ADR §a 변경 없음.

---

*작성: 2026-05-10 | FG 2-5 Saved View — 재진입 전 Pre-flight 갱신*
