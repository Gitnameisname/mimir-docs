# S3 Phase 6 — 운영 안전 (Rate Limit + Retention + Sanitize + Admin 격리)

**작성일**: 2026-04-27
**Phase 상태**: 대기 (Phase 4 1라운드 종결 후 진입 권장 — admin API 격리 작업이 Phase 4 contract drift 패턴 재사용)
**선행 조건**: Phase 3 종결 (annotations / notifications / document.viewed audit), Phase 4 1라운드 종결 (R5 contract drift 패턴 정착)
**후행 조건**: Phase 7 진입 또는 S3 2라운드 종결
**Handoff Level**: `extended` — Rate Limit (외부 I/O 흐름) + Retention (DB 데이터 폐기) + Admin API 격리 (보안 P0 영역)
**Approver**: `@최철균` (P1 — Meta-1, 제45조)

---

## 0. Phase 4 와의 관계

본 Phase 의 **FG 6-4 (admin API organization_id 격리)** 는 Phase 4 의 R5 (REST/MCP 1:1 복제 금지) 와 contract drift 테스트 패턴을 **재사용**한다:

- Phase 4 의 `tests/integration/test_mcp_rest_drift.py` (4 종) 패턴을 admin API 의 organization_id 격리 회귀에 그대로 적용
- 두 작업 모두 "REST 의 어떤 엔드포인트가 의도된 권한 게이트를 우회 가능한가" 라는 같은 형태의 검증

따라서 Phase 4 1라운드 종결 후 본 Phase 진입이 효율적. Phase 4 미종결 상태에서는 contract drift 패턴 부재로 admin 격리 작업이 자체 패턴 설계부터 해야 함 (비효율).

다른 FG (rate-limit / retention / sanitize) 는 Phase 4 와 독립.

---

## 1. Phase 개요

### 1.1 목적

S3 Phase 1~3 종결 시점의 4 가지 운영 잔존을 일괄 처리한다. 각각 단독으로는 작은 작업이지만 보안 / PII / 운영 안정성에 직결되어 별 라운드에 묶인다.

1. **Rate Limit 일괄 적용** — 신규 API (annotations / contributors / notifications) 와 기존 일부 API 에 per-user / per-IP rate-limit dependency 통일
2. **Retention 정책 + cron** — `audit_events.event_type='document.viewed'` 7일 / 해결된 annotation 90일 archive (PII 최소화)
3. **Content Sanitize** — annotation content / metadata 의 ANSI control / null byte / 비정상 유니코드 sanitize
4. **Admin API organization_id 격리 일괄 패치** — admin 이 다른 조직의 자원 (scope_profile / agent / user) 변경 가능한 잠재 위험을 전수 차단 (FG 3-2 §3 T5 + 다른 admin 엔드포인트)

### 1.2 절대 규칙

- **R-O1. Rate Limit 단일 진입점** — 모든 신규 rate-limit 은 같은 dependency (`@limiter.limit(...)`) 패턴. 각 라우터에 별도 구현 금지.
- **R-O2. Retention 은 archive-first** — 데이터 삭제 전 archive 테이블에 이동 (PII 시 mask 후). 직접 DELETE 금지. 운영자 수동 PURGE 단계 별도.
- **R-O3. Sanitize 는 read 시 적용, write 시 raw 보존** — 원본 본문은 DB 에 raw 저장 (사용자 의도 보존). 응답 직렬화 단계에서 sanitize. 단, 명백한 위험 문자 (null byte 등) 는 write reject.
- **R-O4. Admin 은 자기 조직만** — 모든 admin API 가 `actor.organization_id == resource.organization_id` 검증. SUPER_ADMIN 만 organization 횡단 허용 (감사 로그 의무).

### 1.3 선행 조건

- Phase 3 종결 (annotations / notifications / document.viewed audit emit)
- Phase 4 1라운드 종결 (contract drift 테스트 패턴) — 권장
- 기존 audit_events retention 정책 부재 확인

### 1.4 기대 결과

- annotations / contributors / notifications API 에 per-user rate-limit
- audit_events 의 view 이벤트 7일 후 자동 archive
- 해결된 annotation 90일 후 archive (cascade 답글)
- annotation content 의 ANSI / null byte 차단 + write reject 회귀
- admin API 전수 점검 후 organization_id 격리 적용

---

## 2. Feature Group (FG) 요약

본 Phase 는 **4개 FG (6-1 ~ 6-4)** 로 구성한다.

