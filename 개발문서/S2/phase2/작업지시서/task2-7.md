# Task 2-7. QueryRewriter 및 ConversationCompressor

## 1. 작업 목적

멀티턴 대화에서 "이게 뭐야?" 같은 follow-up 질의를 **자립적인 단발 검색 쿼리로 재작성**하고, 길어진 대화 이력을 **압축(요약)**하여 컨텍스트 토큰 비용을 절감한다. Phase 1의 `LLMProvider`와 `PromptRegistry`를 활용하고, Prompt Registry에 쿼리 재작성 프롬프트 시드를 추가한다.

## 2. 작업 범위

### 포함 범위

1. **`QueryRewriter` 클래스**
   - `rewrite_query(original_query, conversation_history, mode)` → `str`
   - LLM 호출 기반 (Phase 1 LLMProvider)
   - Prompt Registry에서 템플릿 로드
   - 폐쇄망 폴백: LLM 호출 실패 시 원본 쿼리 그대로 반환

2. **`ConversationCompressor` 클래스**
   - `compress(messages, max_tokens)` → `str` (요약)
   - 전략 1: 슬라이딩 윈도우 (최근 N턴만 유지)
   - 전략 2: LLM 요약 (Phase 1 LLMProvider)

3. **Prompt Registry 시드 추가**
   - 쿼리 재작성 프롬프트 (`query_rewrite_standalone`)
   - 대화 요약 프롬프트 (`conversation_summary`)

4. **단위 테스트**
   - QueryRewriter 정상 재작성 (Mock LLM)
   - LLM 실패 시 원본 쿼리 폴백
   - ConversationCompressor 슬라이딩 윈도우 동작

### 제외 범위

- 멀티턴 RAG 엔드포인트 통합 (Task 2-8)
- Citation 캐시 (Task 2-8)
- 쿼리 재작성 품질 평가 (Task 2-8)

## 3. 선행 조건

- Phase 1 완료 (LLMProvider, PromptRegistry)
- Phase 1의 PromptRegistry 스키마 확인 (`/backend/app/services/prompt/`)
- Phase 1의 LLMProvider 인터페이스 확인 (`/backend/app/services/llm/`)

## 4. 주요 작업 항목

### 4-1. 대화 메시지 모델

**파일:** `/backend/app/schemas/conversation.py` (신규 또는 기존 확장)

```python
"""멀티턴 대화 도메인 모델 (Phase 3 Conversation의 최소 사전 정의).

Phase 3에서 확장될 예정이므로 최소 인터페이스만 정의한다.
"""
from __future__ import annotations

from enum import Enum
from typing import List, Optional
from uuid import UUID

from pydantic import BaseModel


class MessageRole(str, Enum):
    USER = "user"
    ASSISTANT = "assistant"
    SYSTEM = "system"


class ConversationMessage(BaseModel):
    """단일 대화 메시지."""
    role: MessageRole
    content: str
    turn_number: int = 0
```

### 4-2. QueryRewriter

**파일:** `/backend/app/services/retrieval/query_rewriter.py`

