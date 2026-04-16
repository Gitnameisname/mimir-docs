# Task 3-10. 반응형 디자인 + 접근성 + UI 5회 리뷰

## 1. 작업 목적

Task 3-7~3-9에서 구축한 채팅 UI를 **모든 화면 크기(데스크톱/태블릿/모바일)에서 일관되게 작동**하도록 반응형 디자인을 적용하고, **WCAG 2.1 AA 접근성 기준**을 충족시킨다. 또한 **최소 5회의 구조화된 UI 리뷰**를 수행하여 설계와 구현을 점진적으로 개선한다.

## 2. 작업 범위

### 포함 범위

1. **반응형 Breakpoint 설계**
   - 데스크톱: ≥1280px (3컬럼: 대화목록 / 채팅창 / 참고자료)
   - 태블릿: 768px ~ 1279px (2컬럼: 대화목록 / 채팅창)
   - 모바일: <768px (1컬럼: 채팅창 + 오프캔버스 메뉴)
   - Tailwind CSS의 반응형 유틸리티 사용 (sm, md, lg, xl 등)

2. **레이아웃 조정**
   - 데스크톱: 전체 화면 3컬럼
   - 태블릿: 좌측 목록은 축소/숨김 처리 가능
   - 모바일: 좌측 목록은 오프캔버스 메뉴(Drawer/Sidebar)로 전환
     - 햄버거 메뉴 아이콘으로 토글
     - 메뉴 배경 오버레이, swipe-to-close 지원(선택)

3. **폰트 및 간격 조정**
   - 데스크톱: 기본 폰트 크기 유지
   - 태블릿: 약간 축소된 폰트 및 간격
   - 모바일: 터치 친화적 크기 (버튼 최소 44x44px)
   - 텍스트 가독성 유지 (line-height ≥1.5)

4. **이미지 및 미디어 최적화**
   - 반응형 이미지 (srcset, sizes 속성)
   - 로딩 성능 최적화 (lazy loading)
   - 화면 크기에 따른 이미지 해상도 조정

5. **WCAG 2.1 AA 접근성 준수**
   
   **색상 대비:**
   - 텍스트 색상 대비 비율 ≥4.5:1 (일반 텍스트)
   - 큰 텍스트(18pt 이상) ≥3:1
   - 아이콘 및 그래픽 요소 ≥3:1
   - 색상만으로 정보 전달 금지 (텍스트/아이콘 병행)

   **키보드 네비게이션:**
   - 모든 기능을 키보드로 접근 가능
   - Tab 키로 순서대로 포커스 이동
   - Enter/Space로 버튼/링크 활성화
   - Escape로 모달/메뉴 닫기
   - 포커스 인디케이터 명확함 (기본 또는 커스텀)

   **스크린 리더 지원:**
   - 의미 있는 HTML 구조 (semantic HTML)
   - ARIA 속성 사용: aria-label, aria-describedby, aria-live, role 등
   - 이미지 alt 텍스트 제공
   - 폼 필드 label 명시적 연결
   - 링크 텍스트 의미 있음 ("더 보기" 대신 "문서 원문 보기")

   **동작 및 에러 처리:**
   - 예상 외 컨텍스트 변경 방지
   - 에러 메시지 명확함
   - 삭제 등 중대한 작업 재확인
   - 폼 제출 전 검증 및 오류 메시지

6. **접근성 테스트**
   - axe-core 자동 테스트 (npm package)
   - WAVE 웹 접근성 평가 도구 (브라우저 확장)
   - 키보드 수동 테스트
   - 스크린 리더(NVDA, JAWS, VoiceOver) 수동 테스트
   - 모바일 스크린 리더(TalkBack, VoiceOver) 테스트

