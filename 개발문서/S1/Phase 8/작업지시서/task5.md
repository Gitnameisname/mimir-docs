# Task 8-5. 전문 검색 (Full-Text Search) 구현

## 1. 작업 목적

PostgreSQL FTS를 활용하여 Document 및 Version 단위 전문 검색 기능을 실제로 구현한다.

이 작업의 목표는 다음과 같다.

- Task 8-2에서 설계한 tsvector 컬럼 및 GIN 인덱스를 실제 DB에 적용
- 검색 쿼리 빌더(tsquery 생성 로직) 구현
- 관련도(ts_rank) 계산 및 스니펫(ts_headline) 생성 구현
- Task 8-4에서 설계한 권한 필터링을 검색 쿼리에 통합
- 기존 데이터 일괄 인덱싱 실행

---

## 2. 작업 범위

### 포함 범위

- DB 마이그레이션: tsvector 컬럼 추가, GIN 인덱스 생성
- 인덱스 자동 갱신 트리거 구현
- 검색 서비스 레이어 구현 (FTSSearchProvider)
- 검색 쿼리 빌더 구현 (tsquery 생성)
- 관련도 계산 및 스니펫 생성 구현
- 권한 필터링 통합
- 기존 데이터 일괄 인덱싱 배치 실행

### 제외 범위

- 노드 단위 검색 구현 (Task 8-6에서 다룸)
- 검색 결과 UI (Task 8-8에서 다룸)
- 성능 튜닝 (Task 8-10에서 다룸)

---

## 3. 선행 조건

- Task 8-1 (검색 시스템 아키텍처 설계) 완료
- Task 8-2 (검색 인덱스 구조 설계) 완료
- Task 8-3 (검색 API 설계) 완료
- Task 8-4 (권한 기반 검색 필터링 설계) 완료

---

## 4. 주요 구현 대상

### 4-1. DB 마이그레이션

#### documents 테이블

- `search_vector tsvector` 컬럼 추가
- `idx_documents_search_vector GIN` 인덱스 생성
- `status` 컬럼 인덱스 확인 (권한 필터 성능)

#### versions 테이블

- `content_text text` 컬럼 추가 (본문 평문 추출 저장)
- `search_vector tsvector` 컬럼 추가
- `idx_versions_search_vector GIN` 인덱스 생성

#### 마이그레이션 순서

1. 컬럼 추가 (기본값 NULL)
2. GIN 인덱스 생성 (CONCURRENTLY 옵션으로 서비스 영향 최소화)
3. 트리거 함수 생성
4. 트리거 등록
5. 기존 데이터 일괄 인덱싱 (배치)

---

### 4-2. 인덱스 자동 갱신 트리거

#### documents 테이블 트리거

갱신 대상 이벤트: INSERT, UPDATE (title, metadata, status, type 변경 시)

구현 포인트:
- 변경된 컬럼이 검색 관련 컬럼인 경우에만 search_vector 재계산
- `setweight()` 적용하여 가중치 반영
- 문서가 삭제(논리 삭제 또는 DELETED 상태)된 경우 search_vector를 NULL로 설정

#### versions 테이블 트리거

갱신 대상 이벤트: INSERT, UPDATE (content 변경 시)

구현 포인트:
- content(JSON/트리 구조)에서 평문 텍스트 추출 함수 호출
- 추출된 텍스트로 search_vector 갱신
- Published 상태가 아닌 버전은 search_vector 설정 제어 (설정에 따라)

---

### 4-3. 본문 평문 추출 함수

Version의 content(트리 구조 JSON)에서 검색 가능한 평문 텍스트를 추출하는 함수:

구현 포인트:
- Node 트리를 재귀적으로 순회하여 텍스트 컨텐츠 추출
- 마크다운/HTML 마크업 제거
- 추출된 텍스트는 versions.content_text에 저장

---

### 4-4. 검색 쿼리 빌더 (tsquery 생성)

사용자 입력 검색어를 안전한 tsquery로 변환하는 로직:

#### 처리 단계

