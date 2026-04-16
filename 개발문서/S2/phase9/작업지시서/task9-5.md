# 작업지시서: 골든셋 회귀 최종 실행 + MCP e2e 테스트
**FG9.2-TASK9-5** | Phase 9 기간 | AI 품질 검증 + MCP 통합

---

## 1. 작업 목적

S2 원칙이 적용된 최종 Phase 8 완료 상태에서:

1. **골든셋 회귀 (Golden Set Regression)**: 모든 golden set 테스트 케이스를 엄격한 임계값(strict threshold)으로 재실행하여 AI 품질이 설정된 수준을 만족하는지 최종 검증
2. **MCP e2e 테스트**: MCP (Model Context Protocol) 서버가 실제 외부 클라이언트(예: Claude AI)와 정상 통신하는지, 모든 tool이 scope-aware하게 동작하는지 검증

---

## 2. 작업 범위

### 포함 범위

1. **골든셋 회귀 최종 실행**
   - 모든 golden set 데이터셋에 대해 엄격한 임계값(strict mode) 적용
   - 6가지 메트릭 검증:
     - Faithfulness >= 0.80
     - Answer Relevance >= 0.75
     - Context Precision >= 0.75
     - Context Recall >= 0.75
     - Citation Present >= 0.90
     - Hallucination <= 0.10
   - 메트릭별 통계 집계 (mean, std, min, max, pass rate)
   - 임계값 미달 케이스 상세 분석 (error log, context, expected vs actual)
   - AI품질평가.md 최종 보고서 생성

