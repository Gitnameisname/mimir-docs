# Task 6-7: 에이전트 제안 큐 (Diff 뷰, 일괄 승인/거절, 실시간)

## 1. 작업 목적
에이전트가 제출한 문서 수정 제안을 관리자가 승인 또는 거절할 수 있는 관리 콘솔을 구축합니다. Diff 뷰로 기존 content와 제안된 content를 line-by-line으로 비교하고, WebSocket 또는 polling으로 실시간 상태 업데이트를 제공합니다. 일괄 승인/거절로 대량 제안을 효율적으로 처리합니다.

## 2. 작업 범위

### 포함 사항
- `/admin/proposals` 페이지
- 제안 목록 (AdminTable: 에이전트명, 문서명, 제안 요약, 시각, 상태)
- 필터링: 에이전트별, 문서별, 상태별, 기간별
- 제안 상세 보기 (메타데이터, DiffView line-by-line, 승인/거절+피드백)
- 일괄 작업 (체크박스 다중 선택 → 모두 승인/거절)
- 실시간 상태 업데이트 (WebSocket 또는 React Query polling 10초)
- API 통합 (proposalApi.ts unwrapEnvelope)
- React Testing Library 단위 테스트

### 제외 사항
- 에이전트 관리 (Task 6-5)
- Scope Profile 관리 (Task 6-6)
- 에이전트 활동 대시보드 (Task 6-8)
- 제안 내용 자동 수정 (관리자는 승인만, 백엔드에서 적용)
- WebSocket 서버 구현 (백엔드 책임)

## 3. 선행 조건
- Phase 4 완료: Proposal, ProposalDecision DB 모델
- Phase 5 완료: 제안 큐 API 엔드포인트
- 백엔드 API 문서: `/api/v1/proposals` (GET), `/api/v1/proposals/{id}` (GET), `/api/v1/proposals/{id}/approve` (POST), `/api/v1/proposals/{id}/reject` (POST), `/api/v1/proposals/batch` (POST)
- WebSocket 엔드포인트: `wss://api.example.com/ws/proposals` (선택)
- Next.js 16.2.2, React 19, TypeScript, Zustand, TanStack Table 설치 완료
- react-diff-viewer 또는 유사 Diff 뷰 라이브러리 준비

## 4. 주요 작업 항목

### 4-1 제안 목록 페이지
**목표**: `/admin/proposals` 페이지에서 전체 제안 조회 및 필터링

**세부 작업**:

1. **페이지 컴포넌트 생성** (`app/admin/proposals/page.tsx`)
   - 헤더: "에이전트 제안 큐" 제목
   - 필터 섹션: 에이전트, 문서, 상태, 기간 필터
   - AdminTable 컴포넌트 (이름, 문서, 요약, 시각, 상태)
   - 체크박스 (일괄 선택용)
   - 일괄 작업 버튼 (선택된 제안이 있을 때만 활성)

   ```typescript
   // app/admin/proposals/page.tsx
   'use client';
   
   import { useState, useEffect } from 'react';
   import { useQuery } from '@tanstack/react-query';
   import AdminTable from '@/components/admin/AdminTable';
   import ProposalDetailModal from '@/components/admin/proposals/ProposalDetailModal';
   import { proposalApi } from '@/lib/api/proposalApi';
   import { useProposalStore } from '@/store/proposalStore';
   import toast from 'react-hot-toast';
   
   export default function ProposalsPage() {
     const [selectedProposalId, setSelectedProposalId] = useState<string | null>(null);
     const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
     const [filters, setFilters] = useState({
       agentId: '',
       documentId: '',
       status: '', // pending, approved, rejected
       startDate: '',
       endDate: ''
     });
     
     const { proposals, setProposals, updateProposal } = useProposalStore();
     
     // 주 쿼리: 제안 목록 조회
     const { data, isLoading, refetch } = useQuery({
       queryKey: ['proposals', filters],
       queryFn: async () => {
         const response = await proposalApi.listProposals(filters);
         return response;
       }
     });
     
     // 실시간 업데이트: WebSocket 또는 polling
     useEffect(() => {
       const interval = setInterval(() => {
         refetch();
       }, 10000); // 10초마다 갱신
       
       return () => clearInterval(interval);
     }, [refetch]);
     
     // 데이터 로드 시 store 업데이트
     useEffect(() => {
       if (data?.proposals) {
         setProposals(data.proposals);
       }
     }, [data?.proposals, setProposals]);
     
     const columns = [
       {
         id: 'checkbox',
         header: ({ table }: any) => (
           <input
             type="checkbox"
             checked={selectedIds.size === proposals.length}
             onChange={() => {
               if (selectedIds.size === proposals.length) {
                 setSelectedIds(new Set());
               } else {
                 setSelectedIds(new Set(proposals.map((p: any) => p.id)));
               }
             }}
           />
         ),
         cell: (row: any) => (
           <input
             type="checkbox"
             checked={selectedIds.has(row.original.id)}
             onChange={() => {
               const newSet = new Set(selectedIds);
               if (newSet.has(row.original.id)) {
                 newSet.delete(row.original.id);
               } else {
                 newSet.add(row.original.id);
               }
               setSelectedIds(newSet);
             }}
           />
         )
       },
       { id: 'agentName', header: '에이전트명', accessorKey: 'agentName' },
       { id: 'documentTitle', header: '문서명', accessorKey: 'documentTitle' },
       { 
         id: 'summary', 
         header: '요약', 
         cell: (row: any) => row.original.summary?.substring(0, 50) + '...' 
       },
       { 
         id: 'createdAt', 
         header: '시각', 
         cell: (row: any) => new Date(row.original.createdAt).toLocaleString('ko-KR') 
       },
       { 
         id: 'status', 
         header: '상태', 
         cell: (row: any) => (
           <span className={`px-2 py-1 text-xs rounded font-semibold ${
             row.original.status === 'approved' ? 'bg-green-100 text-green-800' :
             row.original.status === 'rejected' ? 'bg-red-100 text-red-800' :
             'bg-yellow-100 text-yellow-800'
           }`}>
             {row.original.status === 'approved' ? '승인됨' :
              row.original.status === 'rejected' ? '거절됨' :
              '대기 중'}
           </span>
         )
       },
       {
         id: 'actions',
         header: '작업',
         cell: (row: any) => (
           <button 
             onClick={() => setSelectedProposalId(row.original.id)}
             className="text-blue-600 hover:underline"
           >
             상세보기
           </button>
         )
       }
     ];
     
     const handleApproveSelected = async () => {
       const ids = Array.from(selectedIds);
       try {
         await proposalApi.batchApprove({ proposalIds: ids });
         toast.success(`${ids.length}개 제안 승인 완료`);
         setSelectedIds(new Set());
         refetch();
       } catch (error) {
         toast.error('승인 처리 중 오류 발생');
       }
     };
     
     const handleRejectSelected = async () => {
       const ids = Array.from(selectedIds);
       try {
         await proposalApi.batchReject({ proposalIds: ids });
         toast.success(`${ids.length}개 제안 거절 완료`);
         setSelectedIds(new Set());
         refetch();
       } catch (error) {
         toast.error('거절 처리 중 오류 발생');
       }
     };
     
     if (isLoading) return <div className="p-6">로딩 중...</div>;
     
     return (
       <div className="p-6">
         <h1 className="text-2xl font-bold mb-6">에이전트 제안 큐</h1>
         
         {/* 필터 섹션 */}
         <div className="bg-white rounded-lg p-4 mb-6 border space-y-4">
           <h2 className="font-semibold">필터</h2>
           <div className="grid grid-cols-4 gap-4">
             <div>
               <label className="block text-sm font-medium mb-1">에이전트</label>
               <select
                 value={filters.agentId}
                 onChange={(e) => setFilters({...filters, agentId: e.target.value})}
                 className="w-full border rounded px-3 py-2 text-sm"
               >
                 <option value="">모두</option>
                 {/* 에이전트 목록 동적 로드 */}
               </select>
             </div>
             <div>
               <label className="block text-sm font-medium mb-1">문서</label>
               <input
                 type="text"
                 value={filters.documentId}
                 onChange={(e) => setFilters({...filters, documentId: e.target.value})}
                 placeholder="문서명 검색"
                 className="w-full border rounded px-3 py-2 text-sm"
               />
             </div>
             <div>
               <label className="block text-sm font-medium mb-1">상태</label>
               <select
                 value={filters.status}
                 onChange={(e) => setFilters({...filters, status: e.target.value})}
                 className="w-full border rounded px-3 py-2 text-sm"
               >
                 <option value="">모두</option>
                 <option value="pending">대기 중</option>
                 <option value="approved">승인됨</option>
                 <option value="rejected">거절됨</option>
               </select>
             </div>
             <div>
               <label className="block text-sm font-medium mb-1">기간</label>
               <div className="flex gap-1">
                 <input
                   type="date"
                   value={filters.startDate}
                   onChange={(e) => setFilters({...filters, startDate: e.target.value})}
                   className="flex-1 border rounded px-2 py-2 text-sm"
                 />
                 <span className="px-2 py-2">~</span>
                 <input
                   type="date"
                   value={filters.endDate}
                   onChange={(e) => setFilters({...filters, endDate: e.target.value})}
                   className="flex-1 border rounded px-2 py-2 text-sm"
                 />
               </div>
             </div>
           </div>
         </div>
         
         {/* 일괄 작업 버튼 */}
         {selectedIds.size > 0 && (
           <div className="bg-blue-50 border border-blue-200 rounded-lg p-4 mb-4 flex justify-between items-center">
             <span className="text-sm font-medium">{selectedIds.size}개 선택됨</span>
             <div className="flex gap-2">
               <button
                 onClick={handleApproveSelected}
                 className="px-4 py-2 bg-green-600 text-white rounded hover:bg-green-700 text-sm"
               >
                 모두 승인
               </button>
               <button
                 onClick={handleRejectSelected}
                 className="px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700 text-sm"
               >
                 모두 거절
               </button>
             </div>
           </div>
         )}
         
         {/* 제안 테이블 */}
         <div className="bg-white rounded-lg border overflow-hidden">
           <AdminTable 
             columns={columns} 
             data={proposals} 
           />
         </div>
         
         {/* 상세 보기 모달 */}
         {selectedProposalId && (
           <ProposalDetailModal
             proposalId={selectedProposalId}
             onClose={() => setSelectedProposalId(null)}
             onStatusChange={() => refetch()}
           />
         )}
       </div>
     );
   }
   ```

