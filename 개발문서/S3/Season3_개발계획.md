# Mimir Season 3 개발계획
## Editing · Grouping · Contributors — 사용자 경험 심화 스프린트

**작성일**: 2026-04-22
**기반**: S2 종결 보고서 · S2 재검수 2026-04-18 · S2-5 UI 개선 산출물(`docs/개발문서/S2_5/`) · 사용자 S3 킥오프 합의 (2026-04-22)
**상태**: 초안 — Phase 0 착수 전 합의 문서

---

## 0. S3 정체성

### 한 줄 정의

> **S3 = S2에서 확보한 "AI 1급 소비자" 기반 위에, 사람 작성자가 "쓰고·묶고·공동작업하는" 경험을 심화시키는 스프린트**

S1이 "문서 플랫폼 0→1", S2가 "AI가 지식 백본으로 쓸 수 있게" 였다면, **S3는 "사람이 매일 쓰고 싶은 편집·탐색 경험을 내재화"하는 국면**이다. 동시에 S2 종결 시점에 드러난 P0 품질 부채(실 DB CI 부재, 커버리지 35%, AI 품질 실측 부재)를 Phase 0에서 선행 정리한다.

### S2에서 S3로의 전환 근거

S2 종결 시 Mimir는 다음을 확보했다: MCP Tool 계약, 검증 가능한 citation 5-tuple, RAG 회귀 평가 인프라, 에이전트 Draft 쓰기 경로, 챗봇 세션 도메인, Scope Profile 기반 ACL, 폐쇄망 동등성 구조. 그러나 사용자 경험 관점에서는 다음 갭이 남아 있다:

1. **편집 경험의 단일 모달**: 현재 편집기는 블록 중심 ProseMirror 뷰 단일 — 일반 리치텍스트처럼 흐르는 편집을 선호하는 사용자가 막힘을 느낌
2. **문서 탐색의 단일 축**: DocumentType + Scope Profile + 제목 검색만 존재. 사용자가 자신의 맥락으로 "묶어 보기"(컬렉션·태그·백링크·폴더·그래프) 할 수단 없음
3. **협업 가시성의 단일 뷰**: 작성자·편집자·승인자·최근 열람자가 어느 한 화면에 모이지 않아 누가 무엇에 관여했는지 한눈에 파악 불가
4. **S2 실측 부채**: 단위 테스트 479건 통과에도 실 DB CI가 없어 BUG-01~05(트랜잭션 오염, pgvector 차원 불일치 등)가 종결 직전에야 드러남. 커버리지 35.44%, AI 품질 Faithfulness/Citation-present 실측 전무. 이 상태로 기능을 더 얹으면 부채가 기하적으로 증가함

S3는 (4)를 Phase 0 게이트로 먼저 해소한 뒤 (1)(2)(3)을 차례로 개발한다.

---

## 1. 설계 원칙

### S1 4대 원칙 (유지)

| # | 원칙 | 요약 |
|---|------|------|
| ① | API-first | 모든 기능은 API로 먼저 정의. UI는 클라이언트 중 하나 |
| ② | Dual UI | User UI / Admin UI 분리 |
| ③ | Document = 구조, Type = 행동 | Document는 generic, DocumentType이 동작 정의 |
| ④ | 권한은 모든 계층에 적용 | API · 검색 · 벡터 · RAG |

### S2 3대 원칙 (유지)

| # | 원칙 | 요약 |
|---|------|------|
| ⑤ | AI는 1급 사용자다 | 모든 기능은 사람 UI 이전에 AI 에이전트가 안전하게 호출·검증·기록 가능해야 함 |
| ⑥ | 접근 범위도 하드코딩 금지 | 외부 소비자의 scope 어휘는 Scope Profile로 관리자가 설정 |
| ⑦ | 폐쇄망 동등성 | 외부 의존은 환경변수로 off 가능, off 시 degrade 하지만 실패하지 않음 |

**우선순위**: S1 ①~④ > S2 ⑤~⑦. S3에서 추가되는 기능이 S1/S2 원칙과 충돌할 경우 S1이 승.

### S3 추가 고려사항 (원칙을 추가하지 않는다)

S3는 **새 원칙을 도입하지 않는다**. 그 대신 기존 7개 원칙의 엄격 적용을 강조한다:

- **⑥의 재확인 (핵심)**: 폴더·컬렉션·태그·백링크·그래프 뷰·Saved View는 **순수 뷰(view) 레이어**다. ACL은 오직 Scope Profile만이 결정한다. 폴더 이동이 권한을 바꾸면 안 되고, 컬렉션에 추가했다고 접근 범위가 넓어지면 안 된다. "뷰"와 "권한"의 분리는 S3 모든 Phase에서 절대 규칙이다.
- **①의 재확인**: 문서 편집기의 일반 뷰/블록 뷰 토글은 **UI 모드 전환일 뿐**이며, 저장 모델은 단일(ProseMirror 트리)이다. 두 개의 저장 포맷을 만들지 않는다 — BUG-02(content_snapshot vs nodes 이중 경로) 유형 원천 차단.
- **⑦의 재확인**: Vault Import(옵시디언 zip 업로드)는 외부 API 호출이 없는 일회성 로컬 파이프라인으로 설계한다. 폐쇄망 Mimir 내에서 네트워크 없이 동작.
- **산출물 규약 강화**: 모든 FG는 작업지시서 + 검수보고서 + 보안취약점검사보고서가 의무이며, AI 기능 포함 FG는 AI품질평가보고서를 추가로 제출한다. Phase 0는 각 FG마다 **근거 데이터(JSON/로그)** 첨부가 추가 의무.

---

## 2. 핵심 제약 조건

### 2.1 1차 소비자는 여전히 사람과 챗봇, 둘 다

S3의 사용자 경험 개선은 사람 사용자를 타겟하지만, 챗봇(S2의 1차 외부 소비자)도 새 기능을 API로 접근할 수 있어야 한다. 예:
- 에이전트가 "내가 최근 편집한 문서" 쿼리를 호출할 수 있어야 함 (Contributors 기능의 API 표면)
- 에이전트가 백링크(`[[문서명]]`)를 통해 관련 문서를 탐색할 수 있어야 함 (MCP Tool로 노출)
- 폴더·컬렉션·태그는 에이전트 검색의 메타필터로도 사용 가능해야 함 (단, ACL은 여전히 Scope Profile 단독 결정)

### 2.2 폐쇄망 배포 지속

- Vault Import는 로컬 zip 처리만 — 외부 URL 호출 금지
- 그래프 뷰 렌더링 라이브러리는 로컬 번들(sigma.js / cytoscape.js 등 폐쇄망 허용 OSS). CDN 의존 금지
- 편집기 모드 토글은 클라이언트 코드만 — 서버는 단일 저장 포맷

### 2.3 S2 종결 부채 우선 정리 (Phase 0 게이트)

S2 재검수 2026-04-18 보고서가 명시한 P0 3건 + 인계 P0 1건을 Phase 0에서 해소한다. 이 게이트를 통과하지 않으면 Phase 1 개발에 진입하지 않는다:

1. 실 DB CI 부재 → testcontainers 또는 docker-compose 기반 PostgreSQL + Valkey + pgvector CI 도입
2. 테스트 커버리지 35.44% → 80% 복구 (서비스/리포지토리 중심)
3. AI 품질 실측 부재 → 골든셋 50+건 확장, Faithfulness ≥ 0.80, Citation-present ≥ 0.90 **실측값**으로 확보
4. `embedding_dim` config ↔ `document_chunks` 스키마 자동 검증 (BUG-04 재발 방지)

### 2.4 UI 호환성

- 데스크탑/웹 모두 호환 (CLAUDE.md UI 규칙)
- UI 디자인 리뷰 **FG당 최소 5회** (CLAUDE.md)
- 모바일은 S3 범위 밖이나, 레이아웃이 모바일을 **원천 차단하지 않도록** (반응형 breakpoint는 도입 단계에서만 설계)

### 2.5 마이그레이션 안전

- 기존 published 문서의 content_snapshot은 단일 ProseMirror 트리로 정규화되어 있음 (S2 종결 기준). Phase 1에서 "일반 에디터 뷰" 추가 시 저장 포맷은 **변경 금지**
- 폴더·컬렉션·태그는 **신규 테이블**로 추가. 기존 `documents` 스키마는 필드 추가 수준에서만 수정 (기본값 null 허용)
- Alembic revision 경로를 통해서만 스키마 변경. `init_db` 직접 수정 금지

---

## 3. Phase 구조

### 3.1 Phase 개요

| Phase | 목적 | 게이트 | 산출 FG 수 | 이전 Phase 의존 |
|------|------|-------|-----------|----------------|
| Phase 0 | S2 잔존 P0 정리 | 실 DB CI 녹색 + 커버리지 ≥ 80% + AI 품질 실측 제출 + embedding_dim 검증 Alembic | 4 | — |
| Phase 1 | 에디터 모드 토글 (단일 ProseMirror 저장) | 블록/일반 뷰 양방향 편집 실제 동작, node_id stable 규약 테스트 통과 | 3 | Phase 0 |
| Phase 2 | 그룹화 풀세트 + 뷰 전환 + Vault Import | 컬렉션/태그/백링크/그래프/폴더 + 레이아웃 4종 + Saved View + Vault Import A안 | 6 | Phase 1 |
| Phase 3 | Contributors 패널 + 인라인 주석 | 작성자/편집자/승인자/최근 열람자 한 패널 + node_id 기반 인라인 주석 + 권한 설정 on/off | 3 | Phase 2 |

