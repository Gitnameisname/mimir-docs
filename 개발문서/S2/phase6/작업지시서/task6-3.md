# FG6.1 작업지시서 3: 프롬프트 관리 페이지 (CRUD + 버전 이력 + A/B 테스트)

## 1. 작업 목적
Phase 1에서 구축된 Prompt Registry를 Admin 콘솔에 노출하여, 관리자가 프롬프트를 생성/수정/활성화하고, 버전 이력을 추적하며, A/B 테스트 설정을 통해 프롬프트 최적화를 관리할 수 있는 통합 UI를 구축합니다.

## 2. 작업 범위

### 포함 사항
- **프롬프트 관리 페이지** (`/admin/prompts`):
  - 프롬프트 목록: AdminTable
    - 이름, 설명, 활성 버전, 생성일, 수정일
    - 정렬/필터/페이지네이션
  - "프롬프트 생성" 버튼 → AdminModal (이름, 설명, 초기 내용)
  - 각 프롬프트 행 클릭 → 상세 보기 페이지

- **프롬프트 상세 페이지** (`/admin/prompts/[id]`):
  - 기본 정보: 이름, 설명, 카테고리
  - 버전 이력 Timeline:
    - 버전 번호, 생성일, 수정자, 내용 미리보기 (truncated)
    - "이 버전 활성화" 버튼
  - 활성 버전 표시
  - "프롬프트 편집" 버튼 → 편집 모달
  - A/B 테스트 설정 섹션

- **프롬프트 편집 모달** (PromptEditor):
  - 코드 에디터 스타일 (Monaco Editor 또는 CodeMirror)
  - Markdown 문법 하이라이팅
  - 실시간 미리보기 (선택)
  - "저장" 클릭 → 새 버전 자동 생성
  - 저장 시 버전 메타데이터 기록 (수정자, 변경 사항 요약)

- **버전 이력 Timeline** (PromptVersionTimeline):
  - 세로 타임라인으로 버전 표시
  - 각 버전:
    - 버전 번호 (v1, v2, ...)
    - 생성일시
    - 수정자 (사용자명)
    - 내용 미리보기 (처음 200자)
    - "이 버전 활성화" 버튼
    - "버전 상세보기" 링크 (확인용)

- **A/B 테스트 설정**:
  - 2개 버전 선택 (A, B)
  - 트래픽 분할 비율 (0~100%)
  - "A/B 테스트 시작" 버튼
  - 활성 A/B 테스트 표시
  - "종료" 버튼

- **API 통합**:
  - GET /prompts (목록)
  - GET /prompts/{id} (상세)
  - POST /prompts (생성)
  - PUT /prompts/{id} (프롬프트 정보 수정)
  - POST /prompts/{id}/versions (새 버전 생성)
  - PUT /prompts/{id}/active-version (활성 버전 지정)
  - GET /prompts/{id}/versions (버전 목록)
  - POST /prompts/{id}/ab-test (A/B 테스트 설정)
  - DELETE /prompts/{id}/ab-test (A/B 테스트 종료)
  - unwrapEnvelope 활용

- **단위 테스트**:
  - PromptList 렌더링 및 생성/편집
  - PromptDetail 버전 이력
  - PromptEditor 저장
  - A/B 테스트 설정

- **UI/UX**:
  - 반응형 (640px, 768px, 1024px)
  - WCAG 2.1 AA 접근성
  - 로딩/에러 상태

### 제외 사항
- 프로바이더 관리 (FG6.2)
- 비용·사용량 (FG6.4)
- Admin 레이아웃 (FG6.1 task6-1)
- 외부 에디터 라이브러리 커스터마이징

## 3. 선행 조건
- FG6.1 task6-1 완료
- Phase 1 Prompt Registry API 구현 완료
- React Query 설정 완료
- (선택) Monaco Editor 또는 CodeMirror 라이브러리 설치

## 4. 주요 작업 항목

### 4-1 프롬프트 API 클라이언트 구현
**목표**: Prompt Registry API 추상화

