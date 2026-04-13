# Task 13-6. 데이터 보존 정책

## 1. 작업 목적

Mimir에서 생성되는 각종 데이터의 보존 기간과 삭제 정책을 수립하고 구현한다.

이 작업의 목표는 다음과 같다.

- 감사 로그 보존 기간 정책 및 자동 삭제 구현
- 버전 데이터 보존 정책 (삭제된 문서 포함)
- 개인정보 보존 기간 정책 (GDPR/PIPA 고려)
- Stale 데이터 정리 배치 통합 관리

---

## 2. 작업 범위

### 포함 범위

- 데이터 유형별 보존 기간 정책 정의
- 자동 삭제 배치 Job 구현
- 개인정보 삭제 요청 처리 (사용자 탈퇴)
- Admin 보존 정책 설정 화면

### 제외 범위

- 법적 컴플라이언스 심층 검토 (법무 담당)
- 암호화 at-rest (인프라 수준)

---

## 3. 주요 구현 대상

### 3-1. 데이터 유형별 보존 기간

| 데이터 | 기본 보존 기간 | 삭제 방식 | 설정 가능 여부 |
|-------|------------|---------|------------|
| 감사 로그 (audit_logs) | 2년 | 자동 삭제 배치 | Admin 설정 |
| RAG 대화 이력 (rag_messages) | 1년 | 자동 삭제 배치 | 사용자 선택 |
| Stale 청크 (is_current=false) | 7일 | Phase 10 Cleanup Job | Admin 설정 |
| 삭제된 문서 (soft delete) | 90일 | 자동 물리 삭제 배치 | Admin 설정 |
| LLM 사용 로그 (llm_usage_logs) | 1년 | 자동 삭제 배치 | 고정 |
| 사용자 계정 (탈퇴) | 탈퇴 즉시 | 즉시 익명화 | 고정 |
| 백그라운드 Job 이력 | 90일 | 자동 삭제 배치 | Admin 설정 |

---

### 3-2. 자동 삭제 배치 Job

```python
class DataRetentionJob:
    job_type = "DATA_RETENTION_CLEANUP"

    def execute(self, payload):
        results = {}

        # 1. 감사 로그 정리
        retention_days = settings.AUDIT_LOG_RETENTION_DAYS  # 기본 730일
        cutoff = now() - timedelta(days=retention_days)
        deleted = db.execute(
            "DELETE FROM audit_logs WHERE created_at < :cutoff",
            {"cutoff": cutoff}
        ).rowcount
        results["audit_logs_deleted"] = deleted

        # 2. 만료된 RAG 대화 이력 정리
        retention_days = settings.RAG_CONVERSATION_RETENTION_DAYS  # 기본 365일
        cutoff = now() - timedelta(days=retention_days)
        deleted = db.execute(
            "DELETE FROM rag_conversations WHERE updated_at < :cutoff",
            {"cutoff": cutoff}
        ).rowcount
        results["rag_conversations_deleted"] = deleted

        # 3. 삭제된 문서 물리 삭제 (소프트 삭제 후 90일)
        cutoff = now() - timedelta(days=90)
        deleted = db.execute(
            "DELETE FROM documents WHERE deleted_at < :cutoff AND deleted_at IS NOT NULL",
            {"cutoff": cutoff}
        ).rowcount
        results["documents_purged"] = deleted

        AuditService.record("DATA_RETENTION_CLEANUP", results)
        return results
```

#### 스케줄 등록

```
스케줄: 매일 새벽 4시 (0 4 * * *)
Job 타입: DATA_RETENTION_CLEANUP
```

---

### 3-3. 사용자 탈퇴 처리 (개인정보 삭제)

```python
def process_user_withdrawal(user_id: str):
    # 1. 즉시 익명화 (재식별 불가)
    db.execute("""
        UPDATE users SET
          email = :anon_email,
          name = '탈퇴한 사용자',
          password_hash = NULL,
          deleted_at = now()
        WHERE id = :user_id
    """, {"anon_email": f"deleted_{user_id}@deleted", "user_id": user_id})

    # 2. 세션/토큰 무효화
    TokenBlacklist.invalidate_all(user_id)

    # 3. 사용자 생성 콘텐츠 처리 (선택)
    #    - 문서: 소유권 이전 또는 삭제 (정책에 따라)
    #    - 감사 로그: 보존 (법적 요건) — user_id만 익명화

    AuditService.record("USER_WITHDRAWAL_PROCESSED", {
        "user_id": user_id,
        "anonymized": True,
    })
```

---

### 3-4. 보존 정책 설정 Admin API

```
GET /api/admin/retention-policy
Response: 각 데이터 유형별 현재 보존 기간 설정

PUT /api/admin/retention-policy
Body: {"audit_log_retention_days": 730, "rag_conversation_retention_days": 365, ...}

POST /api/admin/retention-policy/run-now
Response: 즉시 정리 배치 실행 (Job 생성)
```

---

## 4. 산출물

1. 데이터 유형별 보존 기간 정책 문서
2. DataRetentionJob 구현
3. 사용자 탈퇴 처리 구현 (익명화)
4. 보존 정책 Admin API
5. 정책 설정 Admin UI (간단한 설정 화면)

---

## 5. 완료 기준

- 각 데이터 유형별 보존 정책이 정의되어 있다
- DataRetentionJob이 매일 실행되어 만료 데이터를 삭제한다
- 사용자 탈퇴 시 개인정보가 즉시 익명화된다
- Admin에서 보존 기간을 설정할 수 있다

---

## 6. Codex 작업 지침

- 감사 로그는 보존 기간 내에서는 절대 삭제하지 않으며, 보존 기간은 최소 1년 이상으로 제한한다
- 개인정보 삭제는 물리 삭제 대신 익명화 방식을 채택하여 데이터 무결성(외래 키 등)을 유지한다
- 보존 기간 설정 변경 시 감사 이벤트를 기록한다
- 삭제 배치는 대량 DELETE 대신 배치 크기를 제한하여 DB 부하를 줄인다 (기본 1000건씩)
