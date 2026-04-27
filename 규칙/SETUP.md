# SETUP.md — Ready for AI SW 헌법 시스템 도입 가이드

> 이 문서는 **신규 또는 기존 프로젝트에 Ready for AI SW 헌법 시스템을 도입하는 방법**을 단계별로 안내한다.
>
> 대상: 헌법을 처음 채택하는 팀 리드·플랫폼 엔지니어·아키텍트, 그리고 도입을 돕는 Claude·Codex.
>
> 이 문서는 **한 번만 읽는 부트스트랩 가이드**이므로 `CLAUDE.md`·`AGENTS.md`의 매-작업 읽기 순서에는 들어가지 않는다. 셋업이 끝나면 역할은 6개 정착 문서가 이어받는다.

---

## 0. 문서 세트 구성

Ready for AI SW 시스템은 **6개 문서**로 구성된다.

| 파일 | 성격 | 변경 주기 |
| --- | --- | --- |
| `CONSTITUTION.md` | 규범 (불변에 가까움) | 매우 드묾 — Meta-1 절차 |
| `AGENT_MODE.md` | 프로젝트 선언 | 운영 모드 전환 시 |
| `AUTOMATION.md` | 운영 명세 | CI·디스패처·오케스트레이터 추가·변경 시 |
| `CLAUDE.md` | 역할 지침 (Claude) | 역할·루틴 조정 시 |
| `AGENTS.md` | 역할 지침 (Codex) | 역할·루틴 조정 시 |
| `CONTRIBUTING.md` | 일상 루틴 | 기여 관행 변경 시 |

셋업은 이 6개 파일을 프로젝트에 **심고, 프로젝트별 사정에 맞게 채우는 것**이다.

---

## 1. 이 시스템을 쓸지 결정하기

**도입 권장**:

- AI 에이전트(Claude·Codex·기타)가 **실제로 코드를 쓰거나 리뷰하는** 프로젝트
- 에이전트 행동이 시스템 안전·사용자 권한·데이터 무결성에 영향을 줄 수 있는 경우
- 복수 에이전트 또는 복수 세션이 같은 repo에서 작동하는 경우
- 팀이 AI 협업의 일관성·감사 가능성을 높이고 싶은 경우

**도입 비권장**:

- 에이전트를 자동완성·단순 생성에만 쓰는 경우
- 개인 토이 프로젝트로 감사·정책 요구가 없는 경우
- 팀이 아직 AI 협업 경험이 적어 먼저 실무 감각을 쌓아야 하는 경우

기존 코드베이스가 크고 AI 도입이 처음이라면 **`human-only` 모드로 먼저 도입**해서 문서 루틴(`functions.index.md`, Change Boundary, 테스트 계층)만 정착시키고, 이후 `codex-only` 또는 `dual-agent`로 승격하는 경로가 안전하다.

> **Mimir 본 프로젝트의 적용**: 본 SETUP.md 는 framework 가이드 (다른 repo 도입용) 로 유지된다. Mimir 자체는 헌법 제11조의 per-directory `functions.index.md` 모델 대신 **중앙 함수도서관 모델**(`docs/함수도서관/{backend,frontend}.md`) 로 구현했다 (2026-04-25 신설). 본 문서의 `functions.index.md` 표현을 Mimir 적용 시 `docs/함수도서관/{backend,frontend}.md` 로 치환해 읽는다. 헌법 제11조 자체와의 정합은 별 ADR 사안 (S3 종료 후 처리).

---

## 2. 사전 준비물

**도구**:

- git 접근 권한 (repo owner)
- Markdown 편집기
- (선택) Claude — Claude Code, 데스크톱, 또는 API 중 하나
- (선택) Codex — Codex CLI 또는 IDE 플러그인
- CI 환경 (GitHub Actions, GitLab CI, CircleCI 등) — Phase 6에서 필요

**접근·구독**:

- 최소 1개 LLM 구독 또는 API 키. Dual-agent는 Claude + ChatGPT(또는 Codex) 둘 다 필요.
- CI 환경의 secret 관리 권한

**조직적 준비**:

- P1 승인을 실행할 수 있는 사람(메인테이너 또는 팀 리드)이 지정되어 있어야 함
- 민감 경로(auth, migration, policy) 소유자가 정해져 있어야 함

---

## 3. 빠른 시작 (Quick Start)

새 프로젝트에 **최소 셋업**으로 시작하는 경로. 약 30~60분 소요.

### 3.1 파일 복사

본 ready-for-ai-sw 디렉토리의 6개 파일을 대상 프로젝트 루트로 복사한다.

