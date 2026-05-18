# S3 Phase 5 1라운드 독립 검수보고서 (별 reviewer 관점)

| 항목 | 값 |
|------|----|
| 작성일 | 2026-05-14 |
| 작성자 | Claude (Review Actor — 헌법 제27조 No Self-Review 충족용) |
| 대상 | S3 Phase 5 (TipTap Mark 통합 + UX 본격화) — FG 5-1 ~ FG 5-4 1라운드 |
| 입력 | task5-1~5-4 + Mark_통합_ADR + Annotation_Decoration_방식 + FG5-1~5-4 검수보고서 + FG5-3 보안보고서 + Phase5_1라운드_종결회고 |
| 운영 모드 | dual-agent (Claude=Design/Review, Codex=Implementation). 본 검수는 별 reviewer 1차 — `@최철균` P1 게이트의 일부 |
| Handoff Level | extended (P1 — Mark 통합 ADR + API 표면 1 endpoint 추가 + DocumentDetailPage 재구조화) |
| 결론 | **샌드박스 1차 PASS, 운영 게이트 CONDITIONAL** — 코드 정합·단위 회귀·R-A4 4중 방어는 견고. 단 §3 의 5건이 공식 종결 전 닫혀야 한다. |
| 2026-05-14 패치 후 | **§3 5건 중 4건 코드 패치로 닫힘** (§11 참조). 별 reviewer 합의만 잔여. |

---

## 0. 검수 범위 / 방법

- **범위**: 1라운드 4 FG 전체. 회고 §2 매트릭스 9 항목 + §3 회귀 게이트 4 항목 + R-A1~R-A4 코드 차원 정합 + 헌법 제5·11·12·15·17·24·27·45·46조 정합.
- **방법**: 코드 직접 읽기 — `markNames.ts` / 4 mark 파일 / `AnnotationMark.ts` / `AnnotationGutter.tsx` / `useDocumentAnnotations.ts` / `DocumentTipTapEditor.tsx` / `DocumentDetailPage.tsx` / `AnnotationsPanel.tsx` / `DocumentSidebar.tsx` / `sidebarTabs.ts` / `useDocumentSidebar.ts` / `users_search.py` / `user_search.py` / `users_repository.search_by_display_name_in_orgs` / `MentionPopup.tsx` / `useUserSearch.ts` / `users_search.ts` / `globals.css` / `frontend.md` §8/§9/§10 / `backend.md` §8 + 회귀 테스트 4편 + 라우터 등록.
- **미검증 영역 (운영자 환경 의존)**: 실 pytest/node:test 녹색 숫자 (검수보고서 청구 587 / 9 인용만), 실 Chrome 실측, jsdom 통합 회귀, 실 DB JOIN 동작.

