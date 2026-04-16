# FG6.3 작업지시서 (Task 6-9): 골든셋 관리 + 평가 결과 대시보드 (스켈레톤)

**최종 수정**: 2026-04-17  
**상태**: 작업 지시서 초안

---

## 1. 작업 목적

Phase 6 통합 관리 콘솔 단계에서 평가(Evaluation) 기능의 스켈레톤 UI를 구축합니다. Phase 7 평가 인프라 백엔드가 미완성 상태이므로, 다음을 먼저 구현합니다:

- **골든셋 관리 페이지** (`/admin/golden-sets`): 평가 기준 데이터셋 목록, 상세 조회, 버전 관리, Import/Export
- **평가 결과 대시보드** (`/admin/evaluations`): 평가 실행 결과 요약, 지표별 현황, 추이 그래프, 필터, CI 게이트 상태

목 데이터(mock data)로 UI/UX를 검증하고, Phase 7 API 완료 후 엔드포인트만 교체하면 즉시 운영 가능한 구조로 설계합니다.

---

## 2. 작업 범위

### 포함 항목

1. **프론트엔드 페이지 및 컴포넌트**
   - `/admin/golden-sets`: 골든셋 목록, 생성 모달, 상세 페이지
   - `/admin/evaluations`: 평가 대시보드, 상세 결과 조회

2. **UI 컴포넌트**
   - AdminTable (TanStack Table 기반)
   - 모달 컴포넌트 ("골든셋 생성")
   - 요약 카드 (평가 요약)
   - 지표 현황 컴포넌트 (Faithfulness, Answer Relevance, Citation-present)
   - LineChart (recharts, 30일 추이)
   - 필터 UI (기간, 프롬프트 버전, 모델)
   - CI 게이트 상태 배지
   - 버전 이력 타임라인

3. **Mock API 서비스**
   - `services/mockApi/goldenSetMockApi.ts`
   - `services/mockApi/evaluationMockApi.ts`

4. **목 데이터**
   - 골든셋 3~5개 (각 5~10개 Q&A 항목)
   - 평가 실행 결과 (최근 7일)

5. **라우팅 및 네비게이션**
   - `/admin` 사이드바에 "평가·추출" 섹션 추가
   - 메뉴 항목: "골든셋", "평가 결과", "추출 스키마", "추출 검토"

6. **단위 테스트 (Jest)**
   - 골든셋 목록 렌더링
   - 평가 대시보드 렌더링
   - Mock API 호출 검증

7. **접근성**
   - WCAG 2.1 AA 준수
   - 스크린리더 호환성 (ARIA labels)

### 제외 항목

- 추출 스키마 관리 (`/admin/extraction-schemas`) → Task 6-10
- 추출 결과 검토 큐 (`/admin/extraction-review`) → Task 6-10
- 실제 Phase 7 API 연동 (Phase 7 완료 후)
- 골든셋 "항목 추가" 기능 활성화 (Phase 7 후)
- 평가 실행 기능 (Phase 7 후)

---

## 3. 선행 조건

### 기술 스택
- **프론트엔드**: Next.js 16.2.2, React 19, TypeScript 5.x
- **상태 관리**: Zustand
- **테이블**: TanStack Table v8
- **차트**: recharts
- **UI 라이브러리**: shadcn/ui (기존 컴포넌트 재사용)
- **테스트**: Jest, React Testing Library

### 사전 준비
1. 관리자 레이아웃 (`/admin`) 기본 구조 확인
2. Zustand 스토어 (auth 상태) 확인
3. Mock API 구조 검토 (`services/mockApi/` 디렉토리)
4. 개발환경 실행 (`npm run dev`)

### S2 원칙 준수 체크
- **원칙 ①**: DocumentType은 동적으로 표시 (하드코딩 금지)
- **원칙 ⑥**: Scope Profile 기반 접근 제어 (admin 권한만)
- **원칙 ⑦**: 폐쇄망 환경 지원 (mock API는 로컬 동작)

---

## 4. 주요 작업 항목

### 4-1. 골든셋 관리 페이지 구축

#### 4-1-1. 페이지 구조 및 라우팅
- **경로**: `/admin/golden-sets`
- **레이아웃**: 전체 너비, 좌측 패널 없음
- **메인 섹션**:
  - 헤더: "골든셋 관리" 제목 + "골든셋 생성" 버튼 + "Import" 버튼
  - 골든셋 목록 테이블
  - 선택된 골든셋 상세 정보 (우측 패널 또는 탭)

#### 4-1-2. 골든셋 목록 테이블
**컴포넌트**: `components/admin/GoldenSetTable.tsx`

