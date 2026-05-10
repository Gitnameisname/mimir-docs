# task5-1 — Mark 통합 ADR + 직렬화 round-trip 회귀 (FG 5-1)

**작성일**: 2026-05-10
**Phase / FG**: S3 Phase 5 / FG 5-1
**상태**: 착수 대기 (Phase 4 1라운드 게이트 통과 권장)
**담당**: 설계 Claude / 구현 Claude (또는 Codex) / 리뷰 Codex (extended — ProseMirror 도메인 + ACL 영향)
**예상 소요**: 3~4일 (사실 표면화 §2.1.0 + ADR §2.1.1 + 회귀 §2.1.4 + 합의)
**선행 산출물**: `docs/개발문서/S3/phase5/Phase 5 개발계획서.md` (P1 — `@최철균` 승인 완료 가정)
**후행**: task5-2 / task5-3 / task5-4 모두 본 FG 의 ADR 결정 위에서 진행

---

## 1. 작업 목적

S3 Phase 1~3 누적으로 도입된 TipTap mark/extension 계열의 **schema 우선순위·결합 규칙·직렬화 round-trip** 을 한 ADR 로 고정한다. ADR 합의 이전에는 task5-2 (AnnotationMark) / task5-3 (멘션 typeahead) 가 schema 충돌 위험에 노출되므로 본 FG 가 Phase 5 진입 게이트다.

또한 본 FG 실측에서 **현재 구현된 mark 가 2 종 (NodeId / HashtagMark) 뿐이며 WikiLinkMark (FG 2-3) 가 미구현** 임이 확인됐다. 이 사실을 ADR §"실측 inventory" 에서 표면화하고, WikiLinkMark 의 본 Phase 흡수 여부를 운영자 결정 사항으로 등록한다(§2.1.0 참고).

본 FG 산출물:
1. **`docs/개발문서/S3/phase5/산출물/Mark_통합_ADR.md`** — schema / 우선순위 / 결합 / 직렬화 정본
2. **`docs/개발문서/S3/phase5/산출물/Mark_inventory_실측.md`** — 현 구현 mark / 미구현 mark / Phase 5 신설 예정 mark inventory
3. **직렬화 round-trip 회귀 테스트 ≥ 10 시나리오** — node:test (frontend) + 가능 시 backend snapshot_sync_service round-trip 동등성

---

## 2. 작업 범위

### 2.1 포함

#### 2.1.0 사실 표면화 (Mark inventory 실측) — **선행 게이트**

다음 항목을 실측해 `Mark_inventory_실측.md` 보고서로 남긴다. 본 보고서가 ADR 의 입력 사실 기반이므로 1차 운영자 검토 후 ADR 작성 진입.

- **현 구현 mark/extension inventory**:
  - `frontend/src/features/editor/tiptap/extensions/NodeId.ts` — Global Attribute (Mark 아님 — 노드 속성). schema name / `keepOnSplit` / `appendTransaction` / data attribute 직렬화 키 정리
  - `frontend/src/features/editor/tiptap/extensions/HashtagMark.ts` — Inline Mark. `inclusive` / `excludes` / attribute (`tag`) / InputRule 정규식 / HTML 직렬화 (`<span class="tag-pill" data-tag>`) 정리
- **미구현 mark inventory**:
  - **WikiLinkMark** — Phase 2 FG 2-3 의 task2-3 §4 Step 6 에 `frontend/src/features/editor/tiptap/extensions/WikiLinkMark.ts` 신설이 명시되었으나, 현 코드베이스에 파일이 부재하고 `docs/개발문서/S3/phase2/산출물/` 에 FG 2-3 종결보고서가 없음. **FG 2-3 미완료** 상태로 결론 (실측 시 다시 확인).
  - **AnnotationMark** — task5-2 에서 신설 예정 (FG 5-2)
- **Phase 5 신설 예정 mark inventory** — task5-2 / task5-3 의 신설 항목 (AnnotationMark / MentionSuggestion) 을 잠정 명시
- **TipTap editor 진입점** — `frontend/src/features/editor/tiptap/DocumentTipTapEditor.tsx` 의 `useEditor()` extensions 배열 (현재: StarterKit + Placeholder + NodeId + HashtagMark)
- **ProseMirror schema 우선순위 현황** — TipTap 가 자동으로 결정한 우선순위 (extension 등록 순서) 를 ProseMirror schema spec 으로 출력해 첨부

