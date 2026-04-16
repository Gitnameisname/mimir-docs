# Phase 0 개발계획서
## S2 플랫폼 원칙 수립 + S1 부채 청산

---

## 1. 단계 개요

### 단계명
Phase 0. S2 플랫폼 원칙 수립 + S1 부채 청산

### 목적
S2 설계 기준을 확정하고, S1에서 이월된 기술 부채를 완전히 해소하여, Phase 1 이후 모든 개발의 토대를 마련한다. 특히 사용자 경험의 핵심 경로(로그아웃, 설정, 관리자 접근)가 현재 완전히 부재하므로, 이를 긴급 복구하는 것이 이 Phase의 최우선이다.

### 선행 조건
- S1 완수 보고서 및 회고 검토 완료
- Season 2 개발계획 승인 및 기본 원칙(①~⑦) 합의
- 프로젝트 팀의 S2 개발 규약 이해도 확보

### 기대 결과
- S2 추가 원칙(⑤⑥⑦) 및 설계 결정(3.1~3.6) 공식 문서화
- AI품질평가.md 산출물 규약 확정
- Feature Group 소분 운영 규약 정의
- 사용자/관리자 네비게이션 정상 작동
- S1 부채 0건 (Fresh-boot 환경 검증)
- Phase 1 착수 가능 상태

---

## 2. Phase 0의 역할

Phase 0은 "S2를 위한 기초 설정" Phase이다. 다음을 담당한다:

1. **설계 기준 수립**: S1의 4대 원칙에 S2 추가 원칙(AI-as-first-class-consumer, Scope generic, 폐쇄망 동등성)을 추가 정의
2. **산출물 규약 확정**: 기존 "검수보고서 + 보안취약점검사보고서"에 "AI품질평가.md"를 의무 산출물로 추가. 평가 항목, 기준값, 보고서 포맷을 명시
3. **S1 잔존 부채 해소**: 특히 사용자 메뉴(로그아웃, 설정) 및 Admin 네비게이션의 완전한 부재를 긴급 복구
4. **기술 부채 청산**: Fresh-boot CI, capabilities endpoint, seed_users 가드 등 선결 기술 이슈 해결

Phase 1 이후는 이 Phase에서 확립된 원칙과 규약에 따라 순항한다.

---

## 3. Feature Group 상세

### FG0.1 — S2 원칙 및 규약 확정

#### 목적
S2 설계의 기본 원칙과 운영 규약을 문서화하여 모든 팀원이 동일한 기준으로 개발할 수 있도록 한다.

#### 주요 작업 항목
1. S2 추가 원칙(⑤⑥⑦) 공식 승인 및 CLAUDE.md §2에 추가
   - 원칙 ⑤ "AI는 1급 사용자다"의 함의 명시
   - 원칙 ⑥ "접근 범위도 하드코딩 금지" 규칙 추가
   - 원칙 ⑦ "폐쇄망 동등성"의 구현 기준 정의
2. 핵심 설계 결정(3.1~3.6) 상세 검토 및 확정
   - OAuth client-credentials 인증 방식 승인
   - Scope Profile 모델 최종 검토
   - MCP 스펙 2025-11-25 핀 확정
   - Citation 5-tuple 계약 검수
   - Agent Principal 모델 최종 정의
   - 평가용 판정 전략 승인
3. AI품질평가.md 산출물 규약 확정
   - 평가 항목 정의 (Faithfulness, Answer Relevance, Context Precision, Citation-present rate 등)
   - 기준값(threshold) 설정 (Faithfulness ≥0.80, Answer Relevance ≥0.75, Citation-present ≥0.90)
   - 보고서 포맷 정의 (항목, 실측값, 기준값, 통과/미통과, 개선 항목)
   - AI품질평가.md 템플릿 작성 및 공유
4. Feature Group 소분 운영 규약 확정
   - FG 단위 독립 검수 게이트 규칙 정의
   - FG별 의무 산출물 목록 확정 (작업지시서, 검수보고서, 보안취약점검사보고서, AI품질평가보고서)
   - FG 완료 기준 정의
5. 새로운 CLAUDE.md 절대 규칙 추가
   - "문서 타입은 하드코딩 금지" → "접근 범위도 하드코딩 금지" 추가
   - Scope Profile 기반 ACL 필터링의 의무 적용

#### 입력 (선행 의존)
- Season 2 개발계획.md (제출됨)
- S1 완수 보고서 및 회고
- 팀 회의 기록 및 합의 사항

#### 출력 (산출물)
- 확정된 S2 원칙 문서 (CLAUDE.md 업데이트)
- 핵심 설계 결정 문서 (확정본)
- AI품질평가.md 규약 및 템플릿
- Feature Group 운영 규약 문서