### 4-2 제안 상세 보기 모달
**목표**: 제안 메타데이터, Diff 뷰, 승인/거절 처리

**세부 작업**:

1. **ProposalDetailModal 컴포넌트** (`components/admin/proposals/ProposalDetailModal.tsx`)

   ```typescript
   // components/admin/proposals/ProposalDetailModal.tsx
   'use client';
   
   import { useState } from 'react';
   import { useQuery, useMutation } from '@tanstack/react-query';
   import { proposalApi } from '@/lib/api/proposalApi';
   import DiffView from '@/components/admin/proposals/DiffView';
   import toast from 'react-hot-toast';
   
   interface ProposalDetailModalProps {
     proposalId: string;
     onClose: () => void;
     onStatusChange: () => void;
   }
   
   export default function ProposalDetailModal({
     proposalId,
     onClose,
     onStatusChange
   }: ProposalDetailModalProps) {
     const [feedback, setFeedback] = useState('');
     const [showFeedbackInput, setShowFeedbackInput] = useState(false);
     
     const { data: proposal, isLoading } = useQuery({
       queryKey: ['proposal', proposalId],
       queryFn: async () => {
         const response = await proposalApi.getProposal(proposalId);
         return response.proposal;
       }
     });
     
     const approveMutation = useMutation({
       mutationFn: async () => {
         await proposalApi.approveProposal(proposalId, { feedback });
       },
       onSuccess: () => {
         toast.success('제안 승인 완료');
         onStatusChange();
         onClose();
       },
       onError: () => {
         toast.error('승인 처리 실패');
       }
     });
     
     const rejectMutation = useMutation({
       mutationFn: async () => {
         if (!feedback.trim()) {
           throw new Error('거절 사유를 입력하세요');
         }
         await proposalApi.rejectProposal(proposalId, { feedback });
       },
       onSuccess: () => {
         toast.success('제안 거절 완료');
         onStatusChange();
         onClose();
       },
       onError: (error: any) => {
         toast.error(error.message || '거절 처리 실패');
       }
     });
     
     if (isLoading) return null;
     if (!proposal) return null;
     
     return (
       <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4">
         <div className="bg-white rounded-lg p-6 w-full max-w-4xl max-h-[90vh] overflow-y-auto">
           <div className="flex justify-between items-center mb-4">
             <h2 className="text-xl font-bold">제안 상세</h2>
             <button onClick={onClose} className="text-2xl">×</button>
           </div>
           
           {/* 메타데이터 섹션 */}
           <section className="mb-6 border-b pb-4">
             <h3 className="text-lg font-semibold mb-3">메타데이터</h3>
             <div className="grid grid-cols-2 gap-4 text-sm">
               <div>
                 <label className="text-gray-600 font-medium">에이전트</label>
                 <p>{proposal.agentName}</p>
               </div>
               <div>
                 <label className="text-gray-600 font-medium">문서</label>
                 <p>{proposal.documentTitle}</p>
               </div>
               <div>
                 <label className="text-gray-600 font-medium">목표 상태</label>
                 <p>{proposal.targetStatus}</p>
               </div>
               <div>
                 <label className="text-gray-600 font-medium">생성일</label>
                 <p>{new Date(proposal.createdAt).toLocaleString('ko-KR')}</p>
               </div>
               <div className="col-span-2">
                 <label className="text-gray-600 font-medium">사유</label>
                 <p className="text-gray-800">{proposal.reason}</p>
               </div>
             </div>
           </section>
           
           {/* Diff View 섹션 */}
           <section className="mb-6 border-b pb-4">
             <h3 className="text-lg font-semibold mb-3">변경 사항 (Diff)</h3>
             <DiffView 
               oldContent={proposal.currentContent}
               newContent={proposal.proposedContent}
             />
           </section>
           
           {/* 피드백 입력 */}
           <section className="mb-6">
             <h3 className="text-lg font-semibold mb-3">승인/거절</h3>
             {proposal.status === 'pending' && (
               <>
                 {showFeedbackInput && (
                   <div className="mb-4">
                     <label className="block text-sm font-medium mb-2">피드백 (선택)</label>
                     <textarea
                       value={feedback}
                       onChange={(e) => setFeedback(e.target.value)}
                       className="w-full border rounded px-3 py-2"
                       placeholder="거절 또는 승인 사유 입력"
                       rows={4}
                     />
                   </div>
                 )}
                 
                 <div className="flex gap-2">
                   <button
                     onClick={() => approveMutation.mutate()}
                     disabled={approveMutation.isPending}
                     className="px-6 py-2 bg-green-600 text-white rounded hover:bg-green-700 disabled:opacity-50"
                   >
                     {approveMutation.isPending ? '승인 중...' : '승인'}
                   </button>
                   <button
                     onClick={() => setShowFeedbackInput(!showFeedbackInput)}
                     className={`px-6 py-2 rounded ${
                       showFeedbackInput ? 'bg-red-600 text-white hover:bg-red-700' : 'bg-gray-200 hover:bg-gray-300'
                     }`}
                   >
                     {showFeedbackInput ? '거절하기' : '거절'}
                   </button>
                   {showFeedbackInput && (
                     <button
                       onClick={() => {
                         setShowFeedbackInput(false);
                         setFeedback('');
                       }}
                       className="px-4 py-2 border rounded hover:bg-gray-100"
                     >
                       취소
                     </button>
                   )}
                 </div>
                 
                 {showFeedbackInput && (
                   <button
                     onClick={() => rejectMutation.mutate()}
                     disabled={rejectMutation.isPending}
                     className="mt-2 w-full px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700 disabled:opacity-50"
                   >
                     {rejectMutation.isPending ? '거절 중...' : '거절 확인'}
                   </button>
                 )}
               </>
             )}
             
             {proposal.status !== 'pending' && (
               <div className={`p-4 rounded ${
                 proposal.status === 'approved' ? 'bg-green-50' : 'bg-red-50'
               }`}>
                 <p className="font-semibold">
                   {proposal.status === 'approved' ? '승인됨' : '거절됨'}
                 </p>
                 {proposal.decision?.feedback && (
                   <p className="text-sm mt-2">{proposal.decision.feedback}</p>
                 )}
               </div>
             )}
           </section>
         </div>
       </div>
     );
   }
   ```

