# FG 2-6 Pre-flight 갱신 메모

**작성일**: 2026-05-10
**대상**: `docs/개발문서/S3/phase2/작업지시서/task2-6.md` (2026-04-24)

## 1. 진행 상태 실측

- ❌ vault_imports 테이블 / 모델 / 서비스 / 라우터 / Frontend 페이지 — 전부 부재
- ✅ PyYAML 6.0.3 설치 (`backend/requirements.txt`) — 추가 의존성 불필요
- ✅ BackgroundTasks 패턴 활용 (evaluations / batch_extractions 선례)
- ✅ UploadFile 패턴 (golden_sets:517)
- ✅ FG 2-1~2-5 종결 — folders / document_links / tags / saved_views 파생 동기화 호출 가능
- ✅ documents.scope_profile_id (FG 2-0) 활용 가능

## 2. 갱신 항목

### 2.1 Alembic head

- 현 head: `s3_p2_saved_views` (FG 2-5 종결 후)
- 새 revision `s3_p2_vault_imports` 의 `down_revision = "s3_p2_saved_views"`

### 2.2 PyYAML 안전 정책

- `yaml.safe_load` 만 사용 — `yaml.load` 호출은 코드 리뷰에서 차단 (task2-6.md §8 R-04)
- 검수보고서에 grep 검증 결과 첨부 의무

### 2.3 워커 — BackgroundTasks 채택

- task2-6.md §2.1 (2) "BackgroundTasks 또는 별도 queue" 중 BackgroundTasks 채택
- 근거: 기존 evaluations / batch_extractions 가 이미 같은 패턴. 별도 queue (Celery/RQ) 도입은 별 인프라 결정 필요.
- 한계: process restart 시 진행 중 작업 유실. **status='running' 인 row 가 시작 시각 + WORKER_TIMEOUT 초과면 자동 'failed' 처리**하는 보강 필요 (별 라운드 운영 도구 또는 startup hook)

### 2.4 임시 파일 — 로컬 디스크 (폐쇄망 기본)

- `tempfile.NamedTemporaryFile` 또는 `tempfile.mkstemp` 사용 (Python 표준)
- 처리 완료 또는 실패 시 finally 블록에서 즉시 삭제
- task2-6.md §9 "S3 쓸 때는 서버측 암호화 필수" — 본 세션은 로컬만 (S3 옵션은 별 라운드)

### 2.5 config 상한값

task2-6.md §5 6개 키:

| 환경변수 | 기본 |
|---------|-----|
| `VAULT_IMPORT_MAX_ZIP_BYTES` | 100 MB |
| `VAULT_IMPORT_MAX_ENTRY_COUNT` | 10000 |
| `VAULT_IMPORT_MAX_FILE_BYTES` | 10 MB |
| `VAULT_IMPORT_MAX_TOTAL_EXTRACTED_BYTES` | 500 MB |
| `VAULT_IMPORT_MAX_COMPRESSION_RATIO` | 100 (100:1) |
| `VAULT_IMPORT_WORKER_TIMEOUT_SEC` | 1800 (30 분) |

`os.environ.get(...)` 직접 사용 (config.py 패턴이 부재). 단일 정본 모듈 (`app/services/vault_import_config.py`) 신설.

### 2.6 폴더 트리 재현 — `_imported/<import_id>/` 루트 격리

task2-6.md §8 R-06 — import 전용 루트 폴더로 격리 후 사용자 확정. **본 세션 1차 종결**: 폴더 격리 정책 까지만 코드에 반영. 사용자 확정 UI 는 잔여 (1차 진행 시 직접 _imported/<id>/ 루트로 폴더 트리 생성 + 별 라운드에서 이동 UI).

### 2.7 본 세션 진행 범위

| Step | 진행 |
|------|----|
| 1 zip safety | ✅ pytest ≥ 10 |
| 2 markdown → ProseMirror | ✅ pytest ≥ 25 |
| 3 PII 스캐너 | ✅ pytest ≥ 10 |
| 4 DB + service 오케스트레이션 | ✅ (단위 테스트 mock 위주) |
| 5 라우터 (POST/GET/cancel/list) | ✅ |
| 6 Frontend 업로드 페이지 | 🟡 stub or 간단 (시간 봐서) |
| 7/8 UI 리뷰 / 보안 리뷰 | 🟡 운영자 |
| 9 종결보고서 + 함수도서관 + memory | ✅ |

## 3. P1 승인 게이트

- Alembic `s3_p2_vault_imports` 적용 (운영자 환경)
- 4 보안 도메인 (zip bomb / path traversal / PII / 권한 우회) 전수 검증 — 종결보고서에 첨부
- 4 endpoint API 표면 추가
- 승인자: `@최철균`

## 4. Phase 5 정합

ProseMirror schema / mark 와 무관. task5-1 ADR 변경 없음. 단 변환된 doc 가 NodeId / HashtagMark / WikiLinkMark 포맷을 그대로 따르는지 단위 회귀 ≥ 3 (markdown → ProseMirror Step 2 안에 포함).

---

*작성: 2026-05-10 | FG 2-6 Vault Import — Pre-flight 갱신*