#### 검수 기준
- S1 4대 원칙과의 정합성 확인
- S2 추가 원칙이 구체적 구현 지침으로 명시되었는가
- AI품질평가 기준값이 정량적으로 측정 가능한가
- FG 소분 규칙이 Phase 1~9에 적용 가능한가
- 팀 전체 리뷰 및 동의 확보

---

### FG0.2 — S1 기술 부채 청산

#### 목적
S1에서 이월된 7개 Action Item을 완전히 해소하고, 사용자 경험의 핵심 경로(로그아웃, 설정, Admin)를 정상화한다.

#### 주요 작업 항목

**[Critical Priority]**

1. 사용자 메뉴 및 네비게이션 구축 (로그아웃, 설정, Admin 전환)
   - User UI 헤더 우측에 "사용자 메뉴" (아바타 또는 user icon) 추가
     - 프로필 보기
     - 사용자 설정 진입
     - 로그아웃
   - Admin 역할 사용자에게만 "관리자 설정" 메뉴 항목 노출
     - User UI → Admin UI 전환 링크
   - Admin UI 헤더에 "User UI로 돌아가기" 링크
   - S1 Phase 7/14에서 구축된 Admin 페이지로의 라우팅 연결
     - 사용자/역할 관리
     - 시스템 설정
     - 모니터링 대시보드
     - 감사 로그
     - API 키 관리
   - UI 5회 이상 리뷰 실시 및 개선

2. 프런트엔드 unwrapEnvelope<T>() 도입 및 리팩토링
   - 모든 API 호출 응답에 envelope 구조(success, data, error) 통일
   - `unwrapEnvelope<T>(response): T` 유틸리티 함수 구현
   - 기존 모든 `*Api` 호출부 리팩토링
   - API 응답 처리 산발(ad-hoc) 로직 제거
   - 타입 안전성 강화 (TypeScript generic 활용)

3. Fresh-boot 스모크 CI 잡 구축
   - PostgreSQL 신규 인스턴스에서 fresh-boot 환경 테스트
   - pgvector 설치 유무에 따른 2가지 매트릭스 실행
     - Matrix 1: pgvector 활성 상태
     - Matrix 2: pgvector 비활성 상태 (RAG 비활성)
   - 각 매트릭스에서 기본 CRUD, 검색, 인증 흐름 정상 작동 확인
   - 초기 admin 사용자 생성 및 로그인 검증
   - 실패 시 빌드 실패 처리

4. `/api/v1/system/capabilities` 엔드포인트 신설
   - `.well-known` 규약 정합: `/.well-known/mimir-capabilities` 또는 명시적 `/api/v1/system/capabilities`
   - 최소 노출 정보:
     - `supported_providers`: 현재 활성화된 LLM provider 목록 (Phase 1에서 확장)
     - `pgvector_enabled`: boolean (pg벡터 설치 여부)
     - `rag_available`: boolean (RAG 서비스 가용 여부)
     - `chunking_enabled`: boolean (문서 chunking 활성 여부)
   - 인증 불필요 (public endpoint)
   - 응답 캐시 (5분)

5. seed_users.py production 가드 추가
   - 환경변수 `ALLOW_SEED_USERS=true`일 때만 실행 가능하도록 가드
   - seed_users 실행 시 감사 로그에 `action=admin_seed_users` 기록
   - 프로덕션 환경에서 실수로 실행되지 않도록 warning 출력

**[High Priority]**

6. UI 5회 리뷰 체크리스트 표준화
   - "여정 기반" 사용자 시나리오 정의 (로그인→메뉴→설정→로그아웃)
   - 각 Phase별 UI 리뷰 체크리스트 템플릿 작성
   - 리뷰 항목: 접근성, 반응형, 오류 메시지, 네비게이션, 데이터 로딩 상태
   - 최소 5회 리뷰의 기록 양식 정의

7. 회귀 게이트 스크립트 FG 단위 복제 준비
   - S1의 Phase 14/16 통합 보안 회귀 스크립트를 참조
   - Phase별 FG 단위로 회귀 테스트 스크립트 디렉토리 구조 정의
   - 각 FG 완료 시 자동 실행 가능한 pytest 스크립트 템플릿 제공

#### 입력 (선행 의존)
- S1 회고의 7개 Action Item 상세 기술
- S1 Phase 7/14 Admin UI 기존 구현물
- 기존 API envelope 구조 분석

#### 출력 (산출물)
- User UI 헤더 네비게이션 컴포넌트 (로그아웃, 설정, Admin 메뉴)
- Admin UI 상단 네비게이션 컴포넌트
- 리팩토링된 모든 API 호출부 (`unwrapEnvelope<T>()` 적용)
- Fresh-boot CI 잡 (`.github/workflows/fresh-boot-smoke.yml`)
- `/api/v1/system/capabilities` 엔드포인트 구현
- seed_users.py 보호 로직
- UI 5회 리뷰 체크리스트 템플릿
- FG 단위 회귀 테스트 스크립트 템플릿