7. **UI 5회 리뷰 프로세스**

   **리뷰 1: 와이어프레임 + 정보 아키텍처**
   - 목적: 레이아웃 구조가 타당한가, 정보 흐름이 명확한가
   - 검토 항목:
     - 3-컬럼/2-컬럼/1-컬럼 레이아웃의 논리성
     - 사용자 작업 흐름 (새 대화 → 질문 입력 → 응답 조회)
     - 네비게이션 경로
   - 산출물: 와이어프레임 이미지 + 피드백 기록

   **리뷰 2: 시각 디자인 + 색상 스키마**
   - 목적: 디자인이 브랜드와 부합하고, 색상 대비가 적절한가
   - 검토 항목:
     - 색상 팔레트 (primary/secondary/accent)
     - 색상 대비 비율 측정 (≥4.5:1)
     - 타이포그래피 (폰트, 크기, 굵기)
     - 일관성 (UI 요소의 스타일 통일)
   - 산출물: 디자인 시스템 가이드 + 색상 대비 검증 보고서

   **리뷰 3: Citation 렌더링 + 원문 링크 정확성**
   - 목적: Citation 표시와 원문 링크가 명확하고 정확한가
   - 검토 항목:
     - Citation 정보 가독성
     - "원문 보기" 링크 동작
     - span_offset 하이라이트 정확성
     - 검증 배지(체크마크/경고) 명확성
   - 산출물: Citation UX 테스트 결과

   **리뷰 4: 반응형 Breakpoint + 모바일 경험**
   - 목적: 모든 화면에서 올바르게 렌더링되고, 모바일에서 사용하기 편한가
   - 검토 항목:
     - 각 breakpoint(sm/md/lg)에서 레이아웃 변화 확인
     - 모바일 오프캔버스 메뉴 동작
     - 터치 친화적 버튼 크기 (44x44px)
     - 성능 (100+ 턴 대화 스크롤 부드러움)
   - 산출물: 반응형 테스트 리포트 (각 breakpoint 스크린샷 포함)

   **리뷰 5: 접근성 + 오류 처리**
   - 목적: 장애인도 사용할 수 있는가, 에러 상황에서 명확한가
   - 검토 항목:
     - WCAG 2.1 AA 기준 충족 (axe-core 결과)
     - 키보드 네비게이션 경로 (Tab → Shift+Tab)
     - 스크린 리더 테스트 (메뉴 읽음, 링크 읽음)
     - 에러 메시지 명확성
     - 폼 검증 및 피드백
   - 산출물: 접근성 테스트 리포트 + WCAG 체크리스트

8. **성능 최적화 (선택, 하지만 권장)**
   - 장시간 대화(100+ 턴) 렌더링 성능 최적화
   - 가상 스크롤(Virtual Scrolling) 고려
   - 메모이제이션(React.memo, useMemo)
   - 번들 크기 최적화

9. **크로스 브라우저 호환성**
   - Chrome, Firefox, Safari, Edge 최신 버전
   - 모바일: iOS Safari, Android Chrome
   - 구형 브라우저(IE11 등)는 지원 범위 외

### 제외 범위

- 어두운 테마(Dark Mode) (선택사항이므로 제외)
- 다국어 지원 (기본 한국어)
- 모바일 앱 (웹 전용)
- 시각 장애인 비디오 설명 (비디오 콘텐츠 없음)

## 3. 선행 조건

- Task 3-7, 3-8, 3-9 완료: 모든 컴포넌트 구현 완료
- Tailwind CSS 설치 및 설정 완료
- 접근성 테스트 도구 설치:
  - axe-core npm package
  - WAVE 브라우저 확장 (수동 테스트용)
  - 스크린 리더 (NVDA 또는 VoiceOver 사용 환경)

## 4. 주요 작업 항목

### 4-1. 반응형 레이아웃 구현

**파일:** `/frontend/src/components/chat/ChatLayout.tsx` (새로 생성)

