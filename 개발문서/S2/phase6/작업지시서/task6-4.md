# FG6.1 작업지시서 4: 비용·사용량 대시보드 (차트, 시계열, CSV Export)

## 1. 작업 목적
AI 플랫폼의 토큰 사용량, API 호출 빈도, 예상 비용을 시각화하는 대시보드를 구축하여, 관리자가 AI 서비스 비용을 모니터링하고 최적화 의사결정을 내릴 수 있도록 지원합니다.

## 2. 작업 범위

### 포함 사항
- **비용·사용량 페이지** (`/admin/usage`):
  - 시계열 차트 (recharts 활용):
    - 모델별 토큰 사용량: StackedBarChart (최근 30일)
    - 프로바이더별 호출 수: LineChart (최근 30일)
    - 총 예상 비용 추이: AreaChart (최근 30일)
  
  - 모델별 상세 테이블 (AdminTable):
    - 모델명, 누적 토큰(input/output 분리), 호출 수, 평균 지연(ms), 예상 비용
    - 정렬/필터 지원
  
  - 필터 & 제어:
    - 기간 선택: 7일 / 30일 / 90일 버튼
    - 프로바이더 필터: 드롭다운 (다중 선택 선택사항)
    - CSV Export 버튼
  
  - 통계 카드:
    - 누적 토큰 수, 총 호출 수, 총 예상 비용, 평균 지연
  
  - 갱신 전략: React Query polling (30초)
  - 마지막 갱신 시각 표시

- **API 통합**:
  - GET /usage (시계열 데이터)
  - GET /usage/models (모델별 상세)
  - GET /providers (필터용)
  - unwrapEnvelope 활용

- **CSV Export**:
  - 선택 기간의 데이터를 CSV로 다운로드
  - 헤더: 모델, 날짜, 토큰(input/output), 호출 수, 비용

- **단위 테스트**:
  - UsageChart 렌더링
  - DateRangeFilter 상호작용
  - CSV Export 로직

- **UI/UX**:
  - 반응형 (640px, 768px, 1024px)
  - WCAG 2.1 AA
  - 로딩/에러/빈 상태 처리
  - 차트 hover 상태 (tooltip)

### 제외 사항
- 실시간 스트리밍 데이터 (배치 갱신만)
- 머신러닝 기반 비용 예측
- 외부 빌링 시스템 연동
- Admin 레이아웃, 공통 컴포넌트 (FG6.1 task6-1)
- 프로바이더/프롬프트 관리 (FG6.2/6-3)

## 3. 선행 조건
- FG6.1 task6-1 완료
- Phase 1~5 백엔드 API 완성 (usage 엔드포인트)
- Phase 3 Conversation 도메인의 세션별 모델 호출 메트릭 API (대화 세션에서 발생한 토큰 사용량·비용을 대시보드에 포함하기 위함)
- recharts 라이브러리 설치 완료
- React Query 설정 완료

## 4. 주요 작업 항목

### 4-1 사용량 API 클라이언트 구현
**목표**: 사용량 데이터 조회 API 추상화

