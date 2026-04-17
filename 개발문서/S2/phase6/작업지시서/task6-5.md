# Task 6-5: 에이전트 관리 페이지 (목록, 상세, 킬스위치, API Key)

## 1. 작업 목적
Mimir 플랫폼의 관리 콘솔에서 에이전트 생명주기 전체를 관리할 수 있는 UI를 구축합니다. 에이전트 등록, 위임 사용자 관리, 킬스위치(차단) 기능, API Key 재발급을 통해 에이전트를 안전하고 효율적으로 제어하는 페이지입니다.

## 2. 작업 범위

### 포함 사항
- `/admin/agents` 페이지 구현
- 에이전트 목록 조회 (AdminTable: 이름, 설명, 위임 사용자 수, 상태, 마지막 활동)
- "에이전트 생성" 모달 (이름, 설명, 대행 권한 범위)
- 에이전트 상세 보기 UI 및 상호작용
  - 기본 정보 섹션
  - 위임 상태 및 위임 해제 기능
  - delegate scope 목록 (추가/제거)
  - 킬스위치 (1시간/24시간/영구, 대기 제안 자동거절 옵션)
  - API Key 재발급 (새 key 생성, 이전 key 즉시 무효)
- API 통합 (agentApi.ts unwrapEnvelope)
- Zustand agentStore 상태 관리
- React Testing Library 단위 테스트

### 제외 사항
- Scope Profile 관리 (Task 6-6)
- 에이전트 제안 큐 (Task 6-7)
- 에이전트 활동 대시보드 (Task 6-8)
- 에이전트 생성 시 자동 API Key 발급 (백엔드 책임)

## 3. 선행 조건
- Phase 4 완료: Agent Principal, Delegation DB 모델, API 엔드포인트 구현
- Phase 5 완료: 제안 큐 API, 에이전트 감시 API
- Next.js 16.2.2, React 19, TypeScript, Zustand, TanStack Table 설치 완료
- recharts, react-hot-toast 등 UI 라이브러리 설치 완료
- 백엔드 API 문서: `/api/v1/agents` (GET, POST, PATCH), `/api/v1/agents/{id}` (GET, DELETE), `/api/v1/agents/{id}/api-keys` (POST)

## 4. 주요 작업 항목

### 4-1 에이전트 목록 페이지 구현
**목표**: `/admin/agents` 페이지에서 전체 에이전트를 표 형식으로 조회

**세부 작업**:

1. **페이지 컴포넌트 생성** (`app/admin/agents/page.tsx`)
   - 헤더: "에이전트 관리" 제목, "에이전트 생성" 버튼
   - AdminTable 컴포넌트 추가
   - 로딩/에러/빈 상태 처리

   ```typescript
   // app/admin/agents/page.tsx
   'use client';
   
   import { useState } from 'react';
   import { useQuery } from '@tanstack/react-query';
   import AdminTable from '@/components/admin/AdminTable';
   import CreateAgentModal from '@/components/admin/agents/CreateAgentModal';
   import AgentDetailModal from '@/components/admin/agents/AgentDetailModal';
   import { agentApi } from '@/lib/api/agentApi';
   import { useAgentStore } from '@/store/agentStore';
   
   export default function AgentsPage() {
     const [showCreateModal, setShowCreateModal] = useState(false);
     const [selectedAgentId, setSelectedAgentId] = useState<string | null>(null);
     const { agents, setAgents } = useAgentStore();
     
     const { data, isLoading, error } = useQuery({
       queryKey: ['agents'],
       queryFn: async () => {
         const response = await agentApi.listAgents();
         return response;
       }
     });
     
     const columns = [
       { id: 'name', header: '이름', accessorKey: 'name' },
       { id: 'description', header: '설명', accessorKey: 'description' },
       { 
         id: 'delegationCount', 
         header: '위임 사용자 수', 
         cell: (row: any) => row.original.delegations?.length ?? 0 
       },
       { 
         id: 'status', 
         header: '상태', 
         cell: (row: any) => (
           <span className={`px-2 py-1 text-sm rounded ${
             row.original.isBlocked ? 'bg-red-100 text-red-800' : 'bg-green-100 text-green-800'
           }`}>
             {row.original.isBlocked ? '차단됨' : '활성'}
           </span>
         )
       },
       { 
         id: 'lastActivity', 
         header: '마지막 활동', 
         cell: (row: any) => row.original.lastActivityAt ? 
           new Date(row.original.lastActivityAt).toLocaleString('ko-KR') : '-' 
       },
       {
         id: 'actions',
         header: '작업',
         cell: (row: any) => (
           <button 
             onClick={() => setSelectedAgentId(row.original.id)}
             className="text-blue-600 hover:underline"
           >
             상세보기
           </button>
         )
       }
     ];
     
     if (isLoading) return <div className="p-6">로딩 중...</div>;
     if (error) return <div className="p-6 text-red-600">에러 발생</div>;
     
     return (
       <div className="p-6">
         <div className="flex justify-between items-center mb-6">
           <h1 className="text-2xl font-bold">에이전트 관리</h1>
           <button 
             onClick={() => setShowCreateModal(true)}
             className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
           >
             에이전트 생성
           </button>
         </div>
         
         <AdminTable 
           columns={columns} 
           data={data?.agents ?? []} 
         />
         
         {showCreateModal && (
           <CreateAgentModal onClose={() => setShowCreateModal(false)} />
         )}
         
         {selectedAgentId && (
           <AgentDetailModal 
             agentId={selectedAgentId}
             onClose={() => setSelectedAgentId(null)}
           />
         )}
       </div>
     );
   }
   ```

2. **AdminTable 컴포넌트 재사용** (`components/admin/AdminTable.tsx`)
   - TanStack Table 기반, 정렬/필터링 기능
   - 페이지네이션 (기본 20개 항목)

### 4-2 에이전트 생성 모달 구현
**목표**: 새 에이전트 등록 폼 UI

**세부 작업**:

1. **CreateAgentModal 컴포넌트** (`components/admin/agents/CreateAgentModal.tsx`)
   - 입력 필드: 이름, 설명
   - delegate scope 드롭다운 (Scope Profile 목록 로드)
   - "생성" 버튼 → POST `/api/v1/agents`
   - 유효성 검사: 이름 필수, 설명 권장

   ```typescript
   // components/admin/agents/CreateAgentModal.tsx
   'use client';
   
   import { useState } from 'react';
   import { useMutation } from '@tanstack/react-query';
   import { agentApi } from '@/lib/api/agentApi';
   import { useAgentStore } from '@/store/agentStore';
   import toast from 'react-hot-toast';
   
   interface CreateAgentModalProps {
     onClose: () => void;
   }
   
   export default function CreateAgentModal({ onClose }: CreateAgentModalProps) {
     const [formData, setFormData] = useState({
       name: '',
       description: '',
       delegateScopeId: ''
     });
     const { addAgent } = useAgentStore();
     
     const { mutate, isPending } = useMutation({
       mutationFn: async () => {
         const response = await agentApi.createAgent({
           name: formData.name,
           description: formData.description,
           delegateScopeId: formData.delegateScopeId
         });
         return response;
       },
       onSuccess: (data) => {
         addAgent(data.agent);
         toast.success('에이전트 생성 완료');
         onClose();
       },
       onError: (error) => {
         toast.error('에이전트 생성 실패');
       }
     });
     
     const handleSubmit = (e: React.FormEvent) => {
       e.preventDefault();
       if (!formData.name.trim()) {
         toast.error('이름을 입력하세요');
         return;
       }
       mutate();
     };
     
     return (
       <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
         <div className="bg-white rounded-lg p-6 w-full max-w-md">
           <h2 className="text-xl font-bold mb-4">에이전트 생성</h2>
           
           <form onSubmit={handleSubmit} className="space-y-4">
             <div>
               <label className="block text-sm font-medium mb-1">이름 *</label>
               <input
                 type="text"
                 value={formData.name}
                 onChange={(e) => setFormData({...formData, name: e.target.value})}
                 className="w-full border rounded px-3 py-2"
                 placeholder="예: 문서 분류 에이전트"
               />
             </div>
             
             <div>
               <label className="block text-sm font-medium mb-1">설명</label>
               <textarea
                 value={formData.description}
                 onChange={(e) => setFormData({...formData, description: e.target.value})}
                 className="w-full border rounded px-3 py-2"
                 placeholder="에이전트의 역할 설명"
                 rows={3}
               />
             </div>
             
             <div>
               <label className="block text-sm font-medium mb-1">대행 권한 범위</label>
               <select
                 value={formData.delegateScopeId}
                 onChange={(e) => setFormData({...formData, delegateScopeId: e.target.value})}
                 className="w-full border rounded px-3 py-2"
               >
                 <option value="">선택하세요</option>
                 {/* Scope Profile 목록 동적 로드 */}
               </select>
             </div>
             
             <div className="flex justify-end gap-2 mt-6">
               <button
                 type="button"
                 onClick={onClose}
                 className="px-4 py-2 border rounded hover:bg-gray-100"
               >
                 취소
               </button>
               <button
                 type="submit"
                 disabled={isPending}
                 className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 disabled:opacity-50"
               >
                 {isPending ? '생성 중...' : '생성'}
               </button>
             </div>
           </form>
         </div>
       </div>
     );
   }
   ```

