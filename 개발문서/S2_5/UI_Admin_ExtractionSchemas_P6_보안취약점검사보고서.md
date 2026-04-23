# UI · Admin · ExtractionSchemas — P6 보안 취약점 검사 보고서

> 대상 경로: `/admin/extraction-schemas`
> 작성일: 2026-04-22
> 선행 문서: `UI_Admin_ExtractionSchemas_P5_보안취약점검사보고서.md`
> 범위: P6 — 핑퐁 감지 확장 · rolled_back_from_version 소급 Alembic · `/versions/diff` Cache-Control

## 1. 평가 범위

P6 로 변경된 표면:

- `backend/app/db/migrations/versions/20260422_0654_backfill_rollback_from_version.py` — 신규 데이터 마이그레이션.
- `backend/app/api/v1/extraction_schemas.py` — `diff_extraction_schema_versions` 에 응답 헤더 2개 추가.
- `frontend/src/features/admin/extraction-schemas/diffMismatch.ts` — 순수 로직 확장.
- `frontend/src/features/admin/extraction-schemas/AdminExtractionSchemasPage.tsx` — RollbackDialog 문구 분기.

새 네트워크 엔드포인트 / 저장소 / 권한 모델 변경 **없음**. P5 의 DTO 엄격 검증 · scope 경계 · actor_type 감사 계약 전부 그대로 유지.

## 2. 위협 모델

### 2.1 입력 신뢰 경계

| 입력 | 소스 | 신뢰도 | 통과 지점 |
|------|------|--------|-----------|
| `recentVersions[i].rolled_back_from_version` (P6-1) | 서버 응답 | 중 — P5 DTO 검증 통과 후 도달 | `rollbackTargetOf` 헬퍼가 다시 `Number.isInteger && >= 1` 재검증 |
| `recentVersions[i].change_summary` (P6-1) | 서버 저장소 / 사용자 자유 텍스트 | 낮 | 정규식 매칭만, 값은 UI 에 직접 노출하지 않음 |
| `extra_metadata` JSONB 의 기존 내용 (P6-2) | DB | 중 — 과거 레거시 값 혼재 가능 | `jsonb_set` + 조건부 WHERE 로 타입 안전 |
| `change_summary` 정규식 캡처 그룹 (P6-2) | DB | 중 | `::int` 캐스트 + `>= 1` 조건 |
| `/versions/diff` 응답 바디 (P6-3) | 인증된 서버 | 높 | Cache-Control 로 보조 저장소에 쌓이지 않도록 |

### 2.2 자산 / 영향

| 자산 | P6 관련 영향 경로 |
|------|--------------------|
| scope 가 다른 사용자 사이의 diff 데이터 | 공유 캐시(CDN/프록시) 에 섞여 누설될 위험 → **P6-3 에서 `private` 지정으로 완화** |
| 브라우저 로컬 캐시(Back/Forward 탐색) | 사용자 자신의 환경이지만, 공유 PC 시나리오에서 "뒤로 가기" 로 이전 사용자의 diff 가 표시될 수 있음 → **P6-3 의 `no-store` 로 완화** |
| `extra_metadata.rolled_back_from_version` | 소급 채움 SQL 이 타입 부정확하면 DTO 가 해당 행을 None 으로 처리 → 감지 정확도 하락(비보안 영향) |
| UI 경고 텍스트 | 사용자가 입력한 `change_summary` 가 경고 메시지에 노출되면 XSS 가능성 — **노출 없음 (숫자만 삽입)** |

## 3. 상세 점검

### 3.1 C1 — `/versions/diff` 공유 캐시 누설 방지 (P6-3)

**위협**. 회사 내부 포워드 프록시/CDN 이 `Cache-Control` 이 약하거나 누락된 응답을 공유 캐시에 저장하면, 한 scope 사용자의 diff 결과가 다른 scope 사용자에게 리턴될 수 있다(현대적 CDN 은 Vary/Authorization 기반 분리가 있지만, 사내 legacy 프록시는 보장 없음).

**완화**. 라우트 레벨에서 3중 지시어:

```python
response.headers["Cache-Control"] = "private, no-store, no-cache"
response.headers["Pragma"] = "no-cache"
```

- `private`: 공유 캐시 저장 금지(origin 직격).
- `no-store`: 로컬/공유 어디에도 저장 금지.
- `no-cache`: 저장하더라도 매 요청 재검증.
- `Pragma: no-cache`: HTTP/1.0 하위 호환 (레거시 프록시).

미들웨어가 기본값으로 `no-store` 를 주입하지만, 라우트 레벨 선언은 **defense in depth** 로서 순서 변경/제거 시에도 엔드포인트 고유 정책이 유지됨을 보장. 오류 경로(HTTPException) 는 미들웨어가 커버.