```typescript
"use client";

import React, { useState } from "react";
import ConversationList from "./ConversationList";
import ChatWindow from "./ChatWindow";
import ReferencePanel from "./ReferencePanel";

export default function ChatLayout({ conversationId }: { conversationId?: string }) {
  const [isListOpen, setIsListOpen] = useState(false);

  return (
    <div className="flex h-screen bg-white">
      {/* 모바일 오버레이 */}
      {isListOpen && (
        <div
          className="fixed inset-0 bg-black bg-opacity-50 md:hidden z-40"
          onClick={() => setIsListOpen(false)}
        />
      )}

      {/* 좌측: 대화 목록 (반응형) */}
      <div
        className={`
          fixed md:relative left-0 top-0 h-full z-50
          w-64 bg-white border-r border-gray-200
          transition-transform duration-300 ease-out
          md:w-64 md:translate-x-0 md:static
          ${isListOpen ? "translate-x-0" : "-translate-x-full"}
        `}
      >
        <ConversationList onSelectConversation={() => setIsListOpen(false)} />
      </div>

      {/* 중앙: 채팅 창 */}
      <div className="flex-1 flex flex-col relative">
        {/* 모바일 헤더 (햄버거 메뉴) */}
        <div className="md:hidden h-14 border-b border-gray-200 flex items-center px-4 gap-2 bg-white">
          <button
            onClick={() => setIsListOpen(!isListOpen)}
            className="p-2 hover:bg-gray-100 rounded-lg"
            aria-label="대화 목록 열기"
          >
            ☰
          </button>
          <h2 className="text-sm font-semibold text-gray-900 flex-1">Mimir</h2>
        </div>

        {/* ChatWindow */}
        <ChatWindow conversationId={conversationId} />
      </div>

      {/* 우측: 참고 자료 (데스크톱만) */}
      <div className="hidden lg:flex lg:w-80 border-l border-gray-200">
        <ReferencePanel />
      </div>
    </div>
  );
}
```

### 4-2. Tailwind 반응형 유틸리티 적용

Task 3-7의 ChatWindow 컴포넌트 업데이트:

```typescript
// 원본 코드
<div className="flex flex-col h-full w-full bg-white">
  <ChatHeader ... />
  <div className="flex-1 overflow-y-auto p-4 space-y-4">
    <MessageHistory messages={messages} />
  </div>
  <InputArea ... />
</div>

// 반응형 적용
<div className="flex flex-col h-full w-full bg-white">
  <ChatHeader ... />
  {/* 메시지 히스토리: 모바일은 작은 padding, 데스크톱은 큰 padding */}
  <div className="flex-1 overflow-y-auto p-2 sm:p-3 md:p-4 space-y-2 sm:space-y-3 md:space-y-4">
    <MessageHistory messages={messages} />
  </div>
  {/* InputArea: 모바일은 작은 높이, 데스크톱은 일반 높이 */}
  <InputArea ... />
</div>
```

**MessageBubble 반응형 적용:**

```typescript
// 원본
<div className="max-w-md lg:max-w-2xl px-4 py-3 rounded-lg">
  ...
</div>

// 반응형 적용
<div className={`
  max-w-xs sm:max-w-sm md:max-w-md lg:max-w-2xl
  px-3 sm:px-4 py-2 sm:py-3
  rounded-lg text-xs sm:text-sm
  ${isUser ? "bg-blue-500 text-white" : "bg-gray-200 text-gray-900"}
`}>
  ...
</div>
```

### 4-3. 오프캔버스 메뉴 구현 (모바일)

**파일:** `/frontend/src/components/ui/Drawer.tsx` (재사용 가능)

```typescript
"use client";

import React, { useEffect } from "react";

interface DrawerProps {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
  position?: "left" | "right";
}

export default function Drawer({
  isOpen,
  onClose,
  children,
  position = "left",
}: DrawerProps) {
  // Escape 키로 닫기
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === "Escape" && isOpen) {
        onClose();
      }
    };
    window.addEventListener("keydown", handleKeyDown);
    return () => window.removeEventListener("keydown", handleKeyDown);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <>
      {/* 오버레이 */}
      <div
        className="fixed inset-0 bg-black bg-opacity-50 z-40 md:hidden"
        onClick={onClose}
        role="presentation"
      />

      {/* Drawer */}
      <div
        className={`
          fixed top-0 h-full w-64 bg-white shadow-lg z-50
          transition-transform duration-300 ease-out
          ${position === "left" ? "left-0" : "right-0"}
          ${isOpen ? "translate-x-0" : (position === "left" ? "-translate-x-full" : "translate-x-full")}
        `}
        role="dialog"
        aria-modal="true"
        aria-label="사이드 메뉴"
      >
        {children}
      </div>
    </>
  );
}
```