**테이블 칼럼**:
| 칼럼명 | 타입 | 설명 |
|--------|------|------|
| 이름 | string | 골든셋 이름 (클릭 시 상세 페이지 이동) |
| 설명 | string | 골든셋 설명 (최대 100자 요약) |
| 문항 수 | number | Q&A 항목 개수 |
| 최신 버전 | string | 버전 번호 (예: v1.0.2) |
| 생성일 | date | YYYY-MM-DD HH:mm 형식 |
| 작업 | actions | "상세", "수정", "삭제" 메뉴 |

**기능**:
- TanStack Table 기반 (정렬, 페이지네이션)
- 페이지 크기: 10개/페이지
- 행 선택 (체크박스)
- 검색 필터 (이름, 설명)
- 상세 보기 시 우측 패널 또는 모달 열기

**코드 예시**:
```typescript
// components/admin/GoldenSetTable.tsx
import { useMemo } from 'react';
import { createColumnHelper, getCoreRowModel, useReactTable } from '@tanstack/react-table';
import { AdminTable } from '@/components/admin/AdminTable';
import { GoldenSet } from '@/types/evaluation';

const columnHelper = createColumnHelper<GoldenSet>();

const columns = [
  columnHelper.accessor('name', {
    cell: (info) => <span className="font-semibold">{info.getValue()}</span>,
    header: '이름',
  }),
  columnHelper.accessor('description', {
    cell: (info) => <span className="text-sm text-gray-600">{truncate(info.getValue(), 100)}</span>,
    header: '설명',
  }),
  columnHelper.accessor('itemCount', {
    cell: (info) => info.getValue(),
    header: '문항 수',
  }),
  columnHelper.accessor('latestVersion', {
    cell: (info) => <span className="font-mono text-sm">{info.getValue()}</span>,
    header: '최신 버전',
  }),
  columnHelper.accessor('createdAt', {
    cell: (info) => new Date(info.getValue()).toLocaleString('ko-KR'),
    header: '생성일',
  }),
];

export function GoldenSetTable() {
  const [data] = useState<GoldenSet[]>(goldenSetMockData);
  const table = useReactTable({ data, columns, getCoreRowModel: getCoreRowModel() });

  return <AdminTable table={table} />;
}
```

#### 4-1-3. 골든셋 생성 모달
**컴포넌트**: `components/admin/CreateGoldenSetModal.tsx`

**폼 필드**:
- **이름** (필수): 텍스트 입력, 최대 100자
- **설명**: 텍스트 영역, 최대 500자
- **문서 타입** (동적): 드롭다운 (DocumentType 목록에서 동적 로드)

**버튼**: "생성", "취소"

**기능**:
- 폼 검증 (이름 필수)
- 제출 시 Mock API 호출 (`createGoldenSet()`)
- 성공 후 목록 새로고침 + 토스트 메시지
- S2 원칙 ①: DocumentType은 설정에서 로드

**코드 예시**:
```typescript
// components/admin/CreateGoldenSetModal.tsx
import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { goldenSetMockApi } from '@/services/mockApi/goldenSetMockApi';

export function CreateGoldenSetModal({ onClose, onSuccess }: Props) {
  const { register, handleSubmit, formState: { errors } } = useForm({
    defaultValues: { name: '', description: '' }
  });

  const onSubmit = async (data: CreateGoldenSetInput) => {
    try {
      await goldenSetMockApi.createGoldenSet(data);
      onSuccess?.();
      onClose();
    } catch (error) {
      console.error('Failed to create golden set', error);
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        {...register('name', { required: '이름은 필수입니다', maxLength: 100 })}
        placeholder="골든셋 이름"
      />
      {errors.name && <span className="text-red-600">{errors.name.message}</span>}
      <textarea
        {...register('description', { maxLength: 500 })}
        placeholder="설명"
      />
      <button type="submit">생성</button>
    </form>
  );
}
```

#### 4-1-4. 골든셋 상세 페이지
**컴포넌트**: `components/admin/GoldenSetDetail.tsx` 또는 우측 패널

**섹션**:

1. **기본 정보**
   - 이름, 설명, 생성일, 최신 버전, 항목 수
   - 버튼: "수정", "삭제", "버전 이력"

2. **Q&A 항목 테이블**
   - **칼럼**: 질문, 기대 답변, 기대 근거, 기대 Citation
   - **기능**:
     - 정렬, 페이지네이션
     - 검색 필터 (질문 텍스트)
     - "항목 추가" 버튼 (disabled, Phase 7 후 활성화 예정)
     - 행 클릭 → 상세 조회 모달

3. **버전 이력 타임라인**
   - 버전, 수정 일시, 수정자, 변경 내용 요약
   - 버전 클릭 → 해당 버전 Q&A 조회 (mock)

4. **Import/Export 버튼**
   - Export: JSON 다운로드 (골든셋 + Q&A 전체)
   - Import: JSON 파일 업로드 (Phase 7 후 활성)

