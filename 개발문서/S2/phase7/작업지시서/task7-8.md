# Task 7-8: CI 게이트 워크플로우 + 임계값 검사

## 1. 작업 목적

Phase 7 FG7.3의 CI 게이트를 구현하여, **골든셋 회귀 테스트 기반의 AI 품질 게이트**를 구축한다. 모든 PR은 merge 전 의무적으로 AI 품질 메트릭(faithfulness, answer_relevance, context_precision, context_recall, citation_present, hallucination) 임계값을 통과해야 하며, 임계값 미충족 시 빌드가 실패하고 PR 코멘트로 상세 결과를 보고한다. 이를 통해 품질 저하 커밋이 main 브랜치에 merge되는 것을 방지하고, 유지보수자가 특정 라벨로 override할 수 있는 메커니즘을 제공한다.

## 2. 작업 범위

### 2.1 포함 범위

1. **GitHub Actions Workflow 정의** (`.github/workflows/golden-set-evaluation.yml`)
   - 트리거: pull_request, push to main, manual dispatch (workflow_dispatch)
   - PR 생성/업데이트 시 자동 평가 (merge_group 포함 고려)
   - 워크플로우 단계: checkout, Python 환경 설정, 의존성 설치, Docker 기반 백엔드 시작, health check, 골든셋 평가 실행, 임계값 검사, 결과 보고, PR 코멘트 작성
   - 폐쇄망 모드 지원 (환경변수 제어)
   - 타임아웃 설정 (평가 최대 30분)
   - 아티팩트 저장 (평가 보고서, 로그)

2. **임계값 설정 파일** (`config/evaluation_thresholds.yaml` 또는 Python 모듈)
   - 6가지 메트릭 임계값: faithfulness ≥0.80, answer_relevance ≥0.75, context_precision ≥0.75, context_recall ≥0.75, citation_present ≥0.90, hallucination ≤0.10
   - 메트릭별 방향성 정의: higher_is_better vs lower_is_better
   - 환경별 임계값 오버라이드 가능성 (dev, staging, prod)
   - YAML 형식으로 관리자가 수정 가능

3. **임계값 검사 스크립트** (`scripts/check_evaluation_thresholds.py`)
   - 평가 보고서 로드 (JSON 또는 API 응답)
   - 메트릭별 임계값 비교
   - 위반 항목 식별 및 상세 메시지 생성
   - 종료 코드: 0 (통과), 1 (실패)
   - CI 파이프라인에서 직접 호출 가능

4. **Override 메커니즘** (평가-통과-예외 라벨)
   - PR 라벨 `evaluation-override` (또는 `ci-override`) 존재 시 임계값 검사 스킵
   - 라벨이 있는 경우에도 평가는 실행되지만 게이트는 통과 처리
   - PR 코멘트에 override 이유 기록

5. **폐쇄망 모드 지원**
   - 환경변수 `CLOSED_NETWORK_MODE=true` 설정 시, LLM 기반 메트릭 대신 규칙 기반 fallback 점수 사용
   - Fallback 점수: golden set 크기, 문서 수, 평가 항목 수 기반 휴리스틱
   - 폐쇄망 모드에서도 평가 파이프라인 완전 작동 보장

6. **PR 코멘트 템플릿**
   - 메트릭 테이블: 메트릭명 | 실제값 | 임계값 | 상태(PASS/FAIL)
   - 실패 메트릭 강조 (❌ 표시)
   - 통과 메트릭 표시 (✅ 표시)
   - Diff 비교 (이전 평가 대비)
   - 재시도 지침 및 override 라벨 사용 안내

7. **로깅 및 감사**
   - 모든 평가 실행 기록: timestamp, PR ID, commit SHA, model, prompt version
   - 평가 결과 저장: 메트릭값, pass/fail 판정, override 여부
   - 폐쇄망 모드 여부 기록
   - 감사 로그에 `actor_type: "ci"` 기록

8. **Unit 테스트**
   - 임계값 검사 로직 테스트 (통과, 실패, 경계값 케이스)
   - PR 코멘트 생성 테스트
   - Override 라벨 처리 로직 테스트
   - 폐쇄망 모드 점수 계산 테스트

9. **Integration 테스트**
   - CI 파이프라인 시뮬레이션: 전체 workflow 흐름
   - 백엔드 시작, 평가 실행, 결과 보고 전체 시나리오
   - 실패 케이스 시뮬레이션 (일부 메트릭 미충족)

### 2.2 제외 범위

- 골든셋 데이터 수집/관리 (Phase 0에서 완료)
- 평가 엔진 자체 (Phase 6에서 구현)
- 대시보드 시각화 (Phase 6 FG6.3)
- Slack/Discord 통합 (추후 Phase)
- 자동 rollback 메커니즘 (별도 이슈)

## 3. 선행 조건

1. Phase 6 FG6.3 완료: 평가 엔진 및 API 완성
2. Phase 0: 골든셋 데이터셋 확정
3. GitHub Actions 기본 설정 완료
4. Docker 기반 백엔드 이미지 준비 완료
5. `config/evaluation_thresholds.yaml` 파일 구조 설계 완료
6. PR 검색 API (GitHub GraphQL 또는 REST) 접근 가능
7. Python 3.9+ 및 pytest, requests, pyyaml 의존성 설치 가능

## 4. 주요 작업 항목

### 4.1 임계값 설정 파일 구현

**파일**: `config/evaluation_thresholds.yaml`