**세부 작업**:
- `lib/api/usageApi.ts` 생성:
  ```typescript
  import { unwrapEnvelope } from '@/lib/api/utils';
  import { useQuery } from '@tanstack/react-query';
  
  export interface UsageDataPoint {
    date: string; // YYYY-MM-DD
    modelName: string;
    providerName: string;
    inputTokens: number;
    outputTokens: number;
    totalTokens: number;
    callCount: number;
    latencyMs: number;
    costUSD: number;
  }
  
  export interface ModelUsageSummary {
    modelName: string;
    providerName: string;
    totalInputTokens: number;
    totalOutputTokens: number;
    totalTokens: number;
    callCount: number;
    avgLatencyMs: number;
    totalCostUSD: number;
  }
  
  export interface UsageStatistics {
    totalTokens: number;
    totalCallCount: number;
    totalCostUSD: number;
    avgLatencyMs: number;
    data: UsageDataPoint[];
    modelSummaries: ModelUsageSummary[];
    lastUpdatedAt: string;
  }
  
  // 시계열 데이터 조회
  export function useUsageStats(
    days: number = 30,
    providerFilter?: string[]
  ) {
    const params = new URLSearchParams();
    params.append('days', String(days));
    if (providerFilter?.length) {
      providerFilter.forEach(p => params.append('provider', p));
    }
    
    return useQuery<UsageStatistics, Error>({
      queryKey: ['usage', { days, providerFilter }],
      queryFn: async () => {
        const res = await fetch(`/api/usage?${params}`);
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
      refetchInterval: 30000, // 30초 주기 갱신
    });
  }
  
  // 모델별 상세 조회
  export function useModelUsageDetail(
    days: number = 30,
    providerFilter?: string[]
  ) {
    const params = new URLSearchParams();
    params.append('days', String(days));
    if (providerFilter?.length) {
      providerFilter.forEach(p => params.append('provider', p));
    }
    
    return useQuery<ModelUsageSummary[], Error>({
      queryKey: ['usage', 'models', { days, providerFilter }],
      queryFn: async () => {
        const res = await fetch(`/api/usage/models?${params}`);
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
      refetchInterval: 30000,
    });
  }
  
  // CSV Export 데이터 생성
  export async function exportUsageAsCSV(
    days: number = 30,
    providerFilter?: string[]
  ): Promise<string> {
    const params = new URLSearchParams();
    params.append('days', String(days));
    if (providerFilter?.length) {
      providerFilter.forEach(p => params.append('provider', p));
    }
    
    const res = await fetch(`/api/usage/export?${params}`);
    return res.text();
  }
  ```

### 4-2 사용량 대시보드 페이지 구현
**목표**: 차트와 테이블을 통합한 대시보드

**세부 작업**:
- `app/admin/usage/page.tsx`:
  ```typescript
  import { UsageDashboard } from '@/components/admin/usage/UsageDashboard';
  
  export default function UsagePage() {
    return (
      <div className="space-y-8">
        <div>
          <h1 className="text-2xl font-bold text-gray-900">비용·사용량 현황</h1>
          <p className="mt-1 text-gray-600">
            AI 모델 사용량, 호출 빈도, 예상 비용을 모니터링합니다.
          </p>
        </div>
        
        <UsageDashboard />
      </div>
    );
  }
  ```

- `components/admin/usage/UsageDashboard.tsx`:
  ```typescript
  import { useState, useMemo } from 'react';
  import { useUsageStats, useModelUsageDetail } from '@/lib/api/usageApi';
  import { useProviders } from '@/lib/api/providerApi';
  import { LoadingSkeleton, ErrorState } from '@/components/admin';
  import { StatisticsCards } from './StatisticsCards';
  import { UsageCharts } from './UsageCharts';
  import { ModelDetailTable } from './ModelDetailTable';
  import { DateRangeFilter } from './DateRangeFilter';
  import { ProviderFilter } from './ProviderFilter';
  import { CSVExportButton } from './CSVExportButton';
  import { RefreshCw } from 'lucide-react';
  
  export function UsageDashboard() {
    const [days, setDays] = useState(30);
    const [providerFilter, setProviderFilter] = useState<string[]>([]);
    
    const { data: usage, isLoading: usageLoading, error: usageError, refetch } = useUsageStats(days, providerFilter);
    const { data: modelDetail, isLoading: modelLoading } = useModelUsageDetail(days, providerFilter);
    const { data: providers } = useProviders();
    
    const isLoading = usageLoading || modelLoading;
    const error = usageError;
    
    if (isLoading) return <LoadingSkeleton rows={5} />;
    if (error) return <ErrorState error={error} />;
    if (!usage) return null;
    
    return (
      <div className="space-y-8">
        {/* 필터 & 제어 */}
        <div className="bg-white rounded-lg border border-gray-200 p-6">
          <div className="space-y-4">
            {/* 기간 선택 */}
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-3">기간</label>
              <DateRangeFilter
                selectedDays={days}
                onDaysChange={setDays}
              />
            </div>
            
            {/* 프로바이더 필터 */}
            {providers && providers.length > 0 && (
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-3">프로바이더</label>
                <ProviderFilter
                  providers={providers}
                  selectedProviders={providerFilter}
                  onProvidersChange={setProviderFilter}
                />
              </div>
            )}
            
            {/* 새로고침 & Export */}
            <div className="flex justify-between items-center pt-4 border-t border-gray-200">
              <button
                onClick={() => refetch()}
                className="flex items-center gap-2 px-4 py-2 text-gray-700 bg-gray-100 hover:bg-gray-200 rounded-lg"
              >
                <RefreshCw className="w-4 h-4" />
                새로고침
              </button>
              <CSVExportButton days={days} providerFilter={providerFilter} />
            </div>
            
            {/* 마지막 갱신 시각 */}
            <p className="text-xs text-gray-500 text-right">
              마지막 갱신: {new Date(usage.lastUpdatedAt).toLocaleString('ko-KR')}
            </p>
          </div>
        </div>
        
        {/* 통계 카드 */}
        <StatisticsCards
          totalTokens={usage.totalTokens}
          totalCalls={usage.totalCallCount}
          totalCost={usage.totalCostUSD}
          avgLatency={usage.avgLatencyMs}
        />
        
        {/* 차트 */}
        <UsageCharts data={usage.data} />
        
        {/* 모델별 상세 테이블 */}
        {modelDetail && <ModelDetailTable data={modelDetail} />}
      </div>
    );
  }
  ```

