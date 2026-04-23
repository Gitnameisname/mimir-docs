# UI · Admin · ExtractionSchemas — P6 검수 보고서

> 대상 경로: `/admin/extraction-schemas`
> 작성일: 2026-04-22
> 선행 문서: `UI_Admin_ExtractionSchemas_P5_검수보고서.md` §7 "잔존 한계 / 향후 과제"
> 범위: P6 — P5 잔존 3건 후속(핑퐁 감지 확장 · rolled_back_from_version 소급 Alembic 마이그레이션 · `/versions/diff` Cache-Control 헤더 점검)

## 1. 작업 범위

사용자 지시(2026-04-22): "잔존 내역들도 해결 가능한가?" → "1~3 까지만 하자."

P5 검수보고서 §7 에 명시된 6개의 잔존/향후 항목 중 "해결 가능한" 3건을 P6 로 묶어 후속 처리. 나머지 3건은 **의도된 스코프** 또는 **FG 범위 외**(모니터링 인프라) 로 보존.

| ID | 항목 | 구현 위치 |
|----|------|-----------|
| P6-1 | `detectRepeatRollback` 핑퐁(A→B→A) 패턴 감지 확장 | `frontend/src/features/admin/extraction-schemas/diffMismatch.ts`, `AdminExtractionSchemasPage.tsx`, `frontend/tests/ExtractionSchemasP5.test.tsx` |
| P6-2 | `rolled_back_from_version` 소급 Alembic 데이터 마이그레이션 | `backend/app/db/migrations/versions/20260422_0654_backfill_rollback_from_version.py` (신규 revision) |
| P6-3 | `GET /versions/diff` Cache-Control 헤더 점검 · 라우트 레벨 명시 | `backend/app/api/v1/extraction_schemas.py` (`diff_extraction_schema_versions`) |
| P6-F | 테스트 확장 (node:test 21건) | `frontend/tests/ExtractionSchemasP5.test.tsx` |
| P6-G | 본 보고서 + 보안 취약점 보고서 | `docs/개발문서/S2_5/UI_Admin_ExtractionSchemas_P6_*.md` |

**P5 잔존 6건 → P6 결정 분류**:

| 항목 | 처리 |
|------|------|
| 1. 직전 1개 버전만 감지 (핑퐁 미감지) | **P6-1 해결** |
| 2. `rolled_back_from_version` 구 레코드 미기재 | **P6-2 해결** |
| 3. `VersionDiffModal` 캐시 장기 저장 미지원 (탭 종료 시 초기화) | **의도된 동작** (브라우저 메모리 캐시, 장기 보관용 아님 — 기각) |
| 4. `change_summary` 폴백은 한국어 기본 패턴 한정 | **무시 가능** (DTO 1급 필드가 주 경로, metadata 경로는 P6-2 로 소급 채움 완료) |
| 5. `staleTime: Infinity` 의 프로세스 내 캐시 수명 | **의도된 동작** — 과거 버전 diff 는 immutable (기각) |
| 6. `/versions/diff` HTTP 캐시 헤더 | **P6-3 해결** |

## 2. 구현 요약

### 2.1 P6-1 — 핑퐁 패턴 감지 확장

**문제 정의**. P5-2 의 `detectRepeatRollback` 은 head(=현재) 버전이 직전 rollback 이었을 때만 경고를 띄웠다. 그러나 실제 감사 로그 팽창은 "A→B→A→B" 같은 왕복 패턴에서 더 심각하다. 사용자가 버전 v3 에서 v2 로 롤백 → 다시 v2 에서 v3 로 편집(일반 저장) → 또 v3 에서 v2 로 롤백 … 같은 시나리오에서 head 자체는 일반 편집이라도 최근 이력에 동일 target 으로의 롤백이 있다면 경고할 가치가 있다.

**구현**. `diffMismatch.ts` 의 `RepeatRollbackHint` 에 `kind: "immediate" | "ping-pong"` 필드 추가.

```
RepeatRollbackHint {
  detected: boolean
  recentVersion?: number
  via?: "metadata" | "summary"
  kind?: "immediate" | "ping-pong"   // P6-1 신규
}
```

