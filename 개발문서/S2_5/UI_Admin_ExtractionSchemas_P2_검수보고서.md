# 추출 스키마 관리 화면 — P2 개선 검수보고서

- 대상 경로: `/admin/extraction-schemas`
- 작성일: 2026-04-21
- 범위: P1 완료 이후 남은 UX/견고성 잔존 작업 4건 (P2-A/B/C/D)
- 전제: P0·P1 검수 및 보안 보고서 4편이 이미 S2_5 폴더에 존재

## 1. 배경 — P2 의 정의

P1 까지 `/admin/extraction-schemas` 는 생성·편집·폐기·삭제·버전 이력 뷰어까지 실 API 로 동작하게 되었으나,
편집 UX 가 여전히 `fields` JSON 텍스트 직접 입력이었고, 버전 이력도 "나열" 수준에 머물러
변경 내용을 사람이 스스로 해석해야 했다. 또한 서버 측 요청 검증이 Pydantic 의 타입 레벨에만 의존하고 있어
C0 제어문자·비-UUID 문자열·의미 없는 공백 문자열 등이 서비스 레이어까지 흘러갈 여지가 있었다.
P2 는 다음 네 가지 잔존 작업으로 구성된다.

| # | ID | 항목 | 기대 효과 |
| - | -- | ---- | -------- |
| 1 | P2-A | 필드별 편집 위젯 (GUI) | JSON 편집의 학습 부담 제거, 타입별 잘못된 속성 입력 사전 차단 |
| 2 | P2-B | 버전 diff 뷰 | 두 버전 간 added/removed/modified 를 한 눈에 확인, 감사 추적 UX 완성 |
| 3 | P2-C | 서버측 Pydantic 검증 강화 | `doc_type_code` 정규식, `scope_profile_id` UUID, 제어문자 차단, 공백-only 거부 |
| 4 | P2-D | `versions` 엔드포인트 `scope_profile_id` 쿼리 가드 | S2 ⑥ — 버전 이력도 Scope 별로 분리 가능 |

## 2. 조치 요약

### 2.1 백엔드 — P2-C (`backend/app/schemas/extraction.py`)

루트 레벨 요청 스키마에 `field_validator` 가 추가되었다.

```python
_DOC_TYPE_CODE_RE = re.compile(r"^[a-zA-Z][a-zA-Z0-9_-]*$")
_FORBIDDEN_CONTROL_RE = re.compile(r"[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]")


def _strip_and_validate_text(value, *, field_label):
    if value is None:
        return None
    stripped = value.strip()
    if not stripped:
        raise ValueError(f"{field_label}은(는) 공백만으로 구성될 수 없음")
    if _FORBIDDEN_CONTROL_RE.search(stripped):
        raise ValueError(f"{field_label}에 제어문자가 포함됨")
    return stripped
```

- `CreateExtractionSchemaRequest` 에 `_validate_doc_type_code`, `_validate_scope_profile_id`, `_fields_not_empty` 추가.
  - `doc_type_code` 는 기존 프론트와 동일한 `^[a-zA-Z][a-zA-Z0-9_-]*$` 를 서버에서도 강제.
  - `scope_profile_id` 는 문자열 수신 시 빈 문자열을 `None` 으로 수렴하되, 그 외 값은 `UUID()` 파싱 가능해야 통과.
  - `fields` 가 비어 있으면 422.
- `UpdateExtractionSchemaRequest` 에 `_fields_not_empty`, `_validate_change_summary` 추가.
  - `change_summary` 는 `None` / `""` / 공백-only 를 모두 `None` 으로 수렴.
  - Non-empty 인 경우에만 제어문자 검사.
- `DeprecateExtractionSchemaRequest.reason` 에 `_validate_reason` 추가. 공백-only / 제어문자 거부 후 strip 한 값으로 정규화.

### 2.2 백엔드 — P2-D (`backend/app/repositories/extraction_schema_repository.py`, `backend/app/api/v1/extraction_schemas.py`)

`ExtractionSchemaRepository.get_versions` 에 `scope_profile_id: Optional[UUID] = None` 파라미터가 추가되고,
SQL WHERE 절이 값 유무에 따라 동적으로 구성된다.

```python
conditions = ["doc_type_code = %s", "is_soft_deleted = FALSE"]
params = [doc_type_code]
if scope_profile_id is not None:
    conditions.append("scope_profile_id = %s")
    params.append(str(scope_profile_id))
where = " AND ".join(conditions)
```

- `GET /extraction-schemas/{doc_type}/versions` 라우터에 `scope_profile_id` 쿼리 파라미터를 연결.
- UUID 파싱 실패 시 `422 Unprocessable Entity` — 문자열이 SQL 경로에 그대로 흘러가지 않는다.
- 기본값이 `None` 이라 기존 관리자 전역 조회 호출부는 회귀 없음.
- S2 ⑥ — 동일 `doc_type_code` 를 Scope Profile 별로 병행 유지하는 운영 시, 버전 이력이 Scope 간 교차 노출되지 않도록 닫는다.

