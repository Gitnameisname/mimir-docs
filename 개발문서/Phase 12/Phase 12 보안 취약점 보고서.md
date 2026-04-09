# Phase 12 보안 취약점 검사 보고서

**작성일**: 2026-04-09  
**검사 범위**: Phase 12 — 문서 유형 확장 프레임워크 (플러그인 시스템)  
**검사 방법**: 코드 정적 분석 (수동)  
**검사 대상 파일**:
- `backend/app/plugins/base.py`
- `backend/app/plugins/builtin/` (policy, manual, report, faq)
- `backend/app/api/v1/admin.py` (Phase 12 추가 엔드포인트)
- `backend/app/services/chunking_service.py`
- `backend/app/services/rag_service.py`
- `frontend/src/features/admin/document-types/`
- `scripts/check_hardcoded_types.py`

---

## 요약

| ID | 심각도 | 제목 | 상태 |
|----|--------|------|------|
| P12-SEC-01 | **HIGH** | DB 저장 `system_prompt`가 Python `.format()`에 무방비 전달 | ✅ 수정 완료 |
| P12-SEC-02 | **HIGH** | `chunking_config` 숫자 값 범위 검증 없음 | ✅ 수정 완료 |
| P12-SEC-03 | **MEDIUM** | JSON Schema 검증 예외 전문(全文)이 HTTP 응답에 노출 | ✅ 수정 완료 |
| P12-SEC-04 | **MEDIUM** | JSONB 병합 시 NULL 전파로 업데이트 무효화 가능 | ✅ 수정 완료 |
| P12-SEC-05 | **LOW** | 플러그인 엔드포인트에서 `type_code` 경로 파라미터 형식 미검증 | ✅ 수정 완료 |

취약점 **5건** 발견, 전건 수정 완료.

---

## 상세 내역

---

### P12-SEC-01 — DB 저장 `system_prompt`의 Python Format String 취약점

**심각도**: HIGH  
**위치**: `backend/app/plugins/base.py` — `ConfigurableRAGPlugin.get_prompt_template()`

#### 문제

Admin UI에서 RAG 설정의 `system_prompt` 필드를 DB에 저장하면, 이 값이 아무 처리 없이 Python `.format()`의 템플릿 문자열로 사용된다.

```python
# 취약 코드 (수정 전)
template._SYSTEM_TEMPLATE = custom_prompt + "\n\n<document_context>\n{context}\n</document_context>"
# ...
return self._SYSTEM_TEMPLATE.format(context=context)  # custom_prompt에 {}가 있으면 KeyError/AttributeError
```

**공격 시나리오**:
- 악의적(또는 실수한) 관리자가 `system_prompt`에 `{role}`, `{user_input}` 등 형식 지정자를 포함하면 `.format()` 호출 시 `KeyError` 발생 → RAG 기능 전체 중단 (해당 DocType에 대해)
- Python 형식 지정자를 통한 제한적 객체 속성 접근: `{context.__class__.__name__}` 등으로 내부 타입 정보 노출 가능

**영향**: 해당 DocumentType에 대한 RAG 질의 전체 불능(DoS), 내부 구조 정보 노출

#### 수정

```python
# 수정 후
safe_prompt = str(custom_prompt).replace("{", "{{").replace("}", "}}")
template._SYSTEM_TEMPLATE = safe_prompt + "\n\n<document_context>\n{context}\n</document_context>"
```

`custom_prompt`의 모든 `{`, `}` 문자를 `{{`, `}}`로 이스케이프하여 `.format()`이 오직 `{context}` 하나만 치환하도록 제한.

---

### P12-SEC-02 — `chunking_config` 숫자 값 범위 미검증

**심각도**: HIGH  
**위치**: `backend/app/api/v1/admin.py` — `PUT /admin/document-types/{type_code}/plugin`

#### 문제

