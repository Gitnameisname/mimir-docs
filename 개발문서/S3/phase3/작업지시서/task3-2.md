# task3-2 — 열람자 노출 제어 (Scope Profile `expose_viewers`)

**작성일**: 2026-04-27
**Phase / FG**: S3 Phase 3 / FG 3-2
**상태**: 착수 대기 (FG 3-1 의 `include_viewers` 메커니즘 완료 후)
**담당**: 백엔드 + 프런트 (관리자 UI)
**예상 소요**: 1.5~2일
**선행 산출물**:
- `task3-1.md` — `GET /documents/{id}/contributors?include_viewers=` 메커니즘 완료
- Phase 3 개발계획서 §FG 3-2

---

## 1. 작업 목적

"누가 이 문서를 봤는가" 는 정보보안 / 개인정보 / 조직문화 측면에서 민감하다. 따라서 **Scope Profile 단위 설정**으로 열람자 노출을 on/off 가능하게 한다.

기본값은 보수적으로 **off (false)**. 관리자가 명시적으로 켜야만 해당 Scope 의 사용자가 다른 사용자의 열람 흔적을 본다. **사용자 의지에 의한 on 도 Scope Profile 정책에 의해 거부 가능** — 즉 FG 3-1 의 `include_viewers=true` 쿼리는 정책 우선순위로 강제 false 가 될 수 있다.

---

## 2. 작업 범위

### 2.1 포함

1. **DB 스키마 변경 (Alembic revision `s3_p3_scope_profiles_settings_json`)**
   - `scope_profiles` 테이블에 `settings_json JSONB NOT NULL DEFAULT '{}'::jsonb` 추가
   - 기존 row 는 `{}` 로 채워짐 (boolean 키 부재 = false)
   - 인덱스 불필요 (Profile 수가 적고 read-heavy 가 아님)
   - **헌법 §46 결합**: 본 마이그레이션은 P1. 사람 승인 경로 명시

2. **모델 / Repository 확장**
   - `backend/app/models/scope_profile.py` dataclass 에 `settings: ScopeProfileSettings` 필드 추가
   - `ScopeProfileSettings` 는 별도 dataclass: `expose_viewers: bool = False` (Phase 3 후속 설정 키 추가 가능하도록 컨테이너 분리)
   - `scope_profiles_repository` 의 read/upsert 가 `settings_json` 직렬화/역직렬화 담당
   - 알 수 없는 키는 **무시 + 보존** (forward compatibility) — load 시 dataclass 필드만 추출, save 시 raw dict 와 머지

3. **정책 게이트 — Contributors API 결합**
   - `services/contributors_service.get_contributors()` 진입부:
     ```python
     scope_settings = scope_profile_settings_for(viewer_actor)
     effective_include_viewers = include_viewers and scope_settings.expose_viewers
     ```
   - `effective_include_viewers=False` 면 viewers 섹션을 응답에서 **제거** (FG 3-1 의 옵셔널 키 처리 그대로)
   - **fail-closed**: scope settings 조회 실패 / settings_json 파싱 실패 시 false 로 강제 (기본 보수)
   - 응답 헤더에 `X-Viewers-Hidden-Reason: scope-policy` 추가 (디버깅 / 관리자 UI 가 사유 표시 가능하게). 일반 사용자에게는 영향 없음.

4. **다른 viewer-노출 경로 전수 점검**
   - audit_events 기반의 어떤 다른 API/패널이 viewer 정보를 노출하는지 grep:
     - `audit_events` + `event_type='document.viewed'` 사용처
     - admin UI 의 활동 타임라인, RAG citation panel, history panel 등
   - 정책 우회 가능 경로(직접 audit_events query 등) 가 발견되면 **본 FG 의 게이트 함수**(`should_expose_viewers(viewer_actor) -> bool`)를 호출하도록 통일
   - 산출물: `FG3-2_viewer노출경로_점검표.md`