본 보고서는 **헌법 제15조 (의도 vs 구현 일치 검증) + 제27조 (셀프 리뷰 금지)** 정합용. 본 라운드 구현자가 Claude(1차 산출물) → 본 reviewer 도 Claude(별 세션) 라 엄격한 의미의 "별 주체 리뷰" 는 아니다. 정식 종결 전 Codex 또는 사람 reviewer 의 동의가 필요하다 (AGENT_MODE §2.1, 회고 §6 #1).

---

## 1. 종합 결과 매트릭스

| FG | 코드 정합 | 단위 회귀 | 사용자 시점 동작 | 비고 |
|----|---------|----------|---------------|------|
| 5-1 Mark 통합 ADR + markNames 정본 | ✅ | ✅ 15건 | ✅ (정본 적용 정상) | round-trip 회귀가 schema 정합 위주 — 실 round-trip 은 task5-2 sanitize 와 jsdom 통합으로 보강 |
| 5-2 AnnotationMark + Gutter + Panel 양방향 | 🟡 부분 | ✅ 13건 | ❌ **본문 시각화 어디서도 안 됨** | DetailPage NodeRenderer mark 미인식 / EditPage AnnotationsPanel/state 통합 0 / Gutter stub / sanitize 호출 0 |
| 5-3 멘션 typeahead + R-A4 ACL | ✅ (4중 방어 견고) | ✅ 9건 (mock) | ❌ **MentionPopup 어디에도 마운트 안 됨** | API+UI 컴포넌트 분리 작성 완료. AnnotationsPanel textarea 통합 미수행. 401/429/실 DB JOIN 통합 회귀 부재 |
| 5-4 DocumentSidebar 5 탭 | ✅ | ✅ 8건 | ✅ (desktop 고정 정상) | 4 viewport drawer / focus trap / UI 리뷰 ≥ 2회 미수행 |

**총평**: FG별 산출물의 **코드 정합·단위 회귀는 깨끗**. 단 §2~§5 의 **사용자 시점 통합 누락**이 게이트 §5.1 의 "실 동작" 요건을 일부 충족 못한다. 검수보고서들이 이를 모두 잔여로 명시했으나, **회고 §2 매트릭스가 #5 (AnnotationMark click → 패널 자동 펼침) 를 ✅ 로 표기한 것은 회로의 일부 (panel→panel) 만 PASS 임을 본문→panel PASS 와 혼동** — Disagreement Record 1건 (§4).

---

## 2. R-A1 ~ R-A4 절대 규칙 정합

### 2.1 R-A1 (Mark 직렬화 정본 단일)

- ADR §c 결정 (i) TipTap default 순서 채택 + round-trip 비교는 `{type, attrs}` set 비교. **결정 합리**. 마이그레이션 비용 회피 + 새 mark 등록 시 §10 갱신으로 결정성 확보.
- 4 mark 가 `excludes: ""` + `inclusive: false` 통일. `markNames.ts` 단일 정본 — string literal 인라인 0건 (회귀 A 4건 + B 6건이 정본 일치 검증).
- **부족**: ADR §11 의 S5 (4 mark 동시 round-trip — backend pytest `test_save_draft_preserves_marks_roundtrip.py`) **미작성**. FG 5-2 sanitize 회귀가 S5 의 frontend 측면만 일부 대체 — backend round-trip 동등성은 코드 차원 미증명.
- 판정: **CODE PASS, 통합 회귀 미수행**

### 2.2 R-A2 (Mark 우선순위 명시)

- ADR §b 의 C1~C8 표 정합. `excludes: ""` 정책으로 다른 mark 와 결합 보장 — code 차원 확인.
- 회귀: HashtagMark / WikiLinkMark / AnnotationMark 의 `excludes=""` config 정합 회귀 (FG5-1 + FG5-2 통합 6건).
- **잠재**: C8 (WikiLinkMark + HashtagMark 동시) 회귀 부재 — ADR §11 S11 가 잠정. wikilink target 안 `#tag` 의 InputRule 동작은 검증 안 됨.
- 판정: **CODE PASS, S11 미수행**

### 2.3 R-A3 (편집 차단 없음)

- `AnnotationMark.addProseMirrorPlugins` 의 `handleClick → return false` — TipTap default cursor / selection 동작 보존. `preventDefault()` 호출 0건 — 정적 분석으로 정합 확인.
- **사용자 시점 검증 부재**: DocumentEditPage 가 AnnotationMark 의존 통합을 하지 않음. **실 환경에서 mark click 동작 검증 path 부재**. 검수보고서 FG5-2 §5 잔여 #4 + #7 정직 명시.
- **잠재 Stale closure 위험**: `AnnotationMark.configure({ onAnnotationClick })` 의 콜백이 `this.options` 에 capture 되어 mark 인스턴스 생명주기 동안 유지. `useEditor` 가 dependency 배열 없이 호출되므로, `onAnnotationClick` prop 이 시점에 따라 변하는 콜백 (예: useState 가 생성한 setter wrapper) 이면 **첫 render 의 closure 만 호출됨**. 현 코드는 EditPage 통합이 없어 무위험이나, 통합 시 noticeably **양방향 연동 깨질 위험** — 회귀 권고.
- 판정: **CODE PASS (정적), USER 미증명 (통합 부재)**

### 2.4 R-A4 (Typeahead ACL)

- **4중 방어 패턴** — 본 Phase 5 의 가장 견고한 영역:
  1. **Pydantic 모델** `UserSearchItem = {user_id, display_name}` — email/role/status 자체 부재 (회귀 `test_response_model_excludes_email_role_status`)
  2. **라우터** `viewer_user_id = actor.actor_id` — ActorContext only, query/body 주입 차단 (회귀 `test_viewer_user_id_must_be_keyword_only`)
  3. **Repository 시그니처** `viewer_user_id` keyword-only — positional → TypeError (회귀 동상)
  4. **SQL** `user_org_roles` 2회 JOIN + `viewer_uor.user_id = %s` + `u.id != %s` + `status='ACTIVE'` + `ILIKE %s ESCAPE '\\'` (회귀 `test_sql_includes_user_org_roles_join_and_viewer_self_exclude`, `test_sql_params_include_viewer_id_twice`)
- **부족**: 단위 회귀가 모두 mock 기반 — **실 DB JOIN 동작 미검증**. 통합 회귀(TestClient + 2 organization 매트릭스) 잔여. timing attack 정량 측정 부재.
- 판정: **CODE PASS (정적·단위 견고), 통합 회귀 부재**

---

## 3. 공식 종결 전 닫혀야 할 5건 (CONDITIONAL)

본 5건은 회고 §4 의 잔여 항목 중 **Phase 5 §5.1 게이트의 실 충족에 직결되는 항목**만 추린 것. 별 라운드로 이월하더라도, "Phase 5 1라운드 공식 종결" 선언 전에는 닫히는 게 정합이다.

### 3.1 본문 ↔ Panel 양방향의 **본문 측 누락** (FG 5-2)

- **사실**: DocumentDetailPage 는 `<NodeRenderer nodes={nodes} />` 로 read-only 렌더 — AnnotationMark `<span class="annotation-mark">` 인식 0. DocumentEditPage 는 `<DocumentTipTapEditor>` 로 본문 mark 가 렌더되지만 **AnnotationsPanel / selectedAnnotationId state 통합 0건** (grep 결과 0 매칭).
- **영향**: Phase 5 §5.1 항목 5 ("AnnotationMark click → AnnotationsPanel 자동 펼침 + highlight + scroll") 가 **사용자 시점 어디서도 동작하지 않음**.
- **회고 §2 매트릭스의 ✅ 표기와 충돌** — Disagreement Record §4 항목 1.
- **권고**: 별 라운드 진입 전 (a) NodeRenderer 의 mark 인식 또는 (b) DocumentEditPage 의 AnnotationsPanel 통합 — 둘 중 하나 필수. 또는 회고 §2 매트릭스를 🟡 로 정정.

### 3.2 AnnotationGutter 좌측 도트 미구현 (FG 5-2)

- **사실**: `AnnotationGutter.tsx` 가 좌측 여백 도트 대신 "주석 분포 요약 박스" stub만 표시. `(gutter 미완)` 라벨 명시.
- **영향**: Phase 5 §5.1 항목 6 (Gutter 노드별 카운트 정확) — 회고 §2 가 🟡 로 정직 표기 ✅. 단 task5-2 §7 성공 기준의 핵심 누락.
- **권고**: 별 라운드 우선순위 상위. NodeRenderer 가 `data-node-id` 마운트 필요.

### 3.3 MentionPopup 마운트 부재 (FG 5-3)

- **사실**: `MentionPopup` 컴포넌트가 구현됐으나 **어디에도 import/mount 0건** (grep 결과 자체 파일 외 0건). AnnotationsPanel 의 textarea 에 `@<prefix>` 패턴 감지 통합 미수행.
- **영향**: Phase 5 §1.4 기대 결과 "멘션 typeahead — `@` 입력 시 floating 자동완성 popup" — **사용자 시점 어디서도 동작 안 함**. R-A4 정합은 견고하나 실제 호출 path 부재 → 게이트의 "정합성 검증 + 누설 가능 경로 분석" 도 일부 무위 (호출되지 않는 함수는 누설 불가능).
- **권고**: AnnotationsPanel textarea 의 `@` typeahead 통합 — 별 라운드 진입 전 닫는 게 권장. 통합 시 401 / rate limit 429 TestClient 회귀도 함께.

### 3.4 save_draft sanitize 미호출 (FG 5-2)

- **사실**: `cleanInvalidAnnotationMarks` 가 구현·export됨 + `useDocumentAnnotations.validIds` 가 derived state 로 노출됨. 단 **save_draft 진입 직전 호출 path 0건** (grep 결과: 정의/회귀 2건 외 사용 0).
- **영향**: annotation 삭제 후 본문 mark stale → 다음 save_draft 시 invalid `annotation_id` 가 그대로 DB 영속화. ADR §c 의 round-trip 안전성이 client→DB 의 한 방향에서 깨질 수 있음.
- **권고**: DocumentEditPage 의 onChange 또는 saveDraft 직전 sanitize 호출 — 비용 낮음, 단순 통합. 별 라운드 진입 전 권장.

### 3.5 Mark 통합 ADR 별 reviewer 합의 미수행 (FG 5-1)

- **사실**: Mark_통합_ADR §12 자체가 "Claude 1차 작성 + 합의자 = 작성자" 잠정 합의 명시. 본 보고서가 별 reviewer 1차 역할이지만 동일 모델 (Claude). 헌법 제27조 정합은 Codex 또는 사람 reviewer 가 필요.
- **권고**: P1 승인 게이트의 일부 — Codex 검토 또는 `@최철균` 의 reviewer 위임 결정.

---

## 4. Disagreement Record (헌법 제34조)

### 4.1 회고 §2 매트릭스 #5 ✅ 표기

- **회고 진술**: 항목 5 "AnnotationMark click → AnnotationsPanel 자동 펼침 + highlight + scroll" → **✅ 충족**
- **본 검수 입장**: **🟡 부분 충족** — panel→panel 자동 펼침 (AnnotationsPanel의 `selectedAnnotationId` useEffect → setOpen + scrollIntoView) 은 정합. 그러나 본문 mark click → panel 활성화의 본문 측 path 가 사용자 환경에서 부재 (§3.1). 게이트의 의도 (AnnotationMark click 이 사용자 환경에서 panel 펼침을 트리거) 가 미충족.
- **근거**:
  - DocumentDetailPage line 250: `<NodeRenderer nodes={nodes} />` — read-only, mark 미인식
  - DocumentEditPage: grep `onAnnotationClick|selectedAnnotationId|AnnotationsPanel|AnnotationMark` → 0 매칭
  - 검수보고서 FG5-2 §5 #2 #5 자체가 잔여로 명시
- **권고**: 회고 §2 매트릭스 #5 를 🟡 로 정정하거나, "panel→panel 부분 PASS / 본문→panel 잔여" 로 분리 표기.

### 4.2 회고 §2 #1 R-A1 ✅ 표기

- **회고 진술**: R-A1 충족 (FG 5-1 ADR §c + FG 5-2 sanitize 7건)
- **본 검수 입장**: **🟢 CODE PASS, S5 backend round-trip 회귀 미수행**. ADR §11 의 S5/S6 가 task5-2 위임이라 명시됐으나 task5-2 산출물에 backend pytest `test_save_draft_preserves_marks_roundtrip.py` 부재. ADR §c 의 "TipTap default 결정성" 가정이 실 환경에서 미증명.
- **권고**: 회고가 ✅ 보다는 "🟢 정합 확인 + 통합 회귀 잔여" 정도로 정직.

---

## 5. 헌법 정합성 결과

| 조항 | 충족 | 근거 |
|------|----|----|
| 제5조 Agent-Facing Contract | ✅ | `/api/v1/users` 의 description / input / response model / 권한 / 실패 코드 / rate limit 모두 명시 |
| 제11조 함수도서관 인덱스 | ✅ | `frontend.md` §8/§9/§10, `backend.md` §8 — 본 Phase 신규 항목 모두 등재. 중앙 인덱스 모델 (2026-04-25 정책) 정합 |
| 제12조 Index Updates Must Follow Code Changes | ✅ | 코드 변경과 동시 갱신 — 같은 PR / 같은 일자(2026-05-11) commit |
| 제14조 에러 계약 | ✅ | `ApiAuthenticationError` + structured error code — 본 Phase 신규 에러 경로는 인증 401 만 |
| 제15조 의도 ↔ 구현 일치 | 🟡 | §4.1 Disagreement — 회고 매트릭스의 실 동작 표기가 일부 낙관 |
| 제17조 외부 문서 데이터로 취급 | N/A | 본 Phase 외부 문서 의존 0 |
| 제24조 PII 장기 저장 금지 | ✅ | audit log 가 `q` prefix 미저장 — `logger.info` 명시 |
| 제27조 No Self-Review as Final Review | 🟡 | §3.5 — Mark 통합 ADR 의 별 reviewer 부재. 본 검수는 별 reviewer 1차이나 동일 모델 (Claude). 정식 종결은 Codex 또는 사람 |
| 제32조 PRs Must Be Reviewable | ✅ | 4 FG 분리 산출 + Change Boundary 의 의도 명시 |
| 제36~40조 테스트 계층 | 🟡 | Unit ✅ / Contract 🟡 (단위 회귀 위주) / Policy 🟡 (R-A4 mock) / Eval 미적용 / Recovery 미적용 |
| 제41~43조 Tests-as-Specs | ✅ | skip/xfail 신규 0건. 회귀 0건 weakened |
| 제45조 Human Approval Gates | 🟡 | `@최철균` P1 승인 대기 — Mark ADR + API 1 endpoint + 사이드바 도입 |
| 제46조 Approval Binds to Change Set | N/A | 승인 아직 미수령 — 본 검수 후 게이트 |
| 제47조 회귀 케이스 보존 | ✅ | BUG-FG22-02 (HashtagMark inclusive) 의 회귀가 본 Phase 의 inclusive=false 정책으로 보존 + markNames 정본 위에 명시 |
| 제48조 Reproducibility | 🟡 | 실 pytest/node:test 숫자 (587 / 9 / tsc 0) 가 운영자 환경 의존 — 본 검수에서 재현 불가 |

---

## 6. 보안 정합 (FG 5-3 보안취약점검사보고서 재검증)

| 영역 | 본 검수 판단 |
|------|------------|
| R-A4 ACL 누설 (모델/라우터/repo/SQL 4중) | ✅ 견고. mock 단위 회귀 9건 일관. 단 실 DB JOIN 통합 회귀 부재 |
| SQL Injection (psycopg2 placeholder + ILIKE wildcard escape) | ✅ `query.replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_") + "%"` 정합. `ESCAPE '\\'` 절 동반 |
| Rate Limit (60/min slowapi) | ✅ 코드 적용. 429 통합 회귀 부재 |
| PII (제24조) | ✅ `logger.info` 에 q prefix 부재 — 정적 grep 확인 |
| Timing attack | 🟡 정량 측정 부재. JOIN+LIMIT+INDEX 로 mitigation 의도는 명시 |
| 응답 모델 누설 | ✅ Pydantic `UserSearchItem` 이 user_id+display_name 만 — email/role/status 자동 차단 |

보안 보고서가 식별한 잔여 (TestClient 통합 회귀 + timing 정량 측정) 는 본 검수에서도 동일하게 확인. **추가 발견 없음**.

---

## 7. 부가 관찰 (별 라운드 입력)

1. **AnnotationMark stale closure 잠재 위험** (§2.3) — `useEditor` 가 dependency 배열 없이 호출되므로 `onAnnotationClick` prop 변경 미반영. EditPage 통합 시 회귀 1건 추가 권고.
2. **MentionPopup `aria-activedescendant` 부재** — `role="listbox"` + `role="option"` + `aria-selected` 는 있으나 active 항목의 ID 가 listbox 에 묶이지 않음. 스크린리더 UX 마이너 미흡.
3. **DocumentSidebar 의 `metaContent` prop 미사용** — DocumentDetailPage 에서 본 prop 을 전달하지 않아 "meta" 탭은 `<p>메타 정보가 없습니다.</p>` 폴백만 표시. 메타데이터 / 워크플로 이력 패널이 본문 하단에 그대로 남음. 작업지시서 §2.1.1.b 의 "탭 콘텐츠 마운트" 부분 충족.
4. **`AnnotationsPanel.tsx` toast 7건 tsc 오류 (Phase 3 잔재)** — 회고 §5.2 #3 명시. Phase 6 진입 전 별 cleanup.
5. **FG 5-4 의 탭 수 5 vs 작업지시서 6** — VectorizationPanel / AgentProposalsTab 흡수 미수행. 회고 §4.3 #6 잔여. 단순화는 합리적 trade-off, 단 task5-4 §2.1.1.b 의 명시와 차이.

---

## 8. 최종 권고

### 8.1 본 라운드 PASS 항목 (운영 게이트 통과 권장)

- FG 5-1 Mark 통합 ADR + markNames.ts — **별 reviewer 합의 후 즉시 PASS**
- FG 5-3 사용자 검색 API + R-A4 4중 방어 — **TestClient 401/429/multi-org 회귀 1세션 추가로 PASS**
- FG 5-4 DocumentSidebar 5 탭 (desktop 고정) — **현재로도 PASS, 4 viewport drawer / UI 리뷰 ≥ 2회 / focus trap 은 별 라운드**

### 8.2 본 라운드 PASS 보류 항목 (§3 의 5건)

다음 5건은 회고 §4 의 잔여 중 **공식 종결 선언 전 닫는 것 권장** (별 라운드라도 종결 게이트의 일부):

1. **§3.1 본문→Panel 양방향 본문 측 통합** (NodeRenderer mark 인식 또는 DocumentEditPage AnnotationsPanel 통합) — 최소 하나
2. **§3.2 AnnotationGutter 좌측 도트** — task5-2 §7 핵심 누락
3. **§3.3 MentionPopup 마운트 통합** — AnnotationsPanel textarea
4. **§3.4 save_draft sanitize 호출** — 단순 통합, 우선순위 높음 (round-trip 안전)
5. **§3.5 Mark 통합 ADR 별 reviewer 합의** — Codex 또는 사람

### 8.3 회고 매트릭스 정정 권고

- §2 매트릭스 #5 → 🟡 (panel→panel 부분 / 본문→panel 잔여)
- §2 매트릭스 #1 (R-A1) → 🟢 "정합 확인 + 통합 round-trip 잔여" 분리 표기

### 8.4 운영자 (`@최철균`) 결정 입력

다음 4 결정이 P1 승인 게이트의 입력:

1. **Phase 5 1라운드 공식 종결 선언 여부**:
   - **옵션 A**: §3 의 5건 모두 닫은 후 종결 — 안전, 일정 +1 세션
   - **옵션 B**: §3 의 #1/#3/#4 만 닫고 (Gutter + 별 reviewer 는 별 라운드) 종결 — 균형, 일정 +0.5 세션
   - **옵션 C**: 현 상태로 종결 + 회고 매트릭스 정정만 — 빠름, 사용자 시점 통합 부재 인정
2. **별 reviewer 위임**: Codex 또는 사람 reviewer 의 Mark 통합 ADR 검토
3. **FG 5-5 (한국어 username 정책)** 진행 여부
4. **Phase 6 진입 시점** — 본 라운드 종결 후 또는 §3 5건 닫은 후

---

## 9. 결론

S3 Phase 5 1라운드는 **코드 정합·단위 회귀·R-A4 4중 방어 패턴의 견고함**으로 샌드박스 1차 PASS. 본 검수는 검수보고서 4편 + 보안보고서 + 회고의 솔직성에 대체로 동의한다 (잔여 항목을 숨기지 않음).

다만 회고 §2 매트릭스의 일부 ✅ 표기와 §3 의 5건 (특히 §3.1 본문측 통합, §3.3 MentionPopup 마운트 부재) 가 **"사용자 시점에서 동작하는 path 부재"** 라는 같은 패턴을 가진다 — Phase 5 §1.4 의 기대 결과 (AnnotationMark 인라인 mark + Gutter + 멘션 typeahead + 우측 사이드바) 중 사이드바 외 3건이 *코드는 있으나 호출되지 않는다*.

이는 **헌법 제15조 (의도-구현 일치)** 의 회색 영역이다. 1라운드의 의도는 "기반 마련 + 통합은 별 라운드" 로 합리적이나, 그 의도가 회고 매트릭스에는 일관되게 반영되지 못함. §4 Disagreement 와 §8.3 정정 권고로 닫는다.

**최종 판정**: **샌드박스 1차 PASS / 공식 종결 CONDITIONAL** — §3 의 5건과 §4 Disagreement 의 해소가 정식 종결의 입력이다.

---

## 10. 참조

- `docs/개발문서/S3/phase5/Phase 5 개발계획서.md`
- `docs/개발문서/S3/phase5/작업지시서/task5-1.md` ~ `task5-4.md`
- `docs/개발문서/S3/phase5/산출물/Mark_inventory_실측.md`
- `docs/개발문서/S3/phase5/산출물/Mark_통합_ADR.md`
- `docs/개발문서/S3/phase5/산출물/Annotation_Decoration_방식.md`
- `docs/개발문서/S3/phase5/산출물/FG5-1_검수보고서.md` ~ `FG5-4_검수보고서.md`
- `docs/개발문서/S3/phase5/산출물/FG5-3_보안취약점검사보고서.md`
- `docs/개발문서/S3/phase5/산출물/Phase5_1라운드_종결회고.md`
- `CONSTITUTION.md` 제5·11·12·14·15·17·24·27·32·34·36~43·45·46·47·48조
- `AGENT_MODE.md` §3.3 (extended) §2.1 (dual-agent Actor 배정)
- `AUTOMATION.md` §3 (Dispatcher Policy) §8 (Layer별 경계)
- `CLAUDE.md` §3 (Review 체크리스트)

---

*작성: 2026-05-14 | S3 Phase 5 1라운드 독립 검수보고서 — 샌드박스 PASS / 공식 종결 CONDITIONAL — 별 reviewer 1차 (헌법 제27조)*

---

## 11. 2026-05-14 통합 패치 결과 (본 보고서 발행 후속)

본 보고서 §3 의 5건 + Codex 검수보고서 §5 P1/P2 9건을 통합 패치로 닫음. 작업 항목 A~I 9건 + 문서 갱신.

| §3 항목 | 상태 | 패치 위치 |
|---------|----|---------|
| §3.1 본문→Panel 양방향의 본문 측 누락 | ✅ (2026-05-14 1차 + **2026-05-18 보강**) | (a) `NodeRenderer.tsx` 가 `contentSnapshot` prop 으로 mark 인식 → DetailPage 본문에 `<span class="annotation-mark">` 렌더. (b) `DocumentDetailPage.tsx` 본문 wrapper 에 delegated click → `setSelectedAnnotationId`. (c) `DocumentEditPage.tsx` 에 `selectedAnnotationId` state + `onAnnotationClick` 연결 + AnnotationsPanel 우측 마운트. **2026-05-18 Codex 2차 검수 F-04 시정**: section 재귀에서 `contentSnapshot={null}` 로 끊겨 section 하위 paragraph/heading 의 annotation/hashtag/wikilink mark 가 plain text 로 렌더되던 회귀 발견·수정. 자식 NodeRenderer 호출에 `contentSnapshot` 그대로 전달 + 실 렌더 결과 HTML 검증 회귀 3건 추가 (`NodeRendererMarkFg52.test.tsx` §D). |
| §3.2 AnnotationGutter 좌측 도트 미구현 | ✅ | `AnnotationGutter.tsx` 재작성. ResizeObserver + window scroll/resize debounce 50ms + `[data-node-id]` 좌표 추적. DetailPage 본문 좌측 (`pl-7 relative` wrapper) 마운트. |
| §3.3 MentionPopup 마운트 부재 | ✅ | `AnnotationsPanel.tsx` 의 신규 주석 textarea 에 통합. `MENTION_PREFIX_REGEX` cursor 직전 `@<prefix>` 감지 + 선택 시 `@display_name ` 치환 + cursor 이동. backend `extract_mentions` 가 영문 username 자동 매칭 → `mentioned_user_ids` 채움. 한국어는 FG 5-5 별 라운드 (MentionPopup.tsx docstring 명시). |
| §3.4 save_draft sanitize 미호출 | ✅ (2026-05-14 1차 + **2026-05-18 보강**) | `DocumentEditPage.tsx` 의 `saveMutation.mutationFn` 가 `cleanInvalidAnnotationMarks(doc, validAnnotationIdsRef.current)` 호출. `validAnnotationIdsRef` 가 최신 set 유지 (stale closure 차단). **2026-05-18 Codex 2차 검수 F-05 시정**: annotation list 로딩 중 빈 validIds 로 sanitize 가 정상 mark 까지 stale 로 판정해 삭제하는 회귀 발견·수정. `useDocumentAnnotations` 가 `isLoaded` 노출 + `sanitizeForSave` 가드 헬퍼 (`frontend/src/features/editor/sanitizeForSave.ts`) 도입. 단위 회귀 5건 추가 (`SanitizeForSaveFg52F05.test.tsx`). |
| §3.5 Mark 통합 ADR 별 reviewer 합의 | ⏳ | Codex 1차 검수보고서 (2026-05-14) 가 1차 별 reviewer 역할. **2026-05-18 Codex 2차 검수**가 1차 패치 자체의 재현성 결함 5건을 추가 발견·시정함으로써 별 reviewer 역할이 한 번 더 작동. 본 회고/종결 게이트의 `@최철균` P1 승인은 여전히 잔여. |

### 11.1 추가 회귀 (운영자 실측 — 2026-05-14 1차 / 2026-05-18 재측정 갱신)

| 회귀 파일 | 건수 | 영역 | 실측 (2026-05-18) |
|---------|-----|----|----|
| `frontend/tests/NodeRendererMarkFg52.test.tsx` | 14 + **3 (F-04)** | indexInlineByNodeId / wrapWithMarks 우선순위 / robustness / **section 재귀 실 렌더 회귀** | ✅ PASS (`npm run test` 640/0 — F-01 시정 후 처음 게이트 통과) |
| `frontend/tests/MentionDetectFg53.test.tsx` | 14 | MENTION_PREFIX_REGEX cursor 직전 감지 (boundary / 한국어 / 64자 / 이메일) | ✅ PASS |
| `frontend/tests/SanitizeForSaveFg52F05.test.tsx` (신규) | **5 (F-05)** | annotation loading 가드 — 로드 미완료/완료 × 빈/매칭 validIds 매트릭스 | ✅ PASS |
| `backend/tests/integration/test_user_search_fg53_integration.py` | 18 | TestClient + dependency_overrides — 401·400 (Mimir validation handler 매핑) / R-A4 query 주입 / SQL injection 6 payload / 응답 모델 / truncated / trace | ✅ PASS — **F-02 시정으로 외부 DB 의존 없이 in-process 통합 검증** (`db_dependency` override) |
| `backend/tests/unit/test_annotations_mentions_fg55.py` | 23 | FG 5-5 한국어 멘션 + display_name fallback | ✅ PASS — **F-03 시정으로 stub row `role_name` 컬럼 정합** |

> 2026-05-14 의 "운영자 실측 ✅" 는 Codex 2차 검수 (2026-05-18) 가 실제 환경에서
> 재현 실패 (frontend test gate 가 TS5107 로 종료, backend 통합이 실 DB 접속,
> FG 5-5 fixture `KeyError`) 를 보고함에 따라 정정되었다. 본 절의 ✅ 는
> F-01~F-05 시정 후 같은 환경에서 다시 측정한 결과다.

**Fix 적용 사항** (운영자 실행 중 발견):
- `AuthMethod.JWT` 부재 → `AuthMethod.BEARER` 사용 (실제 enum 값: SESSION / BEARER / API_KEY / INTERNAL_SERVICE)
- `MagicMock(spec=ActorContext)` 가 dataclass 와 충돌 가능 → 실제 `ActorContext(...)` 인스턴스 생성
- FastAPI 기본 422 가정 → Mimir global `request_validation_error_handler` 가 `RequestValidationError → 400 validation_error` 로 매핑 (`app/api/errors/handlers.py:172`)

### 11.2 §4 Disagreement 해소

- §4.1 매트릭스 #5 (AnnotationMark click → panel 활성화): **2026-05-14 본문 측 path 추가 → ✅ 로 승격**. DetailPage 의 NodeRenderer mark 인식 + delegated click + AnnotationGutter + EditPage 의 onAnnotationClick 연결로 본문 → panel 흐름 완성.
- §4.2 매트릭스 #1 (R-A1): 🟢 (코드 차원 PASS + saveDraft sanitize 호출 통합 추가). backend round-trip pytest 만 별 라운드.

### 11.3 §8 권고 결과

- §8.1 PASS 항목 — 그대로 유지
- §8.2 PASS 보류 5건 — 4건 ✅, 1건 (별 reviewer) ⏳ → 운영자 결정 입력으로 이관
- §8.3 매트릭스 정정 — 종결회고 §2 갱신 완료
- §8.4 운영자 결정 — Codex 보고서 §7 의 분류와 정합. 옵션 A (모두 닫고 종결) 의 코드 부분이 본 패치로 완료. 남은 것은 (a) Codex 1차 검수 → `@최철균` P1 승인 (b) UI 디자인 리뷰 ≥ 2회 (c) jsdom 도입 (별 라운드)
