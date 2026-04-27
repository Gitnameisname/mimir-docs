# FG 3-2 UI 검수 로그

**작성일**: 2026-04-27
**대상**: AdminScopeProfilesPage 의 운영 설정 섹션 (`expose_viewers` 토글)
**검수 기준**: CLAUDE.md 프로젝트 §5.2 — UI 리뷰 ≥ 1회 (Phase 1 완화)

---

## 1. 검수 환경

- 검수자: Claude (자기 검수). 사람 최종 리뷰는 Codex / @최철균
- 빌드 / 타입체크: `npm test` (tsc + node:test 540/540 PASS)
- 화면 조작 시뮬레이션: 코드 정독 + 컴포넌트 상호작용 추적

---

## 2. 검수 항목

### 2.1 데스크톱 (≥ 1024px)

| 항목 | 결과 |
|-----|-----|
| 사이드 슬라이드 패널 (`fixed inset-0 ... max-w-xl`) 안에 운영 설정 섹션 등장 | ✅ Scope 목록 위에 배치 |
| 카드 형태 (border + bg-gray-50) | ✅ |
| 토글 (`<input type="checkbox">`) + 라벨 + 설명 | ✅ |
| 로딩 중 disabled | ✅ `detailQ.isLoading || updateSettingsMut.isPending` |
| 저장 중 inline 피드백 | ✅ disabled 동안 React Query mutation 진행 |
| 성공 시 "저장되었습니다." 표시 | ✅ green-700 텍스트 |
| 실패 시 "권한을 확인하세요." | ✅ red-600 텍스트 |
| 30초 캐시 TTL 안내 | ✅ "최대 30초 안에 반영됩니다" 문구 |

### 2.2 좁은 데스크톱 / 태블릿

| 항목 | 결과 |
|-----|-----|
| max-w-xl 패널이 viewport 내 들어감 (~768px ≥) | ✅ |
| 토글 + 설명 텍스트 줄바꿈 | ✅ `flex-1` + 자연 wrap |

### 2.3 모바일

| 항목 | 결과 |
|-----|-----|
| 패널이 화면 가득 (full-width slide) | ✅ `max-w-xl` 가 viewport ≤ 576px 에선 100% |
| 체크박스 hit area | ⚠️ 16px (`w-4 h-4`) — 권장 24~44px 미달. Phase 2 admin 패턴 일관 (별 라운드) |
| 라벨 클릭 (`<label>` 안 input) | ✅ 라벨 전체가 클릭 영역 |

### 2.4 접근성 (a11y)

| 항목 | 결과 |
|-----|-----|
| `<label>` 가 `<input>` 을 wrap → 라벨 클릭으로 토글 | ✅ |
| `aria-label="이 Scope 사용자에게 다른 사용자의 열람 흔적을 표시"` | ✅ |
| 패널 자체 `role="dialog" aria-modal="true"` (기존) | ✅ |
| 에러 메시지 `role="alert"` 누락 | ⚠️ 기존 admin 페이지 패턴 일관. 별 라운드 |

---

## 3. 발견 / 결정

| # | 항목 | 결정 |
|---|-----|-----|
| 1 | 토글 vs 명시적 "On / Off" 라디오 | 토글 채택 (단일 boolean) |
| 2 | 저장 직후 confirm modal | 미적용 (단일 클릭 토글이라 이중 확인 과도) |
| 3 | 30초 캐시 안내 | UI 에 명시 (운영자 혼란 방지) |
| 4 | 변경 audit 표기 | UI 에 직접 표시 안 함 (audit_events 는 admin 의 다른 페이지에서 조회) |
| 5 | 다크모드 토큰 | 미적용 (Phase 2 admin 다크모드 미도입과 일관) |

---

## 4. 회귀 영향

- AdminScopeProfilesPage 에 운영 설정 섹션 1 개 신설 (Scope 목록 위)
- 기존 Scope CRUD 동작 영향 없음 (별도 mutation)
- `npm test` 540/540 PASS — 기존 회귀 0
- tsc 0 touched 오류

---

## 5. 사람 운영자 후속 권장

1. 실 브라우저에서 토글 → 30초 후 ContributorsPanel viewers 응답 변화 확인
2. 권한 없는 사용자 (admin 아님) 에서 토글 시도 시 403 처리
3. 다중 워커 환경에서 다른 워커의 캐시 invalidation 시점 확인

---

*작성: 2026-04-27 | task3-2 Step 6 산출물*
