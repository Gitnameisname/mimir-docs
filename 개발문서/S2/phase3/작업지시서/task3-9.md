# Task 3-9. 대화 이력 ConversationList + 검색 + 관리

## 1. 작업 목적

**좌측 사이드바**에서 사용자의 이전 대화 목록을 검색하고, 필터(시간, 보관 상태)를 적용하며, 제목 수정, 아카이빙, 영구삭제 등의 관리 기능을 제공한다. 사용자가 과거 대화를 빠르게 찾아 재참조할 수 있도록 지원한다.

## 2. 작업 범위

### 포함 범위

1. **ConversationList 컴포넌트 (좌측 사이드바)**
   - 최근순 정렬된 대화 목록
   - 각 대화 항목: 제목, 생성 날짜, 마지막 업데이트 시간
   - 클릭 시 해당 대화로 네비게이션
   - 로딩 상태 및 빈 상태(empty state) 처리

2. **SearchBox 컴포넌트**
   - 실시간 검색 (또는 debounce 적용)
   - 검색 범위: 대화 제목 + 메시지 내용 (전체텍스트검색)
   - 검색어 입력 시 결과 동적 필터링
   - 검색 초기화 버튼

3. **FilterTabs 컴포넌트**
   - 탭 구성: 전체 | 지난 7일 | 지난 30일 | 보관함 | 삭제됨
   - 각 탭은 서버의 필터 파라미터를 변경 (Task 3-1 API의 status/filter 활용)
   - 선택된 탭 시각적 강조
   - 각 탭의 대화 개수 표시(선택)

4. **무한 스크롤 또는 페이지네이션**
   - 기본: 페이지네이션 (더 명확함)
   - 선택: 무한 스크롤 (더 나은 UX)
   - 초기 로드: 최대 20개
   - "더 보기" 버튼 또는 자동 로드

5. **ConversationItem 컴포넌트 (개별 대화 항목)**
   - 제목 (최대 2줄, ellipsis 처리)
   - 생성 날짜 (상대 시간 "2시간 전")
   - 마지막 업데이트 시간 (선택)
   - hover 시 옵션 메뉴 표시 (점 3개 아이콘)
   - 대화 상태 배지 (active/archived/deleted)

6. **대화 관리 기능**
   - **제목 수정**: inline 편집 또는 모달 다이얼로그
   - **아카이빙**: soft delete (상태를 archived로 변경)
   - **보관함 복원**: archived 대화를 active로 변경
   - **영구삭제**: soft delete를 물리 삭제로 진행 (재확인 모달)
   - **내보내기**: JSON 또는 Markdown 포맷
   - 모든 기능은 Task 3-1의 PATCH/DELETE API 호출

7. **대화 공유 구조 (placeholder)**
   - UI 메뉴 항목으로 "공유" 버튼 준비 (실제 구현은 Phase 6 FG6.2)
   - 비활성화 상태로 표시 또는 "곧 출시" 배지

8. **Zustand 상태 관리**
   - conversationListStore
     - conversations: Conversation[]
     - selectedConversationId: string | null
     - searchQuery: string
     - filterTab: 'all' | 'week' | 'month' | 'archived' | 'deleted'
     - loading: boolean
     - hasMore: boolean (페이지네이션)
     - page: number
   - 액션: setConversations, addConversation, deleteConversation, updateConversation, setSearchQuery, setFilterTab, loadMore

### 제외 범위

- 대화 공유 링크 생성 (Phase 6 FG6.2)
- 대화 공동 편집 (Phase 8)
- 대화 내용 전체 보기 (ChatWindow에서 처리)
- 대화 분석 대시보드 (Phase 6)

## 3. 선행 조건

- Task 3-7 완료: ChatWindow, MessageHistory 컴포넌트
- FG3.1 완료: 대화 조회/검색 API 엔드포인트
  - `GET /api/v1/conversations` (리스트, 페이지네이션, 필터)
  - `PATCH /api/v1/conversations/{id}` (제목 수정)
  - `DELETE /api/v1/conversations/{id}` (삭제)
  - `POST /api/v1/conversations/{id}/archive` (아카이빙)
  - `POST /api/v1/conversations/{id}/export` (내보내기)

## 4. 주요 작업 항목

### 4-1. ConversationList 컴포넌트 (좌측 사이드바)

**파일:** `/frontend/src/components/chat/ConversationList.tsx`

