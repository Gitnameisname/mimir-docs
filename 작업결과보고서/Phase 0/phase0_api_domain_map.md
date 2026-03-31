# Phase 0 — API 도메인 맵

## 문서 목적

이 문서는 Mimir 플랫폼의 기능을 화면 단위가 아닌 도메인 단위로 분해하여 API 영역 초안을 정의한다. 각 도메인의 책임, 소비자, 주요 리소스, 비동기 연계 가능 지점을 정리한다. 이 맵은 Phase 2 API 설계의 기준점이 된다.

---

## 시스템 참여자와 API 관계

| 참여자 | API를 사용하는 이유 | 접근 가능 도메인 | 권한 수준 |
|--------|-------------------|----------------|----------|
| **User UI** | 문서 탐색·조회·작성·편집 화면을 구성하기 위해 API를 호출 | Document, Version, Node, Metadata, Search, Attachment | 인증된 일반 사용자 권한 |
| **Admin UI** | 문서 유형 관리, 권한 설정, 운영 정책 적용, 감사 조회를 위해 API를 호출 | DocumentType, Admin/Governance, Audit, Auth/Permission | 관리자 권한 |
| **External Application** | 다른 업무 시스템에서 Mimir의 지식 자산을 조회·연동하기 위해 API를 호출 | Document, Version, Node, Metadata, Search | 서비스 계정 또는 OAuth 기반 제한 권한 |
| **Internal Enterprise System** | 사내 시스템과의 자동화 동기화, 문서 상태 이벤트 수신 | Document, Status/Lifecycle, Integration/Event | 서비스 계정 기반, 이벤트 구독 가능 |
| **AI Agent / Automation Consumer** | 지식 자산을 구조 단위로 수집·처리하거나 검색 기반 조회를 수행 | Document, Node, Metadata, Search, Version | 읽기 전용 또는 제한적 쓰기 권한 |

---

## API 도메인 맵

### 1. Document API

| 항목 | 내용 |
|------|------|
| **책임** | 문서 엔티티의 생성·조회·수정·삭제 및 기본 라이프사이클 관리 |
| **대표 리소스** | `Document` |
| **주요 동작** | 생성(Create), 조회(Read), 수정(Update), 삭제(Delete), 목록 조회, 상태 전이 트리거 |
| **주요 소비자** | User UI, Admin UI, External Application, AI Agent |
| **비동기/이벤트 포인트** | 문서 생성·수정 시 검색 인덱싱 트리거, 상태 변경 이벤트 발행 |

---

### 2. Version API

| 항목 | 내용 |
|------|------|
| **책임** | 문서 버전 이력 관리, 특정 버전 조회, 버전 간 비교 |
| **대표 리소스** | `Version` |
| **주요 동작** | 버전 목록 조회, 특정 버전 조회, 버전 생성(문서 수정 시 자동), 버전 diff 조회 |
| **주요 소비자** | User UI, Admin UI, AI Agent, External Application |
| **비동기/이벤트 포인트** | 버전 생성 후 검색 인덱스 갱신, 감사 로그 기록 |

---

### 3. Node / Structure API

| 항목 | 내용 |
|------|------|
| **책임** | 문서 내부 구조(섹션, 조항, 단락 등) 노드 관리 및 계층 구조 조회 |
| **대표 리소스** | `Node` |
| **주요 동작** | 노드 트리 조회, 특정 노드 조회, 노드 생성·수정·삭제, 노드 순서 변경 |
| **주요 소비자** | User UI, AI Agent, External Application |
| **비동기/이벤트 포인트** | 노드 구조 변경 시 검색 인덱스 갱신 |

---

### 4. Metadata API

| 항목 | 내용 |
|------|------|
| **책임** | 문서·노드에 부착된 메타데이터 값 관리 |
| **대표 리소스** | `Metadata` (Document 또는 Node에 종속) |
| **주요 동작** | 메타데이터 조회, 메타데이터 값 설정·수정, 메타데이터 스키마 기반 유효성 검증 |
| **주요 소비자** | User UI, Admin UI, AI Agent, External Application |
| **비동기/이벤트 포인트** | 메타데이터 변경 시 검색 인덱스 갱신 |

---

### 5. DocumentType API

| 항목 | 내용 |
|------|------|
| **책임** | 문서 유형 정의, 메타데이터 스키마 관리, 유형별 구조·정책 설정 |
| **대표 리소스** | `DocumentType`, `MetadataSchema` |
| **주요 동작** | 유형 목록 조회, 유형 생성·수정·비활성화, 메타데이터 스키마 정의 |
| **주요 소비자** | Admin UI |
| **비동기/이벤트 포인트** | 유형 스키마 변경 시 기존 문서 메타데이터 유효성 재검증 (비동기 배치) |

---