**세부 작업**:
- `lib/api/promptApi.ts` 생성:
  ```typescript
  import { unwrapEnvelope } from '@/lib/api/utils';
  import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
  
  export interface Prompt {
    id: string;
    name: string;
    description: string;
    category: string;
    activeVersionId: string;
    createdAt: string;
    updatedAt: string;
    createdBy: string;
    updatedBy: string;
  }
  
  export interface PromptVersion {
    id: string;
    promptId: string;
    versionNumber: number;
    content: string;
    createdAt: string;
    createdBy: string;
    changeSummary?: string; // 변경 사항 요약
    isActive: boolean;
  }
  
  export interface ABTest {
    id: string;
    promptId: string;
    versionAId: string;
    versionBId: string;
    trafficSplitPercentageA: number; // 0~100
    trafficSplitPercentageB: number;
    startedAt: string;
    endedAt?: string;
    status: 'active' | 'completed';
  }
  
  // 프롬프트 목록
  export function usePrompts() {
    return useQuery<Prompt[], Error>({
      queryKey: ['prompts'],
      queryFn: async () => {
        const res = await fetch('/api/prompts');
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
    });
  }
  
  // 프롬프트 상세
  export function usePrompt(id: string) {
    return useQuery<Prompt, Error>({
      queryKey: ['prompts', id],
      queryFn: async () => {
        const res = await fetch(`/api/prompts/${id}`);
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
    });
  }
  
  // 버전 목록
  export function usePromptVersions(promptId: string) {
    return useQuery<PromptVersion[], Error>({
      queryKey: ['prompts', promptId, 'versions'],
      queryFn: async () => {
        const res = await fetch(`/api/prompts/${promptId}/versions`);
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
    });
  }
  
  // 프롬프트 생성
  export function useCreatePromptMutation() {
    const queryClient = useQueryClient();
    return useMutation<
      Prompt,
      Error,
      { name: string; description: string; content: string; category: string }
    >({
      mutationFn: async (data) => {
        const res = await fetch('/api/prompts', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(data),
        });
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
      onSuccess: () => {
        queryClient.invalidateQueries({ queryKey: ['prompts'] });
      },
    });
  }
  
  // 새 버전 생성 (프롬프트 편집)
  export function useCreateVersionMutation(promptId: string) {
    const queryClient = useQueryClient();
    return useMutation<
      PromptVersion,
      Error,
      { content: string; changeSummary?: string }
    >({
      mutationFn: async (data) => {
        const res = await fetch(`/api/prompts/${promptId}/versions`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(data),
        });
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
      onSuccess: () => {
        queryClient.invalidateQueries({
          queryKey: ['prompts', promptId, 'versions'],
        });
      },
    });
  }
  
  // 활성 버전 변경
  export function useSetActiveVersionMutation(promptId: string) {
    const queryClient = useQueryClient();
    return useMutation<Prompt, Error, { versionId: string }>({
      mutationFn: async (data) => {
        const res = await fetch(`/api/prompts/${promptId}/active-version`, {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(data),
        });
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
      onSuccess: () => {
        queryClient.invalidateQueries({ queryKey: ['prompts', promptId] });
        queryClient.invalidateQueries({
          queryKey: ['prompts', promptId, 'versions'],
        });
      },
    });
  }
  
  // A/B 테스트 설정
  export function useSetupABTestMutation(promptId: string) {
    const queryClient = useQueryClient();
    return useMutation<
      ABTest,
      Error,
      {
        versionAId: string;
        versionBId: string;
        trafficSplitPercentageA: number;
      }
    >({
      mutationFn: async (data) => {
        const res = await fetch(`/api/prompts/${promptId}/ab-test`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            ...data,
            trafficSplitPercentageB: 100 - data.trafficSplitPercentageA,
          }),
        });
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
      onSuccess: () => {
        queryClient.invalidateQueries({
          queryKey: ['prompts', promptId, 'ab-test'],
        });
      },
    });
  }
  
  // A/B 테스트 조회
  export function useABTest(promptId: string) {
    return useQuery<ABTest | null, Error>({
      queryKey: ['prompts', promptId, 'ab-test'],
      queryFn: async () => {
        const res = await fetch(`/api/prompts/${promptId}/ab-test`);
        if (res.status === 404) return null;
        const envelope = await res.json();
        return unwrapEnvelope(envelope).data;
      },
    });
  }
  
  // A/B 테스트 종료
  export function useEndABTestMutation(promptId: string) {
    const queryClient = useQueryClient();
    return useMutation<void, Error>({
      mutationFn: async () => {
        await fetch(`/api/prompts/${promptId}/ab-test`, { method: 'DELETE' });
      },
      onSuccess: () => {
        queryClient.invalidateQueries({
          queryKey: ['prompts', promptId, 'ab-test'],
        });
      },
    });
  }
  ```

