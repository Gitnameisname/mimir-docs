# Phase 9 보안 취약점 검사 리포트

**검사 일시:** 2026-04-09  
**검사 대상:** Phase 9 — 변경 비교 및 이력 가시화 기능  
**검사 기준:** OWASP Top 10, CWE, 프로젝트 보안 정책  

---

## 1. 검사 요약

| 항목 | 결과 |
|---|---|
| 검사 파일 수 | 백엔드 3개, 프론트엔드 8개 |
| 발견된 취약점 | 2개 (수정 완료) |
| 설계 이슈 | 1개 (권고 사항) |
| 보안 양호 항목 | 8개 |

| 취약점 ID | 분류 | 심각도 | 상태 |
|---|---|---|---|
| VULN-1 | LCS DP 메모리/CPU DoS | **높음** | ✅ 수정 완료 |
| VULN-2 | f-string SQL 안티패턴 | 낮음 | ✅ 수정 완료 |
| DESIGN-1 | 미인증 diff 접근 정책 | 설계 이슈 | ⚠ 권고 사항 |

---

## 2. 발견된 취약점

---

### VULN-1: LCS DP 테이블 메모리/CPU DoS

**분류:** CWE-400 Uncontrolled Resource Consumption  
**심각도:** 높음  
**OWASP:** A05:2021 Security Misconfiguration  
**파일:** `backend/app/services/diff_service.py`

#### 취약점 설명

`_lcs_diff` 함수는 LCS(Longest Common Subsequence) 알고리즘을 위해 `(n+1) × (m+1)` 크기의 2차원 DP 테이블을 메모리에 할당한다. 여기서 n, m은 각각 버전 A, B의 텍스트 토큰 수다.

API에서 `max_inline_length` 쿼리 파라미터 최대값은 50,000자(`le=50_000`)로 제한되어 있으나, 텍스트 길이 체크(`len(a) + len(b) > max_length`)만 수행하고 **토큰 수 상한 검사는 없었다.**

**최악 케이스 공격 시나리오:**

```
GET /api/v1/documents/{doc_id}/versions/{v1_id}/diff/{v2_id}?inline_diff=true&max_inline_length=50000
```

- 대상 문서에 "a b c d e ..." 형식 노드 (모든 단어가 공백으로 구분된 1자 어절)
- `len(a) = len(b) = 25,000` → 체크 통과
- 토큰화 결과: `tokens_a ≈ tokens_b ≈ 25,000개`
- DP 테이블: `25,000 × 25,000 = 625,000,000 셀`
- Python list 메모리: **약 5GB** (단일 요청)

공격자가 이런 구조의 문서를 작성하고 `max_inline_length=50000`으로 반복 요청하면 서버 메모리 고갈 가능.

#### 영향 범위

- anonymous 사용자 포함 모든 API 접근자 (require_authenticated=False)
- 서비스 전체 가용성 저하 (DoS)

#### 수정 내용

`MAX_LCS_CELLS = 1_000_000` 상수를 도입하고, `_lcs_diff` 함수 시작에 셀 수 상한 검사 추가:

```python
# 수정 전
def _lcs_diff(tokens_a, tokens_b):
    n, m = len(tokens_a), len(tokens_b)
    dp = [[0] * (m + 1) for _ in range(n + 1)]  # 셀 수 무제한
    ...

# 수정 후
MAX_LCS_CELLS = 1_000_000  # 약 8MB RAM 상한

def _lcs_diff(tokens_a, tokens_b):
    n, m = len(tokens_a), len(tokens_b)
    if n * m > MAX_LCS_CELLS:  # ← 추가
        raise ValueError(
            f"LCS DP 셀 수({n * m:,})가 최대({MAX_LCS_CELLS:,})를 초과합니다."
        )
    dp = [[0] * (m + 1) for _ in range(n + 1)]
    ...
```

`TextDiffer.diff`의 기존 `except Exception` 핸들러가 `ValueError`를 포착하여 `([], True)`(skipped=True)를 반환하므로 API 계층 변경 불필요.

**수정 후 최대 리소스:**
- 1,000,000 셀 × 8 bytes ≈ **8MB** (단일 요청, 허용 범위)