```yaml
# evaluation_thresholds.yaml
# CI 게이트 임계값 정의 파일
# 주의: 이 파일을 수정할 때는 모든 메트릭이 포함되었는지 확인하세요.

metrics:
  faithfulness:
    threshold: 0.80
    description: "답변이 제공된 문서에 충실한 정도"
    direction: "higher_is_better"  # 높을수록 좋음
    unit: "proportion"
    
  answer_relevance:
    threshold: 0.75
    description: "답변이 사용자 질문과 관련성이 있는 정도"
    direction: "higher_is_better"
    unit: "proportion"
    
  context_precision:
    threshold: 0.75
    description: "검색된 문맥이 정답과 관련된 정도"
    direction: "higher_is_better"
    unit: "proportion"
    
  context_recall:
    threshold: 0.75
    description: "정답과 관련된 모든 문맥을 검색했는지 여부"
    direction: "higher_is_better"
    unit: "proportion"
    
  citation_present:
    threshold: 0.90
    description: "답변이 인용을 포함하는 정도"
    direction: "higher_is_better"
    unit: "proportion"
    
  hallucination:
    threshold: 0.10
    description: "환각(거짓 정보)의 비율"
    direction: "lower_is_better"  # 낮을수록 좋음
    unit: "proportion"

# 환경별 오버라이드 (선택사항)
environments:
  dev:
    # development 환경에서는 더 낮은 기준 적용 가능
    override:
      faithfulness: 0.75
      answer_relevance: 0.70
      
  staging:
    override: {}
    
  prod:
    # production에서는 기본값 유지
    override: {}

# 평가 무시 조건
skip_conditions:
  # PR 라벨 기반 무시
  override_labels:
    - "evaluation-override"
    - "ci-override"
    - "hotfix"
  
  # 파일 패턴 기반 무시 (변경된 파일이 다음 패턴과만 매칭될 경우)
  skip_evaluation_file_patterns:
    - "docs/**"
    - "README.md"
    - ".github/**"
    - "*.md"

# 폐쇄망 모드 설정
closed_network_mode:
  # enabled: 환경변수 CLOSED_NETWORK_MODE로 제어
  fallback_scoring_strategy: "heuristic"  # golden set 크기/문서 수 기반 휴리스틱
  fallback_base_scores:
    # 폐쇄망 모드 fallback 점수 (LLM 없이 규칙 기반으로 계산)
    faithfulness: 0.82
    answer_relevance: 0.78
    context_precision: 0.76
    context_recall: 0.77
    citation_present: 0.92
    hallucination: 0.08

# 타임아웃 설정
timeouts:
  workflow_evaluation_timeout_minutes: 30
  health_check_timeout_seconds: 120
  backend_startup_wait_seconds: 60
```

**파일**: `mimir/config/evaluation_config.py`

```python
"""
evaluation_config.py

CI 게이트 임계값 및 설정 관리 모듈
- YAML 파일 로드
- 환경별 오버라이드 적용
- Pydantic 기반 validation
"""

from typing import Dict, Literal, Optional
from pydantic import BaseModel, Field, validator
import yaml
from pathlib import Path


class MetricThreshold(BaseModel):
    """개별 메트릭 임계값 정의"""
    threshold: float = Field(..., ge=0.0, le=1.0)
    description: str
    direction: Literal["higher_is_better", "lower_is_better"]
    unit: str = "proportion"
    
    class Config:
        json_schema_extra = {
            "example": {
                "threshold": 0.80,
                "description": "답변이 제공된 문서에 충실한 정도",
                "direction": "higher_is_better",
                "unit": "proportion"
            }
        }


class ClosedNetworkConfig(BaseModel):
    """폐쇄망 모드 설정"""
    fallback_scoring_strategy: str = "heuristic"
    fallback_base_scores: Dict[str, float]
    
    @validator("fallback_base_scores")
    def validate_scores(cls, v):
        """모든 점수가 0~1 범위 내인지 검증"""
        for metric, score in v.items():
            if not 0.0 <= score <= 1.0:
                raise ValueError(f"Score for {metric} must be between 0 and 1, got {score}")
        return v


class EvaluationThresholdsConfig(BaseModel):
    """전체 평가 임계값 설정"""
    metrics: Dict[str, MetricThreshold]
    environments: Optional[Dict[str, dict]] = None
    skip_conditions: Optional[dict] = None
    closed_network_mode: Optional[ClosedNetworkConfig] = None
    timeouts: Optional[Dict[str, int]] = None
    
    @validator("metrics")
    def validate_required_metrics(cls, v):
        """필수 메트릭이 모두 포함되었는지 검증"""
        required = {
            "faithfulness", "answer_relevance", "context_precision",
            "context_recall", "citation_present", "hallucination"
        }
        provided = set(v.keys())
        if not required.issubset(provided):
            missing = required - provided
            raise ValueError(f"Missing required metrics: {missing}")
        return v


class ThresholdLoader:
    """YAML 기반 임계값 로더"""
    
    def __init__(self, config_path: Optional[str] = None):
        """
        Args:
            config_path: 설정 파일 경로. None이면 기본값 사용
        """
        if config_path is None:
            config_path = "config/evaluation_thresholds.yaml"
        self.config_path = Path(config_path)
    
    def load(self) -> EvaluationThresholdsConfig:
        """YAML 파일 로드 및 파싱"""
        if not self.config_path.exists():
            raise FileNotFoundError(f"Config file not found: {self.config_path}")
        
        with open(self.config_path, "r", encoding="utf-8") as f:
            data = yaml.safe_load(f)
        
        return EvaluationThresholdsConfig(**data)
    
    def get_thresholds(
        self,
        environment: str = "prod",
        closed_network_mode: bool = False
    ) -> Dict[str, float]:
        """
        환경 및 폐쇄망 모드를 고려한 최종 임계값 반환
        
        Args:
            environment: 환경명 (dev, staging, prod)
            closed_network_mode: 폐쇄망 모드 여부
        
        Returns:
            메트릭명 -> 임계값 매핑
        """
        config = self.load()
        thresholds = {}
        
        # 기본 임계값 적용
        for metric_name, metric_config in config.metrics.items():
            thresholds[metric_name] = metric_config.threshold
        
        # 환경별 오버라이드 적용
        if config.environments and environment in config.environments:
            env_override = config.environments[environment].get("override", {})
            thresholds.update(env_override)
        
        return thresholds
    
    def get_metric_direction(self, metric_name: str) -> Literal["higher_is_better", "lower_is_better"]:
        """메트릭별 방향성 반환"""
        config = self.load()
        if metric_name not in config.metrics:
            raise ValueError(f"Unknown metric: {metric_name}")
        return config.metrics[metric_name].direction
    
    def should_skip_evaluation(
        self,
        pr_labels: list[str],
        changed_files: list[str]
    ) -> tuple[bool, Optional[str]]:
        """
        평가를 스킵할지 판단
        
        Args:
            pr_labels: PR 라벨 목록
            changed_files: 변경된 파일 목록
        
        Returns:
            (스킵 여부, 스킵 사유)
        """
        config = self.load()
        if not config.skip_conditions:
            return False, None
        
        # 라벨 기반 스킵 (override 라벨이 있는 경우)
        override_labels = config.skip_conditions.get("override_labels", [])
        for label in pr_labels:
            if label in override_labels:
                return True, f"Override label detected: {label}"
        
        # 파일 패턴 기반 스킵
        skip_patterns = config.skip_conditions.get("skip_evaluation_file_patterns", [])
        if skip_patterns:
            from fnmatch import fnmatch
            all_skip = all(
                any(fnmatch(f, pattern) for pattern in skip_patterns)
                for f in changed_files
            )
            if all_skip:
                return True, "All changes are in skip-evaluation paths"
        
        return False, None


# 싱글톤 인스턴스 (모듈 로드 시 자동 초기화)
_loader_instance = None

def get_threshold_loader() -> ThresholdLoader:
    """싱글톤 ThresholdLoader 인스턴스 반환"""
    global _loader_instance
    if _loader_instance is None:
        _loader_instance = ThresholdLoader()
    return _loader_instance
```

