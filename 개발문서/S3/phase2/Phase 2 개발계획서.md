# Phase 2 — 그룹화 풀세트 + 뷰 전환 + Vault Import

**작성일**: 2026-04-22
**Phase 상태**: 대기 (Phase 1 완료 선행)
**선행 조건**: Phase 1 게이트 통과
**후행 조건**: Phase 3

---

## 1. Phase 개요

### 1.1 목적

사용자가 **자기 맥락으로 문서를 묶어 보고 탐색**할 수 있도록 한다. 옵시디언 수준의 그룹화(컬렉션·태그·백링크·그래프·계층 폴더)를 도입하고, 리스트/트리/카드/그래프 4종 레이아웃 전환과 Saved View를 제공한다. 선택적으로 옵시디언 vault zip을 일회성 업로드하여 Mimir로 가져올 수 있는 Vault Import(A안)를 Phase 2의 마지막 FG로 포함한다.

### 1.2 절대 규칙 (반복 강조)

- **폴더·컬렉션·태그·그래프·Saved View는 순수 뷰(view) 레이어다**
- **ACL은 Scope Profile 단독 결정**. 폴더 이동이 권한을 바꾸지 않는다. 컬렉션 추가가 접근 범위를 넓히지 않는다.
- 이 원칙은 S1 ① · S2 ⑥ 위배의 핵심 위험 지점 — 모든 Phase 2 FG의 검수보고서에 **"뷰 ≠ 권한 준수 확인"** 섹션 의무

### 1.3 선행 조건

- Phase 1 완료 (단일 ProseMirror 저장 + 에디터 토글)
- 그래프 뷰 라이브러리 폐쇄망 OSS 선정 (sigma.js / cytoscape.js 등 로컬 번들)
- 태그 추출 규약 (`#tag` 인라인 vs frontmatter) 사전 결정

### 1.4 기대 결과

- 사용자가 컬렉션/폴더/태그/백링크/그래프로 문서를 묶어 봄
- 리스트/트리/카드/그래프 4종 뷰 전환
- Saved View로 자주 쓰는 필터+정렬+레이아웃 조합을 저장·공유
- 옵시디언 vault zip 업로드로 기존 자산을 Mimir에 1회 import

---

## 2. Feature Group (FG) 요약

### FG 2-1 — 수동 컬렉션 + 계층 폴더

- 사용자가 만들고 관리하는 컬렉션(임의 문서 집합)
- 계층 폴더 트리 구조
- 문서 하나가 여러 컬렉션에 속할 수 있음 (N:M)
- 문서는 하나의 폴더 경로에 속함 (N:1, Optional)
- **뷰만** — ACL 무영향

### FG 2-2 — 태그 동적 그룹

- `#tag` 추출: 본문 인라인 또는 frontmatter (단일 규약 확정 후 진행)
- 태그 기반 동적 그룹: "이 태그를 가진 문서들"
- 태그 정규화 (대소문자/공백)
- 태그 자동 완성

### FG 2-3 — 백링크 `[[문서명]]`

- 본문 인라인 참조 `[[문서명]]` 파싱 → 양방향 그래프 에지 생성
- 문서 상세 페이지에 "이 문서를 참조하는 문서" 섹션
- 이름 충돌 시 disambiguation (제목 + 경로 표시)

### FG 2-4 — 그래프 뷰 + 레이아웃 전환

- 4종 레이아웃: 리스트 / 트리 / 카드 / 그래프
- 토글 UI (키보드 단축키 포함)
- 그래프 뷰: 문서를 노드, 백링크를 엣지로. 선택적으로 태그/컬렉션도 노드로
- 폐쇄망 라이브러리 (로컬 번들)
- 대규모(수천 노드) 렌더링 성능 기준 마련

### FG 2-5 — Saved View

- 필터(document_type / scope / tag / collection / folder / 기간) + 정렬 + 레이아웃을 Saved View로 저장
- 사용자별 보관, URL 공유 가능
- URL 공유 시 뷰어의 Scope Profile ACL이 여전히 적용 (S1 ④ / S2 ⑥ 준수)

