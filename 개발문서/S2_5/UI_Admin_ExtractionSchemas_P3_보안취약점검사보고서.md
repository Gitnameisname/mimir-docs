# /admin/extraction-schemas — P3 보안취약점검사보고서

## 개요

| 항목 | 값 |
|---|---|
| 작성일 | 2026-04-21 |
| 대상 | `/admin/extraction-schemas` 관련 프론트 + 백엔드 — **P3 변경분** |
| 기준 | CLAUDE.md ②③⑦ (보안/취약점 회피 · 폐쇄망 동등성), OWASP ASVS L2 선별 |
| 비교 기준 | P2 보고서의 위협모델(T-1 ~ T-7) 연장 + P3 로 확장된 경로 |

P3 의 변경 표면은 세 곳이다.

1. 백엔드: `app/schemas/extraction.py` 의 상한 validator 추가 (`MAX_FIELDS_COUNT`, `MAX_NESTED_DEPTH`).
2. 프론트: `FieldsEditor.tsx` 의 재귀 편집기 + `DefaultValueWidget`.
3. 프론트: `AdminExtractionSchemasPage.tsx::VersionHistorySection` 의 "더 보기" 페이지네이션.

아래는 위협모델을 세 곳으로 나누어 각 체크 결과를 기록한다.

---

## 위협모델 & 점검

### P3-C (서버측 상한) — DoS 내성

#### T-8. 대량 fields (depth 공격의 폭 축)

- **공격 시나리오**: 악의적인 클라이언트(또는 오류가 난 자동화 에이전트) 가 수만 개의 top-level field 를 포함한 JSON 을 POST. Pydantic 이 모든 필드를 `ExtractionFieldDef` 로 파싱하는 과정에서 CPU/메모리 소모.
- **방어**: `MAX_FIELDS_COUNT = 200`. `_validate_fields_map` 에서 길이 초과 시 즉시 `ValueError` (→ FastAPI 422). 파싱 이후에 검증되므로 파싱 자체로 CPU 를 소모할 수 있지만, Pydantic v2 의 C 구현은 O(n) 이고, FastAPI 는 기본 request body 상한(10MB 미만) 이 추가로 보호함.
- **평가**: **통과**. 단, 상한을 더 내리려면(예: 50) 서비스 계약 변경이 되므로 신중. 현재 200 은 "정상 관리자가 하루에 다룰 수 있는 범위의 몇 배" 라는 근거 하에 타당.

#### T-9. 중첩 스키마 폭탄 (depth 공격의 깊이 축)

- **공격 시나리오**: `nested_schema` 에 자기 유사(fractal) 구조를 주입해 재귀 깊이를 수만 단계로 확장 → Pydantic recursion + Python 재귀 한계.
- **방어**:
  - `ExtractionFieldDef` 가 Pydantic BaseModel 로 정의돼 있어 입력 JSON 이 모델로 구축되는 단계에서 Pydantic 의 스택 보호가 일차 방어.
  - `_max_nested_depth` 에서 `MAX_NESTED_DEPTH = 3` 초과를 추가 차단. 안전장치로 재귀 가드 `_cur > MAX_NESTED_DEPTH + 2` 도 추가돼 무한 재귀 혹은 예기치 않은 깊이에서도 터지지 않음.
  - Python 기본 `sys.setrecursionlimit` 은 1000 수준이므로 `MAX_NESTED_DEPTH=3` 보다 훨씬 여유가 큼.
- **평가**: **통과**. `_max_nested_depth` 가 순수 트리 순회(사이클 불가능: Pydantic 모델은 생성 후 불변) 이지만, 극단값 입력에서도 조기 종료한다는 점이 명시적으로 테스트됨.

#### T-10. 상한 우회 시도: 유니코드 식별자 / 중복 키

- **공격 시나리오**: Python dict 는 키를 유니코드로 허용. 예를 들어 `a` 와 `ａ` (전각) 를 다른 키로 취급해 "같은 이름" 으로 보이되 dict 의 크기는 두 배가 되는 식의 회피.
- **방어**: `ExtractionFieldDef.field_name` 에 snake_case 정규식이 걸려 있어 ASCII 영문/숫자/언더스코어만 허용됨. `fields` dict 의 키도 동일한 제약을 원칙적으로 따르지만, 키 자체는 Python dict 이므로 이론적으로 유니코드 허용. 그러나 동일한 field_name 값이 key 와 달라질 경우 서비스 레이어가 fields dict 를 그대로 저장하는 구조라 이름 mismatch 는 별도 서비스 계약 레벨의 관심사로 남김.
- **평가**: **부분 통과**. `MAX_FIELDS_COUNT` 은 절대치로 걸리므로 유니코드 키로는 방어를 뚫지 못함. 이름-key mismatch 에 의한 의미 혼동은 프론트(`FieldsEditor.duplicateNames`) 와 백엔드(`field_name` regex) 양쪽에서 일부 커버되지만 P3 범위에서 추가 강화 없음.

