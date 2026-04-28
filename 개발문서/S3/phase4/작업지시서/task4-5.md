# task4-5 — Capability Manifest 확장 (FG 4-5)

**작성일**: 2026-04-28
**Phase / FG**: S3 Phase 4 / FG 4-5 (이월 — 조건부)
**상태**: **착수 대기 — 사용자 합의 필요** (Phase 4 1라운드 종결 후, Phase 4 §2.2)
**담당**: 백엔드 + 프론트엔드 (설계 Claude / 구현 Claude / 리뷰 Codex — extended)
**예상 소요**: 2~3일
**Approver**: `@최철균` (P1 — Meta-1, 제45조)
**선행 산출물**:
- Phase 4 1라운드 (FG 4-0 ~ 4-4) 모두 종결
- 1라운드 R1~R5 회귀 모두 녹색
- `산출물/MCP_도구_매니페스트.json` 정합 확인
**후행**: FG 4-6 (L2 draft 쓰기) 가 본 FG 의 `default_enabled` / `requires` / `policy_profile` 필드를 사용하여 노출 정책을 표현

---

## 1. 작업 목적

Phase 4 1라운드의 4 manifest 필드 (`risk_tier` / `maturity` / `status` / `exposure_policy`) 위에, 운영 정책 / 클라이언트 호환 / 의존성 표현을 위한 5 확장 필드를 추가한다.

| 필드 | 의미 | 예시 |
|---|---|---|
| `default_enabled` | 신규 ScopeProfile 생성 시 자동 등록 후보 여부 | `True` (search_documents) / `False` (resolve_document_reference) |
| `requires` | 본 도구가 정상 동작하기 위해 필요한 다른 capability/도구 | `["streaming_supported"]` 또는 `["search_documents"]` |
| `preferred_use` | 권장 사용 사례 (자연어, 외부 클라이언트 안내용) | `"자연어 참조를 document_id 로 정규화 — disambiguation 필요 시 candidates 순회"` |
| `policy_profile` | 운영 정책 그룹 — `read_safe` / `write_audited` / `admin_only` 등 | `"read_safe"` (L0/L1 read 전체) |
| `streaming_supported` | SSE 스트리밍 (`tools/call/stream`) 가능 여부 | `True` (search_documents) / `False` (verify_citation) |

본 FG 는 **새 도구 추가 없음** — 기존 8 도구의 manifest 확장만. FG 4-6 (L2 draft 쓰기) 이 본 FG 의 `default_enabled=False` + `policy_profile="write_audited"` 를 사용해 노출 정책을 표현.

---

## 2. 작업 범위

### 2.1 포함

#### 2.1.1 schemas/mcp.py — 5 신규 필드 + Literal 타입

```python
DefaultEnabled = bool
Requires = list[str]  # 다른 capability 또는 도구 이름
PolicyProfile = Literal[
    "read_safe",          # L0/L1 read — 신규 ScopeProfile default
    "write_audited",      # L2 write — idempotency + human approval + audit (FG 4-6)
    "admin_only",         # 관리자 전용 (현재 MCP 미노출 — REST_ADMIN_ONLY)
    "experimental",       # 정밀도 검증 미완 — default 비활성
]
StreamingSupported = bool
PreferredUse = str  # 자연어 — 길이 제약 없음 (운영자 결정)
```

`TOOL_SCHEMAS` 의 8 항목에 5 필드 부착 (제안 default):

| # | 도구 | default_enabled | requires | preferred_use (요약) | policy_profile | streaming_supported |
|---|---|---|---|---|---|---|
| 1 | search_documents | ✅ | [] | "FTS/벡터 문서 검색 — top_k≤50" | read_safe | ✅ |
| 2 | fetch_node | ✅ | [] | "노드 단위 본문 조회" | read_safe | ❌ |
| 3 | verify_citation | ✅ | [] | "5중 검증 — citation_basis 명시" | read_safe | ❌ |
| 4 | mimir.vectorization.status | ❌ | [] | "RAG 답변 부재 진단용" | read_safe | ❌ |
| 5 | read_annotations | ✅ | [] | "인라인 주석 + 멘션 조회" | read_safe | ❌ |
| 6 | search_nodes | ✅ | [] | "노드 그래뉼래리티 검색 — citation 후보" | read_safe | ❌ |
| 7 | read_document_render | ✅ | [] | "rendered_text + node_anchors — citation_basis=rendered_text" | read_safe | ❌ |
| 8 | resolve_document_reference | ❌ | [] | "자연어 참조 → document_id (experimental)" | experimental | ❌ |

