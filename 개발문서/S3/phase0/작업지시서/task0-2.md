# task0-2 — `embedding_dim` config ↔ DB 스키마 자동 검증

**작성일**: 2026-04-22
**Phase / FG**: S3 Phase 0 / FG 0-2
**상태**: 착수 대기
**담당**: 백엔드
**예상 소요**: 1일

---

## 1. 작업 목적

S2 종결 직전 탐지된 BUG-04는 `app.core.config.EMBEDDING_DIM`(기본 768)과 `document_chunks.embedding VECTOR(<dim>)` 컬럼 차원이 불일치하면 인서트 시점까지 버그가 발견되지 않고, 벡터화 실패로 관찰되는 현상이었다. 본 FG는 불일치를 **Alembic revision 시점** 또는 **애플리케이션 기동 시점**에 단호하게 실패시킨다.

두 시점의 검증이 모두 필요한 이유:
- Alembic revision: 차원 변경을 포함한 revision이 부분 적용되어 코드/DB 미스매치가 발생하는 상황을 예방
- 애플리케이션 기동: 서로 다른 환경에서 revision은 같지만 `.env`의 `EMBEDDING_DIM`이 다르게 설정된 경우 (운영자 실수)

---

## 2. 작업 범위

### 2.1 포함

- Alembic revision 단계 검증 로직
- 애플리케이션 기동 시점 healthcheck subcheck
- pytest 유닛 테스트 (정합/불일치 케이스)
- 운영가이드 (차원 변경 절차)

### 2.2 제외

- 차원 변경의 자동 마이그레이션 (본 작업은 **검증**만; 차원 변경은 운영자 명시적 revision)
- 임베딩 재계산 배치 (차원 변경 시 재벡터화는 별도 배치; 본 FG 범위 밖)
- Milvus collection 차원 (현재 pgvector 중심; Milvus 사용 시 별도)

---

## 3. 선행 조건

- `backend/app/core/config.py`에 `EMBEDDING_DIM` 설정 존재 확인
- `backend/app/db/models/` 하위에 `document_chunks` 모델 존재 확인
- Alembic revision 생성 권한 (OWNER DB 유저 계정)
- FG 0-1이 병행 진행될 수 있음; 본 FG의 IT-07 시나리오는 FG 0-1 완료 후 활성화

---

## 4. 주요 작업 항목

### 4.1 Alembic revision 구현

신규 revision 파일: `backend/app/db/migrations/versions/s3_p0_embedding_dim_check.py`

동작:
1. 애플리케이션 설정값 `EMBEDDING_DIM` 로드 (기존 경로와 동일한 방식)
2. 현재 DB의 `document_chunks.embedding` 컬럼 타입에서 차원 추출 (PostgreSQL `pg_attribute` / `pg_type` 조회, 또는 `information_schema`)
3. 불일치 시 `op.execute(text("RAISE EXCEPTION ..."))` 또는 Python 레벨 `raise` 로 revision 실패
4. upgrade 함수에서만 검증 (downgrade는 no-op)

주의:
- 본 revision은 **스키마를 바꾸지 않는다**. 검증만 수행.
- revision 순서: Alembic revision 트리의 head 바로 다음. revision 이후 추가 revision이 생기면 `down_revision`만 업데이트.

### 4.2 Healthcheck subcheck

- `backend/app/api/v1/system.py` (또는 동등 경로)의 healthcheck에 `embedding_dim` 세부 정보 추가:
  ```json
  {
    "embedding_dim": {
      "config": 768,
      "db": 768,
      "match": true
    }
  }
  ```
- 전체 health status에 포함 (`match: false` 시 health → degraded)
- 응답은 S2 시스템 health 라우트 패턴 유지

### 4.3 pytest 유닛 테스트

`backend/tests/unit/test_embedding_dim_check.py` (또는 integration 가능):
- 정합 케이스: config `EMBEDDING_DIM=768`, DB 컬럼 `VECTOR(768)` — revision 성공
- 불일치 케이스: config 1536, DB 768 — revision 실패, 메시지 확인
- healthcheck 정합 케이스 응답 형태 확인
- healthcheck 불일치 시 `match: false` 반환 (monkeypatch로 설정값 조작)

테스트 수: ≥ 4건

### 4.4 운영가이드 작성

`docs/개발문서/S3/phase0/산출물/FG0-2_운영가이드.md`:
- 임베딩 차원을 변경해야 할 때의 절차 (5~7단계):
  1. `.env`의 `EMBEDDING_DIM` 변경 예정
  2. 새 차원의 migration revision 생성
  3. 재벡터화 배치 계획 수립 (범위 밖이지만 포인터 제공)
  4. 검증 revision이 통과함을 로컬에서 확인
  5. 배포 창에서 변경 적용
  6. `/api/v1/system/health`에서 `match: true` 확인
  7. 재벡터화 실행 (`scripts/reindex.py` — 가상 경로; 존재 여부에 따라 주석)

### 4.5 산출물

- `backend/app/db/migrations/versions/s3_p0_embedding_dim_check.py`
- Healthcheck 확장 (기존 system 라우트 수정)
- pytest 테스트 ≥ 4건
- `docs/개발문서/S3/phase0/산출물/FG0-2_검수보고서.md`
- `docs/개발문서/S3/phase0/산출물/FG0-2_보안취약점검사보고서.md`
- `docs/개발문서/S3/phase0/산출물/FG0-2_운영가이드.md`

---

## 5. 완료 기준

- [ ] 정합 케이스 Alembic revision 녹색
- [ ] 불일치 케이스 Alembic revision 실패 + 메시지 검증
- [ ] Healthcheck 응답에 `embedding_dim.match` 필드 존재
- [ ] pytest ≥ 4건 녹색
- [ ] FG 0-1 CI에 본 revision이 포함되어도 녹색 유지
- [ ] 검수보고서 · 보안보고서 · 운영가이드 제출

---

## 6. 리스크와 대처

| 리스크 | 대처 |
|--------|------|
| pgvector 차원 조회 쿼리의 PostgreSQL 버전 호환성 | pgvector 확장 기준 — `format_type(a.atttypid, a.atttypmod)` 문자열 파싱 + 안전한 fallback |
| downgrade 시 side effect | downgrade는 no-op으로 고정 |
| CI에서 revision 실행 실패가 전체 파이프라인을 막음 | FG 0-1 통합 테스트 중 IT-07은 정합 케이스만, 불일치 케이스는 유닛 테스트로 격리 |

---

## 7. 참조

- `docs/개발문서/S3/phase0/Phase 0 개발계획서.md` §2 FG 0-2
- `docs/개발문서/S2/S2 개발계획서 검수 패치안.md` — BUG-04 상세
- Alembic 경로: `/sessions/eloquent-practical-ramanujan/mnt/.auto-memory/reference_backend_alembic.md`

---

*작성: 2026-04-22 | 착수 대기*
