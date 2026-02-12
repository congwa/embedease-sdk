# embedease-sdk

统一的流式聊天 SDK 工具包，包含前端和后端组件。

> 当前版本：**v0.2.0**

## 包结构

```
embedease-sdk/
├── frontend/
│   ├── chat-sdk/           # 前端核心 SDK (@embedease/chat-sdk)
│   └── chat-sdk-react/     # React 封装 (@embedease/chat-sdk-react)
└── backend/
    └── langgraph-agent-kit/ # 后端 Python SDK (langgraph-agent-kit)
```

---

## 前端 SDK

### 安装

在消费方项目的 `package.json` 中配置（源码拷贝方式）：

```json
{
  "dependencies": {
    "@embedease/chat-sdk": "file:./packages/chat-sdk",
    "@embedease/chat-sdk-react": "file:./packages/chat-sdk-react"
  }
}
```

### 构建

```bash
cd frontend/chat-sdk && pnpm install && pnpm build
cd frontend/chat-sdk-react && pnpm install && pnpm build
```

### 快速开始 — Zustand Store Factory（v0.2.0 新增）

最简用法，10 行代码接入聊天功能：

```typescript
import { create } from "zustand";
import { createChatStoreSlice } from "@embedease/chat-sdk-react";

export const useChatStore = create(
  createChatStoreSlice({ baseUrl: "/api" })
);
```

Store 内置以下功能：
- `sendMessage(message)` — 发送消息并自动流式处理
- `abortStream()` — 中止当前流
- `clearMessages()` — 清空消息
- `setConversationId(id)` — 设置会话 ID
- `initFromHistory(messages)` — 从历史消息初始化

### 自定义事件 — Reducer 插件机制（v0.2.0 新增）

通过 `composeReducers` 支持自定义事件类型，无需 hack：

```typescript
import { create } from "zustand";
import {
  createChatStoreSlice,
  composeReducers,
  insertItem,
  type CustomReducer,
  type TimelineItemBase,
} from "@embedease/chat-sdk-react";

// 定义自定义 TimelineItem 类型
type MyItem = TimelineItemBase | { type: "custom.event"; id: string; turnId: string; ts: number; data: unknown };

// 自定义 reducer
const myReducer: CustomReducer<MyItem> = (state, event) => {
  const evt = event as Record<string, unknown>;
  if (evt.type === "custom.event") {
    return insertItem(state, { type: "custom.event", id: String(evt.seq), turnId: "", ts: Date.now(), data: evt.payload } as MyItem);
  }
  return null; // 未处理，交给 SDK 内置 reducer
};

export const useChatStore = create(
  createChatStoreSlice<MyItem>({
    baseUrl: "/api",
    reducer: composeReducers<MyItem>(myReducer),
  })
);
```

### 事件中间件

通过 `onEvent` 回调在事件到达时执行自定义逻辑：

```typescript
createChatStoreSlice({
  baseUrl: "/api",
  onEvent: (event, api) => {
    // 注入自定义字段、过滤事件等
    if (event.type === "tool.end") {
      // 自定义处理...
    }
    return event; // 返回 null 跳过此事件
  },
  onStreamEnd: async ({ conversationId, fullContent }) => {
    // 流结束回调（如落库通知）
  },
});
```

### 泛型类型系统（v0.2.0 新增）

`TimelineState`、`insertItem`、`updateItemById` 等核心类型和函数支持泛型扩展：

```typescript
import type { TimelineState, TimelineItemBase } from "@embedease/chat-sdk";

// 自定义 Item 类型
interface MyCustomItem extends TimelineItemBase {
  type: "my.custom";
  data: string;
}

type MyTimelineItem = TimelineItem | MyCustomItem;
type MyState = TimelineState<MyTimelineItem>;
```

### 原有 API

以下 API 保持不变（向后兼容）：

- `ChatClient` — SSE 流式客户端
- `timelineReducer` — 内置 Timeline reducer
- `createInitialState` / `addUserMessage` / `startAssistantTurn` / `endTurn` / `clearTurn`
- `historyToTimeline` — 历史消息转 Timeline
- `useChat` / `useTimeline` / `useWebSocket` — React Hooks

---

## 后端 SDK

### 安装

