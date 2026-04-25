# task2-6 — Vault Import (옵시디언 zip, A안 일회성)

**작성일**: 2026-04-24
**Phase / FG**: S3 Phase 2 / FG 2-6
**상태**: 착수 대기
**담당**: 백엔드 (파서 · 보안) + 프런트 (업로드 UX · 진행률 · 리포트)
**예상 소요**: 4~5일
**선행 산출물**:
- `docs/개발문서/S3/phase2/산출물/Pre-flight_실측.md`
- `task2-1.md` ~ `task2-3.md` — 컬렉션/폴더/태그/백링크 테이블이 이미 존재해야 수집 가능

---

## 1. 작업 목적

옵시디언 vault 의 zip 업로드를 1회성으로 받아, **markdown + YAML frontmatter + `[[링크]]` + `#태그` + 폴더 구조** 를 Mimir 의 **ProseMirror 트리** + 파생 테이블 (tags / document_links / folders / document_folder) 로 변환한다. 업로드 시 **Scope Profile 필수** (기본 `private`). 변환 후 Mimir 내부 편집으로만 작업 (양방향 동기화 없음). zip bomb / PII 누출에 대한 방어를 명시.

---

## 2. 작업 범위

### 2.1 포함

1. **DB 스키마 (Alembic revision `s3_p2_vault_imports`)**
   - `vault_imports (id UUID pk, owner_id UUID FK users, uploaded_filename VARCHAR(500), bytes_original BIGINT, bytes_extracted BIGINT, file_count INT, status VARCHAR(16), scope_profile_id UUID FK scope_profiles, started_at TIMESTAMPTZ, finished_at TIMESTAMPTZ, report JSONB)`
   - `status ∈ {'pending','running','succeeded','failed','cancelled'}`
   - 인덱스: `vault_imports(owner_id, created_at DESC)`

2. **백엔드 — 업로드 / 파서 / 변환**
   - `POST /vault-imports` (multipart/form-data)
     - fields: `file: UploadFile`, `scope_profile_id: UUID` (필수, 기본 private 도 명시적으로)
     - 업로드 직후 `vault_imports` row `pending` 생성, 파일은 오브젝트 스토리지 임시 경로 저장
     - 백그라운드 워커가 파싱 시작 (BackgroundTasks 또는 별도 queue)
   - `GET /vault-imports/{id}` — 상태 + report
   - `POST /vault-imports/{id}/cancel` — running 에서만
   - `services/vault_import_service.py`:
     - zip 안전 처리 → markdown 각 파일 파싱 → ProseMirror doc 변환 → documents/versions 생성 → tags / document_links / folders / document_folder 파생 동기화
     - 폴더 구조 → folders 트리로 재현 (이 import 안에서만 만든 폴더 tag `"imported_from": "<import_id>"` metadata)

3. **zip 안전 처리 (zip bomb / path traversal / entry bomb 방어)**
   - 압축 해제 **전** 체크:
     - zip 파일 크기 상한 (기본 100 MB, config)
     - 엔트리 수 상한 (기본 10,000, config)
   - 압축 해제 **중** 체크 (streaming):
     - 개별 파일 크기 상한 (기본 10 MB)
     - 총 압축 해제 크기 상한 (기본 500 MB)
     - **압축 비율 상한** (entry 의 `compress_size * 100 < file_size` 이면 거부 — 100:1 초과 = 폭탄)
   - **경로 traversal 방어**:
     - `zipfile.ZipInfo.filename` 에서 `..` 포함 또는 절대경로 거부
     - 정규화 후 루트 벗어나는 경로 거부
   - **심볼릭 링크 거부**: zip 스펙상 symlink 엔트리 존재 → external_attr 체크
   - 의존: Python 표준 `zipfile` (추가 의존성 없음)
   - 구현 참고: `backend/app/services/vault_zip_safety.py` util 모듈 분리 + pytest 공격 케이스 10+ 건

