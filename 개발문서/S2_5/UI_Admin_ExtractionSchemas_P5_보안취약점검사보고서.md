# UI · Admin · ExtractionSchemas — P5 보안 취약점 검사 보고서

> 대상 경로: `/admin/extraction-schemas`
> 작성일: 2026-04-22
> 선행 문서: `UI_Admin_ExtractionSchemas_P4_보안취약점검사보고서.md`
> 범위: P5 — diff 불일치 인디케이터 · 반복 rollback 경고 · VersionDiffModal 캐시

## 1. 평가 범위

P5 로 추가/변경된 코드:
- `backend/app/schemas/extraction.py` — `ExtractionSchemaVersionResponse.rolled_back_from_version` 1급 필드 + `from_domain` 검증.
- `frontend/src/features/admin/extraction-schemas/diffMismatch.ts` — 순수 로직 유틸(`diffMismatchSummary`, `detectRepeatRollback`).
- `frontend/src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx` — `VersionDiffModal` / `RollbackDialog` UI 통합, queryKey/staleTime 조정.
- `frontend/src/types/s2admin.ts` — `ExtractionSchemaVersion` 타입 확장.

새 네트워크 표면/엔드포인트/저장소는 **추가되지 않음**. P4 엔드포인트(`/versions/diff`,
`/rollback`, `/versions`) 의 계약·권한 가드는 그대로 유지.

## 2. 위협 모델

### 2.1 입력 신뢰 경계

| 입력 | 소스 | 신뢰도 | 통과 지점 |
|------|------|--------|-----------|
| `rolled_back_from_version` (DB row 의 `extra_metadata` 내부 키) | 서버 저장소 | 중 — 과거 레거시 레코드에서 이상 값 가능 | `from_domain` 엄격 검증 (§3.1) |
| `change_summary` (P5-2 감지의 2순위 입력) | 서버 저장소 / 사용자 입력 | 낮 — 사용자 자유 텍스트 | 정규식 매칭만 수행, 값 사용 없음 (§3.2) |
| `ExtractionSchemaDiff` (서버 정본) | 인증된 서버 응답 | 높 | `diffMismatchSummary` 가 Set/Map 으로 구조화 집계만 수행 |
| `FieldsDiff` (클라 `computeFieldsDiff` 출력) | 클라 내부 계산 | 중 — 같은 스키마 객체 유래 | 내부 비교만 수행, 렌더링 경로 없음 |

### 2.2 자산 / 영향

| 자산 | 영향 경로 |
|------|-----------|
| 관리자 스키마 버전 이력 | Rollback 남용(연속 동일 버전 rollback) → 감사 로그 오염 · 버전 테이블 팽창 |
| `extra_metadata` 내부 메타 | DTO 설계 오류로 누설 시 내부 구조 노출 |
| 관리자 페이지 렌더링 컨텍스트 | 사용자가 저장한 필드 이름(`name`) 이 배너/alert 에 표시 → XSS 위험 |

## 3. 상세 점검

### 3.1 C1 — 서버 응답 DTO 의 `rolled_back_from_version` 엄격 타입 검증

**위협**. `extra_metadata` 는 JSONB 컬럼으로, 구버전 로직/수동 DB 작업으로 인해
`"2"` (문자열), `True` (bool), `0`, 음수, 리스트 등 예상치 못한 값이 저장될
수 있다. 클라이언트가 이를 그대로 받으면 `typeof === "number"` 가드가 잘못된
경로로 트리거되어 반복 rollback 경고가 오탐/미탐 될 수 있다.

**완화**. `from_domain` 이 다음 순서로 엄격 판정:

```python
if isinstance(raw, bool):              # 1. Python bool 은 int 의 서브타입
    rolled_back_from = None            #    → 명시적 배제 (의미상 버전 번호 아님)
elif isinstance(raw, int) and raw >= 1: # 2. 1 이상의 정수만 허용
    rolled_back_from = raw
else:                                   # 3. 그 외(문자열·리스트·0·음수·None·누락)
    rolled_back_from = None             #    → 모두 None
```

**검증**. pytest `TestRolledBackFromVersionExtraction` 의 10개 케이스(bool /
string / list / 0 / 음수 / 비-dict / 기타 키) 가 전 경로 커버. ✅

### 3.2 C2 — `extra_metadata` 딕셔너리 누설 방지

