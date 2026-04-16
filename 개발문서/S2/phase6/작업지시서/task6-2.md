# FG6.1 작업지시서 2: 모델/프로바이더 관리 페이지 + Capabilities 대시보드

## 1. 작업 목적
등록된 LLM/Embedding 프로바이더의 메타데이터를 관리하고, 연결 상태를 모니터링하며, 시스템 기능(RAG/Embedding/LLM/Chunking) 가용성을 시각화하는 Admin 페이지를 구축하여, 관리자가 AI 플랫폼 인프라를 효과적으로 관리할 수 있도록 지원합니다.

## 2. 작업 범위

### 포함 사항
- **프로바이더 관리 페이지** (`/admin/providers`):
  - 등록된 LLM/Embedding 프로바이더 목록 (AdminTable)
  - 각 프로바이더 메타데이터:
    - 이름, 타입(LLM/Embedding), 상태(활성/비활성/오류), 마지막 테스트 시각
    - 모델 수, 평균 응답 시간
  - "연결 테스트" 버튼 → POST /system/providers/{id}/health-check → 결과 모달
  - "기본 모델 지정" 드롭다운: 타입별(LLM, Embedding) 기본 모델 선택
  - API 키 갱신 알림: 최근 갱신일, 만료 예상일
  - 프로바이더 수정 모달 (API 키 업데이트 등)

- **Capabilities 대시보드** (`/admin/capabilities`):
  - GET /system/capabilities 응답 시각화
  - 카드 레이아웃 (최소 4개):
    - RAG 가용 여부 (on/off)
    - Embedding 모델 상태 (연결 모델명, 차원)
    - LLM 프로바이더 상태 (활성 프로바이더 수)
    - Chunking 전략 상태 (활성 전략 목록)
  - Degrade 배너: "시스템이 제한된 모드로 동작 중입니다" (일부 기능 비활성화 시)
  - Refresh 버튼 (캐시 무효화, 실시간 갱신)
  - 마지막 갱신 시각 표시

- **API 통합**:
  - `lib/api/providerApi.ts`: GET /providers, POST /providers/{id}/health-check, PUT /providers/{id}
  - `lib/api/capabilitiesApi.ts`: GET /system/capabilities
  - unwrapEnvelope 활용하여 응답 언래핑
  - React Query로 데이터 캐싱 및 동기화

- **단위 테스트** (React Testing Library):
  - ProviderList 렌더링 및 상호작용
  - HealthCheck 버튼 클릭 및 모달
  - CapabilitiesDashboard 렌더링
  - API 호출 모킹

- **UI/UX**:
  - 반응형 디자인 (640px, 768px, 1024px)
  - 로딩/에러 상태 처리 (EmptyState, ErrorState, LoadingSkeleton)
  - Toast 알림 (성공, 실패, 진행 중)
  - WCAG 2.1 AA 접근성

### 제외 사항
- 프롬프트 관리 (FG6.3)
- 비용·사용량 대시보드 (FG6.4)
- Admin 레이아웃 및 공통 컴포넌트 (FG6.1의 task6-1)
- 프로바이더 생성/삭제 (수정만 포함)
- 외부 모니터링 서비스 연동 (Datadog, New Relic 등)

## 3. 선행 조건
- FG6.1 task6-1 (Admin 레이아웃, 공통 컴포넌트) 완료
- Phase 1~5 백엔드 API 완성 (프로바이더, capabilities 엔드포인트)
- React Query 설정 완료
- API 모킹 (MSW 또는 유사) 설정 (선택)

## 4. 주요 작업 항목

### 4-1 프로바이더 API 클라이언트 구현
**목표**: 프로바이더 데이터 조회/업데이트 API 추상화

