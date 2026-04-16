# Task 8-3: 스키마 마이그레이션 + 필드 매핑 도구

**작업 식별자**: FG8.1-task8-3  
**작업 단계**: Phase 8 - DocumentType metadata schema → 추출 타겟 스키마  
**예상 소요 시간**: 36-44시간  
**담당자**: [Backend/Data Engineer]  

---

## 1. 작업 목적

Task 8-1, 8-2에서 구현한 추출 스키마의 기본 CRUD 기능에 더하여, **스키마 버전 간 마이그레이션 및 기존 추출 결과의 호환성 검증**을 제공한다.

주요 목표:
1. **스키마 마이그레이션 규칙 정의** — 이전 버전 스키마 → 새 버전 스키마로의 필드 매핑 규칙
2. **마이그레이션 변환 작업** — rename, type_cast, merge, split 등 4가지 변환 타입 지원
3. **마이그레이션 프리뷰** — 실제 마이그레이션 전 샘플 데이터로 검증
4. **호환성 검사** — 기존 추출 결과의 새 스키마 호환성 판정
5. **역호환성 보고서** — 호환성 여부 및 영향 분석 리포트 생성
6. **단위 + 통합 테스트** — 마이그레이션 로직 및 end-to-end 플로우 검증

---

## 2. 작업 범위

### 포함 범위

#### 2.1 마이그레이션 규칙 도메인 모델

**MigrationRule** — 필드 변환 규칙
```
source_field: str (이전 필드명)
target_field: str (새 필드명)
transform_type: Literal["rename", "type_cast", "merge", "split"]
transform_config: Dict[str, Any] (변환별 설정)
is_required: bool (새 스키마에서 required인지)
fallback_value: Optional[Any] (변환 실패 시 기본값)
validation_rule: Optional[str] (변환 후 검증 규칙)
```

**MigrationPlan** — 마이그레이션 계획
```
id: UUID
from_schema_version: int (이전 버전)
to_schema_version: int (새 버전)
doc_type_id: UUID
rules: List[MigrationRule] (필드별 변환 규칙)
created_at: datetime
created_by: str (actor_id)
created_by_type: str (actor_type)
status: Literal["draft", "approved", "active", "deprecated"]
execution_count: int (마이그레이션 실행 횟수)
last_executed_at: Optional[datetime]
notes: Optional[str]
```

**MigrationResult** — 개별 레코드 마이그레이션 결과
```
id: UUID
migration_plan_id: UUID
source_data: Dict[str, Any] (이전 스키마 데이터)
target_data: Dict[str, Any] (새 스키마 데이터)
status: Literal["success", "warning", "error"]
issues: List[Dict] (경고/오류 목록)
execution_time_ms: float
created_at: datetime
```

**CompatibilityReport** — 역호환성 보고서
```
id: UUID
doc_type_id: UUID
from_version: int
to_version: int
total_records: int
compatible_records: int
incompatible_records: int
compatibility_percentage: float
issues_by_field: Dict[str, List[str]]
migration_rules_required: List[MigrationRule]
recommendation: str ("safe_to_migrate" | "review_required" | "manual_fix_required")
generated_at: datetime
```

#### 2.2 마이그레이션 변환 로직

**변환 타입 1: rename**
- 필드명 변경만 수행
- 데이터는 그대로 유지
- 예: "invoice_no" → "invoice_number"
- Config:
  ```json
  {
    "new_field_name": "invoice_number"
  }
  ```

**변환 타입 2: type_cast**
- 필드 타입 변환 (string ↔ number, etc.)
- 변환 함수 또는 포맷 지정
- 예: string "1000" → number 1000
- Config:
  ```json
  {
    "target_type": "number",
    "cast_function": "float",
    "format": "decimal"
  }
  ```

**변환 타입 3: merge**
- 여러 필드를 하나로 병합
- 구분자 또는 템플릿 지정
- 예: "first_name", "last_name" → "full_name"
- Config:
  ```json
  {
    "merge_fields": ["first_name", "last_name"],
    "separator": " ",
    "target_field": "full_name"
  }
  ```

**변환 타입 4: split**
- 하나의 필드를 여러 필드로 분리
- 분리자 또는 정규식 패턴 지정
- 예: "2026-04-17" → "year", "month", "day"
- Config:
  ```json
  {
    "source_field": "date",
    "target_fields": ["year", "month", "day"],
    "separator": "-",
    "pattern": "(?P<year>\\d{4})-(?P<month>\\d{2})-(?P<day>\\d{2})"
  }
  ```

#### 2.3 API 엔드포인트

**1. POST /api/v1/extraction-schemas/{doc_type}/migrate**
- 목표: 마이그레이션 계획 정의 (또는 기존 계획 업데이트)
- 요청:
  ```json
  {
    "from_version": 1,
    "to_version": 2,
    "rules": [
      {
        "source_field": "invoice_no",
        "target_field": "invoice_number",
        "transform_type": "rename",
        "transform_config": {...}
      },
      ...
    ]
  }
  ```
- 응답: MigrationPlanResponse (201 또는 200)

**2. POST /api/v1/extraction-schemas/{doc_type}/migrate/preview**
- 목표: 마이그레이션 프리뷰 (샘플 데이터로 검증)
- 요청:
  ```json
  {
    "migration_plan_id": "uuid",
    "sample_records": [
      {"invoice_no": "INV-001", "amount": "1000", ...},
      {...}
    ]
  }
  ```
- 응답:
  ```json
  {
    "data": {
      "migration_plan_id": "...",
      "total_samples": 2,
      "successful_migrations": 1,
      "failed_migrations": 1,
      "results": [
        {
          "source": {...},
          "target": {...},
          "status": "success",
          "issues": []
        },
        {...}
      ]
    }
  }
  ```

