# Phase 1 개발계획서
## 모델·프롬프트 추상화 계층

---

## 1. 단계 개요

### 단계명
Phase 1. 모델·프롬프트 추상화 계층

### 목적
외부 LLM(OpenAI, Anthropic 등)과 폐쇄망 모델(vLLM, Ollama, llama.cpp)을 동등하게 사용할 수 있는 추상화 계층을 구축한다. 이 계층은 S2의 **기반층**이며, Phase 2~9의 모든 AI 기능이 이 추상화 위에서 동작한다. 또한 폐쇄망 배포 요구사항(원칙 ⑦)을 구조적으로 충족한다.

### 선행 조건
- Phase 0 완료 (S2 원칙, 설계 결정 확정)
- 프로젝트에 필요한 외부 LLM API 키 및 폐쇄망 모델 준비 (선택)
- 기존 RAG, 프롬프트 구현 상태 분석 완료

### 기대 결과
- LLM, Embedding, Prompt를 하드코딩 없이 런타임에 선택 가능
- OpenAI, Anthropic, vLLM, Ollama, llama.cpp를 모두 지원
- 외부망 없이 폐쇄망에서도 동일 기능 제공
- 판정 LLM(평가용)도 동일 추상화 사용
- Prompt Registry를 통해 프롬프트 A/B 테스트 가능
- `/api/v1/system/capabilities`에 사용 가능 provider 목록 노출

---

## 2. Phase 1의 역할

Phase 1은 S2의 **기반층**이다. 후속 모든 Phase는 이 계층에 의존한다:

- **Phase 2 (Grounded Retrieval v2)**: 검색 재작성에 LLM 사용 (이 Phase의 LLM Provider로 실행)
- **Phase 3 (Conversation)**: 멀티턴 컨텍스트 압축에 LLM 사용
- **Phase 4 (Agent-Facing Interface)**: MCP Tools의 응답을 구조화할 때 선택적으로 LLM 호출
- **Phase 5 (Agent Action Plane)**: 에이전트 제안의 프롬프트 인젝션 탐지에 LLM 사용
- **Phase 7 (AI 품질 평가)**: 모든 평가(Faithfulness, Context Precision 등)는 이 Phase의 판정 LLM으로 실행
- **Phase 8 (Structured Extraction)**: 구조화 추출의 모든 LLM 호출은 이 계층으로 라우팅

즉, Phase 1 없이 후속 Phase들이 모두 "외부 LLM 의존"이 되거나, 각각 LLM 로직을 중복 작성하게 된다.

---

## 3. Feature Group 상세

### FG1.1 — LLM Provider Abstraction

#### 목적
서로 다른 LLM 서비스(OpenAI, Anthropic, vLLM 등)를 단일 인터페이스로 통합하여, 런타임에 provider를 교체 가능하게 한다.

#### 주요 작업 항목

1. **LLMProvider 인터페이스 설계**
   - 핵심 메서드: `generate(prompt, **kwargs) → LLMResponse`
   - 스트리밍 지원: `stream_generate(prompt, **kwargs) → AsyncIterator[LLMDelta]`
   - 배치 처리: `batch_generate(prompts: List[str]) → List[LLMResponse]`
   - 토큰 계산: `count_tokens(text: str) → int`
   - 응답 구조: `LLMResponse = {content: str, tokens_used: int, cost: float, latency_ms: float, model: str, provider: str}`

2. **OpenAI Provider 구현**
   - 지원 모델: `gpt-4o`, `gpt-4-turbo`, `gpt-3.5-turbo` (선택 가능)
   - 환경변수: `OPENAI_API_KEY`, `OPENAI_MODEL`
   - 스트리밍 지원: SSE 방식
   - 토큰 계산: `tiktoken` 라이브러리 활용
   - 비용 추적: API 응답의 `usage` 필드에서 추출
   - 오류 처리: rate limit, timeout, invalid key 구분

3. **Anthropic Provider 구현**
   - 지원 모델: `claude-3-opus`, `claude-3-sonnet`, `claude-3-haiku` (선택 가능)
   - 환경변수: `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL`
   - 스트리밍 지원: EventSource 방식
   - 토큰 계산: Anthropic API의 `usage` 필드 사용
   - 비용 추적: 모델별 가격표 기반 계산
   - 오류 처리: rate limit, context window overflow 구분

