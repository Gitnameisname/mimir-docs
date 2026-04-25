# task1-3 — 토글 안정성 검증 (node_id stable + 뷰 선호 복원)

**작성일**: 2026-04-24
**Phase / FG**: S3 Phase 1 / FG 1-3
**상태**: 착수 대기 (FG 1-1, FG 1-2 블로커)
**담당**: 백엔드(소규모) + 프런트엔드
**예상 소요**: 1.5~2일
**선행 산출물**: `task1-1.md`, `task1-2.md` 완료

---

## 1. 작업 목적

Phase 1 의 핵심 불변식은 **"블록 뷰 ↔ 일반 뷰 토글 시 node_id 가 변하지 않는다"** 이다. 이 불변식이 깨지면 Phase 3 의 인라인 주석과 기존 citation 5-tuple(`document_id / version_id / chunk_id / node_id / content_hash`) 의 유효성이 모두 무너진다.

FG 1-2 에서 node_id extension 을 도입했지만, 회귀 방어는 별도 FG 에서 **전용 테스트 세트 + 장기 불변식 문서화 + 사용자 뷰 선호 저장** 까지 포함해야 한다.

본 작업은:
1. node_id stable 규약을 테스트로 못 박고
2. citation 좌표가 토글·편집 전후 유효한지 간접 검증하고
3. 사용자의 뷰 선호(블록 / 일반) 를 서버에 저장해 다음 세션에 복원
4. 인라인 주석(Phase 3) 의 anchor 계약을 문서로 고정한다.

---

## 2. 작업 범위

### 2.1 포함

1. **node_id stable 전용 회귀 (node:test)**
   - 토글 왕복 (block → flow → block) 후 `editor.getJSON()` 의 모든 block-level 노드 `attrs.node_id` 집합이 **동일 순서로 동일 값**
   - 중간에 문단 추가/삭제/이동한 뒤에도 **다른 노드의 node_id 는 불변**
   - `editor.commands.setContent(sameDoc)` 반복 시 node_id 일관성
   - 마크 변경(bold/italic)/텍스트 수정은 node_id 에 영향 없음
   - 새 block 생성 시에만 `crypto.randomUUID()` 가 생성됨
2. **Citation 좌표 유효성 (pytest + 간접)**
   - 기존 `citation_builder` / `citation_cache` 서비스가 생성하는 5-tuple 중 `node_id` 가 `content_snapshot` 의 노드와 존재 여부 + 위치 매칭되는지 검증하는 헬퍼 테스트 추가
   - Draft 편집 후 `save_draft` → `vectorization` → `citation_builder` 경로를 이어 실행하는 integration smoke (Phase 0 FG 0-1 testcontainers CI 위에서 skip 해제 또는 신규)
3. **뷰 선호 저장 (users.preferences)**
   - Alembic revision `s3_p1_users_preferences_editor_view_mode`:
     - `users.preferences` JSONB 컬럼 NULL 가능 추가 (기존 `scope_profile_id` 와 동일한 마이그레이션 경로 — S2-5 선례 참조)
     - 기본값은 `{}`
   - Backend API:
     - `GET /me/preferences` — 현재 유저 preferences 조회 (authz: self only)
     - `PATCH /me/preferences` — partial update
     - Schema: `UserPreferences` (`editor_view_mode?: "block" | "flow"`, 향후 확장 위한 open dict)
   - 모든 읽기/쓰기 API 에 Scope Profile ACL 통과 (self 접근이므로 traversal 은 해당 없으나 actor_type 감사 기록 의무)
4. **프런트 — 뷰 선호 로드/저장**
   - 에디터 진입 시 `usePreferences()` 훅으로 `editor_view_mode` 로드, `useState` 기본값에 주입
   - 토글 시 400ms debounce PATCH
   - 실패 시 로컬 상태는 유지, toast 안내
5. **인라인 주석 Anchor 계약 문서화**
   - `docs/개발문서/S3/phase1/산출물/FG1-3_주석앵커계약.md`:
     - "Phase 3 에서 도입될 인라인 주석은 `{document_id, version_id, node_id, (optional) span_offset}` 로 앵커링한다"
     - "Phase 1 의 TipTap node_id extension 이 이 앵커의 기반이며, 토글/편집 후에도 node_id 가 변하지 않는다는 불변식을 유지한다"
     - 향후 블록 분할/합병 시의 node_id 정책(분할 시 한쪽만 id 유지, 다른쪽은 신규 생성) 예비 기록

### 2.2 제외