**세부 작업**:
- `lib/api/providerApi.ts` 생성:
  ```typescript
  import { unwrapEnvelope } from '@/lib/api/utils';
  import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
  
  export interface Provider {
    id: string;
    name: string;
    type: 'llm' | 'embedding';
    status: 'active' | 'inactive' | 'error';
    modelCount: number;
    lastTestedAt: string | null;
    avgResponseTimeMs: number | null;
    apiKeyUpdatedAt: string;
    apiKeyExpiresAt?: string;
    config: Record<string, unknown>; // 상세 설정
  }
  
  interface HealthCheckResponse {
    providerId: string;
    status: 'healthy' | 'unhealthy';
    latencyMs: number;
    message?: string;
    testedAt: string;
  }
  
  // 프로바이더 목록 조회
  export function useProviders() {
    return useQuery<Provider[], Error>({
      queryKey: ['providers'],
      queryFn: async () => {
        const res = await fetch('/api/providers');
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
    });
  }
  
  // 연결 테스트
  export function useHealthCheckMutation(providerId: string) {
    const queryClient = useQueryClient();
    return useMutation<HealthCheckResponse, Error, void>({
      mutationFn: async () => {
        const res = await fetch(`/api/providers/${providerId}/health-check`, {
          method: 'POST',
        });
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
      onSuccess: (data) => {
        // 프로바이더 목록 갱신
        queryClient.setQueryData(['providers'], (oldData?: Provider[]) => {
          if (!oldData) return oldData;
          return oldData.map((p) =>
            p.id === providerId
              ? {
                  ...p,
                  status: data.status === 'healthy' ? 'active' : 'error',
                  lastTestedAt: data.testedAt,
                  avgResponseTimeMs: data.latencyMs,
                }
              : p
          );
        });
      },
    });
  }
  
  // 프로바이더 업데이트 (API 키 갱신 등)
  export function useUpdateProviderMutation(providerId: string) {
    const queryClient = useQueryClient();
    return useMutation<Provider, Error, Partial<Provider>>({
      mutationFn: async (updateData) => {
        const res = await fetch(`/api/providers/${providerId}`, {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(updateData),
        });
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
      onSuccess: (data) => {
        queryClient.setQueryData(['providers'], (oldData?: Provider[]) => {
          if (!oldData) return oldData;
          return oldData.map((p) => (p.id === providerId ? data : p));
        });
      },
    });
  }
  
  // 기본 모델 지정
  export function useSetDefaultModelMutation(providerType: 'llm' | 'embedding') {
    const queryClient = useQueryClient();
    return useMutation<
      { modelName: string; providerType: string },
      Error,
      { providerType: string; modelName: string }
    >({
      mutationFn: async (data) => {
        const res = await fetch('/api/providers/default-model', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(data),
        });
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
      onSuccess: () => {
        // 캐시 무효화
        queryClient.invalidateQueries({ queryKey: ['providers'] });
      },
    });
  }
  ```

### 4-2 프로바이더 관리 페이지 구현
**목표**: 프로바이더 목록 조회, 상태 모니터링, 기본 모델 설정

**세부 작업**:
- `app/admin/providers/page.tsx` 생성:
  ```typescript
  import { ProviderList } from '@/components/admin/providers/ProviderList';
  import { DefaultModelSelector } from '@/components/admin/providers/DefaultModelSelector';
  
  export default function ProvidersPage() {
    return (
      <div className="space-y-8">
        {/* 페이지 헤더 */}
        <div>
          <h1 className="text-2xl font-bold text-gray-900">모델/프로바이더 관리</h1>
          <p className="mt-1 text-gray-600">
            등록된 LLM/Embedding 프로바이더의 상태를 모니터링하고 기본 모델을 설정합니다.
          </p>
        </div>
        
        {/* 기본 모델 선택 섹션 */}
        <DefaultModelSelector />
        
        {/* 프로바이더 목록 */}
        <ProviderList />
      </div>
    );
  }
  ```

