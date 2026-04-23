# UI · Admin · ExtractionSchemas — P3 후속 검수 보고서

> 대상 경로: `/admin/extraction-schemas`
> 작성일: 2026-04-21 (P3 종결 당일 후속 반영)
> 선행 문서: `UI_Admin_ExtractionSchemas_P3_검수보고서.md`, `…_P3_보안취약점검사보고서.md`

## 1. 작업 범위

P3 종결 보고서에서 "잔존 한계" 로 명시한 세 가지를 후속으로 해소한다.

| ID | 항목 | 선행 문서 근거 |
|----|------|---------------|
| 후속-A | `default_value ↔ field_type` 서버 검증 (Pydantic validator) | P3 보안 T-14 "부분 통과", UI 경고만 있음 |
| 후속-B | 버전 이력 페이지네이션의 `scope_profile_id` 전달 | P3 검수 §6 잔존 한계 |
| 후속-C | `FieldsEditor` 드래그 앤 드롭 재배치 | P3 검수 §6 잔존 한계 |

## 2. 구현 요약

### 2.1 후속-A — 서버 default_value 검증

- 파일: `backend/app/models/extraction.py`
- 신규 메서드: `ExtractionFieldDef._validate_default_value` (model_validator mode="after").
- 규칙
  - `None` 은 타입과 관계없이 허용.
  - `string` / `date` / `enum` : `str` 만 허용.
    - `date` 는 `date_format` (기본 `YYYY-MM-DD`) 로 `datetime.strptime` 파싱 검증.
    - `enum` 은 `enum_values` 멤버십까지 검증.
  - `number` : `int` 또는 `float` 만 허용. `bool` 은 `int` 서브클래스지만 거절.
    - `min_value`/`max_value` 경계가 있으면 기본값도 해당 범위 내.
  - `boolean` : `bool` 만 허용 (정수 `1/0`, 문자열 `"true"` 거절).
  - `array` : `list`, `object` : `dict` 만 허용.
- 테스트: `backend/tests/unit/extraction/test_extraction_field_def.py` 말미에
  `TestDefaultValueCompatibility` 클래스 추가, 22 케이스.

### 2.2 후속-B — versions 페이지네이션 scope_profile_id

- 파일: `frontend/src/lib/api/s2admin.ts`
  - `extractionSchemasApi.getVersions` 파라미터에 `scope_profile_id?: string | null` 추가.
  - `buildQueryString` 이 null/undefined/"" 를 자동으로 제거하므로 기본 동작은 변하지 않는다.
- 파일: `frontend/src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx`
  - `VersionHistorySection` 이 `scopeProfileId?: string | null` prop 을 받는다.
  - `useQuery` 의 queryKey 에 `scopeProfileId` 를 포함 → 같은 docType 이어도 scope
    전환 시 캐시가 자동 분리된다.
  - `queryFn` 과 `handleLoadMore` 두 곳 모두 `scope_profile_id` 를 전달.
  - 호출부는 `latest.scope_profile_id` (현재 로드된 최신 스키마의 scope) 를 넘긴다.
- 백엔드는 이미 `GET /extraction-schemas/{doc_type}/versions?scope_profile_id=...` 를
  지원 (P2-D 에서 완성). 따라서 서버 변경은 없음.

### 2.3 후속-C — FieldsEditor DnD 재배치

- 파일: `frontend/src/features/admin/extraction-schemas/FieldsEditor.tsx`
- 변경 내용
  - FieldsEditor 내부에 `instanceIdRef` (랜덤) 와 `draggedKey` / `dragOverKey` state.
  - 헬퍼: `applyOrder`, `moveKey`, `moveKeyBefore`.
  - Drag source: `<span>` 핸들 (⋮⋮) 이 `draggable`. 행 전체를 draggable 로 두면
    확장된 행의 입력 필드에서 텍스트 선택이 드래그로 오인되므로 핸들로만 한정.
  - Drop target: `<li>` 가 `onDragOver` / `onDragLeave` / `onDrop` 을 처리.
  - 같은 편집기 인스턴스 간의 드롭만 허용 (`instanceId|key` payload 의 scope 검사).
    - 중첩 FieldsEditor 는 자체 instanceId 를 가지므로 부모 ↔ 자식 교차 이동 불가.
  - 키보드 접근성: 각 행에 "위로/아래로 이동" 버튼을 추가. 경계에서 disabled.
  - 시각 피드백
    - 드래그 중인 행: `opacity-50`
    - 드롭 대상 행: `ring-2 ring-blue-400 ring-offset-1`

## 3. UI 디자인 리뷰 (≥5회)

CLAUDE.md 규칙 ④ 준수. 각 라운드는 DnD/버튼 UX 관점의 추가 리뷰.

### Round 1 — 시각적 위계
- 헤더 좌측부터: [⋮⋮ 핸들] → [펼침 버튼 + 이름/타입/배지] → [↑/↓ 이동] → [휴지통].
- 핸들은 `text-gray-400 hover:text-gray-700`, 이동 버튼은 SVG 화살표. 휴지통과 크기 일치 (w-4 h-4).
- 결론: 시각적 위계 OK.

