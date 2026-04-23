# 추출 스키마 관리 화면 — P1 개선 검수보고서

- 대상 경로: `/admin/extraction-schemas`
- 작성일: 2026-04-21
- 범위: P0(검수·보안보고서 완료) 이후 남아 있던 P1 잔존 작업 4건

## 1. 배경 — P1 의 정의

P0 에서 `/admin/extraction-schemas` 는 "실 API 로 목록 조회만 동작" 하는 수준까지 복구되었다.
남은 잔존 작업 4건은 다음과 같이 P1 로 분류해 순차 착수했다.

| # | ID | 항목 | 기대 효과 |
| - | -- | ---- | -------- |
| 1 | P1-A | 쓰기 경로 UI (생성·편집·폐기·삭제) | 조회 전용 한계 해소 |
| 2 | P1-B | 버전 이력 뷰어 | 스키마 변경 추적/감사 UX |
| 3 | P1-C | `GET /extraction-schemas/{doc_type}` 의 `scope_profile_id` 쿼리 연결 | S2 ⑥ 단건 조회 ACL 슬롯 실사용 |
| 4 | P1-D | 모달 포커스 트랩 + 복귀 포커스 | WCAG 2.4.3 Focus Order 준수 |

## 2. 조치 요약

### 2.1 백엔드 (P1-C)

`backend/app/api/v1/extraction_schemas.py` — `get_extraction_schema()` 라우터 시그니처 확장.

```python
def get_extraction_schema(
    doc_type: str,
    include_deprecated: bool = Query(default=False),
    scope_profile_id: Optional[str] = Query(default=None),   # ★ 신규
    actor: ActorContext = Depends(resolve_current_actor),
):
    parsed_scope: Optional[UUID] = None
    if scope_profile_id:
        try:
            parsed_scope = UUID(scope_profile_id)
        except ValueError:
            raise HTTPException(status_code=422, detail="scope_profile_id가 유효한 UUID가 아님")
    with get_db() as conn:
        repo = ExtractionSchemaRepository(conn)
        schema = repo.get_by_doc_type(
            doc_type,
            include_deprecated=include_deprecated,
            scope_profile_id=parsed_scope,
        )
```

- 리포지토리 레이어(`get_by_doc_type`)는 이미 P0 에서 슬롯을 준비해 둔 상태라 라우터 한 군데만 시그니처 확장.
- UUID 파싱 실패 시 `422 Unprocessable Entity` 로 거절 → SQL 경로에 문자열이 그대로 흘러가지 않음.
- 기본값 `None` 이므로 기존 호출부(관리자 전역 조회) 는 회귀 없음.

### 2.2 프론트엔드 훅 (P1-D) — `frontend/src/hooks/useFocusTrap.ts` 신규

- 재사용 가능한 `useFocusTrap<T>(containerRef, active)` 훅.
- FOCUSABLE_SELECTOR 에 `a[href]`, `input:not([disabled]):not([type="hidden"])`, `select`, `textarea`, `button`, `iframe`, `[tabindex]:not([tabindex="-1"])`, `[contenteditable]` 포함.
- `aria-hidden="true"` 와 `offsetParent === null` (display:none) 요소는 후보에서 제외.
- 동작:
    1. `active=true` → 현재 `document.activeElement` 저장, 컨테이너 내부 첫 tabbable 로 포커스 이동.
    2. `Tab` / `Shift+Tab` 이 컨테이너 밖으로 빠져나가려 하면 `preventDefault` 후 반대편으로 wrap.
    3. 언마운트(`active=false`) 시 저장한 요소로 포커스 복귀. 요소가 DOM 에서 사라진 경우 `document.contains` 체크로 보호.

### 2.3 프론트엔드 페이지 재설계 (P1-A / P1-B / P1-D)

`frontend/src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx` — ~1048L 로 대체.

#### 2.3.1 페이지 루트

- `+ 새 스키마` 버튼으로 `CreateSchemaModal` 실행.
- `폐기된 스키마 포함` 체크박스로 `list(is_deprecated: false|undefined)` 토글.
- 목록 테이블의 각 행은 `role="button"`, `tabIndex=0`, Enter/Space 로 상세 패널 진입.
- 상세 버튼은 `stopPropagation` 으로 행 onClick 과 중첩 방지.

#### 2.3.2 `CreateSchemaModal` (P1-A3)