```typescript
"use client";

import React, { useEffect, useState } from "react";
import { useRouter } from "next/navigation";
import SearchBox from "./SearchBox";
import FilterTabs from "./FilterTabs";
import ConversationItem from "./ConversationItem";
import { useConversationListStore } from "@/store/conversationListStore";
import { useConversationApi } from "@/hooks/useConversationApi";

export default function ConversationList() {
  const router = useRouter();
  const {
    conversations,
    selectedConversationId,
    searchQuery,
    filterTab,
    loading,
    hasMore,
    page,
    setConversations,
    setSelectedConversationId,
    setSearchQuery,
    setFilterTab,
    loadMore,
  } = useConversationListStore();

  const { listConversations } = useConversationApi();
  const [loadingMore, setLoadingMore] = useState(false);

  // 초기 로드
  useEffect(() => {
    loadConversations();
  }, [searchQuery, filterTab]);

  const loadConversations = async () => {
    try {
      const result = await listConversations({
        query: searchQuery,
        status: getStatusFilter(filterTab),
        limit: 20,
        offset: 0,
      });
      setConversations(result.conversations);
    } catch (error) {
      console.error("Failed to load conversations:", error);
    }
  };

  const getStatusFilter = (
    tab: "all" | "week" | "month" | "archived" | "deleted"
  ) => {
    switch (tab) {
      case "archived":
        return "archived";
      case "deleted":
        return "deleted";
      default:
        return "active";
    }
  };

  const handleSelectConversation = (id: string) => {
    setSelectedConversationId(id);
    router.push(`/chat/${id}`);
  };

  const handleLoadMore = async () => {
    setLoadingMore(true);
    try {
      const result = await listConversations({
        query: searchQuery,
        status: getStatusFilter(filterTab),
        limit: 20,
        offset: page * 20,
      });
      // 기존 목록에 추가
      setConversations([...conversations, ...result.conversations]);
      loadMore();
    } catch (error) {
      console.error("Failed to load more conversations:", error);
    } finally {
      setLoadingMore(false);
    }
  };

  return (
    <div className="w-64 bg-white border-r border-gray-200 flex flex-col h-full">
      {/* Header */}
      <div className="p-4 border-b border-gray-200">
        <button className="w-full px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 text-sm font-semibold">
          새로운 대화
        </button>
      </div>

      {/* Search */}
      <SearchBox
        value={searchQuery}
        onChange={(query) => setSearchQuery(query)}
      />

      {/* Filter Tabs */}
      <FilterTabs
        activeTab={filterTab}
        onTabChange={(tab) => setFilterTab(tab)}
      />

      {/* Conversation List */}
      <div className="flex-1 overflow-y-auto">
        {loading ? (
          <div className="p-4 text-center text-gray-500 text-sm">
            로딩 중...
          </div>
        ) : conversations.length === 0 ? (
          <div className="p-4 text-center text-gray-500 text-sm">
            대화 없음
          </div>
        ) : (
          <div className="space-y-1 p-2">
            {conversations.map((conversation) => (
              <ConversationItem
                key={conversation.id}
                conversation={conversation}
                isSelected={selectedConversationId === conversation.id}
                onSelect={() => handleSelectConversation(conversation.id)}
              />
            ))}
          </div>
        )}

        {/* Load More Button */}
        {hasMore && !loading && (
          <div className="p-4 text-center">
            <button
              onClick={handleLoadMore}
              disabled={loadingMore}
              className="px-4 py-2 text-sm border border-gray-300 rounded-lg hover:bg-gray-50 disabled:opacity-50"
            >
              {loadingMore ? "로딩 중..." : "더 보기"}
            </button>
          </div>
        )}
      </div>
    </div>
  );
}
```

### 4-2. SearchBox 컴포넌트

**파일:** `/frontend/src/components/chat/SearchBox.tsx`

```typescript
"use client";

import React, { useState, useEffect } from "react";

interface SearchBoxProps {
  value: string;
  onChange: (query: string) => void;
}

export default function SearchBox({ value, onChange }: SearchBoxProps) {
  const [input, setInput] = useState(value);

  // Debounce 적용 (500ms)
  useEffect(() => {
    const timer = setTimeout(() => {
      onChange(input);
    }, 500);

    return () => clearTimeout(timer);
  }, [input, onChange]);

  const handleClear = () => {
    setInput("");
    onChange("");
  };

  return (
    <div className="p-3 border-b border-gray-200">
      <div className="relative">
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="대화 검색..."
          className="w-full px-3 py-2 pr-8 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm"
        />
        {input && (
          <button
            onClick={handleClear}
            className="absolute right-2 top-1/2 transform -translate-y-1/2 text-gray-500 hover:text-gray-700"
          >
            ✕
          </button>
        )}
      </div>
    </div>
  );
}
```

