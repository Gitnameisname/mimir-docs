# task4-4 — REST/MCP Contract Drift 테스트 (FG 4-4)

**작성일**: 2026-04-27
**Phase / FG**: S3 Phase 4 / FG 4-4
**상태**: 착수 대기 (FG 4-0 ~ 4-3 완료 후)
**담당**: 백엔드 (설계 Claude / 구현 Codex / 리뷰 Claude — extended)
**예상 소요**: 1.5~2일
**선행 산출물**:
- `task4-0` 완료 (manifest 표준 + L4 차단)
- `task4-1` 완료 (envelope + mimir:// + latest resolve)
- `task4-2` 완료 (신규 도구 3종)
- `task4-3` 완료 (verify_citation 강화 + citation_basis)
**후행**: 본 FG 종결 = Phase 4 1라운드 종결 후보 (5.1 게이트와 함께)

---

## 1. 작업 목적

REST와 MCP가 동일 도메인 코어를 공유한다는 전제(R5)가 시간이 지나도 **자동으로 보호**되도록 한다. PR 한 건이 어느 한쪽만 수정해 권한·결과·등급화에서 drift가 발생하는 패턴을 CI 게이트로 차단한다.

업로드 요약 §8의 4종 최소 테스트를 구현하고, 그 중 R1·R3 직결 테스트는 **Phase 4 1라운드 종결 게이트**로 등록한다.

---

## 2. 작업 범위

### 2.1 포함 — 4종 Contract Drift 테스트

#### 2.1.1 테스트 1: 동일 노드 결과 일치성

**목적**: MCP `fetch_node`와 REST의 노드 조회가 같은 콘텐츠와 ACL을 적용함을 보장.

**위치**: `backend/tests/integration/contract_drift/test_same_node_result.py`

**구현 개요**:
1. fixture: 다양한 (Scope Profile × document_type × node_kind) 조합으로 시드 데이터 N≥30
2. 각 노드에 대해:
   - REST: `GET /documents/{id}/versions/{version_id}/nodes/{node_id}` (또는 동등 경로)
   - MCP: `tool_fetch_node(document_id, version_id, node_id)`
3. 비교:
   - **canonical content** (본문, content_hash) — **반드시 일치**
   - **ACL 판단** (같은 actor가 둘 다 접근 가능 / 둘 다 거부) — **반드시 일치**
   - **projection** (응답에 어떤 필드를 포함하는지) — 차이 허용. 예: REST는 audit metadata 포함, MCP는 envelope 포함
4. 시나리오:
   - 동일 actor + 정상 노드 → 둘 다 콘텐츠 동일
   - 다른 scope의 actor → REST 404 + MCP not_found (동일 거부)
   - draft 버전 → REST/MCP 모두 published 외 차단 (또는 둘 다 허용 — 정책 일치)

**실패 시 표기**: drift 발생 — 본문 hash 불일치 또는 ACL 차이.

#### 2.1.2 테스트 2: latest 해석 동일성

**목적**: `latest` 입력 시 REST와 MCP가 **같은 vN**을 반환함을 보장. 권한에 따라 보이는 latest도 일치.

**위치**: `backend/tests/integration/contract_drift/test_latest_resolution.py`

**구현 개요**:
1. fixture: 한 문서에 v1~v5의 published version + 일부 draft. 사용자별 published 가시성 다름 (Scope Profile)
2. 같은 actor에 대해:
   - REST: `GET /documents/{id}?version=latest` 또는 동등
   - MCP: `tool_fetch_node` 또는 `read_document_render`에 `version_id="latest"`
3. 응답에 포함된 resolved version (REST는 `version_id` 필드, MCP는 envelope의 `source.uri`)을 비교 — **반드시 일치**
4. 시나리오:
   - 일반 사용자 → 둘 다 v5
   - draft만 본 사용자 → 둘 다 동일 published 결과 (정책에 따라 v5 또는 v3 등)
   - 권한 없는 사용자 → 둘 다 거부

**실패 시 표기**: REST와 MCP가 다른 vN을 반환하면 drift.

#### 2.1.3 테스트 3: 비노출 도구 검증 (R1 핵심)

**목적**: manifest의 `status: not_exposed` 또는 `maturity: forbidden` 도구가 실제로 MCP에 노출되지 **않음**을 보장.

**위치**: `backend/tests/integration/contract_drift/test_not_exposed_tools.py`

**구현 개요**:
1. `TOOL_SCHEMAS`를 순회 — `status="not_exposed"` 또는 `maturity="forbidden"` 항목 추출
2. 각 항목에 대해:
   - `POST /mcp/tools/list` 응답에 해당 도구 이름이 **없음**을 확인
   - `POST /mcp/tools/call` 호출 시 표준 거부 에러 (METHOD_NOT_FOUND 또는 동등)
3. **L4 도구 강제 검증**:
   - `risk_tier="L4"` 인 항목이 1개 이상 정의되어 있다면, 모두 `status="not_exposed"` AND `maturity="forbidden"` 임을 assertion
   - 정의가 0개여도 통과(false safe)
4. 가짜 L4 도구를 fixture로 임시 추가하여 **차단이 실제로 작동**함을 입증

**실패 시 표기**: L4 도구가 노출되거나, manifest와 실제 노출이 불일치 → 즉시 보안 사고로 격상.

#### 2.1.4 테스트 4: pinned citation 강제 (R3 핵심)

**목적**: 인용에 `latest`가 남지 않고 pinned 버전만 검증 통과함을 보장. 또한 `citation_basis` 누락 케이스가 안전 기본값으로 처리됨을 확인.

**위치**: `backend/tests/integration/contract_drift/test_pinned_citation.py`

**구현 개요**:
1. **저장 경로 검증**:
   - REST `POST /citations` (또는 동등)에 `version_id="latest"` 입력 → DB에 vN 저장됨 + 응답에도 vN
   - DB 직접 쿼리: `SELECT COUNT(*) FROM citations WHERE version_id = 'latest'` → 항상 0
2. **검증 경로 검증**:
   - `verify_citation`에 `version_id="latest"` 입력 → 거부 (INVALID_PARAMS)
   - draft 버전(미공개)으로 검증 → `pinned=False` 반환
3. **citation_basis 기본값**:
   - 입력에서 `citation_basis` 누락 → `node_content`로 처리됨
4. **MCP/REST 일관성**:
   - REST에서 만든 인용을 MCP `verify_citation`으로 검증하면 정상 통과 (양쪽 인용 표현이 같은 시맨틱)

**실패 시 표기**: `latest`가 DB에 저장되거나, MCP/REST에서 다른 검증 결과 → drift.

#### 2.1.5 CI 게이트 등록

`.gitlab-ci.yml` (backend repo) 또는 동등 CI 설정에:

```yaml
contract_drift:
  stage: test
  script:
    - cd backend && pytest tests/integration/contract_drift/ -v
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
  needs: ["unit_tests"]
```

**실패 시 머지 차단**. 본 게이트는 R1·R3·R5의 자동 보호 장치.

#### 2.1.6 manifest drift 게이트

FG 4-0의 `dump_mcp_manifest.py` 출력(`산출물/MCP_도구_매니페스트.json`)이 코드 상수와 동기화되어 있는지 CI에서 확인:

```yaml
manifest_drift:
  stage: test
  script:
    - cd backend && python scripts/dump_mcp_manifest.py /tmp/current_manifest.json
    - diff /tmp/current_manifest.json ../docs/개발문서/S3/phase4/산출물/MCP_도구_매니페스트.json
```

차이가 있으면 fail — 개발자가 commit 갱신 필요.

#### 2.1.7 회귀 매트릭스 (선택, 권장)

위 4종 테스트의 결과를 표로 정리한 `산출물/FG4-4_Drift매트릭스.md`:
- 각 테스트의 통과/실패 여부
- 마지막 실행 시각
- CI run URL

### 2.2 제외 (이월)

- write 도구 contract drift (publish/draft) — FG 4-6 진입 시 함께
- 외부 클라이언트 호환 매트릭스 (예: Claude Desktop / 다른 MCP 클라이언트별 테스트) — 별도 작업
- Performance regression test (latency drift) — 본 FG는 정합성만

### 2.3 하드코딩 금지 재확인

- 4종 테스트 fixture는 helper 함수로 추출. 각 테스트가 데이터 시드 코드를 복제하지 않도록.
- L4 도구 검증은 manifest 순회로 동작 — tool name 인라인 금지.

---

## 3. 선행 조건

- FG 4-0 / 4-1 / 4-2 / 4-3 모두 종결
- Phase 0의 실 DB CI 인프라(testcontainers) 동작 — 본 FG는 실 DB 통합 테스트
- 운영자 승인: CI 게이트 등록 시점 — 머지 차단 정책 적용 의미가 있으므로

---

## 4. 구현 단계

### Step 1 — 공용 fixture 작성

1. `backend/tests/integration/contract_drift/conftest.py`
2. seed_helpers.py — Scope Profile + document + version + node 시드 함수
3. dual_client.py — REST/MCP를 동일 actor로 호출하는 thin wrapper

### Step 2 — 테스트 1 (동일 노드 결과)

1. seed N≥30 노드 (다양한 document_type/Scope/node_kind)
2. REST/MCP 호출 + 비교 함수 — content_hash + ACL 결과
3. 시나리오 6+

### Step 3 — 테스트 2 (latest 해석)

1. seed: 한 문서의 v1~v5 + draft 일부
2. 다양한 사용자 perspective로 latest 해석
3. resolved vN 비교

### Step 4 — 테스트 3 (비노출 도구)

1. `TOOL_SCHEMAS` 순회 + `not_exposed`/`forbidden` 추출
2. tools/list / tools/call 검증
3. 가짜 L4 도구 fixture로 차단 동작 입증

### Step 5 — 테스트 4 (pinned citation)

1. 저장 경로 / 검증 경로 / 기본값 / 일관성 4 분기
2. DB 직접 쿼리로 `latest` 미저장 단언

### Step 6 — CI 게이트 등록

1. `.gitlab-ci.yml` 갱신 (운영자 승인 후)
2. 첫 main pipeline 녹색 확인 (3회 연속 권장)
3. 머지 차단 활성화 확인

### Step 7 — manifest drift 게이트

1. dump 스크립트가 commit된 JSON과 매치되는지 검증
2. CI 단계 추가

### Step 8 — 검수 / Drift 매트릭스 보고서

- `FG4-4_검수보고서.md` — R1·R3·R5 자동 보호 확인
- `FG4-4_Drift매트릭스.md` — 4종 + manifest drift 게이트 통과 증거

---

## 5. API 계약 변경 요약

본 FG는 **테스트 추가만** — API 변경 없음.

CI 설정만 추가 — 머지 차단 정책 활성화.

---

## 6. 데이터 모델 주의사항

- 본 FG는 스키마 변경 없음
- 테스트 fixture는 통합 테스트의 testcontainers DB 사용 (Phase 0 자산)

---

## 7. 성공 기준

- [ ] 4종 contract drift 테스트 모두 녹색 (각 시나리오 5+)
- [ ] L4 도구 차단 (R1) 자동 보호 입증 — 가짜 L4 도구 fixture로 검증
- [ ] pinned citation (R3) 자동 보호 입증 — DB에 `latest` 0건
- [ ] CI 게이트 등록 + main 3회 연속 녹색
- [ ] manifest drift 게이트 등록
- [ ] 검수 보고서 + Drift 매트릭스 제출
- [ ] pytest 신규 ≥ 30 / 전체 베이스라인 유지

---

## 8. 리스크

| 리스크 | 대응 |
|-------|-----|
| 통합 테스트 flaky → CI 잦은 실패로 머지 정책 무력화 | testcontainers 안정화 + 테스트 격리 + 실패 시 retry 정책 (최대 1회). flaky 발견 즉시 격리 후 재현 의무 |
| 기존 PR이 본 게이트로 일괄 빨간색 → 머지 정체 | 게이트 등록 직후 1주는 warning-only 모드 (실패해도 머지 가능, 알림만). 운영자 합의 후 차단 모드 활성화 |
| L4 도구 정의가 0개 → 테스트 3이 무의미 | 가짜 L4 도구 fixture로 차단 동작 자체를 입증 (정의 0개여도 메커니즘은 검증) |
| CI 환경에서 pgvector 확장 미설치 | Phase 0 FG 0-1의 testcontainers 이미지 재사용 |
| 본 FG 이후 어떤 PR이 4종 중 하나를 우회 | 게이트가 머지 차단 — 우회 불가. 단 force push 등은 운영자 정책으로 별도 차단 |

---

## 9. 참조

- `Phase 4 개발계획서.md` §1.2 (R1·R3·R5), §2.1 (FG 4-4), §5.1 (게이트)
- `uploads/Mimir API : MCP 운영 논의 요약` §8 (Contract Drift 최소 테스트)
- `task4-0.md` ~ `task4-3.md`
- `CONSTITUTION.md` 제36·37·38조 (Test Layers), 제43조 (No Silent Test Weakening)
- `backend/tests/integration/conftest.py` (Phase 0 testcontainers 자산)
- `docs/개발문서/S3/phase0/산출물/FG0-1_보안취약점검사보고서.md` (실 DB CI 베이스라인)

---

*작성: 2026-04-27 | FG 4-4 — REST/MCP Contract Drift 자동 보호*