**3. GET /api/v1/extraction-schemas/{doc_type}/compatibility**
- 목표: 역호환성 검사 (기존 추출 결과와 새 스키마의 호환성)
- 요청: Query params
  - from_version: int
  - to_version: int
  - sample_size: int (기본: 100)
- 응답: CompatibilityReportResponse

**4. POST /api/v1/extraction-schemas/{doc_type}/migrate/execute**
- 목표: 마이그레이션 실행 (extraction_results 테이블의 결과 일괄 변환)
- 요청:
  ```json
  {
    "migration_plan_id": "uuid",
    "batch_size": 1000,
    "dry_run": false
  }
  ```
- 응답:
  ```json
  {
    "data": {
      "migration_plan_id": "...",
      "status": "completed",
      "total_records": 5000,
      "migrated_records": 4950,
      "failed_records": 50,
      "duration_seconds": 123,
      "errors": [...]
    }
  }
  ```

#### 2.4 마이그레이션 로직 구현

**TransformationEngine** — 변환 실행 엔진

```python
class TransformationEngine:
    """필드 변환 엔진"""
    
    async def apply_transformation(
        self,
        rule: MigrationRule,
        source_data: Dict[str, Any],
        target_schema: ExtractionTargetSchema,
    ) -> Tuple[Dict[str, Any], List[Dict[str, str]]]:
        """
        변환 규칙을 적용하여 source_data를 변환
        
        Returns:
            (transformed_data, issues_list)
            - transformed_data: 변환된 데이터
            - issues_list: 경고/오류 목록
        """
        
        if rule.transform_type == "rename":
            return self._apply_rename(rule, source_data)
        elif rule.transform_type == "type_cast":
            return self._apply_type_cast(rule, source_data, target_schema)
        elif rule.transform_type == "merge":
            return self._apply_merge(rule, source_data, target_schema)
        elif rule.transform_type == "split":
            return self._apply_split(rule, source_data, target_schema)
        else:
            raise ValueError(f"Unknown transform_type: {rule.transform_type}")
    
    async def _apply_rename(
        self,
        rule: MigrationRule,
        source_data: Dict[str, Any],
    ) -> Tuple[Dict[str, Any], List[Dict]]:
        """rename 변환: 필드명 변경"""
        issues = []
        
        if rule.source_field not in source_data:
            if rule.is_required:
                issues.append({
                    "type": "error",
                    "message": f"필수 필드 '{rule.source_field}' 없음"
                })
                return {}, issues
            else:
                return {}, issues
        
        return {
            rule.target_field: source_data[rule.source_field]
        }, issues
    
    async def _apply_type_cast(
        self,
        rule: MigrationRule,
        source_data: Dict[str, Any],
        target_schema: ExtractionTargetSchema,
    ) -> Tuple[Dict[str, Any], List[Dict]]:
        """type_cast 변환: 데이터 타입 변환"""
        issues = []
        
        if rule.source_field not in source_data:
            if rule.is_required:
                issues.append({
                    "type": "error",
                    "message": f"필수 필드 '{rule.source_field}' 없음"
                })
                return {}, issues
            return {}, issues
        
        source_value = source_data[rule.source_field]
        target_type = rule.transform_config.get("target_type")
        
        try:
            # 타입별 캐스팅
            if target_type == "number":
                target_value = float(source_value)
            elif target_type == "string":
                target_value = str(source_value)
            elif target_type == "boolean":
                target_value = self._to_boolean(source_value)
            elif target_type == "date":
                target_value = self._parse_date(source_value)
            else:
                raise ValueError(f"Unsupported target_type: {target_type}")
            
            return {
                rule.target_field: target_value
            }, issues
        
        except Exception as e:
            if rule.fallback_value is not None:
                issues.append({
                    "type": "warning",
                    "message": f"캐스팅 실패, 기본값 사용: {e}"
                })
                return {
                    rule.target_field: rule.fallback_value
                }, issues
            else:
                issues.append({
                    "type": "error",
                    "message": f"캐스팅 실패: {e}"
                })
                return {}, issues
    
    async def _apply_merge(
        self,
        rule: MigrationRule,
        source_data: Dict[str, Any],
        target_schema: ExtractionTargetSchema,
    ) -> Tuple[Dict[str, Any], List[Dict]]:
        """merge 변환: 여러 필드 병합"""
        issues = []
        
        merge_fields = rule.transform_config.get("merge_fields", [])
        separator = rule.transform_config.get("separator", " ")
        
        # 병합할 필드들 수집
        values = []
        for field in merge_fields:
            if field in source_data:
                val = source_data[field]
                if val is not None:
                    values.append(str(val))
            else:
                if rule.is_required:
                    issues.append({
                        "type": "warning",
                        "message": f"병합 필드 '{field}' 없음"
                    })
        
        if not values:
            if rule.is_required:
                issues.append({
                    "type": "error",
                    "message": f"병합할 필드 없음"
                })
                return {}, issues
            return {}, issues
        
        merged_value = separator.join(values)
        return {
            rule.target_field: merged_value
        }, issues
    
    async def _apply_split(
        self,
        rule: MigrationRule,
        source_data: Dict[str, Any],
        target_schema: ExtractionTargetSchema,
    ) -> Tuple[Dict[str, Any], List[Dict]]:
        """split 변환: 하나의 필드를 여러 필드로 분리"""
        import re
        issues = []
        
        source_field = rule.transform_config.get("source_field")
        if source_field not in source_data:
            if rule.is_required:
                issues.append({
                    "type": "error",
                    "message": f"필드 '{source_field}' 없음"
                })
            return {}, issues
        
        source_value = str(source_data[source_field])
        target_fields = rule.transform_config.get("target_fields", [])
        pattern = rule.transform_config.get("pattern")
        separator = rule.transform_config.get("separator")
        
        result = {}
        
        # 정규식 패턴 사용
        if pattern:
            try:
                match = re.match(pattern, source_value)
                if match:
                    groups = match.groupdict()
                    for field in target_fields:
                        if field in groups:
                            result[field] = groups[field]
                        else:
                            issues.append({
                                "type": "warning",
                                "message": f"분리 필드 '{field}'를 패턴에서 찾을 수 없음"
                            })
                else:
                    issues.append({
                        "type": "error",
                        "message": f"값 '{source_value}'가 패턴과 일치하지 않음"
                    })
            except Exception as e:
                issues.append({
                    "type": "error",
                    "message": f"정규식 처리 오류: {e}"
                })
        
        # 구분자 사용
        elif separator:
            parts = source_value.split(separator)
            for i, field in enumerate(target_fields):
                if i < len(parts):
                    result[field] = parts[i]
                else:
                    issues.append({
                        "type": "warning",
                        "message": f"분리 필드 '{field}'에 대한 값 없음"
                    })
        
        return result, issues
    
    @staticmethod
    def _to_boolean(value: Any) -> bool:
        """다양한 형식의 값을 bool로 변환"""
        if isinstance(value, bool):
            return value
        if isinstance(value, str):
            return value.lower() in ("true", "1", "yes", "y")
        return bool(value)
    
    @staticmethod
    def _parse_date(value: Any) -> str:
        """다양한 형식의 날짜를 ISO 8601 형식으로 변환"""
        from datetime import datetime
        if isinstance(value, str):
            # 간단한 ISO 파싱
            try:
                dt = datetime.fromisoformat(value.replace("Z", "+00:00"))
                return dt.date().isoformat()
            except:
                pass
        return str(value)
```