### 4-2 프롬프트 목록 페이지 구현
**목표**: 프롬프트 CRUD UI 구축

**세부 작업**:
- `app/admin/prompts/page.tsx`:
  ```typescript
  import { PromptList } from '@/components/admin/prompts/PromptList';
  
  export default function PromptsPage() {
    return (
      <div className="space-y-8">
        <div>
          <h1 className="text-2xl font-bold text-gray-900">프롬프트 관리</h1>
          <p className="mt-1 text-gray-600">
            LLM 프롬프트를 생성, 수정, 버전 관리하고 A/B 테스트를 설정합니다.
          </p>
        </div>
        
        <PromptList />
      </div>
    );
  }
  ```

- `components/admin/prompts/PromptList.tsx`:
  ```typescript
  import { useState, useMemo } from 'react';
  import Link from 'next/link';
  import { AdminTable } from '@/components/admin/AdminTable';
  import { LoadingSkeleton, ErrorState, EmptyState } from '@/components/admin';
  import { CreatePromptModal } from './CreatePromptModal';
  import { usePrompts } from '@/lib/api/promptApi';
  import { ColumnDef } from '@tanstack/react-table';
  
  export function PromptList() {
    const { data: prompts, isLoading, error } = usePrompts();
    const [isCreateOpen, setIsCreateOpen] = useState(false);
    
    const columns = useMemo<ColumnDef<Prompt>[]>(
      () => [
        {
          accessorKey: 'name',
          header: '이름',
          cell: (info) => {
            const prompt = info.row.original;
            return (
              <Link
                href={`/admin/prompts/${prompt.id}`}
                className="font-medium text-blue-600 hover:text-blue-700 hover:underline"
              >
                {info.getValue()}
              </Link>
            );
          },
        },
        {
          accessorKey: 'description',
          header: '설명',
          cell: (info) => {
            const desc = info.getValue() as string;
            return (
              <span className="text-gray-600 line-clamp-2">
                {desc || '-'}
              </span>
            );
          },
        },
        {
          accessorKey: 'activeVersionId',
          header: '활성 버전',
          cell: (info) => {
            const versionId = info.getValue() as string;
            return <span className="font-mono text-sm">{versionId.slice(0, 8)}...</span>;
          },
        },
        {
          accessorKey: 'createdAt',
          header: '생성일',
          cell: (info) => {
            const date = info.getValue() as string;
            return (
              <span className="text-sm text-gray-600">
                {new Date(date).toLocaleDateString('ko-KR')}
              </span>
            );
          },
        },
        {
          accessorKey: 'updatedAt',
          header: '수정일',
          cell: (info) => {
            const date = info.getValue() as string;
            return (
              <span className="text-sm text-gray-600">
                {new Date(date).toLocaleDateString('ko-KR')}
              </span>
            );
          },
        },
      ],
      []
    );
    
    if (isLoading) return <LoadingSkeleton rows={5} />;
    if (error) return <ErrorState error={error} />;
    if (!prompts || prompts.length === 0) {
      return (
        <>
          <button
            onClick={() => setIsCreateOpen(true)}
            className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
          >
            프롬프트 생성
          </button>
          <EmptyState message="생성된 프롬프트가 없습니다." />
          <CreatePromptModal
            isOpen={isCreateOpen}
            onClose={() => setIsCreateOpen(false)}
          />
        </>
      );
    }
    
    return (
      <>
        <div className="flex justify-end mb-4">
          <button
            onClick={() => setIsCreateOpen(true)}
            className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
          >
            프롬프트 생성
          </button>
        </div>
        
        <AdminTable<Prompt>
          columns={columns}
          data={prompts}
          isLoading={isLoading}
          pageSize={10}
        />
        
        <CreatePromptModal
          isOpen={isCreateOpen}
          onClose={() => setIsCreateOpen(false)}
        />
      </>
    );
  }
  ```

