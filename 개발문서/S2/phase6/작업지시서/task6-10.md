# FG6.3 작업지시서 (Task 6-10): 추출 스키마 관리 + 추출 결과 검토 큐 (스켈레톤)

**최종 수정**: 2026-04-17  
**상태**: 작업 지시서 초안

---

## 1. 작업 목적

Phase 6 통합 관리 콘솔 단계에서 추출(Structured Extraction) 기능의 스켈레톤 UI를 구축합니다. Phase 8 Structured Extraction 백엔드가 미완성 상태이므로, 다음을 먼저 구현합니다:

- **추출 타겟 스키마 관리 페이지** (`/admin/extraction-schemas`): DocumentType별 추출 스키마 정의, 필드 관리, 추출 모드 설정
- **추출 결과 검토 큐 페이지** (`/admin/extraction-review`): 미승인 추출 결과 목록, 상세 검토, 승인/반려

목 데이터(mock data)로 UI/UX를 검증하고, Phase 8 API 완료 후 엔드포인트만 교체하면 즉시 운영 가능한 구조로 설계합니다.

---

## 2. 작업 범위

### 포함 항목

1. **프론트엔드 페이지 및 컴포넌트**
   - `/admin/extraction-schemas`: 스키마 목록, 편집
   - `/admin/extraction-review`: 검토 대기 목록, 상세 검토 모달

2. **UI 컴포넌트**
   - AdminTable (TanStack Table 기반)
   - 스키마 편집 폼 (메타데이터, 필드 테이블)
   - 필드 추가 버튼 (disabled, Phase 8 후 활성)
   - 추출 모드 라디오 버튼
   - 검토 상세 모달 (문서 미리보기, 필드 값 테이블, 승인/반려)
   - 필터 UI (DocumentType, 상태)

3. **Mock API 서비스**
   - `services/mockApi/extractionMockApi.ts`

4. **목 데이터**
   - 추출 스키마 2~3개 (각 5~8개 필드)
   - 미승인 추출 결과 5~10개

5. **라우팅 및 네비게이션**
   - Task 6-9와 함께 사이드바 "평가·추출" 섹션 구성
   - 메뉴: "추출 스키마", "추출 검토"

6. **단위 테스트 (Jest)**
   - 스키마 목록 렌더링
   - 스키마 편집 폼
   - 검토 상세 페이지
   - Mock API 호출 검증

7. **접근성**
   - WCAG 2.1 AA 준수
   - 스크린리더 호환성

### 제외 항목

- 골든셋 관리 (`/admin/golden-sets`) → Task 6-9
- 평가 결과 대시보드 (`/admin/evaluations`) → Task 6-9
- 실제 Phase 8 API 연동 (Phase 8 완료 후)
- 필드 추가 기능 활성화 (Phase 8 후)
- Span 역참조 상세 (Phase 8 FG8.3 후)

---

## 3. 선행 조건

### 기술 스택
- **프론트엔드**: Next.js 16.2.2, React 19, TypeScript 5.x
- **상태 관리**: Zustand
- **테이블**: TanStack Table v8
- **UI 라이브러리**: shadcn/ui (기존 컴포넌트 재사용)
- **테스트**: Jest, React Testing Library

### 사전 준비
1. 관리자 레이아웃 (`/admin`) 기본 구조 확인
2. DocumentType 설정 (동적 로드 구조) 확인
3. Mock API 디렉토리 (`services/mockApi/`) 확인
4. Task 6-9 페이지 구조 일관성 유지

### S2 원칙 준수 체크
- **원칙 ①**: DocumentType은 설정에서 동적으로 로드 (하드코딩 금지)
- **원칙 ⑥**: Scope Profile 기반 접근 제어 (admin 권한만)
- **원칙 ⑦**: 폐쇄망 환경 지원 (mock API 로컬 동작)

---

## 4. 주요 작업 항목

### 4-1. 추출 타겟 스키마 관리 페이지

#### 4-1-1. 페이지 구조 및 라우팅
- **경로**: `/admin/extraction-schemas`
- **레이아웃**: 2열 또는 1열
- **메인 섹션**:
  - 헤더: "추출 스키마 관리" 제목
  - 스키마 목록 테이블 (좌측 또는 전체 너비)
  - 스키마 편집 폼 (우측 또는 아래)

#### 4-1-2. 스키마 목록 테이블
**컴포넌트**: `components/admin/ExtractionSchemaTable.tsx`

**테이블 칼럼**:
| 칼럼명 | 타입 | 설명 |
|--------|------|------|
| DocumentType | string | 문서 타입명 (동적 로드) |
| 필드 수 | number | 추출 필드 개수 |
| 수정일 | date | YYYY-MM-DD HH:mm 형식 |
| 상태 | badge | 활성 / 비활성 |
| 작업 | actions | "편집", "복제", "삭제" |

