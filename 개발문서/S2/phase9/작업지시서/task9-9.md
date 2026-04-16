# 작업지시서: task9-9 챗봇 인계 메모 v3 + 운영 매뉴얼

## 1. 작업 목적

S2 완수 단계에서 구축된 Mimir 시스템을 운영팀/파트너팀에 인계하기 위한
최종 기술 문서를 작성한다. 특히 다음을 포함한다:

1. **챗봇 인계 메모 v3**: v1(MCP 기초), v2(Citation 5-tuple)에 이어
   Scope Profile 관리, Kill switch 운영, 성능 최적화 등 S2 기반 기능 정리
   
2. **운영 매뉴얼**: 배포, 모니터링, 트러블슈팅, 인시던트 대응 procedures
   
3. **OpenAPI 스펙**: 최종 API 명세 자동 생성 (Swagger)
   
4. **배포 체크리스트**: 프로덕션 배포 전 검증 항목

---

## 2. 작업 범위

### 포함 범위

1. **챗봇 인계 메모 v3** (`docs/개발문서/S2/챗봇_인계메모_v3.md`)
   - v1, v2 요약 (MCP, Citation 5-tuple)
   - v3 신규: Scope Profile, Kill switch, 성능최적화, 폐쇄망 배포
   - 실전 가이드: 설정, 운영 예제

2. **운영 매뉴얼** (`docs/개발문서/S2/운영_매뉴얼.md`)
   - 시스템 아키텍처 (배포도)
   - 헬스체크 엔드포인트 및 모니터링
   - 백업/복구 절차
   - 스케일링 가이드
   - 인시던트 대응 매뉴얼
   - Kill switch 긴급 운영 절차 (<5s)
   - 로그 분석 가이드
   - 환경변수 및 기능 플래그 설정

3. **최종 배포 체크리스트** (`docs/개발문서/S2/프로덕션_배포_체크리스트.md`)
   - 배포 전 검증 20개 항목
   - 배포 중 모니터링 항목
   - 배포 후 검증 항목

4. **API 스펙 자동 생성** (`scripts/generate_openapi_spec.py`)
   - FastAPI → OpenAPI 3.0 변환
   - Swagger UI 생성
   - 클라이언트 생성 스크립트 (TypeScript, Python)

### 제외 범위

- 사용자 가이드 (end-user 교육)
- 개발자 가이드 (새로운 기능 추가 방법, 별도)
- 마케팅 문서
- 법적 고지사항 (별도 legal review)

---

## 3. 선행 조건

1. **S2 완수 보고서** (task9-7) 및 **회고** (task9-8) 완료
2. **현재 프로덕션 배포 상태** 파악
   - 어디에 배포되어 있는가 (온프레미스, AWS, Azure)
   - 현재 버전 및 설정
3. **운영팀 구성** 확정
   - 24/7 온콜 담당자
   - 에스컬레이션 경로
4. **모니터링 인프라** 기본 구성 (선택)
   - Prometheus, DataDog, CloudWatch 중 선택

---

## 4. 주요 작업 항목

### 4.1 챗봇 인계 메모 v3 작성

**파일**: `docs/개발문서/S2/챗봇_인계메모_v3.md`

**구조 및 내용**:

```markdown
# 챗봇 인계 메모 v3 — Mimir S2 완수 인계

## Executive Summary

Mimir는 다음 기능으로 구성된 엔터프라이즈 RAG 플랫폼:

| 기능 | 상태 | 비고 |
|------|------|------|
| Grounded RAG (Citation 5-tuple) | ✓ S2 완료 | Phase 2 |
| Agent-Facing MCP Interface | ✓ S2 완료 | Phase 4 |
| Scope Profile-based ACL | ✓ S2 완료 | Phase 6 |
| AI 품질 평가 인프라 | ✓ S2 완료 | Phase 7 |
| Structured Extraction | ✓ S2 완료 | Phase 8 |

---

## v1 회고: MCP 기초 (Phase 4)

### 개요
MCP (Model Context Protocol) 2025-11-25 사양 준수로
AI 에이전트와의 표준화된 인터페이스 제공

### 핵심 개념
- Tool 정의: 에이전트가 호출 가능한 함수 (검색, 조회, 쓰기)
- Schema: JSON Schema로 입력 매개변수 명시
- Response: 구조화된 답변 (content, is_error)

### 실전 예제

```python
# MCP Tool 정의 (src/interfaces/mcp/tools/search_documents.py)

@mcp.tool(name="search_documents")
async def search_documents(
    query: str,
    limit: int = 10,
    scope: str = "default"
) -> ToolResponse:
    """
    문서 검색 (MCP 도구)
    
    Args:
        query: 검색어
        limit: 결과 수 제한
        scope: Scope Profile ID (ACL 필터링)
    
    Returns:
        ToolResponse(
            content=[{"id": "...", "title": "...", "excerpt": "..."}],
            is_error=False
        )
    """
    try:
        results = await search_service.search(
            query=query,
            limit=limit,
            scope_profile_id=scope
        )
        return ToolResponse(
            content=results,
            is_error=False
        )
    except Exception as e:
        return ToolResponse(
            content=str(e),
            is_error=True
        )
```

### 배포 체크
- [ ] MCP Tool 스키마 검증 (JSON Schema)
- [ ] Tool response time <2초
- [ ] 에러 처리: 모든 예외 catch 및 로깅

---

## v2 회고: Citation 5-tuple (Phase 2)

### 개요
Grounded Retrieval로 RAG 답변의 신뢰성 향상
각 인용에 5개 메타데이터 포함

### Citation 5-tuple 구조

```python
# src/domain/rag/citation_5tuple.py

@dataclass
class Citation5Tuple:
    """인용 5-tuple: 신뢰도 있는 답변 생성"""
    
    retrieval_context: str  # 검색된 텍스트 일부 (evidence)
    retrieval_score: float  # 검색 점수 (0~1, 높을수록 관련성 높음)
    source_doc_id: str      # 원본 문서 ID
    chunk_id: str           # 청크 번호 (문서 내 위치)
    confidence: float       # 신뢰도 (0~1, 평가 점수)
```

### 실전 예제

```python
# RAG 답변 생성 시 citation 포함