> 본 보고서가 §2.1.1 ADR 의 "사실 baseline" 이므로, ADR 의 결정이 보고서를 갱신하면 보고서도 같이 commit (Meta-2 — 사실/규범 분리, 사실 변경은 별 항목으로 추적).

#### 2.1.1 Mark 통합 ADR — `Mark_통합_ADR.md`

ADR 본문에 다음 섹션 의무 포함:

##### a. 적용 범위 mark 목록

본 ADR 이 규율하는 4 항목을 명시. 단, **실측 결과 미구현인 항목은 "Phase 5 흡수 / Phase 6 이월 / FG 2-3 별 라운드 재개" 중 하나를 결정**해야 한다.

| 항목 | 종류 | 현재 상태 | 본 ADR 결정 |
|------|------|---------|-----------|
| NodeId | Global Attribute | 구현 완료 (FG 1-2) | 변경 없음 — 우선순위 계산 시 노드 속성으로 취급 |
| HashtagMark | Inline Mark | 구현 완료 (FG 2-2) | schema 우선순위 / 결합 규칙 본 ADR 에서 명시 |
| WikiLinkMark | Inline Mark | **구현 완료 (FG 2-3, 2026-05-10 — Phase 2 미완료 흡수)** | (a) 본 Phase 흡수 완료. schema attribute `target`, `inclusive: false`, HTML data-attr `data-wikilink-target`, CSS class `wikilink` (+ modifier `wikilink--resolved/--ambiguous/--missing`). 본 ADR §d 표·§e 정책 준수. |
| AnnotationMark | Inline Mark | 미구현 (task5-2 에서 신설) | schema 우선순위 / 결합 규칙 본 ADR 에서 명시 |

> **WikiLinkMark 처리 결정 — 갱신 (2026-05-10)**: Phase 5 진입 전 사전 합의에 따라 **(a) 본 Phase 흡수** 가 채택됐다. 단 흡수 작업이 본 task5-1 ADR 작성과 동시에 진행되지 않고, **선행적으로 Phase 2 FG 2-3 의 task2-3.md 를 재진입해 종결**했다 (`docs/개발문서/S3/phase2/산출물/FG2-3_종결보고서.md` — 2026-05-10). 본 ADR 작성 시 WikiLinkMark 의 schema 가 이미 코드에 존재하므로 §a~§f 의 결합 규칙 / 직렬화 / inclusive 정책을 모두 적용 가능.

##### b. ProseMirror schema 우선순위 / 결합 규칙

다음 케이스에 대해 **결정된 동작 + 근거 + 회귀 테스트 시나리오 ID** 를 표로 명시.

| 케이스 | 본문 | 기대 직렬화 / 동작 | R-A 매핑 |
|-------|------|---------------|---------|
| C1: HashtagMark 안에 AnnotationMark | `#tag` 의 일부에 주석 | 두 mark 모두 적용. `<a-mark><tag-pill>#tag</tag-pill></a-mark>` 또는 동등한 nested 직렬화 | R-A2 |
| C2: AnnotationMark 안에 HashtagMark | 주석 mark 가 걸린 텍스트에 `#tag` 입력 | 두 mark 모두 적용. 직렬화 순서는 ProseMirror schema priority 기준 단일 정본 | R-A2 |
| C3: HashtagMark 의 InputRule 이 AnnotationMark 영역에서 발동 | 주석 mark 가 걸린 텍스트에서 `#word ` 입력 | InputRule 동작 보존 + 두 mark 동시 적용 | R-A2 + R-A3 |
| C4: AnnotationMark 클릭 | mark 클릭 시 cursor / selection 동작 | TipTap default mark 동작 — 클릭 위치에 cursor, 본문 편집 차단 없음. AnnotationsPanel 활성화는 별 이벤트 (panel highlight) | R-A3 |
| C5: NodeId 변경 없이 mark 만 toggle | 텍스트 일부에 mark 적용/해제 | NodeId 안정 유지 (`keepOnSplit:false` 와 무관한 영역 — mark 는 노드 속성 미변경) | R-A1 + Phase 1 회귀 |
| C6: save_draft round-trip | 4 mark 동시 적용된 ProseMirror doc → JSON → DB → 다시 doc | ProseMirror JSON 기준 정확히 일치. mark order 정규화 함수가 있다면 그 결과 일치 | R-A1 |
| C7: HashtagMark 결합 (`inclusive:false`) 와 AnnotationMark 결합 | 주석 끝에서 텍스트 입력 | HashtagMark 의 `inclusive:false` 가 AnnotationMark 의 `inclusive` 결정과 충돌하지 않음 | R-A1 |
| C8: WikiLinkMark (도입 결정 시) 와 다른 mark 동시 | (운영자 결정 후 추가) | 결정 시 본 표 확장 | R-A2 |

