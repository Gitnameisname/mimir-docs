# Task 10-10. Admin 벡터화 상태 관리 UI (Phase 7 확장)

## 1. 작업 목적

Phase 7 Admin UI에 벡터화 상태 관리 섹션을 추가하여 관리자가 벡터화 현황을 모니터링하고 수동으로 제어할 수 있게 한다.

이 작업의 목표는 다음과 같다.

- 벡터화 현황 대시보드 구현 (문서별 청크 수, 모델, 마지막 색인 시각)
- 수동 재색인 UI 구현
- 임베딩 비용 모니터링 화면 구현
- 권한 정합성 검증 UI 구현

---

## 2. 작업 범위

### 포함 범위

- 벡터화 현황 대시보드 (Admin)
- 문서별 벡터화 상태 조회 화면
- 수동 재색인 트리거 UI
- 임베딩 비용 모니터링
- 권한 정합성 검증 화면

### 제외 범위

- 검색 UI 자체 (Phase 8에서 완성됨)
- Background Job 모니터링 UI (Phase 7에서 완성됨, 재사용)

---

## 3. 선행 조건

- Task 10-6 (벡터화 파이프라인) 완료 — 상태 조회 API 사용 가능
- Task 10-7 (재색인 정책) 완료 — 수동 재색인 API 사용 가능
- Task 10-8 (권한 메타데이터) 완료 — 권한 정합성 API 사용 가능
- Phase 7 Admin UI 구조 이해

---

## 4. 주요 구현 대상

### 4-1. 벡터화 현황 대시보드

Admin 메인 대시보드에 벡터화 요약 위젯 추가:

```
벡터화 현황 요약 카드
  - 총 벡터화 문서 수 / 전체 문서 수
  - 총 청크 수
  - 미색인 문서 수 (Published이지만 청크 없음)
  - 마지막 전체 색인 시각
```

---

### 4-2. 문서별 벡터화 상태 테이블

```
/admin/vectorization 페이지

컬럼:
  - 문서 제목
  - DocumentType
  - 청크 수 (is_current=true)
  - Stale 청크 수 (is_current=false)
  - 임베딩 모델
  - 마지막 색인 시각
  - 상태 (색인됨 / 미색인 / 진행 중)
  - 액션 (수동 재색인 버튼)

필터:
  - DocumentType
  - 상태 (색인됨 / 미색인 / 실패)
  - 검색 (문서 제목)

페이지네이션 지원
```

---

### 4-3. 수동 재색인 UI

문서 목록에서 개별 수동 재색인:

```
수동 재색인 버튼 클릭 → 확인 모달
  "이 문서를 다시 벡터화하시겠습니까?
   현재 청크가 교체됩니다."
  → 확인 → POST /api/admin/vectorization/reindex
  → Job ID 표시 → Background Job 상태 링크 제공
```

전체 재색인:

```
[전체 재색인] 버튼 → 경고 모달
  "모든 Published 문서를 다시 벡터화합니다.
   시간이 걸릴 수 있습니다."
  → 확인 → VECTORIZE_ALL Job 생성
```

---

### 4-4. 임베딩 비용 모니터링

```
/admin/vectorization/costs 페이지

기간별 비용 조회:
  - 일별 / 주별 / 월별 집계
  - 모델별 비용 분리
  - 총 토큰 수 / 예상 비용(USD)

테이블:
  - 날짜
  - 모델
  - 총 토큰 수
  - 예상 비용 (USD)
  - 문서 수
```

EmbeddingUsageLog 테이블 집계 쿼리 기반.

---

### 4-5. 권한 정합성 검증 UI

```
/admin/vectorization/permission-sync 페이지

현황:
  - Stale 권한 청크 수
  - 마지막 검증 시각

액션:
  [지금 동기화] 버튼 → POST /api/admin/vectorization/sync-permissions
  → 갱신된 청크 수 토스트 알림
```

---

### 4-6. API 목록 (Admin)

| Method | Path | 설명 |
|--------|------|------|
| GET | /api/admin/vectorization/summary | 대시보드 요약 |
| GET | /api/admin/vectorization/documents | 문서별 상태 목록 |
| GET | /api/admin/vectorization/status | 특정 문서 상태 |
| POST | /api/admin/vectorization/reindex | 수동 재색인 |
| GET | /api/admin/vectorization/costs | 비용 집계 |
| GET | /api/admin/vectorization/permission-sync-status | 권한 정합성 |
| POST | /api/admin/vectorization/sync-permissions | 권한 동기화 |

---

## 5. 산출물

1. 벡터화 현황 대시보드 UI
2. 문서별 벡터화 상태 테이블 UI
3. 수동 재색인 UI (개별 + 전체)
4. 임베딩 비용 모니터링 UI
5. 권한 정합성 검증 UI
6. Admin API 구현 (상기 7개 엔드포인트)

---

## 6. 완료 기준

- Admin 대시보드에서 전체 벡터화 현황을 확인할 수 있다
- 문서별 청크 수와 마지막 색인 시각을 확인할 수 있다
- 수동 재색인 버튼이 동작하고 Job ID를 반환한다
- 임베딩 비용이 일/주/월별로 집계되어 표시된다
- 권한 정합성 동기화 버튼이 동작한다
- 데스크탑 및 웹 브라우저에서 모두 정상 동작한다

---

## 7. Codex 작업 지침

- CLAUDE.md의 UI 규칙에 따라 UI 리뷰를 최소 5회 이상 수행한다
- 수동 재색인(전체)은 의도하지 않은 실행을 방지하기 위해 확인 모달을 반드시 표시한다
- 비용 모니터링 화면에서 예상 비용은 소수점 4자리까지 표시한다 ($0.0000)
- 벡터화 상태는 실시간 폴링 없이 수동 새로고침 방식으로 구현하여 서버 부하를 줄인다
