# #33 — 골든셋 삭제 버튼 + 테스트 row 정리 구현 보고서

- 작성일: 2026-04-21
- 범위: `AdminGoldenSetsPage` 상세 패널에 소프트 삭제(UI 노출 기준 영구 삭제) 액션을 추가하고, 런타임 회귀 리뷰(#24) 중 남아 있던 테스트 row "S2-5 회귀 테스트 골든셋" 정리
- 관련 이슈: 런타임 회귀 리뷰(`UI_Admin_GoldenSets_런타임회귀리뷰.md`) §정리 작업
- 관련 PR 스레드: 본 저장소 S2-5 UI 개선 브랜치

---

## 1. 배경

S2-5 런타임 회귀 리뷰(#24) 과정에서 `/admin/golden-sets` 에 P0 #30(`item_count` 필드 누락) 재현을 위해 테스트 row 한 건을 남겼다. 이 row 는 실데이터가 아니므로 제거해야 하는데, 기존 UI 에는 골든셋 단위 삭제 액션이 없었다(항목 단위 `deleteItem` 만 존재). 운영 관점에서도 destructive action 은 UI 에서 제공해야 하므로, 다음 두 가지를 함께 처리했다:

1. `AdminGoldenSetsPage` 상세 패널에 "위험 구역" 섹션 + type-to-confirm 모달을 추가
2. 새 버튼으로 테스트 row 정리

## 2. 백엔드 현황(기 구현, 신규 변경 없음)

- 라우트: `backend/app/api/v1/golden_sets.py` L248 `DELETE /{golden_set_id}` (status 204)
  - `_require_write(actor)` + `_require_scope(actor)` 로 ACL 이중 게이트
  - 404 분기: `repo.soft_delete` 가 0 rows 를 반환하면 HTTPException(404)
  - 감사 로그: `audit_emitter.emit(event_type="golden_set.deleted", action="golden_set.delete", actor_type=…)` 로 S2 ⑥ `actor_type` 필수 필드 포함
- 저장소: `backend/app/repositories/golden_set_repository.py` L270 `soft_delete(golden_set_id, scope_id)`
  - `UPDATE golden_sets SET is_deleted=TRUE, deleted_at=%s WHERE id=%s AND scope_id=%s AND is_deleted=FALSE`
  - 성공 시 하위 `golden_items` 도 동일 타임스탬프로 soft delete 캐스케이드
  - **scope_id 로 IDOR 차단** — 다른 scope 의 id 가 넘어와도 UPDATE 건수 0 → 404 반환

프런트엔드 API 도 이미 존재: `frontend/src/lib/api/s2admin.ts` L350 `goldenSetsApi.delete(id)` (`api.delete<void>(…)`).

따라서 이번 작업은 **프런트엔드 단일 파일 수정**으로 끝남.

## 3. 프런트엔드 수정

파일: `frontend/src/features/admin/golden-sets/AdminGoldenSetsPage.tsx`

### 3.1 에러 분류 헬퍼 (`classifyDeleteError`)

`classifyListError`(#31) 와 대칭 구조. 상태 코드별로 제목/본문/힌트/재시도 여부를 결정한다.

- 401 → "세션이 만료되었습니다" / 재로그인 안내, canRetry=false
- 403 → "삭제 권한이 없습니다" + **S2 ⑥ 힌트** (이 분기에만), canRetry=false
- 404 → "이미 삭제되었거나 존재하지 않습니다" (삭제 행동에 맞춘 특화 문구)
- 429 → 요청 빈도 안내, canRetry=true
- 5xx → "서버에서 일시적 오류가 발생했습니다" + 재시도 힌트
- `TypeError` → "서버에 연결하지 못했습니다" (네트워크 단절)
- fallback → 일반 메시지 + canRetry=true

### 3.2 `GoldenSetDeleteConfirmModal` 컴포넌트 (type-to-confirm)

GitHub/GitLab 삭제 모달 관례. 대상 골든셋 이름을 정확히 재입력해야만 "영구 삭제" 버튼이 활성화된다.

| 접근성 요소 | 구현 |
| --- | --- |
| Role/modal | `role="dialog"` + `aria-modal="true"` + `aria-labelledby="golden-set-delete-title"` + `aria-describedby="golden-set-delete-desc"` |
| 초기 포커스 | `useEffect(() => inputRef.current?.focus(), [])` 로 입력란 자동 포커스 |
| ESC 닫기 | pending 중에는 닫기 금지(이중 실행 방지) |
| 최소 터치 타겟 | 취소/영구 삭제 버튼 `min-h-[44px]` |
| Focus ring | 입력·버튼 모두 `focus-visible:ring-2 focus-visible:ring-red-500` |

### 3.3 상세 패널 하단 "위험 구역" 섹션

- GitHub 관례대로 상세 패널 최하단에 배치하여 닫기(X) 버튼과 물리적으로 분리
- 빨강 계통 카드(`border-red-200 bg-red-50/40`) + 경고 문구 + 아웃라인 버튼 "골든셋 삭제…"
- 버튼 `aria-label` 에 골든셋 이름 포함해 스크린 리더 대응
- 클릭 시 `setShowDeleteConfirm(true)` + 이전 에러 상태 초기화

### 3.4 `useMutation` 배선

```ts
const deleteMutation = useMutation({
  mutationFn: () => goldenSetsApi.delete(goldenSetId),
  onSuccess: () => {
    setDeleteErrorInfo(null);
    setShowDeleteConfirm(false);
    queryClient.invalidateQueries({ queryKey: ["admin", "golden-sets"] });
    onClose();
  },
  onError: (err) => setDeleteErrorInfo(classifyDeleteError(err)),
});
```

- 성공: 에러 초기화 → 모달 닫기 → 목록 invalidate → 패널 닫기 (한 번의 렌더 사이클)
- 실패: 모달 유지, 인라인 에러 배너 표시, 사용자가 재시도 또는 취소 선택

## 4. UI 디자인 리뷰(6회, CLAUDE.md §4 규칙 충족)

| # | 관점 | 검증 결과 |
| --- | --- | --- |
| 1 | 위험 구역 배치/시각 | ✅ 상세 패널 최하단, 빨강 테두리/배경, 설명 + 아웃라인 버튼 — 닫기/생성 버튼과 물리적으로 격리 |
| 2 | 확인 모달 레이아웃 | ✅ 빨강 헤더 + 경고 아이콘, 대상 이름 하이라이트 박스, 입력란, 취소/영구 삭제 2 버튼 그리드 |
| 3 | 초기 포커스·포커스 링 | ✅ 진입 시 입력란 포커스(빨강 링), tab 이동 시 취소→영구 삭제 순으로 자연 이동 |
| 4 | type-to-confirm 가드 | ✅ "wrong name" 입력 시 pink-300 disabled, 정확히 일치하면 bg-red-600 활성화 |
| 5 | 해피 패스 | ✅ 올바른 이름 입력 → 영구 삭제 → DELETE 전송 → 패널·모달 닫힘 → 목록 빈 상태로 재렌더 |
| 6 | 에러 분기 4 종 | ✅ fetch mock 으로 403/404/503/network 각각 확인 (§5 참조) |

## 5. Chrome 런타임 검증

### 5.1 해피 패스

- 대상: "S2-5 회귀 테스트 골든셋" (#24 리뷰 중 남겨둔 row)
- 결과: 목록 행 → 상세 패널 → 위험 구역 → 모달 → type-to-confirm → `DELETE /api/v1/golden-sets/{id}` 204 → 패널 닫힘 → 목록 빈 상태 렌더(= "아직 생성된 골든셋이 없습니다")

### 5.2 에러 분기 (신규 테스트 row "삭제 에러 분기 검증용" 생성 후 `window.fetch` mock)

fetch mock 은 **DELETE 메서드 + `/api/v1/golden-sets/` URL 매칭** 에만 개입하도록 한정(목록 GET 은 우회). 각 모드에서 type-to-confirm 입력 → 영구 삭제 클릭 → 배너 확인 → 다음 모드로 전환.

| 모드 | 응답 | 배너 제목 | 배너 본문 | 힌트 | 모달 유지 |
| --- | --- | --- | --- | --- | --- |
| `403` | 403 permission_denied | "삭제 권한이 없습니다" | backend 메시지 | "Scope Profile 바인딩 또는 역할을 확인해 주세요. (S2 ⑥)" | ✅ |
| `404` | 404 not_found | "이미 삭제되었거나 존재하지 않습니다" | "목록을 새로고침하면 반영되어 있을 수 있습니다." | — | ✅ |
| `503` | 503 service_unavailable | "서버에서 일시적 오류가 발생했습니다" | backend 메시지 | "잠시 후 다시 시도해 주세요." | ✅ |
| `network` | `TypeError("Failed to fetch")` | "서버에 연결하지 못했습니다" | "네트워크 상태 또는 서버 가용성을 확인해 주세요." | — | ✅ |

**S2 ⑥ 경계**: `hint` 에 S2 ⑥ 문자열이 등장하는 경로는 403 하나뿐(#31 정책과 동일). 5xx 는 "잠시 후 다시 시도" 로, 네트워크는 "연결 확인" 으로 분기되어 사용자가 장애 종류를 혼동하지 않는다.

### 5.3 정리

- fetch mock 은 검증 후 원본 `window.fetch` 로 복원, `__mimirOriginalFetch` · `__mimirDeleteMockMode` 제거 → 런타임 상태 오염 없음
- 네트워크 실패 시나리오에서 모달이 닫히지 않고 에러 배너만 교체되는 것도 육안 확인 완료

## 6. 접근성 / 보안 영향

### 접근성(P0-4 정책 연장)

- 모달 `role="dialog"` + `aria-modal="true"` + `aria-labelledby/aria-describedby`
- 버튼 최소 터치 영역 44px, focus-visible ring
- 키보드 경로: Tab 으로 취소→영구 삭제, Enter 로 `type` 액션(pending 중이 아니면), Esc 로 닫기

### 보안

- **ACL**: 백엔드 `_require_write` + `_require_scope` 로 이중 게이트, 저장소 `WHERE scope_id=%s` 로 IDOR 차단
- **감사 로그**: `golden_set.deleted` 이벤트에 `actor_type` 필수 필드 기록(S2 ⑥)
- **XSS 회피**: 모달은 `detail.name` 을 React text child 로만 렌더(`<p>{goldenSetName}</p>`) — 템플릿 리터럴·dangerouslySetInnerHTML 등 주입 경로 없음
- **이중 실행 방지**: `deleteMutation.isPending` 동안 모든 버튼 disabled + Esc 닫기 금지
- **삭제 범위 최소화**: soft delete(복구 가능) — 하드 delete 는 관리자 DB 작업으로만 수행 가능
- **본 작업으로 새로 도입된 3rd-party 라이브러리 없음** → 공급망 위험 증가 없음

## 7. 정적 분석

- `npx tsc --noEmit` — AdminGoldenSetsPage 관련 오류 0 건 (기존 `AdminUserDetailPage.tsx:279` 는 별도 이슈)
- `npx eslint src/features/admin/golden-sets/AdminGoldenSetsPage.tsx` — EXIT 0 (clean)

## 8. 후속 / 알려진 제약

- (Nice to have) 삭제 후 "되돌리기" 5초 배너 — soft delete 이므로 기술적으로 가능하나 현재 범위 밖. 사용자 피드백 누적 시 고려
- (커버리지 갭 파생) 백엔드 `DELETE /golden-sets/{id}` 에 대한 pytest 회귀 테스트 — 기존 golden_set CRUD 전반에 테스트가 얇으므로 S2 종결 P0 커버리지 개선 태스크에 포함 권장
- AdminUserDetailPage 의 기존 `tsc` 에러(`Date(undefined)` 오버로드 불일치)는 본 작업 범위와 무관, 별도 이슈로 분리

## 9. 결론

- 프런트엔드 단일 파일 수정으로 destructive action UI 를 완비 (type-to-confirm · 접근성 · 에러 분기 전부 충족)
- 백엔드 ACL/IDOR/감사 로그는 기 구현 상태 재확인 — 추가 변경 불필요
- 테스트 row 2 건 모두 새 버튼을 통해 정상 삭제 → 런타임 리뷰(#24) 의 "정리 작업" 최종 종결
- CLAUDE.md UI 규칙(리뷰 ≥5 회, 데스크탑/웹 호환) 및 S1/S2 원칙(하드코딩 금지·S2 ⑥ ACL·감사 로그 `actor_type`) 준수
