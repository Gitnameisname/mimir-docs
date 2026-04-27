# Phase 0 — 기반 부채 정리 (S2 잔존 P0 해소)

**작성일**: 2026-04-22
**Phase 상태**: 착수 대기
**선행 조건**: 없음 (Phase 0는 S3 최초 착수 구간)
**후행 조건**: Phase 1~3 전체의 게이트

---cl

## 1. Phase 개요

### 1.1 Phase 명

Phase 0 — 기반 부채 정리

### 1.2 목적

S2 종결 시점에 드러난 P0 4건(실 DB CI 부재, 커버리지 35%, AI 품질 실측 부재, embedding_dim 검증 부재)을 해소하여 Phase 1 이후 기능 확장이 회귀 위험 없이 진행될 수 있는 토대를 만든다.

### 1.3 선행 조건

- S2 재검수 2026-04-18 보고서 수령 상태
- `scripts/rag_smoke/` 기반 스모크 스크립트 (S2-5 산출물) 존재
- Alembic 경로 정상 (최신 revision `head`가 머지된 상태)
- `docker-compose` `.env` 필수 키가 채워진 상태 (S2 재검수 후 강제 구문 적용됨)

### 1.4 기대 결과

- CI가 매 PR마다 실 PostgreSQL/Valkey/pgvector 컨테이너 위에서 통합 테스트를 돌리고, 녹색으로 머지됨
- 커버리지 리포트가 CI artifact로 저장되며 백엔드 서비스/리포지토리 기준 ≥ 80%
- `rag_smoke.py` 확장본으로 골든셋 50+건에 대해 Faithfulness ≥ 0.80, Citation-present ≥ 0.90 실측 완료
- Alembic revision이 `embedding_dim` 설정과 `document_chunks` vector 차원 불일치를 탐지하고 실패 종료함
- Phase 0 각 FG마다 검수·보안 보고서 + (AI 품질 FG는) AI 품질 평가 보고서 제출

### 1.5 Phase 역할

Phase 0는 **방어적 구간**이다. 기능을 추가하지 않고 "이미 있는 것이 실제로 동작함"을 증명한다. S2 종결의 "선언적 완료 / 실질적 완료" 갭을 Phase 0에서 폐쇄한다.

---

## 2. Feature Group (FG) 상세

### FG 0-1 — 실 DB 통합 테스트 CI

**목적**: 단위 테스트(Mock 기반)만으로는 탐지되지 않은 BUG-01(트랜잭션 오염) / BUG-03(pgvector 연결) / BUG-04(차원 불일치) 유형을 머지 전에 잡는다.

**주요 작업**:
1. CI 워크플로에 `services:` 블록 추가 (PostgreSQL 16 + pgvector, Valkey 7)
   - 또는 `testcontainers-python` 기반 테스트 하네스 선택
2. 기존 단위 테스트(479건)는 그대로 유지, **신규 통합 테스트 디렉터리** `backend/tests/integration/`에 주요 경로 시나리오 추가
3. 통합 테스트 대상 경로 (최소):
   - 문서 create → draft → submit_review → approve → publish → auto-vectorize
   - 문서 publish → RAG answer (SBERT + pgvector 경로)
   - Scope Profile 기반 검색 ACL (권한 없는 사용자의 쿼리가 차단됨)
   - 워크플로 전이 race condition (동시 approve 요청)
4. CI 러너 사양: Ubuntu 22.04 / Python 3.11 / docker 24+
5. CI 소요 시간 상한: 통합 테스트 ≤ 10분, 전체 파이프라인 ≤ 15분

**입력**:
- 기존 `backend/tests/` 구조
- `backend/docker-compose.yml` (서비스 정의 참고)