#### 검수 기준
- 사용자 메뉴(로그아웃, 설정) 접근 가능 여부
- Admin 역할 사용자가 "관리자 설정" 진입 가능 여부
- Admin UI와 User UI 간 전환 정상 작동 여부
- Fresh-boot 매트릭스(pgvector 있음/없음) 모두 통과 여부
- `/api/v1/system/capabilities` 응답 정상 및 필수 필드 포함 여부
- 모든 API 호출이 `unwrapEnvelope<T>()` 패턴 사용 여부
- seed_users 실행 시 감사 로그 기록 여부
- UI 5회 리뷰 기록 및 개선 사항 적용 여부

---

## 4. 기술 설계 요약

### 4.1 네비게이션 아키텍처
- User UI와 Admin UI를 별도 라우팅 경로로 분리 (이미 구현됨)
- 헤더 컴포넌트에서 사용자 역할에 따라 메뉴 조건부 렌더링
- Session storage에 현재 UI 모드(user/admin) 저장

### 4.2 API 응답 정규화
- 모든 API 응답: `{success: boolean, data?: T, error?: {code: string, message: string}}`
- 클라이언트에서 `unwrapEnvelope<T>(response)` 호출 시 성공/실패 자동 분기
- TypeScript generic으로 응답 타입 추론

### 4.3 Fresh-boot 환경 검증
- GitHub Actions 매트릭스 전략: `[pgvector: [true, false]]`
- 각 매트릭스에서 독립적 PostgreSQL 인스턴스 생성
- 초기화 스크립트 실행 → seed_users 실행(비활성) → 기본 CRUD 검증

### 4.4 Capabilities Endpoint 설계
- 상태 기반 응답 (현재 환경 설정에 따라 동적)
- Phase 1에서 `supported_providers` 필드 확장 (OpenAI, Anthropic, vLLM, Ollama 등)
- 클라이언트가 수동으로 기능 확인 가능하도록 설계

---

## 5. 의존 관계

### 선행 Phase
- 없음 (S2 첫 번째 Phase)

### 후행 Phase (이 Phase의 산출물을 소비하는 Phase)
- **Phase 1**: S2 원칙(⑤⑥⑦) 및 설계 결정(OAuth, Scope Profile, Citation Contract) 기반 사용
- **Phase 2~9**: AI품질평가.md 규약 및 FG 소분 운영 규약 적용
- **모든 Phase**: UI 5회 리뷰 체크리스트 및 Fresh-boot CI 매트릭스 활용

---

## 6. 검수 기준 종합

### Phase 0 수준의 검수 기준

| 항목 | 기준 |
|------|------|
| **FG0.1** | S2 원칙이 CLAUDE.md에 등재되었는가 / AI품질평가.md 템플릿이 Phase 1에서 즉시 사용 가능한가 / FG 운영 규약이 모호하지 않은가 |
| **FG0.2 - 네비게이션** | 로그아웃 메뉴 클릭 시 세션 종료 여부 / Admin 역할 사용자만 "관리자 설정" 메뉴 표시 여부 / Admin UI ↔ User UI 전환 정상 여부 |
| **FG0.2 - unwrapEnvelope** | 모든 API 호출부에서 `unwrapEnvelope<T>()` 사용 여부 / 타입 안전성 확보 여부 |
| **FG0.2 - Fresh-boot** | pgvector 활성/비활성 양쪽 매트릭스 모두 통과 여부 / seed_users 실행 가능/불가 제어 여부 |
| **FG0.2 - Capabilities** | `/api/v1/system/capabilities` 응답에 pgvector_enabled, rag_available 필드 포함 여부 |
| **FG0.2 - UI 리뷰** | 최소 5회 리뷰 기록 및 개선 사항 반영 여부 |
| **종합** | S1 부채 7개 Action Item 모두 해결 여부 / Phase 1 착수 준비 완료 여부 |

---

## 7. 예상 산출물 목록

