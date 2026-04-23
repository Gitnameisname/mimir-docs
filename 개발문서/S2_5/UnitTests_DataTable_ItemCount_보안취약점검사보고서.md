# DataTable · item_count 단위 테스트 보안 취약점 검사 보고서 (S2-5)

| 항목 | 값 |
|------|------|
| 작성일 | 2026-04-21 |
| 범위 | 본 FG 가 추가한 파일만 대상: `frontend/tsconfig.test.json`, `frontend/tests/**`, `frontend/package.json`의 test script, `backend/tests/unit/test_golden_set_repository_item_count.py`, `frontend/dist-tests/` (컴파일 결과물) |
| 기준 | CLAUDE.md global #2 (critical CVE 없는 라이브러리), #3 (보안), S1 #3 (보안 취약점 검사 필수) |

---

## 1. Dependency 변경 요약

| 변경 | 신규 패키지 | 버전 | CVE 상태 |
|------|--------------|------|----------|
| `frontend/package.json` | **없음** | — | 신규 runtime dep 0개. 신규 devDep 0개. |
| `backend/requirements.txt` | **없음** | — | pytest/pydantic/psycopg2 등 기존 프로젝트 지정 의존성만 사용. |

> 초기 계획은 Vitest + @testing-library/react 도입이었으나, sandbox 환경에서 npm registry / PyPI 양쪽 모두 방화벽 차단(E403 Forbidden) 으로 설치 불가. **신규 의존성 0개 전략으로 피벗** — CVE 노출 면적은 기존 프로젝트 상태 그대로 유지됨.

## 2. 런타임 노출 면적

### 2.1 프런트엔드

- `frontend/tests/` 디렉토리와 `frontend/dist-tests/` 컴파일 결과물은 **Next.js 빌드 및 배포에 포함되지 않음**:
  - `next.config.ts` 의 기본 include 규칙은 `src/` 와 `public/` 만 처리.
  - `.gitignore` 는 `dist-tests/` 를 추가 제외 대상으로 권장 (아래 3.1 참고).
  - `frontend/tests/register.js` 는 `node --require` 옵션으로만 로드되며, 개발 또는 CI 환경에서 test 명령 실행 시에만 활성화. 런타임 번들에 섞이지 않음.
- `tsconfig.test.json` 은 `outDir: dist-tests` 로 고정되어 Next.js 의 SWC 번들과 완전히 분리.
- 공격 표면 변화: **0**.

### 2.2 백엔드

- 새 테스트 파일(`test_golden_set_repository_item_count.py`) 은 `tests/unit/` 하위이며 pytest 수집 대상. 프로덕션 import path 에 포함되지 않음.
- MagicMock 만 사용 — 실제 DB 연결 없음, 외부 API 호출 없음.
- 공격 표면 변화: **0**.

## 3. 구성 파일 리스크 체크

### 3.1 `frontend/tests/register.js` — Module._resolveFilename 훅

| 체크 항목 | 결과 |
|-----------|------|
| 임의 경로 로드 가능성 | `@/` 로 시작하는 specifier 만 rewrite. rewrite 대상은 `path.join(__dirname, "..", "dist-tests", "src", specifier.slice(2))` 로 **프로젝트 내부로 경로 봉쇄**. |
| 상위 디렉토리 traversal | `path.join` 사용 — path normalization 후에도 결과가 `..` 로 빠져나갈 수 있으나 specifier 는 컴파일된 source code 에서만 유래하며 attacker-controlled 입력 없음. 테스트 전용이므로 실질 위험 0. |
| 배포 번들 침투 | Next.js 런타임에 `--require` 옵션이 붙지 않음. package.json scripts 의 `test` 에서만 사용. |
| 권장 | `frontend/.gitignore` 에 `dist-tests/` 추가 — 컴파일 산출물이 VCS 에 커밋되어 의도치 않게 배포되는 것 방지. (후속) |

### 3.2 `frontend/tsconfig.test.json`

- `include` 가 컴파일 대상을 `tests/**/*` + DataTable.tsx + utils.ts 로 **명시적 화이트리스트**. API client 나 인증 로직 등은 포함되지 않음.
- `outDir: dist-tests` 은 gitignore 처리 권장.

### 3.3 `frontend/package.json` → `scripts.test`