### 4-3. FilterTabs 컴포넌트

**파일:** `/frontend/src/components/chat/FilterTabs.tsx`

```typescript
"use client";

import React from "react";

type FilterTab = "all" | "week" | "month" | "archived" | "deleted";

interface FilterTabsProps {
  activeTab: FilterTab;
  onTabChange: (tab: FilterTab) => void;
}

const tabs: { id: FilterTab; label: string; icon: string }[] = [
  { id: "all", label: "전체", icon: "📋" },
  { id: "week", label: "7일", icon: "📅" },
  { id: "month", label: "30일", icon: "📆" },
  { id: "archived", label: "보관함", icon: "📦" },
  { id: "deleted", label: "삭제됨", icon: "🗑️" },
];

export default function FilterTabs({
  activeTab,
  onTabChange,
}: FilterTabsProps) {
  return (
    <div className="flex gap-1 p-2 border-b border-gray-200 overflow-x-auto">
      {tabs.map((tab) => (
        <button
          key={tab.id}
          onClick={() => onTabChange(tab.id)}
          className={`px-3 py-1 rounded text-xs font-semibold whitespace-nowrap ${
            activeTab === tab.id
              ? "bg-blue-500 text-white"
              : "bg-gray-100 text-gray-700 hover:bg-gray-200"
          }`}
        >
          {tab.icon} {tab.label}
        </button>
      ))}
    </div>
  );
}
```

### 4-4. ConversationItem 컴포넌트

**파일:** `/frontend/src/components/chat/ConversationItem.tsx`

```typescript
"use client";

import React, { useState } from "react";
import ConversationItemMenu from "./ConversationItemMenu";
import { Conversation } from "@/types/conversation";
import { formatRelativeTime } from "@/utils/dateUtils";

interface ConversationItemProps {
  conversation: Conversation;
  isSelected: boolean;
  onSelect: () => void;
}

export default function ConversationItem({
  conversation,
  isSelected,
  onSelect,
}: ConversationItemProps) {
  const [showMenu, setShowMenu] = useState(false);

  const getStatusBadge = (status: string) => {
    switch (status) {
      case "archived":
        return <span className="text-xs bg-yellow-100 text-yellow-700 px-2 py-0.5 rounded">보관됨</span>;
      case "deleted":
        return <span className="text-xs bg-red-100 text-red-700 px-2 py-0.5 rounded">삭제됨</span>;
      default:
        return null;
    }
  };

  return (
    <div
      onClick={onSelect}
      className={`p-3 rounded-lg cursor-pointer transition ${
        isSelected
          ? "bg-blue-100 border-l-4 border-blue-500"
          : "hover:bg-gray-100"
      }`}
    >
      <div className="flex items-start justify-between gap-2">
        <div className="flex-1 min-w-0">
          <h3 className="text-sm font-semibold text-gray-900 truncate">
            {conversation.title || "제목 없음"}
          </h3>
          <p className="text-xs text-gray-500 mt-1">
            {formatRelativeTime(conversation.created_at)}
          </p>
        </div>

        <button
          onClick={(e) => {
            e.stopPropagation();
            setShowMenu(!showMenu);
          }}
          className="text-gray-500 hover:text-gray-700"
        >
          ⋮
        </button>
      </div>

      {/* Status Badge */}
      {conversation.status !== "active" && (
        <div className="mt-2">{getStatusBadge(conversation.status)}</div>
      )}

      {/* Menu */}
      {showMenu && (
        <ConversationItemMenu
          conversation={conversation}
          onClose={() => setShowMenu(false)}
        />
      )}
    </div>
  );
}
```

### 4-5. ConversationItemMenu 컴포넌트

**파일:** `/frontend/src/components/chat/ConversationItemMenu.tsx`

