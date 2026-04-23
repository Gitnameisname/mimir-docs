# task0-5 — 문서별 벡터화 상태 가시성 + 재벡터화 버튼

**작성일**: 2026-04-23
**Phase / FG**: S3 Phase 0 / FG 0-5
**상태**: 착수 대기
**담당**: 백엔드 + 프런트엔드
**예상 소요**: 2~3일

---

## 1. 작업 목적

문서를 **publish** 하면 `app/api/v1/workflow.py:_trigger_vectorization_async` 가 ThreadPoolExecutor 에 벡터화 작업을 큐잉한다. 이 경로는 주 요청(publish API)의 성공/실패와 **분리**되어 있어, 벡터화가 실패해도 사용자는 "publish 성공"으로 인지한다. 실패는 로그에만 남고 챗봇/RAG는 해당 문서를 찾지 못한다.

2026-04-23 실사례: LLM 연결 복구 후 챗봇 호출은 정상이나 문서 기반 답변이 계속 비었다. 원인 진단에 상태 조회 API·UI 가 전무하여 PostgreSQL/Milvus 를 직접 들여다봐야 했다.

본 작업은 **(a) 문서 상세 페이지에서 벡터화 상태를 한눈에 인지**하고 **(b) Admin 또는 문서 작성자가 즉시 재벡터화를 트리거**할 수 있는 수단을 제공한다.

---

## 2. 작업 범위

### 2.1 포함

- **상태 조회 API 신설**: `GET /api/v1/vectorization/documents/{document_id}/status`
- **재벡터화 API 권한 확장**: 기존 `POST /api/v1/vectorization/documents/{document_id}` 를 Admin 전용에서 **Admin + 문서 작성자**로 확장
- **서버측 쿨다운**: 문서당 10초 재시도 방지 (동일 액터 기준)
- **프런트엔드 벡터화 패널**: 문서 상세 페이지에 상태 뱃지 + 재벡터화 버튼
- **MCP Tool**: `mimir.vectorization.status(document_id)` 추가 (읽기 전용)
- **회귀 테스트 ≥ 12건**
- **UI 디자인 리뷰 ≥ 5회**
- 검수·보안 보고서

### 2.2 제외

- 벡터화 진행률 실시간 WebSocket 스트리밍 (본 FG 는 폴링)
- 관리자 대시보드 목록형 UI (S3 2라운드 또는 별도 FG)
- 에이전트의 재벡터화 실행 권한 (Tool 에 노출하지 않음 — 사람 명시적 클릭만)
- 실패한 벡터화 건의 자동 재시도 배치 (현재 FG 범위 밖; S3 2라운드 검토)

### 2.3 상태 분류 (정의)

| status | 의미 | 판정 기준 |
|--------|-----|-----------|
| `not_applicable` | 벡터화 대상 아님 | 발행된 버전 없음 (모든 버전 DRAFT/IN_REVIEW/APPROVED) |
| `pending` | 큐잉 됨, 아직 시작 전 | 최근 publish 이벤트 있고 `document_chunks` 0건, 실패 로그 없음 |
| `in_progress` | 실행 중 (best-effort) | (옵션) 실행 중 플래그 테이블 또는 transient marker. 확실치 않으면 `pending` 으로 수렴 가능 |
| `failed` | 최근 시도 실패 | 실패 기록 (로그 테이블 또는 `vectorization_jobs` 등) 에 최근 실패 존재 |
| `stale` | 이전 버전은 색인 · 최신 publish 버전은 미색인 | `latest_published_version_id != indexed_version_id` |
| `indexed` | 최신 publish 버전이 정상 색인 | 위 외 정상 |

---

## 3. 선행 조건

- FG 0-1 통합 테스트 CI (testcontainers) 구축 중이거나 완료 — IT-02 시나리오 확장이 본 FG 검증의 핵심
- 기존 `app/api/v1/vectorization.py` 의 stats/reindex 엔드포인트가 운영 중
- `documents.created_by` 컬럼 존재 확인 (S1 이전부터 존재로 추정)
- `audit_logs` 에 벡터화 실패 이벤트 기록 여부 확인 — 없으면 간단한 `vectorization_jobs` 테이블 추가 (아래 §4.2)

---

## 4. 주요 작업 항목

### 4.1 상태 조회 API (백엔드)

**엔드포인트**: `GET /api/v1/vectorization/documents/{document_id}/status`

**응답 스키마**:
```jsonc
{
  "document_id": "uuid",
  "latest_published_version_id": "uuid | null",
  "indexed_version_id": "uuid | null",
  "status": "indexed" | "pending" | "in_progress" | "failed" | "stale" | "not_applicable",
  "chunk_count": 42,
  "last_vectorized_at": "2026-04-23T07:00:00+00:00 | null",
  "last_error": "MilvusConnectionError: ... | null",
  "can_reindex": true,
  "reindex_cooldown_sec": 0
}
```

