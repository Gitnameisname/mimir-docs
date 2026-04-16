# Task 8-6. 추출 결과 검토 UI (React)

## 1. 작업 목적

Task 8-5의 추출 결과 검토 API를 사용하여 **인간-중심의 검토 UI**를 React로 구현한다. 사용자는 pending 추출 결과 목록을 보고, 각 결과에 대해 원문 미리보기와 추출 필드를 나란히 검토하며, 필드별로 승인/수정/거절하는 직관적 인터페이스를 제공한다. 데스크톱 및 웹 환경 모두 호환하고 (프로젝트 규칙), WCAG 2.1 AA 접근성 기준을 준수하며, **UI 리뷰를 5회 이상** 수행하여 점진적으로 개선한다.

## 2. 작업 범위

### 포함 범위

1. **ExtractionReviewPage 컴포넌트**
   - 추출 결과 pending 목록 조회 및 상태 필터 (pending/approved/rejected/modified)
   - 페이지네이션 (offset/limit)
   - 각 항목의 요약 카드: 문서명, 문서타입, 생성시간, 확신도 표시
   - 항목 클릭 시 상세 검토 화면으로 전환

2. **ExtractionReviewDetail 컴포넌트**
   - 분할 화면: 왼쪽 문서 미리보기 (읽기 전용), 오른쪽 추출 필드 테이블
   - 문서 미리보기: 제목, 문서타입, 전체 텍스트 (스크롤 가능)
   - 추출 필드 테이블:
     * 필드명, 자동 추출값, 신뢰도 배지 (%)
     * 필드별 액션 버튼: "승인", "수정", "거절"
     * 필드별 편집 모드 (인라인 또는 모달)
   - 필드 타입별 입력 위젯: string (text input), date (date picker), array (tag input), object (JSON editor)
   - 하단 버튼: "모두 승인", "모두 거절", "취소"

3. **필드 에디터 컴포넌트 (FieldEditor)**
   - String: text input (최대 길이 제약)
   - Date: date picker (YYYY-MM-DD 형식)
   - Integer/Float: number input
   - Array: tag input (add/remove 태그)
   - Object: JSON editor (Monaco Editor 또는 간단한 text)
   - Type mismatch 경고 표시

4. **상태 필터 탭**
   - Tabs: "검토 대기" (pending), "승인됨" (approved), "거절됨" (rejected), "수정됨" (modified)
   - 탭 클릭 시 목록 재로드

5. **Zustand 상태 관리**
   - `useExtractionStore`: pending 리스트, 선택된 항목, 필터, 페이지네이션
   - 액션: fetchPendingList, selectExtraction, updateFilter, paginationNext/prev

6. **React hooks**
   - `useExtractionCandidates(scope_id)`: pending 목록 조회 (API)
   - `useExtractionReview(extraction_id)`: 상세 데이터 조회
   - `useExtractionActions()`: 승인/수정/거절 액션 (API 호출)

7. **데이터 테이블 (TanStack Table)**
   - ExtractionReviewDetail의 필드 테이블을 TanStack Table로 구현
   - 컬럼: 필드명, 타입, 자동값, 신뢰도, 액션
   - 정렬 지원 (필드명, 신뢰도)
   - Row 선택 체크박스 (일괄 승인/거절용)

8. **실시간 업데이트 (SSE)**
   - ExtractionReviewPage: SSE로 new extractions 감지
   - 자동으로 목록 상단에 새 항목 추가 (또는 알림)
   - 사용자 거절 후 다른 사용자가 같은 항목을 보지 못하도록 상태 업데이트

9. **레이아웃 & 반응형 디자인**
   - Desktop (1920px): 분할 화면 50:50
   - Tablet (768px): 분할 화면 30:70 또는 stack (위/아래)
   - Mobile (320px): 단일 열 (문서 → 필드)
   - 네비게이션: 목록으로 돌아가기 버튼

10. **접근성 (WCAG 2.1 AA)**
    - Semantic HTML (button, table, form)
    - ARIA labels/roles (확신도 배지, 필드 에디터)
    - 키보드 네비게이션: Tab, Enter, Escape
    - 포커스 인디케이터 (고대비)
    - 스크린 리더 지원 (필드 설명, 에러 메시지)

11. **UI 리뷰 사이클 (최소 5회)**
    - Iteration 1: 기본 레이아웃 + 필드 테이블
    - Iteration 2: 필드 에디터 + 인라인 편집
    - Iteration 3: 분할 화면 문서 미리보기
    - Iteration 4: 상태 필터 + 페이지네이션
    - Iteration 5: 스타일링 + 접근성 + 반응형 조정

### 제외 범위

- 추출 파이프라인 (Task 8-4)
- 추출 결과 검토 API (Task 8-5)
- 추출 재실행 배치 (Task 8-7)

## 3. 선행 조건

