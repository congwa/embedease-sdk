# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.18] - 2025-02-12

### 核心亮点

本版本完成 **开发工具集成**，为 embedease-sdk 项目添加完整的 Windsurf skills 工具集，包括图片上传、README 图片处理、技能创建和发版流程自动化，提升 SDK 开发和维护效率。

### Added

- **Windsurf Skills 工具集**
  - `qiniu-upload` - 七牛云图片上传工具，支持 PicList/PicGo 本地服务
  - `readme-images` - README.md 图片引用处理，支持 GitHub/Gitee 双平台
  - `skill-creator` - 技能创建指南，提供完整的技能开发流程
  - `release` - 发版流程自动化，支持版本管理、CHANGELOG 更新、标签创建

- **开发流程优化**
  - 标准化发版流程，包含版本检查、变更分析、CHANGELOG 更新
  - 自动化标签创建和推送
  - 支持 GitHub 平台发布

### Changed

- **项目结构**
  - 添加 `.windsurf/skills/` 目录结构
  - 集成开发工具脚本和文档

---

## [0.1.17] - 2025-02-05

### 核心亮点

本版本完成 **SDK 完整迁移**，将 streaming 组件、chat_models（v0/v1）、Payload TypedDict 全部迁移到 `langgraph-agent-kit` SDK，简化项目结构，增强代码复用性。

### Added

- **langgraph-agent-kit 增强**
  - `chat_models/` - v0/v1 双版本模型基类和 `create_chat_model()` 工厂
  - `SiliconFlowV1ChatModel` / `SiliconFlowReasoningChatModel` - 硅基流动推理模型支持
  - 24 个 Payload TypedDict 定义，支持 IDE 自动补全和类型检查
  - `parse_content_blocks()` / `ParsedContent` - content_blocks 解析工具
  - 类型守卫函数：`is_text_block`, `is_reasoning_block`, `is_tool_call_block` 等

- **前端 SDK 包**
  - `@embedease/chat-sdk` - 前端聊天核心 SDK
  - `@embedease/chat-sdk-react` - React Hooks 封装

### Changed

- **项目结构简化**：删除重复文件，统一使用 SDK 版本
- **导入路径更新**：文件更新为从 SDK 导入
- **README.md 更新**：补充 Chat Models、Content Parser、Payload TypedDict 文档