- `components/admin/providers/ProviderList.tsx`:
  ```typescript
  import { useMemo, useState } from 'react';
  import { AdminTable } from '@/components/admin/AdminTable';
  import { StatusBadge } from '@/components/admin/StatusBadge';
  import { HealthCheckModal } from './HealthCheckModal';
  import { useProviders } from '@/lib/api/providerApi';
  import { LoadingSkeleton, ErrorState, EmptyState } from '@/components/admin';
  import { ColumnDef } from '@tanstack/react-table';
  
  export function ProviderList() {
    const { data: providers, isLoading, error } = useProviders();
    const [selectedProviderId, setSelectedProviderId] = useState<string | null>(null);
    const [isHealthCheckOpen, setIsHealthCheckOpen] = useState(false);
    
    const columns = useMemo<ColumnDef<Provider>[]>(
      () => [
        {
          accessorKey: 'name',
          header: '이름',
          cell: (info) => <span className="font-medium">{info.getValue()}</span>,
        },
        {
          accessorKey: 'type',
          header: '타입',
          cell: (info) => {
            const type = info.getValue() as string;
            return (
              <span className="px-2 py-1 bg-blue-100 text-blue-700 rounded text-sm font-medium">
                {type === 'llm' ? 'LLM' : 'Embedding'}
              </span>
            );
          },
        },
        {
          accessorKey: 'status',
          header: '상태',
          cell: (info) => {
            const status = info.getValue() as string;
            return (
              <StatusBadge
                status={status as 'active' | 'inactive' | 'error'}
                label={
                  status === 'active'
                    ? '활성'
                    : status === 'inactive'
                      ? '비활성'
                      : '오류'
                }
              />
            );
          },
        },
        {
          accessorKey: 'lastTestedAt',
          header: '마지막 테스트',
          cell: (info) => {
            const date = info.getValue() as string | null;
            return date ? (
              <span className="text-sm text-gray-600">
                {new Date(date).toLocaleString('ko-KR')}
              </span>
            ) : (
              <span className="text-sm text-gray-400">테스트 안 함</span>
            );
          },
        },
        {
          accessorKey: 'avgResponseTimeMs',
          header: '평균 응답시간',
          cell: (info) => {
            const time = info.getValue() as number | null;
            return time ? (
              <span className="text-sm">{time.toFixed(0)}ms</span>
            ) : (
              <span className="text-sm text-gray-400">-</span>
            );
          },
        },
        {
          id: 'actions',
          header: '작업',
          cell: (info) => {
            const provider = info.row.original;
            return (
              <div className="flex gap-2">
                <button
                  onClick={() => {
                    setSelectedProviderId(provider.id);
                    setIsHealthCheckOpen(true);
                  }}
                  className="px-3 py-1 text-sm font-medium text-blue-600 hover:bg-blue-50 rounded"
                >
                  연결 테스트
                </button>
              </div>
            );
          },
        },
      ],
      []
    );
    
    if (isLoading) return <LoadingSkeleton rows={5} />;
    if (error) return <ErrorState error={error} />;
    if (!providers || providers.length === 0) return <EmptyState message="등록된 프로바이더가 없습니다." />;
    
    return (
      <>
        <AdminTable<Provider>
          columns={columns}
          data={providers}
          isLoading={isLoading}
          pageSize={10}
        />
        
        {selectedProviderId && (
          <HealthCheckModal
            isOpen={isHealthCheckOpen}
            onClose={() => setIsHealthCheckOpen(false)}
            providerId={selectedProviderId}
          />
        )}
      </>
    );
  }
  ```

- `components/admin/providers/HealthCheckModal.tsx`:
  ```typescript
  import { useState } from 'react';
  import { AdminModal } from '@/components/admin/AdminModal';
  import { useHealthCheckMutation } from '@/lib/api/providerApi';
  import { useAdminToast } from '@/hooks/useAdminToast';
  import { CheckCircle, AlertCircle } from 'lucide-react';
  
  interface HealthCheckModalProps {
    isOpen: boolean;
    onClose: () => void;
    providerId: string;
  }
  
  export function HealthCheckModal({ isOpen, onClose, providerId }: HealthCheckModalProps) {
    const [result, setResult] = useState<HealthCheckResult | null>(null);
    const { mutate: healthCheck, isPending } = useHealthCheckMutation(providerId);
    const toast = useAdminToast();
    
    const handleHealthCheck = async () => {
      try {
        healthCheck(undefined, {
          onSuccess: (data) => {
            setResult(data);
            toast.success('연결 테스트 완료');
          },
          onError: (error) => {
            toast.error('연결 테스트 실패: ' + error.message);
          },
        });
      } catch (err) {
        // 에러 처리
      }
    };
    
    return (
      <AdminModal
        isOpen={isOpen}
        onClose={onClose}
        title="연결 테스트"
        onSubmit={result ? undefined : handleHealthCheck}
        submitLabel={result ? undefined : "테스트 시작"}
        isLoading={isPending}
      >
        {!result ? (
          <div className="space-y-4">
            <p className="text-gray-600">
              이 프로바이더의 연결 상태를 확인합니다. 테스트에는 몇 초가 소요될 수 있습니다.
            </p>
          </div>
        ) : (
          <div className="space-y-4">
            <div
              className={`p-4 rounded-lg flex items-start gap-3 ${
                result.status === 'healthy'
                  ? 'bg-green-50 text-green-900'
                  : 'bg-red-50 text-red-900'
              }`}
            >
              {result.status === 'healthy' ? (
                <CheckCircle className="flex-shrink-0 w-5 h-5" />
              ) : (
                <AlertCircle className="flex-shrink-0 w-5 h-5" />
              )}
              <div>
                <h4 className="font-semibold">
                  {result.status === 'healthy' ? '연결 성공' : '연결 실패'}
                </h4>
                <p className="text-sm mt-1">{result.message || '프로바이더가 정상적으로 응답합니다.'}</p>
                <p className="text-xs mt-2">응답 시간: {result.latencyMs}ms</p>
              </div>
            </div>
          </div>
        )}
      </AdminModal>
    );
  }
  ```

