# task0-1 — 실 DB 통합 테스트 CI 구축

**작성일**: 2026-04-22
**Phase / FG**: S3 Phase 0 / FG 0-1
**상태**: 착수 대기
**담당**: 백엔드
**예상 소요**: 2~3일

---

## 1. 작업 목적

S2 종결 시점 단위 테스트 479건이 통과한 상태에서 BUG-01(트랜잭션 오염), BUG-03(pgvector 연결 누수), BUG-04(차원 불일치) 등 **실 DB 연동이 있어야만 드러나는 버그**가 재검수 단계에서 발견되었다. Mock 기반 단위 테스트로는 이 유형을 근본적으로 잡을 수 없다.

본 작업은 **CI 파이프라인이 매 PR마다 실 PostgreSQL (pgvector 포함) + Valkey 컨테이너 위에서 통합 테스트를 실행하고, 실패 시 머지를 차단**하도록 만든다. Phase 1 이후 Mimir에 기능을 얹기 전에 이 게이트를 먼저 세운다.

---

## 2. 작업 범위

### 2.1 포함

- CI 워크플로 추가 또는 기존 워크플로 확장 (GitHub Actions 기준; 타 CI라면 동등)
- PostgreSQL 16 + pgvector 컨테이너 기동 (`services:` 또는 testcontainers)
- Valkey 7 컨테이너 기동
- 신규 `backend/tests/integration/` 디렉터리 생성 및 초기 시나리오 구현
- 초기 통합 시나리오 (아래 2.3)
- CI 아티팩트로 테스트 리포트 업로드
- 검수보고서 · 보안취약점검사보고서 제출

### 2.2 제외

- Milvus 컨테이너 (S3 Phase 0는 pgvector 경로 검증에 집중; Milvus는 FG 0-1의 후속 확장으로 잔존 가능)
- E2E UI 테스트 (본 FG는 백엔드 한정)
- 부하/성능 테스트
- 테스트 데이터 팩토리의 전면 리팩토링 (최소한만 추가)

### 2.3 초기 통합 시나리오 (최소 10건)

| ID | 시나리오 | 검증 포인트 |
|----|----------|-------------|
| IT-01 | 로그인 → 문서 생성 → draft 저장 → submit_review → approve → publish | 트랜잭션 경계, FK 정합 |
| IT-02 | publish → auto-vectorize 완료 대기 → `/rag/answer` 응답에 citation 포함 | ThreadPoolExecutor 큐잉, pgvector insert |
| IT-03 | 두 사용자 동시 approve 시도 (Scope 동일) | race condition 보호 (row lock 또는 UNIQUE 제약) |
| IT-04 | Scope Profile A 사용자 검색 시 Scope B 문서 노출 안 됨 | S2 ⑥ 원칙 실 DB 검증 |
| IT-05 | 폐쇄망 모드 (OPENAI_API_KEY 없음) — SBERT + FTS 경로로 answer 반환 | S2 ⑦ 원칙 실 DB 검증 |
| IT-06 | Draft 저장 중 하나의 트랜잭션이 롤백될 때 `documents` 테이블에 고아 레코드 없음 | BUG-01 재발 방지 |
| IT-07 | `embedding_dim` 설정과 `document_chunks.embedding` 차원 일치 확인 | BUG-04 재발 방지 (FG 0-2와 연계) |
| IT-08 | 감사 로그 `actor_type` 필드가 user/agent 구분으로 남음 | S2 ⑤ 원칙 |
| IT-09 | 버전 diff + rollback 후 content_snapshot 일관성 | S2 Phase 6 회귀 |
| IT-10 | RAG 429 rate limit 재시도 후 성공 | 클라이언트 관점 — `rag_smoke.py`의 `HttpClient._request` 호환성 |

추가 5~10건은 본 작업 진행 중 식별되는 "Mock으로 잡히지 않을 경로"를 우선 추가한다.

---

## 3. 선행 조건

- `backend/docker-compose.yml`에 PostgreSQL(pgvector) + Valkey 서비스 정의가 있음 (확인 필요)
- `backend/app/db/migrations/versions/` 최신 head가 머지 상태
- CI secret 또는 env에 `POSTGRES_PASSWORD` / `JWT_SECRET` 등 필수 값 주입 경로 확보 (CI용 강한 랜덤 기본값 OK)
- FG 0-2의 embedding_dim 검증 revision이 아직 없으므로, IT-07 시나리오는 **FG 0-2 완료 후 활성화**로 명시 (블록 플래그 `pytest.mark.skip(reason="pending FG 0-2")`)

---

## 4. 주요 작업 항목

### 4.1 기술 선택 (사전 결정 필요)

- **후보 A — GitHub Actions `services:` 블록**
  - 장점: 단순, CI YAML만으로 완결
  - 단점: 로컬에서 동일 조건 재현 시 별도 docker compose 필요
