# UI · Admin · ExtractionSchemas — P3 후속 보안 취약점 검사 보고서

> 대상: 후속-A/B/C 변경분
> 작성일: 2026-04-21
> 선행 문서: `UI_Admin_ExtractionSchemas_P3_보안취약점검사보고서.md`
>   - 선행 문서에서 다룬 위협 T-8 ~ T-17 은 여전히 유효하며, 본 문서는 그 중
>     T-14 "default_value 서버 검증 부재" 를 종결 처리하고, 본 변경이 새로 만든 위협만 추가 평가한다.

## 1. 선행 위협 재평가

| ID | 위협 | P3 보고서 평가 | 본 보고서 평가 | 변화 |
|----|------|---------------|----------------|------|
| T-14 | `default_value ↔ field_type` 불일치가 API 경유로 저장되어 이후 추출 시 런타임 오류를 유발 | 부분 통과 (UI 경고만) | ✅ 통과 | 서버 `model_validator` 에서 거절 |

나머지 T-8 ~ T-17 은 본 변경과 무관하므로 평가 상태 변동 없음.

## 2. 신규 위협 평가

### T-18. DnD payload 변조로 다른 편집기 인스턴스의 필드를 이동

**시나리오** — 악의적 스크립트/확장이 drag 중 `dataTransfer` 를 가로채 다른 편집기
(예: 같은 페이지 내 중첩 nested_schema 편집기) 의 필드를 이동하려 시도.

**방어**
- payload 포맷: `${instanceIdRef.current}|${key}`.
- `instanceIdRef` 는 컴포넌트 mount 시 1회 생성되는 `fields-editor-{depth}-{random}`.
- drop 핸들러에서 scope 가 일치하지 않으면 무시.

**잔여 리스크** — 같은 탭 내 JavaScript 는 React state/ref 직접 읽기가 가능하므로
이는 "일반 XSS 방어" 수준까지의 방어만 제공한다. XSS 자체가 발생하면 이 방어는 무력.
→ 이는 T-11 (XSS) 방어선으로 커버됨.

**결론: 통과 (잔여 리스크는 T-11 과 같은 계층)**

### T-19. DnD 로 필드 수가 제한을 우회할 수 있는가

**시나리오** — DnD 재배치 중 `applyOrder` 가 `fields` 를 재구성하면서 중복/누락으로
필드 수가 `MAX_FIELDS_COUNT=200` 을 넘을 수 있는가.

**방어**
- `applyOrder` 는 입력된 `orderedKeys` 에서 fields 에 실존하는 키만 복사하고,
  누락된 키는 말미에 붙인다. → 수는 항상 `Object.keys(fields).length` 와 동일, 순서만 변경.
- 개수 상한은 추가/업로드/업데이트 시에 이미 서버 `_validate_fields_map` 이 강제 (200 상한).

**결론: 통과**

### T-20. DnD 로 깊이 상한(MAX_NESTED_DEPTH=3) 우회

**시나리오** — DnD 가 nested_schema 의 자식 필드를 부모 레벨로 꺼낸 뒤 object 필드 아래로
재배치하면 깊이가 증가할 수 있는가.

**방어**
- 레벨 간 이동은 원천 차단 (instanceId scope 불일치로 drop 거절).
- 동일 레벨 내 재배치는 객체 구조를 변경하지 않으므로 최대 깊이가 바뀌지 않는다.
- 서버 `_max_nested_depth` 체크가 최종 게이트.

**결론: 통과**

### T-21. scope_profile_id URL 인젝션

**시나리오** — `scope_profile_id` 쿼리 값이 악의적 문자열(`?scope_profile_id=../`, SQL 특수문자 등) 을 포함하여 백엔드 경로/쿼리에 인젝션될 수 있는가.

**방어**
- 프론트: `buildQueryString` 이 `URLSearchParams` 를 사용 → 문자열 인코딩 자동.
- 백엔드: 엔드포인트가 `UUID(scope_profile_id)` 로 파싱하여 형식 불일치 시 `HTTPException(422)`.
- 레파지토리 계층은 파라미터화된 SQL (psycopg) 사용.

