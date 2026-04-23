# 추출 스키마 관리 화면 — P1 개선 보안취약점 검사보고서

- 대상 경로: `/admin/extraction-schemas`
- 작성일: 2026-04-21
- 범위: P1-A (생성·편집·폐기·삭제 UI) + P1-B (버전 이력) + P1-C (단건 조회 `scope_profile_id`) + P1-D (포커스 트랩)

## 1. 결론

P1 은 **보안 자세를 추가로 강화**한다.

1. **S2 ⑥ Scope Profile ACL 슬롯 실사용** — P0 에서 준비만 되어 있던 단건 조회 `scope_profile_id` 쿼리가 라우터에 실제로 연결됨. UUID 파싱 실패는 422 로 차단.
2. **쓰기 경로 감사 로그 실 호출** — P0 에서 계약(F-07 `action=`)을 맞춘 4개 `emit_for_actor` 경로가 P1-A 쓰기 UI 에서 최초로 실제 호출된다. 생성/수정/폐기/삭제 모두 `actor_type` 과 함께 기록.
3. **파괴적 액션 확정 플로우** — 삭제는 공용 `ConfirmDialog` (destructive), 폐기는 사유 textarea 를 요구하는 `DeprecateDialog` 로 의도치 않은 1-클릭 파괴를 방지.
4. **포커스 트랩 도입 (P1-D)** — 모달 내에서 Tab 이 빠져나가지 않고, 닫힐 때 트리거 요소로 포커스가 복귀. 스크린리더 사용자가 모달 밖 맥락으로 잘못 이동하는 UX 취약점 제거.

새로 도입된 보안 위험은 없다. 아래 §3–§10 에서 항목별로 점검한다.

## 2. 위협 모델 델타 (P0 대비)

| 자산 | 위협 | P0 상태 | P1 상태 |
| ---- | ---- | ------ | ------ |
| 추출 스키마 단건 | Scope 경계 위반 열람 | 라우터에 `scope_profile_id` 파라미터 없음 → 슬롯만 존재 | 라우터에 쿼리 추가 + UUID 파싱 422 |
| 감사 로그 (S2 ⑤) | 쓰기 행위 미기록 | 계약 수정됨 but 쓰기 UI 부재로 **실사용 경로 0** | 실 UI → 실제 `emit_for_actor` 호출 (4 action 모두) |
| 파괴적 액션(폐기/삭제) | 사고성 클릭 | 미구현 | Confirm + Deprecate 두 다이얼로그로 분기 |
| 접근성(모달 포커스) | Tab 탈출 / 포커스 유실 | 없음 (배경 클릭 / Esc 정도만) | useFocusTrap 으로 wrap + 복귀 |
| JSON textarea 입력 | XSS 유입 경로 | 없음 | 클라이언트 파싱 후 React 속성 바인딩으로 렌더 |

## 3. P1-C — `GET /extraction-schemas/{doc_type}` `scope_profile_id` 연결

### 3.1 SQL 안전성
- 리포지토리 `get_by_doc_type()` 는 이미 `%s` prepared statement 만 사용. `scope_profile_id` 는 `UUID → str()` 로 바인딩.
- 라우터에서 쿼리 문자열을 받은 즉시 `UUID(scope_profile_id)` 로 1차 파싱하여 실패 시 **422 Unprocessable Entity** 로 거절. 그대로 SQL 에 흘려 보내지 않는다.
- `include_deprecated` 는 `bool` 플래그로 SQL 조건 포함 여부를 제어하므로 파라미터 주입 경로 없음.

### 3.2 정보 노출
- `scope_profile_id` 미지정 시 **기존 관리자 전역 조회 동작 유지** — 신규 열람 범위 확장 없음.
- 지정 시 해당 Scope 에 속하지 않는 스키마는 `None` → 404 로 반환. 타 Scope 의 존재 자체는 드러나지 않음 (doc_type 열거는 P0 에서도 저위험으로 수용된 바 있다).

### 3.3 S2 ⑥ 원칙 적합성
- 코드에 scope 문자열(`"team"`, `"public"`) 등장 없음. UUID 만 다룬다.
- 단건 조회에서도 관리자 설정 기반 ACL 이 적용될 수 있게 슬롯이 "실제로 호출되는" 상태로 승격.

## 4. P1-A — 쓰기 UI 가 만드는 새 공격면

