# UI · Admin · ExtractionSchemas — P5 검수 보고서

> 대상 경로: `/admin/extraction-schemas`
> 작성일: 2026-04-22
> 선행 문서: `UI_Admin_ExtractionSchemas_P4_검수보고서.md` §6 "잔존 한계"
> 범위: P5 — P4 잔존 3건 후속 처리 (서버/클라 diff 불일치 인디케이터 · 반복 rollback 경고 · VersionDiffModal 캐시 엣지 케이스)

## 1. 작업 범위

사용자 지시(2026-04-22): "이왕 하는거 다 끝내고 가자" — P4 검수보고서 §6 에 명시된
잔존 한계 3건을 P5 로 묶어 후속 처리.

| ID | 항목 | 구현 위치 |
|----|------|-----------|
| P5-1 | `VersionDiffModal` 에 서버/클라 diff 불일치 인디케이터 | `frontend/src/features/admin/extraction-schemas/diffMismatch.ts`, `AdminExtractionSchemasPage.tsx` |
| P5-2 | 반복 rollback 경고 (동일 `target_version` 연속 롤백 감지) | `backend/app/schemas/extraction.py` (DTO 확장), `frontend/src/types/s2admin.ts`, `diffMismatch.ts`, `AdminExtractionSchemasPage.tsx` |
| P5-3 | `VersionDiffModal` staleTime/cache 엣지 케이스 보강 | `AdminExtractionSchemasPage.tsx` (`VersionDiffModal` queryKey/staleTime) |
| P5-F | pytest + node:test 확장 | `backend/tests/unit/extraction/test_extraction_schema_version_response.py`, `frontend/tests/ExtractionSchemasP5.test.tsx` |
| P5-G | 본 보고서 + 보안 취약점 보고서 | `docs/개발문서/S2_5/UI_Admin_ExtractionSchemas_P5_*.md` |

## 2. 구현 요약

### 2.1 P5-1 — 서버/클라 diff 불일치 인디케이터

**문제 정의**. 서버 `compute_fields_diff` 는 `_deep_equal` 로 bool ↔ int 를 엄격히
구분하지만(Python `True == 1` 을 명시적으로 다른 값으로 취급), 클라이언트의 빠른
미리보기 `computeFieldsDiff` 는 JS strict equality(`===`) 에 의존한다. JSON 직렬
화를 거치는 과정에서 수치 타입이 섞이면 두 결과가 미묘하게 어긋날 수 있다.
P4 까지는 모달 타이틀("서버 정본 버전 비교") 로만 기준을 표시했으나, 사용자가
구체적으로 어느 필드가 다르게 보였는지 확인할 방법이 없었다.

**구현**. `diffMismatch.ts` 의 순수 함수 `diffMismatchSummary(server, client)` 가
7개의 정렬된 배열을 반환한다.

```
DiffMismatchSummary {
  equal: boolean
  serverOnlyAdded: string[]
  clientOnlyAdded: string[]
  serverOnlyRemoved: string[]
  clientOnlyRemoved: string[]
  serverOnlyModified: string[]    // 서버는 수정, 클라는 unchanged
  clientOnlyModified: string[]    // 클라는 수정, 서버는 unchanged
  modifiedKeyDiffers: string[]    // 둘 다 modified, 속성 키 집합 불일치
}
```

`VersionDiffModal` 은 `useMemo` 로 mismatch 를 계산하고, `!mismatch.equal` 일
때만 indigo-50 배너를 렌더한다. 배너에는 각 카테고리별로 해당하는 필드 이름
이 `font-mono` 로 나열되며, `modifiedKeyDiffers` 에는 "bool vs 1/0 등 타입 엄격성 차이 가능" 힌트가 함께 표시된다.

**서버 정본 원칙**. 이 집계는 UI 라벨용일 뿐, 실제 저장/편집은 항상 서버 결과
를 기준으로 한다. 배너 상단 문구 "서버 결과가 로컬 미리보기와 다릅니다 — 이
화면이 기준입니다." 로 사용자에게 명시한다.

### 2.2 P5-2 — 반복 rollback 경고

**문제 정의**. 같은 `target_version` 으로의 연속 롤백은 서버 계약상 허용되지만,
감사 로그(`extraction_schema.rolled_back`) 와 `extraction_schema_versions`
테이블이 의미 없는 중복으로 팽창한다. 사용자가 실수로 같은 버튼을 두 번 클릭
하거나, 방금 롤백한 뒤 즉시 다시 같은 버전을 시도하는 케이스를 UI 에서 경고
해야 한다.

