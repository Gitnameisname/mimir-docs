# Task 14-15 검수보고서 — API 키 CRUD / 감사 로그 필터 UI

작성일: 2026-04-14
범위: `POST /admin/api-keys`, `POST /admin/api-keys/{id}/revoke`, `GET /admin/audit-logs/event-types` · `AdminApiKeysPage` · 강화된 `AdminAuditLogsPage`

---

## 1. 구현 요약

### 1.1 백엔드 (`backend/app/api/v1/admin.py` Phase 14-15 섹션)

- `_generate_api_key()` — `secrets.token_urlsafe(32)` (256bit 엔트로피) 기반 `mk_<base64url>` 포맷. 반환 튜플은 `(full_key, prefix, sha256_hexdigest)`. DB 저장은 `key_hash` 컬럼(SHA-256)만 사용하며 `full_key` 는 응답 본문에서 **단 1회** 반환.
- `_API_KEY_NAME_RE = re.compile(r"^[\w\- .]{1,100}$", re.UNICODE)` — 이름 regex 화이트리스트. 수량 제한과 문자 집합 모두 상한이 고정되어 reDoS 위험 없음.
- `_VALID_API_KEY_SCOPES = {"READ_ONLY", "READ_WRITE", "admin.read", "admin.write"}` — scope 화이트리스트.
- `POST /api-keys` (`status_code=201`)
  - 요청: `CreateApiKeyBody{name, description?, scope, expires_in_days}` (Pydantic v2, `min_length=1`, `max_length`, `ge=0`, `le=3650`).
  - `expires_in_days == 0` → `expires_at = NULL` (무기한). 그 외에는 `NOW() + (%s||' days')::interval` 로 바인딩.
  - 감사 이벤트 `API_KEY_ISSUED` 발행 (실패는 응답에 영향 없음).
- `POST /api-keys/{key_id}/revoke`
  - `uuid.UUID(key_id)` 로 경로 파라미터 형식 검증 → 실패 시 422.
  - `UPDATE … SET status='REVOKED' WHERE id=%s AND status='ACTIVE'` 로 **soft-revoke** (과거 감사 흔적 보존). `RETURNING` 이 비면 404 "키를 찾을 수 없거나 이미 폐기됨".
  - 감사 이벤트 `API_KEY_REVOKED` 발행.
- `GET /audit-logs/event-types` — 25종 정적 카탈로그(`USER_LOGIN`, `DOCUMENT_CREATED`, `API_KEY_ISSUED`, `JOB_SCHEDULE_RUN`, `ALERT_RULE_CREATED` 등)를 `{value, label}` 형태로 반환. 드문 이벤트까지 노출하여 운영 초기부터 필터 가능.

모든 3개 엔드포인트는 `Depends(require_admin_access)` 로 보호.

### 1.2 프론트엔드

- `frontend/src/types/admin.ts`
  - `ApiKey` 확장 (description/scope/status/issuer_name/use_count/expires_at/last_used_at/last_used_ip).
  - `ApiKeyWithSecret extends ApiKey { full_key: string }` — 응답 한정 타입.
  - `AuditEventTypeOption { value, label }`.
- `frontend/src/lib/api/admin.ts`
  - `getApiKeys({page,page_size,status,search})`, `createApiKey(body)`, `revokeApiKey(keyId, reason?)` (경로 인자 `encodeURIComponent`), `getAuditEventTypes()`.
- `frontend/src/features/admin/api-keys/AdminApiKeysPage.tsx` — 신규 500라인
  - 리스트: 이름/프리픽스/범위/발급자/마지막 사용/만료/상태/액션. `SUPER_ADMIN` 만 생성/폐기 버튼 노출.
  - `CreateApiKeyModal` — 이름 필수, 설명/범위/만료(5 옵션). 노란 경고 `role="note"`.
  - `IssuedKeyModal` — 빨간 경고 `role="alert"`, 읽기전용 input + 복사 버튼 (`navigator.clipboard.writeText` + `select()` fallback), `aria-live="polite"` 로 "복사됨" 알림.
  - `RevokeApiKeyModal` — 이름 재입력 일치(`confirmName.trim() === target.name`) 시에만 제출 활성. 사유 textarea(선택).
