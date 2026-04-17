# Task 7-12: PH5-CARRY-003 제안 시스템 부하 테스트

> **이월 출처**: Phase 5 FG5.2 검수보고서 §이월 항목 PH5-CARRY-003  
> **이월 경로**: Phase 5 → Phase 6 미처리 → Phase 7 처리  
> **실행 시점**: 배포 직전 스테이징 환경에서 실행

## 1. 작업 목적

배포 전 제안(Proposal) 시스템이 1,000개 이상 동시 요청을 안정적으로 처리할 수 있는지 검증한다. P95 응답시간 ≤ 2s, 에러율 ≤ 1% 기준을 충족해야 운영 배포를 진행한다.

## 2. 작업 범위

### 포함 범위

1. **부하 테스트 스크립트** (`scripts/load_test_proposals.py`)
   - Locust 기반 (또는 k6)
   - 시나리오: 1,000개 가상 사용자 동시 제안 제출
   - 측정 지표: P50, P95, P99 응답시간, 에러율, 처리량(RPS)

2. **테스트 데이터 생성 스크립트** (`scripts/seed_load_test_data.py`)
   - 부하 테스트용 에이전트 계정 및 문서 시드 데이터 생성
   - 테스트 완료 후 정리(cleanup) 기능 포함

3. **결과 리포트 템플릿** (`docs/부하테스트결과.md`)
   - 실행 결과 자동 채워넣기 형식 정의
   - 통과/실패 판정 기준 명시

4. **실행 가이드** (이 문서 §7)

### 제외 범위

- CI 자동 실행 — 수동 배포 직전에만 실행
- 프로덕션 환경 실행 — 스테이징 전용

## 3. 선행 조건

- Phase 5 FG5.2: 제안 제출/승인/거절 API 완료
- Phase 6 FG6.2: AdminProposalsPage + 배치 처리 API 완료
- 스테이징 환경: Docker Compose 또는 K8s 스테이징 클러스터
- `locust` 패키지 설치: `pip install locust`

## 4. 주요 작업 항목

### 4-1. Locust 부하 테스트 스크립트

**파일:** `scripts/load_test_proposals.py`