- **후보 B — testcontainers-python**
  - 장점: 로컬/CI 동일 하네스, Python 내에서 컨테이너 수명 관리
  - 단점: 러너에 docker가 있어야 하고, 테스트마다 컨테이너 재기동 시 속도 느려짐 → 세션 scope 픽스처로 재사용 필요

**권고**: **후보 B (testcontainers)** — 로컬 개발자가 `pytest backend/tests/integration/` 로 동일한 통합 테스트를 돌릴 수 있는 것이 큰 가치. CI에서는 세션 scope 픽스처로 컨테이너 1회 기동 후 재사용하여 10분 이내로 유지.

### 4.2 pytest 픽스처 설계

- `backend/tests/integration/conftest.py`
  - `postgres_container` (session scope): testcontainers `PostgresContainer` + pgvector 이미지 (`pgvector/pgvector:pg16` 또는 `ankane/pgvector`)
  - `valkey_container` (session scope): `valkey/valkey:7`
  - `db_session` (function scope): 트랜잭션 내에서 실행 후 롤백 (테스트 격리)
  - `async_client` (function scope): httpx `AsyncClient` + FastAPI app
  - `alembic_upgrade` (session scope): `alembic upgrade head` 자동 실행
  - `seed_admin` (session scope): admin 계정 시드
  - `scope_profile_private` (session scope): 기본 Scope Profile 시드
- 임베딩 모델은 CI에서 SBERT 로컬 사용 (모델 캐시를 CI artifact cache로 보관)

### 4.3 CI 워크플로

- `.github/workflows/backend-integration.yml` 신설 (또는 기존 backend-test.yml 확장)
- 트리거: PR + push to `main`/`develop`
- 잡 구성:
  1. checkout
  2. Python 3.11 setup
  3. 의존성 설치 (`pip install -e .[test]`)
  4. SBERT 모델 캐시 복원/저장
  5. `pytest backend/tests/unit/` (빠른 단위 테스트)
  6. `pytest backend/tests/integration/ -v --maxfail=3` (testcontainers 기반)
  7. 커버리지 리포트 업로드 (FG 0-3과 연계; 본 FG에서는 업로드 경로만 마련)

### 4.4 로컬 실행 가이드 추가

- `docs/개발문서/S3/phase0/산출물/FG0-1_로컬실행가이드.md`
  - `pytest backend/tests/integration/` 명령
  - 로컬에 docker가 없을 때 대체 경로 (docker compose 기반 수동 기동)
  - 디버깅: `--pdb`, `--lf` (last failed)

### 4.5 보안 체크

- CI secret 평문 노출 금지 (workflow YAML 내 하드코딩 금지)
- 통합 테스트에 실제 운영 데이터 반입 금지 (sample_documents 등 공개 가능 데이터만)
- testcontainers의 이미지 pin (태그 고정, digest 권장)

### 4.6 산출물

- `.github/workflows/backend-integration.yml`
- `backend/tests/integration/` 디렉터리 + 초기 10+건
- `backend/tests/integration/conftest.py`
- `docs/개발문서/S3/phase0/산출물/FG0-1_검수보고서.md`
- `docs/개발문서/S3/phase0/산출물/FG0-1_보안취약점검사보고서.md`
- `docs/개발문서/S3/phase0/산출물/FG0-1_로컬실행가이드.md`

---

## 5. 완료 기준

- [ ] 신규 통합 테스트 ≥ 10건 녹색 (IT-07 제외 9건 + FG 0-2 완료 후 10건)
- [ ] CI 워크플로 `main` 3회 연속 녹색
- [ ] 로컬 `pytest backend/tests/integration/` 단일 명령 실행 성공
- [ ] CI 소요 ≤ 15분
- [ ] 검수보고서 · 보안보고서 제출

---

## 6. 리스크와 대처

| 리스크 | 대처 |
|--------|------|
| testcontainers 컨테이너 기동 지연으로 CI 타임아웃 | session scope 픽스처 + 이미지 캐시 |
| SBERT 모델 다운로드 지연 | CI 캐시 + 사전 준비 스크립트 |
| 테스트 간 격리 실패 (트랜잭션이 새는 경우) | `db_session` 픽스처를 savepoint 기반으로, 실패 시 격리 회복 |
| pgvector 이미지 보안 취약점 | 이미지 태그 pin + Trivy 스캔 옵션 (보안보고서에 스캔 결과 첨부) |

---

## 7. 참조

- `docs/개발문서/S3/Season3_개발계획.md` §2.3, §5
- `docs/개발문서/S3/phase0/Phase 0 개발계획서.md` §2 FG 0-1
- `docs/개발문서/S2/S2 개발계획서 검수 패치안.md` — BUG-01~05 상세
- `CLAUDE.md` S1 ⑤ ⑥

---

*작성: 2026-04-22 | 착수 대기*