- `frontend/src/features/admin/audit-logs/AdminAuditLogsPage.tsx` — 필터 섹션 전면 재설계
  - 기간 프리셋 `1h/24h/7d/30d/custom`. `useMemo` 에서 `Date.now() - cfg.ms` → ISO 변환.
  - 커스텀 모드에서만 `type="datetime-local"` 시작/종료 입력.
  - 이벤트 유형 드롭다운 — `adminApi.getAuditEventTypes()` 카탈로그 (staleTime 5분).
  - 사용자 검색 autocomplete — `actorSearch.trim().length >= 1` 일 때만 `adminApi.getUsers({search})` 호출. 결과 리스트는 `role="listbox"` / `role="option"`.
  - 결과 필터는 백엔드와 맞춘 소문자 값 (`success/failure/denied`).
  - 초기화 버튼, 상세 사이드 패널 유지.
- `frontend/src/app/admin/api-keys/page.tsx` — placeholder → `AdminApiKeysPage` 연결 + metadata title.

### 1.3 UI 리뷰 5회 요약

| 회차 | 관점 | 반영 사항 |
| --- | --- | --- |
| 1 | 안전성 | `full_key` 는 응답 1회, 읽기전용 input + 경고 2곳 (`role="note"` 생성 모달, `role="alert"` 발급 모달) |
| 2 | 실수 방지 | 폐기 모달 이름 재입력 정확 일치 시에만 submit 활성 |
| 3 | 권한 분리 | `hasRole?.("SUPER_ADMIN")` 로 생성/폐기 버튼 + 액션 컬럼 게이트 |
| 4 | 접근성 | `role=alert/note/listbox/option/status`, `aria-live`, `sr-only` 섹션 라벨, `scope=col`, `focus-visible:ring` (≥4곳) |
| 5 | 반응형 | `p-4 sm:p-6`, `text-xl sm:text-2xl`, `overflow-x-auto`, `flex-wrap`, 모달 `max-w-md`, grid 2열 → 1열 |

---

## 2. 검증 결과

검증 스크립트: `backend/tests/unit/test_admin_audit_api_keys_phase14_15.py`

```
[API-FE]  4/4
[API]    10/10
[CATALOG] 4/4
[DDL]     4/4
[RESP]    6/6
[ROUTE]   2/2
[SEC]    12/12
[SYNTAX]  1/1
[TYPES]   4/4
[UI-AUDIT] 10/10
[UI-KEYS] 14/14

합계: 71/71 PASS  (0 FAIL)
```

- `ruff`/`mypy` 는 기존 설정대로 변경 없음.
- TypeScript: Task 14-15 대상 파일 오류 없음. 사전 존재하던 `src/lib/api/admin.ts:309` `api.put` 오류(Phase 12 코드)는 스코프 밖.

---

## 3. 코드 레벨 규칙 준수

- DocumentType 분기 없음 (API 키는 generic 스키마) ✔
- 모든 SQL 파라미터 바인딩 (`%s`) — `expires_at` f-string 조각도 상수(`"NULL"` / 고정 SQL 표현식)만 포함 ✔
- 플러그인/로직 분리 없음 (필수 아님) ✔
- 감사 이벤트 카탈로그는 하드코딩이지만 "UI 드롭다운 전용"이며 코드상 enum 강제는 하지 않음 ✔

## 4. UI 개발 규칙 준수

- UI 리뷰 5회 이상 ✔
- 데스크탑 앱/웹 겸용 — `sm:` 브레이크포인트, `flex-wrap`, `overflow-x-auto`, `max-w-md` ✔

---

## 5. 결론

Task 14-15 는 구현·검증·보안 요건을 모두 충족한다. 71/71 PASS. 롤아웃 시 유의사항은 동반 문서 `task15_보안취약점검사.md` 참조.