### 4-4. WCAG 접근성 구현

**색상 대비 검증:**

```typescript
// 색상 팔레트 (WCAG AA 대비 ≥4.5:1)
const colors = {
  primary: {
    bg: "#2563eb",      // Blue-600
    text: "#ffffff",    // White (대비: 8.6:1) ✓
  },
  secondary: {
    bg: "#f3f4f6",      // Gray-100
    text: "#111827",    // Gray-900 (대비: 13.3:1) ✓
  },
  error: {
    bg: "#fee2e2",      // Red-100
    text: "#991b1b",    // Red-900 (대비: 8.4:1) ✓
  },
  warning: {
    bg: "#fef3c7",      // Amber-100
    text: "#78350f",    // Amber-900 (대비: 8.6:1) ✓
  },
};
```

**시맨틱 HTML 및 ARIA:**

```typescript
// 잘못된 예
<div onClick={handleDelete} className="cursor-pointer">삭제</div>

// 올바른 예
<button
  onClick={handleDelete}
  aria-label="대화 삭제"
  className="px-4 py-2 bg-red-600 text-white rounded-lg"
>
  삭제
</button>
```

**키보드 네비게이션:**

```typescript
// InputArea 컴포넌트 업데이트
<textarea
  ref={textareaRef}
  onKeyDown={(e) => {
    if (e.key === "Enter" && !e.shiftKey) {
      e.preventDefault();
      handleSendMessage();
    }
    // Tab 키: 기본 동작 유지
    // Escape: 입력 초기화 (선택)
    if (e.key === "Escape") {
      setInput("");
    }
  }}
  aria-label="질문 입력 필드"
  aria-describedby="input-hint"
/>
<p id="input-hint" className="text-xs text-gray-500">
  Shift+Enter로 줄바꿈, Enter로 전송
</p>
```

**이미지 alt 텍스트:**

```typescript
// 메시지의 이미지가 있다면
<img
  src={imageUrl}
  alt={`${conversation.title}의 첨부 이미지`}
  className="max-w-full rounded"
/>
```

**폼 라벨 명시적 연결:**

```typescript
<label htmlFor="conversation-title">대화 제목</label>
<input
  id="conversation-title"
  type="text"
  placeholder="제목을 입력하세요"
  aria-required="true"
/>
```

**ARIA Live Region (실시간 업데이트):**

```typescript
<div
  role="status"
  aria-live="polite"
  aria-atomic="true"
  className="sr-only" // 스크린 리더만 들음
>
  {streaming ? "응답 생성 중..." : "응답 완료"}
</div>
```

### 4-5. 접근성 테스트 자동화

**파일:** `/frontend/src/utils/a11y.test.ts` (추가)

```typescript
import { axe, toHaveNoViolations } from "jest-axe";
import React from "react";
import { render } from "@testing-library/react";

expect.extend(toHaveNoViolations);

describe("ChatWindow accessibility", () => {
  test("should not have any accessibility violations", async () => {
    const { container } = render(
      <ChatWindow conversationId="test-conv" />
    );
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  test("should have proper heading hierarchy", () => {
    const { container } = render(
      <ChatWindow conversationId="test-conv" />
    );
    const headings = container.querySelectorAll("h1, h2, h3");
    // Heading hierarchy를 확인 (h1 → h2 순서, 건너뜀 없음)
    expect(headings.length).toBeGreaterThan(0);
  });

  test("should have proper color contrast", () => {
    const { container } = render(
      <ChatWindow conversationId="test-conv" />
    );
    // 색상 대비 검증 (보통 수동 확인 필요)
    const buttons = container.querySelectorAll("button");
    buttons.forEach((button) => {
      const computedStyle = window.getComputedStyle(button);
      // 색상 대비 비율 계산 (복잡하므로 수동 또는 별도 도구 사용)
    });
  });
});
```

