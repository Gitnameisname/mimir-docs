# S2 Phase 7 검수 보고서

**검수 일자**: 2026-04-17  
**검수 범위**: S2 Phase 7 — AI 품질 평가 인프라 (task7-8 ~ task7-12)  
**검수 담당**: Claude Code (자동 검수)  
**검수 방법**: 소스 코드 정적 분석 + 단위 테스트 실행 + S2 원칙 준수 검토

---

## 1. 검수 범위 및 구성

| FG | 내용 | Task | 산출물 |
|----|------|------|--------|
| FG7.1 | CI Gate 인프라 | task7-8 | evaluation_thresholds.yaml, ci_gate.py, check_evaluation_thresholds.py, golden-set-evaluation.yml |
| FG7.2 | AI 품질 평가 보고서 생성 | task7-9 | ai_quality_evaluation.md.jinja2, generate_evaluation_report.py, evaluate_golden_set.py |
| FG7.3 | Carry-over: FTS 검색 | task7-10 | _CONVERSATIONS_TITLE_TSV_DDL, search_conversations() |
| FG7.3 | Carry-over: Batch Rollback UI | task7-11 | batch_rollback(), batch-rollback endpoint, Undo 토스트 UI |
| FG7.3 | Carry-over: 부하 테스트 | task7-12 | load_test_proposals.py, seed_load_test_data.py |

---

## 2. 단위 테스트 결과

| 테스트 파일 | 테스트 수 | 통과 | 실패 | 상태 |
|------------|----------|------|------|------|
| test_evaluation_thresholds.py | 26 | 26 | 0 | ✅ |
| test_evaluation_report_generation.py | 19 | 19 | 0 | ✅ |
| test_conversation_fts.py | 19 | 19 | 0 | ✅ |
| test_batch_rollback.py | 10 | 10 | 0 | ✅ |
| **합계** | **74** | **74** | **0** | **✅ 전체 통과** |

---

## 3. FG별 검수 결과

### FG7.1 — CI Gate 인프라 (task7-8)

#### 3.1.1 evaluation_thresholds.yaml

| 항목 | 결과 |
|------|------|
| 6개 메트릭 임계값 정의 | ✅ |
| 환경별(dev/staging/prod) 오버라이드 | ✅ |
| 폐쇄망 모드 fallback_base_scores | ✅ |
| skip_conditions (labels + file patterns) | ✅ |
| direction(higher_is_better/lower_is_better) 구분 | ✅ |

**판정**: ✅ 이상 없음

#### 3.1.2 ci_gate.py

| 항목 | 결과 | 비고 |
|------|------|------|
| ThresholdLoader 구현 | ✅ | |
| EvaluationThresholdChecker 구현 | ✅ | |
| 폐쇄망 모드 fallback_base_scores 적용 | ✅ (수정 완료) | 검수 중 발견 → 수정 |
| lower_is_better 방향 처리 | ✅ | |
| 환경별 임계값 오버라이드 | ✅ | |

**검수 중 발견 및 수정 (High)**:
- `get_thresholds()` 메서드가 `closed_network_mode=True`일 때 YAML의 `fallback_base_scores`를 활용하지 않고 일반 임계값을 반환하는 버그 발견 → S2 원칙 ⑦(폐쇄망 지원) 위반이므로 수정

**판정**: ✅ 수정 완료

#### 3.1.3 check_evaluation_thresholds.py

| 항목 | 결과 |
|------|------|
| flat / nested JSON 두 가지 형식 지원 | ✅ |
| exit 0=PASS / exit 1=FAIL | ✅ |
| 환경/폐쇄망 플래그 CLI 지원 | ✅ |
| --output JSON 결과 저장 | ✅ |

**판정**: ✅ 이상 없음

#### 3.1.4 golden-set-evaluation.yml

| 항목 | 결과 | 비고 |
|------|------|------|
| check-evaluation-requirements 선행 Job | ✅ | |
| 스킵 조건 처리 (labels, file patterns) | ✅ | |
| 평가 실패 시 폴백 처리 (`\|\| true`) | ⚠️ | 아래 참고 |
| require-evaluation-pass Job | ✅ | |
| PR 코멘트 자동 작성 | ✅ | |

**검수 의견 (Medium, 미수정)**:
- `evaluate_golden_set.py`가 실패해도 `|| true`로 워크플로우가 계속 진행되고 fallback 점수(모두 0.0/1.0)가 주입되어 최종적으로 FAIL 판정이 남. 이 동작은 실제 스테이징 서버 미구동 환경에서 워크플로우가 종료되지 않도록 의도된 것으로, 미수정 채택 (배포 환경에서는 fallback 제거 고려).

**판정**: ✅ 허용 가능

---

### FG7.2 — AI 품질 평가 보고서 생성 (task7-9)

#### 3.2.1 ai_quality_evaluation.md.jinja2

