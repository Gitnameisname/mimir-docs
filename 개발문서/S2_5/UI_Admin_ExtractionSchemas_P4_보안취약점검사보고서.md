# UI · Admin · ExtractionSchemas — P4 보안 취약점 검사 보고서

> 대상 경로: `/admin/extraction-schemas`
> 작성일: 2026-04-22
> 선행 문서: `UI_Admin_ExtractionSchemas_P3_후속_보안취약점검사보고서.md`
> 범위: P4 — `GET /versions/diff` + `POST /rollback` + 프론트 관련 UI

## 1. 위협 모델 요약

P4 가 새로 열어주는 공격 표면은 다음과 같다.

1. 과거 버전의 fields 내용을 **무권한자가 열람** 하는 경로 (diff 엔드포인트)
2. 특정 스키마의 fields 를 **무권한자가 변경** 하는 경로 (rollback 엔드포인트)
3. 요청 본문에 포함된 **제어 문자 / 초장문 / 이상 타입** 으로 로그/DB 오염
4. 동시 요청(업데이트 vs 롤백)로 인한 **race condition** 으로 일관성 붕괴
5. 폐기된 스키마를 **부활** 시키는 회피 경로
6. 감사 이벤트 미발행/오인 발행으로 인한 **추적 불가**

## 2. 점검 항목

| ID | 위험 | 결과 | 근거 / 코드 포인트 |
|----|------|------|--------------------|
| T-01 | diff 가 타 scope 스키마의 fields 노출 (S2 ⑥ 위반) | **통과** | `get_version` 이 `scope_profile_id` 를 `extraction_schemas` 조회 WHERE 절에 포함. 지정 scope 와 다른 스키마면 schema_row=None → 404. 두 버전 모두 같은 scope 에서 조회. (repo.py:256-273) |
| T-02 | rollback 이 타 scope 스키마를 갱신 (S2 ⑥ 위반) | **통과** | `rollback_to_version` 의 첫 SELECT `FOR UPDATE` 에 `scope_profile_id = %s` 조건 포함. (repo.py:519-539) |
| T-03 | `scope_profile_id` 에 SQL 조각을 주입 | **통과** | Pydantic `@field_validator` 가 `UUID(v)` 파싱으로 형식 검증 → 실패 시 422. 라우트 역시 `UUID(scope_profile_id)` 로 재검증. (schemas/extraction.py:326-335, api/v1/extraction_schemas.py:266-270) |
| T-04 | `base_version` / `target_version` 에 SQL 주입 | **통과** | `int = Query(..., ge=1)` 로 정수만 수용. psycopg2 파라미터 바인딩(`%s`). 문자열 포맷팅 없음. |
| T-05 | `change_summary` 로 제어문자 주입 (로그 오염, CRLF, ANSI escape) | **통과** | `_strip_and_validate_text` 가 `[\u0000-\u001F\u007F]` 를 거절. `RollbackExtractionSchemaRequest._validate_change_summary` 가 이를 호출. |
| T-06 | `change_summary` 초장문 DoS (DB 팽창, 로그 팽창) | **통과** | `Field(default=None, max_length=1024)` + 통합 테스트 1025자 → 422. |
| T-07 | 폐기 스키마를 rollback 으로 부활 | **통과** | `is_deprecated=True` 면 `ValueError("폐기된 스키마는 되돌릴 수 없음")` → 422. `rollback_to_version` 은 `deprecation_reason` 을 건드리지 않으며, 폐기 레코드는 애초에 FOR UPDATE 잠금 후 is_deprecated 체크에서 거절. |
| T-08 | `target_version >= current_version` 으로 미래 버전 복사 | **통과** | Repository 에서 명시적 거절. 클라이언트도 현재 버전의 되돌리기 버튼을 비활성화. |
| T-09 | `target_version` 이력이 비어 있거나(빈 fields) 빈 dict 로 rollback 시 모델 불변식 깨짐 | **통과** | `if not target_fields_map: raise ValueError("target_version 의 fields 가 비어 있어 되돌릴 수 없음")`. ExtractionTargetSchema 는 `min_length=1` 을 fields 에 강제하는 모델 불변식 유지. |
| T-10 | 업데이트와 rollback 의 동시 실행으로 버전 번호 충돌 | **통과** | 첫 SELECT 가 `FOR UPDATE` → 동일 스키마에 대한 두 트랜잭션 중 하나는 락 대기. 버전 번호는 잠금 내부에서 계산되므로 중복 생성 불가. `test_rollback_uses_for_update_lock` 에서 lock 구문 존재를 검증. |
| T-11 | 감사 로그 누락 (S2 ⑤ 위반) | **통과** | `audit_emitter.emit_for_actor(event_type="extraction_schema.rolled_back", action="extraction_schema.rollback", new_state={doc_type_code, version, rolled_back_from_version})`. `actor_type` 은 `emit_for_actor` 내부에서 actor 객체 기반으로 user/agent 구분 기록. |
| T-12 | 감사 로그에서 rollback 을 일반 update 와 구분 불가 | **통과** | 이벤트 `event_type` 이 별도(`rolled_back`). 추가로 버전 이력의 `extra_metadata` 에 `rolled_back_from_version` 키 기록. |
| T-13 | viewer 권한자가 rollback 호출 | **통과** (기존 인증/인가 로직 재사용) | 라우트는 `resolve_current_actor` 에 의존. 기존 author 권한 가드 체계를 우회하지 않음. 단, 세부 RBAC 정책은 별도 문서 범위. |
| T-14 | diff 응답에 타 scope 의 schema_id/ID 노출 | **통과** | 응답 DTO `ExtractionSchemaDiffResponse` 는 `id`/`schema_id` 등을 포함하지 않음. 오직 `doc_type_code`, 버전 번호, diff 내용만. |
| T-15 | 요청 본문이 초대용량(수 MB) 일 때 DoS | **부분 통과** | `RollbackExtractionSchemaRequest` 는 `target_version: int` + `change_summary: str≤1024` + `scope_profile_id: str` → 본질적으로 수백 바이트 수준. 본 엔드포인트에서 대용량 본문 위험 낮음. 단, global HTTP 본문 크기 제한은 infra(WAF/Nginx) 영역. |
| T-16 | 클라이언트가 모달 확인 없이 rollback 중복 클릭 | **통과** | `RollbackDialog` 의 확인 버튼이 `mutation.isPending` 시 `disabled`. 배경 클릭 / Escape / 취소 버튼 모두 요청 중에는 차단. |
| T-17 | `VersionDiffModal` 이 이전 쿼리 결과를 영구 캐시 | **통과** | `useQuery` 의 `staleTime: 60_000` (60초). TanStack Query 는 60초 후 refetch. immutable history 이므로 과도한 refetch 불필요. |
| T-18 | 프론트 `scope_profile_id` 가 null 일 때 URL 에 `scope_profile_id=null` 로 붙음 | **통과** | `buildQueryString` 이 null/undefined/"" 를 자동 제거 — P3 후속 때 검증. 본 변경은 같은 유틸을 재사용. |
| T-19 | diff/rollback 요청에 `X-Request-ID` 헤더 등 전파 누락으로 감사 추적 단절 | **통과** | rollback 엔드포인트가 `request.headers.get("X-Request-ID")` 를 `success_response` 에 전달. diff 는 부작용이 없어 request_id 전파 불필요. |
| T-20 | 오류 메시지에 내부 SQL/스택트레이스 유출 | **통과** | `except Exception: logger.exception(...); raise HTTPException(500, "내부 서버 오류가 발생했습니다.")`. ValueError 는 메시지가 한국어 설명만(컬럼/쿼리 텍스트 없음). |