### 4-3 에이전트 상세 보기 모달 구현
**목표**: 선택된 에이전트의 상세 정보 및 관리 기능 제공

**세부 작업**:

1. **AgentDetailModal 컴포넌트** (`components/admin/agents/AgentDetailModal.tsx`)
   - 기본 정보: 이름, 설명, 생성일, API Key (마스크)
   - 위임 상태: 현재 위임 사용자 목록 테이블, "위임 해제" 버튼
   - delegate scope 관리: 현재 scope 목록, "추가" / "제거" 버튼
   - 킬스위치: "차단" 버튼 (모달 팝업)
   - API Key 재발급: "새 키 생성" 버튼

   ```typescript
   // components/admin/agents/AgentDetailModal.tsx
   'use client';
   
   import { useState } from 'react';
   import { useQuery, useMutation } from '@tanstack/react-query';
   import { agentApi } from '@/lib/api/agentApi';
   import toast from 'react-hot-toast';
   
   interface AgentDetailModalProps {
     agentId: string;
     onClose: () => void;
   }
   
   export default function AgentDetailModal({ agentId, onClose }: AgentDetailModalProps) {
     const [showBlockModal, setShowBlockModal] = useState(false);
     const [blockDuration, setBlockDuration] = useState<'1h' | '24h' | 'permanent'>('1h');
     const [autoRejectProposals, setAutoRejectProposals] = useState(false);
     
     const { data: agent, isLoading } = useQuery({
       queryKey: ['agent', agentId],
       queryFn: async () => {
         const response = await agentApi.getAgent(agentId);
         return response.agent;
       }
     });
     
     const blockMutation = useMutation({
       mutationFn: async () => {
         await agentApi.blockAgent(agentId, {
           duration: blockDuration,
           autoRejectProposals
         });
       },
       onSuccess: () => {
         toast.success('에이전트 차단 완료');
         setShowBlockModal(false);
       }
     });
     
     const regenerateKeyMutation = useMutation({
       mutationFn: async () => {
         const response = await agentApi.regenerateApiKey(agentId);
         return response;
       },
       onSuccess: (data) => {
         toast.success('새 API Key 생성 완료');
         // 새 Key 표시 모달 또는 복사 기능
       }
     });
     
     if (isLoading) return null;
     if (!agent) return null;
     
     return (
       <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
         <div className="bg-white rounded-lg p-6 w-full max-w-2xl max-h-[90vh] overflow-y-auto">
           <div className="flex justify-between items-center mb-4">
             <h2 className="text-xl font-bold">에이전트 상세</h2>
             <button onClick={onClose} className="text-2xl">×</button>
           </div>
           
           {/* 기본 정보 섹션 */}
           <section className="mb-6 border-b pb-4">
             <h3 className="text-lg font-semibold mb-3">기본 정보</h3>
             <div className="grid grid-cols-2 gap-4">
               <div>
                 <label className="text-sm text-gray-600">이름</label>
                 <p className="font-medium">{agent.name}</p>
               </div>
               <div>
                 <label className="text-sm text-gray-600">상태</label>
                 <p className={`font-medium ${agent.isBlocked ? 'text-red-600' : 'text-green-600'}`}>
                   {agent.isBlocked ? '차단됨' : '활성'}
                 </p>
               </div>
               <div className="col-span-2">
                 <label className="text-sm text-gray-600">설명</label>
                 <p>{agent.description || '-'}</p>
               </div>
               <div>
                 <label className="text-sm text-gray-600">생성일</label>
                 <p>{new Date(agent.createdAt).toLocaleString('ko-KR')}</p>
               </div>
               <div>
                 <label className="text-sm text-gray-600">API Key</label>
                 <p className="font-mono text-sm">{agent.apiKeyPrefix}****</p>
               </div>
             </div>
           </section>
           
           {/* 위임 상태 섹션 */}
           <section className="mb-6 border-b pb-4">
             <h3 className="text-lg font-semibold mb-3">위임 상태</h3>
             {agent.delegations && agent.delegations.length > 0 ? (
               <div className="space-y-2">
                 {agent.delegations.map((deleg: any) => (
                   <div key={deleg.id} className="flex justify-between items-center p-2 bg-gray-50 rounded">
                     <span>{deleg.delegatedUserName}</span>
                     <button
                       onClick={() => {}}
                       className="text-sm text-red-600 hover:underline"
                     >
                       위임 해제
                     </button>
                   </div>
                 ))}
               </div>
             ) : (
               <p className="text-gray-500">위임 사용자 없음</p>
             )}
           </section>
           
           {/* Delegate Scope 섹션 */}
           <section className="mb-6 border-b pb-4">
             <h3 className="text-lg font-semibold mb-3">대행 권한 범위 (Scope)</h3>
             {agent.delegateScopes && agent.delegateScopes.length > 0 ? (
               <div className="space-y-2">
                 {agent.delegateScopes.map((scope: any) => (
                   <div key={scope.id} className="flex justify-between items-center p-2 bg-gray-50 rounded">
                     <span>{scope.name}</span>
                     <button
                       onClick={() => {}}
                       className="text-sm text-red-600 hover:underline"
                     >
                       제거
                     </button>
                   </div>
                 ))}
               </div>
             ) : (
               <p className="text-gray-500">권한 범위 없음</p>
             )}
             <button className="mt-3 text-sm text-blue-600 hover:underline">
               + 범위 추가
             </button>
           </section>
           
           {/* 킬스위치 섹션 */}
           <section className="mb-6 border-b pb-4">
             <h3 className="text-lg font-semibold mb-3">킬스위치 (차단)</h3>
             {agent.blockInfo ? (
               <div className="p-3 bg-red-50 border border-red-200 rounded">
                 <p className="text-sm"><strong>차단 시간:</strong> {agent.blockInfo.duration}</p>
                 <p className="text-sm"><strong>차단일:</strong> {new Date(agent.blockInfo.blockedAt).toLocaleString('ko-KR')}</p>
                 <p className="text-sm"><strong>자동 거절:</strong> {agent.blockInfo.autoRejectProposals ? '예' : '아니오'}</p>
               </div>
             ) : (
               <p className="text-gray-500 text-sm">현재 차단되지 않음</p>
             )}
             <button
               onClick={() => setShowBlockModal(true)}
               className="mt-3 px-3 py-1 text-sm bg-red-600 text-white rounded hover:bg-red-700"
             >
               {agent.isBlocked ? '차단 해제' : '에이전트 차단'}
             </button>
           </section>
           
           {/* API Key 재발급 섹션 */}
           <section className="mb-6">
             <h3 className="text-lg font-semibold mb-3">API Key 관리</h3>
             <button
               onClick={() => regenerateKeyMutation.mutate()}
               disabled={regenerateKeyMutation.isPending}
               className="px-4 py-2 bg-orange-600 text-white rounded hover:bg-orange-700 disabled:opacity-50"
             >
               {regenerateKeyMutation.isPending ? '생성 중...' : '새 API Key 생성'}
             </button>
             <p className="text-xs text-gray-500 mt-2">기존 Key는 즉시 무효화됩니다</p>
           </section>
           
           {/* 차단 모달 */}
           {showBlockModal && (
             <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-60">
               <div className="bg-white rounded-lg p-4 max-w-sm">
                 <h3 className="font-bold mb-3">에이전트 차단</h3>
                 <div className="space-y-3 mb-4">
                   <div>
                     <label className="block text-sm font-medium mb-2">차단 기간</label>
                     <select
                       value={blockDuration}
                       onChange={(e) => setBlockDuration(e.target.value as any)}
                       className="w-full border rounded px-3 py-2"
                     >
                       <option value="1h">1시간</option>
                       <option value="24h">24시간</option>
                       <option value="permanent">영구</option>
                     </select>
                   </div>
                   <label className="flex items-center">
                     <input
                       type="checkbox"
                       checked={autoRejectProposals}
                       onChange={(e) => setAutoRejectProposals(e.target.checked)}
                       className="mr-2"
                     />
                     <span className="text-sm">대기 중인 제안 자동 거절</span>
                   </label>
                 </div>
                 <div className="flex gap-2 justify-end">
                   <button
                     onClick={() => setShowBlockModal(false)}
                     className="px-3 py-1 border rounded hover:bg-gray-100"
                   >
                     취소
                   </button>
                   <button
                     onClick={() => blockMutation.mutate()}
                     disabled={blockMutation.isPending}
                     className="px-3 py-1 bg-red-600 text-white rounded hover:bg-red-700 disabled:opacity-50"
                   >
                     {blockMutation.isPending ? '처리 중...' : '차단'}
                   </button>
                 </div>
               </div>
             </div>
           )}
         </div>
       </div>
     );
   }
   ```