**MigrationService** — 마이그레이션 관리 서비스

```python
class MigrationService:
    """스키마 마이그레이션 서비스"""
    
    def __init__(
        self,
        session: Session,
        extraction_repo: ExtractionSchemaRepository,
        transformation_engine: TransformationEngine,
    ):
        self.session = session
        self.extraction_repo = extraction_repo
        self.engine = transformation_engine
    
    async def create_migration_plan(
        self,
        doc_type_id: UUID,
        from_version: int,
        to_version: int,
        rules: List[MigrationRule],
        actor_info: ActorInfo,
        notes: Optional[str] = None,
    ) -> MigrationPlan:
        """마이그레이션 계획 생성"""
        
        # 버전 유효성 확인
        from_schema = self.extraction_repo.get_by_doc_type_and_version(
            doc_type_id, from_version
        )
        to_schema = self.extraction_repo.get_by_doc_type_and_version(
            doc_type_id, to_version
        )
        
        if not from_schema or not to_schema:
            raise NotFoundError("스키마 버전을 찾을 수 없음")
        
        if to_version <= from_version:
            raise ValidationError("to_version은 from_version보다 커야 함")
        
        # 마이그레이션 계획 검증
        await self._validate_migration_plan(from_schema, to_schema, rules)
        
        # ORM 저장
        plan_orm = MigrationPlanORM(
            doc_type_id=doc_type_id,
            from_schema_version=from_version,
            to_schema_version=to_version,
            rules_json=[r.model_dump() for r in rules],
            created_by=actor_info.actor_id,
            created_by_type=actor_info.actor_type,
            status="draft",
            notes=notes,
        )
        self.session.add(plan_orm)
        self.session.commit()
        
        return plan_orm.to_pydantic()
    
    async def preview_migration(
        self,
        migration_plan_id: UUID,
        sample_records: List[Dict[str, Any]],
    ) -> List[MigrationResult]:
        """마이그레이션 프리뷰 (샘플로 검증)"""
        
        plan = self.session.query(MigrationPlanORM).filter_by(
            id=migration_plan_id
        ).one_or_none()
        
        if not plan:
            raise NotFoundError("마이그레이션 계획을 찾을 수 없음")
        
        plan_obj = plan.to_pydantic()
        to_schema = self.extraction_repo.get_by_doc_type_and_version(
            plan.doc_type_id, plan.to_schema_version
        )
        
        results = []
        
        for sample in sample_records:
            target_data = {}
            issues = []
            
            for rule in plan_obj.rules:
                try:
                    transformed, rule_issues = await self.engine.apply_transformation(
                        rule, sample, to_schema
                    )
                    target_data.update(transformed)
                    issues.extend(rule_issues)
                except Exception as e:
                    issues.append({
                        "type": "error",
                        "message": str(e)
                    })
            
            status = "error" if any(i["type"] == "error" for i in issues) else (
                "warning" if issues else "success"
            )
            
            result = MigrationResult(
                id=uuid4(),
                migration_plan_id=migration_plan_id,
                source_data=sample,
                target_data=target_data,
                status=status,
                issues=issues,
                execution_time_ms=0.0,
                created_at=datetime.utcnow(),
            )
            results.append(result)
        
        return results
    
    async def check_compatibility(
        self,
        doc_type_id: UUID,
        from_version: int,
        to_version: int,
        sample_size: int = 100,
    ) -> CompatibilityReport:
        """기존 추출 결과와 새 스키마의 호환성 검사"""
        
        # extraction_results 테이블에서 해당 doc_type의 결과 샘플 조회
        # (실제 extraction_results 테이블 스키마에 따라 조정 필요)
        
        from_schema = self.extraction_repo.get_by_doc_type_and_version(
            doc_type_id, from_version
        )
        to_schema = self.extraction_repo.get_by_doc_type_and_version(
            doc_type_id, to_version
        )
        
        if not from_schema or not to_schema:
            raise NotFoundError("스키마 버전을 찾을 수 없음")
        
        # 호환성 규칙 정의
        incompatibilities = self._check_schema_compatibility(from_schema, to_schema)
        
        report = CompatibilityReport(
            id=uuid4(),
            doc_type_id=doc_type_id,
            from_version=from_version,
            to_version=to_version,
            total_records=sample_size,
            compatible_records=sample_size - len(incompatibilities),
            incompatible_records=len(incompatibilities),
            compatibility_percentage=(
                (sample_size - len(incompatibilities)) / sample_size * 100
                if sample_size > 0 else 0
            ),
            issues_by_field=incompatibilities,
            recommendation=self._get_recommendation(incompatibilities),
            generated_at=datetime.utcnow(),
        )
        
        return report
    
    def _check_schema_compatibility(
        self,
        from_schema: ExtractionTargetSchema,
        to_schema: ExtractionTargetSchema,
    ) -> Dict[str, List[str]]:
        """스키마 호환성 검사"""
        issues = {}
        
        from_fields = set(from_schema.fields.keys())
        to_fields = set(to_schema.fields.keys())
        
        # 제거된 필드 (required 필드면 호환성 문제)
        removed_fields = from_fields - to_fields
        for field in removed_fields:
            if to_schema.fields.get(field, {}).required:
                if "removed_required_fields" not in issues:
                    issues["removed_required_fields"] = []
                issues["removed_required_fields"].append(field)
        
        # 새로 추가된 필수 필드 (기존 데이터에 없을 수 있음)
        added_required_fields = [
            f for f in to_fields
            if f not in from_fields and to_schema.fields[f].required
        ]
        if added_required_fields:
            issues["added_required_fields"] = added_required_fields
        
        # 필드 타입 변경 (호환성 문제 가능)
        common_fields = from_fields & to_fields
        for field in common_fields:
            from_type = from_schema.fields[field].field_type
            to_type = to_schema.fields[field].field_type
            if from_type != to_type:
                if "type_changed_fields" not in issues:
                    issues["type_changed_fields"] = []
                issues["type_changed_fields"].append(
                    f"{field}: {from_type} → {to_type}"
                )
        
        return issues
    
    def _get_recommendation(self, issues: Dict[str, List]) -> str:
        """호환성 분석 결과에 따른 권장사항"""
        if not issues:
            return "safe_to_migrate"
        elif "removed_required_fields" in issues or "type_changed_fields" in issues:
            return "manual_fix_required"
        else:
            return "review_required"
    
    async def execute_migration(
        self,
        migration_plan_id: UUID,
        batch_size: int = 1000,
        dry_run: bool = False,
    ) -> Dict[str, Any]:
        """마이그레이션 실행 (extraction_results 테이블의 결과 일괄 변환)"""
        
        plan = self.session.query(MigrationPlanORM).filter_by(
            id=migration_plan_id
        ).one_or_none()
        
        if not plan:
            raise NotFoundError("마이그레이션 계획을 찾을 수 없음")
        
        plan_obj = plan.to_pydantic()
        to_schema = self.extraction_repo.get_by_doc_type_and_version(
            plan.doc_type_id, plan.to_schema_version
        )
        
        # extraction_results에서 해당 데이터 조회 (페이징)
        # (실제 구현은 extraction_results 테이블 구조에 따라 조정)
        
        total_records = 0
        migrated_records = 0
        failed_records = 0
        start_time = datetime.utcnow()
        
        # 배치 처리
        offset = 0
        while True:
            # extraction_results 조회
            records = self._fetch_extraction_results(
                doc_type_id=plan.doc_type_id,
                schema_version=plan.from_schema_version,
                limit=batch_size,
                offset=offset,
            )
            
            if not records:
                break
            
            for record in records:
                total_records += 1
                try:
                    # 마이그레이션 적용
                    target_data = {}
                    
                    for rule in plan_obj.rules:
                        transformed, _ = await self.engine.apply_transformation(
                            rule, record["extracted_data"], to_schema
                        )
                        target_data.update(transformed)
                    
                    if not dry_run:
                        # extraction_results 업데이트
                        self._update_extraction_result(
                            record["id"],
                            target_data,
                            plan.to_schema_version,
                        )
                    
                    migrated_records += 1
                
                except Exception as e:
                    failed_records += 1
                    # 오류 로깅
            
            offset += batch_size
        
        duration = (datetime.utcnow() - start_time).total_seconds()
        
        # 마이그레이션 계획 상태 업데이트
        if not dry_run:
            plan.status = "active"
            plan.execution_count += 1
            plan.last_executed_at = datetime.utcnow()
            self.session.commit()
        
        return {
            "migration_plan_id": str(migration_plan_id),
            "status": "completed",
            "total_records": total_records,
            "migrated_records": migrated_records,
            "failed_records": failed_records,
            "duration_seconds": duration,
            "dry_run": dry_run,
        }
```

