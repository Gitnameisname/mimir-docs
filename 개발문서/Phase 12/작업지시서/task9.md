# Task 12-9. Admin DocumentType 관리 UI 확장

## 1. 작업 목적

Phase 7 Admin UI의 DocumentType 관리 화면을 확장하여 플러그인 설정(metadata schema, chunking, RAG, 검색)을 Admin이 편집할 수 있게 한다.

이 작업의 목표는 다음과 같다.

- DocumentType 플러그인 현황 조회 화면
- 타입별 설정 편집 화면 (chunking_config, rag_config, search_config)
- metadata schema 편집 화면 (JSON Schema 편집기)
- 새 DocumentType 생성 화면 (플러그인 등록)

---

## 2. 작업 범위

### 포함 범위

- DocumentType 목록 (플러그인 현황 포함)
- 타입별 설정 탭 편집 UI
- metadata schema 편집기 (JSON 기반)
- 새 타입 생성/편집 UI
- 설정 변경 감사 이벤트

### 제외 범위

- 코드 기반 플러그인 등록 (개발자 작업)
- 렌더링/편집기 플러그인 UI (복잡도 높아 별도)

---

## 3. 선행 조건

- Task 12-2~12-8 완료 — 플러그인 레지스트리 및 내장 타입 동작
- Phase 7 Admin UI DocumentType 관리 화면 구조 이해

---

## 4. 주요 구현 대상

### 4-1. DocumentType 목록 화면 확장

기존 Phase 7 목록에 플러그인 정보 컬럼 추가:

```
컬럼:
  - 타입명 (type_name)
  - 표시 이름
  - 등록 방식 (내장 플러그인 / DB 설정)
  - 문서 수
  - 벡터화 청크 수
  - 마지막 설정 변경 시각
  - 액션 (설정 편집 / 내장 타입: 오버라이드 편집)
```

---

### 4-2. 타입별 설정 편집 탭 UI

설정 편집 페이지 `/admin/document-types/{type_name}/settings`:

```
탭 구성:
  [기본 정보] [Metadata Schema] [청킹 설정] [검색 설정] [RAG 설정] [워크플로 설정]
```

#### 청킹 설정 탭

```
max_chunk_tokens:     [입력, 기본값 512]
min_chunk_tokens:     [입력, 기본값 50]
overlap_tokens:       [입력, 기본값 50]
include_parent_context: [체크박스]
parent_context_depth: [입력, 기본값 2]
index_version_policy: [선택: published_only / latest / all]
exclude_node_types:   [태그 입력]
```

#### RAG 설정 탭

```
max_context_tokens:   [입력, 기본값 6000]
top_n:                [입력, 기본값 5]
[시스템 프롬프트 템플릿 텍스트 편집기]
```

---

### 4-3. metadata schema 편집기

JSON Schema를 직접 편집하는 코드 편집기 (Monaco Editor 또는 CodeMirror):

```
편집기:
  - JSON Schema 입력 (좌측)
  - 실시간 미리보기 (우측, 렌더링된 폼)
  - 문법 검사 및 오류 표시
  - [저장] [초기화] 버튼
```

---

### 4-4. 새 DocumentType 생성

```
/admin/document-types/new

필드:
  - 타입명 (영문 대문자, 밑줄)
  - 표시 이름
  - 설명
  - 기반 타입 선택 (POLICY 설정을 복사해서 시작 등)

[생성] → DB에 저장 → ConfigurableDocumentTypePlugin으로 동작
```

---

### 4-5. Admin API 목록

| Method | Path | 설명 |
|--------|------|------|
| GET | /api/admin/document-types | 타입 목록 (플러그인 정보 포함) |
| GET | /api/admin/document-types/{type} | 타입 상세 (전체 설정) |
| PUT | /api/admin/document-types/{type}/config | 설정 업데이트 |
| POST | /api/admin/document-types | 새 타입 생성 |
| DELETE | /api/admin/document-types/{type} | 타입 삭제 (내장 타입 삭제 불가) |

---

### 4-6. 설정 변경 감사 이벤트

```python
AuditService.record("DOCUMENT_TYPE_CONFIG_CHANGED", {
    "type_name": type_name,
    "changed_fields": list(changed.keys()),
    "changed_by": user_id,
})
```

---

## 5. 산출물

1. DocumentType 목록 화면 확장 UI
2. 타입별 설정 탭 편집 UI
3. metadata schema 편집기
4. 새 타입 생성 UI
5. Admin API 구현 (상기 5개 엔드포인트)
6. 감사 이벤트 기록

---

## 6. 완료 기준

- Admin이 DocumentType 설정 탭에서 chunking/RAG/검색 설정을 편집할 수 있다
- metadata schema 편집기가 JSON Schema를 저장하고 검증한다
- 새 DocumentType을 생성하면 ConfigurableDocumentTypePlugin으로 동작한다
- 설정 변경 시 감사 이벤트가 기록된다
- UI 리뷰를 5회 이상 수행한다 (CLAUDE.md 규칙)

---

## 7. Codex 작업 지침

- CLAUDE.md의 UI 규칙에 따라 UI 리뷰를 최소 5회 이상 수행한다
- 내장 플러그인 타입(POLICY 등)은 삭제 불가 처리하고, 설정은 "오버라이드" 방식으로 편집한다
- metadata schema 편집기에서 잘못된 JSON Schema 저장 시 즉시 오류 표시하고 저장을 막는다
- 설정 변경 후 해당 타입의 청크를 자동 재벡터화할지 여부를 Admin에게 확인 모달로 묻는다