- Task 8-5 완료: 추출 결과 검토 API (/api/v1/extractions/*)
- React 18+, TypeScript, Vite 설정 완료
- Zustand, TanStack Table, TailwindCSS 설치
- axios 또는 fetch API 설정

## 4. 주요 작업 항목

### 4-1. 타입 정의 (TypeScript)

**파일:** `/frontend/src/types/extraction.ts`

```typescript
/**
 * 추출 결과 검토 UI의 타입 정의.
 */

export enum ExtractionStatus {
  PENDING = "pending",
  APPROVED = "approved",
  REJECTED = "rejected",
  MODIFIED = "modified",
}

export interface ConfidenceScore {
  field_name: string;
  confidence: number;  // 0.0 ~ 1.0
  reason?: string;
}

export interface ExtractionCandidate {
  id: string;
  document_id: string;
  document_version: number;
  extraction_schema_id: string;
  extraction_schema_version: number;
  extracted_fields: Record<string, any>;
  confidence_scores: ConfidenceScore[];
  status: ExtractionStatus;
  extraction_model: string;
  extraction_latency_ms: number;
  created_at: string;
  updated_at: string;
}

export interface DocumentPreview {
  id: string;
  title?: string;
  type: string;
  content_text: string;
  created_at: string;
}

export interface HumanEdit {
  field_name: string;
  before_value: any;
  after_value: any;
  edited_at: string;
  edited_by: string;
  reason?: string;
}

export interface ApprovedExtraction {
  id: string;
  document_id: string;
  approved_fields: Record<string, any>;
  human_edits: HumanEdit[];
  approved_by: string;
  approved_at: string;
  approval_comment?: string;
}

export interface ExtractionField {
  name: string;
  type: string;
  value: any;
  confidence: number;
  required: boolean;
  reason?: string;
}

export interface ExtractionListResponse {
  total_count: number;
  offset: number;
  limit: number;
  items: ExtractionCandidate[];
}

export interface ApproveRequest {
  approval_comment?: string;
}

export interface ModifyRequest {
  modifications: Record<string, any>;
  reasons?: Record<string, string>;
  approval_comment?: string;
}

export interface RejectRequest {
  reason: string;
}

export interface BatchApproveRequest {
  extraction_ids: string[];
  approval_comment?: string;
}

export interface BatchRejectRequest {
  extraction_ids: string[];
  reason: string;
}
```

### 4-2. Zustand 스토어

**파일:** `/frontend/src/stores/extractionStore.ts`

```typescript
/**
 * Zustand 상태 관리: 추출 결과 검토.
 */

import { create } from "zustand";
import { ExtractionCandidate, ExtractionStatus } from "@/types/extraction";

interface ExtractionState {
  // ==================== 데이터 ====================
  pendingList: ExtractionCandidate[];
  selectedExtraction: ExtractionCandidate | null;
  filteredList: ExtractionCandidate[];
  
  // ==================== UI 상태 ====================
  currentStatus: ExtractionStatus;
  currentOffset: number;
  currentLimit: number;
  totalCount: number;
  isLoading: boolean;
  error: string | null;
  
  // ==================== 액션 ====================
  fetchPendingList: (offset: number, limit: number, status: ExtractionStatus) => Promise<void>;
  selectExtraction: (extraction: ExtractionCandidate) => void;
  deselectExtraction: () => void;
  
  setCurrentStatus: (status: ExtractionStatus) => void;
  nextPage: () => Promise<void>;
  prevPage: () => Promise<void>;
  
  updateListItem: (id: string, updates: Partial<ExtractionCandidate>) => void;
  removeListItem: (id: string) => void;
  
  setError: (error: string | null) => void;
  clearStore: () => void;
}

export const useExtractionStore = create<ExtractionState>((set, get) => ({
  // ==================== 초기 상태 ====================
  pendingList: [],
  selectedExtraction: null,
  filteredList: [],
  currentStatus: ExtractionStatus.PENDING,
  currentOffset: 0,
  currentLimit: 50,
  totalCount: 0,
  isLoading: false,
  error: null,
  
  // ==================== 액션 ====================
  fetchPendingList: async (offset, limit, status) => {
    set({ isLoading: true, error: null });
    
    try {
      const response = await fetch(
        `/api/v1/extractions/pending?offset=${offset}&limit=${limit}`,
        {
          headers: {
            "Authorization": `Bearer ${localStorage.getItem("auth_token")}`,
          },
        }
      );
      
      if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
      }
      
      const data = await response.json();
      
      set({
        pendingList: data.items,
        filteredList: data.items.filter(item => item.status === status),
        currentOffset: offset,
        currentLimit: limit,
        totalCount: data.total_count,
        currentStatus: status,
        isLoading: false,
      });
    } catch (error) {
      const errorMsg = error instanceof Error ? error.message : "Unknown error";
      set({ error: errorMsg, isLoading: false });
    }
  },
  
  selectExtraction: (extraction) => {
    set({ selectedExtraction: extraction });
  },
  
  deselectExtraction: () => {
    set({ selectedExtraction: null });
  },
  
  setCurrentStatus: (status) => {
    set({ currentStatus: status, currentOffset: 0 });
    
    // 즉시 목록 재로드
    const state = get();
    state.fetchPendingList(0, state.currentLimit, status);
  },
  
  nextPage: async () => {
    const state = get();
    const nextOffset = state.currentOffset + state.currentLimit;
    
    if (nextOffset >= state.totalCount) {
      set({ error: "No more items" });
      return;
    }
    
    await state.fetchPendingList(nextOffset, state.currentLimit, state.currentStatus);
  },
  
  prevPage: async () => {
    const state = get();
    const prevOffset = Math.max(0, state.currentOffset - state.currentLimit);
    
    await state.fetchPendingList(prevOffset, state.currentLimit, state.currentStatus);
  },
  
  updateListItem: (id, updates) => {
    set((state) => ({
      pendingList: state.pendingList.map((item) =>
        item.id === id ? { ...item, ...updates } : item
      ),
      filteredList: state.filteredList.map((item) =>
        item.id === id ? { ...item, ...updates } : item
      ),
    }));
  },
  
  removeListItem: (id) => {
    set((state) => ({
      pendingList: state.pendingList.filter((item) => item.id !== id),
      filteredList: state.filteredList.filter((item) => item.id !== id),
      totalCount: Math.max(0, state.totalCount - 1),
    }));
  },
  
  setError: (error) => {
    set({ error });
  },
  
  clearStore: () => {
    set({
      pendingList: [],
      selectedExtraction: null,
      filteredList: [],
      currentStatus: ExtractionStatus.PENDING,
      currentOffset: 0,
      currentLimit: 50,
      totalCount: 0,
      isLoading: false,
      error: null,
    });
  },
}));
```

### 4-3. React Hooks

**파일:** `/frontend/src/hooks/useExtractionActions.ts`

```typescript
/**
 * 추출 결과 검토 API 호출 hooks.
 */

import { useState } from "react";
import {
  ApproveRequest,
  ModifyRequest,
  RejectRequest,
  BatchApproveRequest,
  BatchRejectRequest,
} from "@/types/extraction";