#### 2.5 SQLAlchemy ORM 모델

**파일**: `src/infrastructure/database/models/migration.py`

```python
from datetime import datetime
from typing import Any, Dict, List, Optional
from uuid import UUID, uuid4

from sqlalchemy import (
    Boolean,
    Column,
    DateTime,
    ForeignKey,
    Index,
    Integer,
    String,
    Text,
    func,
)
from sqlalchemy.dialects.postgresql import JSONB, UUID as PG_UUID
from sqlalchemy.orm import relationship

from src.infrastructure.database.base import Base


class MigrationPlanORM(Base):
    """마이그레이션 계획 ORM"""
    
    __tablename__ = "migration_plans"
    
    id = Column(PG_UUID(as_uuid=True), primary_key=True, default=uuid4)
    doc_type_id = Column(
        PG_UUID(as_uuid=True),
        ForeignKey("document_types.id", ondelete="RESTRICT"),
        nullable=False,
        index=True,
    )
    from_schema_version = Column(Integer, nullable=False)
    to_schema_version = Column(Integer, nullable=False)
    
    # 마이그레이션 규칙 (JSON)
    rules_json = Column(JSONB, nullable=False)
    
    # 메타데이터
    status = Column(String(50), nullable=False, default="draft", index=True)
    notes = Column(Text, nullable=True)
    
    # 감사 정보
    created_at = Column(DateTime(timezone=True), nullable=False, default=func.now())
    created_by = Column(String(255), nullable=False)
    created_by_type = Column(String(50), nullable=False)  # "user" or "agent"
    
    # 실행 추적
    execution_count = Column(Integer, nullable=False, default=0)
    last_executed_at = Column(DateTime(timezone=True), nullable=True)
    
    # 인덱스
    __table_args__ = (
        Index("idx_doc_type_version", "doc_type_id", "from_schema_version", "to_schema_version"),
        Index("idx_status", "status"),
    )
    
    def to_pydantic(self) -> "MigrationPlan":
        """ORM → Pydantic 변환"""
        from src.domain.extraction.migration import MigrationPlan, MigrationRule
        
        rules = [
            MigrationRule(**rule_data)
            for rule_data in (self.rules_json or [])
        ]
        
        return MigrationPlan(
            id=self.id,
            from_schema_version=self.from_schema_version,
            to_schema_version=self.to_schema_version,
            doc_type_id=self.doc_type_id,
            rules=rules,
            created_at=self.created_at,
            created_by=self.created_by,
            created_by_type=self.created_by_type,
            status=self.status,
            execution_count=self.execution_count,
            last_executed_at=self.last_executed_at,
            notes=self.notes,
        )
```