4. **markdown → ProseMirror 변환**
   - `services/markdown_to_prosemirror.py` 신설
   - 지원 요소: heading(#,##,###), paragraph, bulletList, orderedList, codeBlock, blockquote, horizontalRule, text, bold/italic/code/link marks
   - **hashtag 인라인 mark**: 정규식 매칭으로 `HashtagMark` 부여 (TipTap 과 동일 스펙)
   - **wikilink 인라인 mark**: `[[제목]]` / `[[제목|별명]]` 패턴 매칭
   - **YAML frontmatter** 파싱:
     - 파일 시작의 `---` 블록을 `PyYAML` (의존성 추가 필요 — 본 FG 블로커 체크) 로 파싱
     - `tags: [...]` → `documents.metadata.tags` 에 저장
     - 기타 키는 `documents.metadata.obsidian_frontmatter` 에 원본 보관
   - NodeId 자동 부여: 변환 시점에 모든 block 레벨 노드에 `node_id: UUID` 생성 (TipTap NodeId extension 과 동일 포맷)

5. **PII 스캔**
   - `services/pii_scanner.py` 신설
   - 탐지 대상 (최소):
     - 주민등록번호 (한국): `\d{6}[-]\d{7}` + checksum 검증
     - 이메일: RFC 5322 단순 정규식
     - 전화번호: `010-?\d{4}-?\d{4}` 등 한국 휴대폰/유선
     - 신용카드 번호: Luhn 검증 포함
   - 각 파일별로 건수 카운트, 실제 값은 저장 금지 → **마스킹 후** report 에 샘플 snippet 저장 (앞뒤 context 20 자 + 값 마스킹 `***`)
   - `vault_imports.report.pii` 에 집계
   - 옵션: 업로드 폼에서 "PII 자동 마스킹" 체크 시 변환된 ProseMirror doc text 에도 마스킹 적용

6. **프런트엔드 — 업로드 UX**
   - `/vault-imports` 페이지 (관리자만 또는 owner 본인만)
   - 드롭존 + Scope Profile 선택 드롭다운 + PII 자동 마스킹 체크박스
   - 업로드 후 진행 상태 폴링 (1초 간격, pending/running/succeeded/failed/cancelled)
   - 완료 후 report 렌더: 가져온 문서 수, 생성한 폴더 수, 태그 수, 백링크 수, **PII 감지 건수** (종류별)
   - 실패 시 원인 코드 (zip_bomb / path_traversal / file_too_large / entry_limit / parse_error) + 재시도 버튼

7. **AI 품질 평가 보고서 대상 아님** (Phase 2 개발계획서 §FG 2-6 명시) — 단, import 후 RAG 재벡터화는 기존 경로 (`vectorization_service`) 로

### 2.2 제외

- 옵시디언 플러그인/테마/자산(이미지) 대규모 처리 — 본 FG 에서는 **markdown + frontmatter + 폴더** 만. 이미지는 attachments 폴더가 있으면 별도 처리는 Phase 3 이후 (단, `![[image.png]]` mark 는 **placeholder** 로 변환하고 report 에 "미변환 자산 N 개" 표시)
- 양방향 동기화 — A안 확정
- 부분 재시도 (파일 단위) — succeeded/failed 전체 단위만

### 2.3 하드코딩 금지

- 상한값 (zip_size, entry_count, file_size, total_size, compression_ratio) 전부 환경변수 / config. 코드에 직접 숫자 금지

---

## 3. 선행 조건

- FG 2-1 ~ 2-3 완료 — folders / document_links / tags 테이블이 이미 존재해야 함
- `PyYAML` 라이선스 / CVE 점검 + 설치 (현재 백엔드 의존성 확인 필요. CLAUDE.md 전역 §2 critical 취약점 여부)
- 오브젝트 스토리지 임시 경로 (기존 프로젝트에서 S3/MinIO 쓰면 그 path. 폐쇄망 기본은 local disk 로 off switch)

---

## 4. 구현 단계

### Step 1 — zip 안전 처리 util

1. `vault_zip_safety.py` 구현
2. pytest: 공격 케이스 ≥ 10 건
   - 42.zip (재귀 zip bomb)
   - 단일 큰 파일 (압축 비 100:1 초과)
   - 엔트리 수 초과
   - path traversal `../../etc/passwd`
   - absolute path `/etc/passwd`
   - symlink entry
   - null byte 파일명
   - 중복 파일명 (덮어쓰기 공격)
   - 유니코드 정규화 공격 (UTF-8 multi-byte)
   - 정상 vault (파일 20 개, 폴더 3 개)

### Step 2 — markdown → ProseMirror 변환기

1. `markdown_to_prosemirror.py` — 기본 요소 지원
2. frontmatter 파싱 (PyYAML)
3. HashtagMark / WikiLinkMark 정규식 재사용
4. NodeId 자동 부여
5. pytest: 단위 ≥ 25 건

### Step 3 — PII 스캐너

1. `pii_scanner.py`
2. 탐지 규칙 + Luhn (카드), 주민번호 checksum
3. 마스킹 함수 `mask_pii(text) -> (masked_text, findings)`
4. pytest: 각 PII 유형별 2 건씩 ≥ 10 건

### Step 4 — vault_import_service

1. DB schema Alembic
2. `services/vault_import_service.py` — 전체 오케스트레이션
3. 워커: `BackgroundTasks` + Idempotency (같은 파일 해시 진행 중 중복 차단)
4. report 구조 확정 후 JSONB 저장
5. pytest integration: 샘플 vault (20 파일) → documents N 생성, tags/links/folders 반영

### Step 5 — 라우터

1. `api/v1/vault_imports.py` — POST/GET/cancel
2. 업로드 시 scope_profile_id 필수 validation
3. 권한: owner 본인 또는 admin

### Step 6 — 프런트 업로드 페이지

1. `/vault-imports` 드롭존 + 폼 + 진행 폴링
2. report 렌더 (PII 종류별 카운트, sample snippet)
3. 실패 원인 코드별 메시지

### Step 7 — UI 검수 / 반응형

- UI 리뷰 ≥ 1회

### Step 8 — 검수 / 보안 보고서

- 일반 + 재검수
- **보안 보고서 최상위 FG — zip bomb / PII / path traversal / 권한 우회 각 섹션 필수**
- **"뷰 ≠ 권한 준수 확인"**: 업로드 시 지정한 scope_profile_id 를 벗어난 사용자는 import 된 문서를 볼 수 없음 (FG 2-0 필터로 자연 보장)

---

## 5. 상한값 기본 (config, override 가능)

| 키 | 기본 |
|----|------|
| VAULT_IMPORT_MAX_ZIP_BYTES | 100 * 1024 * 1024 (100 MB) |
| VAULT_IMPORT_MAX_ENTRY_COUNT | 10000 |
| VAULT_IMPORT_MAX_FILE_BYTES | 10 * 1024 * 1024 (10 MB) |
| VAULT_IMPORT_MAX_TOTAL_EXTRACTED_BYTES | 500 * 1024 * 1024 (500 MB) |
| VAULT_IMPORT_MAX_COMPRESSION_RATIO | 100 (100:1) |
| VAULT_IMPORT_WORKER_TIMEOUT_SEC | 1800 (30 분) |

---

## 6. API 계약 요약

| 메서드 | 경로 | 설명 |
|-------|------|-----|
| POST | /vault-imports | multipart: file + scope_profile_id + options |
| GET | /vault-imports/{id} | 상태 + report |
| POST | /vault-imports/{id}/cancel | running 에서만 |
| GET | /vault-imports?page= | 본인 import 목록 |

---

## 7. 성공 기준

- [ ] zip 안전 처리 pytest ≥ 10 건 녹색 (공격 전부 차단)
- [ ] markdown → ProseMirror 변환 단위 ≥ 25 건
- [ ] PII 스캐너 ≥ 10 건
- [ ] 샘플 vault (≥ 20 파일, 폴더 3 층) import 성공 — Phase 개발계획서 §5 의 게이트
- [ ] PII 리포트 생성 (감지된 건 있을 때 마스킹 샘플 포함)
- [ ] ACL 우회 없음 — 지정 scope 밖 사용자는 결과 미노출
- [ ] 업로드 UX 진행률 / 실패 코드별 메시지 / 재시도
- [ ] pytest 신규 ≥ 40 건 (zip + parser + pii + integration)
- [ ] node:test 신규 ≥ 12 건 (업로드 폼, 폴링, report 렌더)

---

## 8. 리스크

| 리스크 | 대응 |
|-------|-----|
| zip bomb 탐지 오탐 (정상 고압축 vault) | 압축 비 상한 100:1 은 일반 md 에서는 발생 어려움. 발생 시 config 완화 경로 제공 + owner 에게 명시적 오류 |
| PII 오탐 / 누락 | Luhn / 주민번호 checksum 으로 오탐 감소. 누락은 v2 에서 ML 기반 확장 |
| 대형 vault 업로드 중 timeout | 30분 상한. 초과 시 cancelled 처리 + 재업로드 안내 |
| PyYAML 보안 이슈 (`yaml.load` 대 `yaml.safe_load`) | **`safe_load` 만 사용**. `load` 호출은 lint rule 로 차단 |
| 업로드 파일 임시 경로 정리 누락 | finally 블록에서 삭제 + orphan cleanup 스크립트 (운영 도구) |
| 폴더 트리 재현 시 기존 folders 와 충돌 | import 전용 루트 폴더 `_imported/<import_id>/` 아래로 배치 + 사용자 확정 후 이동 |

---

## 9. 폐쇄망 / 운영 고려

- 외부 fetch 금지 — markdown 이 `http://...` 링크 포함해도 서버는 fetch 안 함 (단순 link mark 로만 저장)
- 업로드된 zip 임시 경로는 로컬 디스크 (폐쇄망 기본). S3 쓸 때는 서버측 암호화 필수
- PII 감지 결과는 DB 에 마스킹 후 저장. 원문은 저장 안 함

---

*작성: 2026-04-24 | FG 2-6 Vault Import A안*
