# Task 13-5. 백업 및 복구 체계

## 1. 작업 목적

데이터 손실에 대비하는 자동화된 백업 체계를 구축하고 복구 절차를 문서화한다.

이 작업의 목표는 다음과 같다.

- PostgreSQL 자동 백업 설정 (RPO < 24시간)
- 백업 검증 자동화 (복구 테스트)
- 오브젝트 스토리지(첨부파일) 백업
- 복구 절차 문서화 (RTO < 4시간)
- 백업 보존 정책 설정

---

## 2. 작업 범위

### 포함 범위

- PostgreSQL 백업 설정 (pg_dump / WAL 아카이빙)
- 오브젝트 스토리지 백업 설정
- 백업 복구 테스트 자동화
- 복구 절차 Runbook 작성
- 백업 모니터링 알림

### 제외 범위

- 멀티 리전 복제 (고가용성 — 규모별 별도)
- 실시간 복제 / Standby DB (운영 환경별 별도)

---

## 3. 주요 구현 대상

### 3-1. PostgreSQL 백업 전략

| 백업 유형 | 주기 | 보존 기간 | 방법 |
|---------|------|---------|------|
| 전체 백업 | 매일 새벽 2시 | 30일 | pg_dump |
| 증분 백업 (WAL) | 연속 | 7일 | WAL 아카이빙 |
| 주간 스냅샷 | 매주 일요일 | 3개월 | pg_dump |

```bash
# 일일 백업 스크립트 예시
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="mimir_backup_${DATE}.sql.gz"

pg_dump -h $DB_HOST -U $DB_USER $DB_NAME | gzip > /backup/$BACKUP_FILE
# 오브젝트 스토리지에 업로드
aws s3 cp /backup/$BACKUP_FILE s3://${BACKUP_BUCKET}/daily/
# 로컬 파일 삭제
rm /backup/$BACKUP_FILE

# 30일 이전 백업 삭제
aws s3 ls s3://${BACKUP_BUCKET}/daily/ | \
  awk '{print $4}' | \
  head -n -30 | \
  xargs -I{} aws s3 rm s3://${BACKUP_BUCKET}/daily/{}
```

---

### 3-2. 오브젝트 스토리지 백업

문서 첨부파일, 업로드된 이미지 등:

```
원본 버킷 → 백업 버킷 동기화
  - 주기: 매일 1회
  - 방법: 버킷 복제 (S3 Cross-Region Replication) 또는 rclone sync
  - 보존: 30일 버전 관리
```

---

### 3-3. 복구 절차 (Runbook)

#### DB 복구 (전체 장애 시)

```
1. 백업 파일 목록 확인
   aws s3 ls s3://${BACKUP_BUCKET}/daily/ --recursive

2. 최신 백업 다운로드
   aws s3 cp s3://${BACKUP_BUCKET}/daily/{최신파일} ./restore.sql.gz

3. PostgreSQL 재설치 또는 신규 인스턴스 생성

4. 복구 실행
   gunzip -c restore.sql.gz | psql -h $NEW_DB_HOST -U $DB_USER $DB_NAME

5. pgvector 확장 활성화 확인
   psql -c "CREATE EXTENSION IF NOT EXISTS vector;"

6. 애플리케이션 연결 설정 변경 후 재시작

7. 기능 확인 (Health Check + 문서 조회 테스트)
```

예상 소요 시간: 1~4시간 (데이터 규모에 따라)

---

### 3-4. 백업 복구 테스트 자동화

월 1회 자동으로 복구 테스트 실행:

```python
class BackupVerificationJob:
    job_type = "VERIFY_BACKUP"

    def execute(self, payload):
        # 1. 최신 백업 다운로드
        backup = download_latest_backup()

        # 2. 임시 DB에 복구
        temp_db = create_temp_db()
        restore_to_db(backup, temp_db)

        # 3. 기본 쿼리 검증
        doc_count = temp_db.query("SELECT COUNT(*) FROM documents").scalar()
        assert doc_count > 0, "문서 테이블이 비어 있음"

        chunk_count = temp_db.query("SELECT COUNT(*) FROM document_chunks").scalar()

        # 4. 임시 DB 삭제
        temp_db.drop()

        AuditService.record("BACKUP_VERIFIED", {
            "doc_count": doc_count,
            "chunk_count": chunk_count,
            "backup_file": backup.filename,
        })
```

---

### 3-5. 백업 실패 알림

백업 실패 시 즉시 알림:

```
알림 조건:
  - 지난 24시간 내 백업 완료 기록 없음 → WARNING
  - 백업 스크립트 오류 발생 → CRITICAL
  - 오브젝트 스토리지 업로드 실패 → CRITICAL
```

---

## 4. 산출물

1. 자동 백업 스크립트 (DB + 오브젝트 스토리지)
2. 백업 Cron 설정
3. 복구 절차 Runbook (DB 전체 복구, 부분 복구)
4. 백업 복구 테스트 자동화 Job
5. 백업 실패 알림 설정

---

## 5. 완료 기준

- 매일 자동 백업이 실행되고 오브젝트 스토리지에 저장된다
- 백업 복구 테스트가 성공적으로 통과한다
- 복구 Runbook을 따라 담당자 없이도 복구 가능하다
- 백업 실패 시 알림이 발송된다
- RPO < 24시간, RTO < 4시간 목표 달성 확인

---

## 6. Codex 작업 지침

- 백업 스크립트는 환경 변수에서 자격증명을 읽으며 코드에 하드코딩하지 않는다
- 백업 완료 여부를 Phase 7 Admin 대시보드에서 확인 가능하도록 백업 이벤트를 기록한다
- 복구 테스트는 운영 DB가 아닌 별도 임시 인스턴스에서 수행하여 운영 데이터에 영향이 없도록 한다