`chunking_config`의 숫자 필드(`max_chunk_tokens`, `min_chunk_tokens`, `overlap_tokens`, `parent_context_depth`)를 서버에서 범위 검증 없이 DB에 저장한다.

```python
# 취약 코드 (수정 전)
if body.chunking_config is not None:
    plugin_config_update["chunking_config"] = body.chunking_config  # 검증 없음
```

**공격 시나리오**:
- `max_chunk_tokens=0` 또는 음수 저장 → `_split_by_tokens()` 내 무한루프 → CPU 고갈
- `max_chunk_tokens=99999999` → 벡터화 시 단일 청크에 수 GB 문자열 로드 시도 → OOM
- `overlap_tokens >= max_chunk_tokens` → 슬라이딩 윈도우가 무한히 같은 위치를 반복 → 무한루프

**영향**: 벡터화 파이프라인 전체 중단, 서버 메모리/CPU 고갈

#### 수정

```python
# 수정 후
def _validate_chunking_config(cfg: dict) -> None:
    """P12-SEC-02: chunking_config 숫자 범위 검증."""
    max_tokens = cfg.get("max_chunk_tokens")
    # ...
    if max_tokens is not None:
        if not isinstance(max_tokens, int) or max_tokens <= 0:
            errors.append("max_chunk_tokens는 1 이상의 정수여야 합니다.")
        elif max_tokens > 32768:
            errors.append("max_chunk_tokens는 32768 이하여야 합니다.")
    # overlap < max, max > min 등 상호 검증 포함
```

검증 규칙:
| 필드 | 최솟값 | 최댓값 | 추가 조건 |
|------|--------|--------|-----------|
| `max_chunk_tokens` | 1 | 32768 | `> min_chunk_tokens` |
| `min_chunk_tokens` | 0 | — | `< max_chunk_tokens` |
| `overlap_tokens` | 0 | — | `< max_chunk_tokens` |
| `parent_context_depth` | 0 | 10 | — |

---

### P12-SEC-03 — JSON Schema 검증 예외 전문 HTTP 응답 노출

**심각도**: MEDIUM  
**위치**: `backend/app/api/v1/admin.py` — `update_document_type_plugin_config()`

#### 문제

`metadata_schema` JSON Schema 검증 실패 시 라이브러리 예외 객체 전체를 HTTP 422 응답 `detail`에 포함한다.

```python
# 취약 코드 (수정 전)
except Exception as exc:
    raise HTTPException(
        status_code=422,
        detail=f"유효하지 않은 JSON Schema입니다: {exc}"  # 내부 구현 상세 노출
    )
```

**문제점**: `jsonschema` 라이브러리 예외 메시지에는 내부 파일 경로, 라이브러리 버전, Python 인터프리터 경로 등이 포함될 수 있어 공격자의 환경 정보 수집에 활용 가능.

#### 수정

```python
# 수정 후
except Exception:
    raise HTTPException(
        status_code=422,
        detail="유효하지 않은 JSON Schema (Draft-07) 형식입니다. 스키마 구조를 확인하세요."
    )
```

---

### P12-SEC-04 — JSONB 병합 시 NULL 전파로 업데이트 무효화

**심각도**: MEDIUM  
**위치**: `backend/app/api/v1/admin.py` — `update_document_type_plugin_config()`

#### 문제

PostgreSQL의 JSONB 병합 연산자 `||`는 좌변이 `NULL`이면 결과가 `NULL`이 된다. `plugin_config` 컬럼이 `NULL`인 레코드(레거시 데이터 또는 마이그레이션 누락)에 업데이트를 적용하면 플러그인 설정이 손실된다.

```sql
-- 취약 쿼리 (수정 전)
UPDATE document_types
SET plugin_config = plugin_config || %s::jsonb  -- plugin_config IS NULL → 결과 = NULL
WHERE type_code = %s
```

**영향**: 설정 저장 성공 응답을 받았음에도 실제 DB에는 반영되지 않는 무결성 문제. 추후 조회 시 기본값이 적용되어 관리자 설정이 무시됨.

