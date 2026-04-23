# Phase 1 — 에디터 모드 토글 (단일 ProseMirror 저장)

**작성일**: 2026-04-22
**Phase 상태**: 대기 (Phase 0 게이트 선행)
**선행 조건**: Phase 0 게이트 통과 (CI 녹색 3회 + 커버리지 ≥ 80% + AI 품질 실측 제출 + embedding_dim 검증 머지)
**후행 조건**: Phase 2

---

## 1. Phase 개요

### 1.1 목적

현재 단일 블록 뷰만 지원하는 편집기에 **일반 리치텍스트 뷰**를 추가한다. 사용자가 두 뷰를 자유롭게 토글하면서도 **저장 모델은 단일 ProseMirror 트리**로 유지되어 BUG-02(content_snapshot vs nodes 이중 쓰기 경로) 유형이 재발하지 않도록 한다.

### 1.2 원칙 강조

- **저장 단일성**: content_snapshot 하나만 정본. nodes 테이블은 **파생 동기화** 전용. 두 개의 쓰기 경로 금지.
- **node_id 안정성**: 뷰 토글 시 node_id가 변하지 않는다. 인라인 주석/citation 좌표가 node_id에 매달려 있기 때문.
- **UI 호환**: 데스크탑/웹 모두. UI 디자인 리뷰 ≥ 5회.

### 1.3 선행 조건

- Phase 0 완료
- 기존 편집기가 사용하는 ProseMirror 스키마가 정리되어 있음 (S2 종결 상태 기준)
- `content_snapshot` 루트가 `{type: "document", content: [...]}` 임을 확인 (S2-5 RAG 스모크의 계약과 동일)

### 1.4 기대 결과

- 사용자가 블록 뷰 ↔ 일반 뷰를 토글할 수 있음
- 저장 포맷은 단일 ProseMirror 트리
- node_id 안정성 회귀 테스트 녹색
- 인라인 주석/citation의 좌표가 토글 후에도 유효 (Phase 3에서 본격 검증, Phase 1에서 기반 마련)

---

## 2. Feature Group (FG) 요약

### FG 1-1 — 저장 모델 정규화

- content_snapshot 단일 소스 확정. nodes 테이블은 read-only 동기화 경로로 고정
- 기존 이중 쓰기 경로가 남아있다면 제거 + 회귀 테스트
- 마이그레이션: 기존 published 문서의 content_snapshot 정합성 검증 배치 (범위 한정)

### FG 1-2 — 일반 에디터 뷰 구현

- 리치텍스트 뷰 (블록 테두리 없음, 흐르는 편집)
- 블록 뷰와 동일한 ProseMirror 스키마 사용
- 노드 타입 매핑 (heading, paragraph, list, blockquote, code, callout 등)
- 뷰 전환 토글 UI
- 접근성: 키보드 단축키 / screen reader 호환

### FG 1-3 — 토글 안정성 검증

- node_id stable id 규약 테스트 (토글 전후 동일 node_id)
- citation 좌표 유효성 테스트
- 인라인 주석 anchoring 기반 마련 (실제 주석은 Phase 3에서)
- 저장 후 재로드 시 뷰 선택 복원 (사용자 선호 저장)

---

## 3. 산출물 (각 FG)

- 작업지시서 (`task1-1.md` ~ `task1-3.md`, 본 문서 확정 후 생성)
- 검수보고서
- 보안취약점검사보고서
- UI 디자인 리뷰 ≥ 5회 로그

---

## 4. 게이트 / 완료 기준

- [ ] 블록/일반 뷰 양방향 편집 실제 동작
- [ ] node_id 안정성 테스트 녹색
- [ ] 저장 모델 단일성 회귀 테스트 녹색
- [ ] UI 디자인 리뷰 ≥ 5회 로그 제출
- [ ] 각 FG의 검수·보안 보고서 제출

---

## 5. 설계 결정 (Phase 0 완료 시 재확인)

다음은 Phase 0 진행 중에 확정할 사항:
- ProseMirror 스키마 변경 최소화 여부 (현재 스키마로 일반 뷰 구현 가능한지)
- 뷰 전환 상태 저장 위치 (사용자 preference 테이블 vs 로컬 storage)
- 모바일 호환성 수준 (차단하지 않을 정도)

---

## 6. 범위 밖 / 이월

- 실시간 공동 편집 (CRDT / OT) — 별도 스프린트
- 마크다운 입력 모드 — Phase 2 Vault Import와 겹치지 않도록 별도 검토
- 슬래시 커맨드 팔레트 — 선택적; Phase 2 이후로

---

## 7. 참조

- `docs/개발문서/S3/Season3_개발계획.md` §5 Phase 1
- S2 BUG-02 이력

---

*작성: 2026-04-22 | Phase 0 게이트 통과 후 확정*