### 4-3 Diff View 컴포넌트
**목표**: line-by-line 문서 변경 사항 시각화

**세부 작업**:

1. **DiffView 컴포넌트** (`components/admin/proposals/DiffView.tsx`)
   - 기존 content와 제안 content 비교
   - 삭제된 줄: 빨간 배경, 추가된 줄: 초록 배경
   - 라인 넘버 표시

   ```typescript
   // components/admin/proposals/DiffView.tsx
   'use client';
   
   import React from 'react';
   
   interface DiffViewProps {
     oldContent: string;
     newContent: string;
   }
   
   interface DiffLine {
     type: 'unchanged' | 'added' | 'removed' | 'modified';
     oldLine?: string;
     newLine?: string;
     oldLineNum?: number;
     newLineNum?: number;
   }
   
   function computeDiff(old: string, newStr: string): DiffLine[] {
     const oldLines = old.split('\n');
     const newLines = newStr.split('\n');
     const diffs: DiffLine[] = [];
     
     let i = 0, j = 0;
     let oldLineNum = 1, newLineNum = 1;
     
     // 간단한 diff 알고리즘 (실제로는 diff-match-patch 같은 라이브러리 권장)
     while (i < oldLines.length || j < newLines.length) {
       if (i < oldLines.length && j < newLines.length && oldLines[i] === newLines[j]) {
         diffs.push({
           type: 'unchanged',
           oldLine: oldLines[i],
           newLine: newLines[j],
           oldLineNum,
           newLineNum
         });
         i++;
         j++;
         oldLineNum++;
         newLineNum++;
       } else if (i < oldLines.length && (j >= newLines.length || oldLines[i] !== newLines[j])) {
         diffs.push({
           type: 'removed',
           oldLine: oldLines[i],
           oldLineNum
         });
         i++;
         oldLineNum++;
       } else {
         diffs.push({
           type: 'added',
           newLine: newLines[j],
           newLineNum
         });
         j++;
         newLineNum++;
       }
     }
     
     return diffs;
   }
   
   export default function DiffView({ oldContent, newContent }: DiffViewProps) {
     const diffs = computeDiff(oldContent, newContent);
     
     return (
       <div className="border rounded-lg overflow-hidden bg-gray-50">
         <div className="overflow-x-auto">
           <table className="w-full font-mono text-sm">
             <thead>
               <tr className="bg-gray-200 border-b">
                 <th className="w-12 p-2 text-right text-gray-600">이전</th>
                 <th className="w-12 p-2 text-right text-gray-600">현재</th>
                 <th className="p-2 text-left">변경</th>
               </tr>
             </thead>
             <tbody>
               {diffs.map((diff, idx) => (
                 <tr
                   key={idx}
                   className={`border-b ${
                     diff.type === 'added'
                       ? 'bg-green-50'
                       : diff.type === 'removed'
                       ? 'bg-red-50'
                       : 'bg-white'
                   }`}
                 >
                   <td className="w-12 p-2 text-right text-gray-500 bg-gray-100 border-r">
                     {diff.oldLineNum || ''}
                   </td>
                   <td className="w-12 p-2 text-right text-gray-500 bg-gray-100 border-r">
                     {diff.newLineNum || ''}
                   </td>
                   <td className="p-2 whitespace-pre-wrap break-words">
                     {diff.type === 'removed' && (
                       <span className="bg-red-200 text-red-900 px-1">- {diff.oldLine}</span>
                     )}
                     {diff.type === 'added' && (
                       <span className="bg-green-200 text-green-900 px-1">+ {diff.newLine}</span>
                     )}
                     {diff.type === 'unchanged' && (
                       <span>{diff.oldLine}</span>
                     )}
                   </td>
                 </tr>
               ))}
             </tbody>
           </table>
         </div>
       </div>
     );
   }
   ```

   **참고**: 실무에서는 `diff-match-patch` 또는 `react-diff-viewer-continued` 라이브러리 사용 권장