**Phase 4 (이월)**: 외부 커넥터(Confluence/Notion/SharePoint Import) — S3 2라운드 또는 S4로.

### 3.2 Phase 의존 그래프

```
[Phase 0] -+-> [Phase 1] --> [Phase 2] --> [Phase 3]
           |
           +-> (Phase 0 게이트는 Phase 1~3 공통 선행)
```

### 3.3 FG별 절대 규칙 (Phase 1~3 공통)

1. **저장 모델 단일성**: content_snapshot이 단일 소스. nodes는 파생 동기화 테이블. 두 개의 쓰기 경로 금지.
2. **뷰 ≠ 권한**: 폴더·컬렉션·태그·그래프·Saved View 어느 것도 ACL을 결정하지 않음. ACL은 오직 Scope Profile.
3. **node_id 안정성**: 블록/일반 뷰 전환 시 node_id가 변하지 않아야 함. 인라인 주석·citation 좌표가 node_id에 매달려 있기 때문.
4. **에이전트 동등성**: 새 기능은 API로 먼저 정의. MCP Tool 노출 대상(Contributors/백링크 탐색 등)은 S3 내에서 Tool schema에 등록.

---

## 4. 산출물 규약

### 4.1 경로

- S3 전체 산출물은 `docs/개발문서/S3/` 하위
- Phase별 디렉터리: `phaseN/`
- Phase 개발계획서: `phaseN/Phase N 개발계획서.md`
- 작업지시서: `phaseN/작업지시서/taskN-*.md`
- 검수·보안 보고서 및 AI 품질 평가: `phaseN/산출물/`

### 4.2 FG별 의무 산출물

| 산출물 | 모든 FG | AI 기능 포함 FG |
|--------|--------|----------------|
| 작업지시서 (`taskN-*.md`) | ✅ | ✅ |
| 검수보고서 | ✅ | ✅ |
| 보안취약점검사보고서 | ✅ | ✅ |
| AI품질평가보고서 | — | ✅ |
| 근거 데이터 (JSON/로그) | Phase 0 한정 | Phase 0 한정 |

### 4.3 검수 기준

- CLAUDE.md S1 ⑤: 개발 후 검수 수행 및 검수 결과 보고서 작성
- CLAUDE.md S1 ⑥: 보안 취약점 검사 필수
- S2 산출물 규약: Faithfulness ≥ 0.80, Citation-present ≥ 0.90 (AI 기능 한정)
- S3 UI FG(Phase 1~3의 모든 UI FG): UI 디자인 리뷰 최소 5회 — 리뷰 요약을 검수보고서에 별지로 첨부

---

## 5. Phase별 범위 요약

### Phase 0 — 기반 부채 정리 (즉시 착수)

| FG | 제목 | 핵심 |
|----|------|-----|
| 0-1 | 실 DB 통합 테스트 CI | testcontainers 기반 PostgreSQL + Valkey + pgvector, 기존 전부 단위 테스트 + 주요 통합 시나리오 실행, 녹색 필수 |
| 0-2 | embedding_dim 자동 검증 | Alembic revision에서 config와 `document_chunks` vector 차원 자동 비교, 불일치 시 revision 실패. 기동 시 헬스체크에도 포함 |
| 0-3 | 테스트 커버리지 80% 복구 | 서비스/리포지토리 중심, 35.44% → 80%. 커버리지 리포트는 CI artifact로 보관 |
| 0-4 | AI 품질 실측 — 골든셋 50+ | `scripts/rag_smoke/golden_queries.py`를 기반으로 50건 이상으로 확장. Faithfulness ≥ 0.80, Citation-present ≥ 0.90 실측 확보. 판정 LLM은 폐쇄망 호환(로컬 모델 사용 가능) |
| 0-5 | 문서별 벡터화 상태 가시성 + 재벡터화 버튼 | 문서 상세에 상태 뱃지(indexed/stale/failed/…) + Admin·작성자용 재벡터화 버튼. 조용한 실패(publish 성공·벡터화 실패) 재발 방지 (2026-04-23 실사례) |

### Phase 1 — 에디터 모드 토글 (Phase 0 후 착수)

| FG | 제목 | 핵심 |
|----|------|-----|
| 1-1 | 저장 모델 정규화 | content_snapshot 단일 소스 확정, nodes 동기화 테이블 정리. BUG-02 유형 차단 회귀 테스트 |
| 1-2 | 일반 에디터 뷰 구현 | 리치텍스트 뷰. 블록 뷰와 node_id를 공유. 토글 버튼 |
| 1-3 | 토글 안정성 | node_id 변화 없음 테스트, 인라인 주석/citation의 좌표 보존 테스트 |

### Phase 2 — 그룹화 풀세트 + 뷰 전환 + Vault Import (Phase 1 후 착수)

