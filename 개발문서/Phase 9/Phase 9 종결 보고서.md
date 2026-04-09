# Phase 9 종결 보고서
## 변경 비교 및 이력 가시화 기능 구축

**종결일**: 2026-04-09  
**Phase 기간**: 2026-04-09 (단일 세션)  
**작업 범위**: 구현 → 검수 → 보안 취약점 검사 및 수정 → 종결

---

## 1. Phase 개요

Phase 9는 Mimir 플랫폼에 **버전 간 구조 단위 diff 추적 및 가시화 시스템**을 구축하는 단계다.  
단순한 텍스트 비교가 아닌, Node 트리 구조의 변경을 ID 기반으로 추적하여  
검토자가 승인/반려 결정 시 무엇이 어떻게 바뀌었는지 즉시 파악할 수 있는 인프라를 확립한다.

계획서 MVP 범위를 초과하여 **MOVED 노드 감지**, **단어 단위 LCS 인라인 diff**, **인메모리 캐싱**까지  
단일 세션에서 완료했다.

---

## 2. 완료 기준 충족 여부

| # | 완료 기준 | 상태 | 비고 |
|---|-----------|------|------|
| 1 | 두 버전 간 구조 단위 diff API가 동작한다 | ✅ | 4개 엔드포인트 구현 |
| 2 | diff 결과에 변경 유형이 포함된다 | ✅ | ADDED/DELETED/MODIFIED/MOVED/UNCHANGED |
| 3 | 변경 요약이 생성된다 | ✅ | 자연어 요약 + 섹션 칩 + 심각도 |
| 4 | User UI에서 두 버전을 선택하여 diff를 볼 수 있다 | ✅ | VersionComparePage, 기준 버전 드롭다운 |
| 5 | 워크플로 검토 화면에서 변경 요약이 표시된다 | ✅ | DiffSummaryBanner + DiffViewerAuto 통합 |
| 6 | 버전 이력 타임라인이 동작한다 | ✅ | VersionsPage 타임라인 강화 |
| 7 | diff 요청 시 권한 검증이 적용된다 | ✅ | `require_authenticated=True` (보안 강화 포함) |

**7개 완료 기준 전체 충족.**

### MVP 이후 계획 항목 중 이번 Phase에 추가 구현된 항목

| 항목 | 비고 |
|------|------|
| MOVED 노드 감지 및 시각화 | REORDER / HIERARCHY_CHANGE 구분 포함 |
| 텍스트 인라인 diff (단어 수준) | LCS 기반, 한국어 어절 처리 |
| diff 결과 캐싱 | 버전 불변성 활용 인메모리 캐시 |

---

## 3. 구현 산출물

### 3.1 백엔드 신규 파일

| 파일 | 설명 |
|------|------|
| `backend/app/schemas/diff.py` | DiffResult, NodeDiff, InlineDiffToken, DiffSummary 등 Pydantic 스키마 |
| `backend/app/services/diff_service.py` | TextDiffer(LCS), NodeDiffer(ID 기반), DiffSummaryGenerator, DiffService, 인메모리 캐시 |
| `backend/app/api/v1/diff.py` | diff API 라우터 (4개 엔드포인트) |

### 3.2 백엔드 수정 파일

| 파일 | 변경 내용 |
|------|-----------|
| `backend/app/api/v1/router.py` | `/documents/{document_id}/versions` prefix로 diff router 등록 |

### 3.3 프론트엔드 신규 파일

| 파일 | 설명 |
|------|------|
| `frontend/src/types/diff.ts` | TypeScript diff 타입 전체 정의 |
| `frontend/src/lib/api/diff.ts` | diffApi 클라이언트 (4개 엔드포인트) |
| `frontend/src/features/diff/DiffSummaryBanner.tsx` | 변경 요약 배너 (섹션 칩, 접기/펼치기) |
| `frontend/src/features/diff/InlineDiffRenderer.tsx` | 인라인 diff 토큰 렌더러 (XSS 안전) |
| `frontend/src/features/diff/DiffViewer.tsx` | Unified diff 뷰어 (NodeDiffCard, 토글) |
| `frontend/src/features/diff/DiffViewerAuto.tsx` | 직전 버전 자동 비교 래퍼 |
| `frontend/src/features/diff/VersionComparePage.tsx` | 버전 비교 페이지 기능 컴포넌트 |
| `frontend/src/app/documents/[id]/versions/[vid]/compare/page.tsx` | 버전 비교 라우트 |