#### 2.6 FastAPI 라우터

**파일**: `src/api/v1/routers/extraction_migrations.py`

```python
from typing import List, Optional
from uuid import UUID
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from src.api.v1.schemas.extraction_migrations import (
    CreateMigrationPlanRequest,
    PreviewMigrationRequest,
    ExecuteMigrationRequest,
    MigrationPlanResponse,
    MigrationPreviewResponse,
    CompatibilityReportResponse,
)
from src.domain.extraction.migration import MigrationService
from src.domain.extraction.repository import ActorInfo
from src.infrastructure.database.connection import get_session
from src.infrastructure.exceptions import NotFoundError, ValidationError


router = APIRouter(
    prefix="/extraction-schemas",
    tags=["extraction-migrations"],
)


def get_actor_info_from_header(...):
    """헤더에서 actor_info 추출 (task8-2와 동일)"""
    # ... 구현 동일 ...
    pass


@router.post(
    "/{doc_type}/migrate",
    response_model=MigrationPlanResponse,
    status_code=status.HTTP_201_CREATED,
    summary="마이그레이션 계획 생성",
)
async def create_migration_plan(
    doc_type: UUID,
    request: CreateMigrationPlanRequest,
    session: Session = Depends(get_session),
    actor_info: ActorInfo = Depends(get_actor_info_from_header),
):
    """
    마이그레이션 계획을 생성합니다.
    
    - 필드 변환 규칙 (rename, type_cast, merge, split)
    - 마이그레이션 검증
    """
    try:
        service = MigrationService(session=session, ...)
        
        plan = await service.create_migration_plan(
            doc_type_id=doc_type,
            from_version=request.from_version,
            to_version=request.to_version,
            rules=request.rules,
            actor_info=actor_info,
            notes=request.notes,
        )
        
        return MigrationPlanResponse(data=plan.model_dump())
    
    except ValidationError as e:
        raise HTTPException(status_code=400, detail=str(e))
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))


@router.post(
    "/{doc_type}/migrate/preview",
    response_model=MigrationPreviewResponse,
    summary="마이그레이션 프리뷰",
)
async def preview_migration(
    doc_type: UUID,
    request: PreviewMigrationRequest,
    session: Session = Depends(get_session),
    actor_info: ActorInfo = Depends(get_actor_info_from_header),
):
    """
    마이그레이션을 샘플 데이터로 검증합니다.
    
    - 실제 실행 전 확인
    - 변환 결과 및 오류 확인
    """
    try:
        service = MigrationService(session=session, ...)
        
        results = await service.preview_migration(
            migration_plan_id=request.migration_plan_id,
            sample_records=request.sample_records,
        )
        
        return MigrationPreviewResponse(
            data={
                "migration_plan_id": str(request.migration_plan_id),
                "total_samples": len(request.sample_records),
                "successful_migrations": sum(1 for r in results if r.status == "success"),
                "failed_migrations": sum(1 for r in results if r.status == "error"),
                "results": [r.model_dump() for r in results],
            }
        )
    
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))


@router.get(
    "/{doc_type}/compatibility",
    response_model=CompatibilityReportResponse,
    summary="호환성 검사",
)
async def check_compatibility(
    doc_type: UUID,
    from_version: int,
    to_version: int,
    sample_size: int = 100,
    session: Session = Depends(get_session),
    actor_info: ActorInfo = Depends(get_actor_info_from_header),
):
    """
    기존 추출 결과와 새 스키마의 호환성을 검사합니다.
    
    - 호환성 백분율
    - 문제 필드 목록
    - 권장사항
    """
    try:
        service = MigrationService(session=session, ...)
        
        report = await service.check_compatibility(
            doc_type_id=doc_type,
            from_version=from_version,
            to_version=to_version,
            sample_size=sample_size,
        )
        
        return CompatibilityReportResponse(data=report.model_dump())
    
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))


@router.post(
    "/{doc_type}/migrate/execute",
    response_model=dict,
    summary="마이그레이션 실행",
)
async def execute_migration(
    doc_type: UUID,
    request: ExecuteMigrationRequest,
    session: Session = Depends(get_session),
    actor_info: ActorInfo = Depends(get_actor_info_from_header),
):
    """
    마이그레이션을 실행합니다.
    
    - extraction_results 테이블의 데이터 일괄 변환
    - dry_run 옵션으로 검증 후 실행 가능
    """
    try:
        service = MigrationService(session=session, ...)
        
        result = await service.execute_migration(
            migration_plan_id=request.migration_plan_id,
            batch_size=request.batch_size,
            dry_run=request.dry_run,
        )
        
        return {
            "data": result,
            "meta": {
                "request_id": "...",
                "timestamp": datetime.utcnow().isoformat(),
            }
        }
    
    except NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))
```

