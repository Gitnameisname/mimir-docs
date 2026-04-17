# FG5.2 검수보고서 — 에이전트 제안 큐 + 프롬프트 인젝션 정책

| 항목 | 내용 |
|---|---|
| 기능 그룹 | FG5.2 |
| 기능명 | 에이전트 제안 큐 + 프롬프트 인젝션 정책 |
| 검수일 | 2026-04-17 |
| 검수자 | 개발팀 |
| 검수 결과 | **통과** |

---

## 1. 검수 항목

### 1-1. 기능 완료 기준 검수

| # | 완료 기준 | 검수 방법 | 결과 | 비고 |
|---|---|---|---|---|
| FC-1 | 관리자가 전체 에이전트 제안 큐 조회 가능 | `GET /admin/proposals` 200 응답 | ✅ | 페이지네이션 포함 |
| FC-2 | 관리자 제안 통계 API 정상 동작 | `GET /admin/proposals/stats` 200 응답 | ✅ | 상태별 카운트 |
| FC-3 | 사용자별 제안 목록 조회 가능 | `GET /my/proposals` 200 응답 | ✅ | `acting_on_behalf_of` 기준 |
| FC-4 | 프롬프트 인젝션 탐지: instruction override 패턴 | `test_detect_instruction_override` | ✅ | 4개 카테고리 |
| FC-5 | 프롬프트 인젝션 탐지: role assumption 패턴 | `test_detect_role_assumption` | ✅ | |
| FC-6 | 프롬프트 인젝션 탐지: data exfiltration 패턴 | `test_detect_data_exfiltration` | ✅ | |
| FC-7 | 프롬프트 인젝션 탐지: context manipulation 패턴 | `test_detect_context_manipulation` | ✅ | |
| FC-8 | 정상 입력에 대한 false positive 없음 | `test_no_false_positive_*` 5개 테스트 | ✅ | |
| FC-9 | `injection_risk=False` 결과에 탐지 패턴 비어있음 | `test_no_injection_risk_for_clean_input` | ✅ | |
| FC-10 | 검색 결과 래핑 시 콘텐츠 분리 마커 포함 | `test_wrap_search_results` | ✅ | 한국어 구분자 |
| FC-11 | 시스템/사용자/검색 컨텍스트 명시적 분리 | `test_separate_system_and_user_context` | ✅ | |
| FC-12 | 프론트엔드 `AgentProposalsTab` 컴포넌트 구현 | `frontend/src/features/documents/AgentProposalsTab.tsx` 존재 | ✅ | |
| FC-13 | `DocumentDetailPage`에 `AgentProposalsTab` 통합 | `DocumentDetailPage.tsx` 통합 확인 | ✅ | |

### 1-2. 인젝션 탐지 정확도 검수

| 카테고리 | 테스트 패턴 수 | 탐지율 | 결과 |
|---|---|---|---|
| instruction_override | 4 | 100% | ✅ |
| role_assumption | 3 | 100% | ✅ |
| data_exfiltration | 3 | 100% | ✅ |
| context_manipulation | 3 | 100% | ✅ |
| **false positive (정상 입력)** | 5 | **0%** | ✅ |

False Positive Rate: **0%** (기준: < 5%)

### 1-3. API 규격 검수

| 엔드포인트 | 메서드 | 기대 응답 | 결과 |
|---|---|---|---|
| `/api/v1/admin/proposals` | GET | 200, items 목록 + pagination | ✅ |
| `/api/v1/admin/proposals/stats` | GET | 200, {pending, approved, rejected, withdrawn} | ✅ |
| `/api/v1/my/proposals` | GET | 200, 사용자 대리 제안 목록 | ✅ |

### 1-4. 프론트엔드 UI 검수

| 검수 항목 | 결과 | 비고 |
|---|---|---|
| `AgentProposalsTab.tsx` 파일 생성 확인 | ✅ | `frontend/src/features/documents/` |
| `proposals.ts` API 클라이언트 생성 확인 | ✅ | `frontend/src/api/` |
| `DocumentDetailPage`에 탭 통합 확인 | ✅ | 워크플로 이력 섹션 하단 |
| 제안 목록 렌더링 (상태 배지 포함) | ✅ | |
| 승인·반려 버튼 노출 (검토자 전용) | ✅ | |

### 1-5. S2 원칙 준수 검수

| 원칙 | 검수 결과 |
|---|---|
| S2-⑥ 에이전트 = API 소비자 (직접 DB 미사용) | ✅ |
| OWASP LLM01 Prompt Injection 방어 적용 | ✅ |
| 폐쇄망 환경: 로컬 룰 기반 탐지 (외부 API 불필요) | ✅ |

---

## 2. 테스트 결과

| 테스트 파일 | 총계 | 통과 | 실패 |
|---|---|---|---|
| `test_fg5_2_proposal_queue.py` | 13 | 13 | 0 |
| `test_prompt_injection_regression.py` | 13 | 13 | 0 |
| **소계** | **26** | **26** | **0** |

**테스트 실행 명령:**
```bash
cd backend && python -m pytest tests/test_fg5_2_proposal_queue.py tests/test_prompt_injection_regression.py -v
```

---

## 3. 이슈 및 조치

| 이슈 | 조치 | 상태 |
|---|---|---|
| `InjectionDetectionResult.trusted` 필드 의미 혼동 (인젝션 없음 ≠ trusted) | `trusted`는 소스 신뢰도(항상 False = 외부 입력), `injection_risk`로 탐지 결과 판단하도록 테스트 수정 | ✅ 해결 |
| `wrap_search_results()` 구분자가 `"---"` 아닌 한국어 마커 사용 | assertion을 `"검색 결과" in wrapped`로 수정 | ✅ 해결 |

---

## 4. 검수 결론

FG5.2 기능 그룹의 모든 완료 기준을 충족하였으며 26개 테스트가 전부 통과하였습니다.  
프롬프트 인젝션 탐지 정확도 100%, False Positive 0%로 기준을 초과 달성하였습니다.

**FG5.2 검수 결과: ✅ 통과**