| FG | 제목 | 핵심 | 산출물 |
|----|------|------|--------|
| **6-1** | Rate Limit 일괄 적용 | annotations 생성 30/min / 조회 60/min / contributors 60/min / notifications polling 120/min. 기존 admin / search 패턴 재사용 (`backend/app/api/auth/rate_limit.py`) | task6-1 + 검수·보안 보고서 |
| **6-2** | Retention 정책 + cron | (a) audit_events `document.viewed` 7일 archive (별 테이블 또는 cold storage). (b) 해결된 annotation 90일 archive. cron 스케줄러 (`apscheduler` 또는 기존 cron_util 재사용) | task6-2 + 검수·보안 보고서 + 마이그레이션 가이드 |
| **6-3** | Content Sanitize | annotation content / mention payload 의 ANSI control / null byte / 비정상 유니코드 reject (write 시) + sanitize (read 응답 시) | task6-3 + 검수·보안 보고서 |
| **6-4** | Admin API organization_id 격리 일괄 패치 | scope_profiles / agents / users / api_keys / 기타 admin 엔드포인트 전수 점검 후 `actor.organization_id == resource.organization_id` 검증 추가. SUPER_ADMIN 횡단 허용 + audit emit. **Phase 4 contract drift 패턴 재사용** | task6-4 + 검수·보안 보고서 + admin 엔드포인트 점검표 |

---

## 3. 데이터 / 스키마 모델 (개요)

### 3.1 신규 테이블 (FG 6-2 retention)

- `audit_events_archive` — `audit_events` 와 동일 구조 + `archived_at TIMESTAMPTZ`. cron 이 7일 경과 viewer 이벤트를 INSERT INTO ... SELECT + DELETE 로 이동.
- `annotations_archive` — `annotations` 와 동일 구조 + `archived_at`. 90일 경과 + status='resolved' annotation 이동 (cascade 답글 포함).

### 3.2 변경 없음

- annotations / notifications / scope_profiles / contributors API 응답 schema 변경 없음 (sanitize 는 직렬화 단계에서만)
- documents / versions / nodes 변경 없음

### 3.3 신규 cron job

- `backend/app/jobs/retention_cron.py` 또는 기존 `backend/app/services/expiration_batch.py` 확장
- 스케줄: 일 1회 (off-peak 시간, 환경변수 `RETENTION_CRON_HOUR=2` 기본)
- 환경변수: `RETENTION_DOCUMENT_VIEWED_DAYS=7` / `RETENTION_RESOLVED_ANNOTATION_DAYS=90`

---

## 4. 산출물 규약

| 산출물 | FG 6-1 | FG 6-2 | FG 6-3 | FG 6-4 |
|--------|--------|--------|--------|--------|
| 작업지시서 (`task6-N.md`) | ✅ | ✅ | ✅ | ✅ |
| 검수보고서 (R-O1~O4 준수 확인) | ✅ | ✅ | ✅ | ✅ |
| 보안취약점검사보고서 | ✅ | ✅ | ✅ | ✅ |
| Admin 엔드포인트 점검표 | — | — | — | ✅ |
| 마이그레이션 가이드 (archive 테이블) | — | ✅ | — | — |
| UI 디자인 리뷰 | — | — | — | — (admin UI 변경 없음) |

---

## 5. 게이트 / 완료 기준

### 5.1 1라운드 완료 기준

- [ ] **R-O1 (Rate Limit 단일)** — 신규 4 라우터 (annotations / contributors / notifications / 그 외) 가 모두 `@limiter.limit` dependency 사용. 코드 중복 0
- [ ] **R-O2 (Archive-first)** — retention cron 이 archive INSERT 후 source DELETE. 직접 DELETE 0. 회귀 테스트
- [ ] **R-O3 (Sanitize 정책)** — annotation content 의 null byte 가 write 시 ApiValidationError. 응답 직렬화 시 ANSI control 제거. 회귀 ≥ 6 시나리오
- [ ] **R-O4 (Admin 격리)** — admin API 전수 점검표 제출 + 조직 횡단 시도가 차단됨을 입증하는 회귀 ≥ 8 시나리오 (각 admin 엔드포인트별)
- [ ] cron 스케줄 정상 동작 (테스트 환경에서 1일치 데이터 archive 회귀)
- [ ] retention 정책의 환경변수가 (7일 / 90일) default + override 가능
- [ ] SUPER_ADMIN 횡단이 audit_events 에 별 event_type 으로 emit
- [ ] Phase 1~5 모든 회귀 녹색

### 5.2 회귀 게이트