---

### 제외 범위

- 자동 마이그레이션 규칙 생성 (AI 기반, 별도 작업)
- 마이그레이션 롤백 (버전 관리 방식에 따라 별도 고려)
- UI 통합 (별도 작업)

---

## 3. 선행 조건

1. **Task 8-1, 8-2 완료**
   - ExtractionTargetSchema, ExtractionFieldDef 모델
   - ExtractionSchemaRepository, CRUD API 완성
   - task8-2의 API 라우터 완성

2. **extraction_results 테이블**
   - 기존 추출 결과를 저장하는 테이블
   - doc_type_id, schema_version, extracted_data(JSONB) 컬럼 필수

3. **ScopeProfileService**
   - ACL 필터링 지원

4. **테스트 환경**
   - pytest, pytest-asyncio
   - 마이그레이션용 샘플 데이터

---

## 4. 주요 작업 항목

### 4.1 마이그레이션 도메인 모델

**파일**: `src/domain/extraction/migration.py`

```python
from typing import Any, Dict, List, Literal, Optional
from uuid import UUID
from datetime import datetime
from pydantic import BaseModel, Field


class MigrationRule(BaseModel):
    """필드 변환 규칙"""
    
    source_field: str = Field(..., description="이전 필드명")
    target_field: str = Field(..., description="새 필드명")
    transform_type: Literal["rename", "type_cast", "merge", "split"] = Field(
        ..., description="변환 타입"
    )
    transform_config: Dict[str, Any] = Field(
        ..., description="변환별 설정"
    )
    is_required: bool = Field(default=False, description="새 스키마에서 필수인가")
    fallback_value: Optional[Any] = Field(
        None, description="변환 실패 시 기본값"
    )
    validation_rule: Optional[str] = Field(
        None, description="변환 후 검증 규칙"
    )


class MigrationPlan(BaseModel):
    """마이그레이션 계획"""
    
    id: UUID
    from_schema_version: int
    to_schema_version: int
    doc_type_id: UUID
    rules: List[MigrationRule]
    created_at: datetime
    created_by: str
    created_by_type: str  # "user" or "agent"
    status: Literal["draft", "approved", "active", "deprecated"] = "draft"
    execution_count: int = 0
    last_executed_at: Optional[datetime] = None
    notes: Optional[str] = None


class MigrationResult(BaseModel):
    """개별 레코드 마이그레이션 결과"""
    
    id: UUID
    migration_plan_id: UUID
    source_data: Dict[str, Any]
    target_data: Dict[str, Any]
    status: Literal["success", "warning", "error"]
    issues: List[Dict[str, str]]
    execution_time_ms: float
    created_at: datetime


class CompatibilityReport(BaseModel):
    """역호환성 보고서"""
    
    id: UUID
    doc_type_id: UUID
    from_version: int
    to_version: int
    total_records: int
    compatible_records: int
    incompatible_records: int
    compatibility_percentage: float
    issues_by_field: Dict[str, List[str]]
    recommendation: Literal[
        "safe_to_migrate", "review_required", "manual_fix_required"
    ]
    generated_at: datetime
```

### 4.2 TransformationEngine 구현

(위 2.4 섹션 참조)

### 4.3 MigrationService 구현

(위 2.4 섹션 참조)

### 4.4 ORM 모델

(위 2.5 섹션 참조)

### 4.5 API 라우터

(위 2.6 섹션 참조)

### 4.6 단위 및 통합 테스트

**파일**: `tests/domain/extraction/test_transformation_engine.py`