**서버 DTO 확장**. `ExtractionSchemaVersionResponse` 에 `rolled_back_from_version:
Optional[int]` 를 1급 필드로 추가. `from_domain` 이 `extra_metadata["rolled_back_from_version"]`
에서 값을 뽑되 엄격 검증:

```python
if isinstance(raw, bool):
    rolled_back_from = None        # Python bool 은 int 의 서브타입 → 배제
elif isinstance(raw, int) and raw >= 1:
    rolled_back_from = raw
else:
    rolled_back_from = None        # 문자열·리스트·0·음수 → 모두 None
```

`extra_metadata` 딕셔너리 자체는 응답에 포함하지 않아 내부 메타데이터 누설을
방지한다(최소권한 원칙).

**클라이언트 감지 로직**. `diffMismatch.ts` 의 `detectRepeatRollback(recentVersions,
targetVersion, currentVersion)`:

1. `recentVersions` 가 비어 있거나 `head.version !== currentVersion` 이면
   false. (동시 편집으로 헤드가 일시적으로 어긋난 경우 false-positive 방지.)
2. **1순위**: `head.rolled_back_from_version === targetVersion` → `via: "metadata"`.
3. **2순위 호환**: `head.change_summary` 가 기본 패턴 `/^v(\d+)\s*로\s*되돌리기/`
   와 일치하고 N === targetVersion → `via: "summary"`. 구버전 서버나 `extra_metadata`
   가 없는 스키마를 위한 폴백.

**UX**. `RollbackDialog` 에 orange-50 alert 가 뜨고 "그래도 v{N} 로 되돌리겠습니다"
체크박스가 등장한다. 체크박스가 해제되어 있으면 "되돌리기" 확인 버튼이
`disabled` 로 잠긴다(색상 amber-300 + `cursor-not-allowed`). 서버는 여전히
허용하므로 사용자가 체크하면 진행 가능 — **소프트 게이트**(intentional repeat
rollback 을 명시적 행동으로 승격).

### 2.3 P5-3 — VersionDiffModal 캐시 엣지 케이스

**문제 정의**. P4 의 `VersionDiffModal` 은 `staleTime: 60_000` 으로 60초 동안
동일 키의 결과를 재사용한다. 과거 버전 pair 의 fields 는 **immutable** 이므로
이 캐싱은 의미상 안전하지만, 같은 `doc_type_code` 로 **하드 삭제→재생성**된
스키마를 열면 이전 스키마의 diff 결과가 잘못 재사용될 가능성이 있다.

**구현**:

- `VersionDiffModal` 시그니처에 `schemaId: string` 추가.
- queryKey: `["admin", "extraction-schemas", docTypeCode, "diff", schemaId, baseVersion, targetVersion, scopeProfileId ?? null]`.
  `schema_id` 가 달라지면 전혀 다른 캐시 슬롯으로 저장 — 하드 삭제→재생성에서도
  충돌 없음.
- `staleTime: Infinity`. 과거 버전 diff 는 불변이므로 stale 이 될 수 없다.
- `gcTime` 은 TanStack 기본값(5분) 유지. 모달 닫았다가 잠깐 뒤에 다시 열 때
  네트워크 없이 즉시 렌더.
- 스키마 수정 mutation 의 `invalidateKeys: [["admin", "extraction-schemas"]]` 는
  TanStack v5 의 **prefix 매칭** 으로 이 쿼리도 함께 무효화한다(defense-in-depth).

### 2.4 P5-F — 테스트

**Frontend (`node:test`)** — `frontend/tests/ExtractionSchemasP5.test.tsx`, 15건 추가.

| 스위트 | 케이스 수 | 주요 검증 |
|--------|-----------|-----------|
| `detectRepeatRollback` | 7 | 빈 배열, metadata 감지, summary 패턴 감지, target 불일치, head!=current, 일반 편집, metadata 우선순위, summary 비일치 |
| `diffMismatchSummary`  | 8 | 양쪽 empty, 완전 일치, serverOnlyAdded, clientOnlyRemoved, modifiedKeyDiffers, serverOnlyModified, 정렬 보장 |

`diffMismatch.ts` 는 React 를 import 하지 않는 순수 로직 모듈이므로 Node 22
내장 `node:test` 로 ReactDOM 없이 빠르게 실행 가능. 로컬 실행 결과:

```
# tests 15
# pass  15
# fail  0
# duration_ms 39.5
```

`tsconfig.test.json` 의 `include` 에 `diffMismatch.ts` 와 `types/s2admin.ts`
를 추가해 테스트 컴파일이 React 트리 전체를 돌지 않도록 격리.

**Backend (`pytest`)** — `backend/tests/unit/extraction/test_extraction_schema_version_response.py`,
13건 추가(2개 스위트).

| 스위트 | 케이스 수 | 주요 검증 |
|--------|-----------|-----------|
| `TestRolledBackFromVersionExtraction` | 10 | 양의 정수 추출, 키 부재, None 메타, 문자열 거절, bool 거절, list 거절, 0 거절, 음수 거절, 비-dict 메타 안전, 기타 키 미누설 |
| `TestVersionResponseSerialization`    | 3  | 핵심 필드 직렬화, `rolled_back_from_version` 덤프, None 덤프 |

DB 미의존(SimpleNamespace + 실제 `ExtractionFieldDef`) 로 단위 테스트 수준에서
결정론적.

## 3. UI 디자인 리뷰 (≥5회)

CLAUDE.md 규칙 ④. P5 에서 새로 추가된 UI(diff 불일치 indigo 배너, 반복 rollback
orange alert + 체크박스) 를 대상으로 5라운드 리뷰 수행.

### Round 1 — 색상 의미(semantic color) 경고 레벨 분리

- **blue** (기본 정보, Diff 모달 본문): 중립.
- **indigo-50 / 200 / 900** (P5-1): "신경 쓸 만하지만 막지는 않는다" — 투명성 목적.
- **amber** (기존 P4 "되돌리기" 버튼, 신중 요구 액션): unchanged.
- **orange-50 / 300 / 900** (P5-2): "amber 보다 한 단계 더 강한 주의 — 행동 게이트".
- **red** (`ErrorBanner`, 이미 실패): 파괴·오류.

같은 P4/P5 흐름 안에서 **blue → indigo → amber → orange → red** 로 색이 단계
적으로 강해지도록 배치해 사용자가 한눈에 심각도를 구분할 수 있다.

### Round 2 — 접근성

- indigo 배너: `role="status"`, `aria-label="서버 결과와 클라이언트 미리보기 간 차이"`.
  정보 전달용이므로 `alert` 가 아닌 `status` (낮은 우선순위 정중한 스크린리더 알림).
- orange alert: `role="alert"` — 사용자의 행동을 가로막는 경고이므로 즉시 알림.
- 반복 rollback 체크박스: `<label>` 감싸기로 클릭 영역 확장, `focus:ring-2 focus:ring-orange-500`
  로 키보드 포커스 가시성, `disabled={mutation.isPending}` 로 진행 중 잠금.
- 되돌리기 버튼의 `title`: 비활성 이유(`"반복 롤백 경고를 확인하고 체크박스를 선택해 주세요"`)
  를 명시 — 마우스 호버/스크린리더 읽기 모두 대응.
- indigo 배너의 아이콘 `<svg aria-hidden="true">` — 텍스트에 이미 의미가 담겨
  있으므로 아이콘은 장식으로 숨김 처리.

### Round 3 — 오류 방지 & 기본값 안전성

- `recentVersions` 가 undefined/empty 면 `detectRepeatRollback` 이 바로 false
  반환 → 초기 로딩 중에는 경고가 뜨지 않아 false-positive 없음.
- `head.version !== currentVersion` 체크: 동시 편집으로 목록이 잠시 어긋난
  경우 경고를 억제.
- 버튼 disabled 는 `(repeatHint.detected && !userAcknowledged)` 에만 의존 —
  경고가 뜨지 않으면 아무런 영향 없음(기본 UX 회귀 없음).
- `onSuccess` 에서 `setUserAcknowledged(false)` 리셋 → 모달이 다음에 다시
  열리면 초기 상태.

### Round 4 — 모바일 / 좁은 뷰포트

- 두 새 배너 모두 `p-3 text-[11px]` 기반. 작은 폰트라도 `space-y-1` 로 수직 숨쉬는 공간 확보.
- `<ul className="ml-4 list-disc space-y-0.5">` — 좁은 뷰에서도 불릿 + 들여쓰기 유지.
- 체크박스: `w-4 h-4` + `cursor-pointer select-none` — 터치 타깃 16px 이상, 텍스트 드래그 선택 방지.
- 기존 `RollbackDialog` 가 `max-w-md` 모달이므로 orange alert 가 추가되어도 overflow 없음.

