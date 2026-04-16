# Task 3-7. 채팅 UI 기본 구조 (ChatWindow + MessageBubble + InputArea)

## 1. 작업 목적

사용자가 Mimir와 멀티턴 대화를 나누기 위한 **기본 채팅 인터페이스**를 구현한다. 3-컬럼 레이아웃(좌측 대화 목록, 중앙 채팅창, 우측 참고 자료)으로 구성되며, 실시간 스트리밍 응답(SSE)을 지원하는 메시지 렌더링과 입력 필드를 완성한다.

## 2. 작업 범위

### 포함 범위

1. **3-컬럼 레이아웃 구조 (데스크톱 기준)**
   - 좌측: ConversationList (placeholder 또는 기본 구현)
   - 중앙: ChatWindow (메인 채팅 영역)
   - 우측: ReferencePanel (선택사항, Task 3-8과 협력)
   - 레이아웃 관리: Tailwind CSS grid 또는 flexbox

2. **ChatWindow 컴포넌트 (중앙 영역)**
   - ChatHeader: 현재 대화 제목, 옵션 메뉴(더보기 버튼), 로딩 상태 표시
   - MessageHistory: 턴별 메시지 시퀀스 (스크롤 컨테이너)
   - InputArea: 질문 입력 필드, 전송 버튼, 로딩 인디케이터

3. **MessageBubble 컴포넌트**
   - User 메시지: 우측 정렬, 파란색 계열 배경
   - Assistant 메시지: 좌측 정렬, 회색 계열 배경
   - 마크다운 렌더링: `react-markdown` 또는 유사 라이브러리 사용
   - 콘텐츠: 텍스트, 코드 블록, 리스트 등 마크다운 요소 지원
   - 스트리밍 상태: 부분 응답 표시, 타이핑 인디케이터(선택)

4. **SSE(Server-Sent Events) 스트리밍 응답 처리**
   - API 응답을 청크 단위로 수신하여 메시지 실시간 업데이트
   - 스트리밍 중 로딩 상태 UI
   - 스트리밍 완료 후 최종 응답 표시
   - 에러 처리: 스트리밍 중단, 재시도 옵션 (선택)

5. **InputArea 컴포넌트**
   - TextInput: multiline 텍스트 입력 (Shift+Enter로 줄바꿈)
   - SendButton: 전송 버튼 (로딩 중 비활성화)
   - 키보드 이벤트: Enter 키로 전송, Shift+Enter로 줄바꿈
   - 문자 수 제한: 없음 또는 합리적 제한(예: 5000자)
   - 입력 중 자동 저장(선택): draft 상태 저장

6. **Zustand 상태 관리**
   - conversationStore
     - currentConversation: 현재 대화 객체 (id, title, created_at, metadata)
     - messages: 턴별 메시지 배열 (turn_id, role, content, created_at)
     - loading: 대화 로딩 중 여부
     - streaming: SSE 스트리밍 중 여부
     - streamingContent: 현재 스트리밍 중인 메시지 부분 텍스트
   - 액션: setCurrentConversation, addMessage, updateStreamingContent, clearConversation

7. **API 호출 계층 (conversationApi.ts, ragApi.ts)**
   - unwrapEnvelope 패턴 사용: API 응답 구조 표준화
   - conversationApi
     - getConversation(conversationId): 대화 조회
     - createConversation(): 새 대화 생성
     - listTurns(conversationId): 대화의 모든 턴 조회
   - ragApi
     - answerWithStreaming(query, conversationId): SSE 스트리밍 응답
   - 에러 처리: 네트워크 에러, 타임아웃, 서버 에러 핸들링

8. **새 대화 시작 기능**
   - "새로운 대화" 버튼 클릭 시 새 Conversation 생성 (Task 3-1 API 호출)
   - 첫 질문 입력 후 자동으로 제목 생성 (첫 질문의 일부, 예: 최대 50자)
   - 빈 상태(empty state): "질문을 입력하여 시작하세요" 프롬프트 및 예제 질문 제시(선택)

9. **키보드 단축키 및 UX**
   - Enter: 메시지 전송
   - Shift+Enter: 줄바꿈
   - Escape: (선택) 스트리밍 중단
   - Tab: InputArea에서 SendButton으로 포커스 이동