### 4-4 API 통합 (agentApi.ts)
**목표**: 백엔드 API와의 통신

**세부 작업**:

1. **agentApi.ts 확장** (`lib/api/agentApi.ts`)
   - `listAgents()`: GET `/api/v1/agents`
   - `getAgent(id)`: GET `/api/v1/agents/{id}`
   - `createAgent(payload)`: POST `/api/v1/agents`
   - `blockAgent(id, options)`: PATCH `/api/v1/agents/{id}/block`
   - `regenerateApiKey(id)`: POST `/api/v1/agents/{id}/api-keys`
   - unwrapEnvelope 래퍼 적용

   ```typescript
   // lib/api/agentApi.ts
   import { apiClient } from '@/lib/api/client';
   import { unwrapEnvelope } from '@/lib/api/envelope';
   
   export const agentApi = {
     async listAgents() {
       const response = await apiClient.get('/api/v1/agents');
       return unwrapEnvelope(response.data);
     },
     
     async getAgent(id: string) {
       const response = await apiClient.get(`/api/v1/agents/${id}`);
       return unwrapEnvelope(response.data);
     },
     
     async createAgent(payload: {
       name: string;
       description?: string;
       delegateScopeId?: string;
     }) {
       const response = await apiClient.post('/api/v1/agents', payload);
       return unwrapEnvelope(response.data);
     },
     
     async blockAgent(id: string, options: {
       duration: '1h' | '24h' | 'permanent';
       autoRejectProposals: boolean;
     }) {
       const response = await apiClient.patch(`/api/v1/agents/${id}/block`, options);
       return unwrapEnvelope(response.data);
     },
     
     async regenerateApiKey(id: string) {
       const response = await apiClient.post(`/api/v1/agents/${id}/api-keys`);
       return unwrapEnvelope(response.data);
     }
   };
   ```

