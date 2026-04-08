

# 📘 Task 5-9 작업 지시서
## 예외 및 운영 정책 정의

---

## 1. 🎯 작업 목적

Phase 5에서 설계한 워크플로를 실제 운영 환경에서 안정적으로 적용하기 위해,
예외 상황과 운영 정책을 명확히 정의한다.

이 작업의 목적은 다음과 같다.

- 동시성, 충돌, 권한 우회 등 **현실적인 운영 이슈 대응**
- 승인/게시 이후 변경에 대한 **정책 일관성 확보**
- 시스템과 사용자 간 책임 경계 명확화
- 감사/컴플라이언스 요구 충족

---

## 2. 📌 설계 범위

본 작업에서는 다음을 정의한다.

- 승인 중 수정 제한 정책
- 게시 이후 수정 정책
- 동시성/충돌 처리 정책
- Admin override 정책
- 롤백/복원 정책
- 시스템 장애/부분 실패 대응
- 감사(Audit) 필수 기록 규칙

---

## 3. 🧩 승인 중 수정 제한 정책

### 정책 원칙

```text
IN_REVIEW 상태의 Version은 기본적으로 직접 수정 금지
```

### 허용 옵션 (선택 정책)

옵션 A (권장, 엄격):
- 수정하려면 반드시 `REJECTED → DRAFT`로 되돌린 후 수정

옵션 B (완화):
- 제한적 수정 허용 (오탈자 등)
- 변경 시 자동으로 Review 상태 유지 또는 재검토 트리거

👉 Phase 5 MVP 권장: **옵션 A (엄격 정책)**

---

## 4. 🧩 게시(PUBLISHED) 이후 수정 정책

### 정책 원칙

```text
PUBLISHED 상태의 Version은 immutable (수정 불가)
```

### 수정 방법

```text
새 Version 생성 → DRAFT → (다시 워크플로)
```

### 금지 사항

- 기존 Published Version 직접 수정
- 상태를 되돌려 수정 (PUBLISHED → DRAFT 직접 전이 금지)

---

## 5. 🧩 동시성 및 충돌 처리 정책

### 5.1 상태 충돌 (State Conflict)

시나리오:

- 사용자 A: 승인 클릭
- 사용자 B: 이미 반려 처리

정책:

```text
expected_current_status 기반 검증 실패 시 → 409 Conflict 반환
```

UI 처리:

- 상태 재조회
- 사용자에게 충돌 알림

---

### 5.2 중복 승인/반려 방지

정책:

```text
이미 APPROVED 상태인 Version에 대해 approve 요청 금지
이미 REJECTED 상태인 Version에 대해 reject 요청 금지
```

---

### 5.3 낙관적 락(Optimistic Lock)

권장 필드:

```text
- expected_current_status
- version_number 또는 updated_at
```

---

## 6. 🧩 Admin Override 정책

### 정책 목적

운영 중 예외 상황 대응

예:

- 잘못 승인된 문서
- 긴급 게시 필요
- 권한 체계 오류

### 정책 정의

```text
ADMIN은 모든 상태 전이 수행 가능
```

### 필수 조건

- reason 필수 입력
- Audit Log 필수 기록
- override 여부 flag 기록

### 권장 UI

- 경고 다이얼로그
- "이 작업은 감사 대상입니다" 안내

---

## 7. 🧩 롤백 및 복원 정책

### 7.1 상태 롤백

기본 정책:

```text
직접 상태 롤백 금지
```

대신:

```text
새 Version 생성 → 과거 내용 복사 → DRAFT 시작
```

---

### 7.2 Version 복원

허용 정책:

- 과거 Version을 기반으로 새로운 Version 생성
- 기존 Version은 immutable 유지

---

## 8. 🧩 부분 실패 및 트랜잭션 정책

### 8.1 트랜잭션 단위

다음은 반드시 하나의 트랜잭션으로 묶는다.

```text
- 상태 변경
- WorkflowHistory 생성
- ReviewAction 생성 (필요 시)
- Audit Log 생성
```

---

### 8.2 실패 처리

정책:

```text
하나라도 실패하면 전체 롤백
```

예:

- 상태는 바뀌었는데 로그가 없는 상황 방지

---

## 9. 🧩 Audit 필수 기록 정책

다음 이벤트는 반드시 Audit Log에 기록한다.

```text
- 상태 전이 (모든 workflow action)
- Admin override
- publish / archive
- reject / approve
```

필수 필드:

```text
- event_type
- actor_id
- document_id
- version_id
- action
- result (success/failure)
- request_id
```

---

## 10. 🧩 보안 및 권한 예외

### 정책 원칙

```text
UI는 보조 수단, 최종 검증은 서버
```

### 금지 사항

- 클라이언트만으로 권한 제어
- 상태 검증 없이 처리

---

## 11. 🧩 자동화 및 시스템 액션 정책

### 예시

- 자동 게시 (스케줄링)
- 정책 엔진 자동 승인

### 정책

```text
actor_id = system
actor_role = SYSTEM
metadata에 자동 처리 여부 기록
```

---

## 12. 🧩 SLA 및 운영 정책 (선택)

향후 확장:

```text
- 검토 SLA (예: 24시간 이내)
- 승인 SLA
- 자동 알림 (email / webhook)
```

---

## 13. 🧩 예외 상황 요약

| 상황 | 대응 정책 |
|---|---|
| 상태 충돌 | 409 반환 + 재조회 |
| 권한 없음 | 403 반환 |
| 승인 중 수정 | 금지 또는 제한 |
| 게시 후 수정 | 새 Version 생성 |
| 로그 기록 실패 | 전체 롤백 |
| Admin override | reason + audit 필수 |

---

## 14. 📦 산출물

- 운영 정책 정의 문서
- 예외 처리 규칙
- 동시성 처리 정책
- Admin override 정책
- Audit 기록 기준

---

## 15. 🎉 Phase 5 완료 의미

이 Task까지 완료되면 다음이 확보된다.

```text
✔ 상태 모델
✔ 상태 전이
✔ 권한 제어
✔ API
✔ ReviewAction
✔ ChangeLog
✔ WorkflowHistory
✔ UI 연결
✔ 운영 정책
```

👉 즉, 문서 플랫폼의 “운영 프로세스 시스템”이 완성된다.

---

## 16. ✅ 완료 기준

- 모든 주요 예외 상황에 대한 대응 정책이 정의되어야 한다.
- 서버 기준으로 강제 가능한 정책이어야 한다.
- Audit/보안/운영 관점에서 누락이 없어야 한다.
- 실제 구현 시 바로 적용 가능한 수준이어야 한다.
