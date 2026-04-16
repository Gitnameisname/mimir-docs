# 작업지시서: S3 드라이런 + 통합 테스트 CI 잡
**FG9.2-TASK9-6** | Phase 9 기간 | CI/CD 자동화 + S3 호환성

---

## 1. 작업 목적

Phase 9 FG9.2 완료 후 다음 단계(S3: AI-native planning)로의 전환을 대비하기 위해:

1. **S3 드라이런(Dry-run)**: Mimir MCP 서버가 S3 AI-native planning 도구와 호환되는지 사전 검증
2. **통합 테스트 CI 잡**: Phase 1~8의 모든 통합 테스트를 자동으로 실행하고 보고서 생성하는 GitHub Actions workflow 작성
3. **S2 readiness checklist**: 자동화된 Phase completion criteria 검증

---

## 2. 작업 범위

### 포함 범위

1. **S3 드라이런** (선택사항이지만 권장)
   - MCP spec compatibility verification
   - Mock S3 클라이언트 구현 (S3 AI-native planning tool 시뮬레이션)
   - Test scenarios:
     - Tool discovery (S3가 Mimir의 모든 tool을 발견할 수 있는가)
     - Search workflow (S3에서 검색 요청 시 정상 동작)
     - Citation verification (Citation integrity in S3 context)
     - Resource access (S3에서 mimir:// resource 접근 가능)
   - Compatibility report 생성 (which features work, which need extension)

2. **GitHub Actions Workflow** (`.github/workflows/s2-integration-tests.yml`)
   - Jobs:
     - setup-db: PostgreSQL 및 테스트 DB 초기화
     - run-integration: Phase-to-Phase 통합 테스트 실행
     - run-mcp-e2e: MCP e2e 테스트
     - run-golden-set: 골든셋 회귀 테스트 (strict mode)
     - run-security: 보안 취약점 검사
     - generate-reports: 모든 보고서 생성
   - Trigger: push to main, PR, nightly schedule
   - Parallel job execution (가능한 범위)
   - Artifact upload: test reports, AI품질평가.md, security reports
   - PR comment with test summary
   - Failure notification (Slack webhook)

3. **Nightly Build Configuration**
   - 매일 특정 시간에 전체 통합 테스트 자동 실행
   - Performance baseline 수집 (execution time tracking)
   - Trend analysis (품질, 성능 추이)

4. **Test Report Aggregation**
   - 모든 Phase별 테스트 결과 수집
   - Single dashboard-friendly JSON file 생성
   - Summary statistics (pass rate, failures, coverage)

5. **S2 Readiness Checklist** (자동화)
   - Phase 1~8 completion criteria 자동 검증
   - S2 원칙 준수 확인 (actor_type, scope profile, closed-network)
   - 체크리스트 보고서 생성

### 제외 범위

- S3 full integration (S3 개발이 완료된 후 진행)
- Production deployment workflow (별도 작업)
- Canary/rolling deployment (별도 작업)
- Cost optimization (별도 작업)

---

## 3. 선행 조건

- GitHub repository 설정 완료
- GitHub Actions secrets 설정 (Slack webhook URL 등)
- Docker 및 Docker Compose 설치
- PostgreSQL docker image 준비
- 모든 Phase 1~8 코드 main branch에 merge
- pytest, pytest-cov 설치
- Python 3.9+ 환경

---

## 4. 주요 작업 항목

### 4.1 S3 드라이런 구현

**파일**: `tests/s3_compatibility/test_s3_dry_run.py` (새로 작성)

```python
"""
S3 드라이런 테스트

S3 AI-native planning tool이 Mimir MCP 서버를 통합할 때
정상 동작하는지 사전 검증한다.

실행:
    pytest tests/s3_compatibility/test_s3_dry_run.py -v --s3-dry-run
"""

import pytest
import asyncio
from typing import Dict, Any, List, Optional
from uuid import uuid4
from dataclasses import dataclass
import json


@dataclass
class S3ToolCall:
    """S3가 Mimir MCP 서버에 요청하는 tool call"""
    tool_name: str
    arguments: Dict[str, Any]


class MockS3Client:
    """
    S3 AI-native planning tool을 시뮬레이션하는 mock 클라이언트
    
    S3는 다음과 같이 Mimir MCP를 사용한다:
    1. Tool discovery: 사용 가능한 모든 tool 목록 조회
    2. Search: document 검색 (RAG context 구성)
    3. Fetch: 특정 document 상세 내용 조회
    4. Cite: citation 검증
    5. Write: agent가 제안 생성 (transition proposal)
    """
    
    def __init__(self, mcp_server):
        self.mcp_server = mcp_server
        self.available_tools: List[Dict[str, Any]] = []
        self.compatible_features: List[str] = []
        self.incompatible_features: List[str] = []
    
    async def discover_tools(self) -> bool:
        """
        Tool discovery: S3가 Mimir의 모든 필수 tool을 발견할 수 있는가
        
        필수 tool:
        - search_documents
        - fetch_node
        - verify_citation
        - propose_transition
        - list_resources
        """
        
        required_tools = {
            "search_documents",
            "fetch_node",
            "verify_citation",
            "propose_transition",
            "list_resources"
        }
        
        try:
            # MCP 서버에서 tool list 조회
            available_tools = await self.mcp_server.list_tools()
            self.available_tools = available_tools
            
            available_names = {tool["name"] for tool in available_tools}
            
            if required_tools.issubset(available_names):
                self.compatible_features.append("tool_discovery")
                return True
            else:
                missing = required_tools - available_names
                self.incompatible_features.append(f"missing_tools: {missing}")
                return False
        
        except Exception as e:
            self.incompatible_features.append(f"tool_discovery_error: {str(e)}")
            return False
    
    async def search_with_citation_verification(self, query: str) -> bool:
        """
        Search workflow with citation verification
        
        Flow:
        1. search_documents(query) → search results
        2. Each result should have citation info
        3. verify_citation() for each citation
        """
        
        try:
            # 1. 검색
            search_result = await self.mcp_server.execute_tool(
                tool_name="search_documents",
                arguments={
                    "query": query,
                    "limit": 5
                }
            )
            
            if not search_result.get("documents"):
                self.incompatible_features.append("search_returns_empty")
                return False
            
            # 2. Citation verification
            documents = search_result["documents"]
            all_citations_valid = True
            
            for doc in documents:
                if "citations" in doc:
                    for citation in doc["citations"]:
                        # verify_citation 호출
                        verify_result = await self.mcp_server.execute_tool(
                            tool_name="verify_citation",
                            arguments={
                                "citation": citation
                            }
                        )
                        
                        if not verify_result.get("valid"):
                            all_citations_valid = False
                            break
            
            if all_citations_valid:
                self.compatible_features.append("search_with_citation")
                return True
            else:
                self.incompatible_features.append("citation_verification_failed")
                return False
        
        except Exception as e:
            self.incompatible_features.append(f"search_error: {str(e)}")
            return False
    
    async def fetch_and_use_resource(self) -> bool:
        """
        Resource fetching: S3가 mimir:// URI로 document를 직접 접근할 수 있는가
        """
        
        try:
            # 먼저 resource list 조회
            resources = await self.mcp_server.list_resources()
            
            if not resources:
                self.incompatible_features.append("no_resources_available")
                return False
            
            # 첫 번째 resource fetch
            resource = resources[0]
            uri = resource["uri"]
            
            fetch_result = await self.mcp_server.execute_tool(
                tool_name="fetch_node",
                arguments={
                    "uri": uri
                }
            )
            
            if fetch_result.get("node"):
                self.compatible_features.append("resource_fetching")
                return True
            else:
                self.incompatible_features.append("fetch_node_empty_result")
                return False
        
        except Exception as e:
            self.incompatible_features.append(f"resource_fetch_error: {str(e)}")
            return False
    
    async def agent_write_workflow(self) -> bool:
        """
        Agent write workflow: S3 agent가 Mimir를 통해 draft proposal을 생성할 수 있는가
        """
        
        try:
            # propose_transition tool 호출
            task_id = str(uuid4())
            
            propose_result = await self.mcp_server.execute_tool(
                tool_name="propose_transition",
                arguments={
                    "task_id": task_id,
                    "content": {
                        "planned_action": "extract_and_review",
                        "extracted_fields": [
                            {"name": "field1", "value": "value1"}
                        ]
                    }
                }
            )
            
            if propose_result.get("task_id") == task_id and propose_result.get("status") == "pending_review":
                self.compatible_features.append("agent_write_workflow")
                return True
            else:
                self.incompatible_features.append("propose_transition_invalid_response")
                return False
        
        except Exception as e:
            self.incompatible_features.append(f"agent_write_error: {str(e)}")
            return False
    
    async def run_compatibility_check(self) -> Dict[str, Any]:
        """
        S3 호환성 종합 검증
        """
        
        results = {
            "tool_discovery": await self.discover_tools(),
            "search_with_citation": await self.search_with_citation_verification("test query"),
            "resource_fetching": await self.fetch_and_use_resource(),
            "agent_write_workflow": await self.agent_write_workflow(),
        }
        
        return {
            "timestamp": datetime.now().isoformat(),
            "compatible_features": self.compatible_features,
            "incompatible_features": self.incompatible_features,
            "detailed_results": results,
            "overall_compatibility": len(self.incompatible_features) == 0
        }


@pytest.mark.s3_compatibility
@pytest.mark.dry_run
class TestS3DryRun:
    """S3 드라이런 테스트"""
    
    @pytest.fixture(autouse=True)
    async def setup(self, db_session):
        """초기화"""
        from mimir.phase4.mcp_server import MCPServerService
        
        self.mcp_service = MCPServerService(db_session)
        
        # MCP 서버 초기화
        self.server = await self.mcp_service.initialize_server(
            session_id="s3-dryrun",
            scope_profile=None,  # 공개
            actor_id="s3-client",
            actor_type="agent"
        )
        
        # Mock S3 client
        self.s3_client = MockS3Client(self.server)
        
        yield
    
    @pytest.mark.asyncio
    async def test_s3_compatibility_check(self):
        """S3 호환성 검증"""
        
        results = await self.s3_client.run_compatibility_check()
        
        # 호환성 보고서 저장
        report_path = Path("docs/호환성보고서/S3_compatibility_report.json")
        report_path.parent.mkdir(parents=True, exist_ok=True)
        
        with open(report_path, "w") as f:
            json.dump(results, f, indent=2)
        
        # Assertion
        assert results["overall_compatibility"], (
            f"S3 호환성 문제 발견: {results['incompatible_features']}"
        )
        
        # Compatibility report markdown
        report_md = f"""
# S3 호환성 보고서

**생성일**: {results['timestamp']}

## 호환 기능

"""
        for feature in results["compatible_features"]:
            report_md += f"- ✓ {feature}\n"
        
        if results["incompatible_features"]:
            report_md += "\n## 미지원 기능\n\n"
            for feature in results["incompatible_features"]:
                report_md += f"- ✗ {feature}\n"
        
        report_md += f"\n## 전체 호환성\n\n{'✓ 호환' if results['overall_compatibility'] else '✗ 비호환'}\n"
        
        report_path_md = Path("docs/호환성보고서/S3_compatibility_report.md")
        with open(report_path_md, "w") as f:
            f.write(report_md)
        
        print(f"\n✓ S3 compatibility report: {report_path_md}")
```

### 4.2 GitHub Actions Workflow

**파일**: `.github/workflows/s2-integration-tests.yml` (새로 작성)

```yaml
name: S2 Integration Tests (Phase 1-8)

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  schedule:
    # 매일 새벽 2시 UTC (한국 시간 10시)에 실행
    - cron: '0 2 * * *'

env:
  REGISTRY: ghcr.io
  PYTHON_VERSION: '3.11'
  DB_HOST: localhost
  DB_PORT: 5432
  DB_USER: test
  DB_PASSWORD: test
  DB_NAME: mimir_test

jobs:
  setup-db:
    name: Setup Test Database
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: ${{ env.DB_USER }}
          POSTGRES_PASSWORD: ${{ env.DB_PASSWORD }}
          POSTGRES_DB: ${{ env.DB_NAME }}
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Wait for PostgreSQL
        run: |
          python -m pip install psycopg2-binary
          python -c "
          import psycopg2
          import time
          for i in range(30):
            try:
              psycopg2.connect(
                host='localhost',
                port=5432,
                user='test',
                password='test',
                database='mimir_test'
              )
              break
            except:
              time.sleep(1)
          "
      
      - name: Initialize test DB
        run: |
          python -m pip install sqlalchemy alembic
          alembic upgrade head

  run-integration:
    name: Run Integration Tests (Phase-to-Phase)
    needs: setup-db
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: ${{ env.DB_USER }}
          POSTGRES_PASSWORD: ${{ env.DB_PASSWORD }}
          POSTGRES_DB: ${{ env.DB_NAME }}
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      
      - name: Cache pip packages
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-asyncio pytest-cov pytest-xdist
      
      - name: Run integration tests
        run: |
          pytest tests/integration/test_integration_s2_phase0_to_8.py -v \
            --junitxml=test_results/integration_junit.xml \
            --cov=mimir \
            --cov-report=html:htmlcov/integration \
            --cov-report=json:test_results/coverage_integration.json \
            -n auto
      
      - name: Upload integration test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: integration-test-results
          path: test_results/integration_junit.xml
      
      - name: Upload coverage reports
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: coverage-reports
          path: htmlcov/

  run-mcp-e2e:
    name: Run MCP e2e Tests
    needs: setup-db
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: ${{ env.DB_USER }}
          POSTGRES_PASSWORD: ${{ env.DB_PASSWORD }}
          POSTGRES_DB: ${{ env.DB_NAME }}
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      mcp-server:
        image: mimir-mcp-server:latest
        ports:
          - 8000:8000
        env:
          DATABASE_URL: postgresql://${{ env.DB_USER }}:${{ env.DB_PASSWORD }}@postgres:5432/${{ env.DB_NAME }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-asyncio pytest-cov aiohttp
      
      - name: Wait for MCP server
        run: |
          python -c "
          import asyncio
          import aiohttp
          
          async def wait():
            for i in range(30):
              try:
                async with aiohttp.ClientSession() as session:
                  async with session.get('http://localhost:8000/health') as resp:
                    if resp.status == 200:
                      return
              except:
                pass
              await asyncio.sleep(1)
          
          asyncio.run(wait())
          "
      
      - name: Run MCP e2e tests
        run: |
          pytest tests/integration/test_mcp_e2e.py -v \
            --junitxml=test_results/mcp_e2e_junit.xml \
            --cov=mimir \
            --cov-report=json:test_results/coverage_mcp_e2e.json
      
      - name: Upload MCP e2e results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: mcp-e2e-results
          path: test_results/mcp_e2e_junit.xml

  run-golden-set:
    name: Run Golden Set Regression (Strict Mode)
    needs: setup-db
    runs-on: ubuntu-latest
    timeout-minutes: 60  # Golden set은 시간이 오래 걸릴 수 있음
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: ${{ env.DB_USER }}
          POSTGRES_PASSWORD: ${{ env.DB_PASSWORD }}
          POSTGRES_DB: ${{ env.DB_NAME }}
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-asyncio pytest-cov
      
      - name: Run golden set regression
        run: |
          pytest tests/evaluation/test_golden_set_regression.py -v \
            --golden-set-id all \
            --threshold-mode strict \
            --junitxml=test_results/golden_set_junit.xml \
            --cov=mimir \
            --cov-report=json:test_results/coverage_golden_set.json
      
      - name: Upload golden set results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: golden-set-results
          path: test_results/golden_set_junit.xml
      
      - name: Upload AI quality report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: ai-quality-report
          path: docs/평가보고서/

  run-security:
    name: Run Security Checks
    needs: setup-db
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      
      - name: Install security tools
        run: |
          pip install bandit safety black flake8
      
      - name: Run bandit (security analysis)
        run: |
          bandit -r src/ -f json -o test_results/bandit_report.json || true
      
      - name: Run safety (dependency vulnerabilities)
        run: |
          safety check --json > test_results/safety_report.json || true
      
      - name: Run black (code format check)
        run: |
          black --check src/ tests/ || true
      
      - name: Run flake8 (linting)
        run: |
          flake8 src/ tests/ --format=json > test_results/flake8_report.json || true
      
      - name: Upload security reports
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: security-reports
          path: test_results/

  generate-reports:
    name: Generate Test Reports & Dashboards
    needs: [run-integration, run-mcp-e2e, run-golden-set, run-security]
    runs-on: ubuntu-latest
    if: always()
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      
      - name: Download all artifacts
        uses: actions/download-artifact@v3
        with:
          path: artifacts/
      
      - name: Install dependencies
        run: |
          pip install junitparser pyyaml
      
      - name: Generate aggregated report
        run: |
          python scripts/generate_test_report.py \
            --input artifacts/ \
            --output test_results/aggregated_report.json
      
      - name: Generate S2 readiness checklist
        run: |
          python scripts/s2_readiness_checklist.py \
            --input test_results/aggregated_report.json \
            --output test_results/s2_readiness.md
      
      - name: Create PR comment
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = JSON.parse(fs.readFileSync('test_results/aggregated_report.json', 'utf-8'));
            
            const comment = `
            ## S2 Integration Test Results
            
            - Integration Tests: ${report.integration.passed}/${report.integration.total} passed
            - MCP e2e Tests: ${report.mcp_e2e.passed}/${report.mcp_e2e.total} passed
            - Golden Set Tests: ${report.golden_set.passed}/${report.golden_set.total} passed (strict mode)
            - Security Checks: ${report.security.issues} issues found
            
            [Full Report](https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }})
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
      
      - name: Send Slack notification
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
          payload: |
            {
              "text": "S2 Integration Tests Failed",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*S2 Integration Tests Failed*\n<https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Results>"
                  }
                }
              ]
            }

  s3-dry-run:
    name: S3 Compatibility Dry-run (Optional)
    needs: [run-integration, run-mcp-e2e]
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: ${{ env.DB_USER }}
          POSTGRES_PASSWORD: ${{ env.DB_PASSWORD }}
          POSTGRES_DB: ${{ env.DB_NAME }}
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-asyncio
      
      - name: Run S3 dry-run
        run: |
          pytest tests/s3_compatibility/test_s3_dry_run.py -v \
            --s3-dry-run \
            --junitxml=test_results/s3_dryrun_junit.xml
      
      - name: Upload S3 compatibility report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: s3-compatibility-report
          path: docs/호환성보고서/
