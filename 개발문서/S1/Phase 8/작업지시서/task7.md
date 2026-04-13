# Task 8-7. DocumentType 기반 검색 차별화 구현

## 1. 작업 목적

DocumentType에 따라 검색 가중치와 메타데이터 검색 방식을 차별화하여 타입별 최적 검색 경험을 제공한다.

이 작업의 목표는 다음과 같다.

- DocumentType별 검색 가중치 설정을 하드코딩 없이 설정 기반으로 관리
- DocumentType별 메타데이터 JSONB 필드 검색 지원
- 타입별 검색 필터 파라미터 지원

---

## 2. 작업 범위

### 포함 범위

- DocumentType 설정에 검색 관련 설정 항목 추가 (search_config)
- 타입별 메타데이터 검색 필드 설정 구조 구현
- 타입별 검색 가중치 적용 로직 구현
- DocumentType 필터 파라미터 처리 구현

### 제외 범위

- DocumentType 관리 UI (Phase 7에서 다룸)
- 신규 DocumentType 추가 프레임워크 (Phase 12에서 다룸)

---

## 3. 선행 조건

- Task 8-2 (검색 인덱스 구조 설계) 완료
- Task 8-5 (전문 검색 구현) 완료
- Phase 1: DocumentType 구조 이해

---

## 4. 주요 구현 대상

### 4-1. DocumentType 검색 설정 구조 (search_config)

DocumentType 설정에 검색 관련 항목을 추가한다.

#### search_config 스키마 정의

```json
{
  "search_config": {
    "searchable_metadata_fields": ["policy_number", "department", "summary"],
    "title_weight": "A",
    "metadata_weight": "B",
    "content_weight": "B",
    "boost_factor": 1.0,
    "exclude_from_search": false
  }
}
```

#### 설정 항목 설명

| 항목 | 타입 | 설명 |
|------|------|------|
| `searchable_metadata_fields` | string[] | 검색 대상으로 포함할 메타데이터 필드명 목록 |
| `title_weight` | A/B/C/D | 문서 제목의 FTS 가중치 (기본 A) |
| `metadata_weight` | A/B/C/D | 메타데이터 텍스트의 FTS 가중치 (기본 B) |
| `content_weight` | A/B/C/D | 본문 텍스트의 FTS 가중치 (기본 B) |
| `boost_factor` | float | 이 타입 문서의 관련도 점수 배율 (기본 1.0) |
| `exclude_from_search` | boolean | 이 타입 문서를 검색에서 완전 제외 (기본 false) |

#### 기본값 설정

search_config가 없는 DocumentType은 플랫폼 기본값 사용:

```json
{
  "searchable_metadata_fields": [],
  "title_weight": "A",
  "metadata_weight": "B",
  "content_weight": "B",
  "boost_factor": 1.0,
  "exclude_from_search": false
}
```

---

### 4-2. 메타데이터 검색 필드 추출

`searchable_metadata_fields` 설정에 따라 메타데이터 JSONB에서 검색 텍스트를 추출한다.

#### 추출 로직

```
extract_searchable_metadata_text(metadata JSONB, fields string[]) → text
  필드 목록 순회하여 metadata에서 값 추출
  배열이나 객체인 경우 문자열로 변환
  NULL인 경우 빈 문자열로 처리
  결과를 공백으로 연결하여 반환
```

#### 예시

POLICY 타입 문서의 메타데이터:
```json
{
  "policy_number": "SEC-2025-001",
  "department": "보안팀",
  "summary": "정보보안 기본 정책 요약",
  "effective_date": "2025-01-01",
  "internal_ref": "INTERNAL-XYZ"
}
```

`searchable_metadata_fields = ["policy_number", "department", "summary"]`인 경우:
추출 텍스트: `"SEC-2025-001 보안팀 정보보안 기본 정책 요약"`

---

### 4-3. tsvector 갱신 시 타입별 가중치 적용

문서 저장 시 search_vector 갱신 트리거에서 DocumentType 설정을 참조하여 가중치를 동적으로 적용한다.

#### 트리거 수정 포인트

- 트리거 함수 내에서 해당 문서의 DocumentType 설정 조회
- 설정의 weight 값을 `setweight()` 호출에 적용
- DocumentType 설정이 없는 경우 기본값 사용

#### 성능 고려

- 트리거 내에서 DocumentType 설정 조회 비용
- 자주 변경되지 않는 설정이므로 애플리케이션 레벨 캐싱 고려

---

### 4-4. exclude_from_search 처리

특정 DocumentType 문서를 검색에서 완전히 제외하는 경우:

- 해당 타입 문서의 search_vector를 NULL로 유지
- 검색 쿼리에 `exclude_from_search = false` 조건 추가
- Admin UI에서 이 설정을 변경하면 일괄 재인덱싱 필요 (Admin 연계)

---

### 4-5. DocumentType 필터 파라미터 처리

`GET /search/documents?type=POLICY&type=MANUAL` 파라미터 처리:

- 다중 타입 필터 지원 (OR 조건)
- 타입 필터가 없으면 모든 검색 가능 타입 대상
- `exclude_from_search = true`인 타입은 필터 없이도 자동 제외

---

### 4-6. 타입별 검색 결과 표시 힌트

검색 결과 응답에 DocumentType 기반 표시 힌트 포함:

```json
{
  "id": "doc_abc123",
  "type": "POLICY",
  "type_display_name": "정책 문서",
  "type_icon": "policy",
  "search_highlight_fields": ["policy_number", "department"]
}
```

UI에서 이 힌트를 활용하여 타입별로 다른 카드 형태로 결과를 표시 가능.

---

## 5. 산출물

1. DocumentType search_config 스키마 정의서
2. 메타데이터 검색 필드 추출 함수 구현
3. tsvector 갱신 트리거에 타입별 가중치 적용 구현
4. DocumentType 필터 파라미터 처리 구현

---

## 6. 완료 기준

- DocumentType 설정에 search_config 항목이 추가되어 있다
- 타입별 메타데이터 검색 필드 추출이 동작한다
- 타입별 검색 가중치가 search_vector 생성에 반영된다
- `type` 파라미터로 DocumentType 필터링이 동작한다
- `exclude_from_search = true` 타입 문서가 검색 결과에서 제외된다

---

## 7. Codex 작업 지침

- DocumentType 설정은 하드코딩 금지 — 반드시 DB 설정 기반으로 동적 처리
- CLAUDE.md의 "문서 타입은 하드코딩 금지" 원칙을 검색 가중치 설정에서도 철저히 준수한다
- 메타데이터 추출 시 JSONB 접근 오류(필드 미존재, 타입 불일치)에 안전하게 처리한다
- DocumentType 설정 변경 시 영향 받는 문서들에 대한 재인덱싱 트리거 포인트를 구현한다