**출력**:
- `.github/workflows/backend-integration.yml` 또는 `pyproject.toml` + `Makefile` 타깃
- `backend/tests/integration/` 디렉터리 + 초기 시나리오 테스트
- 검수보고서: `docs/개발문서/S3/phase0/산출물/FG0-1_검수보고서.md`
- 보안취약점검사보고서: `docs/개발문서/S3/phase0/산출물/FG0-1_보안취약점검사보고서.md`

**완료 기준**:
- `main` 브랜치에서 CI 3회 연속 녹색
- 신규 통합 테스트 ≥ 10건
- 단위 + 통합 테스트 모두 녹색

---

### FG 0-2 — `embedding_dim` config ↔ DB 스키마 자동 검증

**목적**: BUG-04(`EMBEDDING_DIM=768` 설정과 `document_chunks.embedding VECTOR(1536)` 스키마 불일치)의 재발을 코드 레벨에서 차단.

**주요 작업**:
1. 신규 Alembic revision: 기동 시 또는 `upgrade head` 시 `app.core.config.EMBEDDING_DIM` 값과 `document_chunks.embedding` 컬럼 차원을 비교
2. 불일치 시 revision 실패 + 명확한 에러 메시지
3. 애플리케이션 기동 시 `/api/v1/system/health`의 세부 정보에도 포함 (healthcheck subchecks)
4. `pytest` 유닛 테스트:
   - 정합 케이스 통과
   - 불일치 케이스 Alembic revision 실패 검증 (pytest에서 monkeypatch로 config 변경)
5. 문서화: `docs/개발문서/S3/phase0/산출물/FG0-2_운영가이드.md`에 운영자가 차원 변경 시 수행해야 할 절차 명시

**입력**:
- `backend/app/core/config.py` (EMBEDDING_DIM)
- `backend/app/db/models/document_chunks.py` (embedding 컬럼 정의)

**출력**:
- Alembic revision 파일 (`backend/app/db/migrations/versions/s3_p0_embedding_dim_check.py`)
- Healthcheck 확장
- pytest 테스트 ≥ 4건
- 검수보고서: `docs/개발문서/S3/phase0/산출물/FG0-2_검수보고서.md`
- 보안취약점검사보고서: `docs/개발문서/S3/phase0/산출물/FG0-2_보안취약점검사보고서.md`

**완료 기준**:
- 정합 케이스: `alembic upgrade head` 성공
- 불일치 케이스: `alembic upgrade head` 실패 + 명확한 메시지
- FG 0-1 CI가 이 revision을 포함해도 녹색

---

### FG 0-3 — 테스트 커버리지 35.44% → 80% 복구

**목적**: S2 종결 시점 커버리지 35.44%는 유효 안전망이 아님. 서비스/리포지토리 중심으로 80%까지 복구하여 Phase 1 이후 변경의 회귀를 실제로 탐지 가능하게 한다.

**주요 작업**:
1. 현재 커버리지 베이스라인 측정 및 `coverage_by_module.md`로 모듈별 커버리지 기록
2. 우선순위 순서 (bottom-up):
   - **최우선**: `app/services/` (현재 약 30% 추정) — 비즈니스 로직 핵심
   - **차순위**: `app/repositories/` — 쿼리·트랜잭션 경계
   - **3순위**: `app/api/v1/` 라우터 계층의 핵심 분기 (전부는 불필요)
3. 미커버 분기의 신규 유닛 테스트 추가
4. 통합 테스트(FG 0-1 산출물)도 커버리지에 합산
5. CI artifact로 커버리지 HTML 리포트 업로드
6. `.coveragerc` 또는 `pyproject.toml`에 제외 경로 명시 (마이그레이션·init_data 등)

**입력**:
- 기존 `backend/tests/` 전체
- FG 0-1 통합 테스트 결과

**출력**:
- 커버리지 리포트 (HTML + JSON summary)
- 신규 유닛 테스트
- 검수보고서: `docs/개발문서/S3/phase0/산출물/FG0-3_검수보고서.md`
- 보안취약점검사보고서: `docs/개발문서/S3/phase0/산출물/FG0-3_보안취약점검사보고서.md`

