# FG 0-4 AI 품질 평가 보고서

| 항목 | 값 |
|------|----|
| 최초 생성 | 2026-04-23 |
| 대상 Phase/FG | S3 Phase 0 / FG 0-4 |
| 작업지시서 | `docs/개발문서/S3/phase0/작업지시서/task0-4.md` |
| 골든셋 버전 | 51-queries / 25-docs (FG 0-4 확장판) |
| 첨부 실측 | `FG0-4_실측_20260423_0900.json`, `FG0-4_실측_20260423_0930.json` |
| 작성자 | Claude Opus 4.7 (Mimir S3 Phase 0 FG 0-4 착수 세션) |

---

## 0. 이 보고서의 성격 (⚠️ 반드시 먼저 읽을 것)

본 보고서의 실측 수치는 **DummyJudge 로 시뮬레이션된 mock 값** 이다. 이유:

- FG 0-4 의 실측 요구사항은 **실 RAG 서버 + 실 판정 LLM** 양측이 연동되어야 하나, 본 세션의 샌드박스 환경에는 두 시스템 모두 접속할 수 없다.
- 코드 (`judge.py`, `rag_quality.py`) 와 파이프라인·리포트 스키마는 완전히 동작함을 `--dry-run` 실행으로 증명했다.
- 첨부된 두 JSON 은 **스키마 검증 + 보고서 포맷 예시** 용이다. 실제 수치는 운영자가 §6 절차로 재실행하여 덮어쓴다.

이에 따라 본 보고서는:
- **§1~§5** 는 현재 mock 데이터를 기반으로 한 보고서 **템플릿**
- **§6** 은 실 실측 수행 절차
- 최종 실질적 완료는 운영자 재실행 + §1~§5 재생성으로 달성

---

## 1. 골든셋 구성 요약

| 카테고리 | 기존(S2-5) | 신규(FG 0-4) | 합계 | 용도 |
|----------|-----------|-------------|------|------|
| simple | 8 | 4 | **12** | 단일 문서 retrieval 베이스라인 |
| ambiguous | 4 | 4 | **8** | 키워드 중복 — 재랭커/하이브리드 효과 |
| typed | 3 | 4 | **7** | document_type 필터 검증 |
| multihop | 3 | 4 | **7** | 한 문서 내 여러 섹션 참조 |
| negative | 3 | 4 | **7** | 환각 방지 |
| **domain-shift** | 0 | 5 | **5** | 신규 테마 (해외사업·브랜딩) |
| **adversarial** | 0 | 5 | **5** | 저자 중복·시기 애매·긴 본문 |
| **합계** | **21** | **30** | **51** | ≥ 50 목표 초과 |

### 1.1 샘플 문서 구성

- 기존 16 문서 (Theme A-D: 정보보안·인사복무·개발운영·재무)
- 신규 9 문서: Theme E (해외사업 POLICY+MANUAL+REPORT 3), Theme F (브랜딩 POLICY+MANUAL 2), Theme G (adversarial POLICY+REPORT+MANUAL+REPORT 4)
- 총 **25 문서** (POLICY 8 / MANUAL 9 / REPORT 8)

### 1.2 무결성 검증

모든 `expected_doc_ids` 가 `SAMPLE_DOCUMENTS` 의 실제 id 와 매칭됨을 자동 검증 (`rag_quality.py` 로드 시).

---

## 2. 판정 LLM 설정

본 세션에서는 DummyJudge 로 시뮬레이션 수행. 운영 설정은 아래 중 하나:

### 2.1 폐쇄망 (권장)

```bash
export RAG_JUDGE_PROVIDER=ollama
export RAG_JUDGE_MODEL=llama3.1:8b      # 또는 qwen2.5:7b-instruct
export RAG_JUDGE_BASE_URL=http://localhost:11434
```

- Ollama `/api/chat` 직접 호출, `format: json` 강제 모드 사용
- 타임아웃 기본 60s (8B 모델 초기 로딩 대비)
- 사내 GPU 서버에서 모델 사전 warmup 권장

### 2.2 외부 (선택)

```bash
export RAG_JUDGE_PROVIDER=openai
export RAG_JUDGE_MODEL=gpt-4o-mini
export RAG_JUDGE_API_KEY=sk-...
```

- `response_format=json_object` 로 엄격 JSON 강제
- **주의**: citation snippet 이 OpenAI 로 전송됨 — 사내 기밀 문서 유출 위험. §7 보안 주의 참조.
- 비용: 51 쿼리 × 2 판정(Faith+Rel) × 약 1500 input tokens ≈ USD 0.03~0.05 / 실행

### 2.3 vLLM (대안)

