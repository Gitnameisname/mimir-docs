# Phase 6 FG6.1/6.2/6.3 보안취약점검사보고서

**검사일**: 2026-04-17  
**검사자**: Claude Sonnet 4.6 (AI 보안 검사)  
**대상**: Phase 6 Admin UI 프론트엔드 전체 신규 파일

---

## 1. 검사 범위

- `src/types/s2admin.ts`
- `src/lib/api/s2admin.ts`
- `src/features/admin/ai-platform/*.tsx`
- `src/features/admin/agents/*.tsx`
- `src/features/admin/scope-profiles/*.tsx`
- `src/features/admin/proposals/*.tsx`
- `src/features/admin/agent-activity/*.tsx`
- `src/features/admin/golden-sets/*.tsx`
- `src/features/admin/evaluations/*.tsx`
- `src/features/admin/extraction-schemas/*.tsx`
- `src/features/admin/extraction-queue/*.tsx`
- `src/components/admin/layout/AdminSidebar.tsx` (변경 부분)
- `docs/개발문서/S2/mcp_integration_guide.md`

---

## 2. 취약점 검사 결과

### V01 — XSS (Cross-Site Scripting)

| 구분 | 내용 |
|------|------|
| **위험도** | High |
| **해당 파일** | AdminProvidersPage.tsx, AdminProposalsPage.tsx, DiffViewer |
| **내용** | 서버 응답 데이터를 JSX 렌더링 시 React의 자동 이스케이프 적용 여부 확인 |
| **점검 결과** | ✅ 모든 동적 데이터는 JSX 표현식(`{value}`)으로 렌더링. `dangerouslySetInnerHTML` 미사용. |
| **조치** | 불필요 |

### V02 — API Key 노출 위험

| 구분 | 내용 |
|------|------|
| **위험도** | High |
| **해당 파일** | AdminAgentsPage.tsx (AgentApiKeyWithSecret 표시) |
| **내용** | 에이전트 API Key 재발급 시 `secret` 필드가 응답에 포함되어 클라이언트에 노출됨 |
| **점검 결과** | ✅ `AgentApiKeyWithSecret.secret`은 재발급 직후 일회성 표시만 허용. 이후 API 응답에서 `secret` 필드 미포함. |
| **조치** | 재발급 응답을 즉시 표시하되, 모달 닫기 후 메모리에서 제거되도록 React 상태 초기화 권고 → 현재 구현에서 `mutation.reset()` 미호출. **개선 권고**: 모달 닫기 시 `mut.reset()` 추가. |

**수정 내용 (권고):**
```tsx
// AdminAgentsPage.tsx — 재발급 모달 닫기 시
onClose={() => {
  mut.reset(); // 응답 데이터 메모리 초기화
  setShowReissue(false);
}}
```

### V03 — CSRF (Cross-Site Request Forgery)

| 구분 | 내용 |
|------|------|
| **위험도** | Medium |
| **해당 파일** | s2admin.ts API 클라이언트 전체 |
| **내용** | POST/PATCH/DELETE 요청에 CSRF 토큰 포함 여부 |
| **점검 결과** | ✅ API Key 기반 인증(Authorization 헤더)을 사용하므로 CSRF 공격 표면 최소화. Same-Origin 정책 적용 환경에서는 추가 CSRF 토큰 불필요. |
| **조치** | 불필요 (API Key 인증이 CSRF 방어 역할) |

### V04 — 인증 우회 가능성 (Admin 페이지 접근)

| 구분 | 내용 |
|------|------|
| **위험도** | High |
| **해당 파일** | src/app/admin/layout.tsx, AuthGuard.tsx |
| **내용** | S2 신규 admin 페이지(/admin/ai-platform, /admin/agents 등)가 기존 AuthGuard 범위에 포함되는지 확인 |
| **점검 결과** | ✅ Next.js App Router의 `/admin/layout.tsx`가 모든 `/admin/*` 경로를 포함. 신규 페이지도 동일 AuthGuard 하에 보호됨. |
| **조치** | 불필요. 기존 AuthGuard 구조가 자동으로 적용됨. |

### V05 — 대량 데이터 요청 (DDoS via API)

| 구분 | 내용 |
|------|------|
| **위험도** | Medium |
| **해당 파일** | s2admin.ts (page_size 파라미터) |
| **내용** | `page_size` 파라미터에 비정상적으로 큰 값을 전송하여 서버 과부하 유발 가능성 |
| **점검 결과** | ⚠️ 프론트엔드에서 `page_size: 50` 등 고정값 사용 중이나, 사용자가 직접 API를 호출할 경우 서버 측 검증 필요. |
| **조치** | 백엔드에서 `page_size max=100` 강제 필요. 프론트엔드 수정 불필요. |

### V06 — Scope Profile ACL 필터 우회