### 제외 범위

- Citation 표시 (Task 3-8)
- 대화 이력 목록 상세 구현 (Task 3-9, 이 Task에서는 placeholder)
- 반응형 디자인 breakpoint (Task 3-10)
- 접근성(a11y) 상세 구현 (Task 3-10)

## 3. 선행 조건

- FG3.1 완료: Conversation/Turn API 엔드포인트 정상 작동
- FG3.2 완료: 세션 기반 RAG API (`/rag/answer` with conversation_id) 정상 작동
- Phase 1 완료: 모델 추상화, 토큰 카운팅
- Phase 2 완료: Citation 5-tuple 계약
- 프론트엔드 환경 설정:
  - Next.js 16.2.2, React 19, TypeScript
  - Zustand 상태 관리 (이미 설치)
  - Tailwind CSS (이미 설정)
  - `react-markdown` 또는 유사 라이브러리 설치 필요

## 4. 주요 작업 항목

### 4-1. 3-컬럼 레이아웃 및 ChatWindow 컴포넌트

**파일:** `/frontend/src/components/chat/ChatWindow.tsx`

```typescript
"use client";

import React, { useState, useEffect, useRef } from "react";
import { useConversationStore } from "@/store/conversationStore";
import ChatHeader from "./ChatHeader";
import MessageHistory from "./MessageHistory";
import InputArea from "./InputArea";
import { useConversationApi } from "@/hooks/useConversationApi";

interface ChatWindowProps {
  conversationId?: string;
}

export default function ChatWindow({ conversationId }: ChatWindowProps) {
  const {
    currentConversation,
    messages,
    loading,
    setCurrentConversation,
    addMessage,
  } = useConversationStore();
  
  const { getConversation, createConversation } = useConversationApi();
  const messageHistoryRef = useRef<HTMLDivElement>(null);

  // 대화 로드
  useEffect(() => {
    if (conversationId) {
      loadConversation(conversationId);
    }
  }, [conversationId]);

  // 메시지 스크롤 처리
  useEffect(() => {
    if (messageHistoryRef.current) {
      messageHistoryRef.current.scrollTop = messageHistoryRef.current.scrollHeight;
    }
  }, [messages]);

  const loadConversation = async (id: string) => {
    try {
      const conversation = await getConversation(id);
      setCurrentConversation(conversation);
      // Task 3-1에서 구현된 listTurns API 호출하여 메시지 로드
      // (별도 API 호출 필요)
    } catch (error) {
      console.error("Failed to load conversation:", error);
    }
  };

  const handleNewConversation = async () => {
    try {
      const conversation = await createConversation();
      setCurrentConversation(conversation);
      // 메시지 초기화
    } catch (error) {
      console.error("Failed to create conversation:", error);
    }
  };

  return (
    <div className="flex flex-col h-full w-full bg-white">
      {/* Header */}
      {currentConversation ? (
        <ChatHeader
          conversation={currentConversation}
          onNewConversation={handleNewConversation}
        />
      ) : (
        <div className="border-b border-gray-200 p-4 text-center">
          <button
            onClick={handleNewConversation}
            className="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600"
          >
            새로운 대화
          </button>
        </div>
      )}

      {/* Message History */}
      {currentConversation ? (
        <div
          ref={messageHistoryRef}
          className="flex-1 overflow-y-auto p-4 space-y-4"
        >
          {messages.length === 0 ? (
            <div className="text-center text-gray-500 mt-8">
              질문을 입력하여 시작하세요
            </div>
          ) : (
            <MessageHistory messages={messages} />
          )}
          {loading && <div className="text-sm text-gray-400">응답 생성 중...</div>}
        </div>
      ) : (
        <div className="flex-1 flex items-center justify-center bg-gray-50">
          <div className="text-center text-gray-500">
            <p className="text-lg font-semibold mb-2">대화를 시작하세요</p>
            <p className="text-sm">새로운 대화 버튼을 클릭하여 시작합니다</p>
          </div>
        </div>
      )}

      {/* Input Area */}
      {currentConversation && (
        <InputArea conversationId={currentConversation.id} />
      )}
    </div>
  );
}
```