### 4-3 통계 카드 구현
**목표**: 핵심 지표를 한눈에 보기

**세부 작업**:
- `components/admin/usage/StatisticsCards.tsx`:
  ```typescript
  import { TrendingUp, Zap, DollarSign, Clock } from 'lucide-react';
  
  interface StatisticsCardsProps {
    totalTokens: number;
    totalCalls: number;
    totalCost: number;
    avgLatency: number;
  }
  
  export function StatisticsCards({
    totalTokens,
    totalCalls,
    totalCost,
    avgLatency,
  }: StatisticsCardsProps) {
    const stats = [
      {
        icon: Zap,
        label: '누적 토큰',
        value: totalTokens.toLocaleString(),
        color: 'bg-blue-50 text-blue-700',
      },
      {
        icon: TrendingUp,
        label: '총 호출 수',
        value: totalCalls.toLocaleString(),
        color: 'bg-green-50 text-green-700',
      },
      {
        icon: DollarSign,
        label: '총 예상 비용',
        value: `$${totalCost.toFixed(2)}`,
        color: 'bg-purple-50 text-purple-700',
      },
      {
        icon: Clock,
        label: '평균 지연',
        value: `${avgLatency.toFixed(0)}ms`,
        color: 'bg-orange-50 text-orange-700',
      },
    ];
    
    return (
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
        {stats.map((stat, index) => {
          const Icon = stat.icon;
          return (
            <div
              key={index}
              className={`${stat.color} rounded-lg p-6 border border-current border-opacity-20`}
            >
              <div className="flex items-center gap-4">
                <Icon className="w-8 h-8 opacity-80" />
                <div>
                  <p className="text-sm font-medium opacity-75">{stat.label}</p>
                  <p className="text-2xl font-bold mt-1">{stat.value}</p>
                </div>
              </div>
            </div>
          );
        })}
      </div>
    );
  }
  ```

### 4-4 차트 구현
**목표**: recharts를 활용한 시계열 시각화