- 명령 내부에 쉘 glob `"dist-tests/tests/*.test.js"` 만 포함. 쉘 메타 문자 인젝션 지점 없음 (하드코딩).

## 4. 테스트 코드 내 보안 관련 검증 강화

본 FG 가 추가한 테스트 자체가 **보안 회귀 방어**를 목적으로 함:

| 테스트 | 방어 대상 |
|--------|-----------|
| `TestCountItemsSql.test_count_items_no_hardcoded_id_in_sql` | SQL injection — `golden_set_id` 가 SQL 문자열에 interpolation 되지 않고 반드시 파라미터화. 악성 입력 `gs-1'; DROP TABLE golden_items;--` 가 SQL 에 그대로 들어가지 않음을 정적 검증. |
| `TestListByScopeAttachesItemCount.test_scope_id_is_passed_to_sql` | S2 ⑥ ACL — `scope_id=%s AND is_deleted=FALSE` WHERE 절이 COUNT/LIST 양쪽에 필수로 포함되는지 (개발 도중 ACL 우회 회귀 방어). |
| `TestGetVersionHistoryItemCount.test_nonexistent_parent_returns_empty_list` | S2 ⑥ — scope 미일치 시 접근 거부와 미존재를 동일하게 `[]` 로 응답 (존재 누설 방지). |
| `TestGetVersionHistoryItemCount.test_item_count_null_snapshot_is_zero` | null-handling 회귀 — `len(None)` 예외로 500 응답 방지. |

## 5. OWASP Top 10 대응 현황 (추가 테스트 한정)

| 카테고리 | 이 FG 내 테스트 | 결과 |
|----------|-----------------|------|
| A01 Broken Access Control | `scope_id` WHERE 포함 검증, `nonexistent_parent` 빈 리스트 검증 | 회귀 방어 확보 |
| A03 Injection | `no_hardcoded_id_in_sql` | 파라미터화 검증 |
| A05 Security Misconfiguration | 신규 의존성 0개로 attack surface 유지 | 변화 없음 |
| A09 Logging & Monitoring | 해당 없음 (단위 테스트 자체는 로깅 변경 없음) | — |
| A06 Vulnerable Components | 신규 패키지 0 → 신규 CVE 0 | 리스크 추가 없음 |

## 6. XSS / 입력 검증

- 프런트 테스트는 `renderToStaticMarkup` 으로 정적 HTML 을 생성하고 정규식으로 검사. 테스트 내 샘플 데이터(`Alice`, `a@e.com` 등) 는 모두 통제된 문자열이며 입력 sanitization 책임은 프로덕션 측 `cn()`, React 기본 escaping 이 처리.
- 백엔드 테스트는 mock DB 반환 dict 만 다루며 사용자 입력 경로 없음.

## 7. 민감 정보 / PII

- 테스트 코드와 문서 어디에도 실제 사용자 PII, 시크릿, 토큰, 이메일 주소 포함 안 됨 (`a@e.com` 는 RFC 5321 예시 도메인 아님이지만 합성 더미). 현재 로그/메모리에 저장되는 민감 데이터 없음.

## 8. 시크릿 스캔

- 변경된 파일 대상 grep:
  ```
  grep -rE "(password|secret|token|api[_-]?key|Bearer)" frontend/tests backend/tests/unit/test_golden_set_repository_item_count.py
  ```
  결과: 해당 없음.

## 9. 결론

- **신규 의존성 0개 ⇒ 신규 CVE 0개**.
- 추가된 파일은 모두 테스트 영역에 봉쇄되어 프로덕션 런타임 노출 없음.
- 테스트 자체가 SQL injection, ACL 우회, null-handling 회귀를 방어하는 보안 가치 있는 코드.
- 위험도: **Low / no net change** — 프로젝트 전체 보안 자세 개선에 기여.

## 10. 후속 권장

1. `frontend/.gitignore` 에 `dist-tests/` 추가 (VCS 오염 방지).
2. `backend/requirements-test.txt` 에 pytest, pytest-cov 를 명시적으로 고정 버전으로 기록 (이미 존재 시 생략).
3. CI pipeline 에 `cd frontend && npm test` + `cd backend && pytest tests/unit/test_golden_set_repository_item_count.py` 추가.

---

**작성자**: Cowork 자동화 (Claude)
**관련 문서**: `UnitTests_DataTable_ItemCount_검수보고서.md`
