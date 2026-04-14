# Task 14-7. 계정 관리 UI 구현

## 1. 작업 목적

로그인한 사용자가 자신의 프로필, 비밀번호, 연결된 계정, 활성 세션을 관리할 수 있는 계정 관리 페이지를 구현한다.

이 작업의 목표는 다음과 같다.

- 프로필 조회/수정 페이지
- 비밀번호 변경 페이지
- GitLab 계정 연결/해제 관리
- 활성 세션 목록 및 강제 종료
- 계정 관리 API 백엔드 구현

---

## 2. 작업 범위

### 포함 범위

- `GET /api/v1/account/profile` — 프로필 조회 API
- `PATCH /api/v1/account/profile` — 프로필 수정 API
- `POST /api/v1/account/change-password` — 비밀번호 변경 API
- `GET /api/v1/account/oauth-accounts` — 연결된 소셜 계정 목록 API
- `POST /api/v1/account/oauth-accounts/gitlab/link` — GitLab 계정 연결 API
- `DELETE /api/v1/account/oauth-accounts/gitlab/unlink` — GitLab 계정 해제 API
- `GET /api/v1/account/sessions` — 활성 세션 목록 API
- `DELETE /api/v1/account/sessions/{session_id}` — 세션 강제 종료 API
- 계정 관리 UI 페이지 (`/account/profile`, `/account/security`, `/account/sessions`)

### 제외 범위

- 계정 삭제 (별도 정책 필요)
- 프로필 이미지 업로드 (avatar_url 직접 입력으로 대체)

---

## 3. 선행 조건

- Task 14-2~14-4 인증 API 완료
- Task 14-8 프론트엔드 인증 상태 관리 (병행 가능)
- `users_repository.py` 이해
- `oauth_accounts` 테이블 이해

---

## 4. 주요 구현 대상

### 4-1. 프로필 관리 API

```python
# GET /api/v1/account/profile
# 인증 필수 (Bearer Token)

Response 200:
{
  "id": "uuid",
  "email": "user@example.com",
  "display_name": "홍길동",
  "avatar_url": "https://...",
  "auth_provider": "local",
  "email_verified": true,
  "role_name": "AUTHOR",
  "created_at": "2026-01-01T00:00:00Z"
}

# PATCH /api/v1/account/profile
Request Body: { "display_name": "새이름", "avatar_url": "https://..." }
→ 감사 이벤트 기록
```

---

### 4-2. 비밀번호 변경 API

```python
# POST /api/v1/account/change-password

Request Body:
{
  "current_password": "OldP@ss1",
  "new_password": "NewP@ss2"
}

처리 흐름:
1. 현재 비밀번호 검증 (bcrypt)
2. 새 비밀번호 복잡도 검증
3. 새 비밀번호 해싱 → 저장
4. 현재 세션 제외, 다른 모든 RT family 폐기
5. 감사 이벤트 기록

에러:
  401: 현재 비밀번호 불일치
  422: 새 비밀번호 복잡도 미충족
```

GitLab 전용 계정(password_hash=NULL)인 경우 비밀번호 변경 비활성화.

---

### 4-3. GitLab 계정 연결/해제

```python
# GET /api/v1/account/oauth-accounts
Response: [{ "provider": "gitlab", "provider_email": "...", "created_at": "..." }]

# POST /api/v1/account/oauth-accounts/gitlab/link
→ GitLab OAuth 플로우 시작 (Task 14-4와 유사, 기존 계정에 연결)

# DELETE /api/v1/account/oauth-accounts/gitlab/unlink
처리:
1. 로컬 비밀번호가 설정되어 있는지 확인 (없으면 해제 거부)
2. oauth_accounts에서 해당 레코드 삭제
3. 감사 이벤트 기록

에러:
  400: 비밀번호 미설정 상태에서 유일한 로그인 수단 해제 시도
```

---

### 4-4. 세션 관리

```python
# GET /api/v1/account/sessions
# refresh_tokens 테이블에서 현재 사용자의 활성(revoked=FALSE) 토큰 목록

Response:
[
  {
    "id": "uuid",
    "ip_address": "192.168.1.1",
    "user_agent": "Mozilla/5.0...",
    "created_at": "2026-04-13T10:00:00Z",
    "is_current": true
  }
]

# DELETE /api/v1/account/sessions/{session_id}
→ 해당 RT의 family 폐기
→ 현재 세션은 삭제 불가 (별도 에러)
```

---

### 4-5. 계정 관리 UI 구조

```
/account
  ├── /profile     프로필 편집 (이름, 아바타 URL)
  ├── /security    보안 설정 (비밀번호 변경, GitLab 연결)
  └── /sessions    활성 세션 목록

레이아웃:
  - 좌측 사이드바 네비게이션 (프로필, 보안, 세션)
  - 우측 컨텐츠 영역
  - 모바일: 탭 네비게이션으로 전환
```

---

## 5. 산출물

1. Account API 엔드포인트 (profile, change-password, oauth-accounts, sessions)
2. `/account/profile` UI 컴포넌트
3. `/account/security` UI 컴포넌트 (비밀번호 변경 + GitLab 연결)
4. `/account/sessions` UI 컴포넌트
5. Account 레이아웃 컴포넌트 (사이드바 네비게이션)
6. 단위 테스트 (API)

---

## 6. 완료 기준

- 프로필 조회/수정이 동작한다
- 비밀번호 변경 시 다른 세션이 모두 종료된다
- GitLab 계정 연결/해제가 동작한다
- 유일한 로그인 수단 해제가 방지된다
- 활성 세션 목록이 표시되고 원격 세션 종료가 가능하다
- 계정 관리 페이지가 데스크탑/모바일에서 반응형으로 동작한다
- UI 디자인 리뷰를 최소 5회 수행한다

---

## 7. Codex 작업 지침

- Account API는 인증된 사용자 본인의 데이터만 접근하도록 한다 (타인 접근 불가)
- 비밀번호 변경 시 현재 비밀번호를 반드시 확인한다
- GitLab 해제 시 비밀번호가 없으면 먼저 비밀번호 설정을 안내한다
- 세션 목록에서 현재 세션을 "현재 세션" 뱃지로 표시한다