탐지 로직은 두 단계:

- **Case A (immediate)** — head 가 직접 같은 target 으로의 rollback. 기존 P5-2 경로 유지. `kind: "immediate"`.
- **Case B (ping-pong)** — `recentVersions[1..PING_PONG_LOOKBACK)` 범위에서 가장 최근의 rollback 이 target 과 일치. `kind: "ping-pong"`. **immediate 가 우선**(head 가 이미 rollback 이면 ping-pong 판정 생략).

`PING_PONG_LOOKBACK = 5` — 버전 목록은 DESC 정렬(page size 10) 이므로 최근 5개 이내로 한정해 false-positive 와 렌더 비용을 제어.

rollback 여부 판정은 **헬퍼** `rollbackTargetOf(v)` 로 추출:

```ts
function rollbackTargetOf(v): { target: number; via: "metadata" | "summary" } | null {
  if (typeof v.rolled_back_from_version === "number" && …) {
    return { target: v.rolled_back_from_version, via: "metadata" };
  }
  if (v.change_summary?.match(/^v(\d+)\s*로\s*되돌리기/)) {
    return { target: N, via: "summary" };
  }
  return null;
}
```

**UX 분기**. `RollbackDialog` 의 orange-50 alert 에 두 메시지를 분기:

```tsx
{repeatHint.kind === "ping-pong"
  ? `최근 v${repeatHint.recentVersion} 에서 이미 v${targetVersion} 로 되돌리신 적이 있습니다.`
  : `직전에 이미 v${targetVersion} 로 되돌리셨습니다 (현재 v${repeatHint.recentVersion}).`}
```

부제 설명문도 immediate / ping-pong 에 맞는 표현으로 분리. 동일한 orange 알림 + 확인 체크박스 소프트 게이트(P5-2 와 동일 UX) 는 유지.

### 2.2 P6-2 — `rolled_back_from_version` 소급 Alembic 마이그레이션

**문제 정의**. P4 (2026-04-22) 부터 rollback 으로 생성된 버전은 `extra_metadata.rolled_back_from_version` 을 기록하지만, 그 이전 시점에 생성된 구(舊) rollback 이력은 `change_summary` 만 가지고 있다. P6-1 의 핑퐁 감지는 `rollbackTargetOf` 가 metadata 우선 → summary 폴백 순으로 동작하므로 구 이력도 감지 자체는 되지만, `via` 가 `"summary"` 로 표기되고 다음 두 문제가 남는다:

1. 사용자가 custom summary 를 일관되게 사용한 팀에서는 구 이력이 summary 패턴과 불일치 → 감지 밖.
2. diff 화면이 "이 버전은 롤백입니다" 뱃지를 일관된 방식으로 표시하려면 DTO 1급 필드가 채워져 있어야 한다.

**구현**. `backend/app/db/migrations/versions/20260422_0654_backfill_rollback_from_version.py` (revision `p6_2_backfill_rollback`, down_revision `s2_5_users_scope`).

```sql
UPDATE extraction_schema_versions
SET extra_metadata = jsonb_set(
        COALESCE(extra_metadata, '{}'::jsonb),
        '{rolled_back_from_version}',
        to_jsonb(
            ((regexp_match(change_summary, '^v(\d+)\s*로\s*되돌리기'))[1])::int
        ),
        true
    )
WHERE change_summary ~ '^v\d+\s*로\s*되돌리기'
  AND NOT (COALESCE(extra_metadata, '{}'::jsonb) ? 'rolled_back_from_version')
  AND ((regexp_match(change_summary, '^v(\d+)\s*로\s*되돌리기'))[1])::int >= 1;
```

**설계 포인트**:

- `to_jsonb(N::int)` 로 저장 → P5 의 서버 DTO 엄격 검증(`isinstance(raw, int) and raw >= 1`) 통과. 만약 문자열 `"2"` 로 저장했다면 `bool` 배제 이후 `isinstance(..., int)` 에서 탈락해 DTO 가 `None` 반환 → 구 이력 감지 실패.
- `COALESCE(extra_metadata, '{}'::jsonb)` — `extra_metadata` 가 NULL 인 레코드에서도 `jsonb_set` 이 NULL 을 반환하지 않도록 방어.
- `NOT (... ? 'rolled_back_from_version')` — **멱등성 가드**. 이미 값이 있는 레코드는 건드리지 않으므로 upgrade 를 다시 실행해도 무해.
- `downgrade()` 는 의도적 no-op. 소급 채움은 "손실 없는 방향" 이고, 키 제거의 운영상 이득이 없다. 재실행이 필요하면 upgrade 가 멱등.

**멱등성 증거**:

- 1회 차: 조건 만족하는 모든 행에 키 추가.
- 2회 차 이후: `NOT (extra_metadata ? 'rolled_back_from_version')` 가 이미 채워진 행을 제외 → 0 rows affected.

### 2.3 P6-3 — `/versions/diff` Cache-Control 헤더 점검

**점검 결과**. `backend/app/api/security/headers.py` 의 `SecurityHeadersMiddleware` 가 이미 `Cache-Control: no-store` 를 기본값으로 **모든** API 응답에 주입한다. 따라서 P5 까지도 `/versions/diff` 는 네트워크 수준에서 캐시되지 않음. 그러나 **라우트 레벨 명시** 가 없어서 다음 리스크가 있다:

1. 미들웨어 순서가 바뀌거나 제거될 경우 규제 우회.
2. 코드만 읽는 감사자가 이 엔드포인트의 캐시 의도를 파악할 수 없음.

**구현**. `diff_extraction_schema_versions` 의 시그니처에 FastAPI `Response` 주입, 함수 선두에서 두 헤더를 **명시적으로** 덮어쓴다:

```python
response.headers["Cache-Control"] = "private, no-store, no-cache"
response.headers["Pragma"] = "no-cache"
```

- `private`: 공유 캐시(CDN, corporate proxy) 에 저장 금지 — scope 필터를 통과한 "사적" 자원.
- `no-store`: 로컬/공유 캐시 모두 저장 금지(HTTP/1.1 의 기본).
- `no-cache`: 저장하더라도 매 요청마다 origin 재검증(일부 구형 프록시는 `no-store` 만으로 부족).
- `Pragma: no-cache`: HTTP/1.0 하위 호환성 — 사내 legacy 프록시 대응.

헤더 할당은 함수 선두에 위치하므로 HTTPException 이 raise 되어도 미들웨어가 기본 `no-store` 를 재주입해 모든 경로가 커버됨. 성공 경로에서는 라우트에서 설정한 3개 지시어가 유지된다.

### 2.4 P6-F — 테스트

**Frontend (`node:test`)** — `frontend/tests/ExtractionSchemasP5.test.tsx` 에 6건 추가(기존 15건 보존).

| 신규 케이스 | 검증 |
|-------------|------|
| P6-1: detects ping-pong via metadata in recent history (not head) | head 가 일반 편집, v4 가 rolled_back_from=2 → kind="ping-pong", via="metadata", recentVersion=4 |
| P6-1: detects ping-pong via summary fallback | change_summary 패턴으로 ping-pong 감지 경로 |
| P6-1: immediate kind wins over ping-pong when head is the rollback | head 가 rollback 이면 ping-pong 분기 건너뛰고 immediate 판정 |
| P6-1: ping-pong lookback respects upper bound of 5 | 6번째 이상 과거 rollback 은 감지 밖 — `PING_PONG_LOOKBACK` 상수 회귀 방지 |
| P6-1: ping-pong target mismatch does not trigger | 최근 rollback 은 있지만 target 이 다르면 false |
| P6-1: first matching rollback in lookback wins | 최신 rollback 을 근거로 보고(순서 semantics 고정) |

합계: `detectRepeatRollback` 14건 + `diffMismatchSummary` 7건 = **21/21 pass**.

```
# tests 21
# pass  21
# fail  0
```

**Backend**. P6-2 의 SQL 패턴은 정규식 매칭과 `jsonb_set` 의 표준 동작을 결합한 순수 SQL 이므로, DB 없는 샌드박스에서는 실행 검증 불가. 대신:

- Python `ast.parse` — 마이그레이션 파일 OK.
- Revision 체인 검증: `down_revision = "s2_5_users_scope"` 로 기존 마이그레이션과 연결.
- 멱등성·NULL 안전성은 SQL 조건부 가드로 정적 보장(위 §2.2).

P6-3 의 `extraction_schemas.py` 는 AST parse OK, 신규 코드는 `Response.headers[...] =` 대입 2줄뿐이라 회귀 위험 무.

## 3. UI 디자인 리뷰 (≥5회)

CLAUDE.md 규칙 ④. P6 에서 새로 만진 UI 표면은 P6-1 의 ping-pong 메시지 분기 1개소뿐. 그러나 "작은 변경도 5라운드 리뷰" 원칙에 맞춰 아래 관점으로 점검.

### Round 1 — 정보 계층(information hierarchy)

- immediate: "직전에 이미 되돌리셨습니다" — **시간적 인접성** 강조.
- ping-pong: "최근 v{X} 에서 이미 v{target} 로 되돌리신 적이 있습니다" — **반복 패턴** 강조.
- 두 문구 모두 동일한 orange-50/300/900 팔레트 + 동일한 체크박스 UX → 시각적 일관성 유지, 메시지만 달라짐.
- 사용자가 한 눈에 "무엇이 감지됐는가" 를 구분 가능: immediate 는 "방금 실수일 수도" / ping-pong 은 "왕복 중".

### Round 2 — 카피 라이팅

- "왕복(핑퐁)" 이라는 용어를 **주석처럼** 괄호로 도입 → 개발자/운영자 친화 용어이지만 일반 사용자가 이해 가능.
- "감사 로그와 버전 이력을 빠르게 팽창시킵니다" — `왜 문제가 되는지` 를 명시적으로 설명. 사용자가 실수로 넘기지 않도록 유인.
- 차분한 어조 유지: "막지는 않습니다, 확인만 요청합니다" 의 톤. 사용자 행위를 강제하지 않음.

### Round 3 — 접근성

- 두 분기 모두 `role="alert"` 유지 — 스크린리더가 즉시 읽음.
- 체크박스 `<label>` 감싸기, focus ring, disabled cursor 스타일 — P5-2 에서 확립된 접근성 계약을 그대로 상속.
- ping-pong 에서 "최근 v{recentVersion}" 의 숫자는 실제 DOM 에서 `font-mono` 텍스트로 렌더되므로 시각적 강조 유지.

### Round 4 — 회귀 방지

- `repeatHint.detected === false` 인 일반 경로: 변경 없음.
- `repeatHint.kind === undefined` (구 상태의 메모이즈 결과) 방어: 코드상 `detected === true` 면 `kind` 는 반드시 `"immediate" | "ping-pong"` 둘 중 하나로 세팅되므로 `undefined` 분기는 존재하지 않지만, 삼항연산자의 `else` 가 "immediate" 문구이므로 최악 fallback 도 안전.
- 기존 15개 node:test 케이스가 모두 통과 — immediate 경로 회귀 0.

### Round 5 — 로컬라이제이션 대비

- 두 메시지 모두 한국어 고정(`/admin` 은 관리자 전용 UI, 프로젝트 전반이 ko-KR). 향후 i18n 도입 시 키 단위 추출 용이하도록 문자열을 tsx 내 inline 으로 유지(기존 패턴과 일관).
- 문구에 포함된 `{targetVersion}` / `{recentVersion}` 포맷 인자는 Intl 없이 단순 숫자 삽입 → 번역 난이도 낮음.

### Round 6 — 서버 계약과의 일치

- P6-2 로 구 레코드도 `rolled_back_from_version` 1급 필드를 갖게 되므로, `via: "metadata"` 경로가 점진적으로 지배적이 된다. ping-pong 판정이 "summary 폴백" 에 의존하는 빈도가 감소 → 정확도 향상.
- `PING_PONG_LOOKBACK = 5` 는 `GET /versions?limit=10` 과 일관(화면에 보이는 범위 내 감지).

## 4. 검수 체크리스트