| 항목 | 결과 |
|------|------|
| 헤더 (골든셋/모델/환경/타임스탬프) | ✅ |
| 요약 테이블 (메트릭 × 임계값 × 상태 × 변화) | ✅ |
| 변화 지시자 (📈📉➡️) | ✅ |
| 통과/미통과 상세 섹션 | ✅ |
| 항목별 상세 (최대 10개) | ✅ |
| 인사이트 (강점/약점/권장) | ✅ |
| 이전 보고서 대비 추세 | ✅ |
| 폐쇄망 모드 경고 표시 | ✅ |
| Dataclass 속성 접근 (`.threshold` 방식) | ✅ |

**판정**: ✅ 이상 없음

#### 3.2.2 generate_evaluation_report.py

| 항목 | 결과 | 비고 |
|------|------|------|
| load_report_from_json (flat/nested) | ✅ | |
| flat float 메트릭 passed=False 버그 | ✅ (수정 완료) | 검수 중 발견 → 수정 |
| update_evaluation_metadata (최대 100건) | ✅ | |
| EvaluationReportGenerator.generate() | ✅ | |
| EvaluationReportGenerator.generate_metadata() | ✅ | |
| _report_to_metadata 직렬화 | ✅ | |

**검수 중 발견 및 수정 (Critical)**:
- `load_report_from_json()`에서 `{"metric": 0.85}` 형식 float 값을 `passed=False`로 고정하여 올바른 보고서도 전체 실패로 표시되는 버그 → `passed=True`로 수정 (passed 판정은 `check_evaluation_thresholds.py`의 역할)

**판정**: ✅ 수정 완료

#### 3.2.3 evaluate_golden_set.py

| 항목 | 결과 | 비고 |
|------|------|------|
| GoldenSetEvaluationClient 구현 | ✅ | |
| health_check / start_evaluation / wait_for_completion | ✅ | |
| overall_score=0.0 처리 버그 | ✅ (수정 완료) | 검수 중 발견 → 수정 |
| 폴백 메트릭 이름 불일치 | ✅ (수정 완료) | 검수 중 발견 → 수정 |
| --closed-network 플래그 전달 | ✅ | |

**검수 중 발견 및 수정 (High)**:
- `(overall or 0) >= 0.7` 표현식에서 `overall_score=0.0`이 falsy 처리되어 기본값 0으로 평가됨 → `if overall is not None else 0.0` 으로 수정
- 집계 루프에서 `citation_present_rate` / `hallucination_rate` 사용 → YAML 메트릭명과 불일치 → `citation_present` / `hallucination`으로 수정

**판정**: ✅ 수정 완료

---

### FG7.3 — Carry-over 항목

#### 3.3.1 FTS 검색 (task7-10) — conversation_repository.py + connection.py

| 항목 | 결과 |
|------|------|
| title_tsv tsvector 컬럼 DDL | ✅ |
| GIN 인덱스 (idx_conv_title_fts) | ✅ |
| 트리거 함수 (update_conv_title_tsv) | ✅ |
| IF NOT EXISTS 멱등성 보장 | ✅ |
| 기존 데이터 마이그레이션 UPDATE | ✅ |
| search_conversations() 구현 | ✅ |
| plainto_tsquery('simple', ...) 사용 | ✅ |
| @@ 연산자 + ts_rank 정렬 | ✅ |
| ILIKE 폴백 (title_tsv IS NULL) | ✅ |
| 빈 검색어 즉시 반환 | ✅ |
| S2 ⑥ scope_id ACL 필터 | ✅ |

**판정**: ✅ 이상 없음

#### 3.3.2 Batch Rollback UI (task7-11) — service + API + frontend

| 항목 | 결과 | 비고 |
|------|------|------|
| batch_rollback() 서비스 메서드 | ✅ | |
| S2 ⑤ actor_type 하드코딩 | ✅ (수정 완료) | 검수 중 발견 → 수정 |
| committed 버전 skip 처리 | ✅ | |
| POST /admin/proposals/batch-approve | ✅ | |
| POST /admin/proposals/batch-reject | ✅ | |
| POST /admin/proposals/batch-rollback | ✅ | |
| batch 에러 처리 (except pass) | ✅ (수정 완료) | 검수 중 발견 → 수정 |
| 배치 크기 제한 (최대 200) | ✅ (수정 완료) | 검수 중 발견 → 수정 |
| batchRollback() API 클라이언트 | ✅ | |
| 30초 카운트다운 Undo 토스트 | ✅ | |
| useEffect 클린업 (clearInterval) | ✅ | |

**검수 중 발견 및 수정**:
- (High) `batch_rollback()` `actor_type="user"` 하드코딩 → `actor_type` 파라미터로 변경 (S2 원칙 ⑤)
- (Critical) `batch_approve/reject` 엔드포인트에서 `except Exception: pass`로 모든 오류 무시, 부분 실패 정보 미반환 → 로깅 + `skipped_ids` 응답 추가
- (Medium) 배치 크기 제한 없음 → 최대 200건 제한 추가