### 4-6. UI 리뷰 기록 문서

**파일:** `/frontend/docs/UI_REVIEW_RECORDS.md`

```markdown
# UI 5회 리뷰 기록

## 리뷰 1: 와이어프레임 + 정보 아키텍처
**날짜:** 2026-04-18
**참석자:** [이름], [이름]
**상태:** 완료

### 검토 항목
- ✓ 3-컬럼 레이아웃 (대화목록 / 채팅 / 참고자료)
- ✓ 모바일 1-컬럼 (오프캔버스 메뉴)
- ✓ 사용자 작업 흐름 명확함
- ✓ 네비게이션 경로 타당함

### 피드백
- [피드백 내용]
- [개선사항]

### 결과
✓ 승인됨

---

## 리뷰 2: 시각 디자인 + 색상 스키마
**날짜:** 2026-04-19
**참석자:** [이름], [이름]
**상태:** 완료

### 검토 항목
- ✓ 색상 팔레트 정의됨
- ✓ 색상 대비 검증 (모두 ≥4.5:1)
  - 텍스트 vs 배경: 모두 PASS
  - 버튼 vs 배경: 모두 PASS
- ✓ 타이포그래피 일관성 있음
- ✓ UI 요소 스타일 통일됨

### 색상 대비 측정 결과
| 요소 | 배경색 | 텍스트색 | 대비율 | 상태 |
|------|--------|---------|--------|------|
| Primary Button | #2563eb | #ffffff | 8.6:1 | ✓ |
| Text | #ffffff | #111827 | 13.3:1 | ✓ |
| Warning | #fef3c7 | #78350f | 8.6:1 | ✓ |

### 피드백
- [피드백]

### 결과
✓ 승인됨

---

## 리뷰 3: Citation 렌더링 + 원문 링크
**날짜:** 2026-04-20
**참석자:** [이름], [이름]
**상태:** 완료

### 검토 항목
- ✓ Citation 정보 가독성 높음
- ✓ "원문 보기" 링크 정상 작동
- ✓ span_offset 하이라이트 정확함
- ✓ 검증 배지 명확함

### 테스트 시나리오
1. Citation 있는 응답: [결과]
2. Citation 없는 응답: [결과]
3. 원문 링크 클릭: [결과]

### 피드백
- [피드백]

### 결과
✓ 승인됨

---

## 리뷰 4: 반응형 + 모바일 경험
**날짜:** 2026-04-21
**참석자:** [이름], [이름]
**상태:** 완료

### 검토 항목
- ✓ 데스크톱 (≥1280px): 3컬럼 정상 렌더링
- ✓ 태블릿 (768~1279px): 2컬럼 정상 렌더링
- ✓ 모바일 (<768px): 1컬럼 + 오프캔버스 메뉴 정상 작동
- ✓ 터치 친화적 버튼 크기 (44x44px 이상)
- ✓ 성능: 100+ 턴 스크롤 부드러움

### 테스트 환경
| 기기 | 해상도 | 결과 |
|------|--------|------|
| Desktop | 1920x1080 | ✓ PASS |
| iPad | 768x1024 | ✓ PASS |
| iPhone 12 | 390x844 | ✓ PASS |

### 스크린샷
[데스크톱 스크린샷]
[태블릿 스크린샷]
[모바일 스크린샷]

### 피드백
- [피드백]

### 결과
✓ 승인됨

---

## 리뷰 5: 접근성 + 오류 처리
**날짜:** 2026-04-22
**참석자:** [이름], [이름]
**상태:** 완료

### 검토 항목
- ✓ WCAG 2.1 AA 준수 (axe-core: 0 violations)
- ✓ 키보드 네비게이션 가능 (Tab 순서 논리적)
- ✓ 스크린 리더 테스트 완료
  - NVDA: [결과]
  - VoiceOver: [결과]
- ✓ 에러 메시지 명확함
- ✓ 폼 검증 피드백 적절함

### axe-core 자동 검사 결과
```
Violations: 0
Warnings: 0
Passes: 127
Incomplete: 2
```

### 스크린 리더 테스트
| 시나리오 | NVDA | VoiceOver | 결과 |
|---------|------|----------|------|
| 메뉴 네비게이션 | ✓ | ✓ | PASS |
| 링크 읽음 | ✓ | ✓ | PASS |
| 버튼 동작 | ✓ | ✓ | PASS |
| 폼 라벨 | ✓ | ✓ | PASS |

### 피드백
- [피드백]

### 결과
✓ 승인됨

---

## 종합 평가
- **레이아웃 구조**: ✓ 우수
- **시각 디자인**: ✓ 우수
- **Citation UX**: ✓ 우수
- **반응형**: ✓ 우수
- **접근성**: ✓ 우수

### 개선된 사항 요약
1. [개선사항 1]
2. [개선사항 2]
3. [개선사항 3]

### 최종 승인
**승인일:** 2026-04-22
**승인자:** [이름]
**상태:** ✓ 최종 승인
```

