# RAG 스모크 테스트 실행 가이드

| 항목 | 값 |
|------|----|
| 작성일 | 2026-04-22 |
| 대상 스크립트 | `scripts/rag_smoke/rag_smoke.py` + `sample_documents.py` + `golden_queries.py` |
| 적용 버전 | Mimir S2 (Phase 0~9 완료 상태) |
| 실행 주체 | 사용자 호스트 (docker compose 로 Mimir 기동 후) |
| 의존성 | Python ≥ 3.10 표준 라이브러리만 사용 (추가 pip 불필요) |

---

## 1. 목적

S2 종결 후 "선언적 완료 / 실질적 완료" 갭을 좁히기 위한 **1차 기능 스모크**.
단위 테스트 479건, 기능 테스트 34건이 통과한 상태에서 실제 인프라(PostgreSQL + Valkey + Milvus + SBERT + FastAPI) 연동으로 RAG 파이프라인이 `create → draft → submit_review → approve → publish → auto-vectorize → retrieve → answer` 경로를 오류 없이 흐르는지 확인한다.

**범위 밖**: 동시성/성능 부하, LLM 답변 품질(Faithfulness 실측)의 정량 평가. 답변 품질은 OpenAI/로컬 LLM 구성 시 추가 측정 가능하며, 현재 가이드는 `retrieval_hit_rate`, `citation_present_rate`, `precision_at_k` 중심이다.

---

## 2. 사전 준비

### 2.1 Mimir 기동

```bash
cd backend
cp .env.example .env  # 최초 1회 — 필수 시크릿이 세팅되어야 docker compose up이 뜸
# .env에서 반드시 설정:
#   POSTGRES_PASSWORD=<강한 비밀번호>
#   JWT_SECRET=<최소 32자 랜덤>
#   GRAFANA_ADMIN_PASSWORD=<강한 비밀번호>
#   EMBEDDING_DIM=768
#   EMBEDDING_MODEL=sbert
docker compose up -d
```

> docker compose는 `.env`의 필수 키가 비어 있으면 기동 실패한다 (S2 재검수 후 `${VAR:?...}` 강제 구문 적용). 기동 실패 시 에러 메시지대로 누락 키를 채운다.

### 2.2 DB 마이그레이션 & 시드 사용자

```bash
cd backend
alembic upgrade head
python -m scripts.seed_users   # admin@mimir.local / Admin!2345 (SUPER_ADMIN)
```

### 2.3 document_type 시드 확인

```bash
curl -s http://localhost:8000/api/v1/system/document-types | jq '.data'
```

`POLICY`, `MANUAL`, `REPORT` 세 타입이 **대문자**로 보여야 한다 (P7-2에서 대소문자 정규화 강제됨).

### 2.4 폐쇄망 모드 확인

`OPENAI_API_KEY` 미설정 상태로 기동해도 서비스는 정상 동작한다 — LLM은 placeholder/폴백으로 동작하고 SBERT + FTS 기반 retrieval 은 항상 동작한다. 스모크 스크립트는 이 조건을 기본 가정으로 작성되어 있다.

---

## 3. 실행

### 3.1 기본 실행 (풀 사이클)

```bash
cd /path/to/mimir
python scripts/rag_smoke/rag_smoke.py \
    --base-url http://localhost:8000 \
    --admin-email admin@mimir.local \
    --admin-password 'Admin!2345' \
    --wait-vectorize 30 \
    --rag-sleep 2.2 \
    --report-out docs/개발문서/S2_5/RAG_스모크_실측_$(date +%Y%m%d_%H%M).json
```

진행 순서:

1. `POST /api/v1/auth/login` → Bearer 토큰 획득
2. 16개 샘플 문서에 대해 `POST /documents` → `PUT /documents/{id}/draft` → `POST /workflow/submit-review|approve|publish`
3. `--wait-vectorize` 초 대기 (publish 가 `ThreadPoolExecutor`(max_workers=3)로 auto-vectorization 큐잉)
4. 21개 골든 쿼리에 대해 `POST /api/v1/rag/answer` 호출 (질의 간 `--rag-sleep` 초 sleep — `/rag/answer` 는 30/min rate limit)
5. 원시 결과 + 집계 지표를 `--report-out` 경로에 JSON 저장

