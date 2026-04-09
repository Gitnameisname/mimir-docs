# Phase 12 종결 보고서
## 문서 유형 확장 프레임워크 구축 (플러그인 시스템)

**종결 일자**: 2026-04-09  
**상태**: ✅ 완료

---

## 1. 단계 요약

Phase 12는 프로젝트 핵심 원칙인 **"문서 타입은 하드코딩 금지 — 구조는 generic + config 기반"** 을 코드 레벨에서 완전히 구현한 단계다. 기존 서비스 레이어 전역에 흩어져 있던 DocumentType 분기 로직을 `DocumentTypeRegistry` 싱글턴 레지스트리로 중앙화하고, 4개 내장 타입(POLICY, MANUAL, REPORT, FAQ)을 플러그인으로 전환했다.

새 DocumentType은 플러그인 파일 하나를 작성해 레지스트리에 등록하거나, Admin UI에서 DB 기반 설정으로 추가할 수 있다. 어느 경우든 ChunkingService, RAGService, SearchService 코드를 수정할 필요가 없다.

**플러그인 서브시스템 7개**: `MetadataSchemaPlugin` · `EditorPlugin` · `RendererPlugin` · `ChunkingPlugin` · `RAGPlugin` · `SearchPlugin` · `WorkflowPlugin`

---

## 2. 완료 기준 달성 현황

| 완료 기준 | 달성 여부 | 비고 |
|-----------|----------|------|
| 새 DocumentType 추가 시 플러그인 파일 1개만 작성하면 된다 | ✅ | `DocumentTypePlugin` 서브클래스 구현 후 레지스트리 등록 |
| 기존 POLICY · MANUAL · REPORT · FAQ가 플러그인으로 전환된다 | ✅ | `builtin/` 4개 플러그인, `register_builtin_plugins()` 부팅 시 자동 등록 |
| ChunkingService · RAGService · SearchService에 타입 하드코딩이 없다 | ✅ | `check_hardcoded_types.py` CI 스캔 통과 (0건) |
| Admin UI에서 플러그인 설정을 DB 오버라이드로 조정할 수 있다 | ✅ | `GET/PUT /admin/document-types/{type_code}/plugin`, 6탭 UI |
| 플러그인 규격 준수를 자동 검증하는 테스트가 있다 | ✅ | `PluginConformanceTest` 베이스 클래스, 4개 내장 타입 전부 통과 |

**전체 완료 기준 5/5 달성**

---

## 3. Task별 구현 결과

| Task | 내용 | 결과 |
|------|------|------|
| 12-1 | 플러그인 인터페이스 설계 | ✅ `app/plugins/base.py` — 7개 서브플러그인, `DocumentTypeRegistry`, `DefaultDocumentTypePlugin`, `ConfigurableDocumentTypePlugin`, `PromptTemplate` |
| 12-2 | POLICY 플러그인 구현 | ✅ 조항 단위 청킹(max=512), 조항 인용 RAG 프롬프트, `policy_number` 검색 부스트(5.0), 승인 필수 |
| 12-3 | MANUAL 플러그인 구현 | ✅ 절차 단위 청킹(max=400, min=30, overlap=30), 단계별 답변 RAG 프롬프트 |
| 12-4 | REPORT 플러그인 구현 | ✅ 보고서 섹션 청킹(max=600, overlap=100), 데이터 기반 답변 RAG 프롬프트 |
| 12-5 | FAQ 플러그인 구현 | ✅ Q&A 쌍 청킹(max=256, overlap=0), 승인 불필요, `faq_question/faq_answer` 검색 특화 |
| 12-6 | ChunkingService 마이그레이션 | ✅ `get_chunking_config_for_type()` → 레지스트리 경유, `should_index_version()` 플러그인 위임 |
| 12-7 | RAGService 마이그레이션 | ✅ `build_system_prompt()` → 타입별 `PromptTemplate`, `prepare_context()` → 타입별 `top_n` · `max_context_tokens` |
| 12-8 | Admin API 플러그인 엔드포인트 | ✅ `GET/PUT /plugin`, `GET /plugin/schema`, 내장 타입 목록 포함, 삭제 보호, 감사 로그 |
| 12-9 | Admin UI 플러그인 설정 탭 | ✅ 6탭(기본정보 · Metadata · 청킹 · RAG · 검색 · 워크플로), 내장 배지, 오버라이드 저장 |
| 12-10 | 플러그인 테스트 프레임워크 + CI 스캔 | ✅ `PluginConformanceTest` 11개 규격 테스트, `check_hardcoded_types.py` CI 스캔 스크립트 |

---

## 4. 신규/변경 파일 목록