- `components/admin/providers/DefaultModelSelector.tsx`:
  ```typescript
  import { useMemo, useState } from 'react';
  import { useProviders, useSetDefaultModelMutation } from '@/lib/api/providerApi';
  import { useAdminToast } from '@/hooks/useAdminToast';
  
  export function DefaultModelSelector() {
    const { data: providers = [] } = useProviders();
    const { mutate: setDefaultModel } = useSetDefaultModelMutation('llm');
    const [selectedLLM, setSelectedLLM] = useState<string>('');
    const [selectedEmbedding, setSelectedEmbedding] = useState<string>('');
    const toast = useAdminToast();
    
    const llmProviders = useMemo(
      () => providers.filter((p) => p.type === 'llm'),
      [providers]
    );
    
    const embeddingProviders = useMemo(
      () => providers.filter((p) => p.type === 'embedding'),
      [providers]
    );
    
    const handleSetDefault = (providerType: 'llm' | 'embedding', modelName: string) => {
      setDefaultModel(
        { providerType, modelName },
        {
          onSuccess: () => {
            toast.success(`${providerType === 'llm' ? 'LLM' : 'Embedding'} 기본 모델 설정 완료`);
          },
          onError: (error) => {
            toast.error('설정 실패: ' + error.message);
          },
        }
      );
    };
    
    return (
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
        {/* LLM 기본 모델 */}
        <div className="bg-white p-6 rounded-lg border border-gray-200">
          <h3 className="text-lg font-semibold mb-4">기본 LLM 모델</h3>
          <select
            value={selectedLLM}
            onChange={(e) => {
              setSelectedLLM(e.target.value);
              if (e.target.value) {
                handleSetDefault('llm', e.target.value);
              }
            }}
            className="w-full px-3 py-2 border border-gray-300 rounded-lg"
          >
            <option value="">선택하세요</option>
            {llmProviders.map((p) => (
              <option key={p.id} value={p.name}>
                {p.name}
              </option>
            ))}
          </select>
        </div>
        
        {/* Embedding 기본 모델 */}
        <div className="bg-white p-6 rounded-lg border border-gray-200">
          <h3 className="text-lg font-semibold mb-4">기본 Embedding 모델</h3>
          <select
            value={selectedEmbedding}
            onChange={(e) => {
              setSelectedEmbedding(e.target.value);
              if (e.target.value) {
                handleSetDefault('embedding', e.target.value);
              }
            }}
            className="w-full px-3 py-2 border border-gray-300 rounded-lg"
          >
            <option value="">선택하세요</option>
            {embeddingProviders.map((p) => (
              <option key={p.id} value={p.name}>
                {p.name}
              </option>
            ))}
          </select>
        </div>
      </div>
    );
  }
  ```

### 4-3 Capabilities API 클라이언트 구현
**목표**: 시스템 기능 가용성 조회 추상화

**세부 작업**:
- `lib/api/capabilitiesApi.ts` 생성:
  ```typescript
  import { unwrapEnvelope } from '@/lib/api/utils';
  import { useQuery } from '@tanstack/react-query';
  
  export interface Capabilities {
    ragEnabled: boolean;
    embeddingAvailable: boolean;
    embeddingModelName?: string;
    embeddingDimension?: number;
    llmProvidersActive: number;
    llmProvidersTotal: number;
    chunkingStrategies: string[];
    degradedMode: boolean;
    lastCheckedAt: string;
  }
  
  export function useCapabilities() {
    return useQuery<Capabilities, Error>({
      queryKey: ['capabilities'],
      queryFn: async () => {
        const res = await fetch('/api/system/capabilities');
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
      refetchInterval: 30000, // 30초마다 갱신
    });
  }
  
  // 캐시 무효화 (Refresh 버튼)
  export function useRefreshCapabilities() {
    const queryClient = useQueryClient();
    return useMutation<void, Error>({
      mutationFn: async () => {
        await fetch('/api/system/capabilities/refresh', { method: 'POST' });
      },
      onSuccess: () => {
        queryClient.invalidateQueries({ queryKey: ['capabilities'] });
      },
    });
  }
  ```