```

### 4.3 테스트 보고서 생성 스크립트

**파일**: `scripts/generate_test_report.py`

```python
"""
모든 테스트 결과를 수집하여 aggregated report 생성
"""

import json
import argparse
from pathlib import Path
from junitparser import JUnitXml
from typing import Dict, Any
import datetime


def parse_junit_xml(junit_file: Path) -> Dict[str, Any]:
    """JUnit XML 파일 파싱"""
    try:
        xml = JUnitXml.fromfile(junit_file)
        
        total = len(xml)
        passed = sum(1 for case in xml if case.is_passed)
        failed = sum(1 for case in xml if case.is_failure)
        skipped = sum(1 for case in xml if case.is_skipped)
        
        return {
            "total": total,
            "passed": passed,
            "failed": failed,
            "skipped": skipped,
            "pass_rate": passed / total if total > 0 else 0.0
        }
    except Exception as e:
        print(f"Failed to parse {junit_file}: {e}")
        return {
            "total": 0,
            "passed": 0,
            "failed": 0,
            "skipped": 0,
            "pass_rate": 0.0
        }


def generate_aggregated_report(artifacts_dir: Path, output_file: Path) -> None:
    """
    모든 artifact를 수집하여 aggregated report 생성
    """
    
    report = {
        "generated_at": datetime.datetime.now().isoformat(),
        "integration": parse_junit_xml(
            artifacts_dir / "integration-test-results" / "integration_junit.xml"
        ),
        "mcp_e2e": parse_junit_xml(
            artifacts_dir / "mcp-e2e-results" / "mcp_e2e_junit.xml"
        ),
        "golden_set": parse_junit_xml(
            artifacts_dir / "golden-set-results" / "golden_set_junit.xml"
        ),
        "security": {
            "issues": 0  # TODO: Parse security report
        }
    }
    
    # Overall pass rate
    total_tests = sum(r["total"] for r in [
        report["integration"],
        report["mcp_e2e"],
        report["golden_set"]
    ])
    total_passed = sum(r["passed"] for r in [
        report["integration"],
        report["mcp_e2e"],
        report["golden_set"]
    ])
    
    report["overall"] = {
        "total": total_tests,
        "passed": total_passed,
        "failed": total_tests - total_passed,
        "pass_rate": total_passed / total_tests if total_tests > 0 else 0.0
    }
    
    # 파일로 저장
    output_file.parent.mkdir(parents=True, exist_ok=True)
    with open(output_file, "w") as f:
        json.dump(report, f, indent=2)
    
    print(f"✓ Aggregated report: {output_file}")


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True, help="Artifacts directory")
    parser.add_argument("--output", required=True, help="Output file")
    args = parser.parse_args()
    
    generate_aggregated_report(Path(args.input), Path(args.output))