```typescript
"use client";

import React, { useState } from "react";
import TitleEditDialog from "./TitleEditDialog";
import DeleteConfirmDialog from "./DeleteConfirmDialog";
import { useConversationListStore } from "@/store/conversationListStore";
import { useConversationApi } from "@/hooks/useConversationApi";
import { Conversation } from "@/types/conversation";

interface ConversationItemMenuProps {
  conversation: Conversation;
  onClose: () => void;
}

export default function ConversationItemMenu({
  conversation,
  onClose,
}: ConversationItemMenuProps) {
  const [showEditDialog, setShowEditDialog] = useState(false);
  const [showDeleteDialog, setShowDeleteDialog] = useState(false);
  const { updateConversation, deleteConversation } = useConversationListStore();
  const { updateTitle, archiveConversation, deleteConversationApi, exportConversation } =
    useConversationApi();

  const handleEditTitle = async (newTitle: string) => {
    try {
      await updateTitle(conversation.id, newTitle);
      updateConversation(conversation.id, { ...conversation, title: newTitle });
      setShowEditDialog(false);
      onClose();
    } catch (error) {
      console.error("Failed to update title:", error);
    }
  };

  const handleArchive = async () => {
    try {
      await archiveConversation(conversation.id);
      updateConversation(conversation.id, {
        ...conversation,
        status: "archived",
      });
      onClose();
    } catch (error) {
      console.error("Failed to archive:", error);
    }
  };

  const handleDelete = async () => {
    try {
      await deleteConversationApi(conversation.id);
      deleteConversation(conversation.id);
      setShowDeleteDialog(false);
      onClose();
    } catch (error) {
      console.error("Failed to delete:", error);
    }
  };

  const handleExport = async (format: "json" | "markdown") => {
    try {
      await exportConversation(conversation.id, format);
      onClose();
    } catch (error) {
      console.error("Failed to export:", error);
    }
  };

  return (
    <>
      <div className="mt-2 space-y-1 bg-white border border-gray-200 rounded-lg shadow-lg p-1">
        <button
          onClick={() => setShowEditDialog(true)}
          className="w-full px-3 py-2 text-left text-sm hover:bg-gray-50 rounded"
        >
          제목 수정
        </button>

        {conversation.status === "active" && (
          <button
            onClick={handleArchive}
            className="w-full px-3 py-2 text-left text-sm hover:bg-gray-50 rounded"
          >
            보관하기
          </button>
        )}

        {conversation.status === "archived" && (
          <button
            onClick={() => {
              updateConversation(conversation.id, {
                ...conversation,
                status: "active",
              });
              onClose();
            }}
            className="w-full px-3 py-2 text-left text-sm hover:bg-gray-50 rounded"
          >
            보관 해제
          </button>
        )}

        <div className="border-t border-gray-100"></div>

        <button
          onClick={() => handleExport("json")}
          className="w-full px-3 py-2 text-left text-sm hover:bg-gray-50 rounded"
        >
          JSON으로 내보내기
        </button>

        <button
          onClick={() => handleExport("markdown")}
          className="w-full px-3 py-2 text-left text-sm hover:bg-gray-50 rounded"
        >
          Markdown으로 내보내기
        </button>

        <div className="border-t border-gray-100"></div>

        <button
          onClick={() => setShowDeleteDialog(true)}
          className="w-full px-3 py-2 text-left text-sm text-red-600 hover:bg-red-50 rounded"
        >
          삭제
        </button>
      </div>

      {/* Dialogs */}
      {showEditDialog && (
        <TitleEditDialog
          currentTitle={conversation.title}
          onSave={handleEditTitle}
          onCancel={() => setShowEditDialog(false)}
        />
      )}

      {showDeleteDialog && (
        <DeleteConfirmDialog
          onConfirm={handleDelete}
          onCancel={() => setShowDeleteDialog(false)}
        />
      )}
    </>
  );
}
```

### 4-6. TitleEditDialog 컴포넌트

**파일:** `/frontend/src/components/chat/TitleEditDialog.tsx`