### 3.2 드라이런 (로그인만 확인)

```bash
python scripts/rag_smoke/rag_smoke.py --dry-run
```

### 3.3 재실행 시 시딩 생략

`skip-seed` 는 이전 실행에서 `seeded` 가 남아 있을 때만 의미 있다. 단, 현재 스크립트는 이전 실행의 `document_id` 를 파일에 남기지 않으므로 `--skip-seed` 실행 시 `citation → sample_id` 매핑이 비어 `retrieval_hit_rate` 는 0 으로 집계된다. 사용 용도는 "질의 엔드포인트 동작 확인 + 응답 스키마 점검"에 한정된다.

### 3.4 벡터화 지연이 긴 환경

호스트 성능이 낮거나 임베딩 서비스가 막 기동한 직후에는 30초로 부족할 수 있다. 다음 조합을 권장:

```bash
--wait-vectorize 60
```

벡터화 완료 여부는 다음으로 수동 확인 가능:

```bash
curl -s http://localhost:8000/api/v1/vectorization/stats | jq '.data'
# queued / processing / succeeded / failed 수치
```

---

## 4. 결과 해석

### 4.1 집계 지표 정의

| key | 정의 | 경고 임계 |
|-----|------|-----------|
| `retrieval_hit_rate` | positive(답이 있는 질의) 중 `expected_doc_ids` 가 citation 에 1건 이상 등장한 비율 | **≥ 0.80** — 미만이면 retriever/인덱스 점검 |
| `citation_present_rate_positive` | positive 질의 중 citation 이 1건 이상 반환된 비율 | ≥ 0.90 (S2 AI 품질 목표와 동일) |
| `avg_precision_at_k` | citation 중 `expected_doc_ids` 에 속한 비율의 평균 | ≥ 0.40 (top-k=10 기준, 단일 expected) |
| `avg_keyword_coverage` | answer + citation snippets 합본에서 `expected_keywords` 매칭 평균 | ≥ 0.60 (LLM 없으면 0 가능) |
| `multihop_ok` | multihop 카테고리 중 동일 doc에서 citation ≥ N 조건 충족 건수 | ≥ 2 / 3 |
| `negative_total_met` | negative 카테고리 중 "모르겠다/없음" 응답 또는 citation 0 건수 | ≥ 2 / 3 |
| `errors` | HTTP/네트워크 에러 수 | **0 이어야 함** |
| `avg_latency_ms` | `/rag/answer` 응답 시간 평균 | 폐쇄망 placeholder LLM 기준 ≤ 2,000 ms |

### 4.2 카테고리별 기대 베이스라인

| 카테고리 | 샘플 수 | retrieval_hit 기대 | 메모 |
|---------|---------|-------------------|-----|
| simple | 8 | 8/8 | 단일 문서 keyword가 명확해 SBERT만으로도 히트해야 정상 |
| ambiguous | 4 | 3/4 이상 | 재랭커 + 하이브리드의 유효성 검증 구간. 미달 시 cross-encoder 가중치 점검 |
| typed | 3 | 3/3 | `document_type` 필터 적용이 안 되면 이 구간이 먼저 깨진다 |
| multihop | 3 | 3/3 hit, 2/3 multihop_ok | hit은 높아도 citation 수가 1로 수렴하면 multihop_ok 는 저조 |
| negative | 3 | hit 개념 없음, `negative_total_met ≥ 2` | LLM 미구성 시 answer가 비어 `citation==0` 조건으로 통과 가능 |

### 4.3 종료 코드

- `0` — 정상
- `2` — 로그인 실패 (토큰 없음/계정 미시드/비밀번호 불일치)
- `3` — PUBLISHED 문서가 0건 (워크플로 전이 전면 실패)
- `4` — 질의 단계에서 HTTP 에러 발생
- `5` — `retrieval_hit_rate < 0.60` (경고; 에러는 아니지만 조사 필요)