### 6. Status / Lifecycle API

| 항목 | 내용 |
|------|------|
| **책임** | 문서 상태 전이 관리 (초안 → 검토 → 승인 → 게시 → 폐기 등) |
| **대표 리소스** | `DocumentStatus`, `StatusTransition` |
| **주요 동작** | 현재 상태 조회, 상태 전이 실행, 허용 가능한 다음 상태 조회 |
| **주요 소비자** | User UI, Admin UI, Internal Enterprise System |
| **비동기/이벤트 포인트** | 상태 전이 이벤트 발행 (외부 시스템 알림, 워크플로 트리거) |

---

### 7. Search / Query API

| 항목 | 내용 |
|------|------|
| **책임** | 전문 검색, 메타데이터 필터, 구조 탐색, 복합 쿼리 제공 |
| **대표 리소스** | `SearchResult`, `SearchQuery` |
| **주요 동작** | 키워드 검색, 메타데이터 필터링, 유형·상태 기반 필터, 노드 단위 검색 |
| **주요 소비자** | User UI, AI Agent, External Application, Automation Consumer |
| **비동기/이벤트 포인트** | 검색 인덱스는 문서·버전·노드 변경 시 비동기로 갱신 |

---

### 8. Attachment / Resource API

| 항목 | 내용 |
|------|------|
| **책임** | 첨부 파일 및 외부 리소스 참조 관리 |
| **대표 리소스** | `Attachment`, `ExternalResourceRef` |
| **주요 동작** | 첨부 업로드·조회·삭제, 외부 리소스 참조 등록·조회 |
| **주요 소비자** | User UI, Admin UI |
| **비동기/이벤트 포인트** | 대용량 파일 업로드는 비동기 처리 (업로드 상태 폴링 또는 콜백) |

---

### 9. Auth / Permission API

| 항목 | 내용 |
|------|------|
| **책임** | 인증 처리, 사용자·역할 조회, 접근 권한 확인 |
| **대표 리소스** | `User`, `Role`, `Permission` |
| **주요 동작** | 토큰 발급·검증, 현재 사용자 조회, 권한 조회, 역할 할당 |
| **주요 소비자** | 모든 클라이언트 (모든 API 요청의 선행 조건) |
| **비동기/이벤트 포인트** | 권한 변경 이벤트 (캐시 무효화, 감사 기록) |

---

### 10. Admin / Governance API

| 항목 | 내용 |
|------|------|
| **책임** | 운영 정책 관리, 사용자·역할 관리, 시스템 설정, 데이터 운영 |
| **대표 리소스** | `Policy`, `SystemConfig`, `UserManagement` |
| **주요 동작** | 정책 조회·수정, 사용자 관리, 시스템 설정 변경 |
| **주요 소비자** | Admin UI 전용 |
| **비동기/이벤트 포인트** | 정책 변경 이벤트 발행 (관련 캐시 무효화) |

---

### 11. Audit / History API

| 항목 | 내용 |
|------|------|
| **책임** | 플랫폼 내 모든 주요 행위의 감사 로그 조회 |
| **대표 리소스** | `AuditLog` |
| **주요 동작** | 감사 로그 조회, 기간·대상·행위자 기반 필터링 |
| **주요 소비자** | Admin UI, 외부 컴플라이언스 시스템 |
| **비동기/이벤트 포인트** | 감사 로그는 모든 주요 API 동작에서 비동기로 적재 |

---

### 12. Integration / Event API

| 항목 | 내용 |
|------|------|
| **책임** | 외부 시스템과의 이벤트 연계, Webhook 관리, 동기화 인터페이스 |
| **대표 리소스** | `Webhook`, `EventSubscription`, `SyncJob` |
| **주요 동작** | Webhook 등록·수정·삭제, 이벤트 구독, 동기화 작업 상태 조회 |
| **주요 소비자** | External Application, Internal Enterprise System, Admin UI |
| **비동기/이벤트 포인트** | 이벤트 발행은 전적으로 비동기. 문서 생성·수정·상태 전이·삭제가 이벤트 발생 원천 |

---

## 도메인 간 의존 관계 요약

```
Auth/Permission
      ↓ (모든 API에 선행)
Document ──→ Version
    ↓             ↓
  Node        Audit/History
    ↓
Metadata ──→ Search/Query
    ↑
DocumentType

Status/Lifecycle ──→ Integration/Event
                           ↑
                    Admin/Governance
```

- 모든 API 요청은 Auth/Permission을 거친다.
- Document는 Version, Node, Metadata를 포함하며, 이 변경들은 Search 인덱스에 반영된다.
- 상태 전이는 Integration/Event를 통해 외부 시스템에 전파된다.
- DocumentType은 Metadata 스키마를 정의하며, Admin API를 통해 관리된다.