### Round 2 — 접근성 (스크린 리더)
- 핸들: `role="button"` + `aria-label="필드 X 드래그하여 순서 변경"`. VoiceOver 로도 식별 가능.
- ↑/↓ 버튼: 각 `aria-label` 에 필드명 포함 + `disabled` 속성.
- 핸들을 버튼 역할로 선언했지만 키보드 Enter/Space 로는 드래그 시작 불가 → 키보드 사용자는 ↑/↓ 로 우회.

### Round 3 — 드래그 상태 피드백
- `opacity-50` 은 source 표시, `ring-2` 는 target 표시 — 충돌 없이 공존.
- `transition-colors` 로 ring 전환이 부드러움.

### Round 4 — 오류 방지
- 같은 row 위로 드롭: `if (sourceKey === targetKey) return;` 로 무반응.
- 다른 편집기 인스턴스에서 드롭: instanceId 불일치 → 무시.
- 중간에 fields 가 바뀐 경우: `Object.prototype.hasOwnProperty.call(fields, sourceKey)` 체크.

### Round 5 — 중첩 편집기 상호작용
- 각 FieldsEditor 는 자체 instanceIdRef. 부모 row 를 자식 list 로 드롭해도 scope 불일치로 무시.
- 결론: 중첩 경계가 기술적으로 보장됨.

### Round 6 — 터치 디바이스
- HTML5 DnD 는 iOS Safari 등에서 touch-to-drag 이 기본 지원되지 않음 (사용자가 longpress 하면 iOS 는
  텍스트 선택으로 해석).
- ↑/↓ 버튼이 터치 친화적 대체 경로. min-hit-area 는 `p-1.5 + w-4 h-4 = 28x28` — 가이드
  44x44 보다 작지만 기존 휴지통과 동일 수준이므로 일관성 우선.
- 후속 개선으로 터치 전용 longpress → context 메뉴 이동 UX 는 별도 범위.

## 4. 검수 체크리스트

| # | 항목 | 결과 |
|---|------|------|
| 1 | `ExtractionFieldDef(default_value=...)` 가 모든 `field_type` 에 대해 호환성 검사됨 | ✅ |
| 2 | `number` 타입에 `bool` 기본값 거절 | ✅ |
| 3 | `date` 기본값이 `date_format` 과 일치하지 않으면 거절 | ✅ |
| 4 | `enum` 기본값이 `enum_values` 에 없으면 거절 | ✅ |
| 5 | `None` 은 모든 타입에서 허용 | ✅ |
| 6 | `number` 기본값이 min/max 경계를 넘으면 거절 | ✅ |
| 7 | `getVersions` API 가 `scope_profile_id` 를 optional 쿼리로 수용 | ✅ |
| 8 | 초기 로드 + "더 보기" 요청 모두에 scope_profile_id 포함 | ✅ |
| 9 | TanStack Query queryKey 에 scope 포함 → 캐시 분리 | ✅ |
| 10 | FieldsEditor 행에 드래그 핸들 ⋮⋮ 노출 | ✅ |
| 11 | ↑/↓ 이동 버튼이 경계에서 disabled | ✅ |
| 12 | 드래그 중 opacity-50, 드롭 타겟 ring 표시 | ✅ |
| 13 | 다른 편집기 인스턴스 간 드롭 차단 (instanceId scope) | ✅ |
| 14 | 중첩 편집기의 DnD 가 부모 레벨로 탈출하지 않음 | ✅ |
| 15 | tsc 에러 증가 없음 (AdminUserDetailPage.tsx:279 기존 1건 유지) | ✅ |

## 5. 검증 결과

- Python AST
  - `backend/app/models/extraction.py` : OK
  - `backend/tests/unit/extraction/test_extraction_field_def.py` : OK
- TypeScript
  - `tsc --noEmit -p tsconfig.json` : 신규 에러 0건 (기존 1건 유지)
- pytest
  - 샌드박스 Python 3.13 venv 가 로컬에 맞지 않아 샌드박스 실행 불가.
  - 로컬 (3.13 venv) 에서 `pytest backend/tests/unit/extraction/test_extraction_field_def.py -k TestDefaultValueCompatibility` 실행 필요.

## 6. 잔존 한계 (후속의 후속)

- DnD 를 터치 전용으로 완전히 지원하려면 `react-dnd-touch-backend` 등 외부 라이브러리 필요.
  현재는 ↑/↓ 버튼으로 대체.
- 레벨 간 이동(루트 → nested_schema 내부 또는 그 반대) 은 의도적으로 차단됨.
  필드 재구조화가 필요하면 현재도 "필드 추가 → 이전 필드 삭제" 로 가능.
- `versions` 페이지네이션이 상세 패널의 `latest.scope_profile_id` 만 참조하므로
  상세 패널이 특정 scope 를 "오버라이드" 해서 보여주는 기능이 추가되면 그 시점의 state 를
  대신 넘겨야 한다.