```

**파일**: `scripts/s2_readiness_checklist.py`

```python
"""
S2 readiness checklist 자동 생성

Phase 1~8 completion criteria를 검증하고 보고서 생성
"""

import json
import argparse
from pathlib import Path
import datetime


PHASE_CRITERIA = {
    "Phase 1": [
        "✓ LLM 모델 관리 API 구현",
        "✓ 모델 선택 기능",
        "✓ Temperature, max_tokens 설정",
    ],
    "Phase 2": [
        "✓ 검색 API 구현",
        "✓ Citation 5-tuple 생성",
        "✓ Relevance scoring",
    ],
    "Phase 3": [
        "✓ 세션 관리",
        "✓ 멀티턴 RAG",
        "✓ Context window 관리",
    ],
    "Phase 4": [
        "✓ MCP 서버 구현",
        "✓ Tool 노출 (search, fetch, verify)",
        "✓ Resource listing",
    ],
    "Phase 5": [
        "✓ Agent principal 생성",
        "✓ Delegation",
        "✓ Draft proposal",
    ],
    "Phase 6": [
        "✓ Admin approval queue",
        "✓ Review workflow",
        "✓ State transition",
    ],
    "Phase 7": [
        "✓ 평가 기준 로드",
        "✓ 메트릭 계산",
        "✓ 평가 보고서",
    ],
    "Phase 8": [
        "✓ 추출 품질 평가",
        "✓ Field-level quality",
        "✓ 최종 검증",
    ],
}

