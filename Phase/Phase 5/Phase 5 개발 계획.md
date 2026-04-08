📘 Phase 5 전체 구조

핵심 설계 축

Phase 5는 아래 5개의 축으로 나뉜다:
	1.	Workflow 상태 모델 정의
	2.	상태 전이(State Machine) 설계
	3.	역할 기반 전이 제한 (RBAC 연동)
	4.	검토/승인/반려 기능 구현
	5.	이력/감사 추적 시스템 구축

⸻

🧩 Task 분해

⸻

🧩 Task 5-1. Workflow 상태 모델 정의

🎯 목적

문서의 생명주기를 표현하는 상태 모델 정의

📌 작업 내용
	•	문서 상태 정의

DRAFT
IN_REVIEW
APPROVED
PUBLISHED
REJECTED
ARCHIVED (선택)

	•	상태 Enum 정의
	•	상태별 의미 명확화

📦 산출물
	•	DocumentStatus Enum
	•	상태 정의 문서

⚠️ 설계 포인트
	•	“REJECTED”는 종료 상태가 아니라 다시 Draft로 돌아갈 수 있음
	•	Published는 immutable 정책 고려 (Phase 4와 연결)

⸻

🧩 Task 5-2. 상태 전이(State Machine) 설계

🎯 목적

문서 상태 변경 규칙을 명확히 정의

📌 작업 내용
	•	허용 전이 정의

DRAFT → IN_REVIEW
IN_REVIEW → APPROVED
IN_REVIEW → REJECTED
REJECTED → DRAFT
APPROVED → PUBLISHED
PUBLISHED → (새 버전 생성 → DRAFT)

	•	상태 전이 테이블 정의
	•	불법 전이 차단 로직 설계

📦 산출물
	•	WorkflowTransitionMap
	•	상태 전이 정책 문서

⚠️ 설계 포인트
	•	상태 전이는 반드시 서버에서만 허용 (UI 신뢰 금지)
	•	이벤트 기반 처리 고려 가능

⸻

🧩 Task 5-3. 역할 기반 상태 전이 제한 (RBAC 연동)

🎯 목적

누가 어떤 상태 변경을 할 수 있는지 정의

📌 작업 내용
	•	역할 정의

AUTHOR
REVIEWER
APPROVER
ADMIN

	•	역할별 권한 매핑

역할	가능 액션
AUTHOR	Draft 작성, Review 요청
REVIEWER	검토, 반려
APPROVER	승인
ADMIN	전체

	•	상태 전이 + 역할 매핑

📦 산출물
	•	WorkflowPermissionMap
	•	RBAC 연계 정의서

⚠️ 설계 포인트
	•	Phase 2 ACL과 반드시 연결
	•	향후 조직별 커스터마이징 가능 구조

⸻

🧩 Task 5-4. Workflow Action API 설계

🎯 목적

워크플로를 API로 제어 가능하게 함

📌 작업 내용

다음 API 정의:

POST /documents/{id}/workflow/submit-review
POST /documents/{id}/workflow/approve
POST /documents/{id}/workflow/reject
POST /documents/{id}/workflow/publish
POST /documents/{id}/workflow/revert

	•	요청 payload

{
  "comment": "검토 의견",
  "reason": "반려 사유"
}

	•	서버에서:
	•	현재 상태 확인
	•	사용자 권한 확인
	•	상태 전이 수행

📦 산출물
	•	API spec
	•	request/response schema

⚠️ 설계 포인트
	•	idempotency 고려
	•	optimistic lock (version 기반) 고려

⸻

🧩 Task 5-5. 검토 / 승인 / 반려 기능 구현

🎯 목적

실제 워크플로 동작 구현

📌 작업 내용
	•	Review 요청 기능
	•	승인 처리
	•	반려 처리
	•	코멘트 저장

📦 데이터 구조

ReviewAction
- id
- document_id
- version_id
- action_type (submit/review/approve/reject)
- actor_id
- comment
- created_at

📦 산출물
	•	ReviewAction 모델
	•	서비스 로직

⚠️ 설계 포인트
	•	version 기준으로 기록해야 함
	•	동일 문서라도 버전별 review 독립

⸻

🧩 Task 5-6. 변경 사유 기록 구조 설계

🎯 목적

모든 변경에 대한 이유를 남김

📌 작업 내용
	•	변경 시 reason 필수화 (옵션/필수 정책 결정)

ChangeLog
- document_id
- version_id
- change_type
- reason
- actor_id

📦 산출물
	•	ChangeLog 모델
	•	reason 정책 정의

⚠️ 설계 포인트
	•	승인/반려뿐 아니라 수정도 포함할지 결정 필요

⸻

🧩 Task 5-7. Workflow 이력 관리 시스템 구축

🎯 목적

문서의 모든 상태 변화 추적

📌 작업 내용

WorkflowHistory
- document_id
- version_id
- from_status
- to_status
- actor_id
- action
- timestamp

	•	상태 변경 시 자동 기록
	•	조회 API 설계

GET /documents/{id}/workflow/history

📦 산출물
	•	WorkflowHistory 모델
	•	조회 API

⚠️ 설계 포인트
	•	Audit Log와 중복 여부 검토 (Phase 2 연계)
	•	immutable 로그 권장

⸻

🧩 Task 5-8. UI 연동 포인트 정의

🎯 목적

Phase 6 UI 연결 준비

📌 작업 내용
	•	UI 필요 기능 정의

- 상태 표시 배지
- 승인/반려 버튼
- 코멘트 입력
- 이력 타임라인

	•	API ↔ UI 매핑 정의

📦 산출물
	•	UI 연동 스펙

⸻

🧩 Task 5-9. 예외 및 정책 정의

🎯 목적

운영 중 발생할 문제 대응

📌 작업 내용
	•	승인 중 변경 방지 정책
	•	동시 승인 충돌 처리
	•	승인 이후 수정 정책

📦 산출물
	•	정책 문서

⸻

🚀 최종 실행 순서 (중요)

1. 상태 모델 정의 (5-1)
2. 상태 전이 설계 (5-2)
3. RBAC 연동 (5-3)
4. API 설계 (5-4)
5. Review 기능 구현 (5-5)
6. ChangeLog 설계 (5-6)
7. WorkflowHistory 구축 (5-7)1
8. UI 연동 정의 (5-8)
9. 예외 정책 정리 (5-9)