- [ ] pytest 베이스라인 유지 (운영자 macOS 실측치)
- [ ] node:test 베이스라인 유지
- [ ] tsc 0 error
- [ ] 신규 cron 의 dry-run 모드 (`RETENTION_DRY_RUN=1`) 가 archive / DELETE 안 함

### 5.3 1라운드 종결 정의

위 5.1 + 5.2 모두 통과 시 **S3 Phase 6 1라운드 공식 종결**.

---

## 6. 범위 밖 / 이월

### 6.1 본 Phase 1라운드 명시적 제외

- **외부 SIEM / Slack 알림 연동** (audit_events) — 본 Phase 는 in-DB retention 만. 외부 sink 는 별 ADR + S4
- **Cluster-wide rate-limit** (다중 워커 동기) — 본 Phase 는 워커별 in-process. cluster-wide 는 Phase 7 (Valkey)
- **GDPR / 한국 개인정보법 우편 응답 자동화** — 별 Compliance 작업
- **Annotation 의 Hard delete (PURGE)** — 본 Phase 는 archive 까지. 운영자 수동 PURGE 별 도구

### 6.2 보류 결정 (FG 진행 중 결정)

- archive 테이블이 별 schema 인지 cold storage (S3 / Glacier) 인지 → FG 6-2 Pre-flight (운영 인프라 가용성 확인)
- ANSI control 차단의 strict 도 (모든 ASCII < 0x20 차단 vs 일부 (\t / \n) 허용) → FG 6-3 결정
- SUPER_ADMIN 의 조직 횡단 시 추가 confirm UI 필요 여부 → FG 6-4 결정

---

## 7. 리스크

| # | 리스크 | 대응 |
|---|-------|-----|
| R-01 | retention cron 이 의도보다 많은 row 삭제 (off-by-one) | dry-run 모드 + 실 운영 전 7일 / 90일 카운트 출력. 운영자 승인 후 실행 |
| R-02 | Rate Limit 통일이 기존 워크플로 회귀 (예: 자동화 스크립트) | 기존 호출 빈도 측정 후 limit 결정. 실측 기반 (7일 트래픽 분석) |
| R-03 | sanitize 가 정상 본문을 변형 (한글 일부 깨짐) | 화이트리스트 기반 sanitize (BMP 외 char 만 거부). 한글 NFC / NFD 모두 보존 회귀 |
| R-04 | admin API 격리 후 운영자가 정상 작업도 차단됨 | 점검표 작성 후 운영자 검토. SUPER_ADMIN escape hatch 제공 (audit emit 의무) |
| R-05 | retention archive 테이블이 빠르게 거대화 | archive 자체에 sub-retention (1년 후 cold storage / 2년 후 PURGE) — 별 라운드 |
| R-06 | Phase 4 contract drift 패턴 미정착 상태에서 본 Phase 진입 | Phase 4 1라운드 종결을 권장 선행 (필수는 아님; 자체 패턴 설계 가능) |
| R-07 | Sanitize 가 prompt injection (Phase 4) 와 결이 다름 → 두 보안 레이어 충돌 | Phase 4 의 detected_risks 와 본 sanitize 는 별 layer (text byte vs LLM 명령 해석). 충돌 없음 |

---

## 8. 협업 / 라우팅

`AGENT_MODE.md` §3.3 (extended):
- Design: Claude
- Implementation: Claude or Codex
- Review: Codex
- Human Approval: @최철균 (retention 정책 + admin 격리는 P1)

---

## 9. 참조

- `docs/개발문서/S3/phase4/Phase 4 개발계획서.md` (R5 contract drift 패턴 — FG 6-4 가 재사용)
- `docs/개발문서/S3/phase3/산출물/FG3-2_검수보고서.md` §3 T5 (admin 다른 조직 수정 잠재 위험)
- `docs/개발문서/S3/phase3/산출물/FG3-3_보안취약점검사보고서.md` §4 (sanitize 잔존 항목)
- `backend/app/api/auth/rate_limit.py` (기존 rate-limit 패턴)
- `backend/app/services/expiration_batch.py` (기존 expiration 패턴)
- `backend/app/utils/html_sanitizer.py` (기존 sanitize)

---

## 10. 변경 이력

| 일자 | 변경 | 작성자 |
|------|------|-------|
| 2026-04-27 | 초안 — Phase 4 (MCP) 신설 후 재정렬. 기존 "Phase 5 (운영 안전)" 안을 Phase 6 으로 이동 | Claude |

---

*작성: 2026-04-27 | 운영 안전 — S3 Phase 6 킥오프*