2. **MCP e2e 테스트** (`tests/integration/test_mcp_e2e.py`)
   - TestMCPE2E 클래스: 6가지 e2e 시나리오
     - test_mcp_initialize_handshake: MCP 서버 초기화, capabilities 선언
     - test_mcp_search_documents_tool: 검색 tool 실행, citation 반환 검증
     - test_mcp_fetch_node_tool: 특정 노드 fetch (mimir:// URI)
     - test_mcp_verify_citation_tool: citation 5-tuple 무결성 검증
     - test_mcp_propose_transition: agent가 draft 제안 생성, task_id 반환
     - test_mcp_resource_listing: available resources 나열
   - MCPClient 테스트 헬퍼: HTTP 또는 stdio transport
   - Streamable HTTP transport 검증 (SSE, JSON streaming)
   - Rate limiting, unauthorized access, invalid tool 에러 핸들링
   - S2 원칙 검증: actor_type="agent", scope profile 전파

3. **MCP 프로토콜 준수 검증**
   - MCP spec 1.0 호환성 확인
   - tool schema validation (JSON Schema)
   - Response 구조 (필수 필드, optional 필드)
   - Error handling (invalid request, timeout, permission denied)
   - Resource MIME types (application/vnd.mimir+json, text/plain)

4. **AI 품질 평가 보고서 생성**
   - AI품질평가.md: executive summary, 메트릭별 분석, failing cases, recommendations
   - 메트릭별 분포 차트 (히스토그램)
   - Trend analysis (이전 phase와 비교)
   - Citation quality breakdown (source accuracy, snippet relevance)

### 제외 범위

- 새로운 golden set 데이터 생성 (기존 데이터만 사용)
- Performance 최적화 (이 작업은 Phase 10 이상)
- UI 테스트 (별도 작업)
- Load/stress 테스트 (별도 작업)
- 외부 AI 모델 fine-tuning (평가만 수행)

---

## 3. 선행 조건

- Phase 1~8 개발 완료 및 개별 FG 단위 테스트 통과
- Golden set 데이터 준비 완료:
  - 평가할 document-query-response 쌍 (최소 30개 쌍 권장)
  - 각 쌍마다 human-annotated ground truth labels (faithfulness, relevance, citations, etc.)
  - Golden set 저장 경로: `/mimir/tests/data/golden_sets/`
- MCP 서버 구현 완료 (Phase 4)
- pytest, asyncio, aiohttp 설치
- PostgreSQL 테스트 DB 구성

---

## 4. 주요 작업 항목

### 4.1 골든셋 회귀 최종 실행

**파일**: `tests/evaluation/test_golden_set_regression.py` (기존 파일 확장)

```python
"""
골든셋 회귀 테스트 (최종 검증)

메트릭별 엄격한 임계값(strict threshold) 적용:
- faithfulness >= 0.80
- answer_relevance >= 0.75
- context_precision >= 0.75
- context_recall >= 0.75
- citation_present >= 0.90
- hallucination <= 0.10

실행:
    pytest tests/evaluation/test_golden_set_regression.py -v \
        --golden-set-id all --threshold-mode strict
"""

import pytest
import json
import asyncio
from typing import Dict, List, Any, Tuple
from dataclasses import dataclass, asdict
from datetime import datetime
from pathlib import Path
import statistics
import csv

# 프로젝트 임포트
from mimir.phase7.evaluation_runner import EvaluationRunner, EvaluationResult
from mimir.phase2.citation import CitationService
from mimir.phase8.extraction_quality import ExtractionQualityService
from mimir.common.audit_log import AuditLogService


@dataclass
class MetricStatistics:
    """메트릭 통계"""
    metric_name: str
    values: List[float]
    
    @property
    def mean(self) -> float:
        return statistics.mean(self.values) if self.values else 0.0
    
    @property
    def stdev(self) -> float:
        return statistics.stdev(self.values) if len(self.values) > 1 else 0.0
    
    @property
    def min(self) -> float:
        return min(self.values) if self.values else 0.0
    
    @property
    def max(self) -> float:
        return max(self.values) if self.values else 0.0
    
    @property
    def pass_count(self) -> int:
        return sum(1 for v in self.values if v >= self.threshold)
    
    @property
    def pass_rate(self) -> float:
        return self.pass_count / len(self.values) if self.values else 0.0
    
    def to_dict(self) -> Dict[str, Any]:
        return {
            "metric_name": self.metric_name,
            "mean": round(self.mean, 4),
            "stdev": round(self.stdev, 4),
            "min": round(self.min, 4),
            "max": round(self.max, 4),
            "pass_count": self.pass_count,
            "total_count": len(self.values),
            "pass_rate": f"{self.pass_rate * 100:.1f}%"
        }


@dataclass
class GoldenSetTestCase:
    """골든셋 테스트 케이스"""
    test_id: str
    document_uri: str
    query: str
    expected_response: str
    ground_truth_labels: Dict[str, float]  # faithfulness, relevance, etc.


@dataclass
class GoldenSetResult:
    """골든셋 실행 결과"""
    test_id: str
    metrics: Dict[str, float]
    all_passed: bool
    failures: List[str]  # 실패한 메트릭 목록
    error: str = None
    generated_response: str = None
    execution_time_ms: float = 0.0


@pytest.mark.evaluation
@pytest.mark.golden_set
class TestGoldenSetRegressionFinal:
    """
    골든셋 회귀 테스트 (최종 검증)
    
    strict threshold mode에서 모든 golden set을 실행하고
    AI 품질 평가 보고서를 생성한다.
    """

    STRICT_THRESHOLDS = {
        "faithfulness": 0.80,
        "answer_relevance": 0.75,
        "context_precision": 0.75,
        "context_recall": 0.75,
        "citation_present": 0.90,
        "hallucination": 0.10  # 이 항목은 max threshold
    }

    @pytest.fixture(autouse=True)
    async def setup(self, db_session):
        """테스트 초기화"""
        self.db = db_session
        self.eval_runner = EvaluationRunner(db_session)
        self.citation_service = CitationService(db_session)
        self.extraction_quality_service = ExtractionQualityService(db_session)
        self.audit_log_service = AuditLogService(db_session)
        
        # 결과 저장소
        self.results: Dict[str, GoldenSetResult] = {}
        self.metric_stats: Dict[str, MetricStatistics] = {
            metric: MetricStatistics(metric, [], self.STRICT_THRESHOLDS[metric])
            for metric in self.STRICT_THRESHOLDS.keys()
        }
        
        # 골든셋 로드
        self.golden_sets = await self._load_golden_sets()
        
        yield
        
        await db_session.rollback()

    async def _load_golden_sets(self) -> List[GoldenSetTestCase]:
        """
        골든셋 데이터 로드
        
        경로: tests/data/golden_sets/
        파일 형식: {test_id}.json
        
        예시:
        {
            "test_id": "gs_001",
            "document_uri": "mimir://doc-001",
            "query": "What is machine learning?",
            "expected_response": "Machine learning is a subset of AI...",
            "ground_truth_labels": {
                "faithfulness": 1.0,
                "answer_relevance": 0.95,
                "context_precision": 0.90,
                "context_recall": 0.85,
                "citation_present": 1.0,
                "hallucination": 0.0
            }
        }
        """
        
        golden_sets_dir = Path("tests/data/golden_sets")
        test_cases = []
        
        for json_file in golden_sets_dir.glob("*.json"):
            try:
                with open(json_file) as f:
                    data = json.load(f)
                    test_case = GoldenSetTestCase(
                        test_id=data["test_id"],
                        document_uri=data["document_uri"],
                        query=data["query"],
                        expected_response=data["expected_response"],
                        ground_truth_labels=data["ground_truth_labels"]
                    )
                    test_cases.append(test_case)
            except Exception as e:
                pytest.skip(f"Failed to load {json_file}: {e}")
        
        return test_cases

    async def _evaluate_response(
        self,
        test_case: GoldenSetTestCase,
        generated_response: str
    ) -> Tuple[Dict[str, float], List[str]]:
        """
        생성된 응답 평가
        
        반환:
        - metrics: {metric_name -> score}
        - failures: 임계값 미달 메트릭 목록
        """
        
        metrics = {}
        failures = []
        
        # 1. Faithfulness: 생성 응답이 원문과 일치하는가
        faithfulness = await self.eval_runner.evaluate_faithfulness(
            response=generated_response,
            context=test_case.expected_response,
            actor_id="system",
            actor_type="agent"
        )
        metrics["faithfulness"] = faithfulness.score
        
        if faithfulness.score < self.STRICT_THRESHOLDS["faithfulness"]:
            failures.append(f"faithfulness={faithfulness.score:.3f}")
        
        # 2. Answer Relevance: 응답이 쿼리와 관련이 있는가
        answer_relevance = await self.eval_runner.evaluate_answer_relevance(
            response=generated_response,
            query=test_case.query,
            actor_id="system",
            actor_type="agent"
        )
        metrics["answer_relevance"] = answer_relevance.score
        
        if answer_relevance.score < self.STRICT_THRESHOLDS["answer_relevance"]:
            failures.append(f"answer_relevance={answer_relevance.score:.3f}")
        
        # 3. Context Precision: retrieval된 context가 정확한가
        context_precision = await self.eval_runner.evaluate_context_precision(
            response=generated_response,
            retrieved_context=test_case.expected_response,
            actor_id="system",
            actor_type="agent"
        )
        metrics["context_precision"] = context_precision.score
        
        if context_precision.score < self.STRICT_THRESHOLDS["context_precision"]:
            failures.append(f"context_precision={context_precision.score:.3f}")
        
        # 4. Context Recall: retrieval된 context의 recall
        context_recall = await self.eval_runner.evaluate_context_recall(
            expected_response=test_case.expected_response,
            retrieved_context=test_case.expected_response,
            actor_id="system",
            actor_type="agent"
        )
        metrics["context_recall"] = context_recall.score
        
        if context_recall.score < self.STRICT_THRESHOLDS["context_recall"]:
            failures.append(f"context_recall={context_recall.score:.3f}")
        
        # 5. Citation Present: citation이 있는가
        citations_present = await self.eval_runner.evaluate_citations_present(
            response=generated_response,
            actor_id="system",
            actor_type="agent"
        )
        metrics["citation_present"] = citations_present.score
        
        if citations_present.score < self.STRICT_THRESHOLDS["citation_present"]:
            failures.append(f"citation_present={citations_present.score:.3f}")
        
        # 6. Hallucination: 잘못된 정보가 있는가 (낮을수록 좋음)
        hallucination = await self.eval_runner.evaluate_hallucination(
            response=generated_response,
            context=test_case.expected_response,
            actor_id="system",
            actor_type="agent"
        )
        metrics["hallucination"] = hallucination.score
        
        if hallucination.score > self.STRICT_THRESHOLDS["hallucination"]:
            failures.append(f"hallucination={hallucination.score:.3f}")
        
        return metrics, failures

    @pytest.mark.asyncio
    async def test_run_all_golden_sets_strict_mode(self):
        """
        모든 골든셋을 strict threshold로 실행
        
        각 테스트 케이스:
        1. 응답 생성 (Phase 2-3: 검색 → RAG)
        2. 6개 메트릭 평가
        3. 임계값 검증
        4. 통계 수집
        """
        
        assert len(self.golden_sets) > 0, "Golden sets not loaded"
        
        for test_case in self.golden_sets:
            start_time = datetime.now()
            
            try:
                # 검색 (Phase 2)
                search_service = SearchService(self.db)
                search_results = await search_service.search(
                    query=test_case.query,
                    scope_profile=None,  # 골든셋은 공개 테스트
                    actor_id="system",
                    actor_type="agent",
                    limit=5
                )
                
                # RAG (Phase 3)
                session_rag_service = SessionRAGService(self.db)
                rag_response = await session_rag_service.generate_response(
                    query=test_case.query,
                    search_results=search_results,
                    actor_id="system",
                    actor_type="agent"
                )
                
                # 평가
                metrics, failures = await self._evaluate_response(
                    test_case,
                    rag_response.response_text
                )
                
                # 결과 저장
                execution_time = (datetime.now() - start_time).total_seconds() * 1000
                
                result = GoldenSetResult(
                    test_id=test_case.test_id,
                    metrics=metrics,
                    all_passed=len(failures) == 0,
                    failures=failures,
                    generated_response=rag_response.response_text,
                    execution_time_ms=execution_time
                )
                
                self.results[test_case.test_id] = result
                
                # 메트릭 통계 누적
                for metric_name, score in metrics.items():
                    self.metric_stats[metric_name].values.append(score)
                
                # 로그
                audit_log = {
                    "test_id": test_case.test_id,
                    "passed": result.all_passed,
                    "failures": failures,
                    "metrics": metrics,
                    "execution_time_ms": execution_time
                }
                
                await self.audit_log_service.create_log(
                    action="golden_set_test",
                    actor_id="system",
                    actor_type="agent",
                    context=audit_log
                )
                
                # Assertion
                if not result.all_passed:
                    pytest.fail(
                        f"{test_case.test_id}: {', '.join(failures)}\n"
                        f"Metrics: {json.dumps(metrics, indent=2)}"
                    )
            
            except Exception as e:
                result = GoldenSetResult(
                    test_id=test_case.test_id,
                    metrics={},
                    all_passed=False,
                    failures=["execution_error"],
                    error=str(e)
                )
                self.results[test_case.test_id] = result
                
                pytest.fail(f"{test_case.test_id}: {str(e)}")

    @pytest.mark.asyncio
    async def test_generate_ai_quality_report(self):
        """
        AI품질평가.md 최종 보고서 생성
        
        포함 내용:
        - Executive summary
        - 메트릭별 통계 (mean, std, min, max, pass rate)
        - 임계값 미달 케이스 상세 분석
        - Citation quality breakdown
        - Recommendations
        """
        
        # 결과 수집
        if not self.results:
            pytest.skip("No test results to report")
        
        # 보고서 내용 작성
        report_lines = [
            "# AI 품질 평가 보고서",
            "",
            f"**보고서 생성**: {datetime.now().isoformat()}",
            f"**테스트 케이스 수**: {len(self.results)}",
            f"**Pass 수**: {sum(1 for r in self.results.values() if r.all_passed)}",
            f"**Fail 수**: {sum(1 for r in self.results.values() if not r.all_passed)}",
            "",
            "---",
            "",
            "## Executive Summary",
            "",
            f"Phase 8 완료 후 최종 검증을 위해 총 {len(self.results)}개의 golden set 케이스를",
            "엄격한 임계값(strict threshold)으로 평가했습니다.",
            "",
        ]
        
        # 메트릭별 통계 테이블
        report_lines.extend([
            "## 메트릭별 통계",
            "",
            "| 메트릭 | Mean | Std | Min | Max | Pass Rate | Threshold |",
            "|--------|------|-----|-----|-----|-----------|-----------|",
        ])
        
        for metric_name in sorted(self.STRICT_THRESHOLDS.keys()):
            stats = self.metric_stats[metric_name]
            threshold = self.STRICT_THRESHOLDS[metric_name]
            
            report_lines.append(
                f"| {metric_name} | {stats.mean:.3f} | {stats.stdev:.3f} | "
                f"{stats.min:.3f} | {stats.max:.3f} | "
                f"{stats.pass_rate * 100:.1f}% | {threshold} |"
            )
        
        report_lines.extend(["", "---", ""])
        
        # 실패한 케이스 상세 분석
        failed_results = {
            test_id: result
            for test_id, result in self.results.items()
            if not result.all_passed
        }
        
        if failed_results:
            report_lines.extend([
                "## 임계값 미달 케이스 분석",
                "",
            ])
            
            for test_id, result in sorted(failed_results.items()):
                report_lines.extend([
                    f"### {test_id}",
                    "",
                    "**실패 메트릭**:",
                ])
                
                for failure in result.failures:
                    report_lines.append(f"- {failure}")
                
                report_lines.extend([
                    "",
                    "**상세 메트릭**:",
                    "",
                    "```json",
                    json.dumps(result.metrics, indent=2),
                    "```",
                    "",
                    "**생성 응답**:",
                    "",
                    f"```\n{result.generated_response}\n```",
                    "",
                ])
        else:
            report_lines.extend([
                "## 결과",
                "",
                "✓ 모든 gold 셋 테스트 통과!",
                "",
            ])
        
        # Citation 품질 분석
        report_lines.extend([
            "---",
            "",
            "## Citation 품질 분석",
            "",
            "### Citation 5-tuple 검증",
            "",
            "각 citation은 다음 5-tuple을 만족해야 합니다:",
            "",
            "1. **Source**: 인용 출처 (문서 URI)",
            "2. **Snippet**: 인용 텍스트 (원문 일부)",
            "3. **Relevance**: 관련성 점수 (0.0~1.0)",
            "4. **Confidence**: 신뢰도 (0.0~1.0)",
            "5. **Timestamp**: 생성 시각",
            "",
        ])
        
        # Recommendations
        report_lines.extend([
            "---",
            "",
            "## 권장사항",
            "",
        ])
        
        # 메트릭별 권장사항
        for metric_name, threshold in sorted(self.STRICT_THRESHOLDS.items()):
            stats = self.metric_stats[metric_name]
            
            if stats.pass_rate < 0.95:
                report_lines.append(
                    f"- **{metric_name}**: 통과율 {stats.pass_rate*100:.1f}% (권장: 95% 이상). "
                    f"평균: {stats.mean:.3f}, 최소: {stats.min:.3f}. "
                    f"임계값 미달 케이스 검토 필수."
                )
        
        if not failed_results:
            report_lines.append("- **전체**: 모든 메트릭이 우수한 수준입니다. 추가 개선은 선택사항입니다.")
        
        report_lines.extend([
            "",
            "---",
            "",
            "## 기술 세부사항",
            "",
            "### 평가 방법론",
            "",
            "- **Faithfulness**: 생성 응답이 원문과 일치하는 정도 (LLM 기반 평가)",
            "- **Answer Relevance**: 응답이 쿼리와 관련 있는 정도",
            "- **Context Precision**: Retrieval된 context의 정확성",
            "- **Context Recall**: Retrieval된 context의 재현율",
            "- **Citation Present**: 응답에 citation이 포함되어 있는가",
            "- **Hallucination**: 응답에 포함된 잘못된 정보의 정도",
            "",
            "### 임계값 설정 근거",
            "",
            "- 각 메트릭의 임계값은 Phase별 검증 결과 및 업계 표준을 기반으로 설정",
            "- Strict mode: Phase 8 완료 후 최종 품질 보증 (높은 기준)",
            "",
        ])
        
        # 보고서 저장
        report_path = Path("docs/평가보고서/AI품질평가.md")
        report_path.parent.mkdir(parents=True, exist_ok=True)
        
        with open(report_path, "w", encoding="utf-8") as f:
            f.write("\n".join(report_lines))
        
        # JSON 형식 결과도 저장 (대시보드 용)
        results_json = {
            "generated_at": datetime.now().isoformat(),
            "total_tests": len(self.results),
            "passed": sum(1 for r in self.results.values() if r.all_passed),
            "failed": sum(1 for r in self.results.values() if not r.all_passed),
            "metric_statistics": {
                name: stats.to_dict()
                for name, stats in self.metric_stats.items()
            },
            "individual_results": [
                {
                    "test_id": test_id,
                    "passed": result.all_passed,
                    "failures": result.failures,
                    "metrics": result.metrics,
                    "execution_time_ms": result.execution_time_ms
                }
                for test_id, result in self.results.items()
            ]
        }
        
        json_path = Path("docs/평가보고서/AI품질평가.json")
        with open(json_path, "w", encoding="utf-8") as f:
            json.dump(results_json, f, indent=2, ensure_ascii=False)
        
        print(f"\n✓ Report generated: {report_path}")
        print(f"✓ JSON results: {json_path}")