#### 수정

```sql
-- 수정 후
UPDATE document_types
SET plugin_config = COALESCE(plugin_config, '{}') || %s::jsonb
WHERE type_code = %s
```

`COALESCE`로 `NULL`을 빈 JSONB 객체 `{}`로 대체하여 항상 병합이 성공하도록 보장.

---

### P12-SEC-05 — `type_code` 경로 파라미터 형식 미검증

**심각도**: LOW  
**위치**: `backend/app/api/v1/admin.py` — `GET/PUT /document-types/{type_code}/plugin`

#### 문제

플러그인 관련 엔드포인트에서 `type_code` 경로 파라미터를 형식 검증 없이 레지스트리 조회 및 DB 쿼리에 사용한다.

- SQL 인젝션: 파라미터화 쿼리(`%s`)를 사용하므로 직접 인젝션은 불가
- 그러나 `type_code=lowercase`, `type_code=A B C` 등 비정규 입력이 레지스트리와 DB 양쪽에 조회를 발생시킴
- 공격자가 비정규 type_code를 반복 제출하면 불필요한 DB 부하 발생

**영향**: 경미한 서비스 자원 낭비, 예상치 못한 응답 형태

#### 수정

```python
_TYPE_CODE_PATTERN = re.compile(r'^[A-Z][A-Z0-9_]*$')

def _validate_type_code_format(type_code: str) -> None:
    if not _TYPE_CODE_PATTERN.match(type_code):
        raise HTTPException(
            status_code=422,
            detail="type_code는 영문 대문자, 숫자, 밑줄만 허용됩니다."
        )
```

`GET /document-types/{type_code}/plugin`, `PUT /document-types/{type_code}/plugin` 양쪽 엔드포인트 진입 시 즉시 검증.

---

## 미수정 항목 (수용 가능 위험)

### RAG 시스템 프롬프트 관리자 권한 남용 가능성

**심각도**: ACCEPTED  
**이유**: `system_prompt`를 저장할 수 있는 주체는 `SUPER_ADMIN` 권한 보유자로 한정된다. `admin.write` 권한 자체가 조직의 최고 신뢰 레벨을 전제하므로, 정상적인 위협 모델 범위 내에서는 허용 가능한 수준. P12-SEC-01 수정으로 format string 오용은 이미 차단됨.

### 플러그인 설정 JSON 크기 제한 없음

**심각도**: ACCEPTED  
**이유**: FastAPI의 기본 request body 크기 제한(보통 1MB)이 1차 방어선 역할을 한다. 실제 플러그인 설정 JSON은 수백 바이트 수준으로, 실질적 위험은 낮음. 향후 `Content-Length` 헤더 기반 명시적 제한 추가를 권고.

---

## 검사 결과 요약

| 항목 | 결과 |
|------|------|
| SQL 인젝션 | **없음** (모든 쿼리 파라미터화) |
| XSS | **해당 없음** (백엔드 API, 프론트엔드는 React 자동 이스케이프) |
| 인증/인가 우회 | **없음** (`require_admin_access` + RBAC 일관 적용) |
| 하드코딩 타입 비교 | **없음** (`check_hardcoded_types.py` 스캔 통과) |
| 감사 로그 누락 | **없음** (플러그인 설정 변경 시 `audit_emitter.emit` 호출) |
| Format String 취약점 | ✅ P12-SEC-01 수정 완료 |
| 입력 범위 미검증 | ✅ P12-SEC-02 수정 완료 |
| 예외 정보 노출 | ✅ P12-SEC-03 수정 완료 |
| DB NULL 전파 | ✅ P12-SEC-04 수정 완료 |
| 경로 파라미터 미검증 | ✅ P12-SEC-05 수정 완료 |

**Phase 12 보안 검수: 통과** — 발견된 취약점 5건 전건 수정 완료, Phase 13 진행 가능.