在消费方项目的 `pyproject.toml` 中配置：

```toml
dependencies = [
    "langgraph-agent-kit @ file:./packages/embedease-sdk/backend/langgraph-agent-kit"
]
```

或使用可编辑安装：

```bash
uv pip install -e ./packages/embedease-sdk/backend/langgraph-agent-kit
```

### 快速开始 — Orchestrator（v0.2.0 新增）

钩子化的编排器，消除重复的 queue 消费 + 落库逻辑：

```python
from langgraph_agent_kit import Orchestrator, OrchestratorHooks, StreamEndInfo

# 1. 实现 AgentRunner
class MyAgentRunner:
    async def run(self, message, context, **kwargs):
        """通过 context.emitter 发送事件"""
        agent = build_my_agent(context.emitter)
        async for chunk in agent.astream({"messages": [{"role": "user", "content": message}]}):
            await handler.handle(chunk)
        await context.emitter.aemit("__end__", None)

# 2. 定义钩子
async def save_to_db(info: StreamEndInfo):
    await db.save_message(
        conversation_id=info.conversation_id,
        content=info.aggregator.full_content,
        tool_calls=info.aggregator.tool_calls_list,
    )

# 3. 创建编排器
orchestrator = Orchestrator(
    agent_runner=MyAgentRunner(),
    hooks=OrchestratorHooks(on_stream_end=save_to_db),
)

# 4. 在路由中使用
async for event in orchestrator.run(message="你好", conversation_id="c1", user_id="u1"):
    yield event
```

**Orchestrator 自动提供：**
- 事件队列管理（`QueueDomainEmitter` + `asyncio.Queue`）
- `ContentAggregator` — 自动追踪 `full_content`、`reasoning`、`tool_calls`、`products`
- `meta.start` / `error` 事件自动发送
- 钩子系统：`on_stream_start` / `on_event` / `on_stream_end` / `on_error`

### 原有 API

以下 API 保持不变（向后兼容）：

- `ChatStreamKit` — 简易编排器（适合简单场景）
- `StreamEventType` — 事件类型枚举
- `StreamEvent` / `make_event` / `encode_sse` — 事件构造
- `ChatContext` / `QueueDomainEmitter` — 上下文与事件发射
- `StreamingResponseHandler` — 流响应处理
- Middleware / Tools 注册系统

---

## 版本管理

使用 Git Tag 管理版本：

```bash
git tag v0.2.0
git push origin v0.2.0
```

## 消费方使用

### 添加为 Submodule

```bash
git submodule add https://github.com/congwa/embedease-sdk.git packages/embedease-sdk
```

### 更新到最新版本

```bash
cd packages/embedease-sdk
git pull origin main
cd ../..
git add packages/embedease-sdk
git commit -m "chore: 更新 embedease-sdk"
```

### 更新到指定 Tag

```bash
cd packages/embedease-sdk
git fetch --tags
git checkout v0.2.0
cd ../..
git add packages/embedease-sdk
git commit -m "chore: 更新 embedease-sdk 到 v0.2.0"
```

---

## v0.2.0 更新摘要

### 前端新增

| API | 说明 |
|-----|------|
| `createChatStoreSlice()` | Zustand store 工厂，一行创建聊天 store |
| `composeReducers()` | Reducer 组合器，支持自定义事件类型 |
| `TimelineItemBase` | 基础 Item 接口，用于泛型扩展 |
| `TimelineState<T>` | 泛型化 Timeline 状态 |
| `insertItem<T>()` | 泛型化 item 插入（供自定义 reducer 使用） |
| `updateItemById<T>()` | 泛型化 item 更新 |
| `removeWaitingItem<T>()` | 泛型化等待项移除 |
| `CustomReducer<T>` | 自定义 reducer 类型定义 |

### 后端新增

| API | 说明 |
|-----|------|
| `Orchestrator` | 钩子化编排器 |
| `OrchestratorHooks` | 编排器钩子配置 |
| `AgentRunner` | Agent 运行器协议 |
| `ContentAggregator` | 内容聚合器 |
| `StreamStartInfo` | 流开始信息（传递给钩子） |
| `StreamEndInfo` | 流结束信息（传递给钩子） |

## License

MIT