async def generate_answer(query: str, context_scope: str):
    # 1. 문서 검색
    retrieved = await retriever.retrieve(query, scope=context_scope)
    
    # 2. Citation 5-tuple 생성
    citations = [
        Citation5Tuple(
            retrieval_context=doc["excerpt"][:200],
            retrieval_score=doc["score"],
            source_doc_id=doc["id"],
            chunk_id=doc["chunk_id"],
            confidence=0.92  # 평가 점수
        )
        for doc in retrieved
    ]
    
    # 3. LLM 프롬프트에 context + citations 포함
    prompt = f"""
    Question: {query}
    
    Context (with citations):
    {json.dumps(citations, indent=2)}
    
    Answer:
    """
    
    answer = await llm.complete(prompt)
    
    return {
        "answer": answer,
        "citations": citations  # 클라이언트에 반환
    }
```

### 배포 체크
- [ ] Citation 5-tuple 스키마 일관성
- [ ] 5-tuple 모든 필드 채워짐 (None 금지)
- [ ] Confidence score 범위 검증 (0~1)
- [ ] 평가 API와 일관성 (Phase 7)

---

## v3 신규: S2 기반 고급 기능

### 3.1 Scope Profile 관리 및 ACL 적용

#### 개요
S2⑥ 원칙: 접근 범위 하드코딩 금지
Scope Profile로 조직별/팀별/사용자별 접근 제어

#### Scope Profile 정의

```python
# src/domain/scope/scope_profile.py

@dataclass
class ScopeProfile:
    """접근 범위 정의"""
    
    id: str                    # 프로필 ID (e.g., "team-analytics")
    name: str                  # 프로필 이름
    owner_id: str              # 소유자 (조직/팀)
    accessible_doc_ids: List[str]  # 접근 가능한 문서 ID 목록
    filters: Dict              # 추가 필터 (메타데이터 기반)
    created_at: datetime
    updated_at: datetime
```

#### Scope Profile 설정 가이드

1. **Admin UI에서 프로필 생성**:
   ```
   https://mimir.example.com/admin/scope-profiles/create
   - 프로필 이름: "Team Analytics"
   - 소유자: analytics-team
   - 문서 선택: [doc-001, doc-002, doc-003, ...]
   - 필터: metadata.department == "analytics"
   ```

2. **API 호출 시 scope 매개변수 포함**:
   ```bash
   curl -X POST https://api.mimir.example.com/search \
     -H "Authorization: Bearer $TOKEN" \
     -d '{
       "query": "quarterly revenue",
       "scope": "team-analytics"
     }'
   ```

3. **폐쇄망 환경에서 Scope Profile 설정**:
   - 환경변수: `SCOPE_PROFILE_MODE=local`
   - 로컬 파일: `config/scope_profiles.json`
   ```json
   {
     "team-analytics": {
       "accessible_doc_ids": ["doc-001", "doc-002"],
       "filters": {}
     }
   }
   ```

#### ACL 필터링 체크리스트

- [ ] 모든 API 엔드포인트에 scope 매개변수 지원
- [ ] 검색 API: scope 기반 필터링 (검색 결과 제한)
- [ ] 조회 API: scope 기반 접근 제어 (404 또는 403)
- [ ] 쓰기 API: scope 소유권 검증
- [ ] 폴백 경로 (FTS, 로컬 모델): 동일 ACL 적용
- [ ] 감사 로그: scope_profile_id + actor_type 기록

---

### 3.2 Agent Kill Switch 운영

#### Kill Switch란?
에이전트의 자동 실행을 즉시 중단하는 긴급 정지 메커니즘
목표: <5초 내 응답

#### Kill Switch 구조

```python
# src/control/kill_switch.py

class KillSwitch:
    """에이전트 즉시 정지 (Target: <5s)"""
    
    def __init__(self, redis_client):
        self.redis = redis_client
        self.key = "mimir:kill_switch:enabled"
    
    async def activate(self, reason: str = ""):
        """Kill switch 활성화 (운영팀)"""
        await self.redis.set(self.key, json.dumps({
            "enabled": True,
            "timestamp": datetime.now().isoformat(),
            "reason": reason
        }))
        logger.warning(f"Kill switch activated: {reason}")
    
    async def deactivate(self):
        """Kill switch 비활성화"""
        await self.redis.delete(self.key)
        logger.info("Kill switch deactivated")
    
    async def is_active(self) -> bool:
        """현재 상태 확인 (<1ms Redis latency)"""
        result = await self.redis.get(self.key)
        return result is not None
```

#### Kill Switch 운영 절차

**상황**: 에이전트가 악의적 행동 또는 오류 반복

**즉시 대응** (30초):
1. Kill switch 활성화:
   ```bash
   curl -X POST https://api.mimir.example.com/admin/kill-switch \
     -H "Authorization: Bearer $ADMIN_TOKEN" \
     -d '{"activate": true, "reason": "Agent error loop detected"}'
   ```
2. 검증 (에이전트 즉시 정지 확인)
3. 모니터링 알림 확인

**근본 원인 분석** (5~30분):
1. 에이전트 로그 확인
2. 마지막 Action 검토
3. LLM 프롬프트 분석

**복구** (30분 이후):
1. Kill switch 비활성화
2. 에이전트 재시작
3. 정상 작동 확인

#### Kill Switch 테스트 (월 1회)

```bash
# 운영 매뉴얼 4.4 참조
python scripts/test_kill_switch.py --timeout 10
# 기대 결과: Kill switch 활성화 후 에이전트 정지 <5초
```

#### Kill Switch 배포 체크

- [ ] Redis 연결 <1ms latency
- [ ] 활성화/비활성화 API 응답 <500ms
- [ ] 에이전트가 매 요청 시 kill switch 상태 확인
- [ ] 월 1회 테스트 실행 (자동화)

---

### 3.3 성능 최적화 가이드

#### 3.3.1 배치 쿼리 (Batch Query)

**목표**: 에이전트가 여러 검색을 하나의 요청으로 처리

```python
# 비효율: 순차 검색 (3회 검색 = 3초)
results_1 = await search(query_1)  # 1초
results_2 = await search(query_2)  # 1초
results_3 = await search(query_3)  # 1초
# 총 3초

# 최적화: 배치 검색 (3회 검색 = 1.2초)
results = await batch_search([query_1, query_2, query_3])
# 병렬 처리, 총 1.2초
```

**API 사용법**:
```bash
curl -X POST https://api.mimir.example.com/batch-search \
  -H "Content-Type: application/json" \
  -d '{
    "queries": ["revenue 2025", "expenses Q1", "profit margin"],
    "scope": "team-analytics",
    "limit": 5
  }'