**코드 예시**:
```typescript
// components/admin/GoldenSetDetail.tsx
import { GoldenSet, QAItem } from '@/types/evaluation';
import { AdminTable } from '@/components/admin/AdminTable';

export function GoldenSetDetail({ goldenSetId }: { goldenSetId: string }) {
  const [goldenSet, setGoldenSet] = useState<GoldenSet | null>(null);
  const [versions, setVersions] = useState<VersionHistory[]>([]);

  useEffect(() => {
    goldenSetMockApi.getGoldenSet(goldenSetId).then(setGoldenSet);
    goldenSetMockApi.getVersionHistory(goldenSetId).then(setVersions);
  }, [goldenSetId]);

  const columns = [
    { accessorKey: 'question', header: '질문' },
    { accessorKey: 'expectedAnswer', header: '기대 답변' },
    { accessorKey: 'expectedContext', header: '기대 근거' },
    { accessorKey: 'expectedCitation', header: '기대 Citation' },
  ];

  return (
    <div className="space-y-6">
      <BasicInfo goldenSet={goldenSet} />
      <QAItemsTable items={goldenSet?.items || []} />
      <VersionTimeline versions={versions} />
      <ImportExportButtons goldenSetId={goldenSetId} />
    </div>
  );
}
```

---

### 4-2. 평가 결과 대시보드 페이지 구축

#### 4-2-1. 페이지 구조 및 라우팅
- **경로**: `/admin/evaluations`
- **레이아웃**: 풀 너비
- **주요 섹션** (위에서 아래):
  1. 평가 요약 카드 영역
  2. 지표별 현황 + 추이 그래프 (2열 또는 1열)
  3. 필터 및 CI 게이트 상태
  4. 상세 실행 결과 테이블

#### 4-2-2. 평가 요약 카드
**컴포넌트**: `components/admin/EvaluationSummaryCards.tsx`

**카드 구성** (3개):
| 카드 | 표시 내용 | 단위 |
|------|----------|------|
| 마지막 실행 | 실행 시간, 항목 수 | 개수 |
| 평균 스코어 | Faithfulness, Answer Relevance, Citation-present 평균 | % |
| 통과율 | 임계값 초과 항목 비율 | % |

**스타일**: 배경색, 큰 숫자, 차이 표시 (상향/하향 화살표)

**코드 예시**:
```typescript
// components/admin/EvaluationSummaryCards.tsx
import { useEffect, useState } from 'react';
import { evaluationMockApi } from '@/services/mockApi/evaluationMockApi';

export function EvaluationSummaryCards() {
  const [summary, setSummary] = useState<EvaluationSummary | null>(null);

  useEffect(() => {
    evaluationMockApi.getLatestSummary().then(setSummary);
  }, []);

  return (
    <div className="grid grid-cols-3 gap-4">
      <Card title="마지막 실행" value={summary?.executionTime} subtext={`${summary?.itemCount}개 항목`} />
      <Card title="평균 스코어" value={`${summary?.avgScore}%`} />
      <Card title="통과율" value={`${summary?.passRate}%`} trend={summary?.trend} />
    </div>
  );
}
```

#### 4-2-3. 지표별 현황 컴포넌트
**컴포넌트**: `components/admin/MetricsStatus.tsx`

**지표 3개** (세로 또는 가로 배열):

1. **Faithfulness**
   - 값: 85.2%
   - 임계값: 80%
   - 통과 배지: ✓ 통과 (초록색)

2. **Answer Relevance**
   - 값: 78.5%
   - 임계값: 75%
   - 통과 배지: ✓ 통과

3. **Citation-present**
   - 값: 72.1%
   - 임계값: 90% (엄격)
   - 통과 배지: ✗ 미통과 (빨간색)

**시각화**: 프로그레스 바 + 숫자 + 배지

**코드 예시**:
```typescript
// components/admin/MetricsStatus.tsx
interface MetricItem {
  name: string;
  value: number;
  threshold: number;
}

export function MetricsStatus({ metrics }: { metrics: MetricItem[] }) {
  return (
    <div className="grid grid-cols-3 gap-4">
      {metrics.map((metric) => (
        <div key={metric.name} className="border rounded p-4">
          <h3 className="font-semibold">{metric.name}</h3>
          <ProgressBar value={metric.value} max={100} />
          <div className="mt-2 text-sm">
            <span>{metric.value.toFixed(1)}%</span>
            <span className="ml-2 text-gray-500">임계값: {metric.threshold}%</span>
          </div>
          <Badge className={metric.value >= metric.threshold ? 'bg-green-100' : 'bg-red-100'}>
            {metric.value >= metric.threshold ? '✓ 통과' : '✗ 미통과'}
          </Badge>
        </div>
      ))}
    </div>
  );
}
```

#### 4-2-4. 추이 그래프 (30일)
**컴포넌트**: `components/admin/EvaluationTrendChart.tsx`