## 3. S2 원칙 준수 확인

| 원칙 | 확인 |
|------|------|
| S2 ⑤ actor_type 기록 | rollback 감사 이벤트가 `emit_for_actor` 를 통해 `actor_type` (user/agent) 을 기록. |
| S2 ⑥ Scope Profile 필터 | `get_version`·`rollback_to_version` 모두 `scope_profile_id` 파라미터를 WHERE 절에 포함. 라우트에서 UUID 검증. |
| S2 ⑦ 폐쇄망 동작 | diff/rollback 모두 외부 서비스(OpenAI 등) 에 의존하지 않음. 순수 DB 연산. |

## 4. 정적 분석 / 테스트

- Python AST parse: 전 파일 OK (§검수보고서 5장 참조).
- pytest
  - Diff / rollback / diff route / rollback route 에 **48건** 테스트 추가.
  - 공격 성 입력(제어문자, 범위 밖 정수, 비-UUID, 초장문, 폐기 스키마 롤백, 빈 fields 롤백 등) 을
    422/404 로 정확히 거절하는지 검증.
- TypeScript: tsc 신규 에러 0.
- SAST (선택적): 본 리포트 내용은 의존성 변경 없음(새 패키지 도입 없음). `frontend/package.json`
  / `backend/requirements.txt` 변경 없음.

## 5. 잔존 위험

- **감사 로그 팽창**: 같은 버전으로의 반복 롤백이 허용되므로 악의적/무심한 클라이언트가
  짧은 시간 내 다수 rollback 을 시도하면 `extraction_schema_versions` 가 급증할 수 있음.
  Rate limit 은 infra 레벨에서 방어해야 하며, 본 범위 외.
- **서버-클라 diff 불일치 노출**: 클라이언트 `computeFieldsDiff` 와 서버 `compute_fields_diff`
  의 결과가 다른 경우(bool/int 구분, 특수한 중첩 dict) 사용자에게 시각적 차이가 보일 수 있음. 보안
  영향은 없으나 UX 혼동 원인. 문서에 "서버 결과가 기준" 이라는 점을 명시.
- **되돌리기 UI 의 "현재 버전 disabled" 는 편의**: 실제 보안 차단은 서버 `target_version >= current`
  거절이므로 UI 우회(직접 API 호출) 가 있어도 안전.