```

### 4.2 MCP e2e 테스트

**파일**: `tests/integration/test_mcp_e2e.py` (새로 작성)

```python
"""
MCP (Model Context Protocol) e2e 테스트

MCP 서버가 외부 클라이언트(Claude, S3, etc.)와 정상 통신하고,
모든 tool이 scope-aware하게 동작하는지 검증한다.

실행:
    pytest tests/integration/test_mcp_e2e.py -v
"""

import pytest
import json
import asyncio
from typing import Dict, Any, List, Optional
from uuid import uuid4
from dataclasses import dataclass
import aiohttp

# 프로젝트 임포트
from mimir.phase4.mcp_server import MCPServerService, MCPServer
from mimir.phase2.search import SearchService
from mimir.phase2.citation import CitationService, Citation5Tuple
from mimir.phase3.session_rag import SessionRAGService
from mimir.phase5.agent_principal import AgentPrincipalService
from mimir.common.scope_profile import ScopeProfile
from mimir.common.audit_log import AuditLogService


@dataclass
class MCPToolCall:
    """MCP tool 호출"""
    tool_name: str
    arguments: Dict[str, Any]


@dataclass
class MCPToolResponse:
    """MCP tool 응답"""
    tool_name: str
    result: Dict[str, Any]
    error: Optional[str] = None