- 실제 인라인 주석 UI 구현 → Phase 3
- Version 간 인라인 주석 마이그레이션 → Phase 3
- Saved View / 레이아웃 선호 → Phase 2
- 뷰 선호 외 다른 preferences (테마, 언어 등) → 본 FG 에서 컬럼만 도입, 타 preferences 확장은 별도 FG

### 2.3 S2 ⑤ 원칙 적용

- `GET/PATCH /me/preferences` 는 에이전트도 호출 가능하도록 현행 actor 해석 경로(`resolve_current_actor`) 를 따름
- 감사 로그에 `actor_type ∈ {"user","agent"}` 기록

---

## 3. 선행 조건

- FG 1-1 완료 (저장 모델 단일화)
- FG 1-2 완료 (TipTap + node_id extension + 토글)
- Phase 0 FG 0-1 testcontainers CI 가 로컬에서도 실행 가능 (integration smoke 용)

---

## 4. 구현 단계

### Step 1 — node:test 회귀 스위트

- `frontend/tests/node_id_stable.test.ts` (신설)
- 케이스:
  - `roundtrip_block_to_flow_to_block`: 동일 doc 으로 토글 2회 후 node_id 집합 동일
  - `new_block_gets_new_uuid`: 블록 추가 시 기존 노드 id 영향 없음, 신규 노드만 새 UUID
  - `edit_text_keeps_node_id`: 텍스트 수정 후 동일 노드 id
  - `move_block_keeps_node_id`: ▲▼ 이동 후 동일 노드 id (순서만 변경)
  - `delete_block_removes_only_target_id`: 삭제 후 다른 노드 id 불변
  - `set_content_preserves_existing_ids`: `setContent` 호출 시 입력 JSON 의 id 보존
- TipTap Editor 를 headless 로 생성하는 테스트 헬퍼 (`tests/helpers/makeEditor.ts`)

### Step 2 — pytest citation 좌표 유효성

- `backend/tests/services/test_citation_builder_fg13.py` (신설)
- 케이스:
  - 주어진 `content_snapshot` 에서 빌드한 citation 의 `node_id` 가 실제 doc 트리의 노드와 매칭
  - `content_hash` 재계산 시 저장된 값과 일치
  - Snapshot 편집(노드 내용 수정) → content_hash 가 바뀌어 `hash_matches=False` 로 검출
- 기존 Phase 0 skip 8건 중 `test_retrieval_s7_fg03.py::test_hash_mismatch_returns_modified_true` 가 본 스위트와 스키마 겹침 → 해제 후 이 스위트 안에서 커버

### Step 3 — Alembic: users.preferences

- `backend/app/db/migrations/versions/s3_p1_users_preferences.py`
- `upgrade()`:
  - `ALTER TABLE users ADD COLUMN preferences JSONB NOT NULL DEFAULT '{}'::jsonb`
  - 기존 레코드는 기본값 자동 적용
- `downgrade()`: 컬럼 drop
- 검증: `pytest tests/migrations/` 의 Alembic head 이동 테스트 포함

### Step 4 — Backend API: /me/preferences

- `backend/app/api/v1/me.py` (이미 있으면 확장, 없으면 신설)
- Schema:
  ```python
  class UserPreferences(BaseModel):
      editor_view_mode: Optional[Literal["block", "flow"]] = None
      # 향후 확장을 위해 model_config = ConfigDict(extra="allow")
  ```
- 라우트:
  - `GET /me/preferences` → 200 + `UserPreferences`
  - `PATCH /me/preferences` → 200 + 갱신 후 `UserPreferences`
  - authz: `require_authenticated=True`, actor.resolved_id == path-less self
- 감사: `event_type="preferences.updated"`, `actor_type` 필드 기록

### Step 5 — Frontend: usePreferences 훅

- `frontend/src/hooks/useUserPreferences.ts`
- `useQuery(['me','preferences'], () => mePreferencesApi.get())`
- `useMutation` PATCH + optimistic update
- 400ms debounce util

### Step 6 — Frontend: 토글 상태와 연동

- `DocumentEditPage` 의 `useState<EditorViewMode>('block')` → `useUserPreferences()` 의 `editor_view_mode ?? 'block'` 을 초기값으로
- 토글 시 debounced PATCH 호출
- 실패 시 toast "뷰 설정을 저장하지 못했어요. 다음 접속에 복원되지 않을 수 있습니다." (로컬 상태는 유지)

### Step 7 — Integration smoke (Phase 0 CI 활용)

- `backend/tests/integration/test_fg13_editor_roundtrip.py`
- 흐름:
  - 로그인 → 문서 생성 → PUT /draft (ProseMirror doc) → vectorize (Phase 0 경로) → 같은 node_id 가 citation 에 포함되는지 확인 → `draft.content_snapshot` 노드 수정 → content_hash 불일치 감지