### 4-4 Capabilities 대시보드 구현
**목표**: 시스템 기능 가용성 시각화

**세부 작업**:
- `app/admin/capabilities/page.tsx`:
  ```typescript
  import { CapabilitiesDashboard } from '@/components/admin/capabilities/CapabilitiesDashboard';
  
  export default function CapabilitiesPage() {
    return (
      <div className="space-y-8">
        <div>
          <h1 className="text-2xl font-bold text-gray-900">시스템 Capabilities</h1>
          <p className="mt-1 text-gray-600">
            현재 활성화된 AI 기능 및 서비스 상태를 확인합니다.
          </p>
        </div>
        
        <CapabilitiesDashboard />
      </div>
    );
  }
  ```

- `components/admin/capabilities/CapabilitiesDashboard.tsx`:
  ```typescript
  import { useCapabilities, useRefreshCapabilities } from '@/lib/api/capabilitiesApi';
  import { LoadingSkeleton, ErrorState } from '@/components/admin';
  import { CapabilityCard } from './CapabilityCard';
  import { DegradeBanner } from './DegradeBanner';
  import { RefreshCw } from 'lucide-react';
  
  export function CapabilitiesDashboard() {
    const { data: capabilities, isLoading, error } = useCapabilities();
    const { mutate: refresh, isPending } = useRefreshCapabilities();
    
    if (isLoading) return <LoadingSkeleton rows={4} />;
    if (error) return <ErrorState error={error} />;
    if (!capabilities) return null;
    
    return (
      <div className="space-y-6">
        {/* Degrade 배너 */}
        {capabilities.degradedMode && (
          <DegradeBanner />
        )}
        
        {/* 새로고침 버튼 */}
        <div className="flex justify-end">
          <button
            onClick={() => refresh()}
            disabled={isPending}
            className="flex items-center gap-2 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:opacity-50"
          >
            <RefreshCw className={`w-4 h-4 ${isPending ? 'animate-spin' : ''}`} />
            새로고침
          </button>
        </div>
        
        {/* Capability 카드들 */}
        <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
          <CapabilityCard
            title="RAG (Retrieval-Augmented Generation)"
            enabled={capabilities.ragEnabled}
            description={
              capabilities.ragEnabled
                ? "RAG 기능이 활성화되었습니다."
                : "RAG 기능이 비활성화되었습니다."
            }
          />
          
          <CapabilityCard
            title="Embedding 모델"
            enabled={capabilities.embeddingAvailable}
            description={
              capabilities.embeddingAvailable
                ? `모델: ${capabilities.embeddingModelName} (차원: ${capabilities.embeddingDimension})`
                : "활성화된 Embedding 모델이 없습니다."
            }
          />
          
          <CapabilityCard
            title="LLM 프로바이더"
            enabled={capabilities.llmProvidersActive > 0}
            description={`${capabilities.llmProvidersActive}/${capabilities.llmProvidersTotal} 활성`}
          />
          
          <CapabilityCard
            title="Chunking 전략"
            enabled={capabilities.chunkingStrategies.length > 0}
            description={
              capabilities.chunkingStrategies.length > 0
                ? capabilities.chunkingStrategies.join(", ")
                : "활성화된 전략이 없습니다."
            }
          />
        </div>
        
        {/* 마지막 갱신 시각 */}
        <p className="text-xs text-gray-500 text-right">
          마지막 갱신: {new Date(capabilities.lastCheckedAt).toLocaleString('ko-KR')}
        </p>
      </div>
    );
  }
  ```