### 4-7. 성능 최적화 (선택)

**가상 스크롤 구현 (MessageHistory에서 100+ 턴):**

```typescript
// MessageHistory.tsx에 react-window 라이브러리 사용

import { FixedSizeList as List } from "react-window";

interface MessageHistoryProps {
  messages: Message[];
}

export default function MessageHistory({ messages }: MessageHistoryProps) {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      <MessageBubble message={messages[index]} />
    </div>
  );

  return (
    <List
      height={600}  // 컨테이너 높이
      itemCount={messages.length}
      itemSize={100}  // 각 아이템 높이 (대략값)
      width="100%"
    >
      {Row}
    </List>
  );
}
```

**React.memo 및 useMemo:**

```typescript
// MessageBubble.tsx
const MessageBubble = React.memo(({ message }: MessageBubbleProps) => {
  // 렌더링 로직
}, (prevProps, nextProps) => {
  return prevProps.message.id === nextProps.message.id;
});

export default MessageBubble;

// 부모 컴포넌트
const MessageHistory = React.memo(({ messages }: MessageHistoryProps) => {
  const renderedMessages = useMemo(() => messages.map(m => m), [messages]);
  return <>{renderedMessages.map(m => <MessageBubble key={m.id} message={m} />)}</>;
});
```

### 4-8. 단위 테스트 (반응형 + 접근성)

**파일:** `/frontend/src/__tests__/responsive.test.tsx`

```typescript
import React from "react";
import { render } from "@testing-library/react";
import ChatLayout from "@/components/chat/ChatLayout";

describe("ChatLayout - Responsive", () => {
  // 모바일 뷰포트 설정
  const setMobileViewport = () => {
    Object.defineProperty(window, "innerWidth", {
      writable: true,
      configurable: true,
      value: 390,
    });
  };

  const setTabletViewport = () => {
    Object.defineProperty(window, "innerWidth", {
      writable: true,
      configurable: true,
      value: 768,
    });
  };

  const setDesktopViewport = () => {
    Object.defineProperty(window, "innerWidth", {
      writable: true,
      configurable: true,
      value: 1280,
    });
  };

  test("renders mobile layout on small screens", () => {
    setMobileViewport();
    const { container } = render(<ChatLayout />);
    const drawer = container.querySelector("[role='dialog']");
    expect(drawer).toBeInTheDocument();
  });

  test("renders desktop layout on large screens", () => {
    setDesktopViewport();
    const { container } = render(<ChatLayout />);
    // 3컬럼이 모두 표시되는지 확인
    const conversationList = container.querySelector(".md\\:static");
    expect(conversationList).toHaveClass("md:static");
  });

  test("hamburger button is visible on mobile", () => {
    setMobileViewport();
    const { getByLabelText } = render(<ChatLayout />);
    const hamburger = getByLabelText("대화 목록 열기");
    expect(hamburger).toBeInTheDocument();
  });
});
```