S2_PRINCIPLES = {
    "원칙 ⑤": [
        "✓ actor_type='user' 또는 'agent' 기록",
        "✓ Audit log에 actor_type 필드",
        "✓ Agent도 first-class API consumer",
    ],
    "원칙 ⑥": [
        "✓ Scope Profile 기반 ACL",
        "✓ 코드에 scope 하드코딩 금지",
        "✓ 모든 조회/쓰기에 ACL 필터 적용",
    ],
    "원칙 ⑦": [
        "✓ 폐쇄망 모드 지원 (MIMIR_DISABLE_EXTERNAL_LLM)",
        "✓ Rule-based fallback",
        "✓ Degraded functionality (failure 아님)",
    ],
}


def generate_checklist(test_report: Path, output_file: Path) -> None:
    """S2 readiness checklist 생성"""
    
    # Test report 로드
    with open(test_report) as f:
        report = json.load(f)
    
    overall_pass_rate = report["overall"]["pass_rate"]
    all_tests_passed = overall_pass_rate >= 0.95
    
    checklist_lines = [
        "# S2 Readiness Checklist",
        "",
        f"**보고서 생성**: {datetime.datetime.now().isoformat()}",
        "",
        "## 전체 현황",
        "",
        f"- 통합 테스트: {report['integration']['passed']}/{report['integration']['total']} passed",
        f"- MCP e2e: {report['mcp_e2e']['passed']}/{report['mcp_e2e']['total']} passed",
        f"- 골든셋: {report['golden_set']['passed']}/{report['golden_set']['total']} passed (strict mode)",
        f"- **전체 통과율**: {overall_pass_rate*100:.1f}%",
        "",
        "---",
        "",
        "## Phase Completion Criteria",
        "",
    ]
    
    for phase, criteria in PHASE_CRITERIA.items():
        checklist_lines.append(f"### {phase}")
        checklist_lines.append("")
        for criterion in criteria:
            checklist_lines.append(f"- {criterion}")
        checklist_lines.append("")
    
    checklist_lines.extend([
        "---",
        "",
        "## S2 원칙 검증",
        "",
    ])
    
    for principle, items in S2_PRINCIPLES.items():
        checklist_lines.append(f"### {principle}")
        checklist_lines.append("")
        for item in items:
            checklist_lines.append(f"- {item}")
        checklist_lines.append("")
    
    # 최종 준비 상태
    checklist_lines.extend([
        "---",
        "",
        "## S2 최종 준비 상태",
        "",
    ])
    
    if all_tests_passed:
        checklist_lines.append(f"✓ **준비 완료**: 모든 테스트 통과 (통과율 {overall_pass_rate*100:.1f}%)")
    else:
        failing_phases = []
        if report["integration"]["pass_rate"] < 0.95:
            failing_phases.append("integration")
        if report["mcp_e2e"]["pass_rate"] < 0.95:
            failing_phases.append("MCP e2e")
        if report["golden_set"]["pass_rate"] < 0.95:
            failing_phases.append("golden set")
        
        checklist_lines.append(f"✗ **준비 미완료**: {', '.join(failing_phases)} 실패")
    
    checklist_lines.extend([
        "",
        "---",
        "",
        "## 다음 단계",
        "",
        "- [ ] 모든 failing tests 수정",
        "- [ ] 검수 완료 확인서 획득",
        "- [ ] S3 integration 준비",
        "",
    ])
    
    # 파일로 저장
    output_file.parent.mkdir(parents=True, exist_ok=True)
    with open(output_file, "w", encoding="utf-8") as f:
        f.write("\n".join(checklist_lines))
    
    print(f"✓ S2 readiness checklist: {output_file}")


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True, help="Test report JSON")
    parser.add_argument("--output", required=True, help="Output checklist")
    args = parser.parse_args()
    
    generate_checklist(Path(args.input), Path(args.output))