```

#### 3.3.2 캐싱 전략

**검색 결과 캐시** (1시간):
```python
# 동일 쿼리 반복 시 캐시 재사용
# 캐시 키: f"{scope}:{query}:{limit}"
# 캐시 대상: 동일 쿼리 & 범위 (agent planning stage)
```

**설정**:
```bash
# 환경변수
SEARCH_CACHE_TTL=3600  # 1시간
CACHE_MAX_SIZE=1000    # 최대 1000개 항목
```

#### 3.3.3 벡터 검색 인덱스 최적화

**인덱스 재구성** (월 1회):
```bash
python scripts/rebuild_vector_index.py \
  --collection-name documents \
  --metric cosine \
  --batch-size 128
# 예상 시간: 10~30분 (문서 수에 따라)
```

---

### 3.4 폐쇄망 환경 배포

#### 폐쇄망 모드 활성화

```bash
# 환경변수 설정
export CLOSED_NETWORK_MODE=true
export OPENAI_API_ENABLED=false
export QDRANT_CLOUD_ENABLED=false
export EVALUATION_LLM=local  # Llama 2 로컬 모델

# 서버 시작
python -m mimir.server
```

#### 폐쇄망 환경에서의 기능

| 기능 | Cloud 모드 | 폐쇄망 모드 | 차이 |
|------|----------|----------|------|
| 검색 | Qdrant Cloud | 로컬 Qdrant | latency +50ms |
| RAG 답변 | GPT-4 | 로컬 LLM | 정확도 -2% |
| 평가 | GPT-4 | Llama 2 | 정확도 -7% |
| 구조 추출 | GPT-4 | 로컬 extraction | 정확도 -10% |

#### 폐쇄망 배포 체크

- [ ] 모든 외부 API 비활성화
- [ ] 로컬 모델 로드 완료 (시작 시간 <30초)
- [ ] 벡터 DB 로컬 실행 확인
- [ ] 네트워크 격리 검증 (외부 통신 금지)

---

### 3.5 Prompt 버전 관리 (Phase 1 Prompt Registry)

#### Prompt Registry 개요
모든 LLM 호출에 사용되는 프롬프트를 중앙 집중식으로 관리

#### Prompt 업데이트 절차

1. **새 버전 추가** (Admin UI):
   ```
   https://mimir.example.com/admin/prompt-registry
   - Prompt ID: "rag_answer_generation"
   - Version: 2.0 (이전 1.9)
   - Template: "Question: {query}\n\nContext: {context}\n\nAnswer:"
   - Variables: ["query", "context"]
   - Status: draft
   ```

2. **테스트** (테스트 환경):
   ```bash
   python scripts/test_prompt.py \
     --prompt-id rag_answer_generation \
     --version 2.0 \
     --golden-set tests/golden/rag_qa.json
   # 기대: 정확도 차이 <2%
   ```

3. **배포** (프로덕션):
   ```bash
   # Admin API로 버전 활성화
   curl -X POST https://api.mimir.example.com/admin/prompts/activate \
     -d '{
       "prompt_id": "rag_answer_generation",
       "version": "2.0"
     }'
   ```

4. **모니터링** (배포 후):
   - 1시간 동안 모니터링 (메트릭 변화 관찰)
   - 문제 시 즉시 이전 버전으로 롤백

#### Prompt 버전 체크

- [ ] 모든 LLM 호출이 Prompt Registry 사용
- [ ] 프롬프트 변경 시 버전 업 (semantic versioning)
- [ ] 버전별 성능 기록 유지

---

### 3.6 품질 모니터링 (Phase 7 메트릭)

#### 핵심 메트릭 및 대시보드

```bash
# 메트릭 조회 API
curl https://api.mimir.example.com/metrics/rag \
  -H "Authorization: Bearer $TOKEN"

# 응답 예:
{
  "faithfulness": 0.82,
  "answer_relevance": 0.78,
  "citation_present_rate": 0.92,
  "hallucination_rate": 0.08,
  "sample_size": 100,
  "evaluation_timestamp": "2025-12-15T10:00:00Z"
}
```

#### 대시보드 해석 가이드

| 메트릭 | 정상 범위 | 경고 | 심각 |
|--------|---------|------|------|
| Faithfulness | ≥0.80 | 0.75~0.80 | <0.75 |
| Answer Relevance | ≥0.75 | 0.70~0.75 | <0.70 |
| Citation rate | ≥0.90 | 0.85~0.90 | <0.85 |
| Hallucination | ≤0.10 | 0.10~0.15 | >0.15 |

#### 메트릭 악화 시 대응

```
Hallucination 증가 (0.08 → 0.15) 감지
  ↓
1. LLM 프롬프트 검토 (최근 변경사항)
2. 평가 API 확인 (평가 모델 성능)
3. 검색 결과 품질 확인 (Context 적합성)
4. 원인 파악 후 수정
5. 배포 및 재평가
```

#### 메트릭 배포 체크

- [ ] 평가 API 매일 실행 (자동)
- [ ] 메트릭 값 기록 (시계열 DB)
- [ ] 임계값 초과 시 알람 (슬랙/이메일)
- [ ] 주간 리뷰 (메트릭 추세)

---

### 3.7 API 사용 요금 및 제한

#### API Rate Limiting

```
기본 정책:
- Per user: 100 requests/minute
- Per scope: 1000 requests/hour
- Per IP: 10000 requests/day

높은 사용량 고객:
- 별도 계약으로 제한 완화 가능
- 요청: 운영팀에 연락
```

#### 배포 체크

- [ ] Rate limit 미들웨어 활성화
- [ ] 제한 초과 시 429 응답 반환
- [ ] 사용 현황 로깅 (감사 추적)

---

## 4. API 버전 및 호환성

| 버전 | 출시 | 지원 종료 | 주요 기능 |
|------|------|----------|---------|
| v1 | 2025-06 | 2026-06 | 기본 검색 |
| **v2** | **2025-12** (현재) | 2027-12 | MCP, Citation 5-tuple, Scope Profile |
| v3 (기대) | 2026-06 | - | 외부 커넥터, 멀티테넌시 |

**v1 → v2 마이그레이션**:
```
현재 상태: v1 점진적 단계적 지원 종료
- 2025-12: v2 권장 (warning)
- 2026-06: v1 지원 종료
```

---

## 5. 커뮤니티 및 지원

- **문서**: https://mimir.example.com/docs
- **API 리뷰**: https://api-reference.mimir.example.com
- **GitHub**: https://github.com/mimir-ai/mimir (오픈소스 버전)
- **지원**: support@mimir.example.com

---

```