```python
"""
Mimir 제안 시스템 부하 테스트 (PH5-CARRY-003).

실행 방법:
  locust -f scripts/load_test_proposals.py \
    --host http://localhost:8000 \
    --users 1000 \
    --spawn-rate 50 \
    --run-time 5m \
    --html reports/load_test_$(date +%Y%m%d_%H%M%S).html
"""

import json
import random
import string
from locust import HttpUser, task, between, events


def random_content(length: int = 200) -> str:
    """부하 테스트용 랜덤 텍스트 생성."""
    return "".join(random.choices(string.ascii_letters + " \n", k=length))


class ProposalUser(HttpUser):
    """에이전트가 제안을 제출하는 시나리오."""

    wait_time = between(0.1, 0.5)  # 요청 간 0.1~0.5초 대기

    def on_start(self):
        """테스트 시작 시 API Key 획득."""
        # 미리 생성된 테스트 에이전트 API Key 풀에서 랜덤 선택
        self.api_key = f"mim_sk_test_{random.randint(1, 100):04d}"
        self.headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json",
        }
        # 제안 대상 문서 ID 목록 (시드 데이터에서 생성된 것)
        self.document_ids = [f"load-test-doc-{i:04d}" for i in range(1, 101)]

    @task(5)
    def submit_proposal(self):
        """제안 제출 (주요 시나리오, 가중치 5)."""
        doc_id = random.choice(self.document_ids)
        payload = {
            "document_id": doc_id,
            "proposed_content": random_content(500),
            "reason": f"부하 테스트 제안 {random.randint(1, 9999)}",
        }

        with self.client.post(
            "/api/v1/mcp/tools/call",
            json={"name": "propose_document_change", "arguments": payload},
            headers=self.headers,
            catch_response=True,
            name="submit_proposal",
        ) as response:
            if response.status_code == 429:
                response.success()  # Rate Limit은 정상 응답으로 처리
            elif response.status_code not in (200, 201):
                response.failure(f"Unexpected status: {response.status_code}")

    @task(2)
    def list_proposals(self):
        """제안 목록 조회 (가중치 2)."""
        with self.client.get(
            "/api/v1/proposals",
            headers=self.headers,
            params={"page": 1, "page_size": 20},
            catch_response=True,
            name="list_proposals",
        ) as response:
            if response.status_code != 200:
                response.failure(f"Unexpected status: {response.status_code}")


class AdminUser(HttpUser):
    """관리자가 제안을 일괄 처리하는 시나리오."""

    wait_time = between(1, 3)
    weight = 1  # 에이전트 1000명당 관리자 1명 비율

    def on_start(self):
        """관리자 인증."""
        self.headers = {
            "Authorization": "Bearer admin-test-token",
            "Content-Type": "application/json",
        }

    @task(1)
    def batch_approve_proposals(self):
        """대기 중 제안 일괄 승인."""
        # 먼저 대기 중 제안 ID 목록 조회
        list_resp = self.client.get(
            "/api/v1/admin/proposals",
            headers=self.headers,
            params={"status": "pending", "page_size": 50},
        )
        if list_resp.status_code != 200:
            return

        proposals = list_resp.json().get("items", [])
        if not proposals:
            return

        ids = [p["id"] for p in proposals[:10]]  # 최대 10개씩 일괄 처리

        with self.client.post(
            "/api/v1/admin/proposals/batch-approve",
            json={"proposal_ids": ids},
            headers=self.headers,
            catch_response=True,
            name="batch_approve",
        ) as response:
            if response.status_code not in (200, 207):
                response.failure(f"Unexpected status: {response.status_code}")


@events.quitting.add_listener
def on_quitting(environment, **kwargs):
    """테스트 완료 후 결과 요약 출력."""
    stats = environment.runner.stats
    total = stats.total

    print("\n" + "=" * 60)
    print("부하 테스트 결과 요약")
    print("=" * 60)
    print(f"총 요청: {total.num_requests:,}")
    print(f"실패 요청: {total.num_failures:,}")
    print(f"에러율: {total.fail_ratio:.2%}")
    print(f"P50 응답시간: {total.get_response_time_percentile(0.5):.0f}ms")
    print(f"P95 응답시간: {total.get_response_time_percentile(0.95):.0f}ms")
    print(f"P99 응답시간: {total.get_response_time_percentile(0.99):.0f}ms")
    print(f"처리량(RPS): {total.current_rps:.1f}")

    # 통과 기준 판정
    p95 = total.get_response_time_percentile(0.95)
    error_rate = total.fail_ratio

    print("\n판정:")
    if p95 <= 2000 and error_rate <= 0.01:
        print("✅ PASS — P95 ≤ 2s, 에러율 ≤ 1%")
        environment.process_exit_code = 0
    else:
        reasons = []
        if p95 > 2000:
            reasons.append(f"P95 {p95:.0f}ms > 2000ms")
        if error_rate > 0.01:
            reasons.append(f"에러율 {error_rate:.2%} > 1%")
        print(f"❌ FAIL — {', '.join(reasons)}")
        environment.process_exit_code = 1
```

### 4-2. 시드 데이터 스크립트

**파일:** `scripts/seed_load_test_data.py`

```python
"""
부하 테스트용 시드 데이터 생성/정리 스크립트.

실행:
  python scripts/seed_load_test_data.py --action seed --backend-url http://localhost:8000
  python scripts/seed_load_test_data.py --action cleanup --backend-url http://localhost:8000
"""

import argparse
import httpx
import asyncio


async def seed(backend_url: str, admin_token: str):
    """테스트용 문서 100개 + 에이전트 100개 생성."""
    async with httpx.AsyncClient(base_url=backend_url) as client:
        headers = {"Authorization": f"Bearer {admin_token}"}

        # 문서 100개 생성
        print("문서 시드 데이터 생성 중...")
        for i in range(1, 101):
            await client.post(
                "/api/v1/admin/documents",
                json={
                    "id": f"load-test-doc-{i:04d}",
                    "title": f"부하 테스트 문서 {i}",
                    "content": f"이 문서는 부하 테스트를 위해 생성되었습니다. 문서 번호: {i}",
                    "type_code": "general",
                },
                headers=headers,
            )

        # 에이전트 API Key 100개 생성
        print("에이전트 시드 데이터 생성 중...")
        for i in range(1, 101):
            await client.post(
                "/api/v1/admin/agents",
                json={
                    "name": f"load-test-agent-{i:04d}",
                    "api_key": f"mim_sk_test_{i:04d}",
                    "expires_at": "2099-12-31T00:00:00Z",
                    "scope_profile_id": "default-scope",
                },
                headers=headers,
            )

        print(f"시드 완료: 문서 100개, 에이전트 100개")


async def cleanup(backend_url: str, admin_token: str):
    """테스트 데이터 정리."""
    async with httpx.AsyncClient(base_url=backend_url) as client:
        headers = {"Authorization": f"Bearer {admin_token}"}

        for i in range(1, 101):
            await client.delete(f"/api/v1/admin/documents/load-test-doc-{i:04d}", headers=headers)
            await client.delete(f"/api/v1/admin/agents/load-test-agent-{i:04d}", headers=headers)

        print("테스트 데이터 정리 완료")


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--action", choices=["seed", "cleanup"], required=True)
    parser.add_argument("--backend-url", default="http://localhost:8000")
    parser.add_argument("--admin-token", default="admin-test-token")
    args = parser.parse_args()

    if args.action == "seed":
        asyncio.run(seed(args.backend_url, args.admin_token))
    else:
        asyncio.run(cleanup(args.backend_url, args.admin_token))
```