```typescript
"use client";

import React, { useState } from "react";

interface TitleEditDialogProps {
  currentTitle: string;
  onSave: (newTitle: string) => void;
  onCancel: () => void;
}

export default function TitleEditDialog({
  currentTitle,
  onSave,
  onCancel,
}: TitleEditDialogProps) {
  const [input, setInput] = useState(currentTitle);

  const handleSave = () => {
    if (input.trim()) {
      onSave(input.trim());
    }
  };

  return (
    <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div className="bg-white rounded-lg shadow-lg p-6 max-w-md w-full">
        <h2 className="text-lg font-bold mb-4">제목 수정</h2>

        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          maxLength={256}
          className="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 mb-4"
          autoFocus
        />

        <p className="text-xs text-gray-500 mb-4">
          {input.length} / 256
        </p>

        <div className="flex gap-2">
          <button
            onClick={handleSave}
            className="flex-1 px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600"
          >
            저장
          </button>
          <button
            onClick={onCancel}
            className="flex-1 px-4 py-2 bg-gray-300 text-gray-800 rounded-lg hover:bg-gray-400"
          >
            취소
          </button>
        </div>
      </div>
    </div>
  );
}
```

### 4-7. DeleteConfirmDialog 컴포넌트

**파일:** `/frontend/src/components/chat/DeleteConfirmDialog.tsx`

```typescript
"use client";

import React from "react";

interface DeleteConfirmDialogProps {
  onConfirm: () => void;
  onCancel: () => void;
}

export default function DeleteConfirmDialog({
  onConfirm,
  onCancel,
}: DeleteConfirmDialogProps) {
  return (
    <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div className="bg-white rounded-lg shadow-lg p-6 max-w-md w-full">
        <h2 className="text-lg font-bold text-red-600 mb-4">대화 삭제</h2>

        <p className="text-gray-700 mb-6">
          이 대화를 영구적으로 삭제하시겠습니까? 이 작업은 되돌릴 수 없습니다.
        </p>

        <div className="flex gap-2">
          <button
            onClick={onConfirm}
            className="flex-1 px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700"
          >
            삭제
          </button>
          <button
            onClick={onCancel}
            className="flex-1 px-4 py-2 bg-gray-300 text-gray-800 rounded-lg hover:bg-gray-400"
          >
            취소
          </button>
        </div>
      </div>
    </div>
  );
}
```

### 4-8. Zustand 상태 관리

**파일:** `/frontend/src/store/conversationListStore.ts`

```typescript
import { create } from "zustand";
import { Conversation } from "@/types/conversation";

type FilterTab = "all" | "week" | "month" | "archived" | "deleted";

interface ConversationListState {
  // 상태
  conversations: Conversation[];
  selectedConversationId: string | null;
  searchQuery: string;
  filterTab: FilterTab;
  loading: boolean;
  hasMore: boolean;
  page: number;

  // 액션
  setConversations: (conversations: Conversation[]) => void;
  addConversation: (conversation: Conversation) => void;
  updateConversation: (id: string, updates: Partial<Conversation>) => void;
  deleteConversation: (id: string) => void;
  setSelectedConversationId: (id: string | null) => void;
  setSearchQuery: (query: string) => void;
  setFilterTab: (tab: FilterTab) => void;
  setLoading: (loading: boolean) => void;
  loadMore: () => void;
}

export const useConversationListStore = create<ConversationListState>((set) => ({
  conversations: [],
  selectedConversationId: null,
  searchQuery: "",
  filterTab: "all",
  loading: false,
  hasMore: true,
  page: 0,

  setConversations: (conversations) =>
    set({ conversations, page: 0, loading: false }),

  addConversation: (conversation) =>
    set((state) => ({
      conversations: [conversation, ...state.conversations],
    })),

  updateConversation: (id, updates) =>
    set((state) => ({
      conversations: state.conversations.map((c) =>
        c.id === id ? { ...c, ...updates } : c
      ),
    })),

  deleteConversation: (id) =>
    set((state) => ({
      conversations: state.conversations.filter((c) => c.id !== id),
    })),

  setSelectedConversationId: (id) =>
    set({ selectedConversationId: id }),

  setSearchQuery: (query) =>
    set({ searchQuery: query, page: 0 }),

  setFilterTab: (tab) =>
    set({ filterTab: tab, page: 0 }),

  setLoading: (loading) =>
    set({ loading }),

  loadMore: () =>
    set((state) => ({ page: state.page + 1 })),
}));
```

### 4-9. API 호출 확장

**파일:** `/frontend/src/api/conversationApi.ts` (추가)