**파일:** `/frontend/src/components/chat/ChatHeader.tsx`

```typescript
"use client";

import React, { useState } from "react";
import { Conversation } from "@/types/conversation";

interface ChatHeaderProps {
  conversation: Conversation;
  onNewConversation: () => void;
}

export default function ChatHeader({
  conversation,
  onNewConversation,
}: ChatHeaderProps) {
  const [showMenu, setShowMenu] = useState(false);

  return (
    <div className="border-b border-gray-200 p-4 flex items-center justify-between bg-white">
      <div>
        <h1 className="text-xl font-bold text-gray-900">{conversation.title}</h1>
        <p className="text-sm text-gray-500">
          {new Date(conversation.created_at).toLocaleDateString()}
        </p>
      </div>

      <div className="flex gap-2">
        <button
          onClick={onNewConversation}
          className="px-3 py-2 text-sm border border-gray-300 rounded-lg hover:bg-gray-50"
        >
          새 대화
        </button>

        <div className="relative">
          <button
            onClick={() => setShowMenu(!showMenu)}
            className="px-3 py-2 text-sm border border-gray-300 rounded-lg hover:bg-gray-50"
          >
            ⋮
          </button>

          {showMenu && (
            <div className="absolute right-0 mt-2 w-40 bg-white border border-gray-200 rounded-lg shadow-lg z-10">
              <button className="w-full px-4 py-2 text-left text-sm hover:bg-gray-50">
                내보내기
              </button>
              <button className="w-full px-4 py-2 text-left text-sm hover:bg-gray-50">
                삭제
              </button>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}
```

### 4-2. MessageBubble 컴포넌트

**파일:** `/frontend/src/components/chat/MessageBubble.tsx`

```typescript
"use client";

import React from "react";
import ReactMarkdown from "react-markdown";
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
                ul: ({ node, ...props }) => <ul className="list-disc list-inside mb-2" {...props} />,
                ol: ({ node, ...props }) => <ol className="list-decimal list-inside mb-2" {...props} />,
                a: ({ node, ...props }) => (
                  <a className="text-blue-600 underline hover:text-blue-800" {...props} />
                ),
              }}
            >
              {message.content}
            </ReactMarkdown>
          </div>
        )}

        {/* Citation 섹션 (Task 3-8에서 추가) */}
        {/* <CitationBlock citations={message.citations} /> */}

        <p className="text-xs opacity-75 mt-2">
          {new Date(message.created_at).toLocaleTimeString()}
        </p>
      </div>
    </div>
  );
}
```

**파일:** `/frontend/src/components/chat/MessageHistory.tsx`

```typescript
"use client";

import React from "react";
import MessageBubble from "./MessageBubble";
import { Message } from "@/types/conversation";

interface MessageHistoryProps {
  messages: Message[];
}

export default function MessageHistory({ messages }: MessageHistoryProps) {
  return (
    <div className="space-y-4">
      {messages.map((message) => (
        <MessageBubble key={message.id} message={message} />
      ))}
    </div>
  );
}
```

### 4-3. InputArea 컴포넌트

**파일:** `/frontend/src/components/chat/InputArea.tsx`