```python
import pytest
from uuid import uuid4

from src.domain.extraction.migration import MigrationRule
from src.domain.extraction.models import ExtractionFieldDef, ExtractionTargetSchema
from src.infrastructure.extraction.transformation import TransformationEngine


@pytest.fixture
def engine():
    return TransformationEngine()


@pytest.fixture
def target_schema():
    """테스트용 target 스키마"""
    return ExtractionTargetSchema(
        id=uuid4(),
        doc_type_id=uuid4(),
        version=2,
        fields={
            "invoice_number": ExtractionFieldDef(...),
            "full_name": ExtractionFieldDef(...),
            "invoice_date": ExtractionFieldDef(...),
        },
        created_at=datetime.utcnow(),
        updated_at=datetime.utcnow(),
        created_by="user_001",
        updated_by="user_001",
    )


class TestTransformationEngine:
    """TransformationEngine 테스트"""
    
    @pytest.mark.asyncio
    async def test_rename_transformation(self, engine):
        """rename 변환 테스트"""
        rule = MigrationRule(
            source_field="invoice_no",
            target_field="invoice_number",
            transform_type="rename",
            transform_config={},
            is_required=True,
        )
        
        source_data = {"invoice_no": "INV-001"}
        target, issues = await engine.apply_transformation(
            rule, source_data, None
        )
        
        assert target == {"invoice_number": "INV-001"}
        assert len(issues) == 0
    
    @pytest.mark.asyncio
    async def test_type_cast_string_to_number(self, engine):
        """type_cast 변환: string → number"""
        rule = MigrationRule(
            source_field="amount",
            target_field="amount_numeric",
            transform_type="type_cast",
            transform_config={"target_type": "number"},
            is_required=True,
        )
        
        source_data = {"amount": "1000.50"}
        target, issues = await engine.apply_transformation(
            rule, source_data, None
        )
        
        assert target["amount_numeric"] == 1000.50
        assert len(issues) == 0
    
    @pytest.mark.asyncio
    async def test_merge_transformation(self, engine):
        """merge 변환 테스트"""
        rule = MigrationRule(
            source_field=None,  # merge는 source_field 없음
            target_field="full_name",
            transform_type="merge",
            transform_config={
                "merge_fields": ["first_name", "last_name"],
                "separator": " "
            },
            is_required=True,
        )
        
        source_data = {"first_name": "홍", "last_name": "길동"}
        target, issues = await engine.apply_transformation(
            rule, source_data, None
        )
        
        assert target["full_name"] == "홍 길동"
        assert len(issues) == 0
    
    @pytest.mark.asyncio
    async def test_split_transformation_with_pattern(self, engine):
        """split 변환: 정규식 패턴"""
        rule = MigrationRule(
            source_field=None,
            target_field=None,  # split은 여러 target
            transform_type="split",
            transform_config={
                "source_field": "date",
                "target_fields": ["year", "month", "day"],
                "pattern": r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})"
            },
            is_required=True,
        )
        
        source_data = {"date": "2026-04-17"}
        target, issues = await engine.apply_transformation(
            rule, source_data, None
        )
        
        assert target["year"] == "2026"
        assert target["month"] == "04"
        assert target["day"] == "17"
        assert len(issues) == 0
    
    @pytest.mark.asyncio
    async def test_missing_required_field(self, engine):
        """필수 필드 누락 오류"""
        rule = MigrationRule(
            source_field="invoice_no",
            target_field="invoice_number",
            transform_type="rename",
            transform_config={},
            is_required=True,
        )
        
        source_data = {}  # invoice_no 없음
        target, issues = await engine.apply_transformation(
            rule, source_data, None
        )
        
        assert len(issues) > 0
        assert issues[0]["type"] == "error"
```

**파일**: `tests/domain/extraction/test_migration_service.py`

```python
import pytest
from uuid import uuid4

from src.domain.extraction.migration import (
    MigrationService,
    MigrationRule,
)
from src.domain.extraction.repository import ActorInfo


@pytest.mark.asyncio
async def test_create_migration_plan(
    migration_service,
    sample_extraction_schema,
):
    """마이그레이션 계획 생성 테스트"""
    
    rules = [
        MigrationRule(
            source_field="invoice_no",
            target_field="invoice_number",
            transform_type="rename",
            transform_config={},
            is_required=True,
        ),
    ]
    
    actor_info = ActorInfo(actor_id="user_001", actor_type="user")
    
    plan = await migration_service.create_migration_plan(
        doc_type_id=sample_extraction_schema.doc_type_id,
        from_version=1,
        to_version=2,
        rules=rules,
        actor_info=actor_info,
    )
    
    assert plan.from_schema_version == 1
    assert plan.to_schema_version == 2
    assert plan.status == "draft"
    assert len(plan.rules) == 1


@pytest.mark.asyncio
async def test_preview_migration(migration_service, migration_plan):
    """마이그레이션 프리뷰 테스트"""
    
    sample_records = [
        {"invoice_no": "INV-001", "amount": "1000"},
        {"invoice_no": "INV-002", "amount": "2000"},
    ]
    
    results = await migration_service.preview_migration(
        migration_plan_id=migration_plan.id,
        sample_records=sample_records,
    )
    
    assert len(results) == 2
    assert results[0].status in ("success", "warning", "error")


@pytest.mark.asyncio
async def test_check_compatibility(migration_service):
    """호환성 검사 테스트"""
    
    report = await migration_service.check_compatibility(
        doc_type_id=uuid4(),
        from_version=1,
        to_version=2,
        sample_size=100,
    )
    
    assert 0 <= report.compatibility_percentage <= 100
    assert report.recommendation in (
        "safe_to_migrate", "review_required", "manual_fix_required"
    )
```

---

## 5. 산출물 (Deliverables)

| 파일 경로 | 설명 | 요구사항 |
|---------|------|--------|
| `src/domain/extraction/migration.py` | 마이그레이션 도메인 모델 | 완성도 100% |
| `src/infrastructure/extraction/transformation.py` | TransformationEngine 구현 | 4가지 변환 타입 모두 |
| `src/domain/extraction/migration_service.py` | MigrationService 구현 | 완성도 100%, actor_type 포함 |
| `src/infrastructure/database/models/migration.py` | MigrationPlanORM, MigrationResultORM | 완성도 100% |
| `src/api/v1/routers/extraction_migrations.py` | 마이그레이션 API 라우터 (4개 엔드포인트) | 완성도 100% |
| `src/api/v1/schemas/extraction_migrations.py` | 요청/응답 Pydantic 모델 | 완성도 100% |
| `tests/domain/extraction/test_transformation_engine.py` | TransformationEngine 단위 테스트 | 20개 이상 케이스 |
| `tests/domain/extraction/test_migration_service.py` | MigrationService 단위 테스트 | 15개 이상 케이스 |
| `tests/api/v1/test_extraction_migrations_api.py` | 마이그레이션 API 통합 테스트 | 15개 이상 케이스 |
| `alembic/versions/{timestamp}_add_migration_tables.py` | Alembic 마이그레이션 | migration_plans, migration_results 테이블 |