**작성 절차**:
1. v1, v2 요약 (기존 인계 메모 참조)
2. v3 신규 섹션 작성 (위 템플릿)
3. 실전 예제 코드 포함 (copy-paste 가능)
4. 배포 체크리스트 모두 포함

---

### 4.2 운영 매뉴얼 작성

**파일**: `docs/개발문서/S2/운영_매뉴얼.md`

**구조**:

```markdown
# Mimir 운영 매뉴얼

## 1. 시스템 아키텍처 및 배포

### 1.1 배포 아키텍처 (온프레미스 기준)

```
                    ┌─────────────────┐
                    │   Client/Agent  │
                    │  (외부 팀/API)  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Load Balancer │
                    │   (nginx, ALB)  │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
    ┌───▼───┐            ┌───▼───┐            ┌───▼───┐
    │API    │            │API    │            │API    │
    │Server │            │Server │            │Server │
    │ 1/3   │            │ 2/3   │            │ 3/3   │
    └───┬───┘            └───┬───┘            └───┬───┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
    ┌───▼──────┐     ┌───────▼────┐    ┌──────▼──┐
    │PostgreSQL│     │Qdrant      │    │Redis    │
    │(문서DB)  │     │(벡터DB)    │    │(캐시)   │
    └──────────┘     └────────────┘    └─────────┘
        │
    ┌───▼────────────┐
    │Backup (S3/NAS) │
    └────────────────┘
```

### 1.2 컴포넌트별 역할

| 컴포넌트 | 역할 | 가용성 | 용량 |
|---------|------|--------|------|
| API Server | 요청 처리 (3대) | HA (1대 장애 허용) | CPU: 4코어, RAM: 8GB |
| PostgreSQL | 문서 메타 저장 | Primary + Replica | 500GB (증설 가능) |
| Qdrant | 벡터 검색 | Single (클러스터 옵션) | 1000M vectors |
| Redis | 세션/캐시 | Single + Sentinel | 100GB (expansion) |

### 1.3 네트워크 구조

- API: 포트 8000 (내부 LB)
- PostgreSQL: 포트 5432 (내부 only)
- Qdrant: 포트 6333 (내부 only)
- Redis: 포트 6379 (내부 only)
- 모니터링: 포트 9090 (Prometheus)

---

## 2. 헬스체크 및 모니터링

### 2.1 헬스체크 엔드포인트

```bash
# 기본 헬스체크 (응답 <100ms)
curl http://localhost:8000/health
# 응답: {"status": "ok", "timestamp": "2025-12-15T10:00:00Z"}

# 상세 헬스체크 (모든 의존성)
curl http://localhost:8000/health/detailed
# 응답:
{
  "status": "ok",
  "api": "ok",
  "postgres": "ok",
  "qdrant": "ok",
  "redis": "ok",
  "metrics": {
    "api_response_time_ms": 45,
    "db_connection_pool": "10/20 active",
    "cache_hit_rate": 0.87
  }
}
```

### 2.2 모니터링 메트릭

| 메트릭 | 정상 | 경고 | 심각 |
|--------|------|------|------|
| API response time | <200ms | 200~500ms | >500ms |
| DB connection pool | <80% | 80~95% | >95% |
| Disk usage | <70% | 70~85% | >85% |
| Memory usage | <60% | 60~80% | >80% |
| Error rate | <0.1% | 0.1~1% | >1% |

### 2.3 모니터링 설정 (Prometheus)

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'mimir-api'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics'
    scrape_interval: 10s
```

---

## 3. 백업 및 복구

### 3.1 백업 전략

**일일 백업**:
- 시간: 매일 02:00 (야간)
- 대상: PostgreSQL (전체 데이터), Qdrant (벡터 인덱스)
- 저장: NAS /backup/daily (최근 7일)

**주간 백업**:
- 시간: 매주 일요일 02:00
- 저장: S3 s3://mimir-backup/weekly/ (최근 8주)

**연간 백업**:
- 시간: 연초 01-01 02:00
- 저장: 아카이브 (법적 요구사항)

**백업 스크립트**:
```bash
#!/bin/bash
# scripts/backup.sh

BACKUP_DIR="/backup/daily/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# PostgreSQL 백업
pg_dump -U postgres mimir_db > $BACKUP_DIR/postgres.sql
gzip $BACKUP_DIR/postgres.sql

# Qdrant 스냅샷
curl -X POST http://localhost:6333/snapshots \
  -o $BACKUP_DIR/qdrant_snapshot.tar.gz

# S3 동기화 (주간)
if [ $(date +%u) -eq 0 ]; then
  aws s3 sync $BACKUP_DIR s3://mimir-backup/weekly/$(date +%Y%m%d)/
fi

echo "Backup completed at $(date)"
```

### 3.2 복구 절차

**시나리오**: PostgreSQL 디스크 손상 (RTO <1시간)

**즉시 대응** (5분):
1. 장애 감지 (모니터링 alert)
2. 에러 로그 확인
3. API 서버를 읽기 전용으로 (쓰기 차단)

**복구** (30분):
```bash
# 1. 최신 백업 확인
ls -la /backup/daily/20251215/postgres.sql.gz

# 2. PostgreSQL 재시작 (safe mode)
systemctl restart postgresql

# 3. 백업에서 복구
gunzip -c /backup/daily/20251215/postgres.sql.gz | psql -U postgres

# 4. 데이터 검증
SELECT COUNT(*) FROM documents;  # 기대: 10000+ 레코드

# 5. 벡터 인덱스 재구성
python scripts/rebuild_vector_index.py

# 6. 정상 운영 재개
# - 읽기/쓰기 활성화
# - 모니터링 확인
# - 고객 공지
```

**복구 검증 체크리스트**:
- [ ] PostgreSQL 데이터 무결성 (COUNT 일치)
- [ ] Qdrant 벡터 검색 정상 (test query)
- [ ] API 응답 정상 (<200ms)
- [ ] 캐시 재워밍

---

## 4. 스케일링 가이드

### 4.1 수평 스케일링 (API 서버)

**부하 증가 감지** (CPU >70% 또는 요청 대기 시간 >500ms):