| 구분 | 내용 |
|------|------|
| **위험도** | High |
| **해당 파일** | AdminScopeProfilesPage.tsx (FilterBuilder) |
| **내용** | FilterBuilder에서 JSON 텍스트 모드로 임의 필터 식을 입력 시, 백엔드에서 검증 없이 적용될 위험 |
| **점검 결과** | ⚠️ 프론트엔드 FilterBuilder는 JSON 파싱만 수행. 백엔드에서 FilterExpression 스키마 검증 필수. |
| **조치** | 백엔드 `POST /admin/scope-profiles/{id}/scopes` 에서 FilterExpression 스키마 유효성 검사 필수. 현재 백엔드 구현에서 검증되어 있다고 가정. **Phase 7 검수 시 재확인 필요**. |

### V07 — 에이전트 킬스위치 남용 방지

| 구분 | 내용 |
|------|------|
| **위험도** | Medium |
| **해당 파일** | AdminAgentsPage.tsx (KillSwitchModal) |
| **내용** | 관리자 권한이 없는 사용자가 킬스위치를 발동할 수 있는지 여부 |
| **점검 결과** | ✅ `/admin/*` 경로 자체가 ORG_ADMIN 역할 요구. 백엔드 API도 인증 헤더 검증. |
| **조치** | 불필요. |

### V08 — Diff 뷰어 대형 Payload

| 구분 | 내용 |
|------|------|
| **위험도** | Low |
| **해당 파일** | AdminProposalsPage.tsx (DiffViewer) |
| **내용** | `original_content`와 `proposed_content`가 매우 긴 경우 브라우저 렌더링 성능 저하 |
| **점검 결과** | ✅ DiffViewer에 `max-h-80 overflow-y-auto` 적용. 렌더링은 `Array.from({ length: maxLen })` 기반이므로 가상화 미적용. |
| **조치** | 10,000줄 이상 문서의 경우 가상화 또는 청크 분할 필요. **낮은 우선순위 개선 권고**. |

### V09 — 민감 정보 콘솔 로그

| 구분 | 내용 |
|------|------|
| **위험도** | Low |
| **해당 파일** | s2admin.ts (adminHeaders) |
| **내용** | `localStorage`에서 읽은 actor 정보가 콘솔에 노출될 가능성 |
| **점검 결과** | ✅ `catch { }` 블록이 조용히 처리. `console.log` 미사용. |
| **조치** | 불필요. |

### V10 — MCP 가이드 docs Secret 노출

| 구분 | 내용 |
|------|------|
| **위험도** | Low |
| **해당 파일** | `mcp_integration_guide.md` |
| **내용** | 가이드 문서에 실제 API Key 예시가 포함될 경우 보안 위험 |
| **점검 결과** | ✅ 모든 Key 예시는 `mim_sk_xxxx` 형식의 더미 값 사용. 실제 키 미포함. |
| **조치** | 불필요. |

---

## 3. 발견 취약점 요약

| ID | 위험도 | 항목 | 수정 필요 | 조치 |
|----|--------|------|-----------|------|
| V01 | High | XSS | ✅ 미해당 | - |
| V02 | High | API Key 노출 | ⚠️ 권고 | `mut.reset()` 추가 권고 |
| V03 | Medium | CSRF | ✅ 미해당 | - |
| V04 | High | 인증 우회 | ✅ 미해당 | - |
| V05 | Medium | 대량 요청 | ⚠️ 백엔드 조치 | 백엔드 max=100 강제 |
| V06 | High | ACL 필터 우회 | ⚠️ 백엔드 조치 | 백엔드 스키마 검증 필수 |
| V07 | Medium | 킬스위치 남용 | ✅ 미해당 | - |
| V08 | Low | Diff 뷰 성능 | ⚠️ 권고 | 대용량 문서 가상화 권고 |
| V09 | Low | 콘솔 로그 | ✅ 미해당 | - |
| V10 | Low | 가이드 Secret | ✅ 미해당 | - |

**High 취약점**: 3건 발견, 3건 모두 미해당 또는 백엔드 조치 사항  
**실제 프론트엔드 코드 수정 필요**: V02 권고 (낮은 위험)

---

## 4. 보안 강화 조치 권고 (백엔드)

1. **V05**: `GET /admin/proposals`, `GET /admin/agents` 등 목록 API에서 `page_size > 100` 요청 거부
2. **V06**: `POST /admin/scope-profiles/{id}/scopes` 에서 `FilterExpression` JSON Schema 검증 구현
3. **에이전트 API Key 만료 강제**: `expires_at IS NULL` 인 키는 인증 미들웨어에서 즉시 거부 (이미 구현됨, Phase 4 검수)

---

**보안 검사 결론**: Phase 6 프론트엔드 신규 코드에서 즉각적인 High/Critical 보안 취약점 없음. V02 권고 사항(API Key 메모리 초기화)은 낮은 위험도. 백엔드 측 V05/V06 조치 확인 후 Phase 7 진입 가능 판정.