**기능**:
- TanStack Table 기반 (정렬, 페이지네이션)
- 행 클릭 → 우측 패널에서 스키마 편집 폼 열기
- 검색 필터 (DocumentType)
- 상태 필터 (활성/비활성)

**코드 예시**:
```typescript
// components/admin/ExtractionSchemaTable.tsx
import { useMemo, useState } from 'react';
import { createColumnHelper, useReactTable, getCoreRowModel } from '@tanstack/react-table';
import { AdminTable } from '@/components/admin/AdminTable';
import { ExtractionSchema } from '@/types/extraction';

const columnHelper = createColumnHelper<ExtractionSchema>();

export function ExtractionSchemaTable() {
  const [selectedSchema, setSelectedSchema] = useState<ExtractionSchema | null>(null);

  const columns = [
    columnHelper.accessor('documentType', {
      cell: (info) => <span className="font-semibold">{info.getValue()}</span>,
      header: 'DocumentType',
    }),
    columnHelper.accessor('fieldCount', {
      cell: (info) => info.getValue(),
      header: '필드 수',
    }),
    columnHelper.accessor('updatedAt', {
      cell: (info) => new Date(info.getValue()).toLocaleString('ko-KR'),
      header: '수정일',
    }),
    columnHelper.accessor('isActive', {
      cell: (info) => (
        <Badge className={info.getValue() ? 'bg-green-100' : 'bg-gray-100'}>
          {info.getValue() ? '활성' : '비활성'}
        </Badge>
      ),
      header: '상태',
    }),
    columnHelper.display({
      id: 'actions',
      cell: (info) => (
        <DropdownMenu>
          <DropdownMenuTrigger>작업</DropdownMenuTrigger>
          <DropdownMenuContent>
            <DropdownMenuItem onClick={() => setSelectedSchema(info.row.original)}>편집</DropdownMenuItem>
            <DropdownMenuItem>복제</DropdownMenuItem>
            <DropdownMenuItem className="text-red-600">삭제</DropdownMenuItem>
          </DropdownMenuContent>
        </DropdownMenu>
      ),
      header: '작업',
    }),
  ];

  const [data] = useState<ExtractionSchema[]>(extractionMockData);
  const table = useReactTable({ data, columns, getCoreRowModel: getCoreRowModel() });

  return (
    <div className="grid grid-cols-3 gap-4">
      <div className="col-span-2">
        <AdminTable table={table} />
      </div>
      {selectedSchema && (
        <div className="col-span-1">
          <ExtractionSchemaEditForm schema={selectedSchema} />
        </div>
      )}
    </div>
  );
}
```

#### 4-1-3. 스키마 편집 폼
**컴포넌트**: `components/admin/ExtractionSchemaEditForm.tsx`

**섹션**:

1. **메타데이터**
   - DocumentType (읽기 전용, 표시만)
   - 설명 (선택사항)
   - 활성 상태 (체크박스)

2. **필드 테이블**
   - **칼럼**: 필드명, 타입, 필수 여부, 설명
   - **필드 타입**: string, number, boolean, array, object
   - **기능**:
     - 필드명, 타입 수정 가능 (Phase 7까지 읽기 전용)
     - "필드 추가" 버튼 (disabled, Phase 8 후 활성)
     - "필드 제거" (행별, Phase 8 후 활성)

3. **추출 모드 설정**
   - 라디오 버튼: "결정론적" (Deterministic) / "확률적" (Probabilistic)
   - 설명: 각 모드별 특징 설명

4. **저장/취소 버튼**
   - "저장": Mock API 호출
   - "취소": 편집 취소

**테이블 구조** (Phase 8에서 추가):
```typescript
interface ExtractionField {
  id: string;
  name: string;
  type: 'string' | 'number' | 'boolean' | 'array' | 'object';
  required: boolean;
  description?: string;
}
```