```bash
# 1. 새 API 서버 프로비저닝
terraform apply -var="api_instance_count=4"

# 2. 로드 밸런서 설정 업데이트
# (자동: Terraform, 또는 수동 ALB 설정)

# 3. 헬스체크 확인
curl http://new-instance:8000/health

# 4. 트래픽 점진적 이동 (Canary)
# - 1시간: 10% 트래픽
# - 2시간: 50% 트래픽
# - 3시간: 100% 트래픽
```

### 4.2 수직 스케일링 (데이터베이스)

**데이터베이스 부하 증가**:

```bash
# PostgreSQL 메모리 증가
aws rds modify-db-instance \
  --db-instance-identifier mimir-db \
  --db-instance-class db.r5.xlarge \
  --apply-immediately

# Qdrant 용량 증가
# (수동: 클러스터 재구성 필요, 다운타임 발생)
```

---

## 5. 인시던트 대응 매뉴얼

### 5.1 일반 인시던트 대응 프로세스

**단계 1: 감지 (0~5분)**
- 모니터링 알람 또는 고객 보고
- 심각도 판정 (P1/P2/P3)
- On-call 엔지니어 호출

**단계 2: 격리 (5~15분)**
- 근본 원인 파악
- 임시 완화 (kill switch 등)
- 의사소통 시작 (고객, 팀)

**단계 3: 수정 (15~60분)**
- 코드 수정 또는 설정 변경
- 테스트 환경에서 검증
- 프로덕션 배포

**단계 4: 복구 (1~6시간)**
- 모니터링 확인
- 데이터 무결성 검증
- 기능 재개

**단계 5: 포스트모템 (24시간 이내)**
- 근본 원인 분석
- 예방책 수립
- 팀 공유

### 5.2 주요 인시던트별 대응

#### 인시던트 A: API 응답 시간 지연 (>1초)

**원인 분석**:
```bash
# 1. 로그 확인
tail -f /var/log/mimir/api.log | grep "duration_ms"

# 2. 데이터베이스 쿼리 성능
psql -U postgres -c "SELECT query, calls, mean_time FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 10;"

# 3. 벡터 검색 성능
curl http://localhost:6333/metrics | grep qdrant_search_duration_ms
```

**대응**:
- 느린 쿼리: 인덱스 추가 또는 쿼리 최적화
- DB 연결 고갈: 연결 풀 확대 또는 재시작
- 높은 부하: API 서버 추가 또는 캐시 활성화

#### 인시던트 B: 벡터 검색 실패 (500 error)

**원인 분석**:
```bash
# Qdrant 상태 확인
curl http://localhost:6333/health

# 디스크 공간 확인
df -h | grep qdrant

# 메모리 확인
free -h
```

**대응**:
- Qdrant 재시작: `systemctl restart qdrant`
- 디스크 확대: 용량 부족 시
- 인덱스 재구성: 인덱스 손상 시

#### 인시던트 C: 에이전트 무한 루프 (빠른 토큰 소진)

**대응**:
1. Kill switch 활성화 (즉시, <30초):
   ```bash
   curl -X POST http://localhost:8000/admin/kill-switch \
     -d '{"activate": true, "reason": "Agent error loop"}'
   ```
2. 에이전트 로그 분석
3. 프롬프트 또는 설정 수정
4. Kill switch 비활성화 후 재시작

---

## 6. Kill Switch 긴급 운영 (Target: <5s)

### 6.1 상황별 Kill Switch 활성화

| 상황 | 활성화 여부 | 예상 시간 |
|------|-----------|---------|
| API 오류율 >10% | 예 | 30초 |
| 에이전트 무한 루프 | 예 | 30초 |
| 보안 위협 (prompt injection) | 예 | 10초 |
| DB 다운 | 아니오 (API가 자체 오류 반환) | - |
| 정상 배포 중 (canary) | 아니오 | - |

### 6.2 Kill Switch 활성화 절차

**준비** (개발 중):
- Redis 연결 확인: `redis-cli ping` → PONG
- Kill switch 엔드포인트 접근 가능: `curl http://localhost:8000/admin/kill-switch`

**활성화** (긴급):
```bash
# Option 1: API 호출 (권장)
curl -X POST http://localhost:8000/admin/kill-switch \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{
    "activate": true,
    "reason": "Agent error loop at 2025-12-15T10:23:45Z"
  }'

# Option 2: Redis 직접 설정 (백업)
redis-cli SET "mimir:kill_switch:enabled" \
  '{"enabled":true,"timestamp":"2025-12-15T10:23:45Z"}'
```

**확인** (5초):
```bash
# 에이전트가 즉시 정지했는지 확인
curl http://localhost:8000/agent/status
# 기대: {"status": "stopped", "reason": "kill_switch_active"}
```

**비활성화** (복구 후):
```bash
curl -X POST http://localhost:8000/admin/kill-switch \
  -d '{"activate": false}'
```

---

## 7. 로그 분석 가이드

### 7.1 구조화된 로깅

Mimir는 모든 로그를 JSON 형식으로 기록:

```json
{
  "timestamp": "2025-12-15T10:23:45.123Z",
  "level": "INFO",
  "logger": "mimir.api.search",
  "message": "Search executed",
  "trace_id": "abc123def456",
  "user_id": "user-001",
  "actor_type": "user",  // "user" 또는 "agent"
  "scope_profile_id": "team-analytics",
  "duration_ms": 234,
  "query": "revenue 2025",
  "results_count": 5,
  "source": "api"
}
```

### 7.2 로그 조회

```bash
# 최근 1시간 ERROR 로그
grep '"level":"ERROR"' /var/log/mimir/api.log \
  | jq 'select(.timestamp > "'$(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%S)'Z")'

# 특정 사용자 활동
grep '"user_id":"user-001"' /var/log/mimir/api.log | jq .

# Agent 활동 추적
grep '"actor_type":"agent"' /var/log/mimir/api.log | jq 'select(.level == "ERROR")'

# 응답 시간 분석
grep '"source":"api"' /var/log/mimir/api.log \
  | jq 'select(.duration_ms > 500)' \
  | jq -s 'map(.duration_ms) | {avg: (add/length), max: max, min: min}'
```

### 7.3 감사 추적 (Audit Log)

모든 쓰기 작업 기록:

```json
{
  "timestamp": "2025-12-15T10:23:45Z",
  "action": "document_created",
  "actor_type": "agent",  // ← S2⑤: 에이전트도 기록
  "actor_id": "agent-llm-001",
  "resource_type": "document",
  "resource_id": "doc-new-001",
  "scope_profile_id": "team-analytics",
  "changes": {
    "title": "Q4 Report",
    "content": "..."
  },
  "result": "success"
}
```

**감사 로그 조회**:
```bash
grep '"action":' /var/log/mimir/audit.log \
  | grep '"actor_type":"agent"' \
  | jq 'select(.timestamp > "'$(date -d '1 day ago' -u +%Y-%m-%dT%H:%M:%S)'Z")'
```

---

## 8. 환경변수 및 기능 플래그

### 8.1 핵심 환경변수

```bash
# 데이터베이스
DATABASE_URL=postgresql://user:pass@localhost:5432/mimir_db
DATABASE_POOL_SIZE=20

# 벡터 검색
QDRANT_URL=http://localhost:6333
QDRANT_API_KEY=<optional>

# 캐시
REDIS_URL=redis://localhost:6379/0
CACHE_TTL=3600

# LLM & 평가
OPENAI_API_KEY=<optional, for cloud mode>
EVALUATION_LLM=openai  # or "local"
CLOSED_NETWORK_MODE=false  # true = disable external APIs

# 보안
JWT_SECRET=<base64-encoded-secret>
ADMIN_TOKEN=<admin-access-token>

# 로깅
LOG_LEVEL=INFO
LOG_FORMAT=json  # or "text"

# 모니터링
PROMETHEUS_ENABLED=true
PROMETHEUS_PORT=9090

# Kill switch
KILL_SWITCH_REDIS_KEY=mimir:kill_switch:enabled
```

### 8.2 기능 플래그 (Feature Flags)

```bash
# Admin UI에서 동적 활성화/비활성화

FEATURE_BATCH_SEARCH=true      # 배치 검색
FEATURE_STRUCTURED_EXTRACTION=true  # 구조 추출
FEATURE_AGENT_MCP=true         # Agent MCP 인터페이스
FEATURE_SCOPE_PROFILE_ACL=true # Scope Profile ACL
```

---

## 9. 업그레이드 절차 (Zero-Downtime 배포)

### 9.1 배포 전 준비

```bash
# 1. 현재 버전 확인
curl http://localhost:8000/version
# 응답: {"version": "2.0.0"}

# 2. 현재 상태 백업
python scripts/backup.sh

# 3. 변경사항 리뷰
cat CHANGELOG.md | head -50

# 4. 새 버전 테스트 (테스트 환경)
docker run -p 8001:8000 mimir:2.0.1
curl http://localhost:8001/health
```

### 9.2 Canary 배포 (3단계)

```bash
# Stage 1: 10% 트래픽 (1시간)
# ALB 가중치: instance-1-new (10%), instance-1 (90%)
aws elbv2 modify-target-group-attributes ...
# 모니터링: 에러율, latency, 메트릭

# Stage 2: 50% 트래픽 (1시간)
# ALB 가중치: instance-1-new (50%), instance-1 (50%)

# Stage 3: 100% 트래픽
# ALB 가중치: instance-1-new (100%), instance-1 (0%)
# → 기존 인스턴스 종료
```

### 9.3 롤백 절차

문제 발생 시 즉시 롤백:

```bash
# ALB 가중치 복원 (구 버전으로)
aws elbv2 modify-target-group-attributes \
  --target-group-arn <arn> \
  --attributes Key=deregistration_delay.timeout_seconds,Value=0

# 헬스체크 확인
curl http://localhost:8000/health

# 데이터 복구 (필요 시)
python scripts/rollback_data.py --version 2.0.0
```

---

```

**작성 절차**:
1. 아키텍처 다이어그램 포함 (ASCII art)
2. 명령어 예제 copy-paste 가능
3. 체크리스트 포함
4. 실제 운영팀이 사용할 수 있는 수준의 상세함

---

### 4.3 배포 체크리스트 작성

**파일**: `docs/개발문서/S2/프로덕션_배포_체크리스트.md`

```markdown
# 프로덕션 배포 체크리스트

## 배포 전 검증 (배포 당일 아침)

### Phase 1: 코드 및 설정 (30분)

- [ ] 배포할 브랜치가 main 또는 release 확인
- [ ] 최신 코드 pull: `git pull origin main`
- [ ] 버전 번호 업데이트 (version.py)
- [ ] CHANGELOG.md 업데이트 완료
- [ ] 환경변수 파일 검토 (prod.env)
  - [ ] DATABASE_URL 정확 (프로덕션 DB)
  - [ ] OPENAI_API_KEY / EVALUATION_LLM 선택
  - [ ] CLOSED_NETWORK_MODE 설정 확인