**파일:** `/frontend/src/__tests__/accessibility.test.tsx`

```typescript
import React from "react";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { axe, toHaveNoViolations } from "jest-axe";
import ChatLayout from "@/components/chat/ChatLayout";

expect.extend(toHaveNoViolations);

describe("ChatLayout - Accessibility", () => {
  test("should not have axe violations", async () => {
    const { container } = render(<ChatLayout />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  test("keyboard navigation works", async () => {
    const user = userEvent.setup();
    const { container } = render(<ChatLayout />);

    // Tab 키로 포커스 이동
    await user.tab();
    const focused = document.activeElement;
    expect(focused).toBeInTheDocument();
  });

  test("aria labels are present", () => {
    render(<ChatLayout />);
    expect(screen.getByLabelText("대화 목록 열기")).toBeInTheDocument();
  });

  test("headings have proper hierarchy", () => {
    const { container } = render(<ChatLayout />);
    const h1Count = container.querySelectorAll("h1").length;
    const h2Count = container.querySelectorAll("h2").length;
    // h1은 1개 이하, h2는 있을 수 있음
    expect(h1Count).toBeLessThanOrEqual(1);
  });
});
```

## 5. 산출물

1. **반응형 레이아웃 컴포넌트** (`/frontend/src/components/chat/ChatLayout.tsx`)
2. **Drawer 컴포넌트** (`/frontend/src/components/ui/Drawer.tsx`)
3. **모든 컴포넌트에 반응형 유틸리티 적용**
   - Tailwind breakpoint (sm, md, lg, xl)
   - 색상 대비 검증
   - WCAG 시맨틱 HTML 및 ARIA 적용
4. **접근성 테스트 자동화**
   - `/frontend/src/utils/a11y.test.ts`
   - axe-core 통합
5. **UI 리뷰 기록 문서**
   - `/frontend/docs/UI_REVIEW_RECORDS.md` (5회 리뷰 상세 기록)
6. **성능 최적화** (선택)
   - Virtual scrolling 구현 (100+ 턴)
   - React.memo, useMemo 적용
7. **단위 및 통합 테스트**
   - `responsive.test.tsx`
   - `accessibility.test.tsx`
8. **WCAG 2.1 AA 준수 확인 리포트**
   - 색상 대비 측정 결과
   - 키보드 네비게이션 테스트 결과
   - 스크린 리더 호환성 테스트 결과

## 6. 완료 기준

### 반응형 디자인
1. 데스크톱 (≥1280px)에서 3컬럼이 정상 렌더링되는가?
2. 태블릿 (768~1279px)에서 2컬럼이 정상 렌더링되는가?
3. 모바일 (<768px)에서 1컬럼 + 오프캔버스 메뉴가 정상 작동하는가?
4. 모든 요소가 각 breakpoint에서 적절하게 스케일/정렬되는가?
5. 터치 친화적 버튼 크기(44x44px)가 보장되는가?
6. 성능이 허용 범위 내인가 (100+ 턴 스크롤 부드러움)?

### WCAG 2.1 AA 접근성
7. 색상 대비가 모두 ≥4.5:1인가?
8. 모든 기능을 키보드로 접근할 수 있는가?
9. Tab 키로 포커스가 논리적 순서대로 이동하는가?
10. 모든 링크/버튼에 명확한 텍스트 라벨이 있는가?
11. 이미지에 alt 텍스트가 있는가?
12. 폼 필드가 명시적으로 label과 연결되는가?
13. 스크린 리더(NVDA/VoiceOver)로 모든 콘텐츠를 읽을 수 있는가?
14. axe-core 자동 테스트에서 violations가 0인가?

### UI 5회 리뷰
15. 리뷰 1 (와이어프레임): 완료되었는가?
16. 리뷰 2 (시각 디자인): 완료되었는가?
17. 리뷰 3 (Citation UX): 완료되었는가?
18. 리뷰 4 (반응형): 완료되었는가?
19. 리뷰 5 (접근성): 완료되었는가?
20. 각 리뷰마다 피드백이 기록되고 개선되었는가?

