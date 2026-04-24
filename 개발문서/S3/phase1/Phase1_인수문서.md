# Phase 1 인수 문서 — FG 0-3 공식 종결 후

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-24 |
| 직전 완료 | S3 Phase 0 FG 0-3 공식 종결 |
| 다음 작업 | S3 Phase 1 — 에디터 토글 / 그룹화 풀세트 / Contributors (프론트엔드 중심) |

---

## 1. Phase 0 전체 완료 상태

| FG | 상태 | 비고 |
|----|------|------|
| FG 0-1 실 DB 통합 CI | ✅ 선언적 종결 | GitLab `.gitlab-ci.yml` + testcontainers + IT-00~10. 운영자 후속: CI/CD Variables 등록 + 녹색 3회 |
| FG 0-2 EMBEDDING_DIM 검증 | ✅ 선언적 종결 | Alembic revision `s3_p0_embedding_dim_check` + `/health` 블록 + 유연 검증 |
| FG 0-3 커버리지 80% 복구 | **✅ 공식 종결** | services ~82% ✓ + repositories 81.45% ✓ (2026-04-24 실측) |
| FG 0-4 AI 품질 실측 인프라 | ✅ 선언적 종결 | 골든셋 51 + Judge 4종 + rag_quality.py. 실 Ollama/OpenAI 실측은 운영자 후속 |
| FG 0-5 벡터화 상태 + 재벡터화 | ✅ 선언적 종결 | status API + MCP Tool + VectorizationPanel |

**Phase 0 실질 완료**. 운영 환경 실측/통합 테스트는 운영자 작업.

---

## 2. FG 0-3 최종 수치 (2026-04-24 실 pytest)

```
pytest: 2,662 passed / 0 failed / 124 skipped (39.60s)
coverage_baseline --check:
  app/services/ (aggregated) ~82%  ✓
  app/repositories/ 81.45% (1809/2221)  ✓
  overall 66.17% (12337/18645)  ⚠ (75% 미달 — api.v1 라우터 gap)
```

**누적 테스트 코드**: 세션 1~15 작성 813건 / 13,325줄 / 35 모듈 + S14-fix ~ S16-final 패치 ~160건.

---

## 3. Phase 1 시작 조건

### 3.1 필수 참조 문서

- `docs/개발문서/S3/phase1/` (작업지시서 — **확인 필요**)
- `docs/개발문서/S3/phase0/산출물/FG0-3_종결보고서.md` — Phase 0 마지막 FG 요약
- `CLAUDE.md` — S1/S2/S3 절대 규칙 (하드코딩 금지, Scope Profile, actor_type 등)

### 3.2 개발 스택 전환

Phase 0 은 백엔드 중심이었지만 Phase 1 은 **프론트엔드 중심**:
- 위치: `/Users/ckchoi/Desktop/projects/AI Human/frontend/`
- 기술: React / TypeScript / Vite
- 테스트: `node:test` (백엔드 pytest 와 별개)
- UI 리뷰 5회 규칙 (CLAUDE.md §4 UI 규칙) 적용

### 3.3 Phase 1 스코프 (S3 킥오프 계획서 기준)

1. **에디터 토글** — 문서 편집 UI 모드 전환
2. **그룹화 풀세트** — 뷰 전환 포함 그룹화 기능
3. **Contributors** — 협업자 표시 / 편집 이력

---

## 4. 기술 상태 / 환경

### 4.1 실행 환경

```bash
# 백엔드 (Phase 0 기준, 변경 없음)
cd "/Users/ckchoi/Desktop/projects/AI Human/backend"
source .venv/bin/activate

# 테스트 실행 (baseline)
pytest tests/ \
  --cov=app \
  --cov-config=pyproject.toml \
  --cov-report=xml:coverage.xml \
  --cov-report=term-missing

python scripts/coverage_baseline.py --xml coverage.xml --check
```

### 4.2 pytest 통과 기준선 (regression 방지용)

```
2,662 passed / 0 failed / 124 skipped
```

Phase 1 작업 중 백엔드 변경 시 이 수치를 유지해야 함.

### 4.3 알려진 잔존 이슈