**라이브러리**: recharts (LineChart)

**축**:
- X축: 날짜 (MM-DD)
- Y축: 스코어 (0~100%)

**선**:
- Faithfulness (파란색)
- Answer Relevance (초록색)
- Citation-present (주황색)

**기능**:
- 범례 (클릭하여 선 표시/숨김)
- 툴팁 (호버 시 값 표시)
- 반응형 (모바일 대응)

**코드 예시**:
```typescript
// components/admin/EvaluationTrendChart.tsx
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';

export function EvaluationTrendChart({ data }: { data: TrendPoint[] }) {
  return (
    <ResponsiveContainer width="100%" height={300}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="date" />
        <YAxis />
        <Tooltip />
        <Legend />
        <Line type="monotone" dataKey="faithfulness" stroke="#3b82f6" />
        <Line type="monotone" dataKey="answerRelevance" stroke="#10b981" />
        <Line type="monotone" dataKey="citationPresent" stroke="#f97316" />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

#### 4-2-5. 필터 UI
**컴포넌트**: `components/admin/EvaluationFilters.tsx`

**필터 항목**:
1. **기간**: 날짜 범위 선택 (최근 7일 / 30일 / 커스텀)
2. **프롬프트 버전**: 드롭다운 (최신 / v1.2.0 / v1.1.5 등)
3. **모델**: 드롭다운 (gpt-4 / gpt-3.5-turbo 등)

**기능**:
- 필터 변경 시 즉시 차트/테이블 업데이트
- "초기화" 버튼

**코드 예시**:
```typescript
// components/admin/EvaluationFilters.tsx
export function EvaluationFilters({ onFilterChange }: Props) {
  const [filters, setFilters] = useState<EvaluationFilter>({
    dateRange: 'last30days',
    promptVersion: 'latest',
    model: 'all',
  });

  const handleFilterChange = (key: string, value: any) => {
    const newFilters = { ...filters, [key]: value };
    setFilters(newFilters);
    onFilterChange(newFilters);
  };

  return (
    <div className="flex gap-4 items-center">
      <SelectInput label="기간" options={dateRangeOptions} onChange={(v) => handleFilterChange('dateRange', v)} />
      <SelectInput label="프롬프트 버전" options={promptVersions} onChange={(v) => handleFilterChange('promptVersion', v)} />
      <SelectInput label="모델" options={models} onChange={(v) => handleFilterChange('model', v)} />
      <button onClick={() => setFilters(defaultFilters)}>초기화</button>
    </div>
  );
}
```

#### 4-2-6. CI 게이트 상태 표시
**컴포넌트**: `components/admin/CIGateStatus.tsx`

**표시 내용**:
- "마지막 CI 실행: 2026-04-17 14:30:00"
- 상태 배지:
  - ✓ 통과 (초록색)
  - ✗ 실패 (빨간색)
  - ⏳ 진행 중 (노란색)

**상세 보기**:
- 클릭 시 CI 로그 미리보기 (모달)

**코드 예시**:
```typescript
// components/admin/CIGateStatus.tsx
export function CIGateStatus({ ciStatus }: { ciStatus: CIStatus }) {
  const statusColor = {
    passed: 'bg-green-100 text-green-800',
    failed: 'bg-red-100 text-red-800',
    running: 'bg-yellow-100 text-yellow-800',
  };

  return (
    <div className="flex items-center gap-2 p-4 border rounded">
      <span className="text-sm">마지막 CI 실행:</span>
      <span className="font-mono text-sm">{new Date(ciStatus.timestamp).toLocaleString('ko-KR')}</span>
      <Badge className={statusColor[ciStatus.status]}>
        {ciStatus.status === 'passed' && '✓ 통과'}
        {ciStatus.status === 'failed' && '✗ 실패'}
        {ciStatus.status === 'running' && '⏳ 진행 중'}
      </Badge>
      <button onClick={() => /* show logs */}>상세 보기</button>
    </div>
  );
}
```

#### 4-2-7. 상세 실행 결과 테이블
**컴포넌트**: `components/admin/DetailedResultsTable.tsx`

**테이블 칼럼**:
| 칼럼 | 타입 | 설명 |
|------|------|------|
| Q&A ID | string | 질문 ID (클릭 시 상세) |
| 질문 | string | 질문 텍스트 (최대 100자) |
| Faithfulness | percentage | 점수 + 임계값 표시 |
| Answer Relevance | percentage | 점수 + 임계값 표시 |
| Citation-present | percentage | 점수 + 임계값 표시 |
| 상태 | badge | 통과/미통과 |

**기능**:
- TanStack Table (정렬, 페이지네이션)
- 페이지 크기: 20개
- 행 클릭 → 상세 결과 모달 (각 지표별 상세 정보)

**코드 예시**:
```typescript
// components/admin/DetailedResultsTable.tsx
const columns = [
  columnHelper.accessor('qaId', { header: 'Q&A ID' }),
  columnHelper.accessor('question', {
    cell: (info) => truncate(info.getValue(), 100),
    header: '질문',
  }),
  columnHelper.accessor('faithfulness', {
    cell: (info) => <MetricCell value={info.getValue()} threshold={80} />,
    header: 'Faithfulness',
  }),
  columnHelper.accessor('answerRelevance', {
    cell: (info) => <MetricCell value={info.getValue()} threshold={75} />,
    header: 'Answer Relevance',
  }),
  columnHelper.accessor('citationPresent', {
    cell: (info) => <MetricCell value={info.getValue()} threshold={90} />,
    header: 'Citation-present',
  }),
  columnHelper.accessor('status', {
    cell: (info) => (
      <Badge className={info.getValue() === 'passed' ? 'bg-green-100' : 'bg-red-100'}>
        {info.getValue() === 'passed' ? '통과' : '미통과'}
      </Badge>
    ),
    header: '상태',
  }),
];