### 4-3 프롬프트 생성 모달 구현
**목표**: 새 프롬프트 생성 UI

**세부 작업**:
- `components/admin/prompts/CreatePromptModal.tsx`:
  ```typescript
  import { AdminModal } from '@/components/admin/AdminModal';
  import { AdminForm } from '@/components/admin/AdminForm';
  import { useCreatePromptMutation } from '@/lib/api/promptApi';
  import { useAdminToast } from '@/hooks/useAdminToast';
  import { z } from 'zod';
  
  const createPromptSchema = z.object({
    name: z.string().min(1, '이름은 필수입니다'),
    description: z.string(),
    category: z.string(),
    content: z.string().min(1, '내용은 필수입니다'),
  });
  
  interface CreatePromptModalProps {
    isOpen: boolean;
    onClose: () => void;
  }
  
  export function CreatePromptModal({ isOpen, onClose }: CreatePromptModalProps) {
    const { mutate: createPrompt, isPending } = useCreatePromptMutation();
    const toast = useAdminToast();
    
    const handleSubmit = async (data: z.infer<typeof createPromptSchema>) => {
      createPrompt(data, {
        onSuccess: () => {
          toast.success('프롬프트가 생성되었습니다');
          onClose();
        },
        onError: (error) => {
          toast.error('생성 실패: ' + error.message);
        },
      });
    };
    
    return (
      <AdminModal
        isOpen={isOpen}
        onClose={onClose}
        title="프롬프트 생성"
        onSubmit={() => {}} // Form으로 처리
        submitLabel="생성"
        isLoading={isPending}
      >
        <AdminForm
          schema={createPromptSchema}
          onSubmit={handleSubmit}
          defaultValues={{
            name: '',
            description: '',
            category: '',
            content: '',
          }}
        >
          {(form) => (
            <div className="space-y-4">
              <div>
                <label className="block text-sm font-medium text-gray-700">이름</label>
                <input
                  {...form.register('name')}
                  type="text"
                  className="mt-1 w-full px-3 py-2 border border-gray-300 rounded-lg"
                  placeholder="예: GPT-4 생성 템플릿"
                />
                {form.formState.errors.name && (
                  <p className="text-red-600 text-sm mt-1">
                    {form.formState.errors.name.message}
                  </p>
                )}
              </div>
              
              <div>
                <label className="block text-sm font-medium text-gray-700">설명</label>
                <textarea
                  {...form.register('description')}
                  className="mt-1 w-full px-3 py-2 border border-gray-300 rounded-lg"
                  placeholder="프롬프트의 목적을 설명하세요"
                  rows={3}
                />
              </div>
              
              <div>
                <label className="block text-sm font-medium text-gray-700">카테고리</label>
                <input
                  {...form.register('category')}
                  type="text"
                  className="mt-1 w-full px-3 py-2 border border-gray-300 rounded-lg"
                  placeholder="예: generation, summarization"
                />
              </div>
              
              <div>
                <label className="block text-sm font-medium text-gray-700">내용</label>
                <textarea
                  {...form.register('content')}
                  className="mt-1 w-full px-3 py-2 border border-gray-300 rounded-lg font-mono text-sm"
                  placeholder="프롬프트 내용을 입력하세요"
                  rows={6}
                />
                {form.formState.errors.content && (
                  <p className="text-red-600 text-sm mt-1">
                    {form.formState.errors.content.message}
                  </p>
                )}
              </div>
            </div>
          )}
        </AdminForm>
      </AdminModal>
    );
  }
  ```