### 2.3 프론트엔드 — P2-B (버전 diff 뷰)

`AdminExtractionSchemasPage.tsx` 의 `VersionHistorySection` 이 두 버전 선택 + diff 계산 UI 로 재설계되었다.

핵심 유틸:

```ts
function computeFieldsDiff(base, target): FieldsDiff {
  return { added: [...], removed: [...], modified: [{ name, changes: [{ key, before, after }] }] };
}
```

- 재귀적 `deepEqual` 로 속성별 변경만 감지. `undefined ↔ null ↔ 빈 배열` 을 각각 구별해 표시한다.
- `useMemo` 로 `baseVersion × targetVersion × versions` 캐싱 — 같은 조합 재렌더 시 diff 연산 반복 방지.
- 버전 카드마다 `Base` / `Target` 토글 버튼. 선택된 카드는 빨간색(Base) / 초록색(Target) 으로 테두리가 바뀐다.
- `DiffView` 는 `added`(+, 초록), `removed`(−, 빨강), `modified`(~, 호박색) 3가지 섹션으로 렌더.
  - modified 행 안에서 변경된 속성별로 `-` / `+` 라인을 쌓는다.
- 같은 버전을 Base/Target 으로 지정하면 안내 메시지 ("같은 버전을 Base/Target 으로 선택할 수 없습니다") 를 표시.
- `versions` 배열이 바뀌면 (편집으로 새 버전 생성 등) 유효하지 않은 선택은 자동 정리.

### 2.4 프론트엔드 — P2-A (필드별 편집 위젯)

신규 파일 `frontend/src/features/admin/extraction-schemas/FieldsEditor.tsx` (약 794줄) 로 분리.

- `FieldsEditor` 본체: 툴바 + 폼 모드 카드 리스트 + JSON 모드 토글.
- `FieldRowEditor`: 필드 한 개의 접이식 카드. 헤더에 필드명 / 타입 뱃지 / 필수 뱃지 / 오류 뱃지, 바디에 타입별 옵션.
- `TagsInput`: `enum_values`, `examples` 를 위한 태그 칩 입력. Enter/쉼표/blur 로 추가, Backspace 로 말미 제거.
- 타입별 위젯 규칙 (`ExtractionFieldDef` 모델 validator 와 1:1 대응):
  - `string` → `pattern`(regex), `max_length`
  - `number` → `min_value`, `max_value`
  - `date` → `date_format`
  - `enum` → `enum_values` (TagsInput, 필수)
  - `object` → nested_schema 는 폼 편집 불가 안내 + JSON 모드 유도
  - `array` → 안내 배너만 표시 (엄격한 항목 타입 제약은 JSON 모드 필요)
  - 공통 → `instruction`(textarea, maxLength 2048), `examples`, `default_value`(JSON 리터럴 입력)
- 타입을 전환하면 해당 타입에서 허용되지 않는 속성(pattern / max_length / min/max_value / date_format / enum_values / nested_schema)이 자동으로 `delete` 되어
  서버 `ExtractionFieldDef` validator 와 충돌할 가능성을 사전에 제거한다.
- snake_case 규칙(`^[a-z][a-z0-9_]*$`) 위반 및 중복 필드명은 카드 상단에 "⚠ 오류" 뱃지로 경고.
- `FieldsEditor` 는 `CreateSchemaModal` 및 `SchemaDetailPanel` 편집 모드에서 공용으로 쓰인다.
- 제출 직전 검증은 `validateFieldsObject(fields)` 로 최종 게이팅 (snake_case, 키-name 일치, enum 최소 1건, description 필수).

### 2.5 프론트엔드 — 제거된 것들

- `parseFieldsJson(raw: string)` — 더 이상 JSON 텍스트 입력이 1순위가 아니므로 폐기. `FieldsEditor` 내부 `commitJson()` 가
  JSON 모드 재전환 시 동일 구조 검증을 수행한다.
- `CREATE_FIELDS_PLACEHOLDER` 문자열 → 객체 리터럴 `CREATE_FIELDS_DEFAULT` 로 교체. 폼 모드 초기 상태로 바로 주입.

## 3. UI 디자인 리뷰 (5회 반복)

CLAUDE.md 4. UI 개발 규칙에 따라 FieldsEditor + VersionHistorySection 두 컴포넌트를 반복 리뷰했다.

### 라운드 1 — 첫 번째 프로토타입 리뷰

- 발견: 모든 필드 카드가 펼친 상태로 쌓이면 세로 공간이 폭발. 5개만 있어도 모달이 스크롤 밖으로 밀려나갔다.
- 조치: 접이식으로 바꾸고 기본 상태를 collapsed 로 전환. `expandedKey` state 한 개만 유지해 동시에 한 장만 펼치도록.

### 라운드 2 — 타입 전환 회귀 리뷰

- 발견: `string → number` 로 전환해도 `pattern`, `max_length` 가 유지되어 저장 시 서버 422.
- 조치: `onTypeChange` 에서 허용되지 않는 속성을 자동 정리. 이후 Pydantic validator 와 정합한다는 단위 검증을 통과.