---

## 5. 트러블슈팅

| 증상 | 가능한 원인 | 대처 |
|------|-----------|------|
| 로그인 422 "identifier required" | body 필드명이 `email` | 본 스크립트는 `identifier` 사용 — 최신 구조에 맞음. 본인이 직접 curl 호출하는 경우만 이슈 |
| 문서 생성 422 `document_type` | document_types 테이블에 POLICY/MANUAL/REPORT 미시드 | `POST /api/v1/admin/document-types` 로 시드하거나 마이그레이션 다시 실행 |
| draft 저장 422 `content_snapshot` | `type: document` 루트 누락 | `sample_documents.build_content_snapshot()` 이 루트를 포함하므로 수정 금지 |
| workflow 403 | admin 계정의 role 매핑 문제 | `workflow_service._ROLE_MAP` 에서 `super_admin → ADMIN` 확인 |
| vectorization 0건 완료 | `.env` 의 `EMBEDDING_DIM` / `EMBEDDING_MODEL` 과 `document_chunks` 컬럼 차원 불일치 | BUG-04 재발 — config 와 DB 스키마 정렬 (768 vector) |
| 429 Too Many Requests | `/rag/answer` 30/min 초과 | `--rag-sleep` 을 2.5 이상으로 늘린다. 스크립트는 Retry-After 자동 대기 후 1회 재시도함 |
| retrieval_hit_rate 급락 (<0.5) | Milvus 미기동 또는 index 생성 안 됨 | `docker compose ps`, `curl /api/v1/system/health` 확인 |
| answer 가 비어 있고 citation 만 있음 | 폐쇄망 모드 (LLM 미구성) — 기대되는 동작 | citation 기반 지표만 해석 대상 |

---

## 6. 산출물

실행 후 `--report-out` 경로에 JSON 이 생성된다. 권장 보관 위치:

```
docs/개발문서/S2_5/RAG_스모크_실측_YYYYMMDD_HHMM.json
```

보고서 스키마:

```jsonc
{
  "run_at": "2026-04-22T10:00:00+00:00",
  "base_url": "http://localhost:8000",
  "summary": {
    "total": 21,
    "positive_total": 18,
    "negative_total": 3,
    "retrieval_hit_rate": 0.944,
    "citation_present_rate_positive": 1.0,
    "avg_precision_at_k": 0.412,
    "avg_keyword_coverage": 0.733,
    "avg_latency_ms": 1240.5,
    "errors": 0,
    "multihop_total": 3, "multihop_ok": 3,
    "negative_total_met": 3,
    "by_category": {
      "simple":    {"n": 8, "hit": 8, "citation_present": 8},
      "ambiguous": {"n": 4, "hit": 3, "citation_present": 4},
      "typed":     {"n": 3, "hit": 3, "citation_present": 3},
      "multihop":  {"n": 3, "hit": 3, "citation_present": 3},
      "negative":  {"n": 3, "hit": 0, "citation_present": 0}
    }
  },
  "seeded":  [{ "sample_id": "...", "document_id": "...", "version_id": "...", "status": "published", "errors": [] }],
  "results": [{ "id": "Q01", "category": "simple", "hit": true, "citations": [...], "answer": "...", "latency_ms": 980.1, ... }]
}
```

---

## 7. S2 산출물 규약 준수

- `UI/기능 개선 산출물은 docs/개발문서/S2_5/ 에 작성` (사용자 기 지정 규칙) — 본 문서 + 검수보고서 + 보안취약점검사보고서가 동일 경로에 존재한다.
- 본 스모크는 FG 단위의 신규 개발이 아니라 **기존 기능의 실 인프라 검증**이므로 산출물 3종 중 "작업지시서"는 본 가이드가 겸한다.
- AI 품질 평가(Faithfulness ≥ 0.80 등)는 LLM 연동이 포함된 별도 실행(OpenAI 키 세팅)에서 `evaluations` API 로 수행 — 본 스모크의 범위 밖.

---

*작성: 2026-04-22 | Mimir S2 종결 후 1차 RAG 스모크 준비*
