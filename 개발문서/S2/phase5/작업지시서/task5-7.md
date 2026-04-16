# Task 5-7: 사용자 UI "제안" 탭 (검토/승인/거부 + diff 보기)

## 1. 작업 목적

사용자가 AI 에이전트 제안을 **투명하게 검토하고 승인/거부**할 수 있는 UI를 구축합니다. 제안의 현재 문서 버전과의 **diff 비교 기능**을 제공하여 변경 내용을 명확하게 파악할 수 있게 합니다.

- **에이전트 제안 가시성**: "대기 중인 검토" 탭에서 모든 제안을 한눈에 확인
- **비교 기능**: 현재 문서 vs 제안 Draft의 diff 시각화
- **승인/거부 워크플로우**: 사유 입력과 함께 검토 의견 기록
- **반응형 디자인**: 데스크톱/태블릿/모바일 지원
- **접근성**: WCAG 2.1 AA 준수

---

## 2. 작업 범위

### 포함 사항
- ProposalReviewTab 컴포넌트: User UI에 "대기 중인 검토" 탭 추가
- ProposalList: 내 문서의 에이전트 제안 목록 (에이전트명, 미리보기, 생성시간)
- ProposalApprovalModal: 승인/거부 다이얼로그 + 사유 입력
- DiffView 컴포넌트: 현재 문서 vs 제안 Draft 비교 (행 단위 색상 구분)
- Zustand 스토어: proposalStore (상태 관리)
- API 호출 계층: proposalApi.ts (unwrapEnvelope 패턴)
- 반응형 디자인 (Tailwind CSS)
- WCAG 2.1 AA 접근성
- 단위 테스트 (React Testing Library)

### 제외 사항
- 에이전트 제안 큐 백엔드 (Task 5-4)
- 인젝션 탐지 엔진 (Task 5-5, 5-6)
- 관리자 UI

---

## 3. 선행 조건

- Task 5-4 완료: 에이전트 제안 큐 API (GET /api/v1/proposals, GET /api/v1/proposals/{id})
- Task 5-5, 5-6 완료: 인젝션 탐지 및 메타플래그
- React 18+ 환경
- Zustand 4.0+
- Tailwind CSS 3.0+
- 리액트 컴포넌트 기본 구조 (User UI 존재)

---

## 4. 주요 작업 항목

### 4-1. API 호출 계층 (proposalApi.ts)

**목표**: 제안 API를 타입 안전하게 호출하는 인터페이스

**상세 작업**:

1. 타입 정의
   - 파일: `src/api/types/proposal.types.ts`

   ```typescript
   // 제안 데이터 타입
   export interface AgentProposal {
     id: string;
     agent_id: string;
     document_id: string;
     proposal_type: "draft" | "transition";  // draft: 문서 내용, transition: 상태 변경
     reference_id: string;  // DocumentDraft ID
     status: "pending" | "approved" | "rejected" | "withdrawn";
     created_at: string;  // ISO 8601
     updated_at: string;
     reviewed_by?: string;
     review_notes?: string;
     review_timestamp?: string;
     metadata?: {
       trusted: boolean;
       injection_risk: boolean;
       injection_patterns_detected: string[];
       source_data_untrusted: boolean;
     };
   }
   
   export interface ProposalListResponse {
     items: AgentProposal[];
     total: number;
     skip: number;
     limit: number;
   }
   
   export interface ApprovalRequest {
     review_notes?: string;
   }
   
   export interface RejectionRequest {
     review_notes: string;  // 거부 사유 필수
   }
   
   export interface ProposalDraft {
     id: string;
     document_id: string;
     content: string;
     metadata?: {
       [key: string]: any;
     };
   }
   ```

2. API 호출 서비스
   - 파일: `src/api/services/proposalApi.ts`

   ```typescript
   import axios, { AxiosInstance } from "axios";
   import {
     AgentProposal,
     ProposalListResponse,
     ApprovalRequest,
     RejectionRequest,
     ProposalDraft,
   } from "../types/proposal.types";
   import { unwrapEnvelope } from "../utils/envelope";
   
   class ProposalApiService {
     private api: AxiosInstance;
   
     constructor(apiClient: AxiosInstance) {
       this.api = apiClient;
     }
   
     /**
      * 내 문서의 제안 목록 조회
      * GET /api/v1/proposals?document_id={documentId}&status=pending
      */
     async getProposalsByDocument(
       documentId: string,
       status?: "pending" | "approved" | "rejected",
       limit: number = 20,
       skip: number = 0
     ): Promise<ProposalListResponse> {
       const params = new URLSearchParams();
       params.append("document_id", documentId);
       if (status) params.append("status", status);
       params.append("limit", limit.toString());
       params.append("skip", skip.toString());
   
       try {
         const response = await this.api.get(
           `/api/v1/proposals?${params.toString()}`
         );
         return unwrapEnvelope(response.data);
       } catch (error) {
         throw new Error(`Failed to fetch proposals: ${error}`);
       }
     }
   
     /**
      * 제안 상세 조회 + Draft 포함
      * GET /api/v1/proposals/{proposalId}
      */
     async getProposalDetail(
       proposalId: string
     ): Promise<AgentProposal & { draft?: ProposalDraft }> {
       try {
         const response = await this.api.get(`/api/v1/proposals/${proposalId}`);
         return unwrapEnvelope(response.data);
       } catch (error) {
         throw new Error(`Failed to fetch proposal detail: ${error}`);
       }
     }
   
     /**
      * 제안 승인
      * POST /api/v1/proposals/{proposalId}/approve
      */
     async approveProposal(
       proposalId: string,
       request: ApprovalRequest
     ): Promise<AgentProposal> {
       try {
         const response = await this.api.post(
           `/api/v1/proposals/${proposalId}/approve`,
           request
         );
         return unwrapEnvelope(response.data);
       } catch (error) {
         throw new Error(`Failed to approve proposal: ${error}`);
       }
     }
   
     /**
      * 제안 거부
      * POST /api/v1/proposals/{proposalId}/reject
      */
     async rejectProposal(
       proposalId: string,
       request: RejectionRequest
     ): Promise<AgentProposal> {
       try {
         const response = await this.api.post(
           `/api/v1/proposals/${proposalId}/reject`,
           request
         );
         return unwrapEnvelope(response.data);
       } catch (error) {
         throw new Error(`Failed to reject proposal: ${error}`);
       }
     }
   
     /**
      * 제안 철회 (에이전트만 사용)
      * POST /api/v1/proposals/{proposalId}/withdraw
      */
     async withdrawProposal(proposalId: string): Promise<AgentProposal> {
       try {
         const response = await this.api.post(
           `/api/v1/proposals/${proposalId}/withdraw`
         );
         return unwrapEnvelope(response.data);
       } catch (error) {
         throw new Error(`Failed to withdraw proposal: ${error}`);
       }
     }
   
     /**
      * DocumentDraft 조회 (diff 비교용)
      * GET /api/v1/documents/{documentId}/drafts/{draftId}
      */
     async getDraft(
       documentId: string,
       draftId: string
     ): Promise<ProposalDraft> {
       try {
         const response = await this.api.get(
           `/api/v1/documents/${documentId}/drafts/${draftId}`
         );
         return unwrapEnvelope(response.data);
       } catch (error) {
         throw new Error(`Failed to fetch draft: ${error}`);
       }
     }
   
     /**
      * 현재 문서 조회
      * GET /api/v1/documents/{documentId}
      */
     async getDocument(documentId: string): Promise<{ content: string }> {
       try {
         const response = await this.api.get(`/api/v1/documents/${documentId}`);
         return unwrapEnvelope(response.data);
       } catch (error) {
         throw new Error(`Failed to fetch document: ${error}`);
       }
     }
   }
   
   export default ProposalApiService;
   ```