5. **관리자 UI**
   - `frontend/src/app/admin/scope-profiles/[id]/page.tsx` 의 설정 섹션에 토글 추가
   - 토글 라벨: "이 Scope 사용자에게 다른 사용자의 열람 흔적을 표시"
   - 보조 설명: "off (기본값) 시 Contributors 패널의 '최근 열람자' 섹션이 숨겨집니다."
   - 토글 변경 → `PATCH /admin/scope-profiles/{id}` 로 `settings.expose_viewers` 갱신
   - 변경 직후 audit_events 에 `scope_profile.settings.changed` emit (감사 추적)

6. **백엔드 API — 관리자 전용**
   - `PATCH /admin/scope-profiles/{id}` body 에 `settings: { expose_viewers: bool }` 허용
   - `require_admin` 가드 + 본인 조직의 profile 만 변경 가능
   - validation: settings 는 dataclass schema 통과한 키만 저장. 미지의 키는 raw dict 에 보존 (위 §2.2 forward compat)

### 2.2 제외

- 다른 설정 키들 (예: `allow_agent_actions`, `default_visibility`) — 본 FG 는 `expose_viewers` 1 키만. 컨테이너만 마련
- 사용자 개인 단위 opt-out — 사용자가 자기 view 흔적을 가리는 기능은 S4 검토
- 조직 단위 일괄 적용 — Phase 3 범위 밖. 관리자가 Scope 별로 개별 설정

### 2.3 하드코딩 금지

- 토글 라벨/설명은 i18n 키
- default 값 false 는 dataclass 의 default 1 곳에서만 정의 (라우터/UI/마이그레이션이 각자 false 를 적지 않음)

---

## 3. 선행 조건

- task3-1 완료 — `include_viewers` 메커니즘이 응답 스키마/프런트에서 옵셔널로 동작
- `scope_profiles` 테이블 안정 (S2 종결 상태)
- 관리자 UI 의 Scope Profile 편집 페이지 골격 존재 (S2 Phase 6)

---

## 4. 구현 단계

### Step 1 — 스키마 / 마이그레이션

1. Alembic revision `s3_p3_scope_profiles_settings_json` 작성
2. dry-run: 행 수 / NOT NULL default 적용 시간 측정 (Profile 수가 적어 즉시 완료 예상)
3. 운영자 승인 후 적용

### Step 2 — 모델 / Repository / Settings dataclass

1. `app/models/scope_profile_settings.py` (또는 `scope_profile.py` 내부) — `ScopeProfileSettings(expose_viewers: bool = False)`
2. `scope_profiles_repository` 의 read/write 가 raw dict 와 dataclass 사이 변환
3. unit ≥ 6 건 — default 값, 알 수 없는 키 보존, 잘못된 타입 방어(JSON 안 boolean 외)

### Step 3 — 정책 게이트 함수

1. `services/scope_profile_policy.py` 신설 (또는 기존 ACL 모듈에 함수 추가):
   ```python
   def should_expose_viewers(viewer_actor: ActorContext) -> bool:
       """viewer 의 scope profile settings.expose_viewers 를 fail-closed 로 반환"""
   ```
2. 결과 캐시: process-local 캐시 30s (Profile 변경 후 즉시 반영은 약간 지연 허용)
3. unit ≥ 5 건 — true/false/조회 실패/파싱 실패/캐시 hit

### Step 4 — Contributors API 결합

1. `services/contributors_service.get_contributors()` 진입부 게이트 적용
2. 응답에서 viewers 키 제거 + `X-Viewers-Hidden-Reason` 헤더 추가
3. integration test ≥ 6 건:
   - profile A (off) → viewers 응답 없음 (헤더 사유)
   - profile B (on) + include_viewers=true → viewers 응답 포함
   - profile B (on) + include_viewers=false → viewers 응답 없음 (사용자 의사 우선 false)
   - profile A (off) + include_viewers=true 강제 → 강제 false (사유 헤더)
   - settings_json 파싱 실패 시 → false (fail-closed)
   - admin/non-admin 차이 없음 (정책은 actor-scope 기반)