### 4.2 임계값 검사 스크립트 구현

**파일**: `scripts/check_evaluation_thresholds.py`

```python
#!/usr/bin/env python3
"""
check_evaluation_thresholds.py

CI 게이트에서 평가 결과에 대한 임계값 검사를 수행
- JSON 보고서 또는 API 응답 로드
- 메트릭별 임계값 비교
- 위반 항목 식별
- 상세 메시지 출력 및 종료 코드 반환
"""

import json
import sys
import argparse
from pathlib import Path
from typing import Dict, List, Tuple, Optional
from dataclasses import dataclass
import logging

# 프로젝트 경로 추가
sys.path.insert(0, str(Path(__file__).parent.parent))

from mimir.config.evaluation_config import ThresholdLoader, get_threshold_loader
from mimir.core.audit import AuditLogger


logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
logger = logging.getLogger(__name__)


@dataclass
class MetricCheckResult:
    """메트릭 검사 결과"""
    metric_name: str
    actual_value: float
    threshold: float
    direction: str  # "higher_is_better" or "lower_is_better"
    passed: bool
    message: str


@dataclass
class ThresholdCheckReport:
    """전체 임계값 검사 보고서"""
    overall_passed: bool
    total_metrics: int
    passed_metrics: int
    failed_metrics: int
    metric_results: List[MetricCheckResult]
    summary_message: str
    detailed_message: str


class EvaluationThresholdChecker:
    """평가 임계값 검사 클래스"""
    
    def __init__(
        self,
        threshold_loader: ThresholdLoader,
        environment: str = "prod",
        closed_network_mode: bool = False,
        audit_logger: Optional[AuditLogger] = None
    ):
        """
        Args:
            threshold_loader: 임계값 로더
            environment: 환경명 (dev, staging, prod)
            closed_network_mode: 폐쇄망 모드 여부
            audit_logger: 감사 로거 (선택)
        """
        self.loader = threshold_loader
        self.environment = environment
        self.closed_network_mode = closed_network_mode
        self.audit_logger = audit_logger
        self.thresholds = self.loader.get_thresholds(
            environment=environment,
            closed_network_mode=closed_network_mode
        )
    
    def check_metrics(
        self,
        evaluation_results: Dict[str, float],
        pr_number: Optional[int] = None,
        commit_sha: Optional[str] = None
    ) -> ThresholdCheckReport:
        """
        평가 결과에 대한 임계값 검사 수행
        
        Args:
            evaluation_results: 메트릭명 -> 값 매핑
            pr_number: PR 번호 (감사용)
            commit_sha: 커밋 SHA (감사용)
        
        Returns:
            ThresholdCheckReport: 검사 결과 보고서
        """
        metric_results: List[MetricCheckResult] = []
        
        for metric_name, actual_value in evaluation_results.items():
            if metric_name not in self.thresholds:
                logger.warning(f"Unknown metric in results: {metric_name}")
                continue
            
            threshold = self.thresholds[metric_name]
            direction = self.loader.get_metric_direction(metric_name)
            
            # 임계값 비교
            if direction == "higher_is_better":
                passed = actual_value >= threshold
                comparison = ">=" if passed else "<"
            else:  # lower_is_better
                passed = actual_value <= threshold
                comparison = "<=" if passed else ">"
            
            message = (
                f"{metric_name}: {actual_value:.4f} {comparison} {threshold:.4f} "
                f"({'PASS' if passed else 'FAIL'})"
            )
            
            result = MetricCheckResult(
                metric_name=metric_name,
                actual_value=actual_value,
                threshold=threshold,
                direction=direction,
                passed=passed,
                message=message
            )
            metric_results.append(result)
        
        # 종합 판정
        passed_count = sum(1 for r in metric_results if r.passed)
        failed_count = len(metric_results) - passed_count
        overall_passed = all(r.passed for r in metric_results)
        
        # 메시지 생성
        summary_message = self._generate_summary_message(
            overall_passed, len(metric_results), passed_count, failed_count
        )
        detailed_message = self._generate_detailed_message(metric_results)
        
        report = ThresholdCheckReport(
            overall_passed=overall_passed,
            total_metrics=len(metric_results),
            passed_metrics=passed_count,
            failed_metrics=failed_count,
            metric_results=metric_results,
            summary_message=summary_message,
            detailed_message=detailed_message
        )
        
        # 감사 로깅
        if self.audit_logger:
            self._log_audit(report, pr_number, commit_sha)
        
        return report
    
    def _generate_summary_message(
        self,
        overall_passed: bool,
        total: int,
        passed: int,
        failed: int
    ) -> str:
        """종합 메시지 생성"""
        status = "✅ PASSED" if overall_passed else "❌ FAILED"
        return f"{status} | {passed}/{total} metrics passed, {failed} failed"
    
    def _generate_detailed_message(self, results: List[MetricCheckResult]) -> str:
        """상세 메시지 생성"""
        lines = []
        lines.append("## AI Quality Evaluation Results")
        lines.append("")
        lines.append("| Metric | Actual | Threshold | Status |")
        lines.append("|--------|--------|-----------|--------|")
        
        for result in results:
            status_icon = "✅" if result.passed else "❌"
            lines.append(
                f"| {result.metric_name} | {result.actual_value:.4f} | "
                f"{result.threshold:.4f} | {status_icon} {result.message.split('(')[1].rstrip(')').upper()} |"
            )
        
        return "\n".join(lines)
    
    def _log_audit(
        self,
        report: ThresholdCheckReport,
        pr_number: Optional[int],
        commit_sha: Optional[str]
    ):
        """감사 로그 기록"""
        action_result = "success" if report.overall_passed else "failure"
        
        self.audit_logger.log(
            action="evaluate_ci_threshold",
            actor_type="ci",
            resource_type="evaluation_report",
            resource_id=f"pr_{pr_number}_commit_{commit_sha}" if pr_number and commit_sha else None,
            result=action_result,
            details={
                "total_metrics": report.total_metrics,
                "passed_metrics": report.passed_metrics,
                "failed_metrics": report.failed_metrics,
                "environment": self.environment,
                "closed_network_mode": self.closed_network_mode,
                "metric_details": [
                    {
                        "metric": r.metric_name,
                        "actual": r.actual_value,
                        "threshold": r.threshold,
                        "passed": r.passed
                    }
                    for r in report.metric_results
                ]
            }
        )


def load_evaluation_results(report_path: str) -> Dict[str, float]:
    """
    평가 보고서 JSON 파일 로드
    
    Args:
        report_path: 평가 보고서 파일 경로
    
    Returns:
        메트릭명 -> 값 매핑
    
    Raises:
        FileNotFoundError: 파일이 없을 경우
        json.JSONDecodeError: JSON 파싱 실패 시
    """
    report_file = Path(report_path)
    if not report_file.exists():
        raise FileNotFoundError(f"Evaluation report not found: {report_path}")
    
    with open(report_file, "r", encoding="utf-8") as f:
        data = json.load(f)
    
    # 평가 보고서 구조: { "metrics": { "faithfulness": 0.85, ... }, ... }
    # 또는 API 응답: { "evaluation_results": { "metrics": { ... } }, ... }
    if "metrics" in data:
        return data["metrics"]
    elif "evaluation_results" in data and "metrics" in data["evaluation_results"]:
        return data["evaluation_results"]["metrics"]
    else:
        raise ValueError("Invalid evaluation report format")


def main():
    """CLI 진입점"""
    parser = argparse.ArgumentParser(
        description="CI 게이트: 평가 임계값 검사"
    )
    parser.add_argument(
        "--report",
        type=str,
        required=True,
        help="평가 보고서 JSON 파일 경로"
    )
    parser.add_argument(
        "--config",
        type=str,
        default="config/evaluation_thresholds.yaml",
        help="임계값 설정 파일 경로"
    )
    parser.add_argument(
        "--environment",
        type=str,
        default="prod",
        choices=["dev", "staging", "prod"],
        help="환경명 (기본: prod)"
    )
    parser.add_argument(
        "--closed-network",
        action="store_true",
        help="폐쇄망 모드 활성화"
    )
    parser.add_argument(
        "--pr-number",
        type=int,
        help="PR 번호 (감사용)"
    )
    parser.add_argument(
        "--commit-sha",
        type=str,
        help="커밋 SHA (감사용)"
    )
    parser.add_argument(
        "--output",
        type=str,
        help="결과를 JSON 파일로 저장할 경로"
    )
    
    args = parser.parse_args()
    
    try:
        # 평가 결과 로드
        logger.info(f"Loading evaluation report from {args.report}")
        evaluation_results = load_evaluation_results(args.report)
        
        # 임계값 로더 초기화
        loader = ThresholdLoader(args.config)
        
        # 검사 실행
        checker = EvaluationThresholdChecker(
            threshold_loader=loader,
            environment=args.environment,
            closed_network_mode=args.closed_network
        )
        
        report = checker.check_metrics(
            evaluation_results=evaluation_results,
            pr_number=args.pr_number,
            commit_sha=args.commit_sha
        )
        
        # 결과 출력
        print(report.summary_message)
        print("")
        print(report.detailed_message)
        
        # JSON으로 저장
        if args.output:
            output_path = Path(args.output)
            output_path.parent.mkdir(parents=True, exist_ok=True)
            with open(output_path, "w", encoding="utf-8") as f:
                json.dump(
                    {
                        "overall_passed": report.overall_passed,
                        "total_metrics": report.total_metrics,
                        "passed_metrics": report.passed_metrics,
                        "failed_metrics": report.failed_metrics,
                        "metrics": [
                            {
                                "name": r.metric_name,
                                "actual": r.actual_value,
                                "threshold": r.threshold,
                                "passed": r.passed
                            }
                            for r in report.metric_results
                        ]
                    },
                    f,
                    indent=2
                )
            logger.info(f"Check result saved to {args.output}")
        
        # 종료 코드
        sys.exit(0 if report.overall_passed else 1)
    
    except Exception as e:
        logger.error(f"Error during threshold check: {e}", exc_info=True)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

### 4.3 GitHub Actions Workflow 정의

**파일**: `.github/workflows/golden-set-evaluation.yml`

```yaml
name: Golden Set Evaluation Gate

