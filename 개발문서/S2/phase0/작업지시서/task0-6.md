# Task 0-6. [Critical] 프런트엔드 unwrapEnvelope<T>() 도입 및 리팩토링

## 1. 작업 목적

모든 API 응답을 **표준화된 envelope 구조**로 통일하고, 이를 처리하는 **재사용 가능한 유틸리티 함수** `unwrapEnvelope<T>()`를 도입하여:

1. API 응답 처리의 산발적(ad-hoc) 로직 제거
2. TypeScript 제네릭을 통한 타입 안전성 강화
3. 에러 처리 일관화 (toast, 401 처리, 재시도 정책)
4. AI-as-first-class-consumer 원칙 준수 (S2 원칙 ⑤)
5. 모든 API 호출부의 자동 검증 (ESLint 규칙)

---

## 2. 작업 범위

### 포함 범위

1. **API 응답 Envelope 구조 설계 및 문서화**
   - `{success: boolean, data?: T, error?: {code: string, message: string}}`
   - Phase 0 개발계획서 §4.2 기준

2. **unwrapEnvelope<T>() 유틸리티 함수 구현**
   - 파일: `/frontend/src/lib/api/envelope.ts` (신규)
   - 제네릭 타입 인수로 응답 데이터 타입 지정
   - 성공 시 data 반환, 실패 시 throw Error
   - 커스텀 에러 객체 (code, message 포함)

3. **모든 API 호출부 리팩토링 (12개 파일)**
   - `/frontend/src/lib/api/account.ts`
   - `/frontend/src/lib/api/admin.ts`
   - `/frontend/src/lib/api/auth.ts`
   - `/frontend/src/lib/api/client.ts`
   - `/frontend/src/lib/api/diff.ts`
   - `/frontend/src/lib/api/documents.ts`
   - `/frontend/src/lib/api/nodes.ts`
   - `/frontend/src/lib/api/rag.ts`
   - `/frontend/src/lib/api/search.ts`
   - `/frontend/src/lib/api/versions.ts`
   - `/frontend/src/lib/api/workflow.ts`
   - `/frontend/src/lib/api/index.ts`

4. **에러 처리 표준화**
   - 401 응답 → 자동 로그아웃 및 login 리다이렉트
   - 4xx 응답 → toast 알림 (사용자 친화적 메시지)
   - 5xx 응답 → toast 알림 + 로그 기록
   - Network error → 재시도 정책 or toast

5. **ESLint 규칙 추가 (선택)**
   - 모든 API 호출이 unwrapEnvelope()를 사용하도록 강제

### 제외 범위

- 기존 API 서버의 응답 포맷 변경 (프론트엔드 적응만 수행)
- API 응답 캐싱 레이어 (별도 작업)
- GraphQL 또는 다른 API 스타일 (REST API만)

---

## 3. 선행 조건

- 기존 API 클라이언트 파일 12개 모두 읽음 및 분석 완료
- 현재 API 응답 포맷 조사 완료 (envelope 구조 확인 또는 설정)
- Zustand/Context API 등 전역 상태 관리 방식 이해
- TypeScript 제네릭 패턴 숙지

---

## 4. 주요 작업 항목

### 4-1. API 응답 Envelope 구조 설계

**공식 정의:**

```typescript
// /frontend/src/lib/api/types.ts (신규 또는 기존 확장)

/**
 * 모든 API 응답의 표준 구조
 * @template T 응답 데이터 타입
 */
export interface ApiResponse<T = unknown> {
  /** 요청 성공 여부 */
  success: boolean;
  
  /** 성공 시 응답 데이터 */
  data?: T;
  
  /** 실패 시 에러 정보 */
  error?: {
    /** 에러 코드 (예: 'INVALID_REQUEST', 'UNAUTHORIZED', 'NOT_FOUND') */
    code: string;
    
    /** 사용자 친화적 에러 메시지 */
    message: string;
    
    /** 추가 정보 (선택) */
    details?: Record<string, unknown>;
  };
}

/**
 * 커스텀 에러 객체
 */
export class ApiError extends Error {
  constructor(
    public code: string,
    message: string,
    public details?: Record<string, unknown>
  ) {
    super(message);
    this.name = 'ApiError';
  }
}
```

### 4-2. unwrapEnvelope<T>() 구현

**파일:** `/frontend/src/lib/api/envelope.ts` (신규)