interface UseExtractionActionsReturn {
  approveExtraction: (
    extractionId: string,
    req: ApproveRequest
  ) => Promise<any>;
  modifyExtraction: (
    extractionId: string,
    req: ModifyRequest
  ) => Promise<any>;
  rejectExtraction: (
    extractionId: string,
    req: RejectRequest
  ) => Promise<void>;
  batchApprove: (req: BatchApproveRequest) => Promise<any>;
  batchReject: (req: BatchRejectRequest) => Promise<any>;
  isLoading: boolean;
  error: string | null;
}

export const useExtractionActions = (): UseExtractionActionsReturn => {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  const baseUrl = "/api/v1/extractions";
  const authToken = localStorage.getItem("auth_token");
  
  const approveExtraction = async (
    extractionId: string,
    req: ApproveRequest
  ) => {
    setIsLoading(true);
    setError(null);
    
    try {
      const response = await fetch(`${baseUrl}/${extractionId}/approve`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${authToken}`,
        },
        body: JSON.stringify(req),
      });
      
      if (!response.ok) {
        throw new Error(`Approval failed: ${response.status}`);
      }
      
      return await response.json();
    } catch (err) {
      const errorMsg = err instanceof Error ? err.message : "Unknown error";
      setError(errorMsg);
      throw err;
    } finally {
      setIsLoading(false);
    }
  };
  
  const modifyExtraction = async (
    extractionId: string,
    req: ModifyRequest
  ) => {
    setIsLoading(true);
    setError(null);
    
    try {
      const response = await fetch(`${baseUrl}/${extractionId}/modify`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${authToken}`,
        },
        body: JSON.stringify(req),
      });
      
      if (!response.ok) {
        throw new Error(`Modification failed: ${response.status}`);
      }
      
      return await response.json();
    } catch (err) {
      const errorMsg = err instanceof Error ? err.message : "Unknown error";
      setError(errorMsg);
      throw err;
    } finally {
      setIsLoading(false);
    }
  };
  
  const rejectExtraction = async (
    extractionId: string,
    req: RejectRequest
  ) => {
    setIsLoading(true);
    setError(null);
    
    try {
      const response = await fetch(`${baseUrl}/${extractionId}/reject`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${authToken}`,
        },
        body: JSON.stringify(req),
      });
      
      if (!response.ok) {
        throw new Error(`Rejection failed: ${response.status}`);
      }
    } catch (err) {
      const errorMsg = err instanceof Error ? err.message : "Unknown error";
      setError(errorMsg);
      throw err;
    } finally {
      setIsLoading(false);
    }
  };
  
  const batchApprove = async (req: BatchApproveRequest) => {
    setIsLoading(true);
    setError(null);
    
    try {
      const response = await fetch(`${baseUrl}/batch-approve`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${authToken}`,
        },
        body: JSON.stringify(req),
      });
      
      if (!response.ok) {
        throw new Error(`Batch approval failed: ${response.status}`);
      }
      
      return await response.json();
    } catch (err) {
      const errorMsg = err instanceof Error ? err.message : "Unknown error";
      setError(errorMsg);
      throw err;
    } finally {
      setIsLoading(false);
    }
  };
  
  const batchReject = async (req: BatchRejectRequest) => {
    setIsLoading(true);
    setError(null);
    
    try {
      const response = await fetch(`${baseUrl}/batch-reject`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${authToken}`,
        },
        body: JSON.stringify(req),
      });
      
      if (!response.ok) {
        throw new Error(`Batch rejection failed: ${response.status}`);
      }
      
      return await response.json();
    } catch (err) {
      const errorMsg = err instanceof Error ? err.message : "Unknown error";
      setError(errorMsg);
      throw err;
    } finally {
      setIsLoading(false);
    }
  };
  
  return {
    approveExtraction,
    modifyExtraction,
    rejectExtraction,
    batchApprove,
    batchReject,
    isLoading,
    error,
  };
};
```

### 4-4. FieldEditor 컴포넌트

**파File:** `/frontend/src/components/extraction/FieldEditor.tsx`

```typescript
/**
 * 필드 편집 컴포넌트.
 * 
 * 타입별 입력 위젯:
 * - string: text input
 * - date: date picker
 * - integer/float: number input
 * - array: tag input
 * - object: JSON editor
 */

import React, { useState } from "react";

interface FieldEditorProps {
  fieldName: string;
  fieldType: string;
  currentValue: any;
  onSave: (newValue: any) => void;
  onCancel: () => void;
  required?: boolean;
  maxLength?: number;
}

export const FieldEditor: React.FC<FieldEditorProps> = ({
  fieldName,
  fieldType,
  currentValue,
  onSave,
  onCancel,
  required = false,
  maxLength,
}) => {
  const [value, setValue] = useState(currentValue);
  const [error, setError] = useState<string | null>(null);
  
  const handleSave = () => {
    setError(null);
    
    // 타입 검증
    if (required && (value === null || value === undefined || value === "")) {
      setError("This field is required");
      return;
    }
    
    // 타입별 검증
    switch (fieldType) {
      case "string":
        if (maxLength && value.length > maxLength) {
          setError(`Maximum length is ${maxLength}`);
          return;
        }
        break;
      
      case "integer":
        if (isNaN(parseInt(value))) {
          setError("Invalid integer");
          return;
        }
        break;
      
      case "float":
        if (isNaN(parseFloat(value))) {
          setError("Invalid number");
          return;
        }
        break;
      
      case "date":
        if (!/^\d{4}-\d{2}-\d{2}$/.test(value)) {
          setError("Invalid date format (YYYY-MM-DD)");
          return;
        }
        break;
      
      case "array":
        if (!Array.isArray(value)) {
          setError("Invalid array format");
          return;
        }
        break;
    }
    
    onSave(value);
  };
  
  // ==================== String ====================
  if (fieldType === "string") {
    return (
      <div className="space-y-2">
        <label className="block text-sm font-medium text-gray-700">
          {fieldName}
          {required && <span className="text-red-500">*</span>}
        </label>
        <textarea
          value={value || ""}
          onChange={(e) => setValue(e.target.value)}
          maxLength={maxLength}
          rows={4}
          className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-blue-500"
          aria-label={`Edit field: ${fieldName}`}
          aria-required={required}
        />
        {maxLength && (
          <p className="text-xs text-gray-500">
            {value?.length || 0} / {maxLength}
          </p>
        )}
        {error && <p className="text-sm text-red-600">{error}</p>}
        <div className="flex gap-2 justify-end">
          <button
            onClick={onCancel}
            className="px-3 py-1 text-sm bg-gray-200 hover:bg-gray-300 rounded"
            aria-label="Cancel edit"
          >
            Cancel
          </button>
          <button
            onClick={handleSave}
            className="px-3 py-1 text-sm bg-blue-600 hover:bg-blue-700 text-white rounded"
            aria-label="Save changes"
          >
            Save
          </button>
        </div>
      </div>
    );
  }
  
  // ==================== Date ====================
  if (fieldType === "date") {
    return (
      <div className="space-y-2">
        <label className="block text-sm font-medium text-gray-700">
          {fieldName}
          {required && <span className="text-red-500">*</span>}
        </label>
        <input
          type="date"
          value={value || ""}
          onChange={(e) => setValue(e.target.value)}
          className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-blue-500"
          aria-label={`Edit date field: ${fieldName}`}
          aria-required={required}
        />
        {error && <p className="text-sm text-red-600">{error}</p>}
        <div className="flex gap-2 justify-end">
          <button
            onClick={onCancel}
            className="px-3 py-1 text-sm bg-gray-200 hover:bg-gray-300 rounded"
          >
            Cancel
          </button>
          <button
            onClick={handleSave}
            className="px-3 py-1 text-sm bg-blue-600 hover:bg-blue-700 text-white rounded"
          >
            Save
          </button>
        </div>
      </div>
    );
  }
  
  // ==================== Number ====================
  if (fieldType === "integer" || fieldType === "float") {
    return (
      <div className="space-y-2">
        <label className="block text-sm font-medium text-gray-700">
          {fieldName}
          {required && <span className="text-red-500">*</span>}
        </label>
        <input
          type="number"
          value={value || ""}
          onChange={(e) => setValue(e.target.value)}
          step={fieldType === "float" ? "0.01" : "1"}
          className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-blue-500"
          aria-label={`Edit number field: ${fieldName}`}
          aria-required={required}
        />
        {error && <p className="text-sm text-red-600">{error}</p>}
        <div className="flex gap-2 justify-end">
          <button
            onClick={onCancel}
            className="px-3 py-1 text-sm bg-gray-200 hover:bg-gray-300 rounded"
          >
            Cancel
          </button>
          <button
            onClick={handleSave}
            className="px-3 py-1 text-sm bg-blue-600 hover:bg-blue-700 text-white rounded"
          >
            Save
          </button>
        </div>
      </div>
    );
  }
  
  // ==================== Array ====================
  if (fieldType === "array") {
    const [tags, setTags] = useState<string[]>(Array.isArray(value) ? value : []);
    const [tagInput, setTagInput] = useState("");
    
    const addTag = () => {
      if (tagInput.trim()) {
        setTags([...tags, tagInput.trim()]);
        setTagInput("");
      }
    };
    
    const removeTag = (index: number) => {
      setTags(tags.filter((_, i) => i !== index));
    };
    
    const handleSaveArray = () => {
      onSave(tags);
    };
    
    return (
      <div className="space-y-2">
        <label className="block text-sm font-medium text-gray-700">
          {fieldName}
          {required && <span className="text-red-500">*</span>}
        </label>
        
        {/* Tag Display */}
        <div className="flex flex-wrap gap-2 mb-2">
          {tags.map((tag, index) => (
            <span
              key={index}
              className="inline-flex items-center gap-1 px-2 py-1 bg-blue-100 text-blue-800 rounded-full text-sm"
              role="button"
            >
              {tag}
              <button
                onClick={() => removeTag(index)}
                className="text-blue-600 hover:text-blue-900"
                aria-label={`Remove tag: ${tag}`}
              >
                ×
              </button>
            </span>
          ))}
        </div>
        
        {/* Tag Input */}
        <div className="flex gap-1">
          <input
            type="text"
            value={tagInput}
            onChange={(e) => setTagInput(e.target.value)}
            onKeyPress={(e) => e.key === "Enter" && addTag()}
            placeholder="Enter tag and press Enter"
            className="flex-1 px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-blue-500"
            aria-label={`Add tag to ${fieldName}`}
          />
          <button
            onClick={addTag}
            className="px-3 py-2 bg-gray-200 hover:bg-gray-300 rounded"
          >
            Add
          </button>
        </div>
        
        {error && <p className="text-sm text-red-600">{error}</p>}
        
        <div className="flex gap-2 justify-end">
          <button
            onClick={onCancel}
            className="px-3 py-1 text-sm bg-gray-200 hover:bg-gray-300 rounded"
          >
            Cancel
          </button>
          <button
            onClick={handleSaveArray}
            className="px-3 py-1 text-sm bg-blue-600 hover:bg-blue-700 text-white rounded"
          >
            Save
          </button>
        </div>
      </div>
    );
  }
  
  // Default: object or unknown
  return (
    <div className="space-y-2">
      <label className="block text-sm font-medium text-gray-700">
        {fieldName}
        {required && <span className="text-red-500">*</span>}
      </label>
      <textarea
        value={JSON.stringify(value, null, 2)}
        onChange={(e) => {
          try {
            setValue(JSON.parse(e.target.value));
            setError(null);
          } catch (err) {
            setError("Invalid JSON");
          }
        }}
        rows={6}
        className="w-full px-3 py-2 border border-gray-300 rounded-md font-mono text-sm focus:outline-none focus:ring-blue-500"
        aria-label={`Edit JSON field: ${fieldName}`}
        aria-required={required}
      />
      {error && <p className="text-sm text-red-600">{error}</p>}
      <div className="flex gap-2 justify-end">
        <button
          onClick={onCancel}
          className="px-3 py-1 text-sm bg-gray-200 hover:bg-gray-300 rounded"
        >
          Cancel
        </button>
        <button
          onClick={handleSave}
          className="px-3 py-1 text-sm bg-blue-600 hover:bg-blue-700 text-white rounded"
        >
          Save
        </button>
      </div>
    </div>
  );
};
```

### 4-5. ExtractionReviewDetail 컴포넌트

**파일:** `/frontend/src/components/extraction/ExtractionReviewDetail.tsx`

```typescript
/**
 * 추출 결과 상세 검토 컴포넌트.
 * 
 * 분할 화면:
 * - 왼쪽: 문서 미리보기 (읽기 전용)
 * - 오른쪽: 추출 필드 테이블 (검토 + 편집)
 */

import React, { useState, useMemo } from "react";
import {
  useReactTable,
  getCoreRowModel,
  ColumnDef,
  flexRender,
} from "@tanstack/react-table";
import { ExtractionCandidate, ExtractionField } from "@/types/extraction";
import { FieldEditor } from "./FieldEditor";
import { useExtractionActions } from "@/hooks/useExtractionActions";
import { useExtractionStore } from "@/stores/extractionStore";

interface ExtractionReviewDetailProps {
  extraction: ExtractionCandidate;
  documentPreview: {
    title?: string;
    content_text: string;
    type: string;
  };
  onBack: () => void;
}

export const ExtractionReviewDetail: React.FC<ExtractionReviewDetailProps> = ({
  extraction,
  documentPreview,
  onBack,
}) => {
  const [editingField, setEditingField] = useState<string | null>(null);
  const [modifications, setModifications] = useState<Record<string, any>>({});
  const [selectedRows, setSelectedRows] = useState<Set<string>>(new Set());
  const [feedbackReason, setFeedbackReason] = useState("");
  
  const { approveExtraction, modifyExtraction, rejectExtraction, isLoading } =
    useExtractionActions();
  const { updateListItem, removeListItem } = useExtractionStore();
  
  // ==================== Fields Data ====================
  const fields: ExtractionField[] = useMemo(() => {
    return Object.entries(extraction.extracted_fields).map(([name, value]) => {
      const confidence = extraction.confidence_scores.find(
        (cs) => cs.field_name === name
      );
      
      return {
        name,
        type: typeof value,
        value,
        confidence: confidence?.confidence || 0,
        required: false,  // TODO: load from schema
        reason: confidence?.reason,
      };
    });
  }, [extraction]);
  
  // ==================== Table Columns ====================
  const columns: ColumnDef<ExtractionField>[] = useMemo(
    () => [
      {
        id: "select",
        header: () => (
          <input
            type="checkbox"
            onChange={(e) => {
              if (e.target.checked) {
                setSelectedRows(new Set(fields.map((f) => f.name)));
              } else {
                setSelectedRows(new Set());
              }
            }}
            aria-label="Select all fields"
          />
        ),
        cell: ({ row }) => (
          <input
            type="checkbox"
            checked={selectedRows.has(row.original.name)}
            onChange={(e) => {
              const newRows = new Set(selectedRows);
              if (e.target.checked) {
                newRows.add(row.original.name);
              } else {
                newRows.delete(row.original.name);
              }
              setSelectedRows(newRows);
            }}
            aria-label={`Select ${row.original.name}`}
          />
        ),
      },
      {
        accessorKey: "name",
        header: "Field Name",
        cell: ({ getValue }) => (
          <span className="font-medium">{getValue() as string}</span>
        ),
      },
      {
        accessorKey: "type",
        header: "Type",
        cell: ({ getValue }) => (
          <span className="text-xs bg-gray-100 px-2 py-1 rounded">
            {getValue() as string}
          </span>
        ),
      },
      {
        accessorKey: "value",
        header: "Auto-Extracted Value",
        cell: ({ row }) => {
          const modified = modifications[row.original.name];
          const displayValue = modified !== undefined ? modified : row.original.value;
          
          return (
            <div className="text-sm text-gray-700 truncate max-w-xs">
              {displayValue ? String(displayValue).substring(0, 100) : "N/A"}
            </div>
          );
        },
      },
      {
        accessorKey: "confidence",
        header: "Confidence",
        cell: ({ getValue }) => {
          const confidence = getValue() as number;
          const percentage = Math.round(confidence * 100);
          const color =
            percentage >= 80
              ? "bg-green-100 text-green-800"
              : percentage >= 60
              ? "bg-yellow-100 text-yellow-800"
              : "bg-red-100 text-red-800";
          
          return (
            <span
              className={`px-2 py-1 rounded-full text-xs font-medium ${color}`}
              aria-label={`Confidence score: ${percentage}%`}
            >
              {percentage}%
            </span>
          );
        },
      },
      {
        id: "actions",
        header: "Actions",
        cell: ({ row }) => (
          <div className="flex gap-1">
            {editingField === row.original.name ? (
              <button
                onClick={() => setEditingField(null)}
                className="px-2 py-1 text-xs bg-gray-300 hover:bg-gray-400 rounded"
                aria-label={`Cancel editing ${row.original.name}`}
              >
                Cancel
              </button>
            ) : (
              <>
                <button
                  onClick={() => {
                    setModifications((prev) => {
                      const newMods = { ...prev };
                      delete newMods[row.original.name];
                      return newMods;
                    });
                    // Approve just this field
                  }}
                  className="px-2 py-1 text-xs bg-green-600 hover:bg-green-700 text-white rounded"
                  aria-label={`Approve ${row.original.name}`}
                >
                  Approve
                </button>
                <button
                  onClick={() => setEditingField(row.original.name)}
                  className="px-2 py-1 text-xs bg-blue-600 hover:bg-blue-700 text-white rounded"
                  aria-label={`Edit ${row.original.name}`}
                >
                  Edit
                </button>
                <button
                  onClick={() => {
                    // Mark for rejection
                  }}
                  className="px-2 py-1 text-xs bg-red-600 hover:bg-red-700 text-white rounded"
                  aria-label={`Reject ${row.original.name}`}
                >
                  Reject
                </button>
              </>
            )}
          </div>
        ),
      },
    ],
    [editingField, selectedRows, modifications]
  );
  
  // ==================== Table Instance ====================
  const table = useReactTable({
    data: fields,
    columns,
    getCoreRowModel: getCoreRowModel(),
  });
  
  // ==================== Action Handlers ====================
  const handleApproveAll = async () => {
    try {
      await approveExtraction(extraction.id, {
        approval_comment: feedbackReason,
      });
      removeListItem(extraction.id);
      onBack();
    } catch (error) {
      console.error("Approval failed:", error);
    }
  };
  
  const handleModifyAndApprove = async () => {
    try {
      await modifyExtraction(extraction.id, {
        modifications,
        approval_comment: feedbackReason,
      });
      removeListItem(extraction.id);
      onBack();
    } catch (error) {
      console.error("Modification failed:", error);
    }
  };
  
  const handleRejectAll = async () => {
    if (!feedbackReason.trim()) {
      alert("Please provide a rejection reason");
      return;
    }
    
    try {
      await rejectExtraction(extraction.id, {
        reason: feedbackReason,
      });
      removeListItem(extraction.id);
      onBack();
    } catch (error) {
      console.error("Rejection failed:", error);
    }
  };
  
  // ==================== Render ====================
  return (
    <div className="flex flex-col h-screen bg-gray-50">
      {/* Header */}
      <div className="bg-white border-b border-gray-200 px-4 py-3 flex items-center justify-between">
        <div>
          <button
            onClick={onBack}
            className="text-blue-600 hover:text-blue-800 text-sm"
            aria-label="Back to list"
          >
            ← Back to List
          </button>
          <h1 className="text-lg font-bold text-gray-900 mt-1">
            Review Extraction: {extraction.extraction_schema_id}
          </h1>
        </div>
      </div>
      
      {/* Main Content: Split View */}
      <div className="flex flex-1 overflow-hidden">
        {/* Left: Document Preview */}
        <div className="w-1/3 border-r border-gray-200 overflow-y-auto bg-white p-4">
          <div className="space-y-4">
            <div>
              <h2 className="text-sm font-semibold text-gray-900">
                {documentPreview.title || "Untitled Document"}
              </h2>
              <p className="text-xs text-gray-500">Type: {documentPreview.type}</p>
            </div>
            
            <div className="bg-gray-50 p-3 rounded-md border border-gray-200 max-h-96 overflow-y-auto">
              <p className="text-sm text-gray-700 whitespace-pre-wrap">
                {documentPreview.content_text}
              </p>
            </div>
          </div>
        </div>
        
        {/* Right: Extraction Fields Table */}
        <div className="w-2/3 overflow-y-auto p-4">
          {editingField ? (
            <div className="bg-white p-4 rounded-md border border-gray-200 mb-4">
              <FieldEditor
                fieldName={editingField}
                fieldType={
                  fields.find((f) => f.name === editingField)?.type || "string"
                }
                currentValue={
                  modifications[editingField] !== undefined
                    ? modifications[editingField]
                    : fields.find((f) => f.name === editingField)?.value
                }
                onSave={(newValue) => {
                  setModifications((prev) => ({
                    ...prev,
                    [editingField]: newValue,
                  }));
                  setEditingField(null);
                }}
                onCancel={() => setEditingField(null)}
              />
            </div>
          ) : null}
          
          {/* Fields Table */}
          <div className="bg-white rounded-md border border-gray-200 overflow-x-auto mb-4">
            <table className="w-full text-sm">
              <thead className="bg-gray-50 border-b border-gray-200">
                {table.getHeaderGroups().map((headerGroup) => (
                  <tr key={headerGroup.id}>
                    {headerGroup.headers.map((header) => (
                      <th
                        key={header.id}
                        className="px-4 py-2 text-left font-medium text-gray-900"
                      >
                        {header.isPlaceholder
                          ? null
                          : flexRender(
                              header.column.columnDef.header,
                              header.getContext()
                            )}
                      </th>
                    ))}
                  </tr>
                ))}
              </thead>
              <tbody>
                {table.getRowModel().rows.map((row) => (
                  <tr key={row.id} className="border-b border-gray-200 hover:bg-gray-50">
                    {row.getVisibleCells().map((cell) => (
                      <td key={cell.id} className="px-4 py-2">
                        {flexRender(
                          cell.column.columnDef.cell,
                          cell.getContext()
                        )}
                      </td>
                    ))}
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
          
          {/* Feedback & Actions */}
          <div className="bg-white p-4 rounded-md border border-gray-200">
            <label className="block text-sm font-medium text-gray-900 mb-2">
              Review Comment (Required for Rejection)
            </label>
            <textarea
              value={feedbackReason}
              onChange={(e) => setFeedbackReason(e.target.value)}
              rows={3}
              placeholder="Enter your feedback or rejection reason..."
              className="w-full px-3 py-2 border border-gray-300 rounded-md text-sm focus:outline-none focus:ring-blue-500"
              aria-label="Review feedback"
            />
            
            <div className="flex gap-2 mt-4">
              <button
                onClick={handleApproveAll}
                disabled={isLoading}
                className="flex-1 px-4 py-2 bg-green-600 hover:bg-green-700 text-white rounded-md font-medium disabled:bg-gray-400"
                aria-label="Approve all fields"
              >
                Approve All
              </button>
              <button
                onClick={handleModifyAndApprove}
                disabled={isLoading || Object.keys(modifications).length === 0}
                className="flex-1 px-4 py-2 bg-blue-600 hover:bg-blue-700 text-white rounded-md font-medium disabled:bg-gray-400"
                aria-label="Save modifications and approve"
              >
                Save & Approve
              </button>
              <button
                onClick={handleRejectAll}
                disabled={isLoading}
                className="flex-1 px-4 py-2 bg-red-600 hover:bg-red-700 text-white rounded-md font-medium disabled:bg-gray-400"
                aria-label="Reject all fields"
              >
                Reject
              </button>
              <button
                onClick={onBack}
                disabled={isLoading}
                className="px-4 py-2 bg-gray-300 hover:bg-gray-400 rounded-md font-medium disabled:bg-gray-400"
                aria-label="Cancel and go back"
              >
                Cancel
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
};
```

### 4-6. ExtractionReviewPage 컴포넌트

**파일:** `/frontend/src/pages/ExtractionReviewPage.tsx`

```typescript
/**
 * 추출 결과 검토 페이지 (메인).
 * 
 * - Pending 추출 결과 목록
 * - 상태 필터 탭
 * - 각 항목 클릭 시 상세 검토 화면으로 전환
 */

import React, { useEffect, useState } from "react";
import { ExtractionStatus } from "@/types/extraction";
import { useExtractionStore } from "@/stores/extractionStore";
import { ExtractionReviewDetail } from "@/components/extraction/ExtractionReviewDetail";

export const ExtractionReviewPage: React.FC = () => {
  const [showDetail, setShowDetail] = useState(false);
  const [documentPreview, setDocumentPreview] = useState<any>(null);
  
  const {
    filteredList,
    selectedExtraction,
    currentStatus,
    currentOffset,
    currentLimit,
    totalCount,
    isLoading,
    error,
    fetchPendingList,
    selectExtraction,
    setCurrentStatus,
    nextPage,
    prevPage,
  } = useExtractionStore();
  
  // ==================== Initial Load ====================
  useEffect(() => {
    fetchPendingList(0, 50, ExtractionStatus.PENDING);
  }, [fetchPendingList]);
  
  // ==================== Load Document Preview ====================
  useEffect(() => {
    if (selectedExtraction) {
      // TODO: Fetch document preview from API
      setDocumentPreview({
        title: `Document ${selectedExtraction.document_id.substring(0, 8)}`,
        type: selectedExtraction.extraction_schema_id,
        content_text: "Mock document content...",
      });
      setShowDetail(true);
    }
  }, [selectedExtraction]);
  
  // ==================== Handlers ====================
  const handleStatusChange = (status: ExtractionStatus) => {
    setCurrentStatus(status);
  };
  
  const handleSelectExtraction = (id: string) => {
    const extraction = filteredList.find((item) => item.id === id);
    if (extraction) {
      selectExtraction(extraction);
    }
  };
  
  const handleBack = () => {
    setShowDetail(false);
    setDocumentPreview(null);
  };
  
  // ==================== Detail View ====================
  if (showDetail && selectedExtraction && documentPreview) {
    return (
      <ExtractionReviewDetail
        extraction={selectedExtraction}
        documentPreview={documentPreview}
        onBack={handleBack}
      />
    );
  }
  
  // ==================== List View ====================
  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header */}
      <div className="bg-white border-b border-gray-200 px-6 py-4">
        <h1 className="text-2xl font-bold text-gray-900">
          Extraction Review
        </h1>
        <p className="text-sm text-gray-600 mt-1">
          Review and approve LLM-assisted extraction results
        </p>
      </div>
      
      {/* Content */}
      <div className="max-w-7xl mx-auto px-6 py-8">
        {/* Status Tabs */}
        <div className="mb-6 border-b border-gray-200">
          <div className="flex gap-4">
            {Object.values(ExtractionStatus).map((status) => (
              <button
                key={status}
                onClick={() => handleStatusChange(status)}
                className={`px-4 py-2 font-medium border-b-2 transition-colors ${
                  currentStatus === status
                    ? "border-blue-600 text-blue-600"
                    : "border-transparent text-gray-600 hover:text-gray-900"
                }`}
                aria-current={currentStatus === status}
              >
                {status.charAt(0).toUpperCase() + status.slice(1)}
              </button>
            ))}
          </div>
        </div>
        
        {/* Error Message */}
        {error && (
          <div className="mb-4 p-4 bg-red-50 border border-red-200 rounded-md">
            <p className="text-sm text-red-800">{error}</p>
          </div>
        )}
        
        {/* List */}
        {isLoading ? (
          <div className="text-center py-8">
            <p className="text-gray-600">Loading extractions...</p>
          </div>
        ) : filteredList.length === 0 ? (
          <div className="text-center py-8">
            <p className="text-gray-600">No extractions to review</p>
          </div>
        ) : (
          <div className="space-y-3">
            {filteredList.map((extraction) => (
              <div
                key={extraction.id}
                onClick={() => handleSelectExtraction(extraction.id)}
                className="bg-white p-4 rounded-md border border-gray-200 hover:border-blue-400 cursor-pointer transition-colors"
                role="button"
                tabIndex={0}
                aria-label={`Review extraction for document ${extraction.document_id.substring(0, 8)}`}
              >
                <div className="flex justify-between items-start">
                  <div>
                    <h3 className="font-medium text-gray-900">
                      {extraction.extraction_schema_id}
                    </h3>
                    <p className="text-sm text-gray-600">
                      Document: {extraction.document_id.substring(0, 8)}...
                    </p>
                    <p className="text-xs text-gray-500 mt-1">
                      Created: {new Date(extraction.created_at).toLocaleString()}
                    </p>
                  </div>
                  
                  {/* Confidence Badge */}
                  <div className="text-right">
                    <div className="text-2xl font-bold text-gray-900">
                      {Math.round(
                        (extraction.confidence_scores.reduce((acc, cs) => acc + cs.confidence, 0) /
                          extraction.confidence_scores.length) *
                          100
                      )}
                      %
                    </div>
                    <p className="text-xs text-gray-600">Avg Confidence</p>
                  </div>
                </div>
              </div>
            ))}
          </div>
        )}
        
        {/* Pagination */}
        <div className="mt-8 flex justify-between items-center">
          <p className="text-sm text-gray-600">
            Showing {currentOffset + 1}–
            {Math.min(currentOffset + currentLimit, totalCount)} of {totalCount}
          </p>
          
          <div className="flex gap-2">
            <button
              onClick={() => prevPage()}
              disabled={currentOffset === 0}
              className="px-4 py-2 border border-gray-300 rounded-md hover:bg-gray-50 disabled:opacity-50"
              aria-label="Previous page"
            >
              Previous
            </button>
            <button
              onClick={() => nextPage()}
              disabled={currentOffset + currentLimit >= totalCount}
              className="px-4 py-2 border border-gray-300 rounded-md hover:bg-gray-50 disabled:opacity-50"
              aria-label="Next page"
            >
              Next
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};
```

### 4-7. 스타일시트 (TailwindCSS)

모든 컴포넌트는 TailwindCSS로 스타일되어 있으며, 추가 파일이 불필요하다.

### 4-8. UI 리뷰 체크리스트

**Iteration 1: 기본 레이아웃 + 필드 테이블**
- [ ] ExtractionReviewPage 목록 화면 렌더링
- [ ] 필드 테이블이 정확히 표시되는가
- [ ] 페이지네이션이 작동하는가

**Iteration 2: 필드 에디터 + 인라인 편집**
- [ ] FieldEditor가 모든 필드 타입 지원하는가
- [ ] "Edit" 버튼 클릭 시 에디터가 나타나는가
- [ ] 수정 후 "Save"하면 테이블이 업데이트되는가

**Iteration 3: 분할 화면 문서 미리보기**
- [ ] 왼쪽 문서 미리보기가 50% 너비로 표시되는가
- [ ] 문서 텍스트가 스크롤 가능한가
- [ ] 오른쪽 테이블이 50% 너비로 표시되는가

**Iteration 4: 상태 필터 + 페이지네이션**
- [ ] Status tabs이 정확하게 작동하는가
- [ ] 각 탭에서 올바른 상태의 항목만 표시되는가
- [ ] Next/Prev 버튼이 정확히 작동하는가

**Iteration 5: 스타일링 + 접근성 + 반응형**
- [ ] 색상 대비가 WCAG AA 기준을 만족하는가
- [ ] 키보드 네비게이션이 가능한가 (Tab, Enter, Escape)
- [ ] 모바일 (320px)에서 단일 열로 나타나는가
- [ ] 태블릿 (768px)에서 스택 레이아웃으로 나타나는가

## 5. 산출물

1. **타입 정의**
   - 파일: `/frontend/src/types/extraction.ts`
   - 크기: ~4KB

2. **Zustand 스토어**
   - 파일: `/frontend/src/stores/extractionStore.ts`
   - 크기: ~6KB

3. **React hooks**
   - 파일: `/frontend/src/hooks/useExtractionActions.ts`
   - 크기: ~8KB

4. **FieldEditor 컴포넌트**
   - 파일: `/frontend/src/components/extraction/FieldEditor.tsx`
   - 크기: ~12KB

5. **ExtractionReviewDetail 컴포넌트**
   - 파일: `/frontend/src/components/extraction/ExtractionReviewDetail.tsx`
   - 크기: ~15KB

6. **ExtractionReviewPage 컴포넌트**
   - 파일: `/frontend/src/pages/ExtractionReviewPage.tsx`
   - 크기: ~10KB

## 6. 완료 기준

1. ExtractionReviewPage가 pending 추출 결과 목록을 페이지네이션으로 표시한다.
2. 각 항목을 클릭하면 ExtractionReviewDetail로 전환된다.
3. 분할 화면에서 왼쪽 문서 미리보기와 오른쪽 필드 테이블이 나란히 표시된다.
4. 필드별 "승인", "수정", "거절" 버튼이 작동한다.
5. "모두 승인", "모두 거절" 버튼이 일괄 작업을 수행한다.
6. 상태 필터 탭이 정확하게 작동한다.
7. FieldEditor가 모든 필드 타입 (string, date, integer, array, object)을 지원한다.
8. 접근성 (WCAG 2.1 AA): 키보드 네비게이션, ARIA labels, 포커스 인디케이터가 모두 작동한다.
9. 반응형 디자인: Desktop (1920px), Tablet (768px), Mobile (320px) 모두에서 정상 작동한다.
10. UI 리뷰를 5회 이상 수행하였고, 각 iteration의 체크리스트를 만족한다.

## 7. 작업 지침

### 지침 7-1. 컴포넌트 분리
ExtractionReviewPage는 목록 화면만, ExtractionReviewDetail은 상세 검토 화면만 담당하도록 명확히 분리하라.

### 지침 7-2. 상태 관리
Zustand로 전역 상태를 관리하되, 컴포넌트 로컬 상태 (editingField, modifications 등)는 컴포넌트 내에서만 관리하라.

### 지침 7-3. 접근성 우선
모든 interactive 요소에 aria-label, aria-required, role을 명시하고, 키보드 네비게이션을 지원하라.

### 지침 7-4. 반응형 디자인
Tailwind responsive classes (sm:, md:, lg:)를 사용하여 Mobile-First 방식으로 설계하라.

### 지침 7-5. API 통합
useExtractionActions hook으로 모든 API 호출을 관리하고, 에러 처리와 로딩 상태를 명확하게 표시하라.

### 지침 7-6. UI 리뷰 문서화
각 iteration마다 스크린샷과 함께 피드백을 기록하고, 최종 버전에 포함시키라.

### 지침 7-7. 테스트
React Testing Library로 컴포넌트 렌더링, 사용자 인터랙션, API 호출을 모두 테스트하라.

---

End of Task 8-6
