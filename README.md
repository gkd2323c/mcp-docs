# MCP Documentation

[Model Context Protocol](https://modelcontextprotocol.io) 官方文档的本地化整理版。去除了 Mintlify JSX 噪声，按主题拆分为多个 Markdown 文件，方便本地阅读、搜索和 LLM 上下文加载。

## 文档来源

原始内容来自 modelcontextprotocol.io 的 `llms-full.txt`（官方提供的 LLM 友好完整转储），经过清洗和拆分处理：

- 去除 `theme={null}`、`<Note>`、`<Warning>`、`<Badge>`、`<McpClient>` 等 JSX 组件标记
- 清理 `<img>`、`<Icon>`、`<FeatureBadge />`、`<CHECK />` 等无意义标签
- 简化内部锚点链接，保留外部有用链接
- 按主题拆分为独立文件，每文件首部自带章节目录

## 文件结构

```
mcp-docs/
├── 01-getting-started.md   入门概念、架构概览、版本策略、SDK 清单
├── 02-development.md       构建 MCP Server/Client、连接本地/远程、最佳实践
├── 03-security-auth.md     安全最佳实践、OAuth 2.1 认证、企业级授权
├── 04-spec-core.md         规范核心：生命周期、传输协议（stdio/SSE/Streamable HTTP）、取消/进度
├── 05-spec-features.md     规范功能：授权、Tasks、Elicitation、Roots、Sampling、Prompts、Resources、Tools 等
├── 06-schema-reference.md  JSON-RPC Schema 详细参考
├── 07-ecosystem.md         Example Clients/Server 列表、MCP Apps 开发
├── 08-tools.md             调试指南、MCP Inspector
├── 09-registry.md          MCP Registry 发布、认证、版本管理、GitHub Actions 自动化
├── 10-community.md         贡献指南、治理结构、工作组章程、SDK 分级
└── 11-seps.md              全部 35+ 个 SEP（协议改进提案）
```

## 内容范围

涵盖 MCP 官方文档站的全部页面：

- **入门指南**：什么是 MCP、核心架构、客户端/服务端概念
- **开发教程**：Python / TypeScript / Java / .NET / Ruby / Go / Kotlin 多语言构建指南
- **规范 2025-11-25**：架构、授权（OAuth 2.1）、生命周期、传输层、JSON-RPC 消息格式
- **扩展机制**：MCP Apps（交互式 UI）、认证扩展（企业托管授权、OAuth Client Credentials）
- **Registry**：服务器注册、版本管理、npm 发布、GitHub Actions 自动化
- **安全**：安全最佳实践、威胁模型、数据防泄露
- **30+ SEPs**：从安全要求、工具命名、OAuth 到任务系统、JSON Schema 方言等协议改进提案
- **社区治理**：工作组章程模板、贡献者阶梯、设计原则

## 使用方法

直接打开对应话题的文件即可。每个文件首部有内部子目录，可快速定位到具体章节。

### 示例

想了解 MCP 生命周期 → 打开 `04-spec-core.md`，搜索 "Lifecycle"
想了解 OAuth 认证 → 打开 `03-security-auth.md`
想查看某个 SEP 提案 → 打开 `11-seps.md`，搜索对应的 SEP 编号

## 与官方文档的关系

这份文档是 MCP 官方内容的**格式优化版本**，内容与 https://modelcontextprotocol.io 保持一致。当官方网站更新时，可以重新拉取 `llms-full.txt` 并运行同样的清理流程来同步。

## License

内容版权归 [Model Context Protocol](https://modelcontextprotocol.io) 项目所有。本仓库仅作格式优化和本地化整理。