4. **vLLM Provider 구현** (폐쇄망)
   - 자체 호스팅 vLLM 서버와 HTTP 통신
   - 환경변수: `VLLM_BASE_URL` (e.g., `http://localhost:8000`), `VLLM_MODEL`
   - OpenAI 호환 API 활용 (`/v1/completions`, `/v1/chat/completions`)
   - 스트리밍 지원: 동일 OpenAI 호환 스트림
   - 토큰 계산: vLLM 자체 tokenizer 사용
   - 비용 추적: 0 (자체 호스팅)
   - 외부망 불필요

5. **Ollama Provider 구현** (폐쇄망)
   - 로컬 Ollama 서버와 HTTP 통신
   - 환경변수: `OLLAMA_BASE_URL` (e.g., `http://localhost:11434`), `OLLAMA_MODEL` (e.g., `llama2`, `mistral`)
   - API: `/api/generate` (단순), `/api/chat` (채팅)
   - 스트리밍 지원: JSON 라인 스트림
   - 토큰 계산: Ollama 자체 tokenizer (근사)
   - 비용 추적: 0 (자체 호스팅)
   - 외부망 불필요

6. **llama.cpp Provider 구현** (폐쇄망)
   - llama.cpp HTTP 서버와 통신
   - 환경변수: `LLAMA_CPP_BASE_URL` (e.g., `http://localhost:8000`), `LLAMA_CPP_MODEL`
   - API: 기본 `/completions`, `/chat/completions`
   - 스트리밍 지원: OpenAI 호환 스트림
   - 토큰 계산: llama.cpp 기본 tokenizer
   - 비용 추적: 0 (자체 호스팅)
   - 외부망 불필요

7. **Provider Factory 구현**
   - `LLMProviderFactory.create(provider_name: str) → LLMProvider`
   - 환경변수 `LLM_PROVIDER` (openai | anthropic | vllm | ollama | llama_cpp)에서 읽음
   - 해당 환경변수 검증 (API 키 또는 서버 URL 필수)
   - Provider 생성 실패 시 명확한 오류 메시지
   - 캐싱: 동일 provider 반복 생성 시 싱글톤 재사용

8. **LLM Provider 레지스트리**
   - 런타임에 등록된 모든 provider 목록 조회 가능
   - `/api/v1/system/capabilities`의 `supported_providers` 필드 채우기
   - 각 provider의 상태(활성/비활성/오류) 표시
   - 조회 API: `GET /api/v1/llm/providers` (Admin 권한 필요)

#### 입력 (선행 의존)
- Phase 0 완료 (capabilities endpoint 기반)
- 외부 API 키 또는 폐쇄망 모델 준비 (선택)
- 기존 RAG 프롬프트 분석

#### 출력 (산출물)
- `LLMProvider` 인터페이스 정의 (abstract base class)
- `OpenAILLMProvider`, `AnthropicLLMProvider` 구현
- `vLLMProvider`, `OllamaProvider`, `LlamaCppProvider` 구현
- `LLMProviderFactory` 구현
- LLM Provider 레지스트리 및 조회 API
- 통합 테스트 (모든 provider 동작 확인)

#### 검수 기준
- 모든 provider에서 `generate()` 메서드 정상 작동
- 스트리밍 응답이 청크 단위로 정상 수신
- 토큰 계산이 모델별로 정확 (±5% 오차 허용)
- 비용 추적이 정상 작동 (외부망 provider 기준)
- vLLM, Ollama, llama.cpp에서 외부망 미사용 확인
- Provider 교체 시 동일 프롬프트로 동일 구조의 응답 반환

---

### FG1.2 — Embedding Model Abstraction

#### 목적
텍스트를 벡터로 변환하는 임베딩 모델(OpenAI, BGE-M3, E5 등)을 동일하게 사용할 수 있도록 추상화한다.

#### 주요 작업 항목