```
<target-project>/
├── CONSTITUTION.md        # 수정하지 않는다 (규범 원천)
├── AGENT_MODE.md          # Phase 1에서 채운다
├── AUTOMATION.md          # Phase 3에서 채운다
├── CLAUDE.md              # Phase 2에서 조정
├── AGENTS.md              # Phase 2에서 조정
└── CONTRIBUTING.md        # Phase 4에서 프로젝트화
```

### 3.2 최소 결정 3가지

- **Operation Mode**: `dual-agent` / `claude-only` / `codex-only` / `human-only` 중 하나
- **Dispatcher Maturity**: `declared`로 시작 (자동화 없이)
- **프로젝트 소유자**: P1 승인을 할 수 있는 1명 지정

### 3.3 첫 커밋

```
git add CONSTITUTION.md AGENT_MODE.md AUTOMATION.md CLAUDE.md AGENTS.md CONTRIBUTING.md
git commit -m "chore: adopt Ready for AI SW constitution v1.0"
```

이제 세부 단계로 넘어간다.

---

## 4. 단계별 상세 셋업

### Phase 0 — 문서 복사 & 프로젝트 정체성 기록 (10분)

6개 파일을 복사한 후, **수정하지 않는다**. 먼저 각 파일 상단의 메타데이터만 채운다.

`AGENT_MODE.md` Section 1:

```
- project_name: <실제 이름>
- repository: <URL 또는 경로>
- constitution_version: v1.0 (2026-04-24)
- owner: <사람 이름 또는 팀>
- last_updated: <오늘 날짜>
```

`AUTOMATION.md` Section 1:

```
- scope: <예: 이 monorepo 전체 / 특정 서비스만>
- owner: <자동화 운영 책임자>
- last_reviewed: <오늘 날짜>
- constitution_version: v1.0
```

### Phase 1 — Operation Mode 결정 (15~30분)

`AGENT_MODE.md` Section 2·3·4를 채운다.

**Mode 선택 기준**:

- **dual-agent**: Claude·Codex 둘 다 구독 + 팀이 두 도구 경험 있음 → 권장 표준
- **claude-only**: Claude만 있음 + 사람 리뷰 체력 있음
- **codex-only**: Codex만 있음 + 설계는 사람이 주도
- **human-only**: AI 도구 없이 문서 루틴만 채택

**Routing Policy** (Section 3) 채우기:

`minimal` / `standard` / `extended` 각각에 대해 **이 프로젝트에서 누가 무엇을 하는지** 실제 이름·역할로 채운다. 예:

```
## 3.3 extended
- design: Claude
- implementation: Codex
- review: Claude
- human approval: @alice (팀 리드)
- validation: unit + contract + policy + recovery
- required Change Boundary level: extended
```

**Single-Agent Rules** (Section 4)는 Single-Agent 모드를 선택한 경우에만 채운다. 그 외 `N/A`.

**Dispatcher Maturity** (Section 6)는 **무조건 `declared`로 시작**한다. 자동화는 나중이다.

### Phase 2 — CLAUDE.md / AGENTS.md 조정 (15분)

두 파일은 **대부분 그대로 유지**한다. 다만 다음 부분만 프로젝트에 맞게 조정한다.

- **Section 1.1 Operation Mode 표**: Phase 1에서 고른 모드에 해당하는 행을 강조하거나 나머지를 삭제해서 읽기 부담을 줄여도 됨
- **Section 10 (크기 규율)**: 팀이 3000~4000 tokens보다 더 긴·짧은 제약을 원하면 조정
- **Section 11 (참조 인덱스)**: 자주 쓰이는 조항 매핑. 팀이 선호하는 것을 추가

**주의**: CLAUDE.md / AGENTS.md를 과도하게 뜯어고치지 말 것. 큰 변경은 P1(Meta-1)이다. 초기 도입 단계에서는 `minimal change` 원칙.

### Phase 3 — AUTOMATION.md 초기 선언 (20분)

도입 초기에는 Layer 1 CI 체크 **2~3개만** 선언하고 나머지는 "TBD"로 둔다.

**초기 최소 CI 체크** (Section 2):

```
- check name: check_change_boundary_present
  related constitution articles: 제28·29조
  mode: warning                    # 시작은 warning
  trigger: 모든 PR
  failure message: "PR 본문 또는 링크된 handoff 파일에 Change Boundary Block이 없습니다."
  bypass risk: "필드를 형식적으로 채우기 (Failure Mode #2 참고)"
  owner: @alice

- check name: check_p1_tag_required
  related constitution articles: 제33조
  mode: warning
  trigger: extended 라벨이 붙은 PR
  failure message: "..."
  bypass risk: "extended 성격인데 라벨을 빼고 제출"
  owner: @alice
```