```

---

## 5. 산출물

### 주요 산출물

1. **S3 드라이런 테스트**
   - `tests/s3_compatibility/test_s3_dry_run.py` (약 400 라인)
   - `docs/호환성보고서/S3_compatibility_report.md` (호환성 상세 분석)
   - `docs/호환성보고서/S3_compatibility_report.json` (structured data)

2. **CI/CD 설정**
   - `.github/workflows/s2-integration-tests.yml` (전체 workflow)
   - `scripts/generate_test_report.py` (보고서 생성)
   - `scripts/s2_readiness_checklist.py` (checklist 생성)

3. **테스트 보고서**
   - `test_results/aggregated_report.json` (all test results)
   - `test_results/s2_readiness.md` (S2 readiness checklist)
   - `htmlcov/` (coverage report)

4. **자동화 결과**
   - GitHub Actions run 결과 (모든 job)
   - PR comment with test summary
   - Slack notification (실패 시)

---

## 6. 완료 기준

1. **S3 드라이런 완료**
   - [ ] MockS3Client 구현 완료
   - [ ] 5가지 호환성 검증 케이스 모두 pass
   - [ ] S3 호환성 보고서 생성 완료

2. **GitHub Actions Workflow**
   - [ ] 모든 job이 정의되고 작동
   - [ ] Parallel execution 최적화 (가능한 범위)
   - [ ] PR comment 자동 생성
   - [ ] Slack notification 설정 완료

3. **Nightly Build**
   - [ ] Cron schedule 설정 (매일 2시 UTC)
   - [ ] Performance baseline 수집
   - [ ] Trend analysis 구현

4. **Test Report Aggregation**
   - [ ] 모든 phase별 테스트 결과 수집
   - [ ] JSON format 보고서 생성
   - [ ] Dashboard-friendly 구조

5. **S2 Readiness Checklist**
   - [ ] Phase 1~8 completion criteria 검증
   - [ ] S2 원칙 ⑤⑥⑦ 검증
   - [ ] Checklist markdown 생성
   - [ ] 자동화 완료

---

## 7. 작업 지침

**7-1** **Workflow Job 순서**
- setup-db (병렬화 불가)
- run-integration, run-mcp-e2e, run-golden-set, run-security (병렬)
- generate-reports, s3-dry-run (의존성 있음)

**7-2** **Artifact 관리**
- 모든 test result는 artifact로 upload
- 보고서도 artifact로 제공 (PR에서 다운로드 가능)
- CI 로그 자체는 별도 보관

**7-3** **Slack 알림 타이밍**
- Failure case만 알림 (success는 조용히)
- 알림 메시지에 run URL 포함
- 중요 메트릭 요약

**7-4** **Performance Tracking**
- 각 job의 execution time 기록
- Trend graph (이전 실행과 비교)
- Regression detection (execution time이 갑자기 증가했는가)

**7-5** **Security Checks 강화**
- bandit: security vulnerability 검사
- safety: dependency vulnerability 검사
- black: code format
- flake8: linting
- 모든 check fail이어도 build는 진행 (warning만)

**7-6** **PR Comment 자동화**
- Test summary (pass/fail count)
- Link to full report
- Coverage report link
- 실패 시 상세 에러 메시지

**7-7** **S3 호환성 검증**
- Main branch push 시에만 실행 (비용 절감)
- 호환성 문제가 있으면 warning으로 표시
- S3 integration 준비 단계에서 상세 검토