| 산출물 | 형태 | 설명 |
|--------|------|------|
| S2 원칙 문서 (CLAUDE.md §2 업데이트) | Markdown | 원칙 ①~⑦ 최종본 |
| 핵심 설계 결정 확정서 | Markdown | 3.1~3.6 설계 결정 상세 기술 |
| AI품질평가.md 규약 | Markdown | 평가 항목, 기준값, 템플릿 |
| Feature Group 운영 규약 | Markdown | FG 소분, 독립 검수, 의무 산출물 정의 |
| User UI 네비게이션 컴포넌트 | TypeScript/React | 로그아웃, 설정, Admin 메뉴 |
| Admin UI 네비게이션 컴포넌트 | TypeScript/React | User UI 복귀 링크 |
| unwrapEnvelope<T>() 유틸리티 | TypeScript | API 응답 정규화 함수 |
| 리팩토링된 API 호출부 | TypeScript | 모든 `*Api` 객체 업데이트 |
| Fresh-boot CI 잡 | YAML | GitHub Actions workflow |
| /api/v1/system/capabilities 엔드포인트 | Python (FastAPI) | Capabilities 조회 API |
| seed_users.py 보호 로직 | Python | 환경변수 기반 가드 |
| UI 리뷰 체크리스트 | Markdown | 5회 리뷰 기록 템플릿 |
| FG 회귀 테스트 템플릿 | pytest/Python | FG 단위 테스트 스크립트 |
| **FG0.1 검수보고서** | Markdown | 설계 결정 검수 결과 |
| **FG0.1 보안취약점검사보고서** | Markdown | S2 원칙의 보안 함의 검토 |
| **FG0.2 검수보고서** | Markdown | 네비게이션, 부채 청산 검수 |
| **FG0.2 보안취약점검사보고서** | Markdown | seed_users 가드, 인증 경로 보안 검토 |
| **Phase 0 종결 보고서** | Markdown | Phase 0 전체 완료 보고 |

---

## 8. 리스크 및 대응

| 리스크 | 영향도 | 대응 방안 |
|--------|--------|----------|
| UI 리뷰 5회 이상 반복으로 일정 지연 | 중간 | 리뷰 팀 규모 확대, 평행 처리로 일정 압축 |
| Fresh-boot pgvector 설치 오류 | 높음 | CI 환경 미리 검증, 대체 DB 준비 (또는 비활성 매트릭스 우선) |
| 네비게이션 페이지 라우팅 충돌 | 낮음~중간 | 라우팅 전략 사전 설계, 테스트 우선 |
| seed_users 가드로 인한 프로덕션 배포 이슈 | 낮음 | 문서화 및 배포 절차 사전 공유 |
| API envelope 일괄 리팩토링 누락 | 중간 | 자동화된 린트(eslint rule) 추가, PR 리뷰 강화 |
| Capabilities endpoint 필드 부족 (Phase 1 확장 미준비) | 낮음 | 필드를 map/dict로 동적 확장 가능하도록 설계 |

---

## 9. 선행 조건

Phase 0을 시작하려면:
- S1 완수 보고서 및 회고 검토 완료
- Season 2 개발계획.md 전사 검토 및 기본 합의
- 프로젝트 팀의 S2 원칙 이해도 확보
- S1 Admin UI 기존 구현 상태 파악

---

## 10. 완료 기준

Phase 0 완료로 판단하는 조건:

1. S2 원칙(⑤⑥⑦)이 CLAUDE.md에 등재됨
2. AI품질평가.md 규약이 정의되고 템플릿이 제공됨
3. Feature Group 소분 운영 규약이 문서화됨
4. 사용자 메뉴(로그아웃)에 접근 가능함
5. Admin 역할 사용자가 관리자 설정 메뉴 접근 가능
6. 모든 API 호출이 `unwrapEnvelope<T>()` 패턴 사용
7. Fresh-boot CI 매트릭스(pgvector 있음/없음) 양쪽 모두 통과
8. `/api/v1/system/capabilities` 엔드포인트 정상 작동
9. seed_users 환경변수 가드 작동
10. UI 5회 이상 리뷰 기록 및 개선 반영
11. 모든 FG 검수보고서 및 보안취약점검사보고서 승인
12. Phase 0 종결 보고서 작성 완료

---

## 11. 권장 투입 순서

1. FG0.1 원칙 및 규약 확정 (우선: 팀 회의)
2. AI품질평가.md 규약 및 템플릿 정의
3. FG0.2 Critical Priority 항목들 병행
   - 네비게이션 구축 (UI 5회 리뷰 포함)
   - Fresh-boot CI 구축
4. FG0.2 High Priority 항목들
   - unwrapEnvelope 리팩토링
   - Capabilities endpoint
   - seed_users 가드
5. FG0.1 검수 및 보안 검사
6. FG0.2 검수 및 보안 검사
7. Phase 0 종결 보고서 작성

---

## 12. 기대 효과

Phase 0 완료 시:
- 사용자가 "로그아웃"할 수 있는 정상적인 제품이 됨
- S2 원칙이 명확해져 Phase 1~9의 방향성이 일관됨
- AI품질평가 기준이 정해져 각 FG에서 즉시 적용 가능
- S1의 잔존 부채가 완전히 해소되어 Phase 1 착수 가능
- Fresh-boot 검증으로 폐쇄망 환경 지원 준비 완료