**세부 작업**:
- `components/admin/usage/UsageCharts.tsx`:
  ```typescript
  import {
    BarChart,
    Bar,
    LineChart,
    Line,
    AreaChart,
    Area,
    XAxis,
    YAxis,
    CartesianGrid,
    Tooltip,
    Legend,
    ResponsiveContainer,
  } from 'recharts';
  import { UsageDataPoint } from '@/lib/api/usageApi';
  
  interface UsageChartsProps {
    data: UsageDataPoint[];
  }
  
  export function UsageCharts({ data }: UsageChartsProps) {
    // 데이터 전처리: 날짜별로 그룹화
    const chartData = useMemo(() => {
      const grouped = new Map<string, Map<string, number>>();
      const costByDate = new Map<string, number>();
      const callsByDate = new Map<string, number>();
      
      data.forEach((point) => {
        if (!grouped.has(point.date)) {
          grouped.set(point.date, new Map());
        }
        
        const dateMap = grouped.get(point.date)!;
        const current = dateMap.get(point.modelName) || 0;
        dateMap.set(point.modelName, current + point.totalTokens);
        
        costByDate.set(point.date, (costByDate.get(point.date) || 0) + point.costUSD);
        callsByDate.set(point.date, (callsByDate.get(point.date) || 0) + point.callCount);
      });
      
      return Array.from(grouped.entries()).map(([date, models]) => ({
        date: new Date(date).toLocaleDateString('ko-KR', {
          month: 'short',
          day: 'numeric',
        }),
        ...Object.fromEntries(models),
        cost: costByDate.get(date) || 0,
        calls: callsByDate.get(date) || 0,
      }));
    }, [data]);
    
    // 고유 모델명 추출
    const modelNames = useMemo(() => {
      const names = new Set<string>();
      data.forEach((point) => names.add(point.modelName));
      return Array.from(names);
    }, [data]);
    
    const colors = [
      '#3B82F6', '#EF4444', '#10B981', '#F59E0B',
      '#8B5CF6', '#EC4899', '#14B8A6', '#F97316',
    ];
    
    return (
      <div className="space-y-8">
        {/* 모델별 토큰 사용량 (StackedBarChart) */}
        <div className="bg-white rounded-lg border border-gray-200 p-6">
          <h3 className="text-lg font-semibold mb-4">모델별 토큰 사용량</h3>
          <ResponsiveContainer width="100%" height={300}>
            <BarChart data={chartData} margin={{ top: 20, right: 30, left: 0, bottom: 0 }}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="date" />
              <YAxis />
              <Tooltip
                formatter={(value) => value.toLocaleString()}
                contentStyle={{ backgroundColor: '#fff', border: '1px solid #ccc' }}
              />
              <Legend />
              {modelNames.map((modelName, index) => (
                <Bar
                  key={modelName}
                  dataKey={modelName}
                  stackId="a"
                  fill={colors[index % colors.length]}
                  name={modelName}
                />
              ))}
            </BarChart>
          </ResponsiveContainer>
        </div>
        
        {/* 호출 수 추이 (LineChart) */}
        <div className="bg-white rounded-lg border border-gray-200 p-6">
          <h3 className="text-lg font-semibold mb-4">프로바이더별 호출 수</h3>
          <ResponsiveContainer width="100%" height={300}>
            <LineChart data={chartData} margin={{ top: 20, right: 30, left: 0, bottom: 0 }}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="date" />
              <YAxis />
              <Tooltip
                formatter={(value) => value.toLocaleString()}
                contentStyle={{ backgroundColor: '#fff', border: '1px solid #ccc' }}
              />
              <Legend />
              <Line
                type="monotone"
                dataKey="calls"
                stroke="#3B82F6"
                name="호출 수"
                dot={{ fill: '#3B82F6', r: 4 }}
              />
            </LineChart>
          </ResponsiveContainer>
        </div>
        
        {/* 비용 추이 (AreaChart) */}
        <div className="bg-white rounded-lg border border-gray-200 p-6">
          <h3 className="text-lg font-semibold mb-4">총 예상 비용 추이</h3>
          <ResponsiveContainer width="100%" height={300}>
            <AreaChart data={chartData} margin={{ top: 20, right: 30, left: 0, bottom: 0 }}>
              <defs>
                <linearGradient id="colorCost" x1="0" y1="0" x2="0" y2="1">
                  <stop offset="5%" stopColor="#8B5CF6" stopOpacity={0.8} />
                  <stop offset="95%" stopColor="#8B5CF6" stopOpacity={0} />
                </linearGradient>
              </defs>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="date" />
              <YAxis />
              <Tooltip
                formatter={(value) => `$${value.toFixed(2)}`}
                contentStyle={{ backgroundColor: '#fff', border: '1px solid #ccc' }}
              />
              <Area
                type="monotone"
                dataKey="cost"
                stroke="#8B5CF6"
                fillOpacity={1}
                fill="url(#colorCost)"
                name="비용 (USD)"
              />
            </AreaChart>
          </ResponsiveContainer>
        </div>
      </div>
    );
  }
  ```