각 케이스의 회귀 테스트 ID 를 §2.1.4 회귀 시나리오와 매핑.

##### c. 직렬화 정본 (canonical serialization)

- ProseMirror JSON 의 mark 배열 순서는 **TipTap 의 default order** 를 따른다 (extension 등록 순서 → schema spec). 우리는 이를 변경하지 않는다.
- 단, **save_draft 직전에 mark order 정규화** 가 필요한지 결정. 현재 저장 모델 단일성 (`content_snapshot` JSONB) 위에서 round-trip 일관성을 보장하려면 다음 중 하나 채택:
  - (i) **TipTap 가 시리얼라이즈한 그대로 저장** — 추가 정규화 없음. round-trip 동등은 TipTap 의 결정성에 의존.
  - (ii) **저장 직전 mark order 정렬** (예: `nodeId` (없음, 노드 속성) → `annotation` → `wiki_link` → `hashtag`) — 결정성 보강.
- 본 ADR 에서 (i) 와 (ii) 중 결정. 결정 후 §2.1.4 회귀 시나리오에 동등성 기준 명시.

##### d. attribute 키 / data attribute / CSS class 네임스페이스

| mark | attribute | HTML data attr | CSS class |
|------|-----------|----------------|----------|
| HashtagMark | `tag` (string) | `data-tag` | `tag-pill` |
| AnnotationMark | `annotation_id` (uuid) | `data-annotation-id` | `annotation-mark` |
| WikiLinkMark (도입 결정 시) | `target_title` (string) + `target_doc_id?` | `data-wikilink-target` + `data-wikilink-doc-id` | `wikilink-mark` |
| NodeId (Global Attr) | `node_id` (uuid) | `data-node-id` | — |

> mark / data attribute 네임스페이스는 본 ADR 이후 변경 시 **CSS / 백엔드 파서 / TipTap rule 동시 수정** 이 필요하다. ADR 변경은 별 P1 항목.

##### e. excludes / inclusive 정책

- HashtagMark: `inclusive: false` (현재 코드 그대로)
- AnnotationMark: `inclusive: ` ← 본 ADR 결정. 권장 `false` (주석 끝에서 입력하면 mark 자동 확장 금지)
- WikiLinkMark: `inclusive: false` (도입 시)
- `excludes` 는 빈 문자열 (다른 mark 와 결합 가능). C1~C3 케이스 보장.

##### f. ProseMirror 변환 / Backend 영향

- `backend/app/services/snapshot_sync_service.py` 의 `rebuild_nodes_from_snapshot` 는 mark 정보를 무시하고 텍스트 + node_id 만 활용 (현행 유지). 단, 본 ADR 변경이 백엔드 파서 (`tag_rules.py`) 의 `#tag` 파싱과 일관해야 한다 — 본문 내 mark 안에 다시 `#tag` 가 있을 때 backend 가 동일하게 인식하는지 회귀 1건 추가 (§2.1.4 시나리오 S6).

#### 2.1.2 mark 우선순위 / 결합 코드 적용

ADR §b 결정에 따라 다음 변경:
- `frontend/src/features/editor/tiptap/extensions/HashtagMark.ts` — `excludes` 또는 schema 우선순위 명시 (현재 코드는 default 의존). ADR 결정에 맞춰 명시적 schema spec 추가.
- AnnotationMark 신설은 **task5-2 범위** — 본 FG 에서는 schema spec 텍스트만 ADR 에 명시하고, 실제 파일은 task5-2 에서 생성.
- WikiLinkMark 처리는 운영자 결정 결과에 따라.