**코드 예시**:
```typescript
// components/admin/ExtractionSchemaEditForm.tsx
import { useState } from 'react';
import { useForm, useFieldArray } from 'react-hook-form';
import { ExtractionSchema, ExtractionField } from '@/types/extraction';

export function ExtractionSchemaEditForm({ schema }: { schema: ExtractionSchema }) {
  const { register, control, handleSubmit } = useForm({
    defaultValues: schema,
  });
  const { fields } = useFieldArray({ control, name: 'fields' });

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
      {/* 메타데이터 */}
      <div className="space-y-2">
        <label className="font-semibold">DocumentType</label>
        <input
          {...register('documentType')}
          disabled
          className="w-full border rounded px-2 py-1 bg-gray-100"
        />
      </div>

      {/* 필드 테이블 */}
      <div className="space-y-2">
        <label className="font-semibold">필드</label>
        <table className="w-full border-collapse border border-gray-300">
          <thead>
            <tr className="bg-gray-100">
              <th className="border p-2">필드명</th>
              <th className="border p-2">타입</th>
              <th className="border p-2">필수</th>
              <th className="border p-2">설명</th>
            </tr>
          </thead>
          <tbody>
            {fields.map((field, index) => (
              <tr key={field.id}>
                <td className="border p-2">
                  <input
                    {...register(`fields.${index}.name`)}
                    disabled
                    className="w-full border rounded px-1"
                  />
                </td>
                <td className="border p-2">
                  <select {...register(`fields.${index}.type`)} disabled className="w-full border rounded px-1">
                    <option value="string">string</option>
                    <option value="number">number</option>
                    <option value="boolean">boolean</option>
                    <option value="array">array</option>
                    <option value="object">object</option>
                  </select>
                </td>
                <td className="border p-2">
                  <input
                    {...register(`fields.${index}.required`)}
                    type="checkbox"
                    disabled
                  />
                </td>
                <td className="border p-2">
                  <input
                    {...register(`fields.${index}.description`)}
                    disabled
                    className="w-full border rounded px-1 text-sm"
                  />
                </td>
              </tr>
            ))}
          </tbody>
        </table>
        <button
          type="button"
          disabled
          className="px-3 py-1 bg-gray-300 text-gray-600 rounded text-sm"
          title="Phase 8에서 활성화 예정"
        >
          + 필드 추가 (Phase 8)
        </button>
      </div>

      {/* 추출 모드 */}
      <div className="space-y-2">
        <label className="font-semibold">추출 모드</label>
        <div className="space-y-1">
          <label className="flex items-center gap-2">
            <input {...register('extractionMode')} type="radio" value="deterministic" />
            <span>결정론적 (Deterministic)</span>
          </label>
          <label className="flex items-center gap-2">
            <input {...register('extractionMode')} type="radio" value="probabilistic" />
            <span>확률적 (Probabilistic)</span>
          </label>
        </div>
      </div>

      {/* 버튼 */}
      <div className="flex gap-2 justify-end">
        <button type="button" onClick={() => onCancel()} className="px-4 py-2 border rounded">
          취소
        </button>
        <button type="submit" className="px-4 py-2 bg-blue-600 text-white rounded">
          저장
        </button>
      </div>
    </form>
  );
}
```

---

### 4-2. 추출 결과 검토 큐 페이지

#### 4-2-1. 페이지 구조 및 라우팅
- **경로**: `/admin/extraction-review`
- **레이아웃**: 풀 너비
- **메인 섹션**:
  1. 헤더: "추출 결과 검토" 제목
  2. 필터 UI (DocumentType, 상태)
  3. 미승인 목록 테이블
  4. 검토 상세 모달

#### 4-2-2. 필터 UI
**컴포넌트**: `components/admin/ExtractionReviewFilters.tsx`

**필터 항목**:
1. **DocumentType**: 드롭다운 (동적 로드)
2. **상태**: 드롭다운 (대기, 승인, 반려)
3. **기간**: 날짜 범위 선택 (최근 7일 / 30일 / 커스텀)

**기능**:
- 필터 변경 시 목록 업데이트
- "초기화" 버튼

**코드 예시**:
```typescript
// components/admin/ExtractionReviewFilters.tsx
import { useState } from 'react';

export function ExtractionReviewFilters({ onFilterChange }: Props) {
  const [filters, setFilters] = useState({
    documentType: 'all',
    status: 'pending',
    dateRange: 'last7days',
  });

  const handleFilterChange = (key: string, value: any) => {
    const newFilters = { ...filters, [key]: value };
    setFilters(newFilters);
    onFilterChange(newFilters);
  };

  return (
    <div className="flex gap-4 items-center p-4 bg-gray-50 rounded">
      <SelectInput
        label="DocumentType"
        options={documentTypes}
        onChange={(v) => handleFilterChange('documentType', v)}
      />
      <SelectInput
        label="상태"
        options={[
          { value: 'pending', label: '대기' },
          { value: 'approved', label: '승인' },
          { value: 'rejected', label: '반려' },
        ]}
        onChange={(v) => handleFilterChange('status', v)}
      />
      <button onClick={() => setFilters({ documentType: 'all', status: 'pending', dateRange: 'last7days' })}>
        초기화
      </button>
    </div>
  );
}
```

#### 4-2-3. 미승인 목록 테이블
**컴포넌트**: `components/admin/ExtractionReviewTable.tsx`

**테이블 칼럼**:
| 칼럼명 | 타입 | 설명 |
|--------|------|------|
| 문서명 | string | 원본 문서 이름 (클릭 시 상세) |
| DocumentType | string | 문서 타입 |
| 추출 시각 | datetime | YYYY-MM-DD HH:mm 형식 |
| 상태 | badge | 대기 / 승인 / 반려 |
| 작업 | actions | "검토", "승인", "반려" |

**기능**:
- TanStack Table 기반 (정렬, 페이지네이션)
- 페이지 크기: 10개
- 행 클릭 → 검토 상세 모달 열기
- 행 선택 (체크박스) → "모두 승인" 버튼 활성화 (선택 항목만)