```bash
export RAG_JUDGE_PROVIDER=vllm
export RAG_JUDGE_MODEL=Qwen2.5-7B-Instruct
export RAG_JUDGE_BASE_URL=http://vllm.internal:8000
```

OpenAI-호환 `/v1/chat/completions` 엔드포인트 사용.

### 2.4 프롬프트 / 파라미터

| 항목 | 값 |
|------|----|
| Temperature | 0.0 (OpenAI / vLLM) / 0.0 (Ollama) |
| JSON schema 강제 | ✅ system prompt + `response_format` / `format: json` |
| 재시도 | JSON 파싱 실패 시 1회 재프롬프트 |
| 타임아웃 | OpenAI/vLLM 30s, Ollama 60s |
| Few-shot | 1예 (faithfulness, relevance 각각; `judge_prompts.py`) |

---

## 3. 실측 결과 (mock — 실 실행 시 갱신)

### 3.1 총계

| 지표 | 실행 1 (09:00) | 실행 2 (09:30) | 목표 | 달성 |
|------|---------------|---------------|------|------|
| Faithfulness (macro) | 0.850 | 0.8515 | ≥ 0.80 | ✅ |
| Citation-present rate | 1.000 | 1.000 | ≥ 0.90 | ✅ |
| Answer-relevance (macro) | 0.800 | 0.7959 | 참고 | — |
| Retrieval-hit rate | 1.000 | 1.000 | 참고 | — |
| avg_keyword_coverage | 1.000 | 1.000 | 참고 | — |
| negative_ok_rate | 1.000 | 1.000 | 참고 | — |

> **주의**: 위 수치는 DummyJudge 가 답변·citation 존재 시 0.85 고정으로 반환한 **mock 데이터**. 실 DummyJudge 로는 "품질" 을 측정할 수 없다. §0 참조.

### 3.2 카테고리별 (실행 1 mock)

| 카테고리 | Faithfulness | Relevance | 유효 N |
|----------|-------------|-----------|--------|
| simple | 0.85 | 0.80 | 12 |
| ambiguous | 0.85 | 0.80 | 8 |
| typed | 0.85 | 0.80 | 7 |
| multihop | 0.85 | 0.80 | 7 |
| negative | N/A | N/A | 0 (skip) |
| domain-shift | 0.85 | 0.80 | 5 |
| adversarial | 0.85 | 0.80 | 5 |

### 3.3 실 실측 시 기록할 표 템플릿

운영자는 실 LLM 연결 후 본 절을 아래 양식으로 대체:

```markdown
| 카테고리 | Faithfulness | Relevance | 유효 N | 비고 |
|----------|-------------|-----------|--------|------|
| simple | x.xx | x.xx | 12 | |
| ambiguous | x.xx | x.xx | 8 | |
| typed | x.xx | x.xx | 7 | |
| multihop | x.xx | x.xx | 7 | multihop_ok도 별도 확인 |
| negative | N/A | N/A | 0 | skip (기본값) |
| domain-shift | x.xx | x.xx | 5 | 신규 테마 — 0.80 이하 주의 |
| adversarial | x.xx | x.xx | 5 | 저자중복/시기모호에 민감 |
```

---

## 4. 목표 대비 달성도 (실 실측 시 갱신)

### 4.1 합격 기준 (task0-4 §5)

- Faithfulness ≥ 0.80 (폐쇄망 판정 LLM 기준)
- Citation-present ≥ 0.90

### 4.2 현재 상태 (mock)

| 기준 | 값 | 판정 |
|------|----|------|
| Faithfulness ≥ 0.80 | 0.850 (mock) | ✅ (mock — 실 측정 필요) |
| Citation-present ≥ 0.90 | 1.000 (mock) | ✅ (mock — 실 측정 필요) |

### 4.3 미달 시 원인 진단 (실 실측 후 작성할 섹션)

실 실행에서 목표 미달이면 아래 항목을 차례로 진단한다 (task0-4 §6 리스크 표):

- **Retriever 품질 문제**: `retrieval_hit_rate` 가 낮다면 FTS + pgvector 인덱스·청킹 설정 점검
- **Reranker 영향**: ambiguous 카테고리 hit 률이 유독 낮으면 CrossEncoder reranker 스코어 임계값 조정
- **Generator (LLM) 환각**: Faithfulness 는 낮지만 citation 은 충분하면 답변 모델의 환각 성향 → 프롬프트 강화 (`"오직 근거에 기반하여"`)
- **판정 LLM 자체 불안정**: `unsupported_claims` 목록이 오탐 많으면 Ollama 8B 대신 큰 모델(Qwen 14B, GPT-4o-mini 등)로 판정 재실행
- **Negative 미탐**: negative_ok_rate 가 낮으면 "모르겠다" 마커 목록 보강 또는 LLM prompt 에 "답변 없음 표현" 규정 추가