2. **좀 더 견고한 DiffView** (선택)
   
   ```bash
   npm install diff-match-patch react-diff-viewer-continued
   ```

### 4-4 API 통합 (proposalApi.ts)
**목표**: 제안 관련 API 호출

**세부 작업**:

1. **proposalApi.ts 작성** (`lib/api/proposalApi.ts`)

   ```typescript
   // lib/api/proposalApi.ts
   import { apiClient } from '@/lib/api/client';
   import { unwrapEnvelope } from '@/lib/api/envelope';
   
   export const proposalApi = {
     async listProposals(filters: {
       agentId?: string;
       documentId?: string;
       status?: string;
       startDate?: string;
       endDate?: string;
     }) {
       const params = new URLSearchParams();
       Object.entries(filters).forEach(([key, value]) => {
         if (value) params.append(key, value);
       });
       
       const response = await apiClient.get(`/api/v1/proposals?${params}`);
       return unwrapEnvelope(response.data);
     },
     
     async getProposal(id: string) {
       const response = await apiClient.get(`/api/v1/proposals/${id}`);
       return unwrapEnvelope(response.data);
     },
     
     async approveProposal(id: string, payload: { feedback?: string }) {
       const response = await apiClient.post(`/api/v1/proposals/${id}/approve`, payload);
       return unwrapEnvelope(response.data);
     },
     
     async rejectProposal(id: string, payload: { feedback: string }) {
       const response = await apiClient.post(`/api/v1/proposals/${id}/reject`, payload);
       return unwrapEnvelope(response.data);
     },
     
     async batchApprove(payload: { proposalIds: string[] }) {
       const response = await apiClient.post('/api/v1/proposals/batch/approve', payload);
       return unwrapEnvelope(response.data);
     },
     
     async batchReject(payload: { proposalIds: string[] }) {
       const response = await apiClient.post('/api/v1/proposals/batch/reject', payload);
       return unwrapEnvelope(response.data);
     }
   };
   ```