### Step 5 — 다른 viewer 노출 경로 점검

1. grep `event_type='document.viewed'` / audit_events 직접 query / "viewer" "viewed_by" 키워드
2. 발견된 경로 각각에 게이트 함수 적용 또는 "본 FG 영향 없음" 사유 기록
3. 산출물 `FG3-2_viewer노출경로_점검표.md` 작성

### Step 6 — 관리자 UI

1. Scope Profile 편집 페이지에 토글 + 보조 설명 추가
2. PATCH 호출 후 즉시 React Query invalidate
3. 변경 후 audit emit (`scope_profile.settings.changed`, before/after 값 포함)
4. node:test ≥ 6 건 — 토글 렌더, 변경 호출, 권한 거부 표시

### Step 7 — 검수 / 보안 보고서

- `FG3-2_검수보고서.md` + 재검수 1회
- `FG3-2_보안취약점검사보고서.md` — **개인정보 영향 분석 필수**:
  - 기본값 off 가 모든 신규/기존 Profile 에 적용되는지 마이그레이션 확인 (스크린샷 첨부)
  - 강제 호출 시도(`include_viewers=true`) 가 정책에 의해 차단되는 회귀
  - audit_events 직접 read 우회 경로 부재 확인
  - 관리자 권한 검증 (require_admin) 누락 없음
  - settings.changed audit 가 actor/before/after 모두 보존
- **"뷰 ≠ 권한 준수 확인"**: 게이트 함수가 viewer 의 scope 만 보고, 대상 문서의 scope 와 무관하게 **viewer 정책으로 결정**된다는 점 명시 (조직 정책)

---

## 5. API 계약 요약

| 메서드 | 경로 | 설명 |
|-------|------|-----|
| PATCH | /api/v1/admin/scope-profiles/{id} | body 에 `settings.expose_viewers` 포함 가능 |
| GET | /api/v1/admin/scope-profiles/{id} | 응답에 `settings` dict 포함 |

응답 헤더:
- `X-Viewers-Hidden-Reason: scope-policy` — Contributors API 가 정책으로 viewers 를 가렸을 때

---

## 6. 성공 기준

- [ ] `scope_profiles.settings_json` 마이그레이션 적용 + 기본값 `{}` 검증
- [ ] `expose_viewers` default false (모든 기존 profile)
- [ ] Contributors API 결합 회귀 6 시나리오 녹색
- [ ] viewer 노출 경로 점검표 제출, 우회 경로 0
- [ ] 관리자 UI 토글 + audit emit 동작
- [ ] pytest 신규 ≥ 15 건, 베이스라인 유지
- [ ] node:test 신규 ≥ 6 건
- [ ] 보안 보고서의 개인정보 영향 분석 섹션 + 기본값 off 스크린샷 첨부

---

## 7. 리스크

| 리스크 | 대응 |
|-------|-----|
| 마이그레이션 후 기본값이 의도와 달리 on 으로 배포 | DEFAULT '{}' + dataclass default False + 마이그레이션 자체 검증 + 보안 보고서에 스크린샷 의무화 |
| 다른 viewer 노출 경로 누락 → 정책 우회 | Step 5 점검표 산출물 의무화. 점검 누락 시 FG 게이트 미통과 |
| 게이트 함수 캐시로 인한 토글 변경 30초 지연 | UX 경고 메시지 ("최대 30초 후 적용") + 관리자 강제 invalidate API (선택) |
| settings_json 의 미지 키 손실 | load → dataclass 추출, save → raw 머지로 보존. unit 테스트 명시 |
| 관리자가 자기 Scope 외 Profile 변경 | require_admin + organization_id 매칭 검증 (S2 admin 패턴 그대로) |

---

*작성: 2026-04-27 | FG 3-2 열람자 노출 제어*
