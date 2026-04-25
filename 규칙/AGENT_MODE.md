# AGENT_MODE.md — 프로젝트 운영 모드 선언

> 이 문서는 **프로젝트별 선언 파일**이다. 헌법(`CONSTITUTION.md`)을 바꾸지 않고, 이 프로젝트가 헌법을 **어떻게 해석·라우팅**할지를 선언한다.
>
> 에이전트(Claude·Codex 등)는 매 작업 시작 시 **이 파일을 가장 먼저 읽어야 한다.** 선언이 없거나 모호하면 보수적으로 `extended + human-approval` 경로를 따른다.
>
> 이 파일의 변경은 Meta-1에 따라 **P1 변경**으로 취급된다.

---

## 1. Project Identity

- **project_name**: `<프로젝트명>`
- **repository**: `<repo URL 또는 경로>`
- **constitution_version**: `v1.0 (2026-04-24)` — 이 선언이 따르는 헌법 버전
- **owner**: `<의사결정 owner 또는 팀>`
- **last_updated**: `<YYYY-MM-DD>`

---

## 2. Operation Mode

다음 중 하나를 선언한다. 기본 권장은 `dual-agent`.

- [ ] `dual-agent` — Claude + Codex 공존 (표준 권장)
- [ ] `claude-only`
- [ ] `codex-only`
- [ ] `multi-claude` — 복수 Claude 세션을 서로 다른 역할 프롬프트로 분리
- [ ] `multi-model` — Claude + GPT + Gemini 등 이기종 혼합
- [ ] `human-only` — 에이전트 없이 사람만 개발

### 2.1 Actor 배정

헌법 본문 제26조의 "Claude"·"Codex"는 각 모드에서 다음 일반 역할의 인스턴스로 해석된다.

| Mode | Design/Review Actor | Implementation Actor | Final Review Authority |
| --- | --- | --- | --- |
| dual-agent | Claude | Codex | Claude + Human (P1/extended 시) |
| claude-only | Claude | Claude or Human | Human + Automated Verification |
| codex-only | Human | Codex | Human + Automated Verification |
| multi-claude | Claude-A (review 프롬프트) | Claude-B (impl 프롬프트) | Human + Automated Verification |
| multi-model | Claude 또는 Other | Other 또는 Claude | Human + Automated Verification |
| human-only | Human | Human | Human + Automated Verification |

**주의**: `multi-claude`에서 세션 분리만으로 독립 리뷰로 간주하지 않는다(Meta-3, 운영 원칙 2). 역할·컨텍스트·검토 기준이 **명시적으로 분리**되어야 한다.

---

## 3. Routing Policy

작업의 Handoff Level(제29조)별 협업 경로. 본 프로젝트의 실제 운영값을 채운다.

### 3.1 `minimal` — 문서/오타/주석/포맷

- **implementation**: `<actor>`
- **review**: `<actor 또는 "automated-only">`
- **validation**: `<예: lint, spellcheck, 링크 체크>`
- **required Change Boundary level**: `minimal`

### 3.2 `standard` — 일반 코드 변경

- **implementation**: `<actor>`
- **review**: `<actor — 반드시 implementation actor와 달라야 함>`
- **validation**: `<예: unit + contract tests>`
- **required Change Boundary level**: `standard`

### 3.3 `extended` — P1/권한/외부 I/O/DB/정책/보안/마이그레이션

- **design**: `<actor>`
- **implementation**: `<actor>`
- **review**: `<actor>`
- **human approval**: `<approver role>`
- **validation**: `<예: unit + contract + policy + eval + recovery>`
- **required Change Boundary level**: `extended`

---

## 4. Single-Agent Rules

단일 에이전트 모드(`claude-only`/`codex-only`)일 때만 채운다. 그 외에는 "N/A".

- **session separation**: `<예: 구현은 session A, 리뷰는 session B — 다른 시스템 프롬프트 + 비어있는 컨텍스트로 시작>`
- **required automated tests**: `<예: Policy Test(제38조) 필수, Agent Behavior Eval(제39조) 필수, Recovery Test(제40조) 필수>`
- **human review requirement**: `<예: 모든 standard/extended 변경>`
- **prohibited self-approval**: `<반드시 명시 — 예: 같은 세션의 같은 Actor가 구현·리뷰 동시 서명 금지>`

---

## 5. Approval Gates (제45조)

본 프로젝트의 human approval 게이트 실제 위치·담당자.

- **P1 changes**: `<approver>`
- **policy engine rule changes**: `<approver>`
- **migration execution**: `<approver>`
- **agent scope_ceiling expansion**: `<approver>`
- **long-term memory schema changes**: `<approver>`

승인 기록 포맷은 제46조(Approval Binds to the Change Set)에 따른다.

---

## 6. Dispatcher Maturity

자동화는 다음 3단계로 점진 도입한다. 현재 단계 하나를 선택한다.

- [ ] `declared` — AGENT_MODE.md 선언만 존재. 라우팅은 수동.
- [ ] `checklist` — PR 템플릿·pre-commit 체크리스트가 Handoff Level 판정을 보조.
- [ ] `thin-dispatcher` — 파일 경로·diff 유형·키워드·테스트 영향·권한/DB/외부 I/O 변경 여부로 `handoff_level`과 담당 Actor를 **추천**하는 얇은 도구 존재. **자동 실행 금지, 추천 + 이유만 제공.**

자동 디스패처는 어떤 단계에서도 정책 게이트·승인 게이트·제27조·제35조를 우회할 수 없다(Meta-3 운영 원칙 3).

---

## 7. Exceptions

예외 발생 시 기록해야 할 포맷.

**허용 예외 예시**:

- `single-agent exception` (제26조) — 사용자가 Dual-Agent 모드에서 명시적으로 단독 처리를 지시한 경우
- `large-diff exception` (제32조) — generated code, lockfile, migration output 등
- `skip/xfail exception` (제43조) — 외부 의존·환경 문제·flaky 임시 격리

**필수 기록**:

```
Exception Record:
- type: <single-agent | large-diff | skip-xfail | other>
- scope: <대상 PR/커밋/테스트>
- reason: <사유>
- approver: <승인자 — single-agent는 user_direct, large-diff는 리뷰어>
- expiry or review condition: <언제 재검토>
- tracking issue: <링크>
```

예외는 Change Boundary Block에 함께 기록되어야 한다(제28·29조).

---

## 8. 변경 관리

이 파일의 변경은 **P1 변경**이다(Meta-1). 변경 시:

- Change Boundary `extended` 형식으로 의도·영향 조항·위험·검증 방법·승인자 기록
- 헌법 본문 48조를 변경하는 것은 이 파일로 우회할 수 없다(Meta-3).
- Operation Mode 전환(예: `dual-agent` → `claude-only`)은 특히 제27조의 충족 경로가 바뀌므로, 전환 즉시 Section 4(Single-Agent Rules)를 완성해야 한다.

---

## 참고: 헌법 본문에서 이 파일이 언급되는 조항

- Meta-3 (헌법 해석의 전제)
- 제26조 (Role Separation by Default) — Actor 이름의 추상화 근거
- 제27조 (No Self-Review as Final Review) — Single-Agent Mode에서 재해석
- 제29조 (Handoff Metadata Scales with Risk) — Routing Policy의 근거
- 제45조 (Human Approval Gates) — Section 5의 근거