### 4-5 Zustand agentStore 구현
**목표**: 클라이언트 상태 관리

**세부 작업**:

1. **agentStore.ts 작성** (`store/agentStore.ts`)
   - agents 리스트 상태
   - 선택된 에이전트 상태
   - CRUD 메서드 (addAgent, updateAgent, removeAgent)
   - 필터링 헬퍼 메서드

   ```typescript
   // store/agentStore.ts
   import { create } from 'zustand';
   
   interface Agent {
     id: string;
     name: string;
     description?: string;
     isBlocked: boolean;
     blockInfo?: {
       duration: string;
       blockedAt: string;
       autoRejectProposals: boolean;
     };
     delegations: any[];
     delegateScopes: any[];
     apiKeyPrefix: string;
     lastActivityAt?: string;
     createdAt: string;
   }
   
   interface AgentStore {
     agents: Agent[];
     selectedAgent: Agent | null;
     setAgents: (agents: Agent[]) => void;
     addAgent: (agent: Agent) => void;
     updateAgent: (id: string, updates: Partial<Agent>) => void;
     removeAgent: (id: string) => void;
     setSelectedAgent: (agent: Agent | null) => void;
     getBlockedAgents: () => Agent[];
   }
   
   export const useAgentStore = create<AgentStore>((set, get) => ({
     agents: [],
     selectedAgent: null,
     
     setAgents: (agents) => set({ agents }),
     
     addAgent: (agent) => set((state) => ({
       agents: [...state.agents, agent]
     })),
     
     updateAgent: (id, updates) => set((state) => ({
       agents: state.agents.map((a) => a.id === id ? { ...a, ...updates } : a),
       selectedAgent: state.selectedAgent?.id === id 
         ? { ...state.selectedAgent, ...updates } 
         : state.selectedAgent
     })),
     
     removeAgent: (id) => set((state) => ({
       agents: state.agents.filter((a) => a.id !== id),
       selectedAgent: state.selectedAgent?.id === id ? null : state.selectedAgent
     })),
     
     setSelectedAgent: (agent) => set({ selectedAgent: agent }),
     
     getBlockedAgents: () => get().agents.filter((a) => a.isBlocked)
   }));
   ```