```typescript
"use client";

import React, { useState, useRef } from "react";
import { useConversationStore } from "@/store/conversationStore";
import { useRagApi } from "@/hooks/useRagApi";
import { Message } from "@/types/conversation";

interface InputAreaProps {
  conversationId: string;
}

export default function InputArea({ conversationId }: InputAreaProps) {
  const [input, setInput] = useState("");
  const textareaRef = useRef<HTMLTextAreaElement>(null);
  const {
    addMessage,
    updateStreamingContent,
    loading,
    streaming,
  } = useConversationStore();
  const { answerWithStreaming } = useRagApi();

  const handleSendMessage = async () => {
    if (!input.trim()) return;

    const userMessage: Message = {
      id: `temp-${Date.now()}`,
      role: "user",
      content: input,
      created_at: new Date().toISOString(),
      turn_id: "",
    };

    // 사용자 메시지 추가
    addMessage(userMessage);
    setInput("");

    // 자동 제목 생성 (첫 질문인 경우)
    // TODO: Task 3-1의 updateTitle API 호출

    // SSE 스트리밍으로 응답 수신
    try {
      await answerWithStreaming(input, conversationId, {
        onChunk: (chunk: string) => {
          updateStreamingContent(chunk);
        },
        onComplete: (fullResponse: string, citations: any[]) => {
          const assistantMessage: Message = {
            id: `temp-${Date.now()}-assistant`,
            role: "assistant",
            content: fullResponse,
            created_at: new Date().toISOString(),
            citations: citations,
            turn_id: "",
          };
          addMessage(assistantMessage);
          updateStreamingContent("");
        },
        onError: (error: Error) => {
          console.error("Failed to get response:", error);
          const errorMessage: Message = {
            id: `temp-${Date.now()}-error`,
            role: "assistant",
            content: `오류가 발생했습니다: ${error.message}`,
            created_at: new Date().toISOString(),
            turn_id: "",
          };
          addMessage(errorMessage);
        },
      });
    } catch (error) {
      console.error("Failed to send message:", error);
    }
  };

  const handleKeyDown = (e: React.KeyboardEvent<HTMLTextAreaElement>) => {
    if (e.key === "Enter" && !e.shiftKey) {
      e.preventDefault();
      handleSendMessage();
    }
    // Shift+Enter는 기본 동작(줄바꿈) 허용
  };

  const handleInput = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    setInput(e.target.value);
    // 텍스트 높이에 따라 자동 조정
    if (textareaRef.current) {
      textareaRef.current.style.height = "auto";
      textareaRef.current.style.height = `${Math.min(
        textareaRef.current.scrollHeight,
        200
      )}px`;
    }
  };

  return (
    <div className="border-t border-gray-200 bg-white p-4">
      <div className="flex gap-2">
        <textarea
          ref={textareaRef}
          value={input}
          onChange={handleInput}
          onKeyDown={handleKeyDown}
          placeholder="질문을 입력하세요... (Shift+Enter로 줄바꿈)"
          className="flex-1 px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 resize-none"
          rows={1}
          maxLength={5000}
          disabled={loading || streaming}
        />

        <button
          onClick={handleSendMessage}
          disabled={loading || streaming || !input.trim()}
          className="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 disabled:bg-gray-400 disabled:cursor-not-allowed h-10 flex items-center justify-center"
        >
          {streaming ? "전송 중..." : "전송"}
        </button>
      </div>

      <p className="text-xs text-gray-500 mt-2">
        {input.length} / 5000
      </p>
    </div>
  );
}
```

### 4-4. Zustand 상태 관리 스토어

**파일:** `/frontend/src/store/conversationStore.ts`

```typescript
import { create } from "zustand";
import { Conversation, Message } from "@/types/conversation";

interface ConversationState {
  // 상태
  currentConversation: Conversation | null;
  messages: Message[];
  loading: boolean;
  streaming: boolean;
  streamingContent: string;

  // 액션
  setCurrentConversation: (conversation: Conversation | null) => void;
  addMessage: (message: Message) => void;
  updateStreamingContent: (content: string) => void;
  setLoading: (loading: boolean) => void;
  setStreaming: (streaming: boolean) => void;
  clearConversation: () => void;
}

export const useConversationStore = create<ConversationState>((set) => ({
  currentConversation: null,
  messages: [],
  loading: false,
  streaming: false,
  streamingContent: "",

  setCurrentConversation: (conversation) =>
    set({ currentConversation: conversation, messages: [] }),

  addMessage: (message) =>
    set((state) => ({
      messages: [...state.messages, message],
    })),

  updateStreamingContent: (content) =>
    set({ streamingContent: content }),

  setLoading: (loading) =>
    set({ loading }),

  setStreaming: (streaming) =>
    set({ streaming }),

  clearConversation: () =>
    set({ currentConversation: null, messages: [], streamingContent: "" }),
}));
```

### 4-5. API 호출 계층

**파일:** `/frontend/src/api/conversationApi.ts`

