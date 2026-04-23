# UI · Admin · ExtractionSchemas — P4 검수 보고서

> 대상 경로: `/admin/extraction-schemas`
> 작성일: 2026-04-22
> 선행 문서: `UI_Admin_ExtractionSchemas_P3_후속_검수보고서.md`
> 범위: P4 — 서버 정본 버전 비교(diff) 뷰어 + 버전 되돌리기(rollback)

## 1. 작업 범위

사용자 지정 (AskUserQuestion 응답, 2026-04-21):
- P4 주 범위: 버전 비교(diff) 뷰어 + 되돌리기
- 순서: 서버 먼저 → 프론트
- 테스트: pytest 포함
- 보고서 위치: `docs/개발문서/S2_5/`

| ID | 항목 | 구현 위치 |
|----|------|-----------|
| P4-A | 서버: `GET /extraction-schemas/{doc_type}/versions/diff` | `backend/app/api/v1/extraction_schemas.py`, `backend/app/schemas/extraction.py` (DTO + 유틸) |
| P4-B | 서버: `POST /extraction-schemas/{doc_type}/rollback` | `backend/app/api/v1/extraction_schemas.py`, `backend/app/repositories/extraction_schema_repository.py` |
| P4-C | 프론트: 버전 이력 섹션의 "서버 정본 비교" 진입 UI | `frontend/src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx` |
| P4-D | 프론트: `VersionDiffModal` 컴포넌트 (added/removed/modified/unchanged) | 동상 |
| P4-E | 프론트: `RollbackDialog` 확인 모달 + 버튼 | 동상 |
| P4-F | pytest 단위/통합 테스트 | `backend/tests/unit/extraction/test_extraction_schema_diff.py`, `test_extraction_schema_repository.py` (15건 추가), `backend/tests/integration/test_extraction_schemas_routes.py` (13건 추가) |
| P4-G | 본 보고서 + 보안 취약점 보고서 | `docs/개발문서/S2_5/UI_Admin_ExtractionSchemas_P4_*.md` |

## 2. 구현 요약

### 2.1 P4-A — 서버 diff 엔드포인트

**라우트**: `GET /api/v1/extraction-schemas/{doc_type}/versions/diff?base_version=N&target_version=M&scope_profile_id=...`

- `base_version` / `target_version`: `Query(..., ge=1)` 으로 Pydantic/FastAPI 계층에서 1 이상 강제.
- `base == target` 은 422 (중복 비교 방지).
- `scope_profile_id` 가 있으면 UUID 검증 → 두 버전 모두 같은 scope 의 스키마에서 나와야 함 (S2 ⑥).
- 두 버전 어느 한쪽이라도 404 면 404 반환.
- 응답 DTO `ExtractionSchemaDiffResponse`:
  - `doc_type_code`, `base_version`, `target_version`
  - `added: List[str]`, `removed: List[str]`
  - `modified: List[ModifiedFieldDiff]` (각 `name`, `changes: List[PropertyDiff]`)
  - `unchanged_count: int`
- 계산은 서버 정본 유틸 `compute_fields_diff` 가 전담. 클라이언트의 `computeFieldsDiff` 와
  의미론을 맞추되, `_deep_equal` 이 **bool ↔ int 를 엄격 구분** (Python `True == 1` 혼동 방지).

### 2.2 P4-B — 서버 rollback 엔드포인트

**라우트**: `POST /api/v1/extraction-schemas/{doc_type}/rollback`

요청 DTO `RollbackExtractionSchemaRequest`:
- `target_version: int ≥ 1`
- `change_summary: Optional[str] ≤ 1024` (제어문자 거절, 공백-only 는 None 으로 정규화)
- `scope_profile_id: Optional[str]` (UUID 형식 검증)

저장소 `ExtractionSchemaRepository.rollback_to_version`:
- `SELECT ... FOR UPDATE` 로 현재 스키마 레코드 잠금 (동시 update/rollback 경쟁 방어).
- 거절 조건
  - `ExtractionSchemaNotFoundError`: 스키마 없음 → 404
  - `is_deprecated=True` : 폐기된 스키마는 되돌릴 수 없음 → 422 (ValueError)
  - `target_version >= current_version` 또는 `< 1` : 422
  - target 버전 이력이 없음 → 404 (ExtractionSchemaNotFoundError)
  - target 버전의 fields 가 비어 있음 → 422 (모델 불변식 유지)