### Round 5 — 서버/클라 일관성

- P5-1 배너의 `mismatch.equal === true` 분기는 "완전히 일치했다" 는 **green
  연두 메시지** 를 `role="status"` 로 띄운다(기존 P4 엔드). 기존 UX 유지하면서 mismatch 가 없는 것도 가시적으로 확인 가능.
- P5-2 의 감지 근거(`via: "metadata" | "summary"`) 를 경고 본문에 노출해,
  사용자가 어느 경로로 감지되었는지 파악할 수 있다 — 내부 동작 투명성.
- 구버전 서버(extra_metadata 가 없거나 `rolled_back_from_version` 미기록)는
  summary 패턴으로 폴백 감지 → 점진 배포 중에도 UX 일관.

### Round 6 — 코드 구조 / 테스트성

- `diffMismatch.ts` 를 별도 파일로 분리: React `"use client"` 클라이언트
  번들에서 순수 로직을 떼어냄. Node 22 `node:test` 러너가 ReactDOM 없이 실행.
- `AdminExtractionSchemasPage.tsx` 는 `./diffMismatch` 에서 import 후 alias
  재export (`export const diffMismatchSummary = _diffMismatchSummary;`) —
  기존 호출부 호환성 유지하면서 테스트 격리.
- P5 추가로 생긴 `memo` 2개(`mismatch`, `repeatHint`) 는 의존 배열이 모두
  안정 참조 기반이라 불필요한 재계산 없음.

## 4. 검수 체크리스트

| # | 항목 | 결과 |
|---|------|------|
| 1 | `diffMismatchSummary` 가 equal 케이스를 정확히 판정 | ✅ (node:test) |
| 2 | `diffMismatchSummary` 가 bool vs int 케이스에서 `modifiedKeyDiffers` 로 분류 | ✅ (node:test) |
| 3 | `diffMismatchSummary` 의 7 배열이 모두 정렬됨 | ✅ (node:test "result fields are sorted") |
| 4 | `detectRepeatRollback` 가 metadata 경로를 summary 보다 우선 | ✅ (node:test "metadata takes priority over summary") |
| 5 | `detectRepeatRollback` 가 head.version != currentVersion 에서 false 반환 | ✅ |
| 6 | `ExtractionSchemaVersionResponse.from_domain` 이 bool 을 거절 | ✅ (pytest "test_bool_value_rejected") |
| 7 | `from_domain` 이 0 / 음수 / 문자열 / 리스트를 거절 | ✅ |
| 8 | `from_domain` 이 `extra_metadata` 비-dict 값에서 예외 없이 None 반환 | ✅ |
| 9 | `ExtractionSchemaVersionResponse` 가 `extra_metadata` 딕셔너리를 누설하지 않음 | ✅ (pytest "test_other_metadata_keys_ignored") |
| 10 | `VersionDiffModal` queryKey 에 `schema_id` 포함 | ✅ |
| 11 | `VersionDiffModal` staleTime === Infinity | ✅ |
| 12 | 스키마 update/rollback mutation 의 `invalidateKeys` prefix 가 diff 쿼리를 무효화 | ✅ (기존 `["admin", "extraction-schemas"]` prefix 매칭) |
| 13 | `RollbackDialog` 에 `recentVersions` 전달 시 repeatHint 계산 | ✅ |
| 14 | 반복 rollback 감지 시 확인 버튼 disabled + title 메시지 | ✅ |
| 15 | 반복 rollback 체크박스가 해제 상태로 시작, 성공 시 리셋 | ✅ |
| 16 | 반복 rollback alert 이 `role="alert"` 로 선언 | ✅ |
| 17 | P5-1 배너가 `role="status"` 로 선언 (정보 수준) | ✅ |
| 18 | `diffMismatch.ts` 가 React 의존성 없이 import 가능 | ✅ (node:test 실행 성공이 증거) |
| 19 | `AdminExtractionSchemasPage.tsx` 의 중복 인라인 유틸 제거 | ✅ (이번 세션 최종 정리) |
| 20 | tsc `--noEmit` 신규 에러 0 (`AdminUserDetailPage:279` 기존 1건 유지) | ✅ |

## 5. 검증 결과