#### 2.1.3 mark order 정규화 (§c (ii) 채택 시)

만약 ADR 결정이 (ii) 라면:
- `frontend/src/features/editor/tiptap/markOrder.ts` (신규) — `normalizeMarkOrder(doc: ProseMirrorDoc): ProseMirrorDoc` — save_draft 직전 호출
- 호출 지점: `DocumentTipTapEditor.tsx` 의 onUpdate 또는 saveDraft 진입 직전
- 단위 테스트: 입력 mark 순서 무관하게 출력 동등 (≥ 6 시나리오)

만약 (i) 채택 시 본 §2.1.3 은 생략. ADR 에 "정규화 미도입" 명시.

#### 2.1.4 직렬화 round-trip 회귀 테스트 (≥ 10 시나리오)

`frontend/src/features/editor/tiptap/__tests__/markRoundtrip.test.ts` (신규):

| ID | 시나리오 | 매핑 |
|----|---------|------|
| S1 | NodeId + HashtagMark 단독 — 텍스트 → JSON → 텍스트 round-trip | C5 |
| S2 | HashtagMark 안에 AnnotationMark | C1 |
| S3 | AnnotationMark 안에 HashtagMark | C2 |
| S4 | mark 토글 후 NodeId 안정 (3 회 toggle 동안 node_id 불변) | C5 |
| S5 | save_draft round-trip — 4 mark 동시 적용된 doc → backend → 다시 fetch → ProseMirror JSON 동일 | C6 |
| S6 | backend `tag_rules.py` 가 AnnotationMark 안의 `#tag` 도 인식 | §f |
| S7 | mark 의 inclusive 정책 — 주석 끝에서 입력 시 mark 미확장 | §e |
| S8 | InputRule 동작 — 주석 mark 안에서 `#word ` 입력 시 HashtagMark 적용 | C3 |
| S9 | mark order 정규화 (선택 — ADR (ii) 채택 시) — 입력 순서 무관 동일 출력 | §c |
| S10 | 클릭 동작 — AnnotationMark 클릭이 cursor / selection 정상 | C4 |
| S11 | (선택) WikiLinkMark 도입 시 — WikiLinkMark + HashtagMark 동시 | C8 |

> 시나리오 S5 / S6 는 backend round-trip 이 필요하므로 frontend node:test 만으로 부족. backend pytest `test_snapshot_sync_marks_roundtrip.py` (신규) 와 함께 진행.

#### 2.1.5 Phase 1 NodeId 안정성 회귀 보존

본 FG 의 schema 변경이 Phase 1 의 NodeId 안정성 회귀를 깨지 않음을 입증. 기존 회귀 테스트 (`test_node_id_stability.py` 또는 동등 — Pre-flight 에서 위치 확인) 가 그대로 녹색이어야 한다.

#### 2.1.6 Phase 3 annotations 회귀 보존

AnnotationsPanel 의 anchoring 4 시나리오 (FG 3-3 종결보고서 §3 의 시나리오 A~D) 가 본 FG 변경 후에도 녹색 유지.

### 2.2 제외 (이월)

- **AnnotationMark 코드 신설** — task5-2 범위 (본 FG 는 schema spec 만 ADR 에 명시)
- **멘션 typeahead 자동완성 UI** — task5-3 범위
- **DocumentDetailPage 사이드바 재구조화** — task5-4 범위
- **WikiLinkMark 구현** — 운영자 결정에 따라 별 task 또는 본 Phase 흡수 (§2.1.0 결과)
- **mark / 데이터 마이그레이션** — 본 ADR 은 schema 결정만. 기존 저장 doc 의 mark 변환은 별 항목 (현재 저장 형태 분석 후 마이그레이션 필요 시 별 task)

### 2.3 하드코딩 금지 재확인

- mark name / data attribute key 는 ADR §d 표가 단일 정본. 다른 모듈에서 string literal 인라인 금지 — 공통 상수 모듈 (`frontend/src/features/editor/tiptap/markNames.ts` 신설) 로 단일화.
- backend `tag_rules.py` 가 HTML 파싱 시 `data-tag` 키를 사용한다면 동일 상수 (또는 backend 측 별 정본 명시). cross-stack 동기화 의무.

