# Claude Code 配置进阶指南

> 深度覆盖：复杂配置管理 · 项目隔离策略 · 自定义模型接入 · 多模型共存方案

---

## 目录

| 文件 | 内容 |
|------|------|
| [01-config-architecture.md](./01-config-architecture.md) | 配置层级架构、合并规则、调试诊断 |
| [02-project-isolation.md](./02-project-isolation.md) | 项目级配置隔离：团队共享 vs 个人私有、多环境 |
| [03-custom-models.md](./03-custom-models.md) | 自定义模型接入：第三方代理、本地模型、云厂商 |
| [04-profile-system.md](./04-profile-system.md) | Profile 化配置管理：切换脚本、多端点共存 |
| [05-model-routing.md](./05-model-routing.md) | 按任务路由模型：成本优化、质量分层 |

---

## 适用场景

- 团队项目需要统一配置，又不能提交密钥
- 需要在多个 API 端点（主/备/测试）之间切换
- 使用私有部署的模型或企业 API 网关
- 想用本地模型（Ollama/LM Studio）降低成本
- 不同任务需要不同模型（快速响应 vs 复杂推理）
- 开发/测试/生产环境需要不同的配置

---

## 配置文件位置速查

```
~/.claude/settings.json          ← 用户全局（CLI + VS Code）
~/.claude/settings.local.json    ← 用户本地私有（不共享）
<repo>/.claude/settings.json     ← 项目共享（提交 git）
<repo>/.claude/settings.local.json  ← 项目本地私有（gitignore）
<repo>/CLAUDE.md                 ← 项目记忆（提交 git）
<repo>/CLAUDE.local.md           ← 项目本地记忆（gitignore）
~/.claude/commands/              ← 用户级自定义命令
<repo>/.claude/commands/         ← 项目级自定义命令
<repo>/.mcp.json                 ← 项目 MCP 配置（可提交）
```

## 配置优先级（高 → 低）

```
会话命令（/model）
  > CLI 参数（--model）
  > 环境变量（ANTHROPIC_MODEL）
  > 项目本地（.claude/settings.local.json）
  > 项目共享（.claude/settings.json）
  > 用户全局（~/.claude/settings.json）
  > 企业托管（managed-settings.json）
  > 默认值
```

> **合并规则**：`permissions.allow/deny`、`hooks`、`mcpServers` 等数组**跨层合并**；
> `model`、`env`、`effortLevel` 等标量**高优先级覆盖**低优先级。