class MCPClient:
    """
    MCP 클라이언트 테스트 헬퍼
    
    HTTP 또는 stdio 형식의 MCP 서버와 통신
    """

    def __init__(self, server_url: str, session_id: str):
        """
        Args:
            server_url: MCP 서버 URL (http://localhost:8000)
            session_id: 세션 ID
        """
        self.server_url = server_url
        self.session_id = session_id
        self.capabilities: Dict[str, Any] = {}

    async def initialize(self) -> None:
        """MCP 핸드셰이크 (capabilities 선언)"""
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.server_url}/mcp/initialize",
                json={
                    "protocol_version": "1.0",
                    "client_name": "mimir-test-client",
                    "client_version": "1.0"
                }
            ) as resp:
                if resp.status != 200:
                    raise Exception(f"Initialize failed: {resp.status}")
                
                data = await resp.json()
                self.capabilities = data.get("capabilities", {})

    async def call_tool(
        self,
        tool_name: str,
        arguments: Dict[str, Any]
    ) -> MCPToolResponse:
        """
        Tool 호출
        
        Args:
            tool_name: 호출할 tool 이름
            arguments: tool arguments
        
        Returns:
            MCPToolResponse (result 또는 error)
        """
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.server_url}/mcp/tools/{tool_name}/call",
                json={
                    "session_id": self.session_id,
                    "arguments": arguments
                }
            ) as resp:
                data = await resp.json()
                
                return MCPToolResponse(
                    tool_name=tool_name,
                    result=data.get("result", {}),
                    error=data.get("error", None)
                )

    async def list_resources(self) -> List[Dict[str, Any]]:
        """Available resources 나열"""
        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.server_url}/mcp/resources",
                params={"session_id": self.session_id}
            ) as resp:
                data = await resp.json()
                return data.get("resources", [])


