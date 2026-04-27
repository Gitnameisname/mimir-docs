# S3 Phase 5 — TipTap Mark 통합 + UX 본격화

**작성일**: 2026-04-27
**Phase 상태**: 대기 (Phase 3 종결 + Phase 4 1라운드 게이트 통과 권장)
**선행 조건**: Phase 3 종결 (annotations 기본 모델 / REST), Phase 4 의 `read_annotations` envelope 표준 합의 (FG 4-1)
**후행 조건**: Phase 6 진입 또는 S3 2라운드 종결
**Handoff Level**: `extended` — DocumentDetailPage 재구조화 + ProseMirror 다중 mark 통합 + 멘션 자동완성 (외부 API 표면 추가)
**Approver**: `@최철균` (P1 — Meta-1, 제45조)

---

## 0. Phase 4 와의 관계

본 Phase 는 Phase 4 (MCP/Chatbot 강화) 와 **독립적**으로 진행 가능하다. ProseMirror 통합은 별 도메인이라 충돌 없음.

다만 다음은 Phase 4 의 결과를 입력으로 받는다:
- **`read_annotations` 응답 envelope** — Phase 4 FG 4-1 이 `content_role`/`instruction_authority`/`trust_level`/`detected_risks` 를 부착하면, 본 Phase 의 `AnnotationsPanel` 이 `detected_risks` 를 시각화 가능 (선택, FG 5-2 결정)
- **`mimir://` URI 스키마** — Phase 4 FG 4-1 결정. WikiLinkMark 의 클릭 라우팅이 본 스키마와 정합되어야 함

Phase 4 1라운드 게이트가 통과되지 않은 상태에서 본 Phase 진입 가능하나, 위 두 항목은 Phase 4 종결 후 후속 patch 로 보강.

---

## 1. Phase 개요

### 1.1 목적

S3 Phase 1~3 에서 도입한 4 종 mark (NodeId / HashtagMark / WikiLinkMark / AnnotationMark) 를 한 에디터에서 충돌 없이 운영하고, 인라인 주석의 핵심 UX (본문 내 mark 클릭 → 패널 활성화 / 멘션 typeahead) 를 본격 도입한다. 동시에 DocumentDetailPage 의 우측 사이드바를 재구조화해 Contributors / Annotations / 기타 메타 패널이 좁은 화면에서도 일관 동작하도록 만든다.

### 1.2 절대 규칙

S3 1라운드 / Phase 4 의 공통 규칙(저장 모델 단일성·뷰 ≠ 권한·node_id 안정성·에이전트 동등성·instruction_authority=none) 은 그대로 유지하면서, 본 Phase 한정 추가:

- **R-A1. Mark 직렬화 정본 단일** — 4 mark 의 ProseMirror 직렬화/역직렬화 결과가 `save_draft` round-trip 에서 정확히 일치해야 한다 (Phase 1 NodeId 안정성 위에 얹힘).
- **R-A2. Mark 우선순위 명시** — 한 텍스트 범위가 여러 mark 로 표시 가능 (예: `#tag` 가 annotation 안에 있을 때). 우선순위 / 결합 규칙은 ADR (FG 5-1) 로 명시.
- **R-A3. AnnotationMark 클릭이 본문 편집 차단 안 함** — mark 클릭 → 패널 highlight 만. 본문 텍스트는 클릭 위치에서 정상 편집 가능 (TipTap mark 기본 동작 보존).
- **R-A4. Typeahead API 가 ACL 누설 금지** — `@` 입력 시 사용자 검색 popup 이 viewer 의 organization scope 안 사용자만 반환. 다른 조직의 user.display_name 은 절대 노출 X.

### 1.3 선행 조건

- Phase 3 종결 (annotations / annotation_mentions / notifications 테이블 + AnnotationsPanel)
- Phase 1 NodeId 안정성 회귀 테스트 유지
- Phase 2 HashtagMark / WikiLinkMark 안정 (FG 2-2, FG 2-3)
- (권장) Phase 4 FG 4-1 의 envelope 표준 + `mimir://` URI 결정

### 1.4 기대 결과

- 4 mark 통합 ADR 문서 (mark 우선순위 / 결합 / 직렬화 / 충돌 해소)
- AnnotationMark 인라인 mark + Gutter (좌측 여백 카운트) 도입
- AnnotationMark 클릭 → AnnotationsPanel 자동 펼침 + 해당 주석 highlight
- 멘션 typeahead — `@` 입력 시 floating 자동완성 popup (사용자 검색 API 신설 또는 기존 재사용)
- DocumentDetailPage 우측 사이드바 본격 도입 (Contributors / Annotations / 기존 메타 통합)
- 한국어 username 정책 합의 + 멘션 정규식 확장 (선택)

### 1.5 헌법/원칙 정합성