**완료 기준**:
- `app/services/` 커버리지 ≥ 80%
- `app/repositories/` 커버리지 ≥ 80%
- 전체 백엔드 커버리지 ≥ 75% (라우터 제외 시 ≥ 80%)
- CI에서 커버리지가 thresholds 아래로 떨어지면 실패

---

### FG 0-4 — AI 품질 실측 (골든셋 50+)

**목적**: S2 목표인 Faithfulness ≥ 0.80, Citation-present ≥ 0.90을 **실측값**으로 확보.

**주요 작업**:
1. `scripts/rag_smoke/golden_queries.py`를 50건 이상으로 확장
   - 기존 21건 유지
   - 신규 카테고리 추가: `domain-shift`, `adversarial`, `long-context` (각 ≥ 5건)
   - 전체 최소 50건, 가급적 60건 (positive 45 + negative 15 내외)
2. 샘플 문서 `sample_documents.py` 확장 (현재 16건 → 25건 이상)
   - domain-shift: 신규 테마 2종 추가 (예: 해외사업 / 브랜딩)
   - adversarial: 긴 본문·저자 중복·시기 애매 문서
3. 판정 LLM 추상화 (폐쇄망 호환):
   - 판정 어댑터 인터페이스 (`rag_smoke/judge.py`) 추가
   - OpenAI 어댑터 + 로컬(Ollama / vLLM) 어댑터 2종
   - 환경변수 `RAG_JUDGE_MODEL` / `RAG_JUDGE_BASE_URL`로 스왑
   - 폐쇄망 환경에서는 로컬 LLM으로 판정 (설정 가이드 포함)
4. 지표 계산 로직 추가:
   - **Faithfulness**: 답변의 각 주장이 citation snippet에 근거하는지 판정 LLM로 0/1 채점, 평균
   - **Citation-present rate**: 기존 로직 유지
   - **Answer relevance**: (선택) 질의-답변 관련성
5. 실측 실행 + 리포트 제출
6. 결과 미달 시 원인 진단 섹션 (retriever / reranker / prompt / 판정 불일치 등) 작성

**입력**:
- 기존 `scripts/rag_smoke/` 전체
- `docs/개발문서/S2_5/RAG_스모크_실행가이드.md`

**출력**:
- 확장된 `sample_documents.py` / `golden_queries.py`
- `scripts/rag_smoke/judge.py` (판정 어댑터)
- 실측 JSON 리포트 `docs/개발문서/S3/phase0/산출물/FG0-4_실측_YYYYMMDD_HHMM.json`
- AI품질평가보고서: `docs/개발문서/S3/phase0/산출물/FG0-4_AI품질평가보고서.md`
- 검수보고서: `docs/개발문서/S3/phase0/산출물/FG0-4_검수보고서.md`
- 보안취약점검사보고서: `docs/개발문서/S3/phase0/산출물/FG0-4_보안취약점검사보고서.md`

**완료 기준**:
- 골든셋 ≥ 50건
- **실측** Faithfulness ≥ 0.80
- **실측** Citation-present ≥ 0.90
- 판정 어댑터가 폐쇄망/외부 양쪽에서 동작함을 보이는 테스트 ≥ 2건

---

### FG 0-5 — 문서별 벡터화 상태 가시성 + 재벡터화 버튼

**목적**: 현재 publish 시 자동 벡터화가 비동기로 큐잉되지만, 실패 시 주 요청은 성공으로 반환되고 실패는 로그로만 남아 **"publish는 성공인데 RAG는 못 찾는" 조용한 실패** 상태가 누적된다. 이를 **문서 상세 페이지에서 즉시 인지하고 재시도할 수 있게** 한다. S3 Phase 0의 기반 부채 정리 의도와 일치한다 (2026-04-23 실사례 — RAG가 문서를 찾지 못한 원인을 탐지/복구할 수단이 없었음).