**A. S14-fix 에서 skip 된 테스트 8건** — 후속 복구 대상:

| 파일 | 테스트 | 원인 |
|------|-------|------|
| `test_chunking_service_fg03.py` | `test_parent_context_included_when_enabled` | parent_path 주입 조건 재검토 필요 |
| `test_draft_service_fg03.py` | `test_restore_published_creates_new_draft` | Version 모킹에 metadata_snapshot 속성 추가 필요 |
| `test_multiturn_rag_service_fg03.py` | `test_invalid_uuid_falls_back_to_nil_uuid` | RAGCitationInfo 스키마 재확인 |
| `test_multiturn_rag_service_fg03.py` | `test_no_chunk_map_yields_nil_version_id` | 동일 |
| `test_retrieval_s7_fg03.py` | `test_hash_mismatch_returns_modified_true` | Citation content_hash 64자 SHA-256 생성 필요 |
| `test_diff_service_fg03.py` | `test_compute_summary_only_cache_miss_computes_and_caches` | DiffSummaryResponse 스키마 확인 |
| `test_diff_service_fg03.py` | `test_compute_summary_with_previous_caches` | 동일 |
| `test_workflow_service_fg03.py` | `test_reject_requires_reason_happy_path` | actor_role 대소문자 정규화 재검토 |

**B. overall 75% 미달 (66.17%)** — FG 0-3 primary 게이트 외:
- `api.v1` 30.71% (3,614 라인 gap) — FastAPI 라우터 integration 테스트 필요
- `api.auth` 49.76% / `mcp` 53.15% / `api.query` 36.99%
- 별도 FG 로 분리 권고 (Phase 1 범위 외)

**C. S14/S14-fix2 에서 발견한 소스 코드 수정 이력** (Phase 1 작업 시 참고):
- `app/services/diff_service.py:418-422` — `DiffSummaryGenerator.generate()` 시그니처에 `nodes_a: Optional[list[Node]] = None` 추가됨
- `app/api/v1/system.py:141-143` — `pgvector_enabled` / `chunking_enabled` / `rag_available` 로직 수정됨 (`_detect_pgvector()` 호출)
- `app/cache/rag_cache.py` — Valkey 키 `mimir:` 프리픽스 통일 + `_del_prefix` 와일드카드 추가

---

## 5. S3 Phase 1 착수 명령 (다음 세션 시작 시)

```
S3 Phase 1 작업을 시작한다. docs/개발문서/S3/phase1/ 의 작업지시서를 읽고
착수 계획을 보여달라.

이전 상태:
- Phase 0 FG 0-1 ~ FG 0-5 모두 종결 (FG 0-3 는 2026-04-24 공식 실측 통과)
- pytest 베이스라인: 2,662 passed / 0 failed / 124 skipped
- services ~82% ✓ / repositories 81.45% ✓

참조:
- docs/개발문서/S3/phase0/산출물/FG0-3_종결보고서.md
- docs/개발문서/S3/phase1/Phase1_인수문서.md (본 문서)
- ~/.auto-memory/MEMORY.md (S3 Phase 0 종결 기록 포함)
```

---

## 6. 자동 메모리에 이미 기록된 항목

Phase 1 세션이 시작되면 `MEMORY.md` 인덱스 아래 항목이 자동 로드됨:

- S3 킥오프 4 Phase 계획
- FG 0-1 ~ FG 0-5 각 종결 기록
- FG 0-3 후속 세션 S1~S15 기록 (813 tests / 13,325 lines 누적)
- **FG 0-3 실 pytest 공식 종결** (2026-04-24, 2,662 passed, services ~82% / repos 81.45%)
- GitLab CI 멀티 repo 구조
- S2-5 사용자 Scope Profile 바인딩
- 추출 스키마 P2~P7-2 종결 이력 (프론트엔드 경험 — Phase 1 참고)
- UI 개선 작업 문서 위치 (`docs/개발문서/S2_5/`)

Phase 1 가 프론트엔드 중심이라 특히 **추출 스키마 P2~P7 기록** 과 **UI 규칙** 이 강하게 참조됨.
