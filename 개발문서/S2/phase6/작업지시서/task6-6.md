# Task 6-6: Scope Profile 관리 + ACL 필터 시각 빌더

## 1. 작업 목적
Mimir의 핵심 권한 관리 기능인 Scope Profile을 관리자가 GUI로 구성할 수 있는 관리 콘솔을 구축합니다. S2 원칙 ⑥(접근 범위 하드코딩 금지)를 실현하기 위해, 코드 수정 없이 FilterExpression을 드래그&드롭 및 JSON 편집으로 동적으로 구성할 수 있는 노코드 ACL 필터 시각 빌더를 포함합니다.

## 2. 작업 범위

### 포함 사항
- `/admin/scope-profiles` 페이지
- Scope Profile 목록 조회 (AdminTable: 이름, 설명, scope 개수, API Key 수)
- Scope Profile 생성 모달 (이름, 설명)
- Scope Profile 상세 편집 페이지 (메타데이터, Scope 테이블)
- **ACL 필터 시각 빌더 (FilterExpressionBuilder)**
  - 필드 드롭다운 (organization_id, team_id, visibility, classification)
  - 연산자 드롭다운 (eq, neq, in, not_in, contains)
  - 값 입력 또는 $ctx 변수 선택
  - AND/OR 논리 연산 tree (드래그&드롭)
  - JSON 텍스트 편집 모드 토글
  - 프리뷰 ("이 필터는 문서 N건 중 M건을 허용합니다")
- Scope Profile 삭제 (사용 중 API Key 있으면 경고)
- API 통합 (scopeProfileApi.ts unwrapEnvelope)
- React Testing Library 단위 테스트

### 제외 사항
- 에이전트 관리 (Task 6-5)
- 에이전트 제안 큐 (Task 6-7)
- 에이전트 활동 대시보드 (Task 6-8)
- FilterExpression 백엔드 평가 (이미 Phase 4 완료)
- 실시간 필터 미리보기 (기술적 복잡도, 추후 Phase)

## 3. 선행 조건
- Phase 4 완료: Scope Profile, FilterExpression DB 모델
- Phase 4 완료: FilterExpression 검증 및 평가 로직
- 백엔드 API 문서: `/api/v1/scope-profiles` (GET, POST, PATCH, DELETE), `/api/v1/scope-profiles/{id}/preview` (POST)
- Next.js 16.2.2, React 19, TypeScript, Zustand, TanStack Table, react-dnd 설치 완료
- react-hot-toast, recharts 등 UI 라이브러리 준비

## 4. 주요 작업 항목

### 4-1 Scope Profile 목록 페이지
**목표**: `/admin/scope-profiles` 페이지에서 전체 Scope Profile 조회 및 관리

**세부 작업**:

1. **페이지 컴포넌트 생성** (`app/admin/scope-profiles/page.tsx`)
   - 헤더: "Scope Profile 관리" 제목, "생성" 버튼
   - AdminTable 컴포넌트 (이름, 설명, scope 개수, API Key 수)
   - 행 클릭 → 상세 편집 페이지로 이동 또는 모달 열기

   ```typescript
   // app/admin/scope-profiles/page.tsx
   'use client';
   
   import { useState } from 'react';
   import { useQuery } from '@tanstack/react-query';
   import AdminTable from '@/components/admin/AdminTable';
   import CreateProfileModal from '@/components/admin/scope-profiles/CreateProfileModal';
   import { scopeProfileApi } from '@/lib/api/scopeProfileApi';
   import Link from 'next/link';
   
   export default function ScopeProfilesPage() {
     const [showCreateModal, setShowCreateModal] = useState(false);
     
     const { data, isLoading, error } = useQuery({
       queryKey: ['scope-profiles'],
       queryFn: async () => {
         const response = await scopeProfileApi.listProfiles();
         return response;
       }
     });
     
     const columns = [
       { id: 'name', header: '이름', accessorKey: 'name' },
       { id: 'description', header: '설명', accessorKey: 'description' },
       { 
         id: 'scopeCount', 
         header: 'Scope 개수', 
         cell: (row: any) => row.original.scopes?.length ?? 0 
       },
       { 
         id: 'apiKeyCount', 
         header: 'API Key 수', 
         cell: (row: any) => row.original.apiKeys?.length ?? 0 
       },
       {
         id: 'actions',
         header: '작업',
         cell: (row: any) => (
           <Link 
             href={`/admin/scope-profiles/${row.original.id}`}
             className="text-blue-600 hover:underline"
           >
             편집
           </Link>
         )
       }
     ];
     
     if (isLoading) return <div className="p-6">로딩 중...</div>;
     if (error) return <div className="p-6 text-red-600">에러 발생</div>;
     
     return (
       <div className="p-6">
         <div className="flex justify-between items-center mb-6">
           <h1 className="text-2xl font-bold">Scope Profile 관리</h1>
           <button 
             onClick={() => setShowCreateModal(true)}
             className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
           >
             생성
           </button>
         </div>
         
         <AdminTable 
           columns={columns} 
           data={data?.profiles ?? []} 
         />
         
         {showCreateModal && (
           <CreateProfileModal onClose={() => setShowCreateModal(false)} />
         )}
       </div>
     );
   }
   ```

