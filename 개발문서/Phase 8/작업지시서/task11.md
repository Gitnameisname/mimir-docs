# Task 8-11. Phase 10 벡터 검색 연계 포인트 설계

## 1. 작업 목적

Phase 10 벡터 검색 도입 시 현재 FTS 기반 검색 시스템을 최소한의 변경으로 확장하거나 대체할 수 있도록 연계 구조를 설계한다.

이 작업의 목표는 다음과 같다.

- 검색 서비스 레이어 추상화 구조를 검토하고 Phase 10 확장을 위해 보완
- FTS → 하이브리드 검색(FTS + pgvector) 전환 시 API 계약 유지 방안 확정
- 벡터 검색 결과와 FTS 결과를 통합하는 방식 사전 설계
- Phase 10에서 활용할 검색 인터페이스 Reserved 포인트 명시

---

## 2. 작업 범위

### 포함 범위

- SearchService Interface 검토 및 Phase 10 확장 포인트 보완
- API 계약 유지 전략 확정
- 하이브리드 검색 결과 통합(ranking fusion) 방식 사전 설계
- Phase 10에서 추가될 검색 기능 Reserved 설계
- 마이그레이션 전략 문서화

### 제외 범위

- pgvector 설치 및 embedding 생성 (Phase 10에서 다룸)
- 실제 벡터 검색 구현 (Phase 10에서 다룸)

---

## 3. 선행 조건

- Task 8-1 (검색 시스템 아키텍처 설계) 완료
- Task 8-3 (검색 API 설계) 완료
- Task 8-5 ~ 8-7 (검색 구현) 완료

---

## 4. 주요 설계 대상

### 4-1. SearchService Interface 검토 및 보완

Task 8-1에서 설계한 추상화 구조를 Phase 10 도입 관점에서 재검토한다.

#### 현재 구조 (Phase 8)

```
SearchService Interface
  └── FTSSearchProvider (구현체)
```

#### Phase 10 이후 구조

```
SearchService Interface
  ├── FTSSearchProvider (Phase 8 구현체, 유지)
  └── HybridSearchProvider (Phase 10 신규 구현체)
         ├── FTS 검색 결과
         └── 벡터 검색 결과
         → Reciprocal Rank Fusion으로 통합
```

#### 보완 포인트

- SearchService Interface에 벡터 검색 필요 시 추가될 메서드 시그니처 Reserved
- Provider 선택 로직 (Feature Flag 또는 설정 기반)
- SearchContext에 벡터 검색 파라미터 추가 가능 구조 (확장 가능 options 필드)

---

### 4-2. API 계약 유지 전략

Phase 10 도입 이후에도 현재 검색 API(`GET /search`, `GET /search/documents`, `GET /search/nodes`)의 계약이 변경되지 않아야 한다.

#### 유지 전략

| 항목 | Phase 8 | Phase 10 이후 |
|------|---------|---------------|
| 엔드포인트 경로 | 동일 유지 | 동일 유지 |
| 요청 파라미터 | 기본 파라미터 유지 | 선택적 벡터 파라미터 추가 (하위 호환) |
| 응답 스키마 | 기본 스키마 | 동일 유지 (벡터 점수 필드 추가 가능) |
| 권한 필터링 | 동일 | 동일 (벡터 검색에도 동일 적용) |

#### 하위 호환 파라미터 확장 예시

Phase 10에서 추가될 선택적 파라미터:
- `mode=hybrid` (FTS + 벡터 혼합 검색)
- `mode=semantic` (순수 벡터/의미 검색)
- `mode=keyword` (기존 FTS 검색, 기본값)

현재 `mode` 파라미터 없이 호출하면 기존 FTS 동작 유지.

---

### 4-3. 하이브리드 검색 결과 통합 방식 사전 설계

FTS 결과와 벡터 검색 결과를 통합하는 Reciprocal Rank Fusion(RRF) 방식:

#### RRF 원리

```
RRF_score(d) = Σ 1 / (k + rank_i(d))
  - k: 상수 (기본 60)
  - rank_i: 각 검색 방식에서의 순위
```

#### 통합 흐름 (Phase 10에서 구현할 방향)

1. FTS 검색 결과 (상위 N개) + 관련도 점수
2. 벡터 검색 결과 (상위 N개) + 유사도 점수
3. RRF 점수로 통합 랭킹 계산
4. 통합 랭킹 기준 상위 결과 반환

#### 권한 필터링 적용

- 통합 결과에서도 권한 필터링은 동일하게 적용
- 벡터 검색 결과도 반드시 권한 범위 내에서만 포함

---

### 4-4. Phase 10 Reserved 설계 포인트

Phase 10에서 활용할 수 있도록 현재 구조에서 예약할 항목:

#### SearchResult 응답에 Reserved 필드 추가

```json
{
  "id": "doc_abc123",
  "score": 0.91,
  "_search_meta": {
    "fts_score": 0.91,
    "vector_score": null,
    "search_mode": "keyword"
  }
}
```

`_search_meta`는 Phase 8에서는 `fts_score`만 채워지고 나머지는 null. Phase 10에서 채움.

#### 검색 인덱스 테이블 확장 예약

`documents` 테이블에 Phase 10에서 추가될 컬럼 예약:
- `embedding_vector vector(1536)` — pgvector 설치 후 추가 예정
- `embedding_updated_at timestamp` — 마지막 embedding 갱신 시각

현재는 컬럼 추가하지 않고 마이그레이션 계획만 문서화.

---

### 4-5. Phase 8 → Phase 10 마이그레이션 전략

현재 FTS 기반 검색에서 하이브리드 검색으로 전환하는 단계:

#### 전환 단계

1. **Phase 10 준비**: pgvector 확장 설치, embedding 컬럼 추가
2. **병행 운영**: FTS + 벡터 검색 결과 모두 생성, 내부 비교 (사용자에게는 FTS 결과만 노출)
3. **A/B 테스트**: 일부 사용자에게 하이브리드 검색 결과 제공
4. **전환**: 하이브리드 검색을 기본으로 변경
5. **FTS 유지**: FTS는 fallback으로 유지 (벡터 파이프라인 장애 시)

---

## 5. 산출물

1. SearchService Interface 검토 및 보완 설계서
2. API 계약 유지 전략 문서
3. 하이브리드 검색 결과 통합 방식 사전 설계서 (RRF 방식)
4. Phase 10 Reserved 포인트 정의서
5. Phase 8 → Phase 10 마이그레이션 전략 문서

---

## 6. 완료 기준

- SearchService Interface가 Phase 10 확장 가능 구조로 보완되어 있다
- 현재 API 계약이 Phase 10 이후에도 유지될 전략이 확정되어 있다
- 하이브리드 검색 결과 통합 방식(RRF)이 사전 설계되어 있다
- Phase 10에서 활용할 Reserved 포인트가 명시되어 있다
- Phase 8 → Phase 10 전환 단계가 문서화되어 있다

---

## 7. Codex 작업 지침

- 코드 작성 금지 (설계 중심)
- Phase 10에서 구현할 내용을 현재 단계에서 미리 구현하지 않는다 — Reserved 포인트 명시만 한다
- API 계약의 하위 호환성을 설계의 핵심 제약으로 유지한다
- 벡터 검색에서도 권한 필터링이 동일하게 적용되어야 함을 설계에서 명시한다
- FTS를 Phase 10 이후에도 fallback으로 유지하는 이유(벡터 파이프라인 장애 대응)를 문서에 명시한다