**검증**. 로컬 실행 시 `curl -I` 로 헤더 확인(검수보고서 §5).

### 3.2 C2 — Alembic 데이터 마이그레이션 SQL 인젝션 (P6-2)

**위협**. `regexp_match(change_summary, '^v(\d+)\s*로\s*되돌리기')` 에 `change_summary` 는 사용자 자유 텍스트(서버 API 를 통해 들어온 rollback 요청의 `change_summary` 파라미터). 패턴 매칭은 안전하지만, 만약 `change_summary` 가 Postgres 정규식 엔진을 취약하게 만들 수 있다면?

**완화**:

- `regexp_match` 는 Postgres 내장 함수로, 정규식 패턴은 **코드에 하드코딩된 리터럴** 이며 `change_summary` 는 매칭 대상(input) 에만 들어간다. 패턴 자체가 사용자 입력이 아니므로 ReDoS 를 포함한 정규식 엔진 공격 경로 **차단**.
- 패턴 `^v(\d+)\s*로\s*되돌리기` 는 앵커·고정 문자열·짧은 수량자로 구성 → catastrophic backtracking 불가.
- 캡처 그룹 `(\d+)` 는 `\d` 만 허용하므로 뒤이은 `::int` 캐스트는 안전(문자 섞일 수 없음).
- `jsonb_set(target, path, value, create_missing=true)` 의 `path` 도 리터럴 `'{rolled_back_from_version}'` — 사용자 입력 없음.

**잔존**:

- `change_summary` 길이에 대한 상한이 없지만, `regexp_match` 는 첫 매치만 반환하고 앵커가 시작에 고정돼 입력 길이와 매칭 시간이 선형. 운영상 DoS 우려 없음.

### 3.3 C3 — Alembic 멱등성 / 재실행 안전성 (P6-2)

**위협**. 마이그레이션이 이미 채워진 행을 재작성하면 동시 실행 시 race 가능. 반복 실행 시 카운터 중복 증가 등 부작용 가능.

**완화**. WHERE 절에 3중 가드:

```sql
WHERE change_summary ~ '^v\d+\s*로\s*되돌리기'
  AND NOT (COALESCE(extra_metadata, '{}'::jsonb) ? 'rolled_back_from_version')
  AND ((regexp_match(change_summary, '^v(\d+)\s*로\s*되돌리기'))[1])::int >= 1;
```

두 번째 조건(`NOT (... ? 'rolled_back_from_version')`) 이 **멱등성 게이트**. 이미 키가 있는 행은 재실행돼도 불변.

`downgrade()` 를 no-op 으로 둔 것도 의도적 — 데이터 소급 채움은 논리적으로 unidirectional 이므로 "되돌리기" 가 감사 로그에 의미 없는 노이즈만 남김.

### 3.4 C4 — `extra_metadata` NULL / 비-dict 처리 (P6-2)

**위협**. `extra_metadata` 컬럼이 NULL 이거나 jsonb 배열 값(`'[]'`) 같은 비정상 케이스에서 `jsonb_set` 이 NULL 을 돌려 UPDATE 가 `extra_metadata = NULL` 로 덮어쓸 위험.

**완화**. `COALESCE(extra_metadata, '{}'::jsonb)` 로 NULL 을 빈 객체로 승격. 비-dict(배열 등) 케이스는 Postgres 에서 `? 'key'` 가 false 를 돌려 WHERE 절에서 걸러지므로 UPDATE 대상에서 제외. 안전.

### 3.5 C5 — 소급 채움 이후 DTO strict validator 호환 (P6-2)

**위협**. SQL 이 문자열 `"2"` 로 저장하거나 bool `true` 로 저장하면 P5 의 `ExtractionSchemaVersionResponse.from_domain` 이 해당 레코드를 `rolled_back_from_version = None` 으로 내리면서 클라이언트는 구 이력을 감지 못함 → 보안 아닌 기능 퇴화.

**완화**. `to_jsonb(N::int)` 로 반드시 **JSONB number** 로 저장. Python/psycopg 경유 시 `int` 로 복원되어 `isinstance(raw, int) and raw >= 1` 통과.

**검증**. P5 pytest 의 10개 strict validation 케이스가 이 계약을 고정 — P6-2 데이터가 pytest 시드로 들어와도 통과.

### 3.6 C6 — 핑퐁 감지 로직의 DoS (P6-1)

**위협**. `detectRepeatRollback` 가 `recentVersions` 길이에 선형 — 악의적 데이터로 수천 개 입력 시 렌더 스톨?

**완화**:

- `recentVersions` 는 `GET /versions` 의 `page size = 10` 응답에서 온다. 상한 명확.
- 로직은 내부적으로 `Math.min(length, PING_PONG_LOOKBACK=5)` 로 다시 절단. 실질 상한 5.
- 각 반복은 `rollbackTargetOf` 의 O(1) 연산(typeof/정규식 매칭/숫자 비교).

