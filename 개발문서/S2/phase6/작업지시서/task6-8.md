# Task 6-8: 에이전트 활동 대시보드 (차트, 통계, 이상 행동 알림)

## 1. 작업 목적
에이전트의 활동 패턴을 시각적으로 분석하고 모니터링할 수 있는 관리 대시보드를 구축합니다. 제안 수, 승인률, 소요 시간 추이를 시계열 차트로 표시하고, 에이전트별 상세 통계로 성능을 평가하며, 이상 행동(거절률 급증, 킬스위치 발동)을 강조 표시합니다.

## 2. 작업 범위

### 포함 사항
- `/admin/agent-dashboard` 페이지
- 시계열 요약 (recharts)
  - 에이전트별 제안 수 (BarChart, 30일)
  - 승인률 추이 (LineChart, %)
  - 평균 승인 소요 시간 (BarChart)
- 에이전트별 상세 통계 테이블
  - 총 제안, 승인, 거절, 승인률%, 거절 사유 상위 5개, 자주 제안 문서 타입
- 이상 행동 알림
  - 24시간 내 거절률 > 50% 에이전트 강조 (빨간 배지)
  - 킬스위치 발동 히스토리 (timeline)
- CSV Export (통계 데이터)
- React Query polling (30초)
- 단위 테스트

### 제외 사항
- 에이전트 관리 (Task 6-5)
- Scope Profile 관리 (Task 6-6)
- 에이전트 제안 큐 (Task 6-7)
- 머신러닝 기반 이상 탐지 (Phase 7)
- 커스텀 대시보드 저장 (Phase 7)

## 3. 선행 조건
- Phase 4 완료: Agent, Proposal, ProposalDecision 모델
- Phase 5 완료: 에이전트 감시 API (통계 조회)
- 백엔드 API 문서:
  - `/api/v1/agents/stats` (GET) - 전체 에이전트 통계
  - `/api/v1/agents/{id}/stats` (GET) - 개별 에이전트 통계
  - `/api/v1/agents/stats/timeline` (GET, startDate, endDate) - 시계열 데이터
  - `/api/v1/agents/{id}/block-history` (GET) - 킬스위치 히스토리
- Next.js 16.2.2, React 19, TypeScript, Zustand, recharts, react-csv 설치 완료

## 4. 주요 작업 항목

### 4-1 대시보드 페이지 구성
**목표**: `/admin/agent-dashboard` 페이지 레이아웃 설계

**세부 작업**:

1. **페이지 컴포넌트 생성** (`app/admin/agent-dashboard/page.tsx`)
   - 헤더: "에이전트 활동 대시보드" 제목, 날짜 범위 필터, Export 버튼
   - 4개 섹션: 시계열 차트(상단), 상세 통계(중단), 이상 행동 알림(중단), 킬스위치 히스토리(하단)

   ```typescript
   // app/admin/agent-dashboard/page.tsx
   'use client';
   
   import { useState } from 'react';
   import { useQuery } from '@tanstack/react-query';
   import { agentStatsApi } from '@/lib/api/agentStatsApi';
   import TimeSeriesCharts from '@/components/admin/dashboard/TimeSeriesCharts';
   import AgentStatsTable from '@/components/admin/dashboard/AgentStatsTable';
   import AnomalyAlerts from '@/components/admin/dashboard/AnomalyAlerts';
   import BlockHistory from '@/components/admin/dashboard/BlockHistory';
   import { CSVLink } from 'react-csv';
   import toast from 'react-hot-toast';
   
   interface DateRange {
     startDate: string;
     endDate: string;
   }
   
   export default function AgentDashboard() {
     const today = new Date();
     const thirtyDaysAgo = new Date(today.getTime() - 30 * 24 * 60 * 60 * 1000);
     
     const [dateRange, setDateRange] = useState<DateRange>({
       startDate: thirtyDaysAgo.toISOString().split('T')[0],
       endDate: today.toISOString().split('T')[0]
     });
     
     // 시계열 데이터
     const { data: timeSeriesData, isLoading: timeSeriesLoading } = useQuery({
       queryKey: ['agent-stats-timeline', dateRange],
       queryFn: async () => {
         const response = await agentStatsApi.getTimeline({
           startDate: dateRange.startDate,
           endDate: dateRange.endDate
         });
         return response;
       },
       refetchInterval: 30000 // 30초마다 갱신
     });
     
     // 에이전트별 통계
     const { data: agentStats, isLoading: statsLoading } = useQuery({
       queryKey: ['agent-stats', dateRange],
       queryFn: async () => {
         const response = await agentStatsApi.getAllAgentStats({
           startDate: dateRange.startDate,
           endDate: dateRange.endDate
         });
         return response;
       },
       refetchInterval: 30000
     });
     
     // 킬스위치 히스토리
     const { data: blockHistory, isLoading: blockHistoryLoading } = useQuery({
       queryKey: ['block-history', dateRange],
       queryFn: async () => {
         const response = await agentStatsApi.getBlockHistory({
           startDate: dateRange.startDate,
           endDate: dateRange.endDate
         });
         return response;
       },
       refetchInterval: 30000
     });
     
     // 이상 행동 감지
     const anomalies = agentStats?.agents
       ?.filter((agent: any) => {
         // 24시간 내 거절률 > 50%
         const recentRejections = agent.recentRejectRate > 0.5;
         return recentRejections;
       }) ?? [];
     
     // CSV Export 데이터
     const csvData = agentStats?.agents?.map((agent: any) => ({
       agentName: agent.name,
       totalProposals: agent.totalProposals,
       approved: agent.approvedProposals,
       rejected: agent.rejectedProposals,
       approvalRate: (agent.approvalRate * 100).toFixed(2) + '%',
       avgApprovalTime: agent.avgApprovalTime + 'ms',
       recentRejectRate: (agent.recentRejectRate * 100).toFixed(2) + '%',
       topRejectReasons: agent.topRejectReasons?.slice(0, 3).join(', ')
     })) ?? [];
     
     if (timeSeriesLoading || statsLoading) {
       return <div className="p-6">로딩 중...</div>;
     }
     
     return (
       <div className="p-6 space-y-6 bg-gray-50 min-h-screen">
         {/* 헤더 */}
         <div className="flex justify-between items-center">
           <h1 className="text-3xl font-bold">에이전트 활동 대시보드</h1>
           <div className="flex gap-4">
             <div className="flex gap-2">
               <div>
                 <label className="block text-sm font-medium mb-1">시작일</label>
                 <input
                   type="date"
                   value={dateRange.startDate}
                   onChange={(e) => setDateRange({...dateRange, startDate: e.target.value})}
                   className="border rounded px-3 py-2"
                 />
               </div>
               <div>
                 <label className="block text-sm font-medium mb-1">종료일</label>
                 <input
                   type="date"
                   value={dateRange.endDate}
                   onChange={(e) => setDateRange({...dateRange, endDate: e.target.value})}
                   className="border rounded px-3 py-2"
                 />
               </div>
             </div>
             
             {csvData.length > 0 && (
               <CSVLink
                 data={csvData}
                 filename={`agent-stats-${dateRange.startDate}-${dateRange.endDate}.csv`}
                 className="px-4 py-2 bg-green-600 text-white rounded hover:bg-green-700 h-fit"
               >
                 CSV Export
               </CSVLink>
             )}
           </div>
         </div>
         
         {/* 시계열 차트 섹션 */}
         {timeSeriesData && (
           <TimeSeriesCharts data={timeSeriesData} />
         )}
         
         {/* 이상 행동 알림 섹션 */}
         {anomalies.length > 0 && (
           <AnomalyAlerts anomalies={anomalies} />
         )}
         
         {/* 에이전트별 상세 통계 */}
         {agentStats && (
           <AgentStatsTable agents={agentStats.agents} />
         )}
         
         {/* 킬스위치 히스토리 */}
         {blockHistory && (
           <BlockHistory events={blockHistory.blockEvents} />
         )}
       </div>
     );
   }
   ```

### 4-2 시계열 차트 컴포넌트
**목표**: recharts를 사용한 3개 차트 표시

**세부 작업**:

1. **TimeSeriesCharts 컴포넌트** (`components/admin/dashboard/TimeSeriesCharts.tsx`)

   ```typescript
   // components/admin/dashboard/TimeSeriesCharts.tsx
   'use client';
   
   import React from 'react';
   import {
     BarChart,
     LineChart,
     Bar,
     Line,
     XAxis,
     YAxis,
     CartesianGrid,
     Tooltip,
     Legend,
     ResponsiveContainer
   } from 'recharts';
   
   interface TimeSeriesData {
     timeline: Array<{
       date: string;
       proposalCount: number;
       approvalRate: number;
       avgApprovalTime: number;
       agentMetrics: Array<{
         agentId: string;
         agentName: string;
         proposalCount: number;
         approvalRate: number;
         avgApprovalTime: number;
       }>;
     }>;
   }
   
   export default function TimeSeriesCharts({ data }: { data: TimeSeriesData }) {
     // 에이전트별 제안 수 준비
     const proposalCountData = data.timeline.map((day) => {
       const entry: any = { date: day.date };
       day.agentMetrics.forEach((metric) => {
         entry[metric.agentName] = metric.proposalCount;
       });
       return entry;
     });
     
     // 승인률 추이 준비
     const approvalRateData = data.timeline.map((day) => ({
       date: day.date,
       approvalRate: (day.approvalRate * 100).toFixed(2)
     }));
     
     // 소요 시간 준비
     const approvalTimeData = data.timeline.map((day) => ({
       date: day.date,
       avgTime: day.avgApprovalTime
     }));
     
     const agentNames = data.timeline[0]?.agentMetrics?.map((m) => m.agentName) ?? [];
     const colors = ['#3b82f6', '#ef4444', '#10b981', '#f59e0b', '#8b5cf6'];
     
     return (
       <div className="space-y-6">
         {/* 1. 에이전트별 제안 수 (BarChart) */}
         <div className="bg-white rounded-lg p-6 border shadow-sm">
           <h2 className="text-lg font-semibold mb-4">에이전트별 제안 수 (30일)</h2>
           <ResponsiveContainer width="100%" height={300}>
             <BarChart data={proposalCountData}>
               <CartesianGrid strokeDasharray="3 3" />
               <XAxis dataKey="date" />
               <YAxis />
               <Tooltip />
               <Legend />
               {agentNames.map((name, idx) => (
                 <Bar key={name} dataKey={name} fill={colors[idx % colors.length]} />
               ))}
             </BarChart>
           </ResponsiveContainer>
         </div>
         
         {/* 2. 승인률 추이 (LineChart) */}
         <div className="bg-white rounded-lg p-6 border shadow-sm">
           <h2 className="text-lg font-semibold mb-4">전체 승인률 추이 (%)</h2>
           <ResponsiveContainer width="100%" height={300}>
             <LineChart data={approvalRateData}>
               <CartesianGrid strokeDasharray="3 3" />
               <XAxis dataKey="date" />
               <YAxis domain={[0, 100]} />
               <Tooltip formatter={(value) => value + '%'} />
               <Legend />
               <Line 
                 type="monotone" 
                 dataKey="approvalRate" 
                 stroke="#3b82f6" 
                 dot={{ r: 4 }}
                 name="승인률"
               />
             </LineChart>
           </ResponsiveContainer>
         </div>
         
         {/* 3. 평균 승인 소요 시간 (BarChart) */}
         <div className="bg-white rounded-lg p-6 border shadow-sm">
           <h2 className="text-lg font-semibold mb-4">평균 승인 소요 시간 (밀리초)</h2>
           <ResponsiveContainer width="100%" height={300}>
             <BarChart data={approvalTimeData}>
               <CartesianGrid strokeDasharray="3 3" />
               <XAxis dataKey="date" />
               <YAxis />
               <Tooltip formatter={(value) => value + 'ms'} />
               <Legend />
               <Bar dataKey="avgTime" fill="#10b981" name="소요시간" />
             </BarChart>
           </ResponsiveContainer>
         </div>
       </div>
     );
   }
   ```

### 4-3 에이전트별 상세 통계 테이블
**목표**: 에이전트 성능 메트릭 표시

**세부 작업**:

1. **AgentStatsTable 컴포넌트** (`components/admin/dashboard/AgentStatsTable.tsx`)

   ```typescript
   // components/admin/dashboard/AgentStatsTable.tsx
   'use client';
   
   import React, { useState } from 'react';
   import AdminTable from '@/components/admin/AdminTable';
   
   interface AgentStatRow {
     id: string;
     name: string;
     totalProposals: number;
     approvedProposals: number;
     rejectedProposals: number;
     approvalRate: number;
     avgApprovalTime: number;
     topRejectReasons?: string[];
     topDocumentTypes?: string[];
     recentRejectRate: number;
   }
   
   export default function AgentStatsTable({ agents }: { agents: AgentStatRow[] }) {
     const [expandedAgent, setExpandedAgent] = useState<string | null>(null);
     
     const columns = [
       { id: 'name', header: '에이전트명', accessorKey: 'name' },
       { 
         id: 'totalProposals', 
         header: '총 제안', 
         accessorKey: 'totalProposals',
         cell: (row: any) => row.original.totalProposals
       },
       { 
         id: 'approvedProposals', 
         header: '승인', 
         accessorKey: 'approvedProposals',
         cell: (row: any) => row.original.approvedProposals
       },
       { 
         id: 'rejectedProposals', 
         header: '거절', 
         accessorKey: 'rejectedProposals',
         cell: (row: any) => row.original.rejectedProposals
       },
       { 
         id: 'approvalRate', 
         header: '승인률', 
         cell: (row: any) => (
           <span className={`font-semibold ${
             row.original.approvalRate >= 0.8 ? 'text-green-600' :
             row.original.approvalRate >= 0.5 ? 'text-yellow-600' :
             'text-red-600'
           }`}>
             {(row.original.approvalRate * 100).toFixed(1)}%
           </span>
         )
       },
       { 
         id: 'avgApprovalTime', 
         header: '평균 소요시간', 
         cell: (row: any) => row.original.avgApprovalTime.toFixed(0) + 'ms'
       },
       {
         id: 'actions',
         header: '상세',
         cell: (row: any) => (
           <button
             onClick={() => setExpandedAgent(
               expandedAgent === row.original.id ? null : row.original.id
             )}
             className="text-blue-600 hover:underline text-sm"
           >
             {expandedAgent === row.original.id ? '숨기기' : '보기'}
           </button>
         )
       }
     ];
     
     return (
       <div className="bg-white rounded-lg border shadow-sm overflow-hidden">
         <div className="p-6 border-b">
           <h2 className="text-lg font-semibold">에이전트별 상세 통계</h2>
         </div>
         
         <AdminTable columns={columns} data={agents} />
         
         {/* 확장 정보 */}
         {expandedAgent && (
           <div className="border-t p-6 bg-gray-50">
             {agents
               .filter((a) => a.id === expandedAgent)
               .map((agent) => (
                 <div key={agent.id} className="space-y-4">
                   <div>
                     <h3 className="font-semibold mb-2">거절 사유 상위 5개</h3>
                     <ul className="list-disc list-inside space-y-1 text-sm">
                       {agent.topRejectReasons?.map((reason, idx) => (
                         <li key={idx}>{reason}</li>
                       ))}
                     </ul>
                   </div>
                   
                   <div>
                     <h3 className="font-semibold mb-2">자주 제안 문서 타입</h3>
                     <ul className="list-disc list-inside space-y-1 text-sm">
                       {agent.topDocumentTypes?.map((docType, idx) => (
                         <li key={idx}>{docType}</li>
                       ))}
                     </ul>
                   </div>
                 </div>
               ))}
           </div>
         )}
       </div>
     );
   }
   ```

### 4-4 이상 행동 알림 컴포넌트
**목표**: 거절률 급증 에이전트 강조

**세부 작업**:

1. **AnomalyAlerts 컴포넌트** (`components/admin/dashboard/AnomalyAlerts.tsx`)

   ```typescript
   // components/admin/dashboard/AnomalyAlerts.tsx
   'use client';
   
   import React from 'react';
   import Link from 'next/link';
   
   interface Anomaly {
     id: string;
     name: string;
     recentRejectRate: number;
     rejectionTrend: 'increasing' | 'stable' | 'decreasing';
   }
   
   export default function AnomalyAlerts({ anomalies }: { anomalies: Anomaly[] }) {
     if (anomalies.length === 0) {
       return null;
     }
     
     return (
       <div className="bg-white rounded-lg border shadow-sm border-red-200 bg-red-50">
         <div className="p-6">
           <div className="flex items-center gap-3 mb-4">
             <div className="w-2 h-2 bg-red-600 rounded-full animate-pulse"></div>
             <h2 className="text-lg font-semibold text-red-900">이상 행동 감지</h2>
           </div>
           
           <div className="space-y-3">
             {anomalies.map((anomaly) => (
               <div key={anomaly.id} className="flex justify-between items-start p-3 bg-white rounded border border-red-100">
                 <div>
                   <p className="font-semibold text-red-900">{anomaly.name}</p>
                   <p className="text-sm text-red-700">
                     24시간 내 거절률: {(anomaly.recentRejectRate * 100).toFixed(1)}%
                   </p>
                   <p className="text-xs text-red-600 mt-1">
                     추세: {
                       anomaly.rejectionTrend === 'increasing' ? '📈 상승 중' :
                       anomaly.rejectionTrend === 'decreasing' ? '📉 하강 중' :
                       '➡️ 안정적'
                     }
                   </p>
                 </div>
                 <Link 
                   href={`/admin/agents?agentId=${anomaly.id}`}
                   className="px-3 py-1 text-sm bg-red-600 text-white rounded hover:bg-red-700 whitespace-nowrap"
                 >
                   보기
                 </Link>
               </div>
             ))}
           </div>
         </div>
       </div>
     );
   }
   ```

### 4-5 킬스위치 히스토리 타임라인
**목차**: 에이전트 차단 이력 시각화

**세부 작업**:

1. **BlockHistory 컴포넌트** (`components/admin/dashboard/BlockHistory.tsx`)

   ```typescript
   // components/admin/dashboard/BlockHistory.tsx
   'use client';
   
   import React from 'react';
   
   interface BlockEvent {
     id: string;
     agentId: string;
     agentName: string;
     duration: string; // '1h', '24h', 'permanent'
     blockedAt: string;
     unblocked?: boolean;
     unlockedAt?: string;
     autoRejectProposals: boolean;
     reason?: string;
   }
   
   export default function BlockHistory({ events }: { events: BlockEvent[] }) {
     if (events.length === 0) {
       return null;
     }
     
     const sortedEvents = [...events].sort(
       (a, b) => new Date(b.blockedAt).getTime() - new Date(a.blockedAt).getTime()
     );
     
     return (
       <div className="bg-white rounded-lg border shadow-sm">
         <div className="p-6 border-b">
           <h2 className="text-lg font-semibold">킬스위치 히스토리</h2>
         </div>
         
         <div className="divide-y">
           {sortedEvents.map((event, idx) => (
             <div key={event.id} className="p-4 hover:bg-gray-50 transition">
               <div className="flex gap-4">
                 {/* 타임라인 점 */}
                 <div className="flex flex-col items-center">
                   <div className={`w-3 h-3 rounded-full ${
                     event.unblocked ? 'bg-gray-300' : 'bg-red-500'
                   }`}></div>
                   {idx < sortedEvents.length - 1 && (
                     <div className="w-0.5 h-12 bg-gray-200"></div>
                   )}
                 </div>
                 
                 {/* 이벤트 내용 */}
                 <div className="flex-1">
                   <div className="flex justify-between items-start">
                     <div>
                       <p className="font-semibold">{event.agentName}</p>
                       <p className="text-sm text-gray-600">
                         {new Date(event.blockedAt).toLocaleString('ko-KR')}
                       </p>
                     </div>
                     <span className={`px-2 py-1 text-xs rounded font-semibold ${
                       event.duration === '1h' ? 'bg-blue-100 text-blue-800' :
                       event.duration === '24h' ? 'bg-orange-100 text-orange-800' :
                       'bg-red-100 text-red-800'
                     }`}>
                       {event.duration === '1h' ? '1시간' :
                        event.duration === '24h' ? '24시간' :
                        '영구'}
                     </span>
                   </div>
                   
                   <div className="mt-2 space-y-1 text-sm">
                     <p>
                       자동 거절: <span className="font-medium">
                         {event.autoRejectProposals ? '예' : '아니오'}
                       </span>
                     </p>
                     {event.reason && (
                       <p>사유: <span className="font-medium">{event.reason}</span></p>
                     )}
                     {event.unblocked && event.unlockedAt && (
                       <p className="text-green-600">
                         차단 해제: {new Date(event.unlockedAt).toLocaleString('ko-KR')}
                       </p>
                     )}
                   </div>
                 </div>
               </div>
             </div>
           ))}
         </div>
       </div>
     );
   }
   ```