### 4.1 CreateSchemaModal
- `doc_type_code`: 클라이언트측 regex `^[a-zA-Z][a-zA-Z0-9_-]*$` 검증. 서버에도 동일 패턴의 `field_code` 검증이 `ExtractionFieldDef` 에 존재하므로 bypass 시에도 DB 레벨 제약이 최종 방어선.
- `scope_profile_id`: 길이 ≥32 사전 체크 + 서버 `UUID()` 파싱(422 반환) 최종 검증. 빈 문자열은 `scope_profile_id=null` 로 전송.
- `fields`: textarea 로 JSON 을 받되 `parseFieldsJson` 이 (a) `JSON.parse` 실패 → 메시지, (b) 객체가 아닌 경우 → 거절, (c) 각 필드의 `field_name/field_type/description` 존재·타입 검증 → 거절. 서버에 도달하기 전에 기본 구조가 강제된다.
- 서버 역시 `ExtractionSchemaRepository.create` 에서 `field_name`, `field_type`, `description` 유효성과 중복 체크를 수행하므로 프론트 검증이 bypass 돼도 서버 경고(409/422) 로 귀결.

### 4.2 SchemaDetailPanel 편집 모드
- 편집은 `PUT /{doc_type}` 이며, 서버는 **새 버전 row 를 생성** 하고 `changed_fields` 를 계산해 저장. 과거 버전은 변경 불가.
- `change_summary` 는 `maxLength=1024` 제약(HTML + 서버 재검증 가능). XSS 는 DB 에 저장되지만 렌더 시 React 의 기본 escape 를 거쳐 안전.

### 4.3 위험 구역 (폐기/삭제)
- **삭제**는 `ConfirmDialog` (destructive) 로 사용자가 명시 확인해야 실행 — `DELETE /{doc_type}`.
    - 소프트 삭제이며 감사 기록(`extraction_schema.deleted` + `actor_type`)은 유지되므로 "되돌아올 수 있는 파괴" 모델을 유지.
- **폐기**는 `DeprecateDialog` 에서 `reason` textarea 가 `reason.trim().length > 0` 일 때만 버튼 활성 — 사유 미입력 폐기 차단.
    - 서버는 `DeprecateExtractionSchemaRequest.reason` 을 `min_length=1, max_length=1024` 로 검증 (Pydantic 모델).

### 4.4 감사 로그 (S2 ⑤)
- P0 에서 맞춘 4개 action(`extraction_schema.create/update/delete/deprecate`) 이 **P1 에서 최초로 실 UI 에서 호출**된다. `actor_type` 은 `resolve_current_actor` → `ActorContext` 경로로 "user"/"agent" 로 매핑되어 항상 기록.

## 5. P1-B — 버전 이력 조회

### 5.1 정보 노출 검토
- 응답 필드: `id / schema_id / version / fields / is_deprecated / deprecation_reason / change_summary / changed_fields / created_at / created_by`.
- `created_by` 는 actor_id 문자열 — 관리자 경로에서 감사 가시성 제공 목적이며, 외부 유출 시 권한 상승에 직접 기여하지 않는다.
- `fields` 전체 JSON 이 이력에도 포함되지만, 이는 관리자용이며 `extra_metadata` 는 이력 응답에 포함되지 않음.

### 5.2 렌더 안전성
- `change_summary`, `changed_fields` 배열 모두 React 텍스트 바인딩. chip 컴포넌트는 inline `<span>` 로 렌더되며 `dangerouslySetInnerHTML` 미사용.
- 날짜 포맷은 `new Date(...).toLocaleString("ko")` 문자열 처리 — 서버가 잘못된 포맷을 보내도 주입 경로 없음.

## 6. P1-D — 포커스 트랩 훅

### 6.1 탈취 방지
- `useFocusTrap` 은 키보드 이벤트만 다루며 **외부 데이터 바인딩 없음**. `document.activeElement` 를 참조해 복귀하고 DOM 에 여전히 붙어 있는지 `document.contains` 로 검증.
- 요소가 제거된 경우 조용히 스킵 — `focus()` 를 null 에 호출하지 않음. UAF 패턴 없음.

### 6.2 접근성 관점의 보안 가치
- WCAG 2.4.3 (Focus Order) / 2.4.11 (Focus Not Obscured) 보완. 스크린리더 사용자가 파괴적 액션 모달 밖 컨텍스트로 잘못 포커스가 튀어 의도치 않은 클릭을 트리거하는 시나리오 차단.
- 포커스 복귀는 "트리거 버튼 → 모달 → 닫기 → 트리거 버튼" 의 가역 흐름을 보장, 마우스 없는 사용자가 삭제/폐기 의사결정 후 위치를 잃지 않도록 함.

## 7. XSS / DOM 인젝션 재검토