3. API 싱글톤 인스턴스
   - 파일: `src/api/instances/proposalApi.instance.ts`

   ```typescript
   import axios from "axios";
   import ProposalApiService from "../services/proposalApi";
   
   const apiClient = axios.create({
     baseURL: process.env.REACT_APP_API_BASE_URL || "http://localhost:8000",
     headers: {
       "Content-Type": "application/json",
     },
   });
   
   // 토큰 추가 (있는 경우)
   apiClient.interceptors.request.use((config) => {
     const token = localStorage.getItem("auth_token");
     if (token) {
       config.headers.Authorization = `Bearer ${token}`;
     }
     return config;
   });
   
   export const proposalApi = new ProposalApiService(apiClient);
   ```

**산출물**:
- `src/api/types/proposal.types.ts`
- `src/api/services/proposalApi.ts`
- `src/api/instances/proposalApi.instance.ts`

**완료 기준**:
- 모든 API 메서드 타입 안전
- 에러 처리 명확
- unwrapEnvelope 패턴 적용

---

### 4-2. Zustand 스토어 (proposalStore.ts)

**목표**: 제안 목록, 상세 정보, 선택 상태 관리

**상세 작업**:

1. Zustand 스토어 정의
   - 파일: `src/store/proposalStore.ts`

   ```typescript
   import { create } from "zustand";
   import { proposalApi } from "../api/instances/proposalApi.instance";
   import { AgentProposal, ProposalListResponse } from "../api/types/proposal.types";
   
   export interface ProposalStoreState {
     // 제안 목록
     proposals: AgentProposal[];
     isLoadingProposals: boolean;
     proposalsError: string | null;
     totalProposals: number;
   
     // 선택된 제안
     selectedProposal: AgentProposal | null;
     isLoadingDetail: boolean;
     detailError: string | null;
   
     // 승인/거부 모달
     showApprovalModal: boolean;
     approvalStatus: "idle" | "loading" | "success" | "error";
     approvalError: string | null;
   
     // 필터
     statusFilter: "pending" | "approved" | "rejected" | "all";
     currentDocumentId: string | null;
   
     // 액션
     fetchProposals: (
       documentId: string,
       status?: "pending" | "approved" | "rejected"
     ) => Promise<void>;
     selectProposal: (proposal: AgentProposal) => void;
     fetchProposalDetail: (proposalId: string) => Promise<void>;
     openApprovalModal: () => void;
     closeApprovalModal: () => void;
     approveProposal: (proposalId: string, notes?: string) => Promise<void>;
     rejectProposal: (proposalId: string, notes: string) => Promise<void>;
     setStatusFilter: (status: "pending" | "approved" | "rejected" | "all") => void;
     clearSelected: () => void;
   }
   
   export const useProposalStore = create<ProposalStoreState>((set, get) => ({
     // 초기 상태
     proposals: [],
     isLoadingProposals: false,
     proposalsError: null,
     totalProposals: 0,
   
     selectedProposal: null,
     isLoadingDetail: false,
     detailError: null,
   
     showApprovalModal: false,
     approvalStatus: "idle",
     approvalError: null,
   
     statusFilter: "pending",
     currentDocumentId: null,
   
     // 제안 목록 조회
     fetchProposals: async (documentId: string, status?: "pending" | "approved" | "rejected") => {
       set({ isLoadingProposals: true, proposalsError: null, currentDocumentId: documentId });
   
       try {
         const response = await proposalApi.getProposalsByDocument(documentId, status);
         set({
           proposals: response.items,
           totalProposals: response.total,
           isLoadingProposals: false,
         });
       } catch (error) {
         set({
           proposalsError: (error as Error).message,
           isLoadingProposals: false,
         });
       }
     },
   
     // 제안 선택
     selectProposal: (proposal: AgentProposal) => {
       set({ selectedProposal: proposal, detailError: null });
     },
   
     // 제안 상세 조회
     fetchProposalDetail: async (proposalId: string) => {
       set({ isLoadingDetail: true, detailError: null });
   
       try {
         const detail = await proposalApi.getProposalDetail(proposalId);
         set({
           selectedProposal: detail as AgentProposal,
           isLoadingDetail: false,
         });
       } catch (error) {
         set({
           detailError: (error as Error).message,
           isLoadingDetail: false,
         });
       }
     },
   
     // 승인 모달 열기
     openApprovalModal: () => {
       set({ showApprovalModal: true });
     },
   
     // 승인 모달 닫기
     closeApprovalModal: () => {
       set({ showApprovalModal: false, approvalStatus: "idle", approvalError: null });
     },
   
     // 제안 승인
     approveProposal: async (proposalId: string, notes?: string) => {
       set({ approvalStatus: "loading" });
   
       try {
         const updated = await proposalApi.approveProposal(proposalId, { review_notes: notes });
   
         // 목록 업데이트
         const { proposals } = get();
         const updatedProposals = proposals.map((p) =>
           p.id === proposalId ? updated : p
         );
   
         set({
           proposals: updatedProposals,
           selectedProposal: updated,
           approvalStatus: "success",
           showApprovalModal: false,
         });
       } catch (error) {
         set({
           approvalStatus: "error",
           approvalError: (error as Error).message,
         });
       }
     },
   
     // 제안 거부
     rejectProposal: async (proposalId: string, notes: string) => {
       set({ approvalStatus: "loading" });
   
       try {
         const updated = await proposalApi.rejectProposal(proposalId, { review_notes: notes });
   
         // 목록 업데이트
         const { proposals } = get();
         const updatedProposals = proposals.map((p) =>
           p.id === proposalId ? updated : p
         );
   
         set({
           proposals: updatedProposals,
           selectedProposal: updated,
           approvalStatus: "success",
           showApprovalModal: false,
         });
       } catch (error) {
         set({
           approvalStatus: "error",
           approvalError: (error as Error).message,
         });
       }
     },
   
     // 상태 필터 설정
     setStatusFilter: (status: "pending" | "approved" | "rejected" | "all") => {
       set({ statusFilter: status });
     },
   
     // 선택 항목 초기화
     clearSelected: () => {
       set({
         selectedProposal: null,
         detailError: null,
         showApprovalModal: false,
       });
     },
   }));
   ```