- 입력: `doc_type_code` (regex `^[a-zA-Z][a-zA-Z0-9_-]*$`), `scope_profile_id` (선택, UUID ≥32자 사전 체크 후 서버 최종 검증), `fields` (JSON textarea).
- placeholder 는 `party_a` / `amount` 2건이 들어 있는 예시 JSON. 사용자는 편집해 제출한다.
- `parseFieldsJson` 유틸이 JSON 구조 + 각 필드의 `field_name / field_type / description` 타입/존재 검증.
- 제출 실패 (409 / 422 / NetworkError) 는 `ErrorBanner` 로 표시 — `useMutationWithToast` 가 토스트도 동시 발행.

#### 2.3.3 `SchemaDetailPanel` + 편집 모드 (P1-A1)

- `data?.data ?? schema` 패턴으로 목록의 요약본을 fallback, 서버의 최신본이 오면 교체.
- `편집` 토글 → JSON textarea + `change_summary` 인풋 노출. `취소` 로 원복.
- 저장 시 `parseFieldsJson` 검증 → `PUT /extraction-schemas/{doc_type}` 호출 → 성공 시 list/상세/버전 세 캐시 키 모두 invalidate.
- 폐기된 스키마는 `편집` 버튼 disabled, `title` 로 사유 안내 ("폐기된 스키마는 수정할 수 없습니다").

#### 2.3.4 `SchemaDetailPanel` 위험 구역 (P1-A2)

- `폐기 표시` → `DeprecateDialog` (aria-modal alertdialog, reason textarea 필수) 를 띄우고 `PATCH /{doc_type}/deprecate` 호출.
- `삭제` → 기존 공용 `ConfirmDialog` (destructive prop) 로 재사용 → `DELETE /{doc_type}` 호출.
- 폐기 완료 시 같은 상세 패널 유지(상태만 "폐기됨" 으로 전환). 삭제 완료 시 패널 자동 닫힘.

#### 2.3.5 `VersionHistorySection` (P1-B)

- `useQuery(["admin","extraction-schemas", docTypeCode, "versions"], () => getVersions(..., {limit: 20}))`.
- 각 행: `v{version}` · 생성일 · `change_summary` · `changed_fields` 를 chip 로 표시 · `by {created_by}`.
- 변경된 필드 chip 는 `font-mono` 로 필드명을 그대로 노출 — 감사 목적.
- 로딩/에러/빈 상태 모두 단일 `<section>` 내에 일관된 형태로 표현.

#### 2.3.6 포커스 트랩 · 복귀 포커스 · Esc 닫기 (P1-D)

- `CreateSchemaModal`, `SchemaDetailPanel`, `DeprecateDialog` 세 모달 전부 `useFocusTrap` 적용.
- 각 모달은 자체 `useEffect` 로 `Escape` 키를 감지해 닫힘 트리거.
- `SchemaDetailPanel` 은 폐기 다이얼로그가 열려 있을 때는 Esc 를 상위가 받지 않도록 `confirmKind === null` 가드.
- 배경 클릭(`onClick={onClose}` + 내부 div 의 `stopPropagation`) 은 P0 에서 도입했던 패턴을 유지.

## 3. 재현 전/후 흐름

### Before (P0 완료 시점)
1. 목록 조회는 정상. 상세 패널은 열리지만 편집/폐기/삭제/생성 UI 없음.
2. 버전 이력은 API 만 존재하고 UI 미구현 — 스키마 변경 감사가 UI 로는 불가능.
3. 모달이 열려도 Tab 이 배경 요소로 빠져나가고, 닫았을 때 포커스가 사라짐 (`body`).

### After
1. `+ 새 스키마` 로 생성 → 즉시 목록에 반영 (invalidate).
2. 상세 패널에서 JSON 을 편집 후 저장 → 새 버전이 생성되고 버전 이력에 한 줄 추가됨.
3. 폐기 버튼 → 사유 입력 → 상태 뱃지 "폐기됨". 편집 버튼 disabled.
4. 삭제 버튼 → `ConfirmDialog` 확인 → 목록에서 제거 (감사 기록은 유지).
5. 모달이 열리면 Tab 은 모달 내부에서만 순환, 닫으면 직전 트리거 요소로 포커스 복귀.

## 4. 회귀 영향