**코드 예시**:
```typescript
// components/admin/ExtractionReviewTable.tsx
import { useMemo, useState } from 'react';
import { createColumnHelper, useReactTable, getCoreRowModel } from '@tanstack/react-table';
import { AdminTable } from '@/components/admin/AdminTable';
import { ExtractionResult } from '@/types/extraction';

const columnHelper = createColumnHelper<ExtractionResult>();

export function ExtractionReviewTable({ results }: { results: ExtractionResult[] }) {
  const [selectedRows, setSelectedRows] = useState<Set<string>>(new Set());
  const [detailsOpen, setDetailsOpen] = useState(false);
  const [selectedItem, setSelectedItem] = useState<ExtractionResult | null>(null);

  const columns = [
    columnHelper.display({
      id: 'select',
      cell: ({ row }) => (
        <input
          type="checkbox"
          checked={selectedRows.has(row.original.id)}
          onChange={(e) => {
            const newSelected = new Set(selectedRows);
            if (e.target.checked) newSelected.add(row.original.id);
            else newSelected.delete(row.original.id);
            setSelectedRows(newSelected);
          }}
        />
      ),
      header: '선택',
    }),
    columnHelper.accessor('documentName', {
      cell: (info) => (
        <button className="text-blue-600 underline" onClick={() => {
          setSelectedItem(info.row.original);
          setDetailsOpen(true);
        }}>
          {info.getValue()}
        </button>
      ),
      header: '문서명',
    }),
    columnHelper.accessor('documentType', {
      cell: (info) => <span className="text-sm">{info.getValue()}</span>,
      header: 'DocumentType',
    }),
    columnHelper.accessor('extractedAt', {
      cell: (info) => new Date(info.getValue()).toLocaleString('ko-KR'),
      header: '추출 시각',
    }),
    columnHelper.accessor('status', {
      cell: (info) => (
        <Badge
          className={
            info.getValue() === 'pending'
              ? 'bg-yellow-100'
              : info.getValue() === 'approved'
              ? 'bg-green-100'
              : 'bg-red-100'
          }
        >
          {info.getValue() === 'pending' && '대기'}
          {info.getValue() === 'approved' && '승인'}
          {info.getValue() === 'rejected' && '반려'}
        </Badge>
      ),
      header: '상태',
    }),
  ];

  const table = useReactTable({
    data: results,
    columns,
    getCoreRowModel: getCoreRowModel(),
  });

  return (
    <>
      <AdminTable table={table} />
      {detailsOpen && selectedItem && (
        <ExtractionReviewModal item={selectedItem} onClose={() => setDetailsOpen(false)} />
      )}
      {selectedRows.size > 0 && (
        <div className="mt-4 flex gap-2">
          <button className="px-4 py-2 bg-green-600 text-white rounded">
            선택 항목 모두 승인 ({selectedRows.size}개)
          </button>
          <button className="px-4 py-2 bg-red-600 text-white rounded">
            선택 항목 모두 반려
          </button>
        </div>
      )}
    </>
  );
}
```

#### 4-2-4. 검토 상세 모달
**컴포넌트**: `components/admin/ExtractionReviewModal.tsx`

**섹션**:

1. **원본 문서 미리보기**
   - 파일 타입별 처리:
     - PDF: iframe 또는 react-pdf 라이브러리 (선택사항)
     - 텍스트: 마크다운 미리보기
     - 이미지: 이미지 표시
   - 높이: 300-400px (스크롤 가능)

2. **추출된 필드 값 테이블**
   - **칼럼**: 필드명, 추출된 값, 승인 여부
   - **기능**:
     - 필드별 "승인" / "수정" / "삭제" 버튼
     - "수정" 클릭 시 텍스트 영역에서 값 수정 가능
     - Span 역참조: Phase 8 FG8.3에서 원본 텍스트의 위치 하이라이트 추가 예정

3. **하단 액션**
   - "모두 승인" 버튼 (초록색)
   - "반려 사유" 텍스트 영역 + "반려" 버튼 (빨강색)
   - "취소" 버튼