- [ ] 설정 파일 (config/*.yaml) 최종 검토

### Phase 2: 빌드 및 테스트 (45분)

- [ ] Docker 이미지 빌드: `docker build -t mimir:2.0.1 .`
- [ ] 이미지 스캔 (보안 취약점): `trivy image mimir:2.0.1`
  - [ ] Critical 취약점 0개
  - [ ] High 취약점 <5개 (예외 승인 필요)
- [ ] 단위 테스트: `pytest tests/ -v --cov=mimir --cov-report=html`
  - [ ] 커버리지 ≥85%
  - [ ] 모든 테스트 통과 (failed: 0)
- [ ] 통합 테스트: `pytest tests/integration/ -v`
  - [ ] API 엔드포인트 정상
  - [ ] DB 연결 정상
  - [ ] 벡터 검색 정상
- [ ] OWASP 보안 검사: `./scripts/security_scan.py`
  - [ ] 20/20 항목 통과
- [ ] 성능 벤치마크: `python scripts/benchmark.py`
  - [ ] API 응답 시간 <200ms (p99)
  - [ ] 벡터 검색 <500ms

### Phase 3: 데이터베이스 (15분)

- [ ] 마이그레이션 스크립트 검토: `alembic history`
  - [ ] 새 마이그레이션 포함
  - [ ] Rollback 가능 (down revision)
- [ ] 테스트 DB에서 마이그레이션 실행
  - [ ] 성공, 데이터 무결성 확인
  - [ ] Rollback 테스트 완료

### Phase 4: 백업 (10분)

- [ ] 전체 DB 백업: `python scripts/backup.sh full`
  - [ ] 파일 크기 >100MB (데이터 있음을 확인)
  - [ ] S3 업로드 완료: `aws s3 ls s3://mimir-backup/`
- [ ] 벡터 인덱스 스냅샷: Qdrant snapshot 생성
  - [ ] 파일 존재 확인

---

## 배포 중 모니터링 (배포 당일)

### Canary 배포 Stage 1: 10% (1시간)

**05:00 - 배포 시작**

- [ ] 새 API 인스턴스 프로비저닝
- [ ] 헬스체크: `curl http://new-instance:8000/health`
- [ ] ALB 가중치 설정: 10% → new, 90% → old
- [ ] 로그 모니터링 시작

**모니터링 (5분 간격)**:

```bash
# 에러율 확인
watch -n 5 'curl -s http://localhost:8000/metrics | grep http_requests_total'

# 응답 시간 확인
tail -f /var/log/mimir/api.log | grep '"duration_ms"'

# 에이전트 활동 확인
grep '"actor_type":"agent"' /var/log/mimir/api.log | jq '.level' | sort | uniq -c
```

**중단 기준** (즉시 롤백):
- 에러율 >2% (정상: <0.5%)
- P99 응답 시간 >500ms (정상: <200ms)
- 데이터 무결성 이슈 발생

**06:00 - Stage 1 완료 검증**:
- [ ] 에러 로그 분석: `grep ERROR /var/log/mimir/api.log | wc -l` <50건
- [ ] 메트릭 정상: 수집 데이터 정상
- [ ] 고객 피드백: 문제 보고 없음

---

### Canary 배포 Stage 2: 50% (1시간)

**06:00 - Stage 2 시작**

- [ ] ALB 가중치: 50% → new, 50% → old
- [ ] 모니터링 계속

**06:30 - 중간 검증**:
- [ ] 에러율 <1%
- [ ] P99 <250ms
- [ ] Kill switch 테스트: 정상 작동 확인

**07:00 - Stage 2 완료**:
- [ ] 위와 동일한 검증

---

### Canary 배포 Stage 3: 100% (즉시)

**07:00 - Stage 3 시작**

- [ ] ALB 가중치: 100% → new, 0% → old
- [ ] 기존 인스턴스 점진적 종료 (30초 interval)

**07:15 - 최종 검증**:

```bash
# 모든 트래픽이 새 인스턴스로 라우팅 확인
curl -I http://localhost:8000/health

# 데이터 일관성 확인
python scripts/data_integrity_check.py

# 에이전트 기능 테스트
python scripts/test_agent_mcp.py
```

---

## 배포 후 검증 (배포 후 24시간)

### Phase 1: 즉시 검증 (배포 후 1시간)

- [ ] 헬스체크 정상: `curl http://localhost:8000/health/detailed`
- [ ] 로그 확인: 에러 없음
- [ ] 메트릭 수집: Prometheus 정상
- [ ] 에이전트 기능: MCP 도구 호출 정상
- [ ] 사용자 보고사항: 장애 없음

### Phase 2: 단기 모니터링 (배포 후 4시간)

**메트릭 추적**:

| 메트릭 | 정상 | 경고 | 조치 |
|--------|------|------|------|
| API error rate | <0.5% | 0.5~2% | 로그 분석 |
| P99 latency | <200ms | 200~500ms | 캐시 확인 |
| DB connections | <50% | 50~80% | 풀 확대 |
| Memory | <60% | 60~80% | 인스턴스 확대 |

- [ ] Slack 알람: 비정상 메트릭 없음
- [ ] PagerDuty: 심각도 높은 알람 없음

### Phase 3: 중기 모니터링 (배포 후 24시간)

- [ ] 24시간 로그 분석:
  ```bash
  python scripts/analyze_logs.py --duration 24h --level ERROR
  ```
- [ ] AI 품질 메트릭:
  - [ ] Faithfulness ≥0.80 (이전과 비교 <2% 변화)
  - [ ] Hallucination rate ≤0.10 (이전과 비교 <2% 변화)
  - [ ] Citation rate ≥0.90
- [ ] 사용자 피드백: 수집 및 분류
- [ ] 성능 비교: 배포 전후 비교
  - [ ] 응답 시간 변화 <5%
  - [ ] 에러율 변화 <0.2%

---

## 긴급 롤백 절차

**롤백 결정 기준**:
- 에러율 >5%
- P99 응답 시간 >1초
- 데이터 손상 발생
- 중대 보안 취약점 발견

**롤백 실행** (5분 이내):

```bash
# 1. ALB 가중치 복원
aws elbv2 modify-target-group-attributes \
  --target-group-arn <arn> \
  --attributes Key=active.deregistration_delay.timeout_seconds,Value=0

# 2. 새 인스턴스 종료
docker stop mimir-new-001 mimir-new-002 mimir-new-003

# 3. 헬스체크 확인
curl http://localhost:8000/health

# 4. 데이터 복구 (필요 시)
python scripts/rollback_data.py --version 2.0.0

# 5. 검증
python scripts/smoke_test.py
```

---

## 배포 완료 체크리스트

배포 당일 완료 후:

- [ ] 배포 완료 공지 (팀, 고객)
- [ ] 배포 기록 업데이트
  ```bash
  git tag -a v2.0.1 -m "Production deployment 2025-12-15"
  ```
- [ ] 포스트모템 예약 (배포 후 1주일)
- [ ] 릴리스 노트 공개
- [ ] 모니터링 알람 정상화

---

**배포 담당자**: _____________________________
**배포 시간**: _____________ ~ _____________
**배포 결과**: [ ] 성공 [ ] 부분 성공 [ ] 롤백
**검증자**: _____________________________

```

---

### 4.4 OpenAPI 스펙 자동 생성 스크립트

**파일**: `scripts/generate_openapi_spec.py`

```python
"""
OpenAPI 3.0 스펙 자동 생성
FastAPI 엔드포인트에서 스키마 추출 및 Swagger 문서 생성
"""

import json
from fastapi.openapi.utils import get_openapi
from mimir.server import app

def generate_spec():
    """OpenAPI 스펙 생성"""
    openapi_schema = get_openapi(
        title="Mimir RAG Platform",
        version="2.0.1",
        description="Grounded RAG + MCP + Scope Profile ACL",
        routes=app.routes,
    )
    
    # 커스텀 정보 추가
    openapi_schema["info"]["x-logo"] = {
        "url": "https://mimir.example.com/logo.png"
    }
    
    # Scope Profile 설명 추가
    for path, methods in openapi_schema["paths"].items():
        for method, details in methods.items():
            if method == "parameters" not in details:
                details["parameters"] = []
            
            # 모든 API에 scope 매개변수 추가 (query)
            if not any(p.get("name") == "scope" for p in details.get("parameters", [])):
                details["parameters"].append({
                    "name": "scope",
                    "in": "query",
                    "required": False,
                    "schema": {"type": "string"},
                    "description": "Scope Profile ID (ACL 필터링)"
                })
    
    return openapi_schema

def save_spec(spec: dict, output_file: str = "openapi.json"):
    """스펙 저장"""
    with open(output_file, 'w') as f:
        json.dump(spec, f, indent=2)
    print(f"✓ OpenAPI 스펙 저장: {output_file}")

def generate_swagger_ui(spec: dict, output_dir: str = "docs/swagger"):
    """Swagger UI HTML 생성"""
    import os
    os.makedirs(output_dir, exist_ok=True)
    
    html_content = f"""
    <!DOCTYPE html>
    <html>
    <head>
        <title>Mimir API</title>
        <meta charset="utf-8"/>
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Montserrat:300,400,700|Roboto:300,400,700">
        <style>
            body {{
                margin: 0;
                padding: 0;
            }}
        </style>
    </head>
    <body>
        <redoc spec-url='openapi.json'></redoc>
        <script src="https://cdn.jsdelivr.net/npm/redoc@latest/bundles/redoc.standalone.js"></script>
    </body>
    </html>
    """
    
    with open(f"{output_dir}/index.html", 'w') as f:
        f.write(html_content)
    
    with open(f"{output_dir}/openapi.json", 'w') as f:
        json.dump(spec, f, indent=2)
    
    print(f"✓ Swagger UI 생성: {output_dir}/index.html")

def generate_client_code(spec: dict, language: str = "python"):
    """클라이언트 코드 생성 (OpenAPI Generator 사용)"""
    import subprocess
    
    if language == "python":
        subprocess.run([
            "openapi-generator-cli", "generate",
            "-i", "openapi.json",
            "-g", "python",
            "-o", "clients/python",
            "--additional-properties=packageName=mimir_client"
        ])
    elif language == "typescript":
        subprocess.run([
            "openapi-generator-cli", "generate",
            "-i", "openapi.json",
            "-g", "typescript-axios",
            "-o", "clients/typescript"
        ])
    
    print(f"✓ {language.upper()} 클라이언트 생성")

if __name__ == "__main__":
    spec = generate_spec()
    save_spec(spec)
    generate_swagger_ui(spec)
    generate_client_code(spec, "python")
    generate_client_code(spec, "typescript")
```

**실행**:
```bash
python scripts/generate_openapi_spec.py
# 생성 파일:
# - docs/swagger/openapi.json
# - docs/swagger/index.html
# - clients/python/ (OpenAPI 클라이언트)
# - clients/typescript/ (OpenAPI 클라이언트)
```

---

## 5. 산출물

1. **챗봇 인계 메모 v3** (`docs/개발문서/S2/챗봇_인계메모_v3.md`)
   - 크기: ~35KB
   - 내용: v1~v2 요약 + v3 신규 섹션 (Scope Profile, Kill switch, 성능최적화, 폐쇄망)
   - 대상: 운영팀, 파트너팀

2. **운영 매뉴얼** (`docs/개발문서/S2/운영_매뉴얼.md`)
   - 크기: ~50KB
   - 내용: 아키텍처, 헬스체크, 백업/복구, 스케일링, 인시던트, 로그 분석, 배포
   - 대상: DevOps팀, 운영팀

3. **배포 체크리스트** (`docs/개발문서/S2/프로덕션_배포_체크리스트.md`)
   - 크기: ~20KB
   - 내용: 배포 전/중/후 검증 항목, 롤백 절차
   - 대상: Release 매니저, DevOps팀

4. **OpenAPI 스펙** (`scripts/generate_openapi_spec.py`)
   - 크기: ~5KB (스크립트)
   - 출력: `openapi.json`, `swagger/index.html`
   - 생성: 배포 시마다 자동

---

## 6. 완료 기준

1. ✓ 챗봇 인계 메모 v3 작성 완료
   - v1, v2 요약 포함
   - v3 신규 섹션 5개 이상

2. ✓ 운영 매뉴얼 작성 완료
   - 9개 섹션 (아키텍처~배포)
   - 실전 명령어 포함

3. ✓ 배포 체크리스트 작성 완료
   - 배포 전/중/후 항목 포함
   - 긴급 롤백 절차 포함

4. ✓ OpenAPI 스펙 생성 스크립트 작성 완료
   - FastAPI 자동 추출
   - Swagger UI 생성

5. ✓ 모든 문서 최종 검수 완료
   - 링크 무결성 확인
   - 명령어 실행 검증

---

## 7. 작업 지침

### 지침 7-1: 인계 메모 v3의 신규 섹션 균형

v3 신규 섹션 (Scope Profile, Kill switch, 성능, 폐쇄망)은
실전 운영팀이 즉시 사용 가능하도록 작성:

- **이론**: 간단히 (1~2 문단)
- **실전 예제**: 충분히 (코드, 명령어)
- **체크리스트**: 모든 설정에 포함

---

### 지침 7-2: 운영 매뉴얼의 명령어 정확성

모든 bash 명령어는 실제 프로덕션 환경에서 테스트:

```bash
# 배포 전
./scripts/validate_manual.sh  # 매뉴얼의 모든 명령어 검증
```

실패하는 명령어는 수정하거나 주석 추가

---

### 지침 7-3: Kill Switch <5s 목표 검증

매뉴얼 작성 후 Kill switch 성능 테스트 수행:

```bash
python scripts/test_kill_switch.py \
  --samples 10 \
  --target-response-time 5000  # ms

# 기대: 모든 테스트 <5초 통과
```

---

### 지침 7-4: 배포 체크리스트의 자동화

체크리스트 항목 중 자동화 가능한 것은 스크립트로 구현:

```bash
# 배포 전 자동 검증
python scripts/pre_deployment_check.py
  ✓ 코드 빌드
  ✓ 테스트 실행
  ✓ 보안 검사
  ✓ 백업 확인
  → 모든 항목 통과 시만 배포 진행 가능
```

---

