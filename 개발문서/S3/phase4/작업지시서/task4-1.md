# task4-1 — MCP read 응답 메타데이터 표준 (FG 4-1)

**작성일**: 2026-04-27
**Phase / FG**: S3 Phase 4 / FG 4-1
**상태**: 착수 대기 (FG 4-0 완료 후)
**담당**: 백엔드 (설계 Claude / 구현 Codex / 리뷰 Claude — extended)
**예상 소요**: 2~3일
**선행 산출물**:
- `task4-0` 완료 (Pre-flight + manifest 표준)
- `산출물/Pre-flight_실측.md`
- `산출물/도구등급화_매핑.md`
**후행**: task4-2 (신규 도구 3종)는 본 FG의 envelope 위에서 동작

---

## 1. 작업 목적

외부 Chatbot이 Mimir의 응답을 **명령(instruction)이 아닌 참고(evidence)로 다루도록** 강제하는 응답 envelope을 표준화한다. AI 에이전트가 문서 내용을 system prompt로 오해해 prompt injection에 노출되는 것을 구조적으로 차단하는 것이 핵심이다.

업로드 요약 §5의 5개 필드(`content_role`, `instruction_authority`, `trust_level`, `detected_risks`, `source.uri`)를 모든 MCP read 도구·리소스 응답에 의무화하고, `mimir://` URI 스키마를 확정한다.

본 FG는 **신규 기능이 아니라 기존 응답에 envelope을 추가**하는 작업이다. 따라서 호환성(외부 Chatbot이 envelope을 무시해도 기존 필드 동작 보존)이 가장 중요한 제약이다.

---

## 2. 작업 범위

### 2.1 포함

#### 2.1.1 응답 envelope 스키마 정의

`backend/app/schemas/mcp.py`에 다음을 추가한다.

```python
class MCPSourceRef(BaseModel):
    """근거 위치를 가리키는 mimir:// URI 컨테이너."""
    uri: str = Field(..., description="mimir://documents/{doc}/versions/{ver}/nodes/{node}")
    document_id: str
    version_id: Optional[str] = None
    node_id: Optional[str] = None


class MCPDetectedRisk(BaseModel):
    """prompt injection 등 탐지된 위험 항목."""
    code: str  # 예: "prompt_injection", "directive_pattern", "url_obfuscation"
    severity: Literal["info", "low", "medium", "high"]
    note: Optional[str] = None
    span: Optional[tuple[int, int]] = None  # 본문 내 좌표 (선택)


class MCPReadEnvelope(BaseModel):
    """모든 MCP read 도구·리소스 응답이 포함해야 하는 공통 envelope."""
    content_role: Literal["retrieved_evidence", "tool_metadata", "system_status"] = "retrieved_evidence"
    instruction_authority: Literal["none", "system", "user"] = "none"  # **항상 none** (R4)
    trust_level: Literal["source_document", "agent_generated", "synthetic", "unknown"] = "source_document"
    detected_risks: list[MCPDetectedRisk] = Field(default_factory=list)
    source: Optional[MCPSourceRef] = None
    # 결과 단위 메타 (선택)
    items_total: Optional[int] = None
    items_truncated: bool = False
```

`MCPResponse` (기존)에 `envelope: MCPReadEnvelope` 필드를 추가한다 — read 도구만 포함, write 도구는 별도 envelope (본 FG 외 — FG 4-6 결정 시).

#### 2.1.2 mimir:// URI 스키마 확정

URI 형식을 다음으로 고정한다.

| 대상 | URI 형식 | 예 |
|------|----------|-----|
| 문서 | `mimir://documents/{document_id}` | `mimir://documents/abc-123` |
| 특정 버전 | `mimir://documents/{document_id}/versions/{version_id}` | `mimir://documents/abc/versions/v7` |
| 특정 노드 | `mimir://documents/{document_id}/versions/{version_id}/nodes/{node_id}` | `mimir://documents/abc/versions/v7/nodes/n42` |
| 렌더링 텍스트 | `mimir://documents/{document_id}/versions/{version_id}/render` | `mimir://documents/abc/versions/v7/render` |

- `version_id`는 항상 `vN` 형태(또는 UUID — Pre-flight 결과로 정함). **`latest`는 URI에 포함하지 않는다(R3 — pinned 강제)**. 입력으로 `latest`가 들어오면 서버가 즉시 resolve 후 envelope에 resolved URI를 반환한다.
- `node_id`는 Phase 1의 stable node_id를 사용한다.

`backend/app/mcp/resources.py`의 `parse_resource_uri`를 위 형식에 맞춰 갱신한다(현재 구현이 어느 패턴을 받는지는 Pre-flight에서 확인).

#### 2.1.3 detected_risks 매핑 — prompt_injection_detector 연결

`backend/app/security/prompt_injection.py`의 `PromptInjectionDetector` 결과를 `detected_risks` 리스트로 매핑한다. 매핑 규칙:

| Detector 결과 | code | severity |
|---------------|------|----------|
| 명령어 패턴 ("ignore previous", "system:", 등) 탐지 | `directive_pattern` | high |
| URL 난독화 (homograph, punycode) | `url_obfuscation` | medium |
| 잠재적 데이터 유출 키워드 (API key 패턴 등) | `secret_leak` | high |
| 일반 이상 신호 (비정상 길이, 반복) | `anomaly` | low |

`mcp_router.py`에서 detector를 호출하는 모든 read 응답 경로에 매핑을 적용한다.

> ⚠️ **R6 (false positive 정책)**: `detected_risks`는 **차단이 아닌 경고**다. 응답 자체는 정상 반환하고 risk 목록만 첨부한다. 차단 정책은 클라이언트가 결정한다. 본 정책은 검수보고서 §"R4·R6 준수 확인"에 명시.

#### 2.1.4 기존 도구 회귀 적용

다음 도구의 응답에 envelope을 추가한다 (FG 4-0의 manifest와 함께 구조화).

| 도구 | content_role | instruction_authority | trust_level | source | detected_risks |
|------|--------------|----------------------|-------------|--------|----------------|
| `search_documents` | `retrieved_evidence` | `none` | `source_document` | (각 결과 항목별 추가) | 본문 스니펫에 적용 |
| `fetch_node` | `retrieved_evidence` | `none` | `source_document` | 노드 URI | 노드 본문에 적용 |
| `verify_citation` | `tool_metadata` | `none` | `source_document` | 검증된 노드 URI | 적용 안 함 (검증 결과는 메타) |
| `mimir.vectorization.status` | `system_status` | `none` | `unknown` | 문서 URI | 적용 안 함 |

각 도구 함수(`tool_search_documents` 등)는 본문 결과 외에 `MCPReadEnvelope`를 함께 반환한다. `mcp_router.py`가 이를 `MCPResponse.envelope`에 주입.

#### 2.1.5 결과 항목별 envelope (검색류 한정)

`search_documents`처럼 결과가 여러 개인 도구는 **상위 envelope** (응답 전체)와 **항목별 envelope** (각 결과)을 모두 가진다. 항목별 envelope은 다음 필드만 포함한다.

```python
class MCPItemEnvelope(BaseModel):
    source: MCPSourceRef
    detected_risks: list[MCPDetectedRisk] = Field(default_factory=list)
    trust_level: Literal["source_document", "agent_generated", "synthetic", "unknown"] = "source_document"
```

이를 통해 항목 단위로 위험 분석이 가능해진다. (업로드 요약 §5 후반부 합의)

#### 2.1.6 폐쇄망 동등성 (S2 ⑦)

prompt_injection_detector가 외부 모델 호출 없이 **로컬 패턴 기반**으로 동작함을 회귀 테스트로 보장 (`MIMIR_OFFLINE=1`에서도 detected_risks가 정상 채워짐). 외부 LLM 기반 탐지를 추가하더라도 off 시 degrade(false → 빈 리스트)는 허용, 응답 자체 실패는 금지.

### 2.2 제외 (이월)

- write 도구용 별도 envelope (audit trace, idempotency key 등) — FG 4-6
- envelope 결과를 외부 모니터링(Prometheus 등)에 노출 — 본 Phase 외
- detected_risks의 LLM 기반 정밀화 — 별도 작업

### 2.3 하드코딩 금지 재확인

- envelope 기본값(`content_role` 등)은 모델 정의에서 단일 정본. tool 함수가 다시 default를 박지 말 것.
- mimir:// URI 생성은 `app.mcp.resources` 또는 신설 `app.mcp.uri_builder` 모듈로 단일화. 라우터에서 문자열 인라인 금지.

---

## 3. 선행 조건

- FG 4-0 완료 (manifest 표준 + L4 차단 회귀)
- `Pre-flight_실측.md` 의 `prompt_injection_detector` 호출 경로 확인 결과 반영
- (확인) `backend/app/security/prompt_injection.py`의 `PromptInjectionDetector`/`InjectionDetectionResult` 인터페이스 안정

---

## 4. 구현 단계

### Step 1 — envelope 스키마 정의

1. `MCPSourceRef`/`MCPDetectedRisk`/`MCPReadEnvelope`/`MCPItemEnvelope` 추가
2. `MCPResponse`에 `envelope` 필드 추가 (Optional → 기본값 제공)
3. 단위 테스트: 모델 직렬화·기본값·`instruction_authority`가 `none`만 허용되는지 (R4)

### Step 2 — mimir:// URI 빌더

1. `app/mcp/uri_builder.py` 신설 — `build_doc_uri`, `build_version_uri`, `build_node_uri`, `build_render_uri`
2. `parse_resource_uri` 갱신 — 새 패턴 4종 모두 인식
3. `latest` 입력 시 자동 resolve (현재 시각 기준 latest published version → vN). resolve 함수는 기존 `versions_repository` 재사용
4. 단위 테스트: parse / build round-trip + `latest` resolve 동작 + 잘못된 형식 거부

### Step 3 — detected_risks 매핑 함수