---

### VULN-2: f-string SQL 안티패턴

**분류:** CWE-89 SQL Injection (잠재적 패턴)  
**심각도:** 낮음  
**파일:** `backend/app/services/diff_service.py:765`

#### 취약점 설명

`compute_diff_with_previous`의 fallback 쿼리에서 f-string 형식이 사용되었다. 현재 코드에는 f-string에 변수가 삽입되지 않아 실제 SQL Injection이 발생하지 않으나, f-string을 SQL 작성에 사용하는 **패턴 자체**가 보안 안티패턴이다.

향후 개발자가 이 패턴을 참고해 변수를 f-string에 직접 삽입하면 SQL Injection으로 이어질 수 있다.

```python
# 수정 전 — f-string (안티패턴)
sql = f"""
    SELECT id, ...
    FROM versions
    WHERE document_id = %s AND version_number = %s
    LIMIT 1
"""

# 수정 후 — 일반 문자열
sql = """
    SELECT id, ...
    FROM versions
    WHERE document_id = %s AND version_number = %s
    LIMIT 1
"""
```

---

## 3. 설계 이슈

---

### DESIGN-1: 미인증 사용자의 모든 문서 diff 접근 허용

**분류:** CWE-284 Improper Access Control  
**심각도:** 설계 이슈 (플랫폼 수준)  
**파일:** `backend/app/api/v1/diff.py:49`

#### 이슈 설명

diff API의 `_authorize_diff` 헬퍼는 `require_authenticated=False`로 권한을 검증한다. 이로 인해 미인증(anonymous) 사용자가 인증 없이 모든 문서의 diff에 접근할 수 있다.

```python
def _authorize_diff(actor: ActorContext, document_id: str) -> None:
    authorization_service.authorize(
        actor=actor,
        action="document.read",
        resource=ResourceRef(resource_type="document", resource_id=document_id),
        require_authenticated=False,  # ← anonymous 허용
    )
```

`AuthorizationService.authorize` 동작:
```
anonymous actor + require_authenticated=False → RBAC 검사 없이 통과
```

**영향:**
- DRAFT, IN_REVIEW, REJECTED 상태 문서의 변경 이력 전체가 비인증 접근에 노출
- diff에는 실제 문서 콘텐츠(이전/이후 텍스트)가 포함되므로, 일반 읽기보다 민감

**플랫폼 일관성:**
이 패턴은 Phase 9에서 도입된 것이 아닌 플랫폼 전체에 걸쳐 동일하게 적용되어 있다:
- `nodes.py`: `node.list`, `node.read` → `require_authenticated=False`
- `versions.py`: `version.read` → `require_authenticated=False`
- `workflow.py`: `workflow.history.read`, `workflow.review_actions.read` → `require_authenticated=False`

따라서 이는 Phase 9 구현 버그가 아니라 **플랫폼 설계 결정**이다.

**권고 사항:**

옵션 A (현행 유지 + 명문화): 공개 접근 정책을 CLAUDE.md 또는 설계 문서에 명시적으로 기록.

옵션 B (향후 강화): 문서의 `workflow_status`를 확인하여 PUBLISHED가 아닌 문서는 인증 필요:
```python
# 향후 적용 예시
if doc.workflow_status not in PUBLIC_STATUSES:
    authorization_service.authorize(..., require_authenticated=True)
```

---

## 4. 보안 양호 항목

다음 항목들은 검사를 통해 취약점이 없음을 확인함.

---

### SEC-OK-1: XSS 방지 — InlineDiffRenderer

**파일:** `frontend/src/features/diff/InlineDiffRenderer.tsx`

React JSX로 토큰 텍스트를 렌더링하며 `dangerouslySetInnerHTML`을 사용하지 않는다. React의 기본 자동 이스케이프가 적용되어 악성 스크립트 삽입 불가.

```tsx
// 안전: React가 token.text를 textContent로 삽입 (HTML 이스케이프)
<mark>{token.text}</mark>
<del>{token.text}</del>
<span>{token.text}</span>
```

---

### SEC-OK-2: SQL Injection 방지

**파일:** `backend/app/services/diff_service.py`