```python
"""QueryRewriter — LLM 기반 쿼리 재작성.

멀티턴 대화의 follow-up 질의를 자립적인 단발 쿼리로 재작성한다.
LLM 호출 실패 시 원본 쿼리를 그대로 반환하여 서비스 연속성을 보장한다.
"""
from __future__ import annotations

import logging
from typing import List, Optional

from app.schemas.conversation import ConversationMessage
from app.services.llm.base import LLMProvider

logger = logging.getLogger(__name__)

_REWRITE_PROMPT_KEY = "query_rewrite_standalone"

# 기본 프롬프트 (Prompt Registry에 시드로 등록)
_DEFAULT_REWRITE_PROMPT = """\
당신은 검색 질의 재작성 전문가입니다.

원본 질의: {original_query}
이전 대화 맥락:
{conversation_history}

위 맥락을 고려하여, 원본 질의를 더 자립적이고 명확한 단발 검색 쿼리로 다시 작성하세요.
검색 엔진이 이해할 수 있도록 주요 키워드를 포함하세요.
재작성된 쿼리만 출력하세요 (설명 없이).
"""


class QueryRewriter:
    """멀티턴 대화 컨텍스트 기반 쿼리 재작성.

    Attributes:
        _llm: LLMProvider 인스턴스
        _prompt_registry: PromptRegistry (선택, 없으면 기본 프롬프트 사용)
    """

    def __init__(
        self,
        llm: LLMProvider,
        prompt_registry=None,
    ) -> None:
        self._llm = llm
        self._prompt_registry = prompt_registry

    async def rewrite_query(
        self,
        original_query: str,
        conversation_history: List[ConversationMessage],
        mode: str = "standalone",
    ) -> str:
        """쿼리를 재작성한다.

        Args:
            original_query: 원본 쿼리
            conversation_history: 이전 턴의 메시지 리스트
            mode: "standalone" (기본) — 자립적 쿼리로 재작성

        Returns:
            재작성된 쿼리 문자열.
            LLM 호출 실패 또는 대화 이력 없을 때 → original_query 그대로 반환
        """
        # 첫 번째 턴이거나 대화 이력 없으면 재작성 불필요
        if not conversation_history:
            return original_query

        prompt_template = self._load_prompt_template()
        history_text = self._format_history(conversation_history)
        prompt = prompt_template.format(
            original_query=original_query,
            conversation_history=history_text,
        )

        try:
            response = await self._llm.generate(
                prompt=prompt,
                temperature=0.0,   # 결정론적 재작성
                max_tokens=200,
            )
            rewritten = response.content.strip()
            if not rewritten:
                return original_query
            logger.debug(
                "Query rewritten: %r → %r", original_query, rewritten
            )
            return rewritten
        except Exception as exc:
            logger.warning(
                "QueryRewriter LLM call failed (%s) — falling back to original query: %r",
                exc,
                original_query,
            )
            return original_query

    def _load_prompt_template(self) -> str:
        """Prompt Registry에서 템플릿 로드. 없으면 기본값 사용."""
        if self._prompt_registry is None:
            return _DEFAULT_REWRITE_PROMPT
        try:
            return self._prompt_registry.get(_REWRITE_PROMPT_KEY) or _DEFAULT_REWRITE_PROMPT
        except Exception:
            return _DEFAULT_REWRITE_PROMPT

    @staticmethod
    def _format_history(messages: List[ConversationMessage]) -> str:
        """대화 이력을 프롬프트용 텍스트로 포맷한다."""
        lines = []
        for msg in messages:
            prefix = "사용자" if msg.role.value == "user" else "어시스턴트"
            lines.append(f"{prefix}: {msg.content}")
        return "\n".join(lines)
```

### 4-3. ConversationCompressor

**파일:** `/backend/app/services/retrieval/conversation_compressor.py`

```python
"""ConversationCompressor — 대화 이력 압축.

두 가지 전략:
  1. 슬라이딩 윈도우: 최근 N턴만 유지 (빠름, LLM 불필요)
  2. LLM 요약: 전체 대화를 짧은 텍스트로 압축 (느림, 고품질)
"""
from __future__ import annotations

import logging
from typing import List, Optional

from app.schemas.conversation import ConversationMessage
from app.services.llm.base import LLMProvider

logger = logging.getLogger(__name__)

_SUMMARY_PROMPT_KEY = "conversation_summary"

_DEFAULT_SUMMARY_PROMPT = """\
다음 대화를 300자 이내로 핵심 내용만 요약하세요.
검색에 유용한 주요 키워드와 주제를 반드시 포함하세요.

대화:
{conversation_text}

요약:
"""


class ConversationCompressor:
    """대화 이력을 압축하여 컨텍스트 토큰 비용을 줄인다."""

    def __init__(
        self,
        llm: Optional[LLMProvider] = None,
        prompt_registry=None,
        window_size: int = 10,
    ) -> None:
        self._llm = llm
        self._prompt_registry = prompt_registry
        self._window_size = window_size

    async def compress(
        self,
        messages: List[ConversationMessage],
        max_tokens: int = 1000,
        strategy: str = "sliding_window",
    ) -> str:
        """대화 이력을 압축하여 문자열로 반환한다.

        Args:
            messages: 전체 대화 메시지 리스트
            max_tokens: 최대 허용 토큰 수 (LLM 요약 시 목표)
            strategy: "sliding_window" | "summarize"

        Returns:
            압축된 대화 컨텍스트 문자열
        """
        if not messages:
            return ""

        if strategy == "summarize" and self._llm is not None:
            return await self._llm_summarize(messages)
        return self._sliding_window(messages)

    def _sliding_window(self, messages: List[ConversationMessage]) -> str:
        """최근 window_size개 메시지만 유지한다."""
        recent = messages[-self._window_size:]
        lines = []
        for msg in recent:
            prefix = "사용자" if msg.role.value == "user" else "어시스턴트"
            lines.append(f"{prefix}: {msg.content}")
        return "\n".join(lines)

    async def _llm_summarize(self, messages: List[ConversationMessage]) -> str:
        """LLM으로 대화를 요약한다. 실패 시 슬라이딩 윈도우로 폴백."""
        conversation_text = self._sliding_window(messages)  # 전체 포맷
        template = self._load_summary_prompt()
        prompt = template.format(conversation_text=conversation_text)

        try:
            response = await self._llm.generate(
                prompt=prompt,
                temperature=0.0,
                max_tokens=400,
            )
            return response.content.strip()
        except Exception as exc:
            logger.warning(
                "ConversationCompressor LLM summarize failed (%s) — fallback to sliding window",
                exc,
            )
            return self._sliding_window(messages)

    def _load_summary_prompt(self) -> str:
        if self._prompt_registry is None:
            return _DEFAULT_SUMMARY_PROMPT
        try:
            return self._prompt_registry.get(_SUMMARY_PROMPT_KEY) or _DEFAULT_SUMMARY_PROMPT
        except Exception:
            return _DEFAULT_SUMMARY_PROMPT
```