- `components/admin/capabilities/CapabilityCard.tsx`:
  ```typescript
  import { CheckCircle, AlertCircle } from 'lucide-react';
  
  interface CapabilityCardProps {
    title: string;
    enabled: boolean;
    description: string;
  }
  
  export function CapabilityCard({ title, enabled, description }: CapabilityCardProps) {
    return (
      <div
        className={`p-6 rounded-lg border-2 transition-colors ${
          enabled
            ? 'bg-green-50 border-green-200'
            : 'bg-gray-50 border-gray-200'
        }`}
      >
        <div className="flex items-start gap-4">
          {enabled ? (
            <CheckCircle className="w-6 h-6 text-green-600 flex-shrink-0 mt-1" />
          ) : (
            <AlertCircle className="w-6 h-6 text-gray-400 flex-shrink-0 mt-1" />
          )}
          <div className="flex-1">
            <h3 className="font-semibold text-lg">{title}</h3>
            <p className={`mt-2 text-sm ${enabled ? 'text-green-700' : 'text-gray-600'}`}>
              {description}
            </p>
          </div>
        </div>
      </div>
    );
  }
  ```

- `components/admin/capabilities/DegradeBanner.tsx`:
  ```typescript
  import { AlertTriangle } from 'lucide-react';
  
  export function DegradeBanner() {
    return (
      <div className="p-4 bg-yellow-50 border border-yellow-200 rounded-lg flex items-start gap-3">
        <AlertTriangle className="w-5 h-5 text-yellow-600 flex-shrink-0 mt-0.5" />
        <div>
          <h4 className="font-semibold text-yellow-900">제한된 모드 (Degraded Mode)</h4>
          <p className="text-sm text-yellow-800 mt-1">
            시스템이 제한된 모드로 동작 중입니다. 일부 기능이 비활성화되었습니다.
          </p>
        </div>
      </div>
    );
  }
  ```

### 4-5 단위 테스트 작성
**목표**: 주요 컴포넌트 및 API 호출 검증

**세부 작업**:
- `components/admin/providers/__tests__/ProviderList.test.tsx`:
  ```typescript
  import { render, screen, waitFor } from '@testing-library/react';
  import userEvent from '@testing-library/user-event';
  import { ProviderList } from '../ProviderList';
  import * as providerApi from '@/lib/api/providerApi';
  
  jest.mock('@/lib/api/providerApi');
  
  describe('ProviderList', () => {
    it('should render provider table with data', async () => {
      const mockProviders = [
        {
          id: '1',
          name: 'OpenAI',
          type: 'llm' as const,
          status: 'active' as const,
          modelCount: 5,
          lastTestedAt: new Date().toISOString(),
          avgResponseTimeMs: 150,
          apiKeyUpdatedAt: new Date().toISOString(),
        },
      ];
      
      (providerApi.useProviders as jest.Mock).mockReturnValue({
        data: mockProviders,
        isLoading: false,
        error: null,
      });
      
      render(<ProviderList />);
      
      await waitFor(() => {
        expect(screen.getByText('OpenAI')).toBeInTheDocument();
        expect(screen.getByText('LLM')).toBeInTheDocument();
      });
    });
    
    it('should open health check modal on button click', async () => {
      const user = userEvent.setup();
      // ... test logic
    });
  });
  ```

- `components/admin/capabilities/__tests__/CapabilitiesDashboard.test.tsx`:
  ```typescript
  import { render, screen, waitFor } from '@testing-library/react';
  import { CapabilitiesDashboard } from '../CapabilitiesDashboard';
  import * as capabilitiesApi from '@/lib/api/capabilitiesApi';
  
  jest.mock('@/lib/api/capabilitiesApi');
  
  describe('CapabilitiesDashboard', () => {
    it('should render capability cards', async () => {
      const mockCapabilities = {
        ragEnabled: true,
        embeddingAvailable: true,
        embeddingModelName: 'ada',
        embeddingDimension: 1536,
        llmProvidersActive: 2,
        llmProvidersTotal: 3,
        chunkingStrategies: ['recursive', 'semantic'],
        degradedMode: false,
        lastCheckedAt: new Date().toISOString(),
      };
      
      (capabilitiesApi.useCapabilities as jest.Mock).mockReturnValue({
        data: mockCapabilities,
        isLoading: false,
      });
      
      render(<CapabilitiesDashboard />);
      
      await waitFor(() => {
        expect(screen.getByText(/RAG/)).toBeInTheDocument();
        expect(screen.getByText(/Embedding 모델/)).toBeInTheDocument();
      });
    });
    
    it('should show degrade banner when degraded mode is true', async () => {
      const mockCapabilities = {
        degradedMode: true,
        // ... other properties
      };
      
      (capabilitiesApi.useCapabilities as jest.Mock).mockReturnValue({
        data: mockCapabilities,
        isLoading: false,
      });
      
      render(<CapabilitiesDashboard />);
      
      expect(screen.getByText(/제한된 모드/)).toBeInTheDocument();
    });
  });
  ```