### 4-6 React Testing Library 단위 테스트
**목표**: 주요 컴포넌트 테스트

**세부 작업**:

1. **AgentDetailModal.test.tsx** (`__tests__/components/admin/agents/AgentDetailModal.test.tsx`)
   - 에이전트 데이터 로드 및 표시 확인
   - 킬스위치 버튼 클릭 → 모달 열기
   - API Key 재발급 버튼 동작
   - 위임 해제 기능

   ```typescript
   // __tests__/components/admin/agents/AgentDetailModal.test.tsx
   import { render, screen, fireEvent, waitFor } from '@testing-library/react';
   import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
   import AgentDetailModal from '@/components/admin/agents/AgentDetailModal';
   import * as agentApi from '@/lib/api/agentApi';
   
   jest.mock('@/lib/api/agentApi');
   
   const queryClient = new QueryClient();
   
   describe('AgentDetailModal', () => {
     it('에이전트 정보를 로드하고 표시한다', async () => {
       const mockAgent = {
         id: '1',
         name: 'Test Agent',
         description: 'Test Description',
         isBlocked: false,
         delegations: [],
         delegateScopes: [],
         apiKeyPrefix: 'sk_live_abc',
         createdAt: new Date().toISOString()
       };
       
       (agentApi.agentApi.getAgent as jest.Mock).mockResolvedValue({
         agent: mockAgent
       });
       
       render(
         <QueryClientProvider client={queryClient}>
           <AgentDetailModal agentId="1" onClose={() => {}} />
         </QueryClientProvider>
       );
       
       await waitFor(() => {
         expect(screen.getByText('Test Agent')).toBeInTheDocument();
       });
     });
     
     it('킬스위치 버튼 클릭 시 모달을 표시한다', async () => {
       const mockAgent = {
         id: '1',
         name: 'Test Agent',
         isBlocked: false,
         delegations: [],
         delegateScopes: [],
         apiKeyPrefix: 'sk_live_abc',
         createdAt: new Date().toISOString()
       };
       
       (agentApi.agentApi.getAgent as jest.Mock).mockResolvedValue({
         agent: mockAgent
       });
       
       render(
         <QueryClientProvider client={queryClient}>
           <AgentDetailModal agentId="1" onClose={() => {}} />
         </QueryClientProvider>
       );
       
       await waitFor(() => {
         const blockButton = screen.getByText('에이전트 차단');
         fireEvent.click(blockButton);
       });
       
       expect(screen.getByText('에이전트 차단')).toBeInTheDocument();
     });
     
     it('새 API Key 생성 버튼 클릭 시 뮤테이션을 실행한다', async () => {
       const mockAgent = {
         id: '1',
         name: 'Test Agent',
         isBlocked: false,
         delegations: [],
         delegateScopes: [],
         apiKeyPrefix: 'sk_live_abc',
         createdAt: new Date().toISOString()
       };
       
       (agentApi.agentApi.getAgent as jest.Mock).mockResolvedValue({
         agent: mockAgent
       });
       
       (agentApi.agentApi.regenerateApiKey as jest.Mock).mockResolvedValue({
         apiKey: 'sk_new_xyz'
       });
       
       render(
         <QueryClientProvider client={queryClient}>
           <AgentDetailModal agentId="1" onClose={() => {}} />
         </QueryClientProvider>
       );
       
       await waitFor(() => {
         const regenerateButton = screen.getByText('새 API Key 생성');
         fireEvent.click(regenerateButton);
       });
       
       await waitFor(() => {
         expect(agentApi.agentApi.regenerateApiKey).toHaveBeenCalledWith('1');
       });
     });
   });
   ```

