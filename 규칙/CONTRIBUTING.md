# CONTRIBUTING.md — Ready for AI SW 프로젝트 기여 지침

> 이 문서는 **Ready for AI SW 헌법을 채택한 프로젝트의 일상 개발 루틴**을 정의한다.
> 헌법(`CONSTITUTION.md`)의 규범 원칙을 실제 기여·리뷰·회의·커밋 흐름에 어떻게 녹일지 구체화한다.
>
> 이 문서는 **템플릿**이다. 각 프로젝트는 자기 사정에 맞게 수정한다. 단, 헌법 조항과 충돌할 수 없다(Meta-2).

---

## 0. 문서 위치와 읽기 순서

- `AGENT_MODE.md` — 프로젝트의 운영 모드 (Meta-3)
- `AUTOMATION.md` — CI·디스패처·오케스트레이터 운영 (Meta-4·5)
- `CLAUDE.md` / `AGENTS.md` — 역할별 에이전트 운영 지침
- `CONSTITUTION.md` — 규범 원천
- **이 `CONTRIBUTING.md`** — 일상 기여 루틴
- 각 디렉토리의 `functions.index.md` — 함수 탐색

---

## 1. 기여 전 체크리스트

처음 기여하거나 새 기능·리팩토링을 시작하기 전에 확인한다.

- [ ] 헌법 본문 48조 + Meta-1~5 한 번 숙지
- [ ] 이 프로젝트의 `AGENT_MODE.md`를 읽어 본인의 Actor 역할 파악
- [ ] `AUTOMATION.md` Section 2(CI 체크 목록)와 Section 6(Failure Modes) 확인
- [ ] 해당 디렉토리의 `functions.index.md` 먼저 읽기 (제11조) — **이미 존재하는 함수를 재발명하지 않는다**
- [ ] 작업의 Handoff Level(minimal/standard/extended) 판정 (제29조)

---

## 2. Change Boundary Block 작성법 (제28·29조 실무)

모든 인수인계·리뷰 요청·구현 완료 보고에 필수.

### 2.1 Level 판정 기준

- **minimal**: 오타, 주석, 문서 한 줄, 포맷 변경
- **standard**: 함수 변경, 테스트 변경, 일반 코드 수정
- **extended**: 아키텍처·권한·DB·외부 I/O·정책·보안·마이그레이션·P1 변경

**애매하면 한 단계 위로 올린다.** (AGENTS.md 루틴 규칙)

### 2.2 표준 포맷 (standard 예시)

```
Change Boundary:
- intent: auth.verify_token이 만료 토큰에서 safe_retry=false를 반환하지 않던 버그 수정
- touched files: src/auth/jwt.py, src/auth/functions.index.md, tests/auth/test_verify_token.py
- changed functions: auth.verify_token
- behavior changes: 만료 시 safe_retry=false로 변경, required_scope 필드 생략
- tests added/updated: tests/auth/test_verify_token.py::test_expired_token_error_shape
- validation performed: pytest tests/auth/test_verify_token.py passed (5/5)
- risks: 기존에 safe_retry=true로 재시도하던 클라이언트 영향 조사 필요
- open questions: 재시도 클라이언트 영향 조사는 Design/Review Actor에게 이관
```

### 2.3 자주 빠뜨리는 필드

- `validation performed` — "테스트 추가"와 "실제 실행"은 다르다. 실행 로그·통과 수를 남긴다.
- `unchanged by design` — "건드리지 않기로 한 인접 영역"을 명시해야 리뷰어가 범위를 좁힐 수 있다.
- `open questions` — 다음 에이전트나 사람이 결정해야 할 것. 빈 필드는 허용이 아니라 "없음"으로 명시.

---

## 3. `functions.index.md` 관리 (제11·12조 실무)

### 3.1 본 프로젝트의 "주요 패키지·모듈" 범위

**이 프로젝트는 다음 경로를 인덱스 필수 대상으로 정의한다:**

- `<src/ 하위 1~2 depth 예시>` — 각 팀이 실제 경로로 채움
- `<tools/>`, `<policy/>`, `<auth/>` 등 민감 경로는 모두 포함

### 3.2 Required vs Optional

**Required (제11조)**:

- public/exported functions
- `@tool` / `@effectful` / `@external_io` / `@policy` 어노테이션 함수
- service-layer entrypoints
- repository/database mutation functions
- external I/O functions
- policy/security/permission functions
- 에이전트가 직접 호출하는 함수 (orchestration flow)

**Optional**:

- small private helpers
- local pure transformation functions
- test-only utilities

### 3.3 갱신 3단계 (제12조)

1. **PR 체크리스트 단계** — 템플릿의 "functions.index.md 갱신 완료" 체크
2. **pre-commit hook / CI soft check** — 경고만
3. **CI hard fail** — 인덱스 누락·불일치는 머지 차단

현재 단계: `AUTOMATION.md` Section 9(Automation Maturity Stage)와 일치.

### 3.4 작성 포맷

```
# functions.index.md
- `auth.verify_token(token: str) -> User`
  - purpose: JWT 검증
  - effects: none
  - errors: `TOKEN_EXPIRED`, `TOKEN_INVALID`
  - tests: tests/auth/test_verify_token.py
- `auth.issue_token(user: User, scope: list[str]) -> str`
  - purpose: 권한 범위를 포함한 JWT 발급
  - effects: writes `audit_log`
  - errors: `INVALID_SCOPE`
  - tests: tests/auth/test_issue_token.py
```

한 함수당 3~5줄 이내. 긴 설명은 docstring에 둔다.

---

## 4. Multi-Model Design Review as Practice

설계·원칙·아키텍처·방법론 수준의 과제에서는 단일 에이전트·단일 리뷰어의 블라인드스팟을 줄이기 위해 **크로스-모델 회의**를 기본 프로세스로 사용한다. 이 절은 헌법 제27조(No Self-Review as Final Review)와 Meta-3 `multi-model` 모드를 실제 워크플로로 구체화한 것이다.

본 헌법 자체도 이 방식으로 수립됐다 — Claude(Framer) + ChatGPT(Refiner) + 사용자(Topic Owner, Checkpoint Reviewer)의 8턴 회의로 Meta-1~5 + 48조가 완성됐다. 본 섹션은 그 경험을 재현 가능한 절차로 박제한 것이다.

### 4.1 언제 쓰는가 / 언제 쓰지 않는가

**권장 대상 (extended 성격)**:

- 아키텍처 결정, ADR 작성
- 새 원칙·정책 수립 (이 프로젝트의 헌법 자체가 예시)
- 보안 모델·권한 설계
- 대규모 리팩토링 계획
- 외부 API 계약 설계
- 헌법·역할 파일·AUTOMATION.md 개정

**비권장 대상 (minimal·standard 성격)**:

- 단일 함수 수정
- 즉답이 가능한 질문
- 명확한 요구사항의 기계적 구현
- 오타·포맷 수정

규모 판단은 Handoff Level(제29조)과 일치시킨다. `extended`면 크로스-모델 회의를 기본, `standard`는 선택, `minimal`은 불필요.

### 4.2 4단계 구조

**Phase 1 — Topic Seed (사람)**

사용자가 대 주제를 선언한다. 범위·목표·다룰 영역 후보·결과물 형식을 한 번에 명시. 예: "Ready for AI SW 개발 헌법을 수립하자. CLAUDE.md / AGENTS.md에서 참조할 것. 5개 영역 다루기."

애매한 시드는 Phase 2를 흐트러뜨린다. 시드 수준의 질문은 이 단계에서 AI 듀오가 사용자에게 **되물어** 범위를 고정한다.

**Phase 2 — AI Deliberation (듀오)**

두 LLM이 구조화된 회의를 진행. 본 프로젝트 8턴에서 검증된 턴 규칙:

- 한 턴에 **핵심 논점 2~3개**로 제한 — 많으면 답변이 얇아짐
- 상대 질문에 먼저 답한 후 새 논점 제시
- 턴 말미에 **짧은 합의 초안**을 Markdown 조항 형식으로 고정
- 합의된 조항에는 **번호 부여** — 후속 턴에서 "제X조 확장" 같은 참조가 가능해야 함
- 턴 말미에 **"다음 턴 주제 한 줄"** 제안

한 턴이 끝날 때마다 합의가 **누적되는 조항 리스트**로 쌓인다. 대화가 흘러가는 대신 구조물로 축적된다.

**Phase 3 — Checkpoint Review (사람)**

사용자가 중간 결과를 검수. 의문·실무적 간극·놓친 축이 있으면 **개입한다**. 개입은 예외가 아니라 설계상의 필수 축이다.

본 헌법 수립 중 사용자 개입이 실제로 방향을 바꾼 지점들:

- "Claude와 Codex의 중심 부분이 왜 다른가?" → Design/Review Actor vs Implementation Actor 추상화 (Meta-3)
- "하나만 사용할 경우도 고려해야 한다" → Operation Mode 6종으로 확장
- "자동화를 어떻게 할 수 있을까?" → 3레이어 모델 + Meta-4·5 신설

이런 개입이 없으면 AI 듀오만으로는 "그럴듯한 완결"에 도달해서 멈춰버린다. **사용자의 질문이 유일하게 공유 블라인드스팟을 깬다.**

**Phase 4 — Refined Iteration**

사용자가 재정의한 방향으로 Phase 2 재개. 합의가 수렴하면 산출물 문서로 고정하고 P1 승인 절차에 올린다(제33·45조).

### 4.3 역할 배분

| 역할 | 담당 | 주 책임 |
| --- | --- | --- |
| **Topic Owner** | 사용자 | 대 주제 설정, 방향 재조정, 최종 승인 |
| **Framer** | LLM-A (Claude 기본) | 초안·논점 구조 제시, 조항 번호 부여, 문서 작성 |
| **Refiner** | LLM-B (ChatGPT 기본) | 반박·보강·엣지 케이스 제시, 추가 축 제안 |
| **Scribe** | 둘 중 1명 (주로 Framer) | 매 턴 합의 초안 Markdown 고정 |

역할은 턴마다 고정되지 않는다 — Framer가 특정 턴에서 Refiner 역할을 할 수 있다. 다만 **구현과 최종 리뷰를 동시에 수행하지 않는다**는 제27조는 설계 회의에서도 유지. Framer가 ADR 초안을 쓰면 Refiner가 리뷰 주체.

### 4.4 Disagreement 처리 (제34조 실무)

두 모델 간 의견이 합치되지 않으면 **즉시 결론 내리지 않고** Disagreement Record를 남긴다. 사용자가 Phase 3에서 이 레코드를 보고 판정한다.

레코드가 없는 암묵적 합의는 위험 신호다. 본 헌법 수립 과정에서도 암묵 수렴 지점이 있었고(예: Claude-only 모드에서 세션 분리만으로 제27조를 충족하는가 — 결국 ChatGPT의 "부족하다"로 수렴), 돌아보니 명시 레코드로 남겼어야 하는 것들이 있었다.

포맷은 제34조 표준을 따른다. 기록 위치는 `docs/meetings/<date>-<topic>.md` 또는 `docs/disagreements/`.

### 4.5 회의 산출물 관리

- **회의 트랜스크립트** — `docs/meetings/<YYYY-MM-DD>-<topic>.md` (선택, extended 과제는 권장)
- **합의된 조항** — CONSTITUTION·ADR·설계 문서에 즉시 반영
- **생성된 Disagreement Record** — Change Boundary에 첨부 또는 별도 디렉토리
- **사용자 개입 기록** — Phase 3 결정 내용을 메모. "왜 방향을 바꿨는가"가 후속 리뷰의 맥락이 됨

### 4.6 한계와 주의

- **공유 블라인드스팟**: 두 LLM은 훈련 데이터·시대적 편향이 겹치는 영역에서 "둘 다 동의 = 정답"이라는 착각을 만들 수 있다. Phase 3의 사용자 개입이 이를 깨는 유일한 장치.
- **비용과 시간**: 8턴 회의는 누적 토큰·실시간 소요가 크다. `extended` 규모가 아닌 과제에 적용하면 오버헤드. 제29조 Handoff Level과 항상 일치시킬 것.
- **자동화 금지**: 이 프로세스는 오케스트레이터(`AUTOMATION.md` Layer 3)가 자동으로 돌리면 안 된다. 사용자 개입 축이 설계상 필수이므로, 자동화는 "회의 기록·Disagreement 형식 강제" 수준에서만 허용(Meta-5 운영 원칙 3).
- **단일 모델 환경**: Claude·ChatGPT 중 하나만 가용한 환경에서는 동일 모델의 **다른 세션** 간 회의로 대체 가능하나, Meta-3 운영 원칙 2에 따라 독립 리뷰로 인정하지 않는다. 사람 리뷰 + 자동 검증이 추가 필요.
- **AI의 "서로 맞장구"**: 두 LLM은 공손·협조적 경향이 강해 반박이 약해질 수 있다. 의도적으로 Refiner 프롬프트에 "반박 우선, 합의 보류" 기조를 넣는 편이 좋다.

---

## 5. Disagreement Record 작성 (제34조 실무)

형식은 헌법 제34조 표준. 본 프로젝트에서는 다음 경우 **반드시** 작성한다:

- 두 에이전트의 설계 판단이 갈릴 때
- 에이전트 판단이 기존 ADR·테스트·현재 동작과 충돌할 때
- 사용자 지시가 헌법·보안 정책과 충돌할 때 (이 경우 헌법 우선, 제35조)
- `AUTOMATION.md` Dispatcher 추천을 거부할 때

기록 장소: `docs/disagreements/<YYYY-MM-DD>-<topic>.md` 또는 관련 PR·이슈에 첨부.

---

## 6. Commit / PR 규약 (제30~33조 실무)

### 6.1 Commit

- 한 커밋 = 한 의도 (제30조). 포맷·리팩토링·기능을 섞지 않는다.
- 메시지 첫 줄: "왜" 바꿨는지 (제31조). 예: `fix(auth): return safe_retry=false on expired tokens` (what) 가 아니라 `fix(auth): 만료 토큰 재시도로 인한 과부하 차단` (why) 지향.
- 인덱스·테스트·코드를 같은 커밋에 포함 (제12·15조).

### 6.2 PR

- 제목에 Handoff Level 또는 `[P1]` 태그 명시 (제33조)
- 본문에 Change Boundary Block 포함 (제28·29조)
- 500줄 이상 → 분할 검토 (제32조). 1000줄 이상 → 예외 사유 필수.
- Large Diff 예외: generated code, lockfile, formatting-only, mechanical rename, schema migration output, vendored snapshot — **Change Boundary에 사유 명시.**

### 6.3 PR 템플릿 (`.github/pull_request_template.md` 권장)

```markdown
## Change Boundary
<!-- minimal / standard / extended 중 하나 -->
- intent:
- touched files:
- changed functions:
- unchanged by design:
- behavior changes:
- tests added/updated:
- validation performed:
- risks:
- open questions:

## Handoff Level
- [ ] minimal
- [ ] standard
- [ ] extended — `[P1]` 태그를 제목에 추가하셨습니까?

## Constitution Checklist
- [ ] `functions.index.md` 갱신 완료 (제12조)
- [ ] 관련 테스트 추가/갱신 및 `validation performed` 기록 (제15·36~40조)
- [ ] Dispatcher 추천과 일치 또는 Disagreement Record 첨부 (Meta-3)
- [ ] `AUTOMATION.md` Failure Modes에 등재된 우회 패턴 미사용 (Meta-4)
```

---

## 7. 테스트 계층별 작성 가이드 (제36~40조 실무)

| 계층 | 무엇을 검증 | 어느 위험도에서 필수 |
| --- | --- | --- |
| Unit (제36) | 순수 함수, 변환, 계산 | 모든 변경 |
| Contract (제37) | tool/function/error/API 스키마 | tool·API·계약 변경 |
| Policy (제38) | 권한·승인·scope intersection·P0/P1/P2 실행 경로 | 권한·정책·scope·민감정보 변경 |
| Agent Behavior Eval (제39) | 프롬프트 인젝션, trust level 준수, refusal | 에이전트 행동 범위 변경 |
| Recovery / E2E (제40) | 실패·재시도·승인 대기·롤백 | extended 변경 전반 |

**Single-Agent Mode(Meta-3)에서는 Policy·Eval·Recovery 계층을 필수로 포함한다** — 교차 에이전트 리뷰의 공백을 자동 검증이 메운다.

---

## 8. 변경 관리

- 이 문서의 변경은 **P1 변경**이다(Meta-1).
- 헌법 본문이나 Meta 조항을 바꾸는 것은 이 문서로 **우회할 수 없다**. 헌법 변경은 별도 P1 절차.
- Section 3.1("주요 패키지·모듈" 범위)과 Section 6.3(PR 템플릿)은 프로젝트별로 수정 가능. 단, 제11·28·29조와 충돌할 수 없다.

---

## 참고: 헌법 본문에서 이 문서가 언급되는 조항

- 제11조 (Function Index as Agent Navigation Map) — Section 3.1 범위 정의 위임
- 제12조 (Index Updates Must Follow Code Changes) — Section 3.3 3단계 강제
- Meta-3 Operation Mode — Section 4 Multi-Model Review의 근거 모드
- 제27조 (No Self-Review as Final Review) — Section 4.3 역할 배분의 근거
- 제34조 (Disagreement Protocol) — Section 4.4·5의 근거
- 제28~33조 — Section 2·6의 근거
- Meta-4·5 (Automation) — Section 6.3 체크리스트·Section 7 테스트 계층과 연결