### 4-4. Prompt Registry 시드

**파일:** `/backend/app/services/prompt/seeds/query_rewrite_standalone.json`

```json
{
  "key": "query_rewrite_standalone",
  "version": "1.0",
  "description": "멀티턴 대화 컨텍스트 기반 쿼리 재작성 프롬프트",
  "template": "당신은 검색 질의 재작성 전문가입니다.\n\n원본 질의: {original_query}\n이전 대화 맥락:\n{conversation_history}\n\n위 맥락을 고려하여, 원본 질의를 더 자립적이고 명확한 단발 검색 쿼리로 다시 작성하세요.\n검색 엔진이 이해할 수 있도록 주요 키워드를 포함하세요.\n재작성된 쿼리만 출력하세요 (설명 없이).",
  "variables": ["original_query", "conversation_history"],
  "tags": ["retrieval", "query_rewrite", "multiturn"]
}
```

**파일:** `/backend/app/services/prompt/seeds/conversation_summary.json`

```json
{
  "key": "conversation_summary",
  "version": "1.0",
  "description": "멀티턴 대화 요약 프롬프트",
  "template": "다음 대화를 300자 이내로 핵심 내용만 요약하세요.\n검색에 유용한 주요 키워드와 주제를 반드시 포함하세요.\n\n대화:\n{conversation_text}\n\n요약:",
  "variables": ["conversation_text"],
  "tags": ["retrieval", "compression", "multiturn"]
}
```

### 4-5. 단위 테스트

**파일:** `/backend/tests/unit/services/test_query_rewriter.py`

