# task2-5 — Saved View

**작성일**: 2026-04-24
**Phase / FG**: S3 Phase 2 / FG 2-5
**상태**: 착수 대기 (FG 2-1 ~ 2-4 필터 키 확정 후)
**담당**: 백엔드 + 프런트
**예상 소요**: 2~3일
**선행 산출물**:
- `docs/개발문서/S3/phase2/산출물/Pre-flight_실측.md`
- `task2-1.md` ~ `task2-4.md` — 필터 키 / 레이아웃 확정

---

## 1. 작업 목적

사용자가 자주 쓰는 **필터(document_type / scope / tag / collection / folder / 기간) + 정렬 + 레이아웃** 조합을 **Saved View** 로 저장하고 URL 로 공유한다. 공유 URL 로 접근해도 **뷰어의 Scope Profile ACL 이 여전히 적용** (결과는 뷰어 Scope 에서 재필터).

---

## 2. 작업 범위

### 2.1 포함

1. **DB 스키마 (Alembic revision `s3_p2_saved_views`)**
   - `saved_views (id UUID pk, owner_id UUID FK users, name VARCHAR(200), filter JSONB NOT NULL, sort JSONB NOT NULL, layout VARCHAR(16) NOT NULL, include_tag_nodes BOOL DEFAULT false, created_at, updated_at)`
   - 인덱스: `saved_views(owner_id)`
   - **ACL**: owner 만 접근. URL 공유 시 "정의" 만 공개, "결과" 는 viewer Scope 로 재필터

2. **백엔드 — API**
   - `GET /saved-views` — owner 본인
   - `POST /saved-views` — {name, filter, sort, layout, include_tag_nodes?}
   - `GET /saved-views/{id}` — owner 또는 공유 URL 접근자(정의 응답만)
   - `PATCH /saved-views/{id}`
   - `DELETE /saved-views/{id}`
   - `GET /saved-views/{id}/apply` — 정의를 조회 컨텍스트에 주입해 `/documents` 로 리디렉트 URL 반환. 또는 프런트가 직접 정의 fetch 해서 URL 구성
   - 모든 쓰기/조회는 FG 2-0 documents 필터를 그대로 통과

3. **필터 / 정렬 JSON 스키마 (schemas/saved_views.py)**
   - filter: `{document_type?: str[], status?: str[], tag?: str[], collection?: UUID[], folder?: UUID, include_subfolders?: bool, created_from?: ISODate, created_to?: ISODate, owner_id?: UUID}`
     - 모든 필드 optional. 여러 필드 **AND** 조건
   - sort: `{field: "created_at"|"updated_at"|"title", direction: "asc"|"desc"}` 배열 (multi-sort)
   - layout: `"list"|"tree"|"cards"|"graph"`
   - **schema validation**: Pydantic 모델로 serve/reject. 허용 외 필드는 거부 (extra="forbid")

4. **프런트엔드 — Saved View UI**
   - `features/documents/SaveViewButton.tsx` — "현재 필터 저장" 버튼. 이름 입력 모달
   - `features/documents/SavedViewsMenu.tsx` — 사이드바 하단 또는 /documents 상단에 드롭다운. 선택 시 `/documents?view=<id>` 로 이동
   - `/documents?view=<id>` 파라미터 있으면 URL 의 다른 필터는 view 정의로 **덮어쓰기** (view 가 우선)
   - "공유 URL 복사" 버튼 (클립보드)

5. **정책 — 공유 URL 의 ACL**
   - 공유 URL `/documents?view=<id>` 를 C 가 열면:
     1. `GET /saved-views/{id}` 로 정의 fetch (owner 가 아니어도 정의는 읽기 허용, 단 `name/filter/sort/layout` 만 반환. `owner_id` 는 마스킹)
     2. 정의를 `/documents` query 로 풀어 fetch
     3. `/documents` 는 **C 의 Scope Profile** 로 ACL 필터
   - 결과가 비면 "권한 밖 결과는 제외되었습니다" 안내 표시 (단, 구체 건수는 노출 안 함 — 존재 유출 방지)

### 2.2 제외

- 팀 공용 Saved View — S4 범위
- 대시보드 / 위젯 구성 — Phase 3 이후
- AI 기반 필터 추천 — Phase 3 이후

