# Task 13-9. 운영 문서화 및 출시 준비

## 1. 작업 목적

Mimir 플랫폼의 운영 문서를 완성하고 최종 출시 준비를 완료한다.

이 작업의 목표는 다음과 같다.

- 운영 Runbook 작성 (장애 대응, 주요 운영 절차)
- 장애 대응 절차 문서화 (SOP)
- Pre-launch 체크리스트 작성 및 검증
- 운영 이관 체크리스트 작성
- API 문서 최종 검토 및 공개

---

## 2. 작업 범위

### 포함 범위

- 운영 Runbook 작성
- 주요 장애 시나리오별 대응 절차
- Pre-launch 체크리스트
- 운영 이관 문서
- API 문서 (OpenAPI) 최종 완성

### 제외 범위

- 사용자 교육 자료 (별도 진행)
- 마케팅/홍보 자료

---

## 3. 주요 구현 대상

### 3-1. 운영 Runbook 목록

| Runbook | 내용 |
|---------|------|
| RB-01 | 서비스 재시작 절차 |
| RB-02 | DB 연결 장애 대응 |
| RB-03 | 디스크 용량 부족 대응 |
| RB-04 | 백그라운드 Job 장애 대응 |
| RB-05 | 임베딩/LLM API 장애 대응 |
| RB-06 | 전체 DB 복구 절차 |
| RB-07 | 보안 인시던트 대응 |
| RB-08 | 배포 롤백 절차 |
| RB-09 | 인증서 갱신 절차 |
| RB-10 | 스케일 업/아웃 절차 |

#### Runbook 작성 형식

```markdown
# RB-02. DB 연결 장애 대응

## 증상
- 500 오류가 급증
- 로그에 "Connection refused" 또는 "Too many connections" 오류

## 확인 절차
1. DB 서버 상태 확인: `pg_isready -h $DB_HOST`
2. 연결 수 확인: `SELECT count(*) FROM pg_stat_activity;`
3. 슬로우 쿼리 확인: `SELECT * FROM pg_stat_activity WHERE state = 'active';`

## 대응 절차
### 연결 수 초과 시
1. 유휴 연결 종료: `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle';`
2. 앱 연결 풀 크기 조정 (DB_POOL_SIZE 환경 변수)
3. 앱 재시작

### DB 서버 다운 시
1. RB-06 (전체 DB 복구 절차) 참조

## 예방 조치
- Connection pool 크기 모니터링 알림 설정 (Task 13-3 참조)
```

---

### 3-2. Pre-launch 체크리스트

#### 보안

```
□ OWASP Top 10 점검 완료 (Task 13-1)
□ 모든 시크릿이 환경 변수로 관리됨
□ HTTPS 강제 설정
□ Rate Limiting 활성화
□ 의존성 취약점 없음 (High 이상)
□ 보안 헤더 설정 완료
```

#### 운영

```
□ 자동 백업 실행 및 복구 테스트 통과 (Task 13-5)
□ 모니터링 대시보드 가동 (Task 13-3)
□ 핵심 알림 규칙 설정 및 테스트 알림 수신 확인
□ Health Check 엔드포인트 동작
□ 로그 수집 파이프라인 가동 (Task 13-2)
□ 데이터 보존 정책 배치 등록 (Task 13-6)
```

#### 성능

```
□ 부하 테스트 통과 (50 동시 사용자, Task 13-8)
□ API P95 SLO 달성
□ RAG TTFT P95 < 2초 달성
```

#### 품질

```
□ 단위 테스트 커버리지 80% 이상 (Task 13-4)
□ E2E 핵심 시나리오 통과
□ CI/CD 파이프라인 가동 (Task 13-7)
□ Phase 12 플러그인 규격 검증 통과
```

#### 데이터

```
□ 운영 DB 마이그레이션 완료
□ 초기 DocumentType 설정 (POLICY, MANUAL, REPORT, FAQ)
□ Admin 계정 생성 및 접근 확인
□ pgvector 확장 활성화 확인
```

---

### 3-3. 운영 이관 체크리스트

신규 운영 담당자에게 전달할 항목:

```
□ 환경 변수 목록 및 설명 문서 전달
□ Admin 계정 인계 (비밀번호 변경 요청)
□ 모니터링 대시보드 접근 권한 부여
□ 알림 채널 (Slack/PagerDuty) 담당자 등록
□ Runbook 위치 안내
□ 배포 절차 교육
□ 백업/복구 절차 교육
□ 비상 연락 체계 수립
```

---

### 3-4. API 문서 (OpenAPI) 최종 완성

FastAPI 자동 생성 문서 최종 검토:

```python
app = FastAPI(
    title="Mimir API",
    description="범용 문서/지식 플랫폼 API",
    version="1.0.0",
    docs_url="/api/docs",      # Swagger UI
    redoc_url="/api/redoc",    # ReDoc
    openapi_url="/api/openapi.json",
)
```

```
점검 항목:
  □ 모든 엔드포인트에 summary와 description 작성
  □ 요청/응답 스키마 예시(example) 포함
  □ 인증 방식(Bearer Token) 문서화
  □ 오류 응답 코드 문서화
  □ 운영 환경에서 /api/docs 접근 제한 설정 (내부망 또는 Admin 전용)
```

---

## 4. 산출물

1. 운영 Runbook 10개 (RB-01~RB-10)
2. Pre-launch 체크리스트 완료 보고서
3. 운영 이관 문서
4. OpenAPI 문서 최종 완성

---

## 5. 완료 기준

- Pre-launch 체크리스트 전 항목이 완료되어 있다
- 운영 Runbook이 작성되어 있어 담당자 없이도 장애 대응 가능하다
- API 문서가 완성되어 있다
- 운영 이관 체크리스트가 준비되어 있다

---

## 6. Codex 작업 지침

- Runbook은 실제로 장애 상황에서 읽는 문서이므로 명확하고 단계별로 작성한다
- Pre-launch 체크리스트는 모든 항목이 실제 확인 후 체크되어야 하며 미완료 항목이 있으면 출시를 보류한다
- API 문서는 내부 개발자뿐 아니라 외부 연동 개발자도 이해할 수 있는 수준으로 작성한다
- 운영 이관 시 Admin 초기 비밀번호는 반드시 최초 로그인 시 변경하도록 강제한다

---

## 7. Phase 13 완료 = Mimir 플랫폼 v1.0 출시

Phase 13 완료 시점에서 Mimir는 다음을 갖춘 엔터프라이즈 지식 플랫폼이 된다:

- **문서 관리**: 구조화된 문서 생성/버전 관리/워크플로
- **검색**: 키워드 + 시맨틱 하이브리드 검색 (권한 반영)
- **RAG**: 문서 기반 질의응답 (근거 연결, 대화 이력)
- **확장**: DocumentType 플러그인으로 무한 확장 가능
- **운영**: 보안 강화, 모니터링, 자동 배포, 백업/복구 체계 완비