```typescript
import { unwrapEnvelope } from "@/utils/apiUtils";
import { Conversation, Turn } from "@/types/conversation";

export const conversationApi = {
  async getConversation(conversationId: string): Promise<Conversation> {
    const response = await fetch(
      `/api/v1/conversations/${conversationId}`
    );
    const envelope = await response.json();
    return unwrapEnvelope(envelope);
  },

  async createConversation(title: string = "새로운 대화"): Promise<Conversation> {
    const response = await fetch("/api/v1/conversations", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ title }),
    });
    const envelope = await response.json();
    return unwrapEnvelope(envelope);
  },

  async listTurns(conversationId: string): Promise<Turn[]> {
    const response = await fetch(
      `/api/v1/conversations/${conversationId}/turns`
    );
    const envelope = await response.json();
    return unwrapEnvelope(envelope);
  },

  async updateTitle(conversationId: string, title: string): Promise<void> {
    const response = await fetch(
      `/api/v1/conversations/${conversationId}`,
      {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ title }),
      }
    );
    if (!response.ok) throw new Error("Failed to update title");
  },
};
```

**파일:** `/frontend/src/api/ragApi.ts`

```typescript
import { unwrapEnvelope } from "@/utils/apiUtils";
import { useConversationStore } from "@/store/conversationStore";

interface StreamOptions {
  onChunk: (chunk: string) => void;
  onComplete: (fullResponse: string, citations: any[]) => void;
  onError: (error: Error) => void;
}

export const ragApi = {
  async answerWithStreaming(
    query: string,
    conversationId: string,
    options: StreamOptions
  ): Promise<void> {
    const { setStreaming } = useConversationStore.getState();
    setStreaming(true);

    try {
      const response = await fetch("/api/v1/rag/answer", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          query,
          conversation_id: conversationId,
        }),
      });

      if (!response.ok) {
        throw new Error(`Server error: ${response.statusText}`);
      }

      const reader = response.body?.getReader();
      if (!reader) throw new Error("No response stream");

      let fullResponse = "";
      let citations = [];

      const decoder = new TextDecoder();

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value, { stream: true });
        const lines = chunk.split("\n");

        for (const line of lines) {
          if (line.startsWith("data: ")) {
            try {
              const data = JSON.parse(line.slice(6));

              if (data.answer_chunk) {
                fullResponse += data.answer_chunk;
                options.onChunk(data.answer_chunk);
              }

              if (data.citations) {
                citations = data.citations;
              }

              if (data.is_complete) {
                options.onComplete(fullResponse, citations);
              }
            } catch (e) {
              console.error("Failed to parse SSE data", e);
            }
          }
        }
      }
    } catch (error) {
      options.onError(error instanceof Error ? error : new Error(String(error)));
    } finally {
      setStreaming(false);
    }
  },
};
```

### 4-6. 타입 정의

**파일:** `/frontend/src/types/conversation.ts`

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
  retrieval_metadata?: Record<string, any>;
  created_at: string;
}
```

### 4-7. Hook 정의

**파일:** `/frontend/src/hooks/useConversationApi.ts`

```typescript
import { conversationApi } from "@/api/conversationApi";

export function useConversationApi() {
  return {
    getConversation: async (id: string) => {
      return await conversationApi.getConversation(id);
    },
    createConversation: async (title?: string) => {
      return await conversationApi.createConversation(title);
    },
    listTurns: async (conversationId: string) => {
      return await conversationApi.listTurns(conversationId);
    },
    updateTitle: async (conversationId: string, title: string) => {
      return await conversationApi.updateTitle(conversationId, title);
    },
  };
}
```

**파일:** `/frontend/src/hooks/useRagApi.ts`

```typescript
import { ragApi } from "@/api/ragApi";

export function useRagApi() {
  return {
    answerWithStreaming: async (query: string, conversationId: string, options: any) => {
      return await ragApi.answerWithStreaming(query, conversationId, options);
    },
  };
}
```

### 4-8. 유틸리티 함수

**파일:** `/frontend/src/utils/apiUtils.ts`

```typescript
export interface ApiEnvelope<T> {
  success: boolean;
  data?: T;
  error?: string;
  message?: string;
}

export function unwrapEnvelope<T>(envelope: ApiEnvelope<T>): T {
  if (!envelope.success) {
    throw new Error(envelope.error || "API request failed");
  }
  if (!envelope.data) {
    throw new Error("No data in response");
  }
  return envelope.data;
}
```

### 4-9. 단위 테스트

**파일:** `/frontend/src/components/chat/__tests__/ChatWindow.test.tsx`

```typescript
import React from "react";
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import ChatWindow from "../ChatWindow";
import { useConversationStore } from "@/store/conversationStore";