---

## 3. 선행 조건

- Phase 5 개발계획서 §1.2 R-A1~R-A4 가 운영자 승인 완료 (P1)
- Phase 4 1라운드 게이트 통과 권장 (강제 아님 — Phase 5 §0)
- 코드베이스 현 상태 깨끗 (`git status` clean)
- `Mark_inventory_실측.md` (§2.1.0) 1차 작성 후 운영자 검토 — ADR 작성 진입 전 게이트
- `docs/함수도서관/frontend.md` 의 TipTap extension 섹션 위치 확인 (ADR 결정 후 갱신 의무)

---

## 4. 구현 단계

### Step 1 — Mark inventory 실측 (§2.1.0)

1. 현 구현 mark / 미구현 mark / TipTap editor 진입점 전수 읽기
2. `Mark_inventory_실측.md` 작성
3. 운영자 1차 검토 — WikiLinkMark 처리 결정 (3 옵션 중 1) 합의
4. 합의 결과를 ADR §a 에 반영 준비

### Step 2 — Mark 통합 ADR 작성

1. `Mark_통합_ADR.md` §a~§f 초안 작성
2. ADR §c 직렬화 정본 (i) vs (ii) 결정 — 회귀 비용 / 결정성 보강 효과 비교 후 운영자 합의
3. ADR §e inclusive 정책 결정 (AnnotationMark `inclusive`)
4. **별 reviewer 합의** — Phase 5 개발계획서 §1.5 헌법 제27조 — Claude 단독 결정 금지. Codex 또는 다른 reviewer 검토 의무 (extended Handoff)
5. 운영자 최종 승인 (P1)

### Step 3 — mark schema 코드 적용 (§2.1.2)

1. `HashtagMark.ts` 의 schema spec 명시화
2. `markNames.ts` 신규 — mark name / data attribute / CSS class 단일 정본
3. (ADR (ii) 채택 시) `markOrder.ts` 신규 — 정규화 함수 + 단위 테스트
4. tsc 0 error 유지

### Step 4 — round-trip 회귀 테스트 (§2.1.4)

1. `markRoundtrip.test.ts` 작성 — S1~S10 (S11 은 WikiLinkMark 결정 시)
2. backend `test_snapshot_sync_marks_roundtrip.py` (S5 / S6) 추가
3. node:test 전체 베이스라인 유지 + 신규 시나리오 ≥ 10 녹색
4. pytest 베이스라인 유지 + 신규 ≥ 2 녹색

### Step 5 — Phase 1 / Phase 3 회귀 보존 (§2.1.5 / §2.1.6)

1. NodeId 안정성 회귀 위치 확인 + 본 FG 변경 후 녹색 확인
2. AnnotationsPanel anchoring 4 시나리오 회귀 녹색 확인

### Step 6 — 함수도서관 동기화

1. `docs/함수도서관/frontend.md` 의 TipTap extension 섹션에 본 FG 변경 반영 (markNames / markOrder / 우선순위)
2. ADR §d 표 일부를 frontend.md 에 발췌 (정본은 ADR — frontend.md 는 인덱스)

### Step 7 — 검수 보고서 / UI 디자인 리뷰

- `FG5-1_검수보고서.md` — R-A1 / R-A2 준수 확인 + S1~S10 회귀 결과 첨부
- UI 디자인 리뷰 ≥ 1회 — mark 시각 우선순위 (`#tag` 가 주석 안에 있을 때 시각적으로 어떻게 보이는지) 미리 합의
- (보안 보고서는 본 FG 범위 외 — task5-2 / 5-3 / 5-4 가 작성)

---

## 5. API 계약 변경 요약

| 메서드 | 경로 | 변경 |
|-------|------|------|
| POST | `/api/v1/documents/{id}/draft` (save_draft) | 응답 round-trip 동등성 회귀 (변경 없음, 회귀만) |

API 표면 변경 없음. 본 FG 는 schema 결정 + 회귀 + ADR 만.

---

## 6. 데이터 모델 주의사항

