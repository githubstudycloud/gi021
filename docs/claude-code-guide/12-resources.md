# 社区资源与推荐阅读

> 精选 Claude Code 生态的优质资源，包括官方文档、GitHub 项目、中文教程。

---

## 一、官方资源

| 资源 | 地址 | 说明 |
|------|------|------|
| 官方文档 | https://docs.anthropic.com/en/docs/claude-code | 完整 API 文档、配置参考 |
| GitHub | https://github.com/anthropics/claude-code | 官方仓库，Issues 追踪 |
| 变更日志 | https://docs.anthropic.com/en/docs/claude-code/changelog | 版本更新记录 |
| 模型概览 | https://docs.anthropic.com/en/docs/about-claude/models | 所有可用模型列表 |

---

## 二、精选 GitHub 项目

### 配置与技巧

| 项目 | 说明 |
|------|------|
| [hesreallyhim/awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) | Claude Code 生态精选列表，CLAUDE.md 模板、自定义命令、工具集合 |
| [trailofbits/claude-code-config](https://github.com/trailofbits/claude-code-config) | Trail of Bits 安全团队的生产级配置，三层沙箱架构 |
| [feiskyer/claude-code-settings](https://github.com/feiskyer/claude-code-settings) | Vibe Coding 完整配置，Kiro 工作流，GitHub Copilot 代理集成 |
| [ykdojo/claude-code-tips](https://github.com/ykdojo/claude-code-tips) | 45 条实用技巧，YouTube 视频配套资料 |
| [januff/claude-code-tips](https://github.com/januff/claude-code-tips) | 397 条社区精选技巧 |

### 工具与扩展

| 项目 | 说明 |
|------|------|
| [FlorianBruniaux/claude-code-ultimate-guide](https://github.com/FlorianBruniaux/claude-code-ultimate-guide) | 英文全面指南 |

---

## 三、中文教程资源

### claude-cn.org

国内最完整的 Claude Code 中文教程站：

| 文章 | 关键内容 |
|------|---------|
| Claude Code 最佳实践 | 探索-计划-编码-提交工作流，TDD，MCP 配置 |
| 34 条实用技巧 | CLI 技巧、CLAUDE.md 优化、斜杠命令、UI 迭代 |
| 命令速查 | 10 分钟掌握所有命令，MCP 安装实战 |
| 工作流指南 | 12 种常见工作流，Worktree 并行开发 |
| 提示词大全 | 各场景提示词模板库 |
| Vibe Coding | feiskyer 开源配置，Kiro 规范驱动流程 |
| Sub-agents 指南 | 自定义 Agent 文件格式，链式调用 |
| CLAUDE.md 约束技巧 | 代码架构约束，Python/React 规范 |

---

## 四、MCP 服务器资源

### 官方 MCP 服务器

| 服务器 | npm 包 | 功能 |
|--------|--------|------|
| GitHub | `@modelcontextprotocol/server-github` | GitHub 仓库、Issue、PR 操作 |
| 文件系统 | `@modelcontextprotocol/server-filesystem` | 扩展文件访问范围 |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | PostgreSQL 数据库 |
| Sequential Thinking | `@modelcontextprotocol/server-sequential-thinking` | 结构化多步推理 |
| Puppeteer | `@modelcontextprotocol/server-puppeteer` | 浏览器自动化 |

### 第三方精选 MCP

| 服务器 | npm 包 | 功能 |
|--------|--------|------|
| Context7 | `@upstash/context7-mcp` | 第三方库实时最新文档 |
| Playwright | `@playwright/mcp` | 浏览器自动化（功能更全） |
| Exa | `exa-mcp-server` | AI 原生语义搜索 |
| Firecrawl | `firecrawl-mcp` | 高质量网页爬取 |

### MCP 发现平台

- **mcp.so**：MCP 服务器搜索引擎
- **MCP Hub**：https://mcphub.io
- **Smithery**：https://smithery.ai

---

## 五、视频与课程

| 资源 | 平台 | 说明 |
|------|------|------|
| ykdojo Claude Code Tips | YouTube | 45 条技巧视频讲解 |
| Anthropic 官方演示 | YouTube | 产品演示和最佳实践 |

---

## 六、社区与讨论

| 平台 | 说明 |
|------|------|
| Claude Code GitHub Issues | 官方 Bug 报告和功能请求 |
| Reddit r/ClaudeAI | Claude 用户社区 |
| Discord Anthropic 官方 | 开发者交流 |

---

## 七、本指南相关文件

```
docs/claude-code-guide/
├── README.md                 ← 总览与快速入门
├── 01-install-setup.md       ← 安装、认证、国内配置、VS Code
├── 02-claude-md-mastery.md   ← CLAUDE.md 深度指南
├── 03-commands-reference.md  ← CLI 命令与斜杠命令速查
├── 04-workflows.md           ← 核心工作流（TDD、UI、并行）
├── 05-context-management.md  ← 上下文管理与成本控制
├── 06-hooks-system.md        ← Hooks 事件钩子系统
├── 07-mcp-servers.md         ← MCP 服务器配置与推荐
├── 08-skills-slash-commands.md ← 自定义斜杠命令
├── 09-advanced-tips.md       ← 100+ 进阶技巧
├── 10-multi-agent.md         ← 多 Agent 并行架构
├── 11-security-sandbox.md    ← 安全与权限管理
└── 12-resources.md           ← 资源与推荐阅读（本文）
```

---

## 八、版本说明

本指南基于以下版本编写：
- **Claude Code CLI**：v2.1.76
- **VS Code 插件**：anthropic.claude-code-2.1.76
- **模型**：Claude Sonnet 4.6 / Opus 4.6 / Haiku 4.5
- **更新时间**：2026-03-16

### 如何保持最新

```bash
# 更新 Claude Code CLI
npm update -g @anthropic-ai/claude-code

# 查看当前版本
claude --version

# 查看官方更新日志
# https://docs.anthropic.com/en/docs/claude-code/changelog
```