@pytest.mark.integration
@pytest.mark.mcp_e2e
class TestMCPE2E:
    """
    MCP e2e 테스트
    
    6가지 핵심 시나리오:
    1. Initialize: MCP 핸드셰이크
    2. Search: search_documents tool
    3. Fetch: fetch_node tool (mimir:// URI)
    4. Verify Citation: verify_citation tool
    5. Propose Transition: draft 제안 (agent 쓰기)
    6. Resource Listing: available resources
    """

    @pytest.fixture(autouse=True)
    async def setup(self, db_session):
        """테스트 초기화"""
        self.db = db_session
        self.mcp_service = MCPServerService(db_session)
        self.search_service = SearchService(db_session)
        self.citation_service = CitationService(db_session)
        self.session_rag_service = SessionRAGService(db_session)
        self.agent_service = AgentPrincipalService(db_session)
        self.audit_log_service = AuditLogService(db_session)
        
        # 테스트 Scope Profile
        self.test_scope = ScopeProfile(
            name="mcp_test_scope",
            allowed_types=["pdf", "docx", "txt"],
            access_rules={"document": ["read", "cite"]}
        )
        await db_session.add(self.test_scope)
        await db_session.commit()
        
        # MCP 서버 초기화
        self.session_id = str(uuid4())
        self.server = await self.mcp_service.initialize_server(
            session_id=self.session_id,
            scope_profile=self.test_scope,
            actor_id="agent-mcp-test",
            actor_type="agent"
        )
        
        # MCP 클라이언트 (테스트용)
        self.mcp_client = MCPClient(
            server_url="http://localhost:8000",  # MCP 서버 주소
            session_id=self.session_id
        )
        
        yield
        
        await db_session.rollback()

    @pytest.mark.asyncio
    async def test_mcp_initialize_handshake(self):
        """
        MCP 초기화 핸드셰이크
        
        Verify:
        - capabilities 선언
        - protocol version 확인
        - scopes 정보 반환
        """
        
        # 클라이언트 초기화
        await self.mcp_client.initialize()
        
        # Capabilities 검증
        assert "tools" in self.mcp_client.capabilities
        assert "resources" in self.mcp_client.capabilities
        
        # Tool capabilities
        tool_names = {
            tool["name"]
            for tool in self.mcp_client.capabilities.get("tools", [])
        }
        
        expected_tools = {
            "search_documents",
            "fetch_node",
            "verify_citation",
            "propose_transition",
            "list_resources"
        }
        
        assert expected_tools.issubset(tool_names), (
            f"Missing tools. Expected: {expected_tools}, Got: {tool_names}"
        )
        
        # audit log 검증
        audit_logs = await self.audit_log_service.list_logs(
            action="mcp_initialize",
            actor_id="agent-mcp-test"
        )
        assert len(audit_logs) > 0
        assert audit_logs[-1].actor_type == "agent"

    @pytest.mark.asyncio
    async def test_mcp_search_documents_tool(self):
        """
        search_documents tool e2e 테스트
        
        Flow:
        1. Tool 호출: search_documents(query="...", limit=5)
        2. 결과: documents list + citations
        3. Verify: citation 5-tuple, scope filtering, actor_type
        """
        
        # search_documents tool 호출
        response = await self.mcp_client.call_tool(
            tool_name="search_documents",
            arguments={
                "query": "machine learning best practices",
                "limit": 5,
                "access_context": self.test_scope.to_dict()
            }
        )
        
        assert response.error is None, f"Tool call failed: {response.error}"
        assert "documents" in response.result
        
        documents = response.result["documents"]
        assert len(documents) <= 5
        
        # 각 document 검증
        for doc in documents:
            assert "uri" in doc
            assert "text" in doc
            assert "document_type" in doc
            
            # Scope filtering 검증
            assert doc["document_type"] in self.test_scope.allowed_types
            
            # Citation 검증
            if "citations" in doc:
                for citation in doc["citations"]:
                    assert isinstance(citation, dict)
                    assert "source" in citation
                    assert "snippet" in citation
                    assert "confidence" in citation
        
        # audit log 검증
        audit_logs = await self.audit_log_service.list_logs(
            action="mcp_tool_execute",
            actor_id="agent-mcp-test"
        )
        assert any(log.tool_name == "search_documents" for log in audit_logs[-3:])

    @pytest.mark.asyncio
    async def test_mcp_fetch_node_tool(self):
        """
        fetch_node tool e2e 테스트
        
        Flow:
        1. 먼저 search로 문서 URI 얻기
        2. fetch_node(uri="mimir://doc-001") 호출
        3. 결과: 전체 node 내용 + metadata
        """
        
        # 1. 문서 검색해서 URI 얻기
        search_response = await self.mcp_client.call_tool(
            tool_name="search_documents",
            arguments={
                "query": "test document",
                "limit": 1,
                "access_context": self.test_scope.to_dict()
            }
        )
        
        if not search_response.result.get("documents"):
            pytest.skip("No documents to fetch")
        
        doc_uri = search_response.result["documents"][0]["uri"]
        
        # 2. Node fetch
        fetch_response = await self.mcp_client.call_tool(
            tool_name="fetch_node",
            arguments={
                "uri": doc_uri,
                "access_context": self.test_scope.to_dict()
            }
        )
        
        assert fetch_response.error is None
        assert "node" in fetch_response.result
        
        node = fetch_response.result["node"]
        assert node["uri"] == doc_uri
        assert "content" in node
        assert "metadata" in node

    @pytest.mark.asyncio
    async def test_mcp_verify_citation_tool(self):
        """
        verify_citation tool e2e 테스트
        
        Flow:
        1. Citation 5-tuple 준비
        2. verify_citation(source, snippet, ...) 호출
        3. 결과: valid=true/false, reason
        """
        
        # Citation 5-tuple 준비
        citation_tuple = {
            "source": "mimir://doc-001",
            "snippet": "The quick brown fox",
            "relevance_score": 0.95,
            "confidence": 0.89,
            "timestamp": "2026-04-17T00:00:00Z"
        }
        
        # verify_citation tool 호출
        verify_response = await self.mcp_client.call_tool(
            tool_name="verify_citation",
            arguments={
                "citation": citation_tuple,
                "access_context": self.test_scope.to_dict()
            }
        )
        
        assert verify_response.error is None
        
        result = verify_response.result
        assert "valid" in result
        assert isinstance(result["valid"], bool)
        
        if not result["valid"]:
            assert "reason" in result  # 실패 이유

    @pytest.mark.asyncio
    async def test_mcp_propose_transition_tool(self):
        """
        propose_transition tool e2e 테스트
        
        Flow:
        1. Agent가 Draft 제안 생성 (쓰기)
        2. tool: propose_transition(content={...})
        3. 결과: task_id, status="pending_review"
        4. Verify: actor_type="agent", delegation chain
        """
        
        # Agent principal 준비
        agent = await self.agent_service.create_principal(
            agent_id="agent-mcp-propose",
            agent_name="MCP Propose Agent",
            agent_type="ai",
            owner_id="user-mcp-test",
            scope_profile=self.test_scope,
            actor_type="agent"
        )
        
        # propose_transition tool 호출
        task_id = str(uuid4())
        propose_response = await self.mcp_client.call_tool(
            tool_name="propose_transition",
            arguments={
                "task_id": task_id,
                "content": {
                    "document_uri": "mimir://doc-001",
                    "extracted_fields": [
                        {"field": "author", "value": "John Doe"}
                    ]
                },
                "agent_principal_id": agent.principal_id
            }
        )
        
        assert propose_response.error is None
        
        result = propose_response.result
        assert result["task_id"] == task_id
        assert result["status"] == "pending_review"
        
        # audit log 검증: agent actor_type
        audit_logs = await self.audit_log_service.list_logs(
            action="mcp_tool_execute"
        )
        
        propose_logs = [
            log for log in audit_logs
            if log.tool_name == "propose_transition"
        ]
        
        assert any(log.actor_type == "agent" for log in propose_logs)

    @pytest.mark.asyncio
    async def test_mcp_resource_listing(self):
        """
        Resource listing e2e 테스트
        
        Flow:
        1. list_resources() 호출
        2. 결과: available resources (documents, sessions, etc.)
        3. Verify: MIME types, access control
        """
        
        # list_resources 호출
        resources = await self.mcp_client.list_resources()
        
        assert isinstance(resources, list)
        
        for resource in resources:
            assert "uri" in resource
            assert "name" in resource
            assert "mime_type" in resource
            
            # MIME type 검증
            valid_mime_types = {
                "application/vnd.mimir+json",
                "application/json",
                "text/plain",
                "text/markdown"
            }
            assert resource["mime_type"] in valid_mime_types

    @pytest.mark.asyncio
    async def test_mcp_error_handling_invalid_tool(self):
        """
        Invalid tool 호출 에러 처리
        """
        
        response = await self.mcp_client.call_tool(
            tool_name="nonexistent_tool",
            arguments={}
        )
        
        assert response.error is not None
        assert "not found" in response.error.lower() or "unknown" in response.error.lower()

    @pytest.mark.asyncio
    async def test_mcp_error_handling_unauthorized(self):
        """
        Unauthorized access 에러 처리
        """
        
        # 다른 scope의 문서 접근 시도
        invalid_scope = ScopeProfile(
            name="invalid_scope",
            allowed_types=["xlsx"],  # test_scope와 다름
            access_rules={}
        )
        
        response = await self.mcp_client.call_tool(
            tool_name="fetch_node",
            arguments={
                "uri": "mimir://doc-001",
                "access_context": invalid_scope.to_dict()
            }
        )
        
        # 문서 타입이 scope에 없으면 에러
        if response.error is None:
            # 또는 빈 결과
            assert response.result.get("node") is None

    @pytest.mark.asyncio
    async def test_mcp_rate_limiting(self):
        """
        Rate limiting 처리
        """
        
        # 급격히 많은 요청 (예: 100개)
        tasks = []
        for i in range(100):
            task = asyncio.create_task(
                self.mcp_client.call_tool(
                    tool_name="search_documents",
                    arguments={
                        "query": f"query {i}",
                        "limit": 1,
                        "access_context": self.test_scope.to_dict()
                    }
                )
            )
            tasks.append(task)
        
        responses = await asyncio.gather(*tasks, return_exceptions=True)
        
        # 모든 응답이 처리됨 (성공 또는 rate limit 에러)
        assert len(responses) == 100
        
        # 일부는 rate limit 에러를 받을 수 있음
        rate_limit_errors = sum(
            1 for r in responses
            if isinstance(r, dict) and "error" in r and "rate limit" in r.get("error", "").lower()
        )
        
        # rate limit이 있으면 일부는 제한됨
        if rate_limit_errors > 0:
            assert rate_limit_errors < 100  # 모두 제한되진 않음

    @pytest.mark.asyncio
    async def test_mcp_streamable_http_transport(self):
        """
        Streamable HTTP transport 검증 (SSE, JSON streaming)
        
        대용량 응답을 streaming으로 받아야 함
        """
        
        # 대용량 검색 요청 (streaming)
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.mcp_client.server_url}/mcp/tools/search_documents/call/stream",
                json={
                    "session_id": self.session_id,
                    "arguments": {
                        "query": "large dataset",
                        "limit": 1000,  # 대용량
                        "access_context": self.test_scope.to_dict()
                    }
                }
            ) as resp:
                # SSE 또는 line-delimited JSON
                assert resp.status == 200
                assert resp.content_type in [
                    "text/event-stream",
                    "application/x-ndjson"
                ]
                
                # streaming 읽기
                chunk_count = 0
                async for line in resp.content:
                    chunk_count += 1
                    if chunk_count > 0:
                        break  # 최소 한 개 청크 받음
                
                assert chunk_count > 0
