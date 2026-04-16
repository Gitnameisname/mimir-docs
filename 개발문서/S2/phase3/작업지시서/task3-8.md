# Task 3-8. Citation 표시 + 원문 링크 + 신뢰도 배지

## 1. 작업 목적

각 Assistant 응답(Turn)의 하단에 **Citation 정보**를 명확하게 표시하고, "원문 보기" 링크를 통해 Document의 해당 노드로 직접 이동할 수 있게 한다. 또한 Citation의 신뢰도를 content_hash 검증 결과로 시각화하여 사용자가 근거의 정확성을 판단할 수 있도록 지원한다.

## 2. 작업 범위

### 포함 범위

1. **CitationBlock 컴포넌트**
   - Assistant 메시지의 하단에 "인용 출처" 섹션 추가
   - 각 Citation을 개별 아이템으로 표시
   - Collapsible 또는 항상 표시(선택에 따라)
   - 없으면 빈 상태 처리

2. **Citation 5-tuple 구조 표시**
   - document_id: 문서 고유 ID
   - version_id: 문서 버전 ID
   - node_id: 문서의 특정 노드(섹션/문단) ID
   - span_offset: 하이라이트 시작 위치 (선택)
   - content_hash: 콘텐츠 무결성 검증 해시
   - content: (선택) 인용된 실제 텍스트 미리보기

3. **CitationItem 렌더링**
   - 문서명 (Document.metadata.display_name 또는 title)
   - 버전 정보 (버전 번호 또는 "최신 버전")
   - 노드 ID 또는 노드 타이틀
   - "원문 보기" 링크
   - (선택) 인용된 텍스트 미리보기 (최대 100자)

4. **원문 보기 링크**
   - 링크 형식: `/documents/{document_id}/nodes/{node_id}?version={version_id}&highlight=span_offset`
   - 클릭 시 Document 상세 뷰로 이동 (Task 3-9와 협력)
   - span_offset이 있으면 해당 위치를 하이라이트 표시

5. **CitationVerificationBadge 컴포넌트**
   - content_hash 검증 결과 시각화
   - 상태:
     - 검증됨 (체크마크): content_hash가 유효하고 실시간 검증 성공
     - 검증 불가 (경고 아이콘): 원본 노드 삭제, 버전 변경 등으로 hash 불일치
     - 검증 대기 중: 백그라운드 검증 진행 중 (선택)
   - 마우스 오버 시 상태 설명 Tooltip 표시

6. **Citation 없는 경우 처리**
   - Assistant 응답에 citations 배열이 없거나 비어있으면
   - "근거 없음" 경고 배지 표시 (주황색)
   - 사용자에게 응답의 신뢰성 한계 알림

7. **Citation 캐싱 및 성능**
   - 같은 citation은 중복 렌더링하지 않음
   - content_hash 검증 결과 캐싱 (로컬 스토리지 또는 인메모리)
   - TTL: 1시간 또는 유저가 문서 업데이트 시 캐시 무효화

8. **Citation 상호작용**
   - Citation 아이템 클릭 시 Document 상세 뷰로 이동 (라우팅)
   - (선택) Citation 정보 복사 (copy to clipboard)
   - (선택) Citation 인용 형식 제공 (BibTeX, APA 등)

### 제외 범위

- Document 상세 뷰 페이지 (Task 3-9)
- Document 조회 권한 필터링 (Task 3-1, FG3.2에서 처리)
- span_offset 하이라이트의 정확한 구현 (Document 뷰에서 처리)
- Citation 기반 재검색 기능 (Task 3-9+)

## 3. 선행 조건

- Task 3-7 완료: ChatWindow, MessageBubble 컴포넌트
- FG3.1, FG3.2 완료: Citation 5-tuple 구조가 API 응답에 포함됨
- Phase 2 완료: Citation 계약 정의
- Document 도메인 모델 분석: document_id, version_id, node_id 구조 이해

## 4. 주요 작업 항목

### 4-1. CitationBlock 컴포넌트

**파일:** `/frontend/src/components/chat/CitationBlock.tsx`