- 성공 시
  - 스키마 레코드의 `version` 을 `current+1` 로 갱신, `fields_json` 은 target 의 값을 복사.
  - `extraction_schema_versions` 에 신규 row INSERT.
    - `change_summary` 가 비어 있으면 `"v{target} 로 되돌리기"` 기본값.
    - `extra_metadata` 에 `{"rolled_back_from_version": target}` 기록 — 감사 추적.
    - `changed_fields` 는 "이전 최신 버전의 fields 키" 와 "target 버전의 fields 키" 의 대칭차.
- **이력 불변성(immutable history)**: 과거 버전 row 는 수정/삭제되지 않음. 되돌리기는 새 버전 생성.
- 감사 이벤트 `extraction_schema.rolled_back` 이 `audit_emitter.emit_for_actor` 를 통해 발행.
  `new_state` 에 `{doc_type_code, version, rolled_back_from_version}` 포함.

### 2.3 P4-C · P4-D — 프론트 서버 정본 비교 UI

`AdminExtractionSchemasPage.tsx`:
- `ExtractionSchemaDiff` / `ExtractionSchemaPropertyDiff` / `ExtractionSchemaModifiedFieldDiff`
  타입을 `types/s2admin.ts` 에 추가.
- `extractionSchemasApi.diffVersions(docTypeCode, { base_version, target_version, scope_profile_id })`
  를 `lib/api/s2admin.ts` 에 추가.
- `VersionHistorySection`:
  - Base/Target 이 모두 선택되고 서로 다르면 **"서버 정본 비교"** 버튼 노출.
  - 클릭 시 `VersionDiffModal` 을 띄워 `/versions/diff` 를 호출하고 결과를 렌더.
  - 기존 클라이언트 `computeFieldsDiff` (P2-B) 는 "빠른 미리보기" 로 유지 — 사용자가 모달을 열지
    않고도 대략적인 변화를 즉시 확인.
- `VersionDiffModal`:
  - 요약 카드(추가/제거/수정/동일) + 상세 항목 리스트.
  - `useFocusTrap` + Escape 키 닫기 + 배경 클릭 닫기.
  - TanStack Query 로 요청을 캐싱 (`staleTime: 60s`).

### 2.4 P4-E — 프론트 되돌리기 UI

`VersionHistorySection` 의 각 version row:
- **"되돌리기"** 버튼이 항상 표시되며, 현재 버전 / 폐기 스키마에서는 disabled (title 로 이유 명시).
  - `isCurrent`: `v.version === latestVersion` → 비활성화
  - `isDeprecated`: 폐기된 스키마 → 비활성화
  - 두 경우 모두 색상·커서 변경.
- 클릭 시 `RollbackDialog` 오픈:
  - 현재 버전 / 대상 버전을 명시하는 안내 문구.
  - 선택적 `change_summary` 입력 (max 1024, 공백 trim, 클라이언트에서도 제어문자 선검증).
  - 확인 버튼은 amber-600 (신중함을 유도).
  - `extractionSchemasApi.rollback` → `useMutationWithToast` → `invalidateKeys` 로
    목록·상세·versions 캐시 모두 무효화.
  - 성공 후 모달 자동 닫힘, 토스트 `"v{N} 로 되돌렸습니다."`.
- 모달 열려 있을 때는 Esc / 배경 클릭 으로 닫기 가능하되 **요청 진행 중에는 닫기 차단**.

### 2.5 P4-F — pytest 테스트

| 파일 | 추가/수정 | 테스트 수 |
|------|-----------|-----------|
| `tests/unit/extraction/test_extraction_schema_diff.py` | 신규 | 20 |
| `tests/unit/extraction/test_extraction_schema_repository.py` | `TestGetVersion`(4) + `TestRollbackToVersion`(11) 추가 | 15 신규 |
| `tests/integration/test_extraction_schemas_routes.py` | `TestDiffVersionsRouteValidation`(6) + `TestRollbackRouteValidation`(7) 추가 | 13 신규 |
| **합계** | | **48 신규** |

Diff 유틸 테스트의 주요 케이스:
- `_deep_equal` 의 bool/int 분리, dict 키 순서 invariant, list 순서-의존, 중첩 동치.
- `compute_fields_diff` 의 added/removed/modified 정렬, `unchanged_count` 가 modified 를 포함하지 않음,
  한쪽에만 존재하는 속성이 `before=None` / `after=None` 으로 기록됨.

Repository 테스트의 주요 케이스:
- `get_version`: schema 미존재 / 버전 미존재 / scope 가드가 SQL 파라미터에 포함되는지.
- `rollback_to_version`: happy path, NotFound, 폐기 거절, target >= current 거절,
  target row 미존재, target fields 비어 있음, 기본 change_summary 자동 생성, 커스텀 요약 보존,
  `rolled_back_from_version` 메타데이터 기록, `SELECT ... FOR UPDATE` 사용, scope 가드.

