# embedease-sdk

统一的流式聊天 SDK 工具包，包含前端和后端组件。

> 当前版本：**v0.2.0**

> [!IMPORTANT]
> 本项目不再继续维护。现有代码会保留用于历史项目兼容、协议参考和迁移对照，但后续不再新增功能，也不再继续演进自定义聊天协议。
>
> 推荐新项目直接使用 [AI SDK](https://ai-sdk.dev/) 作为前端协议与渲染层，后端可继续搭配 LangChain Python / LangGraph。更准确地说，建议保留 LangChain 负责 agent 编排与 tool 调用，把流式协议切换为 AI SDK 的 UIMessage Stream / SSE 协议，并通过 `data-*` part 扩展业务 UI。

## 停维说明

### 为什么不再维护

- 本项目本质上维护了一套自定义聊天协议与前端状态层：前端围绕 `timelineReducer` / `useChat` / Zustand Store，后端围绕 `meta.start`、`assistant.delta`、`tool.start`、`tool.end` 等事件。
- 这套能力和 AI SDK 当前提供的 `useChat`、transport、UIMessage、SSE stream protocol、tool parts、`data-*` custom data parts 已经高度重叠。
- 继续维护独立协议，意味着要持续投入在协议演进、多前端框架适配、消息持久化、工具调用渲染、流重连与错误恢复上，维护收益已经低于直接复用 AI SDK 官方体系。
- LangChain / LangGraph 更适合专注在 agent 编排、RAG、tool orchestration；而前端消息协议与生成式 UI 渲染，交给 AI SDK 更标准，也更容易和社区生态对齐。

### 推荐替代方案

- 前端：使用 AI SDK 的 `useChat`，按 `message.parts` 渲染文本、工具结果和自定义业务卡片。
- 后端：输出 AI SDK UIMessage Stream 协议，保持 SSE 返回头 `x-vercel-ai-ui-message-stream: v1`。
- 扩展协议：不要再扩一套新的聊天事件名，优先使用官方 `data-*` custom data parts 承载业务 UI。
- Agent 层：继续使用 LangChain Python / LangGraph，不需要迁移 agent 编排思路，只需要把流式输出映射到 AI SDK 协议。

### 和当前 SDK 的使用对比

| 场景 | 当前 `embedease-sdk` | 推荐方案 |
|-----|------|------|
| 前端状态入口 | `createChatStoreSlice()` / `useChat()` / `timelineReducer` | `useChat()` + `DefaultChatTransport` |
| 后端输出协议 | 自定义 SSE 事件：`meta.start`、`assistant.delta`、`tool.start`、`tool.end` | AI SDK UIMessage Stream：`start`、`text-*`、`tool-*`、`start-step`、`finish-step`、`finish` |
| 业务扩展方式 | `composeReducers()` 组合自定义 reducer | 官方 `data-*` custom data parts |
| 工具调用展示 | 自己维护 `tool.call` timeline item | 按 `message.parts` 中的 tool parts / data parts 渲染 |
| Python 后端接入 | 自己维护 SSE 编码和事件语义 | Python 只需输出兼容的 SSE part，前端仍可直接使用 `useChat` |
| 跨后端/传输层 | 需要自己封装 | AI SDK 原生支持 transport-based architecture，可继续走 HTTP，也可自定义 transport |

### 如何使用对比

**当前 SDK**

前端通常这样接入：

```typescript
import { create } from "zustand";
import { createChatStoreSlice } from "@embedease/chat-sdk-react";

export const useChatStore = create(
  createChatStoreSlice({ baseUrl: "/api" })
);
```

后端通常这样组织语义：

```text
meta.start -> assistant.delta -> tool.start -> tool.end -> assistant.final
```

**推荐方案：AI SDK**

前端保留 `useChat`，但把后端改成输出 AI SDK UIMessage Stream：

```typescript
import { useChat } from "@ai-sdk/react";
import { DefaultChatTransport } from "ai";

export function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit } = useChat({
    transport: new DefaultChatTransport({
      api: "http://localhost:8000/chat",
    }),
  });

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
        <button type="submit">Send</button>
      </form>

      {messages.map((message) => (
        <div key={message.id}>
          {message.parts.map((part, index) => {
            if (part.type === "text") {
              return <p key={index}>{part.text}</p>;
            }

            if (part.type === "data-status") {
              return (
                <div key={index}>
                  <strong>{part.data.stage}</strong>: {part.data.message}
                </div>
              );
            }

            return null;
          })}
        </div>
      ))}
    </div>
  );
}
```

对应的后端语义会变成：

```text
start -> start-step -> text-start -> text-delta -> data-status -> text-end -> finish-step -> finish
```

### AI SDK + LangChain Python 最小示例

下面这个最小示例保留 LangChain Python 负责模型调用，前端直接使用 AI SDK 的 `useChat`。示例里额外加入了 `data-status`，用来演示官方 `data-*` 扩充协议。

**前端：React + `useChat`**

```tsx
import { useChat } from "@ai-sdk/react";
import { DefaultChatTransport } from "ai";

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit } = useChat({
    transport: new DefaultChatTransport({
      api: "http://localhost:8000/chat",
    }),
  });

  return (
    <main>
      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="问一个问题"
        />
        <button type="submit">发送</button>
      </form>

      {messages.map((message) => (
        <section key={message.id}>
          {message.parts.map((part, index) => {
            if (part.type === "text") {
              return <p key={index}>{part.text}</p>;
            }

            if (part.type === "data-status") {
              return (
                <div key={index}>
                  阶段：{part.data.stage}，说明：{part.data.message}
                </div>
              );
            }

            return null;
          })}
        </section>
      ))}
    </main>
  );
}
```

**后端：FastAPI + LangChain Python + AI SDK SSE 协议**

```python
import json
import uuid
from collections.abc import AsyncIterator

from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from langchain_core.messages import HumanMessage
from langchain_openai import ChatOpenAI

app = FastAPI()
model = ChatOpenAI(model="gpt-4o-mini", temperature=0)


def to_sse(part: dict) -> str:
    return f"data: {json.dumps(part, ensure_ascii=False)}\n\n"


def get_last_user_text(messages: list[dict]) -> str:
    for message in reversed(messages):
        if message.get("role") != "user":
            continue

        parts = message.get("parts", [])
        for part in parts:
            if part.get("type") == "text":
                return part.get("text", "")

        content = message.get("content")
        if isinstance(content, str):
            return content

    return ""


def chunk_to_text(chunk) -> str:
    content = getattr(chunk, "content", "")
    if isinstance(content, str):
        return content
    if isinstance(content, list):
        return "".join(
            item.get("text", "")
            for item in content
            if isinstance(item, dict) and item.get("type") == "text"
        )
    return ""


@app.post("/chat")
async def chat(request: Request) -> StreamingResponse:
    body = await request.json()
    prompt = get_last_user_text(body.get("messages", []))

    async def event_stream() -> AsyncIterator[str]:
        message_id = f"msg_{uuid.uuid4().hex}"
        text_id = f"text_{uuid.uuid4().hex}"

        yield to_sse({"type": "start", "messageId": message_id})
        yield to_sse({"type": "start-step"})
        yield to_sse({
            "type": "data-status",
            "data": {
                "stage": "langchain",
                "message": "LangChain 已开始执行模型调用",
            },
        })
        yield to_sse({"type": "text-start", "id": text_id})

        async for chunk in model.astream([HumanMessage(content=prompt)]):
            delta = chunk_to_text(chunk)
            if delta:
                yield to_sse({
                    "type": "text-delta",
                    "id": text_id,
                    "delta": delta,
                })

        yield to_sse({"type": "text-end", "id": text_id})
        yield to_sse({
            "type": "data-status",
            "data": {
                "stage": "done",
                "message": "本轮回答已完成",
            },
        })
        yield to_sse({"type": "finish-step"})
        yield to_sse({"type": "finish"})

    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "x-vercel-ai-ui-message-stream": "v1",
        },
    )
```

### 迁移建议

- 如果你当前已经使用 `meta.start`、`assistant.delta`、`tool.start`、`tool.end`，可以先做一层事件映射，不必一次性重写 agent。
- 如果你当前依赖自定义 reducer 渲染业务卡片，迁移时优先把这些事件改为 `data-*` part，例如 `data-status`、`data-search-results`、`data-file-preview`。
- 如果你仍然需要保留 Python 后端，没有问题；AI SDK 官方协议本身就支持由 Python / FastAPI 输出兼容的 SSE。

### 参考资料

- [AI SDK 官网](https://ai-sdk.dev/)
- [AI SDK `useChat`](https://ai-sdk.dev/docs/reference/ai-sdk-ui/use-chat)
- [AI SDK Stream Protocol](https://ai-sdk.dev/docs/ai-sdk-ui/stream-protocol)
- [AI SDK Streaming Custom Data](https://ai-sdk.dev/docs/ai-sdk-ui/streaming-data)
- [AI SDK Transport](https://ai-sdk.dev/docs/ai-sdk-ui/transport)

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
