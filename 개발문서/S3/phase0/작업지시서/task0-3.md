# task0-3 — 테스트 커버리지 35.44% → 80% 복구

**작성일**: 2026-04-22
**Phase / FG**: S3 Phase 0 / FG 0-3
**상태**: 착수 대기
**담당**: 백엔드
**예상 소요**: 3~5일

---

## 1. 작업 목적

S2 종결 시점 백엔드 커버리지는 35.44%이다. 이 수치는 "테스트는 있지만, 실제로 프로덕션 경로의 대부분이 테스트되지 않는다"는 뜻이다. Phase 1 이후 기능 추가가 회귀를 일으킬 때 이를 **탐지 가능**하게 만들기 위해 본 FG에서 **서비스/리포지토리 계층 기준 ≥ 80%** 수준으로 회복시킨다.

라우터 계층은 통합 테스트(FG 0-1)로 상당 부분 커버되므로, 유닛 테스트 확보의 우선순위는 **서비스 로직 → 리포지토리 → 라우터 핵심 분기** 순서다.

---

## 2. 작업 범위

### 2.1 포함

- 현재 커버리지 베이스라인 측정 및 모듈별 리포트
- 서비스 계층 (`app/services/`) 신규 유닛 테스트
- 리포지토리 계층 (`app/repositories/`) 신규 유닛 테스트
- 라우터 계층의 핵심 분기 (에러 처리, ACL 우회 시도 등) 유닛 테스트
- CI에서 커버리지 thresholds 체크
- 커버리지 HTML artifact 업로드
- 검수보고서 · 보안취약점검사보고서

### 2.2 제외

- UI(프런트엔드) 테스트 커버리지 (본 FG는 백엔드 한정)
- 테스트 설계 철학의 전면 개편 (pytest fixture 개편은 필요 최소만)
- 스냅샷/골든 테스트 도입 (필요 시 FG 0-4와 겹치지 않게)

---

## 3. 선행 조건

- `pytest-cov` 또는 `coverage` 설치 (이미 존재 추정)
- `.coveragerc` 또는 `pyproject.toml`의 `[tool.coverage]` 섹션 경로 확인
- FG 0-1 기반 통합 테스트가 커버리지에 합산되도록 연결 (`pytest --cov=app backend/tests/`)

---

## 4. 주요 작업 항목

### 4.1 베이스라인 측정

1. 로컬에서 현재 커버리지 리포트 생성:
   ```
   cd backend
   pytest --cov=app --cov-report=html --cov-report=term-missing
   ```
2. `coverage.json` 또는 HTML 리포트에서 모듈별 커버리지 추출
3. `docs/개발문서/S3/phase0/산출물/FG0-3_베이스라인.md`에 모듈별 % + missing line 수 기록

### 4.2 우선순위 식별

`FG0-3_베이스라인.md` 기반으로 다음 기준 적용:
- 커버리지 < 50% 인 서비스 모듈을 최우선
- 과거 BUG-01~05 연관 모듈을 최우선에 포함
- 라우터는 통합 테스트에 맡기고, 서비스/리포지토리에 집중

권장 우선순위 예시:
1. `app/services/workflow_service.py` (S2에서 BUG-01 연관)
2. `app/services/vectorization_service.py` (BUG-04 연관)
3. `app/services/rag_service.py` (AI 품질의 핵심)
4. `app/services/document_service.py`
5. `app/services/scope_service.py` (S2 ⑥)
6. `app/repositories/*` 전반

실제 모듈명은 기존 트리 탐색 후 확정. 위는 예시 관점.

### 4.3 테스트 작성 패턴

- **서비스 유닛 테스트**: DB는 mock 또는 실제 SQLAlchemy session (FG 0-1의 픽스처 재활용 가능)
- **리포지토리 유닛 테스트**: 반드시 실 DB(FG 0-1 픽스처). SQL 분기 포함 시 mock 금지
- **라우터 분기**: 에러 경로(401/403/422/429/500) 중심
- 각 테스트는 명확한 `arrange / act / assert` 3섹션 주석
- flaky 금지: 시간 기반 assertion은 freeze_gun 또는 동결된 fixture 사용

### 4.4 CI 통합

`backend-integration.yml` (또는 단일 워크플로)에 다음 단계 추가:
- `pytest --cov=app --cov-report=xml --cov-report=html backend/tests/`
- `coverage xml` → `coverage.xml` artifact 업로드
- `coverage html` → `htmlcov/` artifact 업로드
- thresholds 체크:
  - 전체 ≥ 75%
  - `app/services/` ≥ 80%
  - `app/repositories/` ≥ 80%
- thresholds 미달 시 CI 실패

### 4.5 `.coveragerc` / `pyproject.toml` 업데이트

- `exclude_lines`: `pragma: no cover`, `if TYPE_CHECKING:`, `raise NotImplementedError`, `__repr__`
- `omit`: `app/db/migrations/*`, `app/*/init_data.py`, `scripts/*`
- `fail_under`: 75 (전체)
- `precision: 2`

### 4.6 산출물

- `docs/개발문서/S3/phase0/산출물/FG0-3_베이스라인.md`
- 신규 유닛 테스트 파일 (수십 건 예상)
- `.coveragerc` 또는 `pyproject.toml` 변경
- CI 워크플로 변경
- `docs/개발문서/S3/phase0/산출물/FG0-3_검수보고서.md`
- `docs/개발문서/S3/phase0/산출물/FG0-3_보안취약점검사보고서.md`
- HTML 커버리지 리포트 (artifact 또는 스크린샷으로 검수보고서에 첨부)

---

## 5. 완료 기준

- [ ] 전체 백엔드 커버리지 ≥ 75%
- [ ] `app/services/` 커버리지 ≥ 80%
- [ ] `app/repositories/` 커버리지 ≥ 80%
- [ ] CI에서 thresholds 미달 시 실패함을 확인 (일부러 한 줄 주석 처리 후 실패 재현 — 스크린샷 첨부)
- [ ] 검수보고서 · 보안보고서 제출

---

## 6. 리스크와 대처

| 리스크 | 대처 |
|--------|------|
| 커버리지 복구 과정에서 의미 없는 "행 통과용" 테스트 양산 | 테스트 리뷰 체크리스트 적용 (assertion 존재 + 분기 의미 확인). 검수보고서에 무효 테스트 제거 기록 |
| 테스트 실행 시간 증가 | pytest-xdist 병렬 실행 고려. CI 소요 목표 ≤ 15분 |
| 리포지토리 테스트의 실 DB 의존으로 로컬 개발 친화성 저하 | FG 0-1의 testcontainers 픽스처 재활용 |
| Flaky 테스트로 인한 CI 신뢰 저하 | flaky 탐지 시 즉시 격리 후 재설계. `@pytest.mark.flaky` 임시 사용 금지 |

---

## 7. 참조

- `docs/개발문서/S3/phase0/Phase 0 개발계획서.md` §2 FG 0-3
- `docs/개발문서/S3/phase0/작업지시서/task0-1.md` — 픽스처 재사용
- `docs/개발문서/S2/S2 개발계획서 검수 패치안.md` — 커버리지 35.44% 근거
- `CLAUDE.md` S1 ⑤

---

*작성: 2026-04-22 | 착수 대기*