| 원칙 | 적용 |
|------|-----|
| S2 ⑤ AI 1급 사용자 | mark 통합 ADR 의 우선순위가 에이전트가 만든 본문에도 일관 적용 |
| S2 ⑥ Scope 하드코딩 금지 | typeahead 가 viewer 의 scope 만 통과한 사용자만 반환 |
| 헌법 제5조 Agent-Facing Contract | 사용자 검색 API 신설 시 description / input·output schema / 권한 / 부작용 / 실패 코드 명시 |
| 헌법 제27조 No Self-Review | mark 통합 ADR 은 별 reviewer 필요 (ProseMirror 도메인 + ACL 영향) |

---

## 2. Feature Group (FG) 요약

본 Phase 는 **5개 FG (5-1 ~ 5-5)** 로 구성한다. 한국어 username 정책 (FG 5-5) 은 합의 비용이 큰 항목이므로 조건부.

### 2.1 1라운드 (즉시 착수 대상)

| FG | 제목 | 핵심 | 산출물 |
|----|------|------|--------|
| **5-1** | Mark 통합 ADR + 직렬화 회귀 | 4 mark 우선순위 / 결합 규칙 / `save_draft` round-trip 회귀 테스트 | task5-1 + ADR + 회귀 테스트 결과 |
| **5-2** | AnnotationMark + Gutter 구현 | TipTap inline mark + 좌측 gutter (노드별 주석 카운트 점) + Panel 클릭 anchor 통합 | task5-2 + 검수·보안 보고서 |
| **5-3** | 멘션 typeahead 자동완성 | `@` 입력 → floating popup (debounce 150ms) + 키보드 네비 + 사용자 검색 API (`GET /api/v1/users?q=` — 신설 또는 재사용) | task5-3 + 검수·보안 보고서 |
| **5-4** | DocumentDetailPage 우측 사이드바 본격 도입 | 현 collapsible 카드 (Contributors / Annotations) → side panel + 탭 + 좁은 화면 collapsible fallback | task5-4 + 검수 보고서 + UI 디자인 리뷰 ≥ 2회 |

### 2.2 조건부 (게이트 통과 후 사용자 합의)

| FG | 제목 | 진행 조건 |
|----|------|---------|
| **5-5** | 한국어 username 정책 + 멘션 정규식 확장 | 사용자 이름 정책 합의 (운영자 + 사용자) 후 진행. 합의 부재 시 영문 username 만 유지 |

> ⚠️ FG 5-5 는 본 Phase 1라운드 게이트의 일부가 **아니다**.

---

## 3. 데이터 / 스키마 모델 (개요)

본 Phase 는 영구 스키마 변경 **없음**. 대부분의 변경은 frontend (TipTap extension + 컴포넌트) + backend 의 사용자 검색 API 추가 (FG 5-3, 기존 users 테이블 read-only 사용).

### 3.1 변경 없음

- annotations / annotation_mentions / notifications 스키마 그대로
- documents / versions / nodes / scope_profiles 그대로

### 3.2 신규 API (Backend)

- `GET /api/v1/users?q=<prefix>&limit=20` — 사용자 검색 (typeahead 용). viewer 의 organization scope 안 사용자만 반환. ACL 검증 후 display_name 만 노출 (raw user_id 는 admin 전용은 별 라운드).

### 3.3 신규 TipTap Extension (Frontend)

- `frontend/src/features/editor/tiptap/extensions/AnnotationMark.ts` — inline mark + `data-annotation-id` 속성
- `frontend/src/features/documents/AnnotationGutter.tsx` — 좌측 여백 마커
- `frontend/src/features/editor/tiptap/extensions/MentionSuggestion.ts` — `@` typeahead suggestion

---

## 4. 산출물 규약

`Season3_개발계획.md §4.2` 규약 + Phase 5 한정:

| 산출물 | FG 5-1 | FG 5-2 | FG 5-3 | FG 5-4 | FG 5-5 |
|--------|--------|--------|--------|--------|--------|
| 작업지시서 (`task5-N.md`) | ✅ | ✅ | ✅ | ✅ | (조건부) |
| ADR | ✅ (mark 통합) | — | — | — | — |
| 검수보고서 (R-A1~A4 준수 확인) | ✅ | ✅ | ✅ | ✅ | (조건부) |
| 보안취약점검사보고서 | — | ✅ | ✅ | ✅ | — |
| UI 디자인 리뷰 | ≥ 1회 | ≥ 1회 | ≥ 1회 | ≥ 2회 | — |

---

## 5. 게이트 / 완료 기준

### 5.1 1라운드 완료 기준 (FG 5-1 ~ 5-4)

- [ ] **R-A1 (Mark 직렬화)** — 4 mark 가 동시에 적용된 본문이 `save_draft` round-trip 후 ProseMirror JSON 상 정확히 일치 (회귀 테스트 ≥ 10 시나리오)
- [ ] **R-A2 (Mark 우선순위)** — ADR 에 명시된 우선순위 규칙이 코드에 반영됨 (예: AnnotationMark 가 HashtagMark 와 같은 범위에 있을 때 두 mark 다 적용 가능)
- [ ] **R-A3 (편집 차단 없음)** — AnnotationMark 클릭이 텍스트 selection / cursor 위치에 영향 없음 (TipTap default mark 동작 검증)
- [ ] **R-A4 (Typeahead ACL)** — 다른 조직의 user 가 typeahead 결과에 노출 안 됨 (회귀 테스트)
- [ ] AnnotationMark 클릭 → AnnotationsPanel 자동 펼침 + 해당 주석 highlight + 스크롤
- [ ] Gutter 가 노드별 주석 카운트 정확히 표시
- [ ] DocumentDetailPage 우측 사이드바 4 viewport (desktop / narrow / tablet / mobile) UI 리뷰 통과
- [ ] Phase 1 NodeId 안정성 회귀 녹색
- [ ] Phase 3 annotations 회귀 녹색 (anchoring 4 시나리오)