export function DetailedResultsTable({ results }: Props) {
  const table = useReactTable({ data: results, columns, getCoreRowModel: getCoreRowModel() });
  return <AdminTable table={table} />;
}
```

---

### 4-3. Mock API 서비스 구축

#### 4-3-1. 골든셋 Mock API
**파일**: `services/mockApi/goldenSetMockApi.ts`

**인터페이스**:
```typescript
// services/mockApi/goldenSetMockApi.ts
export interface GoldenSet {
  id: string;
  name: string;
  description: string;
  itemCount: number;
  latestVersion: string;
  createdAt: string;
  items?: QAItem[];
}

export interface QAItem {
  id: string;
  question: string;
  expectedAnswer: string;
  expectedContext: string;
  expectedCitation: string;
}

export interface VersionHistory {
  version: string;
  createdAt: string;
  createdBy: string;
  changeLog: string;
}

export const goldenSetMockApi = {
  async listGoldenSets(): Promise<GoldenSet[]> {
    // 목 데이터 반환
  },
  async getGoldenSet(id: string): Promise<GoldenSet> {
    // 상세 조회
  },
  async createGoldenSet(input: CreateGoldenSetInput): Promise<GoldenSet> {
    // 생성
  },
  async updateGoldenSet(id: string, input: UpdateGoldenSetInput): Promise<GoldenSet> {
    // 수정
  },
  async deleteGoldenSet(id: string): Promise<void> {
    // 삭제
  },
  async getVersionHistory(id: string): Promise<VersionHistory[]> {
    // 버전 이력
  },
  async exportAsJSON(id: string): Promise<string> {
    // JSON 내보내기
  },
  async importFromJSON(file: File): Promise<GoldenSet> {
    // JSON 가져오기
  },
};
```

**목 데이터** (3개 샘플):
```typescript
// Mock 데이터 예시
const mockGoldenSets: GoldenSet[] = [
  {
    id: 'gs-001',
    name: '고객 서비스 FAQ',
    description: '일반적인 고객 서비스 질문에 대한 답변',
    itemCount: 8,
    latestVersion: 'v1.0.2',
    createdAt: '2026-03-15T10:30:00Z',
    items: [
      {
        id: 'qa-001',
        question: '제품 반품 절차는 어떻게 되나요?',
        expectedAnswer: '구매 후 30일 이내에 반품 가능하며...',
        expectedContext: '반품 정책 문서 제3.1절',
        expectedCitation: 'https://company.com/refund-policy#section-3.1',
      },
      // ... 추가 항목
    ],
  },
  // ... 추가 골든셋
];
```

#### 4-3-2. 평가 Mock API
**파일**: `services/mockApi/evaluationMockApi.ts`

**인터페이스**:
```typescript
// services/mockApi/evaluationMockApi.ts
export interface EvaluationSummary {
  executionTime: string;
  itemCount: number;
  avgScore: number;
  passRate: number;
  trend: 'up' | 'down' | 'stable';
}

export interface MetricItem {
  name: 'Faithfulness' | 'Answer Relevance' | 'Citation-present';
  value: number;
  threshold: number;
  passed: boolean;
}

export interface TrendPoint {
  date: string;
  faithfulness: number;
  answerRelevance: number;
  citationPresent: number;
}

export interface DetailedResult {
  qaId: string;
  question: string;
  faithfulness: number;
  answerRelevance: number;
  citationPresent: number;
  status: 'passed' | 'failed';
}

export interface CIStatus {
  status: 'passed' | 'failed' | 'running';
  timestamp: string;
}