1. **EmbeddingProvider 인터페이스 설계**
   - 핵심 메서드: `embed(text: str) → np.ndarray` (1D vector)
   - 배치: `embed_batch(texts: List[str]) → List[np.ndarray]`
   - 차원 조회: `get_dimension() → int`
   - 모델명 조회: `get_model_name() → str`
   - 응답 구조: `EmbeddingResponse = {vector: np.ndarray, dimension: int, tokens_used: int, cost: float}`

2. **OpenAI Embedding Provider 구현**
   - 모델: `text-embedding-3-small` (기본), `text-embedding-3-large` (선택)
   - 차원: small=1536, large=3072
   - 배치 최대: 2048 건
   - 비용: `$0.02 / 1M tokens`
   - 환경변수: `OPENAI_API_KEY`, `OPENAI_EMBEDDING_MODEL`

3. **BGE-M3 로컬 Provider 구현**
   - 패키지: `sentence-transformers` 라이브러리
   - 모델: `BAAI/bge-m3` (또는 ONNX 가속 버전)
   - 차원: 1024
   - 비용: 0 (로컬 실행)
   - 장점: 다국어 지원, 속도 우수
   - 환경변수: `BGE_MODEL_PATH` (필수 — 로컬 모델 파일 경로 지정)
   - **폐쇄망 지원 정책**:
     - Hugging Face 자동 다운로드에 의존하지 않음 (폐쇄망 원칙 위반 방지)
     - 사전 반입(pre-provisioning) 경로: 관리자가 모델 아티팩트를 `BGE_MODEL_PATH`에 수동 배치
     - 무결성 검증: 모델 로드 시 SHA256 체크섬 비교 (`BGE_MODEL_CHECKSUM` 환경변수)
     - 미설치 시 fallback: 모델 파일 미존재 시 `EmbeddingProvider`를 `unavailable` 상태로 등록하고, FTS(Full-Text Search) 전용 모드로 degrade (서비스 실패 아님)
     - 시작 로그에 `embedding_provider=bge, status=unavailable, reason=model_not_found` 기록

4. **E5 로컬 Provider 구현**
   - 패키지: `sentence-transformers` 라이브러리
   - 모델: `intfloat/e5-base-v2` (또는 large)
   - 차원: base=768, large=1024
   - 비용: 0 (로컬 실행)
   - 장점: 높은 품질, 다국어 지원
   - 환경변수: `E5_MODEL_PATH` (필수 — 로컬 모델 파일 경로 지정)
   - **폐쇄망 지원 정책**: BGE-M3와 동일 (사전 반입, 무결성 검증, 미설치 fallback)

5. **EmbeddingProviderFactory 구현**
   - `EmbeddingProviderFactory.create(provider_name: str) → EmbeddingProvider`
   - 환경변수 `EMBEDDING_PROVIDER` (openai | bge | e5)
   - **모델 해결(resolution) 정책**:
     - 온라인 환경: `MODEL_AUTO_DOWNLOAD=true` 설정 시에만 Hugging Face에서 자동 다운로드 허용 (기본값: `false`)
     - 폐쇄망 환경: `MODEL_AUTO_DOWNLOAD=false` (기본값) → 로컬 경로에서만 모델 로드
     - 모델 미존재 시: 해당 provider를 `unavailable`로 등록, FTS fallback 활성화, 서비스는 계속 동작
   - 첫 사용 시 모델 로딩 지연 고려 (사전 로드 옵션 제공)

6. **임베딩 벡터 차원 검증**
   - S1 Phase 10 벡터화에서 모델을 교체할 때 차원 불일치 감지
   - pgvector 인덱스 재구축 절차 설계 (마이그레이션 플레이북)
   - 차원 변경 시 기존 벡터 삭제 또는 변환 전략 정의

7. **재임베딩 마이그레이션 파이프라인**
   - 상황: 기존 벡터(OpenAI 3-small, 1536차)가 있는데, BGE-M3(1024차)로 변경하려는 경우
   - 절차:
     - 새 모델로 document_chunks 전체 재임베딩 실행
     - 기존 벡터는 `is_current=false` 처리 (소프트 삭제)
     - 신규 벡터는 새 차원으로 저장 (pgvector 인덱스 재구축)
     - 마이그레이션 전/후 품질 비교 테스트
   - API: `POST /api/v1/admin/embeddings/migrate` (Admin 권한)
   - 백그라운드 잡으로 비동기 실행