```python
"""QueryRewriter 단위 테스트."""
import pytest
from unittest.mock import AsyncMock, MagicMock
from app.services.retrieval.query_rewriter import QueryRewriter
from app.services.retrieval.conversation_compressor import ConversationCompressor
from app.schemas.conversation import ConversationMessage, MessageRole


def _make_msg(role: str, content: str) -> ConversationMessage:
    return ConversationMessage(role=MessageRole(role), content=content)


@pytest.mark.asyncio
async def test_rewrite_returns_original_when_no_history():
    """대화 이력 없으면 원본 쿼리 반환."""
    llm = MagicMock()
    rewriter = QueryRewriter(llm)
    result = await rewriter.rewrite_query("테스트 쿼리", [])
    assert result == "테스트 쿼리"
    llm.generate.assert_not_called()


@pytest.mark.asyncio
async def test_rewrite_calls_llm_with_history():
    """대화 이력 있으면 LLM 호출 후 재작성된 쿼리 반환."""
    llm = MagicMock()
    llm.generate = AsyncMock(return_value=MagicMock(content="Kubernetes 배포 전략은?"))
    history = [
        _make_msg("user", "Kubernetes에 대해 알려줘"),
        _make_msg("assistant", "Kubernetes는 컨테이너 오케스트레이션 플랫폼입니다."),
    ]
    rewriter = QueryRewriter(llm)
    result = await rewriter.rewrite_query("이게 뭐야?", history)
    assert result == "Kubernetes 배포 전략은?"


@pytest.mark.asyncio
async def test_rewrite_fallback_on_llm_error():
    """LLM 오류 시 원본 쿼리 반환."""
    llm = MagicMock()
    llm.generate = AsyncMock(side_effect=Exception("LLM unavailable"))
    history = [_make_msg("user", "이전 대화")]
    rewriter = QueryRewriter(llm)
    result = await rewriter.rewrite_query("원본 쿼리", history)
    assert result == "원본 쿼리"


@pytest.mark.asyncio
async def test_compressor_sliding_window():
    """슬라이딩 윈도우 — 최근 N개 메시지만 유지."""
    compressor = ConversationCompressor(window_size=2)
    messages = [_make_msg("user", f"메시지 {i}") for i in range(5)]
    result = await compressor.compress(messages, strategy="sliding_window")
    assert "메시지 4" in result
    assert "메시지 3" in result
    assert "메시지 0" not in result


@pytest.mark.asyncio
async def test_compressor_llm_summarize():
    """LLM 요약 전략."""
    llm = MagicMock()
    llm.generate = AsyncMock(return_value=MagicMock(content="요약된 내용"))
    compressor = ConversationCompressor(llm=llm, window_size=5)
    messages = [_make_msg("user", "내용")]
    result = await compressor.compress(messages, strategy="summarize")
    assert result == "요약된 내용"


@pytest.mark.asyncio
async def test_compressor_fallback_on_llm_error():
    """LLM 요약 실패 시 슬라이딩 윈도우로 폴백."""
    llm = MagicMock()
    llm.generate = AsyncMock(side_effect=Exception("LLM error"))
    compressor = ConversationCompressor(llm=llm, window_size=3)
    messages = [_make_msg("user", f"msg {i}") for i in range(5)]
    result = await compressor.compress(messages, strategy="summarize")
    # 폴백: 최근 3개 메시지 포함
    assert "msg 4" in result
```

## 5. 산출물

1. **ConversationMessage 스키마** (`/backend/app/schemas/conversation.py`)
2. **QueryRewriter** (`/backend/app/services/retrieval/query_rewriter.py`)
3. **ConversationCompressor** (`/backend/app/services/retrieval/conversation_compressor.py`)
4. **Prompt 시드 파일** (`query_rewrite_standalone.json`, `conversation_summary.json`)
5. **단위 테스트** (`/backend/tests/unit/services/test_query_rewriter.py`)

## 6. 완료 기준

1. `QueryRewriter`: 대화 이력 없을 때 원본 쿼리 반환 (LLM 미호출)
2. `QueryRewriter`: LLM 호출 성공 시 재작성된 쿼리 반환
3. `QueryRewriter`: LLM 실패 시 원본 쿼리 폴백 (예외 미발생)
4. `ConversationCompressor`: 슬라이딩 윈도우 정상 작동
5. `ConversationCompressor`: LLM 요약 실패 시 슬라이딩 윈도우 폴백
6. Prompt Registry에 두 시드 파일 등록 확인
7. 단위 테스트 전체 통과

## 7. 작업 지침

### 지침 7-1. 폴백 우선 설계

`QueryRewriter`와 `ConversationCompressor` 모두 LLM 호출 실패 시 **서비스 중단 없이** 원본/슬라이딩 윈도우 결과를 반환한다. S2 원칙: LLM off 시 서비스 degrade하지만 실패하지 않음.

### 지침 7-2. 재작성 투명성

`QueryRewriter.rewrite_query()` 반환값은 **재작성된 쿼리 문자열만** 반환한다. 호출자(Task 2-8의 RAG 엔드포인트)가 `rewritten_query` 필드로 응답에 포함시켜 클라이언트에 투명하게 공개한다.

### 지침 7-3. Prompt Registry 통합

Phase 1에서 구현된 PromptRegistry의 `get(key)` 인터페이스를 사용한다. PromptRegistry가 없거나 키 조회 실패 시 내부 기본 프롬프트를 사용하여 외부 의존성을 줄인다.

### 지침 7-4. ConversationMessage 최소 정의

`ConversationMessage` 스키마는 Phase 3 Conversation 도메인에서 확장될 예정이다. 이 Task에서는 최소 필드(`role`, `content`, `turn_number`)만 정의하고 Phase 3와 충돌하지 않도록 설계한다.
