# Phase 2 인수 문서 — Phase 1 공식 종결 후

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-24 |
| 직전 완료 | S3 Phase 1 전체 (FG 1-1/1-2/1-3) 공식 종결 |
| 다음 작업 | S3 Phase 2 — 그룹화 풀세트 + 뷰 전환 + Vault Import (6 FG) |

---

## 1. Phase 1 공식 종결 실측

| 지표 | 값 |
|------|----|
| pytest | **2,718 passed / 0 failed / 126 skipped** |
| services (prefix 합산) | **81.09%** ≥ 80% ✓ |
| repositories | **81.68%** ≥ 80% ✓ |
| overall | 66.58% (WARN, `api.v1` 3,629 라인 gap — 별도 FG 이월) |
| frontend node:test | **165 passed / 31 suites / 0 failed** |
| Alembic heads | `s3_p1_content_snapshot_backfill`, `s3_p1_users_preferences` |
| coverage_baseline --gates services,repositories | exit 0 ✓ |

---

## 2. Phase 1 완료 시점 확보된 기반

Phase 2 작업은 아래 Phase 1 결과물 위에 쌓인다:

### 2.1 백엔드
- **`content_snapshot` 단일 정본** (ProseMirror doc JSONB) — 모든 쓰기 경로(save_draft / save_draft_nodes deprecated / agent_proposal) 통합
- **`snapshot_sync_service`** — ProseMirror ↔ nodes 양방향 변환 + `rebuild_nodes_from_snapshot` 파생 동기화
- **`schemas/versions.py::validate_content_snapshot`** — `type=="doc"` + `content: list` 강제
- **`users.preferences` JSONB** 컬럼 + `/account/preferences` GET/PATCH + `UserPreferences` schema (Literal + extra allow)
- **content_snapshot backfill Alembic** — 레거시 Draft 정제 (dry-run 지원)

### 2.2 프런트엔드
- **TipTap 3.22 에디터** (`DocumentTipTapEditor`) + **NodeId extension** (`keepOnSplit:false` + `appendTransaction` UUID 자동 부여)
- **블록 뷰 ↔ 일반 뷰 토글** (세그먼트 버튼 + `⌘/Ctrl+Alt+M` 단축키) + CSS class 분기 + flex-wrap 모바일 호환
- **`useUserPreferences`** 훅 (React Query + 400ms debounce + optimistic + rollback)
- **`ProseMirrorDoc`** 공통 타입 (`types/prosemirror.ts`)

### 2.3 DB / 인프라
- **public 스키마 전체 OWNER = `mimir_admin`** 로 통일. Alembic 실행은 `ALEMBIC_POSTGRES_USER=mimir_admin` 자격
- Alembic migration 유틸 스크립트: `backend/scripts/sql/switch_owner_to_mimir_admin.sql`

### 2.4 규정 / 정책
- **UI 디자인 리뷰 규정: 최소 1회 이상** (CLAUDE.md 프로젝트 §5.2 기준 Phase 1 에서 공식 완화. 전역 5회 규정보다 프로젝트 규정 우선)
- **coverage_baseline `--gates services,repositories` 플래그** — primary 게이트만 엄격, overall 은 WARN (Phase 1~2 기간 정책)

### 2.5 Phase 3 인계 계약
- `docs/개발문서/S3/phase1/산출물/FG1-3_주석앵커계약.md` — node_id 기반 인라인 주석 앵커 키 규약

---

## 3. Phase 2 스코프 — 6 FG

Phase 2 개발계획서 `docs/개발문서/S3/phase2/Phase 2 개발계획서.md` 참조. 핵심 FG:

| FG | 제목 | 주요 산출물 |
|----|------|-----------|
| 2-1 | 수동 컬렉션 + 계층 폴더 | `collections`, `collection_documents` (N:M), `folders`, `document_folder` 테이블. CRUD API. UI: 사이드바 트리 |
| 2-2 | 태그 동적 그룹 | `#tag` 파싱 규약 확정 → `tags`, `document_tags`. 자동 완성 UI. 태그 정규화 |
| 2-3 | 백링크 `[[문서명]]` | `document_links` 테이블 + 본문 파서. "이 문서를 참조하는 문서" 섹션. 이름 충돌 disambiguation |
| 2-4 | 그래프 뷰 + 레이아웃 전환 4종 | 리스트/트리/카드/그래프 토글. 그래프 라이브러리 폐쇄망 OSS (sigma.js / cytoscape.js 중 1개 선정) |
| 2-5 | Saved View | `saved_views` 테이블. 필터+정렬+레이아웃 저장. URL 공유 (뷰어 Scope Profile ACL 여전히 적용) |
| 2-6 | Vault Import (A안) | 옵시디언 zip → markdown + YAML frontmatter + `[[링크]]` + `#태그` + 폴더 → ProseMirror 트리 변환. zip bomb 방어, PII 스캔 |

### 3.1 절대 규칙 (S2 ⑥ 재확인)
- **폴더·컬렉션·태그·그래프·Saved View 는 순수 뷰 레이어**
- **ACL 은 Scope Profile 단독**. 폴더 이동이 권한을 바꾸지 않고, 컬렉션 추가가 접근 범위를 넓히지 않음
- 모든 Phase 2 FG 검수보고서에 "**뷰 ≠ 권한 준수 확인**" 섹션 의무

### 3.2 Phase 1 산출물 확장
- FG 2-3 백링크 파싱은 **Phase 1 `snapshot_sync_service` 확장** — doc 순회 시 text run 에서 `[[...]]` 토큰 감지
- FG 2-6 Vault Import 의 ProseMirror 변환도 동일 모듈에 `prosemirror_from_markdown` 추가
- Phase 2 전체에서 **새로운 쓰기 경로를 만들면 반드시 `rebuild_nodes_from_snapshot`** 호출 유지

---

## 4. 다음 세션 착수 프롬프트 (복사용)

```
S3 Phase 2 작업을 시작한다. docs/개발문서/S3/phase2/Phase 2 개발계획서.md 와
task 작업지시서가 아직 없으므로, Phase 1 의 Pre-flight → 작업지시서 → 구현
패턴을 그대로 따른다.

이전 상태:
- Phase 1 전체 공식 종결 (FG 1-1/1-2/1-3)
- pytest 2,718 passed / 0 failed / 126 skipped
- services 합산 81.09% / repositories 81.68% / overall 66.58% (WARN 별도 FG 이월)
- frontend node:test 165 녹색
- Alembic heads: s3_p1_content_snapshot_backfill + s3_p1_users_preferences
- DB 소유권 mimir_admin 통일

Phase 2 스코프:
- FG 2-1 수동 컬렉션 + 계층 폴더
- FG 2-2 태그 동적 그룹 (#tag 파싱)
- FG 2-3 백링크 [[문서명]]
- FG 2-4 그래프 뷰 + 레이아웃 4종 전환
- FG 2-5 Saved View
- FG 2-6 Vault Import (옵시디언 zip A안)

참조:
- docs/개발문서/S3/phase2/Phase 2 개발계획서.md
- docs/개발문서/S3/phase2/Phase2_인수문서.md (본 문서)
- docs/개발문서/S3/phase1/산출물/FG1-3_주석앵커계약.md (Phase 3 인계이지만 node_id 규약 참고)
- ~/.auto-memory/MEMORY.md (Phase 1 종결 기록 포함)

우선순위 제안:
1. 태그 추출 규약 (#tag 인라인 vs frontmatter) 사전 결정 (FG 2-2 블로커)
2. 그래프 뷰 OSS 라이브러리 선정 (FG 2-4 블로커). sigma.js vs cytoscape.js 폐쇄망 호환 검토
3. Phase 2 Pre-flight: 기존 document/scope 구조에 컬렉션/폴더/태그 테이블 추가 시 ACL 필터 복잡도 영향 실측
4. task2-1.md ~ task2-6.md 작업지시서 작성
5. FG 2-1 부터 순차 구현
```