#### 입력 (선행 의존)
- Phase 0 완료
- 필요한 로컬 모델 또는 OpenAI API 키

#### 출력 (산출물)
- `EmbeddingProvider` 인터페이스 정의
- `OpenAIEmbeddingProvider`, `BGEEmbeddingProvider`, `E5EmbeddingProvider` 구현
- `EmbeddingProviderFactory` 구현
- 재임베딩 마이그레이션 파이프라인
- 차원 검증 및 오류 처리 로직
- 마이그레이션 플레이북

#### 검수 기준
- 모든 provider에서 `embed()` 메서드 정상 작동
- 벡터 차원이 모델별로 올바른가 (small=1536, BGE=1024, E5=768)
- 배치 처리 시 모든 벡터가 정상 생성됨
- 재임베딩 마이그레이션이 무중단 실행 가능
- 마이그레이션 전후 검색 품질 비교 결과 기록

---

### FG1.3 — Prompt Registry

#### 목적
프롬프트 템플릿을 DB에 버전 관리하고, 런타임에 선택 가능하게 한다. A/B 테스트 및 평가 자동화를 지원한다.

#### 주요 작업 항목

1. **Prompt 도메인 모델**
   - `Prompt` 엔티티:
     - `id`: UUID
     - `name`: 문자열 (e.g., "retrieval_rewrite_prompt")
     - `description`: 설명
     - `category`: enum (retrieval, evaluation, extraction, etc.)
     - `template`: 프롬프트 텍스트 (Jinja2 템플릿 지원)
     - `variables`: List[str] (template의 변수 목록, 자동 추출)
     - `created_at`, `updated_at`
     - `created_by`: 사용자 ID
   - `PromptVersion` 엔티티 (자동 생성):
     - `id`: UUID
     - `prompt_id`: FK
     - `version`: int (1, 2, 3, ...)
     - `template`: 프롬프트 텍스트
     - `is_active`: boolean (활성 버전 여부)
     - `created_at`
     - `usage_count`: 사용 횟수 (통계용)
     - `average_quality_score`: 평균 품질 점수 (Phase 7 연계)

2. **Prompt 저장소 API**
   - `POST /api/v1/admin/prompts`: Prompt 생성 (Admin 권한)
   - `GET /api/v1/admin/prompts`: Prompt 목록 (Admin 권한)
   - `GET /api/v1/admin/prompts/{id}`: Prompt 상세 조회
   - `PUT /api/v1/admin/prompts/{id}`: Prompt 수정 (새 버전 자동 생성)
   - `DELETE /api/v1/admin/prompts/{id}`: Prompt 삭제 (soft delete)
   - `GET /api/v1/admin/prompts/{id}/versions`: 버전 이력 조회
   - `POST /api/v1/admin/prompts/{id}/versions/{v}/activate`: 특정 버전 활성화
   - 응답 포맷: 표준 envelope `{success, data, error}`

3. **Prompt 런타임 로더**
   - `PromptRegistry` 클래스:
     - `load_prompt(name: str) → str`: 활성 버전의 프롬프트 템플릿 로드
     - `render(name: str, **variables) → str`: 프롬프트 렌더링 (Jinja2 처리)
     - `load_version(name: str, version: int) → str`: 특정 버전 로드
     - `list_variables(name: str) → List[str]`: 프롬프트의 변수 목록 조회
   - 캐싱: 활성 프롬프트는 메모리에 캐시 (변경 시 자동 갱신)
   - 오류 처리: 미존재 프롬프트, 변수 부족, 렌더링 실패 구분

4. **Jinja2 템플릿 지원**
   - 기본 프롬프트 템플릿 예시:
     ```jinja2
     당신은 검색 질의 재작성 전문가입니다.
     원본 질의: {{ original_query }}
     이전 대화 맥락:
     {% for msg in conversation_history %}
     - {{ msg.role }}: {{ msg.content }}
     {% endfor %}
     
     위 맥락을 고려하여, 원본 질의를 더 자립적으로 다시 작성하세요.
     ```
   - 변수 추출: regex로 `{{ var }}` 패턴 찾아서 `variables` 필드에 저장
   - Jinja2 안전성: 비신뢰 입력 차단 (template injection 방어)