**코드 예시**:
```typescript
// components/admin/ExtractionReviewModal.tsx
import { useState } from 'react';
import { ExtractionResult, ExtractedField } from '@/types/extraction';
import { Dialog, DialogContent, DialogHeader, DialogTitle } from '@/components/ui/dialog';

interface Props {
  item: ExtractionResult;
  onClose: () => void;
  onApprove?: (approvedFields: ExtractedField[]) => void;
  onReject?: (reason: string) => void;
}

export function ExtractionReviewModal({ item, onClose, onApprove, onReject }: Props) {
  const [fields, setFields] = useState<ExtractedField[]>(item.fields);
  const [rejectReason, setRejectReason] = useState('');
  const [editingFieldId, setEditingFieldId] = useState<string | null>(null);

  const handleApproveField = (fieldId: string) => {
    setFields(
      fields.map((f) => (f.id === fieldId ? { ...f, approved: true } : f))
    );
  };

  const handleRejectField = (fieldId: string) => {
    setFields(
      fields.map((f) => (f.id === fieldId ? { ...f, approved: false } : f))
    );
  };

  const handleApproveAll = () => {
    onApprove?.(fields.map((f) => ({ ...f, approved: true })));
    onClose();
  };

  const handleRejectAll = () => {
    onReject?.(rejectReason);
    onClose();
  };

  return (
    <Dialog open onOpenChange={onClose}>
      <DialogContent className="max-w-4xl max-h-screen overflow-y-auto">
        <DialogHeader>
          <DialogTitle>{item.documentName} 검토</DialogTitle>
        </DialogHeader>

        {/* 문서 미리보기 */}
        <div className="border rounded p-4 bg-gray-50 h-80 overflow-y-auto">
          <DocumentPreview document={item.document} />
        </div>

        {/* 추출 필드 테이블 */}
        <div className="space-y-2">
          <h3 className="font-semibold">추출된 필드</h3>
          <table className="w-full border-collapse border border-gray-300 text-sm">
            <thead>
              <tr className="bg-gray-100">
                <th className="border p-2">필드명</th>
                <th className="border p-2">값</th>
                <th className="border p-2">작업</th>
              </tr>
            </thead>
            <tbody>
              {fields.map((field) => (
                <tr key={field.id} className={field.approved ? 'bg-green-50' : 'bg-red-50'}>
                  <td className="border p-2 font-semibold">{field.name}</td>
                  <td className="border p-2">
                    {editingFieldId === field.id ? (
                      <textarea
                        value={field.value}
                        onChange={(e) => {
                          setFields(
                            fields.map((f) =>
                              f.id === field.id ? { ...f, value: e.target.value } : f
                            )
                          );
                        }}
                        className="w-full border rounded px-2 py-1 text-xs"
                      />
                    ) : (
                      <span className="text-xs">{field.value}</span>
                    )}
                  </td>
                  <td className="border p-2">
                    {editingFieldId === field.id ? (
                      <button
                        onClick={() => setEditingFieldId(null)}
                        className="text-xs px-2 py-1 bg-blue-100 rounded"
                      >
                        완료
                      </button>
                    ) : (
                      <div className="flex gap-1">
                        <button
                          onClick={() => handleApproveField(field.id)}
                          className="text-xs px-2 py-1 bg-green-100 rounded"
                        >
                          승인
                        </button>
                        <button
                          onClick={() => setEditingFieldId(field.id)}
                          className="text-xs px-2 py-1 bg-blue-100 rounded"
                        >
                          수정
                        </button>
                        <button
                          onClick={() => handleRejectField(field.id)}
                          className="text-xs px-2 py-1 bg-red-100 rounded"
                        >
                          거절
                        </button>
                      </div>
                    )}
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>

        {/* 반려 사유 */}
        <div className="space-y-2">
          <label className="font-semibold">반려 사유 (선택)</label>
          <textarea
            value={rejectReason}
            onChange={(e) => setRejectReason(e.target.value)}
            placeholder="반려 사유를 입력하세요"
            className="w-full border rounded px-3 py-2 text-sm"
          />
        </div>

        {/* 하단 액션 */}
        <div className="flex gap-2 justify-end">
          <button onClick={onClose} className="px-4 py-2 border rounded">
            취소
          </button>
          <button
            onClick={handleRejectAll}
            disabled={!rejectReason}
            className="px-4 py-2 bg-red-600 text-white rounded disabled:bg-gray-300"
          >
            반려
          </button>
          <button
            onClick={handleApproveAll}
            className="px-4 py-2 bg-green-600 text-white rounded"
          >
            모두 승인
          </button>
        </div>
      </DialogContent>
    </Dialog>
  );
}

// 문서 타입별 미리보기 컴포넌트
function DocumentPreview({ document }: { document: Document }) {
  if (document.type === 'pdf') {
    return <iframe src={document.url} className="w-full h-full" />;
  }
  if (document.type === 'markdown' || document.type === 'text') {
    return <pre className="text-xs whitespace-pre-wrap">{document.content}</pre>;
  }
  if (document.type === 'image') {
    return <img src={document.url} alt="원본 문서" className="max-w-full h-auto" />;
  }
  return <p className="text-gray-500">미지원 파일 형식</p>;
}
```

---

### 4-3. Mock API 서비스 구축

#### 4-3-1. 추출 Mock API
**파일**: `services/mockApi/extractionMockApi.ts`