```typescript
"use client";

import React, { useState } from "react";
import CitationItem from "./CitationItem";
import { Citation } from "@/types/conversation";

interface CitationBlockProps {
  citations?: Citation[];
  turnId: string;
}

export default function CitationBlock({ citations, turnId }: CitationBlockProps) {
  const [isExpanded, setIsExpanded] = useState(false);

  if (!citations || citations.length === 0) {
    return (
      <div className="mt-3 pt-3 border-t border-gray-300">
        <div className="inline-flex items-center gap-2 px-2 py-1 bg-orange-100 text-orange-700 rounded text-xs">
          <span className="font-semibold">⚠</span>
          <span>근거 없음</span>
        </div>
      </div>
    );
  }

  return (
    <div className="mt-3 pt-3 border-t border-gray-300">
      <button
        onClick={() => setIsExpanded(!isExpanded)}
        className="flex items-center gap-2 text-sm font-semibold text-gray-700 hover:text-gray-900 cursor-pointer"
      >
        <span>{isExpanded ? "▼" : "▶"}</span>
        <span>인용 출처 ({citations.length})</span>
      </button>

      {isExpanded && (
        <div className="mt-2 space-y-2">
          {citations.map((citation, index) => (
            <CitationItem key={`${turnId}-citation-${index}`} citation={citation} />
          ))}
        </div>
      )}
    </div>
  );
}
```

### 4-2. CitationItem 컴포넌트

**파일:** `/frontend/src/components/chat/CitationItem.tsx`

```typescript
"use client";

import React, { useState, useEffect } from "react";
import { useRouter } from "next/navigation";
import CitationVerificationBadge from "./CitationVerificationBadge";
import { Citation } from "@/types/conversation";
import { verifyCitationHash } from "@/services/citationService";

interface CitationItemProps {
  citation: Citation;
}

export default function CitationItem({ citation }: CitationItemProps) {
  const router = useRouter();
  const [verificationStatus, setVerificationStatus] = useState<
    "verified" | "failed" | "pending"
  >("pending");
  const [showPreview, setShowPreview] = useState(false);

  // content_hash 검증 (백그라운드)
  useEffect(() => {
    verifyHash();
  }, [citation.document_id, citation.version_id, citation.node_id, citation.content_hash]);

  const verifyHash = async () => {
    try {
      const isValid = await verifyCitationHash(
        citation.document_id,
        citation.version_id,
        citation.node_id,
        citation.content_hash
      );
      setVerificationStatus(isValid ? "verified" : "failed");
    } catch (error) {
      console.error("Failed to verify citation:", error);
      setVerificationStatus("failed");
    }
  };

  const handleNavigateToDocument = () => {
    const url = new URL("/documents/" + citation.document_id, window.location.origin);
    url.searchParams.append("version", citation.version_id);
    url.searchParams.append("node", citation.node_id);
    if (citation.span_offset !== undefined) {
      url.searchParams.append("highlight", String(citation.span_offset));
    }
    router.push(url.toString());
  };

  const handleCopyBibTex = () => {
    const bibtex = `@document{${citation.document_id},
  title={Document},
  year={${new Date().getFullYear()}},
  url={/documents/${citation.document_id}/nodes/${citation.node_id}}
}`;
    navigator.clipboard.writeText(bibtex);
  };

  return (
    <div className="p-2 bg-gray-50 border border-gray-200 rounded text-xs">
      {/* Citation Header */}
      <div className="flex items-start justify-between gap-2 mb-1">
        <div className="flex-1">
          <p className="font-semibold text-gray-900">
            {citation.content ? "문서 인용" : "참고 출처"}
          </p>
          <p className="text-gray-600 text-xs">
            {citation.version_id && `v${citation.version_id.slice(0, 8)}`}
            {citation.node_id && ` • ${citation.node_id.slice(0, 12)}`}
          </p>
        </div>

        <CitationVerificationBadge status={verificationStatus} />
      </div>

      {/* Preview */}
      {citation.content && (
        <div className="mb-2">
          <button
            onClick={() => setShowPreview(!showPreview)}
            className="text-blue-600 hover:text-blue-800 underline text-xs"
          >
            {showPreview ? "숨기기" : "미리보기"}
          </button>
          {showPreview && (
            <p className="mt-1 p-2 bg-white border border-gray-200 rounded text-gray-700 italic">
              "{citation.content.slice(0, 100)}
              {citation.content.length > 100 ? "..." : ""}"
            </p>
          )}
        </div>
      )}

      {/* Actions */}
      <div className="flex gap-2">
        <button
          onClick={handleNavigateToDocument}
          className="px-2 py-1 bg-blue-500 text-white rounded hover:bg-blue-600 text-xs"
        >
          원문 보기
        </button>

        <button
          onClick={handleCopyBibTex}
          className="px-2 py-1 bg-gray-300 text-gray-800 rounded hover:bg-gray-400 text-xs"
        >
          인용 형식
        </button>
      </div>
    </div>
  );
}
```