5. **A/B 테스트 지원**
   - 프롬프트 설정 시 `ab_variant` 필드 추가 (선택)
   - 예: "retrieval_rewrite_v1" vs "retrieval_rewrite_v2"
   - Phase 7에서 두 버전의 품질 점수 비교 가능

6. **Prompt 기본 라이브러리**
   - Phase 1 종료 시 기본 제공할 프롬프트 목록:
     - `retrieval_rewrite_prompt`: 검색 질의 재작성
     - `evaluation_faithfulness_prompt`: Faithfulness 평가
     - `evaluation_relevance_prompt`: Answer Relevance 평가
     - `injection_detection_prompt`: 프롬프트 인젝션 탐지
     - `extraction_prompt_template`: 구조화 추출의 기본 템플릿
   - 각 프롬프트를 기본값(v1)과 함께 DB에 시드

7. **감사 로깅**
   - Prompt 생성/수정/삭제 시 감사 로그 기록
   - 활성 버전 변경 시도 및 성공 로깅
   - 프롬프트 사용 통계 (usage_count 추적)

#### 입력 (선행 의존)
- Phase 0 완료
- 기존 RAG 프롬프트 수집 (Phase 2를 위한 사전작업)

#### 출력 (산출물)
- `Prompt`, `PromptVersion` DB 모델
- Prompt 저장소 API (FastAPI endpoints)
- `PromptRegistry` 런타임 로더
- Jinja2 템플릿 처리 로직
- 기본 프롬프트 라이브러리 (시드 데이터)
- A/B 테스트 설정 기능

#### 검수 기준
- Prompt CRUD 정상 작동
- 버전 자동 생성 및 이력 조회 정상
- 활성 버전 변경 정상 작동
- 프롬프트 렌더링 (변수 치환) 정상 작동
- Jinja2 안전성 검증 (injection 차단)
- 캐시 갱신 정상 여부
- 기본 프롬프트 라이브러리 로드 성공

---

## 4. 기술 설계 요약

### 4.1 계층 아키텍처
```
Application Layer (Phase 2~9: Retrieval, Conversation, Agent, etc.)
            ↓
LLM Provider Factory (LLMProviderFactory, EmbeddingProviderFactory)
            ↓
Provider Implementations
├─ OpenAI / Anthropic (외부망)
├─ vLLM / Ollama / llama.cpp (폐쇄망)
└─ Embedding: OpenAI / BGE / E5
            ↓
External APIs / Local Servers
```

### 4.2 환경변수 기반 설정
```
# LLM 설정
LLM_PROVIDER=openai|anthropic|vllm|ollama|llama_cpp
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-3-opus
VLLM_BASE_URL=http://localhost:8000
VLLM_MODEL=meta-llama/Llama-2-7b-hf
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=llama2

# Embedding 설정
EMBEDDING_PROVIDER=openai|bge|e5
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
BGE_MODEL_PATH=/path/to/bge-m3
E5_MODEL_PATH=/path/to/e5-base-v2

# 평가용 LLM (Phase 7에서 사용)
EVALUATION_LLM_PROVIDER=openai|vllm|ollama
EVALUATION_MODEL=gpt-4o|llama2
```

### 4.3 오류 처리 전략
- Provider 초기화 실패: 명확한 오류 메시지 + 환경변수 체크리스트 출력
- API 호출 실패: rate limit, timeout, invalid key 등 세부 분류
- 폐쇄망 모델 불가: "LLM_PROVIDER=ollama이지만 OLLAMA_BASE_URL이 응답 없음" 등 명시
- 폴백: 일부 provider 오류 시 다른 provider로 자동 전환 가능 (선택적)

### 4.4 성능 고려사항
- 배치 임베딩: 건당 100ms → 배치 1000건 → 전체 1초 수준
- LLM 스트리밍: 청크 단위 응답 (첫 토큰까지 지연 최소화)
- 프롬프트 캐시: 메모리 캐시 + DB 백업 (캐시 미스 시 DB 조회)
- 토큰 계산: 온디맨드 계산 vs 사전 계산 (API 호출과 함께 제공된 usage 사용)

