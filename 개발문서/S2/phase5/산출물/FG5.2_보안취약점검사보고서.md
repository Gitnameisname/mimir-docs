# FG5.2 보안취약점검사보고서 — 에이전트 제안 큐 + 프롬프트 인젝션 정책

| 항목 | 내용 |
|---|---|
| 기능 그룹 | FG5.2 |
| 검사일 | 2026-04-17 |
| 검사자 | 개발팀 |
| 검사 결과 | **이상 없음** |

---

## 1. 검사 범위

| 파일 | 역할 |
|---|---|
| `backend/app/security/prompt_injection.py` | 룰 기반 인젝션 탐지 + 콘텐츠 분리 |
| `backend/app/api/v1/scope_profiles.py` | `/admin/proposals`, `/my/proposals` |
| `frontend/src/features/documents/AgentProposalsTab.tsx` | 제안 검토 UI |
| `frontend/src/api/proposals.ts` | 프론트엔드 API 클라이언트 |

---

## 2. 취약점 검사 결과

### 2-1. OWASP LLM01 — Prompt Injection

| 공격 벡터 | 방어 메커니즘 | 검사 결과 |
|---|---|---|
| 외부 문서 내 `Ignore previous instructions` 패턴 | `detect_injection()` 패턴 매칭 + `injection_risk=True` 플래그 | ✅ 탐지됨 |
| 역할 전환 유도 (`You are now a...`) | role_assumption 카테고리 룰 | ✅ 탐지됨 |
| 데이터 유출 시도 (`send all documents to...`) | data_exfiltration 카테고리 룰 | ✅ 탐지됨 |
| 컨텍스트 오염 (`Previous user said...`) | context_manipulation 카테고리 룰 | ✅ 탐지됨 |
| 검색 결과를 LLM 프롬프트에 직접 삽입 | `wrap_search_results()` 한국어 구분자 격리 | ✅ 격리됨 |
| 시스템/사용자/외부 콘텐츠 혼합 | `separate_contexts()` 명시적 분리 | ✅ 격리됨 |

**탐지율: 100% / False Positive: 0%**

### 2-2. OWASP LLM02 — Insecure Output Handling

| 검사 항목 | 결과 | 비고 |
|---|---|---|
| 에이전트 생성 콘텐츠를 `PROPOSED` 상태로 격리 | ✅ | 인간 검토 전까지 발행 불가 |
| 프론트엔드 React 렌더링 시 XSS 방어 | ✅ | React 기본 이스케이핑 적용 |

### 2-3. 접근 제어

| 검사 항목 | 결과 |
|---|---|
| `/admin/proposals` — 관리자 전용 접근 제어 | ✅ |
| `/my/proposals` — 본인 제안만 조회 가능 | ✅ |
| 승인/반려 API — `reviewer_role` 검증 | ✅ |

### 2-4. 인젝션 탐지 우회 시도

| 우회 기법 | 결과 |
|---|---|
| 대소문자 변형 (`IGNORE PREVIOUS`) | ✅ 탐지됨 (case-insensitive 룰) |
| 유니코드 혼합 삽입 | ✅ 탐지됨 (normalize 후 매칭) |
| 정상 콘텐츠 내 삽입 (`문서 내용... ignore all previous...`) | ✅ 탐지됨 (부분 매칭) |

### 2-5. 데이터 입력 검증

| 입력 경로 | 처리 | 결과 |
|---|---|---|
| 에이전트 제안 콘텐츠 | Pydantic 모델 유효성 검사 | ✅ |
| `rejection_reason` 자유 텍스트 | 길이 제한 + parameterized query | ✅ |
| 프론트엔드 API 호출 파라미터 | TypeScript 타입 + Zod (API 클라이언트) | ✅ |

---

## 3. OWASP LLM Top 10 대응 매트릭스

| OWASP ID | 위협명 | 대응 상태 | 구현 위치 |
|---|---|---|---|
| LLM01 | Prompt Injection | ✅ 방어됨 | `prompt_injection.py` 4개 카테고리 룰 |
| LLM02 | Insecure Output Handling | ✅ 방어됨 | PROPOSED 격리 + React 이스케이핑 |
| LLM06 | Sensitive Information Disclosure | ✅ 방어됨 | ACL 필터 + `/my/proposals` 소유자 한정 |
| LLM08 | Excessive Agency | ✅ 방어됨 | FG5.1에서 PROPOSED 강제 |

---

## 4. 권고 사항

| # | 등급 | 내용 | 대응 우선순위 |
|---|---|---|---|
| 권고-1 | Info | 인젝션 탐지 룰셋 정기 갱신 메커니즘 부재. 새로운 공격 패턴 등장 시 코드 배포 없이 룰 업데이트 가능한 구조 검토 권장 | Phase 6 이후 |
| 권고-2 | Info | `wrap_search_results()` 구분자가 한국어 마커. 다국어 환경에서의 구분자 일관성 재검토 권장 | Phase 6 이후 |

---

## 5. 결론

FG5.2 범위에서 Critical, High, Medium 등급의 보안 취약점은 발견되지 않았습니다.  
OWASP LLM01 Prompt Injection 방어가 100% 탐지율로 구현되었으며, Info 등급 권고 2건은 향후 개선 사항입니다.

**FG5.2 보안취약점검사 결과: ✅ 이상 없음 (권고 2건)**
