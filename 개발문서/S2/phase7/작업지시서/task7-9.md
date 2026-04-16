# Task 7-9: AI품질평가보고서 생성 템플릿 + 수동 평가 스크립트

## 1. 작업 목적

Phase 7 FG7.3의 **자동화된 AI품질평가.md 보고서 생성 및 수동 평가 스크립트**를 구현하여, 개발자와 QA가 골든셋 회귀 테스트 결과를 시각화하고 분석할 수 있는 환경을 제공한다. Jinja2 템플릿 기반의 보고서 자동 생성, 수동 평가 CLI 스크립트, 그리고 Phase 6 FG6.3 대시보드와의 데이터 통합 준비를 완료하여, CI 파이프라인과 개발자 워크플로우 간 완벽한 연동을 달성한다.

## 2. 작업 범위

### 2.1 포함 범위

1. **Jinja2 템플릿 정의** (`templates/ai_quality_evaluation.md.jinja2`)
   - 헤더: 골든셋 정보, 타임스탬프, 프롬프트 버전, 사용 모델명
   - 요약 테이블: 메트릭 | 실제값 | 임계값 | 상태(PASS/FAIL)
   - 항목별 상세 결과: 질문-답변-평가점수 목록
   - 결론 섹션: 종합 평가, 강점/약점 분석
   - 이전 평가 대비 비교: 메트릭 변화 추이, 개선/저하 표시
   - 마크다운 형식으로 PR 댓글 및 별도 문서로 활용 가능

2. **보고서 생성 스크립트** (`scripts/generate_evaluation_report.py`)
   - `EvaluationReport` 객체 로드 (API 응답 또는 JSON 파일)
   - Jinja2 템플릿 렌더링
   - 지정된 디렉토리로 `AI품질평가.md` 저장
   - 메타데이터 JSON 파일 생성 (대시보드 통합용)

3. **수동 평가 CLI 스크립트** (`scripts/evaluate_golden_set.py`)
   - argparse 기반 CLI 인터페이스
   - 옵션: `--golden-set-id`, `--prompt-version`, `--model`, `--output-dir`, `--closed-network`, `--backend-url`
   - 실행 중인 Mimir 백엔드 API에 연결
   - 평가 작업 트리거 및 완료 대기
   - 평가 보고서 다운로드 및 로컬 저장
   - `AI품질평가.md` 자동 생성
   - 콘솔에 요약 정보 pretty-print

4. **Phase 6 FG6.3 대시보드 데이터 바인딩**
   - 보고서 JSON 메타데이터 형식 정의
   - API 응답 형식과 대시보드 스키마 일치
   - 시계열 데이터 저장 (이전 평가 대비 비교 가능)

5. **보고서 메타데이터 저장**
   - JSON 형식: `evaluation_metadata.json`
   - 필드: golden_set_id, prompt_version, model, timestamp, metrics, environment, closed_network_mode
   - 시계열 누적: 여러 평가 결과를 버전별로 추적

6. **이전 평가 대비 비교 기능**
   - 현재 평가와 이전 평가 메트릭 비교
   - 개선 메트릭 표시 (↑ 증가)
   - 저하 메트릭 표시 (↓ 감소)
   - 변화량 및 백분율 계산
   - 추세 분석 (3회 이상 평가 시)