### 4-5 Zustand proposalStore
**목표**: 클라이언트 제안 상태 관리

**세부 작업**:

1. **proposalStore.ts 작성** (`store/proposalStore.ts`)

   ```typescript
   // store/proposalStore.ts
   import { create } from 'zustand';
   
   interface Proposal {
     id: string;
     agentId: string;
     agentName: string;
     documentId: string;
     documentTitle: string;
     summary: string;
     reason: string;
     currentContent: string;
     proposedContent: string;
     targetStatus: string;
     status: 'pending' | 'approved' | 'rejected';
     decision?: {
       feedback: string;
       decidedAt: string;
       decidedBy: string;
     };
     createdAt: string;
   }
   
   interface ProposalStore {
     proposals: Proposal[];
     setProposals: (proposals: Proposal[]) => void;
     updateProposal: (id: string, updates: Partial<Proposal>) => void;
     removeProposal: (id: string) => void;
     getPendingProposals: () => Proposal[];
     getProposalsByAgent: (agentId: string) => Proposal[];
   }
   
   export const useProposalStore = create<ProposalStore>((set, get) => ({
     proposals: [],
     
     setProposals: (proposals) => set({ proposals }),
     
     updateProposal: (id, updates) => set((state) => ({
       proposals: state.proposals.map((p) => p.id === id ? { ...p, ...updates } : p)
     })),
     
     removeProposal: (id) => set((state) => ({
       proposals: state.proposals.filter((p) => p.id !== id)
     })),
     
     getPendingProposals: () => get().proposals.filter((p) => p.status === 'pending'),
     
     getProposalsByAgent: (agentId) => get().proposals.filter((p) => p.agentId === agentId)
   }));
   ```

### 4-6 실시간 업데이트 (선택: WebSocket)
**목표**: 제안 상태 변경 실시간 감지

**세부 작업**:

1. **useProposalWebSocket 훅** (`hooks/useProposalWebSocket.ts`)

   ```typescript
   // hooks/useProposalWebSocket.ts
   import { useEffect, useRef } from 'react';
   import { useProposalStore } from '@/store/proposalStore';
   
   export function useProposalWebSocket() {
     const wsRef = useRef<WebSocket | null>(null);
     const { updateProposal } = useProposalStore();
     
     useEffect(() => {
       const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
       const ws = new WebSocket(`${protocol}//${window.location.host}/api/v1/ws/proposals`);
       
       ws.onmessage = (event) => {
         const message = JSON.parse(event.data);
         
         if (message.type === 'proposal_updated') {
           const { proposalId, status, decision } = message.data;
           updateProposal(proposalId, { status, decision });
         }
       };
       
       ws.onerror = (error) => {
         console.error('WebSocket error:', error);
       };
       
       wsRef.current = ws;
       
       return () => {
         ws.close();
       };
     }, [updateProposal]);
   }
   ```

   **페이지에서 사용**:
   
   ```typescript
   // app/admin/proposals/page.tsx에서
   import { useProposalWebSocket } from '@/hooks/useProposalWebSocket';
   
   export default function ProposalsPage() {
     useProposalWebSocket(); // WebSocket 활성화
     // ... 나머지 코드
   }
   ```

### 4-7 React Testing Library 단위 테스트

1. **ProposalDetailModal.test.tsx** (`__tests__/components/admin/proposals/ProposalDetailModal.test.tsx`)

   ```typescript
   // __tests__/components/admin/proposals/ProposalDetailModal.test.tsx
   import { render, screen, fireEvent, waitFor } from '@testing-library/react';
   import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
   import ProposalDetailModal from '@/components/admin/proposals/ProposalDetailModal';
   import * as proposalApi from '@/lib/api/proposalApi';
   
   jest.mock('@/lib/api/proposalApi');
   
   const queryClient = new QueryClient();
   
   describe('ProposalDetailModal', () => {
     it('제안 정보를 로드하고 표시한다', async () => {
       const mockProposal = {
         id: '1',
         agentName: 'Agent A',
         documentTitle: 'Doc 1',
         reason: 'Improve clarity',
         currentContent: 'Old text',
         proposedContent: 'New text',
         status: 'pending',
         createdAt: new Date().toISOString()
       };
       
       (proposalApi.proposalApi.getProposal as jest.Mock).mockResolvedValue({
         proposal: mockProposal
       });
       
       render(
         <QueryClientProvider client={queryClient}>
           <ProposalDetailModal 
             proposalId="1" 
             onClose={() => {}}
             onStatusChange={() => {}}
           />
         </QueryClientProvider>
       );
       
       await waitFor(() => {
         expect(screen.getByText('Agent A')).toBeInTheDocument();
       });
     });
     
     it('승인 버튼 클릭 시 API를 호출한다', async () => {
       const mockProposal = {
         id: '1',
         agentName: 'Agent A',
         status: 'pending',
         createdAt: new Date().toISOString()
       };
       
       (proposalApi.proposalApi.getProposal as jest.Mock).mockResolvedValue({
         proposal: mockProposal
       });
       
       (proposalApi.proposalApi.approveProposal as jest.Mock).mockResolvedValue({});
       
       render(
         <QueryClientProvider client={queryClient}>
           <ProposalDetailModal 
             proposalId="1" 
             onClose={() => {}}
             onStatusChange={() => {}}
           />
         </QueryClientProvider>
       );
       
       await waitFor(() => {
         const approveButton = screen.getByText('승인');
         fireEvent.click(approveButton);
       });
       
       await waitFor(() => {
         expect(proposalApi.proposalApi.approveProposal).toHaveBeenCalledWith('1', expect.any(Object));
       });
     });
   });
   ```

2. **DiffView.test.tsx** (`__tests__/components/admin/proposals/DiffView.test.tsx`)
   - Diff 계산 정확성 확인
   - 색상 표시 (removed, added) 확인

## 5. 산출물

- `/app/admin/proposals/page.tsx` - 제안 목록 페이지
- `/components/admin/proposals/ProposalDetailModal.tsx` - 상세 보기 모달
- `/components/admin/proposals/DiffView.tsx` - Diff 뷰어
- `/lib/api/proposalApi.ts` - API 클라이언트
- `/store/proposalStore.ts` - Zustand 상태 관리
- `/hooks/useProposalWebSocket.ts` - WebSocket 훅 (선택)
- `/__tests__/components/admin/proposals/` - 단위 테스트 3개 이상

## 6. 완료 기준

- [ ] `/admin/proposals` 목록 페이지 완성 (필터 모두 작동)
- [ ] 체크박스 다중 선택 기능 완성
- [ ] 일괄 승인/거절 버튼 완성
- [ ] 제안 상세 보기 모달 완성
- [ ] Diff View 완성 (line-by-line, 색상 구분)
- [ ] 피드백 입력 및 승인/거절 기능 완성
- [ ] 실시간 업데이트 완성 (polling 최소 10초)
- [ ] proposalApi.ts 모든 메서드 구현 및 unwrapEnvelope 적용
- [ ] proposalStore 구현 완료
- [ ] React Testing Library 테스트 커버리지 ≥ 80%
- [ ] 모든 UI 컴포넌트 반응형 설계
- [ ] 검수 보고서 작성 완료
- [ ] 보안 취약점 검사 보고서 작성 완료

## 7. 작업 지침

### 7-1 실시간 업데이트 전략
- **기본: React Query polling** (10초 간격, 모든 브라우저 호환)
  ```typescript
  useQuery({
    queryKey: ['proposals', filters],
    queryFn: async () => { ... },
    refetchInterval: 10000 // 10초
  })
  ```
- **고급: WebSocket** (선택 사항, 서버 지원 시)
  - 메시지 형식: `{ type: 'proposal_updated', data: { proposalId, status, decision } }`
  - 재연결 로직 필수 (exponential backoff)

### 7-2 Diff 뷰 알고리즘
- 간단한 구현: 라인별 문자열 비교
- 고급 구현: `diff-match-patch` 또는 `react-diff-viewer-continued` 라이브러리
- 색상: 삭제(빨강), 추가(초록), 수정(노랑) 구분

### 7-3 일괄 작업 사용성
- 체크박스 헤더로 "모두 선택/해제" 기능
- 선택된 항목 수 실시간 표시
- 선택이 없으면 일괄 버튼 비활성

### 7-4 보안 체크리스트
- 거절 피드백은 에이전트에 노출됨 → 민감 정보 검증
- 승인/거절 API 호출은 감사 로그에 기록 필수
- 대량 승인/거절은 rate limiting 적용 (초당 최대 N개)

### 7-5 성능 최적화
- 대량 제안 목록: 가상화 (react-window) 고려
- Diff 계산: 대용량 파일 시 웹 워커 활용
- 실시간 업데이트: 백그라운드 리페치 (focus 감지)

### 7-6 [PH5-CARRY-003] 배포 전 부하 테스트 — Phase 6 완료 후 필수 실행

FG5.3 잔여 항목. Task 6-7 UI 완성 후, Phase 6 배포 전에 아래 시나리오를 실행해 `agent_proposals` 처리 성능을 검증한다.

**테스트 시나리오**:
1. `agent_proposals` 테이블에 1000건 이상의 레코드를 삽입한다.
2. `GET /admin/proposals?page=1&page_size=50` 응답 시간 측정 → 목표 **< 500ms**
3. `GET /admin/proposals?agent_id=X&status=pending` 복합 인덱스 효율 측정
4. 동시 요청 10개 병렬 실행 → 오류 없이 처리되는가 확인

**DB 인덱스 확인 포인트** (`backend/app/db/connection.py`):
- `agent_proposals.agent_id` 인덱스 존재 여부
- `agent_proposals.status` 인덱스 존재 여부
- `agent_proposals.created_at DESC` 정렬 인덱스 존재 여부

성능 미달 시 인덱스 추가 마이그레이션 DDL을 `connection.py`에 추가하고 재측정한다.

---

**예상 소요 시간**: 40시간 (분석 8시간 + DiffView 개발 8시간 + 나머지 CRUD 16시간 + 실시간 구현 4시간 + 테스트 4시간)