### 4-5 필터 컴포넌트 구현
**목표**: 기간 및 프로바이더 필터링

**세부 작업**:
- `components/admin/usage/DateRangeFilter.tsx`:
  ```typescript
  interface DateRangeFilterProps {
    selectedDays: number;
    onDaysChange: (days: number) => void;
  }
  
  export function DateRangeFilter({
    selectedDays,
    onDaysChange,
  }: DateRangeFilterProps) {
    const options = [7, 30, 90];
    
    return (
      <div className="flex gap-2">
        {options.map((days) => (
          <button
            key={days}
            onClick={() => onDaysChange(days)}
            className={`px-4 py-2 rounded-lg font-medium transition-colors ${
              selectedDays === days
                ? 'bg-blue-600 text-white'
                : 'bg-gray-100 text-gray-700 hover:bg-gray-200'
            }`}
          >
            {days}일
          </button>
        ))}
      </div>
    );
  }
  ```

- `components/admin/usage/ProviderFilter.tsx`:
  ```typescript
  import { Provider } from '@/lib/api/providerApi';
  
  interface ProviderFilterProps {
    providers: Provider[];
    selectedProviders: string[];
    onProvidersChange: (providers: string[]) => void;
  }
  
  export function ProviderFilter({
    providers,
    selectedProviders,
    onProvidersChange,
  }: ProviderFilterProps) {
    const handleToggle = (providerId: string) => {
      setSelectedProviders(
        selectedProviders.includes(providerId)
          ? selectedProviders.filter((p) => p !== providerId)
          : [...selectedProviders, providerId]
      );
    };
    
    return (
      <div className="flex flex-wrap gap-2">
        {providers.map((provider) => (
          <button
            key={provider.id}
            onClick={() => handleToggle(provider.id)}
            className={`px-3 py-1 rounded-lg text-sm font-medium transition-colors ${
              selectedProviders.includes(provider.id)
                ? 'bg-blue-600 text-white'
                : 'bg-gray-100 text-gray-700 hover:bg-gray-200'
            }`}
          >
            {provider.name}
          </button>
        ))}
      </div>
    );
  }
  ```

### 4-6 CSV Export 구현
**목표**: 데이터 다운로드 기능

**세부 작업**:
- `components/admin/usage/CSVExportButton.tsx`:
  ```typescript
  import { Download } from 'lucide-react';
  import { exportUsageAsCSV } from '@/lib/api/usageApi';
  import { useAdminToast } from '@/hooks/useAdminToast';
  
  interface CSVExportButtonProps {
    days: number;
    providerFilter?: string[];
  }
  
  export function CSVExportButton({ days, providerFilter }: CSVExportButtonProps) {
    const [isLoading, setIsLoading] = useState(false);
    const toast = useAdminToast();
    
    const handleExport = async () => {
      setIsLoading(true);
      try {
        const csv = await exportUsageAsCSV(days, providerFilter);
        
        // Blob 생성 및 다운로드
        const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
        const link = document.createElement('a');
        const url = URL.createObjectURL(blob);
        
        link.setAttribute('href', url);
        link.setAttribute(
          'download',
          `usage-${days}days-${new Date().toISOString().split('T')[0]}.csv`
        );
        link.style.visibility = 'hidden';
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        
        toast.success('CSV 파일이 다운로드되었습니다');
      } catch (error) {
        toast.error('Export 실패: ' + (error as Error).message);
      } finally {
        setIsLoading(false);
      }
    };
    
    return (
      <button
        onClick={handleExport}
        disabled={isLoading}
        className="flex items-center gap-2 px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 disabled:opacity-50"
      >
        <Download className="w-4 h-4" />
        CSV Export
      </button>
    );
  }
  ```

### 4-7 모델별 상세 테이블 구현
**목표**: 모델 레벨의 상세 통계

