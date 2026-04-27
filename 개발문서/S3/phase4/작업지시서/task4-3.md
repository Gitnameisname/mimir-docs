# task4-3 — verify_citation 강화 + citation_basis 도입 (FG 4-3)

**작성일**: 2026-04-27
**Phase / FG**: S3 Phase 4 / FG 4-3
**상태**: 착수 대기 (FG 4-1 / 4-2 완료 후)
**담당**: 백엔드 (설계 Claude / 구현 Codex / 리뷰 Claude — extended)
**예상 소요**: 3~4일
**선행 산출물**:
- `task4-0` 완료 (manifest 표준)
- `task4-1` 완료 (envelope + mimir:// URI + `latest` resolve)
- `task4-2` 완료 (read_document_render → render_hash)
**후행**: task4-4 (contract drift 테스트의 (4) pinned citation 강제 검증이 본 FG 산출물 위에서 동작)

---

## 1. 작업 목적

문서 버전이 빠르게 변하는 환경에서, 인용이 **여전히 정확한지를 시간이 지난 후에도 자동으로 검증할 수 있는** 5중 검증을 도입한다. 동시에 인용 데이터의 모호성(원문 노드 vs 렌더링된 텍스트)을 `citation_basis` 필드로 명시적으로 분리한다.

5중 검증:
1. **존재성**: document/version/node 모두 존재
2. **pinned**: version_id가 `latest`가 아닌 구체 vN
3. **hash 일치**: `citation_basis`에 따라 `node_content_hash` 또는 `render_hash` 비교
4. **텍스트 포함**: 인용된 텍스트가 실제 노드/렌더 텍스트에 포함됨
5. **span 유효성**: span_offset이 노드/텍스트 길이 범위 내

본 FG는 **데이터 마이그레이션을 동반**하는 유일한 Phase 4 1라운드 작업이다. 헌법 제45조에 따라 마이그레이션 실행 시점에 사용자 승인이 필요하다.

---

## 2. 작업 범위

### 2.1 포함

#### 2.1.1 `citation_basis` 필드 도입 — Alembic revision

**revision**: `s3_p4_citation_basis`

```sql
-- 1단계: NULL 허용으로 컬럼 추가
ALTER TABLE citations
    ADD COLUMN citation_basis VARCHAR(20);

-- 2단계: 기존 행 backfill (모두 node_content로 — 안전 기본값)
UPDATE citations SET citation_basis = 'node_content' WHERE citation_basis IS NULL;

-- 3단계: NOT NULL + CHECK constraint
ALTER TABLE citations
    ALTER COLUMN citation_basis SET NOT NULL,
    ALTER COLUMN citation_basis SET DEFAULT 'node_content',
    ADD CONSTRAINT citations_citation_basis_check
        CHECK (citation_basis IN ('node_content', 'rendered_text'));

-- 인덱스: 분석 쿼리에 사용 가능 (선택)
CREATE INDEX idx_citations_citation_basis
    ON citations(citation_basis) WHERE citation_basis = 'rendered_text';
```

**dry-run 지원**: `SCHEMA_MIGRATION_DRY_RUN=1` 환경변수로 backfill 건수만 출력.

**downgrade**: 컬럼 삭제. 단, 다운그레이드 시 정보 손실(rendered_text 표지 사라짐)이 있으므로 검수 보고서에 명시.

#### 2.1.2 `citations` 테이블 스키마 모델

`backend/app/models/citation.py` (또는 기존 모델) 갱신:

```python
@dataclass
class Citation:
    ...  # 기존 필드
    citation_basis: Literal["node_content", "rendered_text"] = "node_content"
```

#### 2.1.3 `verify_citation` 도구 입력/출력 스키마 갱신

`backend/app/schemas/mcp.py`의 `VerifyCitationRequest`/`VerifyCitationData`:

**입력 (변경)**:
```python
class VerifyCitationRequest(BaseModel):
    document_id: str
    version_id: str  # vN 또는 UUID — 'latest' **금지** (R3)
    node_id: str
    content_hash: str  # citation_basis에 따라 의미 다름
    citation_basis: Literal["node_content", "rendered_text"] = "node_content"  # 신규
    quoted_text: Optional[str] = None  # 검증 4 (텍스트 포함)
    span_offset: Optional[int] = None  # 검증 5 (span 유효성)
    span_length: Optional[int] = None  # span 끝 좌표
    access_context: Optional[AccessContext] = None
```

**출력 (확장)**:
```python
class VerifyCitationData(BaseModel):
    verified: bool  # 5중 검증 모두 통과
    checks: dict  # 각 검사 결과
    # checks = {
    #   "exists": True/False,
    #   "pinned": True/False,
    #   "hash_matches": True/False,
    #   "quoted_text_in_content": True/False/None (quoted_text 미입력 시 None),
    #   "span_valid": True/False/None,
    # }
    current_hash: Optional[str] = None  # citation_basis에 따른 현재 hash
    rendered_snapshot: Optional[str] = None  # span 추출 (rendered_text 모드)
    node_snapshot: Optional[str] = None  # span 추출 (node_content 모드)
    version_resolved: Optional[str] = None  # 'latest' 거부 후 사용자에게 vN을 알려주기 위한 안내값 (None — latest는 도구 자체에서 거부)
    message: str
```

#### 2.1.4 `verify_citation` 도구 함수 갱신

`backend/app/mcp/tools.py`의 `tool_verify_citation` 5중 검증으로 재작성:

```python
def tool_verify_citation(request, actor, conn) -> VerifyCitationData:
    # R3: latest 거부
    if request.version_id == "latest":
        raise MCPError(
            MCPErrorCode.INVALID_PARAMS,
            "verify_citation은 pinned 버전(vN)만 수용합니다. latest는 인용 시점에 resolve 후 저장하세요.",
            400,
        )

    # 1. 존재성
    exists = _check_document_version_node_exists(conn, request)
    if not exists:
        return VerifyCitationData(verified=False, checks={"exists": False, ...}, message="...")

    # 2. pinned (version_id가 published vN인지 — draft 아닌지)
    pinned = _is_published_version(conn, request.version_id)

    # 3. hash 분기
    if request.citation_basis == "node_content":
        current_hash = _compute_node_content_hash(conn, request.node_id, request.version_id)
        node_text = _fetch_node_text(conn, request.node_id, request.version_id)
        text_for_check = node_text
    else:  # rendered_text
        current_hash, rendered = _compute_render_hash_and_text(conn, request.document_id, request.version_id)
        text_for_check = rendered

    hash_matches = (current_hash == request.content_hash)

    # 4. 텍스트 포함
    if request.quoted_text:
        quoted_in = (request.quoted_text in text_for_check)
    else:
        quoted_in = None

    # 5. span 유효성
    if request.span_offset is not None and request.span_length is not None:
        span_end = request.span_offset + request.span_length
        span_valid = (0 <= request.span_offset < len(text_for_check) and
                      0 < request.span_length and
                      span_end <= len(text_for_check))
    else:
        span_valid = None

    verified = exists and pinned and hash_matches and \
               (quoted_in if quoted_in is not None else True) and \
               (span_valid if span_valid is not None else True)

    # envelope (FG 4-1) — content_role=tool_metadata
    return VerifyCitationData(
        verified=verified,
        checks={
            "exists": exists,
            "pinned": pinned,
            "hash_matches": hash_matches,
            "quoted_text_in_content": quoted_in,
            "span_valid": span_valid,
        },
        current_hash=current_hash,
        ...
    )
```

#### 2.1.5 `node_content_hash` vs `render_hash` 분기

- **node_content_hash**: 기존 `nodes.content_hash` 또는 source_text의 SHA-256. ProseMirror 트리에서의 노드 본문 기반.
- **render_hash**: FG 4-2에서 도입한 `read_document_render`의 출력 hash. 렌더링된 텍스트 전체의 SHA-256.

> ⚠️ **render_hash 영구 저장 여부**: Pre-flight 결정 사항. 옵션:
> - (A) 매 검증 시 동적 계산 — 단순. 단 render 함수 결정성 + 일관성 의무.
> - (B) `versions.render_hash` 컬럼 추가 — 영구 저장. 검증 빠름. 단 render 변경(예: 마크다운 정책 변경) 시 일괄 재계산 필요.
>
> **권장**: 1라운드는 (A) — render_service의 결정성을 회귀 테스트로 보장. 성능 이슈 시 (B)로 전환.

#### 2.1.6 인용 저장 시점의 `latest` resolve (REST 측)

REST에서 인용을 저장하는 모든 경로(citations 생성)에 다음을 강제:
- 입력에 `latest`가 들어오면 즉시 vN으로 resolve 후 저장
- DB에는 vN만 들어가도록 service 계층에서 차단

이는 본 FG의 회귀 테스트로 보장 + FG 4-4의 contract drift 테스트 (4)에서 재확인.

#### 2.1.7 회귀 테스트 (5중 검증 전수)

`backend/tests/integration/test_verify_citation_v2.py`:
- 검사 1 (exists) 실패 케이스
- 검사 2 (pinned) — `latest` 입력 거부 / draft 버전 거부
- 검사 3 (hash_matches) — node_content 모드 / rendered_text 모드 / 둘 다 변조 케이스
- 검사 4 (quoted_text) — 포함 / 미포함 / quoted_text 미입력
- 검사 5 (span) — 정상 / 음수 offset / 길이 초과
- 5중 모두 통과 케이스
- ACL 우회 시도 (다른 scope의 노드 검증 시 거부)
- `citation_basis` 누락 시 default `node_content` 적용

`backend/tests/integration/test_citation_save_pinned.py`:
- REST 측에서 `latest`로 인용 저장 요청 → vN으로 resolve되어 저장됨
- DB에 `latest` 문자열이 절대 들어가지 않음

#### 2.1.8 마이그레이션 가이드

`산출물/FG4-3_마이그레이션_가이드.md`:
- dry-run 절차
- 운영 적용 절차 (비업무 시간대 권장)
- 롤백 절차 (downgrade + 정보 손실 명시)
- 운영 후 확인 쿼리 (`SELECT citation_basis, COUNT(*) FROM citations GROUP BY citation_basis;`)

### 2.2 제외 (이월)

- 인용에 multi-span 지원 (현재는 단일 span) — 별도 작업
- citation_basis별 별도 검증 도구 분기 (verify_node_citation / verify_render_citation) — 단일 verify_citation에 분기로 처리하는 것이 R5 (REST 1:1 복제 금지)에 부합
- render_hash의 `versions` 테이블 영구 저장 (옵션 B) — Pre-flight 결과에 따라 1라운드 외 처리

### 2.3 하드코딩 금지 재확인

- `citation_basis` 허용값은 모델의 Literal 타입이 단일 정본. SQL CHECK constraint와 동기화.
- hash 계산 함수는 `app.utils.hashes` (또는 신규)로 단일화. 도구 함수에서 인라인 hashlib 호출 금지.
- `latest` 거부 메시지는 일관된 i18n 키로 (사용자 언어 한국어).

---

## 3. 선행 조건

- FG 4-0 / 4-1 / 4-2 완료
- `read_document_render` 도구의 `render_hash` 결정성 회귀 녹색 (FG 4-2)
- `citations` 테이블 현 스키마 확인 (Pre-flight)
- 운영자 승인: Alembic 적용 시점 — `@최철균` (P1)

---

## 4. 구현 단계

### Step 1 — Pre-flight (Phase 4 본편 Pre-flight 외 추가 점검)

1. `citations` 테이블 현 스키마 + 행 수 확인
2. `latest` 문자열이 현재 `version_id`에 저장된 행이 있는지 검색 (있다면 backfill 대상)
3. `nodes.content_hash` 컬럼 존재 여부 + 일관성 (Phase 1 자산)
4. render_service의 결정성 — 같은 입력에 대해 같은 출력 보장 여부 빠른 검증

### Step 2 — Alembic revision 작성

1. `s3_p4_citation_basis` 작성 (3 단계)
2. dry-run 모드 분기 추가
3. downgrade 작성 + 정보 손실 경고
4. 로컬 테스트 DB에서 upgrade/downgrade 왕복 녹색

### Step 3 — 모델 + 스키마 갱신

1. `Citation` dataclass에 `citation_basis` 추가
2. `VerifyCitationRequest`/`VerifyCitationData` 갱신 (§2.1.3)
3. 단위 테스트: Pydantic 검증 + Literal 강제

### Step 4 — `tool_verify_citation` 재작성

1. 5중 검증 함수 분리 — `_check_exists`, `_check_pinned`, `_check_hash`, `_check_quoted_text`, `_check_span`
2. `latest` 거부 (R3 — 도구 진입점)
3. citation_basis 분기 (node_content vs rendered_text)
4. 각 검사 결과를 `checks` 딕셔너리로 반환 (디버깅 가시성)
5. envelope 채움 (FG 4-1 표준, content_role=tool_metadata)

### Step 5 — REST 측 `latest` resolve 강제

1. `app/services/citation_service.py` (또는 인용 저장 함수) 점검
2. 저장 직전 `latest` → vN resolve 호출
3. resolve 실패 시 에러 메시지 명확화
4. 단위 테스트: `latest` 입력 → DB에 vN 저장됨

### Step 6 — 회귀 테스트

1. `test_verify_citation_v2.py` (§2.1.7) 작성 — 12+ 시나리오
2. `test_citation_save_pinned.py` 작성 — 4+ 시나리오
3. ACL 회귀: 다른 scope의 노드 검증 시도 → 거부
4. envelope 회귀 (FG 4-1 표준 통과)

### Step 7 — AI 품질 평가 (citation 정확성 회귀)

`scripts/rag_smoke/` 기존 골든셋의 인용 검증 결과 변화 측정:
- Citation-present 지표 (S2 KPI) 변동 ≤ 0.02
- Faithfulness 변동 ≤ 0.02

`산출물/FG4-3_AI품질평가보고서.md` 작성.

### Step 8 — 마이그레이션 가이드 + 운영자 승인

1. `FG4-3_마이그레이션_가이드.md` 작성
2. 운영자(`@최철균`)에게 승인 요청 (P1 게이트)
3. 승인 후 운영 적용 — 적용 결과 확인 쿼리 첨부

### Step 9 — 검수 / 보안 보고서

- `FG4-3_검수보고서.md` — R1~R5 + R3 강제 확인 (latest 거부 케이스 + DB에 latest 미저장 회귀)
- `FG4-3_보안취약점검사보고서.md` — citation_basis 입력 변조 / hash 위조 / span overflow 시나리오

---

## 5. API 계약 변경 요약

| 메서드 | 도구 / 경로 | 변경 |
|-------|-------------|------|
| POST | /mcp/tools/call (verify_citation) | 입력에 `citation_basis`/`quoted_text`/`span_length` 추가 (모두 선택, 기본값 호환). 출력에 `checks` 딕셔너리 추가 |
| POST | /mcp/tools/call (verify_citation) | `version_id="latest"` 거부 (400 INVALID_PARAMS) |
| (REST) | citation 생성 경로 | `latest` 입력 시 vN으로 자동 resolve |

기존 호출자는 `citation_basis` 미입력 시 `node_content` 기본값으로 동작 — 호환.

---

## 6. 데이터 모델 주의사항

- Alembic revision은 단일 revision 안에서 3 단계(NULL 허용 추가 → backfill → NOT NULL)를 트랜잭션으로 묶음
- 대용량 backfill 가능성 — `citations` 행 수가 큰 경우 배치 처리 (LIMIT/OFFSET 1000)
- downgrade는 정보 손실 — 검수보고서에 강조

---

## 7. 성공 기준

- [ ] Alembic revision 적용 완료, 모든 기존 행에 `citation_basis` 채워짐
- [ ] `verify_citation`이 5중 검증 모두 수행 (`checks` 출력 확인 가능)
- [ ] **R3 강제**: `latest` 입력 즉시 거부 (도구) + DB에 `latest` 절대 저장 안 됨 (REST)
- [ ] node_content 모드와 rendered_text 모드 모두 동작
- [ ] envelope (FG 4-1) 표준 통과
- [ ] AI 품질 회귀 ≤ 0.02 변동
- [ ] pytest 신규 ≥ 25 / 전체 베이스라인 유지
- [ ] 마이그레이션 가이드 + 운영자 승인 + 적용 후 확인 쿼리
- [ ] 검수·보안·AI 품질 보고서 제출

---

## 8. 리스크

| 리스크 | 대응 |
|-------|-----|
| backfill 중 `citations` 테이블 lock 시간 | 배치 처리 + 야간 적용 |
| render_hash 비결정성 (마크다운 normalization 차이) | render_service 결정성 회귀 + 동일 입력 → 동일 hash 단위 테스트 |
| `latest` resolve 시 race condition (resolve 시점과 인용 저장 시점 사이 publish) | resolve + 저장을 단일 트랜잭션으로 / 격리 수준 점검 |
| 기존 클라이언트가 `citation_basis` 미입력 → 잘못된 기본값 적용 | default `node_content`는 안전 기본값. 검수보고서에서 명시 |
| span_length 누락 시 검증 4가 항상 통과로 판정될 위험 | quoted_text/span_length 모두 선택. 미입력 시 결과 None — verified는 강제 검사만 |
| downgrade 시 정보 손실로 운영 사고 | downgrade 가이드에 강조 + 백업 권장 |
| 마이그레이션이 production 적용 시 dry-run 결과와 차이 | dry-run을 staging에서도 실행. row count 확인 |

---

## 9. 참조

- `Phase 4 개발계획서.md` §1.2 (R3), §2.1 (FG 4-3), §3.1 (스키마)
- `uploads/Mimir API : MCP 운영 논의 요약` §6 (citation/pinned)
- `task4-1.md` (envelope), `task4-2.md` (render_hash, latest resolve)
- `CONSTITUTION.md` 제45·46·47조
- `backend/app/services/citation_service.py` (있다면)
- `backend/app/db/migrations/versions/` (Alembic 경로)

---

*작성: 2026-04-27 | FG 4-3 — verify_citation 5중 검증 + citation_basis*