| FG | 제목 | 핵심 |
|----|------|-----|
| 2-1 | 수동 컬렉션/폴더 | 사용자가 만들고 관리하는 컬렉션. 계층 폴더 구조. ACL과 독립 |
| 2-2 | 태그 동적 그룹 | `#tag` 추출 + 태그로 동적 그룹. 태그 ∈ frontmatter 또는 본문 인라인 |
| 2-3 | 백링크 `[[문서명]]` | 본문 인라인 참조 파싱, 양방향 그래프 에지 생성 |
| 2-4 | 그래프 뷰 + 뷰 전환 | 리스트/트리/카드/그래프 4종 레이아웃 전환 |
| 2-5 | Saved View | 필터 + 정렬 + 레이아웃을 Saved View로 저장. 사용자별. URL 공유 가능 (ACL 필터는 여전히 뷰어 기준) |
| 2-6 | Vault Import (A안) | 옵시디언 vault zip 업로드 → markdown + frontmatter + `[[링크]]` + `#태그` + 폴더 → ProseMirror 트리 변환. 업로드 시 Scope Profile 선택 필수·기본값 private. zip bomb 방어·파일 수 상한·개인정보 스캔 |

### Phase 3 — Contributors 패널 + 인라인 주석

| FG | 제목 | 핵심 |
|----|------|-----|
| 3-1 | Contributors 패널 | 작성자/편집자/승인자/최근 열람자를 한 사이드 패널에 표시. `audit_logs` + `workflow_history` + `documents.*_by` 조합 |
| 3-2 | 열람자 노출 제어 | Scope Profile 설정 항목으로 열람자 노출 on/off. 정보보안/개인정보 충돌 방지 |
| 3-3 | 인라인 주석 | node_id 기반 anchoring. 버전 간 보존. 해결/미해결 상태 |

---

## 6. 게이트 정의 (완수 기준)

### 6.1 Phase 0 완료 = S3 게이트 통과

- [ ] 실 DB CI 워크플로가 `main` 브랜치 녹색으로 3회 연속 성공
- [ ] 백엔드 커버리지 리포트 ≥ 80% (서비스/리포지토리 기준)
- [ ] `scripts/rag_smoke/` 기반 실측 JSON 제출, Faithfulness ≥ 0.80 · Citation-present ≥ 0.90
- [ ] Alembic revision이 embedding_dim 불일치 시 실패함을 증명하는 테스트 통과
- [ ] 각 FG별 검수보고서·보안취약점검사보고서 제출

### 6.2 Phase별 완료 기준

각 Phase 개발계획서의 §7 "완료 기준"에서 구체화한다. 공통 조항:
- 해당 Phase의 모든 FG가 검수·보안 보고서 제출 완료
- UI 변경이 있는 Phase는 UI 디자인 리뷰 ≥ 5회 로그 제출
- 회귀 테스트: 이전 Phase가 설정한 테스트가 모두 녹색 유지

### 6.3 S3 1라운드 종결 기준

- 모든 Phase 게이트 통과
- S1 ①~④ + S2 ⑤~⑦ 원칙 위배 회귀 테스트 통과
- 종결 회고 작성

---

## 7. 범위 밖 / 이월

### 7.1 S3 1라운드 명시적 제외

- **외부 SaaS 커넥터 import** (Confluence, Notion, SharePoint 등) — S3 2라운드 또는 S4
- **실시간 공동 편집** (CRDT/OT 기반 동시 편집) — 별도 스프린트
- **모바일 네이티브 UX 최적화** — 반응형 레이아웃 차단 안 하는 수준만 준수
- **Vault Import 이외의 양방향 동기화** — Vault Import는 A안(일회성) 확정

### 7.2 이월 관리

Phase 4 이월 후보는 `docs/개발문서/S3/이월.md`로 별도 관리한다(필요 시). 단, S3 1라운드 범위에 포함되지 않은 항목을 "사실상 포함"시키지 않도록 한다(범위 크립 방지).

---

## 8. 참조

- `docs/개발문서/S2/Season2 개발계획.md` — S2 전체 계획
- `docs/개발문서/S2/S2 개발계획서 검수 패치안.md` — S2 재검수 (2026-04-18)
- `docs/개발문서/S2_5/RAG_스모크_실행가이드.md` — Phase 0-4의 베이스라인 스크립트
- `docs/개발문서/S2_5/RAG_스모크_검수보고서.md` — Q05/Q15 keyword 교정 이력
- `CLAUDE.md` — 프로젝트 절대 규칙 (S1 ①~④, S2 ⑤~⑦)
- `/sessions/eloquent-practical-ramanujan/mnt/.auto-memory/project_s3_kickoff.md` — S3 킥오프 사용자 합의

---

*작성: 2026-04-22 | Mimir S3 킥오프 합의 문서*