### 4-3. CitationVerificationBadge 컴포넌트

**파일:** `/frontend/src/components/chat/CitationVerificationBadge.tsx`

```typescript
"use client";

import React, { useState } from "react";

interface CitationVerificationBadgeProps {
  status: "verified" | "failed" | "pending";
}

export default function CitationVerificationBadge({
  status,
}: CitationVerificationBadgeProps) {
  const [showTooltip, setShowTooltip] = useState(false);

  const getStatusDisplay = () => {
    switch (status) {
      case "verified":
        return {
          icon: "✓",
          color: "bg-green-100 text-green-700",
          label: "검증됨",
          tooltip: "인용 출처의 콘텐츠가 검증되었습니다.",
        };
      case "failed":
        return {
          icon: "⚠",
          color: "bg-orange-100 text-orange-700",
          label: "검증 실패",
          tooltip: "원본 문서가 변경되었을 수 있습니다.",
        };
      case "pending":
        return {
          icon: "○",
          color: "bg-gray-100 text-gray-600",
          label: "검증 중...",
          tooltip: "검증이 진행 중입니다.",
        };
      default:
        return {
          icon: "?",
          color: "bg-gray-100 text-gray-600",
          label: "상태 불명",
          tooltip: "상태를 확인할 수 없습니다.",
        };
    }
  };

  const { icon, color, label, tooltip } = getStatusDisplay();

  return (
    <div className="relative">
      <div
        className={`inline-flex items-center gap-1 px-2 py-1 rounded text-xs font-semibold ${color} cursor-help`}
        onMouseEnter={() => setShowTooltip(true)}
        onMouseLeave={() => setShowTooltip(false)}
      >
        <span>{icon}</span>
        <span>{label}</span>
      </div>

      {showTooltip && (
        <div className="absolute right-0 top-full mt-1 w-40 p-2 bg-gray-900 text-white text-xs rounded shadow-lg z-10 pointer-events-none">
          {tooltip}
        </div>
      )}
    </div>
  );
}
```

### 4-4. MessageBubble 업데이트 (Task 3-7과 통합)

**파일:** `/frontend/src/components/chat/MessageBubble.tsx` (수정)

```typescript
"use client";

import React from "react";
import ReactMarkdown from "react-markdown";
import CitationBlock from "./CitationBlock";
import { Message } from "@/types/conversation";

interface MessageBubbleProps {
  message: Message;
}

export default function MessageBubble({ message }: MessageBubbleProps) {
  const isUser = message.role === "user";

  return (
    <div className={`flex ${isUser ? "justify-end" : "justify-start"} mb-4`}>
      <div
        className={`max-w-md lg:max-w-2xl px-4 py-3 rounded-lg ${
          isUser
            ? "bg-blue-500 text-white rounded-br-none"
            : "bg-gray-200 text-gray-900 rounded-bl-none"
        }`}
      >
        {isUser ? (
          <p className="text-sm">{message.content}</p>
        ) : (
          <>
            <div className="prose prose-sm max-w-none text-gray-900">
              <ReactMarkdown
                components={{
                  p: ({ node, ...props }) => <p className="mb-2" {...props} />,
                  code: ({ node, inline, ...props }) =>
                    inline ? (
                      <code className="bg-gray-300 px-1 py-0.5 rounded text-xs" {...props} />
                    ) : (
                      <code className="block bg-gray-700 text-white p-2 rounded mb-2 text-xs overflow-x-auto" {...props} />
                    ),
                  ul: ({ node, ...props }) => (
                    <ul className="list-disc list-inside mb-2" {...props} />
                  ),
                  ol: ({ node, ...props }) => (
                    <ol className="list-decimal list-inside mb-2" {...props} />
                  ),
                  a: ({ node, ...props }) => (
                    <a className="text-blue-600 underline hover:text-blue-800" {...props} />
                  ),
                }}
              >
                {message.content}
              </ReactMarkdown>
            </div>

            {/* Citation Block */}
            {!isUser && (
              <CitationBlock citations={message.citations} turnId={message.turn_id} />
            )}
          </>
        )}

        <p className="text-xs opacity-75 mt-2">
          {new Date(message.created_at).toLocaleTimeString()}
        </p>
      </div>
    </div>
  );
}
```