## 5. 산출물

### 필수 산출물
1. **API 클라이언트 코드**:
   - `lib/api/providerApi.ts`
   - `lib/api/capabilitiesApi.ts`

2. **페이지 컴포넌트**:
   - `app/admin/providers/page.tsx`
   - `app/admin/capabilities/page.tsx`

3. **기능 컴포넌트**:
   - `components/admin/providers/ProviderList.tsx`
   - `components/admin/providers/HealthCheckModal.tsx`
   - `components/admin/providers/DefaultModelSelector.tsx`
   - `components/admin/capabilities/CapabilitiesDashboard.tsx`
   - `components/admin/capabilities/CapabilityCard.tsx`
   - `components/admin/capabilities/DegradeBanner.tsx`

4. **단위 테스트**:
   - `components/admin/providers/__tests__/ProviderList.test.tsx`
   - `components/admin/providers/__tests__/HealthCheckModal.test.tsx`
   - `components/admin/capabilities/__tests__/CapabilitiesDashboard.test.tsx`

5. **검수보고서**:
   - UI 리뷰 기록 (5회)
   - 개선 사항 누적 기록

6. **보안취약점검사보고서**:
   - API 호출 보안 검사
   - 민감정보 노출 검사 (API 키)
   - XSS/CSRF 검사

## 6. 완료 기준

- [ ] 프로바이더 목록이 AdminTable로 렌더링됨
- [ ] "연결 테스트" 버튼 클릭 시 health-check API 호출 및 모달 표시
- [ ] "기본 모델 지정" 드롭다운 작동
- [ ] Capabilities 대시보드가 4개 카드로 표시됨
- [ ] Degrade 배너가 조건부로 표시됨
- [ ] Refresh 버튼이 캐시 무효화 및 데이터 갱신
- [ ] 반응형 디자인: 640px, 768px, 1024px 브레이크포인트 정상 동작
- [ ] WCAG 2.1 AA 준수
- [ ] 단위 테스트 커버리지 > 80%
- [ ] 5회 UI 리뷰 완료 및 개선사항 반영
- [ ] 보안취약점검사 완료

## 7. 작업 지침

### 7-1 API 응답 처리
- unwrapEnvelope 활용하여 응답 정규화
- 에러 상황 처리: try-catch, onError 콜백
- 로딩/에러/빈 상태별 UI 표시

### 7-2 React Query 활용
- useQuery로 데이터 캐싱 및 자동 갱신
- useMutation으로 POST/PUT 작업
- queryClient.invalidateQueries로 캐시 무효화
- refetchInterval로 주기적 갱신 (30초)

### 7-3 테스트 작성
- 각 컴포넌트의 렌더링, 상호작용, 상태 변화 검증
- API 호출 모킹 (jest.mock)
- 비동기 작업 처리 (waitFor, act)

### 7-4 UI/UX 리뷰 체크리스트
1. 레이아웃 구조 (테이블, 카드, 버튼 배치)
2. 반응형 (640px, 768px, 1024px)
3. 색상/타이포그래피 (WCAG AA)
4. 인터랙션 (호버, 클릭, 로딩 상태)
5. 접근성 (키보드 네비게이션, 스크린 리더)

### 7-5 에러 메시지
- 사용자 친화적: "프로바이더 연결 실패. 잠시 후 다시 시도하세요."
- 기술 정보 포함: `error.message` 로깅
- Toast 알림으로 시각적 피드백

### 7-6 보안 주의사항
- API 키는 URL에 포함하지 말 것 (request body 사용)
- 민감정보 console.log 금지
- CSRF 토큰 포함 (필요시)
- 인증 헤더 자동 추가 (fetch wrapper 활용)

---

**작업 예상 소요 시간**: 4~6일
**의존 관계**: FG6.1 task6-1 완료 필수