---

## 5. 기술 상태 / 환경 (Phase 1 과 동일 유지)

### 5.1 백엔드
```bash
cd /Users/ckchoi/Desktop/projects/mimir/backend
source .venv/bin/activate
# pytest baseline: 2,718 passed / 0 failed / 126 skipped

# Alembic 은 OWNER 자격으로
ALEMBIC_POSTGRES_USER=mimir_admin \
ALEMBIC_POSTGRES_PASSWORD='<pw>' \
alembic upgrade head
```

### 5.2 프런트엔드
```bash
cd /Users/ckchoi/Desktop/projects/mimir/frontend
npm test   # baseline: tests 165 / suites 31 / pass 165
```

---

## 6. 알려진 제약 / 선제 확인 필요 항목

| 항목 | 상태 | 대응 |
|------|------|------|
| overall 커버리지 75% 미달 (api.v1 gap 3,629 라인) | WARN, exit 0 | Phase 2 전 또는 병행 — api.v1 integration 테스트 보완 FG 신설 권고 |
| Phase 0 skip `test_restore_published_creates_new_draft` (1건) | skip 유지 | FG 1-1 metadata_snapshot 속성 관련. 소규모 복구 별건 |
| `AdminUserDetailPage.tsx:279` TS 에러 | 기존 버그 | admin users 범위 별건 — Phase 2 진행 중 터치하지 말 것 |
| Integration smoke (test_fg13_editor_roundtrip) 실 testcontainers | 전수 pytest 에서 skip (3건) | testcontainers CI 세션 별도 |
| FG 2-4 그래프 라이브러리 선정 | 미결정 | Phase 2 Pre-flight 에서 sigma.js / cytoscape.js 중 1개 선정 + 폐쇄망 번들 확인 |
| FG 2-2 태그 규약 | 미결정 | Phase 2 Pre-flight 에서 `#tag` 인라인 / frontmatter / 둘 다 중 1개 확정 |

---

## 7. 자동 메모리에 이미 기록된 항목

Phase 2 세션이 시작되면 `MEMORY.md` 인덱스 아래 항목이 자동 로드됨:

- S3 Phase 1 공식 종결 (본 인수문서와 짝)
- FG 1-1/1-2/1-3 각 종결 기록 (구현 경로·결정사항·제약)
- Alembic OWNER 경로 (reference_backend_alembic)
- S2-5 사용자 Scope Profile 바인딩 (컬렉션/폴더 ACL 설계의 선행 패턴)
- 추출 스키마 P2~P7-2 종결 이력 (프런트엔드 대규모 개발 경험 — Phase 2 에도 유효한 node:test / UI 리뷰 패턴)

Phase 2 가 **데이터 모델 확장 + 프런트 UI 대공사** 조합이라 특히 **Phase 1 의 `snapshot_sync_service` 확장 패턴** 과 **추출 스키마 P3~P7 의 CRUD UI 설계** 를 강하게 참조하게 된다.

---

## 8. Phase 2 게이트 (완료 기준)

개발계획서 §6 기준:
- 6 FG 모두 검수·보안 보고서 제출
- 뷰 ≠ 권한 회귀 테스트 녹색 (Scope Profile 필터가 폴더/컬렉션/태그/Saved View 에 적용)
- 그래프 라이브러리 폐쇄망 번들 확인
- Vault Import: zip bomb 방어 / 파일 수·크기 상한 / PII 스캔 리포트
- pytest 베이스라인 유지 (2,718 이상 pass, skip 정책 유지)
- primary 커버리지 게이트 (services / repositories ≥ 80%) 유지

---

*작성: 2026-04-24 | Phase 1 공식 종결 직후 Phase 2 착수 준비 문서.*