### 4-4 프롬프트 상세 페이지 및 편집 구현
**목표**: 버전 관리 및 A/B 테스트 설정

**세부 작업**:
- `app/admin/prompts/[id]/page.tsx`:
  ```typescript
  import { PromptDetail } from '@/components/admin/prompts/PromptDetail';
  
  export default function PromptDetailPage({
    params,
  }: {
    params: { id: string };
  }) {
    return <PromptDetail promptId={params.id} />;
  }
  ```

- `components/admin/prompts/PromptDetail.tsx`:
  ```typescript
  import { useState } from 'react';
  import { usePrompt } from '@/lib/api/promptApi';
  import { LoadingSkeleton, ErrorState } from '@/components/admin';
  import { PromptVersionTimeline } from './PromptVersionTimeline';
  import { PromptEditor } from './PromptEditor';
  import { ABTestSection } from './ABTestSection';
  
  interface PromptDetailProps {
    promptId: string;
  }
  
  export function PromptDetail({ promptId }: PromptDetailProps) {
    const { data: prompt, isLoading, error } = usePrompt(promptId);
    const [isEditOpen, setIsEditOpen] = useState(false);
    
    if (isLoading) return <LoadingSkeleton rows={5} />;
    if (error) return <ErrorState error={error} />;
    if (!prompt) return null;
    
    return (
      <div className="space-y-8">
        {/* 헤더 */}
        <div>
          <h1 className="text-2xl font-bold text-gray-900">{prompt.name}</h1>
          <p className="mt-1 text-gray-600">{prompt.description}</p>
          <p className="mt-2 text-sm text-gray-500">
            수정: {new Date(prompt.updatedAt).toLocaleString('ko-KR')} by {prompt.updatedBy}
          </p>
        </div>
        
        {/* 편집 버튼 */}
        <button
          onClick={() => setIsEditOpen(true)}
          className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
        >
          프롬프트 편집
        </button>
        
        {/* 버전 이력 */}
        <PromptVersionTimeline promptId={promptId} activeVersionId={prompt.activeVersionId} />
        
        {/* A/B 테스트 */}
        <ABTestSection promptId={promptId} />
        
        {/* 편집 모달 */}
        <PromptEditor
          promptId={promptId}
          isOpen={isEditOpen}
          onClose={() => setIsEditOpen(false)}
        />
      </div>
    );
  }
  ```

### 4-5 버전 이력 Timeline 구현
**목표**: 프롬프트 버전 관리 UI