**인터페이스**:
```typescript
// services/mockApi/extractionMockApi.ts
export interface ExtractionField {
  id: string;
  name: string;
  type: 'string' | 'number' | 'boolean' | 'array' | 'object';
  required: boolean;
  description?: string;
}

export interface ExtractionSchema {
  id: string;
  documentType: string;
  fields: ExtractionField[];
  extractionMode: 'deterministic' | 'probabilistic';
  isActive: boolean;
  fieldCount: number;
  updatedAt: string;
}

export interface ExtractedField {
  id: string;
  name: string;
  value: string;
  approved?: boolean;
  spanReference?: {
    start: number;
    end: number;
  };
}

export interface Document {
  id: string;
  type: 'pdf' | 'text' | 'markdown' | 'image';
  url?: string;
  content?: string;
}

export interface ExtractionResult {
  id: string;
  documentName: string;
  documentType: string;
  document: Document;
  fields: ExtractedField[];
  extractedAt: string;
  status: 'pending' | 'approved' | 'rejected';
  rejectionReason?: string;
}

export interface ExtractionFilter {
  documentType?: string;
  status?: 'pending' | 'approved' | 'rejected';
  dateRange?: string;
}

export const extractionMockApi = {
  async listSchemas(): Promise<ExtractionSchema[]> {
    // 스키마 목록
  },
  async getSchema(documentType: string): Promise<ExtractionSchema> {
    // 스키마 상세
  },
  async updateSchema(documentType: string, schema: ExtractionSchema): Promise<ExtractionSchema> {
    // 스키마 수정
  },
  async listPendingResults(filter?: ExtractionFilter): Promise<ExtractionResult[]> {
    // 미승인 결과 목록
  },
  async getPendingResult(id: string): Promise<ExtractionResult> {
    // 미승인 결과 상세
  },
  async approveResult(id: string, approvedFields: ExtractedField[]): Promise<ExtractionResult> {
    // 승인
  },
  async rejectResult(id: string, reason: string): Promise<ExtractionResult> {
    // 반려
  },
};
```

**목 데이터** (2~3개 스키마, 5~10개 결과):
```typescript
// Mock 데이터 예시
const mockSchemas: ExtractionSchema[] = [
  {
    id: 'schema-001',
    documentType: 'Invoice',
    fields: [
      { id: 'field-001', name: 'invoice_number', type: 'string', required: true, description: '인보이스 번호' },
      { id: 'field-002', name: 'amount', type: 'number', required: true, description: '금액' },
      { id: 'field-003', name: 'due_date', type: 'string', required: false, description: '지불일' },
    ],
    extractionMode: 'deterministic',
    isActive: true,
    fieldCount: 3,
    updatedAt: '2026-04-10T14:30:00Z',
  },
  // ... 추가 스키마
];

const mockResults: ExtractionResult[] = [
  {
    id: 'result-001',
    documentName: 'Invoice_2026_001.pdf',
    documentType: 'Invoice',
    document: {
      id: 'doc-001',
      type: 'pdf',
      url: '/mock-docs/invoice-001.pdf',
    },
    fields: [
      { id: 'field-001', name: 'invoice_number', value: 'INV-2026-001' },
      { id: 'field-002', name: 'amount', value: '10000' },
      { id: 'field-003', name: 'due_date', value: '2026-05-10' },
    ],
    extractedAt: '2026-04-17T10:00:00Z',
    status: 'pending',
  },
  // ... 추가 결과
];
```

---

### 4-4. 타입 정의

**파일**: `types/extraction.ts`

```typescript
// types/extraction.ts
// 4-3-1에서 정의한 인터페이스 모두 포함
// Task 6-9의 types/evaluation.ts와 분리 유지

export interface ExtractionField {
  // ... 위 참조
}

export interface ExtractionSchema {
  // ... 위 참조
}

export interface ExtractionResult {
  // ... 위 참조
}

// ... 기타 타입
```

---

### 4-5. 라우팅 및 네비게이션

#### 4-5-1. 라우트 구성
```
/admin
├── /admin/extraction-schemas (새)
├── /admin/extraction-review (새)
└── (Task 6-9) /admin/golden-sets, /admin/evaluations, ...
```

#### 4-5-2. 사이드바 메뉴
Task 6-9와 함께 추가 (이미 계획됨):
```
평가·추출
├── 골든셋 (Task 6-9)
├── 평가 결과 (Task 6-9)
├── 추출 스키마 (Task 6-10)
└── 추출 검토 (Task 6-10)
```

---

### 4-6. 단위 테스트 (Jest + React Testing Library)

#### 4-6-1. 스키마 테이블 테스트
**파일**: `components/admin/__tests__/ExtractionSchemaTable.test.tsx`

```typescript
import { render, screen } from '@testing-library/react';
import { ExtractionSchemaTable } from '@/components/admin/ExtractionSchemaTable';

describe('ExtractionSchemaTable', () => {
  it('should render schema table with correct columns', () => {
    render(<ExtractionSchemaTable />);
    
    expect(screen.getByText('DocumentType')).toBeInTheDocument();
    expect(screen.getByText('필드 수')).toBeInTheDocument();
    expect(screen.getByText('수정일')).toBeInTheDocument();
  });

  it('should display mock schemas', () => {
    render(<ExtractionSchemaTable />);
    
    expect(screen.getByText('Invoice')).toBeInTheDocument();
  });

  it('should open edit form on row click', async () => {
    const { user } = render(<ExtractionSchemaTable />);
    
    await user.click(screen.getByText('Invoice'));
    expect(screen.getByText('필드')).toBeInTheDocument();
  });
});
```

