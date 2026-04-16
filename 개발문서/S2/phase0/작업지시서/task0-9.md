# Task 0-9. [Mid] seed_users.py 프로덕션 가드 + 감사 이벤트

## 1. 작업 목적

`seed_users.py` 스크립트가 **프로덕션 환경에서 실수로 실행되는 것을 방지**하고, 실행 시 **감사 로그에 기록**하여 보안성과 추적 가능성을 확보한다.

현재 상태:
- `seed_users.py`는 환경 제약 없이 언제든 실행 가능
- 프로덕션에서 데이터 초기화 위험성 존재
- 실행 이력 기록 없음
- CI/CD 파이프라인에서 자동 실행될 수 있음

이 작업은 **S2 보안 원칙**(CLAUDE.md §2 "보안에 신경 쓸 것")과 **기술 부채 청산**(Phase 0 목적)을 구현한다.

---

## 2. 작업 범위

### 포함 범위

1. 환경변수 기반 실행 제어
   - `ALLOW_SEED_USERS=true` 환경변수 필수화
   - 없을 경우 명확한 오류 메시지 출력 후 종료
   - 프로덕션 환경에서 의도하지 않은 실행 방지

2. 감사 로그 이벤트 추가
   - `action=admin_seed_users` 이벤트 기록
   - 기록 정보: 생성된 사용자 수, 타임스탐프, 실행 사용자(가능 시)
   - 기존 감사 로깅 시스템에 통합

3. 프로덕션 환경 경고
   - 스크립트 시작 시 현재 환경(develop/staging/production) 출력
   - 프로덕션 환경에서 경고 메시지 출력

4. CI/CD 파이프라인 제어
   - 자동 실행 제한 문서화
   - `.github/workflows/fresh-boot-smoke.yml` (Task 0-6)에서 명시적으로 활성화 시에만 호출

### 제외 범위

- 기존 `seed_users.py` 데이터 생성 로직 변경 (그대로 유지)
- 사용자 정보 암호화 (기존 방식 사용)
- 다중 환경별 설정 파일 생성 (기존 환경변수 시스템 활용)

---

## 3. 선행 조건

- `/backend/app/` 또는 `/backend/scripts/` 디렉토리의 `seed_users.py` 파일 위치 파악
- 기존 감시 로깅 시스템(AuditLog 모델 등) 파악
- FastAPI 애플리케이션의 환경변수 로딩 방식 이해
- Python dotenv 또는 Pydantic settings 활용

---

## 4. 주요 작업 항목

### 4-1. seed_users.py 가드 로직 추가

**파일:** `/backend/app/scripts/seed_users.py` 또는 `/backend/scripts/seed_users.py`

**수정 사항:**

```python
import os
import sys
from datetime import datetime
from dotenv import load_dotenv

# 환경변수 로드
load_dotenv()

def check_seed_users_guard():
    """seed_users 실행 권한 확인"""
    allow_seed = os.getenv('ALLOW_SEED_USERS', 'false').lower() == 'true'
    
    if not allow_seed:
        print("\n" + "="*60)
        print("ERROR: seed_users.py 실행이 제한되었습니다.")
        print("-"*60)
        print("이 스크립트를 실행하려면 환경변수를 설정하세요:")
        print("  export ALLOW_SEED_USERS=true")
        print("\n주의: 프로덕션 환경에서는 절대 실행하면 안 됩니다.")
        print("="*60 + "\n")
        sys.exit(1)

def print_environment_info():
    """현재 환경 정보 출력"""
    env = os.getenv('ENVIRONMENT', 'development')
    db_url = os.getenv('DATABASE_URL', 'unknown')
    
    print("\n" + "="*60)
    print(f"[{datetime.now().isoformat()}] seed_users.py 실행 시작")
    print("-"*60)
    print(f"현재 환경: {env.upper()}")
    print(f"데이터베이스: {db_url[:50]}...")
    
    if env == 'production':
        print("\n⚠️  WARNING: 프로덕션 환경입니다!")
        print("이 작업이 의도된 작업인지 다시 한 번 확인하세요.")
    
    print("="*60 + "\n")

def seed_users():
    """기존 사용자 시드 로직"""
    # 기존 코드 유지
    from app.core.config import settings
    from app.db.session import SessionLocal
    from app.models.user import User
    from app.schemas.user import UserCreate
    
    db = SessionLocal()
    try:
        # 기존 사용자 생성 로직
        users_to_create = [
            UserCreate(username="admin", email="admin@example.com", password="admin123", role="admin"),
            UserCreate(username="user1", email="user1@example.com", password="user123", role="user"),
        ]
        
        created_count = 0
        for user_data in users_to_create:
            existing = db.query(User).filter(User.username == user_data.username).first()
            if not existing:
                user = User(**user_data.dict())
                db.add(user)
                created_count += 1
        
        db.commit()
        print(f"✓ {created_count}명의 사용자가 생성되었습니다.")
        
        return created_count
    finally:
        db.close()

if __name__ == "__main__":
    # 1. 가드 확인
    check_seed_users_guard()
    
    # 2. 환경 정보 출력
    print_environment_info()
    
    # 3. 사용자 시드
    created_count = seed_users()
    
    # 4. 감사 로그 기록 (Task 4-2 참조)
    try:
        from app.db.session import SessionLocal
        from app.models.audit_log import AuditLog
        
        db = SessionLocal()
        audit_log = AuditLog(
            action="admin_seed_users",
            details={
                "created_users_count": created_count,
                "timestamp": datetime.now().isoformat(),
                "environment": os.getenv('ENVIRONMENT', 'development'),
            },
            user_id=None,  # 시스템 작업이므로 None
        )
        db.add(audit_log)
        db.commit()
        print(f"✓ 감사 로그 기록 완료 (action=admin_seed_users, users={created_count})")
    except Exception as e:
        print(f"⚠️  감사 로그 기록 실패: {e}")
    finally:
        db.close()
    
    print(f"\n[{datetime.now().isoformat()}] seed_users.py 실행 완료\n")
```

