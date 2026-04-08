

# 📘 Task 5-6 작업 지시서
## 변경 사유 기록 구조 (ChangeLog) 설계

---

## 1. 🎯 작업 목적

문서의 **내용 변경(편집)**에 대해 “왜 변경되었는가”를 구조적으로 기록하는 **ChangeLog 시스템**을 설계한다.

이 작업의 목적은 다음과 같다.

- 단순 버전 증가가 아닌 **변경 의도와 맥락**을 보존
- 승인/반려 사유(ReviewAction)와 **편집 사유(Change)**를 명확히 분리
- 감사(Audit), 회귀 분석, 품질 관리, 규정 준수의 근거 확보

---

## 2. 📌 설계 범위

본 작업에서는 다음을 정의한다.

- ChangeLog 개념 정의
- 데이터 모델 구조
- 변경 유형(ChangeType) 정의
- reason 저장 정책
- Version/Node 변경과의 연결 방식
- 서비스 레이어 처리 흐름
- 조회/분석 활용 포인트

---

## 3. 🧩 ChangeLog 개념 정의

`ChangeLog`는 **문서 내용이 실제로 변경된 행위**에 대한 기록이다.

포함 대상:

```text
- 문서 본문 수정
- 노드(섹션/조항) 추가/삭제/이동
- 메타데이터 변경 (title, tags, category 등)
- 첨부/이미지 변경
```

제외 대상:

```text
- 승인/반려/검토 (→ ReviewAction)
- 상태 전이 (→ WorkflowHistory)
```

---

## 4. 🏗 데이터 모델 설계

권장 모델 예시는 다음과 같다.

```text
ChangeLog
- id
- document_id
- version_id
- change_type
- actor_id
- actor_role
- reason
- diff_summary (optional)
- metadata (optional)
- created_at
```

### 필드 설명

- `change_type`: 어떤 종류의 변경인지
- `reason`: 변경 사유 (핵심 필드)
- `diff_summary`: 변경 요약 (문자열 또는 JSON)
- `metadata`: 확장 정보 (예: affected_nodes, field_changes 등)

---

## 5. 🧠 ChangeType 정의

권장 Enum:

```python
from enum import Enum

class ChangeType(str, Enum):
    CONTENT_UPDATE = "content_update"
    NODE_ADD = "node_add"
    NODE_REMOVE = "node_remove"
    NODE_MOVE = "node_move"
    METADATA_UPDATE = "metadata_update"
    ATTACHMENT_UPDATE = "attachment_update"
    BULK_UPDATE = "bulk_update"
```

설계 원칙:

- 최소 단위로 세분화하되, 과도한 분할은 피한다.
- 이후 분석/통계를 고려하여 의미 단위로 정의한다.

---

## 6. 📝 reason 저장 정책

### 6.1 기본 원칙

- 모든 변경에는 가능한 한 `reason`을 남긴다.
- 자동 변경의 경우 system reason을 사용한다.

### 6.2 권장 정책

| 변경 유형 | reason 필수 여부 |
|---|---|
| CONTENT_UPDATE | required (권장) |
| NODE_ADD / REMOVE | recommended |
| METADATA_UPDATE | optional |
| BULK_UPDATE | required |

예시:

```text
- 오탈자 수정
- 정책 기준 변경 반영
- 고객 요구사항 반영
- 구조 재정리
```

---

## 7. 🔁 Version과의 관계

핵심 원칙:

> ChangeLog는 Version 내부의 변경을 설명하는 기록이다.

정책:

- 하나의 Version 생성 과정에서 여러 ChangeLog가 발생할 수 있음
- 또는 “저장 시점 1회 기록” 전략도 선택 가능

권장 2가지 전략:

### 전략 A (Granular)
- 변경 발생 시마다 ChangeLog 기록
- 장점: 상세 추적 가능
- 단점: 로그 많아짐

### 전략 B (Snapshot)
- 저장 시점에 하나의 ChangeLog 기록
- 장점: 단순
- 단점: 세부 변경 추적 어려움

👉 Phase 5 MVP에서는 **Snapshot 방식 권장**

---

## 8. 🧠 Node 기반 문서와의 연계

Mimir 구조(Node 기반)를 고려하여 다음을 지원한다.

```json
{
  "metadata": {
    "affected_nodes": ["node_1", "node_3"],
    "change_scope": "section",
    "field_changes": {
      "title": ["old", "new"]
    }
  }
}
```

---

## 9. 🛡 서비스 레이어 처리 흐름

```text
1. 사용자 수정 요청 수신
2. 변경 내용 분석 (diff 또는 field change)
3. Version 저장
4. ChangeLog 생성
5. (선택) Workflow 영향 여부 판단
6. Audit Log 기록
7. 응답 반환
```

권장 사항:

- Version 저장과 ChangeLog 생성은 동일 트랜잭션
- diff 계산은 선택적 (성능 고려)

---

## 10. 🔐 actor_role 스냅샷

ReviewAction과 동일하게,
변경 시점의 역할을 기록한다.

이유:

- 나중에 권한 변경과 무관하게 당시 행위 정당성 검증 가능

---

## 11. ⚠️ 자동 변경(System Change)

시스템이 수행하는 변경도 기록해야 한다.

예:

```text
- 마이그레이션
- 자동 포맷팅
- 정책 엔진 자동 수정
```

정책:

```text
actor_id = system
actor_role = SYSTEM
reason = "system_update"
```

---

## 12. 📊 조회 및 활용

### 12.1 문서 변경 이력

- Version별 변경 이유 표시
- 최근 수정 내역 요약

### 12.2 감사 화면

- 누가 어떤 이유로 수정했는지
- 특정 사용자 변경 패턴 분석

### 12.3 품질 분석

- 잦은 수정 구간 식별
- 반려 후 수정 반복 패턴 분석

---

## 13. 🔄 다른 로그와의 관계

| 로그 타입 | 목적 |
|---|---|
| ChangeLog | 내용 변경 기록 |
| ReviewAction | 검토/승인/반려 |
| WorkflowHistory | 상태 전이 |
| AuditLog | 시스템 이벤트 |

👉 네 가지는 서로 역할이 다르므로 분리 유지

---

## 14. 🧾 예시 데이터

```json
{
  "id": "chg_001",
  "document_id": "doc_123",
  "version_id": "ver_6",
  "change_type": "content_update",
  "actor_id": "user_002",
  "actor_role": "AUTHOR",
  "reason": "정책 문구 최신화",
  "diff_summary": "Section 2 문구 수정",
  "created_at": "2026-04-06T12:00:00Z",
  "metadata": {
    "affected_nodes": ["node_2"]
  }
}
```

---

## 15. 📦 산출물

- ChangeLog 개념 정의
- ChangeType Enum
- 데이터 모델 설계
- reason 정책
- Version/Node 연계 구조
- 서비스 처리 흐름

---

## 16. 🚀 다음 작업 연결

👉 Task 5-7: Workflow 이력 관리 시스템 설계

다음 내용을 정의한다.

- 상태 전이 로그 구조
- WorkflowHistory 모델
- 조회 API

---

## 17. ✅ 완료 기준

- 모든 변경이 reason 기반으로 기록 가능해야 한다.
- ReviewAction과 명확히 구분되어야 한다.
- Version 구조와 자연스럽게 연결되어야 한다.
- 이후 감사/분석/UI로 확장 가능해야 한다.
