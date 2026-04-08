# Task 12-10. 플러그인 테스트 프레임워크 및 검증

## 1. 작업 목적

DocumentType 플러그인의 규격 준수 여부를 자동 검증하는 테스트 프레임워크를 구현하고, 전체 Phase 12 통합 검증을 수행한다.

이 작업의 목표는 다음과 같다.

- 플러그인 규격 준수 검증 테스트 스위트 구현
- 기존 하드코딩 타입 분기 코드 완전 제거 검증
- Phase 8/10/11 연계 동작 통합 테스트
- Phase 13 연계 포인트 정의

---

## 2. 작업 범위

### 포함 범위

- 플러그인 규격 검증 테스트 스위트 (PluginConformanceTest)
- 하드코딩 타입 분기 코드 스캔 도구
- Phase 8/10/11 통합 테스트
- Phase 13 연계 포인트 문서

### 제외 범위

- 운영/보안/배포 체계 (Phase 13에서 다룸)
- 신규 DocumentType 추가 (Phase 12 이후 독립 진행)

---

## 3. 선행 조건

- Task 12-2~12-9 모두 완료

---

## 4. 주요 구현 대상

### 4-1. PluginConformanceTest (플러그인 규격 검증)

모든 플러그인이 준수해야 할 최소 규격을 자동 검증:

```python
class PluginConformanceTest:
    """
    새 플러그인을 작성할 때 이 테스트 클래스를 상속하여
    규격 준수 여부를 자동 검증.
    """
    plugin: DocumentTypePlugin  # 서브클래스에서 설정

    def test_type_name_format(self):
        """type_name이 영문 대문자 + 밑줄 형식인지"""
        assert re.match(r'^[A-Z][A-Z0-9_]*$', self.plugin.get_type_name())

    def test_display_name_not_empty(self):
        assert len(self.plugin.get_display_name()) > 0

    def test_chunking_config_valid(self):
        config = self.plugin.chunking_plugin().get_config()
        assert config.max_chunk_tokens > 0
        assert config.min_chunk_tokens >= 0
        assert config.max_chunk_tokens > config.min_chunk_tokens

    def test_rag_prompt_template_returns(self):
        template = self.plugin.rag_plugin().get_prompt_template()
        assert template is not None

    def test_search_plugin_returns_dict(self):
        config = self.plugin.search_plugin().get_config()
        assert isinstance(config, dict)

    def test_metadata_schema_valid_json_schema(self):
        schema = self.plugin.metadata_schema_plugin().get_schema()
        if schema:
            # JSON Schema 유효성 검사
            jsonschema.Draft7Validator.check_schema(schema)


# 내장 타입 규격 검증 예시
class TestPOLICYPluginConformance(PluginConformanceTest):
    plugin = POLICYPlugin()
```

---

### 4-2. 하드코딩 타입 분기 코드 스캔

CI에서 실행하는 타입 분기 하드코딩 탐지 스크립트:

```python
# scripts/check_hardcoded_types.py

KNOWN_TYPES = ["POLICY", "MANUAL", "REPORT", "FAQ"]
EXCLUDED_PATHS = ["doc/", "tests/", "app/plugins/"]

def scan_hardcoded_types(src_path: str):
    issues = []
    for py_file in glob.glob(f"{src_path}/**/*.py", recursive=True):
        if any(py_file.startswith(ex) for ex in EXCLUDED_PATHS):
            continue
        with open(py_file) as f:
            content = f.read()
        for type_name in KNOWN_TYPES:
            # 플러그인 파일 외에서 타입명 문자열 비교 탐지
            if f'== "{type_name}"' in content or f"== '{type_name}'" in content:
                issues.append(f"{py_file}: 하드코딩된 타입 비교 발견: {type_name}")
    return issues
```

---

### 4-3. Phase 8/10/11 통합 테스트

플러그인 교체 후 기존 동작이 유지되는지 확인:

| 테스트 | 시나리오 | 기대 결과 |
|-------|---------|----------|
| Phase 10 | POLICY 문서 청킹 | max_tokens=512, 조항 컨텍스트 포함 |
| Phase 10 | FAQ 문서 청킹 | max_tokens=256, 컨텍스트 없음 |
| Phase 11 | POLICY RAG 질의 | 조항 원문 인용 프롬프트 적용 |
| Phase 8 | POLICY 검색 | policy_number 부스트 적용 |
| Phase 12 | 새 타입 DB 등록 | ConfigurablePlugin으로 동작 |

---

### 4-4. Phase 13 연계 포인트

Phase 13 운영/보안/테스트 체계에서 플러그인과 관련된 고려사항:

| 항목 | 내용 |
|------|------|
| 테스트 커버리지 | 모든 플러그인 서브클래스는 PluginConformanceTest 통과 필수 |
| CI 검사 | check_hardcoded_types.py를 PR 단계에서 자동 실행 |
| 플러그인 로드 검증 | 앱 시작 시 모든 내장 플러그인 등록 성공 여부 확인 |
| 설정 변경 감사 | Admin UI에서 설정 변경 시 감사 로그 보존 기간 설정 |
| 플러그인 배포 | 신규 내장 타입 추가는 코드 배포 필요 (DB 기반 타입은 무배포) |

---

### 4-5. 검증 보고서 항목

| 항목 | 내용 |
|------|------|
| 플러그인 규격 통과율 | 4개 내장 타입 × PluginConformanceTest |
| 하드코딩 타입 분기 잔존 여부 | scan_hardcoded_types 결과 |
| Phase 8/10/11 통합 테스트 통과 여부 | 변경 전후 동일 동작 확인 |
| Phase 13 준비 완료 | 연계 포인트 정의 완료 여부 |

---

## 5. 산출물

1. PluginConformanceTest 구현
2. 하드코딩 타입 분기 스캔 스크립트
3. Phase 8/10/11 통합 테스트
4. Phase 13 연계 포인트 문서
5. Phase 12 완료 검증 보고서

---

## 6. 완료 기준

- 내장 4개 타입이 PluginConformanceTest를 모두 통과한다
- 코드베이스에서 하드코딩된 타입 분기 코드가 없다
- Phase 8/10/11 통합 테스트가 통과한다
- Phase 13 연계 포인트가 문서화되어 있다

---

## 7. Codex 작업 지침

- PluginConformanceTest는 새 플러그인 개발자가 반드시 상속하여 사용하도록 문서화한다
- 하드코딩 스캔은 플러그인 파일과 테스트 파일을 제외한 src 경로에서만 실행한다
- Phase 12 완료 시점이 Mimir의 "타입 확장 가능 플랫폼" 전환점임을 완료 보고서에 명시한다