### 4-5. Citation 검증 서비스

**파일:** `/frontend/src/services/citationService.ts`

```typescript
import { unwrapEnvelope } from "@/utils/apiUtils";

interface VerificationResponse {
  is_valid: boolean;
  content_hash: string;
  current_content_hash: string;
  reason?: string;
}

const verificationCache = new Map<string, { valid: boolean; expiry: number }>();

const getCacheKey = (docId: string, verId: string, nodeId: string, hash: string) =>
  `${docId}:${verId}:${nodeId}:${hash}`;

const isCacheValid = (expiry: number) => expiry > Date.now();

export async function verifyCitationHash(
  documentId: string,
  versionId: string,
  nodeId: string,
  contentHash: string
): Promise<boolean> {
  const cacheKey = getCacheKey(documentId, versionId, nodeId, contentHash);

  // 캐시 확인
  if (verificationCache.has(cacheKey)) {
    const cached = verificationCache.get(cacheKey)!;
    if (isCacheValid(cached.expiry)) {
      return cached.valid;
    }
    // 만료된 캐시는 제거
    verificationCache.delete(cacheKey);
  }

  try {
    const response = await fetch(
      `/api/v1/citations/verify`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          document_id: documentId,
          version_id: versionId,
          node_id: nodeId,
          content_hash: contentHash,
        }),
      }
    );

    if (!response.ok) {
      throw new Error(`Verification failed: ${response.statusText}`);
    }

    const envelope = await response.json();
    const data: VerificationResponse = unwrapEnvelope(envelope);

    // 캐시 저장 (1시간 TTL)
    const expiry = Date.now() + 60 * 60 * 1000;
    verificationCache.set(cacheKey, { valid: data.is_valid, expiry });

    return data.is_valid;
  } catch (error) {
    console.error("Citation verification error:", error);
    // 에러 시 검증 실패로 처리
    return false;
  }
}

export function clearCitationCache(): void {
  verificationCache.clear();
}

export function invalidateCitationCache(documentId: string, versionId: string): void {
  // 특정 문서 버전의 캐시 무효화
  for (const [key] of verificationCache) {
    if (key.startsWith(`${documentId}:${versionId}`)) {
      verificationCache.delete(key);
    }
  }
}
```

### 4-6. 단위 테스트

**파일:** `/frontend/src/components/chat/__tests__/CitationBlock.test.tsx`

```typescript
import React from "react";
import { render, screen, fireEvent } from "@testing-library/react";
import CitationBlock from "../CitationBlock";
import { Citation } from "@/types/conversation";

describe("CitationBlock", () => {
  test("renders warning badge when no citations", () => {
    render(<CitationBlock citations={[]} turnId="turn-1" />);
    expect(screen.getByText("근거 없음")).toBeInTheDocument();
  });

  test("renders citation count when citations exist", () => {
    const citations: Citation[] = [
      {
        document_id: "doc-1",
        version_id: "v-1",
        node_id: "node-1",
        content_hash: "hash1",
      },
      {
        document_id: "doc-2",
        version_id: "v-2",
        node_id: "node-2",
        content_hash: "hash2",
      },
    ];

    render(<CitationBlock citations={citations} turnId="turn-1" />);
    expect(screen.getByText("인용 출처 (2)")).toBeInTheDocument();
  });

  test("expands and collapses citation items", () => {
    const citations: Citation[] = [
      {
        document_id: "doc-1",
        version_id: "v-1",
        node_id: "node-1",
        content_hash: "hash1",
      },
    ];

    render(<CitationBlock citations={citations} turnId="turn-1" />);

    const button = screen.getByText("인용 출처 (1)");
    expect(button).toBeInTheDocument();

    // 초기 상태: 폐쇄
    expect(screen.queryByText("원문 보기")).not.toBeInTheDocument();

    // 클릭하여 확장
    fireEvent.click(button);
    expect(screen.getByText("원문 보기")).toBeInTheDocument();

    // 다시 클릭하여 폐쇄
    fireEvent.click(button);
    expect(screen.queryByText("원문 보기")).not.toBeInTheDocument();
  });
});
```