모든 DB 쿼리는 psycopg2 파라미터 바인딩(`%s`)을 사용한다. 사용자 입력이 SQL 문자열에 직접 삽입되는 위치 없음.

```python
cur.execute(sql, (document_id, version_b.version_number - 1))  # 파라미터 바인딩
```

---

### SEC-OK-3: ReDoS 방지

**파일:** `backend/app/services/diff_service.py:114`

`_tokenize` 함수의 정규식 `r'\S+|\s+'`은 단순 문자 클래스 교대로 구성되어 catastrophic backtracking이 불가능하다.

```python
re.findall(r'\S+|\s+', text)
# \S+ : 1개 이상의 비공백 문자 (백트래킹 없음)
# \s+ : 1개 이상의 공백 문자 (백트래킹 없음)
```

---

### SEC-OK-4: 캐시 포이즈닝 방지

**파일:** `backend/app/services/diff_service.py`

캐시 키는 `f"diff:{document_id}:{min(v1,v2)}:{max(v1,v2)}"` 형식이다. 모든 구성 요소는 DB에서 검증된 UUID로, 사용자가 임의 캐시 키를 조작할 수 없다.

---

### SEC-OK-5: 재귀 무한 루프 방지

**파일:** `backend/app/services/diff_service.py`

`_find_top_section` 함수에 `MAX_PARENT_DEPTH = 50` 깊이 제한이 있어 순환 참조 등 비정상 트리에서도 무한 재귀가 발생하지 않는다.

---

### SEC-OK-6: 노드 수 초과 처리

**파일:** `backend/app/services/diff_service.py`

`MAX_NODES_SYNC = 10,000` 초과 시 `DiffTooLargeError` → `ApiValidationError(400)` 반환. 대용량 문서로 인한 서버 과부하 방지.

---

### SEC-OK-7: 권한 미정의 action 거부

**파일:** `backend/app/api/auth/authorization.py`

권한 매트릭스에 없는 action은 default-deny 처리된다:

```python
if allowed_roles is None:
    raise ApiPermissionDeniedError(
        f"Unknown action '{action}'. Access denied by default."
    )
```

---

### SEC-OK-8: 프론트엔드 URL 경로 주입 위험 없음

**파일:** `frontend/src/lib/api/diff.ts`

`diffApi`에서 URL에 삽입하는 `documentId`, `versionId`는 서버에서 받은 UUID다. 외부 사용자 입력(URL 파라미터, 폼 입력)이 직접 삽입되지 않아 경로 조작 위험이 없다.

---

## 5. 수정 적용 목록

| 파일 | 수정 내용 | 수정 일시 |
|---|---|---|
| `backend/app/services/diff_service.py` | VULN-1: MAX_LCS_CELLS 상수 추가, _lcs_diff 셀 수 상한 검사 | 2026-04-09 |
| `backend/app/services/diff_service.py` | VULN-2: f-string SQL → 일반 문자열 변경 | 2026-04-09 |

---

## 6. 미조치 항목 및 후속 작업

| ID | 내용 | 담당 | 기한 |
|---|---|---|---|
| DESIGN-1 | 공개 접근 정책 명문화 또는 비공개 문서 보호 추가 | 아키텍처 결정 필요 | Phase 10 이전 |

---

## 7. 결론

Phase 9 구현에서 발견된 보안 취약점은 2건이며 모두 수정 완료되었다.

- **VULN-1** (LCS DoS)은 악의적 사용자가 단일 API 요청으로 수GB RAM을 소비할 수 있는 실질적 위협으로, `MAX_LCS_CELLS` 상한 검사를 통해 수정되었다.
- **VULN-2** (f-string SQL)는 현재 실제 인젝션은 없으나 코드 패턴 위험으로, 일반 문자열로 수정되었다.
- **DESIGN-1** (anonymous 접근)은 플랫폼 전체 정책과 일관된 설계 결정이나, 문서의 기밀성 요구사항에 따라 강화가 필요할 수 있다.

XSS, SQL Injection, ReDoS, 캐시 포이즈닝, 무한 루프 등 주요 보안 항목은 모두 적절히 처리되어 있다.