```typescript
import { ApiResponse, ApiError } from './types';

/**
 * API 응답 envelope를 언래핑하고 데이터 또는 에러를 반환/throw
 * @template T 응답 데이터 타입
 * @param response API 응답 객체
 * @returns 성공 시 data, 실패 시 throw
 * @throws ApiError
 */
export function unwrapEnvelope<T>(response: ApiResponse<T>): T {
  if (response.success && response.data !== undefined) {
    return response.data;
  }

  if (response.error) {
    throw new ApiError(
      response.error.code,
      response.error.message,
      response.error.details
    );
  }

  // 예상치 못한 응답 구조
  throw new ApiError(
    'UNKNOWN_ERROR',
    '알 수 없는 오류가 발생했습니다.'
  );
}

/**
 * unwrapEnvelope with auto-handling (toast, redirect 등)
 * 권장: 컴포넌트/훅에서만 사용
 */
export async function unwrapAndHandle<T>(
  responsePromise: Promise<ApiResponse<T>>,
  options?: {
    onError?: (error: ApiError) => void;
    showToast?: boolean;
  }
): Promise<T> {
  try {
    const response = await responsePromise;
    return unwrapEnvelope(response);
  } catch (error) {
    if (error instanceof ApiError) {
      // 401: 권한 없음 → 자동 로그아웃 (authStore에서 처리)
      if (error.code === 'UNAUTHORIZED') {
        // useAuthStore().logout(); // or trigger logout action
        window.location.href = '/login';
      }

      // 4xx/5xx: toast 알림
      if (options?.showToast !== false) {
        // showToast(error.message, 'error'); // toast 라이브러리 활용
        console.error(`[${error.code}] ${error.message}`);
      }

      if (options?.onError) {
        options.onError(error);
      }
    }

    throw error;
  }
}
```

### 4-3. API 클라이언트 파일 리팩토링 (예시)

**리팩토링 패턴:**

```typescript
// Before: /frontend/src/lib/api/documents.ts (수정 전)
export async function getDocument(id: string) {
  const response = await fetch(`/api/v1/documents/${id}`);
  const data = await response.json();
  
  // 산발적 에러 처리
  if (!response.ok) {
    if (response.status === 401) {
      // logout 로직 (inline, 중복)
    }
    throw new Error(data.error?.message || 'Unknown error');
  }

  return data.document; // 타입 추론 불가
}

// After: unwrapEnvelope 도입
import { ApiResponse, ApiError, unwrapEnvelope } from './types';

interface Document {
  id: string;
  title: string;
  // ...
}

interface GetDocumentResponse {
  document: Document;
}

export async function getDocument(id: string): Promise<Document> {
  const response = await fetch(`/api/v1/documents/${id}`);
  const envelope = (await response.json()) as ApiResponse<GetDocumentResponse>;
  
  const data = unwrapEnvelope(envelope);
  return data.document;
}
```

**12개 파일별 리팩토링 체크리스트:**

1. `/frontend/src/lib/api/account.ts`
   - getProfile(), updateProfile(), updatePassword() 등
   
2. `/frontend/src/lib/api/admin.ts`
   - listUsers(), createUser(), updateRole() 등
   
3. `/frontend/src/lib/api/auth.ts`
   - login(), logout(), refresh() 등
   
4. `/frontend/src/lib/api/client.ts`
   - listClients(), createClient(), revokeClient() 등
   
5. `/frontend/src/lib/api/diff.ts`
   - getDiff(), compareDocs() 등
   
6. `/frontend/src/lib/api/documents.ts`
   - getDocument(), listDocuments(), createDocument() 등
   
7. `/frontend/src/lib/api/nodes.ts`
   - getNode(), createNode(), updateNode() 등
   
8. `/frontend/src/lib/api/rag.ts`
   - search(), retrieve(), augment() 등
   
9. `/frontend/src/lib/api/search.ts`
   - search(), autocomplete() 등
   
10. `/frontend/src/lib/api/versions.ts`
    - listVersions(), getVersion(), compareVersions() 등
    
11. `/frontend/src/lib/api/workflow.ts`
    - listWorkflows(), executeWorkflow(), approve() 등
    
12. `/frontend/src/lib/api/index.ts`
    - 모든 API 함수를 export하는 barrel file

### 4-4. 에러 처리 통일

**공통 에러 처리 미들웨어:**

```typescript
// /frontend/src/lib/api/middleware.ts (신규)

import { ApiError } from './types';
import { useAuthStore } from '@/stores/authStore';

/**
 * API 에러 처리 미들웨어
 * 모든 API 호출의 catch 블록에서 사용
 */
export async function handleApiError(
  error: unknown,
  options?: { silent?: boolean }
) {
  if (!(error instanceof ApiError)) {
    console.error('Unknown error:', error);
    return;
  }

  // 401: 권한 없음
  if (error.code === 'UNAUTHORIZED') {
    const authStore = useAuthStore();
    await authStore.logout();
    window.location.href = '/login';
    return;
  }

  // 403: 금지됨
  if (error.code === 'FORBIDDEN') {
    // toast('접근 권한이 없습니다.', 'error');
    return;
  }

  // 422: 검증 실패
  if (error.code === 'VALIDATION_ERROR') {
    // form validation 에러 (자세한 처리는 컴포넌트에서)
    return;
  }

  // 기타 4xx/5xx
  if (!options?.silent) {
    // toast(error.message, 'error');
  }

  console.error(`[${error.code}] ${error.message}`, error.details);
}
```

### 4-5. 컴포넌트 사용 예시

