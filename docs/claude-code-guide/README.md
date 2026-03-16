# Claude Code 完全指南

> 综合来源：官方文档 · claude-cn.org · GitHub 社区精选 · Trail of Bits 实战配置
> 版本：v2.1.76 | 更新：2026-03-16

---

## 目录

| 文件 | 内容 |
|------|------|
| [01-install-setup.md](./01-install-setup.md) | 安装、认证、国内配置、跨平台指南 |
| [02-claude-md-mastery.md](./02-claude-md-mastery.md) | CLAUDE.md 深度指南：结构、模板、分层策略 |
| [03-commands-reference.md](./03-commands-reference.md) | CLI 命令、斜杠命令、快捷键完整速查 |
| [04-workflows.md](./04-workflows.md) | 核心工作流：TDD、UI迭代、并行开发、CI/CD |
| [05-context-management.md](./05-context-management.md) | 上下文管理：压缩、清理、会话恢复、成本控制 |
| [06-hooks-system.md](./06-hooks-system.md) | Hooks 系统：事件、写法、安全应用 |
| [07-mcp-servers.md](./07-mcp-servers.md) | MCP 协议：配置、推荐服务器、最佳实践 |
| [08-skills-slash-commands.md](./08-skills-slash-commands.md) | Skills 与自定义斜杠命令：创建、模板、自动触发 |
| [09-advanced-tips.md](./09-advanced-tips.md) | 100+ 实用技巧：社区精华、高级技法 |
| [10-multi-agent.md](./10-multi-agent.md) | 多 Agent 与 Sub-agents：并行架构、Worktree |
| [11-security-sandbox.md](./11-security-sandbox.md) | 安全与沙箱：权限、Hooks防护、容器隔离 |
| [12-resources.md](./12-resources.md) | 社区资源、GitHub 项目、推荐阅读 |

---

## 核心理念

```
探索(Explore) → 计划(Plan) → 编码(Code) → 验证(Verify) → 提交(Commit)
```

**三条铁律**：
1. **上下文是生命线** — 70% 时上下文开始降精度，85% 幻觉增加，90%+ 响应混乱
2. **CLAUDE.md 是记忆** — 把项目知识写进去，不要依赖 Claude 自己记住
3. **env 变量是最后手段** — 配置优先用 settings.json，不要依赖 Shell export

---

## 快速上手

```bash
# 安装
npm install -g @anthropic-ai/claude-code

# 首次初始化（在项目根目录）
claude
> /init

# 检查状态
> /status

# 查看帮助
> /help
```

---

## 参考来源

- [官方文档](https://docs.anthropic.com/en/docs/claude-code)
- [claude-cn.org 中文教程](https://www.claude-cn.org)
- [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code)
- [ykdojo/claude-code-tips](https://github.com/ykdojo/claude-code-tips)
- [trailofbits/claude-code-config](https://github.com/trailofbits/claude-code-config)
- [FlorianBruniaux/claude-code-ultimate-guide](https://github.com/FlorianBruniaux/claude-code-ultimate-guide)
- [januff/claude-code-tips](https://github.com/januff/claude-code-tips)（397 条社区技巧）