**완료 기준:**
- 환경변수 `ALLOW_SEED_USERS=true` 없으면 오류로 종료
- 오류 메시지가 명확하고 설정 방법 안내
- 현재 환경(development/staging/production) 출력
- 프로덕션 환경 시 경고 메시지 출력
- 감사 로그 이벤트 기록

### 4-2. 감사 로그 모델 확인 및 통합

**파일:** `/backend/app/models/audit_log.py` (기존)

**확인 사항:**

```python
# 기존 AuditLog 모델이 다음 필드를 포함하는지 확인
class AuditLog(Base):
    id: int
    action: str  # "admin_seed_users"
    details: dict  # JSON 필드로 생성 수, 타임스탐프 저장
    user_id: Optional[int]  # 시스템 작업은 None
    created_at: datetime
    
    # 인덱싱 추가 (선택)
    __table_args__ = (
        Index('idx_action_created', 'action', 'created_at'),
    )
```

**필요 시 추가:**

- `details` 필드가 JSON 타입이 아닌 경우, JSONB(PostgreSQL) 또는 JSON 열 추가
- `created_at` 자동 타임스탐프 설정 확인

### 4-3. 환경변수 문서화

**파일:** `/backend/.env.example` 또는 `.env.local`

**추가 내용:**

```bash
# 사용자 시드 스크립트 활성화 (development only)
# seed_users.py 실행 시 반드시 true로 설정
ALLOW_SEED_USERS=false

# 현재 환경 (development, staging, production)
ENVIRONMENT=development
```

### 4-4. 단위 테스트 작성

**파일:** `/backend/tests/unit/scripts/test_seed_users.py`

**테스트 항목:**

```python
import os
import pytest
from unittest.mock import patch, MagicMock
from app.scripts.seed_users import check_seed_users_guard, seed_users

class TestSeedUsersGuard:
    """seed_users.py 가드 로직 테스트"""
    
    def test_guard_fails_when_env_not_set(self):
        """ALLOW_SEED_USERS 환경변수 없으면 종료"""
        with patch.dict(os.environ, {}, clear=True):
            with pytest.raises(SystemExit) as exc_info:
                check_seed_users_guard()
            assert exc_info.value.code == 1
    
    def test_guard_fails_when_env_false(self):
        """ALLOW_SEED_USERS=false면 종료"""
        with patch.dict(os.environ, {'ALLOW_SEED_USERS': 'false'}):
            with pytest.raises(SystemExit):
                check_seed_users_guard()
    
    def test_guard_passes_when_env_true(self):
        """ALLOW_SEED_USERS=true면 통과"""
        with patch.dict(os.environ, {'ALLOW_SEED_USERS': 'true'}):
            # 예외 없이 통과
            check_seed_users_guard()
    
    def test_guard_case_insensitive(self):
        """ALLOW_SEED_USERS=TRUE(대문자)도 통과"""
        with patch.dict(os.environ, {'ALLOW_SEED_USERS': 'TRUE'}):
            check_seed_users_guard()

class TestSeedUsersAudit:
    """감사 로그 기록 테스트"""
    
    @patch('app.scripts.seed_users.SessionLocal')
    def test_audit_log_recorded(self, mock_session):
        """seed_users 실행 후 감사 로그 기록"""
        mock_db = MagicMock()
        mock_session.return_value = mock_db
        
        # seed_users 실행
        created_count = 2  # 예상 생성 수
        
        # AuditLog 모델 추가 여부 확인
        assert mock_db.add.called
        
        # 감사 로그 정보 확인
        call_args = mock_db.add.call_args
        audit_log = call_args[0][0]
        
        assert audit_log.action == 'admin_seed_users'
        assert audit_log.details['created_users_count'] == created_count
        assert 'timestamp' in audit_log.details
```