### 4-6 API 통합 (agentStatsApi.ts)
**목표**: 통계 데이터 조회 API

**세부 작업**:

1. **agentStatsApi.ts 작성** (`lib/api/agentStatsApi.ts`)

   ```typescript
   // lib/api/agentStatsApi.ts
   import { apiClient } from '@/lib/api/client';
   import { unwrapEnvelope } from '@/lib/api/envelope';
   
   export const agentStatsApi = {
     async getAllAgentStats(filters: {
       startDate: string;
       endDate: string;
     }) {
       const params = new URLSearchParams();
       params.append('startDate', filters.startDate);
       params.append('endDate', filters.endDate);
       
       const response = await apiClient.get(`/api/v1/agents/stats?${params}`);
       return unwrapEnvelope(response.data);
     },
     
     async getAgentStats(agentId: string, filters: {
       startDate: string;
       endDate: string;
     }) {
       const params = new URLSearchParams();
       params.append('startDate', filters.startDate);
       params.append('endDate', filters.endDate);
       
       const response = await apiClient.get(`/api/v1/agents/${agentId}/stats?${params}`);
       return unwrapEnvelope(response.data);
     },
     
     async getTimeline(filters: {
       startDate: string;
       endDate: string;
     }) {
       const params = new URLSearchParams();
       params.append('startDate', filters.startDate);
       params.append('endDate', filters.endDate);
       
       const response = await apiClient.get(`/api/v1/agents/stats/timeline?${params}`);
       return unwrapEnvelope(response.data);
     },
     
     async getBlockHistory(filters: {
       startDate: string;
       endDate: string;
     }) {
       const params = new URLSearchParams();
       params.append('startDate', filters.startDate);
       params.append('endDate', filters.endDate);
       
       const response = await apiClient.get(`/api/v1/agents/block-history?${params}`);
       return unwrapEnvelope(response.data);
     }
   };
   ```

### 4-7 React Testing Library 단위 테스트

1. **TimeSeriesCharts.test.tsx** (`__tests__/components/admin/dashboard/TimeSeriesCharts.test.tsx`)

   ```typescript
   // __tests__/components/admin/dashboard/TimeSeriesCharts.test.tsx
   import { render, screen } from '@testing-library/react';
   import TimeSeriesCharts from '@/components/admin/dashboard/TimeSeriesCharts';
   
   describe('TimeSeriesCharts', () => {
     it('3개의 차트를 렌더링한다', () => {
       const mockData = {
         timeline: [
           {
             date: '2026-04-01',
             proposalCount: 10,
             approvalRate: 0.8,
             avgApprovalTime: 1000,
             agentMetrics: [
               {
                 agentId: '1',
                 agentName: 'Agent A',
                 proposalCount: 5,
                 approvalRate: 0.8,
                 avgApprovalTime: 1000
               }
             ]
           }
         ]
       };
       
       render(<TimeSeriesCharts data={mockData} />);
       
       expect(screen.getByText(/에이전트별 제안 수/i)).toBeInTheDocument();
       expect(screen.getByText(/승인률 추이/i)).toBeInTheDocument();
       expect(screen.getByText(/평균 승인 소요 시간/i)).toBeInTheDocument();
     });
   });
   ```

2. **AnomalyAlerts.test.tsx** (`__tests__/components/admin/dashboard/AnomalyAlerts.test.tsx`)

   ```typescript
   // __tests__/components/admin/dashboard/AnomalyAlerts.test.tsx
   import { render, screen } from '@testing-library/react';
   import AnomalyAlerts from '@/components/admin/dashboard/AnomalyAlerts';
   
   describe('AnomalyAlerts', () => {
     it('이상 행동이 없으면 아무것도 렌더링하지 않는다', () => {
       const { container } = render(<AnomalyAlerts anomalies={[]} />);
       expect(container.firstChild).toBeNull();
     });
     
     it('거절률 > 50% 에이전트를 강조한다', () => {
       const mockAnomalies = [
         {
           id: '1',
           name: 'Agent A',
           recentRejectRate: 0.75,
           rejectionTrend: 'increasing' as const
         }
       ];
       
       render(<AnomalyAlerts anomalies={mockAnomalies} />);
       
       expect(screen.getByText('Agent A')).toBeInTheDocument();
       expect(screen.getByText(/75.0%/)).toBeInTheDocument();
     });
   });
   ```

