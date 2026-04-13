# Task 12-3. 타입별 metadata schema 플러그인

## 1. 작업 목적

DocumentType별로 metadata 필드를 JSON Schema로 정의하고, 저장/편집 시 스키마 기반 검증이 동작하는 플러그인을 구현한다.

이 작업의 목표는 다음과 같다.

- MetadataSchemaPlugin 구현
- JSON Schema 기반 metadata 검증 로직
- Admin UI에서 스키마 편집 지원 (Task 12-9와 연계)
- 내장 타입의 metadata schema 정의 (POLICY, MANUAL, REPORT, FAQ)

---

## 2. 작업 범위

### 포함 범위

- MetadataSchemaPlugin 인터페이스 상세 구현
- JSON Schema 검증 로직
- 내장 타입 metadata schema 정의
- metadata 편집 UI 설정 (필드 타입, 라벨, 필수 여부)

### 제외 범위

- Admin UI 편집 화면 (Task 12-9에서 다룸)
- 내장 타입 플러그인 통합 (Task 12-8에서 다룸)

---

## 3. 선행 조건

- Task 12-2 (DocumentTypePlugin 인터페이스) 완료

---

## 4. 주요 구현 대상

### 4-1. MetadataSchemaPlugin 인터페이스

```python
class MetadataSchemaPlugin:
    def get_schema(self) -> dict:
        """JSON Schema (draft-07) 반환"""
        return {}

    def validate(self, metadata: dict) -> list[str]:
        """검증 오류 메시지 목록 반환 (빈 리스트 = 유효)"""
        from jsonschema import validate, ValidationError
        errors = []
        try:
            validate(instance=metadata, schema=self.get_schema())
        except ValidationError as e:
            errors.append(e.message)
        return errors

    def get_ui_schema(self) -> dict:
        """UI 렌더링 힌트 (필드 순서, 라벨, 플레이스홀더 등)"""
        return {}
```

---

### 4-2. 내장 타입 metadata schema 예시

#### POLICY (정책/규정)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "policy_number": {
      "type": "string",
      "title": "정책 번호",
      "description": "예: POL-2025-001"
    },
    "effective_date": {
      "type": "string",
      "format": "date",
      "title": "시행일"
    },
    "review_date": {
      "type": "string",
      "format": "date",
      "title": "검토 예정일"
    },
    "owner_department": {
      "type": "string",
      "title": "담당 부서"
    },
    "compliance_standards": {
      "type": "array",
      "items": {"type": "string"},
      "title": "준거 기준",
      "description": "예: ISO 27001, ISMS-P"
    }
  }
}
```

#### MANUAL (매뉴얼)

```json
{
  "type": "object",
  "properties": {
    "target_audience": {"type": "string", "title": "대상 독자"},
    "version_tag": {"type": "string", "title": "버전 태그"},
    "prerequisite": {"type": "string", "title": "선행 지식/작업"}
  }
}
```

#### FAQ

```json
{
  "type": "object",
  "properties": {
    "category": {"type": "string", "title": "FAQ 카테고리"},
    "tags": {"type": "array", "items": {"type": "string"}, "title": "태그"}
  }
}
```

---

### 4-3. UI Schema

Admin/사용자 편집기에서 metadata 필드 렌더링 힌트:

```json
{
  "ui:order": ["policy_number", "effective_date", "review_date", "owner_department"],
  "policy_number": {
    "ui:placeholder": "POL-2025-001",
    "ui:autofocus": true
  },
  "effective_date": {
    "ui:widget": "date"
  },
  "compliance_standards": {
    "ui:widget": "checkboxes",
    "ui:options": ["ISO 27001", "ISMS-P", "SOC 2", "GDPR"]
  }
}
```

---

### 4-4. 검증 통합

문서 저장/업데이트 API에서 metadata 검증 호출:

```python
def save_document(document_id, metadata, document_type):
    plugin = DocumentTypeRegistry.instance().get(document_type)
    errors = plugin.metadata_schema_plugin().validate(metadata)
    if errors:
        raise ValidationError(f"metadata 검증 실패: {errors}")
    # ... 저장 로직
```

---

### 4-5. 단위 테스트

| 시나리오 | 기대 결과 |
|---------|----------|
| 유효한 POLICY metadata | 빈 오류 목록 |
| 필수 필드 누락 | 오류 메시지 반환 |
| 잘못된 날짜 형식 | 오류 메시지 반환 |
| 스키마 없는 타입 | 모든 metadata 허용 |

---

## 5. 산출물

1. MetadataSchemaPlugin 구현
2. POLICY / MANUAL / REPORT / FAQ metadata schema 정의
3. UI Schema 정의
4. 문서 저장 API 검증 통합 코드
5. 단위 테스트

---

## 6. 완료 기준

- MetadataSchemaPlugin이 JSON Schema를 반환한다
- validate()가 잘못된 metadata에 대해 오류를 반환한다
- 내장 4개 타입의 metadata schema가 정의되어 있다
- 문서 저장 API에서 metadata 검증이 호출된다

---

## 7. Codex 작업 지침

- JSON Schema 검증 라이브러리는 `jsonschema` (Python) 또는 `ajv` (TypeScript/Node)를 사용한다
- metadata schema가 없는 타입은 모든 metadata를 허용한다 (검증 통과)
- UI Schema는 react-jsonschema-form 또는 유사 라이브러리 규격을 따른다
- 내장 타입의 metadata schema는 강제(required) 필드를 최소화하여 도입 장벽을 낮춘다