**ACL**:
- 기본적으로 Scope Profile 기반 문서 조회 권한이 있는 사용자는 상태를 읽을 수 있다
- `can_reindex` 판정:
  - `actor.roles` 에 `SUPER_ADMIN` 또는 `ADMIN` 포함 → true
  - 또는 `actor.user_id == documents.created_by` → true
  - 그 외 → false

**구현 힌트**:
- 단일 엔드포인트 하나의 SELECT 로 끝낼 수 있으면 좋음
- PostgreSQL 에서:
  - `documents` 테이블의 최신 published `version_id` 조회
  - `document_chunks` 테이블의 해당 document 의 가장 최근 `version_id` + `COUNT(*)` + `MAX(created_at)` 조회
  - `audit_logs` 또는 `vectorization_jobs` 에서 최근 실패 기록 조회 (LEFT JOIN 또는 분리 쿼리)

### 4.2 벡터화 작업 로그 (선택적 스키마 확장)

`audit_logs` 의 현재 스키마로 충분하지 않다면 작은 전용 테이블을 추가:

```sql
-- Alembic revision 추가
CREATE TABLE IF NOT EXISTS vectorization_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL,
    version_id      UUID,
    status          VARCHAR(16) NOT NULL CHECK (status IN ('queued','in_progress','succeeded','failed')),
    started_at      TIMESTAMPTZ DEFAULT NOW(),
    finished_at     TIMESTAMPTZ,
    error_summary   TEXT,
    chunk_count     INTEGER,
    actor_id        UUID,
    actor_type      VARCHAR(16),    -- user / agent
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_vec_jobs_doc ON vectorization_jobs(document_id, created_at DESC);
```

`_trigger_vectorization_async` 와 `vectorize_version` 시작/종료 지점에서 이 테이블에 쓴다. 기존 구조를 최소 침해로 확장.

### 4.3 재벡터화 API 권한 확장

`POST /api/v1/vectorization/documents/{document_id}` (기존) 의 권한 가드를:
```
require_admin_access
```
에서
```
require_admin_access_or_document_owner(document_id)
```
같은 커스텀 guard 로 변경. Admin 이거나 `documents.created_by == actor.user_id` 일 때만 통과.

**추가 처리**:
- 서버측 쿨다운: Valkey `SET vec_reindex:{document_id}:{actor_id} 1 EX 10 NX` — 실패 시 `429 + Retry-After: <남은 초>`
- `audit_logs` 에 `actor_type` 포함 기록 (S2 ⑤)
- 비동기 큐잉은 기존 경로(`_trigger_vectorization_async` 와 동일한 스레드 풀) 재사용

### 4.4 MCP Tool 표면

`app/schemas/mcp.py` (또는 동등 경로) 의 TOOL_SCHEMAS 에:
```python
{
    "name": "mimir.vectorization.status",
    "description": "문서의 벡터화 상태를 조회. 에이전트가 RAG 답변을 찾지 못할 때 진단 용도.",
    "inputSchema": {
        "type": "object",
        "properties": {"document_id": {"type": "string", "format": "uuid"}},
        "required": ["document_id"]
    }
}
```
재벡터화 실행 Tool 은 **노출하지 않는다** (운영 안전, 사람 명시적 클릭만).

### 4.5 프런트엔드 — 문서 상세 벡터화 패널

**위치 옵션** (UI 리뷰로 확정):
- A. 문서 헤더 우측 영역에 소형 뱃지 + 아이콘 버튼
- B. 사이드 패널 독립 섹션 (기존 Contributors/메타데이터 인근)

**표시 요소**:
- 뱃지: `indexed`=green, `stale`=amber, `pending`=gray, `in_progress`=blue(spinner), `failed`=red, `not_applicable`=neutral
- 라벨: 상태 텍스트 (한국어) + 마지막 성공 시각 (local + relative "3분 전")
- 청크 수 (hover tooltip 또는 details)
- `failed` 시: 에러 1줄 요약 + details 토글(스택)
- **재벡터화 버튼**:
  - `can_reindex=true` 일 때만 렌더
  - 클릭 → 확인 모달 "이 문서를 재벡터화합니다. 계속하시겠습니까?" (Admin/작성자 구분 문구)
  - 실행 중에는 disabled + spinner
  - 쿨다운 중에는 disabled + tooltip "남은 대기: n초"
  - 성공 응답 후 2초 간격으로 `/status` 폴링 (최대 60초) — `indexed` 또는 `failed` 로 수렴하면 중지

**접근성**:
- 뱃지: `aria-live="polite"` (상태 변경 시 스크린리더 알림)
- 버튼: 명시적 `aria-label="재벡터화 실행"`
- 키보드 네비게이션: Tab 순서 상 편집 컨트롤 이후

**UI 디자인 리뷰 (CLAUDE.md 규칙)**:
- 최소 5회 리뷰, 각 리뷰에서 개선점/결론 기록
- 데스크탑/웹 호환 (모바일 차단하지 않도록 반응형 breakpoint 고려)
- 리뷰 로그: `docs/개발문서/S3/phase0/산출물/FG0-5_UI리뷰로그.md`

### 4.6 회귀 테스트