**Dispatcher (Section 3)**:

처음엔 `enabled: no`. 팀이 익숙해지면 thin-dispatcher 단계에서 활성화.

**Orchestrator (Section 4)**:

처음엔 `enabled: no`. 이걸 `yes`로 바꾸는 것은 한참 후의 결정이다.

**Section 6·7 (Failure Modes / Regression Cases)**:

**초기 4개 우회 패턴을 즉시 등록**한다. 이건 CONSTITUTION.md Meta-4에서 이미 관찰된 패턴이다:

```
- known bypass pattern: "public 함수를 _private 접두사로 바꿔 인덱스 CI 회피"
- known bypass pattern: "Change Boundary intent를 'misc' 또는 공백으로 채움"
- known bypass pattern: "실패 테스트를 기존 파일에 합쳐 삭제한 것처럼 보이게"
- known bypass pattern: "skip/xfail reason을 형식적으로만 기입"
```

셋업 시점에 이게 있어야, 첫 기여자들이 같은 실수를 하지 않는다.

### Phase 4 — CONTRIBUTING.md 프로젝트화 (15분)

다음 3개 섹션만 프로젝트에 맞게 수정한다:

- **Section 3.1** ("주요 패키지·모듈" 범위) — 실제 `src/` 하위 경로 나열
- **Section 6.3** (PR 템플릿) — 템플릿 그대로 복사해서 `.github/pull_request_template.md`로 저장
- **Section 7** (테스트 계층) — 프로젝트 테스트 프레임워크·디렉토리에 맞게 예시 조정

Section 4 "Multi-Model Design Review as Practice"는 **그대로 유지**한다. 이건 모든 프로젝트가 공유하는 방법론이다.

### Phase 5 — 첫 `functions.index.md` 만들기 (30분~2시간, repo 크기에 따름)

> **Mimir 본 프로젝트는 본 Phase 5 의 per-directory 모델 대신 중앙 도서관 모델 (`docs/함수도서관/`) 을 채택했다** (§1 주석 참조). 본 절은 framework 가이드의 일반 절차로, 다른 repo 도입 시 사용한다.

**신규 프로젝트**: 함수를 작성할 때마다 같은 커밋에서 인덱스를 추가·갱신한다. 처음부터 엄격하게.

**기존 프로젝트 (legacy 도입)**: **일괄 백필**이 아니라 **촉발 단위 백필**을 권장한다.

1. 최상위 `src/` 하위 1~2 depth 디렉토리마다 빈 `functions.index.md` 파일 생성
2. 각 디렉토리 README에 "이 인덱스는 점진적으로 채워집니다" 한 줄 추가
3. **해당 디렉토리 코드가 수정될 때마다**, 그 커밋에서 수정된 함수만 인덱스에 추가
4. 3개월 후 커버리지를 측정해 Maturity Stage를 `checklist`로 승격 고려

일괄 자동 생성은 CONSTITUTION.md 제12조에서 반대된다 — 자동 생성은 의미 필드를 못 채운다.

**중앙 도서관 모델 (Mimir 채택)**: per-directory 인덱스 대신 단일 카탈로그 (`docs/함수도서관/backend.md` / `frontend.md` + `README.md` index) 로 함수 지도를 구성하는 변형도 가능하다. 트레이드오프는: (장) 단일 진입점·중복 발견 용이·운영자 한 곳만 관리. (단) 디렉토리 로컬리티 약화·헌법 제11조 본문(per-directory)와 정합 별 ADR 필요. 도입은 자율, 채택 시 본 Phase 5 의 1~4 번 절차를 단일 카탈로그 갱신으로 대체한다.

### Phase 6 — PR 템플릿 + 최소 CI 체크 (30분)

`.github/pull_request_template.md`를 `CONTRIBUTING.md` Section 6.3에서 복사해서 저장.

**최소 CI workflow** 예시 (`.github/workflows/constitution-check.yml`):

```yaml
name: Constitution Check
on: [pull_request]
jobs:
  basic-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check Change Boundary present in PR body
        run: |
          if ! echo "${{ github.event.pull_request.body }}" | grep -q "Change Boundary:"; then
            echo "::warning::PR body missing Change Boundary Block (제28·29조)"
          fi
      - name: Check P1 tag consistency
        run: |
          # extended 라벨이 붙은 PR인지 확인
          # 본문에 reversibility impact가 있는지 확인
          # 간단한 heuristic 스크립트
          ...
```