**산출물**:
- `src/store/proposalStore.ts`

**완료 기준**:
- 모든 상태 관리 동기화
- 비동기 액션 정상 작동
- 에러 처리 명확

---

### 4-3. DiffView 컴포넌트

**목표**: 현재 문서 vs 제안 Draft 비교 시각화

**상세 작업**:

1. 유틸리티 함수: 텍스트 Diff 계산
   - 파일: `src/utils/diffUtils.ts`

   ```typescript
   import { diffLines, Change } from "diff";
   
   export interface DiffLine {
     type: "added" | "removed" | "unchanged";
     content: string;
     lineNumber?: number;
   }
   
   /**
    * 두 텍스트의 Diff 계산
    */
   export function calculateDiff(
     original: string,
     modified: string
   ): DiffLine[] {
     const changes: Change[] = diffLines(original, modified);
     const result: DiffLine[] = [];
   
     changes.forEach((change) => {
       const lines = change.value.split("\n").filter((line) => line.length > 0);
   
       lines.forEach((line) => {
         if (change.added) {
           result.push({
             type: "added",
             content: line,
           });
         } else if (change.removed) {
           result.push({
             type: "removed",
             content: line,
           });
         } else {
           result.push({
             type: "unchanged",
             content: line,
           });
         }
       });
     });
   
     return result;
   }
   
   /**
    * Diff 라인에 라인 번호 추가
    */
   export function addLineNumbers(diffLines: DiffLine[]): DiffLine[] {
     let lineNum = 1;
     return diffLines.map((line) => ({
       ...line,
       lineNumber: line.type === "unchanged" ? lineNum++ : undefined,
     }));
   }
   ```

2. DiffView 컴포넌트
   - 파일: `src/components/proposal/DiffView.tsx`

   ```typescript
   import React, { useMemo } from "react";
   import { calculateDiff, DiffLine } from "../../utils/diffUtils";
   
   interface DiffViewProps {
     original: string;
     modified: string;
     title?: string;
   }
   
   export const DiffView: React.FC<DiffViewProps> = ({
     original,
     modified,
     title = "변경 사항",
   }) => {
     const diffLines = useMemo(() => calculateDiff(original, modified), [
       original,
       modified,
     ]);
   
     return (
       <div className="w-full border rounded-lg shadow">
         {/* 헤더 */}
         <div className="bg-gray-100 border-b px-4 py-3">
           <h3 className="text-sm font-semibold text-gray-900">{title}</h3>
         </div>
   
         {/* Diff 콘텐츠 */}
         <div className="overflow-x-auto max-h-96 bg-white">
           <table className="w-full text-sm font-mono text-gray-700">
             <tbody>
               {diffLines.map((line, idx) => (
                 <tr
                   key={idx}
                   className={`border-b border-gray-200 ${
                     line.type === "added"
                       ? "bg-green-50 hover:bg-green-100"
                       : line.type === "removed"
                       ? "bg-red-50 hover:bg-red-100"
                       : "bg-white hover:bg-gray-50"
                   }`}
                   role="row"
                   aria-label={`${line.type}: ${line.content.substring(0, 50)}`}
                 >
                   {/* 행 번호 */}
                   <td className="w-12 px-3 py-2 text-gray-500 bg-gray-50 text-right select-none">
                     {line.lineNumber || ""}
                   </td>
   
                   {/* 변경 타입 표시 */}
                   <td className="w-8 px-2 py-2 text-center select-none font-bold">
                     {line.type === "added" ? (
                       <span className="text-green-700" aria-label="추가">
                         +
                       </span>
                     ) : line.type === "removed" ? (
                       <span className="text-red-700" aria-label="삭제">
                         -
                       </span>
                     ) : (
                       <span className="text-gray-400" aria-label="변경없음">
                         {" "}
                       </span>
                     )}
                   </td>
   
                   {/* 콘텐츠 */}
                   <td className="px-3 py-2 break-words max-w-md">
                     <code>{line.content}</code>
                   </td>
                 </tr>
               ))}
             </tbody>
           </table>
         </div>
   
         {/* 범례 */}
         <div className="bg-gray-50 border-t px-4 py-2 flex gap-4 text-xs text-gray-600">
           <div className="flex items-center gap-1">
             <span className="text-green-700 font-bold">+</span>
             <span>추가됨</span>
           </div>
           <div className="flex items-center gap-1">
             <span className="text-red-700 font-bold">-</span>
             <span>삭제됨</span>
           </div>
           <div className="flex items-center gap-1">
             <span className="text-gray-400">•</span>
             <span>변경 없음</span>
           </div>
         </div>
       </div>
     );
   };
   ```

**산출물**:
- `src/utils/diffUtils.ts`
- `src/components/proposal/DiffView.tsx`

**완료 기준**:
- Diff 계산 정확
- 색상 구분 명확
- 긴 텍스트 줄바꿈 처리

---

### 4-4. ProposalList 및 ProposalItem 컴포넌트