**위협**. `ExtractionSchemaVersionResponse` 가 `extra_metadata` 필드를 전체로
노출하면 내부 운영 메타(예: `scope_profile_id`, 실험 플래그, 내부 트레이싱 키)
가 관리자 패널 응답에 함께 노출될 수 있다.

**완화**. DTO 에는 `rolled_back_from_version` 1급 필드만 정의. `extra_metadata`
딕셔너리 자체는 매핑되지 않으며 `model_dump()` 결과에도 포함되지 않음.

**검증**. pytest `test_other_metadata_keys_ignored`:

```python
v = _make_domain(extra_metadata={
    "rolled_back_from_version": 2,
    "secret_internal_flag": "should-not-leak",
})
dumped = ExtractionSchemaVersionResponse.from_domain(v).model_dump()
assert "extra_metadata" not in dumped
assert "secret_internal_flag" not in dumped
```

통과. ✅

### 3.3 C3 — XSS 내성 (사용자 입력 렌더링 경로)

**위협 경로**:
1. P5-1 indigo 배너가 `mismatch.serverOnly*.join(", ")` 로 필드 이름 목록을 렌더.
2. P5-2 orange alert 가 `repeatHint.recentVersion` (number), `repeatHint.via`
   (literal string) 를 렌더. `targetVersion`, `currentVersion` 모두 number.

**완화**. React JSX 의 텍스트 보간은 기본적으로 HTML-escape 된다. 본 경로에서
`dangerouslySetInnerHTML` 사용 없음. 필드 이름은 서버가 이미 P2-C Pydantic
validator 로 snake_case 에 맞춰 검증되어 있으므로(`^[a-z][a-z0-9_]*$`) 이론적
으로 HTML 메타문자가 포함될 수 없지만, React escaping 이 2차 방어선.

**검증**. `Grep dangerouslySetInnerHTML` → 해당 컴포넌트 내 0건. ✅

### 3.4 C4 — 반복 rollback 감지의 서비스 거부 방어

**위협**. `detectRepeatRollback` 가 O(recentVersions.length) 를 돌면 공격자가
매우 많은 버전을 가진 스키마로 렌더링을 지연시킬 수 있다.

**완화**. 실제 로직은 `recentVersions[0]` 만 검사(O(1)). `versions` 배열 자체
는 서버의 `VERSION_PAGE_SIZE=10` 페이지네이션(P3-D) 으로 이미 상한이 있다.
memoization(`useMemo`) 으로 리렌더 폭주 방지.

**검증**. 소스 확인 — `recentVersions[0]` 만 조회. ✅

### 3.5 C5 — `diffMismatchSummary` 의 O(n) 안전성

**위협**. 서버/클라 diff 배열 크기 과대 → 메모리/CPU 폭주.

**완화**. 서버 diff 는 `MAX_FIELDS_COUNT=200` 상한(P3-C) 이 DTO 생성 전부터
적용되어 있어, 배열 최대 크기가 200 + 재귀 nested 포함 최대 200·MAX_NESTED_DEPTH
수준. Set/Map 구성은 O(n), 중복 루프 없음.

### 3.6 C6 — TanStack Query `staleTime: Infinity` 의 보안 관점

**위협**. 캐시 무한 유지로 인한 (a) 메모리 누수, (b) 권한 변경 후 오래된
데이터 노출.

**완화**:
- `gcTime` 기본 5분 유지 → 마지막 observer 가 unmount 되면 5분 뒤 수집.
- `invalidateKeys: [["admin", "extraction-schemas"]]` (TanStack v5 prefix 매칭)
  로 스키마 mutation 시 자동 무효화. 권한 축소 후의 mutation 도 이 prefix 를
  터치하면 캐시 무효.
- 과거 버전 diff 는 **immutable**(서버에서 과거 버전 row 를 수정하지 않음) 이
  므로 stale 상태가 의미상 존재하지 않음. P4 §6 에서 문서화한 hard-delete→
  recreate 엣지 케이스는 P5-3 에서 queryKey 에 `schema_id` 를 추가해 해결.
- 로그아웃 시 전체 QueryClient 가 리셋되므로 세션 누적 없음.

### 3.7 C7 — CSRF / Session / Origin

본 P5 는 기존 `extractionSchemasApi` 호출 표면을 바꾸지 않음. `GET /versions/diff`
는 멱등 조회 — 기존 세션 쿠키/헤더 인증 경로 재사용. CSRF 취약점 도입 없음.

### 3.8 C8 — 감사 로그 무결성