첫 주엔 모든 체크를 `warning`으로 둔다. 한 달 후 `hard-fail`로 승격.

---

## 5. Dispatcher Maturity 승격 경로

```
declared → checklist → thin-dispatcher → orchestrator-experimental
```

각 승격은 **P1 변경**이다(Meta-1). Change Boundary `extended`로 기록.

### 5.1 declared → checklist 승격 기준

- [ ] PR 템플릿이 최소 2주간 사용됨
- [ ] 팀원의 70% 이상이 Change Boundary를 자연스럽게 작성함
- [ ] `functions.index.md` 커버리지가 주요 public 함수의 50% 이상
- [ ] CI 체크 최소 2개가 `warning` 단계로 작동 중
- [ ] Failure Modes Section 6에 최소 2개 사례가 등록됨

### 5.2 checklist → thin-dispatcher 승격 기준

- [ ] 전체 CI 체크가 `hard-fail`로 안정 운영 중 (최소 2주 무장애)
- [ ] `functions.index.md` 커버리지 90% 이상
- [ ] Bypass Regression Cases가 Agent Behavior Eval에 등록됨
- [ ] Audit log 스키마(`AUTOMATION.md` Section 5)가 실제로 수집되고 조회 가능
- [ ] 팀이 Disagreement Record를 실제로 써본 경험 있음

### 5.3 thin-dispatcher → orchestrator-experimental 승격 기준

- [ ] Stop Conditions 8개가 모두 실제로 구현·검증됨
- [ ] Kill Switch가 테스트됨 (실제로 중단 가능 확인)
- [ ] Layer 3 허용 경로가 3개 이하로 좁게 정의됨
- [ ] P0·P1 액션에 대한 자동화 우회가 존재하지 않음을 Policy Test로 증명
- [ ] 사람 승인 없이 자동 머지되는 경로가 `none`임을 문서화

**건너뛰기 금지**. 성숙도가 두 단계 위로 갑자기 올라가는 것은 Meta-3 운영 원칙 3("Dispatchers Recommend, Policies Decide")을 위반하기 쉽다.

---

## 6. 첫 작업으로 검증하기

셋업 후 3개 시나리오로 시스템을 검증한다.

### 6.1 시나리오 A (minimal) — README 오타 수정

- Change Boundary Block `minimal` 예시를 따라 작성
- CI가 무엇을 체크하는지 확인
- PR 머지까지 5분 이내 목표

**검증 항목**: 템플릿이 불편하지 않은지, 체크리스트가 과도하지 않은지.

### 6.2 시나리오 B (standard) — 기존 함수에 버그 수정

- `functions.index.md`의 해당 함수 항목 업데이트 필요 여부 확인
- 테스트 추가·갱신
- `validation performed` 필드에 실제 실행 로그 기록
- Change Boundary `standard`로 PR 제출

**검증 항목**: 인덱스 갱신이 자연스러운가, `validation performed` 필드가 실제로 유용한가.

### 6.3 시나리오 C (extended) — 새 도구(tool) 추가

- CONSTITUTION.md 제5조 Agent-Facing Contract로 설계
- ADR 초안 작성
- **이 시나리오에서 Multi-Model Design Review 활용**:
  1. Topic Seed: 사용자가 "이런 도구가 필요하다, A/B/C 기능 포함"
  2. Claude와 ChatGPT(또는 가용한 다른 모델)가 계약·에러 스키마·권한 설계 회의
  3. Checkpoint Review: 사용자가 결과 검수
  4. 합의 후 Codex 구현, Claude 리뷰, 사람 승인
- Change Boundary `extended`로 PR
- `[P1]` 태그

**검증 항목**: CONTRIBUTING.md Section 4의 4단계 구조가 실제로 흐르는가. Disagreement Record가 필요한 지점이 있었는가.

---

## 7. 흔한 셋업 실패와 함정