`default_enabled=False` 의 의미: 신규 ScopeProfile 생성 시 본 도구는 `allowed_tools` 에 자동 추가되지 **않음**. 운영자가 명시 등록 필요.

#### 2.1.2 dump_mcp_manifest 갱신

`backend/scripts/dump_mcp_manifest.py` 의 `build_manifest()` 결과에 5 필드 포함:

```python
{
    "name": "search_documents",
    "risk_tier": "L0",
    "maturity": "stable",
    "status": "enabled",
    "exposure_policy": "MCP_ENABLED",
    # FG 4-5 신규
    "default_enabled": True,
    "requires": [],
    "preferred_use": "...",
    "policy_profile": "read_safe",
    "streaming_supported": True,
}
```

CI drift 게이트 (FG 4-4 §2.1.6) 가 자동으로 새 필드 정합 검증.

#### 2.1.3 `mcp_exposed_public_view` 정책 결정

본 FG 의 5 필드 중 **외부 `tools/list` 응답에 노출할 것** vs **운영자 전용** 분리:

| 필드 | 외부 노출 후보 | 결정 |
|---|---|---|
| `default_enabled` | ❌ — 운영 정책 정찰 | 운영자 endpoint 만 |
| `requires` | ✅ — 클라이언트가 의존성 인지 가능 | 외부 노출 |
| `preferred_use` | ✅ — 외부 안내용 (도구등급화_매핑.md §4 옵션 A 의 description 확장) | 외부 노출 |
| `policy_profile` | ❌ — 운영 그룹화 정책 | 운영자 endpoint 만 |
| `streaming_supported` | ✅ — 클라이언트가 stream endpoint 호출 가능 여부 분기 | 외부 노출 |

`mcp_exposed_public_view` 가 외부 응답에 위 ✅ 필드만 포함하도록 갱신.

#### 2.1.4 운영자 endpoint 신설 — `GET /admin/mcp/manifest`

