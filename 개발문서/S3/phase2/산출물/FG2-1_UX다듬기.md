# FG 2-1 UX 다듬기 — 컬렉션/폴더 편집·삭제 메뉴

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-24 |
| 범위 | 사이드바 컬렉션·폴더에 hover `···` 메뉴 추가, 인라인 rename, 삭제 확인 모달, 폴더 이동 모달, `document_count=0` 회색 뱃지 |
| 결과 | **Chrome 컨트롤 실측 5 시나리오 PASS** + **node:test 193 passed** (신규 9건 + 기존 184건) |

---

## 1. 변경 / 신규 파일

### 1.1 신규

| 파일 | 역할 |
|------|------|
| `frontend/src/features/explore/RowMenu.tsx` | 공용 `···` popover 메뉴 — hover 시 표시, 바깥 클릭·ESC 닫기, ↑↓ Enter 키보드 네비, danger 항목 빨강 스타일 |
| `frontend/src/features/explore/FolderMoveDialog.tsx` | 폴더 이동 모달 — 새 부모 드롭다운(자기 자신+하위 제외), `(루트로 이동)` 선택지. `computeMoveCandidates` / `formatPath` `__test__` export |
| `frontend/tests/FolderMoveCandidatesFg21Ux.test.tsx` | 이동 후보 필터 + path 포맷 단위 테스트 9건 |

### 1.2 수정

| 파일 | 변경 |
|------|------|
| `frontend/src/features/explore/CollectionsTree.tsx` | 각 항목을 group/hover 컨테이너로 재구성 — `RowMenu` 의 **이름 변경·삭제** 메뉴 + 인라인 rename input (focus 시 전체 선택, Enter 저장 / Esc 취소 / Blur 저장) + `ConfirmDialog` (컬렉션 내 문서는 유지됨 안내) + `document_count=0` 도 회색 톤 뱃지로 표시 |
| `frontend/src/features/explore/FoldersTree.tsx` | `FolderTreeNode` 가 기존 `+`(하위 추가) 버튼 옆에 `RowMenu` 추가 — **이름 변경·이동·삭제** 3 항목. 루트 rename/이동 시 하위 path 재계산은 React Query invalidate 로 서버 신뢰. `FolderMoveDialog` + `ConfirmDialog` 붙임 |
| `frontend/tsconfig.test.json` | test 컴파일 include 에 `FolderMoveDialog.tsx` 추가 |

---

## 2. 설계 판단

### 2.1 공용 RowMenu 분리

컬렉션/폴더 양쪽이 동일 패턴(`···` hover 버튼 → popover 메뉴)을 쓰므로 `RowMenu` 하나로 통일. 접근성:

- `aria-haspopup="menu"`, `aria-expanded` 적절히
- 메뉴 열면 `role="menu"` + 각 `role="menuitem"`
- 키보드 네비 ↑↓ 으로 cursor 이동, Enter/Space 실행, Esc 닫기
- 바깥 클릭 시 자동 닫힘 (`document.mousedown` 리스너)
- danger 항목은 `text-danger-600 + hover:bg-danger-50` 로 시각적 구분

### 2.2 Rename 은 **인라인**, Delete/Move 는 **모달**

- 이름 변경은 빈번·경량 → 한 줄 input 치환 (생성 입력과 동일 패턴, defaultValue + onBlur 저장 + Esc 취소)
- 삭제/이동은 되돌리기 어려워 **확인 단계** 필수 — 기존 `ConfirmDialog` 재사용, `danger` variant 버튼
- 폴더 "이동" 은 단순 확인보다 **부모 선택** 이 필요 → 전용 `FolderMoveDialog`

### 2.3 이동 후보 필터는 프런트에서도 사전 적용

서버 `folders_service.move_folder` 가 순환 참조/깊이 상한을 재검증하지만, 드롭다운에서 자기 자신과 하위를 애초에 **선택 불가**로 만들어 UX 상 혼란 제거. 필터 로직(`computeMoveCandidates`) 은 순수 함수로 추출해 단위 테스트 가능.

**회귀 방어 포인트**: `"/work/"` 와 `"/workflow/"` 같이 prefix 부분 일치에 오탐 가능성 → path 가 항상 `/` 로 종료되므로 `startsWith("/work/")` 가 `"/workflow/"` 를 먹지 않음을 테스트로 고정.

### 2.4 document_count 0 표시