export const evaluationMockApi = {
  async getLatestSummary(): Promise<EvaluationSummary> {
    // 최신 평가 요약
  },
  async getMetrics(): Promise<MetricItem[]> {
    // 지표별 현황
  },
  async getTrendData(days: number = 30): Promise<TrendPoint[]> {
    // 추이 데이터
  },
  async getDetailedResults(filter?: EvaluationFilter): Promise<DetailedResult[]> {
    // 상세 결과
  },
  async getCIStatus(): Promise<CIStatus> {
    // CI 게이트 상태
  },
};
```

**목 데이터** (최근 7일):
```typescript
const mockTrendData: TrendPoint[] = [
  { date: '04-11', faithfulness: 82.1, answerRelevance: 76.5, citationPresent: 68.3 },
  { date: '04-12', faithfulness: 83.5, answerRelevance: 77.2, citationPresent: 70.1 },
  // ... 5일 추가
];

const mockDetailedResults: DetailedResult[] = [
  {
    qaId: 'qa-001',
    question: '제품 반품 절차는?',
    faithfulness: 88,
    answerRelevance: 85,
    citationPresent: 92,
    status: 'passed',
  },
  // ... 추가 항목
];
```

---

### 4-4. 라우팅 및 네비게이션

#### 4-4-1. 라우트 구성
```
/admin
├── /admin/golden-sets (새)
│   └── /admin/golden-sets/[id] (상세)
├── /admin/evaluations (새)
└── (기존) /admin/users, /admin/settings, ...
```

#### 4-4-2. 사이드바 메뉴 추가
**파일**: `components/admin/AdminSidebar.tsx`

**추가 섹션**: "평가·추출"
```
평가·추출
├── 골든셋
├── 평가 결과
├── 추출 스키마 (Task 6-10)
└── 추출 검토 (Task 6-10)
```

**코드 예시**:
```typescript
// components/admin/AdminSidebar.tsx
export const sidebarConfig = [
  // ... 기존 항목
  {
    section: '평가·추출',
    items: [
      { label: '골든셋', href: '/admin/golden-sets', icon: 'BookOpen' },
      { label: '평가 결과', href: '/admin/evaluations', icon: 'BarChart3' },
      { label: '추출 스키마', href: '/admin/extraction-schemas', icon: 'Layers' },
      { label: '추출 검토', href: '/admin/extraction-review', icon: 'CheckCircle' },
    ],
  },
];
```

---

### 4-5. 단위 테스트 (Jest + React Testing Library)

#### 4-5-1. 골든셋 목록 테이블 테스트
**파일**: `components/admin/__tests__/GoldenSetTable.test.tsx`

```typescript
import { render, screen } from '@testing-library/react';
import { GoldenSetTable } from '@/components/admin/GoldenSetTable';

describe('GoldenSetTable', () => {
  it('should render golden sets table with correct columns', () => {
    render(<GoldenSetTable />);
    
    expect(screen.getByText('이름')).toBeInTheDocument();
    expect(screen.getByText('문항 수')).toBeInTheDocument();
    expect(screen.getByText('최신 버전')).toBeInTheDocument();
  });

  it('should display mock data', () => {
    render(<GoldenSetTable />);
    
    expect(screen.getByText('고객 서비스 FAQ')).toBeInTheDocument();
  });

  it('should be accessible', () => {
    const { container } = render(<GoldenSetTable />);
    const table = container.querySelector('table');
    
    expect(table).toHaveAccessibleName();
  });
});
```

#### 4-5-2. 평가 대시보드 테스트
**파일**: `components/admin/__tests__/EvaluationDashboard.test.tsx`

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import { EvaluationDashboard } from '@/components/admin/EvaluationDashboard';

describe('EvaluationDashboard', () => {
  it('should render summary cards', async () => {
    render(<EvaluationDashboard />);
    
    await waitFor(() => {
      expect(screen.getByText('마지막 실행')).toBeInTheDocument();
      expect(screen.getByText('평균 스코어')).toBeInTheDocument();
      expect(screen.getByText('통과율')).toBeInTheDocument();
    });
  });

  it('should render metrics status with correct values', async () => {
    render(<EvaluationDashboard />);
    
    await waitFor(() => {
      expect(screen.getByText(/Faithfulness/)).toBeInTheDocument();
      expect(screen.getByText(/Citation-present/)).toBeInTheDocument();
    });
  });

  it('should render trend chart', async () => {
    render(<EvaluationDashboard />);
    
    await waitFor(() => {
      const svg = screen.getByRole('img', { hidden: true });
      expect(svg).toBeInTheDocument();
    });
  });

  it('should render CI gate status', async () => {
    render(<EvaluationDashboard />);
    
    await waitFor(() => {
      expect(screen.getByText(/마지막 CI 실행/)).toBeInTheDocument();
    });
  });
});
```

---

### 4-6. 접근성 및 WCAG 2.1 AA 준수