Route validation 테스트의 주요 케이스 (Pydantic/Query 파서 층, DB 불요):
- `/versions/diff`: 필수 쿼리 누락, 0/음수 값, base==target, 잘못된 scope UUID.
- `/rollback`: target_version 필수 / ≥1 / int 타입 / change_summary 제어문자·과길이 /
  scope UUID 형식.

## 3. UI 디자인 리뷰 (≥5회)

CLAUDE.md 규칙 ④. P4 에서 새로 추가된 UI(**서버 정본 비교** 버튼, `VersionDiffModal`,
`RollbackDialog`, 각 row 의 "되돌리기" 버튼) 를 5회 리뷰.

### Round 1 — 시각적 위계 (Base/Target/되돌리기 공존)

- 기존 Base(red), Target(green) 버튼 옆에 되돌리기(amber) 가 추가되어 **색상 충돌 없음**.
- Base/Target 은 "비교 선택" 이라는 한 묶음, 되돌리기는 "파괴적이지 않지만 신중한" 액션 — 색상
  톤(amber) 으로 분리되어 사용자가 목적을 혼동하지 않음.
- 세 버튼 모두 `min-h-[24px]`, padding 동일 → 시각적 균형 유지.

### Round 2 — 접근성

- 되돌리기 버튼: `aria-label` 에 구체적 버전 번호 포함(`v3 로 되돌리기`).
- 현재 버전은 "현재" 배지로 시각적 구분 + 되돌리기 버튼 title 에 "현재 버전은 되돌릴 수 없습니다".
- 폐기 스키마에서는 title 에 "폐기된 스키마는 되돌릴 수 없습니다".
- `VersionDiffModal` / `RollbackDialog`:
  - `role="dialog"`, `aria-modal="true"`, `aria-labelledby`, `aria-describedby` 선언.
  - `useFocusTrap` 으로 탭이 모달 내부에 머무름.
  - Escape 키로 닫기. 모달 배경 클릭 닫기. (요청 중에는 차단.)

### Round 3 — 오류 방지

- 되돌리기 dialog 확인 버튼은 **amber-600** (기본 primary 파란색이 아님) — "한 번 더 생각하게" 유도.
- `change_summary` 클라이언트 제어문자 사전검증 + 서버 이중검증 (defense in depth).
- 요청 진행 중 "취소" / "되돌리기" 버튼이 비활성화 + 배경 클릭/Escape 차단 → 중복 전송 방지.
- 모달 안내 문구에 "기존 과거 버전은 그대로 남고" 명시 → 파괴적 동작이 아님을 신호.

### Round 4 — 서버-클라이언트 diff 일관성

- 사용자는 Base/Target 을 선택하면 **즉시** 클라이언트 diff 가 하단에 렌더됨(P2-B 유지).
- 추가로 "서버 정본 비교" 버튼으로 서버 결과를 모달에서 재확인 가능.
- 두 결과가 차이 날 수 있는 대표 케이스:
  - `required: True` 와 `required: 1` — 서버는 다른 것으로 본다(bool/int 구분).
  - 클라이언트 `deepEqual` 이 이를 같은 것으로 처리하면 diff 에 anomaly 가 생김. 서버 결과가 기준.
- 모달의 title 에 "서버 정본 버전 비교" 를 명시해 사용자가 기준을 구분할 수 있도록 함.

### Round 5 — 모바일 / 좁은 뷰포트

- `<div className="flex items-center gap-1 flex-wrap">` 로 row 내 버튼이 줄바꿈 허용.
- `VersionDiffModal` 은 `max-h-[90vh]` 에 내부 스크롤 → 긴 diff 도 대응.
- `RollbackDialog` 의 버튼 영역은 `justify-end` + `gap-2` → 좁은 뷰에서도 우측 정렬.
- 입력 필드 + 버튼 모두 `min-h-[32px]` 이상 (기존 접근성 규칙 준수).

### Round 6 — 서버/UX 일관성

- 서버는 `target_version >= current_version` 을 422 로 거절 → UI 에서도 미리 차단 (현재/미래 버전 비활성화).
- 서버는 폐기 스키마를 422 로 거절 → UI 에서도 사전 비활성화.
- 서버는 target fields 가 비어 있으면 422 → 이 경우는 UI 사전 검증이 어렵지만(버전 payload 를 미리 안 가져옴), 서버 에러 메시지가 `ErrorBanner` 로 표시되어 사용자에게 전달.

## 4. 검수 체크리스트