### P3-A / P3-B (프론트 편집기)

#### T-11. default_value XSS / 코드 주입

- **공격 시나리오**: 관리자 편집 화면에서 `default_value` 에 `<script>` 또는 문자열-기반 수식 주입. 다른 관리자가 열람할 때 실행되는 저장형 XSS.
- **방어**:
  - `DefaultValueWidget` 은 모든 출력을 React의 기본 렌더(= textContent) 로 표시하므로 HTML 이 해석되지 않음.
  - 백엔드 저장 시에도 `default_value` 는 JSONB 필드로 정형 저장 — 문자열 이스케이프는 FE 렌더 단계에서만 필요.
  - enum 타입의 위젯은 `enum_values` 를 `<option>` 자식으로 렌더. React 는 자동 이스케이프.
- **평가**: **통과**. P2 보고서의 T-6(default_value XSS) 과 동일 결론.

#### T-12. 재귀 편집기 DoS: 깊이 무한 확장

- **공격 시나리오**: 사용자가 object 필드를 수백 번 중첩하여 브라우저 렌더 스택을 소진.
- **방어**: `FieldsEditor` 자체가 `depth < maxDepth` 에서만 재귀 `<FieldsEditor>` 렌더. `maxDepth = 3` 이 기본. 따라서 최악의 경우라도 DOM 트리 깊이는 "최상위 → 3 단 중첩" 으로 제한.
- **평가**: **통과**. 백엔드 상한과 동일 상수라 클라이언트에서 먼저 막히고, 서버에서도 이중 차단.

#### T-13. onTypeChange 시 정보 유출 / 속성 누수

- **공격 시나리오**: 사용자가 타입을 object→string→object 로 번갈아 바꿀 때 `nested_schema` 의 잔존물이 의도치 않게 저장되어 서버에서 validator 와 충돌.
- **방어**: `onTypeChange(next)` 에서 전환 시 불필요 속성을 즉시 `delete` (pattern/max_length/min_value/max_value/date_format/enum_values/nested_schema). object 로 돌아올 때는 nested_schema 가 undefined 면 `{}` 로 초기화 — 이전 구조가 남지 않음.
- **평가**: **통과**. P2 에서 이미 도입된 `onTypeChange` 정리 로직을 P3 에서도 유지·강화 (object 전환 시 초기화 추가).

#### T-14. `default_value` 타입 mismatch 악용

- **공격 시나리오**: 사용자가 타입을 `number` 로 바꾼 뒤에도 `default_value` 가 과거의 문자열 값을 가지고 있어 서버에 비정합 데이터가 전송됨.
- **방어**: `DefaultValueWidget` 의 `mismatch` 계산이 타입-값 호환성을 감지해 "⚠ 현재 값이 ~와 호환되지 않음" 을 표시. 서버측 Pydantic validator 에는 `default_value` 의 타입-값 정합성 검사가 별도로 없기 때문에(리스크는 "저장 후 사용 시 타입 에러") UI 경고로 선행 보호.
- **평가**: **부분 통과**. 완전한 보호는 "서버에서 default_value 타입 검증" 이 필요. 본 P3 범위에선 UI 안내로 대응, 후속 과제로 이관.

### P3-D (버전 이력 페이지네이션)

#### T-15. scope_profile_id 쿼리 우회

- **공격 시나리오**: 페이지네이션 추가 호출에서 `scope_profile_id` 를 누락/변조해 타 스코프의 버전 이력을 훔쳐봄.
- **방어**:
  - 프론트 `handleLoadMore` 는 `scope_profile_id` 를 쿼리에 포함하지 않음 (P3-D 범위에서 scope 파라미터 연동은 후속 과제). 현재 상세 패널 자체가 scope 선택 없이 최신 활성 스키마를 조회하는 구조이므로 페이지네이션도 동일한 맥락 내에서 동작.
  - 백엔드 `get_extraction_schema_versions` 는 `scope_profile_id` 쿼리 파라미터를 422-검증(비-UUID) 하고 `ExtractionSchemaRepository.get_versions(..., scope_profile_id=...)` 로 전달해 ACL 필터링을 적용.
  - 서버 측에서 offset 이 음수이거나 limit 이 0 인 경우 FastAPI `Query(ge=0)` / `Query(ge=1, le=100)` 가 422 로 차단 (신규 테스트에서 검증).