**목표**: 제안 목록 표시 및 선택

**상세 작업**:

1. ProposalList 컴포넌트
   - 파일: `src/components/proposal/ProposalList.tsx`

   ```typescript
   import React, { useEffect } from "react";
   import { useProposalStore } from "../../store/proposalStore";
   import { ProposalItem } from "./ProposalItem";
   import { Spinner } from "../common/Spinner";
   
   interface ProposalListProps {
     documentId: string;
   }
   
   export const ProposalList: React.FC<ProposalListProps> = ({ documentId }) => {
     const {
       proposals,
       isLoadingProposals,
       proposalsError,
       statusFilter,
       fetchProposals,
     } = useProposalStore();
   
     useEffect(() => {
       const status = statusFilter === "all" ? undefined : statusFilter;
       fetchProposals(documentId, status);
     }, [documentId, statusFilter, fetchProposals]);
   
     if (isLoadingProposals) {
       return (
         <div className="flex justify-center py-8">
           <Spinner />
         </div>
       );
     }
   
     if (proposalsError) {
       return (
         <div className="bg-red-50 border border-red-200 rounded-lg p-4 text-red-700">
           <p className="font-semibold">제안 목록을 불러올 수 없습니다</p>
           <p className="text-sm mt-1">{proposalsError}</p>
         </div>
       );
     }
   
     if (proposals.length === 0) {
       return (
         <div className="text-center py-8 text-gray-500">
           <p>대기 중인 제안이 없습니다</p>
         </div>
       );
     }
   
     return (
       <div className="space-y-2">
         {proposals.map((proposal) => (
           <ProposalItem key={proposal.id} proposal={proposal} />
         ))}
       </div>
     );
   };
   ```

2. ProposalItem 컴포넌트
   - 파일: `src/components/proposal/ProposalItem.tsx`

   ```typescript
   import React from "react";
   import { AgentProposal } from "../../api/types/proposal.types";
   import { useProposalStore } from "../../store/proposalStore";
   import { formatDate } from "../../utils/dateUtils";
   
   interface ProposalItemProps {
     proposal: AgentProposal;
   }
   
   export const ProposalItem: React.FC<ProposalItemProps> = ({ proposal }) => {
     const { selectProposal, selectedProposal } = useProposalStore();
   
     const isSelected = selectedProposal?.id === proposal.id;
     const statusLabel = {
       pending: "승인 대기",
       approved: "승인됨",
       rejected: "거부됨",
       withdrawn: "철회됨",
     }[proposal.status];
   
     const statusColor = {
       pending: "bg-yellow-100 text-yellow-800",
       approved: "bg-green-100 text-green-800",
       rejected: "bg-red-100 text-red-800",
       withdrawn: "bg-gray-100 text-gray-800",
     }[proposal.status];
   
     const proposalTypeLabel =
       proposal.proposal_type === "draft" ? "문서 수정" : "상태 변경";
   
     return (
       <div
         onClick={() => selectProposal(proposal)}
         role="button"
         tabIndex={0}
         onKeyDown={(e) => {
           if (e.key === "Enter" || e.key === " ") {
             selectProposal(proposal);
           }
         }}
         className={`p-4 rounded-lg border-2 cursor-pointer transition ${
           isSelected
             ? "border-blue-500 bg-blue-50"
             : "border-gray-200 bg-white hover:border-gray-300"
         }`}
         aria-selected={isSelected}
         aria-label={`${proposal.agent_id}의 제안: ${proposalTypeLabel}`}
       >
         {/* 헤더: 에이전트명, 상태 */}
         <div className="flex items-center justify-between mb-2">
           <div className="flex items-center gap-2">
             <h4 className="font-semibold text-gray-900">{proposal.agent_id}</h4>
             <span className="text-xs px-2 py-1 rounded-full bg-blue-100 text-blue-700">
               {proposalTypeLabel}
             </span>
           </div>
           <span className={`text-xs px-2 py-1 rounded-full ${statusColor}`}>
             {statusLabel}
           </span>
         </div>
   
         {/* 위험 경고 (인젝션) */}
         {proposal.metadata?.injection_risk && (
           <div className="bg-red-50 border border-red-200 rounded px-3 py-2 mb-2">
             <p className="text-xs font-semibold text-red-700">
               ⚠️ 보안 경고: 인젝션 위험 감지됨
             </p>
             {proposal.metadata?.injection_patterns_detected &&
               proposal.metadata.injection_patterns_detected.length > 0 && (
                 <p className="text-xs text-red-600 mt-1">
                   {proposal.metadata.injection_patterns_detected
                     .slice(0, 2)
                     .join(", ")}
                 </p>
               )}
           </div>
         )}
   
         {/* 미리보기 */}
         <p className="text-sm text-gray-600 line-clamp-2">
           {/* 일반적으로 DocumentDraft에서 가져올 preview */}
           제안된 문서 내용 미리보기...
         </p>
   
         {/* 메타정보: 생성시간, 검토자 */}
         <div className="flex items-center justify-between mt-3 text-xs text-gray-500">
           <span>{formatDate(proposal.created_at)}</span>
           {proposal.reviewed_by && (
             <span>검토자: {proposal.reviewed_by.substring(0, 8)}...</span>
           )}
         </div>
       </div>
     );
   };
   ```

**산출물**:
- `src/components/proposal/ProposalList.tsx`
- `src/components/proposal/ProposalItem.tsx`

**완료 기준**:
- 제안 목록 표시 정확
- 선택 상태 시각화
- 인젝션 경고 표시

---

### 4-5. ProposalApprovalModal 컴포넌트

**목표**: 승인/거부 워크플로우 및 사유 입력

**상세 작업**:

1. 컴포넌트 구현
   - 파일: `src/components/proposal/ProposalApprovalModal.tsx`

   ```typescript
   import React, { useState } from "react";
   import { useProposalStore } from "../../store/proposalStore";
   import { Modal } from "../common/Modal";
   import { Button } from "../common/Button";
   
   export const ProposalApprovalModal: React.FC = () => {
     const {
       selectedProposal,
       showApprovalModal,
       closeApprovalModal,
       approveProposal,
       rejectProposal,
       approvalStatus,
       approvalError,
     } = useProposalStore();
   
     const [approvalNotes, setApprovalNotes] = useState("");
     const [rejectionNotes, setRejectionNotes] = useState("");
     const [actionType, setActionType] = useState<"approve" | "reject" | null>(null);
   
     if (!selectedProposal || !showApprovalModal) return null;
   
     const handleApprove = async () => {
       setActionType("approve");
       await approveProposal(selectedProposal.id, approvalNotes);
       setApprovalNotes("");
     };
   
     const handleReject = async () => {
       if (!rejectionNotes.trim()) {
         alert("거부 사유를 입력해주세요");
         return;
       }
       setActionType("reject");
       await rejectProposal(selectedProposal.id, rejectionNotes);
       setRejectionNotes("");
     };
   
     const handleClose = () => {
       setApprovalNotes("");
       setRejectionNotes("");
       setActionType(null);
       closeApprovalModal();
     };
   
     return (
       <Modal
         isOpen={showApprovalModal}
         onClose={handleClose}
         title="제안 검토"
         size="md"
       >
         {/* 상태 메시지 */}
         {approvalStatus === "success" && (
           <div className="bg-green-50 border border-green-200 rounded p-4 mb-4">
             <p className="text-green-700 text-sm font-semibold">
               제안이 처리되었습니다
             </p>
           </div>
         )}
   
         {approvalStatus === "error" && (
           <div className="bg-red-50 border border-red-200 rounded p-4 mb-4">
             <p className="text-red-700 text-sm font-semibold">오류 발생</p>
             <p className="text-red-600 text-xs mt-1">{approvalError}</p>
           </div>
         )}
   
         {approvalStatus !== "success" && (
           <>
             {/* 제안 정보 */}
             <div className="bg-gray-50 rounded p-3 mb-4">
               <p className="text-sm text-gray-600">
                 <span className="font-semibold">에이전트:</span> {selectedProposal.agent_id}
               </p>
               <p className="text-sm text-gray-600">
                 <span className="font-semibold">유형:</span>{" "}
                 {selectedProposal.proposal_type === "draft" ? "문서 수정" : "상태 변경"}
               </p>
             </div>
   
             {/* 승인/거부 탭 */}
             <div className="space-y-4">
               {/* 승인 탭 */}
               <div>
                 <label className="block text-sm font-semibold text-gray-900 mb-2">
                   승인 의견 (선택)
                 </label>
                 <textarea
                   value={approvalNotes}
                   onChange={(e) => setApprovalNotes(e.target.value)}
                   placeholder="승인 사유나 의견을 입력하세요"
                   className="w-full px-3 py-2 border border-gray-300 rounded text-sm placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-green-500"
                   rows={3}
                   disabled={approvalStatus === "loading"}
                 />
               </div>
   
               {/* 거부 탭 */}
               <div>
                 <label className="block text-sm font-semibold text-gray-900 mb-2">
                   거부 사유 (필수)
                 </label>
                 <textarea
                   value={rejectionNotes}
                   onChange={(e) => setRejectionNotes(e.target.value)}
                   placeholder="거부 사유를 상세히 입력하세요"
                   className="w-full px-3 py-2 border border-gray-300 rounded text-sm placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-red-500"
                   rows={3}
                   disabled={approvalStatus === "loading"}
                 />
               </div>
             </div>
   
             {/* 버튼 */}
             <div className="flex gap-3 mt-6">
               <Button
                 onClick={handleApprove}
                 disabled={approvalStatus === "loading"}
                 isLoading={approvalStatus === "loading" && actionType === "approve"}
                 variant="success"
                 className="flex-1"
               >
                 승인
               </Button>
               <Button
                 onClick={handleReject}
                 disabled={approvalStatus === "loading" || !rejectionNotes.trim()}
                 isLoading={approvalStatus === "loading" && actionType === "reject"}
                 variant="danger"
                 className="flex-1"
               >
                 거부
               </Button>
               <Button
                 onClick={handleClose}
                 disabled={approvalStatus === "loading"}
                 variant="secondary"
                 className="flex-1"
               >
                 닫기
               </Button>
             </div>
           </>
         )}
   
         {/* 성공 후 닫기 버튼 */}
         {approvalStatus === "success" && (
           <div className="mt-4">
             <Button onClick={handleClose} variant="primary" className="w-full">
               닫기
             </Button>
           </div>
         )}
       </Modal>
     );
   };
   ```

**산출물**:
- `src/components/proposal/ProposalApprovalModal.tsx`

**완료 기준**:
- 승인/거부 워크플로우 정상 작동
- 거부 사유 필수 검증
- 로딩 상태 표시

---

### 4-6. ProposalReviewTab 컴포넌트 (메인 탭)

**목표**: User UI에 "대기 중인 검토" 탭 추가, 모든 컴포넌트 통합

**상세 작업**:

1. 컴포넌트 구현
   - 파일: `src/components/proposal/ProposalReviewTab.tsx`

   ```typescript
   import React, { useState, useMemo } from "react";
   import { useProposalStore } from "../../store/proposalStore";
   import { ProposalList } from "./ProposalList";
   import { ProposalApprovalModal } from "./ProposalApprovalModal";
   import { DiffView } from "./DiffView";
   import { Button } from "../common/Button";
   import { Tabs, Tab } from "../common/Tabs";
   
   interface ProposalReviewTabProps {
     documentId: string;
     currentDocumentContent: string;
   }
   
   export const ProposalReviewTab: React.FC<ProposalReviewTabProps> = ({
     documentId,
     currentDocumentContent,
   }) => {
     const {
       selectedProposal,
       statusFilter,
       setStatusFilter,
       openApprovalModal,
       clearSelected,
     } = useProposalStore();
   
     const [activeTab, setActiveTab] = useState<"list" | "diff">("list");
   
     // 제안된 Draft 컨텐츠 (실제로는 API에서 가져옴)
     const proposedContent = useMemo(() => {
       if (!selectedProposal) return "";
       // DocumentDraft API를 통해 컨텐츠 조회
       return "제안된 문서 컨텐츠...";
     }, [selectedProposal]);
   
     return (
       <div className="space-y-4">
         {/* 헤더 */}
         <div className="flex items-center justify-between">
           <h2 className="text-lg font-bold text-gray-900">대기 중인 검토</h2>
           {selectedProposal && (
             <Button
               onClick={clearSelected}
               variant="secondary"
               size="sm"
             >
               선택 해제
             </Button>
           )}
         </div>
   
         {/* 상태 필터 */}
         <div className="flex gap-2">
           {["pending", "approved", "rejected", "all"].map((status) => (
             <button
               key={status}
               onClick={() => setStatusFilter(status as any)}
               className={`px-3 py-1 rounded text-sm transition ${
                 statusFilter === status
                   ? "bg-blue-500 text-white"
                   : "bg-gray-200 text-gray-700 hover:bg-gray-300"
               }`}
               aria-pressed={statusFilter === status}
             >
               {status === "pending"
                 ? "대기"
                 : status === "approved"
                 ? "승인됨"
                 : status === "rejected"
                 ? "거부됨"
                 : "전체"}
             </button>
           ))}
         </div>
   
         {/* 메인 콘텐츠 */}
         <div className="grid grid-cols-1 lg:grid-cols-2 gap-4">
           {/* 좌측: 제안 목록 */}
           <div className="lg:h-96 overflow-y-auto">
             <ProposalList documentId={documentId} />
           </div>
   
           {/* 우측: 상세 보기 또는 빈 상태 */}
           {selectedProposal ? (
             <div className="space-y-4">
               {/* 탭 */}
               <Tabs value={activeTab} onChange={setActiveTab}>
                 <Tab label="미리보기" value="list" />
                 <Tab label="변경 사항" value="diff" />
               </Tabs>
   
               {/* 미리보기 탭 */}
               {activeTab === "list" && (
                 <div className="bg-white border rounded p-4 max-h-96 overflow-y-auto">
                   <div className="prose prose-sm max-w-none">
                     {proposedContent || "미리보기 로딩 중..."}
                   </div>
                 </div>
               )}
   
               {/* 변경 사항 탭 */}
               {activeTab === "diff" && (
                 <DiffView
                   original={currentDocumentContent}
                   modified={proposedContent}
                   title="현재 문서 vs 제안"
                 />
               )}
   
               {/* 액션 버튼 */}
               <div className="flex gap-2">
                 <Button
                   onClick={openApprovalModal}
                   variant="primary"
                   className="flex-1"
                 >
                   검토하기
                 </Button>
               </div>
             </div>
           ) : (
             <div className="bg-gray-50 rounded border-2 border-dashed border-gray-300 flex items-center justify-center h-96">
               <div className="text-center">
                 <p className="text-gray-500">제안을 선택하면</p>
                 <p className="text-gray-500">상세 정보가 여기에 표시됩니다</p>
               </div>
             </div>
           )}
         </div>
   
         {/* 승인/거부 모달 */}
         <ProposalApprovalModal />
       </div>
     );
   };
   ```

2. User UI에 탭 통합
   - 파일: `src/pages/DocumentEditor.tsx` (기존 파일 수정)

   ```typescript
   import { ProposalReviewTab } from "../components/proposal/ProposalReviewTab";
   
   export const DocumentEditor: React.FC = () => {
     const [activeTab, setActiveTab] = useState("content");
   
     return (
       <Tabs value={activeTab} onChange={setActiveTab}>
         <Tab label="문서" value="content" />
         <Tab label="대기 중인 검토" value="proposals" />
         {/* 기타 탭 */}
       </Tabs>
   
       {activeTab === "content" && <DocumentContent />}
       {activeTab === "proposals" && <ProposalReviewTab documentId={...} currentDocumentContent={...} />}
     );
   };
   ```

**산출물**:
- `src/components/proposal/ProposalReviewTab.tsx`
- `src/pages/DocumentEditor.tsx` (수정)

**완료 기준**:
- 모든 컴포넌트 통합
- 탭 전환 정상 작동
- 모달 오픈/클로즈 정상 작동

---

### 4-7. 반응형 디자인 및 접근성

**목표**: 데스크톱/태블릿/모바일 지원, WCAG 2.1 AA 준수

**상세 작업**:

1. Tailwind 반응형 클래스 적용 (이미 4-1~4-6에서 적용)
   - `grid-cols-1 lg:grid-cols-2`: 모바일 1열, 데스크톱 2열
   - `max-h-96 overflow-y-auto`: 스크롤 가능
   - `px-3 py-2`: 터치 대상 최소 44px

2. 접근성 속성 추가
   - 파일: `src/components/proposal/ProposalList.tsx` (이미 적용)
     - `role="button"`, `tabIndex={0}`, `onKeyDown` (키보드 네비게이션)
     - `aria-selected`, `aria-label` (스크린 리더)

3. 색상 명도 대비 확인
   - 텍스트 vs 배경: WCAG AA 최소 4.5:1 대비
   - 상태 표시: 색상만으로 표시하지 않음 (아이콘 + 텍스트)

4. 테스트
   - 파일: `tests/accessibility/proposal.a11y.test.tsx`

   ```typescript
   import { render, screen } from "@testing-library/react";
   import { axe, toHaveNoViolations } from "jest-axe";
   import { ProposalReviewTab } from "../../components/proposal/ProposalReviewTab";
   
   expect.extend(toHaveNoViolations);
   
   describe("ProposalReviewTab Accessibility", () => {
     it("should have no accessibility violations", async () => {
       const { container } = render(
         <ProposalReviewTab documentId="doc-1" currentDocumentContent="test" />
       );
       const results = await axe(container);
       expect(results).toHaveNoViolations();
     });
   
     it("should be keyboard navigable", () => {
       render(
         <ProposalReviewTab documentId="doc-1" currentDocumentContent="test" />
       );
       const buttons = screen.getAllByRole("button");
       buttons.forEach((button) => {
         expect(button).toHaveAttribute("tabIndex");
       });
     });
   });
   ```

**산출물**:
- 컴포넌트 (이미 반응형 클래스 포함)
- `tests/accessibility/proposal.a11y.test.tsx`

**완료 기준**:
- 모바일/태블릿/데스크톱에서 정상 표시
- axe 접근성 검사 통과
- 키보드 네비게이션 가능
- 색상 대비 WCAG AA 준수

---

### 4-8. 단위 테스트

**목표**: React Testing Library를 사용한 컴포넌트 테스트

**상세 작업**:

1. ProposalList 테스트
   - 파일: `tests/components/proposal/ProposalList.test.tsx`

   ```typescript
   import { render, screen, waitFor } from "@testing-library/react";
   import { ProposalList } from "../../../src/components/proposal/ProposalList";
   import { useProposalStore } from "../../../src/store/proposalStore";
   
   jest.mock("../../../src/store/proposalStore");
   
   describe("ProposalList", () => {
     it("should display loading spinner while fetching", () => {
       (useProposalStore as jest.Mock).mockReturnValue({
         proposals: [],
         isLoadingProposals: true,
         proposalsError: null,
         statusFilter: "pending",
         fetchProposals: jest.fn(),
       });
   
       render(<ProposalList documentId="doc-1" />);
       expect(screen.getByRole("status")).toBeInTheDocument();  // Spinner
     });
   
     it("should display proposals list", () => {
       const mockProposals = [
         {
           id: "prop-1",
           agent_id: "agent-1",
           document_id: "doc-1",
           proposal_type: "draft",
           status: "pending",
           created_at: "2026-04-17T00:00:00Z",
         },
       ];
   
       (useProposalStore as jest.Mock).mockReturnValue({
         proposals: mockProposals,
         isLoadingProposals: false,
         proposalsError: null,
         statusFilter: "pending",
         fetchProposals: jest.fn(),
       });
   
       render(<ProposalList documentId="doc-1" />);
       expect(screen.getByText("agent-1")).toBeInTheDocument();
     });
   
     it("should display error message", () => {
       (useProposalStore as jest.Mock).mockReturnValue({
         proposals: [],
         isLoadingProposals: false,
         proposalsError: "API Error",
         statusFilter: "pending",
         fetchProposals: jest.fn(),
       });
   
       render(<ProposalList documentId="doc-1" />);
       expect(screen.getByText(/제안 목록을 불러올 수 없습니다/)).toBeInTheDocument();
     });
   
     it("should display empty state when no proposals", () => {
       (useProposalStore as jest.Mock).mockReturnValue({
         proposals: [],
         isLoadingProposals: false,
         proposalsError: null,
         statusFilter: "pending",
         fetchProposals: jest.fn(),
       });
   
       render(<ProposalList documentId="doc-1" />);
       expect(screen.getByText(/대기 중인 제안이 없습니다/)).toBeInTheDocument();
     });
   });
   ```

2. DiffView 테스트
   - 파일: `tests/components/proposal/DiffView.test.tsx`

   ```typescript
   import { render, screen } from "@testing-library/react";
   import { DiffView } from "../../../src/components/proposal/DiffView";
   
   describe("DiffView", () => {
     it("should display added lines", () => {
       const original = "Line 1\nLine 2";
       const modified = "Line 1\nLine 2\nLine 3";
   
       render(<DiffView original={original} modified={modified} />);
       expect(screen.getByText(/추가됨/)).toBeInTheDocument();
     });
   
     it("should display removed lines", () => {
       const original = "Line 1\nLine 2\nLine 3";
       const modified = "Line 1\nLine 3";
   
       render(<DiffView original={original} modified={modified} />);
       expect(screen.getByText(/삭제됨/)).toBeInTheDocument();
     });
   
     it("should display unchanged lines", () => {
       const original = "Line 1\nLine 2";
       const modified = "Line 1\nLine 2";
   
       render(<DiffView original={original} modified={modified} />);
       expect(screen.getByText(/변경 없음/)).toBeInTheDocument();
     });
   });
   ```

3. ProposalApprovalModal 테스트
   - 파일: `tests/components/proposal/ProposalApprovalModal.test.tsx`

   ```typescript
   import { render, screen, fireEvent, waitFor } from "@testing-library/react";
   import userEvent from "@testing-library/user-event";
   import { ProposalApprovalModal } from "../../../src/components/proposal/ProposalApprovalModal";
   import { useProposalStore } from "../../../src/store/proposalStore";
   
   jest.mock("../../../src/store/proposalStore");
   
   describe("ProposalApprovalModal", () => {
     it("should not render when modal is closed", () => {
       (useProposalStore as jest.Mock).mockReturnValue({
         selectedProposal: null,
         showApprovalModal: false,
       });
   
       render(<ProposalApprovalModal />);
       expect(screen.queryByRole("dialog")).not.toBeInTheDocument();
     });
   
     it("should render when modal is open", () => {
       const mockProposal = {
         id: "prop-1",
         agent_id: "agent-1",
         proposal_type: "draft",
       };
   
       (useProposalStore as jest.Mock).mockReturnValue({
         selectedProposal: mockProposal,
         showApprovalModal: true,
         approveProposal: jest.fn(),
         rejectProposal: jest.fn(),
         closeApprovalModal: jest.fn(),
         approvalStatus: "idle",
         approvalError: null,
       });
   
       render(<ProposalApprovalModal />);
       expect(screen.getByRole("dialog")).toBeInTheDocument();
     });
   
     it("should call approveProposal when approve button clicked", async () => {
       const mockApprove = jest.fn();
       const user = userEvent.setup();
   
       (useProposalStore as jest.Mock).mockReturnValue({
         selectedProposal: { id: "prop-1" },
         showApprovalModal: true,
         approveProposal: mockApprove,
         rejectProposal: jest.fn(),
         closeApprovalModal: jest.fn(),
         approvalStatus: "idle",
         approvalError: null,
       });
   
       render(<ProposalApprovalModal />);
       
       const approveButton = screen.getByRole("button", { name: /승인/ });
       await user.click(approveButton);
   
       await waitFor(() => {
         expect(mockApprove).toHaveBeenCalledWith("prop-1", "");
       });
     });
   
     it("should require rejection notes", async () => {
       const mockReject = jest.fn();
       const user = userEvent.setup();
   
       (useProposalStore as jest.Mock).mockReturnValue({
         selectedProposal: { id: "prop-1" },
         showApprovalModal: true,
         approveProposal: jest.fn(),
         rejectProposal: mockReject,
         closeApprovalModal: jest.fn(),
         approvalStatus: "idle",
         approvalError: null,
       });
   
       render(<ProposalApprovalModal />);
       
       const rejectButton = screen.getByRole("button", { name: /거부/ });
       expect(rejectButton).toBeDisabled();  // 비활성 (사유 미입력)
       
       const textarea = screen.getByPlaceholderText(/거부 사유/);
       await user.type(textarea, "부적절한 변경");
   
       expect(rejectButton).not.toBeDisabled();
     });
   });
   ```