**세부 작업**:
- `components/admin/usage/ModelDetailTable.tsx`:
  ```typescript
  import { AdminTable } from '@/components/admin/AdminTable';
  import { ModelUsageSummary } from '@/lib/api/usageApi';
  import { useMemo } from 'react';
  import { ColumnDef } from '@tanstack/react-table';
  
  interface ModelDetailTableProps {
    data: ModelUsageSummary[];
  }
  
  export function ModelDetailTable({ data }: ModelDetailTableProps) {
    const columns = useMemo<ColumnDef<ModelUsageSummary>[]>(
      () => [
        {
          accessorKey: 'modelName',
          header: '모델명',
          cell: (info) => <span className="font-medium">{info.getValue()}</span>,
        },
        {
          accessorKey: 'providerName',
          header: '프로바이더',
        },
        {
          accessorKey: 'totalInputTokens',
          header: 'Input 토큰',
          cell: (info) => (info.getValue() as number).toLocaleString(),
        },
        {
          accessorKey: 'totalOutputTokens',
          header: 'Output 토큰',
          cell: (info) => (info.getValue() as number).toLocaleString(),
        },
        {
          accessorKey: 'totalTokens',
          header: '총 토큰',
          cell: (info) => (info.getValue() as number).toLocaleString(),
        },
        {
          accessorKey: 'callCount',
          header: '호출 수',
          cell: (info) => (info.getValue() as number).toLocaleString(),
        },
        {
          accessorKey: 'avgLatencyMs',
          header: '평균 지연',
          cell: (info) => `${(info.getValue() as number).toFixed(0)}ms`,
        },
        {
          accessorKey: 'totalCostUSD',
          header: '총 비용',
          cell: (info) => `$${(info.getValue() as number).toFixed(2)}`,
        },
      ],
      []
    );
    
    return (
      <div className="bg-white rounded-lg border border-gray-200 p-6">
        <h3 className="text-lg font-semibold mb-4">모델별 상세 통계</h3>
        <AdminTable<ModelUsageSummary>
          columns={columns}
          data={data}
          pageSize={15}
          sortable={true}
        />
      </div>
    );
  }
  ```

### 4-8 단위 테스트 작성
**세부 작업**:
- `components/admin/usage/__tests__/UsageDashboard.test.tsx`:
  ```typescript
  import { render, screen, waitFor } from '@testing-library/react';
  import userEvent from '@testing-library/user-event';
  import { UsageDashboard } from '../UsageDashboard';
  import * as usageApi from '@/lib/api/usageApi';
  import * as providerApi from '@/lib/api/providerApi';
  
  jest.mock('@/lib/api/usageApi');
  jest.mock('@/lib/api/providerApi');
  
  describe('UsageDashboard', () => {
    it('should render statistics cards', async () => {
      const mockUsageData = {
        totalTokens: 1000000,
        totalCallCount: 5000,
        totalCostUSD: 250.5,
        avgLatencyMs: 150,
        data: [],
        modelSummaries: [],
        lastUpdatedAt: new Date().toISOString(),
      };
      
      (usageApi.useUsageStats as jest.Mock).mockReturnValue({
        data: mockUsageData,
        isLoading: false,
      });
      
      render(<UsageDashboard />);
      
      await waitFor(() => {
        expect(screen.getByText('누적 토큰')).toBeInTheDocument();
        expect(screen.getByText('1,000,000')).toBeInTheDocument();
      });
    });
    
    it('should filter by date range', async () => {
      const user = userEvent.setup();
      
      (usageApi.useUsageStats as jest.Mock).mockReturnValue({
        data: { /* ... */ },
        isLoading: false,
      });
      
      render(<UsageDashboard />);
      
      const button30 = screen.getByText('30일');
      await user.click(button30);
      
      expect(usageApi.useUsageStats).toHaveBeenCalledWith(30, []);
    });
  });
  ```

- `components/admin/usage/__tests__/UsageCharts.test.tsx`
- `components/admin/usage/__tests__/CSVExportButton.test.tsx`

## 5. 산출물

### 필수 산출물
1. **API 클라이언트**: `lib/api/usageApi.ts`
2. **페이지 컴포넌트**: `app/admin/usage/page.tsx`
3. **기능 컴포넌트**:
   - `components/admin/usage/UsageDashboard.tsx`
   - `components/admin/usage/StatisticsCards.tsx`
   - `components/admin/usage/UsageCharts.tsx`
   - `components/admin/usage/ModelDetailTable.tsx`
   - `components/admin/usage/DateRangeFilter.tsx`
   - `components/admin/usage/ProviderFilter.tsx`
   - `components/admin/usage/CSVExportButton.tsx`