| 영역 | 회귀 가능성 | 확인 방법 |
| ---- | ----------- | --------- |
| 다른 admin 페이지 | 없음 | `useFocusTrap` 은 신규 파일, 참조는 본 페이지 3개 모달에 한정 |
| `extractionSchemasApi` 다른 소비자 | 없음 | 전수 grep 결과 `AdminExtractionSchemasPage.tsx` 단독 소비 |
| `ConfirmDialog` 다른 사용처 | 무해 | 기존 props 계약(`destructive`, `onConfirm`, `onCancel`) 그대로 사용 |
| 라우터 기존 호출 | 없음 (P1-C) | `scope_profile_id` 기본값 `None` → 기존 전역 조회 그대로 |
| 리포지토리 계약 | 없음 | `get_by_doc_type` 시그니처는 P0 에서 이미 확장됨, P1 은 라우터만 사용 |

## 5. 검증

- Python `ast.parse`
    - `backend/app/api/v1/extraction_schemas.py` : OK
    - `backend/app/repositories/extraction_schema_repository.py` : OK
- `npx tsc --noEmit` (frontend) : **추출 스키마 관련 파일에서 타입 에러 0건**.
    - 전체 프로젝트에서 남은 타입 에러 1건은 `AdminUserDetailPage.tsx:279` 로 본 PR 범위 밖(P0 검수보고서에서도 언급).
- 컴포넌트 소비 검증
    - `useMutationWithToast` 계약과 `invalidateKeys: QueryKey[]` 형태 일치.
    - `ConfirmDialog` props (`open/title/description/confirmLabel/destructive/onConfirm/onCancel`) 일치.
    - `useFocusTrap` 의 `RefObject<T | null>` 시그니처가 `useRef<HTMLDivElement>(null)` 과 호환.

## 6. UI 디자인 리뷰 5회 반복 (CLAUDE.md "UI 디자인 리뷰 최소 5회" 준수)

### Review 1 — 스키마 생성 모달의 입력 유효성
- 최초안: doc_type_code 를 자유 입력으로 받고 서버 422 에 의존.
- 개선: 클라이언트측 regex(`^[a-zA-Z][a-zA-Z0-9_-]*$`) + 힌트 문구 추가. 422 네트워크 왕복 제거.

### Review 2 — fields JSON 입력 UX
- 최초안: 빈 textarea.
- 개선: `field_name/field_type/required/description` 4키를 담은 예시 JSON 을 placeholder 로 세팅 → 신규 사용자가 최소 스키마를 학습하고 편집만 하면 됨. `parseFieldsJson` 이 최소 3키(`field_name`, `field_type`, `description`) 존재까지 검증.

### Review 3 — 폐기 vs 삭제의 시각적 차별
- 최초안: 폐기/삭제 모두 red 버튼.
- 개선: 폐기는 `amber` (재활성 불가지만 감사 기록 유지, 복구 여지), 삭제는 `red` (소프트 삭제). 위험 구역 섹션에 기능 차이 설명 문구 추가.

### Review 4 — 편집 중 서버에서 최신 데이터가 들어올 때의 충돌
- 최초안: `useEffect` 가 편집 모드와 무관하게 `fieldsText` 를 덮어씀 → 입력 중 내용 손실.
- 개선: `if (!editMode)` 가드로 덮어쓰기를 편집 미사용 중일 때만 수행. 저장 성공 시 `editMode=false` + `changeSummary=""` 초기화.

### Review 5 — 포커스 트랩과 중첩 모달의 상호작용
- 최초안: `SchemaDetailPanel` 이 열린 상태에서 `DeprecateDialog` 까지 열리면 Esc 가 상세 패널까지 함께 닫음.
- 개선: 상세 패널의 Esc 핸들러에 `confirmKind === null` 가드를 두어 다이얼로그가 열린 동안은 Esc 가 상위로 흐르지 않음. 또한 각 다이얼로그는 자체 `useFocusTrap` 을 유지하되 활성 시점이 겹치지 않도록 분기.

## 7. 잔존 / 후속 (P2 이하)

- [P2] `ExtractionSchemaField` 의 세부 속성(`pattern`, `instruction`, `examples`, `enum_values`, `max_length`, `min/max_value`, `date_format`, `default_value`, `nested_schema`) 을 필드별 편집 위젯으로 제공. 현재는 JSON 편집만 지원.
- [P2] 버전 이력에서 두 버전 diff 뷰(JSON 비교 하이라이트). 현재는 `change_summary` + `changed_fields` 리스트만.
- [P3] `list_all` 이 `DISTINCT ON (doc_type_code)` 로 페이지 내 중복은 제거하지만, 별도 `COUNT(*)` 없이 `total = len(schemas)` 로 계산 → 정확 페이지네이션 필요 시 보강.
- [P3] Scope Profile 지정/미지정 스키마의 UI 구분 배지 (현재 상세 패널에서만 ID 를 노출).