2. **CreateAgentModal.test.tsx** (`__tests__/components/admin/agents/CreateAgentModal.test.tsx`)
   - 폼 입력 및 유효성 검사
   - 생성 버튼 클릭 → API 호출
   - 성공/실패 토스트 메시지

## 5. 산출물

- `/app/admin/agents/page.tsx` - 에이전트 목록 페이지
- `/components/admin/agents/CreateAgentModal.tsx` - 에이전트 생성 모달
- `/components/admin/agents/AgentDetailModal.tsx` - 에이전트 상세 보기 모달
- `/lib/api/agentApi.ts` - API 클라이언트 (확장)
- `/store/agentStore.ts` - Zustand 상태 관리
- `/__tests__/components/admin/agents/` - 단위 테스트 파일 3개 이상
- `docs/개발문서/S2/mcp_integration_guide.md` - **MCP 통합 가이드** ← *Phase 4 DOCS-001 이월*

  > 외부 개발자(LangChain, n8n 등)가 Mimir를 MCP 도구로 연결할 때 참조하는 가이드. 이 Task에서 API Key 재발급 UI를 완성하는 시점에 함께 작성하는 것이 맥락상 자연스럽다.  
  > 필수 포함 내용: MCP 엔드포인트 7종, 인증 방법(API Key + `expires_at` 필수), Tool Schema 3종, Python/TypeScript 연동 예시.

## 6. 완료 기준