**파일:** `/frontend/src/components/chat/__tests__/CitationVerificationBadge.test.tsx`

```typescript
import React from "react";
import { render, screen, fireEvent } from "@testing-library/react";
import CitationVerificationBadge from "../CitationVerificationBadge";

describe("CitationVerificationBadge", () => {
  test("renders verified badge", () => {
    render(<CitationVerificationBadge status="verified" />);
    expect(screen.getByText("✓")).toBeInTheDocument();
    expect(screen.getByText("검증됨")).toBeInTheDocument();
  });

  test("renders failed badge", () => {
    render(<CitationVerificationBadge status="failed" />);
    expect(screen.getByText("⚠")).toBeInTheDocument();
    expect(screen.getByText("검증 실패")).toBeInTheDocument();
  });

  test("renders pending badge", () => {
    render(<CitationVerificationBadge status="pending" />);
    expect(screen.getByText("검증 중...")).toBeInTheDocument();
  });

  test("shows tooltip on hover", () => {
    const { container } = render(
      <CitationVerificationBadge status="verified" />
    );

    const badge = container.querySelector(".cursor-help");
    expect(badge).toBeInTheDocument();

    // 마우스 오버
    fireEvent.mouseEnter(badge!);
    expect(screen.getByText("인용 출처의 콘텐츠가 검증되었습니다.")).toBeInTheDocument();

    // 마우스 아웃
    fireEvent.mouseLeave(badge!);
    expect(screen.queryByText("인용 출처의 콘텐츠가 검증되었습니다.")).not.toBeInTheDocument();
  });
});
```

### 4-7. 타입 업데이트

**파일:** `/frontend/src/types/conversation.ts` (수정)

```typescript
export interface Conversation {
  id: string;
  owner_id: string;
  organization_id: string;
  title: string;
  status: "active" | "archived" | "expired" | "deleted";
  metadata?: Record<string, any>;
  created_at: string;
  updated_at: string;
}

export interface Message {
  id: string;
  turn_id: string;
  role: "user" | "assistant" | "system";
  content: string;
  created_at: string;
  citations?: Citation[];
  metadata?: Record<string, any>;
}

export interface Citation {
  document_id: string;
  version_id: string;
  node_id: string;
  span_offset?: number;
  content_hash: string;
  content?: string;
}

export interface Turn {
  id: string;
  conversation_id: string;
  turn_number: number;
  user_message: string;
  assistant_response: string;
  citations?: Citation[];
  retrieval_metadata?: Record<string, any>;
  created_at: string;
}
```

## 5. 산출물

1. **CitationBlock 컴포넌트** (`/frontend/src/components/chat/CitationBlock.tsx`)
   - Collapsible citation 섹션

2. **CitationItem 컴포넌트** (`/frontend/src/components/chat/CitationItem.tsx`)
   - 개별 citation 렌더링
   - 원문 링크, 미리보기, 인용 형식

3. **CitationVerificationBadge 컴포넌트** (`/frontend/src/components/chat/CitationVerificationBadge.tsx`)
   - 검증 상태 시각화
   - Tooltip 지원

4. **Citation 검증 서비스** (`/frontend/src/services/citationService.ts`)
   - Hash 검증 API 호출
   - 캐싱 및 TTL 관리

5. **MessageBubble 업데이트**
   - CitationBlock 통합