### 3.4 프론트엔드 수정 파일

| 파일 | 변경 내용 |
|------|-----------|
| `frontend/src/types/index.ts` | diff 타입 re-export 추가 |
| `frontend/src/lib/api/index.ts` | diffApi export 추가 |
| `frontend/src/lib/api/client.ts` | 개발 환경 X-Actor-Id/X-Actor-Role 헤더 자동 첨부 |
| `frontend/src/features/versions/VersionsPage.tsx` | 타임라인 강화 (변경 요약 칩, 비교 버튼) |
| `frontend/src/features/workflow/WorkflowActionModal.tsx` | approve/reject 시 DiffSummaryBanner + DiffViewerAuto 통합 |

### 3.5 API 엔드포인트

| 엔드포인트 | 설명 |
|-----------|------|
| `GET /api/v1/documents/{doc_id}/versions/{v_id}/diff` | 직전 버전 대비 전체 diff |
| `GET /api/v1/documents/{doc_id}/versions/{v_id}/diff/summary` | 직전 버전 대비 변경 요약 (경량) |
| `GET /api/v1/documents/{doc_id}/versions/{v1_id}/diff/{v2_id}` | 두 버전 간 전체 diff |
| `GET /api/v1/documents/{doc_id}/versions/{v1_id}/diff/{v2_id}/summary` | 두 버전 간 변경 요약 (경량) |

공통 쿼리 파라미터: `inline_diff`, `include_unchanged`, `max_inline_length`

### 3.6 문서 산출물

| 파일 | 설명 |
|------|------|
| `doc/개발문서/Phase 9/Phase 9 검수 리포트.md` | 구현 검수 결과 (결함 6건 분류, 전체 수정) |
| `doc/개발문서/Phase 9/Phase 9 보안 취약점 검사 리포트.md` | 보안 취약점 2건 수정, 설계 이슈 1건 강화 |

---

## 4. 검수 결과 요약

**검수일**: 2026-04-09 | 검수 방식: 정적 코드 분석

| 등급 | ID | 결함 내용 | 처리 |
|------|----|-----------|------|
| 🔴 Critical | BUG-2 | `invalidate_cache_for_document` 캐시 키 불일치 — 무효화 불가 | ✅ 수정 |
| 🟠 Major | BUG-5 | `DiffViewer` queryKey에 `showUnchanged` 누락 — 토글 무반응 | ✅ 수정 |
| 🟠 Major | BUG-6 | DELETED 노드 섹션 식별 불가 — 삭제 섹션 누락 | ✅ 수정 |
| 🟡 Minor | BUG-1 | `_lcs_diff` 이중 DP 계산 — 성능 낭비 | ✅ 수정 |
| 🟡 Minor | BUG-3 | `compute_summary_with_previous` 캐시 확인 순서 역전 | ✅ 수정 |
| 🔵 Low | BUG-4 | `DiffViewer.tsx` 미사용 import (`DiffSummaryBanner`) | ✅ 수정 |

**6건 전체 수정 완료.**

---

## 5. 보안 취약점 검사 결과 요약

**검사일**: 2026-04-09 | OWASP Top 10 기준

| 등급 | ID | 취약점 | 처리 |
|------|----|--------|------|
| 🟠 High | VULN-1 | LCS DP 테이블 메모리/CPU DoS — max_inline_length=50,000 시 최대 5GB RAM | ✅ 수정 |
| 🔵 Low | VULN-2 | f-string SQL 안티패턴 — 향후 SQL Injection 실수 유도 가능 | ✅ 수정 |
| ⚠ Design | DESIGN-1 | 미인증 사용자 diff 전면 허용 | ✅ 강화 |