- Phase 0 FG 0-1 의 IT-02, IT-07 과 조합 가능

### Step 8 — 앵커 계약 문서

- `docs/개발문서/S3/phase1/산출물/FG1-3_주석앵커계약.md`
- Phase 3 에 인계 예정. 섹션:
  1. 앵커 키 구조
  2. node_id 불변식
  3. 블록 분할/합병 정책(예비)
  4. Phase 3 에 남긴 TODO

---

## 5. 검수 기준

| # | 기준 | 근거 |
|---|------|------|
| C1 | node:test 회귀 6종 녹색 | `tests/node_id_stable.test.ts` |
| C2 | pytest citation 좌표 유효성 4종 녹색 | `tests/services/test_citation_builder_fg13.py` |
| C3 | Phase 0 skip 중 `test_hash_mismatch_returns_modified_true` 해제 | pytest 실행 로그 + skip 건수 감소 |
| C4 | Alembic head 이동/되돌리기 정상 (upgrade → downgrade → upgrade 사이클) | pytest migration 테스트 |
| C5 | `GET/PATCH /me/preferences` 인증/권한/감사 정상 | backend pytest + 수동 curl |
| C6 | 에디터 진입 시 마지막 선택 뷰로 복원 | 수동 smoke |
| C7 | 뷰 토글 debounce 400ms, 실패 시 로컬 상태 유지 + toast | 수동 smoke |
| C8 | Integration smoke 통과 (Phase 0 CI 위에서 실행) | CI 아티팩트 |
| C9 | `FG1-3_주석앵커계약.md` 제출, Phase 3 가 참조할 수 있는 수준으로 명시 | 리뷰 1회 |
| C10 | pytest 베이스라인 `2,662 passed` 이상(본 FG 가 +N 증가), skip 은 Phase 0 시 124 → 123 → FG1-1 시 123 → 본 FG 에서 122 (hash mismatch 해제) | 실 실행 로그 |

---

## 6. 산출물

| 파일 | 설명 |
|------|------|
| `docs/개발문서/S3/phase1/산출물/FG1-3_검수보고서.md` | C1~C10 체크리스트 + 실행 로그 |
| `docs/개발문서/S3/phase1/산출물/FG1-3_보안취약점검사보고서.md` | `/me/preferences` 권한 경계 검증, JSONB injection 방어, actor_type 기록 |
| `docs/개발문서/S3/phase1/산출물/FG1-3_주석앵커계약.md` | Phase 3 인계 |

---

## 7. 리스크 / 열린 질문

| # | 리스크 | 대응 |
|---|-------|-----|
| R1 | TipTap 내부에서 `setContent` 호출 시 일부 plugin 이 doc 을 rewrite 해 node_id 를 날림 | 테스트로 조기 발견. StarterKit 의 history / dropcursor 등 plugin 이 영향 있는지 확인 |
| R2 | users.preferences 컬럼이 타 preferences 확장에 의해 schema drift 발생 | `UserPreferences` 를 `ConfigDict(extra="allow")` 로 두되, alembic 에서는 JSONB 자유 형식 유지. 주요 키는 validator 로 명시 |
| R3 | `/me/preferences` 가 agent 호출 시 의미 불분명 ("agent 의 view_mode" 가 없음) | agent 는 정상 응답하되 `editor_view_mode` 는 자기 필드가 아닌 "호출자 user" 의 것으로 해석. Phase 3 에서 재검토 |
| R4 | integration smoke 가 로컬에서 flaky | Phase 0 CI 인프라 재활용. 로컬 실패 시 `pytest -m "not integration"` 로 격리 |
| Q1 | debounce 값 400ms 는 적절한가? | 토글 빈도 고려해 200~500ms 범위 스모크 후 확정 |
| Q2 | PATCH 가 partial update 시 서버 측 merge semantics (deep vs shallow)? | 1레벨 shallow 로 시작. 중첩 키가 생기면 Phase 3 에서 재설계 |

---

## 8. 참조

- `docs/개발문서/S3/phase1/Phase 1 개발계획서.md` §1, §2 FG 1-3
- `docs/개발문서/S3/phase1/산출물/Pre-flight_실측.md` §6.3
- `CLAUDE.md` S1 ④ + S2 ⑤⑥
- Phase 0 FG 0-1 `IT-07` (embedding_dim 검증 경로와 동일한 testcontainers 기반)
- S2-5 `users.scope_profile_id` Alembic 경로 (동일 패턴)
- FG 1-1 / FG 1-2 작업지시서

---

*작성: 2026-04-24 | Phase 1 FG 1-3 착수 대기 (FG 1-1, FG 1-2 블로커)*