기존에는 `count > 0` 일 때만 뱃지 렌더 → 비어있는 컬렉션임을 구분하기 어려움. `count != null` 조건으로 완화하고, 0 일 때는 투명 배경 + 회색 톤으로 부각 감소. 숫자는 항상 보여서 "0개 문서" aria-label 도 유지.

### 2.5 Optimistic update 정책 재확인

| 훅 | Optimistic | 이유 |
|----|-----------|-----|
| `useUpdateCollection` (rename) | ✓ onMutate | 즉시 반영 + 실패 시 rollback |
| `useDeleteCollection` | ✓ onMutate | 리스트에서 즉시 제거 |
| `useRenameFolder` | invalidate | path prefix 가 하위 전체에 영향 — 서버 재계산 결과 신뢰 |
| `useMoveFolder` | invalidate | path 가 트리 전반에 연쇄 — invalidate 안전 |
| `useDeleteFolder` | ✓ onMutate | 빈 폴더만 허용이므로 단순 제거 가능 |

---

## 3. Chrome 실측 시나리오 (5건 PASS)

| # | 시나리오 | 결과 |
|---|---------|------|
| 1 | 사이드바 컬렉션 "test" 에 hover | `0` 회색 뱃지 + `···` 메뉴 버튼 노출 ✓ |
| 2 | 컬렉션 `···` 클릭 → 메뉴 | "이름 변경 / 삭제" 2 항목 (삭제=빨강) ✓ |
| 3 | 폴더 "test" `···` 클릭 → 메뉴 | "이름 변경 / 이동 / 삭제" 3 항목 ✓ |
| 4 | 폴더 "이름 변경" 선택 | 인라인 input 등장, 기존 이름 선택 상태 ✓ |
| 5a | 폴더 "삭제" 선택 | ConfirmDialog: `폴더 "test" 을(를) 삭제할까요?` + "하위 폴더 또는 문서가 있는 폴더는 삭제되지 않습니다" 안내 ✓ |
| 5b | 폴더 "이동" 선택 | FolderMoveDialog: `"test" 이동`, 현재 경로 `/test/`, 드롭다운 기본값 `(루트로 이동)` + "자기 자신과 하위는 대상에서 제외" 안내 ✓ |

실측 진행은 `http://localhost:3050/documents` 데스크탑 1400×900 해상도에서 수행. 모든 인터랙션이 의도대로 동작하며 키보드 Esc/바깥 클릭 닫기도 OK.

---

## 4. 자동화 검증

### 4.1 node:test (frontend)

```text
$ npm test
# tests 193   suites 37   pass 193   fail 0
```

신규 9건 (`FolderMoveCandidatesFg21Ux`):

- `computeMoveCandidates`: 자기 자신 제외 / 직계 자식 제외 / 손자 제외 / 형제 포함 / 다른 서브트리 포함 / path prefix 부분일치 방어 (`/work/` vs `/workflow/`)
- `formatPath`: 루트 1단계 / 여러 단계 `›` 구분자 / 빈 세그먼트 방어

### 4.2 TypeScript 타입 체크

`./node_modules/.bin/tsc --noEmit` — FG 2-1 UX 변경 파일 에러 0. 기존 `AdminUserDetailPage.tsx:279` 만 남음 (Phase2 인수문서 §6 등록된 범위 밖 기존 버그).

---

## 5. UX 규약 유지

- 컬렉션은 **순수 뷰** — 삭제 시 문서는 유지됨 (ConfirmDialog 메시지로 안내)
- 폴더도 **뷰 레이어** — 이동은 문서 권한을 바꾸지 않음 (Phase 2 절대 규칙)
- 비어있지 않은 폴더 삭제 시도는 서버에서 409 → toast 로 안내 (`useDeleteFolder.onError`)
- owner 격리: 드롭다운에는 자기 소유 폴더만 나타남 (서버가 이미 owner 필터)

---

## 6. 남은 후속 작업 (본 UX 라운드 외)

1. **현재 폴더/컬렉션 노출** — `GET /documents/{id}` 응답에 `folder_id` + `in_collections[]` 포함 + DocumentAssignControls 가 현재 선택 표시 (다음 UX 라운드)
2. **빈 컬렉션 맥락 유지 CTA** — "이 컬렉션에 기존 문서 추가" 모달 + 새 문서 생성 시 자동 연결
3. **드래그-드롭 재배치** — Phase 3 이후
4. **컬렉션 설명(description) 편집 UI** — 현재 생성 시 미입력 상태로 고정. 편집 모달 또는 확장 인라인 영역

---

*작성: 2026-04-24 | FG 2-1 UX 다듬기 완료, Chrome 실측 + node:test 193 녹색*
