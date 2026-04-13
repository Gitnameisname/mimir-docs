# Task 7-6 산출물: 감사 로그 조회 UI 설계서

---

## 1. 감사 로그 목록 화면 (`/admin/audit-logs`)

### 1-1. 화면 레이아웃

```
Admin > Audit Log

┌─────────────────────────────────────────────────────────────────────┐
│  [기간 ▾]  [행위자 검색]  [이벤트 타입 ▾]  [결과 ▾]  [중요도 ▾]     │
│  활성 필터: [기간: 최근 7일 ×]  [중요도: CRITICAL ×]  [전체 초기화]  │
├──────────────┬──────┬──────────────────────┬──────────┬─────┬───────┤
│ 발생 시각     │ 중요도│ 이벤트 타입           │ 행위자   │ 결과 │ IP  │
├──────────────┼──────┼──────────────────────┼──────────┼─────┼───────┤
│ 04-07 14:32  │🔴CRIT│ PERMISSION_CHANGED   │ admin@.. │ 🟢  │ 1.1.1│
│ 04-07 13:21  │🟠HIGH│ USER_DEACTIVATED     │ admin@.. │ 🟢  │ 1.1.1│
│ 04-07 12:05  │ -    │ USER_LOGIN           │ hong@..  │ 🟢  │ 2.2.2│
└──────────────┴──────┴──────────────────────┴──────────┴─────┴───────┘
```

**CRITICAL 이벤트 행**: 옅은 빨간 배경으로 강조

### 1-2. 목록 테이블 컬럼

| 컬럼 | 데이터 | 정렬 |
|------|--------|------|
| 발생 시각 | `occurred_at` | ✅ |
| 중요도 | `severity` | StatusBadge (CRITICAL/HIGH/NORMAL) |
| 이벤트 타입 | `event_type` | - |
| 행위자 | `actor.display_name` | - |
| 대상 리소스 | `resource_type + resource_id` | - |
| 결과 | `result` | SUCCESS(🟢) / FAILURE(🔴) |
| IP 주소 | `ip_address` | - |

### 1-3. 필터 정의

| 필터 | 옵션 | 비고 |
|------|------|------|
| 기간 | 오늘 / 최근 7일 / 최근 30일 / 직접 입력 | 기본: 최근 7일 |
| 행위자 | 이름 또는 이메일 검색 | |
| 이벤트 타입 | 다중 선택 체크박스 | 아래 목록 참조 |
| 결과 | ALL / SUCCESS / FAILURE | |
| 중요도 | ALL / CRITICAL / HIGH / NORMAL | |
| 리소스 유형 | 동적 조회 | |

---

## 2. 감사 이벤트 상세 패널 설계서

목록에서 이벤트 클릭 시 RightSidePanel으로 표시

```
┌─────────────────────────────────────────────────────────┐
│  감사 이벤트 상세                                   [×] │
├─────────────────────────────────────────────────────────┤
│  이벤트 ID: evt-abc123                                  │
│  발생 시각: 2026-04-07 14:32:18                         │
│  이벤트 타입: PERMISSION_CHANGED                        │
│  중요도: 🔴 CRITICAL                                    │
├─────────────────────────────────────────────────────────┤
│  행위자 정보                                            │
│  이름: admin@system.com                                 │
│  IP: 192.168.1.1                                       │
│  세션: sess-xyz789                                      │
├─────────────────────────────────────────────────────────┤
│  대상 리소스                                            │
│  유형: Permission                                       │
│  ID: perm-def456                                        │
│  이름: Document Write Policy                            │
│  [정책 상세 보기 →]                                     │
├─────────────────────────────────────────────────────────┤
│  변경 내역                                              │
│  필드: actions                                          │
│  Before: ["CREATE", "READ", "UPDATE"]                  │
│  After:  ["CREATE", "READ", "UPDATE", "DELETE"]        │
│                                                         │
│  [JSON diff 보기 ▾]                                    │
│  ┌──────────────────────────────────────────────────┐  │
│  │  - "actions": ["CREATE","READ","UPDATE"]          │  │
│  │  + "actions": ["CREATE","READ","UPDATE","DELETE"] │  │
│  └──────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│  요청 정보                                              │
│  API: PATCH /api/v1/admin/permissions/perm-def456      │
│  [연관 이벤트 보기]                                     │
└─────────────────────────────────────────────────────────┘
```

---

## 3. 이벤트 중요도 분류 기준 문서

| 중요도 | 배지 색상 | 해당 이벤트 예시 | 행 표시 |
|--------|-----------|----------------|---------|
| CRITICAL | 빨간색 | PERMISSION_CHANGED, ADMIN_ACCOUNT_MODIFIED, API_KEY_REVOKED, DOCUMENT_TYPE_DEACTIVATED | 빨간 배경 강조 |
| HIGH | 주황색 | USER_DEACTIVATED, ROLE_CHANGED, DOCUMENT_FORCE_DELETED, ORG_DELETED | 주황 배경 (옅게) |
| NORMAL | 없음 | USER_LOGIN, USER_LOGOUT, DOCUMENT_CREATED, DOCUMENT_VIEWED, VERSION_CREATED | 기본 |

### 우선 표시 규칙

- CRITICAL/HIGH 이벤트는 기간 내 항목 중 상단 정렬 (기본 정렬: 시각 desc이므로 필터로 유도)
- 대시보드의 "최근 감사 이벤트" 요약에서 CRITICAL/HIGH 우선 표시

---

## 4. 변경 전후 비교 표시 가이드

### 4-1. 단순 필드 변경

```
필드명: status
Before: ACTIVE
After:  INACTIVE
```

### 4-2. 배열/구조 변경 (JSON diff)

```diff
- "actions": ["CREATE", "READ", "UPDATE"]
+ "actions": ["CREATE", "READ", "UPDATE", "DELETE"]
```

### 4-3. 보안 규칙

| 필드명 패턴 | 처리 방식 |
|-----------|-----------|
| `password`, `secret`, `token` | `[MASKED]` 로 대체 |
| `api_key`, `key_value` | `[MASKED]` |
| IP 주소 | 그대로 표시 (운영 목적) |
| 이메일 | 그대로 표시 |

### 4-4. 변경 내역 없는 이벤트

- 로그인, 조회 등 → 변경 내역 섹션 숨김
- "이 이벤트는 변경 내역이 없습니다" 안내 (섹션 자체 숨기는 게 더 깔끔)

---

## 5. 이벤트 타입 목록

| 이벤트 타입 | 설명 | 중요도 |
|------------|------|--------|
| USER_LOGIN | 사용자 로그인 | NORMAL |
| USER_LOGOUT | 사용자 로그아웃 | NORMAL |
| USER_STATUS_CHANGED | 사용자 상태 변경 | HIGH |
| USER_ROLE_CHANGED | 사용자 역할 변경 | HIGH |
| PERMISSION_CHANGED | 권한 정책 변경 | CRITICAL |
| API_KEY_ISSUED | API 키 발급 | HIGH |
| API_KEY_REVOKED | API 키 폐기 | CRITICAL |
| DOCUMENT_TYPE_DEACTIVATED | DocType 비활성화 | CRITICAL |
| DOCUMENT_FORCE_DELETED | 문서 강제 삭제 | HIGH |
| JOB_FORCE_STOPPED | 작업 강제 종료 | HIGH |
| INDEXING_RETRY | 인덱싱 재시도 | NORMAL |
| AUDIT_LOG_EXPORTED | 감사 로그 내보내기 | HIGH |