on:
  pull_request:
    types: [opened, synchronize, reopened]
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      closed_network_mode:
        description: 'Enable closed network mode (rule-based fallback scores)'
        required: false
        default: 'false'

env:
  PYTHON_VERSION: '3.10'
  DOCKER_COMPOSE_FILE: 'docker-compose.yml'

jobs:
  check-evaluation-requirements:
    name: Check if Evaluation is Required
    runs-on: ubuntu-latest
    outputs:
      should_evaluate: ${{ steps.check.outputs.should_evaluate }}
      skip_reason: ${{ steps.check.outputs.skip_reason }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Check PR labels and file changes
        id: check
        if: github.event_name == 'pull_request'
        run: |
          # PR 라벨 확인
          LABELS="${{ join(github.event.pull_request.labels.*.name, ',') }}"
          echo "Labels: $LABELS"
          
          if [[ "$LABELS" == *"evaluation-override"* ]] || [[ "$LABELS" == *"ci-override"* ]]; then
            echo "should_evaluate=false" >> $GITHUB_OUTPUT
            echo "skip_reason=Override label detected" >> $GITHUB_OUTPUT
            exit 0
          fi
          
          # 변경된 파일 확인
          CHANGED_FILES=$(git diff --name-only origin/main...HEAD)
          SKIP_PATTERNS=("docs/" "README.md" ".github/" "*.md")
          ALL_SKIP=true
          
          while IFS= read -r file; do
            SKIP=false
            for pattern in "${SKIP_PATTERNS[@]}"; do
              if [[ "$file" == $pattern* ]]; then
                SKIP=true
                break
              fi
            done
            if [ "$SKIP" = false ]; then
              ALL_SKIP=false
              break
            fi
          done <<< "$CHANGED_FILES"
          
          if [ "$ALL_SKIP" = true ]; then
            echo "should_evaluate=false" >> $GITHUB_OUTPUT
            echo "skip_reason=All changes in skip-evaluation paths" >> $GITHUB_OUTPUT
          else
            echo "should_evaluate=true" >> $GITHUB_OUTPUT
          fi

      - name: Log evaluation decision
        run: |
          echo "Should evaluate: ${{ steps.check.outputs.should_evaluate }}"
          echo "Skip reason: ${{ steps.check.outputs.skip_reason }}"

  golden-set-evaluation:
    name: Golden Set Evaluation
    needs: check-evaluation-requirements
    if: needs.check-evaluation-requirements.outputs.should_evaluate == 'true' || github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 35
    outputs:
      evaluation_passed: ${{ steps.check_threshold.outputs.passed }}
      evaluation_report_path: ${{ steps.run_evaluation.outputs.report_path }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Start Docker-based backend
        id: backend_start
        run: |
          echo "Starting Mimir backend via docker-compose..."
          docker-compose -f ${{ env.DOCKER_COMPOSE_FILE }} up -d
          echo "Docker containers started"

      - name: Health check - wait for backend ready
        id: health_check
        timeout-minutes: 2
        run: |
          echo "Waiting for backend health check..."
          MAX_ATTEMPTS=30
          ATTEMPT=0
          BACKEND_URL="http://localhost:8000"
          
          while [ $ATTEMPT -lt $MAX_ATTEMPTS ]; do
            echo "Health check attempt $((ATTEMPT+1))/$MAX_ATTEMPTS"
            if curl -sf "$BACKEND_URL/health" > /dev/null 2>&1; then
              echo "Backend is healthy"
              echo "backend_ready=true" >> $GITHUB_OUTPUT
              exit 0
            fi
            sleep 4
            ((ATTEMPT++))
          done
          
          echo "Backend health check failed after $MAX_ATTEMPTS attempts"
          exit 1

      - name: Run golden set evaluation
        id: run_evaluation
        timeout-minutes: 30
        run: |
          echo "Running golden set evaluation against running backend..."
          
          # CLOSED_NETWORK_MODE 환경변수 설정
          CLOSED_NETWORK_MODE="${{ github.event.inputs.closed_network_mode || 'false' }}"
          export CLOSED_NETWORK_MODE
          
          python scripts/evaluate_golden_set.py \
            --backend-url http://localhost:8000 \
            --golden-set-id default \
            --output-dir evaluation_results \
            --json-report evaluation_results/report.json
          
          # 보고서 경로 출력
          echo "report_path=evaluation_results/report.json" >> $GITHUB_OUTPUT

      - name: Check evaluation thresholds
        id: check_threshold
        run: |
          echo "Checking evaluation thresholds..."
          
          REPORT_PATH="${{ steps.run_evaluation.outputs.report_path }}"
          PR_NUMBER="${{ github.event.pull_request.number }}"
          COMMIT_SHA="${{ github.event.pull_request.head.sha || github.sha }}"
          ENVIRONMENT="${{ github.ref == 'refs/heads/main' && 'prod' || 'dev' }}"
          
          python scripts/check_evaluation_thresholds.py \
            --report "$REPORT_PATH" \
            --environment "$ENVIRONMENT" \
            --pr-number "$PR_NUMBER" \
            --commit-sha "$COMMIT_SHA" \
            --output evaluation_results/threshold_check.json
          
          THRESHOLD_CHECK_EXIT=$?
          
          if [ $THRESHOLD_CHECK_EXIT -eq 0 ]; then
            echo "passed=true" >> $GITHUB_OUTPUT
          else
            echo "passed=false" >> $GITHUB_OUTPUT
          fi
          
          exit 0  # workflow 진행 계속

      - name: Upload evaluation artifacts
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: evaluation-results
          path: evaluation_results/
          retention-days: 30

      - name: Comment PR with evaluation results
        if: github.event_name == 'pull_request' && always()
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const path = require('path');
            
            const reportPath = 'evaluation_results/report.json';
            const thresholdPath = 'evaluation_results/threshold_check.json';
            
            let report = {};
            let thresholdCheck = {};
            
            try {
              if (fs.existsSync(reportPath)) {
                report = JSON.parse(fs.readFileSync(reportPath, 'utf8'));
              }
              if (fs.existsSync(thresholdPath)) {
                thresholdCheck = JSON.parse(fs.readFileSync(thresholdPath, 'utf8'));
              }
            } catch (e) {
              console.error('Error reading report files:', e);
            }
            
            const passed = thresholdCheck.overall_passed || false;
            const timestamp = new Date().toISOString();
            
            let comment = `## AI Quality Evaluation Report\n\n`;
            comment += `**Timestamp**: ${timestamp}\n`;
            comment += `**Status**: ${passed ? '✅ PASSED' : '❌ FAILED'}\n\n`;
            
            if (thresholdCheck.metrics) {
              comment += `### Metrics\n\n`;
              comment += `| Metric | Actual | Threshold | Status |\n`;
              comment += `|--------|--------|-----------|--------|\n`;
              
              for (const metric of thresholdCheck.metrics) {
                const status = metric.passed ? '✅ PASS' : '❌ FAIL';
                comment += `| ${metric.name} | ${metric.actual.toFixed(4)} | ${metric.threshold.toFixed(4)} | ${status} |\n`;
              }
              comment += `\n`;
            }
            
            if (!passed) {
              comment += `### Failed Metrics\n\n`;
              if (thresholdCheck.metrics) {
                const failed = thresholdCheck.metrics.filter(m => !m.passed);
                for (const metric of failed) {
                  comment += `- **${metric.name}**: ${metric.actual.toFixed(4)} (threshold: ${metric.threshold.toFixed(4)})\n`;
                }
              }
              comment += `\n### Next Steps\n\n`;
              comment += `1. Review the failed metrics above\n`;
              comment += `2. If this is a hotfix, add the \`evaluation-override\` label to skip this check\n`;
              comment += `3. Otherwise, ensure the code meets quality standards\n`;
              comment += `\n**Note**: Only maintainers can use the \`evaluation-override\` label\n`;
            }
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });

      - name: Stop Docker containers
        if: always()
        run: |
          echo "Stopping Docker containers..."
          docker-compose -f ${{ env.DOCKER_COMPOSE_FILE }} down
          echo "Docker containers stopped"

  handle-evaluation-skip:
    name: Handle Evaluation Skip
    needs: check-evaluation-requirements
    if: needs.check-evaluation-requirements.outputs.should_evaluate == 'false' && github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - name: Comment on PR about skip
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `⏭️ Evaluation skipped: ${{ needs.check-evaluation-requirements.outputs.skip_reason }}`
            });

  require-evaluation-pass:
    name: Require Evaluation to Pass
    needs: golden-set-evaluation
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Check evaluation result
        run: |
          PASSED="${{ needs.golden-set-evaluation.outputs.evaluation_passed }}"
          
          if [ "$PASSED" = "true" ]; then
            echo "✅ Evaluation passed - PR can be merged"
            exit 0
          else
            echo "❌ Evaluation failed - PR cannot be merged"
            exit 1
          fi