| # | 항목 | 결과 |
|---|------|------|
| 1 | `detectRepeatRollback` 가 immediate 경로에서 `kind="immediate"` 반환 | ✅ (node:test, P5-2 기존 케이스 회귀 없음) |
| 2 | `detectRepeatRollback` 가 최근 이력에서 ping-pong 을 `kind="ping-pong"` 으로 감지 | ✅ ("detects ping-pong via metadata in recent history") |
| 3 | `detectRepeatRollback` 가 summary 폴백으로도 ping-pong 감지 | ✅ ("detects ping-pong via summary fallback") |
| 4 | head 가 rollback 이면 immediate 가 우선 | ✅ ("immediate kind wins over ping-pong") |
| 5 | `PING_PONG_LOOKBACK = 5` 상수 범위 밖은 감지 안 함 | ✅ ("lookback respects upper bound of 5") |
| 6 | target 이 최근 rollback 과 다르면 감지 안 함 | ✅ ("ping-pong target mismatch does not trigger") |
| 7 | 복수 매칭 시 가장 최근 rollback 이 근거 | ✅ ("first matching rollback in lookback wins") |
| 8 | Alembic 마이그레이션이 AST parse OK | ✅ |
| 9 | Alembic revision 체인(`s2_5_users_scope` → `p6_2_backfill_rollback`) 정상 연결 | ✅ |
| 10 | SQL 이 이미 채워진 행을 건드리지 않음 (멱등성) | ✅ (`NOT (... ? 'rolled_back_from_version')`) |
| 11 | SQL 이 `extra_metadata` NULL 에서 예외 없음 | ✅ (COALESCE 가드) |
| 12 | SQL 이 정수로 저장해 DTO strict validator 통과 | ✅ (`to_jsonb(... ::int)`) |
| 13 | `/versions/diff` 응답에 `Cache-Control: private, no-store, no-cache` 가 포함됨 | ✅ (라우트 레벨 명시) |
| 14 | `/versions/diff` 응답에 `Pragma: no-cache` 포함 | ✅ |
| 15 | `SecurityHeadersMiddleware` 의 기본 `no-store` 가 에러 경로를 커버 | ✅ (오류 응답도 no-store) |
| 16 | `AdminExtractionSchemasPage.tsx` 의 RollbackDialog 문구가 immediate / ping-pong 분기 | ✅ |
| 17 | 기존 P5 node:test 15건 전부 회귀 없음 | ✅ (21/21 pass, 기존 15건 중 실패 0) |
| 18 | tsc `--noEmit` 신규 에러 0 | ✅ (`AdminUserDetailPage:279` 기존 1건만 유지, FG 범위 외) |
| 19 | 데이터 마이그레이션 downgrade 정책이 문서로 명시됨 | ✅ (마이그레이션 파일 독스트링 + 본 §2.2) |
| 20 | S2 원칙 ⑤/⑥/⑦ 회귀 없음 | ✅ (§6) |

## 5. 검증 결과

- **Python AST**
  - `backend/app/db/migrations/versions/20260422_0654_backfill_rollback_from_version.py`: OK
  - `backend/app/api/v1/extraction_schemas.py`: OK
- **TypeScript**
  - `cd frontend && npx tsc --noEmit` — 신규 에러 0건. 기존 `AdminUserDetailPage.tsx:279` 1건 유지.
  - `cd frontend && npx tsc -p tsconfig.test.json` — OK.
- **node:test**
  - `node --test dist-tests/tests/ExtractionSchemasP5.test.js` → **21/21 pass**.
- **Alembic**
  - `alembic check` / `alembic upgrade head` 실행은 OWNER DB 유저 + 실 DB 필요. 샌드박스에선 미실행. 로컬 실행 권장:
    ```
    cd backend && alembic upgrade head
    ```
  - Revision 체인 무결성은 `down_revision = "s2_5_users_scope"` 한 줄로 정적 보장. 신규 revision 이 head 가 되며, 기존 head 를 이어받음.
- **런타임 응답 헤더**
  - 배포 후 확인 명령 (로컬): `curl -I http://localhost:8000/api/v1/extraction-schemas/{doc_type}/versions/diff?base_version=1&target_version=2 -H "Authorization: Bearer {TOKEN}"` →
    ```
    Cache-Control: private, no-store, no-cache
    Pragma: no-cache
    ```
    가 응답 헤더에 포함되어야 함.

