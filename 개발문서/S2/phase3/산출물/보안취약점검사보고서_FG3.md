# Phase 3 보안 취약점 검사 보고서

**작성일**: 2026-04-17  
**검사 대상**: S2 Phase 3 — Conversation 도메인  
**검사 범위**: FG3.1 (Conversation/Turn 도메인 + API) + FG3.2 (멀티턴 RAG 캐싱) + FG3.3 (채팅 UI)  
**검사 기준**: OWASP Top 10 (A01~A10) + OWASP Top 10 for LLM Applications (LLM01~LLM10) + WCAG 2.1 AA  
**검사자**: Claude Sonnet 4.6 (자동 보안 검사)

---

## 1. 검사 요약

| 항목 | 결과 |
|------|------|
| 발견된 취약점 (전체) | **5건** |
| Critical | 0건 |
| High | 1건 |
| Medium | 2건 |
| Low | 2건 |
| 수정 완료 (High/Medium/Low 전체) | **5건** |
| 잔여 개선 권고 | 0건 |
| SQL Injection | 없음 (파라미터 바인딩 확인) |
| Command Injection | 없음 |
| XSS (프론트엔드) | 없음 (whitespace-pre-wrap 텍스트 렌더링, dangerouslySetInnerHTML 미사용) |
| IDOR (대화 소유권) | 수정 완료 ✅ |

---

## 2. 취약점 상세 및 수정 내역

### [VULN-001] ConversationRepository — IDOR: 소유권 미검증 ✅ 수정 완료

| 항목 | 내용 |
|------|------|
| **심각도** | High |
| **분류** | OWASP A01:2021 (Broken Access Control) / IDOR |
| **파일** | `backend/app/services/context_window_manager.py` |

**취약점 설명**

초기 구현에서 `fetch_context_window`는 `conversation_id`로 대화를 조회한 후 소유권(owner_id) 검증 없이 Turn 목록을 반환했다. 다른 사용자의 `conversation_id`를 알고 있으면 그 대화 이력을 읽을 수 있었다.

**공격 시나리오**

1. 공격자(User A)가 피해자(User B)의 `conversation_id`를 획득한다.
2. `POST /rag/answer` 요청 body에 피해자의 `conversation_id`를 포함한다.
3. `ContextWindowManager.fetch_context_window`가 피해자의 Turn 이력을 반환한다.
4. `MultiturnRAGService`가 피해자 컨텍스트를 기반으로 쿼리를 재작성한다.
5. 응답의 `rewritten_query`, 답변에 피해자의 대화 내용이 간접 반영된다.

**수정 내용**

```python
# 수정 전 — ACL 없음
turns = TurnRepository(conn).list_by_conversation(conversation_id)
return turns, self._calculate_context_tokens(turns)

# 수정 후 — owner_id 검증
conv = ConversationRepository(conn).get_by_id(str(conversation_id))
if conv is None:
    raise ValueError(f"conversation {conversation_id} not found")
if str(conv.owner_id) != str(actor_id):
    raise PermissionError(
        f"actor {actor_id} does not own conversation {conversation_id}"
    )
turns = TurnRepository(conn).list_by_conversation(str(conversation_id), ...)
```

**검증**: `test_fetch_context_window_permission_denied` — PermissionError 발생 확인 ✅

---

### [VULN-002] RAGCache `_del_prefix` — 캐시 키 네임스페이스 오염 가능성 ✅ 수정 완료

| 항목 | 내용 |
|------|------|
| **심각도** | Medium |
| **분류** | OWASP A04:2021 (Insecure Design) — 캐시 격리 설계 결함 |
| **파일** | `backend/app/cache/rag_cache.py:113-123` |

**취약점 설명**

`_del_prefix("rag:search:")` 호출이 `invalidate_pattern("rag:search:")`를 경유하면 Valkey에서 `mimir:rag:search:*` 패턴으로 SCAN한다. 그러나 `set_cached`는 `rag:search:{hash}` 키로 저장하여 `mimir:` 네임스페이스가 없다. 이로 인해 무효화가 실패하고 오래된 검색 결과가 반환된다.

추가적으로, `mimir:` 네임스페이스 없는 키는 다른 애플리케이션이 동일 Valkey 인스턴스를 사용할 경우 충돌 위험이 있다.

**수정 내용**

`_del_prefix`를 `response_cache.invalidate_pattern` 대신 Valkey 직접 SCAN으로 교체. 키 패턴 `{prefix}*`로 통일.

```python
# 수정 후
from app.cache.valkey import get_valkey
r = get_valkey()
keys = list(r.scan_iter(f"{prefix}*"))
if keys:
    count += r.delete(*keys)
```

**검증**: `test_invalidate_search_cache_for_document` PASSED ✅

---

### [VULN-003] CitationPopup — 오버레이 배경 클릭 시 포커스 함정 ✅ 수정 완료