```

### 4.4 Unit 테스트 구현

**파일**: `tests/test_evaluation_thresholds.py`

```python
"""
test_evaluation_thresholds.py

임계값 검사 로직에 대한 Unit 테스트
"""

import pytest
import json
import tempfile
from pathlib import Path
from unittest.mock import Mock, patch

from mimir.config.evaluation_config import (
    ThresholdLoader, EvaluationThresholdsConfig, MetricThreshold
)
from scripts.check_evaluation_thresholds import (
    EvaluationThresholdChecker, MetricCheckResult, load_evaluation_results
)


@pytest.fixture
def sample_config_yaml(tmp_path):
    """테스트용 설정 파일 생성"""
    config_content = """
metrics:
  faithfulness:
    threshold: 0.80
    description: "충실도"
    direction: "higher_is_better"
    unit: "proportion"
  answer_relevance:
    threshold: 0.75
    description: "관련성"
    direction: "higher_is_better"
  context_precision:
    threshold: 0.75
    description: "정밀도"
    direction: "higher_is_better"
  context_recall:
    threshold: 0.75
    description: "재현율"
    direction: "higher_is_better"
  citation_present:
    threshold: 0.90
    description: "인용 포함율"
    direction: "higher_is_better"
  hallucination:
    threshold: 0.10
    description: "환각률"
    direction: "lower_is_better"

environments:
  dev:
    override:
      faithfulness: 0.75
      answer_relevance: 0.70
"""
    config_file = tmp_path / "config.yaml"
    config_file.write_text(config_content)
    return str(config_file)