// Mock stores
jest.mock("@/store/conversationStore");
jest.mock("@/hooks/useConversationApi");

describe("ChatWindow", () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test("renders new conversation button when no conversation loaded", () => {
    (useConversationStore as jest.Mock).mockReturnValue({
      currentConversation: null,
      messages: [],
      loading: false,
      streaming: false,
      setCurrentConversation: jest.fn(),
      addMessage: jest.fn(),
    });

    render(<ChatWindow />);
    expect(screen.getByText("대화를 시작하세요")).toBeInTheDocument();
  });

  test("renders message history when conversation exists", () => {
    const mockConversation = {
      id: "conv-1",
      title: "Test Conversation",
      created_at: new Date().toISOString(),
    };

    const mockMessages = [
      {
        id: "msg-1",
        role: "user",
        content: "Hello",
        created_at: new Date().toISOString(),
      },
      {
        id: "msg-2",
        role: "assistant",
        content: "Hi there!",
        created_at: new Date().toISOString(),
      },
    ];

    (useConversationStore as jest.Mock).mockReturnValue({
      currentConversation: mockConversation,
      messages: mockMessages,
      loading: false,
      streaming: false,
      setCurrentConversation: jest.fn(),
      addMessage: jest.fn(),
    });

    render(<ChatWindow />);
    expect(screen.getByText("Test Conversation")).toBeInTheDocument();
  });
});
```

**파일:** `/frontend/src/components/chat/__tests__/MessageBubble.test.tsx`

```typescript
import React from "react";
import { render, screen } from "@testing-library/react";
import MessageBubble from "../MessageBubble";