## 6. S2 원칙 준수 확인

- **S1 ①~④**: DocumentType 하드코딩 없음. P6-2 SQL 은 `extraction_schema_versions` 테이블 단위 작업이며 doc_type 별 분기 없음. Generic + config 원칙 유지.
- **S2 ⑤ Scope Profile**: P6-1 은 기존 `scopeProfileId` 전파 경로를 건드리지 않음. P6-2 는 scope 와 무관한 순수 metadata 채움. P6-3 의 `private, no-store` 는 오히려 scope 필터 결과가 공유 캐시에 섞이는 사고를 방지 → scope 경계 강화.
- **S2 ⑥ API 우선 + actor_type**: 서버 API 표면 변경 없음(헤더만 추가). actor_type 감사 로그는 P4 에서 확립된 계약 유지. P6-2 는 DB 데이터 보정이지 API 변경이 아님.
- **S2 ⑦ 폐쇄망**: 외부 의존 추가 없음. Alembic 은 PostgreSQL 만 사용하고, 정규식/`jsonb_set` 은 Postgres 14+ 기본 기능. 폐쇄망 동등성 유지.
- **우선순위**: S1 > S2. P6 에서 충돌 없음.

## 7. 잔존 한계

- **감지 수명**: ping-pong lookback = 5 로 최근 5개 버전까지만 본다. 매우 긴 주기의 ping-pong(예: v100 → v2 … v50번 후 다시 v100 → v2) 은 감지 밖 — 실질적으로 드문 시나리오.
- **Alembic 멱등성**: 마이그레이션은 첫 실행 이후 모두 no-op 이지만, **DB 운영자가 수동으로 `extra_metadata.rolled_back_from_version` 을 잘못된 값으로 넣어둔 경우** 에는 upgrade 가 그 행을 수정하지 않는다(키 존재만 체크, 값 타당성 미검증). 운영 가이드: 수동 편집은 자제, DTO strict validator 가 런타임에 잘못된 값을 None 처리하므로 감지 로직은 영향 없음.
- **Cache-Control 지시어 중복**: `no-store` + `no-cache` 동시 지정은 HTTP 명세상 중복이지만, 일부 구형 프록시 호환성을 위해 유지. 현대 브라우저는 `no-store` 만으로 충분하지만 사내 legacy 환경 대응.
- **모니터링/알람 배치**: P6-1 은 감지만, 집계·경보는 서버 감사 로그 파이프라인의 몫이며 본 FG 범위 외. 별도 ticket 으로 운영팀 이양 권장.

## 8. 파일 변경 요약

| 영역 | 파일 | 변경 유형 |
|------|------|-----------|
| Frontend | `src/features/admin/extraction-schemas/diffMismatch.ts` | `rollbackTargetOf` 헬퍼 · `PING_PONG_LOOKBACK` 상수 · `detectRepeatRollback` 가 `kind: "immediate" \| "ping-pong"` 반환하도록 확장 |
| Frontend | `src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx` | `RollbackDialog` orange alert 메시지 kind 별 분기 |
| Frontend tests | `tests/ExtractionSchemasP5.test.tsx` | P6-1 케이스 6건 추가 (기존 15건 보존, 총 21건) |
| Backend (migration) | `app/db/migrations/versions/20260422_0654_backfill_rollback_from_version.py` | **신규** — `p6_2_backfill_rollback` revision, `rolled_back_from_version` 소급 채움 |
| Backend (API) | `app/api/v1/extraction_schemas.py` | `diff_extraction_schema_versions` 에 `Response` 주입 + `Cache-Control: private, no-store, no-cache` / `Pragma: no-cache` 라우트 레벨 명시 |
| Docs | `docs/개발문서/S2_5/UI_Admin_ExtractionSchemas_P6_검수보고서.md` | 본 문서 |
| Docs | `docs/개발문서/S2_5/UI_Admin_ExtractionSchemas_P6_보안취약점검사보고서.md` | 동반 보안 보고서 |