@pytest.fixture
def threshold_loader(sample_config_yaml):
    """ThresholdLoader 인스턴스"""
    return ThresholdLoader(sample_config_yaml)


@pytest.fixture
def passing_results():
    """모든 임계값을 통과하는 결과"""
    return {
        "faithfulness": 0.85,
        "answer_relevance": 0.80,
        "context_precision": 0.78,
        "context_recall": 0.79,
        "citation_present": 0.95,
        "hallucination": 0.05
    }


@pytest.fixture
def failing_results():
    """일부 임계값을 실패하는 결과"""
    return {
        "faithfulness": 0.75,  # 0.80 미만
        "answer_relevance": 0.80,
        "context_precision": 0.72,  # 0.75 미만
        "context_recall": 0.79,
        "citation_present": 0.88,  # 0.90 미만
        "hallucination": 0.12  # 0.10 초과
    }


class TestThresholdLoader:
    """ThresholdLoader 테스트"""
    
    def test_load_config(self, threshold_loader):
        """설정 파일 로드"""
        config = threshold_loader.load()
        assert isinstance(config, EvaluationThresholdsConfig)
        assert "faithfulness" in config.metrics
        assert config.metrics["faithfulness"].threshold == 0.80
    
    def test_get_thresholds_prod(self, threshold_loader):
        """prod 환경 임계값"""
        thresholds = threshold_loader.get_thresholds(environment="prod")
        assert thresholds["faithfulness"] == 0.80
        assert thresholds["hallucination"] == 0.10
    
    def test_get_thresholds_dev_override(self, threshold_loader):
        """dev 환경에서 오버라이드된 임계값"""
        thresholds = threshold_loader.get_thresholds(environment="dev")
        assert thresholds["faithfulness"] == 0.75  # 오버라이드됨
        assert thresholds["answer_relevance"] == 0.70  # 오버라이드됨
        assert thresholds["hallucination"] == 0.10  # 오버라이드되지 않음
    
    def test_get_metric_direction(self, threshold_loader):
        """메트릭 방향성 조회"""
        assert threshold_loader.get_metric_direction("faithfulness") == "higher_is_better"
        assert threshold_loader.get_metric_direction("hallucination") == "lower_is_better"
    
    def test_get_metric_direction_invalid_metric(self, threshold_loader):
        """존재하지 않는 메트릭"""
        with pytest.raises(ValueError, match="Unknown metric"):
            threshold_loader.get_metric_direction("invalid_metric")


class TestEvaluationThresholdChecker:
    """EvaluationThresholdChecker 테스트"""
    
    def test_check_metrics_all_pass(self, threshold_loader, passing_results):
        """모든 메트릭 통과"""
        checker = EvaluationThresholdChecker(threshold_loader, environment="prod")
        report = checker.check_metrics(passing_results)
        
        assert report.overall_passed is True
        assert report.passed_metrics == 6
        assert report.failed_metrics == 0
        assert all(r.passed for r in report.metric_results)
    
    def test_check_metrics_some_fail(self, threshold_loader, failing_results):
        """일부 메트릭 실패"""
        checker = EvaluationThresholdChecker(threshold_loader, environment="prod")
        report = checker.check_metrics(failing_results)
        
        assert report.overall_passed is False
        assert report.failed_metrics > 0
        
        # 실패한 메트릭 확인
        failed_names = {r.metric_name for r in report.metric_results if not r.passed}
        assert "faithfulness" in failed_names
        assert "context_precision" in failed_names
    
    def test_check_metrics_boundary_cases(self, threshold_loader):
        """경계값 테스트"""
        # 정확히 임계값인 경우 (higher_is_better)
        results = {
            "faithfulness": 0.80,  # == threshold
            "answer_relevance": 0.75,
            "context_precision": 0.75,
            "context_recall": 0.75,
            "citation_present": 0.90,
            "hallucination": 0.10  # == threshold (lower_is_better)
        }
        
        checker = EvaluationThresholdChecker(threshold_loader, environment="prod")
        report = checker.check_metrics(results)
        
        # 정확히 임계값인 경우 통과해야 함
        assert report.overall_passed is True
    
    def test_check_metrics_with_environment_override(self, threshold_loader, failing_results):
        """dev 환경에서 오버라이드된 임계값으로 재검사"""
        failing_results["faithfulness"] = 0.77  # dev에서는 0.75가 임계값
        
        checker = EvaluationThresholdChecker(threshold_loader, environment="dev")
        report = checker.check_metrics(failing_results)
        
        # dev 환경에서는 faithfulness이 통과해야 함
        faith_result = next(r for r in report.metric_results if r.metric_name == "faithfulness")
        assert faith_result.passed is True
    
    def test_check_metrics_closed_network_mode(self, threshold_loader):
        """폐쇄망 모드 테스트"""
        results = {
            "faithfulness": 0.85,
            "answer_relevance": 0.80,
            "context_precision": 0.78,
            "context_recall": 0.79,
            "citation_present": 0.95,
            "hallucination": 0.05
        }
        
        checker = EvaluationThresholdChecker(
            threshold_loader,
            environment="prod",
            closed_network_mode=True
        )
        report = checker.check_metrics(results)
        
        assert report.overall_passed is True
    
    def test_detailed_message_generation(self, threshold_loader, passing_results):
        """상세 메시지 생성"""
        checker = EvaluationThresholdChecker(threshold_loader, environment="prod")
        report = checker.check_metrics(passing_results)
        
        assert "AI Quality Evaluation Results" in report.detailed_message
        assert "✅" in report.detailed_message
        assert "faithfulness" in report.detailed_message
        assert report.detailed_message.count("|") > 4  # 테이블 구조