```typescript
// 기존 코드에 추가

export const conversationApi = {
  // ... 기존 메서드들

  async listConversations(filters: {
    query?: string;
    status?: string;
    limit?: number;
    offset?: number;
  }): Promise<{ conversations: Conversation[]; total: number }> {
    const params = new URLSearchParams();
    if (filters.query) params.append("query", filters.query);
    if (filters.status) params.append("status", filters.status);
    if (filters.limit) params.append("limit", String(filters.limit));
    if (filters.offset) params.append("offset", String(filters.offset));

    const response = await fetch(
      `/api/v1/conversations?${params.toString()}`
    );
    const envelope = await response.json();
    return unwrapEnvelope(envelope);
  },

  async archiveConversation(conversationId: string): Promise<void> {
    const response = await fetch(
      `/api/v1/conversations/${conversationId}/archive`,
      {
        method: "POST",
      }
    );
    if (!response.ok) throw new Error("Failed to archive");
  },

  async deleteConversationApi(conversationId: string): Promise<void> {
    const response = await fetch(
      `/api/v1/conversations/${conversationId}`,
      {
        method: "DELETE",
      }
    );
    if (!response.ok) throw new Error("Failed to delete");
  },

  async exportConversation(
    conversationId: string,
    format: "json" | "markdown"
  ): Promise<void> {
    const response = await fetch(
      `/api/v1/conversations/${conversationId}/export?format=${format}`
    );
    if (!response.ok) throw new Error("Failed to export");

    const blob = await response.blob();
    const url = window.URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `conversation.${format === "json" ? "json" : "md"}`;
    document.body.appendChild(a);
    a.click();
    window.URL.revokeObjectURL(url);
  },
};
```

### 4-10. 유틸리티 함수

**파일:** `/frontend/src/utils/dateUtils.ts`

```typescript
export function formatRelativeTime(date: string): string {
  const now = new Date();
  const then = new Date(date);
  const diffMs = now.getTime() - then.getTime();
  const diffMins = Math.floor(diffMs / (1000 * 60));
  const diffHours = Math.floor(diffMs / (1000 * 60 * 60));
  const diffDays = Math.floor(diffMs / (1000 * 60 * 60 * 24));

  if (diffMins < 1) return "방금 전";
  if (diffMins < 60) return `${diffMins}분 전`;
  if (diffHours < 24) return `${diffHours}시간 전`;
  if (diffDays < 7) return `${diffDays}일 전`;

  return then.toLocaleDateString("ko-KR");
}
```

### 4-11. 단위 테스트

**파일:** `/frontend/src/components/chat/__tests__/ConversationList.test.tsx`

```typescript
import React from "react";
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import ConversationList from "../ConversationList";
import { useConversationListStore } from "@/store/conversationListStore";

jest.mock("@/store/conversationListStore");
jest.mock("next/navigation", () => ({
  useRouter: () => ({
    push: jest.fn(),
  }),
}));

describe("ConversationList", () => {
  beforeEach(() => {
    (useConversationListStore as jest.Mock).mockReturnValue({
      conversations: [
        {
          id: "conv-1",
          title: "Test Conversation 1",
          created_at: new Date().toISOString(),
          status: "active",
        },
      ],
      selectedConversationId: null,
      searchQuery: "",
      filterTab: "all",
      loading: false,
      hasMore: false,
      page: 0,
      setConversations: jest.fn(),
      setSelectedConversationId: jest.fn(),
      setSearchQuery: jest.fn(),
      setFilterTab: jest.fn(),
      setLoading: jest.fn(),
      loadMore: jest.fn(),
    });
  });

  test("renders conversation list", () => {
    render(<ConversationList />);
    expect(screen.getByText("Test Conversation 1")).toBeInTheDocument();
  });

  test("shows new conversation button", () => {
    render(<ConversationList />);
    expect(screen.getByText("새로운 대화")).toBeInTheDocument();
  });

  test("filters conversations by tab", () => {
    const mockSetFilterTab = jest.fn();
    (useConversationListStore as jest.Mock).mockReturnValue({
      conversations: [],
      selectedConversationId: null,
      searchQuery: "",
      filterTab: "all",
      loading: false,
      hasMore: false,
      page: 0,
      setConversations: jest.fn(),
      setSelectedConversationId: jest.fn(),
      setSearchQuery: jest.fn(),
      setFilterTab: mockSetFilterTab,
      setLoading: jest.fn(),
      loadMore: jest.fn(),
    });

    render(<ConversationList />);
    const weekTab = screen.getByText("7일");
    fireEvent.click(weekTab);
    expect(mockSetFilterTab).toHaveBeenCalledWith("week");
  });
});
```

## 5. 산출물