**세부 작업**:
- `components/admin/prompts/PromptVersionTimeline.tsx`:
  ```typescript
  import { usePromptVersions, useSetActiveVersionMutation } from '@/lib/api/promptApi';
  import { useAdminToast } from '@/hooks/useAdminToast';
  import { LoadingSkeleton } from '@/components/admin';
  import { CheckCircle } from 'lucide-react';
  
  interface PromptVersionTimelineProps {
    promptId: string;
    activeVersionId: string;
  }
  
  export function PromptVersionTimeline({
    promptId,
    activeVersionId,
  }: PromptVersionTimelineProps) {
    const { data: versions, isLoading } = usePromptVersions(promptId);
    const { mutate: setActiveVersion } = useSetActiveVersionMutation(promptId);
    const toast = useAdminToast();
    
    if (isLoading) return <LoadingSkeleton rows={3} />;
    if (!versions || versions.length === 0) return null;
    
    const sortedVersions = [...versions].sort(
      (a, b) => new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime()
    );
    
    const handleSetActive = (versionId: string) => {
      setActiveVersion(
        { versionId },
        {
          onSuccess: () => {
            toast.success('활성 버전이 변경되었습니다');
          },
          onError: (error) => {
            toast.error('변경 실패: ' + error.message);
          },
        }
      );
    };
    
    return (
      <div className="bg-white rounded-lg border border-gray-200 p-6">
        <h2 className="text-lg font-semibold mb-6">버전 이력</h2>
        
        <div className="space-y-6">
          {sortedVersions.map((version, index) => (
            <div key={version.id} className="flex gap-4">
              {/* Timeline 점 */}
              <div className="flex flex-col items-center">
                <div
                  className={`w-3 h-3 rounded-full ${
                    version.id === activeVersionId
                      ? 'bg-blue-600'
                      : 'bg-gray-300'
                  }`}
                />
                {index < sortedVersions.length - 1 && (
                  <div className="w-0.5 h-16 bg-gray-300 my-2" />
                )}
              </div>
              
              {/* 버전 카드 */}
              <div className="flex-1 bg-gray-50 rounded-lg p-4">
                <div className="flex items-start justify-between">
                  <div className="flex-1">
                    <div className="flex items-center gap-2">
                      <h3 className="font-semibold">v{version.versionNumber}</h3>
                      {version.id === activeVersionId && (
                        <span className="inline-flex items-center gap-1 px-2 py-1 bg-blue-100 text-blue-700 text-xs font-medium rounded">
                          <CheckCircle className="w-3 h-3" />
                          활성
                        </span>
                      )}
                    </div>
                    <p className="text-sm text-gray-600 mt-1">
                      {new Date(version.createdAt).toLocaleString('ko-KR')}
                    </p>
                    <p className="text-sm text-gray-600">by {version.createdBy}</p>
                    {version.changeSummary && (
                      <p className="text-sm text-gray-700 mt-2">{version.changeSummary}</p>
                    )}
                  </div>
                  
                  {version.id !== activeVersionId && (
                    <button
                      onClick={() => handleSetActive(version.id)}
                      className="px-3 py-1 text-sm font-medium text-blue-600 hover:bg-blue-50 rounded"
                    >
                      활성화
                    </button>
                  )}
                </div>
                
                {/* 내용 미리보기 */}
                <div className="mt-4 bg-white rounded border border-gray-300 p-3">
                  <pre className="text-xs text-gray-600 font-mono whitespace-pre-wrap overflow-hidden">
                    {version.content.slice(0, 200)}
                    {version.content.length > 200 ? '...' : ''}
                  </pre>
                </div>
              </div>
            </div>
          ))}
        </div>
      </div>
    );
  }
  ```

### 4-6 프롬프트 편집 모달 및 에디터 구현
**목표**: 코드 에디터 스타일 프롬프트 편집

**세부 작업**:
- `components/admin/prompts/PromptEditor.tsx`:
  ```typescript
  import { useState } from 'react';
  import { AdminModal } from '@/components/admin/AdminModal';
  import { usePrompt, useCreateVersionMutation } from '@/lib/api/promptApi';
  import { useAdminToast } from '@/hooks/useAdminToast';
  import { LoadingSkeleton } from '@/components/admin';
  
  interface PromptEditorProps {
    promptId: string;
    isOpen: boolean;
    onClose: () => void;
  }
  
  export function PromptEditor({
    promptId,
    isOpen,
    onClose,
  }: PromptEditorProps) {
    const { data: prompt, isLoading } = usePrompt(promptId);
    const { mutate: createVersion, isPending } = useCreateVersionMutation(promptId);
    const [content, setContent] = useState('');
    const [changeSummary, setChangeSummary] = useState('');
    const toast = useAdminToast();
    
    if (isLoading) return <LoadingSkeleton rows={3} />;
    if (!prompt) return null;
    
    const handleSave = () => {
      if (!content.trim()) {
        toast.error('내용을 입력하세요');
        return;
      }
      
      createVersion(
        { content, changeSummary },
        {
          onSuccess: () => {
            toast.success('새 버전이 생성되었습니다');
            setContent('');
            setChangeSummary('');
            onClose();
          },
          onError: (error) => {
            toast.error('저장 실패: ' + error.message);
          },
        }
      );
    };
    
    return (
      <AdminModal
        isOpen={isOpen}
        onClose={onClose}
        title="프롬프트 편집"
        onSubmit={handleSave}
        submitLabel="저장 (새 버전 생성)"
        isLoading={isPending}
      >
        <div className="space-y-4">
          {/* 코드 에디터 */}
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-2">내용</label>
            <textarea
              value={content}
              onChange={(e) => setContent(e.target.value)}
              className="w-full h-80 px-3 py-2 border border-gray-300 rounded-lg font-mono text-sm focus:outline-none focus:border-blue-500"
              placeholder="프롬프트 내용을 입력하세요..."
              spellCheck="false"
            />
            <p className="text-xs text-gray-500 mt-1">
              {content.length} 자 | Markdown 문법 지원
            </p>
          </div>
          
          {/* 변경 사항 요약 */}
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-2">
              변경 사항 요약 (선택)
            </label>
            <input
              type="text"
              value={changeSummary}
              onChange={(e) => setChangeSummary(e.target.value)}
              className="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm"
              placeholder="이 버전의 주요 변경 사항을 설명하세요"
            />
          </div>
        </div>
      </AdminModal>
    );
  }
  ```