### 4-3. 결과 리포트 템플릿

**파일:** `docs/부하테스트결과_템플릿.md`

```markdown
# 제안 시스템 부하 테스트 결과

**테스트 일시**: {datetime}
**환경**: 스테이징 ({backend_url})
**테스트 시간**: {run_time}
**동시 사용자**: {users}명

## 결과 요약

| 지표 | 측정값 | 기준 | 판정 |
|------|--------|------|------|
| P95 응답시간 | {p95}ms | ≤ 2,000ms | {p95_pass} |
| P99 응답시간 | {p99}ms | — | — |
| 에러율 | {error_rate}% | ≤ 1% | {error_pass} |
| 처리량 | {rps} RPS | — | — |
| 총 요청 | {total_requests}건 | — | — |

## 엔드포인트별 결과

| 엔드포인트 | P50 | P95 | 에러율 |
|-----------|-----|-----|--------|
| submit_proposal | {p50_submit}ms | {p95_submit}ms | {err_submit}% |
| list_proposals | {p50_list}ms | {p95_list}ms | {err_list}% |
| batch_approve | {p50_batch}ms | {p95_batch}ms | {err_batch}% |

## 판정

**{PASS/FAIL}** — {reason}

## 비고

{notes}
```

## 5. 실행 절차

```bash
# 1. 스테이징 환경 시작
docker-compose -f docker-compose.staging.yml up -d
sleep 10

# 2. 시드 데이터 생성
python scripts/seed_load_test_data.py --action seed --backend-url http://staging.mimir.internal

# 3. 부하 테스트 실행 (headless 모드)
locust -f scripts/load_test_proposals.py \
  --host http://staging.mimir.internal \
  --users 1000 \
  --spawn-rate 50 \
  --run-time 5m \
  --headless \
  --html reports/load_test_$(date +%Y%m%d_%H%M%S).html

# 4. 결과 확인 (exit code: 0=PASS, 1=FAIL)
echo "테스트 결과: $?"

# 5. 시드 데이터 정리
python scripts/seed_load_test_data.py --action cleanup --backend-url http://staging.mimir.internal
```

## 6. 산출물

1. **부하 테스트 스크립트** (`scripts/load_test_proposals.py`)
2. **시드 데이터 스크립트** (`scripts/seed_load_test_data.py`)
3. **결과 리포트 템플릿** (`docs/부하테스트결과_템플릿.md`)
4. **실제 테스트 결과 리포트** (`reports/load_test_YYYYMMDD_HHMMSS.html`)

## 7. 완료 기준

1. 스크립트가 1,000명 동시 사용자로 5분간 안정 실행되는가?
2. P95 응답시간 ≤ 2,000ms 달성하는가?
3. 에러율 ≤ 1% 달성하는가?
4. 시드 데이터 생성 및 정리 스크립트가 정상 동작하는가?
5. HTML 결과 리포트가 자동 생성되는가?
6. FAIL 시 exit code 1을 반환하여 CI/CD 파이프라인이 중단 가능한가?

## 8. 작업 지침

### 지침 7-12-1. Rate Limit 제외
테스트 중 429 응답은 정상 동작이므로 에러로 집계하지 않는다. Locust에서 `response.success()`로 처리.

### 지침 7-12-2. 스테이징 전용
프로덕션 데이터베이스/환경에서는 절대 실행하지 않는다. 스크립트 상단에 환경 확인 로직 추가 권장.

### 지침 7-12-3. 실패 시 대응
FAIL 판정 시:
- P95 초과: 인덱스 점검, 쿼리 최적화, 커넥션 풀 증설 고려
- 에러율 초과: 로그에서 500/503 원인 분석, Rate Limit 설정 검토