1. `app/mcp/risk_mapper.py` 신설 — `map_injection_result(result: InjectionDetectionResult) -> list[MCPDetectedRisk]`
2. 매핑 규칙(§2.1.3) 적용
3. 단위 테스트: 각 패턴별 매핑 결과 검증

### Step 4 — 기존 도구에 envelope 적용

1. `tool_search_documents` — 결과 본문은 그대로, 추가로 `MCPReadEnvelope`(상위) + 각 결과 `MCPItemEnvelope`
2. `tool_fetch_node` — 노드 URI를 source로
3. `tool_verify_citation` — `content_role=tool_metadata`
4. `tool_vectorization_status` — `content_role=system_status`
5. `mcp_router.py`의 `tools/call` 핸들러가 도구 함수 반환값을 받아 `MCPResponse.envelope`에 주입

### Step 5 — 리소스(read) 표면

`/mcp/resources/read` 응답에도 envelope 적용. 리소스가 문서 내용을 반환하는 경우 `content_role=retrieved_evidence` + `instruction_authority=none`.

### Step 6 — 회귀 테스트

1. **R4 회귀**: 모든 read 응답이 `instruction_authority=none`을 가짐을 단위로 검증 (table-driven, 도구 4개 + 리소스 1종)
2. **호환성 회귀**: envelope 추가 후에도 기존 응답 본문 키들이 그대로 유지됨 (외부 클라이언트 호환)
3. **detected_risks 회귀**: 알려진 prompt injection 페이로드 5종을 본문에 심어 응답 envelope에 risk가 출현하는지 검증
4. **mimir:// URI 회귀**: 4 패턴 모두 build/parse round-trip 녹색
5. **`latest` resolve 회귀**: `latest` 입력 시 envelope의 `source.uri`에 `vN`이 포함됨 (R3 — pinned)

### Step 7 — 검수 / 보안 보고서

- `FG4-1_검수보고서.md` — R1~R5 준수 + R6(차단 아닌 경고) 준수 확인
- `FG4-1_보안취약점검사보고서.md` — prompt injection 시나리오 5+종 페이로드 결과 첨부

---

## 5. API 계약 변경 요약

| 메서드 | 경로 | 변경 |
|-------|------|------|
| POST | /mcp/tools/call | 응답에 `envelope` 추가 (read 도구 한정). 기존 `data` 본문 보존 |
| GET | /mcp/resources/read | 응답에 `envelope` 추가 |
| (전체) | mimir:// URI | 4 패턴 표준 확정 + `latest` 자동 resolve |

---

## 6. 데이터 모델 주의사항

- DB 스키마 변경 없음. 모든 변경은 응답 직렬화 시점.
- `content_hash` 등 기존 필드는 envelope 외부에 유지 (검증 도구 호환).

---

## 7. 성공 기준

- [ ] `MCPReadEnvelope`/`MCPItemEnvelope`/`MCPSourceRef`/`MCPDetectedRisk` 정의 + `instruction_authority`가 `none`만 강제되는 단위 테스트 녹색
- [ ] mimir:// URI 4 패턴 build/parse 단위 테스트 녹색
- [ ] `latest` 입력 → `vN` resolve 회귀 녹색 (R3 사전 보장)
- [ ] 기존 도구 4종 모두 envelope 포함, 외부 클라이언트 호환 회귀 녹색
- [ ] detected_risks 매핑 5+ 패턴 회귀 녹색
- [ ] 폐쇄망 모드(`MIMIR_OFFLINE=1`)에서 envelope 정상 채워짐
- [ ] pytest 신규 ≥ 25 / 전체 베이스라인 유지
- [ ] 검수·보안 보고서 제출

---

## 8. 리스크

| 리스크 | 대응 |
|-------|-----|
| 외부 Chatbot 일부가 미지정 키 거부(strict schema) | envelope은 기본값 포함 + 응답 본문 키 변경 없음. 호환성 회귀 |
| `instruction_authority`를 코드 어딘가에서 `system`으로 설정 | Literal 타입으로 컴파일 타임 차단 + 단위 테스트로 런타임 차단 (이중 방어) |
| `latest` resolve가 race condition으로 다른 버전 반환 | resolve 시점의 transaction 격리 수준 확인. 동시성 회귀 테스트 1건 추가 |
| detected_risks의 false positive로 정상 문서 표시 깨짐 | 차단이 아닌 경고 — 응답은 정상. 검수 보고서에서 명시 |
| risk_mapper가 외부 LLM에 의존 | 폐쇄망 회귀로 차단 — 로컬 패턴만 사용 |

---

## 9. 참조

- `Phase 4 개발계획서.md` §1.2 (R4·R6), §2.1 (FG 4-1)
- `uploads/Mimir API : MCP 운영 논의 요약` §5 (응답 메타데이터)
- `CONSTITUTION.md` 제5·17·18조 (Untrusted/Provenance)
- `backend/app/security/prompt_injection.py`
- `backend/app/mcp/resources.py`, `backend/app/api/v1/mcp_router.py`

---

*작성: 2026-04-27 | FG 4-1 — read 응답 envelope 표준*