- [ ] `/admin/agents` 페이지 완성 (목록, 정렬, 페이지네이션)
- [ ] "에이전트 생성" 모달 완성 (유효성 검사 포함)
- [ ] 에이전트 상세 보기 모달 완성 (6개 섹션 모두 구현)
- [ ] 킬스위치 기능 완성 (3가지 차단 기간, 자동 거절 옵션)
- [ ] API Key 재발급 기능 완성
- [ ] agentApi.ts 모든 메서드 구현 및 unwrapEnvelope 적용
- [ ] agentStore 모든 메서드 구현
- [ ] React Testing Library 테스트 커버리지 ≥ 80%
- [ ] 모든 UI 성분 반응형 설계 (데스크탑/테블릿/모바일)
- [ ] **MCP 통합 가이드 작성 완료** (`mcp_integration_guide.md`) — 엔드포인트 7종·인증·Tool Schema·예시 코드 포함
- [ ] 검수 보고서 작성 완료
- [ ] 보안 취약점 검사 보고서 작성 완료

## 7. 작업 지침

### 7-1 S2 원칙 준수
- **원칙 ①~④**: 에이전트 관리도 사용자 관리처럼 generic 로직으로 구현
- **원칙 ⑤ 에이전트 동등**: UI에서 에이전트 목록, 차단, API Key 관리를 사용자와 동등하게 제공
- **원칙 ⑦ 폐쇄망**: 모든 차트/데이터는 로컬 API에서만 로드, 외부 CDN 사용 금지

### 7-2 코딩 가이드
- **타입 안전성**: 모든 props, API 응답에 TypeScript 인터페이스 정의
- **에러 처리**: try-catch 및 React Query 에러 핸들링 필수
- **접근성**: 모든 버튼/입력에 aria-label, 색상 대비 WCAG AA 이상
- **성능**: 
  - 모달은 React.memo로 메모이제이션
  - 리스트는 virtualization 라이브러리 고려 (1000개 이상 항목 시)
  - 이미지는 Next.js Image 컴포넌트 사용

### 7-3 UI/UX 가이드
- **일관성**: AdminTable, 모달 스타일을 전체 관리 콘솔과 통일
- **피드백**: 모든 비동기 작업에 로딩 상태, 성공/실패 토스트
- **확인 대화**: 에이전트 삭제, 킬스위치, API Key 재발급은 반드시 확인 모달 표시

### 7-4 테스트 가이드
- **단위 테스트**: React Testing Library (react-test-renderer 대신)
- **mocking**: jest.mock으로 agentApi, useQuery 모킹
- **에지 케이스**:
  - 빈 위임 사용자 목록
  - 차단된 에이전트 상태
  - API 에러 시나리오

### 7-5 성능 최적화
- **쿼리 캐싱**: React Query staleTime 5분으로 설정
- **배치 업데이트**: 여러 에이전트 선택 시 batch API 고려
- **디바운싱**: 목록 검색/필터는 300ms 디바운싱

### 7-6 보안 체크리스트
- API Key는 절대 콘솔에 로깅하지 않기
- 차단 사유, 피드백은 감사 로그에 기록
- 킬스위치 해제는 관리자 전용 (role 기반 ACL)
- 위임 사용자 정보(이름, 이메일)는 표시하되, 민감 정보는 마스킹

### 7-7 배포 전 체크리스트
- [ ] 로컬 개발 서버에서 모든 기능 동작 확인
- [ ] 프로덕션 API 엔드포인트 확인
- [ ] 환경 변수 설정 (API_BASE_URL, 토큰 등)
- [ ] 에러 로깅 (Sentry 또는 로컬 로거) 설정
- [ ] 라우트 보호 (인증, 권한 확인) 구현

---

**예상 소요 시간**: 40시간 (분석 8시간 + 개발 24시간 + 테스트 8시간)

**담당자 검수**: 2시간 (동작 확인, 피드백 반영)