describe("MessageBubble", () => {
  test("renders user message with correct styling", () => {
    const message = {
      id: "msg-1",
      role: "user" as const,
      content: "Hello",
      created_at: new Date().toISOString(),
      turn_id: "turn-1",
    };

    const { container } = render(<MessageBubble message={message} />);
    const bubble = container.querySelector(".bg-blue-500");
    expect(bubble).toBeInTheDocument();
    expect(screen.getByText("Hello")).toBeInTheDocument();
  });

  test("renders assistant message with markdown", () => {
    const message = {
      id: "msg-2",
      role: "assistant" as const,
      content: "# Hello\n\nThis is **bold**",
      created_at: new Date().toISOString(),
      turn_id: "turn-1",
    };

    const { container } = render(<MessageBubble message={message} />);
    const bubble = container.querySelector(".bg-gray-200");
    expect(bubble).toBeInTheDocument();
  });
});
```

## 5. 산출물

1. **ChatWindow 컴포넌트** (`/frontend/src/components/chat/ChatWindow.tsx`)
   - 3-컬럼 레이아웃 구조
   - 대화 로딩, 새 대화 생성 기능

2. **ChatHeader 컴포넌트** (`/frontend/src/components/chat/ChatHeader.tsx`)
   - 제목, 생성 날짜 표시
   - 옵션 메뉴 (더보기)

3. **MessageBubble 컴포넌트** (`/frontend/src/components/chat/MessageBubble.tsx`)
   - User/Assistant 분기 렌더링
   - 마크다운 지원

4. **MessageHistory 컴포넌트** (`/frontend/src/components/chat/MessageHistory.tsx`)
   - 메시지 시퀀스 렌더링

5. **InputArea 컴포넌트** (`/frontend/src/components/chat/InputArea.tsx`)
   - Multiline 입력, SSE 스트리밍 처리
   - 키보드 단축키 (Enter/Shift+Enter)

6. **Zustand 스토어** (`/frontend/src/store/conversationStore.ts`)
   - 상태 관리 및 액션

7. **API 호출 계층**
   - `conversationApi.ts` (Conversation CRUD)
   - `ragApi.ts` (SSE 스트리밍)

8. **타입 정의** (`/frontend/src/types/conversation.ts`)

9. **Hook 정의**
   - `useConversationApi.ts`
   - `useRagApi.ts`

10. **유틸리티** (`/frontend/src/utils/apiUtils.ts`)

11. **단위 테스트**
    - `ChatWindow.test.tsx`
    - `MessageBubble.test.tsx`
    - `InputArea.test.tsx` (추가 작성)

## 6. 완료 기준

1. ChatWindow가 3-컬럼 레이아웃으로 정상 렌더링되는가?
2. 새 대화 버튼 클릭 시 새 Conversation 생성 API 호출이 작동하는가?
3. 메시지 입력 후 Enter 키로 전송이 정상 작동하는가?
4. SSE 스트리밍으로 응답이 실시간 업데이트되는가?
5. MessageBubble이 User/Assistant 분기 렌더링을 정확하게 하는가?
6. 마크다운 렌더링이 코드 블록, 리스트, 링크 등을 지원하는가?
7. Shift+Enter로 줄바꿈이 정상 작동하는가?
8. 텍스트 입력 제한(5000자)이 적용되는가?
9. 스트리밍 중 로딩 상태가 명확하게 표시되는가?
10. 모든 단위 테스트가 통과하는가?
11. 타입스크립트 컴파일이 정상 작동하는가?

## 7. 작업 지침

### 지침 7-1. SSE 스트리밍 구현

SSE(Server-Sent Events)를 사용하여 API 응답을 청크 단위로 수신한다. 응답 형식:

```json
data: {"answer_chunk":"응답의 일부","is_complete":false}
data: {"answer_chunk":" 계속","is_complete":false}
data: {"citations":[...],"is_complete":true}
```

클라이언트는 각 청크를 수신할 때마다 UI를 즉시 업데이트한다.

### 지침 7-2. 자동 제목 생성

첫 메시지 입력 후 Conversation의 제목이 없으면, 사용자의 첫 질문 일부(최대 50자)로 자동 생성한다. 선택적으로 LLM을 사용한 요약도 가능하다.

```typescript
// 예: 첫 질문이 "Mimir 플랫폼의 권한 관리 구조에 대해 설명해주세요"일 때
// 자동 생성 제목: "Mimir 플랫폼의 권한 관리 구조에..."
```

### 지침 7-3. unwrapEnvelope 패턴

모든 API 응답은 표준 Envelope 형식을 따른다:

```typescript
{
  "success": boolean,
  "data": T,
  "error"?: string,
  "message"?: string
}
```

`unwrapEnvelope` 함수는 envelope에서 data를 추출하거나 에러를 throw한다.

### 지침 7-4. 메시지 ID 임시 생성

UI 상에서 즉시 메시지를 표시하기 위해 클라이언트 측에서 임시 ID(`temp-${timestamp}`)를 생성한다. 서버에서 Turn을 생성한 후 실제 ID로 업데이트한다(Task 3-2에서 처리).

### 지침 7-5. 마크다운 보안

`react-markdown` 라이브러리를 사용할 때, XSS 방지를 위해 HTML 렌더링을 기본적으로 비활성화한다. 신뢰된 콘텐츠만 렌더링하도록 설정한다.

```typescript
<ReactMarkdown
  disallowedElements={["script", "iframe"]}
  unwrapDisallowed
>
  {message.content}
</ReactMarkdown>
```

### 지침 7-6. 로딩 상태 관리

conversationStore의 `loading`과 `streaming` 상태를 구분한다:
- `loading`: 대화 데이터 로드 중
- `streaming`: SSE 스트리밍 응답 수신 중

이 두 상태가 true이면 InputArea의 전송 버튼을 비활성화한다.

### 지침 7-7. 에러 처리

SSE 스트리밍 중 네트워크 에러 또는 서버 에러 발생 시:
1. 에러 메시지를 메시지 히스토리에 추가
2. 사용자에게 재시도 옵션 제공 (선택)
3. 감사 로그에 에러 기록

### 지침 7-8. 접근성 기본 사항

이 Task에서는 최소한의 접근성을 고려한다. Task 3-10에서 완전한 접근성을 구현한다:
- 입력 필드에 label 또는 placeholder 제공
- 버튼에 명확한 텍스트 레이블 제공
- 색상만으로 상태 표현 금지 (로딩 중에 텍스트도 추가)