본 endpoint 가 8 도구의 **전체 manifest** (운영자 전용 필드 포함) 를 반환. Admin UI 가 동적 fetch 용으로 사용 → `KNOWN_MCP_TOOLS` 하드코딩 제거 (FG 4-0 §7 #6 후속).

```python
@router.get("/admin/mcp/manifest")
def admin_mcp_manifest(actor: ActorContext = Depends(resolve_current_actor)):
    _require_admin(actor)
    return {"tools": [_full_view(s) for s in TOOL_SCHEMAS], "extensions": MIMIR_EXTENSIONS}
```

응답: `default_enabled` / `requires` / `preferred_use` / `policy_profile` / `streaming_supported` 모두 포함.

#### 2.1.5 ScopeProfile.create() 갱신 — `default_enabled` 통합

신규 ScopeProfile 생성 시:
- `allowed_tools` 명시 입력 → 그대로 사용 (현행 동작)
- `allowed_tools` 미입력 → **`default_enabled=True` 도구만 자동 등록**

이는 FG 4-0 §2.1.6 의 default-deny 정책 완화 — 운영자 명시 결정 시점에 옵트인 가능.

```python
def create(self, *, ..., allowed_tools: Optional[list[str]] = None, use_defaults: bool = False):
    if allowed_tools is None and use_defaults:
        from app.schemas.mcp import default_enabled_tool_names
        allowed_tools = list(default_enabled_tool_names())
    # ...
```

`use_defaults` flag 는 명시적 옵트인 — 기본 False 로 default-deny 보존.

#### 2.1.6 Admin UI — 동적 manifest fetch

`AdminScopeProfilesPage.tsx` 의 `KNOWN_MCP_TOOLS` 상수 제거, `/admin/mcp/manifest` fetch 로 대체:

```ts
const { data: manifest } = useQuery({
  queryKey: ["admin", "mcp-manifest"],
  queryFn: () => mcpManifestApi.get(),
});
const tools = manifest?.tools ?? [];
// 토글 그룹: tools.filter(t => t.is_mcp_exposed)
```

도구 토글 UI 에 `policy_profile` 색상 + `requires` 의존성 + `preferred_use` 툴팁 추가.

#### 2.1.7 회귀 테스트

`backend/tests/integration/test_mcp_manifest_extended_fg45.py`:
- 8 도구 모두 5 신규 필드 보유
- `default_enabled=False` 도구 (`mimir.vectorization.status` / `resolve_document_reference`) 가 자동 등록 제외됨
- `mcp_exposed_public_view` 가 외부 노출 후보만 포함 (default_enabled / policy_profile 제외)
- `requires` 가 비어있지 않은 도구 시뮬레이션 (가짜 도구) — 의존성 미충족 시 거부 동작 (별 라운드 — 본 FG 에서는 메타만)
- `/admin/mcp/manifest` endpoint 응답 완전성 (전체 필드) + admin only 권한
- ScopeProfile.create(use_defaults=True) → default_enabled 도구만 등록

`backend/tests/integration/contract_drift/test_manifest_drift.py` 갱신:
- 신규 5 필드가 manifest JSON 에 반영됨
- dump_mcp_manifest --check 가 새 필드 drift 도 감지

#### 2.1.8 `default_enabled_tool_names` / `tools_by_policy_profile` 헬퍼

`schemas/mcp.py` 에 추가:

```python
def default_enabled_tool_names() -> list[str]:
    """default_enabled=True 인 노출 도구 이름 (정렬)."""
    return sorted(
        s["name"] for s in TOOL_SCHEMAS
        if is_tool_mcp_exposed(s) and s.get("default_enabled", False)
    )

def tools_by_policy_profile(profile: str) -> list[str]:
    """policy_profile 별 노출 도구 이름."""
    return sorted(
        s["name"] for s in TOOL_SCHEMAS
        if is_tool_mcp_exposed(s) and s.get("policy_profile") == profile
    )
```

함수도서관 §1.7-fg45 등재.

### 2.2 제외 (이월)

- `requires` 의 런타임 의존성 검사 (도구 호출 시점에 requires 가 충족되었는지 검증) — 별 라운드. 본 FG 는 메타데이터 표면화만.
- `policy_profile` 별 런타임 정책 분기 (예: `write_audited` 그룹은 모두 idempotency 강제) — FG 4-6 에서 일부 구현, 일반화는 별 라운드.
- 외부 클라이언트가 `requires` / `streaming_supported` 를 인지하도록 SDK 업데이트 — 별도 SDK 작업.

### 2.3 하드코딩 금지 재확인

- 5 필드 default 값은 TOOL_SCHEMAS 가 단일 정본. 헬퍼 (`default_enabled_tool_names` 등) 가 그것을 순회.
- Admin UI 의 도구 목록은 동적 fetch — 정적 상수 (`KNOWN_MCP_TOOLS`) 영구 제거.
- `policy_profile` Literal 은 schema 정본 — 다른 곳에서 자유 문자열 사용 금지.

---

## 3. 선행 조건

- Phase 4 1라운드 (FG 4-0 ~ 4-4) 종결 + 운영자 합의로 본 FG 진입 결정
- 운영자 `@최철균` P1 승인 — 신규 manifest 필드 + 운영자 endpoint 신설
- TOOL_SCHEMAS 8 도구 정합 (FG 4-2 종결 후 상태)

---

## 4. 구현 단계

### Step 1 — schemas/mcp.py 확장

1. Literal 타입 4종 (`PolicyProfile` 등) 정의
2. TOOL_SCHEMAS 8 항목에 5 필드 default 부착
3. 헬퍼 (`default_enabled_tool_names` / `tools_by_policy_profile`) 추가
4. 단위 테스트: default 정합, Literal 강제

### Step 2 — public view 갱신

1. `mcp_exposed_public_view` 가 `requires` / `preferred_use` / `streaming_supported` 포함
2. 운영자 전용 (`default_enabled` / `policy_profile`) 외부 응답 미노출 검증

### Step 3 — Admin manifest endpoint

1. `app/api/v1/admin_mcp.py` (또는 기존 admin 라우터) 에 `GET /admin/mcp/manifest` 추가
2. `_require_admin` 게이트
3. 전체 필드 응답 + audit log
4. 단위 테스트: admin / non-admin 권한 분기

### Step 4 — ScopeProfile.create use_defaults 옵션

1. Repository.create signature 확장 (`use_defaults: bool = False`)
2. allowed_tools 미입력 + use_defaults=True 시 default_enabled 도구 자동 등록
3. 단위 테스트: 기본 default-deny 동작 보존 + use_defaults=True 시 자동 등록

### Step 5 — dump_mcp_manifest + drift 게이트 갱신

1. `build_manifest()` 가 5 신규 필드 포함
2. JSON 재생성 + commit
3. drift 테스트가 새 필드 정합 검증

### Step 6 — Admin UI 동적 manifest

1. `mcpManifestApi.get()` API client 추가
2. `KNOWN_MCP_TOOLS` 상수 제거 → useQuery 로 대체
3. `policy_profile` 별 색상 + `requires` 표시 + `preferred_use` 툴팁
4. UI 리뷰 1회 (CONTRIBUTING.md §5)

### Step 7 — 검수 / 보안 보고서

- `FG4-5_검수보고서.md` — 5 필드 정합 + 외부 노출 정책 + ScopeProfile 자동 등록
- `FG4-5_보안취약점검사보고서.md` — `policy_profile` / `default_enabled` 누설 검사 + admin endpoint 권한 우회 시도

---

## 5. API 계약 변경 요약

| 메서드 | 경로 | 변경 |
|---|---|---|
| GET | /mcp/tools (외부) | 응답에 `requires` / `preferred_use` / `streaming_supported` 추가. `default_enabled` / `policy_profile` 비노출 |
| GET | /admin/mcp/manifest | **신규** — 8 도구 전체 manifest (운영자 전용) |
| POST | /admin/scope-profiles (Create) | `use_defaults: bool` 옵션 추가. 기본 false (default-deny 보존) |

기존 호출자는 영향 0 — 응답에 추가 필드만 출현. 5 필드 default 값으로 backward-compat.

---

## 6. 데이터 모델 주의사항

- DB 스키마 변경 0 — manifest 는 코드 상수
- 5 필드 default 가 시스템 boot 마다 정해진 값 — 환경별 차이 없음
- `requires` 가 빈 리스트로 시작 — 의존성 표현은 본 FG 에선 메타만, 런타임 검사는 별 라운드

---

## 7. 성공 기준

- [ ] 8 도구 모두 5 신규 필드 부착 (default 명시)
- [ ] `mcp_exposed_public_view` 가 외부 노출 후보만 포함
- [ ] `GET /admin/mcp/manifest` endpoint 동작 + admin only
- [ ] `default_enabled_tool_names` / `tools_by_policy_profile` 헬퍼 + 단위 테스트
- [ ] ScopeProfile.create(use_defaults=True) 자동 등록 회귀
- [ ] dump_mcp_manifest 가 5 필드 포함 + drift 게이트 통과
- [ ] Admin UI 가 동적 fetch 로 도구 목록 표시 + UI 리뷰 통과
- [ ] pytest 신규 ≥ 20 / 전체 베이스라인 유지
- [ ] 검수·보안 보고서 제출

---

## 8. 리스크

| 리스크 | 대응 |
|---|---|
| 외부 클라이언트가 새 필드 무시 → 의존성 인지 못함 | 추가형 변경 (기존 키 보존) — 무시해도 동작. SDK 업데이트는 별 작업 |
| `default_enabled` 정책 변경이 운영 중인 ScopeProfile 에 회귀 | use_defaults 는 신규 생성 시에만 적용. 기존 프로파일 영향 0 |
| `policy_profile` Literal 변경 시 외부 호환 깨짐 | 운영자 전용 — 외부 응답에 포함 안 함 |
| Admin UI 동적 fetch 실패 시 도구 토글 표시 실패 | useQuery 의 isError 분기로 명시적 에러 표시 + retry 버튼 |
| `requires` 빈 리스트 default — 의존성 표현이 형식적 | 본 FG 는 메타만. 런타임 검사 도입 시 별 라운드에서 default 갱신 |

---

## 9. 참조

- `Phase 4 개발계획서.md` §2.2 (FG 4-5 이월 — 1라운드 R1~R5 회귀 녹색 + 사용자 승인 시 진행)
- `task4-0.md` §2.1.3 (manifest 표준 + KNOWN_MCP_TOOLS 정적 상수 제거 후속 — FG 4-0 §7 #6)
- `task4-4.md` §2.1.6 (manifest drift 게이트 — 본 FG 가 새 필드 추가하면 drift 게이트가 자동 검증)
- `uploads/Mimir API : MCP 운영 논의 요약` (운영자 manifest 의도)
- `CONSTITUTION.md` 제5조 (Agent-Facing Contract — 본 FG 가 contract 표면화)

---

*작성: 2026-04-28 | FG 4-5 — Capability manifest 확장 (이월 / 조건부)*