### FG 2-6 — Vault Import (A안, 일회성)

- 옵시디언 vault의 zip 업로드
- markdown + YAML frontmatter + `[[링크]]` + `#태그` + 폴더 구조를 **ProseMirror 트리**로 변환
- 업로드 시 **Scope Profile 선택 필수** · 기본값 `private`
- 변환 후에는 Mimir 내부 편집으로만 작업 (양방향 동기화 없음)
- 보안: zip bomb 방어, 파일 수/크기 상한, 개인정보(PII) 스캔 리포트 첨부
- AI 품질 평가 보고서 대상 아님 (import는 AI 기능 아님; 단, import된 문서의 RAG 재벡터화는 기존 경로로)

---

## 3. 데이터 모델 (개요, Phase 1 완료 후 확정)

신규 테이블 후보:
- `collections` (id, owner_id, name, ...)
- `collection_documents` (collection_id, document_id) — N:M
- `folders` (id, parent_id, name, path, ...) — 폴더는 owner 또는 Scope Profile 단위
- `document_folder` (document_id, folder_id) — N:1
- `tags` (id, name_normalized)
- `document_tags` (document_id, tag_id)
- `document_links` (from_doc_id, to_doc_id, node_id, text) — 백링크
- `saved_views` (id, owner_id, name, filter_json, sort_json, layout_key)
- `vault_imports` (id, owner_id, status, report_json)

주의: 모든 신규 테이블은 **필드 추가 수준**이고 기존 `documents` 스키마는 변경 최소. Alembic revision으로만 진행.

---

## 4. 산출물 (각 FG)

- 작업지시서 (`task2-1.md` ~ `task2-6.md`)
- 검수보고서 (각 FG + **"뷰 ≠ 권한 준수 확인"** 섹션 의무)
- 보안취약점검사보고서 (특히 FG 2-6은 zip bomb·PII 스캔 상세)
- UI 디자인 리뷰 ≥ 5회 로그 (UI 포함 FG)

---

## 5. 게이트 / 완료 기준

- [ ] 컬렉션/폴더/태그/백링크/그래프 기본 동작 확인
- [ ] 레이아웃 4종 + Saved View 동작
- [ ] Vault Import A안이 샘플 vault(≥ 20 파일)를 성공적으로 변환 + PII 스캔 리포트 생성
- [ ] **뷰 ≠ 권한** 회귀 테스트 녹색 (컬렉션 추가/폴더 이동이 ACL에 영향 없음)
- [ ] 각 FG의 검수·보안 보고서 제출
- [ ] UI 디자인 리뷰 ≥ 5회 로그 제출

---

## 6. 범위 밖 / 이월

- 외부 SaaS 커넥터(Confluence / Notion / SharePoint) — S3 2라운드 또는 S4
- 양방향 옵시디언 동기화 — A안(일회성) 확정
- 태그/폴더 자동 추천 (AI) — Phase 3 이후 검토
- 공유 Saved View(팀 공용) — Phase 3 이후 or S4

---

## 7. 리스크 (개요)

| 리스크 | 대처 방향 |
|--------|----------|
| 백링크 이름 충돌 스케일링 | 제목 + 경로 + ID fallback disambiguation |
| 그래프 뷰 대규모 성능 | 뷰포트 컬링, 단계별 렌더, 초기 node 수 상한 |
| Vault Import zip bomb | 압축 해제 전 크기 상한, 엔트리 수 상한, 경로 traversal 방어 |
| Vault Import PII 누출 | 변환 전 PII 스캔 → 업로더에게 경고 / 옵션으로 마스킹 |
| Saved View URL 공유 시 권한 노출 | URL 공유는 정의(필터 조건)만; 결과는 뷰어의 Scope Profile로 재필터 |

---

## 8. 참조

- `docs/개발문서/S3/Season3_개발계획.md` §5 Phase 2
- `/sessions/eloquent-practical-ramanujan/mnt/.auto-memory/project_s3_kickoff.md`

---

*작성: 2026-04-22 | Phase 1 완료 후 확정*