- 모든 사용자 입력(`doc_type_code`, `scope_profile_id`, `fieldsText`, `changeSummary`, `deprecationReason`) 은 React 의 텍스트/속성 바인딩만 사용.
- 서버에서 돌아오는 `deprecation_reason`, `change_summary`, `field_name`, `description` 등도 동일.
- `dangerouslySetInnerHTML` / `innerHTML` 직접 설정 경로 없음.
- `<textarea>`, `<input>` 의 `value` 는 controlled — DOM 직접 조작 없음.

## 8. CSRF / 인증 회귀

- 모든 mutation 은 공용 `api.get/post/put/patch/delete` 래퍼를 사용. P0 부터 유지된 `Bearer Token` 인터셉터 + `credentials: "include"` 와 동일.
- 신규 엔드포인트는 없다 — 기존 백엔드 스펙(`POST/PUT/PATCH/DELETE`) 을 UI 가 처음으로 호출할 뿐.
- 폐기/삭제가 `GET` 이 아닌 **안전하지 않은 HTTP 메서드** 로 라우팅되어 CSRF 선결제 플리싱에 취약하지 않다.

## 9. 의존성 / 라이선스

- 신규 라이브러리 도입 없음. `@tanstack/react-query`, React, 내부 `useMutationWithToast`, `ConfirmDialog`, `ErrorBanner` 재사용.
- `useFocusTrap` 은 자체 구현(외부 패키지 미사용) — `focus-trap-react` 같은 3rd-party 대비 공급망 위험 감소.

## 10. 폐쇄망 동등성 (S2 ⑦)

- P1 에서 추가된 경로는 외부 네트워크를 호출하지 않음.
- `useFocusTrap` 은 브라우저 표준 DOM API (`document.activeElement`, `querySelectorAll`, `addEventListener`) 만 사용. 폐쇄망 브라우저에서도 동일 동작.
- `ErrorBanner` 가 `NetworkError` 발생 시 표시하는 URL 은 자체 `API_BASE` — 외부 도메인이 섞이지 않는다.

## 11. Enumeration / Information Disclosure 재평가

- 라우터 `GET /{doc_type}` 는 여전히 존재 여부를 404/200 으로 구분. P0 보고서에서 "관리자 인증 가드가 방어" 로 저위험 수용한 상태가 P1 에서도 유지.
- `scope_profile_id` 지정 시 엄밀한 Scope 경계 밖 스키마는 `None` 을 반환하므로, 자신의 Scope 를 벗어난 doc_type 의 존재는 드러나지 않음(null 과 404 가 동일 코드 경로로 귀결).
- 버전 이력 엔드포인트는 doc_type 이 존재할 때만 목록 반환 — 미존재 doc_type 은 빈 배열이지만, 이는 `get_by_doc_type` 단건 404 로 사실상 감지 가능 (기존 저위험 수용 범위).

## 12. 체크리스트

| 항목 | 결과 |
| ---- | ---- |
| 신규 외부 네트워크 경로 | 없음 (S2 ⑦ 유지) |
| 신규 scope 문자열 하드코딩 | 없음 (S2 ⑥ 유지, UUID 만) |
| 신규 DocumentType 하드코딩 | 없음 (S1 ① 유지 — `parseFieldsJson` 이 doc_type 별 분기 없이 동작) |
| SQL 파라미터 바인딩 | `%s` prepared statement 유지 |
| 감사 로그 actor_type 기록 | 실 UI → 4개 action 실제 호출 (S2 ⑤) |
| XSS / DOM 주입 | 없음 (React escape + dangerouslySetInnerHTML 미사용) |
| 파괴적 액션 확정 플로우 | ConfirmDialog / DeprecateDialog 두 단계 |
| 인증/세션 처리 | 변경 없음 |
| 라이브러리 신규 도입 | 없음 |
| 포커스 트랩 외부 이벤트 리스너 누수 | 없음 (`useEffect` cleanup 에서 `removeEventListener`) |

## 13. 후속 보안 작업 제안

- [P2] `DeprecateDialog` / `CreateSchemaModal` 의 reason/description 텍스트에 서버측 길이·문자 조합 제약을 Pydantic validator 수준에서 추가 검증. 클라이언트 우회 시 서버 방어.
- [P2] 버전 이력 조회에도 `scope_profile_id` 쿼리를 받아 타 Scope 의 이력을 조회할 수 없도록 가드. 현재는 doc_type 단위로 가드.
- [P3] `useFocusTrap` 에 포털(portal)된 자식 컨테이너를 고려한 테스트 케이스 추가 — 현재 본 페이지에서는 포털 사용이 없으나, 다른 모달에서 재사용할 때를 대비.
- [P3] 감사 로그 retention 에 `extraction_schema.*` 4개 action 이 S2 ⑤ 보존 정책에 포함되는지 점검 (P0 후속과 동일 문맥).