**결론: 통과**

### T-22. Versions 페이지네이션 scope 불일치로 인한 ACL 우회

**시나리오** — 초기 페이지는 scope A 로 로드되었는데 "더 보기" 요청이 scope 없이 전송되어
scope 밖의 버전이 포함될 가능성.

**방어**
- `handleLoadMore` 는 `scope_profile_id: scopeProfileId ?? null` 을 항상 전달.
- `useQuery` queryKey 에 scopeProfileId 포함 → scope 전환 시 캐시 재로딩.
- `useEffect([data])` 에서 초기 페이지 교체 시 `extraPages` 버퍼 리셋 → 과거 scope 의 잔존 페이지가 이어지지 않음.

**결론: 통과**

### T-23. DnD 가 편집 중 입력 텍스트를 드래그로 오인

**시나리오** — 확장된 행에서 <input>/<textarea> 의 텍스트 선택이 드래그로 오인되어 의도치 않은
재배치가 발생할 수 있는가.

**방어**
- `draggable` 속성은 핸들 `<span>` 에만 부여. `<li>` 나 `<input>` 은 draggable 이 아니므로
  일반 텍스트 선택이 드래그 이벤트로 승격되지 않는다.
- `<span>` 은 `select-none` 으로 텍스트 선택 자체도 차단.

**결론: 통과**

### T-24. 서버 default_value 검증이 거대한 페이로드에 대해 DoS 를 유발하는가

**시나리오** — `array` / `object` 타입의 default_value 로 거대한 구조를 전송하여
`isinstance` 체크 이후에 구조를 재귀 순회하는 코드가 있는가.

**방어**
- `_validate_default_value` 는 `array` → `list` 여부만, `object` → `dict` 여부만 검사.
  내부 구조 재귀는 하지 않는다. (재귀 검증은 상위 `_validate_fields_map` 의 `_max_nested_depth`
  가 `nested_schema` 구조에 대해서만 수행.)
- FastAPI/Pydantic 기본 JSON payload 한도(없으면 `app.config` 의 상한) 는 프레임워크 레벨에서 관리.

**결론: 통과**

## 3. 종합

| 영역 | 평가 |
|------|------|
| T-14 (선행 부분 통과 → 후속-A 로 완전 통과) | ✅ 종결 |
| T-18 ~ T-24 (본 변경에서 새로 평가된 위협) | ✅ 모두 통과 |
| 신규 critical/high 취약점 | 0건 |
| 의존성 추가 | 없음 (HTML5 DnD 네이티브 사용) |

## 4. 보안 권고

1. **서버 검증 선회 금지** — 프론트 `DefaultValueWidget` 의 경고는 UX 피드백이며,
   보안 경계는 `ExtractionFieldDef.model_validator` 에 있다. 이 validator 를 우회하거나
   임의로 해제하지 말 것.
2. **scope_profile_id queryKey 동기화** — 상세 패널이 "scope 오버라이드" UX 를 추가할 때는
   `VersionHistorySection` 의 queryKey 에 해당 오버라이드 값을 함께 넣어 캐시 분리를 유지해야 한다.
3. **DnD 확장 시 scope 체크 유지** — DnD 를 react-dnd 등 외부 라이브러리로 교체할 경우에도
   payload scope 검사를 유지할 것.

## 5. 로컬 검증 필요

- `pytest backend/tests/unit/extraction/test_extraction_field_def.py -k TestDefaultValueCompatibility`
  → 22 케이스 통과 확인 (샌드박스 Python 버전 불일치로 원격 실행 불가).
- 브라우저 수동 확인
  - Chrome/Firefox 에서 같은 레벨 필드 드래그 재배치.
  - 중첩 object 필드 내부에서 필드 재배치 (부모 레벨로 탈출 불가 확인).
  - 키보드 Tab 순회 → ↑/↓ 버튼 포커스 가능 → 경계에서 disabled.
  - 확장된 행에서 <input> 텍스트 선택이 드래그로 오인되지 않음.