## 5. 산출물

- `/app/admin/agent-dashboard/page.tsx` - 대시보드 메인 페이지
- `/components/admin/dashboard/TimeSeriesCharts.tsx` - 시계열 차트
- `/components/admin/dashboard/AgentStatsTable.tsx` - 상세 통계 테이블
- `/components/admin/dashboard/AnomalyAlerts.tsx` - 이상 행동 알림
- `/components/admin/dashboard/BlockHistory.tsx` - 킬스위치 히스토리
- `/lib/api/agentStatsApi.ts` - API 클라이언트
- `/__tests__/components/admin/dashboard/` - 단위 테스트 4개 이상

## 6. 완료 기준

- [ ] `/admin/agent-dashboard` 페이지 완성
- [ ] 시계열 차트 3개 모두 완성 (제안 수, 승인률, 소요 시간)
- [ ] 에이전트별 상세 통계 테이블 완성
  - [ ] 기본 통계 (총 제안, 승인, 거절, 승인률, 소요 시간)
  - [ ] 확장 정보 (거절 사유 상위 5개, 문서 타입)
- [ ] 이상 행동 알림 완성 (거절률 > 50%)
- [ ] 킬스위치 히스토리 타임라인 완성
- [ ] 날짜 범위 필터 완성
- [ ] CSV Export 기능 완성
- [ ] React Query polling (30초) 구현
- [ ] agentStatsApi.ts 모든 메서드 구현 및 unwrapEnvelope 적용
- [ ] React Testing Library 테스트 커버리지 ≥ 80%
- [ ] 모든 UI 컴포넌트 반응형 설계
- [ ] 검수 보고서 작성 완료
- [ ] 보안 취약점 검사 보고서 작성 완료

## 7. 작업 지침

### 7-1 차트 라이브러리 선택
- **선택**: recharts (React, TypeScript 기본 지원, 가벼움)
- **대안**: Apache ECharts, Chart.js (필요 시)
- **커스터마이징**: recharts 제공 차트로 충분, 색상/범례 조정만

### 7-2 통계 계산 로직
- **승인률**: approvedCount / totalCount
- **평균 소요 시간**: 모든 제안의 소요 시간 평균 (밀리초)
- **24시간 거절률**: 최근 24시간 내 제안 중 거절 비율
- **거절 사유 상위 5개**: 거절 피드백 텍스트 빈도 분석

### 7-3 이상 행동 조건
- **거절률 > 50%**: 24시간 내
- **킬스위치**: 최근 발동 이력 3개월 표시
- **추세**: 과거 7일 기준 거절률 증감 판단

### 7-4 성능 최적화
- **차트 데이터**: 30일을 기본값으로 제한
- **폴링 간격**: 30초 (대시보드 갱신)
- **대용량 데이터**: 필요 시 데이터 축약 (날짜별 집계)

### 7-5 보안 체크리스트
- CSV Export는 관리자 전용 (role 확인)
- 내보낸 데이터에 민감 정보 미포함
- 대시보드 접근은 감사 로그 기록
- 통계 계산은 서버에서만 수행 (클라이언트 계산 금지)

### 7-6 UX 가이드
- **색상 코드**:
  - 승인률 ≥ 80%: 초록
  - 승인률 50~80%: 노랑
  - 승인률 < 50%: 빨강
- **레이아웃**: 상단 헤더, 중단 차트, 하단 테이블
- **반응형**: 태블릿 이하에서는 차트 세로 배열

### 7-7 배포 전 체크리스트
- [ ] 차트 렌더링 성능 확인 (1000+ 데이터 포인트)
- [ ] 로컬 및 프로덕션 API 동작 확인
- [ ] CSV 인코딩 (UTF-8 BOM) 확인
- [ ] 시간대 변환 (로컬 vs UTC) 확인

---

**예상 소요 시간**: 32시간 (분석 6시간 + 차트 개발 10시간 + 테이블 및 알림 8시간 + 테스트 4시간 + 최적화 4시간)

**담당자 검수**: 2시간 (차트 정확성, 성능, 이상 탐지 로직 검증)