- **Python AST**
  - `backend/app/schemas/extraction.py`: OK
  - `backend/tests/unit/extraction/test_extraction_schema_version_response.py`: OK
- **TypeScript**
  - `cd frontend && npx tsc --noEmit` — 신규 에러 0건. 기존 `AdminUserDetailPage.tsx:279` 1건 유지(P4 이전부터 있던 이슈, 본 FG 범위 외).
  - `cd frontend && npx tsc -p tsconfig.test.json` — OK.
- **node:test**
  - `node --test dist-tests/tests/ExtractionSchemasP5.test.js` → **15/15 pass**.
- **pytest**
  - 샌드박스 환경의 venv 가 다른 경로로 고정된 shebang(`/Users/ckchoi/...`) 을 가지고 있어 샌드박스에서 직접 실행 불가. 로컬 실행 권장:
    `cd backend && pytest tests/unit/extraction/test_extraction_schema_version_response.py -v`
  - AST parse OK, 기존 pytest 컨벤션(fixture 미사용, in-memory `SimpleNamespace`) 준수.

## 6. S2 원칙 준수 확인

- **S1 ①~④**: DocumentType 하드코딩 없음. generic + config 기반 유지.
- **S2 ⑤ Scope Profile**: `VersionDiffModal` 이 `scopeProfileId` 를 queryKey/API 인자로 지속 전달. 본 P5 는 scope 로직을 건드리지 않음(회귀 없음).
- **S2 ⑥ API 우선 + actor_type**: 본 P5 는 기존 API 표면을 확장(`rolled_back_from_version` 필드 추가)만 함. actor_type 감사 계약은 P4 에서 확립, 회귀 없음.
- **S2 ⑦ 폐쇄망**: 외부 의존 추가 없음. 모든 감지 로직은 클라이언트 순수 함수 + 서버 DTO 필드 노출로 완결.
- **우선순위**: S1 > S2 — P5 에서 충돌 없음.

## 7. 잔존 한계 / 향후 과제

- 감지 UX 는 "직전 1개 버전" 만 본다. 더 과거의 반복(v3→v2→v3→v2) 같은 핑퐁 패턴은
  amplification 축적을 보이지 않는 이상 감지하지 않는다 — 의도된 스코프.
- `change_summary` 패턴 폴백(`"v{N} 로 되돌리기"`) 은 default 한국어 문자열에
  한정. 사용자가 모든 롤백에 custom summary 를 쓰면 metadata 경로만으로 동작.
  이는 DTO 1급 필드가 있으므로 향후 서버가 해당 필드를 채우는 한 항상 정확.
- `VersionDiffModal` 의 `staleTime: Infinity` 는 브라우저 탭이 열려 있는 동안의
  메모리 캐시만 의미함 — 탭 닫히면 초기화. 장기 보관용이 아니므로 메모리 누수
  우려 없음.

## 8. 파일 변경 요약

| 영역 | 파일 | 변경 유형 |
|------|------|-----------|
| Backend | `app/schemas/extraction.py` | `ExtractionSchemaVersionResponse` 에 `rolled_back_from_version` 필드 + `from_domain` 검증 로직 추가 |
| Backend tests | `tests/unit/extraction/test_extraction_schema_version_response.py` | 신규 13건 |
| Frontend | `src/types/s2admin.ts` | `ExtractionSchemaVersion` 에 `rolled_back_from_version?: number \| null` 추가 |
| Frontend | `src/features/admin/extraction-schemas/diffMismatch.ts` | **신규** — 순수 로직 유틸(`diffMismatchSummary`, `detectRepeatRollback`) 및 공통 타입 |
| Frontend | `src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx` | P5-1/P5-2/P5-3 UI 통합, `./diffMismatch` 에서 재export, `VersionDiffModal` queryKey 에 `schema_id` 추가, `RollbackDialog` 에 `recentVersions` 전달, 중복 인라인 유틸 제거 |
| Frontend tests | `tests/ExtractionSchemasP5.test.tsx` | 신규 15건 |
| Frontend config | `tsconfig.test.json` | `diffMismatch.ts`, `types/s2admin.ts` 를 `include` 에 추가 |
| Docs | `docs/개발문서/S2_5/UI_Admin_ExtractionSchemas_P5_검수보고서.md` | 본 문서 |
| Docs | `docs/개발문서/S2_5/UI_Admin_ExtractionSchemas_P5_보안취약점검사보고서.md` | 동반 보안 보고서 |