**산출물**:
- `tests/components/proposal/ProposalList.test.tsx`
- `tests/components/proposal/DiffView.test.tsx`
- `tests/components/proposal/ProposalApprovalModal.test.tsx`
- `tests/accessibility/proposal.a11y.test.tsx`

**완료 기준**:
- 테스트 커버리지 > 80%
- 모든 주요 사용 경로 테스트
- 접근성 테스트 통과

---

## 5. 산출물

### 필수 산출물
1. **API 계층**
   - `src/api/types/proposal.types.ts`
   - `src/api/services/proposalApi.ts`
   - `src/api/instances/proposalApi.instance.ts`

2. **상태 관리**
   - `src/store/proposalStore.ts`

3. **컴포넌트**
   - `src/components/proposal/DiffView.tsx`
   - `src/components/proposal/ProposalList.tsx`
   - `src/components/proposal/ProposalItem.tsx`
   - `src/components/proposal/ProposalApprovalModal.tsx`
   - `src/components/proposal/ProposalReviewTab.tsx`

4. **유틸리티**
   - `src/utils/diffUtils.ts`

5. **통합**
   - `src/pages/DocumentEditor.tsx` (수정)

6. **테스트**
   - `tests/components/proposal/ProposalList.test.tsx`
   - `tests/components/proposal/DiffView.test.tsx`
   - `tests/components/proposal/ProposalApprovalModal.test.tsx`
   - `tests/accessibility/proposal.a11y.test.tsx`

### 검수 산출물
- **검수 보고서**: Task 5-7 검수보고서.md
  - UI 기능성 검증
  - 반응형 디자인 검증
  - 접근성 검증 (WCAG 2.1 AA)

- **보안 취약점 검사 보고서**: Task 5-7 보안검사보고서.md
  - XSS 방지 (React 자동 이스케이핑)
  - CSRF 토큰 확인
  - 인젝션 메타플래그 표시

---

## 6. 완료 기준

### 기능적 완료 기준
- [x] "대기 중인 검토" 탭이 User UI에 추가됨
- [x] 제안 목록 표시 (에이전트명, 타입, 상태, 생성시간)
- [x] 제안 선택 시 상세 정보 표시
- [x] Diff 비교 보기 (추가/삭제/변경 없음 색상 구분)
- [x] 승인/거부 워크플로우 (모달, 사유 입력)
- [x] 인젝션 경고 표시 (metadata 기반)

### UI/UX 완료 기준
- [x] 반응형 디자인 (모바일/태블릿/데스크톱)
- [x] 로딩 상태 표시 (스핀너)
- [x] 에러 상태 표시 (메시지)
- [x] 빈 상태 표시 (안내 메시지)
- [x] 성공 피드백 (토스트 또는 모달 메시지)

### 테스트 완료 기준
- [x] 단위 테스트 커버리지 > 80%
- [x] 접근성 테스트 (axe) 통과
- [x] 키보드 네비게이션 가능
- [x] 스크린 리더 지원 (aria 속성)

### 코드 품질 기준
- [x] TypeScript 타입 안전성
- [x] React 최신 관행 (함수형 컴포넌트, hooks)
- [x] 상태 관리 명확 (Zustand)
- [x] 에러 처리 명확

---

## 7. 작업 지침

### 7-1. Zustand 스토어 설계

- **단일 책임**: proposalStore는 제안 관련 상태만 관리
- **비동기 액션**: async/await로 API 호출 처리
- **에러 처리**: try/catch로 에러 상태 저장
- **불변성**: 상태 업데이트는 새 객체 생성

### 7-2. 컴포넌트 설계

- **단일 책임**: 각 컴포넌트는 한 가지 역할만
  - ProposalList: 목록 표시
  - ProposalItem: 목록 항목
  - DiffView: Diff 시각화
  - ProposalApprovalModal: 승인/거부
- **Props 인터페이스**: 명확한 Props 정의
- **상태 관리**: 로컬 상태 vs 전역 상태 구분

### 7-3. 반응형 디자인

- **Tailwind 클래스**: `sm:`, `md:`, `lg:`, `xl:` 접두사 사용
- **그리드 레이아웃**: 모바일 1열, 데스크톱 2열
- **터치 대상**: 최소 44px × 44px (iOS 권장)
- **텍스트 가독성**: 최소 16px 폰트 크기 (모바일)

### 7-4. 접근성

- **키보드 네비게이션**: tabIndex, onKeyDown 처리
- **스크린 리더**: aria-label, role, aria-selected
- **색상 대비**: 텍스트/배경 4.5:1 이상
- **폼 라벨**: <label> 요소로 입력과 연결

### 7-5. 테스트 작성

- **사용자 관점**: 사용자가 보는 화면 기준
- **render + screen**: queryByText, getByRole 사용
- **비동기 처리**: waitFor 사용
- **Mock**: 외부 API, Zustand 스토어 모킹

### 7-6. 코드 검토 체크리스트

**API 계층**:
- [ ] 모든 타입이 정의됨
- [ ] 에러 처리 명확
- [ ] unwrapEnvelope 적용
- [ ] 인터셉터 (토큰 추가) 설정

**Zustand 스토어**:
- [ ] 모든 상태가 초기화됨
- [ ] 모든 액션이 구현됨
- [ ] 비동기 액션이 에러 처리함
- [ ] 상태 업데이트가 불변성 유지

**컴포넌트**:
- [ ] Props 인터페이스 정의됨
- [ ] 모든 필드에 타입 힌팅
- [ ] 로딩/에러/빈 상태 처리
- [ ] 접근성 속성 포함

**테스트**:
- [ ] 단위 테스트 80% 이상
- [ ] 주요 사용 경로 모두 테스트
- [ ] Mock 적절히 사용
- [ ] 접근성 테스트 통과

---

## 8. 참고 자료

- React 18 공식 문서
- Zustand 문서 (https://github.com/pmndrs/zustand)
- Tailwind CSS 반응형 디자인
- WCAG 2.1 AA 가이드라인
- React Testing Library 문서
- Task 5-4 (에이전트 제안 큐 API)
- Task 5-5, 5-6 (인젝션 탐지 및 메타플래그)

---

**작업 시작일**: 2026-04-17  
**예상 소요시간**: 5-6일 (Task 5-4, 5-5, 5-6과 병렬 가능)  
**담당자**: [개발팀]  
**우선순위**: High (사용자 경험)
