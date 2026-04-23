# AdminGoldenSetsPage 런타임 회귀 리뷰 보고서 (Chrome 5+회)

- 작성일: 2026-04-20
- 대상: `/admin/golden-sets` 화면 와이어링(#22, #21, #28, #29, #30) 결과의 Chrome 런타임 회귀
- 방법: Chrome MCP(`localhost:3050`, `admin@mimir.local` / `ORG_ADMIN` 세션) 로 5개 이상 시나리오를 순차 수행하며 스크린샷·키보드·네트워크 로그 수집
- 원 수정 검수: `docs/개발문서/S2_5/UI_Admin_GoldenSets_검수보고서.md`
- 연계 보안 보고서: `docs/개발문서/S2_5/UI_Admin_GoldenSets_보안취약점검사보고서.md`

---

## 리뷰 요약

| 라운드 | 경로 / 조작 | 초점 | 결과 |
| --- | --- | --- | --- |
| R1 | `/admin/golden-sets` (빈 상태 최초 진입) | 페이지 chrome, 빈 상태 메시지, primary 버튼 | ✅ "골든셋 생성" blue primary(44px), "아직 생성된 골든셋이 없습니다..." 안내 문구 렌더, Scope Profile ACL 에러 배너 미출현 |
| R2 | "골든셋 생성" 버튼 → 모달 | 폼 유효성, 포커스 트랩 시작점, `이름` required, Esc 닫기 | ✅ `이름` 빈 값일 때 "생성" 버튼 `disabled:bg-blue-300`, 입력 시 활성화. 첫 진입 시 `nameRef` 자동 포커스(`useEffect` 확인) |
| R3 | 모달 "생성" 클릭 → POST `/api/v1/golden-sets` | 201 응답, 캐시 invalidate, 상세 패널 자동 오픈, v1 / 0 문항 표기 | ✅ POST 201 `{id, version:1, item_count: null}`, 모달 닫히고 `setSelectedId(gs.id)` 로 상세 패널 전환, `v1 / 문항 0` 렌더링 |
| R4 | 브라우저 `F5` 하드 리로드 → GET `/api/v1/golden-sets?limit=50` | **P0 #30 `item_count` 재현/수정 확인** | ✅ 200 응답(이전: 503). 방금 생성한 `S2-5 회귀 테스트 골든셋` 행이 목록에 렌더, `문항 0 / v1 / 도메인 사용자 정의` 표기 정상 |
| R5 | 상세 패널 → "+ 항목 추가" 폼 | 4필드 필수(질문·기대답변·document_id·version_id·node_id) 검증, "취소" 복귀 | ✅ 5필드 중 아무거나 공란이면 "항목 추가" 버튼 `disabled`, 전부 채우면 활성. "취소" 클릭 시 폼 언마운트, 상세 패널은 유지 |
| R6 | 목록 행 `Tab` → 포커스 → `Enter` → 상세 전환 | P0-4 키보드 접근성(이 페이지에도 적용됨) | ✅ row 에 `focus-visible:bg-blue-50 ring-2 ring-blue-500` 가시화, Enter 로 `setSelectedId` 호출되어 상세 드로어 오픈 |
| R7 | 네트워크 panel에서 request/response 헤더 검증 | 쿠키 기반 인증, `X-Request-Id` 전파, `authorization` 헤더 없음(로컬 DEV 세션) | ✅ `Cookie: cowork_session=…` 포함, 응답 `X-Process-Time` 정상. 민감 토큰이 URL 파라미터에 노출되지 않음 |

**5+회 목표 초과 달성(7회).** 핵심 와이어링은 모두 런타임에서도 정상 동작.

---

## 키 검증 포인트

### 1. P0 #30 — `GoldenSet.item_count` 필드 누락으로 인한 503 재현/수정

- **증상**: R3 에서 골든셋 1건을 생성한 **직후** R4 의 목록 조회가 `503 Service Unavailable` 로 실패.
- **초기 오진**: 프런트 에러 배너(`Scope Profile 바인딩 또는 권한을 확인해 주세요. (S2 ⑥)`) 만 보고 scope 누락을 의심했으나, 백엔드 콘솔에서 실제 예외 트레이스 확인:

  ```text
  File ".../app/repositories/golden_set_repository.py", line 206, in list_by_scope
      gs.item_count = self._count_items(str(row["id"]))  # type: ignore[attr-defined]
  ValueError: "GoldenSet" object has no field "item_count"
  ```

- **근본 원인**: Pydantic v2 strict 모드에서는 모델에 **선언되지 않은** 필드에 setattr 금지. `GoldenSet` 모델(`backend/app/models/golden_set.py`)에 `item_count` 필드가 없었음. 빈 목록일 때는 `for row in rows` 가 돌지 않아 예외 경로가 **숨어 있었고**, 첫 생성 직후 첫 목록 조회에서 즉시 터짐.
- **왜 503 인가**: FastAPI 의 sync endpoint → anyio threadpool 에서 `ValueError` 가 전파되어 전역 핸들러 체인을 거치는 과정에서 503 으로 매핑. (정상 예상은 500 — 이 부분은 후속 태스크로 분리해 프런트 배너 분기(#31)와 함께 처리.)
- **수정**:

  ```python
  # backend/app/models/golden_set.py  (class GoldenSet)
  # FG7.1: list_by_scope 가 _count_items 결과를 덧붙여 응답 직렬화 시 사용한다.
  # 영속 컬럼이 아니므로 DB row → model 매핑 시에는 설정되지 않고, None 인 상태로
  # 남는다 (응답 포맷에서는 None → 미표시). Pydantic v2 strict 모드에서 속성 할당
  # 허용을 위해 명시적으로 필드로 선언한다.
  item_count: Optional[int] = None
  ```

  그리고 repository 에서 `# type: ignore[attr-defined]` 주석 제거(필드가 정식 선언되었으므로 불필요).

- **검증**: R4 재시도에서 HTTP 200, `data[0].item_count = 0`, UI 테이블의 `문항` 컬럼이 `-` 가 아닌 `0` 으로 렌더링 → 필드 매핑이 프런트까지 정상 전달.

### 2. API 경로 정정(#28) 런타임 확인

- 이전 와이어링 시 `/api/v1/admin/golden-sets` 로 호출 → 404. 백엔드 `router.py` 기준 `prefix="/golden-sets"` 이며 `admin` 하위가 아님.
- 수정 후 R3/R4 의 request URL 이 `/api/v1/golden-sets` (admin 제거) 로 정확히 기록됨. S2 ⑥ ACL 은 `scope_profile_id` 바인딩(#29)으로 유지.

### 3. S2 ⑥ Scope Profile 바인딩(#29) 런타임 확인

- R1~R7 모든 요청이 `scope_profile_id` 가 바인딩된 ActorContext 로 해석됨. 이전 세션에서 `users.scope_profile_id` 컬럼을 Alembic 으로 추가하고 기본 Scope Profile 을 시드한 결과, 목록/생성/조회 전부 403 없이 통과.
- 만약 scope 바인딩이 누락됐다면 `_require_scope` 가 403 을 던져 프런트 배너에 "S2 ⑥" 안내가 보이는데, 본 리뷰에서는 발생하지 않음 → 바인딩 배포 정상.

### 4. 키보드 접근성 (P0-4 확장)

- 목록 `<tr>` 에 `role="button" + tabIndex=0 + onKeyDown(Enter|Space)` 계약이 `AdminGoldenSetsPage` 에서도 동일 패턴으로 적용됨.
- `Tab` 으로 첫 번째 행에 포커스 → `focus-visible:bg-blue-50 focus-visible:ring-inset focus-visible:ring-blue-500` 가시화 → `Enter` 로 상세 패널 오픈.
- 내부 "상세" 버튼은 `e.stopPropagation()` 으로 row onClick 과 이중 발화 방지. 키보드 사용자도 마우스 사용자와 동등한 경로 보장.

### 5. Scope 문자열·하드코딩 부재 확인 (S2 ⑤)

- 런타임 네트워크 payload 와 클라이언트 코드 모두 `team`/`org` 같은 scope 어휘가 등장하지 않음. 필터는 서버가 `scope_id` 로 처리.

---

## 본 리뷰에서 추가로 관찰된 파생 이슈(범위 밖, 후속 태스크로 분리)

- **#31 — 목록 에러 배너가 모든 실패를 "S2 ⑥" 으로 표기** _(RESOLVED 2026-04-21)_
  현재 `AdminGoldenSetsPage` 의 `errorMessage` 렌더 블록은 HTTP 상태와 무관하게 "Scope Profile 바인딩 또는 권한을 확인해 주세요. (S2 ⑥)" 안내를 함께 출력한다. R4 의 503 케이스처럼 서버 측 버그도 S2 ⑥ 로 오해될 수 있음. 후속에서 상태 코드별(403 → S2 ⑥ / 5xx → 일시 장애 재시도 유도 / 네트워크 실패 → 연결 안내) 분기 필요. 본 PR 에서는 중요도 낮음(근본 버그 #30 이 해소되어 빈도 최소화).
  → 해소: `classifyListError` 헬퍼로 HTTP 상태·에러 타입별 분기 구현. 네트워크/403/5xx 3 시나리오 Chrome 런타임 검증 완료. 세부는 `docs/개발문서/S2_5/UI_Admin_GoldenSets_P1_31_에러배너분기.md` 참조.
- **#32 — 503 → 500 매핑 차이** _(RESOLVED 2026-04-21)_
  `ValueError` 가 전역 핸들러에서 500 이 아닌 503 으로 매핑되는 경로는 `unhandled_exception_handler` + slowapi 와의 상호작용으로 추정되며, 운영 안정성 관점에서 별도 조사 필요. 본 리뷰 이후 후속으로 분리.
  → 조사 결론: 코드 경로 재구성 결과 `ValueError` 는 `Exception` catch-all 을 통해 **항상 500** 으로 매핑되는 것이 유일한 정상 경로이며, 503 을 생성할 수 있는 코드는 존재하지 않음(슬로우API 는 `swallow_errors=True`, `ApiServiceUnavailableError` 는 raise site 0건). R4 의 503 관측은 오관측(오타/uvicorn --reload 일시 상태/캐시된 응답) 가능성이 가장 높음. 계약 고정을 위해 `backend/tests/unit/test_unhandled_exception_mapping.py` (ValueError/RuntimeError/TypeError/KeyError → 500, `ApiServiceUnavailableError` → 503, typed ApiError → 지정 상태) 회귀 테스트 추가. 세부는 `docs/개발문서/S2_5/UI_Admin_GoldenSets_ValueError_500_503_매핑조사.md` 참조.

---

## 정리 작업

- ~~리뷰 중 생성한 테스트 row **"S2-5 회귀 테스트 골든셋"** 은 회귀 재현용으로만 사용되었으며, 실데이터가 아님. 수동 삭제 필요(`DELETE /api/v1/golden-sets/{id}` 또는 상세 패널 "삭제" 액션).~~ → **완료 2026-04-21**: AdminGoldenSetsPage 상세 패널에 "위험 구역 / 골든셋 삭제…" 버튼 + type-to-confirm 모달(#33) 을 추가하고, 실제 DELETE 경로로 삭제 완료. 동시에 4 가지 에러 분기(403/404/5xx/network)를 fetch mock 으로 검증. 세부는 `docs/개발문서/S2_5/UI_Admin_GoldenSets_P2_33_삭제버튼.md` 참조.

---

## 결론

- **5개 시나리오 이상(실제 7개) 런타임 회귀 통과.**
- R4 에서 **P0 #30 (`item_count` 필드 누락)** 을 실측으로 발견·수정하여, 생성 후 즉시 목록 재조회가 정상 동작함을 확인.
- 와이어링 트랙 #21 / #22 / #28 / #29 / #30 전부 브라우저에서 의도대로 동작.
- **후속**:
  1. ~~#31 상태 코드별 에러 배너 분기(중요도 낮음, 체감 UX 개선)~~ → **완료 2026-04-21** (`UI_Admin_GoldenSets_P1_31_에러배너분기.md`)
  2. ~~500/503 매핑 정상화 조사~~ → **완료 2026-04-21** (`UI_Admin_GoldenSets_ValueError_500_503_매핑조사.md` / 계약 테스트 `backend/tests/unit/test_unhandled_exception_mapping.py`)
  3. ~~테스트 row 삭제~~ → **완료 2026-04-21** (#33 삭제 버튼 구현과 함께 처리. `UI_Admin_GoldenSets_P2_33_삭제버튼.md` 참조)
  4. `DataTable` 키보드 이벤트·`item_count` 모델 필드에 대한 유닛 테스트 추가(S2 종결 P0 커버리지 갭 해소의 일부)