### 5.2 회귀 게이트

- [ ] Phase 0~4 모든 회귀 녹색
- [ ] pytest 베이스라인 유지
- [ ] node:test 베이스라인 유지
- [ ] tsc 0 error

### 5.3 1라운드 종결 정의

위 5.1 + 5.2 모두 통과 시 **S3 Phase 5 1라운드 공식 종결**. FG 5-5 진행 여부는 사용자 합의로 별도 결정.

---

## 6. 범위 밖 / 이월

### 6.1 본 Phase 1라운드 명시적 제외

- **Span-level annotation** (한 노드 안에서 일부 텍스트만 마킹) — 본 Phase 는 노드 전체 또는 span_start/end 기반 (이미 schema 지원). UI 의 drag-to-select 는 별 라운드
- **Annotation 의 풍부한 본문 (rich text)** — 본 Phase 는 plain text content. markdown / mention link 만 인식
- **답글 깊이 2+ (스레드 nesting)** — 본 Phase 는 1단계 답글 유지 (Phase 3 결정)
- **Notifications 의 grouping / digest** — 별 라운드
- **외부 SaaS 커넥터** — Phase 4 와 동일하게 S3 2라운드 또는 S4 이월

### 6.2 보류 결정 (FG 진행 중 결정)

- AnnotationMark 의 색상 / 스타일 → FG 5-2 디자인 리뷰
- Typeahead 의 사용자 검색 API 가 신설 vs 기존 admin users API 재사용 → FG 5-3 Pre-flight
- 우측 사이드바의 collapsible 임계 viewport (현재 1024px) → FG 5-4 디자인 리뷰

---

## 7. 리스크

| # | 리스크 | 대응 |
|---|-------|-----|
| R-01 | Mark 통합 시 직렬화 충돌 (예: HashtagMark 안 AnnotationMark 가 round-trip 후 깨짐) | FG 5-1 회귀 테스트 ≥ 10 시나리오 + ProseMirror schema 명시 |
| R-02 | AnnotationMark 클릭이 cursor 위치 변경 → 사용자 편집 흐름 깨짐 | TipTap default 동작 보존 검증 + 키보드 단축키 (Esc 로 panel 만 close) |
| R-03 | Typeahead 가 다른 조직 user 검색 → ACL 누설 | 사용자 검색 API 가 viewer 의 organization scope 강제 + 회귀 테스트 |
| R-04 | 우측 사이드바 도입이 좁은 화면에서 본문을 가림 | 모바일 fallback (collapsible 또는 bottom-sheet) + 4 viewport UI 리뷰 의무 |
| R-05 | Mark 통합 ADR 합의 지연 | Phase 5 의 FG 5-1 가 ADR 작성 + 사용자 / 다른 reviewer 합의 게이트 — 1라운드 진입 전 게이트 |
| R-06 | DocumentDetailPage 재구조화로 기존 위젯 회귀 | 기존 위젯 (DocumentAssignControls / TagChipsEditor / ContributorsPanel / AnnotationsPanel / VectorizationPanel) 모두 사이드바에 마운트 + 회귀 테스트 |
| R-07 | 한국어 username 정책 합의 부재 | FG 5-5 조건부 — 합의 없으면 본 라운드 진행 안 함 |

---

## 8. 협업 / 라우팅

`AGENT_MODE.md` §3.3 (extended):
- Design: Claude
- Implementation: Claude (또는 Codex — frontend 비중 큼)
- Review: Codex
- Human Approval: @최철균 (mark 통합 ADR + 우측 사이드바 도입)

---

## 9. 참조

- `docs/개발문서/S3/phase4/Phase 4 개발계획서.md` (envelope 표준 + `mimir://` URI)
- `docs/개발문서/S3/phase3/` (annotations / AnnotationsPanel 기반)
- `docs/개발문서/S3/phase1/` (NodeId 안정성)
- `docs/개발문서/S3/phase2/` (HashtagMark / WikiLinkMark)
- `frontend/src/features/editor/tiptap/extensions/` (TipTap 통합 지점)

---

## 10. 변경 이력

| 일자 | 변경 | 작성자 |
|------|------|-------|
| 2026-04-27 | 초안 — Phase 4 (MCP) 신설 후 재정렬. 기존 "Phase 4 (TipTap mark)" 안을 Phase 5 로 이동 | Claude |

---

*작성: 2026-04-27 | TipTap Mark 통합 + UX 본격화 — S3 Phase 5 킥오프*