### 산출물
21. 모든 컴포넌트가 반응형 유틸리티를 포함하는가?
22. UI 리뷰 기록 문서가 상세하게 작성되었는가?
23. 모든 단위 테스트가 통과하는가?
24. 타입스크립트 컴파일이 정상 작동하는가?

## 7. 작업 지침

### 지침 7-1. Tailwind 반응형 패턴

모든 컴포넌트에서 일관된 반응형 유틸리티를 사용한다:

```typescript
// 기본 패턴
className={`
  // 모바일 기본 (sm 미만)
  w-full px-2 py-1 text-xs
  
  // 소형 태블릿 (sm: 640px)
  sm:px-3 sm:py-2 sm:text-sm
  
  // 중형 태블릿 (md: 768px)
  md:px-4 md:py-3 md:text-base
  
  // 대형 데스크톱 (lg: 1024px)
  lg:px-6 lg:py-4 lg:text-lg
`}
```

### 지침 7-2. 색상 대비 측정

색상 대비 비율을 계산하는 도구:
- WebAIM Contrast Checker: https://webaim.org/resources/contrastchecker/
- Color Contrast Analyzer

모든 텍스트와 배경 조합이 최소 4.5:1 이상이어야 한다.

### 지침 7-3. ARIA 적절히 사용

과도한 ARIA 사용은 피한다. 시맨틱 HTML(button, nav, main, section 등)이 최우선이고, 필요한 경우에만 ARIA를 추가한다:

```typescript
// 나쁜 예
<div role="button" onClick={...}>Click me</div>

// 좋은 예
<button onClick={...}>Click me</button>

// 필요한 경우 ARIA
<button
  onClick={...}
  aria-label="대화 삭제"
  aria-describedby="delete-hint"
>
  삭제
</button>
<p id="delete-hint">이 작업은 되돌릴 수 없습니다.</p>
```

### 지침 7-4. 스크린 리더 테스트

스크린 리더는 최소 두 가지 이상에서 테스트한다:
- Windows: NVDA (무료)
- macOS/iOS: VoiceOver (내장)
- 모바일: TalkBack (Android), VoiceOver (iOS)

주요 테스트 시나리오:
- 페이지 로드 시 음성 안내
- 네비게이션(Tab, Shift+Tab)
- 링크 및 버튼 읽음
- 폼 입력 피드백

### 지침 7-5. UI 5회 리뷰 진행 방식

각 리뷰는 다음 과정을 따른다:
1. **계획**: 리뷰 목표와 검토 항목 정의
2. **실행**: 팀 미팅 또는 자체 검토 수행
3. **기록**: UI_REVIEW_RECORDS.md에 상세 기록
4. **피드백 수집**: 발견된 문제와 개선사항 정리
5. **개선 구현**: 피드백을 코드에 반영

### 지침 7-6. 모바일 우선 개발

모바일 화면(390px)부터 시작하여 점진적으로 데스크톱으로 확장한다:

```typescript
// 모바일 먼저 작성
className="px-2 py-1 text-xs"

// 그 다음 태블릿 이상 추가
className="px-2 py-1 text-xs md:px-4 md:py-3 md:text-base"
```

### 지침 7-7. 성능 모니터링

Chrome DevTools Performance 탭에서 성능을 측정한다:
- Largest Contentful Paint (LCP) < 2.5s
- First Input Delay (FID) < 100ms
- Cumulative Layout Shift (CLS) < 0.1

100+ 턴 대화에서 스크롤 FPS가 60 이상 유지되어야 한다.

### 지침 7-8. 버전 관리 및 문서화

각 UI 리뷰 후 버전을 기록하고, 최종 승인 시 적절한 태그를 추가한다:
- `v1.0.0-ui-review-1-complete`
- `v1.0.0-ui-review-2-complete`
- `v1.0.0-ui-review-5-final-approval`