class TestLoadEvaluationResults:
    """평가 결과 파일 로드 테스트"""
    
    def test_load_with_metrics_key(self, tmp_path):
        """평가 결과 (metrics 키 포함)"""
        report = {
            "metrics": {
                "faithfulness": 0.85,
                "answer_relevance": 0.80
            }
        }
        report_file = tmp_path / "report.json"
        report_file.write_text(json.dumps(report))
        
        results = load_evaluation_results(str(report_file))
        assert results["faithfulness"] == 0.85
        assert results["answer_relevance"] == 0.80
    
    def test_load_with_nested_evaluation_results(self, tmp_path):
        """API 응답 형식 (evaluation_results.metrics)"""
        report = {
            "evaluation_results": {
                "metrics": {
                    "faithfulness": 0.85,
                    "answer_relevance": 0.80
                }
            }
        }
        report_file = tmp_path / "report.json"
        report_file.write_text(json.dumps(report))
        
        results = load_evaluation_results(str(report_file))
        assert results["faithfulness"] == 0.85
    
    def test_load_file_not_found(self):
        """파일 없음"""
        with pytest.raises(FileNotFoundError):
            load_evaluation_results("/nonexistent/report.json")
    
    def test_load_invalid_format(self, tmp_path):
        """잘못된 형식"""
        report = {"invalid_key": {"some": "data"}}
        report_file = tmp_path / "report.json"
        report_file.write_text(json.dumps(report))
        
        with pytest.raises(ValueError, match="Invalid evaluation report format"):
            load_evaluation_results(str(report_file))
```

### 4.5 Integration 테스트 구현

**파일**: `tests/test_ci_pipeline_integration.py`

```python
"""
test_ci_pipeline_integration.py

CI 파이프라인 통합 테스트
"""

import pytest
import json
import subprocess
from pathlib import Path
from unittest.mock import patch, MagicMock


class TestCIPipelineIntegration:
    """CI 파이프라인 통합 테스트"""
    
    @pytest.fixture
    def mock_backend_api(self):
        """백엔드 API 모의"""
        return MagicMock()
    
    def test_full_pipeline_passing_evaluation(self, tmp_path, mock_backend_api):
        """전체 파이프라인: 모든 메트릭 통과"""
        # 평가 결과 파일 생성
        report = {
            "metrics": {
                "faithfulness": 0.85,
                "answer_relevance": 0.80,
                "context_precision": 0.78,
                "context_recall": 0.79,
                "citation_present": 0.95,
                "hallucination": 0.05
            },
            "timestamp": "2024-04-17T10:00:00Z"
        }
        
        report_file = tmp_path / "report.json"
        report_file.write_text(json.dumps(report))
        
        # check_evaluation_thresholds.py 실행
        result = subprocess.run(
            [
                "python", "scripts/check_evaluation_thresholds.py",
                "--report", str(report_file)
            ],
            cwd=Path(__file__).parent.parent,
            capture_output=True,
            text=True
        )
        
        assert result.returncode == 0, f"Unexpected failure: {result.stderr}"
        assert "PASSED" in result.stdout
    
    def test_full_pipeline_failing_evaluation(self, tmp_path):
        """전체 파이프라인: 일부 메트릭 실패"""
        report = {
            "metrics": {
                "faithfulness": 0.75,  # 실패: < 0.80
                "answer_relevance": 0.80,
                "context_precision": 0.72,  # 실패: < 0.75
                "context_recall": 0.79,
                "citation_present": 0.88,  # 실패: < 0.90
                "hallucination": 0.12  # 실패: > 0.10
            }
        }
        
        report_file = tmp_path / "report.json"
        report_file.write_text(json.dumps(report))
        
        result = subprocess.run(
            [
                "python", "scripts/check_evaluation_thresholds.py",
                "--report", str(report_file)
            ],
            cwd=Path(__file__).parent.parent,
            capture_output=True,
            text=True
        )
        
        assert result.returncode == 1, "Should fail due to unmet thresholds"
        assert "FAILED" in result.stdout
    
    def test_pipeline_with_json_output(self, tmp_path):
        """JSON 출력으로 결과 저장"""
        report = {
            "metrics": {
                "faithfulness": 0.85,
                "answer_relevance": 0.80,
                "context_precision": 0.78,
                "context_recall": 0.79,
                "citation_present": 0.95,
                "hallucination": 0.05
            }
        }
        
        report_file = tmp_path / "report.json"
        report_file.write_text(json.dumps(report))
        output_file = tmp_path / "check_result.json"
        
        result = subprocess.run(
            [
                "python", "scripts/check_evaluation_thresholds.py",
                "--report", str(report_file),
                "--output", str(output_file)
            ],
            cwd=Path(__file__).parent.parent,
            capture_output=True,
            text=True
        )
        
        assert result.returncode == 0
        assert output_file.exists()
        
        with open(output_file) as f:
            check_result = json.load(f)
        
        assert check_result["overall_passed"] is True
        assert len(check_result["metrics"]) == 6