---

## 5. 의존 관계

### 선행 Phase
- **Phase 0**: 원칙 확정, capabilities endpoint 기반 제공

### 후행 Phase (이 Phase의 산출물을 소비하는 Phase)
- **Phase 2 (Grounded Retrieval v2)**: 검색 질의 재작성에 LLMProvider 사용
- **Phase 3 (Conversation)**: 멀티턴 컨텍스트 압축에 LLMProvider 사용
- **Phase 4 (Agent-Facing Interface)**: 응답 구조화에 선택적 LLM 호출
- **Phase 5 (Agent Action Plane)**: 프롬프트 인젝션 탐지에 LLMProvider 사용
- **Phase 7 (AI 품질 평가)**: 모든 평가 지표의 판정 LLM으로 사용
- **Phase 8 (Structured Extraction)**: 추출 파이프라인의 핵심 LLM

---

## 6. 검수 기준 종합

| 항목 | 기준 |
|------|------|
| **FG1.1 - LLM Provider** | 모든 provider (OpenAI, Anthropic, vLLM, Ollama, llama.cpp) 정상 작동 / 스트리밍 응답 정상 / 토큰 계산 ±5% 오차 내 / 비용 추적 정상 / 폐쇄망 provider 외부망 미사용 확인 |
| **FG1.2 - Embedding** | 모든 임베딩 provider 정상 작동 / 벡터 차원 정확 / 배치 처리 성공 / 재임베딩 마이그레이션 무중단 / 마이그레이션 전후 품질 비교 기록 |
| **FG1.3 - Prompt Registry** | Prompt CRUD 정상 / 버전 자동 생성 및 이력 / 활성 버전 변경 / 프롬프트 렌더링(변수 치환) / Jinja2 injection 차단 / 캐시 갱신 |
| **AI품질평가** | Phase 1의 LLM 추상화가 외부망/폐쇄망 동등성을 구조적으로 지원하는가 |
| **종합** | 특정 provider 변경 후 Phase 2~9 기능이 동일하게 동작하는가 |

---

## 7. 예상 산출물 목록

| 산출물 | 형태 | 설명 |
|--------|------|------|
| LLMProvider 인터페이스 | Python (abc.ABC) | 추상 베이스 클래스 |
| OpenAI/Anthropic/vLLM/Ollama/llama.cpp Provider | Python | 각 provider 구현체 |
| LLMProviderFactory | Python | provider 생성 팩토리 |
| EmbeddingProvider 인터페이스 | Python (abc.ABC) | 임베딩 추상 베이스 |
| OpenAI/BGE/E5 Embedding Provider | Python | 각 임베딩 구현체 |
| EmbeddingProviderFactory | Python | embedding provider 팩토리 |
| 재임베딩 마이그레이션 파이프라인 | Python | 모델 변경 시 벡터 재계산 |
| Prompt 도메인 모델 | SQLAlchemy ORM | Prompt, PromptVersion 엔티티 |
| Prompt 저장소 API | FastAPI endpoints | CRUD + 버전 관리 |
| PromptRegistry | Python | 런타임 프롬프트 로더 |
| Jinja2 템플릿 처리 | Python | 변수 치환 및 안전성 |
| 기본 프롬프트 라이브러리 | JSON seed data | 기본 프롬프트 시드 |
| 환경변수 설정 가이드 | Markdown | 설정 방법 문서 |
| provider 통합 테스트 | pytest | 모든 provider 동작 검증 |
| **FG1.1 검수보고서** | Markdown | LLM Provider 검수 결과 |
| **FG1.1 보안취약점검사보고서** | Markdown | API 키 관리, injection 방어 |
| **FG1.2 검수보고서** | Markdown | Embedding 검수 결과 |
| **FG1.2 보안취약점검사보고서** | Markdown | 모델 로드 보안 |
| **FG1.3 검수보고서** | Markdown | Prompt Registry 검수 |
| **FG1.3 보안취약점검사보고서** | Markdown | Jinja2 injection 방어 |
| **FG1.3 AI품질평가** | Markdown | 프롬프트 품질 평가 (선택) |
| **Phase 1 종결 보고서** | Markdown | Phase 1 완료 보고 |