### 4-7 A/B 테스트 설정 구현
**목표**: A/B 테스트 관리 UI

**세부 작업**:
- `components/admin/prompts/ABTestSection.tsx`:
  ```typescript
  import { useState } from 'react';
  import { usePromptVersions, useABTest, useSetupABTestMutation, useEndABTestMutation } from '@/lib/api/promptApi';
  import { useAdminToast } from '@/hooks/useAdminToast';
  import { LoadingSkeleton } from '@/components/admin';
  
  interface ABTestSectionProps {
    promptId: string;
  }
  
  export function ABTestSection({ promptId }: ABTestSectionProps) {
    const { data: versions, isLoading: versionsLoading } = usePromptVersions(promptId);
    const { data: activeTest } = useABTest(promptId);
    const { mutate: setupTest } = useSetupABTestMutation(promptId);
    const { mutate: endTest } = useEndABTestMutation(promptId);
    const [versionAId, setVersionAId] = useState('');
    const [versionBId, setVersionBId] = useState('');
    const [trafficSplitA, setTrafficSplitA] = useState(50);
    const toast = useAdminToast();
    
    if (versionsLoading) return <LoadingSkeleton rows={2} />;
    if (!versions || versions.length < 2) {
      return (
        <div className="bg-white rounded-lg border border-gray-200 p-6">
          <p className="text-gray-600">A/B 테스트를 진행하려면 최소 2개의 버전이 필요합니다.</p>
        </div>
      );
    }
    
    const handleSetupTest = () => {
      if (!versionAId || !versionBId) {
        toast.error('두 버전을 모두 선택하세요');
        return;
      }
      if (versionAId === versionBId) {
        toast.error('다른 버전을 선택하세요');
        return;
      }
      
      setupTest(
        { versionAId, versionBId, trafficSplitPercentageA: trafficSplitA },
        {
          onSuccess: () => {
            toast.success('A/B 테스트가 시작되었습니다');
            setVersionAId('');
            setVersionBId('');
          },
          onError: (error) => {
            toast.error('설정 실패: ' + error.message);
          },
        }
      );
    };
    
    const handleEndTest = () => {
      if (confirm('A/B 테스트를 종료하시겠습니까?')) {
        endTest(undefined, {
          onSuccess: () => {
            toast.success('A/B 테스트가 종료되었습니다');
          },
          onError: (error) => {
            toast.error('종료 실패: ' + error.message);
          },
        });
      }
    };
    
    return (
      <div className="bg-white rounded-lg border border-gray-200 p-6">
        <h2 className="text-lg font-semibold mb-6">A/B 테스트</h2>
        
        {activeTest ? (
          <div className="space-y-4">
            <div className="bg-blue-50 border border-blue-200 rounded-lg p-4">
              <p className="font-semibold text-blue-900">활성 테스트 중</p>
              <p className="text-sm text-blue-800 mt-1">
                버전 A {activeTest.trafficSplitPercentageA}% vs 버전 B {activeTest.trafficSplitPercentageB}%
              </p>
            </div>
            <button
              onClick={handleEndTest}
              className="px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700"
            >
              테스트 종료
            </button>
          </div>
        ) : (
          <div className="space-y-4">
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-2">버전 A</label>
                <select
                  value={versionAId}
                  onChange={(e) => setVersionAId(e.target.value)}
                  className="w-full px-3 py-2 border border-gray-300 rounded-lg"
                >
                  <option value="">선택하세요</option>
                  {versions.map((v) => (
                    <option key={v.id} value={v.id}>
                      v{v.versionNumber} ({new Date(v.createdAt).toLocaleDateString('ko-KR')})
                    </option>
                  ))}
                </select>
              </div>
              
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-2">버전 B</label>
                <select
                  value={versionBId}
                  onChange={(e) => setVersionBId(e.target.value)}
                  className="w-full px-3 py-2 border border-gray-300 rounded-lg"
                >
                  <option value="">선택하세요</option>
                  {versions.map((v) => (
                    <option key={v.id} value={v.id}>
                      v{v.versionNumber} ({new Date(v.createdAt).toLocaleDateString('ko-KR')})
                    </option>
                  ))}
                </select>
              </div>
            </div>
            
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-2">
                트래픽 분할 (버전 A: {trafficSplitA}% / 버전 B: {100 - trafficSplitA}%)
              </label>
              <input
                type="range"
                min="0"
                max="100"
                value={trafficSplitA}
                onChange={(e) => setTrafficSplitA(parseInt(e.target.value))}
                className="w-full"
              />
            </div>
            
            <button
              onClick={handleSetupTest}
              className="w-full px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
            >
              A/B 테스트 시작
            </button>
          </div>
        )}
      </div>
    );
  }
  ```