**주요 작업**:

1. 상태 조회 API 신설
   - `GET /api/v1/vectorization/documents/{document_id}/status`
   - 응답 스키마 (예시):
     ```jsonc
     {
       "document_id": "...",
       "latest_published_version_id": "...",
       "indexed_version_id": "...",
       "status": "indexed" | "pending" | "in_progress" | "failed" | "stale" | "not_applicable",
       "chunk_count": 42,
       "last_vectorized_at": "2026-04-23T07:00:00+00:00",
       "last_error": null | "MilvusConnectionError: ...",
       "can_reindex": true,        // actor 권한 판정 결과
       "reindex_cooldown_sec": 0   // 남은 쿨다운(서버 rate limit 반영)
     }
     ```
   - 상태 판정 로직:
     - `not_applicable`: 발행된 버전이 없음
     - `pending`: 발행됐으나 청크 0건 · 실패 로그 없음
     - `in_progress`: 현재 스레드 풀에서 실행 중 (짧게만 관찰됨; best-effort)
     - `failed`: 최근 실패 로그 존재
     - `stale`: `latest_published_version_id != indexed_version_id`
     - `indexed`: 위 외 정상

2. 재벡터화 트리거 API 확장 (기존 `POST /api/v1/vectorization/documents/{document_id}` 재사용)
   - 권한 확장: 기존 Admin-only → **Admin + `documents.created_by == actor.user_id`**
   - 서버측 쿨다운: 문서당 10초 (동일 사용자 연속 클릭 방지). 쿨다운 위반 시 429 + `Retry-After`
   - `actor_type` 을 audit_logs 에 기록 (S2 ⑤ 원칙)

3. 프런트엔드: 문서 상세 페이지 **벡터화 패널**
   - 위치: 문서 상세 상단(헤더 우측) 또는 사이드 패널 (UI 리뷰로 결정)
   - 표시 요소:
     - 상태 뱃지 색: `indexed`=green, `stale`=amber, `pending`=gray, `in_progress`=blue(spin), `failed`=red, `not_applicable`=neutral
     - 마지막 성공 시각 (local tz + relative)
     - 청크 수
     - `failed` 시 에러 요약 1줄 (긴 스택은 details 토글)
   - **재벡터화 버튼**:
     - `can_reindex=true` 인 경우에만 활성화
     - 클릭 → 확인 모달 → API 호출
     - 실행 중에는 spinner + 버튼 비활성
     - 쿨다운 중: tooltip 으로 남은 초 표시
   - 상태 자동 갱신: 버튼 클릭 후 2초 폴링 (최대 60초). WebSocket 미도입.
   - 접근성: 버튼 aria-label / 상태 aria-live="polite"
   - UI 디자인 리뷰 **≥ 5회** (CLAUDE.md UI 규칙)

4. MCP Tool 표면 확장 (S2 ⑤ 일관성)
   - 에이전트가 문서의 벡터화 상태를 조회할 수 있도록 Tool 추가: `mimir.vectorization.status(document_id)`
   - 재벡터화 실행은 Tool 에 노출하지 않음 (운영 안전 — 사람 명시적 클릭만)

5. 회귀 테스트
   - 상태 판정 분기 유닛 테스트 ≥ 8건 (각 status 케이스)
   - 권한 테스트: admin / 작성자 / 제3자 각각 응답
   - 쿨다운 테스트: 연속 호출 시 429
   - FG 0-1 통합 시나리오 IT-02 확장: publish → status=pending → 완료 대기 → status=indexed

6. 폐쇄망 안전 (S2 ⑦)
   - Milvus · 임베딩 서비스 off 상태에서 상태 조회는 **정상 동작**해야 한다 (마지막 성공 시각만으로 구성 가능한 필드는 유지)
   - 재벡터화 버튼 클릭 시 서비스 down 이 감지되면 `status=failed` + `last_error` 에 의존 서비스 명시