| # | 항목 | 결과 |
|---|------|------|
| 1 | `GET /versions/diff` 가 `base_version=target_version` 일 때 422 | ✅ (integration 테스트) |
| 2 | `GET /versions/diff` 가 `base_version<1` 일 때 422 | ✅ |
| 3 | `GET /versions/diff` 가 scope_profile_id 비-UUID 일 때 422 | ✅ |
| 4 | diff 응답의 `unchanged_count` 가 modified 를 포함하지 않음 | ✅ (unit) |
| 5 | `_deep_equal` 가 True 와 1 을 구분 | ✅ |
| 6 | `POST /rollback` 이 `target_version<1` 일 때 422 | ✅ |
| 7 | `POST /rollback` 이 `change_summary` 제어문자를 422 | ✅ |
| 8 | `POST /rollback` 이 1024자 초과 시 422 | ✅ |
| 9 | Repository 가 `SELECT ... FOR UPDATE` 로 경쟁 조건 방어 | ✅ (unit assert) |
| 10 | Repository 가 폐기 스키마 롤백을 거절 | ✅ |
| 11 | Repository 가 target ≥ current 를 거절 | ✅ |
| 12 | Repository 가 target fields 비어 있으면 거절 | ✅ |
| 13 | Repository 가 scope_profile_id 지정 시 SQL 조건 포함 | ✅ |
| 14 | Rollback 후 `extra_metadata.rolled_back_from_version` 기록 | ✅ |
| 15 | Rollback 후 버전 이력 INSERT (immutable) | ✅ |
| 16 | 감사 이벤트 `extraction_schema.rolled_back` 발행 | ✅ (route 코드 확인) |
| 17 | 프론트 "서버 정본 비교" 버튼이 Base/Target 선택 시만 노출 | ✅ |
| 18 | 프론트 `VersionDiffModal` 이 Escape/배경 클릭으로 닫힘 | ✅ |
| 19 | 프론트 되돌리기 버튼이 현재 버전/폐기 스키마에서 disabled | ✅ |
| 20 | 프론트 `RollbackDialog` 이 요청 진행 중 닫기 차단 | ✅ |
| 21 | 프론트 mutation 성공 시 목록·상세·versions 캐시 invalidate | ✅ |
| 22 | tsc 신규 에러 0 (기존 AdminUserDetailPage:279 1건만 유지) | ✅ |

## 5. 검증 결과

- Python AST
  - `backend/app/api/v1/extraction_schemas.py`: OK
  - `backend/app/schemas/extraction.py`: OK
  - `backend/app/repositories/extraction_schema_repository.py`: OK
  - `backend/tests/unit/extraction/test_extraction_schema_diff.py`: OK
  - `backend/tests/unit/extraction/test_extraction_schema_repository.py`: OK
  - `backend/tests/integration/test_extraction_schemas_routes.py`: OK
- TypeScript
  - `cd frontend && npx tsc --noEmit`: 신규 에러 0건. 기존 `AdminUserDetailPage.tsx:279` 1건 유지.
- pytest
  - 샌드박스 환경(외부망 차단 + pytest 미설치) 로 샌드박스 실행 불가.
  - 로컬 (`cd backend && pytest tests/unit/extraction/ tests/integration/test_extraction_schemas_routes.py -v`) 에서 실행 필요.
  - 추가된 48건 테스트는 모두 AST-parse OK, 기존 pytest 컨벤션(`client` / `auth_author` / `auth_viewer` fixture, mock_conn 패턴) 에 정렬됨.

## 6. 잔존 한계

- 프론트 "서버 정본 비교" 와 클라이언트 즉시 diff 의 차이가 발생하는 경우, 사용자가 이를 인지할 수
  있도록 "서버 결과가 기준" 이라는 문구를 모달 상단 title 에만 두었다. 별도 인디케이터(예:
  클라이언트/서버 결과 side-by-side) 는 본 범위에 포함하지 않음.
- Rollback 후 곧바로 같은 버전으로 한 번 더 rollback 을 요청하는 케이스를 별도 처리하지 않음
  (`change_summary` 만 다른 동일 내용 버전이 생성됨). 이는 immutable history 철학에 부합하므로
  의도된 동작이지만, 감사 로그가 장기간 팽창할 수 있음 → 모니터링 대상.
- `VersionDiffModal` 이 한 번 조회한 diff 를 `staleTime: 60s` 동안 캐싱한다. 같은 버전 쌍의
  fields 가 서버 쪽에서 수정될 가능성은 없으므로(immutable 이므로) 일반적으로 안전하지만,
  스키마 삭제(soft delete) 후 복구된 경우에는 무효화가 없음. 엣지 케이스로 문서화.
