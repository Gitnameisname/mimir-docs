# FG 3-1 — audit 이벤트 인벤토리 실측

**작성일**: 2026-04-27
**대상**: task3-1 Step 1 (Contributors 패널이 의존하는 audit_events.event_type 의 실측 + 누락 emit 보강 결정)

---

## 1. 측정 방법

```
grep -rn 'event_type="' backend/app --include='*.py' \
  | sed -E 's/.*event_type="([^"]+)".*/\1/' \
  | sort -u
```

총 distinct `event_type`: **115 종**.

본 문서는 Contributors 패널 (FG 3-1) 의 4 카테고리 (작성자 / 편집자 / 승인자 / 최근 열람자) 가 의존하는 이벤트만 추린다.

---

## 2. 카테고리별 매핑

### 2.1 작성자 (Creator)

audit_events 와 무관. `documents.created_by` 컬럼이 정본.

### 2.2 편집자 (Editors)

| event_type | emit 위치 (파일:라인) | 비고 |
|-----------|---------------------|-----|
| `document.created` | `api/v1/documents.py:204` | 작성자와 중복 — 응답에서 제외 또는 `actor_user_id == created_by` 필터 |
| `document.updated` | `api/v1/documents.py:304` | metadata · title · status 변경 |
| `draft.updated` | `api/v1/documents.py:556` | draft body 변경 |
| `draft.nodes_saved` | `api/v1/documents.py:618` | draft node 단위 저장 |
| `draft.discarded` | `api/v1/documents.py:679` | draft 폐기 (편집자로 카운트) |
| `version.created` | `api/v1/documents.py:500` | 새 버전 생성 |
| `version.restored` | `api/v1/documents.py:894` | 이전 버전 복원 |
| `document.folder_set` | `api/v1/documents.py:360` | **편집자에서 제외** — 폴더 분류는 메타 행위 (FG 3-1 §2.1 정책 결정) |

### 2.3 승인자 (Approvers)

audit_events 의 `document.published` 이벤트는 emit 자체로는 actor 가 published 를 트리거한 사람이지만, **승인 흐름의 정본은 `workflow_history`**:

```sql
SELECT DISTINCT actor_id, actor_role, MAX(created_at) AS last_activity_at
FROM workflow_history
WHERE document_id = $1 AND to_status = 'published'
GROUP BY actor_id, actor_role
```

- 컬럼: `actor_id` (varchar 255), `actor_role` (varchar 100), `to_status='published'` 가 published 전이 표지
- `workflow_history.action` 값은 별도 enum 미정의, 자유 문자열. `to_status` 가 더 안정적

### 2.4 최근 열람자 (Recent Viewers)

**현재 emit 없음.** `document.viewed` 이벤트가 어떤 라우터에서도 발화되지 않는다 (115 distinct event_types 에 부재).

**보강 결정**: `GET /api/v1/documents/{document_id}` 라우터에서 emit 추가. 다음 절 §3 의 throttle 정책 적용.

---

## 3. `document.viewed` emit 보강 정책

### 3.1 emit 위치

- 파일: `backend/app/api/v1/documents.py` `get_document` (line 231)
- 시점: service 호출이 성공한 직후 (404 시 emit 없음)

### 3.2 actor 처리

- 인증 사용자만 emit (anonymous viewer 는 actor_user_id 가 없어 contributors 카테고리에 의미 없음)
- `actor.actor_type` 그대로 (user / agent / system)
- service actor (시스템 호출) 는 emit skip — Contributors 패널의 viewer 카운트를 오염시키지 않기 위함

### 3.3 Throttle / Dedup

- **per (actor_id, document_id) 5분 윈도우 내 중복 emit 1건으로 합침**
- 이유: 새로고침 / SPA 라우팅 / 폴링이 동일 viewer 의 view 를 폭주시키는 것을 방지
- 구현: `app.audit.viewed_throttle` 모듈의 in-process LRU 캐시
  - 다중 워커 환경에서 워커별 독립 캐시 — Strict dedup 필요 시 후속에 Valkey 도입 (현재는 best-effort)
  - max entries = 5000, eviction = LRU
  - window = `AUDIT_VIEWED_DEDUP_WINDOW_SEC` 환경변수 (기본 300)
- emit skip 시에도 응답은 정상 반환 (사용자 영향 0)

### 3.4 Contributors 패널 수렴

- §2.2 의 7 종 event_type 를 distinct actor_user_id 로 union → editors 섹션
- §2.3 의 workflow_history `to_status='published'` distinct → approvers 섹션
- §2.4 의 새 `document.viewed` emit → viewers 섹션 (FG 3-2 정책 게이트로 노출 제어)

---

## 4. event_type 레지스트리 위치 결정

- `backend/app/audit/event_types.md` 신설 (본 PR에서)
- 기존 라우터들이 hardcoded literal 을 사용 중이므로 enum 강제는 별 라운드 (현재는 문서만)
- 본 PR 범위: §2 의 ≥ 1 회 사용된 7 종 + 신규 `document.viewed` 1 종 = 8 종 명시

---

## 5. 회귀 영향 분석

| 항목 | 영향 |
|-----|-----|
| 기존 audit_events 레코드 | 없음 (신규 emit만) |
| 응답 latency | get_document 직후 emit (별도 트랜잭션) — 응답 path 비차단 |
| 테이블 크기 증가 | 5분 dedup 적용 시 일반적인 viewer 수 × 0.2~0.5 배. 7일 retention 정책 (별도) 와 결합 권장 |
| 멀티 워커 dedup 누수 | 최대 워커 수만큼 중복 emit (4 워커 = 최대 4배). 운영 영향 미미. Strict dedup 은 별 라운드 |

---

## 6. 결론

- `document.viewed` 1 종 신설 + throttle helper 1 모듈 + 라우터 1 곳 emit 추가로 §1 인벤토리 격차 봉합 완료
- §2 의 매핑은 `services/contributors_service.py` 의 입력 사양으로 직결됨 (task3-1 Step 2~3)

---

*작성: 2026-04-27 | task3-1 Step 1 산출물*