1. **입력 정제**: 특수문자 이스케이프, SQL 인젝션 방지
2. **토크나이징**: 공백 기준으로 단어 분리
3. **tsquery 생성**: 단어들을 AND(`&`) 또는 OR(`|`) 조합
4. **접두어 매칭**: 단어 끝에 `:*` 추가로 부분 매칭 지원 (선택)
5. **언어 설정**: `to_tsquery('simple', ...)` 적용

#### 검색어 처리 정책

| 입력 형식 | 처리 방식 |
|----------|----------|
| 단일 단어 `보안` | `보안:*` (접두어 매칭) |
| 다중 단어 `보안 정책` | `보안:* & 정책:*` (AND) |
| 따옴표 구문 `"보안 정책"` | `'보안 정책'` (구문 검색) |
| 특수문자 포함 | 이스케이프 처리 후 포함 |

---

### 4-5. 검색 서비스 구현 (FTSSearchProvider)

Task 8-1에서 설계한 SearchService Interface를 구현하는 FTSSearchProvider:

#### `search(query, filters, pagination, user_context)` 구현

주요 처리 흐름:
1. 검색어 → tsquery 변환
2. 권한 조건 생성 (user_context 기반)
3. 문서 상태 필터 적용
4. FTS 쿼리 실행 (documents + versions JOIN)
5. ts_rank로 관련도 계산
6. ts_headline로 스니펫 생성
7. SearchResult 스키마로 응답 직렬화

#### 관련도 계산 방식

```
ts_rank_cd(search_vector, query, normalization_option)
```

- `normalization_option = 1`: 문서 길이로 나눔 (긴 문서 페널티)
- 제목 매칭 가중치 A와 본문 가중치 B가 결합된 최종 점수

#### 스니펫 생성 설정

```
ts_headline(
  'simple',
  content_text,
  query,
  'MaxFragments=3, MaxWords=30, MinWords=10, StartSel=<mark>, StopSel=</mark>'
)
```

---

### 4-6. 기존 데이터 일괄 인덱싱 배치

기존에 적재된 Document/Version 데이터에 대한 일괄 인덱싱:

#### 배치 처리 방식

- 처리 단위: 100건 단위 배치
- 처리 순서: documents → versions
- 진행 상태 로깅: 처리 건수, 오류 건수, 소요 시간
- 오류 발생 시: 해당 건 스킵 후 로그 기록, 전체 중단하지 않음

#### Admin 연계

- 배치 실행 상태를 Phase 7 Admin 대시보드에서 조회 가능
- 완료 비율, 실패 건수 노출

---

## 5. 산출물

1. DB 마이그레이션 파일 (tsvector 컬럼, GIN 인덱스)
2. 인덱스 자동 갱신 트리거 구현
3. 검색 쿼리 빌더 구현 (tsquery 생성)
4. FTSSearchProvider 구현 (SearchService Interface 구현체)
5. 기존 데이터 일괄 인덱싱 배치 스크립트

---

## 6. 완료 기준

- documents, versions 테이블에 tsvector 컬럼과 GIN 인덱스가 적용되어 있다
- 문서 저장/수정 시 search_vector가 자동으로 갱신된다
- 검색 API에서 전문 검색이 동작하며 결과가 반환된다
- 검색 결과에 관련도 점수가 포함된다
- 검색 결과에 스니펫이 포함된다
- 권한 없는 문서가 검색 결과에 포함되지 않는다
- 기존 데이터 일괄 인덱싱이 완료되어 있다

---

## 7. Codex 작업 지침

- Task 8-1~8-4의 설계 내용을 기반으로 구현한다
- 검색 쿼리 빌더에서 SQL 인젝션 방지를 반드시 구현한다 (`to_tsquery` 파라미터 바인딩 사용)
- 스니펫의 `<mark>` 태그는 서버에서 생성되므로 XSS 안전성을 확인한다
- 권한 필터링은 검색 쿼리 내에서 강제 적용되며 우회 불가한 구조로 구현한다
- 트리거 함수는 검색 관련 컬럼 변경 시에만 동작하도록 최적화한다