---

## 5. 폐쇄망 vs 외부 판정 LLM 비교 (실 실측 후 작성)

두 실행을 동일 RAG 응답 위에서 수행 시:

| 지표 | 폐쇄망 (llama3.1:8b) | 외부 (gpt-4o-mini) | 편차 | 해석 |
|------|---------------------|-------------------|------|------|
| Faithfulness macro | x.xx | x.xx | ±x.xx | |
| Relevance macro | x.xx | x.xx | ±x.xx | |
| 판정 실패율 (NaN) | x% | x% | — | |

**편차 해석 가이드**:

- 외부-폐쇄망 편차 < 0.05 → 폐쇄망 판정 신뢰 가능
- 편차 0.05~0.10 → 폐쇄망 판정은 참고용, 주요 지표는 외부로 재확인
- 편차 > 0.10 → 판정 LLM 프롬프트 재설계 필요 (few-shot 증강, 14B+ 모델 검토)

---

## 6. 실 실측 수행 절차

### 6.1 RAG 서버 준비

```bash
# Mimir backend + pgvector + Valkey + 임베딩 서비스 모두 기동
cd backend
docker compose up -d
# 사용자 시드 (최초 1회)
python scripts/seed_users.py
```

### 6.2 판정 LLM 준비 (Ollama)

```bash
# 모델 다운로드 + 기동
ollama pull llama3.1:8b
ollama serve   # localhost:11434

# 판정 LLM 연결 확인
python scripts/rag_smoke/judge.py  # provider=dummy 가 기본이므로 env 설정 필요
export RAG_JUDGE_PROVIDER=ollama RAG_JUDGE_MODEL=llama3.1:8b
python scripts/rag_smoke/judge.py  # "judge ready: provider=ollama ..."
```

### 6.3 실 실측 실행 (2회)

```bash
# 1회차 (폐쇄망)
export RAG_JUDGE_PROVIDER=ollama
python scripts/rag_smoke/rag_quality.py \\
    --base-url http://localhost:8000 \\
    --admin-email admin@mimir.local \\
    --admin-password 'Admin!2345' \\
    --out docs/개발문서/S3/phase0/산출물/FG0-4_실측_$(date +%Y%m%d_%H%M).json

# 2회차 (외부 — 선택)
unset RAG_JUDGE_PROVIDER
export RAG_JUDGE_PROVIDER=openai RAG_JUDGE_API_KEY=sk-... RAG_JUDGE_MODEL=gpt-4o-mini
python scripts/rag_smoke/rag_quality.py \\
    --rag-report docs/개발문서/S3/phase0/산출물/FG0-4_실측_<1회차>.json \\
    --out docs/개발문서/S3/phase0/산출물/FG0-4_실측_$(date +%Y%m%d_%H%M)_openai.json
```

### 6.4 본 보고서 갱신

1. §3 표를 실 실측 JSON 두 건의 `summary` 필드로 대체
2. §4 "현재 상태" mock 표 → 실제 pass/fail 로 변경
3. §5 편차 표 작성
4. 미달 시 §4.3 원인 진단 작성
5. §0 "이 보고서의 성격" 섹션 제거

---

## 7. 보안 주의

- 외부 OpenAI 판정 시 **citation snippet (= 사내 문서 내용)** 이 OpenAI 에 전송됨. 사내 정책상 허용되지 않으면 반드시 `RAG_JUDGE_PROVIDER=ollama` 또는 `vllm` 사용.
- 판정 결과 JSON (`FG0-4_실측_*.json`) 에는 citation snippet 이 포함될 수 있음. **git push 전** 외부 repo 로 가는지 확인하거나 `.gitignore` 적용.
- `RAG_JUDGE_API_KEY` 는 `.env` 또는 secret manager 에만 두며 로그 출력 금지 — 본 러너는 키를 `Authorization: Bearer ...` 에 한 번만 사용하고 콘솔 노출 없음.
- 상세: `FG0-4_보안취약점검사보고서.md`

---

## 8. 참조

- `docs/개발문서/S3/phase0/작업지시서/task0-4.md`
- `docs/개발문서/S2_5/RAG_스모크_실행가이드.md`
- `scripts/rag_smoke/judge.py`
- `scripts/rag_smoke/rag_quality.py`
- `scripts/rag_smoke/golden_queries.py`
- `scripts/rag_smoke/sample_documents.py`
- `CLAUDE.md` S1 ⑤ + S2 ⑦

---

*작성: 2026-04-23 | 실 실측은 운영자가 §6 절차로 수행하여 §3~§5 를 갱신한다*