**입력**:
- 기존 `app/api/v1/vectorization.py` (stats, reindex 엔드포인트 재사용 기반)
- 기존 `app/services/vectorization_service.py` (파이프라인)
- 기존 `workflow.py:_trigger_vectorization_async` (큐잉 경로 재사용)
- `documents.created_by`, `audit_logs`, `document_chunks`

**출력**:
- 백엔드: 상태 조회 라우트 + 권한 확장 + 쿨다운
- 프런트엔드: 벡터화 패널 컴포넌트
- MCP Tool: `mimir.vectorization.status`
- 작업지시서: `task0-5.md`
- 검수보고서: `docs/개발문서/S3/phase0/산출물/FG0-5_검수보고서.md`
- 보안취약점검사보고서: `docs/개발문서/S3/phase0/산출물/FG0-5_보안취약점검사보고서.md`
- UI 디자인 리뷰 로그: `docs/개발문서/S3/phase0/산출물/FG0-5_UI리뷰로그.md`

**완료 기준**:
- 모든 published 문서에 대해 상태 API가 정상 응답
- 권한 확장(Admin + 작성자) 동작 + 제3자 403
- 재벡터화 버튼 클릭 시 실제 파이프라인 트리거 확인
- 쿨다운 429 동작
- UI 디자인 리뷰 ≥ 5회 로그
- 회귀 테스트 ≥ 12건 녹색

---

## 3. Phase 0 산출물 규약 강화

S3 전체 산출물 규약 위에, Phase 0는 다음을 추가 의무로 한다:

1. **근거 데이터 첨부**: 각 FG의 검수보고서는 "표만"으로 끝내지 않고, 실측 JSON / 커버리지 HTML summary / CI 녹색 런 URL 중 해당되는 것을 첨부한다.
2. **`선언적 완료 vs 실질적 완료`**: 검수보고서 §결론에 "본 FG는 단위 테스트 수준 완료가 아니라, CI 녹색 / 실측 리포트 / 불일치 케이스 실패 증명 중 해당 기준을 충족하는가"를 명시한다.

---

## 4. 작업지시서 목록

| 파일 | FG | 제목 |
|------|-----|------|
| `task0-1.md` | 0-1 | 실 DB 통합 테스트 CI 구축 |
| `task0-2.md` | 0-2 | embedding_dim config ↔ DB 스키마 자동 검증 |
| `task0-3.md` | 0-3 | 테스트 커버리지 35% → 80% 복구 |
| `task0-4.md` | 0-4 | AI 품질 실측 — 골든셋 50+ |
| `task0-5.md` | 0-5 | 문서별 벡터화 상태 가시성 + 재벡터화 버튼 |

---

## 5. Phase 0 완료 기준 (게이트)

- [ ] FG 0-1 / 0-2 / 0-3 / 0-4 / 0-5 각 FG의 검수·보안 보고서 제출
- [ ] FG 0-4 AI 품질 평가 보고서 제출 (Faithfulness ≥ 0.80, Citation-present ≥ 0.90 실측 포함)
- [ ] FG 0-5 UI 디자인 리뷰 ≥ 5회 로그 제출
- [ ] CI 녹색 3회 연속
- [ ] 커버리지 ≥ 80% (서비스/리포지토리)
- [ ] Alembic embedding_dim 검증 revision 머지

Phase 0 게이트를 통과한 시점이 **S3 Phase 1 착수 가능 시점**이다.

---

## 6. 참조

- `docs/개발문서/S3/Season3_개발계획.md`
- `docs/개발문서/S2/S2 개발계획서 검수 패치안.md` (2026-04-18 재검수)
- `docs/개발문서/S2_5/RAG_스모크_실행가이드.md`
- `docs/개발문서/S2_5/RAG_스모크_검수보고서.md`
- `CLAUDE.md`

---

*작성: 2026-04-22 | Phase 0 착수 대기 상태*
