# embedease-sdk 易用性优化规划

> 版本：v0.2.0 目标  
> 当前版本：v0.1.17  
> 编写日期：2025-07  

---

## 目录

1. [概述](#1-概述)
2. [改造总览](#2-改造总览)
3. [第一阶段：前端 SDK 改造](#3-第一阶段前端-sdk-改造)
4. [第二阶段：后端 SDK 改造](#4-第二阶段后端-sdk-改造)
5. [第三阶段：SDK 版本发布与源码同步](#5-第三阶段sdk-版本发布与源码同步)
6. [第四阶段：embedease-ai 项目改造](#6-第四阶段embedease-ai-项目改造)
7. [第五阶段：Skill-Know 项目改造](#7-第五阶段skill-know-项目改造)
8. [第六阶段：mobile-mcp 项目改造](#8-第六阶段mobile-mcp-项目改造)
9. [风险与注意事项](#9-风险与注意事项)
10. [验收标准](#10-验收标准)

---

## 1. 概述

### 1.1 当前问题

经过对三个项目（embedease-ai、Skill-Know、mobile-mcp）使用 SDK 的深度分析，发现以下核心痛点：

| # | 痛点 | 影响 |
|---|------|------|
| 1 | **前端 `useChat` hook 没人用** | 每个项目都手写 80-130 行 zustand store 代码 |
| 2 | **Timeline reducer 不可扩展** | Skill-Know 要写 `businessReducer` 包装，mobile-mcp 要手动 patch state |
| 3 | **后端 Orchestrator 模式高度重复** | 两个项目各写 ~200 行几乎相同的编排逻辑 |
| 4 | **类型系统封闭** | `TimelineItem` 是固定联合类型，扩展需要 `as unknown` 强转 |
| 5 | **SDK 副本可能分叉** | 各项目 packages/ 目录下的 SDK 源码版本可能不一致 |

### 1.2 改造目标

- **新项目接入成本**：从 300+ 行降到 30-50 行
- **自定义事件支持**：无需 hack，通过正式 API 扩展
- **后端编排**：一个钩子化的 Orchestrator，消除重复的 queue 消费 + 落库逻辑
- **版本一致性**：统一版本号，源码拷贝时有明确的版本标识

### 1.3 改造原则

- **向后兼容**：所有现有 API 保留，新功能以新导出提供
- **渐进式**：每个项目可以逐步迁移，不需要一次性改完
- **源码拷贝**：保持 `file:./packages/chat-sdk` 的引用方式，但确保版本号可追踪

---

## 2. 改造总览

```
阶段 1: 前端 SDK 改造 (chat-sdk + chat-sdk-react)
  ├── 1.1 类型系统泛型化
  ├── 1.2 Reducer 插件机制
  ├── 1.3 Zustand Store Factory
  └── 1.4 事件中间件

阶段 2: 后端 SDK 改造 (langgraph-agent-kit)
  ├── 2.1 Orchestrator Hook 系统
  ├── 2.2 AgentRunner 协议
  └── 2.3 内置 ContentAggregator

阶段 3: SDK 版本发布与源码同步
  ├── 3.1 版本号升级到 0.2.0
  ├── 3.2 更新 CHANGELOG
  └── 3.3 源码拷贝到各项目

阶段 4-6: 各项目改造
  ├── embedease-ai (前端 + 后端)
  ├── Skill-Know (前端 + 后端)
  └── mobile-mcp (仅前端)
```

---

## 3. 第一阶段：前端 SDK 改造

### 3.1 类型系统泛型化

**目标**：让 `TimelineState`、`insertItem`、`updateItemById` 等核心函数支持自定义 `TimelineItem` 类型，消除 `as unknown` 强转。

**涉及文件**：
- `frontend/chat-sdk/src/timeline/types.ts`
- `frontend/chat-sdk/src/timeline/helpers.ts`
- `frontend/chat-sdk/src/timeline/reducer.ts`
- `frontend/chat-sdk/src/timeline/actions.ts`

#### 3.1.1 修改 `types.ts`

新增基础 Item 接口，让 `TimelineState` 支持泛型：

```typescript
// 新增：所有 TimelineItem 的基础接口
export interface TimelineItemBase {
  type: string;
  id: string;
  turnId: string;
  ts: number;
}

// 现有类型保持不变，但都实现 TimelineItemBase
// UserMessageItem, LLMCallClusterItem, ToolCallItem, ... 无需修改（已满足）

// TimelineItem 保持不变（向后兼容）
export type TimelineItem = 
  | UserMessageItem
  | LLMCallClusterItem
  | ... ;

// 新增：泛型 TimelineState
export interface TimelineState<T extends TimelineItemBase = TimelineItem> {
  timeline: T[];
  indexById: Record<string, number>;
  activeTurn: {
    turnId: string | null;
    currentLlmCallId: string | null;
    currentToolCallId: string | null;
    isStreaming: boolean;
  };
}
```

> **向后兼容**：默认泛型参数 `= TimelineItem`，现有代码无需修改。

#### 3.1.2 修改 `helpers.ts`

让 `insertItem` 和 `updateItemById` 支持泛型：

```typescript
// 改为泛型
export function insertItem<T extends TimelineItemBase = TimelineItem>(
  state: TimelineState<T>,
  item: T
): TimelineState<T> {
  const timeline = [...state.timeline, item];
  const indexById = { ...state.indexById, [item.id]: timeline.length - 1 };
  return { ...state, timeline, indexById };
}

export function updateItemById<T extends TimelineItemBase = TimelineItem>(
  state: TimelineState<T>,
  id: string,
  updater: (item: T) => T
): TimelineState<T> {
  const index = state.indexById[id];
  if (index === undefined) return state;
  const timeline = [...state.timeline];
  timeline[index] = updater(timeline[index]);
  return { ...state, timeline };
}

export function removeWaitingItem<T extends TimelineItemBase = TimelineItem>(
  state: TimelineState<T>,
  turnId: string
): TimelineState<T> {
  // 逻辑不变，只是类型泛型化
  ...
}

// getCurrentLlmCluster, appendSubItemToCurrentCluster 等
// 这些函数内部依赖具体类型 (LLMCallClusterItem)，保持不变
// 它们只在 SDK 内置 reducer 中使用
```

#### 3.1.3 修改 `actions.ts`

action 函数改为泛型（仅在类型签名层面）：

```typescript
export function addUserMessage<T extends TimelineItemBase = TimelineItem>(
  state: TimelineState<T>,
  id: string,
  content: string,
  images?: ImageAttachment[]
): TimelineState<T> {
  const item: UserMessageItem = { ... };
  return insertItem(state, item as T);  // UserMessageItem 是 T 的子类型
}

// startAssistantTurn, clearTurn, endTurn 同理
```

#### 3.1.4 导出新增类型

在 `frontend/chat-sdk/src/timeline/index.ts` 中新增导出：

```typescript
export type { TimelineItemBase } from "./types";
```

在 `frontend/chat-sdk/src/index.ts` 中也确保导出。

---

### 3.2 Reducer 插件机制

**目标**：让 `timelineReducer` 支持链式扩展，未识别的事件自动传递给用户注册的 reducer。

**涉及文件**：
- `frontend/chat-sdk/src/timeline/reducer.ts`（修改）
- `frontend/chat-sdk/src/timeline/compose-reducer.ts`（新建）
- `frontend/chat-sdk/src/timeline/index.ts`（更新导出）

#### 3.2.1 新建 `compose-reducer.ts`

```typescript
import type { ChatEvent } from "../core/events";
import type { TimelineItemBase, TimelineState } from "./types";
import { timelineReducer as builtinReducer } from "./reducer";

/**
 * 自定义 Reducer 类型
 * 
 * - 返回 TimelineState 表示已处理
 * - 返回 null/undefined 表示未处理，交给下一个 reducer
 */
export type CustomReducer<T extends TimelineItemBase> = (
  state: TimelineState<T>,
  event: ChatEvent | Record<string, unknown>
) => TimelineState<T> | null | undefined;

/**
 * 组合 reducer：先走用户 reducer，未处理的再走 SDK 内置 reducer
 * 
 * 用法：
 * ```typescript
 * const myReducer = composeReducers(
 *   businessReducer,    // 处理 intent.extracted, search.results.found 等
 *   screenshotReducer,  // 处理 tool.end 后的 screenshot_id 注入
 * );
 * ```
 */
export function composeReducers<T extends TimelineItemBase>(
  ...customReducers: CustomReducer<T>[]
): (state: TimelineState<T>, event: ChatEvent | Record<string, unknown>) => TimelineState<T> {
  return (state, event) => {
    // 先尝试用户 reducer（按顺序）
    for (const reducer of customReducers) {
      const result = reducer(state, event);
      if (result !== null && result !== undefined) {
        return result;
      }
    }
    // 未被任何用户 reducer 处理，走内置 reducer
    return builtinReducer(state as TimelineState, event as ChatEvent) as TimelineState<T>;
  };
}
```

#### 3.2.2 更新导出

在 `frontend/chat-sdk/src/timeline/index.ts` 新增：

```typescript
export { composeReducers } from "./compose-reducer";
export type { CustomReducer } from "./compose-reducer";
```

#### 3.2.3 Skill-Know 的改造效果预览

改造前（`sdk-extensions/reducer.ts`，110 行）：
```typescript
// 需要 import SDK 内部函数，手动判断 default case
export function businessReducer(state, event) {
  switch (event.type) {
    case "intent.extracted": { ... return insertItem(state, item as unknown as ...) }
    ...
    default: return sdkReducer(state, event as ChatEvent);
  }
}
```

改造后（约 30 行）：
```typescript
import { composeReducers, insertItem, type CustomReducer } from "@embedease/chat-sdk";
import type { BusinessTimelineItem } from "./types";

const businessReducer: CustomReducer<BusinessTimelineItem> = (state, event) => {
  const evt = event as Record<string, unknown>;
  switch (evt.type as string) {
    case "intent.extracted": 
      return insertItem(state, { type: "intent.extracted", ... } as BusinessTimelineItem);
    case "search.results.found":
      return insertItem(state, { ... } as BusinessTimelineItem);
    ...
    default: return null; // 未处理，交给内置 reducer
  }
};

export const timelineReducer = composeReducers<BusinessTimelineItem>(businessReducer);
```

---

### 3.3 Zustand Store Factory

**目标**：提供开箱即用的 zustand store 创建函数，消除各项目手写 80-130 行 store 代码。

**涉及文件**：
- `frontend/chat-sdk-react/src/create-chat-store.ts`（新建）
- `frontend/chat-sdk-react/src/index.ts`（更新导出）

#### 3.3.1 新建 `create-chat-store.ts`

> 注意：`zustand` 作为 `peerDependency`，不直接依赖。

```typescript
/**
 * Zustand Chat Store Factory
 * 
 * 提供开箱即用的聊天状态管理，消除各项目重复的 store 代码。
 * 
 * 最简用法：
 * ```typescript
 * import { createChatStore } from "@embedease/chat-sdk-react";
 * 
 * export const useChatStore = createChatStore({ baseUrl: "/api" });
 * ```
 * 
 * 自定义事件：
 * ```typescript
 * import { createChatStore, composeReducers } from "@embedease/chat-sdk-react";
 * 
 * export const useChatStore = createChatStore({
 *   baseUrl: "/api",
 *   reducer: composeReducers(myBusinessReducer),
 *   onEvent: (event, api) => { ... },
 *   onStreamEnd: (result, api) => { ... },
 * });
 * ```
 */

import type { ChatEvent } from "@embedease/chat-sdk";
import type { TimelineItemBase, TimelineState } from "@embedease/chat-sdk";
import {
  ChatClient,
  createInitialState,
  addUserMessage,
  startAssistantTurn,
  timelineReducer as defaultReducer,
  clearTurn,
  endTurn,
  historyToTimeline,
  type ChatRequest,
  type HistoryMessage,
} from "@embedease/chat-sdk";

// ==================== 类型定义 ====================

export interface ChatStoreApi<T extends TimelineItemBase = TimelineItemBase> {
  getState: () => ChatStoreState<T>;
  setState: (partial: Partial<ChatStoreState<T>>) => void;
}

export interface StreamEndResult {
  conversationId: string;
  assistantMessageId: string;
  fullContent: string;
}

export interface CreateChatStoreOptions<T extends TimelineItemBase = TimelineItemBase> {
  /** API 基础 URL */
  baseUrl: string | (() => string);
  /** 自定义请求头 */
  headers?: Record<string, string> | (() => Record<string, string>);
  /** 自定义 reducer（默认使用 SDK 内置 reducer） */
  reducer?: (state: TimelineState<T>, event: ChatEvent | Record<string, unknown>) => TimelineState<T>;
  /** 
   * 事件中间件：每个 SSE 事件到达时调用
   * 可用于：提取 conversation_id、注入自定义字段等
   * 返回 event 继续处理，返回 null 跳过此事件
   */
  onEvent?: (
    event: ChatEvent,
    api: ChatStoreApi<T>
  ) => ChatEvent | null;
  /**
   * 流结束回调
   * 可用于：落库、通知等
   */
  onStreamEnd?: (result: StreamEndResult, api: ChatStoreApi<T>) => void | Promise<void>;
  /**
   * 构建 ChatRequest 的钩子
   * 默认行为：{ user_id, conversation_id, message }
   * 可用于：添加 images、自定义字段等
   */
  buildRequest?: (params: {
    message: string;
    conversationId: string;
    userId: string;
  }) => ChatRequest;
  /** 初始 user_id */
  userId?: string;
}

export interface ChatStoreState<T extends TimelineItemBase = TimelineItemBase> {
  // 状态
  timelineState: TimelineState<T>;
  conversationId: string;
  isStreaming: boolean;
  error: string | null;
  
  // 操作
  sendMessage: (message: string) => Promise<void>;
  abortStream: () => void;
  clearMessages: () => void;
  setConversationId: (id: string) => void;
  initFromHistory: (messages: HistoryMessage[]) => void;
}

// ==================== Store Factory ====================

/**
 * 创建聊天 Store 的状态和操作定义
 * 
 * 这个函数返回一个 zustand store creator 函数，
 * 兼容 zustand 的 create() API。
 * 
 * @example
 * ```typescript
 * import { create } from "zustand";
 * import { createChatStoreSlice } from "@embedease/chat-sdk-react";
 * 
 * export const useChatStore = create(
 *   createChatStoreSlice({ baseUrl: "/api" })
 * );
 * ```
 */
export function createChatStoreSlice<T extends TimelineItemBase = TimelineItemBase>(
  options: CreateChatStoreOptions<T>
) {
  return (set: (partial: Partial<ChatStoreState<T>> | ((state: ChatStoreState<T>) => Partial<ChatStoreState<T>>)) => void, get: () => ChatStoreState<T>): ChatStoreState<T> => {
    const client = new ChatClient({
      baseUrl: typeof options.baseUrl === "function" ? options.baseUrl() : options.baseUrl,
      headers: typeof options.headers === "function" ? options.headers() : options.headers,
    });

    const reducer = options.reducer ?? ((state: TimelineState<T>, event: ChatEvent | Record<string, unknown>) => {
      return defaultReducer(state as TimelineState, event as ChatEvent) as TimelineState<T>;
    });

    const api: ChatStoreApi<T> = { getState: get, setState: (partial) => set(partial) };

    return {
      timelineState: createInitialState() as TimelineState<T>,
      conversationId: "",
      isStreaming: false,
      error: null,

      sendMessage: async (message: string) => {
        const state = get();
        if (state.isStreaming) return;

        const userId = options.userId || "default_user";
        const conversationId = state.conversationId || crypto.randomUUID();
        const userMessageId = crypto.randomUUID();
        const assistantTurnId = crypto.randomUUID();

        // 1. 添加用户消息 + 开始助手 turn
        let timelineState = state.timelineState;
        timelineState = addUserMessage(timelineState as TimelineState, userMessageId, message) as TimelineState<T>;
        timelineState = startAssistantTurn(timelineState as TimelineState, assistantTurnId) as TimelineState<T>;

        set({
          timelineState,
          conversationId,
          isStreaming: true,
          error: null,
        });

        // 2. 构建请求
        const request = options.buildRequest
          ? options.buildRequest({ message, conversationId, userId })
          : { user_id: userId, conversation_id: conversationId, message };

        // 3. 流式处理
        let fullContent = "";
        let finalConversationId = conversationId;
        let assistantMessageId = assistantTurnId;

        try {
          for await (const event of client.stream(request)) {
            // 从 meta.start 提取真实 ID
            if (event.type === "meta.start") {
              const payload = event.payload as Record<string, unknown>;
              if (payload.assistant_message_id) {
                assistantMessageId = payload.assistant_message_id as string;
              }
              if (event.conversation_id) {
                finalConversationId = event.conversation_id;
              }
            }

            // 累积内容
            if (event.type === "assistant.delta") {
              const payload = event.payload as { delta?: string };
              fullContent += payload.delta || "";
            } else if (event.type === "assistant.final") {
              const payload = event.payload as { content?: string };
              fullContent = payload.content || fullContent;
            }

            // 事件中间件
            let processedEvent: ChatEvent | null = event;
            if (options.onEvent) {
              processedEvent = options.onEvent(event, api);
            }

            // reducer 更新状态
            if (processedEvent) {
              set((s) => ({
                timelineState: reducer(s.timelineState, processedEvent!),
                conversationId: finalConversationId,
              }));
            }
          }

          // 4. 结束 turn
          set((s) => ({
            timelineState: endTurn(s.timelineState as TimelineState) as TimelineState<T>,
            isStreaming: false,
          }));

          // 5. 流结束回调
          if (options.onStreamEnd) {
            await options.onStreamEnd(
              { conversationId: finalConversationId, assistantMessageId, fullContent },
              api
            );
          }
        } catch (error) {
          const errorMessage = error instanceof Error ? error.message : String(error);
          if (error instanceof Error && error.name === "AbortError") {
            set((s) => ({
              timelineState: endTurn(s.timelineState as TimelineState) as TimelineState<T>,
              isStreaming: false,
            }));
            return;
          }
          set({
            error: errorMessage,
            isStreaming: false,
          });
        }
      },

      abortStream: () => {
        client.abort();
        set((s) => ({
          timelineState: endTurn(s.timelineState as TimelineState) as TimelineState<T>,
          isStreaming: false,
        }));
      },

      clearMessages: () => {
        set({
          timelineState: createInitialState() as TimelineState<T>,
          error: null,
        });
      },

      setConversationId: (id: string) => {
        set({ conversationId: id });
      },

      initFromHistory: (messages: HistoryMessage[]) => {
        set({
          timelineState: historyToTimeline(messages) as TimelineState<T>,
        });
      },
    };
  };
}
```

#### 3.3.2 更新 `chat-sdk-react` 的 `package.json`

新增 `zustand` 为可选 peerDependency：

```json
{
  "peerDependencies": {
    "react": ">=18.0.0",
    "react-dom": ">=18.0.0",
    "zustand": ">=4.0.0"
  },
  "peerDependenciesMeta": {
    "zustand": {
      "optional": true
    }
  }
}
```

#### 3.3.3 更新导出

在 `frontend/chat-sdk-react/src/index.ts` 新增：

```typescript
export { createChatStoreSlice } from "./create-chat-store";
export type { 
  CreateChatStoreOptions,
  ChatStoreState,
  ChatStoreApi,
  StreamEndResult 
} from "./create-chat-store";
```

同时从 `@embedease/chat-sdk` 重新导出 `composeReducers`：

```typescript
export { composeReducers, type CustomReducer } from "@embedease/chat-sdk";
```

#### 3.3.4 各项目改造效果预览

**mobile-mcp 改造后**（从 78 行 → ~25 行）：

```typescript
import { create } from "zustand";
import { createChatStoreSlice } from "@embedease/chat-sdk-react";

export const useAgentStore = create(
  createChatStoreSlice({
    baseUrl: () => getApiRoot(),
    onEvent: (event, api) => {
      // 注入 screenshot_id
      if (event.type === "tool.end") {
        const payload = event.payload as { tool_call_id?: string; screenshot_id?: string };
        if (payload.screenshot_id && payload.tool_call_id) {
          const state = api.getState();
          // ... 注入逻辑
        }
      }
      return event;
    },
  })
);
```

**Skill-Know 改造后**（从 130 行 → ~20 行）：

```typescript
import { create } from "zustand";
import { createChatStoreSlice, composeReducers } from "@embedease/chat-sdk-react";
import { businessReducer, type BusinessTimelineItem } from "@/lib/sdk-extensions";

export const useChatStore = create(
  createChatStoreSlice<BusinessTimelineItem>({
    baseUrl: () => getApiUrl(),
    reducer: composeReducers<BusinessTimelineItem>(businessReducer),
    onStreamEnd: async ({ conversationId }) => {
      // 业务回调
    },
  })
);
```

---

### 3.4 事件中间件（已集成在 Store Factory 中）

通过 `onEvent` 回调实现，不需要单独的中间件系统。`onEvent` 的能力：

- **提取信息**：从 `meta.start` 中读取后端分配的 `conversation_id`（已内置）
- **过滤事件**：返回 `null` 跳过不想处理的事件
- **修改事件**：修改 event 后返回（如注入 `screenshot_id`）
- **副作用**：在事件到达时执行业务逻辑

---

## 4. 第二阶段：后端 SDK 改造

### 4.1 Orchestrator Hook 系统

**目标**：提供一个钩子化的编排器，消除各项目重复的 queue 消费 + 落库 + 内容聚合逻辑。

**涉及文件**：
- `backend/langgraph-agent-kit/src/langgraph_agent_kit/orchestrator.py`（新建）
- `backend/langgraph-agent-kit/src/langgraph_agent_kit/__init__.py`（更新导出）

#### 4.1.1 新建 `orchestrator.py`

```python
"""Orchestrator - 可组合的聊天流编排器

使用钩子系统消除各项目重复的编排代码。

最简用法：
    orchestrator = Orchestrator(
        agent_runner=my_runner,
    )
    async for event in orchestrator.run(message="你好", ...):
        yield event

完整用法：
    orchestrator = Orchestrator(
        agent_runner=my_runner,
        hooks=OrchestratorHooks(
            on_stream_start=save_user_message,
            on_event=custom_event_handler,
            on_stream_end=save_assistant_message,
        ),
    )
"""

from __future__ import annotations

import asyncio
import uuid
from collections.abc import AsyncGenerator
from dataclasses import dataclass, field
from typing import Any, Protocol, runtime_checkable

from langgraph_agent_kit.core.events import StreamEventType
from langgraph_agent_kit.core.stream_event import StreamEvent
from langgraph_agent_kit.core.context import ChatContext
from langgraph_agent_kit.core.emitter import QueueDomainEmitter
from langgraph_agent_kit.streaming.sse import make_event


# ==================== AgentRunner 协议 ====================

@runtime_checkable
class AgentRunner(Protocol):
    """Agent 运行器协议
    
    用户需要实现此协议来对接自己的 Agent。
    只需实现一个方法：通过 context.emitter 发送事件。
    
    示例实现：
        class MyAgentRunner:
            async def run(self, message, context, **kwargs):
                agent = build_my_agent(context.emitter)
                async for chunk in agent.astream({"messages": [...]}):
                    await handler.handle(chunk)
                await context.emitter.aemit("__end__", None)
    """
    async def run(
        self,
        message: str,
        context: ChatContext,
        **kwargs: Any,
    ) -> None:
        """运行 Agent
        
        通过 context.emitter 发送领域事件。
        运行结束时 必须 调用 `await context.emitter.aemit("__end__", None)`。
        """
        ...


# ==================== ContentAggregator ====================

@dataclass
class ContentAggregator:
    """内容聚合器 - 自动追踪流中的内容"""
    full_content: str = ""
    reasoning: str = ""
    products: Any | None = None
    tool_calls: dict[str, dict[str, Any]] = field(default_factory=dict)
    _tool_call_start_times: dict[str, float] = field(default_factory=dict)

    def process_event(self, evt_type: str, payload: dict[str, Any]) -> None:
        """处理单个事件，更新聚合状态"""
        if evt_type == StreamEventType.ASSISTANT_DELTA.value:
            delta = payload.get("delta", "")
            if delta:
                self.full_content += delta

        elif evt_type == StreamEventType.ASSISTANT_REASONING_DELTA.value:
            delta = payload.get("delta", "")
            if delta:
                self.reasoning += delta

        elif evt_type == StreamEventType.ASSISTANT_PRODUCTS.value:
            self.products = payload.get("items")

        elif evt_type == StreamEventType.TOOL_START.value:
            tc_id = payload.get("tool_call_id")
            if tc_id:
                self.tool_calls[tc_id] = {
                    "tool_call_id": tc_id,
                    "name": payload.get("name", "unknown"),
                    "input": payload.get("input", {}),
                    "status": "pending",
                }
                self._tool_call_start_times[tc_id] = asyncio.get_event_loop().time()

        elif evt_type == StreamEventType.TOOL_END.value:
            tc_id = payload.get("tool_call_id")
            if tc_id and tc_id in self.tool_calls:
                self.tool_calls[tc_id]["status"] = payload.get("status") or "success"
                self.tool_calls[tc_id]["output"] = payload.get("output_preview")
                if payload.get("error"):
                    self.tool_calls[tc_id]["error_message"] = payload["error"]
                start = self._tool_call_start_times.get(tc_id)
                if start:
                    self.tool_calls[tc_id]["duration_ms"] = int(
                        (asyncio.get_event_loop().time() - start) * 1000
                    )

        elif evt_type == StreamEventType.ASSISTANT_FINAL.value:
            self.full_content = payload.get("content") or self.full_content
            self.reasoning = payload.get("reasoning") or self.reasoning
            self.products = payload.get("products") or self.products

    @property
    def tool_calls_list(self) -> list[dict[str, Any]]:
        return list(self.tool_calls.values()) if self.tool_calls else []


# ==================== Hooks ====================

@dataclass
class StreamStartInfo:
    """流开始时的信息"""
    conversation_id: str
    user_id: str
    user_message_id: str
    assistant_message_id: str
    message: str

@dataclass
class StreamEndInfo:
    """流结束时的信息"""
    conversation_id: str
    user_id: str
    assistant_message_id: str
    aggregator: ContentAggregator
    context: ChatContext

@dataclass
class OrchestratorHooks:
    """编排器钩子
    
    所有钩子都是可选的。
    """
    on_stream_start: Any | None = None   # async (info: StreamStartInfo) -> None
    on_event: Any | None = None          # async (evt_type: str, payload: dict, aggregator: ContentAggregator) -> None
    on_stream_end: Any | None = None     # async (info: StreamEndInfo) -> None
    on_error: Any | None = None          # async (error: Exception, conversation_id: str) -> None


# ==================== Orchestrator ====================

class Orchestrator:
    """可组合的聊天流编排器
    
    提供：
    - 自动事件队列管理
    - 内置 ContentAggregator（自动追踪 full_content, reasoning, tool_calls）
    - 钩子系统（on_stream_start, on_event, on_stream_end, on_error）
    - 自动 meta.start / error 事件发送
    """

    def __init__(
        self,
        *,
        agent_runner: AgentRunner,
        hooks: OrchestratorHooks | None = None,
        event_queue_size: int = 10000,
    ):
        self._agent_runner = agent_runner
        self._hooks = hooks or OrchestratorHooks()
        self._event_queue_size = event_queue_size

    async def run(
        self,
        *,
        message: str,
        conversation_id: str,
        user_id: str,
        assistant_message_id: str | None = None,
        user_message_id: str | None = None,
        db: Any = None,
        **runner_kwargs: Any,
    ) -> AsyncGenerator[StreamEvent, None]:
        """运行编排流程
        
        Args:
            message: 用户消息
            conversation_id: 会话 ID
            user_id: 用户 ID
            assistant_message_id: 助手消息 ID（可选，自动生成）
            user_message_id: 用户消息 ID（可选，自动生成）
            db: 数据库会话（可选，会传入 ChatContext）
            **runner_kwargs: 传递给 agent_runner.run() 的额外参数
        
        Yields:
            StreamEvent
        """
        if assistant_message_id is None:
            assistant_message_id = str(uuid.uuid4())
        if user_message_id is None:
            user_message_id = str(uuid.uuid4())

        seq = 0
        def next_seq() -> int:
            nonlocal seq
            seq += 1
            return seq

        # 流开始钩子
        start_info = StreamStartInfo(
            conversation_id=conversation_id,
            user_id=user_id,
            user_message_id=user_message_id,
            assistant_message_id=assistant_message_id,
            message=message,
        )
        if self._hooks.on_stream_start:
            await self._hooks.on_stream_start(start_info)

        # meta.start
        yield make_event(
            seq=next_seq(),
            conversation_id=conversation_id,
            message_id=assistant_message_id,
            type=StreamEventType.META_START.value,
            payload={
                "user_message_id": user_message_id,
                "assistant_message_id": assistant_message_id,
            },
        )

        aggregator = ContentAggregator()

        try:
            loop = asyncio.get_running_loop()
            domain_queue: asyncio.Queue[dict[str, Any]] = asyncio.Queue(
                maxsize=self._event_queue_size
            )
            emitter = QueueDomainEmitter(queue=domain_queue, loop=loop)

            context = ChatContext(
                conversation_id=conversation_id,
                user_id=user_id,
                assistant_message_id=assistant_message_id,
                emitter=emitter,
                db=db,
            )

            # 启动 Agent
            producer_task = asyncio.create_task(
                self._agent_runner.run(
                    message=message,
                    context=context,
                    **runner_kwargs,
                )
            )

            # 消费事件
            while True:
                evt = await domain_queue.get()
                evt_type = evt.get("type")
                if evt_type == "__end__":
                    break

                payload = evt.get("payload", {})

                # 聚合
                aggregator.process_event(evt_type, payload)

                # 事件钩子
                if self._hooks.on_event:
                    await self._hooks.on_event(evt_type, payload, aggregator)

                # yield 事件
                yield make_event(
                    seq=next_seq(),
                    conversation_id=conversation_id,
                    message_id=assistant_message_id,
                    type=evt_type,
                    payload=payload,
                )

            await producer_task

            # 流结束钩子
            end_info = StreamEndInfo(
                conversation_id=conversation_id,
                user_id=user_id,
                assistant_message_id=assistant_message_id,
                aggregator=aggregator,
                context=context,
            )
            if self._hooks.on_stream_end:
                await self._hooks.on_stream_end(end_info)

        except Exception as e:
            # 错误钩子
            if self._hooks.on_error:
                await self._hooks.on_error(e, conversation_id)

            yield make_event(
                seq=next_seq(),
                conversation_id=conversation_id,
                message_id=assistant_message_id,
                type=StreamEventType.ERROR.value,
                payload={"message": str(e)},
            )

    def create_sse_response(self, **kwargs: Any) -> Any:
        """创建 FastAPI SSE 响应"""
        from langgraph_agent_kit.integrations.fastapi import create_sse_response
        return create_sse_response(self.run(**kwargs))
```

#### 4.1.2 更新 `__init__.py` 导出

```python
# 新增
from langgraph_agent_kit.orchestrator import (
    Orchestrator,
    OrchestratorHooks,
    AgentRunner,
    ContentAggregator,
    StreamStartInfo,
    StreamEndInfo,
)
```

#### 4.1.3 各项目改造效果预览

**embedease-ai 改造后**（从 221 行 `chat_stream_sdk.py` → ~50 行）：

```python
from langgraph_agent_kit import Orchestrator, OrchestratorHooks, StreamEndInfo

class MyAgentRunner:
    def __init__(self, agent_service):
        self._agent_service = agent_service
    
    async def run(self, message, context, **kwargs):
        await self._agent_service.chat_emit(
            message=message,
            conversation_id=context.conversation_id,
            user_id=context.user_id,
            context=context,
            agent_id=kwargs.get("agent_id"),
        )

async def save_assistant_message(info: StreamEndInfo):
    """落库钩子"""
    await conversation_service.add_message(
        conversation_id=info.conversation_id,
        role="assistant",
        content=info.aggregator.full_content,
        products=info.aggregator.products,
        tool_calls_data=info.aggregator.tool_calls_list,
        latency_ms=info.context.response_latency_ms,
    )

orchestrator = Orchestrator(
    agent_runner=MyAgentRunner(agent_service),
    hooks=OrchestratorHooks(on_stream_end=save_assistant_message),
)
```

**Skill-Know 改造后**（从 180 行 `chat.py` → ~60 行）：

```python
from langgraph_agent_kit import Orchestrator, OrchestratorHooks

class SkillKnowAgentRunner:
    async def run(self, message, context, **kwargs):
        agent = await agent_service.get_agent(...)
        handler = StreamingResponseHandler(emitter=context.emitter, ...)
        async for item in agent.astream(...):
            await handler.handle_message(item)
        await handler.finalize()
        await context.emitter.aemit("__end__", None)

orchestrator = Orchestrator(
    agent_runner=SkillKnowAgentRunner(),
    hooks=OrchestratorHooks(
        on_stream_end=save_to_db,
    ),
)
```

---

### 4.2 保留现有 API

`ChatStreamKit` 保持不变，不做删除。`Orchestrator` 作为新的推荐方式并存。

---

## 5. 第三阶段：SDK 版本发布与源码同步

### 5.1 版本号升级

所有三个包同步升级到 `0.2.0`：

| 包 | 文件 | 当前版本 | 目标版本 |
|---|---|---|---|
| `@embedease/chat-sdk` | `frontend/chat-sdk/package.json` | 0.1.17 | 0.2.0 |
| `@embedease/chat-sdk-react` | `frontend/chat-sdk-react/package.json` | 0.1.17 | 0.2.0 |
| `langgraph-agent-kit` | `backend/langgraph-agent-kit/pyproject.toml` | 0.1.17 | 0.2.0 |

### 5.2 更新 CHANGELOG

在 `CHANGELOG.md` 中记录所有新增 API。

### 5.3 构建前端 SDK

```bash
cd frontend/chat-sdk && pnpm build
cd frontend/chat-sdk-react && pnpm build
```

### 5.4 源码拷贝到各项目

#### embedease-ai

```bash
# 前端 SDK
rm -rf /Users/wang/code/my/embedease-ai/frontend/packages/chat-sdk
rm -rf /Users/wang/code/my/embedease-ai/frontend/packages/chat-sdk-react
cp -r /Users/wang/code/my/embedease-sdk/frontend/chat-sdk /Users/wang/code/my/embedease-ai/frontend/packages/
cp -r /Users/wang/code/my/embedease-sdk/frontend/chat-sdk-react /Users/wang/code/my/embedease-ai/frontend/packages/

# 后端 SDK
# embedease-ai 通过 pip install -e 或 sys.path 引用，需确认具体方式后同步
```

#### Skill-Know

```bash
# 前端 SDK
rm -rf /Users/wang/code/my/Skill-Know/frontend/packages/chat-sdk
rm -rf /Users/wang/code/my/Skill-Know/frontend/packages/chat-sdk-react
cp -r /Users/wang/code/my/embedease-sdk/frontend/chat-sdk /Users/wang/code/my/Skill-Know/frontend/packages/
cp -r /Users/wang/code/my/embedease-sdk/frontend/chat-sdk-react /Users/wang/code/my/Skill-Know/frontend/packages/

# 后端 SDK
# 同步 langgraph-agent-kit 源码
```

#### mobile-mcp

```bash
# mobile-mcp 的 package.json 指向 embedease-ai 的 packages 目录
# 更新 embedease-ai 后 mobile-mcp 自动获取最新版本
# 或者改为自己的 packages/ 目录：
mkdir -p /Users/wang/code/gitee/mobile-mcp/frontend/packages
cp -r /Users/wang/code/my/embedease-sdk/frontend/chat-sdk /Users/wang/code/gitee/mobile-mcp/frontend/packages/

# 更新 package.json 引用路径
# "@embedease/chat-sdk": "file:./packages/chat-sdk"
```

### 5.5 版本一致性检查

每次拷贝后，校验各项目 `node_modules/@embedease/chat-sdk/package.json` 中的 version 是否为 `0.2.0`。

> **建议**：可在 SDK 仓库增加一个脚本 `scripts/sync-to-projects.sh`，自动化拷贝 + 校验流程。

---

## 6. 第四阶段：embedease-ai 项目改造

### 6.1 前端改造

#### 6.1.1 删除 Adapter 层

**删除文件**：
- `frontend/lib/chat-adapter/new-adapter.ts`
- `frontend/lib/chat-adapter/types.ts`（如果不再需要）

**原因**：`createChatStoreSlice` 已经内置了 Adapter 提供的所有功能。

#### 6.1.2 改造 Chat Store

将现有的 chat store（使用 Adapter 的方式）改为使用 `createChatStoreSlice`。

#### 6.1.3 验证

- 聊天功能正常
- SSE 流式响应正常
- WebSocket 功能正常（如果使用）
- 历史消息加载正常

### 6.2 后端改造

#### 6.2.1 改造 `chat_stream_sdk.py`

将 `ChatStreamOrchestratorSDK` 替换为使用 `Orchestrator` + `OrchestratorHooks`。

保留 `chat_stream_adapter.py` 的适配器模式（SDK vs Legacy 切换），只是 SDK 分支改用新的 `Orchestrator`。

#### 6.2.2 验证

- SSE 流完整性：meta.start, assistant.delta, tool.start/end, assistant.final
- 落库正确性：assistant 消息、tool calls、products
- 错误处理：异常时正确发送 error 事件

---

## 7. 第五阶段：Skill-Know 项目改造

### 7.1 前端改造

#### 7.1.1 简化 `sdk-extensions/reducer.ts`

将 `businessReducer` 改为 `CustomReducer<BusinessTimelineItem>` 类型，使用 `composeReducers`。

#### 7.1.2 简化 `sdk-extensions/types.ts`

让 `BusinessTimelineItem` 继承 `TimelineItemBase`，移除 `as unknown` 强转。

#### 7.1.3 改造 `chat-store.ts`

用 `createChatStoreSlice` 替换手写的 store，传入自定义 reducer。

#### 7.1.4 验证

- 标准事件（assistant.delta, tool.start/end）正常显示
- 自定义事件（intent.extracted, search.results.found, tools.registered, phase.changed）正常显示
- 历史消息加载正常

### 7.2 后端改造

#### 7.2.1 改造 `services/chat.py`

将 `ChatService.chat_stream_with_tools` 中的手写编排逻辑替换为 `Orchestrator`。

提取 `AgentRunner` 实现：

```python
class SkillKnowAgentRunner:
    """Skill-Know 的 Agent 运行器"""
    
    def __init__(self, agent_service, prompt_service, session):
        self._agent_service = agent_service
        self._prompt_service = prompt_service
        self._session = session
    
    async def run(self, message, context, **kwargs):
        llm = await self._get_llm()
        system_prompt = await self._prompt_service.get_content("system.chat") or ""
        agent = await self._agent_service.get_agent(
            model=llm,
            system_prompt=system_prompt,
            session=self._session,
            emitter=context.emitter,
        )
        handler = StreamingResponseHandler(
            emitter=context.emitter,
            conversation_id=context.conversation_id,
        )
        async for item in agent.astream(..., context=context):
            msg = item[0] if isinstance(item, (tuple, list)) and item else item
            await handler.handle_message(msg)
        await handler.finalize()
        await context.emitter.aemit("__end__", None)
```

#### 7.2.2 验证

- 聊天流正常
- 自定义事件（intent.extracted 等）仍能正常发送
- 落库正确

---

## 8. 第六阶段：mobile-mcp 项目改造

### 8.1 前端改造

#### 8.1.1 改造 `agent-store.ts`

用 `createChatStoreSlice` 替换手写的 store，通过 `onEvent` 回调处理 screenshot_id 注入。

#### 8.1.2 改造 SDK 引用路径

将 `package.json` 中的绝对路径改为本地 `packages/` 目录：

```json
"@embedease/chat-sdk": "file:./packages/chat-sdk"
```

#### 8.1.3 验证

- 聊天功能正常
- screenshot_id 在 tool.end 后正确注入到 ToolCallItem
- 流式响应正常

### 8.2 无后端改造

mobile-mcp 的后端是独立的 Python 项目，不使用 `langgraph-agent-kit`，无需改造。

---

## 9. 风险与注意事项

### 9.1 向后兼容风险

| 风险 | 缓解措施 |
|---|---|
| 泛型化可能导致类型推断变化 | 默认泛型参数 `= TimelineItem`，确保现有代码不需修改 |
| `composeReducers` 的 reducer 顺序 | 文档明确：用户 reducer 优先，未处理的交给内置 |
| `createChatStoreSlice` 的 `set` 签名 | 兼容 zustand v4 和 v5 |

### 9.2 源码拷贝注意事项

| 注意项 | 说明 |
|---|---|
| **不要拷贝 node_modules** | 拷贝 `chat-sdk/` 时排除 `node_modules/` |
| **不要拷贝 .git** | 如有子模块要排除 |
| **拷贝后重新 install** | 各项目 `pnpm install` 重新链接依赖 |
| **拷贝后构建** | 确保 `dist/` 目录是最新的，或各项目配置 tsup watch |

### 9.3 改造顺序

**必须按阶段顺序执行**：
1. 先改 SDK，确保 build 通过
2. 再拷贝到项目
3. 最后改项目代码

**项目改造顺序建议**：embedease-ai → Skill-Know → mobile-mcp（从复杂到简单）

---

## 10. 验收标准

### SDK 验收

- [ ] `pnpm build` 前端两个包均成功
- [ ] `TimelineState<T>` 泛型工作正常
- [ ] `composeReducers` 支持链式 reducer
- [ ] `createChatStoreSlice` 创建的 store 功能完整
- [ ] `Orchestrator` 后端编排器功能完整
- [ ] `ContentAggregator` 正确聚合内容
- [ ] 所有现有 API 未被破坏（向后兼容）
- [ ] 版本号统一为 0.2.0

### 项目验收

- [ ] embedease-ai：删除 adapter 层，chat 功能正常
- [ ] Skill-Know：自定义事件通过 `composeReducers` 正常工作
- [ ] mobile-mcp：screenshot_id 通过 `onEvent` 正常注入
- [ ] 各项目 SDK 版本号一致为 0.2.0