1. **ConversationList 컴포넌트** (`/frontend/src/components/chat/ConversationList.tsx`)
2. **SearchBox 컴포넌트** (`/frontend/src/components/chat/SearchBox.tsx`)
3. **FilterTabs 컴포넌트** (`/frontend/src/components/chat/FilterTabs.tsx`)
4. **ConversationItem 컴포넌트** (`/frontend/src/components/chat/ConversationItem.tsx`)
5. **ConversationItemMenu 컴포넌트** (`/frontend/src/components/chat/ConversationItemMenu.tsx`)
6. **TitleEditDialog 컴포넌트** (`/frontend/src/components/chat/TitleEditDialog.tsx`)
7. **DeleteConfirmDialog 컴포넌트** (`/frontend/src/components/chat/DeleteConfirmDialog.tsx`)
8. **Zustand 스토어** (`/frontend/src/store/conversationListStore.ts`)
9. **API 호출 확장** (`/frontend/src/api/conversationApi.ts` 추가)
10. **유틸리티** (`/frontend/src/utils/dateUtils.ts`)
11. **단위 테스트** (`ConversationList.test.tsx`)

## 6. 완료 기준

1. ConversationList가 좌측 사이드바에 정상 렌더링되는가?
2. 대화 목록이 최근순으로 정렬되는가?
3. SearchBox에서 검색 시 목록이 동적 필터링되는가?
4. FilterTabs 탭 클릭 시 필터가 적용되는가?
5. ConversationItem 클릭 시 해당 대화로 네비게이션되는가?
6. 옵션 메뉴가 정상적으로 표시되고 기능하는가?
7. 제목 수정 다이얼로그가 정상 작동하는가?
8. 보관/보관 해제 기능이 작동하는가?
9. 삭제 재확인 다이얼로그가 정상 작동하는가?
10. 내보내기 기능이 JSON/Markdown 포맷으로 정상 작동하는가?
11. 페이지네이션 또는 무한 스크롤이 "더 보기" 버튼으로 작동하는가?
12. 로딩 상태 및 빈 상태가 명확하게 표시되는가?
13. 모든 단위 테스트가 통과하는가?

## 7. 작업 지침

### 지침 7-1. 검색 디바운싱

SearchBox에서 입력할 때마다 즉시 API 호출하지 않고, debounce(500ms)를 적용하여 네트워크 요청을 줄인다.

```typescript
const [input, setInput] = useState(value);

useEffect(() => {
  const timer = setTimeout(() => {
    onChange(input);  // 500ms 후 실제 검색 수행
  }, 500);

  return () => clearTimeout(timer);
}, [input, onChange]);
```

### 지침 7-2. 필터 탭 구현

시간 필터(7일/30일)는 `created_at` 필드를 기준으로 서버에서 필터링한다. 보관함/삭제됨은 `status` 필드로 필터링한다.

```typescript
// 서버 API 쿼리 예시
GET /api/v1/conversations?status=active&created_after=2026-04-10

// archived/deleted는 status 파라미터로 처리
GET /api/v1/conversations?status=archived
```

### 지침 7-3. 상대 시간 표시

ConversationItem의 생성 날짜는 절대 시간이 아니라 상대 시간("2시간 전")으로 표시한다. `formatRelativeTime` 함수를 사용한다.

### 지침 7-4. 옵션 메뉴 위치

ConversationItem의 호버 또는 클릭 시 옵션 메뉴를 표시한다. 선택사항으로 메뉴를 별도 모달이 아닌 Popover/Dropdown으로 구현하여 공간을 절약할 수 있다.

### 지침 7-5. 내보내기 기능

내보내기 후 브라우저의 다운로드 기능을 사용하여 파일을 사용자 로컬에 저장한다:

```typescript
const blob = await response.blob();
const url = window.URL.createObjectURL(blob);
const a = document.createElement("a");
a.href = url;
a.download = "conversation.json";
a.click();
```

### 지침 7-6. 권한 필터링

대화 목록 조회 시 ACL이 자동으로 적용되어 사용자가 접근 가능한 대화만 반환된다. 프론트엔드에서는 별도의 권한 검사 불필요.

### 지침 7-7. 상태 배지

대화 상태(archived/deleted)를 시각적으로 구분한다:
- active: 배지 없음
- archived: 노란색 "보관됨"
- deleted: 빨간색 "삭제됨"
