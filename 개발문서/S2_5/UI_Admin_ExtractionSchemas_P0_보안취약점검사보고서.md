# 추출 스키마 관리 화면 — P0 개선 보안취약점 검사보고서

- 대상 경로: `/admin/extraction-schemas`
- 작성일: 2026-04-21
- 범위: S2 원칙 ⑤(actor_type 감사 로그), ⑥(Scope Profile ACL 슬롯), ⑦(폐쇄망 동등성) 준수 여부

## 1. 결론

P0 수정은 **보안 자세를 개선**한다. 구체적으로,

1. **감사 로그 누수 해소** — 기존 `emit_for_actor()` 4개 호출이 모두 `action=` 누락으로 F-07 이후 TypeError 를 일으켜 감사 이벤트가 **전혀 남지 않고** 500 으로 터졌다. 이번 수정으로 create/update/delete/deprecate 모두 정상 로깅된다.
2. **권한 에러 신호 복구** — `ExtractionSchemaAlreadyExistsError` 와 `ExtractionSchemaNotFoundError` 가 `KeyError/ValueError` 를 상속하지 않아 라우터가 잡지 못하고 **500 으로 탈출**하고 있었다. 이제 각각 409/404 로 매핑되어 공격자/사용자 모두에게 정확한 신호가 나간다.
3. **S2 ⑥ Scope Profile ACL 강화** — `get_by_doc_type` 에 `scope_profile_id` 파라미터를 추가해 단건 조회도 프로파일 필터링이 가능해졌다 (기존에는 `list_all` 만 지원).

새로 도입된 위험은 없으며, 아래 §3–§6 에서 잠재적 우려사항을 항목별로 검증한다.

## 2. 위협 모델 요약

| 자산 | 위협 | 기존 상태 | P0 후 상태 |
| ---- | ---- | --------- | ---------- |
| 추출 스키마(DocumentType별 정의) | 무권한 열람/수정 | Scope ACL 슬롯은 있으나 실제 단건 조회에 필터 미적용 | `get_by_doc_type(scope_profile_id=…)` 지원 |
| 감사 로그(S2 ⑤) | 쓰기 행위의 기록 누락 | **모든 쓰기 경로에서 감사 로그 미기록 + 500** | 정상 기록 + 200/204 |
| 스키마 생성 중복 | 409/500 혼선 | `ValueError` catch 가 실제로 발생하는 예외를 잡지 못해 500 으로 노이즈 증가 | 정확히 409 로 응답 |
| UI 정보 노출 | MOCK 데이터 오표시 | 서버 오류를 MOCK 으로 덮어 사용자가 "잘 됨" 으로 오인 | `ErrorBanner` 가 HTTP/네트워크 오류를 구분해 표시 |

## 3. S2 원칙 ⑤ (actor_type 감사 로그) 준수

### Before — 감사 누수

```python
# extraction_schemas.py (정정 전)
audit_emitter.emit_for_actor(
    actor=actor,
    event_type="extraction_schema.created",
    resource_type="extraction_schema",
    resource_id=str(schema.id),
    new_state={...},
)  # F-07 이후 action= 누락 → TypeError → except Exception 로 500 반환
```

→ 이 경로는 `create()` 성공 직후 호출되므로 **데이터는 기록되지만 감사는 남지 않는다**. 공격자 입장에서 이는 스키마 변조의 완전한 은폐 기회이며, 감사 누수에 해당.

### After — 감사 복구

4 × `emit_for_actor(... action=…)` 로 정정:

| 엔드포인트 | `event_type` | `action` (신규) |
| ---------- | ------------ | --------------- |
| `POST /extraction-schemas` | `extraction_schema.created` | `extraction_schema.create` |
| `PUT /extraction-schemas/{doc_type}` | `extraction_schema.updated` | `extraction_schema.update` |
| `DELETE /extraction-schemas/{doc_type}` | `extraction_schema.deleted` | `extraction_schema.delete` |
| `PATCH /extraction-schemas/{doc_type}/deprecate` | `extraction_schema.deprecated` | `extraction_schema.deprecate` |

`emit_for_actor` 내부에서 `actor.actor_type` → `"user" | "agent"` 로 Literal 매핑되므로 **S2 원칙 ⑤ 의 actor_type 필드가 항상 기록**된다.

## 4. S2 원칙 ⑥ (Scope Profile ACL 슬롯) 강화

`get_by_doc_type` 리포지토리 메서드에 `scope_profile_id: Optional[UUID]` 추가. `None` 이면 기존 동작과 동일(전체 scope 열람 — 관리자 경로). 지정 시 `AND scope_profile_id = %s` 로 SQL 필터.

라우터 `GET /extraction-schemas/{doc_type}` 는 현재 `scope_profile_id` 쿼리 파라미터를 받지는 않지만, **API 레이어에서 ACL 을 추후에 추가할 슬롯**이 정비됐다. S2 원칙 ⑥ 은 "관리자 설정으로 관리, 코드에 scope 문자열 등장 금지" 를 요구하며, 이 패치는 문자열을 등장시키지 않고 UUID 파라미터로만 다룬다.

### 신규 `GET /extraction-schemas` 목록 엔드포인트의 ACL

- `scope_profile_id` 쿼리 파라미터를 받아 UUID 파싱 후 `list_all(scope_profile_id=…)` 에 전달.
- UUID 파싱 실패 시 `422 Unprocessable Entity` 로 거절 — SQL 인젝션 경로로 회귀하지 않음.
- 기본 `include_deleted=False`, `is_deprecated` 미지정 시 전체 반환. 소프트 삭제된 스키마는 기본적으로 숨김.

## 5. SQL 인젝션 / 파라미터 안전성