#### 4-6-2. 검토 테이블 테스트
**파일**: `components/admin/__tests__/ExtractionReviewTable.test.tsx`

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import { ExtractionReviewTable } from '@/components/admin/ExtractionReviewTable';

describe('ExtractionReviewTable', () => {
  it('should render extraction results table', async () => {
    render(<ExtractionReviewTable results={mockResults} />);
    
    await waitFor(() => {
      expect(screen.getByText('Invoice_2026_001.pdf')).toBeInTheDocument();
    });
  });

  it('should open review modal on row click', async () => {
    const { user } = render(<ExtractionReviewTable results={mockResults} />);
    
    await user.click(screen.getByText('Invoice_2026_001.pdf'));
    expect(screen.getByText(/검토/)).toBeInTheDocument();
  });

  it('should enable bulk actions when rows selected', async () => {
    const { user } = render(<ExtractionReviewTable results={mockResults} />);
    
    const checkbox = screen.getAllByRole('checkbox')[1];
    await user.click(checkbox);
    
    expect(screen.getByText(/선택 항목 모두 승인/)).toBeInTheDocument();
  });
});
```

#### 4-6-3. 검토 모달 테스트
**파일**: `components/admin/__tests__/ExtractionReviewModal.test.tsx`

```typescript
import { render, screen } from '@testing-library/react';
import { ExtractionReviewModal } from '@/components/admin/ExtractionReviewModal';

describe('ExtractionReviewModal', () => {
  it('should render document preview', () => {
    render(<ExtractionReviewModal item={mockResult} onClose={() => {}} />);
    
    expect(screen.getByText('Invoice_2026_001.pdf 검토')).toBeInTheDocument();
  });

  it('should render extracted fields table', () => {
    render(<ExtractionReviewModal item={mockResult} onClose={() => {}} />);
    
    expect(screen.getByText('invoice_number')).toBeInTheDocument();
    expect(screen.getByText('INV-2026-001')).toBeInTheDocument();
  });

  it('should approve field when approve button clicked', async () => {
    const { user } = render(
      <ExtractionReviewModal item={mockResult} onClose={() => {}} />
    );
    
    const approveButtons = screen.getAllByText('승인');
    await user.click(approveButtons[0]);
    
    expect(approveButtons[0].closest('tr')).toHaveClass('bg-green-50');
  });

  it('should call onApprove when all approved', async () => {
    const onApprove = jest.fn();
    const { user } = render(
      <ExtractionReviewModal item={mockResult} onClose={() => {}} onApprove={onApprove} />
    );
    
    await user.click(screen.getByText('모두 승인'));
    
    expect(onApprove).toHaveBeenCalled();
  });
});
```

---

### 4-7. 접근성 및 WCAG 2.1 AA 준수

#### 4-7-1. 체크리스트
- [ ] 모든 테이블에 `<caption>` 또는 aria-label 추가
- [ ] 모달 포커스 트래핑 구현
- [ ] 폼 필드에 레이블 태그 연결
- [ ] 버튼 텍스트 명확 (아이콘만 사용 금지)
- [ ] 색상 대비 4.5:1 (AA 기준)
- [ ] 키보드 네비게이션 지원

#### 4-7-2. 구현 예시
```typescript
// 테이블 aria-label
<table
  role="table"
  aria-label="추출 스키마 목록"
  aria-describedby="schema-table-desc"
>
  <caption id="schema-table-desc">DocumentType별 추출 스키마 정의</caption>

// 모달 포커스 트래핑
<Dialog open onOpenChange={onClose}>
  {/* Dialog 자동으로 포커스 트래핑 처리 (radix-ui) */}
</Dialog>

// 버튼 레이블
<button aria-label="문서 검토 시작">
  <Icon />
  검토