**판정**: ✅ 수정 완료

#### 3.3.3 부하 테스트 (task7-12)

| 항목 | 결과 |
|------|------|
| Locust 스크립트 (ProposalUser + AdminUser) | ✅ |
| 429 Rate Limit 성공 처리 | ✅ |
| PASS/FAIL 판정 (P95≤2s, 에러율≤1%) | ✅ |
| exit code 0=PASS / 1=FAIL | ✅ |
| 프로덕션 실행 방지 가드 | ✅ |
| 시드 데이터 스크립트 (seed/cleanup) | ✅ |
| 스테이징 호스트 가드 | ✅ |
| 결과 리포트 템플릿 | ✅ |

**판정**: ✅ 이상 없음

---

## 4. S2 원칙 준수 검토

| 원칙 | 항목 | 결과 | 비고 |
|------|------|------|------|
| ⑤ actor_type 감사 로그 | batch_rollback actor_type 파라미터화 | ✅ (수정 완료) | |
| ⑥ scope_id ACL 필터 | search_conversations organization_id 필터 | ✅ | |
| ⑦ 폐쇄망 모드 | ci_gate.py fallback_base_scores 활성화 | ✅ (수정 완료) | |
| ⑦ 폐쇄망 모드 | evaluate_golden_set.py --closed-network 전달 | ✅ | |
| ⑦ 폐쇄망 모드 | evaluation_thresholds.yaml fallback_base_scores 정의 | ✅ | |

---

## 5. 검수 중 발견 및 수정 완료 이슈

| # | 심각도 | 파일 | 문제 | 수정 |
|---|--------|------|------|------|
| 1 | Critical | generate_evaluation_report.py:123 | flat float 메트릭을 항상 `passed=False`로 처리 | `passed=True`로 변경 |
| 2 | Critical | proposal_queue.py:344,380 | batch approve/reject에서 모든 예외 무시, 부분 실패 정보 미반환 | 로깅 추가 + `skipped_ids` 응답 추가 |
| 3 | High | ci_gate.py:88 | `closed_network_mode=True`일 때 fallback_base_scores 미적용 (S2 ⑦ 위반) | YAML fallback_base_scores 적용 |
| 4 | High | evaluate_golden_set.py:172 | `overall_score=0.0`이 falsy로 처리되어 잘못된 결과 판정 | `if overall is not None else 0.0` 수정 |
| 5 | High | evaluate_golden_set.py:161 | 집계 메트릭명 `citation_present_rate`/`hallucination_rate` — YAML 불일치 | `citation_present`/`hallucination`으로 수정 |
| 6 | High | agent_proposal_service.py:604 | `batch_rollback()` actor_type="user" 하드코딩 (S2 ⑤ 위반) | `actor_type` 파라미터 추가 |
| 7 | Medium | proposal_queue.py:326,362 | 배치 크기 제한 없음 | 최대 200건 제한 추가 |

---

## 6. 잔존 이슈 (미수정 — 이월)

| # | 심각도 | 파일 | 내용 | 이월 사유 |
|---|--------|------|------|----------|
| 1 | Medium | golden-set-evaluation.yml | 평가 실패 시 `\|\| true` 폴백 — 스테이징 미구동 환경 대응 목적 | 의도된 동작, 향후 스테이징 구축 시 재검토 |
| 2 | Medium | connection.py | 대규모 UPDATE 마이그레이션 배치 처리 미적용 | 운영 배포 시 DBA 검토 필요 |
| 3 | Low | AdminProposalsPage.tsx | Undo 토스트 "되돌리기" 버튼 aria-label 누락 | 접근성 개선 이슈, Phase 8에서 일괄 처리 |
| 4 | Low | test_evaluation_report_generation.py | 실제 YAML 파일 의존 테스트 (고립도 부족) | 기능상 문제 없음, 테스트 구조 개선 후 처리 |

---

## 7. 종합 판정

| FG | 완성도 | 검수 결과 |
|----|--------|----------|
| FG7.1 CI Gate 인프라 | 100% | ✅ PASS (수정 1건) |
| FG7.2 보고서 생성 | 100% | ✅ PASS (수정 3건) |
| FG7.3 FTS 검색 | 100% | ✅ PASS |
| FG7.3 Batch Rollback UI | 100% | ✅ PASS (수정 3건) |
| FG7.3 부하 테스트 | 100% | ✅ PASS |

**단위 테스트**: 74/74 통과 (100%)  
**발견 이슈**: Critical 2건, High 4건, Medium 1건 — **전원 수정 완료**  
**잔존 이슈**: 4건 (Low/Medium, 기능 영향 없음)

## ✅ Phase 7 검수 결과: **통과**

> Phase 8(보안 취약점 검사) 진행 가능.