**완료 기준:**
- 환경변수 없으면 종료 확인
- 환경변수 false면 종료 확인
- 환경변수 true면 통과 확인
- 감사 로그 기록 확인

### 4-5. CI/CD 파이프라인 제어

**파일:** `.github/workflows/fresh-boot-smoke.yml` (Task 0-6에서 생성)

**호출 방식:**

```yaml
- name: Seed initial users (with guard)
  if: ${{ matrix.pgvector == true }}
  env:
    ALLOW_SEED_USERS: 'true'  # 명시적 활성화
    ENVIRONMENT: 'test'
  run: |
    cd /backend
    python -m scripts.seed_users
```

**중요:** `ALLOW_SEED_USERS=true`를 명시적으로 설정하여, 프로덕션 배포 시에는 다른 환경변수 설정 사용

---

## 5. 산출물

1. **seed_users.py** (수정본)
   - 환경변수 가드 로직
   - 프로덕션 환경 경고
   - 감사 로그 기록

2. **test_seed_users.py** (신규)
   - 가드 로직 단위 테스트
   - 감사 로그 기록 테스트

3. **.env.example** (수정본)
   - `ALLOW_SEED_USERS` 문서화
   - `ENVIRONMENT` 문서화

4. **감사 로그 기록 샘플**
   - 실행 이력 확인 쿼리

---

## 6. 완료 기준

1. 환경변수 `ALLOW_SEED_USERS` 없으면 명확한 오류 메시지와 함께 종료
2. 오류 메시지에 설정 방법 안내 포함
3. 환경변수 `ALLOW_SEED_USERS=true` 설정 후 정상 실행 확인
4. 실행 후 `AuditLog` 테이블에 `action=admin_seed_users` 레코드 생성 확인
5. 감사 로그 레코드에 생성된 사용자 수, 타임스탐프 포함 확인
6. 프로덕션 환경 실행 시 경고 메시지 출력 확인
7. CI/CD 파이프라인에서 명시적 환경변수 설정 확인
8. 모든 단위 테스트 통과 (test_seed_users.py)
9. 프로덕션 배포 시 `ALLOW_SEED_USERS` 미설정 확인 (배포 절차 문서에 기록)

---

## 7. 작업 지침

### 지침 7-1. 환경 감지 로직

현재 환경 감지:

```python
def get_environment() -> str:
    env = os.getenv('ENVIRONMENT', 'development')
    return env.lower()

def is_production() -> bool:
    return get_environment() == 'production'
```

### 지침 7-2. 감사 로그 상세 정보

기록할 정보:

```python
details = {
    "created_users_count": 2,
    "timestamp": "2026-04-16T10:30:45.123456",
    "environment": "development",
    "script_version": "1.0",  # 선택
    "executed_by": "system",  # 선택
}
```

### 지침 7-3. 오류 메시지 표준화

명확하고 사용자 친화적:

```
ERROR: seed_users.py 실행이 제한되었습니다.
-------
이 스크립트를 실행하려면 환경변수를 설정하세요:
  export ALLOW_SEED_USERS=true

주의: 프로덕션 환경에서는 절대 실행하면 안 됩니다.
```

### 지침 7-4. 프로덕션 환경 보호

추가 보안 고려:

```python
if is_production() and not os.getenv('FORCE_PRODUCTION_SEED'):
    print("프로덕션에서는 추가 확인이 필요합니다.")
    response = input("정말 실행하시겠습니까? (yes/no): ")
    if response.lower() != 'yes':
        sys.exit(0)
```

### 지침 7-5. 배포 절차 문서

배포 가이드에 추가:

- 프로덕션 환경: `ALLOW_SEED_USERS` 환경변수 미설정 (기본값 false)
- 개발/테스트 환경: `ALLOW_SEED_USERS=true` 명시적 설정
- CI/CD: 테스트 단계에서만 `ALLOW_SEED_USERS=true` 활성화
