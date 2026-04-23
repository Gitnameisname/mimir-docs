# 추출 스키마 관리 화면 — P0 개선 검수보고서

- 대상 경로: `/admin/extraction-schemas`
- 작성일: 2026-04-21
- 범위: 백엔드 CRUD + 프론트 API 클라이언트/타입/렌더링까지 이어지는 "목록이 실데이터로 연결되지 않는" 결함 4건 (P0)

## 1. 배경 — 왜 P0 였는가

재검수 중 추출 스키마 관리 페이지(`AdminExtractionSchemasPage`)가 화면상 정상으로 보이지만
실제로는 **어떤 API 호출도 성공하지 못하고, MOCK 데이터로 조용히 폴백**되는 상태가 확인됐다.
원인은 4개의 독립 결함이 동일 경로에 직렬로 쌓여 있었기 때문이다.

| # | 결함 | 증상 | 영향 |
| - | ---- | ---- | ---- |
| ① | 프론트 URL `/api/v1/admin/extraction-schemas` — 백엔드는 `/admin/` 없이 마운트 | 목록 조회 404 | 화면은 MOCK 으로 조용히 폴백 — 사용자가 "잘 되는 줄" 착각 |
| ② | `emit_for_actor(...)` 4곳에서 `action=` kwarg 누락 (F-07 regression) | 모든 쓰기 경로 500 (TypeError) | 생성/수정/삭제/폐기 전부 터짐 + 감사 로그 누수 |
| ③ | 프론트 `ExtractionSchema` 타입이 백엔드 `ExtractionSchemaResponse` 와 완전 불일치 | 실데이터로 바꾸는 순간 모든 필드 참조가 undefined | 실 API 연결 시 즉시 화면 붕괴 |
| ④ | `extraction_mode` 가 프론트에만 존재 (하드코딩) | 백엔드·DB 어디에도 없는 필드 | S1 원칙 ① 위반 (문서 타입 메타 하드코딩) |

추가로 라우터/리포지토리 레벨의 동반 결함 2건(③, ④ 실데이터 연결 시 반드시 드러남):

| 부가 결함 | 설명 |
| -------- | ---- |
| a | `get_by_doc_type(include_deprecated=…)` 호출 측 존재 vs 리포지토리 시그니처에 미정의 → `TypeError` |
| b | 라우터 `except ValueError` 가 `ExtractionSchemaAlreadyExistsError(Exception)` 를 잡지 못해 **409 대신 500** |
| c | 라우터 `except KeyError` 가 `ExtractionSchemaNotFoundError(Exception)` 를 잡지 못해 **404 대신 500** |

## 2. 조치 요약

### 2.1 백엔드

`backend/app/api/v1/extraction_schemas.py`

- **GET `/extraction-schemas` (목록 조회) 신규 추가** — `list_all()` repository 메서드를 활용.
  `is_deprecated`, `include_deleted`, `scope_profile_id`, `limit`, `offset` 쿼리 파라미터 지원.
  `list_response` helper 로 `PagedResponse` 포장 (페이지 정보 포함).
- **4 × `emit_for_actor` 에 `action=` kwarg 추가** — F-07 (2026-04-18 규약) 준수:
    - `create` → `action="extraction_schema.create"`
    - `update` → `action="extraction_schema.update"`
    - `delete` → `action="extraction_schema.delete"`
    - `deprecate` → `action="extraction_schema.deprecate"`
- **예외 매핑 정정**:
    - `create` : `ValueError → 409` ⇒ `ExtractionSchemaAlreadyExistsError → 409`
      (기존 `ValueError` 는 실제로 발생하지 않으므로 `422` 로 fallback 재정의)
    - `update`, `deprecate` : `KeyError → 404` ⇒ `ExtractionSchemaNotFoundError → 404`
- `ExtractionSchemaAlreadyExistsError`, `ExtractionSchemaNotFoundError` import 추가.

`backend/app/repositories/extraction_schema_repository.py`

- `get_by_doc_type(include_deprecated: bool = False, scope_profile_id: Optional[UUID] = None)`
  파라미터 추가.
    - `include_deprecated=False` 가 기본값이라 **폐기된 스키마는 최신이어도 자동 제외**됨.
    - `scope_profile_id` 로 필터링해 S2 원칙 ⑥ (Scope Profile ACL) 슬롯 강화.

### 2.2 프론트엔드

`frontend/src/lib/api/s2admin.ts`

- `extractionSchemasApi` 를 전면 재설계. 기존 `list/get` 2개에서
  `list / get / getVersions / create / update / deprecate / delete` 7개로 확장.
- URL prefix `/api/v1/admin/` ⇒ `/api/v1/` (백엔드 실제 마운트와 정합).
- `list` 는 `PagedResponse<ExtractionSchema>` 로 타입 갱신.
- `ExtractionSchemaVersion` import 추가.

`frontend/src/types/s2admin.ts`

- `ExtractionFieldType` 열거 신설 (백엔드 `FieldType` 과 1:1 정합):
  `string / number / date / boolean / array / object / enum`.
- `ExtractionSchemaField` 를 `field_name / field_type / required / description / pattern /
  instruction / examples / max_length / min_value / max_value / date_format / enum_values /
  default_value / nested_schema` 로 재정의 (백엔드 `ExtractionFieldDef` 와 정합).
- `ExtractionSchema` 를 `id / doc_type_code / version / fields: Record<…> / is_deprecated /
  deprecation_reason / created_at / updated_at / created_by / updated_by / scope_profile_id /
  extra_metadata` 로 재정의 (백엔드 `ExtractionSchemaResponse` 와 정합).