상태 판정 유닛 테스트 (≥ 8 건):
- not_applicable: 발행 안 됨
- pending: publish 직후 청크 0
- indexed: 최신 버전과 indexed 버전 일치
- stale: 최신 버전 > indexed 버전
- failed: 최근 실패 기록 존재
- failed 후 재시도 → pending 전이
- in_progress 관찰(실행 중 플래그 사용 시) — 어려우면 `pending` 에 흡수

권한/쿨다운 테스트:
- admin → 200
- 작성자 → 200
- 제3자 → 403
- 연속 2회 호출 → 두 번째 429 + `Retry-After`

통합 테스트 (FG 0-1 IT-02 확장):
- publish → status 조회 → 벡터화 완료 대기 → status=indexed 검증

프런트엔드 컴포넌트 테스트 (RTL/Playwright 등 프로젝트 스택에 맞게):
- status 각 분기별 렌더 스냅샷
- 버튼 활성/비활성 조건
- 쿨다운 tooltip

### 4.7 폐쇄망 안전 (S2 ⑦)

- Milvus / 임베딩 서비스 off 상태에서도 **상태 조회는 정상 응답** (DB 만으로 구성 가능한 필드 유지)
- 재벡터화 클릭 시 의존 서비스 down 감지 → 즉시 `status=failed` + `last_error` 에 `"embedding service unavailable"` 등 명시적 문자열

### 4.8 산출물

- 백엔드: 라우트 + Guard + Alembic revision(선택)
- 프런트엔드: 벡터화 패널 컴포넌트 + 상태 훅
- MCP Tool: `mimir.vectorization.status`
- `docs/개발문서/S3/phase0/산출물/FG0-5_검수보고서.md`
- `docs/개발문서/S3/phase0/산출물/FG0-5_보안취약점검사보고서.md`
- `docs/개발문서/S3/phase0/산출물/FG0-5_UI리뷰로그.md`

---

## 5. 완료 기준

- [ ] `GET /api/v1/vectorization/documents/{id}/status` 모든 status 분기 응답 확인
- [ ] 권한: Admin + 작성자 통과, 제3자 403
- [ ] 쿨다운: 연속 호출 429 + Retry-After
- [ ] 버튼 클릭 → 파이프라인 실제 트리거 → 상태 폴링 → indexed 수렴 확인 (E2E)
- [ ] MCP Tool 등록, 에이전트 호출 샘플 성공
- [ ] 회귀 테스트 ≥ 12건 녹색
- [ ] UI 디자인 리뷰 ≥ 5회 로그 제출
- [ ] 폐쇄망 모드(Milvus off) 에서 상태 조회 정상 + `failed` 로 명확히 마킹
- [ ] 검수 · 보안 보고서 제출

---

## 6. 리스크와 대처

| 리스크 | 대처 |
|--------|------|
| `in_progress` 상태 판정의 정확도 | transient marker 구현이 부담되면 `pending` 에 흡수하고 UI 에는 "대기 중/실행 중" 통합 레이블 |
| 대용량 문서의 재벡터화로 서비스 부하 | 스레드 풀은 기존 max_workers=3 유지. 버튼 쿨다운 10초로 폭주 억제. 추가 대응은 S3 2라운드 |
| 쿨다운 Valkey 의존 | Valkey off 시 in-memory fallback (동일 프로세스 내에서만 유효) + 경고 로그. 폐쇄망 원칙 유지 |
| 작성자 권한 확장에 따른 잘못된 사용 | audit_logs 에 actor_id/actor_type 기록. Admin 대시보드에서 재벡터화 이력 확인 가능하도록 기록만 확보 |
| 상태 폴링이 네트워크 낭비 | 지수 백오프 고려 (2s → 4s → 8s), 60초 상한. 완료 수렴 시 즉시 중지 |

---

## 7. 보안 주의

- 상태 응답은 해당 문서에 대한 조회 권한(Scope Profile) 있는 사용자에게만 반환
- 에러 메시지 노출: 사용자 화면에는 요약만, 상세 스택은 Admin 에게만 보이도록 details 권한 가드
- 429 Retry-After 남용 시 DDoS 방지: 전역 rate limit 에 함께 포함
- MCP Tool 은 읽기 전용. 재벡터화 Tool 미노출 — 명시적으로 PR 설명에 기록

---

## 8. 참조

- `docs/개발문서/S3/phase0/Phase 0 개발계획서.md` §2 FG 0-5
- `docs/개발문서/S3/phase0/작업지시서/task0-1.md` — IT-02 시나리오 확장
- `app/api/v1/workflow.py:_trigger_vectorization_async`
- `app/api/v1/vectorization.py` (stats / reindex 기존 엔드포인트)
- `app/services/vectorization_service.py:vectorize_version`, `semantic_search`
- 2026-04-23 세션 로그 (RAG 결과 부재 사건 — 이 FG 의 동기)
- `CLAUDE.md` (UI 리뷰 5회 규칙, S1 ④, S2 ⑤ ⑦)

---

*작성: 2026-04-23 | 착수 대기*