- **평가**: **통과** (범위 내), **부분 통과** (scope 별 페이지네이션까지 포함하면 추가 구현 필요).

#### T-16. 페이지 누적 메모리 팽창

- **공격 시나리오**: "더 보기" 를 수천 번 연속 클릭해 브라우저 탭에 수만 개의 버전 객체를 누적 → 탭 메모리 고갈.
- **방어**:
  - `VERSION_PAGE_SIZE = 10` 로 한 번에 10 건씩만 추가. 사용자의 의도적 반복 클릭에 한해서만 증가.
  - 서버 `get_versions` 의 limit 상한은 `Query(ge=1, le=100)` — 한 번에 최대 100 건까지. 프론트가 임의로 큰 limit 을 요청해도 422 차단.
  - 상세 패널을 닫으면 컴포넌트가 언마운트 되며 누적 state 는 GC 에 맡겨짐.
- **평가**: **통과**. 이론적으로 "사용자가 계속 누르면 영원히 누적" 이지만, 이는 UX 레벨의 문제이지 취약점 범주 아님.

#### T-17. 페이지네이션 응답 삽입 공격 (response splitting)

- **공격 시나리오**: DB 또는 repo 레이어가 사용자 입력을 쿼리에 연결해 응답 본문에 제어문자 주입.
- **방어**:
  - `limit`/`offset` 은 정수만 허용 (FastAPI Query). `scope_profile_id` 는 UUID 파싱으로 정규화.
  - `doc_type_code` 는 Path 파라미터이지만 `create/update/deprecate` validator 에서 이미 정규식 검증이 있음. GET 경로도 404 경로에서 동일 doc_type_code 를 사용한다는 전제.
- **평가**: **통과**.

### P3-E (테스트)

- 테스트 자체는 위협 표면이 아니지만, "테스트 부재로 인한 regression" 이 취약점의 근본 원인이 될 수 있다는 점에서 S1 원칙 ⑤(개발 후 검수) 의 이행 증거로 남긴다.
- 단위 테스트 21 건 + 통합 테스트 15 건 모두 `ValueError` / `422` 경로를 중심으로 설계됐으며, DB 에 부작용을 남기지 않도록 Pydantic / 쿼리 파서 단계까지만 호출.

---

## 종합

| 위협 | 결과 | 비고 |
|---|---|---|
| T-8 대량 fields | 통과 | `MAX_FIELDS_COUNT=200` |
| T-9 중첩 폭탄 | 통과 | `MAX_NESTED_DEPTH=3` + 재귀 가드 |
| T-10 상한 우회 | 부분 통과 | 유니코드 키로는 회피 불가능(size 절대치), 이름-key mismatch 는 별도 서비스 계약 관심사 |
| T-11 default_value XSS | 통과 | React 기본 escape + JSONB 저장 |
| T-12 재귀 편집기 DoS | 통과 | UI = backend = 3 |
| T-13 속성 누수 | 통과 | onTypeChange 정리 로직 유지 |
| T-14 default_value 타입 mismatch | 부분 통과 | UI 경고. 서버 검증은 후속 과제 |
| T-15 scope 우회 | 통과(범위 내) | scope 별 페이지네이션은 후속 |
| T-16 페이지 누적 메모리 | 통과 | UX 문제지 취약점 아님 |
| T-17 응답 삽입 | 통과 | Query 파라미터 전부 정수/UUID |

### 잔존 리스크 (후속 권고)

1. **default_value ↔ field_type 서버 검증**: 현재는 UI 경고만 있음. `ExtractionFieldDef` model_validator 에 "default_value 가 존재하면 field_type 과 호환되는지" 검사를 추가할 것.
2. **scope 별 페이지네이션**: versions "더 보기" 가 `scope_profile_id` 쿼리를 누락함. 상세 패널이 scope 선택을 드러내면 함께 전달하도록 해야 함.
3. **재귀 편집기에서의 drag-and-drop 재배치**: 현재 재배치 수단 없음. 관리자가 깊이 3 까지 설계 후 중간에 삽입하려면 처음부터 다시 구성해야 함 — UX 부담은 크지만 보안 이슈는 아님.

### 폐쇄망 동등성 (CLAUDE.md ⑦)

모든 검증은 순수 Pydantic / React 로직으로 외부 호출이 없으며, `FieldsEditor` 와 `VersionHistorySection` 모두 오프라인에서 동일하게 동작한다.