```typescript
// /frontend/src/components/DocumentView.tsx

'use client';
import { useEffect, useState } from 'react';
import { getDocument } from '@/lib/api/documents';
import { ApiError } from '@/lib/api/types';

interface Document {
  id: string;
  title: string;
  content: string;
}

export function DocumentView({ docId }: { docId: string }) {
  const [doc, setDoc] = useState<Document | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    (async () => {
      try {
        setLoading(true);
        const data = await getDocument(docId);
        setDoc(data);
      } catch (err) {
        if (err instanceof ApiError) {
          setError(err.message);
        } else {
          setError('알 수 없는 오류가 발생했습니다.');
        }
      } finally {
        setLoading(false);
      }
    })();
  }, [docId]);

  if (loading) return <div>로딩 중...</div>;
  if (error) return <div className="text-red-600">{error}</div>;
  if (!doc) return <div>문서를 찾을 수 없습니다.</div>;

  return (
    <div>
      <h1>{doc.title}</h1>
      <p>{doc.content}</p>
    </div>
  );
}
```

### 4-6. ESLint 규칙 (선택)

**목표:** 모든 API 호출이 unwrapEnvelope()를 거치도록 강제

```javascript
// .eslintrc.js (추가 규칙, custom plugin 또는 manual check)

module.exports = {
  rules: {
    // "no-direct-fetch-usage": "warn", // custom rule
    // "require-envelope-unwrap": "warn", // custom rule
  },
};
```

**수동 검증 (CI에서):**
```bash
# 스크립트: scripts/check-api-envelope.js
# 정규식으로 직접 fetch/axios 호출 검사
# unwrapEnvelope() 없이 사용되는 API 응답 감지
```

---

## 5. 산출물

1. **`/frontend/src/lib/api/types.ts`** (신규 또는 확장)
   - ApiResponse<T> 인터페이스
   - ApiError 클래스

2. **`/frontend/src/lib/api/envelope.ts`** (신규)
   - unwrapEnvelope<T>() 함수
   - unwrapAndHandle<T>() 함수

3. **`/frontend/src/lib/api/middleware.ts`** (신규)
   - handleApiError() 통일된 에러 처리

4. **12개 API 클라이언트 파일 (수정본)**
   - 모든 함수에 unwrapEnvelope() 적용
   - 타입 안전성 강화 (response 구조 명시)

5. **리팩토링 체크리스트**
   - 각 파일별 리팩토링 완료 여부
   - 테스트 결과

---

## 6. 완료 기준

1. unwrapEnvelope<T>() 함수 구현 및 테스트 통과
2. 12개 API 클라이언트 파일 모두 unwrapEnvelope() 적용 완료
3. 401 응답 시 자동 로그아웃 및 login 리다이렉트 동작
4. 모든 에러 코드에 대해 적절한 toast/로그 처리 확인
5. TypeScript 타입 체크 통과 (strict mode)
6. 기존 기능 회귀 테스트 통과 (manual or e2e)
7. ESLint 검증 통과
8. 개발자 문서 작성 (API 응답 구조, 사용법)

---

## 7. 작업 지침

### 지침 7-1. 점진적 리팩토링

모든 파일을 동시에 수정하지 말 것. 순서대로 진행:

1. types.ts, envelope.ts, middleware.ts 먼저 작성 및 테스트
2. auth.ts, account.ts (핵심 인증 관련) 리팩토링
3. documents.ts, search.ts, nodes.ts (주요 기능) 리팩토링
4. 나머지 파일들 순차 리팩토링

### 지침 7-2. 타입 안전성

각 API 함수마다 명시적 response 타입 정의:

```typescript
interface ListDocumentsResponse {
  documents: Document[];
  total: number;
  page: number;
}

export async function listDocuments(
  params: ListParams
): Promise<ListDocumentsResponse> {
  const response = await fetch(`/api/v1/documents?...`);
  const envelope = (await response.json()) as ApiResponse<ListDocumentsResponse>;
  return unwrapEnvelope(envelope);
}
```

### 지침 7-3. 기존 코드와의 호환성

만약 기존 코드가 직접 response를 다루고 있다면:

```typescript
// Deprecated: 이전 방식
// const data = response.data || response; // 호환성 레이어

// Recommended: unwrapEnvelope 사용
const data = unwrapEnvelope(response);
```

### 지침 7-4. 테스트 커버리지

unwrapEnvelope() 단위 테스트:

```typescript
// /frontend/src/lib/api/__tests__/envelope.test.ts

import { unwrapEnvelope, ApiError } from '../types';

test('success case', () => {
  const response = { success: true, data: { id: '1' } };
  expect(unwrapEnvelope(response)).toEqual({ id: '1' });
});

test('error case', () => {
  const response = {
    success: false,
    error: { code: 'NOT_FOUND', message: 'Not found' },
  };
  expect(() => unwrapEnvelope(response)).toThrow(ApiError);
});
```

### 지침 7-5. 배포 전 검증

리팩토링 완료 후:

```bash
# TypeScript 타입 검사
npm run type-check

# ESLint
npm run lint

# 테스트
npm run test

# 빌드
npm run build

# e2e 테스트 (선택)
npm run test:e2e
```
