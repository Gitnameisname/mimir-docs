# Disagreement Record — FG 4-3 작업지시서 vs Mimir 현실

**작성일**: 2026-04-28
**FG**: S3 Phase 4 / FG 4-3
**작성자**: Claude (Design+Implementation Actor)
**대상**: [task4-3.md](../개발문서/S3/phase4/작업지시서/task4-3.md) §2.1.1, §2.1.6, §2.1.8 (Alembic 마이그레이션 + REST citation save 강제)
**근거 조항**: CONSTITUTION 제34조 (Disagreement Protocol), §35조 (Decision Precedence)

---

## 1. 사실 (Facts)

`task4-3.md` 가 가정한 사항과 실제 코드베이스 사이에 다음 갭이 존재한다:

### 1.1 `citations` 테이블 부재

task4-3.md §2.1.1 은 다음 SQL 을 명시한다:

```sql
ALTER TABLE citations ADD COLUMN citation_basis VARCHAR(20);
```

그러나 Mimir 코드베이스에는 `citations` 테이블이 **존재하지 않는다**.

| 검증 | 결과 |
|---|---|
| `grep -rn "CREATE TABLE.*citations" backend/app/db/connection.py` | 매칭 0건 |
| `grep -rn "FROM citations\|INTO citations" backend/app` | 매칭 0건 |
| 모든 Alembic revisions 검사 (`grep "citations" backend/app/db/migrations/versions/*.py`) | 매칭 0건 |
| `expected_citations JSONB` (line 1343 of connection.py) | 별 테이블 (golden_set / evaluations) 의 컬럼이지 citations 테이블 아님 |

### 1.2 인용의 실제 저장 모델

Citation 은 다음 두 표면에 존재:

1. **Pydantic 모델** [`app.schemas.citation.Citation`](../../backend/app/schemas/citation.py) — 5-tuple 구조의 in-memory 표현
2. **JSONB 임베드** — 다른 테이블의 컬럼에 직렬화되어 저장 (예: `golden_set_items.expected_citations`, `evaluation_runs.citations` 등)
3. **검증의 정본** [`document_chunks.source_text`](../../backend/app/services/retrieval/citation_service.py) — 시간 후 재검증의 hash 계산 대상

따라서 Mimir 의 인용 모델은 **"행이 아닌 좌표 + 정본 검증"** 패턴이다. 작업지시서가 가정한 "citations 테이블에 행을 저장하고 후일 검증" 패턴과 다르다.

### 1.3 REST citation 저장 경로 부재

task4-3.md §2.1.6 은 "REST 에서 citation 저장 시 latest → vN resolve" 강제를 요구하지만, 단일 citation 저장 endpoint 자체가 존재하지 않는다. Citation 은 retrieval 시점에 계산되어 응답에 임베드되며, 영구 저장은 evaluation / golden_set 등 다른 도메인의 부산물이다.

---

## 2. 작업지시서의 의도 추정

작업지시서가 본질적으로 추구한 보안 가치는:

1. **시간 후 재검증 가능성** — 인용이 저장된 시점의 정본 (hash) 이 변하지 않았음을 확인 → ✅ 이미 `document_chunks.source_text` + Pydantic `Citation.content_hash` 로 충족
2. **`citation_basis` 명시** (node_content vs rendered_text) — 어떤 텍스트를 hash 대상으로 삼았는지 → ✅ Pydantic 스키마에 필드 추가로 충족
3. **R3 강제 (latest 단독 미저장)** — JSONB 임베드 시점에서도 동일하게 강제 → ✅ 검증 도구 진입점에서 `latest` 거부
4. **5중 검증** — exists / pinned / hash / quoted_text / span — verify_citation 도구의 강화 → ✅ 도구 함수 재작성으로 충족

따라서 **DDL 변경 없이도 작업지시서의 의도는 모두 달성 가능**하다.

---

## 3. 결정 (Decision)

### 3.1 적용

작업지시서를 **Mimir 의 실제 구조에 맞게 적응**하여 다음을 수행한다:

| task4-3 §  | 원본 의도 | 적응 |
|---|---|---|
| §2.1.1 Alembic | citations 테이블에 `citation_basis` 컬럼 추가 | **Skip** — 테이블 부재. Pydantic `Citation` 모델에 필드 추가로 충족 |
| §2.1.2 Citation 모델 | dataclass 에 `citation_basis` 추가 | **적용** — `app.schemas.citation.Citation` Pydantic 모델 |
| §2.1.3 verify_citation 입출력 스키마 | 신규 필드 (citation_basis / quoted_text / span_length / checks) | **적용** — `VerifyCitationRequest` / `VerifyCitationData` 갱신 |
| §2.1.4 verify_citation 함수 | 5중 검증 재작성 + R3 거부 | **적용** — `tool_verify_citation` 재작성 |
| §2.1.5 hash 분기 | node_content_hash vs render_hash | **적용** — citation_basis 분기 |
| §2.1.6 REST latest resolve | citation 저장 경로 강제 | **N/A** — 단일 저장 endpoint 부재. 본 FG 의 verify_citation 진입점 R3 거부로 동등 효과 (저장된 latest 가 후일 검증 시 거부됨) |
| §2.1.7 회귀 테스트 | 5중 + REST + ACL | **적용** (REST 부분은 verify 진입점 R3 거부로 흡수) |
| §2.1.8 마이그레이션 가이드 | Alembic 적용 절차 | **간단화** — DDL 없으므로 코드 배포 절차만. 가이드 자체는 작성하되 "Alembic 미적용" 명기 |

### 3.2 보고

- 본 Disagreement Record 를 [FG4-3_검수보고서.md](../개발문서/S3/phase4/산출물/FG4-3_검수보고서.md) §"Disagreement" 섹션에 인용
- Phase 4 개발계획서 §10 변경 이력에 흡수 결정 명기
- 검수 시 운영자 / Codex 가 본 결정을 검토 가능하도록 명시적 링크 유지

---

## 4. 우선순위 적용 (제35조)

| 출처 | 결론 |
|---|---|
| 헌법 (제45조 P1 게이트) | DDL 변경이 없으면 P1 게이트 (Alembic 적용 직전 승인) 자체 발동 안 함 — 적용 |
| 사용자 지시 ("진행해") | 본 FG 의 핵심 가치 (5중 검증 + R3) 를 적용한다는 의도와 일치 |
| ADR | 본 사안에 대한 별 ADR 없음 |
| 통과하는 테스트 | citations 테이블 부재 — 모든 회귀가 현 구조 기반. 본 결정이 깨는 회귀 0 |
| 에이전트 추천 (작업지시서) | 작업지시서가 가정한 테이블 부재 — 의도 충족하는 적응이 합당 |

→ 적응안 채택 (헌법·사용자 지시·테스트 우선).

---

## 5. 변경 이력

| 일자 | 변경 | 작성자 |
|---|---|---|
| 2026-04-28 | 초안 — citations 테이블 부재 확인 + 적응안 결정 | Claude |