### 4-8 단위 테스트 작성
**세부 작업**: React Testing Library를 사용한 주요 컴포넌트 테스트
- `components/admin/prompts/__tests__/PromptList.test.tsx`
- `components/admin/prompts/__tests__/PromptDetail.test.tsx`
- `components/admin/prompts/__tests__/ABTestSection.test.tsx`

## 5. 산출물

### 필수 산출물
1. **API 클라이언트**: `lib/api/promptApi.ts`
2. **페이지 컴포넌트**:
   - `app/admin/prompts/page.tsx`
   - `app/admin/prompts/[id]/page.tsx`
3. **기능 컴포넌트**:
   - `components/admin/prompts/PromptList.tsx`
   - `components/admin/prompts/CreatePromptModal.tsx`
   - `components/admin/prompts/PromptDetail.tsx`
   - `components/admin/prompts/PromptVersionTimeline.tsx`
   - `components/admin/prompts/PromptEditor.tsx`
   - `components/admin/prompts/ABTestSection.tsx`
4. **단위 테스트** (3개 이상)
5. **검수보고서** (5회 UI 리뷰)
6. **보안취약점검사보고서**

## 6. 완료 기준

- [ ] 프롬프트 목록 렌더링 및 CRUD 작동
- [ ] 버전 Timeline 표시 및 활성 버전 변경
- [ ] 프롬프트 편집 모달에서 새 버전 생성
- [ ] A/B 테스트 설정/종료 작동
- [ ] 반응형 디자인 정상 동작
- [ ] WCAG 2.1 AA 준수
- [ ] 단위 테스트 커버리지 > 80%
- [ ] 5회 UI 리뷰 완료
- [ ] 보안취약점검사 완료

## 7. 작업 지침

### 7-1 S1 원칙 준수
- 프롬프트 데이터 조회/수정은 API 계층에서만
- UI는 데이터 표시 및 폼 입력만 담당

### 7-2 에러 처리
- API 호출 실패 시 Toast 알림
- 네트워크 에러, 검증 에러 구분

### 7-3 성능
- React Query로 캐싱 및 자동 갱신
- useMemo로 컬럼 정의 메모이제이션

---

**작업 예상 소요 시간**: 5~7일