- **DB 스키마 변경 없음.** 본 FG 는 ProseMirror JSON 직렬화 규칙만 결정.
- 단, ADR §c 가 (ii) 채택 시 기존 저장된 doc 들의 mark 순서가 정규화 형식과 다를 수 있다. 회귀 시 발견되면 별 마이그레이션 task 신설 필요 — Pre-flight (§2.1.0) 에서 사전 분석.

---

## 7. 성공 기준

- [ ] `Mark_inventory_실측.md` 제출 + 운영자 1차 검토
- [ ] WikiLinkMark 처리 결정 (3 옵션 중 1) 합의
- [ ] `Mark_통합_ADR.md` §a~§f 작성 + 별 reviewer 합의 + `@최철균` 승인
- [ ] mark schema 코드 적용 (§2.1.2) + tsc 0 error
- [ ] (ADR (ii) 시) `markOrder.ts` + 단위 테스트 ≥ 6 녹색
- [ ] round-trip 회귀 ≥ 10 시나리오 녹색 (frontend node:test + backend pytest)
- [ ] Phase 1 NodeId 안정성 회귀 녹색
- [ ] Phase 3 annotations 회귀 (anchoring 4 시나리오) 녹색
- [ ] `docs/함수도서관/frontend.md` 갱신
- [ ] `FG5-1_검수보고서.md` + UI 디자인 리뷰 1회 통과
- [ ] pytest / node:test / tsc 베이스라인 유지

---

## 8. 리스크

| # | 리스크 | 대응 |
|---|-------|-----|
| R-01 | WikiLinkMark 결정 지연 → Phase 5 진입 게이트 막힘 | 보수적 default — (c) "schema 만 예약" 으로 진행하고 ADR 본문에 후속 task 명시. 운영자 합의 부재 시에도 task5-2~5-4 진행 가능 |
| R-02 | ADR (ii) 정규화 채택 후 기존 doc round-trip 깨짐 | Pre-flight 에서 prod-like 샘플 N건 round-trip 검증. 깨짐 발견 시 (i) 로 회귀하거나 마이그레이션 task 신설 |
| R-03 | TipTap 의 mark order 가 비결정적 → S5 회귀 flaky | 회귀 테스트는 정규화 후 비교 또는 mark set 비교 (순서 무시). 결정 (i) 일 경우에도 회귀 비교 함수 명시 |
| R-04 | backend `tag_rules.py` 가 AnnotationMark 안의 `#tag` 를 누락 | S6 회귀 추가. 누락 발견 시 별 task (Phase 5 본 FG 또는 task5-2 안) 수정 |
| R-05 | mark name / data attribute 네임스페이스가 CSS / backend 와 불일치 | `markNames.ts` 단일 정본 + cross-stack grep 검증. `data-tag` 사용 위치 전수 확인 |
| R-06 | ADR 합의 지연 (Phase 5 §7 R-05) | 본 FG 의 합의 게이트는 `@최철균` + 별 reviewer 1인. 2명 합의로 한정 |

---

## 9. 참조

- `docs/개발문서/S3/phase5/Phase 5 개발계획서.md` §1.2 (R-A1~R-A4), §2.1 (FG 5-1)
- `docs/개발문서/S3/phase1/산출물/FG1-3_주석앵커계약.md` (NodeId 기반 앵커 키)
- `docs/개발문서/S3/phase2/작업지시서/task2-3.md` §4 Step 6 (WikiLinkMark 미구현 사실 baseline)
- `docs/개발문서/S3/phase3/산출물/FG3-3_종결보고서.md` (annotations / anchoring 4 시나리오)
- `frontend/src/features/editor/tiptap/extensions/NodeId.ts`
- `frontend/src/features/editor/tiptap/extensions/HashtagMark.ts`
- `frontend/src/features/editor/tiptap/DocumentTipTapEditor.tsx`
- `backend/app/services/snapshot_sync_service.py`, `backend/app/services/tag_rules.py`
- `CONSTITUTION.md` 제5·11·12·15·27·48조

---

*작성: 2026-05-10 | FG 5-1 — Mark 통합 ADR + 직렬화 round-trip 회귀 (Phase 5 진입 게이트)*