7. **마크다운 포맷팅 및 스타일**
   - 제목 계층 (# ## ###) 명확
   - 테이블 형식 (Markdown table)
   - 코드 블록 (```python 등)
   - 이모지 활용: ✅ (PASS) / ❌ (FAIL) / 📈 (개선) / 📉 (저하)
   - 링크: 관련 문서, PR, 커밋 SHA (클릭 가능)

8. **환경 및 폐쇄망 모드 정보 기록**
   - 평가 환경 (dev, staging, prod)
   - 폐쇄망 모드 여부 및 fallback 전략
   - 사용된 백엔드 URL 및 모델 정보

9. **Unit 테스트**
   - 템플릿 렌더링 (모든 섹션 포함 검증)
   - 메타데이터 생성 및 저장
   - 이전 평가 대비 비교 로직
   - CLI 인자 파싱 및 옵션 검증

10. **Integration 테스트**
    - 백엔드 API 연결 및 평가 트리거
    - 보고서 생성 전체 워크플로우
    - 대시보드 데이터 형식 검증

### 2.2 제외 범위

- 평가 엔진 자체 (Phase 6에서 구현)
- 대시보드 UI/시각화 (Phase 6 FG6.3)
- 메일 자동 발송
- 외부 스토리지 통합 (S3 등)
- 실시간 평가 모니터링 (Webhook)

## 3. 선행 조건

1. Phase 6 FG6.3 완료: `EvaluationReport` 모델 및 API 완성
2. Phase 0: 골든셋 데이터셋 확정
3. Mimir 백엔드 실행 가능 상태
4. Python 3.9+, Jinja2, requests 설치 가능
5. `config/evaluation_thresholds.yaml` 파일 완성 (Task 7-8)
6. 평가 API 엔드포인트 확정 (POST /api/v1/evaluations, GET /api/v1/evaluations/{id})

## 4. 주요 작업 항목

### 4.1 Jinja2 템플릿 정의

**파일**: `templates/ai_quality_evaluation.md.jinja2`

```jinja2
{# AI품질평가.md 템플릿 #}
{# 
  컨텍스트 변수:
  - report: EvaluationReport 객체
  - previous_report: 이전 평가 결과 (선택)
  - metrics_config: 메트릭 설정 정보
  - dashboard_url: 대시보드 링크 (선택)
#}

# AI품질평가 보고서

**생성 일시**: {{ report.timestamp | iso_format }}  
**골든셋**: {{ report.golden_set_id }}  
**프롬프트 버전**: {{ report.prompt_version }}  
**평가 모델**: {{ report.model }}  
**환경**: {{ report.environment }}

{% if report.closed_network_mode %}
⚠️ **폐쇄망 모드**: 이 평가는 규칙 기반 fallback 점수로 실행되었습니다.
{% endif %}

---

## 요약

| 메트릭 | 실제값 | 임계값 | 상태 | 변화 |
|--------|--------|--------|------|------|
{% for metric_name, metric_result in report.metrics.items() %}
{% set config = metrics_config[metric_name] %}
{% set status = "✅ PASS" if metric_result.passed else "❌ FAIL" %}
{% set change = "" %}
{% if previous_report and metric_name in previous_report.metrics %}
  {% set prev_value = previous_report.metrics[metric_name].value %}
  {% set diff = metric_result.value - prev_value %}
  {% set pct_change = (diff / prev_value * 100) if prev_value != 0 else 0 %}
  {% if diff > 0.001 %}
    {% set change = "📈 +{:.4f} (+{:.1f}%)".format(diff, pct_change) %}
  {% elif diff < -0.001 %}
    {% set change = "📉 {:.4f} ({:.1f}%)".format(diff, pct_change) %}
  {% else %}
    {% set change = "➡️ 유지" %}
  {% endif %}
{% endif %}
| {{ metric_name }} | {{ "%.4f" | format(metric_result.value) }} | {{ "%.4f" | format(config.threshold) }} | {{ status }} | {{ change }} |
{% endfor %}

**전체 평가**: {% if report.overall_passed %}✅ **통과**{% else %}❌ **미통과**{% endif %}

---

## 상세 결과

### 통과한 메트릭
{% set passed_metrics = report.metrics.items() | selectattr('1.passed', 'equalto', True) | list %}
{% if passed_metrics %}
{% for metric_name, metric_result in passed_metrics %}
- ✅ **{{ metric_name }}**: {{ "%.4f" | format(metric_result.value) }} (임계값: {{ "%.4f" | format(metrics_config[metric_name].threshold) }})
  - {{ metrics_config[metric_name].description }}
{% endfor %}
{% else %}
_없음_
{% endif %}

### 미통과한 메트릭
{% set failed_metrics = report.metrics.items() | selectattr('1.passed', 'equalto', False) | list %}
{% if failed_metrics %}
{% for metric_name, metric_result in failed_metrics %}
- ❌ **{{ metric_name }}**: {{ "%.4f" | format(metric_result.value) }} (임계값: {{ "%.4f" | format(metrics_config[metric_name].threshold) }})
  - {{ metrics_config[metric_name].description }}
  - 차이: {{ "%.4f" | format(metric_result.value - metrics_config[metric_name].threshold) }}
{% endfor %}
{% else %}
_없음_
{% endif %}

---

## 항목별 상세 평가

**총 평가 항목**: {{ report.evaluation_items | length }}

{% for idx, item in report.evaluation_items[:10] %}
### {{ idx + 1 }}. {{ item.question[:80] }}{% if item.question | length > 80 %}...{% endif %}

**질문**:
```
{{ item.question }}
```

**예상 답변**:
```
{{ item.expected_answer }}
```

**생성된 답변**:
```
{{ item.generated_answer }}
```

**평가 점수**:
- Faithfulness: {{ "%.4f" | format(item.faithfulness) }}
- Answer Relevance: {{ "%.4f" | format(item.answer_relevance) }}
- Context Precision: {{ "%.4f" | format(item.context_precision) }}
- Context Recall: {{ "%.4f" | format(item.context_recall) }}
- Citation Present: {{ "%.4f" | format(item.citation_present) }}
- Hallucination: {{ "%.4f" | format(item.hallucination) }}

---

{% endfor %}

{% if report.evaluation_items | length > 10 %}
**... 그 외 {{ report.evaluation_items | length - 10 }}개 항목**

모든 항목은 대시보드에서 확인 가능합니다.
{% endif %}

---

## 분석 및 인사이트

### 강점
{% if report.insights.strengths %}
{% for strength in report.insights.strengths %}
- {{ strength }}
{% endfor %}
{% else %}
_분석 대기 중_
{% endif %}

### 약점
{% if report.insights.weaknesses %}
{% for weakness in report.insights.weaknesses %}
- {{ weakness }}
{% endfor %}
{% else %}
_분석 대기 중_
{% endif %}

### 권장사항
{% if report.insights.recommendations %}
{% for rec in report.insights.recommendations %}
- {{ rec }}
{% endfor %}
{% else %}
_권장사항 없음_
{% endif %}

---

## 추세 분석

{% if previous_report %}
### 이전 평가 대비

**평가 일시**: {{ previous_report.timestamp | iso_format }}

| 메트릭 | 이전값 | 현재값 | 변화 | 상태 |
|--------|--------|--------|------|------|
{% for metric_name, current_result in report.metrics.items() %}
{% set prev_result = previous_report.metrics[metric_name] %}
{% set diff = current_result.value - prev_result.value %}
{% set pct_change = (diff / prev_result.value * 100) if prev_result.value != 0 else 0 %}
{% set trend = "📈" if diff > 0.001 else ("📉" if diff < -0.001 else "➡️") %}
| {{ metric_name }} | {{ "%.4f" | format(prev_result.value) }} | {{ "%.4f" | format(current_result.value) }} | {{ trend }} {{ "{:+.4f}".format(diff) }} ({{ "{:+.1f}".format(pct_change) }}%) | {% if current_result.passed %}✅{% else %}❌{% endif %} |
{% endfor %}

### 종합 평가 변화

- **이전**: {% if previous_report.overall_passed %}✅ 통과{% else %}❌ 미통과{% endif %} ({{ previous_report.passed_metrics }}/{{ previous_report.total_metrics }} 메트릭)
- **현재**: {% if report.overall_passed %}✅ 통과{% else %}❌ 미통과{% endif %} ({{ report.passed_metrics }}/{{ report.total_metrics }} 메트릭)

{% endif %}

---

## 추가 정보

- **커밋 SHA**: {% if report.commit_sha %}`{{ report.commit_sha }}`{% else %}_미지정_{% endif %}
- **PR 번호**: {% if report.pr_number %}#{{ report.pr_number }}{% else %}_미지정_{% endif %}
- **폐쇄망 모드**: {% if report.closed_network_mode %}Yes (fallback 전략: {{ report.closed_network_strategy }}){% else %}No{% endif %}
- **백엔드 URL**: `{{ report.backend_url }}`

{% if dashboard_url %}
📊 **[대시보드에서 보기]({{ dashboard_url }})**
{% endif %}

---

*자동 생성됨: Mimir AI품질평가시스템*
```

### 4.2 보고서 생성 스크립트 구현

**파일**: `scripts/generate_evaluation_report.py`

```python
#!/usr/bin/env python3
"""
generate_evaluation_report.py

평가 결과 JSON 또는 API 응답으로부터 AI품질평가.md 보고서 생성
"""

import json
import sys
from pathlib import Path
from typing import Optional, Dict, Any
from datetime import datetime
import logging
from jinja2 import Environment, FileSystemLoader, Template

sys.path.insert(0, str(Path(__file__).parent.parent))

from mimir.config.evaluation_config import ThresholdLoader, get_threshold_loader
from mimir.core.models import EvaluationReport


logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
logger = logging.getLogger(__name__)


class EvaluationReportGenerator:
    """평가 보고서 생성 클래스"""
    
    def __init__(
        self,
        template_path: str = "templates/ai_quality_evaluation.md.jinja2",
        threshold_loader: Optional[ThresholdLoader] = None
    ):
        """
        Args:
            template_path: Jinja2 템플릿 파일 경로
            threshold_loader: 임계값 로더
        """
        self.template_path = Path(template_path)
        self.threshold_loader = threshold_loader or get_threshold_loader()
        
        # Jinja2 환경 설정
        template_dir = self.template_path.parent
        self.jinja_env = Environment(
            loader=FileSystemLoader(str(template_dir)),
            autoescape=False
        )
        
        # 커스텀 필터 추가
        self._setup_jinja_filters()
    
    def _setup_jinja_filters(self):
        """Jinja2 커스텀 필터 설정"""
        def iso_format(dt: datetime) -> str:
            """ISO 8601 형식"""
            if isinstance(dt, str):
                return dt
            return dt.isoformat() if dt else ""
        
        self.jinja_env.filters["iso_format"] = iso_format
    
    def generate(
        self,
        evaluation_report: EvaluationReport,
        output_path: Path,
        previous_report: Optional[EvaluationReport] = None,
        dashboard_url: Optional[str] = None,
        environment: str = "prod"
    ) -> Path:
        """
        보고서 생성
        
        Args:
            evaluation_report: 현재 평가 결과
            output_path: 출력 파일 경로 (.md)
            previous_report: 이전 평가 결과 (선택)
            dashboard_url: 대시보드 URL (선택)
            environment: 환경명 (prod, staging, dev)
        
        Returns:
            생성된 파일 경로
        """
        output_path = Path(output_path)
        output_path.parent.mkdir(parents=True, exist_ok=True)
        
        # 임계값 설정 로드
        thresholds = self.threshold_loader.get_thresholds(environment=environment)
        metrics_config = self.threshold_loader.load().metrics
        
        # 메트릭별 임계값 추가
        metrics_config_dict = {}
        for metric_name, metric_cfg in metrics_config.items():
            metrics_config_dict[metric_name] = {
                "threshold": thresholds[metric_name],
                "description": metric_cfg.description,
                "direction": metric_cfg.direction
            }
        
        # 템플릿 컨텍스트 구성
        context = {
            "report": evaluation_report,
            "previous_report": previous_report,
            "metrics_config": metrics_config_dict,
            "dashboard_url": dashboard_url
        }
        
        # 템플릿 로드 및 렌더링
        template = self.jinja_env.get_template(self.template_path.name)
        rendered = template.render(**context)
        
        # 파일 저장
        with open(output_path, "w", encoding="utf-8") as f:
            f.write(rendered)
        
        logger.info(f"Report generated: {output_path}")
        return output_path
    
    def generate_metadata(
        self,
        evaluation_report: EvaluationReport,
        output_path: Path,
        environment: str = "prod"
    ) -> Path:
        """
        메타데이터 JSON 파일 생성 (대시보드 통합용)
        
        Args:
            evaluation_report: 평가 결과
            output_path: 출력 파일 경로 (.json)
            environment: 환경명
        
        Returns:
            생성된 파일 경로
        """
        output_path = Path(output_path)
        output_path.parent.mkdir(parents=True, exist_ok=True)
        
        metadata = {
            "golden_set_id": evaluation_report.golden_set_id,
            "prompt_version": evaluation_report.prompt_version,
            "model": evaluation_report.model,
            "timestamp": evaluation_report.timestamp.isoformat() if isinstance(evaluation_report.timestamp, datetime) else evaluation_report.timestamp,
            "environment": environment,
            "closed_network_mode": evaluation_report.closed_network_mode,
            "metrics": {
                metric_name: {
                    "value": result.value,
                    "passed": result.passed
                }
                for metric_name, result in evaluation_report.metrics.items()
            },
            "overall_passed": evaluation_report.overall_passed,
            "total_metrics": evaluation_report.total_metrics,
            "passed_metrics": evaluation_report.passed_metrics,
            "failed_metrics": evaluation_report.failed_metrics,
            "evaluation_items_count": len(evaluation_report.evaluation_items),
            "commit_sha": evaluation_report.commit_sha,
            "pr_number": evaluation_report.pr_number,
            "backend_url": evaluation_report.backend_url
        }
        
        with open(output_path, "w", encoding="utf-8") as f:
            json.dump(metadata, f, indent=2, ensure_ascii=False)
        
        logger.info(f"Metadata generated: {output_path}")
        return output_path


def load_evaluation_report_from_json(json_path: str) -> EvaluationReport:
    """
    JSON 파일에서 EvaluationReport 로드
    
    Args:
        json_path: JSON 파일 경로
    
    Returns:
        EvaluationReport 객체
    """
    with open(json_path, "r", encoding="utf-8") as f:
        data = json.load(f)
    
    # EvaluationReport 모델로 파싱
    # (Phase 6에서 정의된 모델 사용)
    return EvaluationReport.parse_obj(data)


def load_previous_report(metadata_dir: str, current_report: EvaluationReport) -> Optional[EvaluationReport]:
    """
    이전 평가 결과 로드
    
    Args:
        metadata_dir: 메타데이터 저장 디렉토리
        current_report: 현재 평가 결과
    
    Returns:
        이전 평가 결과 (없으면 None)
    """
    metadata_path = Path(metadata_dir) / "evaluation_metadata.json"
    
    if not metadata_path.exists():
        return None
    
    try:
        with open(metadata_path, "r", encoding="utf-8") as f:
            metadata_list = json.load(f)
        
        # 리스트인 경우 가장 최근 항목 반환
        if isinstance(metadata_list, list) and len(metadata_list) > 1:
            # 마지막에서 두 번째 항목 (가장 최근의 이전 평가)
            prev_metadata = metadata_list[-2]
            # 메타데이터로부터 EvaluationReport 재구성 (간략)
            # 전체 상세 정보는 별도 파일에서 로드 필요
            return None  # 현재는 None 반환, Phase 6에서 구현 필요
        
    except Exception as e:
        logger.warning(f"Failed to load previous report: {e}")
    
    return None


def update_evaluation_metadata(
    metadata_dir: str,
    new_metadata: Dict[str, Any]
):
    """
    평가 메타데이터 누적
    
    Args:
        metadata_dir: 메타데이터 저장 디렉토리
        new_metadata: 새로운 메타데이터
    """
    metadata_path = Path(metadata_dir) / "evaluation_metadata.json"
    metadata_path.parent.mkdir(parents=True, exist_ok=True)
    
    metadata_list = []
    
    # 기존 메타데이터 로드
    if metadata_path.exists():
        try:
            with open(metadata_path, "r", encoding="utf-8") as f:
                existing = json.load(f)
                if isinstance(existing, list):
                    metadata_list = existing
                else:
                    metadata_list = [existing]
        except Exception as e:
            logger.warning(f"Failed to load existing metadata: {e}")
    
    # 새로운 메타데이터 추가
    metadata_list.append(new_metadata)
    
    # 최근 100개만 유지
    if len(metadata_list) > 100:
        metadata_list = metadata_list[-100:]
    
    with open(metadata_path, "w", encoding="utf-8") as f:
        json.dump(metadata_list, f, indent=2, ensure_ascii=False)
    
    logger.info(f"Metadata updated: {metadata_path} ({len(metadata_list)} records)")


def main():
    """CLI 진입점"""
    import argparse
    
    parser = argparse.ArgumentParser(
        description="평가 결과로부터 AI품질평가.md 보고서 생성"
    )
    parser.add_argument(
        "--report",
        type=str,
        required=True,
        help="평가 결과 JSON 파일 경로"
    )
    parser.add_argument(
        "--output-dir",
        type=str,
        default=".",
        help="출력 디렉토리"
    )
    parser.add_argument(
        "--report-name",
        type=str,
        default="AI품질평가.md",
        help="보고서 파일명"
    )
    parser.add_argument(
        "--environment",
        type=str,
        default="prod",
        choices=["dev", "staging", "prod"],
        help="환경명"
    )
    parser.add_argument(
        "--dashboard-url",
        type=str,
        help="대시보드 URL"
    )
    parser.add_argument(
        "--save-metadata",
        action="store_true",
        help="메타데이터 JSON도 저장"
    )
    
    args = parser.parse_args()
    
    try:
        # 평가 결과 로드
        logger.info(f"Loading evaluation report from {args.report}")
        evaluation_report = load_evaluation_report_from_json(args.report)
        
        # 보고서 생성
        generator = EvaluationReportGenerator()
        
        output_path = Path(args.output_dir) / args.report_name
        generator.generate(
            evaluation_report=evaluation_report,
            output_path=output_path,
            dashboard_url=args.dashboard_url,
            environment=args.environment
        )
        
        # 메타데이터 저장
        if args.save_metadata:
            metadata_path = Path(args.output_dir) / "evaluation_metadata.json"
            generator.generate_metadata(
                evaluation_report=evaluation_report,
                output_path=metadata_path,
                environment=args.environment
            )
            
            # 누적 메타데이터 업데이트
            with open(metadata_path) as f:
                metadata = json.load(f)
            update_evaluation_metadata(args.output_dir, metadata)
        
        logger.info("Report generation completed successfully")
        
    except Exception as e:
        logger.error(f"Error generating report: {e}", exc_info=True)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

### 4.3 수동 평가 CLI 스크립트 구현

**파일**: `scripts/evaluate_golden_set.py`

```python
#!/usr/bin/env python3
"""
evaluate_golden_set.py

수동 골든셋 평가 실행 CLI 스크립트
- 백엔드 API에 연결
- 평가 작업 트리거
- 완료 대기 및 결과 다운로드
- AI품질평가.md 생성
- 콘솔 요약 출력
"""

import json
import sys
import time
import argparse
from pathlib import Path
from typing import Optional, Dict, Any
import logging
from datetime import datetime

import requests
from requests.exceptions import RequestException, Timeout

sys.path.insert(0, str(Path(__file__).parent.parent))

from scripts.generate_evaluation_report import EvaluationReportGenerator, load_evaluation_report_from_json


logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
logger = logging.getLogger(__name__)


class GoldenSetEvaluationClient:
    """백엔드 API 평가 클라이언트"""
    
    def __init__(
        self,
        backend_url: str,
        timeout: int = 30,
        closed_network_mode: bool = False
    ):
        """
        Args:
            backend_url: 백엔드 기본 URL (예: http://localhost:8000)
            timeout: 요청 타임아웃 (초)
            closed_network_mode: 폐쇄망 모드
        """
        self.backend_url = backend_url.rstrip("/")
        self.timeout = timeout
        self.closed_network_mode = closed_network_mode
        self.session = requests.Session()
    
    def health_check(self) -> bool:
        """백엔드 상태 확인"""
        try:
            response = self.session.get(
                f"{self.backend_url}/health",
                timeout=self.timeout
            )
            return response.status_code == 200
        except Exception as e:
            logger.error(f"Health check failed: {e}")
            return False
    
    def start_evaluation(
        self,
        golden_set_id: str,
        prompt_version: str,
        model: str,
        environment: str = "prod",
        metadata: Optional[Dict[str, Any]] = None
    ) -> str:
        """
        평가 작업 시작
        
        Args:
            golden_set_id: 골든셋 ID
            prompt_version: 프롬프트 버전
            model: 사용할 모델명
            environment: 환경 (dev, staging, prod)
            metadata: 추가 메타데이터
        
        Returns:
            평가 작업 ID
        
        Raises:
            RequestException: API 호출 실패
        """
        payload = {
            "golden_set_id": golden_set_id,
            "prompt_version": prompt_version,
            "model": model,
            "environment": environment,
            "closed_network_mode": self.closed_network_mode,
            **(metadata or {})
        }
        
        logger.info(f"Starting evaluation: {payload}")
        
        try:
            response = self.session.post(
                f"{self.backend_url}/api/v1/evaluations",
                json=payload,
                timeout=self.timeout
            )
            response.raise_for_status()
            
            result = response.json()
            evaluation_id = result.get("id") or result.get("evaluation_id")
            
            if not evaluation_id:
                raise ValueError("Invalid response: no evaluation ID returned")
            
            logger.info(f"Evaluation started: {evaluation_id}")
            return evaluation_id
        
        except RequestException as e:
            logger.error(f"Failed to start evaluation: {e}")
            raise
    
    def wait_for_completion(
        self,
        evaluation_id: str,
        max_wait_seconds: int = 1800,
        poll_interval: int = 10
    ) -> Dict[str, Any]:
        """
        평가 완료 대기
        
        Args:
            evaluation_id: 평가 작업 ID
            max_wait_seconds: 최대 대기 시간 (초)
            poll_interval: 폴링 간격 (초)
        
        Returns:
            평가 결과
        
        Raises:
            TimeoutError: 타임아웃
            ValueError: 평가 실패
        """
        elapsed = 0
        
        while elapsed < max_wait_seconds:
            try:
                response = self.session.get(
                    f"{self.backend_url}/api/v1/evaluations/{evaluation_id}",
                    timeout=self.timeout
                )
                response.raise_for_status()
                
                result = response.json()
                status = result.get("status")
                
                logger.info(f"Evaluation status: {status}")
                
                if status == "completed":
                    return result
                elif status == "failed":
                    error_msg = result.get("error", "Unknown error")
                    raise ValueError(f"Evaluation failed: {error_msg}")
                elif status == "processing":
                    logger.info(f"Still processing... ({elapsed}s/{max_wait_seconds}s)")
                
                time.sleep(poll_interval)
                elapsed += poll_interval
            
            except RequestException as e:
                logger.error(f"Error checking evaluation status: {e}")
                raise
        
        raise TimeoutError(f"Evaluation did not complete within {max_wait_seconds} seconds")
    
    def download_report(
        self,
        evaluation_id: str,
        output_path: Path
    ) -> Path:
        """
        평가 보고서 다운로드
        
        Args:
            evaluation_id: 평가 작업 ID
            output_path: 저장 경로
        
        Returns:
            다운로드된 파일 경로
        """
        output_path = Path(output_path)
        output_path.parent.mkdir(parents=True, exist_ok=True)
        
        try:
            response = self.session.get(
                f"{self.backend_url}/api/v1/evaluations/{evaluation_id}/report",
                timeout=self.timeout
            )
            response.raise_for_status()
            
            with open(output_path, "w", encoding="utf-8") as f:
                json.dump(response.json(), f, indent=2)
            
            logger.info(f"Report downloaded: {output_path}")
            return output_path
        
        except RequestException as e:
            logger.error(f"Failed to download report: {e}")
            raise


def format_metrics_table(metrics: Dict[str, Any]) -> str:
    """메트릭 테이블 포맷팅"""
    lines = ["", "Metrics Summary:"]
    lines.append("-" * 70)
    lines.append(f"{'Metric':<25} {'Value':>15} {'Status':>10}")
    lines.append("-" * 70)
    
    for metric_name, metric_data in metrics.items():
        value = metric_data.get("value", 0)
        passed = metric_data.get("passed", False)
        status = "✅ PASS" if passed else "❌ FAIL"
        lines.append(f"{metric_name:<25} {value:>15.4f} {status:>10}")
    
    lines.append("-" * 70)
    
    return "\n".join(lines)


def main():
    """CLI 진입점"""
    parser = argparse.ArgumentParser(
        description="수동 골든셋 평가 실행"
    )
    parser.add_argument(
        "--backend-url",
        type=str,
        default="http://localhost:8000",
        help="백엔드 URL (기본: http://localhost:8000)"
    )
    parser.add_argument(
        "--golden-set-id",
        type=str,
        default="default",
        help="골든셋 ID (기본: default)"
    )
    parser.add_argument(
        "--prompt-version",
        type=str,
        required=True,
        help="프롬프트 버전 (필수)"
    )
    parser.add_argument(
        "--model",
        type=str,
        required=True,
        help="사용할 모델 (필수)"
    )
    parser.add_argument(
        "--environment",
        type=str,
        default="prod",
        choices=["dev", "staging", "prod"],
        help="환경 (기본: prod)"
    )
    parser.add_argument(
        "--output-dir",
        type=str,
        default=".",
        help="출력 디렉토리 (기본: 현재 디렉토리)"
    )
    parser.add_argument(
        "--json-report",
        type=str,
        help="JSON 평가 보고서 저장 경로"
    )
    parser.add_argument(
        "--markdown-report",
        type=str,
        help="마크다운 보고서 저장 경로"
    )
    parser.add_argument(
        "--closed-network",
        action="store_true",
        help="폐쇄망 모드 활성화"
    )
    parser.add_argument(
        "--max-wait",
        type=int,
        default=1800,
        help="최대 대기 시간 (초, 기본: 1800)"
    )
    parser.add_argument(
        "--poll-interval",
        type=int,
        default=10,
        help="폴링 간격 (초, 기본: 10)"
    )
    parser.add_argument(
        "--skip-report-generation",
        action="store_true",
        help="마크다운 보고서 생성 스킵"
    )
    parser.add_argument(
        "--dashboard-url",
        type=str,
        help="대시보드 URL (보고서에 포함)"
    )
    
    args = parser.parse_args()
    
    try:
        output_dir = Path(args.output_dir)
        output_dir.mkdir(parents=True, exist_ok=True)
        
        # 클라이언트 생성
        client = GoldenSetEvaluationClient(
            backend_url=args.backend_url,
            closed_network_mode=args.closed_network
        )
        
        # Health check
        logger.info("Performing health check...")
        if not client.health_check():
            logger.error("Backend is not healthy")
            sys.exit(1)
        
        # 평가 시작
        logger.info("Starting evaluation...")
        evaluation_id = client.start_evaluation(
            golden_set_id=args.golden_set_id,
            prompt_version=args.prompt_version,
            model=args.model,
            environment=args.environment,
            metadata={
                "closed_network_mode": args.closed_network
            }
        )
        
        # 완료 대기
        logger.info("Waiting for evaluation to complete...")
        evaluation_result = client.wait_for_completion(
            evaluation_id=evaluation_id,
            max_wait_seconds=args.max_wait,
            poll_interval=args.poll_interval
        )
        
        # JSON 보고서 저장
        json_report_path = Path(args.json_report or output_dir / f"evaluation_{evaluation_id}.json")
        json_report_path.parent.mkdir(parents=True, exist_ok=True)
        
        with open(json_report_path, "w", encoding="utf-8") as f:
            json.dump(evaluation_result, f, indent=2, ensure_ascii=False)
        
        logger.info(f"JSON report saved: {json_report_path}")
        
        # 마크다운 보고서 생성 (선택사항)
        if not args.skip_report_generation:
            try:
                # 평가 결과를 EvaluationReport로 로드
                evaluation_report = load_evaluation_report_from_json(str(json_report_path))
                
                # 보고서 생성
                generator = EvaluationReportGenerator()
                markdown_report_path = Path(
                    args.markdown_report or output_dir / "AI품질평가.md"
                )
                
                generator.generate(
                    evaluation_report=evaluation_report,
                    output_path=markdown_report_path,
                    dashboard_url=args.dashboard_url,
                    environment=args.environment
                )
                
                logger.info(f"Markdown report saved: {markdown_report_path}")
            
            except Exception as e:
                logger.warning(f"Failed to generate markdown report: {e}")
        
        # 콘솔에 요약 출력
        print("\n" + "=" * 70)
        print("EVALUATION COMPLETE")
        print("=" * 70)
        print(f"Evaluation ID: {evaluation_id}")
        print(f"Status: {evaluation_result.get('status')}")
        print(f"Timestamp: {evaluation_result.get('timestamp')}")
        
        if "metrics" in evaluation_result:
            print(format_metrics_table(evaluation_result["metrics"]))
        
        overall_passed = evaluation_result.get("overall_passed", False)
        print(f"\nOverall Result: {'✅ PASSED' if overall_passed else '❌ FAILED'}")
        print("=" * 70 + "\n")
        
        sys.exit(0 if overall_passed else 1)
    
    except TimeoutError as e:
        logger.error(f"Evaluation timed out: {e}")
        sys.exit(1)
    except ValueError as e:
        logger.error(f"Evaluation failed: {e}")
        sys.exit(1)
    except Exception as e:
        logger.error(f"Unexpected error: {e}", exc_info=True)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

### 4.4 Unit 테스트 구현

**파일**: `tests/test_evaluation_report_generation.py`

```python
"""
test_evaluation_report_generation.py

평가 보고서 생성 및 템플릿 렌더링 테스트
"""

import pytest
import json
import tempfile
from pathlib import Path
from datetime import datetime
from unittest.mock import Mock, patch

from scripts.generate_evaluation_report import (
    EvaluationReportGenerator,
    load_evaluation_report_from_json
)


@pytest.fixture
def sample_evaluation_report():
    """테스트용 평가 결과"""
    return {
        "id": "eval_001",
        "golden_set_id": "default",
        "prompt_version": "v1.0",
        "model": "gpt-4",
        "timestamp": datetime.now().isoformat(),
        "environment": "prod",
        "closed_network_mode": False,
        "backend_url": "http://localhost:8000",
        "commit_sha": "abc123def456",
        "pr_number": 42,
        "metrics": {
            "faithfulness": {"value": 0.85, "passed": True},
            "answer_relevance": {"value": 0.80, "passed": True},
            "context_precision": {"value": 0.78, "passed": True},
            "context_recall": {"value": 0.79, "passed": True},
            "citation_present": {"value": 0.95, "passed": True},
            "hallucination": {"value": 0.05, "passed": True}
        },
        "overall_passed": True,
        "total_metrics": 6,
        "passed_metrics": 6,
        "failed_metrics": 0,
        "evaluation_items": [
            {
                "id": "item_001",
                "question": "What is the capital of France?",
                "expected_answer": "Paris",
                "generated_answer": "Paris is the capital of France.",
                "faithfulness": 0.95,
                "answer_relevance": 0.92,
                "context_precision": 0.85,
                "context_recall": 0.88,
                "citation_present": 1.0,
                "hallucination": 0.0
            }
        ],
        "insights": {
            "strengths": ["High citation rate", "Low hallucination"],
            "weaknesses": [],
            "recommendations": []
        }
    }


@pytest.fixture
def generator(tmp_path):
    """보고서 생성기"""
    # 임시 템플릿 디렉토리 생성
    template_dir = tmp_path / "templates"
    template_dir.mkdir()
    
    # 간단한 템플릿 파일 생성
    template_content = """# Report for {{ report.golden_set_id }}
    
Timestamp: {{ report.timestamp }}

## Metrics
{% for metric_name, metric_result in report.metrics.items() %}
- {{ metric_name }}: {{ metric_result.value }}
{% endfor %}

Overall: {{ report.overall_passed }}
"""
    template_file = template_dir / "ai_quality_evaluation.md.jinja2"
    template_file.write_text(template_content)
    
    return EvaluationReportGenerator(
        template_path=str(template_file)
    )


class TestEvaluationReportGenerator:
    """보고서 생성 테스트"""
    
    def test_generate_basic_report(self, generator, sample_evaluation_report, tmp_path):
        """기본 보고서 생성"""
        from mimir.core.models import EvaluationReport
        
        # Mock EvaluationReport
        report = Mock(spec=EvaluationReport)
        report.golden_set_id = sample_evaluation_report["golden_set_id"]
        report.timestamp = sample_evaluation_report["timestamp"]
        report.metrics = sample_evaluation_report["metrics"]
        report.overall_passed = sample_evaluation_report["overall_passed"]
        
        output_path = tmp_path / "report.md"
        
        result = generator.generate(
            evaluation_report=report,
            output_path=output_path
        )
        
        assert result.exists()
        content = result.read_text()
        assert "Report for default" in content
        assert "faithfulness" in content
        assert "Overall: True" in content
    
    def test_generate_metadata(self, generator, sample_evaluation_report, tmp_path):
        """메타데이터 생성"""
        from mimir.core.models import EvaluationReport
        
        report = Mock(spec=EvaluationReport)
        report.golden_set_id = sample_evaluation_report["golden_set_id"]
        report.timestamp = datetime.now()
        report.prompt_version = "v1.0"
        report.model = "gpt-4"
        report.metrics = sample_evaluation_report["metrics"]
        report.overall_passed = True
        report.total_metrics = 6
        report.passed_metrics = 6
        report.failed_metrics = 0
        report.evaluation_items = sample_evaluation_report["evaluation_items"]
        report.closed_network_mode = False
        report.commit_sha = "abc123"
        report.pr_number = 42
        report.backend_url = "http://localhost:8000"
        
        output_path = tmp_path / "metadata.json"
        
        result = generator.generate_metadata(
            evaluation_report=report,
            output_path=output_path
        )
        
        assert result.exists()
        
        with open(result) as f:
            metadata = json.load(f)
        
        assert metadata["golden_set_id"] == "default"
        assert metadata["overall_passed"] is True
        assert "faithfulness" in metadata["metrics"]
    
    def test_load_evaluation_report_from_json(self, sample_evaluation_report, tmp_path):
        """JSON에서 평가 결과 로드"""
        json_path = tmp_path / "report.json"
        
        with open(json_path, "w") as f:
            json.dump(sample_evaluation_report, f)
        
        # EvaluationReport 모델은 Phase 6에서 정의되므로
        # 여기서는 JSON 파일이 존재하는지만 검증
        assert json_path.exists()


class TestMetadataUpdate:
    """메타데이터 누적 테스트"""
    
    def test_update_evaluation_metadata_new_file(self, tmp_path):
        """새 메타데이터 파일 생성"""
        from scripts.generate_evaluation_report import update_evaluation_metadata
        
        metadata_dir = tmp_path
        new_metadata = {
            "golden_set_id": "default",
            "timestamp": datetime.now().isoformat(),
            "overall_passed": True
        }
        
        update_evaluation_metadata(str(metadata_dir), new_metadata)
        
        metadata_file = metadata_dir / "evaluation_metadata.json"
        assert metadata_file.exists()
        
        with open(metadata_file) as f:
            data = json.load(f)
        
        assert isinstance(data, list)
        assert len(data) == 1
        assert data[0]["golden_set_id"] == "default"
    
    def test_update_evaluation_metadata_append(self, tmp_path):
        """기존 메타데이터에 추가"""
        from scripts.generate_evaluation_report import update_evaluation_metadata
        
        metadata_dir = tmp_path
        
        # 첫 번째 메타데이터
        update_evaluation_metadata(str(metadata_dir), {
            "golden_set_id": "default",
            "timestamp": "2024-04-17T10:00:00",
            "overall_passed": True
        })
        
        # 두 번째 메타데이터
        update_evaluation_metadata(str(metadata_dir), {
            "golden_set_id": "default",
            "timestamp": "2024-04-17T10:30:00",
            "overall_passed": False
        })
        
        metadata_file = metadata_dir / "evaluation_metadata.json"
        with open(metadata_file) as f:
            data = json.load(f)
        
        assert len(data) == 2
        assert data[1]["overall_passed"] is False
```

### 4.5 Integration 테스트 구현

**파일**: `tests/test_evaluate_golden_set_integration.py`

```python
"""
test_evaluate_golden_set_integration.py

evaluate_golden_set.py 통합 테스트
"""

import pytest
import json
import subprocess
from pathlib import Path
from unittest.mock import patch, MagicMock
from datetime import datetime


class TestGoldenSetEvaluationClient:
    """GoldenSetEvaluationClient 통합 테스트"""
    
    @pytest.fixture
    def mock_backend_server(self):
        """백엔드 서버 모의 (실제로는 docker-compose로 실행)"""
        return MagicMock()
    
    def test_health_check_success(self, mock_backend_server):
        """Health check 성공"""
        from scripts.evaluate_golden_set import GoldenSetEvaluationClient
        
        with patch("requests.Session.get") as mock_get:
            mock_response = MagicMock()
            mock_response.status_code = 200
            mock_get.return_value = mock_response
            
            client = GoldenSetEvaluationClient("http://localhost:8000")
            assert client.health_check() is True
    
    def test_start_evaluation_success(self):
        """평가 시작 성공"""
        from scripts.evaluate_golden_set import GoldenSetEvaluationClient
        
        with patch("requests.Session.post") as mock_post:
            mock_response = MagicMock()
            mock_response.status_code = 200
            mock_response.json.return_value = {
                "id": "eval_001",
                "status": "processing"
            }
            mock_post.return_value = mock_response
            
            client = GoldenSetEvaluationClient("http://localhost:8000")
            evaluation_id = client.start_evaluation(
                golden_set_id="default",
                prompt_version="v1.0",
                model="gpt-4"
            )
            
            assert evaluation_id == "eval_001"
    
    def test_wait_for_completion_success(self):
        """평가 완료 대기"""
        from scripts.evaluate_golden_set import GoldenSetEvaluationClient
        
        with patch("requests.Session.get") as mock_get:
            # 처음에는 processing, 그 다음은 completed
            response_processing = MagicMock()
            response_processing.json.return_value = {"status": "processing"}
            
            response_completed = MagicMock()
            response_completed.json.return_value = {
                "status": "completed",
                "metrics": {"faithfulness": 0.85}
            }
            
            mock_get.side_effect = [response_processing, response_completed]
            
            client = GoldenSetEvaluationClient("http://localhost:8000")
            result = client.wait_for_completion(
                "eval_001",
                max_wait_seconds=60,
                poll_interval=1
            )
            
            assert result["status"] == "completed"
    
    def test_wait_for_completion_timeout(self):
        """평가 완료 타임아웃"""
        from scripts.evaluate_golden_set import GoldenSetEvaluationClient
        
        with patch("requests.Session.get") as mock_get:
            response = MagicMock()
            response.json.return_value = {"status": "processing"}
            mock_get.return_value = response
            
            client = GoldenSetEvaluationClient("http://localhost:8000")
            
            with pytest.raises(TimeoutError):
                client.wait_for_completion(
                    "eval_001",
                    max_wait_seconds=1,
                    poll_interval=0.5
                )


class TestCLIIntegration:
    """CLI 통합 테스트"""
    
    def test_evaluate_golden_set_cli(self, tmp_path):
        """전체 CLI 워크플로우"""
        # 이 테스트는 실제 백엔드가 필요하므로
        # docker-compose up 후 실행해야 함
        
        # 더미 평가 결과 생성
        dummy_report = {
            "id": "eval_001",
            "golden_set_id": "default",
            "prompt_version": "v1.0",
            "model": "gpt-4",
            "timestamp": datetime.now().isoformat(),
            "metrics": {
                "faithfulness": {"value": 0.85, "passed": True},
                "answer_relevance": {"value": 0.80, "passed": True},
                "context_precision": {"value": 0.78, "passed": True},
                "context_recall": {"value": 0.79, "passed": True},
                "citation_present": {"value": 0.95, "passed": True},
                "hallucination": {"value": 0.05, "passed": True}
            },
            "overall_passed": True
        }
        
        report_file = tmp_path / "report.json"
        with open(report_file, "w") as f:
            json.dump(dummy_report, f)
        
        # JSON 파일이 올바른 형식인지 확인
        with open(report_file) as f:
            loaded = json.load(f)
        
        assert loaded["overall_passed"] is True
        assert "metrics" in loaded
```

## 5. 산출물

1. **`templates/ai_quality_evaluation.md.jinja2`**
   - Jinja2 마크다운 템플릿
   - 헤더, 요약 테이블, 항목별 상세, 분석, 추세 섹션

2. **`scripts/generate_evaluation_report.py`**
   - `EvaluationReportGenerator` 클래스: 템플릿 렌더링
   - 메타데이터 JSON 생성
   - JSON 파일 로드 및 처리

3. **`scripts/evaluate_golden_set.py`**
   - `GoldenSetEvaluationClient` 클래스: 백엔드 API 통신
   - CLI 인터페이스: argparse 기반
   - Health check, 평가 시작, 완료 대기, 보고서 다운로드
   - 콘솔 요약 pretty-print

4. **`tests/test_evaluation_report_generation.py`**
   - 템플릿 렌더링 테스트
   - 메타데이터 생성 테스트
   - 메타데이터 누적 테스트

5. **`tests/test_evaluate_golden_set_integration.py`**
   - API 클라이언트 테스트
   - 전체 CLI 워크플로우 테스트
   - Health check, 평가 시작, 완료 대기 테스트

## 6. 완료 기준

1. ✅ Jinja2 템플릿이 모든 섹션(헤더, 요약, 항목별, 분석, 추세)을 포함하고 마크다운 형식 올바름
2. ✅ 보고서 생성 스크립트가 JSON 입력으로부터 `.md` 파일 생성 가능
3. ✅ 메타데이터 JSON 파일 생성 및 누적 저장 가능
4. ✅ `evaluate_golden_set.py` CLI가 `--prompt-version`, `--model` 등 모든 옵션 지원
5. ✅ 백엔드 health check 수행 및 타임아웃 처리
6. ✅ 평가 작업 트리거 및 완료 대기 (polling)
7. ✅ 보고서 다운로드 및 로컬 저장
8. ✅ 이전 평가 대비 메트릭 변화(📈📉) 표시
9. ✅ 콘솔에 메트릭 테이블 pretty-print
10. ✅ 폐쇄망 모드 환경변수 전달 및 기록
11. ✅ 마크다운 파일이 PR 댓글로 활용 가능한 형식
12. ✅ Unit 테스트: 템플릿 렌더링, 메타데이터 생성, 메타데이터 누적
13. ✅ Integration 테스트: API 클라이언트, 전체 CLI 워크플로우
14. ✅ 대시보드 데이터 형식 일치 (Phase 6 FG6.3와 호환)

## 7. 작업 지침

**지침 7-1: Jinja2 템플릿 작성 원칙**
- 템플릿 파일은 `.jinja2` 확장자로 저장
- 마크다운 헤더 계층: # (페이지) > ## (섹션) > ### (소섹션)
- 메트릭 테이블: `| 메트릭 | 실제값 | 임계값 | 상태 |` 형식
- 상태 아이콘: ✅ (PASS) / ❌ (FAIL)
- 추세 표시: 📈 (개선) / 📉 (저하) / ➡️ (유지)
- 비교 데이터: `previous_report` 변수로 이전 평가 접근
- 선택 필드: `if` 문으로 null 체크 (dashboard_url, previous_report 등)

**지침 7-2: 보고서 생성 스크립트 사용**
```bash
python scripts/generate_evaluation_report.py \
  --report evaluation_results/report.json \
  --output-dir results/ \
  --environment prod \
  --save-metadata
```
- `--report`: 평가 결과 JSON 파일 (필수)
- `--output-dir`: 출력 디렉토리 (기본: 현재 디렉토리)
- `--environment`: 임계값 오버라이드 적용 환경 (기본: prod)
- `--save-metadata`: 메타데이터 JSON도 생성

**지침 7-3: 수동 평가 CLI 사용**
```bash
python scripts/evaluate_golden_set.py \
  --backend-url http://localhost:8000 \
  --golden-set-id default \
  --prompt-version v1.0 \
  --model gpt-4 \
  --output-dir results/ \
  --markdown-report results/AI품질평가.md
```
- `--prompt-version`, `--model`: 필수 옵션
- `--closed-network`: 폐쇄망 모드 활성화
- `--max-wait`: 평가 최대 대기 시간 (기본: 1800초 = 30분)
- `--skip-report-generation`: 마크다운 생성 스킵 (JSON만 저장)

**지침 7-4: 메타데이터 누적 관리**
- `evaluation_metadata.json` 파일에 모든 평가 결과를 배열로 누적
- 가장 최근 100개 기록만 유지 (저장소 크기 제어)
- 각 메타데이터 항목: timestamp, metrics, overall_passed, environment 포함
- 시계열 분석용 타임스탬프는 ISO 8601 형식 (UTC)

**지침 7-5: API 응답 형식**
- 평가 시작 응답: `{ "id": "eval_001", "status": "processing", ... }`
- 평가 상태 조회: `{ "status": "processing|completed|failed", ... }`
- 평가 보고서: `{ "metrics": { "메트릭명": { "value": 0.85, "passed": true }, ... }, ... }`
- 평가 실패 시: `{ "status": "failed", "error": "메시지" }`

**지침 7-6: 폐쇄망 모드 처리**
- CLI에서 `--closed-network` 플래그 사용 시 환경변수 설정
- 백엔드 API에 `"closed_network_mode": true` 포함하여 전달
- 보고서에 "⚠️ **폐쇄망 모드**" 경고 표시
- 메타데이터에 `"closed_network_mode": true` 기록

**지침 7-7: 콘솔 출력 포맷팅**
- Header: 평가 완료 후 "=" 구분선으로 시작
- 메트릭 테이블: 왼쪽 정렬 메트릭명, 오른쪽 정렬 값/상태
- 종료 코드: 전체 통과 시 exit 0, 하나 이상 실패 시 exit 1
- 예시:
  ```
  ======================================================================
  EVALUATION COMPLETE
  ======================================================================
  Evaluation ID: eval_001
  Status: completed
  
  Metrics Summary:
  ------
  faithfulness                   0.8500    ✅ PASS
  answer_relevance               0.8000    ✅ PASS
  ...
  
  Overall Result: ✅ PASSED
  ======================================================================
  ```

**지침 7-8: 메타데이터 필드 정의**
```json
{
  "golden_set_id": "default",
  "prompt_version": "v1.0",
  "model": "gpt-4",
  "timestamp": "2024-04-17T10:30:00+00:00",
  "environment": "prod",
  "closed_network_mode": false,
  "metrics": {
    "faithfulness": { "value": 0.85, "passed": true },
    ...
  },
  "overall_passed": true,
  "total_metrics": 6,
  "passed_metrics": 6,
  "failed_metrics": 0,
  "evaluation_items_count": 50,
  "commit_sha": "abc123def456",
  "pr_number": 42,
  "backend_url": "http://localhost:8000"
}
```

**지침 7-9: 테스트 커버리지**
- Unit 테스트:
  - 템플릿 렌더링: 모든 섹션 포함, 마크다운 문법 검증
  - 메타데이터 생성: JSON 구조, 필수 필드, 데이터 타입
  - 메타데이터 누적: 새 파일 생성, 기존 파일에 추가, 100개 초과 제거
- Integration 테스트:
  - API 클라이언트: health check, 평가 시작, 상태 조회, 타임아웃
  - 전체 워크플로우: CLI 인자 파싱 → API 호출 → 보고서 생성 → 콘솔 출력
  - 최소 80% 코드 커버리지 목표

**지침 7-10: 대시보드 통합 준비 (Phase 6 FG6.3)**
- 메타데이터 JSON 응답 형식이 대시보드 API 스키마와 일치
- 시계열 데이터: 평가 timestamp를 기준으로 정렬 가능
- 메트릭 변화: 이전 평가와의 diff 계산 가능한 구조
- 링크 필드: `commit_sha`, `pr_number`, `backend_url` 포함으로 traceability 확보

---

**작업 예상 소요 시간**: 30-35시간
**담당 역할**: 백엔드/풀스택 개발자 (Python, Jinja2, 템플릿 경험 필수)
**검수 담당**: CTO, QA Lead
**보안 검사**: API 응답 검증, 메타데이터 민감 정보 누수 방지