- `ExtractionSchemaVersion` 신설 (버전 이력 응답).
- **`extraction_mode` 완전 제거** — 백엔드·DB 어디에도 없는 가상 필드였고 S1 원칙 ① 위반.

`frontend/src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx`

- MOCK 데이터(`MOCK_SCHEMAS`, `MOCK_FIELDS`) 전부 제거 — 실 API 오류를 조용히 숨기던 코드.
- `SkeletonBanner` 제거 (Phase 8 대기 상태 아님, 실제 API 가 살아있음).
- `ErrorBanner` 도입 — `NetworkError` / `ApiError` 분기 (403/404/일반 오류) 로 진단 메시지 차별화.
- **폐기된 스키마 포함** 체크박스 추가 — 기본 off (is_deprecated=false 필터).
- 상세 패널에 버전·Scope Profile·생성자·수정자·생성/수정일 메타 표시 (dl/dt/dd 시맨틱).
- 상세 패널 접근성 보완: 백드롭 클릭 닫기, Esc 키 닫기, `onClick` 버블링 차단.
- 테이블 컬럼을 **DocumentType / 버전 / 정의된 필드 / 상태(활성·폐기됨) / 최종 수정일 / 작업**
  으로 재구성. `scope="col"` 헤더 속성, aria-label 부여.
- 로딩/빈 상태 셀을 행 대체(`colSpan=6`)로 표현.
- `fieldsCount` 는 `Object.keys(s.fields).length` 로 계산 (백엔드가 Dict 로 보내므로).

## 3. 재현 전/후 흐름

**Before**:
1. 페이지 진입 → `GET /api/v1/admin/extraction-schemas` 404
2. `useQuery` 가 `isError=true` 반환 → `MOCK_SCHEMAS` 로 폴백 → 화면은 "정상처럼" 보임
3. 사용자가 상세/편집을 시도해도 상세 API 도 404 → MOCK_FIELDS 폴백
4. (가상 시나리오) 만약 URL 을 고쳐도, 프론트 `document_type_code` 참조가 백엔드 응답의 `doc_type_code` 와 달라 `undefined` 표시
5. (가상 시나리오) 쓰기 API 호출 성공 경로에서도 `audit_emitter.emit_for_actor(...)` 가 action= 누락으로 500 → 감사 로그 남지 않음 + 사용자에게 오류

**After**:
1. `GET /api/v1/extraction-schemas?is_deprecated=false` 200 → 실제 스키마 목록 렌더.
2. 상세 패널 열면 단건 조회 200, `fields` Dict 가 테이블로 표시됨.
3. 쓰기 경로 호출 시 F-07 감사 로그가 `action` 포함 상태로 정상 기록.

## 4. 회귀 영향

| 영역 | 회귀 가능성 | 확인 방법 |
| ---- | ----------- | --------- |
| 다른 admin 페이지 | 없음 | `extractionSchemasApi` 는 이 페이지 단독 소비자 (전수 grep 확인) |
| 기존 `ExtractionSchema` 타입 참조 | 해당 페이지 외 없음 | `grep "ExtractionSchema\\b"` — 프론트 전체에서 이 파일 + 타입 정의만 매칭 |
| `extraction_mode` 참조 | 없음 | `types/extraction.ts` 의 `ExtractionMode` 는 다른 도메인(ExtractionCandidate) 이며 미건드림 |
| `get_by_doc_type` 호출 측 | 없음 (하위 호환) | 기존 kwarg(`scope_profile_id`) 유지 + `include_deprecated=False` 가 기존 동작과 동일 |
| 라우터 예외 매핑 | **개선** | 기존 500 로 돌려주던 NotFound/AlreadyExists 가 정확한 404/409 로 전달됨 |

## 5. 검증

- Python `ast.parse` : 3개 파일(`extraction_schemas.py` 라우터·리포지토리, `schemas/extraction.py`) 모두 OK.
- Router 데코레이터 구조 검사 : 7개 엔드포인트(`GET/POST /` + `GET/PUT/DELETE /{doc}` + `GET /{doc}/versions` + `PATCH /{doc}/deprecate`) 모두 올바른 순서·HTTP 메서드로 배치됨.
- `emit_for_actor` 4건 모두 `action=` 포함 검증 (AST 방문자).
- Frontend `tsc --noEmit` : 추출 스키마 관련 파일에서 타입 에러 0건 (기존 `AdminUserDetailPage.tsx:279` 무관 에러 1건 확인, 본 PR 범위 밖).

## 6. 잔존 / 후속 (P1 이하)

- 현재 페이지는 **조회 전용**. 생성/수정/폐기/삭제 UI 위젯은 미구현 (API 클라이언트만 준비 완료). → P1
- 프론트에서 `ExtractionFieldDef` 의 `pattern / instruction / examples / max_length / enum_values`
  등 세부 속성을 표시하지 않음. → P2
- `GET /extraction-schemas` 는 `list_all` 를 통해 `DISTINCT ON (doc_type_code)` 로 가져오지만
  `total` 을 별도 `COUNT(*)` 쿼리로 계산하지 않고 현재 페이지의 길이를 쓴다. 페이지네이션 정확도가
  필요해지면 별도 total 쿼리를 추가해야 한다. → P2
- Scope Profile 가 지정된 스키마와 미지정(전역) 스키마가 섞여 있을 때 UI 에서 구분 배지가 없음. → P3