### 4.1 Backend — 신규 파일

| 파일 | 역할 |
|------|------|
| `backend/app/plugins/__init__.py` | 패키지 init, 공개 인터페이스 재익스포트 |
| `backend/app/plugins/base.py` | 플러그인 코어 — 7개 서브플러그인 기본 구현, `DocumentTypeRegistry`, `DefaultDocumentTypePlugin`, `ConfigurableDocumentTypePlugin`, `PromptTemplate` |
| `backend/app/plugins/builtin/__init__.py` | 내장 플러그인 4개 패키지 + `register_builtin_plugins()` |
| `backend/app/plugins/builtin/policy.py` | `POLICYPlugin` — 조항 단위 청킹, 정책 인용 RAG, policy_number 검색 부스트 |
| `backend/app/plugins/builtin/manual.py` | `MANUALPlugin` — 절차 단위 청킹, 단계별 RAG |
| `backend/app/plugins/builtin/report.py` | `REPORTPlugin` — 섹션 청킹, 데이터 분석 RAG |
| `backend/app/plugins/builtin/faq.py` | `FAQPlugin` — Q&A 쌍 청킹, FAQ 특화 검색/RAG, 승인 불필요 |
| `backend/tests/test_plugins.py` | `PluginConformanceTest` 기반 규격 검증 테스트 스위트 |
| `scripts/check_hardcoded_types.py` | CI용 하드코딩 타입 비교 탐지 스크립트 (exit 1 시 빌드 실패) |

### 4.2 Backend — 변경 파일

| 파일 | 변경 내용 |
|------|----------|
| `backend/app/main.py` | `on_startup()`에 `register_builtin_plugins()` 추가 |
| `backend/app/services/chunking_service.py` | `ChunkingConfig` 필드 완성(3개 추가), `get_chunking_config_for_type()` 레지스트리 경유, `should_index_version()` 추가, `parent_context_depth` 실제 적용 |
| `backend/app/services/rag_service.py` | `build_system_prompt()` 타입별 `PromptTemplate` 사용, `prepare_context()` 타입별 `top_n` · `max_context_tokens` 선적용 |
| `backend/app/services/search_service.py` | `_get_search_boost_for_type()` 레지스트리 경유 |
| `backend/app/schemas/rag.py` | `RAGQueryRequest`에 `document_type: Optional[str]` 추가 |
| `backend/app/api/v1/rag.py` | `_stream_rag()` `document_type` 파라미터 전달 |
| `backend/app/api/v1/admin.py` | `list_document_types()` 내장 타입 포함, `get_document_type()` 내장 타입 폴백, `deactivate_document_type()` 내장 타입 보호, `GET/PUT /plugin`, `GET /plugin/schema` 신규, 입력 검증 함수 추가 |

### 4.3 Frontend — 변경 파일

| 파일 | 변경 내용 |
|------|----------|
| `frontend/src/types/admin.ts` | `AdminDocumentType` 확장(`is_builtin`, `plugin_registered`), `ChunkingPluginConfig` · `RAGPluginConfig` · `SearchPluginConfig` · `DocTypePluginStatus` 추가 |
| `frontend/src/lib/api/admin.ts` | `getDocumentTypePlugin()`, `updateDocumentTypePlugin()`, `getDocumentTypeMetadataSchema()` 추가 |
| `frontend/src/features/admin/document-types/AdminDocTypesPage.tsx` | 내장 뱃지, `plugin_registered` 컬럼 |
| `frontend/src/features/admin/document-types/AdminDocTypeDetailPage.tsx` | 6탭 UI 전면 재작성 — 기본정보, Metadata Schema, 청킹 설정, RAG 설정, 검색 설정, 워크플로 탭 |

---

## 5. 검수 및 보안 수정 이력

### 5.1 Phase 12 검수 지적 사항 수정 (5건)

| ID | 내용 | 수정 |
|----|------|------|
| 검수-1 | `prepare_context()`가 플러그인 `top_n` 무시 — Reranker가 항상 `settings.rag_top_n` 사용 | 플러그인 설정 선적용 구조로 리팩터링 (`top_n` · `max_context_tokens` Rerank 전 로드) |
| 검수-2 | `GET /admin/document-types/{type_code}` — 내장 타입 DB 레코드 없을 때 404 | 내장 타입 레지스트리 폴백 반환 추가 |
| 검수-3 | `parent_context_depth`가 청킹 로직에서 실제로 미적용 | `_visit()` 내 `parent_path[-depth:]` 슬라이싱 적용 |
| 검수-4 | `chunking_service.ChunkingConfig`에 `parent_context_depth` 등 3개 필드 누락 | 필드 추가, `_resolve_config()` 완성 |
| 검수-5 | `MetadataSchemaPlugin.validate()` 중첩 try-import 구조 불명확 | `jsonschema` import 선처리 후 `ValidationError` / `SchemaError` 분리 |

