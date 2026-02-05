# embedease-sdk

统一的流式聊天 SDK 工具包，包含前端和后端组件。

## 包结构

```
embedease-sdk/
├── frontend/
│   ├── chat-sdk/           # 前端核心 SDK
│   └── chat-sdk-react/     # React 封装
└── backend/
    └── langgraph-agent-kit/ # 后端 Python SDK
```

## 前端 SDK

### 安装（通过 submodule）

在消费方项目的 `package.json` 中配置：

```json
{
  "dependencies": {
    "@embedease/chat-sdk": "file:./packages/embedease-sdk/frontend/chat-sdk",
    "@embedease/chat-sdk-react": "file:./packages/embedease-sdk/frontend/chat-sdk-react"
  }
}
```

### 构建

```bash
cd frontend/chat-sdk && pnpm install && pnpm build
cd frontend/chat-sdk-react && pnpm install && pnpm build
```

## 后端 SDK

### 安装（通过 submodule）

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

## 版本管理

使用 Git Tag 管理版本：

```bash
git tag v0.1.16
git push origin v0.1.16
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

## License

MIT