### 2.3 하드코딩 금지

- filter 키 집합은 `schemas/saved_views.py` 가 정본. 프런트 타입은 OpenAPI 생성 또는 hand-written 공유 타입으로 동기화

---

## 3. 선행 조건

- FG 2-1 ~ 2-4 의 필터 키 집합 확정 (document_type, status, tag, collection, folder, 기간, layout)
- `users.preferences` 에 `last_used_view_id` 또는 유사 키 추가는 본 FG 에서 옵션

---

## 4. 구현 단계

### Step 1 — DB / Repository

1. Alembic revision `s3_p2_saved_views`
2. `app/models/saved_view.py`, `SavedViewsRepository`
3. pytest: CRUD + owner 필터

### Step 2 — Pydantic Schema

1. `schemas/saved_views.py` — `SavedViewFilter`, `SavedViewSort`, `SavedViewLayout` (Literal)
2. extra="forbid" 적용
3. validator: date 형식, UUID 형식, tag 정규화 (소문자)

### Step 3 — 서비스 / 라우터

1. `services/saved_views_service.py`
2. `api/v1/saved_views.py` — 5 엔드포인트
3. 공유 URL 접근 시 owner 검증 **생략**, 응답은 `owner_id` 마스킹
4. pytest integration: 공유 URL 을 다른 사용자가 열 때 ACL 필터 동작

### Step 4 — 프런트 저장 UI

1. `SaveViewButton` + 모달
2. 저장 후 토스트 + 좌측 메뉴에 즉시 반영 (React Query invalidate)

### Step 5 — 프런트 메뉴 / 적용

1. `SavedViewsMenu` — 목록 + 클릭 시 `/documents?view=<id>`
2. `/documents` 페이지: `view` 쿼리 우선, 없으면 개별 필터 query 사용
3. view 정의 → URL 변환 유틸 `viewToQueryString(view)`

### Step 6 — 공유 URL

1. "복사" 버튼 — `navigator.clipboard.writeText`
2. 공유 후 다른 사용자로 열기 시 ACL 적용 + 안내 배너

### Step 7 — UI 검수 / 반응형

- UI 리뷰 ≥ 1회
- 빈 상태, 긴 이름 말줄임, 많은 view 수 (20+) 스크롤

### Step 8 — 검수 / 보안 보고서

- 일반 + 재검수
- 보안: **"뷰 ≠ 권한 준수 확인"**
  - 공유 URL → viewer Scope 로 재필터 동작
  - `name/filter/sort/layout` 외 필드 노출 안 함 (특히 `owner_id` 마스킹)
  - `apply` 엔드포인트가 서버 측에서 ACL 우회 없는지

---

## 5. API 계약 요약

| 메서드 | 경로 | 설명 |
|-------|------|-----|
| GET | /saved-views | owner 본인 목록 |
| POST | /saved-views | 정의 저장 |
| GET | /saved-views/{id} | 정의 조회 (공개, owner_id 마스킹) |
| PATCH | /saved-views/{id} | owner 만 |
| DELETE | /saved-views/{id} | owner 만 |

---

## 6. 성공 기준

- [ ] CRUD + 공유 URL 동작
- [ ] ACL 재필터 검증 (owner 가 아닌 viewer 에서 Scope 적용)
- [ ] 필터 키 집합 스키마 validator (extra="forbid")
- [ ] `/documents?view=` 적용 + 기존 필터 query 와 상호작용 규약 명시
- [ ] pytest 신규 ≥ 15 건
- [ ] node:test 신규 ≥ 15 건
- [ ] UI 리뷰 로그
- [ ] 검수 · 재검수 · 보안 보고서

---

## 7. 리스크

| 리스크 | 대응 |
|-------|-----|
| filter JSON 에 임의 키 저장 → 서버가 해석 못해 500 | Pydantic extra="forbid" + 서비스 계층에서 whitelist |
| 공유 URL 로 owner 식별 | GET /saved-views/{id} 응답에서 owner_id 마스킹 + `created_at/updated_at` 만 반환 |
| 많은 view 로 목록 비대 | 사용자당 상한 50 + 초과 시 409 |
| 이름 중복 | 사용자당 UNIQUE(owner_id, name) |

---

*작성: 2026-04-24 | FG 2-5 Saved View*