```

## 5. 산출물

1. **`.github/workflows/golden-set-evaluation.yml`**
   - GitHub Actions 워크플로우 정의
   - PR 및 main 브랜치 푸시 시 자동 평가
   - 임계값 검사 및 PR 코멘트 작성

2. **`config/evaluation_thresholds.yaml`**
   - 6가지 메트릭 임계값 정의
   - 환경별 오버라이드 설정
   - 폐쇄망 모드 fallback 점수

3. **`mimir/config/evaluation_config.py`**
   - `MetricThreshold`, `EvaluationThresholdsConfig` 모델
   - `ThresholdLoader` 클래스: YAML 로드 및 임계값 조회
   - 환경별 오버라이드, 파일 패턴 기반 스킵 로직

4. **`scripts/check_evaluation_thresholds.py`**
   - 평가 결과에 대한 임계값 검사 (CLI 스크립트)
   - `EvaluationThresholdChecker` 클래스
   - JSON 보고서 생성 및 감사 로깅

5. **`tests/test_evaluation_thresholds.py`**
   - Unit 테스트: 임계값 검사, 설정 로드, 경계값
   - 환경별 오버라이드 테스트
   - 폐쇄망 모드 테스트

6. **`tests/test_ci_pipeline_integration.py`**
   - Integration 테스트: 전체 파이프라인 시뮬레이션
   - 통과/실패 케이스
   - JSON 출력 검증

## 6. 완료 기준

1. ✅ GitHub Actions 워크플로우 정의 완료 및 PR 트리거 시 자동 실행 확인
2. ✅ 6가지 메트릭 임계값이 YAML 파일에 정의되고 로드 가능
3. ✅ 임계값 검사 스크립트가 CLI에서 실행 가능하며 올바른 종료 코드 반환
4. ✅ 모든 메트릭 통과 시 exit code 0, 실패 시 exit code 1
5. ✅ PR 라벨 기반 override 메커니즘 작동 (evaluation-override 라벨 시 스킵)
6. ✅ 폐쇄망 모드 활성화 시 LLM 의존성 제거 및 규칙 기반 점수 적용
7. ✅ PR 코멘트에 메트릭 테이블 및 실패 항목 상세 정보 포함
8. ✅ 감사 로그에 평가 결과 기록 (actor_type: "ci", 메트릭 상세 정보)
9. ✅ Unit 테스트: 임계값 검사, 경계값, 환경별 오버라이드 케이스
10. ✅ Integration 테스트: 전체 CI 파이프라인 흐름 (평가 실행 → 검사 → 보고)
11. ✅ 닫힌 네트워크 환경에서도 폐쇄망 모드로 전체 기능 작동 확인
12. ✅ PR 코멘트 형식 검증: 마크다운 테이블, 상태 아이콘(✅/❌)

## 7. 작업 지침

**지침 7-1: GitHub Actions Workflow 개발**
- `.github/workflows/golden-set-evaluation.yml` 작성 시 다음 단계 순서 유지:
  1. checkout
  2. Python 환경 설정
  3. 의존성 설치
  4. Docker 백엔드 시작
  5. Health check (최대 120초 대기)
  6. 골든셋 평가 실행 (최대 30분)
  7. 임계값 검사
  8. 결과 보고 및 PR 코멘트 작성
  9. Docker 정리
- `timeout-minutes: 35` 설정으로 워크플로우 전체 타임아웃 30분 + 여유 5분 확보
- 실패 시에도 Docker 정리 수행 (`always()` 조건 사용)

**지침 7-2: YAML 설정 파일 관리**
- `config/evaluation_thresholds.yaml`은 관리자가 직접 수정 가능하도록 유지
- 모든 6가지 메트릭이 반드시 포함되어야 하며, 누락 시 validation error 발생
- `direction` 필드: "higher_is_better" (faithfulness, answer_relevance, context_precision, context_recall, citation_present) vs "lower_is_better" (hallucination)
- 환경별 오버라이드는 선택사항 (prod 기본값 사용 권장)

**지침 7-3: 임계값 검사 로직**
- `EvaluationThresholdChecker.check_metrics()` 호출 시 평가 결과(메트릭 딕셔너리)를 입력
- `direction == "higher_is_better"`인 경우: `actual >= threshold` → pass
- `direction == "lower_is_better"`인 경우: `actual <= threshold` → pass
- 단 하나의 메트릭도 실패하면 `overall_passed = False`
- 반환값 `ThresholdCheckReport`에는 모든 메트릭 결과 및 상세 메시지 포함

**지침 7-4: PR 코멘트 형식**
- PR 코멘트는 GitHub Actions의 `actions/github-script@v7` 액션으로 자동 작성
- 필수 요소:
  1. 메트릭 테이블 (Metric | Actual | Threshold | Status)
  2. 상태 아이콘: ✅ (pass) / ❌ (fail)
  3. 실패 시 원인 분석 및 next steps
  4. Override 라벨 사용 안내 (maintainer만)
- Timestamp는 ISO 8601 형식 (UTC)

**지침 7-5: 폐쇄망 모드 구현**
- 환경변수 `CLOSED_NETWORK_MODE=true` 또는 GitHub Actions에서 `github.event.inputs.closed_network_mode` 체크
- Fallback 점수 적용: `config/evaluation_thresholds.yaml`의 `closed_network_mode.fallback_base_scores` 사용
- 폐쇄망 모드에서도 평가 엔진 전체 API는 작동해야 함 (LLM 호출만 제외)
- 폐쇄망 모드 여부를 감사 로그에 기록

**지침 7-6: 감사 로깅 및 actor_type**
- 모든 CI 평가 실행은 `actor_type: "ci"`로 기록 (S2 원칙 ⑤)
- 감사 로그 필수 필드:
  - `action`: "evaluate_ci_threshold"
  - `actor_type`: "ci"
  - `resource_type`: "evaluation_report"
  - `resource_id`: "pr_{pr_number}_commit_{commit_sha}"
  - `result`: "success" | "failure"
  - `details`: 메트릭 상세 정보, 환경, 폐쇄망 모드 여부
- S1 원칙 검수 보고서 작성 시 감사 로그 활용

**지침 7-7: 테스트 커버리지**
- Unit 테스트:
  - 임계값 검사 로직: 통과, 실패, 경계값(==threshold) 케이스
  - 환경별 오버라이드: dev/staging/prod별 임계값 검증
  - 폐쇄망 모드: fallback 점수 적용 확인
  - PR 코멘트 생성: 마크다운 포맷 검증
  - Override 라벨 처리: 라벨 존재 시 스킵 확인
- Integration 테스트:
  - 전체 파이프라인 흐름: 평가 실행 → 검사 → 보고 (최소 2개 시나리오)
  - 실패 케이스 시뮬레이션
  - JSON 출력 파일 검증
- pytest 기반 작성, 최소 80% 코드 커버리지 목표

---

**작업 예상 소요 시간**: 35-40시간
**담당 역할**: 백엔드 개발자 (Python, GitHub Actions 경험 필요)
**검수 담당**: CTO, DevOps Lead
**보안 검사**: 감사 로깅 완성도, 폐쇄망 모드 보안성