6. **단위 테스트**
   - `CitationBlock.test.tsx`
   - `CitationVerificationBadge.test.tsx`

7. **타입 정의 업데이트**

## 6. 완료 기준

1. CitationBlock이 Assistant 메시지 하단에 정상 렌더링되는가?
2. Collapsible 동작이 정상 작동하는가?
3. CitationItem이 각 citation을 개별 아이템으로 표시하는가?
4. "원문 보기" 링크가 올바른 Document URL을 생성하는가?
5. span_offset 파라미터가 URL에 포함되는가?
6. CitationVerificationBadge가 검증 상태를 올바르게 표시하는가?
7. Tooltip이 마우스 오버 시 정상 표시되는가?
8. Citation이 없는 경우 "근거 없음" 배지가 표시되는가?
9. 미리보기 기능이 toggle 동작하는가?
10. Citation 검증 캐시가 1시간 TTL로 동작하는가?
11. 인용 형식 복사 기능이 작동하는가?
12. 모든 단위 테스트가 통과하는가?
13. 타입스크립트 컴파일이 정상 작동하는가?

## 7. 작업 지침

### 지침 7-1. Citation 5-tuple 검증

Citation의 content_hash는 문서의 해당 노드 콘텐츠의 SHA256 해시이다. 백그라운드에서 현재 노드의 콘텐츠로 재계산한 해시와 비교하여 검증한다. 일치하면 "검증됨", 불일치하면 "검증 실패"로 표시한다.

```python
# 백엔드 검증 로직 (참고)
import hashlib

def verify_citation(document_id, version_id, node_id, content_hash):
    node = get_node(document_id, version_id, node_id)
    current_hash = hashlib.sha256(node.content.encode()).hexdigest()
    return current_hash == content_hash
```

### 지침 7-2. 원문 보기 링크 형식

링크는 다음 형식을 따른다:

```
/documents/{document_id}/nodes/{node_id}?version={version_id}&highlight={span_offset}
```

- `highlight` 파라미터는 span_offset이 있을 때만 포함
- Document 상세 뷰 페이지에서 해당 매개변수를 파싱하여 노드를 하이라이트 처리

### 지침 7-3. 캐시 무효화 전략

Citation 검증 캐시는 다음 경우에 무효화된다:
1. TTL 만료 (1시간)
2. 사용자가 Document를 업데이트할 때 (invalidateCitationCache 호출)
3. 문서 버전이 새로 생성될 때

### 지침 7-4. XSS 방지

Citation.content 필드는 신뢰되는 서버에서 전송된 데이터이지만, 렌더링 시에는 여전히 XSS 방지를 고려한다. React의 자동 이스케이핑을 활용하고, 마크다운 렌더링이 필요할 경우 `disallowedElements`를 통해 위험한 요소를 필터링한다.

### 지침 7-5. 근거 없음 경고

Citation이 없는 답변은 LLM이 hallucination을 생성했을 가능성이 높다. 사용자에게 명확히 경고하여 응답의 신뢰도를 낮게 인식하도록 유도한다.

### 지침 7-6. 인용 형식 지원

BibTeX 형식 외에 추가적으로 APA, MLA 등을 지원할 수 있다. 현재는 BibTeX만 구현하고, 확장성을 위해 serviceFormat을 매개변수화한다.

```typescript
const formats = {
  bibtex: generateBibTeX,
  apa: generateAPA,
  mla: generateMLA,
};
```

### 지침 7-7. 성능 최적화

Citation 렌더링 시 여러 citation을 동시에 검증하면 API 호출이 과다할 수 있다. 다음 최적화를 고려한다:
1. 백그라운드 검증으로 UI 블로킹 방지
2. 요청 배치(batch) 처리 (여러 citation을 한 번에 검증)
3. 검증 결과 캐싱

### 지침 7-8. 접근성 (a11y)

이 Task에서는 기본적 접근성 고려:
- Badge 색상만으로 상태 표현 금지 (아이콘 + 텍스트)
- Tooltip에 role="tooltip" 속성 추가 (Task 3-10에서 강화)
- 링크에 명확한 텍스트 ("원문 보기", "인용 형식")