- **CLAUDE.md / AGENTS.md를 너무 일찍 뜯어고침** — 초기엔 템플릿 그대로 써본 후 실제 마찰 지점을 관찰하고 수정. 처음부터 상상으로 수정하면 과도 설계.
- **AGENT_MODE.md를 비우고 시작** — "나중에 채우자"는 실수. 모드 선언이 없으면 에이전트는 `extended + human-approval`로 보수적으로 동작하므로 모든 작업이 느려진다.
- **모든 CI를 처음부터 hard-fail** — 팀 반발로 체크를 전부 우회하게 됨. 1~2주 `warning` 단계 필수.
- **Single-Agent Mode인데 Policy/Eval 테스트 생략** — Meta-3 운영 원칙 2 위반. 자동 검증이 교차 에이전트 리뷰를 대체해야 함.
- **자동 생성으로 functions.index.md 일괄 채움** — 시그니처는 맞지만 intent·effects가 엉망. 제12조가 반자동(자동 초안 + 사람 보강)을 권장하는 이유.
- **Orchestrator(Layer 3)를 초기에 활성화** — `AUTOMATION.md` Section 4 Stop Conditions가 채워지기 전에 `enabled: yes`로 설정하면 심각한 사고 위험.
- **Disagreement Record를 "귀찮으니 생략"** — 암묵 수렴은 미래의 비용이다. 본 헌법 수립 과정에서도 몇 번 겪음.
- **constitution_version 미기록** — 나중에 Meta-1 개정 시 어느 버전의 헌법에 대한 변경인지 못 찾음.

---

## 8. 기존 프로젝트에 도입하기 (Retrofit)

**원칙**: 처음부터 완벽을 시도하지 말 것. 점진 도입.

1. **Week 0**: 6개 문서 복사 + `AGENT_MODE.md` 채움 + `CONSTITUTION.md` 팀 읽기 세션 (1시간)
2. **Week 1**: PR 템플릿 배포. `functions.index.md`는 빈 파일만. Change Boundary를 `warning`으로만.
3. **Week 2~4**: 새 PR에만 Change Boundary 적용. legacy 코드는 건드리지 않음.
4. **Month 2**: 수정된 파일의 `functions.index.md`만 점진 백필. CI 체크 2~3개 `warning` 운영.
5. **Month 3**: Failure Modes 카탈로그에 실제 관찰 사례 2~3개 누적. CI 체크 `hard-fail` 승격 검토.
6. **Month 4+**: `checklist` 단계 승격 검토 (Section 5.1 기준 확인).

**legacy `functions.index.md` 작성 전략**:

- 자동 생성 금지
- 수정될 때만 채우기 (촉발 단위 백필)
- "모든 public 함수를 당장 인덱스화" 시도는 실패하거나 쓰레기 데이터를 남김

**Bypass Regression Cases 백필**:

- 도입 전 이미 관찰된 "에이전트가 자주 저지르는 실수"를 `AUTOMATION.md` Section 7에 최소 3건 등록
- 예: 과거 PR에서 테스트를 몰래 skip한 사례, 인덱스를 건너뛴 사례 등
- 이 백필이 있어야 도입 초기부터 회귀 방어가 작동

---

## 9. 셋업 완료 체크리스트

- [ ] 6개 문서가 프로젝트 루트에 존재
- [ ] `AGENT_MODE.md` Section 1·2·3 채움
- [ ] `AUTOMATION.md` Section 1·6·7 최소 채움, Section 9 `declared` 선언
- [ ] `CONTRIBUTING.md` Section 3.1·6.3 프로젝트화
- [ ] `.github/pull_request_template.md` 저장
- [ ] 최소 CI 체크 2개가 `warning` 단계로 작동
- [ ] 각 주요 디렉토리에 빈 `functions.index.md` 파일 생성
- [ ] 3개 시나리오(minimal/standard/extended) 중 1개 이상 실제 실행
- [ ] P1 승인자 지정 + 팀이 인지

이 중 80% 이상이 체크되면 셋업 완료. 나머지는 `checklist` 승격 단계에서 마무리.

---

## 10. 문서 업데이트 절차

이 `SETUP.md` 자체의 변경은 **P1 변경**이다(Meta-1). Change Boundary `extended` + 사람 승인 필요.

개선 제안은 다음 시나리오에서 기여한다:

- 실제 셋업 중 발견한 새로운 함정·실패 패턴 → Section 7·8 추가
- 새 도구·CI 시스템 지원 → Section 6 추가 예시
- 승격 기준의 현실성 문제 → Section 5 재조정

셋업 가이드는 **경험으로만 개선된다**. 상상으로 증보하지 말 것.

---

## 참고: 헌법에서 이 문서가 언급되는 조항

- Meta-1 (Constitution Amendment Procedure) — 이 문서 변경 절차의 근거
- 제11조 (Function Index as Agent Navigation Map) — Phase 5의 근거
- 제12조 (Index Updates Must Follow Code Changes) — Phase 5·6의 근거
- 제28·29조 (Change Boundary Block) — Phase 6 PR 템플릿
- Meta-3 (Operation Mode Declaration) — Phase 1의 근거
- Meta-4·5 (Automation) — Phase 3의 근거
- CONTRIBUTING.md Section 4 (Multi-Model Design Review) — 시나리오 C의 근거
