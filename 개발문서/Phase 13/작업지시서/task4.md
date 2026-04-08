# Task 13-4. 테스트 전략 및 커버리지

## 1. 작업 목적

Mimir 플랫폼 전체에 대한 테스트 전략을 수립하고 커버리지 목표를 달성하며 CI에 자동화한다.

이 작업의 목표는 다음과 같다.

- 테스트 계층 전략 수립 (단위/통합/E2E)
- 단위 테스트 커버리지 80% 이상 달성
- 핵심 API 엔드포인트 통합 테스트 100%
- E2E 테스트 핵심 시나리오 구현
- CI 단계 자동 실행 설정

---

## 2. 작업 범위

### 포함 범위

- 테스트 전략 문서 수립
- 커버리지 측정 및 미달 영역 보완
- E2E 테스트 시나리오 구현 (Playwright)
- 부하 테스트 기초 설정 (Locust)
- CI 테스트 자동화

### 제외 범위

- 성능 최적화 (Task 13-8에서 다룸)
- 보안 테스트 (Task 13-1에서 다룸)

---

## 3. 주요 구현 대상

### 3-1. 테스트 계층 전략

```
E2E 테스트 (Playwright)
  └── 핵심 사용자 시나리오 10개 이상
      속도 느림, CI 야간 실행

통합 테스트 (pytest + 실제 DB)
  └── 모든 API 엔드포인트
      실제 PostgreSQL (pgvector 포함)
      Mock LLM / 실제 임베딩 (선택)

단위 테스트 (pytest / Jest)
  └── 서비스 레이어, 유틸리티, 플러그인
      Mock DB, Mock 외부 API
      빠른 실행 (< 2분)
```

---

### 3-2. 커버리지 목표

| 영역 | 커버리지 목표 | 측정 방법 |
|------|------------|---------|
| Backend 전체 | 80% | pytest-cov |
| 핵심 서비스 레이어 | 90% | 별도 측정 |
| API 엔드포인트 | 100% (통합 테스트 기준) | 수동 확인 |
| Frontend (TypeScript) | 70% | Jest --coverage |
| 플러그인 (Phase 12) | 100% PluginConformanceTest | 별도 측정 |

---

### 3-3. E2E 테스트 핵심 시나리오 (Playwright)

| # | 시나리오 | 검증 내용 |
|---|---------|---------|
| 1 | 로그인 | 토큰 발급, 대시보드 이동 |
| 2 | 문서 생성 및 편집 | 노드 추가, 저장 |
| 3 | 문서 워크플로 | Draft → Review → Published |
| 4 | 문서 검색 | 키워드 검색, 결과 표시 |
| 5 | RAG 질의 | 질문 입력, 스트리밍 응답, 출처 확인 |
| 6 | Admin 사용자 관리 | 사용자 생성, 역할 할당 |
| 7 | Admin DocumentType 설정 | 청킹 설정 변경 저장 |
| 8 | 버전 비교 (Phase 9) | 두 버전 diff 표시 |
| 9 | 권한 격리 | 권한 없는 문서 접근 불가 |
| 10 | 대화 이력 | 이전 대화 이어서 질의 |

---

### 3-4. 부하 테스트 시나리오 (Locust)

```python
# locustfile.py

class MimirUser(HttpUser):
    wait_time = between(1, 3)

    @task(5)
    def search_documents(self):
        self.client.post("/api/search", json={"text": "정보보안 정책"})

    @task(3)
    def view_document(self):
        self.client.get(f"/api/documents/{self.document_id}")

    @task(2)
    def rag_query(self):
        self.client.post("/api/rag/query", json={
            "question": "접근통제 기준은?",
            "stream": False,
        })

# 목표: 50 동시 사용자에서 API P95 < 500ms
```

---

### 3-5. CI 테스트 자동화

```yaml
# .github/workflows/test.yml

jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - pytest tests/unit/ --cov=app --cov-report=xml
      - codecov upload coverage.xml

  integration-test:
    services:
      postgres:
        image: pgvector/pgvector:pg16
    steps:
      - pytest tests/integration/ -v

  e2e-test:
    if: github.ref == 'refs/heads/main'
    steps:
      - npx playwright test
```

---

## 4. 산출물

1. 테스트 전략 문서
2. 커버리지 측정 결과 및 보완 테스트
3. E2E 테스트 시나리오 구현 (Playwright)
4. 부하 테스트 설정 (Locust)
5. CI 테스트 자동화 설정

---

## 5. 완료 기준

- Backend 단위 테스트 커버리지 80% 이상 달성
- 핵심 API 엔드포인트 통합 테스트 100% 작성
- E2E 핵심 시나리오 10개가 통과한다
- PR 생성 시 단위/통합 테스트가 자동 실행된다

---

## 6. Codex 작업 지침

- 커버리지 80% 미달 영역은 핵심 서비스 레이어 중심으로 보완한다
- E2E 테스트는 실제 외부 API(OpenAI) 대신 Mock 서버를 사용하여 비용 없이 실행한다
- 부하 테스트는 스테이징 환경에서만 실행하며 운영 환경에서는 금지한다
- 통합 테스트는 각 테스트 후 DB 롤백 처리하여 독립성을 보장한다