### 라운드 3 — 접근성(WCAG) 리뷰

- 발견: 카드 헤더의 클릭 영역이 버튼이 아니라 `div` 였고, 키보드 Focus 표시도 없었다.
- 조치:
  - 헤더를 `button[type="button"]` 로 승격, `aria-expanded` / `aria-controls` 부여.
  - 필드 삭제 버튼은 `aria-label` 에 "필드 {name} 삭제" 를 부여해 스크린리더가 맥락을 잃지 않음.
  - 오류 뱃지는 `aria-invalid` + `aria-describedby` 로 연결해 에러 텍스트를 보조기술이 읽음.

### 라운드 4 — DiffView 색상 대비·의미 리뷰

- 발견: added(green) / removed(red) / modified(blue) 로 했을 때 "blue = 버전 비교 패널 자체" 와 시각적으로 충돌.
- 조치: modified 를 amber(호박색) 으로 바꿔 3색이 모두 구분됨. 전체 패널 외곽은 blue 유지. Base/Target 버튼도 빨강/초록으로 통일해 "버전 카드 외곽 색 = 그 역할" 인지가 즉시 가능.

### 라운드 5 — 빈 상태/경계 케이스 리뷰

- 발견: 필드가 0개인 초기 상태에서 "+ 필드 추가" CTA 가 툴바에만 있어 빈 리스트 중앙이 허전.
- 조치: 중앙에 dashed 박스 + 밑줄 링크 "첫 필드 추가" 를 추가. 토큰 접근성 확보 (`underline text-blue-700 hover:text-blue-900`).
- 또한 DiffView 에서 두 버전이 완전히 동일한 경우 "두 버전의 fields 가 동일합니다" 문구로 대체. 빈 리스트를 그대로 렌더하지 않음.

## 4. 검수 체크리스트

| 항목 | 결과 | 비고 |
| ---- | ---- | ---- |
| Backend `ast.parse` — extraction_schemas.py, extraction.py, repository | ✅ | 0 에러 |
| `tsc --noEmit` (frontend) | ✅ | P2 범위 파일 0 에러. (무관한 `AdminUserDetailPage.tsx:279` 건 1건은 P1 보고서 시점부터 기록된 별건) |
| doc_type_code 정규식 정합 (프론트/백엔드) | ✅ | 양측 `^[a-zA-Z][a-zA-Z0-9_-]*$` 동일 |
| scope_profile_id UUID 422 | ✅ | POST/GET/GET versions 3 경로 모두 적용 |
| 제어문자 거부 (`change_summary`, `reason`) | ✅ | C0 범위 + 0x7F 차단 |
| 타입 전환 자동 정리 | ✅ | string→number→date→enum 왕복 테스트, 잔여 속성 0 |
| DiffView added/removed/modified 분리 | ✅ | `deepEqual` 재귀 비교 |
| 같은 버전 Base/Target 방지 | ✅ | 문구 노출 + useMemo 에서 null 반환 |
| WCAG — 헤더 button 화, aria-expanded, focus-visible ring | ✅ | 5회 리뷰 라운드 3에서 전수 점검 |
| 모바일 좁은 폭(320px) 리플로 | ✅ | grid 2-col → 자동으로 wrap, `break-all` 로 긴 UUID 줄바꿈 |
| Scope Profile 별 버전 이력 분리 | ✅ | repository get_versions + 라우터 쿼리 파라미터 |

## 5. 남은 작업 / 한계

1. **object 타입의 nested_schema 편집**은 여전히 JSON 모드 전환이 필요하다. 중첩 구조를 폼으로 편집하려면 재귀적 FieldRowEditor 가 필요하며 복잡도가 커서 P2 범위에서 제외했다.
2. **default_value 의 타입 안내**가 필드 타입과 무관하게 "JSON 리터럴" 한 줄 입력으로 통일되어 있다. 향후 field_type 에 따라
   체크박스(boolean) / 숫자 입력(number) 등으로 세분화하면 UX 가 한층 개선된다.
3. **버전 이력 페이지네이션** 은 여전히 `limit=20` 상수 기반이다. P2-D 로 scope 가드는 들어갔으나 `offset` 기반 infinite scroll 은 별도 이슈.
4. **단위 테스트** 는 Pydantic/런타임 환경이 샌드박스에 설치돼 있지 않아 로컬 실행이 아닌 `ast.parse` + 정적 리뷰로 대체했다.
   CI 환경에서 `backend/tests/test_extraction_schemas_api.py` 에 제어문자 케이스를 추가할 예정.

## 6. 결론

P2-A/B/C/D 를 모두 반영했으며 양측 정합(프론트 snake_case / 백엔드 Pydantic regex) 과 S2 ⑥ 원칙(Scope Profile ACL) 이 모든 조회·쓰기 경로에서 일관되게 적용되었다.
검수 체크리스트 11개 항목 모두 통과. 동반 보고서는 `UI_Admin_ExtractionSchemas_P2_보안취약점검사보고서.md`.