`POST /rollback` 은 P4 에서 `audit_emitter.emit_for_actor` 를 통해
`extraction_schema.rolled_back` 이벤트를 발행. P5 는 이 흐름을 변경하지 않는다.
오히려 반복 rollback 을 UI 단에서 소프트 게이트로 감속 → 감사 로그 팽창 공격
표면을 축소한다.

### 3.9 C9 — 타입 안전성 (TypeScript)

`ExtractionSchemaVersion.rolled_back_from_version?: number | null` 로 선언되어
`typeof head.rolled_back_from_version === "number"` 가드가 정적으로 강제된다.
null/undefined/문자열이 오면 가드가 막아 summary 폴백 경로로 진행. ✅

## 4. 정적 분석 결과

| 도구 | 결과 |
|------|------|
| `tsc --noEmit` (전 프로젝트) | 신규 에러 0건. 기존 `AdminUserDetailPage.tsx:279` 1건 유지(본 범위 외). |
| `tsc -p tsconfig.test.json` | OK (컴파일 + dist-tests 생성) |
| Python AST parse | `app/schemas/extraction.py`, `tests/unit/extraction/test_extraction_schema_version_response.py` 모두 OK |
| node:test | 15/15 pass |

## 5. 의존성 / SBOM 델타

- 신규 라이브러리 도입 **없음**.
- 신규 외부 네트워크 호출 **없음**.
- 신규 환경변수 **없음**.
- 폐쇄망 동작 영향 **없음** (S2 ⑦ 준수).

## 6. 위험 매트릭스

| ID | 항목 | 발생 가능성 | 영향 | 완화 후 잔여 위험 |
|----|------|-------------|------|-------------------|
| C1 | `extra_metadata` 이상 값으로 오경고 | 낮 | 낮 | Minimal — 엄격 검증 + bool 배제 |
| C2 | `extra_metadata` 누설 | 낮 | 중 | Minimal — DTO 에 매핑 자체가 없음 |
| C3 | XSS | 매우 낮 | 중 | None — React escape + snake_case 서버 검증 |
| C4 | 감지 로직 DoS | 매우 낮 | 낮 | None — O(1) + 서버 페이지네이션 상한 |
| C5 | diff summary DoS | 매우 낮 | 낮 | None — 서버 `MAX_FIELDS_COUNT=200` 상한 |
| C6 | 캐시 stale 데이터 노출 | 낮 | 낮 | Minimal — immutable + prefix invalidate |
| C7 | CSRF/Origin | 해당 없음 | — | — |
| C8 | 감사 로그 오염 | 낮 | 중 | **감소** — UI soft gate 가 반복 rollback 을 억제 |
| C9 | 타입 오검출 | 매우 낮 | 낮 | None — 정적 가드 |

## 7. 권고 / 후속

1. **로깅**: 백엔드의 `rollback_to_version` 서비스가 metadata 에 `rolled_back_from_version` 을 쓰는 것은 P4 에서 구현됨. 구버전 레코드에 없는 경우 P5 의 summary 폴백 경로가 동작하지만, 장기적으로는 **백엔드 마이그레이션**으로 과거 rollback 레코드의 `extra_metadata` 에 해당 키를 소급 기록하면 클라이언트가 metadata 경로 하나로 수렴 가능. (현재 범위 외)
2. **모니터링**: 동일 `(schema_id, target_version)` 로의 rollback 빈도를 주기 집계하는 배치/알람을 운영에서 별도로 도입하면 좋다. (감사 파이프라인의 의무가 아니므로 본 FG 외부.)
3. **브라우저 캐시**: TanStack Query 외 브라우저의 HTTP 캐시는 본 문서 범위 외(서버 응답이 `Cache-Control: no-store` 이면 이슈 없음 — 기존 API 기본 설정 유지).

## 8. 결론

P5 에서 추가된 코드 경로는 **새로운 공격 표면을 도입하지 않는다**. 오히려
(a) `extra_metadata` 누설 방지 설계와 (b) 반복 rollback 의 UI 감속 게이트로
보안/감사 건전성이 소폭 개선되었다. 기존 P4 의 권한 경로(scope_profile_id
가드, 저장소 `SELECT ... FOR UPDATE`, Pydantic 검증) 는 모두 유지되며 회귀
없음.

**판정**: ✅ 배포 가능. 위에서 나열된 C1~C9 항목은 모두 "Minimal/None 잔여 위험"
수준.