#### 4-6-1. 체크리스트
- [ ] 모든 테이블에 `<caption>` 또는 aria-label 추가
- [ ] 폼 필드에 레이블 태그 `<label>` 연결
- [ ] 버튼, 링크에 명확한 텍스트 (아이콘만 사용 금지)
- [ ] 색상만으로 정보 전달 금지 (텍스트/배지 함께 사용)
- [ ] 최소 4.5:1 명도 대비 (AA 기준)
- [ ] 키보드 네비게이션 지원 (Tab, Shift+Tab)
- [ ] 포커스 표시 명확 (outline)
- [ ] 모달 포커스 트래핑

#### 4-6-2. 구현 예시
```typescript
// 테이블에 aria-label 추가
<table
  role="table"
  aria-label="골든셋 목록"
  aria-describedby="golden-set-table-desc"
>

// 배지에 aria-label 추가
<span className="badge bg-green-100" aria-label="평가 통과">
  ✓ 통과
</span>

// 버튼 텍스트 명확화
<button>
  <Icon />
  <span>상세 보기</span>
</button>
```

---

## 5. 산출물

### 5-1. 코드 산출물
1. **페이지**:
   - `app/admin/golden-sets/page.tsx`
   - `app/admin/golden-sets/[id]/page.tsx`
   - `app/admin/evaluations/page.tsx`

2. **컴포넌트**:
   - `components/admin/GoldenSetTable.tsx`
   - `components/admin/GoldenSetDetail.tsx`
   - `components/admin/CreateGoldenSetModal.tsx`
   - `components/admin/VersionTimeline.tsx`
   - `components/admin/EvaluationSummaryCards.tsx`
   - `components/admin/MetricsStatus.tsx`
   - `components/admin/EvaluationTrendChart.tsx`
   - `components/admin/EvaluationFilters.tsx`
   - `components/admin/CIGateStatus.tsx`
   - `components/admin/DetailedResultsTable.tsx`

3. **Mock API**:
   - `services/mockApi/goldenSetMockApi.ts`
   - `services/mockApi/evaluationMockApi.ts`

4. **타입 정의**:
   - `types/evaluation.ts`

5. **테스트**:
   - `components/admin/__tests__/GoldenSetTable.test.tsx`
   - `components/admin/__tests__/GoldenSetDetail.test.tsx`
   - `components/admin/__tests__/EvaluationDashboard.test.tsx`
   - `components/admin/__tests__/EvaluationFilters.test.tsx`

6. **라우팅 및 네비게이션**:
   - 사이드바 메뉴 추가 (기존 `AdminSidebar.tsx` 수정)

### 5-2. 문서 산출물
- **작업 완료 보고서** (task6-9_보고서.md)
  - 구현된 기능 목록
  - UI 스크린샷 (또는 스크린샷 경로)
  - 성능 지표 (로딩 시간, 번들 크기 변화)
  - 테스트 커버리지
  - 알려진 제한사항 (Phase 7 대기 기능)

- **보안 검토 보고서** (task6-9_보안검토.md)
  - Mock API 데이터 격리 확인
  - XSS 방지 (스트립 처리)
  - CSRF 토큰 (해당 사항 확인)
  - 인증/인가 검증

- **UI/UX 리뷰 보고서** (task6-9_UI검토_1차~5차.md)
  - 최소 5회 UI 리뷰 기록
  - 개선 사항 추적
  - 최종 디자인 결과

### 5-3. 하드웨어 요구사항 없음

---

## 6. 완료 기준

### 기능 완료
- [ ] 골든셋 목록 페이지 렌더링 (정렬, 페이지네이션 동작)
- [ ] 골든셋 상세 페이지 (Q&A 항목 테이블, 버전 이력 타임라인, Import/Export)
- [ ] 골든셋 생성 모달 (폼 검증, 제출 동작)
- [ ] 평가 대시보드 (요약 카드, 지표, 그래프, 필터, CI 상태 모두 렌더링)
- [ ] 상세 결과 테이블 (정렬, 페이지네이션 동작)
- [ ] Mock API 모두 구현 및 목 데이터 충분 (테스트 용이)
- [ ] 라우팅 정상 동작
- [ ] 사이드바 메뉴 추가 및 네비게이션 동작

### 테스트 완료
- [ ] 단위 테스트 성공율 100% (최소 8개 테스트)
- [ ] 접근성 검증 (WCAG 2.1 AA 준수 확인)
- [ ] E2E 테스트 (필수 플로우: 페이지 진입 → 데이터 로드 → 필터 적용)

### 성능
- [ ] Lighthouse 성능 점수 ≥80
- [ ] First Contentful Paint (FCP) <1.5s
- [ ] 번들 크기 증가 <100KB (gzipped)

### 문서
- [ ] 코드 주석 (복잡한 로직에만)
- [ ] 타입 정의 완전 (any 금지)
- [ ] 작업 완료 보고서 작성

---

## 7. 작업 지침