### 4-2 Scope Profile 생성 모달
**목표**: 새 Scope Profile 생성 폼

**세부 작업**:

1. **CreateProfileModal 컴포넌트** (`components/admin/scope-profiles/CreateProfileModal.tsx`)
   - 입력: 이름, 설명
   - POST `/api/v1/scope-profiles` 호출
   - 성공 시 목록 갱신 및 모달 닫기

   ```typescript
   // components/admin/scope-profiles/CreateProfileModal.tsx
   'use client';
   
   import { useState } from 'react';
   import { useMutation, useQueryClient } from '@tanstack/react-query';
   import { scopeProfileApi } from '@/lib/api/scopeProfileApi';
   import toast from 'react-hot-toast';
   
   interface CreateProfileModalProps {
     onClose: () => void;
   }
   
   export default function CreateProfileModal({ onClose }: CreateProfileModalProps) {
     const [formData, setFormData] = useState({
       name: '',
       description: ''
     });
     const queryClient = useQueryClient();
     
     const { mutate, isPending } = useMutation({
       mutationFn: async () => {
         const response = await scopeProfileApi.createProfile({
           name: formData.name,
           description: formData.description
         });
         return response;
       },
       onSuccess: () => {
         queryClient.invalidateQueries({ queryKey: ['scope-profiles'] });
         toast.success('Scope Profile 생성 완료');
         onClose();
       },
       onError: () => {
         toast.error('Scope Profile 생성 실패');
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
           <h2 className="text-xl font-bold mb-4">Scope Profile 생성</h2>
           
           <form onSubmit={handleSubmit} className="space-y-4">
             <div>
               <label className="block text-sm font-medium mb-1">이름 *</label>
               <input
                 type="text"
                 value={formData.name}
                 onChange={(e) => setFormData({...formData, name: e.target.value})}
                 className="w-full border rounded px-3 py-2"
                 placeholder="예: 팀 A 권한범위"
               />
             </div>
             
             <div>
               <label className="block text-sm font-medium mb-1">설명</label>
               <textarea
                 value={formData.description}
                 onChange={(e) => setFormData({...formData, description: e.target.value})}
                 className="w-full border rounded px-3 py-2"
                 placeholder="이 Profile의 목적 설명"
                 rows={3}
               />
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

### 4-3 Scope Profile 상세 편집 페이지
**목표**: 선택된 Profile의 메타데이터, Scope 테이블 관리

**세부 작업**:

1. **DetailPage 컴포넌트** (`app/admin/scope-profiles/[id]/page.tsx`)
   - 메타데이터: 이름, 설명 (편집 가능)
   - Scope 테이블: scope name + ACL filter 표시
   - "ACL 필터 편집" 버튼 → FilterExpressionBuilder 모달 열기
   - "Scope 추가" 버튼
   - "삭제" 버튼 (사용 중 API Key 경고)

   ```typescript
   // app/admin/scope-profiles/[id]/page.tsx
   'use client';
   
   import { useState } from 'react';
   import { useQuery, useMutation } from '@tanstack/react-query';
   import { scopeProfileApi } from '@/lib/api/scopeProfileApi';
   import FilterExpressionBuilderModal from '@/components/admin/scope-profiles/FilterExpressionBuilderModal';
   import toast from 'react-hot-toast';
   
   interface DetailPageProps {
     params: { id: string };
   }
   
   export default function DetailPage({ params }: DetailPageProps) {
     const [editingScopeId, setEditingScopeId] = useState<string | null>(null);
     const [name, setName] = useState('');
     const [description, setDescription] = useState('');
     const [showBuilderModal, setShowBuilderModal] = useState(false);
     
     const { data: profile, isLoading } = useQuery({
       queryKey: ['scope-profile', params.id],
       queryFn: async () => {
         const response = await scopeProfileApi.getProfile(params.id);
         return response.profile;
       },
       onSuccess: (data) => {
         setName(data.name);
         setDescription(data.description);
       }
     });
     
     const updateMutation = useMutation({
       mutationFn: async () => {
         await scopeProfileApi.updateProfile(params.id, {
           name,
           description
         });
       },
       onSuccess: () => {
         toast.success('저장 완료');
       }
     });
     
     const deleteMutation = useMutation({
       mutationFn: async () => {
         await scopeProfileApi.deleteProfile(params.id);
       },
       onSuccess: () => {
         toast.success('삭제 완료');
         window.location.href = '/admin/scope-profiles';
       }
     });
     
     const handleDelete = () => {
       if (profile?.apiKeys?.length > 0) {
         toast.error('사용 중인 API Key가 있어 삭제할 수 없습니다.');
         return;
       }
       if (confirm('정말로 삭제하시겠습니까?')) {
         deleteMutation.mutate();
       }
     };
     
     if (isLoading) return <div className="p-6">로딩 중...</div>;
     if (!profile) return <div className="p-6">Profile을 찾을 수 없습니다</div>;
     
     return (
       <div className="p-6">
         <div className="flex justify-between items-center mb-6">
           <h1 className="text-2xl font-bold">Scope Profile: {name}</h1>
           <button
             onClick={handleDelete}
             disabled={deleteMutation.isPending}
             className="px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700 disabled:opacity-50"
           >
             삭제
           </button>
         </div>
         
         {/* 메타데이터 섹션 */}
         <section className="bg-white rounded-lg p-6 mb-6 border">
           <h2 className="text-lg font-semibold mb-4">메타데이터</h2>
           <div className="space-y-4">
             <div>
               <label className="block text-sm font-medium mb-1">이름</label>
               <input
                 type="text"
                 value={name}
                 onChange={(e) => setName(e.target.value)}
                 className="w-full border rounded px-3 py-2"
               />
             </div>
             <div>
               <label className="block text-sm font-medium mb-1">설명</label>
               <textarea
                 value={description}
                 onChange={(e) => setDescription(e.target.value)}
                 className="w-full border rounded px-3 py-2"
                 rows={3}
               />
             </div>
             <div>
               <label className="block text-sm font-medium mb-1">생성일</label>
               <p className="text-gray-600">{new Date(profile.createdAt).toLocaleString('ko-KR')}</p>
             </div>
             <button
               onClick={() => updateMutation.mutate()}
               disabled={updateMutation.isPending}
               className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 disabled:opacity-50"
             >
               {updateMutation.isPending ? '저장 중...' : '저장'}
             </button>
           </div>
         </section>
         
         {/* Scope 테이블 섹션 */}
         <section className="bg-white rounded-lg p-6 border">
           <div className="flex justify-between items-center mb-4">
             <h2 className="text-lg font-semibold">Scope 목록</h2>
             <button
               onClick={() => setShowBuilderModal(true)}
               className="px-4 py-2 bg-green-600 text-white rounded hover:bg-green-700"
             >
               Scope 추가
             </button>
           </div>
           
           <table className="w-full border-collapse">
             <thead>
               <tr className="bg-gray-50">
                 <th className="border p-3 text-left">Scope 이름</th>
                 <th className="border p-3 text-left">ACL 필터</th>
                 <th className="border p-3 text-center">작업</th>
               </tr>
             </thead>
             <tbody>
               {profile.scopes?.map((scope: any) => (
                 <tr key={scope.id} className="border-b hover:bg-gray-50">
                   <td className="border p-3">{scope.name}</td>
                   <td className="border p-3 font-mono text-sm text-gray-600">
                     {JSON.stringify(scope.filterExpression).substring(0, 60)}...
                   </td>
                   <td className="border p-3 text-center space-x-2">
                     <button
                       onClick={() => setEditingScopeId(scope.id)}
                       className="text-blue-600 hover:underline"
                     >
                       편집
                     </button>
                     <button className="text-red-600 hover:underline">
                       제거
                     </button>
                   </td>
                 </tr>
               ))}
               {!profile.scopes || profile.scopes.length === 0 && (
                 <tr>
                   <td colSpan={3} className="border p-3 text-center text-gray-500">
                     Scope가 없습니다
                   </td>
                 </tr>
               )}
             </tbody>
           </table>
         </section>
         
         {/* FilterExpressionBuilder 모달 */}
         {showBuilderModal && (
           <FilterExpressionBuilderModal
             profileId={params.id}
             onClose={() => setShowBuilderModal(false)}
           />
         )}
       </div>
     );
   }
   ```

### 4-4 ACL 필터 시각 빌더 (핵심 컴포넌트)
**목표**: 노코드 FilterExpression 구성기

**세부 작업**:

1. **FilterExpressionBuilder 컴포넌트** (`components/admin/scope-profiles/FilterExpressionBuilder.tsx`)

   ```typescript
   // components/admin/scope-profiles/FilterExpressionBuilder.tsx
   'use client';
   
   import React, { useState } from 'react';
   import { useMutation } from '@tanstack/react-query';
   import { scopeProfileApi } from '@/lib/api/scopeProfileApi';
   import toast from 'react-hot-toast';
   
   export interface FilterNode {
     id: string;
     type: 'condition' | 'and' | 'or';
     field?: string;
     operator?: string;
     value?: any;
     isVariable?: boolean;
     children?: FilterNode[];
   }
   
   interface FilterExpressionBuilderProps {
     initialFilter?: FilterNode;
     profileId: string;
     scopeName: string;
     onSave: (filter: FilterNode) => void;
   }
   
   const AVAILABLE_FIELDS = [
     { id: 'organization_id', label: 'Organization ID', type: 'string' },
     { id: 'team_id', label: 'Team ID', type: 'string' },
     { id: 'visibility', label: 'Visibility', type: 'enum', values: ['public', 'private', 'internal'] },
     { id: 'classification', label: 'Classification', type: 'enum', values: ['public', 'confidential', 'secret'] }
   ];
   
   const OPERATORS = {
     string: ['eq', 'neq', 'in', 'not_in', 'contains'],
     enum: ['eq', 'neq', 'in', 'not_in']
   };
   
   export default function FilterExpressionBuilder({
     initialFilter,
     profileId,
     scopeName,
     onSave
   }: FilterExpressionBuilderProps) {
     const [filterTree, setFilterTree] = useState<FilterNode>(
       initialFilter || { id: '0', type: 'condition', field: '', operator: 'eq', value: '' }
     );
     const [jsonMode, setJsonMode] = useState(false);
     const [jsonText, setJsonText] = useState(JSON.stringify(initialFilter || {}, null, 2));
     const [previewCount, setPreviewCount] = useState<{ total: number; allowed: number } | null>(null);
     
     const previewMutation = useMutation({
       mutationFn: async () => {
         const response = await scopeProfileApi.previewFilter(profileId, {
           filterExpression: jsonMode ? JSON.parse(jsonText) : filterTree
         });
         return response;
       },
       onSuccess: (data) => {
         setPreviewCount(data.preview);
       }
     });
     
     const handleFieldChange = (nodeId: string, field: string) => {
       const newTree = updateNode(filterTree, nodeId, { field, operator: 'eq', value: '' });
       setFilterTree(newTree);
     };
     
     const handleOperatorChange = (nodeId: string, operator: string) => {
       const newTree = updateNode(filterTree, nodeId, { operator });
       setFilterTree(newTree);
     };
     
     const handleValueChange = (nodeId: string, value: any, isVariable: boolean = false) => {
       const newTree = updateNode(filterTree, nodeId, { value, isVariable });
       setFilterTree(newTree);
     };
     
     const handleAddCondition = (nodeId: string) => {
       const newTree = addCondition(filterTree, nodeId);
       setFilterTree(newTree);
     };
     
     const handleRemoveNode = (nodeId: string) => {
       const newTree = removeNode(filterTree, nodeId);
       setFilterTree(newTree);
     };
     
     const updateNode = (node: FilterNode, nodeId: string, updates: Partial<FilterNode>): FilterNode => {
       if (node.id === nodeId) {
         return { ...node, ...updates };
       }
       if (node.children) {
         return {
           ...node,
           children: node.children.map(child => updateNode(child, nodeId, updates))
         };
       }
       return node;
     };
     
     const addCondition = (node: FilterNode, nodeId: string): FilterNode => {
       if (node.id === nodeId) {
         const newChild: FilterNode = {
           id: Math.random().toString(),
           type: 'condition',
           field: '',
           operator: 'eq',
           value: ''
         };
         if (node.type === 'condition') {
           return {
             type: 'and',
             id: Math.random().toString(),
             children: [node, newChild]
           };
         }
         return {
           ...node,
           children: [...(node.children || []), newChild]
         };
       }
       if (node.children) {
         return {
           ...node,
           children: node.children.map(child => addCondition(child, nodeId))
         };
       }
       return node;
     };
     
     const removeNode = (node: FilterNode, nodeId: string): FilterNode => {
       if (node.children) {
         const filtered = node.children.filter(child => child.id !== nodeId);
         if (filtered.length === 0) {
           return { id: '0', type: 'condition', field: '', operator: 'eq', value: '' };
         }
         if (filtered.length === 1) {
           return filtered[0];
         }
         return { ...node, children: filtered };
       }
       return node;
     };
     
     const handleSave = () => {
       try {
         const filterToSave = jsonMode ? JSON.parse(jsonText) : filterTree;
         onSave(filterToSave);
         toast.success('필터 저장 완료');
       } catch (e) {
         toast.error('필터 형식이 잘못되었습니다');
       }
     };
     
     const renderConditionNode = (node: FilterNode, depth: number = 0): React.ReactNode => {
       if (node.type === 'condition') {
         const selectedField = AVAILABLE_FIELDS.find(f => f.id === node.field);
         const operators = selectedField ? OPERATORS[selectedField.type as keyof typeof OPERATORS] || [] : [];
         
         return (
           <div key={node.id} className="border rounded-lg p-4 bg-blue-50 space-y-3" style={{ marginLeft: `${depth * 20}px` }}>
             <div className="grid grid-cols-4 gap-2">
               <select
                 value={node.field || ''}
                 onChange={(e) => handleFieldChange(node.id, e.target.value)}
                 className="border rounded px-2 py-1"
               >
                 <option value="">필드 선택</option>
                 {AVAILABLE_FIELDS.map(f => (
                   <option key={f.id} value={f.id}>{f.label}</option>
                 ))}
               </select>
               
               <select
                 value={node.operator || ''}
                 onChange={(e) => handleOperatorChange(node.id, e.target.value)}
                 className="border rounded px-2 py-1"
                 disabled={!node.field}
               >
                 <option value="">연산자</option>
                 {operators.map(op => (
                   <option key={op} value={op}>{op}</option>
                 ))}
               </select>
               
               <input
                 type="text"
                 value={node.isVariable ? '$ctx.' + node.value : node.value || ''}
                 onChange={(e) => {
                   const val = e.target.value;
                   const isVar = val.startsWith('$ctx.');
                   handleValueChange(node.id, isVar ? val.substring(5) : val, isVar);
                 }}
                 placeholder="값 또는 $ctx.변수"
                 className="border rounded px-2 py-1"
                 disabled={!node.operator}
               />
               
               <button
                 onClick={() => handleRemoveNode(node.id)}
                 className="text-red-600 hover:bg-red-100 px-2 py-1 rounded"
               >
                 제거
               </button>
             </div>
             
             <button
               onClick={() => handleAddCondition(node.id)}
               className="text-sm text-blue-600 hover:underline"
             >
               + 조건 추가
             </button>
           </div>
         );
       }
       
       if (node.type === 'and' || node.type === 'or') {
         return (
           <div key={node.id} className="border-2 border-gray-300 rounded-lg p-4 space-y-3" style={{ marginLeft: `${depth * 20}px` }}>
             <div className="text-sm font-semibold text-gray-700">
               {node.type === 'and' ? '모두 만족 (AND)' : '하나 이상 만족 (OR)'}
               <button
                 onClick={() => {
                   const newType = node.type === 'and' ? 'or' : 'and';
                   const newTree = updateNode(filterTree, node.id, { type: newType as any });
                   setFilterTree(newTree);
                 }}
                 className="ml-2 text-xs text-blue-600 hover:underline"
               >
                 변경
               </button>
             </div>
             <div className="space-y-3">
               {node.children?.map((child) => (
                 <div key={child.id}>
                   {renderConditionNode(child, depth + 1)}
                 </div>
               ))}
             </div>
           </div>
         );
       }
       
       return null;
     };
     
     return (
       <div className="space-y-6">
         {/* 모드 토글 */}
         <div className="flex gap-4">
           <button
             onClick={() => {
               setJsonMode(false);
               setJsonText(JSON.stringify(filterTree, null, 2));
             }}
             className={`px-4 py-2 rounded ${!jsonMode ? 'bg-blue-600 text-white' : 'bg-gray-200'}`}
           >
             시각 편집기
           </button>
           <button
             onClick={() => {
               setJsonMode(true);
               setJsonText(JSON.stringify(filterTree, null, 2));
             }}
             className={`px-4 py-2 rounded ${jsonMode ? 'bg-blue-600 text-white' : 'bg-gray-200'}`}
           >
             JSON 편집기
           </button>
         </div>
         
         {/* 시각 편집기 */}
         {!jsonMode && (
           <div className="bg-white border rounded-lg p-6 space-y-4">
             {renderConditionNode(filterTree)}
           </div>
         )}
         
         {/* JSON 편집기 */}
         {jsonMode && (
           <div className="bg-white border rounded-lg p-6">
             <textarea
               value={jsonText}
               onChange={(e) => setJsonText(e.target.value)}
               className="w-full border rounded px-3 py-2 font-mono text-sm"
               rows={15}
             />
           </div>
         )}
         
         {/* 프리뷰 */}
         <div className="bg-white border rounded-lg p-6">
           <button
             onClick={() => previewMutation.mutate()}
             disabled={previewMutation.isPending}
             className="px-4 py-2 bg-gray-600 text-white rounded hover:bg-gray-700 disabled:opacity-50 mb-4"
           >
             {previewMutation.isPending ? '미리보기 중...' : '필터 미리보기'}
           </button>
           
           {previewCount && (
             <div className="p-4 bg-green-50 border border-green-200 rounded">
               <p className="text-sm font-semibold">
                 이 필터는 총 <strong>{previewCount.total}</strong>건 중 <strong>{previewCount.allowed}</strong>건을 허용합니다
               </p>
               <p className="text-xs text-gray-600 mt-1">
                 (허용률: {((previewCount.allowed / previewCount.total) * 100).toFixed(2)}%)
               </p>
             </div>
           )}
         </div>
         
         {/* 저장 버튼 */}
         <div className="flex gap-2">
           <button
             onClick={handleSave}
             className="px-6 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
           >
             저장
           </button>
         </div>
       </div>
     );
   }
   ```

2. **FilterExpressionBuilderModal** (`components/admin/scope-profiles/FilterExpressionBuilderModal.tsx`)
   - 모달 래퍼로 빌더 컴포넌트 감싸기
   - Scope 추가 또는 편집 처리

   ```typescript
   // components/admin/scope-profiles/FilterExpressionBuilderModal.tsx
   'use client';
   
   import { useState } from 'react';
   import { useMutation, useQueryClient } from '@tanstack/react-query';
   import FilterExpressionBuilder, { FilterNode } from './FilterExpressionBuilder';
   import { scopeProfileApi } from '@/lib/api/scopeProfileApi';
   import toast from 'react-hot-toast';
   
   interface FilterExpressionBuilderModalProps {
     profileId: string;
     existingScopeId?: string;
     onClose: () => void;
   }
   
   export default function FilterExpressionBuilderModal({
     profileId,
     existingScopeId,
     onClose
   }: FilterExpressionBuilderModalProps) {
     const [scopeName, setScopeName] = useState('');
     const [filter, setFilter] = useState<FilterNode | null>(null);
     const queryClient = useQueryClient();
     
     const saveMutation = useMutation({
       mutationFn: async () => {
         if (!scopeName.trim() || !filter) {
           throw new Error('Scope 이름과 필터를 입력하세요');
         }
         
         if (existingScopeId) {
           await scopeProfileApi.updateScope(profileId, existingScopeId, {
             name: scopeName,
             filterExpression: filter
           });
         } else {
           await scopeProfileApi.createScope(profileId, {
             name: scopeName,
             filterExpression: filter
           });
         }
       },
       onSuccess: () => {
         queryClient.invalidateQueries({ queryKey: ['scope-profile', profileId] });
         toast.success('Scope 저장 완료');
         onClose();
       },
       onError: (error) => {
         toast.error('Scope 저장 실패');
       }
     });
     
     return (
       <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
         <div className="bg-white rounded-lg p-6 w-full max-w-2xl max-h-[90vh] overflow-y-auto">
           <div className="flex justify-between items-center mb-4">
             <h2 className="text-xl font-bold">ACL 필터 빌더</h2>
             <button onClick={onClose} className="text-2xl">×</button>
           </div>
           
           {/* Scope 이름 입력 */}
           <div className="mb-6">
             <label className="block text-sm font-medium mb-2">Scope 이름</label>
             <input
               type="text"
               value={scopeName}
               onChange={(e) => setScopeName(e.target.value)}
               placeholder="예: 팀 A 문서"
               className="w-full border rounded px-3 py-2"
             />
           </div>
           
           {/* 필터 빌더 */}
           <FilterExpressionBuilder
             profileId={profileId}
             scopeName={scopeName}
             onSave={(f) => setFilter(f)}
           />
           
           {/* 작업 버튼 */}
           <div className="flex justify-end gap-2 mt-6">
             <button
               onClick={onClose}
               className="px-4 py-2 border rounded hover:bg-gray-100"
             >
               취소
             </button>
             <button
               onClick={() => saveMutation.mutate()}
               disabled={saveMutation.isPending || !scopeName.trim() || !filter}
               className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 disabled:opacity-50"
             >
               {saveMutation.isPending ? '저장 중...' : '저장'}
             </button>
           </div>
         </div>
       </div>
     );
   }
   ```

### 4-5 API 통합 (scopeProfileApi.ts)
**목표**: Scope Profile 및 필터 관련 API 호출

**세부 작업**:

1. **scopeProfileApi.ts 작성** (`lib/api/scopeProfileApi.ts`)

   ```typescript
   // lib/api/scopeProfileApi.ts
   import { apiClient } from '@/lib/api/client';
   import { unwrapEnvelope } from '@/lib/api/envelope';
   
   export const scopeProfileApi = {
     async listProfiles() {
       const response = await apiClient.get('/api/v1/scope-profiles');
       return unwrapEnvelope(response.data);
     },
     
     async getProfile(id: string) {
       const response = await apiClient.get(`/api/v1/scope-profiles/${id}`);
       return unwrapEnvelope(response.data);
     },
     
     async createProfile(payload: {
       name: string;
       description?: string;
     }) {
       const response = await apiClient.post('/api/v1/scope-profiles', payload);
       return unwrapEnvelope(response.data);
     },
     
     async updateProfile(id: string, payload: {
       name?: string;
       description?: string;
     }) {
       const response = await apiClient.patch(`/api/v1/scope-profiles/${id}`, payload);
       return unwrapEnvelope(response.data);
     },
     
     async deleteProfile(id: string) {
       const response = await apiClient.delete(`/api/v1/scope-profiles/${id}`);
       return unwrapEnvelope(response.data);
     },
     
     async createScope(profileId: string, payload: {
       name: string;
       filterExpression: any;
     }) {
       const response = await apiClient.post(
         `/api/v1/scope-profiles/${profileId}/scopes`,
         payload
       );
       return unwrapEnvelope(response.data);
     },
     
     async updateScope(profileId: string, scopeId: string, payload: {
       name?: string;
       filterExpression?: any;
     }) {
       const response = await apiClient.patch(
         `/api/v1/scope-profiles/${profileId}/scopes/${scopeId}`,
         payload
       );
       return unwrapEnvelope(response.data);
     },
     
     async previewFilter(profileId: string, payload: {
       filterExpression: any;
     }) {
       const response = await apiClient.post(
         `/api/v1/scope-profiles/${profileId}/preview`,
         payload
       );
       return unwrapEnvelope(response.data);
     }
   };
   ```

### 4-6 React Testing Library 단위 테스트

1. **FilterExpressionBuilder.test.tsx** (`__tests__/components/admin/scope-profiles/FilterExpressionBuilder.test.tsx`)
   - 필드 드롭다운 선택 → 연산자 활성화 확인
   - 조건 추가/제거 기능
   - AND/OR 변경
   - JSON 모드 토글

   ```typescript
   // __tests__/components/admin/scope-profiles/FilterExpressionBuilder.test.tsx
   import { render, screen, fireEvent, waitFor } from '@testing-library/react';
   import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
   import FilterExpressionBuilder from '@/components/admin/scope-profiles/FilterExpressionBuilder';
   
   const queryClient = new QueryClient();
   
   describe('FilterExpressionBuilder', () => {
     it('필드 선택 시 연산자가 활성화된다', async () => {
       render(
         <QueryClientProvider client={queryClient}>
           <FilterExpressionBuilder
             profileId="1"
             scopeName="test"
             onSave={() => {}}
           />
         </QueryClientProvider>
       );
       
       const fieldSelect = screen.getByRole('combobox', { name: /필드 선택/i });
       fireEvent.change(fieldSelect, { target: { value: 'organization_id' } });
       
       await waitFor(() => {
         const operatorSelect = screen.getByRole('combobox', { name: /연산자/i });
         expect(operatorSelect).not.toBeDisabled();
       });
     });
     
     it('JSON 모드 토글이 작동한다', () => {
       render(
         <QueryClientProvider client={queryClient}>
           <FilterExpressionBuilder
             profileId="1"
             scopeName="test"
             onSave={() => {}}
           />
         </QueryClientProvider>
       );
       
       const jsonButton = screen.getByRole('button', { name: /JSON 편집기/i });
       fireEvent.click(jsonButton);
       
       expect(screen.getByRole('textbox', { name: '' })).toBeInTheDocument();
     });
   });
   ```

## 5. 산출물

- `/app/admin/scope-profiles/page.tsx` - Scope Profile 목록 페이지
- `/app/admin/scope-profiles/[id]/page.tsx` - Scope Profile 상세 편집 페이지
- `/components/admin/scope-profiles/CreateProfileModal.tsx` - 생성 모달
- `/components/admin/scope-profiles/FilterExpressionBuilder.tsx` - 시각 빌더 (핵심)
- `/components/admin/scope-profiles/FilterExpressionBuilderModal.tsx` - 빌더 모달
- `/lib/api/scopeProfileApi.ts` - API 클라이언트
- `/__tests__/components/admin/scope-profiles/` - 단위 테스트 파일 3개 이상

## 6. 완료 기준

- [ ] `/admin/scope-profiles` 목록 페이지 완성
- [ ] Scope Profile 생성/편집 기능 완성
- [ ] FilterExpressionBuilder 시각 편집기 완성
  - [ ] 필드/연산자/값 입력 UI
  - [ ] AND/OR 논리 구성
  - [ ] 조건 추가/제거
- [ ] JSON 편집 모드 완성
- [ ] 필터 미리보기 기능 완성 (문서 N건 중 M건)
- [ ] Scope 추가/편집 기능 완성
- [ ] 삭제 시 API Key 사용 중 경고
- [ ] scopeProfileApi.ts 모든 메서드 구현 및 unwrapEnvelope 적용
- [ ] React Testing Library 테스트 커버리지 ≥ 80%
- [ ] 모든 UI 컴포넌트 반응형 설계
- [ ] 검수 보고서 작성 완료
- [ ] 보안 취약점 검사 보고서 작성 완료

## 7. 작업 지침

### 7-1 S2 원칙 준수
- **원칙 ① generic + config**: Scope Profile은 문서 타입별 특화 로직 없이 일반적인 필터 메커니즘으로 관리
- **원칙 ⑥ 접근 범위 하드코딩 금지**: ACL 필터는 UI로 동적으로 구성, 코드 수정 불필요
- **원칙 ⑦ 폐쇄망**: 필터 미리보기도 로컬 API에서만 평가

### 7-2 필터 빌더 UX
- **드래그&드롭**: react-dnd 또는 react-beautiful-dnd 사용 고려
- **조건 추가**: "+" 버튼으로 쉽게 복합 조건 구성
- **모드 전환**: 시각/JSON 전환 시 데이터 동기화 유지
- **프리뷰**: 백엔드 플랫 폼 API 호출로 실제 필터 결과 표시

### 7-3 API 설계
- FilterExpression 검증은 백엔드 책임
- 미리보기는 `/api/v1/scope-profiles/{id}/preview` POST 엔드포인트
- 응답: `{ preview: { total: N, allowed: M } }`

### 7-4 테스트 가이드
- FilterExpressionBuilder: 필드 선택 → 연산자 활성화 → 값 입력
- 모달: 저장 시 API 호출 확인
- 에지 케이스: 복합 AND/OR 구조, $ctx 변수 포함

### 7-5 보안 체크리스트
- FilterExpression JSON 주입 방지 (JSON.parse 안전성)
- 필터 미리보기는 감사 로그에 기록
- 삭제 시 API Key 사용 확인 필수

---

**예상 소요 시간**: 48시간 (분석 8시간 + FilterExpressionBuilder 개발 20시간 + 나머지 CRUD 12시간 + 테스트 8시간)