### 5.2 Phase 12 보안 취약점 수정 (5건)

상세 내용: [Phase 12 보안 취약점 보고서.md](Phase%2012%20보안%20취약점%20보고서.md)

| ID | 등급 | 내용 | 수정 |
|----|------|------|------|
| P12-SEC-01 | HIGH | DB 저장 `system_prompt` → Python `.format()` 직접 전달 (KeyError/정보노출) | `{`·`}` → `{{`·`}}` 이스케이프 후 `.format()` |
| P12-SEC-02 | HIGH | `chunking_config` 숫자 범위 미검증 (`max_chunk_tokens=0` → 무한루프) | `_validate_chunking_config()` 신규 추가 — 상한·하한·상호 조건 검증 |
| P12-SEC-03 | MEDIUM | JSON Schema 검증 예외 전문이 HTTP 422 응답에 노출 | 일반화된 고정 메시지로 교체, 상세는 서버 로그에만 유지 |
| P12-SEC-04 | MEDIUM | `plugin_config IS NULL` 행에 JSONB `||` 연산 → NULL 전파로 업데이트 무효 | `COALESCE(plugin_config, '{}') \|\| %s::jsonb` |
| P12-SEC-05 | LOW | 플러그인 엔드포인트에서 `type_code` 경로 파라미터 형식 미검증 | `_validate_type_code_format()` — `GET/PUT /plugin` 진입 시 즉시 검증 |

---

## 6. 주요 설계 결정 기록

| 결정 | 근거 |
|------|------|
| 새 DB 컬럼 추가 없이 기존 `plugin_config` JSONB 서브키 활용 | Phase 12를 마이그레이션 없이 배포 가능하도록 기존 컬럼 재사용 |
| 싱글턴 레지스트리 패턴 | 서비스 전 계층에서 일관된 단일 진입점 유지, 테스트에서 `reset()`으로 격리 |
| 폴백 우선순위: 코드 등록 → DB 오버라이드 → Default | 내장 플러그인의 안전성을 보장하면서 Admin UI에서 세부 조정 허용 |
| `PluginConformanceTest` 베이스 클래스 | 새 플러그인 작성 시 상속만으로 11개 규격 자동 검증 — 플러그인 품질 하한 보장 |
| `check_hardcoded_types.py` CI 스크립트 | 향후 개발에서 하드코딩 재발을 자동 탐지, exit 1로 빌드 실패 처리 |
| `jsonschema` 미설치 시 `validate()` 빈 목록 반환 (permissive) | 라이브러리 미설치 환경(개발/테스트)에서 서비스 중단 없이 동작 |
| `ConfigurableDocumentTypePlugin`의 `_base_plugin` 주입 | 내장 플러그인 기본값을 유지하면서 Admin UI 오버라이드를 계층적으로 병합 |

---

## 7. 미구현 항목

| 항목 | 내용 | 권고 시점 |
|------|------|-----------|
| `accessible_org_ids` 조직 단위 접근 제어 | Phase 10-M-2 잔존 항목. 플러그인 구조와 별개이며 Phase 13+ 처리 예정 | Phase 13+ |
| Cross-encoder Reranker 실제 모델 연동 | 현재 유사도 점수 정렬만 사용. `Reranker.rerank()` 시그니처 유지하며 내부만 교체 가능 | Phase 13+ |
| 플러그인 설정 JSON 크기 제한 명시 | 현재 FastAPI 기본 Body 크기 제한에 의존. `Content-Length` 기반 명시적 제한 추가 권고 | Phase 13+ |
| `jsonschema` 패키지 requirements 추가 | 현재 미설치 시 metadata 검증이 무음으로 건너뜀. 프로덕션 환경 필수 의존성 명시 필요 | 즉시 권고 |

---

## 8. 결론

Phase 12는 계획된 10개 Task를 모두 완료했다. 검수에서 5건, 보안 검사에서 HIGH 2건을 포함한 5건을 발견해 전부 수정했다.

CLAUDE.md의 핵심 원칙 **"문서 타입은 하드코딩 금지"** 가 코드 레벨에서 완전히 실현되었다. `check_hardcoded_types.py` 스캔 결과 `backend/app/` 전체에서 하드코딩된 타입 비교 코드 **0건** 확인. 새 DocumentType 추가에 필요한 코드 변경 범위가 플러그인 파일 1개로 제한된다.

> Phase 13 진행 가능 조건 충족 ✅