### 7-1. 개발 환경 설정
1. `npm run dev` 실행 후 `http://localhost:3000/admin/golden-sets` 접근 확인
2. 개발자 도구(F12) 콘솔에서 에러 없음 확인
3. Mock API 응답이 정상인지 네트워크 탭에서 확인 (또는 로컬 콘솔 로그)

### 7-2. S2 원칙 준수
- **원칙 ①**: DocumentType은 설정 파일(`constants/documentTypes.ts`)에서 동적으로 로드
  ```typescript
  // 하드코딩 금지
  // const documentTypes = ['Invoice', 'Contract'];
  
  // 올바른 방식
  import { getDocumentTypes } from '@/config/documentTypes';
  const documentTypes = getDocumentTypes();
  ```

- **원칙 ⑥**: Scope Profile 기반 접근 제어
  ```typescript
  // /admin 경로는 admin 권한 보호
  export async function middleware(request: NextRequest) {
    const scope = request.auth?.scope;
    if (request.nextUrl.pathname.startsWith('/admin') && scope !== 'admin') {
      return NextResponse.redirect(new URL('/forbidden', request.url));
    }
  }
  ```

- **원칙 ⑦**: 폐쇄망 환경
  ```typescript
  // 외부 CDN 최소화, mock API 로컬 동작 확인
  // public/index.html에서 외부 스크립트 로드 없음 확인
  // recharts 등은 npm 의존성이므로 OK
  ```

### 7-3. 목 데이터 마크 표시
모든 목 데이터 로드 시 콘솔 또는 UI에 마크 표시:
```typescript
// Mock 데이터 로드 시
console.log('[MOCK DATA] Loading golden sets from mock API');
console.warn('⚠️ This is mock data for Phase 6 skeleton UI. Will be replaced in Phase 7.');

// UI에도 배지로 표시
<Badge className="bg-yellow-100" title="Phase 7 API 연동 전 목 데이터">
  [Mock]
</Badge>
```

### 7-4. Phase 7·8 API 핸드오프 준비
Mock API 주석에 Phase 7 API 호출 방식 기록:
```typescript
// services/mockApi/goldenSetMockApi.ts
/**
 * Phase 7 핸드오프:
 * 이 함수는 아래의 실제 API로 교체 예정:
 * - Endpoint: GET /api/admin/golden-sets
 * - Response: { data: GoldenSet[], total: number }
 * - 변경 필요: URL, 에러 처리, 응답 형식 매핑
 */
export async function listGoldenSets() {
  // 현재: mock data 반환
  // Phase 7: fetch('/api/admin/golden-sets').then(...)
}
```

### 7-5. UI 리뷰 프로세스 (5회)
1. **1차 리뷰**: 레이아웃 및 그리드 정렬 (모바일/데스크탑)
2. **2차 리뷰**: 색상, 타이포그래피, 간격 (디자인 일관성)
3. **3차 리뷰**: 상호작용 (버튼, 필터, 모달 UX)
4. **4차 리뷰**: 접근성 (WCAG 준수, 스크린리더)
5. **5차 리뷰**: 반응형 (태블릿, 모바일 최적화)

각 리뷰 후 `task6-9_UI검토_Nc차.md` 파일에 기록.

### 7-6. 테스트 작성 가이드
- **Unit Test**: 컴포넌트 렌더링, Mock API 호출
- **실패 케이스**: 네트워크 오류, 빈 데이터
- **접근성 테스트**: axe-core 또는 jest-axe 사용
```typescript
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

test('should not have accessibility violations', async () => {
  const { container } = render(<GoldenSetTable />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

### 7-7. 커밋 메시지 규칙
```
[FG6.3] feat: 골든셋 관리 페이지 스켈레톤 구축

- 골든셋 목록 테이블 (정렬, 페이지네이션)
- 골든셋 상세 페이지 (Q&A 항목, 버전 이력)
- 생성 모달 (폼 검증)
- Mock API 구현 (3개 샘플 골든셋)
```

### 7-8. 빌드 및 배포 검증
```bash
# 빌드 성공 확인
npm run build

# 번들 크기 확인
npm run analyze (또는 bundle analyzer 도구)

# 테스트 실행
npm run test

# 린트 검사
npm run lint
```

---

## 8. 참고 문헌 및 링크

- **Phase 6 개발계획서**: `/docs/개발문서/S2/phase6/Phase 6 개발계획서.md`
- **Zustand 가이드**: https://github.com/pmndrs/zustand
- **TanStack Table**: https://tanstack.com/table/v8/docs/guide/introduction
- **recharts**: https://recharts.org/
- **WCAG 2.1**: https://www.w3.org/WAI/WCAG21/quickref/
- **Next.js 16 가이드**: https://nextjs.org/docs

---

**다음 단계**: Task 6-10 (추출 스키마 관리 + 추출 검토 큐)