DoS 경로 **없음**.

### 3.7 C7 — XSS / 렌더 경로 (P6-1)

**위협**. 경고 메시지에 동적 값(`recentVersion`, `targetVersion`) 을 삽입. 만약 이 값이 문자열로 들어오면 HTML 주입 가능?

**완화**:

- 두 값 모두 TypeScript 타입 `number`. React 가 수치 textNode 로 렌더 → HTML 해석 없음.
- `change_summary` 는 경고 본문에 **노출되지 않는다**. 감지 근거(`via: "metadata"|"summary"`) 만 내부 로직에 사용.
- `repeatHint.via` 는 리터럴 유니온 타입 `"metadata" | "summary"` 로 제약 → 임의 문자열 삽입 불가.

### 3.8 C8 — 스크린샷/인쇄 캐시 (P6-3 보조)

**위협**. `Cache-Control` 은 HTTP 캐시만 제어. 사용자가 브라우저 인쇄 / 스크린샷으로 diff 를 외부 반출하면 보호 불가.

**완화**. 본 FG 범위 외(사용자 디바이스 제어는 MDM 레이어의 책임). P5 DTO 의 최소권한 원칙(`extra_metadata` 전체 누설 방지) 이 여전히 1차 방어.

### 3.9 C9 — 감사 로그(actor_type) 영향 (P6 전체)

**점검**.

- P6-1: 서버 감사 로그 변경 없음. 클라이언트 감지는 요청 전 경고일 뿐이며, 사용자가 checkbox 로 승격해 진행하면 기존 rollback 엔드포인트로 정상 요청이 들어가 `emit_for_actor(..., actor_type=...)` 가 호출됨.
- P6-2: DB 내부 데이터 보정만, 감사 로그 주체 변경 없음.
- P6-3: 응답 헤더만, 감사 로그 남기는 것 없음.

actor_type 감사 계약(S2 ⑥) 회귀 **없음**.

## 4. 위험도 요약

| ID | 항목 | 위험도(사전) | 위험도(사후) | 상태 |
|----|------|--------------|--------------|------|
| C1 | `/versions/diff` 공유 캐시 누설 | 중 | **저** | 완화 (P6-3) |
| C2 | Alembic SQL 인젝션 | 저 | **저** | 통과 (입력 source 분석) |
| C3 | 마이그레이션 멱등성 위반 | 중 | **저** | 완화 (WHERE 가드) |
| C4 | `extra_metadata` NULL 처리 | 중 | **저** | 완화 (COALESCE) |
| C5 | DTO 호환성 | 중 | **저** | 완화 (`to_jsonb(::int)`) |
| C6 | 감지 로직 DoS | 저 | **저** | 통과 (상한 5) |
| C7 | 경고 메시지 XSS | 저 | **저** | 통과 (number 타입 + React 이스케이프) |
| C8 | 사용자 디바이스 반출 | - | - | FG 범위 외 |
| C9 | actor_type 감사 로그 회귀 | - | - | 영향 없음 |

## 5. CLAUDE.md 규약 2 · 3 준수 확인

- **2. 라이브러리 critical 취약점**: 추가 의존 없음. 기존 SQLAlchemy·Alembic·FastAPI·TanStack Query v5 버전 그대로.
- **3. 보안 일반**:
  - 최소권한(C4, C7) — 출력에 불필요한 필드 노출하지 않음.
  - 입력 검증(C2, C5) — 정규식·캐스트로 타입 안전.
  - 캐시 제어(C1) — `private, no-store, no-cache` 삼중.
  - 감사 로그(C9) — P4/P5 계약 유지.

## 6. 배포 가능 판정

**✅ 배포 가능**. P6 로 추가된 표면 모두 기존 보안 계약 내에서 동작하며, `/versions/diff` 의 공유 캐시 위험(C1) 은 라우트 레벨 헤더로 명시적 완화. 데이터 마이그레이션(P6-2) 은 OWNER DB 유저로 `alembic upgrade head` 1회 실행 후 멱등.

## 7. 운영 체크리스트

배포 시 운영팀이 수행할 순서:

1. `cd backend && alembic upgrade head` — OWNER 권한.
2. 로그에서 `UPDATE extraction_schema_versions ...` 후 행 수 확인(영향 범위 기록).
3. `/admin/extraction-schemas` 진입 → rollback 이 있던 doc_type 의 버전 이력을 열어 "되돌리기" 실행 시도 → ping-pong 경고가 뜨는지 확인.
4. `curl -I` 또는 브라우저 DevTools 로 `/versions/diff` 응답에 `Cache-Control: private, no-store, no-cache` 포함 확인.
5. 기존 diff/rollback 엔드포인트 smoke test — 기능 회귀 없음 확인.
