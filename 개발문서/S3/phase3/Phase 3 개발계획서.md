# Phase 3 — Contributors 패널 + 인라인 주석

**작성일**: 2026-04-22
**Phase 상태**: 대기 (Phase 2 완료 선행)
**선행 조건**: Phase 2 게이트 통과
**후행 조건**: S3 1라운드 종결

---

## 1. Phase 개요

### 1.1 목적

문서의 **작성자 · 편집자 · 승인자 · 최근 열람자**를 한 패널에 모아 "누가 무엇에 관여했는지" 한눈에 보게 한다. 더불어 node_id 기반 **인라인 주석**을 추가하여 특정 블록/문장에 대한 토론이 가능하게 한다.

정보 공개의 민감성 때문에 **열람자 노출은 Scope Profile 설정 항목으로 on/off** 가능해야 한다 (정보보안 / 개인정보 충돌 방지).

### 1.2 원칙 강조

- 정보 출처는 기존 자료 재조합: `audit_logs` + `workflow_history` + `documents.*_by`. 신규 추적 로그 도입 최소.
- 열람자 노출 on/off는 조직 정책 문제. 개인 식별을 강제 공개하지 않는다.
- 인라인 주석의 좌표는 **node_id** 기준. 버전 간 보존. Phase 1의 node_id 안정성 위에 얹힘.

### 1.3 선행 조건

- Phase 2 완료
- `audit_logs` · `workflow_history` 스키마 안정 (S2 종결 상태)
- Phase 1의 node_id 안정성 회귀 테스트가 유지되고 있음

### 1.4 기대 결과

- 문서 상세에 Contributors 사이드 패널
- 열람자 노출 on/off 설정
- 인라인 주석: 특정 노드에 주석 추가, 버전 업 후에도 anchoring 유지
- 주석 해결/미해결 상태, 멘션 알림

---

## 2. Feature Group (FG) 요약

### FG 3-1 — Contributors 패널

- 작성자 (`documents.created_by`)
- 편집자 목록 (`audit_logs` 중 doc 편집 이벤트의 `actor_user_id`)
- 승인자 (`workflow_history` 중 approve 이벤트)
- 최근 열람자 (`audit_logs` view 이벤트, 기간 필터)
- **에이전트 액터 구분**: `actor_type` (user/agent) 표시 (S2 ⑤ 일관성)
- 사이드 패널 UI, 아바타/이름/역할 뱃지
- API: `GET /api/v1/documents/{id}/contributors?since=&include_viewers=`

### FG 3-2 — 열람자 노출 제어

- Scope Profile 설정 스키마에 `expose_viewers: bool` 추가
- off 시 Contributors 패널에서 "최근 열람자" 섹션 숨김 + API에서 제거
- 관리자 UI에서 Scope Profile별 설정 가능
- 회귀 테스트: off인 Scope Profile에서 API가 열람자를 반환하지 않음

### FG 3-3 — 인라인 주석

- 특정 node_id (+ 문자 오프셋 범위)에 주석 추가
- 해결/미해결 상태, 답글 스레드
- 버전 diff 후에도 같은 node_id의 주석은 보존
- node_id가 더 이상 존재하지 않으면 "고아 주석"으로 표시
- 멘션 `@user` → 알림 (S2 알림 시스템 재사용)
- API: `POST /documents/{id}/annotations`, `PATCH /annotations/{id}`, `GET /documents/{id}/annotations`
- MCP Tool 표면: 에이전트가 주석을 읽을 수 있도록 (S2 ⑤ 일관성)

---

## 3. 데이터 모델 (개요)

신규 테이블:
- `annotations` (id, document_id, version_id, node_id, span_start, span_end, author_id, actor_type, content, status, parent_id, ...)
- `annotation_mentions` (annotation_id, mentioned_user_id)

Scope Profile 스키마 확장:
- `scope_profiles.settings_json.expose_viewers` 추가 (기본값 false — 보수적)

---

## 4. 산출물 (각 FG)

- 작업지시서 (`task3-1.md` ~ `task3-3.md`)
- 검수보고서
- 보안취약점검사보고서 (특히 열람자 노출의 개인정보 영향 분석)
- UI 디자인 리뷰 ≥ 5회 로그

---

## 5. 게이트 / 완료 기준

- [ ] Contributors 패널 API + UI 동작
- [ ] 열람자 노출 on/off 회귀 테스트 녹색
- [ ] 인라인 주석: node_id 기반 anchoring + 버전 간 보존 테스트 녹색
- [ ] 고아 주석 표시 케이스 테스트
- [ ] MCP Tool로 주석 읽기 가능
- [ ] 각 FG의 검수·보안 보고서 제출
- [ ] UI 디자인 리뷰 ≥ 5회 로그 제출

---

## 6. 범위 밖 / 이월

- 리뷰 요청 / 승인 워크플로 확장 — S2 기존 워크플로 유지
- 주석의 문서 간 링크 — S4 검토
- 오프라인 주석 동기화 — S4

---

## 7. 리스크 (개요)

| 리스크 | 대처 방향 |
|--------|----------|
| 열람자 노출이 기본 on으로 배포되어 개인정보 이슈 발생 | 기본값 false 고정. 검수보고서에서 관리자 UI 기본값 스크린샷 첨부 |
| 인라인 주석 anchoring 드리프트 (버전 diff 후 위치 어긋남) | node_id 안정성 회귀 테스트 (Phase 1 유지) + 주석 anchoring 테스트 셋 ≥ 10 |
| 알림 폭주 (멘션 남용) | S2 알림 rate limit 재사용 |
| 에이전트가 대량 주석 생성 | `actor_type=agent` 필터 + 관리자 UI에서 에이전트 주석 일괄 숨김 옵션 |

---

## 8. S3 1라운드 종결

Phase 3 게이트 통과 = S3 1라운드 종결. 종결 회고 작성:
- `docs/개발문서/S3/종결회고.md`
- S1/S2/S3 원칙 준수 여부 자기 검증
- 잔존 부채 및 이월 항목 정리
- S3 2라운드 또는 S4 킥오프 기초 자료

---

## 9. 참조

- `docs/개발문서/S3/Season3_개발계획.md` §5 Phase 3
- S2 MCP Tool 표면 (actor_type, 감사 로그 구조)
- `/sessions/eloquent-practical-ramanujan/mnt/.auto-memory/project_s3_kickoff.md`

---

*작성: 2026-04-22 | Phase 2 완료 후 확정*