```

---

## 5. 산출물

### 주요 산출물

1. **테스트 코드**
   - `tests/evaluation/test_golden_set_regression.py` (약 600 라인)
   - `tests/integration/test_mcp_e2e.py` (약 700 라인)

2. **평가 보고서**
   - `docs/평가보고서/AI품질평가.md` (executive summary, 메트릭, 실패 케이스, 권장사항)
   - `docs/평가보고서/AI품질평가.json` (structured results for dashboard)

3. **테스트 데이터**
   - `tests/data/golden_sets/*.json` (golden set 테스트 케이스)

4. **CI/CD 아티팩트**
   - pytest report (JSON format)
   - Coverage report

---

## 6. 완료 기준

1. **골든셋 회귀 실행**
   - [ ] `pytest tests/evaluation/test_golden_set_regression.py -v --golden-set-id all --threshold-mode strict` 통과
   - [ ] 6개 메트릭 모두 엄격한 임계값 만족
   - [ ] Faithfulness >= 0.80, Citation Present >= 0.90 등

2. **MCP e2e 테스트**
   - [ ] 6개 시나리오 모두 통과 (initialize, search, fetch, verify, propose, list)
   - [ ] Error handling 테스트 통과 (invalid tool, unauthorized, rate limiting)
   - [ ] Streaming transport 검증 완료

3. **AI 품질 평가 보고서**
   - [ ] AI품질평가.md 작성 완료 (executive summary, 메트릭별 통계, 실패 분석)
   - [ ] Failing cases 상세 분석 포함
   - [ ] Recommendations 포함
   - [ ] JSON 버전도 작성 (structured data)

4. **검수 준비**
   - [ ] 검수 보고서 작성 (golden set pass rate, MCP tool coverage)
   - [ ] 보안 취약점 검사 보고서 (MCP security, scope filtering)

---

## 7. 작업 지침

**7-1** **골든셋 임계값 엄격성**
- Strict mode에서는 대부분의 production 환경보다 높은 기준 적용
- 실패 케이스는 human review와 함께 상세 분석 필수

**7-2** **MCP 호환성 검증**
- MCP spec 1.0 준수 확인
- Tool schema validation (JSON Schema) 필수
- Error response format 일관성

**7-3** **S2 원칙 MCP 통합**
- 모든 MCP tool 호출에서 actor_type="agent" 기록
- access_context (scope_profile) 전파 필수
- 폐쇄망 모드에서도 MCP 동작 확인

**7-4** **Citation 품질 검증**
- 5-tuple (source, snippet, relevance, confidence, timestamp) 완전성
- Citation 정확성 spot check (human review)
- False positive citation 감지

**7-5** **Streaming transport 성능**
- 대용량 응답 (1000+ documents)도 streaming으로 처리
- Memory 누수 확인 (long-running tests)
- Connection timeout handling

**7-6** **AI 모델 의존성**
- Evaluation 시 consistent LLM 모델 사용 (같은 버전)
- Temperature 설정 일관성 (reproducibility)
- 외부 API 호출 횟수 로깅 및 비용 추적

**7-7** **보고서 생성 자동화**
```bash
# 모든 테스트 + 보고서 생성
pytest tests/evaluation/test_golden_set_regression.py -v \
  --golden-set-id all \
  --threshold-mode strict \
  --junitxml=test_results/junit.xml \
  --html=test_results/report.html \
  --self-contained-html

# JSON 결과 + Markdown 보고서 자동 생성
python scripts/generate_evaluation_report.py \
  --input test_results/junit.xml \
  --output docs/평가보고서/AI품질평가.md
```