---

## 8. 리스크 및 대응

| 리스크 | 영향도 | 대응 방안 |
|--------|--------|----------|
| 외부 API 속도 저하 (OpenAI, Anthropic) | 중간 | timeout 설정, 재시도 정책, 폐쇄망 모델 백업 |
| 폐쇄망 모델 메모리 부족 (vLLM, Ollama) | 중간 | 양자화 모델, 모델 크기 사전 계획 |
| Embedding 차원 변경 시 기존 벡터 호환성 | 높음 | 마이그레이션 절차 사전 설계 및 테스트 |
| Jinja2 template injection 취약점 | 높음 | sandboxed environment 사용, 입력 검증 |
| Provider별 응답 형식 차이 | 중간 | 단위 테스트로 응답 정규화 검증 |
| 프롬프트 DB 중복/버전 관리 혼동 | 낮음~중간 | 버전 자동 관리, 명확한 활성/비활성 상태 |

---

## 9. 선행 조건

Phase 1을 시작하려면:
- Phase 0 완료 (원칙, capabilities endpoint)
- 필요한 외부 API 키 확보 (OpenAI, Anthropic) 또는 폐쇄망 모델 설치 (vLLM, Ollama)
- 기존 RAG 프롬프트 수집 (FG1.3 기본 라이브러리 작성을 위해)

---

## 10. 완료 기준

Phase 1 완료로 판단하는 조건:

1. LLMProvider 인터페이스 정의 및 모든 구현체(OpenAI, Anthropic, vLLM, Ollama, llama.cpp) 동작
2. EmbeddingProvider 인터페이스 정의 및 모든 구현체(OpenAI, BGE, E5) 동작
3. LLMProviderFactory, EmbeddingProviderFactory 정상 작동
4. 특정 provider 변경 후 동일 프롬프트로 동일 응답 구조 확인
5. 토큰 계산 및 비용 추적 정상
6. 폐쇄망 provider(vLLM, Ollama, llama.cpp)에서 외부망 미사용 확인
7. Prompt 저장소 API (CRUD, 버전 관리) 정상 작동
8. PromptRegistry 캐시 갱신 정상
9. Jinja2 템플릿 injection 방어 검증
10. 기본 프롬프트 라이브러리 로드 성공
11. 재임베딩 마이그레이션 파이프라인 테스트 통과
12. `/api/v1/system/capabilities`의 `supported_providers` 필드 정확
13. 모든 FG 검수보고서 및 보안취약점검사보고서 승인
14. Phase 1 종결 보고서 작성 완료

---

## 11. 권장 투입 순서

1. FG1.1 LLM Provider Abstraction
   - LLMProvider 인터페이스 설계
   - OpenAI / Anthropic Provider 구현 (외부망)
   - vLLM / Ollama / llama.cpp Provider 구현 (폐쇄망)
   - LLMProviderFactory 및 레지스트리
2. FG1.2 Embedding Model Abstraction
   - EmbeddingProvider 인터페이스 설계
   - OpenAI / BGE / E5 Provider 구현
   - EmbeddingProviderFactory
   - 재임베딩 마이그레이션 파이프라인
3. FG1.3 Prompt Registry
   - Prompt 도메인 모델 (DB 마이그레이션)
   - Prompt 저장소 API
   - PromptRegistry 런타임 로더
   - Jinja2 템플릿 처리
   - 기본 프롬프트 라이브러리
4. 통합 테스트 및 검수
5. AI품질평가 및 보안 검사
6. Phase 1 종결 보고서

---

## 12. 기대 효과

Phase 1 완료 시:
- 후속 Phase(2~9)가 모두 이 추상화 계층에 의존 가능
- "gpt-4o 사고류"의 구조적 차단 (provider 교체로 해결)
- 폐쇄망 배포 시 로컬 모델로 자동 전환 가능
- 프롬프트를 DB에서 관리하여 A/B 테스트 및 버전 관리 가능
- 평가 LLM도 동일 추상화 사용으로 판정 모델 스왑 가능