---

## 6. 완료 기준

1. **마이그레이션 도메인 모델**
   - MigrationRule: 4가지 변환 타입 모두 지원
   - MigrationPlan: 계획 관리
   - MigrationResult: 결과 추적
   - CompatibilityReport: 호환성 분석

2. **TransformationEngine**
   - rename: 필드명 변경
   - type_cast: 타입 변환 (string, number, boolean, date)
   - merge: 여러 필드 병합
   - split: 필드 분리 (구분자, 정규식)
   - 오류 처리 및 fallback_value

3. **MigrationService**
   - create_migration_plan()
   - preview_migration()
   - check_compatibility()
   - execute_migration()
   - actor_type 추적

4. **API 엔드포인트** (4개)
   - POST /api/v1/extraction-schemas/{doc_type}/migrate
   - POST /api/v1/extraction-schemas/{doc_type}/migrate/preview
   - GET /api/v1/extraction-schemas/{doc_type}/compatibility
   - POST /api/v1/extraction-schemas/{doc_type}/migrate/execute

5. **호환성 검사**
   - 제거된 필드 감지
   - 새로운 필수 필드 감지
   - 필드 타입 변경 감지
   - 호환성 백분율 계산
   - 권장사항 제시

6. **단위 테스트**
   - TransformationEngine: 변환 타입별 20개 이상 케이스
   - MigrationService: 15개 이상 케이스
   - 오류 처리, fallback 검증
   - 커버리지 ≥90%

7. **통합 테스트**
   - API 엔드포인트별 15개 이상 케이스
   - end-to-end 마이그레이션 플로우
   - dry_run 검증
   - 커버리지 ≥85%

8. **감사 추적**
   - created_by_type (actor_type) 기록
   - 마이그레이션 실행 이력 추적

---

## 7. 작업 지침

**7-1**: **변환 규칙 구성** — MigrationRule은 source_field, target_field, transform_type, transform_config를 포함. transform_type별로 필요한 config 필드가 다름:
  - rename: {} (빈 config)
  - type_cast: {target_type, cast_function}
  - merge: {merge_fields: [], separator}
  - split: {source_field, target_fields: [], separator 또는 pattern}

**7-2**: **마이그레이션 검증** — create_migration_plan() 단계에서:
  - from_version, to_version 유효성 확인
  - 각 규칙의 source_field가 from_schema에 존재하는지 확인
  - 각 규칙의 target_field가 to_schema에 존재하는지 확인
  - 규칙 순서 (merge 규칙이 rename 규칙 이전에 와야 함 등)

**7-3**: **오류 처리 전략**:
  - 필수 필드 누락: error 상태, fallback_value 있으면 사용
  - 선택 필드 누락: warning 또는 skip
  - 타입 변환 실패: fallback_value 또는 error
  - 정규식 매칭 실패: error + message

**7-4**: **호환성 분석**:
  - 제거된 필드 (old에만 있음): 경고
  - 새로운 필수 필드 (new에만 있음 + required): 오류
  - 필드 타입 변경: 호환성 판정 (string ↔ number 가능, but number ↔ boolean 불가 등)
  - 호환성_percentage = (compatible_records / total_records * 100)

**7-5**: **dry_run 옵션** — execute_migration()의 dry_run=True일 때:
  - 마이그레이션 결과 계산
  - 데이터베이스 업데이트 금지
  - 최종 통계만 반환 (검증용)

**7-6**: **마이그레이션 계획 상태**:
  - draft: 생성됨, 아직 검증 안 함
  - approved: 검증 완료, 승인됨
  - active: 실행됨
  - deprecated: 더 이상 사용 안 함

**7-7**: **배치 처리** — execute_migration()에서 extraction_results를 배치로 처리:
  - batch_size 기본값: 1000
  - 각 배치마다 트랜잭션 처리
  - 실패한 레코드는 로그 기록, 계속 진행
  - 최종 통계 반환

**7-8**: **actor_type 추적** — MigrationPlan:
  - created_by_type: "user" 또는 "agent"
  - execution_count, last_executed_at으로 마이그레이션 실행 이력 추적
  - 마이그레이션 결과에도 actor_type 기록

**7-9**: **프리뷰 결과** — preview_migration()은 실제 데이터 변환 없이:
  - 각 샘플에 대해 변환 시뮬레이션
  - source_data, target_data, status, issues 반환
  - 전체 success/warning/error 통계 계산

**7-10**: **Alembic 마이그레이션** — 다음 테이블 생성:
  ```sql
  CREATE TABLE migration_plans (
    id UUID PRIMARY KEY,
    doc_type_id UUID NOT NULL FK,
    from_schema_version INT NOT NULL,
    to_schema_version INT NOT NULL,
    rules_json JSONB NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    created_by VARCHAR(255) NOT NULL,
    created_by_type VARCHAR(50) NOT NULL,
    execution_count INT DEFAULT 0,
    last_executed_at TIMESTAMP,
    notes TEXT
  );
  
  CREATE INDEX idx_doc_type_version ON migration_plans(
    doc_type_id, from_schema_version, to_schema_version
  );
  ```

---

**작업 예상 소요 시간**: 36-44시간  
**마지막 업데이트**: 2026-04-17