| 항목 | 내용 |
|------|------|
| **심각도** | Medium |
| **분류** | WCAG 2.1 SC 2.1.2 (No Keyboard Trap) |
| **파일** | `frontend/src/components/chat/ChatWindow.tsx` |

**취약점 설명**

초기 Citation 팝업 구현에서 팝업이 열릴 때 포커스가 팝업 내부로 이동하지 않았고 Escape 키 핸들러가 없었다. 스크린 리더 사용자 또는 키보드 사용자가 팝업을 닫을 수 없어 키보드 함정이 발생할 수 있었다.

**수정 내용**

```tsx
// 팝업 열릴 때 닫기 버튼에 자동 포커스
useEffect(() => {
    if (activeCitation) {
        popupCloseRef.current?.focus();
    }
}, [activeCitation]);

// Escape 핸들러
useEffect(() => {
    if (!activeCitation) return;
    const handleKey = (e: KeyboardEvent) => {
        if (e.key === "Escape") setActiveCitation(null);
    };
    document.addEventListener("keydown", handleKey);
    return () => document.removeEventListener("keydown", handleKey);
}, [activeCitation]);

// role="dialog" + aria-modal
<div role="dialog" aria-modal="true" aria-labelledby="citation-popup-title">
```

---

## 3. 추가 수정 완료 (Low)

### [VULN-004] ConversationList / ChatWindow — `window.confirm` → ConfirmDialog ✅ 수정 완료

| 항목 | 내용 |
|------|------|
| **심각도** | Low |
| **분류** | OWASP A05:2021 (Security Misconfiguration) — UX 보안 |
| **파일** | `frontend/src/components/chat/ConversationList.tsx`, `ChatWindow.tsx` |

**취약점**

`window.confirm`은 브라우저 기본 대화상자이며 WCAG 2.1 미준수(포커스 관리 부재, 스타일 불가), 일부 브라우저 임베디드 환경(iframe)에서 차단된다.

**수정 내용**

`ConfirmDialog` 컴포넌트 신규 작성 (`frontend/src/components/common/ConfirmDialog.tsx`):
- `role="alertdialog"` + `aria-modal` + `aria-labelledby` / `aria-describedby`
- 열릴 때 취소 버튼(안전한 기본 선택)에 자동 포커스
- Escape 키로 취소
- `destructive=true` 시 확인 버튼 빨간색

`ConversationList`, `ChatWindow` 양쪽 `window.confirm` → `ConfirmDialog` 교체.

---

### [VULN-005] RAGCache LRU — `datetime.utcnow()` Deprecation ✅ 수정 완료

| 항목 | 내용 |
|------|------|
| **심각도** | Low |
| **분류** | OWASP A06:2021 (Vulnerable and Outdated Components) — 코드 품질 |
| **파일** | `backend/app/cache/rag_cache.py` |

**수정 내용**

`datetime.utcnow()` → `datetime.now(timezone.utc)` 전체 교체 (3곳). `from datetime import datetime, timedelta, timezone` import 추가.

---

## 4. 추가 검사 항목

### LLM 보안 (OWASP LLM Top 10)

| 검사 항목 | 결과 | 비고 |
|-----------|------|------|
| LLM01 (Prompt Injection) | ✅ 통과 | PromptBuilder 내 `=== 현재 질문 ===` 섹션으로 쿼리 격리. 사용자 입력이 시스템 프롬프트 앞에 삽입 불가 |
| LLM02 (Insecure Output Handling) | ✅ 통과 | 프론트엔드 `whitespace-pre-wrap` 텍스트 렌더링. dangerouslySetInnerHTML 미사용 |
| LLM06 (Sensitive Information Disclosure) | ✅ 통과 | ContextWindowManager ACL 검증으로 타인 대화 컨텍스트 주입 차단 |
| LLM07 (Insecure Plugin Design) | ✅ 통과 | CitationReuseService는 score 조작만 수행, 외부 명령 없음 |
| LLM09 (Overreliance) | ✅ 권고 반영 | "근거 없음" 배지로 citation 없는 응답 사용자에게 경고 |

### SQL Injection

`ConversationRepository`, `TurnRepository` 모두 `%s` 파라미터 바인딩 사용. 문자열 포맷팅 없음 ✅

### XSS

| 검사 항목 | 결과 |
|-----------|------|
| 메시지 콘텐츠 렌더링 | ✅ `whitespace-pre-wrap` 텍스트 — HTML 이스케이프됨 |
| Citation snippet 렌더링 | ✅ `{activeCitation.snippet}` — React 자동 이스케이프 |
| 대화 제목 렌더링 | ✅ `{conversation.title}` — React 자동 이스케이프 |

---

## 5. 종합 판정

| 항목 | 결과 |
|------|------|
| Critical/High 취약점 잔여 | **0건** |
| Medium 취약점 잔여 | **0건** |
| Low 취약점 잔여 | **0건** |
| **최종 판정** | **승인** (발견된 5건 전체 수정 완료) |