### 취약점별 수정 상세

**VULN-1 (LCS DoS):** `MAX_LCS_CELLS = 1,000,000` 상수 도입, `_lcs_diff` 함수 시작부에 셀 수 초과 시 `ValueError` raise → 기존 `except Exception` 핸들러가 `skipped=True`로 처리.

**VULN-2 (f-string SQL):** `compute_diff_with_previous` fallback SQL의 `f"""..."""` → `"""..."""` 변경.

**DESIGN-1 (인증 강화):** diff API `require_authenticated=False` → `require_authenticated=True` 변경. 프론트엔드 `client.ts`에 개발 환경 인증 헤더 자동 첨부 로직 추가 (`useAuthzStore.getState()` → `X-Actor-Id`/`X-Actor-Role`).

**보안 양호 확인 항목:** XSS(InlineDiffRenderer React 자동 이스케이프), SQL Injection(파라미터 바인딩), ReDoS(`\S+|\s+` 단순 패턴), 캐시 포이즈닝(UUID 기반 캐시 키), 재귀 무한루프(MAX_PARENT_DEPTH=50), 노드 수 초과(MAX_NODES_SYNC=10,000), default-deny 권한.

---

## 6. 핵심 기술 결정 사항

| 결정 항목 | 선택 방향 | 근거 |
|-----------|-----------|------|
| 트리 diff 알고리즘 | ID 기반 O(n+m) 매칭 | 노드 ID 존재 → Myers O(n²) 불필요 |
| 텍스트 diff | LCS 단어 단위 | 한국어 어절 가독성 확보 |
| diff 저장 방식 | 온디맨드 계산 + 인메모리 캐시 | 버전 불변성 → TTL 무제한 |
| 캐시 키 형식 | `diff:{doc_id}:{min_vid}:{max_vid}` | 방향 무관, 문서 단위 무효화 가능 |
| 인증 수준 | `require_authenticated=True` | diff = 변경 이력 포함, 일반 read보다 민감 |

---

## 7. MVP 이후 잔여 항목

| 항목 | 내용 | 권장 Phase |
|------|------|------------|
| Side-by-side 뷰 | 이전/이후 버전을 좌우 분할 표시 | Phase 10 이후 |
| Admin 변경 이력 조회 강화 | Admin UI에서 전체 문서 diff 이력 조회 | Phase 10 이후 |
| diff 비동기 사전 계산 | 대용량 문서 요청 전 미리 계산 | Phase 10 이후 (필요 시) |
| Valkey 기반 캐시 전환 | 인메모리 캐시 → 프로세스 공유 가능 캐시 | Phase 인프라 정비 시 |

---

## 8. 다음 Phase 연계 사항

### Phase 10 (벡터 검색 / 지식 그래프) 시작 시 참고

- **diff 캐시 키 구조 확립**: `diff:{doc_id}:{min_vid}:{max_vid}` — Valkey 전환 시 그대로 사용 가능
- **DiffSummaryBanner 범용성**: `baseVersionId` prop으로 임의 버전 쌍 비교 가능 — 검색 결과 화면 등에서도 재사용 가능
- **인증 헤더 연동 완비**: `client.ts`에 개발 헤더 자동 첨부 구현 — 이후 모든 API 엔드포인트의 인증 강화 시 기반 활용 가능
- **`require_authenticated=True` 전환 패턴**: diff 엔드포인트에 적용한 패턴을 다른 민감 엔드포인트에도 순차 적용 권고

---

## 9. 종결 판정

**Phase 9 완료.**

계획서 완료 기준 7개 전체 충족,  
MVP 이후 예정이었던 MOVED 감지, 인라인 diff, 캐싱까지 이번 Phase에 포함하여 구현.  
검수 결함 6건 전체 수정, 보안 취약점 2건 수정 + 인증 설계 이슈 강화 완료.

> Phase 10 진행 가능.