- 신규 `list_extraction_schemas` 는 기존 `ExtractionSchemaRepository.list_all()` 을 재사용. 해당 메서드는 이미 **prepared statement(`%s`)** 만 사용하며, `where` 는 리스트를 `" AND ".join` 한 정적 조건문 조합(사용자 입력을 SQL 식으로 엮지 않음).
- `scope_profile_id` 는 `UUID(str)` 로 1차 파싱 후 `str()` 로 바인딩. 문자열 그대로 SQL 에 삽입되는 경로 없음.
- 수정된 `get_by_doc_type` 도 동일 패턴 유지. `include_deprecated` 는 `bool` 플래그로 SQL 조건 추가 여부만 제어하므로 파라미터 주입 경로 없음.

## 6. S2 원칙 ⑦ (폐쇄망 동등성)

- 신규/변경된 코드 경로는 외부 네트워크 호출을 추가하지 않음.
- 프론트 `NetworkError`/`ApiError` 는 자체 정의 클래스이며 API_BASE 가 폐쇄망 주소로 설정되어도 동작함.
- `ErrorBanner` 는 브라우저 fetch 오류 메시지를 그대로 표시하되 외부 주소는 포함되지 않음 (요청한 자체 `API_BASE` 만 노출).

## 7. 정보 노출 축소 vs 증가 분석

### 축소된 정보 노출 (보안상 이점)

- MOCK 데이터 제거 — 과거에는 API 가 404/500 이어도 가상의 "contract/invoice/report" 스키마가 노출되어, **실제 설치에 없는 DocumentType 을 암시**하는 부작용이 있었다. 이제는 실제 데이터만 표시된다.
- 예외 응답 정확도 — `ExtractionSchemaAlreadyExistsError` 는 이제 409 로, `ExtractionSchemaNotFoundError` 는 404 로 매핑된다. 이는 사용자 신호에는 좋지만 **열거(enumeration) 공격에도 유용**하다. 이에 대한 완화는 §8 참조.

### 증가된 정보 노출 (추가 검증 필요)

- 상세 패널이 `scope_profile_id`, `created_by`, `updated_by`, `created_at`, `updated_at` 를 표시하게 됨.
    - **판정 — 허용**. 이 화면은 관리자(/admin) 경로이며, 감사/추적 기능의 일부다. 이들 필드는 공격자가 얻어도 추가 권한을 주지 않는다.
- `extra_metadata` 는 현재 렌더링하지 않음 (관리자용 자유 형식). 노출되는 항목만 UI 에 표시.

## 8. Enumeration 완화 — 현재 상태

- `GET /extraction-schemas/{doc_type}` 가 존재/미존재를 404 로 구분하므로 doc_type_code 열거가 가능하다.
- 단, **이 경로는 관리자 인증을 요구**하며(`resolve_current_actor`), 추출 스키마의 doc_type_code 는 본질적으로 기업 도메인 어휘(예: contract, invoice)로 공개 정보에 가깝다.
- **판정 — 저위험 수용**. 일반 사용자 경로에서 이 엔드포인트가 노출되면 안 된다는 조건은 기존 인증 가드가 담당한다.

## 9. 프론트엔드 XSS / DOM 주입 점검

- 모든 표시 필드는 React 의 기본 escape 를 거침 (`{field_name}`, `{schema.deprecation_reason}` 등).
- `new Date(...).toLocaleString("ko")` 는 문자열 포맷팅만 수행 — 서버가 잘못된 형식을 보내도 브라우저 레벨에서 예외 발생할 뿐 주입 경로 없음.
- `dangerouslySetInnerHTML` 미사용. `aria-label`, `title` 도 바인딩된 값 그대로 사용하며 HTML 로 해석되지 않음.

## 10. CSRF / 인증 회귀

- API 클라이언트(`api.get/post/put/patch/delete`)는 기존 `Bearer Token` 인터셉터를 공유. P0 패치는 클라이언트 라우팅(URL)만 변경했으며 인증 헤더 처리 로직은 건드리지 않음.
- 동일 origin 하에서 `credentials: "include"` 로 쿠키 동반 — 기존 대비 회귀 없음.

## 11. 의존성/라이선스

- 신규 라이브러리 도입 없음. `@tanstack/react-query`, React, lucide 아이콘 세트 등 기존에 이미 사용 중인 패키지만 참조.

## 12. 체크리스트

| 항목 | 결과 |
| ---- | ---- |
| 신규 외부 네트워크 경로 | 없음 (S2 ⑦ 유지) |
| 신규 문자열 기반 scope 비교 | 없음 (S2 ⑥ 유지) |
| 신규 DocumentType 하드코딩 | 없음 (S1 ① 유지; 오히려 `extraction_mode` 제거로 강화) |
| SQL 파라미터 바인딩 | `%s` prepared statement 유지 |
| 감사 로그 actor_type 기록 | 4 쓰기 경로 복구 (S2 ⑤) |
| XSS / DOM 주입 | 기존 React escape 만 의존 |
| 인증/세션 처리 | 변경 없음 |
| 라이브러리 신규 도입 | 없음 |

## 13. 후속 보안 작업 제안

- [P1] `GET /extraction-schemas/{doc_type}` 에 `scope_profile_id` 쿼리 파라미터를 라우터 시그니처로 받아, 리포지토리의 새 슬롯을 실제로 사용하도록 연결.
- [P2] 404 enumeration 이 문제가 되는 환경(공개 관리 콘솔 등)에서는 인증되지 않은 요청을 401 로 먼저 차단하는 가드가 `resolve_current_actor` 이전 단계에서 확실히 동작하는지 컴포즈별 점검.
- [P3] 감사 로그 retention 에 `extraction_schema.*` 액션이 S2 ⑤ 보존 정책에 포함되는지 확인.