</button>
```

---

## 5. 산출물

### 5-1. 코드 산출물
1. **페이지**:
   - `app/admin/extraction-schemas/page.tsx`
   - `app/admin/extraction-review/page.tsx`

2. **컴포넌트**:
   - `components/admin/ExtractionSchemaTable.tsx`
   - `components/admin/ExtractionSchemaEditForm.tsx`
   - `components/admin/ExtractionReviewFilters.tsx`
   - `components/admin/ExtractionReviewTable.tsx`
   - `components/admin/ExtractionReviewModal.tsx`
   - `components/admin/DocumentPreview.tsx`

3. **Mock API**:
   - `services/mockApi/extractionMockApi.ts`

4. **타입 정의**:
   - `types/extraction.ts`

5. **테스트**:
   - `components/admin/__tests__/ExtractionSchemaTable.test.tsx`
   - `components/admin/__tests__/ExtractionSchemaEditForm.test.tsx`
   - `components/admin/__tests__/ExtractionReviewTable.test.tsx`
   - `components/admin/__tests__/ExtractionReviewModal.test.tsx`

### 5-2. 문서 산출물
- **작업 완료 보고서** (task6-10_보고서.md)
  - 구현된 기능 목록
  - 목 데이터 설명
  - 성능 지표
  - 테스트 커버리지
  - Phase 8 대기 기능 목록

- **보안 검토 보고서** (task6-10_보안검토.md)
  - Mock 데이터 격리
  - 민감 정보 처리 (없음)
  - 파일 업로드 보안 (Phase 8 후)

- **UI/UX 리뷰 보고서** (task6-10_UI검토_1차~5차.md)
  - 최소 5회 UI 리뷰 기록

- **Phase 8 핸드오프 문서** (task6-10_Phase8_API계약.md)
  - API 엔드포인트 명세
  - Mock API → 실제 API 교체 가이드
  - 필드 매핑 정보

### 5-3. 하드웨어 요구사항 없음

---

## 6. 완료 기준

### 기능 완료
- [ ] 스키마 목록 페이지 렌더링 (정렬, 필터 동작)
- [ ] 스키마 편집 폼 (필드 테이블, 추출 모드 표시)
- [ ] 검토 대기 목록 페이지 (정렬, 필터 동작)
- [ ] 검토 상세 모달 (문서 미리보기, 필드 값, 승인/반려 버튼)
- [ ] 필터 UI 동작 (DocumentType, 상태, 기간)
- [ ] Mock API 모두 구현 (스키마 2~3개, 결과 5~10개)
- [ ] 라우팅 정상 동작
- [ ] 사이드바 메뉴 추가 (Task 6-9와 함께)

### 테스트 완료
- [ ] 단위 테스트 성공율 100% (최소 8개 테스트)
- [ ] 접근성 검증 (WCAG 2.1 AA)
- [ ] E2E 테스트 (필수 플로우: 페이지 진입 → 데이터 로드 → 검토 → 승인/반려)

### 성능
- [ ] Lighthouse 성능 점수 ≥80
- [ ] FCP <1.5s
- [ ] 번들 크기 증가 <100KB

### 문서
- [ ] 코드 주석 (복잡한 로직에만)
- [ ] 타입 정의 완전 (any 금지)
- [ ] 작업 완료 보고서 + 보안 검토 + Phase 8 핸드오프 문서 작성

---

## 7. 작업 지침

### 7-1. 개발 환경 설정
1. `npm run dev` 실행
2. `http://localhost:3000/admin/extraction-schemas` 접근 확인
3. 콘솔 에러 없음 확인
4. Mock API 응답 정상 확인

### 7-2. S2 원칙 준수
- **원칙 ①**: DocumentType 동적 로드 (설정 파일에서)
- **원칙 ⑥**: Admin 권한 기반 접근 제어
- **원칙 ⑦**: 폐쇄망 환경 (Mock API 로컬 동작)

### 7-3. 목 데이터 마크 표시
```typescript
console.warn('[MOCK DATA] Extraction results loaded. This will be replaced in Phase 8.');
// UI 배지: [Mock] 표시
```

### 7-4. Phase 8 API 핸드오프
Mock API 주석에 실제 API 호출 방식 기록:
```typescript
/**
 * Phase 8 핸드오프:
 * Endpoint: GET /api/admin/extraction/results?status=pending
 * Response: { data: ExtractionResult[], total: number }
 */
```

### 7-5. UI 리뷰 (5회)
1. 레이아웃 및 그리드
2. 색상, 타이포그래피
3. 상호작용 및 UX
4. 접근성
5. 반응형 최적화

### 7-6. 테스트 작성
- Unit Test + E2E 테스트
- 접근성 테스트 (jest-axe)

### 7-7. 커밋 메시지
```
[FG6.3] feat: 추출 스키마 관리 + 검토 큐 스켈레톤 구축

- 스키마 관리 페이지 (목록, 편집 폼)
- 검토 대기 목록 페이지
- 검토 상세 모달 (문서 미리보기, 필드 승인/반려)
- Mock API 구현 (스키마 2개, 결과 8개)
```

### 7-8. 빌드 및 배포
```bash
npm run build
npm run analyze
npm run test
npm run lint
```

---

## 8. 참고 문헌 및 링크

- **Phase 6 개발계획서**: `/docs/개발문서/S2/phase6/Phase 6 개발계획서.md`
- **Task 6-9 작업지시서**: `task6-9.md`
- **TanStack Table**: https://tanstack.com/table/v8
- **shadcn/ui Dialog**: https://ui.shadcn.com/docs/components/dialog
- **WCAG 2.1**: https://www.w3.org/WAI/WCAG21/quickref/

---

**다음 단계**: Phase 7 (평가 인프라) 완료 후 API 바인딩, Phase 8 (Structured Extraction) 완료 후 API 바인딩