4. **단위 테스트** (3개 이상)
5. **검수보고서** (5회 UI 리뷰)
6. **보안취약점검사보고서**

## 6. 완료 기준

- [ ] 통계 카드 4개 모두 표시
- [ ] 3개 차트(StackedBar, Line, Area) 정상 렌더링
- [ ] 기간 필터(7/30/90일) 작동
- [ ] 프로바이더 필터 작동
- [ ] CSV Export 버튼이 파일 다운로드
- [ ] 모델별 상세 테이블 표시 및 정렬
- [ ] 반응형 디자인 정상
- [ ] WCAG 2.1 AA 준수
- [ ] 단위 테스트 커버리지 > 80%
- [ ] 5회 UI 리뷰 완료
- [ ] 보안취약점검사 완료

## 7. 작업 지침

### 7-1 recharts 활용
- ResponsiveContainer로 반응형 차트 구현
- Tooltip 커스터마이징 (포맷, 스타일)
- 범례(Legend) 포함
- 색상 일관성 유지

### 7-2 데이터 전처리
- useMemo로 차트 데이터 최적화
- 날짜별 그룹화 및 모델별 합계 계산
- 모델명 추출 및 색상 할당

### 7-3 CSV Export
- BlobURL 생성 및 동적 다운로드
- 파일명: usage-{days}days-{date}.csv
- 헤더 행 포함: 모델, 날짜, 토큰, 호출수, 비용

### 7-4 성능 최적화
- React Query polling (30초)
- useMemo로 컬럼 및 데이터 메모이제이션
- 차트 렌더링 성능 모니터링

### 7-5 접근성
- Chart title로 차트 목적 명확화
- Tooltip 포함 (마우스 hover)
- 색상 대비 확인

### 7-6 UI 리뷰 체크리스트
1. 카드 배치 및 간격
2. 차트 가독성 (범례, 축 레이블)
3. 필터 UI 직관성
4. Export 버튼 위치 및 사용성
5. 반응형 (640px, 768px, 1024px)

### 7-7 에러 처리
- API 호출 실패 시 ErrorState
- CSV Export 실패 시 Toast 에러
- 빈 데이터 처리

---

**작업 예상 소요 시간**: 5~7일
**의존 관계**: FG6.1 task6-1 완료 필수

## 부록: API 응답 예시

### GET /usage 응답
```json
{
  "code": "OK",
  "data": {
    "totalTokens": 5000000,
    "totalCallCount": 25000,
    "totalCostUSD": 1250.75,
    "avgLatencyMs": 145,
    "lastUpdatedAt": "2026-04-17T10:30:00Z",
    "data": [
      {
        "date": "2026-04-10",
        "modelName": "gpt-4",
        "providerName": "OpenAI",
        "inputTokens": 100000,
        "outputTokens": 50000,
        "totalTokens": 150000,
        "callCount": 500,
        "latencyMs": 150,
        "costUSD": 12.5
      }
    ],
    "modelSummaries": [
      {
        "modelName": "gpt-4",
        "providerName": "OpenAI",
        "totalInputTokens": 3000000,
        "totalOutputTokens": 1500000,
        "totalTokens": 4500000,
        "callCount": 15000,
        "avgLatencyMs": 145,
        "totalCostUSD": 1000.0
      }
    ]
  }
}
```

### GET /usage/models 응답
```json
{
  "code": "OK",
  "data": [
    {
      "modelName": "gpt-4",
      "providerName": "OpenAI",
      "totalInputTokens": 3000000,
      "totalOutputTokens": 1500000,
      "totalTokens": 4500000,
      "callCount": 15000,
      "avgLatencyMs": 145,
      "totalCostUSD": 1000.0
    },
    {
      "modelName": "claude-3",
      "providerName": "Anthropic",
      "totalInputTokens": 1200000,
      "totalOutputTokens": 300000,
      "totalTokens": 1500000,
      "callCount": 10000,
      "avgLatencyMs": 200,
      "totalCostUSD": 250.75
    }
  ]
}
```
